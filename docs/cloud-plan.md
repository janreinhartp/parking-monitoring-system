# Parking Monitoring System: Cloud Architecture and Implementation Plan

## 1. Project Overview

Build a low-cost cloud backend and public website for a parking monitoring system supporting 1–10 parking bays in one facility.

Transport, parking-state rules, timing defaults, and non-goals are defined in [system-architecture.md](system-architecture.md). This document covers the cloud schema, API, website, and cloud test phases.

The system uses a Raspberry Pi 4B as the local gateway. It receives occupancy updates from Wio-E5 mini sensor nodes over private point-to-point LoRa, maintains local parking state, and synchronizes data with the cloud through a 4G LTE router.

The public website displays real-time parking availability without requiring visitors to create an account.

### Primary objectives

- Minimize recurring hosting costs.
- Provide a public parking availability website.
- Synchronize parking status from the Raspberry Pi to the cloud.
- Store current occupancy, gateway health and occupancy history.
- Support a private administrator dashboard.
- Keep local parking operations working during internet outages.
- Avoid paid video streaming, camera playback and video hosting.
- Start with free hosting tiers while allowing future migration to paid services.

## 2. Recommended Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Public website | Cloudflare Pages | Host the public parking page |
| Backend API | Cloudflare Workers | Validate gateway updates and serve public status |
| Database | Supabase PostgreSQL | Store parking configuration, current status and event history |
| Authentication | Supabase Auth | Protect administrator functions |
| Realtime updates | Supabase Realtime or lightweight polling | Refresh public status |
| Local gateway | Raspberry Pi 4B | Maintain local state and synchronize with the cloud |
| Local storage | SQLite | Preserve parking data during internet outages |
| Communication | HTTPS | Secure gateway-to-cloud communication |
| Domain | Optional custom domain | Provide a recognizable public URL |

Reference documentation:

- Cloudflare Pages: https://developers.cloudflare.com/pages/
- Cloudflare Workers: https://developers.cloudflare.com/workers/
- Supabase: https://supabase.com/docs
- Supabase pricing: https://supabase.com/pricing
- Supabase Auth: https://supabase.com/docs/guides/auth
- Supabase Realtime: https://supabase.com/docs/guides/realtime

Free-tier availability, limits and service terms must be checked before deployment. Supabase Free projects can pause after a period of inactivity, so verify whether the selected plan meets production uptime requirements.

## 3. System Architecture

```text
PARKING BAY SENSOR NODES
Wio-E5 mini + A02YYUW ultrasonic sensors
              |
              | Private point-to-point LoRa
              v
RASPBERRY PI 4B GATEWAY
              |
              +-- LoRa receiver service
              +-- Parking state manager
              +-- SQLite database
              +-- Local HDMI dashboard
              +-- Outbound synchronization queue
              |
              | HTTPS over Ethernet and 4G LTE
              v
CLOUDFLARE WORKERS API
              |
              +-- Validate gateway identity
              +-- Validate incoming payloads
              +-- Apply rate limits
              +-- Write authorized data to Supabase
              +-- Serve public parking status
              |
              v
SUPABASE POSTGRESQL
              |
              +-- Parking lots
              +-- Parking slots
              +-- Gateways
              +-- Occupancy events
              +-- Administrator accounts
              |
              v
CLOUDFLARE PAGES
              |
              +-- Public parking page
              +-- Administrator dashboard
```

The Raspberry Pi remains the source of truth for immediate local operation. The cloud provides remote access and persistence.

The cloud must not be required for the entrance HDMI display to function.

## 4. Cloud Hosting Strategy

### 4.1 Initial deployment

Use free tiers wherever practical:

- Cloudflare Pages for frontend hosting.
- Cloudflare Workers for API functions.
- Supabase Free for PostgreSQL, authentication and optional realtime notifications.
- The existing Raspberry Pi for local processing and the entrance display.

Target additional cloud hosting cost: ₱0 per month during development and initial testing, within free-tier limits.

This target excludes internet service, electricity, an optional domain and any paid services required for production availability.

### 4.2 Cost-control rules

- Do not rent a VPS for the initial parking-status system.
- Do not host a separate video server.
- Do not send repeated unchanged sensor measurements to the cloud.
- Send occupancy changes immediately and periodic heartbeats at a configurable interval.
- Keep payloads small.
- Retain only useful event history.
- Avoid high-frequency polling when realtime notifications or slower polling are sufficient.
- Monitor database size, API requests, bandwidth and function usage.
- Avoid relying on free tiers for guaranteed uninterrupted production service.

### 4.3 Camera exclusion

The IP camera is out of this project.

Do not implement:

- Remote video playback
- MediaMTX hosting
- Cloudflare Stream
- Video recording or storage
- Camera streaming APIs
- Video bandwidth monitoring

The authoritative non-goal list is in [system-architecture.md](system-architecture.md). A camera could be a separate later project. It is not part of this cloud plan.

## 5. Database Design

Use PostgreSQL in Supabase.

All tables should use primary keys, foreign keys, timestamps and appropriate indexes.

### 5.1 `parking_lots`

Stores parking facility information.

Suggested fields:

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `name` | TEXT | Public parking location name |
| `slug` | TEXT | Unique URL-friendly identifier |
| `description` | TEXT, nullable | Optional public description |
| `timezone` | TEXT | Local timezone, default `Asia/Manila` |
| `is_active` | BOOLEAN | Whether the facility is active |
| `created_at` | TIMESTAMPTZ | Creation time |
| `updated_at` | TIMESTAMPTZ | Last modification time |

The public page uses the slug to identify the parking location.

### 5.2 `gateways`

Stores gateway identity and health.

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `parking_lot_id` | UUID | Associated parking lot |
| `name` | TEXT | Gateway display name |
| `last_seen_at` | TIMESTAMPTZ, nullable | Last valid gateway heartbeat |
| `software_version` | TEXT, nullable | Gateway software version |
| `status` | TEXT | Gateway health status |
| `created_at` | TIMESTAMPTZ | Creation time |

Use a constrained set of values for `status`, such as `ONLINE`, `OFFLINE` and `FAULT`.

### 5.3 `parking_slots`

Stores current state for each parking bay.

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `parking_lot_id` | UUID | Associated parking lot |
| `gateway_id` | UUID | Responsible gateway |
| `node_id` | TEXT | Unique sensor node identifier |
| `label` | TEXT | Public bay label |
| `state` | TEXT | Current occupancy state |
| `distance_mm` | INTEGER, nullable | Latest valid distance measurement |
| `battery_mv` | INTEGER, nullable | Latest reported battery voltage |
| `sensor_health` | TEXT | Sensor health status |
| `last_seen_at` | TIMESTAMPTZ, nullable | Last valid sensor report |
| `updated_at` | TIMESTAMPTZ | Last state update |

Allowed occupancy states:

- `VACANT`
- `OCCUPIED`
- `UNKNOWN`
- `SENSOR_FAULT`

A unique constraint should prevent duplicate node IDs within the applicable parking facility or gateway configuration.

### 5.4 `slot_events`

Stores occupancy transitions and important faults.

| Field | Type | Description |
|---|---|---|
| `id` | BIGINT or UUID | Primary key |
| `parking_lot_id` | UUID | Associated parking lot |
| `slot_id` | UUID | Associated parking bay |
| `previous_state` | TEXT, nullable | State before transition |
| `new_state` | TEXT | New state |
| `source_sequence` | BIGINT | Node sequence number for this event |
| `event_time` | TIMESTAMPTZ | Event time |
| `received_at` | TIMESTAMPTZ | Cloud receipt time |

Only store meaningful state transitions and selected diagnostics. Do not store every repeated ultrasonic measurement unless needed for debugging.

Sequence numbers are per node. A unique constraint on (`slot_id`, `source_sequence`) enforces idempotency. Two nodes may use the same sequence number because each bay has its own slot.

Apply retention rules as history grows.

### 5.5 `gateway_credentials`

Do not store plaintext gateway secrets in an ordinary public table.

Use a suitable secret store or hashed credential verification scheme. Provision a unique credential for each gateway and allow credentials to be rotated or revoked.

## 6. Parking State Rules

The cloud must preserve the state semantics in [system-architecture.md](system-architecture.md). Summary `status` is `AVAILABLE`, `FULL`, `INCOMPLETE`, or `OFFLINE`. The default bay stale threshold and the gateway offline threshold are both 180 s.

| State | Available count |
|---|---:|
| `VACANT` | +1 |
| `OCCUPIED` | +0 |
| `UNKNOWN` | +0 |
| `SENSOR_FAULT` | +0 |

Calculate available spaces from bays whose current state is `VACANT`.

Display `FULL` only when every configured bay is confirmed occupied.

If any bay is unknown, faulty or stale, indicate incomplete status instead of claiming the facility is definitely full.

### Freshness rules

- Track `last_seen_at` for every bay.
- Use the 180 s stale-data threshold unless configuration overrides it.
- Treat data beyond the threshold as stale, even if the stored state is `VACANT`.
- Expose a derived freshness status through the API.
- Never allow an old cached state to appear current indefinitely.

The gateway and cloud should use synchronized timestamps. The cloud should also record its own receipt time to help identify clock problems.

## 7. API Design

Use Cloudflare Workers as the controlled public and gateway-facing API layer.

Suggested endpoints:

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| `GET` | `/api/v1/public/parking/:slug` | Public | Facility summary and availability |
| `GET` | `/api/v1/public/parking/:slug/slots` | Public | Public slot status |
| `POST` | `/api/v1/gateway/heartbeat` | Authenticated gateway | Update gateway health |
| `POST` | `/api/v1/gateway/sync` | Authenticated gateway | Upload current slot states and events |
| `GET` | `/api/v1/admin/parking/:id` | Administrator | Read detailed facility information |
| `PATCH` | `/api/v1/admin/slots/:id` | Administrator | Update permitted slot configuration |

Use a consistent JSON response format.

Example public response:

```json
{
  "parking_lot": {
    "name": "Example Parking",
    "slug": "example-parking"
  },
  "summary": {
    "total": 10,
    "available": 6,
    "occupied": 3,
    "unknown": 1,
    "status": "INCOMPLETE"
  },
  "updated_at": "2026-10-30T12:00:00Z"
}
```

The values above are illustrative only.

The API must derive totals from the current configured slots rather than trusting counts supplied by a browser.

### API requirements

- Validate request methods, authentication and payload schemas.
- Reject unknown node IDs.
- Reject invalid occupancy states.
- Enforce request size limits.
- Add rate limiting.
- Use parameterized database operations.
- Return appropriate HTTP status codes.
- Avoid exposing stack traces, secrets or internal database errors.
- Document API versioning.
- Use UTC timestamps in storage and responses.

## 8. Gateway Synchronization

The Raspberry Pi sends data only after it has validated the LoRa packet and updated its local state.

### Normal operation

1. A node reports a confirmed state change.
2. The Pi validates the packet.
3. The Pi updates SQLite.
4. The HDMI dashboard updates immediately.
5. The Pi sends a cloud synchronization request over HTTPS.
6. The API authenticates the gateway and validates the payload.
7. The backend updates the current slot record.
8. A slot event is written when a meaningful transition occurs.
9. The public website receives or retrieves the updated status.

### Internet outage

1. The Pi continues receiving LoRa messages.
2. Local SQLite remains the active local state store.
3. The HDMI dashboard continues operating.
4. Cloud synchronization attempts are queued or coalesced.
5. Retries use exponential backoff.
6. When connectivity returns, the Pi uploads the latest states and required unsynchronized events.

The idempotency key is the gateway id, the node id, and the node `sequence` from [radio-protocol.md](radio-protocol.md). Repeated uploads of that triple must not create duplicate events.

When the gateway reconnects, synchronize current state before replaying older optional diagnostic events.

## 9. Public Website

Build a responsive frontend hosted on Cloudflare Pages.

### 9.1 Public parking page

The public page should display:

- Parking location name
- Total configured bays
- Available bays
- Occupied bays
- Unknown or faulty bays
- Per-bay status
- Last update time
- Gateway or data freshness warning

Example layout:

```text
------------------------------------
        EXAMPLE PARKING
------------------------------------

       AVAILABLE SPACES

               6

       out of 10 parking bays

------------------------------------
Bay 01      VACANT
Bay 02      OCCUPIED
Bay 03      VACANT
Bay 04      UNKNOWN
...
------------------------------------
Last updated: 8:30 PM
Status: SOME SENSOR DATA UNAVAILABLE
------------------------------------
```

The numbers are examples.

### 9.2 Public access

Visitors must not need to register or log in to see parking availability.

Use a public URL such as:

`https://your-domain.example/parking/example-parking`

A custom domain is optional. Start with a provider subdomain during development.

### 9.3 Admin dashboard

Require authentication for:

- Gateway health details
- Sensor diagnostics
- Slot configuration
- Node-to-slot mapping
- Historical occupancy events
- Maintenance operations

Public visibility must not imply public write access.

### 9.4 Refresh strategy

Poll the public API every 10 seconds. Supabase Realtime may replace polling later if usage stays inside the selected plan.

The page must show the time of the last successful data update. If the API is unavailable or the data is stale, show a warning instead of presenting old data as live.

## 10. Authentication and Security

### Public users

- Read access only to explicitly public parking data.
- No account required.
- No direct database write permissions.
- No access to gateway credentials or private diagnostics.

### Administrator

- Supabase Auth or another supported identity provider.
- Enforce authorization on the server, not only in the frontend.
- Use row-level security for direct database access where applicable.
- Restrict administrative operations to authorized users.
- Protect account recovery and administrative sessions.

### Gateway

- Assign a unique credential to each gateway.
- Store secrets outside source control.
- Authenticate every gateway update.
- Rotate or revoke credentials when necessary.
- Validate every incoming field.
- Apply rate limits.
- Use HTTPS.
- Avoid exposing the gateway's local API to the public internet.

### Database

- Enable row-level security where required.
- Keep service-role credentials exclusively in trusted server-side environments.
- Never embed service-role keys in frontend JavaScript.
- Restrict public queries to required columns and records.
- Back up important configuration and review restoration procedures.

## 11. Realtime and Database Load

For 1–10 parking bays, the expected data volume is small.

Recommended behavior:

- Write to the database when a confirmed state transition occurs.
- Send a periodic gateway heartbeat.
- Avoid writing every repeated sensor sample.
- Use indexes on parking lot ID, slot ID and event time.
- Keep the public response compact.
- Limit historical event retention to the period required.
- Monitor database size, API requests, egress and realtime usage.

If Supabase Realtime usage approaches the free-tier limits, replace it with polling or adjust the hosting plan.

## 12. Monitoring and Fault Handling

Track at least:

- Last gateway heartbeat
- Last update from each slot
- Gateway synchronization failures
- Invalid payload counts
- API errors
- Database write failures
- Stale slot count
- Last successful cloud synchronization

The public page should distinguish between:

- All sensors reporting normally
- Some sensors stale or faulty
- Gateway offline
- Cloud data unavailable

Do not infer that the parking facility is full when the gateway stops reporting.

## 13. Backup and Data Retention

- Keep the latest local state in SQLite.
- Back up the Raspberry Pi configuration.
- Back up cloud database configuration and required data.
- Define a retention period for `slot_events`.
- Avoid unlimited diagnostic logging.
- Test recovery from a gateway SD card failure.
- Test cloud data restoration before production deployment.

The local database is for continuity, not a replacement for an independent backup.

## 14. Implementation Phases

### Phase 1: Cloud project setup

- Create the Cloudflare account.
- Create the Supabase project.
- Create the PostgreSQL schema.
- Configure environment variables and secrets.
- Create development and production configuration separately.

### Phase 2: Backend API

- Implement gateway authentication.
- Implement the heartbeat endpoint.
- Implement the synchronization endpoint.
- Implement public read endpoints.
- Add request validation and idempotency.
- Configure database permissions and rate limits.
- Test malformed requests and unauthorized access.

### Phase 3: Gateway integration

- Add the HTTPS client to the Raspberry Pi service.
- Synchronize state changes and heartbeats.
- Implement local queueing and retry logic.
- Test internet loss and restoration.
- Verify duplicate uploads do not create duplicate events.

### Phase 4: Public website

- Build the parking availability page.
- Connect the page to the public API.
- Implement refresh and stale-data warnings.
- Test mobile and desktop layouts.
- Verify the page works without an account.

### Phase 5: Admin dashboard

- Implement login.
- Add gateway health and slot diagnostics.
- Add authorized configuration functions.
- Enforce server-side authorization.

### Phase 6: Deployment and testing

- Deploy the frontend.
- Deploy the API.
- Apply production database policies.
- Configure the public URL.
- Test gateway synchronization.
- Test stale sensor and gateway failure behavior.
- Test administrator permissions.
- Monitor usage against free-tier limits.

## 15. Acceptance Criteria

The cloud implementation is ready when:

- The public website is accessible without login.
- Visitors can see current parking availability.
- The website shows stale or incomplete status correctly.
- The Raspberry Pi synchronizes state changes securely.
- Internet outages do not interrupt the local HDMI display.
- Queued updates synchronize after reconnection.
- Duplicate uploads do not create duplicate events.
- Administrator functions require authorization.
- No privileged database credentials are exposed to the public.
- Hosting usage stays within the selected plan's limits.
- The system works without any video streaming infrastructure.

## 16. Final Recommended Setup

Start with:

- Cloudflare Pages for the public website.
- Cloudflare Workers for the API.
- Supabase PostgreSQL and Auth.
- Supabase Realtime or lightweight polling.
- Raspberry Pi 4B with SQLite for local state.
- HTTPS for synchronization.
- No VPS and no video streaming service.

Target ₱0 per month in additional cloud hosting costs during development and initial testing.

Before relying on this for a continuously available public service, verify current free-tier limits, inactivity behavior, request quotas, backups and uptime requirements. Upgrade only when measured usage or reliability requirements justify the cost.

The design should remain portable so the backend or database can be moved to another provider later without changing the LoRa sensor nodes.
