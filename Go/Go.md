# Go Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [derekvmcintire/go-receipt-processor](https://github.com/derekvmcintire/go-receipt-processor)

---

## Overview

This Go code demonstrates a receipt processing system that calculates reward points based on receipt details. It implements a clean architecture design using ports and adapters, and leveraging dependency injection for flexibility and modularity. The system processes receipts via HTTP requests, calculates points, stores receipts in memory, and retrieves points based on receipt IDs.

---

## Key Concepts

- **Clean Architecture**: Separation of concerns between HTTP handlers, core business logic, and storage adapters.
- **Dependency Injection**: Injecting dependencies like services and repositories to enhance testability and flexibility.
- **Ports and Adapters**: Interfaces (ports) and implementations (adapters) for core logic and storage.
- **Singleton Design Pattern**: Ensuring a single instance of the in-memory receipt store.
- **Gin Framework**: Handling HTTP requests and responses.
- **Validation**: Input validation using Gin's binding and error handling mechanisms.
- **UUIDs**: Unique identifiers for receipts to ensure traceability.

---

## Code

### **`ReceiptProcessHandler`**

Handles HTTP requests for processing receipts and returning calculated points.

```golang
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

---

### **`ReceiptService` Port**

Defines the interface for processing receipts and managing reward points.

```golang
package http

import "go-receipt-processor/internal/domain"

// ReceiptService defines the interface for processing receipts and managing points.
type ReceiptService interface {
	ProcessReceipt(receipt domain.Receipt) (receiptID string, err error)
	GetPoints(id string) (points int, err error)
}
```

---

### **`ReceiptService` Adapter**

Implements the ReceiptService interface, integrating receipt processing, point calculation, and storage.

```golang
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

---

### **`ReceiptRepository` Port**

Defines the interface for storing and retrieving receipt data.

```golang
package repository

import "go-receipt-processor/internal/domain"

// ReceiptStore defines the methods required for storing and retrieving receipts.
type ReceiptStore interface {
	Save(receipt domain.Receipt) (receiptID string, err error)
	Find(id string) (receipt domain.Receipt, err error)
}
```

---

### **`ReceiptRepository` Adapter**

Implements the ReceiptStore interface using an in-memory storage mechanism.

```golang
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

---

### **`Receipt` Models**

Defines the data structures for receipts and receipt items.

```golang
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
