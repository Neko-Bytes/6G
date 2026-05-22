# Open5GS Network Function (NF) Lifecycle and Execution Model

This document specifies the operational lifecycle and event-processing mechanics of an Open5GS Network Function (NF). It details how a single-threaded process manages asynchronous I/O, software queuing, and state transitions.

---

## 1. Lifecycle Phases

Every NF (e.g., AMF, SMF) progresses through three distinct operational phases, typically orchestrated in the NF's entry-point file (e.g., `src/amf/init.c`).

### A. Initialization Phase
The `initialize()` function (e.g., `amf_initialize()`) establishes the environment:
1.  **Configuration:** Parses YAML files into the NF's global context (`amf_self()`).
2.  **Memory Architecture:** Pre-allocates static memory pools (`ogs_pool_t`) for UE and session contexts.
3.  **Socket Setup:** Opens listening sockets for peer communication (SCTP/NGAP, HTTP2/SBI, UDP/PFCP) and registers their file descriptors (FDs) with the `ogs_pollset`.
4.  **Worker Creation:** Spawns the primary execution thread which runs the `main()` function.

### B. Execution Phase
The NF enters an infinite loop (e.g., `amf_main()`). This phase is characterized by a sequential "Wait-then-Process" pattern described in Section 2.

### C. Termination Phase
Triggered by signals like `SIGTERM`, the `terminate()` function performs a graceful teardown:
1.  **Peer Notification:** Sends deregistration messages to the NRF and `GOAWAY` frames to SBI peers.
2.  **Thread Joins:** Destroys the primary execution thread.
3.  **Memory Cleanup:** Finalizes memory pools and destroys the global context.

---

## 2. The Execution Loop (Reactor Pattern)

The execution thread iterates through a strict two-stage cycle. This design ensures that the NF logic remains single-threaded and race-condition free.

### Stage 1: The Multi-I/O Poll (`ogs_pollset_poll`)
The thread invokes the polling abstraction (`epoll_wait` or `kqueue`). 
*   **Multiplexing:** The poll monitors all registered sockets (FDs) simultaneously.
*   **Timer Integration:** The loop queries the timer manager (`ogs_timer_mgr_next()`) to determine the exact millisecond the next timer expires. This value is used as the `epoll_wait` timeout.
*   **Blocking:** The thread sleeps in the kernel until one of three events occur:
    1.  A packet arrives on a socket.
    2.  A timer deadline is reached.
    3.  An internal **Notification** is received.

### Stage 2: The Queue Drain
Once the poll returns, the thread processes any pending activity:
1.  **I/O Handlers:** Low-level callbacks (e.g., `ogs_sbi_server_handler`) read data from ready FDs, decode the protocol, and **push** new events onto the software queue (`ogs_app()->queue`).
2.  **Serial Processing:** The thread enters an inner loop that continues until the queue is exhausted:
    ```c
    while (ogs_queue_trypop(queue, &event) == OGS_OK) {
        ogs_fsm_dispatch(sm, event); // Execute FSM logic
        ogs_event_free(event);       // Release memory to pool
    }
    ```
3.  **Yielding:** Only when the queue is completely empty does the thread return to Stage 1.

---

## 3. Asynchronous Notification Mechanism

A common architectural challenge in single-threaded loops is waking the thread if an event is added to the queue while the thread is blocked in `poll`. Open5GS solves this using a **Self-Pipe/Notification** system.

### The Notification Flow (`lib/core/ogs-notify.c`)
1.  **Initialization:** During setup, `ogs_notify_init()` creates an `eventfd` (on Linux) or a `socketpair` and adds it to the NF's `pollset`.
2.  **Trigger:** Whenever `ogs_queue_push()` is called by any component (including internal library threads), it is immediately followed by `ogs_notify_pollset()`.
3.  **Wake-up:** `ogs_notify_pollset()` writes a small buffer to the notification FD. 
4.  **Interrupt:** This write causes the kernel to immediately wake the main thread from its `epoll_wait` (Stage 1), ensuring that new queue events are processed with minimal latency.

---

## 4. Socket and Data Handling

### Socket Registration
Sockets are never "read" in the main loop logic. Instead, they are managed via handlers:
*   **NGAP (SCTP):** Managed via `ogs_sctp_serv_add()`.
*   **SBI (HTTP/2):** Server uses `nghttp2` integration; Client uses `libcurl` multi-interface.
*   **Packet Buffer (`pkbuf`):** Raw socket data is immediately copied into a reference-counted `pkbuf_t`. This allows the data to be passed from the I/O handler, through the queue, and into the FSM without further copies.

### Data Path Trace
1.  **Arrival:** Data hits the socket buffer.
2.  **Poll Return:** `ogs_pollset_poll` detects the activity.
3.  **Library Callback:** A library handler (e.g., `recv_handler` in `nghttp2-server.c`) reads the data into a `pkbuf`.
4.  **Event Creation:** The handler identifies the event type (e.g., `OGS_EVENT_SBI_SERVER`).
5.  **Enqueuing:** The event (containing the `pkbuf`) is pushed to the queue and the pollset is notified.
6.  **FSM Dispatch:** The main loop pops the event and executes the 3GPP logic.

---

## 5. Technical References

### Core Files
*   `src/<nf>/init.c`: Orchestration of `initialize()`, `terminate()`, and the `main()` loop.
*   `lib/core/ogs-poll.c`: OS abstraction for I/O multiplexing.
*   `lib/core/ogs-notify.c`: Inter-thread and queue-to-poll notification logic.
*   `lib/core/ogs-queue.c`: The thread-safe FIFO queue implementation.
*   `lib/core/ogs-timer.c`: Timer management and poll timeout calculation.

### Key Structures
*   `ogs_pollset_t`: Stores the list of monitored FDs and the notification pipe.
*   `ogs_event_t`: A unified structure for SBI, NGAP, Timer, and Internal events.
*   `ogs_fsm_t`: The state machine context that processes events popped from the queue.
