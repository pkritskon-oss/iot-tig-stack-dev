# IoT TIG Dev Stack

Developer-ready TIG stack (Telegraf, InfluxDB, Grafana) for ingesting IoT payloads from an external Mosquitto broker and visualizing them in Grafana.

This repository distributes curated configuration and initialization files. The core value lies in reproducible configs and a single source of truth for database initialization (`init.sql`).

It is not a framework, but a **baseline distribution**: a ready-to-use stack that saves hours of setup and ensures consistency across environments.

---

## 📡 Data Flow

> Note: The `mosquitto_repo` service in this repository is **not the primary broker** 
for IoT devices. Real devices should connect to an **external Mosquitto broker**.  
Here, `mosquitto_repo` only acts as a **relay/bridge** to forward payloads into 
the local stack for development and testing purposes.

Flow:

External Mosquitto (simulated device payloads)  
→ `mosquitto_repo` (relay/bridge)  
→ Telegraf (MQTT consumer)  
→ InfluxDB (time-series database)  
→ Grafana (visualization)

---

## 📂 Core Files

- docker-compose.yml  
  Orchestrates TIG containers and mounts volumes for configs and data

- configs/telegraf.conf  
  Generic MQTT → InfluxDB pipeline, with topic patterns and field mapping

- influxdb/init/init.sql  
  **The most important file**: creates database `iotdata`, retention policies, users, and baseline measurement(s)

- grafana/provisioning/  
  Auto-provision datasource and dashboards

---

## 📂 Repository Structure

.
├── configs/  
│   ├── influxdb.conf          # InfluxDB config (minimal, DB init handled by init.sql)  
│   ├── mosquitto.conf         # Mosquitto relay/bridge configuration  
│   └── telegraf.conf          # Telegraf agent configuration  
├── docker-compose.yml         # Compose file to run the stack  
├── docs/  
│   └── README.md              # Documentation notes  
├── grafana/  
│   └── provisioning/          # Datasource provisioning (InfluxDB)  
└── influxdb/  
    └── init/init.sql          # Initialization script (DB, RP, CQ)  

---

## 🗄 Database Initialization (init.sql)

init.sql provides a known-good baseline:
- Database: iotdata
- Retention policy: raw_90d
- Users and permissions aligned to least privilege
- Baseline measurement (e.g., iot_raw) for generic IoT ingestion

Rationale:
- A single, versioned source of truth for schema and policies
- Reproducibility across dev machines and environments

---

## 📊 Example Payload

{
  "device": "esp32-01",
  "sensor": "temp",
  "value": 28.5,
  "ts": 1730000000
}

Stored in InfluxDB as:
- measurement: iot_raw
- tags: device, sensor
- fields: value
- time: ts

---

## 🛠 Development Notes

The `.gitignore` file ensures that runtime data and temporary files are not committed 
to the repository. Key ignored paths include:

- **InfluxDB runtime**: `influxdb/data/`, `influxdb/meta/`, `influxdb/wal/`  
- **Grafana runtime**: `grafana/grafana.db`, `grafana/png/`, `grafana/pdf/`, `grafana/csv/`  
- **Logs**: `*.log`, `logs/`  
- **Editor/OS files**: `.DS_Store`, `.vscode/`, `.idea/`  
- **Secrets**: `.env`, `.env.*`  

This keeps the repository clean and reproducible, focusing only on configuration, 
initialization scripts, and provisioning files.

## ⚙️ Configuration Highlights

### Mosquitto (Relay/Bridge)
- Listens on port **1883**
- `allow_anonymous true` for development (not recommended in production)
- Bridges to external broker at `192.168.22.228:1883`
- Forwards all topics (`#`) into the local stack

### Telegraf
- Input: `mqtt_consumer`
  - Server: `tcp://mosquitto:1883`
  - Topic: `8E215788_nx100_raw/+/data`
  - Data format: JSON
  - Measurement override: `iot_raw`
- Output: `influxdb`
  - URL: `http://influxdb_repo:8086`
  - Database: `iotdata`
  - Timeout: 10s
  - Write consistency: any
- Agent settings:
  - Interval: 10s
  - Flush interval: 10s
  - Batch size: 500
  - Buffer limit: 10,000

### InfluxDB
- Version: **1.8.10 (LTS)**
- Database: `iotdata`
- Retention Policies:
  - `raw_90d` → 90 days (default)
  - `avg_1y` → 365 days
- Continuous Query:
  - `cq_downsample_5m` → downsample `iot_raw` into `avg_1y.downsampled_5m` with 5‑minute mean

### Grafana
- Version: **10.4.2**
- Port: `3000`
- Provisioning:
  - Datasource: `InfluxDB-iotdata`
  - URL: `http://influxdb_repo:8086`
  - Database: `iotdata`
  - Default datasource enabled

## 🚀 Quick Start

1. Clone the repository:
   git clone https://github.com/your-org/iot-tig-repo.git
   cd iot-tig-repo

2. Start the stack:
   docker-compose up -d

3. Verify services:
   - Mosquitto → tcp://localhost:1883
   - InfluxDB → http://localhost:8086
   - Grafana → http://localhost:3000 (default user: admin / admin)

4. Publish a test payload (simulate device):
   mosquitto_pub -h localhost -t "8E215788_nx100_raw/test/data" \
     -m '{"device":"esp32-001","power":120.5,"soc":86,"ts":1699999999}'

5. Open Grafana:
   - Login at http://localhost:3000
   - Datasource `InfluxDB-iotdata` is pre-provisioned
   - Create a new panel and query measurement `iot_raw`
