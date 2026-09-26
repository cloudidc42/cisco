# Part 58 — Advanced BGP Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 571–580 จาก 1000**

> ต่อจาก [Part 57 — Advanced OSPF Troubleshooting](part-057-ospf-troubleshooting.md) ที่ฝึกวินิจฉัย
> ปัญหา LSA/Area/Neighbor Adjacency ของ OSPF อย่างเป็นระบบ Part นี้ย้ายกลับมาที่ **BGP** —
> Routing Protocol ที่ขับเคลื่อน WAN Edge ทั้งหมดของ Lab ตาม [Part 29](part-029-bgp-fundamentals.md)
> (eBGP ระหว่าง WAN-EDGE-1/2 กับ ISP-RTR) และ [Part 30](part-030-bgp-advanced.md) (iBGP Full-Mesh
> ระหว่าง WAN-EDGE-1/2 ↔ CORE-SW1/2 พร้อม LOCAL_PREF, AS-Path Prepending, Community) — Config
> ทั้งหมดใน 2 Part นั้น**ทำงานถูกต้องสมบูรณ์** แต่ในโลกจริง BGP เป็น Protocol ที่มีจุด Fail ได้
> หลายชั้นมาก เพราะพึ่งพา TCP, พึ่งพา IGP ให้พา Next-Hop, และพึ่งพา Policy (Route-map,
> Prefix-list, Community) ที่มนุษย์พิมพ์ผิดได้เสมอ — Part นี้คือคลังกรณีศึกษา **Broken Scenario
> สไตล์ข้อสอบ CCNP ENARSI (300-410)** ที่หยิบทุกจุด Fail สำคัญของ BGP มาสร้างเป็นสถานการณ์จริง
> ให้ฝึกวินิจฉัยทีละขั้นตอน ตั้งแต่ TCP Session ยังไม่ตั้งขึ้นเลย ไปจนถึง Route ที่ตั้งเป็น Best Path
> แล้วแต่ไม่ยอมเดินทางออกไปยัง Neighbor ที่ต้องการ

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 571 | BGP Troubleshooting Methodology — Decision Tree จาก TCP Session ถึง Advertisement |
| 572 | Scenario 1 — Neighbor ค้างที่ Active/Idle เพราะ AS Number พิมพ์ผิด (Remote-AS Mismatch) |
| 573 | Scenario 2 — Neighbor ค้างที่ Active เพราะ ACL/Firewall บล็อก TCP 179 อย่างเงียบ |
| 574 | Scenario 3 — iBGP ไม่ Establish เพราะไม่มี Route ไปยัง Loopback ของ Peer (IGP ไม่พา Loopback) |
| 575 | Scenario 4 — Route ถูก Received แต่ไม่ถูกติดตั้งใน Routing Table (ลืม next-hop-self) |
| 576 | Scenario 5 — Prefix-list/AS-Path Filter บล็อก Route โดยไม่ตั้งใจ (Seq/ge-le/Direction ผิด) |
| 577 | Scenario 6 — เลือก Path ผิดเพราะ WEIGHT ไม่สอดคล้องกัน หรือ LOCAL_PREF ถูกลืมใน Route-map |
| 578 | Scenario 7 — BGP Flapping จาก MTU Mismatch และ Maximum-Prefix Limit |
| 579 | Scenario 8 — Route ไม่ถูก Advertise ไปยัง Neighbor ที่ต้องการ (send-community/Route-map Deny) |
| 580 | Lab เต็มรูปแบบ: วินิจฉัยและแก้ปัญหา BGP 3 ชั้นซ้อนกันใน Dual-ISP Topology |

---

## Step 571 — BGP Troubleshooting Methodology: Decision Tree จาก TCP Session ถึง Advertisement

### ทำไม BGP ต้องมี Methodology ของตัวเอง (ต่างจาก OSPF ที่เพิ่งฝึกใน Part 57)

OSPF ล้มเหลวส่วนใหญ่อยู่ที่ชั้น Adjacency/LSA เพราะ Protocol ควบคุมทุกอย่างเอง (Hello, Metric,
Flooding) แต่ **BGP เป็น Protocol ที่ "ยืมมือ" ระบบอื่นตลอดทาง**: ยืม TCP ทำ Transport, ยืม IGP
พา Next-Hop ให้ถึงกัน, และยืม Policy ของ Admin (Route-map/Prefix-list/Community) ตัดสินใจว่าจะ
รับ-ส่ง Route อะไร ผลคือ **จุดที่ BGP fail ได้มีมากกว่า 1 ชั้นเสมอ** และอาการที่เห็นจาก
`show ip bgp summary` เพียงอย่างเดียวไม่พอจะบอกว่าปัญหาอยู่ชั้นไหน — ต้องไล่เป็นลำดับขั้นแบบ
Decision Tree

### Decision Tree หลัก 5 ชั้น (เรียงจากชั้นล่างสุดขึ้นไป)

```
                    ┌─────────────────────────────────────────┐
                    │  Q1: TCP Session (Port 179) ตั้งขึ้นหรือยัง? │
                    └───────────────────┬───────────────────┘
                     show ip bgp summary → State เป็นตัวเลข (PfxRcd) หรือคำ?
                                          │
                ┌─────────────────────────┴─────────────────────────┐
                │ ยัง (State = Idle/Connect/Active)                  │ แล้ว (Established)
                ▼                                                    ▼
   ┌─────────────────────────────┐                    ┌─────────────────────────────────┐
   │ Q2: ค้าง State ไหน?           │                    │ Q3: Route ถูก Received แล้วหรือยัง? │
   │  - Idle       → ยังไม่พยายาม   │                    │  show ip bgp neighbors ...        │
   │    Connect เลย (Config/       │                    │  received-routes                  │
   │    Shutdown/Admin ผิด)        │                    └───────────────┬─────────────────┘
   │  - Connect    → TCP กำลังพยายาม│                                    │
   │    (มักผ่านเร็ว ไม่ค้างนาน)     │                     ┌──────────────┴──────────────┐
   │  - Active     → TCP ต่อไม่ติด  │                     │ ไม่มา (0 Prefix)              │ มาแล้ว
   │    ซ้ำๆ (Step 572-574)         │                     ▼                              ▼
   │  - OpenSent/OpenConfirm →      │          ┌─────────────────────┐   ┌─────────────────────────┐
   │    TCP ติดแล้วแต่ OPEN ไม่ผ่าน   │          │ ผู้ส่งไม่ Advertise    │   │ Q4: ติดตั้งใน RIB หรือยัง? │
   │    (Capability/Hold-time      │          │ ออกมา (ดู Step 579   │   │  show ip route bgp        │
   │    mismatch)                  │          │ ที่ต้นทาง)             │   │  show ip bgp <prefix>     │
   └─────────────────────────────┘          └─────────────────────┘   └──────────┬──────────────┘
                                                                                    │
                                                              ┌──────────────────────┴──────────────────┐
                                                              │ ไม่ติดตั้ง (Received แต่ไม่อยู่ใน RIB)      │ ติดตั้งแล้ว
                                                              ▼                                          ▼
                                                  ┌─────────────────────────┐        ┌──────────────────────────────┐
                                                  │ สาเหตุคลาสสิก:            │        │ Q5: เป็น Best Path ที่ต้องการ  │
                                                  │ Next-Hop unreachable      │        │ หรือไม่? (Path ที่คาดไม่ตรง    │
                                                  │ (ไม่มี IGP Route ถึง        │        │ กับที่ตั้งใจออกแบบ)             │
                                                  │ Next-Hop) → Step 575       │        │ show ip bgp <prefix>          │
                                                  └─────────────────────────┘        │ เทียบ WEIGHT/LOCAL_PREF/       │
                                                                                       │ AS_PATH → Step 577            │
                                                                                       └──────────────────────────────┘
```

### ตารางสรุปคำสั่งหลักที่ใช้ในแต่ละชั้น (จะอ้างอิงซ้ำตลอด Part นี้)

| ชั้นที่ตรวจ | คำสั่งหลัก | ตอบคำถาม |
|---|---|---|
| Layer 3 พื้นฐาน (ก่อนพูดเรื่อง BGP เลย) | `ping <peer-ip>`, `show ip route <peer-ip>` | มี Route ไปถึง Peer IP หรือไม่ (Connected/IGP) |
| Layer 4 — TCP 179 | `telnet <peer-ip> 179` (หรือ `show ip bgp neighbors \| include state`) | TCP Handshake ผ่านจริงหรือไม่ แยกจากปัญหา BGP Process |
| BGP Neighbor State | `show ip bgp summary`, `show ip bgp neighbors <ip>` | State อยู่ที่ไหน, ข้อความ Error/Notification ล่าสุด |
| Route Received | `show ip bgp neighbors <ip> received-routes` (ต้องมี `soft-reconfiguration inbound`) | Peer ส่ง Prefix นี้มาจริงหรือไม่ ก่อนโดน Filter |
| Route ใน BGP Table | `show ip bgp <prefix>` | Attribute ครบไหม, มี Flag `inaccessible`/`RIB-failure` ไหม |
| Route ใน RIB | `show ip route bgp`, `show ip route <prefix>` | ถูกติดตั้งจริงหรือไม่ (แข่งกับ AD ของ Protocol อื่นด้วย) |
| Route ที่ Advertise ออก | `show ip bgp neighbors <ip> advertised-routes` | Neighbor ปลายทางควรเห็น Prefix นี้หรือไม่ |
| Filter ที่ใช้งานจริง | `show ip prefix-list detail <name>`, `show route-map <name>` | Sequence ไหน Hit, ไหนไม่ Hit (Hit Count เป็น 0 = สงสัยทันที) |

> **กฎเหล็กของ Step นี้**: อย่าข้ามชั้น — ถ้า Neighbor ยังไม่ Established (Q1-Q2) แต่ไปนั่งวิเคราะห์
> Route-map/Prefix-list (Q4-Q5) ก่อน จะเสียเวลาโดยเปล่าประโยชน์เสมอ เพราะ Filter ไม่มีความหมายอะไร
> เลยถ้า Session ยังไม่มีอยู่จริง ทุก Scenario ใน Step 572-580 จะเรียง**ตามลำดับ Decision Tree นี้**

---

## Step 572 — Scenario 1: Neighbor ค้างที่ Active/Idle เพราะ AS Number พิมพ์ผิด

### อาการที่รายงานเข้ามา

หลังจากเปลี่ยน ISR ตัวใหม่ที่ WAN-EDGE-1 ทีมงานพิมพ์ Config `neighbor` ใหม่จาก Template เดิม
แต่ Session ไปยัง ISP-RTR ไม่ยอม Established เลย

### Config ที่ผิด (WAN-EDGE-1)

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 remote-as 65500
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 description ** eBGP to ISP-RTR **
```

สังเกต `remote-as 65500` — พิมพ์ผิดจาก `65000` ที่ควรเป็น ASN จริงของ ISP-RTR (สลับเลข `0` กับ `5`
เป็นความผิดพลาดที่พบบ่อยที่สุดเวลา Copy-paste Config ข้าม Session)

### ขั้นที่ 1: `show ip bgp summary` — ดู State ก่อนเสมอตาม Decision Tree

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65500       8       6        0    0    0 00:02:14 Active
```

`State/PfxRcd = Active` (ไม่ใช่ตัวเลข) และคอลัมน์ `AS` แสดง **65500** — ค่านี้คือ ASN ที่
**Router ของเราคาดหวัง** ไม่ใช่ ASN ที่ Peer ส่งมาจริง ต้องขุดต่อว่าทำไมค้าง

### ขั้นที่ 2: `show ip bgp neighbors` — หา Notification/Error ล่าสุด

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1
BGP neighbor is 203.0.113.1,  remote AS 65500, external link
  BGP version 4, remote router ID 0.0.0.0
  BGP state = Active
  Last read never, hold time is 180, keepalive interval is 60 seconds
  Message statistics:
    ...
  Connections established 0; dropped 0
  Last reset never
```

`remote router ID 0.0.0.0` และ `Connections established 0` ยืนยันว่า **ไม่เคย Established
สักครั้งเดียว** ต่างจาก Scenario ACL (Step 573) ที่ TCP จะพยายามซ้ำแบบไม่มี Response เลย —
กรณี AS Mismatch นี้ TCP มักต่อ**ติด**ได้ (เพราะ Layer 3/4 ปกติดี) แต่ OPEN Message จะถูกปฏิเสธ
ทันที ทำให้เห็น Log ฝั่งที่ตรวจพบความผิดปกติ:

```
%BGP-3-NOTIFICATION: sent to neighbor 203.0.113.1 2/2 (peer in wrong AS) 2 bytes FDE8
%BGP-5-ADJCHANGE: neighbor 203.0.113.1 Down BGP Notification sent
```

- **Error Code 2 / Subcode 2** = OPEN Message Error → **Bad Peer AS** ตรงตัว — นี่คือ Signature
  ที่ชัดเจนที่สุดของ AS Mismatch อย่าสับสนกับ Subcode อื่น (เช่น 2/1 = Unsupported Version Number,
  2/3 = Bad BGP Identifier)
- Session จะวนเป็นวงจร **Idle → Connect → OpenSent → (Notification) → Idle** ซ้ำไปเรื่อยๆ
  ทุก ConnectRetryTimer จนกว่าจะแก้ Config

### Root Cause และการแก้ไข

**สาเหตุ**: `remote-as` ที่ Config บน WAN-EDGE-1 (`65500`) ไม่ตรงกับ ASN จริงของ ISP-RTR (`65000`)
ที่ ISP-RTR ประกาศตัวเองในทุก OPEN Message ที่ส่งออกมา

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# no neighbor 203.0.113.1 remote-as 65500
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 remote-as 65000
```

> **หมายเหตุสำคัญ**: การเปลี่ยน `remote-as` ของ Neighbor ที่มีอยู่แล้วจะ**ลบ Session เดิมทิ้งทันที**
> (Neighbor ถูกมองว่าเป็นคนละตัวเมื่อ ASN เปลี่ยน) ไม่จำเป็นต้อง `no neighbor` ก่อนก็ได้ — พิมพ์
> `neighbor 203.0.113.1 remote-as 65000` ทับได้เลย IOS จะ Reset Session ให้เองอัตโนมัติ

**ยืนยันผล:**

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000      12      11        3    0    0 00:00:45        1
```

---

## Step 573 — Scenario 2: Neighbor ค้างที่ Active เพราะ ACL บล็อก TCP 179 อย่างเงียบ

### อาการ

ทีม Security เพิ่งรัดกุม ACL ขาเข้าที่ Interface ของ WAN-EDGE-2 ตาม Audit (อ้างอิงแนวคิดจาก
[Part 45 — Advanced ACL/ZBFW](part-045-advanced-acl-zbfw.md)) หลังจากนั้น eBGP Session ไปยัง
ISP-RTR (ISP2 ที่ `203.0.113.5`) หลุดและไม่กลับมาเลย ทั้งที่ Ping ยังผ่านปกติ

### Config ที่ผิด (WAN-EDGE-2)

```
WAN-EDGE-2(config)# ip access-list extended HARDEN-WAN-IN
WAN-EDGE-2(config-ext-nacl)# permit icmp any any
WAN-EDGE-2(config-ext-nacl)# permit tcp any host 203.0.113.6 eq 22
WAN-EDGE-2(config-ext-nacl)# permit tcp any host 203.0.113.6 eq 23
WAN-EDGE-2(config-ext-nacl)# deny ip any any log
WAN-EDGE-2(config)# interface GigabitEthernet0/0/0
WAN-EDGE-2(config-if)# ip access-group HARDEN-WAN-IN in
```

ACL อนุญาต ICMP (จึง Ping ผ่าน) และ SSH/Telnet เพื่อบริหารอุปกรณ์ แต่**ไม่มี ACE ที่อนุญาต
TCP Port 179 เลย** — ทุก Packet ที่ ISP-RTR พยายามส่งมาสร้าง BGP Session จะถูก Deny โดย ACE
สุดท้าย (`deny ip any any log`) อย่างเงียบๆ

### ขั้นที่ 1: แยกปัญหา Layer 3 กับ Layer 4 ออกจากกันก่อนสงสัย BGP

```
WAN-EDGE-2# ping 203.0.113.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 203.0.113.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

Ping ผ่าน 100% — **สรุปได้ทันทีว่า Layer 3 (Routing) ไม่มีปัญหา** ต้องขยับไปตรวจ Layer 4/TCP
โดยเฉพาะ ไม่ใช่ไปแก้ที่ Routing

### ขั้นที่ 2: ทดสอบ TCP 179 แยกจาก BGP Process โดยตรงด้วย `telnet`

```
WAN-EDGE-2# telnet 203.0.113.5 179
Trying 203.0.113.5, 179 ...
% Connection timed out; remote host not responding
```

นี่คือเทคนิคสำคัญที่สุดของ Scenario นี้: ใช้ `telnet <peer-ip> 179` เป็น "TCP Client ปลอม"
เพื่อทดสอบว่า **Port 179 เปิดถึงกันจริงหรือไม่** โดยไม่ต้องเกี่ยวกับ BGP Process เลย — ถ้า Ping ผ่าน
แต่ `telnet ... 179` Time Out แบบนี้ = สัญญาณเกือบ 100% ว่ามี ACL/Firewall กรอง TCP 179 อยู่
ระหว่างทาง (ต่างจากกรณี AS Mismatch ใน Step 572 ที่ TCP จะต่อติดปกติ)

### ขั้นที่ 3: ตรวจ ACL และ Hit Counter

```
WAN-EDGE-2# show ip access-lists HARDEN-WAN-IN
Extended IP access list HARDEN-WAN-IN
    10 permit icmp any any (182 matches)
    20 permit tcp any host 203.0.113.6 eq 22
    30 permit tcp any host 203.0.113.6 eq 23
    40 deny ip any any log (57 matches)
```

`deny ip any any log` มี Hit Count เพิ่มขึ้นตลอด (57 matches และเพิ่มเรื่อยๆ) — ตรงกับจำนวนครั้ง
ที่ ISP-RTR พยายาม Retry สร้าง Session เข้ามา ยืนยัน Root Cause ชัดเจน

### Root Cause และการแก้ไข

**สาเหตุ**: ACL `HARDEN-WAN-IN` ที่ Apply แบบ `in` บน Interface ขาเข้าจาก ISP ไม่มี ACE
อนุญาต TCP Port 179 — SYN Packet จาก ISP-RTR ที่พยายามสร้าง Session ถูก Deny ทุกครั้ง

```
WAN-EDGE-2(config)# ip access-list extended HARDEN-WAN-IN
WAN-EDGE-2(config-ext-nacl)# no deny ip any any log
WAN-EDGE-2(config-ext-nacl)# permit tcp host 203.0.113.5 host 203.0.113.6 eq 179
WAN-EDGE-2(config-ext-nacl)# permit tcp host 203.0.113.5 eq 179 host 203.0.113.6
WAN-EDGE-2(config-ext-nacl)# deny ip any any log
```

> **ทำไมต้องมี 2 บรรทัด**: BGP Session **ไม่รู้ว่าใครจะเป็นฝ่าย Initiate TCP Connection ก่อน**
> (ฝั่งใดพร้อมก่อนจะส่ง SYN ก่อน) จึงต้องอนุญาตทั้ง (1) Traffic ที่ปลายทางเป็น Port 179 (กรณี
> ISP-RTR เป็นฝ่ายเริ่ม) และ (2) Traffic ที่ต้นทางเป็น Port 179 (กรณี WAN-EDGE-2 เป็นฝ่ายเริ่ม
> แล้ว Return Traffic วิ่งกลับเข้ามาโดยมี Source Port = 179) — ลืมข้อใดข้อหนึ่งจะทำให้ Session
> Established ได้เฉพาะเมื่อฝั่งใดฝั่งหนึ่งบังเอิญ Initiate ก่อนเท่านั้น ซึ่งไม่แน่นอนและมัก Flap
> ซ้ำหลัง Router Reload

**ยืนยันผล:**

```
WAN-EDGE-2# telnet 203.0.113.5 179
Trying 203.0.113.5, 179 ... Open
[Connection to 203.0.113.5 closed by foreign host]
WAN-EDGE-2# show ip bgp summary | include 203.0.113.5
203.0.113.5     4 65002      15      14        4    0    0 00:01:02        1
```

`telnet` ครั้งนี้ขึ้น `Open` (แล้วถูกปิดทันทีเพราะ ISP-RTR ไม่รู้จัก Telnet Protocol บน Port 179 —
เป็นเรื่องปกติ **สิ่งที่ต้องดูคือคำว่า `Open` ปรากฏ ไม่ใช่ Time Out**) และ BGP Established สำเร็จ

---

## Step 574 — Scenario 3: iBGP ไม่ Establish เพราะไม่มี Route ไปยัง Loopback ของ Peer

### อาการ

ทีม Network เพิ่ง "Clean up" Config OSPF บน CORE-SW2 (ลบ Network Statement ที่คิดว่าไม่ได้ใช้)
หลังจากนั้น iBGP Session จาก WAN-EDGE-1 ไปยัง CORE-SW2 (ที่ตั้งไว้ตาม
[Part 30 Step 292-293](part-030-bgp-advanced.md#step-292--lab-จริง-ibgp-full-mesh-ระหว่าง-wan-edge-1--wan-edge-2))
ก็หลุดและไม่กลับมา — Session อื่นในกลุ่ม iBGP ยังปกติดีทุกตัว

### Config ที่ผิด (CORE-SW2) — เทียบกับ CORE-SW1 ที่ปกติดี

```
! CORE-SW2 (ผิด — Loopback0 หลุดออกจาก OSPF ไปแล้วหลังการ "Clean up")
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 10.255.30.0 0.0.0.3 area 0
CORE-SW2(config-router)# network 10.255.40.0 0.0.0.3 area 0
! ขาด: network 1.1.1.2 0.0.0.0 area 0   <-- ถูกลบไปโดยไม่ตั้งใจ
```

Loopback0 (`1.1.1.2/32`) ยัง `up/up` ปกติทุกประการที่ตัว CORE-SW2 เอง แต่**ไม่เคยเข้า OSPF Area 0
เลย** จึงไม่มีทาง Router ตัวอื่นในองค์กรจะรู้จัก Prefix นี้ได้

### ขั้นที่ 1: `show ip bgp summary` — ยืนยันค้างที่ Active

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.1         4 65001     201     198       15    0    0 01:20:33        4
1.1.1.2         4 65001       0       0        0    0    0 00:05:40 Active
1.1.1.22        4 65001     180     177       15    0    0 01:15:02        2
```

Session ไปยัง `1.1.1.1` (CORE-SW1) และ `1.1.1.22` (WAN-EDGE-2) ปกติดี มีเฉพาะ `1.1.1.2`
(CORE-SW2) ที่ค้าง Active และ `MsgRcvd/MsgSent = 0` (ไม่มีการแลก Message เลยแม้แต่ครั้งเดียว
ต่างจาก Step 572 ที่ยังมี OPEN ถูกส่ง/รับก่อนถูก Reject)

### ขั้นที่ 2: ตรวจ Route ไปยัง Loopback ของ Peer — จุดที่ Decision Tree บอกให้เช็คก่อนดู BGP เลย

```
WAN-EDGE-1# ping 1.1.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)

WAN-EDGE-1# show ip route 1.1.1.2
% Network not in table
```

**ไม่มี Route ไปยัง 1.1.1.2 เลยในตาราง Routing ของ WAN-EDGE-1** — นี่คือสาเหตุตรงตามที่
[Part 30 Step 291.4](part-030-bgp-advanced.md#291-4-ทำไมต้องมี-next-hop-self) เตือนไว้ตั้งแต่แรก:
**iBGP ที่ Peer ผ่าน Loopback (`update-source Loopback0`) ต้องพึ่ง IGP พา Loopback ให้ถึงกันเสมอ**
ถ้า IGP ไม่พา TCP SYN จะไม่มีทางออกจาก Router ได้เลย (ไม่รู้จะ Route ไปทางไหน) ทำให้ Session
ค้างที่ **Active** ตลอดไปโดยไม่มี Notification ใดๆ เกิดขึ้น (ต่างจาก AS Mismatch ที่จะมี Error
ชัดเจน เพราะกรณีนี้ TCP **ไม่ได้พยายามเริ่ม** สร้าง Connection เลยด้วยซ้ำ — ไม่มีปลายทางให้ส่ง)

### ขั้นที่ 3: ยืนยัน Root Cause ที่ต้นตอ — เทียบ OSPF Config ระหว่าง CORE-SW1/CORE-SW2

```
CORE-SW1# show run | section router ospf
router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.255.0.0 0.0.0.3 area 0
 network 10.255.10.0 0.0.0.3 area 0
 network 10.255.20.0 0.0.0.3 area 0

CORE-SW2# show run | section router ospf
router ospf 1
 network 10.255.30.0 0.0.0.3 area 0
 network 10.255.40.0 0.0.0.3 area 0
```

CORE-SW1 มี `network 1.1.1.1 0.0.0.0 area 0` ครบ แต่ CORE-SW2 **ไม่มีบรรทัดเทียบเท่าสำหรับ
1.1.1.2 เลย** — ยืนยัน Root Cause ตรงจุด

### Root Cause และการแก้ไข

**สาเหตุ**: OSPF บน CORE-SW2 ไม่ได้ Advertise Loopback0 (`1.1.1.2/32`) เข้า Area 0 ทำให้ Router
อื่นในองค์กรไม่มี Route ไปถึง ปิดกั้น iBGP TCP Session ตั้งแต่ชั้น Layer 3 ก่อนจะไปถึงชั้น BGP เลย

```
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 1.1.1.2 0.0.0.0 area 0
```

**ยืนยันผล:**

```
WAN-EDGE-1# ping 1.1.1.2
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

WAN-EDGE-1# show ip bgp summary | include 1.1.1.2
1.1.1.2         4 65001       6       5        3    0    0 00:00:20        2
```

> **ข้อสังเกตสำหรับข้อสอบ**: Scenario นี้เป็นตัวอย่างชัดเจนที่สุดว่า **BGP Troubleshooting มักไม่ใช่
> ปัญหาที่ BGP เลย** — Root Cause จริงอยู่ที่ IGP (OSPF) แต่ปรากฏอาการที่ BGP (`State = Active`
> ค้าง) วิศวกรที่ไม่ไล่ Decision Tree ตาม Step 571 อาจเสียเวลาไปแก้ BGP Config ทั้งที่ไม่มีอะไรผิด
> เลยในนั้น

---

## Step 575 — Scenario 4: Route ถูก Received แต่ไม่ถูกติดตั้งในตาราง Routing

### อาการ

หลังจากแก้ Step 574 เสร็จ CORE-SW1 ยังบ่นว่าไม่เห็น Default Route จาก Internet ใน
`show ip route` เลย ทั้งที่ `show ip bgp summary` บอกว่า Session Established ปกติทุกตัว

### Config ที่ผิด (WAN-EDGE-1) — ลืม `next-hop-self` ตอน Config iBGP ใหม่

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 description ** iBGP to CORE-SW1 **
! ลืม: neighbor 1.1.1.1 next-hop-self
```

### ขั้นที่ 1: ยืนยัน Session Established จริง (Q1 ผ่าน)

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.1         4 65001      45      44       10    0    0 00:20:10        1
```

`State/PfxRcd = 1` (ตัวเลข ไม่ใช่คำ) — Session ปกติ และ WAN-EDGE-1 ได้ยืนยันว่า**ส่ง**
Prefix ไปให้ CORE-SW1 แล้ว 1 รายการ ปัญหาไม่ได้อยู่ที่ Neighbor เลย ต้องขยับไป Q3-Q4

### ขั้นที่ 2: ตรวจที่ CORE-SW1 — Route ถูก Received หรือยัง (Q3)

```
CORE-SW1# show ip bgp neighbors 1.1.1.21 received-routes
BGP table version is 10, local router ID is 1.1.1.1
   Network          Next Hop            Metric LocPrf Weight Path
 * 0.0.0.0/0        203.0.113.1              0    100      0 65000 i
```

**Received แล้ว** (มีเครื่องหมาย `*` = valid) — ผ่าน Q3 แต่ Path นี้**ไม่มีเครื่องหมาย `>`**
(best) เลย ต้องดูรายละเอียด Attribute ต่อ

### ขั้นที่ 3: `show ip bgp <prefix>` — หา Flag `inaccessible`

```
CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 10
Paths: (1 available, no best path)
  Not advertised to any peer
  65000
    203.0.113.1 (inaccessible) from 1.1.1.21 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, valid, internal
```

พบคำตอบชัดเจน: `(inaccessible)` ต่อท้าย Next-Hop `203.0.113.1` และหัวตารางบอกตรงๆว่า
**`no best path`** — Path นี้ **Valid แต่ไม่ผ่านการตัดสินเป็น Best Path เพราะ Next-Hop
ไปไม่ถึง** ผลคือไม่ถูกติดตั้งใน RIB เลย (`show ip route bgp` จะไม่มีอะไรแสดง)

### ขั้นที่ 4: ยืนยัน Next-Hop ไปไม่ถึงจริง

```
CORE-SW1# show ip route 203.0.113.1
% Network not in table
```

`203.0.113.1` คือ IP ของ ISP-RTR บน Transit Link `203.0.113.0/30` — Subnet นี้**ไม่เคยถูก
Advertise เข้า OSPF ภายในองค์กรเลยตามการออกแบบ** (เป็น Transit Link ระหว่าง WAN-EDGE-1 กับ ISP
เท่านั้น) เพราะ eBGP เปลี่ยน Next-Hop เป็น IP ของ ISP-RTR เอง (ตามพฤติกรรม Default ของ eBGP)
แล้ว WAN-EDGE-1 ก็ส่งต่อ Next-Hop เดิมนี้เข้า iBGP โดยไม่แก้ไข (เพราะ iBGP ไม่เปลี่ยน Next-Hop
โดย Default) — CORE-SW1 จึงได้ Next-Hop ที่ไม่มีวันไปถึงได้

### Root Cause และการแก้ไข

**สาเหตุ**: ไม่ได้ตั้ง `next-hop-self` บน WAN-EDGE-1 (จุดเชื่อมระหว่าง eBGP กับ iBGP) — ตรงตาม
"กฎจำง่าย" ที่เตือนไว้ใน [Part 30 Step 291.4](part-030-bgp-advanced.md#291-4-ทำไมต้องมี-next-hop-self)

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 next-hop-self
```

**ยืนยันผล:**

```
CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 12
Paths: (1 available, best #1, table default)
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, valid, internal, best

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.21, from 1.1.1.21, 00:00:05 ago
```

Next-Hop เปลี่ยนเป็น `1.1.1.21` (Loopback ของ WAN-EDGE-1 เอง — ไปถึงได้แน่นอนผ่าน OSPF) Path
กลายเป็น `best` และถูกติดตั้งใน RIB ทันที

> **จุดสังเกตสำหรับข้อสอบ**: คำว่า **`inaccessible`** ใน `show ip bgp <prefix>` คือ Signature
> ที่ชัดที่สุดของปัญหานี้ — เมื่อเห็นคำนี้ ให้สงสัย `next-hop-self` ที่ ASBR/Border Router ก่อน
> เสมอ อย่าเพิ่งไปแก้ Prefix-list หรือ Route-map เพราะ Attribute อื่นๆ (`localpref`, `valid`)
> ในตัวอย่างนี้ถูกต้องสมบูรณ์ทุกตัว มีแค่ Next-Hop เท่านั้นที่ผิด

---

## Step 576 — Scenario 5: Prefix-list บล็อก Route โดยไม่ตั้งใจ (Seq/ge-le/Direction ผิด)

### อาการ

ทีม Network ต้องการเผื่อไว้สำหรับอนาคตที่จะแบ่ง Public Block `203.0.113.0/24` เป็น Subnet ย่อย
จึงแก้ไข Prefix-list `PL-TO-ISP` (ตัวเดียวกับที่ตั้งใน
[Part 29 Step 286](part-029-bgp-fundamentals.md#step-286--bgp-route-filtering-พื้นฐาน))
ให้รองรับ Prefix ที่เฉพาะเจาะจงกว่าด้วย — หลังแก้เสร็จ ISP-RTR **หลุด Route `203.0.113.0/24`
ไปเลยทั้งก้อน** องค์กรมองไม่เห็นจาก Internet ทันที

### Config ที่ผิด (WAN-EDGE-1)

```
WAN-EDGE-1(config)# ip prefix-list PL-TO-ISP seq 10 permit 203.0.113.0/24 ge 25 le 32
```

โดยที่ ACE เดิม `ip prefix-list PL-TO-ISP seq 5 permit 203.0.113.0/24` (Match แบบเป๊ะ `/24`
เท่านั้น) **ถูกลบไปก่อนหน้านี้ระหว่างการแก้ไข** (คิดว่า Seq 10 ตัวใหม่ "ครอบคลุม" อยู่แล้ว) ทำให้
Prefix-list เหลือ ACE เดียวที่กำหนด `ge 25` — ซึ่งหมายถึง **ต้อง Match Prefix Length ตั้งแต่ /25
ขึ้นไปเท่านั้น** Prefix `/24` เป๊ะๆ (ที่ไม่มี ge/le) จะ**ไม่ Match ACE นี้เลย** และตกไปโดน
Implicit Deny ที่ปิดท้าย Prefix-list ทุกตัวเสมอ

### ขั้นที่ 1: ยืนยันว่า Session ปกติ แต่ Route หายไปตอน Advertise (Q1-Q3 ผ่าน, ตกที่ Q3 ฝั่ง Advertise)

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 advertised-routes
Total number of prefixes advertised: 0
```

Session Established (ไม่งั้นคำสั่งนี้จะไม่ทำงานเลย) แต่ **จำนวน Prefix ที่ Advertise = 0** ทั้งที่
`network 203.0.113.0 mask 255.255.255.0` ยังอยู่ใน Config ครบ

### ขั้นที่ 2: ตรวจว่า Prefix อยู่ใน BGP Table ของ Router เองหรือไม่ (แยกจากปัญหา Filter)

```
WAN-EDGE-1# show ip bgp 203.0.113.0/24
BGP routing table entry for 203.0.113.0/24, version 3
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     (ไม่มี — ไม่ถูกส่งให้ใครเลย)
  Local
    0.0.0.0 from 0.0.0.0 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, weight 32768, valid, sourced, local, best
```

Prefix `203.0.113.0/24` **มีอยู่ในตาราง BGP ของ WAN-EDGE-1 เองแน่นอน** (มาจาก `network`
statement, `sourced, local, best`) แต่ **`Advertised to update-groups` เป็นค่าง่าง** — สรุปได้
ว่าปัญหาอยู่ที่ **Outbound Filter** เท่านั้น ไม่ใช่ Route หายจาก RIB เหมือน Step 575

### ขั้นที่ 3: ตรวจ Hit Count ของ Prefix-list — จุดวินิจฉัยที่ตรงเป้าที่สุด

```
WAN-EDGE-1# show ip prefix-list detail PL-TO-ISP
ip prefix-list PL-TO-ISP:
   count: 1, range entries: 1, sequences: 10 - 10, refcount: 3
   seq 10 permit 203.0.113.0/24 ge 25 le 32 (hit count: 0, refcount: 3)
```

`hit count: 0` บน ACE เดียวที่มีอยู่ — ยืนยันว่าไม่มี Prefix ใดผ่าน ACE นี้เลย (เพราะ Prefix
`/24` เป๊ะไม่ Match เงื่อนไข `ge 25`) และเพราะไม่มี ACE อื่นเหลืออยู่ Prefix ทั้งหมดจึงตกไปโดน
Implicit Deny ที่ปิดท้ายเสมอ

### Root Cause และการแก้ไข

**สาเหตุ**: ACE ที่ตั้งใจให้ Match เฉพาะ Subnet ย่อยในอนาคต (`ge 25 le 32`) **ไม่ครอบคลุม
Prefix เดิม `/24`** และ ACE เดิมที่ Match แบบเป๊ะถูกลบไปโดยไม่ได้เพิ่มกลับมา

```
WAN-EDGE-1(config)# ip prefix-list PL-TO-ISP seq 5 permit 203.0.113.0/24
```

เพิ่ม ACE Seq 5 กลับมาให้ Match `/24` แบบเป๊ะ (ไม่มี `ge`/`le`) ไว้เป็นลำดับ**ก่อน** Seq 10
ตาม Rule ของ Prefix-list ที่ตรวจตามลำดับ Sequence จากน้อยไปมากเสมอ

**ยืนยันผล:**

```
WAN-EDGE-1# show ip prefix-list detail PL-TO-ISP
ip prefix-list PL-TO-ISP:
   count: 2, range entries: 1, sequences: 5 - 10, refcount: 3
   seq 5 permit 203.0.113.0/24 (hit count: 1, refcount: 3)
   seq 10 permit 203.0.113.0/24 ge 25 le 32 (hit count: 0, refcount: 3)

WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 advertised-routes
   Network          Next Hop            Metric LocPrf Weight Path
*> 203.0.113.0/24   0.0.0.0                  0         32768 i
```

> **กับดักที่สองที่ต้องระวังคู่กัน — สลับ `in`/`out`**: ถ้า Config ผิดแบบ
> `neighbor 203.0.113.1 prefix-list PL-TO-ISP in` (ทั้งที่ตั้งใจจะกรอง Outbound) Prefix-list
> จะไปกรอง Route ที่**รับเข้า**จาก ISP-RTR แทน — อาการที่เห็นจะกลายเป็น "รับ Default Route ไม่ได้"
> ไม่ใช่ "ส่ง Prefix ออกไม่ได้" แม้ Prefix-list เขียนถูกทุกตัวอักษร วิธีตรวจคือดูที่
> `show run | section router bgp` เทียบกับที่ตั้งใจเสมอ — Direction ผิดพบบ่อยพอๆกับ Seq/ge-le ผิด

---

## Step 577 — Scenario 6: เลือก Path ผิดเพราะ WEIGHT ไม่สอดคล้องกัน / LOCAL_PREF ถูกลืม

### 577.1 Sub-scenario A: WEIGHT ตั้งไม่สอดคล้องกันข้าม Router ทำให้เกิด Asymmetric Routing

ตาม Design ของ [Part 30 Step 296](part-030-bgp-advanced.md#step-296--กำหนดเส้นทางขาออก-outbound-ด้วย-weight-และ-local_pref)
องค์กรตั้งใจให้**ทั้ง AS** ออก Internet ผ่าน ISP1 เป็นหลักด้วย LOCAL_PREF=200 — แต่วิศวกรคนหนึ่ง
เคย Config `weight 500` ไว้บน CORE-SW1 (ตามตัวอย่าง Override เฉพาะจุดใน Step 296.2) สำหรับ
Prefix พิเศษ `198.51.100.0/24` เพื่อให้ออกทาง ISP2 แทน แล้ว**ลืมว่าเคยตั้งไว้** ต่อมามีคนแก้ Policy
LOCAL_PREF ใหม่บนอีก Router โดยไม่รู้ว่ามี WEIGHT ซ้อนอยู่ ผลคือ Traffic ของ Prefix นี้จาก
CORE-SW1 กับ CORE-SW2 **ออกคนละ ISP กัน** (Asymmetric Routing) ทำให้ Stateful Firewall บาง
Session ตัดการเชื่อมต่อ

**วินิจฉัยด้วยการเทียบ Attribute ระหว่าง Router ที่ให้ผลต่างกัน:**

```
CORE-SW1# show ip bgp 198.51.100.0
BGP routing table entry for 198.51.100.0/24, version 20
Paths: (2 available, best #2, table default)
  65002
    1.1.1.22 from 1.1.1.22 (1.1.1.22)
      Origin IGP, localpref 200, weight 500, valid, internal, best   <- WEIGHT ชนะตั้งแต่ขั้นที่ 1
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 200, weight 0, valid, internal

CORE-SW2# show ip bgp 198.51.100.0
BGP routing table entry for 198.51.100.0/24, version 18
Paths: (2 available, best #1, table default)
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 200, weight 0, valid, internal, best     <- ไม่มี WEIGHT ให้แข่ง จึงใช้ LOCAL_PREF ตามปกติ
  65002
    1.1.1.22 from 1.1.1.22 (1.1.1.22)
      Origin IGP, localpref 200, weight 0, valid, internal
```

ทั้งสอง Path มี `localpref 200` เท่ากัน (เพราะ LOCAL_PREF ถูกออกแบบให้ทั้ง AS เห็นเหมือนกัน
ตาม Step 296.1) แต่ **CORE-SW1 มี `weight 500`** ค้างอยู่เฉพาะ Local ตัวเดียว (WEIGHT ไม่ถูก
ส่งต่อผ่าน iBGP ให้ CORE-SW2 เห็นเลย) — เพราะ WEIGHT (ขั้นที่ 1 ของ Best Path Algorithm) ถูกเทียบ
**ก่อน** LOCAL_PREF (ขั้นที่ 2) เสมอ CORE-SW1 จึงเลือกออกทาง ISP2 ในขณะที่ CORE-SW2 เลือกออกทาง
ISP1 ตามปกติ

**Root Cause**: ใช้ WEIGHT (Local-only) แก้ปัญหาที่ควรใช้ LOCAL_PREF (Org-wide) หรือควบคุมผ่าน
Community/Route-map แบบรวมศูนย์ ทำให้ Router สองตัวมีมุมมองต่างกันโดยไม่มีใครรู้ตัว

**การแก้ไข**: ลบ WEIGHT Override ที่ CORE-SW1 ออก แล้วถ้าต้องการให้ Prefix นี้ออกทาง ISP2 จริง
ให้ทำผ่าน LOCAL_PREF ที่จุดเดียว (บน WAN-EDGE ที่รับ Route เข้ามา) เพื่อให้ทั้ง AS เห็นตรงกัน:

```
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# no neighbor 1.1.1.22 route-map OVERRIDE-WEIGHT-ISP2 in
```

### 577.2 Sub-scenario B: Route-map ลืม `set local-preference` หลังแก้ไข Sequence ใหม่

ทีมงานเพิ่ม Sequence ใหม่ให้ Route-map `SET-LOCALPREF-ISP1` (จาก
[Part 30 Step 296.1](part-030-bgp-advanced.md#296-1-local_pref--วิธีที่ถูกต้องสำหรับทั้ง-as))
เพื่อ Set Community เพิ่มไปด้วย แต่พิมพ์ตกหล่นบรรทัด `set local-preference`:

```
WAN-EDGE-1(config)# route-map SET-LOCALPREF-ISP1 permit 10
WAN-EDGE-1(config-route-map)# set community 65001:100 additive
WAN-EDGE-1(config-route-map)# exit
! ผลคือ Sequence 10 นี้ตั้งค่า Community แต่ไม่ตั้ง Local-Preference อีกต่อไป
```

(Sequence 10 เดิมมี `set local-preference 200` อยู่แล้ว การพิมพ์คำสั่งใหม่ในโหมด Route-map
Sequence เดียวกันจะ**เพิ่ม**ค่าใหม่ แต่ถ้าวิศวกรลบ Sequence เดิมด้วย `no route-map ... 10` ก่อน
แล้วพิมพ์ใหม่ทั้งหมดโดยลืมบรรทัด Local-Preference — Route ทุก Prefix ที่ Match Sequence นี้จะ
กลับไปใช้ค่า Default **100** ทันที)

**วินิจฉัย:**

```
CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 100, valid, internal      <- ควรเป็น 200 ตามที่ Design ไว้!
      Community: 65001:100
```

`localpref 100` (ค่า Default) ทั้งที่ Route นี้ผ่าน Route-map `SET-LOCALPREF-ISP1` มาแล้ว —
ตรวจ Route-map ตรงๆ:

```
WAN-EDGE-1# show route-map SET-LOCALPREF-ISP1
route-map SET-LOCALPREF-ISP1, permit, sequence 10
  Match clauses:
  Set clauses:
    community 65001:100 additive
  Policy routing matches: 0 packets, 0 bytes
```

ใน `Set clauses` **ไม่มี `local-preference` เหลืออยู่เลย** — ยืนยัน Root Cause ตรงจุด

**การแก้ไข**: เติมบรรทัดที่ขาดกลับเข้าไปใน Sequence เดิม

```
WAN-EDGE-1(config)# route-map SET-LOCALPREF-ISP1 permit 10
WAN-EDGE-1(config-route-map)# set local-preference 200
WAN-EDGE-1(config-router)# end
WAN-EDGE-1# clear ip bgp 203.0.113.1 soft in
```

> **บทเรียนของ Sub-scenario ทั้งสอง**: WEIGHT และ LOCAL_PREF เป็นเครื่องมือคนละแบบที่ตอบโจทย์
> คนละขนาด — **ถ้าผลลัพธ์ที่เห็นคือ "Router บางตัวเลือกต่างจาก Router อื่นในองค์กรเดียวกัน"
> ให้สงสัย WEIGHT ก่อนเสมอ (มันไม่แชร์กับใคร)** ส่วน **ถ้าทั้ง AS เห็นค่าไม่ตรงกับ Policy ที่ตั้งใจ
> ให้สงสัยว่า Route-map ที่ตั้ง LOCAL_PREF มีบรรทัด `set` หายไปหรือไม่** — ทั้งสองกรณีเช็คด้วย
> `show ip bgp <prefix>` และ `show route-map` เป็นหลักเหมือนกัน

---

## Step 578 — Scenario 7: BGP Flapping จาก Maximum-Prefix Limit และ MTU Mismatch

### 578.1 Sub-scenario A: Maximum-Prefix Limit ถูกชนหลัง ISP เปลี่ยนนโยบาย Summarization

### อาการ

Session ไปยัง ISP-RTR (ISP1) เคยเสถียรมาหลายเดือน จนกระทั่งวันหนึ่ง Flap (Established → Idle
→ Established) ซ้ำหลายรอบโดยไม่มีใครแก้ Config ฝั่งองค์กรเลย

### Config ที่มีอยู่ (WAN-EDGE-1) — ตั้งไว้ตั้งแต่แรกเป็น Safety Net

```
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 maximum-prefix 100
```

(ไม่ได้ใส่ `warning-only` หรือ Threshold Percentage — เป็น Config แบบ Default ที่ **Enforce
เต็มรูปแบบ** ทันทีที่เกินขีดจำกัด)

### ขั้นที่ 1: ตรวจ Log ก่อนเสมอเมื่อเจอ Flapping (Flapping มักทิ้ง Log ไว้ชัดกว่า State เฉยๆ)

```
WAN-EDGE-1# show logging | include BGP
%BGP-3-MAXPFXEXCEED: No. of prefix received from 203.0.113.1 (afi 0) : 105 exceeds limit 100
%BGP-5-ADJCHANGE: neighbor 203.0.113.1 Down BGP Notification sent
%BGPMIB-5-MAXPFX_START: ...
```

`MAXPFXEXCEED` คือ Signature ตรงตัวที่สุดของ Scenario นี้ — ISP-RTR ส่ง Prefix มา **105 รายการ**
ทะลุขีดจำกัด **100** ที่ตั้งไว้

### ขั้นที่ 2: ยืนยัน State ปัจจุบันหลังชน Limit

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000     512     498       30    0    0 00:00:12 Idle (PfxCt)
```

`Idle (PfxCt)` คือ State พิเศษที่บอกตรงๆว่า **Session ถูกปิดเพราะ Prefix Count เกิน** และที่
สำคัญ **BGP จะไม่พยายาม Reconnect เองอัตโนมัติ** จนกว่าจะมีคนสั่ง `clear ip bgp` ด้วยมือ
(ต่างจากปัญหาอื่นๆก่อนหน้าที่ยัง Retry เองไปเรื่อยๆ)

### ขั้นที่ 3: หาสาเหตุที่แท้จริงว่าทำไม Prefix เพิ่มขึ้นกะทันหัน

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 received-routes | begin Network
   Network          Next Hop            Metric LocPrf Weight Path
*> 8.8.8.0/24       203.0.113.1              0             0 65000 i
*> 8.8.9.0/24       203.0.113.1              0             0 65000 i
*> 8.8.10.0/24      203.0.113.1              0             0 65000 i
... (ต่อไปอีกกว่า 100 บรรทัด แบ่งเป็น Subnet ย่อยที่ก่อนหน้านี้เคยมาเป็น 8.8.8.0/22 ก้อนเดียว)
```

**ISP หยุดทำ Summarization** ฝั่งของตัวเอง (อาจเพราะ Traffic Engineering หรือ Config เปลี่ยนโดย
ISP โดยไม่แจ้งลูกค้า) ส่ง Prefix ย่อยแยกออกมาแทน Aggregate เดิม — จำนวน Prefix ที่แต่ก่อนอยู่ที่
~50 กระโดดเป็น 105 ทันที ทะลุ Limit ที่ตั้งไว้แบบไม่มีช่องว่างสำรอง (Headroom)

### Root Cause และการแก้ไข

**สาเหตุ**: `maximum-prefix 100` ตั้งไว้พอดีเกินไปไม่มี Headroom และไม่มี Threshold เตือนล่วงหน้า
เมื่อ ISP เปลี่ยนพฤติกรรมการ Summarize Prefix ที่แต่ก่อนน้อยกว่า Limit ก็ทะลุขึ้นมาในพริบตา

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# no neighbor 203.0.113.1 maximum-prefix 100
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 maximum-prefix 200 80 restart 5
WAN-EDGE-1(config-router)# end
WAN-EDGE-1# clear ip bgp 203.0.113.1
```

- `200` = Limit ใหม่ (เผื่อ Headroom จากที่ ISP อาจ De-aggregate เพิ่มอีกในอนาคต)
- `80` = Threshold เตือนที่ 80% (160 Prefix) ด้วย Syslog **ก่อน**ถึง Limit จริง ให้มีเวลาตรวจสอบ
- `restart 5` = ถ้าเกิน Limit จริง ให้ Router **พยายาม Reconnect เองอัตโนมัติทุก 5 นาที** แทนการ
  ค้างที่ `Idle (PfxCt)` รอคนสั่ง Clear ตลอดไป (ทางเลือกที่ปลอดภัยกว่าการปิดสมบูรณ์ในบาง Design)
- `clear ip bgp 203.0.113.1` (Hard Reset) จำเป็นเพราะ Session อยู่ใน `Idle (PfxCt)` ซึ่งจะไม่ฟื้น
  เองจนกว่าจะ Clear หรือมี `restart` Option — ควรพิจารณาเพิ่ม `ip prefix-list` กรอง Subnet ย่อย
  ที่ไม่จำเป็น (ตาม Step 576) ควบคู่กันด้วยถ้าต้องการยึด Full Table เฉพาะที่จำเป็นจริง

### 578.2 Sub-scenario B: MTU Mismatch ทำให้ Session Flap เฉพาะตอนส่ง Update ก้อนใหญ่

**อาการ**: Session Established ได้ปกติทุกครั้ง (OPEN/KEEPALIVE ผ่านหมด เพราะ Message เล็ก) แต่
จะ Flap ทุกครั้งที่มีการ Re-advertise Full Table ก้อนใหญ่ (เช่นหลัง Clear Session หรือ ISP
Convergence) — ตรวจพบว่า Interface ฝั่ง WAN-EDGE ตั้ง MTU ไว้ 1500 ปกติ แต่ฝั่ง ISP-RTR (ตั้งใจ
รองรับ Jumbo Frame สำหรับ Backbone อื่น) ตั้งไว้ 9216 บน Interface เดียวกัน โดยไม่ได้ตกลงกันไว้

**วินิจฉัย:**

```
WAN-EDGE-1# show interfaces GigabitEthernet0/0/0 | include MTU
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,

ISP-RTR# show interfaces GigabitEthernet0/0/0 | include MTU
  MTU 9216 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
```

TCP Session ต่อรอง MSS ตาม MTU ของแต่ละฝั่งตอน 3-Way Handshake — เมื่อ UPDATE Message ขนาดใหญ่
(Full Table) ถูกส่งจากฝั่งที่ MTU สูงกว่าและ Path MTU Discovery ใช้งานไม่ได้ (เช่น ICMP
"Fragmentation Needed" ถูกกรองไปตามทาง) Segment จะถูก Drop เงียบๆ กลายเป็น TCP Retransmit ซ้ำ
จนเกิน Hold Timer แล้ว Session ล่ม — Pattern ที่สังเกตได้คือ **Flap สัมพันธ์กับปริมาณ Route
ที่ต้องส่ง ไม่ใช่สัมพันธ์กับเวลาแบบสุ่ม**

**การแก้ไข**: ปรับ MTU ให้ตรงกันทั้งสองฝั่งของ Transit Link (ไม่ใช่ปรับทั้งวง เพราะ ISP-RTR
อาจต้องคง Jumbo Frame ไว้สำหรับ Backbone ฝั่งอื่น):

```
WAN-EDGE-1(config-if)# mtu 1500
ISP-RTR(config-if)# mtu 1500
```

หรือถ้าจำเป็นต้องคง MTU ต่างกันจริง ให้เปิด Path MTU Discovery ของ TCP และตรวจสอบว่า ICMP
Type 3 Code 4 (Fragmentation Needed) ไม่ถูกบล็อกระหว่างทาง — แต่ **วิธีที่มั่นคงที่สุดสำหรับ
Transit Link แบบ Point-to-Point คือบังคับ MTU ให้เท่ากันตรงๆ** ไม่ต้องพึ่งกลไก Discovery ที่
เปราะบางกว่า

---

## Step 579 — Scenario 8: Route ไม่ถูก Advertise ไปยัง Neighbor ที่ต้องการ

### 579.1 Sub-scenario A: ลืม `send-community` ทำให้ Policy ปลายทางที่ Match Community ล้มเหลว

ตาม [Part 30 Step 298](part-030-bgp-advanced.md#step-298--bgp-communities) องค์กร Tag Route
จาก ISP1 ด้วย Community `65001:100` แล้วใช้ Community นี้เป็นเงื่อนไขกรองต่อบน CORE-SW2 ก่อน
Advertise ต่อไปยัง Router อื่น (สมมติสถานการณ์ขยายสำหรับ Redistribution ใน
[Part 59](part-059-redistribution-troubleshooting.md) ที่กำลังจะเรียน) แต่วิศวกรที่เพิ่ม iBGP
Neighbor ใหม่ (CORE-SW2) ลืมเปิด `send-community` ไว้

**Config ที่ผิด (WAN-EDGE-1):**

```
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 remote-as 65001
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 update-source Loopback0
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 next-hop-self
! ลืม: neighbor 1.1.1.2 send-community both
```

**Route-map บน CORE-SW2 ที่ Match Community เพื่อกรอง Outbound:**

```
CORE-SW2(config)# route-map PERMIT-ISP1-ONLY permit 10
CORE-SW2(config-route-map)# match community CL-ISP1-TAG
CORE-SW2(config)# ip community-list standard CL-ISP1-TAG permit 65001:100
```

**วินิจฉัย:**

```
CORE-SW2# show ip bgp 0.0.0.0 community
BGP routing table entry for 0.0.0.0/0
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 200, valid, internal, best
      (ไม่มีบรรทัด Community เลย — ควรมี "Community: 65001:100")
```

Route มาถึง CORE-SW2 และเป็น `best` แต่**ไม่มี Community ติดมาด้วยเลย** — เพราะ WAN-EDGE-1
ไม่ได้เปิด `send-community` บน Neighbor Statement ฝั่งที่ส่งไปยัง CORE-SW2 (Default ของ BGP
คือไม่ส่ง Community ให้ Neighbor ไหนเลยจนกว่าจะสั่งเปิด) ผลคือ Route-map `PERMIT-ISP1-ONLY`
บน CORE-SW2 หา Community ไม่พบ เข้า Implicit Deny และ**ไม่ถูก Advertise ต่อไปยัง Neighbor
ปลายทางเลย** ทั้งที่ตัว Route เองอยู่ใน BGP Table และเป็น Best Path ปกติทุกประการ

**การแก้ไข:**

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 send-community both
```

### 579.2 Sub-scenario B: Route-map Outbound มี Sequence `deny` ที่ครอบ Prefix เกินความตั้งใจ

ทีมงานเพิ่ม Sequence ใหม่ให้กับ Route-map ที่ใช้กรอง Outbound ไปยัง WAN-EDGE-2 เพื่อกันไม่ให้
Prefix ทดสอบภายในหลุดออกไป แต่ Prefix-list ที่อ้างอิงเขียน Boundary กว้างเกินไปจนกิน Prefix
Production ที่ต้องการ Advertise จริงด้วย:

```
WAN-EDGE-1(config)# ip prefix-list PL-BLOCK-TEST seq 5 permit 192.0.2.0/24 ge 24 le 32
WAN-EDGE-1(config)# route-map OUT-TO-WANEDGE2 deny 5
WAN-EDGE-1(config-route-map)# match ip address prefix-list PL-BLOCK-TEST
WAN-EDGE-1(config)# route-map OUT-TO-WANEDGE2 permit 10
```

(ภายหลังพบว่า Public Block จริงขององค์กร `203.0.113.0/24` บังเอิญเคย Summarize ไปเป็น
`192.0.2.0/23` ชั่วคราวระหว่าง Migration ทำให้ Prefix-list `ge 24` ที่ตั้งใจกันแค่ Prefix ทดสอบ
ไปครอบ Production Prefix เข้าด้วยโดยไม่ตั้งใจ)

**วินิจฉัย — จุดสำคัญคือต้องแยกให้ออกว่า Route มี Best Path ใน BGP Table แล้ว แต่ตกที่ขั้น
Advertise เท่านั้น:**

```
WAN-EDGE-1# show ip bgp 192.0.2.0/23
BGP routing table entry for 192.0.2.0/23, version 40
Paths: (1 available, best #1, table default)
  Not advertised to any peer          <- สัญญาณเตือนสำคัญที่สุด
  Local
    0.0.0.0 from 0.0.0.0 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, weight 32768, valid, sourced, local, best
```

`Not advertised to any peer` แม้ Route เป็น `best` และ `valid` ครบ — ต้องดู Route-map ที่ผูกกับ
Neighbor ทุกตัว:

```
WAN-EDGE-1# show route-map OUT-TO-WANEDGE2
route-map OUT-TO-WANEDGE2, deny, sequence 5
  Match clauses:
    ip address prefix-list PL-BLOCK-TEST
  Matches: 1
route-map OUT-TO-WANEDGE2, permit, sequence 10
  Match clauses:
  Matches: 0
```

Sequence 5 (deny) มี `Matches: 1` — ยืนยันว่า Prefix `192.0.2.0/23` ถูก Match เข้ากับเงื่อนไข
`deny` โดยไม่ตั้งใจ ก่อนจะถึง Sequence 10 (permit ทุกอย่างที่เหลือ) เลย

**การแก้ไข**: จำกัด Prefix-list ให้ตรงเฉพาะ Prefix ทดสอบจริงเท่านั้น (ระบุ Prefix เป๊ะ ไม่ใช้
`ge`/`le` แบบกว้าง) หรือย้าย Sequence Production ให้ Permit ก่อน Deny เสมอ:

```
WAN-EDGE-1(config)# no ip prefix-list PL-BLOCK-TEST seq 5
WAN-EDGE-1(config)# ip prefix-list PL-BLOCK-TEST seq 5 permit 192.0.2.100/32
WAN-EDGE-1(config)# ip prefix-list PL-BLOCK-TEST seq 6 permit 192.0.2.101/32
```

> **บทเรียนคู่กันของ Step นี้**: "Route มี Best Path ที่ต้องการแล้ว แต่ไม่ไปถึง Neighbor" มีแค่
> 2 จุดหลักให้ตรวจเสมอ — (1) **`send-community`** ถ้า Policy ปลายทาง Match บน Community
> (2) **Route-map/Prefix-list ทิศทาง `out`** ที่ผูกกับ Neighbor นั้นโดยเฉพาะ ใช้
> `show ip bgp <prefix>` หาคำว่า `Not advertised to any peer` เป็นจุดเริ่มต้นเสมอ แล้วไล่ตาม
> Route-map ที่ผูกกับ Neighbor ปลายทางทีละตัว

---

## Step 580 — Lab เต็มรูปแบบ: วินิจฉัยและแก้ปัญหา BGP 3 ชั้นซ้อนกันใน Dual-ISP Topology

### สถานการณ์: หลัง Maintenance Window คืนวันเสาร์ Internet ขององค์กรมีปัญหาเป็นระยะ

ทีม NOC รายงานเข้ามาเป็นชุดอาการปนกัน: "Internet ช้าเป็นพักๆ", "บาง Site เข้า Internet ไม่ได้เลย
ผ่าน ISP2", และ "แม้จะยัง Ping Internet ได้จาก WAN-EDGE-1 แต่ CORE-SW1/CORE-SW2 บางตัวก็ยัง
เข้า Internet ไม่ได้บางช่วง" — จากการตรวจสอบ Change Log พบว่าคืนนั้นมีการเปลี่ยนแปลง 3 อย่าง
พร้อมกันโดยทีมต่างกัน (Security, Network, Compliance) โดยไม่มีใครรู้ว่าอีกทีมทำอะไรบ้าง:

```
Topology ที่ใช้ใน Lab นี้ (ตาม Part 29-30):

  ISP-RTR (AS 65000/65002)
    │ 203.0.113.0/30              │ 203.0.113.4/30
    ▼                              ▼
  WAN-EDGE-1 (AS 65001) ◄──iBGP──► WAN-EDGE-2 (AS 65001)
    │                              │
    │         iBGP Full-Mesh       │
    ▼                              ▼
  CORE-SW1 (AS 65001) ◄──iBGP──► CORE-SW2 (AS 65001)
```

### ปัญหาที่ซ่อนอยู่ 3 ชั้น (ไม่มีใครรู้ทั้งหมดตอนเริ่มตรวจ)

| # | ทีมที่แก้ | Change ที่ทำ | ผลกระทบที่ซ่อนอยู่ |
|---|---|---|---|
| 1 | Security | เพิ่ม ACL รัดกุมบน WAN-EDGE-2 → ISP-RTR (ISP2) | ปิด TCP 179 อย่างไม่ตั้งใจ (เหมือน Step 573) |
| 2 | Network | Config iBGP Neighbor ใหม่ CORE-SW1 ↔ WAN-EDGE-1 ซ้ำ (คิดว่าตัวเดิมหาย) | Neighbor Statement ใหม่ไม่มี `next-hop-self` (เหมือน Step 575) |
| 3 | Compliance | เข้มงวด Prefix-list ขาเข้าจาก ISP1 ให้รับแค่ Prefix ที่ "อนุมัติแล้ว" | Boundary ผิดจนกรอง Default Route ตัวเองออกไปด้วย (เหมือน Step 576) |

### ขั้นที่ 1: เริ่มจาก Q1 ของ Decision Tree เสมอ — ภาพรวมทุก Session พร้อมกัน

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000     820     815       25    0    0 3d02h          1
1.1.1.1         4 65001       5       3        0    0    0 00:01:40        0
1.1.1.2         4 65001     150     148       25    0    0 02:10:00        4
1.1.1.22        4 65001     900     880       25    0    0 3d02h           2

WAN-EDGE-2# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.5     4 65002       0       0        0    0    0 00:15:22 Active
1.1.1.1         4 65001     140     138       25    0    0 02:09:50        1
1.1.1.2         4 65001     140     139       25    0    0 02:09:50        1
1.1.1.21        4 65001     900     880       25    0    0 3d02h           2
```

พบ 2 จุดผิดปกติทันที: (A) `203.0.113.5` บน WAN-EDGE-2 ค้าง **Active** (แบบ Step 573/574) และ
(B) `1.1.1.1` บน WAN-EDGE-1 Established แล้ว (`00:01:40` — Up ไม่นาน หมายความว่าพึ่งเกิดจาก
Config ใหม่ตามที่ Network Team ทำ) แต่ **`State/PfxRcd = 0`** ทั้งที่ WAN-EDGE-1 ควรมี Default
Route ให้ Advertise เข้า CORE-SW1 อยู่แล้ว

### ขั้นที่ 2: แก้ปัญหาที่ 1 — ACL บล็อก TCP 179 บน WAN-EDGE-2 (ตาม Step 573)

```
WAN-EDGE-2# ping 203.0.113.5
Success rate is 100 percent (5/5)     <- Layer 3 ปกติ

WAN-EDGE-2# telnet 203.0.113.5 179
% Connection timed out; remote host not responding    <- ยืนยัน TCP 179 ถูกบล็อก

WAN-EDGE-2# show ip access-lists HARDEN-WAN-IN
Extended IP access list HARDEN-WAN-IN
    10 permit icmp any any (940 matches)
    20 permit tcp any host 203.0.113.6 eq 22
    30 deny ip any any log (312 matches)
```

**แก้ไข** (เหมือน Step 573):

```
WAN-EDGE-2(config)# ip access-list extended HARDEN-WAN-IN
WAN-EDGE-2(config-ext-nacl)# no deny ip any any log
WAN-EDGE-2(config-ext-nacl)# permit tcp host 203.0.113.5 host 203.0.113.6 eq 179
WAN-EDGE-2(config-ext-nacl)# permit tcp host 203.0.113.5 eq 179 host 203.0.113.6
WAN-EDGE-2(config-ext-nacl)# deny ip any any log
```

**ยืนยัน:**

```
WAN-EDGE-2# show ip bgp summary | include 203.0.113.5
203.0.113.5     4 65002      10       9        3    0    0 00:00:30        1
```

ปัญหาที่ 1 หายไป — **แต่ต้องไล่ต่อ อย่าหยุดที่ปัญหาแรกที่เจอ** เพราะ NOC ยังรายงานอาการอื่นอยู่

### ขั้นที่ 3: แก้ปัญหาที่ 2 — next-hop-self หายไปที่ CORE-SW1 (ตาม Step 575)

```
WAN-EDGE-1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 25
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     2
  65000
    203.0.113.1 from 203.0.113.1 (1.1.1.254)
      Origin IGP, localpref 100, valid, external, best
```

Route จริงยัง `best` ที่ WAN-EDGE-1 ปกติดี (Path มาจาก eBGP ตรง) แต่ต้องดูว่า **ตอน Advertise
ต่อเข้า CORE-SW1 (iBGP)** เกิดอะไรขึ้น — ตรวจที่ CORE-SW1 โดยตรง:

```
CORE-SW1# show ip bgp neighbors 1.1.1.21 received-routes
BGP table version is 25, local router ID is 1.1.1.1
   Network          Next Hop            Metric LocPrf Weight Path
 * 0.0.0.0/0        203.0.113.1              0    100      0 65000 i

CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0
Paths: (1 available, no best path)
  65000
    203.0.113.1 (inaccessible) from 1.1.1.21 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, valid, internal
```

`(inaccessible)` — Signature เดียวกับ Step 575 พอดี ยืนยันว่า Network Team ที่ Config Neighbor
ใหม่ให้ CORE-SW1 ลืม `next-hop-self`:

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 next-hop-self
```

**ยืนยัน:**

```
CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0
Paths: (1 available, best #1, table default)
  65000
    1.1.1.21 from 1.1.1.21 (1.1.1.21)
      Origin IGP, metric 0, localpref 100, valid, internal, best

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  * 1.1.1.21, from 1.1.1.21, 00:00:04 ago
```

### ขั้นที่ 4: แก้ปัญหาที่ 3 — Prefix-list ของ Compliance กรอง Default Route ตัวเองออกไปด้วย (ตาม Step 576)

หลังแก้ 2 ข้อแรก NOC ยังรายงานว่า**บางครั้ง**ยังเข้า Internet ไม่ได้ — ตรวจ Inbound Filter
จาก ISP1 ที่ Compliance Team เพิ่งแก้ (ตาม Q3-Q4 ของ Decision Tree ฝั่ง Inbound):

```
WAN-EDGE-1(config)# ip prefix-list PL-APPROVED-IN seq 5 permit 203.0.113.0/24 ge 25 le 32
```

(Compliance ตั้งใจให้รับแค่ Prefix เฉพาะเจาะจงที่ "อนุมัติแล้ว" แต่ลืมว่า **Default Route
`0.0.0.0/0`** ที่ ISP1 ส่งมาเป็น Prefix Length `/0` ซึ่งไม่ Match เงื่อนไข `ge 25` เลย)

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 prefix-list PL-APPROVED-IN in
```

**วินิจฉัย:**

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 received-routes
Total number of prefixes 0

WAN-EDGE-1# show ip prefix-list detail PL-APPROVED-IN
ip prefix-list PL-APPROVED-IN:
   count: 1, range entries: 1, sequences: 5 - 5, refcount: 2
   seq 5 permit 203.0.113.0/24 ge 25 le 32 (hit count: 0, refcount: 2)
```

`received-routes` เป็น 0 ทั้งหมด (ต้องมี `soft-reconfiguration inbound` เปิดอยู่ก่อนถึงจะเห็นค่านี้
ได้ — ถ้ายังไม่เปิดให้เปิดก่อนด้วย `neighbor 203.0.113.1 soft-reconfiguration inbound` แล้ว
`clear ip bgp 203.0.113.1 soft in` เพื่อ Refresh) และ `hit count: 0` ยืนยันว่า ACE เดียวที่มี
ไม่ Match อะไรเลยรวมถึง Default Route ที่ ISP1 พยายามส่งมา

**การแก้ไข**: เพิ่ม ACE อนุญาต Default Route แยกไว้ให้ชัดเจน (ตามที่เคยตั้งใจไว้ตั้งแต่
[Part 29 Step 288](part-029-bgp-fundamentals.md#step-288--bgp-กับ-default-route))

```
WAN-EDGE-1(config)# ip prefix-list PL-APPROVED-IN seq 1 permit 0.0.0.0/0
WAN-EDGE-1(config)# clear ip bgp 203.0.113.1 soft in
```

### ขั้นที่ 5: ยืนยันผลรวมทั้งระบบหลังแก้ครบ 3 ปัญหา

```
WAN-EDGE-1# show ip bgp summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000     825     820       28    0    0 3d02h           1
1.1.1.1         4 65001      10       9        3    0    0 00:03:12        1
1.1.1.2         4 65001     150     148       28    0    0 02:14:00        4
1.1.1.22        4 65001     905     885       28    0    0 3d02h           2

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  * 1.1.1.21, from 1.1.1.21, 00:03:15 ago

CORE-SW2# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  * 1.1.1.21, from 1.1.1.21, 00:02:58 ago

WAN-EDGE-2# show ip bgp summary | include 203.0.113.5
203.0.113.5     4 65002      15      14        4    0    0 00:03:45        1
```

ทุก Session Established พร้อม `State/PfxRcd` เป็นตัวเลขทั้งหมด, CORE-SW1/CORE-SW2 ทั้งคู่มี
Default Route ใน RIB ผ่าน `1.1.1.21` (Next-Hop ที่ไปถึงได้จริงหลังแก้ next-hop-self), และ
WAN-EDGE-2/ISP2 กลับมา Established ปกติหลังแก้ ACL

### สรุปแนวทางวินิจฉัยจาก Lab นี้ — ใช้ Decision Tree ของ Step 571 นำทางเสมอ

```
1. show ip bgp summary ทุกตัวพร้อมกัน  → เจอ 2 อาการ: Active ค้าง (WAN-EDGE-2) และ PfxRcd=0 (WAN-EDGE-1→CORE-SW1)
2. แยกแก้ปัญหาที่ "ชั้นล่างสุด" ก่อนเสมอ → ACL/TCP 179 (Layer 4) ก่อน Attribute/Filter (BGP Layer)
3. หลังแก้ปัญหาแรก อย่าเข้าใจว่าจบ — วนกลับไป show ip bgp summary ใหม่ทุกครั้ง
4. ปัญหาที่ดูเหมือนเกี่ยวกัน (Internet เข้าไม่ได้) อาจมี Root Cause คนละที่กันโดยสิ้นเชิง
   และมาจาก 3 ทีมที่ไม่ได้ประสานงานกัน — เป็นเหตุผลที่ Change Management ต้องมี Peer Review เสมอ
```

---

## แบบฝึกหัดทวนความเข้าใจ Part 58

1. เมื่อ `show ip bgp summary` แสดง State เป็น **Active** ค้างอยู่นาน ให้ตรวจอะไรก่อนเป็นอันดับแรก
   และเพราะเหตุใดจึงต้องแยกการทดสอบ Layer 3 (`ping`) ออกจาก Layer 4 (`telnet <ip> 179`)?
2. คำว่า **`(inaccessible)`** ที่ปรากฏใน `show ip bgp <prefix>` บอกอะไร และคำสั่งใดที่ใช้แก้ปัญหา
   นี้บน Router ที่เป็นจุดเชื่อมระหว่าง eBGP กับ iBGP?
3. เพราะเหตุใด `ip prefix-list PL-X seq 10 permit 203.0.113.0/24 ge 25` จึง**ไม่**อนุญาตให้
   Prefix `203.0.113.0/24` (เป๊ะ) ผ่านได้ ทั้งที่ดูเหมือนน่าจะครอบคลุมอยู่แล้ว?
4. อธิบายความแตกต่างระหว่าง WEIGHT กับ LOCAL_PREF ในบริบทของการ Troubleshoot Asymmetric Routing
   — ทำไม WEIGHT ที่ตั้งบน Router ตัวเดียวจึงทำให้ Router อื่นในองค์กรเลือก Path ต่างกันได้?
5. Neighbor ที่ State เป็น `Idle (PfxCt)` ต่างจาก `Idle` ธรรมดาอย่างไร และต้องทำอะไรเพื่อให้
   Session กลับมา Established ได้อีกครั้ง?

**เฉลย:**

1. ต้องตรวจ **`ping`** ก่อนเพื่อยืนยัน Layer 3/Routing ไปถึง Peer ได้จริง จากนั้นตรวจ
   **`telnet <peer-ip> 179`** เพื่อทดสอบว่า TCP Port 179 เปิดถึงกันจริงหรือไม่ **แยกจาก BGP
   Process โดยสิ้นเชิง** — ถ้า Ping ผ่านแต่ Telnet Time Out แสดงว่ามี ACL/Firewall กรอง TCP 179
   อยู่ระหว่างทาง (Step 573) ถ้า Ping ไม่ผ่านเลยแสดงว่าไม่มี Route ถึง Peer เลย (Step 574) —
   การแยก 2 ชั้นนี้ทำให้ไม่เสียเวลาสงสัย BGP Config ทั้งที่ปัญหาอยู่ที่ Layer ต่ำกว่า
2. `(inaccessible)` บอกว่า **Next-Hop ของ Route นี้ไม่มี Route ใน RIB ให้ไปถึงได้** ทำให้ Path
   ไม่ถูกเลือกเป็น Best Path และไม่ถูกติดตั้งใน Routing Table แม้ Route จะ `valid` ก็ตาม — แก้ไข
   ด้วยคำสั่ง **`neighbor <ip> next-hop-self`** บน Router ที่เป็นจุดเชื่อม eBGP/iBGP (เช่น
   WAN-EDGE-1/2) เพื่อเขียน Next-Hop ใหม่เป็น IP ของตัวเองที่ IGP ภายในพาไปถึงได้แน่นอน
3. เพราะ `ge 25` กำหนดว่า **ต้อง Match Prefix Length ตั้งแต่ /25 ขึ้นไปเท่านั้น** — Prefix
   `/24` เป๊ะไม่เข้าเงื่อนไขนี้ (สั้นกว่า /25) จึงตกไปโดน Implicit Deny เสมอ ต้องมี ACE แยกที่
   Match `/24` แบบเป๊ะ (ไม่มี `ge`/`le`) ไว้ด้วยเสมอถ้าต้องการให้ Prefix หลักตัวเองผ่านได้
4. **WEIGHT เป็น Cisco Proprietary Attribute ที่อยู่ Local เฉพาะ Router ตัวเดียว ไม่ถูกส่งต่อให้
   ใครเลยแม้ผ่าน iBGP** ในขณะที่ **LOCAL_PREF ถูกส่งต่อผ่าน iBGP ให้ทุก Router ในกลุ่มเห็นค่า
   เดียวกัน** — ถ้า Router ตัวหนึ่งมี WEIGHT ที่ Router อื่นไม่มี (และ WEIGHT ถูกเทียบในขั้นที่ 1
   ก่อน LOCAL_PREF ในขั้นที่ 2 เสมอ) Router ตัวนั้นจะเลือก Path ต่างจากที่ LOCAL_PREF บอกไว้
   ทำให้เกิด Asymmetric Routing ข้าม Router ในองค์กรเดียวกัน
5. `Idle (PfxCt)` หมายถึง Session ถูกปิดเพราะชน **Maximum-Prefix Limit** (ต่างจาก `Idle`
   ธรรมดาที่อาจเกิดจาก Reset ทั่วไปแล้วพยายาม Reconnect เองต่อ) — Session แบบนี้**จะไม่ฟื้นเอง
   อัตโนมัติ** จนกว่าจะสั่ง `clear ip bgp <neighbor>` ด้วยมือ (หรือถ้าตั้ง Option `restart <นาที>`
   ไว้ตั้งแต่แรก Router จะพยายาม Reconnect เองตามเวลาที่กำหนด) และควรปรับ `maximum-prefix`
   ให้มี Headroom/Threshold เตือนล่วงหน้าเพื่อป้องกันไม่ให้เกิดซ้ำ

---

## สรุป Part 58

Part นี้ฝึกวินิจฉัย **BGP ผ่าน 8 Broken Scenario ที่ครอบคลุมทุกจุด Fail หลักของ Protocol** โดยใช้
**Decision Tree เดียวกันตลอด** (TCP Session → Neighbor State → Route Received → Route Installed
→ Best Path ที่ถูกต้อง → Advertised ต่อ): AS Number พิมพ์ผิดทำให้เกิด Notification Bad-Peer-AS,
ACL บล็อก TCP 179 อย่างเงียบที่แยกวินิจฉัยได้ด้วย `telnet <ip> 179`, iBGP ไม่ Establish เพราะ
IGP ไม่พา Loopback ของ Peer ให้ถึงกัน, Route ติด Flag `(inaccessible)` เพราะลืม `next-hop-self`,
Prefix-list/AS-Path Filter บล็อก Route จาก Seq/ge-le/Direction ที่ผิดพลาด, WEIGHT ที่ไม่สอดคล้อง
กันข้าม Router ทำให้เกิด Asymmetric Routing, BGP Flapping จาก Maximum-Prefix และ MTU Mismatch,
และ Route ที่เป็น Best Path แล้วแต่ไม่ถูก Advertise ต่อเพราะลืม `send-community` หรือ Route-map
Deny ที่ครอบเกินเป้า ปิดท้ายด้วย **Lab เต็มรูปแบบที่ผสมทั้ง 3 ปัญหาเข้าด้วยกัน** พิสูจน์ว่า
Methodology ของ Step 571 ใช้แก้ปัญหาซ้อนกันหลายชั้นได้จริงโดยไม่หลงทาง ✅

**สิ่งที่ Part นี้ยังไม่ครอบคลุม**: ทุก Scenario ใน Part นี้สมมติว่า BGP เป็น Protocol เดียวที่
ทำงานอยู่ — แต่ Lab จริงของหลักสูตรมี **OSPF/EIGRP ทำงานร่วมกับ BGP ผ่าน Redistribution**
(ที่ CORE-SW1/CORE-SW2 ต้อง Redistribute BGP เข้า OSPF ตามที่ [Part 30](part-030-bgp-advanced.md)
ทิ้งท้ายไว้) จุดนี้เปิดช่องให้เกิด **Routing Loop, Route Feedback, และ Metric ที่ไม่สอดคล้องกัน
ข้าม Protocol** ซึ่งเป็นปัญหาคนละชนิดจากที่เจอใน Part นี้โดยสิ้นเชิง — Part ถัดไปจะเจาะลึกเรื่องนี้
โดยเฉพาะ

**ไปต่อ:** [Part 59 — Route Redistribution Troubleshooting →](part-059-redistribution-troubleshooting.md)
