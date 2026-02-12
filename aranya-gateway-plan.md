# aranya-gateway: Development Plan

## Context

The goal is to build an **aranya-gateway** — a per-node security layer that uses Aranya's policy engine, sync protocol, and cryptography to authenticate, authorize, and protect all messages exchanged between nodes in aerospace/robotics environments (drones, satellites, ground stations, robots). The gateway must be **transport-agnostic**, operating over CCSDS, MAVLink, ROS2/DDS, IPC, and serial — none of which the current Aranya daemon supports (it is hardcoded to QUIC).

**Architecture choice**: Option B — build a new `aranya-gateway` binary directly on `aranya-runtime`, `aranya-crypto`, and `aranya-policy-vm` as libraries. Do not use or modify the existing `aranya-daemon`.

**Key constraints from the user**:
- Per-node security layer: every node runs its own aranya-gateway instance
- Per-message seal/verify: every application message is individually cryptographically protected
- Soft real-time: milliseconds OK for policy checks, throughput matters
- Pre-onboarded: team + membership provisioned before deployment
- Single team per mission
- Two roles: (1) sync Aranya state across nodes, (2) policy-control application traffic
- Both network-level isolation and data classification via policy

---

## Design Decisions (Recommended)

### 1. Audit Trail: Hybrid

- **Graph commands** for: onboarding state, policy changes, configuration updates, critical control commands (mission-critical actions that need ordering and audit)
- **Out-of-band sealed messages** for: high-rate telemetry, sensor streams, periodic status — sealed with AEAD + signed per-message but NOT persisted to the DAG
- **Rationale**: A drone sending attitude telemetry at 100Hz would create 100 graph nodes/second, overwhelming storage and sync. Out-of-band sealed messages give per-message crypto protection without graph overhead. Control commands (arm, disarm, waypoint upload) are low-rate and need audit trails.

### 2. Transport Multiplexing: Shared Channel with Type Header

- Use a **2-byte message envelope** at the start of every aranya-gateway message:
  ```
  [1 byte: message_type] [1 byte: flags] [payload...]
  message_type: 0x01 = sync, 0x02 = sealed_app_message, 0x03 = sealed_app_command (graph)
  flags: reserved for future use
  ```
- **Rationale**: Simplest universal approach. Works on all transports (serial, CCSDS, MAVLink, ROS2/DDS, UDP). Transports that natively support multiplexing (ROS2 topics, CCSDS APIDs) can optionally use dedicated channels instead, but the header approach works everywhere as a baseline.

### 3. Application Interface: Start with Rust Library, Design for IPC Later

- **Phase 1**: Rust library API — apps link `aranya-gateway` as a crate and call `gateway.seal(dest, msg_type, payload)` / `gateway.open(source, sealed_blob)` directly. Lowest latency, simplest to implement.
- **Phase 2**: Add IPC interface (Unix socket or shared memory) for apps in other languages or that need process isolation.
- **Phase 3** (optional): Transparent proxy mode for specific transports (e.g., ROS2 node that intercepts topics).
- **Rationale**: In-process Rust API is fastest to build and test. IPC adds process isolation later. Transparent proxy is transport-specific and can be built per-transport as needed.

---

## Reusable Components from Existing Codebase

### From `aranya-runtime` (used as-is)
- `SyncRequester` / `SyncResponder` — sync algorithm, produces/consumes `&[u8]`
- `ClientState` — DAG management, command insertion
- `LinearStorageProvider<impl IO>` — graph storage with pluggable I/O
- `PeerCache` — tracks which peers have which commands
- `MAX_SYNC_MESSAGE_SIZE` constant

### From `aranya-crypto` (used as-is)
- AEAD encryption/decryption (`aranya-crypto::aead`)
- Ed25519/ECDSA signing (`aranya-crypto::sign`)
- HPKE key encapsulation (for session key establishment)
- `FileKeyStore` (Linux) or `MemStore` (embedded) for key management
- Key generation utilities

### From `aranya-policy-vm` (used as-is)
- Policy bytecode execution
- `VmPolicy` — seal/open commands with policy enforcement
- Effect emission and processing

### From `aranya-policy-compiler` (build dependency)
- Policy DSL → bytecode compilation
- FFI schema generation for envelope module

### From `aranya-embedded` (patterns to adapt)
- `NetworkInterface` trait pattern — adapt for aranya-gateway's transport abstraction
- `SyncEngine` orchestration — adapt sync scheduling and peer management
- `Sink` trait pattern — adapt for effect handling
- `Daemon` struct pattern — single-binary architecture

### Key Files Reference
| Component | Source | Path |
|-----------|--------|------|
| Sync algorithm | aranya-runtime | External crate (0.14.0 or 0.16.1) |
| Policy VM | aranya-policy-vm | External crate |
| Crypto engine | aranya-crypto | External crate |
| NullEnvelope (to replace) | aranya-embedded | `/home/user/aranya-embedded/crates/envelope-ffi/src/lib.rs` |
| NetworkInterface trait (pattern) | aranya-embedded | `/home/user/aranya-embedded/crates/chat-app/src/net.rs` |
| SyncEngine (pattern) | aranya-embedded | `/home/user/aranya-embedded/crates/demo-esp32-s3/src/aranya/syncer.rs` |
| Daemon struct (pattern) | aranya-embedded | `/home/user/aranya-embedded/crates/demo-esp32-s3/src/aranya/daemon.rs` |
| Full daemon sync (QUIC, for reference) | aranya | `/home/user/aranya/crates/aranya-daemon/src/sync/task/quic.rs` |
| Full daemon PSK (for reference) | aranya | `/home/user/aranya/crates/aranya-daemon/src/sync/task/quic/psk.rs` |
| AFC seal/open (pattern) | aranya | `/home/user/aranya/crates/aranya-client/src/afc.rs` |
| HPKE transport encryption (pattern) | aranya | `/home/user/aranya/crates/aranya-daemon-api/src/crypto/txp.rs` |
| Policy config (pattern) | aranya | `/home/user/aranya/crates/aranya-daemon/src/config.rs` |

---

## What Must Be Built

### Layer 1: Transport Abstraction (~200 LOC)

```rust
/// Core transport trait — adapted from embedded's NetworkInterface
#[async_trait]
pub trait Transport {
    type Addr: Clone + Eq + Hash + Display;
    type Error: std::error::Error;

    async fn send(&mut self, dest: Self::Addr, payload: &[u8]) -> Result<(), Self::Error>;
    async fn recv(&mut self) -> Result<(Self::Addr, Vec<u8>), Self::Error>;
    fn local_addr(&self) -> Self::Addr;
}
```

Implementations needed:
- **UDP** (~200 LOC): Datagram send/recv, IP:port addressing
- **Serial** (~300 LOC): Framing (magic + length + CRC), static peer addressing
- **CCSDS** (~500 LOC): Space packet framing, APID-based addressing. Depends on whether CCSDS is over UDP, serial, or SpaceWire.
- **MAVLink** (~500 LOC): MAVLink v2 framing, system/component ID addressing. Likely wraps UDP or serial.
- **ROS2/DDS** (~800 LOC): DDS participant, topic-based pub/sub. Most complex due to DDS middleware.

### Layer 2: Sync Engine (~500 LOC)

Orchestrates `aranya-runtime`'s sync algorithm over the transport abstraction:
- Periodic sync initiation (configurable interval per peer)
- Hello/request/response message flow (adapt from embedded's `SyncEngine`)
- Multi-transport support (sync over whichever transport reaches each peer)
- Peer tracking and stall timeout handling

### Layer 3: Message Gateway Pipeline (~800 LOC)

The core security gateway:

**Outbound path**:
1. App calls `gateway.send(dest, msg_type, payload)`
2. Policy check: Is this node authorized to send `msg_type` to `dest`?
3. Seal: `AEAD_encrypt(key[dest], nonce, payload)` + `Ed25519_sign(msg_type || dest || nonce || ciphertext)`
4. Wrap: `[header][sealed_blob]`
5. Transport.send(dest, wrapped)

**Inbound path**:
1. Transport.recv() → (source, wrapped)
2. Unwrap: parse header, extract sealed_blob
3. Verify: `Ed25519_verify(source_pubkey, signature)`
4. Unseal: `AEAD_decrypt(key[source], nonce, ciphertext)`
5. Policy check: Is `source` authorized to send `msg_type` to us?
6. Deliver to app

**Sequence number / replay protection**:
- Per-peer monotonic sequence counter (u64)
- Included in AEAD additional data (authenticated but not encrypted)
- Receiving side maintains sliding window to reject replays
- ~100 LOC additional

### Layer 4: Policy Extensions (~300 LOC DSL + ~200 LOC Rust)

Custom policy DSL for the gateway use case:
```
# Message type classification
label TelemetryData
label CommandAuthority
label MissionCritical

# Node-to-node authorization
action authorize_link(source_id id, dest_id id, msg_class label)
action send_message(dest_id id, msg_type int, payload bytes)

command SendMessage:
    # Check sender has appropriate label for this message type
    # Check destination is an authorized recipient
    # Emit effect for the gateway to process

# Data classification rules
command ClassifyMessage:
    # Map protocol-specific message type IDs to Aranya labels
    # e.g., CCSDS APID 0x100 → TelemetryData label
```

### Layer 5: Crypto (Per-Message Seal/Verify) (~400 LOC)

- Adapt the HPKE pattern from `/home/user/aranya/crates/aranya-daemon-api/src/crypto/txp.rs`
- Per-peer AEAD contexts (SealCtx / OpenCtx) established from shared key material
- Key material provisioned at config time (pre-onboarded)
- Persistent keystore using `aranya-crypto`'s `FileKeyStore` (Linux) or flash-backed (embedded)

### Layer 6: Configuration & Provisioning (~400 LOC)

- Load pre-onboarded team state (team ID, member list, key bundles)
- Load transport configuration (which transports, peer addresses per transport)
- Load policy binary (compiled at build time or from file)
- TOML or RON config file format

### Layer 7: Integration Interface (~300 LOC initial)

Phase 1 — Rust API:
```rust
pub struct Gateway { /* ... */ }

impl Gateway {
    pub async fn new(config: GatewayConfig) -> Result<Self>;
    pub async fn send(&self, dest: NodeId, msg_type: u16, payload: &[u8]) -> Result<()>;
    pub async fn recv(&self) -> Result<InboundMessage>;
    pub async fn run_sync(&self);  // Background sync task
}
```

---

## Crate Structure

```
aranya-gateway/
├── Cargo.toml
├── src/
│   ├── lib.rs                  # Public API (Gateway struct)
│   ├── config.rs               # Configuration loading
│   ├── sync.rs                 # Sync engine (wraps aranya-runtime)
│   ├── pipeline.rs             # Message gateway pipeline (seal/unseal/policy)
│   ├── crypto.rs               # Per-message crypto (AEAD + signatures)
│   ├── policy.rs               # Policy engine wrapper + effect handling
│   ├── transport/
│   │   ├── mod.rs              # Transport trait definition
│   │   ├── udp.rs
│   │   ├── serial.rs
│   │   ├── ccsds.rs
│   │   ├── mavlink.rs
│   │   └── ros2.rs
│   └── main.rs                 # Optional standalone binary
├── policy/
│   └── gateway.md              # Policy DSL for gateway use case
└── tests/
    ├── sync_test.rs            # Sync over mock transport
    ├── pipeline_test.rs        # Seal/unseal/policy round-trip
    └── integration_test.rs     # Multi-node test with UDP transport
```

---

## Development Phases

### Phase 1: Core Gateway (Weeks 1-3)
**Goal**: Two nodes exchanging per-message sealed blobs over UDP, with policy enforcement.

1. Create `aranya-gateway` crate with `aranya-runtime`, `aranya-crypto`, `aranya-policy-vm` dependencies
2. Implement `Transport` trait + UDP adapter
3. Implement per-message seal/verify using `aranya-crypto` AEAD + Ed25519
4. Implement basic policy (allow/deny based on node identity + message type)
5. Implement `Gateway` Rust API
6. Test: Two processes on localhost, sealed message round-trip over UDP

### Phase 2: Sync Integration (Weeks 3-5)
**Goal**: Aranya graph state syncs between nodes over the same transport.

1. Implement sync engine (adapt embedded's `SyncEngine` pattern)
2. Add message type header to distinguish sync vs app traffic
3. Implement graph command path (critical commands persisted to DAG)
4. Implement provisioning/config loader (pre-onboarded team + keys)
5. Test: Two nodes sync graph state over UDP, policy changes propagate

### Phase 3: Target Transports (Weeks 5-8)
**Goal**: Gateway works over all target transports.

1. Implement serial adapter (framing + CRC)
2. Implement CCSDS adapter (space packet format + APID addressing)
3. Implement MAVLink adapter (v2 framing + system/component ID)
4. Implement ROS2/DDS adapter (DDS participant + topic mapping)
5. Test each transport with the sync + gateway pipeline

### Phase 4: Policy & Classification (Weeks 8-10)
**Goal**: Full data classification and network segmentation via policy.

1. Design policy DSL extensions for message classification
2. Implement label-based message type mapping (CCSDS APID → label, MAVLink MSG_ID → label)
3. Implement network segmentation rules (node-to-node authorization matrix)
4. Test: Multi-node deployment with mixed authorized/unauthorized message flows

### Phase 5: Hardening & Integration (Weeks 10-12)
**Goal**: Production-ready for target environments.

1. Add IPC interface for non-Rust applications
2. Add metrics / logging (tracing crate)
3. Replay protection testing (sequence number window)
4. Multi-transport simultaneous operation (node reachable via both serial and UDP)
5. Key rotation support
6. Performance benchmarking (throughput, latency)

---

## Verification Plan

### Unit Tests
- Transport trait: mock transport with send/recv verification
- Per-message crypto: seal → unseal round-trip, tamper detection, replay rejection
- Policy: allow/deny decisions for various node + message type combinations
- Sync: graph merge correctness over mock transport

### Integration Tests
- **Two-node UDP**: sealed message exchange + sync
- **Multi-node**: 3+ nodes with different authorization levels, verify segmentation
- **Multi-transport**: node reachable via two transports, verify sync and gateway work on both
- **Provisioning**: load pre-onboarded config, verify nodes recognize each other

### Target Environment Tests
- **Serial loopback**: two processes connected via virtual serial port
- **ROS2**: two ROS2 nodes with aranya-gateway intercepting topic traffic
- **CCSDS**: simulated space link with CCSDS packet exchange

---

## Key Risks

| Risk | Mitigation |
|------|-----------|
| `aranya-runtime` version mismatch (0.14.0 vs 0.16.1) | Pin to 0.16.1 (embedded's version) and verify sync wire format compatibility with full daemon if interop needed later |
| `aranya-crypto` `no_std` gaps on embedded targets | Start on Linux (std), defer embedded targets to later phase |
| ROS2/DDS adapter complexity (DDS middleware is heavy) | Start with simpler transports (UDP, serial), add ROS2 last |
| Policy DSL expressiveness for message classification | Prototype classification rules early, iterate on DSL |
| Per-message crypto performance at high telemetry rates | Benchmark early; AEAD (AES-GCM) is hardware-accelerated on most platforms |
