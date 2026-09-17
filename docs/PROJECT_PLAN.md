# RoomMesh — Project Plan & Engineering Roadmap

**Project**: Multi-Room Environmental IoT Monitoring Platform  
**Authors**: Janwalkar Pooja, Moçi AnnaMaria, Mosudi Isiaka, Philip Julia  
**Repository**: `imosudi/multi-room-iot-monitoring` • **Public Identity**: RoomMesh (`roommesh.site`)  
**Baseline Hardware**: 2 × ESP32-S3 • 2 × DHT22 • 1 × Raspberry Pi 5 (8GB) Edge Gateway  
**Document Status**: Final Planning Document (2-Page Executive Blueprint)

---

## 1. Executive Summary & Mission

**RoomMesh** is an open-source, edge-native IoT platform engineered to collect, validate, persist, and compare multi-room indoor environmental telemetry in real time. Built around a Raspberry Pi 5 edge gateway and ESP32-S3 sensor microcontrollers, the platform emphasizes **comparative environmental intelligence**—calculating real-time room differentials ($\Delta\text{Temperature}$, $\Delta\text{Humidity}$) rather than displaying isolated readings.

### Core Non-Negotiable Tenets
1. **Strict Physical Sensor Truth**: The hardware deployment consists exclusively of calibrated DHT22 sensors measuring temperature and relative humidity. **Fabricated CO₂ or air-quality metrics are strictly prohibited.**
2. **Deterministic Entity Hierarchy**: One-to-one mapping across physical room, device identity, MQTT topic namespace, oneM2M Application Entity (AE), and InfluxDB time-series tags.
3. **Security-by-Design**: Mutual TLS (mTLS) or TLS-authenticated transport on port 8883 with strict room-level Access Control Lists (ACLs); no anonymous broker access.
4. **Configuration-Driven Extensibility**: Scaling from the 2-room baseline to $N$ rooms is driven by declarative configuration (`rooms.yaml`) without modifying core service code.

---

## 2. Hardware Reality & System Baseline

| Component | Model / Specification | Deployed Role | Quantity |
| :--- | :--- | :--- | :---: |
| **Edge Gateway** | Raspberry Pi 5 (ARM64 / aarch64, Debian Linux) | Podman container host, broker, CSE, DB, API, Grafana | 1 |
| **Room 1 Node** | Espressif ESP32-S3 (Wi-Fi 802.11 b/g/n, TLS client) | Dedicated room telemetry node (`esp32-room1`) | 1 |
| **Room 2 Node** | Espressif ESP32-S3 (Wi-Fi 802.11 b/g/n, TLS client) | Dedicated room telemetry node (`esp32-room2`) | 1 |
| **Sensors** | Aosong DHT22 / AM2302 (Digital 1-Wire protocol) | Physical temp ($-40..80^\circ\text{C}$) & RH ($0..100\%$) capture | 2 |

### Interface & Domain Topology
- `roommesh.site`: Public project landing page, documentation, and architecture specifications.
- `dashboard.roommesh.site`: Primary human-facing RoomMesh real-time comparison dashboard.
- `api.roommesh.site`: Public/internal REST API exposing room state, historical trends, and differentials.
- `grafana.roommesh.site`: Operational observability, sensor health telemetry, and time-series analytics.

---

## 3. Architecture & End-to-End Data Pipeline

```text
[Room 1: ESP32-S3 + DHT22] ──(MQTT over TLS :8883)──┐
                                                    ├──► [Mosquitto Broker] ──► [oneM2M CSE Base]
[Room 2: ESP32-S3 + DHT22] ──(MQTT over TLS :8883)──┘           │                      │
                                                                ▼                      ▼
                                                   [Ingestion & Validation Pipeline (Node-RED/Python)]
                                                                │                      │
                                                    (Line Protocol Write)       (Entity Status)
                                                                ▼                      ▼
                                                        [InfluxDB TSDB] ◄────── [RoomMesh REST API]
                                                                │                      │
                                                        (Flux Queries)           (JSON Stream)
                                                                ▼                      ▼
                                                      [Grafana Dashboards]    [Web UI / Clients]
```

1. **Sensing Layer**: Nodes sample DHT22 sensors at 10-second intervals and publish JSON payloads to `iot/rooms/{room_id}/telemetry` containing `room_id`, `device_id`, `sensor`, `temperature_c`, `humidity_pct`, and ISO-8601 `timestamp`.
2. **Edge Gateway (Podman Rootless)**: Mosquitto enforces TLS 1.3 and per-client ACLs. 
3. **Semantic Hierarchy (oneM2M)**: A standardized CSE tree maintains explicit resources: `CSEBase` $\rightarrow$ `Room1AE` / `Room2AE` $\rightarrow$ `temperature` / `humidity` $\rightarrow$ `contentInstance`.
4. **Ingestion & Persistence**: Ingestion services validate data boundaries, dropouts, and stale flags before writing to InfluxDB measurement `environmental`.
5. **Analytics & Presentation**: Grafana and FastAPI consume InfluxDB data to display current observations, historical series, and signed differential metrics ($\Delta T = T_{\text{room1}} - T_{\text{room2}}$).

---

## 4. Phased Work Breakdown Structure (WBS)

### Phase 1: Edge Gateway & Security Infrastructure
- [x] Configure Podman rootless runtime and systemd service orchestration on Raspberry Pi 5.
- [x] Initialize local Public Key Infrastructure (PKI) with CA, server certificates, and client certificates.
- [x] Deploy Eclipse Mosquitto with TLS listener (port 8883), password hashing, and room-isolated ACL rules.
- [x] Establish canonical room abstraction schema (`rooms.yaml`).

### Phase 2: Sensor Firmware & Edge Node Telemetry
- [ ] Implement robust ESP32-S3 firmware with non-blocking DHT22 driver and Wi-Fi auto-reconnect logic.
- [ ] Embed TLS credentials and certificate validation into firmware storage.
- [ ] Implement deterministic JSON telemetry publisher with fail-safe error states (marking NaN on read failure).
- [ ] Validate hardware bench tests for both Room 1 and Room 2 nodes under network jitter conditions.

### Phase 3: oneM2M Semantic Model & Ingestion Engine
- [ ] Deploy containerized oneM2M CSEBase (`RoomMeshCSE`) with persistent storage.
- [ ] Configure automatic AE and Container bootstrap provisioning (`Room1AE`, `Room2AE`).
- [ ] Deploy Node-RED / Python ingestion microservice to bridge MQTT messages into oneM2M `contentInstance` records.
- [ ] Implement schema validation: reject out-of-bound readings and guard against zero-value substitution.

### Phase 4: Time-Series Persistence & Observability
- [ ] Configure InfluxDB v2 bucket `roommesh` with retention policies and tag schema (`room_id`, `device_id`, `sensor`).
- [ ] Build high-throughput Influx Line Protocol ingestion worker.
- [ ] Create Grafana dashboards (`grafana.roommesh.site`):
  - Room 1 vs Room 2 side-by-side temperature and humidity graphs.
  - Signed difference panels ($\Delta\text{Temperature}$, $\Delta\text{Humidity}$) with directional color grading.
  - Device availability, telemetry cadence, and stale data indicators.

### Phase 5: RoomMesh API, Web Surfaces & Verification
- [ ] Develop FastAPI backend (`api.roommesh.site`) exposing `/api/v1/rooms`, `/api/v1/compare`, and `/health`.
- [ ] Deploy lightweight web dashboard (`dashboard.roommesh.site`) and landing page (`roommesh.site`).
- [ ] Execute end-to-end integration tests: sensor publish $\rightarrow$ broker $\rightarrow$ oneM2M $\rightarrow$ InfluxDB $\rightarrow$ Grafana.
- [ ] Verify plug-and-play addition of Room 3 via `rooms.yaml` configuration alone.

---

## 5. Risk Management & Mitigation Matrix

| Risk / Failure Mode | Impact | Severity | Mitigation Strategy |
| :--- | :--- | :---: | :--- |
| **Physical Sensor Read Failure** | Stale or missing telemetry | Medium | Firmware retries 3 times; if failure persists, publishes explicit `null` state rather than fabricating or repeating old values. |
| **Wi-Fi Disconnection / Jitter** | Telemetry data gaps | Medium | Non-volatile flash ring buffer on ESP32-S3 stores up to 60 minutes of readings, flushed upon MQTT reconnect. |
| **Broker Security Breach** | Unauthorized access to telemetry | High | Enforce mTLS client certs, TLS 1.3 only, disable anonymous access, and restrict ACLs so nodes can only write to their assigned topic. |
| **Data Fabrication / Scope Creep** | Scientific invalidity | Critical | Code reviews enforce rejection of dummy CO₂ generators; panels without physical sensors remain explicitly marked "unavailable". |
| **Gateway Storage Saturation** | Container crash on RPi 5 | Medium | Define InfluxDB 90-day downsampling retention policy; log rotation enabled on all Podman container units. |

---

## 6. Deliverables & Acceptance Criteria

1. **Firmware Release**: Flashed and bench-tested firmware images for `esp32-room1` and `esp32-room2`.
2. **Infrastructure-as-Code**: Single-command `podman-compose.yml` deploying Mosquitto, oneM2M CSE, InfluxDB, Grafana, Node-RED, and FastAPI.
3. **Verified Telemetry Flow**: Documented end-to-end telemetry verification with `< 2s` propagation latency from sensor read to dashboard display.
4. **Observable Comparative Analytics**: Functional Grafana dashboard displaying real-time dual-room differential curves.
5. **Reproducible Documentation**: Complete installation, certificate generation, and operations guide ensuring a new engineer can reproduce the stack from scratch.
