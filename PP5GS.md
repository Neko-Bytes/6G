# Comprehensive Technical Guide: Normal Statelessness vs. PP5gs (Procedural Statelessness)

## 1. Introduction
This documentation outlines the architectural evolution of 5G core networks, focusing on the research concept of **PP5gs (Procedure-Based and Stateless Architecture for 5G)**. This guide synthesizes concepts regarding network state management, as referenced in *6G Trust Anchor Architecture* (series_36.pdf) and *5G Architecture* (series_36.pdf).

## 2. The Evolution of State Management
"State" in telecommunications refers to the temporary, active data required by a Network Function (NF) to maintain a user's connection (e.g., encryption keys, QoS rules, routing paths).

### 2.1 Normal Stateless Architecture (Standard 5G)
In standard 5G Service-Based Architecture (SBA), NFs are "stateless" because they do not store user session data locally.
- **Workflow:** When processing a network procedure (like UE registration), the NF queries the central database (UDSF/UDR) to fetch data for Step 1, processes it, writes it back, then queries again for Step 2, and so on.
- **Performance:** This causes severe **remote database latency and serialization overhead** due to continuous, fragmented network hops (a "chatty" communication pattern).

### 2.2 Procedural Statelessness (PP5gs)
PP5gs is a next-generation research framework that optimizes stateless operations by fetching state only *once* per procedure.
- **Workflow:** 
    1. The NF fetches a comprehensive **Unified Context Object** from the database at the initiation of the procedure.
    2. The NF executes all sub-steps locally in fast RAM.
    3. The NF writes the final state back to the database exactly *once* upon completion.
- **Performance:** This minimizes network traffic to a single read/write pair per procedure, significantly reducing latency.

---

## 3. Data Flow Visualization

### 3.1 Normal Stateless 5G Procedure (Fragmented Hops)
```text
  [ Network Function (SMF) ]                  [ Central DB (UDSF/UDR) ]
              |                                           |
              | --- 1. Fetch Auth State ----------------> |
              | <== 2. Return Auth Data ================= |  (Hop 1)
              |                                           |
              | --- 3. Fetch QoS Rules -----------------> |
              | <== 4. Return QoS Data ================== |  (Hop 2)
              |                                           |
              | --- 5. Fetch Routing Path --------------> |
              | <== 6. Return Routing Data ============== |  (Hop 3)
              v                                           v

```

### 3.2 PP5gs Procedure (Single Read/Write)

```text
  [ Network Function (SMF) ]                  [ Central DB (UDSF/UDR) ]
              |                                           |
              | --- 1. Fetch Complete Procedure Block --> |
              | <== 2. Return Unified Context Object ==== |  (Only 1 Data Hop!)
              |                                           |
              | [Executes Auth/QoS/Routing locally]       |
              |                                           |
              | --- 3. Write Final Session State -------> |
              v                                           v

```

---

## 4. Data Structure Comparison

### 4.1 Normal 5G (Fragmented Payload)

Returns small, isolated JSON payloads specific to the single sub-step being processed.

```json
{
  "ue_identity": "imsi-208950000000001",
  "auth_status": "authenticated"
}

```

### 4.2 PP5gs (Unified Context Object)

Returns a fully populated, nested object containing all variables required for the entire procedure lifecycle.

```json
{
  "procedure_id": "proc-reg-99823",
  "ue_identity": "imsi-208950000000001",
  "security_context": { 
      "encryption_key": "0x5F3C...", 
      "key_lifetime": "3600s" 
  },
  "subscription_context": { 
      "slice_id": "sst-1-default", 
      "max_bandwidth_dl": "100Mbps" 
  },
  "routing_context": { 
      "assigned_upf": "upf-edge-01", 
      "tunnel_id_dl": "0xAA12B" 
  }
}

```

---

## 5. Failure Resilience

* **The PP5gs Advantage:** If an NF executing a PP5gs procedure crashes midway through, the central database remains 100% uncorrupted because no partial, mid-step updates were written. A backup NF can immediately fetch the original, clean context object from the database and restart the procedure in milliseconds.

## 6. Trust Anchor Integration

The PP5gs model is inherently compatible with a **Trust Anchor (TA)** architecture. Because the TA acts as an autonomous reputation governance layer, it only needs to validate the NF's reputation *once* when the initial "Unified Context Object" is fetched, rather than repeatedly validating the NF's reputation for every single fragmented sub-step request. This drastically lowers the governance overhead in a zero-trust environment.

