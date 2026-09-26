# Part 54 — Network Assurance & Telemetry
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 531–540 จาก 1000**

> ต่อจาก [Part 53 — DMVPN Advanced](part-053-dmvpn-advanced.md) ที่เราปิดท้ายฝั่ง WAN/Overlay
> Technology ไปแล้ว Part นี้พาเรากลับมาที่แกนกลางของ CCNP ENCOR อีกครั้ง — **Network Assurance**
> [Part 17](part-017-ntp-syslog-snmp.md) สอน SNMP Polling/Trap ไว้เป็นพื้นฐาน และ
> [Part 49 Step 489](part-049-netconf-restconf-yang-deep-dive.md) ได้ "แอบดู" **Model-Driven
> Telemetry (MDT)** ไว้สั้นๆ พร้อมสัญญาว่าจะเจาะลึกในภายหลัง — Part 54 นี้คือ Part ที่สัญญานั้น
> ครบกำหนด เราจะ (1) เจาะลึก MDT เต็มรูปแบบทั้ง Dial-in/Dial-out, gRPC Transport, Sensor-Path,
> (2) ต่อ Pipeline ไปยัง Collector จริง (TIG Stack หรือ Cisco Crosswork), (3) เขียน Python
> gRPC Receiver และ Anomaly Correlation Script ที่เชื่อมกับทักษะ Automation จาก
> [Part 47](part-047-automation-python-advanced.md) และ (4) ปิดท้ายด้วย Lab เต็มรูปแบบที่ Stream
> ข้อมูลจากทุกอุปกรณ์ Core/Distribution พร้อมจำลองเหตุการณ์ Interface Flap ที่เชื่อมโยงกับ UDLD/OSPF
> จาก [Part 33](part-033-advanced-stp-campus-design.md) — นี่คือ Part ที่เปลี่ยนเราจากคนที่แค่
> "config อุปกรณ์ได้" ไปเป็นคนที่ "รู้ว่าเครือข่ายกำลังเป็นอย่างไรอยู่ตลอดเวลา แบบ real-time"

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 531 | ข้อจำกัดของ SNMP Polling (ทวนจาก Part 17) — ทำไมต้องมี Streaming Telemetry |
| 532 | Model-Driven Telemetry (MDT) พื้นฐาน — YANG Model เดิม, Dial-in vs Dial-out |
| 533 | Telemetry Transport — gRPC, Protobuf, เทียบกับ Transport รุ่นเก่า |
| 534 | Configuring MDT บน IOS-XE — sensor-path, destination-group, Worked Config เต็มรูปแบบ |
| 535 | Telemetry Collector & Pipeline — TIG Stack vs Cisco Crosswork/NAE |
| 536 | Python gRPC Telemetry Receiver — โครงสร้างจริง (Simplified) |
| 537 | Network Assurance — Baseline/Anomaly Detection, Proactive vs Reactive |
| 538 | ออกแบบ Dashboard — Top 10 Metrics ที่ต้อง Monitor สำหรับ Lab นี้ |
| 539 | เชื่อม Telemetry กับ Automation (Part 47) — Anomaly-Triggered Remediation |
| 540 | Lab เต็มรูปแบบ: MDT ทุกอุปกรณ์ Core/Distribution + จำลอง Interface Flap Correlation |

---

## Step 531 — ข้อจำกัดของ SNMP Polling: ทำไมต้องมี Streaming Telemetry

### ทวนความเข้าใจจาก Part 17

[Part 17 Step 167-169](part-017-ntp-syslog-snmp.md) สอน **SNMP** ไว้สองรูปแบบ: **Polling**
(NMS ยิง GET Request ไปถามอุปกรณ์เป็นระยะๆ) และ **Trap** (อุปกรณ์ส่งแจ้งเตือนเองเมื่อมี Event
สำคัญเกิดขึ้น) ทั้งสองรูปแบบใช้งานได้จริงมาหลายสิบปี แต่เมื่อ Enterprise Network ขยายขนาดขึ้น
(อุปกรณ์เป็นพัน, ต้องการ Visibility ระดับวินาที) ข้อจำกัดเชิงโครงสร้างของ SNMP ก็เริ่มปรากฏชัด

### ข้อจำกัดที่ 1 — Pull-Based คือ "เดา" ความถี่ที่เหมาะสมไม่ได้

SNMP Polling ต้องกำหนด **Polling Interval** ตายตัวไว้ล่วงหน้า (เช่นทุก 60 วินาที, ทุก 5 นาที)
ปัญหาคือ Engineer ไม่มีทางรู้ล่วงหน้าว่าเหตุการณ์ผิดปกติจะเกิดขึ้นเมื่อไหร่ และเหตุการณ์หลายอย่าง
เป็น**เหตุการณ์ชั่วครู่ (Transient Event)** ที่เกิดขึ้นและหายไปเร็วกว่าช่วง Polling Interval มาก

```
เวลา (วินาที) →   0        10       20       30       40       50       60
                   │        │        │        │        │        │        │
SNMP Poll (60s):   ▼────────────────────────────────────────────────────▼
                   Poll #1                                              Poll #2
                   ifInErrors = 0                                       ifInErrors = 0
                                    ┌──┐
Interface Errors:                  │▓▓│  <- Micro-burst error spike (เกิดวินาทีที่ 22-24
                                    └──┘     แล้ว counter reset หรือ error หยุดก่อน Poll #2 มาถึง)

ผลลัพธ์: NMS เห็นค่า 0 ทั้งสองครั้งที่ Poll — Micro-burst ที่เกิดขึ้นจริงถูก "มองไม่เห็นเลย"
```

นี่คือปัญหาคลาสสิกของ Pull-Based Monitoring: **ถ้าเหตุการณ์เกิดและหายไปเร็วกว่าคาบเวลา Poll
เหตุการณ์นั้นจะไม่ถูกบันทึกเลย** แม้ว่ามันอาจเป็นสาเหตุของปัญหา Application ที่ผู้ใช้ร้องเรียนมาก็ตาม

### ข้อจำกัดที่ 2 — Scalability: Polling หลาย OID จากหลายอุปกรณ์พร้อมกัน

ลองคำนวณ Load จริงของการ Poll แบบดั้งเดิม:

| ตัวแปร | ค่าตัวอย่าง |
|---|---|
| จำนวนอุปกรณ์ที่ต้อง Monitor | 500 อุปกรณ์ (Enterprise ขนาดกลาง) |
| จำนวน OID ต่ออุปกรณ์ (CPU, Memory, ทุก Interface ×4 counter, Temperature, ...) | ~200 OID |
| Polling Interval | 60 วินาที |
| **จำนวน SNMP GET Request ต่อวินาที** | 500 × 200 ÷ 60 ≈ **1,666 request/วินาที** |

ทุก Request คือ 1 UDP Round-trip ที่ต้องสร้าง PDU, encode ASN.1/BER, ส่งไปที่ Agent, ให้ Agent
Process (มักใช้ CPU Cycle บน Control Plane ของอุปกรณ์เอง), แล้ว Encode กลับมา — ที่ Load ระดับนี้
ทั้ง NMS (ฝั่ง Poll) และ Agent (ฝั่งอุปกรณ์) ต้องแบก Overhead จำนวนมาก และถ้าต้องการความถี่สูงขึ้น
(เช่น Poll ทุก 5 วินาทีเพื่อจับ Micro-burst ให้ได้) Load จะเพิ่มขึ้นแบบเป็นเส้นตรงทันที (12x)

### ข้อจำกัดที่ 3 — Data Model ไม่มี Schema ตายตัว (ปัญหาเดียวกับที่ Part 49 พูดถึง CLI)

MIB/OID ของ SNMP เป็น Flat Namespace แบบเก่า (`1.3.6.1.2.1.2.2.1.10.1` คือ `ifInOctets` ของ
interface index 1) — ไม่มีโครงสร้าง Hierarchical ที่ชัดเจนแบบ YANG และ Vendor แต่ละเจ้ามี MIB
ของตัวเองปนกับ MIB มาตรฐาน (`IF-MIB`, `CISCO-PROCESS-MIB`) ทำให้ Tooling ที่ทำ Multi-vendor
Correlation ทำได้ยากกว่า YANG ที่มี Schema ตายตัวและตรวจสอบได้ล่วงหน้า (ตามที่ Part 49 Step 481
อธิบายไว้)

### สรุปตารางเปรียบเทียบ — จุดที่ผลักดันให้เกิด Streaming Telemetry

| ปัญหาของ SNMP Polling | ผลกระทบจริง | สิ่งที่ Streaming Telemetry แก้ได้ |
|---|---|---|
| Fixed Interval พลาด Transient Event | มองไม่เห็น Micro-burst / Flapping สั้นๆ | Push ทันทีที่ค่าเปลี่ยน (Event-driven) หรือ Cadence ที่ถี่กว่ามาก (ระดับวินาที) โดยไม่เพิ่ม Overhead แบบ Poll |
| Scalability แย่เมื่อ Poll ถี่ + อุปกรณ์เยอะ | CPU อุปกรณ์และ NMS โอเวอร์โหลด | Subscribe ครั้งเดียว อุปกรณ์ส่งข้อมูลเองต่อเนื่อง (ไม่ต้องส่ง Request ซ้ำ) |
| MIB/OID ไม่มี Schema มาตรฐานเทียบ Vendor อื่น | Correlation ข้าม Vendor ยาก | ใช้ **YANG Model เดียวกัน** กับที่ Part 49 สอน (Native/OpenConfig/IETF) |
| UDP ไม่มี Delivery Guarantee ที่ดี | Trap อาจหายได้ (UDP ไม่ Retransmit) | gRPC บน TCP มี Reliable Transport ในตัว (Step 533) |

> **ประเด็นสำคัญที่สุด**: Streaming Telemetry **ไม่ได้มาแทนที่ SNMP ทั้งหมด** — SNMP ยังใช้ได้ดีกับ
> อุปกรณ์เก่าหรือ Use Case ที่ไม่ต้องการ Real-time สูง (Part 17 ยังใช้งานได้จริงเสมอ) แต่สำหรับ
> **Assurance ระดับ Production ที่ต้องการเห็นเครือข่าย "ตอนนี้ ณ วินาทีนี้"** Streaming Telemetry
> คือคำตอบที่อุตสาหกรรมกำลังเปลี่ยนไปใช้ (Cisco DNA Center Assurance ที่กล่าวถึงใน
> [Part 50](part-050-dna-center.md) ก็ใช้ MDT เป็น Engine หลักเบื้องหลัง Dashboard ที่เห็น)

---

## Step 532 — Model-Driven Telemetry (MDT) พื้นฐาน

### แนวคิดหลัก: YANG Model เดิม แต่ใช้กับ "การรับข้อมูล" แทน "การส่ง Config"

จุดที่สำคัญที่สุดที่ต้องเข้าใจก่อนเรียน MDT คือ — **เราไม่ได้เรียนรู้ Data Model ใหม่เลย**
[Part 49 Step 481-482](part-049-netconf-restconf-yang-deep-dive.md) สอน YANG Module 3 ประเภท
(Native, OpenConfig, IETF) ไว้สำหรับ **Config** ผ่าน NETCONF/RESTCONF — MDT ใช้ YANG Module
กลุ่มเดียวกันนี้เลย แต่เปลี่ยนจาก Path ที่เป็น **Config Container** (`rw`, read-write) ไปเป็น
Path ที่เป็น **Operational/State Container** (`ro`, read-only, มักลงท้ายด้วย `-oper`) แล้ว
**Subscribe** ให้อุปกรณ์ Push ค่าออกมาเรื่อยๆ แทนการ GET ทีละครั้ง

```
NETCONF/RESTCONF (Part 49)                          Model-Driven Telemetry (Part นี้)
┌─────────────────────────────┐                     ┌─────────────────────────────┐
│ YANG Module: ietf-interfaces  │                     │ YANG Module: Cisco-IOS-XE-    │
│ Path: /interfaces/interface   │   ── Schema ──►     │ interfaces-oper (state)       │
│ Operation: get-config/edit    │   เดียวกัน           │ Path: /interfaces/interface/  │
│ ทิศทาง: Client ร้องขอ (Pull)  │   (แนวคิด)           │       statistics               │
│ ผลลัพธ์: Snapshot ครั้งเดียว  │                     │ Operation: Subscribe (ครั้งเดียว)│
└─────────────────────────────┘                     │ ทิศทาง: Device ส่งเอง (Push)   │
                                                       │ ผลลัพธ์: Stream ต่อเนื่องตลอดเวลา│
                                                       └─────────────────────────────┘
```

### สององค์ประกอบหลักที่ต้องกำหนดเสมอเมื่อ Subscribe

| องค์ประกอบ | ความหมาย |
|---|---|
| **Sensor-Path** | YANG Path ที่ต้องการ Stream ข้อมูล (เช่น interface counters, CPU utilization) — เทียบเท่า OID ของ SNMP แต่มี Schema ตายตัวและลึกกว่า |
| **Destination** | ปลายทางที่จะส่งข้อมูลไป (IP + Port ของ Collector) พร้อม Encoding ที่ใช้ (Step 533) |
| **Subscription** | การ "ผูก" Sensor-Path เข้ากับ Destination พร้อม Cadence (ความถี่) หรือ Trigger (Event-driven) |

### Dial-in vs Dial-out: สองโหมด Subscription ที่ต้องเลือก

| ประเด็น | **Dial-in** | **Dial-out** |
|---|---|---|
| ใครเป็นผู้เริ่มเชื่อมต่อ | **Collector** เชื่อมต่อเข้าไปที่อุปกรณ์ (อุปกรณ์ต้องรัน gRPC Server รอรับ) | **อุปกรณ์** เป็นผู้เชื่อมต่อออกไปที่ Collector (อุปกรณ์เป็น gRPC Client) |
| Subscription ถูกสร้างที่ไหน | สร้างแบบ **Dynamic** ผ่าน gRPC RPC Call ตอน Runtime (ไม่ต้อง config ล่วงหน้าบนอุปกรณ์ก็ได้) หรือ config ไว้ล่วงหน้าก็ได้ | ต้อง **Config ไว้ล่วงหน้าบนอุปกรณ์เสมอ** (persistent, อยู่ใน running-config) |
| Firewall/NAT Friendliness | ต้องเปิด Port เข้าไปที่อุปกรณ์ (inbound) — ยากกว่าถ้าอุปกรณ์อยู่หลัง NAT/Firewall | **เหมาะกับ Production มากกว่า** — อุปกรณ์เป็นฝ่าย Initiate ออก (outbound) เหมือน Syslog/NTP client ปกติ ไม่ต้องเปิด Inbound Port ที่ Edge |
| Use Case ที่เหมาะ | Collector ต้องการ Subscribe/Unsubscribe แบบ Dynamic บ่อยๆ (เช่น Tool วิเคราะห์ที่เปิด Subscribe ชั่วคราวตอน Troubleshoot) | Production Monitoring แบบถาวร ที่ต้องการ Stream ต่อเนื่องตลอดเวลาโดยไม่ต้องมี Human สั่ง Subscribe ใหม่ทุกครั้ง |
| ใช้ใน Lab นี้ | กล่าวถึงเพื่อความเข้าใจ | **✅ ใช้จริงใน Step 534/540** — เพราะ NOC-SRV (Collector) ควรอยู่ฝั่ง MGMT VLAN 99 ที่ปิด ไม่ต้องเปิด Inbound Port จากอุปกรณ์ Core/Dist เข้ามาหา |

> **ทำไม Dial-out ถึงเป็นตัวเลือกที่ Enterprise ส่วนใหญ่ใช้จริง**: แนวคิดเดียวกับ Syslog
> (`logging host <ip>`, [Part 17 Step 165](part-017-ntp-syslog-snmp.md)) — อุปกรณ์เป็นฝ่าย
> ส่งออกไปหา Collector เสมอ ทำให้ Security Model ง่ายกว่ามาก (Collector ไม่ต้องเปิด Listening
> Port ที่ Trust Boundary ต่ำกว่า, ไม่ต้องยุ่งกับ NAT/ACL ขาเข้าที่ Core/Distribution)

---

## Step 533 — Telemetry Transport: gRPC และ Protobuf

### gRPC คืออะไร

**gRPC (Google Remote Procedure Call)** คือ Framework สำหรับเรียก Function ข้าม Network
(Remote Procedure Call) ที่ Google พัฒนาและปล่อยเป็น Open Source — เป็น Transport มาตรฐานที่
Cisco เลือกใช้สำหรับ MDT เพราะมีคุณสมบัติที่เหมาะกับงาน Streaming Telemetry โดยเฉพาะ:

| คุณสมบัติของ gRPC | ประโยชน์ต่อ Telemetry |
|---|---|
| ทำงานบน **HTTP/2** | รองรับ **Multiplexing** — ส่งหลาย Stream ข้อมูล (เช่น interface stats, CPU stats) พร้อมกันบน Connection เดียว ไม่ต้องเปิด Connection ใหม่ทุกครั้ง |
| **Bidirectional Streaming** ในตัว | อุปกรณ์ Push ข้อมูลต่อเนื่องได้โดยไม่ต้องมี Client Request ซ้ำๆ (ต่างจาก HTTP/1.1 REST แบบ Request-Response ทีละครั้ง) |
| ใช้ **Protocol Buffers (Protobuf)** เป็น Serialization Format | ข้อมูลถูก Encode เป็น **Binary** ที่กระชับกว่า JSON/XML มาก (เล็กกว่า, Parse เร็วกว่า) — สำคัญมากเมื่อต้อง Stream ข้อมูลปริมาณสูงต่อเนื่อง |
| รันบน **TCP** | มี Reliable Delivery ในตัว (Retransmit อัตโนมัติถ้า Packet หาย) — ต่างจาก SNMP Trap ที่วิ่งบน UDP และอาจหายไปเงียบๆ |

### Protobuf คืออะไร (สั้นๆ พอเข้าใจ)

**Protocol Buffers** คือภาษาสำหรับนิยาม Structure ของข้อมูล (คล้าย YANG แต่เป็นระดับ
Serialization ไม่ใช่ระดับ Business Data Model) แล้ว Compiler (`protoc`) จะ Generate Code
(Python/Go/Java/C++) ให้ Encode/Decode ข้อมูลตาม Structure นั้นโดยอัตโนมัติ ตัวอย่างไฟล์
`.proto` แบบง่าย:

```protobuf
// telemetry_sample.proto (ตัวอย่างแนวคิด - ย่อจาก Cisco telemetry.proto จริง)
syntax = "proto3";

message TelemetryField {
  string name = 1;
  uint64 uint64_value = 2;
  string string_value = 3;
}

message Telemetry {
  string node_id_str = 1;        // hostname อุปกรณ์ต้นทาง
  string subscription_id_str = 2; // อ้างอิง subscription ที่ config ไว้
  uint64 msg_timestamp = 3;       // timestamp แบบ Unix epoch millisecond
  repeated TelemetryField data_gpbkv = 4;  // ข้อมูลจริงแบบ Key-Value ซ้อนกันได้หลายชั้น
}
```

Cisco IOS-XE ใช้รูปแบบ Encoding ที่เรียกว่า **`encode-kvgpb`** (Key-Value Google Protobuf) —
คือการ Encode Path/Value เป็นคู่ Key-Value ซ้อนกันตามโครงสร้าง YANG Path แทนที่จะสร้าง
`.proto` message เฉพาะเจาะจงสำหรับทุก YANG Model (ซึ่งจะมีจำนวนมากเกินจะ compile ล่วงหน้าได้
ครบทุก Model) — ข้อดีคือ Generic ใช้ได้กับทุก Sensor-Path โดยไม่ต้อง Compile Proto File ใหม่
ทุกครั้งที่เปลี่ยน YANG Model ที่ Subscribe

### เปรียบเทียบ Transport รุ่นเก่า vs gRPC

| Transport | ยุค | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **SNMP Trap (UDP/162)** | รุ่นเก่าที่สุด | ง่าย, รองรับกว้างมากที่สุด | UDP ไม่ Reliable, MIB Schema จำกัด, ไม่รองรับ Volume สูง |
| **Raw TCP Streaming (Cisco Legacy "Telemetry over TCP")** | ยุคเริ่มต้นของ MDT (ก่อน gRPC เป็นมาตรฐาน) | Reliable กว่า UDP (บน TCP) | ไม่มี Multiplexing, ต้องเปิด Connection แยกต่อ Subscription, ไม่มี Flow Control ระดับ Stream |
| **UDP Streaming (Cisco Legacy "Telemetry over UDP")** | ยุคเดียวกับ Raw TCP | Overhead ต่ำที่สุด, เหมาะกับ Volume สูงมากที่ยอมรับข้อมูลหายได้บ้าง | ไม่ Reliable เหมือน SNMP Trap, ไม่เหมาะกับข้อมูลที่ต้องครบทุก Record |
| **gRPC (Dial-in/Dial-out)** | **มาตรฐานปัจจุบัน** | Reliable (TCP), Multiplexing, Protobuf กระชับ, Standard-based (ใช้กับ Vendor อื่นที่รองรับ gNMI ได้ด้วย) | ซับซ้อนกว่าในการ Implement Client (ต้องมี Protobuf Stub) — แต่ Library สำเร็จรูปช่วยได้มาก (Step 536) |

> **สรุปสำหรับ Production ปัจจุบัน**: ให้ใช้ **gRPC เป็นค่าเริ่มต้นเสมอ** สำหรับ Deployment ใหม่
> ทุกกรณี — Transport รุ่นเก่า (Raw TCP/UDP) ยังพบได้ในอุปกรณ์ IOS-XE รุ่นเก่ามาก แต่ Cisco เอง
> ก็ผลักดันให้ Deprecate ไปใช้ gRPC (และมาตรฐาน **gNMI — gRPC Network Management Interface**
> ที่ต่อยอดจาก gRPC สำหรับงาน Multi-vendor โดยเฉพาะ) เป็นทิศทางหลักของอุตสาหกรรม

---

## Step 534 — Configuring MDT บน IOS-XE: Worked Config เต็มรูปแบบ

### สามส่วนหลักที่ต้อง Config (แนวทาง Dial-out — ตามที่เลือกใน Step 532)

```
telemetry ietf destination-group <destination-group-name>
 destination-id <id>
 destination ip address <collector-ip> port <port> protocol grpc-tcp encoding encode-kvgpb
!
telemetry ietf sensor-group <sensor-group-name>
 sensor-path <yang-path>
!
telemetry ietf subscription <subscription-id>
 sensor-group-id <sensor-group-name> sample-interval <milliseconds>
 destination-id <destination-group-name>
```

| ส่วน | หน้าที่ | เทียบเท่าแนวคิดกับ Syslog (Part 17) |
|---|---|---|
| `destination-group` | กำหนด "ปลายทาง" (Collector IP/Port/Encoding) — ใช้ซ้ำได้กับหลาย Subscription | เทียบเท่า `logging host <ip>` |
| `sensor-group` | กำหนด "อะไร" ที่จะ Stream (YANG Sensor-Path) — ใช้ซ้ำได้กับหลาย Subscription | เทียบเท่า `logging trap <level>` (กำหนดว่าจะส่งอะไร) |
| `subscription` | "ผูก" sensor-group เข้ากับ destination-group พร้อมกำหนด Cadence (`sample-interval`) | เทียบเท่าการรวม Host + Level เข้าด้วยกันเป็น Policy เดียว |

### Worked Config เต็มรูปแบบ — CORE-SW1 Stream CPU + Interface Statistics ไปยัง NOC-SRV

เราจะให้ **NOC-SRV (10.10.99.50)** ทำหน้าที่เป็น Telemetry Collector เพิ่มเติมจากบทบาท
NTP/Syslog/SNMP เดิม ([Part 17](part-017-ntp-syslog-snmp.md)) — รับ gRPC Dial-out บน
**TCP Port 57500** (ตัวเลข Port นี้กำหนดเองได้ ไม่ใช่ Well-known Port ตายตัว)

```
CORE-SW1(config)# telemetry ietf destination-group DG-NOC-COLLECTOR
CORE-SW1(config-telemetry-dest-grp)# destination-id 1
CORE-SW1(config-telemetry-dest-grp)# destination ip address 10.10.99.50 port 57500 protocol grpc-tcp encoding encode-kvgpb
CORE-SW1(config-telemetry-dest-grp)# exit

! --- Sensor-Group 1: CPU Utilization (YANG Model: Cisco-IOS-XE-process-cpu-oper) ---
CORE-SW1(config)# telemetry ietf sensor-group SG-CPU
CORE-SW1(config-telemetry-sensor-grp)# sensor-path Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
CORE-SW1(config-telemetry-sensor-grp)# exit

! --- Sensor-Group 2: Interface Statistics (YANG Model: Cisco-IOS-XE-interfaces-oper) ---
CORE-SW1(config)# telemetry ietf sensor-group SG-INTERFACE
CORE-SW1(config-telemetry-sensor-grp)# sensor-path Cisco-IOS-XE-interfaces-oper:interfaces/interface
CORE-SW1(config-telemetry-sensor-grp)# exit

! --- Subscription 100: ผูก SG-CPU กับ DG-NOC-COLLECTOR ทุก 10 วินาที ---
CORE-SW1(config)# telemetry ietf subscription 100
CORE-SW1(config-telemetry-subscription)# sensor-group-id SG-CPU sample-interval 10000
CORE-SW1(config-telemetry-subscription)# destination-id DG-NOC-COLLECTOR
CORE-SW1(config-telemetry-subscription)# source-address 1.1.1.1
CORE-SW1(config-telemetry-subscription)# exit

! --- Subscription 101: ผูก SG-INTERFACE กับ DG-NOC-COLLECTOR ทุก 30 วินาที ---
CORE-SW1(config)# telemetry ietf subscription 101
CORE-SW1(config-telemetry-subscription)# sensor-group-id SG-INTERFACE sample-interval 30000
CORE-SW1(config-telemetry-subscription)# destination-id DG-NOC-COLLECTOR
CORE-SW1(config-telemetry-subscription)# source-address 1.1.1.1
CORE-SW1(config-telemetry-subscription)# end
```

| คำสั่ง/Parameter | ความหมาย |
|---|---|
| `destination ip address ... protocol grpc-tcp` | ระบุ Transport เป็น gRPC over TCP (ทางเลือกอื่นคือ `grpc-tls` สำหรับ Production ที่ต้องการ Encryption) |
| `encoding encode-kvgpb` | Encoding แบบ Key-Value GPB ตามที่อธิบายใน Step 533 |
| `sensor-path <yang-path>` | YANG Path ของข้อมูลที่ต้องการ — ใช้ Path เดียวกันกับที่ RESTCONF ใช้ query ได้ (ดู [Part 49 Step 485](part-049-netconf-restconf-yang-deep-dive.md)) |
| `sample-interval <ms>` | ความถี่ในการส่งข้อมูล หน่วย **millisecond** — `10000` = ทุก 10 วินาที (Cadence-based, ตรงข้ามกับ Event-driven ที่ส่งเมื่อค่าเปลี่ยนเท่านั้น) |
| `source-address` | Source IP ที่ใช้ส่ง gRPC Connection ออกไป — ใช้ Loopback0 เพื่อความ Stable เหมือนหลักการเดียวกับ `logging source-interface` ([Part 17](part-017-ntp-syslog-snmp.md)) |

### คำสั่ง Verify MDT

```
CORE-SW1# show telemetry ietf subscription all

Telemetry subscription brief:
ID          Type            State        Filter type
--------------------------------------------------------
100         Configured      Valid        By Sensor-Path
101         Configured      Valid        By Sensor-Path

CORE-SW1# show telemetry ietf subscription 100 detail

Subscription ID: 100
Type: Configured
State: Valid
Destination-group: DG-NOC-COLLECTOR
  Destination: 10.10.99.50 : 57500, Protocol: grpc-tcp, Encoding: encode-kvgpb
  State of last connection attempt: Connected
Sensor-group-id: SG-CPU, Sample-Interval: 10000 ms
  Sensor-path: Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
  Sensor-path State: Resolved

CORE-SW1# show telemetry transport grpc statistics

Sessions:
Address              Port   Sessions
10.10.99.50           57500  1
Total Bytes Sent: 128456
Total Bytes Sent Errors: 0
Total Bytes Received: 512
```

**อ่าน output อย่างไร**: `State: Valid` และ `State of last connection attempt: Connected`
คือสองบรรทัดที่ต้องเห็นเพื่อยืนยันว่า Config ถูกต้องและ CORE-SW1 กำลัง Stream ข้อมูลไปยัง
NOC-SRV จริง ถ้าเห็น `State: Invalid` หรือ `Not Connected` ให้ตรวจสอบว่า Collector Listening
อยู่ที่ Port 57500 จริงหรือไม่ (ดู Step 535-536) และ Connectivity ระหว่าง CORE-SW1 กับ NOC-SRV
ผ่าน VLAN 99 (MGMT) ปกติดีหรือไม่

---

## Step 535 — Telemetry Collector และ Pipeline

### Streaming Telemetry ต้องมี "ผู้รับ" ที่ทำอะไรได้มากกว่าแค่รับข้อมูล

การ Config MDT ใน Step 534 ทำให้ CORE-SW1 ส่งข้อมูลออกไป แต่ข้อมูลที่ได้จะไม่มีประโยชน์เลย
ถ้าไม่มี **Pipeline** ที่ (1) รับข้อมูล gRPC ดิบ, (2) Decode Protobuf, (3) แปลงเป็นรูปแบบที่
เก็บใน Time-series Database ได้, (4) แสดงผลเป็น Dashboard ที่คนอ่านเข้าใจได้ทันที — ในอุตสาหกรรม
มีสองแนวทางหลักที่ใช้กันจริง

### แนวทางที่ 1: TIG Stack (Telegraf → InfluxDB → Grafana) — Open Source

```
┌──────────┐  gRPC    ┌────────────────────┐  write   ┌────────────┐  query   ┌──────────┐
│ CORE-SW1 │ ───────► │ Telegraf            │ ───────► │ InfluxDB    │ ───────► │ Grafana   │
│ (Device) │  Dial-out│ (cisco_telemetry_mdt│  metrics │ (Time-series│  PromQL/ │ (Dashboard│
└──────────┘          │  input plugin)      │          │  Database)  │  Flux    │  UI)      │
                       └────────────────────┘          └────────────┘          └──────────┘
                       รันบน NOC-SRV
```

| Component | หน้าที่ |
|---|---|
| **Telegraf** | Agent เขียนด้วย Go ที่มี Input Plugin ชื่อ `cisco_telemetry_mdt` โดยเฉพาะ — เปิด gRPC Server รอรับ Dial-out จากอุปกรณ์ Cisco, Decode `encode-kvgpb` ให้อัตโนมัติ แล้วส่งต่อ (Output Plugin) ไปยัง InfluxDB |
| **InfluxDB** | Time-series Database ที่ Optimize สำหรับข้อมูลที่มี Timestamp มหาศาล (Metric ทุกวินาทีจากหลายร้อยอุปกรณ์) — Query เร็วกว่า Relational DB ทั่วไปมากสำหรับ Pattern แบบนี้ |
| **Grafana** | Web Dashboard ที่ Query จาก InfluxDB มาแสดงเป็น Graph/Panel — รองรับ Alert Rule ในตัว (แจ้งเตือนเมื่อ Metric เกิน Threshold) |

**Config ตัวอย่าง Telegraf (`telegraf.conf` บน NOC-SRV) — ส่วนที่เกี่ยวกับ MDT**:

```toml
[[inputs.cisco_telemetry_mdt]]
  transport = "grpc"
  service_address = ":57500"        # Port เดียวกับที่ config ไว้บน CORE-SW1 (Step 534)
  max_msg_size = 4000000

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "$INFLUX_TOKEN"
  organization = "netops-lab"
  bucket = "telemetry"
```

### แนวทางที่ 2: Cisco Crosswork / Network Assurance Engine (NAE) — Commercial

**Cisco Crosswork** (รวมถึง Network Assurance Engine ที่เป็นส่วนประกอบ) คือ Platform เชิงพาณิชย์
ที่ Cisco สร้างขึ้นมาโดยเฉพาะสำหรับงาน Assurance ระดับ Enterprise/Service Provider — ทำหน้าที่
เดียวกับ TIG Stack (รับ Telemetry → เก็บ → วิเคราะห์ → แสดงผล) แต่เพิ่มเติมสิ่งที่ Open Source
Stack ไม่มีมาให้ในตัว:

| ความสามารถเพิ่มเติมของ Crosswork/NAE เทียบกับ TIG Stack | รายละเอียด |
|---|---|
| **Built-in Anomaly Detection Model** | มี Machine Learning Model ที่เรียนรู้ Baseline ของเครือข่ายโดยอัตโนมัติ (ไม่ต้องตั้ง Threshold ตายตัวเองแบบ Grafana Alert) |
| **Intent-based Assurance** | เชื่อมกับ Cisco DNA Center ([Part 50](part-050-dna-center.md)) เพื่อตรวจสอบว่า Network State ตรงกับ "Intent" ที่ Design ไว้หรือไม่ (เช่น ควรมี ECMP 2 Path เสมอ ถ้าเหลือ 1 Path = Anomaly) |
| **Root Cause Analysis อัตโนมัติ** | เชื่อม Event หลายตัวที่เกิดต่อเนื่องกัน (เช่น Interface Flap → OSPF Neighbor Down → Route Reconverge) เข้าเป็น "1 เหตุการณ์" แทนแจ้งเตือนแยกกันเป็น 10 Alert |
| **License/Cost** | Commercial License ตามจำนวนอุปกรณ์ — เหมาะกับองค์กรขนาดใหญ่ที่มีงบและต้องการ Support จาก Cisco โดยตรง |

### ตารางเปรียบเทียบสรุปสำหรับการเลือกใช้

| ประเด็น | TIG Stack (Open Source) | Cisco Crosswork/NAE (Commercial) |
|---|---|---|
| ค่าใช้จ่าย | ฟรี (License) แต่ต้องมี Engineer ดูแลเอง | มีค่า License ตาม Scale |
| ความยืดหยุ่นในการ Customize | สูงมาก (Config/Dashboard ปรับได้ทุกอย่าง) | จำกัดตาม Feature ที่ Cisco ให้มา |
| Anomaly Detection | ต้องตั้ง Threshold เอง (Grafana Alert Rule) หรือเขียน Script เอง (Step 539) | มี ML Model ในตัว, เรียนรู้ Baseline อัตโนมัติ |
| Time-to-Value | ต้องใช้เวลา Setup/Tune เอง | เร็วกว่า (Deploy แล้วใช้ Feature สำเร็จรูปได้ทันที) |
| เหมาะกับ | Lab, SMB, องค์กรที่มีทีม NetOps/DevOps แข็งแรง | Enterprise/SP ขนาดใหญ่ที่ต้องการ Support และ Feature ครบ |
| ใช้ในหลักสูตรนี้ | **✅ ใช้จริงใน Step 536-540** (เพราะเข้าถึงได้ทุกคนไม่ต้องมี License) | กล่าวถึงเพื่อความรู้ (แนวคิดเดียวกับที่ [Part 50](part-050-dna-center.md) พูดถึง Assurance Dashboard) |

---

## Step 536 — Python gRPC Telemetry Receiver

### ข้อจำกัดสำคัญที่ต้องเข้าใจก่อนอ่าน Code

Client/Server gRPC ที่ทำงานได้จริง 100% ต้อง **Compile ไฟล์ `.proto`** (เช่น `telemetry.proto`
และ `mdt_dialout.proto` ที่ Cisco เผยแพร่ใน [ios-xe-mdt-telemetry GitHub repo](https://github.com/cisco-ios-xe-programmability))
ด้วยคำสั่ง `python -m grpc_tools.protoc` ก่อน จึงจะได้ไฟล์ `_pb2.py` และ `_pb2_grpc.py`
ที่มี Class สำหรับ Encode/Decode Message จริง — Script ด้านล่างนี้แสดง **โครงสร้างแนวคิด**
ของ gRPC Dial-out Server ให้เห็นภาพชัดเจน โดยจำลอง Decode Function ไว้แบบง่าย (`decode_kvgpb`)
เพื่อไม่ต้องพา Environment การ Compile Proto เข้ามาเป็นเงื่อนไขของการเข้าใจแนวคิดหลัก

### ติดตั้ง Library ที่จำเป็น

```bash
pip install grpcio grpcio-tools protobuf
```

### โครงสร้าง Proto ของ MDT Dial-out (ย่อจาก `mdt_dialout.proto` จริงของ Cisco)

```protobuf
// mdt_dialout.proto (แนวคิดจากของจริง — Cisco เผยแพร่แบบ Open Source)
syntax = "proto3";
package mdt_dialout;

service gRPCMdtDialout {
  rpc MdtDialout(stream MdtDialoutArgs) returns (stream MdtDialoutArgs) {}
}

message MdtDialoutArgs {
  int64 ReqId = 1;
  bytes data = 2;      // <- ข้อมูล Telemetry จริง เข้ารหัสแบบ encode-kvgpb (Step 533)
  string errors = 3;
}
```

### Script เต็ม: `mdt_grpc_receiver.py` — gRPC Dial-out Server รับ Stream จาก CORE-SW1

```python
#!/usr/bin/env python3
"""
mdt_grpc_receiver.py
เปิด gRPC Server รอรับ Model-Driven Telemetry Dial-out จาก CORE-SW1 (และอุปกรณ์อื่นใน Lab)
บน Port 57500 (ตรงกับที่ config ไว้ใน telemetry ietf destination-group ของ Step 534)

หมายเหตุสำคัญ: decode_kvgpb() ด้านล่างเป็น Placeholder แบบง่าย เพื่อแสดงแนวคิดโครงสร้าง
ของการรับ+แยกข้อมูล Telemetry เท่านั้น การ Decode จริงต้องใช้ Class ที่ Generate จาก
telemetry.proto ตัวจริงของ Cisco (ผ่าน grpc_tools.protoc) ซึ่งมี GPBKV structure ที่ซับซ้อนกว่านี้
"""

import time
from concurrent import futures
import grpc

# ในของจริงต้อง import Class ที่ generate จาก mdt_dialout_pb2 / mdt_dialout_pb2_grpc
# from mdt_dialout_pb2 import MdtDialoutArgs
# from mdt_dialout_pb2_grpc import gRPCMdtDialoutServicer, add_gRPCMdtDialoutServicer_to_server

LISTEN_PORT = 57500


def decode_kvgpb(raw_bytes: bytes) -> dict:
    """
    Placeholder function จำลองการ Decode encode-kvgpb payload
    ของจริง: ต้อง parse Telemetry protobuf message แล้วไล่ recursive ผ่าน
    data_gpbkv (list ของ TelemetryField ที่ซ้อนกันได้หลายชั้นตาม YANG Path)
    ในที่นี้จำลองผลลัพธ์ให้เห็นภาพว่า Decode แล้วจะได้ dict หน้าตาแบบนี้
    """
    return {
        "node_id_str": "CORE-SW1",
        "subscription_id_str": "101",
        "encoding_path": "Cisco-IOS-XE-interfaces-oper:interfaces/interface",
        "collection_id": int(time.time()),
        "interfaces": [
            {"name": "TenGigabitEthernet1/0/1", "in-octets": 128456789, "in-errors": 0},
            {"name": "TenGigabitEthernet1/0/2", "in-octets": 98234123, "in-errors": 0},
        ],
    }


class MdtDialoutServicer:
    """
    Servicer ที่ Implement RPC Method MdtDialout()
    ของจริงต้อง inherit จาก gRPCMdtDialoutServicer (generated class)
    """

    def MdtDialout(self, request_iterator, context):
        for request in request_iterator:
            # request คือ MdtDialoutArgs message ที่มี field .data เป็น bytes ดิบ
            payload = decode_kvgpb(request.data if hasattr(request, "data") else b"")
            self._print_sample(payload)

            # ตัวอย่าง: ส่งต่อให้ระบบอื่น (InfluxDB, Anomaly Detector ใน Step 539) ที่นี่
            yield_downstream(payload)

        return iter([])   # Bidirectional stream — ส่ง Ack กลับได้ถ้าจำเป็น (มักไม่ใช้ฝั่งนี้)

    @staticmethod
    def _print_sample(payload: dict):
        print(f"\n[{time.strftime('%H:%M:%S')}] Telemetry Sample จาก {payload['node_id_str']}")
        print(f"  Subscription ID : {payload['subscription_id_str']}")
        print(f"  Encoding Path   : {payload['encoding_path']}")
        for intf in payload.get("interfaces", []):
            print(f"    - {intf['name']:30s} in-octets={intf['in-octets']:<12} "
                  f"in-errors={intf['in-errors']}")


def yield_downstream(payload: dict):
    """จุดเชื่อมต่อไปยัง Pipeline อื่น เช่น เขียนลง InfluxDB หรือส่งเข้า Anomaly Detector (Step 539)"""
    pass  # ใน Lab จริง: influx_client.write(payload) หรือ anomaly_queue.put(payload)


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))

    # ของจริง: add_gRPCMdtDialoutServicer_to_server(MdtDialoutServicer(), server)
    print(f"[*] MDT gRPC Dial-out Receiver กำลังรอรับ Connection ที่ port {LISTEN_PORT} ...")
    server.add_insecure_port(f"[::]:{LISTEN_PORT}")   # Lab เท่านั้น — Production ควรใช้ TLS credentials
    server.start()

    try:
        server.wait_for_termination()
    except KeyboardInterrupt:
        print("\n[*] ปิด Receiver")
        server.stop(0)


if __name__ == "__main__":
    serve()
```

### ตัวอย่าง Output ที่คาดหวังเมื่อรันคู่กับ CORE-SW1 ที่ Config ไว้จาก Step 534

```
$ python3 mdt_grpc_receiver.py
[*] MDT gRPC Dial-out Receiver กำลังรอรับ Connection ที่ port 57500 ...

[14:32:10] Telemetry Sample จาก CORE-SW1
  Subscription ID : 101
  Encoding Path   : Cisco-IOS-XE-interfaces-oper:interfaces/interface
    - TenGigabitEthernet1/0/1        in-octets=128456789   in-errors=0
    - TenGigabitEthernet1/0/2        in-octets=98234123    in-errors=0

[14:32:40] Telemetry Sample จาก CORE-SW1
  Subscription ID : 101
  Encoding Path   : Cisco-IOS-XE-interfaces-oper:interfaces/interface
    - TenGigabitEthernet1/0/1        in-octets=129801022   in-errors=0
    - TenGigabitEthernet1/0/2        in-octets=99012456    in-errors=0
```

> **สำหรับใช้งานจริงใน Production**: แทนที่จะเขียน gRPC Server เองแบบข้างบน แนะนำให้ใช้
> **Telegraf `cisco_telemetry_mdt` Input Plugin** (Step 535) ที่ Cisco และ InfluxData ดูแล
> Decode Logic ให้ครบถูกต้องตาม `telemetry.proto` ตัวจริงอยู่แล้ว — Code ใน Step นี้มีไว้เพื่อ
> ให้เข้าใจว่า **เบื้องหลัง Telegraf ทำอะไรอยู่** ซึ่งช่วยมากตอน Troubleshoot ปัญหา Pipeline จริง

---

## Step 537 — Network Assurance: Baseline, Anomaly Detection, Proactive vs Reactive

### Network Assurance คืออะไร

**Network Assurance** คือแนวคิด (ไม่ใช่ Tool ตัวเดียว) ของการใช้ข้อมูล Telemetry/State ที่เก็บ
มาอย่างต่อเนื่อง เพื่อ**ยืนยันอย่างต่อเนื่องว่าเครือข่ายทำงานตามที่ตั้งใจไว้ (Intent)** — ต่างจาก
Monitoring แบบเดิมที่แค่ "แสดงค่าปัจจุบัน" Assurance ต้องตอบคำถามที่ลึกกว่า: **"ค่านี้ปกติหรือ
ผิดปกติสำหรับเครือข่ายเรา?"**

### Baseline คืออะไร — "ปกติ" ของเครือข่ายนี้ ไม่เท่ากับ "ปกติ" ของเครือข่ายอื่น

**Baseline** คือช่วงค่าที่ถือว่า "ปกติ" สำหรับ Metric หนึ่งๆ **ของเครือข่ายเรานี้โดยเฉพาะ**
(ไม่ใช่ค่า Threshold ตายตัวที่ก็อปจากที่อื่นมา) เพราะแต่ละเครือข่ายมี Pattern การใช้งานต่างกัน

| ตัวอย่าง Metric | Baseline ที่อาจพบใน Lab นี้ | ทำไมถึงเป็นแบบนี้ |
|---|---|---|
| CPU Utilization ของ CORE-SW1 | 15-25% (Normal), Spike ถึง 60% ช่วง 08:00-09:00 (Peak Login Hour) | ช่วงเช้าพนักงานเข้า Office พร้อมกัน (Authentication burst, DHCP burst) |
| Interface Error Rate บน Uplink Backbone | 0 error/นาที (Normal) | Fiber ที่ดีไม่ควรมี Error เลยแม้แต่ 1 | 
| OSPF Neighbor Count บน DIST-SW1 | คงที่ = 2 เสมอ (CORE-SW1 + Adjacent Dist ถ้ามี) | เปลี่ยนจาก 2 เหลือ 1 = สูญเสีย Redundancy ทันที |
| จำนวนครั้งที่ HSRP เปลี่ยน State ต่อวัน | 0 ครั้ง (Normal — Active ควรอยู่ตัวเดิมตลอด) | HSRP Flap บ่อยผิดปกติ = สัญญาณของ Physical Link ไม่แน่นอน |

> **กฎสำคัญ**: Baseline ต้องมาจาก**การสังเกตข้อมูลจริงของเครือข่ายนั้นเป็นระยะเวลาหนึ่ง**
> (มักอย่างน้อย 1-2 สัปดาห์เพื่อจับ Pattern รายวัน/รายสัปดาห์ให้ครบ) ไม่ใช่ตัวเลขที่เดาขึ้นมาเอง
> หรือลอกจาก Best Practice ทั่วไปที่ไม่ได้คำนึงถึง Pattern เฉพาะขององค์กรนั้น

### Anomaly Detection — เมื่อค่าจริงเบี่ยงเบนไปจาก Baseline

**Anomaly (ความผิดปกติ)** คือค่าที่เบี่ยงเบนออกจาก Baseline อย่างมีนัยสำคัญ — วิธี Detect
มีตั้งแต่ง่ายไปซับซ้อน:

| ระดับความซับซ้อน | วิธี | ตัวอย่าง |
|---|---|---|
| **ง่ายที่สุด: Static Threshold** | ตั้งค่าคงที่ตายตัว (เช่น "CPU > 80% = Alert") | ง่าย ใช้ได้ทันที แต่ไม่รู้จัก Pattern เวลา (Alert ตอน Peak Hour ปกติทุกวันก็ได้ False Positive) |
| **ปานกลาง: Threshold ตาม Time Window** | ตั้ง Threshold ต่างกันตามช่วงเวลา (เช่น Peak Hour อนุญาต CPU สูงกว่า Off-hour) | ลด False Positive ได้มาก แต่ต้อง Maintain Rule เยอะขึ้น |
| **ซับซ้อน: Statistical Baseline (Standard Deviation)** | คำนวณค่าเฉลี่ย + Standard Deviation ของ Metric ย้อนหลัง แล้ว Alert เมื่อค่าปัจจุบันเกิน N-Sigma | ปรับตัวตาม Pattern จริงอัตโนมัติ — แนวทางที่ใช้ใน Step 539 |
| **ซับซ้อนที่สุด: Machine Learning Model** | เรียนรู้ Pattern หลายมิติพร้อมกัน (เวลา, Metric หลายตัวพร้อมกัน, Correlation ข้าม Metric) | แนวทางที่ Cisco Crosswork/NAE ใช้ (Step 535) — แม่นยำสุดแต่ Setup ซับซ้อนสุด |

### Proactive vs Reactive Monitoring — ปรัชญาที่ Streaming Telemetry เปลี่ยนไป

```
Reactive Monitoring (แนวทางเดิมที่พึ่ง SNMP Polling + User Complaint)
┌────────────┐      ┌────────────┐      ┌────────────┐      ┌────────────┐
│  ปัญหาเกิด   │ ───► │ User ร้องเรียน│ ───► │ Engineer เริ่ม│ ───► │ ค้นหา Root   │
│  (ยังไม่รู้)  │      │ (ล่าช้าไปแล้ว)│      │ Investigate  │      │ Cause + แก้ │
└────────────┘      └────────────┘      └────────────┘      └────────────┘
     เวลาที่เสียไปก่อนแก้ได้ = สูงมาก (User Impact เกิดขึ้นแล้วเสมอ)

Proactive Monitoring (แนวทางใหม่ด้วย Streaming Telemetry + Baseline/Anomaly Detection)
┌────────────┐      ┌────────────┐      ┌────────────┐      ┌────────────┐
│  Metric เริ่ม│ ───► │ Anomaly     │ ───► │ Engineer/   │ ───► │ แก้ไขก่อน   │
│  เบี่ยงเบน   │      │ Detection   │      │ Automation   │      │ User สังเกต │
│  (Trend ชัด) │      │ แจ้งเตือนทันที│      │ ตอบสนอง      │      │ ได้เลย       │
└────────────┘      └────────────┘      └────────────┘      └────────────┘
     ตรวจจับตั้งแต่ "แนวโน้ม" ก่อนกลายเป็น "ปัญหาจริง" — เช่น Error Rate ค่อยๆเพิ่มขึ้น
     ต่อเนื่อง 3 วัน (ยังไม่ถึงจุด Fiber ขาดสมบูรณ์) ก็ Detect และเปลี่ยน Transceiver ได้ก่อน
```

> **นี่คือคำมั่นสัญญาหลักของ Network Assurance**: เปลี่ยนวิศวกรจากคนที่ "ดับไฟที่ลุกแล้ว" ไปเป็น
> คนที่ "เห็นควันก่อนไฟลุก" — ซึ่งเป็นไปไม่ได้เลยถ้าไม่มี Streaming Telemetry ที่ให้ข้อมูลละเอียด
> พอในระดับวินาที/นาที (SNMP Polling ทุก 5 นาทีจับ Trend แบบนี้ไม่ทันเวลาพอ)

---

## Step 538 — ออกแบบ Dashboard: Top 10 Metrics ที่ต้อง Monitor สำหรับ Lab นี้

### แนวคิดการเลือก Metric — ไม่ใช่ "เก็บทุกอย่างที่เก็บได้"

Dashboard ที่ดีไม่ใช่ Dashboard ที่แสดง Metric มากที่สุด — Grafana ต่อกับ TIG Stack (Step 535)
สามารถ Query Metric ได้เป็นร้อยตัว แต่ Engineer ที่เปิด Dashboard ตอนตี 2 เพื่อ Troubleshoot
ต้องการเห็น **10 อันดับที่บอกสุขภาพเครือข่ายได้เร็วที่สุด** ก่อนอื่น ตารางด้านล่างคือ Top 10 Metric
ที่ควรอยู่หน้าแรกของ Dashboard สำหรับ Lab Topology ของหลักสูตรนี้โดยเฉพาะ

| # | Metric | อุปกรณ์/Layer | เหตุผลที่ต้อง Monitor | อ้างอิง Part ที่เกี่ยวข้อง |
|---|---|---|---|---|
| 1 | **OSPF Neighbor Count** (ต่อ Interface Backbone) | CORE-SW1/2, DIST-SW1-4 | ค่าที่ลดลงจาก Baseline = สูญเสีย Redundancy ทันที (แม้ Traffic ยังผ่านได้ผ่าน Path อื่น) | [Part 11-12](part-011-ospfv2-fundamentals.md), [Part 33](part-033-advanced-stp-campus-design.md) (ECMP) |
| 2 | **HSRP State + State Change Count** | DIST-SW1-4 (VIP ทุก VLAN) | State Change บ่อย = Physical Link หรือ Timer ไม่แน่นอน ผู้ใช้ปลายทางจะรู้สึกกระตุกทุกครั้งที่ Failover | [Part 18](part-018-fhrp.md), [Part 35](part-035-advanced-fhrp-campus-ha.md) |
| 3 | **Interface Error Rate** (`in-errors`, `out-errors`, CRC) | ทุก Backbone Link (Core↔Dist) | Error ที่เพิ่มขึ้นเรื่อยๆ คือสัญญาณเริ่มต้นของ Fiber/Transceiver เสื่อม ก่อนที่ Link จะ Down จริง | [Part 1 Step 6](part-001-networking-fundamentals.md), [Part 33](part-033-advanced-stp-campus-design.md) (UDLD) |
| 4 | **CPU Utilization (5-second avg)** | ทุกอุปกรณ์ Core/Dist | CPU สูงต่อเนื่องผิดปกติ = อาจมี Routing Loop, Log Flooding, หรือ Attack (CoPP ที่ [Part 46](part-046-control-plane-policing.md) ควรกันไว้ แต่ยัง Monitor เผื่อพลาด) | [Part 46](part-046-control-plane-policing.md) |
| 5 | **Memory Utilization / Free Memory** | ทุกอุปกรณ์ Core/Dist | Memory Leak สะสม (พบได้ใน IOS Bug บางเวอร์ชัน) นำไปสู่ Crash ถ้าไม่จับก่อน | — |
| 6 | **Interface Bandwidth Utilization (%)** | ทุก Uplink/Backbone | ใกล้ Saturate = ต้อง Capacity Plan เพิ่ม Link ก่อนกระทบ User (Proactive ตาม Step 537) | [Part 6](part-006-etherchannel.md) (EtherChannel Load-balance) |
| 7 | **Port-channel Member Link Status** | CORE-SW1↔CORE-SW2 (Po1), DIST Uplink | สมาชิกหลุดไป 1 เส้นจาก LACP ยัง "Up" ได้ (Bandwidth ลดลงเงียบๆ) ต้องรู้ก่อนสมาชิกที่เหลือ Saturate | [Part 6](part-006-etherchannel.md) |
| 8 | **STP Topology Change Count** | ACCESS-SW1-4, DIST-SW1-4 | เปลี่ยนถี่ผิดปกติ = Loop เสี่ยงเกิด หรือ Port Flap ที่ไม่ควรกระทบ STP เลยถ้า Design ถูกต้อง | [Part 5](part-005-stp-rstp-mstp.md), [Part 33](part-033-advanced-stp-campus-design.md) |
| 9 | **UDLD Port State** (Bidirectional/Unidirectional/Err-Disable) | ทุก Fiber Backbone Port | Unidirectional ที่ยังไม่ Err-Disable คือ "กำลังจะเกิดปัญหา" — Metric นี้ควรเป็น Leading Indicator ก่อน Link Down จริง | [Part 33 Step 328](part-033-advanced-stp-campus-design.md) |
| 10 | **NTP Sync Status** (`Clock is synchronized` / `unsynchronized`) | ทุกอุปกรณ์ | ถ้า NTP หลุด Sync จะทำให้ Timestamp ของ Metric อื่นทั้งหมดใน Dashboard **เชื่อถือไม่ได้เลย** — ต้องเป็น "Metric ของ Metric" ที่ Monitor ควบคู่ไปเสมอ | [Part 17](part-017-ntp-syslog-snmp.md) |

### เหตุผลที่ Metric #10 (NTP) สำคัญเป็นพิเศษ

สังเกตว่า Metric #10 ไม่ใช่ Metric เชิง Performance เหมือน 9 อันดับแรก แต่เป็น **Meta-Metric**
— ถ้า NTP ของอุปกรณ์ตัวใดตัวหนึ่งหลุด Sync การ Correlate Timeline ของ Event ข้ามอุปกรณ์
(เช่น Step 540 ที่จะเชื่อม UDLD Event กับ OSPF Neighbor Flap) จะ**ผิดพลาดทั้งหมด** — นี่คือ
เหตุผลที่ Dashboard ที่ดีต้อง Monitor "ความน่าเชื่อถือของข้อมูล" ควบคู่กับ "ข้อมูลจริง" เสมอ
ตรงตามหลักการที่ [Part 17 Step 161](part-017-ntp-syslog-snmp.md) วางไว้ตั้งแต่ต้น

### ตัวอย่างการจัด Layout Dashboard (แนวคิด Grafana Panel)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Row 1: Health Summary (Single-Stat Panel สีเขียว/เหลือง/แดง)         │
│  [OSPF Nbr: 8/8 OK]  [HSRP Flap 24h: 0]  [NTP Sync: 6/6 OK]           │
├─────────────────────────────────────────────────────────────────────┤
│  Row 2: Time-series Graph (Metric #3, #4, #6 — แสดงเป็น Trend Line)   │
│  [Interface Errors over time]  [CPU % over time]  [Bandwidth % ]      │
├─────────────────────────────────────────────────────────────────────┤
│  Row 3: Table/Log Panel (Metric #7, #8, #9 — แสดงเป็นรายการ Event)     │
│  [Port-channel member status]  [STP TC count]  [UDLD state per port]  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 539 — เชื่อม Telemetry กับ Automation จาก Part 47

### แนวคิด: Telemetry บอก "อะไรผิดปกติ" — Automation ตอบว่า "แล้วทำอะไรต่อ"

[Part 47](part-047-automation-python-advanced.md) สอน Configuration Compliance Checking
(Step 465) และ Error Handling/Retry Logic (Step 466) ไว้แล้ว — Step นี้จะเชื่อมสองโลกเข้าด้วยกัน:
ใช้ Telemetry Sample ที่ได้จาก Step 536 มาคำนวณหา Anomaly (ตามหลักการ Step 537) แบบ Real-time
แล้ว**Trigger Automation Script โดยอัตโนมัติ**ทันทีที่พบ — ไม่ต้องรอ Human มาเห็น Dashboard ก่อน

### Script เต็ม: `anomaly_detector_remediation.py`

```python
#!/usr/bin/env python3
"""
anomaly_detector_remediation.py
รับ Telemetry Sample ของ Interface Statistics แบบต่อเนื่อง (จำลองมาจาก mdt_grpc_receiver.py
ใน Step 536), คำนวณ Error Rate เทียบกับ Baseline (แนวทาง Statistical Baseline จาก Step 537),
และถ้าพบ Anomaly จะ Trigger สองอย่างพร้อมกัน:
  1. รัน Compliance/Diagnostic Check ผ่าน Netmiko (แนวทางเดียวกับ Part 47 Step 465)
  2. ส่ง Alert ไปยัง Slack (Stub ที่ใช้ requests เรียก Incoming Webhook)
"""

import os
import sys
import time
import statistics
from collections import deque
from netmiko import ConnectHandler
import requests

# --- ค่า Config ---
ERROR_RATE_HISTORY_WINDOW = 20     # เก็บ Sample ย้อนหลัง 20 ครั้งไว้คำนวณ Baseline
ANOMALY_SIGMA_THRESHOLD = 3        # Alert ถ้าค่าปัจจุบันเกิน Mean + 3 Standard Deviation
SLACK_WEBHOOK_URL = os.environ.get("SLACK_WEBHOOK_URL")   # ต้อง export ก่อนใช้งานจริง

# --- เก็บ History แยกตาม Interface (Key = interface name) ---
error_rate_history = {}


def compute_error_rate(current_errors: int, previous_errors: int, interval_sec: int) -> float:
    """คำนวณ Error Rate ต่อวินาที จากผลต่าง Counter (Counter เป็นค่าสะสม/cumulative)"""
    delta = current_errors - previous_errors
    if delta < 0:
        delta = current_errors   # Counter Reset (เช่น อุปกรณ์ Reload) — เริ่มนับใหม่
    return delta / interval_sec


def is_anomaly(interface: str, current_rate: float) -> bool:
    """
    ตรวจสอบว่า Error Rate ปัจจุบันเป็น Anomaly หรือไม่ ด้วยวิธี Statistical Baseline
    (Mean + N-Sigma) ตามหลักการที่อธิบายใน Part 54 Step 537
    """
    history = error_rate_history.setdefault(interface, deque(maxlen=ERROR_RATE_HISTORY_WINDOW))

    if len(history) < 5:          # ข้อมูลยังไม่พอคำนวณ Baseline ที่น่าเชื่อถือ
        history.append(current_rate)
        return False

    mean = statistics.mean(history)
    stdev = statistics.stdev(history) if len(history) > 1 else 0.0
    threshold = mean + (ANOMALY_SIGMA_THRESHOLD * stdev)

    history.append(current_rate)   # อัปเดต History ด้วยค่าปัจจุบันเสมอ (Sliding Window)

    is_spike = current_rate > threshold and current_rate > 0.5   # กัน False Positive จาก Noise เล็กๆ
    if is_spike:
        print(f"    [ANOMALY] {interface}: rate={current_rate:.2f}/s "
              f"(baseline mean={mean:.2f}, threshold={threshold:.2f})")
    return is_spike


def run_diagnostic_check(interface: str) -> str:
    """
    ขั้นที่ 1 ของ Remediation: SSH เข้าอุปกรณ์เพื่อดึงสถานะ Interface โดยละเอียด
    (แนวทางเดียวกับ Configuration Compliance Check ของ Part 47 Step 465)
    """
    device = {
        "device_type": "cisco_ios",
        "host": "10.10.99.10",         # CORE-SW1 ตาม docs/00-ip-address-plan.md
        "username": os.environ.get("NET_USERNAME", "admin"),
        "password": os.environ.get("NET_PASSWORD"),
        "secret": os.environ.get("NET_SECRET"),
    }
    try:
        conn = ConnectHandler(**device)
        conn.enable()
        output = conn.send_command(f"show interfaces {interface} | include error|CRC|input rate")
        conn.disconnect()
        return output
    except Exception as exc:
        return f"[!] Diagnostic check ล้มเหลว: {exc}"


def send_slack_alert(interface: str, current_rate: float, diagnostic_output: str):
    """ขั้นที่ 2 ของ Remediation: ส่ง Alert ไปยัง Slack ผ่าน Incoming Webhook (Stub)"""
    message = {
        "text": (
            f":rotating_light: *Anomaly Detected* — Interface Error Rate Spike\n"
            f"*Device:* CORE-SW1\n"
            f"*Interface:* {interface}\n"
            f"*Current Rate:* {current_rate:.2f} errors/sec\n"
            f"*Diagnostic Snapshot:*\n```{diagnostic_output.strip()}```"
        )
    }
    if not SLACK_WEBHOOK_URL:
        print("    [!] SLACK_WEBHOOK_URL ไม่ได้ตั้งค่า — พิมพ์ Alert ออก Console แทน:")
        print(f"    {message['text']}")
        return
    try:
        requests.post(SLACK_WEBHOOK_URL, json=message, timeout=5)
        print("    [OK] ส่ง Slack Alert สำเร็จ")
    except requests.exceptions.RequestException as exc:
        print(f"    [!] ส่ง Slack Alert ล้มเหลว: {exc}")


def handle_telemetry_sample(interface: str, current_errors: int, previous_errors: int, interval_sec: int):
    """เรียกจาก Pipeline หลัก (เช่น yield_downstream() ใน mdt_grpc_receiver.py ของ Step 536)"""
    rate = compute_error_rate(current_errors, previous_errors, interval_sec)

    if is_anomaly(interface, rate):
        print(f"[*] เริ่ม Remediation Workflow สำหรับ {interface} ...")
        diagnostic = run_diagnostic_check(interface)
        send_slack_alert(interface, rate, diagnostic)
    else:
        print(f"    {interface}: rate={rate:.2f}/s (ปกติ)")


# --- ตัวอย่างการจำลอง Sample ต่อเนื่อง (แทนการต่อ mdt_grpc_receiver.py จริง) ---
if __name__ == "__main__":
    print("[*] จำลอง Telemetry Sample ต่อเนื่องสำหรับ TenGigabitEthernet1/0/1 ...")
    simulated_errors = [0, 0, 1, 0, 1, 0, 2, 1, 450, 610]   # ค่า Cumulative Error Counter
    prev = 0
    for sample in simulated_errors:
        handle_telemetry_sample("TenGigabitEthernet1/0/1", sample, prev, interval_sec=30)
        prev = sample
        time.sleep(0.2)
```

### ตัวอย่าง Output ที่คาดหวัง (สังเกตจุดที่ Error พุ่งขึ้นแบบผิดปกติ)

```
[*] จำลอง Telemetry Sample ต่อเนื่องสำหรับ TenGigabitEthernet1/0/1 ...
    TenGigabitEthernet1/0/1: rate=0.00/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.00/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.03/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.00/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.03/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.00/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.07/s (ปกติ)
    TenGigabitEthernet1/0/1: rate=0.03/s (ปกติ)
    [ANOMALY] TenGigabitEthernet1/0/1: rate=14.97/s (baseline mean=0.02, threshold=0.09)
[*] เริ่ม Remediation Workflow สำหรับ TenGigabitEthernet1/0/1 ...
    [!] SLACK_WEBHOOK_URL ไม่ได้ตั้งค่า — พิมพ์ Alert ออก Console แทน:
    :rotating_light: *Anomaly Detected* — Interface Error Rate Spike
    *Device:* CORE-SW1
    *Interface:* TenGigabitEthernet1/0/1
    *Current Rate:* 14.97 errors/sec
    *Diagnostic Snapshot:*
```

> **จุดสำคัญของ Design นี้**: `handle_telemetry_sample()` ถูกออกแบบให้เรียกได้จากทั้ง Sample
> จำลอง (ตอนทดสอบ) และจาก `yield_downstream()` ใน `mdt_grpc_receiver.py` จริง (Step 536) —
> แยก Logic การ Detect Anomaly ออกจาก Logic การรับข้อมูล Telemetry ทำให้ Test ได้ง่ายและนำไปต่อ
> กับ Pipeline อื่น (เช่น Kafka Queue ระดับ Production) ได้โดยไม่ต้องเขียนใหม่ทั้งหมด

---

## Step 540 — Lab เต็มรูปแบบ: MDT ทุกอุปกรณ์ Core/Distribution + จำลอง Interface Flap

### เป้าหมายของ Lab

Deploy MDT (Interface Statistics + CPU + Memory) ให้ครบทุกอุปกรณ์ **CORE-SW1, CORE-SW2,
DIST-SW1, DIST-SW2, DIST-SW3, DIST-SW4** ไปยัง NOC-SRV (10.10.99.50) แล้วจำลองเหตุการณ์
**Interface Flap บน Backbone Link CORE-SW1 ↔ DIST-SW1** (Te1/0/1 ↔ Te1/1/1 ตาม
`00-ip-address-plan.md`) เพื่อดูว่า Telemetry Data ที่ Stream อยู่ตลอดเวลา "เล่าเรื่อง" อะไรได้
เมื่อเทียบกับ UDLD ([Part 33 Step 328](part-033-advanced-stp-campus-design.md)) และ OSPF
Neighbor Flap ที่เกิดขึ้นพร้อมกัน

### Config เต็มรูปแบบ — Template ที่ใช้ซ้ำได้กับทุกอุปกรณ์ (Source-address ต่างกันตาม Loopback0)

```
telemetry ietf destination-group DG-NOC-COLLECTOR
 destination-id 1
 destination ip address 10.10.99.50 port 57500 protocol grpc-tcp encoding encode-kvgpb
!
telemetry ietf sensor-group SG-INTERFACE
 sensor-path Cisco-IOS-XE-interfaces-oper:interfaces/interface
!
telemetry ietf sensor-group SG-CPU
 sensor-path Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization
!
telemetry ietf sensor-group SG-MEMORY
 sensor-path Cisco-IOS-XE-memory-oper:memory-statistics/memory-statistic
!
telemetry ietf subscription 100
 sensor-group-id SG-CPU sample-interval 10000
 destination-id DG-NOC-COLLECTOR
!
telemetry ietf subscription 101
 sensor-group-id SG-INTERFACE sample-interval 15000
 destination-id DG-NOC-COLLECTOR
!
telemetry ietf subscription 102
 sensor-group-id SG-MEMORY sample-interval 60000
 destination-id DG-NOC-COLLECTOR
```

> **หมายเหตุการเลือก `sample-interval`**: Interface Statistics ใช้ 15 วินาที (ต้องการเห็น
> Error/Flap ให้เร็วพอ), CPU ใช้ 10 วินาที (เปลี่ยนแปลงเร็ว), Memory ใช้ 60 วินาที (เปลี่ยนแปลง
> ช้ากว่ามาก ไม่จำเป็นต้อง Stream ถี่) — การเลือก Cadence ให้เหมาะกับธรรมชาติของแต่ละ Metric
> ช่วยลด Load บน Pipeline โดยไม่เสีย Visibility ที่จำเป็น

### `source-address` ต่อแต่ละอุปกรณ์ (เติมต่อท้าย Template ด้านบนในแต่ละ Subscription)

| อุปกรณ์ | Loopback0 (ใช้เป็น `source-address`) |
|---|---|
| CORE-SW1 | 1.1.1.1 |
| CORE-SW2 | 1.1.1.2 |
| DIST-SW1 | 1.1.1.11 |
| DIST-SW2 | 1.1.1.12 |
| DIST-SW3 | 1.1.1.13 |
| DIST-SW4 | 1.1.1.14 |

ตัวอย่างเติมให้ DIST-SW1 (ใช้ Template เดียวกันทั้งหมด เปลี่ยนแค่ `source-address` ในทุก
Subscription และ `hostname` ใน Prompt):

```
DIST-SW1(config)# telemetry ietf subscription 100
DIST-SW1(config-telemetry-subscription)# source-address 1.1.1.11
DIST-SW1(config-telemetry-subscription)# exit
DIST-SW1(config)# telemetry ietf subscription 101
DIST-SW1(config-telemetry-subscription)# source-address 1.1.1.11
DIST-SW1(config-telemetry-subscription)# exit
DIST-SW1(config)# telemetry ietf subscription 102
DIST-SW1(config-telemetry-subscription)# source-address 1.1.1.11
DIST-SW1(config-telemetry-subscription)# end
```

ทำแบบเดียวกันกับ **CORE-SW2, DIST-SW2, DIST-SW3, DIST-SW4** โดยใช้ Template เต็มรูปแบบ
(destination-group + sensor-group ×3 + subscription ×3) พร้อมเปลี่ยน `source-address` ตาม
ตารางด้านบน — Config Block ทั้งหมดเหมือนกัน 100% ยกเว้นค่านี้ค่าเดียว

### Verify ก่อนจำลอง Event — ยืนยันว่าทุกอุปกรณ์ Stream สำเร็จ

```
NOC-SRV$ tail -f /var/log/telegraf/telegraf.log | grep cisco_telemetry_mdt

2026-09-26T14:00:10Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.10.2
2026-09-26T14:00:10Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.0.1
2026-09-26T14:00:11Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.0.2
2026-09-26T14:00:11Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.20.2
2026-09-26T14:00:12Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.30.2
2026-09-26T14:00:12Z I! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from 10.255.40.2
```

เห็น Connection ครบ 6 เส้นจาก 6 อุปกรณ์ (ระบุด้วย Transit IP ตาม `00-ip-address-plan.md`) —
ตอนนี้ Dashboard บน Grafana ควรเริ่มมี Data Point ไหลเข้ามาต่อเนื่องทุก 10-60 วินาทีตามที่
Config ไว้ในแต่ละ Subscription

### จำลอง Event: Interface Flap บน CORE-SW1 ↔ DIST-SW1 (เชื่อมกับ Part 33 UDLD/OSPF)

จำลองเหตุการณ์ Fiber มีปัญหาแบบ Unidirectional บน Te1/0/1 (CORE-SW1) — เพื่อดูว่า UDLD Aggressive
([Part 33 Step 328](part-033-advanced-stp-campus-design.md)) ตรวจจับได้และ Err-Disable Port
ก่อน Loop จะเกิด พร้อมดูว่า OSPF Neighbor ([Part 11-12](part-011-ospfv2-fundamentals.md))
Flap ตามไปด้วย — และดูว่า Telemetry Timeline บอกเรื่องราวนี้ได้ตรงกับ Log จริงหรือไม่

**Timeline เหตุการณ์ (ทุก Timestamp Sync กันเพราะ NTP — Part 17 Step 161-163):**

| Timestamp (ICT) | เหตุการณ์ | แหล่งข้อมูล |
|---|---|---|
| 14:22:00.000 | Interface Error Rate บน Te1/0/1 (CORE-SW1) เริ่มเพิ่มขึ้นจาก 0/s เป็น ~8/s | **Telemetry** (SG-INTERFACE, Subscription 101 — เห็นก่อนใครเพราะ Cadence 15 วิ) |
| 14:22:03.112 | `%UDLD-4-UDLD_PORT_DISABLED: UDLD disabled interface TenGigabitEthernet1/0/1` | **Syslog** (ส่งไปยัง NOC-SRV ตาม [Part 17 Step 165](part-017-ntp-syslog-snmp.md)) |
| 14:22:03.340 | UDLD Port State: Bidirectional → Err-Disable | **Telemetry** (Sensor-Path เดียวกัน, Sample ถัดไปยืนยัน State ใหม่) |
| 14:22:04.891 | `%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.11 on TenGigabitEthernet1/0/1 from FULL to DOWN` | **Syslog** — DIST-SW1 มองไม่เห็น CORE-SW1 อีกต่อไปบน Link นี้ |
| 14:22:04.920 | OSPF Neighbor Count บน CORE-SW1 ลดลงจาก 4 เป็น 3 | **Telemetry** — Metric #1 จาก Top 10 (Step 538) เปลี่ยนค่าทันที |
| 14:22:05.200 | Traffic Reroute ผ่าน Path สำรอง (ECMP/OSPF Cost คำนวณใหม่) | **Telemetry** — Bandwidth Utilization ของ Link สำรองเพิ่มขึ้นตามที่คาด |
| 14:23:15.750 | Engineer เปลี่ยน Transceiver, `no shutdown` ที่ Te1/0/1 | Manual (จากผลของ UDLD Err-Disable ที่ต้องเข้าไป Reset Port เอง — Part 33 Step 328) |
| 14:23:22.001 | `%OSPF-5-ADJCHG: ... from LOADING to FULL, Loading Done` | **Syslog** — Neighbor กลับมาเป็น FULL |
| 14:23:22.400 | OSPF Neighbor Count กลับเป็น 4, Error Rate กลับเป็น 0/s | **Telemetry** — Metric กลับสู่ Baseline |

### สิ่งที่ Timeline นี้พิสูจน์ — ทำไม Telemetry ให้คุณค่ามากกว่า Log อย่างเดียว

1. **Telemetry เห็นก่อน Syslog เสมอในกรณีนี้** — Error Rate เริ่มขึ้นตั้งแต่ 14:22:00.000
   (3 วินาทีก่อน UDLD ถึงจะประกาศ Err-Disable) เพราะ UDLD ต้องรอ Echo Packet Timeout ตาม
   Algorithm ของมันก่อน แต่ Interface Counter สะท้อนปัญหาทันทีที่เริ่มมี Error จริง — นี่คือ
   ตัวอย่างจริงของ **Proactive Monitoring** ที่ Step 537 อธิบายไว้ (เห็น Trend ก่อนเห็น Event)
2. **Correlation ข้าม Metric ทำได้เพราะ Timestamp ตรงกัน** — Error Rate, UDLD State, OSPF
   Neighbor Count, Bandwidth Utilization ทั้งหมดมี Timestamp ที่ตรงกันแม่นยำถึงระดับ
   Millisecond เพราะทุกอุปกรณ์ Sync NTP จาก NOC-SRV ตัวเดียวกัน ([Part 17](part-017-ntp-syslog-snmp.md))
   — ถ้า NTP ไม่ตรง Timeline นี้จะเรียงลำดับผิดและตีความเหตุการณ์ผิดทันที (ตรงตาม Metric #10
   ของ Step 538)
3. **Dashboard "เล่าเรื่อง" ได้ครบโดยไม่ต้อง Login เข้าอุปกรณ์เลย** — Engineer เห็น Panel
   ที่แสดง Error Rate Spike → UDLD State Change → OSPF Neighbor Drop → Reroute เรียงกันบน
   กราฟเดียว โดยไม่ต้อง SSH เข้า CORE-SW1 หรือ DIST-SW1 แม้แต่ครั้งเดียวจนกว่าจะถึงขั้นตอนแก้ไข
   Physical Layer จริง (เปลี่ยน Transceiver)

### คำอธิบาย Dashboard View ระหว่างเหตุการณ์ (แนวคิด Grafana Panel)

```
Panel: "Backbone Link CORE-SW1 ↔ DIST-SW1 Health" (Time-series, ช่วง 14:21:30 - 14:24:00)

Error Rate (errors/sec)
  15 ┤                    ╭─╮
  10 ┤                  ╭─╯ ╰─╮
   5 ┤                ╭─╯     ╰─╮
   0 ┤━━━━━━━━━━━━━━━━╯         ╰━━━━━━━━━━━━━━━━━━━━━━━━━━
     14:21:30      14:22:00      14:22:30      14:23:00      14:23:30

OSPF Neighbor Count (CORE-SW1, ทุก Interface รวม)
   4 ┤━━━━━━━━━━━━━━━━━━╮                                ╭━━━━━━━━
   3 ┤                  ╰━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╯
     14:21:30      14:22:00      14:22:30      14:23:00      14:23:30
                    ▲ 14:22:04.920                        ▲ 14:23:22.400
                    Neighbor Down (Annotation จาก Syslog) Neighbor Full (Annotation)

Backup Path Utilization (Link สำรองที่รับ Traffic ระหว่าง Failover)
  40%┤                  ╭──────────────────────────────╮
  10%┤━━━━━━━━━━━━━━━━━━╯                                ╰━━━━━━━━
     14:21:30      14:22:00      14:22:30      14:23:00      14:23:30
```

Annotation (เส้นแนวตั้งพร้อม Label) ที่ Grafana ดึงมาจาก Syslog Event โดยอัตโนมัติ (ผ่าน
Telegraf `syslog` Input Plugin แยกอีกตัวที่รันคู่กับ `cisco_telemetry_mdt` บน NOC-SRV เดียวกัน)
ช่วยให้ Engineer เห็นจุดที่ Log Event เกิดขึ้นวางซ้อนบน Graph ของ Telemetry Metric ได้ในหน้าจอ
เดียว — นี่คือภาพรวมของ **Network Assurance ที่ครบวงจร**: Telemetry (Trend/Metric) + Syslog
(Discrete Event) + NTP (Timeline ที่เชื่อถือได้) ประกอบกันเป็น Visibility เดียวที่สมบูรณ์

---

## แบบฝึกหัดทวนความเข้าใจ Part 54

1. อธิบายข้อจำกัดหลัก 2 ข้อของ SNMP Polling ที่ทำให้ต้องมี Streaming Telemetry
2. Dial-in และ Dial-out ต่างกันอย่างไร และทำไม Production ส่วนใหญ่เลือกใช้ Dial-out
3. `sensor-path`, `destination-group`, และ `subscription` ใน MDT Config แต่ละตัวทำหน้าที่อะไร
4. Baseline ต่างจาก Static Threshold อย่างไร และทำไม Baseline ต้องมาจากข้อมูลจริงของเครือข่ายนั้น
5. ในตัวอย่าง Interface Flap ของ Step 540 เพราะเหตุใด Telemetry จึงตรวจพบความผิดปกติได้ก่อน
   Syslog Message ของ UDLD

**เฉลย:**
1. (1) Fixed Polling Interval พลาด Transient Event ที่เกิดและหายเร็วกว่าคาบเวลา Poll
   (2) Scalability แย่ลงเป็นเส้นตรงเมื่อต้องการ Poll ถี่ขึ้นหรือ Poll อุปกรณ์/OID จำนวนมาก
2. Dial-in = Collector เชื่อมต่อเข้าไปที่อุปกรณ์ (อุปกรณ์รัน gRPC Server); Dial-out = อุปกรณ์
   เชื่อมต่อออกไปที่ Collector เอง — Production เลือก Dial-out เพราะอุปกรณ์เป็นฝ่าย Initiate
   Connection ออก (Outbound) เหมือน Syslog/NTP Client ทำให้ไม่ต้องเปิด Inbound Port ที่ Trust
   Boundary ต่ำกว่า
3. `sensor-path` = YANG Path ที่จะ Stream ข้อมูล (อะไร); `destination-group` = ปลายทาง Collector
   (ที่ไหน); `subscription` = ผูก sensor-group เข้ากับ destination-group พร้อม Cadence (เมื่อไหร่
   /บ่อยแค่ไหน)
4. Static Threshold เป็นค่าคงที่ตายตัวที่ไม่คำนึงถึง Pattern การใช้งานจริง ทำให้เกิด False
   Positive/Negative ได้ง่าย ส่วน Baseline คำนวณจากข้อมูลจริงของเครือข่ายนั้นเป็นระยะเวลาหนึ่ง
   ทำให้รู้จัก Pattern (เช่น Peak Hour) และ Detect ความเบี่ยงเบนจริงได้แม่นยำกว่า
5. เพราะ Interface Error Counter สะท้อนปัญหาทันทีที่ Error เกิดขึ้นจริง ในขณะที่ UDLD ต้องรอ
   ตาม Algorithm ของมัน (รอ Echo Packet Timeout ตาม Aggressive/Normal Mode) ก่อนจะประกาศ
   Err-Disable และสร้าง Syslog Message — Telemetry จึงเป็น Leading Indicator ที่เร็วกว่า

---

## สรุป Part 54

Part นี้ปิดช่องว่างที่ [Part 49 Step 489](part-049-netconf-restconf-yang-deep-dive.md) เปิดไว้
เต็มรูปแบบ: เราเข้าใจข้อจำกัดของ SNMP Polling ที่ผลักดันให้เกิด Model-Driven Telemetry, เจาะลึก
gRPC/Protobuf เป็น Transport มาตรฐาน, Config MDT จริงบน IOS-XE ด้วย destination-group/
sensor-group/subscription, ต่อ Pipeline ไปยัง TIG Stack หรือ Cisco Crosswork, เขียน Python
gRPC Receiver และ Anomaly-Triggered Remediation ที่เชื่อมกับทักษะ Automation จาก
[Part 47](part-047-automation-python-advanced.md) และปิดท้ายด้วย Lab ที่พิสูจน์ว่า Telemetry
Timeline สามารถ "เล่าเรื่อง" เหตุการณ์ Interface Flap ได้ครบถ้วนกว่า Log อย่างเดียว — ทักษะ
Network Assurance นี้คือรากฐานเดียวกันที่ขับเคลื่อน Assurance Dashboard ของ Cisco DNA Center
ที่กล่าวถึงใน [Part 50](part-050-dna-center.md)

✅ **พร้อมสำหรับ Part 55**: เราจะนำทุกทักษะจากทั้ง CCNP ENCOR (Part 26-54) มารวมกันใน
**ENCOR Capstone Lab** — ออกแบบ, Config, Automate, และ Monitor เครือข่ายเดียวกันตั้งแต่ต้นจนจบ
ด้วยเทคนิคทั้งหมดที่เรียนมา รวมถึง Streaming Telemetry ที่เพิ่งเรียนใน Part นี้ด้วย

**ไปต่อ:** [Part 55 — ENCOR Capstone Lab →](part-055-encor-capstone-lab.md)
