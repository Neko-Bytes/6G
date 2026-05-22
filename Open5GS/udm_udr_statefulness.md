# UDM and UDR: Stateful vs. Stateless Architecture

This document provides a formal technical analysis of the statefulness of the Unified Data Management (UDM) and Unified Data Repository (UDR) Network Functions in Open5GS. It distinguishes between infrastructure-level state and application-level state to clarify the architectural design of these components.

---

## 1. Defining "State" in 5G Core Architecture

To understand the UDM and UDR, one must distinguish between two types of state:
1.  **Infrastructure State (Stateful Resources):** This includes open TCP/SCTP sockets, database connection pools, and file descriptors. All Network Functions (NFs) are stateful at this level because they must maintain physical connections to the network.
2.  **Application State (Subscriber State):** This refers to data about a specific User Equipment (UE) or session. A "Stateless" NF does not retain any memory of a UE between individual Service-Based Interface (SBI) transactions.

---

## 2. UDR: The Stateless Data Repository

The UDR is implemented as a **stateless database proxy**. It acts as a protocol translator between the 3GPP SBI (HTTP/2/JSON) and the persistent storage layer (MongoDB/BSON).

### Technical Implementation (`src/udr/context.h`)
*   **Empty Context:** The `udr_context_t` structure is empty. Unlike other NFs, the UDR does not maintain lists of active UEs (`ogs_list_t`) or session contexts in its local RAM.
*   **Transactional Autonomy:** Each incoming HTTP request is handled as an independent operation. The UDR performs a direct lookup in MongoDB via `lib/dbi/ogs-mongoc.c`, generates a response, and immediately releases all local variables associated with that transaction.
*   **Proof of Statelessness:** If the UDR process restarts mid-procedure, it does not lose any subscriber data, as it was never the "owner" of that data. Any other UDR instance can handle the subsequent retry without any loss of continuity.

---

## 3. UDM: The Stateful Logic Orchestrator

The UDM is implemented as a **stateful transaction manager**. While it offloads permanent data to the UDR, it maintains local state to manage asynchronous, multi-step procedures.

### Technical Implementation (`src/udm/context.h`)
*   **Populated Contexts:** The UDM maintains a robust context store including `udm_ue_t` and `udm_sess_t`. These structures are pre-allocated from memory pools (`ogs_pool_t`).
*   **Procedure Tracking:** Many UDM tasks, such as 5G-AKA authentication, require the UDM to generate data (like a RAND and AUTN), send it to the AUSF/AMF, and **wait** for a response.
*   **Asynchronous Correlation:** Because SBI calls are non-blocking, the UDM uses local UE contexts to correlate an incoming response from the UDR back to the original request from an AMF. Without this local state, the UDM would "forget" which NF it was supposed to reply to once the database query returned.

---

## 4. Architectural Interaction Flow

The following sequence illustrates how state is distributed between the two NFs during a Subscriber Data retrieval:

1.  **Ingress:** AMF sends a request to UDM.
2.  **State Initiation (UDM):** UDM allocates a slot in its `udm_ue_t` pool to track this specific transaction.
3.  **Data Retrieval:** UDM sends an SBI request to UDR.
4.  **Processing (UDR):** 
    *   UDR receives the request.
    *   UDR queries MongoDB.
    *   UDR sends JSON response.
    *   **UDR releases all memory associated with the request.**
5.  **State Resumption (UDM):** UDM receives the response, uses the local `udm_ue_t` to find the original AMF context, and completes the procedure.
6.  **State Cleanup (UDM):** Once the AMF receives its reply, the UDM releases the `udm_ue_t` slot back to the pool.

---

## 5. Summary Comparison

| Feature | UDR (Repository) | UDM (Management) |
| :--- | :--- | :--- |
| **Primary Role** | Database Front-end | Logic Orchestrator |
| **UE Contexts** | None (Stateless) | Memory Pool based (Stateful) |
| **Persistence** | None (Immediate DB Handoff) | Transactional (Procedure-based) |
| **Scalability** | Horizontal (No synchronization) | Requires Transaction Correlation |
| **Key Files** | `src/udr/nudr-handler.c` | `src/udm/udm-sm.c` |

---

## 6. Design Conclusion
The UDR's statelessness provides a high-performance, scalable gateway to MongoDB. The UDM's statefulness is a deliberate design choice in Open5GS to ensure efficient correlation of complex asynchronous procedures in C, providing the "glue" that links stateless storage with procedural logic.
