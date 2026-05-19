# Open5GS Data Management and Subscriber Storage Reference

This document provides a formal technical specification of the data management strategies, persistent storage architecture, and the interaction models between Network Functions (NFs) and subscriber data repositories in Open5GS.

---

## 1. Internal Data Management Primitives

For high-performance, real-time signaling, Open5GS utilizes deterministic data structures within `lib/core` to manage transient state.

### A. Memory Pools (`ogs_pool_t`)
To eliminate the latency of dynamic memory allocation, NFs utilize pre-allocated memory pools.
*   **Static Provisioning:** All critical objects (UE, Session, Transaction) are allocated as fixed-size arrays during process startup.
*   **Identification:** Each slot in a pool is identified by an `ogs_pool_id_t`. This ID is often used as the internal correlation key across different protocol layers.

### B. High-Speed Lookups
NFs utilize two primary structures for retrieving context from memory pools:
*   **Hash Tables (`ogs_hash_t`):** Used for $O(1)$ retrieval using unique keys such as SUPI (IMSI), GUTI, or IP addresses.
*   **Red-Black Trees (`ogs_rbtree_t`):** Used for ordered data retrieval and managing internal timers where the nearest deadline must be identified in $O(\log n)$ time.

---

## 2. Persistent Subscriber Storage: MongoDB

Open5GS utilizes **MongoDB** as its primary persistent data store for subscriber profiles, authentication vectors, and session persistence.

### A. The Database Interface Layer (`lib/dbi`)
The `lib/dbi` library abstracts the interaction with the MongoDB C Driver (`mongoc`).
*   **Connection Management (`ogs_mongoc.c`):** Manages a connection pool to the MongoDB instance, handling failover and reconnection logic.
*   **BSON Mapping:** Since 3GPP SBI uses JSON, the DBI layer handles the translation between JSON objects and BSON (Binary JSON) documents used by MongoDB.

### B. Data Schema
Subscriber data is stored in the `open5gs` database, primarily within the following collections:
*   **Subscribers:** Contains the SUPI, authentication keys (K, OPC), and subscribed profiles (AMBR, slice information).
*   **Sessions:** Used for persistent storage of active PDU sessions, allowing state recovery across NF restarts.

---

## 3. The Unified Data Architecture (UDM/UDR)

The 5G Core separates data logic from data storage through two specialized NFs.

### A. UDR (Unified Data Repository)
The UDR is the **only** Network Function that directly interacts with the MongoDB database.
*   **Role:** Acts as a state-less front-end for the database.
*   **Interface:** Exposes the `Nudr` Service-Based Interface (SBI) to other NFs.
*   **Logic:** It translates incoming HTTP/2 REST requests into MongoDB queries.

### B. UDM (Unified Data Management)
The UDM serves as the logic layer for subscriber management.
*   **Role:** Handles authentication, credential generation (5G-AKA), and subscription management.
*   **Dependency:** The UDM does not have its own database; it retrieves all necessary data by acting as an SBI client to the UDR.

---

## 4. Inter-NF Interaction Flow

When an NF (such as the AMF) requires subscriber data, the following interaction sequence occurs:

1.  **Request (AMF -> UDM):** The AMF sends an HTTP/2 GET request to the UDM's `Nudm_SDM` (Subscriber Data Management) service.
2.  **Logic (UDM):** The UDM determines that it needs the subscriber profile. It initiates its own SBI request to the UDR (`Nudr_DM`).
3.  **Storage (UDR -> MongoDB):** The UDR receives the request, translates the parameters into a MongoDB query, and retrieves the BSON document via `lib/dbi`.
4.  **Response Path:**
    *   UDR converts BSON to JSON and sends it back to UDM.
    *   UDM processes the subscription data (e.g., checking if the requested slice is allowed).
    *   UDM sends the final processed profile to the AMF.
5.  **Caching:** The AMF then stores this data in its internal **UE Context** (`amf_ue_t`) memory pool for subsequent high-speed local lookups.

---

## 5. Technical Integrity and Data Consistency

*   **Stateless NFs:** By centralizing data in the UDR/MongoDB, other NFs (AMF, SMF) can remain largely stateless, facilitating horizontal scaling and graceful recovery.
*   **Asynchronous Database I/O:** Database interactions are performed asynchronously within the event loop, ensuring that the NF main thread is not blocked while waiting for disk I/O from MongoDB.
*   **Transaction Correlation:** All database-related SBI calls are correlated using the `ogs_sbi_xact_t` mechanism to ensure that responses from the UDR are correctly applied to the corresponding UE context.
