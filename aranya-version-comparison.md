# Aranya vs Aranya Embedded: Capability Comparison

## Executive Summary

Aranya Embedded is **not a production-ready replacement** for the full Aranya daemon in its current state. It is a **demonstration/proof-of-concept** adaptation targeting ESP32-S3 microcontrollers that preserves the core sync protocol and policy engine but **strips out all real cryptography, dynamic onboarding, and standard networking**. For a mixed Linux+MCU environment that requires authenticated/encrypted commands and the onboarding, sync, and ephemeral command features you currently rely on, the full version remains necessary — or Aranya Embedded would require substantial development to close the gaps.

The sections below provide a detailed feature-by-feature comparison.

---

## 1. Architecture Overview

| Aspect | Aranya (Full) | Aranya Embedded |
|--------|---------------|-----------------|
| **Deployment model** | Client-daemon (separate processes) | Single monolithic binary |
| **Runtime** | Tokio async (multi-threaded) | Embassy async (single/dual-core MCU) |
| **Language** | Rust (std) | Rust (no_std + alloc) |
| **Target platforms** | Linux, macOS (POSIX required) | ESP32-S3 only |
| **Version** | 3.0.0 | Pre-release / demo |
| **License** | AGPL-3.0 | AGPL |
| **Crate count** | 7 core + external deps | 11 crates (demo-focused) |

### Architectural Implications for Replacement

The full version uses a **client-daemon architecture** where the daemon is a long-running process managing graph state, sync, and crypto, and clients connect via Unix domain sockets with encrypted IPC (tarpc + MessagePack). This cleanly separates application logic from Aranya internals.

Aranya Embedded collapses everything into a **single binary** with direct function calls. There is no IPC layer, no daemon process, and no client library. Your application code calls `daemon.create_team()` and `imp.call_action(...)` directly. This simplifies deployment on MCUs but means there's no process isolation and no C API equivalent.

---

## 2. Feature Comparison: What You Currently Use

### 2.1 Onboarding Process

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Team creation | `create_team()` with crypto key bundle | `create_team()` with fixed nonce `[0u8; 16]` |
| Member addition | `add_member()` with device public keys | **Not implemented** — peers manually configured via flash tool |
| Role assignment | Owner/Admin/Operator/Member hierarchy | **No role management** — demo policy has no roles |
| Peer discovery | Dynamic via sync protocol | **None** — static peer list burned to flash |
| Certificate exchange | Cryptographic key negotiation | **None** — NullEnvelope, no key exchange |
| Key bundle generation | `aranya-keygen` tool | Hardcoded `NULL_KEY` (32 zero bytes) |

**Gap Assessment: CRITICAL**

Your current onboarding workflow — adding members with key bundles, assigning roles, having policy enforce who can onboard whom — has **no equivalent in Aranya Embedded**. The embedded version requires each node to be manually configured with a static peer list using the `aranya-embedded-config` CLI tool, which writes addresses directly to flash. There is no dynamic member addition, no role hierarchy, and no cryptographic identity verification.

### 2.2 State Sync Across Endpoints

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Sync protocol | QUIC-based with PSK auth | Custom protocol over IR/ESP-NOW |
| Transport encryption | TLS/PSK per team | **None** — plaintext broadcast |
| Connection management | Connection pooling, auto-reconnect | Broadcast flood, no connections |
| Sync trigger | Configurable interval + on-demand | 500ms periodic Hello broadcast |
| Error correction | Standard QUIC reliability | RaptorQ fountain codes (lossy-tolerant) |
| Message format | postcard binary over QUIC streams | postcard + RaptorQ chunks + CRC-16 |
| Peer tracking | Shared connection map | PeerCache (which commands each peer has) |
| Stall handling | QUIC timeout/retry | 8-second stall timeout, resets session |
| Max message size | `MAX_SYNC_MESSAGE_SIZE` constant | Same constant (from aranya-runtime) |

**Gap Assessment: MODERATE-TO-HIGH**

The core sync **algorithm** (CRDT-based linear merge) is preserved — both versions use `aranya-runtime`'s `SyncRequester`/`SyncResponder` and the same command DAG structure. However, the **transport layer is completely different**:

- Full version: QUIC over TCP/IP with TLS/PSK encryption, connection pooling, and standard networking.
- Embedded: Custom packet protocol over IR or ESP-NOW (broadcast only), with RaptorQ for loss tolerance but **zero encryption or authentication**.

For your mixed environment with TCP/IP and serial/bus networks, neither the full version's QUIC-only transport nor the embedded version's IR/ESP-NOW transport may be ideal. The embedded version's `NetworkInterface` trait is at least abstractable to other transports, but no TCP/IP or serial implementations exist today.

### 2.3 Ephemeral Commands & Policy Enforcement

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Policy language | Full DSL with 23 effect types | Same DSL, but simplified policies |
| Policy compilation | Build-time or runtime | **Build-time only** (rkyv serialized binary) |
| Policy VM | `aranya-policy-vm` | Same `aranya-policy-vm` |
| Ephemeral sessions | `session_new()` / `session_receive()` | **Not implemented** |
| Command sealing | Real cryptographic signatures | `NullEnvelope` — SHA256 hash, signature = `b"LOL"` |
| Command validation | Full crypto verification on open | **No validation** — open always succeeds |
| Role-based enforcement | Owner > Admin > Operator > Member | **No roles in demo policy** |
| Effect types | 23 types (admin, channel, query) | 2-3 types (demo: LedColorChanged; chat: MessageReceived, AmbientColorChanged) |
| Fact storage | Full fact database | Basic facts (e.g., `CurrentColor[]`) |
| Labels | Create, assign, revoke labels | **Not implemented** |
| Channel creation | AFC + AQC channel policies | **Not implemented** |

**Gap Assessment: CRITICAL (for ephemeral commands), MODERATE (for basic policy)**

The **policy engine itself is the same** — `aranya-policy-vm` runs in both environments, and the DSL compiler is identical. The embedded version compiles policies at build time and embeds them as binary blobs, which is actually a reasonable approach for constrained devices.

However, **ephemeral sessions are not implemented** in the embedded version. The `session_new()` / `session_receive()` API that you use for policy enforcement without persisting to the DAG simply doesn't exist. All commands in the embedded version are persisted to the graph.

The demo policies are also trivially simple (set LED color, send chat messages) and lack the role-based access control, label management, and channel creation logic present in the full version's policy. That said, the policy engine *could* run a more complex policy — the limitation is in the surrounding infrastructure (no real crypto envelope, no role management in the daemon code), not the VM itself.

---

## 3. Feature Comparison: Other Capabilities

### 3.1 Cryptography

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Envelope implementation | Real AEAD encryption + signatures | `NullEnvelope` — hash only, dummy signature |
| Key storage | Persistent file-based keystore with key wrapping | In-memory `MemStore` (lost on reboot) |
| Key generation | `aranya-keygen` tool | Hardcoded 32 zero bytes |
| Identity keys | SHA-based device ID from public key | No real identity keys |
| Signing keys | ECDSA/EdDSA signatures | Not implemented |
| Encryption keys | AEAD for channel data | Not implemented |
| TLS/PSK | Per-team PSK with rustls | Not implemented |
| Zeroization | Keys zeroized on drop | Not implemented |
| RNG | OS-provided | ESP32 hardware RNG |

**Gap Assessment: CRITICAL**

This is the **single largest gap**. The embedded version has **no real cryptography**. Commands are "sealed" with a SHA256 hash and a literal `b"LOL"` signature. There is no encryption in transit, no peer authentication, and no key management. The code is littered with TODOs acknowledging this:

- `"TODO(chip): use actual keys"`
- `"TODO(chip): use an actual signature"`
- `"Temporarily fix the nonce for demo purposes"`

Given your requirement for essential cryptographic authentication and encryption, this alone makes the embedded version unsuitable as a drop-in replacement.

### 3.2 Networking & Transport

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Primary transport | QUIC (s2n-quic with PSK) | IR / ESP-NOW (broadcast) |
| TCP/IP support | Yes (QUIC over UDP/IP) | WiFi backend exists but non-functional |
| Serial/UART | Not supported | IR transceiver over UART |
| IPC | Unix domain sockets (tarpc) | None (direct function calls) |
| AFC (Fast Channels) | Shared memory ring buffer | Not implemented |
| AQC (QUIC Channels) | QUIC data streams (bidi + uni) | Not implemented |
| Transport abstraction | Hardcoded to QUIC | `NetworkInterface` trait (abstractable) |
| Connection model | Connection-oriented (pooled) | Connectionless (broadcast) |

**Gap Assessment: HIGH**

For your mixed network environment (TCP/IP + serial/bus), neither version provides out-of-the-box support for all your transports. The full version is locked to QUIC/TCP/IP. The embedded version at least has a `NetworkInterface` trait that could be implemented for other transports (serial, CAN, etc.), but the existing implementations are IR and ESP-NOW only.

### 3.3 Storage

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Storage backend | POSIX filesystem (`libc::FileManager`) | ESP32 flash partitions / SD card |
| Provider | `LinearStorageProvider<FileManager>` | `LinearStorageProvider<EspPartitionIoManager>` |
| Persistence | Yes (survives restarts) | Yes (flash), but keys are in-memory only |
| Configuration | TOML config files | Binary parameter block in flash |
| Graph storage | `{state_dir}/storage/` directory | Named "graph" partition in flash |

**Gap Assessment: LOW**

Both use the same `LinearStorageProvider` from `aranya-runtime` — just with different I/O backends. The storage layer is actually well-abstracted and is one of the most portable parts of the system.

### 3.4 API & Integration Surface

| Capability | Aranya (Full) | Aranya Embedded |
|------------|---------------|-----------------|
| Rust client library | `aranya-client` crate | Direct `Daemon` struct calls |
| C API | `aranya-client-capi` (cbindgen) | None |
| RPC interface | tarpc-based `DaemonApi` trait | None |
| Effect handling | `EffectHandler` with async processing | `Sink` trait (synchronous consume) |
| Metrics | Prometheus/Datadog/TCP export | None |
| Logging | tracing + tracing-subscriber | `defmt` / `log` (embedded logging) |

**Gap Assessment: HIGH**

The full version provides a well-defined client library, a C FFI for non-Rust integrations, and a formal RPC API. The embedded version has none of these — your application code must directly call into the `Daemon` struct. If your flight software has components in C/C++ or other languages, the embedded version offers no integration path today.

---

## 4. What Aranya Embedded Does Well

Despite the gaps, the embedded version demonstrates some valuable properties:

1. **Core sync protocol is portable**: The `aranya-runtime` sync algorithm (CRDT, `SyncRequester`/`SyncResponder`) works identically in both versions. The graph merge logic is sound and transport-agnostic.

2. **Policy VM runs in no_std**: The policy compiler and VM work in constrained environments. Build-time compilation to rkyv binary blobs is an efficient approach.

3. **Transport abstraction exists**: The `NetworkInterface` trait is a clean abstraction that could be implemented for TCP, serial, CAN, or any other transport.

4. **Small footprint**: Runs in 96 KB heap + PSRAM on an ESP32-S3 at 240 MHz. The release profile optimizes for size (`opt-level = "z"`, LTO, strip).

5. **Loss-tolerant sync**: RaptorQ fountain codes make the sync protocol robust over unreliable links — useful for radio, IR, or noisy serial buses.

---

## 5. Gap Summary for Your Use Case

Given your requirements (replace full version, essential crypto, mixed Linux+MCU environment, multiple network types), here is the gap analysis ranked by severity:

| Gap | Severity | Effort to Close |
|-----|----------|-----------------|
| No real cryptography (NullEnvelope) | **BLOCKING** | High — requires implementing a real envelope with signing, encryption, and key management for no_std |
| No ephemeral sessions | **BLOCKING** | Medium — the runtime supports it; needs daemon-level API |
| No onboarding / member addition | **BLOCKING** | High — requires role management, key exchange, and policy enforcement |
| No TCP/IP transport | **HIGH** | Medium — `NetworkInterface` trait exists; needs TCP/UDP implementation |
| No C API or RPC interface | **HIGH** | Medium-High — needs IPC design for embedded context |
| No role-based access control in daemon | **HIGH** | Medium — policy VM supports it; daemon code needs role tracking |
| No AFC/AQC channels | **MODERATE** | High — fundamentally different transport model |
| No label management | **MODERATE** | Low-Medium — policy supports it; needs daemon wiring |
| No key persistence | **MODERATE** | Medium — needs persistent keystore for flash/filesystem |
| No metrics/observability | **LOW** | Low — nice to have, not blocking |

---

## 6. Recommendation

**Aranya Embedded is not viable as a replacement for the full Aranya version** for your use case. The three features you rely on — onboarding, state sync (with authentication), and ephemeral commands — all have critical gaps in the embedded version:

- **Onboarding**: No dynamic member addition, no role management, no key exchange.
- **State sync**: Core algorithm works, but no transport encryption and no TCP/IP backend.
- **Ephemeral commands**: Not implemented at all.

### If you still want a lighter-weight solution, consider these paths:

1. **Stick with the full version** on your Linux nodes, and use Aranya Embedded (with development investment) only for the most constrained MCU nodes that need to participate in sync. This hybrid approach plays to each version's strengths.

2. **Invest in closing the crypto gap** in Aranya Embedded. The `aranya-crypto` crate already has some no_std-compatible components. Implementing a real `Envelope` (replacing `NullEnvelope`) and persistent keystore would be the highest-impact improvement.

3. **Fork the full version for size optimization** rather than trying to build up from the embedded version. The full version's release profile already optimizes for size (`opt-level = "z"`, LTO, strip). A stripped-down daemon without AFC/AQC/metrics might run on a resource-constrained Linux system.

---

## Appendix: Shared Dependencies (Same Crate, Both Versions)

These crates from the Aranya ecosystem are used by **both** versions, confirming protocol compatibility at the algorithm level:

| Crate | Full Version | Embedded Version | Role |
|-------|-------------|-----------------|------|
| `aranya-runtime` | 0.14.0 | git dep | DAG/graph state, sync protocol, storage providers |
| `aranya-crypto` | 0.10.0 | git dep | Crypto primitives, keystore interface |
| `aranya-policy-vm` | 0.13.0 | git dep | Policy bytecode execution |
| `aranya-policy-compiler` | 0.13.0 | build dep | Policy DSL compilation |
| `postcard` | 1.x | 1.0.10 | Binary serialization |
