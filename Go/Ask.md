Can you help me update my code sample in Golang? Here is the general format I would like to keep:

# Go Code Sample

**Author**: Derek McIntire  
**Date**: November 2024  
**Project Repository**: [derekvmcintire/go-receipt-processor](https://github.com/derekvmcintire/go-receipt-processor)

---

## Overview

This Go application implements a receipt processing system where each receipt is processed, and associated with a set of reward points. The system is built using a **Clean Architecture** approach, promoting separation of concerns and ensuring that different components (core logic, HTTP handling, and storage) are independently modifiable and testable.

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

But I don't want to use that code or project. Instead, I want to use something from this project:

MBTA Train Tracker API
A Go-based API for tracking live vehicle locations and fetching route information from the MBTA v3 API. This application provides a streaming endpoint for live subway vehicle data, along with endpoints for fetching routes, stops, and shapes.

Features
Live Streaming: Stream live vehicle positions for the MBTA subway system.
Static Data Fetching:
Fetch MBTA routes.
Fetch stops for a specific route.
Fetch route shapes for mapping.
Caching: Utilizes Memcached for caching data to improve performance.
CORS Configuration: Configured for development with a default localhost:5173 frontend origin.

this is hte URL to the github repo for the project: https://github.com/derekvmcintire/MBTA-Explorer-API

And here is some code that I want to use:

```
package usecases

import (
	"context"
	ports "explorer/internal/ports/streaming"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

type StreamManagerUseCase struct {
	source      ports.StreamSource
	distributor ports.StreamDistributor
	cancelFunc  context.CancelFunc
}

func NewStreamManagerUseCase(source ports.StreamSource, Distributor ports.StreamDistributor) *StreamManagerUseCase {
	return &StreamManagerUseCase{
		source:      source,
		distributor: Distributor,
	}
}

var streamOnce sync.Once

func (sm *StreamManagerUseCase) EnsureStreaming(url, apiKey string) {
	streamOnce.Do(func() {
		log.Println("Ensuring streaming is started...")
		// Create a new context with cancellation support
		ctx, cancel := context.WithCancel(context.Background())
		sm.cancelFunc = cancel // Store the cancel function

		// Start streaming MBTA data in a separate goroutine
		go func() {
			// Check if the API key is provided, if not, log an error and stop
			if apiKey == "" {
				log.Fatal("MBTA_API_KEY environment variable not set")
			}

			// Start the MBTA stream with the provided URL and API key
			sm.source.Start(ctx, url, apiKey)
		}()

		// Handle system shutdown signals (e.g., SIGINT, SIGTERM) in a separate goroutine
		go func() {
			// Create a channel to receive shutdown signals
			sigChan := make(chan os.Signal, 1)
			// Notify the channel for interrupt (Ctrl+C) or termination signals
			signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
			// Wait for a signal to shut down the stream
			<-sigChan
			// Cancel the context to stop the streaming process
			cancel()
			// Allow a brief moment for goroutines to clean up
			time.Sleep(1 * time.Second)
			// Exit the program
			os.Exit(0)
		}()
	})
}

// Start delegates to the underlying StreamSource
func (sm *StreamManagerUseCase) Start(ctx context.Context, url, apiKey string) {
	sm.source.Start(ctx, url, apiKey) // Delegate to the actual StreamDistributor
}

// AddClient delegates to the underlying StreamDistributor
func (sm *StreamManagerUseCase) AddClient(client chan string) {
	sm.distributor.AddClient(client) // Delegate to the actual StreamDistributor
}

// RemoveClient delegates to the underlying StreamDistributor
func (sm *StreamManagerUseCase) RemoveClient(client chan string) {
	sm.distributor.RemoveClient(client) // Delegate to the actual StreamDistributor
}

// Broadcast delegates to the underlying StreamDistributor
func (sm *StreamManagerUseCase) Broadcast(data string) {
	sm.distributor.Broadcast(data) // Delegate to the actual StreamDistributor
}

// Stop delegates to the underlying StreamDistributor
func (sm *StreamManagerUseCase) Stop() {
	sm.distributor.Stop() // Delegate to the actual StreamDistributor
}

```

```
package usecases

import (
	"context"
	ports "explorer/internal/ports/streaming"
)

type StreamVehiclesUseCase struct {
	streamManager ports.StreamManager
}

func NewStreamVehiclesUseCase(sm ports.StreamManager) *StreamVehiclesUseCase {
	return &StreamVehiclesUseCase{
		streamManager: sm,
	}
}

// StreamSetup initializes the stream and returns a client channel
func (uc *StreamVehiclesUseCase) StreamSetup(url, apiKey string) chan string {
	// Ensure the stream is running
	uc.streamManager.EnsureStreaming(url, apiKey)

	// Create and register client channel
	clientChan := make(chan string, 100)
	uc.streamManager.AddClient(clientChan)

	return clientChan
}

// HandleDisconnect sets up disconnection handling for a client
func (uc *StreamVehiclesUseCase) HandleDisconnect(ctx context.Context, clientChan chan string) {
	go func() {
		<-ctx.Done()
		uc.streamManager.RemoveClient(clientChan)
	}()
}
```

```
package mbta

import (
	"context"
	ports "explorer/internal/ports/streaming"
	"log"
	"net/http"
	"time"
)

type MBTAStreamSource struct {
	distributor ports.StreamDistributor
}

// NewMBTAStreamSource initializes a new MBTAStreamSource with the given distributor.
func NewMBTAStreamSource(distributor ports.StreamDistributor) *MBTAStreamSource {
	return &MBTAStreamSource{
		distributor: distributor,
	}
}

// createRequest creates an HTTP GET request for streaming data from the MBTA API.
//
// Parameters:
// - ctx: The context to manage request lifecycle (e.g., timeouts, cancellations).
// - url: The endpoint to connect to.
// - apiKey: The API key for authorization.
//
// Returns:
// - A pointer to the created HTTP request or an error if the request creation fails.
func (m *MBTAStreamSource) createRequest(ctx context.Context, url, apiKey string) (*http.Request, error) {
	req, err := http.NewRequestWithContext(ctx, "GET", url, nil) // Create a GET request.
	if err != nil {
		return nil, err // Return error if request creation fails.
	}
	// Set necessary headers for SSE.
	req.Header.Set("Accept", "text/event-stream") // Specify content type for SSE.
	req.Header.Set("X-API-Key", apiKey)           // Add API key for authorization.
	return req, nil
}

// Start begins streaming data from the MBTA API.
//
// Parameters:
// - ctx: The context to manage the streaming lifecycle (e.g., cancellations, timeouts).
// - url: The endpoint to fetch SSE data from.
// - apiKey: The API key for authenticating the request.
//
// This method:
// - Continuously attempts to fetch and process the stream unless the context is cancelled.
// - Implements retries with a delay upon errors to avoid tight retry loops.
func (m *MBTAStreamSource) Start(ctx context.Context, url, apiKey string) {
	go func() { // Run the streaming logic in a goroutine.
		for { // Infinite loop to retry on errors or disconnections.
			select {
			case <-ctx.Done(): // Exit loop if context is cancelled.
				log.Println("Context cancelled, stopping stream")
				return
			default:
				// Fetch the stream from the MBTA API.
				respBody, err := m.fetchStream(ctx, url, apiKey)
				if err != nil {
					log.Printf("Failed to fetch stream: %v", err)
					time.Sleep(time.Second * 5) // Delay before retrying to avoid tight loops.
					continue
				}

				// Process the stream in a separate goroutine.
				processDone := make(chan struct{}) // Channel to signal completion of stream processing.
				go func() {
					defer close(processDone)    // Ensure channel closure when processing finishes.
					defer respBody.Close()      // Ensure response body is closed.
					m.scanStream(ctx, respBody) // Scan and process the stream.
				}()

				// Wait for either context cancellation or stream processing to finish.
				select {
				case <-ctx.Done(): // Stop processing if context is cancelled.
					respBody.Close()
					return
				case <-processDone: // Restart the loop on stream processing completion.
					log.Println("Stream processing ended, will retry")
				}
			}
		}
	}()
}

```

```
package distribute

import (
	"log"
	"sync"
)

type ClientDistributor struct {
	clients      map[chan string]struct{}
	clientsMutex sync.Mutex
	stop         chan struct{} // Channel to signal when to stop streaming
}

func NewClientDistributor() *ClientDistributor {
	return &ClientDistributor{
		clients: make(map[chan string]struct{}),
		stop:    make(chan struct{}),
	}
}

// Broadcast sends the given data to all connected clients.
// This method ensures thread-safe access to the client map and skips clients
// that are unable to keep up with the data flow.
func (cd *ClientDistributor) Broadcast(data string) {
	// Acquire the mutex lock to safely access the client map.
	cd.clientsMutex.Lock()
	defer cd.clientsMutex.Unlock() // Ensure the mutex is unlocked after the operation.

	// Iterate over all registered clients.
	for client := range cd.clients {
		select {
		case client <- data:
			// Attempt to send data to the client's channel.
			// If the client is ready to receive, this operation succeeds immediately.
		default:
			// Skip clients whose channels are full or unresponsive.
			// Log a message to indicate the client was skipped due to being slow.
			log.Println("Stream Manager Client is slow, skipping...")
		}
	}
}

// AddClient adds a new client channel to the manager to receive data updates.
// It locks the client list to ensure thread safety during modification.
func (cd *ClientDistributor) AddClient(client chan string) {
	cd.clientsMutex.Lock()          // Lock to ensure safe access to the clients map
	defer cd.clientsMutex.Unlock()  // Unlock once the operation is done
	cd.clients[client] = struct{}{} // Add the client channel to the map of active clients
}

// RemoveClient removes a client channel when they disconnect.
// It locks the client list to ensure thread safety during modification.
func (cd *ClientDistributor) RemoveClient(client chan string) {
	cd.clientsMutex.Lock()         // Lock to ensure safe access to the clients map
	defer cd.clientsMutex.Unlock() // Unlock once the operation is done
	// Check if the client exists in the map
	if _, ok := cd.clients[client]; ok {
		// Remove the client from the map and close the channel to signal disconnection
		delete(cd.clients, client)
		close(client)
	}
}

// Stop stops the stream manager and signals all processes to stop.
// It closes the stop channel to initiate the shutdown process.
func (cd *ClientDistributor) Stop() {
	// Close the stop channel to signal all goroutines to stop
	close(cd.stop)
}
```

```
package mbta

import (
	"bufio"
	"context"
	"io"
	"log"
	"strings"
)

// scanStream reads and processes server-sent events (SSE) from the response body stream.
//
// Parameters:
// - ctx: The context to manage cancellation or timeouts.
// - responseBody: The stream to be read, typically the HTTP response body.
//
// Functionality:
// - Reads lines from the stream using a buffered scanner.
// - Buffers lines for each SSE message until a blank line indicates the end of the event.
// - Processes complete SSE messages and handles errors in the stream.
func (m *MBTAStreamSource) scanStream(ctx context.Context, responseBody io.Reader) {
	// Create a buffered scanner to read the response body line by line.
	scanner := bufio.NewScanner(responseBody)

	// Set a 1 MB buffer size for the scanner to handle large event streams.
	buffer := make([]byte, 1024*1024)
	scanner.Buffer(buffer, len(buffer))

	var eventBuffer []string // Buffer to accumulate lines for a single SSE event.

	// Loop through each line in the stream.
	for scanner.Scan() {
		select {
		case <-ctx.Done(): // Exit if the context is canceled.
			return
		default:
			line := scanner.Text() // Get the current line from the stream.

			// Check for an empty line signaling the end of an SSE event.
			if line == "" {
				// If the buffer has accumulated lines, process the event.
				if len(eventBuffer) > 0 {
					fullEvent := strings.Join(eventBuffer, "\n") // Combine buffered lines.
					m.processSSE(fullEvent)                      // Process the complete SSE event.
					eventBuffer = []string{}                     // Clear the buffer for the next event.
				}
				continue // Skip to the next line.
			}

			// Accumulate lines for the current SSE event.
			eventBuffer = append(eventBuffer, line)
		}
	}

	// Handle errors that may occur while scanning the stream.
	if err := scanner.Err(); err != nil {
		log.Printf("Error reading stream: %v", err) // Log the error for debugging.
	}
}
```

```
package mbta

import (
	"fmt"
	"strings"
)

// processSSE parses a Server-Sent Events (SSE) message, extracts its fields,
// formats it into an SSE-compliant message, and broadcasts it to connected clients.
//
// Parameters:
// - event: The raw SSE event string received from the server.
//
// Functionality:
// - Splits the raw event string into lines to parse individual fields.
// - Extracts the "event" and "data" fields from the message.
// - Formats the parsed fields into an SSE-compliant message.
// - Broadcasts the formatted message to all connected clients via the distributor.
func (m *MBTAStreamSource) processSSE(event string) {
	// Split the raw event string into lines for processing.
	lines := strings.Split(event, "\n")

	var eventType string   // Holds the extracted "event" field value.
	var eventData []string // Accumulates "data" field values.

	// Process each line to extract relevant SSE fields.
	for _, line := range lines {
		if strings.HasPrefix(line, "event:") {
			// Extract and trim the value of the "event" field.
			eventType = strings.TrimSpace(line[len("event:"):])
		} else if strings.HasPrefix(line, "data:") {
			// Extract and trim the value of the "data" field and add it to eventData.
			eventData = append(eventData, strings.TrimSpace(line[len("data:"):]))
		}
	}

	// Combine all data lines into a single string, separated by newline characters.
	fullData := strings.Join(eventData, "\n")

	// If data exists, format and broadcast the SSE-compliant message.
	if fullData != "" {
		// Format the SSE message with the event type and data.
		formattedEvent := fmt.Sprintf("event: %s\ndata: %s\n\n", eventType, fullData)

		// Broadcast the formatted message to all connected clients.
		m.distributor.Broadcast(formattedEvent)
	}
}

```

```
package mbta

import (
	"context"
	"fmt"
	"io"
	"net/http"
)

// fetchStream handles the HTTP request logic to establish a stream connection
// to the specified URL and returns the response body for further processing.
//
// Parameters:
// - ctx: Context to manage request lifecycle and handle cancellations or timeouts.
// - url: The URL to connect to for the stream.
// - apiKey: API key for authentication.
//
// Returns:
// - io.ReadCloser: The response body for reading the stream data.
// - error: An error if the request fails or the response status is not OK.
func (m *MBTAStreamSource) fetchStream(ctx context.Context, url, apiKey string) (io.ReadCloser, error) {
	// Create a new HTTP GET request with the provided context, URL, and API key.
	req, err := m.createRequest(ctx, url, apiKey)
	if err != nil {
		return nil, err // Return error if the request creation fails
	}

	// Initialize an HTTP client and make the GET request.
	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err // Return error if the request execution fails
	}

	// Verify that the response status code indicates success (200 OK).
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close() // Close the response body to avoid resource leaks
		return nil, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
	}

	// Return the response body for reading the stream.
	return resp.Body, nil
}
```

Can you create a concise code sample using markdown? Only use the parts of the code that you think are essential for demonstrating my coding skills, and try to keep the entire markdown file less than 300 lines. Feel free to Add comments to the code or make changes for readability, just let me know what you changed.
