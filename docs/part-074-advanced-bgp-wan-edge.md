# Part 74 — Advanced BGP for Enterprise WAN Edge
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 731–740 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 73 — Large-Scale Routing Design (OSPF+BGP+EIGRP)](part-073-large-scale-routing-design.md)
> ที่ออกแบบ **BGP เป็น "กาว" (Glue Protocol)** เชื่อม IGP หลายตัว (OSPF ที่ Campus, EIGRP ที่
> Data Center, ฯลฯ) เข้าด้วยกันในองค์กรขนาดใหญ่ระดับ CCIE — Part นี้จะ**ดำดิ่งกลับเข้าไปใน BGP
> เอง** และเจาะลึกทุกกลไกที่ [Part 29](part-029-bgp-fundamentals.md) (eBGP พื้นฐาน) และ
> [Part 30](part-030-bgp-advanced.md) (iBGP, Route Reflector แบบ Conceptual, Best Path,
> Community เบื้องต้น) ยังไม่ลงลึกพอสำหรับข้อสอบ **CCIE Enterprise Infrastructure Lab**: การ
> ออกแบบ **Route Reflector Cluster ระดับ Production จริง** (หลาย Cluster, Hierarchical RR),
> **BGP Add-Path** (โฆษณาหลาย Path ให้ Neighbor เดียวกัน), **BGP PIC** (Failover ที่ไม่ต้องรอ
> BGP เดินตาม Prefix ทีละตัว), **BGP Unnumbered/Dynamic Peering**, ภาพรวม **BGP สำหรับ Multi-Cloud
> Connectivity**, **Graceful Restart/NSF สำหรับ BGP โดยเฉพาะ**, **RPKI Origin Validation**,
> **Community-Based Policy Framework** ระดับ Enterprise จริง, และ**การ Troubleshoot BGP Table
> ขนาดหลักพัน/หลักหมื่น Prefix อย่างมีประสิทธิภาพ**
>
> Part นี้ยังคงใช้ Topology หลักของหลักสูตร (**WAN-EDGE-1/WAN-EDGE-2 ↔ ISP-RTR** สำหรับ eBGP,
> **WAN-EDGE-1/2 ↔ CORE-SW1/2** สำหรับ iBGP) ตาม
> [00-ip-address-plan.md](00-ip-address-plan.md) เป็นแกนหลักทุก Config จริง — ส่วนหัวข้อที่ต้อง
> แสดงการ Scale เกินกว่า 4 Router ของ Lab (เช่น Route Reflector หลาย Cluster) จะระบุไว้อย่าง
> ชัดเจนว่าเป็น **"ส่วนขยายเชิงแนวคิด" (Conceptual Extension)** เหมือนที่ Part 30 Step 293.4 เคย
> ทำไว้ เพื่อสาธิตหลักการระดับ Enterprise ขนาดใหญ่โดยไม่ต้องเปลี่ยน Physical Topology ของหลักสูตร

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 731 | Route Reflector Cluster ระดับ Enterprise จริง — หลาย Cluster, Cluster-ID Design, Hierarchical RR (RR of RRs) |
| 732 | BGP Add-Path — โฆษณาหลาย Path ให้ Neighbor เดียวกัน เพื่อ Load-Balancing/Fast Convergence ใน RR-Heavy Design |
| 733 | BGP PIC (Prefix-Independent Convergence) — Failover ที่ไม่ต้อง Walk ทุก Prefix |
| 734 | BGP Dynamic Peering / BGP Unnumbered — ลดภาระ Config ที่ WAN Edge ด้วย Peer-Group และ IPv6 Link-Local eBGP |
| 735 | BGP สำหรับ Multi-Cloud Connectivity (Preview) — eBGP บน Direct Connect/ExpressRoute-style Circuit |
| 736 | Graceful Restart และ BGP NSF — รักษา Session ไว้ระหว่าง RP Switchover/Software Upgrade |
| 737 | BGP Security Hardening — RPKI Origin Validation (ภาพรวมสำหรับ Enterprise WAN Edge) |
| 738 | Real-World BGP Policy Design Pattern — Community เป็นภาษา Policy (Site/Region/Service-Tier) |
| 739 | BGP Troubleshooting ที่ Scale — เทคนิค Filter ตาราง BGP หลักพัน/หมื่น Prefix อย่างมีประสิทธิภาพ |
| 740 | Lab เต็มรูปแบบ: รีดีไซน์ Dual-ISP + iBGP Mesh จาก Part 30 เป็น Scaled Community Policy Framework พร้อม Add-Path + PIC |

---

## Step 731 — Route Reflector Cluster ระดับ Enterprise จริง

### 731.1 ทบทวนข้อจำกัดของ Part 30

[Part 30 Step 293](part-030-bgp-advanced.md#step-293--route-reflector-rr--แก้ปัญหา-full-mesh-scaling)
สอน RR แบบ **Single Cluster เดียว** (WAN-EDGE-1 เป็น RR ให้ CORE-SW1/CORE-SW2) และบอกไว้ตรงๆ ว่า
Lab หลักสูตรมี iBGP Router แค่ 4 ตัวจึงยังไม่จำเป็นต้องใช้ RR จริงจัง — ระดับ **CCIE Enterprise
Infrastructure** ต้องออกแบบ RR ให้รองรับองค์กรที่มี iBGP Router **หลักสิบถึงหลักร้อยตัว** กระจาย
หลาย Site/Region ซึ่งมีข้อกำหนดที่ Single Cluster เดียวตอบไม่ได้:

| ข้อจำกัดของ Single-Cluster RR | ทำไมเป็นปัญหาที่ Enterprise ขนาดใหญ่ |
|---|---|
| RR ตัวเดียวเป็น Single Point of Failure | ถ้า RR ล่ม Client ทุกตัวในกลุ่มขาดการเชื่อมต่อ iBGP ทั้งหมดทันที |
| RR ตัวเดียวต้องเก็บ Full BGP Table ของ Client **ทุกตัว** | เมื่อจำนวน Client เพิ่มเป็นร้อยตัว RAM/CPU ของ RR ตัวเดียวรับภาระไม่ไหว |
| ไม่มีการแบ่งตาม Geography/Region | Router ที่ Sydney กับ Router ที่ Bangkok ไม่ควร Peer RR ตัวเดียวกันข้าม WAN ที่มี Latency สูง |

### 731.2 หลักการ: Redundant RR Pair ต่อ 1 Cluster เสมอ

กฎเหล็กของการออกแบบ RR ระดับ Production คือ **ทุก Cluster ต้องมี RR อย่างน้อย 2 ตัว** (Redundant
Pair) ที่ใช้ **Cluster-ID เดียวกัน** โดยตั้งใจ — Client แต่ละตัว Peer กับ RR **ทั้งสองตัว**ในคู่
เพื่อไม่มี Single Point of Failure:

```
router bgp 65001
 bgp cluster-id 1.1.1.100      ! ทั้ง RR-1 และ RR-2 ในคู่เดียวกัน ใช้ Cluster-ID นี้เหมือนกัน
```

> **ทำไม Cluster-ID ต้องเหมือนกันในคู่ Redundant Pair**: ORIGINATOR_ID/CLUSTER_LIST (Part 30
> Step 293.5) ใช้ Cluster-ID เป็นตัวกันลูป — ถ้า RR-1 และ RR-2 ใช้ Cluster-ID **คนละค่า** ตัว
> Client จะได้รับ Route **ชุดเดิม 2 ครั้ง** ผ่าน Cluster-List ที่ต่างกัน (มองว่าเป็น Path ต่างกัน
> ไม่ Suppress ซ้ำ) ทำให้ตาราง BGP บวมเป็น 2 เท่าโดยไม่จำเป็น — ตั้ง Cluster-ID เหมือนกันคือ
> วิธีบอก BGP ว่า "RR ทั้งสองตัวนี้คือ Cluster เดียวกัน ทำหน้าที่ซ้ำกัน (Redundant) เท่านั้น"

### 731.3 การออกแบบ Hierarchical RR (RR of RRs) — 2-Tier สำหรับหลักสูตรนี้

Enterprise ขนาดใหญ่มักออกแบบ RR เป็น **2 ระดับ (Tier)**:

| Tier | บทบาท | ตัวอย่างในหลักสูตร (ของจริง) |
|---|---|---|
| **Tier 1 — Global/Core RR** | RR ของ RR อีกที — Peer กันเองแบบ Full-Mesh เล็กๆ (จำนวนน้อย) และเป็น RR ให้ Regional RR ของแต่ละ Site | **WAN-EDGE-1 + WAN-EDGE-2** (Cluster-ID `1.1.1.100`) |
| **Tier 2 — Regional RR** | RR ให้ Client ภายใน Region/Site เดียวกัน และเป็น **RR-Client ของ Tier 1** พร้อมกัน | **CORE-SW1 + CORE-SW2** (Cluster-ID `1.1.1.101`) |
| **Client ปลายทาง** | Router ทั่วไปที่ Peer กับ Regional RR เท่านั้น ไม่ต้อง Full-Mesh กับใครเลย | **DIST-SW1–DIST-SW4** |

```
                     ┌─────────────────────────────────────────┐
                     │   Tier 1 — GLOBAL RR (Cluster 1.1.1.100)   │
                     │      WAN-EDGE-1 <====iBGP====> WAN-EDGE-2   │
                     └──────────┬─────────────────────┬──────────┘
                   RR ให้ CORE-SW1/2 (RR-Client ของ Tier1)
                     ┌──────────┴──────────┐ ┌─────────┴──────────┐
                     │  Tier 2 — REGIONAL   │ │  Tier 2 — REGIONAL  │
                     │  RR (Cluster .101)   │ │  RR (Cluster .101)   │
                     │      CORE-SW1         │ │      CORE-SW2        │
                     └──┬────────────┬──────┘ └──────┬───────────┬──┘
                   RR-Client      RR-Client       RR-Client    RR-Client
                 ┌──────┴───┐  ┌───┴──────┐    ┌─────┴────┐ ┌───┴──────┐
                 │DIST-SW1  │  │DIST-SW2  │    │DIST-SW3  │ │DIST-SW4  │
                 └──────────┘  └──────────┘    └──────────┘ └──────────┘
```

จุดสำคัญของโครงสร้างนี้: **CORE-SW1 และ CORE-SW2 สวมหมวก 2 บทบาทพร้อมกัน** — เป็น
**RR-Client** เมื่อคุยกับ WAN-EDGE-1/2 (Tier 1) และเป็น **RR** เองเมื่อคุยกับ DIST-SW1-4 (Tier 2)
ซึ่งเป็นสาระสำคัญของคำว่า "Hierarchical" — Route จาก DIST-SW1 เดินทาง `DIST-SW1 → CORE-SW1(RR)
→ WAN-EDGE-1(RR) → WAN-EDGE-2 → CORE-SW2(RR) → DIST-SW3` โดยไม่ต้องมี Session ตรงระหว่าง
DIST-SW1 กับ DIST-SW3 เลย

### 731.4 Worked Config — Tier 2: CORE-SW1 เป็น Regional RR ให้ DIST-SW1/DIST-SW2

```
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# bgp router-id 1.1.1.1
CORE-SW1(config-router)# bgp cluster-id 1.1.1.101
CORE-SW1(config-router)# neighbor 1.1.1.11 remote-as 65001
CORE-SW1(config-router)# neighbor 1.1.1.11 update-source Loopback0
CORE-SW1(config-router)# neighbor 1.1.1.11 description ** iBGP RR-Client: DIST-SW1 **
CORE-SW1(config-router)# neighbor 1.1.1.12 remote-as 65001
CORE-SW1(config-router)# neighbor 1.1.1.12 update-source Loopback0
CORE-SW1(config-router)# neighbor 1.1.1.12 description ** iBGP RR-Client: DIST-SW2 **
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# neighbor 1.1.1.11 activate
CORE-SW1(config-router-af)# neighbor 1.1.1.11 next-hop-self
CORE-SW1(config-router-af)# neighbor 1.1.1.11 route-reflector-client
CORE-SW1(config-router-af)# neighbor 1.1.1.12 activate
CORE-SW1(config-router-af)# neighbor 1.1.1.12 next-hop-self
CORE-SW1(config-router-af)# neighbor 1.1.1.12 route-reflector-client
CORE-SW1(config-router-af)# exit-address-family
```

**CORE-SW2 คือกระจกเงา** (Mirror) ของ Config ข้างบน แต่เป็น RR ให้ DIST-SW3/DIST-SW4 และใช้
Cluster-ID `1.1.1.101` **ค่าเดียวกัน** เพราะ CORE-SW1/CORE-SW2 คือ Redundant Pair ของ Tier 2
(แม้ในทาง Topology จริง CORE-SW1 ดูแล DIST-SW1/2 และ CORE-SW2 ดูแล DIST-SW3/4 คนละกลุ่ม — แต่
Cluster-ID เดียวกันยังจำเป็นเพื่อให้ Tier 1 มองทั้งคู่เป็น "แหล่งเดียวกัน" ตอน Reflect ข้ามกลับลงมา)

### 731.5 Worked Config — Tier 1: WAN-EDGE-1 เป็น Global RR ให้ CORE-SW1/CORE-SW2

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp router-id 1.1.1.21
WAN-EDGE-1(config-router)# bgp cluster-id 1.1.1.100
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 description ** iBGP RR-Client (Tier1): CORE-SW1 **
WAN-EDGE-1(config-router)# neighbor 1.1.1.2 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.2 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.2 description ** iBGP RR-Client (Tier1): CORE-SW2 **
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 description ** iBGP Full-Mesh Peer (Tier1 คู่ RR): WAN-EDGE-2 **
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 next-hop-self
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 route-reflector-client
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 next-hop-self
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 route-reflector-client
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 activate
WAN-EDGE-1(config-router-af)# exit-address-family
```

สังเกตว่า `neighbor 1.1.1.22` (WAN-EDGE-2, คู่ RR ของกันและกันใน Tier 1) **ไม่มี**
`route-reflector-client` — เพราะทั้งสองตัวเป็น RR ระดับเดียวกัน (Non-Client ของกันและกัน) ต้อง
Full-Mesh กันตามปกติ (คู่เดียวจึงยังจัดการง่าย)

### 731.6 การขยายเกิน 4 Router ของ Lab — Multiple Regional Cluster (ส่วนขยายเชิงแนวคิด)

ในองค์กรที่มีหลาย Site จริง (เกินกว่าที่ Lab หลักสูตรมี Hardware ให้) รูปแบบเดียวกันนี้ขยายได้
โดยเพิ่ม **Regional RR Cluster ใหม่ต่อ Site** แล้ว Peer ขึ้นไปที่ Tier 1 เดิม:

| Cluster | Cluster-ID | RR Pair (ตัวอย่างสมมติ) | Client |
|---|---|---|---|
| Tier 1 (Global) | `1.1.1.100` | WAN-EDGE-1 / WAN-EDGE-2 | CORE-SW1, CORE-SW2, และ Regional-RR ของทุก Site |
| Tier 2 — Site HQ | `1.1.1.101` | CORE-SW1 / CORE-SW2 | DIST-SW1–4 |
| Tier 2 — Site B (สมมติ) | `1.1.1.102` | Regional-RR-B1 / Regional-RR-B2 | Router ภายใน Site B ทั้งหมด |
| Tier 2 — Site C (สมมติ) | `1.1.1.103` | Regional-RR-C1 / Regional-RR-C2 | Router ภายใน Site C ทั้งหมด |

> **ข้อสำคัญ**: Cluster-ID ของแต่ละ Regional Cluster **ต้องไม่ซ้ำกัน** ข้าม Site (ต่างจาก
> Redundant Pair ภายใน Cluster เดียวกันที่ตั้งใจให้ซ้ำ) — ถ้า Site B และ Site C ใช้ Cluster-ID
> ชนกัน BGP จะเข้าใจผิดว่าเป็น Cluster เดียวกันและอาจ Suppress Route ที่ไม่ควร Suppress
> (False Loop Detection) ทำให้บาง Prefix หายไปจากบาง Region โดยไม่มีสาเหตุชัดเจน — นี่คือ
> Checklist ข้อแรกที่ต้องตรวจเวลาออกแบบ RR หลาย Cluster ในข้อสอบ CCIE Lab

---

## Step 732 — BGP Add-Path: โฆษณาหลาย Path ให้ Neighbor เดียวกัน

### 732.1 ปัญหาที่ Add-Path แก้: RR ปกติส่งได้แค่ 1 Best Path

Behavior ปกติของ BGP (ทั้ง eBGP และ iBGP) คือ **Router จะโฆษณา Best Path เพียง 1 เส้นทางต่อ
Prefix ให้ Neighbor แต่ละตัว** แม้ Router นั้นจะมีหลาย Path ที่ดีพอๆ กันอยู่ใน BGP Table ก็ตาม —
นี่คือพฤติกรรมที่ยิ่งเป็นปัญหาชัดเจนขึ้นในโครงสร้าง **RR-Heavy** อย่างที่ออกแบบใน Step 731:

```
สถานการณ์: WAN-EDGE-1 (Global RR) มี Path ไปยัง 0.0.0.0/0 อยู่ 2 เส้นทางที่ดีเท่ากัน
  Path A: ผ่าน ISP-RTR โดยตรง (eBGP, Learned ที่ WAN-EDGE-1 เอง)
  Path B: ผ่าน WAN-EDGE-2 (iBGP, ซึ่ง WAN-EDGE-2 เรียนจาก ISP-RTR อีก Link)

ปกติ WAN-EDGE-1 (RR) จะเลือก Best Path เพียง 1 เส้น (สมมติ Path A ชนะ) แล้ว Reflect ให้
CORE-SW1/CORE-SW2 (RR-Client) **เห็นแค่ Path A** เท่านั้น — ถ้า Path A ล่ม CORE-SW1/2 ต้องรอ
WAN-EDGE-1 คำนวณ Best Path ใหม่ (สลับเป็น Path B) แล้ว Advertise Update รอบใหม่ทั้งหมด
```

ผลกระทบ: (1) **ไม่มี ECMP ที่ปลาย Client** เพราะเห็น Path เดียว (2) **Convergence ช้าลง** เพราะ
ต้องรอ RR คำนวณและ Re-advertise Path ใหม่ทุกครั้งที่ Best Path เปลี่ยน — ยิ่งมี Client จำนวนมาก
(อย่าง CORE-SW1/2 ที่เป็น RR-Client ของ Tier 1 พร้อมทำหน้าที่ RR ของ Tier 2 ต่อ) การ Re-converge
แบบ Sequential นี้ยิ่งขยายเวลารวมมากขึ้นตามจำนวน Tier

### 732.2 BGP Add-Path (RFC 7911) คืออะไร

**Add-Path** คือ BGP Capability ที่ต่อรองกันตอน OPEN Message — เมื่อเปิดใช้ Router จะแนบ
**Path-ID** (เลข 4 byte) ไปกับทุก Path ที่โฆษณา ทำให้สามารถส่ง **หลาย Path สำหรับ Prefix
เดียวกัน** ไปยัง Neighbor ตัวเดียวได้ในเวลาเดียวกัน (Neighbor แยกแต่ละ Path ด้วย Path-ID)

| โหมด | ความหมาย |
|---|---|
| `receive` | ยอมรับ Add-Path จาก Neighbor (รับได้หลาย Path) |
| `send` | ส่ง Add-Path ให้ Neighbor (โฆษณาได้หลาย Path) |
| `advertise-best <n>` | ส่งเฉพาะ N Path ที่ดีที่สุดเท่านั้น (ไม่ใช่ทุก Path ที่มี) |
| `advertise-best all` (เทียบเท่า `bgp additional-paths select all`) | ส่งทุก Path ที่ผ่านเกณฑ์ (ทั่วไปจะ Cap ด้วยจำนวน ECMP ที่ตั้งไว้) |

### 732.3 Worked Config: WAN-EDGE-1 (Global RR) ส่ง Add-Path ให้ CORE-SW1/CORE-SW2

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# bgp additional-paths select all
WAN-EDGE-1(config-router-af)# bgp additional-paths install
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 additional-paths send
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 additional-paths receive
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 advertise additional-paths all
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 additional-paths send
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 additional-paths receive
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 advertise additional-paths all
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 additional-paths send
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 additional-paths receive
WAN-EDGE-1(config-router-af)# exit-address-family
```

- `bgp additional-paths select all` — สั่งให้ Router **คำนวณ (Select)** ว่า Path ไหนบ้างที่
  "ดีพอ" จะเป็น Additional Path (นอกเหนือ Best Path ปกติ) — ค่านี้ต้องตั้งก่อน `advertise`
  จึงจะมี Path ให้เลือกส่ง
- `bgp additional-paths install` — ติดตั้ง Path เพิ่มเติมเข้า **RIB** ด้วย (ไม่ใช่แค่เก็บใน BGP
  Table) ซึ่งเป็นกลไกที่เชื่อมโยงตรงกับ **PIC** ใน Step 733 — ถ้าไม่ Install เข้า RIB, Path
  สำรองจะยังอยู่แค่ในตาราง BGP ใช้ทำ ECMP/PIC ไม่ได้จริง

### 732.4 Verification

```
CORE-SW1# show bgp ipv4 unicast 0.0.0.0/0
BGP routing table entry for 0.0.0.0/0, version 12
Paths: (2 available, best #1, table default)
  Advertised to update-groups:
     1
  Path #1: Received by speaker 0
  Not advertised to any peer
  Local
    1.1.1.21 (metric 20) from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 100, valid, internal, best
      rx pathid: 0, tx pathid: 0x0
  Path #2: Received by speaker 0
  Local
    1.1.1.21 (metric 20) from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 100, valid, internal, backup
      rx pathid: 0x1, tx pathid: 0

CORE-SW1# show ip bgp neighbors 1.1.1.21 additional-paths
```

- `rx pathid`/`tx pathid` ไม่เท่ากับ 0 หมายความว่า Path นี้มาถึงผ่าน Add-Path Capability จริง
- คำว่า `backup` ในบรรทัด Attribute คือสัญญาณว่า Path #2 ถูกเก็บไว้เป็น **Backup Path พร้อมใช้
  ทันที** — เชื่อมต่อไปยังกลไก PIC ที่จะอธิบายต่อใน Step 733

### 732.5 ทำไม Add-Path เหมาะกับ RR-Heavy Design เป็นพิเศษ

| ประเด็น | Full-Mesh (ไม่มี RR) | Single-Path RR (Part 30) | RR + Add-Path (Step นี้) |
|---|---|---|---|
| จำนวน Session | มากที่สุด (Scale ไม่ได้) | น้อย (Scale ได้) | น้อย (เท่า RR) |
| Client เห็นกี่ Path | ทุก Path จริง (เพราะ Peer ตรง) | **แค่ 1 Path** (ข้อจำกัดสำคัญ) | **หลาย Path** (ตามที่ RR เลือกส่ง) |
| ECMP ที่ Client | ได้เต็มรูปแบบ | ไม่ได้ (เห็น Path เดียว) | **ได้กลับมา** โดยไม่ต้องเสีย Scale ของ RR |
| ความเร็ว Failover | เร็ว (Client รู้ทุก Path เอง) | ช้า (ต้องรอ RR คำนวณใหม่) | **เร็ว** (Client มี Backup Path พร้อมอยู่แล้ว) |

Add-Path จึงคือคำตอบที่ทำให้องค์กรได้ **ทั้ง Scale ของ RR และ Convergence ที่เร็วของ Full-Mesh**
พร้อมกัน — เป็นเทคนิคมาตรฐานของ Enterprise/ISP ขนาดใหญ่ทุกรายที่ใช้ RR จริงจัง

---

## Step 733 — BGP PIC (Prefix-Independent Convergence)

### 733.1 ปัญหาที่ PIC แก้: Convergence แบบ Linear ตามจำนวน Prefix

Behavior ปกติเมื่อ BGP Path หลักล่ม (เช่น eBGP Session ไปยัง ISP1 Down) คือ Control Plane ต้อง:

```
1. ตรวจพบว่า Neighbor ตาย (Hold Timer หมด หรือ BFD แจ้ง)
2. เดินตาราง BGP (BGP Table Walk) หา **ทุก Prefix** ที่ Best Path ชี้ไปที่ Neighbor ตัวนั้น
3. สำหรับแต่ละ Prefix: คำนวณ Best Path ใหม่ (รัน Best Path Algorithm ซ้ำ)
4. Update RIB → Update FIB (CEF) ทีละ Prefix
5. Advertise Update ออกไปให้ Neighbor อื่นทีละ Prefix
```

ปัญหาคือขั้นตอนนี้ **ใช้เวลาเป็นสัดส่วนตรงกับจำนวน Prefix** (Linear Time, O(n)) — ในบริบท ISP
ที่มี Full Table 900,000+ Prefix (ตามที่กล่าวถึงใน [Part 29 Step 281](part-029-bgp-fundamentals.md#step-281--bgp-คืออะไร-path-vector-protocol-และเหตุผลที่ต้องใช้ที่-wan-edgemultihoming))
Reconvergence แบบนี้อาจใช้เวลาหลายสิบวินาทีถึงเป็นนาที — แม้ในบริบท Enterprise ที่ตาราง BGP
เล็กกว่ามาก (หลักร้อยถึงหลักพัน Prefix จาก Data Center/Multi-Site ตาม Part 73) เวลานี้ก็ยังส่งผล
กระทบต่อ SLA ของ Application สำคัญได้ถ้าไม่ควบคุม

### 733.2 แนวคิดของ PIC: แยก "ตรวจจับ + สลับ Forwarding" ออกจาก "คำนวณ Control Plane"

**PIC (Prefix-Independent Convergence)** ทำให้เวลา Failover **ไม่ขึ้นกับจำนวน Prefix** โดยใช้
หลักการ **Pre-computation** — คำนวณ Backup Path ไว้ **ล่วงหน้า** (ก่อนเกิด Failure จริง) แล้วฝัง
ลงใน FIB (CEF) ให้พร้อมสลับได้ทันทีที่ Hardware/Data Plane ระดับเดียว โดยไม่ต้องรอ Control
Plane เดินตารางทีละ Prefix เลย:

```
ไม่มี PIC (Default):                          มี PIC (Backup Path ติดตั้งไว้ล่วงหน้า):

Failure ────► ตรวจพบ ────► BGP Table Walk      Failure ────► ตรวจพบ ────► สลับ FIB Pointer
              (ทีละ Prefix, ช้าตามจำนวน)                       ไปยัง Backup ที่มีอยู่แล้วทันที
                    │                                                │
              คำนวณ Best Path ใหม่ทีละ Prefix                  (Prefix-Independent: ทุก Prefix
                    │                                            ที่ใช้ Backup เดียวกันสลับ
              Update FIB ทีละ Prefix                             พร้อมกันในเวลาเดียว ไม่ต้อง
                    │                                            รอคำนวณใหม่เลย)
              (เวลารวม = O(จำนวน Prefix))                      (เวลารวม ≈ คงที่)
```

### 733.3 PIC Edge vs PIC Core

| ประเภท | ป้องกันความล้มเหลวแบบไหน | กลไกที่ใช้ในหลักสูตรนี้ |
|---|---|---|
| **PIC Edge** | Edge Router/Exit Point ตัวหนึ่งล่ม (เช่น eBGP ไปยัง ISP1 ล่ม) — ต้องมี Path สำรองไปยัง Exit Point อื่น (ISP2) พร้อมอยู่แล้ว | **BGP Add-Path** (Step 732) + `bgp additional-paths install` — ทำให้ Backup Path จาก WAN-EDGE-2 ถูก Install ไว้ใน RIB/FIB ของ CORE-SW1/2 ล่วงหน้า |
| **PIC Core** | Link/Node ภายใน Core ล่ม (ระหว่าง BGP Next-Hop กับปลายทางจริง) — ต้องมี Path ทาง IGP สำรองไปยัง Next-Hop เดิม | **BGP Next-Hop Tracking** ผูกกับ IGP Fast Convergence (Loopback-based Peering ที่ Part 29/30 ใช้อยู่แล้ว ทำให้ Next-Hop Reachability ไม่ขาดแม้ Physical Link ล่ม เพราะมี Path สำรองผ่าน OSPF ECMP) |

### 733.4 Worked Config: เปิด PIC Edge ที่ CORE-SW1/CORE-SW2 (Client ของ Global RR)

```
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# bgp additional-paths install
CORE-SW1(config-router-af)# bgp additional-paths select all
CORE-SW1(config-router-af)# neighbor 1.1.1.21 additional-paths receive
CORE-SW1(config-router-af)# maximum-paths ibgp 2
CORE-SW1(config-router-af)# exit-address-family
```

- `bgp additional-paths install` (ฝั่งรับ) คือคำสั่งที่ **สำคัญที่สุด** ของ PIC Edge — สั่งให้
  Path สำรองที่ได้รับผ่าน Add-Path **ถูกฝังลง RIB จริง** (ไม่ใช่แค่เก็บไว้ในตาราง BGP เฉยๆ)
  พร้อมให้ CEF ใช้สลับได้ทันทีที่ Best Path หลักหาย
- `maximum-paths ibgp 2` เปิดให้ติดตั้งได้สูงสุด 2 Path เข้า RIB พร้อมกัน (ทั้ง Best และ Backup)

### 733.5 Verification: ดู Backup Path ใน CEF โดยตรง

```
CORE-SW1# show ip cef 0.0.0.0/0 detail
0.0.0.0/0, epoch 0, flags attached, rr, per-destination sharing
  attached to Loopback0
  nexthop 1.1.1.21 Port-channel1 (primary)
  nexthop 1.1.1.22 Port-channel1 (repair)          <-- Backup ที่ CEF ติดตั้งไว้ล่วงหน้า
    repair: attached-nexthop, fast-reroute capable

CORE-SW1# show bgp ipv4 unicast 0.0.0.0/0 additional-paths
```

คอลัมน์ `(repair)` คือหลักฐานว่า **CEF มี Path สำรองพร้อมสลับทันที** (Fast Reroute) โดยไม่ต้อง
พึ่งการคำนวณของ Control Plane เลยในขณะเกิด Failure จริง — นี่คือความแตกต่างเชิงคุณภาพจาก
Behavior ปกติที่ Part 29/30 สาธิตไว้ (`show ip route 0.0.0.0` สลับ Entry **หลังจาก** BGP
คำนวณเสร็จ ซึ่งช้ากว่า)

> **ข้อควรรู้สำหรับข้อสอบ CCIE**: PIC ไม่ได้แทนที่ BFD (Bidirectional Forwarding Detection) —
> ทั้งสองทำงานร่วมกัน: **BFD ทำหน้าที่ตรวจจับ Failure ให้เร็วที่สุด** (Sub-second) ส่วน **PIC
> ทำหน้าที่สลับ Forwarding ให้เร็วที่สุดเมื่อตรวจพบแล้ว** (ไม่ต้องรอ Control Plane) — Enterprise
> WAN Edge ระดับ Production ต้องเปิดทั้งสองอย่างคู่กันเสมอเพื่อ Convergence ที่เร็วที่สุดจริง

---

## Step 734 — BGP Dynamic Peering / BGP Unnumbered

### 734.1 ปัญหา: ภาระ Config ที่ WAN Edge เพิ่มขึ้นตามจำนวน Neighbor

โครงสร้าง Hierarchical RR ใน Step 731 ทำให้ CORE-SW1 ต้องมี Neighbor Statement ซ้ำๆ กันจำนวนมาก
(RR-Client ทุกตัวใช้ Policy เดียวกันหมด: `next-hop-self`, `route-reflector-client`,
`additional-paths receive`) — ยิ่งจำนวน DIST-SW/Site เพิ่มขึ้น การพิมพ์ Policy ซ้ำทุก Neighbor
ก็ยิ่งเสี่ยง Human Error (ลืมใส่บาง Option ใน Neighbor ตัวใดตัวหนึ่ง)

### 734.2 Peer-Group — รวม Policy ที่ใช้ซ้ำเข้าเป็นชุดเดียว

```
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST peer-group
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST remote-as 65001
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST update-source Loopback0
CORE-SW1(config-router)# neighbor 1.1.1.11 peer-group RR-CLIENT-DIST
CORE-SW1(config-router)# neighbor 1.1.1.11 description ** DIST-SW1 (RR-Client) **
CORE-SW1(config-router)# neighbor 1.1.1.12 peer-group RR-CLIENT-DIST
CORE-SW1(config-router)# neighbor 1.1.1.12 description ** DIST-SW2 (RR-Client) **
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST activate
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST next-hop-self
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST route-reflector-client
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST additional-paths receive
CORE-SW1(config-router-af)# neighbor 1.1.1.11 activate
CORE-SW1(config-router-af)# neighbor 1.1.1.12 activate
CORE-SW1(config-router-af)# exit-address-family
```

- Neighbor ที่ผูกกับ `RR-CLIENT-DIST` **รับ Policy ทั้งหมดจาก Peer-Group อัตโนมัติ** — ถ้าต้องการ
  เพิ่ม DIST-SW ตัวที่ 3 แค่เพิ่ม 2 บรรทัด (`remote-as`/`update-source` มาจาก Group อยู่แล้ว, ใส่
  แค่ `peer-group` + `activate`) แทนที่จะพิมพ์ Policy ทั้งชุดใหม่ทุกครั้ง
- เมื่อต้องแก้ Policy ทีหลัง (เช่น เพิ่ม `soft-reconfiguration inbound`) แก้ที่ Peer-Group **1
  ที่** มีผลกับทุก Client ในกลุ่มทันที — ลด Human Error ได้มาก

### 734.3 BGP Unnumbered — eBGP ผ่าน IPv6 Link-Local โดยไม่ต้องมี Global Address

เทคนิคสมัยใหม่อีกแบบที่ลดภาระ Config ที่ WAN Edge (นิยมมากในงาน Data Center Fabric/DMVPN Hub
สมัยใหม่ และเริ่มใช้ที่ WAN Edge ของ Enterprise ที่ Peer กับวงจรแบบ Point-to-Point จำนวนมาก) คือ
**BGP Unnumbered** — Peer eBGP กัน**ผ่าน IPv6 Link-Local Address** ที่ Auto-generate จาก MAC
Address ของ Interface เอง โดยไม่ต้องจัดสรร IPv4/IPv6 Global Address ให้ Link นั้นเลย

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# description ** eBGP Unnumbered ไปยัง Regional-CE ผ่าน P2P Circuit **
WAN-EDGE-1(config-if)# no ip address
WAN-EDGE-1(config-if)# ipv6 enable
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor GigabitEthernet0/0/2 remote-as 65010
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor GigabitEthernet0/0/2 activate
WAN-EDGE-1(config-router-af)# exit-address-family
```

| จุดสังเกต | คำอธิบาย |
|---|---|
| `neighbor <interface-name> remote-as <AS>` | ระบุ **ชื่อ Interface** แทน IP Address — BGP จะเรียนรู้ IPv6 Link-Local ของ Neighbor ผ่าน RA (Router Advertisement) ที่ IPv6 ส่งกันเองบน Link นั้นโดยอัตโนมัติ |
| `ipv6 enable` | เปิด IPv6 Processing บน Interface (สร้าง Link-Local ให้ตัวเองด้วย) — **จำเป็นแม้ Network นี้จะเป็น IPv4-only Service** เพราะ Unnumbered ใช้ IPv6 Link-Local เป็นแค่ "ท่อ" ขนส่ง TCP Session ของ BGP เท่านั้น NLRI ที่แลกกันยังเป็น IPv4 Unicast ตามปกติ |
| ไม่ต้องมี `update-source` | เพราะ Peer กันผ่าน Interface ตรงไม่ใช่ Loopback — BGP ผูก Session กับ Link-Local ของ Interface นั้นโดยตรง |

> **ข้อจำกัดสำคัญ**: BGP Unnumbered ใช้ได้กับ Link ที่เป็น **Point-to-Point แท้จริง** เท่านั้น
> (Physical P2P, Sub-interface แบบ Point-to-Point, หรือ Tunnel) — **ใช้กับ Multi-access Segment
> ไม่ได้** เพราะ `neighbor <interface>` สมมติว่ามี Neighbor เดียวอยู่ปลายสายเท่านั้น ต่างจากการ
> ระบุ IP Address ตรงที่รองรับ Topology แบบไหนก็ได้ — Enterprise WAN Edge ที่ต่อ Regional-CE
> ผ่าน MPLS/Metro-E แบบ Point-to-Point จำนวนมาก (สถานการณ์ทั่วไปของ Part 73) จึงเป็น Use Case
> ที่เหมาะกับเทคนิคนี้มากที่สุด — ไม่แนะนำใช้แทนที่ eBGP ไปยัง ISP-RTR ของหลักสูตรนี้ (ยังคงใช้
> IPv4 ตรงไปตรงมาตาม Step 283 เพราะต้องรองรับ Public Internet Routing เต็มรูปแบบ)

---

## Step 735 — BGP สำหรับ Multi-Cloud Connectivity (Preview)

### 735.1 ทำไม Cloud Provider ทุกรายใช้ eBGP เป็น Protocol หลักของ Private Connectivity

**AWS Direct Connect**, **Azure ExpressRoute**, และ **Google Cloud Interconnect/Partner
Interconnect** ทั้งหมดใช้หลักการเดียวกัน: WAN Edge ขององค์กร (**WAN-EDGE-1/2** ในบริบทหลักสูตร
นี้) สร้าง **eBGP Session** ไปยัง Router ของ Cloud Provider ผ่าน **Dedicated Circuit** (ไม่ใช่
Internet สาธารณะ) — เป็นการนำหลักการเดียวกันกับ Step 283 (eBGP ไปยัง ISP-RTR) มาใช้ แค่เปลี่ยน
ปลายทางจาก "ISP" เป็น "Cloud Provider Edge Router"

```
                     Enterprise WAN Edge                     Cloud Provider Edge
                  ┌───────────────────────┐                ┌───────────────────────┐
                  │      WAN-EDGE-1/2       │  eBGP (Public/  │   Cloud Router (VGW/  │
                  │      AS 65001            │  Private ASN)  │   ExpressRoute GW)     │
                  └───────────┬───────────┘◄───────────────►└───────────┬───────────┘
                              │        Dedicated Circuit (Direct        │
                              │        Connect/ExpressRoute/Interconnect)│
                        Advertise: On-Prem Prefix              Advertise: VPC/VNet CIDR
                        (เช่น 10.10.0.0/16 ผ่าน Route-map     (Cloud Provider ควบคุม
                         กรองเฉพาะ Summary ไม่ปล่อย /24         ว่าจะส่ง Prefix อะไรกลับมา
                         ทุกวงให้ Cloud เห็น)                    ตาม Setting ของ VPC/VNet)
```

### 735.2 ตารางเปรียบเทียบพารามิเตอร์ BGP ของแต่ละ Cloud Provider (ภาพรวมระดับ Preview)

| พารามิเตอร์ | AWS Direct Connect | Azure ExpressRoute | Google Cloud Interconnect |
|---|---|---|---|
| ASN ฝั่งลูกค้าที่รองรับ | Public หรือ Private ASN | Public หรือ Private ASN | Public หรือ Private ASN (16-bit/32-bit) |
| ASN ฝั่ง Cloud (ตัวอย่าง) | 64512 (Virtual Private Gateway) | 12076 | 16550 |
| จำนวน BGP Session ต่อ Circuit เพื่อ Redundancy | 2 (Primary/Secondary VIF) | 2 (Primary/Secondary ผ่าน /30 คู่) | 2 (คู่ Interconnect Attachment) |
| การเลือก Path หลักระหว่าง 2 Circuit | AS-Path Prepend หรือ BGP Community (`7224:7300` = Local Preference ต่ำ) | BGP Community (`12076:5030` ฯลฯ) หรือ Local-Pref | MED หรือ Local-Pref (ตาม Cloud Router Policy) |
| Community มาตรฐานที่ใช้สื่อสาร Policy | มี (Well-known ของ AWS) | มี (Well-known ของ Microsoft) | จำกัดกว่า (เน้น Local-Pref/MED) |

### 735.3 หลักการที่นำมาจาก Step ก่อนหน้าในการต่อ Cloud จริง

- **`network`/`aggregate-address`** (Part 29 Step 285) — ใช้ประกาศเฉพาะ Summary Prefix ของ
  On-Prem ให้ Cloud เห็น ไม่ปล่อยทุก Subnet ย่อยออกไปโดยไม่จำเป็น
- **Prefix-list ที่ `out`** (Part 29 Step 286) — ป้องกันไม่ให้ Private Subnet ที่ไม่เกี่ยวข้อง
  (เช่น VLAN Management) หลุดไปให้ Cloud Provider เห็น
- **Community-Based Policy** (Step 738 ของ Part นี้) — ใช้เลือกว่า Circuit คู่ไหนควรเป็น Primary
  สำหรับ Site ไหน แทนการเขียน Route-map แยกทุก Circuit

> **หมายเหตุ**: Part นี้ให้แค่ภาพรวมสถาปัตยกรรมเพื่อเชื่อมโยงว่า BGP ที่เรียนมาทั้งหมดใช้ต่อ
> Cloud ได้ตรงๆ — Config จริงแบบ Step-by-Step (การขอ Circuit, Configure VIF/Peering,
> Cloud onRamp for Multi-Cloud) จะสอนแบบเต็มรูปแบบใน **[Part 93 — Cloud Connectivity
> (AWS/Azure/GCP, Cloud onRamp)]** ซึ่งเป็น Part ที่เจาะจงเรื่อง Multi-Cloud Design โดยตรง

---

## Step 736 — Graceful Restart และ BGP NSF

### 736.1 ทบทวน NSF/Graceful Restart ที่เรียนมาแล้วสำหรับ IGP

หลักสูตรนี้สอนแนวคิด Non-Stop Forwarding ไปแล้ว 2 ครั้ง: **[Part 26 Step 256](part-026-advanced-ospf.md#step-256--ospf-graceful-restart--nsf-สำหรับ-high-availability-platform)**
(OSPF Graceful Restart ด้วย Grace-LSA) และ **[Part 35 Step 345](part-035-advanced-fhrp-campus-ha.md#step-345--nsf-และ-sso-supervisor-redundancy-ภายในเครื่องเดียว)**
(SSO + NSF ระดับ Chassis สำหรับทุก Routing Protocol) — Step นี้ปิดช่องว่างสุดท้าย: **BGP มี
กลไก Graceful Restart ของตัวเอง (RFC 4724)** ที่ทำงานคล้าย OSPF Grace-LSA แต่เป็น BGP Capability
ที่ต่อรองกันตอน OPEN Message โดยเฉพาะ

### 736.2 ปัญหาที่ BGP Graceful Restart แก้

```
เหตุการณ์: WAN-EDGE-1 (Platform Dual-RP) ทำ RP Switchover (Software Upgrade/Failover)

ไม่มี Graceful Restart:                        มี Graceful Restart:
  BGP Process Restart                             BGP Process Restart
        │                                                │
  Neighbor (ISP-RTR) ตรวจพบ TCP Session ขาด          Neighbor (ISP-RTR) เห็น Restart State Bit
        │ (Hold Timer หมด)                              ในแรก จึงไม่ Withdraw Route ทันที —
  ISP-RTR ประกาศ Withdraw Route ของ                  **คงค่า Forwarding State เดิมไว้ (Stale)**
  WAN-EDGE-1 ทั้งหมดทันที (Traffic Black-hole)        รอ Restarting Router (WAN-EDGE-1) กลับมา
        │                                                คืน Session ภายใน Restart Timer
  WAN-EDGE-1 กลับมา ต้อง Re-establish Session             │
  และ Re-advertise ทุก Prefix ใหม่ทั้งหมด             Session คืนสำเร็จ → Refresh Route จริง
  (Traffic หยุดตลอดช่วง Downtime)                     (Traffic ไม่หยุดเลยตลอดกระบวนการ)
```

### 736.3 องค์ประกอบสำคัญ: Restart Timer, Stale-Path Timer, และ F-bit

| องค์ประกอบ | ความหมาย |
|---|---|
| **Graceful Restart Capability** | ต่อรองกันใน OPEN Message — บอกว่า Router รองรับ GR หรือไม่ และ Restart Time สูงสุดที่ประกาศ |
| **Restart State Bit (R-bit)** | Router ที่กำลัง Restart ตั้ง Bit นี้ตอน Re-establish Session ใหม่ เพื่อบอก Neighbor ว่า "นี่คือการ Restart ไม่ใช่ Session ใหม่ปกติ" |
| **Forwarding State Bit (F-bit)** | ต่อ Address-Family — บอกว่า Router **ยังคง Forward Traffic ด้วย Route เดิมอยู่** แม้ Control Plane กำลัง Restart (คล้ายกับที่ SSO ทำในระดับ Chassis ตาม Part 35 Step 345.3) |
| **Restart Time** | เวลาสูงสุดที่ Neighbor จะรอให้ Restarting Router กลับมา Establish สำเร็จ ก่อนจะ Withdraw Route จริง (Default 120 วินาที) |
| **Stale-Path Timer** | เวลาที่ Neighbor เก็บ Route เดิมไว้เป็น "Stale" หลัง Session คืนแล้ว รอ Update จริงมาแทนที่ (ป้องกัน Route ค้างตลอดไปถ้า Restarting Router ไม่ส่ง Update Route นั้นมาอีก) |

### 736.4 Worked Config

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp graceful-restart
WAN-EDGE-1(config-router)# bgp graceful-restart restart-time 120
WAN-EDGE-1(config-router)# bgp graceful-restart stalepath-time 300
```

ต้อง Config ทั้งฝั่ง **Restarting Router** (WAN-EDGE-1 ที่จะทำ RP Switchover จริง — ต้องมี
Hardware Dual-RP/SSO รองรับตาม Part 35 Step 345.1) และ **Helper Router** (ISP-RTR, CORE-SW1/2,
WAN-EDGE-2 — ไม่จำเป็นต้องมี Dual-RP เอง แค่รองรับ Capability นี้ก็ช่วยได้ เหมือนหลักการ
NSF-aware ใน OSPF ตาม Step 256.2)

### 736.5 Verification

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 | section Graceful
  Graceful Restart Capability: advertised and received
    Remote Restart timer is 120 seconds
    Address families advertised by peer:
      ipv4 Unicast (was preserved)
```

`(was preserved)` คือสัญญาณว่า Session ล่าสุดฟื้นจาก Graceful Restart จริง (ไม่ใช่ Session ใหม่
ธรรมดา) — ถ้าไม่เคยเกิด Restart ค่านี้จะไม่ปรากฏ

> **ความเชื่อมโยงกับ NSF ระดับ Chassis**: BGP Graceful Restart คือ**ชั้นที่ 2** ของ Defense-in-
> Depth เดียวกันที่ Part 35 Step 347 วางไว้ — **SSO/NSF ระดับ Chassis** (Step 345) ป้องกันไม่ให้
> Data Plane หยุดขณะ RP สลับ ส่วน **BGP Graceful Restart** (Step นี้) ป้องกันไม่ให้ **Neighbor
> ข้างนอก** (ที่ไม่รู้เรื่อง SSO ภายในเครื่องเราเลย) เข้าใจผิดว่า Session ตายแล้วรีบ Withdraw
> Route — ทั้งสองต้องทำงานร่วมกันจึงได้ Non-Stop Forwarding ที่สมบูรณ์จริงข้าม Autonomous System

---

## Step 737 — BGP Security Hardening: RPKI Origin Validation

### 737.1 ปัญหาที่ RPKI แก้: BGP ไม่มีการยืนยันความเป็นเจ้าของ Prefix โดยธรรมชาติ

BGP ที่เรียนมาตั้งแต่ Part 29 **เชื่อทุก Prefix ที่ Neighbor ส่งมาโดย Default** (นอกจากจะกรองด้วย
Prefix-list/AS-Path Filter เอง ตาม Step 286) — ปัญหาคือ **ไม่มีกลไกใดใน BGP ดั้งเดิมที่ยืนยันได้
ว่า AS ที่ Originate Prefix นั้น "มีสิทธิ์" ประกาศ Prefix นั้นจริงหรือไม่** ทำให้เกิดเหตุการณ์
**Route/Prefix Hijacking** ได้ (AS ที่ไม่เกี่ยวข้องประกาศ Prefix ของคนอื่นออกไป โดยตั้งใจหรือ
พิมพ์ผิดก็ตาม) — เหตุการณ์ระดับโลกที่มีชื่อ เช่น AS7007 Incident (1997) และการ Hijack Prefix ของ
Google/Amazon หลายครั้งในอดีต ล้วนเกิดจากช่องโหว่นี้

### 737.2 RPKI คืออะไร

**RPKI (Resource Public Key Infrastructure, RFC 6480)** คือระบบที่ให้เจ้าของ Prefix ออก
**ROA (Route Origin Authorization)** ที่ลงลายเซ็นดิจิทัลรับรองว่า **"Prefix นี้ อนุญาตให้ AS
นี้เท่านั้น Originate ได้ (และยาวได้ไม่เกิน Max-Length เท่านี้)"** ผ่าน RIR (ARIN, RIPE NCC,
APNIC, ฯลฯ) — Router ที่รองรับ RPKI จะดึงข้อมูล ROA จาก **RPKI Validator/Cache Server** (ผ่าน
Protocol RTR — RPKI-to-Router, RFC 6810) แล้วนำมาเทียบกับ Prefix ที่ได้รับจริงผ่าน BGP

### 737.3 Validation State ทั้ง 3 แบบ

| State | ความหมาย | Action ที่แนะนำ |
|---|---|---|
| **Valid** | Prefix + Origin AS ตรงกับ ROA ที่มีอยู่ | รับตามปกติ |
| **Invalid** | มี ROA อยู่จริง แต่ Origin AS หรือ Prefix Length **ไม่ตรง** กับที่ ROA อนุญาต (สัญญาณของ Hijack) | **ควร Reject หรือลด Local-Preference ให้ต่ำที่สุด** |
| **NotFound** | ไม่มี ROA สำหรับ Prefix นี้เลย (ยังไม่ได้ทำ RPKI หรือ Prefix เก่าที่ไม่เคย Register) | ยอมรับตามปกติ (ส่วนใหญ่ยังเป็น NotFound เพราะ RPKI Adoption ทั่วโลกยังไม่ครบ 100%) |

### 737.4 Worked Config (ภาพรวม — Enterprise WAN Edge)

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp rpki server tcp 192.0.2.100 port 323 refresh-time 300
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# bgp bestpath origin-as use validity
WAN-EDGE-1(config-router-af)# exit-address-family
!
! Route-map ปฏิเสธ Prefix ที่ Invalid โดยเฉพาะจาก eBGP Neighbor (ISP-RTR)
WAN-EDGE-1(config)# route-map RPKI-FILTER-IN deny 10
WAN-EDGE-1(config-route-map)# match rpki invalid
WAN-EDGE-1(config-route-map)# exit
WAN-EDGE-1(config)# route-map RPKI-FILTER-IN permit 20
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 route-map RPKI-FILTER-IN in
```

- `bgp rpki server` ชี้ไปยัง RPKI Validator Cache (มักเป็น Software เช่น Routinator, rpki-client
  หรือ Cache ที่ ISP ให้บริการ) — ไม่ได้ Validate เองแบบ Cryptographic บน Router โดยตรง (ภาระ
  หนักเกินไปสำหรับ Router) แต่ดึงผลลัพธ์ที่ Validator คำนวณไว้แล้วผ่าน RTR Protocol
- `bgp bestpath origin-as use validity` — นำ RPKI State มาเป็นส่วนหนึ่งของ Best Path Algorithm
  (Invalid Path จะถูกจัดลำดับให้แพ้ Path อื่นเสมอ แม้ AS_PATH จะสั้นกว่าก็ตาม)
- `match rpki invalid` ใน Route-map — ใช้ Reject Prefix ที่ Invalid แบบเจาะจงได้ (ยืดหยุ่นกว่า
  การพึ่ง Best Path Algorithm เพียงอย่างเดียว)

### 737.5 ขอบเขตความรับผิดชอบของ Enterprise WAN Edge เทียบกับ ISP/Transit AS

| บทบาท | หน้าที่เกี่ยวกับ RPKI | ระดับความสำคัญสำหรับหลักสูตรนี้ |
|---|---|---|
| **ISP/Transit AS** | ต้อง Deploy RPKI Validation เต็มรูปแบบสำหรับ Prefix ทั้งหมดที่รับจาก Customer/Peer ทุกราย (ความรับผิดชอบหลักต่อความปลอดภัยของ Internet Routing ทั้งระบบ) | นอกขอบเขตหลักสูตรนี้ (เนื้อหาระดับ CCIE Service Provider) |
| **Enterprise WAN Edge (WAN-EDGE-1/2)** | (1) **ควรออก ROA รับรอง Prefix ของตัวเอง** (`203.0.113.0/24`) เพื่อให้ ISP อื่นทั่วโลก Validate เราได้ (2) **ควรเปิด Validation ขาเข้า** จาก ISP-RTR เป็น Defense-in-Depth ชั้นเพิ่มเติม แม้ ISP ส่วนใหญ่จะกรองให้อยู่แล้วก็ตาม | **ควรรู้จักและเข้าใจภาพรวม** (ตามที่ Step นี้สอน) — Enterprise ที่ Multihome จริงจัง (Dual-ISP แบบ Lab นี้) ควรพิจารณา Deploy จริงเพิ่มขึ้นเรื่อยๆ ตามเทรนด์ปัจจุบัน |

> **สรุปสำหรับข้อสอบ CCIE**: RPKI ไม่ใช่หัวข้อที่ต้อง Deploy ทุกรายละเอียดเชิงลึกแบบ SP แต่ต้อง
> **เข้าใจแนวคิด ROA/Validation State/Best Path Integration** และรู้ว่า Enterprise สมัยใหม่ที่ทำ
> Direct Peering กับ ISP หลายรายหรือ Cloud Provider (Step 735) ควรมี Awareness เรื่องนี้เพิ่มขึ้น
> เรื่อยๆ ไม่ใช่มองว่าเป็นเรื่องของ ISP อย่างเดียวเหมือนในอดีต

---

## Step 738 — Real-World BGP Policy Design Pattern: Community เป็นภาษา Policy

### 738.1 ปัญหาของ Route-Map แบบ Ad-Hoc ต่อ Neighbor

[Part 30 Step 298](part-030-bgp-advanced.md#step-298--bgp-communities) แนะนำ Community พื้นฐาน
(`65001:100` = จาก ISP1, `65001:200` = จาก ISP2) — วิธีนี้ใช้ได้ดีกับ 2 Neighbor แต่ถ้าองค์กร
ขยายเป็นหลาย Site/Region/Service-Tier (ตามที่ Part 73 ออกแบบไว้) การเขียน Route-map แยกทีละ
Neighbor แบบ Ad-Hoc จะเริ่มซับซ้อนจนดูแลไม่ไหว — **Community-Based Policy Framework** คือการ
ออกแบบ **ระบบเลข Community ที่มีโครงสร้างชัดเจนแบบภาษา** เพื่อให้ Route-map **ชุดเดียว** อ่าน
Community แล้วตัดสินใจ Policy ได้ทันที โดยไม่ต้องรู้จัก Prefix แต่ละตัวเป็นรายบุคคล

### 738.2 ออกแบบ Community Numbering Scheme (ตัวอย่างมาตรฐานระดับ Enterprise)

รูปแบบ `65001:XYYY` (ASN ขององค์กร ตามด้วยเลข 4 หลักที่แบ่งเป็นหมวดหมู่):

| ช่วง Community | หมวดหมู่ | ตัวอย่างในหลักสูตรนี้ |
|---|---|---|
| `65001:1xx` | **Site/Region ที่ Originate Route** | `65001:101` = HQ, `65001:102` = Site B, `65001:103` = Site C |
| `65001:2xx` | **Service-Tier ของ Prefix** | `65001:201` = Tier-1 Critical (ERP/Core App), `65001:202` = Tier-2 Standard, `65001:203` = Tier-3 Best-Effort |
| `65001:3xx` | **Exit-Point ที่เรียนรู้ Route เข้ามา (ทดแทน 65001:100/200 เดิม)** | `65001:301` = เรียนจาก ISP1 (WAN-EDGE-1), `65001:302` = เรียนจาก ISP2 (WAN-EDGE-2) |
| `65001:9xx` | **Action Community (สั่งงาน RR/WAN Edge โดยตรง)** | `65001:901` = Prepend AS 1 ครั้งก่อนส่งออก ISP, `65001:902` = Prepend 2 ครั้ง, `65001:999` = Blackhole (ส่งไป Null0) |

### 738.3 Route-Map "Policy Engine" ที่อ่าน Community แล้วตัดสินใจอัตโนมัติ

**ขั้นที่ 1 — Tag Community ตอน Originate (ที่ CORE-SW1, Route จาก Site HQ):**

```
CORE-SW1(config)# route-map TAG-ORIGIN permit 10
CORE-SW1(config-route-map)# set community 65001:101 65001:201 additive
CORE-SW1(config-route-map)# exit
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# neighbor 1.1.1.11 route-map TAG-ORIGIN in
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST send-community both
```

**ขั้นที่ 2 — Policy Engine ที่ WAN-EDGE (อ่าน Community แล้วตัดสินใจ Prepend/Reject โดยไม่ต้องรู้
ว่า Prefix คืออะไร):**

```
WAN-EDGE-1(config)# ip community-list standard CL-TIER1 permit 65001:201
WAN-EDGE-1(config)# ip community-list standard CL-BLACKHOLE permit 65001:999
!
WAN-EDGE-1(config)# route-map POLICY-ENGINE-OUT deny 5
WAN-EDGE-1(config-route-map)# match community CL-BLACKHOLE
WAN-EDGE-1(config-route-map)# exit
WAN-EDGE-1(config)# route-map POLICY-ENGINE-OUT permit 10
WAN-EDGE-1(config-route-map)# match community CL-TIER1
WAN-EDGE-1(config-route-map)# set as-path prepend 65001
WAN-EDGE-1(config-route-map)# exit
WAN-EDGE-1(config)# route-map POLICY-ENGINE-OUT permit 20
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 route-map POLICY-ENGINE-OUT out
```

> **หลักการสำคัญของ Framework นี้**: เมื่อ Site B/Site C ถูกเพิ่มเข้ามาในอนาคต (Part 73)
> ไม่ต้องแก้ Route-map ที่ WAN-EDGE เลยแม้แต่บรรทัดเดียว — แค่ Tag Community ที่จุด Originate
> ของ Site ใหม่ให้ตรง Scheme เดิม (`65001:1xx` ตาม Site, `65001:2xx` ตาม Tier) Policy ทั้งหมด
> จะ "ไหล" ไปทำงานถูกต้องอัตโนมัติ — นี่คือความหมายของ "Community เป็นภาษา Policy": เขียน
> Policy Engine ครั้งเดียว ให้ Scale ครอบคลุมทุก Site/Prefix ในอนาคตโดยไม่ต้องแก้โค้ดซ้ำ

### 738.4 ตารางสรุป Framework เทียบ Ad-Hoc Route-map แบบเดิม

| ประเด็น | Ad-Hoc Route-map ต่อ Neighbor (Part 30) | Community Policy Framework (Step นี้) |
|---|---|---|
| จำนวน Route-map ที่ต้องดูแลเมื่อมี 10 Site | 10+ (แยกตาม Neighbor/Site) | คงที่ที่ 1-2 Route-map (Policy Engine) |
| เพิ่ม Site ใหม่ต้องแก้ตรงไหน | ทุกจุดที่มี Policy เกี่ยวข้อง (เสี่ยงพลาด) | จุด Originate ของ Site ใหม่เท่านั้น |
| Audit ว่า Prefix ไหนเป็น Policy อะไร | ต้องไล่ดู Route-map ทีละอัน | `show bgp ipv4 unicast community 65001:201` เห็นทันที |
| เหมาะกับ Design ขนาดใหญ่ (Part 73) หรือไม่ | ไม่เหมาะ (Scale ไม่ได้) | **เหมาะมาก — มาตรฐานของ Enterprise ขนาดใหญ่ทุกราย** |

---

## Step 739 — BGP Troubleshooting ที่ Scale: เทคนิค Filter ตาราง BGP ขนาดใหญ่

### 739.1 ปัญหา: `show ip bgp` เพียวๆ ใช้งานไม่ได้จริงเมื่อมีหลักพัน/หมื่น Prefix

เมื่อตาราง BGP มี Prefix จำนวนมาก (Full Table จาก Multi-Cloud/Multi-ISP ตาม Step 735, หรือ
Enterprise ขนาดใหญ่ตาม Part 73 ที่มี Prefix จากหลาย Region) การพิมพ์ `show ip bgp` เปล่าๆ แล้ว
เลื่อนหาด้วยตาเป็นไปไม่ได้ในทางปฏิบัติ — ต้องใช้เทคนิค Filter ที่ตรงจุดทันที

### 739.2 ตารางคำสั่ง Filter หลักที่ต้องใช้คล่องระดับ CCIE

| คำสั่ง | ใช้ทำอะไร | ตัวอย่าง |
|---|---|---|
| `show ip bgp regexp <regex>` | Filter ตาม AS_PATH ด้วย Regular Expression — เร็วที่สุดเมื่อรู้ AS ต้นทาง/ปลายทางที่สงสัย | `show ip bgp regexp ^65000$` (Prefix ที่ Origin คือ AS 65000 เท่านั้น) |
| `show bgp ipv4 unicast community <community>` | Filter ตาม Community — ใช้คู่กับ Framework ใน Step 738 ได้ทันที | `show bgp ipv4 unicast community 65001:201` (ดู Tier-1 Prefix ทั้งหมด) |
| `\| section <pattern>` | ตัดผลลัพธ์ทั้งบล็อกที่เริ่มด้วย Pattern (มีประโยชน์มากกับ `show ip bgp neighbors` ที่ผลยาวมาก) | `show run \| section router bgp` |
| `\| include <pattern>` | แสดงเฉพาะบรรทัดที่ Match (เร็วที่สุด แต่ไม่มี Context บรรทัดรอบข้าง) | `show ip bgp summary \| include Idle\|Active` |
| `\| exclude <pattern>` | ซ่อนบรรทัดที่ Match — มีประโยชน์มากในการหา "ตัวที่ผิดปกติ" จาก List ที่ปกติเหมือนกันหมด | `show ip bgp summary \| exclude 65001` (หา Neighbor AS อื่นที่ไม่ใช่ iBGP ปกติ) |
| `\| begin <pattern>` | ตัดผลลัพธ์ให้เริ่มแสดงจากบรรทัดที่ Match เป็นต้นไป | `show ip bgp \| begin 10.0.0.0` |
| `show ip bgp <prefix> longer-prefixes` | หา Prefix ทั้งหมดที่เป็น Subnet ย่อยของ Prefix นี้ | `show ip bgp 10.0.0.0 255.0.0.0 longer-prefixes` |

### 739.3 Worked Example — ไล่หา Session ที่มีปัญหาในกลุ่ม RR-Client จำนวนมาก

```
CORE-SW1# show ip bgp summary | exclude Established
BGP router identifier 1.1.1.1, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.13        4 65001       0       0        1    0    0 never    Active
```

- Filter `exclude Established` ทำให้เห็น**เฉพาะ Neighbor ที่มีปัญหา** ทันทีจากกลุ่ม Neighbor
  ทั้งหมด (ในตัวอย่างนี้คือ DIST-SW3 ที่ยังไม่ Establish) — ไม่ต้องไล่สายตาหา State ทีละบรรทัด
  เมื่อมี Neighbor เป็นสิบตัว

```
WAN-EDGE-1# show ip bgp regexp _65010$
   Network          Next Hop            Metric LocPrf Weight Path
*> 172.20.1.0/24    203.0.113.1              0             0 65000 65010 i
```

- `_65010$` ("underscore" หมายถึงขอบเขตของ Token ใน Regex ของ Cisco IOS — ช่องว่าง/จุดเริ่ม/
  จุดสิ้นสุด) หา Prefix ที่ Origin AS คือ 65010 โดยไม่สนใจว่าอยู่ Path ลำดับที่เท่าไหร่ก่อนหน้า —
  มีประโยชน์มากเมื่อต้องหาว่า "Prefix จาก Site/Partner รายนี้มีกี่ตัว" ในตาราง BGP ขนาดใหญ่

```
CORE-SW1# show bgp ipv4 unicast community 65001:999
   Network          Next Hop            Metric LocPrf Weight Path
*> 192.0.2.128/25   0.0.0.0                  0         32768 i
```

- ตรวจสอบ Blackhole Community (`65001:999` ตาม Step 738.2) — เห็นทันทีว่ามี Prefix ไหนถูก
  Blackhole อยู่บ้าง โดยไม่ต้องพึ่งความจำว่า Prefix ไหน Tag ไว้เมื่อไหร่

> **เคล็ดลับสำหรับข้อสอบ CCIE Lab (Time-boxed)**: ลำดับการ Troubleshoot ที่เร็วที่สุดเสมอคือ
> **(1)** `show ip bgp summary | exclude Established` หา Session ที่ผิดปกติก่อน **(2)** ถ้า
> Session ปกติหมดแต่ Prefix ไม่มาตามคาด ใช้ `show bgp ipv4 unicast community <tag>` หรือ
> `show ip bgp regexp` เจาะตาม Design (Step 738) **(3)** ค่อยไล่ดู Route-map/Prefix-list ที่
> Neighbor นั้นด้วย `show run | section neighbor <ip>` เป็นขั้นสุดท้าย — เรียงลำดับผิดคือสาเหตุ
> อันดับหนึ่งที่ทำให้เสียเวลาในห้องสอบ Lab ที่จับเวลา

---

## Step 740 — Lab เต็มรูปแบบ: Scaled Community-Based Policy Framework พร้อม Add-Path และ PIC

### 740.1 เป้าหมายของ Lab

รีดีไซน์ **Dual-ISP eBGP + iBGP Full-Mesh จาก Part 30 Step 300** ให้กลายเป็นโครงสร้างที่รวมทุก
เทคนิคของ Part นี้เข้าด้วยกันที่ขอบเขต **WAN-EDGE ↔ CORE**:

- **Hierarchical RR** (Step 731): WAN-EDGE-1/2 = Tier 1 RR, CORE-SW1/2 = Tier 2 RR
- **Add-Path + PIC** (Step 732-733) ที่ขอบเขต WAN-EDGE ↔ CORE เพื่อ Failover เร็วและมี ECMP
- **Community-Based Policy Framework** (Step 738) แทน Ad-Hoc Route-map เดิม
- **Graceful Restart** (Step 736) เปิดทุก Session ในกลุ่มนี้

```
                         ┌─────────────┐        ┌─────────────┐
                         │  ISP-RTR      │        │  ISP-RTR     │  (เส้นทางเดิมจาก Part 29/30)
                         │  AS 65000     │        │  AS 65000    │
                         └──────┬───────┘        └──────┬───────┘
                    203.0.113.0/30 │                    │ 203.0.113.4/30
                         ┌────────┴───────┐    ┌────────┴───────┐
                         │  WAN-EDGE-1     │────│  WAN-EDGE-2     │  Tier 1 RR
                         │  Cluster .100    │iBGP│  Cluster .100   │  (Full-Mesh คู่เดียว)
                         └───────┬────────┘    └────────┬───────┘
                        Add-Path + PIC Edge          Add-Path + PIC Edge
                         ┌───────┴────────┐    ┌────────┴───────┐
                         │  CORE-SW1        │    │  CORE-SW2       │  Tier 2 RR
                         │  Cluster .101     │    │  Cluster .101    │  (RR-Client ของ Tier1
                         └──┬───────────┬──┘    └──┬──────────┬───┘   + RR ของ Tier2)
                       RR-Client    RR-Client   RR-Client   RR-Client
                       DIST-SW1     DIST-SW2    DIST-SW3    DIST-SW4
```

### 740.2 Running-Config เต็มรูปแบบ — WAN-EDGE-1

```
hostname WAN-EDGE-1
!
interface GigabitEthernet0/0/0
 description ** eBGP Link to ISP-RTR (Primary Internet, AS 65000) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 1.1.1.21 255.255.255.255
!
ip route 203.0.113.0 255.255.255.0 Null0 254
!
route-map TAG-EXIT-ISP1 permit 10
 set community 65001:301 additive
!
ip community-list standard CL-BLACKHOLE permit 65001:999
route-map POLICY-ENGINE-OUT deny 5
 match community CL-BLACKHOLE
route-map POLICY-ENGINE-OUT permit 10
!
router bgp 65001
 bgp router-id 1.1.1.21
 bgp cluster-id 1.1.1.100
 bgp graceful-restart
 bgp graceful-restart restart-time 120
 bgp graceful-restart stalepath-time 300
 neighbor 203.0.113.1 remote-as 65000
 neighbor 203.0.113.1 description ** eBGP to ISP-RTR (Primary) **
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 update-source Loopback0
 neighbor 1.1.1.22 description ** iBGP Tier1 RR Peer: WAN-EDGE-2 **
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.1 description ** iBGP Tier1 RR-Client: CORE-SW1 **
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 update-source Loopback0
 neighbor 1.1.1.2 description ** iBGP Tier1 RR-Client: CORE-SW2 **
 !
 address-family ipv4 unicast
  network 203.0.113.0 mask 255.255.255.0
  bgp additional-paths select all
  bgp additional-paths install
  neighbor 203.0.113.1 route-map TAG-EXIT-ISP1 in
  neighbor 203.0.113.1 route-map POLICY-ENGINE-OUT out
  neighbor 1.1.1.22 activate
  neighbor 1.1.1.22 send-community both
  neighbor 1.1.1.22 additional-paths send
  neighbor 1.1.1.22 additional-paths receive
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 next-hop-self
  neighbor 1.1.1.1 route-reflector-client
  neighbor 1.1.1.1 send-community both
  neighbor 1.1.1.1 additional-paths send
  neighbor 1.1.1.1 additional-paths receive
  neighbor 1.1.1.1 advertise additional-paths all
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 next-hop-self
  neighbor 1.1.1.2 route-reflector-client
  neighbor 1.1.1.2 send-community both
  neighbor 1.1.1.2 additional-paths send
  neighbor 1.1.1.2 additional-paths receive
  neighbor 1.1.1.2 advertise additional-paths all
 exit-address-family
```

**WAN-EDGE-2 คือกระจกเงา** ของ Config นี้ (สลับ `203.0.113.1`→`203.0.113.5`, Community
`65001:302` แทน `65001:301`, และ Neighbor iBGP สลับเป็น `1.1.1.21`)

### 740.3 Running-Config เต็มรูปแบบ — CORE-SW1 (Tier 2 RR + Tier 1 RR-Client)

```
hostname CORE-SW1
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
route-map TAG-TIER1-APP permit 10
 match ip address prefix-list PL-TIER1-APP
 set community 65001:101 65001:201 additive
route-map TAG-TIER1-APP permit 20
 set community 65001:101 65001:202 additive
!
ip prefix-list PL-TIER1-APP seq 5 permit 10.10.30.0/24
!
router bgp 65001
 bgp router-id 1.1.1.1
 bgp cluster-id 1.1.1.101
 bgp graceful-restart
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.21 description ** iBGP Tier1 RR: WAN-EDGE-1 **
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 update-source Loopback0
 neighbor 1.1.1.22 description ** iBGP Tier1 RR: WAN-EDGE-2 **
 neighbor RR-CLIENT-DIST peer-group
 neighbor RR-CLIENT-DIST remote-as 65001
 neighbor RR-CLIENT-DIST update-source Loopback0
 neighbor 1.1.1.11 peer-group RR-CLIENT-DIST
 neighbor 1.1.1.11 description ** iBGP Tier2 RR-Client: DIST-SW1 **
 neighbor 1.1.1.12 peer-group RR-CLIENT-DIST
 neighbor 1.1.1.12 description ** iBGP Tier2 RR-Client: DIST-SW2 **
 !
 address-family ipv4 unicast
  redistribute connected route-map TAG-TIER1-APP
  bgp additional-paths select all
  bgp additional-paths install
  maximum-paths ibgp 2
  neighbor 1.1.1.21 activate
  neighbor 1.1.1.21 send-community both
  neighbor 1.1.1.21 additional-paths receive
  neighbor 1.1.1.22 activate
  neighbor 1.1.1.22 send-community both
  neighbor 1.1.1.22 additional-paths receive
  neighbor RR-CLIENT-DIST activate
  neighbor RR-CLIENT-DIST next-hop-self
  neighbor RR-CLIENT-DIST route-reflector-client
  neighbor RR-CLIENT-DIST send-community both
  neighbor RR-CLIENT-DIST additional-paths receive
  neighbor 1.1.1.11 activate
  neighbor 1.1.1.12 activate
 exit-address-family
```

> **หมายเหตุ**: `redistribute connected route-map TAG-TIER1-APP` ในตัวอย่างนี้ใช้เพื่อสาธิตการ
> Tag Community ตาม Service-Tier ที่จุด Originate เท่านั้น (ตาม Step 738.3) — ใน Production
> จริงต้องระวังตามคำเตือนของ [Part 29 Step 285](part-029-bgp-fundamentals.md#step-285--การประกาศ-route-เข้า-bgp-network-vs-redistribute-vs-aggregate-address)
> เรื่องความเสี่ยงของ `redistribute` เสมอ — ควบคุมด้วย Prefix-list ให้เจาะจงที่สุด (ในที่นี้คือ
> `PL-TIER1-APP` ที่จำกัดเฉพาะ `10.10.30.0/24`, VLAN 30 SERVERS ตาม
> [00-ip-address-plan.md](00-ip-address-plan.md))

### 740.4 Verification ครบทั้ง 4 เทคนิคหลัก

**(1) RR Hierarchy ทำงานถูกต้อง — DIST-SW1 เห็น Route จาก ISP ทั้งสองผ่าน CORE-SW1 เพียงจุดเดียว:**

```
DIST-SW1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.1         4 65001      45      42       20    0    0 00:15:02        2
```

**(2) Add-Path ทำงาน — CORE-SW1 เห็น 2 Path พร้อมกันสำหรับ Default Route:**

```
CORE-SW1# show bgp ipv4 unicast 0.0.0.0/0 | begin Paths
Paths: (2 available, best #1, table default)
  Path #1: ... 1.1.1.21 ... best
  Path #2: ... 1.1.1.22 ... backup
```

**(3) PIC ติดตั้ง Backup ใน CEF แล้ว — พร้อม Fast Reroute โดยไม่รอ Control Plane:**

```
CORE-SW1# show ip cef 0.0.0.0/0 detail | include repair
    nexthop 1.1.1.22 Port-channel1 (repair)
```

**(4) Community Policy Framework ทำงาน — ตรวจสอบ Prefix ตาม Tier ได้ทันทีโดยไม่ต้องรู้จัก
Prefix ล่วงหน้า:**

```
CORE-SW1# show bgp ipv4 unicast community 65001:201
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.30.0/24    0.0.0.0                  0         32768 i
```

**(5) ทดสอบ Failover จริง — Shutdown eBGP ของ WAN-EDGE-1 แล้ววัดว่า Traffic ที่ CORE-SW1 สลับ
ทันทีโดยไม่ต้องรอ BGP Reconverge (เทียบผลกับ Part 30 Step 300 ที่ยังไม่มี PIC):**

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 shutdown

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.22, from 1.1.1.22, 00:00:00 ago      <- สลับทันที (Backup ติดตั้งไว้แล้วล่วงหน้า
                                                    ผ่าน PIC ไม่ใช่คำนวณใหม่หลัง Shutdown)
```

Lab นี้พิสูจน์ครบทั้ง 4 เสาหลักของ Part 74: **RR ที่ Scale ได้จริงแบบ Hierarchical, Add-Path ที่
คืน ECMP/Visibility ให้ Client ของ RR, PIC ที่ทำ Failover แบบ Prefix-Independent, และ Community
Framework ที่ทำให้ Policy Scale ไปกับจำนวน Site ได้โดยไม่ต้องแก้ Route-map ซ้ำ** — ครบตามที่
ข้อสอบ CCIE Enterprise Infrastructure Lab คาดหวังจาก WAN Edge ระดับ Production จริง

---

## แบบฝึกหัดทวนความเข้าใจ Part 74

1. เพราะเหตุใด Redundant RR Pair (เช่น WAN-EDGE-1 + WAN-EDGE-2) ต้องใช้ **Cluster-ID เดียวกัน**
   แต่ Regional RR Cluster คนละ Site (เช่น Site HQ กับ Site B) ต้องใช้ **Cluster-ID ต่างกัน**?
2. `bgp additional-paths select all` กับ `bgp additional-paths install` ต่างกันอย่างไร และทำไม
   ต้องมีทั้งสองคำสั่งจึงจะได้ประโยชน์ของ PIC Edge เต็มรูปแบบ?
3. BGP Unnumbered (ผ่าน IPv6 Link-Local) เหมาะกับ Topology แบบไหน และทำไมใช้กับ Multi-access
   Segment ไม่ได้?
4. ในการออกแบบ Community Policy Framework (Step 738) เมื่อมี Site ใหม่เพิ่มเข้ามา ทำไมไม่ต้อง
   แก้ Route-map ที่ WAN-EDGE เลย?
5. เรียงลำดับขั้นตอน Troubleshooting BGP ที่มี Neighbor จำนวนมากให้ถูกต้องตามหลัก Step 739.3
   (จาก Session-level ไปจนถึง Route-map-level)

**เฉลย:**

1. Cluster-ID เดียวกันในคู่ Redundant บอก BGP ว่า RR ทั้งสองตัวทำหน้าที่ซ้ำกัน (กันลูปแบบ
   Suppress ซ้ำอย่างถูกต้อง ไม่บวมตาราง BGP เป็น 2 เท่า) ส่วน Cluster-ID ต่างกันข้าม Site ป้องกัน
   ไม่ให้ BGP เข้าใจผิดว่าเป็น Cluster เดียวกันแล้ว False-Suppress Route ที่ควรกระจายไปยังทุก
   Region จริง
2. `select all` สั่งให้ Router **คำนวณ** ว่า Path ไหนบ้างมีคุณสมบัติเป็น Additional Path (เก็บไว้
   ในตาราง BGP) ส่วน `install` สั่งให้ **ฝัง Path เหล่านั้นเข้า RIB/FIB จริง** — ถ้ามีแค่ `select`
   อย่างเดียว Path สำรองจะอยู่แค่ในตาราง BGP ใช้ทำ Fast Reroute ผ่าน CEF ไม่ได้ ต้องมีทั้งคู่จึงจะ
   ได้ PIC Edge ที่สลับ Forwarding ได้จริงโดยไม่รอ Control Plane
3. เหมาะกับ Link แบบ **Point-to-Point แท้จริง** เท่านั้น (Physical P2P, Tunnel, Sub-interface
   P2P) เพราะคำสั่ง `neighbor <interface> remote-as` สมมติว่ามี Neighbor เดียวอยู่ปลายสาย — บน
   Multi-access Segment (เช่น Ethernet Switch ที่มีหลาย Router ต่ออยู่) ไม่มีทางรู้ว่า "Neighbor"
   ที่ตั้งใจจะ Peer คือตัวไหนจากหลาย Router ที่แชร์ Segment เดียวกัน
4. เพราะ Route-map ที่ WAN-EDGE (Policy Engine) อ่าน **Community** เท่านั้น ไม่ได้ผูกกับ Prefix
   หรือ Neighbor เฉพาะเจาะจง — ตราบใดที่ Site ใหม่ Tag Community ตาม Scheme เดิม (`65001:1xx`
   ตาม Site, `65001:2xx` ตาม Tier) ที่จุด Originate ของตัวเอง Policy เดิมจะ "จับ" Prefix ใหม่
   ได้อัตโนมัติโดยไม่ต้องแก้ Route-map ที่ WAN-EDGE เลย
5. (1) `show ip bgp summary | exclude Established` ตรวจ Session-level ก่อนเสมอ (2) ถ้า Session
   ปกติแต่ Prefix ขาด ใช้ `show bgp ipv4 unicast community <tag>` หรือ `show ip bgp regexp` เจาะ
   ตาม Design (3) ไล่ดู Route-map/Prefix-list ที่ Neighbor นั้นด้วย `show run | section neighbor`
   เป็นขั้นสุดท้าย

---

## สรุป Part 74

Part นี้เจาะลึก BGP เกินกว่าที่ Part 29-30 สอนไว้ เพื่อปิดช่องว่างสู่ระดับ **CCIE Enterprise
Infrastructure** อย่างครบถ้วน: **Route Reflector แบบ Hierarchical/Multi-Cluster** ที่ใช้ได้จริง
กับองค์กรขนาดใหญ่ (ต่างจาก Single-Cluster Conceptual ของ Part 30), **BGP Add-Path** ที่คืน
ความสามารถ ECMP/Visibility ให้ Client ของ RR, **BGP PIC** ที่ทำให้ Failover ไม่ต้องรอ Control
Plane เดินตาม Prefix ทีละตัว, **BGP Unnumbered/Peer-Group** ที่ลดภาระ Config ที่ WAN Edge ขนาด
ใหญ่, ภาพรวม **BGP สำหรับ Multi-Cloud** ที่จะเจาะลึกเต็มรูปแบบใน Part 93, **Graceful Restart/NSF
เฉพาะทางของ BGP** ที่เชื่อมกับ NSF ระดับ Chassis จาก Part 26/35, **RPKI Origin Validation** เพื่อ
ป้องกัน Route Hijacking, **Community-Based Policy Framework** ที่ทำให้ Policy Scale ไปกับจำนวน
Site ได้โดยไม่ต้องแก้ Route-map ซ้ำ, และปิดท้ายด้วย **เทคนิค Troubleshoot ตาราง BGP ขนาดใหญ่**
ที่จำเป็นสำหรับข้อสอบ Lab ที่จับเวลา — Lab เต็มรูปแบบใน Step 740 พิสูจน์ว่าทุกเทคนิครวมกันทำงาน
จริงบน Topology หลักของหลักสูตรได้ ✅

**สิ่งที่ Part นี้ยังไม่ครอบคลุม**: ทุก Step เน้น **BGP สำหรับ Unicast IPv4 Routing** เท่านั้น —
Enterprise ขนาดใหญ่ที่ต้องกระจาย **Multicast Traffic** ข้าม Site จำนวนมาก (เช่น Video
Conferencing, Financial Market Data Feed, IPTV ภายในองค์กร) ต้องการ Protocol เพิ่มเติมที่ทำงาน
คู่กับ Unicast Routing ที่มีอยู่แล้ว — [Part 41](part-041-multicast-fundamentals.md) และ
[Part 42](part-042-multicast-advanced.md) สอนพื้นฐาน PIM/IGMP ไปแล้วในระดับ CCNP แต่ยังไม่ได้
สอนการออกแบบ Multicast ข้าม Site หลาย Region ระดับ CCIE — **Part 75 จะกลับไปเจาะลึก Multicast
อีกครั้งในบริบท Enterprise ขนาดใหญ่** ที่ใช้ BGP/MPLS เป็นโครงสร้างพื้นฐานร่วมกับที่ Part นี้
วางไว้

**ไปต่อ:** [Part 75 — Advanced Multicast for Enterprise →](part-075-advanced-multicast-enterprise.md)
