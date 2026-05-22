# Open5GS Network Function (NF) Architecture Reference

This document provides a formal technical specification of the Open5GS Network Function architecture. It is intended for developers who require a deep understanding of the system's execution model, memory management, and state orchestration.

---

## 1. Architectural Overview: The Single-Threaded Reactor
Open5GS NFs (AMF, SMF, UPF, etc.) are implemented as independent, single-threaded processes utilizing a **Reactor Pattern**. 

### Concurrency Model
To eliminate the overhead of context switching and the complexity of shared-memory synchronization (mutexes, spinlocks), all core business logic for a specific NF process executes within a single thread. Concurrency is achieved through **Asynchronous I/O Multiplexing** rather than multi-threading. High-throughput demands are met by the low overhead of this model and the ability to scale NFs horizontally across multiple process instances.

---

## 2. Infrastructure Layer (`lib/core`)

### A. Event Loop (`ogs_pollset_t`)
The execution is driven by an event loop abstraction found in `lib/core/ogs-poll.c`. 
*   **Abstraction:** It wraps OS-specific APIs (`epoll` on Linux, `kqueue` on BSD) into a unified interface.
*   **Registration:** Every communication channel—including SCTP (NGAP), HTTP/2 (SBI), PFCP, and internal Unix sockets—is registered as an `ogs_poll_t` entry.
*   **Non-Blocking Execution:** All sockets are forced into non-blocking mode. When the OS signals that a file descriptor is ready for I/O, the loop invokes a registered `ogs_poll_handler_f` callback.

### B. Message Serialization (`ogs_queue_t`)
The event loop only handles the **arrival** of data. The **processing** of that data is deferred to a thread-safe message queue.
*   **Handoff:** Low-level I/O handlers (like the `nghttp2` or `libcurl` callbacks) parse raw wire data into high-level C structures, wrap them in an `ogs_event_t`, and push them into the global `ogs_app()->queue`.
*   **Serialized Logic:** The main loop of the NF process continuously pops events from this queue. This ensures that only one "event" is being processed by the NF logic at any given time, providing implicit thread safety for all state transitions.

### C. Static Memory Management (`ogs_pool_t`)
To achieve deterministic performance and avoid heap fragmentation, Open5GS utilizes a specialized pool allocator.
*   **Static Allocation:** Memory for critical objects (UE contexts, session contexts, transactions) is pre-allocated in large contiguous blocks during process initialization.
*   **O(1) Complexity:** Allocation (`ogs_pool_alloc`) and deallocation (`ogs_pool_free`) are $O(1)$ operations involving a simple free-list index.
*   **Stability:** This prevents the NF from experiencing "jitter" or crashes due to heap exhaustion during peak traffic, as the maximum capacity is defined at startup via configuration.

---

## 3. State and Context Orchestration

### A. Finite State Machines (`ogs_fsm_t`)
Procedural logic is implemented using a Finite State Machine (FSM) framework (`lib/core/ogs-fsm.c`).
*   **State Representation:** A "State" is a function pointer.
*   **Signal Dispatching:** Events from the queue are dispatched to the current state function of a context (e.g., `amf_ue_t->sm`).
*   **Transitions:** The `OGS_FSM_TRAN(fsm, target_state)` macro handles state transitions. FSMs support standard entry/exit signals (`OGS_FSM_ENTRY_SIG`, `OGS_FSM_EXIT_SIG`) to handle setup and cleanup during transitions.

### B. Context Store
Every NF maintains a tiered hierarchy of stateful contexts:
1.  **Global Context:** A singleton (e.g., `amf_self()`) containing NF-wide configurations, peer node lists (gNBs), and service endpoints.
2.  **Peer Context:** Represents a connected node (e.g., `amf_gnb_t`).
3.  **User/Session Context:** Represents a specific subscriber (`amf_ue_t`) or session (`amf_sess_t`).
4.  **Indexing:** Contexts are cross-indexed using Hash Tables (`ogs_hash_t`) and Red-Black Trees (`ogs_rbtree_t`) to allow $O(1)$ or $O(\log n)$ retrieval by various IDs (IMSI, GUTI, TEID, IP).

---

## 4. Network and Protocol Layer

### A. Packet Buffer Management (`ogs_pkbuf_t`)
Modeled after the Linux kernel's `sk_buff`, the `pkbuf` is the standard container for packet data.
*   **Zero-Copy Design:** It uses reference counting to allow multiple layers (e.g., NAS, NGAP, SCTP) to access the same memory buffer without duplication.
*   **Headroom/Tailroom:** Buffers are allocated with extra space at the beginning and end, allowing protocol headers to be "pushed" (added) or "pulled" (removed) efficiently as the packet traverses the stack.

### B. SBI Transaction Correlation (`ogs_sbi_xact_t`)
Because HTTP/2 (Service Based Interface) is inherently asynchronous, the NF must correlate responses with their original requests.
*   **Xact Lifecycle:** When an NF initiates an outgoing request (e.g., AMF calling UDM), it creates an `ogs_sbi_xact_t` with a unique ID.
*   **Correlation:** The transaction ID is tracked. When the response arrives via the event loop, the ID is used to retrieve the transaction context, which in turn points back to the specific UE context that initiated the procedure.

---

## 5. The Operational Lifecycle (Step-by-Step)

1.  **I/O Detection:** The `ogs_pollset_poll()` function detects data on a socket.
2.  **I/O Handler:** A protocol-specific handler (SCTP for NGAP, nghttp2 for SBI) is invoked.
3.  **Decoding:** The raw buffer is decoded using generated ASN.1 or TLV codecs into a structured C object.
4.  **Queueing:** An event is created and pushed into the `ogs_queue`.
5.  **Event Loop Execution:** The main thread pops the event.
6.  **Context Lookup:** The NF retrieves the relevant UE/Session context using the ID found in the event.
7.  **FSM Dispatch:** `ogs_fsm_dispatch()` is called, triggering the logic for the current state of that context.
8.  **Action & Transition:** The state function performs business logic, potentially updates the context, sends a response via `pkbuf`, and transitions the FSM to a new state.
9.  **Cleanup:** The event and temporary buffers are released back to their respective memory pools.

---

## 6. Technical Integrity
This architecture ensures that Open5GS is:
*   **Memory Safe:** By using pre-allocated pools and reference-counted buffers.
*   **Race-Condition Free:** By serializing logic into a single thread per NF.
*   **Scaleable:** By decoupling I/O from logic, allowing the process to remain responsive even under heavy signaling load.
