# Free5GC Network Function (NF) Architecture Reference

## 1. Architectural Overview: The Goroutine Concurrency Model
Unlike C-based 5G cores that often rely on single-threaded reactor patterns, Free5GC NFs (AMF, SMF, UPF, etc.) are written in Go (Golang) and leverage its native concurrency primitives.

### Concurrency Model
Free5GC achieves high throughput and concurrent execution through **Goroutines** and the internal Go network poller. 
*   **Goroutine per Request:** For Service-Based Interfaces (SBI) running over HTTP/2, the underlying `net/http` server inherently spawns a lightweight goroutine for every incoming request. 
*   **Worker Pools for Stateful Protocols:** For interfaces like NGAP (SCTP) where sequentiality per-UE is critical to avoid state corruption, Free5GC implements hash-based worker pools (e.g., `UEScheduler`). Incoming packets are hashed by their UE ID and dispatched via channels to a specific, dedicated worker goroutine.

---

## 2. Infrastructure Layer (`pkg` and `internal`)

### A. Network Polling and Event Loop
Free5GC delegates network event polling entirely to the Go Runtime. There is no explicit `epoll` or `kqueue` abstraction maintained by the application code.
*   **Blocking Semantics:** Network reads (e.g., `conn.Read()`) appear synchronous and blocking in the Go source code, but the Go scheduler transparently parks the goroutine and uses the OS network poller in the background, waking the goroutine only when data is ready.

### B. Message Serialization and Dispatching
Message passing and synchronization between concurrent elements are managed using standard Go features:
*   **Channels:** Used extensively for dispatching tasks. For example, NGAP handlers push decoded message structures into buffered Go channels (`taskChan`), which are then consumed by worker goroutines.
*   **WaitGroups:** `sync.WaitGroup` is used universally to coordinate the lifecycle and graceful termination of all active background goroutines and servers.

### C. Memory Management and Garbage Collection
Free5GC abstracts away manual memory management, relying on the **Go Garbage Collector (GC)**.
*   **Dynamic Allocation:** Memory for UE contexts, packets, and structures is dynamically allocated on the heap or stack.
*   **No Fixed Memory Pools:** Unlike systems with strict $O(1)$ memory pools, Free5GC trades deterministic memory layout for development velocity and memory safety provided by the GC.

---

## 3. State and Context Orchestration

### A. Context Store
Every NF maintains global state within an `NFContext` structure (e.g., `AMFContext`). This context maintains lists of active connections, subscribers, and network topologies.
*   **Concurrent Access:** Since multiple goroutines (handling different SBI requests) might need to access or mutate the same state, context storage heavily relies on **`sync.Map`** and **`sync.RWMutex`**.
*   **Tiered Hierarchy:**
    1.  **Global Context:** A singleton (e.g., `amfContext`) storing NF configuration, GUAMI lists, and supported PLMNs.
    2.  **UE Context:** Represents a specific subscriber (`AmfUe`) and contains security contexts and state.
    3.  **RAN Context:** Represents a connected gNB (`AmfRan`).

### B. Finite State Machines (FSM)
Procedural logic (such as NAS processing or PDU Session establishment) is implemented using state handler functions rather than rigid function-pointer FSM arrays. State variables are stored within the UE/Session context, and standard `switch-case` statements or handler maps route the execution flow based on the current state and incoming message.

---

## 4. Network and Protocol Layer

### A. Packet Handling
Instead of reference-counted, zero-copy packet buffers (like `sk_buff` or `pkbuf`), Free5GC predominantly uses Go **byte slices (`[]byte`)**.
*   **Safety over Zero-Copy:** While there is more memory copying when wrapping/unwrapping protocols (e.g., appending headers), the Go slice semantics prevent buffer overflows and simplify memory bounds checking.

### B. SBI Implementation
The Service-Based Interface (SBI) heavily utilizes the **Gin Web Framework** (`github.com/gin-gonic/gin`) acting as a router on top of Go's `http.Server`.
*   **OpenAPI Code Generation:** HTTP handlers deal with high-level Go structs (`models.XXX`) generated directly from 3GPP OpenAPI YAML specifications.
*   **JSON Serialization:** Go's standard `encoding/json` (or optimizations thereof) handles the marshalling between HTTP payloads and C-like structures transparently.

---

## 5. Technical Integrity
This architecture ensures that Free5GC is:
*   **Memory Safe:** By leveraging Go's memory management, eliminating buffer overflows, dangling pointers, and use-after-free vulnerabilities.
*   **Highly Concurrent:** By utilizing lightweight goroutines to scale across multi-core CPUs seamlessly.
*   **Maintainable:** By preferring standard library constructs (`net/http`, `sync.Map`, Channels) over custom low-level system abstractions.
