# Part 38 — SD-Access Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 371–380 จาก 1000**

> ต่อจาก [Part 37 — Wireless Security ขั้นสูง](part-037-wireless-security-advanced.md) ที่ปิดท้าย
> กลุ่มเนื้อหา Wireless ระดับ CCNP ENCOR แล้ว Part นี้จะเปลี่ยนมุมมองแบบสิ้นเชิง — จาก
> [Part 33 Step 322](part-033-advanced-stp-campus-design.md) เราได้ทิ้งท้ายไว้ว่า **"Routed Access
> เหมาะกับสถาปัตยกรรม Fabric-based มากกว่า...เป็นสถาปัตยกรรมคนละแบบที่ต้องเรียนแยกใน Part 38
> โดยเฉพาะ"** — **Part นี้คือ Part ที่ทำตามสัญญานั้น** เราจะแนะนำ **Cisco SD-Access
> (Software-Defined Access)** ซึ่งเป็นการ **ออกแบบ Campus Network ใหม่ทั้งหมดในเชิงแนวคิด**
> โดยใช้ **Fabric** แทน 3-Tier Model แบบดั้งเดิม
>
> **สำคัญมาก — ต้องเข้าใจก่อนเริ่ม**: Part นี้สอน SD-Access ในฐานะ **สถาปัตยกรรมทางเลือก
> (Alternative Design Philosophy)** ที่ CCNP ENCOR Exam Blueprint กำหนดให้ต้องรู้ **ไม่ใช่การ
> รื้อ Lab เดิมที่สร้างมาตั้งแต่ Part 1–35 ทิ้ง** — Lab หลักของหลักสูตร (CORE-SW1/2, DIST-SW1–4,
> ACCESS-SW1–4, WLC-1 แบบ Traditional 3-Tier) จะยังถูกใช้และขยายต่อเนื่องตั้งแต่ **Part 39 เป็นต้นไป**
> ตามปกติ Part นี้เพียง "ยืม" อุปกรณ์ชุดเดิมมา **จำลองว่าถ้าจะ Re-architect เป็น SD-Access จะทำ
> อย่างไร" เพื่อให้เห็นภาพความแตกต่างชัดเจนที่สุด — เนื้อหานี้อยู่ในขอบเขตข้อสอบ CCNP ENCOR
> (350-401) โดยตรง แต่ **ไม่ใช่ CLI-heavy** เหมือน Part ก่อนๆ เพราะ SD-Access ถูกออกแบบให้
> ควบคุมผ่าน **Cisco DNA Center (DNAC)** เป็นหลัก ไม่ใช่ CLI แบบ Manual

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 371 | SD-Access คืออะไร — แรงจูงใจ และ 3 Plane หลัก (Control/Data/Policy) |
| 372 | Fabric Roles — Edge Node, Border Node, Control Plane Node, Fabric WLC |
| 373 | LISP (Locator/ID Separation Protocol) — EID vs RLOC, Map-Server/Resolver, xTR |
| 374 | VXLAN Data Plane ใน SD-Access — VNI ↔ Virtual Network |
| 375 | Macro-Segmentation (VN) vs Micro-Segmentation (SGT/TrustSec) |
| 376 | บทบาทของ DNA Center — Underlay Automation และ Fabric Provisioning Workflow |
| 377 | Underlay vs Overlay — ทำไม Part 33 (OSPF ECMP) คือ Underlay ที่ใช้ซ้ำได้ทันที |
| 378 | SD-Access Wireless — WLC-1 และ AP กลายเป็น Fabric-enabled อย่างไร |
| 379 | Migration Considerations — Extended Node / Policy Extended Node สำหรับ Switch รุ่นเก่า |
| 380 | Lab เชิงแนวคิด: แผนผัง Fabric Role เต็มรูปแบบ + Underlay Reachability + DNAC Workflow + `show` Command |

---

## Step 371 — SD-Access คืออะไร: แรงจูงใจ และ 3 Plane หลัก

### ปัญหาของ Traditional Campus ที่ SD-Access พยายามแก้

ตลอด Part 1–35 เราสร้าง Campus แบบ Traditional (3-Tier: Core/Distribution/Access) ซึ่งทำงานได้ดี
มาก แต่มีข้อจำกัดเชิงสถาปัตยกรรมที่ **ไม่สามารถแก้ได้ด้วยการเพิ่ม Feature ทีละตัว** อีกต่อไปเมื่อ
องค์กรขยายใหญ่ขึ้นถึงจุดหนึ่ง:

| ปัญหาใน Traditional Campus | รายละเอียด | Part ที่เคยเจอปัญหานี้ |
|---|---|---|
| **VLAN Sprawl** | ทุก Policy/Segment ใหม่ต้องสร้าง VLAN ใหม่ → ต้อง Trunk VLAN นั้นทะลุทุก Switch ที่เกี่ยวข้อง → Trunk โตขึ้นเรื่อยๆ, STP Topology ซับซ้อนขึ้นเรื่อยๆ | Part 3–5 |
| **Segmentation ผูกกับ IP Subnet เสมอ** | จะแยก Policy ตาม "กลุ่มผู้ใช้" (เช่น Contractor vs Employee) ต้องสร้าง Subnet/VLAN ใหม่เสมอ แม้ผู้ใช้ทั้งสองกลุ่มอยู่ห้องเดียวกัน | Part 14 (ACL ยึด IP/Subnet) |
| **Host Mobility ยาก** | ถ้าอุปกรณ์ (โดยเฉพาะ Wireless Client) ย้าย Location ข้าม Subnet เดิม ต้องเปลี่ยน IP หรือทำ Mobility Tunnel ซับซ้อน | Part 20, 36–37 |
| **Consistency ของ Policy ทั่วองค์กร** | ACL/QoS/Security Policy ถูก Config แยกเครื่องต่อเครื่อง → เสี่ยง Human Error, ไม่มี Single Source of Truth | Part 14, 21–22, 24 |
| **Provisioning ช้า** | เพิ่ม Switch/AP ใหม่ 1 ตัว ต้อง SSH เข้าไป Config CLI เองทุกบรรทัด (ทั้ง Part 1–37 ที่ผ่านมา) | ทุก Part ที่ผ่านมา |

**Cisco SD-Access** คือคำตอบของ Cisco สำหรับปัญหาเหล่านี้ — เป็นส่วนหนึ่งของ **Cisco Digital
Network Architecture (Cisco DNA)** ที่ใช้แนวคิด **Fabric** (คล้ายกับที่ Data Center ใช้ VXLAN/EVPN
Fabric แก้ปัญหาคล้ายกันในโลก Data Center — จะเรียนเจาะลึกที่ Part 84–85 ระดับ CCIE) มาใช้กับ
Campus/Branch โดยมีเป้าหมาย 4 ข้อหลัก:

1. **Policy Consistency** — กำหนด Policy ครั้งเดียวที่ DNA Center แล้ว Push ไปทุกอุปกรณ์ในองค์กร
   อัตโนมัติ ไม่ต้องไล่ Config เครื่องต่อเครื่อง
2. **Automation** — Provisioning อุปกรณ์ใหม่ (Switch, AP) ทำผ่าน Zero-Touch/Plug-and-Play (PnP)
   และ GUI Workflow ไม่ต้องเขียน CLI ทีละบรรทัด
3. **Segmentation โดยไม่ต้องพึ่ง VLAN Sprawl** — แยก Policy ตาม "กลุ่ม" ของผู้ใช้/อุปกรณ์ได้โดยตรง
   ไม่ต้องผูกกับ IP Subnet เดิม (รายละเอียดที่ Step 375)
4. **Single Source of Truth** — DNA Center เป็นศูนย์กลางเดียวที่รู้ Topology, Policy, และ State
   ของทั้งเครือข่าย (ทั้ง Wired และ Wireless)

### 3 Plane หลักของ SD-Access

SD-Access แยกการทำงานของ Fabric ออกเป็น **3 Plane ที่อิสระจากกัน** — นี่คือหัวใจของสถาปัตยกรรม
ทั้งหมดที่ทุก Step ถัดไปจะอ้างอิงกลับมา:

| Plane | เทคโนโลยีที่ใช้ | หน้าที่ | สอนละเอียดที่ Step |
|---|---|---|---|
| **Control Plane** | **LISP** (Locator/ID Separation Protocol) | รู้ว่า Endpoint (EID) แต่ละตัวอยู่ที่ไหนในเครือข่าย ณ ขณะนี้ (ตอบคำถาม "ใครอยู่ที่ไหน") | Step 373 |
| **Data Plane** | **VXLAN** (Virtual Extensible LAN) | ห่อหุ้ม (Encapsulate) Traffic จริงของ Endpoint เพื่อส่งข้ามเครือข่าย Underlay โดยที่ Endpoint ไม่รู้ตัว | Step 374 |
| **Policy Plane** | **Cisco TrustSec / SGT** (Scalable Group Tag) | กำหนดว่า "กลุ่มไหน" คุยกับ "กลุ่มไหน" ได้บ้าง โดยไม่ขึ้นกับ IP Address | Step 375 |

### ภาพรวมสถาปัตยกรรมระดับสูง (High-Level Architecture)

```
                              ┌─────────────────────────────┐
                              │      Cisco DNA Center         │  <- Single Source of Truth
                              │  (Management/Automation Plane)│     (Step 376)
                              └───────────────┬───────────────┘
                                     REST API / NETCONF
                    ┌───────────────────────────┴───────────────────────────┐
                    │                     SD-Access Fabric                    │
                    │                                                          │
    ┌───────────────┼──────────────────┬───────────────────┬─────────────────┤
    │  CONTROL PLANE │   DATA PLANE     │   POLICY PLANE    │   UNDERLAY       │
    │     (LISP)     │    (VXLAN)       │  (SGT/TrustSec)   │  (Routed L3)     │
    │                │                  │                    │                  │
    │ "ใครอยู่ที่ไหน" │ "ส่งข้อมูลจริง  │ "ใครคุยกับใครได้"  │ "เส้นทาง IP      │
    │  Map-Server/   │  ยังไงข้าม       │  SGT tag ติดไปกับ  │   พื้นฐานระหว่าง │
    │  Map-Resolver  │  Underlay"       │  packet ทุก hop    │   Fabric Node"   │
    └────────────────┴──────────────────┴────────────────────┴──────────────────┘
```

**ข้อสังเกตสำคัญ**: 3 Plane บนนี้ "ซ้อน" อยู่บน **Underlay** อีกชั้นหนึ่งเสมอ (กล่องขวาสุด) —
Underlay เป็นเพียง Routed Network ธรรมดาที่มีหน้าที่แค่ "ให้ Reachability ระหว่าง Loopback ของ
Fabric Node ทุกตัว" เท่านั้น (จะพิสูจน์ว่า Part 33 สร้าง Underlay แบบนี้ไว้แล้วโดยไม่รู้ตัวที่ Step 377)

> **สำหรับข้อสอบ CCNP ENCOR**: คำถามแนว "SD-Access ใช้ Protocol ใดเป็น Control Plane" ตอบ
> **LISP** เสมอ (ไม่ใช่ BGP EVPN แบบ Data Center Fabric) — นี่คือความแตกต่างที่ข้อสอบชอบถามสับสน
> ระหว่าง SD-Access (Campus) กับ VXLAN/EVPN Fabric (Data Center, Part 84–85)

---

## Step 372 — Fabric Roles: Edge Node, Border Node, Control Plane Node, Fabric WLC

SD-Access Fabric ประกอบด้วย **Role ทางตรรกะ (Logical Role)** 4 แบบหลัก ซึ่งอุปกรณ์ทางกายภาพ
1 ตัวสามารถทำหน้าที่ **มากกว่า 1 Role พร้อมกัน** ได้ (เช่น Border + Control Plane Node ในตัวเดียว)

### 4 Role หลักของ Fabric

| Role | หน้าที่ | เทียบกับ 3-Tier เดิม |
|---|---|---|
| **Fabric Edge Node** | จุดที่ Endpoint (PC, Server, AP) เชื่อมต่อเข้า Fabric โดยตรง ทำหน้าที่ Encapsulate/Decapsulate VXLAN (เป็น VTEP) และ Enforce Policy (SGACL) ให้ Endpoint แต่ละตัว | คล้าย **Access Layer** — แต่ต้องเป็น Switch ที่รองรับ VXLAN/LISP เท่านั้น (L3-capable) |
| **Fabric Border Node** | จุดเชื่อม Fabric เข้ากับโลกภายนอก (Internet, Shared Services, Traditional Network ที่ยังไม่ใช่ Fabric) — แปลง Traffic ระหว่าง Fabric (VXLAN/LISP) กับ Network ทั่วไป (Native IP) | คล้าย **Core Layer** — เป็นประตูสู่ WAN Edge/Firewall/Data Center |
| **Fabric Control Plane Node** | รัน LISP Map-Server/Map-Resolver — เป็น "สมุดโทรศัพท์" ที่รู้ว่า Endpoint (EID) ไหนอยู่หลัง Edge Node ตัวใด ณ ขณะนี้ | **ไม่มีในสถาปัตยกรรมเดิม** — เป็น Role ใหม่ล้วนที่ Traditional Campus ไม่ต้องการ |
| **Fabric WLC** | Wireless LAN Controller ที่ Integrate เข้ากับ Fabric — ลงทะเบียน Wireless Client เป็น EID ใน LISP เหมือน Wired Client | คล้าย **WLC เดิม** แต่ทำงานร่วมกับ Fabric Control Plane โดยตรง (ดูรายละเอียดที่ Step 378) |

### Diagram: ตำแหน่ง Role ทั้ง 4 ใน Fabric (เชิงแนวคิด)

```
                         ┌────────────────────────────┐
                         │  ภายนอก Fabric              │
                         │  (Internet, Data Center,    │
                         │   Traditional Network เดิม)  │
                         └──────────────┬─────────────┘
                    ┌────────────────────┴────────────────────┐
                    │           FABRIC BORDER NODE ×2            │  <- ประตูเข้า-ออก Fabric
                    │   (มักรัน Control Plane Node ร่วมด้วย       │     (Redundant pair)
                    │    เพื่อความง่ายในการออกแบบขนาดกลาง)         │
                    └──────┬─────────────────────────────┬──────┘
                 ┌─────────┴─────────┐           ┌─────────┴─────────┐
                 │ Intermediate Node   │           │ Intermediate Node   │  <- แค่ Routed Underlay
                 │ (Underlay routing   │           │ (Underlay routing   │     ไม่รู้จัก VXLAN/LISP
                 │  เท่านั้น ไม่รู้จัก  │           │  เท่านั้น ไม่รู้จัก  │     เลย (Step 377)
                 │  VXLAN/LISP)        │           │  VXLAN/LISP)        │
                 └────┬───────────┬────┘           └────┬───────────┬────┘
              ┌────────┴──┐  ┌────┴───────┐   ┌────────┴──┐  ┌────┴───────┐
              │FABRIC EDGE │  │FABRIC EDGE  │   │FABRIC EDGE │  │FABRIC EDGE  │  <- Endpoint
              │NODE        │  │NODE         │   │NODE        │  │NODE         │     เชื่อมต่อที่นี่
              └─────┬──────┘  └──────┬──────┘   └─────┬──────┘  └──────┬──────┘
                    │                │                │                │
                PC/Server      Fabric AP ◄── FABRIC WLC ──► ควบคุม AP ทุกตัว
                                (Step 378)      (Control-plane ของ Wireless)
```

### แนวคิดสำคัญ: Intermediate Node ไม่ใช่ Role ของ Fabric

สังเกตว่ากล่องกลาง (ระหว่าง Border กับ Edge) เขียนว่า **"Intermediate Node"** ซึ่ง **ไม่ใช่ 1 ใน
4 Role ของ Fabric** — เป็น Switch ที่ทำหน้าที่แค่ **Routed Underlay ธรรมดา** (ส่ง IP Packet ที่ถูก
VXLAN Encapsulate แล้วให้ถึงปลายทาง) โดย **ไม่รู้จักและไม่ต้องรู้จัก VXLAN/LISP เลย** — นี่คือจุดที่
ทำให้ SD-Access Deploy ได้ง่ายกว่าที่คิด เพราะ Layer กลางของ Topology (มักตรงกับ **Distribution
Layer เดิม**) **ไม่ต้อง Upgrade Software/Hardware อะไรเป็นพิเศษเลย** ยังคง Route แบบ OSPF/IS-IS
ปกติ (ดูเหตุผลเต็มที่ Step 377)

### Mapping เชิงแนวคิด: ถ้าจะจำลอง Lab เดิมเป็น Fabric Role (ภาพรวม — รายละเอียดเต็มที่ Step 380)

| อุปกรณ์ Lab เดิม | Role ใน 3-Tier เดิม | Role ที่จะรับใน SD-Access (เชิงแนวคิด) |
|---|---|---|
| CORE-SW1, CORE-SW2 | Core Layer | **Fabric Border Node** (+ Control Plane Node ร่วม) |
| DIST-SW1–4 | Distribution Layer | **Intermediate Node** (Underlay ล้วน ไม่มี Fabric Role) |
| ACCESS-SW1–4 | Access Layer | **Fabric Edge Node** (เชิงแนวคิด — มีข้อจำกัด Hardware ดูคำเตือนที่ Step 379/380) |
| WLC-1 | Wireless Controller | **Fabric WLC** |

> **หมายเหตุสำคัญ**: การ Mapping นี้เป็นแบบฝึกหัดเชิงแนวคิด (Conceptual Exercise) เพื่อให้เห็นภาพ
> Fabric Role จับกับอุปกรณ์จริงในหัวได้ง่ายขึ้นเท่านั้น **ไม่ใช่คำแนะนำให้ไป Deploy จริงกับ Lab
> ปัจจุบัน** เพราะ ACCESS-SW1–4 เป็น Catalyst 9200/2960 ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md)
> กำหนดไว้ ซึ่ง**ไม่รองรับ VXLAN/LISP ทางกายภาพ** (ต้องเป็น Fabric-capable Switch เช่น Catalyst
> 9300 เป็นอย่างน้อย) — รายละเอียดทางออกสำหรับ Hardware แบบนี้อยู่ที่ Step 379 (Extended Node)

---

## Step 373 — LISP (Locator/ID Separation Protocol) พื้นฐาน

### ปัญหาดั้งเดิมที่ LISP แก้: IP Address ทำหน้าที่ 2 อย่างพร้อมกัน

ใน IP Networking แบบเดิม (ที่เราใช้มาตั้งแต่ Part 7) IP Address ของ Host ทำหน้าที่ **2 อย่างปนกัน**
โดยไม่ตั้งใจ:

1. **บอกว่า Host นี้คือใคร** (Identity) — เช่น Server1 คือ `10.10.30.10`
2. **บอกว่า Host นี้อยู่ที่ไหนในเครือข่าย** (Location) — Routing Table ใช้ `10.10.30.0/24` เพื่อ
   หาทางไปหา Server1

ปัญหาคือ ถ้า Server1 **ย้ายที่ตั้งทางกายภาพ** (เช่นย้ายไปอีกตึก อีก Subnet) แต่ต้องการให้ยังใช้
IP `10.10.30.10` เดิม (ไม่ Renumber) — Routing แบบเดิม **ทำไม่ได้** เพราะ Routing Table
Route ตาม Subnet Boundary เท่านั้น

**LISP (Locator/ID Separation Protocol)** แก้ปัญหานี้โดย **แยก Identity ออกจาก Location
อย่างชัดเจนเป็น 2 Address คนละชุด**:

| แนวคิด | ชื่อเต็ม | คืออะไร | ตัวอย่าง |
|---|---|---|---|
| **EID** | Endpoint Identifier | IP ของ Endpoint จริง (Identity) — **ไม่เปลี่ยนแม้ Endpoint ย้ายที่** | `10.10.30.10` (Server1) |
| **RLOC** | Routing Locator | IP ของ Fabric Edge Node ที่ Endpoint นั้นเชื่อมต่ออยู่ ณ ขณะนี้ (Location) — **เปลี่ยนได้ตามที่ Endpoint ย้าย** | `1.1.1.101` (Loopback ของ Edge Node ที่ Server1 ต่ออยู่) |

### สถาปัตยกรรม LISP: Map-Server, Map-Resolver, และ xTR

```
                        ┌─────────────────────────────────┐
                        │   LISP Map-Server / Map-Resolver   │  <- Fabric Control Plane Node
                        │   (มักรวมเป็นตัวเดียว เรียก MS/MR)  │     (Step 372)
                        │                                     │
                        │   สมุดโทรศัพท์:                     │
                        │   EID 10.10.30.10  → RLOC 1.1.1.101 │
                        │   EID 10.10.10.5   → RLOC 1.1.1.102 │
                        └──────────────┬──────────────────────┘
                        ① Register EID          ② Query "EID นี้อยู่ RLOC ไหน"
                        (จาก ETR)                (จาก ITR)
              ┌───────────────────────┴───────────────────────┐
              │                                                 │
      ┌───────┴────────┐                              ┌────────┴───────┐
      │  Fabric Edge    │  ◄── VXLAN Tunnel (RLOC-RLOC) ──►  │  Fabric Edge    │
      │  Node A          │      (Data Plane, Step 374)         │  Node B          │
      │  RLOC=1.1.1.102 │                                     │  RLOC=1.1.1.101 │
      │  (ทำหน้าที่ ITR) │                                     │  (ทำหน้าที่ ETR) │
      └───────┬─────────┘                                     └────────┬────────┘
              │                                                         │
          PC1 (EID 10.10.10.5)                                Server1 (EID 10.10.30.10)
          ต้องการคุยกับ Server1                                  ลงทะเบียนตัวเองไว้กับ MS/MR
```

### xTR: Ingress Tunnel Router (ITR) และ Egress Tunnel Router (ETR)

**xTR** คือชื่อรวมของอุปกรณ์ที่ทำหน้าที่เป็นทั้ง ITR และ ETR พร้อมกัน (ซึ่งก็คือ **Fabric Edge Node
ทุกตัว**) — แยกบทบาทตามทิศทางของ Traffic ขณะนั้น:

| บทบาท | ทำงานเมื่อไหร่ | หน้าที่ |
|---|---|---|
| **ITR (Ingress Tunnel Router)** | เมื่อรับ Packet **ขาเข้า** จาก Endpoint ของตัวเอง ที่ต้องการส่งออกไปหา EID อื่น | ถาม Map-Resolver ว่า EID ปลายทางอยู่ RLOC ไหน แล้ว Encapsulate Packet ด้วย VXLAN ส่งไปยัง RLOC นั้น |
| **ETR (Egress Tunnel Router)** | เมื่อรับ Packet ที่ถูก VXLAN Encapsulate มาจาก Edge Node อื่น ปลายทางเป็น Endpoint ของตัวเอง | Decapsulate VXLAN แล้วส่ง Packet เดิมต่อให้ Endpoint ปลายทาง และ **Register EID ของ Endpoint ตัวเองไปที่ Map-Server** |

### ขั้นตอนการทำงานแบบละเอียด (Worked Example)

```
1. Server1 เชื่อมต่อเข้า Fabric Edge Node B (RLOC 1.1.1.101)
   → Edge Node B (ทำหน้าที่ ETR) Register: "EID 10.10.30.10 อยู่ที่ RLOC 1.1.1.101" ไปยัง MS/MR

2. PC1 (หลัง Edge Node A) ต้องการส่ง Packet ไปหา Server1 (10.10.30.10)
   → Edge Node A (ทำหน้าที่ ITR) ไม่รู้ว่า Server1 อยู่ RLOC ไหน จึงถาม Map-Resolver:
     "EID 10.10.30.10 อยู่ RLOC ไหน?"

3. Map-Resolver/Map-Server ตอบ: "RLOC 1.1.1.101"

4. Edge Node A (ITR) Encapsulate Packet เดิมด้วย VXLAN Header
   (Outer Src = RLOC ของตัวเอง 1.1.1.102, Outer Dst = RLOC 1.1.1.101)
   แล้วส่งผ่าน Underlay (Intermediate Node ธรรมดา ไม่รู้จัก LISP เลย)

5. Edge Node B (ETR) รับ Packet, Decapsulate VXLAN, ส่ง Packet เดิมต่อให้ Server1
```

### ทำไมสิ่งนี้ทำให้ Host Mobility เป็นไปได้โดยไม่ต้อง Renumber

ถ้า Server1 **ย้าย** ไปเชื่อมต่อที่ Fabric Edge Node C (RLOC `1.1.1.103`) แทน:

- **EID ของ Server1 (`10.10.30.10`) ไม่เปลี่ยนเลย** — ไม่มีใครต้อง Renumber Application หรือ
  DNS Record
- Edge Node C (ETR ใหม่) จะ Register EID เดิมไปที่ MS/MR ทันทีว่า **"ตอนนี้ RLOC ของ EID นี้คือ
  1.1.1.103 แล้ว"** — MS/MR แค่ **Update Entry ในสมุดโทรศัพท์** ไม่ต้องเปลี่ยน Routing Table
  ทั้ง Fabric เลย
- Query รอบถัดไปจาก ITR ใดๆ จะได้คำตอบ RLOC ใหม่โดยอัตโนมัติ — นี่คือกลไกที่ทำให้ **Wireless
  Client Roaming ข้าม AP/Edge Node** (Step 378) ทำได้อย่างไร้รอยต่อโดยไม่ต้องเปลี่ยน IP เลย

> **สำหรับข้อสอบ CCNP ENCOR**: จำคู่คำนี้ให้แม่น — **EID = ตัวตน (Identity, ไม่เปลี่ยน)**,
> **RLOC = ตำแหน่ง (Location, เปลี่ยนได้)** — และ **Map-Server/Map-Resolver คือ Fabric Control
> Plane Node**, **ITR/ETR รวมกันเรียก xTR คือหน้าที่ที่ Fabric Edge Node ทำ**

---

## Step 374 — VXLAN Data Plane ใน SD-Access

### VXLAN คืออะไร (สรุปแบบย่อ — เจาะลึกเต็มรูปแบบที่ Part 84–85 ระดับ CCIE)

**VXLAN (Virtual Extensible LAN)** คือเทคนิค **Encapsulation** ที่ห่อหุ้ม Ethernet Frame เดิมของ
Endpoint (Layer 2 Frame ทั้งดวง) ไว้ภายใน UDP/IP Packet ใหม่ ทำให้ส่ง Frame เดิมข้าม Routed
Underlay ธรรมดาได้ **โดยที่ Underlay มองเห็นแค่ IP Packet ปกติ ไม่รู้เลยว่าข้างในมี Frame ซ้อนอยู่**

```
Frame เดิมของ Endpoint (ก่อน Encapsulate):
┌────────────┬────────────┬──────────┬──────────┐
│ Eth Header  │ IP Header   │ TCP/UDP   │ Data     │
└────────────┴────────────┴──────────┴──────────┘

หลัง VXLAN Encapsulate ที่ Fabric Edge Node (ITR):
┌───────────┬──────────┬───────────┬──────────┬────────────┬────────────┬──────────┬──────────┐
│ Outer Eth  │ Outer IP  │ Outer UDP  │ VXLAN     │ Eth Header  │ IP Header   │ TCP/UDP   │ Data     │
│ (Underlay) │ (RLOC↔RLOC)│ (Port 4789)│ Header    │ (Frame เดิมของ Endpoint ทั้งดวง ไม่เปลี่ยนแปลง) │
│            │            │            │ (มี VNI)  │                                                 │
└───────────┴──────────┴───────────┴──────────┴────────────┴────────────┴──────────┴──────────┘
```

- **Outer IP (Src/Dst)**: คือ **RLOC ↔ RLOC** เสมอ (Loopback ของ Fabric Edge Node ต้นทาง-ปลายทาง
  จาก Step 373) — Underlay Route แค่ IP คู่นี้ ไม่ต้องรู้จัก EID ของ Endpoint จริงเลย
- **VXLAN Header** มีฟิลด์สำคัญคือ **VNI (Virtual Network Identifier)** — เลข 24-bit ที่ระบุว่า
  Frame นี้อยู่ใน "Virtual Network" ไหน (คล้าย VLAN ID แต่รองรับได้ถึง 16 ล้านค่า แทนที่ VLAN ที่
  จำกัดแค่ 4094)

### VNI ↔ Virtual Network (VN): กลไก Macro-Segmentation

ใน SD-Access **1 VNI ผูกกับ 1 Virtual Network (VN) เสมอ** — VN คือแนวคิดคล้าย **VRF** (แยก
Routing/Forwarding Table ออกจากกันโดยสมบูรณ์) ที่ใช้แบ่งกลุ่ม Endpoint ระดับใหญ่ (เช่น
"Employee VN" แยกจาก "IoT VN" แยกจาก "Guest VN" อย่างสิ้นเชิง ไม่มีทาง Route ข้ามกันได้เลย
นอกจากผ่าน Border Node ที่ตั้งใจ Fuse ไว้)

```
Virtual Network "EMPLOYEE_VN"  ←→  VNI 4097  ←→  แยก Routing Table (คล้าย VRF) โดยสมบูรณ์
Virtual Network "IOT_VN"       ←→  VNI 4098  ←→  แยก Routing Table (คล้าย VRF) โดยสมบูรณ์
Virtual Network "GUEST_VN"     ←→  VNI 4099  ←→  แยก Routing Table (คล้าย VRF) โดยสมบูรณ์
```

รายละเอียดเชิงลึกของ VN ในฐานะกลไก Macro-Segmentation อยู่ที่ Step 375

### SD-Access ใช้ VXLAN "ต่างจาก" VXLAN/EVPN Data Center Fabric อย่างไร

นี่คือจุดที่ข้อสอบ CCNP ENCOR ชอบทดสอบว่าเข้าใจถูกหรือสับสน — ทั้งสองใช้ **VXLAN เป็น Data
Plane เหมือนกัน** แต่ต่างกันโดยพื้นฐานที่ **Control Plane** และ **จุดประสงค์การออกแบบ**:

| หัวข้อ | SD-Access (Campus) | VXLAN/EVPN Fabric (Data Center, Part 84–85) |
|---|---|---|
| Control Plane | **LISP** (Step 373) | **MP-BGP EVPN** |
| จุดประสงค์หลัก | รองรับ Endpoint Mobility (คนเดิน, Client Roaming) + Policy-based Segmentation | รองรับ VM Mobility ระหว่าง Rack/DC + Multi-tenancy ขนาดใหญ่ |
| Topology ปกติ | Two/Three-Tier (Edge-Intermediate-Border) | Spine-Leaf (Every Leaf ↔ Every Spine) |
| Policy Plane ในตัว | **มี** — SGT ฝังไปกับ VXLAN Header โดยตรง (VXLAN-GPO) | ไม่มีในตัว EVPN เอง (ต้องพึ่งกลไกอื่น เช่น Micro-segmentation ของ ACI) |
| ผู้จัดการ (Orchestrator) | **Cisco DNA Center** | Cisco ACI APIC, หรือ Manual/NX-OS VXLAN EVPN |

> **สรุปสำหรับข้อสอบ**: เจอคำว่า "SD-Access" ให้นึกถึง **LISP + VXLAN + SGT + DNAC** เจอคำว่า
> "VXLAN/EVPN Fabric" หรือ "Data Center Fabric" เฉยๆ ให้นึกถึง **BGP EVPN** แทน — VXLAN Header
> เหมือนกันแต่ระบบที่ควบคุมมันคนละชุดกันเลย

---

## Step 375 — Macro-Segmentation (VN) vs Micro-Segmentation (SGT/TrustSec)

### ทบทวน: Traditional Segmentation ที่เราสร้างมาตลอด Part 3, 14

ตลอดหลักสูตรที่ผ่านมา เราแยก Traffic ด้วยวิธีเดียวเสมอ: **VLAN + Subnet + ACL**
([Part 3](part-003-ethernet-switching-vlan.md) สร้าง VLAN, [Part 14](part-014-access-control-lists.md)
ใช้ ACL กรอง Traffic ระหว่าง VLAN) — วิธีนี้ผูก **Policy เข้ากับ IP Address โดยตรงเสมอ**

SD-Access แยก Concept การ Segment ออกเป็น **2 ระดับที่ทำงานร่วมกัน แต่เป็นอิสระจากกัน**:

### Macro-Segmentation: Virtual Network (VN)

**VN** คือการแยก Traffic ระดับ **"ใหญ่มาก"** ด้วยการแยก Routing/Forwarding Table ทั้งชุด (คล้าย
VRF) — Endpoint ใน VN ต่างกัน **ไม่มีทาง Route ถึงกันได้เลย** เว้นแต่ผ่าน Border Node ที่ตั้งใจ
เปิดทางไว้ (เรียก Fusion Router/Route Leaking)

### Micro-Segmentation: Scalable Group Tag (SGT) / Cisco TrustSec

**SGT** คือการแยก Traffic ระดับ **"ละเอียด"** ภายใน VN เดียวกัน — แทนที่จะดูที่ IP Address, Fabric
Edge Node จะ **แปะ Tag (SGT)** ให้ Endpoint ตาม "กลุ่ม" ที่มันควรอยู่ (เช่น `EMPLOYEE_FULLTIME=10`,
`CONTRACTOR=20`, `PRINTER=30`) ตั้งแต่ตอนเข้า Fabric แล้ว SGT นี้จะ **เดินทางไปกับ Packet ทุก
Hop** (ฝังใน VXLAN Header ตาม Step 374) ทำให้ Border/Edge Node ปลายทางรู้ทันทีว่าควร Permit
หรือ Deny โดย **ไม่ต้องดู IP Address เลย** — เรียก Policy แบบนี้ว่า **SGACL (Security Group ACL)**
รายละเอียดเชิงลึกเต็มรูปแบบของ TrustSec/SGT อยู่ที่ **Part 44**

### ตารางเปรียบเทียบ

| หัวข้อ | Macro-Segmentation (VN) | Micro-Segmentation (SGT) |
|---|---|---|
| ระดับความละเอียด | หยาบ (แยกเป็น "โลกคนละใบ" เช่น Corp vs IoT vs Guest) | ละเอียด (แยกภายใน "โลกเดียวกัน" เช่น Employee vs Contractor ที่อยู่ VN เดียวกัน) |
| ใช้อะไรกำหนด | VNI (คล้าย VRF ID) | SGT (Tag ตัวเลข ผูกกับกลุ่มผู้ใช้/อุปกรณ์) |
| ผูกกับ IP/Subnet ไหม | ผูกในระดับ "VN นี้มี Subnet Pool ของตัวเอง" | **ไม่ผูกกับ IP เลย** — Endpoint ย้าย Subnet/VLAN ได้ SGT ยังติดตัวเหมือนเดิม |
| Enforcement Point | Border Node (ระหว่าง VN) | Edge Node หรือ Border Node (SGACL ระหว่าง SGT) |
| เทียบเท่าของเดิม | คล้าย VRF-Lite/MPLS VPN | คล้าย ACL แต่ไม่ต้องระบุ IP/Subnet เลยแม้แต่บรรทัดเดียว |
| ตัวอย่างคำสั่ง Policy (เชิงแนวคิดที่ Edge Node หลัง Provision) | `vlan/vrf` ต่อ VN | `cts role-based permissions from CONTRACTOR to PRINTER` (SGACL) |

### ทำไม SD-Access แก้ปัญหา Segmentation ได้ดีกว่า VLAN+ACL แบบเดิมในระดับ Scale ใหญ่

```
วิธีเดิม (Part 3/14): ต้องการแยก "Contractor" ออกจาก "Employee" ที่นั่งห้องเดียวกัน
  → ต้องสร้าง VLAN ใหม่ (เช่น VLAN 50) + Trunk VLAN 50 ทะลุทุก Switch ที่ Contractor อาจเสียบ
  → ต้องเขียน ACL แยก Subnet ของ VLAN 50 ออกจากทุก Subnet อื่นด้วยมือทุกจุด (Part 14)
  → ถ้ามี "กลุ่ม" ใหม่อีก 20 กลุ่ม = ต้องทำซ้ำ 20 รอบ (VLAN Sprawl เต็มรูปแบบ)

วิธี SD-Access: ต้องการแยก "Contractor" ออกจาก "Employee"
  → กำหนด SGT "CONTRACTOR" ให้ Endpoint กลุ่มนี้ (ผ่าน 802.1X/ISE ตอน Authenticate เข้า Fabric)
  → เขียน SGACL Policy ครั้งเดียวที่ DNA Center: "CONTRACTOR ห้ามคุยกับ EMPLOYEE_SERVERS"
  → Push ไปทุก Edge Node อัตโนมัติ — ไม่ต้องสร้าง VLAN ใหม่ ไม่ต้องแตะ Subnet เดิมเลย
  → เพิ่มกลุ่มใหม่อีก 20 กลุ่ม = เขียน SGT/Policy เพิ่ม ไม่ต้องยุ่งกับ VLAN/Subnet เลยสักบรรทัด
```

> **สรุปสำหรับข้อสอบ CCNP ENCOR**: SD-Access **ไม่ได้ยกเลิก VLAN ไปเลย** (ยังมี VLAN ภายใน
> Fabric สำหรับแต่ละ VN) แต่ทำให้ **Policy ไม่ต้องผูกกับ VLAN/Subnet อีกต่อไป** — คำถามที่ถูก
> ถามผิดบ่อยคือ "SD-Access ใช้ SGT แทน VLAN ทั้งหมด" ซึ่ง**ไม่ถูกต้อง** ที่ถูกคือ **SGT ทำงาน
> เสริม/ละเอียดกว่า VLAN โดยไม่ผูกกับ IP**

---

## Step 376 — บทบาทของ Cisco DNA Center

### DNA Center คือศูนย์กลางเดียวที่ควบคุม Fabric ทั้งหมด

**Cisco DNA Center (DNAC)** เป็น Controller/Orchestrator ที่ทำหน้าที่ **Single Source of Truth**
ตามที่กล่าวไว้ใน Step 371 — เกือบทุกอย่างที่ SD-Access ทำ **ไม่ได้เขียนผ่าน CLI โดยตรงแบบ Part
1–37 ที่ผ่านมา** แต่เขียนผ่าน **GUI Workflow** บน DNAC แล้วให้ DNAC สร้าง Config (ผ่าน NETCONF/
RESTCONF, ทบทวนได้ที่ [Part 49](part-049-network-automation.md) หากมี) ไป Push เข้าอุปกรณ์แทน

### หน้าที่หลักของ DNA Center 2 อย่างที่ต้องรู้สำหรับ Part นี้

#### 1. Underlay Automation — LAN Automation

**LAN Automation** คือ Feature ของ DNAC ที่ **สร้าง Underlay Network ให้อัตโนมัติทั้งหมด**
โดยวิศวกรไม่ต้อง SSH เข้าไป Config OSPF/IS-IS เองเลยแม้แต่ตัวเดียว:

```
ขั้นตอน LAN Automation (แนวคิด):

1. เลือก "Seed Device" 1 ตัวที่มี IP Reachability ไปยัง DNAC อยู่แล้ว (มักเป็น Border/Core)
2. DNAC สั่ง Seed Device เปิด CDP + DHCP Snooping ชั่วคราว เพื่อ "มองเห็น" Switch ข้างเคียง
   ที่ยังไม่มี Config เลย (Zero-Touch, เสียบสายแล้วเปิดเครื่องรอ)
3. DNAC แจก IP ชั่วคราวผ่าน DHCP ให้ Switch ใหม่ที่เจอ, ดึงข้อมูล Serial/Platform ผ่าน CDP
4. DNAC Push Config ให้ Switch ใหม่โดยอัตโนมัติ: ตั้ง Hostname, Loopback0,
   Underlay Routing Protocol (ค่า Default ของ LAN Automation คือ IS-IS, ดูข้อสังเกตที่ Step 377),
   และ IP บน Interface ที่เชื่อมกับ Seed Device
5. ทำซ้ำแบบ "ไล่ระดับ" ออกไปทีละ Layer จนกว่า Underlay ทั้ง Site จะ Routing ถึงกันหมด
```

#### 2. Fabric Provisioning Workflow: Design → Policy → Provision → Assurance

DNAC จัดหน้า GUI หลักออกเป็น 4 ขั้นตอนใหญ่ตามลำดับที่วิศวกรต้องทำจริง:

| ขั้นตอน | ทำอะไร | ตัวอย่างงานจริงที่คลิกใน GUI |
|---|---|---|
| **1. Design** | สร้าง Network Hierarchy (Site/Building/Floor), กำหนด IP Address Pool, Device Credential Template | สร้าง Site "Bangkok HQ" → Floor "Floor 3" → กำหนด IP Pool `10.10.10.0/24` ให้ Floor นี้ |
| **2. Policy** | กำหนด Virtual Network (VN), สร้าง Scalable Group (SGT), เขียน SGACL Policy ระหว่างกลุ่ม | สร้าง VN "EMPLOYEE_VN", SGT "CONTRACTOR"/"EMPLOYEE", Policy "Contractor ห้ามเข้า Server VN" |
| **3. Provision** | เลือกอุปกรณ์จริงจาก Inventory, กำหนด Fabric Role (Edge/Border/Control Plane/WLC) ให้แต่ละตัว, สั่ง Deploy | เลือก ACCESS-SW1 → Assign Role "Fabric Edge Node" → กด Deploy |
| **4. Assurance** | Monitor สุขภาพเครือข่าย, ดู Client Health, ทำ Path Trace ตรวจสอบปัญหา | ดู Dashboard "Fabric Health", คลิก Path Trace ระหว่าง PC1↔Server1 |

```
┌─────────┐     ┌─────────┐     ┌───────────┐     ┌────────────┐
│  DESIGN  │ ──► │  POLICY  │ ──► │ PROVISION  │ ──► │ ASSURANCE   │
│ (โครงสร้าง│     │ (VN/SGT/ │     │ (Fabric     │     │ (Monitor/   │
│  Site/IP) │     │  SGACL)  │     │  Role Push) │     │  Path Trace)│
└─────────┘     └─────────┘     └───────────┘     └────────────┘
     ▲                                                     │
     └─────────────────── วนกลับเมื่อต้องแก้ Design ──────────┘
```

> **สำหรับข้อสอบ CCNP ENCOR**: ข้อสอบจะถามลำดับ **Design → Policy → Provision → Assurance**
> เป๊ะๆ บางครั้ง ให้จำเป็นลำดับตายตัว — Provision **มาก่อน** ไม่ได้เพราะยังไม่มี IP Pool/Policy
> ให้ผูก และ Assurance เป็นขั้นตอนสุดท้ายเสมอ (Monitor สิ่งที่ Provision ไปแล้ว)

---

## Step 377 — Underlay vs Overlay: ทำไม Part 33 คือ Underlay ที่ใช้ซ้ำได้ทันที

### นิยาม: Underlay คือ Network ชั้นล่างที่ "โง่" โดยตั้งใจ

**Underlay** ของ SD-Access คือ Network ระดับ IP Routing ธรรมดาที่สุด — หน้าที่เดียวของมันคือ
**ให้ทุก Fabric Node (Loopback ของ Edge/Border/Control Plane Node) Route ถึงกันได้หมด**
Underlay **ไม่รู้จัก VXLAN, ไม่รู้จัก LISP, ไม่รู้จัก SGT, ไม่รู้จัก VN เลยแม้แต่น้อย** — มันมองเห็นแค่
IP Packet ธรรมดาที่วิ่งผ่าน (ซึ่งจริงๆ คือ VXLAN-encapsulated Packet จาก Overlay แต่ Underlay
ไม่สนใจข้างในเลย มองเป็น UDP Packet ธรรมดา)

### นิยาม: Overlay คือ Network เสมือนที่ "ฉลาด" ซ้อนอยู่บน Underlay

**Overlay** คือทุกอย่างที่เราเรียนใน Step 371–376 (LISP + VXLAN + SGT ที่ DNAC ควบคุม) — Overlay
"ขี่" อยู่บน Underlay โดยใช้ RLOC (ซึ่งก็คือ Loopback Address ของ Fabric Node) เป็นจุดเชื่อม

```
┌─────────────────────────────────────────────────────────────┐
│                       OVERLAY (Fabric)                        │
│   LISP (Control) + VXLAN (Data) + SGT (Policy)                │
│   ขี่อยู่บน RLOC-to-RLOC Tunnel                                 │
└───────────────────────────┬─────────────────────────────────┘
                             │  ใช้ RLOC (Loopback0) เป็นจุดเชื่อม
┌───────────────────────────┴─────────────────────────────────┐
│                       UNDERLAY (Routed Network)                │
│   แค่ IP Routing ธรรมดา (OSPF/IS-IS/EIGRP) ให้ Loopback         │
│   ของทุก Fabric Node เห็นกันหมด — ไม่รู้จัก VXLAN/LISP/SGT เลย  │
└─────────────────────────────────────────────────────────────┘
```

### ข่าวดี: Part 33 สร้าง Underlay แบบนี้ไว้ให้แล้วโดยไม่รู้ตัว!

นี่คือจุดเชื่อมที่สำคัญที่สุดของ Part นี้ — ย้อนกลับไปดู [Part 33](part-033-advanced-stp-campus-design.md)
Step 321–330 เราทำสิ่งต่อไปนี้ไว้แล้วทั้งหมด:

| สิ่งที่ Part 33 ทำไว้แล้ว | ตรงกับความต้องการของ SD-Access Underlay อย่างไร |
|---|---|
| ทุก Backbone Link เป็น **Routed Port** (`no switchport`) ไม่ใช่ Trunk | Underlay ต้องเป็น Pure L3 เท่านั้น — ตรงเป๊ะ |
| ทุกอุปกรณ์มี **Loopback0** พร้อม Router-ID ที่แน่นอน ([`00-ip-address-plan.md`](00-ip-address-plan.md)) | RLOC ของ Fabric Node **คือ Loopback0 นี่แหละ** ไม่ต้องสร้างใหม่ |
| **OSPF Area 0** ครอบคลุมทุก Backbone Link พร้อม Reachability เต็มรูปแบบ | Underlay Routing Protocol พร้อมใช้ทันที |
| **ECMP** (`maximum-paths 4`) ระหว่าง DIST↔Core ทุกเส้น | VXLAN Tunnel ระหว่าง RLOC จะได้ Load-balance ผ่าน ECMP โดยอัตโนมัติ ไม่ต้อง Config เพิ่ม |
| `auto-cost reference-bandwidth 100000` ปรับ Cost ให้ตรงกับ 10G/40G/100G จริง | Underlay เลือกเส้นทางตามความเร็วจริง ไม่ใช่ปัดเท่ากันหมดแบบ Default |

**สรุป**: ถ้าจะ Migrate Lab ของหลักสูตรนี้ไปเป็น SD-Access จริง (สมมติในเชิงแนวคิดที่ Step 380)
**Underlay จาก Part 33 ใช้ได้ทันทีโดยไม่ต้องแก้ Config OSPF แม้แต่บรรทัดเดียว** — งานที่ต้อง
ทำเพิ่มมีแค่ **ชั้น Overlay** (เปิด LISP, กำหนด Fabric Role, Provision ผ่าน DNAC) เท่านั้น

### ข้อควรระวังทางเทคนิค 1 ข้อที่ต้องเพิ่มเมื่อทำ Overlay จริง: MTU

VXLAN Encapsulation เพิ่ม Overhead ให้ Packet ประมาณ **50 byte** (Outer Eth + IP + UDP + VXLAN
Header) — ถ้า Underlay ยังใช้ MTU มาตรฐาน 1500 byte เท่าเดิม Packet ที่เดิมขนาดพอดี 1500 byte
ของ Endpoint จะกลายเป็น **1550 byte หลัง Encapsulate** และถูก Fragment หรือ Drop ที่ Underlay
ได้ — **Best Practice ของ Cisco คือปรับ Underlay ทุก Interface ให้รองรับ Jumbo MTU (แนะนำ 9100
byte)** ก่อนเปิด Fabric จริง ซึ่งเป็นการปรับเพิ่มเพียงจุดเดียวที่ Part 33 ยังไม่ได้ทำไว้ (Part 33
ไม่จำเป็นต้องทำเพราะยังไม่มี Encapsulation Overhead ใดๆ ในตอนนั้น)

> **สำหรับข้อสอบ CCNP ENCOR**: จำคู่นี้ให้แม่น — **Underlay = Routing ธรรมดา (มองไม่เห็น Fabric)**,
> **Overlay = LISP+VXLAN+SGT (ขี่บน Underlay ผ่าน RLOC)** และ **Underlay ต้องเพิ่ม MTU รองรับ
> VXLAN Overhead ~50 byte เสมอ**

---

## Step 378 — SD-Access Wireless: WLC-1 และ AP กลายเป็น Fabric-enabled อย่างไร

### ทบทวน Wireless แบบ Traditional ที่ Lab ของเราใช้อยู่ (Part 20, 36–37)

ใน Traditional Wireless (ที่ WLC-1 ของเราใช้อยู่ตั้งแต่ [Part 20](part-020-wireless-fundamentals.md))
AP ทุกตัวส่ง Traffic 2 ประเภทกลับไปที่ WLC ผ่าน **CAPWAP Tunnel**:

| ประเภท Tunnel | เนื้อหา |
|---|---|
| CAPWAP **Control** Tunnel | คำสั่งบริหาร AP (RRM, Configuration, Keepalive) |
| CAPWAP **Data** Tunnel | **Traffic จริงของ Wireless Client ทั้งหมด** ถูกส่งกลับไป "Terminate" ที่ WLC เสมอ (Centralized/Local Mode ตามที่ Part 20 สอน) |

ปัญหาของโมเดลนี้คือ Client Traffic ทุก Byte ต้องเดินทางไป-กลับผ่าน WLC เสมอ แม้ Client กับปลายทาง
จะอยู่ใกล้กันมากในเครือข่ายจริงก็ตาม (เรียกว่า "Traffic Tromboning")

### SD-Access Wireless: แยก Control Plane กับ Data Plane ของ AP ออกจากกัน

เมื่อ Integrate WLC-1 เข้า Fabric กลายเป็น **Fabric WLC** และ AP กลายเป็น **Fabric AP**
สถาปัตยกรรมจะเปลี่ยนไปดังนี้:

```
Traditional Wireless (Part 20/36-37):
  AP ──CAPWAP Control──► WLC-1  (คำสั่งบริหาร)
  AP ──CAPWAP Data──────► WLC-1  (Client Traffic ทั้งหมด "อ้อม" ไป WLC ก่อนออกสู่เครือข่าย)

SD-Access Wireless (Fabric-enabled):
  Fabric AP ──CAPWAP Control (คงอยู่)──► Fabric WLC   (คำสั่งบริหาร/RRM เหมือนเดิม)
  Fabric AP ──VXLAN (ใหม่!)────────────► Fabric Edge Node ที่ AP เสียบอยู่
                                            (Client Traffic เข้า Fabric ตรงจาก Edge Node ทันที
                                             ไม่ต้องอ้อมไป WLC เลย)
```

### รายละเอียดที่เปลี่ยนไป

| องค์ประกอบ | Traditional | SD-Access Wireless (Fabric-enabled) |
|---|---|---|
| CAPWAP **Control** Tunnel (AP↔WLC) | มี | **ยังมีเหมือนเดิม** — WLC ยังบริหาร AP (RRM, Config) แบบเดิมทุกประการ |
| CAPWAP **Data** Tunnel (AP↔WLC) | มี — Client Traffic วิ่งผ่านตรงนี้เสมอ | **ไม่มีอีกต่อไป** — ถูกแทนที่ด้วย VXLAN |
| เส้นทาง Client Traffic จริง | AP → WLC → เครือข่าย | **AP → Fabric Edge Node ที่ AP เสียบอยู่โดยตรง** (Local ที่ Edge, ไม่อ้อม WLC) |
| WLC ทำหน้าที่อะไรต่อ Data Plane | Terminate Data Tunnel เอง | เป็นแค่ Control-plane function — ลงทะเบียน Wireless Client เป็น **EID ใหม่ใน LISP** ผ่าน Fabric Control Plane Node |
| Client Roaming ข้าม AP/Edge Node | ต้องพึ่ง Mobility Tunnel ระหว่าง WLC (Part 37 อาจกล่าวถึง Inter-Controller Mobility) | **ใช้กลไก LISP EID/RLOC ล้วนๆ** (Step 373) — Edge Node ใหม่ Register EID ของ Client ทันทีที่ Roam มา ไม่ต้องมี Mobility Tunnel ระหว่าง WLC แบบเดิมเลย |
| ชื่อเรียก AP | Access Point (ธรรมดา) | **Fabric AP** — ทำหน้าที่คล้าย Extension ของ Fabric Edge Node ในมุม Data Plane |

### Diagram: เส้นทาง Client Traffic เปรียบเทียบ

```
Traditional:
  Wireless Client ──► AP ══CAPWAP Data══► WLC-1 ──► Core/DIST ──► ปลายทาง
                                (Traffic ทุก Byte ต้องผ่าน WLC ก่อนเสมอ)

SD-Access Wireless:
  Wireless Client ──► Fabric AP ══VXLAN══► Fabric Edge Node (ตัวที่ AP เสียบอยู่)
                                                    │
                                                    ▼ (Local Switching/Routing ทันที
                                                       ผ่าน Overlay ปกติ Step 373-374)
                                                  ปลายทาง

  (CAPWAP Control ยังวิ่งคู่กันไปหา Fabric WLC เพื่อบริหาร AP เท่านั้น — เส้นบางๆ ไม่ได้แบก Client
   Traffic แล้ว)
```

### ทำไมสิ่งนี้ถึงสำคัญ

1. **ลด Traffic Tromboning** — Client Traffic ไม่ต้องอ้อมไป WLC ก่อนเสมอ ลด Latency และภาระของ
   WLC ลงมาก
2. **Roaming ไร้รอยต่อจริง** — ใช้ LISP EID/RLOC Mobility (Step 373) แทน Mobility Tunnel ที่ WLC
   ต้องดูแลเอง — Client เปลี่ยน AP/Edge Node ได้โดย IP ไม่เปลี่ยนแม้ข้าม Subnet เดิม
3. **Policy เดียวกันทั้ง Wired และ Wireless** — SGT (Step 375) ใช้ Tag Wireless Client ได้เหมือน
   Wired Client เป๊ะๆ ทำให้ Policy Consistency ครอบคลุมทั้งสองโลกในระบบเดียว

> **สำหรับข้อสอบ CCNP ENCOR**: จำให้แม่นว่า **CAPWAP Control ไม่ได้หายไปใน SD-Access Wireless**
> — สิ่งที่ถูกแทนที่คือ **CAPWAP Data Tunnel เท่านั้น** ที่ถูกแทนด้วย VXLAN ไปยัง Fabric Edge Node

---

## Step 379 — Migration Considerations: การย้ายจาก Traditional Campus สู่ SD-Access

### ความจริงในองค์กรจริง: ไม่มีใคร "รื้อทิ้งแล้วสร้างใหม่ทั้งหมด"

Enterprise จริงที่มี Campus ทำงานอยู่แล้ว (เหมือน Lab ของหลักสูตรนี้ Part 1–35) **ไม่มีทางหยุด
Business เพื่อรื้อเครือข่ายทั้งหมดทิ้งแล้วสร้าง SD-Access ใหม่ในคราวเดียว** — Cisco จึงออกแบบ
เส้นทาง Migration แบบ **Incremental (ทีละส่วน)** ไว้เสมอ โดยมีแนวคิดหลัก 2 อย่าง

### แนวทาง Migration แบบ Incremental

```
Phase 1: สร้าง Fabric เล็กๆ คู่ขนานกับ Traditional Network เดิม (Site ใหม่/Floor ใหม่ก่อน)
Phase 2: ค่อยย้าย Site/Floor เดิมเข้า Fabric ทีละส่วน (เริ่มจาก Site ที่มี Hardware รองรับก่อน)
Phase 3: Site ที่ Hardware ยังไม่รองรับ (เช่น Switch รุ่นเก่า) → ใช้ Extended Node เชื่อมเข้า
          Fabric ก่อน โดยยังไม่ต้องเปลี่ยน Hardware ทันที
Phase 4: ทยอย Upgrade Hardware ตามรอบ Lifecycle ปกติ แล้วเปลี่ยน Extended Node เป็น
          Fabric Edge Node เต็มรูปแบบทีละตัว
```

### Extended Node และ Policy Extended Node: ทางออกสำหรับ Switch ที่ไม่รองรับ Fabric

นี่คือคำตอบสำคัญสำหรับคำถามที่ทิ้งไว้ที่ Step 372/380 — **ACCESS-SW1–4 ของ Lab เรา เป็น Catalyst
9200/2960 ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md) กำหนด ซึ่งเป็น Switch ที่**
**ไม่รองรับ VXLAN/LISP ทางกายภาพ** — Switch แบบนี้ **ไม่สามารถเป็น Fabric Edge Node ได้จริง**
Cisco จึงมี Role พิเศษสำหรับกรณีนี้โดยเฉพาะ:

| Role | รองรับ VXLAN เองไหม | ทำหน้าที่อะไร | Enforce SGACL ได้ไหม |
|---|---|---|---|
| **Extended Node (EN)** | ❌ ไม่รองรับ | เชื่อมต่อเข้า Fabric ผ่าน Trunk ไปยัง Fabric Edge Node ตัวจริง (Onboard ผ่าน DNAC PnP) — ให้ Port เพิ่มสำหรับ Endpoint โดยไม่ต้องมี Hardware ใหม่ | ❌ ไม่ได้ — Traffic ต้องวิ่งไปถึง Edge Node ตัวจริงก่อนจึง Enforce Policy ได้ |
| **Policy Extended Node (PEN)** | ❌ ไม่รองรับ (เหมือน EN) | เหมือน Extended Node ทุกประการ แต่ใช้ Hardware/Software รุ่นที่สูงกว่าเล็กน้อย | ✅ **ได้** — Enforce SGACL ได้ตั้งแต่ที่ตัวมันเอง ก่อน Traffic ไปถึง Edge Node ตัวจริง |

```
                    ┌─────────────────────┐
                    │  Fabric Edge Node     │  <- VXLAN/LISP Capable จริง (เช่น Catalyst 9300)
                    │  (ตัวจริงของ Fabric)  │
                    └──────────┬────────────┘
                          Trunk (802.1Q, ไม่ใช่ VXLAN)
                    ┌──────────┴────────────┐
                    │  Extended Node /        │  <- Switch รุ่นเก่าที่ยังใช้อยู่ (เช่น 2960/9200
                    │  Policy Extended Node   │     บางรุ่นที่ไม่รองรับ Edge Node เต็มรูปแบบ)
                    └──────────┬────────────┘
                            PC/Server เสียบตรงนี้
                    (Traffic ถูกส่งผ่าน Trunk ขึ้นไปให้ Fabric Edge Node ตัวจริง
                     ทำ VXLAN Encapsulation แทนให้ทั้งหมด)
```

### Mapping Migration กับ Lab ของหลักสูตรนี้ (เชิงแนวคิด)

| อุปกรณ์ | สถานะ Hardware ปัจจุบัน | Role ที่เป็นไปได้จริงถ้า Migrate วันนี้ | ต้องทำอะไรถึงเป็น Fabric Edge Node เต็มรูปแบบ |
|---|---|---|---|
| DIST-SW1–4 (Catalyst 9300) | รองรับ VXLAN/LISP (L3-capable) | **Fabric Edge Node ได้จริง** (ไม่ต้อง Upgrade) | ไม่ต้องทำอะไรเพิ่มด้าน Hardware |
| ACCESS-SW1–4 (Catalyst 9200/2960) | ขึ้นกับรุ่น 9200 ย่อย — 2960 **ไม่รองรับแน่นอน** | **Extended Node** (เชื่อมเข้า Fabric ผ่าน Trunk ไปยัง DIST-SW ที่เป็น Edge Node) | ต้อง Upgrade เป็น Catalyst 9300/9200 รุ่นที่รองรับ Edge Node เต็มรูปแบบ |

สังเกตว่า Mapping นี้ **ต่างจาก Step 372/380 ที่สอนแบบง่ายว่า "ACCESS-SW = Edge Node"** —
นั่นเป็นการสอนแนวคิด Role แบบทั่วไป (Edge Node อยู่ตำแหน่ง Access Layer โดยหลักการ) แต่ในทาง
ปฏิบัติจริงกับ Hardware ที่หลักสูตรนี้กำหนดไว้ **DIST-SW (Catalyst 9300) ต่างหากที่มีคุณสมบัติ
เพียงพอจะเป็น Edge Node ได้จริงในวันนี้** ส่วน ACCESS-SW ต้องผ่าน Extended Node ไปก่อน — นี่คือ
เหตุผลที่ Step 380 จะระบุ Caveat นี้ไว้ชัดเจนในตาราง Mapping เต็มรูปแบบ

> **สำหรับข้อสอบ CCNP ENCOR**: **Policy Extended Node ต่างจาก Extended Node ตรงที่ Enforce
> SGACL ได้เอง** — คำถามแนว "Switch ตัวไหน Enforce Policy ก่อน Traffic ไปถึง Fabric Edge Node
> ตัวจริง" ตอบ **Policy Extended Node** เท่านั้น ธรรมดา Extended Node ทำไม่ได้

---

## Step 380 — Lab เชิงแนวคิด: แผนผัง Fabric Role เต็มรูปแบบ

> **ข้อจำกัดของ Lab นี้**: SD-Access ของจริงต้องมี **Cisco DNA Center** ทำงานอยู่จริง ซึ่งเป็น
> Physical/Virtual Appliance แยกต่างหาก **ไม่ได้อยู่ใน CLI-based Lab (Packet Tracer/CML/GNS3)**
> ที่หลักสูตรนี้ใช้ตั้งแต่ [Part 1 Step 10](part-001-networking-fundamentals.md) — ดังนั้น Lab ของ
> Part นี้จึงเป็น **Design-level/Conceptual Lab** เท่านั้น (แผนผัง Role + Reachability Table +
> Workflow ที่วิศวกรจะคลิกจริงบน DNAC) ไม่ใช่ CLI จริงที่ Config ได้ในโปรแกรม Lab ของท่าน — Lab
> CLI-based ของหลักสูตรหลัก (Part 1–35, ขยายต่อ Part 39+) ยังคงใช้สถาปัตยกรรม Traditional ตามปกติ

### แผนผังแบ่ง Fabric Role เต็มรูปแบบ (รวม Caveat จาก Step 379)

| Device | Role ใน 3-Tier เดิม | Fabric Role ที่ตั้งใจสอน (Conceptual) | ความเป็นไปได้จริงกับ Hardware ปัจจุบัน |
|---|---|---|---|
| CORE-SW1 | Core | **Fabric Border Node** (External Border — เชื่อมสู่ WAN-EDGE/FW-1/Internet) + **Fabric Control Plane Node** (ร่วม) | ✅ Catalyst 9500 รองรับได้จริง |
| CORE-SW2 | Core | **Fabric Border Node** (สำรอง, Redundant Pair กับ CORE-SW1) + **Fabric Control Plane Node** (ร่วม, สำรอง) | ✅ Catalyst 9500 รองรับได้จริง |
| DIST-SW1 | Distribution | **Intermediate Node** (ตามแนวคิดง่ายใน Step 372) **หรือ Fabric Edge Node** ถ้าต้องการให้ Endpoint เชื่อมต่อที่ Layer นี้เลย | ✅ Catalyst 9300 รองรับทั้ง 2 บทบาท |
| DIST-SW2 | Distribution | Intermediate Node / Fabric Edge Node (Pattern เดียวกับ DIST-SW1) | ✅ Catalyst 9300 |
| DIST-SW3 | Distribution | Intermediate Node / Fabric Edge Node | ✅ Catalyst 9300 |
| DIST-SW4 | Distribution | Intermediate Node / Fabric Edge Node | ✅ Catalyst 9300 |
| ACCESS-SW1 | Access | **Fabric Edge Node** (ตามแนวคิดที่สอนใน Step 372) | ⚠️ **ต้องตรวจสอบรุ่นจริง** — ถ้าเป็น 2960 ต้องใช้ **Extended Node** แทน (Step 379) |
| ACCESS-SW2 | Access | Fabric Edge Node (Pattern เดียวกับ ACCESS-SW1) | ⚠️ เหมือน ACCESS-SW1 |
| ACCESS-SW3 | Access | Fabric Edge Node | ⚠️ เหมือน ACCESS-SW1 |
| ACCESS-SW4 | Access | Fabric Edge Node | ⚠️ เหมือน ACCESS-SW1 |
| WLC-1 | Wireless Controller | **Fabric WLC** | ✅ Catalyst 9800-CL รองรับ SD-Access Wireless เต็มรูปแบบ |
| AP-1, AP-2 | Access Point | **Fabric AP** | ✅ Catalyst 9130 รองรับ |

### Underlay Reachability Table (ใช้ซ้ำจาก Part 33 ทั้งหมด + เพิ่ม Loopback ส่วนขยายเชิงแนวคิด)

ตารางนี้ **ไม่แก้ไข** [`00-ip-address-plan.md`](00-ip-address-plan.md) แต่ **ขยาย** ด้วยแนวคิด
เดียวกับที่ [Part 33](part-033-advanced-stp-campus-design.md) ทำไว้ — ในระบบจริง Loopback ของ
ACCESS-SW/WLC-1 เหล่านี้จะถูก **DNAC LAN Automation สร้างให้อัตโนมัติ** (Step 376) ไม่ต้องกำหนด
ด้วยมือ ค่าด้านล่างจึงเป็นตัวอย่างเชิงแนวคิดเพื่อให้เห็นภาพ RLOC ของทุก Fabric Node ครบถ้วน:

| Device | Fabric Role | Loopback0 (RLOC) | Reachability ผ่าน Underlay |
|---|---|---|---|
| CORE-SW1 | Border + Control Plane | 1.1.1.1/32 (จาก [`00-ip-address-plan.md`](00-ip-address-plan.md)) | OSPF Area 0 (Part 33) |
| CORE-SW2 | Border + Control Plane | 1.1.1.2/32 | OSPF Area 0 (Part 33) |
| DIST-SW1 | Intermediate/Edge | 1.1.1.11/32 | OSPF Area 0, ECMP ผ่าน CORE-SW1+CORE-SW2 (Part 33 Step 327) |
| DIST-SW2 | Intermediate/Edge | 1.1.1.12/32 | OSPF Area 0, ECMP |
| DIST-SW3 | Intermediate/Edge | 1.1.1.13/32 | OSPF Area 0, ECMP |
| DIST-SW4 | Intermediate/Edge | 1.1.1.14/32 | OSPF Area 0, ECMP |
| ACCESS-SW1 | Edge (หรือ Extended Node) | 1.1.1.101/32 *(ตัวอย่างเชิงแนวคิด — LAN Automation จะกำหนดจริง)* | ผ่าน DIST-SW1 |
| ACCESS-SW2 | Edge (หรือ Extended Node) | 1.1.1.102/32 *(ตัวอย่างเชิงแนวคิด)* | ผ่าน DIST-SW2 |
| ACCESS-SW3 | Edge (หรือ Extended Node) | 1.1.1.103/32 *(ตัวอย่างเชิงแนวคิด)* | ผ่าน DIST-SW3 |
| ACCESS-SW4 | Edge (หรือ Extended Node) | 1.1.1.104/32 *(ตัวอย่างเชิงแนวคิด)* | ผ่าน DIST-SW4 |
| WLC-1 | Fabric WLC | 1.1.1.50/32 *(ตัวอย่างเชิงแนวคิด)* | ผ่าน DIST-SW3/DIST-SW4 (ตาม Access-Layer Mapping เดิม) |

> ลิงก์ Backbone ทั้งหมดที่ RLOC เหล่านี้ใช้เดินทางถึงกัน **คือลิงก์ตาราง "Core / Distribution
> Backbone" เดียวกันกับใน [`00-ip-address-plan.md`](00-ip-address-plan.md) และตาราง Subnet ส่วน
> ขยายจาก [Part 33 Step 321](part-033-advanced-stp-campus-design.md) ทุกประการ** — ไม่มีการเพิ่ม
> Subnet ใหม่สำหรับ Underlay เลย เพราะ Underlay เดิมครอบคลุมความต้องการของ Fabric แล้วสมบูรณ์
> (ตามที่พิสูจน์ไว้ที่ Step 377)

### เรื่องเล่า DNAC Provisioning Workflow (สิ่งที่วิศวกรจะคลิกจริงถ้ามี DNAC)

```
1. [Design] สร้าง Site Hierarchy: "Bangkok-HQ" → Building → Floor ตรงกับตำแหน่งจริงของ
   CORE-SW1/2, DIST-SW1-4, ACCESS-SW1-4 ในตึก แล้วผูก IP Pool ตาม VLAN เดิมจาก
   00-ip-address-plan.md (VLAN 10/20/30/40/99) เข้ากับ Floor ที่เกี่ยวข้อง

2. [Policy] สร้าง Virtual Network:
     - "CORP_VN" ครอบ VLAN 10 (SALES), 20 (VOICE), 30 (SERVERS)
     - "WIFI_VN" ครอบ VLAN 40 (WIFI)
   สร้าง Scalable Group: EMPLOYEE, CONTRACTOR, SERVER, PRINTER
   เขียน SGACL: "CONTRACTOR ปฏิเสธเข้าถึง SERVER" (ตัวอย่าง Policy อ้างอิง Step 375)

3. [Provision] เข้าหน้า Fabric Site Provisioning:
     - เพิ่ม CORE-SW1, CORE-SW2 → Assign Role "Border Node" + "Control Plane Node"
     - เพิ่ม DIST-SW1-4 → Assign Role "Edge Node" (ตาม Mapping ในตารางบน)
     - เพิ่ม ACCESS-SW1-4 → ตรวจสอบรุ่นจริงก่อน ถ้าไม่รองรับ Assign เป็น "Extended Node"
       ผ่าน Fabric Edge Node ที่ใกล้ที่สุด (DIST-SW ของฝั่งนั้น)
     - เพิ่ม WLC-1 → Assign Role "Fabric WLC" ผูกกับ Fabric Site เดียวกัน
     - กด "Add to Fabric" แล้ว "Deploy" — DNAC จะสร้าง Config LISP/VXLAN ที่จำเป็นทั้งหมดให้เอง

4. [Assurance] เปิด Dashboard "Fabric Health" ตรวจสอบว่าทุก Node ขึ้นสถานะ "Healthy" แล้วใช้
   "Path Trace" ทดสอบเส้นทางจาก PC1 (หลัง ACCESS-SW1) ไปยัง Server1 (หลัง ACCESS-SW3) ว่า
   วิ่งผ่าน Fabric ตามที่ออกแบบไว้จริง
```

### คำสั่ง `show` ที่วิศวกรจะใช้ตรวจสอบ LISP/Fabric Status หลัง Provision เสร็จ

แม้ SD-Access ควบคุมผ่าน DNAC เป็นหลัก แต่วิศวกรยังต้อง SSH เข้า CLI ของอุปกรณ์เพื่อ Verify
สถานะจริงในระดับ Data Plane/Control Plane ได้เสมอ (เหมือนหลักการ Verification ทุก Part ก่อนหน้า):

```
! บน Fabric Edge Node — ตรวจสอบ LISP Session ไปยัง Control Plane Node
Edge-Node# show lisp session
Sessions for VRF default, total: 2, established: 2
Peer                State      Up/Down       In/Out    Users
1.1.1.1              Up         1w2d          15/12     3
1.1.1.2              Up         1w2d          15/12     3

! ตรวจสอบว่า EID ใดถูก Register ไว้กับ Control Plane Node แล้ว
Edge-Node# show lisp dynamic-eid all
LISP Dynamic EID Information for VRF "default"
EID Prefix              Uptime      Last Used
10.10.30.10/32          00:15:22    00:00:03

! บน Control Plane Node (CORE-SW1/CORE-SW2) — ดูรายชื่อ Edge Node ทั้งหมดใน Fabric
Control-Plane-Node# show fabric edge-node
Fabric Edge Node Summary
Node                 RLOC            State      Site
ACCESS-SW1           1.1.1.101       Active     Bangkok-HQ/Floor3
ACCESS-SW2           1.1.1.102       Active     Bangkok-HQ/Floor3
ACCESS-SW3           1.1.1.103       Active     Bangkok-HQ/Floor3
ACCESS-SW4           1.1.1.104       Active     Bangkok-HQ/Floor3

! ตรวจสอบ SGT ที่ Endpoint แต่ละตัวถูก Tag ไว้ (Policy Plane, Step 375)
Edge-Node# show cts role-based sgt-map all
Active IP-SGT Bindings Information
IP Address     SGT      Source
10.10.10.5     10       LOCAL (802.1X)
10.10.30.10    30       LOCAL (Static)
```

> **หมายเหตุ**: Output ข้างบนเป็นตัวอย่างรูปแบบคำสั่งจริงบน IOS-XE สำหรับ SD-Access แต่ **ค่า
> ตัวเลขเป็นข้อมูลจำลองเพื่อการสอนเท่านั้น** เนื่องจากไม่มี DNA Center จริงมา Provision Fabric
> ให้ Lab นี้ตามที่อธิบายไว้ในหมายเหตุต้น Step

---

## แบบฝึกหัดทวนความเข้าใจ Part 38

1. LISP แยก EID และ RLOC ออกจากกัน — EID คืออะไร, RLOC คืออะไร และทำไมการแยกนี้ถึงทำให้
   Host Mobility เป็นไปได้โดยไม่ต้อง Renumber IP?
2. อธิบายความแตกต่างระหว่าง Macro-Segmentation (Virtual Network) กับ Micro-Segmentation
   (SGT/TrustSec) — ทำไม SD-Access ต้องมีทั้งสองระดับพร้อมกัน ไม่ใช้แค่ระดับเดียว?
3. Underlay ของ SD-Access "รู้จัก" หรือ "ไม่รู้จัก" VXLAN/LISP/SGT? และทำไม Part 33 ที่สร้าง
   OSPF ECMP Underlay ไว้แล้ว จึงถือว่าเป็น SD-Access Underlay ที่ใช้งานได้ทันที?
4. ใน SD-Access Wireless, CAPWAP Tunnel ประเภทใดที่ยังคงอยู่เหมือนเดิม และประเภทใดที่ถูก
   แทนที่ด้วย VXLAN?
5. Extended Node กับ Policy Extended Node ต่างกันอย่างไร และเกี่ยวข้องอย่างไรกับ ACCESS-SW1–4
   ของ Lab หลักสูตรนี้ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md) กำหนด Hardware ไว้?

**เฉลย:**

1. **EID (Endpoint Identifier)** คือ IP ตัวตนจริงของ Endpoint ที่ไม่เปลี่ยนไม่ว่าจะย้ายไปที่ไหน
   **RLOC (Routing Locator)** คือ IP ของ Fabric Edge Node ที่ Endpoint นั้นเชื่อมต่ออยู่ ณ ขณะนี้
   ซึ่งเปลี่ยนได้ตามตำแหน่งจริง — เมื่อ Endpoint ย้าย Fabric Edge Node ใหม่จะ Register EID เดิม
   ไปที่ Map-Server ว่าตอนนี้ RLOC เปลี่ยนไปแล้ว โดย **EID ไม่ต้องเปลี่ยนเลย** ทำให้ Application/
   DNS ที่ผูกกับ EID เดิมยังใช้งานได้ต่อเนื่องแม้ Endpoint ย้ายข้าม Subnet ทางกายภาพจริง
2. Macro-Segmentation (VN) แยก Traffic ระดับหยาบด้วยการแยก Routing Table ทั้งชุด (คล้าย VRF)
   ทำให้กลุ่มใหญ่ๆ (เช่น Corp vs IoT vs Guest) ไม่มีทาง Route ถึงกันได้เลยนอกจากตั้งใจเปิดทาง
   ส่วน Micro-Segmentation (SGT) แยก Traffic ระดับละเอียดภายใน VN เดียวกัน โดยไม่ผูกกับ IP/
   Subnet เลย — SD-Access ต้องมีทั้งสองระดับเพราะ VN ให้ Isolation ระดับใหญ่ที่ปลอดภัยสูงสุด
   (เหมาะกับข้อมูลที่ต้องแยกเด็ดขาด) ส่วน SGT ให้ความละเอียดในการกำหนด Policy ภายในกลุ่ม
   เดียวกันโดยไม่ต้องเพิ่ม VLAN/Subnet ใหม่ทุกครั้งที่มีกลุ่มใหม่
3. Underlay **ไม่รู้จัก** VXLAN/LISP/SGT เลย — มันเป็นแค่ Routed IP Network ที่ให้ Loopback ของ
   Fabric Node ทุกตัว Route ถึงกันได้เท่านั้น Part 33 สร้าง Routed Port ล้วน + Loopback0 ทุกตัว +
   OSPF Area 0 + ECMP ไว้ครบตามที่ SD-Access Underlay ต้องการทุกข้อ จึงใช้ซ้ำได้ทันทีโดยไม่ต้อง
   แก้ Config OSPF เลย (ต้องเพิ่มแค่ Jumbo MTU รองรับ VXLAN Overhead ~50 byte เท่านั้น)
4. **CAPWAP Control Tunnel** (AP↔WLC สำหรับบริหาร AP, RRM, Configuration) **ยังคงอยู่เหมือนเดิม**
   ส่วน **CAPWAP Data Tunnel** (ที่เดิมแบก Client Traffic กลับไป Terminate ที่ WLC) **ถูกแทนที่
   ด้วย VXLAN** จาก Fabric AP ไปยัง Fabric Edge Node ที่ AP เสียบอยู่โดยตรง ทำให้ Client Traffic
   ไม่ต้องอ้อมไปที่ WLC อีกต่อไป
5. **Extended Node** เชื่อมต่อเข้า Fabric ผ่าน Trunk ไปยัง Fabric Edge Node ตัวจริง แต่ **Enforce
   SGACL ไม่ได้เอง** (ต้องรอ Traffic ไปถึง Edge Node ตัวจริงก่อน) ส่วน **Policy Extended Node**
   ทำหน้าที่เหมือนกันแต่ **Enforce SGACL ได้เองตั้งแต่ตัวมัน** เพราะใช้ Hardware/Software รุ่น
   สูงกว่า — เกี่ยวข้องกับ ACCESS-SW1–4 เพราะ Hardware ที่กำหนดไว้ (Catalyst 9200/2960) บางรุ่น
   (โดยเฉพาะ 2960) **ไม่รองรับ VXLAN/LISP ทางกายภาพ จึงเป็น Fabric Edge Node ตัวจริงไม่ได้**
   ต้องเข้า Fabric ผ่าน Extended Node ไปก่อนจนกว่าจะ Upgrade Hardware

---

## สรุป Part 38

Part นี้แนะนำ **Cisco SD-Access** ในฐานะสถาปัตยกรรม Campus แบบใหม่ที่ใช้ **Fabric** แทน 3-Tier
Model ดั้งเดิม โดยแยกการทำงานเป็น **3 Plane** ที่เป็นอิสระจากกัน — **Control Plane (LISP)** ที่
แยก EID/RLOC เพื่อรองรับ Host Mobility, **Data Plane (VXLAN)** ที่ Encapsulate Traffic ผ่าน
Underlay ธรรมดา และ **Policy Plane (SGT/TrustSec)** ที่แยก Macro-Segmentation (VN) กับ
Micro-Segmentation (SGT) ออกจาก IP Address โดยสิ้นเชิง เราทำความเข้าใจ **4 Fabric Role**
(Edge, Border, Control Plane Node, Fabric WLC), บทบาทของ **DNA Center** ในการ Automate
ทั้ง Underlay (LAN Automation) และ Overlay (Design→Policy→Provision→Assurance Workflow),
และพิสูจน์ให้เห็นว่า **Underlay จาก [Part 33](part-033-advanced-stp-campus-design.md) (OSPF
ECMP) นำมาใช้เป็น SD-Access Underlay ได้ทันที** โดยไม่ต้องแก้ไข Config เดิมเลยแม้แต่บรรทัดเดียว
(เพิ่มแค่ Jumbo MTU) ปิดท้ายด้วย **SD-Access Wireless** ที่เปลี่ยน CAPWAP Data Tunnel เป็น
VXLAN และ **Migration Path** ผ่าน Extended Node/Policy Extended Node สำหรับ Hardware รุ่นเก่า

**ย้ำอีกครั้งตามที่บอกไว้ต้น Part**: เนื้อหานี้คือ **สถาปัตยกรรมทางเลือกที่ CCNP ENCOR Exam
กำหนดให้ต้องรู้** ไม่ใช่การเปลี่ยนทิศทางของ Lab หลักของหลักสูตร — **Lab CLI-based เดิม (Part
1–35) จะยังถูกใช้และขยายต่อเนื่องตามปกติตั้งแต่ Part 39 เป็นต้นไป** โดย Part ถัดไปจะย้ายจาก
Campus (LAN) ไปสู่การออกแบบ **WAN สมัยใหม่**: **SD-WAN** — สถาปัตยกรรม Software-Defined
อีกแบบหนึ่งที่ใช้แนวคิด Control/Data/Policy Plane แยกกันคล้ายกับที่เราเพิ่งเรียนใน Part นี้ แต่
นำไปใช้แก้ปัญหาของ **WAN Edge Router** (WAN-EDGE-1/WAN-EDGE-2 จาก Part 19) แทนที่จะเป็น
Campus Access Layer

**ไปต่อ:** [Part 39 — SD-WAN Fundamentals →](part-039-sd-wan-fundamentals.md)
