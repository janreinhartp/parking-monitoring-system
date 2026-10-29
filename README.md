# Parking Monitoring System

A low-cost, solar-powered parking monitoring system designed to detect parking bay occupancy, display real-time availability, and publish parking status to a public website.

The system uses LoRa communication between sensor nodes and a Raspberry Pi gateway. The gateway maintains local parking status and synchronizes updates to the cloud over a 4G LTE connection.

## Features

- Real-time parking bay occupancy monitoring
- Private point-to-point LoRa communication
- Solar-powered sensor nodes
- Local parking dashboard displayed through HDMI
- Public parking availability website
- Cloud synchronization over HTTPS
- Offline operation during internet outages
- Sensor health and stale-data monitoring
- Private administrator dashboard
- Low-cost cloud architecture with free-tier hosting options

## System Architecture

```text
                    PARKING BAY SENSOR NODES
                 Wio-E5 mini + A02YYUW ultrasonic
                              |
                              | LoRa P2P
                              v
                     RASPBERRY PI 4B
                    Local Parking Gateway
                              |
             +----------------+----------------+
             |                |                |
         SQLite DB       HDMI Dashboard    Cloud Sync
             |                                 |
             |                          HTTPS over 4G
             |                                 |
             |                                 v
             |                         CLOUDFLARE WORKERS
             |                                 |
             |                                 v
             |                           SUPABASE
             |                         PostgreSQL + Auth
             |                                 |
             |                                 v
             |                         CLOUDFLARE PAGES
             |                                 |
             |                                 v
             |                       PUBLIC PARKING WEBSITE
```

The Raspberry Pi handles local parking state and the entrance display. Cloud services provide remote access and public status updates.

The system does not require internet access for local parking monitoring.

## Hardware

### Sensor Nodes

- Seeed Studio Wio-E5 mini
- A02YYUW waterproof ultrasonic sensor
- Solar panel
- LiFePO4 battery
- Compatible solar charge controller
- Low-quiescent-current boost converter
- Sensor power switching circuit
- Frequency-matched LoRa antenna

### Gateway

- Raspberry Pi 4B, 4 GB RAM
- Compatible LoRa radio/HAT
- LoRa antenna
- HDMI display
- 4G LTE router
- 32–64 GB high-endurance microSD card
- Appropriate power supply and enclosure

> **Important:** Confirm the exact LoRa radio model, operating frequency, packet compatibility and applicable Philippine NTC regulations before purchasing radio hardware.

## Software Stack

| Component | Technology |
|---|---|
| Sensor firmware | Embedded C/C++ |
| Sensor MCU and radio | STM32WLE5JC / Wio-E5 mini |
| Gateway | Raspberry Pi OS |
| Gateway service | Python |
| Local database | SQLite |
| Cloud API | Cloudflare Workers |
| Cloud database | Supabase PostgreSQL |
| Authentication | Supabase Auth |
| Frontend hosting | Cloudflare Pages |
| Cloud communication | HTTPS |
| Public updates | Poll every 10 s. Supabase Realtime may replace polling later |

The stack is a proposed implementation. Individual components should be validated during development.

## Parking Bay States

| State | Description |
|---|---|
| `VACANT` | Parking bay is confirmed empty |
| `OCCUPIED` | Parking bay is confirmed occupied |
| `UNKNOWN` | No recent valid report or uncertain state |
| `SENSOR_FAULT` | Sensor malfunction or invalid readings |

Unknown, faulty, and stale bays are never counted as available. A bay with no report for 180 seconds is stale and becomes `UNKNOWN`.

The summary is `FULL` only when every configured bay is confirmed occupied. If any bay is unknown, faulty, or stale, the summary is `INCOMPLETE`. If the gateway heartbeat is older than 180 seconds, the public summary is `OFFLINE`. Details are in [docs/system-architecture.md](docs/system-architecture.md).

## Repository Structure

```text
parking-monitoring-system/
├── README.md
└── docs/
    ├── system-architecture.md
    ├── radio-protocol.md
    ├── node-sensor-plan.md
    ├── gateway-plan.md
    └── cloud-plan.md
```

The repository is documentation only. Add firmware, gateway, cloud, and website directories when those components are implemented. A deployment guide waits until the radio link and the node energy budget have been measured.

## Data Flow

1. A sensor node measures the distance to a vehicle or empty parking bay.
2. The sensor firmware filters measurements and determines the occupancy state.
3. The node transmits a LoRa packet to the gateway.
4. The Raspberry Pi validates the packet and updates its local SQLite database.
5. The HDMI dashboard updates from the local state.
6. The gateway sends occupancy changes to the cloud API over HTTPS.
7. The cloud validates the request and updates PostgreSQL.
8. The public website retrieves the latest parking status.
9. The website displays occupancy, available spaces and data freshness.

## Offline Operation

When internet connectivity is unavailable:

- LoRa communication continues locally.
- The gateway continues updating parking states.
- The HDMI dashboard remains functional.
- Cloud updates are queued or coalesced locally.
- Synchronization resumes when connectivity returns.

The website must identify stale cloud data rather than presenting old occupancy information as current.

## Cloud Hosting

The initial deployment targets free-tier services to minimize recurring costs.

| Service | Purpose |
|---|---|
| Cloudflare Pages | Public website hosting |
| Cloudflare Workers | Backend API |
| Supabase PostgreSQL | Parking data and event history |
| Supabase Auth | Administrator authentication |
| Supabase Realtime or polling | Website updates |

The initial hosting cost target is **₱0/month**, subject to current service limits and terms.

Free tiers are intended for development and initial testing. Confirm database inactivity behavior, quotas, backups and uptime requirements before production deployment.

## Security

- Authenticate all gateway synchronization requests.
- Keep privileged API and database credentials out of frontend code.
- Restrict administrator functions to authorized accounts.
- Validate incoming packets and API payloads.
- Use HTTPS for cloud communication.
- Apply database access policies.
- Avoid exposing the gateway's local API directly to the internet.
- Keep radio configuration compliant with local regulations.

## Development Roadmap

The v1 LoRa packet contract is already specified in [docs/radio-protocol.md](docs/radio-protocol.md). Implementation follows the build order in [docs/system-architecture.md](docs/system-architecture.md):

1. Confirm NTC limits, select the LoRa HAT, and bench-test one node to the Pi.
2. One bay, local only: occupancy filtering, SQLite, local API, and the HDMI kiosk.
3. Cloud sync, including the offline queue.
4. Public website.
5. Administrator dashboard.
6. Field test with the planned number of nodes.

## Camera Scope

An IP camera is out of this project. There is no remote playback, cloud video, recording, or video server. A camera could be a separate later project. It must not change parking-availability behavior. The non-goal list is in [docs/system-architecture.md](docs/system-architecture.md).

## Documentation

[docs/system-architecture.md](docs/system-architecture.md) is the source of truth for architecture, state rules, defaults, and build order.

| Document | Contents |
|---|---|
| [docs/system-architecture.md](docs/system-architecture.md) | System decisions, data flow, offline behavior, non-goals |
| [docs/radio-protocol.md](docs/radio-protocol.md) | v1 LoRa packet contract |
| [docs/node-sensor-plan.md](docs/node-sensor-plan.md) | Solar sensor node |
| [docs/gateway-plan.md](docs/gateway-plan.md) | Raspberry Pi gateway and HDMI dashboard |
| [docs/cloud-plan.md](docs/cloud-plan.md) | Cloud API, database, public site, and admin dashboard |

## License

Choose and add an appropriate license before publishing or distributing this project.

## Project Status

**Status:** Planning. No firmware, gateway, cloud, or website code is in the repository yet.

Hardware compatibility, radio configuration, and the node energy budget must be measured before production use. The build order is in [docs/system-architecture.md](docs/system-architecture.md).
