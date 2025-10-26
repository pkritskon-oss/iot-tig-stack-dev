# IoT TIG Dev Stack (สำหรับนักพัฒนา)

Repo นี้เป็น **สภาพแวดล้อม TIG stack (Telegraf + InfluxDB + Grafana)** 
ที่เตรียมไว้สำหรับการพัฒนาและทดสอบ โดยทำหน้าที่รับ payload จาก 
**Mosquitto broker ภายนอก** (ซึ่งจำลองการส่งข้อมูลจากอุปกรณ์จริง) 
แล้วส่งต่อไปยัง InfluxDB และแสดงผลผ่าน Grafana

---

## 📡 ลำดับการทำงานของข้อมูล

> หมายเหตุ: service `mosquitto_repo` ใน repo นี้ **ไม่ใช่ broker หลัก** 
ที่อุปกรณ์ IoT เชื่อมต่อเข้ามา แต่มีไว้เพื่อทำหน้าที่เป็น 
**ตัวส่งผ่าน (relay/bridge)** จาก Mosquitto ภายนอกเข้ามายัง stack 
เพื่อให้ Telegraf สามารถทดสอบการ ingest ข้อมูลได้

Flow:

Mosquitto ภายนอก (จำลอง payload จากอุปกรณ์จริง)  
→ `mosquitto_repo` (relay/bridge)  
→ Telegraf (MQTT consumer)  
→ InfluxDB (time-series database)  
→ Grafana (visualization)

---

## 🗄 Database Initialization (init.sql)

ไฟล์ init.sql คือ baseline ที่สำคัญ:
- สร้าง database: iotdata
- กำหนด retention policy: raw_90d
- สร้าง user และสิทธิ์การเข้าถึง
- กำหนด measurement เริ่มต้น (เช่น iot_raw) สำหรับ ingestion ข้อมูล IoT

เหตุผล:
- เป็น source of truth ที่ version control ได้
- ทำให้ reproducible ข้ามเครื่อง dev และ environment ต่าง ๆ

---

## 📊 ตัวอย่าง Payload

{
  "device": "esp32-001",
  "sensor": "temp",
  "value": 28.5,
  "ts": 1699999999
}

ถูกเก็บใน InfluxDB เป็น:
- measurement: iot_raw
- tags: device, sensor
- fields: value
- time: ts

---

## 📂 โครงสร้าง Repo

.
├── configs/  
│   ├── influxdb.conf          # InfluxDB config (minimal, DB init ทำโดย init.sql)  
│   ├── mosquitto.conf         # Mosquitto relay/bridge configuration  
│   └── telegraf.conf          # Telegraf agent configuration  
├── docker-compose.yml         # Compose file สำหรับรัน stack  
├── docs/  
│   └── README.md              # เอกสารประกอบ  
├── grafana/  
│   └── provisioning/          # Datasource provisioning (InfluxDB)  
└── influxdb/  
    └── init/init.sql          # สคริปต์สร้าง DB, Retention Policy, Continuous Query  

---

## ⚙️ ไฮไลท์การตั้งค่า

### Mosquitto (Relay/Bridge)
- เปิดพอร์ต **1883**
- `allow_anonymous true` (ใช้เพื่อ dev/test เท่านั้น)
- bridge ไปยัง broker ภายนอก `192.168.22.228:1883`
- forward ทุก topic (`#`) เข้ามาใน stack

### Telegraf
- Input: `mqtt_consumer`
  - Server: `tcp://mosquitto:1883`
  - Topic: `8E215788_nx100_raw/+/data`
  - Data format: JSON
  - Measurement: `iot_raw`
- Output: `influxdb`
  - URL: `http://influxdb_repo:8086`
  - Database: `iotdata`
  - Timeout: 10s
  - Write consistency: any
- Agent:
  - Interval: 10s
  - Flush interval: 10s
  - Batch size: 500
  - Buffer limit: 10,000

### InfluxDB
- Version: **1.8.10 (LTS)**
- Database: `iotdata`
- Retention Policy:
  - `raw_90d` → เก็บข้อมูลดิบ 90 วัน (default)
  - `avg_1y` → เก็บข้อมูลเฉลี่ย 1 ปี
- Continuous Query:
  - `cq_downsample_5m` → ทำ downsample ทุก 5 นาที (mean) 
    จาก measurement `iot_raw` → `avg_1y.downsampled_5m`

### Grafana
- Version: **10.4.2**
- Port: `3000`
- Provisioning:
  - Datasource: `InfluxDB-iotdata`
  - URL: `http://influxdb_repo:8086`
  - Database: `iotdata`
  - ตั้งเป็น default datasource

---

## 🚀 วิธีเริ่มต้นใช้งาน

1. Clone repo:
   git clone https://github.com/your-org/iot-tig-repo.git  
   cd iot-tig-repo

2. รัน stack:
   docker-compose up -d

3. ตรวจสอบ service:
   - Mosquitto → tcp://localhost:1883  
   - InfluxDB → http://localhost:8086  
   - Grafana → http://localhost:3000 (user: admin / pass: admin)

4. ทดสอบ publish payload (จำลองอุปกรณ์):
   mosquitto_pub -h localhost -t "8E215788_nx100_raw/test/data" \
     -m '{"device":"esp32-001","power":120.5,"soc":86,"ts":1699999999}'

5. เปิด Grafana:
   - เข้า http://localhost:3000  
   - Datasource `InfluxDB-iotdata` ถูก provision ไว้แล้ว  
   - สร้าง panel ใหม่และ query measurement `iot_raw`

---

## 🛠 หมายเหตุสำหรับนักพัฒนา

ไฟล์ `.gitignore` ถูกตั้งค่าไว้เพื่อกันไม่ให้ commit ไฟล์ runtime และไฟล์ชั่วคราว เช่น:
- **InfluxDB runtime**: `influxdb/data/`, `influxdb/meta/`, `influxdb/wal/`  
- **Grafana runtime**: `grafana/grafana.db`, `grafana/png/`, `grafana/pdf/`, `grafana/csv/`  
- **Logs**: `*.log`, `logs/`  
- **ไฟล์จาก editor/OS**: `.DS_Store`, `.vscode/`, `.idea/`  
- **Secrets**: `.env`, `.env.*`  

ทำให้ repo นี้สะอาดและ reproducible ได้ง่าย
