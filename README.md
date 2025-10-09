# Xcode CAS Protocol Documentation 🚀

## Overview

This document describes the **Compilation Cache Service Protocol** used by Xcode to communicate with remote caching servers. The protocol enables distributed caching of Swift and Clang compilation artifacts.

| Property | Value |
|----------|-------|
| **Protocol Type** | gRPC over HTTP/2 |
| **Transport** | Unix Domain Socket or TCP |
| **Client Library** | `grpc-swift-nio/1.14.1` |
| **Protocol Definitions** | [`proto/`](proto/) |

---

## Services 🛠️

The protocol consists of two gRPC services:

### 1. KeyValueDB Service 🔑

> Handles cache lookups and returns complete artifacts inline

| Property | Value |
|----------|-------|
| **Package** | `compilation_cache_service.keyvalue.v1` |
| **Service** | `KeyValueDB` |
| **Proto** | [`proto/keyvalue.proto`](proto/keyvalue.proto) |

#### GetValue Method

Queries the cache for a specific key and retrieves the complete cached artifact.

**RPC Path:**

```
/compilation_cache_service.keyvalue.v1.KeyValueDB/GetValue
```

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `key` | bytes | Cache key derived from source content, compiler flags, SDK version, module dependencies, and build configuration |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `found` | bool | Whether the artifact exists in the cache |
| `value` | bytes | **The complete artifact data** (only present if `found=true`) |

> [!IMPORTANT]
> The response contains the **full artifact data inline**, not a reference. There is no separate fetch operation.

**Usage Pattern:**

- ✅ Called at the beginning of compilation
- ✅ Multiple parallel requests (one per artifact)
- ✅ If `found=true`, Xcode uses the returned artifact directly
- ✅ If `found=false`, Xcode compiles and uploads via `Save`

---

### 2. CASDBService 💾

> Handles storing compilation artifacts

| Property | Value |
|----------|-------|
| **Package** | `compilation_cache_service.cas.v1` |
| **Service** | `CASDBService` |
| **Proto** | [`proto/cas.proto`](proto/cas.proto) |

#### Save Method

Stores a compilation artifact in the Content-Addressable Storage (CAS).

**RPC Path:**

```
/compilation_cache_service.cas.v1.CASDBService/Save
```

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `cas_id` | bytes | Content hash of the artifact data |
| `data` | bytes | The complete artifact data |
| `type` | string | Artifact type (e.g., "pcm", "o", "metadata") |
| `metadata` | map<string, string> | Optional metadata |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `cas_id` | bytes | Confirmed CAS ID |
| `success` | bool | Whether the save succeeded |
| `message` | string | Optional status message |

**Usage Pattern:**

- ✅ Called during and after compilation
- ✅ Multiple concurrent uploads
- ✅ Stores precompiled modules (.pcm), objects (.o), and metadata

---

## Protocol Details 📋

### Connection

| Property | Value |
|----------|-------|
| **Transport** | Unix Domain Socket only |
| **Protocol** | HTTP/2 with gRPC framing |
| **Socket Path Env Var** | `COMPILATION_CACHE_REMOTE_SERVICE_PATH` |

> [!IMPORTANT]
> Xcode's CAS protocol **only supports Unix domain sockets**. HTTP/TCP URLs (like `http://127.0.0.1:9999`) are not supported and will be silently ignored.

**Example Configuration:**

```bash
COMPILATION_CACHE_REMOTE_SERVICE_PATH=/path/to/cas.sock
```

---

### Request Headers

Every gRPC request includes these HTTP/2 headers:

| Header | Value |
|--------|-------|
| `:method` | `POST` |
| `:path` | `/compilation_cache_service.{package}.{version}.{Service}/{Method}` |
| `:authority` | `localhost` |
| `:scheme` | `http` |
| `content-type` | `application/grpc` |
| `te` | `trailers` |
| `user-agent` | `grpc-swift-nio/1.14.1` |

---

### Response Format

Successful responses include:

| Header | Value |
|--------|-------|
| `:status` | `200` |
| `content-type` | `application/grpc` |
| `grpc-status` | `0` |
| `grpc-message` | `OK` |

---

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

---

## ⚠️ Critical Limitation: Write-Only Remote Cache

> [!WARNING]
> **Xcode's remote CAS protocol implementation is asymmetric and write-only.** This significantly limits its usefulness for distributed caching.

### What Xcode Actually Does

Through extensive testing and protocol analysis, we've discovered that Xcode only calls **2 out of 4** protocol methods:

| Method | Service | Called by Xcode? | Purpose |
|--------|---------|------------------|---------|
| **GetValue** | KeyValueDB | ✅ YES | Query remote action cache (always returns NOT_FOUND) |
| **PutValue** | KeyValueDB | ❌ **NEVER** | Write action cache entries to remote |
| **Save** | CASDBService | ✅ YES | Store compiled artifacts (Mach-O binaries) |
| **Load** | CASDBService | ❌ **NEVER** | Retrieve artifacts from remote |

### Why This Happens

**Symbol analysis of libToolchainCASPlugin.dylib reveals incomplete implementation:**

Analysis of `/Applications/Xcode.app/Contents/Developer/usr/lib/libToolchainCASPlugin.dylib` shows:

```bash
# All 4 gRPC methods ARE defined in the protocol:
/compilation_cache_service.cas.v1.CASDBService/Load
/compilation_cache_service.cas.v1.CASDBService/Save
/compilation_cache_service.keyvalue.v1.KeyValueDB/GetValue
/compilation_cache_service.keyvalue.v1.KeyValueDB/PutValue

# But RemoteCompilationCache class only implements:
RemoteCompilationCache.remoteCachePut(key:value:) → EventLoopFuture
RemoteCompilationCache.remoteCASSave() → EventLoopFuture

# Missing implementations:
❌ No remoteCacheGet method (would call GetValue)
❌ No remoteCASLoad method (would call Load)
```

**Xcode only maintains the action cache (v3.actions) locally:**

1. During compilation, Xcode queries the remote with `GetValue(cache_key)`
2. The remote server has NO action cache entries (Xcode never calls `PutValue`)
3. `GetValue` always returns `NOT_FOUND` or empty responses
4. Xcode compiles locally and calls `Save` to store artifacts remotely
5. Without cache hits from `GetValue`, there's no reason to call `Load`

**Result:** The remote server stores CAS objects but has no way to tell Xcode which artifacts exist for which cache keys.

### Experimental Evidence

```bash
# Build 1 (clean cache)
$ rm -rf ~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/
$ xcodebuild ...
→ 60× GetValue (all MISS)
→ 73× Save
→ 0× PutValue  ❌ Never called
→ 0× Load      ❌ Never called

# Build 2 (with local cache)
$ xcodebuild ...
# NO remote server calls at all!
# All cache hits come from local v3.actions file

# Build 3 (with SWIFT_ENABLE_EXPLICIT_MODULES=YES)
$ rm -rf ~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/
$ xcodebuild ... SWIFT_ENABLE_EXPLICIT_MODULES=YES
→ 60× GetValue (all MISS)
→ 71× Save
→ 0× PutValue  ❌ Still never called
→ 0× Load      ❌ Still never called
```

### How Bitrise Works Around This

Bitrise doesn't rely on Xcode's `PutValue`/`Load` methods. Instead, they use a **filesystem-based approach**:

```bash
# Before build: Restore local cache from remote
bitrise-build-cache cache-restore
# Downloads ~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/
# Includes v3.actions (action cache) + v8.*.leaf files (CAS objects)

# During build: Xcode works with LOCAL cache
xcodebuild ...
# GetValue/Save may be called but are mostly unused
# Action cache hits come from local v3.actions file

# After build: Push updated local cache to remote
bitrise-build-cache cache-push
# Uploads updated v3.actions + new v8.*.leaf files
```

**The gRPC protocol is only used for storing artifacts during builds, not for distributed cache lookup.**

### Implications for Server Implementations

If you're implementing a remote CAS server:

1. **Don't expect `PutValue` calls** - Xcode never writes action cache entries to remote
2. **`GetValue` will always miss** - The remote has no action cache mappings
3. **`Save` stores artifacts** - But without action cache entries, they're unretrievable
4. **`Load` is never called** - Xcode doesn't fetch from remote CAS

**Recommendation:** Implement a Bitrise-style filesystem sync approach instead of relying on the gRPC protocol for distributed caching.

---

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

Cache keys are content hashes derived from multiple inputs to ensure uniqueness:

| Input | Description |
|-------|-------------|
| Source file contents | The actual source code |
| Compiler flags | Arguments passed to the compiler |
| SDK version | Version of the development SDK |
| Module dependencies | Dependencies and their versions |
| Build configuration | Debug, Release, etc. |
| Toolchain version | Version of Xcode/compiler |

**Format:** Base64-encoded binary hash

**Example:**

```
0~YWoYNXXwg7v_Gpj7EqwaHJeXMY6Q0FSYANeEC3z_Laeez9xEdOC9TWkHvdglkVr5U8HVuYxo2G9nK11Cl9N9xQ==
```

**Structure:**

| Component | Description |
|-----------|-------------|
| `0~` | Schema version identifier |
| Hash data | Base64-encoded SHA-512 or similar |

---

## CAS IDs 🆔

Content-Addressable Storage IDs are cryptographic hashes of artifact data.

| Property | Description |
|----------|-------------|
| **Deterministic** | Same content always produces the same ID |
| **Collision-resistant** | Uses SHA-256 or stronger |
| **Deduplication** | Used for server-side deduplication |
| **Integrity** | Enables tamper detection |

---

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
- Request/response logging to JSON for analysis

**2. Protocol Analysis**
- Header inspection and gRPC path extraction
- Protobuf message capture and manual decoding
- Request/response pattern identification
- Symbol analysis of `/Applications/Xcode.app/Contents/Developer/usr/lib/libToolchainCASPlugin.dylib`

**3. Experimental Verification**
- Implemented full gRPC server with all 4 methods (GetValue, PutValue, Save, Load)
- Tested with clean builds to measure method call patterns
- Confirmed GetValue returns full artifacts inline
- Validated by successful cache hits without connection failures

**4. Critical Discovery: Asymmetric Protocol Implementation**

Through systematic testing, we discovered Xcode's implementation is incomplete:

```bash
# Test 1: Clean cache build
$ rm -rf ~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/
$ xcodebuild ... (with remote server)
Result: 60× GetValue, 73× Save, 0× PutValue, 0× Load

# Test 2: Rebuild with local cache
$ xcodebuild ...
Result: 0 remote calls (100% cache hits from local v3.actions file)

# Test 3: Delete local cache again
$ rm -rf ~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/
$ xcodebuild ...
Result: Same as Test 1 - still 0× PutValue, 0× Load
```

**Conclusion:** Xcode never calls `PutValue` or `Load`, making the remote cache write-only.

**5. Bitrise Analysis**
- Reverse-engineered Bitrise's `bitrise-build-cache` binary (Go)
- Discovered they implement filesystem sync, not pure gRPC caching
- Their `cache-restore` and `cache-push` commands sync the entire local cache directory

**6. Local Cache Investigation**
- Analyzed `~/Library/Developer/Xcode/DerivedData/CompilationCache.noindex/plugin/v1.1/`
- Found `v3.actions` (action cache) and `v8.*.leaf` (CAS objects)
- Confirmed Xcode only writes action cache mappings locally

---

**Version:** 2.0
**Last Updated:** 2025-10-09
**Status:** Reverse-engineered and experimentally verified from Xcode 16
**Major Update:** Discovered protocol asymmetry - Xcode never calls PutValue/Load
