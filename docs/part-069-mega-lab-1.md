# Part 69 — Full Troubleshooting Mega-Lab #1
**หลักสูตร Cisco CCNA → CCNP → CCIE (CCNP ENARSI) | Step 681–690 จาก 1000**

> ต่อจาก [Part 68 — QoS Troubleshooting](part-068-qos-troubleshooting.md) ที่ปิดท้ายการไล่ Debug
> ฟีเจอร์ทีละตัวแบบแยกโดเมน (EIGRP → OSPF → BGP → Redistribution → DMVPN/VPN → Infra Security →
> Infra Services → NAT → Path Control/PBR → Multicast → Wireless → Automation → QoS) มาตลอด 13
> Part ([Part 56](part-056-eigrp-troubleshooting.md)–[Part 68](part-068-qos-troubleshooting.md))
> **Part นี้คือจุดที่ทุกอย่างมาบรรจบกัน** ข้อสอบ ENARSI จริงและงานจริงในสนามไม่เคยแยกปัญหาให้เป็น
> "โดเมนละ 1 Ticket" อย่างเรียบร้อยแบบที่เราฝึกมา — เช้าวันจันทร์ที่มี Incident ใหญ่จริงจะมี **Ticket
> หลายใบพร้อมกัน คนละอาการ คนละทีมรับผิดชอบ และ (ที่อันตรายที่สุด) คนละ Root Cause กันจริงๆ** แม้จะ
> ดูเหมือนมาจากเหตุการณ์เดียวกันก็ตาม Part นี้จำลอง **Major Incident แบบ End-to-End เต็มรูปแบบ**
> ตั้งแต่ Help Desk รับสาย จนถึง Post-Incident Review (PIR) ที่ส่งให้ผู้บริหารอ่าน — ใช้ทักษะจาก
> Part 58 (BGP), Part 59 (Redistribution), Part 68 (QoS), Part 60 (DMVPN), Part 61 (Infra
> Security/CoPP) และ Part 66 (Wireless/RADIUS) พร้อมกันในเหตุการณ์เดียว

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 681 | Incident Kickoff — เช้าวันจันทร์ที่ Help Desk รับ Ticket พร้อมกัน 5 ใบ |
| 682 | Triage และ War-Room Process — จัดลำดับความสำคัญ แบ่งทีม เริ่ม Investigate อาการที่ 1 |
| 683 | Root Cause 1 — Internet ใช้ไม่ได้: BGP Prefix-list ตัด Default Route จาก WAN Edge |
| 684 | Root Cause 2 — เสียงสนทนามีปัญหา: QoS Trust Boundary หายไปจากการเปลี่ยน Switch |
| 685 | Root Cause 3 — สาขาหลุดเป็นช่วงๆ: DMVPN NHRP Dual-Hub ไม่สมบูรณ์ + สาย WAN Flap |
| 686 | Root Cause 4 — Wi-Fi Authen ไม่ผ่าน: CoPP บล็อก RADIUS หลัง Security Hardening |
| 687 | Cross-Cutting Analysis — เส้นด้ายร่วมของ 4 ปัญหา และ Change Management ที่ล้มเหลว |
| 688 | แผน Remediation เต็มรูปแบบ — ลำดับการแก้ไข Dependency และ Rollback Plan |
| 689 | Post-Incident Verification — Health-Check ครบทุกโดเมนหลังปิด Incident |
| 690 | Post-Incident Review (PIR) — เอกสารสรุป Incident แบบที่ Senior Engineer ต้องเขียนจริง |

---

## Step 681 — Incident Kickoff: เช้าวันจันทร์ที่ Help Desk รับ Ticket พร้อมกัน 5 ใบ

### 681.1 บริบท

เป็นเช้าวันจันทร์ปกติ พนักงานเริ่มทยอยเข้าออฟฟิศตั้งแต่ 07:45 ตามเวลาทำการ **ทีม NOC (Network
Operations Center)** เพิ่งเปลี่ยนเวรตอน 08:00 — และภายใน 15 นาทีถัดมา ระบบ Ticketing กับ Monitoring
ก็ระเบิดพร้อมกัน:

| Ticket # | เวลา | ผู้รายงาน / แผนก | อาการที่รายงาน | Priority เริ่มต้น (ตามที่ Help Desk ใส่) |
|---|---|---|---|---|
| INC-40231 | 08:02 | ทีม Sales (อาคาร A, VLAN 10 ผ่าน ACCESS-SW1) | "เข้าเว็บไม่ได้ทั้งชั้น เข้า CRM (SaaS บน Internet) ไม่ได้เลย" | P2 — Multiple Users |
| INC-40233 | 08:05 | ทีม Call Center (VLAN 20 Voice, ผ่าน ACCESS-SW2) | "คุยโทรศัพท์แล้วมีเสียงก้อง แตก และหลุดคำพูดเป็นช่วงๆ ลูกค้าบ่นเยอะมาก" | P2 — Customer Facing |
| INC-40235 | 08:08 | สาขาย่อย (BRANCH2-RTR, ผ่าน DMVPN) | "ระบบ ERP ที่สาขาหลุดการเชื่อมต่อเป็นช่วงๆ ประมาณทุก 15-20 นาที กลับมาเองแต่แล้วก็หลุดอีก" | P3 — Single Site |
| INC-40238 | 08:11 | ทีม Operations อาคาร C (Wi-Fi ผ่าน WLC-1/AP-2) | "มือถือ/Laptop ต่อ Wi-Fi ได้สัญญาณเต็ม แต่ค้างที่ 'กำลังตรวจสอบสิทธิ์' (Authenticating) ไม่เข้าเน็ตสักที เพิ่งเป็นตอนกะเช้าเข้างานพร้อมกันทีเดียว" | P2 — Building-Wide |
| ALERT-9042 | 08:14 | ระบบ Monitoring อัตโนมัติ (Syslog/SNMP) | Syslog Spike จาก **WAN-EDGE-1** จำนวนมากผิดปกติ ดูคล้าย Flapping ต่อเนื่อง | P4 — เดิมคิดว่าเป็น Noise |

### 681.2 ทำไม 5 Ticket นี้ถึง "ดูเหมือน" เรื่องเดียวกัน

Engineer ที่ On-call เห็น Timestamp ทั้ง 5 รายการอยู่ในช่วง **08:02-08:14 (ห่างกันไม่ถึง 12 นาที)**
สัญชาตญาณแรกของทุกคนคือ "นี่ต้องเป็นสาเหตุเดียวกันแน่ๆ" — นี่คือ **กับดักข้อแรกของ Incident ขนาดใหญ่**
ที่ Part นี้ต้องการสอนโดยเฉพาะ: **ความใกล้เคียงของเวลา (Time Correlation) ไม่ได้แปลว่ามีสาเหตุเดียวกัน
(Causal Correlation)** อาการทั้ง 5 อยู่คนละ Layer คนละ Technology กันโดยสิ้นเชิง:

```
                          ┌─────────────────────────────────────────┐
                          │              INTERNET / ISP-RTR           │
                          └──────────────────┬────────────────────────┘
                    ┌─────────────────────────┴─────────────────────────┐
                    │        WAN-EDGE-1 (Primary) ⚠ Syslog Spike         │  <- Ticket ALERT-9042
                    │        WAN-EDGE-2 (Backup)                         │     + Ticket INC-40231
                    └──────────────────┬──────────────────────────────────┘
                                       │  DMVPN Tunnel0 (Phase 3, Dual-Hub)
                    ┌──────────────────┼──────────────────────────────────┐
                    │                  │                                  │
              ┌─────┴─────┐     ┌──────┴──────┐                   ┌───────┴────────┐
              │ CORE-SW1   │     │ CORE-SW2     │                   │  BRANCH2-RTR    │  <- Ticket INC-40235
              └─┬───────┬─┘     └─┬─────────┬──┘                   │  (สาขาย่อย)      │
        ┌────────┘       └──────┐ │         │                       └────────────────┘
   ┌────┴────┐             ┌────┴──┴───┐ ┌──┴────────┐
   │ DIST-SW1 │            │  DIST-SW2  │ │ DIST-SW3/4 │
   └────┬────┘             └─────┬─────┘ └─────┬──────┘
   ┌────┴────┐             ┌─────┴─────┐  ┌────┴─────────────┐
   │ACCESS-SW1│            │ ACCESS-SW2 │  │ WLC-1 + AP-2      │  <- Ticket INC-40238
   │ VLAN 10  │            │ VLAN 20    │  │ (อาคาร C)          │
   │ Sales    │            │ Voice/Phone│  └───────────────────┘
   └──────────┘            └─────┬─────┘
        │                        │  <- Ticket INC-40233
     INC-40231                IP Phone
```

### 681.3 ลำดับความสำคัญเบื้องต้น (ก่อนรู้ Root Cause)

ทีม NOC ใช้หลัก **Business Impact ก่อน Technical Severity** ในการจัดลำดับตอนที่ยังไม่รู้สาเหตุ:

| ลำดับ | Ticket | เหตุผลที่จัดลำดับนี้ |
|---|---|---|
| 1 | INC-40231 (Internet) + INC-40238 (Wi-Fi) | กระทบผู้ใช้จำนวนมากพร้อมกัน กระทบรายได้ทางตรง (Sales ใช้ CRM, Operations ใช้ Wi-Fi ทำงาน) |
| 2 | INC-40233 (Voice) | กระทบลูกค้าโดยตรง (Call Center) แต่จำนวนสายที่กระทบยังจำกัดกว่า |
| 3 | INC-40235 (Branch) | กระทบ 1 สาขา ผู้ใช้น้อยกว่า แต่ระบบ ERP มีผลต่อ Operation รายวัน |
| 4 | ALERT-9042 (Syslog) | ยังไม่รู้ว่ากระทบ User จริงหรือไม่ — อาจเป็นแค่ Noise หรืออาจเป็น "ต้นตอ" ของ INC-40231 |

> **บทเรียนสำคัญ**: ที่จุดนี้ **ห้ามสรุปเด็ดขาด** ว่า ALERT-9042 (Syslog Spike จาก WAN-EDGE-1) คือ
> สาเหตุของทุกอย่าง แม้จะน่าสงสัยที่สุดเพราะเป็น Alert เดียวที่ "ดูเหมือนสาเหตุ" — ต้อง Treat ทุก
> Ticket เป็น Hypothesis ที่ต้องพิสูจน์แยกกันก่อน แล้วค่อยมาดูว่าอันไหนเชื่อมกันจริง (Step 687)

Incident Commander (IC) ประกาศ **Sev-1 Major Incident** และเปิด War Room ทันที — รายละเอียดกระบวนการ
อยู่ใน Step 682

---

## Step 682 — Triage และ War-Room Process

### 682.1 โครงสร้างทีม War Room (ตาม ITIL Major Incident Process)

| Role | ผู้รับผิดชอบ | หน้าที่ |
|---|---|---|
| **Incident Commander (IC)** | Network Manager (on-call) | ตัดสินใจ Priority, ประกาศ Severity, สั่งปิด/เปิด Incident, สื่อสารกับผู้บริหาร |
| **Technical Lead — Routing/WAN** | Senior Network Engineer A | ดูแล Ticket INC-40231, ALERT-9042, INC-40235 (Layer 3/WAN/Overlay) |
| **Technical Lead — Voice/QoS** | Senior Network Engineer B | ดูแล Ticket INC-40233 |
| **Technical Lead — Security/Wireless** | Security Engineer + Wireless Engineer (ร่วมกัน) | ดูแล Ticket INC-40238 |
| **Scribe** | Junior Engineer | จด Timeline ทุกการกระทำ พร้อม Timestamp ("ใครทำอะไร เมื่อไหร่ ผลลัพธ์คืออะไร") — สำคัญมากสำหรับ PIR ใน Step 690 |
| **Communications Lead** | IT Service Desk Manager | อัปเดตผู้ใช้/ผู้บริหารทุก 15 นาที ผ่าน Status Page/Email — **ไม่ให้ Technical Lead เสียเวลาตอบคำถามซ้ำๆ** |

### 682.2 Severity Matrix ที่ใช้ตัดสินใจ

| Severity | นิยาม | ตัวอย่างในเหตุการณ์นี้ | Communication Cadence |
|---|---|---|---|
| Sev-1 | หลายระบบ/หลายแผนกกระทบพร้อมกัน มีผลต่อรายได้/ลูกค้าโดยตรง | **สถานะปัจจุบันของเรา** — Internet, Voice, Wi-Fi, Branch กระทบพร้อมกัน | ทุก 15 นาที |
| Sev-2 | ระบบเดียวกระทบ แต่จำนวนผู้ใช้มาก | — | ทุก 30 นาที |
| Sev-3 | กระทบเฉพาะกลุ่มเล็ก/สาขาเดียว | ถ้ามีแค่ INC-40235 เพียงใบเดียว | ทุก 1 ชั่วโมง |
| Sev-4 | ไม่กระทบ User โดยตรง (Alert เชิง Preventive) | ถ้า ALERT-9042 ไม่มี Ticket อื่นตามมา | ตามรอบปกติ |

เนื่องจากมี Ticket P2 อยู่ 3 ใบพร้อมกัน (Internet, Voice, Wi-Fi) IC จึงประกาศ **Sev-1** ทันที และตั้ง
**War-Room Bridge Call** ให้ Technical Lead ทั้ง 3 สายเข้าประชุมพร้อมกัน — Sync ทุก 15 นาทีตาม Matrix

### 682.3 กฎเหล็กของ War Room ที่ IC ประกาศตั้งแต่นาทีแรก

1. **แต่ละ Ticket มี Owner เดียว** — ห้าม 2 คนแก้ปัญหาเดียวกันพร้อมกันโดยไม่บอกกัน (ป้องกัน Config
   ชนกัน ซึ่งจะทำให้ Diagnose ยากขึ้นไปอีก)
2. **ห้าม Rollback/แก้ Config อะไรก่อนแจ้ง War Room** — แม้จะมั่นใจ 100% ก็ตาม เพราะการแก้จุดหนึ่งอาจ
   ไปกระทบการ Diagnose ของอีกทีมที่กำลังดู Symptom คาบเกี่ยวกัน
3. **Scribe บันทึกทุกคำสั่งที่รันจริงบนอุปกรณ์ Production** — ไม่ใช่แค่สรุปคำพูด
4. **ห้ามสมมติว่า Root Cause เดียวอธิบายทุกอาการ** จนกว่าจะพิสูจน์ได้จริงในแต่ละ Ticket แยกกัน

### 682.4 เริ่ม Investigate อาการที่ 1 — Internet ใช้ไม่ได้ (INC-40231)

Technical Lead A เริ่มจากจุดที่ใกล้ผู้ใช้ที่สุดแล้วไล่เข้าไปตาม Layer (Bottom-Up จาก User มองเห็น):

```
DIST-SW1# ping 8.8.8.8
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)

DIST-SW1# show ip route 0.0.0.0
% Network not in table
```

**ไม่มี Default Route ในตาราง Routing ของ DIST-SW1 เลย** — นี่ไม่ใช่แค่ Ping ไม่ผ่าน แต่ **ไม่มี Path
ไป Internet อยู่ในระบบเลยตั้งแต่ต้น** ขยับไปเช็คที่ CORE-SW1 (ต้นทางที่ DIST-SW1 เรียน OSPF มา):

```
CORE-SW1# show ip route 0.0.0.0
% Network not in table

CORE-SW1# show ip ospf database external | include 0.0.0.0
                (พบว่าไม่มี LSA Type-5 default ปรากฏใน Database เลย)
```

Default Route หายไปตั้งแต่ระดับ **OSPF Database** แล้ว — แปลว่าปัญหาไม่ได้อยู่ที่ Campus (OSPF Area
0/1/2) เพราะฝั่งนี้แค่ "ไม่ได้รับ LSA" ไม่ได้ Filter อะไรเอง ต้องขยับไปดูต้นทางที่ Origin LSA
นี้จริงๆ คือ **WAN-EDGE-1 / WAN-EDGE-2** — และนี่คือจุดที่เชื่อมกับ **ALERT-9042 (Syslog Spike จาก
WAN-EDGE-1)** พอดี ทำให้ Technical Lead A ตัดสินใจไปตรวจ WAN-EDGE-1 ก่อนเป็นอันดับแรก → Step 683

---

## Step 683 — Root Cause 1: BGP Prefix-list ตัด Default Route จาก WAN Edge

### 683.1 ตรวจ OSPF ฝั่ง WAN-EDGE-1 ก่อน

```
WAN-EDGE-1# show ip ospf database external
            OSPF Router with ID (1.1.1.21) (Process ID 1)

               Type-5 AS External Link States

  (ตาราง Type-5 LSA ว่างเปล่า — WAN-EDGE-1 ไม่ได้ Originate Default LSA เข้า OSPF เลย)

WAN-EDGE-1# show ip route ospf
            (ไม่มีปัญหา OSPF ฝั่ง Campus เลย — Router-ID อื่นๆ เห็นครบ)

WAN-EDGE-1# show run | section router ospf 1
router ospf 1
 router-id 1.1.1.21
 default-information originate metric 1 metric-type 1
 network 10.10.254.0 0.0.0.3 area 0
```

`default-information originate` **ไม่มี `always`** ตามที่ทีมแก้ไว้ตั้งแต่ [Part 55 Step
55x](part-055-encor-capstone-lab.md) (บทเรียน ENCOR Capstone: `always` เป็นกับดักที่ทำให้ Router
โฆษณา Default ทั้งที่ตัวเองไม่มีทางไป Internet จริง) — หมายความว่า **คำสั่งนี้ถูกต้องอยู่แล้ว** แต่
มันจะ Originate ก็ต่อเมื่อ **WAN-EDGE-1 มี Default Route ของตัวเองอยู่ใน RIB จริง** ไปดูต่อ:

```
WAN-EDGE-1# show ip route 0.0.0.0
% Network not in table
```

**WAN-EDGE-1 เองก็ไม่มี Default Route ของตัวเอง** — สอดคล้องกับพฤติกรรมของคำสั่ง `default-information
originate` (ไม่มี `always`) ที่ทำงาน**ถูกต้องตามที่ Config ไว้** เพียงแต่มันไม่มีอะไรให้ Originate
ต่อ Root Cause จึงต้องขยับไปอีกขั้นหนึ่ง คือ **ทำไม WAN-EDGE-1 ไม่มี Default Route** — Router นี้ควร
เรียน Default ผ่าน eBGP จาก ISP-RTR

### 683.2 ตรวจ BGP Table

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000   48210   48195    91422    0    0 3d18h          3
1.1.1.22        4 65001    9821    9815     8842    0    0 3d18h          412

WAN-EDGE-1# show ip bgp
BGP table version is 91422, local router ID is 1.1.1.21
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
   Network          Next Hop            Metric LocPrf Weight Path
*> 203.0.113.8/30   203.0.113.1              0             0 65000 i
*> 198.51.100.0/24   203.0.113.1              0             0 65000 i
*> 192.0.2.0/24      203.0.113.1              0             0 65000 i
   (ไม่มี 0.0.0.0/0 ปรากฏในตารางเลย)
```

Session BGP กับ ISP-RTR **Established ปกติ** (`PfxRcd = 3`) ไม่ใช่ปัญหา Neighbor แบบ Part 58 —
แต่ ISP ส่ง Prefix มา 3 เส้น (ไม่มี Default) หรือเราได้รับแล้วแต่ถูก Filter ทิ้ง? ตรวจสอบ Filter
ขาเข้าที่ Neighbor Statement:

```
WAN-EDGE-1# show run | section neighbor 203.0.113.1
 neighbor 203.0.113.1 remote-as 65000
 neighbor 203.0.113.1 route-map ISP-IN-FILTER in
```

พบ Route-map `ISP-IN-FILTER` ผูกอยู่กับทิศทาง **`in`** (ขาเข้า) — ตรวจสอบ Route-map นี้:

```
WAN-EDGE-1# show route-map ISP-IN-FILTER
route-map ISP-IN-FILTER, permit, sequence 10
  Match clauses:
    ip address prefix-lists: ISP-IN-PFX
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes

WAN-EDGE-1# show ip prefix-list detail ISP-IN-PFX
ip prefix-list ISP-IN-PFX:
   count: 1, range entries: 1, sequences: 5 - 5, refcount: 2
   seq 5 permit 0.0.0.0/0 ge 1
```

### 683.3 Root Cause — `ge 1` ตัด Prefix `0.0.0.0/0` เองออกจากการ Match

นี่คือ Bug คลาสสิกของ Prefix-list ที่สอนไว้ใน [Part 58 Step 576](part-058-bgp-troubleshooting.md)
(Prefix-list/AS-Path Filter บล็อก Route โดยไม่ตั้งใจจาก `ge`/`le` ผิด) — ความหมายของ
`permit 0.0.0.0/0 ge 1` คือ **"Permit Prefix ใดๆ ที่เป็น Subnet ของ 0.0.0.0/0 ซึ่งมี Prefix-length
ตั้งแต่ 1 บิตขึ้นไป"** — เงื่อนไข `ge`/`le` เปลี่ยนการ Match จาก "Prefix ตัวนี้ตรงตัว" ไปเป็น "Prefix
ที่เป็น Subnet ในช่วง Length ที่กำหนด" **โดยไม่รวม Prefix ต้นฉบับ (Length 0) เข้าไปด้วย** ดังนั้น
Prefix ที่ Match กับบรรทัดนี้จริงคือ `/1` ถึง `/32` ที่เป็น Subnet ของ `0.0.0.0/0` เท่านั้น — **ตัว
`0.0.0.0/0` (Length 0) เองไม่ Match เงื่อนไขนี้** และไม่มีบรรทัดอื่นใน Prefix-list เลย จึงตกไปโดน
**Implicit Deny ท้าย Prefix-list** — นี่คือสาเหตุที่ตาราง BGP เห็น Prefix อื่นๆ (`/24`, `/30`) ผ่าน
ปกติ (เพราะมี Length ≥ 1 อยู่แล้ว) แต่ Default Route ตัวเดียวถูกบล็อกอย่างเงียบๆ

### 683.4 ประวัติของการเปลี่ยนแปลง

```
WAN-EDGE-1# show archive config differences

  ! เปรียบเทียบ Running-config ปัจจุบันกับ Checkpoint ล่าสุดก่อน Maintenance Window
  + ip prefix-list ISP-IN-PFX seq 5 permit 0.0.0.0/0 ge 1
  + route-map ISP-IN-FILTER permit 10
  +  match ip address prefix-list ISP-IN-PFX
  + neighbor 203.0.113.1 route-map ISP-IN-FILTER in
```

การเปลี่ยนแปลงนี้ถูก Push เข้าไปเมื่อ **วันศุกร์ 16:00** ตาม Change Ticket "Q3 Security Hardening —
BGP Inbound Filtering" มีเจตนาดี (ป้องกัน ISP Leak Prefix แปลกปลอม/Bogon เข้ามาในตาราง BGP ของเรา)
แต่เขียน Prefix-list ผิด Syntax ที่ตั้งใจไว้จริงคือ **"Permit ทุกอย่างที่ ISP ส่งมา ยกเว้น Prefix ที่
กว้างเกินไป (สั้นกว่า /8)"** ซึ่งควรเขียนเป็นคนละบรรทัดสำหรับ Default Route:

```
ip prefix-list ISP-IN-PFX seq 5 permit 0.0.0.0/0
ip prefix-list ISP-IN-PFX seq 10 permit 0.0.0.0/0 ge 8
```

### 683.5 การแก้ไข (แจ้ง War Room ก่อนทำตามกฎ Step 682.3)

```
WAN-EDGE-1(config)# ip prefix-list ISP-IN-PFX seq 5 permit 0.0.0.0/0
WAN-EDGE-1(config)# ip prefix-list ISP-IN-PFX seq 10 permit 0.0.0.0/0 ge 8
WAN-EDGE-1(config)# end
WAN-EDGE-1# clear ip bgp 203.0.113.1 soft in
```

> **หมายเหตุสำคัญ**: ใช้ `clear ip bgp ... soft in` (Soft Reset ขาเข้าอย่างเดียว) **ไม่ใช่ `clear ip
> bgp *`** เพื่อไม่ให้ Session อื่นๆ (รวม iBGP ไปยัง WAN-EDGE-2) หลุดไปด้วยโดยไม่จำเป็น — หลักการ
> Blast Radius ที่ต้องคุมให้แคบที่สุดเสมอเวลาแก้ไขระหว่าง Incident ที่ยังเปิดอยู่

### 683.6 Verify

```
WAN-EDGE-1# show ip bgp | include 0.0.0.0
*> 0.0.0.0/0        203.0.113.1              0             0 65000 i

WAN-EDGE-1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "bgp 65001", distance 20, metric 0
  Routing Descriptor Blocks:
  * 203.0.113.1, from 203.0.113.1

WAN-EDGE-1# show ip ospf database external | include 0.0.0.0
0.0.0.0         1.1.1.21        1          0x80000003 0x00d1e2

CORE-SW1# show ip route 0.0.0.0
O*E1  0.0.0.0/0 [110/1] via 10.10.254.1, 00:00:12, GigabitEthernet1/0/1

DIST-SW1# ping 8.8.8.8
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/11/19 ms
```

Internet กลับมาใช้งานได้ทันทีที่ Default Route ไหลกลับเข้าสู่ OSPF Area 0 → Area 1/2 ตามปกติ

### 683.7 ปิดปริศนา ALERT-9042 (Syslog Spike)

```
WAN-EDGE-1# show logging | include BGP
%BGP-5-ADJCHANGE: neighbor 203.0.113.1 Up
```

ตรวจสอบ Log ย้อนหลังพบว่า Syslog Spike ที่ Monitoring แจ้งมาไม่ใช่การโจมตีหรือ Hardware Fail แต่คือ
**OSPF LSA Flooding ซ้ำๆ** ที่เกิดจาก Router หลายตัวใน Area 0/1/2 พยายาม Re-flood Database ทุกครั้งที่
Neighbor Timer ตรวจพบว่า External LSA หายไปเป็นระยะ (Aging/Retransmission ปกติของ LSA ที่ไม่มี
Originator ส่ง Refresh เข้ามาเรื่อยๆ) — เป็น**ผลลัพธ์**ของ Root Cause 1 ไม่ใช่ปัญหาแยก — Ticket
ALERT-9042 ปิดพร้อมกับ INC-40231 ทันที

**INC-40231 และ ALERT-9042 ปิด — เวลารวมในการ Diagnose + แก้ไข: 22 นาที**

---

## Step 684 — Root Cause 2: QoS Trust Boundary หายไปจากการเปลี่ยน Switch (ไม่เกี่ยวกับ Root Cause 1)

### 684.1 ทำไมต้องเช็คก่อนว่า "ไม่เกี่ยวกัน" — กับดักที่อันตรายที่สุดของ Incident ใหญ่

Technical Lead B (ดูแล Voice) ได้ยินข่าวดีว่า Internet กลับมาแล้ว และมีความคิดแรกผ่านหัวว่า **"บางที
ปัญหา Voice อาจจะหายไปเองด้วยเหมือนกัน เพราะ Default Route หายไปพร้อมกัน"** — นี่คือสมมติฐานที่
**สมเหตุสมผลแต่ต้องพิสูจน์ ไม่ใช่เดา** เพราะ:

1. Voice Traffic ภายในองค์กร (โทรระหว่างสาย Extension) **ไม่เดินทางออก Internet เลย** — วิ่งอยู่ใน
   VLAN 20 ภายใน Campus ล้วนๆ ไม่ผ่าน WAN-EDGE
2. ถ้า Root Cause 1 เป็นสาเหตุของปัญหา Voice ด้วยจริง Voice ควรกลับมาเป็นปกติทันทีที่ Internet กลับมา
   — ต้อง**ทดสอบสมมติฐานนี้จริง** ไม่ใช่สรุปจากความรู้สึก

```
Voice Engineer: ทดสอบโทรภายในระหว่าง Extension 2001 ↔ 2002 (ไม่ออก Internet เลย)
ผลลัพธ์: ยังมีเสียงก้อง/แตกเหมือนเดิม แม้ Internet กลับมาใช้งานได้แล้ว 5 นาทีก่อนหน้า
```

**พิสูจน์แล้วว่าไม่เกี่ยวกัน** — นี่คือวินัยที่ Part นี้ต้องการปลูกฝัง: **การมี Incident เดียวไม่ได้
แปลว่ามี Root Cause เดียว** ต้องตรวจสอบทุกสมมติฐานด้วย Evidence จริงเสมอ ไม่ใช่ Correlation ทาง
เวลาอย่างเดียว (ตามที่เตือนไว้ใน Step 681.2)

### 684.2 ไล่ Diagnose แยกจากศูนย์

```
CallCenter-PC# (จำลอง IP Phone ที่ ACCESS-SW2)
ACCESS-SW2# show interfaces GigabitEthernet1/0/5 | include duplex|error
  Full-duplex, 1000Mb/s, media type is 10/100/1000BaseTX
  0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
  (ไม่มีปัญหา Physical Layer — ตัด Root Cause เชิง Hardware/Cable ออกไปได้)

ACCESS-SW2# show mls qos interface GigabitEthernet1/0/5
GigabitEthernet1/0/5
trust state: not trusted
trust mode: not trusted
COS override: dis
default COS: 0
```

**`trust state: not trusted`** — พอร์ตนี้ (พอร์ตที่ต่อ IP Phone) **ไม่มี Trust Boundary ใดๆ เลย**
ทั้งที่ตาม Baseline จาก [Part 24 Step 240](part-024-qos-fundamentals.md) ควรเป็น `trust device
cisco-phone` เสมอ เทียบกับ ACCESS-SW1 (ที่ไม่ได้ถูกแตะเลยในเหตุการณ์นี้):

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/5
GigabitEthernet1/0/5
trust state: trust cos
trust mode: trust device cisco-phone
```

### 684.3 ตรวจสอบ Global QoS

```
ACCESS-SW2# show mls qos
QoS is disabled globally
```

**QoS ไม่ได้ถูกเปิดระดับ Global เลยบน ACCESS-SW2** — ตามที่เตือนไว้ใน [Part 24 Step
235](part-024-qos-fundamentals.md) ว่านี่คือจุดพลาดที่พบบ่อยที่สุด (`mls qos trust ...` ที่
Interface จะไม่มีผลอะไรถ้า Global ปิดอยู่) — แต่ในกรณีนี้แม้แต่คำสั่งระดับ Interface ก็ไม่ปรากฏเลย
ด้วยซ้ำ ชี้ว่านี่คือ **Switch ที่ยังไม่ได้ผ่าน Standard QoS Template เลยทั้งตัว**

### 684.4 สอบถามที่มา — พบว่าไม่เกี่ยวกับ Change Friday เลย

```
ACCESS-SW2# show version | include uptime
ACCESS-SW2 uptime is 1 day, 14 hours, 22 minutes
```

Uptime แค่ 1 วันเศษ! ตรวจ Change Log แยกจาก War Room (Hardware Lifecycle Team คนละทีมกับ Security
Hardening ของ Root Cause 1):

> **Change Ticket CHG-88120 (ทีม Data Center/Hardware แยกจากทีม Security)**: เปลี่ยน ACCESS-SW2
> (Catalyst 2960 เดิม, End-of-Support) เป็น **Catalyst 9200 ตัวใหม่** ตามแผน Lifecycle Refresh
> ประจำปี — ดำเนินการ **เช้าวันเสาร์ 09:00-11:00** (คนละวัน คนละเวลา คนละทีมกับ Change ของ Root
> Cause 1 ที่ทำวันศุกร์ 16:00) วิศวกรที่ทำ Deploy ใช้ **Golden Config Template เวอร์ชันสำหรับ
> "Data-Only Closet"** (ไม่มี IP Phone) โดยพลาดไม่ได้เลือก Template "Access Closet with Voice VLAN"
> ที่มีบรรทัด `mls qos` (global) + `mls qos trust device cisco-phone` + `auto qos voip
> cisco-phone` รวมอยู่ด้วยตามที่ต้องใช้จริง

### 684.5 Fix — คืน Trust Boundary ตาม Baseline Part 24

```
ACCESS-SW2(config)# mls qos
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1 - 24
ACCESS-SW2(config-if-range)# mls qos trust device cisco-phone
ACCESS-SW2(config-if-range)# mls qos trust cos
ACCESS-SW2(config-if-range)# auto qos voip cisco-phone
ACCESS-SW2(config-if-range)# exit
```

### 684.6 Verify

```
ACCESS-SW2# show mls qos
QoS is enabled globally
QoS ip packet dscp rewrite is enabled

ACCESS-SW2# show mls qos interface GigabitEthernet1/0/5
GigabitEthernet1/0/5
trust state: trust cos
trust mode: trust device cisco-phone

ACCESS-SW2# show queueing interface GigabitEthernet1/0/5 | include Priority
    Priority Queue : Enabled
```

Voice Engineer ทดสอบโทรซ้ำระหว่าง Extension 2001↔2002 — เสียงชัดเจนปกติ ไม่มี Echo/Choppy อีก

**INC-40233 ปิด — Root Cause: ไม่มีความเกี่ยวข้องใดๆ กับ Root Cause 1 (พิสูจน์แล้วด้วย Evidence
ไม่ใช่การเดา)**

---

## Step 685 — Root Cause 3: DMVPN NHRP Dual-Hub ไม่สมบูรณ์ + สาย WAN Flap ที่สาขา

### 685.1 อาการ — ทำไม Dual-Hub Design "ควรจะ" กันปัญหานี้ได้อยู่แล้ว

Topology ของ DMVPN ที่ออกแบบไว้ตั้งแต่ [Part 53](part-053-dmvpn-advanced.md) เป็นแบบ **Phase 3,
Dual-Hub, Single-Cloud** — BRANCH2-RTR ควรมี Tunnel0 ใบเดียวที่ลงทะเบียนกับ **ทั้ง WAN-EDGE-1
(Primary, OSPF Cost 100) และ WAN-EDGE-2 (Backup, OSPF Cost 200)** พร้อมกัน ถ้า Path ไปยัง
WAN-EDGE-1 มีปัญหา OSPF ควร Fail Over ไปที่ Neighbor ผ่าน WAN-EDGE-2 ได้ภายในไม่กี่วินาทีโดย
ผู้ใช้ไม่ควรรู้สึกอะไรเลย — คำถามที่ Technical Lead A ต้องตอบคือ **"ทำไม Design ที่ควรกันปัญหานี้ได้
กลับไม่ทำงาน"**

### 685.2 ตรวจสอบ Physical Layer ก่อน (ต้นเหตุที่จุดชนวน)

```
BRANCH2-RTR# show interfaces GigabitEthernet0/1 | include line protocol|input errors
GigabitEthernet0/1 is up, line protocol is up (looks like ISP last-mile — see log below)

BRANCH2-RTR# show logging | include GigabitEthernet0/1
Sep 29 08:07:41: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
Sep 29 08:09:55: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to up
Sep 29 08:22:10: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
Sep 29 08:24:30: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to up
Sep 29 08:38:02: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
```

**สาย WAN Link ของ BRANCH2-RTR Flap เป็นระยะทุก 13-15 นาที** — เปิด Ticket กับผู้ให้บริการ ISP แล้ว
พบว่าเป็น **ปัญหา Last-Mile Fiber เสื่อมสภาพ** ที่ต้องรอทีมช่างของ ISP เข้าซ่อม (Physical Layer
เป็นสิ่งที่ทีมเราแก้เองไม่ได้ — สิ่งที่แก้ได้คือทำให้ผลกระทบจากการ Flap นี้**ไม่ลามไปเป็น Total
Outage** ผ่าน Design ซ้ำซ้อนที่มีอยู่แล้ว)

### 685.3 ตรวจ NHRP/OSPF — พบว่า Backup Hub ไม่เคยขึ้นเลย

```
BRANCH2-RTR# show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket

Interface Tunnel0 is up/up, Addr. is 172.16.100.12, VRF ""
Tunnel Src./Dest. addr: 198.51.100.6/MGRE, Tunnel VRF ""
Interface State Control: Enabled

Type:Spoke, Total NBMA Peers (v4/v6): 1
 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 203.0.113.2        172.16.100.1    UP  02:11:40     S

BRANCH2-RTR# show ip nhrp nhs detail
Legend: E=Expecting replies, R=Responding, W=Waiting
Tunnel0: Nhs is not registered with NHS 172.16.100.2 -- not configured
```

**BRANCH2-RTR ลงทะเบียนกับ WAN-EDGE-1 (172.16.100.1) เท่านั้น** — ไม่มี NHS ตัวที่สองไปยัง
WAN-EDGE-2 (172.16.100.2) เลย ตรวจ Running-config ยืนยัน:

```
BRANCH2-RTR# show run interface Tunnel0
interface Tunnel0
 ip address 172.16.100.12 255.255.255.0
 ip nhrp network-id 100
 ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
 ip nhrp shortcut
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROFILE
```

เทียบกับ Baseline ที่ถูกต้องจาก [Part 53 Step 525](part-053-dmvpn-advanced.md) ที่ต้องมี **สองบรรทัด
`ip nhrp nhs`**:

```
! Baseline ที่ถูกต้อง (ตาม Part 53 Step 525)
 ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast   ! WAN-EDGE-1 (Primary)
 ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast   ! WAN-EDGE-2 (Backup) <-- หายไป
```

```
BRANCH2-RTR# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.21          0   FULL/  -        00:00:35    172.16.100.1    Tunnel0
   (มี Neighbor เดียว — ควรมี 2 ตัวคือ 1.1.1.21 และ 1.1.1.22)
```

### 685.4 Root Cause ที่แท้จริง

`ip nhrp shortcut` (Phase 3) **มีอยู่จริง** และ Config ถูกต้องตามหลักการ แต่ **ไม่มีประโยชน์อะไร
เลยถ้าไม่มี NHS ตัวที่สองให้ Shortcut ไปหา** — คำสั่ง `ip nhrp shortcut` เปิดให้ Router "ยอมรับ
Redirect และสร้าง Path ที่สั้นกว่า" แต่ไม่ได้สร้าง Backup Hub Adjacency ขึ้นมาเอง หน้าที่นั้นเป็นของ
`ip nhrp nhs` ล้วนๆ — Gap ของ Phase 3 Dual-Hub นี้ทำให้ **Design ที่ควรกันปัญหา Flap ได้สมบูรณ์
กลับไม่มี Path สำรองให้ Fail Over เลยจริงๆ** ทุกครั้งที่สาย Flap ลง BRANCH2-RTR จึงขาดการเชื่อมต่อ
กับทั้งระบบโดยสมบูรณ์ (ไม่ใช่แค่ "ช้าลง") จนกว่าสายจะกลับมาและ NHRP Re-register กับ WAN-EDGE-1
ตัวเดิมอีกครั้ง — เกิดเป็น "หลุดเป็นช่วงๆ" ตรงตามที่ Ticket INC-40235 รายงาน

ตรวจ History พบว่าบรรทัดนี้หายไปตั้งแต่เหตุการณ์ **RMA เปลี่ยน Router ที่ BRANCH2-RTR ใน [Part 60
Step 594](part-060-dmvpn-vpn-troubleshooting.md)** — ตอนนั้นทีมแก้ปัญหา NHS ตัวหลักที่ชี้ผิด IP ไป
สำเร็จแล้ว แต่ **ไม่ได้เพิ่มบรรทัด NHS ตัวสำรองกลับเข้าไปให้ครบ** ตาม Design เดิม — เป็น Config
Drift ที่ค้างมานานหลายสัปดาห์โดยไม่มีใครสังเกต เพราะสายไม่เคย Flap มากพอที่จะเผยปัญหาออกมาจนวันนี้

### 685.5 Fix

```
BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast
BRANCH2-RTR(config-if)# exit
BRANCH2-RTR(config)# router ospf 100
BRANCH2-RTR(config-router)# neighbor 172.16.100.1 cost 100
BRANCH2-RTR(config-router)# neighbor 172.16.100.2 cost 200
BRANCH2-RTR(config-router)# exit
```

### 685.6 Verify

```
BRANCH2-RTR# show ip nhrp nhs detail
Tunnel0: 172.16.100.1  RE   NBMA Address: 203.0.113.2  priority = 0  cluster = 0
Tunnel0: 172.16.100.2  RE   NBMA Address: 203.0.113.6  priority = 0  cluster = 0

BRANCH2-RTR# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.21          0   FULL/  -        00:00:31    172.16.100.1    Tunnel0
1.1.1.22          0   FULL/  -        00:00:38    172.16.100.2    Tunnel0

BRANCH2-RTR# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 100", distance 110, metric 100, candidate default path
  Routing Descriptor Blocks:
  * 172.16.100.1, from 1.1.1.21, 00:00:31 ago, via Tunnel0     <- Active (Primary)
    172.16.100.2, from 1.1.1.22, 00:00:38 ago, via Tunnel0     <- Backup (พร้อมใช้ทันที)
```

### 685.7 ทดสอบ Failover จริงระหว่าง Flap ครั้งถัดไป

```
! รอบ Flap ครั้งถัดไปที่สายจริง Down (08:52) — สังเกตผล
BRANCH2-RTR# show logging | include OSPF|LINK-3
08:52:03: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
08:52:04: %OSPF-5-ADJCHG: Process 100, Nbr 1.1.1.21 on Tunnel0 from FULL to DOWN
08:52:05: %OSPF-5-ADJCHG: Process 100, Nbr 1.1.1.22 on Tunnel0 from FULL to FULL, Loading Done
            (Route ไป WAN-EDGE-2 ถูกติดตั้งเป็น Active ทันที — ไม่มี User รายงาน Ticket ซ้ำอีก)
```

**INC-40235 ปิด** — สาย WAN ยังต้องรอ ISP ซ่อมจริง แต่ Impact ต่อผู้ใช้กลายเป็นศูนย์แล้ว เพราะ Design
Redundancy ทำงานได้ตามที่ตั้งใจไว้ในที่สุด

---

## Step 686 — Root Cause 4: CoPP บล็อก RADIUS หลัง Security Hardening

### 686.1 ตรวจสอบชั้น Wireless ก่อน

```
WLC-1# show wireless client summary
Number of Local Clients: 41
              MAC Address    AP Name       WLAN State
              a4b1.c2.xxxx    AP-2         corp-wifi  Authenticating
              a4b1.c2.yyyy    AP-2         corp-wifi  Authenticating
              ... (Client ค้างที่ Authenticating จำนวนมาก เฉพาะที่ AP-2 อาคาร C)

WLC-1# show ap join stats detailed AP-2 | include Radio|Status
              (AP-2 Join CAPWAP สมบูรณ์ปกติ — ไม่ใช่ปัญหา CAPWAP/AP)
```

Client ได้ IP Address จาก DHCP ปกติ (เห็น DHCP ACK) — แต่ค้างที่ขั้น **802.1X/WPA2-Enterprise
Authentication** ซึ่งต้องพึ่ง RADIUS ตรวจสอบ Username/Password กับ ISE

### 686.2 ตรวจสอบ RADIUS

```
WLC-1# show aaa servers | section 10.10.99.61
RADIUS: id 1, priority 1, host 10.10.99.61, auth-port 1812, acct-port 1813
     State: current UP, duration 812000s, previous duration 0s
     Dead: total time 0s, count 0
     Requests: 18442, Timeouts: 3120, Failovers: 0
     (State = UP แต่ Timeout เพิ่มขึ้นเร็วมากตั้งแต่ 08:10 เป็นต้นมา — ต่างจาก Step 604 ที่
      State = DEAD ไปเลย — นี่เป็น Signature คนละแบบกัน)
```

`State: UP` แปลว่า **ISE-1 ไม่ได้ล่ม** (ต่างจากสถานการณ์ใน [Part 61 Step
604](part-061-infra-security-troubleshooting.md)) แต่จำนวน `Timeouts` พุ่งขึ้นเร็วผิดปกติ —
สัญญาณนี้ชี้ไปที่ **Path ระหว่าง WLC-1 กับ ISE-1 บางอย่างกำลัง Drop Packet เป็นบางครั้ง** ไม่ใช่
Drop ทั้งหมด (ถ้า Drop ทั้งหมด State จะกลาย DEAD ไปแล้ว)

```
WLC-1# ping 10.10.99.61
Success rate is 60 percent (3/5)
   (Ping ผ่านบางครั้ง — ไม่ใช่ Routing/Reachability พังสนิท เป็น Pattern ของการถูก Rate-Limit)
```

### 686.3 ตรวจสอบ CoPP บน WLC-1

WLC-1 เป็น Catalyst 9800-CL ซึ่งรัน **IOS-XE เดียวกันกับ Switch/Router ตัวอื่นในหลักสูตร** จึงมี
`control-plane` Submode และรองรับ CoPP ด้วยกลไกเดียวกันกับที่สอนใน [Part
46](part-046-control-plane-policing.md):

```
WLC-1# show policy-map control-plane
 Service-policy input: COPP-POLICY

  Class-map: COPP-CRITICAL (match-any)
    0 packets
  Class-map: COPP-IMPORTANT (match-any)
    142012 packets
  Class-map: class-default (match-any)
    891402 packets
    police:
        cir 8000 bps, bc 2000 bytes
      conformed 612100 packets; actions: transmit
      exceeded 279302 packets; actions: drop      <-- exceeded เพิ่มขึ้นเรื่อยๆ
```

`class-default` มี `exceeded` เพิ่มขึ้นต่อเนื่อง — ตรวจว่า RADIUS Traffic ตกไปอยู่ Class ไหน:

```
WLC-1# show access-list COPP-ACL-MGMT
Extended IP access list COPP-ACL-MGMT
    10 permit tcp 10.10.99.0 0.0.0.255 any eq 22
    20 permit tcp 10.10.99.0 0.0.0.255 any eq 49
    30 permit udp 10.10.99.0 0.0.0.255 any eq 161
    40 permit udp any eq 162 10.10.99.0 0.0.0.255
    50 permit udp 10.10.99.0 0.0.0.255 any eq 123
    (ไม่มีบรรทัดใดที่ Match UDP 1812/1813 เลย — RADIUS ไม่ถูก Classify เข้า COPP-IMPORTANT)
```

**RADIUS (UDP 1812/1813) ไม่ได้อยู่ใน ACL `COPP-ACL-MGMT` เลย** — Traffic ตอบกลับจาก ISE-1
(Access-Accept/Reject ที่ส่งกลับมายัง WLC-1) จึงตกไปที่ **`class-default` โดยอัตโนมัติ** ซึ่งถูกตั้ง
ไว้ที่ **`cir 8000 bps`** — ต่ำมากเมื่อเทียบกับ Baseline เดิมของ Part 46 (`cir 32000 bps`)

### 686.4 ตรวจ Change History บน WLC-1

```
WLC-1# show archive config differences
  ! เทียบกับ Checkpoint ก่อน Maintenance Window วันศุกร์
  - policy-map COPP-POLICY
  -  class class-default
  -   police cir 32000 bc 8000 conform-action transmit exceed-action drop
  + policy-map COPP-POLICY
  +  class class-default
  +   police cir 8000 bc 2000 conform-action transmit exceed-action drop
```

### 686.5 Root Cause — Template CoPP ที่ "Portable" กลับกลายเป็นปัญหาข้ามชนิดอุปกรณ์

[Part 46](part-046-control-plane-policing.md) เคยชี้ให้เห็นข้อดีของการใช้ชื่อ ACL/Class-map/
Policy-map เดียวกันทุกอุปกรณ์ว่า **"Config Portable ข้ามอุปกรณ์ที่มี Role เดียวกันได้ทันที"** — ทีม
Security ใช้หลักการนี้ในการ Rollout **"Q3 Security Hardening"** เดียวกันกับ Root Cause 1 (วันศุกร์
16:00) โดยขยาย CoPP Template จาก Switch/Router **ไปถึงทุกอุปกรณ์ที่รัน IOS-XE รวม WLC-1 ด้วย** และ
พร้อมกันนั้นก็ Tighten `class-default` จาก 32 kbps เป็น **8 kbps** เพื่อลด Attack Surface โดยรวม —
ปัญหาคือ **Template ต้นฉบับถูกออกแบบมาสำหรับ Switch/Router ที่ทำหน้าที่ Client ของ TACACS+/SNMP/NTP
เท่านั้น ไม่เคยมีอุปกรณ์ใดต้องเป็น RADIUS Client เต็มรูปแบบแบบ WLC มาก่อน** — ผลคือ RADIUS
ซึ่งเป็น Traffic ที่ WLC-1 **ต้องพึ่งพาเป็นหัวใจหลัก** กลับไม่ถูกจัดอยู่ใน Class ที่มี CIR สูงพอเลย

ในสภาวะปกติ (Client ทยอยเชื่อมต่อทีละคน) Volume ของ RADIUS ต่ำพอที่ 8 kbps ยังพอไหว — แต่**เช้า
วันจันทร์ที่อาคาร C มีประชุม All-Hands ตอน 08:10 พนักงานหลายสิบคนเชื่อมต่อ Wi-Fi พร้อมกันในเวลา
ไล่เลี่ยกัน** ทำให้ Burst ของ RADIUS Request/Response ในช่วงนาทีนั้นสูงเกิน 8 kbps อย่างมาก จึงถูก
`police` Drop เป็นบางส่วน — อธิบายได้ทั้ง **"ทำไมเพิ่งเกิดตอนนี้"** (Burst สูงพอที่จะชนเพดานใหม่
เป็นครั้งแรก) และ **"ทำไมเกิดที่อาคาร C เท่านั้น"** (อาคารอื่นไม่มี Burst ระดับเดียวกันในเวลาเดียวกัน
— แม้ CoPP Policy จะถูก Apply เหมือนกันทั่วทั้งองค์กรก็ตาม)

### 686.6 Fix

```
WLC-1(config)# ip access-list extended COPP-ACL-MGMT
WLC-1(config-ext-nacl)# permit udp host 10.10.99.61 eq 1812 any
WLC-1(config-ext-nacl)# permit udp host 10.10.99.61 eq 1813 any
WLC-1(config-ext-nacl)# exit
WLC-1(config)# policy-map COPP-POLICY
WLC-1(config-pmap)# class COPP-IMPORTANT
WLC-1(config-pmap-c)#  police cir 4000000 bc 512000 conform-action transmit exceed-action drop
WLC-1(config-pmap-c)# exit
WLC-1(config-pmap)# class class-default
WLC-1(config-pmap-c)#  police cir 32000 bc 8000 conform-action transmit exceed-action drop
WLC-1(config-pmap-c)# end
```

> **หมายเหตุ**: `COPP-ACL-MGMT` ต้อง Match เข้า Class `COPP-IMPORTANT` (ผ่าน `class-map match-any
> COPP-IMPORTANT` ที่ผูกกับ ACL นี้อยู่แล้วจาก Part 46) — เพิ่มบรรทัด ACL อย่างเดียวก็เพียงพอให้
> RADIUS ย้าย Class โดยไม่ต้องแก้ Class-map เพิ่ม ส่วนการปรับ `class-default` กลับไป 32 kbps คือการ
> **คืนค่าเดิมที่ปลอดภัย** ไม่ใช่การปิด CoPP ทิ้ง (ตามหลักการ Part 46: ห้ามลด Security ของ Class อื่น
> เพื่อแก้ปัญหาของ Class เดียว)

### 686.7 Verify

```
WLC-1# show access-list COPP-ACL-MGMT | include 1812|1813
    60 permit udp host 10.10.99.61 eq 1812 any
    70 permit udp host 10.10.99.61 eq 1813 any

WLC-1# show policy-map control-plane class COPP-IMPORTANT
    conformed 4102 packets; actions: transmit
    exceeded 0 packets; actions: drop

WLC-1# show aaa servers | section 10.10.99.61
     Requests: 18690, Timeouts: 0

WLC-1# show wireless client summary
              MAC Address    AP Name       WLAN State
              a4b1.c2.xxxx    AP-2         corp-wifi  Run
              a4b1.c2.yyyy    AP-2         corp-wifi  Run
```

**INC-40238 ปิด** — Client ทุกตัวย้ายจาก `Authenticating` เป็น `Run` ภายใน 90 วินาทีหลัง Apply Fix

---

## Step 687 — Cross-Cutting Analysis: เส้นด้ายร่วมของ 4 ปัญหา

### 687.1 Timeline รวมของการเปลี่ยนแปลงที่นำไปสู่เหตุการณ์นี้

| เวลา | เหตุการณ์ | ทีมที่ทำ | เกี่ยวข้องกับ Root Cause ใด |
|---|---|---|---|
| ศุกร์ 16:00 | Push "Q3 Security Hardening": BGP Inbound Prefix-list ใหม่บน WAN-EDGE-1/2 | ทีม Security/Network | **Root Cause 1** |
| ศุกร์ 16:20 | Push CoPP Template ใหม่ (`class-default` 32k→8k) ไปยังทุกอุปกรณ์ IOS-XE รวม WLC-1 | ทีม Security/Network (Change Ticket **เดียวกัน** กับข้อบน) | **Root Cause 4** |
| เสาร์ 09:00-11:00 | เปลี่ยน ACCESS-SW2 เป็น Catalyst 9200 (Hardware Lifecycle) | ทีม Data Center/Hardware (คนละทีม คนละ Ticket) | **Root Cause 2** |
| (สัปดาห์ก่อนหน้า) | RMA เปลี่ยน Router ที่ BRANCH2-RTR — ลืมเพิ่ม NHS สำรอง | ทีม Field Ops (เหตุการณ์เก่า ไม่เกี่ยวกับวันศุกร์เลย) | **Root Cause 3 (ต้นตอ)** |
| อาทิตย์กลางคืน | สาย Fiber Last-Mile ของ BRANCH2-RTR เริ่มเสื่อมสภาพ (ตรวจพบจาก ISP) | ผู้ให้บริการ ISP ภายนอก | **Root Cause 3 (ตัวจุดชนวน)** |
| จันทร์ 08:00+ | พนักงานเข้างาน + ประชุม All-Hands อาคาร C → Burst RADIUS | ปกติทางธุรกิจ | **Root Cause 4 (ตัวจุดชนวน)** |

### 687.2 สิ่งที่ "เชื่อมกันจริง" — Root Cause 1 และ 4

Root Cause 1 (BGP Prefix-list) และ Root Cause 4 (CoPP บน WLC-1) **มาจาก Change Ticket เดียวกัน**
ในหน้าต่าง Maintenance เดียวกัน (ศุกร์ 16:00-16:30) ที่ทีม Security ผลักดัน "Q3 Security Hardening"
— นี่คือเส้นด้ายร่วมที่แท้จริงเพียงเส้นเดียวในเหตุการณ์นี้ และมีสาเหตุเชิงกระบวนการร่วมกันชัดเจน:

| จุดบกพร่องของกระบวนการ | ผลลัพธ์ |
|---|---|
| **ไม่มี Peer Review ของ Prefix-list/ACL ก่อน Push จริง** | Syntax ผิด (`ge 1`) และ ACL ที่ไม่ครบ (ไม่มี RADIUS) หลุดเข้า Production ตรงๆ |
| **Bundle การเปลี่ยนแปลงหลายจุดไว้ใน Change Window เดียว** (WAN Edge 2 ตัว + ทุกอุปกรณ์ IOS-XE) | Blast Radius กว้างเกินจำเป็น — ถ้าอุปกรณ์ใดมีปัญหา ผลกระทบกระจายไปหลายจุดพร้อมกัน |
| **ไม่มี Staged/Canary Rollout** (Deploy ทุกอุปกรณ์พร้อมกันในคราวเดียว) | ถ้าทดสอบกับอุปกรณ์ 1 ตัวก่อน (เช่น WAN-EDGE-2 หรือ WLC-1 test lab) จะเจอปัญหาตั้งแต่ศุกร์บ่าย ไม่ใช่จันทร์เช้าที่ User สัมผัสจริง |
| **ไม่มีการทดสอบ Traffic จริงหลัง Deploy ก่อนปิด Change Window** | ทั้งสอง Bug ไม่มีอาการทันที (Default Route ที่หายไปตอนดึกไม่มีคนใช้ Internet, RADIUS Burst ไม่เกิดตอนศุกร์เย็น) — Silent Failure ที่รอเวลาแสดงอาการวันทำการถัดไป |

### 687.3 สิ่งที่ "ไม่เกี่ยวกัน" — Root Cause 2 และ 3

Root Cause 2 (QoS, Hardware Refresh วันเสาร์) และ Root Cause 3 (DMVPN, RMA เก่า + สาย ISP เสื่อม)
**ไม่มีความเกี่ยวข้องใดๆ กับ Change Ticket ของวันศุกร์เลย** — เป็นความบังเอิญทางเวลาล้วนๆ ที่ทำให้
ดูเหมือนเป็นเรื่องเดียวกันตอนที่ Ticket มาถึงพร้อมกันในเช้าวันจันทร์

### 687.4 บทเรียนที่ต้องจำ

> **"เหตุการณ์เดียวไม่ได้แปลว่ามี Root Cause เดียว และ Root Cause หลายอันไม่ได้แปลว่าทุกอันต้อง
> เชื่อมกันเสมอ"** — วินัยของวิศวกรที่ดีคือการ **พิสูจน์ความเชื่อมโยงด้วย Evidence จริงทุกครั้ง**
> (Change Log, Timestamp, Config Diff) ไม่ใช่การเดาจาก Correlation ทางเวลา ทั้งสองทิศทาง: **ห้าม
> รวบเป็นสาเหตุเดียวทั้งที่ไม่ใช่ (Over-correlation)** และ **ห้ามมองข้ามความเชื่อมโยงที่มีจริงเพราะ
> ดูเหมือนคนละเรื่อง (Under-correlation)** — Root Cause 1/4 คือตัวอย่างของอันตรายจาก Change
> Management ที่ขาดวินัย ส่วน Root Cause 2/3 คือตัวอย่างของความบังเอิญที่ต้องแยกออกให้ถูกด้วย
> การทดสอบจริง (Step 684.1) ไม่ใช่การสันนิษฐาน

---

## Step 688 — แผน Remediation เต็มรูปแบบ

### 688.1 ลำดับการแก้ไขตาม Dependency (ไม่ใช่ตามลำดับที่ค้นพบ)

| ลำดับ | Fix | เหตุผลของลำดับนี้ | ทีมรับผิดชอบ |
|---|---|---|---|
| 1 | **Root Cause 1** — แก้ Prefix-list `ISP-IN-PFX` บน WAN-EDGE-1 | ต้องทำก่อนสุด เพราะ Internet เป็น Dependency ของการ Verify งานอื่น (เช่น Syslog/NTP ไปยัง Server ภายนอก, Software Update Check) และกระทบ Business Impact สูงสุด | Routing/WAN |
| 2 | **Root Cause 4** — เพิ่ม RADIUS เข้า `COPP-ACL-MGMT` + คืน `class-default` เป็น 32k บน WLC-1 | ทำคู่กันกับข้อ 1 ได้ทันที (คนละอุปกรณ์ ไม่ Dependency กัน) แต่ควรเสร็จก่อนช่วง Peak Wi-Fi ตอนสาย | Security/Wireless |
| 3 | **Root Cause 3** — เพิ่ม NHS สำรองบน BRANCH2-RTR + เปิด Ticket ผู้ให้บริการ ISP เรื่องสาย | ทำได้อิสระจากข้ออื่น — ส่วนสาย Fiber จริงต้องรอ ISP ซึ่งอยู่นอกเหนือการควบคุมของทีมเรา | Routing/WAN |
| 4 | **Root Cause 2** — คืน QoS Trust Boundary บน ACCESS-SW2 | ทำได้อิสระจากข้ออื่นเช่นกัน — จัดคิวหลังสุดเพราะ Business Impact ต่ำกว่า (ไม่กระทบ Internet/Auth) แม้กระทบผู้ใช้ Call Center ที่หงุดหงิดมากก็ตาม | Voice/QoS |

> **ข้อสังเกต**: แม้ Fix ทั้ง 4 ไม่มี Technical Dependency ระหว่างกันเลย (คนละอุปกรณ์ คนละ Feature)
> การจัดลำดับยังต้องอิงตาม **Business Impact และการยืนยันผลลัพธ์ที่ชัดเจนที่สุดก่อน** — แก้ Internet
> ก่อนเสมอเพราะเป็น Baseline ที่ทุกการ Verify อื่นในขั้นตอนถัดไปต้องพึ่งพา (เช่น NTP Sync ไป Server
> ภายนอก, Download Patch)

### 688.2 Rollback Plan ต่อแต่ละ Fix (เตรียมไว้ก่อนเริ่มแก้จริงเสมอ)

| Fix | คำสั่ง Rollback ทันที | เงื่อนไขที่ต้อง Trigger Rollback |
|---|---|---|
| Root Cause 1 (Prefix-list) | `no ip prefix-list ISP-IN-PFX seq 5`<br>`no ip prefix-list ISP-IN-PFX seq 10`<br>กลับไปใช้ Prefix-list เดิมจาก `show archive config differences` | ถ้า BGP Session หลุดหลัง Soft Reset หรือ Route อื่นที่เคย Filter ถูกต้องกลับเข้ามาเกินขอบเขตที่ตั้งใจ (Bogon หลุดเข้ามา) |
| Root Cause 4 (CoPP) | `no permit udp host 10.10.99.61 eq 1812 any` (คืน ACL เดิม)<br>`police cir 8000 bc 2000 ...` (คืน CIR เดิมถ้าจำเป็นจริงๆ) | ถ้าการเพิ่ม CIR ทำให้ Class อื่นถูกแย่ง Bandwidth บน CPU จนกระทบ Routing Protocol (ตรวจผ่าน `show policy-map control-plane class COPP-CRITICAL`) |
| Root Cause 3 (NHRP) | `no ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast`<br>`no neighbor 172.16.100.2 cost 200` | ถ้า WAN-EDGE-2 มีปัญหาอื่นซ่อนอยู่ (ยังไม่เคย Active มา Production จริงมานาน) ที่ทำให้ Failover ไป Backup แล้วแย่กว่าเดิม |
| Root Cause 2 (QoS) | `no auto qos voip cisco-phone`<br>`no mls qos trust device cisco-phone`<br>(กลับไปสถานะ Untrusted ชั่วคราว) | ถ้า `auto qos voip cisco-phone` สร้าง Queue Config ที่ชนกับ QoS Policy อื่นบน ACCESS-SW2 ตัวใหม่ (ต่าง Platform จาก 2960 เดิม อาจมี Default Queue ต่างกัน) |

### 688.3 หลักการ Rollback ร่วม

1. **ทำ Fix ทีละตัว เว้นระยะอย่างน้อย 5 นาทีก่อนทำตัวถัดไป** — เพื่อแยกให้เห็นชัดว่าอาการที่เปลี่ยนไป
   มาจาก Fix ตัวใด (สำคัญเมื่อมี 4 การเปลี่ยนแปลงพร้อมกันในระบบเดียว)
2. **เก็บ `show archive config differences` ก่อนและหลัง Fix ทุกตัว** — เพื่อให้ Rollback ทำได้แม่นยำ
   และเก็บเป็นหลักฐานสำหรับ PIR (Step 690)
3. **แจ้ง War Room ทุกครั้งก่อนและหลัง Apply Fix จริงตามกฎ Step 682.3**

---

## Step 689 — Post-Incident Verification: Health-Check ครบทุกโดเมน

หลัง Fix ครบทั้ง 4 จุด ต้องยืนยันว่า **(1) ทั้ง 4 ปัญหาหายจริง และ (2) ไม่มีปัญหาใหม่เกิดขึ้นในโดเมน
อื่นที่ไม่เกี่ยวข้อง** จากการที่ War Room แก้ไข Config พร้อมกันหลายจุดในเวลาไล่เลี่ยกัน — Sweep ผ่าน
ทุกโดเมนที่เรียนมาตลอด Part 56-68:

| โดเมน (Part อ้างอิง) | คำสั่ง Verify หลัก | ผลลัพธ์ที่คาดหวัง |
|---|---|---|
| EIGRP ([Part 56](part-056-eigrp-troubleshooting.md)) | `show ip eigrp neighbors`<br>`show ip route eigrp` | EIGRP-R1/EIGRP-R2 (AS 100, Legacy Site) ยัง Neighbor ปกติ ไม่กระทบจากการแก้ไขใดๆ |
| OSPF Campus ([Part 57](part-057-ospf-troubleshooting.md)) | `show ip ospf neighbor`<br>`show ip ospf database external` | Area 0/1/2(NSSA) เห็น Default Route ครบ, ไม่มี Area Mismatch |
| BGP ([Part 58](part-058-bgp-troubleshooting.md)) | `show ip bgp summary`<br>`show ip bgp neighbors 203.0.113.1 advertised-routes` | Session Established ทั้งคู่ (ISP + iBGP Cross-link), `PfxRcd` รวม Default แล้ว |
| Redistribution ([Part 59](part-059-redistribution-troubleshooting.md)) | `show ip route ospf 100`<br>`show route-map` | Redistribute ระหว่าง OSPF 100 (Overlay) ↔ OSPF 1 (Campus) ไม่มี Route แปลกปลอมเพิ่มจากการแก้ Prefix-list |
| DMVPN/VPN ([Part 60](part-060-dmvpn-vpn-troubleshooting.md)) | `show dmvpn`<br>`show ip nhrp nhs detail`<br>`show crypto isakmp sa` | BRANCH-RTR และ BRANCH2-RTR ทั้งคู่มี NHS 2 ตัว, IPsec SA `QM_IDLE` ปกติ |
| Infra Security/CoPP ([Part 61](part-061-infra-security-troubleshooting.md)) | `show policy-map control-plane`<br>`show authentication sessions` | `exceeded` ของทุก Class คงที่ (ไม่เพิ่มขึ้นต่อเนื่อง), 802.1X Session อื่นยัง Authorized ปกติ |
| Infra Services ([Part 62](part-062-infra-services-troubleshooting.md)) | `show ntp status`<br>`show logging \| include Syslog server` | ทุกอุปกรณ์ Sync กับ `10.10.99.50` ปกติ (Stratum ถูกต้อง), Syslog ส่งเข้า Server ต่อเนื่องไม่มี Gap |
| NAT ([Part 63](part-063-advanced-nat-scenarios.md)) | `show ip nat translations`<br>`show ip nat statistics` | PAT Overload บน WAN-EDGE-1 (`GigabitEthernet0/0/0`) ทำงานปกติ หลังกลับมามี Default Route |
| Path Control/PBR ([Part 64](part-064-path-control-pbr-troubleshooting.md)) | `show route-map`<br>`show ip policy` | Policy-Based Routing (ถ้ามี) ไม่ถูกกระทบจากการเปลี่ยน Prefix-list/OSPF Cost |
| Multicast ([Part 65](part-065-multicast-troubleshooting.md)) | `show ip mroute`<br>`show ip pim neighbor` | PIM Neighbor และ Multicast Tree ไม่กระทบจากการแก้ CoPP (ตรวจ Class ที่ Multicast Control Traffic ใช้ด้วย) |
| Wireless ([Part 66](part-066-wireless-troubleshooting.md)) | `show wireless client summary`<br>`show wireless stats client-delete reason` | Client ทุกตัวที่ AP-1/AP-2 อยู่สถานะ `Run`, ไม่มี Delete Reason จาก RADIUS Timeout อีก |
| Automation ([Part 67](part-067-automation-troubleshooting.md)) | `show netconf-yang status`<br>ทดสอบ RESTCONF GET ไปยัง WAN-EDGE-1 | API/Automation Pipeline ที่อาจใช้ Monitor Config (เช่น Ansible Playbook ตรวจ Compliance) ทำงานได้ปกติ ไม่ถูก Block โดย CoPP Class-default ที่ยังเข้มงวดอยู่ |
| QoS ([Part 68](part-068-qos-troubleshooting.md)) | `show mls qos interface`<br>`show policy-map interface` | ACCESS-SW2 กลับสู่ Trust Boundary มาตรฐาน, Switch อื่นทั้งหมดไม่กระทบ |

### 689.1 ตัวอย่างการรัน Sweep จริงแบบย่อ (บางส่วน)

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           1   FULL/BDR        00:00:31    10.255.0.2      Port-channel1
1.1.1.11          1   FULL/DR         00:00:38    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.21          0   FULL/  -        00:00:34    10.10.254.1     GigabitEthernet1/0/1

WAN-EDGE-2# show ip bgp summary | include 65001|65000
203.0.113.5     4 65000    5210    5198     8842    0    0 3d18h          3
1.1.1.21        4 65001    9821    9820     8842    0    0 3d18h          412

WLC-1# show policy-map control-plane class COPP-CRITICAL
    conformed 22 packets; actions: transmit
    exceeded 0 packets; actions: drop

ACCESS-SW2# show run | include auto qos|mls qos trust
 mls qos trust device cisco-phone
 mls qos trust cos
```

### 689.2 ผลสรุป Health-Check

| Ticket | สถานะ | ยืนยันโดย |
|---|---|---|
| INC-40231 (Internet) | ✅ Closed | Default Route + Ping 8.8.8.8 100% |
| INC-40233 (Voice) | ✅ Closed | Trust Boundary + ทดสอบสายจริง |
| INC-40235 (Branch) | ✅ Closed (Config) / ⏳ Open (สายจริงรอ ISP) | NHRP Dual-Hub + Failover Test สำเร็จ |
| INC-40238 (Wi-Fi) | ✅ Closed | RADIUS Timeout = 0, Client State = Run |
| ALERT-9042 (Syslog) | ✅ Closed (เป็นผลจาก INC-40231) | LSA Flooding หยุดหลัง Default Route กลับมา |

**ไม่พบปัญหาใหม่ในโดเมนอื่นใดจาก Sweep เต็มรูปแบบ** — War Room ประกาศปิด Incident ที่เวลา 10:47
(รวมเวลาทั้งหมดตั้งแต่ Ticket แรกถึงปิด Incident: **2 ชั่วโมง 45 นาที**)

---

## Step 690 — Post-Incident Review (PIR)

> เอกสารนี้คือสิ่งที่ Senior Network Engineer ต้องเขียนส่งภายใน 48 ชั่วโมงหลังปิด Sev-1 Incident
> ตามมาตรฐาน ITIL/NOC ทั่วไป — ใช้ Format เดียวกันนี้ได้กับทุก Major Incident ในงานจริง

### PIR-2025-0929 — Multi-Domain Outage (Internet / Voice / DMVPN Branch / Wireless Auth)

**สถานะ**: Closed | **Severity**: Sev-1 | **ระยะเวลารวม**: 08:02–10:47 (2 ชม. 45 นาที)

#### 1. Executive Summary

เช้าวันจันทร์เกิดเหตุการณ์ขัดข้องพร้อมกัน 4 จุดในระบบเครือข่าย ได้แก่ (1) ผู้ใช้ทั้งองค์กรเข้า
Internet ไม่ได้ (2) คุณภาพเสียงของระบบ Call Center แย่ลง (3) สาขาย่อยหลุดการเชื่อมต่อเป็นช่วงๆ และ
(4) พนักงานในอาคาร C ไม่สามารถยืนยันตัวตนเข้า Wi-Fi ได้ ทีม NOC ตรวจพบว่าเหตุการณ์นี้มี **Root Cause
4 จุดที่แยกจากกันทางเทคนิค** โดย 2 จุด (Internet, Wi-Fi) มีที่มาจาก Change Window เดียวกันในวันศุกร์
ที่ขาดกระบวนการ Peer Review และ Staged Rollout ที่รัดกุมพอ ส่วนอีก 2 จุด (Voice, Branch) เป็นเหตุ
บังเอิญทางเวลาที่ไม่เกี่ยวข้องกับ Change ดังกล่าว ทุกจุดได้รับการแก้ไขและยืนยันผลสมบูรณ์แล้ว ยกเว้น
ปัญหาสาย Fiber ทางกายภาพของสาขาย่อยที่ยังต้องรอผู้ให้บริการ ISP ดำเนินการซ่อมแซม

#### 2. Timeline โดยละเอียด

| เวลา | เหตุการณ์ |
|---|---|
| ศุกร์ 16:00 | Push Change: BGP Inbound Prefix-list ใหม่ (WAN-EDGE-1/2) + CoPP Template ใหม่ (ทุกอุปกรณ์ IOS-XE) |
| เสาร์ 09:00-11:00 | เปลี่ยน ACCESS-SW2 เป็น Catalyst 9200 (Hardware Lifecycle — ไม่เกี่ยวข้องกับข้างบน) |
| อาทิตย์กลางคืน | สาย Fiber Last-Mile ของ BRANCH2-RTR เริ่ม Flap (ตรวจพบย้อนหลัง) |
| จันทร์ 08:02-08:14 | Ticket 5 ใบเข้าระบบพร้อมกัน (INC-40231, 40233, 40235, 40238, ALERT-9042) |
| จันทร์ 08:15 | ประกาศ Sev-1, เปิด War Room |
| จันทร์ 08:24 | Root Cause 1 พบ (Prefix-list `ge 1`) |
| จันทร์ 08:24 | Root Cause 1 แก้ไข, Internet กลับมา, ALERT-9042 ปิด |
| จันทร์ 08:31 | Root Cause 2 พิสูจน์ว่าไม่เกี่ยวกับ Root Cause 1 (ทดสอบสายภายใน) |
| จันทร์ 08:45 | Root Cause 2 พบและแก้ไข (QoS Trust Boundary), Voice กลับมาปกติ |
| จันทร์ 09:05 | Root Cause 3 พบ (NHS ขาดหาย) |
| จันทร์ 09:15 | Root Cause 3 แก้ไข, ยืนยัน Failover สำเร็จรอบถัดไป (09:52) |
| จันทร์ 09:40 | Root Cause 4 พบ (CoPP class-default 8kbps บล็อก RADIUS) |
| จันทร์ 09:50 | Root Cause 4 แก้ไข, Wi-Fi Authentication กลับมาปกติ |
| จันทร์ 10:00-10:45 | Full Health-Check Sweep ทุกโดเมน (Step 689) |
| จันทร์ 10:47 | ปิด Incident |

#### 3. Impact

| รายการ | รายละเอียด |
|---|---|
| ผู้ใช้ที่กระทบ | ทั้งองค์กร (Internet), Call Center ทั้งทีม (Voice), พนักงานอาคาร C ~120 คน (Wi-Fi), สาขาย่อย 1 แห่ง (Branch) |
| ผลกระทบเชิงธุรกิจ | CRM/SaaS ใช้งานไม่ได้ 22 นาที, คุณภาพสาย Call Center ต่ำ 43 นาที, ERP สาขาหลุดเป็นช่วงๆ ตลอดเช้า, พนักงานอาคาร C เข้า Wi-Fi ไม่ได้ 99 นาที |
| SLA ที่กระทบ | Internet Availability SLA (99.9%), Voice Quality SLA (MOS Score) |

#### 4. Root Causes (สรุป)

| # | Root Cause | โดเมน | ประเภท |
|---|---|---|---|
| 1 | `ip prefix-list ISP-IN-PFX seq 5 permit 0.0.0.0/0 ge 1` ตัด Default Route ออกจากการ Match โดยไม่ตั้งใจ | BGP + Redistribution | Change-Induced |
| 2 | ACCESS-SW2 ตัวใหม่ (Hardware Refresh) ไม่ได้ Apply QoS Trust Boundary Template ที่ถูกต้อง | QoS | Process Gap (Template Selection) |
| 3 | BRANCH2-RTR ขาด NHS สำรองไปยัง WAN-EDGE-2 มาตั้งแต่เหตุการณ์ RMA ก่อนหน้า + สาย Fiber Flap จาก ISP | DMVPN/NHRP | Latent Config Drift + External |
| 4 | CoPP `class-default` บน WLC-1 ถูก Tighten เป็น 8 kbps โดยไม่ได้เพิ่ม RADIUS เข้า ACL ที่มี CIR สูงกว่า | Infra Security/CoPP + Wireless | Change-Induced |

#### 5. สิ่งที่ทำได้ดี (What Went Well)

- Incident Commander ประกาศ Sev-1 และเปิด War Room ภายใน 13 นาทีหลัง Ticket แรกเข้า — เร็วกว่า
  Target ภายในของทีม (15 นาที)
- กฎ "ห้ามสมมติ Root Cause เดียว" (Step 682.3, 684.1) ถูกใช้จริงและป้องกันการเสียเวลาไล่ผิดทาง —
  Technical Lead B ทดสอบสมมติฐานด้วย Evidence จริงก่อนสรุป
- การใช้ `clear ip bgp ... soft in` แทน `clear ip bgp *` และการ Apply Fix ทีละตัวเว้นระยะ ทำให้ไม่มี
  Side Effect จากการแก้ไขซ้อนกัน
- Full Health-Check Sweep ครบทุกโดเมนก่อนปิด Incident ทำให้มั่นใจว่าไม่มีปัญหาใหม่หลุดออกไป

#### 6. สิ่งที่ต้องปรับปรุง (What To Improve)

| ปัญหาเชิงกระบวนการ | ผลกระทบ | ระดับความสำคัญ |
|---|---|---|
| ไม่มี Peer Review บังคับสำหรับการเปลี่ยน Prefix-list/ACL/CoPP Policy บนอุปกรณ์ Production | Bug Syntax หลุดเข้า Production ตรงๆ | สูง |
| Bundle การเปลี่ยนแปลงหลายอุปกรณ์/หลาย Feature ไว้ใน Change Window เดียวโดยไม่มี Staged Rollout | Blast Radius กว้างเกินจำเป็นเมื่อมีปัญหา | สูง |
| CoPP Template ที่ Copy จาก Switch/Router ถูกใช้กับ WLC โดยไม่มีการทบทวนว่า Protocol ที่อุปกรณ์ประเภทนี้ต้องใช้ต่างกัน | RADIUS ไม่ถูกจัดอยู่ใน Class ที่เหมาะสมตั้งแต่ต้น | สูง |
| ไม่มี Post-RMA Configuration Audit Checklist ที่ตรวจ Redundancy Feature ครบ (เช่น NHS สำรอง) | Config Drift ค้างอยู่หลายสัปดาห์โดยไม่มีใครรู้ | กลาง |
| ไม่มีการทดสอบ Traffic จริงหลัง Deploy ก่อนปิด Change Window ในคืนวันศุกร์ | Silent Failure ที่ไม่แสดงอาการจนถึงวันทำการถัดไป | สูง |
| Golden Config Template มีหลายเวอร์ชันที่คล้ายกันมาก (Data-Only vs Voice Closet) เสี่ยงเลือกผิด | ความเสี่ยง Human Error ในการ Deploy Hardware ใหม่ | กลาง |

#### 7. Action Items

| # | Action Item | Owner | Due Date | สถานะ |
|---|---|---|---|---|
| 1 | เพิ่ม Peer Review บังคับ (2-person rule) สำหรับ Config เปลี่ยนแปลงบน WAN Edge/CoPP/Prefix-list ทุกครั้ง | Network Manager | +2 สัปดาห์ | Open |
| 2 | สร้าง CoPP Template แยกสำหรับ WLC/Controller โดยเฉพาะ (รวม RADIUS/CAPWAP) แทนการ Copy จาก Switch Template | Security Engineer | +3 สัปดาห์ | Open |
| 3 | บังคับ Staged/Canary Rollout สำหรับ Change ที่กระทบมากกว่า 1 อุปกรณ์ Critical พร้อมกัน (Deploy ทีละตัว เว้น 24 ชม.) | Change Advisory Board (CAB) | +1 เดือน | Open |
| 4 | สร้าง Post-RMA Audit Checklist บังคับตรวจ Redundancy Feature ครบ (NHS สำรอง, Backup Route, HSRP Standby) ก่อนปิด Ticket RMA | Field Ops Lead | +2 สัปดาห์ | Open |
| 5 | เพิ่ม Automated Config Compliance Check (เทียบ Running-config กับ Golden Template ผ่าน Ansible ทุกคืน) แจ้งเตือนถ้าเจอ Drift เช่น NHS ขาดหาย หรือ QoS Trust หายไป | Automation Team | +6 สัปดาห์ | Open |
| 6 | ทำ Change Window Post-Deploy Smoke Test บังคับ (ทดสอบ Traffic จริงตัวอย่างก่อนปิด Window ทุกครั้ง) | ทุกทีมที่ทำ Change | ทันที | Open |
| 7 | Escalate สาย Fiber Last-Mile ของ BRANCH2-RTR กับผู้ให้บริการ ISP พร้อม SLA Penalty Clause | Vendor Management | +1 สัปดาห์ | In Progress |
| 8 | รวม Golden Config Template เวอร์ชัน "Data-Only" และ "Voice Closet" เป็น Template เดียวที่ Detect อัตโนมัติผ่าน CDP ว่ามี Phone ต่ออยู่หรือไม่ (ลด Human Error ในการเลือก Template) | Data Center/Hardware Team | +4 สัปดาห์ | Open |

#### 8. บทเรียนสำคัญที่สุดของ Incident นี้

> Incident นี้พิสูจน์ทั้งสองด้านของหลักการเดียวกัน: **Root Cause 1 และ 4 แสดงให้เห็นว่า Change
> Management ที่ขาดวินัย (ไม่มี Peer Review, ไม่มี Staged Rollout) สามารถสร้างปัญหาที่ดูเหมือนไม่
> เกี่ยวข้องกันในโดเมนที่ต่างกันโดยสิ้นเชิง (BGP vs Wireless) จาก Change Ticket เดียวกัน** ในขณะที่
> **Root Cause 2 และ 3 แสดงให้เห็นว่าความใกล้เคียงกันทางเวลาไม่ได้แปลว่ามีสาเหตุเดียวกันเสมอไป**
> ทักษะที่สำคัญที่สุดของวิศวกรระดับ ENARSI ไม่ใช่การรู้จักทุก Technology เพียงอย่างเดียว แต่คือ
> **ความสามารถในการแยกแยะ Correlation จาก Causation ด้วย Evidence จริงทุกครั้ง** ไม่ว่าจะเป็นการ
> ยืนยันว่าสองสิ่งเกี่ยวข้องกัน หรือยืนยันว่าไม่เกี่ยวข้องกันก็ตาม

---

## แบบฝึกหัดทวนความเข้าใจ Part 69

1. เพราะเหตุใด `ip prefix-list ISP-IN-PFX seq 5 permit 0.0.0.0/0 ge 1` จึงไม่ Match กับ Route
   `0.0.0.0/0` เอง ทั้งที่ดูเหมือนควรจะ Match ได้?
2. ในการ Investigate ปัญหา Voice (Root Cause 2) เพราะเหตุใด Technical Lead B จึงต้องทดสอบโทรระหว่าง
   Extension ภายในก่อนสรุปว่าไม่เกี่ยวกับ Root Cause 1 — ถ้าข้ามขั้นตอนนี้ไปจะเกิดอะไรขึ้น?
3. คำสั่ง `ip nhrp shortcut` ที่ BRANCH2-RTR ทำหน้าที่อะไร และเพราะเหตุใดการมีคำสั่งนี้อยู่แล้วยัง
   ไม่เพียงพอที่จะทำให้ Dual-Hub Failover ทำงานได้ในเหตุการณ์นี้?
4. เพราะเหตุใด CoPP บน WLC-1 (Catalyst 9800) จึงส่งผลกับ RADIUS Traffic ได้ ทั้งที่ CoPP ปกติไม่ควร
   กระทบ Traffic ที่เป็นแค่ Transit (วิ่งผ่านอุปกรณ์เฉยๆ)?
5. จากการวิเคราะห์ Cross-Cutting ใน Step 687 เหตุใด Root Cause 1 และ 4 จึงถือว่า "เชื่อมกันจริง"
   ในขณะที่ Root Cause 2 และ 3 ถือว่า "เป็นเรื่องบังเอิญ" — ใช้ Evidence อะไรเป็นตัวชี้ขาด?

**เฉลย:**

1. เพราะ Keyword `ge`/`le` ใน Prefix-list เปลี่ยนความหมายการ Match จาก "Prefix ตัวนี้ตรงตัวเป๊ะ"
   ไปเป็น "Prefix ที่เป็น Subnet ของ Base Prefix ในช่วง Length ที่กำหนด" — `ge 1` หมายถึง Length
   ตั้งแต่ 1 บิตขึ้นไป ซึ่ง**ไม่รวม Base Prefix เดิมที่มี Length เท่ากับ 0** (คือ `0.0.0.0/0` เอง)
   ดังนั้น Route Default ตัวจริงจึงตกไปโดน Implicit Deny ท้าย Prefix-list แม้ Prefix อื่นที่ยาวกว่า
   จะผ่านได้ปกติก็ตาม
2. เพื่อพิสูจน์สมมติฐานด้วย Evidence จริง ไม่ใช่การเดาจาก Correlation ทางเวลา — Voice ภายในไม่ได้
   พึ่งพา Internet/Default Route เลย ถ้าข้ามการทดสอบนี้ไปและสรุปผิดว่า "Root Cause 1 คือสาเหตุของ
   ทุกอย่าง" ทีมจะรอดูว่า Voice หายไปเองหลัง Fix ข้อ 1 (ซึ่งจะไม่หาย) เสียเวลาไปโดยไม่ได้ Investigate
   Root Cause ที่แท้จริงเลย ทำให้ Ticket INC-40233 ยังเปิดค้างอยู่นานขึ้นโดยไม่จำเป็น
3. `ip nhrp shortcut` เปิดให้ Spoke ยอมรับ NHRP Redirect จาก Hub แล้วสร้าง Path ที่สั้นกว่า (Spoke-
   to-Spoke หรือ Direct Path) ตามหลักการ DMVPN Phase 3 — แต่มันไม่ได้สร้าง Backup Hub Adjacency ขึ้น
   มาเอง หน้าที่นั้นเป็นของคำสั่ง `ip nhrp nhs` ที่ต้องมีครบทั้งสองบรรทัด (ไปยัง WAN-EDGE-1 และ
   WAN-EDGE-2) เมื่อ BRANCH2-RTR ขาด NHS ตัวที่สอง จึงไม่มี Path/Neighbor สำรองให้ Shortcut ไปหาได้
   เลยตั้งแต่ต้น ไม่ว่า `ip nhrp shortcut` จะ Config ถูกต้องแค่ไหนก็ตาม
4. เพราะ WLC-1 (Catalyst 9800-CL) รัน IOS-XE และเป็น**คู่สนทนาโดยตรง**ของ RADIUS Conversation เอง
   (ส่ง Access-Request ออกไปหา ISE-1 และรับ Access-Accept/Reject กลับมาที่ IP ของตัวเอง) Traffic
   ที่ปลายทางเป็น IP ของอุปกรณ์เอง (Self-destined) จะถูก Punt ขึ้น Control-Plane/CPU ของอุปกรณ์นั้น
   เสมอ ซึ่งเป็น Traffic ประเภทที่ CoPP ถูกออกแบบมาปกป้อง (ต่างจาก Traffic ที่แค่ผ่านอุปกรณ์ไปหา
   ปลายทางอื่นซึ่งถูก Forward ด้วย CEF/Hardware โดยไม่ผ่าน CPU และไม่ถูก CoPP แตะเลย)
5. Evidence ที่ชี้ขาดคือ **Change Log/Timestamp ของ Change Ticket จริง** — Root Cause 1 และ 4 มี
   `show archive config differences` ที่ยืนยันว่าถูก Push เข้าไปในวันเวลาเดียวกัน (ศุกร์ 16:00-16:30)
   จาก Change Ticket เดียวกันของทีม Security ส่วน Root Cause 2 มี Change Ticket คนละใบ คนละทีม
   คนละวัน (เสาร์ 09:00-11:00 โดยทีม Hardware) และ Root Cause 3 ไม่มี Change Ticket ที่เกี่ยวข้องกับ
   วันศุกร์เลย (เป็น Config Drift เก่าที่ถูกเปิดเผยโดยสาย Fiber ของ ISP ซึ่งเป็นเหตุภายนอก) — การ
   ตัดสินว่าเชื่อมกันหรือไม่ต้องอิงหลักฐานที่ตรวจสอบได้จริงเท่านั้น ไม่ใช่ความรู้สึกว่า "เกิดวันเดียว
   กันน่าจะเกี่ยวกัน"

---

## สรุป Part 69

Part นี้คือ Capstone-style Mega-Lab แรกของช่วง ENARSI ที่รวมทักษะ Troubleshooting จาก **Part
56-68 ทั้งหมด** เข้ามาไว้ใน Incident เดียวที่มี 4 Root Cause ซ้อนกัน — ต่างจาก Part ก่อนหน้าที่แยก
โดเมนให้ชัดเจนทีละ Feature Part นี้จำลอง **สภาพความยุ่งเหยิงของ Incident จริง** ที่ Ticket มาพร้อมกัน
โดยไม่มีใครบอกล่วงหน้าว่า Root Cause คือกี่จุดหรือเชื่อมกันหรือไม่ ทักษะหลักที่ Part นี้ปลูกฝังคือ
**War-Room Process ที่มีโครงสร้าง**, **การพิสูจน์สมมติฐานด้วย Evidence แทนการเดาจาก Correlation**,
**การจัดลำดับ Remediation ตาม Business Impact/Dependency พร้อม Rollback Plan**, **Full Health-Check
Sweep ก่อนปิด Incident เสมอ**, และ **การเขียน PIR ที่มีคุณภาพระดับที่ส่งให้ผู้บริหารอ่านได้จริง** —
ทักษะเหล่านี้สำคัญไม่น้อยไปกว่าความรู้ Technical เพราะคือสิ่งที่แยกวิศวกรระดับ Senior/CCNP ออกจาก
วิศวกรที่รู้ Feature แต่ยังไม่เคยผ่านสถานการณ์ Incident ขนาดใหญ่จริงมาก่อน

**พร้อมสำหรับ Part 70**: เราจะปิดท้ายระดับ **CCNP ENARSI (300-410)** ด้วย **Mock Exam เต็มรูปแบบ**
ที่จำลองข้อสอบจริง (Ticket-based Troubleshooting Format) ครอบคลุมทุกโดเมนตั้งแต่ Part 56-69 พร้อม
เฉลยละเอียดและเทคนิคการทำข้อสอบ ENARSI ให้ทันเวลาในห้องสอบจริง

**ไปต่อ:** [Part 70 — ENARSI Capstone & Mock Exam →](part-070-enarsi-capstone-mock-exam.md)
