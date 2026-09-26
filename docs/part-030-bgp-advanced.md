# Part 30 — BGP Advanced (iBGP, Route Reflectors, Attributes, Path Selection)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 291–300 จาก 1000**

> ต่อจาก [Part 29 — BGP Fundamentals](part-029-bgp-fundamentals.md) ที่เปิดใช้งาน **eBGP**
> บน WAN-EDGE-1 และ WAN-EDGE-2 (Enterprise AS **65001**) ให้ peer กับ Upstream ISP ผ่าน
> ISP-RTR — WAN-EDGE-1 เชื่อมไปยัง **ISP1 (AS 65000)** ที่ subnet `203.0.113.0/30` และ
> WAN-EDGE-2 เชื่อมไปยัง **ISP2 (AS 65002)** ที่ subnet `203.0.113.4/30` ทำให้ Enterprise
> เป็น **Dual-Homed to Dual ISP** ตามมาตรฐาน High Availability WAN Design (ใน Lab จำลองทั้งสอง
> ISP ด้วยอุปกรณ์ ISP-RTR ตัวเดียวเพื่อประหยัดทรัพยากร — ในการออกแบบจริงจะเป็นอุปกรณ์ของผู้ให้บริการ
> อินเทอร์เน็ตคนละรายกันจริง) ปัญหาที่ Part 29 ยังไม่ได้แก้คือ: **Route ที่ WAN-EDGE-1 เรียนรู้จาก
> ISP1 ยังไปไม่ถึง WAN-EDGE-2, CORE-SW1, CORE-SW2** เลย เพราะมีแค่ eBGP เท่านั้น ยังไม่มีกลไก
> กระจาย Route ภายใน AS 65001 — Part นี้คือคำตอบ: เราจะเปิดใช้งาน **iBGP (Internal BGP)**
> เจาะลึก **Full-Mesh Requirement**, **Route Reflector**, **BGP Attribute ทั้งหมด** และ
> **Best Path Selection Algorithm** แบบละเอียดที่สุด ปิดท้ายด้วย Lab เต็มรูปแบบที่พิสูจน์ว่า
> Enterprise เลือกเส้นทางออก-เข้า Internet ได้ตามที่ Design ต้องการจริง

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 291 | iBGP คืออะไร — Full-Mesh Requirement และทำไมต้องมี `next-hop-self` |
| 292 | Lab จริง: iBGP Full-Mesh ระหว่าง WAN-EDGE-1 ↔ WAN-EDGE-2 |
| 293 | Route Reflector (RR) — แก้ปัญหา Full-Mesh Scaling |
| 294 | BGP Confederation — ทางเลือกอื่นแทน Route Reflector |
| 295 | BGP Best Path Selection Algorithm แบบเต็มรูปแบบ พร้อมตัวอย่างตัวเลข |
| 296 | กำหนดเส้นทางขาออก (Outbound) ด้วย WEIGHT และ LOCAL_PREF |
| 297 | กำหนดเส้นทางขาเข้า (Inbound) ด้วย AS-Path Prepending และ MED |
| 298 | BGP Communities — Well-Known และ Custom Community |
| 299 | Verification & Troubleshooting BGP Path Selection |
| 300 | Lab เต็มรูปแบบ: iBGP Mesh + Path Control ที่พิสูจน์ผลจริงทั้งสองทิศทาง |

---

## Step 291 — iBGP คืออะไร — Full-Mesh Requirement และทำไมต้องมี `next-hop-self`

### 291.1 eBGP vs iBGP

BGP มี Session สองชนิดตาม AS Number ของเพื่อนบ้าน (`remote-as`):

| คุณสมบัติ | eBGP (External BGP) | iBGP (Internal BGP) |
|---|---|---|
| AS ของเพื่อนบ้าน | `remote-as` **ต่างจาก** Local AS | `remote-as` **เดียวกัน** กับ Local AS |
| Administrative Distance | 20 | 200 |
| TTL เริ่มต้น | 1 (ต้อง `ebgp-multihop` ถ้าไม่ใช่ Directly Connected) | 255 (ไปได้ไกลผ่าน IGP อยู่แล้ว) |
| การเปลี่ยน NEXT_HOP | เปลี่ยนเป็น IP ของตัวเองเสมอเมื่อ advertise ออก | **ไม่เปลี่ยน** โดย default |
| การเพิ่ม AS ใน AS_PATH | เพิ่ม Local AS เข้าไปเสมอ | **ไม่เพิ่ม** (อยู่ AS เดียวกัน) |
| ต้อง Full-Mesh หรือไม่ | ไม่ (peer เป็นคู่ๆ ตาม Physical Link) | **ต้อง** (หรือใช้ RR/Confederation แทน) |
| ใช้ทำอะไรใน Lab นี้ | WAN-EDGE-1/2 ↔ ISP-RTR (Part 29) | WAN-EDGE-1/2 ↔ CORE-SW1/2 (Part นี้) |

จุดสำคัญที่สุดคือสองแถวสุดท้าย และเป็นเหตุผลที่ Part 29 ยังไม่พอ — ต้องมี Part 30 ต่อ

### 291.2 กฎ Split-Horizon ของ iBGP (BGP Synchronization Loop-Prevention Rule)

BGP ไม่มี Metric/Cost แบบ IGP ที่ป้องกัน Loop ได้เองทั้งหมด กลไกป้องกัน Loop หลักของ BGP คือ
**AS_PATH** — Router จะไม่รับ Route ที่มี AS ของตัวเองอยู่ใน AS_PATH อยู่แล้ว (ป้องกัน Loop ระหว่าง AS)
แต่ **ภายใน AS เดียวกัน (iBGP) AS_PATH ไม่ถูกเพิ่มเลย** ดังนั้น BGP จึงต้องมีกฎอีกข้อมาป้องกัน Loop
แทน นั่นคือ:

> **กฎเหล็กของ iBGP**: Router ที่เรียนรู้ Route ผ่าน **iBGP** จะ **ห้าม Advertise route นั้นต่อไปยัง
> iBGP Peer ตัวอื่นอีก** (ส่งต่อได้เฉพาะไปยัง eBGP Peer เท่านั้น)

ผลลัพธ์ของกฎนี้คือ **Full-Mesh Requirement**: ถ้า Router A เรียนรู้ Route จาก eBGP แล้วต้องการให้
Router B และ Router C (อยู่ AS เดียวกัน) เห็น Route นี้ด้วย **A ต้อง peer iBGP กับ B และ C โดยตรง
ทั้งคู่** — B จะไม่สามารถเรียนรู้ Route นี้ผ่าน C ได้เลย แม้ B-C จะมี iBGP session กันอยู่ก็ตาม
เพราะ C ได้ Route นี้มาจาก iBGP (จาก A) จึงห้ามส่งต่อให้ B ตามกฎข้างบน

### 291.3 สูตรคำนวณจำนวน Session ของ Full-Mesh — ปัญหา Scalability

จำนวน iBGP Session ที่ต้องสร้างสำหรับ Full-Mesh คำนวณจาก:

```
จำนวน Session = n × (n-1) / 2        (n = จำนวน iBGP Router ทั้งหมดใน AS)
```

| จำนวน Router (n) | จำนวน Session ที่ต้อง Config |
|---|---|
| 2 (WAN-EDGE-1, WAN-EDGE-2) | 1 |
| 4 (+ CORE-SW1, CORE-SW2) | 6 |
| 10 | 45 |
| 50 | 1,225 |
| 100 | 4,950 |

Lab หลักสูตรนี้มี iBGP Router 4 ตัว (WAN-EDGE-1, WAN-EDGE-2, CORE-SW1, CORE-SW2) จึงต้องการ
**6 Session** เต็ม Full-Mesh — ยังจัดการไหวใน Step 292 แต่ Step 293 จะแสดงให้เห็นว่าเมื่อ Enterprise
โตขึ้น (เพิ่ม DIST-SW เป็น iBGP Router ด้วย หรือมี Data Center Site ที่สอง) ตัวเลขนี้จะระเบิดขึ้น
แบบ Quadratic ทันที — นี่คือเหตุผลที่ Real-World Enterprise ระดับใหญ่ **ไม่ทำ Full-Mesh** แต่ใช้
**Route Reflector** แทน

### 291.4 ทำไมต้องมี `next-hop-self`

eBGP จะเปลี่ยน NEXT_HOP ของ Route เป็น IP ของตัวเองเสมอเมื่อ Advertise ออกไป แต่ **iBGP ไม่เปลี่ยน
NEXT_HOP โดย Default** — ลองไล่ตามสถานการณ์จริงของ Lab นี้:

```
ISP1 (AS 65000)  --eBGP-->  WAN-EDGE-1  --iBGP-->  CORE-SW1
   203.0.113.1                203.0.113.2            1.1.1.1

1. ISP1 advertise prefix 8.8.8.0/24 ให้ WAN-EDGE-1 ผ่าน eBGP
   NEXT_HOP ของ route นี้ตาม WAN-EDGE-1 = 203.0.113.1 (IP ของ ISP1 เอง)

2. WAN-EDGE-1 advertise route 8.8.8.0/24 ต่อให้ CORE-SW1 ผ่าน iBGP
   ถ้าไม่ทำอะไรเพิ่ม NEXT_HOP ยังคงเป็น 203.0.113.1 (ไม่เปลี่ยน เพราะเป็น iBGP)

3. CORE-SW1 ได้ route 8.8.8.0/24 next-hop 203.0.113.1
   แต่ CORE-SW1 "ไม่มีทางไปถึง 203.0.113.1" เลย เพราะ subnet /30 นี้เป็น Transit link
   ระหว่าง WAN-EDGE-1 ↔ ISP1 เท่านั้น ไม่ได้ถูก Advertise เข้า IGP (OSPF) ของ Enterprise

ผลลัพธ์ -> Route ใน BGP Table ของ CORE-SW1 จะขึ้นสถานะ "next hop not reachable"
         และจะไม่ถูกนำไปติดตั้งใน Routing Table (RIB) เลย แม้ BGP จะบอกว่ามี route ก็ตาม
```

**คำสั่งแก้ปัญหา**: `neighbor <ip> next-hop-self` — สั่งให้ Router **เขียน NEXT_HOP ใหม่เป็น IP
ของตัวเอง** (โดยปกติคือ Interface ที่ใช้ทำ Session หรือ Loopback ถ้าใช้ `update-source`) ก่อน
Advertise เข้า iBGP Mesh ทำให้ CORE-SW1 เห็น NEXT_HOP = Loopback0 ของ WAN-EDGE-1 (1.1.1.21)
ซึ่ง **ไปถึงได้แน่นอนผ่าน OSPF ภายใน Enterprise**

> **กฎจำง่าย**: ทุก Router ที่เป็น "ประตู" ระหว่าง eBGP กับ iBGP (ในที่นี้คือ WAN-EDGE-1,
> WAN-EDGE-2) **ต้องใส่ `next-hop-self` เสมอ** บน Neighbor Statement ฝั่ง iBGP — เป็นคำสั่งที่
> ลืมไม่ได้อันดับ 1 เวลาทำ iBGP จริง

---

## Step 292 — Lab จริง: iBGP Full-Mesh ระหว่าง WAN-EDGE-1 ↔ WAN-EDGE-2

เริ่มจาก Session ที่ง่ายที่สุดก่อน: iBGP ระหว่าง WAN-EDGE-1 และ WAN-EDGE-2 ทั้งคู่อยู่ AS 65001
เหมือนกัน ใช้ Loopback0 เป็น Source (ตาม `00-ip-address-plan.md`: WAN-EDGE-1 = `1.1.1.21/32`,
WAN-EDGE-2 = `1.1.1.22/32`) เพื่อให้ Session **ไม่ล่มถ้า Physical Link เส้นใดเส้นหนึ่งตาย** — ตราบใด
ที่ OSPF ยังหาเส้นทางอื่นไปยัง Loopback ได้ (Lab นี้มีทั้ง Cross-link ตรง `10.10.254.8/30` และเส้นทาง
อ้อมผ่าน CORE-SW1 ↔ CORE-SW2 ↔ อีกฝั่ง)

### 292.1 Config บน WAN-EDGE-1

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp router-id 1.1.1.21
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 description iBGP to WAN-EDGE-2
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.22 next-hop-self
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 next-hop-self
WAN-EDGE-1(config-router-af)# exit-address-family
```

### 292.2 Config บน WAN-EDGE-2 (Mirror)

```
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# bgp router-id 1.1.1.22
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 remote-as 65001
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 description iBGP to WAN-EDGE-1
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 update-source Loopback0
WAN-EDGE-2(config-router)# neighbor 1.1.1.21 next-hop-self
WAN-EDGE-2(config-router)# address-family ipv4 unicast
WAN-EDGE-2(config-router-af)# neighbor 1.1.1.21 activate
WAN-EDGE-2(config-router-af)# neighbor 1.1.1.21 next-hop-self
WAN-EDGE-2(config-router-af)# exit-address-family
```

> **ทำไมต้องใส่ `update-source Loopback0`**: eBGP ปกติใช้ IP ของ Physical Interface เป็น Source
> โดย TTL=1 พอสำหรับ Directly-connected Link แต่ iBGP มักไม่ได้ Direct-connect เสมอไป (ในที่นี้
> WAN-EDGE-1 ↔ WAN-EDGE-2 มี Cross-link ตรงจริง แต่ Best Practice ยังใช้ Loopback เพื่อความทนทาน
> ต่อ Link failure — ถ้า Cross-link ตายแต่ OSPF ยังพาไปถึง Loopback อ้อมทาง CORE ได้ Session
> จะไม่ขาด) และเพราะเราอ้างอิง Neighbor ด้วย Loopback (`1.1.1.22`) จึงต้องมี `update-source`
> ไม่งั้น Router จะพยายามส่ง TCP SYN จาก Physical Interface ที่ไม่ตรงกับที่อีกฝั่งคาดหวัง
> ทำให้ Session ค้างที่ **Active** state ตลอด

### 292.3 ตรวจสอบผลลัพธ์

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4  65000     150     148        12    0    0 02:14:10        3
1.1.1.22        4  65001      45      44        12    0    0 00:08:02        2
```

`State/PfxRcd = 2` แสดงว่า WAN-EDGE-1 ได้ Route จาก WAN-EDGE-2 มา 2 Prefix แล้ว (Route ที่
WAN-EDGE-2 เรียนรู้จาก ISP2) — พิสูจน์ว่า iBGP กระจาย Route ข้าม WAN-EDGE ทั้งสองตัวสำเร็จ

```
WAN-EDGE-1# show ip bgp neighbors 1.1.1.22
BGP neighbor is 1.1.1.22,  remote AS 65001, internal link
  BGP version 4, remote router ID 1.1.1.22
  BGP state = Established, up for 00:08:15
  Last read 00:00:12, last write 00:00:18, hold time is 180
  Neighbor capabilities:
    Route refresh: advertised and received
  Update source is Loopback0
```

```
WAN-EDGE-1# show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 0.0.0.0/0        203.0.113.1              0             0 65000 i
*  0.0.0.0/0        1.1.1.22                 0    100      0 65002 i
```

สังเกต **Next Hop ของ Route ที่มาจาก WAN-EDGE-2 = `1.1.1.22`** (Loopback ของ WAN-EDGE-2 เอง)
ไม่ใช่ `203.0.113.5` (IP ของ ISP2) — เพราะ `next-hop-self` บน WAN-EDGE-2 ทำงานแล้ว และเพราะ
1.1.1.22/32 ถูก Advertise ผ่าน OSPF อยู่แล้ว (Loopback ทุกใบใน Enterprise) WAN-EDGE-1 จึงมีเส้นทาง
ไปถึงได้ทันที Route นี้จึงถูกนำไปใช้งานได้จริงใน RIB

---

## Step 293 — Route Reflector (RR) — แก้ปัญหา Full-Mesh Scaling

### 293.1 ทบทวนปัญหา — เพิ่ม CORE-SW1/CORE-SW2 เข้ามาใน iBGP

ถ้าต้องการให้ **CORE-SW1 และ CORE-SW2** เห็น Route จาก Internet ด้วย (จำเป็น เพราะ Part 31 จะ
Redistribute BGP เข้า OSPF ที่ CORE-SW1/CORE-SW2 เพื่อแจก Default Route ต่อไปยัง DIST-SW) วิธี
Full-Mesh ต้องสร้างครบ 6 Session ตามที่คำนวณใน Step 291.3:

```
Full-Mesh 4 โหนด (6 Session):

   WAN-EDGE-1 ─────────── WAN-EDGE-2
      │  \                 /  │
      │    \             /    │
      │      \         /      │
      │        \     /        │
      │          \ /          │
      │          / \          │
      │        /     \        │
      │      /         \      │
      │    /             \    │
      │  /                 \  │
   CORE-SW1 ─────────────── CORE-SW2
```

Config ที่ต้องเพิ่ม (สรุปย่อ ไม่รวม Step 292 ที่ทำไปแล้ว):

| Session | ต้อง Config บน |
|---|---|
| WAN-EDGE-1 ↔ CORE-SW1 | ทั้งสองฝั่ง |
| WAN-EDGE-1 ↔ CORE-SW2 | ทั้งสองฝั่ง |
| WAN-EDGE-2 ↔ CORE-SW1 | ทั้งสองฝั่ง |
| WAN-EDGE-2 ↔ CORE-SW2 | ทั้งสองฝั่ง |
| CORE-SW1 ↔ CORE-SW2 | ทั้งสองฝั่ง |

4 โหนดยังพอไหว แต่ลองจินตนาการว่า Enterprise นี้ขยายเป็น 10 Site (Loopback ของ DIST-SW แต่ละ Site
ก็อยากมี BGP Route ด้วย) — ต้องเพิ่มเป็น 45 Session ทันที และทุกครั้งที่เพิ่ม Router iBGP ใหม่ 1 ตัว
ต้องไปแก้ Config **ทุกตัวที่มีอยู่เดิม** — นี่คือปัญหาที่ Real-World Enterprise/ISP ทุกรายเจอ และ
คำตอบมาตรฐานคือ **Route Reflector (RR)**

### 293.2 แนวคิด Route Reflector

RR คือ iBGP Router พิเศษที่ **ได้รับการยกเว้นกฎ Split-Horizon ของ iBGP** (Step 291.2) — RR
สามารถรับ Route จาก iBGP Peer ตัวหนึ่ง แล้ว **"สะท้อน" (Reflect)** ต่อไปยัง iBGP Peer ตัวอื่นได้
ทำให้ Router ทั่วไปไม่ต้อง Full-Mesh กันเอง เพียง Peer กับ RR ตัวเดียวก็พอ

| บทบาท | ความหมาย |
|---|---|
| **Route Reflector (RR)** | Router ที่ทำหน้าที่สะท้อน Route — ในตัวอย่างคือ WAN-EDGE-1 |
| **RR-Client** | Router ที่ Peer กับ RR และ **ไม่ต้อง Full-Mesh กับ Client ตัวอื่น** — CORE-SW1, CORE-SW2 |
| **Non-Client (Peer ธรรมดา)** | iBGP Peer ปกติของ RR ที่ไม่ใช่ Client — WAN-EDGE-2 |

### 293.3 กฎการ Reflect (แก้ Split-Horizon แบบมีเงื่อนไข)

| Route มาจาก | RR จะ Reflect ต่อไปยัง |
|---|---|
| RR-Client ตัวหนึ่ง | RR-Client ตัวอื่นทุกตัว **+** Non-Client ทุกตัว |
| Non-Client | RR-Client ทุกตัวเท่านั้น (**ไม่** ส่งต่อไปยัง Non-Client ตัวอื่น — ยังคง Split-Horizon เดิมระหว่าง Non-Client) |
| eBGP (ภายนอก) | ทั้ง RR-Client และ Non-Client ทุกตัว (เหมือน iBGP ปกติ) |

จุดนี้เองที่ทำให้ Non-Client ยังคงต้อง Full-Mesh กันเองตามปกติ (ในตัวอย่างนี้มี Non-Client แค่
WAN-EDGE-2 ตัวเดียวจึงไม่มีปัญหา) — RR ช่วยลด Mesh เฉพาะฝั่ง Client เท่านั้น

### 293.4 ตัวอย่าง Config (Conceptual) — WAN-EDGE-1 เป็น RR ให้ CORE-SW1/CORE-SW2

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.1 update-source Loopback0
WAN-EDGE-1(config-router)# neighbor 1.1.1.2 remote-as 65001
WAN-EDGE-1(config-router)# neighbor 1.1.1.2 update-source Loopback0
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 next-hop-self
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.1 route-reflector-client
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 next-hop-self
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.2 route-reflector-client
WAN-EDGE-1(config-router-af)# exit-address-family
```

ผลลัพธ์: จำนวน Session ลดจาก 6 เหลือ 3 (WAN-EDGE-1↔WAN-EDGE-2, WAN-EDGE-1↔CORE-SW1,
WAN-EDGE-1↔CORE-SW2) — CORE-SW1 และ CORE-SW2 **ไม่ต้อง peer กันเองเลย** เพราะ WAN-EDGE-1
สะท้อน Route ระหว่างกันให้แทน

> **ทำไมเรียกว่า "Conceptual" ใน Part นี้**: Lab หลักสูตรมี iBGP Router แค่ 4 ตัว การทำ Full-Mesh
> (6 Session) ยังจัดการได้ไม่ยาก Step 300 (Lab เต็มรูปแบบท้าย Part) จึงยังคงใช้ **Full-Mesh**
> เพื่อให้เข้าใจกลไก iBGP ล้วนๆ ก่อน — RR จะกลับมาเป็น Production-grade Design จริงเมื่อ
> Enterprise นี้ขยายเป็น Multi-Site ใน **Part 33-35 (Advanced Campus Design & HA)** ซึ่งเป็นจุด
> ที่จำนวน iBGP Router จะมากพอจนต้องใช้ RR แก้ปัญหา Scaling อย่างจริงจัง

### 293.5 Cluster-ID, Originator-ID, Cluster-List — กันลูปของ RR เอง

เมื่อมี RR มากกว่า 1 ตัวในกลุ่มเดียวกัน (Redundant RR) ต้อง Config `bgp cluster-id <ID>` ให้เหมือนกัน
ทุกตัวในกลุ่ม เพื่อป้องกัน Loop — RR จะแนบ Attribute พิเศษ 2 ตัวไปกับ Route ที่ Reflect:

- **ORIGINATOR_ID**: Router-ID ของ Router ตัวแรกที่ Originate Route เข้ามาใน AS (ถ้า Client
  ได้รับ Route ที่ ORIGINATOR_ID ตรงกับตัวเอง จะ Drop ทิ้งทันที — ป้องกัน Loop)
- **CLUSTER_LIST**: รายการ Cluster-ID ที่ Route นี้เคยผ่านมา (คล้าย AS_PATH แต่ระดับ Cluster —
  ถ้า RR เห็น Cluster-ID ตัวเองอยู่ใน List แล้ว จะ Drop ทิ้งเช่นกัน)

---

## Step 294 — BGP Confederation — ทางเลือกอื่นแทน Route Reflector

### 294.1 แนวคิด Confederation

แทนที่จะใช้ RR ก็สามารถ **แบ่ง AS ใหญ่ 1 AS ออกเป็น Sub-AS หลายตัว** โดยภายนอกยังคงเห็นเป็น AS
เดียว (Public AS Number) — ภายใน Sub-AS แต่ละตัวใช้ iBGP Full-Mesh ตามปกติ (Mesh เล็กลงเพราะ
แบ่งเป็นกลุ่มย่อย) ส่วนระหว่าง Sub-AS ใช้ Session ที่พฤติกรรมคล้าย eBGP (เปลี่ยน NEXT_HOP,
Reset บาง Timer) แต่ AS Number ของ Sub-AS (Private-use, เช่น 65001.1, 65001.2) จะถูก **ลบออก
จาก AS_PATH ก่อน Advertise ออกสู่ภายนอก** เหลือแต่ AS Number สาธารณะตัวเดียว

```
                    ภายนอกเห็นเป็น AS 65001 เดียว
                              │
        ┌─────────────────────┴─────────────────────┐
        │              Confederation 65001            │
        │  ┌──────────────┐        ┌──────────────┐   │
        │  │ Sub-AS 65001.1│──────│ Sub-AS 65001.2 │   │
        │  │ (Mesh เล็ก)   │ eBGP-like            │   │
        │  └──────────────┘        └──────────────┘   │
        └───────────────────────────────────────────┘
```

### 294.2 เปรียบเทียบ RR vs Confederation

| หัวข้อ | Route Reflector | Confederation |
|---|---|---|
| แก้ปัญหา | Full-Mesh Scaling | Full-Mesh Scaling |
| ความซับซ้อนของ Config | ต่ำ (เพิ่มคำสั่งเดียวที่ RR) | สูง (ต้องออกแบบ Sub-AS ใหม่ทั้งหมด) |
| กระทบ AS_PATH ที่เห็นจากภายนอก | ไม่กระทบ | Sub-AS ถูกซ่อน (ไม่เห็นจากภายนอก) |
| Migration จาก Design เดิม | ง่าย (ไม่ต้องเปลี่ยน AS เดิม) | ยาก (ต้องแบ่ง AS ใหม่) |
| นิยมใช้ที่ระดับ | Enterprise ทุกขนาด, ISP ทั่วไป | ISP ขนาดใหญ่มาก / Tier-1 ที่มี Router หลักพันตัว |
| ใช้ใน Lab หลักสูตรนี้หรือไม่ | ใช่ (Part 33-35) | ไม่ใช้ (เกินขอบเขต Enterprise ทั่วไป) |

**สรุป**: Confederation เหมาะกับ ISP ขนาดมหาศาลที่มี Router เป็นพันตัวและต้องการแบ่งทีมดูแลตาม
ภูมิภาค (แต่ละ Sub-AS ดูแลแยกกันได้) ส่วน Enterprise ทั่วไป (รวมถึง Lab หลักสูตรนี้) RR เพียงพอ
และง่ายกว่ามาก — หัวข้อนี้ในระดับ CCNP ENCOR ให้รู้จักไว้เพื่อแยกแยะกับ RR ได้ ส่วนรายละเอียดการ
Config Confederation เต็มรูปแบบจะอยู่ในเนื้อหาระดับ **CCIE Service Provider**

---

## Step 295 — BGP Best Path Selection Algorithm แบบเต็มรูปแบบ

BGP ไม่ใช้ Metric เดียวในการเลือกเส้นทางเหมือน OSPF/EIGRP แต่ใช้ **Attribute หลายตัวเรียงลำดับ
ความสำคัญ** เทียบทีละขั้นจนกว่าจะมีขั้นที่ตัดสินผลได้ (ถ้าเท่ากันไปขั้นถัดไป) นี่คือลำดับเต็มที่ Cisco
IOS ใช้จริง (ต้องจำลำดับนี้ให้ขึ้นใจ เป็นหัวข้อสอบยอดฮิตของ CCNP ENCOR):

| ลำดับ | Attribute / เงื่อนไข | เลือก Path ที่... | ขอบเขตผลกระทบ |
|---|---|---|---|
| 1 | **WEIGHT** | สูงกว่า | Local เฉพาะ Router (Cisco Proprietary, ไม่ถูกส่งต่อให้ใคร) |
| 2 | **LOCAL_PREF** | สูงกว่า | ทั้ง AS (ส่งต่อผ่าน iBGP เท่านั้น ไม่ส่งออก eBGP) |
| 3 | Locally Originated | Route ที่มาจาก `network`, Aggregate, Redistribute บน Router นี้เอง | Local |
| 4 | **AS_PATH** | สั้นกว่า (จำนวน AS น้อยกว่า) | Global |
| 5 | **ORIGIN** | IGP < EGP < Incomplete (IGP ดีที่สุด) | Global |
| 6 | **MED (Metric)** | ต่ำกว่า (เทียบเฉพาะ Path จาก AS เพื่อนบ้านเดียวกัน เว้นแต่เปิด `bgp always-compare-med`) | ระหว่าง 2 AS ที่ติดกัน |
| 7 | eBGP vs iBGP | เลือก **eBGP** ก่อน iBGP เสมอ | Local |
| 8 | IGP Metric ไปยัง NEXT_HOP | ต่ำกว่า | Local |
| 9 | Path ที่เก่ากว่า (สำหรับ eBGP) | มาถึงก่อน (Stable กว่า) — ข้ามขั้นนี้ถ้าเปิด `bgp bestpath compare-routerid` หรือใช้ Multipath | Local |
| 10 | Router-ID ต่ำสุด (หรือ Originator-ID ถ้าเป็น Route ที่ผ่าน RR) | ต่ำกว่า | Global (Tie-break สุดท้ายเกือบทุกกรณี) |
| 11 | Cluster-List สั้นกว่า | สั้นกว่า | เฉพาะ Route ที่ผ่าน RR |
| 12 | Neighbor IP Address ต่ำสุด | ต่ำกว่า | Local (Fallback สุดท้ายจริงๆ) |

> **เทคนิคจำ**: **W**eight **L**ocal **O**rigin **A**s-path **O**rigin(type) **M**ed **E**bgp
> **I**gp-metric **O**ldest **R**outer-id → นิยมย่อว่า **"Where Legs Or Arms Or Medicine, Eat
> Ice-cream, Old Rats"** (Mnemonic แบบ Cisco Community ที่ใช้กันบ่อย)

### 295.1 ตัวอย่างตัวเลข — 3 Candidate Path ไปยัง Prefix เดียวกัน

สมมติ WAN-EDGE-1 เห็น Path ไปยัง Prefix `8.8.8.0/24` จาก 3 แหล่งพร้อมกัน:

| Path | ที่มา | WEIGHT | LOCAL_PREF | AS_PATH | ORIGIN |
|---|---|---|---|---|---|
| **A** | eBGP จาก ISP1 โดยตรง | 0 (default eBGP) | 200 (ตั้งโดย Policy ขาเข้า — ดู Step 296) | `65000` (ยาว 1) | IGP |
| **B** | iBGP จาก WAN-EDGE-2 (เรียนมาจาก ISP2) | 0 | 100 (default, ไม่ได้ตั้ง Policy) | `65002 65003` (ยาว 2) | IGP |
| **C** | iBGP จาก CORE-SW1 (สำเนา Route เดียวกับ A ที่ถูกสะท้อนวนมา) | 0 | 200 | `65000` (ยาว 1) | IGP |

**ไล่ทีละขั้นตามตาราง Step 295:**

1. **WEIGHT** — ทั้ง 3 Path เท่ากันหมด (0) → ไปขั้นถัดไป
2. **LOCAL_PREF** — Path A = 200, Path C = 200 (เท่ากัน, ชนะ Path B ที่ 100) → **Path B ตกรอบ**
   ที่ขั้นนี้ เหลือ A กับ C แข่งกันต่อ
3. **Locally Originated** — ทั้ง A และ C ไม่ใช่ Route ที่ Originate จาก WAN-EDGE-1 เอง (เรียนจาก
   เพื่อนบ้านทั้งคู่) → เท่ากัน ไปขั้นถัดไป
4. **AS_PATH length** — A = 1 AS, C = 1 AS (เท่ากัน เพราะเป็น Prefix เดียวกันที่โผล่มาจากที่เดียวกัน)
   → เท่ากัน ไปขั้นถัดไป
5. **ORIGIN** — เท่ากันทั้งคู่ (IGP) → ไปขั้นถัดไป
6. **MED** — ไม่มีการตั้งค่า (default 0 ทั้งคู่) → เท่ากัน
7. **eBGP vs iBGP** — **Path A มาจาก eBGP โดยตรง ส่วน Path C มาจาก iBGP (ผ่าน CORE-SW1)**
   → **Path A ชนะที่ขั้นนี้!**

**ผลลัพธ์**: WAN-EDGE-1 เลือก **Path A** (eBGP ตรงจาก ISP1) เป็น Best Path — สังเกตว่าแม้ Path B
จะมี AS_PATH สั้นกว่า Default Policy ทั่วไป (ถ้าไม่มีการตั้ง LOCAL_PREF เลย) แต่เพราะ Step 296
ตั้งค่า **LOCAL_PREF สูงกว่าให้ Path ที่มาจาก ISP1** ผลลัพธ์จึงถูกบังคับให้เลือก A/C ตั้งแต่ขั้นที่ 2
ไปแล้ว — นี่คือพลังของ LOCAL_PREF ที่ "แซง" AS_PATH ได้เสมอ ซึ่งเป็นหัวใจของ Step 296

---

## Step 296 — กำหนดเส้นทางขาออก (Outbound) ด้วย WEIGHT และ LOCAL_PREF

เป้าหมาย: ทำให้ **AS 65001 ทั้งหมด (ทุก Router) เลือกออก Internet ผ่าน ISP1 (WAN-EDGE-1) เป็นหลัก**
และใช้ ISP2 (WAN-EDGE-2) เป็น Backup เท่านั้น (Active/Backup Design) — ตรงข้ามกับข้อมูลขาเข้าซึ่ง
เป็นเรื่องของ Step 297

### 296.1 LOCAL_PREF — วิธีที่ถูกต้องสำหรับทั้ง AS

LOCAL_PREF เป็น Attribute ที่ **ส่งต่อผ่าน iBGP ได้** (แต่ไม่ส่งออก eBGP) จึงเหมาะกับการบังคับ
ทิศทางขาออกของ**ทั้ง AS** พร้อมกัน ค่า Default = 100 ยิ่งสูงยิ่งถูกเลือก

**Config บน WAN-EDGE-1 — ตั้ง LOCAL_PREF สูงให้ทุก Route ที่เรียนจาก ISP1:**

```
WAN-EDGE-1(config)# route-map SET-LOCALPREF-ISP1 permit 10
WAN-EDGE-1(config-route-map)# set local-preference 200
WAN-EDGE-1(config-route-map)# exit
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 route-map SET-LOCALPREF-ISP1 in
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 soft-reconfiguration inbound
WAN-EDGE-1(config-router-af)# end
WAN-EDGE-1# clear ip bgp 203.0.113.1 soft in
```

ไม่ต้องตั้งอะไรเพิ่มบน WAN-EDGE-2 (ปล่อย LOCAL_PREF Default = 100 สำหรับ Route จาก ISP2) —
เพราะ LOCAL_PREF ถูกส่งผ่าน iBGP ไปถึง CORE-SW1, CORE-SW2 และ WAN-EDGE-2 เองโดยอัตโนมัติ ทุก
Router ในกลุ่ม iBGP จะเห็น Route ผ่าน ISP1 มี LOCAL_PREF=200 สูงกว่า Route ผ่าน ISP2 (100) และ
เลือก ISP1 เป็น Best Path ตามขั้นที่ 2 ของ Step 295 ทันที — **โดยที่ CORE-SW1/CORE-SW2 ไม่ต้อง
ถูก Config อะไรเพิ่มเลย**

**ตรวจสอบผล:**

```
CORE-SW1# show ip bgp 0.0.0.0
BGP routing table entry for 0.0.0.0/0, version 8
Paths: (2 available, best #1, table default)
  65000
    1.1.1.21 (metric 20) from 1.1.1.21 (1.1.1.21)
      Origin IGP, localpref 200, valid, internal, best
  65002
    1.1.1.22 (metric 30) from 1.1.1.22 (1.1.1.22)
      Origin IGP, localpref 100, valid, internal
```

`best` อยู่ที่ Path ผ่าน `1.1.1.21` (WAN-EDGE-1/ISP1) ยืนยันว่า CORE-SW1 เลือกออก Internet
ผ่าน ISP1 ตามที่ตั้งใจ แม้ CORE-SW1 จะไม่มี Route-map ใดๆ เป็นของตัวเองเลย

### 296.2 WEIGHT — เมื่อต้องการผลแค่ Router เดียว (Local Override)

WEIGHT เป็น Cisco Proprietary Attribute ที่**อยู่เฉพาะใน Router ที่ตั้งค่า ไม่ถูกส่งต่อให้ใคร
เลยแม้แต่ iBGP** เหมาะกับกรณีที่ต้องการ Override การตัดสินใจ **เฉพาะ Router ตัวนั้นตัวเดียว**
โดยไม่กระทบ Router อื่นในองค์กร — ตัวอย่างเช่น สมมติ CORE-SW1 (ซึ่งปกติควรออกทาง ISP1 ตาม
LOCAL_PREF) มีเหตุผลเฉพาะทาง (เช่น มี Application บางตัวที่ต้อง Egress ผ่าน ISP2 เท่านั้นเพราะ
Contract พิเศษ) วิศวกรสามารถ Override เฉพาะที่ CORE-SW1 ได้โดยไม่ต้องรื้อ Design ทั้ง AS:

```
CORE-SW1(config)# ip prefix-list SPECIAL-APP-VIA-ISP2 seq 5 permit 198.51.100.0/24
CORE-SW1(config)# route-map OVERRIDE-WEIGHT-ISP2 permit 10
CORE-SW1(config-route-map)# match ip address prefix-list SPECIAL-APP-VIA-ISP2
CORE-SW1(config-route-map)# set weight 500
CORE-SW1(config-route-map)# exit
CORE-SW1(config)# route-map OVERRIDE-WEIGHT-ISP2 permit 20
CORE-SW1(config-route-map)# exit
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# neighbor 1.1.1.22 route-map OVERRIDE-WEIGHT-ISP2 in
```

เพราะ WEIGHT (ขั้นที่ 1) ถูกเทียบ**ก่อน** LOCAL_PREF (ขั้นที่ 2) เสมอ การตั้ง WEIGHT=500 ให้ Route
เฉพาะที่เรียนจาก WAN-EDGE-2 (1.1.1.22) จะทำให้ **เฉพาะ CORE-SW1** เลือกออกทาง ISP2 สำหรับ Prefix
198.51.100.0/24 (Application พิเศษ) โดย Router อื่นในองค์กรยังคงออกทาง ISP1 ตามปกติจาก LOCAL_PREF
— นี่คือความต่างสำคัญที่ข้อสอบ CCNP ชอบถาม: **"ถ้าอยากคุมทั้งองค์กรใช้ LOCAL_PREF, ถ้าอยากคุมแค่
Router เดียวใช้ WEIGHT"**

---

## Step 297 — กำหนดเส้นทางขาเข้า (Inbound) ด้วย AS-Path Prepending และ MED

Step 296 คุม**ขาออก**ได้ เพราะ LOCAL_PREF/WEIGHT เป็น Attribute ที่ Router ของเราเองมีสิทธิ์เต็มที่
แต่**ขาเข้า** (Traffic จาก Internet วิ่งเข้าหา Enterprise) เป็นการตัดสินใจของ **ISP และ Router อื่น
บน Internet** ซึ่งเราควบคุมโดยตรงไม่ได้ — ทำได้แค่ **"โน้มน้าว" ผ่าน Attribute ที่ eBGP ส่งออกไป**
ให้ฝั่งตรงข้ามเลือกเส้นทางที่เราต้องการ

### 297.1 AS-Path Prepending — เทคนิคหลักที่ใช้ได้เสมอ

แนวคิด: **AS_PATH ยิ่งยาวยิ่งถูกเลือกน้อยลง** (ขั้นที่ 4 ของ Step 295) ดังนั้นถ้าต้องการให้ฝั่ง
ISP2 (และ Internet ทั้งหมด) **มองเห็นเส้นทางผ่าน ISP2 ว่ายาวกว่าเส้นทางผ่าน ISP1** เราก็แค่
"พ่วง (Prepend)" AS Number ของตัวเอง (65001) ซ้ำๆ เข้าไปใน AS_PATH ก่อน Advertise Prefix
ของ Enterprise ออกทาง ISP2

**สมมติ Enterprise Advertise Public Block `198.51.100.0/24` ออกทั้งสอง ISP:**

```
WAN-EDGE-2(config)# route-map PREPEND-TO-ISP2 permit 10
WAN-EDGE-2(config-route-map)# set as-path prepend 65001 65001 65001
WAN-EDGE-2(config-route-map)# exit
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# address-family ipv4 unicast
WAN-EDGE-2(config-router-af)# network 198.51.100.0 mask 255.255.255.0
WAN-EDGE-2(config-router-af)# neighbor 203.0.113.5 route-map PREPEND-TO-ISP2 out
WAN-EDGE-2(config-router-af)# end
WAN-EDGE-2# clear ip bgp 203.0.113.5 soft out
```

**ผลลัพธ์ที่ฝั่ง ISP2 (และ Router อื่นบน Internet) จะเห็น:**

```
ตาราง AS_PATH ที่ Internet มองเห็นสำหรับ Prefix 198.51.100.0/24:

ผ่าน ISP1 (WAN-EDGE-1):  AS_PATH = 65000 65001            (ยาว 2)
ผ่าน ISP2 (WAN-EDGE-2):  AS_PATH = 65002 65001 65001 65001 65001  (ยาว 5)
```

Router บน Internet ที่เห็นทั้งสองเส้นทางจะเลือก AS_PATH สั้นกว่าคือผ่าน ISP1 — Traffic ขาเข้าสู่
Enterprise จึงไหลผ่าน WAN-EDGE-1/ISP1 เป็นหลัก ตรงกับที่ตั้งใจให้ ISP2 เป็นแค่ Backup ทั้งขาเข้า
และขาออก

### 297.2 MED (Multi-Exit Discriminator) — ใช้เมื่อมีหลายจุดเชื่อมต่อไปยัง AS เดียวกัน

MED บอก **AS เพื่อนบ้านเดียวกัน** ว่า "ถ้ามีทางเข้าเรามากกว่า 1 จุด ให้เลือกจุดที่ MED ต่ำกว่า" —
ข้อจำกัดสำคัญคือ **MED ถูกเทียบกันก็ต่อเมื่อ Path ทั้งสองมาจาก AS เพื่อนบ้านเดียวกันเท่านั้น**
(ยกเว้นเปิด `bgp always-compare-med` ซึ่งไม่ค่อยแนะนำเพราะทำให้พฤติกรรมไม่แน่นอนข้าม AS)

ในสถานการณ์ปัจจุบันของ Lab (ISP1 = AS 65000, ISP2 = AS 65002 คนละ AS กัน) **MED จะไม่ถูกเทียบ
กันเลยตาม Default** — AS-Path Prepending (297.1) จึงเป็นเครื่องมือหลักที่ใช้ได้จริงในสถานการณ์
Dual-ISP คนละราย ส่วน MED จะมีประโยชน์ก็ต่อเมื่อ **Enterprise เชื่อมต่อ ISP เดียวกันสองจุด**
(เช่น มี Circuit สำรองไปยัง ISP1 อีกเส้นที่ตึกอื่น — ทั้งสอง Session ยัง `remote-as 65000`
เหมือนเดิม) ตัวอย่าง Config เชิงสาธิต (สมมติสถานการณ์ขยาย):

```
! สมมติ WAN-EDGE-1 มี Circuit สำรองไปยัง ISP1 อีกเส้นที่ Serial0/1/1 (ตัวอย่างสาธิต MED เท่านั้น)
WAN-EDGE-1(config)# route-map SET-MED-PRIMARY permit 10
WAN-EDGE-1(config-route-map)# set metric 50
WAN-EDGE-1(config)# route-map SET-MED-BACKUP permit 10
WAN-EDGE-1(config-route-map)# set metric 200
! Circuit หลัก (MED ต่ำกว่า = ต้องการให้ ISP1 ส่ง Traffic เข้าทางนี้)
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 route-map SET-MED-PRIMARY out
! Circuit สำรอง (MED สูงกว่า = ให้ ISP1 เลือกจุดนี้เป็นตัวเลือกรอง)
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.9 route-map SET-MED-BACKUP out
```

> **สรุปเปรียบเทียบ AS-Path Prepend vs MED**: Prepend ใช้ได้กับทุกสถานการณ์ (แม้ ISP ต่างรายกัน)
> เพราะ AS_PATH เป็นขั้นที่ Router ทุกตัวบน Internet เทียบกันเสมอ (ขั้น 4 ในตาราง Step 295) ส่วน
> MED ใช้ได้เฉพาะกรณีมีหลายทางเข้า AS เดียวกัน (ขั้น 6 ซึ่งมาทีหลัง AS_PATH ด้วย) — ในการออกแบบ
> Lab หลักสูตรนี้ (Dual-ISP คนละราย) **AS-Path Prepend คือเครื่องมือที่ถูกต้อง** ส่วน MED เก็บไว้
> เป็นความรู้สำหรับสถานการณ์ Multi-homed to Same ISP ในอนาคต

---

## Step 298 — BGP Communities

### 298.1 Community คืออะไร

**Community** คือ Attribute แบบ Tag (32-bit, รูปแบบมาตรฐาน `ASN:VALUE`) ที่ติดไปกับ Route เพื่อ
ใช้ทำ **Policy แบบกลุ่ม** โดยไม่ต้องพึ่ง Prefix-list ยาวๆ — Router ปลายทางแค่ Match Community
แล้วกำหนด Policy ได้เลย (เช่น "Community นี้ = ต้อง set local-preference สูง" โดยไม่ต้องรู้ว่า
Prefix มีอะไรบ้าง)

**Community ไม่ถูกส่งต่อโดย Default** ต้องเปิดด้วย `neighbor <ip> send-community` เสมอ (มี
`standard`, `extended`, `both` ให้เลือก)

### 298.2 Well-Known Community

| Community | ค่า Numeric | ความหมาย |
|---|---|---|
| `no-export` | `0xFFFFFF01` (65535:65281) | ห้าม Advertise ออกนอก AS (iBGP/Confederation เท่านั้น — Route Reflector ยังส่งได้ปกติ) |
| `no-advertise` | `0xFFFFFF02` (65535:65282) | ห้าม Advertise ให้ Neighbor ตัวไหนเลยแม้แต่ iBGP |
| `local-AS` (no-export-subconfed) | `0xFFFFFF03` (65535:65283) | ห้ามออกนอก Sub-AS (ใช้กับ Confederation เท่านั้น) |

### 298.3 Custom Community — ตัวอย่างจริงในบริบท Lab นี้

ออกแบบ Community Tag เพื่อระบุว่า Route มาจาก ISP ไหน (ใช้ต่อยอดทำ Policy อัตโนมัติได้ในอนาคต
เช่น Part 31 ตอน Redistribute เข้า OSPF อาจ Filter ตาม Community นี้):

| Community | ความหมาย |
|---|---|
| `65001:100` | Route ที่เรียนรู้จาก ISP1 |
| `65001:200` | Route ที่เรียนรู้จาก ISP2 |

**Config ติด Tag ตอนรับเข้า (Inbound) บน WAN-EDGE-1:**

```
WAN-EDGE-1(config)# route-map TAG-FROM-ISP1 permit 10
WAN-EDGE-1(config-route-map)# set community 65001:100 additive
WAN-EDGE-1(config-route-map)# set local-preference 200
WAN-EDGE-1(config-route-map)# exit
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 route-map TAG-FROM-ISP1 in
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 send-community both
```

**Config ฝั่ง WAN-EDGE-2 (Tag จาก ISP2):**

```
WAN-EDGE-2(config)# route-map TAG-FROM-ISP2 permit 10
WAN-EDGE-2(config-route-map)# set community 65001:200 additive
WAN-EDGE-2(config-route-map)# exit
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# address-family ipv4 unicast
WAN-EDGE-2(config-router-af)# neighbor 203.0.113.5 route-map TAG-FROM-ISP2 in
WAN-EDGE-2(config-router-af)# neighbor 1.1.1.21 send-community both
```

> **ทำไมต้องมี `send-community` ที่ Neighbor ฝั่ง iBGP ด้วย**: Community ที่ติดมาจาก ISP1
> (Tag บน WAN-EDGE-1) ต้องถูกส่งต่อผ่าน iBGP ไปให้ CORE-SW1/CORE-SW2/WAN-EDGE-2 เห็นด้วย ถ้า
> ลืมใส่ `send-community` บน iBGP Neighbor, Community Attribute จะถูก "ถอดทิ้ง" ก่อน Advertise
> เข้า iBGP ทันที (Default behavior ของ BGP คือไม่ส่ง Community ไปไหนเลยจนกว่าจะสั่ง)

**ตรวจสอบผลบน CORE-SW1 (ปลายทางที่ไม่ได้ตั้ง Route-map อะไรเองเลย):**

```
CORE-SW1# show ip bgp 0.0.0.0 community
BGP routing table entry for 0.0.0.0/0
  65000
    1.1.1.21 from 1.1.1.21
      Origin IGP, localpref 200, valid, internal, best
      Community: 65001:100
```

Community `65001:100` เดินทางมาถึง CORE-SW1 ได้ครบถ้วนผ่าน iBGP — พิสูจน์ว่า Design ทำงานถูกต้อง

---

## Step 299 — Verification & Troubleshooting BGP Path Selection

### 299.1 คำสั่งตรวจสอบ Route ที่ Advertise / Received

```
! ดูว่า WAN-EDGE-1 Advertise Route อะไรออกไปให้ ISP1 บ้าง (ต้องเปิด soft-reconfiguration
! inbound หรือใช้ Route-Refresh Capability ก่อน ถึงจะเก็บสำเนา Route ดิบไว้แสดงได้)
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 advertised-routes

! ดูว่า WAN-EDGE-1 ได้รับ Route ดิบอะไรจาก ISP1 บ้าง (ก่อนถูก Route-map กรอง/แก้ Attribute)
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 received-routes
```

> ถ้าไม่เห็นผลลัพธ์จาก `received-routes` ให้ตรวจว่าเปิด
> `neighbor 203.0.113.1 soft-reconfiguration inbound` แล้วหรือยัง (คำสั่งนี้บอกให้ Router
> เก็บสำเนา Route ดิบไว้ในหน่วยความจำแยกต่างหาก มิเช่นนั้น IOS จะเก็บแค่ Route หลัง Policy เท่านั้น)

### 299.2 ค้นหา Route ด้วย AS_PATH Regular Expression

```
! หา Route ทั้งหมดที่ผ่าน ISP1 (AS 65000) โดยตรง (Origin AS = 65000, ไม่มี AS อื่นต่อท้าย)
WAN-EDGE-1# show ip bgp regexp ^65000$

! หา Route ทั้งหมดที่เคยผ่าน AS 65002 ที่ไหนก็ได้ใน AS_PATH
WAN-EDGE-1# show ip bgp regexp _65002_

! หา Route ที่ Originate จาก AS ตัวเอง (Enterprise Advertise เอง)
WAN-EDGE-1# show ip bgp regexp ^$
```

### 299.3 ตรวจสอบ Community

```
CORE-SW1# show bgp ipv4 unicast community 65001:100
CORE-SW1# show bgp ipv4 unicast community no-export
```

### 299.4 วินิจฉัยปัญหา Asymmetric Routing จาก Path Selection ผิด

**อาการ**: ผู้ใช้บ่นว่า Web Application ช้า/หลุดบ่อย ทั้งที่ Bandwidth ทั้งสอง ISP ปกติ

**ขั้นตอนวินิจฉัย:**

```
Step 1: ตรวจทิศทางขาออกจริง (จาก Enterprise ไป Internet)
CORE-SW1# traceroute 8.8.8.8
  1  10.10.254.1  (WAN-EDGE-1 ทาง ISP1)   <- ขาออกผ่าน ISP1

Step 2: ตรวจทิศทางขาเข้าจริง (จาก Internet กลับมา Enterprise) — ต้องอาศัย
        Looking Glass ของ ISP หรือขอ ISP ช่วย traceroute กลับมา
        ผลลัพธ์: Traffic ขาเข้ากลับผ่าน ISP2 แทน!

Step 3: สรุปว่าเกิด Asymmetric Routing — ขาออกทาง ISP1, ขาเข้าทาง ISP2
        (ไม่ผิดเสมอไป แต่ถ้า Stateful Firewall อยู่คนละจุดกับ Path จะทำให้ Session ถูก Drop)

Step 4: หาสาเหตุ — ตรวจว่า AS-Path Prepend (Step 297) ยังทำงานอยู่หรือไม่
CORE-SW1# show ip bgp 198.51.100.0
  (ตรวจว่า Prefix ของเราเอง Advertise ออกไปด้วย AS_PATH ที่ถูกต้องทั้งสองฝั่งหรือไม่)

WAN-EDGE-2# show ip bgp neighbors 203.0.113.5 advertised-routes
  (ตรวจว่า Route-map PREPEND-TO-ISP2 ยังติดอยู่ที่ Neighbor ฝั่งนี้จริง — สาเหตุที่พบบ่อยที่สุด
   คือมีคน Clear หรือแก้ Route-map แล้วลืม apply "out" ใหม่ หรือ route-map ผิด sequence)

Step 5: แก้ไข — ยืนยัน route-map apply ถูกจุด (`neighbor ... route-map ... out` ไม่ใช่ `in`)
        แล้ว clear BGP session แบบ soft เพื่อให้ Effect มีผลทันทีโดยไม่ตัด Session
WAN-EDGE-2# clear ip bgp 203.0.113.5 soft out
```

> **บทเรียนสำคัญ**: Asymmetric Routing ระหว่างสอง ISP **ไม่ใช่บั๊ก** ของ BGP เอง แต่เป็นผลลัพธ์
> ปกติของการที่ Path Selection ขาออก (เราควบคุมด้วย LOCAL_PREF) กับขาเข้า (ISP/Internet ควบคุม
> ตาม AS_PATH ที่เราโน้มน้าว) เป็นคนละกลไกกัน ถ้า Design ตั้งใจให้ Symmetric ต้องทำทั้งสองฝั่งให้
> สอดคล้องกัน (LOCAL_PREF สูงให้ ISP1 + AS-Path Prepend ทำให้ ISP2 อ่อนแอกว่า พร้อมกันเสมอ)
> เหมือนที่ทำใน Step 296-297 และ Step 300 ต่อไปนี้

---

## Step 300 — Lab เต็มรูปแบบ: iBGP Mesh + Path Control ที่พิสูจน์ผลจริงทั้งสองทิศทาง

### 300.1 Topology สรุป

```
                     ISP1 (AS 65000)          ISP2 (AS 65002)
                    203.0.113.1                203.0.113.5
                         │ eBGP                     │ eBGP
                         │                           │
                 ┌───────┴────────┐         ┌────────┴───────┐
                 │  WAN-EDGE-1     │=========│  WAN-EDGE-2    │  iBGP (Cross-link)
                 │  Lo0: 1.1.1.21  │  iBGP   │  Lo0: 1.1.1.22 │
                 └───────┬────────┘  Mesh    └────────┬───────┘
                    iBGP │      \            /  │ iBGP
                         │        \        /     │
                         │          \    /       │
                         │            \/         │
                         │            /\         │
                         │          /    \       │
                         │        /        \     │
                    ┌────┴────┐            ┌────┴────┐
                    │CORE-SW1 │===iBGP=====│CORE-SW2 │
                    │Lo0:1.1.1.1│           │Lo0:1.1.1.2│
                    └─────────┘            └─────────┘

Full-Mesh iBGP AS 65001: 4 โหนด = 6 Session (WE1-WE2, WE1-C1, WE1-C2, WE2-C1, WE2-C2, C1-C2)
Design เป้าหมาย: Outbound Primary=ISP1 (LOCAL_PREF) / Inbound Primary=ISP1 (AS-Path Prepend ที่ ISP2)
```

### 300.2 Config เต็ม — WAN-EDGE-1

```
hostname WAN-EDGE-1
!
ip prefix-list ENTERPRISE-BLOCK seq 5 permit 198.51.100.0/24
!
route-map SET-LOCALPREF-ISP1 permit 10
 set local-preference 200
 set community 65001:100 additive
!
route-map ADVERTISE-OUT permit 10
 match ip address prefix-list ENTERPRISE-BLOCK
!
router bgp 65001
 bgp router-id 1.1.1.21
 bgp log-neighbor-changes
 neighbor 203.0.113.1 remote-as 65000
 neighbor 203.0.113.1 description eBGP to ISP1
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 description iBGP to WAN-EDGE-2
 neighbor 1.1.1.22 update-source Loopback0
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 description iBGP to CORE-SW1
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 description iBGP to CORE-SW2
 neighbor 1.1.1.2 update-source Loopback0
 !
 address-family ipv4 unicast
  network 198.51.100.0 mask 255.255.255.0
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 route-map SET-LOCALPREF-ISP1 in
  neighbor 203.0.113.1 route-map ADVERTISE-OUT out
  neighbor 203.0.113.1 soft-reconfiguration inbound
  neighbor 1.1.1.22 activate
  neighbor 1.1.1.22 next-hop-self
  neighbor 1.1.1.22 send-community both
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 next-hop-self
  neighbor 1.1.1.1 send-community both
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 next-hop-self
  neighbor 1.1.1.2 send-community both
 exit-address-family
```

### 300.3 Config เต็ม — WAN-EDGE-2

```
hostname WAN-EDGE-2
!
ip prefix-list ENTERPRISE-BLOCK seq 5 permit 198.51.100.0/24
!
route-map TAG-FROM-ISP2 permit 10
 set community 65001:200 additive
!
route-map PREPEND-TO-ISP2 permit 10
 match ip address prefix-list ENTERPRISE-BLOCK
 set as-path prepend 65001 65001 65001
!
router bgp 65001
 bgp router-id 1.1.1.22
 bgp log-neighbor-changes
 neighbor 203.0.113.5 remote-as 65002
 neighbor 203.0.113.5 description eBGP to ISP2
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 description iBGP to WAN-EDGE-1
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.1 remote-as 65001
 neighbor 1.1.1.1 description iBGP to CORE-SW1
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 description iBGP to CORE-SW2
 neighbor 1.1.1.2 update-source Loopback0
 !
 address-family ipv4 unicast
  network 198.51.100.0 mask 255.255.255.0
  neighbor 203.0.113.5 activate
  neighbor 203.0.113.5 route-map TAG-FROM-ISP2 in
  neighbor 203.0.113.5 route-map PREPEND-TO-ISP2 out
  neighbor 203.0.113.5 soft-reconfiguration inbound
  neighbor 1.1.1.21 activate
  neighbor 1.1.1.21 next-hop-self
  neighbor 1.1.1.21 send-community both
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 next-hop-self
  neighbor 1.1.1.1 send-community both
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 next-hop-self
  neighbor 1.1.1.2 send-community both
 exit-address-family
```

### 300.4 Config เต็ม — CORE-SW1 (และ CORE-SW2 กลับด้าน)

```
hostname CORE-SW1
!
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 description iBGP to WAN-EDGE-1
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 description iBGP to WAN-EDGE-2
 neighbor 1.1.1.22 update-source Loopback0
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 description iBGP to CORE-SW2
 neighbor 1.1.1.2 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 1.1.1.21 activate
  neighbor 1.1.1.22 activate
  neighbor 1.1.1.2 activate
 exit-address-family
```

`CORE-SW2` เหมือนกันทุกประการ เพียงสลับ `bgp router-id 1.1.1.2` และเปลี่ยน Neighbor
`1.1.1.2` เป็น `1.1.1.1` (Peer หา CORE-SW1) — CORE-SW1/CORE-SW2 **ไม่ต้องมี Route-map ใดๆ**
เพราะ Policy ทั้งหมด (LOCAL_PREF, Community, AS-Path Prepend) ถูกกำหนดไว้ที่ WAN-EDGE ทั้งสองตัว
แล้วอาศัย iBGP กระจาย Attribute เข้ามาให้อัตโนมัติ — นี่คือหลักการออกแบบที่ถูกต้อง: **ตั้ง Policy
ที่ขอบ AS (Edge) ให้น้อยจุดที่สุด แล้วปล่อยให้ iBGP ทำหน้าที่กระจายผลลัพธ์**

### 300.5 Verification ครบวงจร — พิสูจน์ทั้งสองทิศทาง

**(1) iBGP Session ครบ 6 Session ทุกตัว Established:**

```
CORE-SW1# show ip bgp summary
BGP router identifier 1.1.1.1, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.2         4  65001      40      39         9    0    0 00:12:40        2
1.1.1.21        4  65001      52      50         9    0    0 00:12:55        2
1.1.1.22        4  65001      50      49         9    0    0 00:12:50        2
```

**(2) ขาออก (Outbound) — CORE-SW1 เลือกออกทาง ISP1:**

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.21, from 1.1.1.21, 00:12:55 ago
      Route metric is 0, traffic share count is 1
      AS Hops 1
```

Next-hop `1.1.1.21` = WAN-EDGE-1 → ยืนยันขาออกผ่าน ISP1 ตาม LOCAL_PREF=200 ที่ตั้งไว้

**(3) ขาเข้า (Inbound) — ตรวจ AS_PATH ที่ Advertise ออกทั้งสองฝั่ง:**

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 advertised-routes | include 198.51.100.0
*> 198.51.100.0/24  0.0.0.0                  0         32768 i        <- AS_PATH ว่าง (ยาว 0 จากมุมมอง ISP1: 65000 65001)

WAN-EDGE-2# show ip bgp neighbors 203.0.113.5 advertised-routes | include 198.51.100.0
*> 198.51.100.0/24  0.0.0.0                  0         32768 65001 65001 65001 i   <- Prepend 3 ครั้ง (จากมุมมอง ISP2: 65002 65001 65001 65001 65001)
```

ฝั่ง ISP1 เห็น AS_PATH สั้นกว่าฝั่ง ISP2 อย่างชัดเจน (2 เทียบ 5) → Internet ทั่วโลกจะเลือกส่ง
Traffic เข้าหา Enterprise ผ่าน ISP1 เป็นหลัก ตรงกับทิศทางขาออกพอดี → **Symmetric Routing** สำเร็จ

**(4) ตรวจ Community เดินทางถูกต้องทั่วทั้ง AS:**

```
CORE-SW2# show bgp ipv4 unicast community 65001:100
BGP table version is 9, local router ID is 1.1.1.2
   Network          Next Hop            Metric LocPrf Weight Path
*>i0.0.0.0/0        1.1.1.21                 0    200      0 65000 i
```

**(5) ทดสอบ Failover — ปิด eBGP Session ของ ISP1 ชั่วคราวเพื่อยืนยัน Backup ทำงาน:**

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 shutdown

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "bgp 65001", distance 200, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.22, from 1.1.1.22, 00:00:03 ago     <- สลับไปใช้ WAN-EDGE-2/ISP2 อัตโนมัติทันที
```

Lab นี้พิสูจน์ครบทั้ง 3 เรื่อง: **iBGP กระจาย Route ได้ทั่วทั้ง AS (Full-Mesh), Path Selection
ควบคุมได้ทั้งขาออก (LOCAL_PREF) และขาเข้า (AS-Path Prepend), และ HA Failover ทำงานอัตโนมัติ**
เมื่อ ISP1 ล่ม — ครบตามเป้าหมายการออกแบบ Dual-ISP ระดับ Enterprise

---

## แบบฝึกหัดทวนความเข้าใจ Part 30

1. เหตุใด iBGP จึงต้องมี Full-Mesh (หรือ Route Reflector) ในขณะที่ eBGP ไม่จำเป็น? อธิบายด้วย
   กฎ Split-Horizon ของ iBGP
2. `next-hop-self` แก้ปัญหาอะไร และต้องใส่คำสั่งนี้ที่ Router ตัวไหนของ Design (WAN-EDGE หรือ
   CORE-SW)?
3. ถ้าต้องการให้**ทั้งองค์กร**เลือกออก Internet ผ่าน ISP หนึ่งเป็นหลัก ควรใช้ WEIGHT หรือ
   LOCAL_PREF? เพราะเหตุใด
4. เรียงลำดับ BGP Best Path Selection 5 ขั้นแรกให้ถูกต้อง
5. AS-Path Prepending มีผลต่อ Path Selection ขั้นไหน และทำไมจึงเป็นเครื่องมือหลักในการโน้มน้าว
   Traffic ขาเข้า (Inbound) จาก ISP ต่างรายกัน?

**เฉลย:**

1. เพราะกฎ iBGP กำหนดว่า Route ที่เรียนรู้ผ่าน iBGP ห้ามถูก Advertise ต่อไปยัง iBGP Peer ตัวอื่น
   (ป้องกัน Loop เนื่องจาก AS_PATH ไม่ถูกเพิ่มภายใน AS เดียวกัน) ดังนั้นทุก Router ที่ต้องการ Route
   นั้นต้อง Peer โดยตรงกับผู้ที่ Originate (หรือ Peer กับ Route Reflector ที่ได้รับการยกเว้นกฎนี้)
2. แก้ปัญหา NEXT_HOP ที่ไม่เปลี่ยนแปลงเมื่อ Advertise ผ่าน iBGP (ค่า Default ของ iBGP ไม่แก้ไข
   NEXT_HOP) ทำให้ Router ปลายทาง (เช่น CORE-SW1) ได้ NEXT_HOP เป็น IP วงนอกที่ไม่มี Route ไปถึง
   ต้องใส่ที่ **WAN-EDGE-1/WAN-EDGE-2** (Router ที่เป็นจุดเชื่อมระหว่าง eBGP กับ iBGP) บน
   Neighbor Statement ฝั่ง iBGP เท่านั้น ไม่ต้องใส่ที่ CORE-SW
3. ใช้ **LOCAL_PREF** เพราะเป็น Attribute ที่ถูกส่งต่อผ่าน iBGP ไปถึงทุก Router ในกลุ่มเดียวกัน
   ทำให้ Policy มีผลทั้งองค์กรจากจุดตั้งค่าเดียว ส่วน WEIGHT อยู่เฉพาะ Local Router ตัวเดียว ไม่
   ถูกส่งต่อให้ใครเลย เหมาะกับการ Override เฉพาะจุดเท่านั้น
4. WEIGHT → LOCAL_PREF → Locally Originated → AS_PATH (สั้นกว่า) → ORIGIN (IGP < EGP < Incomplete)
5. มีผลต่อ**ขั้นที่ 4 (ความยาว AS_PATH)** — เป็นเครื่องมือหลักเพราะทุก Router บน Internet เทียบ
   AS_PATH กันเป็นมาตรฐานเสมอ (ไม่ว่าจะเป็น AS เพื่อนบ้านเดียวกันหรือไม่ก็ตาม) ต่างจาก MED ที่ถูก
   เทียบเฉพาะ Path จาก AS เพื่อนบ้านเดียวกันเท่านั้น จึงใช้ไม่ได้ในสถานการณ์ Dual-ISP คนละราย

---

## สรุป Part 30

Part นี้เติมเต็มส่วนที่ Part 29 ยังขาดไป: การกระจาย Route ที่เรียนรู้จาก eBGP ให้ทั่วทั้ง AS 65001
ด้วย **iBGP** — เราเรียนรู้ **Full-Mesh Requirement** และสูตร `n(n-1)/2` ที่อธิบายว่าทำไมมันไม่
Scale, ความจำเป็นของ **`next-hop-self`** ที่ WAN-EDGE ทุกตัว, แนวคิด **Route Reflector** ที่แก้
ปัญหา Scaling (จะกลับมาใช้จริงใน Part 33-35), เปรียบเทียบกับ **BGP Confederation**, เจาะลึก
**Best Path Selection Algorithm เต็มรูปแบบ 12 ขั้น** พร้อมตัวอย่างตัวเลข, ควบคุมเส้นทาง**ขาออก**
ด้วย **WEIGHT/LOCAL_PREF** และ**ขาเข้า**ด้วย **AS-Path Prepending/MED**, ทำความรู้จัก **BGP
Community** สำหรับ Route Tagging, และปิดท้ายด้วย **Lab เต็มรูปแบบ** ที่พิสูจน์ว่า Enterprise
Dual-ISP Design เลือกเส้นทาง Symmetric ได้ทั้งสองทิศทางจริง พร้อม Failover อัตโนมัติเมื่อ ISP1 ล่ม ✅

**สิ่งที่ Lab หลักสูตรนี้ยังไม่มี**: CORE-SW1/CORE-SW2 มี Route BGP อยู่ใน BGP Table แล้ว แต่
**ยังไม่ถูกส่งต่อเข้า OSPF** ให้ DIST-SW และ Access Layer เห็น Default Route เพื่อออก Internet
ได้จริง — นี่คืองานของ **Route Redistribution** ซึ่งเป็นหัวข้อที่มีกับดักเรื่อง Routing Loop และ
ต้องทำอย่างระมัดระวังด้วย Route-map/Distribute-list เสมอ

**ไปต่อ:** [Part 31 — Route Redistribution →](part-031-route-redistribution.md)
