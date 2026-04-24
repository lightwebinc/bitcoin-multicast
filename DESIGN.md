# Bitcoin Multicast Project Design Document

## Overview

The Bitcoin Multicast project is a high-throughput, horizontally-scalable transaction distribution system for Bitcoin SV (BSV) designed to pave the road towards 1 billion+ transactions per second. It uses IPv6 multicast to efficiently distribute transaction data across a fabric of subscribers (miners, exchanges, service providers) with deterministic sharding and NACK-based reliability.

This document provides a comprehensive design overview of the entire multicast ecosystem, encompassing all repositories and their interactions.

**Conceptual Attribution:** The IPv6 multicast transaction broadcast architecture from which this software draws inspiration was articulated by Dr. Craig S. Wright in [Multicast within Multicast: Anycast](https://singulargrit.substack.com/p/multicast-within-multicast-anycast).

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Repository Overview](#repository-overview)
3. [Network Topology](#network-topology)
4. [Data Flow](#data-flow)
5. [Sharding Mechanism](#sharding-mechanism)
6. [Frame Format](#frame-format)
7. [Component Deep Dives](#component-deep-dives)
8. [Retransmission and Reliability](#retransmission-and-reliability)
9. [Subtree Filtering](#subtree-filtering)
10. [Testing and Validation](#testing-and-validation)
11. [Deployment Considerations](#deployment-considerations)

---

## High-Level Architecture

The multicast pipeline consists of three tiers:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BSV Senders (Miners, Services)                       │
│                              (UDP/TCP Ingress)                              │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Ingress Tier (bitcoin-ingress)                       │
│                    Deploys: bitcoin-shard-proxy nodes                       │
│                  Stateless, deterministic, horizontally scalable            │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │  IPv6 UDP Multicast (FF05::<shard>)
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Multicast Fabric (Site-Scoped)                        │
│                    FF05::/16, UDP Port 9001                                 │
│         ┌──────────────┬──────────────┬──────────────┐                      │
│         │ Direct Subs  │   Listeners  │  Retry Nodes │                      │
│         │ (Miners,     │  (Filtered   │  (Cache &    │                      │
│         │  Exchanges)  │   Forward)   │   Retransmit)│                      │
│         └──────────────┴──────────────┴──────────────┘                      │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
        ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
        │ Downstream   │ │ Downstream   │ │ Downstream   │
        │ Consumers    │ │ Consumers    │ │ Consumers    │
        └──────────────┘ └──────────────┘ └──────────────┘
```

**Key Design Principles:**

- **Stateless Ingress:** Proxy nodes carry no state; any number can be deployed without coordination
- **Deterministic Sharding:** Same transaction ID always maps to the same multicast group
- **Consistent Hashing:** Increasing shard bits splits groups without invalidating existing subscriptions
- **Horizontal Scale:** Add nodes to scale capacity; no reconfiguration of existing nodes required
- **NACK-based Recovery:** Listeners detect gaps and request retransmission from cached retry endpoints

---

## Repository Overview

The project is organized into multiple repositories, each with a specific responsibility:

### Core Services (Binaries)

| Repository | Purpose | Main Binary |
|------------|---------|-------------|
| [bitcoin-shard-proxy](https://github.com/lightwebinc/bitcoin-shard-proxy) | Stateless ingress proxy; receives frames, derives multicast group, forwards verbatim | `bitcoin-shard-proxy` |
| [bitcoin-shard-listener](https://github.com/lightwebinc/bitcoin-shard-listener) | Multicast subscriber; filters by shard/subtree, forwards to unicast consumers | `bitcoin-shard-listener` |
| [bitcoin-retry-endpoint](https://github.com/lightwebinc/bitcoin-retry-endpoint) | Caches frames, retransmits on NACK requests | `bitcoin-retry-endpoint` |

### Shared Libraries

| Repository | Purpose | Packages |
|------------|---------|----------|
| [bitcoin-shard-common](https://github.com/lightwebinc/bitcoin-shard-common) | Protocol primitives shared across services | `frame`, `shard`, `sequence` |

### Infrastructure Automation

| Repository | Purpose | Deploys |
|------------|---------|---------|
| [bitcoin-ingress](https://github.com/lightwebinc/bitcoin-ingress) | Ansible/Terraform for ingress proxy deployment | bitcoin-shard-proxy |
| [bitcoin-listener](https://github.com/lightwebinc/bitcoin-listener) | Ansible/Terraform for listener deployment | bitcoin-shard-listener |
| [bitcoin-retransmission](https://github.com/lightwebinc/bitcoin-retransmission) | Ansible/Terraform for retry endpoint deployment | bitcoin-retry-endpoint |

### Testing and Tools

| Repository | Purpose |
|------------|---------|
| [bitcoin-subtx-generator](https://github.com/lightwebinc/bitcoin-subtx-generator) | Traffic generator for load/functional testing |

### Meta Repository

| Repository | Purpose |
|------------|---------|
| [bitcoin-multicast](https://github.com/lightwebinc/bitcoin-multicast) | This repository; project overview and design documentation |

---

## Network Topology

### Full Production Topology

```text
                    ┌─────────────────────────────────────────────────────────┐
                    │                    BSV Senders                          │
                    │            (Miners, Transaction Services)               │
                    └───────────────────────────┬─────────────────────────────┘
                                                │  UDP/TCP (BRC-12/BRC-122 frames)
                    ┌───────────────────────────┼─────────────────────────────┐
                    │                           │                             │
              ┌─────▼─────┐               ┌─────▼─────┐                 ┌─────▼─────┐
              │ Ingress   │               │ Ingress   │                 │ Ingress   │
              │ Node A    │               │ Node B    │                 │ Node C    │
              │ (proxy)   │               │ (proxy)   │                 │ (proxy)   │
              └─────┬─────┘               └─────┬─────┘                 └─────┬─────┘
                    │  IPv6 UDP Multicast (FF05::<shard>, port 9001)          │
                    └───────────────────────────┼─────────────────────────────┘
                                                │
                                ┌───────────────┴───────────────┐
                                │   Multicast Fabric Router     │
                                │   (MLD/PIM, FF05::/16)        │
                                └───────────────┬───────────────┘
                                                │
        ┌───────────────────────┬───────────────┼───────────────┬─────────────────┐
        │                       │               │               │                 │
  ┌───────────┐         ┌───────────┐   ┌───────────┐   ┌───────────┐       ┌───────────┐
  │  Miner 1  │         │  Miner 2  │   │ Listener  │   │ Listener  │       │  Retry    │
  │  (direct) │         │  (direct) │   │  Node A   │   │  Node B   │       │ Endpoint  │
  └───────────┘         └───────────┘   └─────┬─────┘   └─────┬─────┘       └─────┬─────┘
                                              │               │                   │
                                              │ Unicast       │ Unicast           │ NACK (UDP)
                                              │ UDP/TCP       │ UDP/TCP           │ port 9300
                                              ▼               ▼                   │
                                    ┌──────────────┐ ┌──────────────┐             │
                                    │ Consumer A   │ │ Consumer B   │             │
                                    └──────────────┘ └──────────────┘             │
                                                                                  │
                                                   NACK Retransmission ◄──────────┘
                                              (re-multicast to FF05::<shard>)
```

---

## Data Flow

### Normal Flow (No Retransmission)

```text
1. BSV Sender → bitcoin-shard-proxy
   ┌─────────────────────────────────────────────────────────────────────────┐
   │ UDP/TCP: BRC-12/BRC-122 frame (TxID, payload, optional Sequence, Subtree) │
   └─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
2. bitcoin-shard-proxy
   ┌─────────────────────────────────────────────────────────────────────────┐
   │ • Decode frame (extract TxID)                                           │
   │ • Stamp Sender ID in-place (BRC-122 only, bytes 40-43)                    │
   │ • Derive multicast group: FF05::<groupIndex> from TxID top bits         │
   │ • Forward verbatim to all egress interfaces                             │
   └─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
3. Multicast Fabric
   ┌─────────────────────────────────────────────────────────────────────────┐
   │ FF05::<groupIndex>:9001 delivered to all joined subscribers             │
   │ (MLD snooping / PIM distribution tree)                                  │
   └─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
4a. Direct Subscriber              4b. bitcoin-shard-listener
   ┌─────────────────────────────┐         ┌──────────────────────────────────────────┐
   │ Miner / Exchange            │         │ • Join configured shard groups via MLD   │
   │ (consumes directly)         │         │ • Apply shard filter (defense-in-depth)  │
   └─────────────────────────────┘         │ • Apply subtree filter (include/exclude) │
                                           │ • Track sequence gaps per (SenderID,     │
                                           │   groupIndex)                            │
                                           │ • Forward matching frames to egress_addr │
                                           │   (UDP or TCP, optional strip-header)    │
                                           └──────────────────────────────────────────┘
                                                            │
                                                            ▼
                                                5. Downstream Consumer
```

### Retransmission Flow (NACK-based)

```text
bitcoin-shard-listener detects gap:
┌─────────────────────────────────────────────────────────────────────────┐
│ • SeqNum arrives > highestConsec + 1                                    │
│ • Register missing seqs in pending map                                  │
│ • Background sweeper dispatches NACK after nack-gap-ttl                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
NACK Dispatch (UDP to retry-endpoint:9300)
┌─────────────────────────────────────────────────────────────────────────┐
│ 64-byte NACK datagram: (TxID, ShardSeqNum, SubtreeID, SenderID)         │
│ Sent to all configured retry endpoints one after another                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
bitcoin-retry-endpoint
┌─────────────────────────────────────────────────────────────────────────┐
│ • Receive NACK on port 9300                                             │
│ • Rate limit (IP, SenderID, SequenceID)                                 │
│ • Lookup frame in cache (memory or Redis)                               │
│ • If found: re-multicast to FF05::<shard>:9100                          │
│ • Dedup via Redis SET NX (60s window) to prevent duplicate retransmits  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
bitcoin-shard-listener receives repair
┌─────────────────────────────────────────────────────────────────────────┐
│ • Frame arrives via normal multicast path                               │
│ • Gap tracker fills pending entry → bsl_gaps_suppressed_total           │
│ • Frame forwarded to downstream consumer                                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Sharding Mechanism

### Deterministic Group Derivation

The multicast group for a transaction is derived purely from its transaction ID:

```text
groupIndex = (txid[0:4] as uint32 BE) >> (32 - shardBits)
IPv6 group = [FFsc::groupIndex]
```

**Example with shard_bits=2:**

| txid[0:4] (hex) | txid[0:4] (uint32)  | >> 30 | groupIndex | Multicast Address |
|-----------------|---------------------|-------|------------|-------------------|
| 0x12345678      | 305419896           | 0     | 0          | FF05::0           |
| 0x87654321      | 2271560481          | 3     | 3          | FF05::3           |
| 0xABCD1234      | 2882343444          | 2     | 2          | FF05::2           |
| 0x4567ABCD      | 1164413357          | 1     | 1          | FF05::1           |

### Consistent Hashing Property

Using top bits (right shift) instead of modulo provides consistent hashing:

```text
shard_bits = 2  →  4 groups (0, 1, 2, 3)
shard_bits = 3  →  8 groups (0a, 0b, 1a, 1b, 2a, 2b, 3a, 3b)

Group 0 splits into:  0a (txid[0] bit 31 = 0), 0b (txid[0] bit 31 = 1)
Group 1 splits into:  1a (txid[0] bit 31 = 0), 1b (txid[0] bit 31 = 1)
...
```

**Benefit:** When increasing shard_bits, subscribers only need to join additional groups. Existing subscriptions remain valid.

### IPv6 Multicast Address Layout

```text
Bits [127:112]   FFsc   Multicast prefix + scope (e.g., FF05 for site-local)
Bits [111:24]    0x00   Zero padding (assigned address space)
Bits [23:0]      index  Group index (up to 24 bits = 16,777,216 groups)
```

**Scope Codes:**

```text
| Scope        | Code | Example  | Use Case                 |
|--------------|------|----------|--------------------------|
| link-local   | 1    | FF01::   | Single network segment   |
| site-local   | 5    | FF05::   | Entire site (default)    |
| organization | 8    | FF08::   | Multi-site organization  |
| global       | E    | FF0E::   | Internet-wide            |
```

## Frame Format

### BRC-122 Frame Format (Current - 92 bytes)

All multi-byte integers are big-endian. 8-byte alignment for all fields after offset 8.

```text
Offset  Size  Align  Field                 Value / Notes
------  ----  -----  -----                 -------------
     0     4   —     Network magic         0xE3E1F3E8 (BSV mainnet P2P magic)
     4     2   —     Protocol ver          0x02BF = 703 (BSV node version baseline)
     6     1   —     Frame version         0x02 (BRC-122)
     7     1   —     Reserved              0x00
     8    32   8B    Transaction ID        Raw 256-bit txid (internal byte order)
    40     4   8B    Sender ID             CRC32c of source IPv6 address; 0 = unset
    44     4   —     Sequence ID           Random temporal flow identifier; 0 = unset
    48     4   8B    Shard Sequence Number Monotonic sender counter; 0 = unset
    52     4   —     Reserved              Must be 0x00000000
    56    32   8B    Subtree ID            32-byte batch identifier; zeros = unset
    88     4   8B    Payload length        uint32 BE; max 10 MiB
    92     *   —     BSV tx payload        Raw serialised transaction bytes
```

**Field Details:**

- **Network magic (0:4):** BSV mainnet P2P magic; enables standard firewall classification
- **Protocol version (4:6):** Informational; 703 = BSV large-block policy baseline
- **Frame version (6):** 0x02 for BRC-122, 0x01 for legacy BRC-12
- **Transaction ID (8:40):** Raw 256-bit txid in internal byte order (NOT display-reversed)
- **Sender ID (40:44):** CRC32c checksum of source IPv6 address (Castagnoli polynomial 0x1EDC6F41). Enables per-sender gap tracking with minimal header overhead. Given BSV's mining economics, collision probability is ~0.01% at 1,000 senders, negligible at 100 senders.
- **Sequence ID (44:48):** Random temporal flow identifier; reset periodically by sender; combined with Sender ID and Shard Sequence Number for retransmission key
- **Shard Sequence Number (48:52):** Monotonic counter per sender; enables gap detection and NACK-based retransmission
- **Reserved (52:56):** Padding for 8-byte alignment of Subtree ID; must be zero
- **Subtree ID (56:88):** Opaque 32-byte batch identifier for subtree-level filtering
- **Payload (92+):** BSV transaction in BRC-12 raw format (version LE32 + inputs + outputs + locktime LE32)

### BRC-12 / v1 Frame Format (Legacy - 44 bytes)

Accepted and forwarded verbatim for backward compatibility.

```text
Offset  Size  Field
------  ----  -----
     0     4  Network magic    0xE3E1F3E8
     4     2  Protocol ver     0x02BF
     6     1  Frame version    0x01
     7     1  Reserved         0x00
     8    32  Transaction ID
    40     4  Payload length
    44     *  Payload
```

**v1 Limitations:** No Sequence ID, Shard Sequence Number, Subtree ID, or Sender ID fields. Gap tracking and subtree filtering do not apply.

### Frame Processing Rules

**Proxy (bitcoin-shard-proxy):**
- Decode header (v1 or BRC-122); drop on bad magic or unknown version
- For BRC-122: stamp Sender ID in-place at bytes 40-43 from ingress source address CRC32c
- Forward verbatim to all egress interfaces (no re-encoding)

**Listener (bitcoin-shard-listener):**
- Decode header; apply shard filter (group index)
- Apply subtree filter (SubtreeID include/exclude)
- For BRC-122 with non-zero Sender ID and Shard Sequence Number: track gaps per (Sender ID, groupIndex)
- Forward matching frames to egress_addr (UDP or TCP)

---

## Component Deep Dives

### bitcoin-shard-proxy (Ingress)

**Purpose:** Stateless ingress proxy; receives BSV transaction frames, derives multicast group, forwards verbatim.

**Key Characteristics:**
- Zero-copy forwarding: frame never modified after SenderID stamp
- Multi-CPU design: N UDP workers via SO_REUSEPORT + 1 TCP listener
- Deterministic: same txid always maps to same group
- Stateless: no coordination between workers or nodes required

**Architecture:**
```text
UDP Workers (N goroutines, SO_REUSEPORT)
  ┌─────────┐
  │ Worker 0│──┐
  └─────────┘  │
  ┌─────────┐  ├──▶ shared Forwarder ──▶ egress sockets (per iface)
  │ Worker 1│──│
  └─────────┘  │
  ...          │
  ┌─────────┐  │
  │ Worker N│──┘
  └─────────┘

TCP Listener (1 goroutine)
  ┌──────────────┐
  │ Accept loop  │──▶ per-connection goroutines ──▶ shared Forwarder
  └──────────────┘
```

**Hot Path:**
1. `frame.Decode(raw)` → extract TxID
2. Stamp Sender ID in-place (BRC-122 only): `raw[40:44] = CRC32c(sourceAddr.To16())`
3. `shard.Engine.GroupIndex(txid)` → derive group
4. `WriteTo(raw)` → write to all egress interfaces

**Configuration:**
- `-iface`: Egress interface(s) (comma-separated)
- `-shard-bits`: Txid prefix bit width (1-24, default 16)
- `-scope`: Multicast scope (link|site|org|global, default site)
- `-udp-listen-port`: UDP ingress port (default 9000)
- `-tcp-listen-port`: TCP ingress port (0 = disabled)
- `-egress-port`: Multicast egress port (default 9001)

**Metrics (bsp_ prefix):**
- `bsp_packets_received_total`, `bsp_packets_forwarded_total`, `bsp_packets_dropped_total`
- `bsp_bytes_received_total`, `bsp_bytes_forwarded_total`
- `bsp_tcp_connections_total`, `bsp_tcp_errors_total`

**Documentation:** [bitcoin-shard-proxy README](https://github.com/lightwebinc/bitcoin-shard-proxy), [architecture docs](https://github.com/lightwebinc/bitcoin-shard-proxy/blob/main/docs/architecture.md)

---

### bitcoin-shard-listener (Subscriber)

**Purpose:** Multicast subscriber; filters by shard/subtree, forwards to unicast consumers, performs NACK-based gap recovery.

**Key Characteristics:**
- SO_REUSEPORT multi-worker receive (kernel-level source affinity)
- Dual-level filtering: MLD group join + userspace shard/subtree filter
- NORM-inspired gap tracking per (SenderID, groupIndex)
- NACK dispatch to configurable retry endpoints
- Egress via UDP or TCP (optional strip-header mode)

**Architecture:**
```text
Receive Workers (NUM_WORKERS goroutines, SO_REUSEPORT)
  ┌─────────┐
  │ Worker 0│──┐
  └─────────┘  │
  ┌─────────┐  ├──▶ per-worker components:
  │ Worker 1│──│     • frame.Decode
  └─────────┘  │     • shard.Engine.GroupIndex
  ...          │     • filter.Allow
  ┌─────────┐  │     • egress.Send
  │ Worker N│──│     • nack.Tracker.Observe
  └─────────┘  │
               │

NACK Queue (background goroutines)
  ┌──────────────────┐
  │ NACK dispatcher  │──▶ UDP send to retry-endpoint:9300
  └──────────────────┘

Gap Tracker Sweeper (100ms interval)
  ┌──────────────────┐
  │ Evict expired    │
  │ Dispatch pending │
  └──────────────────┘
```

**Important:** Linux delivers multicast to ALL SO_REUSEPORT sockets (no load balancing). For multicast deployments, `NUM_WORKERS` must be set to 1. Multiple workers are only useful for unicast ingress (E2E test suite).

**Filter Behavior:**
| Config | Behavior |
|--------|----------|
| `shard-include` empty | All shard indices accepted |
| `shard-include` non-empty | Only listed indices accepted |
| `subtree-include` empty | All SubtreeIDs accepted |
| `subtree-include` non-empty | Only listed IDs accepted |
| `subtree-exclude` | Listed IDs dropped (overrides include) |

**Gap Tracking:**
- State key: `(SenderID, groupIndex)`
- Per-key state: `highestConsec`, `pending` map
- When seq > highestConsec+1: register missing seqs in pending
- When pending seq arrives: delete from pending, increment `bsl_gaps_suppressed_total`
- Sweeper evicts expired gaps as `bsl_gaps_unrecovered_total`
- NACK dispatch with exponential backoff (capped at `nack-backoff-max`)

**Configuration:**
- `-iface`: Ingress interface for multicast receive
- `-listen-port`: Multicast listen port (default 9001)
- `-shard-bits`: Txid prefix bit width (1-24, default 16)
- `-shard-include`: Comma-separated shard indices to accept (empty = all)
- `-subtree-include`: Comma-separated 32-byte hex SubtreeIDs to accept
- `-subtree-exclude`: Comma-separated 32-byte hex SubtreeIDs to drop
- `-egress-addr`: Downstream consumer address (UDP or TCP)
- `-egress-proto`: udp or tcp (default udp)
- `-strip-header`: Send payload only (default false)
- `-retry-endpoints`: Comma-separated retry endpoint addresses
- `-nack-gap-ttl`: Gap detection TTL (default 100ms)
- `-nack-max-retries`: Maximum NACK retries per gap (default 3)

**Metrics (bsl_ prefix):**
- `bsl_frames_received_total`, `bsl_frames_forwarded_total`, `bsl_frames_dropped_total`
- `bsl_gaps_detected_total`, `bsl_gaps_suppressed_total`, `bsl_gaps_unrecovered_total`
- `bsl_nacks_sent_total`, `bsl_nacks_unrecovered_total`

**Documentation:** [bitcoin-shard-listener README](https://github.com/lightwebinc/bitcoin-shard-listener), [architecture docs](https://github.com/lightwebinc/bitcoin-shard-listener/blob/main/docs/architecture.md)

---

### bitcoin-retry-endpoint (Retransmission)

**Purpose:** Caches frames, retransmits on NACK requests from listeners.

**Key Characteristics:**
- Single-worker multicast receiver (SO_REUSEPORT limitation)
- Modular cache backend: Redis (primary) or in-memory (fallback)
- Three-level rate limiting: IP, SenderID, SequenceID
- Cross-instance deduplication via Redis SET NX (60s window)
- Sharding-based multicast egress for retransmitted frames

**Architecture:**
```text
Multicast Receiver (1 worker, SO_REUSEPORT)
  ┌──────────────────┐
  │ Join all groups  │──▶ Cache (memory or Redis)
  └──────────────────┘

NACK Server (NACK_WORKERS goroutines)
  ┌──────────────────┐
  │ UDP listener     │──▶ Rate limit ──▶ Cache lookup ──▶ Retransmit
  └──────────────────┘

Retransmit Egress
  ┌──────────────────┐
  │ Multicast send   │──▶ FF05::<shard>:9100
  └──────────────────┘
```

**Cache Backends:**
- **memory:** In-memory cache; single-node deployments
- **redis:** External Redis cluster; multi-node deployments, shared cache

**Rate Limiting:**
- IP level: Token bucket (rate + burst)
- SenderID level: Sliding window per sender
- SequenceID level: Max requests per sequence

**Deduplication:**
- Key: SenderID (16B) + SequenceID (8B) + ShardSeqNum (8B) = 32B
- Redis SET NX with 60s TTL prevents multiple endpoints from retransmitting the same frame

**Configuration:**
- `-mc-iface`: Multicast ingress interface
- `-listen-port`: Multicast listen port (default 9001)
- `-shard-bits`: Txid prefix bit width (1-24, default 16)
- `-cache-backend`: redis or memory (default memory)
- `-redis-addr`: Redis server address (default localhost:6379)
- `-cache-ttl`: Cache TTL (default 10m)
- `-nack-port`: NACK listen port (default 9300)
- `-nack-workers`: NACK worker goroutines (default NumCPU)
- `-egress-iface`: Egress interface(s) for retransmission
- `-egress-port`: Retransmission port (default 9100)
- `-dedup-window`: Deduplication window (default 60s)

**Metrics (bre_ prefix):**
- `bre_cache_hits_total`, `bre_cache_misses_total`, `bre_cache_size`
- `bre_nack_requests_total`, `bre_retransmits_total`, `bre_retransmit_dedup_total`
- `bre_rate_limit_drops_total{level=ip|sender|sequence}`
- `bre_frames_received_total`, `bre_frames_cached_total`

**Documentation:** [bitcoin-retry-endpoint README](https://github.com/lightwebinc/bitcoin-retry-endpoint)

---

### bitcoin-shard-common (Protocol Primitives)

**Purpose:** Shared protocol primitives for the BSV transaction sharding pipeline.

**Packages:**

**frame:** v1/BRC-122 wire format encoding/decoding
- `Encode(f *Frame, buf []byte) (int, error)`: Serialize frame to buffer
- `Decode(buf []byte) (*Frame, error)`: Parse buffer to frame (zero-copy)
- Constants: `MagicBSV`, `ProtoVer`, `FrameVerLegacy`, `FrameVerBRC122`, `HeaderSizeLegacy`, `HeaderSize`, `MaxPayload`
- Errors: `ErrBadMagic`, `ErrBadVer`, `ErrTooLarge`, `ErrTooShort`

**shard:** Deterministic txid → IPv6 multicast group derivation
- `Engine`: Immutable sharding parameters (mcPrefix, middleBytes, shardBits)
- `GroupIndex(txid *[32]byte) uint32`: Extract group index from txid
- `Addr(groupIndex uint32, port int) *net.UDPAddr`: Construct multicast address
- Consistent hashing: increasing shardBits splits groups without invalidating existing subscriptions

**sequence:** Per-shard monotonic sequence counters
- Per-shard atomic.Uint64 counters
- Zero allocation, no contention between shards
- Used for proxy-side sequence numbering (currently unused in main binary)

**Documentation:** [bitcoin-shard-common README](https://github.com/lightwebinc/bitcoin-shard-common), [protocol spec](https://github.com/lightwebinc/bitcoin-shard-common/blob/main/docs/protocol.md)

---

## Retransmission and Reliability

### NACK Protocol

Listeners detect sequence gaps and send NACK datagrams to retry endpoints:

**NACK Format (64 bytes):**
```
Offset  Size  Field
------  ----  -----
     0    32  Transaction ID
    32     8  ShardSeqNum
    40    32  SubtreeID
```

**NACK Dispatch Flow:**
```
1. Gap detected (seq > highestConsec + 1)
   → Register in pending map with detected timestamp

2. Background sweeper (100ms interval)
   → If gap > nack-gap-ttl and retries < nack-max-retries
     → Dispatch NACK to all retry-endpoints
     → Increment retries, update nextAttempt (exponential backoff)

3. If retries exhausted
   → Evict as bsl_gaps_unrecovered_total
```

**Exponential Backoff:**
- Initial delay: `nack-gap-ttl` (default 100ms)
- Backoff multiplier: 2x
- Maximum backoff: `nack-backoff-max` (default 5s)

### Retry Endpoint Processing

```
1. Receive NACK on port 9300
2. Rate limit check (IP, SenderID, SequenceID)
   → If exceeded: silent drop, increment bre_rate_limit_drops_total
3. Cache lookup (TxID, ShardSeqNum, SubtreeID)
   → If found in cache:
     • Check dedup key (SenderID + SequenceID + ShardSeqNum)
     • If not recently retransmitted (Redis SET NX):
       → Re-multicast to FF05::<shard>:9100
       → Increment bre_retransmits_total
     • Else: increment bre_retransmit_dedup_total
   → If not found:
     → Increment bre_cache_misses_total
```

### Reliability Characteristics

**Best-effort delivery:**
- Multicast is inherently unreliable (no ACKs)
- NACK provides selective retransmission for detected gaps
- No guarantee of recovery (network partition, cache expiration)

**Cache TTL considerations:**
- Default cache TTL: 10 minutes
- Trade-off: Longer TTL = higher recovery probability, but more memory
- Adjust based on expected gap detection latency and network conditions

**Silent drops:**
- Bad frames (magic, version, payload size) are silently dropped
- Rate-limited NACKs are silently dropped
- Missing cache entries result in silent NACK drops
- All drops are counted in metrics

---

## Subtree Filtering

### Subtree Model

A *subtree* is an ordered set of related transactions sharing a common batch context. The 32-byte `SubtreeID` field allows downstream subscribers to associate frames with a named batch.

**Use Cases:**
- Shard by transaction type (payments, contracts, tokens)
- Shard by application or service
- Shard by geographic region
- Shard by time window

### Subtree Filter Behavior

**Include Mode:**
```
subtree-include = "abc123...,def456..."  (hex, 32-byte each)
→ Only frames with SubtreeID in this set pass
```

**Exclude Mode:**
```
subtree-exclude = "abc123...,def456..."  (hex, 32-byte each)
→ Frames with SubtreeID in this set are dropped (overrides include)
```

**Empty Sets:**
- `subtree-include` empty: all SubtreeIDs accepted
- `subtree-exclude` empty: no exclusion
- Both empty: no subtree filtering (all frames pass)

**V1 Frames:**
- V1 frames have zero SubtreeID
- Only pass subtree filter if zero is explicitly listed in `subtree-include`

### Deterministic Subtree Selection (Testing)

The `bitcoin-subtx-generator` tool uses deterministic subtree selection for reproducible tests:

```
SubtreeID = pool[uint64(TxID[:8]) % N]
```

With N=8 subtrees and a fixed seed, the same txid always maps to the same subtree. This allows listeners filtering on a single subtree to see a predictable traffic fraction (~1/N).

---

## Testing and Validation

### bitcoin-subtx-generator

**Purpose:** Random BSV-shaped frame generator for load and functional testing.

**Features:**
- Random BSV-shaped tx payloads (version/vin/vout/locktime)
- Subtree ID pool (N deterministic 32-byte IDs from seed)
- Sequence numbers with optional gap injection (permanent or delayed retransmission)
- Multi-core sender (one UDP conn per worker)
- Token-bucket pacer (smooth at ≤1 kpps, burst mode above)

**Usage Examples:**

```bash
# Basic load test
subtx-gen \
  -addr [::1]:9000 \
  -frame-version 2 \
  -shard-bits 2 \
  -subtrees 8 \
  -subtree-seed 'test-seed' \
  -pps 1000 \
  -duration 10s \
  -payload-size 512 \
  -workers 0

# Gap injection (NACK/retransmit test)
subtx-gen -pps 1000 -duration 30s -seq-gap-every 500

# Delayed retransmit (gap recovery test)
subtx-gen -pps 1000 -duration 30s -seq-gap-every 500 -seq-gap-delay 50ms

# Inspect subtree pool
subtx-gen -subtrees 8 -subtree-seed 'test-seed' -print-subtrees
```

**Documentation:** [bitcoin-subtx-generator README](https://github.com/lightwebinc/bitcoin-subtx-generator)

### bitcoin-shard-listener E2E Tests

**Purpose:** Self-contained end-to-end tests for listener functionality.

**Approach:** Inject frames as unicast UDP directly to listener's bound port (`[::1]:listen-port`), bypassing proxy and multicast fabric. This avoids Linux loopback multicast reliability issues on CI.

**Test Scenarios:**
1. Basic delivery (all frames, metric verification)
2. Shard filter (single shard acceptance)
3. Strip-header (payload-only forwarding)

**Execution:**
```bash
cd bitcoin-shard-listener
make test-e2e
```

**Documentation:** [bitcoin-shard-listener README](https://github.com/lightwebinc/bitcoin-shard-listener)

---

## Deployment Considerations

### Platform Support

| OS | Service Manager | Network Config | Proxy | Listener | Retry |
|----|-----------------|----------------|-------|----------|-------|
| Ubuntu 24.04 | systemd | Netplan / ip | ✓ | ✓ | ✓ |
| FreeBSD 14 | rc.d | rc.conf / ifconfig | ✓ | ✓ | ✓ |
| AWS EC2 | systemd | ENI + Terraform | ✓ | ✓ | ✓ |

### Networking Requirements

**Ingress (bitcoin-shard-proxy):**
- IPv6 enabled on egress interface(s)
- Multicast routing / MLD snooping configured for subscriber fabric
- Optional: GRE tunnel for cloud VMs
- Optional: eBGP for nearest-node routing

**Listener (bitcoin-shard-listener):**
- IPv6 enabled on ingress interface
- MLDv1/v2 support for multicast group join
- Optional: BGP for listener reachability into fabric
- Firewall: multicast-fabric perimeter (default-on in bitcoin-listener)

**Retry Endpoint (bitcoin-retry-endpoint):**
- IPv6 enabled on multicast interface
- Optional: Redis for shared cache (multi-node deployments)

### Firewall Configuration

**Proxy (bitcoin-ingress):**
- Allow UDP/TCP ingress on listen port (default 9000)
- Allow IPv6 multicast egress on egress interface
- No additional firewall rules required

**Listener (bitcoin-listener):**
- **Multicast-fabric perimeter:** Built-in firewall enforces:
  - Ingress: Only multicast data on ingress interface
  - Egress: Only NACK datagrams outbound
  - All other traffic dropped
- See [bitcoin-listener security docs](https://github.com/lightwebinc/bitcoin-listener/blob/main/docs/security.md)

**Retry Endpoint (bitcoin-retransmission):**
- Simplified UDP-only firewall
- Allow NACK ingress on port 9300
- Allow multicast egress on egress interface

### BGP Integration

**Ingress (bitcoin-ingress):**
- Optional eBGP on ingress interface
- Announce shared prefixes from all proxy nodes
- Senders routed to nearest proxy via BGP best-path selection
- See [bitcoin-ingress BGP docs](https://github.com/lightwebinc/bitcoin-ingress/blob/main/docs/bgp.md)

**Listener (bitcoin-listener):**
- Optional BGP for listener reachability into fabric
- Advertise listener's own unicast prefix
- Enables MLD/PIM distribution trees in L3 fabrics
- See [bitcoin-listener BGP docs](https://github.com/lightwebinc/bitcoin-listener/blob/main/docs/bgp.md)

**Retry Endpoint (bitcoin-retransmission):**
- No BGP integration (pure cache-and-retransmit service)

### Scaling Guidelines

**Ingress Scaling:**
- Add more proxy nodes (stateless, no coordination)
- Increase `shard_bits` to split multicast groups
- Use eBGP for load distribution across nodes

**Listener Scaling:**
- Deploy multiple listeners with different `shard-include` configurations
- Use subtree filtering for application-level sharding
- Horizontal scale: add more listeners per shard/subtree

**Retry Endpoint Scaling:**
- Deploy multiple retry endpoints with shared Redis cache
- Cross-instance deduplication prevents duplicate retransmissions
- Rate limiting protects against NACK storms

### Monitoring and Metrics

All services expose Prometheus metrics on dedicated ports:

```text
| Service                | Metrics Port | Prefix |
|------------------------|--------------|--------|
| bitcoin-shard-proxy    | :9100        | bsp_   |
| bitcoin-shard-listener | :9200        | bsl_   |
| bitcoin-retry-endpoint | :9400        | bre_   |
```

**Key Metrics to Monitor:**

**Proxy:**
- `bsp_packets_dropped_total{reason}`: Packet loss reasons
- `bsp_bytes_forwarded_total`: Throughput
- `bsp_tcp_errors_total`: TCP ingress errors

**Listener:**
- `bsl_frames_dropped_total{reason}`: Drop reasons (shard_filter, subtree_exclude, etc.)
- `bsl_gaps_detected_total`: Gap detection rate
- `bsl_gaps_suppressed_total`: Successful gap recovery
- `bsl_nacks_unrecovered_total`: Unrecoverable gaps

**Retry Endpoint:**
- `bre_cache_misses_total`: Cache effectiveness
- `bre_retransmit_dedup_total`: Duplicate suppression
- `bre_rate_limit_drops_total{level}`: Rate limit violations

### Graceful Shutdown

All services support graceful shutdown via SIGINT/SIGTERM:

**Proxy:**
1. Drain: Set draining flag, `/readyz` returns 503
2. Optional drain timeout (configurable)
3. Close ingress sockets
4. Wait for in-flight processing
5. Flush OTLP exporter

**Listener:**
1. Drain: Set draining flag, `/readyz` returns 503
2. Optional drain timeout (configurable)
3. Close ingress sockets
4. Wait for in-flight processing
5. Flush OTLP exporter

**Retry Endpoint:**
1. Drain: Set draining flag, `/readyz` returns 503
2. Optional drain timeout (configurable)
3. Close ingress sockets
4. Wait for in-flight processing
5. Flush OTLP exporter

---

## References and Further Reading

### Source Documentation

**Protocol:**
- [Wire Protocol Specification](https://github.com/lightwebinc/bitcoin-shard-common/blob/main/docs/protocol.md) - Complete v1/BRC-122 frame format

**Services:**
- [bitcoin-shard-proxy Architecture](https://github.com/lightwebinc/bitcoin-shard-proxy/blob/main/docs/architecture.md)
- [bitcoin-shard-listener Architecture](https://github.com/lightwebinc/bitcoin-shard-listener/blob/main/docs/architecture.md)
- [bitcoin-retry-endpoint README](https://github.com/lightwebinc/bitcoin-retry-endpoint)

**Infrastructure:**
- [bitcoin-ingress Architecture](https://github.com/lightwebinc/bitcoin-ingress/blob/main/docs/architecture.md)
- [bitcoin-listener Architecture](https://github.com/lightwebinc/bitcoin-listener/blob/main/docs/architecture.md)
- [bitcoin-retransmission Architecture](https://github.com/lightwebinc/bitcoin-retransmission/blob/main/docs/architecture.md)

### Conceptual Attribution

The IPv6 multicast transaction broadcast architecture from which this software draws inspiration was articulated by Dr. Craig S. Wright:

- [Multicast within Multicast: Anycast](https://singulargrit.substack.com/p/multicast-within-multicast-anycast)
- [Multicast as the Only Viable Architecture](https://singulargrit.substack.com/p/multicast-as-the-only-viable-architecture)
- [Singulargrit Substack](https://singulargrit.substack.com/)

### Standards

**BRC-12: Raw Transaction Format**
- The v1 wire-frame format transports transactions conforming to BRC-12
- [BSV Blockchain Standards Repository](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0012.md)

---

## Appendix: Quick Reference

### Default Ports

```text
| Service                            | Port | Protocol | Purpose                |
|------------------------------------|------|----------|------------------------|
| bitcoin-shard-proxy (UDP ingress)  | 9000 | UDP      | Frame ingress          |
| bitcoin-shard-proxy (TCP ingress)  | 9100 | TCP      | Reliable frame ingress |
| bitcoin-shard-proxy (egress)       | 9001 | UDP      | Multicast egress       |
| bitcoin-shard-listener (multicast) | 9001 | UDP      | Multicast receive      |
| bitcoin-shard-listener (NACK)      | 9300 | UDP      | NACK send              |
| bitcoin-retry-endpoint (multicast) | 9001 | UDP      | Multicast receive      |
| bitcoin-retry-endpoint (NACK)      | 9300 | UDP      | NACK receive           |
| bitcoin-retry-endpoint (retransmit)| 9100 | UDP      | Retransmission egress  |
```

### Metrics Ports

```text
| Service                | Port | Endpoint                           |
|------------------------|------|------------------------------------|
| bitcoin-shard-proxy    | 9100 | `/metrics`, `/healthz`, `/readyz`  |
| bitcoin-shard-listener | 9200 | `/metrics`, `/healthz`, `/readyz`  |
| bitcoin-retry-endpoint | 9400 | `/metrics`, `/healthz`, `/readyz`  |
```

### Default AS Numbers

```text
| Service                 | AS    |
|-------------------------|-------|
| bitcoin-ingress (proxy) | 65001 |
| bitcoin-listener        | 65002 |
```

### Frame Version Summary

```text
| Version | Header Size | Sequence Support | Subtree Support | SenderID |
|---------|-------------|------------------|-----------------|----------|
| v1      | 44 bytes    | No               | No              | No       |
| BRC-122 | 92 bytes    | Yes              | Yes             | Yes      |
```

---

*Document Version: 1.0*  
*Last Updated: 2026-04-22*
