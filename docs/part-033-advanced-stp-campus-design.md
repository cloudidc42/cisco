# Part 33 — Advanced STP & Campus Multilayer Design
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 321–330 จาก 1000**

> ต่อจาก [Part 32 — Policy-Based Routing (PBR)](part-032-policy-based-routing.md) ที่ปิดท้ายกลุ่ม
> Routing ขั้นสูงของ CCNP ENCOR แล้ว Part นี้จะย้อนกลับไปแก้ **หนี้ทางสถาปัตยกรรม** ที่ค้างไว้ตั้งแต่
> [Part 10](part-010-inter-vlan-routing.md) — ตอนนั้นเราออกแบบให้ DIST-SW แต่ละตัวต่อ CORE เพียง
> **ตัวเดียว** (single-homed) เพื่อให้ Static Route/OSPF เบื้องต้นเรียนง่ายขึ้น พร้อมระบุไว้ชัดเจนใน
> [`00-ip-address-plan.md`](00-ip-address-plan.md) ว่า **"Dual-homed DIST↔CORE จะเพิ่มเข้ามาใน
> Part 33-35"** — **Part นี้คือ Part ที่ทำตามสัญญานั้น** เราจะออกแบบใหม่ให้ DIST-SW ทุกตัวต่อสาย
> Routed ไปทั้ง **CORE-SW1 และ CORE-SW2 พร้อมกัน** เปลี่ยนปัญหา "L2 Redundancy ที่ต้องพึ่ง STP
> Blocking" ให้กลายเป็นปัญหา "L3 ECMP/Failover ที่เร็วกว่าและใช้ Bandwidth เต็มทุกเส้น" พร้อมทบทวน
> ว่า STP (จาก [Part 5](part-005-stp-rstp-mstp.md)) ยังจำเป็นตรงไหนบ้างในสถาปัตยกรรมใหม่นี้ และ
> เจาะลึกกลไกภายในของ Multilayer Switch (CEF/TCAM) ที่ทำให้ทุกอย่างนี้เป็นไปได้ที่ Wire-speed

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 321 | ทบทวนหลักการออกแบบ Campus Network ระดับ CCNP — เป้าหมาย Redundancy ของ 3-Tier Model และ Topology ใหม่แบบ Full Mesh Dual-Homed |
| 322 | Routed Access vs Traditional Design — ทำไมหลักสูตรนี้ยังใช้ Traditional (L2 Access + L3 Distribution) |
| 323 | L2 vs L3 Link ใน Topology ใหม่ — ตัดสินใจว่าลิงก์ไหนควรเป็น Routed และลิงก์ไหนยังต้องเป็น Trunk |
| 324 | Redesigning for Full HA — Config เต็มรูปแบบ Dual-Homed Routed Uplink พร้อม OSPF ECMP |
| 325 | STP ยังจำเป็นตรงไหนบ้างใน Campus ใหม่ — ทบทวนและแก้ไข Root Bridge Design จาก Part 5 |
| 326 | Multilayer Switching Internals — CEF, FIB, Adjacency Table และ TCAM |
| 327 | Equal-Cost Multi-Path (ECMP) — `maximum-paths`, Per-Packet vs Per-Destination Load Balancing |
| 328 | UDLD (UniDirectional Link Detection) — ตรวจจับ Fiber ที่ขาดทางเดียว |
| 329 | Verification ทั้ง Topology ใหม่ — `show ip route`, `show ip cef`, `show udld`, `show ip ospf neighbor` |
| 330 | Lab เต็มรูปแบบ: Dual-Homed Routed Uplink + OSPF ECMP + UDLD Aggressive พร้อมทดสอบ Failover |

---

## Step 321 — ทบทวนหลักการออกแบบ Campus Network ระดับ CCNP

### เป้าหมาย Redundancy ของ 3-Tier Hierarchical Model (ทบทวนเชิงลึกจาก Part 1)

โมเดล Core/Distribution/Access ที่เราใช้มาตั้งแต่ [Part 1](part-001-networking-fundamentals.md)
มีเป้าหมายสูงสุดข้อเดียวคือ **"No Single Point of Failure" (NSPOF)** — หมายความว่า **ไม่มี
อุปกรณ์ตัวใดหรือสายเส้นใดเส้นเดียว ที่ถ้าล่มแล้วจะทำให้ Traffic ระหว่าง Layer หยุดทำงานทั้งหมด**
กฎนี้ต้องถูกบังคับใช้ใน **ทุก Tier** ไม่ใช่แค่บาง Tier:

| Tier | Redundancy ที่ควรมี | สถานะใน Lab ของเราก่อน Part นี้ |
|---|---|---|
| Access ↔ Distribution | Access switch dual-homed ไป Distribution 2 ตัว | ✅ มีแล้วตั้งแต่ Part 1 (ACCESS-SW1 → DIST-SW1+DIST-SW2) และ Part 5 คุม STP ให้ปลอดภัย |
| Distribution ↔ Core | Distribution switch dual-homed ไป Core 2 ตัว | ❌ **ยังเป็น single-homed** — DIST-SW1/2 ต่อ CORE-SW1 เท่านั้น, DIST-SW3/4 ต่อ CORE-SW2 เท่านั้น |
| Core ↔ Core | Link ตรงระหว่าง Core คู่ | ✅ มีแล้วตั้งแต่ Part 6 (Po1 LACP) |
| Default Gateway (SVI) | Gateway ซ้ำกัน 2 ตัวต่อ VLAN | ✅ มีแล้วตั้งแต่ Part 18 (HSRP) |
| Routing Protocol | มากกว่า 1 เส้นทางไปทุกปลายทาง | ⚠️ มีทางกายภาพซ้ำ (ผ่าน Po1) แต่ไม่ใช่ ECMP จริง เพราะ DIST แต่ละตัวเห็น Core แค่ตัวเดียว |

จะเห็นว่า **แถวเดียวที่ยังไม่ผ่านเกณฑ์ NSPOF คือ Distribution ↔ Core** — ถ้า CORE-SW1 ล่ม
ทั้ง DIST-SW1 และ DIST-SW2 จะขาดการเชื่อมต่อไปยัง Core Layer ทันที (แม้ CORE-SW2 จะยังทำงาน
ปกติอยู่ก็ตาม) นี่คือช่องโหว่ทางสถาปัตยกรรมที่ Part 10 ตั้งใจปล่อยไว้เพื่อความง่ายในการสอน
และ Part นี้คือจุดที่เราจะปิดช่องโหว่นี้อย่างสมบูรณ์

### ทำไมต้อง "Dual-Homing ทุก Tier" — ไม่ใช่แค่บาง Tier

หลักการออกแบบ Enterprise Campus ระดับ CCNP (Cisco เรียกว่า **"Full Mesh" หรือ
"Partial Mesh" Distribution-to-Core Design**) กำหนดว่า Distribution switch **ทุกตัว** ต้องมี
สายไปยัง Core switch **ทุกตัว** — ถ้ามี Core 2 ตัวและ Distribution 4 ตัว จะต้องมีสายทั้งหมด
**2 × 4 = 8 เส้น** (Full Mesh เต็มรูปแบบ) แทนที่จะเป็น 4 เส้น (Single-homed แบบเดิม)

```
Full Mesh Core-Distribution (เป้าหมายของ Part นี้):

DIST-SW1 ──┬─────────────► CORE-SW1
           └─────────────► CORE-SW2
DIST-SW2 ──┬─────────────► CORE-SW1
           └─────────────► CORE-SW2
DIST-SW3 ──┬─────────────► CORE-SW1
           └─────────────► CORE-SW2
DIST-SW4 ──┬─────────────► CORE-SW1
           └─────────────► CORE-SW2

= 8 เส้น Physical (แทนที่ 4 เส้นเดิม) — ทุก DIST เห็น Core ทั้งคู่โดยตรง
```

### Full Redesigned Topology Diagram (Enterprise Lab หลังจาก Part นี้)

```
                                   ┌──────────────────────────┐
                                   │   CORE-SW1  ◄══Po1 LACP══► CORE-SW2   │
                                   │  (Catalyst 9500, L3 routed only)      │
                                   └──┬────┬────┬────┬─────┬────┬────┬────┘
                        Te1/0/1 ──────┘    │    │    │     │    │    └────── Te1/0/4 (CORE-SW2)
                        (เดิม)    Te1/0/2──┘    │    │     │    └── Te1/0/3 (เดิม, CORE-SW2)
                                  (เดิม)         │    │     │
                         Te1/0/3 (ใหม่)──────────┘    │     └── Te1/0/4 (ใหม่, CORE-SW1)
                         Te1/0/4 (ใหม่)───────────────┘
        ┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐
        │       DIST-SW1        │  │       DIST-SW2        │  │       DIST-SW3        │  │       DIST-SW4        │
        │ Te1/1/1 → CORE-SW1    │  │ Te1/1/1 → CORE-SW1    │  │ Te1/1/1 → CORE-SW2    │  │ Te1/1/1 → CORE-SW2    │
        │ Te1/1/2 → CORE-SW2 NEW│  │ Te1/1/2 → CORE-SW2 NEW│  │ Te1/1/2 → CORE-SW1 NEW│  │ Te1/1/2 → CORE-SW1 NEW│
        │ SVI: Vlan10/20/99     │  │ SVI: Vlan10/20/99     │  │ SVI: Vlan30/40/99     │  │ SVI: Vlan30/40/99     │
        └───────┬───────┬───────┘  └───────┬───────┬───────┘  └───────┬───────┬───────┘  └───────┬───────┬───────┘
           (Trunk, STP อยู่ตรงนี้เท่านั้น — ดู Step 323/325)
        ┌────────┴──┐ ┌──┴────────┐  ┌────────┴──┐ ┌──┴────────┐  ┌────────┴──┐ ┌──┴────────┐  ┌────────┴──┐ ┌──┴────────┐
        │ACCESS-SW1  │ │(สาย       │  │(สาย       │ │ACCESS-SW2 │  │ACCESS-SW3 │ │(สาย       │  │(สาย       │ │ACCESS-SW4 │
        │            │ │ไขว้ D1-D2)│  │ไขว้ D1-D2)│ │           │  │           │ │ไขว้ D3-D4)│  │ไขว้ D3-D4)│ │           │
        └────────────┘ └───────────┘  └───────────┘ └───────────┘  └───────────┘ └───────────┘  └───────────┘ └───────────┘
```

**สิ่งที่เปลี่ยนจาก Part 10/Part 5**: เพิ่มลิงก์ Routed ใหม่ 4 เส้น (DIST-SW1↔CORE-SW2,
DIST-SW2↔CORE-SW2, DIST-SW3↔CORE-SW1, DIST-SW4↔CORE-SW1) — ส่วน Access↔Distribution
(dual-homed ที่มี STP มาตั้งแต่ Part 5) และ Core↔Core (Po1 จาก Part 6) **ไม่เปลี่ยนแปลง**

### Subnet Plan ส่วนขยาย — ลิงก์ใหม่ที่ Part นี้เพิ่มเข้ามา

> เอกสารนี้ **ขยาย** (ไม่ใช่แก้ไข) ตาราง Backbone Link ใน
> [`00-ip-address-plan.md`](00-ip-address-plan.md) — รูปแบบการตั้งเลข subnet ใหม่ใช้หลักการ
> `10.255.X0.0/30` = ลิงก์เดิมของ DIST-SWX ไปหา Core ประจำตัว (Part 10), และ
> `10.255.X1.0/30` = ลิงก์ใหม่ของ DIST-SWX ไปหา Core อีกตัว (Part 33)

| Link | Subnet | ปลาย Core | ปลาย DIST | Interface | สถานะ |
|---|---|---|---|---|---|
| CORE-SW1 ↔ DIST-SW1 | 10.255.10.0/30 | .1 | .2 | Te1/0/1 ↔ Te1/1/1 | เดิม (Part 10) |
| **CORE-SW2 ↔ DIST-SW1** | **10.255.11.0/30** | **.1** | **.2** | **Te1/0/3 ↔ Te1/1/2** | **ใหม่ (Part 33)** |
| CORE-SW1 ↔ DIST-SW2 | 10.255.20.0/30 | .1 | .2 | Te1/0/2 ↔ Te1/1/1 | เดิม (Part 10) |
| **CORE-SW2 ↔ DIST-SW2** | **10.255.21.0/30** | **.1** | **.2** | **Te1/0/4 ↔ Te1/1/2** | **ใหม่ (Part 33)** |
| CORE-SW2 ↔ DIST-SW3 | 10.255.30.0/30 | .1 | .2 | Te1/0/1 ↔ Te1/1/1 | เดิม (Part 10) |
| **CORE-SW1 ↔ DIST-SW3** | **10.255.31.0/30** | **.1** | **.2** | **Te1/0/3 ↔ Te1/1/2** | **ใหม่ (Part 33)** |
| CORE-SW2 ↔ DIST-SW4 | 10.255.40.0/30 | .1 | .2 | Te1/0/2 ↔ Te1/1/1 | เดิม (Part 10) |
| **CORE-SW1 ↔ DIST-SW4** | **10.255.41.0/30** | **.1** | **.2** | **Te1/0/4 ↔ Te1/1/2** | **ใหม่ (Part 33)** |
| CORE-SW1 ↔ CORE-SW2 | 10.255.0.0/30 | .1 / .2 | — | Port-channel1 | เดิม (Part 6) |

ผลลัพธ์: CORE-SW1 ใช้พอร์ต **Te1/0/1–4** ครบ (2 เดิม + 2 ใหม่), CORE-SW2 ก็เช่นกัน และ
DIST-SW แต่ละตัวใช้ **Te1/1/1 (เดิม) + Te1/1/2 (ใหม่)**

---

## Step 322 — Routed Access vs Traditional Design

### สอง Model หลักของการออกแบบ Campus Access Layer

| หัวข้อ | Traditional (L2 Access + L3 Distribution) | Routed Access |
|---|---|---|
| Access switch ทำงานที่ Layer | 2 ล้วน (ไม่มี Routing) | 3 (มี SVI/Routed Port, route ได้ตั้งแต่ Access) |
| จุดที่ VLAN สิ้นสุด (L2/L3 boundary) | ที่ Distribution (SVI, ตามที่ Lab นี้ใช้ตั้งแต่ Part 10) | ที่ Access switch เอง — แทบไม่มี VLAN ข้าม Access-Distribution เลย |
| STP จำเป็นไหม | จำเป็น — ที่ขอบ Access↔Distribution (dual-homed L2 uplink) | **ไม่จำเป็นเลย** — ทุกลิงก์เป็น Routed (P2P /30 หรือ /31) |
| Convergence เมื่อลิงก์ล่ม | ขึ้นกับ RSTP (~1-3 วิ) | ขึ้นกับ Routing Protocol (OSPF ECMP, sub-second) |
| Hardware ที่ต้องใช้ | Access switch L2-only ราคาถูกได้ (Catalyst 9200/2960) | Access switch ต้องเป็น **Multilayer** เท่านั้น (แพงกว่า) |
| VLAN ที่ต้อง Span ข้าม Access หลายตัว (เช่น Voice VLAN, Guest) | ทำได้ง่าย (Trunk ปกติ) | ทำได้ยากกว่า — ต้องใช้ Overlay (VXLAN) แทน เพราะไม่มี L2 ข้าม Access แล้ว |
| ใช้ในสถาปัตยกรรมสมัยใหม่ใด | Enterprise Campus แบบ Classic (ที่หลักสูตรนี้ใช้) | **Cisco SD-Access** (Fabric, LISP, VXLAN) — ดู [Part 38](part-038-sd-access-fundamentals.md) |

### ทำไมหลักสูตรนี้ยังใช้ Traditional Design

1. **Hardware ไม่รองรับ** — ตาราง Device Platform ใน
   [`00-ip-address-plan.md`](00-ip-address-plan.md) กำหนดให้ **ACCESS-SW1–4 เป็น
   Catalyst 9200/2960 (Layer 2 ล้วน)** ตั้งแต่ Part 1 ซึ่ง **ไม่มีความสามารถสร้าง SVI ที่ route ได้
   เลยทางกายภาพ** — ต่อให้อยากทำ Routed Access ก็ทำไม่ได้จนกว่าจะเปลี่ยน Hardware
2. **สอดคล้องกับ Investment ที่ทำไปแล้วใน Part 10** — SVI + HSRP (Part 18) ถูกวางไว้ที่
   Distribution Layer อยู่แล้วทั้งหมด การเปลี่ยนไป Routed Access ตอนนี้จะทำให้ต้อง Config
   ใหม่หมดทั้ง VLAN/Gateway Design
3. **Routed Access เหมาะกับ Fabric-based Design มากกว่า** — การ Route ตั้งแต่ Access ทำให้ไม่มี
   L2 Boundary ข้าม Access switch เลย ซึ่งเป็นข้อกำหนดพื้นฐานของ **SD-Access Fabric** ที่ใช้
   VXLAN encapsulation คลุม L2 อีกที (Overlay) — เป็นสถาปัตยกรรมคนละแบบที่ต้องเรียนแยกใน
   Part 38 โดยเฉพาะ ไม่ใช่แค่เปลี่ยน config เดิม

> **สรุปสำหรับข้อสอบ CCNP ENCOR**: คำถามแนว "จะเลือก Traditional หรือ Routed Access" ต้องดูที่
> **Hardware ของ Access Layer** เป็นอันดับแรกเสมอ — ถ้า Access switch เป็น L2-only ต้องใช้
> Traditional เท่านั้น ไม่มีทางเลือกอื่น Routed Access คือทางเลือกที่ **เหมาะกับ Design ใหม่/
> SD-Access เท่านั้น** ไม่ใช่สิ่งที่ต้อง "อัปเกรด" ให้ทุก Enterprise Network เสมอไป

---

## Step 323 — L2 vs L3 Link ใน Topology ใหม่: ตัดสินใจออกแบบ

### หลักการเดิมจาก Part 10 (Step 95.2) — ยังใช้ได้และคือกุญแจของ Part นี้

Part 10 วางกฎไว้แล้วว่า **Backbone Link ระหว่าง Distribution↔Core และ Core↔Core ให้ใช้
Routed Port เสมอ (ไม่ใช่ SVI+Trunk)** — กฎนี้คือเหตุผลที่ทำให้การเพิ่ม Dual-Homing ใน Part นี้
**ไม่ต้องแตะ STP เลยแม้แต่บรรทัดเดียว** เพราะ Routed Port ไม่มี STP มาเกี่ยวข้องอยู่แล้วตั้งแต่ต้น

### ถ้าลิงก์ใหม่เป็น L2 Trunk แทน — จะเกิดอะไรขึ้น (Worked Comparison)

สมมติเราออกแบบผิด โดยทำให้ DIST-SW1↔CORE-SW1↔CORE-SW2↔DIST-SW1 (ผ่านลิงก์ใหม่)
เป็น **Trunk ทั้งหมด** (พก VLAN เดียวกันข้าม Core):

```
                    ┌────────────┐         ┌────────────┐
                    │  CORE-SW1  │◄══Trunk═►│  CORE-SW2  │   <- ถ้าเป็น Trunk: เกิด Loop ทันที
                    └──┬──────┬──┘         └──┬──────┬──┘
                Trunk  │      └────Trunk───────┘      │ Trunk
                       │                               │
                    ┌──┴─────┐                    (อีกด้านของ DIST-SW1)
                    │DIST-SW1│◄═══════════════════════╝
                    └────────┘
```

- เกิด **Physical Loop สี่เหลี่ยม** ทันที (CORE-SW1 → CORE-SW2 → DIST-SW1 → CORE-SW1)
- STP ต้อง **Block ลิงก์ 1 เส้นจาก 2 เส้น** ของ DIST-SW1 เพื่อตัด Loop (เหมือนที่ Part 5
  Step 43 ทำกับ DIST-SW1↔CORE-SW1/CORE-SW2 แบบเดิม)
- ผลคือ **ได้แค่ Failover (1:1 backup)** ไม่ใช่ ECMP — ลิงก์สำรองถูก Block เฉยๆ ไม่ได้ใช้งาน
  Bandwidth เลยจนกว่าลิงก์หลักจะล่ม — นี่คือสิ่งที่ Part 5/Part 10 เป็นอยู่แล้ว **Part นี้จึงไม่ได้
  แค่ "เพิ่มสายสำรอง" แต่เปลี่ยนสายสำรองที่เคย Block ให้กลายเป็นสายที่ Active และ Forward
  Traffic จริงพร้อมกันทั้ง 2 เส้น**

### เมื่อไหร่ยัง "จำเป็น" ต้องใช้ L2 Trunk ข้าม Core (ข้อยกเว้น)

มีบางกรณีที่ยังจำเป็นต้องมี VLAN เดียวกัน **Bridge ข้าม Core Layer จริงๆ** เช่น:

| กรณี | เหตุผลที่ต้องใช้ L2 | ทางเลือกใน Production จริง |
|---|---|---|
| Server Cluster ที่ต้องการ vMotion/Live Migration | Hypervisor ต้องการให้ VM ย้าย Host ข้าม Data Center โดย IP เดิม (ต้องอยู่ VLAN/Subnet เดียวกัน) | OTV, VXLAN EVPN, หรือ Data Center Interconnect (DCI) เฉพาะทาง — **ไม่ใช่ Trunk ธรรมดาข้าม Core** |
| VLAN 99 (MGMT) ที่ต้องการ Subnet เดียวจริงทั่วองค์กร | ตาม Part 10 Step 97 ที่ MGMT ถูกแบ่งเป็น 2 บล็อกทางกายภาพเพราะ Routed Core | แก้ด้วย Out-of-Band Management Network แยกต่างหาก (Part 17) ไม่ใช่ Trunk ข้าม Core |
| Legacy Application ที่ Hard-code Broadcast/Multicast ในวง L2 เดียว | Software เก่าไม่รองรับ Routing | ควรย้าย Application หรือใช้ L2VPN/Pseudowire เฉพาะจุด ไม่ใช่ Trunk ทั้ง Core |

> **กฎการออกแบบสำหรับหลักสูตรนี้ (ยืนยันจาก Part 10)**: **Core Layer ต้องเป็น Routed ล้วนเสมอ**
> ไม่มี VLAN ใดๆ ข้าม Core ยกเว้นกรณีพิเศษเฉพาะทาง (Data Center/DCI) ที่นอกเหนือขอบเขต
> Enterprise Campus ปกติ — ด้วยเหตุนี้ **ลิงก์ใหม่ทั้ง 4 เส้นใน Part นี้จึงเป็น Routed Port
> ทั้งหมด ไม่ใช่ Trunk**

---

## Step 324 — Redesigning for Full HA: Config เต็มรูปแบบ

### ภาพรวมสิ่งที่ต้อง Config เพิ่ม

1. **CORE-SW1**: เพิ่ม Te1/0/3 (→DIST-SW3), Te1/0/4 (→DIST-SW4) + OSPF network ใหม่
2. **CORE-SW2**: เพิ่ม Te1/0/3 (→DIST-SW1), Te1/0/4 (→DIST-SW2) + OSPF network ใหม่
3. **DIST-SW1–4**: เพิ่ม Te1/1/2 (→ Core อีกตัวที่ยังไม่มีสาย) + OSPF network ใหม่
4. **OSPF**: เปลี่ยน `auto-cost reference-bandwidth` ให้เหมาะกับยุค 10G/100G (ดูเหตุผลใน Step 327)
   และยืนยัน `maximum-paths` ให้พร้อมรับ ECMP

### Config เพิ่มเติมบน CORE-SW1

```
CORE-SW1(config)# interface TenGigabitEthernet1/0/3
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW3 (NEW - Part 33 HA) **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.31.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/4
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW4 (NEW - Part 33 HA) **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.41.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! ใส่ interface ใหม่เข้า OSPF area 0 (backbone เดียวกับทุกลิงก์ตั้งแต่ Part 11)
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# network 10.255.31.0 0.0.0.3 area 0
CORE-SW1(config-router)# network 10.255.41.0 0.0.0.3 area 0

! ปรับ Reference Bandwidth ให้เหมาะกับลิงก์ 10G/100G ยุคปัจจุบัน (ดู Step 327)
CORE-SW1(config-router)# auto-cost reference-bandwidth 100000
% OSPF: Reference bandwidth is changed.
        Please ensure reference bandwidth is consistent across all routers.
CORE-SW1(config-router)# maximum-paths 4
CORE-SW1(config-router)# end
CORE-SW1# write memory
```

### Config เพิ่มเติมบน CORE-SW2 (Pattern เดียวกัน สลับปลายทางเป็น DIST-SW1/DIST-SW2)

```
CORE-SW2(config)# interface TenGigabitEthernet1/0/3
CORE-SW2(config-if)# description ** Routed downlink to DIST-SW1 (NEW - Part 33 HA) **
CORE-SW2(config-if)# no switchport
CORE-SW2(config-if)# ip address 10.255.11.1 255.255.255.252
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/4
CORE-SW2(config-if)# description ** Routed downlink to DIST-SW2 (NEW - Part 33 HA) **
CORE-SW2(config-if)# no switchport
CORE-SW2(config-if)# ip address 10.255.21.1 255.255.255.252
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 10.255.11.0 0.0.0.3 area 0
CORE-SW2(config-router)# network 10.255.21.0 0.0.0.3 area 0
CORE-SW2(config-router)# auto-cost reference-bandwidth 100000
CORE-SW2(config-router)# maximum-paths 4
CORE-SW2(config-router)# end
CORE-SW2# write memory
```

> **คำเตือนสำคัญ**: `auto-cost reference-bandwidth` ทำงานคล้ายกับ STP Timer ใน Part 5 — **ต้อง
> ตั้งค่าเดียวกันทุกตัวใน OSPF Domain เดียวกันเสมอ** (CORE-SW1, CORE-SW2, DIST-SW1–4 ทุกตัว)
> ไม่งั้นแต่ละ Router จะคำนวณ Cost คนละมาตรฐานกัน ทำให้ Routing Table ไม่ตรงกันและอาจเกิด
> Routing Loop หรือเลือกเส้นทางที่ไม่เหมาะสมได้

### Config เต็มบน DIST-SW1 (เพิ่มลิงก์ใหม่ไปหา CORE-SW2)

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/2
DIST-SW1(config-if)# description ** Routed uplink to CORE-SW2 (NEW - Part 33 HA) **
DIST-SW1(config-if)# no switchport
DIST-SW1(config-if)# ip address 10.255.11.2 255.255.255.252
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# network 10.255.11.0 0.0.0.3 area 0
DIST-SW1(config-router)# auto-cost reference-bandwidth 100000
DIST-SW1(config-router)# maximum-paths 4
DIST-SW1(config-router)# end
DIST-SW1# write memory
```

> **สำคัญ**: เราไม่ต้องลบ static default route (`ip route 0.0.0.0 0.0.0.0 10.255.10.1`) จาก Part 10
> ออกด้วยมือ เพราะตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md) มันถูกแทนที่ด้วย OSPF ไปแล้ว
> ถ้ายังเหลือ static route ค้างอยู่ ให้ลบด้วย `no ip route 0.0.0.0 0.0.0.0 10.255.10.1` ก่อน

### Config บน DIST-SW2, DIST-SW3, DIST-SW4 (Pattern เดียวกัน — เปลี่ยนแค่ IP/Interface)

| Device | Interface ใหม่ | IP ใหม่ | ปลายทาง | คำสั่ง OSPF network เพิ่ม |
|---|---|---|---|---|
| DIST-SW2 | Te1/1/2 | 10.255.21.2/30 | CORE-SW2 | `network 10.255.21.0 0.0.0.3 area 0` |
| DIST-SW3 | Te1/1/2 | 10.255.31.2/30 | CORE-SW1 | `network 10.255.31.0 0.0.0.3 area 0` |
| DIST-SW4 | Te1/1/2 | 10.255.41.2/30 | CORE-SW1 | `network 10.255.41.0 0.0.0.3 area 0` |

ทั้ง 3 ตัวต้องเพิ่ม `auto-cost reference-bandwidth 100000` และ `maximum-paths 4` ใต้
`router ospf 1` เหมือน DIST-SW1 ด้วยเช่นกัน (ค่านี้ต้องตรงกันทุกอุปกรณ์ในเครือข่าย)

### จาก "L2 Redundancy Problem" สู่ "L3 ECMP/Failover Problem"

นี่คือหัวใจของ Part นี้: เพราะลิงก์ทั้งหมดเป็น **Routed (L3)** ไม่ใช่ Trunk (L2) การเพิ่ม
Redundancy จึงไม่ต้องผ่านกลไก STP (Block ลิงก์สำรองทิ้งเฉยๆ) เลยแม้แต่น้อย แต่ใช้กลไกของ
**OSPF ECMP** แทน:

```
แบบเดิม (L2 + STP)                          แบบใหม่ (L3 + OSPF ECMP)
┌──────────┐                                ┌──────────┐
│ DIST-SW1 │──Active───► CORE-SW1           │ DIST-SW1 │──Active (cost 10)──► CORE-SW1
│          │──Blocked──► CORE-SW2           │          │──Active (cost 10)──► CORE-SW2
└──────────┘  (STP Block, ใช้งานไม่ได้      └──────────┘  (ทั้ง 2 เส้น Forward พร้อมกัน
              จนกว่าเส้นหลักจะล่ม)                          จริง — Bandwidth รวม 2x)
```

**OSPF ไม่ต้อง "เลือก" เส้นทางเดียวแล้ว Block อีกเส้นทิ้งแบบ STP** — ถ้า Cost เท่ากันทั้ง 2 เส้น
(ซึ่งออกแบบให้เท่ากันโดยตั้งใจ ดู Step 327) OSPF จะติดตั้งทั้งคู่ลงใน Routing Table พร้อมกัน
(**Equal-Cost Multi-Path**) และ Forward Traffic ผ่านทั้งสองเส้นจริงตลอดเวลา — ไม่มีแนวคิด
"ลิงก์สำรองที่ถูกกันไว้เฉยๆ" อีกต่อไป ทุกเส้นถูกใช้งานเต็มประสิทธิภาพ และเมื่อเส้นใดเส้นหนึ่งล่ม
OSPF แค่ **ถอด Path นั้นออกจาก ECMP Set** (ซึ่งอีก Path ก็ยังอยู่ใน FIB ของ Hardware อยู่แล้ว
ตั้งแต่ต้น) ทำให้ Failover เร็วกว่า STP Convergence มาก (ดู Step 330 การทดสอบจริง)

---

## Step 325 — STP ยังจำเป็นตรงไหนบ้างใน Campus ใหม่

### ทบทวน: STP ที่ Distribution↔Core หายไปตั้งแต่ Part 10 แล้ว ไม่ใช่เพิ่งหายใน Part นี้

ต้องแยกให้ชัดว่า **Routed Port ไม่มี STP ทำงานอยู่เลย** (ไม่มี Port Role, ไม่มี BPDU,
ไม่มี Blocking) มาตั้งแต่ [Part 10 Step 95.2](part-010-inter-vlan-routing.md) ที่เปลี่ยน
Distribution↔Core จาก Trunk เป็น Routed Port แล้ว — Part 5 (STP) เขียนขึ้น**ก่อน**
Part 10 จึงยัง Config `spanning-tree guard loop` ไว้บน `GigabitEthernet1/0/1-2` (อธิบายว่า
"UPLINK TO CORE-SW1/CORE-SW2" แบบ Trunk) ซึ่ง **กลายเป็นค่าที่ไม่มีผลอะไรอีกต่อไปตั้งแต่
Part 10** เพราะพอร์ตอัปลิงก์จริงถูกย้ายไปเป็น `TenGigabitEthernet1/1/1` (Routed, `no switchport`)
แทน — Config เก่าของ Loop Guard บน Gi1/0/1-2 ไม่ได้ทำอะไรเสียหาย (เป็นแค่ Config ค้างบน
Interface ที่อาจไม่ได้ใช้งานแล้ว) แต่ก็ **ไม่มีความหมายอีกต่อไป** ควรลบทิ้งเพื่อความสะอาดของ
Config หากยังเหลืออยู่จริงใน Lab ของท่าน

### ที่เดียวที่ STP ยังทำงานจริงในสถาปัตยกรรมนี้: ขอบ Access ↔ Distribution

```
                CORE-SW1 ◄════ Po1 ════► CORE-SW2      <- Routed ล้วน, ไม่มี STP (Part 10)
                   │  │                    │  │
                   │  └── Routed ──────────┘  │         <- Routed ล้วน, ไม่มี STP (Part 10 + Part 33)
              Routed│                          │Routed
                ┌───┴────┐                ┌────┴───┐
                │DIST-SW1│                │DIST-SW2│
                └─┬────┬─┘                └─┬────┬─┘
                Trunk│  │Trunk            Trunk│  │Trunk
                     │  └───────┐    ┌─────────┘  │      <- Trunk ไขว้กัน = มี Loop จริง
                     │      ┌───┴────┴───┐         │      <- STP ยังต้องทำงานตรงนี้เท่านั้น!
                ┌────┴──┐  │(ไขว้ D1-D2)│    ┌────┴──┐
                │ACCESS-1│  └────────────┘    │ACCESS-2│
                └────────┘                    └────────┘
```

**สรุป**: หลังจาก Part นี้ ขอบเขตที่ STP (RSTP, Rapid-PVST+) ยังต้องทำงานจริง **แคบลงเหลือแค่
Access↔Distribution เท่านั้น** — Core↔Core และ Distribution↔Core ทั้งหมดเป็น Pure L3 ไม่มี
STP เกี่ยวข้องอีกต่อไป

### แก้ไขจุดที่ผิดพลาดจาก Part 5: Root Bridge ควรอยู่ที่ไหนกันแน่?

นี่คือจุดที่ต้อง **ทบทวนอย่างละเอียด**: Part 5 Step 44 config ให้ **CORE-SW1 เป็น Root Primary
และ CORE-SW2 เป็น Root Secondary สำหรับทุก VLAN (10,20,30,40,99)** — แต่คำถามคือ **คำสั่งนี้
มีผลจริงหรือไม่** ในเมื่อตั้งแต่ Part 10 เป็นต้นมา **CORE-SW1/CORE-SW2 ไม่มีพอร์ตใดเป็น
Switchport ของ VLAN เหล่านี้เลยสักพอร์ตเดียว** (ทุกพอร์ตเป็น Routed Port หรือ Routed
Port-channel ทั้งหมด)

**คำตอบ**: `spanning-tree vlan 10 root primary` บน CORE-SW1 ยัง**ทำงาน** ในแง่ที่ IOS จะตั้ง
Priority ของ Bridge ID ให้ต่ำ (4096) จริง แต่ค่านี้ **ไม่มีความหมายในทางปฏิบัติอีกต่อไป** เพราะ
CORE-SW1 **ไม่ได้เป็นส่วนหนึ่งของ STP Topology ของ VLAN 10/20/30/40/99 เลย** (ไม่มีลิงก์ไหน
ที่เป็น Trunk พก VLAN เหล่านี้อยู่ที่ CORE-SW1) — ผลคือ **Root Bridge ตัวจริงของแต่ละ VLAN
กลายเป็นค่าที่เลือกโดย DIST-SW และ ACCESS-SW เท่านั้น** โดยที่ไม่มีใครเคย Config Priority
ไว้ให้ชัดเจนในกลุ่มนี้เลย (ปล่อยให้เป็น Default 32768 ทั้งหมด) — เสี่ยงต่อปัญหา **"Unintended
Root Bridge"** ที่ Part 5 Step 49 เคยเตือนไว้ (switch ที่ MAC ต่ำสุดในกลุ่มจะกลายเป็น Root
โดยไม่ตั้งใจ)

### การแก้ไข: ย้าย Root Primary/Secondary มาไว้ที่ Distribution Layer

```
! บล็อก 1: DIST-SW1 (Root Primary) + DIST-SW2 (Root Secondary) สำหรับ VLAN10/20/99
DIST-SW1(config)# spanning-tree vlan 10,20,99 root primary
DIST-SW2(config)# spanning-tree vlan 10,20,99 root secondary

! บล็อก 2: DIST-SW3 (Root Primary) + DIST-SW4 (Root Secondary) สำหรับ VLAN30/40/99
DIST-SW3(config)# spanning-tree vlan 30,40,99 root primary
DIST-SW4(config)# spanning-tree vlan 30,40,99 root secondary
```

> **หมายเหตุ VLAN 99**: ตาม [Part 10 Step 97](part-010-inter-vlan-routing.md) VLAN 99 (MGMT)
> ถูกแบ่งเป็น **2 L2 Domain แยกกันทางกายภาพ** (บล็อก DIST-SW1/2 ใช้ 10.10.99.0/25 ส่วนหนึ่ง,
> บล็อก DIST-SW3/4 อีกส่วนหนึ่ง) ดังนั้นการตั้ง Root แยกกันคนละบล็อกจึงถูกต้องและไม่ขัดแย้งกัน
> เพราะเป็นคนละ Spanning-Tree Instance กันโดยสมบูรณ์อยู่แล้ว (แม้จะชื่อ VLAN 99 เหมือนกัน)

ส่วน Config เดิมของ CORE-SW1/CORE-SW2 (`spanning-tree vlan ... root primary/secondary`)
ควรถูก **ลบทิ้ง** (`no spanning-tree vlan 10,20,30,40,99 root primary`) เพื่อไม่ให้เกิดความสับสน
ในอนาคตว่า Core ยังเกี่ยวข้องกับ STP ของ VLAN เหล่านี้อยู่ — ส่วน Feature อื่นจาก Part 5 ที่ยัง
**ใช้งานได้ปกติทุกประการ ไม่ต้องแก้ไข**:

| Feature จาก Part 5 | ตำแหน่งเดิม | ยังถูกต้องหรือไม่ |
|---|---|---|
| PortFast + BPDU Guard | พอร์ต user-facing บน ACCESS-SW1–4 | ✅ ถูกต้อง ไม่เปลี่ยน |
| Root Guard | พอร์ตของ DIST ที่หันไปทาง ACCESS | ✅ ถูกต้อง ไม่เปลี่ยน |
| Loop Guard บน Trunk Access↔Distribution | พอร์ตของ DIST/ACCESS ที่เป็น Trunk คู่กัน | ✅ ถูกต้อง ไม่เปลี่ยน (คนละพอร์ตกับที่เคย config ผิดไปหา Core) |
| `spanning-tree mode rapid-pvst` | ทุก switch | ✅ ถูกต้อง ไม่เปลี่ยน |
| Root Primary/Secondary ที่ CORE-SW1/CORE-SW2 | Global บน Core | ❌ **ต้องย้ายมาที่ Distribution ตามที่แก้ไขข้างต้น** |

---

## Step 326 — Multilayer Switching Internals: CEF, FIB, Adjacency Table, TCAM

### ทำไม Multilayer Switch ถึง Route ได้เร็วเท่า Wire-Speed

Part 10 บอกไว้แล้วว่า SVI/Routed Port บน Multilayer Switch "Route ในตัว ASIC เดียวกับที่ทำ
Switching" — กลไกเบื้องหลังคำพูดนี้คือ **Cisco Express Forwarding (CEF)** ซึ่งเป็นวิธี
Forward Packet ที่ Cisco IOS/IOS-XE ใช้เป็น Default บนอุปกรณ์ทุกรุ่นตั้งแต่ปลายยุค 1990 เป็นต้นมา

### วิวัฒนาการของวิธี Forward Packet บน Cisco IOS (ทบทวนประวัติศาสตร์)

| วิธี | ยุค | หลักการ | ความเร็ว |
|---|---|---|---|
| **Process Switching** | เก่าสุด | CPU ประมวลผล **ทุก Packet** ผ่าน Routing Table แบบเต็มรูปแบบทีละ Packet (interrupt CPU ทุกครั้ง) | ช้าที่สุด (Software ล้วน, ใช้เมื่อ debug เท่านั้นในปัจจุบัน) |
| **Fast Switching** | กลาง | Cache ผลลัพธ์ของ Packet **แรก** ที่ Process Switch แล้วใช้ Cache สำหรับ Packet ถัดไปที่ปลายทางเดียวกัน | เร็วขึ้น แต่ Cache ยังต้องสร้างจาก Process Switch อย่างน้อย 1 ครั้งก่อนเสมอ และ Cache ล้างเมื่อ Topology เปลี่ยน |
| **CEF (Cisco Express Forwarding)** | ปัจจุบัน (Default) | **คำนวณตารางล่วงหน้าทั้งหมดก่อน Packet มาถึงเลย** (Pre-computed) ไม่ต้องรอ Packet แรกมากระตุ้น Cache | **เร็วที่สุด** — ทำใน Hardware (ASIC/TCAM) ได้เต็มรูปแบบ ไม่พึ่ง CPU เลยในสภาวะปกติ |

> **สำหรับ CCNP/CCIE**: Fast Switching และ Process Switching ยังมีอยู่ใน IOS เพื่อ Backward
> Compatibility และ Debug เท่านั้น (`no ip route-cache cef` เพื่อปิด CEF ทดสอบ) — **Production
> Network ทุกที่ต้องเปิด CEF เสมอ** ซึ่งเป็นค่า Default อยู่แล้วบน Catalyst และ ISR/ASR ทุกรุ่น
> ปัจจุบัน ไม่มีเหตุผลทาง Business ใดที่ต้องปิด CEF

### สถาปัตยกรรม CEF: FIB + Adjacency Table

CEF แยกข้อมูลเป็น **2 ตารางที่ทำงานร่วมกัน**:

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│   FIB (Forwarding            │        │   Adjacency Table              │
│   Information Base)          │───────►│   (Layer 2 Rewrite Info)       │
│                               │        │                                │
│  "ปลายทาง Network ไหน        │        │  "เมื่อรู้ Next-hop แล้ว       │
│   ควรออกไปทาง Interface     │        │   ต้องเปลี่ยน Header เป็น       │
│   ไหน และ Next-hop คือใคร"   │        │   MAC อะไร (Layer 2 Rewrite)"  │
│                               │        │                                │
│  คัดลอกมาจาก Routing Table  │        │  คัดลอกมาจาก ARP Table/        │
│  (RIB) แต่ optimize สำหรับ   │        │  CDP/Neighbor Table            │
│  การ Lookup เร็วสุด (Trie)   │        │                                │
└─────────────────────────────┘        └──────────────────────────────┘
```

- **FIB**: มาจาก Routing Table (RIB) แต่จัดโครงสร้างใหม่ให้ Lookup เร็วที่สุด (ใช้โครงสร้างข้อมูล
  แบบ M-Trie) — ทุกครั้งที่ Routing Table เปลี่ยน (OSPF SPF คำนวณใหม่) FIB จะถูก **Sync ทันที**
- **Adjacency Table**: เก็บ Layer 2 Header ที่พร้อมใช้ (Pre-built) สำหรับแต่ละ Next-hop —
  เมื่อ Packet มาถึง CEF แค่ **"แปะ" Header ที่เตรียมไว้แล้วลงไปเลย** ไม่ต้อง ARP หรือคำนวณใหม่
  ทุกครั้ง (Adjacency ประเภท "Glean" หมายถึงยังไม่มี MAC จริง ต้อง ARP ก่อน)

### TCAM (Ternary Content-Addressable Memory) — เหตุผลที่ Catalyst Switch เร็วกว่า Router ทั่วไป

Catalyst Multilayer Switch (เช่น CORE-SW1/CORE-SW2, DIST-SW1–4 ในหลักสูตรนี้) มี Hardware
พิเศษเรียกว่า **TCAM** ที่ทำ Lookup ได้ **ในรอบ Clock เดียว (Single Clock Cycle)** ไม่ว่า FIB
จะมีกี่หมื่น Entry ก็ตาม (ต่างจาก Software Lookup แบบ Router ทั่วไปที่ต้องไล่ค้นทีละขั้น) —
นี่คือเหตุผลที่ Catalyst Multilayer Switch Route ได้ที่ Wire-speed เต็ม Backplane โดยไม่มี
Bottleneck แบบ Router-on-a-Stick (Part 10 Step 93)

```
Packet เข้า Interface
        │
        ▼
┌───────────────┐     TCAM Lookup      ┌────────────────┐
│  ASIC ของ      │ ────(1 clock cycle)──►│  ผลลัพธ์:       │
│  Catalyst      │                       │  Out-interface, │
│  Switch        │                       │  Next-hop MAC,  │
└───────────────┘                       │  QoS marking ฯลฯ │
                                          └────────────────┘
                                          ทั้งหมดนี้เกิดใน Hardware
                                          ไม่ผ่าน CPU เลย (ยกเว้น
                                          Packet ที่ CEF "Punt" ขึ้นมา
                                          ให้ CPU จัดการ เช่น TTL=1,
                                          Option Packet, ARP Request)
```

### คำสั่งตรวจสอบ CEF (ใช้จริงใน Step 329/330)

```
DIST-SW1# show ip cef
Prefix              Next Hop         Interface
0.0.0.0/0            10.255.10.1      TenGigabitEthernet1/1/1
                      10.255.11.1      TenGigabitEthernet1/1/2      <- ECMP: 2 Next-hop
10.10.30.0/24        10.255.10.1      TenGigabitEthernet1/1/1
                      10.255.11.1      TenGigabitEthernet1/1/2

DIST-SW1# show adjacency detail
Protocol Interface                 Address
IP       TenGigabitEthernet1/1/1   10.255.10.1(7)
                                    0019AA112233
                                    ARP        03:41:17
IP       TenGigabitEthernet1/1/2   10.255.11.1(9)
                                    0019AA334455
                                    ARP        03:12:05
```

ดูรายละเอียดเต็มของ ECMP Entry เหล่านี้ต่อใน Step 327 และ 329

---

## Step 327 — Equal-Cost Multi-Path (ECMP) ที่ Core/Distribution Layer

### `maximum-paths` — จำกัดจำนวน Path สูงสุดที่ OSPF จะติดตั้งพร้อมกัน

```
DIST-SW1(config-router)# maximum-paths ?
  <1-32>  Number of paths
DIST-SW1(config-router)# maximum-paths 4
```

- ค่า Default บน Cisco IOS Classic คือ **4** ส่วน IOS-XE บน Catalyst 9000 บางรุ่น Default
  อาจสูงถึง **8** — **ควร Config ให้ชัดเจนเสมอ** ไม่พึ่ง Default เพื่อไม่ให้พฤติกรรมเปลี่ยนไปเมื่อ
  ย้าย Platform (บทเรียนเดียวกับที่ Part 5 สอนเรื่อง `spanning-tree mode` — ใส่ให้ชัดเจนเสมอ)
- Topology ของ Part นี้ **ต้องการแค่ 2 Path** (DIST↔Core ทั้ง 2 เส้น) แต่ตั้งไว้ที่ 4 เพื่อรองรับ
  การขยายในอนาคต (เช่น Part 34 ที่จะเพิ่ม StackWise/VSS)

### ทำไมต้อง `auto-cost reference-bandwidth 100000`

OSPF คำนวณ Cost จากสูตร `Cost = Reference Bandwidth ÷ Interface Bandwidth` — ค่า Default
ของ Reference Bandwidth คือ **100 Mbps** (ค่าที่ตั้งมาตั้งแต่ยุค 1990) ซึ่งทำให้เกิดปัญหาที่พบบ่อย
มากในเครือข่ายสมัยใหม่ **ที่ยังไม่ปรับค่านี้**:

```
! ด้วย Reference Bandwidth default (100 Mbps) — ปัญหาที่พบจริง
GigabitEthernet (1 Gbps)   → Cost = 100/1000  = 0.1 → ปัดขึ้นเป็น 1
TenGigabitEthernet (10 Gbps) → Cost = 100/10000 = 0.01 → ปัดขึ้นเป็น 1
                                                            ▲
                                    ทั้งคู่ได้ Cost เท่ากัน! ทั้งที่ความเร็วต่างกัน 10 เท่า
```

การปรับ `auto-cost reference-bandwidth 100000` (100,000 Mbps = 100 Gbps) ทำให้ Cost
สะท้อนความเร็วจริงอีกครั้ง:

```
GigabitEthernet (1 Gbps)     → Cost = 100000/1000  = 100
TenGigabitEthernet (10 Gbps) → Cost = 100000/10000 = 10
                                                        ▲
                              ตอนนี้ Cost ต่างกันจริงตามความเร็ว — OSPF จะเลี่ยงลิงก์ช้ากว่าเสมอ
```

### Worked Example: DIST-SW1 มี 2 Equal-Cost Path หลัง Redesign ใน Step 324

หลังปรับ Reference Bandwidth แล้ว ทุกลิงก์ Backbone ในหลักสูตรนี้เป็น **TenGigabitEthernet
ทั้งหมด** (Cost = 10 ต่อเส้นเท่ากันหมด) ทำให้เกิด ECMP โดยธรรมชาติจาก Full-Mesh Topology
ใน Step 321:

```
DIST-SW1 ต้องการไปหา 10.10.30.0/24 (VLAN30, อยู่หลัง DIST-SW3)

เส้นทาง A: DIST-SW1 --(Te1/1/1, cost10)--> CORE-SW1 --(Te1/0/3, cost10)--> DIST-SW3
           รวม Cost = 10 + 10 = 20

เส้นทาง B: DIST-SW1 --(Te1/1/2, cost10)--> CORE-SW2 --(Te1/0/1, cost10)--> DIST-SW3
           รวม Cost = 10 + 10 = 20

Cost เท่ากันทั้ง 2 เส้นทาง (20 = 20) → OSPF ติดตั้งทั้งคู่เป็น ECMP
```

```
DIST-SW1# show ip route ospf
     10.0.0.0/24 is subnetted, ...
O       10.10.30.0/24 [110/20] via 10.255.11.1, 00:02:14, TenGigabitEthernet1/1/2
                       [110/20] via 10.255.10.1, 00:02:14, TenGigabitEthernet1/1/1
```

สังเกตว่าไม่มีเส้นทางใดต้องข้าม Po1 (CORE-SW1↔CORE-SW2) เลย — เพราะ Full-Mesh Topology
ทำให้ DIST-SW1 เห็น CORE-SW2 ได้โดยตรง (2 Hop เท่ากันทั้งคู่) ลิงก์ Po1 กลายเป็น **Backup
เพิ่มเติม** สำหรับกรณีที่ลิงก์ DIST↔Core บางเส้นล่มเท่านั้น

### Per-Packet vs Per-Destination Load Balancing

| วิธี | หลักการ | ข้อดี/ข้อเสีย | ใช้เมื่อไหร่ |
|---|---|---|---|
| **Per-Destination (Default)** | Hash จาก Source IP + Destination IP (บางรุ่น + L4 Port) แล้วคง Flow เดียวกันวิ่งทาง Path เดิมเสมอ | Packet ไม่มีวันมาถึงปลายทางผิดลำดับ (No Reordering) — แต่ Load อาจไม่สมดุล 100% ถ้ามี Flow น้อย | **ใช้เป็น Default เสมอในหลักสูตรนี้** เหมาะกับ Production ทุกกรณี |
| **Per-Packet** | สลับ Path ทีละ Packet แบบ Round-Robin ไม่สนใจ Flow | Load สมดุลเป๊ะ 100% แต่ **เสี่ยง Packet Reordering สูง** (TCP อาจตีความผิดว่า Packet หาย แล้ว Retransmit ไม่จำเป็น) | ใช้เฉพาะกรณีพิเศษบน Router (ISR/ASR) ด้วยคำสั่ง `ip load-sharing per-packet` เท่านั้น — **Catalyst Multilayer Switch ในหลักสูตรนี้ไม่รองรับ/ไม่แนะนำ** |

```
! ตรวจสอบว่า Flow คู่ใดถูก Hash ไปทาง Path ไหน (Deterministic ตาม 5-tuple)
DIST-SW1# show ip cef exact-route 10.10.10.10 10.10.30.10
10.10.10.10 -> 10.10.30.10 : TenGigabitEthernet1/1/1 (Next Hop 10.255.10.1)

DIST-SW1# show ip cef exact-route 10.10.10.20 10.10.30.10
10.10.10.20 -> 10.10.30.10 : TenGigabitEthernet1/1/2 (Next Hop 10.255.11.1)
```

สังเกตว่า Source IP ต่างกัน (`.10` vs `.20`) ทำให้ Hash ผลลัพธ์ต่างกัน จึงกระจายไปคนละ
Interface — แต่ Flow เดิมซ้ำ (Source/Destination คู่เดิม) จะได้ Path เดิมเสมอ (Consistency)

---

## Step 328 — UDLD (UniDirectional Link Detection)

### ปัญหาที่ UDLD แก้: Fiber ขาดทางเดียว (Unidirectional Link Failure)

Part 5 Step 47 เคยพูดถึง **Loop Guard** ที่แก้ปัญหา Unidirectional Failure ได้ในระดับ STP
(สำหรับ Trunk/Switchport) — แต่ตอนนี้ลิงก์ DIST↔Core และ Core↔Core เป็น **Routed Port**
ที่ไม่มี STP ทำงานอยู่เลย (Step 325) ทำให้ **Loop Guard ใช้ไม่ได้กับลิงก์เหล่านี้อีกต่อไป**

คำถามคือ ถ้า Fiber เส้นหนึ่งเสีย **เฉพาะทิศทางเดียว** (เช่น TX ของฝั่ง A ใช้งานได้ปกติ แต่ RX
ของฝั่ง A เสีย — ฝั่ง A จึงส่งได้แต่รับไม่ได้) จะเกิดอะไรขึ้นกับ OSPF:

```
   DIST-SW1 (RX เสีย)                    CORE-SW1
        │ ◄───────X (รับไม่ได้)──────────  │  (ยังส่ง Hello ออกปกติ)
        │ ────────────(ส่งได้ปกติ)───────► │  (ยังได้รับ Hello จาก DIST-SW1)
```

- **CORE-SW1** ยังคงได้รับ OSPF Hello จาก DIST-SW1 ตามปกติ → คิดว่า Neighbor ยัง **Up**
- **DIST-SW1** ไม่ได้รับ Hello จาก CORE-SW1 เลย → หลังจาก Dead Timer (40 วินาที Default)
  จะประกาศ Neighbor **Down** และถอด Route ออก
- ระหว่างที่ CORE-SW1 ยังคิดว่า Neighbor Up (สูงสุดจนกว่า Dead Timer ฝั่งมันเองจะหมดเช่นกัน)
  Traffic ที่ CORE-SW1 พยายามส่งมาทาง Interface นี้จะ **หายไปเงียบๆ (Blackhole)** เพราะ
  DIST-SW1 รับไม่ได้อยู่แล้ว — ปัญหานี้ **Loop Guard ก็ช่วยไม่ได้เพราะไม่ใช่ STP** และ **OSPF
  เองก็ตรวจจับช้า** (ต้องรอ Dead Timer เต็ม ~40 วินาที)

### UDLD แก้ปัญหานี้ได้เร็วกว่ามาก โดยไม่พึ่ง Layer 3 เลย

**UDLD (UniDirectional Link Detection)** ทำงานที่ **Layer 2** โดยส่ง Echo Packet ที่มี
**Device ID + Port ID ของตัวเอง** ออกไปเป็นระยะ แล้วรอฟัง Echo กลับที่มี Device ID/Port ID
ของตัวเองสะท้อนกลับมา (คล้าย Loopback Test) — ถ้าไม่ได้ยิน Echo สะท้อนกลับตามที่คาด
(หรือได้ยินแต่ข้อมูลไม่ตรง) แปลว่าลิงก์มีปัญหาแบบ Unidirectional และ UDLD จะสั่ง
**Err-Disable พอร์ตทันที** (ไม่ต้องรอ Routing Protocol Timer เลย)

### Normal Mode vs Aggressive Mode

| Mode | พฤติกรรม | ความเร็วตรวจจับ |
|---|---|---|
| **Normal** | ถ้าพอร์ตเคย Sync กันได้แล้วจู่ๆ หยุดได้ยิน UDLD message จะแค่ทำเครื่องหมายเป็น "Undetermined" ไม่ Err-Disable ทันที | ช้ากว่า — เผื่อกรณี Message สูญหายชั่วคราว |
| **Aggressive** | ถ้าพอร์ตเคย Sync กันได้แล้วขาดการติดต่อ (ไม่ว่าจะ Unidirectional หรือ Bidirectional) จะพยายามส่งซ้ำ **8 ครั้งติดกัน** (ทุก 1 วินาที) ก่อนตัดสินใจ Err-Disable | **เร็วและแม่นยำกว่า — แนะนำสำหรับ Fiber Link ทุกเส้นในหลักสูตรนี้** |

### Config UDLD (แบบ Global สำหรับ Fiber ทั้งหมด)

```
! Global: เปิด UDLD Aggressive ให้ทุกพอร์ต Fiber อัตโนมัติ (Copper ต้องเปิดรายพอร์ตเพิ่ม)
CORE-SW1(config)# udld aggressive

! หรือเปิดรายพอร์ต ถ้าต้องการควบคุมเฉพาะจุด (แนะนำสำหรับ Backbone Link สำคัญ)
CORE-SW1(config)# interface range TenGigabitEthernet1/0/1 - 4
CORE-SW1(config-if-range)# udld port aggressive
CORE-SW1(config-if-range)# exit

! ทำแบบเดียวกันบน Port-channel1 (คลุมทั้ง Physical member ด้านใน)
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# udld port aggressive
```

> **ต้อง Config ทั้ง 2 ฝั่งเสมอ** (CORE-SW1 **และ** DIST-SW1 บนลิงก์เดียวกัน) เพราะ UDLD
> เป็น Protocol ที่ต้องเจรจา (Neighbor-based) เหมือน STP/OSPF — ถ้าเปิดฝั่งเดียว UDLD จะไม่มี
> Neighbor ให้ Sync ด้วยเลย

### เหตุการณ์จริงเมื่อ UDLD ตรวจพบปัญหา

```
*Mar  1 14:22:03.112: %UDLD-4-UDLD_PORT_DISABLED: UDLD disabled interface
Te1/1/1, unidirectional link detected
*Mar  1 14:22:03.118: %PM-4-ERR_DISABLE: udld error detected on Te1/1/1,
putting Te1/1/1 in err-disable state

DIST-SW1# show interfaces TenGigabitEthernet1/1/1 status
Port      Name               Status       Vlan       Duplex  Speed Type
Te1/1/1                      err-disabled routed     full    10G   10GBASE-SR
```

เช่นเดียวกับ BPDU Guard (Part 5 Step 46) ควรเปิด Auto-Recovery ไว้เสมอ:

```
DIST-SW1(config)# errdisable recovery cause udld
DIST-SW1(config)# errdisable recovery interval 300
```

### ตรวจสอบสถานะ UDLD

```
DIST-SW1# show udld TenGigabitEthernet1/1/1

Interface Te1/1/1
---
Port enable administrative configuration setting: Enabled / in aggressive mode
Port enable operational state: Enabled / in aggressive mode
Current bidirectional state: Bidirectional
Current operational state: Advertisement - Single Neighbor detected
Message interval: 15
Time out interval: 5

    Entry 1
    ---
    Expiration time: 21
    Device ID: 1
    Current neighbor state: Bidirectional
    Device name: FOC1234A5BC(CORE-SW1)
    Port ID: Te1/0/1
```

`Current bidirectional state: Bidirectional` คือสถานะปกติ — ถ้าเป็น `Unidirectional` แปลว่า
UDLD กำลังจะ Err-Disable พอร์ตนี้ในไม่ช้า

---

## Step 329 — Verification ทั้ง Topology ใหม่

### `show ip route` — ยืนยัน ECMP บน DIST-SW1

```
DIST-SW1# show ip route
Codes: O - OSPF, C - connected, S - static, L - local

Gateway of last resort is 10.255.10.1 to network 0.0.0.0

O*E2    0.0.0.0/0 [110/1] via 10.255.10.1, 00:15:22, TenGigabitEthernet1/1/1
                          [110/1] via 10.255.11.1, 00:15:22, TenGigabitEthernet1/1/2
C       10.10.10.0/24 is directly connected, Vlan10
C       10.10.20.0/24 is directly connected, Vlan20
C       10.10.99.0/24 is directly connected, Vlan99
O       10.10.30.0/24 [110/20] via 10.255.10.1, 00:14:01, TenGigabitEthernet1/1/1
                       [110/20] via 10.255.11.1, 00:14:01, TenGigabitEthernet1/1/2
O       10.10.40.0/24 [110/20] via 10.255.10.1, 00:14:01, TenGigabitEthernet1/1/1
                       [110/20] via 10.255.11.1, 00:14:01, TenGigabitEthernet1/1/2
C       10.255.10.0/30 is directly connected, TenGigabitEthernet1/1/1
C       10.255.11.0/30 is directly connected, TenGigabitEthernet1/1/2
```

**สอง Next-hop ต่อหนึ่ง Prefix คือหลักฐานว่า ECMP ทำงานสำเร็จ** — เทียบกับ Part 10 ที่มี
Next-hop เดียวเท่านั้นในทุก Prefix

### `show ip cef` — ยืนยันว่า FIB ก็มี 2 Path เช่นกัน (ไม่ใช่แค่ RIB)

```
DIST-SW1# show ip cef 10.10.30.0
10.10.30.0/24
  nexthop 10.255.10.1 TenGigabitEthernet1/1/1
  nexthop 10.255.11.1 TenGigabitEthernet1/1/2
```

### `show ip ospf neighbor` — ยืนยัน Full Mesh Adjacency ครบทุกคู่

```
CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/  -        00:00:38    10.255.10.2     Te1/0/1
1.1.1.12        1     FULL/  -        00:00:39    10.255.20.2     Te1/0/2
1.1.1.13        1     FULL/  -        00:00:37    10.255.31.2     Te1/0/3
1.1.1.14        1     FULL/  -        00:00:36    10.255.41.2     Te1/0/4
1.1.1.2         1     FULL/  -        00:00:33    10.255.0.2      Po1
```

**CORE-SW1 ต้องเห็น Neighbor ครบ 5 ตัว** (DIST-SW1–4 ทั้งหมด + CORE-SW2) — ก่อน Part นี้
CORE-SW1 จะเห็นแค่ DIST-SW1, DIST-SW2 และ CORE-SW2 (ไม่เห็น DIST-SW3/DIST-SW4 เลย
เพราะไม่มีลิงก์ตรงถึงกัน) นี่คือหลักฐานชัดเจนที่สุดว่า Full Mesh Topology ทำงานสำเร็จ

```
DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1         1     FULL/  -        00:00:35    10.255.10.1     Te1/1/1
1.1.1.2         1     FULL/  -        00:00:39    10.255.11.1     Te1/1/2
```

**DIST-SW1 ต้องเห็น Neighbor ครบ 2 ตัว** (CORE-SW1 และ CORE-SW2 พร้อมกัน) — ก่อน Part นี้
เห็นแค่ CORE-SW1 เท่านั้น

### `show udld neighbors` — ตรวจสอบทั้งเครือข่ายในคำสั่งเดียว

```
DIST-SW1# show udld neighbors

Port        Device Name    Device ID  Port ID    Neighbor State
----------- -------------- ---------- ---------- --------------
Te1/1/1     CORE-SW1       1          Te1/0/1    Bidirectional
Te1/1/2     CORE-SW2       1          Te1/0/3    Bidirectional
```

### Checklist Verification สรุป ก่อนถือว่า Redesign สำเร็จ

1. `show ip ospf neighbor` ทุก DIST-SW เห็น Neighbor ครบ **2 ตัว** (CORE-SW1 + CORE-SW2)
2. `show ip ospf neighbor` ทุก CORE-SW เห็น Neighbor ครบ **5 ตัว** (DIST-SW1–4 + Core อีกตัว)
3. `show ip route` ของทุก Prefix ข้าม Distribution Block มี **2 Next-hop เท่ากัน** (ECMP)
4. `show udld neighbors` ทุกพอร์ต Backbone แสดง **Bidirectional**
5. `show spanning-tree summary` ยังทำงานถูกต้องที่ **Access↔Distribution เท่านั้น**
   (ไม่มี STP Instance ใดพยายามคลุม Interface ที่เป็น Routed Port)

---

## Step 330 — Lab เต็มรูปแบบ: Redesign + ECMP + UDLD + ทดสอบ Failover

### เป้าหมายของ Lab นี้

Config เต็มรูปแบบให้ **DIST-SW1** (ตัวแทน DIST-SW2/3/4 ที่ใช้ Pattern เดียวกัน) มี:
1. Dual Routed Uplink ไปทั้ง CORE-SW1 และ CORE-SW2
2. OSPF ECMP พร้อม Reference Bandwidth ที่ปรับแล้ว
3. UDLD Aggressive บนทุกพอร์ต Fiber Backbone
4. ทดสอบ Failover จริงด้วยการถอดสาย แล้วพิสูจน์ว่า Traffic ไม่สะดุดแบบ STP เดิม

### Running-Config เต็มของ DIST-SW1 (ส่วนที่เกี่ยวข้องกับ Part นี้)

```
hostname DIST-SW1
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
errdisable recovery cause bpduguard
errdisable recovery cause udld
errdisable recovery interval 300
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW1 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 spanning-tree guard root
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW2 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 spanning-tree guard root
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW1 (Part 10, existing) **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 udld port aggressive
!
interface TenGigabitEthernet1/1/2
 description ** Routed uplink to CORE-SW2 (NEW - Part 33 HA) **
 no switchport
 ip address 10.255.11.2 255.255.255.252
 udld port aggressive
!
router ospf 1
 router-id 1.1.1.11
 auto-cost reference-bandwidth 100000
 maximum-paths 4
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 network 10.255.10.0 0.0.0.3 area 0
 network 10.255.11.0 0.0.0.3 area 0
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,99 root primary
spanning-tree loopguard default
spanning-tree portfast bpduguard default
```

### Verify ก่อนทดสอบ Failover

```
DIST-SW1# show ip route ospf | include via
O       10.10.30.0/24 [110/20] via 10.255.10.1, TenGigabitEthernet1/1/1
                       [110/20] via 10.255.11.1, TenGigabitEthernet1/1/2
O       10.10.40.0/24 [110/20] via 10.255.10.1, TenGigabitEthernet1/1/1
                       [110/20] via 10.255.11.1, TenGigabitEthernet1/1/2

DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1         1     FULL/  -        00:00:34    10.255.10.1     Te1/1/1
1.1.1.2         1     FULL/  -        00:00:38    10.255.11.1     Te1/1/2

DIST-SW1# show udld neighbors
Port        Device Name    Device ID  Port ID    Neighbor State
----------- -------------- ---------- ---------- --------------
Te1/1/1     CORE-SW1       1          Te1/0/1    Bidirectional
Te1/1/2     CORE-SW2       1          Te1/0/3    Bidirectional
```

### ทดสอบ Failover จริง: ถอดสาย Te1/1/1 (Uplink ไป CORE-SW1) ระหว่าง Ping ต่อเนื่อง

```
! เปิด ping ต่อเนื่องจาก PC1 (VLAN10) ไป Server1 (VLAN30) ค้างไว้ก่อนถอดสาย
PC1> ping 10.10.30.10 -t

Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
Reply from 10.10.30.10: bytes=32 time=1ms TTL=125
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
                                                        <-- ถอดสาย Te1/1/1 ตรงนี้ (จำลอง CORE-SW1 ล่ม)
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125      <-- แทบไม่มี packet loss เลย!
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
```

```
*Mar  1 15:03:11.220: %LINK-3-UPDOWN: Interface TenGigabitEthernet1/1/1,
changed state to down
*Mar  1 15:03:11.226: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on
TenGigabitEthernet1/1/1 from FULL to DOWN, Neighbor Down: Interface down or detached

DIST-SW1# show ip route ospf | include via
O       10.10.30.0/24 [110/20] via 10.255.11.1, TenGigabitEthernet1/1/2   <- เหลือ Path เดียว
O       10.10.40.0/24 [110/20] via 10.255.11.1, TenGigabitEthernet1/1/2   <- เหลือ Path เดียว
```

### ทำไม Packet Loss แทบไม่มีเลย (ต่างจาก STP ที่ต้องรอ Convergence)

| ขั้นตอน | แบบเดิม (STP Block ลิงก์สำรอง) | แบบใหม่ (OSPF ECMP) |
|---|---|---|
| สถานะก่อนลิงก์ล่ม | ลิงก์สำรอง Block อยู่ ไม่มี Traffic วิ่งผ่านเลย และ **ไม่มี FIB Entry เตรียมไว้ล่วงหน้า** | ลิงก์สำรอง Forward Traffic อยู่แล้วจริง และ **มี FIB Entry ทั้ง 2 เส้นเตรียมไว้ในฮาร์ดแวร์ล่วงหน้าแล้ว** |
| เมื่อลิงก์หลักล่ม | ต้องรอ STP ตรวจจับ + Proposal/Agreement (RSTP) + เปลี่ยน Port Role จาก Blocking เป็น Forwarding | แค่ **ลบ FIB Entry ของ Path ที่ตายออกไป** — Path ที่เหลือ Active อยู่แล้วไม่ต้องทำอะไรเพิ่ม |
| เวลาที่ใช้ | ไม่กี่วินาที (RSTP) หรือสูงสุด ~50 วินาที (802.1D เดิม) | **Sub-second** (แค่รอ Local Interface Down Event + ลบ FIB Entry เดิม) |
| ต้องพึ่ง Convergence ของ Protocol ควบคุม Topology หรือไม่ | ต้องพึ่ง STP เต็มรูปแบบ | **ไม่ต้องรอ SPF คำนวณใหม่ทั้งหมดด้วยซ้ำ** เพราะ Path ที่เหลือ Active และอยู่ใน FIB ตั้งแต่ต้นอยู่แล้ว |

> **นี่คือเหตุผลที่แท้จริงที่ Enterprise Campus ระดับ CCNP นิยม "Route ให้เร็วที่สุดเท่าที่ทำได้"
> (Push Layer 3 ให้ลึกลงไปในสถาปัตยกรรมให้มากที่สุด)** — ไม่ใช่แค่เพราะ Routing "ฉลาดกว่า"
> Switching แต่เพราะ **ECMP ทำให้ Backup Path Active อยู่ตลอดเวลา** ต่างจาก STP ที่บังคับให้
> Backup Path ว่างเปล่าจนกว่าจะเกิดปัญหา — Bandwidth ที่เคยถูก "กันสำรองทิ้งเฉยๆ" ก็ถูกนำมาใช้
> งานจริงเต็มรูปแบบไปพร้อมกันด้วย

### เสียบสาย Te1/1/1 กลับเข้าที่เดิม — ยืนยัน ECMP กลับมาอัตโนมัติ

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# end

! รอ OSPF Hello + Dead Timer ปกติ (ไม่กี่วินาที) แล้วตรวจสอบอีกครั้ง
DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1         1     FULL/  -        00:00:39    10.255.10.1     Te1/1/1
1.1.1.2         1     FULL/  -        00:00:36    10.255.11.1     Te1/1/2

DIST-SW1# show ip route ospf | include via
O       10.10.30.0/24 [110/20] via 10.255.10.1, TenGigabitEthernet1/1/1
                       [110/20] via 10.255.11.1, TenGigabitEthernet1/1/2
```

ECMP กลับมาเป็น 2 Path เหมือนเดิมโดยอัตโนมัติ — ไม่ต้องเข้าไป Config อะไรเพิ่มเติมเลย
(Self-healing เต็มรูปแบบ สอดคล้องกับหลักการ Dynamic Routing Protocol ที่ Part 11 สอนไว้)

### Config บน DIST-SW2, DIST-SW3, DIST-SW4 — Pattern เดียวกัน

| Device | Root ที่ต้องเป็น | Uplink เดิม | Uplink ใหม่ | OSPF network เพิ่ม |
|---|---|---|---|---|
| DIST-SW2 | Root Secondary VLAN10/20/99 | Te1/1/1 → CORE-SW1 (10.255.20.2/30) | Te1/1/2 → CORE-SW2 (10.255.21.2/30) | `network 10.255.21.0 0.0.0.3 area 0` |
| DIST-SW3 | Root Primary VLAN30/40/99 | Te1/1/1 → CORE-SW2 (10.255.30.2/30) | Te1/1/2 → CORE-SW1 (10.255.31.2/30) | `network 10.255.31.0 0.0.0.3 area 0` |
| DIST-SW4 | Root Secondary VLAN30/40/99 | Te1/1/1 → CORE-SW2 (10.255.40.2/30) | Te1/1/2 → CORE-SW1 (10.255.41.2/30) | `network 10.255.41.0 0.0.0.3 area 0` |

ทุกตัวต้องมี `auto-cost reference-bandwidth 100000`, `maximum-paths 4`, และ
`udld port aggressive` บน Interface ใหม่เหมือนกันหมดตาม Pattern ของ DIST-SW1 ข้างต้น

Lab นี้ถือว่าสำเร็จเมื่อ: (1) ทุก DIST-SW เห็น OSPF Neighbor ครบ 2 ตัว, (2) ทุก Prefix ข้าม
Distribution Block มี ECMP 2 Path, (3) UDLD แสดง Bidirectional ทุกพอร์ต Backbone, และ
(4) ทดสอบถอดสาย Uplink เส้นใดเส้นหนึ่งแล้ว Ping ยังคงต่อเนื่องแทบไม่มี Packet Loss

---

## แบบฝึกหัดทวนความเข้าใจ Part 33

1. เพราะเหตุใดการทำ Dual-Homing ระหว่าง Distribution↔Core ด้วย **Routed Port** จึงไม่ต้อง
   พึ่ง STP เลย ในขณะที่การทำ Dual-Homing แบบเดียวกันด้วย **Trunk (L2)** จะต้องพึ่ง STP เสมอ?
2. ทำไมหลักสูตรนี้ยังเลือกใช้ Traditional Design (L2 Access + L3 Distribution) แทนที่จะเป็น
   Routed Access ทั้งที่ Routed Access ไม่ต้องใช้ STP เลยแม้แต่จุดเดียว?
3. หลังจาก Redesign ใน Part นี้แล้ว STP ยังจำเป็นต้องทำงานอยู่ที่จุดใดของ Topology และทำไม
   Config `spanning-tree vlan ... root primary` ที่เคยตั้งไว้บน CORE-SW1 (Part 5) จึงต้องย้าย
   มาไว้ที่ Distribution Layer แทน?
4. FIB และ Adjacency Table ใน CEF แต่ละอันเก็บข้อมูลอะไร และทำงานร่วมกันอย่างไรเพื่อให้
   Multilayer Switch Forward Packet ได้ที่ Wire-speed?
5. UDLD Aggressive Mode ต่างจาก Normal Mode อย่างไร และทำไมจึงจำเป็นต้องเปิดบน Backbone
   Link แบบ Routed Port ทั้งที่ Loop Guard เคยแก้ปัญหา Unidirectional Link ได้แล้วใน Part 5?

**เฉลย:**

1. Routed Port ไม่มี Layer 2 STP ทำงานอยู่เลยตั้งแต่ต้น (ไม่มี Port Role/BPDU) เพราะไม่มี VLAN
   ข้ามผ่าน ดังนั้นการมี 2 เส้นทางไป Core พร้อมกันจึงไม่สร้าง Layer 2 Loop และ OSPF จัดการ
   ด้วย ECMP แทน ในขณะที่ Trunk (L2) 2 เส้นไป Core ที่ต่างกัน จะสร้าง Physical Loop ที่ Layer 2
   ทันที ซึ่ง STP ต้อง Block เส้นใดเส้นหนึ่งเพื่อป้องกัน Broadcast Storm ตามหลักการ Part 5
2. เพราะ Access Switch ในหลักสูตรนี้ (ACCESS-SW1–4) เป็น Catalyst 9200/2960 ซึ่งเป็น
   Layer 2 ล้วนตามที่กำหนดไว้ตั้งแต่ Part 1 — ไม่มีความสามารถสร้าง SVI ที่ Route ได้เลยทาง
   Hardware Routed Access จึงทำไม่ได้จนกว่าจะเปลี่ยน Hardware หรือย้ายไปสถาปัตยกรรม
   SD-Access (Part 38) ซึ่งเป็นการออกแบบคนละแบบทั้งหมด
3. STP ยังจำเป็นเฉพาะที่ขอบ **Access↔Distribution** (dual-homed L2 Trunk ระหว่าง ACCESS-SW
   กับ DIST-SW คู่ของมัน) เพราะเป็นจุดเดียวที่ยังมี VLAN ข้ามหลาย switch จริง — ส่วน
   `root primary` บน CORE-SW1 ไม่มีผลจริงอีกต่อไปตั้งแต่ Part 10 เพราะ CORE-SW1 ไม่มี
   Switchport ของ VLAN เหล่านั้นเลย (เป็น Routed Port ทั้งหมด) Root Bridge ตัวจริงของแต่ละ
   VLAN จึงต้องถูกกำหนดไว้ที่ DIST-SW ที่แท้จริง (DIST-SW1/DIST-SW3 เป็น Root Primary,
   DIST-SW2/DIST-SW4 เป็น Root Secondary ตามบล็อกของตัวเอง)
4. **FIB** เก็บว่าปลายทาง Network ใดควรออกทาง Interface ใดและ Next-hop คือใคร (คัดลอกจาก
   Routing Table แต่ Optimize สำหรับ Lookup เร็ว) ส่วน **Adjacency Table** เก็บ Layer 2 Header
   ที่เตรียมไว้ล่วงหน้าสำหรับแต่ละ Next-hop (คัดลอกจาก ARP Table) — เมื่อ Packet มาถึง
   CEF ใช้ผลจาก FIB เพื่อรู้ทางออก แล้วดึง Header จาก Adjacency Table มาแปะทันทีโดยไม่ต้อง
   ARP หรือคำนวณใหม่ ทำให้ทำใน TCAM/ASIC ได้ในรอบ Clock เดียว
5. Normal Mode จะไม่ Err-Disable ทันทีเมื่อขาดการติดต่อ (แค่ทำเครื่องหมาย Undetermined)
   ในขณะที่ Aggressive Mode จะพยายามส่งซ้ำ 8 ครั้งก่อนตัดสินใจ Err-Disable ทำให้ตรวจจับ
   Unidirectional Link ได้เร็วและแม่นยำกว่า — จำเป็นต้องเปิดเพราะ Loop Guard ทำงานเฉพาะบน
   Switchport ที่มี STP เท่านั้น แต่ลิงก์ Backbone ตอนนี้เป็น Routed Port ล้วนซึ่งไม่มี STP
   ทำงานอยู่เลย ทำให้ Loop Guard ใช้ป้องกันไม่ได้อีกต่อไป UDLD จึงเป็นกลไกเดียวที่เหลืออยู่
   สำหรับตรวจจับปัญหานี้ที่ Layer 2 ก่อนที่ OSPF จะต้องรอ Dead Timer เต็มรูปแบบ

---

## สรุป Part 33

Part นี้ทำตามสัญญาที่ [Part 10](part-010-inter-vlan-routing.md) และ
[`00-ip-address-plan.md`](00-ip-address-plan.md) ทิ้งไว้ตั้งแต่ต้น — เราออกแบบ Topology ใหม่ให้
**DIST-SW1–4 ทุกตัวเชื่อมต่อ Routed Uplink ไปทั้ง CORE-SW1 และ CORE-SW2 พร้อมกัน**
(Full Mesh Distribution-Core) เปลี่ยนปัญหา "L2 Redundancy ที่ STP ต้อง Block ลิงก์สำรองทิ้ง"
ให้กลายเป็น "L3 ECMP ที่ใช้งานทุกเส้นพร้อมกันจริง" ด้วย OSPF, ทบทวนและแก้ไข Root Bridge
Design จาก Part 5 ให้ตรงกับความเป็นจริงของสถาปัตยกรรม Routed Core, เจาะลึกกลไกภายในของ
Multilayer Switch (CEF/FIB/Adjacency/TCAM) ที่ทำให้ทุกอย่างนี้เกิดขึ้นได้ที่ Wire-speed, และปิดท้าย
ด้วย UDLD ที่อุดช่องโหว่ Unidirectional Link ซึ่ง Loop Guard เดิมทำไม่ได้อีกต่อไปบน Routed Port

ตอนนี้ Campus Network ของเราแข็งแกร่งขึ้นมากในระดับ Logical Design (ECMP, Full Mesh,
UDLD) แต่ยังมีอีกมิติหนึ่งที่ยังไม่ได้แตะเลย นั่นคือ **ความซ้ำซ้อนในระดับ Physical Chassis
เอง** — ถ้า Distribution Switch **ทั้งตัว** ล่ม (ไม่ใช่แค่ Link เส้นเดียว) Access Layer ที่พึ่งพา
Switch ตัวนั้นเพียงตัวเดียวจะยังคงได้รับผลกระทบอยู่ดี **Part 34 — StackWise, VSS & MEC**
จะพาไปแก้ปัญหานี้ต่อ ด้วยเทคนิคการรวมหลาย Chassis ให้เห็นเป็น Logical Switch ตัวเดียว
(StackWise Virtual, VSS) และ Multi-chassis EtherChannel (MEC) ที่ทำให้แม้แต่ Access Switch
เองก็ Dual-Home ไปยัง "Distribution ตัวเดียว" ที่แท้จริงประกอบด้วย Physical Chassis 2 ตัวได้

**ไปต่อ:** [Part 34 — StackWise, VSS & MEC →](part-034-stackwise-vss-mec.md)
