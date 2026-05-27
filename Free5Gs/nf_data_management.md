# Free5GC Data Management and Subscriber Storage Reference

This document provides a formal technical specification of the data management strategies, persistent storage architecture, and interaction models between Network Functions (NFs) and subscriber data repositories in Free5GC.

---

## 1. Internal Data Management Primitives

Free5GC leverages Go's built-in concurrency-safe structures to manage transient state within NFs.

### A. Concurrent Maps (`sync.Map`)
Unlike single-threaded designs that use raw hash tables or arrays, Free5GC uses Go's `sync.Map` for global pools (e.g., `UePool`, `RanUePool`).
*   **Thread Safety:** `sync.Map` provides safe concurrent reads and writes across multiple goroutines without requiring explicit locking logic from the developer.
*   **Dynamic Scaling:** The maps grow dynamically, removing the need for pre-allocated fixed-size memory pools.

### B. Mutexes (`sync.RWMutex`)
For fine-grained synchronization inside individual context structures (like `UdmUeContext`), Free5GC employs Read-Write Mutexes. This allows multiple goroutines to read subscriber data concurrently while ensuring exclusive access when state mutations occur.

---

## 2. Persistent Subscriber Storage: MongoDB

Free5GC utilizes **MongoDB** as its persistent data store for subscriber profiles, authentication vectors, and policies.

### A. The Database Interface Layer (`internal/database/mongodb`)
Free5GC interfaces with MongoDB using the official Go MongoDB Driver (`go.mongodb.org/mongo-driver`).
*   **Connection Management:** The driver manages a connection pool and handles network reconnections automatically.
*   **BSON Mapping:** Go structs representing subscriber data (from `openapi/models`) are directly tagged and serialized into BSON (Binary JSON) documents for storage.

### B. Data Schema
Subscriber data is stored in the `free5gc` database, split into specific collections:
*   **Subscription Data:** Contains SUPI, authentication keys (K, OPc), and network slice profiles.
*   **Policy Data:** Contains QoS parameters and session management policies.

---

## 3. The Unified Data Architecture (UDM/UDR)

The 5G Core separates business logic from data storage through two specialized NFs.

### A. UDR (Unified Data Repository)
The UDR acts as the sole access gateway to MongoDB.
*   **Role:** It is a stateless, RESTful front-end for the database.
*   **Interface:** Exposes the `Nudr` Service-Based Interface (SBI).
*   **Logic:** Through the `DbConnector` interface, the UDR translates incoming HTTP/2 GET/PUT/PATCH requests directly into MongoDB `bson.M` queries (e.g., `PatchDataToDBAndNotify`).

### B. UDM (Unified Data Management)
The UDM serves as the stateful logic orchestrator for subscriber management.
*   **Role:** Handles authentication calculations (5G-AKA), credential generation, and subscription updates.
*   **Dependency:** It retrieves raw data from the UDR via SBI, processes the data, and returns the computed results (or structured profiles) to the requesting NF (like the AMF or SMF).

---

## 4. Inter-NF Interaction Flow

When an NF (such as the AMF) requires subscriber data, the following interaction sequence occurs:

1.  **Request (AMF -> UDM):** The AMF sends an HTTP/2 GET request to the UDM's `Nudm_SDM` service.
2.  **Context Creation (UDM):** The UDM receives the request, spawns a goroutine, and allocates/looks up a local `UdmUeContext` in its `sync.Map`.
3.  **Data Retrieval (UDM -> UDR):** The UDM acts as an SBI client and requests the required data from the UDR.
4.  **Database Query (UDR -> MongoDB):** The UDR translates the request into a MongoDB query, retrieves the BSON document, and returns it as a JSON payload to the UDM.
5.  **Processing and Response:** The UDM stores the retrieved data in its local `UdmUeContext`, processes the requested logic (e.g., slice validation), and sends the final response back to the AMF.
6.  **Caching (AMF):** The AMF caches the validated profile in its own `AmfUe` context for subsequent high-speed local lookups.

---

## 5. Technical Integrity and Data Consistency

*   **Stateless Storage Tier:** By centralizing database connections within the UDR, other NFs remain decoupled from the DB topology.
*   **Synchronous-Style Goroutines:** Database queries via the MongoDB Go Driver block the calling goroutine, but the Go scheduler transparently multiplexes execution, meaning the UDR can handle thousands of concurrent DB queries without complex asynchronous callbacks.
*   **Data Integrity:** The use of `sync.Mutex` inside the UDM ensures that asynchronous responses from the UDR do not race with other procedures modifying the same UE context.
