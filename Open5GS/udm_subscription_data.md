# UDM Subscription Data in Open5GS

The Unified Data Management (UDM) in Open5GS manages various types of subscription data required for UE authentication, mobility management, and session management. While the UDM handles the logic, the actual persistent storage is provided by the Unified Data Repository (UDR), which interacts with a MongoDB database.

Below is a detailed list of the subscription data stored and managed by the UDM.

## 1. Authentication Subscription Data
This data is used by the UDM to perform 5G-AKA or EAP-AKA' authentication procedures.

| Field | Description |
| :--- | :--- |
| **K (Key)** | The 128-bit or 256-bit master secret key shared between the UE (USIM) and the Home Network. |
| **OP / OPc** | Operator Code (OP) or Computed Operator Code (OPc) used to derive authentication vectors. |
| **AMF** | Authentication Management Field, used for key derivation and sequence number management. |
| **SQN** | Sequence Number, used to synchronize the UE and the network and prevent replay attacks. |
| **Authentication Type** | Specifies the authentication method used (e.g., 5G-AKA, EAP-AKA', EAP-TLS). |

## 2. Access and Mobility Subscription Data
This data is used by the AMF (via UDM) to control the UE's access to the network and its mobility.

| Field | Description |
| :--- | :--- |
| **SUPI / IMSI** | Subscription Permanent Identifier. In Open5GS, this is typically the IMSI. |
| **Allowed S-NSSAIs** | A list of Single Network Slice Selection Assistance Information (S-NSSAIs) that the UE is authorized to use. |
| **Subscribed UE-AMBR** | The Aggregate Maximum Bit Rate allowed for all non-GBR (Guaranteed Bit Rate) PDU sessions of the UE. |
| **RFSP Index** | RAT/Frequency Selection Priority index, used by the RAN to provide specific radio resource management policies. |
| **Access Restriction Data** | Defines restrictions such as roaming limitations or disallowed Radio Access Technologies (e.g., NR, E-UTRA). |
| **Network Access Mode** | Specifies if the user is allowed for Packet Core only, or both Packet and Circuit Switched (for EPC/LTE). |

## 3. Session Management Subscription Data
This data is used by the SMF (via UDM) to authorize and configure PDU sessions.

| Field | Description |
| :--- | :--- |
| **DNN (Data Network Name)** | The name of the external data network the UE wants to connect to (equivalent to APN in 4G). |
| **Allowed PDU Session Types** | The types of PDU sessions allowed for a specific DNN (IPv4, IPv6, IPv4v6, Unstructured, Ethernet). |
| **Allowed SSC Modes** | Session and Service Continuity modes supported (Mode 1, Mode 2, or Mode 3). |
| **Default S-NSSAI** | The default network slice associated with a specific DNN if none is requested. |
| **Session-AMBR** | The Maximum Bit Rate authorized for a specific PDU session (per-DNN AMF). |
| **QoS Profile** | Quality of Service parameters including 5QI (5G QoS Identifier) and ARP (Allocation and Retention Priority). |
| **Static IP Address** | (Optional) A fixed IP address assigned to the UE for a specific DNN/Slice. |

## 4. Registration and State Data (Dynamic)
UDM also maintains the current registration status of the UE to route messages correctly.

| Field | Description |
| :--- | :--- |
| **AMF Registration** | Stores the Instance ID and address of the AMF currently serving the UE for 3GPP access. |
| **SMF Registration** | Stores the SMF Instance ID and PDU Session ID for each active PDU session. |
| **SDR Subscription** | Tracks which NFs (like AMF) have subscribed to data change notifications for a specific UE. |

---

## Data Flow Summary
1. **Provisioning**: Data is entered via the Open5GS WebUI or command-line tools into the **MongoDB** `subscribers` collection.
2. **Retrieval**: When a UE attaches, the **AMF/SMF** sends an SBI request to the **UDM**.
3. **Database Access**: The **UDM** requests the data from the **UDR**, which performs the MongoDB query.
4. **Logic**: The **UDM** processes the data (e.g., generates authentication vectors) and returns it to the requesting NF.
