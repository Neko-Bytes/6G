# Trust Anchor & free5gc UDR Integration Roadmap
## Phase 1: Write Mirroring (Coexistence)

This document provides a comprehensive, step-by-step roadmap to guide you through Phase 1 of the integration. At this stage, UDR will continue to use MongoDB as its primary database but will simultaneously mirror all writes and updates to the Trust Anchor over gRPC.

---

## Roadmap Overview

```
+------------------------------------------+
|  Checkpoint 1: Tooling & Code Generation  |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|  Checkpoint 2: Go gRPC Client Wrapper    |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|  Checkpoint 3: Centralizing DB interface |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|  Checkpoint 4: Wiring Dual-Write Logic   |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|  Checkpoint 5: Testing & Verification    |
+------------------------------------------+
```

---

## Checkpoint 1: Tooling & Code Generation

### Goal
Set up your environment to compile the Trust Anchor protobuf schema into Go gRPC client code.

### Step-by-Step
1. **Install Protobuf Compiler (`protoc`)**:
   - On Linux/Ubuntu: `sudo apt update && sudo apt install -y protobuf-compiler`
   - Verify installation: `protoc --version`
2. **Install Go Plugins**:
   Install the compiler plugins that generate standard Go types and gRPC clients from `.proto` files:
   ```bash
   go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
   go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
   ```
3. **Set your PATH**:
   Ensure Go's binary directory is in your system path so `protoc` can find the plugins:
   ```bash
   export PATH="$PATH:$(go env GOPATH)/bin"
   ```
4. **Compile the Protobuf File**:
   Run the compiler inside `free5gc/NFs/udr/internal/` targeting the `trust_anchor.proto` file located in your `trust-anchor/protos` directory.
5. **Tidy Go Modules**:
   Run `go mod tidy` in the UDR directory to resolve gRPC import requirements.

> [!TIP]
> **Common Pitfall**: If `protoc` complains that `protoc-gen-go` is not found, it means Go's bin directory is not in your environment `PATH`. Check it by running `echo $PATH` and verifying it contains your Go workspace `bin` folder.

---

## Checkpoint 2: Building the Go gRPC Client Wrapper

### Goal
Implement a thread-safe singleton client in Go that manages connections, registers UDR, and sends writes asynchronously.

### Struct Pseudocode
```go
// Struct representation
type TrustAnchorClient struct {
    connection  gRPC_ClientConnection
    grpcClient  Generated_TrustAnchorClient
    ownerID     uint64
    readyMutex  RWMutex
    isReady     bool
}
```

### Connection & Registration Pseudocode
```go
// Singleton pattern setup
var clientInstance *TrustAnchorClient
var once sync.Once

func GetClient() *TrustAnchorClient {
    once.Do(func() {
        clientInstance = &TrustAnchorClient{}
        go clientInstance.initialize() // Dial asynchronously in the background
    })
    return clientInstance
}

func (c *TrustAnchorClient) initialize() {
    // 1. Fetch target address from environment variable or default
    address = getEnv("TRUST_ANCHOR_URL", "localhost:9000")
    
    // 2. Dial the gRPC server (non-blocking, insecure credentials)
    conn = grpc.Dial(address, insecure_credentials, blocking_timeout_5s)
    
    // 3. Register client to get OwnerID
    response = c.grpcClient.AddOwner(context, &AddOwner{})
    
    // 4. Save OwnerID and set ready state
    c.ownerID = response.OwnerId
    c.isReady = true
}
```

### Mirroring / Write Pseudocode
```go
func (c *TrustAnchorClient) MirrorPut(collection string, filter bson.M, data map[string]interface{}) {
    if !c.isReady {
        return // Ignore if server is offline or connecting
    }

    // 1. Serialize MongoDB payload to JSON bytes
    valBytes = json.Marshal(data)

    // 2. Design Location details
    // Decide category: e.g. use first byte of collection name as the category id
    category = []byte{ collection[0] }
    // Construct key: extract 'ueId' or '_id' from filter, fallback to string filter
    key = []byte( extractKeyFromFilter(filter) )

    // 3. Prepare payload structures
    req = &WriteRequest{
        Location: &Location{
            OwnerId:  c.ownerID,
            Category: category,
            Key:      key,
        },
        Value: valBytes,
    }

    // 4. Encode metadata (id-bin)
    // The Owner ID (uint64) must be serialized to 8 raw big-endian bytes
    idBytes = encode_uint64_to_big_endian_8bytes(c.ownerID)
    
    // Append binary metadata key "id-bin" to context
    ctx = append_binary_metadata_to_context("id-bin", idBytes)

    // 5. Send asynchronously so UDR does not wait for response
    go func() {
        ctxWithTimeout = context.WithTimeout(ctx, 2 * time.Second)
        c.grpcClient.Write(ctxWithTimeout, req)
    }()
}
```

> [!TIP]
> **Why write asynchronously?** 
> If the Trust Anchor server is temporarily slow or goes offline, a synchronous write would block UDR's request handlers, causing 5G registrations to fail. Running the client call in `go func() { ... }()` with a 2-second timeout protects UDR.

---

## Checkpoint 3: Centralizing the UDR Database Interface

### Goal
Ensure all UDR database write actions route through `database.DbConnector` rather than invoking `mongoapi` package commands directly.

### Interface Refactoring Pseudocode
In `internal/database/database.go`, update the `DbConnector` interface:
```go
type DbConnector interface {
    // Existing methods
    GetDataFromDB(collName string, filter bson.M) (map[string]interface{}, error)
    PatchDataToDBAndNotify(collName string, ueId string, patch []models.PatchItem, filter bson.M) (newValue map[string]interface{}, err error)
    
    // Add these missing write methods:
    PutDataToDB(collName string, filter bson.M, putData bson.M) error
    DeleteDataFromDB(collName string, filter bson.M)
}
```

### Concrete Implementation in Mongo Connector
In `internal/database/mongodb/mongo_db_inplement.go`:
```go
func (m MongoDbConnector) PutDataToDB(collName string, filter bson.M, putData bson.M) error {
    // Intercept and send directly to mongoapi library
    _, err := mongoapi.RestfulAPIPutOne(collName, filter, putData)
    return err
}
```

### Handler Refactoring
In UDR's processor files (e.g. `authentication_status_document.go`):
```go
// BEFORE:
mongoapi.RestfulAPIPutOne(collName, filter, putData)

// AFTER:
p.PutDataToDB(collName, filter, putData)
```

---

## Checkpoint 4: Wiring Dual-Write Logic

### Goal
Trigger write mirroring within the database implementation class.

### Inside `mongo_db_inplement.go`
```go
func (m MongoDbConnector) PutDataToDB(collName string, filter bson.M, putData bson.M) error {
    // 1. Write to MongoDB
    err := mongoapi.RestfulAPIPutOne(collName, filter, putData)
    if err != nil {
        return err
    }
    
    // 2. Mirror to Trust Anchor
    trustanchor.GetClient().MirrorPut(collName, filter, putData)
    return nil
}

func (m MongoDbConnector) PatchDataToDBAndNotify(collName string, ueId string, patchItem []models.PatchItem, filter bson.M) (orig, new map[string]interface{}, err error) {
    // 1. Apply JSON Patch to MongoDB (existing logic)
    orig, new, err = apply_json_patch_in_mongo(...)
    if err != nil {
        return
    }
    
    // 2. Mirror the updated state to the Trust Anchor
    trustanchor.GetClient().MirrorPut(collName, filter, new)
    return
}
```

### Inside `pkg/service/init.go`
Import the `trustanchor` package and trigger connection startup:
```go
func (a *UdrApp) Start() {
    // ...
    // Connect to MongoDB
    mongoapi.SetMongoDB(mongodb.Name, mongodb.Url)
    
    // Connect and Register with Trust Anchor
    _ = trustanchor.GetClient()
    // ...
}
```

---

## Checkpoint 5: Testing & Verification

### Goal
Spin up both services and confirm that a single write to UDR populates both MongoDB and the Trust Anchor RocksDB.

### Steps
1. **Start Trust Anchor**:
   Go to the `trust-anchor` repository and run:
   ```bash
   docker-compose up -d
   ```
2. **Start UDR**:
   ```bash
   export TRUST_ANCHOR_URL="localhost:9000"
   go run cmd/main.go
   ```
3. **Check Connection Logs**:
   Look at UDR stdout. You should see:
   `[INFO] Successfully registered UDR with Trust Anchor. Owner ID: 1`
4. **Trigger a Write**:
   Use a tool like `curl` or Postman to submit subscription details (e.g. AMF registration) to UDR's REST endpoint.
5. **Verify Both Databases**:
   - Check MongoDB using `mongosh` to ensure the collection contains the new record.
   - Check the Trust Anchor server's output or logs. You should see writes logged with `Owner ID: 1` and corresponding keys.

---

## Beginner Learning Playground & Exercises

If you are new to Go, gRPC, and byte manipulation, do not start coding inside the massive free5gc project immediately. Instead, create a temporary folder in your home directory or scratch space, create a file named `main.go`, and run these exercises using the command: `go run main.go`.

### Exercise 1: Go Maps and JSON Serialization (Prep for Checkpoint 2)
In UDR, database records are represented as `map[string]interface{}` (BSON maps), and you must serialize them to JSON bytes to send them to the Trust Anchor.

**Goal**: Create a map, modify it, and convert it to a string of bytes.
**Write this in `main.go` and run it**:
```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	// 1. Create a dynamic dictionary (map)
	subscriber := make(map[string]interface{})
	subscriber["ueId"] = "imsi-208950000000001"
	subscriber["status"] = "Active"
	subscriber["quota"] = 100 // Megabytes

	// 2. Convert map to JSON bytes (this is what you send over gRPC)
	jsonBytes, err := json.Marshal(subscriber)
	if err != nil {
		fmt.Println("Error converting:", err)
		return
	}

	// 3. Print the result
	fmt.Printf("Raw bytes: %v\n", jsonBytes)
	fmt.Printf("As String: %s\n", string(jsonBytes))
}
```

---

### Exercise 2: Raw Byte & Big-Endian Conversion (Prep for Checkpoint 2)
The Trust Anchor expects your client ID as 8 raw big-endian bytes under the `id-bin` header.

**Goal**: Convert a client ID (a large number) to 8 bytes of memory and verify it.
**Write this in `main.go` and run it**:
```go
package main

import (
	"encoding/binary"
	"fmt"
)

func main() {
	var ownerID uint64 = 482918 // Example client ID received from Trust Anchor

	// 1. Allocate a slide of 8 bytes (uint64 requires 8 bytes)
	idBytes := make([]byte, 8)

	// 2. Write the number into the bytes in Big Endian format
	binary.BigEndian.PutUint64(idBytes, ownerID)

	// 3. Print the binary array representation
	fmt.Printf("Numeric ID: %d\n", ownerID)
	fmt.Printf("Binary array: %v\n", idBytes)

	// 4. Convert it back to confirm it matches
	decodedID := binary.BigEndian.Uint64(idBytes)
	fmt.Printf("Decoded back: %d\n", decodedID)
}
```

---

### Exercise 3: Goroutines & Async Execution (Prep for Checkpoint 2)
We call the Trust Anchor database asynchronously so that if the Trust Anchor is slow, UDR is not affected.

**Goal**: Spawn a separate background routine to write data, letting the main code finish immediately.
**Write this in `main.go` and run it**:
```go
package main

import (
	"fmt"
	"time"
)

func mockWriteToTrustAnchor(data string) {
	fmt.Println("[Background] Starting write to Trust Anchor...")
	time.Sleep(1 * time.Second) // Simulate network delay
	fmt.Printf("[Background] Successfully wrote: %s\n", data)
}

func main() {
	fmt.Println("[Main] Writing to primary MongoDB...")
	fmt.Println("[Main] MongoDB Write successful!")

	// Spawn task in background using the 'go' keyword
	go mockWriteToTrustAnchor("User IMSI-12345")

	fmt.Println("[Main] Returning HTTP response to client immediately!")
	
	// Wait a moment so the program doesn't close before the background routine prints
	time.Sleep(2 * time.Second)
}
```

---

### Recommended Beginner Tutorials & References

If you need reference materials to study these topics, here are the most effective and free resources:

1. **Go Language Basics (Syntax & Concepts)**:
   * **[A Tour of Go](https://tour.golang.org/)**: The official interactive tutorial. Do the sections on **Variables**, **Structs**, **Maps**, and **Goroutines**.
   * **[Go by Example](https://gobyexample.com/)**: Practical code snippets demonstrating slices, maps, JSON encoding, and channels.
2. **Web APIs & HTTP REST Basics**:
   * **[MDN HTTP Overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)**: To understand how client-server HTTP requests, REST APIs, JSON payloads, and headers work.
3. **gRPC & Protobuf**:
   * **[gRPC Go Quick Start Guide](https://grpc.io/docs/languages/go/quickstart/)**: Official guide demonstrating how to install plugins and compile protobuf files.
   * **[Protocol Buffers Documentation](https://protobuf.dev/)**: To understand the `.proto` syntax and how serialization works under the hood.

