# 6G Trust Anchor Architecture: Integrating Reputation Systems and Verifiable Datastores
### A Comprehensive Engineering Guide for Transitioning from 5G UDR to 6G User-Centric Trust Governance

---

## 1. Executive Summary & Core Motivation
In the transition from fifth-generation (5G) to sixth-generation (6G) mobile communication systems, the collection of precise, human-centric, and context-aware data (such as centimeter-level localization and behavioral tracking) introduces profound privacy and security concerns (Veith et al., 2023). 

Traditional 5G architectures rely on a **static security perimeter**: if an entity (e.g., an Application Function or a Network Function) passes firewall and cryptographic checks, it gains broad access to user databases. 

This paper proposes a paradigm shift: treating data access as a **dynamic, directed trust graph** where permissions are calculated continuously based on time-varying, multi-dimensional reputation scores. This architecture is called the **Trust Anchor (TA)**.

For your specific engineering task (replacing the free5GC MongoDB-backed Unified Data Repository with a TA-based system), this guide serves as your architectural specification manual.

```
       5G PERIMETER SECURITY (Static)             6G TRUST ANCHOR GOVERNANCE (Dynamic)
      
       +-----------------------+                  +---------+               +---------+
       |   Authenticated NF    |                  |  UE_1   |               |  UE_2   |
       +-----------+-----------+                  +----+----+               +----+----+
                   |                                   |                         |
        [Access Token Approved]                 [Reputation: 0.9]         [Reputation: 0.2]
                   v                                   v                         v
       +-----------------------+                  +----+----+               +----+----+
       |  Read/Write All Data  |                  | Active  |               | Blocked |
       |  in Central Database  |                  |  Link   |               |  Link   |
       +-----------------------+                  +----+----+               +----+----+
                                                       v                         v
                                                  +----+-------------------------+----+
                                                  |   Trust Anchor (TA) Database      |
                                                  +-----------------------------------+
```

---

## 2. Deconstructing the 5G UDR vs. 6G Trust Anchor (TA)

To understand what you are building, you must first understand what you are replacing:

### 2.1 The 5G UDR (Unified Data Repository)
As codified by 3GPP TS 23.501, the UDR is the central storage facility of the 5G Core (5GC).
* **Storage Model:** It stores highly structured data (such as subscription profiles, network slicing policies, and session records) for Network Functions like the UDM (Unified Data Management), PCF (Policy Control Function), and NEF (Network Exposure Function).
* **Access Control:** The UDR performs authorization on a rigid *per-dataset* and *per-consumer* basis. It uses standard database engines (such as MongoDB in free5GC) that do not natively scale to handle real-time, user-controlled, or reputation-based permission shifts.
* **Storage Silos:** 5G edge computing delegates storage entirely to the application layer. External applications must manage their own siloed, proprietary databases, completely isolating them from network-wide trust and security governance.

### 2.2 The 6G Trust Anchor (TA)
The TA replaces static filing cabinets with a unified, secure database and policy execution engine.
1. **Decoupling Trust Evaluation from Request Processing:** Traditional Zero-Trust architectures evaluate access policies *on every single read/write request*, which introduces severe processing latency. The TA pre-computes permissions asynchronously when reputation changes and materializes them as direct database pointers (pointers to memory or key-value structures), ensuring read/write requests perform at bare-metal speeds.
2. **User-Centric Data Spaces:** Users (UEs) own their digital data vaults. They can specify unique evaluation metrics for different classes of information (e.g., allowing an emergency service to see precise GPS data, while restricting an advertising application to coarse zip-code localization).
3. **Traceable and Verifiable Logs:** Every single state transition (reputation updates, configuration adjustments, and data writes) is appended to a cryptographic ledger to guarantee non-repudiation and prevent data tampering by the infrastructure provider itself.

---

## 3. Detailed Architectural Breakdown

The entire Trust Anchor ecosystem consists of a **Trust Anchor Node**, its **Clients** (UEs, NFs, or Applications), and external **Reputation Systems**.

```
+-------------------------------------------------------------------------+
|                           REPUTATION SYSTEM                             |
+------------------------------------+------------------------------------+
                                     | (Trust Vector Updates)
                                     v
+------------------------------------+------------------------------------+
|                           TRUST ANCHOR (TA)                             |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                        REPUTATION HANDLER                         |  |
|  |  * Evaluates: f(r) >= t                                           |  |
|  |  * Dispatches jobs to Link Worker                                  |  |
|  +---------------------------------+---------------------------------+  |
|                                    | (Asynchronous Jobs)                |
|                                    v                                    |
|  +---------------------------------+---------------------------------+  |
|  |                            LINK WORKER                            |  |
|  |  * Asynchronously sets and deletes memory links                   |  |
|  +---------------------------------+---------------------------------+  |
|                                    | (Modifies)                         |
|                                    v                                    |
|  +---------------------------------+---------------------------------+  |
|  |                            TA DATABASE                            |  |
|  |                                                                   |  |
|  |  +-------------------------+      +----------------------------+  |  |
|  |  |   Owner Space: UDR_1    |      |     Owner Space: UE_1      |  |  |
|  |  |   +-------------------+ |      |   +----------------------+ |  |  |
|  |  |   |   Owner Link to   +------------>   Data Category:     | |  |  |
|  |  |   |    UE_1 Space     | |      |   |   Localization Data  | |  |  |
|  |  |   +-------------------+ |      |   |   +----------------------+ |  |  |
|  |  +-------------------------+      +----------------------------+  |  |
|  +---------------------------------+---------------------------------+  |
|                                    | (Appends Signatures/Hashes)        |
|                                    v                                    |
|  +---------------------------------+---------------------------------+  |
|  |                            AUDIT LOG                              |  |
|  |  * Append-Only Merkle Tree (e.g., Google Trillian)                |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```

### 3.1 Owner Space ($O_k$)
An **Owner Space** is a logically isolated, secure workspace assigned to each unique network identity (such as a subscriber UE, a network core function like the UDR, or an external Application Function). 

Mathematically, the complete set of Owner Spaces $S$ for $N_O$ clients is expressed as:

$$S=\{O_k \mid k=0 \dots N_O-1\}$$

Where $O_k$ represents the specific Owner Space owned by client $k$.

### 3.2 Data Categories ($C_{i,j}$) and logical isolation
Each Owner Space is subdivided into **Data Categories**. A Data Category is a self-contained, isolated key-value namespace.
* This logical separation allows identical keys (e.g., `location`) to exist across multiple Data Categories without conflicts.
* A client has absolute read and write rights over its own Data Categories.
* Physical storage organization: The TA database (e.g., RocksDB or MongoDB) prefixes every key with the Owner ID and Data Category ID to maintain absolute physical isolation:

$$\text{Database Key} = \text{Owner\_ID} \parallel \text{Data\_Category\_ID} \parallel \text{User\_Defined\_Key}$$

### 3.3 Owner Links ($L_{k,i}$) and Data Category Links ($C_{i,j}$)
An Owner Space $O_k$ contains a list of **Owner Links** ($L_{k,i}$) pointing to other Owner Spaces:

$$O_k = \{L_{k,i} \mid i = 0 \dots N_O-1\}$$

An Owner Link $L_{k,i}$ represents the collection of actual access pathways granted to client $k$ inside the database of client $i$. These access pathways are composed of individual **Data Category Links** ($C_{i,j}$), which are permitted if and only if a dynamic trust condition $\text{Cond}(i,j,k)$ is satisfied:

$$L_{k,i}=\{C_{i,j} \mid \text{Cond}(i,j,k)\}$$

If the condition $\text{Cond}(i,j,k)$ is true, the Link Worker generates a pointer mapping client $k$'s Owner Space to the corresponding Data Category $j$ of client $i$.

---

## 4. Mathematical Formulation of Trust and Reputation

One of the defining innovations of the Trust Anchor is mapping dynamic social and behavior-driven reputation metrics to explicit, low-level database pointers.

### 4.1 The Dynamic Access Condition
For client $k$ to access Data Category $j$ owned by client $i$, the dynamic condition $\text{Cond}(i,j,k)$ must be satisfied:

$$\text{Cond}(i,j,k) \iff f_{i,j}(\vec{r}_{i,k}) \ge t_{i,j}$$

Where:
* $\vec{r}_{i,k}$ is the **trust vector** describing the multi-dimensional trust client $i$ places in client $k$.
* $f_{i,j}$ is the **evaluation function** defined by owner $i$ specifically for Data Category $j$.
* $t_{i,j}$ is the **minimum trust threshold** required to unlock Data Category $j$.

*Self-Access Constraint:* A data owner must always have complete, unrestricted access to their own data, which means their self-evaluation must always meet or exceed their threshold:

$$f_{i,j}(\vec{r}_{i,i}) \ge t_{i,j} \quad \forall j, i$$

### 4.2 Multi-Dimensional Trust Vectors ($\vec{r}_{i,k}$)
Trust cannot be condensed into a single scalar value (like $0.8$) because different scenarios require different security and capability checks. For example, a high-precision vehicle navigation system requires high operational reliability, but may not require extensive financial audit verifications.

A trust vector contains $D$ dimensions:

$$\vec{r}_{i,k} = [r^{(1)}, r^{(2)}, \dots, r^{(D)}]^T$$

Where each index (referred to as a **Trust Port**) represents a distinct operational vector, such as:
1. **$r^{(1)}$ (Service Availability):** Historical percentage of uptime.
2. **$r^{(2)}$ (Compliance and Audits):** Uptime verification records or certification states.
3. **$r^{(3)}$ (Security Posture):** Device health check states, patch levels, or active vulnerability scores.
4. **$r^{(4)}$ (Operational Performance):** Transmission latencies or bandwidth.

### 4.3 Evaluation via Scalar Products
An intuitive implementation of the evaluation function $f_{i,j}$ is the mathematical scalar product (dot product) of the trust vector and an evaluation weight vector $\vec{f}_{i,j}$:

$$f_{i,j}(\vec{r}_{i,k}) = \vec{r}_{i,k} \cdot \vec{f}_{i,j} = \sum_{d=1}^D r_{i,k}^{(d)} \cdot f_{i,j}^{(d)}$$

By configuring the weights in $\vec{f}_{i,j}$, a client can apply a custom evaluation filter (or mask) to the incoming trust vector.
* **Example:** If Client $i$ wants Data Category $j$ to depend *only* on the trustee's security posture ($r^{(3)}$) and completely ignore historical transmission latency ($r^{(4)}$), it sets $f_{i,j}^{(3)} = 1.0$ and $f_{i,j}^{(4)} = 0.0$.

### 4.4 Embedding Multi-Tenant Trust Models
The foundational base layer of the TA handles directed, asymmetric, node-to-node vectors (one vector from $A \to B$, and a separate vector from $B \to A$). This base layer can easily wrap and support alternative trust models:

#### Bidirectional Trust Scheme
In symmetric networks (such as ad-hoc Vehicle-to-Vehicle communication), trust is mutual and agreed upon. The TA maps this by enforcing a consensus protocol where the directional trust vectors are identical:

$$\vec{t}_{x \leftrightarrow y} = \vec{t}_{x,y} = \vec{t}_{y,x}$$

#### Global Reputation Scheme
In general reputation systems, every node has a single, globally visible reputation score calculated from aggregated historical feedback. The TA supports this by assigning the same global trust vector to a node regardless of who is requesting it:

$$\vec{t}_i = \vec{r}_{k,i} \quad \forall k$$

---

## 5. Procedural Workflows & Algorithmic Logic

### 5.1 Reputation Updates and Link Materialization
When a reputation system reports a new trust vector, the Trust Anchor executes a three-phase workflow to apply these changes asynchronously.

```
+-----------------------------------------------------------------------------------+
| ALGORITHM 1: Trust Vector and Link Update                                        |
+-----------------------------------------------------------------------------------+
| Inputs:                                                                           |
|   - i: Trustor client ID                                                         |
|   - k: Trustee client ID                                                         |
|   - r_ik: New trust vector supplied by the Reputation System                     |
|   - L_ii: Set of Data Categories owned by client i                                |
+-----------------------------------------------------------------------------------+
|                                                                                   |
| PHASE 1: Persistent Metadata Registration                                         |
| 1. Read previous trust vector from T-category metadata block:                     |
|    r_old_ik = Read(T_i, key = "rep_" + k)                                         |
| 2. Persist the updated trust vector to the metadata block:                       |
|    Write(T_i, key = "rep_" + k, value = r_ik)                                     |
| 3. Append metadata write to Audit Log for structural validation:                 |
|    AuditLog.AppendMetadata(T_i, "rep_" + k, r_ik)                                 |
|                                                                                   |
| PHASE 2: Evaluate Dynamic Access Conditions                                       |
| 4. For each Data Category j in L_ii:                                             |
|    a. Fetch the minimum threshold: t_ij = Read(T_i, key = "threshold_" + j)      |
|    b. Fetch the weight configuration: f_ij = Read(T_i, key = "eval_" + j)        |
|    c. Compute the previous evaluation state:                                     |
|       result_old = ( f_ij(r_old_ik) >= t_ij )                                    |
|    d. Compute the new evaluation state:                                          |
|       result_new = ( f_ij(r_ik) >= t_ij )                                        |
|    e. If result_old == FALSE and result_new == TRUE (Access Granted):            |
|       - Queue link creation: LinkWorker.QueueSetLink(O_i, L_k,i, C_i,j)          |
|    f. Else if result_old == TRUE and result_new == FALSE (Access Revoked):       |
|       - Queue link deletion: LinkWorker.QueueDeleteLink(O_i, L_k,i, C_i,j)       |
|                                                                                   |
| PHASE 3: Asynchronous Execution                                                   |
| 5. Process queued operations via background workers:                             |
|    LinkWorker.ProcessQueue()                                                     |
| 6. Return SUCCESS                                                                 |
+-----------------------------------------------------------------------------------+
```

### 5.2 Performance-Optimized Data Operations
By decoupling trust evaluation from requests, read and write operations avoid costly cryptographic or evaluation calculations. They perform simple, low-overhead link lookups.

#### Read Operations

```
+-----------------------------------------------------------------------------------+
| ALGORITHM 2: Read Data                                                            |
+-----------------------------------------------------------------------------------+
| Inputs:                                                                           |
|   - k: Requesting client ID (Consumer)                                           |
|   - i: Owner client ID (Producer)                                                 |
|   - j: Requested Data Category ID                                                 |
|   - key: Data key inside Data Category j                                          |
+-----------------------------------------------------------------------------------+
|                                                                                   |
| 1. Check if the Owner Link and target Data Category Link exist in memory:         |
|    If L_k,i exists AND C_i,j is member of L_k,i Then:                             |
|       - Fetch the target record: value = Read(C_i,j, key)                         |
|       - Return value                                                              |
| 2. Else:                                                                          |
|       - Return ACCESS_DENIED                                                      |
+-----------------------------------------------------------------------------------+
```

#### Write Operations

```
+-----------------------------------------------------------------------------------+
| ALGORITHM 3: Write Data                                                           |
+-----------------------------------------------------------------------------------+
| Inputs:                                                                           |
|   - k: Requesting client ID                                                      |
|   - i: Owner client ID                                                            |
|   - j: Requested Data Category ID                                                 |
|   - key: Data key inside Data Category j                                          |
|   - value: Raw payload data to be written                                         |
+-----------------------------------------------------------------------------------+
|                                                                                   |
| 1. Check if write permissions exist via active links:                             |
|    If L_k,i exists AND C_i,j is member of L_k,i Then:                             |
|       - Write to datastore: Write(C_i,j, key, value)                              |
|       - Append transaction hash to Audit Log:                                    |
|         AuditLog.AppendMetadata(C_i,j, key, value)                                |
|       - Return SUCCESS                                                            |
| 2. Else:                                                                          |
|       - Return ACCESS_DENIED                                                      |
+-----------------------------------------------------------------------------------+
```

---

## 6. Verifiable Cryptographic Datastore (The Audit Engine)

The Trust Anchor is designed to serve as the root of trust across the network. Because the TA is managed by an infrastructure provider (which could be compromised or untrusted), clients need a way to verify its actions. To achieve this, the TA couples its key-value database with an append-only, tamper-evident cryptographic log.

### 6.1 Cryptographic Log Mechanics
Every modification to a data category value or reputation state generates a new transaction record (**Leaf**) in a Merkle Tree structure (such as Google Trillian).

```
                  Merkle Root Hash (Published Regularly)
                               [Root_m]
                                 /  \
                                /    \
                             H_12    H_34
                             / \      / \
                            /   \    /   \
                          H_1   H_2 H_3  H_4
                           |     |   |    |
                         Leaf1 Leaf2 Leaf3 Leaf4
```

A Merkle Tree Leaf $m$ contains the following structured fields:
1. **Target Location:** Address tuple consisting of `Owner_ID`, `Category_ID`, and `Key`.
2. **Current Value Hash:** The cryptographic hash of the newly written value, $H(\text{Value}_n)$.
3. **Chaining Link:** The cryptographic hash of the *previous leaf* associated with this specific key, $H(\text{Leaf}_{m-1})$.

This structuring creates a tamper-proof verification chain. A client can verify the history of any key by validating its linear chain within the append-only Merkle tree.

### 6.2 Structural Validation Proofs
Clients can verify that the Trust Anchor is acting honestly by requesting two types of proofs:

#### 1. Consistency Proof
The consistency proof verifies that the log is append-only and that no historical entries have been modified or deleted. 
* **Mechanism:** The TA regularly publishes its latest signed tree root hash. An auditor client recalculates the previous root hash using its cached tree state and the provided proof, ensuring the older root is structurally embedded within the new root.

```
  Auditor verifies:  Root_new == Recalculate(Root_old, Consistency_Proof)
```

#### 2. Inclusion Proof
The inclusion proof verifies that a specific read/write transaction has been successfully recorded within the cryptographically signed Merkle Tree.
* **Mechanism:** When reading a key-value pair, the client receives the payload and an inclusion proof (containing the sibling hashes along the path to the root). The client hashes the payload and computes the path upward to verify it matches the globally signed root hash.

```
  Client verifies:   Root_monitored == Path_Upward(H(Value), Sibling_Hashes)
```

---

## 7. Performance Benchmarks & Empirical Evaluation

To prove that the TA can scale to handle real-world mobile networks, the researchers built and tested a high-performance prototype in **Rust**. Here is what their benchmarks revealed:

### 7.1 Link Setting Latency
The benchmark evaluated how fast the TA can update access permissions when a client's reputation changes. This setup tested a single data owner (trustor) granting or revoking permissions to up to 10,000 distinct peers (trustees) simultaneously.

```
  Average Link Creation Latency: 2.65 microseconds (µs) per link
```

* **Scalability:** The total size of the database (measured by the number of registered clients) had zero impact on link setting speeds.
* **Concurrency:** Rust’s memory safety tools and atomic reference counters (`Arc<Mutex>`) allowed background workers to update links asynchronously without blocking active data reads or writes.

### 7.2 Audit Log Storage and Engine Overhead
The prototype used **RocksDB** for key-value storage and **Google Trillian** (backed by MySQL 8.4 with the InnoDB storage engine) for the append-only Audit Log.

```
                     AUDIT LOG STORAGE GROW RATES
  +-----------------------+-----------------------+-----------------------+
  | Entry Rate [1/s]      | 50                    | 300                   |
  +-----------------------+-----------------------+-----------------------+
  | DB Growth [MB/s]      | 0.279                 | 0.683                 |
  +-----------------------+-----------------------+-----------------------+
  | Growth per Leaf       | ~5.58 kB              | ~2.28 kB              |
  +-----------------------+-----------------------+-----------------------+
```

* **The Page Allocation Penalty:** Although the raw payload size of a leaf is only 86 bytes, MySQL's InnoDB storage engine allocates storage using fixed-size blocks (pages). This structural allocation behavior means small, high-frequency writes result in more storage growth per entry compared to grouped, batched writes.
* **Optimization Strategy:** For high-throughput streams (like continuous GPS telemetry), developers should group writes into batches or write stream-level metadata to the Audit Log rather than logging individual transactions.

### 7.3 CPU Utilization under Concurrent API Loads
The tests evaluated the containerized Trust Anchor's CPU load under varying request rates from 20 to 60 concurrent clients.

* **Linear CPU Scaling:** The CPU usage showed a perfectly linear relationship with the incoming API request rate. 

```
  CPU % = 0.15% @ 1,000 requests/second  |  CPU % = 0.47% @ 3,000 requests/second
```

* **System Bottleneck:** The primary system bottleneck is the cryptographic log insertion. At request rates exceeding approximately 4,000 transactions per second, the Trillian database engine reaches its processing limit, requiring the TA to queue incoming audit requests.

---

## 8. Step-by-Step Implementation Manual for free5GC

Using the architectural principles detailed above, here is your step-by-step engineering manual to migrate free5GC’s UDR database to the 6G Trust Anchor.

```
       5G Standard Architecture               6G Migrated Target Architecture
       
       +-----------------------+                  +-----------------------+
       |       free5GC         |                  |       free5GC         |
       |  Unified Data Mgmt.   |                  |  Unified Data Mgmt.   |
       |        (UDM)          |                  |        (UDM)          |
       +-----------+-----------+                  +-----------+-----------+
                   | (HTTP REST)                              | (HTTP REST)
                   v                                          v
       +-----------+-----------+                  +-----------+-----------+
       |       free5GC         |                  |  MIGRATED free5GC UDR |
       | Unified Data Repository|                 |  (Acting as TA Client)|
       |        (UDR)          |                  +-----+-----------+-----+
       +-----------+-----------+                        |           |
                   |                                (gRPC Write) (gRPC Read)
                   v                                    |           |
       +-----------+-----------+                        v           v
       |    MongoDB Engine     |                  +-----+-----------+-----+
       |   (No Access Links)   |                  |  6G TRUST ANCHOR Node |
       +-----------------------+                  +-----------------------+
```

### Step 1: Implement the gRPC client inside the free5GC UDR
Historically, free5GC's UDR uses a MongoDB driver to save structured subscriber data. Your first step is to establish a gRPC communication link to the Trust Anchor.
1. Define the protobuf communication interface (`ta_client.proto`) containing the gRPC service definitions for `Write` and `Read` operations matching the TA specifications.
2. Compile the protobuf definition into your target programming language (Go for free5GC) to generate the client stubs.
3. Locate the database interface logic inside free5GC's UDR code (typically located under the `mongodb` database package).
4. Integrate your new gRPC client directly into the write handlers. During this initial step, **dual-write** your data: write to MongoDB as usual, and also send a copy of the write request over gRPC to the Trust Anchor.

### Step 2: Transition the Read Pipeline to use the Trust Anchor
Once you have verified that data is mirroring successfully, you can switch your active data reads to use the TA.
1. Modify the UDR's read handlers so they fetch data by calling your TA gRPC client instead of querying MongoDB.
2. Ensure that the UDR properly maps free5GC’s internal query values (e.g., matching a Subscription Permanent Identifier or SUPI) to the TA's addressing format: `Owner_ID` (the core network ID) and `Data_Category` (such as `SubscriptionData`).

### Step 3: Remove MongoDB from the UDR Core
Once your read and write pipelines are verified and stable, you can safely remove MongoDB.
1. Clean up and remove the MongoDB configuration, connection handlers, and driver dependencies from the UDR code.
2. At this stage, your UDR operates as a stateless gRPC gateway, and the Trust Anchor functions as your primary system database. All subscriber profiles are saved inside the single `UDR_Core` Owner Space.

### Step 4: Implement User-Centric Owner Spaces
Now you will transition from a single core-network database to a fully distributed, user-centric 6G trust model.
1. Instead of storing all subscriber profiles under the single `UDR_Core` Owner Space, configure the system so that **each subscriber (UE) has its own private Owner Space** (using their SUPI as their `Owner_ID`).
2. When the core network needs to read a profile during registration, the UDR (as a client) must query the target UE's private Owner Space.
3. This read request is allowed by the TA if and only if there is an active **Owner Link** mapping the UDR to the target UE's Owner Space, which is calculated based on the UDR's current reputation.

### Step 5: Implement Pre-Configured "Admin" Identity Switching
When a new phone first boots up and attempts to register with the network, its private Owner Space and the required access links do not exist yet. This introduces a bootstrapping challenge.
1. To address this in the short term, configure your TA Client with administrative privileges that allow it to dynamically **switch its identity** (by editing the `Owner ID` header field in its outgoing gRPC requests).
2. Write a system setup script that runs before free5GC boots. This script connects to the TA as an administrative client to pre-configure and pre-populate the private Owner Spaces, thresholds, and initial access links for all your test subscribers.
3. This pre-configuration ensures that when the free5GC core network boots and attempts to register your test users, the required user-centric workspaces are already populated and accessible.

---

## 9. References

* Lindenschmitt, D., Veith, B., Alam, K., Daurembekova, A., Gundall, M., Habibi, M. A., Han, B., Krummacker, D., Rosemann, P., & Schotten, H. D. (2024). Architectural Challenges of Nomadic Networks in 6G. *arXiv preprint arXiv:2409.14863*. https://doi.org/10.48550/arxiv.2409.14863
  *Cited by: 6*
* Veith, B., Krummacker, D., & Schotten, H. D. (2023). The Road to Trustworthy 6G: A Survey on Trust Anchor Technologies. *IEEE Open Journal of the Communications Society*, *4*, 581–595. https://doi.org/10.1109/ojcoms.2023.3244274
  *Cited by: 36*
