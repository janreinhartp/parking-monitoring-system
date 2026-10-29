# Parking Monitoring System: LoRa Radio Protocol

v1 contract between the sensor-node firmware and the Raspberry Pi gateway. State rules and timing defaults live in [system-architecture.md](system-architecture.md). This document defines bytes on the air.

## 1. Scope

- Private point-to-point LoRa between Wio-E5 mini nodes and a LoRa HAT on the Raspberry Pi. Not LoRaWAN.
- v1 message types are `STATUS`, `HEARTBEAT`, and `ACK`.
- `CONFIG` and `CONFIG_ACK` are not part of v1. Commission node ID, thresholds, and radio settings over the debug serial link.
- The radio's hardware CRC is required once PHY settings are chosen. v1 adds no second checksum.

## 2. PHY parameters

Every node and the gateway HAT must use the same PHY configuration. The numeric values are unset until Philippine NTC frequency, power, and airtime rules are confirmed and the HAT is selected. Do not invent channel numbers in firmware.

Parameters that must match, and that must be loaded from configuration:

| Parameter | Status |
|---|---|
| Center frequency | Unset pending NTC and HAT selection |
| Bandwidth | Unset pending NTC and HAT selection |
| Spreading factor | Unset pending NTC and HAT selection |
| Coding rate | Unset pending NTC and HAT selection |
| Sync word | Unset pending NTC and HAT selection |
| Preamble length | Unset pending NTC and HAT selection |
| Header mode (explicit or implicit) | Unset pending NTC and HAT selection |
| Hardware CRC | Required on |
| Transmit power | Unset pending NTC and HAT selection |
| IQ inversion | Same value on nodes and gateway |

Calculate packet airtime from the chosen PHY before changing heartbeat interval or retry timing. The factory Wio-E5 AT firmware is not this protocol. Node firmware must drive the STM32WLE5 radio directly.

## 3. Byte order and version

Multi-byte integers are little-endian. `version` is `1`. Receivers drop any packet whose version is not `1`.

### 3.1 Header

Every v1 packet starts with this header.

| Offset | Size | Field | Description |
|---:|---:|---|---|
| 0 | 1 | `version` | `1` |
| 1 | 1 | `type` | Message type code |
| 2 | 1 | `node_id` | Node identifier. `0` is invalid |
| 3 | 2 | `sequence` | `uint16` sequence for this logical message |

### 3.2 Message types

| Name | Code | Direction | Length |
|---|---:|---|---:|
| `STATUS` | `1` | Node to gateway | 12 bytes |
| `HEARTBEAT` | `2` | Node to gateway | 12 bytes |
| `ACK` | `3` | Gateway to node | 5 bytes (header only) |

Any other type or any other length is malformed and is dropped.

## 4. STATUS and HEARTBEAT body

`STATUS` and `HEARTBEAT` share one body after the 5-byte header.

| Offset | Size | Field | Description |
|---:|---:|---|---|
| 5 | 1 | `state` | Occupancy state code |
| 6 | 2 | `distance_mm` | `uint16` distance in millimetres. `0xFFFF` means no valid distance |
| 8 | 2 | `battery_mv` | `uint16` battery voltage in millivolts. `0xFFFF` means unknown |
| 10 | 1 | `sensor_health` | Health code |
| 11 | 1 | `flags` | Bit flags. Reserved bits must be `0` |

### 4.1 Occupancy state codes

| Name | Code |
|---|---:|
| `VACANT` | `0` |
| `OCCUPIED` | `1` |
| `UNKNOWN` | `2` |
| `SENSOR_FAULT` | `3` |

### 4.2 Sensor health codes

| Name | Code | Meaning |
|---|---:|---|
| `OK` | `0` | Sensor responded with a usable reading |
| `NO_RESPONSE` | `1` | No UART response within the sensor timeout |
| `INVALID_READING` | `2` | Frame received but distance is out of the sensor's valid range or failed the node's checks |
| `POWER_FAULT` | `3` | Switched sensor supply or battery reading failed its check |

`SENSOR_FAULT` occupancy is sent with health other than `OK`. `UNKNOWN` occupancy may be sent with `OK` when readings disagree and the node cannot confirm a transition.

### 4.3 Flags

| Bit | Name | Meaning when set |
|---:|---|---|
| 0 | `BATTERY_LOW` | Battery is under the node's configured low-voltage threshold |
| 1 | `ACK_EXHAUSTED` | The previous `STATUS` ended its retries without an `ACK` |
| 2–7 | Reserved | Senders must set these bits to `0`. Receivers ignore them |

## 5. When each message is sent

- `STATUS`: after a confirmed occupancy transition. This is the only message that expects `ACK`.
- `HEARTBEAT`: while occupancy is unchanged, at the heartbeat interval (default 60 s). No `ACK`.
- The node does not transmit every 1 s measurement. Measurement interval and transmit interval are different. Defaults are in [system-architecture.md](system-architecture.md).

A new logical `STATUS` or `HEARTBEAT` increments `sequence` by 1, modulo 65536. Retries of the same `STATUS` reuse the same `sequence`. `ACK` does not consume a node sequence number.

The node stores the next `sequence` in non-volatile memory. After a reset it sends that stored value, then advances the stored value. A reset does not start again at `0` unless non-volatile memory was erased. If it was erased, the node starts at `0` and the gateway resynchronizes as described in section 6.

### 5.1 ACK

The gateway sends `ACK` only for a `STATUS` it accepts, including a duplicate `STATUS` it has already applied. The `ACK` header uses that packet's `node_id` and `sequence`. The gateway does not `ACK` heartbeats, malformed packets, or unknown node IDs.

`ACK` result is implied by the packet itself: an `ACK` means accepted. There is no negative acknowledgement in v1.

### 5.2 Retries

For one `STATUS` sequence the node transmits at most 3 times. The first transmission follows the transition. Each further attempt waits a random delay from 250 ms to 750 ms. If no `ACK` arrives after the third transmission, the node sets `ACK_EXHAUSTED` on its next `HEARTBEAT` and does not transmit that `STATUS` again. It may send a new `STATUS` when occupancy changes again. It may send the next `HEARTBEAT` on schedule.

The node must not retransmit continuously while the gateway is down.

### 5.3 Collision avoidance

Heartbeat transmit offset within each 60 s period:

```text
offset = (node_id - 1) * 2 s + random(0 ms, 500 ms)
```

`node_id` values in v1 are 1 through 10, so offsets stay inside the first 20 s of the period. Transition `STATUS` packets are sent when the transition is confirmed, then retried as in section 5.2. They are not held for the heartbeat slot.

## 6. Gateway receive rules

The gateway accepts only `STATUS` and `HEARTBEAT`. It drops a packet when any of these are true:

- Length is not 12 bytes
- `version` is not `1`
- `type` is not `STATUS` or `HEARTBEAT`
- `node_id` is `0` or is not in the configured bay map
- `state` is not one of the four codes
- `sensor_health` is not one of the four codes
- `distance_mm` is `0xFFFF` while `state` is `VACANT` or `OCCUPIED`

The node accepts only `ACK`. It drops a packet when the length is not 5 bytes, `version` is not `1`, `type` is not `ACK`, or `node_id` is not its own. An `ACK` whose `sequence` matches the `STATUS` in flight completes that attempt. Any other `ACK` is ignored.

The gateway stores, per configured node, the last accepted `sequence` in SQLite so a reboot does not treat an old replay as new.

Compare a candidate sequence with the stored sequence using unsigned 16-bit distance:

```text
delta = (candidate - stored) mod 65536
```

| `delta` | Treatment |
|---|---|
| `0` | Duplicate. Do not apply state again. If `type` is `STATUS`, send `ACK` again |
| `1` … `32767` | Newer. Apply the packet and store `candidate` |
| `32768` … `65535` | Stale. Drop it. Do not `ACK`, unless the resync rule below applies |

If the node has no stored sequence yet, the first packet that passes the structural checks is accepted and its sequence is stored.

If the bay is already `UNKNOWN` because of the stale timeout, the next structurally valid packet is accepted even when `delta` is in the stale range, and that packet's sequence replaces the stored sequence. `delta` of `0` stays a duplicate and does not refresh the bay. This resync covers a node whose sequence memory was erased. v1 has no application-layer MAC, so a forged packet can use the same path. The hardware CRC and the configured `node_id` list are the v1 checks.

Applying a packet updates the bay's state, distance, battery, health, flags, and last-valid-report time. The parking state manager then runs the stale-data rules from [system-architecture.md](system-architecture.md). A `HEARTBEAT` or `STATUS` whose state matches the stored state still refreshes last-valid-report time.

## 7. Cloud mapping

The gateway forwards accepted `STATUS` transitions and current slot state over HTTPS, as defined in [cloud-plan.md](cloud-plan.md). The cloud idempotency key is the gateway id, this `node_id`, and this `sequence`. Sequence numbers are per node and will collide across bays if the node id is omitted. Heartbeats refresh cloud freshness according to the sync rules. They are not occupancy events unless the applied state changed.

Radio PHY bytes are not sent to the cloud in v1.
