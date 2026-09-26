# Part 76 — SD-WAN Advanced Design (Cisco Viptela Deep Dive)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 751–760 จาก 1000**

> ต่อจาก [Part 75 — Advanced Multicast Enterprise](part-075-advanced-multicast-enterprise.md) ที่
> เจาะลึก Multicast ระดับ Enterprise (MSDP, Anycast RP, Multicast VPN) แล้ว Part นี้จะ **กลับมาที่
> SD-WAN** ตามที่ [Part 39 — SD-WAN Fundamentals](part-039-sd-wan-fundamentals.md) ได้ "ค้างไว้"
> ตั้งใจไว้ในบทสรุป (Step 388-389) ว่า **Multi-Region Topology, Advanced Application-Aware Policy,
> Cloud OnRamp เชิงลึก, Security Integration, และ Migration ระดับองค์กรขนาดใหญ่** จะสอนเต็มรูปแบบ
> ที่ Part 76 — Part นี้คือ Part นั้น
>
> Part 39 สอนสถาปัตยกรรม 4 Planes, OMP, TLOC, ZTP, AAR พื้นฐาน และ VPN Segment ในระดับ HQ + 1
> Branch เท่านั้น (WAN-EDGE-1/2 + BRANCH-RTR) — Part นี้จะขยาย Design เดียวกันขึ้นไปสู่ระดับ
> **หลายร้อย Site** ซึ่งเป็นระดับที่ข้อสอบ **CCIE Enterprise Infrastructure** และงาน Design จริง
> ขององค์กร Retail/Bank/Enterprise ขนาดใหญ่ต้องเจอ

> **หมายเหตุสำคัญเรื่อง Lab**: เช่นเดียวกับ Part 39 — Part นี้เป็น **Conceptual + Design-Level +
> Real CLI/Policy Syntax Reference** เพราะ Fabric ขนาดหลายร้อย Site ไม่สามารถจำลองได้จริงใน CLI
> Lab พื้นฐานของหลักสูตรนี้ ทุก Policy/Command ที่แสดงเป็น Syntax จริงของ Cisco Catalyst SD-WAN
> (Viptela) เพื่อให้นำไปทดสอบบน CML/dCloud/SD-WAN Cloud-hosted Sandbox ได้ทันที

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 751 | SD-WAN Fabric Design ระดับ Enterprise Scale — Hub-Spoke vs Full-Mesh vs Regional-Mesh |
| 752 | Multi-Region Fabric (MRF) — Regional Hub และ Global Core Region |
| 753 | Advanced Application-Aware Routing — SLA Class หลายชั้น + Fallback Chain ข้าม 3 Transport |
| 754 | High Availability Controller Design — vManage/vSmart/vBond Cluster ระดับ Enterprise |
| 755 | Cloud OnRamp for SaaS เชิงลึก — เลือก Local Breakout ที่ดีที่สุดต่อแอปโดยอัตโนมัติ |
| 756 | Cloud OnRamp for IaaS — ขยาย Fabric เข้าสู่ AWS/Azure/GCP |
| 757 | SD-WAN Security Integration — Enterprise Firewall, IPS, URL Filtering, Umbrella |
| 758 | Segmentation at Scale — ออกแบบ VPN Segment แบบ Multi-Tenant ทั้ง Fabric |
| 759 | กลยุทธ์ Migration จาก MPLS/Traditional WAN องค์กรขนาดใหญ่ สู่ SD-WAN |
| 760 | Lab เต็มรูปแบบ (Design-Level): SD-WAN Design Document สำหรับ Retail 200 สาขา |

---

## Step 751 — SD-WAN Fabric Design ระดับ Enterprise Scale: Hub-Spoke vs Full-Mesh vs Regional-Mesh

### ทำไม Topology Design ของ SD-WAN ต่างจาก DMVPN ([Part 52-53](part-052-dmvpn-fundamentals.md))

[Part 52-53](part-052-dmvpn-fundamentals.md) สอนไว้ว่า DMVPN Phase 3 ทำให้ Spoke สร้าง
Dynamic Tunnel ถึงกันเองได้โดยไม่ต้องผ่าน Hub เสมอ (Spoke-to-Spoke) — แต่ **Control Plane ของ
DMVPN ยังพึ่ง NHRP ซึ่งเป็นกลไกที่ค่อนข้าง "เบา"** และ Hub ยังต้องเป็นจุดที่ Spoke ทุกตัว Register
เข้ามาก่อนเสมอ (Multipoint GRE บน Hub ตัวเดียวรับ Load ทุก Spoke)

**SD-WAN ต่างออกไปโดยพื้นฐาน**: **Control Plane (vSmart/OMP) แยกออกจาก Data Plane (cEdge
IPsec Tunnel) อย่างสมบูรณ์** — vSmart ไม่ได้เป็นจุดที่ Traffic วิ่งผ่านเลย มันแค่ "แจก TLOC ให้
cEdge รู้จักกัน" แล้ว **cEdge สร้าง Tunnel ตรงถึงกันเอง** ดังนั้น Full-Mesh ของ SD-WAN ไม่มี
Hub เป็น Bottleneck ของ Data Plane แบบ DMVPN Hub เลย — นี่คือเหตุผลที่ Full-Mesh กลายเป็น
Topology ที่ "เป็นไปได้จริง" ที่ Scale ใหญ่กว่า DMVPN มาก

### ข้อจำกัดจริงที่ทำให้ Full-Mesh เต็มรูปแบบยังมีเพดาน

แม้ Control Plane จะรองรับได้ แต่ **Data Plane ยังมีข้อจำกัดที่ตัว cEdge เอง** — จำนวน IPsec
Tunnel ต่ออุปกรณ์ (Tunnel Count per Device) ที่ Platform รองรับได้จริงมีเพดาน (ขึ้นกับรุ่น CPU/
Crypto Engine) และจำนวน Tunnel จะโตแบบ **N×(N-1)** ต่อ cEdge หนึ่งตัวถ้าเป็น Full-Mesh เต็มรูป
(นับทั้งสองทิศทางของ Tunnel ที่ต้อง Maintain State)

| จำนวน Site (N) | Full-Mesh: Tunnel รวมทั้ง Fabric N×(N-1)/2 | Tunnel ต่อ cEdge หนึ่งตัว (N-1) |
|---|---|---|
| 10 | 45 | 9 |
| 50 | 1,225 | 49 |
| 200 | 19,900 | 199 |
| 500 | 124,750 | 499 |

จาก 200 Site ขึ้นไป **cEdge ที่ Branch เล็กๆ (เช่น ISR1000 Platform ต่ำ) ต้อง Maintain Tunnel
เกือบ 200 เส้นพร้อมกัน** ซึ่งเกินความจำเป็นจริง (Branch ค้าปลีกแทบไม่มี Traffic คุยกับ Branch อื่น
โดยตรงเลย ส่วนใหญ่คุยกับ Data Center/Cloud เท่านั้น) — จึงต้องมี Topology ที่ **ควบคุมได้ (Policy-
Controlled Mesh)** ไม่ใช่ Full-Mesh อัตโนมัติแบบไม่มีข้อจำกัดเสมอไป

### 3 รูปแบบ Topology หลักที่ต้องเลือกตาม Design

```
1) HUB-AND-SPOKE                 2) FULL-MESH                    3) REGIONAL-MESH
                                                                  (แนะนำสำหรับ Enterprise Scale)

      [HUB]                      [S1]══[S2]                  ┌── Region North ──┐
     /  |  \                      ║  ╲  ╱  ║                 │ [S1]≡[S2]≡[Hub-N] │
  [S1] [S2] [S3]                  ║   ╳    ║                 └─────────┬────────┘
                                  [S3]══[S4]                           │ (เชื่อมข้าม Region
  Spoke↔Spoke ต้องผ่าน Hub                                             │  ผ่าน Hub เท่านั้น)
  เท่านั้น (Policy บังคับ)        ทุก Site ↔ ทุก Site               ┌─────────┴────────┐
                                  (เหมาะ Site จำนวนน้อย             │ [S3]≡[S4]≡[Hub-S] │
                                  ที่ต้องคุยกันเองบ่อย)              └── Region South ──┘
```

| Topology | ข้อดี | ข้อเสีย | เหมาะกับ |
|---|---|---|---|
| **Hub-and-Spoke** | Tunnel Count ต่ำสุด, บังคับ Traffic ผ่าน Security Inspection ที่ Hub ได้ง่าย | Latency สูงขึ้นถ้า Spoke ต้องคุยกันเอง (ต้องอ้อม Hub เสมอ), Hub เป็นจุดคอขวดของ Bandwidth | Retail/Branch ที่ Traffic ส่วนใหญ่วิ่งไป Data Center/Cloud เท่านั้น ไม่คุยกันเอง |
| **Full-Mesh** | Latency ต่ำสุดระหว่าง Site คู่ไหนก็ได้, ไม่มี Single Point of Bottleneck | Tunnel Count โตแบบ N² ที่ Scale ใหญ่, ยากต่อการบังคับ Central Security Inspection | องค์กรขนาดเล็ก-กลาง (< 50-80 Site) ที่ Site คุยกันเองบ่อย (เช่น Manufacturing Plant คุยกับ Warehouse) |
| **Regional-Mesh** | ผสมข้อดีทั้งสอง — Full-Mesh *ภายใน* Region (Site ใกล้กันคุยกันตรงได้ Latency ต่ำ) + Hub-and-Spoke *ข้าม* Region (ควบคุม Security/Bandwidth ได้) | ต้องออกแบบ Region และ Control Policy เพิ่ม (ซับซ้อนกว่า 2 แบบข้างบน) | องค์กรขนาดใหญ่ (Retail 100+ สาขา, ธนาคารหลายภูมิภาค) — **มาตรฐาน Design ของ CCIE ระดับ Enterprise** |

### บังคับ Topology ด้วย Control Policy (ไม่ใช่ Full-Mesh อัตโนมัติเสมอ)

ค่า Default ของ Cisco SD-WAN คือ Full-Mesh (ทุก cEdge เห็น TLOC ของทุก cEdge) — การจำกัด
Topology ทำผ่าน **Control Policy** ที่ vSmart (คนละชนิดกับ Data Policy ใน Step 387 ของ Part 39
ซึ่งควบคุม Traffic ที่ Data Plane ส่วน Control Policy ควบคุมว่า **OMP จะแจก TLOC/Route ให้ใครเห็น
ใครบ้าง** — ควบคุมที่ต้นตอก่อน Tunnel จะถูกสร้างเลย):

```
policy
  control-policy HUB-AND-SPOKE-TOPOLOGY
    sequence 10
      match route
        site-list SPOKE-SITES
      action reject                 ! ห้าม Spoke เห็น TLOC ของ Spoke อื่น
    sequence 20
      match route
        site-list HUB-SITES
      action accept                 ! ทุก Site ยังเห็น TLOC ของ Hub เสมอ
    default-action accept
  !
  lists
    site-list HUB-SITES
      site-id 100                   ! HQ (WAN-EDGE-1/2)
    site-list SPOKE-SITES
      site-id 200-299                ! Branch ทั้งหมด (Range Site ID)
!
apply-policy site-list SPOKE-SITES control-policy HUB-AND-SPOKE-TOPOLOGY out
```

`control-policy ... out` แปลว่า Policy นี้ Apply ตอน vSmart **ส่ง** OMP Update ออกไปให้ Site
ในกลุ่ม SPOKE-SITES — ผลคือ Spoke จะไม่เห็น TLOC ของ Spoke อื่นเลย (Reject) จึงไม่มีทางสร้าง
IPsec Tunnel ถึงกันได้ (ไม่มี TLOC = ไม่รู้จะสร้าง Tunnel ไปที่ไหน) แต่ยังเห็น TLOC ของ Hub
เสมอ (Accept) — ได้ผลลัพธ์เป็น Hub-and-Spoke ทั้งที่ Data Plane โดยธรรมชาติรองรับ Full-Mesh

> **ข้อสอบ CCIE มักถามแบบนี้**: "ทำไม Site A กับ Site B ไม่สามารถสร้าง IPsec Tunnel ถึงกันได้
> ทั้งที่ Control Connection ไปยัง vSmart ปกติทั้งคู่" — คำตอบมักไม่ใช่ปัญหา Data Plane เลย แต่เป็น
> **Control Policy ที่ vSmart กรอง TLOC ไม่ให้ทั้งสอง Site เห็นกัน** ต้องเช็คด้วย
> `show sdwan policy from-vsmart` บน cEdge เพื่อดู Policy ที่ได้รับจริง

---

## Step 752 — Multi-Region Fabric (MRF): Regional Hub และ Global Core Region

### ปัญหาของ Design แบบ Single-Region ที่ Scale ใหญ่มาก

Control Policy ใน Step 751 แก้ปัญหา Topology ได้ แต่ยังมีข้อจำกัดที่ **vSmart ทุกตัวยังต้องคง
Full State ของทุก Site ในทั้ง Fabric** (แจก/รับ OMP Update จากทุก cEdge ทุกตัว แค่ "กรอง" ตอนแจก
ต่อ) — ที่ Scale ระดับ **หลายพัน Site** (Global Retail Chain, ธนาคารข้ามทวีป) การให้ vSmart
กลุ่มเดียวดูแล Fabric ทั้งหมดทำให้ **Convergence Time ช้าลง, Scale ของ vSmart เองมีเพดาน, และ
Fault Domain ใหญ่เกินไป** (Policy ผิดที่ Region หนึ่งกระทบ Convergence ของทั้ง Fabric)

### MRF คืออะไร

**Multi-Region Fabric (MRF)** คือ Feature ของ Cisco Catalyst SD-WAN ที่แบ่ง Fabric ทั้งหมดออกเป็น
**Region** อิสระจากกัน โดยแต่ละ Region มี **Border Router** เป็นจุดเชื่อมต่อออกไปยัง Region อื่น
ผ่าน **Core Region** (Region ID 0 พิเศษ ที่ทำหน้าที่เป็น "Backbone" เชื่อม Region ทั้งหมดเข้าด้วยกัน)

```
                              ┌─────────────────────────┐
                              │   CORE REGION (Region 0)   │
                              │  (Transit Backbone ระหว่าง   │
                              │   Region — มักคือ MPLS/     │
                              │   Backbone ของ HQ/DC)       │
                              └──┬─────────────┬─────────┘
                     Border      │             │      Border
                     Router      │             │      Router
              ┌────────┴───────┐   │             │   ┌───────┴────────┐
              │  REGION 1 (North)  │   │             │   │  REGION 2 (South)  │
              │  Regional Hub +    │   │             │   │  Regional Hub +    │
              │  Edge Router หลาย    │   │             │   │  Edge Router หลาย    │
              │  ตัว (Branch สาขา)   │   │             │   │  ตัว (Branch สาขา)   │
              └────────────────┘   │             │   └────────────────┘
                                   │             │
                          ┌────────┴───────┐
                          │  REGION 3 (Central) │
                          │   (เช่น HQ/Data Center)│
                          └────────────────┘
```

### บทบาทของอุปกรณ์แต่ละแบบใน MRF

| Role | คำสั่งกำหนดใน System Config | หน้าที่ |
|---|---|---|
| **Edge Router** | `role edge-router` | cEdge ปกติที่ Branch — เห็นเฉพาะ TLOC ภายใน Region ตัวเอง และ TLOC ของ Border Router ประจำ Region |
| **Border Router** | `role border-router` | cEdge ที่ทำหน้าที่เป็น "ประตู" ระหว่าง Region ตัวเองกับ Core Region — เห็น TLOC ทั้งใน Region ตัวเองและ Core Region |
| **Core Region** | `region-id 0` (ค่าสงวนสำหรับ Core) | Region พิเศษที่ Border Router ทุก Region เชื่อมเข้ามา ทำหน้าที่ Transit ระหว่าง Region |

### ตัวอย่าง System Config กำหนด Region (บน cEdge)

```
system
  system-ip       172.20.10.1
  site-id         210
  region-id       1                  ! Edge Router สังกัด Region 1 (North)
  role            edge-router
!
! ตัวอย่าง Border Router ของ Region 1
system
  system-ip       1.1.1.31
  site-id         100
  region-id       1
  secondary-region-id 0              ! Border Router เห็นทั้ง Region 1 และ Core Region (0)
  role            border-router
```

### เมื่อไหร่ต้องใช้ MRF vs Design แบบ Flat Single-Region

| เงื่อนไข | Flat Single-Region (Control Policy อย่างเดียวพอ) | Multi-Region Fabric (MRF) |
|---|---|---|
| จำนวน Site | < 500-800 Site (ประมาณการตาม Cisco Design Guide) | หลักพัน Site ขึ้นไป |
| การกระจายทางภูมิศาสตร์ | ประเทศเดียว/Time Zone ใกล้กัน | ข้ามทวีป/หลาย Time Zone (เช่น APAC/EMEA/AMER) |
| ทีม Operation | ทีมเดียวดูแลทั้งหมด | หลายทีม Regional NOC ต้องการ Fault Isolation แยกจากกัน |
| ความต้องการ Compliance | ไม่มีข้อจำกัดข้าม Region | ต้องแยก Data Sovereignty (เช่น EU GDPR ต้องเก็บ Control Plane Metadata ในภูมิภาคตัวเอง) |
| Convergence Time ที่ยอมรับได้ | Single vSmart Cluster จัดการได้ทัน | ต้องการ Isolate Convergence Event ไม่ให้กระทบข้าม Region |

> **จุดสำคัญสำหรับ CCIE**: MRF ไม่ใช่ "Controller แยกกันคนละชุดต่อ Region" (นั่นคือการทำ **Multiple
> Fabric แยกกันจริงๆ** ซึ่งเป็นอีก Design ทางเลือกที่สิ้นเปลืองกว่า) — **vManage/vSmart/vBond
> ยังเป็น Controller Stack ชุดเดียวกันสำหรับทั้ง Fabric** MRF เป็นแค่การจัดกลุ่ม TLOC/Route
> Visibility ด้วย Region ID ภายใน Fabric เดียว ทำให้ Scale ดีขึ้นโดยไม่ต้องเพิ่ม Controller Stack

---

## Step 753 — Advanced Application-Aware Routing: SLA Class หลายชั้น + Fallback Chain ข้าม 3 Transport

### ทบทวนจาก Part 39 แล้วขยายให้ซับซ้อนขึ้น

Part 39 Step 386 สอน AAR ด้วย SLA Class เดียว (VOICE-SLA) กับ 2 Transport (mpls, biz-internet)
— งานจริงระดับ Enterprise ต้องรองรับ **Application หลายกลุ่มพร้อมกัน แต่ละกลุ่มมี SLA ต่างกัน**
และมักมี **3 Transport ขึ้นไป**: MPLS (เสถียรสุด แพงสุด), Internet/Broadband (ราคาถูก คุณภาพ
ผันแปร), และ LTE/5G (Backup สุดท้าย แพงต่อ GB มากที่สุด ควรใช้เท่าที่จำเป็น)

### ออกแบบ SLA Class ให้ตรงกับความต้องการแต่ละ Application Tier

| SLA Class | Loss | Latency | Jitter | Application ตัวอย่าง | ลำดับ Transport ที่ต้องการ |
|---|---|---|---|---|---|
| **VOICE-SLA** | 1% | 100 ms | 20 ms | VoIP, Video Conference | mpls → biz-internet → *ห้าม* lte (Jitter ของ LTE สูงเกินไปสำหรับเสียง) |
| **CRITICAL-DATA-SLA** | 2% | 150 ms | 30 ms | POS Transaction, ERP, Database Replication | mpls → biz-internet → lte (ยอมใช้ LTE ได้ถ้าจำเป็นจริงๆ) |
| **BEST-EFFORT-SLA** | 5% | 300 ms | 50 ms | Web Browsing, Email, Software Update | biz-internet → mpls → lte (ให้ Internet ราคาถูกรับ Traffic กลุ่มนี้ก่อนเสมอ ประหยัด MPLS ไว้ให้ Voice/Critical) |

### แนวคิด Fallback Chain — ต่างจาก Preferred-Color ตัวเดียวใน Part 39

Part 39 ใช้ `preferred-color mpls` เป็น Tie-breaker เดียว — Design ระดับ Enterprise ต้องมี
**ลำดับ Fallback ที่ชัดเจนเป็นขั้นบันได (Ordered Preference Chain)** ไม่ใช่แค่ Path เดียวที่ชอบ
ที่สุด เพราะถ้า Path ที่ชอบที่สุดหลุดพร้อมกันทั้งคู่ (เช่น MPLS ล่มทั้ง Circuit) Policy ต้องรู้ว่า
จะ "ร่วง" ไป Transport ไหนต่อโดยอัตโนมัติ **โดยไม่ Drop Traffic ทันทีเหมือน `strict`**

```
                    Fallback Chain สำหรับ CRITICAL-DATA-SLA (POS Transaction)

   ลำดับที่ 1: mpls         ──✅ ผ่าน SLA?──► ใช้ mpls
        │ ❌ ไม่ผ่าน/ล่ม
        ▼
   ลำดับที่ 2: biz-internet ──✅ ผ่าน SLA?──► ใช้ biz-internet
        │ ❌ ไม่ผ่าน/ล่ม
        ▼
   ลำดับที่ 3: lte           ──✅ ผ่าน SLA?──► ใช้ lte (Backup สุดท้าย ยอมให้ Latency สูงขึ้นได้)
        │ ❌ ไม่มี Path ไหนผ่านเลย
        ▼
   Drop Traffic (กรณีนี้เท่านั้นที่ยอม Drop เพราะไม่เหลือ Transport ให้ลองแล้ว)
```

### Policy เต็มรูปแบบ: 3 Application Tier ข้าม 3 Transport พร้อม Fallback Chain

```
policy
  !--- 1) นิยาม SLA Class ทั้ง 3 ระดับ ---
  sla-class VOICE-SLA
    latency        100
    loss           1
    jitter         20
  sla-class CRITICAL-DATA-SLA
    latency        150
    loss           2
    jitter         30
  sla-class BEST-EFFORT-SLA
    latency        300
    loss           5
    jitter         50
  !
  !--- 2) App-Route Policy หลัก ครอบคลุมทุก Service VPN ---
  app-route-policy ENTERPRISE-AAR-POLICY
    vpn-list ALL-SERVICE-VPNS
    !
    sequence 10                       ! VOICE — strict, ห้ามใช้ lte เด็ดขาด
      match
        dscp 46                       ! EF (Voice)
      action
        sla-class VOICE-SLA strict
        preferred-color mpls biz-internet   ! ลำดับ Fallback: mpls ก่อน แล้ว biz-internet
                                             ! (ไม่มี lte ในลิสต์ = lte ไม่ถูกพิจารณาเลยสำหรับ Voice)
    !
    sequence 20                       ! CRITICAL DATA (POS/ERP) — ยอมร่วงไป lte ได้เป็นทางเลือกสุดท้าย
      match
        dscp 26                       ! AF31 (Critical Data ตามมาตรฐาน RFC 4594)
      action
        sla-class CRITICAL-DATA-SLA
        preferred-color mpls biz-internet lte  ! Fallback Chain เต็ม 3 ลำดับ
    !
    sequence 30                       ! BEST EFFORT — ดัน Internet ก่อนเสมอเพื่อประหยัด MPLS
      match
        dscp 0                        ! Default/Best Effort
      action
        sla-class BEST-EFFORT-SLA
        preferred-color biz-internet mpls lte
    !
    default-action allow              ! Traffic อื่นที่ไม่ match sequence ไหน ให้ Routing ปกติ (ECMP)
  !
  lists
    vpn-list ALL-SERVICE-VPNS
      vpn 10
      vpn 20
      vpn 30
!
apply-policy site-list ALL-SITES app-route-policy ENTERPRISE-AAR-POLICY
```

- **Sequence 10 (VOICE)**: ใช้ `strict` เพราะเสียงที่คุณภาพแย่ (ผ่าน LTE) แย่กว่าไม่มีเสียงเลย
  — ยอม Drop ถ้าไม่มี Path ผ่าน SLA แต่จะไม่ยอม Fallback ไป `lte` เพราะไม่อยู่ใน `preferred-color`
  List เลย
- **Sequence 20 (CRITICAL DATA)**: ไม่ใช้ `strict` และมี `lte` เป็นตัวเลือกสุดท้ายใน Chain — POS
  Terminal ที่สาขาเล็กยอมให้ Transaction ช้าลงชั่วคราวผ่าน LTE ดีกว่าระบบขายหน้าร้านหยุดทำงาน
- **Sequence 30 (BEST EFFORT)**: จงใจเรียง `biz-internet` ก่อน `mpls` เพื่อ **Offload Traffic
  ที่ไม่สำคัญออกจาก MPLS Circuit ที่มีราคาแพงและ Bandwidth จำกัด** — เทคนิคนี้เรียกว่า **Transport
  Offload by Application Priority** เป็นเหตุผลทางเศรษฐศาสตร์หลักที่องค์กรใหญ่ลงทุนทำ SD-WAN

### Verify Fallback Chain ทำงานจริง

```
BRANCH-RTR# show sdwan app-route stats

TLOC COLOR      REMOTE COLOR   MEAN LOSS  MEAN LATENCY  MEAN JITTER  APPLIED SLA CLASS       RESULT
mpls            mpls           8          220           40           CRITICAL-DATA-SLA       FAIL (loss)
biz-internet    biz-internet   1          90            15           CRITICAL-DATA-SLA       PASS (selected)
lte             lte            2          140            35           CRITICAL-DATA-SLA       PASS (standby)
```

ตัวอย่างนี้แสดง MPLS Circuit มีปัญหา (Loss 8% เกิน SLA) — Policy จึงร่วงไปที่ `biz-internet`
โดยอัตโนมัติตาม Fallback Chain (ลำดับที่ 2) โดยไม่ต้อง Drop Traffic หรือรอ Routing Protocol
Reconverge แบบ Traditional WAN เลย

---

## Step 754 — High Availability Controller Design ระดับ Enterprise Scale

### ทบทวน Controller Placement จาก Part 39 (Step 390) แล้วขยายเป็นระดับ Production เต็มรูปแบบ

Part 39 แนะนำ vManage 3 ตัว, vSmart 2 ตัว, vBond 2 ตัวสำหรับ Lab ขนาดเล็ก — ที่ Scale ระดับ
หลายร้อย-หลายพัน Site ต้องคำนวณ Sizing จริงจากจำนวน cEdge ที่ต้องรองรับ และต้องมี **Disaster
Recovery (DR) Site** สำหรับ Controller เอง (ไม่ใช่แค่ Redundant ภายใน Data Center เดียว)

### ตาราง Sizing Controller ตามจำนวน cEdge (ตามแนวทาง Cisco Design Guide)

| จำนวน cEdge ใน Fabric | vManage Cluster | vSmart | vBond | หมายเหตุ |
|---|---|---|---|---|
| < 100 | 1-3 Node (Non-Cluster หรือ Cluster เล็ก) | 2 | 2 | ระดับ Lab/POC/องค์กรเล็ก |
| 100 – 1,000 | 3 Node Cluster | 2-4 | 2 | ระดับ Enterprise กลาง (เคสของหลักสูตรนี้ที่ 200 สาขา) |
| 1,000 – 6,000 | 3-6 Node Cluster (Scale ตาม NBI/API Load) | 4-8 (แบ่งตาม Region ถ้าใช้ MRF) | 2-4 | ระดับ Enterprise ใหญ่/Global |
| 6,000+ | 6 Node Cluster (เพดานสูงสุดปัจจุบัน) | 8+ กระจายตาม Region | 4+ | ระดับ Service Provider/Global Retail Chain |

> **หลักการ Sizing ที่ต้องจำ**: **vManage** Scale ตาม "ภาระ Management/API/Monitoring" (จำนวน
> Template Push, จำนวน Statistics ที่ Collect ต่อวินาที) ส่วน **vSmart** Scale ตาม "ภาระ Control
> Plane" (จำนวน OMP Peer และความถี่ Route Update) — ทั้งสองไม่ Scale ในสัดส่วนเดียวกันเสมอไป จึง
> ต้อง Monitor แยกกันและเพิ่ม Node ตามคอขวดจริงที่เจอ ไม่ใช่เพิ่มพร้อมกันตามสัดส่วนคงที่

### Diagram HA เต็มรูปแบบพร้อม Disaster Recovery Site

```
                    ┌─────────────────────────────────────────────────────┐
                    │              PRIMARY SITE (เช่น Data Center A)        │
                    │  ┌───────────────┐  ┌───────────┐  ┌───────────┐  │
                    │  │ vManage Cluster │  │  vSmart-1  │  │  vSmart-2  │  │
                    │  │  (Node 1-3)     │  │            │  │            │  │
                    │  └───────────────┘  └───────────┘  └───────────┘  │
                    │              ┌───────────┐  ┌───────────┐          │
                    │              │  vBond-1   │  │  vBond-2   │          │
                    │              └───────────┘  └───────────┘          │
                    └───────────────────────┬───────────────────────────┘
                                            │ Continuous DB Replication +
                                            │ Configuration Sync (Async/Sync
                                            │ ตามระยะทาง Data Center)
                    ┌───────────────────────┴───────────────────────────┐
                    │           DR SITE (เช่น Data Center B คนละภูมิภาค)     │
                    │  ┌───────────────┐  ┌───────────┐  ┌───────────┐  │
                    │  │ vManage Cluster │  │  vSmart-3  │  │  vSmart-4  │  │
                    │  │  (Standby/Warm) │  │ (Standby)  │  │ (Standby)  │  │
                    │  └───────────────┘  └───────────┘  └───────────┘  │
                    └───────────────────────────────────────────────────┘
                                            │
                    cEdge ทุกตัวถูก Config ให้รู้จัก vSmart/vBond ทั้ง 2 Site
                    (Vbond/Vsmart List มีทั้ง Primary + DR) — ถ้า Primary Site ล่มทั้งหมด
                    cEdge จะยัง Maintain Control Connection ไปยัง DR Site ได้ทันที
```

### หลักการออกแบบ DR สำหรับ Controller

| ประเด็น | คำแนะนำ |
|---|---|
| **vBond** | ควรมีอย่างน้อย 1 ตัวที่ DR Site เสมอ เพราะเป็นจุดแรกที่ Site ใหม่ (ZTP) ติดต่อเข้ามา — ถ้า vBond ทุกตัวอยู่ Site เดียวแล้วล่ม Site ใหม่จะ Onboard ไม่ได้เลย |
| **vSmart** | กระจายให้มีอย่างน้อย 1 ตัวที่ DR Site — cEdge ที่ Config List ของ vSmart ไว้ล่วงหน้าจะยัง Maintain OMP Session ต่อได้แม้ Primary Site ล่มทั้งหมด (Existing Tunnel ยังส่ง Traffic ได้ต่อตาม Fail-safe Behavior ที่ Part 39 อธิบายไว้ แต่ต้องมี vSmart เหลืออย่างน้อย 1 ตัวเพื่อ Push Route/Policy ใหม่ได้) |
| **vManage** | Cluster ที่ DR Site ควรเป็น **Warm Standby** (Sync Configuration Database ต่อเนื่อง) — RTO (Recovery Time Objective) สำหรับ vManage ยอมรับได้นานกว่า vSmart/vBond เพราะ Data Plane เดิมยังทำงานต่อได้แม้ Management Plane ล่มชั่วคราว |
| **Certificate/PKI** | Root CA ที่ใช้ Sign Certificate ของ Controller ทั้งหมดต้อง Backup แยกจาก Controller Site ทั้งสอง (เช่น Offline Root CA หรือ Cisco-managed PKI) เพราะถ้า CA หายไปพร้อม Controller จะไม่สามารถ Onboard Site ใหม่หรือ Renew Certificate ได้เลย |

---

## Step 755 — Cloud OnRamp for SaaS เชิงลึก: เลือก Local Breakout ที่ดีที่สุดต่อแอปโดยอัตโนมัติ

### ทบทวน Preview จาก Part 39 (Step 388) แล้วลงรายละเอียดกลไกจริง

Part 39 อธิบายแค่แนวคิดว่า Direct Cloud Access ทำให้ Branch ไม่ต้อง Hairpin ผ่าน HQ — Step นี้
อธิบายกลไกจริงว่า **Cloud OnRamp for SaaS** เลือก "ทางออก Internet ที่ดีที่สุด" ให้แต่ละ SaaS
App โดยอัตโนมัติได้อย่างไร ผ่าน 2 องค์ประกอบหลัก: **Probe การวัด Performance ต่อ Exit Path** และ
**SaaS Application Definition**

### กลไกการวัด Performance ต่อ Exit Path

```
                    BRANCH-RTR มี 2 ทางออก Internet ที่เป็นไปได้สำหรับ SaaS App (เช่น Office 365)

  ทางออกที่ 1: Direct Internet Access (DIA) จาก Branch เอง
       BRANCH-RTR ──biz-internet (ตรงจาก Site)──► Office 365 Endpoint ที่ใกล้ที่สุด

  ทางออกที่ 2: Backhaul ผ่าน Overlay ไปออกที่ HQ Gateway
       BRANCH-RTR ──Overlay──► WAN-EDGE-1 (HQ) ──Internet──► Office 365 Endpoint

  Cloud OnRamp for SaaS Gateway วัด Performance ทั้ง 2 ทางออกต่อ SaaS App นี้แบบ Real-time
  ด้วย HTTP/HTTPS Probe ไปยัง Endpoint จริงของ Office 365 (ไม่ใช่วัดแค่ Path Quality ทั่วไป
  แบบ AAR ปกติ แต่วัด "ตรงถึงปลายทาง SaaS" เพื่อรู้ Application Response Time จริง)
                                     │
                                     ▼
        เลือกทางออกที่ Response Time/Loss ดีที่สุด ให้ Traffic Office 365 ทั้งหมดของ Branch นี้
        (Sub-second Reroute ถ้า Path ที่เลือกไว้คุณภาพตกลง — เหมือนหลักการ AAR)
```

### Best VPN Exit Selection — เมื่อ Site มีหลาย Internet Exit ให้เลือก

Cisco SD-WAN ยังรองรับกรณีที่มีหลาย Site เป็น **Internet Gateway** ให้ Site อื่นเลือกออกผ่านได้
(เช่น HQ 2 แห่งที่มี Internet Circuit คุณภาพดี) — Feature นี้ชื่อ **Best VPN Exit Selection**
ทำงานร่วมกับ Cloud OnRamp for SaaS โดย Gateway จะ Probe ไปยัง SaaS Endpoint จากทุก Exit ที่มี
สิทธิ์ แล้วเลือก Exit ที่ดีที่สุดให้ Branch แต่ละแห่งโดยอัตโนมัติ ไม่จำเป็นต้อง Fix ว่า Branch
ต้อง Backhaul ไป HQ ไหนเสมอ

### ตัวอย่าง Policy Design เต็มรูปแบบสำหรับ Cloud OnRamp for SaaS

```
policy
  !--- 1) นิยาม SaaS Application ที่ต้องการ ---
  app-list SAAS-APPLICATIONS
    app office365
    app salesforce
    app webex
  !
  !--- 2) นิยาม Probe Path Class (คล้าย SLA Class แต่ผูกกับ SaaS Gateway) ---
  cloud-qos-service-policy SAAS-OPTIMIZATION-POLICY
    vpn-list ALL-SERVICE-VPNS
    saas-app-list SAAS-APPLICATIONS
    action
      probe-path direct-internet     ! Probe ทาง DIA จาก Site ตัวเอง
      probe-path backhaul-hub-1      ! Probe ทาง Backhaul ไป Hub 1 (เช่น WAN-EDGE-1)
      probe-path backhaul-hub-2      ! Probe ทาง Backhaul ไป Hub 2 (เช่น WAN-EDGE-2)
      select best-performance         ! เลือก Path ที่ Response Time ต่อ SaaS Endpoint ดีที่สุด
  !
  lists
    vpn-list ALL-SERVICE-VPNS
      vpn 10
      vpn 30
!
apply-policy site-list ALL-BRANCH-SITES cloud-qos-service-policy SAAS-OPTIMIZATION-POLICY
```

### Verify ผลลัพธ์การเลือก Exit ต่อ SaaS App

```
BRANCH-RTR# show sdwan cloud-onramp saas performance

APPLICATION   EXIT-PATH             AVG RESPONSE-TIME   LOSS   SELECTED
office365     direct-internet       45 ms                0%     YES
office365     backhaul-hub-1        180 ms               1%     NO
salesforce    direct-internet       120 ms               2%     NO
salesforce    backhaul-hub-2        70 ms                0%     YES
```

สังเกตว่า **Office 365 เลือก Direct Internet** แต่ **Salesforce เลือก Backhaul ผ่าน Hub 2** —
เพราะ Cloud OnRamp for SaaS วัด Performance **ต่อแอปแยกกัน** ไม่ได้ตัดสินใจแบบ "ทาง Internet
ดีที่สุดเสมอสำหรับทุกแอป" ซึ่งเป็นจุดที่ต่างจาก AAR ทั่วไปที่มองแค่ Path Quality โดยไม่รู้จัก
SaaS Endpoint จริง

> **เชื่อมต่อไปข้างหน้า**: Step นี้ให้ภาพ Cloud OnRamp for SaaS ในระดับ Policy Design ที่ใช้งาน
> จริงได้ — รายละเอียดการ Integrate กับ Cloud OnRamp for IaaS/Multi-Cloud แบบเต็มรูปแบบ (รวม
> AWS Transit Gateway, Azure Virtual WAN Hub, Multi-Cloud Fabric ข้าม Provider) จะสอนต่อใน
> **[Part 93]** ของหลักสูตร (ช่วง Cloud Networking) ตามที่ Part 39 ได้ตั้งใจค้างไว้

---

## Step 756 — Cloud OnRamp for IaaS: ขยาย Fabric เข้าสู่ AWS/Azure/GCP

### แนวคิด: cEdge เสมือนใน Public Cloud คือ Site หนึ่งของ Fabric เดียวกัน

**Cloud OnRamp for IaaS** ให้ vManage Deploy **cEdge เสมือน** (Cisco Catalyst 8000V หรือรุ่นก่อน
หน้าคือ CSR1000v ที่ปรับให้รัน SD-WAN Mode) เข้าไปใน Virtual Network ของ Public Cloud (VPC บน
AWS, VNet บน Azure, VPC บน GCP) โดยอัตโนมัติผ่าน API ของ Cloud Provider เอง — cEdge เสมือนนี้
**เข้าร่วม Fabric เดียวกันกับ Branch/HQ ทุกตัว** (มี System-IP, Site-ID, TLOC ของตัวเอง) ทำให้
Workload บน Cloud กลายเป็น "อีก Site หนึ่งของ SD-WAN" แทนที่จะต้องคอนฟิก VPN Gateway/Direct
Connect แยกต่างหากแบบ Traditional Design

```
                    Traditional: เชื่อมต่อ Cloud ด้วย VPN Gateway/Direct Connect แยกต่างหาก

  Branch/HQ ──MPLS/Internet──► On-Prem VPN Gateway ──IPsec VPN (คอนฟิกมือ)──► AWS VPN Gateway
                                                                                      │
                                                                              (ต้องคอนฟิกทั้งสองฝั่ง
                                                                               แยกจาก SD-WAN Fabric)

                    Cisco SD-WAN Cloud OnRamp for IaaS: cEdge เสมือนเข้าร่วม Fabric เดียวกันเลย

                         ┌────────────────────────────────────┐
                         │      vManage สั่งผ่าน Cloud Provider API   │
                         │   (สร้าง VPC/VNet, Subnet, Deploy cEdge)   │
                         └───────────────────┬────────────────────┘
                                            │
   ┌───────────────┐              ┌─────────┴─────────┐
   │  BRANCH-RTR     │◄═══════════│  cEdge เสมือน (AWS)  │  Site ID ใหม่, TLOC ของตัวเอง
   │  (Site 200)      │  IPsec     │  (เช่น Site 900)      │  เข้าร่วม OMP/Control Plane เดียวกัน
   └───────────────┘  Overlay     └─────────┬─────────┘
                     (Auto-built            │
                     เหมือน Site ปกติ)  ┌────┴────┐
                                          │ Workload │  (EC2, Database, Application Server)
                                          │  บน AWS   │
                                          └─────────┘
```

### ขั้นตอนที่ vManage ทำอัตโนมัติเมื่อสั่ง Cloud OnRamp for IaaS

| ขั้นตอน | รายละเอียด |
|---|---|
| 1. เชื่อมต่อ Cloud Account | ผู้ดูแลระบบใส่ API Credential ของ AWS/Azure/GCP เข้า vManage (Configuration > Cloud OnRamp for IaaS) |
| 2. เลือก Host VPC/VNet | เลือกหรือให้ vManage สร้าง VPC/VNet ใหม่สำหรับ Transit Gateway ของ SD-WAN โดยเฉพาะ |
| 3. Deploy cEdge เสมือน | vManage เรียก Cloud API สร้าง Instance (Catalyst 8000V), กำหนด Interface, ผูก Elastic IP/Public IP อัตโนมัติ |
| 4. Onboard เข้า Fabric | cEdge เสมือนทำ ZTP เหมือน cEdge จริงทุกขั้นตอน (Step 385 ของ Part 39) — ติดต่อ vBond, รับ Certificate, ขึ้น Control Connection |
| 5. เชื่อม Workload VPC/VNet | vManage ช่วย Peer Transit VPC เข้ากับ Workload VPC อื่นๆ ในบัญชี Cloud เดียวกัน (VPC Peering/Transit Gateway Attachment) โดยอัตโนมัติ |

### ข้อดีเทียบกับการต่อ Cloud แบบ Traditional VPN/Direct Connect

| ประเด็น | Traditional VPN/Direct Connect | Cloud OnRamp for IaaS |
|---|---|---|
| การคอนฟิก IPsec | ต้องตั้ง Pre-shared Key/Certificate คู่ต่อคู่เอง | Auto-built ผ่าน TLOC/OMP เหมือน Site ปกติ |
| Policy/Security | ต้องตั้ง ACL/Route แยกสำหรับ Cloud โดยเฉพาะ | ใช้ Centralized Policy เดียวกันกับทั้ง Fabric (Data Policy, AAR, VPN Segment เดิม) |
| Application-Aware Routing | ไม่มี — Cloud Link เป็นแค่ Static Path | Cloud Link ก็เป็น TLOC ที่ AAR วัดคุณภาพและเลือก Path ได้เหมือน Site อื่น |
| การขยาย Multi-Cloud | ต้องคอนฟิกแยกทุก Provider เอง | vManage จัดการหลาย Cloud Provider จากจุดเดียว (Single Pane of Glass) |

> **Preview เท่านั้น**: รายละเอียดการออกแบบ Transit Gateway แบบ Multi-Region บน Cloud, การทำ
> Multi-Cloud Fabric ข้าม AWS+Azure+GCP พร้อมกัน, และ Security Policy ระหว่าง Cloud Workload
> กับ On-Prem แบบเต็มรูปแบบ จะสอนต่อใน **[Part 93]** เช่นเดียวกับ Step 755

---

## Step 757 — SD-WAN Security Integration: Enterprise Firewall, IPS, URL Filtering, Umbrella

### ทำไม Branch ยุคใหม่ไม่จำเป็นต้องมี Firewall Appliance แยกอีกต่อไป

Design เดิมของหลักสูตรนี้ ([Part 43-45](part-043-security-architecture-firepower-ise.md)) วาง
**FW-1 (ASA/FTD)** เป็น Physical Firewall แยกที่ HQ — Branch เล็กที่ไม่มีทีม Security ประจำที่
มักไม่มี Firewall Appliance ของตัวเอง (Traffic ต้อง Hairpin กลับ HQ เพื่อผ่าน FW-1 เสมอ ซึ่งขัด
กับหลักการ Direct Cloud Access ของ Step 755) — **Cisco SD-WAN ฝัง Security Stack ไว้ในตัว cEdge
เอง** ทำให้ Branch Breakout Internet ตรงได้ **โดยยังมี Security Inspection ระดับ Enterprise**

### Security Stack ที่ฝังอยู่ใน cEdge

| Feature | หน้าที่ | เทียบเคียงกับที่เรียนมาก่อนหน้า |
|---|---|---|
| **Enterprise Firewall** | Zone-based Firewall (คล้าย ZBFW ที่ [Part 45](part-045-advanced-acl-zbfw.md) สอน) ทำงานบน cEdge เอง กรอง Traffic ตาม Zone/Application | เหมือน ZBFW แต่ Policy Push จาก vManage ครั้งเดียวทั้ง Fabric |
| **IPS (Intrusion Prevention System)** | ใช้ Signature จาก **Cisco Talos** ตรวจจับ Traffic อันตราย/Exploit แบบ Real-time | เทียบเคียงกับ IPS Module บน FTD ที่ [Part 43](part-043-security-architecture-firepower-ise.md) สอน แต่ทำที่ Branch Edge โดยตรง |
| **URL Filtering** | บล็อก Website ตาม Category (Malware, Gambling, Social Media) โดยไม่ต้องส่ง Traffic ไปตรวจที่ Proxy กลาง | ลดภาระ Central Web Proxy ที่ HQ |
| **Cisco Umbrella Integration** | ส่ง DNS Query จาก Branch ไปตรวจสอบกับ Umbrella Cloud (DNS-Layer Security) ก่อนแม้แต่จะสร้าง Connection — บล็อกโดเมนอันตรายตั้งแต่ชั้น DNS | เพิ่มชั้นการป้องกันก่อนถึงชั้น Firewall/IPS อีกที (Defense in Depth) |
| **Advanced Malware Protection (AMP)** | ตรวจ File ที่ Download ผ่าน Cloud Sandbox | ทำงานร่วมกับ IPS/URL Filtering เป็นชุดเดียวกัน |

### Diagram สถาปัตยกรรม Security ที่ Branch Edge

```
                       User Traffic ที่ BRANCH-RTR (cEdge)
                                    │
                    ┌───────────────┴────────────────┐
                    │  1. DNS Query ก่อนเชื่อมต่อ         │
                    │     → ส่งไปตรวจกับ Cisco Umbrella   │──► Block ถ้าโดเมนอันตราย
                    │     (Cloud-delivered DNS Security)  │
                    └───────────────┬────────────────┘
                                    │ (DNS ผ่าน = โดเมนปลอดภัย)
                    ┌───────────────┴────────────────┐
                    │  2. Enterprise Firewall (Zone-based) │
                    │     ตรวจ Zone/Application/Port         │──► Drop ถ้าไม่ตรง Policy
                    └───────────────┬────────────────┘
                                    │ (ผ่าน Firewall Rule)
                    ┌───────────────┴────────────────┐
                    │  3. IPS (Cisco Talos Signature)     │──► Drop/Alert ถ้าเจอ Exploit Pattern
                    └───────────────┬────────────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │  4. URL Filtering + AMP (ถ้าเป็น Web/File) │──► Block ตาม Category/Malware Scan
                    └───────────────┬────────────────┘
                                    │ (ผ่านทุกชั้น)
                                    ▼
                         ออก Internet ตรงจาก Branch (DIA)
                         หรือเข้า Overlay ไปยัง Internal App
```

### ตัวอย่าง Unified Security Policy บน vManage (Conceptual Config)

```
policy
  !--- Firewall Zone-based Policy ---
  security-policy UNIFIED-BRANCH-SECURITY
    zone-pair BRANCH-LAN-to-INTERNET
      source-zone   LAN-ZONE
      dest-zone     INTERNET-ZONE
      firewall-policy ALLOW-WEB-BLOCK-P2P
      ips-policy      TALOS-STANDARD-PROTECTION
      url-filtering   BLOCK-MALWARE-GAMBLING-CATEGORY
      advanced-malware-protection ENABLE
  !
  !--- DNS-Layer Security ผ่าน Umbrella ---
  umbrella-integration
    dns-redirect  enable
    umbrella-org-id  123456
    dns-policy    BLOCK-KNOWN-MALICIOUS-DOMAINS
!
apply-policy site-list ALL-BRANCH-SITES security-policy UNIFIED-BRANCH-SECURITY
```

### เมื่อไหร่ยังต้องใช้ Firewall แบบ Centralized (FW-1) แทนหรือควบคู่กับ On-Box Security

| สถานการณ์ | คำแนะนำ |
|---|---|
| Branch ขนาดเล็ก-กลาง, ไม่มี Compliance ซับซ้อน | On-Box Security ของ cEdge เพียงพอ — ประหยัด Hardware และ Latency ต่ำที่สุด |
| Data Center/HQ ที่ต้องการ Next-Gen Firewall เต็มรูปแบบ (Deep Packet Inspection ระดับสูง, Threat Intelligence แบบ Advanced) | ยังควรมี FW-1 (Firepower) แบบ Centralized เพราะ Throughput/Feature Set สูงกว่า On-Box cEdge Security |
| องค์กรที่ต้อง PCI-DSS Compliance สำหรับ POS Traffic (ดู Step 758) | ใช้ **ทั้งคู่ร่วมกัน** — On-Box Security กรองชั้นแรกที่ Branch, บังคับ POS Traffic ผ่าน Centralized Firewall ที่ HQ อีกชั้นก่อนถึง Payment Gateway |

---

## Step 758 — Segmentation at Scale: ออกแบบ VPN Segment แบบ Multi-Tenant ทั้ง Fabric

### ทบทวนแนวคิด VPN Segment จาก Part 39 แล้วขยายเป็น Retail Multi-Tenant Design

Part 39 Step 387 สอน VPN Segment ระดับพื้นฐาน (VPN 10/20/30/40 = SALES/VOICE/SERVERS/WIFI) —
ที่ Scale ระดับ Retail Chain 200 สาขา ความต้องการ Segmentation เปลี่ยนโฟกัสจาก "แยกตาม Department"
เป็น **"แยกตามระดับความไว้วางใจ (Trust Level) และข้อกำหนด Compliance"** ซึ่งต้อง **สม่ำเสมอทุก
สาขา (Consistent VPN Numbering ทั้ง Fabric)** เพื่อให้ Policy ที่ตั้งครั้งเดียวใช้ได้ทุก Site

### แผน VPN Segment มาตรฐานสำหรับ Retail Enterprise

| VPN Number | ชื่อ Segment | Trust Level | ตัวอย่าง Traffic | ข้อกำหนดพิเศษ |
|---|---|---|---|---|
| **VPN 0** | Transport | - | TLOC/Underlay เท่านั้น | มาตรฐานเดิมจาก Part 39 |
| **VPN 512** | Management | สูงสุด | SSH/NETCONF ไปยัง vManage, Out-of-Band | มาตรฐานเดิมจาก Part 39 |
| **VPN 100** | **POS (Point-of-Sale)** | สูง (ต้อง PCI-DSS) | เครื่อง POS, Payment Terminal, Card Reader | ต้อง Isolate เด็ดขาดจาก Segment อื่น, เข้าถึง Payment Gateway ผ่าน Centralized Firewall เท่านั้น |
| **VPN 200** | **Corporate LAN** | กลาง-สูง | Store Manager PC, Back-office, Inventory System | เข้าถึง Data Center/ERP Application ได้ |
| **VPN 300** | **Guest Wi-Fi** | ต่ำสุด | Customer Wi-Fi ในร้าน | Internet-only, ห้ามเข้าถึง Internal Network ใดๆ เด็ดขาด |
| **VPN 400** | **IoT/Digital Signage** | ต่ำ-กลาง | กล้อง CCTV, จอโฆษณาดิจิทัล, Sensor ร้านค้า | เข้าถึงเฉพาะ Management Server ของตัวเอง ห้ามคุยกับ POS/Corporate |

### Diagram Segmentation ทั้ง Fabric แบบ Consistent

```
                     ทุกสาขา (Store-001 ถึง Store-200) ใช้ VPN Numbering เดียวกันเป๊ะ

   Store-001 cEdge          Store-002 cEdge          ...        Store-200 cEdge
   ┌──────────────┐         ┌──────────────┐                    ┌──────────────┐
   │ VPN 100 (POS) │         │ VPN 100 (POS) │                    │ VPN 100 (POS) │
   │ VPN 200 (Corp)│         │ VPN 200 (Corp)│                    │ VPN 200 (Corp)│
   │ VPN 300 (Guest)│         │ VPN 300 (Guest)│                    │ VPN 300 (Guest)│
   │ VPN 400 (IoT) │         │ VPN 400 (IoT) │                    │ VPN 400 (IoT) │
   │ VPN 512 (Mgmt)│         │ VPN 512 (Mgmt)│                    │ VPN 512 (Mgmt)│
   └──────┬───────┘         └──────┬───────┘                    └──────┬───────┘
          │                        │                                    │
          └────────────────────────┴──────────────┬─────────────────────┘
                                                   │ OMP Route แจกแยกตาม VPN
                                                   ▼
                                     Data Center / HQ (VPN 100/200/512 เท่านั้น
                                     — HQ ไม่มี VPN 300/400 เพราะไม่ใช่หน้าร้าน)
```

### Data Policy บังคับ Isolation ระหว่าง Segment ทั้ง Fabric

```
policy
  data-policy MULTI-TENANT-ISOLATION-POLICY
    vpn-list ALL-STORE-VPNS
    !
    sequence 10                         ! Guest ห้ามเข้าถึง POS เด็ดขาด
      match
        source-data-prefix-list GUEST-VPN300-PREFIX
        destination-data-prefix-list POS-VPN100-PREFIX
      action drop
    !
    sequence 20                         ! Guest ห้ามเข้าถึง Corporate เด็ดขาด
      match
        source-data-prefix-list GUEST-VPN300-PREFIX
        destination-data-prefix-list CORP-VPN200-PREFIX
      action drop
    !
    sequence 30                         ! IoT ห้ามเข้าถึง POS เด็ดขาด (ป้องกัน Lateral Movement)
      match
        source-data-prefix-list IOT-VPN400-PREFIX
        destination-data-prefix-list POS-VPN100-PREFIX
      action drop
    !
    sequence 40                         ! POS เข้าถึง Payment Gateway ได้เฉพาะผ่าน Centralized FW ที่ HQ
      match
        source-data-prefix-list POS-VPN100-PREFIX
        destination-data-prefix-list PAYMENT-GATEWAY-PREFIX
      action accept
        set-service   FW-HQ-INSPECT      ! บังคับ Redirect ผ่าน Firewall Service ที่ HQ ก่อนเสมอ
    !
    default-action allow
  !
  lists
    vpn-list ALL-STORE-VPNS
      vpn 100
      vpn 200
      vpn 300
      vpn 400
!
apply-policy site-list ALL-STORE-SITES data-policy MULTI-TENANT-ISOLATION-POLICY
```

- **Sequence 40** ใช้ `set-service` เพื่อบังคับ Route Traffic POS→Payment Gateway ผ่าน **Service
  Chain ไปยัง Firewall ที่ HQ** ก่อนเสมอ (ดูแนวคิด Service Route ที่ OMP แจกใน Part 39 Step 383)
  นี่คือจุดที่ Segmentation ของ SD-WAN ผูกกับ PCI-DSS Compliance ได้จริง — Auditor ต้องเห็นว่า
  Payment Traffic ผ่าน Firewall Inspection เสมอ ไม่มีทาง Bypass ได้แม้แต่ Site เดียว

> **จุดสำคัญสำหรับ CCIE Design**: การรักษา **VPN Numbering ให้เหมือนกันทุก Site (Consistent
> Numbering)** คือกฎเหล็กของ Segmentation at Scale — ถ้า Store บางสาขาใช้ VPN 100 = POS แต่อีก
> สาขาใช้ VPN 100 = Corporate (เพราะทีม Local Deploy เองไม่ตามมาตรฐาน) Centralized Policy ที่
> ตั้งจาก vManage ครั้งเดียวจะพังทันทีเพราะ Policy อ้างอิงเลข VPN เดียวกันแต่ความหมายไม่ตรงกัน

---

## Step 759 — กลยุทธ์ Migration จาก MPLS/Traditional WAN องค์กรขนาดใหญ่ สู่ SD-WAN

### ทบทวนจาก Part 39 แล้วขยายเป็น Migration ระดับ 200+ Site

Part 39 Step 389 สอนแนวคิด Coexistence และ Site-by-Site Migration ในระดับหลักการ — Step นี้จะ
วาง **Phased Rollout Plan แบบ Wave-based** ที่ใช้จริงกับองค์กรขนาดใหญ่ที่มี MPLS Contract ผูกพัน
อยู่กับ Carrier (มักมีสัญญาหลายปี ทำให้ Migration ต้อง "ทยอย" ไม่ใช่ "เปลี่ยนพร้อมกันทีเดียว")

### Wave-based Rollout Plan

```
Wave 0: Pilot (2-5% ของ Site ทั้งหมด)
   เลือก Site ความเสี่ยงต่ำที่สุด (Branch เล็ก, ไม่มี Critical Application, ทีม IT Local เข้มแข็ง)
   วัตถุประสงค์: พิสูจน์ ZTP/Template/Policy ทำงานถูกต้องในสภาพแวดล้อมจริง ไม่ใช่ Lab
              │
              ▼
Wave 1: Non-Critical Sites (20-30% ของ Site ทั้งหมด)
   Branch ทั่วไปที่ไม่มี POS/Payment หรือมี Downtime Window ยอมรับได้
   วัตถุประสงค์: ทดสอบ Scale ของ Template/Policy Push พร้อมกันหลาย Site
              │
              ▼
Wave 2: Standard Retail Sites (40-50% ของ Site ทั้งหมด — ส่วนใหญ่ของ Fabric)
   Branch มาตรฐานที่มี POS Segment เต็มรูปแบบ (Step 758) ต้องทดสอบ Segmentation/Compliance จริง
              │
              ▼
Wave 3: Critical/Flagship Sites + Regional Hub (5-10%)
   Site สำคัญที่ Downtime กระทบธุรกิจสูง, Regional Hub ที่ Site อื่นพึ่งพา (ต้อง Migrate หลัง
   Spoke ทั้งหมด Migrate สำเร็จแล้ว เพื่อลด Blast Radius ถ้ามีปัญหา)
              │
              ▼
Wave 4: HQ / Data Center (Site สุดท้าย)
   Migrate ท้ายสุดเสมอ เพราะเป็น Site ที่มี Application สำคัญที่สุดและ Site อื่นทั้งหมดพึ่งพา
```

### Coexistence Period Design — รายละเอียดที่ต้องวางแผนต่อ Site

| องค์ประกอบ | การออกแบบช่วง Coexistence |
|---|---|
| **MPLS Circuit เดิม** | คงไว้จนกว่า SD-WAN Overlay จะพิสูจน์เสถียรภาพครบ **Soak Period** ที่กำหนด (แนะนำ 30-60 วันต่อ Site ก่อน Decommission) |
| **Router เดิม vs cEdge ใหม่** | ถ้า Router เดิมรองรับ Controller-Mode ([Part 39 Step 389](part-039-sd-wan-fundamentals.md)) ให้ Reimage ได้เลย — ถ้าไม่รองรับ ต้องวาง **cEdge เครื่องใหม่คู่กับ Router เดิม (Parallel Circuit)** ระหว่าง Migrate |
| **Routing ระหว่าง Legacy กับ SD-WAN** | ใช้ **Redistribution** ระหว่าง OMP กับ OSPF/BGP เดิมที่ Site ยังไม่ Migrate (Route Leak ผ่าน Service VPN ตามที่ Part 39 Step 383 อธิบาย) เพื่อให้ Site ที่ Migrate แล้วกับยังไม่ Migrate คุยกันได้ต่อเนื่อง |
| **DNS/Application Dependency** | ตรวจสอบว่า Application ใดๆ Hard-code IP ของ Router เดิมไว้หรือไม่ก่อน Cutover (ปัญหาที่พบบ่อยที่สุดในงาน Migration จริง) |
| **Monitoring ระหว่าง Cutover** | เปิด `show sdwan app-route stats` และ `show sdwan bfd sessions` Monitor ต่อเนื่องช่วง Cutover Window เพื่อจับปัญหาทันทีที่เกิด |

### Risk Mitigation และ Rollback Plan

| ความเสี่ยง | มาตรการลดความเสี่ยง |
|---|---|
| cEdge ใหม่ Onboard ผ่าน ZTP ไม่สำเร็จที่ Site จริง | เตรียม **Golden Config Backup แบบ Manual (USB/Console)** สำรองไว้เผื่อ ZTP ล้มเหลวจาก Network ที่ Site (Firewall บล็อก DTLS Port เป็นต้น) |
| Application สำคัญ Hard-code Path เดิม | ทำ **Application Dependency Mapping** ก่อน Wave 2 เป็นต้นไป (ที่มี Site จำนวนมากขึ้น) |
| MPLS Contract ผูกพันจนกว่าจะหมดสัญญา | วางแผน **Decommission ตามรอบ Contract Renewal** ไม่ตัด MPLS ก่อนสัญญาสิ้นสุดถ้าไม่มี Early-termination Clause ที่คุ้มค่า |
| Cutover ทำให้ Business หยุดชะงัก | กำหนด **Maintenance Window ที่ชัดเจนต่อ Site** พร้อม **Rollback Plan แบบ Time-boxed** (ถ้าภายใน 2 ชั่วโมงแรกหลัง Cutover มีปัญหาที่แก้ไม่ได้ ให้ Rollback กลับ Router เดิมทันที ไม่ต้อง Debug ต่อหน้า Business Impact) |
| ทีม Operation ไม่คุ้นเคยกับ SD-WAN CLI/vManage | จัดทำ **Runbook มาตรฐาน** (Checklist คำสั่ง Verify จาก Part 39 Step 389 + Step นี้) และ Training ทีม NOC ก่อน Wave 1 |

---

## Step 760 — Lab เต็มรูปแบบ (Design-Level): SD-WAN Design Document สำหรับ Retail 200 สาขา

> **บริบท Lab**: บริษัท Retail แห่งหนึ่งมี 200 สาขาทั่วประเทศ (แบ่งเป็น 4 ภูมิภาค) ปัจจุบันใช้
> MPLS L3VPN เป็น WAN หลักทั้งหมด (Contract 3 ปีที่เหลืออีก 8 เดือน) ต้องการ Migrate สู่ Cisco
> SD-WAN พร้อมรองรับ POS/Corporate/Guest Segmentation, ใช้งาน Microsoft 365 เป็น SaaS หลัก และ
> ต้องออกแบบให้รองรับการขยายสาขาต่อเนื่องในอนาคตโดยไม่ต้อง Redesign ใหม่

### 1) MRF Regional Design

| Region ID | ชื่อ Region | จำนวนสาขา | Regional Hub | Border Router |
|---|---|---|---|---|
| 0 (Core) | Core Region | - | Data Center กลาง (HQ) | - |
| 1 | North | 55 สาขา | REGIONAL-HUB-NORTH | BORDER-RTR-NORTH |
| 2 | Central | 60 สาขา | REGIONAL-HUB-CENTRAL (รวม HQ) | BORDER-RTR-CENTRAL |
| 3 | South | 50 สาขา | REGIONAL-HUB-SOUTH | BORDER-RTR-SOUTH |
| 4 | East | 35 สาขา | REGIONAL-HUB-EAST | BORDER-RTR-EAST |

```
                              ┌───────────────────────────┐
                              │   CORE REGION (Region 0)     │
                              │   Data Center กลาง (HQ)       │
                              │   WAN-EDGE-1 / WAN-EDGE-2      │
                              └──┬──────┬──────┬──────┬────┘
                    Border       │      │      │      │      Border
              ┌──────────┴──┐ ┌──┴───┐┌─┴────┐┌─┴───┴──────┐
              │ REGION 1 North│ │Region 2││Region 3││ REGION 4 East │
              │ 55 สาขา         │ │Central ││South   ││ 35 สาขา         │
              │ Hub-Spoke ภายใน │ │60 สาขา ││50 สาขา ││ Hub-Spoke ภายใน │
              │ Region          │ │(รวม HQ)││        ││ Region          │
              └────────────────┘ └────────┘└────────┘└────────────────┘
```

แต่ละ Region ใช้ Topology **Hub-and-Spoke ภายใน Region** (ตาม Step 751 — Retail สาขาไม่คุยกัน
เอง) และเชื่อมข้าม Region ผ่าน Core Region เท่านั้น (ตาม Step 752)

### 2) Application-Aware Policy — 3 Transport (MPLS + Internet + LTE)

ทุกสาขาติดตั้ง Transport 3 ประเภท: **MPLS (Legacy Circuit ที่ยังไม่หมดสัญญา)**, **Broadband
Internet (Circuit ใหม่ที่ติดตั้งเพิ่ม)**, และ **LTE (Backup ฉุกเฉิน)** — ใช้ Policy ตามที่ออกแบบ
ไว้ใน Step 753 (`ENTERPRISE-AAR-POLICY`) ครอบคลุมทั้ง VPN 100 (POS), 200 (Corporate) โดยตรง:

| Application | SLA Class | Fallback Chain |
|---|---|---|
| VoIP ร้านค้า (Call Center Integration) | VOICE-SLA (strict) | mpls → biz-internet (ห้าม lte) |
| POS Transaction | CRITICAL-DATA-SLA | mpls → biz-internet → lte |
| Back-office/Email/Web | BEST-EFFORT-SLA | biz-internet → mpls → lte |

### 3) Cloud OnRamp for SaaS — Microsoft 365

```
policy
  app-list M365-APPS
    app office365
  !
  cloud-qos-service-policy M365-DIRECT-BREAKOUT
    vpn-list CORPORATE-VPN-LIST
    saas-app-list M365-APPS
    action
      probe-path direct-internet
      probe-path backhaul-regional-hub
      select best-performance
  !
  lists
    vpn-list CORPORATE-VPN-LIST
      vpn 200
!
apply-policy site-list ALL-STORE-SITES cloud-qos-service-policy M365-DIRECT-BREAKOUT
```

ทุกสาขา Breakout M365 ตรงจาก Local Internet Circuit ก่อนเสมอ (ตาม Step 755) ลด Load บน MPLS
Circuit เดิมลงอย่างมีนัยสำคัญ — ผลคือ Bandwidth บน MPLS ที่เหลือใช้ได้เต็มที่สำหรับ POS/VoIP

### 4) VPN Segment Plan — POS / Corporate / Guest (ตาม Step 758)

| VPN | Segment | สาขาใช้ | HQ ใช้ | Isolation Policy |
|---|---|---|---|---|
| 100 | POS | ✅ ทุกสาขา | ✅ (เชื่อม Payment Gateway) | ห้าม Guest/IoT เข้าถึง, ต้องผ่าน Centralized FW ก่อนถึง Payment Gateway |
| 200 | Corporate | ✅ ทุกสาขา | ✅ | เข้าถึง ERP/M365/Internal App ได้ |
| 300 | Guest Wi-Fi | ✅ ทุกสาขา | ❌ (HQ ไม่มี Guest Wi-Fi) | Internet-only เด็ดขาด |
| 400 | IoT/CCTV | ✅ ทุกสาขา | ✅ | เข้าถึงเฉพาะ Management Server ของตัวเอง |
| 512 | Management | ✅ ทุกสาขา | ✅ | Out-of-Band เท่านั้น |

Data Policy `MULTI-TENANT-ISOLATION-POLICY` จาก Step 758 Apply กับ `site-list ALL-STORE-SITES`
ทั้ง 200 สาขาโดยไม่ต้องปรับแก้อะไรเพิ่ม (เพราะ VPN Numbering สม่ำเสมอทุกสาขาตั้งแต่ต้น)

### 5) Timeline การ Migration แบบ Phased (อ้างอิง Wave Plan จาก Step 759)

| Wave | ช่วงเวลา | จำนวนสาขา | รายละเอียด |
|---|---|---|---|
| Wave 0 (Pilot) | เดือนที่ 1 | 6 สาขา (3% ของ 200) | เลือกสาขาเล็กจาก Region Central ที่ทีม IT Local เข้มแข็ง |
| Wave 1 (Non-Critical) | เดือนที่ 2-3 | 50 สาขา | สาขาที่ไม่มี Call Center/VoIP ซับซ้อน |
| Wave 2 (Standard Retail) | เดือนที่ 4-6 | 100 สาขา | สาขามาตรฐานที่มี POS เต็มรูปแบบ — ทดสอบ Segmentation Compliance จริง |
| Wave 3 (Regional Hub) | เดือนที่ 7 | 4 Regional Hub | Migrate หลัง Spoke ทั้งหมดในแต่ละ Region สำเร็จแล้ว |
| Wave 4 (HQ/Data Center) | เดือนที่ 8 | HQ (WAN-EDGE-1/2) | Migrate ท้ายสุด ก่อนหมดสัญญา MPLS พอดี (เดือนที่ 8 จาก 8 เดือนที่เหลือ) |
| Decommission MPLS | เดือนที่ 9+ (หลัง Soak Period) | ทั้งหมด | ตัด MPLS Contract เดิมหลัง Soak Period 30-60 วันต่อ Wave ผ่านสำเร็จ |

```
เดือน:     1    2    3    4    5    6    7    8    9
Wave 0    ██
Wave 1         ████████
Wave 2                   ████████████████
Wave 3                                    ██
Wave 4                                         ██
Soak/Decom.                                        ████████
```

### 6) Controller Sizing สำหรับ Design นี้ (อ้างอิง Step 754)

| Controller | จำนวนที่ใช้ | เหตุผล |
|---|---|---|
| vManage | 3-Node Cluster (Primary) + 3-Node Warm Standby ที่ DR Site | 200 cEdge อยู่ในช่วง 100-1,000 ตาม Sizing Table |
| vSmart | 4 ตัว (2 Primary Site + 2 DR Site) กำหนด Region ตาม MRF | รองรับ MRF 4 Region + Core |
| vBond | 2 ตัว (1 Primary + 1 DR, Public-facing) | มาตรฐานขั้นต่ำสำหรับ Production |

### สรุป Design Document Checklist

| องค์ประกอบ | สถานะ Design |
|---|---|
| Topology | Regional-Mesh ผ่าน MRF (4 Region + Core) — ✅ Step 751-752 |
| Application Policy | 3 Transport, 3 SLA Class, Fallback Chain ครบ — ✅ Step 753 |
| Controller HA | Sizing + DR Site ครบตาม Scale 200 cEdge — ✅ Step 754 |
| Cloud Integration | Cloud OnRamp for SaaS (M365) ออกแบบแล้ว, IaaS เตรียมไว้สำหรับอนาคต — ✅ Step 755-756 |
| Security | On-Box Security Stack + Centralized FW สำหรับ POS Compliance — ✅ Step 757 |
| Segmentation | 5 VPN Segment สม่ำเสมอทุกสาขา — ✅ Step 758 |
| Migration Plan | Wave-based 8 เดือน พร้อม Soak Period และ Rollback — ✅ Step 759 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 76

1. ทำไม Full-Mesh Topology ของ SD-WAN จึง "เป็นไปได้จริง" ที่ Scale ใหญ่กว่า DMVPN แต่ก็ยังมี
   เพดานจำกัดอยู่ที่จุดไหน?
2. อธิบายความแตกต่างระหว่าง **Edge Router** กับ **Border Router** ใน Multi-Region Fabric (MRF)
   และ Core Region (Region 0) ทำหน้าที่อะไร?
3. ในตัวอย่าง Policy ของ Step 753 ทำไม VOICE Traffic ถึงไม่มี `lte` อยู่ใน `preferred-color`
   List เลย แต่ CRITICAL-DATA-SLA กลับมี?
4. Cloud OnRamp for SaaS ต่างจาก Application-Aware Routing (AAR) ทั่วไปอย่างไร ในมุมของสิ่งที่
   ถูกวัด (Probe)?
5. ทำไม VPN Numbering ต้องสม่ำเสมอ (Consistent) ทุกสาขาใน Design แบบ Multi-Tenant ที่ Scale ใหญ่?

**เฉลย:**

1. Full-Mesh ของ SD-WAN เป็นไปได้เพราะ **Control Plane (vSmart/OMP) แยกจาก Data Plane อย่าง
   สมบูรณ์** — vSmart แค่แจก TLOC ให้ cEdge รู้จักกันแล้ว cEdge สร้าง Tunnel ตรงถึงกันเอง ไม่มี
   Hub เป็นคอขวดของ Data Plane แบบ DMVPN Hub แต่เพดานจริงอยู่ที่ **จำนวน IPsec Tunnel ต่อ cEdge
   หนึ่งตัว (N-1)** ที่ Platform รองรับได้ — ที่ Scale หลักร้อย Site จำนวน Tunnel ต่ออุปกรณ์จะ
   สูงเกินไปสำหรับ Branch Platform เล็ก จึงต้องใช้ Control Policy จำกัด Topology
2. **Edge Router** คือ cEdge ปกติที่ Branch ซึ่งเห็นเฉพาะ TLOC ภายใน Region ตัวเองและ Border
   Router ของ Region ตัวเองเท่านั้น ส่วน **Border Router** คือ cEdge ที่ทำหน้าที่เป็นประตูเชื่อม
   Region ตัวเองเข้ากับ Core Region (มี `region-id` และ `secondary-region-id` ทั้งสอง) —
   **Core Region (Region 0)** ทำหน้าที่เป็น Transit Backbone ที่ Border Router ของทุก Region
   เชื่อมเข้ามาเพื่อคุยข้าม Region กัน
3. เพราะ LTE มี **Jitter สูงเกินไปสำหรับ Voice Quality** (Jitter สูงทำให้เสียงกระตุกฟังไม่รู้เรื่อง)
   จึงถูกตัดออกจาก Fallback Chain ของ VOICE-SLA เด็ดขาด (ยอม Drop ตาม `strict` ดีกว่าใช้ LTE)
   ส่วน CRITICAL-DATA-SLA (เช่น POS Transaction) ยอมรับ Latency สูงขึ้นได้ในกรณีฉุกเฉิน เพราะ
   Transaction ที่ช้าลงยังดีกว่าระบบหยุดทำงาน จึงมี LTE เป็นตัวเลือกสุดท้ายใน Chain
4. AAR ทั่วไปวัดคุณภาพ **Path (Loss/Latency/Jitter) ระหว่าง cEdge กับ cEdge หรือ TLOC หนึ่งไปอีก
   TLOC** โดยไม่รู้จัก Application ปลายทางจริง ส่วน Cloud OnRamp for SaaS **Probe ตรงไปยัง SaaS
   Endpoint จริง (เช่น Office 365)** เพื่อวัด Response Time ของ Application นั้นโดยเฉพาะ ทำให้
   เลือก Exit Path ที่ดีที่สุด **ต่อแต่ละ SaaS App แยกกัน** ไม่ใช่เลือก Path เดียวสำหรับทุก Traffic
5. เพราะ **Centralized Policy ที่ตั้งจาก vManage ครั้งเดียว Apply กับทุก Site ผ่าน Site List**
   ถ้าเลข VPN เดียวกันมีความหมายต่างกันในแต่ละสาขา (เช่น VPN 100 = POS ที่สาขา A แต่ = Corporate
   ที่สาขา B) Policy ที่อ้างอิงเลข VPN จะ Apply ผิด Segment ทันที ทำให้ Isolation/Compliance ที่
   ตั้งใจไว้ (เช่น PCI-DSS สำหรับ POS) พังทั้ง Fabric โดยไม่รู้ตัว

---

## สรุป Part 76

Part นี้พา SD-WAN จาก Part 39 (สถาปัตยกรรมพื้นฐาน HQ + 1 Branch) ก้าวขึ้นสู่ **Design ระดับ
CCIE Enterprise Infrastructure ที่ Scale ได้จริงหลักร้อยถึงหลักพัน Site** — เริ่มจาก Topology
Design (Hub-Spoke/Full-Mesh/Regional-Mesh) ที่เข้าใจข้อจำกัดจริงของ Tunnel Count, Multi-Region
Fabric (MRF) ที่แบ่ง Fabric ใหญ่ด้วย Border Router/Core Region, Application-Aware Routing ขั้นสูง
ที่รวม SLA Class หลายชั้นกับ Fallback Chain ข้าม 3 Transport, High Availability Controller
Design พร้อม Disaster Recovery, Cloud OnRamp for SaaS ที่เลือก Exit ต่อแอปอัตโนมัติ, Cloud OnRamp
for IaaS ที่ขยาย Fabric เข้าสู่ Public Cloud, Security Stack ที่ฝังอยู่ใน cEdge เอง, Segmentation
at Scale แบบ Multi-Tenant สำหรับ Retail, ไปจนถึงกลยุทธ์ Migration แบบ Wave-based สำหรับองค์กร
ขนาดใหญ่ และปิดท้ายด้วย Design Document เต็มรูปแบบสำหรับ Retail 200 สาขา ✅

**พร้อมสำหรับ Part 77**: หลังจากเจาะลึก SD-WAN (WAN) เต็มรูปแบบแล้ว เราจะย้อนกลับไปที่ฝั่ง
**Campus LAN** ที่ [Part 38 — SD-Access Fundamentals](part-038-sd-access-fundamentals.md) ได้
ปูพื้นฐานไว้ (VXLAN/LISP Fabric, DNA Center) — **Part 77 — SD-Access Advanced Design** จะเจาะลึก
Design ระดับ Enterprise ของ SD-Access ในลักษณะเดียวกับที่ Part นี้ทำกับ SD-WAN (Multi-Site
Fabric, Transit ระหว่าง Fabric Site, Policy ขั้นสูง) เพื่อให้ครบทั้งสองฝั่งของ Software-Defined
Enterprise Network ตามแนวคิด SDN ที่หลักสูตรนี้ใช้เป็นแกนกลางตั้งแต่ Part 38-39

**ไปต่อ:** [Part 77 — SD-Access Advanced Design →](part-077-sd-access-advanced-design.md)
