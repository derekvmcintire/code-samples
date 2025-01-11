To integrate the `StreamManagerUseCase` and `StreamVehiclesUseCase` into your project while adhering to your desired code sample format and keeping the Go Clean Architecture principles, here's how the update might look:

---

# Go Code Sample

**Author**: Derek McIntire  
**Date**: January 2025  
**Project Repository**: [derekvmcintire/MBTA-Explorer-API](https://github.com/derekvmcintire/MBTA-Explorer-API)

---

## Overview

This Go application provides live vehicle tracking for the MBTA subway system, offering real-time and cached data through a streaming architecture. Key features include:

- **Live Streaming**: Implements a singleton `StreamManager` to manage a consistent streaming session of live MBTA vehicle data.
- **Client Management**: Supports dynamic addition and removal of clients for live data broadcasting.
- **Graceful Shutdown**: Handles system signals to cleanly terminate streaming processes.
- **Separation of Concerns**: Adheres to Clean Architecture by separating use cases, interfaces, and core domain logic.

---

## Code

### **`StreamManagerUseCase`**

The `StreamManagerUseCase` ensures a live data stream is established and managed efficiently. It supports starting the stream, handling client connections, broadcasting data, and clean shutdown operations.

```go
package usecases

import (
	"context"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	ports "explorer/internal/ports/streaming"
)

// StreamManagerUseCase manages streaming operations for MBTA data.
type StreamManagerUseCase struct {
	source      ports.StreamSource
	distributor ports.StreamDistributor
	cancelFunc  context.CancelFunc
}

// NewStreamManagerUseCase creates a new instance of StreamManagerUseCase.
func NewStreamManagerUseCase(source ports.StreamSource, distributor ports.StreamDistributor) *StreamManagerUseCase {
	return &StreamManagerUseCase{
		source:      source,
		distributor: distributor,
	}
}

var streamOnce sync.Once

// EnsureStreaming ensures a single instance of the stream is running.
func (sm *StreamManagerUseCase) EnsureStreaming(url, apiKey string) {
	streamOnce.Do(func() {
		log.Println("Starting the MBTA streaming service...")

		ctx, cancel := context.WithCancel(context.Background())
		sm.cancelFunc = cancel

		go func() {
			if apiKey == "" {
				log.Fatal("MBTA_API_KEY environment variable not set")
			}
			sm.source.Start(ctx, url, apiKey)
		}()

		go func() {
			sigChan := make(chan os.Signal, 1)
			signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
			<-sigChan
			cancel()
			time.Sleep(1 * time.Second)
			os.Exit(0)
		}()
	})
}

// AddClient registers a client channel for receiving broadcast data.
func (sm *StreamManagerUseCase) AddClient(client chan string) {
	sm.distributor.AddClient(client)
}

// RemoveClient deregisters a client channel from receiving broadcast data.
func (sm *StreamManagerUseCase) RemoveClient(client chan string) {
	sm.distributor.RemoveClient(client)
}

// Broadcast sends data to all registered clients.
func (sm *StreamManagerUseCase) Broadcast(data string) {
	sm.distributor.Broadcast(data)
}

// Stop halts the streaming process.
func (sm *StreamManagerUseCase) Stop() {
	sm.cancelFunc()
	sm.distributor.Stop()
}
```

---

### **`StreamVehiclesUseCase`**

The `StreamVehiclesUseCase` is responsible for initializing the stream and managing individual client channels for receiving data.

```go
package usecases

import (
	ports "explorer/internal/ports/streaming"
)

// StreamVehiclesUseCase handles vehicle-specific streaming operations.
type StreamVehiclesUseCase struct {
	streamManager ports.StreamManager
}

// NewStreamVehiclesUseCase creates a new instance of StreamVehiclesUseCase.
func NewStreamVehiclesUseCase(sm ports.StreamManager) *StreamVehiclesUseCase {
	return &StreamVehiclesUseCase{
		streamManager: sm,
	}
}

// StreamSetup initializes the stream and returns a client channel.
func (uc *StreamVehiclesUseCase) StreamSetup(url, apiKey string) chan string {
	uc.streamManager.EnsureStreaming(url, apiKey)

	clientChan := make(chan string)
	uc.streamManager.AddClient(clientChan)
	return clientChan
}

// StreamCleanup removes a client channel to stop receiving data.
func (uc *StreamVehiclesUseCase) StreamCleanup(clientChan chan string) {
	uc.streamManager.RemoveClient(clientChan)
	close(clientChan)
}
```

---
