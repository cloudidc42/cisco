# Part 29 — BGP Fundamentals (eBGP)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 281–290 จาก 1000**

> ต่อจาก [Part 28 — EIGRP Advanced (Named Mode, Query/Stuck-in-Active, Route Summarization
> ขั้นสูง)](part-028-eigrp-advanced.md) ที่ปิดท้ายเนื้อหา **IGP (Interior Gateway Protocol)**
> ระดับ CCNP ไปแล้ว Part นี้จะย้ายจุดสนใจไปสู่ Protocol ที่ทำงาน**นอกขอบเขตองค์กร (Exterior
> Gateway Protocol)** นั่นคือ **BGP (Border Gateway Protocol)** — Routing Protocol เพียงตัวเดียว
> ที่ขับเคลื่อน Internet ทั้งใบให้ทำงานได้
>
> ใน [Part 19 Step 187](part-019-wan-technologies.md#step-187--คอนฟิก-mpls-l3vpn-จากมุมมองลูกค้า-ce)
> เราเคยเห็น eBGP แบบสั้นๆ ตอนคอนฟิก **WAN-EDGE-1** เป็น CE คุยกับ PE จำลองที่ `192.0.2.1`
> AS 65001 — นั่นเป็นเพียง "ตัวอย่างผ่านๆ" เพื่อให้ Lab MPLS สมบูรณ์เท่านั้น **Part นี้คือจุดที่
> เราจะเรียน BGP อย่างจริงจังตั้งแต่พื้นฐาน**: BGP คืออะไร, ทำไม Internet ทั้งใบใช้ Protocol นี้,
> AS Number ทำงานอย่างไร, และคอนฟิก **eBGP (External BGP)** ที่ **WAN Edge จริง** ของหลักสูตร —
> ระหว่าง **WAN-EDGE-1 / WAN-EDGE-2 (AS 65001, ASN ขององค์กรเรา)** กับ **ISP-RTR (AS 65000,
> จำลอง ISP)** บน Link `203.0.113.0/30` และ `203.0.113.4/30` ที่ประกาศไว้แล้วใน
> [00-ip-address-plan.md](00-ip-address-plan.md) ตั้งแต่ Part 19
>
> **สิ่งที่ Part นี้ยังไม่ลงลึก (ตั้งใจเก็บไว้)**: กลไก **iBGP แบบเต็มรูปแบบ** (Full Mesh, Route
> Reflector, Synchronization), **BGP Best Path Selection Algorithm ครบทั้ง 13 ขั้นตอน**,
> **Route-Map ขั้นสูงสำหรับ Policy-Based Routing/Traffic Engineering (Local-Preference, MED,
> AS-Path Prepending แบบเจาะลึก)**, และ **BGP Communities** — ทั้งหมดนี้จะสอนแบบเต็มรูปแบบใน
> [Part 30 — BGP Advanced](part-030-bgp-advanced.md)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 281 | BGP คืออะไร — Path-Vector Protocol, AS Number, ทำไมต้องใช้ BGP ที่ WAN Edge/Multihoming |
| 282 | eBGP vs iBGP (Preview) และ BGP Neighbor State Machine |
| 283 | คอนฟิก eBGP พื้นฐาน — `router bgp`, `neighbor remote-as`, `network` statement |
| 284 | ภาพรวม BGP Path Attributes — AS_PATH, NEXT_HOP, ORIGIN, LOCAL_PREF, MED, WEIGHT |
| 285 | การประกาศ Route เข้า BGP — `network` vs `redistribute` vs `aggregate-address` |
| 286 | BGP Route Filtering พื้นฐาน — Prefix-list, AS-Path Access-list |
| 287 | eBGP Multihop และ Load-Sharing ข้าม 2 ISP Link (WAN-EDGE-1 / WAN-EDGE-2) |
| 288 | BGP กับ Default Route — Full Table vs Default-Only, `default-originate` |
| 289 | Verification/Troubleshooting — `show ip bgp summary/neighbors/`, ปัญหาที่พบบ่อย |
| 290 | Lab เต็มรูปแบบ: Dual-Homed eBGP Internet Edge |

---

## Step 281 — BGP คืออะไร: Path-Vector Protocol และเหตุผลที่ต้องใช้ที่ WAN Edge

### BGP คือ Protocol ที่ทำให้ Internet ทำงานได้

**BGP (Border Gateway Protocol, ปัจจุบันคือ BGP-4, RFC 4271)** คือ Routing Protocol ที่ใช้
แลกเปลี่ยน Routing Information **ระหว่างองค์กร (Autonomous System)** — พูดง่ายๆ คือ Protocol
ที่ ISP ทุกรายทั่วโลกใช้คุยกันเพื่อบอกว่า "Prefix นี้ไปทางฉัน" จน Internet ทั้งใบมี Routing Table
ที่รู้จักเส้นทางไปยัง Network กว่า 900,000+ Prefix (IPv4, ปี 2025) ได้

ต่างจาก OSPF/EIGRP (IGP) ที่เราเรียนมาตลอด Part 11-14, 28 ที่ออกแบบมาให้ **Router ภายในองค์กร
เดียวกัน** เรียนรู้ Network ของกันและกันอย่างรวดเร็วและเชื่อถือทุก Metric แบบละเอียด BGP ถูก
ออกแบบมาให้ **ควบคุมได้ (Policy-Based)**, **Scale ได้มหาศาล** (Routing Table ระดับล้าน Prefix),
และ **มีเสถียรภาพสูง** (ไม่ Flap ง่ายเหมือน IGP) เพราะ Internet ประกอบด้วยองค์กรนับหมื่นนับแสน
รายที่ **ไม่ไว้ใจกันโดยอัตโนมัติ** — แต่ละองค์กรต้องการควบคุมว่าจะ "รับ" Route ไหนจากใคร และจะ
"แจก" Route ไหนให้ใครบ้าง

### BGP คือ Exterior Gateway Protocol (EGP) ไม่ใช่ IGP

| ประเด็น | IGP (OSPF, EIGRP) | BGP (EGP) |
|---|---|---|
| ขอบเขตการทำงาน | ภายใน 1 Autonomous System (AS) เดียวกัน | **ระหว่าง** AS ต่างๆ (เดิมทีออกแบบมาแบบนี้ แต่ก็ใช้ภายใน AS เดียวกันได้ผ่าน iBGP) |
| เป้าหมายหลัก | หา Best Path ที่**เร็วที่สุด**ตาม Metric (Cost, Bandwidth+Delay) | หา Best Path ตาม **Policy** ที่ Admin กำหนด ไม่ใช่แค่เร็วที่สุด |
| ความเร็วในการ Converge | เร็วมาก (วินาที) ออกแบบมาเพื่อ Real-time Network ภายใน | ช้ากว่า (สิบวินาทีถึงนาที) เพราะเน้นเสถียรภาพมากกว่าความเร็ว |
| Scale | รองรับ Router หลักร้อยถึงหลักพันตัว, Route หลักพัน-หมื่น | รองรับ Route ระดับ**ล้าน** Prefix ทั่ว Internet |
| อัลกอริทึม | Distance Vector (EIGRP, Diffusing) หรือ Link-State (OSPF) | **Path-Vector** (ลูกผสม มีคุณสมบัติพิเศษเฉพาะตัว) |
| Trust Model | เชื่อ Router เพื่อนบ้านในองค์กรเดียวกันโดย Default | **ไม่เชื่อโดยอัตโนมัติ** — ต้อง Config Neighbor และ Filter Route อย่างระมัดระวังเสมอ |

### Path-Vector Protocol คืออะไร — ต่างจาก Distance-Vector และ Link-State อย่างไร

| ประเภท Algorithm | ตัวอย่าง | ข้อมูลที่ใช้ตัดสินใจ Best Path | จุดเด่น |
|---|---|---|---|
| **Distance-Vector** | EIGRP, RIP | Hop count หรือ Composite Metric (Bandwidth/Delay) ที่ได้รับต่อจาก Neighbor | เรียบง่าย, คำนวณเบา |
| **Link-State** | OSPF, IS-IS | แผนที่ Topology ทั้งหมด (LSDB) แล้วรัน SPF เอง | เห็นภาพรวม Network ทั้งหมด, Converge เร็ว |
| **Path-Vector** | **BGP** | **AS_PATH** — รายการ AS ทั้งหมดที่ Prefix นั้นเดินทางผ่านมา (ไม่ใช่แค่ Hop Count) | ป้องกัน Routing Loop ระดับ AS ได้เอง (เห็น AS ตัวเองใน AS_PATH = Reject ทันที), รองรับ Policy ที่ซับซ้อนผ่าน AS_PATH |

```
ตัวอย่าง AS_PATH ที่ BGP เห็น (แนวคิด Path-Vector):

Prefix 8.8.8.0/24 มาถึงเราด้วย AS_PATH = [65010, 65020, 65030, 15169]
                                            │       │       │       │
                                            │       │       │       └─ Origin AS (เจ้าของ Prefix จริง)
                                            │       │       └───────── AS ที่ 3 ที่ Prefix เดินทางผ่าน
                                            │       └───────────────── AS ที่ 2 ที่ Prefix เดินทางผ่าน
                                            └───────────────────────── AS ที่อยู่ใกล้เราที่สุด (ส่งต่อมาให้เรา)

ถ้า AS ของเราเอง (เช่น 65001) ปรากฏอยู่ใน AS_PATH นี้แล้ว = BGP จะ Reject Prefix นี้ทันที
(ป้องกัน Routing Loop โดยอัตโนมัติ — ไม่ต้องมี Split Horizon/Poison Reverse แบบ Distance-Vector)
```

### Autonomous System Number (ASN) — บัตรประชาชนขององค์กรบน Internet

**Autonomous System (AS)** คือกลุ่มของ Network ที่อยู่ภายใต้การบริหารจัดการเดียวกัน (Single
Administrative Domain) และมี Routing Policy เป็นหนึ่งเดียวเมื่อมองจากภายนอก — แต่ละ AS ต้องมี
**AS Number (ASN)** ที่ไม่ซ้ำกันทั่วโลกเพื่อใช้ระบุตัวตนใน BGP (คล้าย Router-ID ของ OSPF/EIGRP
แต่เป็นระดับ "องค์กร" ไม่ใช่ระดับ "Router")

| ช่วง ASN (2-byte, "asplain") | ประเภท | รายละเอียด |
|---|---|---|
| 0 | Reserved | ห้ามใช้ (RFC 7607) |
| **1 – 64495** | **Public (Assignable)** | จัดสรรโดย IANA ผ่าน RIR ภูมิภาค (ARIN, RIPE NCC, APNIC, LACNIC, AFRINIC) ให้ ISP/องค์กรขนาดใหญ่ทั่วโลกใช้บน Internet จริง (เช่น AS 15169 = Google, AS 7922 = Comcast) |
| 23456 | AS_TRANS | ใช้เฉพาะสำหรับ Backward Compatibility ระหว่าง 2-byte ASN กับ 4-byte ASN (RFC 6793) |
| 64496 – 64511 | Reserved for Documentation | RFC 5398 — ใช้ในเอกสาร/ตัวอย่างเท่านั้น เหมือน RFC 5737 สำหรับ IP Address |
| **64512 – 65534** | **Private Use** | เหมือน **RFC 1918** ของ IP Address — ใช้ภายในองค์กรหรือ Lab เท่านั้น **ห้ามปรากฏบน Internet จริง** |
| 65535 | Reserved | ห้ามใช้ (RFC 7300) |

> **หมายเหตุ**: ยังมี **4-byte ASN** (RFC 6793, ช่วง 65536 – 4,294,967,295) เพิ่มเข้ามาเพราะ
> ASN แบบ 2-byte เริ่มไม่พอใช้งานตั้งแต่ปี 2011 เป็นต้นมา — ISP ใหญ่ๆ สมัยใหม่จำนวนมากใช้ ASN
> รูปแบบนี้แล้ว (เขียนแบบ asplain เช่น `398101` หรือแบบ asdot `6.5535`) แต่ระดับ CCNP ENCOR
> เน้นความเข้าใจ 2-byte ASN เป็นหลักตามที่ข้อสอบกำหนด
>
> **ในหลักสูตรนี้**: WAN-EDGE-1/WAN-EDGE-2 ใช้ **AS 65001** และ ISP-RTR (จำลอง ISP) ใช้
> **AS 65000** — ทั้งคู่อยู่ใน**ช่วง Private Use (64512-65534)** โดยตั้งใจ เหมือนที่หลักสูตรใช้
> IP Address ช่วง RFC 1918/RFC 5737 ตลอดมา เพื่อความปลอดภัยในการ Lab (ในโลกจริง ISP จะมี
> Public ASN เช่น AS 7922, AS 4837 ไม่ใช่ Private ASN)

### ทำไมต้องใช้ BGP ที่ WAN Edge/Internet — ทำไมใช้ OSPF/EIGRP กับ ISP ไม่ได้

| เหตุผล | คำอธิบาย |
|---|---|
| **Administrative Boundary** | OSPF/EIGRP ออกแบบมาให้ Router "ไว้ใจกัน" ในองค์กรเดียวกัน — ถ้าเปิด OSPF คุยกับ ISP โดยตรง เท่ากับให้ ISP (หรือ Router ที่ ISP ควบคุม) เข้ามามีอิทธิพลต่อ Routing Table ภายในทั้งหมดของเรา ซึ่งเป็นความเสี่ยงด้าน Security/Stability มหาศาล |
| **Scale ของ Internet Routing Table** | Internet Full Table มีมากกว่า **900,000 Prefix** (IPv4) — OSPF/EIGRP ไม่ได้ออกแบบมาให้รองรับขนาดนี้ (LSDB จะใหญ่เกินจัดการ, SPF จะหนักเกินไป) BGP ออกแบบมาเฉพาะสำหรับ Scale ระดับนี้ |
| **Policy Control** | องค์กรต้องการควบคุมว่า "จะรับ Route จากใคร", "จะแจก Route อะไรให้ใคร", "จะเลือกเส้นทางไหนเป็นหลักเมื่อมีหลาย ISP" — BGP มี Attribute มากมาย (Step 284) ที่ออกแบบมาเพื่อทำ Policy โดยเฉพาะ ต่างจาก IGP ที่เลือก Path จาก Metric ตัวเลขล้วนๆ |
| **เสถียรภาพ (Route Dampening)** | BGP มีกลไกลดผลกระทบจาก Prefix ที่ Flap บ่อยๆ (Route Flap Dampening) เพื่อไม่ให้ปัญหาเล็กๆ ที่ Edge หนึ่งกระทบ Router อื่นทั่ว Internet — เหมาะกับ Network ขนาดใหญ่ระดับโลกที่ต้อง "ทนทาน" มากกว่า "เร็ว" |
| **Multihoming** | องค์กรที่ต้องการ**ต่อ Internet มากกว่า 1 เส้น** (จาก ISP เดียวกันหรือคนละราย) เพื่อ Redundancy จำเป็นต้องใช้ BGP เพื่อให้ทั้งสองฝั่ง (องค์กร และ ISP) ควบคุม Traffic Engineering ได้ (เลือก Path หลัก/สำรอง, Load-Share) — นี่คือ Use Case ที่ตรงกับ Lab ของเราที่มี **WAN-EDGE-1** และ **WAN-EDGE-2** ต่อ ISP-RTR คนละ Link |

### BGP ใช้ TCP Port 179 — ทำไมไม่ใช้ Multicast Hello เหมือน OSPF/EIGRP

```
OSPF/EIGRP:  ใช้ Multicast (224.0.0.5/6 สำหรับ OSPF, 224.0.0.10 สำหรับ EIGRP) ส่ง Hello
             โดยตรงบน IP — ต้อง**ทำ Reliability เอง**ในตัว Protocol (OSPF ใช้ Ack, EIGRP ใช้ RTP)

BGP:         สร้าง TCP Session (Unicast, Port 179) ระหว่าง Neighbor ก่อนเสมอ
             ── ได้ Reliability, Flow Control, Segmentation จาก TCP โดยตรง ── ไม่ต้องออกแบบ
             กลไก Reliable Delivery ของตัวเองเลย (แตกต่างจาก IGP ที่ทำงานบน Layer 3 ตรงๆ)
```

- BGP Peer สื่อสารกันแบบ **Unicast TCP** เท่านั้น (ไม่มี Multicast Discovery แบบ OSPF/EIGRP)
  หมายความว่า **ต้อง Config Neighbor IP Address ด้วยตนเองเสมอ** (`neighbor <ip> remote-as
  <AS>`) — BGP จะไม่มีวัน "เจอ" Neighbor เองอัตโนมัติเหมือน OSPF Hello บน Broadcast Segment
- เพราะพึ่งพา TCP เต็มรูปแบบ การที่ TCP Port 179 ถูก Firewall/ACL บล็อกจึงเป็นสาเหตุอันดับต้นๆ
  ที่ BGP Neighbor ไม่ Establish (จะกลับมาดูใน Step 289)

### ภาพรวม Topology BGP ของหลักสูตรนี้

```
                         ┌────────────────────────────┐
                         │          ISP-RTR             │   AS 65000 (จำลอง ISP)
                         │      Lo0: 1.1.1.254/32       │
                         └──────┬────────────────┬─────┘
             203.0.113.0/30    │                │    203.0.113.4/30
             (.1 = ISP-RTR)    │                │    (.5 = ISP-RTR)
                               │                │
                    ┌──────────┴──────┐   ┌─────┴────────────┐
                    │   WAN-EDGE-1     │   │   WAN-EDGE-2      │
                    │   AS 65001       │   │   AS 65001        │
                    │   Lo0: 1.1.1.21  │   │   Lo0: 1.1.1.22   │
                    └──────┬───────────┘   └───────────┬───────┘
                           │      10.10.254.8/30        │
                           └─────────── Cross-link ─────┘
                           │  (iBGP + IGP backup path)   │
                    ┌──────┴───────────┐   ┌─────────────┴─────┐
                    │    CORE-SW1       │   │     CORE-SW2       │
                    └───────────────────┘   └────────────────────┘

  WAN-EDGE-1 ↔ ISP-RTR  = eBGP (AS 65001 ↔ AS 65000)  บน 203.0.113.0/30
  WAN-EDGE-2 ↔ ISP-RTR  = eBGP (AS 65001 ↔ AS 65000)  บน 203.0.113.4/30
  WAN-EDGE-1 ↔ WAN-EDGE-2 = iBGP (AS 65001 ↔ AS 65001) บน Cross-link (Step 287)
```

> **หมายเหตุการออกแบบ**: ตาม [00-ip-address-plan.md](00-ip-address-plan.md) อุปกรณ์ **ISP-RTR**
> เป็น Router จำลองตัวเดียวที่ทำหน้าที่แทน "ISP" ในภาพรวม และให้บริการ Internet Link แยกกัน 2 เส้น
> ไปยัง WAN-EDGE-1 และ WAN-EDGE-2 (**Dual-homed ไปยัง ISP รายเดียวกัน ผ่าน 2 Physical Link**
> เพื่อความสอดคล้องกับอุปกรณ์ที่ประกาศไว้แล้วในหลักสูตร) — ในการออกแบบระดับ Enterprise จริงที่
> ต้องการ Diversify ความเสี่ยงระดับ "ISP ทั้งรายล่ม" อาจเลือกต่อกับ **ISP คนละราย** (เช่น ISP2
> AS 65002) แทน ซึ่ง**หลักการคอนฟิก eBGP เหมือนกันทุกประการ** เพียงเปลี่ยนค่า `remote-as` ให้
> ตรงกับ ASN ของ ISP รายที่สองเท่านั้น (จะกล่าวถึงอีกครั้งใน Step 287)

---

## Step 282 — eBGP vs iBGP (Preview) และ BGP Neighbor State Machine

### eBGP vs iBGP — ภาพรวมสั้นๆ (รายละเอียดเต็มใน Part 30)

BGP มี 2 รูปแบบตามความสัมพันธ์ของ AS Number ระหว่าง Neighbor:

| คุณสมบัติ | **eBGP** (External BGP) | **iBGP** (Internal BGP) |
|---|---|---|
| AS ของ Neighbor | **ต่างกัน** (เช่น AS 65001 ↔ AS 65000) | **เหมือนกัน** (เช่น AS 65001 ↔ AS 65001) |
| Use Case หลักในหลักสูตรนี้ | WAN-EDGE-1/2 ↔ ISP-RTR (Part นี้) | WAN-EDGE-1 ↔ WAN-EDGE-2 (Preview Step 287, เต็มรูปแบบ Part 30) |
| Administrative Distance (Cisco) | **20** | **200** |
| TTL เริ่มต้นของ Packet | 1 (สมมติว่า Neighbor ต่อกันโดยตรง) | 255 (รองรับ Neighbor ที่อยู่ไกลกันหลาย Hop ผ่าน IGP ได้โดย Default) |
| การเปลี่ยน NEXT_HOP | เปลี่ยนเป็น IP ของตัวเองเมื่อส่งต่อ Prefix ออกไป (Default Behavior) | **ไม่เปลี่ยน NEXT_HOP** โดย Default (ต้องใช้ `next-hop-self` ถ้าจำเป็น) — ตัวนี้คือกับดักคลาสสิกของ iBGP |
| ต้องทำ Full-Mesh ระหว่าง Router ในองค์กรหรือไม่ | ไม่เกี่ยวข้อง (เป็นการคุยข้าม AS) | **ต้องทำ Full-Mesh หรือใช้ Route Reflector/Confederation** เพราะ iBGP ไม่ Re-advertise Prefix ที่เรียนจาก iBGP Neighbor ตัวหนึ่งไปให้ iBGP Neighbor อีกตัว (กฎ Split-Horizon ของ BGP) |
| Synchronization Rule (ในอดีต) | ไม่เกี่ยวข้อง | Cisco IOS สมัยใหม่ปิด Default อยู่แล้ว (`no synchronization` เป็นค่าเริ่มต้น) |

> ใน Part นี้เราจะสัมผัส iBGP แค่เพียงพอให้ Lab Dual-Homed ทำงานได้จริงใน **Step 287** เท่านั้น
> (สร้าง Session ระหว่าง WAN-EDGE-1 ↔ WAN-EDGE-2) ส่วนกลไกเต็มรูปแบบ (Full-Mesh Scaling
> Problem, Route Reflector, Confederation, Next-Hop-Self ภาคปฏิบัติ) จะสอนละเอียดใน
> [Part 30 — BGP Advanced](part-030-bgp-advanced.md)

### BGP Message Types

| Message | หน้าที่ |
|---|---|
| **OPEN** | Message แรกที่ส่งหลัง TCP Connect สำเร็จ — แลก BGP Version, ASN ของตัวเอง, Hold Time, Router-ID, Capability (เช่น รองรับ 4-byte ASN, Multiprotocol Extension สำหรับ IPv6/VPNv4 หรือไม่) |
| **UPDATE** | Message หลักที่ใช้ส่ง Route จริง — ประกอบด้วย NLRI (Prefix ที่ประกาศ), Path Attribute (AS_PATH, NEXT_HOP, ...), และ Withdrawn Routes (Prefix ที่ถอนออก) |
| **KEEPALIVE** | ส่งเป็นระยะ (Default ทุก 60 วินาที) เพื่อบอกว่า Session ยังมีชีวิตอยู่ — ถ้าไม่ได้รับภายใน Hold Time (Default 180 วินาที) จะถือว่า Neighbor ตาย |
| **NOTIFICATION** | ส่งเมื่อเกิด Error และจะ**ปิด Session ทันที** — มี Error Code/Subcode บอกสาเหตุ (เช่น AS Mismatch, Hold Timer Expired, Malformed Attribute) ใช้ในการ Troubleshoot (Step 289) |

### BGP Neighbor State Machine

BGP Session ต้องผ่าน 6 State ตามลำดับก่อนจะ **Established** (พร้อมแลก Route จริง):

```
   ┌────────┐   TCP เริ่ม Connect        ┌──────────┐
   │  Idle   ├───────────────────────────►│ Connect  │
   └────┬───┘                             └─────┬────┘
        │ Reset/Error                           │ TCP Connect สำเร็จ
        │                                        ▼
        │                                  ส่ง OPEN Message
        │                                        │
        │           TCP Connect ล้มเหลว          ▼
        │        ┌──────────────────────────────────┐
        └────────┤              Active               │◄── ถ้า TCP ต่อไม่ติด (ACL บล็อก,
                 └──────────────┬───────────────────┘    ไม่มี Route ถึง Peer ฯลฯ) จะวนอยู่
                                 │ TCP Connect สำเร็จ (retry)      ที่ State นี้ซ้ำๆ (ดู Step 289)
                                 ▼
                          ส่ง OPEN Message
                                 │
                                 ▼
                          ┌─────────────┐   ได้รับ OPEN จาก Peer, ตรวจสอบผ่าน (AS ตรง,
                          │  OpenSent   ├──►  Hold Time ตกลงกันได้ ฯลฯ)
                          └─────────────┘
                                 │
                                 ▼
                          ┌─────────────┐   แลก KEEPALIVE กัน
                          │ OpenConfirm ├──────────────────────┐
                          └─────────────┘                       │
                                                                 ▼
                                                         ┌──────────────┐
                                                         │ Established   │ ◄── พร้อมแลก UPDATE
                                                         └──────────────┘      (Route) จริง
```

| State | ความหมาย |
|---|---|
| **Idle** | สถานะเริ่มต้น — ยังไม่พยายาม Connect (หรือเพิ่ง Reset) |
| **Connect** | กำลังพยายามสร้าง TCP 3-way Handshake ไปยัง Port 179 ของ Peer |
| **Active** | TCP Connect ครั้งแรกล้มเหลว กำลัง**พยายามใหม่**อยู่เรื่อยๆ — **ถ้าค้างอยู่ State นี้นานผิดปกติ = สัญญาณปัญหา** (ไม่มี Route ถึง Peer, ACL บล็อก TCP 179) |
| **OpenSent** | TCP สำเร็จแล้ว ส่ง OPEN Message ไปแล้ว รอ OPEN Message ตอบกลับ |
| **OpenConfirm** | ได้รับ OPEN Message จาก Peer และตรวจสอบผ่านแล้ว (ASN, Hold Time ที่ระบุถูกต้อง) กำลังรอ KEEPALIVE ยืนยัน |
| **Established** | **สำเร็จสมบูรณ์** — เริ่มแลก UPDATE Message (Route จริง) ได้แล้ว นี่คือ State ที่ต้องเห็นใน `show ip bgp summary` |

### ตัวอย่างการดู State ด้วย `show ip bgp neighbors`

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 | include BGP state
  BGP state = Established, up for 00:05:12
```

จะกลับมาดู Verification แบบละเอียดใน **Step 289**

---

## Step 283 — คอนฟิก eBGP พื้นฐาน

### องค์ประกอบหลัก 3 ส่วนของการคอนฟิก eBGP

| คำสั่ง | หน้าที่ |
|---|---|
| `router bgp <local-AS>` | เปิด BGP Process และประกาศ ASN ของตัวเอง — **ต่างจาก OSPF ที่เปิดหลาย Process ได้ (`router ospf 1`, `router ospf 2`) BGP เปิดได้แค่ Process เดียวต่อ Router เท่านั้น (1 ASN ต่อ 1 Router)** |
| `neighbor <ip> remote-as <AS>` | ประกาศ Neighbor ที่จะสร้าง Session ด้วย พร้อมระบุ ASN ของ Neighbor นั้น — ถ้า ASN ที่ระบุ**ต่างจาก**ของตัวเอง = เป็น eBGP, ถ้า**เหมือนกัน** = เป็น iBGP โดยอัตโนมัติ (BGP ตัดสินจากค่านี้ ไม่ต้องบอกแยกว่า eBGP/iBGP) |
| `network <network> mask <mask>` | ประกาศ Prefix ที่จะโฆษณาออกไปให้ Neighbor รู้จัก |

### จุดสำคัญที่สุดของ `network` Statement: ต้อง Match แบบเป๊ะกับ Routing Table

นี่คือ**ความแตกต่างที่สำคัญที่สุด**ระหว่าง `network` ของ BGP กับ `network` ของ OSPF/EIGRP
ที่เราคุ้นเคย:

```
OSPF/EIGRP `network` statement:
   network 10.10.10.0 0.0.0.255 area 0
   → บอกว่า "Interface ไหนที่ IP อยู่ในช่วงนี้ ให้เข้าร่วม OSPF Area 0"
   → เป็นการ**เลือก Interface ที่จะรัน Protocol** ไม่ใช่การเลือก Prefix ที่จะประกาศโดยตรง
   → Subnet Mask ที่ใส่ไม่จำเป็นต้องตรงกับ Subnet Mask จริงของ Interface เป๊ะ (ใช้ Wildcard)

BGP `network` statement:
   network 203.0.113.0 mask 255.255.255.0
   → บอกว่า "ถ้ามี Route ที่ตรงกับ 203.0.113.0/24 แบบเป๊ะๆ (Exact Match ทั้ง Prefix และ Mask)
      อยู่ใน IP Routing Table (RIB) อยู่แล้ว (ไม่ว่าจะมาจาก Connected, Static, หรือ IGP)
      ให้ดึง Prefix นั้นเข้ามาประกาศผ่าน BGP"
   → **ไม่ได้เลือก Interface ที่จะรัน BGP** (BGP ไม่มีแนวคิด "Interface เข้าร่วม Protocol"
      เหมือน IGP เพราะ Neighbor ต้อง Config ด้วยมือเสมออยู่แล้ว)
   → ถ้าไม่มี Route ที่ Match แบบเป๊ะใน RIB — **จะไม่มีอะไรถูกประกาศเลย แม้จะพิมพ์ network
      statement ถูกต้องทุกตัวอักษร**
```

> **กับดักที่พบบ่อยที่สุดของ BGP มือใหม่**: พิมพ์ `network 203.0.113.0 mask 255.255.255.0`
> แล้วสงสัยว่าทำไม Neighbor ไม่เห็น Prefix นี้เลย — สาเหตุมักเป็นเพราะ Routing Table มี
> `203.0.113.0/30` (Connected, จาก Interface จริง) ไม่ใช่ `203.0.113.0/24` แบบที่ต้องการประกาศ
> ซึ่ง **Mask ไม่ตรงกัน = ไม่ Match = ไม่ประกาศ** ต้องมี Route ที่ Match แบบเป๊ะก่อนเสมอ
> (แก้ด้วย Static Route ไปยัง Null0 — จะสาธิตด้านล่าง)

### Worked Config: WAN-EDGE-1 ↔ ISP-RTR (eBGP พื้นฐาน)

**IP Addressing สำหรับ Step นี้ (ตาม [00-ip-address-plan.md](00-ip-address-plan.md)):**

| Device | Interface | IP Address | AS |
|---|---|---|---|
| WAN-EDGE-1 | GigabitEthernet0/0/0 | 203.0.113.2/30 | 65001 |
| ISP-RTR | GigabitEthernet0/0/0 | 203.0.113.1/30 | 65000 |

**บน WAN-EDGE-1:**

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** eBGP Link to ISP-RTR (AS 65000) - Primary Internet **
WAN-EDGE-1(config-if)# ip address 203.0.113.2 255.255.255.252
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
! Static Route แบบ Discard (Null0) เพื่อสร้าง Exact-match Route ให้ network statement อ้างอิงได้
! (Prefix นี้คือ Public IP Block ขององค์กร — จะอธิบายที่มาแบบเต็มใน Step 285)
WAN-EDGE-1(config)# ip route 203.0.113.0 255.255.255.0 Null0 254
!
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp router-id 1.1.1.21
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 description ** eBGP to ISP-RTR (Primary Internet) **
WAN-EDGE-1(config-router)# network 203.0.113.0 mask 255.255.255.0
WAN-EDGE-1(config-router)# exit
```

**บน ISP-RTR:**

```
ISP-RTR(config)# interface GigabitEthernet0/0/0
ISP-RTR(config-if)# description ** eBGP Link to WAN-EDGE-1 (Customer AS 65001) **
ISP-RTR(config-if)# ip address 203.0.113.1 255.255.255.252
ISP-RTR(config-if)# no shutdown
ISP-RTR(config-if)# exit
!
ISP-RTR(config)# router bgp 65000
ISP-RTR(config-router)# bgp router-id 1.1.1.254
ISP-RTR(config-router)# neighbor 203.0.113.2 remote-as 65001
ISP-RTR(config-router)# neighbor 203.0.113.2 description ** eBGP to WAN-EDGE-1 (Customer) **
ISP-RTR(config-router)# exit
```

> **หมายเหตุ `ip route ... Null0 254`**: Administrative Distance = 254 (สูงมาก) ทำให้ Route นี้
> **จะไม่ถูกใช้จริงในการ Forward Traffic** ถ้ามี Route ที่เฉพาะเจาะจงกว่าหรือ AD ต่ำกว่าอยู่แล้ว
> (เช่น Connected Route ของ Subnet ย่อยจริง) แต่ยังคง**ปรากฏใน RIB** พอให้ `network` statement
> เจอ Exact Match ได้เสมอ — เป็นเทคนิคมาตรฐานที่ใช้กันทั่วไปในการประกาศ Aggregate/Summary Block
> เข้า BGP (จะขยายความเพิ่มเติมใน Step 285)

### Verify: `show ip bgp summary` ต้องเห็น State = Established

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
BGP table version is 3, main routing table version 3
1 network entries using 232 bytes of memory

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000      12      11        3    0    0 00:03:45        0

WAN-EDGE-1# show ip route bgp
     (ยังไม่มี Route จาก BGP เพราะ ISP-RTR ยังไม่ได้ advertise อะไรกลับมา — ปกติในขั้นนี้
      เพราะเรายังไม่ได้ทำ Step 288 ให้ ISP-RTR ส่ง Default Route กลับมา)
```

- คอลัมน์ `State/PfxRcd = 0` หมายความว่า Session **Established แล้ว** (ถ้ายังไม่ Established
  จะโชว์ State เป็นคำ เช่น `Active`, `Idle`, `Connect` แทนตัวเลข) แต่ยังไม่ได้รับ Prefix ใดๆ
  จาก Neighbor เพราะ ISP-RTR ยังไม่ได้ Config `network`/`default-originate`

---

## Step 284 — ภาพรวม BGP Path Attributes

BGP ตัดสินใจเลือก "Best Path" จาก **Path Attribute** ที่แนบมากับทุก Prefix ใน UPDATE Message
— Attribute เหล่านี้คือหัวใจของความสามารถ "Policy-Based Routing" ที่ทำให้ BGP ต่างจาก IGP
โดยสิ้นเชิง Step นี้แนะนำภาพรวมสั้นๆ เท่านั้น **อัลกอริทึม Best Path แบบเต็ม 13 ขั้นตอนจะสอนใน
Part 30**

| Attribute | ประเภท | ความหมาย | ทิศทางที่มีผล |
|---|---|---|---|
| **WEIGHT** | Cisco Proprietary (Local เท่านั้น ไม่ส่งต่อ) | ค่าที่ตั้งบน Router ตัวเอง เพื่อเลือก Path ที่ต้องการใช้ "ออก" (Outbound) — ค่ายิ่งสูงยิ่งดี, Default = 0 (32768 สำหรับ Route ที่ Router สร้างเอง) | มีผลแค่บน Router ตัวเองเท่านั้น ไม่แชร์กับใคร |
| **LOCAL_PREF** | Well-known Discretionary | คล้าย WEIGHT แต่**แชร์กันภายใน AS เดียวกันผ่าน iBGP** — ใช้บอกทุก Router ใน AS ว่า Path ไหนควรเป็น Path ขาออก (Outbound) หลักของทั้งองค์กร ค่ายิ่งสูงยิ่งดี, Default = 100 | ส่งผ่าน iBGP ภายใน AS เท่านั้น (ไม่ส่งออกไป AS อื่นผ่าน eBGP) |
| **AS_PATH** | Well-known Mandatory | รายการ AS ที่ Prefix เดินทางผ่านมา — ใช้ป้องกัน Loop และเป็นตัวชี้วัด "ระยะทาง" หลักของ BGP (AS_PATH สั้นกว่า = มักถูกเลือกก่อน, เป็นค่าที่ Admin แก้ไขได้ผ่าน Prepending เพื่อทำ Traffic Engineering ขาเข้า) | ส่งออกทุกทิศทาง (เพิ่ม ASN ตัวเองทุกครั้งที่ส่งผ่าน eBGP) |
| **ORIGIN** | Well-known Mandatory | บอกที่มาของ Prefix: `IGP` (มาจาก `network` statement), `EGP` (แทบไม่ใช้แล้วในปัจจุบัน), `Incomplete` (มาจาก `redistribute`) — ใช้เป็น Tie-breaker ลำดับท้ายๆ | ส่งออกทุกทิศทาง |
| **MED** (Multi-Exit Discriminator) | Optional Non-transitive | บอก AS ข้างนอกว่า "ถ้ามีหลาย Entry Point เข้ามาที่ AS ของเรา ให้เลือก Entry Point นี้" (ค่ายิ่งต่ำยิ่งดี) — เหมือน "คำแนะนำ" ที่ AS อื่นไม่จำเป็นต้องเชื่อก็ได้ | ส่งไปยัง AS ข้างเคียงโดยตรงเท่านั้น (Non-transitive — AS ถัดไปจะไม่ส่งต่อค่านี้อีก) |
| **NEXT_HOP** | Well-known Mandatory | IP Address ของ Router ถัดไปที่ต้องส่ง Traffic ไปเพื่อไปถึง Prefix นี้ — สำคัญมากใน iBGP (ค่า Default ไม่เปลี่ยนเมื่อส่งผ่าน iBGP ตามที่กล่าวใน Step 282) | เปลี่ยนเป็น IP ตัวเองเสมอเมื่อส่งผ่าน eBGP |

### ตัวอย่างการดู Attribute จริงด้วย `show ip bgp`

```
WAN-EDGE-1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 4
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     1
  65000
    203.0.113.1 from 203.0.113.1 (1.1.1.254)
      Origin IGP, localpref 100, valid, external, best
```

- `65000` = AS_PATH (แค่ 1 AS เพราะเป็น Prefix ที่ ISP-RTR สร้างขึ้นเอง)
- `203.0.113.1` = NEXT_HOP
- `Origin IGP` = มาจาก `network` statement ฝั่ง ISP-RTR
- `localpref 100` = ค่า Default (ยังไม่ได้ปรับแต่ง)
- `best` = Path นี้ถูกเลือกเป็น Best Path แล้ว (มีทางเดียวในตัวอย่างนี้จึงชนะอัตโนมัติ)

---

## Step 285 — การประกาศ Route เข้า BGP: `network` vs `redistribute` vs `aggregate-address`

มี 3 วิธีหลักในการทำให้ Prefix หนึ่งๆ **เข้าสู่ BGP Table** เพื่อประกาศให้ Neighbor รู้จัก
แต่ละวิธีมี Trade-off ต่างกันชัดเจน:

| วิธี | หลักการ | ข้อดี | ข้อเสีย | เหมาะกับ |
|---|---|---|---|---|
| **`network` statement** | ดึง Route ที่ Match แบบเป๊ะจาก RIB (Static/Connected/IGP) เข้า BGP ทีละ Prefix | **ควบคุมได้เป๊ะที่สุด** — รู้แน่ชัดว่าอะไรถูกประกาศบ้าง, เสถียรที่สุด (ไม่ผูกกับการเปลี่ยนแปลงของ IGP ภายใน) | ต้องพิมพ์ทีละ Prefix, ต้องมี Route Match แบบเป๊ะใน RIB ก่อนเสมอ (มักต้องพึ่ง Static Null0) | **WAN Edge ขององค์กรทั่วไป (Enterprise)** ที่รู้ชัดเจนว่ามี Public Block อะไรบ้างที่ต้องการประกาศ — **วิธีที่แนะนำที่สุดสำหรับ Lab นี้** |
| **`redistribute ospf/eigrp/static`** | ดึง**ทุก** Route จาก Routing Protocol อื่นเข้า BGP อัตโนมัติ | เร็ว, ไม่ต้องพิมพ์ทีละ Prefix, Route ใหม่ที่เกิดใน IGP จะเข้า BGP อัตโนมัติ | **อันตรายมากถ้าใช้ไม่ระวัง**: (1) เสี่ยง Redistribute Private Address (RFC 1918 เช่น VLAN 10-99 ของเรา) ออกสู่ Internet จริงโดยไม่ตั้งใจ (2) ทุกครั้งที่ IGP Flap (Link ล่มแม้แค่วินาทีเดียว) จะทำให้ BGP ต้อง Re-advertise/Withdraw ตาม กระทบเสถียรภาพของ BGP ทั้ง AS | Transit AS/ISP จริงที่ต้องการ Automate การ Advertise Customer Route จำนวนมาก (ไม่เหมาะกับ Enterprise Edge ทั่วไป) |
| **`aggregate-address <net> <mask> [summary-only]`** | สร้าง Prefix แบบ**สรุปรวม (Summary)** จาก Prefix ย่อยที่มีอยู่แล้วใน BGP Table (ต้องมีอย่างน้อย 1 Prefix ย่อยที่เป็น Contributing Route) | ลดจำนวน Prefix ที่ประกาศออกไป (สำคัญมากสำหรับผู้ที่ต้องประกาศ Prefix จำนวนมาก), เสถียรกว่า (Flag `summary-only` ซ่อน Prefix ย่อยไม่ให้กระทบ Neighbor เมื่อย่อย Flap) | ต้องมี Contributing Route อยู่แล้วก่อนเสมอ (ไม่สามารถสร้าง Prefix ที่ไม่มีอยู่จริงได้) | องค์กรที่มี**หลาย Public Block ย่อย** ต้องการรวมเป็น Prefix เดียวก่อนประกาศออก Internet |

### Worked Example: ประกาศ Public Block ขององค์กร (203.0.113.0/24) ด้วย `network` (วิธีหลักของ Lab นี้)

องค์กรของเราได้รับจัดสรร **Public IP Block `203.0.113.0/24`** จาก ISP (สมมติว่าเป็น
Provider-Independent Block ที่ ISP มอบให้ ซึ่งครอบคลุมทั้ง Transit Link `203.0.113.0/30`,
`203.0.113.4/30` และ Address ส่วนที่เหลือสำหรับ NAT Pool/Public Server ภายใน) — วิธีประกาศ
ที่ทำไปแล้วใน Step 283 คือ:

```
WAN-EDGE-1(config)# ip route 203.0.113.0 255.255.255.0 Null0 254
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# network 203.0.113.0 mask 255.255.255.0
```

นี่คือรูปแบบที่**แนะนำที่สุด**สำหรับ WAN Edge ของ Enterprise เพราะประกาศเฉพาะ Block สาธารณะ
ที่ต้องการเผยแพร่จริงเท่านั้น — **ไม่มีทางที่ Private Subnet ภายใน (VLAN 10/20/30/40/99 ตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) จะหลุดออกไปสู่ Internet โดยไม่ตั้งใจ** เพราะ
เราไม่เคย `redistribute` อะไรเข้า BGP เลย

### Worked Example: `aggregate-address` เมื่อมีหลาย Block ย่อย (สถานการณ์สมมติเพื่อเปรียบเทียบ)

สมมติว่าองค์กรมี Public Block ย่อยอยู่ 2 ก้อนที่ต้องประกาศแยกกันไว้ก่อน:

```
WAN-EDGE-1(config)# ip route 203.0.113.0 255.255.255.128 Null0 254
WAN-EDGE-1(config)# ip route 203.0.113.128 255.255.255.128 Null0 254
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# network 203.0.113.0 mask 255.255.255.128
WAN-EDGE-1(config-router)# network 203.0.113.128 mask 255.255.255.128
WAN-EDGE-1(config-router)# aggregate-address 203.0.113.0 255.255.255.0 summary-only
WAN-EDGE-1(config-router)# exit
```

```
WAN-EDGE-1# show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 203.0.113.0/24   0.0.0.0                  0         32768 i     <- Aggregate (ประกาศออกจริง)
s> 203.0.113.0/25   0.0.0.0                  0         32768 i     <- ถูก Suppress (s) เพราะมี summary-only
s> 203.0.113.128/25 0.0.0.0                  0         32768 i     <- ถูก Suppress (s) เช่นกัน
```

- สถานะ `s>` (Suppressed) หมายความว่า Prefix ย่อยยังอยู่ใน BGP Table ภายใน Router แต่**จะไม่ถูก
  ส่งออกไปให้ Neighbor** — Neighbor จะเห็นแค่ `203.0.113.0/24` เพียง Entry เดียว
- ถ้าไม่ใส่ `summary-only` (ใช้แค่ `aggregate-address 203.0.113.0 255.255.255.0` เฉยๆ) —
  Neighbor จะเห็น**ทั้ง 3 Prefix พร้อมกัน** (Aggregate + ย่อยทั้งสอง) ซึ่งมักไม่ใช่สิ่งที่ต้องการ

> **สรุปสำหรับ Lab นี้**: Step 290 (Lab เต็มรูปแบบ) จะใช้วิธี `network` statement ตรงไปตรงมา
> ตาม Step 283 เพราะ WAN-EDGE-1/2 มีเพียง Public Block เดียว (`203.0.113.0/24`) ไม่จำเป็นต้อง
> ใช้ `aggregate-address` — แต่เข้าใจไว้เพราะจะพบบ่อยมากในองค์กรที่ซับซ้อนกว่าและในข้อสอบ CCNP

---

## Step 286 — BGP Route Filtering พื้นฐาน

การประกาศ Route แบบไม่มีการกรองเลยเป็นความเสี่ยงร้ายแรงบน Internet Edge — Step นี้แนะนำเครื่องมือ
กรอง Route 2 ตัวหลักที่ใช้กับ eBGP: **Prefix-list** และ **AS-Path Access-list**

### Prefix-list — กรองตาม Network/Prefix Length

```
ip prefix-list PL-TO-ISP seq 5 permit 203.0.113.0/24
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| `seq 5` | ลำดับการตรวจสอบ (เหมือน ACL แต่แก้ไข/แทรกลำดับใหม่ได้ง่ายกว่ามากโดยไม่ต้องลบทั้งหมด) |
| `permit`/`deny` | อนุญาต/ปฏิเสธ Prefix ที่ตรงเงื่อนไข |
| `203.0.113.0/24` | Prefix หลักที่จะ Match — ถ้าไม่ใส่ `ge`/`le` จะ Match แบบ**เป๊ะ**เท่านั้น (/24 พอดี) |
| `ge <n>` | Match Prefix ที่มี Prefix Length **มากกว่าหรือเท่ากับ** ค่านี้ (Subnet ที่เล็กกว่า/เฉพาะเจาะจงกว่า) |
| `le <n>` | Match Prefix ที่มี Prefix Length **น้อยกว่าหรือเท่ากับ** ค่านี้ |

```
! ตัวอย่าง: อนุญาตเฉพาะ Prefix ที่เฉพาะเจาะจงกว่า /24 แต่ไม่เกิน /32 ภายใน Block นี้
ip prefix-list PL-EXAMPLE seq 10 permit 203.0.113.0/24 ge 25 le 32

! ตัวอย่าง: Match เฉพาะ Default Route (0.0.0.0/0) แบบเป๊ะเท่านั้น
ip prefix-list PL-DEFAULT-ONLY seq 5 permit 0.0.0.0/0
```

**Prefix-list ดีกว่า `distribute-list` (ที่อ้างอิง Standard/Extended ACL) เพราะ**: (1) ประมวลผล
เร็วกว่า (ออกแบบมาสำหรับ Route Filtering โดยเฉพาะ ไม่ใช่ Packet Filtering แบบ ACL) (2) แก้ไข
ลำดับ (`seq`) ได้โดยไม่ต้องลบของเดิมทั้งหมด (3) รองรับ `ge`/`le` ทำให้กรองตาม Prefix Length
ได้ยืดหยุ่นกว่า ACL Wildcard Mask มาก

### การนำ Prefix-list มาใช้กับ Neighbor: `in` vs `out`

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 prefix-list PL-TO-ISP out
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 prefix-list PL-DEFAULT-ONLY in
```

| ทิศทาง | ความหมาย |
|---|---|
| `out` | กรอง Prefix **ก่อนส่งออก**ไปให้ Neighbor — ในตัวอย่างนี้ WAN-EDGE-1 จะส่งแค่ `203.0.113.0/24` ให้ ISP-RTR เท่านั้น แม้จะมี Route อื่นใน BGP Table |
| `in` | กรอง Prefix **ก่อนรับเข้า**จาก Neighbor — ในตัวอย่างนี้ WAN-EDGE-1 จะรับแค่ `0.0.0.0/0` (Default Route) จาก ISP-RTR เท่านั้น แม้ ISP-RTR จะพยายามส่ง Full Table มาให้ก็ตาม (เชื่อมโยงกับ Step 288) |

### AS-Path Access-list — กรองตาม AS_PATH ด้วย Regular Expression

```
ip as-path access-list 1 permit ^65000$
```

- `^65000$` คือ Regular Expression หมายถึง "AS_PATH ที่มี **เฉพาะ** AS 65000 ตัวเดียว" (`^` =
  จุดเริ่มต้น, `$` = จุดสิ้นสุด) — ใช้กรอง Prefix ที่ ISP-RTR **สร้างขึ้นเอง**เท่านั้น ปฏิเสธ
  Prefix ใดๆ ที่ ISP-RTR เรียนรู้ต่อมาจาก AS อื่นแล้วส่งต่อให้เรา (ป้องกันไม่ให้เรากลายเป็น
  Transit AS โดยไม่ตั้งใจ)
- `.*` = Match ตัวอักษร/ASN ใดๆ ก็ได้ (เช่น `_65000_` หมายถึง "มี AS 65000 อยู่ที่ไหนก็ได้ใน Path")

**การนำไปใช้กับ Neighbor:**

```
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 filter-list 1 in
```

> **ในทางปฏิบัติของ Lab นี้**: เนื่องจาก ISP-RTR เป็น "Stub" ที่จำลอง ISP เพียงรายเดียวและไม่มี
> AS อื่นต่อพ่วง (ไม่ใช่ Transit จริง) จึง AS-Path Filter จะยังไม่เห็นผลต่างชัดเจนในการ Lab
> แต่เป็นเทคนิคสำคัญมากในโลกจริงที่ทุก Enterprise Edge ควรมี Filter ทั้ง 2 ประเภทนี้ติดตั้งไว้
> เสมอ (Defense-in-depth ระดับ Routing Policy)

---

## Step 287 — eBGP Multihop และ Load-Sharing ข้าม 2 ISP Link

### eBGP Multihop — เมื่อ Neighbor ไม่ได้ต่อกันโดยตรง

Default ของ eBGP คือ TTL = 1 (สมมติว่า Neighbor อยู่ **ห่างกันแค่ 1 Hop** เชื่อมต่อโดยตรง)
ถ้า Packet ต้องเดินทางผ่านมากกว่า 1 Hop กว่าจะถึง Neighbor (เช่น Peering ผ่าน Loopback Address
แทน Physical Interface เพื่อความทนทานต่อ Link ที่ล่ม) ต้องเปิดใช้ **eBGP Multihop** ด้วยตนเอง:

```
WAN-EDGE-1(config-router)# neighbor 1.1.1.254 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 1.1.1.254 ebgp-multihop 2
WAN-EDGE-1(config-router)# neighbor 1.1.1.254 update-source Loopback0
! ต้องมี Static/IGP Route ไปยัง Loopback ของ Neighbor ด้วยเสมอ ไม่เช่นนั้นจะค้างที่ State Active
```

- `ebgp-multihop <ttl>` เพิ่มค่า TTL ที่ยอมรับได้ (ในตัวอย่างนี้ 2 Hop) — ต้องระบุให้เพียงพอกับ
  จำนวน Hop จริงระหว่าง Loopback ทั้งสองฝั่ง
- **Lab หลักของหลักสูตรนี้ไม่จำเป็นต้องใช้ eBGP Multihop** เพราะ WAN-EDGE-1/2 ต่อกับ ISP-RTR
  โดยตรงบน Physical Interface (Directly Connected) อยู่แล้ว — แต่ต้องเข้าใจไว้เพราะเป็นหัวข้อ
  ที่ข้อสอบ CCNP ENCOR ถามบ่อยมาก โดยเฉพาะเมื่อโจทย์อธิบาย Scenario Peering ผ่าน Loopback

### Load-Sharing ข้าม 2 ISP Link ด้วย `maximum-paths eibgp`

```
WAN-EDGE-1(config-router)# maximum-paths eibgp 2
```

- คำสั่งนี้อนุญาตให้ BGP ติดตั้ง**หลาย Path เข้า RIB พร้อมกัน (ECMP)** เมื่อ Path เหล่านั้นมี
  Attribute สำคัญเท่ากันทุกตัว (Weight, Local-Pref, AS-Path Length, Origin, MED) — Default
  ของ eBGP คือ `1` (ไม่มี Load-Sharing เลย แม้จะมีหลาย Path ที่เท่ากันทุกประการ)
- **`maximum-paths eibgp` มีประโยชน์สูงสุดเมื่อ Router ตัวเดียวมีหลาย eBGP Path ไปหา AS
  เดียวกัน** เช่น ถ้า WAN-EDGE-1 มี 2 Physical Link ไปยัง ISP-RTR โดยตรง (ไม่ใช่ Scenario ของ
  เรา) คำสั่งนี้จะทำให้ Traffic กระจายข้ามทั้ง 2 Link ได้ทันที

### ทำไม Dual-Router Design (WAN-EDGE-1 + WAN-EDGE-2) จึงต้องพึ่ง iBGP

Lab ของเราคือ**คนละ Router** (WAN-EDGE-1 กับ WAN-EDGE-2) ต่อ ISP-RTR คนละ Link — สถานการณ์นี้
`maximum-paths eibgp` **ช่วยไม่ได้โดยตรง** เพราะแต่ละ Router รู้จักแค่ Path ของตัวเองเท่านั้น
(WAN-EDGE-1 ไม่รู้ด้วยซ้ำว่า WAN-EDGE-2 มี Path ไปยัง ISP-RTR เช่นกัน) วิธีแก้คือสร้าง **iBGP
Session ผ่าน Cross-link** (`10.10.254.8/30` ตาม [00-ip-address-plan.md](00-ip-address-plan.md))
เพื่อให้ทั้งสอง Router **แชร์ข้อมูล BGP ระหว่างกัน**:

```
! บน WAN-EDGE-1 (สมมติว่า Loopback0 ของ WAN-EDGE-1/2 เห็นกันผ่าน OSPF Area 0 อยู่แล้ว
! ตามที่ออกแบบไว้ตั้งแต่ Part 11-12 เพราะ WAN-EDGE-1/2 อยู่ใน Backbone เดียวกับ CORE-SW1/2)
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 description ** iBGP to WAN-EDGE-2 (Backup Internet Path) **
WAN-EDGE-1(config-router)# exit

! บน WAN-EDGE-2 (Config กลับด้าน)
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 remote-as 65001
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 update-source Loopback0
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 description ** iBGP to WAN-EDGE-1 (Backup Internet Path) **
WAN-EDGE-2(config-router)# exit
```

> **หมายเหตุ**: `remote-as` เท่ากับ ASN ของตัวเอง (65001 ทั้งคู่) — BGP จะรู้ทันทีว่านี่คือ
> iBGP Session (ไม่ใช่ eBGP) จากค่านี้ ตามที่อธิบายไว้ใน Step 282/283 ส่วนเหตุผลที่ใช้
> `update-source Loopback0` แทน Physical Interface ของ Cross-link โดยตรง เพราะ Loopback
> ไม่มีวันล่ม (Always-up) ทำให้ iBGP Session ทนทานกว่า แม้ Path ทางกายภาพจะเปลี่ยนไปตาม IGP
> ก็ตาม (รายละเอียดกลไกนี้จะลงลึกอีกครั้งใน Part 30)

### ทำไม Design นี้ทำงานเป็น Primary/Backup โดยอัตโนมัติ (ไม่ต้องตั้งค่าเพิ่ม)

เมื่อทั้งสอง WAN-EDGE เห็น Default Route (`0.0.0.0/0`) จาก ISP-RTR ผ่านทั้ง eBGP (ของตัวเอง)
**และ** iBGP (Relay มาจากอีกฝั่ง) พร้อมกัน BGP Best Path Algorithm มีกฎ Tie-breaker ข้อหนึ่งที่
สำคัญมาก (จะลงรายละเอียดครบทุกขั้นตอนใน Part 30):

> **"เมื่อ Attribute สำคัญอื่นๆ เท่ากันหมด (Weight, Local-Pref, AS-Path Length, Origin, MED)
> ให้เลือก Path ที่เรียนรู้ผ่าน eBGP เสมอ ก่อน Path ที่เรียนรู้ผ่าน iBGP"**

```
สถานการณ์ปกติ (ทั้ง 2 ISP Link ใช้งานได้):
   WAN-EDGE-1 เห็น 0.0.0.0/0 จาก 2 ทาง:
     (1) eBGP โดยตรงจาก ISP-RTR (AS_PATH = [65000])         ← ชนะ (เลือกใช้จริง)
     (2) iBGP Relay จาก WAN-EDGE-2 (AS_PATH = [65000] เท่ากัน) ← แพ้ (สำรองไว้เฉยๆ)

สถานการณ์ Link หลักล่ม (eBGP Session WAN-EDGE-1 ↔ ISP-RTR down):
   WAN-EDGE-1 เห็น 0.0.0.0/0 เหลือแค่ทางเดียว:
     (2) iBGP Relay จาก WAN-EDGE-2                          ← กลายเป็น Best Path ทันที (Failover อัตโนมัติ)
```

นี่คือเหตุผลที่ **Dual-Homed Internet Design ระดับ Enterprise แทบทุกที่ต้องมี iBGP ระหว่าง
Border Router ทั้งสองตัวเสมอ** — ไม่ใช่แค่เพื่อ "ทำให้ครบ" แต่เป็นกลไก Failover หลักที่ทำงาน
โดยอัตโนมัติผ่าน BGP Attribute ล้วนๆ ไม่ต้องพึ่ง Tracking Script หรือ Routing Protocol อื่นเลย

---

## Step 288 — BGP กับ Default Route: Full Table vs Default-Only

### ทำไม Enterprise ทั่วไปไม่ควรรับ Full Internet Table

| Design | จำนวน Prefix ที่ต้องเก็บใน BGP Table | RAM/CPU ที่ต้องการ | ใช้เมื่อไหร่ |
|---|---|---|---|
| **Default Route เท่านั้น** | 1 Prefix (`0.0.0.0/0`) | ต่ำมาก | **Enterprise ทั่วไปที่มี 1-2 ISP** (Stub AS ไม่ทำ Transit ให้ใคร) — คือ Design ของ WAN-EDGE-1/2 ในหลักสูตรนี้ |
| **Full Table** | 900,000+ Prefix (IPv4, เพิ่มขึ้นทุกปี) + อีกจำนวนมากสำหรับ IPv6 | สูงมาก (ต้องเป็น Router ระดับ Enterprise/ISP-grade RAM หลาย GB เฉพาะสำหรับ BGP) | **ISP หรือ Transit AS** ที่ต้องตัดสินใจเลือกเส้นทางที่ดีที่สุดไปยังทุกมุมโลกด้วยตัวเอง (ไม่พึ่ง Default อย่างเดียว) |
| **Partial Table** (เช่น รับเฉพาะ Route ของลูกค้า ISP รายนั้น + Default) | หลักพัน-หมื่น Prefix | ปานกลาง | Enterprise ขนาดใหญ่ที่ทำ Traffic Engineering ละเอียดขึ้น แต่ยังไม่ใช่ ISP เต็มรูปแบบ |

> **หลักการสำคัญสำหรับ CCNP ENCOR**: องค์กรที่**ไม่ได้ทำ Transit ให้ AS อื่น** (คือไม่ได้ส่งต่อ
> Traffic ระหว่าง AS ที่สามให้ใคร) เรียกว่า **Stub AS** — Stub AS **ไม่มีความจำเป็นต้องรับ Full
> Table เลย** เพราะไม่ว่า Full Table จะบอกเส้นทางไปยังปลายทางไหนในโลก ทางออกของ Traffic จาก
> Stub AS ก็มีแค่ทาง ISP ของตัวเองเท่านั้นอยู่ดี — **Default Route ให้ผลลัพธ์การ Forward Traffic
> เหมือนกันทุกประการ** โดยใช้ทรัพยากรน้อยกว่ามหาศาล

### วิธีรับ Default Route จาก ISP — 2 แนวทาง

**แนวทาง 1: Static Default Route (ไม่ผ่าน BGP เลย)**

```
WAN-EDGE-1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

- ง่ายที่สุด เหมาะกับ **Single-Homed** (ต่อ ISP เดียว ไม่มี Backup Path) เพราะไม่ต้องพึ่ง BGP
  เลยด้วยซ้ำสำหรับงานนี้
- **ข้อเสียสำคัญ**: ถ้า ISP Link ล่ม Static Route ยังคง "มีชีวิต" อยู่ใน RIB ต่อไป (Next-hop
  `203.0.113.1` ยังคง Resolve ได้ตราบใดที่ Interface ยัง Up/Up ทาง Layer ล่าง) ทำให้ Router
  ยังพยายามส่ง Traffic ไปทางนี้ทั้งที่ปลายทางจริงตายไปแล้ว (Black-hole) — ต้องพึ่ง IP SLA
  Tracking เพิ่มเพื่อแก้ปัญหานี้ (ซับซ้อนกว่า และไม่ใช่ขอบเขตของ Part นี้)

**แนวทาง 2: `neighbor default-originate` (แนะนำสำหรับ Design Dual-Homed แบบ Lab นี้)**

```
! Config บน ISP-RTR (ฝั่งที่ "ให้" Default Route)
ISP-RTR(config)# router bgp 65000
ISP-RTR(config-router)# neighbor 203.0.113.2 default-originate
ISP-RTR(config-router)# neighbor 203.0.113.6 default-originate
```

- `default-originate` สั่งให้ ISP-RTR **สร้าง Default Route (`0.0.0.0/0`) และประกาศออกไปให้
  Neighbor ที่ระบุ** โดยที่**ไม่จำเป็นต้องมี `0.0.0.0/0` อยู่ใน BGP Table ของ ISP-RTR เองเลย**
  (ต่างจาก `network 0.0.0.0 mask 0.0.0.0` ที่ต้องมี Default Route จริงใน RIB ก่อน)
- **ข้อดีที่สำคัญที่สุด**: เพราะส่งผ่าน BGP UPDATE Message จริง เมื่อ eBGP Session ระหว่าง
  ISP-RTR ↔ WAN-EDGE-1 ล่ม (Physical Link ขาด, Neighbor ตาย) BGP จะ **Withdraw Default Route
  นี้ออกจาก WAN-EDGE-1 โดยอัตโนมัติทันที** — WAN-EDGE-1 จะไม่มี Default Route ทางนี้เหลืออยู่
  เลย และ (ตาม Step 287) จะ Fail-over ไปใช้ Default ที่เรียนผ่าน iBGP จาก WAN-EDGE-2 แทนทันที
  **นี่คือข้อได้เปรียบหลักของการรับ Default ผ่าน BGP เหนือกว่า Static Route ธรรมดา**

### ตาราง สรุปการเลือก Design

| สถานการณ์ | คำแนะนำ |
|---|---|
| Single-Homed (ISP เดียว, Link เดียว) | Static Default Route พอเพียง (เรียบง่าย ไม่ต้องพึ่ง BGP) |
| **Dual-Homed (2 ISP Link เหมือนหลักสูตรนี้)** | **BGP + `default-originate`** — ใช้ประโยชน์จากกลไก Withdraw อัตโนมัติของ BGP เพื่อ Failover |
| องค์กรขนาดใหญ่ที่ต้องเลือก Exit Point ตาม Path ที่ดีที่สุดจริงๆ (ไม่ใช่แค่ Failover) | พิจารณา Partial/Full Table (ต้องประเมิน Hardware ให้เพียงพอ) |

---

## Step 289 — Verification และ Troubleshooting

### `show ip bgp summary` — ภาพรวม Session ทั้งหมด

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
BGP table version is 8, main routing table version 8
2 network entries using 464 bytes of memory

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000      55      54        8    0    0 00:20:10        1
1.1.1.22        4 65001      40      38        8    0    0 00:15:02        1
```

| คอลัมน์ | ความหมาย |
|---|---|
| `V` | BGP Version (4 เสมอในปัจจุบัน) |
| `AS` | ASN ของ Neighbor |
| `MsgRcvd`/`MsgSent` | จำนวน Message ที่รับ/ส่งแล้วตั้งแต่ Session เริ่ม |
| `TblVer` | BGP Table Version ล่าสุดที่ Sync กับ Neighbor แล้ว |
| `InQ`/`OutQ` | จำนวน Message ที่รอประมวลผลอยู่ในคิว (ควรเป็น 0 เกือบตลอดเวลา — ถ้าค้างสูงแสดงว่า Router ทำงานหนักเกินไป) |
| `Up/Down` | ระยะเวลาที่ Session อยู่ในสถานะปัจจุบัน |
| `State/PfxRcd` | **ถ้า Established แล้วจะโชว์เป็นตัวเลข = จำนวน Prefix ที่ได้รับจาก Neighbor นี้** — ถ้ายังไม่ Established จะโชว์เป็นคำอธิบาย State (Idle, Connect, Active, OpenSent, OpenConfirm) |

### `show ip bgp neighbors` — รายละเอียดเชิงลึกของแต่ละ Session

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1
BGP neighbor is 203.0.113.1,  remote AS 65000, external link
  Description: ** eBGP to ISP-RTR (Primary Internet) **
  BGP version 4, remote router ID 1.1.1.254
  BGP state = Established, up for 00:20:35
  Last read 00:00:12, last write 00:00:18, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    Route refresh: advertised and received
    Address family IPv4 Unicast: advertised and received
  Message statistics:
    InQ depth is 0, OutQ depth is 0
                         Sent       Rcvd
    Opens:                  1          1
    Updates:                3          4
    Keepalives:            51         51
    Total:                 55         56
  Connections established 1; dropped 0
  Local host: 203.0.113.2, Local port: 52341
  Foreign host: 203.0.113.1, Foreign port: 179
```

จุดที่ต้องดูเมื่อ Troubleshoot: `BGP state`, `Connections established` vs `dropped` (ถ้า
`dropped` สูงแปลว่า Session ไม่เสถียร หลุดๆ ติดๆ), `Foreign port: 179` (ยืนยันว่ากำลังคุยกัน
บน TCP 179 จริง)

### `show ip bgp` และ `show ip bgp <prefix>` — ดู BGP Table

```
WAN-EDGE-1# show ip bgp
BGP table version is 8, local router ID is 1.1.1.21
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 0.0.0.0          203.0.113.1              0             0 65000 i
*  0.0.0.0          1.1.1.22                 0    100      0 65000 i
*> 203.0.113.0/24   0.0.0.0                  0         32768 i
```

```
WAN-EDGE-1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 8
Paths: (2 available, best #1, table default)
  Advertised to update-groups:
     1
  65000
    203.0.113.1 from 203.0.113.1 (1.1.1.254)
      Origin IGP, metric 0, localpref 100, valid, external, best
  65000
    1.1.1.22 (metric 20) from 1.1.1.22 (1.1.1.22)
      Origin IGP, metric 0, localpref 100, valid, internal
```

สังเกต `external, best` บน Path แรก (เรียนจาก eBGP) และ `internal` บน Path ที่สอง (เรียนจาก
iBGP ผ่าน WAN-EDGE-2) — ยืนยันตามที่อธิบายใน Step 287 ว่า eBGP ชนะ iBGP เมื่อ Attribute อื่น
เท่ากัน

### ปัญหาที่พบบ่อยที่สุดเมื่อ BGP Neighbor ไม่ Establish

| อาการ | สาเหตุที่เป็นไปได้ | วิธีตรวจสอบ |
|---|---|---|
| **ค้างที่ State `Active`** | (1) ไม่มี Route ไปยัง Neighbor IP เลย (Interface Down, ผิด Subnet) (2) ACL/Firewall บล็อก TCP Port 179 ระหว่างทาง (3) `ebgp-multihop` ไม่ได้ตั้งค่าทั้งที่ Neighbor อยู่ไกลกว่า 1 Hop | `ping <neighbor-ip>`, `show ip route <neighbor-ip>`, ตรวจสอบ ACL ที่ Interface ระหว่างทาง, `show tcp brief` (ควรเห็น Session พยายาม SYN ซ้ำๆ) |
| **ค้างที่ State `Idle`** | ยังไม่มีการพยายาม Connect เลย — มักเกิดจาก Neighbor Statement ผิด IP, หรือ BGP Process เพิ่ง Restart/Reset (`clear ip bgp *`) | ตรวจสอบ `neighbor <ip> remote-as` ว่า IP ถูกต้อง, รอสักครู่ให้ Retry Timer ทำงาน |
| **เห็น `%BGP-3-NOTIFICATION` ใน Log แล้ว Session หลุดทันที** | **AS Mismatch** — ระบุ `remote-as` ผิดจากที่ Neighbor ใช้จริง (Error Code 2 = OPEN Message Error, มักเป็น Subcode "Bad Peer AS") | ตรวจสอบ ASN ที่ทั้งสองฝั่ง Config ตรงกับที่อีกฝั่งประกาศตัวเองจริงหรือไม่ (`show ip protocols` หรือ `show run \| section router bgp` ทั้งสองฝั่ง) |
| **Session ตกกะทันหันหลัง Established ไปแล้วพักหนึ่ง** | **Hold Timer Expired** — Firewall ตัด Keepalive ระหว่างทาง หรือ Link คุณภาพแย่ (Packet Loss สูง) | `debug ip bgp events` (ระวังใช้ใน Production — เพิ่ม CPU Load), ตรวจสอบ Physical Link Error Counter ด้วย `show interface` |
| **TCP ต่อสำเร็จแต่ OPEN Message ถูกปฏิเสธทันที** | **MD5 Authentication Mismatch** — ตั้ง `neighbor <ip> password <key>` ไม่ตรงกันทั้งสองฝั่ง | ตรวจสอบ Password ทั้งสองฝั่งให้ตรงกันเป๊ะ (Case-sensitive) |
| **Established แต่ `PfxRcd = 0`** | Neighbor ไม่ได้ Config `network`/`redistribute`/`default-originate` เลย หรือ Prefix ถูก Filter ออกด้วย Prefix-list/AS-Path ฝั่งตรงข้าม | ตรวจสอบ `show ip bgp neighbors <ip> advertised-routes` บนฝั่งที่ควรจะส่ง, ตรวจสอบ Filter ทุกจุด |

---

## Step 290 — Lab เต็มรูปแบบ: Dual-Homed eBGP Internet Edge

รวมทุกแนวคิดของ Part นี้เข้าด้วยกัน: **WAN-EDGE-1** และ **WAN-EDGE-2** (ทั้งคู่ AS 65001) แต่ละ
ตัวทำ eBGP กับ **ISP-RTR (AS 65000)** ผ่านคนละ Link, ประกาศ Public Block ขององค์กร
(`203.0.113.0/24`) ออกไป, รับ Default Route กลับมาผ่าน `default-originate`, และเชื่อมกันเอง
ด้วย **iBGP ผ่าน Cross-link** เพื่อ Redundancy อัตโนมัติ

```
                              ┌─────────────────────────────┐
                              │           ISP-RTR             │  AS 65000
                              │        Lo0: 1.1.1.254/32      │
                              │  bgp: network 0.0.0.0 (จำลอง  │
                              │  Internet ทั้งใบด้วย default) │
                              └──────┬─────────────────┬─────┘
                203.0.113.0/30 (.1)  │                 │  203.0.113.4/30 (.5)
                                      │                 │
                    ┌─────────────────┴───┐   ┌─────────┴─────────────────┐
                    │      WAN-EDGE-1       │   │       WAN-EDGE-2           │
                    │      AS 65001         │   │       AS 65001             │
                    │  Lo0: 1.1.1.21/32     │   │   Lo0: 1.1.1.22/32         │
                    │  eBGP → ISP-RTR       │   │   eBGP → ISP-RTR            │
                    │  advertise: 203.0.113.0/24 (ทั้งคู่ ผ่าน Null0 + network)│
                    └────────┬──────────────┘   └───────────────┬────────────┘
                             │         iBGP (Lo0-to-Lo0)         │
                             └───────────── 10.10.254.8/30 ──────┘
                             │                                    │
                    ┌────────┴──────────┐               ┌─────────┴──────────┐
                    │     CORE-SW1        │               │     CORE-SW2         │
                    └─────────────────────┘               └──────────────────────┘
```

### สรุป IP Addressing และ AS สำหรับ Lab นี้

| Device | Interface | IP Address | AS | Role |
|---|---|---|---|---|
| ISP-RTR | GigabitEthernet0/0/0 | 203.0.113.1/30 | 65000 | eBGP ไปยัง WAN-EDGE-1 |
| ISP-RTR | GigabitEthernet0/0/1 | 203.0.113.5/30 | 65000 | eBGP ไปยัง WAN-EDGE-2 |
| ISP-RTR | Loopback0 | 1.1.1.254/32 | 65000 | Router-ID |
| WAN-EDGE-1 | GigabitEthernet0/0/0 | 203.0.113.2/30 | 65001 | eBGP ไปยัง ISP-RTR (Primary) |
| WAN-EDGE-1 | GigabitEthernet0/0/2 | 10.10.254.9/30 | 65001 | iBGP Cross-link ไปยัง WAN-EDGE-2 |
| WAN-EDGE-1 | Loopback0 | 1.1.1.21/32 | 65001 | Router-ID / iBGP update-source |
| WAN-EDGE-2 | GigabitEthernet0/0/0 | 203.0.113.6/30 | 65001 | eBGP ไปยัง ISP-RTR (Secondary) |
| WAN-EDGE-2 | GigabitEthernet0/0/2 | 10.10.254.10/30 | 65001 | iBGP Cross-link ไปยัง WAN-EDGE-1 |
| WAN-EDGE-2 | Loopback0 | 1.1.1.22/32 | 65001 | Router-ID / iBGP update-source |

> **สมมติฐาน**: WAN-EDGE-1/WAN-EDGE-2/CORE-SW1/CORE-SW2 ทั้งหมดอยู่ใน **OSPF Area 0** เดียวกัน
> อยู่แล้วตามที่ออกแบบไว้ตั้งแต่ [Part 11-12](part-011-ospfv2-fundamentals.md) ทำให้ Loopback0
> ของ WAN-EDGE-1 และ WAN-EDGE-2 เห็นกันผ่าน IGP ก่อนที่ iBGP Session จะพยายาม Establish ได้
> (ข้อกำหนดพื้นฐานของ iBGP ผ่าน Loopback: ต้องมี IGP หรือ Static Route ไปถึง Loopback ของอีกฝั่งเสมอ)

### Running-Config เต็มรูปแบบ — ISP-RTR

```
hostname ISP-RTR
!
interface Loopback0
 description ** Router-ID **
 ip address 1.1.1.254 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** eBGP Link to WAN-EDGE-1 (Customer AS 65001, Primary) **
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** eBGP Link to WAN-EDGE-2 (Customer AS 65001, Secondary) **
 ip address 203.0.113.5 255.255.255.252
 no shutdown
!
router bgp 65000
 bgp router-id 1.1.1.254
 neighbor 203.0.113.2 remote-as 65001
 neighbor 203.0.113.2 description ** eBGP to WAN-EDGE-1 **
 neighbor 203.0.113.2 default-originate
 neighbor 203.0.113.6 remote-as 65001
 neighbor 203.0.113.6 description ** eBGP to WAN-EDGE-2 **
 neighbor 203.0.113.6 default-originate
!
end
```

### Running-Config เต็มรูปแบบ — WAN-EDGE-1

```
hostname WAN-EDGE-1
!
interface Loopback0
 description ** Router-ID / iBGP update-source **
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** eBGP Link to ISP-RTR (Primary Internet) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Cross-link to WAN-EDGE-2 (iBGP + IGP backup path) **
 ip address 10.10.254.9 255.255.255.252
 no shutdown
!
router ospf 1
 router-id 1.1.1.21
 network 1.1.1.21 0.0.0.0 area 0
 network 10.10.254.8 0.0.0.3 area 0
 network 10.10.254.0 0.0.0.3 area 0
!
ip route 203.0.113.0 255.255.255.0 Null0 254
!
ip prefix-list PL-TO-ISP seq 5 permit 203.0.113.0/24
ip prefix-list PL-DEFAULT-ONLY seq 5 permit 0.0.0.0/0
!
router bgp 65001
 bgp router-id 1.1.1.21
 neighbor 203.0.113.1 remote-as 65000
 neighbor 203.0.113.1 description ** eBGP to ISP-RTR (Primary Internet) **
 neighbor 203.0.113.1 prefix-list PL-TO-ISP out
 neighbor 203.0.113.1 prefix-list PL-DEFAULT-ONLY in
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 update-source Loopback0
 neighbor 1.1.1.22 description ** iBGP to WAN-EDGE-2 (Backup Internet Path) **
 network 203.0.113.0 mask 255.255.255.0
!
end
```

### Running-Config เต็มรูปแบบ — WAN-EDGE-2

```
hostname WAN-EDGE-2
!
interface Loopback0
 description ** Router-ID / iBGP update-source **
 ip address 1.1.1.22 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** eBGP Link to ISP-RTR (Secondary Internet) **
 ip address 203.0.113.6 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Cross-link to WAN-EDGE-1 (iBGP + IGP backup path) **
 ip address 10.10.254.10 255.255.255.252
 no shutdown
!
router ospf 1
 router-id 1.1.1.22
 network 1.1.1.22 0.0.0.0 area 0
 network 10.10.254.8 0.0.0.3 area 0
 network 10.10.254.4 0.0.0.3 area 0
!
ip route 203.0.113.0 255.255.255.0 Null0 254
!
ip prefix-list PL-TO-ISP seq 5 permit 203.0.113.0/24
ip prefix-list PL-DEFAULT-ONLY seq 5 permit 0.0.0.0/0
!
router bgp 65001
 bgp router-id 1.1.1.22
 neighbor 203.0.113.5 remote-as 65000
 neighbor 203.0.113.5 description ** eBGP to ISP-RTR (Secondary Internet) **
 neighbor 203.0.113.5 prefix-list PL-TO-ISP out
 neighbor 203.0.113.5 prefix-list PL-DEFAULT-ONLY in
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.21 description ** iBGP to WAN-EDGE-1 (Backup Internet Path) **
 network 203.0.113.0 mask 255.255.255.0
!
end
```

### Verification 1: `show ip bgp summary` ต้องเห็น Established ทุก Session

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.22        4 65001      22      21       10    0    0 00:08:40        2
203.0.113.1     4 65000      30      29       10    0    0 00:10:15        1

WAN-EDGE-2# show ip bgp summary
BGP router identifier 1.1.1.22, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.21        4 65001      21      22       10    0    0 00:08:40        2
203.0.113.5     4 65000      29      28       10    0    0 00:10:20        1

ISP-RTR# show ip bgp summary
BGP router identifier 1.1.1.254, local AS number 65000
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.2     4 65001      29      30        3    0    0 00:10:15        1
203.0.113.6     4 65001      28      29        3    0    0 00:10:20        1
```

**ทุก Session แสดง `State/PfxRcd` เป็นตัวเลข (ไม่ใช่คำ) = Established ครบทั้ง 4 Session**
(eBGP 2 เส้น + iBGP 1 เส้น นับที่ WAN-EDGE-1 และ WAN-EDGE-2 ฝั่งละ 2 Session)

### Verification 2: `show ip route` แสดง Route `B` (BGP) บน WAN-EDGE-1

```
WAN-EDGE-1# show ip route bgp
Codes: L - local, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

B*   0.0.0.0/0 [20/0] via 203.0.113.1, 00:10:15

WAN-EDGE-1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "bgp 65001", distance 20, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 203.0.113.1, from 203.0.113.1, 00:10:15 ago
      Route metric is 0, traffic share count is 1
```

- `B*` = Route มาจาก BGP และเป็น **Candidate Default** (`*` แปลว่าใช้เป็น Gateway of Last
  Resort ได้) — `[20/0]` คือ `[Administrative Distance/Metric]` โดย AD=20 คือค่าของ eBGP
  ตามที่กล่าวไว้ใน Step 282
- สังเกตว่า Route ที่ติดตั้งจริงมาจาก **eBGP โดยตรง** (`via 203.0.113.1`) ไม่ใช่จาก iBGP —
  ตรงตามที่คาดไว้ใน Step 287 (eBGP ชนะ iBGP เมื่อ Attribute อื่นเท่ากัน)

### Verification 3: ทดสอบ Failover — ปิด eBGP Link หลักของ WAN-EDGE-1

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# shutdown
WAN-EDGE-1(config-if)# end

WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.22        4 65001      25      24       11    0    0 00:12:05        2
203.0.113.1     4 65000       -       -        -    0    0 never    Idle

WAN-EDGE-1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "bgp 65001", distance 200, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 1.1.1.22 (Default-IP-Routing-Table), from 1.1.1.22, 00:00:03 ago
      Route metric is 0, traffic share count is 1

WAN-EDGE-1# ping 8.8.8.8 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.21
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 9/12/18 ms
```

- Neighbor `203.0.113.1` เปลี่ยนเป็น **`Idle`** ทันทีที่ Interface ล่ม (BGP รับรู้จาก Physical
  Link Down เร็วกว่ารอ Hold Timer หมดอายุ)
- Default Route เปลี่ยนมาใช้ `via 1.1.1.22` (**iBGP จาก WAN-EDGE-2**) โดยอัตโนมัติ — สังเกตว่า
  Distance เปลี่ยนจาก `20` เป็น `200` (AD ของ iBGP) แต่ **Traffic ยังคงไปถึงปลายทางได้ปกติ**
  (ผ่าน WAN-EDGE-2 ➜ ISP-RTR แทน) พิสูจน์ว่า **Design Dual-Homed eBGP + iBGP Cross-link ให้
  Automatic Failover ได้จริง** โดยไม่ต้องพึ่ง Static Route/Tracking ใดๆ เพิ่มเติมเลย

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# no shutdown
! (คืนสภาพ Link กลับมาก่อนไปต่อ Part 30)
```

---

## แบบฝึกหัดทวนความเข้าใจ Part 29

1. เพราะเหตุใด `network` statement ของ BGP จึงต้องการ Route ที่ Match แบบเป๊ะ (Exact Match)
   ใน Routing Table ก่อนเสมอ ต่างจาก `network` statement ของ OSPF/EIGRP อย่างไร?
2. AS Number ช่วง `64512 – 65534` เรียกว่าอะไร และเพราะเหตุใดหลักสูตรนี้จึงเลือกใช้ AS 65000/65001
   สำหรับ ISP-RTR และ WAN-EDGE-1/2 แทนที่จะใช้ Public ASN จริง?
3. ในการออกแบบ Dual-Homed Internet Edge (WAN-EDGE-1 + WAN-EDGE-2) เพราะเหตุใดจึงต้องสร้าง
   iBGP Session ระหว่างสอง Router นี้ด้วย ทั้งที่แต่ละตัวมี eBGP Session ของตัวเองอยู่แล้ว?
4. `neighbor <ip> default-originate` ต่างจากการตั้ง Static Default Route (`ip route 0.0.0.0
   0.0.0.0 <next-hop>`) อย่างไร และเพราะเหตุใดจึงเหมาะกับ Design แบบ Dual-Homed มากกว่า?
5. ถ้า BGP Neighbor ค้างอยู่ที่ State `Active` เป็นเวลานานผิดปกติ ควรตรวจสอบอะไรก่อนเป็นอันดับแรก?

**เฉลย:**

1. เพราะ BGP `network` statement **ไม่ได้เลือก Interface ที่จะรัน Protocol** เหมือน OSPF/EIGRP
   (ที่ Wildcard Mask แค่กำหนดว่า Interface ไหนบ้างจะเข้าร่วม Area/Process) แต่เป็นการ **"ดึง"
   Route ที่มีอยู่แล้วใน RIB (Connected/Static/IGP) เข้ามาประกาศผ่าน BGP** — ถ้าไม่มี Route
   ที่ Prefix และ Mask ตรงกันแบบเป๊ะอยู่ใน RIB ก่อน จะไม่มีอะไรถูกประกาศเลยแม้ Syntax จะถูกต้อง
   ทุกตัวอักษร (มักต้องพึ่ง Static Route ไปยัง Null0 เพื่อสร้าง Exact-match Route ให้ครบเงื่อนไข)
2. เรียกว่า **Private Use ASN** (คล้าย RFC 1918 ของ IP Address) — หลักสูตรนี้ใช้ AS 65000/65001
   เพื่อความสอดคล้องกับแนวทางที่ใช้ IP Address ช่วง RFC 1918/RFC 5737 ตลอดหลักสูตร (ป้องกันไม่ให้
   Lab สับสนหรือชนกับ ASN จริงบน Internet โดยไม่ตั้งใจ — ในโลกจริง ISP จะมี Public ASN เสมอ)
3. เพราะ WAN-EDGE-1 และ WAN-EDGE-2 เป็น**คนละ Router** ที่ไม่รู้จัก Path ของกันและกันโดยธรรมชาติ
   — ถ้า eBGP Link ของ WAN-EDGE-1 ล่ม WAN-EDGE-1 จะไม่มีทางรู้เลยว่า WAN-EDGE-2 ยังมี Path ไปยัง
   ISP-RTR ได้อยู่ นอกจากจะมี **iBGP Session** ระหว่างกันเพื่อ Relay ข้อมูล BGP (รวมถึง Default
   Route) ให้กัน — เมื่อมี iBGP แล้ว BGP Best Path Algorithm จะเลือก eBGP Path ของตัวเองเป็นหลัก
   เสมอ และสลับไปใช้ iBGP Path (จากอีกฝั่ง) โดยอัตโนมัติเมื่อ eBGP Path ของตัวเองหายไป
4. Static Default Route เป็นเพียง Entry ตายตัวที่**ไม่รู้จักสถานะจริงของปลายทาง** ถ้า ISP ล่ม
   (แต่ Interface/Next-hop ยัง Resolve ได้) Route จะยังอยู่ใน RIB ต่อไปทำให้เกิด Black-hole
   ส่วน `default-originate` ส่งผ่าน **BGP UPDATE Message จริง** ที่ผูกกับสถานะของ BGP Session
   — เมื่อ Session ล่ม BGP จะ **Withdraw** Default Route นั้นออกจาก Neighbor โดยอัตโนมัติทันที
   ทำให้ Router (ในกรณี Dual-Homed) รู้ทันทีว่าต้อง Fail-over ไปใช้ Path อื่น (ที่เรียนผ่าน iBGP)
   แทน — เหมาะกับ Design ที่ต้องการ Automatic Failover มากกว่า Static Route ธรรมดา
5. ตรวจสอบตามลำดับ: (1) **มี Route ไปยัง Neighbor IP หรือไม่** (`show ip route <neighbor-ip>`,
   `ping <neighbor-ip>`) (2) **มี ACL/Firewall บล็อก TCP Port 179** อยู่ระหว่างทางหรือไม่
   (3) ถ้า Neighbor อยู่ไกลกว่า 1 Hop (Peering ผ่าน Loopback) ตรวจสอบว่าตั้งค่า `ebgp-multihop`
   ครบถ้วนหรือยัง — State `Active` ที่ค้างนานหมายความว่า TCP Connect ยังไม่เคยสำเร็จเลยสักครั้ง
   ซึ่งเป็นปัญหาระดับ Layer 3/4 ก่อนที่ BGP เองจะได้เริ่มทำงานด้วยซ้ำ

---

## สรุป Part 29

Part นี้พาเราออกจากโลกของ IGP ที่ทำงานภายในองค์กรเดียวกัน (OSPF/EIGRP, Part 11-14, 28) ไปสู่
**BGP (Border Gateway Protocol)** — Path-Vector Protocol ที่ขับเคลื่อน Internet ทั้งใบ เราเรียนรู้
ตั้งแต่แนวคิดพื้นฐาน (AS Number, ทำไมต้องใช้ BGP ที่ WAN Edge, TCP Port 179, BGP Neighbor State
Machine), คอนฟิก **eBGP** พื้นฐานระหว่าง **WAN-EDGE-1/WAN-EDGE-2 (AS 65001)** กับ **ISP-RTR
(AS 65000)**, ภาพรวม Path Attribute หลัก, วิธีประกาศ Route เข้า BGP ทั้ง 3 แบบ, การกรอง Route
ด้วย Prefix-list/AS-Path Access-list, และปิดท้ายด้วย **Lab เต็มรูปแบบ Dual-Homed Internet Edge**
ที่พิสูจน์ Automatic Failover จริงผ่านการผสมผสาน **eBGP + iBGP + `default-originate`**

หัวข้อที่ถูก "ค้างไว้" อย่างตั้งใจใน Part นี้ (จะกลับมาสอนเต็มรูปแบบใน Part ถัดไป):
- **iBGP แบบเต็มรูปแบบ**: Full-Mesh Scaling Problem, Route Reflector, Confederation, Next-Hop-Self
  ภาคปฏิบัติ, Synchronization
- **BGP Best Path Selection Algorithm ครบทั้ง 13 ขั้นตอน** (Weight → Local-Pref → Locally
  Originated → AS-Path Length → Origin → MED → eBGP over iBGP → IGP Metric to Next-Hop → ...)
- **Route-Map ขั้นสูงสำหรับ Traffic Engineering**: Local-Preference/MED แบบเจาะลึก, AS-Path
  Prepending, Conditional Advertisement
- **BGP Communities**: การติด Tag ให้ Prefix เพื่อทำ Policy แบบยืดหยุ่นสูงข้ามหลาย Router/AS

**ไปต่อ:** [Part 30 — BGP Advanced (iBGP, Best Path Algorithm, Route-Map, Communities) →](part-030-bgp-advanced.md)
