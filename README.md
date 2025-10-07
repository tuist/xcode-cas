# Xcode CAS Protocol Documentation

## Overview

This document describes the **Compilation Cache Service Protocol** used by Xcode to communicate with remote caching servers. The protocol enables distributed caching of Swift and Clang compilation artifacts.

**Protocol Type:** gRPC over HTTP/2
**Transport:** Unix Domain Socket or TCP
**Client Library:** `grpc-swift-nio/1.14.1`

## Services

The protocol consists of two gRPC services:

### 1. KeyValueDB Service

**Package:** `compilation_cache_service.keyvalue.v1`
**Service:** `KeyValueDB`

#### Methods

##### GetValue
Queries the cache for a specific key to check if a cached artifact exists.

**RPC Path:**
```
/compilation_cache_service.keyvalue.v1.KeyValueDB/GetValue
```

**Request:** Cache key (hash)
**Response:** Cache hit/miss status with optional CAS ID
**Message Size:** ~72-163 bytes

**Usage Pattern:**
- Called at the beginning of compilation
- Multiple parallel requests (typically 7+)
- Determines which artifacts need to be recompiled

---

### 2. CASDBService

**Package:** `compilation_cache_service.cas.v1`
**Service:** `CASDBService`

#### Methods

##### Save
Stores a compilation artifact in the Content-Addressable Storage (CAS).

**RPC Path:**
```
/compilation_cache_service.cas.v1.CASDBService/Save
```

**Request:** CAS ID + artifact data
**Response:** Confirmation with CAS ID
**Message Size:** 163 bytes - 42 KB (varies by artifact type)

**Usage Pattern:**
- Called during and after compilation
- Multiple concurrent uploads (60+ per build)
- Stores precompiled modules (.pcm), objects (.o), and metadata

## Protocol Details

### Connection

**Transport:** Unix Domain Socket (preferred) or TCP
**Protocol:** HTTP/2 with gRPC framing
**Socket Path:** Configurable via `COMPILATION_CACHE_REMOTE_SERVICE_PATH`

**Example:**
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

## Message Flow

### Typical Build Sequence

1. **Initialization**
   - Xcode establishes HTTP/2 connection
   - SETTINGS frame exchange

2. **Cache Query Phase**
   ```
   → GetValue(key1)  \
   → GetValue(key2)   | Parallel
   → GetValue(key3)   | queries
   ...                |
   → GetValue(keyN)  /

   ← Response(hit/miss)
   ```

3. **Compilation Phase**
   - Artifacts that missed cache are compiled

4. **Upload Phase**
   ```
   → Save(artifact1)  \
   → Save(artifact2)   | Concurrent
   → Save(artifact3)   | uploads
   ...                 |
   → Save(artifactN)  /

   ← Confirmation
   ```

## Artifact Types

The protocol handles these compilation artifacts:

| Type | Description | Typical Size |
|------|-------------|--------------|
| **Metadata** | Cache keys, mappings | ~163 bytes |
| **PCM Files** | Precompiled Modules | ~3-10 KB |
| **Object Files** | Compiled objects (.o) | ~40-42 KB |
| **Diagnostics** | Compiler output | Variable |

## Protocol Buffers

### Inferred Schema

Based on observed traffic, the protocol uses these structures:

```protobuf
syntax = "proto3";

package compilation_cache_service.keyvalue.v1;

service KeyValueDB {
  rpc GetValue(GetValueRequest) returns (GetValueResponse);
}

message GetValueRequest {
  bytes key = 1;  // SHA-256 or similar hash
}

message GetValueResponse {
  bool found = 1;
  bytes cas_id = 2;  // Content-addressable storage ID (if found)
}
```

```protobuf
syntax = "proto3";

package compilation_cache_service.cas.v1;

service CASDBService {
  rpc Save(SaveRequest) returns (SaveResponse);
}

message SaveRequest {
  bytes cas_id = 1;      // Content hash of the data
  bytes data = 2;        // The artifact data itself
  string type = 3;       // Artifact type (e.g., "pcm", "o")
  map<string, string> metadata = 4;  // Optional metadata
}

message SaveResponse {
  bytes cas_id = 1;      // Confirmed CAS ID
  bool success = 2;
  string message = 3;    // Optional status message
}
```

## Cache Keys

Cache keys are content hashes derived from:
- Source file contents
- Compiler flags
- SDK version
- Module dependencies
- Build configuration

**Format:** Base64-encoded binary hash (64 bytes)
**Example:** `0~YWoYNXXwg7v_Gpj7EqwaHJeXMY6Q0FSYANeEC3z_Laeez9xEdOC9TWkHvdglkVr5U8HVuYxo2G9nK11Cl9N9xQ==`

**Structure:**
- Prefix: `0~` (schema version)
- Hash: Base64-encoded SHA-512 or similar

## CAS IDs

Content-Addressable Storage IDs are hashes of the artifact data.

**Properties:**
- Deterministic (same content = same ID)
- Collision-resistant
- Used for deduplication
- Used for integrity verification

## Implementation Requirements

### Server Requirements

To implement a compatible CAS server:

1. **gRPC Server**
   - HTTP/2 support
   - Unix socket support
   - Handle concurrent streams

2. **Services**
   - Implement `KeyValueDB.GetValue`
   - Implement `CASDBService.Save`

3. **Storage**
   - Key-value store for cache keys → CAS IDs
   - Content-addressable storage for artifacts
   - Support for objects up to ~50 KB

4. **Performance**
   - Handle 60+ concurrent uploads
   - Low-latency lookups (< 10ms)
   - Efficient deduplication

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

## Performance Characteristics

Based on observed traffic:

- **Queries per build:** 7-10 GetValue calls
- **Uploads per build:** 60+ Save calls
- **Concurrent connections:** 1-2
- **Concurrent streams:** Up to 67 per connection
- **Total data transferred:** ~1-2 MB per build (incremental)

## Error Handling

### gRPC Status Codes

| Code | Status | Meaning |
|------|--------|---------|
| `0` | OK | Success |
| `2` | UNKNOWN | Server error |
| `5` | NOT_FOUND | Cache miss |
| `14` | UNAVAILABLE | Server unreachable |

### Xcode Behavior

- **Server unavailable:** Falls back to local compilation (silent)
- **Cache miss:** Compiles and uploads artifact
- **Upload failure:** Logs warning, continues build
- **Timeout:** 2 seconds per request (configurable)

## Security Considerations

1. **No built-in authentication** - Protocol has no auth headers
2. **No encryption** - Uses plain HTTP/2 (not HTTPS)
3. **Local socket preferred** - Reduces network exposure
4. **Content integrity** - CAS IDs provide tamper detection

### Recommendations

For production deployment:

- Use TLS for network transport
- Implement authentication (mTLS, tokens, etc.)
- Validate CAS IDs before serving artifacts
- Rate limit to prevent abuse
- Monitor for unusual patterns

## Debugging

### Enable Xcode Diagnostics

```bash
COMPILATION_CACHE_ENABLE_DIAGNOSTIC_REMARKS=YES
```

This outputs:
- Cache hits/misses
- CAS connection status
- Upload confirmations
- Error messages

### Traffic Capture

Use our protocol discovery server:

```bash
python3 cas-protocol-server.py
```

This logs:
- All gRPC requests
- Complete headers
- Message contents (hex dump)
- Stream lifecycle

## Related Resources

### Source Code

The protocol implementation is in:
- **Swift Compiler:** `apple/swift` repository
- **Swift Driver:** `apple/swift-driver` repository
- **LLVM:** `llvm/llvm-project` (CAS support)

### Search Hints

Look for:
- `compilation_cache_service`
- `CASDBService`
- `KeyValueDB`
- `grpc-swift` integration
- LLVM CAS APIs

## Comparison with Other Protocols

| Feature | Xcode CAS | Bazel REAPI | ccache |
|---------|-----------|-------------|--------|
| Protocol | gRPC | gRPC | HTTP/REST |
| Services | 2 custom | 6 standard | N/A |
| Compatible | No | No | No |
| Cache Type | CAS | AC + CAS | Hash-based |
| Artifacts | PCM, objects | Actions, blobs | Objects only |

## Appendix: Discovery Methodology

This protocol was reverse-engineered using:

1. **HTTP/2 capture** with socat
2. **Custom gRPC server** (Python h2 library)
3. **Traffic analysis** of 67 complete requests
4. **Header inspection** and path extraction
5. **Message size profiling**

Files generated during discovery:
- `protocol-discovery.log` - 50,205 lines of detailed logs
- `stream-*-message.bin` - 67 captured protobuf messages
- Complete HTTP/2 frame analysis

---

**Version:** 1.0
**Last Updated:** 2025-10-07
**Status:** Reverse-engineered from Xcode 26 (Xcode_26.app)
