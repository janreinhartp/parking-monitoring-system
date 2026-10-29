# Parking Monitoring System: LoRa Gateway Plan

## 1. Project Overview

Build a local-first parking gateway for a system supporting 1–10 parking bays.

The gateway uses a Raspberry Pi 4B (4 GB RAM) with a directly connected LoRa radio to receive occupancy data from Wio-E5 mini sensor nodes.

Shared rules live in [system-architecture.md](system-architecture.md). Packet bytes live in [radio-protocol.md](radio-protocol.md). The cloud API lives in [cloud-plan.md](cloud-plan.md). This document covers gateway hardware, local software, and gateway test phases.

### Main objectives

- Receive parking occupancy updates over private point-to-point LoRa.
- Maintain the current state of every parking bay.
- Display parking availability on an HDMI screen.
- Synchronize parking data to Cloudflare Workers over HTTPS through a 4G LTE router.
- Continue operating locally during internet outages.

## 2. System Architecture

```text
PARKING BAY SENSOR NODES
Wio-E5 mini + A02YYUW ultrasonic sensor
              |
              | Private point-to-point LoRa
              v
       LoRa Radio / HAT
              |
              | Supported interface
              v
     RASPBERRY PI 4B (4 GB)
              |
              +-- LoRa Receiver Service
              +-- Parking State Manager
              +-- SQLite Database
              +-- Local HTTP API
              +-- Chromium Kiosk Dashboard
              |
              +-- HDMI Display
              |
              +-- Ethernet
                      |
                4G LTE Router
                      |
                   Internet
                      |
            Cloudflare Workers API
                      |
               Public website
```

The LoRa HAT on the Raspberry Pi is the receiver. There is no separate Wio-E5 receiver board. The sensor nodes remain Wio-E5 mini modules.

## 3. Hardware Requirements

| Component | Purpose |
|---|---|
| Raspberry Pi 4B, 4 GB RAM | Gateway computer and local dashboard host |
| Compatible LoRa radio/HAT | Receives sensor node packets |
| Frequency-matched antenna | LoRa communication |
| HDMI display | Entrance parking availability display |
| 4G LTE router | Internet connectivity |
| 32–64 GB high-endurance microSD | Operating system, database and logs |
| Stable Raspberry Pi power supply | Reliable gateway operation |
| Enclosure and mounting hardware | Hardware protection |

### LoRa Radio Selection

Evaluate a Raspberry Pi LoRa HAT based on the SX1262 family.

Reference links:

- Waveshare SX1262 LoRa HAT: https://www.waveshare.com/sx1262-868m-lora-hat.htm
- Waveshare LoRa HAT documentation: https://www.waveshare.com/wiki/SX1268_470M_LoRa_HAT

These are evaluation references, not confirmation of compatibility with the Wio-E5 mini.

Before purchasing, verify that the exact gateway board supports the radio configuration and packet protocol planned for the sensor nodes. Some UART LoRa modules use proprietary firmware and serial commands rather than exposing raw LoRa packet access.

Confirm the permitted operating frequency, channel, transmit power and airtime requirements under current Philippine NTC regulations. Select the radio variant and antenna accordingly.

## 4. LoRa Communication Protocol

Use the v1 contract in [radio-protocol.md](radio-protocol.md). That document defines the 12-byte `STATUS` and `HEARTBEAT` layout, the 5-byte `ACK`, sequence rules, and retry limits.

PHY numbers stay unset until NTC limits and the HAT choice are confirmed. The gateway and the nodes must load the same configuration for frequency, bandwidth, spreading factor, coding rate, sync word, preamble length, header mode, hardware CRC, and transmit power.

v1 has no over-the-air `CONFIG` message. Bay count, node IDs, and the stale timeout are local gateway configuration.

## 5. Parking-Bay State Management

Each parking bay must have one of four states.

| State | Description | Available Count |
|---|---|---|
| `VACANT` | Confirmed empty parking bay | Increment |
| `OCCUPIED` | Confirmed occupied parking bay | No increment |
| `UNKNOWN` | No recent valid data or uncertain state | No increment |
| `SENSOR_FAULT` | Sensor failure or invalid measurements | No increment |

### State Rules

1. Never interpret missing sensor data as `VACANT`.
2. Track the last valid report timestamp for every bay.
3. Mark a bay as `UNKNOWN` after the stale-data timeout. The default is 180 s, from [system-architecture.md](system-architecture.md).
4. Use sensor filtering and hysteresis to reduce state changes caused by noise.
5. Require multiple consistent readings before confirming occupancy transitions.
6. Allow local configuration of the number of parking bays and node IDs.
7. Store the most recent state and diagnostics in SQLite.

### Available Parking Count

```text
available_count = number of bays in VACANT state
```

Use the summary statuses in [system-architecture.md](system-architecture.md): `AVAILABLE`, `FULL`, and `INCOMPLETE`. `FULL` only when every configured bay is `OCCUPIED` and none are stale. `INCOMPLETE` when any bay is `UNKNOWN`, `SENSOR_FAULT`, or stale. Missing sensor data is never shown as every bay occupied or every bay free.

## 6. Gateway Software Architecture

### 6.1 LoRa Receiver Service

Responsibilities:

- Read packets from the LoRa radio using its supported interface.
- Validate packet structure and protocol version.
- Validate node IDs.
- Reject malformed, duplicate and stale messages.
- Record reception timestamps.
- Record radio diagnostics when available.
- Update the parking state manager.
- Send `ACK` only for an accepted `STATUS`, including a duplicate `STATUS`, as specified in [radio-protocol.md](radio-protocol.md).

### 6.2 Parking State Manager

Responsibilities:

- Maintain the latest state of each bay.
- Track each node's last-seen timestamp.
- Track battery voltage and sensor health.
- Apply stale-data rules.
- Calculate the available parking count.
- Store state in SQLite.
- Provide data to the local dashboard and cloud synchronization service.

Suggested local API endpoints:

```text
GET /api/status
GET /api/slots
GET /api/health
```

Endpoint purposes:

- `/api/status`: Overall parking status and available count.
- `/api/slots`: State and health information for each bay.
- `/api/health`: Gateway, radio and cloud connection health.

### 6.3 SQLite Database

Store:

- Parking bay configuration
- Node IDs
- Current occupancy state
- Last valid report timestamp
- Battery voltage
- Sensor health
- Latest measured distance
- Relevant diagnostic events
- Pending cloud updates, when required

The database must remain usable without an internet connection.

### 6.4 HDMI Dashboard

Run a local webpage in Chromium kiosk mode.

Display:

- Total parking bays
- Available spaces
- Occupied spaces
- Individual bay status
- Last update time
- Gateway health
- Radio connection status
- Internet connection status
- Sensor fault warnings

The dashboard must read from the local API and must not depend on cloud services to display local parking information.

### 6.5 Cloud Synchronization Service

Use HTTPS to Cloudflare Workers. Endpoints and idempotency rules are in [cloud-plan.md](cloud-plan.md) and [system-architecture.md](system-architecture.md).

Responsibilities:

- `POST /api/v1/gateway/sync` when a confirmed occupancy state changes.
- `POST /api/v1/gateway/heartbeat` on the cloud heartbeat interval (default 60 s).
- Queue or coalesce pending updates during internet outages.
- Retry with exponential backoff.
- Upload current slot state before older diagnostic events when connectivity returns.
- Authenticate the gateway.
- Keep credentials out of source control.
- Avoid blocking the local dashboard or state manager.

## 7. Update Timing and Reliability

Target occupancy update latency: 1–3 seconds under normal radio conditions.

This is a design target, not a guarantee. Actual performance depends on radio configuration, interference, packet loss, number of nodes and legal airtime constraints.

Recommended behavior:

- Send a report after a confirmed occupancy transition.
- Send periodic heartbeats.
- Keep packets small.
- Use bounded retries.
- Configure a stale-data timeout.
- Avoid continuous retries during gateway outages.
- Test packet collisions and latency with all planned nodes operating together.

## 8. Website integration

The public website reads parking availability from the Cloudflare Workers API, as specified in [cloud-plan.md](cloud-plan.md). The entrance display reads the Pi's local parking state and does not call the cloud.

An IP camera is out of this project. See the non-goals in [system-architecture.md](system-architecture.md).

## 9. Internet Outage Behavior

| Failure | Required Behavior |
|---|---|
| Internet or 4G outage | Local LoRa reception and HDMI dashboard continue |
| Workers or Supabase unavailable | Queue or coalesce pending updates and retry later |
| Sensor node stops reporting | Mark its bay `UNKNOWN` after timeout |
| Sensor reports a fault | Mark its bay `SENSOR_FAULT` |
| Raspberry Pi restarts | Restore saved data and apply freshness checks |
| LoRa receiver fails | Show a gateway or radio fault warning |

Persisted data must not be displayed as fresh indefinitely. After startup, use last-seen timestamps to identify stale states until fresh reports arrive.

## 10. Security and Maintenance

- Assign a unique ID to each sensor node.
- Validate packet size, version and node ID.
- v1 authenticates a packet by hardware CRC, protocol version, and a configured `node_id`. An application-layer MAC is not part of the v1 packet.
- Use TLS for cloud communication.
- Store cloud credentials securely.
- Restrict the local API to trusted networks unless remote access is required.
- Rotate logs and prevent the microSD card from filling.
- Configure automatic restart for gateway services.
- Document configuration and backup procedures.
- Provide local configuration for bay count, node IDs and stale timeout.

## 11. Implementation Plan

### Phase 1: Confirm Radio Compatibility

- Select the exact Wio-E5 mini node firmware.
- Verify the permitted Philippine operating frequency and radio limits.
- Select a gateway radio that supports matching LoRa parameters and packet framing.
- Confirm the Raspberry Pi interface and driver.
- Bench-test one node with one gateway radio before purchasing the remaining hardware.

### Phase 2: Configure Raspberry Pi

- Install Raspberry Pi OS.
- Configure Ethernet connectivity to the 4G router.
- Configure the LoRa radio interface.
- Test packet reception.
- Configure services to start automatically after reboot.

### Phase 3: Implement Parking State Management

- Finalize the packet format.
- Implement packet validation and duplicate detection.
- Implement SQLite storage.
- Implement stale-data handling.
- Test `VACANT`, `OCCUPIED`, `UNKNOWN` and `SENSOR_FAULT`.

### Phase 4: Build HDMI Dashboard

- Build the local dashboard webpage.
- Connect it to the local API.
- Configure Chromium kiosk mode.
- Test startup and recovery.
- Verify that the display works without internet access.

### Phase 5: Integrate Cloud

- Implement the HTTPS client for the Workers heartbeat and sync endpoints.
- Implement offline queueing.
- Implement reconnect behavior.
- Test internet disconnection and restoration.

### Phase 6: Field Testing

- Test one sensor node.
- Expand to several nodes.
- Test up to ten nodes.
- Measure latency, packet loss and radio range.
- Test stale-state behavior and sensor faults.
- Test power interruptions, reboots and extended internet outages.
- Validate antenna placement and enclosure protection.

## 12. Acceptance Criteria

The gateway is ready for deployment when:

- It receives valid packets from every configured node.
- It rejects malformed and duplicate packets.
- Occupancy updates meet the measured performance target.
- Unknown and faulty bays are never counted as vacant.
- The HDMI display works during internet outages.
- Cloud synchronization resumes after internet connectivity returns.
- Services restart automatically after a reboot.
- Stale states are identified correctly after startup.
- Radio hardware, antenna and configuration comply with verified local regulations.

## 13. Decisions still open

These are already decided and are not open:

- Transport is HTTPS to Cloudflare Workers.
- The v1 packet format is [radio-protocol.md](radio-protocol.md).
- Starting heartbeat and stale timeouts are in [system-architecture.md](system-architecture.md). They stay configurable.

Still open before field installation:

1. Exact LoRa HAT or radio module.
2. Verified Philippine frequency, channel, power, and airtime rules, which set the PHY numbers.
3. Display size, brightness, and installation location.
4. Workers base URL for the HTTPS sync client.

## Final Recommendation

Use the Raspberry Pi 4B as the local-first gateway computer with a directly attached, protocol-compatible LoRa radio. Keep parking-state management and the HDMI dashboard local. Use the cloud for remote website synchronization, not as a dependency for entrance display operation.
