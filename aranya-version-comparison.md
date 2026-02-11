# Aranya vs Aranya Embedded: Capability Comparison (Revised)

## Executive Summary

With relaxed requirements (manual onboarding acceptable, no ephemeral commands needed, transport-agnostic sync desired), the gap between Aranya and Aranya Embedded narrows significantly. **Two real blockers remain**: (1) the cryptographic envelope and (2) interoperability between the two versions. The embedded version's `NetworkInterface` trait is actually **better positioned** for transport-agnostic sync than the full version's hardcoded QUIC transport. The policy VM is identical in both. This revised analysis focuses on the minimal gap and what it would take to close it.

---

## 1. Revised Requirements & What's No Longer Blocking

| Original Concern | Status | Why |
|-----------------|--------|-----|
| Dynamic onboarding | **Relaxed** | Manual config at build/deploy time is acceptable. Embedded already supports this via `aranya-embedded-config`. |
| Ephemeral commands | **Relaxed** | Graph-persisted commands only. Both versions support this. |
| Transport flexibility | **Embedded is better** | Embedded has the `NetworkInterface` trait; full version has QUIC hardcoded. |
| Real crypto (auth + encryption) | **Still required** | NullEnvelope is a blocker. |
| Interoperability (full + embedded) | **Still required** | Version mismatch and message format differences exist. |
| Direct data channels (AFC/AQC) | **Optional** | Not blocking. |
| C API | **Optional** | Not blocking; alternative integration approaches exist. |

---

## 2. What Works Today (No Gap)

### 2.1 Policy Engine — Identical

The policy VM (`aranya-policy-vm`) is the **same crate** in both versions. The DSL compiler (`aranya-policy-compiler`) is also identical. Your existing policies will compile and run on embedded without modification.

| Aspect | Full | Embedded | Compatible? |
|--------|------|----------|-------------|
| Policy DSL | `aranya-policy-lang` | Same | Yes |
| Policy compiler | `aranya-policy-compiler` | Same | Yes |
| Policy VM | `aranya-policy-vm` | Same | Yes |
| Fact storage | Supported | Supported | Yes |
| Effect emission | `EffectHandler` (async) | `Sink` trait (sync) | API differs, semantics same |
| Build-time compilation | Supported | Default approach | Yes |

The embedded version compiles policies at build time into rkyv-serialized binary blobs embedded in the firmware. This is actually a clean approach — no runtime parsing overhead, smaller binary, and the same policy bytecode executes on both platforms.

### 2.2 Core Sync Algorithm — Identical

Both versions use `aranya-runtime`'s `SyncRequester`/`SyncResponder` and the same CRDT-based linear merge algorithm. The command DAG structure, merge logic, and `PeerCache` tracking are shared.

| Aspect | Full | Embedded | Compatible? |
|--------|------|----------|-------------|
| CRDT merge algorithm | `aranya-runtime` | Same crate | Yes |
| `SyncRequester` | From runtime | Same type | Yes (if version-aligned) |
| `SyncResponder` | From runtime | Same type | Yes (if version-aligned) |
| `PeerCache` | Shared connection map | Per-peer BTreeMap | Functionally equivalent |
| `LinearStorageProvider` | File-based I/O | Flash/SD I/O | Same provider, different backends |

### 2.3 Storage Layer — Well Abstracted

Both use `LinearStorageProvider` from `aranya-runtime` with platform-specific I/O backends. This is one of the most portable parts of the system.

### 2.4 Manual Onboarding — Already Works in Embedded

The embedded version's manual configuration approach (`aranya-embedded-config` tool writing peer addresses to flash) aligns with your relaxed requirement. Team creation via `daemon.create_team()` works in both versions. The remaining gap is that the embedded version uses a fixed nonce and has no key material — but that's a crypto gap, not an onboarding gap.

---

## 3. Transport Agnosticism — Embedded is Better Positioned

This is a surprising finding: **Aranya Embedded is better positioned for transport-agnostic sync than the full version.**

### 3.1 The NetworkInterface Trait (Embedded)

The embedded version defines a clean, minimal trait:

```rust
trait NetworkInterface {
    type Addr: Copy + Display + Hash;
    const BROADCAST: Self::Addr;

    async fn send_message(&mut self, msg: Message<Self::Addr>) -> Result<(), NetworkError>;
    async fn recv_message(&mut self) -> Result<Message<Self::Addr>, NetworkError>;
    fn my_address(&self) -> Self::Addr;
}

struct Message<A> {
    sender: A,
    recipient: A,
    contents: Box<[u8]>,
}
```

This is **transport-agnostic by design**. The sync engine (`SyncEngine<N: NetworkInterface>`) treats the network as a message-oriented abstraction. It doesn't assume broadcast, point-to-point, reliable, or unreliable delivery.

### 3.2 The Full Version's QUIC Lock-In

The full version hardcodes QUIC (`s2n-quic` with PSK TLS) as the transport. There is **no transport abstraction trait** in the full version — QUIC is wired directly into the sync task. To use a different transport, you'd have to refactor the daemon's sync layer.

### 3.3 Implementing Your Target Transports

For each transport you listed, here's what a `NetworkInterface` implementation would involve:

| Transport | Framing Needed | Loss Tolerance | Addressing | Effort |
|-----------|---------------|----------------|------------|--------|
| **UDP** | Length prefix (4 bytes) | RaptorQ recommended | IP:port as Addr | Low |
| **Pub/Sub (e.g., Zenoh)** | None (message-oriented) | None (reliable) | Topic as Addr | Low |
| **CCSDS Cmd/Tlm** | CCSDS packet header | Depends on link | APID as Addr | Medium |
| **MAVLink** | MAVLink v2 framing | Depends on link | System/Component ID | Medium |
| **Serial/UART** | Magic + length + CRC | RaptorQ for noisy links | Static peer ID | Low-Medium |

**Key architectural insight**: RaptorQ encoding (fountain codes for loss tolerance) is part of the **network layer**, not the sync protocol. It's applied inside the `send_message()`/`recv_message()` implementations. For reliable transports (TCP, pub/sub, reliable serial), you skip RaptorQ entirely. For lossy links (UDP broadcast, radio, CCSDS over RF), you add it.

### 3.4 What the Sync Protocol Assumes About the Transport

The sync protocol needs:
- Complete message delivery (no partial messages) — transport must handle framing/reassembly
- Sender/recipient identity preserved
- Bidirectional communication (request triggers response)

The sync protocol does NOT assume:
- Ordering (handles out-of-order messages)
- Guaranteed delivery (handles lost messages via periodic Hello + retry)
- Encryption (currently no crypto in embedded; would need it)

---

## 4. Remaining Blockers

### 4.1 BLOCKER: Cryptographic Envelope (NullEnvelope → Real Crypto)

**This is the single largest gap.** The embedded version's `NullEnvelope`:
- Produces a SHA256 hash as the command ID
- Signs with the literal string `b"LOL"`
- Does not encrypt the payload
- Does not validate anything on `open()`

The full version uses HPKE (Hybrid Public Key Encryption) with:
- AES-GCM AEAD encryption of command payloads
- Ed25519/ECDSA signatures for authentication
- Sequence-number-based authenticated data (RFC 9180)
- Per-peer encryption contexts (SealCtx/OpenCtx)
- Persistent keystore with key wrapping and zeroization

**Can it be closed?** Yes. The core crypto primitives in `aranya-crypto` and `spideroak-crypto` **are `no_std`-compatible**:

| Primitive | Crate | `no_std`? |
|-----------|-------|-----------|
| AES-GCM (AEAD) | `aes-gcm` | Yes |
| Ed25519 (signing) | `ed25519-dalek` | Yes |
| HPKE (key encapsulation) | `spideroak-crypto` | Yes |
| SHA-2/SHA-3 (hashing) | `sha2`/`sha3-utils` | Yes |
| Zeroization | `zeroize` | Yes |
| Random | `getrandom` (ESP32 HW RNG) | Yes |

**What's NOT `no_std`:**
- `fs-keystore` (file-based keystore — depends on `std::fs`)
- `tls` feature (rustls — depends on `std`)
- `rustix` (POSIX syscalls — not needed for core crypto)

**Effort estimate to implement real envelope on embedded:**

| Component | Lines of Code | Effort |
|-----------|--------------|--------|
| Real `seal()` with AEAD encryption + Ed25519 signing | ~150 LOC | Low |
| Real `open()` with verification + decryption | ~120 LOC | Low |
| Persistent keystore for ESP32 NVS/flash | ~300 LOC | Medium |
| Key generation and exchange at config time | ~200 LOC | Medium |
| Integration with existing `Daemon` struct | ~200 LOC | Medium |
| **Total** | **~1000 LOC** | **Medium** |

The `aranya-crypto` crate can be used with features `["alloc", "clone-aead"]` (dropping `std`, `fs-keystore`, `tls`) to get the core crypto on embedded.

### 4.2 BLOCKER: Interoperability Between Full and Embedded

If you want full Aranya nodes and embedded nodes to sync with each other, three sub-gaps must be closed:

#### 4.2.1 aranya-runtime Version Mismatch

| | Full Aranya | Aranya Embedded |
|--|-------------|-----------------|
| `aranya-runtime` | **0.14.0** (crates.io) | **0.16.1** (git main) |
| `aranya-crypto` | **0.10.0** | **0.12.0** (git) |
| `postcard` | 1.1.3 | 1.1.1 |

The `SyncRequester`/`SyncResponder` types, `SyncType` enum, and `SyncRequestMessage` struct may have changed between 0.14.0 and 0.16.1. **Wire format compatibility is unknown** without testing or reviewing the runtime changelog.

**Fix**: Either pin both to the same `aranya-runtime` version, or verify wire compatibility between versions.

#### 4.2.2 Sync Message Wrapping Differs

The two versions wrap sync messages differently:

- **Full version**: Sends bare `SyncType::Poll { request, address }` as postcard bytes over QUIC stream
- **Embedded version**: Wraps in `SyncMessage { t: SyncMessageType, bytes }` enum, then serializes

For interop, either:
- Add the `SyncMessage` wrapper to the full version's transport, or
- Have the embedded version send bare `SyncType` (removing wrapper), or
- Implement a translation layer in a shared transport backend

**Fix**: Align on a common wire format. Since both use postcard serialization for the inner `SyncType`, the actual command data is compatible — only the outer framing differs.

#### 4.2.3 Crypto Envelope Must Match

If the full version seals commands with real AEAD + signatures, the embedded version must be able to `open()` them with the same crypto. This means:
- Both must use the same `Envelope` implementation (or compatible ones)
- Key material must be shared/exchanged at configuration time
- The `CipherSuite` selection must match

**Fix**: Once the embedded version has a real envelope (gap 4.1), use the same key material and cipher suite on both sides.

---

## 5. Revised Gap Summary

| Gap | Severity | Effort | Notes |
|-----|----------|--------|-------|
| NullEnvelope → real crypto envelope | **BLOCKER** | ~1000 LOC, medium | Core crypto is `no_std`-ready; needs implementation + keystore |
| `aranya-runtime` version alignment | **BLOCKER for interop** | Low (dependency pin) | May require testing both versions for wire compat |
| Sync message framing alignment | **BLOCKER for interop** | Low (~50 LOC) | Agree on common wire format |
| Persistent keystore for embedded | **HIGH** | ~300 LOC | Needed for real crypto; ESP32 NVS or flash-backed |
| Custom `NetworkInterface` implementations | **MEDIUM** | ~200 LOC each | For UDP, pub/sub, CCSDS, MAVLink, serial |
| C API (optional) | **LOW** | Medium-High | Not blocking if Rust integration is acceptable |
| Direct data channels (optional) | **LOW** | High | AFC/AQC not in embedded; punt unless needed |

---

## 6. Revised Recommendation

With relaxed requirements, **Aranya Embedded is a viable foundation** — but only after closing the crypto envelope gap. Here is the minimal path:

### Phase 1: Crypto Envelope (Unblocks Auth/Authz)
1. Implement a real `Envelope` FFI module using `aranya-crypto` with `no_std` features
2. Replace `NullEnvelope` with AEAD encryption + Ed25519 signing
3. Implement a flash-backed keystore (replacing `MemStore`)
4. Generate and distribute key material at configuration/provisioning time

### Phase 2: Version Alignment (Unblocks Interop)
1. Pin both codebases to the same `aranya-runtime` version
2. Align sync message framing (common wire format)
3. Test cross-version sync with real commands

### Phase 3: Transport Implementations (Unblocks Your Networks)
1. Implement `NetworkInterface` for each target transport
2. For reliable transports (pub/sub, TCP): simple length-prefix framing, no RaptorQ
3. For lossy transports (UDP broadcast, CCSDS over RF): add RaptorQ encoding
4. For CCSDS/MAVLink: map APID or system/component IDs to the `Addr` type

### Phase 4 (Optional): API & Channels
1. C API via cbindgen if non-Rust integration is needed
2. Direct data channels if application-layer protected exchange is needed

---

## 7. Architecture: Hybrid Deployment

For a mixed environment where some nodes run full Aranya and others run embedded:

```
┌──────────────────────┐     ┌──────────────────────┐
│   Linux Node (Full)  │     │  MCU Node (Embedded)  │
│                      │     │                       │
│  ┌────────────────┐  │     │  ┌─────────────────┐  │
│  │ Aranya Daemon  │  │     │  │ Aranya Embedded  │  │
│  │  (aranya-      │  │     │  │  (monolithic)    │  │
│  │   runtime      │  │     │  │                  │  │
│  │   0.16.x)      │  │     │  │  aranya-runtime  │  │
│  │                │  │     │  │  0.16.x          │  │
│  │  Real Envelope │  │     │  │  Real Envelope   │  │
│  └───────┬────────┘  │     │  └───────┬──────────┘ │
│          │           │     │          │             │
│  ┌───────┴────────┐  │     │  ┌───────┴──────────┐  │
│  │ NetworkIface   │  │     │  │ NetworkIface     │  │
│  │ (UDP/Zenoh/    │  │     │  │ (UDP/CCSDS/      │  │
│  │  CCSDS)        │  │     │  │  MAVLink/Serial) │  │
│  └───────┬────────┘  │     │  └───────┬──────────┘  │
│          │           │     │          │             │
└──────────┼───────────┘     └──────────┼─────────────┘
           │                            │
           └──────── Shared ────────────┘
                    Transport
              (common wire format)
```

Key requirements for this to work:
1. **Same `aranya-runtime` version** on both sides
2. **Same or compatible `Envelope`** implementation (same cipher suite, same key material)
3. **Same sync message wire format** (agree on framing)
4. **Shared `NetworkInterface` wire protocol** for at least one common transport

The full version would need to adopt the `NetworkInterface` trait (or a compatible abstraction) to move beyond QUIC-only. Alternatively, a bridge/gateway node could translate between QUIC (full) and the custom protocol (embedded).

---

## Appendix A: Shared Crates (Protocol Compatibility)

| Crate | Full | Embedded | Role |
|-------|------|----------|------|
| `aranya-runtime` | 0.14.0 | 0.16.1 (git) | Sync algorithm, storage, DAG |
| `aranya-crypto` | 0.10.0 | 0.12.0 (git) | Crypto primitives, keystore trait |
| `aranya-policy-vm` | 0.13.0 | git dep | Policy bytecode execution |
| `aranya-policy-compiler` | 0.13.0 | build dep | Policy DSL → bytecode |
| `postcard` | 1.1.3 | 1.1.1 | Binary serialization |

## Appendix B: NetworkInterface Trait (Full Source)

Location: `/home/user/aranya-embedded/crates/chat-app/src/net.rs`

```rust
pub(crate) trait NetworkInterface {
    type Addr: Copy + core::fmt::Display + core::hash::Hash;
    const BROADCAST: Self::Addr;

    async fn send_message(&mut self, msg: Message<Self::Addr>) -> Result<(), NetworkError>;
    async fn recv_message(&mut self) -> Result<Message<Self::Addr>, NetworkError>;
    fn my_address(&self) -> Self::Addr;
}

pub struct Message<A> {
    pub sender: A,
    pub recipient: A,
    pub contents: Box<[u8]>,
}
```

## Appendix C: NullEnvelope (What Must Be Replaced)

Location: `/home/user/aranya-embedded/crates/envelope-ffi/src/lib.rs`

The FFI schema defines:
```
struct Envelope {
    parent_id id,
    author_id id,
    command_id id,
    payload bytes,
    signature bytes,
}
```

Current `seal()`: SHA256(parent_id || author_id || payload) → command_id, signature = `b"LOL"`, payload = plaintext.

Target `seal()`: AEAD encrypt payload, Ed25519 sign (parent_id || author_id || ciphertext), derive command_id from signature.

Current `open()`: Returns payload unconditionally (Infallible error type).

Target `open()`: Verify signature, AEAD decrypt payload, return plaintext or error.
