# Part 80 — Model-Driven Telemetry & Streaming at Enterprise Scale
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 791–800 จาก 1000**

> ต่อจาก [Part 79 — Full-Stack Automation & CI/CD](part-079-fullstack-automation-cicd.md) ที่สร้าง
> **Pipeline สำหรับ Deploy/Test/Rollback Config** ให้กับ Enterprise Network ทั้งหมดที่หลักสูตรนี้
> สร้างมา — [Part 54](part-054-network-assurance-telemetry.md) เคยสอน **Model-Driven Telemetry
> (MDT)** ไว้เป็นพื้นฐานแล้ว ทั้ง Dial-in/Dial-out, gRPC/Protobuf, Python Receiver แบบง่าย, และ
> Anomaly Detection ด้วย Statistical Baseline — แต่ Part 54 นั้นสอนในสเกลของ **1 Collector รับจาก
> ไม่กี่อุปกรณ์** ซึ่งใช้ได้ดีสำหรับ Lab แต่ยังไม่พอสำหรับ Enterprise/Service Provider ระดับ CCIE
> ที่มีอุปกรณ์เป็นร้อยเป็นพันตัว Part 80 นี้คือ Part ที่ยกระดับทุกอย่างจาก Part 54 ขึ้นไปสู่
> **Scale การผลิตจริง**: (1) แก้ปัญหา Data Volume ด้วย Kafka เป็น Buffering Layer, (2) เจาะลึก
> **gNMI** มาตรฐานใหม่ที่ทดแทน MDT แบบ Vendor-neutral, (3) ออกแบบ Time-series Schema และ
> Retention Policy ที่ไม่ทำให้ Storage ระเบิด, (4) สร้าง Alerting Pipeline ที่ไม่ทำให้ NOC จม
> กับ Alert Fatigue, (5) ปิด Loop กับ CI/CD Pipeline ของ [Part 79](part-079-fullstack-automation-cicd.md)
> ให้ Anomaly ที่ Detect ได้ **Trigger Rollback อัตโนมัติ**, และ (6) ปิดท้ายด้วย Lab เต็มรูปแบบที่
> รวมทุกองค์ประกอบเข้าเป็น Production Telemetry Pipeline หนึ่งเดียว

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 791 | Telemetry ที่ Enterprise Scale — ปัญหา Data Volume และ Architecture แก้ปัญหาด้วย Kafka |
| 792 | Dial-in vs Dial-out ที่ Scale — ทำไม Dial-out ชนะเกือบทุกกรณี Production |
| 793 | gRPC Dial-out Pipeline เชิงลึก — Telegraf `cisco_telemetry_mdt`/`cisco_telemetry_gnmi` แบบ Production |
| 794 | gNMI — มาตรฐาน Streaming Telemetry แบบ Vendor-neutral, Subscribe/Get/Set |
| 795 | Time-series Data Modeling — ออกแบบ Schema InfluxDB/Prometheus, Downsampling/Retention |
| 796 | Alerting & Anomaly Detection ที่ Scale — Prometheus Alertmanager, ลด Alert Fatigue |
| 797 | Telemetry-driven Automated Remediation — ปิด Loop กับ CI/CD Pipeline (Part 79) |
| 798 | Correlating Telemetry ข้าม Domain — Dashboard เดียวเห็น BGP/QoS/Interface พร้อมกัน |
| 799 | Telemetry Security — TLS/Authentication บน gRPC Pipeline |
| 800 | Lab เต็มรูปแบบ: Production Telemetry Pipeline — Telegraf + Kafka + InfluxDB + Grafana + Alertmanager + Auto-Rollback |

---

## Step 791 — Telemetry ที่ Enterprise Scale: ปัญหา Data Volume

### ทวนจุดที่ Part 54 หยุดไว้

[Part 54](part-054-network-assurance-telemetry.md) สอน MDT บน **6 อุปกรณ์** (CORE-SW1/2,
DIST-SW1-4) Stream เข้า Collector ตัวเดียว (NOC-SRV) ด้วย Sensor-Path 3 กลุ่ม (CPU/Interface/
Memory) — สเกลนี้ Telegraf ตัวเดียวรับได้สบายๆ ไม่มีปัญหา แต่ลองคำนวณ Load จริงของ Enterprise
ขนาดกลาง-ใหญ่ที่ CCIE ต้องออกแบบให้รองรับได้

### คำนวณ Data Volume จริง

| ตัวแปร | ค่าตัวอย่าง Enterprise ขนาดกลาง-ใหญ่ |
|---|---|
| จำนวนอุปกรณ์ที่ Stream Telemetry | 500 อุปกรณ์ (Core/Dist/Access/WAN Edge/Firewall) |
| จำนวน Sensor-Path ต่ออุปกรณ์ (Interface × ทุกพอร์ต, CPU, Memory, BGP/OSPF State, QoS Queue, ...) | ~40 Sensor-Path (รวม Sub-path ที่ Expand เป็นหลาย Leaf) |
| Sample-interval เฉลี่ย (บาง Path 5s, บาง Path 60s) | เฉลี่ยรวม ~10 วินาที |
| ขนาด Message เฉลี่ยต่อ Sample (encode-kvgpb, รวม GPBKV Overhead) | ~2 KB |
| **จำนวน Message ต่อวินาที** | 500 × 40 ÷ 10 ≈ **2,000 message/วินาที** |
| **Throughput ขาเข้า Collector** | 2,000 × 2 KB ≈ **4 MB/วินาที ≈ 345 GB/วัน** |

ตัวเลขนี้คือ **Load ขาเข้าอย่างเดียว** — ยังไม่รวม Write Amplification ที่เกิดขึ้นเมื่อ Time-series
Database ต้อง Index ทุก Tag/Field (Step 795), ยังไม่รวม Query Load จาก Grafana Dashboard ที่หลาย
คนเปิดพร้อมกัน และยังไม่รวมกรณีที่ต้อง Re-process ข้อมูลเดิมใหม่ (เช่น เปลี่ยน Downsampling Rule)

### ปัญหาที่เกิดขึ้นเมื่อ Collector เดียวรับตรงจากทุกอุปกรณ์

```
Design แบบ Part 54 ขยายตรงๆ ไปที่ 500 อุปกรณ์: Device 1..500 ──gRPC Dial-out (500 Connection)──►
Telegraf ตัวเดียว (single process: รับ Connection 500 เส้น + Decode encode-kvgpb ทุก Message
CPU-bound + เขียนตรงเข้า InfluxDB ทุก Batch แบบ Blocking I/O) ──► ถ้า InfluxDB ช้าลงชั่วขณะ
(Compaction, Disk I/O spike) Telegraf Buffer เต็ม → เริ่ม Drop Metric หรือ Backpressure ไหลกลับ
ไปที่ gRPC Connection → Device บางตัว Disconnect ตอน Buffer เต็ม
```

ปัญหาหลักคือ **Collector กับ Storage ถูก Couple กันแน่นเกินไป**: ถ้า InfluxDB ช้าลงแม้ชั่วขณะ
(เช่นตอน Compaction รอบดึก) Telegraf ที่รับ gRPC อยู่จะเริ่ม Buffer ค้างจนล้น แล้ว Drop ข้อมูลหรือ
ถูก Backpressure ดันกลับไปกระทบการรับ Connection ใหม่จากอุปกรณ์ — เหตุการณ์ Transient เพียง
ไม่กี่นาทีที่ Storage ก็ทำให้เกิด **Data Loss ที่ฝั่ง Collection** ซึ่งขัดกับเป้าหมายเดิมของ Streaming
Telemetry (เห็นข้อมูลครบ ไม่พลาด Micro-burst ตามที่ Part 54 Step 531 อธิบายไว้)

### สถาปัตยกรรมที่แก้ปัญหา: Kafka เป็น Buffering/Distribution Layer

**Apache Kafka** คือ Distributed Streaming Platform ที่ทำหน้าที่เป็น **Durable Message Queue**
ระหว่าง Collector กับ Storage — หลักการสำคัญคือ **Decouple การรับข้อมูล (Ingest) ออกจากการเขียน
ข้อมูล (Persist)** ทำให้ทั้งสองฝั่งขยายสเกลและ Fail ได้อย่างอิสระจากกัน

```
Architecture ที่ Scale ได้จริง (Enterprise/SP-grade):

500 Devices ──gRPC Dial-out (Load-balance ผ่าน VIP/DNS Round-robin)──► Telegraf ×N
(Collector Tier, stateless, scale horizontally) ──produce (topic: telemetry.raw)──► KAFKA
(3+ Broker, Replicated, Durable)
    ├─ consume ─► Telegraf/Consumer → InfluxDB (Storage Tier, Step 795)
    └─ consume ─► Anomaly Detector Stream (Step 797) → Trigger CI/CD
```

| ประโยชน์ของ Kafka Layer | รายละเอียด |
|---|---|
| **Decoupling** | Collector Tier (รับ gRPC) และ Storage Tier (เขียน InfluxDB) เป็นอิสระจากกันโดยสมบูรณ์ — Storage ช้า/ล่มชั่วคราวไม่กระทบการรับข้อมูลจากอุปกรณ์เลย เพราะ Kafka เก็บ Message ไว้ใน Topic ก่อน (Durable, Replicated ข้าม Broker) |
| **Multiple Consumer** | ข้อมูล Telemetry เดียวกัน (1 Message ใน Topic) ให้ **หลาย Consumer อ่านพร้อมกันได้** — Consumer หนึ่งเขียนเข้า InfluxDB (Step 795), อีก Consumer ทำ Real-time Anomaly Detection (Step 797) แบบ Parallel โดยไม่ต้อง Duplicate การรับข้อมูลจากอุปกรณ์ |
| **Horizontal Scale ทั้งสองฝั่ง** | Collector Tier เพิ่ม Telegraf Instance ได้ตามจำนวนอุปกรณ์ที่โตขึ้น (แต่ละ Instance รับ Connection เฉพาะกลุ่มตนเอง), Consumer ฝั่ง Storage ก็เพิ่ม Partition/Consumer Group ได้เพื่อ Throughput สูงขึ้นโดยไม่กระทบ Collector |
| **Replay ได้** | Message ใน Kafka Topic เก็บไว้ตาม Retention (เช่น 24-72 ชั่วโมง) — ถ้า Consumer มี Bug หรือ Schema เปลี่ยน สามารถ Reset Consumer Offset แล้ว **Re-process ข้อมูลเดิมใหม่** ได้โดยไม่ต้องรอ Device ส่งซ้ำ |

### ตารางเปรียบเทียบ: ควรใช้ Kafka เมื่อไหร่

| สเกล | แนะนำ |
|---|---|
| < 50 อุปกรณ์, Lab/PoC | Telegraf → InfluxDB ตรง (แบบ Part 54) เพียงพอ ไม่ต้องเพิ่มความซับซ้อนของ Kafka |
| 50-200 อุปกรณ์ | เริ่มพิจารณา Kafka ถ้าต้องการ Multiple Consumer (เช่น ต้องการทั้ง Storage และ Real-time Anomaly Detection พร้อมกัน) |
| > 200 อุปกรณ์ หรือ Multi-site/Multi-region | **ต้องมี Kafka** — Decoupling คือสิ่งที่ป้องกัน Data Loss เมื่อ Storage Tier มีปัญหาชั่วขณะ ซึ่งจะเกิดขึ้นแน่นอนที่สเกลนี้ (Compaction, Upgrade, Network Partition) |

> **หลักการสำคัญที่สุดของ Step นี้**: ที่สเกล CCIE ต้องออกแบบให้รองรับ **Telemetry Pipeline ไม่ใช่
> Single Point of Failure ของตัวเอง** — ถ้า Storage ล่มแล้วทำให้ข้อมูล Telemetry จากทั้งเครือข่าย
> หายไปด้วย นั่นคือ Pipeline ที่ออกแบบผิด Kafka คือคำตอบมาตรฐานของอุตสาหกรรมสำหรับปัญหานี้
> (Pattern เดียวกับที่ใช้ใน Log Aggregation, Event Sourcing, และ Data Engineering Pipeline ทั่วไป)

---

## Step 792 — Dial-in vs Dial-out ที่ Scale

### ทวนจาก Part 54 แล้วขยายมุมมองเชิง Scale

[Part 54 Step 532](part-054-network-assurance-telemetry.md) สอนความแตกต่างพื้นฐานไว้แล้ว
(ใครเริ่มเชื่อมต่อ, Firewall/NAT Friendliness) — Step นี้เจาะประเด็นที่สำคัญที่สุดสำหรับสเกล
Enterprise: **Collector ต้องรู้จัก "ที่อยู่" ของอุปกรณ์กี่ตัว และต้องอัปเดตรายชื่อนั้นบ่อยแค่ไหน**

### ปัญหาของ Dial-in ที่ Scale — Collector ต้องดูแล "Inventory" ของทุกอุปกรณ์

```python
# Dial-in ที่ 500 อุปกรณ์ — Collector ต้องรู้ Address ของทุกตัวและ "ไปเชื่อมต่อเอง"
device_list = [
    {"host": "10.255.10.2", "port": 830, ...},   # DIST-SW1
    {"host": "10.255.20.2", "port": 830, ...},   # DIST-SW2
    # ... × 500 รายการ ที่ Collector ต้องดูแลเป็น Inventory File/Database
]
# → อุปกรณ์ใหม่เข้าเครือข่าย ต้องแก้ List นี้ / IP เปลี่ยน (DHCP Renew, Re-IP) ต้อง Sync
# → Collector ต้องเปิด Outbound Connection ไปยัง 500 ปลายทางพร้อมกัน (Connection Pool หนักขึ้น)
```

| ปัญหาของ Dial-in ที่ Scale | ผลกระทบ |
|---|---|
| **Inventory Management เป็นภาระถาวร** | ต้องมี Source of Truth (เช่น NetBox, CMDB) ที่ Sync กับ Collector ตลอดเวลา ทุกอุปกรณ์ใหม่ต้องลงทะเบียนก่อนถึง Monitor ได้ |
| **Collector เป็นฝ่าย Initiate หลาย Connection พร้อมกัน** | Connection Pool/Timeout Management ฝั่ง Collector ซับซ้อนขึ้นตามจำนวนอุปกรณ์ ถ้าอุปกรณ์ตัวหนึ่ง Unreachable Collector ต้อง Retry/Backoff เองสำหรับทุกตัวแยกกัน |
| **NAT/Firewall Rule ต้องเปิดขาเข้าไปทุกอุปกรณ์** | ยิ่งอุปกรณ์เยอะ ยิ่งมี Rule ขาเข้าจำนวนมากที่ต้อง Maintain (ตรงข้ามกับหลักการ Trust Boundary ที่ Part 54 Step 532 อธิบาย) |

### ทำไม Dial-out ชนะที่ Scale — Collector ไม่ต้องรู้จักใครเลย

กลับด้านกันโดยสิ้นเชิง: ทุกอุปกรณ์ Push เข้า Destination เดียวกัน (เหมือน `logging host`
เดียวกันที่อุปกรณ์ทุกตัว Config ไว้) Collector รู้แค่ "Listen ที่ Port เดียว" โดยไม่ต้องรู้ IP
ของอุปกรณ์แม้แต่ตัวเดียว — ที่ฝั่งอุปกรณ์ใช้ Config Template เดียวกัน (เหมือน Part 54 Step 540)
เปลี่ยนแค่ `source-address` แล้ว Deploy ผ่าน Ansible/NETCONF (Part 48-49) ได้พร้อมกับตอน
Provision อุปกรณ์ใหม่เลย

| มุมมอง | Dial-in | Dial-out |
|---|---|---|
| Collector ต้องรู้จักอุปกรณ์กี่ตัว | **ทุกตัว** (ต้องมี Inventory ครบ) | **ไม่ต้องรู้เลยแม้แต่ตัวเดียว** — Config ที่ Collector คือ "รอรับที่ Port X" เท่านั้น |
| เพิ่มอุปกรณ์ใหม่ 1 ตัว | ต้องแก้ Collector Config/Inventory | เพิ่ม Telemetry Config Block เดียวกัน (Template) บนอุปกรณ์ใหม่ตอน Provision — Collector ไม่ต้องแก้อะไรเลย |
| ความสัมพันธ์กับ CI/CD (Part 79) | Onboarding อุปกรณ์ใหม่ต้องมี Pipeline Step แยกไปอัปเดต Collector Inventory | Onboarding อุปกรณ์ใหม่ผ่าน Pipeline เดียวกับที่ Push Config อื่นๆ ทั้งหมด (Golden Config Template รวม Telemetry Block ไว้ในตัว) |
| **สรุปที่สเกล Enterprise** | ไม่เหมาะเป็น Default | **เป็น Default มาตรฐานสำหรับ Production Monitoring ถาวร** |

### แล้ว Dial-in ยังมีที่ใช้ไหมที่ Scale — ใช่ สำหรับ Ad-hoc Deep-dive

Dial-in ไม่ได้ "ตายไปเลย" ที่ Enterprise Scale — มันเปลี่ยนบทบาทจาก **Production Monitoring**
ไปเป็น **Troubleshooting Tool ชั่วคราว**:

| Use Case ที่ Dial-in ยังเหมาะที่สุด | เหตุผล |
|---|---|
| Engineer สงสัยปัญหาที่ Interface เฉพาะจุด ต้องการดู Counter ทุก 1 วินาที **ชั่วคราว 5 นาที** เพื่อ Debug | Dial-out ต้อง Config ล่วงหน้าและอยู่ถาวรใน Running-config — ไม่คุ้มที่จะแก้ Production Config เพื่อ Debug ชั่วคราว |
| Tool วิเคราะห์ที่ Engineer รันจาก Laptop ตัวเอง เชื่อมเข้าอุปกรณ์โดยตรงแบบ Interactive (คล้าย `show` command แบบ Streaming) | ไม่ต้องพึ่ง Collector Pipeline ที่ซับซ้อนเลย — ใช้ gNMI CLI Client (Step 794) ยิง Subscribe ตรงจาก Laptop ได้ทันที |
| Automation Script ที่ต้องการ Subscribe/Unsubscribe แบบ Dynamic ตาม Logic ของ Script เอง (เช่น เปิด Subscribe เฉพาะช่วง Maintenance Window) | Dial-in ทำผ่าน RPC Call ได้ทันทีไม่ต้องแก้ Persistent Config |

> **สรุป Step 792**: ที่ Enterprise Scale ให้ใช้ **Dial-out เป็นค่าเริ่มต้นสำหรับ Production
> Monitoring ถาวรเสมอ** (เหตุผลหลักคือ Collector ไม่ต้องรู้จัก Inventory ของอุปกรณ์เลย ซึ่งลด
> Operational Overhead ได้มหาศาลเมื่ออุปกรณ์เป็นร้อยเป็นพันตัว) แต่เก็บ **Dial-in ไว้เป็น Tool
> สำหรับการ Deep-dive แบบ Ad-hoc** ที่ Engineer ควบคุมเองแบบ Interactive โดยไม่ต้องแก้ Production
> Config Baseline

---

## Step 793 — gRPC Dial-out Pipeline เชิงลึก: Telegraf แบบ Production

### จาก Custom Python Receiver (Part 54) ไปสู่ Production-grade Collector

[Part 54 Step 536](part-054-network-assurance-telemetry.md) เขียน `mdt_grpc_receiver.py` ขึ้นมา
เอง **เพื่อการศึกษา** — Script นั้นบอกตรงๆ ว่า `decode_kvgpb()` เป็น Placeholder และแนะนำให้ใช้
Telegraf ใน Production Step นี้คือ Step ที่ทำตามคำแนะนำนั้นอย่างเต็มรูปแบบ พร้อม `telegraf.conf`
ที่ครบทุกส่วนจริง ไม่ใช่แค่ Snippet สั้นๆ แบบ Step 535 ของ Part 54

### สอง Input Plugin ที่ต้องเข้าใจความต่าง

| Input Plugin | Protocol ที่รับ | ใช้กับ |
|---|---|---|
| `cisco_telemetry_mdt` | Cisco Native MDT (gRPC Dial-out, `encode-kvgpb`) | อุปกรณ์ Cisco ที่ Config ตาม [Part 54 Step 534](part-054-network-assurance-telemetry.md) โดยตรง (`telemetry ietf ...`) |
| `cisco_telemetry_gnmi` | **gNMI** (มาตรฐาน Vendor-neutral, Step 794) | อุปกรณ์ Cisco รุ่นใหม่ที่รองรับ gNMI Server, หรือ Mixed-vendor Environment |

### `telegraf.conf` เต็มรูปแบบ — Production Collector Tier บน NOC-SRV

```toml
# /etc/telegraf/telegraf.conf
# Production-grade Collector Tier สำหรับ Part 80 Lab
# รับ Dial-out จาก CORE-SW1/2, DIST-SW1-4 (MDT) และเตรียมพร้อมสำหรับ gNMI (Step 794)

[global_tags]
  environment = "ccie-lab"
  site = "hq-campus"

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 100000     # กัน Data Loss ช่วง Storage ช้าชั่วขณะ (Step 791)
  flush_interval = "10s"
  flush_jitter = "3s"
  precision = "1ms"
  hostname = "noc-srv-collector-01"

# INPUT 1: Cisco Native MDT (ตาม Part 54 Step 534 ที่ Config ไว้บนอุปกรณ์)
[[inputs.cisco_telemetry_mdt]]
  transport = "grpc"
  service_address = ":57500"
  max_msg_size = 4000000
  aliases = { "ifstats" = "Cisco-IOS-XE-interfaces-oper:interfaces/interface" }

  # Embedded TLS สำหรับ Production (Step 799) — comment ไว้สำหรับ Lab เริ่มต้นแบบ insecure
  # tls_cert = "/etc/telegraf/certs/collector.pem"
  # tls_key  = "/etc/telegraf/certs/collector.key"
  # tls_allowed_cacerts = ["/etc/telegraf/certs/ca.pem"]

# INPUT 2: gNMI Dial-out (Step 794) — สำหรับอุปกรณ์/Vendor ที่ใช้ gNMI แทน MDT
[[inputs.cisco_telemetry_gnmi]]
  transport = "grpc"
  service_address = ":57501"
  max_msg_size = 4000000

# PROCESSOR: แปลง Tag ให้อ่านง่าย + Drop Field ที่ไม่จำเป็น (ลด Data Volume ตาม Step 791)
[[processors.rename]]
  [[processors.rename.replace]]
    tag = "source"
    dest = "device_ip"

[[processors.strings]]
  [[processors.strings.trim]]
    tag = "path"

# OUTPUT: Kafka (Step 791) — ไม่เขียน InfluxDB ตรง แต่ส่งเข้า Kafka Topic ก่อน
[[outputs.kafka]]
  brokers = ["kafka-01:9092", "kafka-02:9092", "kafka-03:9092"]
  topic = "telemetry.raw"
  routing_key = "device_ip"          # Partition ตาม Device เพื่อรักษาลำดับ Message ต่ออุปกรณ์
  compression_codec = "snappy"        # ลด Bandwidth ระหว่าง Telegraf กับ Kafka Broker
  required_acks = 1                    # Balance ระหว่าง Durability กับ Latency
  max_retry = 3

  data_format = "json"
```

| ส่วนของ Config | เหตุผลที่ตั้งค่าแบบนี้ |
|---|---|
| `metric_buffer_limit = 100000` | ถ้า Kafka Broker ไม่ตอบสนองชั่วขณะ (Network Blip, Broker Election) Telegraf เก็บ Metric ไว้ใน Memory Buffer ก่อนโดยไม่ Drop ทันที — ค่านี้ต้องคำนวณจาก Throughput จริง (Step 791: ~2,000 msg/s × Buffer ที่ต้องการทน 30 วินาที ≈ 60,000 อย่างน้อย) |
| `routing_key = "device_ip"` | Kafka Partition ตาม Device IP ทำให้ Message จากอุปกรณ์เดียวกันอยู่ Partition เดียวกันเสมอ **รักษาลำดับเวลา (Ordering)** ของ Sample จากอุปกรณ์นั้น ซึ่งสำคัญมากสำหรับ Correlation (Step 798) |
| `compression_codec = "snappy"` | ลด Network Overhead ระหว่าง Collector Tier กับ Kafka Cluster — สำคัญเมื่อ Throughput สูงถึงระดับ MB/s ตาม Step 791 |
| `required_acks = 1` | Broker ผู้นำ (Leader) Ack กลับก่อนถือว่าสำเร็จ (ไม่รอ Follower Replica ครบ) — Trade-off ระหว่าง Latency กับ Durability สูงสุด (`required_acks = -1`/`all`) ที่ยอมรับได้สำหรับ Telemetry (ต่างจาก Financial Transaction ที่ต้องการ `all`) |

### Consumer ฝั่ง Storage — Telegraf ตัวที่สองอ่านจาก Kafka เขียนเข้า InfluxDB

```toml
# /etc/telegraf/telegraf-storage-consumer.conf
# Storage Tier — Consumer ที่แยก Process จาก Collector Tier โดยสมบูรณ์ (Step 791 Decoupling)

[agent]
  interval = "10s"
  metric_batch_size = 5000
  flush_interval = "5s"

[[inputs.kafka_consumer]]
  brokers = ["kafka-01:9092", "kafka-02:9092", "kafka-03:9092"]
  topics = ["telemetry.raw"]
  consumer_group = "influxdb-writer"
  offset = "oldest"                  # เริ่มจาก Offset เดิมถ้า Consumer เพิ่งเริ่ม (Replay ได้)
  max_processing_time = "1s"
  data_format = "json"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb-01:8086"]
  token = "$INFLUX_TOKEN"
  organization = "netops-lab"
  bucket = "telemetry"
```

> **จุดสำคัญของสถาปัตยกรรมนี้**: Collector Tier (`telegraf.conf`) และ Storage Tier
> (`telegraf-storage-consumer.conf`) เป็น **Process แยกกันคนละเครื่องได้** เชื่อมกันผ่าน Kafka
> Topic เท่านั้น — ตรงตามหลักการ Decoupling ที่ Step 791 อธิบายไว้ ถ้าต้องการเพิ่ม Anomaly
> Detector เป็น Consumer ตัวที่สาม (Step 797) ก็เพิ่ม `consumer_group` ใหม่อ่าน Topic เดียวกันได้
> โดยไม่กระทบ Consumer ตัวอื่นเลย

---

## Step 794 — gNMI: มาตรฐาน Streaming Telemetry แบบ Vendor-neutral

### gNMI คืออะไร และต่างจาก MDT อย่างไร

**gNMI (gRPC Network Management Interface)** คือ Protocol ที่พัฒนาโดย **OpenConfig Working
Group** (กลุ่มที่ Google และ Operator รายใหญ่หลายเจ้าร่วมกันผลักดัน) บนพื้นฐาน gRPC เดียวกับที่
[Part 54 Step 533](part-054-network-assurance-telemetry.md) สอนไว้ — ต่างจาก MDT ที่ใช้ RPC
เฉพาะของ Cisco (`MdtDialout`) และ Encoding แบบ `encode-kvgpb` (Cisco-specific KV format) gNMI
ใช้ RPC มาตรฐาน `Subscribe`/`Get`/`Set` กับ Encoding JSON_IETF/Protobuf ตาม Spec เดียวกันทุก
Vendor ทำให้ MDT "ใช้ได้กับ Cisco เท่านั้น" (แม้ YANG Model จะมาตรฐานแต่ RPC/Encoding เป็นของ
Cisco) ในขณะที่ Client/Collector ตัวเดียวที่ใช้ gNMI คุยกับ Cisco, Juniper, Arista ด้วย Code
เดียวกันได้เลย

### สาม RPC หลักของ gNMI

| RPC | หน้าที่ | เทียบเท่าแนวคิดกับที่เรียนมาก่อน |
|---|---|---|
| **Capabilities** | Client ถามอุปกรณ์ว่ารองรับ YANG Model/Encoding อะไรบ้าง | คล้าย NETCONF `<hello>` ที่ [Part 49](part-049-netconf-restconf-yang-deep-dive.md) สอนไว้ |
| **Get** | ดึงค่า Config หรือ State ณ ขณะนั้นครั้งเดียว (คล้าย Pull) | เทียบเท่า RESTCONF GET ([Part 49 Step 485](part-049-netconf-restconf-yang-deep-dive.md)) |
| **Set** | เปลี่ยนแปลง Config (Update/Replace/Delete) | เทียบเท่า NETCONF `edit-config` |
| **Subscribe** | **Streaming Telemetry ตัวหลักของ Step นี้** — Subscribe แล้วรับ Stream ต่อเนื่อง | เทียบเท่า MDT `subscription` ของ [Part 54 Step 534](part-054-network-assurance-telemetry.md) |

### Subscribe Mode ของ gNMI

| Mode/sample_mode | พฤติกรรม | เทียบเท่าใน MDT |
|---|---|---|
| `ONCE` | Subscribe แล้วรับข้อมูลครั้งเดียวแล้วปิด Stream — เหมือน Get แต่ผ่าน Subscribe RPC | ไม่มีเทียบเท่าตรง (MDT เป็น Continuous เสมอ) |
| `POLL` | Client ส่ง Poll Trigger เมื่อต้องการข้อมูลรอบใหม่ — Subscribe ค้างไว้แต่ Client คุมจังหวะ | คล้าย Dial-in ที่ Client ควบคุม |
| `STREAM` + `SAMPLE` | **ใช้บ่อยที่สุด** — Push ตาม Cadence คงที่ต่อเนื่อง | เทียบเท่า `sample-interval` ของ MDT Subscription |
| `STREAM` + `ON_CHANGE` | Push เฉพาะเมื่อค่าเปลี่ยนจริง — ประหยัด Bandwidth มากสำหรับ Metric ที่นิ่งเป็นส่วนใหญ่ (เช่น Interface Admin State) | ไม่มีเทียบเท่าตรงใน MDT (MDT เป็น Cadence-based เท่านั้น) |

### Python gNMI Client เต็มรูปแบบ — ใช้ Library `pygnmi`

```bash
pip install pygnmi
```

```python
#!/usr/bin/env python3
# gnmi_subscribe_client.py — gNMI Subscribe Client (Dial-in Mode ตาม Step 792, Ad-hoc Deep-dive)
# เชื่อมต่อไปยัง DIST-SW1 โดยตรงเพื่อ Subscribe Interface Counter แบบ SAMPLE ทุก 10 วินาที ผ่าน
# Library pygnmi ที่ Wrap gNMI Protobuf Stub ให้ใช้งานง่ายกว่าเขียน grpc stub เอง

from pygnmi.client import gNMIclient

# ตาม 00-ip-address-plan.md: DIST-SW1 Loopback0 = 1.1.1.11, MGMT ผ่าน VLAN 99
HOST = ("10.10.99.11", 50051)     # gNMI Target — Port 50051 เป็นค่ามาตรฐานทั่วไปของ gNMI

# YANG Path แบบ OpenConfig — ต่างจาก Cisco Native Path ที่ Part 54 ใช้ (Vendor-neutral ตาม Step 794)
SUBSCRIBE_PATHS = [
    "/interfaces/interface[name=TenGigabitEthernet1/1/1]/state/counters",
    "/system/cpus/cpu/state",
]


def run_subscribe():
    with gNMIclient(target=HOST, username="admin", password="admin", insecure=True) as client:
        # Capabilities ก่อนเสมอ — ยืนยันว่าอุปกรณ์รองรับ YANG Model ที่จะ Subscribe จริง
        capabilities = client.capabilities()
        print(f"[*] Supported Models: {len(capabilities.get('supported_models', []))} รายการ")

        subscribe_request = {
            "subscription": [
                {"path": path, "mode": "sample", "sample_interval": 10_000_000_000}  # nanosecond
                for path in SUBSCRIBE_PATHS
            ],
            "mode": "stream",
            "encoding": "json_ietf",
        }

        print(f"[*] เริ่ม Subscribe (STREAM/SAMPLE) ไปยัง {HOST[0]}:{HOST[1]} ...")
        telemetry_stream = client.subscribe(subscribe=subscribe_request)

        for response in telemetry_stream:
            if "update" in response:
                for update in response["update"].get("update", []):
                    print(f"    Path: {update['path']:60s} Value: {update['val']}")


def run_get_snapshot():
    """ตัวอย่าง Get RPC — ดึงค่าครั้งเดียวแบบ Pull (ต่างจาก Subscribe ที่เป็น Stream)"""
    with gNMIclient(target=HOST, username="admin", password="admin", insecure=True) as client:
        result = client.get(path=["/system/state/hostname"])
        print(f"[*] Get Snapshot: {result}")


if __name__ == "__main__":
    run_get_snapshot()
    run_subscribe()
```

### ตัวอย่าง Output ที่คาดหวัง

```
$ python3 gnmi_subscribe_client.py
[*] Get Snapshot: {'notification': [{'update': [{'path': 'system/state/hostname', 'val': 'DIST-SW1'}]}]}
[*] Supported Models: 47 รายการ
[*] เริ่ม Subscribe (STREAM/SAMPLE) ไปยัง 10.10.99.11:50051 ...
    Path: interfaces/interface[name=TenGigabitEthernet1/1/1]/state/counters   Value: {'in-octets': '128456789', 'in-errors': '0'}
    Path: system/cpus/cpu/state                                              Value: {'total': {'instant': '18'}}
    Path: interfaces/interface[name=TenGigabitEthernet1/1/1]/state/counters   Value: {'in-octets': '129012044', 'in-errors': '0'}
```

### ตารางเปรียบเทียบสรุป: เมื่อไหร่ใช้ MDT เมื่อไหร่ใช้ gNMI

| ประเด็น | MDT (Cisco Native) | gNMI |
|---|---|---|
| Environment ที่เหมาะ | All-Cisco Environment ที่ต้องการ Sensor-Path ละเอียดแบบ Cisco-specific (เช่น Platform-specific Hardware Counter) | Multi-vendor Environment หรือต้องการ Tooling ที่ Portable ข้าม Vendor |
| Encoding | `encode-kvgpb` (Cisco proprietary structure) | JSON_IETF หรือ Protobuf ตาม OpenConfig Spec มาตรฐาน |
| Ecosystem Tooling | Telegraf `cisco_telemetry_mdt`, Cisco Crosswork | Telegraf `cisco_telemetry_gnmi`, `pygnmi`, gNMIc (CLI Tool มาตรฐาน), รองรับกว้างกว่าในโลก Open Source |
| ทิศทางอุตสาหกรรม | ยังใช้กว้างขวางในฐาน Cisco เดิม | **กำลังเป็น De-facto Standard ใหม่** สำหรับ Deployment ที่คาดว่าต้อง Multi-vendor ในอนาคต |
| ใช้ในหลักสูตรนี้ | Config หลักบน Cisco IOS-XE (Part 54 + Step 793) | เสริมสำหรับ Ad-hoc Deep-dive (Step 792) และเตรียมพร้อม Multi-vendor (Step 800 Lab) |

> **คำแนะนำสำหรับ Production ปัจจุบัน**: ถ้า Environment เป็น All-Cisco และใช้ Cisco Crosswork
> อยู่แล้ว MDT ยังเป็นตัวเลือกที่ Integrate ง่ายที่สุด แต่ถ้าออกแบบ Pipeline ใหม่ที่ต้องรองรับ
> Vendor อื่นในอนาคต (M&A, Multi-vendor Data Center) ให้เลือก **gNMI เป็น Default** เพราะ Client/
> Collector Code เดียวกันจะใช้ต่อได้โดยไม่ต้องเขียนใหม่ทั้งหมดเมื่อ Vendor เปลี่ยน — ตรงตามหลักการ
> เดียวกับที่ [Part 49](part-049-netconf-restconf-yang-deep-dive.md) ผลักดันเรื่อง Standard YANG
> Model เหนือ CLI Scraping

---

## Step 795 — Time-series Data Modeling สำหรับ Network Telemetry

### ทำไม Schema Design สำคัญกว่าที่คิด

Part 54 ไม่ได้พูดถึง Schema Design เลย เพราะ Volume ต่ำพอที่จะไม่ต้องคิดเรื่องนี้ — ที่สเกล
Enterprise (Step 791: 345 GB/วัน) **Schema ที่ออกแบบผิด** ทำให้ InfluxDB ใช้ Memory และ Disk
มากกว่าที่ควรหลายเท่า และทำให้ Query ช้าลงจนใช้งานจริงไม่ได้

### หลักการ Tag vs Field ของ InfluxDB (Line Protocol)

InfluxDB เก็บข้อมูลแบบ **Line Protocol**: `measurement,tag1=v1,tag2=v2 field1=v1,field2=v2 timestamp`
— ความต่างระหว่าง Tag กับ Field คือหัวใจของการออกแบบ Schema ที่ดี:

| | Tag | Field |
|---|---|---|
| ถูก Index หรือไม่ | **ใช่ — Indexed** (Query เร็วมากเมื่อ Filter ด้วย Tag) | ไม่ Index (ต้อง Scan) |
| ใช้กับข้อมูลแบบไหน | ค่าที่ **ไม่เปลี่ยนบ่อยและมี Cardinality ต่ำ-กลาง** (device, interface, site) | ค่าที่ **เปลี่ยนตลอดเวลา** (counter, utilization %, error count) |
| ผลถ้าใช้ผิดประเภท | Tag ที่มี Cardinality สูงเกินไป (เช่น Timestamp เป็น Tag) ทำให้ Index บวมจนล้น Memory ("Series Cardinality Explosion") | Field ที่ควรเป็น Tag (เช่น interface name เป็น Field) ทำให้ Filter/Group By ช้ามาก |

### ตัวอย่าง Schema ที่ออกแบบถูกและผิด

```
❌ ผิด: interface_stats,device=CORE-SW1 interface="Te1/0/1",in_octets=128456789
        (interface เป็น Field — Query "หา in_octets ของ Te1/0/1" ต้อง Scan ทุก Row
         เพราะไม่ได้ Index, ที่ 500 อุปกรณ์ × 48 พอร์ต ช้าลงเป็นวินาทีถึงนาที)

✅ ถูก:  interface_stats,device=CORE-SW1,interface=Te1/0/1,site=hq-campus in_octets=128456789,in_errors=0
        (device/interface/site เป็น Tag → Indexed, in_octets/in_errors เป็น Field
         → Query filter ด้วย device+interface ใช้ Index ตรง เร็วระดับ millisecond แม้ Volume สูง)
```

| หลักการเลือก Tag/Field สำหรับ Telemetry ของหลักสูตรนี้ | ตัวอย่างจาก Lab |
|---|---|
| **Tag**: อะไรที่ Engineer จะใช้ `WHERE`/`GROUP BY` บ่อย และ Cardinality ไม่สูงเกินไป (< ~100,000 Series รวม) | `device`, `interface`, `site`, `vrf`, `protocol` (BGP/OSPF) |
| **Field**: ค่า Metric จริงที่เปลี่ยนทุก Sample | `in_octets`, `in_errors`, `cpu_percent`, `ospf_neighbor_count`, `bgp_prefix_count` |
| **หลีกเลี่ยงเป็น Tag เด็ดขาด** | Timestamp, Session ID ที่ Unique ทุกครั้ง, Counter Value เอง (Cardinality สูงเกินไปจนล้น) |

### Downsampling และ Retention Policy — เก็บ Raw Data ตลอดไปไม่ได้

ที่ 345 GB/วัน (Step 791) ถ้าเก็บ Raw Data ไว้ตลอดไปจะใช้ **~10 TB ภายใน 1 เดือน** — ต้องมี
กลยุทธ์ลดขนาดข้อมูลเก่าที่ยังต้องการเห็น Trend ระยะยาวแต่ไม่ต้องการ Resolution ระดับวินาทีอีกต่อไป

กลยุทธ์ Retention/Downsampling แบบ Tiered (ใช้ใน InfluxDB Tasks / Continuous Query):
**Raw Data** (Resolution เต็ม 10s, เก็บ 7 วัน, Bucket `telemetry`) → downsample mean/max ทุก
5 นาที → **5-minute Rollup** (เก็บ 90 วัน, Bucket `telemetry_5m`) → downsample ทุก 1 ชั่วโมง →
**1-hour Rollup** (เก็บ 2 ปี, Bucket `telemetry_1h`)

### InfluxDB Task (Flux) สำหรับ Downsampling อัตโนมัติ

```flux
// downsample_5m.flux
// รันทุก 5 นาทีผ่าน InfluxDB Task Scheduler — คำนวณ mean/max ของ 5 นาทีล่าสุด
// จาก Bucket "telemetry" (Raw) เขียนเข้า Bucket "telemetry_5m" (Rollup)

option task = {name: "downsample-5m", every: 5m}

data = from(bucket: "telemetry")
    |> range(start: -5m)
    |> filter(fn: (r) => r._measurement == "interface_stats")

data
    |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
    |> set(key: "_measurement", value: "interface_stats_5m")
    |> to(bucket: "telemetry_5m", org: "netops-lab")

// เก็บ Max ของ error rate ไว้ต่างหาก — mean จะ "กลบ" Spike สั้นๆ ที่สำคัญต่อ Alerting (Step 796)
data
    |> filter(fn: (r) => r._field == "in_errors")
    |> aggregateWindow(every: 5m, fn: max, createEmpty: false)
    |> set(key: "_measurement", value: "interface_errors_max_5m")
    |> to(bucket: "telemetry_5m", org: "netops-lab")
```

> **จุดสำคัญที่มักถูกมองข้าม**: เมื่อ Downsample ด้วย `mean` (ค่าเฉลี่ย) Metric ประเภท Error/Spike
> ที่ Part 54 Step 537 บอกว่าสำคัญมากสำหรับ Anomaly Detection **จะถูก "กลบ" หายไป** เพราะค่าเฉลี่ย
> ของ Spike สั้นๆ ปนกับค่าปกติจะดูเหมือนไม่มีอะไรผิดปกติ — ทางแก้คือเก็บ `max` แยกไว้อีก Field
> เสมอสำหรับ Metric ที่เป็น "Leading Indicator" (Error Rate, Latency Spike, Queue Drop) ตามที่
> แสดงใน Flux Script ด้านบน

### Prometheus/PromQL — ทางเลือกสำหรับ Metric แบบ Pull + Alerting

InfluxDB เหมาะกับ Time-series ที่มาจาก Push (Telemetry Dial-out) โดยตรง แต่ **Prometheus**
(ที่ Grafana/Alertmanager Ecosystem ผูกกันแน่นมาก — Step 796) ใช้ **Pull Model** เป็นค่าเริ่มต้น
— ในสถาปัตยกรรมนี้ใช้ **`telegraf` Output Plugin `prometheus_client`** เป็นสะพาน ให้ Prometheus
มา Scrape Metric ที่ Telegraf แปลงจาก Push (gRPC) ให้เป็น Pull (HTTP `/metrics` Endpoint) ได้

```toml
# เพิ่มใน telegraf-storage-consumer.conf (Step 793) — ให้ Prometheus Scrape ได้ด้วย
[[outputs.prometheus_client]]
  listen = ":9273"
  metric_version = 2
  path = "/metrics"
```

```yaml
# prometheus.yml — ส่วน scrape_configs ที่เพิ่มเข้ามา
scrape_configs:
  - job_name: "telegraf-telemetry"
    scrape_interval: 15s
    static_configs:
      - targets: ["noc-srv-collector-01:9273"]
```

---

## Step 796 — Alerting & Anomaly Detection ที่ Scale

### ทวนข้อจำกัดของ Part 54 Step 539

[Part 54 Step 539](part-054-network-assurance-telemetry.md) เขียน `anomaly_detector_
remediation.py` ที่ตรวจ Anomaly ด้วย Mean + N-Sigma แล้วส่ง Alert เข้า Slack ตรงๆ — ที่ Scale
500 อุปกรณ์ วิธีนี้มีปัญหาใหญ่: **ถ้า Backbone Link ล่ม 1 เส้น อาจเกิด Anomaly พร้อมกัน 50-100
รายการในเวลาไม่กี่วินาที** (Interface Down หลายพอร์ตที่พึ่งพา Link นั้น, OSPF Neighbor Flap
หลายตัว, BGP Session Reset, QoS Queue Drop เพิ่มขึ้นที่ทุก Backup Path) — ถ้าส่ง Slack Message
100 ข้อความแยกกัน คือ **Alert Fatigue** ที่ทำให้ NOC จมกับ Noise จนพลาด Alert ที่สำคัญจริงๆ

### ย้ายจาก Script เดี่ยว ไปสู่ Prometheus Alertmanager

**Prometheus Alertmanager** คือ Component ที่ทำหน้าที่เฉพาะด้าน **จัดการ Alert หลังจากถูก Trigger
แล้ว** (Grouping, Deduplication, Silencing, Routing) แยกออกจาก Prometheus Server ที่ทำหน้าที่
Evaluate Rule เท่านั้น — Architecture แบบนี้คือมาตรฐานอุตสาหกรรมสำหรับ Alerting ที่ Scale

```
Telegraf (Step 793/795) → scrape 15s → Prometheus Server → evaluate Rule → fire
    → Alertmanager (Group / Deduplicate / Silence / Route)
        ├─► Slack (severity=warning)
        ├─► PagerDuty (severity=critical, Page On-call ทันที)
        └─► Webhook → CI/CD Rollback (severity=critical + domain=automation, Step 797)
```

### Prometheus Alert Rule — Interface Error Rate และ BGP/OSPF Flap

```yaml
# /etc/prometheus/rules/network-telemetry-alerts.yml

groups:
  - name: interface-health
    rules:
      - alert: InterfaceErrorRateHigh
        expr: rate(interface_stats_in_errors[5m]) > 1
        for: 2m
        labels:
          severity: warning
          domain: interface
        annotations:
          summary: "Interface {{ $labels.interface }} on {{ $labels.device }} มี Error Rate สูงผิดปกติ"
          description: "Error rate = {{ $value | printf \"%.2f\" }}/s เกิน Threshold 1/s ต่อเนื่อง 2 นาที"

  - name: bgp-health
    rules:
      - alert: BGPSessionFlapping
        expr: changes(bgp_session_state[10m]) > 3
        for: 0m
        labels:
          severity: critical
          domain: routing
        annotations:
          summary: "BGP Session กับ {{ $labels.peer }} Flap เกิน 3 ครั้งใน 10 นาที (ดู Part 74)"

  - name: deployment-correlation
    rules:
      - alert: ErrorSpikeAfterDeployment
        expr: |
          rate(interface_stats_in_errors[5m]) > 5
          and on(device) (time() - last_deployment_timestamp) < 600
        for: 1m
        labels:
          severity: critical
          domain: automation
        annotations:
          summary: "Error Spike บน {{ $labels.device }} ภายใน 10 นาทีหลัง Deploy — สงสัย Correlation กับ CI/CD (Part 79)"
```

### Alertmanager Config — Grouping/Deduplication/Routing เพื่อลด Alert Fatigue

```yaml
# /etc/alertmanager/alertmanager.yml

global:
  resolve_timeout: 5m

route:
  receiver: "default-slack"
  group_by: ["domain", "device"]     # จัดกลุ่ม Alert ตาม domain+device — Alert 50 ตัวจาก Link
                                       # เดียวกันรวมเป็น "1 Notification"
  group_wait: 30s                     # รอ Alert ที่เกี่ยวข้องมาถึงก่อนส่งรวมกัน
  group_interval: 5m                  # Alert ใหม่ในกลุ่มเดิม รอ 5 นาทีก่อนส่งรอบถัดไป
  repeat_interval: 4h                 # Alert เดิมที่ยัง Active ส่งซ้ำทุก 4 ชั่วโมง (ไม่ Spam)

  routes:
    - match:
        severity: critical
        domain: automation
      receiver: "cicd-webhook"        # Route ตรงไป Automated Remediation (Step 797)
      continue: true                   # และยังส่ง Alert แจ้ง Human ด้วย (ดู routes ถัดไป)

    - match:
        severity: critical
      receiver: "pagerduty-oncall"
      continue: false

    - match:
        severity: warning
      receiver: "default-slack"

receivers:
  - name: "default-slack"
    slack_configs:
      - api_url: "$SLACK_WEBHOOK_URL"
        channel: "#noc-alerts"
        title: '{{ .CommonLabels.domain }} Alert — {{ .CommonLabels.device }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ "\n" }}{{ end }}'

  - name: "pagerduty-oncall"
    pagerduty_configs:
      - service_key: "$PAGERDUTY_SERVICE_KEY"

  - name: "cicd-webhook"
    webhook_configs:
      - url: "http://cicd-gateway.internal:8080/webhook/anomaly-triggered-rollback"
        send_resolved: true

inhibit_rules:
  # ถ้า Device ทั้งตัว Down (Critical) ไม่ต้องแจ้ง Alert ย่อยของ Interface บนอุปกรณ์นั้นซ้ำ
  - source_match:
      alertname: "DeviceUnreachable"
    target_match_re:
      domain: "interface|routing"
    equal: ["device"]
```

### สรุปเทคนิคลด Alert Fatigue ที่ใช้ในตัวอย่างนี้

| เทคนิค | Config ที่ใช้ | ผลลัพธ์ |
|---|---|---|
| **Grouping** | `group_by: ["domain", "device"]` | Alert หลายตัวจากเหตุการณ์เดียวกัน (Link ล่ม 1 เส้น) รวมเป็น 1 Notification ต่อ Device/Domain แทนแยกกัน 50 ข้อความ |
| **Deduplication** | Alertmanager กรอง Alert ซ้ำจาก Prometheus Replica หลายตัวโดยธรรมชาติ | ไม่เกิด Notification ซ้ำเมื่อ Prometheus HA Pair ทั้งคู่ Evaluate Rule เดียวกันพร้อมกัน |
| **Inhibition** | `inhibit_rules` กด Alert ย่อยเมื่อ Alert แม่ (Device Down) Active อยู่แล้ว | ไม่แจ้ง "Interface Down" ซ้ำๆ ทุกพอร์ตเมื่อรู้อยู่แล้วว่าทั้งอุปกรณ์ล่ม |
| **Severity-based Routing** | `routes` แยก `critical` ไป PagerDuty (Page ทันที), `warning` ไป Slack (ดูตอนสะดวก) | คนที่ On-call ไม่ถูก Page กลางดึกเพราะ Warning ที่ไม่เร่งด่วน |
| **Escalation ผ่าน `repeat_interval`** | Critical Alert ที่ยังไม่ Resolve ส่งซ้ำทุก 4 ชั่วโมง | เตือนซ้ำถ้ายังไม่มีคนจัดการ แต่ไม่ Spam ทุกรอบ Evaluate (15s) |

---

## Step 797 — Telemetry-driven Automated Remediation: ปิด Loop กับ CI/CD

### แนวคิด: จาก "แจ้งเตือน" ไปสู่ "แก้ไขอัตโนมัติ"

[Part 54 Step 539](part-054-network-assurance-telemetry.md) เชื่อม Telemetry เข้ากับ Automation
แบบ **Diagnostic + Alert** (SSH ไปดึงข้อมูลเพิ่ม แล้วแจ้ง Human) — Step นี้ปิด Loop เต็มรูปแบบ
โดยเชื่อมเข้ากับ **CI/CD Pipeline ของ [Part 79](part-079-fullstack-automation-cicd.md)** ตรงๆ:
ถ้า Anomaly ที่ Detect ได้เกิดขึ้น **ภายในช่วงเวลาสั้นๆ หลัง Deployment ล่าสุด** ระบบจะสรุปได้ว่า
Deployment นั้นน่าจะเป็นสาเหตุ และ **Trigger Rollback ผ่าน CI/CD API โดยอัตโนมัติ** โดยไม่ต้องรอ
Human มา Approve ก่อน (สำหรับกรณีที่ความเสี่ยงของการรอสูงกว่าความเสี่ยงของการ Rollback ผิดพลาด)

### Architecture เต็มรูปแบบของ Closed-loop Remediation

```
CI/CD (Part 79) Deploy Config → บันทึก Deployment Timestamp เป็น Tag (Step 795)
    → Telemetry Pipeline (Step 793-795) feed → Prometheus Alert "ErrorSpikeAfterDeployment" fire
        → Alertmanager route: cicd-webhook → POST เข้า CI/CD Gateway (Part 79 API)
            1. ตรวจสอบว่า Deployment ล่าสุดตรงกับ Device ที่ Alert หรือไม่
            2. เรียก Rollback Pipeline (Part 79)
            3. แจ้งผลกลับเข้า Slack
```

### บันทึก Deployment Timestamp เป็น Tag ใน Time-series (เชื่อมกับ Step 795)

ขั้นแรกที่จำเป็นคือ CI/CD Pipeline ของ Part 79 ต้อง **เขียน Event "Deployment เกิดขึ้นเมื่อไหร่
บนอุปกรณ์ไหน"** เข้าไปใน Time-series Database เดียวกับ Telemetry เพื่อให้ Correlate กันได้:

```python
#!/usr/bin/env python3
# record_deployment_event.py — เรียกจาก CI/CD Pipeline (Part 79) ทันทีหลัง Deploy Config สำเร็จ
# บนอุปกรณ์ใดๆ เขียน Event ลง InfluxDB เพื่อให้ Prometheus Alert Rule ของ Step 796 Correlate ได้

from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS
import os
import sys
import time

INFLUX_URL = "http://influxdb-01:8086"
INFLUX_TOKEN = os.environ["INFLUX_TOKEN"]
INFLUX_ORG = "netops-lab"
INFLUX_BUCKET = "telemetry"


def record_deployment(device: str, pipeline_run_id: str, git_commit: str):
    client = InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG)
    write_api = client.write_api(write_options=SYNCHRONOUS)

    point = (
        Point("deployment_event")
        .tag("device", device)
        .tag("pipeline_run_id", pipeline_run_id)
        .tag("git_commit", git_commit[:8])
        .field("timestamp_epoch", int(time.time()))
        .field("status", 1)     # 1 = success — CI/CD Pipeline (Part 79) เรียกเฉพาะตอน Deploy สำเร็จ
    )
    write_api.write(bucket=INFLUX_BUCKET, record=point)
    print(f"[*] บันทึก Deployment Event: device={device}, commit={git_commit[:8]}")
    client.close()


if __name__ == "__main__":
    # เรียกจาก CI/CD Pipeline Step สุดท้ายหลัง Deploy — argument มาจาก Pipeline Environment
    record_deployment(
        device=sys.argv[1],
        pipeline_run_id=os.environ.get("CI_PIPELINE_ID", "unknown"),
        git_commit=os.environ.get("CI_COMMIT_SHA", "unknown"),
    )
```

### CI/CD Gateway Webhook Receiver — รับ Alert แล้วสั่ง Rollback

```python
#!/usr/bin/env python3
# cicd_gateway_webhook.py — Flask App รอรับ Webhook จาก Alertmanager (Step 796, receiver
# "cicd-webhook") ตรวจสอบว่า Anomaly สัมพันธ์กับ Deployment ล่าสุดจริงหรือไม่ ก่อนสั่ง Rollback
# ผ่าน Part 79 CI/CD Pipeline API — ทำหน้าที่เป็น "สมองกลาง" ของ Closed-loop Remediation

from flask import Flask, request, jsonify
import requests
import os
import time

app = Flask(__name__)

# Endpoint ของ CI/CD Pipeline จริง (Part 79) — สมมติเป็น REST API ที่ Trigger Rollback Job
CICD_ROLLBACK_API = "http://cicd-controller.internal:8443/api/v1/pipelines/rollback"
CICD_API_TOKEN = os.environ.get("CICD_API_TOKEN")

# กันการ Rollback ซ้ำถ้า Alert เดิมยิง Webhook มาซ้ำ (Alertmanager อาจ Retry)
recent_rollbacks = {}
ROLLBACK_COOLDOWN_SEC = 600


@app.route("/webhook/anomaly-triggered-rollback", methods=["POST"])
def handle_anomaly_webhook():
    payload = request.get_json()

    for alert in payload.get("alerts", []):
        if alert.get("status") != "firing":
            continue     # สนใจแค่ Alert ที่กำลัง Active — ไม่ต้อง Rollback ตอน Alert Resolve

        device = alert["labels"].get("device")
        alertname = alert["labels"].get("alertname")

        if alertname != "ErrorSpikeAfterDeployment":
            continue     # Webhook นี้รับหลาย Alert ได้ แต่ Auto-rollback ทำเฉพาะกรณีนี้เท่านั้น

        if not device:
            continue

        last_rollback = recent_rollbacks.get(device, 0)
        if time.time() - last_rollback < ROLLBACK_COOLDOWN_SEC:
            print(f"[!] {device}: อยู่ใน Cooldown — ข้าม Rollback ซ้ำ")
            continue

        print(f"[*] {device}: Anomaly สัมพันธ์กับ Deployment ล่าสุด — เริ่ม Auto-rollback")
        success = trigger_rollback(device)
        if success:
            recent_rollbacks[device] = time.time()

    return jsonify({"status": "processed"}), 200


def trigger_rollback(device: str) -> bool:
    """เรียก Part 79 CI/CD Pipeline API เพื่อ Rollback ไปยัง Config Version ก่อนหน้า"""
    try:
        response = requests.post(
            CICD_ROLLBACK_API,
            headers={"Authorization": f"Bearer {CICD_API_TOKEN}"},
            json={"device": device, "reason": "telemetry-anomaly-auto-rollback", "target": "previous-good-version"},
            timeout=10,
        )
        response.raise_for_status()
        print(f"[OK] Rollback Pipeline Trigger สำเร็จสำหรับ {device}: {response.json()}")
        return True
    except requests.exceptions.RequestException as exc:
        print(f"[!] Trigger Rollback ล้มเหลวสำหรับ {device}: {exc}")
        return False


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

### Sequence Diagram เต็มรูปแบบของ Closed-loop

| ลำดับ | เหตุการณ์ | ระบบที่รับผิดชอบ |
|---|---|---|
| 1 | Engineer Merge PR ที่เปลี่ยน QoS Policy บน CORE-SW1 | Git (Part 79) |
| 2 | CI/CD Pipeline Deploy Config ผ่าน Ansible, บันทึก `deployment_event` ลง InfluxDB | Part 79 + `record_deployment_event.py` |
| 3 | Config ผิดพลาด (เช่น Policy-map ผิด Class ทำให้ Traffic บาง Class ถูก Drop เกินจริง) | — |
| 4 | Interface Error/Drop Rate พุ่งขึ้นภายใน 3 นาทีหลัง Deploy | Telemetry Pipeline (Step 793-795) |
| 5 | Prometheus Rule `ErrorSpikeAfterDeployment` Match (เงื่อนไข: Error สูง **และ** Deploy ล่าสุด < 10 นาที) | Prometheus (Step 796) |
| 6 | Alertmanager Route Alert เข้า `cicd-webhook` **พร้อมกับ** `pagerduty-oncall` (แจ้ง Human ควบคู่) | Alertmanager (Step 796) |
| 7 | `cicd_gateway_webhook.py` ตรวจสอบเงื่อนไข แล้วเรียก Rollback API | CI/CD Gateway (Step 797) |
| 8 | Part 79 Pipeline รัน Rollback Job — Deploy Config Version ก่อนหน้ากลับไป | Part 79 |
| 9 | Error Rate กลับสู่ Baseline ภายในไม่กี่นาที — Alert `Resolved` อัตโนมัติ | Prometheus/Alertmanager |
| 10 | On-call Engineer ได้รับแจ้งทั้ง "มี Anomaly" และ "Rollback อัตโนมัติสำเร็จแล้ว" — Review ทีหลังว่าทำไม Config ผิด | Human |

> **หลักการความปลอดภัยที่สำคัญที่สุดของ Design นี้**: Auto-rollback ไม่ได้แทนที่ Human ทั้งหมด —
> `pagerduty-oncall` ยังถูก Route ควบคู่กันเสมอ (`continue: true` ใน Step 796) เพื่อให้ Human รู้ว่า
> เกิดอะไรขึ้นและ Rollback ทำไปแล้วหรือยัง และมี `ROLLBACK_COOLDOWN_SEC` กันไม่ให้ Rollback วน
> ซ้ำๆ ถ้า Root Cause จริงๆ ไม่ได้มาจาก Deployment (False Correlation) — Automation ที่ดีต้อง
> **Fail Safe และโปร่งใสต่อ Human เสมอ** ไม่ใช่ทำงานแบบ "Black Box" ที่ไม่มีใครรู้ว่าเกิดอะไรขึ้น

---

## Step 798 — Correlating Telemetry ข้าม Domain

### ปัญหา: Metric แยก Domain กัน แต่ Incident จริงข้าม Domain เสมอ

[Part 69 Mega Lab](part-069-mega-lab-1.md) สอนการ Troubleshoot แบบ **After-the-fact** — ใช้
`show` Command ไล่ทีละ Layer หลังจากปัญหาเกิดแล้ว Step นี้ทำสิ่งเดียวกันแต่ **Live** ผ่าน
Telemetry Dashboard เดียว: เมื่อ Incident เกิดขึ้นจริง ผลกระทบมักกระจายข้าม Domain พร้อมกัน —
**BGP Route Change** ([Part 74](part-074-advanced-bgp-wan-edge.md)), **QoS Drop** (Advanced QoS
เชิงลึกที่ต่อยอดจาก [Part 24](part-024-qos-fundamentals.md)/[Part 40](part-040-advanced-qos.md)),
และ **Interface Health** ล้วนเกี่ยวเนื่องกันในเหตุการณ์เดียว — Dashboard ที่ดีต้องแสดงทั้งสาม
Domain นี้ **บนหน้าจอเดียว พร้อม Timestamp เดียวกัน** เพื่อให้ Correlate ได้ทันทีโดยไม่ต้องเปิด
Dashboard แยกทีละตัว

### ตัวอย่าง Grafana Dashboard JSON — Panel เดียวรวม 3 Domain

```json
{
  "title": "Cross-domain Incident Correlation — WAN Edge Health",
  "panels": [
    { "title": "BGP Prefix Count (WAN-EDGE-1 ↔ ISP-RTR)", "type": "timeseries",
      "targets": [{ "query": "SELECT last(\"prefix_count\") FROM bgp_session WHERE peer='203.0.113.1' GROUP BY time(30s)" }] },
    { "title": "QoS Queue Drop Rate (WAN-EDGE-1 Egress)", "type": "timeseries",
      "targets": [{ "query": "SELECT rate(\"drop_packets\") FROM qos_queue_stats WHERE device='WAN-EDGE-1' GROUP BY time(30s), queue_class" }] },
    { "title": "Interface Utilization + Error (WAN Uplink)", "type": "timeseries",
      "targets": [{ "query": "SELECT mean(\"utilization_pct\") FROM interface_stats WHERE device='WAN-EDGE-1' AND interface='Gi0/0/0' GROUP BY time(30s)" }] }
  ],
  "annotations": { "list": [
    { "datasource": "influxdb-syslog", "name": "BGP/OSPF Events", "iconColor": "red" },
    { "datasource": "influxdb-deployment", "name": "CI/CD Deployment Events (Part 79)", "iconColor": "blue" }
  ] }
}
```

### ตัวอย่าง Timeline การ Correlate จริงข้าม 3 Domain

| Timestamp | Domain: Routing (BGP) | Domain: QoS | Domain: Interface |
|---|---|---|---|
| 09:14:00 | BGP Prefix Count ปกติ (450 prefix) | Queue Drop = 0 | Utilization 45% |
| 09:14:12 | **ISP Link Flap** — BGP Session ↔ ISP-RTR Reset | Queue Drop ยังปกติ | Utilization ลดลงกระทันหันเป็น 5% (Traffic หยุดผ่าน Path นี้) |
| 09:14:15 | Traffic Reroute ผ่าน WAN-EDGE-2 (Path สำรอง) | **Queue Drop พุ่งขึ้น** บน WAN-EDGE-2 (รับ Traffic เกิน Capacity ที่ Design ไว้สำหรับ Path สำรองอย่างเดียว) | Utilization บน WAN-EDGE-2 พุ่งเป็น 95% |
| 09:14:45 | BGP Session กลับ Established, Prefix Count กลับ 450 | Queue Drop ค่อยๆ ลดลง | Utilization กระจายกลับสองเส้นเท่าเดิม |

**ข้อสรุปจาก Correlation นี้ที่ไม่เห็นได้ถ้าดู Domain เดียว**: Root Cause คือ ISP Link Flap
(Routing Domain) แต่ **ผลกระทบจริงที่ User รู้สึก** คือ QoS Drop บน Path สำรอง (เพราะ Capacity
Planning ไม่ได้เผื่อไว้สำหรับกรณี Failover เต็มรูปแบบ) — ถ้าดู Dashboard แยก Domain Engineer
อาจเห็นแค่ "QoS Drop สูง" โดยไม่รู้ว่าต้นเหตุจริงคือ BGP Flap ที่เกิดก่อน 15 วินาที การ Correlate
ข้าม Domain แบบนี้คือสิ่งที่เปลี่ยน "แก้ปลายเหตุ" (เพิ่ม Queue) ไปเป็น "แก้ต้นเหตุ" (ตรวจสอบ ISP
Link และ Capacity Plan ของ Path สำรองให้เพียงพอ)

> **เชื่อมกับหลักการของ Part 69**: Mega Lab สอนให้ไล่ Layer ทีละขั้นหลังปัญหาเกิดแล้ว — Dashboard
> Correlation ของ Step นี้ทำสิ่งเดียวกัน **แบบ Real-time ก่อนที่ Engineer จะต้อง SSH เข้าอุปกรณ์เลย**
> เพราะทุก Metric ที่ต้องดูอยู่บนกราฟเดียวกันพร้อม Timestamp ที่ตรงกันแม่นยำ (ตรงตามหลักการ NTP
> Meta-metric ที่ [Part 54 Step 538](part-054-network-assurance-telemetry.md) เน้นย้ำไว้)

---

## Step 799 — Telemetry Security: ป้องกัน Pipeline เอง

### ทำไม Telemetry Pipeline เป็นเป้าหมายที่มีค่าสำหรับผู้โจมตี

Telemetry Data ที่ไหลผ่าน Pipeline นี้ **เผยสภาพเครือข่ายทั้งหมด**: Topology (จาก Interface/BGP
Neighbor), Traffic Pattern (จาก Bandwidth Utilization), และแม้แต่ **จุดอ่อน** (Error Rate สูง
บนจุดไหน = จุดที่ Attacker น่าจะโจมตีให้ล่มง่ายที่สุด) — ถ้า gRPC Stream ระหว่างอุปกรณ์กับ
Collector ถูก Intercept แบบ Plaintext ผู้โจมตีจะได้ **แผนที่เครือข่ายแบบ Real-time** โดยไม่ต้อง
Scan อะไรเลย นอกจากนี้ Pipeline เดียวกันนี้ (ตาม Step 797) ยังมีสิทธิ์ **Trigger Rollback บน
Production Config โดยตรง** — ถ้า Attacker ปลอม Alert เข้า Webhook ได้ ก็สามารถสั่ง Rollback
เพื่อสร้าง Denial of Service ได้ทันที

### สามชั้นที่ต้อง Secure

**ชั้นที่ 1**: Device → Collector (gRPC/gNMI) — **ชั้นที่ 2**: Collector → Kafka → Storage
(ภายใน Data Center) — **ชั้นที่ 3**: Alertmanager → CI/CD Webhook (Step 797, สิทธิ์สูงสุด
เพราะสั่งเปลี่ยน Production Config ได้ตรง)

### ชั้นที่ 1: TLS + Mutual Authentication บน gRPC Dial-out

[Part 54 Step 534](part-054-network-assurance-telemetry.md) ใช้ `protocol grpc-tcp` (Plaintext)
สำหรับ Lab — Production ต้องเปลี่ยนเป็น **`grpc-tls`** พร้อม Certificate ที่ออกจาก CA เดียวกัน
กับที่ [Part 78 Step 778](part-078-security-integration-ise-trustsec-ftd-umbrella.md) ออกแบบไว้
สำหรับ Unified PKI ของทั้งองค์กร

```
CORE-SW1(config)# telemetry ietf destination-group DG-NOC-COLLECTOR
CORE-SW1(config-telemetry-dest-grp)# destination-id 1
CORE-SW1(config-telemetry-dest-grp)# destination ip address 10.10.99.50 port 57500 protocol grpc-tls encoding encode-kvgpb
CORE-SW1(config-telemetry-dest-grp)# destination profile telemetry-tls-profile
CORE-SW1(config-telemetry-dest-grp)# exit

! ผูก Trustpoint ที่ออกจาก Enterprise CA (เดียวกับ Part 78 Unified PKI)
CORE-SW1(config)# telemetry profile telemetry-tls-profile
CORE-SW1(config-telemetry-profile)# trustpoint TP-TELEMETRY-CLIENT
```

Telegraf ฝั่ง Collector ต้องเปิด TLS รับด้วย (ตาม Comment ที่วางไว้ใน Step 793):

```toml
[[inputs.cisco_telemetry_mdt]]
  transport = "grpc"
  service_address = ":57500"
  tls_cert = "/etc/telegraf/certs/collector.pem"
  tls_key  = "/etc/telegraf/certs/collector.key"
  tls_allowed_cacerts = ["/etc/telegraf/certs/enterprise-ca.pem"]   # ยอมรับเฉพาะ Client Cert จาก CA องค์กร (mTLS)
```

| องค์ประกอบ | หน้าที่ |
|---|---|
| `protocol grpc-tls` | Encrypt Data-in-transit ระหว่างอุปกรณ์กับ Collector — ป้องกัน Passive Eavesdropping |
| `trustpoint` บนอุปกรณ์ | อุปกรณ์ใช้ Certificate นี้พิสูจน์ตัวเองต่อ Collector (Client Certificate) |
| `tls_allowed_cacerts` บน Telegraf | Collector ยอมรับเฉพาะ Connection จากอุปกรณ์ที่มี Certificate ออกจาก CA ที่ Trust เท่านั้น — **Mutual TLS (mTLS)** ป้องกันอุปกรณ์ปลอมมา Dial-out เข้า Pipeline |

### ชั้นที่ 2: Kafka Security — SASL/TLS ระหว่าง Broker และ Client

```properties
# server.properties (Kafka Broker) — ส่วนที่เกี่ยวกับ Security
listeners=SASL_SSL://kafka-01:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
ssl.keystore.location=/etc/kafka/certs/kafka.keystore.jks
ssl.truststore.location=/etc/kafka/certs/kafka.truststore.jks
```

```toml
# telegraf.conf (Step 793) — ส่วน outputs.kafka เพิ่ม Security
[[outputs.kafka]]
  brokers = ["kafka-01:9093"]
  topic = "telemetry.raw"
  sasl_username = "$KAFKA_SASL_USER"
  sasl_password = "$KAFKA_SASL_PASSWORD"
  sasl_mechanism = "SCRAM-SHA-512"
  enable_tls = true
  tls_ca = "/etc/telegraf/certs/kafka-ca.pem"
```

### ชั้นที่ 3: Webhook Authentication — ป้องกันการปลอม Alert เพื่อสั่ง Rollback

จุดที่มีความเสี่ยงสูงสุดในทั้ง Pipeline คือ Webhook ของ Step 797 — เพราะมันมีสิทธิ์เปลี่ยน
Production Config ต้องมี **Shared Secret + Signature Verification** ไม่ใช่รับ POST แบบเปิด:

```python
# เพิ่มใน cicd_gateway_webhook.py (Step 797) — ตรวจ HMAC Signature ก่อนประมวลผล Webhook

import hashlib
import hmac

ALERTMANAGER_WEBHOOK_SECRET = os.environ["ALERTMANAGER_WEBHOOK_SECRET"]


def verify_signature(request) -> bool:
    signature_header = request.headers.get("X-Signature-256", "")
    computed = hmac.new(
        ALERTMANAGER_WEBHOOK_SECRET.encode(), request.get_data(), hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={computed}", signature_header)


@app.route("/webhook/anomaly-triggered-rollback", methods=["POST"])
def handle_anomaly_webhook():
    if not verify_signature(request):
        return jsonify({"error": "invalid signature"}), 403
    # ... (ส่วนที่เหลือเหมือน Step 797)
```

> Alertmanager เองไม่ Support HMAC Signing บน Webhook โดยตรงในทุกเวอร์ชัน — Production จริงมัก
> วาง **Reverse Proxy หรือ API Gateway ที่มี mTLS/Network ACL จำกัดเฉพาะ Source IP ของ
> Alertmanager** ไว้หน้า Webhook Endpoint เป็นชั้นป้องกันเสริม (Defense in Depth เดียวกับหลักการ
> ของ [Part 78](part-078-security-integration-ise-trustsec-ftd-umbrella.md))

### ตารางสรุป Security Checklist ของ Telemetry Pipeline

| ชั้น | ต้องมี | อ้างอิง |
|---|---|---|
| Device → Collector | `grpc-tls`/`gnmi` + mTLS Certificate จาก Enterprise CA | Part 78 Step 778 (Unified PKI) |
| Collector → Kafka | SASL_SSL + Encryption at rest บน Broker Disk | — |
| Kafka → Storage | TLS ระหว่าง Consumer กับ Broker (เดียวกับข้างบน) | — |
| Alertmanager → CI/CD Webhook | HMAC Signature Verification + Network ACL/mTLS ที่ Gateway | Part 79 (CI/CD Security Model) |
| ทั่วทั้ง Pipeline | Least Privilege — Service Account ของ Telegraf/Kafka Consumer ไม่ควรมีสิทธิ์เขียน Config อุปกรณ์เลย มีแค่สิทธิ์ Read Telemetry | Part 22 (AAA/Device Hardening) |

---

## Step 800 — Lab เต็มรูปแบบ: Production Telemetry Pipeline

### เป้าหมายของ Lab

รวมทุก Step ของ Part นี้เข้าเป็น Pipeline เดียวที่ทำงานจริงบน Topology หลักของหลักสูตร:
**CORE-SW1/2 + DIST-SW1-4** Stream MDT/gNMI → **Telegraf Collector Tier** → **Kafka** →
**InfluxDB (พร้อม Retention Policy)** → **Grafana Dashboard** → **Prometheus Alertmanager**
→ **Webhook Auto-rollback เชื่อมกับ Part 79**

### Topology ของ Pipeline เต็มรูปแบบ

```
Device Layer (ตาม 00-ip-address-plan.md): CORE-SW1(1.1.1.1) CORE-SW2(1.1.1.2) DIST-SW1-4(1.1.1.11-14)
  MDT Dial-out (grpc-tls, port 57500) + gNMI (port 57501, DIST-SW1 เท่านั้น — ทดสอบ Multi-protocol)
      │ gRPC/gNMI over TLS
      ▼
Collector Tier — telegraf-collector (10.10.99.50): inputs.cisco_telemetry_mdt + cisco_telemetry_gnmi
      │ outputs.kafka topic=telemetry.raw (Step 793)
      ▼
Kafka Cluster (kafka-01/02/03): Topic telemetry.raw, Replication=3, Retention 48h (Step 791)
      ├─ consumer_group=influxdb-writer   → telegraf-storage → InfluxDB (telemetry/_5m/_1h) + prometheus_client :9273
      └─ consumer_group=anomaly-detector  → anomaly_stream_processor.py (Step 797) → deployment_event correlation
                                                │
      ┌─────────────────────────────────────────┘
      ▼
Grafana Dashboard (Step 798)   Prometheus (scrape :9273) → fire → Alertmanager (Step 796, Group/Route)
                                                                        → POST → CI/CD Webhook (Step 797, Part 79 API)
```

### ขั้นตอนที่ 1 — Deploy MDT + gNMI Config ให้ครบ 6 อุปกรณ์

ใช้ Template จาก [Part 54 Step 540](part-054-network-assurance-telemetry.md) เป็นฐาน เปลี่ยน
`protocol grpc-tcp` เป็น `grpc-tls` (Step 799) และเพิ่ม gNMI Config บน DIST-SW1:

```
! เพิ่มบน DIST-SW1 เท่านั้น (นอกจาก MDT Template เดิม) — เปิด gNMI Server
DIST-SW1(config)# gnmi-yang
DIST-SW1(config)# gnmi-yang server
DIST-SW1(config)# gnmi-yang port 57501
```

Deploy ทั้ง 6 อุปกรณ์ผ่าน Ansible Playbook เดียวกับที่ [Part 79](part-079-fullstack-automation-cicd.md)
ใช้เป็น Pipeline หลัก (Golden Config Template ที่รวม Telemetry Block ไว้ในตัว ตามที่ Step 792
อธิบายเรื่องข้อดีของ Dial-out กับ CI/CD):

```yaml
# playbook: deploy-telemetry-config.yml
- name: Deploy MDT Telemetry Config (grpc-tls) ให้ทุกอุปกรณ์ Core/Distribution
  hosts: core_dist_switches
  gather_facts: no
  tasks:
    - name: Push Telemetry Config Block
      cisco.ios.ios_config:
        src: "templates/telemetry-mdt-tls.j2"
      vars:
        collector_ip: "10.10.99.50"
        collector_port: 57500
        source_address: "{{ loopback0_ip }}"   # จาก group_vars ตาม 00-ip-address-plan.md
```

### ขั้นตอนที่ 2 — Deploy Collector Tier + Kafka (Docker Compose สำหรับ Lab)

```yaml
# docker-compose.yml — Lab Environment สำหรับ Part 80 Step 800
version: "3.8"
services:
  kafka-01:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-01:9092
    ports: ["9092:9092"]

  telegraf-collector:
    image: telegraf:1.30
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - ./certs:/etc/telegraf/certs:ro
    ports: ["57500:57500", "57501:57501"]
    depends_on: [kafka-01]

  telegraf-storage:
    image: telegraf:1.30
    volumes:
      - ./telegraf-storage-consumer.conf:/etc/telegraf/telegraf.conf:ro
    ports: ["9273:9273"]
    depends_on: [kafka-01, influxdb]

  influxdb:
    image: influxdb:2.7
    ports: ["8086:8086"]
    volumes: ["influxdb-data:/var/lib/influxdb2"]

  prometheus:
    image: prom/prometheus:v2.52.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
    ports: ["9090:9090"]

  alertmanager:
    image: prom/alertmanager:v0.27.0
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports: ["9093:9093"]

  grafana:
    image: grafana/grafana:11.0.0
    ports: ["3000:3000"]
    depends_on: [influxdb, prometheus]

  cicd-gateway:
    build: ./cicd-gateway
    environment:
      CICD_API_TOKEN: "${CICD_API_TOKEN}"
      ALERTMANAGER_WEBHOOK_SECRET: "${WEBHOOK_SECRET}"
    ports: ["8080:8080"]

volumes:
  influxdb-data:
```

### ขั้นตอนที่ 3 — Verify: ยืนยันว่าทุก Component เชื่อมกันสำเร็จ

```bash
$ docker compose ps
NAME                  STATUS
kafka-01              Up 5 minutes
telegraf-collector    Up 5 minutes
telegraf-storage      Up 5 minutes
influxdb              Up 5 minutes
prometheus            Up 5 minutes
alertmanager          Up 5 minutes
grafana               Up 5 minutes
cicd-gateway          Up 5 minutes

$ docker compose logs telegraf-collector | grep -i "accepted"
2026-09-26T15:00:03Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.0.1  (CORE-SW1)
2026-09-26T15:00:04Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.0.2  (CORE-SW2)
2026-09-26T15:00:04Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.10.2 (DIST-SW1)
2026-09-26T15:00:05Z I! [inputs.cisco_telemetry_gnmi] gNMI Subscribe established from 10.255.10.2:57501  (DIST-SW1 gNMI)
2026-09-26T15:00:05Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.20.2 (DIST-SW2)
... (DIST-SW3, DIST-SW4 เชื่อมสำเร็จเช่นเดียวกัน รวมครบ 6/6 อุปกรณ์)

$ curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'
"up"

$ curl -s http://localhost:9093/api/v2/status | jq '.cluster.status'
"ready"
```

### ขั้นตอนที่ 4 — จำลอง End-to-end Scenario: Bad Deployment → Auto-rollback

จำลองเหตุการณ์: CI/CD Deploy QoS Policy ผิดพลาดบน DIST-SW2 → เห็น Error Spike ผ่าน Telemetry
→ Alert → Webhook → Auto-rollback ครบทุกขั้นภายในไม่กี่นาที

```bash
# 1. จำลอง CI/CD Deploy (บันทึก deployment_event ตาม Step 797)
$ python3 record_deployment_event.py DIST-SW2 --pipeline-run-id run-4821 --git-commit a1b2c3d4e5

# 2. จำลอง Error Spike (ในสถานการณ์จริงมาจาก Telemetry ที่ Stream เข้ามาเองหลัง Config ผิด)
$ curl -X POST http://localhost:8428/api/v1/import/prometheus \
    -d 'interface_stats_in_errors{device="DIST-SW2",interface="Te1/1/1"} 850'

# 3. รอ Prometheus Evaluate Rule (ทุก 15s ตาม prometheus.yml) — ดู Alert ที่ Fire
$ curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname=="ErrorSpikeAfterDeployment")'
{
  "labels": { "alertname": "ErrorSpikeAfterDeployment", "device": "DIST-SW2", "severity": "critical" },
  "state": "firing"
}

# 4. ดู Log ของ CI/CD Gateway — ยืนยันว่า Webhook ถูกเรียกและ Rollback ถูก Trigger
$ docker compose logs cicd-gateway
[*] DIST-SW2: Anomaly สัมพันธ์กับ Deployment ล่าสุด — เริ่ม Auto-rollback
[OK] Rollback Pipeline Trigger สำเร็จสำหรับ DIST-SW2: {'run_id': 'rollback-9931', 'status': 'queued'}

# 5. ยืนยันว่า Part 79 Pipeline รับ Rollback Job และ Deploy Config เดิมกลับไปสำเร็จ
$ curl -s http://cicd-controller.internal:8443/api/v1/pipelines/rollback-9931 | jq '.status'
"completed"
```

### สิ่งที่ Lab นี้พิสูจน์เทียบกับ Lab ของ Part 54

| ประเด็น | Part 54 Step 540 (Lab เดิม) | Part 80 Step 800 (Lab นี้) |
|---|---|---|
| จำนวน Collector | 1 ตัว รับตรงจากทุกอุปกรณ์ | Collector Tier + Kafka แยกจาก Storage Tier (Decoupled ตาม Step 791) |
| Transport | MDT อย่างเดียว | MDT + gNMI พร้อมกัน (Multi-protocol ตาม Step 794) |
| Security | Plaintext (`grpc-tcp`, Lab เท่านั้น) | `grpc-tls` + mTLS + Kafka SASL_SSL (Step 799) |
| Storage | InfluxDB Raw เท่านั้น | InfluxDB + Downsampling/Retention Tier (Step 795) |
| Alerting | Python Script คำนวณ Sigma เอง | Prometheus Alertmanager (Grouping/Dedup/Routing ตาม Step 796) |
| Remediation | Diagnostic + Slack Alert (แจ้ง Human) | **Auto-rollback ผ่าน CI/CD Webhook** (Step 797) พร้อมแจ้ง Human ควบคู่ |
| Correlation | UDLD/OSPF Timeline เดียว | Cross-domain (BGP/QoS/Interface) บน Dashboard เดียว (Step 798) |

> **นี่คือความต่างระหว่าง "เข้าใจ MDT" กับ "ออกแบบ Production Telemetry Pipeline"** — ทุกองค์ประกอบ
> ที่เพิ่มเข้ามาจาก Part 54 (Kafka, TLS, Alertmanager, Auto-rollback, Cross-domain Correlation)
> ไม่ใช่ความซับซ้อนที่เพิ่มเข้ามาแบบไม่มีเหตุผล — แต่ละอย่างแก้ปัญหาที่เกิดขึ้นจริงเมื่อสเกลจาก
> Lab 6 อุปกรณ์ ไปสู่ Enterprise หลักร้อยหลักพันอุปกรณ์ ซึ่งคือระดับความเข้าใจที่ CCIE Lab Exam
> คาดหวังจากผู้สอบในหัวข้อ Assurance และ Automation

---

## แบบฝึกหัดทวนความเข้าใจ Part 80

1. อธิบายว่า Kafka แก้ปัญหา "Data Loss ตอน Storage ช้าชั่วขณะ" ได้อย่างไร และทำไมการ Decouple
   Collector Tier กับ Storage Tier จึงสำคัญที่ Enterprise Scale
2. เพราะเหตุใด Dial-out จึงเหมาะกับ Production Monitoring มากกว่า Dial-in เมื่ออุปกรณ์มีจำนวนมาก
   (หลักร้อยถึงหลักพันตัว) — อธิบายในมุมของ "Collector ต้องรู้จักอุปกรณ์กี่ตัว"
3. gNMI ต่างจาก MDT อย่างไรในระดับ Protocol/Encoding และทำไม gNMI จึงเหมาะกับ Multi-vendor
   Environment มากกว่า
4. ในการออกแบบ Schema InfluxDB ทำไมการเลือก `interface` เป็น Tag แทน Field จึงสำคัญ และการ
   Downsample ด้วย `mean` อย่างเดียวมีข้อเสียอะไรสำหรับ Metric ประเภท Error Rate
5. ในสถาปัตยกรรม Telemetry-driven Automated Remediation (Step 797) เพราะเหตุใดการยังคง Route
   Alert ไปที่ Human (PagerDuty) ควบคู่กับการ Auto-rollback จึงเป็นหลักการออกแบบที่ถูกต้อง

**เฉลย:**

1. Kafka ทำหน้าที่เป็น Durable Message Queue ระหว่าง Collector กับ Storage — เมื่อ Storage
   (InfluxDB) ช้าลงชั่วขณะ (เช่น Compaction) Message ยังถูกเก็บไว้ใน Kafka Topic (Replicated,
   Durable) รอ Consumer มาอ่านต่อได้ โดยไม่กระทบการรับ gRPC Connection จากอุปกรณ์ฝั่ง Collector
   เลย การ Decouple ทำให้ทั้งสองฝั่ง Fail และ Scale ได้อย่างอิสระจากกัน ป้องกัน Single Point of
   Failure ของ Pipeline ทั้งระบบ
2. เพราะ Dial-out ทำให้ Collector ไม่ต้องรู้จัก Address ของอุปกรณ์แม้แต่ตัวเดียว (Collector แค่
   "รอรับที่ Port เดียว") ในขณะที่ Dial-in ต้องให้ Collector มี Inventory ครบทุกอุปกรณ์และ
   Sync ตลอดเวลาเมื่อมีอุปกรณ์ใหม่/IP เปลี่ยน — ที่สเกลหลักร้อยหลักพันตัว การดูแล Inventory
   แบบนั้นเป็นภาระ Operational ที่สูงมาก ในขณะที่ Dial-out เพิ่มอุปกรณ์ใหม่ได้โดย Collector
   Config ไม่ต้องเปลี่ยนอะไรเลย
3. MDT เป็น Protocol เฉพาะของ Cisco ใช้ RPC `MdtDialout` และ Encoding `encode-kvgpb` แบบ
   Cisco-specific ในขณะที่ gNMI เป็นมาตรฐานที่ OpenConfig ผลักดันให้ Vendor ต่างๆ Implement
   ตาม Spec เดียวกัน (RPC `Subscribe`/`Get`/`Set`, Encoding JSON_IETF/Protobuf มาตรฐาน) ทำให้
   Client/Collector Code เดียวกันใช้กับหลาย Vendor ได้ — เหมาะกับ Multi-vendor Environment
   ที่ต้องการ Tooling เดียวไม่ต้องเขียนแยกตาม Vendor
4. `interface` ถูก Filter/Group By บ่อยมากใน Query ทั่วไป — ถ้าเป็น Tag จะถูก Index ทำให้ Query
   เร็ว ถ้าเป็น Field ต้อง Scan ทุก Row ทำให้ช้ามากที่ Volume สูง ส่วนการ Downsample ด้วย `mean`
   อย่างเดียวจะ "กลบ" Error Spike สั้นๆ ที่สำคัญต่อการ Detect Anomaly เพราะค่าเฉลี่ยของ Spike
   ปนกับค่าปกติจะดูเหมือนไม่มีอะไรผิดปกติ — ต้องเก็บ `max` แยกไว้อีก Field สำหรับ Metric ประเภทนี้
5. เพราะ Automation ที่ทำงานแบบ "Black Box" โดยไม่มีใครรู้ว่าเกิดอะไรขึ้นเป็นความเสี่ยงสูง —
   Correlation ระหว่าง Anomaly กับ Deployment อาจผิดพลาดได้ (False Correlation) หรือ Rollback
   อาจไม่ใช่การแก้ปัญหาที่ถูกต้องเสมอไป การแจ้ง Human ควบคู่กันทำให้มีคนตรวจสอบผลของ Automation
   ทันที และสามารถแก้ไขเพิ่มเติมได้ถ้า Auto-rollback ไม่พอหรือ Root Cause จริงไม่ได้มาจาก
   Deployment — หลักการ Fail Safe และโปร่งใสต่อ Human คือสิ่งจำเป็นสำหรับ Automation ที่มีสิทธิ์
   เปลี่ยน Production Config โดยตรง

---

## สรุป Part 80

Part นี้ยกระดับ Model-Driven Telemetry จาก Lab 6 อุปกรณ์ของ [Part 54](part-054-network-assurance-telemetry.md)
ไปสู่ Production Pipeline ระดับ Enterprise เต็มรูปแบบ: แก้ปัญหา Data Volume ด้วย **Kafka** เป็น
Buffering/Distribution Layer ที่ Decouple Collector ออกจาก Storage (Step 791), ยืนยันว่า
**Dial-out ยังเป็น Default ที่ถูกต้องที่สุดที่ Scale** เพราะ Collector ไม่ต้องดูแล Inventory เลย
(Step 792), สร้าง `telegraf.conf` Production-grade ที่เชื่อม MDT/gNMI เข้ากับ Kafka (Step 793),
เจาะลึก **gNMI** เป็นทางเลือก Vendor-neutral สำหรับอนาคต Multi-vendor (Step 794), ออกแบบ
Time-series Schema และ Retention Policy ที่ไม่ทำให้ Storage ระเบิด (Step 795), เปลี่ยน Alerting
จาก Script เดี่ยวไปเป็น **Prometheus Alertmanager** ที่ลด Alert Fatigue ด้วย Grouping/
Deduplication/Routing (Step 796), ปิด Loop เต็มรูปแบบเชื่อมกับ **CI/CD Pipeline ของ
[Part 79](part-079-fullstack-automation-cicd.md)** ให้ Anomaly Trigger Auto-rollback ได้จริง
(Step 797), พิสูจน์คุณค่าของ Dashboard ที่ Correlate ข้าม BGP/QoS/Interface Domain (Step 798),
ปิด Pipeline ด้วย TLS/mTLS/Webhook Authentication ให้ปลอดภัยครบทุกชั้น (Step 799), และรวมทุกอย่าง
เป็น Lab เดียวที่ End-to-end จริงตั้งแต่ Device ไปจน Auto-rollback (Step 800)

หลักการที่สำคัญที่สุดของ Part นี้คือ: **Telemetry ที่ Scale ไม่ใช่แค่ "MDT Config เดิมคูณจำนวน
อุปกรณ์" แต่ต้องออกแบบ Pipeline ใหม่ทั้งระบบให้ Decoupled, Secure, และปิด Loop กลับไปยัง
Automation ได้จริง** — ทักษะนี้คือรากฐานเดียวกับที่ Part 91 (Advanced Troubleshooting Methodology)
จะใช้ต่อ และเป็น 1 ใน 4 เสาหลักของ CCIE Enterprise Infrastructure Lab Exam ตามที่
[Part 71](part-071-ccie-lab-blueprint-overview.md) วางแผนไว้

✅ **พร้อมสำหรับ Part 81**: เราจะย้ายโฟกัสจาก Assurance/Automation ไปสู่ **High Availability**
ระดับ Hardware/Control-plane — **NSF/SSO (Non-stop Forwarding/Stateful Switchover)** และ
**ISSU (In-Service Software Upgrade)** ที่ทำให้ Core/Distribution Switch อัปเกรด Software หรือ
Failover Supervisor ได้โดย Traffic ไม่สะดุด ซึ่ง Telemetry Pipeline ที่สร้างใน Part นี้จะเป็น
เครื่องมือสำคัญในการยืนยันว่า Failover เหล่านั้น "Non-stop" จริงตามชื่อหรือไม่

**ไปต่อ:** [Part 81 — High Availability: NSF/SSO/ISSU →](part-081-high-availability-nsf-sso-issu.md)
