# Part 77 — SD-Access Advanced Design & Deployment
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 761–770 จาก 1000**

> ต่อจาก [Part 76 — SD-WAN Advanced Design](part-076-sdwan-advanced-design.md) ที่เจาะลึกการออกแบบ
> WAN Edge ระดับ CCIE (Multi-Topology, Application-Aware Routing, Cloud onRamp) แล้ว Part นี้จะดึง
> เรื่อง **SD-Access** จาก [Part 38 — SD-Access Fundamentals](part-038-sd-access-fundamentals.md)
> กลับมาต่อยอดในเชิงลึกระดับ **CCIE Enterprise Infrastructure** — Part 38 สอนแนวคิด 1 Fabric Site
> เดียว (Border/Control Plane/Edge/WLC 4 Role) บน Lab หลักของหลักสูตร (CORE-SW/DIST-SW/ACCESS-SW)
> ซึ่งเพียงพอสำหรับ CCNP ENCOR แต่ **ไม่เพียงพอสำหรับองค์กรระดับ Enterprise จริงที่มีหลายอาคาร
> หลายภูมิภาค** — [Part 72 — Large Campus Fabric Design](part-072-large-campus-fabric-design.md)
> ได้ขยาย Lab หลักออกเป็น **Campus 10 อาคาร** (10-Building Enterprise Campus) ไว้แล้ว
> Part นี้จะนำ Campus นั้นมาออกแบบเป็น **Multi-Fabric SD-Access** เต็มรูปแบบ ครอบคลุมทุกมิติที่
> ข้อสอบ CCIE Enterprise Infrastructure Lab กำหนด: Multi-Site Transit, Border Node Type, Anycast
> Gateway ที่ Scale, Multicast บน Fabric, Wireless ข้าม Fabric Site, SGT ระดับ Multi-Site, Extended
> Node Migration Plan, และ Assurance/Troubleshooting เมื่อไม่มี GUI ให้ใช้
>
> **ข้อกำหนดสำคัญ**: ตัวเลข Loopback/Site ID/SGT Range ทั้งหมดใน Part นี้เป็น **ส่วนขยายเชิงแนวคิด
> ของ Campus 10 อาคารจาก Part 72** เขียนขึ้นให้สอดคล้องกับรูปแบบเดิมใน
> [`00-ip-address-plan.md`](00-ip-address-plan.md) โดย**ไม่แก้ไขค่าที่มีอยู่แล้วแม้แต่บรรทัดเดียว**
> (Loopback `1.1.1.x` ของ Lab หลักยังใช้ตามเดิมทุกประการ — ตัวเลขใหม่ในนี้อยู่ในช่วง `1.1.2xx.x`
> ที่ไม่ทับซ้อนกันเลย)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 761 | Multi-Fabric SD-Access ระดับ Enterprise — SD-Access Transit vs IP-based Transit |
| 762 | Fabric-in-a-Box vs Distributed Fabric Role — Sizing สำหรับ Site เล็ก/ใหญ่ |
| 763 | L2 Flooding & L3 Anycast Gateway ที่ Scale — แก้ปัญหา HSRP-per-VLAN จาก Part 35 |
| 764 | Border Node Types — Internal / External / Anywhere Border + Worked Design เชื่อม WAN-EDGE-1/2 |
| 765 | Multicast บน SD-Access — PIM-SM, Head-End Replication vs Native Multicast Underlay |
| 766 | Wireless ที่ Scale — Fabric WLC หลาย Site และ Inter-Site Roaming |
| 767 | Policy ที่ Scale — SGT Numbering Scheme หลาย Site + SGACL Matrix ผ่าน ISE/DNAC |
| 768 | Extended Node / Policy Extended Node ที่ Scale — แผน Phased-Replacement ของ Switch รุ่นเก่า |
| 769 | SD-Access Assurance & Troubleshooting ที่ Scale — DNAC Fabric Health + CLI LISP/VXLAN |
| 770 | Lab เชิงออกแบบเต็มรูปแบบ: Multi-Fabric SD-Access สำหรับ Campus 10 อาคาร |

---

## Step 761 — Multi-Fabric SD-Access ระดับ Enterprise: SD-Access Transit vs IP-based Transit

### ทำไม 1 Fabric Site ไม่เพียงพอสำหรับ Campus 10 อาคาร

[Part 38](part-038-sd-access-fundamentals.md) สอน Fabric Site เดียว (Single Site) ซึ่งมีขีดจำกัดตาม
Cisco Validated Design (CVD): **1 Fabric Site รองรับ Endpoint ได้จำกัด** (ขึ้นกับ Platform ของ
Control Plane Node — ประมาณหลักหมื่น EID ต่อ Site ในทางปฏิบัติ) และสำคัญกว่านั้นคือ **เหตุผลเชิง
Failure Domain**: ถ้า Fabric Site เดียวครอบทั้ง 10 อาคาร แล้ว Control Plane Node (LISP MS/MR)
ของ Site นั้นล่ม จะกระทบทั้งองค์กรพร้อมกัน — Cisco Validated Design จึงแนะนำให้แบ่งเป็น
**หลาย Fabric Site ที่เป็นอิสระจากกัน** แล้วเชื่อมด้วย **Transit**

### Campus 10 อาคารจาก Part 72: จัดกลุ่มเป็น 3 Fabric Site

| Fabric Site | อาคารที่ครอบ | ลักษณะงาน | ขนาด Endpoint (ประมาณ) |
|---|---|---|---|
| **FS-CAMPUS-CORE** | Building 1–4 (HQ Tower, Sales, Finance, HR) | Office/Knowledge Worker | ~4,500 |
| **FS-CAMPUS-RD** | Building 5–7 (R&D, Engineering, Training Center) | R&D/Lab-heavy, IoT ทดสอบสูง | ~3,200 |
| **FS-CAMPUS-OPS** | Building 8–10 (Manufacturing, DC Annex, Guest/Visitor Center) | Manufacturing IoT, Server, Guest | ~2,100 |

การแบ่งนี้ยึดหลัก **Failure Domain ตามภูมิศาสตร์ของกลุ่มอาคาร** (อาคารที่อยู่ใกล้กันมักใช้ Backbone
Fiber ร่วมชุดเดียวกันอยู่แล้ว) และ **แยกลักษณะ Traffic Profile** (Office vs R&D-IoT vs
Manufacturing-IoT) ออกจากกัน เพื่อให้ Policy/SGACL ของแต่ละ Site ปรับแต่งเฉพาะทางได้โดยไม่กระทบ
Site อื่น

### Transit: สิ่งที่เชื่อม Fabric Site เข้าด้วยกัน

**Transit** คือ Fabric Site พิเศษที่ **ไม่มี Edge Node ของตัวเอง** — มีหน้าที่เดียวคือ "ส่งต่อ
Traffic ระหว่าง Fabric Site" Cisco แบ่ง Transit ออกเป็น 2 แบบหลัก:

| หัวข้อ | SD-Access Transit (LISP Pub/Sub) | IP-based Transit (Traditional) |
|---|---|---|
| Control Plane ข้าม Site | **LISP Pub/Sub** ผ่าน **Transit Control Plane Node** โดยเฉพาะ — Site หนึ่ง "Subscribe" การเปลี่ยนแปลง EID ของ Site อื่นโดยตรง | ไม่มี LISP ข้าม Site เลย — แต่ละ Site เป็น "กล่องดำ" ที่มองไม่เห็นภายในกันและกัน |
| Data Plane ข้าม Site | **VXLAN ต่อเนื่อง** (Native VXLAN Encapsulation ทะลุจาก Edge Node ต้นทางถึง Edge Node ปลายทางข้าม Site โดยตรง ไม่ Decapsulate กลางทาง) | **Decapsulate ที่ Border Node แต่ละ Site ก่อน** แล้ว Route แบบ IP ธรรมดาผ่าน Transit — Encapsulate ใหม่อีกครั้งที่ Border Node ฝั่งปลายทาง |
| SGT เดินทางข้าม Site ได้ไหม | ✅ ได้โดยตรง (Inline ผ่าน VXLAN-GPO เหมือนภายใน Site เดียว) | ⚠️ ต้องพึ่ง SXP หรือ Re-classify ที่ Border Node ปลายทาง (อาจเสีย SGT ถ้าไม่ Config SXP ให้ครบ) |
| VN (Macro-Segmentation) ข้าม Site | ✅ คงอยู่ตลอดเส้นทาง | ⚠️ ต้อง Map VRF/VN ที่ทั้งสอง Border Node ให้ตรงกันเอง (Manual) |
| Underlay ที่ต้องการ | Backbone ระหว่าง Site ต้อง Route ถึงกัน (มักเป็น MPLS/Dark Fiber/SD-WAN Underlay) — ไม่ต้องรองรับ Multicast พิเศษ | Backbone ธรรมดา — ใช้ Network เดิมที่มีอยู่ได้ทันที (แม้เป็น Router รุ่นเก่าที่ไม่รองรับ Fabric เลย) |
| Hardware ที่ต้องมีเพิ่ม | ต้องมี **Transit Control Plane Node** โดยเฉพาะ (มักเป็น Router/Switch คู่ที่ Dedicate ให้ Role นี้) | ไม่ต้องมี Hardware ใหม่ — ใช้ Border Node ปกติของแต่ละ Site เชื่อมกันตรงๆ |
| เหมาะกับ | Enterprise ที่มี Backbone เป็นของตัวเอง (Dark Fiber/MPLS Private) ระหว่าง Site และต้องการ Policy/SGT ต่อเนื่องเต็มรูปแบบ | Site ที่เชื่อมกันผ่าน Network ที่ไม่ใช่ของ SD-Access เอง เช่น Internet/Generic WAN/Site คู่ค้า |

### Diagram: Campus 10 อาคาร เชื่อมด้วย SD-Access Transit

```
                         ┌───────────────────────────────┐
                         │   Transit Control Plane Node    │  <- 1.1.200.1 / 1.1.200.2
                         │   (LISP Pub/Sub ข้าม Fabric Site) │     (คู่ Redundant)
                         └────────────┬────────────────────┘
             ┌─────────────────────────┼─────────────────────────┐
             │ VXLAN ต่อเนื่องข้าม Site  │ VXLAN ต่อเนื่อง          │ VXLAN ต่อเนื่อง
  ┌──────────┴──────────┐   ┌──────────┴──────────┐   ┌──────────┴──────────┐
  │  FS-CAMPUS-CORE       │   │  FS-CAMPUS-RD         │   │  FS-CAMPUS-OPS        │
  │  Building 1-4         │   │  Building 5-7         │   │  Building 8-10        │
  │  Border+CP+Edge       │   │  Border+CP+Edge       │   │  Border+CP+Edge       │
  │  Loopback 1.1.201.x   │   │  Loopback 1.1.202.x   │   │  Loopback 1.1.203.x   │
  └───────────────────────┘   └───────────────────────┘   └───────────────────────┘
```

> **สำหรับข้อสอบ CCIE Enterprise Infrastructure**: คำถามแนว "Fabric Site 2 แห่งต้องการให้ SGT
> เดินทางถึงกันโดยตรงไม่มี Gap" ต้องตอบ **SD-Access Transit** เท่านั้น — **IP-based Transit ใช้ได้
> แต่ SGT/VN ไม่ต่อเนื่องอัตโนมัติ** ต้องพึ่งกลไกเสริม (SXP/Manual VRF Map) เสมอ

---

## Step 762 — Fabric-in-a-Box vs Distributed Fabric Role: Sizing สำหรับ Site เล็ก/ใหญ่

### Fabric-in-a-Box (FiaB): รวม Role ทั้งหมดในอุปกรณ์คู่เดียว

**Fabric-in-a-Box** คือการให้ **StackWise Switch คู่เดียว** (หรือ Switch Stack) รับทำ **ทั้ง 3 Role
พร้อมกัน**: Border Node + Control Plane Node + Edge Node ในตัวเดียว — เหมาะกับ Site ขนาดเล็กที่
ไม่คุ้มจะแยก Hardware หลายชุด

```
Fabric-in-a-Box (Small Site):
┌─────────────────────────────────────┐
│   Catalyst 9300 Stack (คู่เดียว)       │
│   Border + Control Plane + Edge      │  <- ทุก Role อยู่ในอุปกรณ์เดียว
│   รองรับ Endpoint ได้จำกัดตาม Platform │
└─────────────────────────────────────┘
        │              │
   Endpoint         Uplink ไปยัง Transit/Site อื่น
```

### Distributed Fabric Role: แยก Role ออกเป็นอุปกรณ์เฉพาะทาง

Site ขนาดใหญ่ (เช่น FS-CAMPUS-CORE ที่ครอบ Building 1-4 ~4,500 Endpoint) ต้อง **แยก Role ออก
เป็นอุปกรณ์เฉพาะทาง** เพื่อ Scale และ Redundancy ที่ดีกว่า:

| Role | Platform ที่แนะนำ | จำนวนแนะนำต่อ Site | เหตุผล |
|---|---|---|---|
| Border Node | Catalyst 9500 (High-end, รองรับ Route Table ใหญ่) | 2 ตัว (Redundant Pair) | ต้องรองรับ Full Route จาก External Network + Handle Traffic ทุก VN ที่ออกนอก Fabric |
| Control Plane Node | Catalyst 9500 หรือแยกเป็น Router เฉพาะทาง | 2 ตัว (Redundant Pair) | ยิ่ง Endpoint มาก ยิ่งต้องมี CPU/Memory สำหรับ LISP Map-Cache มากขึ้น — แยกออกจาก Border ช่วยไม่ให้ Border ที่ Handle Traffic หนักไปแย่ง Resource กับ Control Plane |
| Edge Node | Catalyst 9300 (ต่อ Endpoint โดยตรง) | ตามจำนวนตู้ Wiring Closet ต่ออาคาร | เป็น Role ที่ Scale ตามแนวนอน (Scale-out) ได้ง่ายที่สุด — เพิ่มตู้ใหม่ = เพิ่ม Edge Node ใหม่ |

### เกณฑ์ตัดสินใจ (Sizing Guidance)

| เกณฑ์ | Fabric-in-a-Box | Distributed Role |
|---|---|---|
| จำนวน Endpoint ต่อ Site | < 1,000 | > 1,000 (โดยเฉพาะ > 2,000 ควร Distributed เสมอ) |
| จำนวนอาคารที่ Site ครอบ | 1 อาคารเล็ก/สาขา | หลายอาคาร (เช่น FS-CAMPUS-CORE ครอบ 4 อาคาร) |
| ความต้องการ Redundancy | พอใช้คู่ Stack เดิม | ต้องแยก Failure Domain ของแต่ละ Role ออกจากกันจริง |
| งบประมาณ Hardware | จำกัด | รองรับ Hardware เฉพาะทางหลายชุด |
| ตัวอย่างใน Campus 10 อาคาร | Building 7 (Training Center, ~300 Endpoint) และ Building 10 (Guest Center, ~250 Endpoint) — ใช้ Fabric-in-a-Box ผูกกับ Fabric Site ที่ใกล้ที่สุด | FS-CAMPUS-CORE (Building 1-4), FS-CAMPUS-RD (Building 5-6) — Distributed เต็มรูปแบบ |

### ตัวอย่าง Sizing จริงของ Campus 10 อาคาร

| Fabric Site | รูปแบบ Role | รายละเอียด |
|---|---|---|
| FS-CAMPUS-CORE | Distributed | Border 2 ตัว (9500) + Control Plane 2 ตัว (9500) + Edge ~40 ตัว (9300) กระจาย 4 อาคาร |
| FS-CAMPUS-RD | Distributed (Building 5-6) + Fabric-in-a-Box (Building 7 Training Center ผูกเข้า Site เดียวกัน) | Border+CP ใช้ร่วมกันทั้ง Site แต่ Building 7 มี Edge Node จำนวนน้อยและเป็น Stack เดียว |
| FS-CAMPUS-OPS | Distributed (Building 8-9) + Fabric-in-a-Box (Building 10 Guest Center) | Manufacturing/DC ต้องการ Border แยกเฉพาะเพื่อ Enforce Policy IoT เข้มงวด ส่วน Guest Center เล็กพอจะรวม Role ได้ |

> **สำหรับข้อสอบ CCIE**: Fabric-in-a-Box **ไม่ใช่ Role ใหม่** — เป็นแค่การ **Co-locate 3 Role เดิม**
> (Border+CP+Edge) ไว้ในอุปกรณ์ทางกายภาพเดียว ข้อจำกัดสำคัญคือ **ทำไม่ได้กับ Anywhere Border**
> (Step 764) เพราะต้องมีอุปกรณ์แยกเฉพาะสำหรับ Role นั้น

---

## Step 763 — L2 Flooding และ L3 Anycast Gateway ที่ Scale

### ทวนปัญหาจาก Part 35: HSRP-per-VLAN Scaling Problem

[Part 35](part-035-advanced-fhrp-campus-ha.md) สอน HSRP ที่ต้องมี **1 VIP ต่อ 1 VLAN ต่อคู่ DIST-SW**
— เมื่อ Enterprise ขยายเป็น 10 อาคาร ปัญหานี้ทวีความรุนแรงขึ้นแบบ Non-linear:

```
Traditional HSRP-per-VLAN (Part 18, 35) ที่ Scale 10 อาคาร:
  - แต่ละอาคารมี VLAN ของตัวเอง (แม้เป็น "กลุ่มงานเดียวกัน" ก็ต้องคนละ VLAN/Subnet ต่ออาคาร)
  - ทุก VLAN ต้องมี HSRP Group แยก = ทุกคู่ DIST-SW ต้อง Track/Preempt/Election แยกทุก VLAN
  - Endpoint ย้ายข้ามอาคาร (เช่น Laptop พนักงานเดินไปประชุมอีกตึก) = ต้องเปลี่ยน Subnet เสมอ
  - First-Hop Router Redundancy ต้อง Config ซ้ำ 10 รอบ (1 ต่ออาคาร) และ Trunk ต้องพา VLAN
    ข้ามอาคารถ้าต้องการ Subnet เดียวกันทั้ง Campus (ขัดกับหลัก STP Best Practice จาก Part 33)
```

### SD-Access แก้ด้วย Anycast Gateway: "1 Gateway IP เดียว ซ้ำกันทุก Edge Node"

**Anycast Gateway** คือการกำหนด **IP Address และ MAC Address ของ Default Gateway ค่าเดียวกัน
เป๊ะๆ ให้ Edge Node ทุกตัวใน Fabric Site** (แม้อยู่คนละอาคาร คนละตู้ Wiring Closet) — ไม่มี HSRP/
VRRP/GLBP Election ใดๆ เกิดขึ้นเลย เพราะไม่มี "Active/Standby" ให้เลือก ทุก Edge Node **Active
เท่ากันหมดพร้อมกัน**

```
Anycast Gateway ใน SD-Access:
  VLAN "EMPLOYEE_10_10_10_0/24" ผูกกับ Gateway IP 10.10.10.1, MAC เดียวกัน (Anycast MAC)
  ├── Edge Node (Building 1) มี SVI 10.10.10.1/MAC เดียวกันนี้
  ├── Edge Node (Building 2) มี SVI 10.10.10.1/MAC เดียวกันนี้
  ├── Edge Node (Building 3) มี SVI 10.10.10.1/MAC เดียวกันนี้
  └── ... ทุก Edge Node ใน Fabric Site เดียวกัน มี SVI/MAC นี้ซ้ำกันหมด

  Endpoint ที่เสียบที่ Edge Node ไหนก็ตาม ARP หา 10.10.10.1 แล้วได้ MAC เดียวกันเสมอ
  ไม่ต้องรู้ว่ากำลังเสียบอยู่ Edge Node ตัวไหน/อาคารไหน
```

### กลไกเบื้องหลัง: ทำไม Anycast Gateway ไม่ชนกันเอง (ต่างจาก IP Address ปกติที่ Anycast กันไม่ได้)

หัวใจคือ **LISP Host-Route (/32 หรือ /128)** จาก [Part 38 Step 373](part-038-sd-access-fundamentals.md)
— Edge Node ไม่ได้ Route ตาม Subnet Boundary แบบเดิม แต่ Register **EID ของ Endpoint แต่ละตัวเป็น
Host-Route เข้า Control Plane Node โดยตรง** ดังนั้น "Gateway ซ้ำกันทุกที่" จึงไม่มีปัญหา เพราะ
Traffic ระหว่าง Endpoint กับ Endpoint **ไม่ได้พึ่ง Gateway Election แบบ FHRP เลย** — Edge Node
แต่ละตัวรู้เองว่า Endpoint ที่ต่ออยู่ใต้ตัวเองมี Host-Route ไปทางไหน (Local Switching ทันทีถ้าปลายทาง
อยู่ Edge Node เดียวกัน, หรือ Encapsulate VXLAN ไปหา RLOC ที่ถูก Query จาก Control Plane ถ้าอยู่
ต่าง Edge Node)

### ตารางเปรียบเทียบเต็มรูปแบบ

| หัวข้อ | HSRP-per-VLAN (Part 18/35) | L3 Anycast Gateway (SD-Access) |
|---|---|---|
| จำนวน Gateway IP ต่อ Subnet | 1 (Virtual IP, Active/Standby 2 ตัวจริง) | 1 (แต่ Active พร้อมกันทุก Edge Node — ไม่มี Standby) |
| Election Protocol | มี (Hello/Hold Timer, Priority, Preempt) | **ไม่มี** — ไม่ต้อง Election เลย |
| Failover Time เมื่อ Gateway ตัวหนึ่งล่ม | ขึ้นกับ Hold Timer (มักหลักวินาที แม้ Sub-second Tuning ได้) | **ทันที** — Edge Node ตัวอื่นก็ Active อยู่แล้ว ไม่ต้องรอ Failover เลย |
| VLAN ต้อง Trunk ข้ามอาคารไหม (ถ้าต้องการ Subnet เดียวกันหลายอาคาร) | ต้อง Trunk ทะลุทุก Switch ที่เกี่ยวข้อง (STP Complexity เพิ่มตามจำนวนอาคาร) | **ไม่ต้อง** — VLAN/SVI เดียวกันถูกสร้างซ้ำเป็น Local อิสระที่ Edge Node ทุกตัว ไม่มี L2 Trunk ข้ามอาคารเลย |
| L2 Flooding (Broadcast/Unknown Unicast/Multicast) ข้าม Edge Node | มี STP บังคับ Loop-Free แต่ Flood เต็ม VLAN ทุกครั้ง | **ควบคุมด้วย LISP** — Broadcast ถูกจำกัดเฉพาะ "หลัง Edge Node เดียวกัน" เท่านั้น (Local Flooding) ไม่ Flood ข้าม Fabric โดยอัตโนมัติ ยกเว้นเปิด Layer 2 Overlay/Flooding เฉพาะ VN ที่จำเป็น (เช่น รองรับ Legacy Application ที่ต้องพึ่ง Broadcast ข้าม Site) |
| Endpoint ย้ายอาคาร (เช่น Laptop, ไม่ใช่ Wireless Roaming) | ต้องเปลี่ยน Subnet ถ้าอาคารปลายทางไม่มี VLAN เดิม | **ไม่ต้องเปลี่ยน IP** — เสียบที่ Edge Node ไหนก็ได้ Anycast Gateway เดียวกันตอบทันที (จำกัดตาม IP Pool ที่ Provision กับ Fabric Site นั้น) |
| จำนวน HSRP Group ที่ต้อง Config เมื่อ Scale เป็น 10 อาคาร | ทวีตามจำนวน VLAN × จำนวนคู่ DIST-SW (สูงมาก) | **คงที่** — DNAC Provision Anycast Gateway ให้ Edge Node ทุกตัวจาก Template เดียว ไม่ทวีตามจำนวนอาคาร |

### ข้อจำกัดที่ต้องรู้: Anycast Gateway ผูกกับขอบเขต Fabric Site เดียวเท่านั้น

Anycast Gateway ของ Subnet หนึ่ง **ใช้ได้เฉพาะภายใน 1 Fabric Site** — ถ้า Endpoint ย้ายข้าม
**Fabric Site** (เช่น จาก FS-CAMPUS-CORE ไป FS-CAMPUS-RD ตาม Step 761) จะ**ไม่ใช่ Anycast
Gateway เดิมอีกต่อไป** ต้องพึ่งกลไก Inter-Site Roaming (Step 766) หรือ Renumber ตามปกติ — นี่คือ
เหตุผลสำคัญที่การแบ่ง Fabric Site (Step 761) ต้องคำนึงถึง **Mobility Pattern จริงของผู้ใช้** ด้วย
ไม่ใช่แค่ภูมิศาสตร์อาคาร

> **สำหรับข้อสอบ CCIE**: Anycast Gateway **ไม่ได้ยกเลิก Default Gateway concept** — ยังมี Gateway
> IP/MAC เดิมทุกประการจากมุมของ Endpoint แต่สิ่งที่หายไปคือ **FHRP Protocol (HSRP/VRRP/GLBP)**
> เพราะ LISP Host-Route ทำหน้าที่แทนกลไก Redundancy ทั้งหมดแล้ว

---

## Step 764 — Border Node Types และ External Connectivity Design

### 3 ประเภทของ Fabric Border Node

| ประเภท | หน้าที่ | Route ที่ Advertise เข้า Fabric | ตัวอย่างการใช้งาน |
|---|---|---|---|
| **Internal Border** | เชื่อม Fabric เข้ากับ Network ภายในองค์กรเดียวกันที่ **รู้จักหมดทุก Prefix** (เช่น Data Center ภายใน, Site SD-Access อื่นที่ยัง Migrate ไม่ครบ) | Advertise **Prefix เฉพาะที่รู้จักจริง** เข้า Fabric (Specific Route) | เชื่อม FS-CAMPUS-CORE ไปยัง Data Center Annex (Building 9) ที่ยังเป็น Traditional Network บางส่วน |
| **External Border** | เชื่อม Fabric เข้ากับ Network ที่ **"ไม่รู้จักทั้งหมด"** (Internet, Shared Services, WAN ที่มี Prefix เปลี่ยนแปลงตลอด) | Advertise **Default Route (0.0.0.0/0)** เข้า Fabric แทนการรู้จัก Prefix ทีละเส้น | เชื่อม FS-CAMPUS-CORE ไปยัง **WAN-EDGE-1/WAN-EDGE-2** (Internet/MPLS ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md)) |
| **Anywhere Border** | ทำหน้าที่ **ทั้ง Internal และ External พร้อมกัน** ในอุปกรณ์เดียว — ใช้เมื่อไม่ต้องการแยก Border Node 2 ชุด | Advertise ทั้ง Default Route และ Specific Prefix พร้อมกัน | Site ขนาดเล็ก/กลางที่ไม่คุ้มจะแยก Border 2 คู่ (ไม่ใช้ร่วมกับ Fabric-in-a-Box ได้ในบางกรณีเนื่องจากข้อจำกัด Platform — ต้องตรวจสอบ CVD ของรุ่นที่ใช้จริงเสมอ) |

### ทำไมต้องแยก Internal/External: ป้องกัน Default Route "ชนกัน"

ถ้า Border Node ตัวเดียวรับทั้ง Default Route จาก Internet และ Specific Route จาก Data Center
ภายใน โดยไม่ระบุ Role ให้ชัดเจน DNAC/LISP Control Plane อาจสับสนว่า Traffic ปลายทางที่ไม่รู้จัก
ควรออกทางไหน — การระบุ Role Internal/External อย่างชัดเจนทำให้ **Control Plane Node รู้ว่า
"EID ที่ไม่พบใน Map-Cache ให้ส่งไปทาง External Border เป็น Default เสมอ"** ในขณะที่ Prefix ของ
Data Center ภายในยังคง Resolve ผ่าน Internal Border ตามที่ Register ไว้จริง

### Worked Design: เชื่อม Campus 10 อาคารเข้ากับ WAN-EDGE-1/WAN-EDGE-2

FS-CAMPUS-CORE (Building 1-4) เป็น Fabric Site เดียวที่ทำหน้าที่ **External Border** ให้กับทั้ง
Campus (Fabric Site อื่นออก Internet ผ่าน SD-Access Transit มาที่ Site นี้ก่อน — ดู Step 761)

```
                         ┌───────────────┐
                         │   INTERNET     │
                         └───────┬───────┘
                         ┌───────┴───────┐
                         │   ISP-RTR       │  <- 1.1.1.254 (00-ip-address-plan.md)
                         └───────┬───────┘
              ┌──────────────────┴──────────────────┐
              │      WAN-EDGE-1 / WAN-EDGE-2           │  <- 1.1.1.21 / 1.1.1.22
              └──────────────────┬──────────────────┘
                    Native IP (ไม่ใช่ VXLAN — นอก Fabric)
        ┌──────────────────────────┴──────────────────────────┐
        │         FS-CAMPUS-CORE External Border Node ×2        │  <- Border+CP (Step 762)
        │         (ทำหน้าที่ตัด VXLAN ↔ Native IP ที่จุดนี้)        │     1.1.201.1 / 1.1.201.2
        │         Advertise Default Route (0.0.0.0/0)            │
        │         เข้า Fabric ทั้ง FS-CAMPUS-CORE                 │
        └──────────────────────────┬──────────────────────────┘
                     VXLAN (RLOC↔RLOC ภายใน Fabric Site)
        ┌──────────────────────────┴──────────────────────────┐
        │              Edge Node ทั้งหมดใน FS-CAMPUS-CORE          │
        └───────────────────────────────────────────────────────┘
                     SD-Access Transit (Step 761) — ส่งต่อ Default
                     Route ไปให้ FS-CAMPUS-RD และ FS-CAMPUS-OPS ด้วย
```

### สิ่งที่เกิดขึ้นที่ External Border Node จริง (Data Plane)

1. Traffic จาก Endpoint (EID, เช่น `10.10.10.5` ที่ Building 2) ปลายทางเป็น Internet →
   Edge Node (ITR) Encapsulate VXLAN ส่งไปยัง RLOC ของ External Border Node (เพราะ Control
   Plane Node ตอบว่า "ไม่พบ EID ปลายทางใน Fabric ใดเลย ให้ส่งไปทาง Default")
2. External Border Node **Decapsulate VXLAN** ออก เหลือ Packet ปกติที่มี Source IP = EID เดิม
3. Border Node NAT/Route Packet นี้ (ตาม Policy ของ VN ต้นทาง) ออกไปยัง WAN-EDGE-1/2 เป็น
   **Native IP ปกติ** — จากจุดนี้ไปไม่มี VXLAN/LISP อีกเลย เหมือน Traffic จาก Traditional Network
4. ขากลับ (Return Traffic) วิ่งกลับมาที่ Border Node ตัวเดิม แล้ว Border Node ทำหน้าที่ **ETR**
   Encapsulate VXLAN กลับเข้า Fabric หา Edge Node ต้นทางตามปกติ

### ตาราง Redundancy: External Border 2 ตัวทำงานร่วมกันอย่างไร

| สถานการณ์ | พฤติกรรม |
|---|---|
| ทั้ง 2 External Border Node Up ปกติ | Load-Balance ผ่าน ECMP (LISP รองรับหลาย RLOC ต่อ Default Route เดียว) เหมือนหลักการ ECMP จาก [Part 33](part-033-advanced-stp-campus-design.md) |
| External Border Node ตัวหนึ่งล่ม | Control Plane Node หยุด Advertise RLOC ของตัวที่ล่ม — Traffic ทั้งหมดไปทาง Border Node ที่เหลือทันที ไม่ต้องรอ Convergence แบบ Routing Protocol ปกติ (LISP Map-Cache Update เร็วกว่า) |
| WAN-EDGE-1 ล่มฝั่ง Native IP (ไม่ใช่ Border Node) | Border Node ที่ผูกกับ WAN-EDGE-1 Route ผ่าน WAN-EDGE-2 ตาม Routing ปกติของ Underlay (ไม่เกี่ยวกับ Fabric แล้ว ณ จุดนี้) |

> **สำหรับข้อสอบ CCIE**: จำหลักนี้ให้แม่น — **External Border = Default Route, Internal Border =
> Specific Route** — Anywhere Border ทำทั้งสองอย่างได้ในตัวเดียว แต่ **ไม่ใช่ตัวเลือกแรกสำหรับ
> Design ขนาดใหญ่** เพราะรวม Failure Domain ของทั้ง Internet Edge และ DC Edge ไว้จุดเดียวกัน

---

## Step 765 — Multicast บน SD-Access: PIM-SM, Head-End Replication, Native Multicast Underlay

### ทบทวนสั้นๆ: ทำไม Multicast ซับซ้อนกว่า Unicast บน Fabric

Unicast บน SD-Access ง่าย เพราะ LISP รู้ EID↔RLOC ชัดเจนแบบ 1-ต่อ-1 (Step 373 ของ Part 38) แต่
**Multicast คือ 1-ต่อ-หลาย** — Fabric ต้องหาวิธี "กระจาย" Traffic จาก Source ไปยัง Receiver
หลายตัวที่อาจอยู่หลัง Edge Node คนละตัวกัน โดยยังต้อง**คง VNI/VN เดิมของ Overlay** ไว้เหมือน
Unicast Traffic ทุกประการ (Multicast ต้องอยู่ VN เดียวกันกับ Source เสมอ — ห้ามข้าม VN โดยไม่ผ่าน
Border/Fusion Router)

### 2 วิธีที่ SD-Access ใช้กระจาย Multicast Traffic ผ่าน Fabric

| วิธี | หลักการ | Underlay ต้องรองรับ PIM ไหม | Efficiency | เหมาะกับ |
|---|---|---|---|---|
| **Head-End Replication (HER)** | Edge Node ต้นทาง (ที่ Source เสียบอยู่) **Copy Packet Multicast เป็น Unicast VXLAN แยกไปทีละ RLOC ปลายทาง** ที่มี Receiver Register ไว้ — เหมือน "ส่งจดหมายฉบับเดียวกันซ้ำหลายฉบับ" | **ไม่ต้อง** — Underlay มองเห็นแค่ Unicast VXLAN ธรรมดาหลายชุด ไม่ต้อง Config PIM ที่ Intermediate Node เลย | ต่ำกว่า (Bandwidth โต 1:1 กับจำนวน RLOC ปลายทาง — ถ้ามี Receiver อยู่หลัง Edge Node 20 ตัว ต้องส่ง 20 Copy จาก Edge Node ต้นทาง) | Fabric Site ขนาดเล็ก-กลาง หรือ Multicast Group ที่มี Receiver ไม่มาก — เป็น **ค่า Default ของ SD-Access** เพราะ Config ง่ายที่สุด (ไม่ต้องแตะ Underlay เลย) |
| **Native Multicast (PIM ในการ Underlay)** | เปิด **PIM-SM ที่ Underlay จริง** (Intermediate Node ต้องรัน PIM ด้วย ไม่ใช่แค่ Routing เฉยๆ แบบ Step 372 ของ Part 38 อีกต่อไป) — Underlay เองทำหน้าที่ Replicate Packet ตาม Multicast Tree ปกติ (เหมือน [Part 41-42](part-041-multicast-fundamentals.md)) แล้ว Fabric แค่ VXLAN-Encapsulate ครั้งเดียวเป็น Multicast Group Address ของ Underlay | **ต้อง** — Intermediate Node ทุกตัวต้องรองรับ PIM-SM (เสียหลักการ "Intermediate Node ไม่รู้จัก Fabric" ไปบางส่วน แต่ยังไม่ต้องรู้จัก VXLAN/LISP อยู่ดี — รู้แค่ PIM) | สูงกว่ามาก (Replicate ครั้งเดียวที่ Fanout Point ของ Underlay ตาม Multicast Tree ปกติ ไม่โตตามจำนวน Receiver ที่ Edge Node) | Fabric Site ขนาดใหญ่ หรือ Multicast Group ที่มี Receiver จำนวนมาก กระจายหลาย Edge Node (เช่น Video/Trading Feed ที่กระจายทั้ง Campus) |

### Diagram เปรียบเทียบ

```
Head-End Replication (HER):
  Source ──► Edge Node A ──┬─VXLAN Unicast─► Edge Node B (Receiver 1)
                            ├─VXLAN Unicast─► Edge Node C (Receiver 2)
                            └─VXLAN Unicast─► Edge Node D (Receiver 3)
  (Edge Node A ส่ง 3 Copy แยกกัน — Bandwidth ที่ Edge Node A โตตามจำนวน Receiver)

Native Multicast (PIM ใน Underlay):
  Source ──► Edge Node A ──VXLAN(ครั้งเดียว, ปลายทาง=Underlay Multicast Group)──►
              Intermediate Node (ทำ PIM Replicate ตาม Tree จริง)
                    ├──► Edge Node B (Receiver 1)
                    ├──► Edge Node C (Receiver 2)
                    └──► Edge Node D (Receiver 3)
  (Edge Node A ส่งครั้งเดียว — Replicate เกิดที่ Underlay ตาม Fanout Point จริง ประหยัด Bandwidth
   ต้นทางมากกว่า)
```

### RP (Rendezvous Point) Placement เมื่อใช้ Native Multicast

เมื่อเลือก Native Multicast ต้องวาง **RP** ตามหลักการเดียวกับ [Part 41-42](part-041-multicast-fundamentals.md)
โดยมีจุดที่ต้องระวังเพิ่มสำหรับ SD-Access:

| ตำแหน่ง RP | ข้อดี | ข้อเสีย |
|---|---|---|
| ที่ Border Node ของแต่ละ Fabric Site | ควบคุม Policy Multicast ร่วมกับ Unicast Border Policy ได้ในจุดเดียว | ถ้า Source/Receiver ข้าม Fabric Site (ผ่าน SD-Access Transit, Step 761) ต้องมี **Anycast RP** ระหว่าง Border Node ของแต่ละ Site เพื่อให้ PIM Join ข้าม Site ทำงานถูกต้อง |
| ที่ Intermediate Node กลาง Campus (นอก Fabric Role ทั้งหมด) | แยก Failure Domain ของ RP ออกจาก Fabric Overlay โดยสิ้นเชิง | ต้องดูแล Config PIM แยกจาก DNAC Workflow (DNAC ไม่ Provision RP บน Intermediate Node ให้อัตโนมัติ) |

> **สำหรับข้อสอบ CCIE**: **Head-End Replication คือค่า Default ที่ DNAC ตั้งให้เมื่อเปิด Multicast
> บน Fabric ครั้งแรก** — ถ้าโจทย์ระบุว่า "ต้องการ Efficiency สูงสุดสำหรับ Multicast Group ที่มี
> Receiver จำนวนมากกระจายทั่ว Fabric" คำตอบคือเปลี่ยนไปใช้ **Native Multicast (เปิด PIM-SM ที่
> Underlay)** ซึ่งต้อง Config เพิ่มที่ Intermediate Node เสมอ ไม่ใช่ค่าที่มาให้ตั้งแต่ต้น

---

## Step 766 — Wireless ที่ Scale: Fabric WLC หลาย Site และ Inter-Site Roaming

### ทวนจาก Part 36: ปัญหา Roaming ข้าม Fabric Site ยากกว่า Roaming ภายใน Site เดียว

[Part 36](part-036-wireless-architecture-deep-dive.md) และ [Part 38 Step 378](part-038-sd-access-fundamentals.md)
สอน Roaming **ภายใน Fabric Site เดียว** ซึ่งใช้กลไก LISP EID/RLOC ล้วนๆ (Edge Node ใหม่ Register
EID ของ Client ทันทีที่ Roam มา) — แต่เมื่อ Client เดินทางข้าม **Fabric Site** (เช่น พนักงานเดินจาก
Building 3 ใน FS-CAMPUS-CORE ไปประชุมที่ Building 6 ใน FS-CAMPUS-RD) ปัญหาซับซ้อนกว่ามาก
เพราะ **แต่ละ Fabric Site มี Control Plane Node และ WLC ของตัวเองที่เป็นอิสระจากกัน**

### รูปแบบ Fabric WLC สำหรับ Multi-Site: Centralized WLC vs Distributed WLC ต่อ Site

| รูปแบบ | คำอธิบาย | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **1 WLC (Cluster) ควบคุมทุก Fabric Site** | WLC เดียว (หรือ HA Pair) วางที่ Data Center กลาง ผูก AP จากทุก Fabric Site เข้ามาที่ตัวเดียว | จัดการ RRM/Config จุดเดียว, Roaming ระหว่าง AP คนละ Fabric Site จัดการง่ายกว่า (WLC เห็น Client ทั้งหมดในระบบเดียว) | CAPWAP Control Tunnel ต้องวิ่งไกลข้าม Site ทั้งหมด (Latency), Single Point of Failure ถ้า WLC Cluster ล่มกระทบทุก Site พร้อมกัน |
| **1 WLC ต่อ Fabric Site (Distributed)** | แต่ละ Fabric Site (FS-CAMPUS-CORE, FS-CAMPUS-RD, FS-CAMPUS-OPS) มี Fabric WLC ของตัวเอง | Failure Domain แยกตาม Site (WLC ล่ม กระทบแค่ Site เดียว), CAPWAP Control Tunnel สั้นกว่า (Latency ต่ำ) | Roaming ข้าม Site ต้องพึ่งกลไกเสริม (ดูหัวข้อถัดไป) เพราะ WLC คนละตัวไม่รู้จัก Client ของกันและกันโดยตรง |

Campus 10 อาคารจาก Part 72 ใช้รูปแบบ **Distributed (1 WLC ต่อ Fabric Site)** เพื่อจำกัด Failure
Domain ตามหลักการเดียวกับการแบ่ง Fabric Site ใน Step 761

### กลไก Inter-Site Roaming: ทำไมยากกว่า และ SD-Access แก้อย่างไร

```
Roaming ภายใน Fabric Site เดียว (Part 38 Step 378):
  Client Roam จาก AP1 (Edge Node A) → AP2 (Edge Node B) — ยัง Fabric Site เดิม
  → Control Plane Node ตัวเดิม Update RLOC ของ EID (Client) จาก A เป็น B ทันที
  → WLC ตัวเดิม (Fabric WLC ของ Site นี้) ยังบริหาร AP ทั้งสองตัวเหมือนเดิม
  → ไร้รอยต่อสมบูรณ์ ไม่ต้องเปลี่ยน IP เลย

Roaming ข้าม Fabric Site (Inter-Site — Step นี้):
  Client Roam จาก AP ใน FS-CAMPUS-CORE → AP ใน FS-CAMPUS-RD
  → Control Plane Node ของทั้ง 2 Site เป็นอิสระกัน — Site ปลายทางไม่รู้จัก EID นี้มาก่อน
  → ต้องพึ่ง SD-Access Transit (Step 761) ส่งต่อ EID Registration ข้าม Site ผ่าน
    Transit Control Plane Node (LISP Pub/Sub ทำหน้าที่นี้โดยตรง)
  → WLC คนละตัว: AP ปลายทางผูกกับ Fabric WLC ของ FS-CAMPUS-RD ซึ่งไม่มี Session Context
    เดิมของ Client จาก WLC ของ FS-CAMPUS-CORE — Client ต้อง Re-Authenticate ผ่าน 802.1X/ISE ใหม่
    (แม้ IP ยังอาจคงเดิมได้ถ้า IP Pool เดียวกันถูก Provision ข้าม Site — ขึ้นกับ Design)
```

**สรุปกลไก**: SD-Access Transit ทำให้ **EID/RLOC** เดินทางข้าม Site ได้ (แก้ปัญหาระดับ Network
Layer) แต่ **Wireless Session/Authentication State** ที่ผูกกับ WLC ยังเป็นคนละชุดกันเสมอเมื่อ WLC
Distributed ต่อ Site — Client จะได้ **Roaming ที่ "IP อาจคงเดิม" แต่ "ไม่ไร้รอยต่อระดับ Session
100%"** (มี Re-Authentication Delay สั้นๆ) ต่างจาก Roaming ภายใน Site เดียวที่ไร้รอยต่อสมบูรณ์

### SD-Access Wireless vs Traditional CAPWAP: ทวนที่ Scale หลาย Site

| หัวข้อ | SD-Access Wireless (Fabric-enabled, Multi-Site) | Traditional CAPWAP (Centralized WLC เดิมจาก Part 20) |
|---|---|---|
| Traffic Tromboning | ไม่มีภายใน Site เดียว (Local ที่ Edge Node) — แต่ข้าม Site ยังต้องผ่าน SD-Access Transit | มีเสมอ — Client Traffic วิ่งไป WLC ก่อนทุกครั้งไม่ว่า Client อยู่ใกล้ปลายทางแค่ไหน |
| Roaming ภายใน Site | ไร้รอยต่อสมบูรณ์ผ่าน LISP EID/RLOC | ต้องพึ่ง Mobility Group/Tunnel ระหว่าง WLC (ถ้าหลาย WLC) |
| Roaming ข้าม Site | มี Re-Authentication Delay สั้นๆ (ตามที่อธิบายข้างบน) เว้นแต่ Design ให้ WLC เดียวคุมทุก Site | ต้องพึ่ง Inter-Controller Mobility Tunnel เต็มรูปแบบ ซึ่งมี Latency ใกล้เคียงกันหรือมากกว่า |
| ความซับซ้อนการจัดการที่ Scale 10 อาคาร | ต้องออกแบบ Site Boundary ให้ตรงกับ Mobility Pattern จริง (Step 763 เดียวกัน) | Scale ง่ายกว่าในมุม Config (WLC เดียวคุมทุก AP) แต่ Traffic Tromboning หนักขึ้นเรื่อยๆ ตามจำนวนอาคาร |
| Policy (SGT) ของ Wireless Client | Consistent กับ Wired เต็มรูปแบบทุก Site (ถ้าใช้ SD-Access Transit) | ต้องพึ่ง AAA Override ต่อ WLC แยกกันถ้ามีหลาย WLC |

> **สำหรับข้อสอบ CCIE**: คำถามแนว "องค์กรมีหลาย Fabric Site และต้องการ Roaming ไร้รอยต่อ 100%
> ข้าม Site" คำตอบตรงคือ **ไม่มีทางไร้รอยต่อ 100% แบบภายใน Site เดียว** ต้องยอมรับ Trade-off
> (Re-Authentication) เว้นแต่จะออกแบบใหม่ให้ Site นั้นรวมเป็น Site เดียว (ซึ่งขัดกับหลัก Failure
> Domain จาก Step 761) — นี่คือ Design Trade-off ที่ต้องชี้แจงในข้อสอบ Design/Lab ระดับ CCIE

---

## Step 767 — Policy ที่ Scale: SGT Numbering Scheme หลาย Site + SGACL Matrix ผ่าน ISE/DNAC

### ปัญหา: SGT ชนกันข้าม Site ถ้าไม่วางแผน Numbering ล่วงหน้า

[Part 44](part-044-dot1x-trustsec.md) กำหนด SGT ของ Lab หลักไว้ 4 ค่า (`SALES_SGT=100`,
`VOICE_SGT=110`, `SERVERS_SGT=120`, `PRINTER_SGT=130`) ซึ่งเพียงพอสำหรับ 1 Site — แต่เมื่อ
Campus ขยายเป็น 3 Fabric Site (Step 761) แต่ละ Site มักมีทีมงานเฉพาะทางกำหนด SGT ของตัวเองแยก
กัน (เช่นทีม R&D อาจตั้ง SGT 100 สำหรับ "LAB_EQUIPMENT" โดยไม่รู้ว่า HQ ใช้ 100 สำหรับ "SALES"
ไปแล้ว) — เมื่อ SD-Access Transit เชื่อม 2 Site เข้าด้วยกัน (Step 761) **SGT ที่ชนกันจะทำให้ SGACL
Enforcement ผิดกลุ่มโดยไม่มีการเตือน** (เพราะ SGT เป็นแค่เลข ไม่มีการ Validate ความหมายข้าม Site
โดยอัตโนมัติ)

### SGT Numbering Scheme สำหรับ Multi-Site: แบ่งช่วงตาม Site ID

| ช่วง SGT (Decimal) | ประเภท | คำอธิบาย |
|---|---|---|
| **2 – 999** | **Global/Shared SGT** | SGT ที่มีความหมายเดียวกันทุก Site เสมอ — ใช้ค่าเดิมจาก [Part 44](part-044-dot1x-trustsec.md) ต่อไป: `SALES_SGT=100`, `VOICE_SGT=110`, `SERVERS_SGT=120`, `PRINTER_SGT=130` (กลุ่มที่มีอยู่ทุกอาคารและควร Policy เดียวกันทุกที่) |
| **1000 – 1999** | **FS-CAMPUS-CORE Local SGT** | SGT เฉพาะของ Site นี้เท่านั้น เช่น `HQ_EXEC_SGT=1000`, `FINANCE_CONTRACTOR_SGT=1010` |
| **2000 – 2999** | **FS-CAMPUS-RD Local SGT** | เช่น `LAB_EQUIPMENT_SGT=2000`, `RD_VISITOR_SGT=2010` |
| **3000 – 3999** | **FS-CAMPUS-OPS Local SGT** | เช่น `MFG_PLC_SGT=3000`, `MFG_HMI_SGT=3010`, `GUEST_WIFI_SGT=3020` |
| **4000+** | **สำรองสำหรับ Fabric Site ใหม่ในอนาคต** | ทุก Site ใหม่ที่เพิ่มเข้ามาภายหลัง (ขยายจาก 3 เป็น 4+ Site) จองช่วง 4000-4999, 5000-5999 ตามลำดับ |

```
กฎการตั้งชื่อ: <Site Block> + <Local Sequence>
  ตัวอย่าง: 2010 = FS-CAMPUS-RD (Block 2000) + Local Sequence 010 (RD_VISITOR)
  วิศวกรที่เห็นเลข 2010 อ่านได้ทันทีว่า "SGT นี้เป็นของ FS-CAMPUS-RD" โดยไม่ต้องเปิด ISE ดูก่อน
```

### เหตุผลที่ต้องแยก Global กับ Local SGT อย่างชัดเจน

| คำถาม | คำตอบ |
|---|---|
| SGT อะไรที่ SGACL ควร Enforce เหมือนกันทุก Site | Global SGT (2-999) เท่านั้น — เพราะเป็นกลุ่มที่มีความหมายเดียวกันทุกที่ |
| SGT อะไรที่ต้อง Enforce เฉพาะ Site ตัวเอง | Local SGT (1000+) — SGACL ที่เกี่ยวกับ Local SGT ควร Scope ไว้ที่ Border Node ของ Site นั้นเท่านั้น ไม่ต้อง Push ไปทุก Site |
| ถ้า Local SGT ของ Site หนึ่งข้ามไปโผล่อีก Site ได้ไหม (ผ่าน SD-Access Transit) | ได้ทางเทคนิค (SGT เดินทางไปกับ VXLAN-GPO ตาม Step 761) แต่ **ไม่ควรออกแบบ Policy ให้พึ่งพาแบบนี้** เพราะทีมของอีก Site อาจไม่รู้จัก Local SGT ของ Site อื่นเลย — ควร Promote เป็น Global SGT ถ้าต้องใช้ข้าม Site จริงๆ |

### การจัดการ SGACL Matrix แบบ Centralized ผ่าน ISE/DNAC

Enterprise ระดับ 10 อาคารต้องมี **SGACL Matrix ขนาดใหญ่** (Global SGT 4 กลุ่ม × Local SGT รวม
~9 กลุ่มจาก 3 Site = Matrix หลายสิบ Cell) — Cisco ISE คือระบบที่เก็บ **TrustSec Matrix** ทั้งหมดไว้
ที่จุดเดียว และ DNAC ดึง SGT/VN มาแสดงในหน้า Policy Workflow (ทวนจาก [Part 38 Step 376](part-038-sd-access-fundamentals.md))

```
ISE TrustSec Matrix (แนวคิด — บริหารจากศูนย์กลางเดียว):

                    │ SALES │ SERVERS │ HQ_EXEC │ LAB_EQUIP │ MFG_PLC │ GUEST_WIFI │
                    │ (100) │  (120)  │ (1000)  │  (2000)   │ (3000)  │   (3020)   │
────────────────────┼───────┼─────────┼─────────┼───────────┼─────────┼────────────┤
SALES (100)         │   -   │ Permit* │  Deny   │   Deny    │  Deny   │    Deny    │
SERVERS (120)       │Permit*│    -    │ Permit  │  Permit   │  Deny   │    Deny    │
HQ_EXEC (1000)      │ Permit│ Permit  │    -    │   Deny    │  Deny   │    Deny    │
LAB_EQUIP (2000)    │  Deny │ Permit  │  Deny   │     -     │  Deny   │    Deny    │
MFG_PLC (3000)      │  Deny │  Deny   │  Deny   │   Deny    │    -    │    Deny    │
GUEST_WIFI (3020)   │  Deny │  Deny   │  Deny   │   Deny    │  Deny   │      -     │

* Permit เฉพาะ Port ที่กำหนด (HTTPS/443) ไม่ใช่ Permit ip ทั้งหมด — ตาม SGACL จริงจาก Part 44
```

- **MFG_PLC (SGT 3000)** ถูก Deny จากทุกกลุ่มโดยไม่มีข้อยกเว้น — สะท้อนหลัก **Zero Trust สำหรับ
  OT/IoT Segment** ที่ควร Isolate เด็ดขาดจาก IT Segment ทั่วไป (จะเจาะลึก ISE Profiling สำหรับ
  IoT ที่ [Part 78](part-078-security-integration-ise-trustsec-ftd-umbrella.md))
- **DNAC ไม่ได้เก็บ Matrix เอง** — DNAC เป็นแค่ "หน้าจอ" ที่คุยกับ ISE ผ่าน pxGrid (Platform
  Exchange Grid) เพื่อดึง/Push SGACL — **ISE คือ Single Source of Truth ของ SGT/SGACL จริง**
  ส่วน DNAC เป็น Single Source of Truth ของ VN/Fabric Topology (แบ่งหน้าที่กันชัดเจน)

> **สำหรับข้อสอบ CCIE**: คำถามแนว "ที่ไหนคือแหล่งเก็บ TrustSec Matrix ตัวจริง" ตอบ **Cisco ISE**
> เสมอ ไม่ใช่ DNAC — DNAC เป็นเพียง GUI ที่เรียก ISE ผ่าน **pxGrid** ให้วิศวกรกำหนด Policy ได้จาก
> จุดเดียวโดยไม่ต้องสลับไปเปิดหน้า ISE ตรงๆ

---

## Step 768 — Extended Node / Policy Extended Node ที่ Scale: แผน Phased-Replacement

### ทวนคำเตือนจาก Part 38 Step 379: ACCESS-SW1-4 เป็น Hardware ที่ไม่รองรับ Fabric Edge Node เต็มรูปแบบ

[Part 38 Step 379](part-038-sd-access-fundamentals.md) เตือนไว้ชัดเจนว่า **ACCESS-SW1–4** (Catalyst
9200/2960 ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md)) **ไม่รองรับ VXLAN/LISP ทางกายภาพ**
— ปัญหานี้ **ไม่ได้เกิดแค่ 4 ตัวใน Lab หลัก** แต่เกิดจริงกับ Enterprise ทุกที่ที่มี Switch รุ่นเก่าเป็น
จำนวนมาก (สมมติว่า Campus 10 อาคารจาก Part 72 มี Access Switch แบบนี้กระจายอยู่ ~120 ตัวจาก
ทั้งหมด ~450 ตัว) — การ Upgrade Hardware ทั้งหมดพร้อมกันเป็นไปไม่ได้ในทางปฏิบัติ (งบประมาณ +
Downtime) ต้องมี **แผน Phased-Replacement ที่สมจริง**

### แผน Phased-Replacement 4 Phase สำหรับ Campus 10 อาคาร

```
Phase 0 (เดือน 1-2): Inventory & Classification
  - สำรวจ Hardware จริงทุกตู้ Wiring Closet ทั้ง 10 อาคาร แยกเป็น 3 กลุ่ม:
    (ก) รองรับ Fabric Edge Node ได้จริง (Catalyst 9300/9200 รุ่นที่ผ่านเกณฑ์)
    (ข) รองรับ Policy Extended Node (PEN) ได้ (Catalyst 9200L บางรุ่น)
    (ค) รองรับได้แค่ Extended Node (EN) เท่านั้น หรือไม่รองรับเลย (2960/2960-X รุ่นเก่ามาก)
  - Output: ตารางแผนที่ Switch ต่ออาคารพร้อม Classification ชัดเจน

Phase 1 (เดือน 3-6): Deploy Extended Node ให้กลุ่ม (ค) ทันที — ไม่รอ Hardware ใหม่
  - ต่อ Switch กลุ่ม (ค) เข้า Fabric ผ่าน Trunk ไปยัง Fabric Edge Node ตัวจริงที่ใกล้ที่สุด
  - Onboard ผ่าน DNAC PnP (Plug-and-Play) — ได้ประโยชน์จาก Fabric ทันที (Endpoint เข้า VN/SGT ได้)
    แม้ Enforce SGACL ยังทำไม่ได้ที่ตัวมันเอง (ต้องรอ Traffic ไปถึง Edge Node ตัวจริงก่อน)

Phase 2 (เดือน 7-14): Upgrade กลุ่ม (ค) เป็น Policy Extended Node หรือ Fabric Edge Node เต็มรูปแบบ
  ตามรอบ Budget/Lifecycle ปกติ (ไม่ต้องเร่งนอกรอบ) — เรียงลำดับความสำคัญตาม Risk:
    ลำดับ 1: Building 8-9 (FS-CAMPUS-OPS, Manufacturing/DC) — เพราะต้องการ SGACL Enforcement
             เข้มงวดสำหรับ OT/IoT (ตาม Matrix ใน Step 767) มากที่สุด → เลือก Policy Extended Node
    ลำดับ 2: Building 5-6 (FS-CAMPUS-RD) — Lab-heavy, IoT ทดสอบสูง → Policy Extended Node
    ลำดับ 3: Building 1-4 (FS-CAMPUS-CORE) — Office ทั่วไป, Risk ต่ำกว่า → Fabric Edge Node เต็มรูปแบบ
             ได้เลยเมื่อ Budget อนุมัติ Hardware ใหม่ (ไม่ต้องผ่าน PEN ก่อน)

Phase 3 (เดือน 15+): ทยอย Retire Extended Node/PEN ที่เหลือทั้งหมด
  - เมื่อ Switch ทุกตัวอัพเกรดเป็น Fabric Edge Node เต็มรูปแบบแล้ว ปิด Trunk Uplink ที่ไม่จำเป็น
  - Enforce SGACL ได้ทุกจุดของ Campus (100% Distributed Enforcement — ไม่ต้องพึ่ง Edge Node
    ตัวจริงที่อื่น Enforce แทนอีกต่อไป)
```

### เกณฑ์เลือกระหว่าง Extended Node กับ Policy Extended Node ที่ Scale

| เกณฑ์ | เลือก Extended Node (EN) | เลือก Policy Extended Node (PEN) |
|---|---|---|
| งบประมาณช่วงนี้ | จำกัดมาก ต้องการ Onboard เข้า Fabric เร็วที่สุดโดยไม่เพิ่มค่าใช้จ่าย Hardware | มีงบพอสำหรับ Switch รุ่นสูงกว่าเล็กน้อย (เช่น 9200L บางรุ่น) |
| ความต้องการ SGACL Enforcement ที่จุดนั้น | ต่ำ (ยอมรับให้ Traffic ไปถึง Edge Node ตัวจริงก่อนค่อย Enforce) | สูง (ต้องการ Isolate ทันทีที่ Switch เอง เช่น OT/IoT ใน Building 8-9) |
| ระยะเวลาที่วางแผนใช้งานก่อน Retire | สั้น (เป็น Bridge ไปสู่ Fabric Edge Node เต็มรูปแบบเร็วๆ นี้) | ยาวกว่า (อาจใช้งานเป็น Steady-State ระยะยาวถ้า Site นั้นมีข้อจำกัดพื้นที่ทางกายภาพ) |

### ตาราง Mapping จริงสำหรับ ACCESS-SW1-4 ของ Lab หลัก (ขยายจาก Part 38 Step 379)

| Device | สถานะปัจจุบัน (Part 38) | Phase ที่ควรอยู่ตามแผนนี้ | เป้าหมายสุดท้าย |
|---|---|---|---|
| ACCESS-SW1, ACCESS-SW2 | Catalyst 9200/2960 — Extended Node ชั่วคราว | Phase 1 (Deploy EN ทันที) → Phase 2 ลำดับ 3 (Office, Risk ต่ำ) | Fabric Edge Node เต็มรูปแบบ |
| ACCESS-SW3, ACCESS-SW4 | Catalyst 9200/2960 — Extended Node ชั่วคราว, ผูกกับ VLAN 30/40 (Servers/WIFI) | Phase 1 (Deploy EN ทันที) → Phase 2 ลำดับ 1-2 (มี Server/WIFI Traffic ที่ต้องการ Policy เข้มกว่า) | Policy Extended Node ก่อน แล้วค่อย Fabric Edge Node เมื่อ Budget อนุมัติ |

> **สำหรับข้อสอบ CCIE**: แผน Migration ที่ดีต้อง **จัดลำดับตาม Risk ของ Traffic ที่ Switch นั้น
> Handle** ไม่ใช่จัดลำดับตามอายุ Hardware อย่างเดียว — Switch เก่าที่ Handle Traffic ความเสี่ยงต่ำ
> (Office ทั่วไป) รอได้นานกว่า Switch ที่ Handle Traffic ความเสี่ยงสูง (OT/IoT/Server) แม้ Hardware
> จะเก่าเท่ากันก็ตาม

---

## Step 769 — SD-Access Assurance & Troubleshooting ที่ Scale

### DNAC/Catalyst Center Assurance สำหรับ Fabric Health ข้ามหลาย Site

[Part 50 Step 495](part-050-dna-center.md) สอน Assurance ระดับทั่วไป (Network Health, Client
Health, Path Trace) — สำหรับ **Multi-Fabric ที่ Scale 3 Fabric Site** DNAC/Catalyst Center มี
Dashboard เฉพาะทางเพิ่มเติมที่ต้องรู้:

| Dashboard/View | แสดงอะไร | ใช้ตรวจอะไรที่ Scale หลาย Site |
|---|---|---|
| **Fabric Site Health Summary** | สรุปสถานะรวมของทุก Fabric Site ในหน้าเดียว (Healthy/Warning/Critical ต่อ Site) | เห็นภาพรวมว่า Site ไหนมีปัญหาก่อนไล่ดูรายละเอียด — สำคัญมากเมื่อมี 3+ Site พร้อมกัน |
| **LISP Session Status ต่อ Site** | สถานะ Session ระหว่าง Edge Node กับ Control Plane Node ของแต่ละ Site (Established/Down) รวมถึง **Transit Control Plane Node Session** ระหว่าง Site (Step 761) | แยกแยะได้ว่าปัญหาเกิด "ภายใน Site เดียว" (Edge↔CP Session ของ Site นั้น Down) หรือ "ระหว่าง Site" (Transit CP Session Down — กระทบ Inter-Site Roaming/Transit ทั้งหมด) |
| **VXLAN Tunnel Health** | สถานะ RLOC-to-RLOC Tunnel ทุกคู่ที่ Active อยู่ พร้อม Packet Loss/Latency รายทาง | ตรวจ MTU Mismatch (ทวนจาก [Part 38 Step 377](part-038-sd-access-fundamentals.md) — VXLAN Overhead ~50 byte) ที่มักเป็นสาเหตุ Tunnel "ขึ้นแต่ใช้งานไม่ได้" (Session Up แต่ Traffic ใหญ่ Drop) |
| **Fabric-wide Path Trace ข้าม Site** | Path Trace ที่วิ่งข้าม Fabric Site ผ่าน Transit ได้ในคำสั่งเดียว (ต่างจาก Path Trace ปกติที่มักจำกัดอยู่ Site เดียว) | ตรวจว่า SGT/VN คงอยู่ครบตลอดเส้นทางข้าม Site จริงหรือไม่ (ยืนยันสิ่งที่ Step 761 อธิบายเชิง Theory) |

### CLI-Level Troubleshooting เมื่อไม่มี GUI Access (สถานการณ์จริงที่ต้องเจอ)

แม้ SD-Access ควบคุมผ่าน DNAC เป็นหลัก แต่วิศวกร CCIE ต้อง Troubleshoot ได้แม้ **DNAC ล่มหรือ
เข้าไม่ได้ชั่วคราว** — คำสั่งเหล่านี้ (ทวนพื้นฐานจาก [Part 38 Step 380](part-038-sd-access-fundamentals.md)
และขยายเพิ่มสำหรับ Multi-Site) คือสิ่งที่ต้องใช้จริงบน Fabric Node:

```
! 1. ตรวจ LISP Session ทั้งหมดที่ Edge Node — แยกว่า Session ไหนเป็น Local CP, ไหนเป็น Transit CP
Edge-Node# show lisp session
Sessions for VRF default, total: 3, established: 3
Peer                State      Up/Down       In/Out    Users
1.1.201.1            Up         2w3d          22/18     4     <- Local Control Plane Node (Site เดียวกัน)
1.1.201.2            Up         2w3d          22/18     4     <- Local Control Plane Node (สำรอง)
1.1.200.1            Up         1w0d          8/5       1     <- Transit Control Plane Node (ข้าม Site)

! 2. ตรวจ EID ที่ Register ไว้กับ Site นี้ พร้อม Instance ID (แยก VN แต่ละตัว)
Edge-Node# show lisp instance-id 4097 dynamic-eid all
LISP Dynamic EID Information for VRF "EMPLOYEE_VN" (Instance ID 4097)
EID Prefix              Uptime      Last Used     Site Origin
10.10.10.5/32           00:42:11    00:00:01      Local
10.20.55.3/32           00:03:40    00:00:00      Remote (via Transit — FS-CAMPUS-RD)

! 3. ตรวจ VXLAN Tunnel Interface และ Encapsulation Counter (ยืนยัน Data Plane ทำงานจริง ไม่ใช่
!    แค่ Control Plane Session Up)
Edge-Node# show interface vxlan 1 | include packets
  1,204,552 packets input, 812,004 packets output

! 4. ตรวจ MTU บน Underlay Path ที่ VXLAN Tunnel วิ่งผ่าน (สาเหตุอันดับ 1 ของ "LISP Up แต่ Traffic
!    ใหญ่ใช้งานไม่ได้" — ทวนจาก Part 38 Step 377)
Edge-Node# show interface TenGigabitEthernet1/0/1 | include MTU
  MTU 9216 bytes, BW 10000000 Kbit/sec

! 5. บน Control Plane Node — ดูรายชื่อ Edge Node ทั้งหมดที่ Register อยู่ ณ ขณะนี้ (เทียบกับจำนวน
!    ที่ DNAC ควรเห็น ถ้าไม่ตรงกัน = Site มี Node ที่ Session หลุดโดย DNAC ยังไม่ทัน Alert)
Control-Plane-Node# show lisp site summary
Number of configured sites: 1
Number of requested EID-prefixes: 842
Number of registered EID-prefixes: 838     <- ต่างจาก Configured 4 รายการ = ตรวจต่อว่า EID ใด Register ไม่สำเร็จ

! 6. ตรวจ SGT Binding ที่ Edge Node เพื่อยืนยัน Policy Plane ทำงานถูกต้อง (ทวนจาก Part 38/44)
Edge-Node# show cts role-based sgt-map all
Active IP-SGT Bindings Information
IP Address     SGT      Source
10.10.10.5     100      LOCAL (802.1X)
10.20.55.3     2010     REMOTE (SXP/LISP — จาก FS-CAMPUS-RD ผ่าน Transit)
```

### ลำดับการ Isolate ปัญหาแบบ Systematic (เมื่อ Endpoint คุยกันไม่ได้ข้าม Fabric Site)

```
1. ตรวจ LISP Session ที่ Edge Node ทั้ง 2 ฝั่ง (ต้นทาง/ปลายทาง) ว่า Established กับ Local CP หรือไม่
   → ถ้าไม่ Established: ปัญหาอยู่ที่ Underlay Reachability ภายใน Site นั้น (ปัญหาระดับ Part 33 เดิม)

2. ตรวจ Transit Control Plane Session จาก CP Node ของทั้ง 2 Site ว่า Established กันหรือไม่
   → ถ้าไม่ Established: ปัญหาอยู่ที่ Backbone ระหว่าง Site (WAN/MPLS/Dark Fiber ที่เชื่อม Site)

3. ตรวจว่า EID ปลายทาง Register สำเร็จหรือไม่ที่ Site ต้นทาง (show lisp site summary เทียบ
   Configured vs Registered)
   → ถ้า EID ไม่ปรากฏใน Map-Cache: ปัญหาอยู่ที่ Edge Node ปลายทางไม่ได้ Register EID นั้น
     (ตรวจ Port Status/802.1X Authentication ที่ Edge Node ปลายทางต่อ)

4. ถ้า LISP/EID ทุกอย่างปกติแต่ Traffic ยัง Drop: ตรวจ SGACL Matrix (Step 767) ว่า SGT คู่นั้น
   ถูก Deny ไว้โดยตั้งใจหรือไม่ (มักเป็นสาเหตุที่ถูกมองข้ามที่สุด เพราะ Network Layer ทำงานถูกต้อง
   100% แต่ Policy Layer เป็นตัว Block)

5. ถ้า SGACL ก็ Permit ปกติ แต่ Traffic ขนาดใหญ่ (Bulk Transfer) Drop ขณะที่ Traffic ขนาดเล็ก
   (Ping) ผ่านได้ปกติ: ตรวจ MTU ตลอดเส้นทาง (ข้อ 4 ของคำสั่ง CLI ข้างบน) — เป็น Signature
   ของปัญหา VXLAN Overhead ที่ Fragment/Drop
```

> **สำหรับข้อสอบ CCIE**: ข้อสอบ Lab มักจำลองสถานการณ์ "DNAC ปิดใช้งาน/ไม่มีสิทธิ์เข้า" โดยเจตนา
> เพื่อบังคับให้ Verify ผ่าน CLI เท่านั้น — ต้องจำ `show lisp session`, `show lisp site summary`,
> `show cts role-based sgt-map all` ให้แม่น เพราะเป็น 3 คำสั่งที่ครอบคลุม Control Plane, Registration
> State, และ Policy Plane ตามลำดับ

---

## Step 770 — Lab เชิงออกแบบเต็มรูปแบบ: Multi-Fabric SD-Access สำหรับ Campus 10 อาคาร

> **ข้อจำกัดของ Lab นี้** (ทวนหลักการเดียวกับ [Part 38 Step 380](part-038-sd-access-fundamentals.md)):
> นี่คือ **Design-level/Conceptual Lab** ไม่ใช่ CLI จริงที่ Config ได้ในโปรแกรม Lab ทั่วไป เพราะ
> ต้องมี DNA Center/Catalyst Center และ ISE จริงหลายชุดในการ Provision Multi-Fabric เต็มรูปแบบ
> Lab CLI-based ของหลักสูตรหลักยังใช้สถาปัตยกรรม Traditional ตามปกติ — Part นี้รวบรวมทุก Step
> (761-769) เข้าเป็น **1 Design Document เดียว** ที่ใช้ยื่นสอบ CCIE Enterprise Infrastructure
> Design Track ได้จริง

### 1. Fabric Site Topology สรุปรวม

```
                                     ┌───────────────┐
                                     │   INTERNET     │
                                     └───────┬───────┘
                                     ┌───────┴───────┐
                                     │   ISP-RTR       │
                                     └───────┬───────┘
                          ┌──────────────────┴──────────────────┐
                          │      WAN-EDGE-1 / WAN-EDGE-2           │
                          └──────────────────┬──────────────────┘
                                    Native IP (นอก Fabric)
              ┌────────────────────────────┴────────────────────────────┐
              │       FS-CAMPUS-CORE — External Border ×2 (1.1.201.1-2)    │
              │       Building 1-4 | Distributed Role | ~4,500 Endpoint    │
              └────────────────────────────┬────────────────────────────┘
                                   Transit Control Plane
                                   (1.1.200.1 / 1.1.200.2)
                                   SD-Access Transit (LISP Pub/Sub)
              ┌──────────────────────────┬──────────────────────────────┐
   ┌──────────┴──────────┐     ┌──────────┴──────────┐
   │  FS-CAMPUS-RD          │     │  FS-CAMPUS-OPS         │
   │  Building 5-7          │     │  Building 8-10         │
   │  Internal Border only  │     │  Internal Border only  │
   │  Distributed+FiaB      │     │  Distributed+FiaB      │
   │  1.1.202.x              │     │  1.1.203.x              │
   │  ~3,200 Endpoint        │     │  ~2,100 Endpoint        │
   └─────────────────────────┘     └─────────────────────────┘
```

- เฉพาะ **FS-CAMPUS-CORE** ทำหน้าที่ **External Border** (Step 764) — Site อื่นออก Internet ผ่าน
  SD-Access Transit มาที่ Site นี้เสมอ (ประหยัดค่าใช้จ่าย Internet Circuit ซ้ำซ้อนที่ทุกอาคาร)
- FS-CAMPUS-RD และ FS-CAMPUS-OPS มีแค่ **Internal Border** เชื่อมกลับเข้า FS-CAMPUS-CORE และ
  Data Center Annex (Building 9) เท่านั้น

### 2. Role & Sizing ต่อ Fabric Site (สรุปจาก Step 762)

| Fabric Site | Border Node | Control Plane Node | Edge Node | Fabric-in-a-Box |
|---|---|---|---|---|
| FS-CAMPUS-CORE | 2× Catalyst 9500 (External) | 2× Catalyst 9500 | ~40× Catalyst 9300 (Building 1-4) | ไม่มี (Distributed เต็มรูปแบบ) |
| FS-CAMPUS-RD | 2× Catalyst 9500 (Internal) | 2× Catalyst 9500 | ~28× Catalyst 9300 (Building 5-6) | Building 7 (Training Center) — 1× Stack |
| FS-CAMPUS-OPS | 2× Catalyst 9500 (Internal) | 2× Catalyst 9500 | ~18× Catalyst 9300 (Building 8-9) | Building 10 (Guest Center) — 1× Stack |

### 3. Anycast Gateway & VN/SGT Numbering (สรุปจาก Step 763, 767)

| VN | VLAN เดิมที่ครอบ (00-ip-address-plan.md) | Anycast Gateway (ตัวอย่างต่อ Site) | Global SGT ที่เกี่ยวข้อง |
|---|---|---|---|
| CORP_VN | VLAN 10 (SALES), 20 (VOICE), 30 (SERVERS) | `10.10.10.1` ซ้ำทุก Edge Node ในแต่ Site | SALES=100, VOICE=110, SERVERS=120 |
| WIFI_VN | VLAN 40 (WIFI) | `10.10.40.1` ซ้ำทุก Edge Node | PRINTER=130 (สำหรับ Printer ที่อยู่ VLAN นี้) |
| MGMT_VN | VLAN 99 (MGMT) | `10.10.99.1` ซ้ำทุก Edge Node | (Global — ไม่มี SGT Micro-seg เพิ่ม เพราะเป็น Infra Management) |

SGT Local ต่อ Site อ้างอิงตาราง Numbering เต็มจาก Step 767 (1000+ = CORE, 2000+ = RD,
3000+ = OPS)

### 4. Multicast Design (สรุปจาก Step 765)

| Fabric Site | วิธีที่เลือก | เหตุผล |
|---|---|---|
| FS-CAMPUS-CORE | Head-End Replication (Default) | Office Traffic — Multicast Group มีไม่มาก (ส่วนใหญ่เป็น Video Conference เฉพาะกลุ่ม) |
| FS-CAMPUS-RD | Native Multicast (PIM-SM ที่ Underlay) | R&D มี Lab Testing Feed/Video Streaming กระจายไปหลาย Edge Node พร้อมกัน — Efficiency สำคัญกว่า |
| FS-CAMPUS-OPS | Native Multicast (PIM-SM ที่ Underlay) | Manufacturing ใช้ Multicast สำหรับ SCADA/Control System ที่ต้องการ Replicate ประสิทธิภาพสูงและ Latency ต่ำ |

RP วางแบบ **Anycast RP ที่ Border Node ของแต่ละ Site** พร้อม MSDP/Anycast-RP Sync ข้าม Site
สำหรับ Multicast Group ที่ต้องข้าม Fabric Site (ผ่าน SD-Access Transit)

### 5. Fabric Wireless Design (สรุปจาก Step 766)

| Fabric Site | Fabric WLC | AP Count (ประมาณ) |
|---|---|---|
| FS-CAMPUS-CORE | Fabric WLC-CORE (Catalyst 9800, HA Pair) | ~180 |
| FS-CAMPUS-RD | Fabric WLC-RD (Catalyst 9800, HA Pair) | ~130 |
| FS-CAMPUS-OPS | Fabric WLC-OPS (Catalyst 9800, HA Pair) | ~90 |

Inter-Site Roaming Policy: Client ที่ Roam ข้าม Site ต้อง Re-Authenticate ผ่าน 802.1X/ISE (ยอมรับ
Trade-off ตาม Step 766) — Fast Reauthentication (802.11r/CCKM ที่เคยสอนใน Part 37) ยังใช้ได้
**เฉพาะภายใน Site เดียว** เท่านั้น

### 6. Extended Node Migration Plan (สรุปจาก Step 768)

| Phase | ช่วงเวลา | Site ที่ทำ | เป้าหมาย |
|---|---|---|---|
| Phase 0 | เดือน 1-2 | ทั้ง 3 Site | Inventory Hardware ทุกตู้ Wiring Closet |
| Phase 1 | เดือน 3-6 | ทั้ง 3 Site | Deploy Extended Node ให้ Switch รุ่นเก่าทั้งหมดทันที (รวม ACCESS-SW1-4 ของ Lab หลัก) |
| Phase 2 | เดือน 7-14 | FS-CAMPUS-OPS ก่อน → FS-CAMPUS-RD → FS-CAMPUS-CORE | Upgrade เป็น PEN/Fabric Edge Node ตาม Risk ของ Traffic |
| Phase 3 | เดือน 15+ | ทั้ง 3 Site | Retire Extended Node ที่เหลือทั้งหมด — 100% Fabric Edge Node |

### 7. Assurance Checklist ก่อนถือว่า Design นี้ Production-Ready

```
[ ] Fabric Site Health Summary ทั้ง 3 Site ขึ้น "Healthy" ต่อเนื่อง ≥ 7 วัน
[ ] LISP Session ระหว่าง Edge Node ↔ Local Control Plane Node ของทุก Site = Established 100%
[ ] Transit Control Plane Session ระหว่างทั้ง 3 Site = Established 100%
[ ] VXLAN Tunnel MTU ทุกเส้นทาง ≥ 9100 byte (รองรับ VXLAN Overhead ~50 byte เต็มที่)
[ ] SGACL Matrix ผ่าน ISE Verify แล้วว่าไม่มี Global/Local SGT ชนกัน (ตรวจตาม Numbering Step 767)
[ ] External Border ทั้ง 2 ตัวที่ FS-CAMPUS-CORE Advertise Default Route เข้า Fabric ครบทั้ง
    3 Site ผ่าน Transit (ตรวจด้วย Path Trace ข้าม Site จาก FS-CAMPUS-OPS ไป Internet)
[ ] Extended Node ที่เหลือทั้งหมดมีแผน Phase ชัดเจนพร้อมกำหนดเวลา (ไม่มี Switch ที่ "ลอย" ไม่มี
    แผน Upgrade)
```

Design Document นี้ครอบคลุมทุกมิติที่ CCIE Enterprise Infrastructure Lab Exam กำหนดสำหรับหัวข้อ
SD-Access ระดับ Multi-Site — ตั้งแต่ Topology (Step 761), Sizing (762), Gateway Redundancy (763),
External Connectivity (764), Multicast (765), Wireless (766), Policy (767), Migration (768),
ไปจนถึง Assurance/Troubleshooting (769)

---

## แบบฝึกหัดทวนความเข้าใจ Part 77

1. SD-Access Transit ต่างจาก IP-based Transit อย่างไร และทำไม SGT/VN จึงเดินทางข้าม Fabric Site
   ได้อัตโนมัติเฉพาะเมื่อใช้ SD-Access Transit เท่านั้น?
2. L3 Anycast Gateway แก้ปัญหา HSRP-per-VLAN Scaling ที่เจอใน Part 35 ได้อย่างไร และเพราะเหตุใด
   IP Address/MAC Address ของ Gateway เดียวกันจึงซ้ำกันได้ทุก Edge Node โดยไม่ชนกัน?
3. External Border Node กับ Internal Border Node ต่างกันอย่างไรในมุมของ Route ที่ Advertise เข้า
   Fabric และทำไม Anywhere Border จึงไม่ใช่ตัวเลือกแรกสำหรับ Design ขนาดใหญ่?
4. Head-End Replication กับ Native Multicast (PIM ที่ Underlay) ต่างกันอย่างไร และแบบไหนที่เป็น
   ค่า Default ของ SD-Access เมื่อเปิด Multicast ครั้งแรก?
5. ทำไม SGT Numbering Scheme ของ Multi-Site ต้องแบ่งเป็น Global SGT กับ Local SGT ต่อ Site และ
   ระบบใดคือแหล่งเก็บ TrustSec Matrix ตัวจริง (ไม่ใช่ DNAC)?

**เฉลย:**

1. **SD-Access Transit** ใช้ **Transit Control Plane Node** รัน LISP Pub/Sub ข้าม Fabric Site
   ทำให้ VXLAN Encapsulate ต่อเนื่องจาก Edge Node ต้นทางถึง Edge Node ปลายทางข้าม Site โดยตรง
   (ไม่ Decapsulate กลางทาง) SGT/VN จึงเดินทางไปกับ VXLAN-GPO ได้ครบถ้วนเหมือนอยู่ Site เดียวกัน
   ส่วน **IP-based Transit** ต้อง Decapsulate ที่ Border Node ของแต่ละ Site ก่อนแล้ว Route แบบ IP
   ธรรมดา ทำให้ SGT/VN ขาดตอน ต้องพึ่งกลไกเสริม (SXP/Manual VRF Mapping) จึงจะส่งต่อได้
2. Anycast Gateway ให้ Edge Node **ทุกตัว** ใน Fabric Site มี SVI/MAC เดียวกันของ Subnet นั้น
   Active พร้อมกันหมดโดยไม่ต้อง Election — เพราะ Traffic ระหว่าง Endpoint ไม่ได้พึ่ง Gateway
   ในการหาปลายทางอีกต่อไป แต่พึ่ง **LISP Host-Route (EID/RLOC)** ที่ Control Plane Node รู้ตำแหน่ง
   จริงของ Endpoint แต่ละตัว ทำให้ "Gateway ซ้ำกันทุกที่" ไม่ก่อให้เกิดปัญหา Routing Loop/Conflict
   และไม่ต้องมี HSRP/VRRP Election, ไม่มี Failover Delay เหมือนก่อน
3. **External Border** Advertise **Default Route (0.0.0.0/0)** เข้า Fabric สำหรับปลายทางที่ไม่รู้จัก
   ทั้งหมด (Internet/WAN) ส่วน **Internal Border** Advertise **Specific Prefix** ที่รู้จักจริง (เช่น
   Data Center ภายใน) — Anywhere Border ทำทั้งสองอย่างในอุปกรณ์เดียว แต่รวม Failure Domain ของ
   ทั้ง Internet Edge และ Internal DC Edge ไว้จุดเดียว ซึ่งเสี่ยงเกินไปสำหรับ Design ขนาดใหญ่ที่
   ต้องการแยก Failure Domain ชัดเจนตามหลักการ High Availability
4. **Head-End Replication (HER)** ให้ Edge Node ต้นทาง Copy Packet เป็น Unicast VXLAN แยกส่งไป
   ทีละ RLOC ปลายทาง (ไม่ต้อง Config PIM ที่ Underlay แต่ Bandwidth โตตามจำนวน Receiver) ส่วน
   **Native Multicast** เปิด PIM-SM จริงที่ Underlay ให้ Intermediate Node ทำ Replicate ตาม
   Multicast Tree (Efficient กว่ามากแต่ต้อง Config PIM เพิ่ม) — **HER คือค่า Default** ที่ DNAC
   ตั้งให้เมื่อเปิด Multicast บน Fabric ครั้งแรก
5. ต้องแบ่งเพราะแต่ละ Fabric Site มักมีทีมงานกำหนด SGT ของตัวเองแยกกัน ถ้าไม่จองช่วงเลขไว้ล่วงหน้า
   จะเกิด SGT ชนกันข้าม Site เมื่อเชื่อมด้วย SD-Access Transit ทำให้ SGACL Enforcement ผิดกลุ่ม
   โดยไม่มีการเตือน — **Global SGT (เช่น 2-999)** ใช้กับกลุ่มที่มีความหมายเดียวกันทุก Site ส่วน
   **Local SGT (แบ่ง Block ตาม Site เช่น 1000+, 2000+, 3000+)** ใช้เฉพาะภายใน Site นั้น ระบบที่
   เก็บ **TrustSec Matrix ตัวจริง** คือ **Cisco ISE** (ผ่าน pxGrid ให้ DNAC ดึงมาแสดงเป็น GUI) ไม่ใช่
   DNAC เอง

---

## สรุป Part 77

Part นี้ยกระดับ SD-Access จากแนวคิด 1 Fabric Site ใน [Part 38](part-038-sd-access-fundamentals.md)
ไปสู่การออกแบบ **Multi-Fabric ระดับ CCIE Enterprise Infrastructure** เต็มรูปแบบสำหรับ Campus
10 อาคารจาก [Part 72](part-072-large-campus-fabric-design.md) — เราครอบคลุม **SD-Access Transit
vs IP-based Transit** (Step 761), **Fabric-in-a-Box vs Distributed Role** พร้อม Sizing Guidance
(Step 762), **L3 Anycast Gateway** ที่แก้ปัญหา HSRP-per-VLAN Scaling จาก [Part 35](part-035-advanced-fhrp-campus-ha.md)
อย่างสิ้นเชิง (Step 763), **Border Node 3 ประเภท** พร้อม Worked Design เชื่อม WAN-EDGE-1/2
(Step 764), **Multicast บน Fabric** ผ่าน Head-End Replication และ Native Multicast (Step 765),
**Wireless ข้าม Fabric Site** พร้อม Trade-off ของ Inter-Site Roaming (Step 766), **SGT Numbering
Scheme หลาย Site** และ SGACL Matrix ผ่าน ISE (Step 767), **แผน Phased-Replacement** สำหรับ
Extended Node/Policy Extended Node ที่ Scale (Step 768), และ **Assurance/CLI Troubleshooting**
เมื่อไม่มี GUI (Step 769) ปิดท้ายด้วย **Lab เชิงออกแบบเต็มรูปแบบ** ที่รวมทุก Step เข้าเป็น Design
Document เดียวสำหรับ Campus 10 อาคาร (Step 770)

SD-Access ที่เรียนไปทั้ง Part 38 และ Part 77 นี้คือ **Fabric สำหรับ Wired/Wireless Endpoint บน
Campus** — Part ถัดไปจะเปลี่ยนมุมมองจาก **Network Layer ไปสู่ Security Layer** เต็มรูปแบบ: การ
Integrate **Cisco ISE, TrustSec, Firepower Threat Defense (FTD), และ Umbrella** เข้าด้วยกันเป็น
สถาปัตยกรรม Security แบบองค์รวมที่ครอบทั้ง SD-Access Fabric, SD-WAN (Part 76), และ Endpoint
ทุกจุดขององค์กร

**ไปต่อ:** [Part 78 — Security Integration: ISE, TrustSec, FTD, Umbrella →](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
