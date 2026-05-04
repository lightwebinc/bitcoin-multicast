# BRC-124 — Data-Plane Frame Format

BRC-124 defines the wire format for transporting BSV transactions over IPv6 multicast and TCP/UDP unicast. This document is the canonical reference for the 92-byte BRC-124 header and the 44-byte legacy BRC-12 (v1) header.

> **Status:** Current BRC for the data-plane frame format.

---

## BRC-124 Frame Format (92-byte header)

All multi-byte integers are big-endian. 8-byte alignment for all fields after offset 8.

```text
Offset  Size  Align  Field                 Value / Notes
------  ----  -----  -----                 -------------
     0     4   —     Network magic         0xE3E1F3E8 (BSV mainnet P2P magic)
     4     2   —     Protocol ver          0x02BF = 703 (BSV node version baseline)
     6     1   —     Frame version         0x02 (BRC-124)
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

### Field Details

- **Network magic (0:4):** BSV mainnet P2P magic; enables standard firewall classification.
- **Protocol version (4:6):** Informational; 703 = BSV large-block policy baseline.
- **Frame version (6):** `0x02` for BRC-124, `0x01` for legacy BRC-12.
- **Transaction ID (8:40):** Raw 256-bit txid in internal byte order (NOT display-reversed).
- **Sender ID (40:44):** CRC32c checksum of source IPv6 address (Castagnoli polynomial `0x1EDC6F41`). Enables per-sender gap tracking with minimal header overhead. Given BSV's mining economics, collision probability is ~0.01% at 1,000 senders, negligible at 100 senders.
- **Sequence ID (44:48):** Random temporal flow identifier; reset periodically by sender; combined with Sender ID and Shard Sequence Number for retransmission key `(SenderID, SequenceID, SeqNum)`.
- **Shard Sequence Number (48:52):** Monotonic counter per sender; enables gap detection and NACK-based retransmission.
- **Reserved (52:56):** Padding for 8-byte alignment of Subtree ID; must be zero.
- **Subtree ID (56:88):** Opaque 32-byte batch identifier for subtree-level filtering.
- **Payload (92+):** BSV transaction in BRC-12 raw format (version LE32 + inputs + outputs + locktime LE32).

---

## BRC-12 / v1 Frame Format (Legacy — 44-byte header)

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

---

## Frame Processing Rules

### Proxy (`bitcoin-shard-proxy`)

- Decode header (v1 or BRC-124); drop on bad magic or unknown version.
- For BRC-124: stamp Sender ID in-place at bytes 40–43 from ingress source address CRC32c.
- Forward verbatim to all egress interfaces (no re-encoding).

### Listener (`bitcoin-shard-listener`)

- Decode header; apply shard filter (group index).
- Apply subtree filter (SubtreeID include/exclude).
- For BRC-124 with non-zero Sender ID and Shard Sequence Number: track gaps per `(SenderID, groupIdx, SequenceID)`.
- Forward matching frames to egress address (UDP or TCP).

### Retry Endpoint (`bitcoin-retry-endpoint`)

- Receive multicast frames; decode header.
- Build cache key from `(SenderID, SequenceID, SeqNum)`.
- Store raw frame in cache for NACK-based retransmission.

---

## Backward Compatibility

- v1 frames are decoded with zero-valued BRC-124-only fields (`SenderID = 0`, `SequenceID = 0`, `SeqNum = 0`, `SubtreeID = zeros`).
- The forwarder (proxy) forwards v1 frames verbatim — no upgrade to BRC-124 encoding.
- Unknown frame versions are dropped with `ErrBadVer`.
- All components accept both v1 and BRC-124 frames on the wire.

---

## Implementation

- **Canonical source:** `bitcoin-shard-common/frame/frame.go`
- **Constants:** `MagicBSV = 0xE3E1F3E8`, `ProtoVer = 0x02BF`, `FrameVerV1 = 0x01`, `FrameVerBRC124 = 0x02`, `HeaderSizeLegacy = 44`, `HeaderSize = 92`
