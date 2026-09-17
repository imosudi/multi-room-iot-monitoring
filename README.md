# RoomMesh — Multi-Room IoT Monitoring

![Project logo](assets/logo.svg)

![IoT](https://img.shields.io/badge/IoT-Internet%20of%20Things-0D6EFD?logo=internetofthings&logoColor=white)
![ESP32-S3](https://img.shields.io/badge/ESP32--S3-E7352C?logo=espressif&logoColor=white)
![DHT22](https://img.shields.io/badge/DHT22-Temperature%20%26%20Humidity-5C6BC0)
![Raspberry Pi 5](https://img.shields.io/badge/Raspberry%20Pi%205-Edge%20Gateway-C51A4A?logo=raspberrypi&logoColor=white)
![MQTT](https://img.shields.io/badge/MQTT-3C5280?logo=mqtt&logoColor=white)
![Mosquitto](https://img.shields.io/badge/Mosquitto-Broker-3C5280?logo=eclipse-mosquitto&logoColor=white)
![oneM2M](https://img.shields.io/badge/oneM2M-M2M%20Standard-6A1B9A)
![Podman](https://img.shields.io/badge/Podman-Container%20Platform-892CA0?logo=podman&logoColor=white)
![InfluxDB](https://img.shields.io/badge/InfluxDB-Time%20Series-22ADF6?logo=influxdb&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Visualization-F46800?logo=grafana&logoColor=white)
![Node-RED](https://img.shields.io/badge/Node--RED-Integration-8F0000?logo=nodered&logoColor=white)
![Python](https://img.shields.io/badge/Python-Integration%20Services-3776AB?logo=python&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-ARM64%20Edge-FCC624?logo=linux&logoColor=black)

RoomMesh is a multi-room environmental monitoring platform designed around a Raspberry Pi 5 edge gateway, secure MQTT transport, oneM2M resource modelling, time-series persistence and room-to-room comparison dashboards. The project uses two ESP32-S3 sensor nodes and two DHT22 sensors as the initial deployment baseline, while preserving an extensible architecture for additional rooms and future sensor types.

This repository acts as the implementation blueprint and project documentation for a reproducible IoT platform aligned with the RoomMesh master specification. It is intentionally grounded in the real hardware constraints of the project: the current system measures temperature and humidity only, and does not fabricate CO₂ readings.

## Project identity

- GitHub repository: multi-room-iot-monitoring
- Public project identity: RoomMesh
- Primary domain: roommesh.site
- Dashboard domain: dashboard.roommesh.site
- API domain: api.roommesh.site
- Grafana domain: grafana.roommesh.site

## Mission

The platform must transform the repository into a coherent, secure and extensible multi-room monitoring architecture based on the following flow:

```text
ESP32-S3 room sensor
  ↓
DHT22 sensor
  ↓
MQTT over TLS/mTLS
  ↓
Raspberry Pi 5 edge gateway
  ├── Mosquitto
  ├── oneM2M CSE
  ├── integration / API service
  ├── Node-RED
  ├── InfluxDB
  └── Grafana
```

The architecture is intentionally designed to compare room conditions, not merely display isolated readings.

## Hardware reality

Current physical hardware:

- 2 × ESP32-S3
- 2 × DHT22
- 1 × Raspberry Pi 5

This means the current deployment is a two-room environmental monitoring system with:

- temperature
- relative humidity

The project explicitly does not fabricate CO₂ values. CO₂ support may be designed as a future extension, but it must remain clearly labelled as optional and unavailable unless a real sensor is physically connected and verified.

## System architecture

![Architecture overview](assets/architecture-overview.svg)

### Logical data path

```text
Room
  ↓
ESP32-S3 identity
  ↓
MQTT topic
  ↓
MQTT message
  ↓
oneM2M AE
  ↓
oneM2M container
  ↓
contentInstance
  ↓
InfluxDB
  ↓
API / dashboard
  ↓
Grafana / RoomMesh dashboard
```

## Room model

The canonical room abstraction is configuration-driven and should support room1, room2, room3 and beyond without redesigning the stack.

```yaml
rooms:
  room1:
    device_id: esp32-room1
    mqtt_client_id: esp32-room1
    oneM2M_ae: Room1AE

  room2:
    device_id: esp32-room2
    mqtt_client_id: esp32-room2
    oneM2M_ae: Room2AE
```

This model keeps room identity deterministic and avoids hard-coding room logic across services.

## MQTT architecture

Mosquitto is the broker and MQTT remains the primary telemetry transport path.

Recommended topic namespace:

```text
iot/rooms/{room_id}/temperature
iot/rooms/{room_id}/humidity
```

Examples:

```text
iot/rooms/room1/temperature
iot/rooms/room1/humidity
iot/rooms/room2/temperature
iot/rooms/room2/humidity
```

Example telemetry payload:

```json
{
  "room_id": "room1",
  "device_id": "esp32-room1",
  "sensor": "dht22",
  "temperature_c": 23.4,
  "humidity_pct": 48.2,
  "timestamp": "2026-09-17T08:00:00Z"
}
```

The repository should preserve the existing payload contract if already implemented, but the final design must document a deterministic MQTT message format.

## MQTT security and certificate model

The system must preserve secure broker operation and must not fall back to anonymous or unauthenticated MQTT.

Required security properties:

- TLS enabled
- mTLS client validation where used
- certificate identity checks
- MQTT authentication and authorisation
- ACL enforcement for room-specific topics
- no private keys or production secrets committed to the repository

The certificate model should align service identity with the broker and authentication policy. Certificate identities should match the intended MQTT identity and ACL rules; a valid TLS handshake alone is not sufficient evidence of proper broker authorisation.

## oneM2M architecture

The oneM2M layer should represent a genuine room hierarchy rather than a generic flat container system.

```text
CSE
├── AE: Room1AE
│   ├── container: temperature
│   └── container: humidity
│
└── AE: Room2AE
    ├── container: temperature
    └── container: humidity
```

The key mapping is:

```text
room_id
  ↕
ESP32 identity
  ↕
MQTT topic
  ↕
oneM2M AE
  ↕
oneM2M container
  ↕
contentInstance
```

The system should avoid duplicate entities and ensure that room-level identity remains explicit in the data model.

## MQTT → oneM2M integration

The integration layer should perform the following steps:

1. subscribe to authorised room telemetry topics
2. validate incoming messages
3. identify the room and device
4. identify the sensor type
5. validate measurement values and timestamps
6. map to the target AE and container
7. create the content instance
8. record success or failure
9. expose health and diagnostic information

If Node-RED is used, its role should remain explicit and documented rather than duplicating the main business logic.

## Podman and Raspberry Pi 5 architecture

The project is intended to run on Raspberry Pi 5 using Podman as the container runtime and must remain compatible with ARM64 / aarch64.

The relevant service layout is expected to include:

- Mosquitto
- oneM2M CSE
- integration / API service
- Node-RED
- InfluxDB
- Grafana

The repository should preserve an existing Podman architecture and improve it rather than replacing a functioning setup with a competing deployment model.

## Data model and storage

InfluxDB stores paired historical environmental readings in a format that supports room comparison.

Recommended schema:

```text
measurement: environmental

tags:
  room_id
  device_id
  sensor

fields:
  temperature_c
  humidity_pct
```

This keeps the data model direct and efficient for dashboards while avoiding unnecessary duplication across multiple measurements.

## Grafana and dashboard design

Grafana provides the multi-room comparison view for a building-level environment dashboard.

Required views:

- current room status
- temperature comparison between room1 and room2
- humidity comparison between room1 and room2
- historical time-range comparison
- difference metrics such as ΔTemperature and ΔHumidity
- stale/offline/unknown device indicators

The dashboard must not hide the sign of a calculated difference, and it should not directly present a fabricated CO₂ panel without real sensor validation.

## RoomMesh web and API architecture

The project separates distinct application surfaces:

```text
roommesh.site          → project landing page and documentation
dashboard.roommesh.site → human-facing RoomMesh dashboard
api.roommesh.site      → API
grafana.roommesh.site  → Grafana observability dashboard
```

Suggested API resources:

```text
GET /api/v1/rooms
GET /api/v1/rooms/{room_id}
GET /api/v1/rooms/{room_id}/current
GET /api/v1/rooms/{room_id}/history
GET /api/v1/compare
GET /api/v1/health
```

The API should abstract the database and messaging layer rather than exposing raw storage details directly.

## Data quality and device availability

The implementation should validate and classify telemetry in a way that distinguishes:

- current observation
- historical observation
- device online/offline
- stale data
- missing data
- invalid data
- system error

A missing reading must not silently become a zero value. Device availability should be explicit and observable.

## ESP32-S3 firmware requirements

Each ESP32-S3 node must be configured with:

- room_id
- device_id
- MQTT client identity
- MQTT broker address and port
- TLS certificate configuration
- publish interval

The firmware should handle:

- failed sensor reads
- invalid temperatures or humidity values
- Wi-Fi loss
- TLS failures
- MQTT disconnection and reconnect
- timing problems and handling of stale data

Do not hard-code production credentials into the firmware source files.

## Configuration and secrets

The repository should maintain a clear separation between:

- source code
- configuration
- secrets
- runtime state
- persistent storage

Use an example configuration file such as `.env.example` with placeholders rather than real values. Private keys, certificates, passwords and tokens must not be committed to Git.

## Installation and deployment workflow

The expected deployment process is:

```text
clone repository
  ↓
configure environment variables and room definitions
  ↓
provision certificates and secrets
  ↓
prepare persistent storage
  ↓
start Podman services
  ↓
verify service health
  ↓
verify MQTT telemetry
  ↓
verify oneM2M ingestion
  ↓
verify InfluxDB persistence
  ↓
verify Grafana dashboard data
```

This process must be reproducible by a technically competent user without undocumented steps.

## Certificate provisioning and infrastructure

The project already includes or expects certificate management for relevant services. The implementation should preserve those patterns rather than introducing a parallel PKI workflow.

Relevant certificate identities may include:

- mosquitto
- client
- ble
- influxdb
- backend

The repository must keep all private certificates and keys outside source control.

## Testing strategy

The project should include validation at multiple levels:

### Unit tests
- payload parsing
- room mapping
- sensor mapping
- oneM2M mapping
- API transformation
- validation logic

### Integration tests
- ESP32 → MQTT → integration → oneM2M → InfluxDB → API
- room1 and room2 independent validation
- simultaneous multi-room comparison verification

### Failure testing
- MQTT broker unavailable
- invalid payloads
- invalid DHT22 values
- oneM2M unavailable
- InfluxDB unavailable
- stale readings
- unauthorised MQTT client
- expired TLS certificate
- unknown room identifier

The system must fail observably, not silently.

## Troubleshooting

Common operational checks include:

- verifying MQTT broker availability and TLS validity
- confirming MQTT ACL and client identity behaviour
- checking room-to-topic mapping
- verifying oneM2M AE/container creation
- confirming data appears in InfluxDB with the correct tags and fields
- validating Grafana datasource connectivity
- checking the API for stale or missing data
- confirming room online/offline status logic

## Extension to additional rooms

The architecture is intentionally extensible:

```text
room1
room2
room3
room4
...
roomN
```

A new room should ideally be added through configuration and mapping definitions rather than duplicated source logic.

## Extension to additional sensors

The project architecture allows future sensor types, but only if they are backed by real hardware and verified data paths. CO₂ is explicitly a future extension, not a current claim of measurement capability.

Selected sensor extension pattern:

- define the schema
- map the sensor to room and MQTT topic
- extend oneM2M container hierarchy
- add database fields or tags as needed
- expose the new value in the API and dashboards

## Project limitations

This project currently has real-world constraints that must be documented honestly:

- the available sensor suite is DHT22-based and provides temperature and humidity only
- no confirmed CO₂ sensor is currently available
- fabricated CO₂ data is not acceptable in a production path
- dashboards and APIs must distinguish known, unknown, stale and missing values
- production security requires validated TLS and identity controls

## Authors

- Janwalkar Pooja
- Moçi AnnaMaria
- Mosudi Isiaka
- Philip Julia

## License

This project is distributed under the BSD 3-Clause License. See [LICENSE](LICENSE) for the complete terms and conditions.

## Notes

This repository is structured as an implementation prompt and engineering blueprint for a complete multi-room IoT air-quality comparison system. It is intended to be used as the basis for a reproducible academic platform with secure edge deployment, room-aware resource modelling, and comparative analytics across environmental conditions.
