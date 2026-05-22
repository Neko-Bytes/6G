# Open5GS Network Function (NF) Data Flow and Processing

This document provides a formal technical specification of how data is represented, handled, and processed within an Open5GS Network Function (NF). It covers the transition of data from raw network packets to high-level application structures and back to outgoing transmissions.

---

## 1. Data Representation Layers

Open5GS utilizes a layered data representation model to ensure efficient processing and protocol abstraction.

### A. Network Layer: Packet Buffers (`ogs_pkbuf_t`)
The `pkbuf` (Packet Buffer) is the primary container for raw octet streams.
*   **Design:** Modeled after the Linux kernel's `sk_buff`.
*   **Reference Counting:** Multiple pointers can refer to the same `pkbuf`, allowing layers to share data without memory copies.
*   **Encapsulation Support:** Provides `push` and `pull` operations to efficiently add or remove protocol headers (e.g., GTP-U, UDP, IP) by moving internal pointers within pre-allocated headroom and tailroom.

### B. Serialization Layer: Structured C Messages
Each major protocol has a high-level C structure that represents the "decoded" state of a message:
*   **SBI (HTTP/2):** `ogs_sbi_message_t` encapsulates JSON-encoded data using structures generated from OpenAPI specifications.
*   **NGAP (ASN.1):** `ogs_ngap_message_t` represents the tree structure of NGAP Information Elements (IEs).
*   **NAS (TLV):** `ogs_nas_5gs_message_t` represents the Type-Length-Value structures of the Non-Access Stratum.

---

## 2. Inbound Data Processing (Ingress)

The ingress path transforms wire-format data into actionable state transitions.

### Step 1: Reception and Buffered I/O
The low-level socket handler (e.g., SCTP for NGAP or nghttp2 for SBI) reads the raw stream into an `ogs_pkbuf_t`.

### Step 2: Protocol Decoding
The buffer is passed to a protocol-specific decoder:
*   **ASN.1/TLV:** Decoders (generated via `asn1c` or custom TLV scripts) perform bit-level parsing to populate the corresponding C message structure.
*   **SBI/JSON:** The `ogs_sbi_parse_request()` or `ogs_sbi_parse_response()` functions use the `OpenAPI` library to deserialize JSON strings into C structures.

### Step 3: Event Enqueuing
The decoded C structure and its associated `pkbuf` are wrapped in an `ogs_event_t`. This event is pushed into the NF's main queue, signaling the execution thread to process the new data.

### Step 4: Logic Dispatch
The main loop pops the event and retrieves the relevant **Context** (e.g., `amf_ue_t`) using identifiers in the message (e.g., IMSI, GUTI, or Transaction ID). The event is then dispatched to the context's Finite State Machine (FSM).

---

## 3. Data Handling and Context Management

Once a message reaches the FSM, it is processed against the current state of the NF.

### A. Context-Based Storage
Data is not stored in the message itself for long; relevant fields are extracted and saved into **Context Structures**:
*   **UE Context:** Stores subscriber-specific data (security keys, UE capabilities, registration status).
*   **Session Context:** Stores PDU session-specific data (QoS profiles, tunnel endpoints, charging info).
*   **Peer Context:** Stores connectivity state for gNBs or other NFs.

### B. Ownership and Memory Lifecycle
*   **Short-lived Data:** The `ogs_event_t` and the temporary C message structures are freed back to memory pools immediately after the FSM state function returns.
*   **Long-lived Data:** Persistent information is stored in context structures, which are pre-allocated from `ogs_pool_t` and managed via a strictly defined context lifecycle.

---

## 4. Outbound Data Processing (Egress)

The egress path generates protocol-compliant messages based on the current state and requested actions.

### Step 1: Message Construction
The FSM logic populates a high-level C structure (e.g., `ogs_sbi_message_t` or `NGAP_NGAP_PDU_t`). This is often done using "build" helper functions found in `namf-build.c`, `ngap-build.c`, etc.

### Step 2: Protocol Encoding
The C structure is serialized:
*   **ASN.1/TLV Encoding:** The structure is converted into a binary stream and stored in a new `pkbuf_t`.
*   **SBI/JSON Encoding:** The `ogs_sbi_build_request()` function serializes the OpenAPI C structures into a JSON string and attaches it to an HTTP/2 frame.

### Step 3: Transmission
The resulting `pkbuf` or HTTP/2 frame is passed to the outbound socket handler.
*   **SCTP:** The `ogs_sctp_sendmsg()` function is used for NGAP.
*   **SBI:** The `libcurl` (for client) or `nghttp2` (for server) interface handles the asynchronous transmission.

---

## 5. Technical Integrity of Data Handling

*   **Zero-Copy Principles:** Reference counting on `pkbuf` and efficient pointer manipulation minimize CPU cycles spent on data movement.
*   **Type Safety:** The use of generated C structures from ASN.1 and OpenAPI definitions ensures that the code strictly adheres to 3GPP technical specifications.
*   **Deterministic Allocation:** All message containers and context objects are managed through `ogs_pool_t`, ensuring that data processing remains predictable under high load.
