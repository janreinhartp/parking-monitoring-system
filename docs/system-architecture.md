# Parking Monitoring System: System Architecture

This document is the source of truth for architecture, state rules, defaults, and build order. Component plans describe hardware and implementation phases. They must follow this document when they overlap.

Related documents:

- [radio-protocol.md](radio-protocol.md) — packet contract between sensor nodes and the gateway
- [node-sensor-plan.md](node-sensor-plan.md) — solar sensor node
- [gateway-plan.md](gateway-plan.md) — Raspberry Pi gateway and HDMI dashboard
- [cloud-plan.md](cloud-plan.md) — cloud API, database, public site, and admin dashboard

## 1. Purpose and scale

Build a low-cost, solar-powered system that detects parking-bay occupancy, shows availability on a local HDMI display, and publishes status to a public website.

v1 covers one parking facility and 1–10 bays. The cloud schema may include a parking-lot record and a public slug so a second facility can be added later. v1 configures and operates a single lot.

The Raspberry Pi is the source of truth for immediate local operation. The cloud provides remote access and a durable copy of current status and occupancy history. The entrance display must work when the internet is down.

## 2. Locked decisions

| Decision | Choice |
|---|---|
| Sensor node | Seeed Studio Wio-E5 mini (STM32WLE5JC) and A02YYUW ultrasonic sensor |
| Radio | Private point-to-point LoRa. LoRaWAN is out of v1 |
| Receiver | LoRa HAT on the Raspberry Pi 4B. There is no separate receiver board |
| Gateway computer | Raspberry Pi 4B, 4 GB RAM |
| Local store | SQLite |
| Entrance display | Local webpage in a Chromium kiosk, read from the Pi |
| Cloud API | HTTPS to Cloudflare Workers |
| Cloud database | Supabase PostgreSQL |
| Admin authentication | Supabase Auth |
| Public site | Cloudflare Pages |
| Live updates | Poll the public API every 10 seconds. Supabase Realtime may replace polling later |
| Camera | Out of this project |
| Facility count | One lot in v1 |

PHY settings (frequency, bandwidth, spreading factor, coding rate, sync word, preamble, header mode, CRC, and transmit power) stay unset until Philippine NTC limits are confirmed and the gateway HAT is selected. [radio-protocol.md](radio-protocol.md) lists the parameters that must match. It does not assign channel numbers.

## 3. Architecture

```text
PARKING BAY SENSOR NODES
Wio-E5 mini + A02YYUW ultrasonic sensor
              |
              | Private point-to-point LoRa
              v
         LoRa HAT on the Pi
              |
              v
     RASPBERRY PI 4B (4 GB)
              |
              +-- LoRa receiver service
              +-- Parking state manager
              +-- SQLite
              +-- Local HTTP API
              +-- Chromium kiosk (HDMI)
              +-- Outbound sync queue
              |
              | HTTPS over Ethernet and 4G LTE
              v
     CLOUDFLARE WORKERS API
              |
              +-- Authenticate the gateway
              +-- Validate payloads
              +-- Write authorized data
              +-- Serve public parking status
              |
              v
     SUPABASE POSTGRESQL
              |
              v
     CLOUDFLARE PAGES
              |
              +-- Public parking page
              +-- Administrator dashboard
```

Internet, the 4G router, and the cloud are required only for the public website and remote administration. LoRa reception, SQLite, and the HDMI dashboard stay on the Pi.

## 4. Component responsibilities

### 4.1 Sensor node

Each bay has one node. The node measures distance, filters readings, decides occupancy, and sends compact LoRa packets. It reports distance, battery voltage, sensor health, and a sequence number. It sleeps between measurement and transmission cycles.

Commissioning (node ID, thresholds, radio settings, heartbeat interval) is done over the debug serial link. Over-the-air configuration is deferred. See [node-sensor-plan.md](node-sensor-plan.md) and [radio-protocol.md](radio-protocol.md).

### 4.2 Gateway

The gateway reads packets from the LoRa HAT, validates them, updates SQLite, serves the local dashboard, and synchronizes to the cloud over HTTPS. Cloud failures must not block packet handling or the dashboard.

Local API, bound to the Pi and not published on the internet:

| Endpoint | Purpose |
|---|---|
| `GET /api/status` | Facility summary and available count |
| `GET /api/slots` | Per-bay state and health |
| `GET /api/health` | Gateway, radio, and cloud-sync health |

### 4.3 Cloud

Workers authenticate the gateway, validate payloads, store current slot state and meaningful transitions, and serve public reads. The browser never receives service-role credentials and never writes parking state directly. Schema, endpoints, and acceptance criteria are in [cloud-plan.md](cloud-plan.md).

## 5. Parking state rules

| State | Meaning | Counts as available |
|---|---|---|
| `VACANT` | Confirmed empty | Yes |
| `OCCUPIED` | Confirmed occupied | No |
| `UNKNOWN` | No recent valid report, or the reading is inconclusive | No |
| `SENSOR_FAULT` | Sensor malfunction or invalid readings | No |

Rules:

1. Missing sensor data is never treated as `VACANT`.
2. `available_count` is the number of bays in `VACANT`.
3. The facility status is `FULL` only when every configured bay is `OCCUPIED`.
4. If any bay is `UNKNOWN` or `SENSOR_FAULT`, or any bay is past the stale threshold, the summary status is `INCOMPLETE`. The display must not claim the lot is definitely full or that every remaining bay is free.
5. A confirmed transition requires consecutive readings that agree. Entering and leaving `OCCUPIED` use separate thresholds. Thresholds are stored per bay on the node and set during commissioning. Millimeter cutoffs are not hardcoded in this spec.
6. The gateway and the cloud use the same four states. The cloud derives counts from stored slot rows. It does not trust counts sent by a browser.

### Freshness

Each bay has a last-valid-report time. After the stale threshold, the bay is `UNKNOWN` even if the last stored state was `VACANT` or `OCCUPIED`. The public API exposes a derived freshness status. Stored state must not be presented as live after that threshold.

The gateway records its own receipt time. The cloud records receipt time separately so clock skew can be seen.

### Summary status values

| Status | When |
|---|---|
| `AVAILABLE` | At least one bay is `VACANT`, and no bay is unknown, faulty, or stale |
| `FULL` | Every configured bay is `OCCUPIED`, and none are stale |
| `INCOMPLETE` | Any bay is `UNKNOWN`, `SENSOR_FAULT`, or stale |
| `OFFLINE` | The gateway's last cloud heartbeat is older than 180 s |

Evaluate in this order: `OFFLINE`, then `INCOMPLETE`, then `FULL`, then `AVAILABLE`. A missing gateway heartbeat is `OFFLINE` even if stored bay rows still say `OCCUPIED`.

`OFFLINE` is a cloud and website status. The HDMI dashboard, which reads the Pi directly, shows a radio or gateway fault instead of calling the cloud offline.

## 6. Data flow

1. The node measures about once per second.
2. Firmware filters the distance and updates the local occupancy state.
3. On a confirmed transition, the node sends a `STATUS` packet and waits for `ACK`. It transmits that sequence at most three times.
4. While occupancy is unchanged, the node sends an unacknowledged `HEARTBEAT` on the heartbeat interval.
5. The Pi validates the packet and updates SQLite.
6. The HDMI dashboard reads the local API and updates from SQLite.
7. The Pi sends occupancy changes to `POST /api/v1/gateway/sync` over HTTPS. It sends `POST /api/v1/gateway/heartbeat` on the cloud heartbeat interval.
8. Workers authenticate the gateway, update the current slot, and append a `slot_events` row when the state actually changes.
9. The public site polls `GET /api/v1/public/parking/:slug`.

Unchanged ultrasonic samples are not uploaded. Sequence numbers are per node, so the cloud idempotency key is gateway id, node id, and sequence. Repeated uploads of that triple must not create duplicate events. After an outage, the Pi uploads current slot state before any older diagnostic events.

## 7. Timing defaults

All of these are configuration, not fixed constants. These are the v1 starting values.

| Parameter | Default |
|---|---|
| Ultrasonic measure interval | 1 s |
| Node heartbeat | 60 s |
| Bay stale threshold | 180 s (three missed node heartbeats). The bay becomes `UNKNOWN` |
| Gateway offline threshold | 180 s (three missed cloud heartbeats). The public summary becomes `OFFLINE` |
| `STATUS` transmissions per sequence | At most 3 (the first send plus two retries). Jitter between attempts is 250–750 ms. Then stop until the next heartbeat or a new transition |
| Cloud heartbeat | 60 s |
| Public page poll | 10 s |
| Occupancy cloud sync | On each confirmed local transition |

Target occupancy update latency on the HDMI display is 1–3 seconds under normal radio conditions. That is a design target. It depends on radio settings, interference, packet loss, node count, and legal airtime limits. It is not a guarantee.

## 8. Offline behavior

| Failure | Required behavior |
|---|---|
| Internet or 4G outage | LoRa reception, SQLite, and the HDMI dashboard continue. Cloud updates are queued or coalesced and retried with exponential backoff |
| Workers or Supabase unavailable | Same as an internet outage for the sync path. The local display keeps working |
| Node stops reporting | Its bay becomes `UNKNOWN` after the stale threshold |
| Node reports a sensor fault | Its bay becomes `SENSOR_FAULT` |
| Pi restarts | Restore SQLite, then apply freshness checks. Persisted state is not shown as fresh until a new report arrives or the stale rule marks it `UNKNOWN` |
| LoRa HAT or receiver service fails | Dashboard shows a radio fault. Bays go stale on the existing timeout. Parking state is not invented |
| Sync credential rejected | Stop uploading, keep local operation, and surface the failure on the local health endpoint |

The website must show stale cloud data as stale. If the gateway heartbeat stops, the public page must not infer that the lot is full.

## 9. Security boundaries

- Authenticate every gateway sync and heartbeat. One credential per gateway. Store it outside source control. Support rotation and revocation.
- Validate packet layout, protocol version, node ID, and API payloads. Reject unknown node IDs and unknown states.
- Use HTTPS for cloud traffic.
- Keep service-role and gateway secrets on the Pi and in Workers. Never put them in frontend code.
- Restrict admin operations to authenticated accounts, enforced on the server.
- Apply database row-level security where the client can reach Supabase directly.
- Bind the Pi's local API to the device. Do not expose it on the public internet.
- Keep radio configuration inside verified NTC limits before any field transmission.

## 10. Non-goals

This project does not include:

- An IP camera, remote playback, cloud video, recording, or a video server
- LoRaWAN network servers
- A rented VPS for parking status
- Over-the-air node configuration in v1
- Multiple parking facilities in operation (the schema may still store a lot row)
- Treating free-tier hosting as a guaranteed production uptime plan

A camera could be a separate later project. It must not share the LoRa path, and it must not change occupancy behavior.

Free tiers are for development and initial testing. Confirm Supabase inactivity pauses, request quotas, backups, and uptime needs before relying on the public site continuously. Additional cloud hosting cost target during that initial period is ₱0 per month, excluding electricity, the 4G plan, and an optional domain.

## 11. Open gates before buying more hardware

These are blocked on measurement or regulation, not on an undecided product shape:

1. Confirm the Wio-E5 mini frequency variant and current Philippine NTC frequency, power, and airtime rules.
2. Select a Pi LoRa HAT that can use the same PHY and packet framing as the node firmware. Bench-test one node before buying the rest of the radios.
3. Measure Wio-E5 mini board current in sleep, sense, receive, and transmit, and measure sensor on/off current. The 6 V 10 W panel and 20 Ah LiFePO4 pack in the sensor plan stay preliminary until that energy budget exists.

## 12. Build order

1. **Radio gate.** Confirm NTC limits, select the HAT, and bench-test one node to the Pi. Do not buy the remaining nodes until that link works.
2. **One bay, local only.** Occupancy filtering, SQLite, local API, HDMI kiosk, stale and fault behavior, and reboot restore. No cloud required.
3. **Cloud sync.** Schema, gateway auth, heartbeat, sync, offline queue, and a duplicate-upload test.
4. **Public site.** Availability without login, incomplete and stale warnings, phone and desktop layouts.
5. **Admin.** Login, diagnostics, and slot-to-node mapping, with server-side authorization.
6. **Field.** Add nodes up to the planned count. Measure range, collisions, power loss, and a long internet outage.

## 13. Acceptance for the system as a whole

- Every configured node is received, and malformed or duplicate packets are rejected.
- Unknown and faulty bays are never counted as available.
- The HDMI display works through an internet outage and after a Pi reboot, with stale state identified.
- After connectivity returns, queued updates sync, and duplicate uploads do not duplicate events.
- The public site works without an account and shows incomplete or stale status honestly.
- Admin actions require authorization.
- Privileged credentials stay off the public site.
- The system provides parking status with no video infrastructure.
