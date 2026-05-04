# BRC-125 — Retransmission Protocol

BRC-125 defines the NACK-based retransmission and endpoint discovery protocol for the BSV multicast pipeline. It specifies the ADVERT beacon message, the MISS/ACK response messages, tier/preference-based endpoint selection, and configurable retransmit modes.

> **Status:** To be submitted as BRC-125 PR to jefflightweb/BRCs.

---

## Overview

Retry endpoints cache BRC-124 frames received via multicast and respond to NACK requests from listeners experiencing gaps. BRC-125 adds:

1. **ADVERT** — periodic multicast beacon advertising retry endpoint availability.
2. **ACK/MISS** — unicast responses to every NACK, enabling deterministic escalation.
3. **Tier/Preference** — hierarchical endpoint selection for multi-AS deployments.
4. **Configurable retransmit modes** — multicast, unicast, or both.

---

## ADVERT Wire Format (`MsgType 0x20`) — 56 bytes

Sent periodically (default **60 s**, configurable via `-beacon-interval`) to the site beacon group (`FF05::FF:FFFD`), global beacon group (`FF0E::FF:FFFD`), or both. Listeners derive TTL as `3 × BeaconInterval`.

```text
Offset  Size  Field
------  ----  -----
     0     4  Magic (0xE3E1F3E8)
     4     2  ProtoVer (0x02BF)
     6     1  MsgType = 0x20 (ADVERT)
     7     1  Scope  (0x05=site, 0x0E=global, 0xFF=both)
     8    16  NACKAddr      — IPv6 unicast address for NACK requests
    24     2  NACKPort      — UDP NACK listen port (default 9300)
    26     1  Tier          — operator-assigned; 0 = same AS as proxy (max 255)
    27     1  Preference    — weighting within a tier; higher = more preferred (default 128)
    28     2  BeaconInterval — seconds; listeners compute TTL = 3 × this value
    30     2  Flags         — see below
    32     4  InstanceID    — CRC32c of hostname (matches metrics InstanceID)
    36     4  Reserved
    40    16  Reserved      — future use (BGP community, capability bitmap)
```

### Flags Bitmask

| Bit      | Name               | Meaning |
|----------|--------------------|---------|
| `0x0001` | _(reserved)_       | Unused; was RedisBackedDedup (dropped — internal implementation detail) |
| `0x0002` | HasParent          | Upstream escalation endpoint configured (Phase 2) |
| `0x0004` | Draining           | Entering shutdown; stop routing new NACKs here |
| `0x0008` | UnicastRetransmit  | Supports unicast frame delivery to NACK source |
| `0x0010` | MulticastRetransmit| Retransmits via multicast (default on) |

**HasParent rationale:** When set, the endpoint forwards NACKs internally to a parent endpoint on local cache miss. Benefits: topology encapsulation (listeners need only know local endpoints), reduced global beacon traffic, faster recovery via co-located forwarding.

---

## NACK Wire Format (`MsgType 0x10`) — 56 bytes

Unchanged from existing implementation. Sent by listener to retry endpoint on gap detection.

```text
Offset  Size  Field
------  ----  -----
     0     4  Magic (0xE3E1F3E8)
     4     2  ProtoVer (0x02BF)
     6     1  MsgType = 0x10 (NACK)
     7     1  Reserved = 0x00
     8    32  TxID               — identifies the missing frame (best-effort)
    40     4  SenderID           — CRC32c of sender IPv6; 0 = unknown
    44     4  SequenceID         — flow identifier; 0 = unknown
    48     4  SeqNum             — sender's sequence number; 0 = unknown
    52     4  Reserved           — padding; must be 0x00000000
```

---

## MISS Response (`MsgType 0x11`) — 24 bytes

Sent unicast to the NACK source when the requested frame is not in cache. Listener advances to next endpoint immediately (no backoff wait).

```text
Offset  Size  Field
------  ----  -----
     0     4  Magic (0xE3E1F3E8)
     4     2  ProtoVer (0x02BF)
     6     1  MsgType = 0x11 (MISS)
     7     1  Flags (reserved 0x00)
     8     4  SenderID    — echoed from NACK
    12     4  SequenceID  — echoed from NACK
    16     4  SeqNum      — echoed from NACK
    20     4  Reserved
```

---

## ACK Response (`MsgType 0x12`) — 24 bytes

Sent unicast to the NACK source when the frame is found and retransmit dispatched. Listener suppresses further NACKs for this gap immediately.

```text
Offset  Size  Field
------  ----  -----
     0     4  Magic (0xE3E1F3E8)
     4     2  ProtoVer (0x02BF)
     6     1  MsgType = 0x12 (ACK)
     7     1  Flags (0x01=multicast_sent, 0x02=unicast_sent)
     8     4  SenderID    — echoed from NACK
    12     4  SequenceID  — echoed from NACK
    16     4  SeqNum      — echoed from NACK
    20     4  Reserved
```

`ACK.Flags` tells the listener which delivery method was used, allowing it to either wait for the multicast fill or accept the unicast frame directly.

---

## Tier / Preference Model

| Tier | Meaning |
|------|---------|
| 0    | Same AS as `bitcoin-shard-proxy` (source-adjacent) |
| 1    | One AS boundary from source |
| N    | N hops from source |
| 0xFF | Static seed (no beacon received; lowest priority) |

Operator assigns `-tier` (0–254) and `-preference` (0–255, default 128) on each `bitcoin-retry-endpoint`. Endpoints are sorted by `(Tier ASC, Preference DESC)` — higher-preference endpoints are tried first within a tier.

### Escalation State Machine

```text
                    ┌──────────────────────────────────────┐
                    │                                      │
  ┌─────────┐  dispatch  ┌──────────────┐  ACK    ┌───────▼───────┐
  │ PENDING ├──────────►│ NACKED(Tier-K)├────────►│  GAP CANCELLED │
  └─────────┘           └──────┬───────┘          └───────────────┘
                               │                          ▲
                    MISS       │  Timeout                  │
                    ┌──────────┘  ┌──────┘                 │
                    ▼             ▼                         │
              ┌───────────┐  ┌──────────┐   multicast fill │
              │ advance   │  │ backoff  │──────────────────┘
              │ endpoint  │  │ & retry  │
              └─────┬─────┘  └──────────┘
                    │
                    ▼
              next endpoint at same tier,
              or escalate to next tier
```

- **ACK received** → cancel gap entry immediately.
- **MISS received** → advance to next endpoint at same tier (by Preference); if tier exhausted, escalate to next tier; retry immediately (no backoff).
- **Timeout** → apply exponential backoff; next sweep retries.
- **Multicast fill** (independent receive goroutine) → cancel gap regardless of NACK state.

---

## Configurable Retransmit Modes

| Flag | Default | Meaning |
|------|---------|---------|
| `-retransmit-multicast` | `true` | Send cached frame to multicast group on NACK hit |
| `-retransmit-unicast`   | `false` | Send cached frame unicast to NACK source on NACK hit |
| `-suppress-miss`        | `false` | Do not send MISS responses |
| `-suppress-ack`         | `false` | Do not send ACK responses |

### Deployment profiles

- **On-fabric (default):** multicast retransmit + ACK + MISS
- **Edge endpoint:** `-retransmit-unicast=true -retransmit-multicast=false`
- **High-volume:** `-suppress-ack=true` to reduce response traffic; MISS preserved for escalation

---

## Flood Prevention

- **Multicast fill suppression:** retransmits go to multicast; all listeners receive them; `Tracker.Fill()` cancels pending NACKs.
- **SequenceIDRetransmit marker:** retransmitted frames are tagged; ingress drops them from recaching so endpoints don't re-retransmit.
- **Redis SET NX dedup:** only one endpoint per site retransmits any given frame (60 s window).
- **Inter-AS:** MP-BGP propagates retransmits; remote listeners fill before backoff fires.

---

## Implementation

- **Listener:** `bitcoin-shard-listener/nack/wire.go` (NACK encode/decode), `bitcoin-shard-listener/discovery/` (ADVERT decode, registry, beacon listener)
- **Endpoint:** `bitcoin-retry-endpoint/server/server.go` (NACK receive, ACK/MISS send), `bitcoin-retry-endpoint/beacon/` (ADVERT encode/send)
- **Common:** `bitcoin-shard-common/frame/` (MsgType constants)
