# Xcode CAS Protocol Documentation 🚀

## Overview

This document describes the **Compilation Cache Service Protocol** used by Xcode to communicate with remote caching servers. The protocol enables distributed caching of Swift and Clang compilation artifacts.

- **Protocol Type:** gRPC over HTTP/2
- **Transport:** Unix Domain Socket or TCP
- **Client Library:** `grpc-swift-nio/1.14.1`
- **Protocol Definitions:** [`proto/`](proto/)

## Services 🛠️

The protocol consists of two gRPC services:

### 1. KeyValueDB Service 🔑

**Package:** `compilation_cache_service.keyvalue.v1`

**Service:** `KeyValueDB`

**Proto Definition:** [`proto/keyvalue.proto`](proto/keyvalue.proto)

#### GetValue Method

Queries the cache for a specific key and retrieves the complete cached artifact.

**RPC Path:**
```
/compilation_cache_service.keyvalue.v1.KeyValueDB/GetValue
```

**Request:**
- `key` (bytes): Cache key derived from source content, compiler flags, SDK version, module dependencies, and build configuration

**Response:**
- `found` (bool): Whether the artifact exists in the cache
- `value` (bytes): **The complete artifact data** (only present if `found=true`)

**Important:** The response contains the **full artifact data inline**, not a reference. There is no separate fetch operation.

**Usage Pattern:**
- Called at the beginning of compilation
- Multiple parallel requests (one per artifact)
- If `found=true`, Xcode uses the returned artifact directly
- If `found=false`, Xcode compiles and uploads via `Save`

---

### 2. CASDBService 💾

**Package:** `compilation_cache_service.cas.v1`

**Service:** `CASDBService`

**Proto Definition:** [`proto/cas.proto`](proto/cas.proto)

#### Save Method

Stores a compilation artifact in the Content-Addressable Storage (CAS).

**RPC Path:**
```
/compilation_cache_service.cas.v1.CASDBService/Save
```

**Request:**
- `cas_id` (bytes): Content hash of the artifact data
- `data` (bytes): The complete artifact data
- `type` (string): Artifact type (e.g., "pcm", "o", "metadata")
- `metadata` (map<string, string>): Optional metadata

**Response:**
- `cas_id` (bytes): Confirmed CAS ID
- `success` (bool): Whether the save succeeded
- `message` (string): Optional status message

**Usage Pattern:**
- Called during and after compilation
- Multiple concurrent uploads
- Stores precompiled modules (.pcm), objects (.o), and metadata

## Protocol Details 📋

### Connection

**Transport Options:**
- Unix Domain Socket (preferred for local development)
- TCP (for remote servers)

**Protocol:** HTTP/2 with gRPC framing

**Socket Path Configuration:**
```bash
COMPILATION_CACHE_REMOTE_SERVICE_PATH=/path/to/cas.sock
```

### Request Headers

Every gRPC request includes these HTTP/2 headers:

```
:method: POST
:path: /compilation_cache_service.{package}.{version}.{Service}/{Method}
:authority: localhost
:scheme: http
content-type: application/grpc
te: trailers
user-agent: grpc-swift-nio/1.14.1
```

### Response Format

Successful responses include:

```
:status: 200
content-type: application/grpc
grpc-status: 0
grpc-message: OK
```

## Message Flow 🔄

### Typical Build Sequence

**1. Initialization** ⚡
- Xcode establishes HTTP/2 connection
- SETTINGS frame exchange

**2. Cache Query Phase** 🔍

```
→ GetValue(key1)  ┐
→ GetValue(key2)  │ Parallel
→ GetValue(key3)  │ queries
→ ...             │
→ GetValue(keyN)  ┘

← Response with full artifact data (if found)
```

**3. Compilation Phase** ⚙️
- Only artifacts that missed cache are compiled locally

**4. Upload Phase** ⬆️

```
→ Save(artifact1)  ┐
→ Save(artifact2)  │ Concurrent
→ Save(artifact3)  │ uploads
→ ...              │
→ Save(artifactN)  ┘

← Confirmation
```

## Artifact Types 📦

The protocol handles these compilation artifacts:

| Type             | Description                           |
| ---------------- | ------------------------------------- |
| **Metadata**     | Cache keys, mappings, build metadata  |
| **PCM Files**    | Precompiled Swift Modules (.pcm)      |
| **Object Files** | Compiled objects (.o)                 |
| **Diagnostics**  | Compiler warnings and errors          |

## Protocol Buffers 📝

Complete protocol buffer definitions are available in the [`proto/`](proto/) directory:

- [`proto/keyvalue.proto`](proto/keyvalue.proto) - KeyValueDB service
- [`proto/cas.proto`](proto/cas.proto) - CASDBService

### Key Finding

**GetValueResponse returns the complete artifact data inline:**

```protobuf
message GetValueResponse {
  bool found = 1;
  bytes value = 2;  // Full artifact data, not a reference
}
```

This means:
- ✅ No separate Load/Get/Fetch operation exists
- ✅ Artifacts are transferred inline in GetValue responses
- ✅ Protocol is a simple key-value store with complete data transfer

## Cache Keys 🔐

Cache keys are content hashes derived from:

- Source file contents
- Compiler flags and arguments
- SDK version
- Module dependencies
- Build configuration
- Toolchain version

**Format:** Base64-encoded binary hash

**Example:** `0~YWoYNXXwg7v_Gpj7EqwaHJeXMY6Q0FSYANeEC3z_Laeez9xEdOC9TWkHvdglkVr5U8HVuYxo2G9nK11Cl9N9xQ==`

**Structure:**
- Prefix: `0~` (schema version identifier)
- Hash: Base64-encoded SHA-512 or similar cryptographic hash

## CAS IDs 🆔

Content-Addressable Storage IDs are cryptographic hashes of artifact data.

**Properties:**
- Deterministic: same content always produces the same ID
- Collision-resistant: uses SHA-256 or stronger
- Used for deduplication on the server side
- Enables integrity verification

## Implementation Requirements 🏗️

### Server Requirements

To implement a compatible CAS server:

**1. gRPC Server**
- HTTP/2 support
- Unix socket support
- Handle concurrent streams efficiently

**2. Service Implementation**
- `KeyValueDB.GetValue` - Return full artifacts inline
- `CASDBService.Save` - Store artifacts with CAS ID

**3. Storage Backend**
- Key-value store: cache keys → artifact data
- Content-addressable storage for deduplication
- Support for artifacts up to several MB

**4. Performance Considerations**
- Low-latency GetValue responses (artifacts sent inline)
- Efficient concurrent upload handling
- Content deduplication using CAS IDs

### Client Configuration

Configure Xcode to use the CAS server:

```bash
xcodebuild \
  -project MyProject.xcodeproj \
  -scheme MyScheme \
  COMPILATION_CACHE_REMOTE_SERVICE_PATH=/path/to/cas.sock \
  COMPILATION_CACHE_ENABLE_PLUGIN=YES \
  SWIFT_ENABLE_COMPILE_CACHE=YES \
  CLANG_ENABLE_COMPILE_CACHE=YES
```

## Error Handling ⚠️

### gRPC Status Codes

| Code | Status        | Meaning                      |
| ---- | ------------- | ---------------------------- |
| `0`  | OK            | Success                      |
| `2`  | UNKNOWN       | Server error                 |
| `5`  | NOT_FOUND     | Cache miss (artifact not found) |
| `14` | UNAVAILABLE   | Server unreachable           |

### Xcode Behavior

- **Server unavailable:** Falls back to local compilation silently
- **Cache miss:** Compiles locally and uploads artifact via Save
- **Upload failure:** Logs warning, continues build without caching
- **Malformed response:** Treats as cache miss

## Security Considerations 🔒

### Current Protocol

**Limitations:**
- No built-in authentication
- No encryption (plain HTTP/2)
- Designed for trusted local/network environments

**Mitigations:**
- Local Unix socket reduces network exposure
- Content integrity via CAS IDs (tamper detection)

### Production Recommendations

For production deployment:

- **Use TLS** for network transport
- **Implement authentication** (mTLS, API tokens, OAuth)
- **Validate CAS IDs** before serving artifacts
- **Rate limiting** to prevent abuse
- **Audit logging** for security monitoring
- **Network isolation** for cache servers

## Debugging 🐛

### Enable Xcode Diagnostics

```bash
COMPILATION_CACHE_ENABLE_DIAGNOSTIC_REMARKS=YES
```

This outputs detailed information:
- Cache hit/miss status for each artifact
- CAS connection status and errors
- Upload confirmations and failures
- Detailed error messages

### Common Issues

**No cache hits on second build:**
- Check local Xcode cache at `~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/`
- Clear it to force remote cache usage

**Server connection failures:**
- Verify socket path is absolute
- Check server logs for connection errors
- Ensure server is running before starting build

## Related Resources 📚

### Source Code

The protocol implementation can be found in:

- **Swift Compiler:** [apple/swift](https://github.com/apple/swift) - Compiler integration
- **Swift Driver:** [apple/swift-driver](https://github.com/apple/swift-driver) - Build orchestration
- **LLVM:** [llvm/llvm-project](https://github.com/llvm/llvm-project) - CAS support in Clang

### Search Hints

Useful search terms for exploring the implementation:

- `compilation_cache_service` - Protocol package name
- `CASDBService` - Storage service
- `KeyValueDB` - Lookup service
- `grpc-swift` - Swift gRPC client
- `LLVM CAS` - Content-addressable storage in LLVM

## Comparison with Other Protocols 📊

| Feature       | Xcode CAS     | Bazel REAPI    | ccache         |
| ------------- | ------------- | -------------- | -------------- |
| Protocol      | gRPC          | gRPC           | HTTP/REST      |
| Services      | 2 custom      | 6 standard     | N/A            |
| Compatible    | No            | No             | No             |
| Cache Type    | Inline CAS    | AC + CAS       | Hash-based     |
| Artifacts     | PCM, objects  | Actions, blobs | Objects only   |
| Fetch Method  | Inline return | Separate load  | Separate fetch |

**Key Difference:** Xcode CAS returns artifacts inline in GetValue responses, while Bazel REAPI uses separate ActionCache and ContentAddressableStorage with distinct fetch operations.

## Discovery Methodology 🔬

This protocol was reverse-engineered using:

**1. Traffic Capture**
- Custom HTTP/2 server using Python h2 library
- Unix socket interception with socat

**2. Protocol Analysis**
- Header inspection and gRPC path extraction
- Protobuf message capture and analysis
- Request/response pattern identification

**3. Experimental Verification**
- Implemented test server with artifact storage
- Confirmed GetValue returns full artifacts inline
- Validated by successful cache hits without connection failures

**4. Key Finding**
- Testing with actual artifact data confirmed no separate Load/Get/Fetch operation exists
- Cache hits succeeded without broken pipe errors when returning full artifacts

---

**Version:** 1.1
**Last Updated:** 2025-10-07
**Status:** Reverse-engineered and experimentally verified from Xcode 16
