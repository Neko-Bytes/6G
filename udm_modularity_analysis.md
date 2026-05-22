# Comparative Architectural Analysis: UDM Modularity in free5gc vs. open5gs

This document provides a comprehensive analysis of the Unified Data Management (UDM) implementations in two prominent open-source 5G core projects: **free5gc** (written in Golang) and **open5gs** (written in C). It specifically examines their structural modularity, concurrency models, state management, and how their architectural choices align with emerging 6G paradigms (such as Procedural Statelessness).

---

## 1. Executive Summary

While both free5gc and open5gs implement the 3GPP standards for the UDM network function, their architectural philosophies differ fundamentally due to their language paradigms and design goals.

*   **free5gc (Golang):** Employs a **Layered/Procedural Architecture**. Modularity is achieved through clear software boundaries (Routes $\rightarrow$ Handlers $\rightarrow$ Processors $\rightarrow$ Consumers) leveraging Golang's native goroutines for high-level, thread-based concurrency.
*   **open5gs (C):** Employs a **Reactor Pattern with Finite State Machines (FSM)**. Modularity is achieved through event-driven state transitions, relying on a single-threaded asynchronous I/O loop to avoid context-switching overhead and guarantee race-condition-free execution.

---

## 2. free5gc: Procedural Layering & Goroutine Concurrency

### 2.1 Modularity Model: Vertical Separation of Concerns
free5gc divides the UDM into distinct, decoupled packages. This approach optimizes for developer velocity, readability, and code maintainability.

*   **Service & Routing (`internal/sbi`)**: The HTTP/2 API layer is built using the Gin web framework. Endpoints are logically grouped into files (`api_subscriberdatamanagement.go`, `api_eventexposure.go`), acting as pure ingress modules that handle JSON deserialization and basic structural validation.
*   **Processor Layer (`internal/sbi/processor`)**: The core business logic resides here. Processors are stateless procedural blocks that orchestrate 3GPP workflows. A processor takes an incoming request, interacts with the local context, and decides what external calls need to be made.
*   **Consumer Layer (`internal/sbi/consumer`)**: This module acts as the outbound client, encapsulating NF discovery (talking to NRF) and data retrieval (talking to UDR). It shields the Processor from network transport complexities.

### 2.2 Concurrency & State Management
*   **Concurrency:** It utilizes a **Thread-per-Request** model via lightweight goroutines. Each incoming SBI request spawns a new goroutine, allowing blocking synchronous calls to the database (UDR) without halting the entire UDM process.
*   **State:** The state is managed via a centralized, thread-safe memory object (`internal/context/context.go`). `sync.Map` and `sync.RWMutex` are used heavily to prevent race conditions when multiple goroutines read/write UE contexts simultaneously.

---

## 3. open5gs: The Single-Threaded Reactor & FSMs

### 3.1 Modularity Model: Horizontal State-Based Separation
open5gs abstracts its logic not by procedural layers, but by the lifecycle of network entities (UEs and Sessions). Its modularity is deeply integrated with its event-handling mechanism.

*   **The Main Loop (`ogs_pollset_t` & `ogs_queue_t`)**: Instead of spawning threads per request, a single execution thread polls all sockets (epoll/kqueue) asynchronously. Inbound data is wrapped in an `ogs_event_t` and pushed to a software queue.
*   **Finite State Machines (`udm-sm.c`, `ue-sm.c`)**: The core modularity boundary. The main loop pops an event and dispatches it to a state function (e.g., `udm_ue_state_operational`). Logic branches via massive `SWITCH/CASE` blocks based on the specific SBI resource.
*   **Handlers and Builders (`nudm-handler.c`, `nudr-build.c`)**: Instead of a "Processor," logic is split between "Handlers" (processing incoming messages) and "Builders" (constructing outgoing messages). This strict dichotomy is necessary for non-blocking asynchronous architectures.

### 3.2 Concurrency & State Management
*   **Concurrency:** Driven entirely by **Asynchronous I/O Multiplexing**. It is single-threaded, bypassing context-switch overhead and the need for mutexes.
*   **State & Memory:** Relies heavily on **Static Memory Pools** (`ogs_pool_t`). UDM acts as a *Stateful Logic Orchestrator*. Unlike the stateless UDR, the UDM in open5gs maintains robust `udm_ue_t` contexts locally to correlate asynchronous responses from the UDR back to the AMF requests.

---

## 4. Architectural Comparison Matrix

| Dimension | free5gc (Golang) | open5gs (C) |
| :--- | :--- | :--- |
| **Primary Pattern** | Layered / Procedural | Event-Driven / FSM (Reactor) |
| **Concurrency** | Goroutines (Thread-per-Request) | Single-threaded Event Loop |
| **Memory Allocation** | Dynamic (Garbage Collected) | Static Pools (`ogs_pool_t`) |
| **Locking Strategy** | Heavy (`sync.RWMutex`, `sync.Map`) | Lock-free (Single main thread) |
| **Routing abstraction** | Gin HTTP Router (Path/Method matching) | Internal Event Switch-Case |
| **SBI Implementation** | Standard Go `net/http` | Custom `lib/core` (nghttp2) |
| **Developer Ergonomics**| High (Intuitive linear code flow) | Moderate (Requires FSM mindset) |
| **System Overhead** | Moderate (GC pauses, routine scheduling) | Extremely Low (Deterministic) |

---

## 5. Alignment with 6G and PP5gs (Procedural Statelessness)

As networks evolve toward 6G and concepts like **PP5gs (Procedure-Based and Stateless Architecture)**, the current implementations present different evolutionary paths:

### 5.1 The "Stateful Orchestrator" Challenge
The PP5gs paradigm emphasizes fetching a *Unified Context Object* once, processing it entirely locally in fast RAM, and pushing the final state back, thereby eliminating fragmented database hops.

*   **free5gc:** The procedural nature aligns well with PP5gs conceptually. A Processor could easily be refactored to fetch a massive Procedure Block upfront, run the logic synchronously, and write it back. However, the reliance on dynamic memory and Garbage Collection poses challenges for the ultra-low latency guarantees required by 6G.
*   **open5gs:** open5gs's UDM is already designed as a *Stateful Logic Orchestrator* (as defined in `udm_udr_statefulness.md`). It temporarily holds UE state (`udm_ue_t`) to tie together asynchronous interactions. Transitioning to PP5gs would mean converting the FSM to ingest a singular massive Context Object via `ogs_event_t`, executing the entire state machine locally without intermediate UDR SBI requests, and using `nudr_build` once at the end. Its static memory pools make this highly deterministic and suitable for 6G Trust Anchor integrations.

## 6. Conclusion

*   **free5gc** offers superior vertical modularity. Its distinct separation of Transport (Router), Logic (Processor), and External I/O (Consumer) makes it highly extensible and easy to adapt to changing 3GPP specifications. It is ideal for rapid development and cloud-native Kubernetes environments where horizontal scaling compensates for per-node overhead.
*   **open5gs** offers superior execution-level modularity. By compartmentalizing behavior into FSMs and isolating I/O to a singular event loop, it guarantees memory safety and deterministic latency. It operates much closer to bare-metal efficiency, making it highly robust for embedded or extremely high-throughput environments where CPU and memory jitter must be strictly controlled.