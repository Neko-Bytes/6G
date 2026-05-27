# Free5GC Network Function (NF) Data Flow and Processing

This document provides a formal technical specification of how data is represented, handled, and processed within a Free5GC Network Function (NF). It covers the transition of data from raw network packets to high-level Go structures and back to outgoing transmissions.

---

## 1. Data Representation Layers

Free5GC utilizes Go's standard data types and generated structs to represent protocol layers.

### A. Network Layer: Go Slices (`[]byte`)
The primary container for raw octet streams is the standard Go slice, `[]byte`.
*   **Design:** Unlike custom reference-counted buffers (`pkbuf`), Free5GC relies on Go's runtime to allocate and garbage collect byte slices.
*   **Memory Movement:** When stripping or adding headers (like GTP-U or IP), new slices are typically allocated, or slicing operations (`data[offset:]`) are used. While this may result in more memory copies than a pure zero-copy design, it provides exceptional memory safety and prevents buffer-overflow vulnerabilities.

### B. Serialization Layer: Structured Go Types
Protocols are parsed into high-level Go structures:
*   **SBI (HTTP/2):** The `openapi` module contains structs generated from 3GPP YAML specifications (e.g., `models.Amf3GppAccessRegistration`).
*   **NGAP (ASN.1):** The `internal/ngap/message` module defines Go structures mapped from NGAP ASN.1 definitions.
*   **NAS (TLV):** The `nas` module (`github.com/free5gc/nas`) parses Non-Access Stratum Type-Length-Value streams into deeply nested Go structs representing the NAS hierarchy.

---

## 2. Inbound Data Processing (Ingress)

The ingress path transforms wire-format data into Go structs and dispatches them to handler logic.

### Step 1: Reception
*   **SBI:** The underlying Go `net/http` server accepts the HTTP/2 stream and automatically spawns a goroutine for the request.
*   **NGAP:** The SCTP listener reads raw bytes into a `[]byte` slice.

### Step 2: Protocol Decoding
*   **SBI/JSON:** The **Gin Web Framework** router matches the URI and HTTP method to a specific handler. The JSON payload is decoded directly into a `models.XXX` struct using `c.ShouldBindJSON()`.
*   **ASN.1/TLV:** The raw byte slice is passed to decoding libraries (`nasConvert` and NGAP decoders) which construct the corresponding Go structs.

### Step 3: Goroutine Dispatch
*   **SBI:** The handler executes directly within the goroutine spawned by the HTTP server.
*   **NGAP:** The parsed message is wrapped in a `Task` struct and sent via a Go channel to a dedicated worker pool (`UEScheduler`), ensuring that messages for the same UE are processed sequentially by the same worker goroutine.

---

## 3. Data Handling and Context Management

Once data is dispatched, it is processed against the NF's state.

### A. Context-Based Storage
Data is extracted and stored in thread-safe context structures (e.g., `internal/context/amf_ue.go`):
*   **UE Context:** Stores subscriber-specific data, such as security keys (`SecurityAlgorithm`) and registration state.
*   **Session Context:** Stores PDU session parameters.
*   **Concurrency Control:** Because multiple goroutines may attempt to access the same context concurrently (e.g., an incoming NGAP message and an incoming SBI notification), contexts are protected by `sync.RWMutex` locks, and context pools are managed via `sync.Map`.

### B. Ownership and Memory Lifecycle
*   **Short-lived Data:** Temporary structs representing incoming requests fall out of scope when the handler function returns and are automatically reclaimed by the Go Garbage Collector (GC).
*   **Long-lived Data:** Persistent information is explicitly copied or assigned to the NF's Context variables.

---

## 4. Outbound Data Processing (Egress)

The egress path generates protocol-compliant messages based on the current state.

### Step 1: Message Construction
The logic populates a high-level Go struct (e.g., using `ngap_message.Build...` functions).

### Step 2: Protocol Encoding
*   **SBI/JSON:** The handler passes the response struct to the Gin framework (e.g., `c.JSON(http.StatusOK, responseStruct)`), which serializes it to a JSON string and attaches it to the HTTP/2 frame.
*   **ASN.1/TLV:** The struct is serialized back into a binary `[]byte` slice using the respective encoding libraries.

### Step 3: Transmission
*   **SCTP:** The `[]byte` slice is written back to the network via `conn.Write()`.
*   **SBI:** As an SBI client, Free5GC uses Go's `http.Client` to initiate asynchronous outbound requests.

---

## 5. Technical Integrity of Data Handling

*   **Type Safety & Bounds Checking:** Go's strong typing and slice mechanics inherently prevent memory corruption issues common in C/C++ network stacks.
*   **Automatic Memory Management:** The Go GC ensures that memory allocated for complex, nested protocol structures is freed predictably, reducing the cognitive load on developers.
*   **Concurrent Simplicity:** Routing tasks via channels and processing SBI requests in independent goroutines provides a clean, synchronous-looking programming model that scales efficiently without complex async state machines.
