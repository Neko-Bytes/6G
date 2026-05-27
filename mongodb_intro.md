# Introduction to MongoDB: Persistent Storage in Open5GS

This document provides a foundational overview of MongoDB, the database system utilized by Open5GS for persistent data storage. 

---

## 1. What is MongoDB?

MongoDB is a **NoSQL** (Non-SQL) database. Unlike traditional databases (like MySQL or PostgreSQL) that store data in rigid tables with fixed columns, MongoDB is a **Document-Oriented** database. It is designed to handle high volumes of data and frequent changes in data structure, making it ideal for the dynamic requirements of a 5G Core.

### Key Characteristics:
*   **Schema-less:** You don't need to define the structure of the data before you store it.
*   **High Availability:** It is designed to be distributed across multiple servers.
*   **JSON-like Storage:** Data is stored in a format that looks almost exactly like a C structure or a web-based JSON object.

---

## 2. Core Concepts: SQL vs. MongoDB

To understand MongoDB, it is helpful to compare its terminology with traditional "Relational" (SQL) databases:

| Relational Concept (SQL) | MongoDB Concept | Description |
| :--- | :--- | :--- |
| **Database** | **Database** | A container for multiple collections (e.g., `open5gs`). |
| **Table** | **Collection** | A grouping of similar data entries (e.g., `subscribers`). |
| **Row** | **Document** | A single record (e.g., one specific user profile). |
| **Column** | **Field** | An individual piece of data (e.g., `imsi`, `security_key`). |
| **Join** | **Embedding/Reference** | How data from different groups is linked together. |

---

## 3. Data Format: JSON and BSON

MongoDB stores data in **Documents**. These documents use a format called **BSON** (Binary JSON).

### JSON (JavaScript Object Notation)
JSON is a text-based way to represent data that is easy for humans to read. For example, a subscriber in Open5GS might look like this in JSON:
```json
{
  "imsi": "999700000000001",
  "subscribed_rau_tau_timer": 12,
  "network_access_mode": 0,
  "slice": [
    { "sst": 1, "sd": "000001" }
  ]
}
```

### BSON (Binary JSON)
While the database accepts JSON-like input, it saves the data to the disk as **BSON**. BSON is a binary representation that is optimized for speed and space, allowing the computer to search and retrieve data much faster than reading plain text.

---

## 4. Unique Identifiers: The ObjectID

Every document in MongoDB must have a unique field called `_id`. 
*   If you don't provide one, MongoDB automatically generates a 12-byte **ObjectID**.
*   In Open5GS, identifiers like the **SUPI** (Subscriber Unique Permanent Identifier) or **IMSI** are often used as the primary keys to find a specific document.

---

## 5. Why MongoDB is used in Open5GS

Open5GS chose MongoDB for several technical reasons:

1.  **Alignment with 5G Standards:** The 3GPP Service-Based Interface (SBI) uses JSON for communication between Network Functions. Since MongoDB stores data in a JSON-like format, there is very little translation overhead.
2.  **Scalability:** As the number of subscribers grows from hundreds to millions, MongoDB can be scaled horizontally (adding more servers) without changing the application code.
3.  **Flexibility:** Subscriber profiles in 5G can vary significantly (some have many PDU sessions, others have complex slice configurations). MongoDB’s flexible document structure accommodates these variations easily.

---

## 6. How it functions in this project

*   **The Driver:** Open5GS uses the **MongoDB C Driver** (`libmongoc`) to talk to the database.
*   **The Repository (UDR):** Only the **UDR** (Unified Data Repository) typically talks to MongoDB. Other Network Functions (like the AMF or SMF) ask the UDR for data, and the UDR translates those requests into MongoDB commands.
*   **Collections in Open5GS:**
    *   `subscribers`: Stores authentication keys and subscription profiles.
    *   `sessions`: Stores current active session states for recovery purposes.

---

## 7. Summary for Beginners

If you are new to this:
1.  Think of the **Database** as a filing cabinet.
2.  Think of a **Collection** as a folder inside that cabinet.
3.  Think of a **Document** as a single piece of paper inside the folder.
4.  Think of a **Field** as a specific line of information on that paper.

Instead of needing a pre-printed form with specific boxes (SQL), MongoDB lets you put any information on that piece of paper in an organized, searchable list.
