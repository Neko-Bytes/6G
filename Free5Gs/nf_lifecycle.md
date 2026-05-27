# Free5GC Network Function (NF) Lifecycle and Execution Model

This document specifies the operational lifecycle and execution model of a Free5GC Network Function (NF). It details how a Go-based application initializes, manages concurrent network services, and terminates gracefully.

---

## 1. Lifecycle Phases

Every Free5GC NF (e.g., AMF, SMF) progresses through three primary phases orchestrated via the `cmd/main.go` and `pkg/service/init.go` structure.

### A. Initialization Phase
The `NewApp()` function establishes the foundational environment:
1.  **Configuration:** Parses YAML configuration files into the NF's local config struct using the `factory` package.
2.  **Context Creation:** Initializes the global `NFContext` (e.g., `amf_context.InitAmfContext()`), which instantiates dynamic structures like `sync.Map` for UE tracking.
3.  **Component Instantiation:** Instantiates internal modules such as the `Consumer` (for outbound SBI requests) and `Processor` (for handling inbound logic).
4.  **Server Setup:** Prepares the `http.Server` for the Service-Based Interface (SBI) equipped with the Gin router.

### B. Execution Phase
Triggered by the `Start()` function, the NF launches its concurrent services:
1.  **Worker Pools:** For protocols requiring sequential execution (like NGAP), the NF initializes worker goroutines (e.g., `ngap.InitScheduler()`).
2.  **Service Binding:** Network listeners are started. The SCTP server and the HTTP/2 SBI server (`sbiServer.Run()`) begin accepting connections.
3.  **Registration:** The NF acts as a consumer and registers its profile (IP, supported services) with the NRF (NF Repository Function) via an SBI POST request.
4.  **Goroutine Blocking:** The main thread blocks using a `sync.WaitGroup` (`a.wg.Wait()`), waiting for termination signals.

### C. Termination Phase
Triggered by OS signals (`SIGTERM`, `SIGINT`), a context cancellation (`ctx.Done()`) invokes the shutdown sequence:
1.  **Deregistration:** The NF sends an SBI request to deregister itself from the NRF.
2.  **Peer Notification:** Sends unavailability indications to connected peers (e.g., sending AMF Status Indication to connected RANs).
3.  **Service Shutdown:** Gracefully stops the HTTP server (`httpServer.Shutdown()`) and SCTP listeners.
4.  **Worker Teardown:** Closes task channels, allowing worker goroutines to drain their queues and exit gracefully.
5.  **Unblocking:** Once all goroutines report completion to the `WaitGroup`, the main thread unblocks and the process exits.

---

## 2. The Execution Model (Goroutines and Channels)

Unlike a single-threaded `epoll` event loop (Reactor Pattern), Free5GC relies on the Go Runtime Scheduler.

### A. Multi-Threaded I/O
*   **Accepting Connections:** The `net/http` package accepts incoming TCP/TLS connections and automatically launches a new goroutine for every HTTP request.
*   **Synchronous Code Style:** Inside these goroutines, operations (like reading from a socket or querying a database) are written in a blocking style. The Go scheduler detects blocking I/O and suspends the goroutine, scheduling other active goroutines on the CPU threads.

### B. The Dispatcher and Scheduler
To prevent race conditions on shared UE contexts, Free5GC employs a task dispatching mechanism for stateful protocols like NGAP.
1.  **I/O Goroutine:** A listener goroutine reads raw packets from the SCTP socket.
2.  **Hashing:** The dispatcher hashes the UE ID (e.g., `AMF-UE-NGAP-ID`) to determine a specific worker ID.
3.  **Channel Push:** The message is pushed into a buffered Go channel (`taskChan`) associated with that worker.
4.  **Worker Goroutine:** A dedicated worker goroutine pops tasks from its channel sequentially, executing the finite state machine logic for that UE, guaranteeing that no two messages for the same UE are processed simultaneously.

---

## 3. Technical References

### Core Files
*   `cmd/main.go`: The CLI entry point handling OS signals and invoking `NewApp`.
*   `pkg/service/init.go`: Orchestration of the `Start()`, `Terminate()`, and `listenShutdownEvent()` procedures.
*   `internal/ngap/scheduler.go`: Implementation of the worker pool and channel-based task dispatching.
*   `internal/sbi/server.go`: Configuration and launching of the Gin-based HTTP/2 server.

### Key Structures
*   `sync.WaitGroup`: Synchronizes the termination of multiple background servers and worker goroutines.
*   `context.Context`: Used extensively for propagating timeout and cancellation signals across the goroutine tree.
