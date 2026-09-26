# Part 40 — Advanced QoS (MQC, Queuing, Shaping/Policing)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 391–400 จาก 1000**

> ต่อจาก [Part 39 — SD-WAN Fundamentals](part-039-sd-wan-fundamentals.md) ที่สอนการออกแบบ WAN
> แบบ Software-Defined ด้วย Cisco Catalyst SD-WAN (Viptela) แล้ว Part นี้จะย้อนกลับมาทำตามสัญญาที่
> [Part 24 — QoS Fundamentals](part-024-qos-fundamentals.md) ให้ไว้ตั้งแต่ระดับ CCNA — ตอนนั้นเรา
> เรียนแค่ **Trust Boundary** และ **Auto-QoS** ซึ่งเพียงพอสำหรับ Access-Layer ทั่วไป แต่ยังไม่ได้แตะ
> **MQC (Modular QoS CLI)** แบบ manual เต็มรูปแบบเลย — Part 40 นี้คือ Part ที่เจาะลึกเต็มรูปแบบ:
> `class-map` → `policy-map` → `service-policy`, CBWFQ, LLQ, WRED, Traffic Shaping, Traffic
> Policing (single-rate/dual-rate three-color), Hierarchical QoS (HQoS), และ NBAR/NBAR2 — ทักษะ
> ที่จำเป็นสำหรับ WAN Edge Router และ Core/Distribution Switch ระดับ Enterprise จริง

> **บริบทของ Part นี้**: เราจะใช้ **WAN-EDGE-1** (ISR4451, ดู [`00-ip-address-plan.md`](00-ip-address-plan.md))
> เป็นอุปกรณ์หลักในการสอน MQC เพราะ WAN Edge คือจุดที่ QoS มีผลกระทบสูงสุด — Bandwidth ของ WAN
> Circuit (Metro-Ethernet ไป ISP-RTR ที่ `203.0.113.0/30`) มีจำกัดกว่า LAN มาก (LAN เป็น Gigabit/
> 10-Gigabit แทบทุกจุด แต่ WAN Circuit มักถูกจำกัดด้วยสัญญา CIR ของผู้ให้บริการ) ทำให้ Congestion
> เกิดขึ้นจริงและ QoS มีความหมายชัดเจนที่สุดที่จุดนี้ นอกจากนี้เราจะใช้ **VLAN 40 (WIFI)** ที่
> DIST-SW3/DIST-SW4 เป็นตัวอย่างการทำ Traffic Policing เพื่อจำกัด Guest/BYOD Wi-Fi ไม่ให้แย่ง
> Bandwidth ที่ควรสงวนไว้ให้ Voice และ Business-Critical Application

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 391 | MQC (Modular QoS CLI) Architecture — โมเดล 3 ส่วน class-map → policy-map → service-policy |
| 392 | Class-map เจาะลึก — `match` ทุกรูปแบบ, match-any vs match-all, worked class-map VOICE/VIDEO/CRITICAL-DATA/BEST-EFFORT |
| 393 | Congestion Management — CBWFQ (`bandwidth`/`bandwidth percent`) และ LLQ (`priority`) |
| 394 | Congestion Avoidance — WRED (`random-detect`), DSCP-based Profile, ป้องกัน TCP Global Synchronization |
| 395 | Traffic Shaping — `shape average`/`shape peak`, Token Bucket (Bc/Be/Tc) |
| 396 | Traffic Policing — `police` Single-rate/Dual-rate Three-Color, เปรียบเทียบ Shape vs Police |
| 397 | Hierarchical QoS (HQoS) — Parent Shaping + Child Queuing |
| 398 | NBAR/NBAR2 — Deep Packet Inspection สำหรับ Classification, แนะนำ AVC |
| 399 | Verification — `show policy-map interface`, `show class-map`, `show policy-map`, Troubleshooting |
| 400 | Lab เต็มรูปแบบ: LLQ+CBWFQ+WRED+HQoS บน WAN-EDGE-1 พร้อม Policing บน VLAN 40 WIFI |

---

## Step 391 — MQC (Modular QoS CLI) Architecture

### ทำไมต้องมี MQC

ใน Part 24 เราใช้ `mls qos trust ...` และ `auto qos voip cisco-phone` ซึ่งเป็นคำสั่งที่ IOS
"เดา" การตั้งค่าที่เหมาะสมให้อัตโนมัติ เพียงพอสำหรับ Access-Layer Switch ที่ต่อ IP Phone ตรงๆ
แต่คำสั่งเหล่านั้น **ปรับแต่งละเอียดไม่ได้** — ถ้าต้องการ Policy ที่ซับซ้อนกว่า เช่น "Voice ต้องได้
Priority แบบ Strict แต่จำกัดไม่เกิน 10% ของ Bandwidth, Database Traffic ต้องได้ Bandwidth
การันตีอย่างน้อย 30%, ส่วนที่เหลือแบ่งแบบ Fair" — ต้องใช้ **MQC (Modular QoS Command-Line
Interface)** ซึ่งเป็นภาษา config ที่ใช้ร่วมกันในทุกกลไก QoS ของ IOS (Marking, Queuing, Shaping,
Policing, WRED) ตั้งแต่ Part 40 นี้เป็นต้นไป

### โมเดล 3 ส่วนของ MQC

```
┌─────────────────┐        ┌─────────────────┐        ┌──────────────────────┐
│  1. class-map     │  ───▶ │  2. policy-map    │  ───▶ │ 3. service-policy      │
│  "นี่คือ Traffic    │        │  "ทำอะไรกับ Class   │        │  "ผูก Policy เข้ากับ   │
│   ประเภทไหน?"      │        │   นี้แต่ละอัน?"      │        │   Interface จริง"      │
│                   │        │                   │        │                       │
│  match dscp ef    │        │  class VOICE      │        │  interface Gi0/0/0     │
│  match cos 5      │        │   priority ...    │        │   service-policy       │
│  match access-grp │        │  class VIDEO      │        │   output WAN-EDGE-QOS  │
│  match protocol   │        │   bandwidth ...   │        │                       │
└─────────────────┘        └─────────────────┘        └──────────────────────┘
     CLASSIFICATION              ACTION (Queue/Shape/Police/Mark)      APPLY
```

**หลักการสำคัญที่สุดที่ต้องเข้าใจก่อน config**: `class-map` แค่ "นิยาม" ว่า traffic แบบไหนเข้าเงื่อนไข
— มันไม่ได้ทำอะไรกับ traffic เลยจนกว่าจะถูกอ้างอิงใน `policy-map` และ `policy-map` เองก็ไม่มีผล
อะไรกับ Interface จริงจนกว่าจะถูก "แปะ" เข้า Interface ด้วย `service-policy` — สามอย่างนี้แยกกัน
โดยสมบูรณ์ ทำให้ `class-map` และ `policy-map` เดียวกัน **นำไปใช้ซ้ำ (Reuse)** ได้กับหลาย Interface
พร้อมกัน โดยไม่ต้องเขียนใหม่ทุกครั้ง

### Syntax Reference แบบเต็ม

```
! ส่วนที่ 1: Classification
class-map [match-any | match-all] CLASS-NAME
 match dscp ef
 match cos 5
 match access-group {number | name ACL-NAME}
 match protocol PROTOCOL-NAME          ! ใช้ NBAR — ดู Step 398
 match ip dscp value1 [value2 ...]     ! syntax เก่า ยังใช้ได้ (backward compatible)
 match qos-group value
 match not <เงื่อนไข>                  ! ปฏิเสธเงื่อนไข (negate)

! ส่วนที่ 2: Policy (Action)
policy-map POLICY-NAME
 class CLASS-NAME
  priority {percent PCT | KBPS}        ! LLQ — Strict Priority Queue (Step 393)
  bandwidth {percent PCT | KBPS}       ! CBWFQ — Guaranteed Minimum Bandwidth (Step 393)
  shape {average | peak} BPS [Bc] [Be] ! Traffic Shaping (Step 395)
  police cir BPS [bc BYTES] [pir BPS]  ! Traffic Policing (Step 396)
   conform-action ACTION
   exceed-action ACTION
   violate-action ACTION
  random-detect [dscp-based | precedence-based]   ! WRED (Step 394)
  set dscp VALUE                       ! (Re-)Marking
  queue-limit PACKETS                  ! ขนาด Queue สูงสุด (Tail Drop threshold)
  service-policy CHILD-POLICY-NAME     ! HQoS — Nested Policy (Step 397)
 class class-default                   ! Class พิเศษ — จับทุกอย่างที่ไม่ match class อื่น
  fair-queue
  random-detect dscp-based

! ส่วนที่ 3: Apply เข้า Interface
interface GigabitEthernet0/0/0
 service-policy {input | output} POLICY-NAME
```

### ลำดับที่ IOS ประมวลผล Class ใน Policy-map — สำคัญมากสำหรับ Step 399

IOS จะไล่ตรวจ packet กับ Class ใน `policy-map` **ตามลำดับที่พิมพ์ไว้ จากบนลงล่าง** และ
**หยุดที่ Class แรกที่ match (First-Match-Wins)** — ถ้า packet เข้าเงื่อนไขของหลาย Class
มันจะถูกจัดเข้า Class ที่อยู่บนสุดเท่านั้น ไม่ใช่ทุก Class ที่ match ได้ — Class `class-default`
จะถูกประมวลผล**เป็นลำดับสุดท้ายเสมอ**ไม่ว่าจะเขียนไว้ตรงไหนใน config (IOS ย้ายมันไปไว้ท้ายอัตโนมัติ)
และมันจะรับ packet **ทุกตัวที่ไม่ match Class อื่นใดเลย**

### ตารางสรุปหน้าที่ 3 ส่วนของ MQC

| ส่วน | คำสั่งหลัก | ตอบคำถาม | Reusable? |
|---|---|---|---|
| Classification | `class-map` | "Traffic นี้คือประเภทไหน?" | ใช้ซ้ำได้ในหลาย policy-map |
| Policy (Action) | `policy-map` + `class` | "จะทำอะไรกับ Traffic ประเภทนี้?" | ใช้ซ้ำได้ในหลาย interface |
| Apply | `service-policy` | "ใช้ Policy นี้ตรงจุดไหนของ Interface (input/output)?" | ผูกกับ Interface เฉพาะจุด |

---

## Step 392 — Class-map เจาะลึก: `match` ทุกรูปแบบ และ match-any vs match-all

### ตัวเลือก `match` ที่ใช้บ่อยที่สุด

| Match Option | ตัวอย่าง | ใช้เมื่อ |
|---|---|---|
| `match dscp` | `match dscp ef af41 af42` | Traffic ถูก Mark DSCP มาแล้วจาก Trust Boundary (Part 24) — ใช้บ่อยที่สุดที่ WAN Edge/Core |
| `match cos` | `match cos 5` | ยังอยู่ใน L2 Segment เดียวกัน (ก่อนถูก Route) |
| `match ip precedence` | `match ip precedence 5` | Legacy — อุปกรณ์เก่าที่ยังไม่รองรับ DSCP เต็มรูปแบบ |
| `match access-group` | `match access-group name CRITICAL-SUBNET` | Classify ตาม Source/Destination IP, Port (เมื่อยังไม่มี DSCP Mark หรือต้องการเจาะเฉพาะ subnet/port) |
| `match protocol` | `match protocol webex-meeting` | NBAR/NBAR2 — Classify ตาม Application Signature จริง ไม่พึ่ง Port number (Step 398) |
| `match input-interface` | `match input-interface GigabitEthernet0/0/2` | Classify ตาม Interface ที่ packet เข้ามา |
| `match qos-group` | `match qos-group 5` | Internal tag ที่อุปกรณ์ก่อนหน้าติดไว้ (เช่นจาก MPLS หรือ Platform QoS ภายใน) |

### match-any vs match-all — ตรรกะที่ต้องแยกให้ถูก

```
class-map match-any VOICE            class-map match-all SALES-CRITICAL
 match dscp ef                        match access-group name SALES-SUBNET
 match dscp cs5                       match dscp af21
        │                                      │
        ▼                                      ▼
  packet match ถ้า "DSCP = ef            packet match ต้องเป็น "ทั้งสองเงื่อนไข
  OR DSCP = cs5" (เงื่อนไขใดก็ได้)         พร้อมกัน" (มาจาก SALES subnet
                                          AND ถูก mark DSCP = af21)
  = OR Logic                             = AND Logic
```

- **`match-any`** (มักเป็น default ถ้าไม่ระบุในบาง IOS version — แต่ควรระบุชัดเจนเสมอเพื่อไม่ให้
  เข้าใจผิด): Packet เข้า Class นี้ถ้า match **เงื่อนไขใดเงื่อนไขหนึ่ง** ก็พอ — ใช้เวลาต้องการรวม
  หลายค่า DSCP ที่เป็น "ตระกูลเดียวกัน" ให้อยู่ Class เดียวกัน (เช่น AF41/AF42/AF43 ทั้งหมดคือ Video)
- **`match-all`**: Packet ต้อง match **ทุกเงื่อนไขพร้อมกัน** — ใช้เวลาต้องการเจาะจงมากขึ้น เช่น
  "เฉพาะ Traffic จาก Subnet SALES **และ** ถูก Mark เป็น AF21 เท่านั้น" (ตัดพวกที่มาจาก Subnet อื่น
  แม้จะมี DSCP เดียวกันออกไป)

### Worked Class-map: VOICE / VIDEO / CRITICAL-DATA / BEST-EFFORT

นี่คือ 4 Class ที่จะใช้ตลอด Part นี้ (สอดคล้องกับตาราง DSCP มาตรฐานจาก Part 24 Step 234):

```
WAN-EDGE-1(config)# class-map match-any VOICE
WAN-EDGE-1(config-cmap)# match dscp ef
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any VIDEO
WAN-EDGE-1(config-cmap)# match dscp af41
WAN-EDGE-1(config-cmap)# match dscp af42
WAN-EDGE-1(config-cmap)# match dscp af43
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any CRITICAL-DATA
WAN-EDGE-1(config-cmap)# match dscp af31
WAN-EDGE-1(config-cmap)# match dscp af21
WAN-EDGE-1(config-cmap)# match dscp cs3
WAN-EDGE-1(config-cmap)# exit
```

> **หมายเหตุ**: เราไม่ต้องสร้าง class-map สำหรับ **BEST-EFFORT** เพราะ IOS มี `class-default`
> ให้อัตโนมัติในทุก `policy-map` เสมอ — มันจับ traffic ทุกตัวที่ไม่ match VOICE/VIDEO/CRITICAL-DATA
> อัตโนมัติ (ส่วนใหญ่คือ HTTP/HTTPS ทั่วไป, Email, File transfer ที่ไม่ได้ Mark DSCP พิเศษ = DSCP
> เท่ากับ CS0/Default)

### ตัวอย่าง match-all: Classify เฉพาะ Traffic วิกฤตจาก VLAN 10 SALES

```
WAN-EDGE-1(config)# ip access-list standard SALES-SUBNET
WAN-EDGE-1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# exit
!
WAN-EDGE-1(config)# class-map match-all SALES-CRITICAL
WAN-EDGE-1(config-cmap)# match access-group name SALES-SUBNET
WAN-EDGE-1(config-cmap)# match dscp af21
WAN-EDGE-1(config-cmap)# exit
```

Class นี้จะจับเฉพาะ packet ที่ **ทั้งมาจาก Subnet 10.10.10.0/24 (SALES) และถูก Mark DSCP AF21
มาแล้ว** — ถ้ามาจาก Subnet อื่นที่ Mark AF21 เหมือนกัน (เช่น VLAN 30 SERVERS) จะ**ไม่**เข้า Class
นี้ เพราะไม่ผ่านเงื่อนไข `match-all` ทั้งสองอย่างพร้อมกัน

### `show class-map` — ตรวจสอบว่า class-map ถูกสร้างถูกต้อง

```
WAN-EDGE-1# show class-map VOICE
Class Map match-any VOICE (id 2)
  Match dscp ef (46)

WAN-EDGE-1# show class-map CRITICAL-DATA
Class Map match-any CRITICAL-DATA (id 4)
  Match dscp af31 (26)
  Match dscp af21 (18)
  Match dscp cs3 (24)
```

---

## Step 393 — Congestion Management: CBWFQ และ LLQ

### CBWFQ (Class-Based Weighted Fair Queuing)

CBWFQ แก้ปัญหาของ Priority Queuing แบบ Strict (Part 24 Step 238: Starvation) ด้วยการให้
**แต่ละ Class มี Queue ของตัวเอง พร้อม Bandwidth ขั้นต่ำที่การันตี (Guaranteed Minimum)**
เมื่อไม่มี Congestion ทุก Class ส่งได้เต็มที่ตามที่มี แต่เมื่อเกิด Congestion แต่ละ Class
**จะได้ Bandwidth อย่างน้อยตามที่กำหนดไว้เสมอ** ไม่มีใครถูก "อด" จนหมด (ต่างจาก Strict PQ)

```
WAN-EDGE-1(config)# policy-map bandwidth
WAN-EDGE-1(config-pmap)# class CRITICAL-DATA
WAN-EDGE-1(config-pmap-c)# bandwidth percent 30      ! การันตีขั้นต่ำ 30% ของ Bandwidth Interface
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class VIDEO
WAN-EDGE-1(config-pmap-c)# bandwidth percent 20      ! การันตีขั้นต่ำ 20%
```

**`bandwidth` (kbps ตรงๆ) vs `bandwidth percent` (%)**:

| รูปแบบ | ตัวอย่าง | ข้อดี | ข้อเสีย |
|---|---|---|---|
| `bandwidth 15000` | 15000 kbps ตรงตัว | ชัดเจน แน่นอน | ถ้าย้าย Policy ไปใช้กับ Interface ความเร็วอื่น ต้องคำนวณใหม่ทุกครั้ง |
| `bandwidth percent 30` | 30% ของ Bandwidth Interface ที่ Policy ถูก Apply | **Portable** — Policy เดียวกันใช้ได้กับ Interface ความเร็วต่างกันโดยไม่ต้องแก้ | ต้องรู้ Bandwidth ของ Interface จริง (หรือ Shape ไว้ก่อน — ดู Step 397) เพื่อคำนวณค่าจริง |

> **ข้อจำกัดสำคัญของ CBWFQ ล้วนๆ**: ต่อให้การันตี Bandwidth ขั้นต่ำได้ แต่ **ไม่มี Class ไหน
> ได้รับ "Strict Priority" จริง** — ทุก Class ยังต้องรอ "รอบ" ของตัวเองตาม Weighted Fair
> Scheduling อยู่ดี ซึ่งสำหรับ Data ทั่วไปไม่มีปัญหา (TCP ทนต่อ Delay/Jitter ได้) แต่สำหรับ **Voice
> (RTP/UDP) แม้จะมี Delay เพิ่มขึ้นเพียงเสี้ยววินาทีก็ทำให้เสียงสะดุดได้** — CBWFQ อย่างเดียวไม่พอ
> สำหรับ Voice ต้องเพิ่ม **LLQ**

### LLQ (Low Latency Queuing) — เพิ่ม Strict-Priority Queue เข้าไปใน CBWFQ

LLQ คือ CBWFQ **บวก** Queue พิเศษหนึ่งอัน (หรือหลายอันถ้าใช้ `priority level`) ที่เป็น **Strict
Priority แต่ถูกจำกัดเพดาน Bandwidth (Policed)** — คำสั่งคือ `priority` แทน `bandwidth`

```
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# class VOICE
WAN-EDGE-1(config-pmap-c)# priority percent 10       ! Strict Priority แต่เพดานไม่เกิน 10%
```

**ทำไม LLQ ต้องมี "เพดาน" (Policing) ในตัว ทั้งที่เป็น Strict Priority**: ถ้า Voice Queue ได้
Priority แบบไม่มีเพดานเลย (เหมือน PQ แบบเก่าใน Part 24 Step 238) แล้วมี traffic ผิดพลาดถูก Mark
เป็น EF จำนวนมาก (Misconfiguration หรือถูกโจมตี) จะเกิด **Starvation** กับ Class อื่นทันที — LLQ
จึงบังคับให้ Priority Queue มี **Policer ในตัวเสมอ** (มาจากค่า `priority percent`/`priority kbps`
ที่ตั้งไว้) — ถ้า Voice traffic เกินเพดานนี้จริง ส่วนที่เกินจะถูก **Drop ทันที** (ไม่ Queue รอ)
เพื่อป้องกัน Class อื่นไม่ให้ถูกอด — นี่คือความแตกต่างสำคัญที่สุดระหว่าง LLQ กับ Strict PQ ล้วนๆ

### Worked Policy-map เต็มรูปแบบ 4 Class (VOICE + VIDEO + CRITICAL-DATA + BEST-EFFORT)

```
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# class VOICE
WAN-EDGE-1(config-pmap-c)#  priority percent 10
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class VIDEO
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 20
WAN-EDGE-1(config-pmap-c)#  queue-limit 64
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class CRITICAL-DATA
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 30
WAN-EDGE-1(config-pmap-c)#  queue-limit 64
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  fair-queue
WAN-EDGE-1(config-pmap-c)#  queue-limit 64
WAN-EDGE-1(config-pmap-c)# exit
```

```
┌──────────────────────────────────────────────────────────────┐
│                     WAN-EDGE-QOS (LLQ + CBWFQ)                │
│                                                                │
│  ┌─────────────────────┐  Strict Priority (ออกก่อนเสมอ ถ้ามี)    │
│  │ VOICE — priority 10% │  ← มี Policer ในตัว ห้ามเกิน 10%        │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐  Weighted Fair (การันตีขั้นต่ำ 20%)       │
│  │ VIDEO — bw 20%       │                                     │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐  Weighted Fair (การันตีขั้นต่ำ 30%)       │
│  │ CRITICAL-DATA — bw 30%│                                    │
│  └─────────────────────┘                                     │
│  ┌─────────────────────┐  ส่วนที่เหลือ (~40%) + Fair-Queue ภายใน   │
│  │ class-default (BE)   │    ระหว่าง Flow ย่อยๆ                   │
│  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────┘
```

> **กฎ**: ผลรวมของ `bandwidth percent` (ไม่รวม `priority percent`) ต้อง **ไม่เกิน 75%** ของ
> Bandwidth Interface โดย default (IOS สงวน 25% ให้ Layer 2/Routing Overhead และ class-default
> เสมอ) — ถ้าต้องการเกิน 75% ต้องปรับด้วยคำสั่ง `max-reserved-bandwidth` ที่ระดับ Interface
> (ควรหลีกเลี่ยงถ้าไม่จำเป็นจริงๆ)

---

## Step 394 — Congestion Avoidance: WRED (Weighted Random Early Detection)

### ปัญหาที่ WRED แก้: TCP Global Synchronization

เมื่อ Queue ใกล้เต็ม กลไก Tail Drop แบบเดิม (drop packet ตัวสุดท้ายที่มาถึงเมื่อ Queue เต็มพอดี)
จะ Drop **TCP Flow หลายๆ ตัวพร้อมกันในเวลาใกล้เคียงกัน** เพราะทุก Flow เจอ Queue เต็มในจังหวะ
เดียวกัน — ทุก TCP Flow ที่ถูก Drop จะทำ **TCP Slow Start ใหม่พร้อมกันหมด** (ลด Window Size
ลงพร้อมกัน) ทำให้ Throughput รวมของ Link "ตกฮวบ" เป็นช่วงๆ แล้วค่อยไต่กลับขึ้นพร้อมกันอีกที
วนซ้ำเป็น pattern ฟันเลื่อย — นี่คือ **TCP Global Synchronization**

```
Tail Drop (ไม่มี WRED):
Throughput
    │     ╱╲          ╱╲          ╱╲
    │    ╱  ╲        ╱  ╲        ╱  ╲      ← Flow ทั้งหมดถูก Drop พร้อมกันตอน Queue เต็ม
    │   ╱    ╲      ╱    ╲      ╱    ╲       แล้วค่อยๆ ไต่ Window กลับพร้อมกันหมด (Synchronized)
    └──────────────────────────────────── เวลา

WRED (Random Early Drop ก่อน Queue เต็ม):
Throughput
    │  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~   ← Drop กระจายทีละ Flow ก่อน Queue เต็มจริง
    │                                        Flow ที่เหลือไม่โดน Drop ยังวิ่งต่อได้ปกติ
    └──────────────────────────────────── เวลา   → Throughput เฉลี่ยรวมสูงกว่าและเสถียรกว่า
```

### หลักการทำงานของ WRED

WRED ทำงานด้วยแนวคิด **"Drop ก่อนที่ Queue จะเต็มจริง โดยสุ่ม (Random) เลือก Flow บางส่วน"**
แทนที่จะรอให้ Queue เต็มแล้ว Drop ทุกอย่างที่มาถึง (Tail Drop) — ยิ่ง Queue เริ่มเต็มมากขึ้น
(Average Queue Depth สูงขึ้น) ความน่าจะเป็นที่จะ Drop packet ที่เข้ามาใหม่ก็ยิ่งสูงขึ้นตามลำดับ
ระหว่าง Minimum Threshold ถึง Maximum Threshold:

```
Drop Probability
     │
100% ┤                                        ┌────────  (เกิน Max Threshold = Tail Drop 100%)
     │                                       ╱
Mark │                                     ╱   ← ระหว่าง Min-Max: Drop Probability
Prob.│                                   ╱        เพิ่มแบบ Linear ตาม Mark Probability
     │                                 ╱          Denominator (ยิ่งเลขสูง ยิ่ง Drop ช้าลง)
   0%┤────────────────────────────────╱
     └─────────────────────┬─────────┬──────────────── Average Queue Depth
                     Minimum Threshold  Maximum Threshold
```

### DSCP-based WRED — ให้แต่ละ DSCP มี Threshold ต่างกัน (สอดคล้องกับ AF Drop Precedence)

จำจากตาราง DSCP Part 24 Step 234: **AF Class มี Drop Precedence ในตัว** (AFx1 < AFx2 < AFx3
ตามความสำคัญที่ควร Drop) — WRED แบบ DSCP-based ใช้ประโยชน์จากตรงนี้โดยตรง โดยกำหนด Threshold
ให้ AFx3 (Drop Precedence สูงสุด) เริ่ม Drop **เร็วที่สุด** (Threshold ต่ำสุด) และ AFx1 (Drop
Precedence ต่ำสุด) เริ่ม Drop **ช้าที่สุด** (Threshold สูงสุด)

```
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)#  random-detect dscp 0  20 40 10     ! CS0/Default: Threshold สูงสุด (Drop ท้ายสุด)
WAN-EDGE-1(config-pmap-c)#  random-detect dscp 10 22 40 10     ! AF11 (Drop Precedence ต่ำ)
WAN-EDGE-1(config-pmap-c)#  random-detect dscp 12 18 40 10     ! AF12 (Drop Precedence กลาง)
WAN-EDGE-1(config-pmap-c)#  random-detect dscp 14 14 40 10     ! AF13 (Drop Precedence สูง — Drop เร็วสุด)
```

รูปแบบคำสั่ง: `random-detect dscp <ค่า DSCP decimal> <Min Threshold> <Max Threshold> <Mark
Probability Denominator>` — หน่วย Threshold คือจำนวน Packet (หรือ Byte ถ้าใช้ตัวเลือกเพิ่มเติม
`random-detect dscp <val> <min> <max> <denom> byte-based`) ยิ่ง **Mark Probability Denominator**
สูง Drop Rate สูงสุดที่ Maximum Threshold จะยิ่ง**ต่ำลง** (ค่า default คือ 1/10 ที่ Maximum
Threshold — เขียนเป็น `10` ในคำสั่ง)

### WRED ใช้ได้ผลกับ Traffic ประเภทไหน — จำกฎนี้ให้แม่น

| ประเภท Traffic | ควรใช้ WRED? | เหตุผล |
|---|---|---|
| TCP-based (HTTP, Database, File Transfer, Email) | **ควรใช้อย่างยิ่ง** | TCP มี Congestion Control ในตัว (Slow Start, Congestion Avoidance) — เมื่อ Segment ถูก Drop TCP จะ**ลด Window Size แล้วส่งช้าลงเอง** — WRED "สื่อสาร" กับ TCP ผ่านการ Drop แบบสุ่มได้ผลจริง |
| UDP-based ที่ไม่มี Retransmit/Congestion Control (**Voice — RTP**) | **ห้ามใช้เด็ดขาด** | Voice ไม่มี Mechanism ตอบสนองต่อการ Drop เลย (ไม่ Slow ลง ไม่ Retransmit) — Drop packet เสียง = เสียงหายไปเลย ไม่ได้ช่วยลด Congestion แต่กลับทำให้คุณภาพเสียงแย่ลงโดยไม่จำเป็น |
| UDP-based Video Streaming ที่ไม่ Adaptive | ควรหลีกเลี่ยง หรือ Config ระมัดระวังมาก | เหตุผลเดียวกับ Voice — แต่บาง Video Codec สมัยใหม่ (Adaptive Bitrate) พอรับได้ในระดับหนึ่ง |

> **สรุปกฎเหล็ก**: WRED ใส่ไว้ที่ Class ที่เป็น TCP-heavy เท่านั้น (ใน Lab ของเราคือ
> **CRITICAL-DATA** และ **class-default/BEST-EFFORT**) — **ห้ามใส่ WRED ใน Class VOICE**
> เพราะ Voice ใช้ LLQ (`priority`) ที่มี Policer ของตัวเองอยู่แล้ว (Step 393) ถ้า Voice Queue
> ล้นจริง Policer จะ Drop ส่วนเกินทันที ไม่จำเป็นต้องมี WRED มาช่วยเพิ่ม — และ IOS เองก็
> **ปฏิเสธ (Reject) การ Config `random-detect` ในคลาสเดียวกันกับ `priority`** เพราะสองอย่างนี้
> ขัดแย้งกันในทางแนวคิด (LLQ Queue ไม่ได้ออกแบบให้มี Early-Drop Curve)

---

## Step 395 — Traffic Shaping: `shape average`/`shape peak` และ Token Bucket

### แนวคิด Traffic Shaping

**Shaping** คือการ "ควบคุมอัตราการส่ง (Rate)" ของ Traffic ให้ไม่เกินค่าที่กำหนด โดย **เก็บส่วนที่
เกินไว้ใน Buffer/Queue แล้วค่อยๆ ทยอยส่งออกทีหลัง** (ไม่ Drop ทันที) — ต่างจาก Policing ที่จะ
Drop/Remark ทันทีที่เกิน (Step 396) Shaping จึงทำให้ Traffic ที่ออกจาก Interface มีลักษณะ
**"เรียบ" (Smooth)** สอดคล้องกับอัตราที่ตกลงไว้กับผู้ให้บริการ WAN

### กรณีใช้งานจริง: Metro-Ethernet Handoff เป็น Gigabit แต่สัญญา CIR แค่ 50 Mbps

```
WAN-EDGE-1 ──Physical Port 1 Gbps──▶ ISP Metro-Ethernet ──Physical 1 Gbps──▶ ISP-RTR
                                     (แต่สัญญา CIR = 50 Mbps เท่านั้น)

ถ้า WAN-EDGE-1 ส่งเต็มความเร็ว Physical Port (สูงสุด 1 Gbps ตามที่ Interface ทำได้จริง):
  → ISP จะ Policer/Drop ส่วนที่เกิน CIR "แบบสุ่มไม่เลือกหน้า" ที่จุด Demarcation
  → Voice/Video ที่ถูก Drop ปนไปกับ Data ธรรมดา เพราะ ISP ไม่รู้จัก Priority ของเรา

ถ้า WAN-EDGE-1 "Shape" ตัวเองให้ไม่เกิน 50 Mbps ก่อนส่งออก:
  → เราควบคุม "ใครถูกหน่วง/ใครถูกส่งก่อน" เองที่ WAN-EDGE-1 ด้วย Policy-map ของเรา (LLQ/CBWFQ)
  → ISP ไม่ต้อง Drop อะไรเลย เพราะ Traffic ที่มาถึงไม่เกิน CIR ที่ตกลงไว้อยู่แล้ว
```

**นี่คือเหตุผลที่ต้องทำ Shaping ที่ WAN Edge Router ของเราเองเสมอ เมื่อ Physical Bandwidth
ของ Interface มากกว่า Contracted Rate ที่ตกลงกับ ISP** — ไม่ทำ Shaping = ยกอำนาจการเลือก
Drop Traffic ให้ ISP ทำแทน (ซึ่ง ISP ไม่รู้จัก QoS Policy ภายในขององค์กรเรา)

### Token Bucket Algorithm — กลไกเบื้องหลัง Shaping (และ Policing)

```
                    เติม Token ด้วยอัตรา CIR (bit/second) ตลอดเวลา
                              │
                              ▼
                    ┌───────────────────┐
                    │   Token Bucket      │   ขนาด Bucket = Bc (Committed Burst)
                    │   [●●●●●●●●○○]      │   (Be = Excess Burst สำหรับ shape peak เท่านั้น)
                    └───────────┬───────┘
                                │ packet ที่จะส่งต้อง "จ่าย" Token เท่ากับขนาดของมัน (bit)
                                ▼
                    มี Token พอ?──Yes──▶ ส่ง packet ทันที, หัก Token ออกจาก Bucket
                                │
                                No
                                ▼
                    Shaping: เก็บ packet ไว้ใน Queue รอ Token เติมพอ (ไม่ Drop)
                    Policing: Drop/Remark ทันที (ดู Step 396)
```

**ตัวแปรสำคัญ 3 ตัว**:

| ตัวแปร | ชื่อเต็ม | ความหมาย |
|---|---|---|
| **CIR** | Committed Information Rate | อัตราเฉลี่ยที่ต้องการควบคุม (bit ต่อวินาที) — ค่าหลักของ `shape average` |
| **Bc** | Committed Burst Size | ขนาด Token Bucket (bit) — ปริมาณที่อนุญาตให้ Burst ได้ในช่วงสั้นๆ โดยไม่ถูกหน่วง |
| **Tc** | Committed Time Interval | ช่วงเวลาที่ Bucket เติม Token เต็มหนึ่งรอบ = **Bc ÷ CIR** |

**ตัวอย่างคำนวณ**: `shape average 50000000` (CIR = 50 Mbps) พร้อม `Bc = 1,000,000 bit`
(ถ้าไม่ระบุ IOS จะคำนวณ Bc default ให้เองตาม Interval มาตรฐาน ~125 ms หรือค่าที่ Platform
กำหนด — ในที่นี้สมมติระบุเอง 1,000,000 bit เพื่อสอนสูตรคำนวณ):

```
Tc = Bc / CIR = 1,000,000 bit / 50,000,000 bps = 0.02 วินาที = 20 ms

ความหมาย: ทุกๆ 20 ms, WAN-EDGE-1 จะเติม Token ให้ Bucket ครบ 1,000,000 bit (125,000 byte)
          แล้วปล่อยให้ Interface ส่ง Burst ได้สูงสุด 125,000 byte ในช่วง 20 ms นั้น
          ก่อนต้องรอ Token เติมรอบใหม่ — ผลลัพธ์เฉลี่ยตลอดเวลาคือไม่เกิน 50 Mbps
```

### Syntax คำสั่ง Shaping

```
policy-map WAN-SHAPE-50M
 class class-default
  shape average 50000000                    ! CIR = 50 Mbps, Bc/Be คำนวณ default อัตโนมัติ
  shape average 50000000 1000000 1000000    ! ระบุ CIR, Bc (bit), Be (bit) เอง
  shape peak 60000000                        ! อนุญาต Burst เกิน CIR ได้ถึง Peak Rate โดยใช้ Be
```

- **`shape average`**: ควบคุมอัตราเฉลี่ยไม่ให้เกิน CIR อย่างเคร่งครัดในระยะยาว (ใช้บ่อยที่สุด —
  ตรงกับสัญญา CIR ของผู้ให้บริการ)
- **`shape peak`**: อนุญาตให้ Burst เกิน CIR ได้ชั่วคราวโดยใช้ Excess Burst (Be) แต่อัตราเฉลี่ย
  รวมสูงสุดจะไม่เกิน **CIR × (Bc+Be)/Bc** — ใช้เมื่อ Physical Bandwidth เผื่อไว้ให้ Burst ได้จริง
  และผู้ให้บริการยอมรับ Burst เกิน CIR เป็นครั้งคราว (ระวัง: ถ้า ISP Policer เข้มงวด `shape peak`
  อาจทำให้ ISP Drop Traffic ของเราเองที่ Demarcation ก็ได้)

---

## Step 396 — Traffic Policing: `police` Single-rate และ Dual-rate Three-Color

### แนวคิด Policing

**Policing** ควบคุม Rate เหมือน Shaping แต่ **ไม่มี Buffer เก็บส่วนที่เกิน** — packet ที่เกิน
เพดานจะถูก **Drop ทันที หรือ Remark (ลด DSCP ลง) ทันที** ไม่มีการหน่วงเวลาเก็บไว้ส่งทีหลังเลย
มักใช้ที่ **Ingress** (ขอบเขตทางเข้าเครือข่าย) เพื่อ "บังคับสัญญา" ว่าใครใช้ Bandwidth เกินที่
ตกลงไว้จะถูกจัดการทันที ไม่ให้เข้าไปสร้าง Congestion ในเครือข่ายส่วนที่เหลือ

### Single-Rate Policer (Two-Color: Conform/Exceed)

```
police cir 20000000 bc 1000000 conform-action transmit exceed-action drop
```

```
Token Bucket เดียว (CIR + Bc)
       │
       ▼
  มี Token พอ?
   ├─ Yes → CONFORM (สีเขียว) → conform-action (ปกติคือ transmit — ส่งผ่านตามปกติ)
   └─ No  → EXCEED (สีแดง)   → exceed-action (ปกติคือ drop — ทิ้งทันที)
```

### Dual-Rate Three-Color Policer (Conform/Exceed/Violate) — RFC 2698

ใช้ **สอง Token Bucket** (CIR/Bc และ PIR/Be) เพื่อให้มี 3 ระดับผลลัพธ์ — เหมาะกับ Contract ที่มี
ทั้ง "อัตราการันตี (CIR)" และ "อัตราสูงสุดที่ยอมให้ Burst ถึง (PIR — Peak Information Rate)"

```
police cir 50000000 bc 1500000 pir 75000000 be 1500000 conform-action transmit exceed-action set-dscp-transmit af11 violate-action drop
```

```
        Token Bucket #1 (CIR=50M, Bc)         Token Bucket #2 (PIR=75M, Be)
                    │                                      │
                    ▼                                      ▼
     ≤ CIR ────────▶ CONFORM (เขียว)                       │
     ระหว่าง CIR-PIR ─────────────────────▶ EXCEED (เหลือง) │
     > PIR ──────────────────────────────────────────────▶ VIOLATE (แดง)

  CONFORM  → conform-action transmit                 (ส่งผ่านปกติ ไม่แก้ไขอะไร)
  EXCEED   → exceed-action set-dscp-transmit af11     (ลด DSCP ลง แล้วยังส่งผ่าน — "ตราหน้า"
                                                        ไว้ให้ Hop ถัดไป Drop ก่อนถ้าเกิด Congestion)
  VIOLATE  → violate-action drop                       (เกินขนาดจริง — ทิ้งทันที ไม่ต้องพูดถึง)
```

### เปรียบเทียบ Shaping vs Policing — ต้องแยกให้ขาดสำหรับข้อสอบและงานจริง

| คุณสมบัติ | Traffic Shaping | Traffic Policing |
|---|---|---|
| จัดการ Traffic ส่วนเกินอย่างไร | **Queue/Buffer** ไว้ก่อน แล้วค่อยส่งทีหลัง (Delay) | **Drop หรือ Remark ทันที** ไม่มี Buffer |
| ผลต่อ Latency | เพิ่ม Delay/Jitter เล็กน้อย (เพราะ Buffer) แต่ไม่ Drop | ไม่เพิ่ม Delay เลย แต่ Drop ทันทีถ้าเกิน |
| ใช้ทิศทางไหนเป็นหลัก | **Egress** (ควบคุมสิ่งที่เราส่งออกให้เรียบ) | **Ingress** (บังคับสัญญาก่อนเข้าเครือข่าย) แต่ใช้ Egress ได้เช่นกัน |
| ทำงานร่วมกับ TCP อย่างไร | เป็นมิตรกับ TCP มากกว่า (Buffer แทน Drop ทำให้ TCP ไม่ต้อง Retransmit บ่อย) | รุนแรงกว่า — TCP ต้อง Retransmit ทุกครั้งที่ถูก Drop |
| ใช้เมื่อ | ต้องการปฏิบัติตามสัญญา CIR ของผู้ให้บริการ WAN โดยไม่ให้ ISP เป็นคน Drop เอง | ต้องการจำกัด/ลงโทษ Traffic ประเภทหนึ่งไม่ให้ใช้เกินโควตาที่กำหนด (เช่น Guest Wi-Fi) |

### Worked Example: Policing Guest Wi-Fi (VLAN 40) ให้ไม่เกิน 20 Mbps

```
DIST-SW3(config)# policy-map WIFI-POLICE
DIST-SW3(config-pmap)# class class-default
DIST-SW3(config-pmap-c)# police cir 20000000 bc 1000000 conform-action transmit exceed-action drop
DIST-SW3(config-pmap-c)# exit
DIST-SW3(config-pmap)# exit
!
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# service-policy input WIFI-POLICE
```

Traffic ทั้งหมดจาก VLAN 40 (WIFI, `10.10.40.0/24`) ที่เข้ามาที่ SVI ของ DIST-SW3 จะถูกจำกัดรวม
กันไม่เกิน 20 Mbps — ส่วนที่เกินถูก Drop ทันที ป้องกันไม่ให้ Guest/BYOD Wi-Fi แย่ง Bandwidth
ที่ควรสงวนไว้ให้ VOICE/CRITICAL-DATA ตอนไปถึง WAN Edge (จะเห็นผลลัพธ์เต็มรูปแบบใน Step 400)

---

## Step 397 — Hierarchical QoS (HQoS): Parent Shaping + Child Queuing

### ทำไมต้องมี HQoS

Step 393 (CBWFQ/LLQ) บอกวิธี "แบ่ง Bandwidth ระหว่าง Class" แต่ไม่ได้บอกว่า **"Bandwidth
รวมทั้งหมดที่มีให้แบ่งคือเท่าไหร่"** — ถ้า Physical Interface เป็น 1 Gbps แต่ CIR ตามสัญญาคือ
50 Mbps การใช้ `bandwidth percent` ตรงๆ กับ `policy-map` ที่ Apply บน Interface โดยตรงจะคำนวณ
เป็น % ของ **1 Gbps (ความเร็ว Physical)** ไม่ใช่ 50 Mbps (CIR จริง) ทำให้ Percentage ที่ตั้งไว้
ผิดเพี้ยนทั้งหมด — วิธีแก้คือ **ซ้อน Policy 2 ชั้น (Nested/Hierarchical)**:

```
┌─────────────────────────────────────────────────────────────┐
│  PARENT Policy-map — ทำหน้าที่ "กำหนดกรอบ" (Shape ก่อน)          │
│  policy-map WAN-SHAPE-50M                                     │
│   class class-default                                         │
│    shape average 50000000        ← จำกัดกรอบทั้งหมดไว้ที่ 50 Mbps  │
│    service-policy WAN-EDGE-QOS   ← แล้วค่อยส่งต่อให้ CHILD จัดการ  │
│                    │                                          │
│                    ▼                                          │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ CHILD Policy-map — ทำงาน "ภายใน" กรอบ 50 Mbps ที่ Shape ไว้│   │
│  │ policy-map WAN-EDGE-QOS                                  │   │
│  │  class VOICE          priority percent 10  (= 5 Mbps)    │   │
│  │  class VIDEO          bandwidth percent 20 (= 10 Mbps)   │   │
│  │  class CRITICAL-DATA  bandwidth percent 30 (= 15 Mbps)   │   │
│  │  class class-default  fair-queue            (= ~20 Mbps) │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

ตอนนี้ `bandwidth percent 30` ของ CRITICAL-DATA จะคำนวณจาก **50 Mbps (ค่าที่ Parent Shape
กำหนดไว้)** ไม่ใช่ 1 Gbps ของ Physical Port อีกต่อไป — Percentage ทุกตัวใน Child Policy ตรงกับ
ความเป็นจริงของ CIR ที่ตกลงกับ ISP แล้ว

### Full Worked Config: Nested Policy-map บน WAN-EDGE-1

```
! ---------- CHILD Policy: LLQ + CBWFQ + WRED (ทำงานภายในกรอบ 50 Mbps) ----------
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# class VOICE
WAN-EDGE-1(config-pmap-c)#  priority percent 10
WAN-EDGE-1(config-pmap-c)# class VIDEO
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 20
WAN-EDGE-1(config-pmap-c)#  queue-limit 64
WAN-EDGE-1(config-pmap-c)# class CRITICAL-DATA
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 30
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# class class-default
WAN-EDGE-1(config-pmap-c)#  fair-queue
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit

! ---------- PARENT Policy: Shape ทั้ง Envelope ให้เท่ากับ CIR สัญญา ----------
WAN-EDGE-1(config)# policy-map WAN-SHAPE-50M
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  shape average 50000000
WAN-EDGE-1(config-pmap-c)#  service-policy WAN-EDGE-QOS
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit

! ---------- Apply เฉพาะ PARENT เข้า Interface (Child ถูกเรียกซ้อนอยู่แล้ว) ----------
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)#  service-policy output WAN-SHAPE-50M
```

> **ข้อควรระวัง**: Apply `service-policy` เข้า Interface **เฉพาะ Parent เท่านั้น** — ห้าม Apply
> ทั้ง Parent และ Child แยกกันที่ Interface เดียวกัน (IOS จะ Error หรือทำงานผิดที่ตั้งใจ) Child
> จะถูกเรียกใช้งานโดยอัตโนมัติผ่านคำสั่ง `service-policy WAN-EDGE-QOS` ที่อยู่ **ภายใน** Class
> ของ Parent เท่านั้น

### เมื่อไหร่ต้องใช้ HQoS — จำสั้นๆ

| สถานการณ์ | ต้องใช้ HQoS? |
|---|---|
| Physical Interface Speed = CIR ตามสัญญาพอดี (เช่น Serial Link แบบเก่าที่ Physical = Contract) | ไม่ต้อง — ใช้ Policy-map ชั้นเดียวพอ (`bandwidth percent` คำนวณถูกอยู่แล้ว) |
| Physical Interface Speed **มากกว่า** CIR ตามสัญญาอย่างมีนัยสำคัญ (Metro-Ethernet, Fiber ที่ Provision เกิน CIR) | **ต้องใช้ HQoS** — Shape ที่ Parent ก่อน แล้วให้ Child ทำ Queuing ภายในกรอบนั้น |
| ต้องการ Per-Subscriber/Per-VLAN Shaping บน Interface เดียวที่มีหลาย Customer/Tenant (เช่น ISP PE Router) | ใช้ HQoS ระดับที่ซับซ้อนกว่า (Parent = per-VLAN shape, Child = per-class queuing ต่อ VLAN) — เกินขอบเขต Part นี้ แต่หลักการเดียวกัน |

---

## Step 398 — NBAR/NBAR2: Deep Packet Inspection สำหรับ Classification

### ทำไม Classify ด้วย Port Number อย่างเดียวไม่พอในยุคนี้

Class-map ที่ใช้ `match access-group` กับ ACL ที่เจาะจง TCP/UDP Port (เช่น Port 80 = HTTP)
เคยพอเพียงในยุคที่ Application แต่ละตัวใช้ Port ตายตัว แต่ปัจจุบัน **Web Application สมัยใหม่
(Webex, Teams, YouTube, Streaming ต่างๆ) ส่วนใหญ่วิ่งผ่าน Port 443 (HTTPS) เหมือนกันหมด**
— ถ้า Classify ด้วย Port 443 อย่างเดียว จะแยก Video Conference ออกจาก Web Browsing ทั่วไป
ไม่ได้เลย ทั้งที่ทั้งสองอย่างต้องการ QoS Treatment ต่างกันมาก

### NBAR/NBAR2 (Network-Based Application Recognition)

NBAR ใช้ **Deep Packet Inspection (DPI)** ตรวจ Payload และลักษณะการสื่อสารของ Traffic
(Signature Pattern, Heuristics) เพื่อระบุ **Application จริง** โดยไม่พึ่ง Port Number เพียง
อย่างเดียว — NBAR2 (เวอร์ชันปัจจุบัน) รองรับ Application มากกว่า 1,000 ตัว และ Cisco ปรับปรุง
Protocol Pack (PP) ใหม่เป็นระยะเพื่อรู้จัก Application ใหม่ๆ โดยไม่ต้อง Upgrade IOS ทั้งระบบ

```
┌───────────────────────────────────────────────────────────────┐
│  Packet HTTPS (Port 443) เข้ามาที่ Interface                        │
│                          │                                       │
│                          ▼                                       │
│           ┌───────────────────────────┐                          │
│           │  NBAR2 Deep Packet          │  ตรวจ TLS SNI, Payload   │
│           │  Inspection Engine          │  Pattern, Behavior       │
│           └─────────────┬─────────────┘                          │
│                          ▼                                       │
│      "นี่คือ webex-meeting"  หรือ  "นี่คือ facebook"  หรือ  "นี่คือ google-drive" │
│                          │                                       │
│                          ▼                                       │
│           ใช้ผลลัพธ์นี้ match ใน class-map ต่อได้เลย                     │
└───────────────────────────────────────────────────────────────┘
```

### คำสั่งใช้งาน `match protocol`

```
WAN-EDGE-1(config)# class-map match-any VIDEO-CONFERENCE
WAN-EDGE-1(config-cmap)# match protocol webex-meeting
WAN-EDGE-1(config-cmap)# match protocol webex-media
WAN-EDGE-1(config-cmap)# match protocol ms-teams
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# class-map match-any BUSINESS-CRITICAL-APPS
WAN-EDGE-1(config-cmap)# match protocol citrix
WAN-EDGE-1(config-cmap)# match protocol sap
WAN-EDGE-1(config-cmap)# exit
```

### NBAR Protocol Discovery — ใช้สำรวจ Traffic ก่อนออกแบบ Policy จริง

ก่อนออกแบบ `class-map`/`policy-map` จริงในองค์กร ควรเปิด **Protocol Discovery** เพื่อดูก่อนว่า
Interface นั้นมี Application อะไรวิ่งอยู่บ้าง และแต่ละตัวใช้ Bandwidth เท่าไหร่:

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# ip nbar protocol-discovery

WAN-EDGE-1# show ip nbar protocol-discovery interface GigabitEthernet0/0/0
GigabitEthernet0/0/0

  Last clearing of "show ip nbar protocol-discovery" counters 02:15:40

                            Input                    Output
                            -----                    ------
  Protocol                 Packet Count              Packet Count
                            Byte Count                Byte Count
                            5min Bit Rate (bps)       5min Bit Rate (bps)
  -----------------------  ------------------------  ------------------------
  webex-meeting             450000                    448000
                            288000000                 286000000
                            448000                     446000
  http                      1200000                   150000
                            1500000000                 18000000
                            2100000                    260000
  ms-teams                  320000                    318000
                            240000000                  238000000
                            380000                      378000
```

ข้อมูลนี้บอกทันทีว่า Application ไหนกิน Bandwidth มากที่สุด ช่วยตัดสินใจว่าควรสร้าง Class
เจาะจงให้ Application ไหนบ้างใน Policy-map จริง

### AVC (Application Visibility and Control) — แนะนำสั้นๆ

**AVC** คือ Framework ที่รวม 3 ส่วนเข้าด้วยกัน: **NBAR2** (Classification), **QoS/MQC**
(Policy Enforcement — ที่เรียนใน Part นี้ทั้งหมด), และ **Flexible NetFlow** (Visibility/
Reporting) — เมื่อสามส่วนนี้ทำงานร่วมกัน องค์กรจะเห็นภาพรวมทั้งหมดว่า **Application ไหนใช้งาน
เท่าไหร่ (Visibility)** และ**บังคับ Policy ตาม Application นั้นได้จริง (Control)** โดยไม่ต้อง
พึ่ง Hardware แยกสำหรับ Deep Packet Inspection เพิ่มเติม (NBAR2 ทำงานบน IOS ของ Router/Switch
ที่มีอยู่แล้ว) — AVC มักถูกจัดการรวมศูนย์ผ่าน Cisco DNA Center ในองค์กรขนาดใหญ่

---

## Step 399 — Verification: `show policy-map interface`, `show class-map`, `show policy-map`

### `show policy-map interface` — คำสั่งตรวจสอบที่สำคัญที่สุดของ MQC

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0

  Service-policy output: WAN-SHAPE-50M

    Class-map: class-default (match-any)
      1,520,000 packets, 1,824,000,000 bytes
      30 second offered rate 51,200,000 bps, drop rate 640,000 bps
      Match: any
      Queueing
      queue limit 64 packets
      (queue depth/total drops/no-buffer drops) 8/2100/0
      (pkts output/bytes output) 1,517,900/1,821,480,000
      shape (average) cir 50,000,000, bc 1,000,000, be 1,000,000
      target shape rate 50,000,000

      Service-policy : WAN-EDGE-QOS

        Class-map: VOICE (match-any)
          452,000 packets, 28,928,000 bytes
          30 second offered rate 456,000 bps, drop rate 0 bps
          Match: dscp ef (46)
          Priority: 10% (5,000 kbps), burst bytes 125,000
          (pkts output/bytes output) 452,000/28,928,000
          (no-buffer drops) 0

        Class-map: CRITICAL-DATA (match-any)
          610,000 packets, 780,800,000 bytes
          30 second offered rate 12,400,000 bps, drop rate 40,000 bps
          Match: dscp af31 (26)
          Match: dscp af21 (18)
          Match: dscp cs3 (24)
          Queueing
          queue limit 64 packets
          (queue depth/total drops/no-buffer drops) 3/210/0
          bandwidth 30% (15,000 kbps)
          Exp-weight-constant: 9 (1/512)
          Mean queue depth: 6 packets
            dscp   Transmitted    Random drop   Tail drop   Minimum   Maximum   Mark
                   pkts/bytes     pkts/bytes     pkts/bytes  threshold threshold probability
            af21   380000/486M    0/0            0/0          22        40        1/10
            af31   229790/294M    180/230K       30/38K       26        40        1/10

        Class-map: VIDEO (match-any)
          280,000 packets, 358,400,000 bytes
          30 second offered rate 9,800,000 bps, drop rate 0 bps
          Match: dscp af41 (34)
          Match: dscp af42 (36)
          Match: dscp af43 (38)
          bandwidth 20% (10,000 kbps)

        Class-map: class-default (match-any)
          178,000 packets, 220,000,000 bytes
          30 second offered rate 20,800,000 bps, drop rate 600,000 bps
          Match: any
          Queueing
          queue limit 64 packets
          (queue depth/total drops/no-buffer drops) 5/1890/0
          Fair-queue: per-flow queue limit 16 packets
          Exp-weight-constant: 9 (1/512)
          Mean queue depth: 5 packets
            dscp   Transmitted    Random drop   Tail drop   Minimum   Maximum   Mark
                   pkts/bytes     pkts/bytes     pkts/bytes  threshold threshold probability
            default 176110/218M   1890/2.3M      0/0          20        40        1/10
```

**สิ่งที่ต้องอ่านให้เป็นในผลลัพธ์นี้**:

| ส่วนของ Output | บอกอะไร |
|---|---|
| `offered rate` / `drop rate` | Traffic ที่เข้า Class นี้จริง (bps) และอัตราที่ถูก Drop — ถ้า `drop rate` สูงต่อเนื่อง แสดงว่า Class นั้น Under-provisioned |
| `(queue depth/total drops/no-buffer drops)` | ความยาว Queue ปัจจุบัน / จำนวนที่ถูก Drop สะสม (Tail Drop + WRED) / Drop เพราะ Memory Buffer ของ Router เต็ม (ปัญหาระดับ Hardware ไม่ใช่ QoS Policy) |
| `Priority: 10% (5,000 kbps)` | ยืนยันว่า LLQ ทำงาน และเพดาน Policer ในตัวคือเท่าไหร่จริง |
| ตาราง `dscp ... Random drop / Tail drop` | ผลของ WRED แยกตาม DSCP — เห็นชัดว่า AF13/AF23/AF33/AF43 (Drop Precedence สูง) ถูก Random Drop มากกว่า AF11/AF21/AF31 ตามที่ตั้งใจออกแบบ |

### `show class-map` และ `show policy-map` (ไม่ระบุ Interface)

```
WAN-EDGE-1# show policy-map WAN-EDGE-QOS
  Policy Map WAN-EDGE-QOS
    Class VOICE
      priority 10 (%)
    Class VIDEO
      bandwidth 20 (%)
      queue-limit 64
    Class CRITICAL-DATA
      bandwidth 30 (%)
      random-detect dscp-based
    Class class-default
      fair-queue
      random-detect dscp-based
```

คำสั่งนี้แสดง **Definition** ของ Policy-map เท่านั้น (ไม่มี Counter/Statistics) — ใช้ตรวจสอบ
ว่า Policy ถูก Config ตรงตามที่ตั้งใจหรือไม่ ก่อนที่จะ Apply เข้า Interface จริง

### Troubleshooting: Policy-map ไม่ทำงานตามที่ตั้งใจ — สาเหตุที่พบบ่อยที่สุด

| อาการ | สาเหตุที่พบบ่อย | วิธีแก้ |
|---|---|---|
| `show policy-map interface` ไม่แสดงอะไรเลย (`% No policy currently attached`) | ลืม `service-policy` เข้า Interface หรือ Apply ผิด Interface | ตรวจด้วย `show running-config interface <if>` ว่ามีบรรทัด `service-policy` จริง |
| Traffic ที่ควรเข้า Class เจาะจง (เช่น CRITICAL-DATA) กลับไปอยู่ที่ `class-default` เสมอ | **ลำดับ Class ใน Policy-map ผิด** — Class ที่กว้างกว่าถูกวางไว้**ก่อน**ที่แคบกว่า (First-Match-Wins ตาม Step 391) หรือ Class-map เขียนเงื่อนไข match ผิด/ACL อ้างชื่อผิด | ตรวจลำดับ Class ด้วย `show policy-map <name>` และตรวจ ACL ด้วย `show ip access-lists <name>` ว่ามี hit count จริง |
| Config `service-policy input` แต่ผลลัพธ์ไม่ตรงกับที่คาด (Marking หายไปตอน Egress) | Apply ผิดทิศทาง — `input` คือ Traffic ที่**เข้า**ผ่าน Interface นั้น, `output` คือที่**ออก** สอง Direction คนละความหมายกันคนละ Config | ยืนยันทิศทางที่ต้องการ Classify/Queue จริงๆ ก่อน Apply (ปกติ Shaping/Queuing ทำที่ Egress, Policing มักทำที่ Ingress) |
| `bandwidth percent`/`priority percent` คำนวณค่าออกมาผิดที่คาดไว้ | Policy-map ถูก Apply บน Interface โดยตรงที่ Physical Speed สูงกว่า Contract จริง (ลืมทำ HQoS ตาม Step 397) | ตรวจด้วย `show policy-map interface` ดูค่า kbps จริงที่คำนวณออกมา เทียบกับที่ต้องการ แล้วเพิ่ม Parent Shape ถ้าจำเป็น |
| `random-detect` ปฏิเสธไม่ให้ Config ใน Class เดียวกับ `priority` | LLQ Class ห้ามมี WRED (ตามกฎ Step 394) | ย้าย WRED ไปใส่ Class ที่เป็น CBWFQ (`bandwidth`) หรือ `class-default` เท่านั้น |

---

## Step 400 — Lab เต็มรูปแบบ: MQC Deployment บน WAN-EDGE-1 พร้อม Policing บน VLAN 40 WIFI

### เป้าหมาย Lab

รวมทุกกลไกจาก Step 391-399 เข้าเป็น Deployment เดียวที่ใช้งานได้จริง:

1. **WAN-EDGE-1** (Interface `GigabitEthernet0/0/0` ที่ต่อไป ISP-RTR, `203.0.113.0/30`):
   Hierarchical QoS — Parent `shape average 50000000` (ตรงกับ CIR สัญญา Metro-Ethernet) +
   Child LLQ (VOICE) + CBWFQ (VIDEO, CRITICAL-DATA) + WRED (CRITICAL-DATA, class-default)
2. **DIST-SW3 / DIST-SW4** (SVI `Vlan40`, `10.10.40.0/24`): Policing จำกัด Guest/BYOD Wi-Fi
   ไม่เกิน 20 Mbps เพื่อไม่ให้แย่ง Bandwidth บน WAN Circuit
3. Verify ด้วย `show policy-map interface` ภายใต้สภาวะ Congestion จำลอง (WAN Circuit ถูกใช้งาน
   เกิน 50 Mbps จาก Data traffic จำนวนมาก)

### Topology

```
                                  ISP-RTR
                                     │ 203.0.113.1
                                     │ /30 — Metro-Ethernet (Physical 1 Gbps, CIR สัญญา 50 Mbps)
                                     │ 203.0.113.2
                              ┌──────┴──────┐
                              │ WAN-EDGE-1   │  Gi0/0/0 ← Apply HQoS ที่นี่ (Egress ไป ISP)
                              └──────┬──────┘
                                     │ 10.10.254.1 /30
                                     │ 10.10.254.2
                              ┌──────┴──────┐
                              │  CORE-SW1    │
                              └──┬───────┬──┘
                          ┌──────┘       └──────┐
                    ┌─────┴─────┐         ┌─────┴─────┐
                    │ DIST-SW1/2 │         │ DIST-SW3/4 │  ← Apply Policing ที่ Vlan40 SVI (Ingress)
                    └───────────┘         └──┬────────┘
                                              │ Trunk (VLAN 30,40,99)
                                        ┌─────┴─────┐
                                        │ ACCESS-SW3/4│
                                        └──┬───────┬──┘
                                    VLAN30 │       │ VLAN40 (WIFI 10.10.40.0/24)
                                     SERVERS       Guest/BYOD Wi-Fi Client
```

### ส่วนที่ 1 — Running-config เต็มรูปแบบของ WAN-EDGE-1

```
hostname WAN-EDGE-1
!
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41
 match dscp af42
 match dscp af43
class-map match-any CRITICAL-DATA
 match dscp af31
 match dscp af21
 match dscp cs3
!
policy-map WAN-EDGE-QOS
 class VOICE
  priority percent 10
 class VIDEO
  bandwidth percent 20
  queue-limit 64
 class CRITICAL-DATA
  bandwidth percent 30
  random-detect dscp-based
  random-detect dscp 26 30 40 10
  random-detect dscp 18 24 40 10
  random-detect dscp 24 34 40 10
 class class-default
  fair-queue
  random-detect dscp-based
  random-detect dscp 0  20 40 10
!
policy-map WAN-SHAPE-50M
 class class-default
  shape average 50000000
  service-policy WAN-EDGE-QOS
!
interface Loopback0
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** WAN link to ISP-RTR (Metro-Ethernet, CIR 50M) **
 ip address 203.0.113.2 255.255.255.252
 service-policy output WAN-SHAPE-50M
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** LAN link to CORE-SW1 **
 ip address 10.10.254.1 255.255.255.252
 no shutdown
!
end
```

### ส่วนที่ 2 — Running-config ของ DIST-SW3 (Policing บน VLAN 40 WIFI)

```
hostname DIST-SW3
!
policy-map WIFI-POLICE
 class class-default
  police cir 20000000 bc 1000000 conform-action transmit exceed-action drop
!
interface Vlan40
 description ** WIFI — Guest/BYOD, policed to 20 Mbps **
 ip address 10.10.40.2 255.255.255.0
 service-policy input WIFI-POLICE
 no shutdown
!
end
```

> **หมายเหตุ**: `10.10.40.2` คือ Real IP ของ DIST-SW3 ตาม Convention ก่อน HSRP VIP
> (`10.10.40.1`) — ดู [`00-ip-address-plan.md`](00-ip-address-plan.md) — DIST-SW4 config
> เหมือนกันทุกประการโดยใช้ `10.10.40.3` (ทั้งสองตัวควร Config `service-policy input WIFI-POLICE`
> เหมือนกัน เพื่อให้ Guest Wi-Fi ถูกจำกัดเท่ากันไม่ว่าจะ Active/Standby HSRP ตัวไหนอยู่ในสถานะใด)

### ส่วนที่ 3 — Verification ภายใต้สภาวะ Congestion จำลอง

**สถานการณ์จำลอง**: มีการโทร Voice หลายสาย (VOICE), Video Conference (VIDEO), Database
Query จำนวนมาก (CRITICAL-DATA), และ Web Browsing/Download (BEST-EFFORT) พร้อมกันจนปริมาณรวม
เกิน 50 Mbps ของ CIR — ให้สังเกตว่า Class ไหนถูก Drop และ Class ไหนไม่ถูก Drop เลย

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0

  Service-policy output: WAN-SHAPE-50M

    Class-map: class-default (match-any)
      2,150,000 packets, 2,580,000,000 bytes
      30 second offered rate 68,500,000 bps, drop rate 18,500,000 bps
      Match: any
      Queueing
      queue limit 64 packets
      (queue depth/total drops/no-buffer drops) 64/94500/0
      shape (average) cir 50,000,000, bc 1,000,000, be 1,000,000
      target shape rate 50,000,000

      Service-policy : WAN-EDGE-QOS

        Class-map: VOICE (match-any)
          452,300 packets, 28,947,200 bytes
          30 second offered rate 458,000 bps, drop rate 0 bps        ← Voice ไม่ถูก Drop เลย
          Match: dscp ef (46)
          Priority: 10% (5,000 kbps), burst bytes 125,000
          (no-buffer drops) 0

        Class-map: CRITICAL-DATA (match-any)
          615,000 packets, 786,000,000 bytes
          30 second offered rate 15,600,000 bps, drop rate 620,000 bps  ← Drop เล็กน้อยจาก WRED
          Match: dscp af31 (26)
          Match: dscp af21 (18)
          Match: dscp cs3 (24)
          bandwidth 30% (15,000 kbps)
          Mean queue depth: 9 packets
            dscp   Transmitted    Random drop   Tail drop   Minimum   Maximum   Mark
            af21   380500/487M    620/790K       0/0          24        40        1/10
            af31   233200/298M    3400/4.3M      0/0          30        40        1/10

        Class-map: VIDEO (match-any)
          281,500 packets, 360,300,000 bytes
          30 second offered rate 10,100,000 bps, drop rate 0 bps
          bandwidth 20% (10,000 kbps)

        Class-map: class-default (match-any)
          801,200 packets, 1,404,752,800 bytes
          30 second offered rate 42,600,000 bps, drop rate 17,880,000 bps  ← Drop หนักสุด (ตามที่ตั้งใจ)
          Match: any
          Fair-queue: per-flow queue limit 16 packets
          Mean queue depth: 61 packets
            dscp    Transmitted    Random drop   Tail drop   Minimum   Maximum   Mark
            default 620100/1.08G   38200/48.7M   51900/66.3M  20        40        1/10
```

**การอ่านผลลัพธ์ (สรุปสิ่งที่ QoS Policy ทำสำเร็จ)**:

| Class | Offered Rate | Drop Rate | สรุปผล |
|---|---|---|---|
| VOICE | 458 kbps | **0 bps** | Strict Priority ทำงานสมบูรณ์ — Voice ไม่ถูกกระทบจาก Congestion เลยแม้ Link เกิน CIR ไปกว่า 18 Mbps |
| VIDEO | 10.1 Mbps | 0 bps | อยู่ในเพดาน `bandwidth percent 20` (10 Mbps) พอดี ยังไม่ต้อง Drop |
| CRITICAL-DATA | 15.6 Mbps | 620 kbps (~4%) | เกินเพดาน `bandwidth percent 30` (15 Mbps) เล็กน้อย — WRED เริ่ม Random Drop เพื่อบอก TCP ให้ชะลอ ก่อนที่จะเกิด Tail Drop รุนแรง |
| **class-default (BEST-EFFORT)** | 42.6 Mbps | **17.88 Mbps (~42%)** | รับ Drop หนักที่สุดตามที่ตั้งใจออกแบบ — ปกป้อง Class อื่นทั้งหมดโดยการยอมให้ Traffic ที่สำคัญน้อยที่สุดถูก Drop ก่อน |

```
DIST-SW3# show policy-map interface Vlan40 input

  Vlan40

  Service-policy input: WIFI-POLICE

    Class-map: class-default (match-any)
      2,850,000 packets, 3,648,000,000 bytes
      30 second offered rate 24,200,000 bps, drop rate 4,200,000 bps
      Match: any
      police:
          cir 20,000,000 bps, bc 1,000,000 bytes
        conformed 2,380,000 packets, 3,046,400,000 bytes; actions:
          transmit
        exceeded 470,000 packets, 601,600,000 bytes; actions:
          drop
        conformed 20,050,000 bps, exceed 4,150,000 bps
```

Guest/BYOD Wi-Fi (VLAN 40) พยายามใช้ประมาณ 24.2 Mbps แต่ถูก Policer จำกัดไม่ให้เกิน 20 Mbps
— ส่วนเกิน (~4.2 Mbps) ถูก Drop ทันทีที่ DIST-SW3 **ก่อนที่จะไปถึง WAN Circuit เลย** ทำให้
Bandwidth บน Metro-Ethernet CIR 50 Mbps เหลือสำหรับ VOICE/VIDEO/CRITICAL-DATA มากขึ้น

### สรุป Lab Step 400

| ส่วนประกอบ | อุปกรณ์/Interface | กลไกที่ใช้ | ผลลัพธ์ที่ Verify แล้ว |
|---|---|---|---|
| Hierarchical Shaping | WAN-EDGE-1 Gi0/0/0 (output) | `shape average 50000000` (Parent) | Traffic ที่ส่งออกไป ISP ไม่เกิน CIR สัญญา 50 Mbps เสมอ |
| LLQ (Voice) | WAN-EDGE-1 — Class VOICE | `priority percent 10` | Drop rate = 0 bps แม้ Link เกิน CIR ไปกว่า 30% |
| CBWFQ (Video/Critical-Data) | WAN-EDGE-1 — Class VIDEO/CRITICAL-DATA | `bandwidth percent 20`/`30` | Guaranteed Minimum Bandwidth ได้จริงตามที่ตั้งใจ |
| WRED (Congestion Avoidance) | WAN-EDGE-1 — CRITICAL-DATA, class-default | `random-detect dscp-based` | Drop กระจายตาม AF Drop Precedence ก่อน Tail Drop รุนแรง |
| Traffic Policing | DIST-SW3/DIST-SW4 Vlan40 (input) | `police cir 20000000` | Guest Wi-Fi ถูกจำกัดไม่เกิน 20 Mbps ก่อนเข้า Core/WAN |

---

## แบบฝึกหัดทวนความเข้าใจ Part 40

1. อธิบายโมเดล 3 ส่วนของ MQC (`class-map` → `policy-map` → `service-policy`) — แต่ละส่วนทำหน้าที่
   อะไร และทำไมต้องแยกกันทั้ง 3 ส่วน?

2. LLQ (`priority`) ต่างจาก CBWFQ (`bandwidth`) อย่างไร และเพราะเหตุใด Voice Class ต้องใช้ LLQ
   เท่านั้น ไม่ใช่ CBWFQ ธรรมดา?

3. ทำไม WRED ถึงห้ามใช้กับ Class ที่เป็น Voice (UDP/RTP) แต่ใช้ได้ผลดีกับ Class ที่เป็น TCP-heavy?

4. Traffic Shaping กับ Traffic Policing ต่างกันอย่างไรในการจัดการกับ Traffic ที่เกินเพดาน และ
   เพราะเหตุใดจึงมักใช้ Shaping ที่ Egress WAN Edge แต่ใช้ Policing ที่ Ingress?

5. เพราะเหตุใดต้องใช้ Hierarchical QoS (Parent Shape + Child Policy) เมื่อ Physical Bandwidth
   ของ Interface สูงกว่า CIR ตามสัญญา WAN — ถ้าใช้ Policy-map ชั้นเดียว Apply ตรงบน Interface
   จะเกิดปัญหาอะไร?

### เฉลย

1. `class-map` ทำหน้าที่ **Classification** — นิยามว่า Traffic แบบไหนเข้าเงื่อนไข (match dscp,
   cos, access-group, protocol ฯลฯ) `policy-map` ทำหน้าที่ **Action** — กำหนดว่าจะทำอะไรกับ
   Traffic ที่ถูก Classify แล้ว (Queue ด้วย bandwidth/priority, Shape, Police, Mark, WRED)
   `service-policy` ทำหน้าที่ **Apply** — ผูก Policy-map เข้ากับ Interface จริงพร้อมกำหนด
   ทิศทาง (input/output) ทั้ง 3 ส่วนแยกกันเพื่อให้ `class-map` และ `policy-map` **Reusable**
   ใช้ซ้ำได้กับหลาย Interface โดยไม่ต้องเขียนใหม่ทุกครั้ง

2. LLQ (`priority`) ให้ **Strict Priority Queue** ที่ถูกส่งออกก่อน Queue อื่นเสมอเมื่อมี packet
   รออยู่ พร้อม Policer ในตัวป้องกัน Starvation ส่วน CBWFQ (`bandwidth`) ให้ **Guaranteed
   Minimum Bandwidth** ผ่าน Weighted Fair Scheduling แต่ยังต้องรอ "รอบ" ของตัวเอง ไม่ใช่ Strict
   Priority — Voice (RTP/UDP) ไวต่อ Delay/Jitter สูงมาก (< 150 ms, < 30 ms ตาม Part 24) แม้
   Delay เพิ่มขึ้นเล็กน้อยจากการรอ Weighted Fair Scheduling ก็ทำให้เสียงสะดุดได้ จึงต้องใช้ LLQ
   ที่การันตี Delay ต่ำสุดจริงๆ

3. Voice/RTP เป็น UDP ที่ไม่มี Congestion Control หรือ Retransmit Mechanism เลย — WRED "สื่อสาร"
   กับ Congestion ผ่านการ Drop packet แบบสุ่มเพื่อให้ TCP ลด Window Size เอง แต่ Voice ไม่มี
   Mechanism ตอบสนองแบบนั้น การ Drop packet เสียงจึงแค่ทำให้เสียงหายไปโดยไม่ได้ช่วยลด Congestion
   เลย ในขณะที่ TCP-heavy Traffic ตอบสนองต่อการ Drop แบบสุ่มได้ดี (ลด Window แล้วส่งช้าลงเอง)
   ทำให้ WRED ป้องกัน TCP Global Synchronization ได้ผลจริง

4. Shaping เก็บส่วนที่เกินไว้ใน Buffer/Queue แล้วค่อยทยอยส่งทีหลัง (เพิ่ม Delay แต่ไม่ Drop)
   ส่วน Policing Drop/Remark ส่วนที่เกินทันที ไม่มี Buffer เลย — ใช้ Shaping ที่ Egress WAN Edge
   เพื่อควบคุมให้ Traffic ที่ส่งออกไม่เกิน CIR สัญญาอย่างนุ่มนวล (ไม่ให้ ISP เป็นคน Drop เอง)
   และใช้ Policing ที่ Ingress เพื่อบังคับสัญญา/โควตาทันทีก่อนที่ Traffic ที่เกินจะเข้าไปสร้าง
   Congestion ในเครือข่ายส่วนที่เหลือ (เช่น จำกัด Guest Wi-Fi ไม่ให้แย่ง Bandwidth)

5. `bandwidth percent`/`priority percent` คำนวณค่าจริง (kbps) จาก **Bandwidth ของ Interface
   ที่ Policy ถูก Apply** — ถ้า Apply ตรงบน Physical Interface ที่เร็วกว่า CIR มาก (เช่น 1 Gbps
   Physical แต่ CIR สัญญา 50 Mbps) Percentage ทุกตัวจะคำนวณจาก 1 Gbps ผิดเพี้ยนไปจากที่ตั้งใจ
   ทั้งหมด (เช่น 30% ของ 1 Gbps = 300 Mbps ซึ่งเกิน CIR จริงไปมาก) วิธีแก้คือทำ HQoS — ให้ Parent
   Policy-map `shape average` จำกัดกรอบให้เท่ากับ CIR ก่อน แล้วให้ Child Policy-map ทำงาน
   Percentage คำนวณจากกรอบที่ Shape ไว้แล้ว ไม่ใช่จาก Physical Speed ดิบ

---

## สรุป Part 40

Part นี้ทำตามสัญญาที่ให้ไว้ใน [Part 24](part-024-qos-fundamentals.md) อย่างเต็มรูปแบบ:
**MQC (Modular QoS CLI)** ทั้งโมเดล 3 ส่วน `class-map`/`policy-map`/`service-policy`, ตัวเลือก
`match` ทุกรูปแบบพร้อม match-any/match-all logic, **CBWFQ** (`bandwidth`/`bandwidth percent`)
และ **LLQ** (`priority`) สำหรับ Congestion Management, **WRED** (`random-detect dscp-based`)
สำหรับ Congestion Avoidance พร้อมกฎว่าใช้ได้กับ TCP-heavy Traffic เท่านั้น, **Traffic Shaping**
(`shape average`/`shape peak`) พร้อม Token Bucket Algorithm (Bc/Be/Tc), **Traffic Policing**
(`police` Single-rate/Dual-rate Three-Color) พร้อมเปรียบเทียบกับ Shaping อย่างชัดเจน,
**Hierarchical QoS (HQoS)** สำหรับสถานการณ์ที่ Physical Bandwidth มากกว่า CIR สัญญา, **NBAR/
NBAR2** สำหรับ Classification ด้วย Deep Packet Inspection แทนการพึ่ง Port Number, และปิดท้าย
ด้วย Lab เต็มรูปแบบที่รวมทุกกลไกเข้าด้วยกันบน WAN-EDGE-1 พร้อม Traffic Policing บน VLAN 40 WIFI
ที่ DIST-SW3/DIST-SW4 — Verify ด้วย `show policy-map interface` ภายใต้สภาวะ Congestion จำลอง
จนเห็นผลลัพธ์จริงว่า Voice ไม่ถูก Drop เลย ในขณะที่ BEST-EFFORT รับ Drop หนักที่สุดตามที่ออกแบบ

ทักษะ MQC ที่เรียนใน Part นี้เป็นพื้นฐานสำคัญที่จะใช้ซ้ำตลอดหลักสูตรระดับ CCNP/CCIE ต่อจากนี้
ไม่ว่าจะเป็นการทำ QoS บน MPLS VPN, SD-WAN Application-Aware Routing (ต่อยอดจาก Part 39), หรือ
Data Center QoS (Priority Flow Control สำหรับ Storage Traffic)

✅ **พร้อมสำหรับ Part 41**: เราจะเปลี่ยนหัวข้อไปที่ **Multicast Fundamentals** — วิธีที่เครือข่าย
ส่งข้อมูลจาก Source เดียวไปยัง Group ของ Receiver หลายตัวพร้อมกันอย่างมีประสิทธิภาพ (IGMP, PIM
Sparse Mode/Dense Mode, Rendezvous Point) ซึ่งเป็นอีกหนึ่งเสาหลักของ CCNP ENCOR ที่มักใช้งานคู่กับ
QoS จริงในสถานการณ์ Video Streaming/IPTV ขององค์กร

**ไปต่อ:** [Part 41 — Multicast Fundamentals →](part-041-multicast-fundamentals.md)
