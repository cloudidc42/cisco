# Part 32 — Policy-Based Routing (PBR)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 311–320 จาก 1000 | ระดับ CCNP ENCOR**

> ต่อจาก [Part 31 — Route Redistribution](part-031-route-redistribution.md) ที่สอนการรวม
> Routing Domain หลายตัว (OSPF ↔ EIGRP ↔ Static ↔ BGP) เข้าด้วยกันผ่าน `redistribute` — ทุก
> เทคนิคที่เรียนมาจนถึงตอนนี้ (Static Route, OSPF, EIGRP, BGP, Redistribution) ล้วนตัดสินใจ
> เส้นทางจาก **IP ปลายทาง (Destination-Based Routing)** เพียงอย่างเดียว Part นี้จะแนะนำเทคนิคที่
> **ฉีกกฎนั้น** — **Policy-Based Routing (PBR)** ที่ทำให้ Router เลือกเส้นทางจาก**เกณฑ์อื่น**ได้ เช่น
> Source IP, Application/Port, ขนาด Packet หรือ DSCP Marking โดยไม่สนใจว่า Routing Table จะบอก
> เส้นทางอะไรไว้ก็ตาม เราจะใช้ PBR บน **CORE-SW1** เพื่อบังคับ Traffic ของ **VLAN 20 (VOICE)**
> ให้วิ่งออก **WAN-EDGE-1** เสมอ (Latency ต่ำกว่า) พร้อม IP SLA Tracking ป้องกัน Black-Hole —
> เตรียมพื้นฐานสำคัญก่อนเข้าสู่ [Part 33 — Advanced STP & Campus Design](part-033-advanced-stp-campus-design.md)
> ที่จะขยาย Topology เป็น Dual-Homed DIST↔CORE เต็มรูปแบบ

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 311 | PBR คืออะไร — Routing ตามเกณฑ์อื่นที่ไม่ใช่ Destination IP |
| 312 | Syntax การคอนฟิก PBR — Route-map, match, set, และการ Apply เข้า Interface |
| 313 | ตัวอย่างจริงที่ 1 — บังคับ VLAN 20 (VOICE) ออก WAN-EDGE-1 เสมอ บน CORE-SW1 |
| 314 | `set ip next-hop verify-availability` — PBR พร้อม IP SLA Tracking ป้องกัน Black-Hole |
| 315 | PBR ตามขนาด Packet และ TOS/DSCP Marking |
| 316 | Local PBR (`ip local policy route-map`) — Policy Routing Traffic ที่ Router สร้างเอง |
| 317 | PBR vs Normal Routing vs PBR-with-VRF |
| 318 | ข้อพิจารณาด้าน Performance ของ PBR |
| 319 | Verification และ Troubleshooting PBR |
| 320 | Lab เต็มรูปแบบ: PBR + IP SLA Failover บน CORE-SW1 |

---

## Step 311 — PBR คืออะไร: Routing ตามเกณฑ์อื่นที่ไม่ใช่ Destination IP

### 311.1 ทบทวน Destination-Based Routing (สิ่งที่เราทำมาตลอด Part 1-31)

ตั้งแต่ [Part 9 — Static Routing](part-009-static-routing.md) จนถึง [Part 31 — Route
Redistribution](part-031-route-redistribution.md) กระบวนการตัดสินใจ Forward Packet ของ Router
ทุกตัวทำงานแบบเดียวกันเสมอ:

```
Packet เข้ามาที่ Router
        │
        ▼
 อ่าน Destination IP Address ใน IP Header
        │
        ▼
 Lookup ใน Routing Table (RIB) หา Longest Prefix Match
        │
        ▼
 CEF (Cisco Express Forwarding) ส่ง Packet ออกตาม Next-Hop ที่ Routing Table บอก
```

ไม่ว่า Packet จะมาจากใคร (Source IP อะไร), เป็น Traffic ประเภทไหน (Voice, Web, File Transfer),
หรือมีขนาดเท่าไร — **ถ้า Destination IP เดียวกัน Router จะเลือกเส้นทางเดียวกันเสมอ** นี่คือ
พฤติกรรม Default ของ IP Routing ทุกกรณีที่เรียนมา และเป็นพฤติกรรมที่ถูกต้องสำหรับงานส่วนใหญ่

### 311.2 ปัญหาที่ Destination-Based Routing แก้ไม่ได้

สมมติสถานการณ์ที่ Enterprise หนึ่งมี Internet Link 2 เส้น (แบบ Lab ของหลักสูตรนี้ —
**WAN-EDGE-1** และ **WAN-EDGE-2**) และ Traffic ปลายทางเดียวกันคือ **Internet (0.0.0.0/0)** ทั้งคู่
— Routing Table (ไม่ว่าจะเป็น Static, OSPF, หรือ BGP ใน Part 29-30) จะเลือก **เส้นทางเดียว** ตาม
Metric ที่ดีที่สุด (หรือ ECMP ถ้า Cost เท่ากัน) แต่ในทางปฏิบัติ วิศวกรเครือข่ายมักต้องการ:

| ความต้องการ | ตัวอย่างจริง | Destination-Based Routing ทำได้ไหม |
|---|---|---|
| Traffic บาง**ประเภท** (Voice) ต้องออกทาง Link ที่ Latency ต่ำกว่าเสมอ แม้ Metric ปกติจะเลือกอีกเส้น | VOICE VLAN ต้องออก WAN-EDGE-1 เสมอ | ❌ ทำไม่ได้ — Routing Table ไม่รู้จัก "ประเภท" ของ Traffic |
| Traffic จาก**แผนก**หนึ่งต้องออกคนละ Internet Link จากแผนกอื่น (แยก Billing/Policy) | SALES ออก ISP A, SERVERS ออก ISP B | ❌ ทำไม่ได้ — Routing ตัดสินจาก Destination อย่างเดียว ไม่สนใจ Source |
| Traffic ขนาดใหญ่ (File Transfer/Backup) ต้องแยกไปอีก Link เพื่อไม่ให้กิน Bandwidth ของ Voice/Video | Backup Job กลางคืนออกทาง Link สำรอง | ❌ ทำไม่ได้ — Routing ไม่สนใจขนาด Packet |
| Traffic ที่ถูก Mark DSCP ไว้แล้วจาก QoS (Part 24) ต้องการเส้นทางเฉพาะตาม Class | EF (Voice) ไปทางหนึ่ง, AF (Bulk Data) ไปอีกทาง | ❌ ทำไม่ได้ — Routing ไม่อ่านค่า DSCP |

ทุกกรณีข้างต้นมีจุดร่วมเดียวกัน: **เกณฑ์ตัดสินใจที่ต้องการไม่ใช่ Destination IP** — นี่คือช่องว่างที่
**Policy-Based Routing (PBR)** เข้ามาเติมเต็ม

### 311.3 นิยามของ PBR

**Policy-Based Routing (PBR)** คือกลไกที่ให้ Router ตัดสินใจ Forward Packet จาก **เกณฑ์ที่วิศวกร
กำหนดเอง (Policy)** ก่อนที่จะปล่อยให้ Routing Table ตัดสินใจตามปกติ โดยเกณฑ์ที่ใช้จับคู่
(Match) ได้ทั่วไปคือทุกอย่างที่ **Extended ACL** ตรวจสอบได้:

- **Source IP Address** (ต้นทาง — ใช้บ่อยที่สุด)
- **Destination IP Address**
- **Protocol/Port** (TCP/UDP, Application เช่น port 5060 สำหรับ SIP Voice)
- **Packet Length** (ขนาด Packet — Step 315)
- **TOS/DSCP Marking** (ค่าที่ QoS Mark ไว้แล้ว — Step 315)

เมื่อ Match แล้ว PBR จะ **"Set" ค่าบางอย่างทับ** เช่น Next-Hop IP หรือ Output Interface —
**Override ผลลัพธ์จาก Routing Table ปกติโดยสิ้นเชิงสำหรับ Packet ที่ Match เท่านั้น** (Packet
ที่ไม่ Match ยังคงใช้ Destination-Based Routing ตามปกติ — PBR ไม่ใช่การแทนที่ Routing Table
ทั้งระบบ แต่เป็น "ทางลัด" เฉพาะจุดที่ประกาศไว้)

### 311.4 ตำแหน่งของ PBR ในกระบวนการ Forward Packet

จุดสำคัญที่สุดที่ต้องเข้าใจคือ **PBR ถูกตรวจสอบก่อน Routing Table เสมอ** สำหรับ Packet ที่วิ่ง
เข้า Interface ที่เปิด PBR ไว้:

```
Packet เข้า Interface ที่มี "ip policy route-map" (Inbound เท่านั้น!)
        │
        ▼
 ┌────────────────────────────────────────┐
 │  ตรวจสอบ Route-map ที่ผูกกับ Interface นี้   │
 │  (เทียบกับ ACL ใน "match ip address")      │
 └───────────────┬────────────────────────┘
                 │
      ┌──────────┴──────────┐
      │ Match (ACL = permit)  │           │ ไม่ Match (ACL = deny หรือไม่ match เลย) │
      ▼                       ▼
 ทำตาม "set" ของ Route-map      ปล่อยผ่านไป Routing Table ปกติ (Destination-Based)
 (เช่น set ip next-hop ...)     ← **นี่คือ Fallthrough ที่สำคัญมาก ดู Step 319**
      │                       │
      ▼                       ▼
    Forward ตาม Policy      Forward ตาม Routing Table/CEF ปกติ
```

> **สรุป Step นี้**: PBR ไม่ใช่ Protocol ใหม่หรือ Routing Protocol ตัวที่ 5 — มันคือ **Mechanism
> ที่แทรกอยู่ก่อนขั้นตอน Routing Table Lookup ปกติ** โดยใช้ **Route-map** (เครื่องมือเดียวกับที่ใช้
> ใน Redistribution ของ Part 31) เป็นตัวกำหนด Policy — Step ถัดไปจะลงรายละเอียด Syntax ทั้งหมด

---

## Step 312 — Syntax การคอนฟิก PBR: Route-map, match, set, และการ Apply

### 312.1 องค์ประกอบ 3 ส่วนของ PBR

PBR ทุกกรณีประกอบด้วย 3 ส่วนเสมอ:

| ส่วน | คำสั่ง | หน้าที่ |
|---|---|---|
| 1. ACL (Match Criteria) | `access-list` (Standard/Extended) | นิยามว่า Packet แบบไหนที่จะถูก Policy Route |
| 2. Route-map | `route-map <name> permit <seq>` + `match` + `set` | เชื่อม ACL เข้ากับ Action (Set Next-Hop/Interface) |
| 3. Apply เข้า Interface | `ip policy route-map <name>` | เปิดใช้งาน PBR บน Interface ขาเข้า (Inbound) |

### 312.2 Syntax ของ Route-map สำหรับ PBR

```
Router(config)# route-map PBR-DEMO permit 10
Router(config-route-map)# match ip address <ACL-number-or-name>
Router(config-route-map)# set ip next-hop <ip-address>
Router(config-route-map)# exit
```

- **`route-map <name> permit <seq>`** — สร้าง/แก้ไข Route-map ชื่อ `<name>` ที่ Sequence
  `<seq>` — เหมือน Part 31 ทุกประการ: **Route-map ประมวลผลจากบนลงล่างตามเลข Sequence**, หยุดที่
  Clause แรกที่ Match (Top-Down เหมือน ACL)
- **`match ip address <ACL>`** — ระบุ ACL (Standard หรือ Extended) ที่ใช้จับคู่ Packet — ถ้า
  Match บรรทัด `permit` ใน ACL = Route-map Clause นี้ Match, ถ้า Match บรรทัด `deny` ใน ACL =
  ถือว่า **ไม่ Match** Clause นี้ (ข้ามไป Clause ถัดไป หรือ Fallthrough ไป Routing ปกติถ้าไม่มี
  Clause เหลือ)
- **`set ip next-hop <ip-address>`** — คำสั่งหลักที่ใช้บ่อยที่สุด — บังคับส่ง Packet ไปที่
  Next-Hop IP ที่ระบุ **ทันที โดยไม่ดู Routing Table เลย** (ยกเว้นต้องมี Route ไปหา Next-Hop
  นั้นอยู่แล้ว — ดู 312.4)

### 312.3 คำสั่ง `set` ที่ใช้ได้ทั้งหมดใน PBR (ตารางอ้างอิง)

| คำสั่ง `set` | ความหมาย | ใช้บ่อยแค่ไหน |
|---|---|---|
| `set ip next-hop <ip>` | บังคับ Next-Hop IP (รองรับได้สูงสุด 4 IP เรียงลำดับ ลองตัวแรกก่อน) | ⭐⭐⭐ บ่อยที่สุด |
| `set ip next-hop verify-availability <ip> <seq> track <obj>` | เหมือนด้านบน แต่ตรวจสอบด้วย IP SLA Tracking ก่อนใช้ (Step 314) | ⭐⭐⭐ แนะนำเสมอใน Production |
| `set interface <interface>` | บังคับ Output Interface โดยตรง (ไม่ใช้ Next-Hop IP) — เหมาะกับ Point-to-Point Link เท่านั้น (Serial, GRE Tunnel) เพราะไม่มี ARP ให้ Resolve | ⭐⭐ ใช้เฉพาะ P2P |
| `set ip default next-hop <ip>` | ใช้ Next-Hop นี้**เฉพาะเมื่อ**ไม่มี Route ปกติไปยัง Destination อยู่แล้ว (Backup เสริม Routing Table แทนที่จะ Override) | ⭐ ใช้น้อย |
| `set default interface <interface>` | เหมือนด้านบนแต่ระบุ Interface | ⭐ ใช้น้อย |
| `set ip precedence <value>` / `set ip dscp <value>` | Mark ค่า IP Precedence/DSCP ใหม่ให้ Packet (ใช้ร่วมกับ QoS Part 24) | ⭐⭐ ใช้ผสม PBR+QoS |
| `set ip qos-group <value>` | Mark QoS Group ภายใน Router (ใช้กับ MQC Policy-map ต่อ) | ⭐ Advanced |

> **ข้อควรระวัง**: `set interface` **ใช้ไม่ได้กับ Multi-Access Network เช่น Ethernet** เพราะ
> Router ไม่รู้ว่าจะ ARP หา MAC Address ของใครที่ปลายทาง (ต่างจาก Serial/Tunnel ที่เป็น
> Point-to-Point ชัดเจน ไม่ต้อง ARP) — Lab ของหลักสูตรนี้เป็น Ethernet ทั้งหมดจึงใช้
> `set ip next-hop` เป็นหลักเสมอ ไม่ใช้ `set interface`

### 312.4 กฎสำคัญ: Next-Hop ต้อง "Reachable" ผ่าน Routing Table เสมอ

`set ip next-hop <ip>` **ไม่ได้ทำให้ Router รู้จักเส้นทางใหม่** — มันแค่บอกว่า "ถ้า Match แล้ว
ให้ส่งไปที่ IP นี้" แต่ Router ยังต้องมี Route (Static, Connected, หรือ Dynamic Routing Protocol)
ที่รู้ว่า IP นั้นไปทางไหนอยู่ดี (Recursive Lookup แบบเดียวกับที่เรียนใน [Step 91-92, Part
9](part-009-static-routing.md)) — ถ้า Next-Hop ที่ตั้งไว้ไม่มี Route ไปถึงเลย Packet จะถูก
**Drop** ทันที (นี่คือความเสี่ยง "Black-Hole" ที่ Step 314 จะแก้ด้วย `verify-availability`)

### 312.5 การ Apply PBR เข้า Interface — ทิศทาง Inbound เท่านั้น

```
Router(config)# interface <interface-id>
Router(config-if)# ip policy route-map PBR-DEMO
```

> **กฎเหล็กที่ต่างจาก ACL (`ip access-group`)**: PBR ใช้คำสั่ง **`ip policy route-map`** ซึ่ง
> ทำงานได้เฉพาะ**ทิศทางขาเข้า (Inbound) เท่านั้น** — ไม่มี `ip policy route-map ... out` ให้เลือก
> เหมือน ACL — เหตุผลคือ PBR ต้องตัดสินใจ**ก่อน**ที่ Router จะเลือก Output Interface เอง ถ้า Apply
> ตอน Packet กำลังจะออก (Outbound) ก็สายเกินไปแล้ว เพราะ Forwarding Decision เกิดขึ้นไปแล้ว

### 312.6 ไดอะแกรมสรุปทั้งกระบวนการ

```
                     access-list 101 permit ip 10.10.20.0 0.0.0.255 any
                                          │
                                          ▼
route-map PBR-VOICE permit 10 ──match ip address 101──┐
        │                                              │ Match
        │ set ip next-hop 10.10.254.1                  ▼
        │                                     Forward ไปที่ 10.10.254.1 (WAN-EDGE-1)
        │                                     ทันที ไม่ดู Routing Table
        ▼
  interface TenGigabitEthernet1/0/1
   ip policy route-map PBR-VOICE     <-- Apply ที่ Interface ขาเข้าจาก DIST-SW1
```

---

## Step 313 — ตัวอย่างจริงที่ 1: บังคับ VLAN 20 (VOICE) ออก WAN-EDGE-1 เสมอ บน CORE-SW1

### 313.1 บริบท Topology (ทวนจาก [00-ip-address-plan.md](00-ip-address-plan.md))

```
                        ┌─────────────┐        ┌─────────────┐
                        │  WAN-EDGE-1  │        │  WAN-EDGE-2  │
                        │ 1.1.1.21    │═══════│ 1.1.1.22    │  cross-link 10.10.254.8/30
                        └──────┬──────┘        └──────┬──────┘
                    10.10.254.0/30 (.1)         10.10.254.4/30 (.5)
                               │                       │
                        Gi1/0/1 (.2)             Gi1/0/1 (.6)
                        ┌──────┴──────┐  Po1   ┌──────┴──────┐
                        │  CORE-SW1    │═══════│  CORE-SW2    │  10.255.0.0/30
                        │  1.1.1.1     │        │  1.1.1.2     │
                        └──┬───────┬──┘        └──┬───────┬──┘
                    Te1/0/1│       │Te1/0/2   Te1/0/1│      │Te1/0/2
                           │       │                 │      │
                     ┌─────┴──┐ ┌──┴─────┐     ┌─────┴──┐ ┌─┴──────┐
                     │DIST-SW1│ │DIST-SW2│     │DIST-SW3│ │DIST-SW4│
                     └────────┘ └────────┘     └────────┘ └────────┘
                     VLAN10/20  VLAN10/20      VLAN30/40  VLAN30/40
                     (SALES/VOICE บนบล็อกนี้)
```

จาก [Part 18](part-018-fhrp.md) เรารู้ว่า **VLAN 10 (SALES)** และ **VLAN 20 (VOICE)** อยู่ใน
"บล็อก 1" ที่ผูกกับ **DIST-SW1/DIST-SW2** ซึ่งทั้งคู่ Uplink ขึ้นไปที่ **CORE-SW1** — ดังนั้น
Traffic ทุกอย่างของ VLAN 10 และ VLAN 20 ที่จะออกไปยัง Internet/WAN ต้องผ่าน **CORE-SW1** เสมอ
นี่คือจุดที่ถูกต้องที่สุดในการวาง PBR

### 313.2 เป้าหมายทางธุรกิจ

- **VLAN 20 (VOICE)**: Latency สำคัญที่สุด — ต้องออก **WAN-EDGE-1 เสมอ** เพราะเป็น Link หลักที่มี
  Latency ต่ำกว่า (WAN-EDGE-2 เป็น Link สำรอง Bandwidth เท่ากันแต่ผ่าน ISP อีกเจ้าที่มี Latency
  สูงกว่าเล็กน้อยตาม SLA ที่ตกลงไว้) — **ไม่ต้องการให้ Routing Protocol (OSPF/BGP ใน Part
  29-30) เปลี่ยนเส้นทางของ Voice โดยอัตโนมัติ** แม้ในอนาคต Metric หรือ BGP Attribute จะถูกปรับ
  เพื่อ Load-Balance Traffic อื่นก็ตาม — Voice ต้อง**เสถียรและคาดเดาได้เสมอ**
- **VLAN 10 (SALES)**: Traffic ทั่วไป (Web, Email, Application) — ปล่อยให้ **OSPF/BGP เลือก
  เส้นทางที่ดีที่สุดตามปกติ** ไม่ต้องบังคับ

นี่คือตัวอย่างคลาสสิกของเหตุผลที่ต้องใช้ PBR: **Requirement ไม่ได้ขึ้นกับปลายทาง (ปลายทางคือ
Internet เหมือนกันทั้งคู่) แต่ขึ้นกับ "ต้นทาง" (VLAN ไหน) และ "นโยบายทางธุรกิจ" (คุณภาพเสียง)**

### 313.3 คอนฟิกจริงบน CORE-SW1

**ขั้นที่ 1 — สร้าง Extended ACL ระบุ Traffic ของ VOICE ที่จะออก Internet**

```
CORE-SW1(config)# ip access-list extended VOICE-TO-INTERNET
CORE-SW1(config-ext-nacl)# deny   ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# permit ip 10.10.20.0 0.0.0.255 any
CORE-SW1(config-ext-nacl)# exit
```

> **จุดสำคัญที่สุดของ Step นี้ — ทำไมต้องมีบรรทัด `deny` ก่อน**: ต้อง **แยก Traffic ภายใน
> องค์กร (VOICE คุยกับ Server อื่นใน 10.10.0.0/16) ออกจาก Traffic ที่จะออก Internet จริงๆ**
> ถ้าไม่มีบรรทัด `deny` นี้ **VOICE ที่คุยกับ Server ภายใน (เช่น Server3 ที่รัน Call Manager ใน
> VLAN 30) ก็จะถูกบังคับส่งไป WAN-EDGE-1 ด้วย** ทั้งที่ควรไปทาง OSPF ปกติภายใน Campus เท่านั้น
> — จำไว้เสมอ: **บรรทัด `deny` ใน ACL ที่ใช้กับ `match ip address` ของ PBR ไม่ได้แปลว่า "บล็อก
> Traffic" แต่แปลว่า "ไม่ต้อง Policy-Route Traffic นี้ ให้ใช้ Routing Table ปกติแทน"** (รายละเอียด
> เต็มอยู่ใน Step 319)

**ขั้นที่ 2 — สร้าง Route-map**

```
CORE-SW1(config)# route-map PBR-VOICE permit 10
CORE-SW1(config-route-map)# match ip address VOICE-TO-INTERNET
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.1
CORE-SW1(config-route-map)# exit
```

**ขั้นที่ 3 — Apply เข้า Interface ขาเข้าจาก DIST-SW1 และ DIST-SW2**

```
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# description ** Uplink to DIST-SW1 **
CORE-SW1(config-if)# ip policy route-map PBR-VOICE
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# description ** Uplink to DIST-SW2 **
CORE-SW1(config-if)# ip policy route-map PBR-VOICE
CORE-SW1(config-if)# exit
```

> **ทำไมต้อง Apply ทั้ง 2 Interface**: เพราะ VLAN 20 เป็น HSRP Active สลับกันได้ระหว่าง DIST-SW1
> และ DIST-SW2 ตาม Design ของ [Part 18](part-018-fhrp.md) — Traffic ของ VOICE อาจไหลเข้า CORE-SW1
> จากฝั่งใดก็ได้ขึ้นกับว่าใครเป็น HSRP Active ตอนนั้น จึงต้อง Apply PBR ให้ครบทุก Interface ขาเข้า
> ที่เป็นไปได้ ไม่ใช่แค่เส้นเดียว

### 313.4 ผลลัพธ์ที่คาดหวัง

```
                    ก่อนมี PBR                         หลังมี PBR
   VOICE (VLAN20) ──► OSPF/BGP เลือกเส้นทาง       VOICE (VLAN20) ──► บังคับ WAN-EDGE-1 เสมอ
                       (อาจเป็น WAN-EDGE-1                          (ไม่สนใจ Metric ของ OSPF/BGP)
                        หรือ WAN-EDGE-2 ก็ได้
                        ขึ้นกับ Metric ขณะนั้น)

   SALES (VLAN10)  ──► OSPF/BGP เลือกเส้นทาง       SALES (VLAN10)  ──► ยังคง OSPF/BGP เลือกเส้นทาง
                       ตามปกติ                                        ตามปกติ (ไม่ Match ACL ของ PBR)
```

### 313.5 Verify เบื้องต้น

```
CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  Match clauses:
    ip address (access-lists): VOICE-TO-INTERNET
  Set clauses:
    ip next-hop 10.10.254.1
  Policy routing matches: 1284 packets, 167920 bytes

CORE-SW1# show ip policy
Interface      Route map
Te1/0/1        PBR-VOICE
Te1/0/2        PBR-VOICE
```

`Policy routing matches` เพิ่มขึ้นเรื่อยๆ เมื่อมี Traffic VOICE ออก Internet จริง ยืนยันว่า PBR
ทำงาน — รายละเอียด Verification เต็มรูปแบบอยู่ใน Step 319 และ 320

---

## Step 314 — `set ip next-hop verify-availability`: PBR พร้อม IP SLA Tracking

### 314.1 ปัญหาของ `set ip next-hop` แบบธรรมดา — Black-Hole

Config ใน Step 313 มีจุดอ่อนร้ายแรง: **ถ้า WAN-EDGE-1 ล่ม หรือ Interface Gi1/0/1 ของ CORE-SW1
หลุด `set ip next-hop 10.10.254.1` จะยังคงพยายามส่ง Packet ไปที่ IP นั้นต่อไปเรื่อยๆ** แม้ว่า
Routing Table จะรู้อยู่แล้วว่ามีเส้นทางสำรองไป Internet ผ่าน WAN-EDGE-2 ก็ตาม — PBR **ไม่สนใจว่า
Next-Hop ที่ตั้งไว้ยังมีชีวิตอยู่หรือไม่** (Static ตายตัว) ผลคือ Voice Traffic ทั้งหมด**ถูกทิ้ง
(Black-Hole)** แม้ทั้ง Campus จะยังมีทางออก Internet ผ่าน WAN-EDGE-2 อยู่เต็มประสิทธิภาพ

```
                     ❌ WAN-EDGE-1 ล่ม (Config Step 313 อย่างเดียว)

   VOICE Packet ──► CORE-SW1 (PBR: set ip next-hop 10.10.254.1) ──► 10.10.254.1 ไม่ตอบสนอง
                                                                     Packet ถูก DROP ทันที
                                                                     (แม้ WAN-EDGE-2 พร้อมใช้งาน
                                                                      100% อยู่เฉยๆ ก็ตาม)
```

นี่คือเหตุผลที่ **ห้ามใช้ `set ip next-hop` แบบธรรมดาใน Production โดยไม่มี Tracking** — ต้องใช้
คู่กับ **IP SLA (Service Level Agreement) + Object Tracking** เสมอ (แนวคิดเดียวกับ HSRP
Interface Tracking ใน [Step 174, Part 18](part-018-fhrp.md) แต่ตรวจสอบ**ความมีชีวิตของ Next-Hop
ทาง IP โดยตรง** ด้วย ICMP Echo แทนที่จะดูแค่ Interface Line Protocol Up/Down)

### 314.2 แนวคิดของ IP SLA + Track

```
┌──────────────────────┐   ICMP Echo ทุก 5 วินาที   ┌──────────────────┐
│   ip sla 1            │─────────────────────────►│  10.10.254.1      │
│   icmp-echo ...        │◄─────────────────────────│  (WAN-EDGE-1)     │
└──────────┬────────────┘   ถ้าไม่ตอบภายใน Timeout   └──────────────────┘
           │
           ▼
┌──────────────────────┐
│   track 1 ip sla 1     │  ── ผล SLA เป็น "Down" → Track Object เปลี่ยนเป็น "Down" ทันที
│   reachability          │
└──────────┬────────────┘
           │
           ▼
┌────────────────────────────────────────────┐
│ set ip next-hop verify-availability          │  ── เห็น Track "Down" → ข้าม Next-Hop นี้ไป
│   10.10.254.1 10 track 1                     │     ทันที → **Fallthrough ไป Routing Table ปกติ**
└────────────────────────────────────────────┘     (ไม่ Black-Hole อีกต่อไป!)
```

### 314.3 คอนฟิกเต็มรูปแบบ — IP SLA + Track + PBR

**ขั้นที่ 1 — สร้าง IP SLA Probe ตรวจสอบ WAN-EDGE-1**

```
CORE-SW1(config)# ip sla 1
CORE-SW1(config-ip-sla)# icmp-echo 10.10.254.1 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla-echo)# frequency 5
CORE-SW1(config-ip-sla-echo)# timeout 1000
CORE-SW1(config-ip-sla-echo)# threshold 800
CORE-SW1(config-ip-sla-echo)# exit
CORE-SW1(config)# ip sla schedule 1 life forever start-time now
```

| คำสั่ง | ความหมาย |
|---|---|
| `ip sla 1` | สร้าง SLA Operation หมายเลข 1 |
| `icmp-echo 10.10.254.1 source-interface Gi1/0/1` | ส่ง ICMP Echo (Ping) ไปที่ WAN-EDGE-1 โดยระบุ Source Interface ชัดเจน (ป้องกันปัญหา Source IP ผิด Interface) |
| `frequency 5` | Ping ทุก 5 วินาที (Default 60 วินาที — ช้าเกินไปสำหรับ Failover เร็ว) |
| `timeout 1000` | รอ Response ไม่เกิน 1000 ms ต่อครั้ง |
| `threshold 800` | ถ้า Response Time เกิน 800 ms ให้เริ่มนับเป็นสัญญาณเตือน (ใช้กับ SLA Reaction เพิ่มเติมได้ ไม่บังคับสำหรับ Track พื้นฐาน) |
| `ip sla schedule 1 life forever start-time now` | เริ่มรัน SLA ทันทีและรันตลอดไป (ถ้าไม่ใส่บรรทัดนี้ SLA จะไม่ทำงานเลย! เป็นจุดพลาดที่พบบ่อย) |

**ขั้นที่ 2 — สร้าง Track Object ผูกกับ SLA**

```
CORE-SW1(config)# track 1 ip sla 1 reachability
CORE-SW1(config-track)# delay down 2 up 5
CORE-SW1(config-track)# exit
```

| คำสั่ง | ความหมาย |
|---|---|
| `track 1 ip sla 1 reachability` | สร้าง Track Object หมายเลข 1 ที่ดูผล "Reachability" ของ SLA 1 (มีทางเลือกอื่น เช่น `rtt` ดู Round-Trip Time โดยตรง) |
| `delay down 2 up 5` | หน่วงเวลา 2 วินาทีก่อนประกาศ "Down" จริง (ป้องกัน Flapping จาก Packet Loss ชั่วคราว) และหน่วง 5 วินาทีก่อนกลับมา "Up" (ให้แน่ใจว่า Link เสถียรจริงก่อน Failback) |

**ขั้นที่ 3 — แก้ Route-map ให้ใช้ `verify-availability`**

```
CORE-SW1(config)# route-map PBR-VOICE permit 10
CORE-SW1(config-route-map)# match ip address VOICE-TO-INTERNET
CORE-SW1(config-route-map)# no set ip next-hop 10.10.254.1
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.10.254.1 10 track 1
CORE-SW1(config-route-map)# exit
```

Syntax เต็ม: `set ip next-hop verify-availability <ip> <sequence> track <track-object>`
- `<ip>` — Next-Hop เดิม (WAN-EDGE-1)
- `<sequence>` — ลำดับความสำคัญถ้ามีหลาย Next-Hop (รองรับสูงสุด 6 รายการต่อ Route-map Clause
  เดียว — เรียงจากน้อยไปมาก ลองตัวที่ Sequence ต่ำสุดที่ Track "Up" ก่อนเสมอ)
- `track <object>` — ผูกกับ Track Object ที่สร้างไว้

### 314.4 พฤติกรรมเมื่อ WAN-EDGE-1 ล่ม (ยังไม่มี Next-Hop สำรอง)

```
CORE-SW1#
%TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
```

เมื่อ Track เห็นว่า Down — เพราะ Route-map นี้มี `set ip next-hop verify-availability` แค่รายการ
เดียวและไม่มีรายการสำรอง — PBR จะ**หา Next-Hop ที่ Track อยู่ไม่เจอเลย** ผลคือ Clause นี้ถือว่า
**"Set ไม่สำเร็จ" และ Packet จะ Fallthrough ไปใช้ Routing Table ปกติทันที** (ไม่ Drop!) นี่คือ
พฤติกรรม Default ที่ปลอดภัยกว่า `set ip next-hop` ธรรมดามาก — Voice Traffic จะไหลไปตามเส้นทางที่
OSPF/BGP เลือกในขณะนั้น (อาจเป็น WAN-EDGE-2) แทนที่จะถูกทิ้งทั้งหมด

> **ข้อสังเกตสำคัญ**: การ Fallthrough ไป Routing Table ปกติเมื่อ Next-Hop เดียวที่มีตายไป **ยังไม่ใช่
> การ Failover ไปยัง WAN-EDGE-2 แบบมี IP SLA Tracking ของ WAN-EDGE-2 ด้วย** — มันแค่ปล่อยให้
> Routing Table ตัดสินใจเองล้วนๆ ถ้าต้องการ Failover แบบ Deterministic ไปยัง WAN-EDGE-2 ที่มี
> Track ของตัวเองด้วย ต้องเพิ่ม `set ip next-hop verify-availability` รายการที่ 2 เข้าไปในลำดับ
> ถัดไป — รายละเอียดเต็มรูปแบบอยู่ใน **Step 320 (Lab เต็มรูปแบบ)**

---

## Step 315 — PBR ตามขนาด Packet และ TOS/DSCP Marking

### 315.1 Match ตามขนาด Packet ด้วย `match length`

นอกจาก Source/Destination IP แล้ว Route-map ยังรองรับการ Match จาก**ขนาด Packet โดยตรง** โดยไม่
ต้องพึ่ง ACL เลย:

```
Router(config-route-map)# match length <min> <max>
```

**Use Case**: แยก Traffic **File Transfer/Backup ขนาดใหญ่** (Packet ใหญ่ใกล้ MTU ~1500 byte)
ออกจาก **Traffic โต้ตอบแบบ Interactive** (Voice RTP Packet เล็ก ~160-200 byte, SSH/Telnet
Keystroke เล็กมาก) เพราะ Traffic ก้อนใหญ่จำนวนมากพร้อมกัน (เช่น Nightly Backup Job) จะทำให้
Router/Link Congest และเพิ่ม Latency ให้ Traffic เล็กที่ไวต่อ Delay อย่าง Voice

```
CORE-SW1(config)# route-map PBR-BY-SIZE permit 10
CORE-SW1(config-route-map)# match length 1400 1500
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.6
! Packet ใหญ่ (File Transfer) → ส่งผ่าน WAN-EDGE-2 (Bandwidth มากกว่า ไม่ปนกับ Voice)

CORE-SW1(config)# route-map PBR-BY-SIZE permit 20
CORE-SW1(config-route-map)# match length 0 200
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.1
! Packet เล็ก (Voice RTP, Keystroke) → ส่งผ่าน WAN-EDGE-1 (Latency ต่ำ)
```

> **ข้อจำกัดของ `match length`**: วัดจากขนาด **IP Packet ทั้งก้อน (Header + Payload)** ไม่ใช่
> เนื้อหาข้างในที่แท้จริง — Application บางตัวส่ง Packet ขนาดแปรผันได้ (เช่น Video Call ที่ปรับ
> Bitrate อัตโนมัติ) ทำให้ `match length` แม่นยำน้อยกว่าการ Match ที่ DSCP โดยตรง (315.2) ซึ่ง
> เป็นวิธีที่ Enterprise จริงนิยมใช้มากกว่าในปัจจุบัน

### 315.2 Match ตาม DSCP/TOS ที่ QoS Mark ไว้แล้ว

ทวนจาก [Part 24 — QoS Fundamentals](part-024-qos-fundamentals.md): Traffic ที่ผ่าน QoS
Classification/Marking มาแล้วจะมีค่า **DSCP** ติดอยู่ใน IP Header เช่น `EF` (Expedited
Forwarding — Voice), `AF41` (Assured Forwarding — Video), `CS1` (Bulk Data) — PBR สามารถ
**อ่านค่าที่ Mark ไว้แล้วนี้มาตัดสินใจเส้นทางต่อได้เลย** โดยไม่ต้องดู Source/Destination อีก
(สะดวกกว่ามากในเครือข่ายที่ทำ QoS Marking ตั้งแต่ Access Layer แล้ว):

```
CORE-SW1(config)# ip access-list extended MATCH-VOICE-EF
CORE-SW1(config-ext-nacl)# permit ip any any dscp ef
CORE-SW1(config-ext-nacl)# exit

CORE-SW1(config)# ip access-list extended MATCH-BULK-CS1
CORE-SW1(config-ext-nacl)# permit ip any any dscp cs1
CORE-SW1(config-ext-nacl)# exit

CORE-SW1(config)# route-map PBR-BY-DSCP permit 10
CORE-SW1(config-route-map)# match ip address MATCH-VOICE-EF
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.10.254.1 10 track 1
CORE-SW1(config-route-map)# exit

CORE-SW1(config)# route-map PBR-BY-DSCP permit 20
CORE-SW1(config-route-map)# match ip address MATCH-BULK-CS1
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.6
CORE-SW1(config-route-map)# exit
```

> **ข้อดีของการ Match ด้วย DSCP เทียบกับ Match ด้วย Source IP (Step 313)**: ไม่ผูกกับ VLAN/Subnet
> เลย — ถ้าในอนาคตมี Voice Endpoint เพิ่มใน VLAN อื่น (เช่น Softphone บน PC ใน VLAN 10) ตราบใดที่
> QoS Mark DSCP EF ให้ถูกต้องตั้งแต่ Access Layer ตาม Part 24 แล้ว PBR ตัวนี้จะยังคงทำงานถูกต้อง
> โดยไม่ต้องแก้ ACL เพิ่มเลย — ในทางปฏิบัติ Enterprise ขนาดใหญ่มักใช้ **DSCP-based PBR** มากกว่า
> **Source-IP-based PBR** ด้วยเหตุผลนี้ แต่ Lab หลักของหลักสูตรนี้ยังคงสาธิตด้วย Source IP (Step
> 313) เพราะเข้าใจง่ายกว่าสำหรับผู้เริ่มต้น CCNP

### 315.3 ตารางสรุปเปรียบเทียบวิธี Match ทั้งหมดของ PBR

| วิธี Match | คำสั่ง | เหมาะกับ |
|---|---|---|
| Source/Destination IP | `match ip address <ACL>` | แยกตาม VLAN/Subnet/Department (Step 313) |
| Protocol/Port | `match ip address <extended-ACL with port>` | แยกตาม Application เฉพาะเจาะจง (เช่น SIP port 5060) |
| ขนาด Packet | `match length <min> <max>` | แยก Traffic ใหญ่ (Bulk) ออกจาก Traffic เล็ก (Interactive) |
| DSCP/TOS | `match ip address <ACL with dscp>` หรือ `match ip dscp <value>` | แยกตาม QoS Class ที่ Mark มาแล้ว — แนะนำสำหรับ Enterprise ขนาดใหญ่ |

---

## Step 316 — Local PBR (`ip local policy route-map`)

### 316.1 ความแตกต่างระหว่าง PBR ปกติกับ Local PBR

ทุก Step ก่อนหน้านี้ (311-315) พูดถึง **Transit Traffic** — Packet ที่ **ผ่าน** Router (มาจาก
Host อื่น ไปหา Host อื่น) เท่านั้น แต่ Router เองก็**สร้าง Traffic ของตัวเอง**ได้เช่นกัน เช่น:

- SSH/Telnet ที่ Router สร้างขึ้นเพื่อ Login ไปยังอุปกรณ์อื่น
- Syslog Message ที่ส่งไปหา Syslog Server ([Part 17](part-017-ntp-syslog-snmp.md))
- SNMP Trap, NTP Query, TFTP/FTP สำหรับโอนไฟล์ IOS
- Ping ที่ Admin สั่งรันจาก Router เอง

Traffic เหล่านี้**ไม่เคยผ่าน Interface ขาเข้าเลย** — มัน**เกิดขึ้นภายใน Router เอง (Originate
Locally)** ดังนั้น `ip policy route-map` ที่ Apply บน Interface (Step 312) **จะไม่มีผลกับ
Traffic ประเภทนี้เลย** เพราะ Traffic เหล่านี้ไม่เคย "เข้า" Interface ใดๆ ก่อนจะถูกส่งออก

```
Transit PBR (Step 311-315)                    Local PBR (Step 316)

  Host A ──► [Interface In] ──PBR──► Router     Router สร้าง Packet เอง (Syslog, SSH, Ping)
                                        │                      │
                                        ▼                      ▼
                                    Forward ออก           ip local policy route-map
                                                                │
                                                                ▼
                                                           Forward ออก
```

### 316.2 Syntax ของ Local PBR

```
Router(config)# ip local policy route-map <name>
```

**ข้อแตกต่างสำคัญจาก Transit PBR**: Local PBR **ใช้ Global Configuration Mode** (ไม่ใช่
Interface Mode) เพราะ Traffic ที่เกิดจาก Router เองไม่ได้ผูกกับ Interface ขาเข้าใดๆ — คำสั่งนี้
มีผลกับ**ทั้ง Router**เพียงชุดเดียว (ไม่สามารถมี Local Policy Route-map หลายอันพร้อมกันแบบ
Interface ที่ Apply คนละ Route-map ได้)

### 316.3 ตัวอย่าง: บังคับ Syslog/SNMP ของ CORE-SW1 ให้ออกทาง WAN-EDGE-1 เสมอ

สมมติ Syslog Server และ NTP Server ขององค์กรอยู่ที่ Data Center ภายนอกที่เข้าถึงผ่าน Internet
(ผ่าน WAN-EDGE-1/2 เหมือนกัน) และต้องการบังคับ Management Traffic ของ CORE-SW1 เองให้ออกทาง
WAN-EDGE-1 เสมอเพื่อความเสถียรของ Log (สอดคล้องกับ Design ของ Part 17):

```
CORE-SW1(config)# ip access-list extended LOCAL-MGMT-TRAFFIC
CORE-SW1(config-ext-nacl)# permit udp host 10.10.99.1 any eq 514
CORE-SW1(config-ext-nacl)# permit udp host 10.10.99.1 any eq 123
CORE-SW1(config-ext-nacl)# exit

CORE-SW1(config)# route-map LOCAL-PBR permit 10
CORE-SW1(config-route-map)# match ip address LOCAL-MGMT-TRAFFIC
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.10.254.1 10 track 1
CORE-SW1(config-route-map)# exit

CORE-SW1(config)# ip local policy route-map LOCAL-PBR
```

> **หมายเหตุ**: `10.10.99.1` ในตัวอย่างคือ HSRP VIP ของ VLAN 99 (MGMT) ตาม
> [00-ip-address-plan.md](00-ip-address-plan.md) — ในทางปฏิบัติต้อง Match ด้วย Source IP ที่
> CORE-SW1 ใช้จริงตอนส่ง Syslog/NTP ออกไป (มักผูกกับ `ip ssh source-interface` หรือ Loopback0
> ถ้าตั้งค่าไว้ตาม Best Practice ของ [Part 17](part-017-ntp-syslog-snmp.md))

### 316.4 Verify Local PBR

```
CORE-SW1# show ip local policy
Local policy routing is enabled, using route map LOCAL-PBR
route-map LOCAL-PBR, permit, sequence 10
  Match clauses:
    ip address (access-lists): LOCAL-MGMT-TRAFFIC
  Set clauses:
    ip next-hop verify-availability 10.10.254.1 10 track 1  [up]
  Policy routing matches: 342 packets, 41280 bytes
```

---

## Step 317 — PBR vs Normal Routing vs PBR-with-VRF

### 317.1 ตารางเปรียบเทียบสรุป

| หัวข้อ | Normal (Destination-Based) Routing | PBR (Policy-Based Routing) |
|---|---|---|
| เกณฑ์ตัดสินใจ | Destination IP เท่านั้น | Source IP, Port, ขนาด Packet, DSCP ฯลฯ (กำหนดเองได้) |
| ใครเป็นคนตัดสินเส้นทาง | Routing Protocol (OSPF/EIGRP/BGP) อัตโนมัติ | วิศวกรกำหนด Policy เองตรงๆ ผ่าน Route-map |
| ปรับตาม Network Failure อัตโนมัติไหม | ✅ ได้ (Reconverge ตาม Routing Protocol) | ⚠️ ต้องพึ่ง `verify-availability` เท่านั้น ไม่งั้น Static ตายตัว |
| Scale ง่ายแค่ไหน | ง่าย — เพิ่ม Network ใหม่ Routing Protocol จัดการเอง | ยากกว่า — ต้องแก้ ACL/Route-map ทุกครั้งที่ Policy เปลี่ยน |
| ใช้ CPU/TCAM มากกว่ากันไหม | น้อยกว่า (CEF ล้วนๆ) | มากกว่าเล็กน้อยถึงมาก ขึ้นกับ Platform (ดู Step 318) |

### 317.2 PBR-with-VRF — กล่าวถึงสั้นๆ (รายละเอียดเต็มใน Part 51)

ใน Enterprise ขนาดใหญ่ที่ต้องการแยก Routing Table หลายชุดพร้อมกันบน Router ตัวเดียว (Multi-Tenant,
Multi-Topology) จะใช้ **VRF (Virtual Routing and Forwarding)** ร่วมกับ PBR ผ่านคำสั่ง
`set vrf <vrf-name>` แทนที่จะ Set Next-Hop ตรงๆ — เทคนิคนี้เรียกว่า **PBR-based VRF Selection**
หรือ **VRF-Lite with PBR** ทำให้ Traffic ที่ Match Policy หนึ่งถูกส่งเข้า Routing Table
(VRF) คนละชุดกันโดยสมบูรณ์ ไม่ใช่แค่ Next-Hop เดียวแบบที่เรียนใน Part นี้

```
Router(config-route-map)# set vrf CUSTOMER-A
```

**หลักสูตรนี้ยังไม่ครอบคลุมแนวคิด VRF** เพราะ VRF ต้องปูพื้นฐาน Multi-VRF Routing ก่อน — เนื้อหา
เต็มรูปแบบของ VRF และ Advanced Multi-Topology PBR จะอยู่ใน **[Part 51 — VRF-Lite &
Multi-Tenancy]** (Part ต่อๆ ไปของหลักสูตร) — ในตอนนี้ให้จำไว้แค่ว่า **PBR ไม่ได้จำกัดแค่การ Set
Next-Hop เท่านั้น แต่ยังส่ง Traffic ข้าม Routing Table ทั้งชุดได้ด้วย เมื่อผสมกับ VRF**

---

## Step 318 — ข้อพิจารณาด้าน Performance ของ PBR

### 318.1 ทำไม PBR "แพงกว่า" Normal Routing บน Platform รุ่นเก่า

Normal Destination-Based Routing บน Cisco IOS สมัยใหม่ใช้ **CEF (Cisco Express Forwarding)** ซึ่ง
เป็น Forwarding แบบ Hardware-Accelerated เต็มรูปแบบ — คำนวณ Forwarding Table (FIB) ไว้ล่วงหน้า
ทั้งหมดตั้งแต่ก่อน Packet มาถึง ทำให้ Forward Packet ได้เร็วมากในระดับ Hardware ล้วนๆ

PBR แบบดั้งเดิมบน Platform รุ่นเก่า (เช่น ISR รุ่นเก่า, Switch รุ่นก่อน TCAM-based PBR) ต้อง
**ประมวลผล Route-map แบบ Process-Switching หรือ CEF-based แต่ยังต้องเดิน Logic ของ ACL/Route-map
เพิ่มเติมทุก Packet** ซึ่งใช้ CPU มากกว่า CEF ล้วนๆ อย่างมีนัยสำคัญ โดยเฉพาะเมื่อ Route-map มี
หลาย Clause หรือ ACL มีหลายบรรทัด

### 318.2 Catalyst 9000 Series — PBR ทำงานใน Hardware (TCAM) แล้ว

ข่าวดีสำหรับ Lab ของหลักสูตรนี้: **Catalyst 9500 (CORE-SW1/2) และ Catalyst 9300 (DIST-SW1-4)**
ตาม [00-ip-address-plan.md](00-ip-address-plan.md) รองรับ **PBR แบบ Hardware-Accelerated ผ่าน
TCAM (Ternary Content-Addressable Memory)** เต็มรูปแบบ — Route-map และ ACL ของ PBR ถูก
Compile ลง TCAM Entry ตั้งแต่ตอน Config เสร็จ (คล้ายกับที่ ACL ปกติทำงานใน TCAM เช่นกัน) ทำให้
การตัดสินใจ PBR เกิดขึ้นที่ **Line-Rate ระดับ Hardware เดียวกับ Normal CEF Forwarding** ไม่มี
Overhead ที่สังเกตเห็นได้ในทางปฏิบัติ

> **ข้อพึงระวังแม้บน Catalyst 9000**: TCAM มีขนาดจำกัด — ถ้า Route-map/ACL ของ PBR มีจำนวน Entry
> มากเกินไป (หลายพันบรรทัด) รวมกับ ACL อื่นๆ ที่ใช้อยู่แล้วในระบบ (Security ACL, QoS ACL) อาจทำให้
> TCAM เต็มและบาง Feature ตกไปทำงานแบบ Software-Switching แทน (ช้าลงมาก) — ควรตรวจสอบ TCAM
> Utilization ด้วย `show platform hardware fed switch active fwd-asic resource tcam
> utilization` เป็นระยะในเครือข่ายที่ใช้ PBR + ACL จำนวนมากพร้อมกัน

### 318.3 ตารางสรุปคำแนะนำ

| Platform | PBR ทำงานแบบ | คำแนะนำ |
|---|---|---|
| Router รุ่นเก่า (ISR G1/G2, Software-based) | Process/CEF-based ผสม Software Logic | ใช้ PBR เท่าที่จำเป็น หลีกเลี่ยง Route-map ที่ซับซ้อนมาก |
| Catalyst 9200/9300/9500 (Lab นี้ใช้) | Hardware TCAM เต็มรูปแบบ | ใช้ PBR ได้อย่างมั่นใจ Performance เทียบเท่า Normal Forwarding |
| ตรวจสอบ TCAM ก่อน Deploy จำนวนมาก | - | `show platform hardware fed switch active fwd-asic resource tcam utilization` |

---

## Step 319 — Verification และ Troubleshooting PBR

### 319.1 คำสั่ง Verify หลัก

| คำสั่ง | ใช้ตรวจอะไร |
|---|---|
| `show route-map <name>` | ดู Route-map ทั้งหมด รวม Hit Counter (`Policy routing matches`) ต่อ Clause |
| `show ip policy` | ดูว่า Interface ไหนผูก Route-map (PBR) ตัวไหนอยู่บ้าง |
| `show ip local policy` | ดู Local PBR (Step 316) ที่ผูกกับ Router เอง |
| `debug ip policy` | ดู Real-time ว่า Packet ไหนถูก PBR จับ Match Clause ไหน (ใช้ระมัดระวังใน Production — CPU Load สูง) |
| `show access-lists <name>` | ดู Hit Counter ของแต่ละบรรทัด ACL ที่ PBR อ้างอิงอยู่ (แยกว่า Match บรรทัด permit หรือ deny) |
| `show track <object-number>` | ดูสถานะ Track Object ที่ผูกกับ `verify-availability` (Step 314) |
| `show ip sla statistics` | ดูผลลัพธ์ ICMP Echo ของ IP SLA ว่า Success/Fail กี่ครั้ง |

### 319.2 ตัวอย่าง `debug ip policy`

```
CORE-SW1# debug ip policy
Policy routing debugging is on
CORE-SW1#
IP: s=10.10.20.15 (TenGigabitEthernet1/0/1), d=8.8.8.8, len 200, policy match
IP: route map PBR-VOICE, item 10, permit
IP: s=10.10.20.15 (TenGigabitEthernet1/0/1), d=8.8.8.8 (Vlan99), len 200, policy routed
IP: TenGigabitEthernet1/0/1 to GigabitEthernet1/0/1 10.10.254.1
CORE-SW1# undebug all
```

อ่านผลลัพธ์: Packet จาก `10.10.20.15` (Host ใน VLAN 20 VOICE) ไปหา `8.8.8.8` (จำลอง Internet)
**Match Route-map PBR-VOICE Clause 10 → ถูกส่งออกทาง Gi1/0/1 ไปที่ 10.10.254.1** ยืนยันว่า PBR
ทำงานถูกต้องตาม Design

### 319.3 ข้อผิดพลาดที่พบบ่อยที่สุด: ACL ไม่ Match อะไรเลย → PBR เงียบๆ ไม่ทำงาน (ตามที่ออกแบบไว้)

นี่คือพฤติกรรมที่ทำให้วิศวกรมือใหม่สับสนมากที่สุด: **ถ้า ACL ที่ `match ip address` อ้างอิงอยู่
ไม่ Match Packet เลยสักบรรทัด (หรือ ACL เขียนผิด Subnet/Wildcard Mask) — PBR จะไม่ Error หรือ
Alert อะไรทั้งสิ้น มันแค่เงียบๆ ปล่อย Packet ไปใช้ Routing Table ปกติทันที** เพราะ Route-map
เป็น **`permit`** โดยธรรมชาติ: Clause ที่ไม่ Match = ข้ามไป Clause ถัดไป และถ้าไม่มี Clause เหลือ
= Fallthrough ไป Destination-Based Routing ตามปกติ — **นี่คือ Design ที่ตั้งใจของ Cisco (Fail-Open,
ไม่ใช่ Fail-Closed) เพื่อป้องกันไม่ให้ PBR Config ผิดพลาดทำให้ Traffic ทั้งหมด Drop ไปเลย**

```
ปัญหาที่พบจริง: ลืมพิมพ์ Subnet ผิด (10.10.2.0 แทนที่จะเป็น 10.10.20.0)

CORE-SW1(config)# ip access-list extended VOICE-TO-INTERNET
CORE-SW1(config-ext-nacl)# permit ip 10.10.2.0 0.0.0.255 any     ← ผิด! ไม่มี VLAN 2 อยู่จริง
                                                                     (ตั้งใจจะพิมพ์ 10.10.20.0)

ผลลัพธ์: VOICE Traffic จริง (10.10.20.0/24) ไม่ Match ACL เลยสักบรรทัด
         → PBR ไม่ทำงานเลย 100% → Traffic ไหลตาม OSPF/BGP ปกติเหมือนไม่มี PBR อยู่เลย
         → show route-map จะเห็น "Policy routing matches: 0 packets" ค้างนิ่งตลอด
         → ไม่มี Error Message ใดๆ ปรากฏใน Log เลย — ต้องสังเกตด้วยตัวเองเท่านั้น!
```

**วิธี Troubleshoot ที่ถูกต้อง**: ทุกครั้งที่ PBR "ดูเหมือนไม่ทำงาน" ให้ตรวจ **`show route-map`
ดู Hit Counter ก่อนเป็นอันดับแรกเสมอ** — ถ้า `Policy routing matches: 0 packets` นิ่งอยู่แม้จะมี
Traffic วิ่งจริง แปลว่า **ปัญหาอยู่ที่ ACL ไม่ Match ไม่ใช่ที่ Route-map หรือ Set Command** ให้ไป
ตรวจ `show access-lists <name>` ต่อว่าบรรทัดไหน Match ตามที่ตั้งใจจริงหรือไม่

### 319.4 ลำดับ Troubleshooting มาตรฐานสำหรับ PBR

```
1. show ip policy                    → ยืนยันว่า Interface ที่ถูกต้องผูก Route-map ที่ถูกต้อง
2. show route-map <name>             → ดู Hit Counter ต่อ Clause (0 = ACL ไม่ Match เลย)
3. show access-lists <acl-name>      → ตรวจ Hit Counter แยกบรรทัด permit/deny ว่า Match ถูกบรรทัดไหม
4. show track <object>               → ถ้าใช้ verify-availability ตรวจว่า Track เป็น Up หรือ Down
5. show ip sla statistics            → ตรวจว่า IP SLA Probe เอง Success หรือ Fail (Root Cause ของ Track)
6. debug ip policy (ระวัง CPU)        → ดู Real-time Packet ที่ถูก PBR จับ ยืนยัน Next-Hop ที่ใช้จริง
```

---

## Step 320 — Lab เต็มรูปแบบ: PBR + IP SLA Failover บน CORE-SW1

### 320.1 เป้าหมาย Lab

รวมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกันเป็น Deployment เดียวที่ใช้งานได้จริงบน **CORE-SW1**:

1. **VOICE (VLAN 20)** ที่จะออก Internet ต้องวิ่งผ่าน **WAN-EDGE-1 เป็นหลักเสมอ** (Primary)
2. ถ้า WAN-EDGE-1 ล่ม (ตรวจสอบด้วย **IP SLA**) ให้ **Failover ไปยัง WAN-EDGE-2 โดยอัตโนมัติ**
   (ผ่าน CORE-SW2) แทนที่จะปล่อยตาม Routing Table เฉยๆ
3. **SALES (VLAN 10)** ไม่ถูกแตะต้อง — ใช้ OSPF/BGP เลือกเส้นทางตามปกติ
4. **Traffic ภายในองค์กร** (10.10.0.0/16 ↔ 10.10.0.0/16) ของทั้งสอง VLAN ไม่ถูก PBR แตะต้อง

### 320.2 Running-Config เต็มรูปแบบบน CORE-SW1

```
! ==========================================================
! PART 32 — POLICY-BASED ROUTING (PBR) — CORE-SW1 FULL CONFIG
! ==========================================================

hostname CORE-SW1

! ---------- 1) IP SLA Probes: ตรวจสอบทั้ง WAN-EDGE-1 และ WAN-EDGE-2 ----------
ip sla 1
 icmp-echo 10.10.254.1 source-interface GigabitEthernet1/0/1
 frequency 5
 timeout 1000
 threshold 800
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 10.10.254.6 source-interface Port-channel1
 frequency 5
 timeout 1000
 threshold 800
ip sla schedule 2 life forever start-time now

! ---------- 2) Track Objects ผูกกับแต่ละ SLA ----------
track 1 ip sla 1 reachability
 delay down 2 up 5
track 2 ip sla 2 reachability
 delay down 2 up 5

! ---------- 3) Extended ACL: แยก Traffic ภายในออกจาก Traffic ที่จะออก Internet ----------
ip access-list extended VOICE-TO-INTERNET
 remark ** Internal VOICE traffic ไม่ต้อง Policy Route (ใช้ OSPF ปกติ) **
 deny   ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255
 remark ** VOICE traffic ที่เหลือทั้งหมด (มุ่งหน้า Internet) ให้ Policy Route **
 permit ip 10.10.20.0 0.0.0.255 any

! ---------- 4) Route-map หลัก: Primary = WAN-EDGE-1, Backup = WAN-EDGE-2 (ผ่าน CORE-SW2) ----------
route-map PBR-VOICE permit 10
 match ip address VOICE-TO-INTERNET
 set ip next-hop verify-availability 10.10.254.1 10 track 1
 set ip next-hop verify-availability 10.10.254.6 20 track 2

! ---------- 5) Apply PBR เข้า Interface ขาเข้าจาก DIST-SW1 / DIST-SW2 ----------
interface TenGigabitEthernet1/0/1
 description ** Uplink to DIST-SW1 (SALES/VOICE block) **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip policy route-map PBR-VOICE
 ip ospf 1 area 0

interface TenGigabitEthernet1/0/2
 description ** Uplink to DIST-SW2 (SALES/VOICE block) **
 no switchport
 ip address 10.255.20.1 255.255.255.252
 ip policy route-map PBR-VOICE
 ip ospf 1 area 0

! ---------- 6) Interface ไปยัง WAN-EDGE-1 (ไม่ต้อง Apply PBR ขาเข้า — Traffic ฝั่งนี้เป็น Return ----------
interface GigabitEthernet1/0/1
 description ** Uplink to WAN-EDGE-1 (Primary Internet Edge) **
 ip address 10.10.254.2 255.255.255.252
 ip ospf 1 area 0

! ---------- 7) Backbone ไปยัง CORE-SW2 (เส้นทาง Backup สู่ WAN-EDGE-2) ----------
interface Port-channel1
 description ** Backbone LACP to CORE-SW2 (also PBR backup path to WAN-EDGE-2) **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf 1 area 0
```

> **หมายเหตุ Design**: `set ip next-hop verify-availability 10.10.254.6 20 track 2` ใช้ IP
> `10.10.254.6` ซึ่งเป็น**ฝั่ง CORE-SW2** ของ Link ไปหา WAN-EDGE-2 (ไม่ใช่ Interface ที่ต่อกับ
> CORE-SW1 โดยตรง) — CORE-SW1 **Resolve เส้นทางนี้แบบ Recursive Lookup** ผ่าน OSPF ที่เรียนรู้
> Network `10.10.254.4/30` มาจาก CORE-SW2 ทาง Port-channel1 อยู่แล้ว (ดู [Part
> 11](part-011-ospfv2-fundamentals.md)) — นี่คือตัวอย่างจริงของ 312.4 ที่ Next-Hop ของ PBR
> ไม่จำเป็นต้อง Directly-Connected เสมอไป ขอแค่มี Route ไปถึงในตาราง Routing

### 320.3 Verify ก่อนทดสอบ Failover — ทุกอย่างทำงานปกติ (WAN-EDGE-1 Up)

```
CORE-SW1# show track 1
Track 1
  IP SLA 1 reachability
  Reachability is Up
    2 changes, last change 00:42:10
  Delay up 5 secs, down 2 secs
  Tracked by:
    Route Map PBR-VOICE 10

CORE-SW1# show track 2
Track 2
  IP SLA 2 reachability
  Reachability is Up
    1 change, last change 01:15:33
  Delay up 5 secs, down 2 secs
  Tracked by:
    Route Map PBR-VOICE 10

CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  Match clauses:
    ip address (access-lists): VOICE-TO-INTERNET
  Set clauses:
    ip next-hop verify-availability 10.10.254.1 10 track 1  [up]
    ip next-hop verify-availability 10.10.254.6 20 track 2  [up]
  Policy routing matches: 8842 packets, 1157104 bytes

CORE-SW1# show ip policy
Interface                Route map
Te1/0/1                  PBR-VOICE
Te1/0/2                  PBR-VOICE
```

`[up]` หลัง Next-Hop แรก (`10.10.254.1`) ยืนยันว่า Track Object 1 เป็น Up และเป็นตัวที่ถูกเลือก
ใช้งานจริง (Sequence 10 มาก่อน Sequence 20 เสมอเมื่อทั้งคู่ Up)

### 320.4 ทดสอบ Failover — จำลอง WAN-EDGE-1 ล่ม

```
! จำลอง Interface ไปยัง WAN-EDGE-1 หลุด
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# shutdown
CORE-SW1(config-if)# end

! รอประมาณ 5-10 วินาที (IP SLA Frequency 5s + Track delay down 2s)
CORE-SW1#
%TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
%LINK-5-CHANGED: Interface GigabitEthernet1/0/1, changed state to administratively down

CORE-SW1# show track 1
Track 1
  IP SLA 1 reachability
  Reachability is Down
    3 changes, last change 00:00:08
  Delay up 5 secs, down 2 secs
  Tracked by:
    Route Map PBR-VOICE 10

CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  Match clauses:
    ip address (access-lists): VOICE-TO-INTERNET
  Set clauses:
    ip next-hop verify-availability 10.10.254.1 10 track 1  [down]
    ip next-hop verify-availability 10.10.254.6 20 track 2  [up]
  Policy routing matches: 9130 packets, 1198320 bytes
```

**ผลลัพธ์**: Next-Hop แรก (`10.10.254.1`) เปลี่ยนเป็น `[down]` ทันที — PBR **ข้ามไปใช้ Next-Hop
ลำดับถัดไปที่ยัง `[up]` โดยอัตโนมัติ (`10.10.254.6` ผ่าน CORE-SW2 ไปยัง WAN-EDGE-2)** โดยไม่ต้อง
แก้ Config ใดๆ เพิ่มเติม — Voice Traffic **ไม่ถูก Black-Hole** และยังคงออก Internet ได้ต่อเนื่อง
เพียงแค่เปลี่ยนไปใช้ WAN-EDGE-2 แทนชั่วคราว

```
CORE-SW1# debug ip policy
IP: s=10.10.20.15 (TenGigabitEthernet1/0/1), d=8.8.8.8, len 200, policy match
IP: route map PBR-VOICE, item 10, permit
IP: s=10.10.20.15 (TenGigabitEthernet1/0/1), d=8.8.8.8 (Port-channel1), len 200, policy routed
IP: TenGigabitEthernet1/0/1 to Port-channel1 10.10.254.6
CORE-SW1# undebug all
```

ยืนยันชัดเจนว่า Next-Hop ที่ใช้จริงตอนนี้คือ `10.10.254.6` ผ่าน `Port-channel1` (เส้นทางไป
CORE-SW2 → WAN-EDGE-2) แทนที่จะเป็น Gi1/0/1 เดิม

### 320.5 คืนสถานะ — Failback อัตโนมัติเมื่อ WAN-EDGE-1 กลับมา

```
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# end

! รอ 5 วินาที (delay up 5) หลัง IP SLA เริ่มเห็นการตอบสนองอีกครั้ง
CORE-SW1#
%LINK-5-CHANGED: Interface GigabitEthernet1/0/1, changed state to up
%TRACK-6-STATE: 1 ip sla 1 reachability Down -> Up

CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  Match clauses:
    ip address (access-lists): VOICE-TO-INTERNET
  Set clauses:
    ip next-hop verify-availability 10.10.254.1 10 track 1  [up]
    ip next-hop verify-availability 10.10.254.6 20 track 2  [up]
  Policy routing matches: 9587 packets, 1254112 bytes
```

Next-Hop แรก (`10.10.254.1`, Sequence 10) กลับมาเป็น `[up]` — เพราะ Sequence 10 มาก่อน
Sequence 20 เสมอ PBR จึง **กลับไปใช้ WAN-EDGE-1 เป็น Primary โดยอัตโนมัติทันที** (ไม่ต้องมี
`preempt` แบบ HSRP — PBR `verify-availability` เลือก Next-Hop ที่ Sequence ต่ำสุดที่ Up เสมอ
โดยธรรมชาติของ Syntax เอง)

### 320.6 ยืนยันว่า SALES (VLAN 10) และ Internal VOICE ไม่ได้รับผลกระทบ

```
! Hit Counter ของ ACL แยกบรรทัด — ยืนยันว่า Internal Traffic (deny) ถูกแยกออกถูกต้อง
CORE-SW1# show access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (632 matches)
    20 permit ip 10.10.20.0 0.0.0.255 any (9587 matches)

! SALES (VLAN 10) ไม่มี ACL ใดอ้างอิงเลย — ยืนยันว่าไม่ถูก PBR แตะต้องโดย show ip policy ไม่มี
! Route-map ใดที่ match VLAN 10 อยู่เลย (มีแค่ PBR-VOICE ที่ match เฉพาะ 10.10.20.0/24)
CORE-SW1# show ip route 10.10.10.0
Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 2
  Routing Descriptor Blocks:
  * 10.255.10.2, from 1.1.1.11, 00:12:44 ago, via TenGigabitEthernet1/0/1
```

`632 matches` บนบรรทัด `deny` ยืนยันว่า Traffic VOICE ที่คุยกันภายในองค์กร (เช่นกับ Call
Manager ใน Data Center ภายใน) ถูกแยกออกจาก PBR อย่างถูกต้องและใช้ OSPF ตามปกติ ในขณะที่ Traffic
มุ่งหน้า Internet จริง (`9587 matches` บน `permit`) ถูก Policy Route ตาม Design ทั้งหมด — **Lab
สมบูรณ์ตามเป้าหมายทั้ง 4 ข้อใน 320.1**

---

## แบบฝึกหัดทวนความเข้าใจ Part 32

1. เพราะเหตุใด `ip policy route-map` จึง Apply ได้เฉพาะทิศทาง Inbound เท่านั้น ไม่มี Outbound ให้เลือกเหมือน ACL?
2. ในบรรทัด ACL ของ PBR (`match ip address`) ถ้า Packet Match บรรทัดที่เป็น `deny` จะเกิดอะไรขึ้น — Packet ถูก Block ไปเลยหรือไม่?
3. `set ip next-hop` ธรรมดา (ไม่มี `verify-availability`) มีความเสี่ยงอะไรที่ร้ายแรงที่สุด และแก้ด้วยอะไร?
4. ถ้าตรวจสอบด้วย `show route-map` แล้วพบว่า `Policy routing matches: 0 packets` นิ่งอยู่ตลอดแม้จะมี Traffic วิ่งจริง สาเหตุที่เป็นไปได้มากที่สุดคืออะไร และควรตรวจอะไรต่อ?
5. เพราะเหตุใด Local PBR (`ip local policy route-map`) จึงต้อง Config ที่ Global Configuration Mode แทนที่จะ Apply เข้า Interface เหมือน PBR ปกติ?

**เฉลย:**

1. เพราะ PBR ต้องตัดสินใจเส้นทาง**ก่อน**ที่ Router จะเลือก Output Interface เอง — ถ้า Apply ตอน
   Packet กำลังจะออก (Outbound) การตัดสินใจ Forward ก็เกิดขึ้นไปแล้ว ทำให้ PBR ไม่มีความหมายอีกต่อไป
2. **ไม่ถูก Block** — บรรทัด `deny` ใน ACL ที่ใช้กับ `match ip address` ของ PBR หมายถึง
   "ไม่ต้อง Policy-Route Packet นี้" เท่านั้น Packet จะ Fallthrough ไปใช้ Routing Table ปกติ
   (Destination-Based Routing) แทน ไม่ใช่การทิ้ง Packet ทิ้ง
3. ความเสี่ยงคือ **Black-Hole** — ถ้า Next-Hop ที่ตั้งไว้ตายไป PBR จะยังพยายามส่ง Packet ไปที่นั่น
   ต่อไปเรื่อยๆ ทำให้ Packet ถูก Drop ทั้งหมด แม้จะมีเส้นทางสำรองอื่นที่ใช้งานได้ก็ตาม — แก้ด้วย
   `set ip next-hop verify-availability <ip> <seq> track <object>` ร่วมกับ IP SLA + Track Object
4. สาเหตุที่พบบ่อยที่สุดคือ **ACL ที่ `match ip address` อ้างอิงอยู่เขียนผิด** (Subnet/Wildcard
   Mask ผิด) จนไม่ Match Traffic จริงเลยสักบรรทัด — ควรตรวจ `show access-lists <name>` ต่อว่า
   Hit Counter ของแต่ละบรรทัดตรงตามที่ตั้งใจหรือไม่
5. เพราะ Traffic ที่ Router **สร้างขึ้นเอง** (Syslog, SSH, NTP, Ping จาก Router) **ไม่เคยผ่าน
   Interface ขาเข้าใดๆ เลย** — มัน Originate ขึ้นภายใน Router โดยตรง จึงไม่มี "Interface ขาเข้า"
   ให้ Apply Policy ได้ ต้องใช้คำสั่ง Global แทนเพื่อครอบคลุม Traffic ทั้งหมดที่ Router สร้างขึ้นเอง

---

## สรุป Part 32

Part นี้แนะนำ **Policy-Based Routing (PBR)** — เทคนิคที่ทำให้ Router เลือกเส้นทางจากเกณฑ์อื่น
นอกเหนือจาก Destination IP ได้ ตั้งแต่ Syntax พื้นฐาน (`route-map` + `match ip address` +
`set ip next-hop` + `ip policy route-map`), ตัวอย่างจริงบน **CORE-SW1** ที่บังคับ **VLAN 20
(VOICE)** ให้ออก **WAN-EDGE-1** เสมอเพื่อ Latency ที่ต่ำกว่า, การป้องกัน Black-Hole ด้วย
`set ip next-hop verify-availability` ร่วมกับ **IP SLA + Track Object**, การ Match ตามขนาด
Packet และ DSCP Marking, Local PBR สำหรับ Traffic ที่ Router สร้างเอง, ข้อพิจารณาด้าน
Performance บน Catalyst 9000 (Hardware TCAM), ไปจนถึง Lab เต็มรูปแบบที่รวม **Primary/Backup
Internet Edge Failover ด้วย PBR + IP SLA** อย่างสมบูรณ์พร้อม Verification และการทดสอบ Failover
จริง ✅ **พร้อมสำหรับ Part 33**: เราจะกลับไปที่ Campus Design ระดับลึกขึ้นด้วย **Advanced STP &
Campus Design** — ขยาย Topology เป็น **Dual-Homed DIST↔CORE เต็มรูปแบบ** ตามที่
[00-ip-address-plan.md](00-ip-address-plan.md) ได้ประกาศไว้ล่วงหน้าตั้งแต่ Part 10 ว่าจะเพิ่ม
เข้ามาใน Part 33-35 ของระดับ CCNP ENCOR

**ไปต่อ:** [Part 33 — Advanced STP & Campus Design →](part-033-advanced-stp-campus-design.md)
