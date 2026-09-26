# Part 82 — End-to-End Enterprise QoS Design
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 811–820 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 81 — High Availability: NSF/SSO/ISSU](part-081-high-availability-nsf-sso-issu.md)
> ที่สอนวิธีทำให้ **Control Plane และ Data Plane ของอุปกรณ์แต่ละตัว** รอดจาก Failover โดยไม่มี
> Traffic หลุด (NSF/SSO ระหว่าง Supervisor, ISSU สำหรับ Upgrade IOS โดยไม่ต้อง Reboot) — Part 81
> ตอบคำถามว่า **"อุปกรณ์ตัวหนึ่งจะพังแล้วยัง Forward Traffic ต่อได้อย่างไร"** แต่ยังไม่ได้ตอบคำถาม
> ที่สำคัญไม่แพ้กัน: **"Traffic ที่ Forward ต่อได้นั้น มีคุณภาพ (Quality) ดีพอสำหรับ Voice/Video/
> Business-Critical Application หรือไม่ ตลอดทั้งเส้นทางจริง"** — HA ทำให้ Path ไม่ขาด แต่ QoS
> ต่างหากที่ทำให้ Path นั้น **"ใช้งานได้ดี"** เมื่อเกิด Congestion — Part 82 นี้จะยกระดับ QoS ขึ้นไป
> อีกขั้นจาก [Part 24](part-024-qos-fundamentals.md) (Trust Boundary ระดับ Access) และ
> [Part 40](part-040-advanced-qos.md) (MQC เต็มรูปแบบบน**อุปกรณ์ตัวเดียว** — WAN-EDGE-1) และ
> [Part 68](part-068-qos-troubleshooting.md) (Troubleshoot ปัญหา QoS ที่เกิดจาก Config ผิดจุดเดียว)
> ไปสู่คำถามระดับ **CCIE Design**: **"จะออกแบบ QoS Policy ที่สอดคล้องกัน (Consistent) ตลอดทั้ง
> เครือข่ายองค์กร — ตั้งแต่ IP Phone/Server ต้นทาง ผ่าน Access → Distribution → Core → WAN Edge
> → สาขาปลายทาง, รวม Wireless, SD-WAN, Data Center Edge, และ Cloud/SaaS — ได้อย่างไร"**
> Part นี้ไม่ได้สอนคำสั่งใหม่มากมาย (คำสั่ง MQC ทั้งหมดสอนไปแล้วใน Part 40) แต่สอน **"จะประกอบ
> คำสั่งที่มีอยู่แล้วให้เป็นสถาปัตยกรรมเดียวที่ทำงานร่วมกันทั้งองค์กร"** ได้อย่างไร

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 811 | ปรัชญาการออกแบบ QoS แบบ End-to-End — Weakest Link และความสอดคล้องกันทุก Hop |
| 812 | Enterprise QoS Baseline — โมเดล 12 Class มาตรฐานของ Cisco พร้อมตาราง DSCP เต็มรูปแบบ |
| 813 | Campus QoS Design — Trust/Mark/Queue ที่ Access, Distribution, Core |
| 814 | WAN Edge QoS Design — LLQ/CBWFQ/HQoS เต็มรูปแบบอ้างอิง Enterprise Baseline |
| 815 | Wireless QoS Integration — แม็บ 802.11e/WMM เข้ากับ DSCP บน Wired |
| 816 | SD-WAN QoS Integration — Application-Aware Routing กับ MQC แบบเดิมทำงานร่วมกันอย่างไร |
| 817 | Data Center Edge QoS — DSCP Preservation ข้าม VXLAN/EVPN Boundary (Preview Part 84-85) |
| 818 | QoS สำหรับ Cloud/SaaS Traffic — ขอบเขตที่ควบคุมไม่ได้ และวิธีบรรเทา |
| 819 | QoS SLA Monitoring & Validation — ยืนยันผลลัพธ์จริงด้วย Telemetry ต่อเนื่อง |
| 820 | Lab เต็มรูปแบบ: End-to-End QoS Design ทั้งองค์กร ACCESS→DIST/CORE→WAN→Wireless พร้อม SLA Dashboard |

---

## Step 811 — ปรัชญาการออกแบบ QoS แบบ End-to-End: Weakest Link

### กฎเหล็กข้อที่ 1 ของ QoS ระดับ Enterprise: "QoS ดีเท่ากับ Hop ที่แย่ที่สุดในเส้นทางเท่านั้น"

Part 24 สอน Trust Boundary ที่ Access Switch หนึ่งตัว, Part 40 สอน MQC เต็มรูปแบบบน WAN-EDGE-1
หนึ่งตัว — ทั้งสอง Part ทำให้เห็นภาพว่า "จุดหนึ่งจุด" ควร Config อย่างไร แต่ **Voice Call หนึ่งสาย
ไม่ได้วิ่งอยู่ที่จุดเดียว** มันวิ่งผ่านอุปกรณ์หลายตัวต่อเนื่องกัน — ถ้าแม้แต่ **หนึ่งตัว** ในเส้นทาง
ทั้งหมดไม่ได้ Trust/Mark/Queue อย่างถูกต้อง คุณภาพของ Call ทั้งสายจะเสียหายจากจุดนั้นจุดเดียว
ไม่ว่าอีก 10 อุปกรณ์ที่เหลือจะ Config สมบูรณ์แบบเพียงใดก็ตาม

```
IP Phone ──▶ ACCESS-SW1 ──▶ DIST-SW1 ──▶ CORE-SW1 ──▶ WAN-EDGE-1 ──▶ ISP ──▶ WAN-EDGE-2 ──▶ CORE-SW2 ──▶ DIST-SW3 ──▶ ACCESS-SW3 ──▶ IP Phone ปลายทาง
   [Mark EF]    [Trust]      [Trust]     [Trust]      [LLQ]                  [LLQ]        [Trust]      [Trust]      [Trust]         [รับสาย]

ถ้าทุก Hop ทำงานถูกต้อง:           Voice ได้ Priority Treatment ตลอดเส้นทาง → เสียงชัด ไม่สะดุด

ถ้า DIST-SW1 ตัวเดียว "no mls qos trust dscp"
(กลับไปเป็น Default = ไม่เชื่อ, Reset DSCP เป็น 0):
   IP Phone ──▶ ACCESS-SW1 ──▶ DIST-SW1 ──▶ CORE-SW1 ──▶ ... ──▶ ปลายทาง
      Mark EF      Trust        ✗ DSCP ถูก Reset      Trust (แต่ตอนนี้เห็น DSCP=0 แล้ว!)
                                 เป็น 0 ที่นี่!         → คิดว่าเป็น Best Effort ไปตลอดที่เหลือ
                                                        แม้ Hop ท้ายๆ Config ถูกทุกจุด

ผลลัพธ์: Voice Call สายนี้ได้ Best Effort Treatment ตั้งแต่ DIST-SW1 เป็นต้นไป
         ทั้งที่ ACCESS-SW1 และทุก Hop ถัดไปนอกจาก DIST-SW1 Config ถูกต้องหมด!
```

**บทเรียนสำคัญที่สุดของ Step นี้**: การออกแบบ QoS ที่ดีต้อง **มองเป็น "เส้นทางเต็ม" (End-to-End
Path) เสมอ ไม่ใช่มองอุปกรณ์แยกเป็นชิ้นๆ** — คำถามที่ต้องตอบได้ทุกครั้งก่อน Sign-off Design คือ
"ถ้า Voice Packet วิ่งจาก Source ถึง Destination ผ่านอุปกรณ์กี่ตัว แต่ละตัวจะปฏิบัติต่อ Packet
นี้อย่างไร และทุกตัวเห็น DSCP เดียวกันตลอดเส้นทางหรือไม่"

### 3 หลักการออกแบบ QoS End-to-End ที่ต้องยึดตลอดทั้งองค์กร

| หลักการ | ความหมาย | เหตุผล |
|---|---|---|
| **1. Classify/Mark ครั้งเดียว ใกล้ Source ที่สุด** | ทำที่ Endpoint (IP Phone, Video Endpoint mark เอง) หรือ Access Switch/AP ตัวแรกที่ Traffic เข้าเครือข่าย | ยิ่ง Mark ช้า/ไกลจาก Source อุปกรณ์ที่อยู่ก่อนหน้าต้องส่ง Traffic แบบ Best Effort ไปโดยไม่รู้ประเภท — เสียโอกาส QoS ไปเปล่าๆ ในช่วงนั้น |
| **2. Trust ทุก Hop ถัดไป — ห้าม Re-classify ซ้ำโดยไม่จำเป็น** | ตั้งแต่ Hop ที่ 2 เป็นต้นไป ใช้ `trust dscp`/`trust cos` เท่านั้น ไม่ต้อง `class-map`+`set dscp` ซ้ำถ้า Mark เดิมถูกต้องอยู่แล้ว | ตาม DiffServ Model (Part 24 Step 233) — Re-classify ทุก Hop กิน CPU/ASIC โดยไม่จำเป็น และเสี่ยง Mark ผิดถ้า Logic ไม่ตรงกับจุดแรก |
| **3. DSCP Semantics ต้องเป็นค่าเดียวกันทั้งองค์กร** | EF ต้องแปลว่า Voice **ทุกที่** ในองค์กร ไม่ใช่ EF=Voice ที่ตึก A แต่ EF=Video ที่ตึก B | ถ้าแต่ละไซต์ Mark ต่างกัน เมื่อ Traffic ข้ามไซต์ (ผ่าน WAN/SD-WAN) อุปกรณ์ปลายทางจะตีความ Class ผิด — นี่คือเหตุผลที่ Step 812 ต้องมี **Baseline เดียวที่ใช้ร่วมกันทั้งองค์กร** |

### ทำไม Part นี้ต้องมาหลัง Part 68 (QoS Troubleshooting) ไม่ใช่มาแทน

Part 68 สอนวิธี**จับปัญหา**ที่เกิดจาก Config ผิดจุดเดียว (Trust Boundary หายหลัง RMA, Policy-map
ผิดทิศทาง ฯลฯ) — Part 82 นี้สอนวิธี **ออกแบบ** ตั้งแต่ต้นไม่ให้เกิดปัญหาเหล่านั้น โดยมองภาพรวม
ทั้งองค์กรเป็นระบบเดียว: กำหนด **Baseline Class Model กลาง** (Step 812) แล้วประกาศว่า **แต่ละ
Tier ของเครือข่าย (Access/Distribution/Core/WAN/Wireless/SD-WAN/DC Edge/Cloud) ต้อง Config
อย่างไรให้สอดคล้องกับ Baseline นั้น** — นี่คือความแตกต่างระหว่าง "Troubleshooting แก้ปัญหาที่
เกิดแล้ว" กับ "Design ป้องกันปัญหาไม่ให้เกิด" ซึ่งเป็นทักษะที่ข้อสอบ **CCIE Enterprise
Infrastructure ทั้ง Written และ Lab (Design Module)** วัดโดยตรง

---

## Step 812 — Enterprise QoS Baseline: โมเดล 12 Class มาตรฐานของ Cisco

### ทำไมต้องมี "Baseline" กลาง

Part 24/40 ใช้ Class ง่ายๆ (VOICE/VIDEO/CRITICAL-DATA/BEST-EFFORT) เพียงพอสำหรับสอนกลไก MQC
แต่องค์กรจริงมี Traffic หลากหลายกว่านั้นมาก — Cisco จึงเผยแพร่ **Enterprise QoS Baseline**
(อ้างอิงจาก RFC 4594 "Configuration Guidelines for DiffServ Service Classes" และ Cisco Medianet/
Enterprise QoS SRND) เป็น **มาตรฐานกลาง 12 Class** ที่ใช้ตั้งชื่อ/ตั้ง DSCP ให้ตรงกันทุกอุปกรณ์
ทุกไซต์ในองค์กรเดียวกัน — ตั้งแต่ Step นี้เป็นต้นไป Lab ของเราจะอ้างอิง Baseline นี้เป็น
**แหล่งความจริงเดียว (Single Source of Truth) ของ QoS Marking** เทียบเท่ากับที่
[`00-ip-address-plan.md`](00-ip-address-plan.md) เป็นแหล่งความจริงของ IP Address

### ตาราง Enterprise QoS Baseline เต็มรูปแบบ (12 Class)

| # | Class Name | DSCP | ค่า Decimal | PHB | Traffic ตัวอย่าง | Bandwidth Guarantee ที่แนะนำ | ต้อง Policed? |
|---|---|---|---|---|---|---|---|
| 1 | **VoIP Telephony** | EF | 46 | Expedited Forwarding | RTP Voice Bearer (G.711/G.722) | LLQ ≤ 33% รวมกับ #2-3 | ใช่ (LLQ มี Policer ในตัว) |
| 2 | **Broadcast Video** | CS5 | 40 | Class Selector | IPTV, Digital Signage แบบ One-way | LLQ (Priority Level 2) | ใช่ |
| 3 | **Realtime Interactive** | CS4 | 32 | Class Selector | Telepresence, Interactive Video Control | LLQ (Priority Level 2) | ใช่ |
| 4 | **Multimedia Conferencing** | AF41/42/43 | 34/36/38 | Assured Forwarding | Webex/Teams Video Call (Two-way) | CBWFQ ~10% | WRED เท่านั้น |
| 5 | **Multimedia Streaming** | AF31/32/33 | 26/28/30 | Assured Forwarding | Video on Demand, E-learning | CBWFQ ~8-10% | WRED เท่านั้น |
| 6 | **Network Control** | CS6 | 48 | Class Selector | OSPF/EIGRP/BGP Hello, HSRP | CBWFQ ~2% | ไม่ (สำคัญเกินกว่าจะ Drop) |
| 7 | **Signaling** | CS3 | 24 | Class Selector | SIP/H.323/SCCP Call Setup (ไม่ใช่เสียง) | CBWFQ ~2% | ไม่ |
| 8 | **Network Management (OAM)** | CS2 | 16 | Class Selector | SNMP, SSH, NETCONF, Syslog, Telemetry (Part 54) | CBWFQ ~3% | ไม่ |
| 9 | **Transactional Data** | AF21/22/23 | 18/20/22 | Assured Forwarding | ERP, CRM, Database Query, Interactive App | CBWFQ ~10-15% | WRED เท่านั้น |
| 10 | **Bulk Data** | AF11/12/13 | 10/12/14 | Assured Forwarding | File Transfer, Email, Backup, Print | CBWFQ ~4-10% | WRED เท่านั้น |
| 11 | **Best Effort** | DF (CS0) | 0 | Default | Traffic ทั่วไปที่ไม่ได้ Mark อะไรเลย (Default ของทุกอย่าง) | ส่วนที่เหลือ (`class-default`, ≥ 25%) | WRED เท่านั้น |
| 12 | **Scavenger** | CS1 | 8 | Class Selector | P2P, Software Update ที่ไม่เร่งรีบ, Traffic ที่ไม่พึงประสงค์ | ≤ 1-5% (ต่ำกว่า Best Effort เสมอ) | ใช่/WRED เข้มงวด |

> **หมายเหตุสำคัญเรื่อง Bandwidth %**: ตัวเลขในตารางเป็น**แนวทางเริ่มต้น (Starting Point)** จาก
> Cisco Enterprise QoS SRND ไม่ใช่กฎตายตัว — ต้องปรับตาม Traffic Mix จริงขององค์กร (Protocol
> Discovery ด้วย NBAR จาก Part 40 Step 398 ช่วยตัดสินใจตัวเลขจริงได้) **กฎที่ตายตัวจริงๆ มีข้อ
> เดียว**: Class ที่เป็น LLQ (Realtime รวมกัน #1-3) **ไม่ควรเกิน 33% ของ Bandwidth Link** — เกิน
> กว่านี้จะเสี่ยง Starve Class อื่นทั้งหมดเมื่อ Realtime Traffic พุ่งสูง (แม้ LLQ จะมี Policer
> ในตัวตาม Part 40 Step 393 ก็ตาม)

### เมื่อ Hardware มี Queue จำกัด — จะ "ยุบ" 12 Class ให้พอกับ Queue จริงอย่างไร

Switch ระดับ Access ส่วนใหญ่มี Hardware Queue แค่ **4-8 Queue** ต่อ Port (ไม่ใช่ Software Queue
ไม่จำกัดแบบ MQC บน Router) — ต้อง**จัดกลุ่ม (Group)** 12 Class ให้เหลือเท่ากับจำนวน Queue จริง
โดยยึดหลัก **จัดกลุ่มเฉพาะ Class ที่มีความต้องการ QoS ใกล้เคียงกัน**:

```
12-Class Baseline (Software/MQC เต็มรูปแบบ — ใช้ได้ที่ WAN Edge Router)
        │
        │ ยุบรวมสำหรับ Hardware ที่มี Queue จำกัด (Access/Distribution Switch)
        ▼
8-Queue Model (ตัวอย่าง Catalyst ทั่วไป):
  Q1 (Priority)  : VoIP Telephony (EF)
  Q2 (Priority)  : Broadcast Video + Realtime Interactive (CS5+CS4)
  Q3             : Multimedia Conferencing + Streaming (AF4x+AF3x)
  Q4             : Network Control + Signaling (CS6+CS3)
  Q5             : Network Management (CS2)
  Q6             : Transactional Data (AF2x)
  Q7             : Bulk Data (AF1x)
  Q8 (ต่ำสุด)     : Best Effort (DF) + Scavenger (CS1) — คั่นด้วย WRED Threshold ต่างกัน
```

**หลักการยุบ Class ที่ถูกต้อง**: จับ Class ที่ **Sensitivity ต่อ Delay/Jitter ใกล้กัน** ไว้
Queue เดียวกัน (เช่น Broadcast Video กับ Realtime Interactive ทั้งคู่เป็น "Video แบบ Realtime")
**ห้าม**จับ Class ที่ Sensitivity ต่างกันมากไว้ Queue เดียวกันเด็ดขาด (เช่น ห้ามเอา VoIP ไปรวม
กับ Bulk Data แม้จะเหลือ Queue ไม่พอก็ตาม — ถ้า Queue ไม่พอจริงๆ ควร Drop Class ที่มีความสำคัญ
น้อยที่สุดออกจากการมี Queue เฉพาะไปเข้า Best Effort ร่วมแทน ไม่ใช่เอา VoIP ไปลดระดับ)

---

## Step 813 — Campus QoS Design: Trust/Mark/Queue ที่ Access, Distribution, Core

### หลักการที่ Part 24 Step 235 วางไว้ ขยายผลใช้ทั่วทั้ง Campus

```
                    Classify + Mark ครั้งเดียว              Trust + Queue ทุก Hop ถัดไป (ไม่ Re-mark)
┌──────────────┐  ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│ IP Phone      │──│ ACCESS-SW1    │───────▶│ DIST-SW1      │───────▶│ CORE-SW1      │
│ (Mark EF เอง) │  │ trust device  │ Trunk  │ trust dscp    │ Routed │ trust dscp    │
│               │  │ cisco-phone   │        │ + Queue Policy│ Uplink │ + Queue Policy│
└──────────────┘  └──────────────┘        └──────────────┘        └──────────────┘
     TRUST BOUNDARY (Part 24)              "แค่เชื่อและจัด Queue"    "แค่เชื่อและจัด Queue"
     จุดเดียวที่ทำ Classification จริง        ไม่มี class-map ใหม่      ไม่มี class-map ใหม่
```

### ตารางสรุป Configuration ต่อ Tier ของ Campus

| Tier | หน้าที่ QoS | คำสั่งหลัก | ทำ Re-mark ไหม |
|---|---|---|---|
| **Access (ACCESS-SW1-4)** | Classify + Mark ครั้งแรก, Trust Boundary | `mls qos trust device cisco-phone` (Part 24 Step 235), Auto-QoS หรือ MQC `class-map`+`set dscp` สำหรับ Endpoint อื่นที่ไม่ใช่ Cisco Phone | **ใช่ — ที่นี่เท่านั้น** |
| **Distribution (DIST-SW1-4)** | Trust สิ่งที่ Access Mark มา + Queue กัน Microburst ตอน Access หลายพอร์ตยิงเข้า Uplink เดียว | `mls qos trust dscp` บน Uplink/Downlink ทุกเส้น + Queuing Policy เบาๆ (ดูด้านล่าง) | ไม่ (Trust อย่างเดียว) |
| **Core (CORE-SW1-2)** | Trust + Queue เหมือน Distribution แต่ Bandwidth สูงกว่า ความเสี่ยง Congestion น้อยกว่า | `mls qos trust dscp` บนทุก Interface ระหว่าง Core↔Distribution/WAN Edge | ไม่ |

### ทำไม Distribution/Core ยังต้องมี "Queue Policy" ทั้งที่ Bandwidth เหลือเฟือ

หลายคนเข้าใจผิดว่า Distribution/Core "แค่ Trust พอ ไม่ต้อง Queue เลย เพราะ Bandwidth เยอะ" — ความ
จริงคือแม้ Uplink ระหว่าง Distribution↔Core จะเป็น 10G/40G แต่ **Access Switch หลายตัวรวมกัน
ยิง Traffic เข้า Uplink เดียวพร้อมกันได้ (Many-to-One Fan-in)** ทำให้เกิด **Microburst** ชั่วครู่
ที่เกิน Bandwidth Uplink ได้เสมอ แม้ Bandwidth เฉลี่ยจะยังเหลือมาก — ถ้าไม่มี Queue แยก Class
Microburst นั้นจะทำให้ Voice Packet บางตัวถูก Drop ไปพร้อมกับ Data ทั่วไปแบบสุ่ม (เหมือน FIFO)

```
DIST-SW1(config)# mls qos
DIST-SW1(config)# class-map match-any CAMPUS-REALTIME
DIST-SW1(config-cmap)#  match dscp ef
DIST-SW1(config-cmap)#  match dscp cs5 cs4
DIST-SW1(config-cmap)# exit
DIST-SW1(config)# class-map match-any CAMPUS-CRITICAL
DIST-SW1(config-cmap)#  match dscp af41 af42 af43 af31 af32 af33 cs6 cs3 cs2
DIST-SW1(config-cmap)# exit
!
DIST-SW1(config)# policy-map CAMPUS-UPLINK-QUEUE
DIST-SW1(config-pmap)# class CAMPUS-REALTIME
DIST-SW1(config-pmap-c)#  priority percent 25
DIST-SW1(config-pmap-c)# class CAMPUS-CRITICAL
DIST-SW1(config-pmap-c)#  bandwidth percent 30
DIST-SW1(config-pmap-c)# class class-default
DIST-SW1(config-pmap-c)#  fair-queue
DIST-SW1(config-pmap-c)# exit
!
DIST-SW1(config)# interface TenGigabitEthernet1/1/1        ! Uplink ไป CORE-SW1
DIST-SW1(config-if)#  mls qos trust dscp
DIST-SW1(config-if)#  service-policy output CAMPUS-UPLINK-QUEUE
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface GigabitEthernet1/0/24            ! Downlink ไป ACCESS-SW1
DIST-SW1(config-if)#  switchport mode trunk
DIST-SW1(config-if)#  switchport trunk allowed vlan 10,20,99
DIST-SW1(config-if)#  mls qos trust dscp
DIST-SW1(config-if)# exit
```

สังเกตว่า **Policy-map นี้ไม่มี `set dscp` เลยแม้แต่บรรทัดเดียว** — มีแต่ `match` (Classify
เพื่อจัด Queue เท่านั้น) กับ `priority`/`bandwidth` (Queue Treatment) — นี่คือความต่างสำคัญจาก
WAN Edge (Step 814) ที่ต้องทำ Shaping/Policing/WRED เข้มข้นกว่ามาก เพราะ Distribution/Core
ทำหน้าที่แค่ **"ป้องกัน Microburst"** ไม่ใช่ **"บริหาร Bandwidth ที่ขาดแคลนจริง"** แบบ WAN

### `show mls qos interface` ยืนยันว่า Trust ทำงานถูกต้องทุก Tier

```
DIST-SW1# show mls qos interface TenGigabitEthernet1/1/1
TenGigabitEthernet1/1/1
trust state: trust dscp
trust mode: trust dscp
COS override: dis
DSCP Mutation Map: Default DSCP Mutation Map
qos mode: port-based

CORE-SW1# show mls qos interface TenGigabitEthernet1/0/1
TenGigabitEthernet1/0/1
trust state: trust dscp
trust mode: trust dscp
qos mode: port-based
```

ทั้ง DIST-SW1 และ CORE-SW1 ต้องแสดง `trust state: trust dscp` เหมือนกันทุก Interface ที่อยู่ใน
เส้นทาง — ถ้ามี Interface ไหนใน Path แสดงค่าอื่น (`not trusted` หรือค่า default) นั่นคือจุดที่
ทำให้ Design "Weakest Link" ตาม Step 811 พังทันที ต้องตรวจสอบให้ครบทุก Hop ก่อน Sign-off

---

## Step 814 — WAN Edge QoS Design: LLQ/CBWFQ/HQoS เต็มรูปแบบอ้างอิง Enterprise Baseline

### ทำไม WAN Edge คือจุดที่ต้อง "ดุดัน" ที่สุดในทั้งเส้นทาง

Bandwidth ภายใน Campus (LAN) เป็น Gigabit/10-Gigabit ทุกจุด แต่ **WAN Circuit ที่ WAN-EDGE-1/2
เช่าจาก ISP มีราคาแพงและถูกจำกัดด้วยสัญญา CIR เสมอ** (ตามที่ Part 40 อธิบายไว้แล้ว) — นี่คือจุด
ที่ Congestion เกิดขึ้น**บ่อยและรุนแรงที่สุด**ในทั้งเส้นทาง Enterprise ทำให้ WAN Edge ต้องใช้
กลไก QoS **ครบทุกกลไก** (LLQ + CBWFQ + WRED + HQoS) พร้อมกัน ในขณะที่ Campus (Step 813) ใช้แค่
Trust + Queue เบาๆ ก็พอ

### ขยาย Policy-map ของ WAN-EDGE-1 จาก Part 40 (4 Class) ให้ครอบคลุม Enterprise Baseline (Step 812)

Part 40 Step 400 ใช้แค่ 4 Class (VOICE/VIDEO/CRITICAL-DATA/BEST-EFFORT) เพื่อสอนกลไก MQC — ตอนนี้
เราจะ **แทนที่** Policy-map เดิมด้วยเวอร์ชันที่ครอบคลุม Baseline เต็มรูปแบบ โดยยุบ 12 Class ให้
เหลือ 8 Class ตามหลักการ Step 812 (Realtime Sensitivity ใกล้เคียงกันไว้ Queue เดียวกัน):

```
WAN-EDGE-1(config)# class-map match-any VOICE
WAN-EDGE-1(config-cmap)#  match dscp ef
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any INTERACTIVE-VIDEO
WAN-EDGE-1(config-cmap)#  match dscp cs5
WAN-EDGE-1(config-cmap)#  match dscp cs4
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any MM-CONFERENCING
WAN-EDGE-1(config-cmap)#  match dscp af41 af42 af43
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any MM-STREAMING
WAN-EDGE-1(config-cmap)#  match dscp af31 af32 af33
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any NET-MGMT-CONTROL
WAN-EDGE-1(config-cmap)#  match dscp cs6
WAN-EDGE-1(config-cmap)#  match dscp cs3
WAN-EDGE-1(config-cmap)#  match dscp cs2
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any TRANSACTIONAL-DATA
WAN-EDGE-1(config-cmap)#  match dscp af21 af22 af23
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any BULK-DATA
WAN-EDGE-1(config-cmap)#  match dscp af11 af12 af13
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any SCAVENGER
WAN-EDGE-1(config-cmap)#  match dscp cs1
WAN-EDGE-1(config-cmap)# exit
```

### Policy-map เต็มรูปแบบ (Child — ทำงานภายในกรอบ Shape ของ Parent)

```
WAN-EDGE-1(config)# policy-map ENTERPRISE-WAN-QOS
WAN-EDGE-1(config-pmap)# class VOICE
WAN-EDGE-1(config-pmap-c)#  priority level 1 percent 10           ! LLQ ระดับ 1 — Strict สูงสุด
WAN-EDGE-1(config-pmap-c)# class INTERACTIVE-VIDEO
WAN-EDGE-1(config-pmap-c)#  priority level 2 percent 20           ! LLQ ระดับ 2 — Strict รองจาก Voice
WAN-EDGE-1(config-pmap-c)# class MM-CONFERENCING
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 10
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# class NET-MGMT-CONTROL
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 5
WAN-EDGE-1(config-pmap-c)# class MM-STREAMING
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 8
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# class TRANSACTIONAL-DATA
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 15
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# class BULK-DATA
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 10
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# class SCAVENGER
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 2
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)#  random-detect dscp 8 10 20 10          ! Threshold ต่ำสุดในทั้ง Policy — Drop ก่อนใครเสมอ
WAN-EDGE-1(config-pmap-c)# class class-default
WAN-EDGE-1(config-pmap-c)#  fair-queue
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# exit
!
WAN-EDGE-1(config)# policy-map WAN-SHAPE-50M
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  shape average 50000000
WAN-EDGE-1(config-pmap-c)#  service-policy ENTERPRISE-WAN-QOS
WAN-EDGE-1(config-pmap-c)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)#  service-policy output WAN-SHAPE-50M
```

### ตรวจสอบผลรวม % ตามกฎ Step 812 (Realtime ≤ 33%, non-priority ≤ 75%)

| กลุ่ม | Class | % | ผลรวมกลุ่ม |
|---|---|---|---|
| LLQ (Priority — ไม่นับใน 75% Cap) | VOICE (10%) + INTERACTIVE-VIDEO (20%) | 30% | **≤ 33%** ✅ ตามกฎ Realtime |
| CBWFQ (`bandwidth percent` — นับใน 75% Cap) | MM-CONF(10) + NET-MGMT(5) + MM-STREAM(8) + TRANS(15) + BULK(10) + SCAV(2) | **50%** | ≤ 75% ✅ เหลือ Margin ให้ `class-default` (~20%) |

**WAN-EDGE-2 Config เหมือนกันทุกประการ** (Interface และ IP ต่างกันตาม
[`00-ip-address-plan.md`](00-ip-address-plan.md) — `GigabitEthernet0/0/0` ไป ISP-RTR ที่
`203.0.113.6/30`) เพื่อให้ Traffic ที่ Failover ไปใช้ WAN-EDGE-2 (Part 81 NSF/SSO Path หรือ
Dual-Homed WAN ปกติ) ได้ QoS Treatment เหมือนเดิมทุกประการ — **ถ้า WAN-EDGE-2 Config ไม่ตรงกับ
WAN-EDGE-1 นี่คือ "Weakest Link" ตาม Step 811 ที่มองข้ามได้ง่ายที่สุด** เพราะ Traffic ปกติไม่เคย
วิ่งผ่าน WAN-EDGE-2 เลยจนกว่าจะเกิด Failover จริง — QoS Design Audit ต้อง Verify อุปกรณ์ Standby
เหมือนกับอุปกรณ์ Active เสมอ

---

## Step 815 — Wireless QoS Integration: แม็บ 802.11e/WMM เข้ากับ DSCP บน Wired

### WMM (Wi-Fi Multimedia) คือ QoS เวอร์ชันของ Wi-Fi เอง (จาก 802.11e)

Wired ใช้ CoS (802.1p, 3 bit) และ DSCP (6 bit) — Wireless มีระบบของตัวเองเรียก **WMM** ซึ่งอิง
มาตรฐาน **802.11e** แบ่ง Traffic เป็น **4 Access Category (AC)** โดยใช้ **User Priority (UP)**
ค่า 0-7 (เหมือน CoS แต่ทำงานภายใน 802.11 MAC Layer ผ่าน EDCA — Enhanced Distributed Channel
Access — ไม่ใช่ 802.1Q Tag):

| Access Category (AC) | User Priority (UP) | CoS ที่เทียบเท่า | Traffic ตัวอย่าง | EDCA Priority |
|---|---|---|---|---|
| **AC_VO (Voice)** | 6, 7 | 6, 7 | VoWLAN (Part 20 Step 194 VoWLAN Roaming) | สูงสุด — Contention Window แคบที่สุด |
| **AC_VI (Video)** | 4, 5 | 4, 5 | Video Conference บน Wi-Fi | สูง |
| **AC_BE (Best Effort)** | 0, 3 | 0, 3 | Web Browsing, Email ทั่วไป | ปานกลาง (Default) |
| **AC_BK (Background)** | 1, 2 | 1, 2 | Print Job, Software Update เบื้องหลัง | ต่ำสุด — Contention Window กว้างที่สุด |

**หลักการทำงานของ EDCA**: แต่ละ AC มี **Contention Window (CW) และ AIFS (Arbitration Inter-Frame
Space) ต่างกัน** — AC_VO มี CW แคบและ AIFS สั้นที่สุด ทำให้ **มีโอกาสเข้าถึง Wireless Media
(Channel Access) ก่อน AC อื่นในทางสถิติ** เมื่อ Client หลายตัวแย่ง Channel เดียวกันพร้อมกัน —
นี่คือความต่างสำคัญจาก Wired QoS ที่ Queue ทำงานที่ Egress Interface เดียว (Wireless เป็น Shared
Medium ที่ Client ทุกตัวแย่ง Channel เดียวกัน จึงต้องมีกลไกจัดลำดับ**ที่ระดับ Channel Access**
ด้วย ไม่ใช่แค่ระดับ Queue เฉยๆ)

### จุดที่ QoS มัก "หลุด" ระหว่าง Wireless กับ Wired: CAPWAP Boundary

```
[Wi-Fi Client]  UP=6 (Voice, WMM)
       │ 802.11 Frame มี WMM/UP=6 ติดมา แต่ "ไม่มี DSCP" (802.11 ไม่มี Concept ของ DSCP)
       ▼
   [AP-1/AP-2]  ── ต้อง "แปล" UP=6 ให้เป็น DSCP ก่อนห่อ CAPWAP ──▶ DSCP=EF (46) ที่ Outer CAPWAP Header
       │ CAPWAP Data Tunnel (UDP 5247, Part 36 Step 351) ไปยัง WLC-1
       ▼
   [WLC-1]  ── แกะ CAPWAP ออก ── Client Frame เดิม (มี UP=6) + DSCP ที่ WLC เพิ่งตั้งไว้
       │ ส่งต่อเข้า Wired VLAN 20 (VOICE)
       ▼
   [DIST-SW1]  trust dscp ── เห็น DSCP=EF ── ปฏิบัติเหมือน Voice ปกติทุกประการ ✅
```

**ปัญหาที่พบบ่อยที่สุด**: ถ้า AP/WLC **ไม่ได้ Config ให้แปล UP→DSCP อย่างถูกต้อง** (ค่า Default
บางรุ่นอาจ Map UP ของ Voice ไปเป็น DSCP ที่ไม่ตรงกับ Enterprise Baseline ของ Step 812 เช่น Map
ไป AF31 แทน EF) — Voice Call บน Wi-Fi จะได้ **Priority ที่ถูกต้องเฉพาะช่วง "ในอากาศ" (WMM/EDCA)
แต่พอเข้าสาย Wired กลับถูกปฏิบัติเป็น Class อื่นที่ผิดจาก Baseline องค์กร** — Wireless และ Wired
Trust Boundary ต้อง **Verify คู่กันเสมอ** ไม่ใช่ตรวจแค่ฝั่งใดฝั่งหนึ่ง

### ตาราง Mapping เต็มรูปแบบที่ WLC-1 ต้องบังคับใช้ (ให้ตรงกับ Enterprise Baseline Step 812)

| WMM Access Category | User Priority | DSCP ที่ต้อง Map ให้ตรง Baseline | Enterprise Class |
|---|---|---|---|
| AC_VO | 6-7 | **EF (46)** | VoIP Telephony |
| AC_VI | 4-5 | **AF41 (34)** | Multimedia Conferencing |
| AC_BE | 0, 3 | **DF (0)** | Best Effort |
| AC_BK | 1-2 | **CS1 (8)** | Scavenger |

### Config WLC-1 (Catalyst 9800): QoS Profile ต่อ WLAN Policy Profile

ต่อยอดจาก Part 20 Step 195 (WLAN Profile → Policy Profile → Policy Tag) เพิ่ม QoS Profile
เข้าไปที่ Policy Profile ของแต่ละ SSID:

```
! ---------- QoS Profile สำหรับ Voice SSID (VLAN 20) ----------
WLC-1(config)# wireless profile policy VOICE-WIFI-POLICY
WLC-1(config-wireless-policy)#  vlan 20
WLC-1(config-wireless-policy)#  qos video default-video-profile
WLC-1(config-wireless-policy)#  qos voice default-voice-profile
WLC-1(config-wireless-policy)#  no shutdown
WLC-1(config-wireless-policy)# exit
!
! ---------- QoS Profile สำหรับ Data/BYOD SSID (VLAN 40) ----------
WLC-1(config)# wireless profile policy WIFI-DATA-POLICY
WLC-1(config-wireless-policy)#  vlan 40
WLC-1(config-wireless-policy)#  qos data default-silver-profile      ! Best Effort — ตรง Baseline DF
WLC-1(config-wireless-policy)#  no shutdown
WLC-1(config-wireless-policy)# exit
```

> **หมายเหตุ**: ชื่อ QoS Profile จริง (`platinum`/`gold`/`silver`/`bronze` เป็นชื่อ Default ของ
> Cisco ตั้งแต่ยุค AireOS ที่ยังใช้อยู่ใน IOS-XE 9800) แม็บกับ Baseline ดังนี้: **Platinum = Voice
> (EF)**, **Gold = Video (AF41)**, **Silver = Best Effort (DF, Default)**, **Bronze = Background/
> Scavenger (CS1)** — คำสั่งที่แสดงในตัวอย่างนี้เป็นชื่อ Custom Profile ที่สื่อความหมายชัดเจนกว่า
> เพื่อการสอน คำสั่งจริงต่างกันเล็กน้อยตามเวอร์ชัน IOS-XE ให้ Verify ด้วย `show wireless profile
> qos summary` ก่อนใช้งานจริงเสมอ

### Verify การแปล UP↔DSCP

```
WLC-1# show wireless profile policy detailed VOICE-WIFI-POLICY | include QOS
QOS Video Profile Name                   : default-video-profile
QOS Voice Profile Name                   : default-voice-profile

WLC-1# show wireless client mac-address <MAC ของ Wi-Fi Phone> detail | include DSCP|UP|WMM
WMM Support                              : Enabled
U-APSD Support                           : Enabled
Uplink DSCP                              : 46 (EF)
Downlink DSCP                            : 46 (EF)
```

`Uplink DSCP` / `Downlink DSCP` ที่ตรงกับ 46 (EF) ยืนยันว่า Voice Traffic จาก Wi-Fi Client
ถูกแปลเป็น DSCP ที่ตรงกับ Enterprise Baseline แล้วก่อนเข้าสู่ Wired Network — DIST-SW3/DIST-SW4
(ที่ต่อกับ WLC-1 ผ่าน VLAN 40, ดู [`00-ip-address-plan.md`](00-ip-address-plan.md)) จะ `trust
dscp` และเห็นค่านี้ทันที ไม่ต้อง Re-classify ซ้ำ — สอดคล้องกับหลักการ Step 811 ข้อ 2

---

## Step 816 — SD-WAN QoS Integration: Application-Aware Routing กับ MQC แบบเดิม

### คำถามสำคัญ: SD-WAN "แทนที่" MQC หรือ "ทำงานคู่กัน"

หลายคนเข้าใจผิดว่า SD-WAN (Part 39/76) ทำให้ไม่ต้อง Config MQC (Part 40) อีกต่อไป — **ความจริง
คือ SD-WAN เพิ่มความสามารถ "เลือก Path" ที่ MQC แบบเดิมไม่มี แต่ไม่ได้แทนที่การ Queue/Shape ที่
Underlay ยังต้องทำอยู่เหมือนเดิม**:

```
QoS แบบเดิม (Part 40) — Router มี "1 Path" ไปปลายทางเสมอ (Routing Table เลือกให้):
┌─────────────┐    LLQ/CBWFQ/WRED บน Interface เดียว     ┌──────────┐
│ WAN-EDGE-1   │──────────────────────────────────────▶│ ISP MPLS  │
└─────────────┘   ตอบได้แค่ "จัดลำดับ Traffic บน Link   └──────────┘
                    เดียวที่มี" — ไม่รู้จัก Path อื่น

SD-WAN AAR (Part 39/76) — cEdge มีหลาย Transport พร้อมกัน (MPLS/Internet/LTE):
┌─────────────┐   Transport 1 (MPLS)     ┌──────────┐
│ cEdge         │──────────────────────▶│           │
│ (WAN-EDGE-1) │   Transport 2 (Internet) │  ปลายทาง  │
│              │──────────────────────▶│           │
│              │   Transport 3 (LTE)      │           │
│              │──────────────────────▶│           │
└─────────────┘                          └──────────┘
       │
       ▼ AAR วัด Loss/Latency/Jitter ของ**ทุก Transport พร้อมกัน** แล้ว "เลือก Transport ที่ดี
         ที่สุดให้ Application นั้นในขณะนั้น" — ตอบคำถามที่ MQC เดิมตอบไม่ได้เลย: "ควรใช้ Path ไหน"

แต่ไม่ว่าจะเลือก Transport ไหน — Physical/Logical Interface ของ Transport นั้นก็ยังต้องมี
LLQ/CBWFQ/WRED ของตัวเอง (Underlay QoS) เพื่อจัดลำดับ Traffic **ภายใน** Transport ที่เลือกแล้ว
```

### ตารางสรุปว่า Layer ไหนตอบคำถามอะไร — ต้องแยกให้ชัดสำหรับ CCIE

| Layer | ตอบคำถาม | กลไก | สอนใน |
|---|---|---|---|
| **Overlay — Application-Aware Routing (AAR)** | "Traffic นี้ควรวิ่งออก Transport/Path ไหน จาก Path ที่มีให้เลือกหลายเส้น" | `sla-class` + `app-route-policy` วัด Loss/Latency/Jitter แบบ Real-time | Part 39 Step 386, Part 76 Step 753 |
| **Underlay — MQC แบบเดิม** | "เมื่อเลือก Transport แล้ว Traffic บน Interface นั้นควรถูกจัดลำดับอย่างไรตอน Congestion" | `class-map`/`policy-map`/`service-policy` (LLQ/CBWFQ/WRED/Shaping) | Part 40, Step 814 ของ Part นี้ |

**ทั้งสอง Layer ใช้ DSCP เป็น Input เดียวกัน**: `sla-class` ของ AAR match Traffic ด้วย DSCP
เหมือนกับที่ `class-map` ของ MQC match — **ถ้า DSCP Marking ไม่สอดคล้องกับ Enterprise Baseline
(Step 812) ทั้งสอง Layer จะทำงานผิดพลาดพร้อมกันทั้งคู่** ยิ่งตอกย้ำว่า Baseline กลางคือรากฐาน
ของทุกกลไก QoS ไม่ว่าจะเป็นแบบเดิมหรือ SD-WAN

### ตัวอย่าง Policy ที่ผสาน AAR (Overlay) เข้ากับ Enterprise Baseline (Step 812) โดยตรง

```
! บน vManage/Template ของ cEdge (WAN-EDGE-1/WAN-EDGE-2 ในโหมด SD-WAN — ดู Part 39/76)
sla-class VOICE-SLA
  latency        150     ! ตรงกับเกณฑ์ ITU-T G.114 จาก Part 24 Step 232
  loss           1
  jitter         30
!
sla-class MM-CONFERENCING-SLA
  latency        200
  loss           2
  jitter         50
!
app-route-policy ENTERPRISE-AAR-BASELINE
 vpn-list CORPORATE-VPN
  sequence 10
   match
    dscp 46                          ! EF — ตรงกับ VoIP Telephony Class ของ Baseline
   action
    sla-class VOICE-SLA strict
  sequence 20
   match
    dscp 34 36 38                    ! AF41/42/43 — Multimedia Conferencing Class
   action
    sla-class MM-CONFERENCING-SLA
!
apply-policy site-list ALL-SITES app-route-policy ENTERPRISE-AAR-BASELINE
```

สังเกตว่า `match dscp 46` และ `match dscp 34 36 38` ใน AAR Policy นี้ **ใช้ DSCP ค่าเดียวกัน
เป๊ะกับที่ `class-map VOICE`/`MM-CONFERENCING` ใน Step 814 match** — นี่คือหลักฐานที่แสดงว่า
Enterprise QoS Baseline (Step 812) เป็น "ภาษากลาง" ที่ทำให้ Overlay (AAR) และ Underlay (MQC)
พูดกันรู้เรื่องโดยไม่ต้องมี Mapping Table แยกซ้อนกันอีกชั้น

---

## Step 817 — Data Center Edge QoS: DSCP Preservation ข้าม VXLAN/EVPN Boundary

> **ขอบเขต Step นี้**: เป็นเพียง Preview แนวคิดเชื่อมต่อไปสู่ Part ถัดๆ ไปของหลักสูตรที่จะเจาะลึก
> VXLAN/EVPN Data Center Fabric เต็มรูปแบบ — Part 82 นี้ยังไม่สอนกลไก VXLAN โดยละเอียด

เมื่อ Campus (Step 813) เชื่อมต่อไปยัง Data Center ที่ใช้ **VXLAN/EVPN Fabric** (แทน 3-Tier
Model แบบ Campus) จุดเชื่อม (DC Edge/Border Leaf) ต้องตอบคำถามใหม่: **Traffic ที่ถูก Mark DSCP
มาจาก Campus แล้ว จะยังคง DSCP เดิมไว้ได้หรือไม่ เมื่อถูกห่อด้วย VXLAN Header เพื่อวิ่งข้าม
Fabric ภายใน Data Center**

```
Campus (DSCP=EF ที่ IP Header เดิม)
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ DC Border Leaf — ห่อ Packet เดิมด้วย VXLAN Encapsulation        │
│                                                                │
│  Original Packet: [IP Hdr DSCP=EF][UDP/TCP][Payload]           │
│                          │                                     │
│                          ▼ ห่อด้วย VXLAN                        │
│  VXLAN Packet: [Outer IP Hdr DSCP=?][UDP 4789][VXLAN Hdr]       │
│                [Original IP Hdr DSCP=EF][UDP/TCP][Payload]     │
└─────────────────────────────────────────────────────────────┘
```

**กฎที่ต้องยึด**: Border Leaf/VTEP (VXLAN Tunnel End Point) ต้อง **Copy DSCP จาก Inner (Original)
Header ไปยัง Outer (VXLAN) Header เสมอ** (เรียก **DSCP Preservation** หรือ Uniform Mode ของ
DiffServ Tunneling ตาม RFC 2983) — ถ้าไม่ทำ อุปกรณ์ทุกตัวภายใน Fabric ที่มองเห็นแค่ Outer Header
(Spine/Leaf Switch ที่ไม่แกะ VXLAN) จะเห็น Traffic ทั้งหมดเป็น DSCP เดียวกัน (มักเป็น 0 ถ้าตั้ง
ผิด) ทำให้ Voice/Video ที่มาจาก Campus **เสีย QoS Treatment ทั้งหมดทันทีที่เข้า Fabric** แม้จะ
ถูก Mark ถูกต้องมาตลอดทางจาก Campus ก็ตาม — เป็นตัวอย่าง "Weakest Link" (Step 811) ที่เกิดขึ้น
ได้ง่ายมากที่จุดเชื่อมระหว่างสอง Architecture ที่ต่างกัน (3-Tier Campus ↔ VXLAN/EVPN Fabric)

| แนวคิด | ความหมาย | ผลถ้าไม่ทำ |
|---|---|---|
| **Uniform Mode (แนะนำ)** | Copy DSCP จาก Inner → Outer ตอนเข้า VXLAN, Copy กลับ Outer → Inner ตอนออก (Decapsulation) | ไม่มี — QoS Consistent ตลอดเส้นทาง |
| **Pipe Mode** | Outer DSCP ตั้งค่าคงที่แยกจาก Inner (Fabric กำหนด QoS ของตัวเองไม่สนใจ Inner) | Fabric ควบคุม QoS ภายในตัวเองได้ แต่ Priority จาก Campus (เช่น Voice) อาจไม่ถูกสะท้อนใน Fabric — ต้องมี Design ที่ตั้งใจแยกกันชัดเจน ไม่ใช่ลืมทำ Uniform Mode โดยไม่ได้ตั้งใจ |

**ประเด็นเพิ่มเติมสำหรับ Storage Traffic (Preview)**: Data Center มี Traffic Class พิเศษที่
Campus ไม่มี — **Storage/RDMA Traffic** (NVMe-oF, FCoE) ที่ต้องการ **Lossless Fabric** ด้วย
**PFC (Priority Flow Control, IEEE 802.1Qbb)** และ **ETS (Enhanced Transmission Selection,
802.1Qaz)** — กลไกเหล่านี้ต่างจาก WRED/LLQ ที่ใช้ใน Campus/WAN โดยสิ้นเชิง (PFC คือ "หยุด
ส่งชั่วคราว" ต่อ Class แทนการ Drop) รายละเอียดเต็มรูปแบบจะเรียนใน Part ถัดๆ ไปที่เจาะ Data
Center Fabric โดยเฉพาะ

---

## Step 818 — QoS สำหรับ Cloud/SaaS Traffic: ขอบเขตที่ควบคุมไม่ได้

### ความจริงที่ต้องยอมรับ: QoS หมดอำนาจทันทีที่ Traffic ออกจากเครือข่ายองค์กร

ทุก Step ก่อนหน้าพูดถึงเส้นทางที่องค์กร**เป็นเจ้าของหรือควบคุมได้** (Campus, WAN Circuit ที่เช่า,
SD-WAN Overlay) — แต่ทันทีที่ Traffic ไปถึง Internet เพื่อเข้าใช้ **SaaS Application** (Microsoft
365, Salesforce, Webex Cloud, Google Workspace) **DSCP Marking ที่องค์กร Mark มาตลอดทางจะถูก
"ล้าง" หรือ "เพิกเฉย" โดย ISP/Cloud Provider ทันที** — ไม่มี Router หรือ ISP รายใดในเส้นทาง
สาธารณะที่จะยอมรับ/ปฏิบัติตาม DSCP ของแต่ละองค์กรที่ส่งเข้ามา (ถ้าทำแบบนั้นได้ ทุกคนก็จะ Mark
Traffic ของตัวเองเป็น EF หมด — Internet จะกลับไปมีปัญหาแบบ Trust Boundary ที่ไม่มีขอบเขตตาม
Part 24 Step 235)

```
Enterprise (ควบคุม QoS ได้ 100%)              Internet/Cloud Provider (ควบคุม QoS ไม่ได้เลย)
┌─────────────────────────────┐              ┌─────────────────────────────────────┐
│ Campus → WAN Edge → SD-WAN    │──DSCP=EF──▶ │ ISP Backbone → Cloud Provider Network │
│ (Trust/Mark/Queue ทุก Hop)     │   (ถูก        │ (Best Effort ล้วนๆ สำหรับ Traffic      │
│                               │    เพิกเฉย)   │  ทั่วไปที่ไม่ได้มีสัญญา SLA พิเศษ)         │
└─────────────────────────────┘              └─────────────────────────────────────┘
```

### Design Implication — จะออกแบบอย่างไรเมื่อควบคุม QoS ปลายทางไม่ได้

| แนวทาง | รายละเอียด | ข้อจำกัด |
|---|---|---|
| **Over-provision Egress Bandwidth** | เผื่อ Bandwidth ขา Internet Egress ให้มากเกินพอ (Headroom สูง) เพื่อลดโอกาส Congestion เกิดขึ้นเองในจุดที่เรายังควบคุมได้ (ก่อนออก ISP) | แก้ได้แค่ช่วงที่ยังอยู่ในมือเรา — ควบคุม Backbone ของ ISP/Cloud Provider ไม่ได้อยู่ดี |
| **Cloud OnRamp for SaaS (Part 76 Step 755)** | เลือก **Local Breakout ที่ดีที่สุด** ต่อแต่ละ SaaS App โดย Probe Performance จริงจากทุก Exit ที่มี แล้วเลือก Exit ที่ Response Time ดีที่สุด ณ ขณะนั้น | เป็นการ **"เลือกทางที่ดีที่สุดจากทางเลือกที่มี"** ไม่ใช่การ "สั่ง" ให้ ISP/Cloud ปฏิบัติต่อ Traffic เราเป็นพิเศษ — คุณภาพ Backbone ของแต่ละ ISP ยังเป็นตัวแปรที่ควบคุมไม่ได้ |
| **SLA จาก Cloud Provider โดยตรง (Direct Interconnect)** | ใช้ Private Connectivity เช่น ExpressRoute/Direct Connect/Cloud Interconnect ที่มี SLA Bandwidth การันตี แทนเส้นทางผ่าน Internet สาธารณะ | มีค่าใช้จ่ายสูงกว่ามาก และยังจำกัดแค่ระหว่างองค์กรกับ Cloud Provider เท่านั้น — ไม่ครอบคลุม SaaS ที่ไม่มี Direct Interconnect ให้ |
| **ยอมรับว่า Application ต้องออกแบบให้ทนต่อ Variable Network Quality** | เลือก Codec/Protocol ที่ Adaptive (เช่น WebRTC ที่ปรับ Bitrate ตาม Network Condition อัตโนมัติ) แทนการพยายามควบคุม Network ที่ควบคุมไม่ได้ | เป็นการยอมรับข้อจำกัดแทนการแก้ไข — แต่มักเป็นแนวทางที่ Cost-Effective ที่สุดในทางปฏิบัติ |

> **สรุปหลักคิดของ Step นี้**: Enterprise QoS Design ที่ดีต้อง **"รู้ขอบเขตของตัวเอง"** — ทำ
> QoS ให้สมบูรณ์แบบที่สุดใน**ส่วนที่ควบคุมได้** (Campus → WAN → SD-WAN Overlay → DC Edge ตาม
> Step 813-817) และใช้กลยุทธ์ **Mitigation** (Over-provision, Cloud OnRamp, Direct Interconnect)
> สำหรับส่วนที่ควบคุมไม่ได้ — ไม่ใช่พยายาม "แก้ QoS" ในเครือข่ายของผู้ให้บริการที่ไม่มีทางทำได้จริง

---

## Step 819 — QoS SLA Monitoring & Validation: ยืนยันผลลัพธ์จริงด้วย Telemetry

### "Config เสร็จแล้ว" ≠ "ใช้งานได้จริงตามที่ออกแบบ"

Step 813-817 สอนวิธี**ออกแบบและ Config** QoS ให้ครบทุก Tier — แต่ Config ที่ถูกต้องตามทฤษฎี
**ไม่ได้แปลว่า Voice/Video จะได้ Latency/Jitter/Loss ตามเกณฑ์จริงเสมอไป** (ตามที่ Part 68 Step
671 อธิบายไว้ว่า QoS มีธรรมชาติพิเศษ — ปัญหาไม่ปรากฏจนกว่าจะเกิด Congestion จริง) การออกแบบ QoS
ระดับ CCIE ต้องปิดท้ายด้วย **Loop การ Validate ต่อเนื่อง** ไม่ใช่ "Config แล้วจบ"

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  1. DESIGN   │────▶│ 2. DEPLOY    │────▶│ 3. MEASURE   │────▶│ 4. VALIDATE  │
│  (Step 812-  │     │  (Config     │     │  (Telemetry/ │     │  (เทียบกับ    │
│   817)       │     │   จริงลง     │     │   IP SLA     │     │   เกณฑ์ Part  │
│              │     │   อุปกรณ์)   │     │   ต่อเนื่อง)  │     │   24 Step232) │
└─────────────┘     └─────────────┘     └──────┬──────┘     └──────┬──────┘
       ▲                                       │                   │
       │                  ไม่ผ่านเกณฑ์ → กลับไปปรับ Design (วนกลับ)  │
       └───────────────────────────────────────────────────────────┘
```

### เครื่องมือที่ 1: `ip sla` แบบ Active Probe ที่จำลอง DSCP ของแต่ละ Class

IOS มีฟีเจอร์ `ip sla` (Part 17 พื้นฐาน) ที่ทำ **Active Measurement** — ส่ง Probe จริงเพื่อวัด
Latency/Jitter/Loss ตาม DSCP ของ Class ที่สนใจ (ใช้ Option `tos` เพื่อจำลอง DSCP เดียวกับ Voice
Traffic จริง ทำให้ Probe ได้รับ Treatment เดียวกับ Voice Call จริงตลอดเส้นทาง):

```
WAN-EDGE-1(config)# ip sla 20
WAN-EDGE-1(config-ip-sla)#  udp-jitter 10.10.30.50 16400 codec g711alaw
WAN-EDGE-1(config-ip-sla-jitter)#   tos 0xB8                     ! DSCP EF (46) — เหมือน Voice จริง
WAN-EDGE-1(config-ip-sla-jitter)#   tag VOICE-SLA-VALIDATION
WAN-EDGE-1(config-ip-sla-jitter)#   frequency 60
WAN-EDGE-1(config-ip-sla-jitter)#  exit
WAN-EDGE-1(config)# ip sla schedule 20 life forever start-time now
!
WAN-EDGE-1(config)# ip sla 21
WAN-EDGE-1(config-ip-sla)#  udp-jitter 10.10.30.50 16401
WAN-EDGE-1(config-ip-sla-jitter)#   tos 0x00                     ! DSCP 0 — เทียบกับ Best Effort
WAN-EDGE-1(config-ip-sla-jitter)#   tag BEST-EFFORT-BASELINE
WAN-EDGE-1(config-ip-sla-jitter)#   frequency 60
WAN-EDGE-1(config-ip-sla-jitter)#  exit
WAN-EDGE-1(config)# ip sla schedule 21 life forever start-time now
```

การรัน Probe คู่กัน (DSCP=EF กับ DSCP=0 ไป Destination เดียวกัน ในเวลาเดียวกัน) ทำให้เห็น
**ผลต่าง (Delta) ที่ QoS Policy สร้างได้จริง** — ถ้า Policy ทำงานถูกต้อง Probe ที่ Mark EF ควรมี
Latency/Jitter **ต่ำกว่าอย่างชัดเจน** เมื่อ Link เกิด Congestion เทียบกับ Probe ที่ไม่ได้ Mark

```
WAN-EDGE-1# show ip sla statistics 20
IPSLAs Latest Operation Statistics
IPSLA operation id: 20
Type of operation: udp-jitter
        Latest RTT: 42 milliseconds
Latest operation start time: 10:15:32
Latest operation return code: OK
RTT Values:
        Number Of RTT: 10       RTT Min/Avg/Max: 38/42/48 milliseconds
Source to Destination Latency one way Values:
        Number of Latency one way Samples: 10
        Source to Destination Latency one way Min/Avg/Max: 18/21/24 milliseconds
Destination to Source Latency one way Values:
        Source to Destination Latency one way Min/Avg/Max: 19/21/23 milliseconds
Source to Destination Jitter Min/Avg/Max: 2/4/8 milliseconds
Destination to Source Jitter Min/Avg/Max: 2/3/7 milliseconds
Packet Loss (calculated): 0.0%

WAN-EDGE-1# show ip sla statistics 21
IPSLAs Latest Operation Statistics
IPSLA operation id: 21
        Latest RTT: 186 milliseconds                              ← Best Effort ช้ากว่าเห็นชัด
Source to Destination Jitter Min/Avg/Max: 15/38/95 milliseconds    ← Jitter สูงกว่ามาก
Packet Loss (calculated): 3.2%                                     ← Loss สูงกว่าเกณฑ์ VoIP ทันที
```

**การอ่านผล**: Probe 20 (EF) ได้ Latency ~21ms, Jitter ~4ms, Loss 0% — **ผ่านเกณฑ์ VoIP ทุกตัว**
(< 150ms, < 30ms, < 1% ตาม Part 24 Step 232) ในขณะที่ Probe 21 (Best Effort ไป Destination
เดียวกัน) มี Loss 3.2% และ Jitter สูงกว่ามาก — นี่คือ**หลักฐานเชิงตัวเลขที่พิสูจน์ว่า QoS Policy
ทำงานได้ผลจริง** ไม่ใช่แค่ "Config ดูถูกต้องบนหน้าจอ"

### เครื่องมือที่ 2: Streaming Telemetry ต่อเนื่อง (ต่อยอด Part 54) สำหรับ `show policy-map interface` แบบ Real-time

Part 54 สอน Model-Driven Telemetry (`telemetry ietf`) สำหรับ CPU/Interface Counter — นำ Sensor
Path เดียวกันมาใช้กับ **QoS Operational YANG Model** (`Cisco-IOS-XE-qos-oper`) เพื่อ Stream
Counter ของ `show policy-map interface` (Drop Rate, Queue Depth ต่อ Class) แบบต่อเนื่องไปยัง
Collector โดยไม่ต้อง SSH เข้าไป `show` เองทุกครั้ง:

```
WAN-EDGE-1(config)# telemetry ietf sensor-group SG-QOS-POLICY
WAN-EDGE-1(config-telemetry-sensor-grp)#  sensor-path Cisco-IOS-XE-qos-oper:qos-oper-data/policy-oper-data
WAN-EDGE-1(config-telemetry-sensor-grp)# exit
!
WAN-EDGE-1(config)# telemetry ietf subscription 200
WAN-EDGE-1(config-telemetry-subscription)#  sensor-group-id SG-QOS-POLICY sample-interval 30000
WAN-EDGE-1(config-telemetry-subscription)#  destination-id DG-NOC-COLLECTOR
WAN-EDGE-1(config-telemetry-subscription)#  source-address 1.1.1.21
WAN-EDGE-1(config-telemetry-subscription)# end
```

`DG-NOC-COLLECTOR` คือ Destination-group เดียวกันที่ตั้งไว้ใน Part 54 — ทำให้ **Dashboard เดียว
กัน** (Grafana ผ่าน InfluxDB ตาม Part 54 Step 535) แสดงทั้ง CPU/Interface Health และ QoS Class
Drop Rate ของทุก Class ในหน้าเดียวกัน ไม่ต้องมีระบบแยก

### กฎการ Alert ที่ควรตั้ง (เทียบกับเกณฑ์ Part 24 Step 232) — ป้องกัน "Config แล้วลืม"

| Metric ที่ Monitor | เกณฑ์ Threshold สำหรับ Alert | Class ที่ต้องเฝ้าเข้มที่สุด |
|---|---|---|
| VOICE Drop Rate (`drop rate` จาก `show policy-map interface`) | **> 0 bps ต่อเนื่องเกิน 5 นาที** | VOICE (ควรเป็น 0 เสมอถ้า LLQ ทำงานถูกต้อง — Drop รอบเดียวก็ควรสงสัยแล้ว) |
| IP SLA Jitter (Probe DSCP=EF) | > 30 ms | VOICE |
| IP SLA Latency (Probe DSCP=EF) | > 150 ms | VOICE, INTERACTIVE-VIDEO |
| IP SLA Packet Loss (Probe DSCP=EF) | > 1% | VOICE |
| `class-default`/SCAVENGER Drop Rate | (ไม่ต้อง Alert — Drop ที่นี่คือผลลัพธ์ที่ตั้งใจออกแบบ ตาม Part 40 Step 400) | — |

> **หลักคิดสำคัญ**: ต้อง Alert เมื่อ **Class ที่สำคัญที่สุดเริ่มมีปัญหา** เท่านั้น — ถ้า Alert
> ทุก Drop ของทุก Class (รวม Scavenger/Best Effort ที่ถูกออกแบบให้ Drop เป็นปกติ) ทีม NOC จะเจอ
> Alert Fatigue จนเพิกเฉย Alert ที่สำคัญจริงๆ ไปด้วย — Threshold ต้องสะท้อน**เจตนาการออกแบบ**
> ของ Policy-map เอง ไม่ใช่ Alert แบบเหมาเข่งทุกอย่างที่ขยับ

---

## Step 820 — Lab เต็มรูปแบบ: End-to-End QoS Design ทั้งองค์กร

### เป้าหมาย Lab

ประกอบทุก Step (811-819) เข้าเป็น **QoS Design Document + Configuration Set เดียว** ที่ Deploy
จริงทั่วทั้ง Lab ของหลักสูตร: ACCESS-SW1-4 (Trust Boundary), DIST-SW1-4/CORE-SW1-2 (Trust+Queue),
WAN-EDGE-1/2 (LLQ+CBWFQ+WRED+HQoS เต็มรูปแบบ), WLC-1 (Wireless DSCP Mapping), และปิดท้ายด้วย
SLA Validation Dashboard Concept

### Topology เต็มรูปแบบของ Lab (End-to-End QoS Path)

```
[IP Phone VLAN20]  [Wi-Fi Client AC_VO]              [IP Phone ปลายทาง — สาขา/DR ผ่าน WAN-EDGE-2]
        │                    │                                        ▲
        │ Mark EF            │ WMM UP=6 → WLC-1 แปลเป็น DSCP=EF          │
        ▼                    ▼                                        │
┌──────────────┐      ┌──────────────┐                                 │
│ ACCESS-SW1/2  │      │ WLC-1         │                                │
│ trust device  │      │ (VLAN20/40)   │                                │
│ cisco-phone   │      │ qos voice/    │                                │
│ (Step 813)    │      │ video profile │                                │
│               │      │ (Step 815)    │                                │
└──────┬───────┘      └──────┬───────┘                                │
       │ Trunk trust dscp           │ trust dscp                        │
       ▼                            ▼                                  │
┌──────────────────────────────────────────┐                          │
│         DIST-SW1-4 (trust dscp + CAMPUS-UPLINK-QUEUE, Step 813)       │
└──────────────────┬───────────────────────┘                          │
                    │ trust dscp                                       │
                    ▼                                                  │
┌──────────────────────────────────────────┐                          │
│         CORE-SW1/CORE-SW2 (trust dscp)     │                          │
└──────────────────┬───────────────────────┘                          │
                    │ trust dscp                                       │
                    ▼                                                  │
┌──────────────────────────────────────────┐   WAN (ISP)               │
│  WAN-EDGE-1 — ENTERPRISE-WAN-QOS +          │═══════════════════════▶│
│  WAN-SHAPE-50M (HQoS เต็มรูปแบบ, Step 814)  │   WAN-EDGE-2 (Config เดียวกันทุกประการ)
└──────────────────────────────────────────┘
                    │
                    ▼ Telemetry + ip sla (Step 819)
┌──────────────────────────────────────────┐
│   SLA Validation Dashboard (NOC)            │
│   - Drop Rate ต่อ Class ทุกอุปกรณ์            │
│   - IP SLA Latency/Jitter/Loss ต่อ DSCP      │
│   - Alert เมื่อ VOICE Class เริ่มมีปัญหา       │
└──────────────────────────────────────────┘
```

### เอกสารออกแบบสรุป (QoS Design Document — ใช้ยื่นตรวจใน CCIE Design Review จริง)

| หัวข้อ | รายละเอียด | อ้างอิง |
|---|---|---|
| Class Model | Enterprise QoS Baseline 12 Class (ยุบเป็น 8 Class ที่ WAN Edge) | Step 812 |
| Trust Boundary | ACCESS-SW1-4 (`trust device cisco-phone`), WLC-1 (แปล WMM→DSCP) | Step 813, 815 |
| Campus Queue | DIST-SW1-4/CORE-SW1-2: `trust dscp` + `CAMPUS-UPLINK-QUEUE` (Realtime 25% LLQ, Critical 30% CBWFQ) | Step 813 |
| WAN Edge Policy | WAN-EDGE-1/2: `ENTERPRISE-WAN-QOS` (LLQ 30% รวม Realtime, CBWFQ 50%, WRED ทุก AF Class) ภายใต้ `WAN-SHAPE-50M` | Step 814 |
| Wireless Mapping | AC_VO→EF, AC_VI→AF41, AC_BE→DF, AC_BK→CS1 | Step 815 |
| SD-WAN (ถ้าใช้) | `app-route-policy` match DSCP เดียวกับ Baseline, ทำงานคู่กับ Underlay MQC | Step 816 |
| DC Edge (Preview) | DSCP Preservation แบบ Uniform Mode ที่ VXLAN VTEP | Step 817 |
| Cloud/SaaS | Over-provision Egress + Cloud OnRamp for SaaS, ยอมรับว่าควบคุม Backbone ผู้ให้บริการไม่ได้ | Step 818 |
| Validation | `ip sla udp-jitter` DSCP=EF ทุก WAN Edge + Streaming Telemetry `qos-oper` ไปยัง Dashboard | Step 819 |

### Verification สุดท้าย: ยืนยัน DSCP เดียวกันทุก Hop ตลอดเส้นทาง (Weakest Link Check ตาม Step 811)

```
! บน ACCESS-SW1 — จุด Mark
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1 | include trust
trust state: trust cos
trust mode: trust device cisco-phone

! บน DIST-SW1 — จุด Trust ที่ 1
DIST-SW1# show mls qos interface TenGigabitEthernet1/1/1 | include trust
trust state: trust dscp

! บน CORE-SW1 — จุด Trust ที่ 2
CORE-SW1# show mls qos interface TenGigabitEthernet1/0/1 | include trust
trust state: trust dscp

! บน WAN-EDGE-1 — จุด Enforcement สุดท้าย
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE
    Class-map: VOICE (match-any)
      Match: dscp ef (46)
      Priority: 10% (5,000 kbps)
      drop rate 0 bps
```

**ผลลัพธ์ที่ต้องเห็นตรงกันทุก Hop**: `trust dscp`/`trust cos` ที่ Trust Boundary ถูกต้องทุกจุด,
`Match: dscp ef (46)` ที่ WAN-EDGE-1 แสดงว่า Voice Packet ที่มาจาก ACCESS-SW1 ยังคง DSCP=EF
เดิมไว้ได้ตลอดเส้นทาง 4 Hop (ACCESS→DIST→CORE→WAN Edge) โดยไม่ถูก Reset เป็น 0 ที่จุดไหนเลย —
นี่คือ**หลักฐานที่พิสูจน์ว่า End-to-End QoS Design ตาม Step 811 สำเร็จจริงในทางปฏิบัติ** ไม่ใช่
แค่ทฤษฎีบนกระดาษ

### สรุป Lab Step 820

| ส่วนประกอบ | อุปกรณ์ | สิ่งที่ Verify แล้ว |
|---|---|---|
| Trust Boundary ต้นทาง | ACCESS-SW1-4, WLC-1 | Mark ครั้งเดียว ถูกต้องตาม Baseline |
| Campus Trust/Queue | DIST-SW1-4, CORE-SW1-2 | `trust dscp` ทุก Interface ไม่มี Re-mark |
| WAN Edge Enforcement | WAN-EDGE-1/2 | LLQ/CBWFQ/WRED ตรงกับ Enterprise Baseline ทั้ง 2 ตัวเท่ากัน |
| Wireless Integration | WLC-1 | WMM UP → DSCP Mapping ตรง Baseline |
| SLA Validation | `ip sla` + Telemetry | Voice DSCP=EF Latency/Jitter/Loss ผ่านเกณฑ์ Part 24 ต่อเนื่อง |

---

## แบบฝึกหัดทวนความเข้าใจ Part 82

1. อธิบายหลักการ "Weakest Link" ของ QoS End-to-End — ถ้า Distribution Switch หนึ่งตัวใน 5 ตัวที่
   Traffic ผ่านไม่ได้ Config `trust dscp` จะเกิดผลอย่างไรกับ Voice Call ทั้งสาย?

2. Enterprise QoS Baseline มีทั้งหมดกี่ Class และเพราะเหตุใด Class ที่เป็น LLQ (Realtime) รวมกัน
   จึงไม่ควรเกิน 33% ของ Bandwidth Link?

3. อธิบายความแตกต่างของงาน QoS ที่ Access Switch เทียบกับ Distribution/Core Switch ในสถาปัตยกรรม
   Campus — เพราะเหตุใด Distribution/Core จึง "แค่ Trust + Queue" โดยไม่ต้อง Re-mark?

4. WMM Access Category AC_VO ควรถูกแปลงเป็น DSCP อะไรเมื่อเข้าสู่ Wired Network ตาม Enterprise
   Baseline และเพราะเหตุใดจึงต้อง Verify ทั้งฝั่ง Wireless และฝั่ง Wired คู่กันเสมอ?

5. เพราะเหตุใดองค์กรจึง "ควบคุม QoS ไม่ได้เลย" ทันทีที่ Traffic ออกไปยัง SaaS Application บน
   Internet และมีแนวทาง Mitigation อะไรบ้างสำหรับข้อจำกัดนี้?

### เฉลย

1. Voice Call ทั้งสายจะได้รับ Best Effort Treatment ตั้งแต่ Distribution Switch ตัวนั้นเป็นต้นไป
   ถึงปลายทาง แม้ ACCESS Switch ต้นทางและอุปกรณ์อื่นทุกตัวใน Path จะ Config ถูกต้องสมบูรณ์แบบก็
   ตาม เพราะ Distribution Switch ที่ไม่ Trust จะ Reset DSCP กลับเป็น 0 (Best Effort) ก่อนส่งต่อ
   ทำให้ Hop ที่เหลือทั้งหมดเห็น Traffic นี้เป็น Best Effort ไปตลอด — นี่คือความหมายของ "QoS ดี
   เท่ากับ Hop ที่แย่ที่สุดในเส้นทางเท่านั้น"

2. Enterprise QoS Baseline มีทั้งหมด **12 Class** (VoIP Telephony, Broadcast Video, Realtime
   Interactive, Multimedia Conferencing, Multimedia Streaming, Network Control, Signaling,
   Network Management, Transactional Data, Bulk Data, Best Effort, Scavenger) — Class ที่เป็น
   LLQ (Realtime 3 Class แรก) ไม่ควรเกิน 33% เพราะ LLQ เป็น Strict Priority ที่ถูกส่งก่อน Class
   อื่นเสมอเมื่อมี Packet รออยู่ ถ้าสัดส่วนนี้สูงเกินไป (Realtime Traffic พุ่งสูงผิดปกติ หรือถูก
   Mark ผิดเข้ามาปนมากเกินสัดส่วนที่ควร) จะเสี่ยง Starve Class อื่นที่ไม่ใช่ Priority ทั้งหมด
   แม้ LLQ แต่ละ Class จะมี Policer ในตัว (ตาม Part 40 Step 393) ก็ตาม

3. Access Switch ทำหน้าที่ **Classify + Mark ครั้งแรก** (เพราะเป็นจุดที่ใกล้ Source/Endpoint
   ที่สุด และต้องตรวจสอบความน่าเชื่อถือของ Endpoint ผ่าน Trust Boundary ตาม Part 24 Step 235)
   ส่วน Distribution/Core แค่ **Trust สิ่งที่ Access Mark มาแล้ว และจัด Queue กัน Microburst**
   จากการที่หลาย Access Switch ยิง Traffic เข้า Uplink เดียวพร้อมกัน (Many-to-One Fan-in) — ไม่
   ต้อง Re-mark ซ้ำเพราะ Mark เดิมถูกต้องอยู่แล้วตามหลักการ DiffServ (Classify ครั้งเดียวที่ Edge,
   ทุก Hop ที่เหลือปฏิบัติตาม Per-Hop Behavior จากป้ายที่มีอยู่แล้ว)

4. AC_VO (User Priority 6-7) ควรถูกแปลงเป็น **DSCP EF (46)** เพื่อให้ตรงกับ VoIP Telephony Class
   ของ Enterprise Baseline — ต้อง Verify ทั้งสองฝั่งเพราะ Wireless (WMM/802.11e) และ Wired
   (CoS/DSCP) เป็นระบบ QoS ที่ทำงานคนละกลไกกันโดยสิ้นเชิง (WMM ทำงานที่ Channel Access ผ่าน
   EDCA, Wired ทำงานที่ Egress Queue ผ่าน DSCP) จุดเชื่อม (AP/WLC ที่ห่อ/แกะ CAPWAP) เป็นจุดที่
   ค่า QoS ต้อง "แปล" ข้ามระบบ — ถ้าแปลผิด Voice Call จะได้ Priority ถูกต้องแค่ตอนอยู่ใน "อากาศ"
   (WMM) แต่พอเข้าสาย Wired กลับถูกปฏิบัติเป็น Class อื่นที่ผิดจาก Baseline องค์กรทันที

5. เพราะทันทีที่ Traffic ออกไปยัง Internet สาธารณะ ไม่มี ISP หรือ Cloud Provider รายใดยอมรับ/
   ปฏิบัติตาม DSCP Marking ขององค์กรที่ส่งเข้ามา (ถ้าทำแบบนั้นได้ทุกองค์กรก็จะ Mark Traffic ของ
   ตัวเองเป็น Priority สูงสุดหมด) — Backbone ของ ISP/Cloud Provider จึงเป็น "Best Effort" สำหรับ
   Traffic ทั่วไปเสมอ ไม่ว่าองค์กรจะ Mark มาดีเพียงใด แนวทาง Mitigation ที่ทำได้คือ Over-provision
   Bandwidth Egress ให้เหลือ Headroom มาก, ใช้ Cloud OnRamp for SaaS (Part 76) เพื่อเลือก Local
   Breakout ที่ Performance ดีที่สุดจากทางเลือกที่มี, ใช้ Direct Interconnect ที่มี SLA จริงถ้า
   จำเป็นและคุ้มค่าใช้จ่าย, และออกแบบ Application ให้ทนต่อ Variable Network Quality (Adaptive
   Codec/Protocol) แทนการพยายามควบคุม Network ที่ควบคุมไม่ได้จริง

---

## สรุป Part 82

Part นี้ยกระดับ QoS จาก "การ Config อุปกรณ์ทีละตัว" (Part 24 ที่ Access, Part 40 ที่ WAN Edge)
ไปสู่ **การออกแบบสถาปัตยกรรมเดียวที่ครอบคลุมทั้งองค์กร**: หลักการ **Weakest Link** ที่บอกว่า
QoS ดีเท่ากับ Hop ที่แย่ที่สุดในเส้นทางเท่านั้น, **Enterprise QoS Baseline 12 Class** ที่เป็น
"ภาษากลาง" DSCP ให้ทุก Tier ขององค์กรพูดกันรู้เรื่อง, **Campus Design** ที่แยกหน้าที่ชัดระหว่าง
Access (Classify+Mark) กับ Distribution/Core (Trust+Queue), **WAN Edge Design** ที่ขยาย MQC
จาก Part 40 ให้ครอบคลุม Baseline เต็มรูปแบบด้วย Multi-level LLQ, **Wireless Integration** ที่
แปล WMM/802.11e ให้ตรงกับ DSCP ฝั่ง Wired, **SD-WAN Integration** ที่แสดงว่า AAR (Overlay) และ
MQC (Underlay) ทำงานคู่กันไม่ใช่แทนกัน, **Data Center Edge** ที่ต้อง Preserve DSCP ข้าม VXLAN
Boundary (Preview), **ขอบเขตของ QoS ต่อ Cloud/SaaS Traffic** ที่ควบคุมไม่ได้เลยนอกเครือข่ายตัวเอง,
และปิดท้ายด้วย **SLA Monitoring/Validation** ที่พิสูจน์ด้วยตัวเลขจริง (`ip sla`, Streaming
Telemetry) ว่า Design ที่วางไว้ทำงานได้ผลจริงในทางปฏิบัติ ไม่ใช่แค่ทฤษฎีบนกระดาษ

ทักษะ "มองภาพรวมทั้งเส้นทาง" ที่ Part นี้สอนคือหัวใจของ **CCIE Enterprise Infrastructure Design
Thinking** ที่จะใช้ซ้ำกับหัวข้ออื่นตลอดช่วง Part 80+ ของหลักสูตร — ไม่ว่าจะเป็น Multicast
(Part 83), Security Architecture, หรือ Automation ระดับองค์กร ทุกหัวข้อล้วนต้องตอบคำถามเดียวกัน:
"ทุก Hop ในเส้นทางทำงานสอดคล้องกันจริงหรือไม่ ไม่ใช่แค่อุปกรณ์ตัวเดียวที่ถูก Config ดี"

✅ **พร้อมสำหรับ Part 83**: เราจะเปลี่ยนหัวข้อไปที่ **Enterprise Multicast Deep Dive** — ต่อยอด
จาก Multicast Fundamentals/Advanced (Part 41-42) และ Advanced Multicast Enterprise (Part 75)
ด้วยมุมมองระดับ CCIE เต็มรูปแบบ: PIM Sparse Mode ขั้นสูง, Multicast ข้าม Multi-site/SD-WAN
Fabric, และการทำงานร่วมกับ QoS ที่เพิ่งเรียนใน Part นี้สำหรับ IPTV/Video Streaming Broadcast
Class (CS5) ที่ต้องการทั้ง Multicast Replication ที่ถูกต้องและ Priority Queue Treatment พร้อมกัน

**ไปต่อ:** [Part 83 — Enterprise Multicast Deep Dive →](part-083-enterprise-multicast-deep-dive.md)
