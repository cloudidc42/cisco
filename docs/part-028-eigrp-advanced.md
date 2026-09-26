# Part 28 — EIGRP Advanced
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 271–280 จาก 1000**

> ต่อจาก [Part 27 — OSPFv3 for IPv6](part-027-ospfv3-ipv6.md) ที่ปิดท้ายการปูพื้นฐาน OSPF ทั้ง
> OSPFv2 (IPv4) และ OSPFv3 (IPv6) บน Lab Topology หลักของหลักสูตร

## Part นี้ยังคงอยู่บน Standalone Demo Topology เดิมจาก Part 13

**ข้อสำคัญที่ต้องเข้าใจก่อนเริ่ม Part นี้เหมือนเดิม**: หลักสูตรนี้ใช้ **OSPF เป็น IGP หลัก** ของ
Enterprise Lab Topology มาตลอดตั้งแต่ Part 11 และจะใช้ต่อไปจนจบหลักสูตร Part นี้**ไม่ได้เปลี่ยน
IGP หลักเป็น EIGRP** แต่อย่างใด — เป็นการกลับมาต่อยอดเนื้อหา EIGRP ระดับ CCNP ENCOR บน
**Standalone Demo Topology เดิมทุกประการจาก [Part 13 — EIGRP Fundamentals](part-013-eigrp-fundamentals.md)**
คือ 4 อุปกรณ์ **DIST-SW3, DIST-SW4** (ใช้ Port สำรองที่ไม่แตะ OSPF/SVI ของจริง) และ Legacy Router
**EIGRP-R1, EIGRP-R2** ที่จำลอง Site ที่ถูกควบรวมเข้ามาใหม่ (M&A) — Address Plan, Bandwidth/Delay
ที่ตั้งไว้ใน Part 13 (Link1-Link5) **ใช้ค่าเดิมทั้งหมด ไม่มีการเปลี่ยนแปลง** Part นี้เพียงแค่ **เพิ่ม
Feature ระดับ Advanced** เข้าไปบน Config เดิมที่ Part 13 วางไว้แล้ว (Named Mode + MD5 Authentication
+ Summarization จาก Step 130)

```
                          EIGRP-R1  (Legacy Site "BKK-OLD" #1)
                          Lo0: 9.9.9.1/32
                         ┌─────────┴─────────┐
                Link1    │                   │   Link2
        10.199.34.0/30   │                   │  10.199.34.4/30
     BW=100,000 DLY=100  │                   │  BW=1,000,000 DLY=10   (GigE เต็มสปีด)
                   ┌──────┴──────┐     ┌──────┴──────┐
                   │  DIST-SW3   │     │  EIGRP-R2    │  Lo0: 9.9.9.2/32
                   │ (จาก Part10)│     │(Legacy Site  │  Lo1: 10.199.200.1/24
                   └──┬───────┬──┘     │  "BKK-OLD"#2)│
             Link4     │       │Link5  └──┬────────┬──┘
        10.199.34.12/30│       └──────────┘        │
       BW=1,000,000    │      10.199.34.16/30      │ Link3
       DLY=10 (GigE)   │      BW=100,000 DLY=100    │ 10.199.34.8/30
                        │                            │ BW=1,544 DLY=20,000
                   ┌────┴────────┐                  │ (จำลอง Serial WAN)
                   │  DIST-SW4    │──────────────────┘
                   │ (จาก Part10) │
                   └──────────────┘
```

จาก Part 13 เราคำนวณไว้แล้วว่า DIST-SW3 ไปยัง `10.199.200.0/24` (Lo1 ของ EIGRP-R2) มี **Successor
ผ่าน Link5 metric 156,160**, **Feasible Successor ผ่าน Link1→Link2 metric 156,416**, และ **Path
ผ่าน Link4→Link3 metric 2,298,112 ที่ไม่ผ่าน Feasibility Condition** — ตัวเลขชุดนี้จะถูกใช้อ้างอิงซ้ำ
ตลอด Part นี้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 271 | EIGRP Named Mode เชิงลึก — `af-interface default` vs Per-Interface Override, แนวคิด Topology-base |
| 272 | EIGRP Stub Routing — ป้องกันไม่ให้ Site เล็กกลายเป็น Transit Path |
| 273 | EIGRP Route Filtering ด้วย `distribute-list` (ACL / Prefix-list / Route-map) |
| 274 | EIGRP Summarization ระดับ Enterprise — Multiple Summary Point, Stub Interaction, `leak-map` |
| 275 | EIGRP SIA (Stuck-In-Active) — สาเหตุ, `timers active-time`, และการออกแบบป้องกัน |
| 276 | EIGRP over WAN — `bandwidth-percent` ป้องกัน EIGRP กิน Bandwidth WAN ความเร็วต่ำ |
| 277 | EIGRP Authentication เชิงลึก — Key Rotation ด้วย `accept-lifetime`/`send-lifetime` |
| 278 | EIGRP + BFD — ตรวจจับ Link ล่มระดับ Sub-second |
| 279 | Verification ขั้นสูง — `topology all-links`, `topology active`, `show ip eigrp accounting` |
| 280 | Lab เต็มรูปแบบ: Stub + Summarization + Leak-map + BFD บน Standalone Topology |

---

## Step 271 — EIGRP Named Mode เชิงลึก: `af-interface default` vs Per-Interface Override, แนวคิด Topology-base

Part 13 Step 126 แนะนำโครงสร้าง Named Mode ไปแล้วแบบคร่าวๆ Step นี้จะลงลึกใน**3 ระดับ Scope**
ที่ Named Mode แบ่งไว้ ซึ่งเป็นจุดที่ข้อสอบ CCNP ENCOR ชอบถามเรื่องลำดับความสำคัญ (Precedence)

### 3 ระดับ Scope ของ Named Mode

| Level | Submode Prompt | เก็บ Command ประเภทไหน |
|---|---|---|
| **Address-Family (Global)** | `(config-router-af)#` | `network`, `eigrp router-id`, `eigrp stub`, `maximum-paths`, `neighbor` |
| **Per-Interface** | `(config-router-af-interface)#` | `hello-interval`, `hold-time`, `authentication`, `split-horizon`, `bandwidth-percent`, `bfd`, `summary-address` |
| **Topology** | `(config-router-af-topology)#` | `variance`, `distance`, `distribute-list`, `redistribute`, `timers active-time`, `summary-metric` |

### `af-interface default` — ตั้งค่า Baseline ให้ทุก Interface ในคราวเดียว

`af-interface default` คือ Pseudo-Interface พิเศษที่ไม่ได้อ้างถึง Interface จริงตัวใดตัวหนึ่ง แต่เป็น
**ค่า Default ที่ทุก Interface ที่เปิด EIGRP รับไปใช้อัตโนมัติ** เว้นแต่จะมี `af-interface <ชื่อจริง>`
มา Override เฉพาะ Parameter นั้นทีหลัง — หลักการ **"เฉพาะเจาะจงชนะค่า Default เสมอ แต่ทีละ
Parameter"** (การ Override เพียง 1 Parameter บน Interface หนึ่ง **ไม่ได้ลบ** Parameter อื่นที่ยัง
สืบทอดมาจาก `default` อยู่)

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# af-interface default
DIST-SW3(config-router-af-interface)# hello-interval 5
DIST-SW3(config-router-af-interface)# hold-time 15
DIST-SW3(config-router-af-interface)# authentication mode md5
DIST-SW3(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW3(config-router-af-interface)# exit-af-interface
!
! Link4 (DIST-SW3 <-> DIST-SW4) เป็น Backbone ระหว่างสวิตช์ 2 ตัว เชื่อถือได้สูง
! ต้องการ Convergence เร็วกว่า Link ไปหา Legacy Router — Override เฉพาะ Timer
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/24
DIST-SW3(config-router-af-interface)# hello-interval 1
DIST-SW3(config-router-af-interface)# hold-time 3
DIST-SW3(config-router-af-interface)# exit-af-interface
DIST-SW3(config-router-af)# exit-address-family
```

**ผลลัพธ์**: ทุก Interface ที่เปิด EIGRP บน DIST-SW3 (Gi1/0/23, Gi1/0/24, Gi1/0/25) ได้ Hello 5/Hold 15
วินาที **และ** MD5 Authentication ด้วย Key-chain `EIGRP-KEY` มาจาก `default` ทั้งหมด — **ยกเว้น**
Gi1/0/24 ที่ Hello/Hold ถูก Override เป็น 1/3 วินาที แต่ **ยังคง MD5 Authentication ที่สืบทอดมาจาก
`default` อยู่เหมือนเดิม** (ไม่ได้หายไปเพราะเราไม่ได้พิมพ์ `authentication` ซ้ำใน Block ของ Gi1/0/24)

> **ข้อควรระวังเรื่อง `split-horizon`**: EIGRP เปิด Split-Horizon เป็น Default บนทุก Interface
> (ป้องกันการ Advertise Route ย้อนกลับออก Interface เดียวกับที่เรียนรู้มา) ซึ่งเหมาะกับ Point-to-Point
> Link ทุกเส้นใน Topology นี้ — **แต่ในสถาปัตยกรรม Hub-and-Spoke แบบ Multipoint** (เช่น DMVPN Hub
> ที่ Spoke หลายตัวใช้ Physical Interface เดียวกัน) Hub ต้อง `no split-horizon` ใต้ `af-interface`
> ของ Interface นั้น ไม่เช่นนั้น Hub จะไม่ยอม Advertise Route ที่เรียนจาก Spoke ตัวหนึ่งกลับออกไปให้
> Spoke ตัวอื่นที่อยู่ Interface เดียวกันได้เลย (ข้อสอบ CCNP ชอบออกโจทย์สถานการณ์นี้)

### แนวคิด `topology base` — ทำไมต้องมี Submode นี้

`topology base` คือ **Topology ID 0** ซึ่งเป็น Topology เดียวที่ใช้งานจริงในทุก Deployment ปัจจุบัน —
ในอดีต EIGRP เคยรองรับ **Multi-Topology Routing (MTR)** ที่สร้างหลาย Topology แยกกันได้ (เช่น
Topology "VOICE" ที่ให้น้ำหนัก Metric ต่างจาก Topology "DATA" บน Interface เดียวกัน) แต่ Feature นี้
ถูก Cisco ถอดออกจาก IOS-XE รุ่นใหม่ไปแล้ว (ใช้ Class-Based/Policy-Based QoS แทน) — Syntax
`topology base` ยังคงอยู่เพื่อความเข้ากันได้ (Backward Compatibility) ของโครงสร้าง CLI เท่านั้น
**ทุก Command ที่เกี่ยวกับการคำนวณ Route ระดับ Topology** (Step 128's `variance`, Step 273's
`distribute-list`, Step 275's `timers active-time`) **ต้องพิมพ์ในนี้เสมอ** ไม่ว่าจะมี Topology เดียว
หรือหลาย Topology ก็ตาม

```
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# variance 2
DIST-SW3(config-router-af-topology)# exit-af-topology
```

### Verify ผลของ `af-interface default` + Override

```
DIST-SW3# show ip eigrp interfaces detail GigabitEthernet1/0/23
GigabitEthernet1/0/23    1    0/0     50    0/1    0/0
  Hello interval is 5 sec, Hold time is 15 sec
  Next xmit serial <none>
  Authentication mode is md5, key-chain is "EIGRP-KEY"

DIST-SW3# show ip eigrp interfaces detail GigabitEthernet1/0/24
GigabitEthernet1/0/24    1    0/0     50    0/1    0/0
  Hello interval is 1 sec, Hold time is 3 sec      <- Override สำเร็จ
  Authentication mode is md5, key-chain is "EIGRP-KEY"    <- สืบทอดจาก default ยังอยู่
```

---

## Step 272 — EIGRP Stub Routing: ป้องกันไม่ให้ Site เล็กกลายเป็น Transit Path

### Stub Routing คืออะไร และแก้ปัญหาอะไร

**Stub Routing** คือกลไกที่ให้ Router ที่เป็น "ปลายทาง" ของ Network (Remote Site, Spoke,
Legacy Site ที่มี Bandwidth/CPU จำกัด และไม่มี Router อื่นพึ่งพาเป็นทางผ่าน) **ประกาศตัวเองว่าเป็น
Stub** ผ่าน Flag พิเศษใน Hello Packet ผลที่เกิดขึ้นมี **2 ชั้น**:

1. **Stub Router เองจะไม่ Advertise Route ที่เรียนรู้จาก Neighbor ตัวหนึ่ง ไปให้ Neighbor อีกตัวหนึ่ง**
   (คุณสมบัติที่ทำให้ Router เป็น "ทางผ่าน/Transit" ได้หายไปโดยสมบูรณ์ — มันจะ Advertise ได้แค่
   Route ประเภทที่ระบุไว้ใน Keyword เท่านั้น ไม่ใช่ทุก Route ที่มันรู้จัก)
2. **Neighbor ของ Stub Router จะไม่ส่ง Query ไปหามันเลย** ตอน DUAL เข้าสถานะ Active (Step 275)
   เพราะ Neighbor รู้อยู่แล้วว่า Stub ไม่มีทางเป็น Path สำรองให้ใครได้ — ถามไปก็เสียเวลาเปล่า
   (ผลดีอย่างมากต่อการลด SIA ในเครือข่ายขนาดใหญ่)

### Syntax และความหมายของแต่ละ Keyword

```
Router(config-router-af)# eigrp stub [receive-only | connected | static | summary | redistributed]
```

| Keyword | ผลลัพธ์ |
|---|---|
| *(ไม่ใส่ Keyword ใดๆ — พิมพ์ `eigrp stub` เฉยๆ)* | ค่า Default = **`connected` + `summary`** |
| `receive-only` | รับ Route จาก Neighbor ได้ปกติ แต่ **ไม่ Advertise อะไรออกไปเลยแม้แต่บรรทัดเดียว** (ใช้เดี่ยวๆ ห้ามผสมกับ Keyword อื่น) |
| `connected` | Advertise เฉพาะ Connected Network ที่อยู่ใน `network` statement |
| `static` | Advertise Static Route ที่ถูก `redistribute static` เข้ามาใน EIGRP |
| `summary` | Advertise **เฉพาะ** Manual Summary Route ที่ตั้งเองด้วย `ip summary-address` (ดู Step 274) |
| `redistributed` | Advertise Route ที่ Redistribute มาจาก Routing Protocol อื่น |

### ตัวอย่างจริง — ตั้ง EIGRP-R2 เป็น Stub Site

EIGRP-R2 คือ Legacy Router ปลายทางของ Site "BKK-OLD" #2 ที่มี Loopback1 (`10.199.200.0/24`)
เป็น Server LAN ของตัวเองเท่านั้น — ไม่ควรถูกใช้เป็นทางผ่านระหว่าง DIST-SW3 ↔ DIST-SW4 แม้แต่กรณี
ฉุกเฉิน (Bandwidth ของ Link3 ที่จำลอง Serial WAN มีแค่ 1,544 Kbps)

```
EIGRP-R2(config)# router eigrp EIGRP-DEMO
EIGRP-R2(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R2(config-router-af)# eigrp stub connected summary
```

**Verify ที่ DIST-SW3 (Neighbor ของ EIGRP-R2 ทาง Link5)**:

```
DIST-SW3# show ip eigrp neighbors detail
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        12  00:02:10    2    100  0  9
   Version 23.0/3.0, Retrans: 0, Retries: 0
   Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
   Suppressing queries
```

บรรทัด **`Suppressing queries`** คือหลักฐานตรงตัวว่า DIST-SW3 **จะไม่ส่ง Query ไปหา EIGRP-R2 เลย**
ไม่ว่า Route ใดจะเข้าสถานะ Active ก็ตาม

### ทำไมสิ่งนี้คือ "การป้องกัน Transit Path" ตัวจริง

ลองพิจารณาสถานการณ์: **ถ้า Link4 (DIST-SW3 ↔ DIST-SW4 โดยตรง) ล่ม** และ EIGRP-R1/EIGRP-R2
**ไม่ได้** ตั้งเป็น Stub — DIST-SW3 จะยังมี Path สำรองไปหา DIST-SW4 ผ่าน **EIGRP-R1 → EIGRP-R2**
(Link1 + Link2 + Link3 รวมกัน คือเส้นทาง Legacy Site ความเร็วต่ำที่สุดในทั้ง Topology) ซึ่งจะทำให้
Traffic ระหว่าง Backbone หลักสองตัวไหลผ่าน Legacy Serial-simulated Link (1,544 Kbps) โดยไม่ตั้งใจ
— อันตรายมากในสถานการณ์จริง

เมื่อตั้ง **ทั้ง EIGRP-R1 และ EIGRP-R2 เป็น Stub (`connected summary`)** ทั้งคู่ **จะไม่มีวัน
Advertise Route ที่เรียนจาก Neighbor ฝั่งหนึ่งไปให้ Neighbor อีกฝั่งหนึ่ง** ผลคือถ้า Link4 ล่มจริง
DIST-SW3 จะ**เสีย Reachability ไปยัง DIST-SW4 ไปเลย** แทนที่จะ Transit ผ่าน Legacy Site แบบ
เงียบๆ — ซึ่งคือพฤติกรรมที่ตั้งใจ (ควรแก้ด้วยการออกแบบ Backbone Redundancy ที่เหมาะสมแทน ไม่ใช่
ปล่อยให้ Legacy Site แบกรับ Backbone Traffic โดยไม่รู้ตัว)

```
! หลังตั้ง Stub ทั้งคู่ — Topology Table ของ DIST-SW3 ไปยัง 10.199.34.12/30 (Link4 ฝั่ง DIST-SW4)
! จะไม่มี Path ผ่าน EIGRP-R1/EIGRP-R2 ปรากฏอีกต่อไป แม้จะเคยเป็น Possible Successor มาก่อนก็ตาม
DIST-SW3# show ip eigrp topology 10.199.34.12/30
IP-EIGRP (AS 100): Topology entry for 10.199.34.12/30
  1 successors, FD is 28160
  via Connected, GigabitEthernet1/0/24
```

---

## Step 273 — EIGRP Route Filtering ด้วย `distribute-list` (ACL / Prefix-list / Route-map)

### เปรียบเทียบเครื่องมือกรอง Route 3 แบบ

| Filter Type | Match ด้วย | จุดเด่น |
|---|---|---|
| **ACL** (Standard/Extended) | Network + Wildcard Mask ของ Route ปลายทาง | ตั้งง่ายที่สุด แต่หยาบ ไม่ระบุ Prefix Length ได้ |
| **Prefix-list** | Network + ควบคุม Prefix Length ด้วย `ge`/`le` | แม่นยำกว่า ACL, เป็นเครื่องมือมาตรฐานที่ Cisco แนะนำสำหรับ Routing Filter ในปัจจุบัน |
| **Route-map** | เรียก Prefix-list/ACL ได้ + เพิ่มเงื่อนไข/Action อื่นร่วมได้ (เช่น `set tag`) | ยืดหยุ่นสูงสุด ใช้เมื่อ Logic ซับซ้อนกว่าการกรองเฉยๆ |

> **ข้อควรระวัง**: `distribute-list` ที่ใช้กับ Distance-Vector Protocol อย่าง EIGRP กรองที่
> **Network ปลายทางของ Route (Destination Prefix)** ไม่ใช่ Source Address ของ Traffic จริง —
> อย่าสับสนกับ ACL ที่ใช้กรอง Data Traffic บน Interface (Part 14)

### Syntax ใน Named Mode (อยู่ใต้ `topology base`)

```
Router(config-router-af-topology)# distribute-list <acl-name-or-number> {in|out} [interface]
Router(config-router-af-topology)# distribute-list prefix <prefix-list-name> {in|out} [interface]
Router(config-router-af-topology)# distribute-list route-map <route-map-name> {in|out} [interface]
```

### ตัวอย่างจริง — DIST-SW3 ปิดไม่ให้ EIGRP-R1 เห็น Subnet ของ Link4 (Backbone Internal)

สมมติต้องการกรอง Subnet ของ Link4 (`10.199.34.12/30` — Backbone ภายในระหว่าง DIST-SW3/DIST-SW4)
ไม่ให้ไปถึง EIGRP-R1 โดยตรง (แยกจาก Summarization ใน Step 274 เพื่อสาธิต Filtering แบบเจาะจง
รายบุคคล)

**วิธีที่ 1 — ACL:**
```
DIST-SW3(config)# ip access-list standard BLOCK-LINK4
DIST-SW3(config-std-nacl)# deny 10.199.34.12 0.0.0.3
DIST-SW3(config-std-nacl)# permit any
DIST-SW3(config-std-nacl)# exit
!
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# distribute-list BLOCK-LINK4 out GigabitEthernet1/0/23
```

**วิธีที่ 2 — Prefix-list (แนะนำ):**
```
DIST-SW3(config)# ip prefix-list BLOCK-LINK4-PFX seq 5 deny 10.199.34.12/30
DIST-SW3(config)# ip prefix-list BLOCK-LINK4-PFX seq 10 permit 0.0.0.0/0 le 32
!
DIST-SW3(config-router-af-topology)# distribute-list prefix BLOCK-LINK4-PFX out GigabitEthernet1/0/23
```

**วิธีที่ 3 — Route-map (ยืดหยุ่นสุด):**
```
DIST-SW3(config)# route-map FILTER-TO-R1 deny 10
DIST-SW3(config-route-map)# match ip address prefix-list BLOCK-LINK4-PFX
DIST-SW3(config-route-map)# exit
DIST-SW3(config)# route-map FILTER-TO-R1 permit 20
DIST-SW3(config-route-map)# exit
!
DIST-SW3(config-router-af-topology)# distribute-list route-map FILTER-TO-R1 out GigabitEthernet1/0/23
```

### Verify

```
! ก่อนกรอง
EIGRP-R1# show ip route eigrp | include 10.199.34.12
D    10.199.34.12/30 [90/158720] via 10.199.34.1, 00:04:12, GigabitEthernet0/0

! หลังกรองด้วยวิธีใดวิธีหนึ่งข้างต้น
EIGRP-R1# show ip route eigrp | include 10.199.34.12
! ไม่มีบรรทัดใดปรากฏเลย — Route ถูกกรองสำเร็จ
```

> **ทิศทาง `in` vs `out`**: `out` กรอง**ก่อนส่งออก**ไปยัง Neighbor (Neighbor จะไม่รู้จัก Route นั้นเลย)
> ส่วน `in` กรอง**ตอนรับเข้ามา** (Router ยัง Advertise ต่อให้ Neighbor อื่นไม่ได้ เพราะตัวเองไม่ยอมใส่
> ลง Topology Table ตั้งแต่แรก) — เลือกทิศทางผิดเป็นสาเหตุ Troubleshoot Filtering ที่พบบ่อยที่สุด

---

## Step 274 — EIGRP Summarization ระดับ Enterprise: Multiple Summary Point, Stub Interaction, `leak-map`

### EIGRP Summarize ได้หลายจุดพร้อมกัน (ต่างจาก OSPF ที่ทำที่ ABR เท่านั้น)

เพราะ `ip summary-address eigrp` เป็นคำสั่งระดับ **Interface** (Local ล้วนๆ ไม่ผูกกับ Area Boundary
แบบ OSPF Part 12) องค์กรขนาดใหญ่จึงสามารถวาง **จุด Summarize ได้หลายตำแหน่ง** ตามแนว
Hierarchical Design ทุกจุดที่มีขอบเขต Bandwidth-rich Core ↔ Low-bandwidth Edge ต่อกัน — ยิ่งมีจุด
Summarize มาก ยิ่งจำกัดขอบเขต Query ของ SIA (Step 275) ได้ดีขึ้นเป็นทวีคูณ

ใน Topology ของเรา นอกจาก DIST-SW3 สรุป `10.199.34.0/24` ออก Link1 ไปหา EIGRP-R1 แล้ว
(Part 13 Step 130) เพิ่ม**จุดที่สอง**ที่ DIST-SW4 สรุปชุดเดียวกันออก Link3 ไปหา EIGRP-R2 ด้วย
เพื่อให้ Legacy Site มองเห็นแค่ขอบเขต Backbone เดียวกันจากทั้งสองทิศทาง:

```
DIST-SW4(config)# interface GigabitEthernet1/0/24
DIST-SW4(config-if)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0
```

### Interaction ระหว่าง Stub `summary` Keyword กับ `ip summary-address` — จุดที่มักเข้าใจผิด

**ข้อควรระวังสำคัญ**: Keyword `summary` ใน `eigrp stub summary` (Step 272) **ไม่ได้แปลว่า Router
จะสรุป Route ให้อัตโนมัติ** มันแค่**อนุญาต**ให้ Advertise Route ประเภท "Manual Summary" ออกไปได้
เท่านั้น — ถ้า Router ตัวนั้น**ไม่มี** `ip summary-address` ตั้งไว้เอง จะไม่มี Route ประเภทนี้ให้
Advertise เลยแม้จะเปิด Keyword `summary` ไว้ก็ตาม (Engineer มือใหม่มักเข้าใจผิดว่า Keyword นี้
"ซ่อนรายละเอียด" ให้อัตโนมัติ)

ตัวอย่าง — ถ้าต้องการให้ EIGRP-R2 (ที่เป็น Stub `connected summary` จาก Step 272) ยัง Advertise
Summary ของตัวเองสำหรับพื้นที่ที่จะขยายในอนาคต (`10.199.200.0/24` ปัจจุบัน + `10.199.201.0/24`
ที่วางแผนเพิ่ม) ต้องตั้ง `ip summary-address` บน EIGRP-R2 เองด้วย ไม่ใช่พึ่ง Keyword `summary`
เฉยๆ:

```
EIGRP-R2(config)# interface GigabitEthernet0/1
EIGRP-R2(config-if)# ip summary-address eigrp 100 10.199.200.0 255.255.254.0
```

### `ip summary-address eigrp ... leak-map` — เปิดช่องยกเว้นเฉพาะบาง Subnet

Summarization ที่ Step 127/130 ทำไว้มีข้อดีคือ "ซ่อนความไม่เสถียร" ของ Subnet ย่อยจาก Neighbor
ฝั่งตรงข้าม (ถ้า Subnet ย่อยหนึ่ง Flap แต่ Summary Route โดยรวมยังเสถียร Neighbor จะไม่รู้สึกอะไรเลย)
— แต่บางครั้งนี่คือข้อเสีย: มี Subnet บางเส้นที่**ต้องการให้สถานะ Reachability ที่แท้จริงปรากฏชัดเจน
แยกต่างหาก** แม้จะอยู่ในขอบเขต Summary ก็ตาม `leak-map` คือ Feature ที่แก้ปัญหานี้

```
Router(config-if)# ip summary-address eigrp <AS> <network> <mask> leak-map <route-map-name>
```

**สถานการณ์ตัวอย่าง**: DIST-SW3 ยังคงสรุป `10.199.34.0/24` ออก Link1 ไปหา EIGRP-R1 ตามปกติ แต่
Subnet ของ **Link5** (`10.199.34.16/30` — เส้นทางตรง DIST-SW3 ↔ EIGRP-R2 ที่เป็น Successor หลัก)
ต้องการให้ EIGRP-R1 มองเห็นสถานะ Up/Down ที่แท้จริงแบบเจาะจง เพื่อใช้เป็นสัญญาณ Monitoring ว่า
Backup Path ตรงยังพร้อมใช้งานอยู่หรือไม่ (ถ้าปล่อยให้ถูก Summarize ปกติ EIGRP-R1 จะไม่มีทางรู้เลยว่า
Link5 ล่มไปแล้วตราบใดที่ Subnet อื่นในขอบเขตยังมีอย่างน้อย 1 เส้นทำงานอยู่)

```
DIST-SW3(config)# ip prefix-list LEAK-LINK5 seq 5 permit 10.199.34.16/30
!
DIST-SW3(config)# route-map LEAK-EXCEPTION permit 10
DIST-SW3(config-route-map)# match ip address prefix-list LEAK-LINK5
DIST-SW3(config-route-map)# exit
!
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0 leak-map LEAK-EXCEPTION
```

**Verify ที่ EIGRP-R1:**

```
EIGRP-R1# show ip route eigrp
D    10.199.34.0/24 [90/156416] via 10.199.34.1, 00:00:08, GigabitEthernet0/0     <- Summary ปกติ
D    10.199.34.16/30 [90/28416] via 10.199.34.1, 00:00:08, GigabitEthernet0/0      <- Leak เฉพาะ Link5
D    10.199.200.0/24 [90/130816] via 10.199.34.6, 00:04:12, GigabitEthernet0/1
```

Subnet `10.199.34.16/30` ยัง**ปรากฏแยกเป็นเส้นของตัวเอง**พร้อม Metric จริง (28,416 — คำนวณตามสูตร
Step 122 ของ Part 13 สำหรับ GigabitEthernet เต็มสปีด 1 Hop) ควบคู่ไปกับ Summary Route ก้อนใหญ่
— ถ้า Link5 ล่มจริง Route นี้จะหายไปทันทีโดยไม่ต้องรอให้ Summary Route ทั้งก้อนหายไปด้วย

---

## Step 275 — EIGRP SIA (Stuck-In-Active): สาเหตุ, `timers active-time`, และการออกแบบป้องกัน

### ทวนกลไก Active State จาก Part 13

เมื่อ Successor ล่มและ**ไม่มี**Feasible Successor สำรอง Route จะเข้าสถานะ **Active** — Router
ต้องส่ง **Query** ไปหา Neighbor ทุกตัว (ยกเว้นตัวที่เพิ่งรายงานว่า Route ล่ม ตาม Split-Horizon) และ
**ต้องรอ Reply ครบทุกตัว** ก่อนจะสรุปผลและกลับสู่ Passive ได้ — **SIA (Stuck-In-Active)** เกิดเมื่อ
Neighbor อย่างน้อยหนึ่งตัว**ไม่ตอบ Reply** ภายในเวลาที่กำหนด (Active Timer, Default = 3 นาที)

### สาเหตุที่ทำให้เกิด SIA

| สาเหตุ | รายละเอียด |
|---|---|
| **Topology ใหญ่/แบนราบเกินไป** | Neighbor จำนวนมาก + หลาย Hop ต้องตอบ Reply กลับมาก่อนสรุปผลได้ ยิ่ง Hop ไกล ยิ่งใช้เวลานาน |
| **WAN Link ไม่เสถียร** | Query/Reply Packet หายระหว่างทาง (RTP ต้อง Retransmit ซ้ำ) กินเวลาไปเกือบหมด Active Timer |
| **Query "ลาม" ต่อเป็นทอดๆ (Query Chaining)** | Neighbor ที่ได้รับ Query อาจไม่มี FS ของตัวเองเช่นกัน จึงต้อง Query ต่อไปยัง Neighbor ของมันอีกที — Diffusing Computation ลามไกลเกินคาด |
| **Neighbor CPU สูงเกินไป** | Router ปลายทางประมวลผล Query ช้าเกินกว่าจะตอบทันเวลา |
| **การสื่อสารทางเดียว (One-way Communication)** | Neighbor ได้รับ Query และตอบ Reply จริง แต่ Reply หายระหว่างทางกลับ — เราไม่มีทางรู้ว่าเขาตอบแล้ว จึงรอจนครบเวลาอยู่ดี |

### กลไก SIA-Query / SIA-Reply — ความช่วยเหลือครึ่งทาง

IOS มีกลไกเสริมเพื่อลด False-Positive: เมื่อผ่านไป**ครึ่งหนึ่ง**ของ Active Timer (Default = 90 วินาที
จาก 180 วินาที) โดยยังไม่ได้ Reply ครบ Router จะส่ง **SIA-Query** ไปถาม Neighbor ที่ยังไม่ตอบว่า
"ยังอยู่ไหม ยังทำงานเรื่องนี้อยู่หรือเปล่า" — ถ้า Neighbor ตอบกลับด้วย **SIA-Reply** (ยังมีชีวิตอยู่และ
กำลังทำงานอยู่จริง) Active Timer จะถูกขยายออกไปอีกครึ่งรอบ (เกิดขึ้นได้แค่ครั้งเดียวต่อ 1 เหตุการณ์
Active) — ถ้ายังไม่มีการตอบกลับแม้แต่ SIA-Reply ก็ตาม Neighbor นั้นจะถูกประกาศว่าตายและถูก
ตัดออกจาก Topology Table ทันทีเพื่อให้ DUAL คำนวณต่อได้โดยไม่ต้องรอ

```
%DUAL-3-SIA: Route 10.199.200.0/24 stuck-in-active state in IPv4-EIGRP(100) 100. Cleaning up
```

### คำสั่งปรับ Active Timer

```
Router(config-router-af-topology)# timers active-time {<1-4294967295> minutes | disabled}
```

Default = 3 นาที `disabled` คือปิดการตรวจจับ SIA ไปเลย (**ไม่แนะนำใน Production** ใช้เฉพาะตอน
Lab/Troubleshoot ชั่วคราวเพื่อยืนยันว่าปัญหาคือ SIA จริงๆ ก่อนไปแก้ที่ต้นเหตุ)

### ทำไม Summarization + Stub คือทางแก้ที่ถูกต้อง (ไม่ใช่แค่ปรับ Timer)

- **Summarization สร้าง "Query Boundary"**: เมื่อ Router ที่ทำ Summarize (เช่น DIST-SW3 ใน
  Step 127/130) เสีย Subnet ย่อยเส้นใดเส้นหนึ่งไป มันไม่จำเป็นต้อง Query Neighbor ที่อยู่**อีกฝั่ง**
  ของ Summary Boundary เลย เพราะฝั่งนั้นเห็นแค่ Summary Route ก้อนเดียว (ที่ยังคงเสถียรตราบใดที่มี
  อย่างน้อย 1 Component Subnet ยังทำงานอยู่) — Query จึงถูก "สกัด" ไว้ที่จุด Summarize ไม่ลามข้าม
  ขอบเขตไปสร้างปัญหาที่อื่น
- **Stub ตัดวงจร Query ทั้งวง**: Neighbor ของ Stub Router (Step 272) จะไม่ส่ง Query ไปหา Stub
  เลยตั้งแต่ต้น ไม่ว่า Route ใดจะ Active ก็ตาม — ในเครือข่ายจริงที่มี Spoke นับร้อยตัวติด Hub เดียว
  การไม่ต้อง Query Spoke แม้แต่ตัวเดียวช่วยลด Query Fan-out และความเสี่ยง SIA ได้มหาศาล
- **สรุป**: Cisco แนะนำให้ **Summarize ทุกจุดที่ทำได้ + Mark ทุก Leaf Site เป็น Stub เสมอ** เป็น
  มาตรการหลักในการป้องกัน SIA — การปรับ `timers active-time` ให้นานขึ้นเป็นเพียงมาตรการรอง
  (Mitigation) ที่ซ่อนปัญหา ไม่ใช่การแก้ที่ต้นเหตุของการออกแบบ

---

## Step 276 — EIGRP over WAN: `bandwidth-percent`

### ทำไม EIGRP ต้องจำกัดตัวเองบน Link ความเร็วต่ำ

Default พฤติกรรมของ EIGRP คือใช้ Bandwidth ได้สูงสุด **50%** ของค่า `bandwidth` ที่ตั้งบน Interface
(ไม่ใช่ความเร็ว Physical จริง) สำหรับ Traffic ของตัวเอง (Hello, Update, Query, Reply/Retransmit)
กลไกนี้ป้องกันไม่ให้ EIGRP เอง "ถล่ม" Bandwidth ทั้งหมดของ Link จนไม่เหลือให้ User Traffic ตอนที่
Network เกิด Convergence ครั้งใหญ่ (Route Flapping จำนวนมาก)

```
สูตร: EIGRP Control-Traffic Cap (Kbps) = bandwidth-percent (%) × Interface Bandwidth ที่ตั้งไว้ (Kbps)
```

> **จุดสำคัญ**: สูตรใช้ค่า `bandwidth` ที่**ตั้งด้วยคำสั่ง** (จาก Step 122 ของ Part 13) ไม่ใช่
> Physical Speed จริงของสาย — ถ้าตั้ง `bandwidth` ต่ำเพื่อจำลอง WAN (แบบที่ Lab นี้ทำกับ Link3)
> Cap ของ `bandwidth-percent` จะถูกคำนวณจากค่าจำลองนั้น ไม่ใช่ค่า Gigabit จริงของสาย

### ตัวอย่างคำนวณจริงบน Link3 (DIST-SW4 ↔ EIGRP-R2, จำลอง Serial `bandwidth 1544`)

```
Default cap = 50% × 1,544 Kbps = 772 Kbps

ปัญหา: Link3 มีความเร็วรวมแค่ 1,544 Kbps — การให้ EIGRP จองไว้ถึง 772 Kbps (ครึ่งหนึ่งของ Link!)
เผื่อไว้สำหรับ Convergence Storm อาจทำให้ User Traffic ที่วิ่งบน Link เดียวกันขาดแคลน Bandwidth
อย่างรุนแรงในช่วงที่ Network กำลัง Reconverge
```

**แก้ไขด้วยการลด `bandwidth-percent`:**

```
DIST-SW4(config)# router eigrp EIGRP-DEMO
DIST-SW4(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW4(config-router-af)# af-interface GigabitEthernet1/0/24
DIST-SW4(config-router-af-interface)# bandwidth-percent 20
DIST-SW4(config-router-af-interface)# exit-af-interface
```

```
Cap ใหม่ = 20% × 1,544 Kbps = 308.8 Kbps   <- เหลือ Bandwidth ให้ User Traffic มากกว่าเดิมชัดเจน
```

> **Trade-off ที่ต้องรู้**: ตั้ง `bandwidth-percent` ต่ำเกินไปก็มีความเสี่ยงย้อนกลับ — ถ้า Link มี
> Route จำนวนมากต้อง Update พร้อมกัน (Reconvergence Storm จริง) EIGRP อาจไม่มี Bandwidth พอ
> ส่ง/Retransmit Reliable Packet (Update, Query, Reply) ได้ทันเวลา จนกลายเป็นสาเหตุของ **SIA
> (Step 275)** เสียเอง — ควรทดสอบ Worst-case Convergence Load ก่อนตั้งค่าต่ำมากในระบบจริง
>
> **ค่าที่มากกว่า 100% ก็ตั้งได้ (Range เต็ม 1-999)**: ใช้เมื่อ `bandwidth` ที่ตั้งไว้ต่ำกว่าความเร็ว
> จริงมาก (เช่นตั้งไว้ให้มีผลกับ Metric การคำนวณ Route แต่ Physical Link จริงเร็วกว่านั้นมาก)
> การตั้ง `bandwidth-percent` สูงกว่า 100% ทำให้ EIGRP ใช้ Bandwidth จริงที่มีอยู่ได้เต็มที่ขึ้น
> โดยไม่กระทบค่าที่ใช้คำนวณ Metric

### Verify

```
DIST-SW4# show ip eigrp interfaces detail GigabitEthernet1/0/24
GigabitEthernet1/0/24    1    0/0     20    0/1    0/0
  Hello interval is 5 sec, Hold time is 15 sec
  ...
  Percent of bandwidth used per EIGRP: 20
```

---

## Step 277 — EIGRP Authentication เชิงลึก: Key Rotation ด้วย `accept-lifetime`/`send-lifetime`

### ปัญหาของ Key เดียวถาวรจาก Part 13

Config MD5 ใน Part 13 Step 130 ใช้ Key เดียว (`key 1`) แบบไม่มีวันหมดอายุ — นโยบายความปลอดภัย
ระดับ Enterprise จริงมักกำหนดให้ **หมุนเวียน Key เป็นระยะ** (เช่น ทุก 90 วัน) โดย**ห้าม Adjacency
หลุดระหว่างการเปลี่ยน Key** — `accept-lifetime` และ `send-lifetime` คือกลไกที่ทำให้ทำได้จริง

### ความหมายของ `accept-lifetime` และ `send-lifetime`

| Command | ความหมาย |
|---|---|
| `send-lifetime` | ช่วงเวลาที่ Router ตัวนี้**อนุญาตให้ใช้ Key นี้ส่ง (Originate)** Packet ที่มี Authentication |
| `accept-lifetime` | ช่วงเวลาที่ Router ตัวนี้**ยอมรับ Key นี้สำหรับ Packet ที่รับเข้ามา** |

การซ้อนทับ (Overlap) ของช่วงเวลาระหว่าง Key เก่ากับ Key ใหม่คือหัวใจของการหมุนเวียนแบบไม่มี
Downtime — ในช่วง Overlap ทั้งสองฝั่งจะมี**อย่างน้อย 1 Key ที่ตรงกัน**เสมอ แม้จะ Deploy Config
ไม่พร้อมกันเป๊ะทุกวินาทีก็ตาม

> **ข้อกำหนดเบื้องต้น**: Lifetime คำนวณจาก **นาฬิกาจริงของ Router** — ต้อง Sync เวลาด้วย **NTP**
> (Part 17) ให้แม่นยำทุกตัวก่อนใช้ Feature นี้ มิฉะนั้น Clock Drift คือสาเหตุอันดับ 1 ที่ทำให้
> Key Rotation ทำให้ Neighbor หลุดกลางอากาศ

### Syntax และตัวอย่างจริง — หมุน Key 1 → Key 2 วันที่ 1 ต.ค. 2026 พร้อม Overlap 24 ชั่วโมง

```
DIST-SW3(config)# key chain EIGRP-KEY
DIST-SW3(config-keychain)# key 1
DIST-SW3(config-keychain-key)# key-string C1sco123!EIGRP
DIST-SW3(config-keychain-key)# accept-lifetime 00:00:00 Jan 1 2026 00:00:00 Oct 2 2026
DIST-SW3(config-keychain-key)# send-lifetime 00:00:00 Jan 1 2026 00:00:00 Oct 1 2026
DIST-SW3(config-keychain-key)# exit
DIST-SW3(config-keychain)# key 2
DIST-SW3(config-keychain-key)# key-string N3wK3y#2026Q4
DIST-SW3(config-keychain-key)# accept-lifetime 00:00:00 Oct 1 2026 infinite
DIST-SW3(config-keychain-key)# send-lifetime 00:00:00 Oct 1 2026 infinite
DIST-SW3(config-keychain-key)# exit
```
*(ต้องตั้ง Key-chain เดียวกันนี้ซ้ำทุกตัวที่เป็น EIGRP Neighbor กัน — EIGRP-R1, EIGRP-R2, DIST-SW4
เวลาต้องตรงกันเป๊ะทุกตัว)*

### ตารางไทม์ไลน์การทำงานจริง

| ช่วงเวลา | Key ที่ Router "ส่ง" | Key ที่ Router "รับได้" |
|---|---|---|
| ก่อน 1 ต.ค. 2026 00:00 | Key 1 | Key 1 |
| 1 ต.ค. – 2 ต.ค. 2026 00:00 (Overlap 24 ชม.) | **Key 2** (send-lifetime ของ Key 2 เริ่มแล้ว) | **Key 1 และ Key 2 ทั้งคู่** (accept-lifetime ของทั้งสอง Key ยัง Valid พร้อมกัน) |
| หลัง 2 ต.ค. 2026 00:00 | Key 2 | Key 2 เท่านั้น |

> **กฎการเลือก Key สำหรับส่ง**: ถ้ามีหลาย Key ที่ `send-lifetime` Valid พร้อมกันในเวลาเดียว Router
> จะเลือก **Key-ID ต่ำสุด** เป็นตัวส่งเสมอ ในตัวอย่างนี้จึงไม่มีช่วงที่ Key 1 กับ Key 2 ถูกใช้ส่งพร้อมกัน
> (Config ด้านบนตั้งใจให้ `send-lifetime` ของ Key 1 หมดพอดีตอน Key 2 เริ่ม ไม่ Overlap ฝั่ง Send —
> Overlap เกิดเฉพาะฝั่ง **Accept** เท่านั้น ซึ่งเพียงพอแล้วสำหรับการเปลี่ยนผ่านที่ไม่มี Downtime)

### Verify

```
DIST-SW3# show key chain
Key-chain EIGRP-KEY:
    key 1 -- text "C1sco123!EIGRP"
        accept lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 2 2026), valid now
        send lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 1 2026), valid now
    key 2 -- text "N3wK3y#2026Q4"
        accept lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid now
        send lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid now
```

> **คำเตือน Production**: หากช่วง Overlap ทับกันไม่พอ (แม้แค่คลาดกันไม่กี่วินาทีเพราะ Clock Drift
> หรือพิมพ์วันที่ผิด) จะเกิด "Key Rotation Gap" ที่ทำให้ Authentication ล้มเหลวทันทีตอน Cutover —
> ควรเผื่อ Overlap อย่างน้อยหลายชั่วโมงถึง 1 วันเต็มในระบบจริงเสมอ ไม่ใช่แค่ไม่กี่นาที

---

## Step 278 — EIGRP + BFD: ตรวจจับ Link ล่มระดับ Sub-second

### ทำไม Hello/Hold Timer ของ EIGRP เองไม่พอ

แม้จะปรับ Hello/Hold ให้เร็วสุดๆ ได้ตาม Step 271 (เช่น 1/3 วินาที) ก็ยังมีข้อจำกัด 2 อย่าง: (1) ยังกิน
CPU Cycle ของ Routing Process ในการประมวลผล Hello ถี่ๆ (2) ตรวจจับได้แค่ Failure ที่ทำให้
Interface Down จริงๆ — ไม่ครอบคลุมกรณี Forwarding Failure แบบ "เงียบ" ที่ Layer 1/2 ยังติดอยู่
(เช่น Fiber ขาดทางเดียว, Hardware Forwarding ผิดพลาดบางส่วน) **BFD (Bidirectional Forwarding
Detection)** คือ Protocol อิสระที่ทำงานเร็วระดับ Millisecond และแจ้งเตือน Routing Protocol
ที่ลงทะเบียนไว้ (EIGRP, OSPF, BGP, Static Route Tracking) **ทันที**โดยไม่ต้องรอ Timer ของ
Routing Protocol เองเลย

### คอนฟิก 2 ขั้นตอน

**ขั้นที่ 1 — เปิด BFD ที่ระดับ Interface พร้อม Timer ของตัวมันเอง:**

```
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# bfd interval 150 min_rx 150 multiplier 3
```

`interval`/`min_rx` หน่วย Millisecond, `multiplier` = จำนวนครั้งที่พลาดก่อนประกาศว่า Neighbor ตาย
→ เวลาตรวจจับโดยประมาณ = 150 ms × 3 = **450 ms** (เร็วกว่า Hold Time 15 วินาทีแบบเดิมมาก)

**ขั้นที่ 2 — ผูก EIGRP เข้ากับ BFD ที่ระดับ `af-interface`:**

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/23
DIST-SW3(config-router-af-interface)# bfd
DIST-SW3(config-router-af-interface)# exit-af-interface
```
*(ต้องตั้งทั้ง `bfd interval` และ EIGRP `bfd` ทั้งสองฝั่งของ Link — EIGRP-R1 ก็ต้องตั้งเหมือนกัน)*

**เปิดแบบรวบรัดให้ทุก Interface พร้อมกันด้วย `af-interface default`:**

```
DIST-SW3(config-router-af)# af-interface default
DIST-SW3(config-router-af-interface)# bfd
DIST-SW3(config-router-af-interface)# exit-af-interface
```
แล้วใช้ `no bfd` ใต้ `af-interface <ชื่อจริง>` เฉพาะตัวที่ Hardware ไม่รองรับ BFD เป็นข้อยกเว้น
(ตัวอย่างการ Override ที่ตรงกับหลักการ Step 271)

### Verify

```
DIST-SW3# show bfd neighbors
OurAddr         NeighAddr       LD/RD    RH/RS   State     Int
10.199.34.1     10.199.34.2     1/1      Up      Up        Gi1/0/23

DIST-SW3# show ip eigrp interfaces detail GigabitEthernet1/0/23
GigabitEthernet1/0/23    1    0/0     50    0/1    0/0
  Hello interval is 5 sec, Hold time is 15 sec
  ...
  BFD is enabled
```

**ผลลัพธ์เมื่อ Link ล่มจริง**: BFD ตรวจจับได้ภายใน ~450 ms และสั่งให้ EIGRP รื้อ Adjacency ทันที
เข้าสู่ DUAL ทันที — ถ้ามี Feasible Successor สำรองอยู่แล้ว (เช่น Path B จาก Step 123 Part 13)
Traffic จะสลับ Path ภายในเวลาต่ำกว่า 1 วินาที แทนที่จะต้องรอ Hold Time เต็ม 15 วินาทีแบบเดิม

---

## Step 279 — Verification ขั้นสูง: `topology all-links`, `topology active`, `show ip eigrp accounting`

### `show ip eigrp topology all-links` — ดูทุก Path รวมถึงที่ไม่ผ่าน Feasibility Condition

```
DIST-SW3# show ip eigrp topology all-links
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.13)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status

P 10.199.200.0/24, 1 successors, FD is 156160, serno 12
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
        via 10.199.34.14 (2298112/2297856), GigabitEthernet1/0/24
```

`all-links` แสดง**ทุก Path ที่รู้จัก**รวมถึง Path C (ผ่าน DIST-SW4, metric 2,298,112) ที่ไม่ผ่าน
Feasibility Condition มาตั้งแต่ Part 13 Step 123 — ใช้ตอนวางแผน `variance` หรือ Troubleshoot
ว่าทำไม Path หนึ่งไม่ถูกเลือกเป็น FS

### `show ip eigrp topology active` — เครื่องมือวินิจฉัย SIA อันดับ 1

ใช้ระหว่างที่ Route กำลังค้างอยู่ในสถานะ Active เพื่อดูว่า **Neighbor ตัวไหนที่ยังไม่ตอบ Reply**:

```
DIST-SW3# show ip eigrp topology active
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.13)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status

A 10.199.200.0/24, 1 successors, FD is 156160, Q
        1 replies, active 00:01:47, query-origin: Local origin
        via 10.199.34.14 (2298112/2297856), r, GigabitEthernet1/0/24
             Waiting for query replies from
```

| ส่วนของ Output | ความหมาย |
|---|---|
| `active 00:01:47` | ระยะเวลาที่ Route นี้ค้าง Active มาแล้ว — เทียบกับ Active Timer Default 180 วินาที (Step 275) ยิ่งใกล้ครบยิ่งเสี่ยง SIA |
| Flag `r` หน้า Neighbor | Neighbor ตัวนี้**ยังไม่ตอบ Reply** (Reply-status pending) |
| `Waiting for query replies from` | บอกตรงๆว่า Neighbor ใดคือตัวที่ทำให้ Route ค้าง — เป็นจุดเริ่มต้น Troubleshoot ที่ตรงเป้าที่สุด |

### `show ip eigrp accounting` — สรุปจำนวน Prefix แยกตามประเภทอย่างรวดเร็ว

```
DIST-SW3# show ip eigrp accounting
EIGRP-IPv4 Accounting for AS(100)/ID(1.1.1.13)
Total Prefix Count: 4     Total Redist Count: 0

Class          Prefix Count   Redist Count
Connected              2              0
Static                 0              0
Summary                1              0
Internal               1              0
External               0              0
```

ใช้ยืนยันผลของ Summarization (Step 274) และ Filtering (Step 273) อย่างรวดเร็วในเครือข่ายขนาดใหญ่
โดยไม่ต้องนับบรรทัดใน `show ip route eigrp` เอง — จำนวน `Internal` ควรลดลงเรื่อยๆ หลังทำ
Summarization ให้ครบทุกจุดตามที่ควร

---

## Step 280 — Lab เต็มรูปแบบ: Stub + Summarization + Leak-map + BFD บน Standalone Topology

Lab นี้ต่อยอดจาก Config Baseline ที่ Part 13 Step 130 วางไว้แล้ว (Named Mode + MD5 Authentication
+ Summarization พื้นฐาน) โดยเพิ่ม **3 Feature ใหม่**: **Stub Routing** บน EIGRP-R1/EIGRP-R2,
**Summarization พร้อม `leak-map`** บน DIST-SW3, และ **BFD** บนทั้ง 2 Link ที่เชื่อมต่อ Legacy Site
โดยตรง (Link1 และ Link5) — Config ที่ไม่ได้แสดงซ้ำในนี้ (Interface IP, Key-chain พื้นฐาน) ให้ถือว่า
ยังคงอยู่ตาม Part 13 Step 130 ทุกประการ

### 1) ส่วนเพิ่มเติมบน DIST-SW3

```
! --- BFD ระดับ Interface (Link1 และ Link5) ---
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# bfd interval 150 min_rx 150 multiplier 3
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface GigabitEthernet1/0/25
DIST-SW3(config-if)# bfd interval 150 min_rx 150 multiplier 3
DIST-SW3(config-if)# exit

! --- Leak-map สำหรับ Summarization (สาธิตตาม Step 274) ---
DIST-SW3(config)# ip prefix-list LEAK-LINK5 seq 5 permit 10.199.34.16/30
DIST-SW3(config)# route-map LEAK-EXCEPTION permit 10
DIST-SW3(config-route-map)# match ip address prefix-list LEAK-LINK5
DIST-SW3(config-route-map)# exit

! --- แก้ไข Summary เดิมจาก Part 13 ให้มี leak-map เพิ่ม ---
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# no ip summary-address eigrp 100 10.199.34.0 255.255.255.0
DIST-SW3(config-if)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0 leak-map LEAK-EXCEPTION
DIST-SW3(config-if)# exit

! --- ผูก BFD เข้ากับ EIGRP บน Link1 และ Link5 ---
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/23
DIST-SW3(config-router-af-interface)# bfd
DIST-SW3(config-router-af-interface)# exit-af-interface
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/25
DIST-SW3(config-router-af-interface)# bfd
DIST-SW3(config-router-af-interface)# exit-af-interface
DIST-SW3(config-router-af)# exit-address-family
```

### 2) ส่วนเพิ่มเติมบน EIGRP-R1 (ตั้งเป็น Stub + BFD ทาง Link1)

```
EIGRP-R1(config)# interface GigabitEthernet0/0
EIGRP-R1(config-if)# bfd interval 150 min_rx 150 multiplier 3
EIGRP-R1(config-if)# exit

EIGRP-R1(config)# router eigrp EIGRP-DEMO
EIGRP-R1(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R1(config-router-af)# eigrp stub connected summary
EIGRP-R1(config-router-af)# af-interface GigabitEthernet0/0
EIGRP-R1(config-router-af-interface)# bfd
EIGRP-R1(config-router-af-interface)# exit-af-interface
EIGRP-R1(config-router-af)# exit-address-family
```

### 3) ส่วนเพิ่มเติมบน EIGRP-R2 (ตั้งเป็น Stub + BFD ทาง Link5)

```
EIGRP-R2(config)# interface GigabitEthernet0/2
EIGRP-R2(config-if)# bfd interval 150 min_rx 150 multiplier 3
EIGRP-R2(config-if)# exit

EIGRP-R2(config)# router eigrp EIGRP-DEMO
EIGRP-R2(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R2(config-router-af)# eigrp stub connected summary
EIGRP-R2(config-router-af)# af-interface GigabitEthernet0/2
EIGRP-R2(config-router-af-interface)# bfd
EIGRP-R2(config-router-af-interface)# exit-af-interface
EIGRP-R2(config-router-af)# exit-address-family
```

> **หมายเหตุ**: DIST-SW4 **ไม่**ตั้งเป็น Stub เพราะเป็นสวิตช์ Distribution เต็มตัว (มี Role เท่าเทียม
> DIST-SW3) ไม่ใช่ Site ปลายทาง — Stub ใช้เฉพาะกับ **EIGRP-R1/EIGRP-R2** ซึ่งเป็น Legacy Leaf
> Router ตัวจริงตามที่ Step 272 อธิบายไว้ Key Rotation จาก Step 277 สามารถนำมาผนวกใช้ร่วมกับ
> Lab นี้ได้เช่นกัน แต่ไม่ได้แสดงซ้ำในนี้เพื่อไม่ให้ Config ยาวเกินจำเป็น

### 4) Verification เต็มรูปแบบ

**Stub Flag ทำงานถูกต้อง:**
```
DIST-SW3# show ip eigrp neighbors detail
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        12  00:05:40    2    100  0  9
   Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
   Suppressing queries
0   10.199.34.2     Gi1/0/23        13  00:05:40    1    100  0  8
   Stub Peer Advertising ( CONNECTED SUMMARY ) Routes
   Suppressing queries
```

**BFD ขึ้นครบทั้ง 2 Link:**
```
DIST-SW3# show bfd neighbors
OurAddr         NeighAddr       LD/RD    RH/RS   State     Int
10.199.34.1     10.199.34.2     1/1      Up      Up        Gi1/0/23
10.199.34.17    10.199.34.18    2/1      Up      Up        Gi1/0/25
```

**Leak-map ทำงานถูกต้อง (เห็นทั้ง Summary และ Link5 แยกกัน) ที่ EIGRP-R1:**
```
EIGRP-R1# show ip route eigrp
D    10.199.34.0/24 [90/156416] via 10.199.34.1, 00:05:50, GigabitEthernet0/0
D    10.199.34.16/30 [90/28416] via 10.199.34.1, 00:05:50, GigabitEthernet0/0
D    10.199.200.0/24 [90/130816] via 10.199.34.6, 00:05:50, GigabitEthernet0/1
```

**Accounting ยืนยันจำนวน Prefix ที่ DIST-SW3 หลังทำ Summarization ครบทุกจุด:**
```
DIST-SW3# show ip eigrp accounting
EIGRP-IPv4 Accounting for AS(100)/ID(1.1.1.13)
Total Prefix Count: 3     Total Redist Count: 0

Class          Prefix Count   Redist Count
Connected              1              0
Summary                1              0
Internal               1              0
```

**All-links ยืนยันว่า Path ผ่าน DIST-SW4/Legacy Site ไม่ปรากฏเป็นทางเลือกอีกต่อไปหลังตั้ง Stub:**
```
DIST-SW3# show ip eigrp topology 10.199.34.12/30 all-links
IP-EIGRP (AS 100): Topology entry for 10.199.34.12/30
  1 successors, FD is 28160
  via Connected, GigabitEthernet1/0/24
```

> **Lab นี้พิสูจน์ครบทุกเป้าหมายของ Part 28**: EIGRP-R1/EIGRP-R2 ไม่มีวันถูกใช้เป็น Transit Path
> ระหว่าง DIST-SW3 ↔ DIST-SW4 อีกต่อไป (Stub), Link1/Link5 Failover ได้ในระดับ Sub-second (BFD),
> Summarization ยังคงซ่อนรายละเอียดส่วนใหญ่แต่เปิดช่องให้เห็น Path สำคัญเฉพาะจุด (Leak-map) —
> ทั้งหมดยังคงอยู่**บน Port สำรองของ DIST-SW3/DIST-SW4 เท่านั้น ไม่กระทบ OSPF Design ของ Lab
> หลักแม้แต่บรรทัดเดียว** เช่นเดียวกับหลักการที่ Part 13 วางไว้

---

## แบบฝึกหัดทวนความเข้าใจ Part 28

1. `eigrp stub` มีผล 2 ชั้นต่อพฤติกรรมของ Router และ Neighbor ของมัน — อธิบายทั้ง 2 ผลนั้น และ
   อธิบายว่าเพราะเหตุใดมันจึงป้องกันไม่ให้ Router กลายเป็น Transit Path ได้จริง
2. `ip summary-address eigrp` กับ `eigrp stub summary` ต่างกันอย่างไร และทำไมการเปิด
   `eigrp stub summary` เพียงอย่างเดียวโดยไม่มี `ip summary-address` จึงไม่ทำให้เกิดการซ่อน
   รายละเอียด Subnet แต่อย่างใด?
3. SIA (Stuck-In-Active) เกิดจากอะไร และเพราะเหตุใด Summarization กับ Stub Routing จึงถูกจัดเป็น
   วิธีป้องกัน SIA ที่ถูกต้องกว่าการเพิ่มค่า `timers active-time` เพียงอย่างเดียว?
4. ถ้า Interface WAN มี `bandwidth 1544` และตั้ง `bandwidth-percent 25` EIGRP จะใช้ Bandwidth
   สูงสุดกี่ Kbps สำหรับ Traffic ของตัวเอง?
5. เพราะเหตุใด Key Rotation ด้วย `accept-lifetime`/`send-lifetime` จึงต้องมีช่วง Overlap ระหว่าง
   Key เก่ากับ Key ใหม่ และอะไรคือสาเหตุอันดับ 1 ที่ทำให้ Key Rotation ล้มเหลวในระบบจริง?

**เฉลย:**

1. ผลชั้นที่ 1: Stub Router จะไม่ Advertise Route ที่เรียนจาก Neighbor ตัวหนึ่งไปให้ Neighbor
   อีกตัวหนึ่ง (Advertise ได้เฉพาะประเภทที่ระบุด้วย Keyword เท่านั้น) — คุณสมบัติการเป็นทางผ่านจึง
   หายไปโดยสมบูรณ์ ผลชั้นที่ 2: Neighbor ของ Stub จะไม่ส่ง Query ไปหามันเลยตอน DUAL Active
   เพราะรู้อยู่แล้วว่ามันไม่มีทางเป็น Path สำรอง — สองผลนี้รวมกันทำให้ Stub Router ไม่มีวันถูกใช้
   เป็นเส้นทางเชื่อมระหว่าง Neighbor สองฝั่งของมันได้เลย ไม่ว่าทางกายภาพจะเชื่อมถึงกันจริงหรือไม่
2. `ip summary-address eigrp` คือคำสั่งที่ **สร้าง Summary Route จริง** ขึ้นมาบน Interface หนึ่ง
   ส่วน `eigrp stub summary` เป็นเพียง **การอนุญาต** ให้ Router Advertise Route ประเภท "Manual
   Summary" ที่มีอยู่แล้วออกไปได้เท่านั้น ถ้าไม่มี `ip summary-address` ตั้งไว้เลย จะไม่มี Route
   ประเภทนี้ให้ Advertise แม้จะเปิด Keyword `summary` ไว้ก็ตาม — ต้องมีทั้งสองคำสั่งควบคู่กันเสมอ
3. SIA เกิดเมื่อ Neighbor อย่างน้อยหนึ่งตัวไม่ตอบ Reply ต่อ Query ภายใน Active Timer (Default
   3 นาที) ซึ่งมีสาเหตุจาก Topology ใหญ่/แบนราบ, WAN ไม่เสถียร, Query ลามหลายทอด, CPU สูง หรือ
   การสื่อสารทางเดียว — Summarization สร้าง Query Boundary ที่สกัด Query ไม่ให้ลามข้ามขอบเขต
   และ Stub ตัดวงจร Query ทั้งหมดไปยัง Leaf Site โดยตรง ทั้งสองอย่างแก้ที่ **สาเหตุของการออกแบบ**
   ในขณะที่การเพิ่ม `timers active-time` เป็นเพียงการซ่อนปัญหาไว้ (ให้เวลารอมากขึ้น) โดยไม่ได้ลด
   ขนาด Query Fan-out ที่แท้จริงเลย
4. 25% × 1,544 Kbps = **386 Kbps**
5. ต้องมี Overlap เพราะทั้งสองฝั่งของ Adjacency อาจไม่ได้ Deploy Config การเปลี่ยน Key พร้อมกัน
   เป๊ะทุกวินาที — ช่วง Overlap รับประกันว่าจะมีอย่างน้อย 1 Key ที่ทั้งสองฝั่งใช้ร่วมกันได้เสมอระหว่าง
   Transition สาเหตุอันดับ 1 ที่ทำให้ล้มเหลวคือ **นาฬิกาของ Router ไม่ตรงกัน (Clock Drift)**
   เนื่องจากไม่ได้ Sync ด้วย NTP อย่างแม่นยำ ทำให้ Lifetime ที่คำนวณจริงคลาดเคลื่อนจนไม่มีช่วง
   Overlap เหลืออยู่จริง

---

## สรุป Part 28

Part นี้ต่อยอด EIGRP จากพื้นฐานใน Part 13 เข้าสู่ระดับ CCNP ENCOR เต็มรูปแบบ ครอบคลุม
Named Mode เชิงลึก (`af-interface default` vs Override, แนวคิด Topology-base), Stub Routing
ที่ป้องกันไม่ให้ Site เล็กกลายเป็น Transit Path, Route Filtering ด้วย `distribute-list` ทั้ง 3 รูปแบบ,
Summarization ระดับ Enterprise พร้อม `leak-map` สำหรับกรณียกเว้น, กลไกและการป้องกัน SIA
(Stuck-In-Active), การจำกัด Bandwidth ของ EIGRP เองบน WAN ด้วย `bandwidth-percent`, Key
Rotation แบบไม่มี Downtime ด้วย `accept-lifetime`/`send-lifetime`, การผสาน BFD เพื่อ Failover
ระดับ Sub-second, และคำสั่ง Verification ขั้นสูงสำหรับวินิจฉัยปัญหาจริง — ปิดท้ายด้วย Lab ที่รวม
ทุก Feature เข้าด้วยกันบน **Standalone Demo Topology เดิมจาก Part 13** โดยไม่กระทบ OSPF Design
ของ Lab หลักแม้แต่บรรทัดเดียว เช่นเดิม

✅ **พร้อมสำหรับ Part 29**: เปลี่ยนโฟกัสจาก IGP (Interior Gateway Protocol) ไปสู่ **BGP (Border
Gateway Protocol)** — Routing Protocol ที่ขับเคลื่อน Internet ทั้งใบและเป็นหัวใจของ WAN Edge/
Multi-homing ระดับ Enterprise ไปจนถึง Service Provider ในระดับ CCNP/CCIE

**ไปต่อ:** [Part 29 — BGP Fundamentals →](part-029-bgp-fundamentals.md)
