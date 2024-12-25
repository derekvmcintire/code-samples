# Go Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [derekvmcintire/go-receipt-processor](https://github.com/derekvmcintire/go-receipt-processor)

---

## Overview

This Go application implements a receipt processing system where each receipt is evaluated, processed, and associated with a set of reward points. The system is built using the **Clean Architecture** approach, promoting separation of concerns and ensuring that different components (such as core logic, HTTP handling, and storage) are independently modifiable and testable.

Key features and architectural decisions:

- **Input Validation**: The system validates incoming JSON payloads using Gin's binding tags (`binding:"required"`, `dive,required`). This ensures that the input data is well-formed before further processing.
- **Dependency Injection**: Core components such as the `ReceiptService` and `ReceiptStore` are injected into their respective handlers and services, allowing for easier testing, mocking, and extensibility.

- **Singleton Pattern**: The `ReceiptStoreImpl` is implemented as a singleton, ensuring that only one instance of the receipt store exists throughout the application, which is critical for consistent in-memory data handling.

- **Reward Points Calculation**: The system calculates reward points dynamically for each receipt based on the items it contains. This logic is encapsulated in a separate service to maintain clarity and modularity.

- **UUID for Receipt Tracking**: Each receipt is assigned a unique identifier (UUID), which is used for tracking and retrieval. This ensures each receipt can be individually processed and queried.

---

## Code

### **`ReceiptProcessHandler`**

The `ReceiptProcessHandler` struct manages HTTP requests related to receipt processing. It takes in receipt data, validates the input, and returns a unique receipt ID upon successful processing.

```go
package http

import (
	"go-receipt-processor/internal/domain"
	internalHttp "go-receipt-processor/internal/ports/core"
	"go-receipt-processor/internal/ports/http/response"
	netHttp "net/http"

	"github.com/gin-gonic/gin"
)

// ReceiptProcessHandler manages HTTP requests for processing receipts.
type ReceiptProcessHandler struct {
	ReceiptService internalHttp.ReceiptService
}

// NewReceiptProcessHandler
//
// Parameters:
//   - service: The ReceiptService responsible for processing the receipt and calculating points.
//
// Returns:
//   - A new instance of ReceiptProcessHandler with the provided ReceiptService.
func NewReceiptProcessHandler(service internalHttp.ReceiptService) *ReceiptProcessHandler {
	return &ReceiptProcessHandler{ReceiptService: service}
}

// ProcessReceipt
//
// Parameters:
//   - c: The Gin context, which contains the HTTP request and response data.
//
// Returns:
//   - A JSON response with either a 200 OK status and the receipt ID, or a 400 Bad Request if input validation fails,
//     or a 500 Internal Server Error if processing the receipt fails.
func (h *ReceiptProcessHandler) ProcessReceipt(c *gin.Context) {
	var receipt domain.Receipt

	if err := c.ShouldBindJSON(&receipt); err != nil {
		c.JSON(netHttp.StatusBadRequest, gin.H{"error": "Invalid request payload", "details": err.Error()})
		return
	}

	receiptID, err := h.ReceiptService.ProcessReceipt(receipt)
	if err != nil {
		c.JSON(netHttp.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(netHttp.StatusOK, response.ReceiptProcessResponse{
		ID: receiptID,
	})
}
```

### **`ReceiptService` Port**

The `ReceiptService` interface defines two key operations for processing receipts: calculating points for a receipt and retrieving the points associated with a specific receipt ID.

```go
package http

import "go-receipt-processor/internal/domain"

// ReceiptService defines the interface for processing receipts and managing points.
type ReceiptService interface {
	ProcessReceipt(receipt domain.Receipt) (receiptID string, err error)
	GetPoints(id string) (points int, err error)
}
```

### **`ReceiptServiceImpl`**

The `ReceiptServiceImpl` struct implements the `ReceiptService` interface and contains the business logic for calculating reward points and interacting with the storage system.

```go
package application

import (
	"fmt"
	"go-receipt-processor/internal/domain"
	http "go-receipt-processor/internal/ports/core"
	"go-receipt-processor/internal/ports/repository"
)

// ReceiptServiceImpl is an implementation of the ReceiptService interface.
type ReceiptServiceImpl struct {
	PointsCalculator http.PointsCalculator
	ReceiptStore     repository.ReceiptStore
}

// NewReceiptService
//
// Parameters:
//   - c: The PointsCalculator used to calculate points for a receipt.
//   - rs: The ReceiptStore used to store and retrieve receipts.
//
// Returns:
//   - A new instance of ReceiptServiceImpl with the provided dependencies.
func NewReceiptService(c http.PointsCalculator, rs repository.ReceiptStore) http.ReceiptService {
	return &ReceiptServiceImpl{
		PointsCalculator: c,
		ReceiptStore:     rs,
	}
}

// ProcessReceipt
//
// Parameters:
//   - receipt: The domain.Receipt object containing receipt details.
//
// Returns:
//   - receiptID: A unique identifier for the processed receipt.
//   - err: An error if processing or saving fails.
func (s *ReceiptServiceImpl) ProcessReceipt(receipt domain.Receipt) (string, error) {
	points, err := s.PointsCalculator.CalculatePoints(receipt)
	if err != nil {
		return "", fmt.Errorf("unable to process receipt: %v", err)
	}

	receipt.Points = points

	receiptID, err := s.ReceiptStore.Save(receipt)
	if err != nil {
		return "", fmt.Errorf("failed to insert receipt: %v", err)
	}

	return receiptID, nil
}

// GetPoints
//
// Parameters:
//   - id: The unique ID of the receipt whose points are being retrieved.
//
// Returns:
//   - points: The points associated with the receipt.
//   - err: An error if the receipt cannot be found or any other issue arises.
func (s *ReceiptServiceImpl) GetPoints(id string) (int, error) {
	receipt, err := s.ReceiptStore.Find(id)
	if err != nil {
		return 0, fmt.Errorf("failed to find receipt: %v", err)
	}

	points := receipt.Points

	return points, nil
}
```

### **`ReceiptStore` Port**

The `ReceiptStore` interface defines methods for storing and retrieving receipts. This allows us to implement different kinds of storage solutions (e.g., in-memory, database-backed) without affecting the rest of the application.

```go
package repository

import "go-receipt-processor/internal/domain"

// ReceiptStore defines methods for interacting with receipt storage.
type ReceiptStore interface {
	Save(receipt domain.Receipt) (receiptID string, err error)
	Find(id string) (receipt domain.Receipt, err error)
}
```

### **`ReceiptStoreImpl`**

The `ReceiptStoreImpl` struct implements an in-memory receipt store, using a singleton pattern to ensure that there is only one instance of the store in the entire application.

```go
package memory

import (
	"github.com/google/uuid"
	"go-receipt-processor/internal/domain"
	"go-receipt-processor/internal/ports/repository"
	"sync"
)

// ReceiptStoreImpl stores receipts in an in-memory map, keyed by unique IDs.
type ReceiptStoreImpl struct {
	receipts map[string]domain.Receipt
}

// Declare a private variable to hold the singleton instance
var instance *ReceiptStoreImpl
var once sync.Once

// NewReceiptStore returns the singleton instance of ReceiptStoreImpl
func NewReceiptStore() repository.ReceiptStore {
	once.Do(func() {
		// Only create the instance once
		instance = &ReceiptStoreImpl{
			receipts: make(map[string]domain.Receipt),
		}
	})
	return instance
}

// Save stores a receipt in memory and returns its ID
func (r *ReceiptStoreImpl) Save(receipt domain.Receipt) (string, error) {
	receiptID := uuid.New().String()
	r.receipts[receiptID] = receipt
	return receiptID, nil
}

// Find retrieves a receipt by ID
func (r *ReceiptStoreImpl) Find(id string) (domain.Receipt, error) {
	foundReceipt := r.receipts[id]
	return foundReceipt, nil
}
```

### **`Receipt` Models**

These are the domain models used to represent the receipt and its items. They include validation tags to ensure that the incoming data is well-formed when received through HTTP requests.

```go
package domain

type Item struct {
	ShortDescription string `json:"shortDescription" binding:"required"`
	Price            string `json:"price" binding:"required"`
}

type Receipt struct {
	ID           string `json:"id"`
	Retailer     string `json:"retailer" binding:"required"`
	PurchaseDate string `json:"purchaseDate" binding:"required"`
	PurchaseTime string `json:"purchaseTime" binding:"required"`
	Items        []Item `json:"items" binding:"required,dive,required"` // Ensure `items` is not empty and each item is validated
	Total        string `json:"total" binding:"required"`
	Points       int    `json:"points"`
}
```

---
