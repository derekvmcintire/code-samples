### **Go Code Sample**

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

// ReceiptProcessHandler manages HTTP endpoints for processing receipts.
type ReceiptProcessHandler struct {
	ReceiptService internalHttp.ReceiptService
}

// NewReceiptProcessHandler initializes the HTTP handler with a ReceiptService.
//
// Parameters:
//   - service: The ReceiptService instance responsible for processing receipts.
//
// Returns:
//   - A ReceiptProcessHandler configured with the given ReceiptService.
func NewReceiptProcessHandler(service internalHttp.ReceiptService) *ReceiptProcessHandler {
	return &ReceiptProcessHandler{ReceiptService: service}
}

// ProcessReceipt validates and processes receipt data.
//
// Uses:
//   - `ShouldBindJSON`: Binds and validates incoming JSON payloads using struct tags.
//   - Gin context for managing HTTP requests and responses.
//
// Responses:
//   - 200: Returns receipt ID on success.
//   - 400: Returns validation errors in JSON format.
//   - 500: Returns internal server error for unexpected issues.
func (h *ReceiptProcessHandler) ProcessReceipt(c *gin.Context) {
	var receipt domain.Receipt

	// Bind and validate the incoming JSON data into a Receipt struct
	if err := c.ShouldBindJSON(&receipt); err != nil {
		// Return 400 Bad Request if the JSON is invalid
		c.JSON(netHttp.StatusBadRequest, gin.H{"error": "Invalid request payload", "details": err.Error()})
		return
	}

	// Process the receipt to calculate points and generate a unique receipt ID
	receiptID, err := h.ReceiptService.ProcessReceipt(receipt)
	if err != nil {
		// Return 500 Internal Server Error if an error occurs while processing the receipt
		c.JSON(netHttp.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// Return the generated receipt ID and points information in the response
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

// ReceiptService defines the interface for receipt operations.
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

// ReceiptServiceImpl handles receipt operations and integrates the core business logic.
type ReceiptServiceImpl struct {
	PointsCalculator http.PointsCalculator
	ReceiptStore     repository.ReceiptStore
}

// NewReceiptService creates a ReceiptService implementation.
//
// Parameters:
//   - c: PointsCalculator for calculating receipt points.
//   - rs: ReceiptStore for storing and retrieving receipt data.
//
// Returns:
//   - A ReceiptServiceImpl with the injected dependencies.
func NewReceiptService(c http.PointsCalculator, rs repository.ReceiptStore) http.ReceiptService {
	return &ReceiptServiceImpl{
		PointsCalculator: c,
		ReceiptStore:     rs,
	}
}

// ProcessReceipt calculates points and stores the receipt.
//
// Details:
//   - Points are calculated using a PointsCalculator.
//   - Receipts are stored with unique IDs generated by UUID.
//
// Errors:
//   - Returns errors for calculation or storage issues.
func (s *ReceiptServiceImpl) ProcessReceipt(receipt domain.Receipt) (string, error) {
	// Calculate points for the receipt using PointsCalculator
	points, err := s.PointsCalculator.CalculatePoints(receipt)
	if err != nil {
		return "", fmt.Errorf("unable to process receipt: %v", err)
	}

	// Assign calculated points to the receipt
	receipt.Points = points

	// Save the receipt to the store and return its ID
	receiptID, err := s.ReceiptStore.Save(receipt)
	if err != nil {
		return "", fmt.Errorf("failed to insert receipt: %v", err)
	}

	// Return the generated receipt ID
	return receiptID, nil
}

// GetPoints retrieves points for a receipt ID.
//
// If the receipt is not found or there is an error, an error is returned.
func (s *ReceiptServiceImpl) GetPoints(id string) (int, error) {
	// Retrieve the receipt by its ID from the store
	receipt, err := s.ReceiptStore.Find(id)
	if err != nil {
		return 0, fmt.Errorf("failed to find receipt: %v", err)
	}

	// Return the points associated with the found receipt
	return receipt.Points, nil
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

// ReceiptStoreImpl provides in-memory storage for receipts.
type ReceiptStoreImpl struct {
	receipts map[string]domain.Receipt
}

var instance *ReceiptStoreImpl
var once sync.Once

// NewReceiptStore initializes a singleton instance of ReceiptStoreImpl.
//
// Uses:
//   - Sync.Once to ensure only one instance is created.
func NewReceiptStore() repository.ReceiptStore {
	once.Do(func() {
		instance = &ReceiptStoreImpl{
			receipts: make(map[string]domain.Receipt),
		}
	})
	return instance
}

// Save stores a receipt and generates a UUID for its ID.
func (r *ReceiptStoreImpl) Save(receipt domain.Receipt) (string, error) {
	// Generate a new UUID for the receipt and store it in memory
	receiptID := uuid.New().String()
	r.receipts[receiptID] = receipt
	return receiptID, nil
}

// Find retrieves a receipt by its unique ID.
func (r *ReceiptStoreImpl) Find(id string) (domain.Receipt, error) {
	// Retrieve the receipt from the in-memory store
	foundReceipt := r.receipts[id]
	return foundReceipt, nil
}
```

### **`Receipt` Models**

These are the domain models used to represent the receipt and its items. They include validation tags to ensure that the incoming data is well-formed when received through HTTP requests.

```go
package domain

// Item represents a receipt line item.
//
// Validation:
//   - `ShortDescription` and `Price` are required fields.
type Item struct {
	ShortDescription string `json:"shortDescription" binding:"required"`
	Price            string `json:"price" binding:"required"`
}

// Receipt represents a complete receipt.
//
// Validation:
//   -

 Nested `items` field uses `dive,required` to validate each item.
type Receipt struct {
	ID           string `json:"id"`
	Retailer     string `json:"retailer" binding:"required"`
	PurchaseDate string `json:"purchaseDate" binding:"required"`
	PurchaseTime string `json:"purchaseTime" binding:"required"`
	Items        []Item `json:"items" binding:"required,dive,required"` // Ensures that each item is validated.
	Total        string `json:"total" binding:"required"`
	Points       int    `json:"points"`
}
```

---
