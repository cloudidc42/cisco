# Part 39 — SD-WAN Fundamentals (Cisco Viptela)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 381–390 จาก 1000**

> ต่อจาก [Part 38 — SD-Access Fundamentals](part-038-sd-access-fundamentals.md) ที่นำแนวคิด
> **Software-Defined Networking (SDN)** มาใช้กับ Campus LAN ผ่าน Cisco DNA Center + Fabric
> (VXLAN/LISP) แล้ว Part นี้จะนำแนวคิด SDN แบบเดียวกัน — **แยก Control Plane ออกจาก Data Plane
> และบริหารจัดการแบบรวมศูนย์ (Centralized Management)** — มาใช้กับฝั่ง **WAN**
>
> [Part 19 — WAN Technologies](part-019-wan-technologies.md) ได้แนะนำ **SD-WAN** ไว้เป็น
> ภาพรวมสั้นๆ ในตาราง Step 181 และตั้งใจ "ค้างไว้" สอนแบบเต็มรูปแบบใน Part นี้ (ภาพรวม
> สถาปัตยกรรม) และ Part 76 (เชิงลึกเรื่อง Policy/Cloud OnRamp/Migration ขนาดใหญ่) — เราจะใช้
> อุปกรณ์เดิมจาก Part 19 คือ **WAN-EDGE-1/WAN-EDGE-2** (HQ WAN Edge) และ **BRANCH-RTR**
> (สำนักงานสาขา) เป็นฐานในการอธิบายว่า ถ้าจะแปลง Design แบบ PPP+GRE ของ Part 19 ให้เป็น
> **Cisco Catalyst SD-WAN (เดิมชื่อ Viptela)** จะต้องเปลี่ยนแนวคิดอะไรบ้าง

> **หมายเหตุสำคัญเรื่อง Lab**: SD-WAN ต้องมี **Controller Stack** (vManage/vSmart/vBond) ซึ่ง
> เป็น VM แยกที่ไม่ได้อยู่ใน CLI Lab หลักของหลักสูตรนี้ (ที่เน้น IOS/IOS-XE CLI บน Router/Switch
> ธรรมดา) ดังนั้น Part นี้จะสอนแบบ **Conceptual + Real CLI/Command Reference** โดยอ้างอิงจาก
> Cisco SD-WAN (Viptela) ของจริงทุกคำสั่ง เพื่อให้อ่านเข้าใจสถาปัตยกรรมและเตรียมพร้อมสำหรับ
> การขึ้น Lab จริงบน Cisco Modeling Labs (CML) ที่รองรับ vManage/vSmart/vBond/vEdge Image
> หรือ Cisco dCloud/SD-WAN Sandbox ต่อไป

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 381 | Traditional WAN vs SD-WAN — ทำไมองค์กรต้องเปลี่ยน |
| 382 | สถาปัตยกรรม Cisco SD-WAN (Viptela) — 4 Planes |
| 383 | Control Plane เชิงลึก — OMP (Overlay Management Protocol) |
| 384 | Data Plane — IPsec Overlay Tunnel และแนวคิด TLOC |
| 385 | Zero-Touch Provisioning (ZTP) |
| 386 | Application-Aware Routing (AAR) — SLA Class และตัวอย่าง Policy สำหรับ VOICE |
| 387 | Centralized Policy และ Segmentation (VPN) |
| 388 | Direct Cloud Access / Cloud OnRamp for SaaS (Preview) |
| 389 | Migration จาก Traditional WAN สู่ SD-WAN |
| 390 | Lab เต็มรูปแบบ (Conceptual/Architectural): แปลง WAN-EDGE-1/2 + BRANCH-RTR เป็น SD-WAN Fabric |

---

## Step 381 — Traditional WAN vs SD-WAN: ทำไมองค์กรต้องเปลี่ยน

### ย้อนดู Traditional WAN จาก Part 19

Part 19 สอน WAN สามรูปแบบหลักที่ยังคอนฟิกทีละอุปกรณ์ (Box-by-Box, CLI ต่อ CLI):

| เทคโนโลยีจาก Part 19 | ข้อจำกัดที่ SD-WAN แก้ไข |
|---|---|
| **PPP บน Leased Line** (Step 182-185) | ต้องเช่า Circuit ราคาแพง, เพิ่ม Site ใหม่ต้องรอ SP ติดตั้ง, ไม่มี Application Awareness — Router รู้แค่ Routing Table ไม่รู้ว่า Traffic เป็น VoIP หรือ Web Browsing |
| **MPLS L3VPN (มุมมอง CE)** (Step 186-187) | SLA ดีแต่ราคาสูง, Provisioning ช้า (สัปดาห์-เดือน), Path Selection ทำได้แค่ตาม Routing Metric ไม่ใช่ตามสภาพ Network จริง |
| **GRE Tunnel + แนวคิด IPsec** (Step 188-190) | ต้องคอนฟิก Tunnel คู่ต่อคู่เอง (Point-to-Point), ขยาย Full-Mesh ระหว่างหลาย Site เป็น N×(N-1)/2 Tunnel ที่ต้องดูแลเอง, ไม่มี Central Policy — เปลี่ยน Policy ต้อง login เข้าไปแก้ทุก Router ทีละตัว |

ปัญหาร่วมของทั้ง 3 แบบ: **ทุก Router ตัดสินใจ Routing ด้วยตัวเอง (Distributed Control Plane
แบบเดิม) และวิศวกรต้องคอนฟิกทีละอุปกรณ์ (Box-by-Box Management)** — ยิ่งมี Site มากขึ้นเท่าไหร่
งาน Operation ก็ยิ่งทวีคูณตามจำนวน Site

### แรงผลักดันที่ทำให้เกิด SD-WAN

```
                    เหตุผลที่ธุรกิจต้องการ SD-WAN
┌─────────────────────────────────────────────────────────────────┐
│ 1. แอปพลิเคชันย้ายขึ้น Cloud/SaaS (O365, Salesforce, Zoom)         │
│    → Traffic ไม่ได้วิ่งกลับ Data Center อีกต่อไป                     │
│                                                                     │
│ 2. Internet เร็วขึ้นมาก ราคาถูกลงมาก เมื่อเทียบกับ MPLS              │
│    → Internet เริ่มเป็น "Transport ที่ใช้งานได้จริง" ไม่ใช่แค่ Backup │
│                                                                     │
│ 3. องค์กรมี Site จำนวนมาก (Retail, Branch สาขา) ต้องขยายเร็ว         │
│    → ต้องการ Provisioning แบบ Plug-and-Play ไม่ใช่รอ CLI ทีละที่      │
│                                                                     │
│ 4. Application แต่ละตัวมีความต้องการ Network ต่างกัน                 │
│    → VoIP ต้องการ Jitter ต่ำ, File Transfer ต้องการ Throughput สูง   │
│      แต่ Traditional Routing เลือก Path จาก Metric เท่านั้น          │
│      ไม่รู้จัก "คุณภาพ Path ณ ขณะนี้" เลย                             │
└─────────────────────────────────────────────────────────────────┘
```

### นิยามของ SD-WAN

**SD-WAN (Software-Defined Wide Area Network)** คือสถาปัตยกรรม WAN ที่:

1. **แยก Control Plane ออกจาก Data Plane** — Controller กลางตัดสินใจ Policy/Routing แทนที่จะให้
   แต่ละ Router ตัดสินใจเอง (คล้ายแนวคิด SDN ที่ใช้ใน SD-Access Part 38 แต่ทำที่ WAN)
2. **มองทุก Transport เป็น "Underlay" เดียวกัน** — MPLS, Internet (Broadband), 4G/5G ถูกมองรวม
   เป็น Transport ที่ Router ปลายทั้งสองฝั่งสร้าง **Encrypted Overlay Tunnel** ทับไปโดยอัตโนมัติ
   ไม่ต้องคอนฟิก Tunnel เองแบบ GRE ใน Part 19 อีกต่อไป
3. **Application-Aware Routing** — วัดคุณภาพ Path (Loss/Latency/Jitter) แบบ Real-time ต่อ Tunnel
   แล้วเลือก Path ที่เหมาะกับ Application แต่ละตัวโดยอัตโนมัติ
4. **Centralized Policy** — ตั้ง Policy ครั้งเดียวที่ Controller แล้ว Push ไปทุก Site พร้อมกัน
5. **Zero-Touch Provisioning (ZTP)** — Router ตัวใหม่เสียบสายแล้วดึง Config มาเองจาก Controller
   โดยไม่ต้องพิมพ์ CLI เลยแม้แต่บรรทัดเดียว

### ตารางเปรียบเทียบสรุป Traditional WAN vs SD-WAN

| คุณสมบัติ | Traditional WAN (Part 19) | SD-WAN (Cisco Viptela) |
|---|---|---|
| Control Plane | กระจายอยู่ในแต่ละ Router (OSPF/eBGP ต่อ Site) | รวมศูนย์ที่ **vSmart Controller** ผ่าน OMP |
| การเชื่อม Site ใหม่ | คอนฟิก Tunnel/Circuit ทีละคู่ (Point-to-Point) | Full-Mesh/Hub-Spoke Overlay สร้างอัตโนมัติ |
| การเลือก Transport | ตาม Routing Metric คงที่ (Static/OSPF Cost) | ตามคุณภาพ Path Real-time (Loss/Latency/Jitter) |
| Provisioning อุปกรณ์ใหม่ | Login CLI เอง ตั้งค่าทุกบรรทัด | **ZTP** — ดึง Config มาเองจาก Controller |
| การเปลี่ยน Policy ทั้งองค์กร | แก้ทีละ Router (Box-by-Box) | แก้ที่ Controller ครั้งเดียว Push ทุก Site |
| การเข้ารหัส | ต้องคอนฟิก IPsec เพิ่มเอง (Step 189) | **IPsec Overlay เป็น Default** ของทุก Tunnel |
| ตัวอย่างในหลักสูตรนี้ | WAN-EDGE-1/2 + BRANCH-RTR (PPP/GRE) | WAN-EDGE-1/2 + BRANCH-RTR แปลงเป็น **cEdge** |

> **ข้อควรระวังเรื่อง Mindset**: SD-WAN **ไม่ได้แทนที่ MPLS/Internet** — มันคือ **Overlay ที่
> วิ่งทับ Underlay เดิม** MPLS ยังใช้เป็น Transport หนึ่งได้ (เรียกว่า Color `mpls`) เพียงแต่
> SD-WAN ทำให้ Internet กลายเป็น Transport ที่ใช้งานคู่กับ MPLS ได้อย่างมีประสิทธิภาพและปลอดภัย
> เท่าเทียมกันเป็นครั้งแรก — นี่คือสิ่งที่ Step 181 ของ Part 19 พูดถึงไว้ว่า "SD-WAN คือ Overlay
> ที่รวมหลาย Underlay (MPLS+Internet+4G/5G) บริหารจาก Controller กลาง"

---

## Step 382 — สถาปัตยกรรม Cisco SD-WAN (Viptela): 4 Planes

Cisco ซื้อบริษัท **Viptela** ในปี 2017 และนำสถาปัตยกรรมมาเป็นแกนหลักของผลิตภัณฑ์
**Cisco Catalyst SD-WAN** (ชื่อเดิม: Cisco SD-WAN / Viptela) สถาปัตยกรรมนี้แบ่งออกเป็น
**4 Plane** ที่ทำงานแยกกันอย่างชัดเจน

### ภาพรวม 4 Planes

| Plane | ชื่อ Component | หน้าที่หลัก | เปรียบเทียบใกล้เคียงจาก Part ก่อนหน้า |
|---|---|---|---|
| **Orchestration Plane** | **vBond** | จุดแรกที่อุปกรณ์ใหม่ติดต่อเข้ามา, ยืนยันตัวตน (Authentication) และ "แนะนำทาง" ให้ไปคุยกับ vManage/vSmart | คล้าย DHCP/ZTP Server ที่ Part 16 พูดถึง แต่ทำหน้าที่ Bootstrap ทั้ง Fabric |
| **Management Plane** | **vManage** | GUI/API สำหรับ Configuration, Monitoring, Template, Policy ทั้งหมด | คล้าย DNA Center ใน Part 38 (Centralized Management) |
| **Control Plane** | **vSmart** | แจก Routing Information (OMP) และ Centralized Policy ให้ทุก Edge Router | คล้าย Route Reflector ของ BGP (Part 30) แต่ทำหน้าที่มากกว่านั้น |
| **Data Plane** | **vEdge / cEdge** | Router ที่ Site จริง — รับ Traffic ผู้ใช้, สร้าง IPsec Tunnel, Forward ตาม Policy ที่ได้รับ | คือ WAN-EDGE-1/2 และ BRANCH-RTR ของเราเอง |

> **vEdge vs cEdge**: **vEdge** คือ Hardware/Software Router สายพันธุ์ Viptela เดิม (ใช้ ViptelaOS)
> ส่วน **cEdge** คือ Router ตระกูล Cisco เดิม (ISR4000, ASR1000, CSR1000v/Catalyst 8000V) ที่รัน
> **Cisco IOS-XE ในโหมด SD-WAN** (`controller-mode` / `sdwan`) ทั้งสองแบบเข้าร่วม Fabric เดียวกัน
> ได้ — Part นี้จะใช้ **cEdge** เป็นหลักเพราะ WAN-EDGE-1/2/BRANCH-RTR เป็น IOS-XE Router (ISR4451)
> อยู่แล้วตาม [00-ip-address-plan.md](00-ip-address-plan.md) เพียงแค่เปลี่ยน Mode การทำงาน

### Diagram สถาปัตยกรรมเต็มรูปแบบ

```
                         ┌───────────────────────────────────────────┐
                         │             ORCHESTRATION PLANE              │
                         │                  vBond                       │
                         │   (Authenticate + Redirect เท่านั้น          │
                         │    ไม่เก็บ Routing/Policy ถาวร)               │
                         └───────────────────┬───────────────────────┘
                                             │ DTLS/TLS (แนะนำทาง)
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
┌─────────────┴─────────────┐   ┌────────────┴────────────┐   ┌─────────────┴─────────────┐
│      MANAGEMENT PLANE       │   │        CONTROL PLANE      │   │      (Redundant vSmart)    │
│          vManage             │◄─►│          vSmart           │   │           vSmart-2          │
│  - Template/Policy GUI/API   │   │  - OMP Route Reflector    │   │                            │
│  - Monitoring/Alarm/Software │   │  - Centralized Policy     │   │                            │
│    Upgrade                   │   │    Distribution            │   │                            │
└─────────────┬─────────────┘   └────────────┬────────────┘   └─────────────┬─────────────┘
              │ NETCONF/REST (Config Push)     │ OMP over DTLS/TLS (Control Connection)         │
              └──────────────────┬──────────────┴──────────────────────────────────────────────┘
                                 │
      ┌──────────────────────────┼──────────────────────────────────┐
      │                          │                                  │
┌─────┴──────┐           ┌───────┴───────┐                 ┌────────┴───────┐
│  cEdge      │◄════════►│  cEdge         │◄═══════════════►│  cEdge          │
│ WAN-EDGE-1  │  IPsec   │  WAN-EDGE-2    │   IPsec Data    │  BRANCH-RTR     │
│ (HQ Site)   │  Overlay │  (HQ Site)     │   Plane Mesh    │  (Branch Site)  │
└─────────────┘           └────────────────┘                 └────────────────┘
        DATA PLANE — Tunnel เข้ารหัส (IPsec) สร้างอัตโนมัติระหว่าง cEdge ทุกตัว
```

### สรุปเส้นทางการเชื่อมต่อ

- **cEdge ↔ vBond**: DTLS/TLS ใช้ครั้งเดียวตอน Bootstrap เพื่อขอรายชื่อ vSmart/vManage (Step 385)
- **cEdge ↔ vSmart**: DTLS/TLS **แบบถาวร (Persistent Control Connection)** ใช้รัน OMP
  (Step 383) — คือเส้นที่สำคัญที่สุดของ Control Plane
- **cEdge ↔ vManage**: NETCONF/REST ใช้รับ Configuration Template และส่ง Monitoring Data กลับ
- **cEdge ↔ cEdge**: IPsec Overlay Tunnel (Data Plane) — สร้างขึ้นเองอัตโนมัติตามที่ vSmart
  แจ้ง TLOC ให้รู้จักกัน (Step 384)

> **หมายเหตุ Design จริง**: vManage/vSmart/vBond มักถูก Deploy เป็น **Cluster สำรอง (Redundant)**
> อย่างน้อย 3 ตัว (Odd Number เพื่อ Quorum แบบเดียวกับหลักการ Election ที่เจอใน STP/HSRP ก่อนหน้า)
> และมักอยู่บน **Cloud (Cisco SD-WAN Cloud-hosted)** หรือ Data Center กลางขององค์กร ไม่ใช่ที่ Site
> สาขา — Controller ล่มไม่ได้ทำให้ Data Plane ที่มีอยู่แล้วหยุดทำงานทันที (Fail-safe: Existing
> Tunnel ยังส่ง Traffic ต่อได้) แต่จะทำให้ Push Policy/Route ใหม่ไม่ได้จนกว่า Controller กลับมา

---

## Step 383 — Control Plane เชิงลึก: OMP (Overlay Management Protocol)

### OMP คืออะไร

**OMP (Overlay Management Protocol)** คือ Routing Protocol ที่ Cisco/Viptela ออกแบบขึ้นมาใหม่
โดยเฉพาะสำหรับ SD-WAN Fabric — **แนวคิดคล้าย BGP มาก** (Path Vector, ใช้ Attribute ตัดสินใจ
Best Path, รองรับ Route Reflector Model) แต่ถูกปรับแต่งให้แจกข้อมูล 3 ประเภทที่ SD-WAN
ต้องการโดยเฉพาะ ซึ่ง BGP ธรรมดาไม่มี Concept เหล่านี้ในตัว

### OMP Route ทั้ง 3 ประเภท

| ประเภท OMP Route | แจกข้อมูลอะไร | เทียบได้กับ (Part ก่อนหน้า) |
|---|---|---|
| **OMP Route (vRoute)** | Prefix ปลายทางของ LAN แต่ละ Site (เช่น `10.10.10.0/24` ของ VLAN 10) | เหมือน BGP Network Prefix ที่ Advertise (Step 187 Part 19) |
| **TLOC Route** | ข้อมูล Transport Locator ของแต่ละ cEdge — บอกว่า "ฉันเข้าถึงได้ทาง Color อะไร ที่ IP ไหน" | ไม่มีเทียบเท่าตรงใน BGP ธรรมดา — คล้าย Next-Hop แต่มีมิติ "Color" เพิ่ม (Step 384) |
| **Service Route** | บอกว่า Site นี้มีบริการอะไรอยู่ (เช่น Firewall, IPS) ให้ Site อื่น Route ผ่านได้ | คล้าย Route Leaking ข้าม VRF |

### Control Connection: DTLS/TLS ระหว่าง cEdge ↔ vSmart

```
cEdge (WAN-EDGE-1)                                        vSmart Controller
      │                                                          │
      │  1. เปิด DTLS (หรือ TLS) Tunnel ไปยัง vSmart ทุกตัวใน List  │
      │ ───────────────────────────────────────────────────────► │
      │  2. Mutual Authentication ด้วย Certificate (PKI)            │
      │ ◄─────────────────────────────────────────────────────── │
      │  3. Control Connection ขึ้น (Persistent, Keepalive ต่อเนื่อง)│
      │ ═══════════════════════════════════════════════════════ │
      │  4. cEdge ส่ง OMP Update: My vRoute + My TLOC              │
      │ ───────────────────────────────────────────────────────► │
      │  5. vSmart คำนวณ Best Path + Policy แล้วส่งกลับทุก cEdge     │
      │ ◄═════════════════════════════════════════════════════ │
```

- ทุก cEdge เปิด Control Connection ไปหา **vSmart ทุกตัว** ใน Cluster (ไม่ใช่แค่ตัวเดียว) เพื่อ
  Redundancy — คล้ายหลักการมี WAN-EDGE-1 **และ** WAN-EDGE-2 ใน Part 19 เพื่อไม่ให้ HQ WAN
  ล่มพร้อมกันทั้งคู่
- **cEdge ไม่คุยกันโดยตรงเพื่อแลก Routing Information** — ทุก cEdge ส่ง OMP Update ให้ vSmart
  เท่านั้น แล้ว vSmart จึงแจกต่อให้ cEdge ตัวอื่นๆ (**Hub-and-Spoke สำหรับ Control Plane**
  แม้ Data Plane จะเป็น Full-Mesh ก็ตาม) — หลักการนี้เหมือน **BGP Route Reflector** ที่ Client
  ไม่ต้อง Full-Mesh กันเองในเรื่อง iBGP (แนวคิดที่กล่าวถึงสั้นๆ ใน Part 30)

### คำสั่ง Verify OMP (Cisco IOS-XE SD-WAN CLI)

```
WAN-EDGE-1# show sdwan control connections

                                                    PEER    PEER  SITE
PEER TYPE   PEER PROT  PEER SYSTEM IP   PEER PRIVATE IP   PORT    ID   STATE  UPTIME
vsmart      dtls       10.1.1.10        10.1.1.10         12446   100  up     3:10:22:15
vsmart      dtls       10.1.1.11        10.1.1.11         12446   100  up     3:10:22:12
vmanage     dtls       10.1.1.20        10.1.1.20         12446   100  up     3:10:22:10

WAN-EDGE-1# show sdwan omp peers

          PEER            PEER      LEGIT
PEER      TYPE    SITE    STATE     TYPE
10.1.1.10 vsmart   100     up       vsmart
10.1.1.11 vsmart   100     up       vsmart

WAN-EDGE-1# show sdwan omp routes

           C   -> chosen
           I   -> installed
           R   -> resolved
                                             PATH
PREFIX             FROM PEER   ID     STATUS  TYPE     TLOC IP
10.10.10.0/24       10.1.1.10   72    C,I,R   installed 1.1.1.22, biz-internet, ipsec
```

- `show sdwan control connections` = สถานะการเชื่อมต่อ Control Plane ไปยัง vSmart/vManage
  ทุกตัว (เทียบได้กับ `show ip bgp summary` ใน Part 19/29 ที่ดู Neighbor State)
- `show sdwan omp peers` = OMP Neighbor ที่เป็น vSmart โดยเฉพาะ
- `show sdwan omp routes` = Routing Table ที่ได้รับผ่าน OMP — สังเกตว่ามีคอลัมน์ **TLOC** ติดมา
  ด้วยเสมอ (บอกว่า Route นี้ไปถึงได้ผ่าน TLOC ไหน) ซึ่งเป็นแนวคิดที่ BGP ไม่มี

> **จุดที่ต้องเข้าใจสำหรับข้อสอบ ENCOR**: OMP ไม่ใช่ BGP และไม่ compatible กับ BGP โดยตรง — มันคือ
> Protocol เฉพาะของ SD-WAN Fabric เท่านั้น การ Redistribute Route จาก OSPF/BGP/Static (ที่ Site
> ใช้อยู่แบบ Part 9-14/29-31) เข้าไปใน OMP เพื่อ Advertise ผ่าน Fabric ทำผ่านคำสั่ง `redistribute`
> ภายใต้ Service VPN ของ cEdge เท่านั้น ไม่ใช่การรัน OMP ตรงกับ OSPF/BGP

---

## Step 384 — Data Plane: IPsec Overlay Tunnel และแนวคิด TLOC

### TLOC (Transport Locator) คืออะไร

**TLOC (Transport Locator)** คือ "ที่อยู่" ของ cEdge บนฝั่ง Transport/Underlay แต่ละเส้น
ประกอบด้วย 3 ส่วน:

```
TLOC = (System-IP, Color, Encapsulation)

ตัวอย่าง: (1.1.1.21, biz-internet, ipsec)
            │            │            │
            │            │            └─ วิธีเข้ารหัส Data Plane (ipsec หรือ gre)
            │            └─ "สี" ของ Transport เส้นนี้ (ระบุประเภท Underlay)
            └─ System-IP ของ cEdge ตัวนี้ (เหมือน Router-ID/Loopback0 แบบ Part เก่า)
```

### Color — แนวคิดใหม่ที่ไม่มีใน Traditional WAN

**Color** คือ Label ที่ติดให้แต่ละ Transport Interface เพื่อบอกว่า "นี่คือ Underlay ประเภทไหน"
Cisco กำหนด Color มาตรฐานไว้ล่วงหน้าให้เลือกใช้:

| กลุ่ม Color | ตัวอย่างชื่อ | ความหมาย |
|---|---|---|
| Private (เชื่อถือได้, ไม่ผ่าน NAT) | `mpls`, `private1`-`private6` | Transport แบบ MPLS L3VPN/Leased Line จาก Part 19 |
| Public (ผ่าน Internet, มักผ่าน NAT) | `biz-internet`, `public-internet`, `custom1` | Internet Broadband ธรรมดา |
| Cellular | `lte` | 4G/5G สำหรับ Site ที่ไม่มี Fixed Line หรือใช้เป็น Backup |

Color เป็นตัวกำหนดว่า cEdge ควรสร้าง Overlay Tunnel ไปยัง Color เดียวกันหรือต่าง Color
ของ cEdge อีกฝั่งหรือไม่ (ควบคุมผ่าน Policy ได้ — เช่นห้าม `lte`↔`lte` เพื่อประหยัดค่า Cellular
Data) และเป็นข้อมูลสำคัญที่ **Application-Aware Routing** (Step 386) ใช้ตัดสินใจว่าจะส่ง
Traffic ประเภทไหนไป Path ที่มี Color อะไร

### การสร้าง IPsec Overlay Tunnel อัตโนมัติ

```
                     ก่อน SD-WAN (Part 19 GRE — คอนฟิกเอง คู่ต่อคู่)
WAN-EDGE-1 ──── ต้องพิมพ์ tunnel source/destination เอง ────► BRANCH-RTR
                (ขยาย Site ใหม่ = คอนฟิก Tunnel ใหม่ทุกคู่)


                     กับ SD-WAN (Cisco Viptela — Auto Full-Mesh)
                              vSmart แจก TLOC ให้ทุก cEdge รู้จักกัน
                    ┌─────────────────────────────────────────┐
                    │   vSmart: "WAN-EDGE-1 มี TLOC (biz-internet)  │
                    │   BRANCH-RTR มี TLOC (biz-internet) ด้วย"     │
                    └─────────────────────────────────────────┘
                                        │
       WAN-EDGE-1 ◄══════ IPsec Tunnel สร้างอัตโนมัติ ══════► BRANCH-RTR
       WAN-EDGE-2 ◄══════ IPsec Tunnel สร้างอัตโนมัติ ══════► BRANCH-RTR
       WAN-EDGE-1 ◄══════ IPsec Tunnel สร้างอัตโนมัติ ══════► WAN-EDGE-2
              (เพิ่ม Site 4 ไม่ต้องคอนฟิก Tunnel เพิ่มเองสักบรรทัด)
```

เมื่อ vSmart แจก TLOC Route ให้ทุก cEdge รู้จักกันหมดแล้ว **cEdge แต่ละตัวจะสร้าง IPsec Tunnel
ไปยัง cEdge ตัวอื่นเองโดยอัตโนมัติ** ตาม Topology ที่ Policy กำหนด (ดูตัวเลือกด้านล่าง)
โดยใช้ **IKEv2 + ESP** อยู่ภายใต้ Concept เดียวกันกับ IPsec ที่ Part 19 Step 189 อธิบายไว้
(ISAKMP/IKE Phase 1-2, Transform Set) เพียงแต่ SD-WAN **จัดการ Key Exchange และ Rekey ให้
อัตโนมัติทั้งหมด** ไม่ต้องตั้ง Pre-shared Key เองทีละคู่

### Topology ของ Data Plane: Full-Mesh vs Hub-and-Spoke

| Topology | ลักษณะ | เหมาะกับ |
|---|---|---|
| **Full-Mesh** | ทุก cEdge สร้าง Tunnel ถึงกันหมด (Default Behavior) | องค์กรที่ต้องการ Site-to-Site โดยตรง (เช่น Branch คุยกันเอง ไม่ต้องผ่าน HQ) |
| **Hub-and-Spoke** | Spoke สร้าง Tunnel ไปยัง Hub เท่านั้น (ควบคุมด้วย Control Policy) | องค์กรที่ต้องการให้ Traffic ทุกอย่างผ่าน Firewall/Security ที่ HQ ก่อน (คล้าย FW-1 ใน Part 43-45) |

> ในตัวอย่าง HQ + Branch ของเรา ถ้าใช้ Full-Mesh เริ่มต้น จะได้ Tunnel ระหว่าง WAN-EDGE-1↔
> WAN-EDGE-2 (Redundancy ภายใน HQ), WAN-EDGE-1↔BRANCH-RTR, และ WAN-EDGE-2↔BRANCH-RTR — คือ
> Path สำรองที่ Part 19 ต้องคอนฟิก GRE Tunnel สำรองเอง (Step 188-190) กลับกลายเป็นสิ่งที่
> SD-WAN สร้างให้ **อัตโนมัติทั้ง 3 เส้น** โดยที่วิศวกรไม่ต้องพิมพ์คำสั่ง `tunnel` เลยแม้แต่ตัวเดียว

### คำสั่ง Verify Data Plane

```
WAN-EDGE-1# show sdwan bfd sessions

SYSTEM IP        SITE ID  STATE  SOURCE TLOC COLOR   DEST TLOC COLOR   SRC IP         DEST IP        ENCAP  DETECT MULTIPLIER
172.20.0.1       200      up     biz-internet         biz-internet      203.0.113.2    198.51.100.2   ipsec  7          300
1.1.1.22         100      up     biz-internet         biz-internet      203.0.113.2    203.0.113.6    ipsec  7          300

WAN-EDGE-1# show sdwan tunnel-statistics

TUNNEL PROTOCOL SOURCE IP    DEST IP        SOURCE PORT DEST PORT  TX PKTS  RX PKTS  TX OCTETS  RX OCTETS
ipsec    203.0.113.2  198.51.100.2   12406       12406      184203   183991   ...
```

- `show sdwan bfd sessions` = คำสั่งที่ใช้บ่อยที่สุดในการ Verify Data Plane — **BFD (Bidirectional
  Forwarding Detection)** วิ่งอยู่บนทุก IPsec Tunnel ตลอดเวลาเพื่อเช็คว่า Tunnel ยัง Up อยู่ไหม
  **และเก็บค่า Loss/Latency/Jitter ไปให้ Application-Aware Routing ใช้ต่อ** (Step 386)
- Session ที่เห็นในตัวอย่าง คือ Tunnel ระหว่าง WAN-EDGE-1 กับ BRANCH-RTR (System IP 172.20.0.1)
  และ WAN-EDGE-1 กับ WAN-EDGE-2 (System IP 1.1.1.22) — ตรงตาม Full-Mesh Topology ด้านบน

---

## Step 385 — Zero-Touch Provisioning (ZTP)

### ปัญหาที่ ZTP แก้ไข

Part 19 ทุก Step ต้อง Console เข้าไปที่ Router เองแล้วพิมพ์ CLI ทีละบรรทัด (`interface`,
`ip address`, `encapsulation ppp`, ...) — ถ้าองค์กรมี Branch สาขาใหม่ 200 แห่งที่เปิดพร้อมกัน
การส่งวิศวกรเครือข่ายไป Config เองทุกที่เป็นไปไม่ได้ในทางปฏิบัติ **ZTP (Zero-Touch Provisioning)**
แก้ปัญหานี้ด้วยการให้ผู้ที่ Site (อาจเป็นพนักงานทั่วไปที่ไม่มีความรู้ Networking) แค่
**เสียบสายไฟและสาย WAN เข้า Router ใหม่แล้วเปิดเครื่อง** ส่วนที่เหลือ Router ทำเอง

### ขั้นตอน ZTP ของ cEdge (แปลง BRANCH-RTR เป็น cEdge ใหม่)

```
   BRANCH-RTR (cEdge เครื่องใหม่ เสียบสาย WAN แล้วเปิดเครื่องครั้งแรก ยังไม่มี Config เลย)
        │
        │ 1. บูตขึ้นมาด้วย Factory-Default (โหมด sdwan, ไม่มี IP Address ใดๆ)
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 2. ขอ IP ผ่าน DHCP บน WAN Interface (จาก ISP/Internet Router) │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 3. Resolve DNS: ztp.viptela.com (ค่า Default ของโรงงาน)      │
   │    หรือ ZTP Server ที่องค์กรกำหนดเองผ่าน DHCP Option 43      │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 4. ต่อไปยัง ZTP Server → ได้รับ IP Address ของ vBond กลับมา  │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 5. เปิด DTLS Tunnel ไปยัง vBond                              │
   │    ยืนยันตัวตนด้วย Serial Number + Chassis ID (ฝังจากโรงงาน)  │
   │    และ Root CA Certificate ที่ทั้งสองฝั่งมีอยู่แล้ว              │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 6. vBond ตรวจสอบผ่าน vManage ว่า Serial Number นี้ถูก           │
   │    "Whitelist" ไว้แล้วหรือยัง (เจ้าหน้าที่เพิ่มไว้ล่วงหน้าใน       │
   │    vManage — Add Device)                                    │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 7. vBond ส่งรายชื่อ vSmart + vManage กลับให้ BRANCH-RTR         │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
   ┌─────────────────────────────────────────────────────────┐
   │ 8. BRANCH-RTR เปิด Control Connection ไปยัง vManage            │
   │    → vManage Push "Device Template" ที่เตรียมไว้ล่วงหน้า        │
   │    (Interface, VPN, Routing, Policy ทั้งหมด) ลงมาให้ทันที        │
   └─────────────────────────────────────────────────────────┘
        │
        ▼
     BRANCH-RTR พร้อมใช้งานสมบูรณ์ — ไม่มีใครพิมพ์ CLI สักบรรทัดที่ Site เลย
```

### สิ่งที่ต้องเตรียมไว้ล่วงหน้าในวันติดตั้งจริง

| ขั้นตอนเตรียมการ (ทำที่ HQ/NOC ล่วงหน้า) | รายละเอียด |
|---|---|
| เพิ่ม Serial Number ของ BRANCH-RTR ใน vManage | เจ้าหน้าที่ Network ทราบ Serial Number จาก Purchase Order แล้วเพิ่มลงใน "Configuration > Certificates > WAN Edge List" ก่อนเครื่องถูกส่งไป Site |
| สร้าง Device Template ใน vManage | เตรียม Template Config ทั้งหมดล่วงหน้า (VPN Segment, Interface Color, Policy ที่จะผูกกับ Site นี้) |
| กำหนด DHCP Option 43 (ถ้าไม่ใช้ Cloud ZTP ของ Cisco) | บอก cEdge ว่า vBond องค์กรอยู่ที่ไหน กรณีใช้ Controller แบบ On-Premises |
| ส่งเครื่อง + คู่มือเสียบสาย ไปให้ผู้ใช้ปลายทาง | ไม่ต้องมีวิศวกร Network เดินทางไป Site เลย |

### เทียบกับสิ่งที่ต้องทำเองใน Part 19

Part 19 Step 183: BRANCH-RTR ต้อง Console เข้าไปพิมพ์ `interface Serial0/0/0`, `ip address`,
`encapsulation ppp` เอง ทีละบรรทัด — ใน SD-WAN ขั้นตอนทั้งหมดนี้ (รวมถึง `ppp authentication chap`,
`tunnel source/destination` ของ GRE ใน Step 188) ถูกแทนที่ด้วย **Device Template หนึ่งชุด** ที่
เตรียมไว้ล่วงหน้าและ Push ผ่าน ZTP ให้ทุก Branch ที่มีลักษณะเดียวกันโดยอัตโนมัติ

---

## Step 386 — Application-Aware Routing (AAR)

### แนวคิด: วัดคุณภาพ Path จริง แล้วเลือกให้ตรงกับความต้องการของ Application

Traditional Routing (OSPF/EIGRP/BGP จาก Part 11-14/29-30) เลือก Path จาก **Metric คงที่**
(Cost, Bandwidth ที่ตั้งไว้ตอน Config) — ถ้า Path นั้นเริ่มมี Packet Loss หรือ Jitter สูงขึ้นจริง
ระหว่างใช้งาน Routing Protocol ธรรมดา **จะไม่รู้เลยและยังส่ง Traffic ไปทางเดิม**

**Application-Aware Routing (AAR)** ของ SD-WAN แก้ปัญหานี้โดยให้ BFD (จาก Step 384) วัดค่า
**Loss, Latency, Jitter แบบ Real-time ทุกวินาที** บนทุก Tunnel แล้วนำค่าที่วัดได้มาเทียบกับ
**SLA Class** ที่ Policy กำหนดไว้สำหรับ Application แต่ละประเภท

### SLA Class — นิยาม "คุณภาพที่ยอมรับได้" ต่อ Application

| SLA Class ตัวอย่าง | Loss สูงสุด | Latency สูงสุด | Jitter สูงสุด | ใช้กับ Application |
|---|---|---|---|---|
| **VOICE-SLA** | 1% | 100 ms | 20 ms | VoIP, Video Conference (VLAN 20 จาก [00-ip-address-plan.md](00-ip-address-plan.md)) |
| **CRITICAL-DATA-SLA** | 2% | 150 ms | 30 ms | ERP, Database Replication |
| **BEST-EFFORT-SLA** | 5% | 300 ms | 50 ms | Web Browsing, Email ทั่วไป |

### ขั้นตอนการทำงานของ AAR

```
         ทุก Tunnel ระหว่าง cEdge ถูกวัดคุณภาพต่อเนื่องด้วย BFD
   ┌─────────────────────────────────────────────────────────────┐
   │  Path 1 (ผ่าน mpls color):     Loss 0.1%  Latency 40ms  Jitter 5ms │
   │  Path 2 (ผ่าน biz-internet):   Loss 0.5%  Latency 60ms  Jitter 15ms│
   │  Path 3 (ผ่าน lte, backup):    Loss 3%    Latency 180ms Jitter 45ms│
   └─────────────────────────────────────────────────────────────┘
                             │
                             ▼
        Traffic ที่ match "VOICE-SLA" (ต้อง Jitter ≤ 20ms, Loss ≤ 1%)
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Path 1 ✅       Path 2 ✅       Path 3 ❌ (Jitter 45ms เกิน)
              │              │
              └──────┬───────┘
                     ▼
     เลือก Path ที่ "ดีที่สุด" ในกลุ่มที่ผ่าน SLA (ปกติเลือก Path 1 เพราะ Jitter ต่ำสุด)
     ถ้า Path 1 คุณภาพตกลงจน Jitter > 20ms ระหว่างใช้งาน → สลับไป Path 2 อัตโนมัติทันที
     (Sub-second Failover โดยไม่ต้องรอ Routing Protocol Reconverge แบบ Part 11-14)
```

### ตัวอย่าง Centralized Policy สำหรับ VOICE VLAN (Conceptual vSmart Policy)

องค์กรต้องการให้ Traffic จาก **VLAN 20 (VOICE, `10.10.20.0/24`)** ที่ Mark เป็น
**DSCP EF (Expedited Forwarding)** ตั้งแต่ต้นทาง (ตามหลัก QoS Marking ที่จะเรียนเต็มรูปแบบใน
[Part 40](part-040-advanced-qos.md)) ให้เดินทางผ่าน Path ที่ Jitter ต่ำที่สุดเสมอ:

```
! ตัวอย่าง Centralized Data Policy + App-Route Policy (สร้างที่ vManage, Push ไปทุก cEdge ครั้งเดียว)

sla-class VOICE-SLA
  latency        100
  loss           1
  jitter         20

app-route-policy VOICE-APP-AWARE-POLICY
  vpn-list       SERVICE-VPN-LIST
  sequence 10
    match
      dscp 46                      ! DSCP EF (Voice) ตาม RFC 4594
    action
      sla-class VOICE-SLA strict   ! ต้องผ่าน SLA เท่านั้น ถ้าไม่มี Path ผ่านเลยให้ Drop
      preferred-color mpls         ! ถ้ามีหลาย Path ผ่าน SLA เท่ากัน ให้เลือก mpls ก่อน
  default-action allow

! Apply Policy เข้ากับ Site List ของทั้ง HQ และ Branch ในคำสั่งเดียว
apply-policy site-list ALL-SITES app-route-policy VOICE-APP-AWARE-POLICY
```

- `strict` = ถ้าไม่มี Path ไหนผ่าน SLA Class เลย ให้ **Drop Traffic นั้นทันที** ดีกว่าส่งผ่าน
  Path ที่คุณภาพแย่แล้วทำให้คุณภาพเสียงพัง (เทียบกับ Best-Effort ที่ยังส่งต่อไปแม้คุณภาพไม่ดี)
- `preferred-color mpls` = Tie-breaker เมื่อมีหลาย Path ผ่าน SLA เท่ากัน — สั่งให้เลือก MPLS
  ก่อน Internet เสมอ (เพราะ MPLS มักเสถียรกว่าในระยะยาวแม้ SLA วัดได้ใกล้เคียงกัน ณ ขณะนั้น)
- Policy นี้ Apply ที่ **Site List** เดียว ครอบคลุมทั้ง WAN-EDGE-1/2 และ BRANCH-RTR พร้อมกัน —
  ตรงข้ามกับ Part 19 ที่ต้องคอนฟิก Static Route AD หรือ PBR (Part 32) แยกทีละ Router เอง

### คำสั่ง Verify AAR

```
WAN-EDGE-1# show sdwan app-route stats

TLOC COLOR      REMOTE COLOR    MEAN LOSS   MEAN LATENCY  MEAN JITTER  SLA CLASS
mpls            mpls            0           42            4           VOICE-SLA (PASS)
biz-internet    biz-internet    1           58            14          VOICE-SLA (PASS)

WAN-EDGE-1# show sdwan policy app-route-policy-filter
```

`show sdwan app-route stats` คือคำสั่งที่วิศวกรใช้ Troubleshoot คุณภาพเสียงบ่อยที่สุดในงานจริง
— ถ้าผู้ใช้ Complain ว่าเสียง VoIP กระตุก คำสั่งนี้จะบอกทันทีว่า Path ที่ถูกเลือกอยู่ ณ ขณะนั้น
มีค่า Loss/Latency/Jitter เท่าไหร่ และยังผ่าน SLA Class ที่กำหนดไว้หรือไม่

---

## Step 387 — Centralized Policy และ Segmentation (VPN)

### VPN Segment: แนวคิดคล้าย VRF แต่บริหารจากศูนย์กลาง

**VPN (ใน SD-WAN Context)** คือการแบ่ง Routing/Forwarding Table ออกเป็นหลายชุดแยกกันบน cEdge
ตัวเดียว — **แนวคิดเดียวกับ VRF (Virtual Routing and Forwarding)** ที่ Part 19 Step 186 พูดถึง
ในมุมของ SP (MPLS L3VPN) เพียงแต่ SD-WAN นำแนวคิดเดียวกันมาใช้ที่ฝั่ง Enterprise เองและทำให้
บริหารจัดการจากศูนย์กลางได้ง่ายกว่ามาก

| VPN Number | ความหมายมาตรฐานของ Cisco SD-WAN |
|---|---|
| **VPN 0** | **Transport VPN** — เก็บเฉพาะ Interface ที่ต่อไปยัง Underlay (MPLS/Internet/LTE) และ TLOC ทั้งหมด ไม่มี LAN User อยู่ในนี้ |
| **VPN 512** | **Management VPN** — สำหรับ Out-of-Band Management (SSH/NETCONF ไปยัง vManage) แยกจาก Traffic User โดยสิ้นเชิง |
| **VPN 1-511** (กำหนดเองได้) | **Service VPN** — เก็บ LAN Subnet ของผู้ใช้จริง (แทน VLAN ที่แต่ละ Site มี) — สามารถสร้างได้หลาย VPN เพื่อแยก Tenant/Department |

### Mapping VLAN เดิมจาก IP Address Plan เข้า Service VPN

```
                     Traditional Design (Part 1-38): แยกด้วย VLAN + SVI
   VLAN 10 (SALES) ─┐
   VLAN 20 (VOICE) ─┼─── แยกกันด้วย VLAN Tag บน Switch, Routing รวมกันที่ตาราง Global เดียว
   VLAN 30 (SERVERS)─┤    (Inter-VLAN Routing ผ่าน SVI ปกติ ตาม Part 10)
   VLAN 40 (WIFI)  ──┘

                     SD-WAN Design: แยกด้วย Service VPN ที่ cEdge
   VPN 10 (SALES)   ─┐
   VPN 20 (VOICE)   ─┼─── แต่ละ VPN มี Routing Table แยกกันจริงๆ (คนละ RIB/FIB)
   VPN 30 (SERVERS) ─┤    ต้องมี Policy ชัดเจนถ้าต้องการให้ VPN คุยข้ามกันได้ (Route Leaking)
   VPN 40 (WIFI)    ─┘
```

> **จุดสำคัญที่ต่างจาก VLAN**: VLAN บน Switch (Part 3-4) แยกกันที่ Layer 2 แต่ Routing (SVI)
> มักอยู่ใน Global Routing Table เดียวกัน (เว้นแต่ทำ VRF-Lite เพิ่ม) — **Service VPN ของ
> SD-WAN แยก Routing Table จริงจังตั้งแต่ต้น** ทำให้ Multi-Tenancy (เช่น แยก Traffic ของ
> บริษัทลูกที่ใช้ Infrastructure ร่วมกัน, หรือแยก Guest Network จาก Corporate อย่างเด็ดขาด)
> ทำได้ง่ายและปลอดภัยกว่ามาก โดยไม่ต้องเพิ่ม VRF-Lite/MPLS L3VPN เองแบบที่ Part 19 อธิบาย

### ตัวอย่าง Centralized Data Policy: ควบคุม Security/Traffic-Engineering ทั้ง Fabric ในครั้งเดียว

เปรียบเทียบกับ Part 19 ที่การทำ Traffic Engineering ต้องตั้ง Static Route AD (Step 190) หรือ
PBR (Part 32) **ทีละ Router** — Centralized Data Policy ของ SD-WAN ทำสิ่งเดียวกันได้จาก
vManage ครั้งเดียวแล้ว Push ไปทุก Site:

```
! ตัวอย่าง: ห้าม VPN 40 (WIFI/Guest) เข้าถึง VPN 30 (SERVERS) โดยตรง ทั้ง Fabric
! (Security Segmentation แบบเดียวกับที่ Firewall Zone ทำ แต่ตั้งที่ Controller ครั้งเดียว)

policy
  data-policy GUEST-ISOLATION-POLICY
    vpn-list ALL-SERVICE-VPNS
    sequence 10
      match
        source-data-prefix-list WIFI-VPN40-PREFIX
        destination-data-prefix-list SERVERS-VPN30-PREFIX
      action drop
    default-action allow
!
apply-policy site-list ALL-SITES data-policy GUEST-ISOLATION-POLICY
```

Policy หนึ่งชุดนี้มีผลกับ **ทุก cEdge ใน Fabric ทันที** (WAN-EDGE-1, WAN-EDGE-2, BRANCH-RTR
และ Site อื่นที่จะเพิ่มเข้ามาในอนาคต) — ต่างจาก Part 19 ที่ต้อง Login เข้าไปตั้ง ACL/PBR เอง
ทีละอุปกรณ์ทุกครั้งที่ต้องการเปลี่ยน Policy Security ขององค์กร

---

## Step 388 — Direct Cloud Access / Cloud OnRamp for SaaS (Preview)

### ปัญหา Hairpin Traffic ของ Traditional WAN

Traditional Design (รวมถึง Design ของ Part 19) มักบังคับให้ Traffic จาก Branch **วิ่งกลับไปออก
Internet ที่ HQ เสมอ** (เพราะ Internet Breakout/Firewall อยู่ที่ HQ เท่านั้น) แม้ปลายทางจริงจะ
เป็น Cloud/SaaS (เช่น Office 365, Salesforce) ที่ไม่มีส่วนเกี่ยวข้องกับ HQ เลยก็ตาม:

```
                     ปัญหา Hairpin: Branch → HQ → Internet → SaaS (อ้อมไกล ช้า)

  BRANCH-RTR ──── GRE/Leased Line ────► WAN-EDGE-1 (HQ) ──── Internet ────► Office 365
     (Branch)                              (ต้องอ้อมผ่าน HQ ก่อนเสมอ)         (Cloud)
```

### แนวทางของ SD-WAN: Direct Cloud Access (DIA/DCA)

**Direct Internet Access (DIA)** และ **Cloud OnRamp for SaaS** ให้ cEdge ที่ Branch
**Breakout Internet ออกไปยัง SaaS ได้ตรงจาก Site ตัวเอง** โดยไม่ต้องอ้อมผ่าน HQ:

```
                     ด้วย SD-WAN: Branch ออก Internet ตรงสำหรับ SaaS Traffic

  BRANCH-RTR ──── Internet (ตรงจาก Site) ────► Office 365 (Cloud)
     (Branch)     Latency ต่ำกว่า, ไม่กิน Bandwidth ของ HQ Internet Circuit

  BRANCH-RTR ──── GRE/SD-WAN Overlay ────► WAN-EDGE-1 (HQ) ──── Internal App เท่านั้น
     (Traffic ที่ต้องเข้า Data Center จริงๆ ยังวิ่งผ่าน Overlay ตามปกติ)
```

Cisco SD-WAN ทำสิ่งนี้ผ่าน **Cloud OnRamp for SaaS** ที่วัด Performance ของหลาย Path
(Direct Internet ที่ Site vs ผ่าน Overlay ไปออกที่ HQ) ไปยัง SaaS Endpoint จริงแบบ Real-time
(คล้ายหลักการ AAR ใน Step 386 แต่วัดไปที่ Public Cloud Endpoint โดยเฉพาะ) แล้วเลือก Path ที่ดี
ที่สุดให้ Traffic ประเภท SaaS โดยอัตโนมัติ — ใช้หลักการวัดคุณภาพเดียวกันกับที่ Step 386
อธิบายไว้ เพียงแต่ปลายทางเป็น Public Cloud ไม่ใช่ cEdge อีกตัวในองค์กรเดียวกัน

> **Preview เท่านั้น**: Part นี้แนะนำแนวคิดพื้นฐานของ Direct Cloud Access เพื่อให้เข้าใจว่าทำไม
> SD-WAN จึงสำคัญกับองค์กรที่ใช้ SaaS มาก — รายละเอียดการคอนฟิก **Cloud OnRamp for SaaS**,
> **Cloud OnRamp for IaaS** (เชื่อมต่อ AWS/Azure/GCP โดยตรง), และ **Cloud OnRamp for
> Multi-Cloud** แบบเต็มรูปแบบจะเรียนใน **[Part 93]** ของหลักสูตร (ช่วง Cloud Networking)

---

## Step 389 — Migration จาก Traditional WAN สู่ SD-WAN

### กลยุทธ์ Coexistence — ไม่ต้อง "รื้อทั้งหมดแล้วเปลี่ยนพร้อมกัน"

องค์กรจริงไม่สามารถปิด WAN ทั้ง Fabric แล้วเปลี่ยนเป็น SD-WAN ในคืนเดียวได้ (Business ต้องทำงาน
ต่อเนื่อง) จึงต้องใช้กลยุทธ์ **Coexistence** — ให้ Traditional WAN และ SD-WAN Overlay ทำงาน
คู่กันไปก่อนระหว่าง Migration:

| กลยุทธ์ | รายละเอียด | เหมาะกับ |
|---|---|---|
| **Site-by-Site Migration** | แปลง Site ทีละแห่ง เริ่มจาก Site เสี่ยงต่ำ (เช่น Branch เล็กที่ไม่ Critical) ก่อน HQ | องค์กรส่วนใหญ่ — ปลอดภัยที่สุด |
| **Parallel Circuit (Dual-Run)** | ติดตั้ง cEdge เพิ่มเข้าไปคู่กับ Router เดิม ให้ SD-WAN Overlay วิ่งบน Link ใหม่ก่อน แล้วค่อย Cutover Traffic | Site สำคัญที่ Downtime ไม่ได้เลย |
| **Router เดิมทำ SD-WAN ได้ (Reimage)** | ถ้า Router เดิมเป็นรุ่นที่รองรับ (เช่น ISR4451 ที่ WAN-EDGE-1/2/BRANCH-RTR ใช้อยู่) สามารถเปลี่ยน Mode จาก Autonomous (IOS-XE ปกติ) เป็น **Controller-Mode (SD-WAN)** ได้โดยไม่ต้องเปลี่ยน Hardware | ประหยัดงบ Hardware — ใช้ของเดิมที่มีอยู่ |

### แปลง Design จาก Part 19 (PPP+GRE) เป็น SD-WAN Fabric — แนวคิดทีละขั้น

```
ขั้นที่ 1: Design เดิมจาก Part 19 (Step 190)
   WAN-EDGE-1 ──PPP+CHAP (Serial, Primary)──► BRANCH-RTR
   WAN-EDGE-1 ──GRE Tunnel (Internet, Backup)──► BRANCH-RTR
   (Path Selection ทำผ่าน Administrative Distance คงที่)

ขั้นที่ 2: Reimage Router เข้าสู่ SD-WAN Controller-Mode
   WAN-EDGE-1(config)# controller-mode enable   ! ย้ายจาก Autonomous สู่ Controller-Mode
   (ต้อง Reload และจะไม่รับ IOS-XE CLI ปกติแบบ Part 1-38 อีก — ต้องคอนฟิกผ่าน vManage เท่านั้น)

ขั้นที่ 3: Onboard เข้า Fabric ผ่าน ZTP (Step 385)
   Serial Interface (เดิมใช้ PPP) → กลายเป็น TLOC Color "private1" (Transport เดิมยังใช้ได้)
   Internet Interface (เดิมใช้ GRE) → กลายเป็น TLOC Color "biz-internet"
   ไม่ต้องคอนฟิก tunnel source/destination เองอีก — cEdge สร้าง IPsec Overlay ให้เอง

ขั้นที่ 4: ย้าย VLAN เดิม (10/20/30/40/99) เข้า Service VPN ที่ตรงกัน
   VLAN 10 (SALES)   → VPN 10   VLAN 20 (VOICE) → VPN 20
   VLAN 30 (SERVERS) → VPN 30   VLAN 40 (WIFI)  → VPN 40

ขั้นที่ 5: ยก Policy เดิม (Static AD, PBR) ขึ้นเป็น Centralized Policy
   Static AD ที่เคยตั้งให้ Leased Line เป็น Primary (Step 190)
   → แทนที่ด้วย preferred-color + SLA Class ผ่าน App-Route Policy (Step 386)
```

### คำสั่ง Verify หลัก Migration เสร็จ (สรุปรวมจาก Step ก่อนหน้า)

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show sdwan control connections` | Control Plane ขึ้นครบกับ vSmart/vManage ทุกตัวหรือยัง |
| `show sdwan omp peers` | OMP Neighbor (vSmart) Up และแลก Route กันสำเร็จหรือยัง |
| `show sdwan bfd sessions` | Data Plane (IPsec Tunnel) ระหว่าง cEdge ทุกคู่ Up และคุณภาพ Path เป็นอย่างไร |
| `show sdwan app-route stats` | Application-Aware Routing เลือก Path ตาม SLA Class ถูกต้องหรือไม่ |
| `show sdwan omp routes vpn 10` | Route ของ Service VPN (เช่น VPN 10 = SALES) ถูกแจกไปยัง Site อื่นครบหรือไม่ |

> **ข้อคิดสำคัญสำหรับวิศวกรที่ทำ Migration จริง**: ช่วง Coexistence ควรเก็บ Path เดิม (PPP/GRE
> จาก Part 19) ไว้เป็น Fallback จนกว่าจะมั่นใจว่า SD-WAN Overlay ทำงานเสถียรเต็มที่แล้วเท่านั้น
> — Migration WAN คือหนึ่งใน Task ที่มี Risk สูงสุดของงาน Network เพราะกระทบ Business
> Connectivity ทั้งองค์กรพร้อมกัน ต้องวางแผน Rollback Plan ไว้เสมอ

---

## แบบฝึกหัดทวนความเข้าใจ Part 39

1. SD-WAN แก้ปัญหาอะไรของ Traditional WAN (Part 19) ที่ทำให้การขยาย Site ใหม่ช้าและกระทบงาน
   Operation มาก?
2. อธิบายหน้าที่ของแต่ละ Plane ทั้ง 4 (Orchestration/Management/Control/Data) และบอกว่า
   vBond ต่างจาก vSmart อย่างไร?
3. OMP มีข้อมูล 3 ประเภทที่แจกจ่ายคืออะไร และ TLOC Route มีข้อมูลอะไรที่ BGP Route ธรรมดา
   ไม่มี?
4. ในตัวอย่าง Policy ของ VOICE Traffic (Step 386) คำว่า `strict` ใน `sla-class VOICE-SLA
   strict` มีผลอย่างไรถ้าไม่มี Path ไหนผ่าน SLA เลย?
5. Service VPN ของ SD-WAN ต่างจาก VLAN+SVI แบบเดิมอย่างไร ในมุมของ Routing Table?

**เฉลย:**
1. SD-WAN แก้ปัญหา **Box-by-Box Configuration** — Traditional WAN ต้อง Login CLI เข้าไปตั้งค่า
   ทีละอุปกรณ์ทุกครั้งที่เพิ่ม Site ใหม่หรือเปลี่ยน Policy ทำให้ Operation ช้าเป็นสัดส่วนกับ
   จำนวน Site ส่วน SD-WAN ใช้ **Zero-Touch Provisioning + Centralized Policy** ทำให้เพิ่ม Site
   ใหม่แค่เสียบสายแล้ว Controller Push Config ให้เองทั้งหมด
2. **Orchestration Plane (vBond)** ทำหน้าที่ Authenticate อุปกรณ์ใหม่และ "แนะนำทาง" ไปยัง
   vSmart/vManage เท่านั้น (ไม่เก็บ Routing/Policy ถาวร) ส่วน **Control Plane (vSmart)** ทำหน้าที่
   เป็น Route Reflector รัน OMP แจก Routing/TLOC/Policy ให้ทุก cEdge อย่างต่อเนื่อง (Persistent
   Connection) **Management Plane (vManage)** คือ GUI/API สำหรับ Template/Monitoring และ
   **Data Plane (vEdge/cEdge)** คือ Router จริงที่ Forward Traffic ผู้ใช้
3. OMP แจก **OMP Route (vRoute)**, **TLOC Route**, และ **Service Route** — TLOC Route มีข้อมูล
   **Color** (ประเภท Transport เช่น mpls/biz-internet/lte) ผูกกับ System-IP และ Encapsulation
   ซึ่งเป็นมิติที่ BGP Route ธรรมดาไม่มี (BGP มีแค่ Next-Hop ธรรมดา)
4. `strict` สั่งให้ **Drop Traffic นั้นทันทีถ้าไม่มี Path ไหนผ่าน SLA Class เลย** ดีกว่าปล่อยให้
   Traffic วิ่งผ่าน Path ที่คุณภาพแย่ (ซึ่งจะทำให้คุณภาพเสียง VoIP พังไปเลยแทนที่จะ Drop บางส่วน)
5. VLAN+SVI แบบเดิมแยกกันที่ Layer 2 (VLAN Tag) แต่มัก Route ผ่าน **Global Routing Table
   เดียวกัน** เว้นแต่ทำ VRF-Lite เพิ่ม ส่วน **Service VPN ของ SD-WAN แยก Routing/Forwarding
   Table จริงจังตั้งแต่ต้น** (คล้าย VRF) ทำให้ Multi-Tenancy และ Security Segmentation ทำได้
   เข้มงวดกว่าโดย Default โดยไม่ต้องเพิ่ม Configuration พิเศษ

---

## Step 390 — Lab เต็มรูปแบบ: แปลง WAN-EDGE-1/2 + BRANCH-RTR เป็น SD-WAN Fabric

> **หมายเหตุรูปแบบ Lab**: เพราะ SD-WAN ต้องมี vManage/vSmart/vBond ที่ไม่ได้อยู่ใน CLI Lab
> พื้นฐานของหลักสูตรนี้ Lab นี้จึงเป็น **Design/Architecture เต็มรูปแบบ** ที่วิศวกรจริงต้องทำ
> ก่อนขึ้นระบบจริง พร้อมตัวอย่าง Output ที่จะเห็นเมื่อ Provision บน Cisco Modeling Labs (CML)
> ที่มี vManage/vSmart/vBond/cEdge Image ครบ หรือ Cisco SD-WAN Cloud-hosted จริง

### 1) Controller Placement Plan

| Controller | จำนวนที่แนะนำ (Production) | ตำแหน่งที่แนะนำ | System IP ตัวอย่าง |
|---|---|---|---|
| vManage | 3 (Cluster) | Cisco Cloud-hosted หรือ HQ Data Center (VM บน ESXi/KVM) | 10.1.1.20 – 10.1.1.22 |
| vSmart | 2 (Minimum สำหรับ Redundancy) | เช่นเดียวกับ vManage คนละ Availability Zone | 10.1.1.10, 10.1.1.11 |
| vBond | 2 (Public-facing, ต้องมี Public IP) | DMZ หรือ Cloud (ต้องเข้าถึงได้จาก Internet เพื่อรับ ZTP จากทุก Site) | 10.1.1.1, 10.1.1.2 |

> Lab ขนาดเล็ก (Non-Production/POC) สามารถรวม vManage+vSmart+vBond ไว้ในเครื่องเดียวได้
> (All-in-One) แต่ **ไม่แนะนำสำหรับ Production** เพราะไม่มี Redundancy หากเครื่องนั้นล่ม

### 2) Site และ System-IP Plan (ผูกกับอุปกรณ์เดิมจาก Part 19)

| Hostname | Role ใหม่ | Site ID | System-IP (VPN 0) | Platform |
|---|---|---|---|---|
| WAN-EDGE-1 | cEdge (HQ) | 100 | 1.1.1.21 (คงเดิมจาก [00-ip-address-plan.md](00-ip-address-plan.md)) | ISR4451 (Controller-Mode) |
| WAN-EDGE-2 | cEdge (HQ) | 100 | 1.1.1.22 (คงเดิม) | ISR4451 (Controller-Mode) |
| BRANCH-RTR | cEdge (Branch) | 200 | 172.20.0.1 (คงเดิมจาก Part 19 Loopback0) | ISR4451 (Controller-Mode) |

> **Site ID**: WAN-EDGE-1 และ WAN-EDGE-2 ใช้ Site ID **เดียวกัน (100)** เพราะอยู่ HQ Site
> เดียวกันและทำหน้าที่ Redundant กัน (เหมือนที่ Part 19 ใช้ทั้งสองตัวเป็น WAN Edge คู่กัน)
> ส่วน BRANCH-RTR ได้ Site ID ใหม่ (200) เพราะเป็นสถานที่ทางภูมิศาสตร์ต่างกัน

### 3) TLOC / Transport (Color) Plan

| Device | Interface | Transport เดิม (Part 19) | Color ใหม่ (SD-WAN) |
|---|---|---|---|
| WAN-EDGE-1 | GigabitEthernet0/0/0 (→ ISP-RTR, `203.0.113.0/30`) | Internet ทาง ISP-RTR | `biz-internet` |
| WAN-EDGE-1 | Serial0/0/0 (→ BRANCH-RTR, PPP `172.16.199.0/30`) | Leased Line | `private1` |
| WAN-EDGE-2 | GigabitEthernet0/0/0 (→ ISP-RTR, `203.0.113.4/30`) | Internet ทาง ISP-RTR | `biz-internet` |
| BRANCH-RTR | GigabitEthernet0/0/1 (→ Internet, `198.51.100.2/30`) | Internet (สำหรับ GRE เดิม) | `biz-internet` |
| BRANCH-RTR | Serial0/0/0 (→ WAN-EDGE-1, PPP) | Leased Line | `private1` |

### 4) VPN Segment Design — Mapping VLAN เดิมเข้า Service VPN

| VLAN เดิม ([00-ip-address-plan.md](00-ip-address-plan.md)) | Subnet | Service VPN ใหม่ | หมายเหตุ |
|---|---|---|---|
| VLAN 10 (SALES) | 10.10.10.0/24 | VPN 10 | Site ที่ไม่มี VLAN นี้ (เช่น BRANCH-RTR) ไม่ต้องสร้าง VPN 10 |
| VLAN 20 (VOICE) | 10.10.20.0/24 | VPN 20 | ผูกกับ App-Route Policy VOICE-SLA (Step 386) |
| VLAN 30 (SERVERS) | 10.10.30.0/24 | VPN 30 | จำกัดสิทธิ์เข้าถึงด้วย Centralized Data Policy (Step 387) |
| VLAN 40 (WIFI) | 10.10.40.0/24 | VPN 40 | แยก Isolation จาก VPN 30 ตาม Policy ตัวอย่าง Step 387 |
| VLAN 99 (MGMT) | 10.10.99.0/24 | VPN 512 (Management) | Mapping ตรงกับหลักการ Out-of-Band Management ของ SD-WAN โดยธรรมชาติ |
| Branch LAN (172.20.0.1/32 เดิม) | 172.20.0.0/24 (สมมติขยายจาก Loopback) | VPN 20 (Branch ใช้ VOICE Segment ร่วม) | Site เล็กอาจ Map เข้า VPN เดียวกับ HQ ถ้าเป็น Function เดียวกัน |

### 5) Diagram รวมของ SD-WAN Fabric ที่สมบูรณ์

```
                         ┌───────────────────────────────────────┐
                         │     vManage / vSmart(x2) / vBond(x2)     │
                         │        (Controller Stack — Cloud)         │
                         └───────────────┬─────────────────────────┘
                                        │ Control Plane (DTLS/TLS + OMP)
                ┌────────────────────────┼────────────────────────┐
                │                        │                        │
   ┌────────────┴────────────┐   ┌───────┴────────────┐   ┌───────┴────────────┐
   │   WAN-EDGE-1 (cEdge)      │   │  WAN-EDGE-2 (cEdge)  │   │  BRANCH-RTR (cEdge) │
   │   Site 100 / Sys-IP        │   │  Site 100 / Sys-IP    │   │  Site 200 / Sys-IP   │
   │   1.1.1.21                 │   │  1.1.1.22              │   │  172.20.0.1           │
   │  TLOC: biz-internet,       │   │  TLOC: biz-internet     │   │  TLOC: biz-internet,   │
   │        private1             │   │                         │   │        private1        │
   │  VPN 10/20/30/40/512       │   │  VPN 10/20/30/40/512    │   │  VPN 20/512             │
   └──────────┬───────────────┘   └──────────┬───────────┘   └──────────┬───────────┘
              │                              │                          │
              └═════════ IPsec Overlay Full-Mesh (Data Plane) ══════════┘
                    (สร้างอัตโนมัติทั้ง 3 เส้นจาก TLOC ที่ vSmart แจก)
              │                                                          │
   ┌──────────┴───────────┐                                   ┌──────────┴───────────┐
   │ CORE-SW1/CORE-SW2      │                                 │  Branch LAN (172.20.0.0/24) │
   │ (VLAN 10/20/30/40/99)  │                                 │   (VPN 20 — VOICE Segment)   │
   └────────────────────────┘                                 └────────────────────────────┘
```

### 6) Application-Aware Policy สำหรับ VOICE — Policy สรุปที่ Apply จริงกับ Fabric นี้

```
policy
  sla-class VOICE-SLA
    latency        100
    loss           1
    jitter         20
  !
  app-route-policy VOICE-APP-AWARE-POLICY
    vpn-list       VPN20-LIST                 ! ครอบคลุม VPN 20 (VOICE) ทุก Site
    sequence 10
      match
        dscp 46
      action
        sla-class VOICE-SLA strict
        preferred-color private1              ! Leased Line เดิม ยังเป็น Path ที่ต้องการก่อน
  !
  lists
    vpn-list VPN20-LIST
      vpn 20
    site-list ALL-SITES
      site-id 100
      site-id 200
!
apply-policy site-list ALL-SITES app-route-policy VOICE-APP-AWARE-POLICY
```

### 7) ตัวอย่าง Output การ Verify ทั้ง Fabric เมื่อ Provision สำเร็จ (สิ่งที่วิศวกรจะเห็นจริง)

**Control Plane ของ BRANCH-RTR ขึ้นครบกับ Controller ทุกตัว:**

```
BRANCH-RTR# show sdwan control connections

PEER TYPE   PEER PROT  PEER SYSTEM IP   SITE ID  STATE  UPTIME
vbond       dtls       10.1.1.1         0        up     4:02:10:33
vsmart      dtls       10.1.1.10        100      up     4:02:09:58
vsmart      dtls       10.1.1.11        100      up     4:02:09:55
vmanage     dtls       10.1.1.20        100      up     4:02:09:50
```

**Data Plane เห็น IPsec Tunnel ไปยัง WAN-EDGE-1 และ WAN-EDGE-2 ทั้งคู่ (Full-Mesh สำเร็จ):**

```
BRANCH-RTR# show sdwan bfd sessions

SYSTEM IP     SITE ID  SOURCE TLOC COLOR  DEST TLOC COLOR   STATE  AVG LATENCY  AVG JITTER  AVG LOSS
1.1.1.21      100      biz-internet       biz-internet      up     58 ms        14 ms       1%
1.1.1.21      100      private1           private1          up     22 ms        3 ms        0%
1.1.1.22      100      biz-internet       biz-internet      up     61 ms        16 ms       1%
```

**Route ของ VLAN 20 (VOICE, VPN 20) จาก HQ ถูกแจกมาถึง Branch ผ่าน OMP:**

```
BRANCH-RTR# show sdwan omp routes vpn 20

           C   -> chosen        I -> installed      R -> resolved

PREFIX             FROM PEER   PATH ID   STATUS   TLOC IP   COLOR      ENCAP
10.10.20.0/24      10.1.1.10   88        C,I,R    1.1.1.21   private1   ipsec
10.10.20.0/24      10.1.1.10   89        C,I,R    1.1.1.21   biz-internet ipsec
```

**App-Route Policy เลือก Path `private1` (Leased Line เดิม) ให้ VOICE Traffic ตามที่ตั้งไว้:**

```
BRANCH-RTR# show sdwan app-route stats

TLOC COLOR    REMOTE COLOR   MEAN LOSS  MEAN LATENCY  MEAN JITTER  APPLIED SLA CLASS   RESULT
private1      private1       0          22            3            VOICE-SLA           PASS (selected)
biz-internet  biz-internet   1          58             14           VOICE-SLA           PASS (backup)
```

Output ทั้ง 4 ชุดนี้คือสิ่งที่วิศวกร SD-WAN ตรวจสอบเป็นประจำ (Daily Operation Checklist)
เพื่อยืนยันว่า **Control Plane, Data Plane, Route Distribution, และ Application-Aware Policy**
ทำงานสอดคล้องกันตามที่ Design ไว้ทั้งหมด — ครบทั้ง 4 Planes ที่อธิบายไว้ใน Step 382

### สรุปตาราง Checklist การ Migration ที่ทำสำเร็จใน Lab นี้

| รายการ | Part 19 (Traditional) | Part 39 (SD-WAN) — ผลลัพธ์ Lab นี้ |
|---|---|---|
| Path Selection | Static AD คงที่ | AAR ตาม SLA Class Real-time |
| จำนวน Tunnel ที่ต้องคอนฟิกเอง | 1 GRE Tunnel (คอนฟิกมือ) | 0 (Auto Full-Mesh จาก TLOC) |
| การเพิ่ม Site ใหม่ | Console + คอนฟิกมือทั้งหมด | ZTP + Device Template |
| การเปลี่ยน Security Policy | ทีละ Router | Centralized ที่ vManage ครั้งเดียว |
| การแยก Traffic ตาม Department | VLAN + SVI (Global Routing) | Service VPN (Routing Table แยกจริง) |

---

## สรุป Part 39

Part นี้พา BRANCH-RTR และ WAN-EDGE-1/2 จาก Design แบบ **PPP+GRE ของ Part 19** ก้าวเข้าสู่โลกของ
**Cisco SD-WAN (Viptela)** อย่างเต็มรูปแบบ — เริ่มจากเหตุผลที่ธุรกิจต้องการเปลี่ยน, สถาปัตยกรรม
4 Planes (Orchestration/Management/Control/Data), OMP ที่แจก vRoute/TLOC/Service Route,
IPsec Overlay ที่สร้างอัตโนมัติผ่านแนวคิด TLOC/Color, Zero-Touch Provisioning ที่ทำให้เพิ่ม
Site ใหม่ไม่ต้องพิมพ์ CLI, Application-Aware Routing ที่เลือก Path ตามคุณภาพจริงสำหรับ VOICE,
Centralized Policy/VPN Segmentation ที่คุม Fabric ทั้งองค์กรจากจุดเดียว, แนวคิด Direct Cloud
Access เบื้องต้น, ไปจนถึงกลยุทธ์ Migration และ Lab การออกแบบเต็มรูปแบบที่แปลง HQ+Branch เดิม
เป็น SD-WAN Fabric ที่สมบูรณ์ ✅

**เชิงลึกที่ยังไม่ครบ (ตั้งใจค้างไว้)**: Cloud OnRamp for SaaS/IaaS/Multi-Cloud แบบเต็มรูปแบบ
จะเรียนใน **Part 93**, ส่วน Policy ขนาดใหญ่ระดับ Enterprise (Multi-Region Topology, Advanced
Security Integration, และ SD-WAN เชิงลึกอื่นๆ) จะกลับมาสอนเต็มรูปแบบใน **Part 76**

**พร้อมสำหรับ Part 40**: เราจะกลับมาที่พื้นฐานที่ Part นี้อ้างถึงหลายครั้ง คือ **DSCP 46 (EF)**
ที่ใช้ Mark VOICE Traffic — Part 40 จะสอน **Advanced QoS** อย่างเต็มรูปแบบ ตั้งแต่ Classification,
Marking (DSCP/CoS), Queuing (LLQ/CBWFQ), Shaping/Policing, ไปจนถึงการนำ QoS Policy นี้ไปใช้
ร่วมกับ Application-Aware Routing ของ SD-WAN ที่เราเพิ่งเรียนใน Part นี้

**ไปต่อ:** [Part 40 — Advanced QoS →](part-040-advanced-qos.md)
