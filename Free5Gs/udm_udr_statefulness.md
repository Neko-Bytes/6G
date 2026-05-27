# UDM and UDR: Stateful vs. Stateless Architecture in Free5GC

This document provides a formal technical analysis of the statefulness of the Unified Data Management (UDM) and Unified Data Repository (UDR) Network Functions in the Free5GC architecture. 

---

## 1. Defining "State" in the Go Architecture

In Free5GC, the distinction between stateful and stateless NFs depends on how application-level subscriber data is handled across multiple Service-Based Interface (SBI) transactions.

1.  **Stateless:** The NF processes an incoming request, queries an external source (like a database), returns the result, and immediately discards all memory associated with the request.
2.  **Stateful:** The NF caches data, correlates asynchronous multi-step procedures, and maintains persistent Go structs (e.g., `UeContext`) in memory (via `sync.Map`) across multiple network interactions.

---

## 2. UDR: The Stateless Data Repository

In Free5GC, the UDR is implemented as a **stateless database proxy**. It acts as the translation layer between the HTTP/2 REST APIs and the persistent MongoDB storage layer.

### Technical Implementation
*   **Direct Database Translation:** The UDR utilizes the `internal/database/mongodb` package. When an HTTP request arrives via the Gin router, the UDR handler parses the parameters, constructs a MongoDB `bson.M` filter, and executes the query via the `go.mongodb.org/mongo-driver`.
*   **Absence of Context Pools:** The UDR does not maintain a `UePool` or `sync.Map` of active subscribers in memory. The `UdrContext` solely contains configuration and routing data.
*   **Stateless execution:** Once the BSON document is retrieved, converted to JSON, and sent back as an HTTP response, the goroutine handling the request terminates, and all temporary variables are garbage collected.
*   **Horizontal Scalability:** Because it holds no subscriber state in memory, any UDR instance can serve any database request without requiring session affinity or synchronization with other UDRs.

---

## 3. UDM: The Stateful Logic Orchestrator

The UDM is implemented as a **stateful transaction manager**. It abstracts complex subscriber logic (like 5G-AKA authentication) from the raw data layer, requiring it to hold state to manage these procedures.

### Technical Implementation
*   **The `UdmUeContext`:** Found in `internal/context/context.go`, the UDM defines a rich `UdmUeContext` struct. This struct caches critical elements like `Amf3GppAccessRegistration`, `SmfSelSubsData`, and `SessionManagementSubsData`.
*   **Concurrent State Management:** The global `UDMContext` maintains a `sync.Map` called `UdmUePool`. Whenever the UDM processes a request for a subscriber (e.g., an AMF requesting subscription data), it attempts to load the UE's context from this pool. If missing, it creates a `NewUdmUe`.
*   **Data Caching and Mutexes:** Because the UDM acts as a middleman, it often retrieves data from the UDR and explicitly caches it inside the `UdmUeContext`. To protect against race conditions from concurrent SBI requests accessing the same UE, fields are protected using Go primitives like `sync.Mutex` (e.g., `amSubsDataLock`, `smfSelSubsDataLock`).

---

## 4. Architectural Interaction Flow

The interaction sequence highlights the differing state mechanics during a Subscriber Data retrieval:

1.  **Ingress:** AMF sends an HTTP request to UDM.
2.  **State Initiation (UDM):** The UDM's HTTP goroutine creates or looks up the `UdmUeContext` in its `UdmUePool` (`sync.Map`).
3.  **Data Retrieval:** The UDM acts as a client, pausing its logic to send a synchronous-style SBI request to the UDR.
4.  **Processing (UDR):** 
    *   UDR translates the request to a MongoDB query.
    *   MongoDB returns the document.
    *   UDR responds to UDM.
    *   **UDR immediately forgets the transaction.**
5.  **State Resumption (UDM):** The UDM receives the response, caches the relevant policies within the `UdmUeContext`, processes the data, and completes the original AMF request.

---

## 5. Summary Comparison

| Feature | UDR (Repository) | UDM (Management) |
| :--- | :--- | :--- |
| **Primary Role** | Database Front-end proxy | Logic Orchestrator |
| **Concurrency Model** | Goroutine per stateless query | Goroutine acting on shared Context |
| **UE Contexts** | None (Stateless) | In-memory `UdmUePool` (`sync.Map`) |
| **Persistence** | None (Immediate MongoDB proxy) | In-memory Caching |
| **Key Package** | `internal/database/mongodb` | `internal/context/context.go` |

---

## 6. Design Conclusion
The UDR provides a highly scalable, stateless translation tier directly to MongoDB, leveraging the Go HTTP server to handle immense concurrent loads. The UDM utilizes Go's concurrent `sync.Map` and mutexes to statefully orchestrate multi-step 3GPP procedures, efficiently managing subscriber context across the core network.
