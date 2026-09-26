# Part 26 — Advanced OSPF for CCNP ENCOR
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 251–260 จาก 1000**

> ต่อจาก [Part 25 — CCNA Capstone Lab](part-025-ccna-capstone-lab.md) ที่ปิดท้ายระดับ **CCNA
> 200-301** อย่างสมบูรณ์ Part นี้คือ**จุดเริ่มต้นของระดับ CCNP Enterprise Core (ENCOR 350-401)**
> อย่างเป็นทางการ — เนื้อหาที่นี่คือ **CCNP ENCOR-level advanced OSPF ที่ต่อยอดจาก Part 11-12**
> โดยตรง (**ไม่ทวนซ้ำ**: Multi-Area Design พื้นฐาน, ตาราง LSA Type 1-7 ภาพรวม, ABR/ASBR, Area
> Type Stub/NSSA เบื้องต้น, Virtual Link, และ MD5 Authentication ที่ Part 11-12 สอนไปแล้วทั้งหมด)
> Part นี้จะเจาะลึกในมุมที่ข้อสอบ ENCOR และงานจริงระดับ Enterprise ต้องใช้: **Route Filtering
> ระดับ LSA**, **Stub Router สำหรับ Maintenance Mode**, **NSSA แบบเจาะลึกถึง P-bit/Forwarding
> Address**, **Fast Convergence ด้วย BFD + SPF Throttling**, **Network Type ขั้นสูงสำหรับ
> Unnumbered/NBMA**, **Graceful Restart/NSF สำหรับ High Availability Platform**, **Reference
> Bandwidth Tuning ระดับ 10G/40G/100G**, **LSA Throttling/Flood Reduction**, และปิดท้ายด้วย
> Advanced Troubleshooting Toolset ที่ CCNP ต้องใช้งานจริง

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 251 | OSPF Route Filtering — `distribute-list` vs `area filter-list` vs `distribute-list route-map` |
| 252 | OSPF Stub Router (`max-metric router-lsa`) — Graceful Shutdown/Maintenance Mode |
| 253 | LSA Type 7 (NSSA) เจาะลึก — N1/N2, P-bit, Forwarding Address, แปลง Area 2 เป็น NSSA |
| 254 | OSPF Fast Convergence — BFD, SPF Throttling Timers, Incremental SPF (iSPF) |
| 255 | OSPF Unnumbered/Loopback-sourced Link และ Network Type ขั้นสูง (NBMA Worked Example) |
| 256 | OSPF Graceful Restart / NSF สำหรับ High Availability Platform |
| 257 | `auto-cost reference-bandwidth` สำหรับ 10G/40G/100G — ทบทวนปัญหา Cost Collision ที่ CORE |
| 258 | LSA Throttling, Flood Reduction, และ Demand Circuit |
| 259 | Advanced Verification/Troubleshooting — Statistics, Traffic, Debug SPF, วินิจฉัย SPF Flapping |
| 260 | Lab เต็มรูปแบบ: BFD + Reference-Bandwidth + Area Filter-list + Stub-Router Maintenance |

---

## Step 251 — OSPF Route Filtering: `distribute-list` vs `area filter-list` vs `route-map`

### 251.1 กับดักสำคัญที่สุดของหัวข้อนี้ — Link-State ≠ Distance-Vector

ใน EIGRP/RIP (Distance-Vector) คำสั่ง `distribute-list` ทำงานตรงไปตรงมา: บล็อก Route ไม่ให้
**Advertise ออกไปหา Neighbor เลย** เพราะ Distance-Vector ส่ง "Routing Table" ให้กันโดยตรง
แต่ **OSPF เป็น Link-State** — Router ไม่ได้ส่ง Routing Table ให้กัน แต่ Flood **LSA** ที่บรรจุ
ข้อมูล Topology ดิบ แล้วให้ทุกตัวรัน SPF เอง **`distribute-list` ใน OSPF จึงมีข้อจำกัดสำคัญที่
เป็นข้อสอบ CCNP ออกบ่อยที่สุดของหัวข้อนี้**:

> **กฎเหล็ก**: `distribute-list <acl> in` บน OSPF **กรองเฉพาะ RIB (Routing Table) ของ Router
> ตัวนั้นเท่านั้น** — **ไม่ได้หยุด LSA ไม่ให้ Flood ต่อไปยัง Neighbor ตัวถัดไปเลย** Neighbor
> ที่อยู่ถัดไปยังคงได้รับ LSA เดิมครบและยังเห็น Route นั้นในการคำนวณ SPF ของตัวเองตามปกติ

### 251.2 ตารางเปรียบเทียบ 3 เครื่องมือ Route Filtering ของ OSPF

| เครื่องมือ | Config ที่ไหน | กรองอะไรจริงๆ | หยุด LSA Flooding หรือไม่ | ใช้ ACL ได้ | ใช้ Prefix-list ได้ | ใช้ Route-map ได้ |
|---|---|---|---|---|---|---|
| `distribute-list <acl\|prefix> in [interface]` | Router ใดก็ได้ | เฉพาะการติดตั้งเข้า RIB ของตัวเองเท่านั้น | ❌ ไม่หยุด (LSDB/LSA ยังครบ) | ✅ | ✅ (`distribute-list prefix`) | ✅ (`distribute-list route-map`) |
| `distribute-list <acl> out <protocol/process>` | **ASBR เท่านั้น** (ใช้ตอน Redistribute เข้า OSPF) | Route ที่กำลังจะถูก Redistribute เข้าเป็น Type 5 LSA | ✅ หยุดจริง (ไม่สร้าง Type 5 LSA เลยสำหรับ Route ที่โดนกรอง) | ✅ | ✅ | ✅ |
| **`area <id> filter-list prefix <name> in \| out`** | **ABR เท่านั้น** | **Type 3 (Summary) LSA ที่ข้าม Area** | ✅ หยุดจริง (ไม่ Originate Type 3 LSA เข้า/ออก Area นั้นเลย) | ❌ (Prefix-list เท่านั้น) | ✅ | ❌ |

> **สรุปเพื่อสอบ**: ถ้าโจทย์ถามว่า "ต้องการหยุด Subnet ไม่ให้ปรากฏใน Routing Table ของ Area อื่น
> เลย (ไม่ใช่แค่ซ่อนจาก RIB ตัวเดียว)" → คำตอบคือ **`area filter-list`** บน ABR เท่านั้น
> เพราะเป็นเครื่องมือเดียวที่กรองที่ต้นตอ (การสร้าง Type 3 LSA) ไม่ใช่กรองปลายทาง (การติดตั้ง RIB)

### 251.3 Worked Example #1 — `distribute-list in` (กรองแค่ RIB ตัวเอง ไม่กระทบ LSA)

สมมติ DIST-SW2 ไม่ต้องการเห็น 10.10.30.0/24 (VLAN30 จาก Area 2) ใน Routing Table ของตัวเอง
(เช่น Policy บังคับห้าม Traffic จาก DIST-SW2 ไปหา VLAN30 โดยตรง):

```
DIST-SW2(config)# ip access-list standard BLOCK-VLAN30
DIST-SW2(config-std-nacl)# deny   10.10.30.0 0.0.0.255
DIST-SW2(config-std-nacl)# permit any
DIST-SW2(config-std-nacl)# exit

DIST-SW2(config)# router ospf 1
DIST-SW2(config-router)# distribute-list BLOCK-VLAN30 in
DIST-SW2(config-router)# end
```

**ผลลัพธ์**:

```
DIST-SW2# show ip route ospf | include 10.10.30
                                                      <- หายไปจาก RIB ของ DIST-SW2 เท่านั้น

! แต่ LSDB ของ DIST-SW2 ยังมี Type 3 LSA ของ 10.10.30.0/24 อยู่ครบ ไม่ถูกกรองเลย
DIST-SW2# show ip ospf database summary | include 10.10.30
10.10.30.0      1.1.1.1         120         0x80000003

! และ DIST-SW1 (Neighbor ข้าง DIST-SW2 ใน Area 1 เดียวกัน) ยังเห็น Route นี้ปกติทุกประการ
DIST-SW1# show ip route ospf | include 10.10.30
O IA    10.10.30.0/24 [110/12] via 10.255.10.1, TenGigabitEthernet1/1/1
```

นี่คือหลักฐานชัดเจนว่า `distribute-list in` **ไม่ใช่เครื่องมือ Suppress LSA** — เป็นแค่ Local
RIB Filter ที่ใช้เฉพาะกรณีต้องการ "ปิดตา" Router ตัวเดียวจาก Route บางเส้น โดยไม่กระทบ Router
อื่นในระบบเลย

### 251.4 Worked Example #2 — `area filter-list` (กรองที่ต้นตอ Type 3 LSA จริง)

สมมติ Policy บังคับว่า **VLAN40 (10.10.40.0/24, Area 2)** ห้ามมีใครใน Area 1 มองเห็นเลย
(Complete Isolation ระดับ Area) — ต้องทำที่ **ABR ของ Area 1 คือ CORE-SW1** ด้วย Prefix-list:

```
CORE-SW1(config)# ip prefix-list BLOCK-VLAN40 seq 5 deny 10.10.40.0/24
CORE-SW1(config)# ip prefix-list BLOCK-VLAN40 seq 10 permit 0.0.0.0/0 le 32

CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# area 1 filter-list prefix BLOCK-VLAN40 in
CORE-SW1(config-router)# end
```

`in` หมายถึง **ทิศทางที่ Route กำลังจะเข้ามาใน Area 1** (จาก Area 0 → Area 1) — ตรงข้ามกับ
`out` ที่กรอง Route ที่กำลังจะออกจาก Area 1 ไปยัง Area อื่น

**ผลลัพธ์ — หายไปทั้ง LSDB ไม่ใช่แค่ RIB**:

```
DIST-SW1# show ip route ospf | include 10.10.40
                                                      <- ไม่มีเลย (เหมือน Example #1)

DIST-SW1# show ip ospf database summary | include 10.10.40
                                                      <- ไม่มีเลยเช่นกัน — ต่างจาก distribute-list
                                                         ตรงนี้ชัดเจน: Type 3 LSA ไม่ถูกสร้างเข้า
                                                         Area 1 เลยตั้งแต่ต้น ไม่ใช่แค่ถูกซ่อนที่ RIB
```

> **ข้อจำกัดที่ต้องรู้**: `area filter-list` กรองได้เฉพาะ **Type 3 (Inter-area)** เท่านั้น
> ไม่สามารถใช้กรอง Type 1/2 (Intra-area ภายใน Area เดียวกัน) ได้ — ถ้าต้องการกรอง Route ภายใน
> Area เดียวกันเอง ต้องใช้ `distribute-list` (ซึ่งกรองแค่ RIB ตามข้อจำกัดที่อธิบายไปแล้ว) เพราะ
> ไม่มีเครื่องมือใดที่กรอง Type 1/2 LSA ได้โดยตรง (Type 1/2 เป็นรายละเอียด Topology ที่ต้องครบ
> ทุก Router ภายใน Area เดียวกันเสมอ)

---

## Step 252 — OSPF Stub Router: `max-metric router-lsa`

### 252.1 ปัญหาที่ Stub Router แก้ — Maintenance โดยไม่ให้ Traffic สะดุด

เวลาต้อง Reload/Maintenance Router ตัวหนึ่งที่เป็น Transit Path (มี Traffic วิ่งผ่าน ไม่ใช่แค่
ปลายทาง) การ `shutdown` Interface หรือ `reload` ตรงๆ ทำให้ Adjacency ขาดทันที และ Router อื่น
ต้องรอ **Dead Timer หมดอายุ** (Default 40 วินาทีบน Broadcast, หรือจนกว่า Physical Link Down)
ก่อนจะ Reroute — ระหว่างนั้น Traffic ที่ยังพยายามวิ่งผ่าน Router ที่กำลังจะปิดจะหายไปบางส่วน
(Black-hole ชั่วคราว) **OSPF Stub Router** แก้ปัญหานี้ด้วยการ "ประกาศตัวเองว่าเป็นเส้นทางที่
แพงที่สุดเท่าที่เป็นไปได้" **ก่อน** ปิดจริง เพื่อให้ Router อื่น**ย้าย Traffic ออกไปเส้นอื่นก่อน
อย่างนุ่มนวล** (ถ้ามี Path สำรอง) แล้วค่อยปิดจริงทีหลัง

### 252.2 Syntax เต็มของ `max-metric router-lsa`

```
Router(config-router)# max-metric router-lsa [on-startup {<seconds> | wait-for-bgp}]
                                              [on-shutdown <seconds>]
                                              [summary-lsa [<max-metric-value>]]
                                              [external-lsa [<max-metric-value>]]
                                              [include-stub]
```

| Option | ความหมาย |
|---|---|
| (ไม่ใส่ Option) | ตั้ง Router LSA Cost = **65535 (Max)** ทันที ค้างจนกว่าจะสั่ง `no max-metric router-lsa` เอง |
| `on-shutdown <seconds>` | ตั้ง Max-Metric อัตโนมัติเป็นระยะเวลาที่กำหนด **ก่อน** Process จะหยุดจริง (ใช้คู่กับ `reload`) |
| `on-startup {seconds \| wait-for-bgp}` | ตั้ง Max-Metric ตอน **บูตเครื่องขึ้นมาใหม่** ค้างไว้จนกว่า Topology/BGP จะ Converge เต็มที่ก่อนเปิดรับ Transit Traffic จริง |
| `summary-lsa [value]` | ทำให้ **Type 3 (Summary)** ที่ Router นี้สร้าง (ถ้าเป็น ABR) มี Cost สูงสุดด้วย ไม่ใช่แค่ Router LSA |
| `external-lsa [value]` | ทำให้ **Type 5 (External)** ที่ Router นี้ Redistribute (ถ้าเป็น ASBR) มี Cost สูงสุดด้วย |
| `include-stub` | รวม Stub Network (เช่น SVI ที่ไม่มี Neighbor OSPF) เข้าไปโดน Max-Metric ด้วย ปกติ Default จะไม่แตะ Subnet ที่ Router นั้นเป็นเจ้าของโดยตรง |

> **จุดสำคัญของ `on-startup`**: OSPF เอง Converge Adjacency เร็วมาก (วินาทีเดียว) แต่ถ้า Router
> ตัวนั้นต้องพึ่ง **BGP** ด้วย (เช่น WAN Edge ในหลักสูตรนี้ที่จะทำ Redistribute ในอนาคต) BGP มัก
> ใช้เวลานานกว่ามากในการ Converge เต็มรูปแบบ — ถ้า OSPF ประกาศ Cost ปกติทันทีที่ Adjacency ขึ้น
> Router อื่นอาจเริ่มส่ง Traffic ผ่านตัวนี้**ก่อน BGP Table จะพร้อมจริง** ทำให้ Traffic หายไปช่วง
> สั้นๆ (Black-hole ตอน Startup) — `wait-for-bgp` แก้ปัญหานี้โดยค้าง Max-Metric ไว้จนกว่า BGP
> จะส่งสัญญาณ Converge เสร็จก่อน

### 252.3 Worked Example — Maintenance Mode บน CORE-SW1 (แบบ Manual ก่อนใช้ `on-shutdown`)

ขั้นตอน Maintenance ที่ถูกต้องตามลำดับ (Runbook มาตรฐาน CCNP):

```
! STEP 1 — ก่อนแตะ Config ใดๆ ให้ดู Cost ปกติเป็น Baseline
CORE-SW1# show ip ospf database router 1.1.1.1 | include Metrics
       TOS 0 Metrics: 10

! STEP 2 — เปิด Max-Metric (Router LSA ของ CORE-SW1 จะกลายเป็น Cost สูงสุดทันที)
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# max-metric router-lsa
CORE-SW1(config-router)# end

! STEP 3 — Verify ว่า LSA ประกาศ Cost ใหม่แล้วจริง
CORE-SW1# show ip ospf database router 1.1.1.1 | include Metrics
       TOS 0 Metrics: 65535                          <- Router อื่นจะเลี่ยง Path ผ่านตัวนี้ทันที
                                                          (ถ้ามี Path สำรอง)

! STEP 4 — รอให้ Traffic Drain ออกจนแน่ใจ (ตรวจ Interface Counter ว่า Traffic ลดลงจริง)
CORE-SW1# show interfaces Port-channel1 | include rate

! STEP 5 — ทำ Maintenance จริง (Reload/Upgrade/เปลี่ยน Hardware)
CORE-SW1# reload

! STEP 6 — หลัง Maintenance เสร็จและ Verify ว่า Adjacency ขึ้นครบปกติแล้ว ค่อยถอด Max-Metric
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# no max-metric router-lsa
CORE-SW1(config-router)# end
```

> **ข้อจำกัดที่ต้องพูดตรงๆ ในบริบท Lab หลักสูตรนี้**: Topology ปัจจุบันยังเป็น **Single-homed
> DIST↔CORE** (ตามที่ระบุใน `00-ip-address-plan.md`) — CORE-SW1 เป็น ABR ตัวเดียวของ Area 1
> ไม่มี ABR สำรอง ดังนั้นการทำ Max-Metric บน CORE-SW1 **จะไม่มี Path สำรองให้ Traffic ย้ายไปจริง**
> (Area 1 จะขาดจาก Area 0 ทันทีไม่ว่าจะ Max-Metric หรือ Shutdown ตรงๆ ก็ตาม) — ประโยชน์เต็ม
> รูปแบบของ Stub Router จะเห็นผลจริงก็ต่อเมื่อมี **Dual-homed CORE-DIST Redundancy** ซึ่งจะถูก
> เพิ่มเข้ามาใน **Part 33-35 (Advanced Campus Design & HA)** ตามที่ IP Plan ระบุไว้ — Step นี้
> สอนกลไกและลำดับขั้นตอนที่ถูกต้องไว้ก่อน เพื่อให้พร้อมใช้งานทันทีเมื่อ Redundancy มาถึง (จะกลับมา
> สาธิตแบบเห็นผลจริงใน Step 260 และเต็มรูปแบบอีกครั้งใน Part 33-35)

---

## Step 253 — LSA Type 7 (NSSA) เจาะลึก: N1/N2, P-bit, Forwarding Address

### 253.1 ทวนสั้นๆ จาก Part 12 (Step 113.6, 116.3) แล้วไปต่อ

Part 12 สอนแล้วว่า NSSA Area อนุญาตให้มี ASBR ภายในสร้าง **Type 7** แทน Type 5 แล้วให้ ABR
แปลงเป็น Type 5 ก่อนส่งออก — Part นี้จะเจาะลึก **กลไกที่อยู่เบื้องหลังการแปลงนั้นจริงๆ** ซึ่ง
CCNP ต้องเข้าใจระดับ Field ใน LSA ไม่ใช่แค่ Concept ผิวเผิน

### 253.2 N1/N2 — เหมือน E1/E2 ทุกประการ แต่มาจาก NSSA

ทวนสูตรจาก Part 12 Step 118.1 แล้วเทียบ:

| Metric Type | สูตร Cost | ใช้กับ LSA Type |
|---|---|---|
| Type 1 (N1/E1) | External Cost + Internal Cost สะสมถึง ASBR | N1 = Type 7, E1 = Type 5 |
| Type 2 (N2/E2, Default) | External Cost อย่างเดียว | N2 = Type 7, E2 = Type 5 |

หลังผ่าน ABR แปลงเป็น Type 5 แล้ว Router ใน Area อื่นจะเห็น **O E1/O E2** ไม่ใช่ O N1/O N2
อีกต่อไป (O N1/O N2 เห็นได้เฉพาะ**ภายใน NSSA Area นั้นเท่านั้น** ก่อนแปลง)

### 253.3 P-bit (Propagate Bit) — สวิตช์ควบคุมว่าจะแปลงหรือไม่

Type 7 LSA มี Bit พิเศษในฟิลด์ Options เรียกว่า **P-bit (Propagate Bit)** ที่ ASBR ผู้สร้าง
Type 7 เป็นคนกำหนด:

| ค่า P-bit | ผลลัพธ์ที่ ABR |
|---|---|
| **P = 1** (Default) | ABR **แปลง** Type 7 → Type 5 แล้ว Flood ออกไป Area อื่น |
| **P = 0** | ABR **ไม่แปลง** — Route นั้นถูกใช้งานได้เฉพาะภายใน NSSA Area เดียวกันเท่านั้น |

> **ทำไมต้องมี P=0**: กรณีมี ABR มากกว่า 1 ตัวใน NSSA Area เดียวกัน (Redundant ABR) ถ้าทุกตัว
> แปลง Type 7 → Type 5 พร้อมกันหมด จะเกิด Type 5 LSA ซ้ำซ้อนหลายชุดสำหรับ Route เดียวกัน — OSPF
> จึงมีกลไก **Translator Election** เลือก ABR เพียงตัวเดียวทำหน้าที่แปลง (ปกติเลือกตัวที่มี
> Router ID สูงสุด) ส่วนตัวอื่นจะตั้ง P-bit ของ Type 7 ที่ตัวเองสร้าง (ถ้ามี) ไปที่ 0 เพื่อไม่ให้
> เกิดการแปลงซ้ำ ควบคุมด้วยคำสั่ง `area <id> nssa translate type7 {always | suppress-fa}`
> บน ABR ที่ต้องการบังคับบทบาท

### 253.4 Forwarding Address — ทำไม Traffic บางเส้นไม่ผ่าน ABR

Type 5/Type 7 LSA มีฟิลด์ **Forwarding Address (FA)** — ถ้าเป็น **0.0.0.0** หมายถึง "ให้ส่ง
Traffic ไปหา Advertising Router (ตัว ABR/ASBR ที่สร้าง LSA นี้) โดยตรง" แต่ถ้า FA เป็น **IP
Address จริงที่ไม่ใช่ 0.0.0.0** หมายถึง "ให้ส่ง Traffic ไปหา IP นี้โดยตรง" (ข้าม ABR ไปเลย)
— เกิดขึ้นเมื่อ **Next-hop ของ Route ที่ Redistribute นั้นเอง Run OSPF อยู่แล้วในทุก Area
ที่เกี่ยวข้อง** ทำให้ทุก Router มองเห็น Next-hop นั้นโดยตรงผ่าน SPF ปกติอยู่แล้ว ไม่จำเป็นต้อง
อ้อมผ่าน ABR/ASBR ที่ประกาศ LSA — ผลคือ **Path จริงสั้นกว่า Path ตามตัว LSA ที่เห็น**

### 253.5 Worked Example เต็มรูปแบบ — แปลง Area 2 เป็น NSSA พร้อม ASBR จริง

**สถานการณ์**: DIST-SW3 มี Local Resource วงหนึ่ง (จำลองด้วย Static Route ไป Null0 แทน Site
ปลายทางสมมติ 172.16.30.0/24) ที่ต้อง Redistribute เข้า OSPF — Area 2 จะถูกแปลงเป็น **NSSA**
เพื่อลด LSDB (Block Type 5 จาก Area อื่น) แต่ยังให้ DIST-SW3 เป็น ASBR ได้

**ขั้นที่ 1 — แปลง Area 2 เป็น NSSA (ต้องตั้งตรงกันทุกตัวใน Area 2 ไม่งั้น Adjacency Down
เหมือนปัญหา Area Type Mismatch ที่ Part 12 Step 119.4 อธิบายไว้)**:

```
! CORE-SW2 (ABR ของ Area 2)
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# area 2 nssa
CORE-SW2(config-router)# end

! DIST-SW3 และ DIST-SW4 (Internal Router ของ Area 2)
DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# area 2 nssa
DIST-SW3(config-router)# end

DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# area 2 nssa
DIST-SW4(config-router)# end
```

**ขั้นที่ 2 — สร้าง Local Resource สมมติ + Redistribute บน DIST-SW3 (กลายเป็น ASBR)**:

```
DIST-SW3(config)# ip route 172.16.30.0 255.255.255.0 Null0

DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# redistribute static subnets metric 20 metric-type 2
DIST-SW3(config-router)# end
```

**ขั้นที่ 3 — Verify Type 7 LSA ที่ต้นทาง (DIST-SW3)**:

```
DIST-SW3# show ip ospf database nssa-external

                Type-7 AS External Link States (Area 2)

  LS age: 25
  Options: (N/P, No TOS-capability, DC)         <- P-bit = 1 (Propagate) เห็นชัดในบรรทัด Options
  LS Type: AS External Link
  Link State ID: 172.16.30.0
  Advertising Router: 1.1.1.13                  <- DIST-SW3 เป็นผู้สร้าง (ASBR จริง)
  Metric Type: 2 (Larger than any link state path)
  Metric: 20
  Forward Address: 10.255.30.2                  <- IP ของ DIST-SW3 เอง (ไม่ใช่ 0.0.0.0)
```

**ขั้นที่ 4 — Verify การแปลงที่ ABR (CORE-SW2)**:

```
CORE-SW2# show ip ospf database external

                Type-5 AS External Link States

  LS age: 26
  LS Type: AS External Link
  Link State ID: 172.16.30.0
  Advertising Router: 1.1.1.2                   <- เปลี่ยนเป็น CORE-SW2 (ABR/Translator)
  Metric Type: 2
  Metric: 20
  Forward Address: 10.255.30.2                  <- FA ยังคงชี้ไปยัง DIST-SW3 เดิม ไม่เปลี่ยน

CORE-SW2# show ip ospf | section Area 2
    Area 2
        Number of interfaces in this area is 3
        It is a NSSA area
           Perform type-7/type-5 translation, always
```

**ขั้นที่ 5 — Route Type ที่ปลายทางต่างๆ**:

```
! DIST-SW4 — อยู่ใน Area 2 เดียวกับ ASBR โดยตรง เห็นเป็น O N2
DIST-SW4# show ip route | include 172.16.30
O N2    172.16.30.0/24 [110/20] via 10.255.40.1, TenGigabitEthernet1/1/1

! DIST-SW1 — อยู่ Area 1 (คนละ Area) เห็นหลังผ่านการแปลงแล้วเป็น O E2
DIST-SW1# show ip route | include 172.16.30
O E2    172.16.30.0/24 [110/20] via 10.255.10.1, TenGigabitEthernet1/1/1
```

> **ผลลัพธ์จริงจาก Forwarding Address**: ถ้า Segment ระหว่าง DIST-SW3 กับ CORE-SW2
> (10.255.30.0/30) ถูกกำหนดให้ทั้งสองฝั่ง Run OSPF อยู่แล้ว (ซึ่งเป็นจริงในกรณีนี้) DIST-SW1
> เมื่อ Forward Packet ไปยัง 172.16.30.0/24 จะ **Route ไปตาม Forwarding Address (10.255.30.2 =
> DIST-SW3) โดยตรงผ่าน SPF ปกติ ไม่ใช่ต้องยิงไปหา CORE-SW2 (1.1.1.2, Advertising Router) ก่อน**
> แล้วให้ CORE-SW2 ส่งต่อให้ DIST-SW3 อีกที — นี่คือสิ่งที่ Forwarding Address มีไว้ป้องกัน
> Suboptimal Routing แบบอ้อมสองรอบ (Router → ABR → ASBR จริง) ทั้งที่ Router ต้นทางเห็น ASBR
> ตรงๆ ผ่าน SPF ปกติอยู่แล้ว

---

## Step 254 — OSPF Fast Convergence: BFD, SPF Throttling, Incremental SPF (iSPF)

### 254.1 ทำไม Hello/Dead Timer เพียงอย่างเดียวไม่พอสำหรับ Enterprise Core

Default Hello/Dead Timer ของ OSPF บน Point-to-Point คือ 10/40 วินาที (ทวนจาก Part 11) — เร็ว
กว่านี้ได้ด้วยการลด Timer ตรงๆ แต่การลด Timer มากเกินไปเสี่ยง False Positive (CPU สูงชั่วขณะ
ทำให้ Router ตอบ Hello ไม่ทัน) ปัญหาที่ใหญ่กว่านั้นคือ**สื่อบางประเภทไม่ Down ทาง Physical Layer
เมื่อปลายทางเสีย** เช่น Link ผ่าน Metro Ethernet/DWDM/MPLS ที่ CE Router มองเห็น Interface
Up/Up ตลอดแม้ปลายทางจริงจะขาดไปแล้ว — กรณีนี้ Dead Timer 40 วินาทีเป็น**เวลาเดียว**ที่ Router
จะรู้ตัวว่า Neighbor หายไป **BFD (Bidirectional Forwarding Detection)** แก้ปัญหานี้โดยตรง

### 254.2 BFD คืออะไร และ Integration กับ OSPF

BFD เป็น Protocol แยกต่างหากที่ทำหน้าที่ **ตรวจจับ Path Failure ระดับ Sub-second** (Default
Interval หลักสิบ ms) โดยไม่ผูกกับ Routing Protocol ใดๆ — เมื่อผูกเข้ากับ OSPF (`ip ospf bfd`)
ทันทีที่ BFD Session ตรวจพบว่า Path ขาด จะแจ้ง OSPF ให้ **ตัด Neighbor ทิ้งทันที** โดยไม่ต้องรอ
Dead Timer เดิมเลย

```
! ตั้งค่า BFD Interval บน Interface ที่ต้องการ (Backbone Link ของ Lab)
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW1(config-if)# ip ospf bfd
CORE-SW1(config-if)# exit

! หรือเปิดทีเดียวทุก Interface ที่ Run OSPF Process นี้
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# bfd all-interfaces
CORE-SW1(config-router)# end
```

`bfd interval 150 min_rx 150 multiplier 3` หมายถึงส่ง/รับ BFD Packet ทุก 150ms และถือว่า Down
ถ้าขาดหายไป 3 ครั้งติด (150ms × 3 = 450ms) — **เร็วกว่า Dead Timer เดิม (40 วินาที) เกือบ 90
เท่า** ต้องตั้ง Interval ให้ตรงกันทั้งสองฝั่งของ Link เสมอ

```
CORE-SW1# show bfd neighbors detail

IPv4 Sessions
NeighAddr             LD/RD    RH/RS   State     Int
10.255.0.2            1/1      Up      Up        Po1
 Session state is UP and using echo function with 150 ms interval.
 ...

CORE-SW1# show ip ospf interface Port-channel1 | include BFD
  BFD enabled
```

### 254.3 SPF Throttling Timers — คุม CPU ตอน Topology เปลี่ยนถี่ๆ

```
Router(config-router)# timers throttle spf <spf-start> <spf-hold> <spf-max-wait>
```

หน่วยเป็น **มิลลิวินาที** ทั้งสามค่า:

| Parameter | ความหมาย |
|---|---|
| `spf-start` | ดีเลย์ก่อนรัน SPF ครั้งแรกหลัง Topology เปลี่ยน (ยิ่งต่ำ ยิ่งเร็ว) |
| `spf-hold` | ดีเลย์ขั้นต่ำระหว่าง SPF Run ครั้งที่ 2 เป็นต้นไป ถ้ามีการเปลี่ยนแปลงถี่ |
| `spf-max-wait` | เพดานสูงสุดของดีเลย์ (ค่าจะเพิ่มแบบ Exponential Backoff จาก `spf-hold` จนถึงเพดานนี้ ถ้า Topology ยังไม่นิ่ง) |

```
CORE-SW1(config-router)# timers throttle spf 10 100 5000
```

ความหมาย: SPF Run แรกหลัง Event ดีเลย์แค่ **10ms** (เร็วมาก) ถ้ามี Event ต่อเนื่องถี่ๆ ดีเลย์
ระหว่างรอบจะเพิ่มเป็น 100ms, 200ms, 400ms... **Exponential** ไปเรื่อยจนแตะเพดาน 5000ms แล้วค้าง
ที่ระดับนั้นตราบใดที่ Topology ยังไม่นิ่ง — กลไกนี้ป้องกันไม่ให้ **Interface Flap รัวๆ** (เช่น
สาย LAN หลวมๆ) ทำให้ CPU รัน SPF ซ้ำจนเกิน 100%

### 254.4 Incremental SPF (iSPF) — แนวคิด

Dijkstra's SPF เต็มรูปแบบ (Full SPF) คำนวณ Shortest Path Tree **ใหม่ทั้งหมด** ทุกครั้งที่มี
Event แม้ Event นั้นจะเป็นแค่ Leaf Network เล็กๆ ที่ไม่กระทบโครงสร้างหลักของ Tree เลยก็ตาม
**iSPF** ฉลาดกว่านั้น — ตรวจสอบก่อนว่า Event ที่เกิดขึ้นกระทบ**เฉพาะกิ่งใบ (Leaf) ของ SPF Tree**
หรือกระทบ**โครงสร้างหลัก (Transit Node)** ถ้าเป็นแค่ Leaf จะคำนวณใหม่**เฉพาะกิ่งนั้น**แทนที่จะ
รื้อทั้ง Tree — ประหยัด CPU ได้มากในระบบใหญ่ที่มี Access-Layer Flap บ่อยแต่ไม่กระทบ Core:

```
Router(config-router)# ispf
```

```
CORE-SW1# show ip ospf | include SPF
  Incremental-SPF enabled
  Initial SPF schedule delay 10 msecs
  Minimum hold time between two consecutive SPFs 100 msecs
  Maximum wait time between two consecutive SPFs 5000 msecs
```

> **ข้อควรรู้**: iSPF เป็น Cisco Enhancement (ไม่ใช่ Standard RFC) ทำงานได้เฉพาะ**ภายใน Router
> ตัวเดียว**ไม่ต้องพึ่งพา Router อื่นในระบบต้องรองรับด้วย (ไม่เหมือน BFD ที่ต้องมีทั้งสองฝั่ง)
> เปิดใช้ได้อิสระต่อ Router โดยไม่กระทบ Interoperability กับ Router ยี่ห้ออื่นในระบบ

---

## Step 255 — OSPF Unnumbered, Loopback-sourced Link, และ Network Type ขั้นสูง

### 255.1 `ip unnumbered` — ยืม IP จาก Interface อื่นเพื่อประหยัด Address Space

Backbone Link ของหลักสูตรนี้ (CORE↔DIST, CORE↔CORE) ใช้ Dedicated /30 ต่อ Link ตามที่กำหนดใน
`00-ip-address-plan.md` ซึ่งเหมาะกับ Enterprise Campus ที่มี Address Space เพียงพอ — แต่ในโลก
WAN/Service Provider ที่มี Point-to-Point Link จำนวนมาก (เช่น Tunnel Interface หรือ Serial WAN
หลักร้อยเส้นที่ WAN Edge) การจอง /30 แยกทุกเส้นสิ้นเปลือง Address มาก **`ip unnumbered`** แก้
ปัญหานี้โดยให้ Interface "ยืม" IP Address จาก Interface อื่น (มักเป็น **Loopback0**) แทนการมี
IP ของตัวเอง:

```
! ตัวอย่างบน WAN-EDGE-1 (สมมติ Tunnel Interface ไปยัง Backup Site ในอนาคต)
WAN-EDGE-1(config)# interface Tunnel10
WAN-EDGE-1(config-if)# ip unnumbered Loopback0
WAN-EDGE-1(config-if)# ip ospf 1 area 0
WAN-EDGE-1(config-if)# ip ospf network point-to-point
WAN-EDGE-1(config-if)# exit
```

ข้อดี: ไม่ต้องจอง Subnet ใหม่ต่อ Tunnel เลยแม้แต่ /30 เดียว ข้อเสีย: `ping`/`traceroute` ไปยัง
Interface นั้นตรงๆ ไม่ได้ (เพราะไม่มี IP เป็นของตัวเอง ต้องอาศัย Loopback0 แทนเสมอ) และ Link
แบบนี้ต้องเป็น **Point-to-Point Network Type เท่านั้น** (`ip unnumbered` ใช้กับ Broadcast/NBMA
ไม่ได้ เพราะไม่มี Subnet ของตัวเองให้ทำ DR Election)

### 255.2 ทบทวน + เจาะลึก OSPF Network Type ทั้งหมด

Part 11 แนะนำ Network Type แบบผิวเผิน (Broadcast, Point-to-Point) — ตารางนี้ครบทั้ง 5 ประเภท
ที่ CCNP ต้องแยกแยะได้:

| Network Type | DR/BDR Election | ค้นหา Neighbor | ต้อง Config `neighbor` Manual? | Hello/Dead Default | ใช้กับสื่อประเภทใด |
|---|---|---|---|---|---|
| **Broadcast** | ✅ มี | Multicast (224.0.0.5) อัตโนมัติ | ❌ ไม่ต้อง | 10/40s | Ethernet, VLAN SVI |
| **Non-Broadcast (NBMA)** | ✅ มี | ต้อง Manual ทั้งหมด | ✅ ต้อง (`neighbor <ip> priority <n>`) | 30/120s | Classic Frame Relay/ATM Hub-Spoke |
| **Point-to-Multipoint** | ❌ ไม่มี | Multicast อัตโนมัติ (ถ้าสื่อรองรับ) | ❌ ไม่ต้อง | 30/120s | mGRE/DMVPN ที่รองรับ Multicast |
| **Point-to-Multipoint Non-Broadcast** | ❌ ไม่มี | ต้อง Manual | ✅ ต้อง | 30/120s | NBMA ที่ไม่รองรับ Multicast เลย |
| **Point-to-Point** | ❌ ไม่มี | Multicast อัตโนมัติ | ❌ ไม่ต้อง | 10/40s | Serial P2P, Routed Port แบบที่ Lab นี้ใช้ทั้งหมด |
| **Loopback** | ❌ ไม่มี | N/A (Passive เสมอ) | ❌ ไม่ต้อง | N/A | Loopback Interface — Advertise เป็น Host Route /32 เสมอ |

> **กฎเหล็ก**: Network Type ต้อง**ตรงกันทั้งสองฝั่งของ Link เสมอ** (คล้าย Area Mismatch ใน
> Part 12 Step 119.4) ไม่งั้น DR/BDR Election จะไม่ตรงกัน (ฝั่งหนึ่งรอ DR อีกฝั่งไม่รอ) ทำให้
> Adjacency ค้างที่ State **2-WAY** ไม่ขึ้น FULL

### 255.3 Worked Example — NBMA Hub-and-Spoke (สมมติ เพื่อสาธิต Concept)

สมมติสถานการณ์ในอนาคตที่ WAN-EDGE-1 ทำหน้าที่ Hub เชื่อมไปยัง Site สาขาใหม่ 2 แห่งผ่าน mGRE
Tunnel (ลักษณะคล้าย DMVPN Hub-Spoke) — ตัวอย่างนี้เป็น**สถานการณ์สมมติ**เพื่อสาธิต Network Type
ขั้นสูง (ไม่ใช่ Device จริงในหลักสูตร เหมือนที่ Part 12 Step 117.3 สาธิต Virtual Link ด้วย Area
สมมติ):

```
                              WAN-EDGE-1 (Hub, สมมติ)
                          Tunnel0: 172.16.100.1/24
                          ┌─────────────────────┐
              mGRE ───────┤                       ├─────── mGRE
                          └─────────────────────┘
                    │                                    │
            SPOKE-1 (สมมติ)                       SPOKE-2 (สมมติ)
         Tunnel0: 172.16.100.2/24              Tunnel0: 172.16.100.3/24
```

**แบบที่ 1 — Non-Broadcast (ต้อง Manual Neighbor + มี DR Election)**:

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip address 172.16.100.1 255.255.255.0
WAN-EDGE-1(config-if)# ip ospf network non-broadcast
WAN-EDGE-1(config-if)# ip ospf priority 255           ! บังคับให้ Hub เป็น DR เสมอ
WAN-EDGE-1(config-if)# exit

WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# neighbor 172.16.100.2 priority 0    ! Spoke Priority 0 = ห้ามเป็น DR
WAN-EDGE-1(config-router)# neighbor 172.16.100.3 priority 0
WAN-EDGE-1(config-router)# end
```

**แบบที่ 2 — Point-to-Multipoint (ไม่ต้อง Manual Neighbor, ไม่มี DR)**:

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip ospf network point-to-multipoint
WAN-EDGE-1(config-if)# exit
! ไม่ต้อง config neighbor เลย — Spoke ค้นหากันเองผ่าน Multicast อัตโนมัติ
```

**ความต่างที่เห็นใน Routing Table**:

```
! Non-Broadcast: Route ไปยัง Spoke เป็น Subnet เดียวกันทั้งหมด (มองเป็น Broadcast Segment เดียว)
WAN-EDGE-1# show ip route ospf
O    172.16.100.0/24 is directly connected

! Point-to-Multipoint: OSPF ประกาศ Host Route /32 แยกต่อ Spoke แต่ละตัว (มองเป็น P2P หลายเส้น)
WAN-EDGE-1# show ip route ospf
O    172.16.100.2/32 [110/1000] via 172.16.100.2, Tunnel0
O    172.16.100.3/32 [110/1000] via 172.16.100.3, Tunnel0
```

> **สรุปเลือกใช้**: ถ้าสื่อ NBMA รองรับ Multicast จริง (เช่น mGRE ทุกวันนี้ส่วนใหญ่รองรับ) ควร
> เลือก **Point-to-Multipoint** เพราะลดภาระ Manual Config `neighbor` ทั้งหมดและไม่มี DR Election
> ที่อาจเป็นจุดคอขวด — **Non-Broadcast** เหมาะกับ Legacy Frame Relay/ATM ที่ไม่รองรับ Multicast
> เลยเท่านั้น ซึ่งพบน้อยลงมากในงานปัจจุบัน

---

## Step 256 — OSPF Graceful Restart / NSF สำหรับ High Availability Platform

### 256.1 ปัญหาที่ NSF แก้ — RP Switchover ไม่ควรทำให้ Traffic หยุด

Platform ระดับ Enterprise Core อย่าง **Catalyst 9500** (ตามที่ระบุใน Device Platform Reference)
มักมี **Dual Supervisor/Route Processor (RP)** พร้อม **SSO (Stateful Switchover)** — เมื่อ RP
หลักล่มและสลับไปใช้ RP สำรอง Hardware ยังคง Forward Packet ต่อได้ปกติ (Data Plane ไม่หยุด) แต่
**Control Plane (OSPF Process) ต้อง Restart ใหม่** บน RP สำรอง — ถ้าไม่มีกลไกพิเศษ Neighbor
รอบข้างจะเห็น Adjacency หายไปทันที (เพราะ OSPF Process ใหม่ยังไม่มี LSDB เดิมเลย) ทำให้เกิด
**SPF Re-run ทั่วทั้ง Area โดยไม่จำเป็น** (Topology ไม่ได้เปลี่ยนจริง แค่ Control Plane Restart)

### 256.2 Graceful Restart (RFC 3623) และ Cisco NSF

**Graceful Restart** ใช้ **Type 9 Opaque LSA ที่เรียกว่า Grace-LSA** ซึ่ง Router ที่กำลังจะ
Restart ส่งออกไปบอก Neighbor ล่วงหน้าว่า "กำลังจะ Restart Control Plane ขอเวลา Grace Period
เท่านี้ อย่าเพิ่งตัด Adjacency" — Neighbor ที่รองรับ (**Helper**) จะ**คง Adjacency และ Route
เดิมไว้ก่อน**ระหว่าง Grace Period แทนที่จะประกาศ Router LSA ใหม่ทันที ให้เวลา Router ที่ Restart
ซิงค์ LSDB กลับมาเหมือนเดิมโดยไม่ต้องมีใครรัน SPF ใหม่เลยหากไม่มีอะไรเปลี่ยนจริง

| บทบาท | ความหมาย |
|---|---|
| **NSF-capable (Restarting Router)** | Router ที่ **ทำ RP Switchover เอง** ต้องมี Hardware รองรับ SSO/Dual-RP จริง |
| **NSF-aware (Helper Router)** | Router **ข้างเคียง** ที่ช่วย "รอ" ให้ NSF-capable Router Restart เสร็จ — ไม่จำเป็นต้องมี Dual-RP เอง แค่รองรับ Protocol Extension (Grace-LSA) ก็เพียงพอ |

### 256.3 Config และ Verification

```
! บน CORE-SW1/CORE-SW2 (Catalyst 9500, สมมติมี Dual Supervisor ติดตั้งจริง)
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# nsf cisco              ! ใช้ Cisco Proprietary NSF
CORE-SW1(config-router)# end

! หรือใช้มาตรฐาน IETF (Interop กับ Vendor อื่นได้)
CORE-SW1(config-router)# nsf ietf
```

```
CORE-SW1# show ip ospf | include Non-Stop
  Non-Stop Forwarding enabled

CORE-SW1# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
           Mode = Duplex
     Manual Swact = enabled
```

> **ข้อจำกัดสำคัญของ Lab หลักสูตรนี้**: NSF มีประโยชน์เต็มรูปแบบเฉพาะบน Platform ที่มี **Dual
> Route Processor จริง** — DIST-SW1–4 (Catalyst 9300, Single Supervisor ตาม Device Platform
> Reference) **ไม่สามารถเป็น NSF-capable (Restarting Router) ได้** เพราะไม่มี RP สำรองให้
> Switchover แต่ยังทำหน้าที่ **NSF-aware (Helper)** ให้ CORE-SW1/CORE-SW2 ได้โดยไม่ต้อง Config
> อะไรเพิ่มเลย (พฤติกรรม Helper เป็น Default ของ IOS ทุกตัวที่รองรับ Grace-LSA อยู่แล้ว)

---

## Step 257 — `auto-cost reference-bandwidth` สำหรับ 10G/40G/100G: ทบทวน Cost Collision

### 257.1 ทวนสั้นๆ — Part 12 แก้ปัญหาอะไรไปแล้ว

Part 12 Step 118.3 พบว่า GigabitEthernet (SVI) และ TenGigabitEthernet (Uplink) ชนกันที่ Cost=1
เมื่อใช้ Reference Bandwidth Default (100 Mbps) จึงปรับเป็น **`auto-cost reference-bandwidth
100000`** (100 Gbps) ทำให้ Gigabit=100, TenGig=10 — แยกจากกันได้ถูกต้อง **Part นี้ไปต่อ**: จะ
เกิดอะไรขึ้นเมื่อ Backbone อัพเกรดไปเป็น **40G/100G จริง** (ตามที่ `00-ip-address-plan.md` ระบุ
ว่า Part 33-35 จะทำ Dual-homed CORE-DIST Redundancy ซึ่งมักมาพร้อมการอัพเกรด Backbone Speed)

### 257.2 คำนวณ Cost ที่ Reference Bandwidth = 100000 ครอบคลุมถึง 100G

```
Cost = Reference-Bandwidth(Mbps) / Interface-Bandwidth(Mbps)   (ปัดเศษลง ค่าต่ำสุด = 1)
```

| Interface Speed | คำนวณ | Cost ที่ Ref-BW=100000 |
|---|---|---|
| GigabitEthernet (1G) | 100000/1000 | 100 |
| TenGigabitEthernet (10G) | 100000/10000 | 10 |
| **40G (QSFP+ LACP Bundle)** | 100000/40000 | **2** |
| **100G (QSFP28)** | 100000/100000 | **1** |

> ข่าวดี: ค่า 100000 ที่ Part 12 เลือกไว้**ยังไม่ชนกันแม้ขยับไปถึง 100G** (2 vs 1 vs 10 vs 100
> ยังแยกกันชัดเจน) — แสดงว่าการเลือก Reference Bandwidth ล่วงหน้าให้สูงพอสมควรตั้งแต่ Part 12
> เป็นการวางแผนที่ถูกต้อง **แต่ต้องระวังทิศตรงข้าม**: การตั้งค่าสูงเกินความจำเป็นก็มีต้นทุนเช่นกัน

### 257.3 อันตรายของการตั้ง Reference Bandwidth สูงเกินไป — Cost Overflow

ค่าสูงสุดที่ `auto-cost reference-bandwidth` ตั้งได้คือ **4294967 (Mbps)** เพื่อ "future-proof"
ไปถึงระดับ 400G — แต่ถ้าตั้งค่านี้ไปเลยตั้งแต่วันนี้ทั้งที่ Backbone ยังเป็น 1G/10G:

| Interface Speed | Cost ที่ Ref-BW=4294967 | ปัญหา |
|---|---|---|
| GigabitEthernet (1G) | 4294967/1000 ≈ **4294** | สูงกว่าค่าที่ Part 12 ใช้ (100) ถึง ~43 เท่า |
| TenGigabitEthernet (10G) | 4294967/10000 ≈ **429** | เช่นกัน |

**OSPF Cost สะสม (Path Cost) มีเพดานสูงสุดที่ 65535 (16-bit)** — ถ้าแต่ละ Hop มี Cost ประมาณ
4294 (Gigabit SVI) แค่ **15-16 Hop** ก็ชนเพดานแล้ว (4294 × 16 ≈ 68,704 > 65535) ในระบบ Enterprise
ขนาดใหญ่ที่มี Hop นับสิบจริง **นี่คือความเสี่ยงที่ทำให้ Path ที่ควรจะ Reachable กลับกลายเป็น
Unreachable เพราะ Cost Overflow** — บทเรียนสำคัญ: **อย่าตั้ง Reference Bandwidth สูงเกินความ
จำเป็นของ Roadmap 5-10 ปีข้างหน้า** ค่า 100000 (100G) เป็นจุดสมดุลที่เหมาะสมสำหรับ Enterprise
ส่วนใหญ่ในปัจจุบัน ถ้าจะมี Link เร็วกว่านั้นจริงๆ (400G) แนะนำใช้ **`cost <value>` กำหนดตรงที่
Interface นั้นเป็นกรณีพิเศษ** แทนการยกเครื่อง Reference Bandwidth ทั้งระบบใหม่

### 257.4 Worked Example — Po1 อัพเกรดจาก 10G เดี่ยว เป็น LACP Bundle 4×10G (40G รวม)

สมมติ Part 33-35 อัพเกรด Port-channel1 (CORE-SW1↔CORE-SW2) จาก Link เดี่ยวเป็น **LACP Bundle
4 เส้น 10G รวมเป็น 40G** — Cost ของ Po1 ต้องคำนวณใหม่จาก Bandwidth รวมของ Bundle:

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# bandwidth 40000000        ! Kbps — บอก IOS ว่า Bundle นี้คือ 40 Gbps จริง
CORE-SW1(config-if)# end

CORE-SW1# show ip ospf interface Port-channel1 | include Cost
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 2
                                                    <- เปลี่ยนจาก 10 (Part 12) เป็น 2
```

**Path Cost ใหม่จาก DIST-SW1 (VLAN10) ไปยัง Server1 (VLAN30, DIST-SW3)** เทียบกับที่ Part 12
Step 118.3 คำนวณไว้ (Total เดิม = 30):

```
DIST-SW1 ──[Te1/1/1, cost 10]── CORE-SW1 ──[Po1, cost 2]── CORE-SW2 ──[Te1/0/1, cost 10]── DIST-SW3

Total Cost ใหม่ = 10 (uplink) + 2 (Po1 หลังอัพเกรด) + 10 (downlink) = 22   (เดิม 30)

DIST-SW1# show ip route | include 10.10.30
O IA    10.10.30.0/24 [110/22] via 10.255.10.1, TenGigabitEthernet1/1/1
```

> **กฎเหล็กที่ต้องย้ำอีกครั้ง**: `auto-cost reference-bandwidth` ต้อง **Match กันทุก Router ใน
> OSPF Domain เดียวกันเสมอ** — ถ้า CORE-SW1 ใช้ 100000 แต่ CORE-SW2 ยังเป็น Default 100 (ลืม
> Sync ตอน Config) **Adjacency จะยังขึ้นปกติ** (ไม่เหมือน Area Mismatch ที่ Reject ตั้งแต่ Hello)
> แต่ **Cost ที่แต่ละฝั่งคำนวณจะไม่ตรงกัน** ทำให้เกิด **Asymmetric Routing** (ขาไปกับขากลับใช้
> คนละ Path) และ Suboptimal Path โดยไม่มี Error แจ้งเตือนชัดเจนเลย — ต้องตรวจด้วย `show ip ospf`
> เทียบค่า `Reference bandwidth unit is ... Mbps` ทุกตัวหลัง Migration เสมอ

---

## Step 258 — LSA Throttling, Flood Reduction, และ Demand Circuit

### 258.1 `timers throttle lsa` — คุมอัตราการสร้าง LSA ของตัวเอง (ต่างจาก SPF Throttle)

Step 254 สอน `timers throttle spf` ที่คุมว่า **Router จะรัน SPF บ่อยแค่ไหน** — ส่วน `timers
throttle lsa` คุมคนละเรื่อง: **ความถี่ที่ Router จะสร้าง/Re-originate LSA ของตัวเอง** เมื่อมี
การเปลี่ยนแปลงถี่ๆ (เช่น Interface Flap ที่ทำให้ Router LSA ของตัวเองต้องอัปเดตซ้ำๆ):

```
Router(config-router)# timers throttle lsa all <start> <hold> <max>
```

```
CORE-SW1(config-router)# timers throttle lsa all 10 100 5000
```

ทำงานคล้าย SPF Throttle (Exponential Backoff) แต่ควบคุมฝั่ง **Originate** ไม่ใช่ฝั่ง **Consume**
— ทั้งสองกลไกทำงานร่วมกัน: LSA Throttle ลดปริมาณ LSA ที่ถูกสร้าง/Flood ออกไปตั้งแต่ต้นตอ ส่วน
SPF Throttle ลดความถี่การประมวลผลที่ปลายทาง เมื่อ Interface Flap ถี่ๆ

### 258.2 `ospf flood-reduction` — ลด Overhead จาก Periodic LSA Refresh

Standard OSPF (RFC 2328) กำหนดให้ LSA ทุกใบต้องถูก **Re-flood (Refresh) ทุก 30 นาที
(LSRefreshTime)** แม้เนื้อหาจะไม่เปลี่ยนเลยก็ตาม เพื่อป้องกันไม่ให้ LSA หมดอายุ (MaxAge, 60
นาที) ในระบบขนาดใหญ่ที่มี LSA จำนวนมากหรือ Link ความเร็วต่ำ (เช่น WAN สำรองแบบ Low-bandwidth)
การ Re-flood ทุก 30 นาทีนี้กิน Bandwidth/CPU โดยไม่จำเป็นถ้า Topology นิ่งสนิท — **Flood
Reduction** แก้ปัญหานี้ด้วยการตั้ง Flag **DoNotAge (DNA)** ให้ LSA ที่ไม่เปลี่ยนแปลง ทำให้ไม่
ต้อง Refresh ตามรอบเดิมอีก (Flood เฉพาะตอนมีการเปลี่ยนแปลงจริงเท่านั้น):

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ip ospf flood-reduction
```

### 258.3 `ip ospf demand-circuit` — สำหรับ Link แบบ Metered/On-Demand

`ip ospf demand-circuit` ไปไกลกว่า Flood Reduction อีกขั้น — ใช้กับ Link ที่มีค่าใช้จ่ายต่อการ
ใช้งานจริง (Metered WAN, ISDN Dial-up สมัยก่อน) หรือ Link ที่ต้องการลด Control-plane Traffic
ให้เหลือน้อยที่สุด: หลัง Adjacency Establish ครั้งแรกสำเร็จ (FULL State) **ทั้ง Periodic Hello
และ Periodic LSA Refresh จะหยุดส่งไปเลย** (ไม่ใช่แค่ชะลอ) — Link จะ "เงียบ" สนิทตราบใดที่ไม่มี
การเปลี่ยนแปลง Topology จริง Adjacency จะไม่ถูกตัดจาก Dead Timer เพราะทั้งสองฝั่งรู้ล่วงหน้าแล้ว
ว่าจะไม่มี Hello มาตามรอบปกติ:

```
Router(config-if)# ip ospf demand-circuit
```

> **ใช้งานจริงในบริบทหลักสูตรนี้**: เหมาะกับ Backup Path ความเร็วต่ำที่ WAN Edge (เช่น 4G/5G
> Backup Link หรือ MPLS สำรองที่คิดค่าบริการตาม Data Usage) ซึ่งปกติควรอยู่เฉยๆ ไม่ Generate
> Traffic OSPF โดยไม่จำเป็นตราบใดที่ Path หลักยังทำงานปกติ — Config ที่ปลายทางเดียวก็เพียงพอ
> (ไม่ต้อง Config ทั้งสองฝั่งเหมือน BFD Interval)

---

## Step 259 — Advanced Verification/Troubleshooting: Statistics, Traffic, Debug SPF

### 259.1 `show ip ospf statistics` — ประวัติการรัน SPF

```
CORE-SW1# show ip ospf statistics

OSPF Router with ID (1.1.1.1) (Process ID 1)

 Area 1: SPF algorithm executed 8 times
  SPF calculation time
        Delta T       Reason
        00:15:32      R, N
        00:04:11      R
        00:00:02      R                     <- ระยะห่างสั้นผิดปกติ = สัญญาณ Flapping
```

คอลัมน์ **Reason** บอกสาเหตุที่ทำให้ต้องรัน SPF: `R` = Router LSA เปลี่ยน, `N` = Network LSA
เปลี่ยน, `SN` = Summary Network LSA เปลี่ยน — ถ้าเห็น Timestamp ห่างกันแค่ไม่กี่วินาทีซ้ำๆ
หลายรอบติดกัน นั่นคือสัญญาณของ **SPF Flapping** ที่ต้องสืบสาเหตุต่อ (ดู 259.4)

### 259.2 `show ip ospf traffic` — นับ Packet OSPF ต่อ Interface

```
CORE-SW1# show ip ospf traffic

TenGigabitEthernet1/0/1
  OSPF packets received/sent
  Type       Rx           Tx
  Hello      1204         1205
  DBD        3            3
  LS-Req     2            2
  LS-Update  5            6
  LS-Ack     5            4
```

ใช้เทียบ **Rx vs Tx** ระหว่าง Interface สองฝั่งเดียวกัน — ถ้าฝั่งหนึ่งส่ง Hello ออกจำนวนมากแต่
อีกฝั่งนับ Rx ต่ำกว่ามาก แปลว่ามี Packet Loss ทาง Physical/L2 ระหว่างทาง (ไม่ใช่ปัญหา OSPF
Config) เป็นเครื่องมือแยกปัญหา **Control-plane Drop** ออกจาก **Config Mismatch** ได้ชัดเจน

### 259.3 `debug ip ospf spf statistic` — ดู SPF Trigger แบบ Real-time

```
CORE-SW1# debug ip ospf spf statistic
CORE-SW1#
*Sep 26 11:02:14.221: OSPF: Schedule SPF: change in LS ID 10.255.10.0
*Sep 26 11:02:14.231: OSPF: Old SPF delay 10 msec, new delay 10 msec
*Sep 26 11:02:14.241: OSPF: Total SPF time 0.412 ms, 2 SPF, 0 next hop calc
CORE-SW1# undebug all
```

> **ข้อควรระวัง**: `debug ip ospf spf` (ไม่ใส่ `statistic`) แสดงรายละเอียด SPF Algorithm ทุก
> ขั้นตอน**กินทรัพยากร CPU สูงมาก**บน Router ที่มี LSDB ใหญ่ — ใช้ `debug ip ospf spf statistic`
> (สรุปแค่เวลาที่ใช้และจำนวนครั้ง) แทนในการวินิจฉัยเบื้องต้นเสมอ และเปิด Debug เฉพาะช่วงสั้นๆ
> ที่จำเป็นจริงบน Production เท่านั้น ปิดด้วย `undebug all` ทันทีที่ได้ข้อมูลพอ

### 259.4 Workflow วินิจฉัย SPF-Run Flapping

```
1. show ip ospf statistics                 → พบ Timestamp ถี่ผิดปกติ (ต่ำกว่า 1 วินาทีซ้ำๆ)
2. show logging | include LINEPROTO|OSPF   → หา Interface ที่ Flap ตรงช่วงเวลาเดียวกัน
3. show interfaces <if> | include error    → เช็ค CRC/Input Error ที่ต้นเหตุ (สาย/SFP เสีย?)
4. show ip ospf statistics detail          → ดู Advertising Router ที่เป็นต้นตอ LSA ซ้ำๆ
5. เมื่อพบ Root Cause ทาง Physical แล้ว    → แก้ที่ต้นเหตุจริง (เปลี่ยนสาย/SFP)
   ระหว่างรอแก้ไข                          → timers throttle spf/lsa (Step 254/258) ช่วยกัน CPU
                                              พังจาก Flap แต่ **ไม่ใช่การแก้ปัญหาที่ต้นเหตุ**
```

> **บทเรียนสำคัญ**: SPF/LSA Throttling เป็นแค่ **มาตรการบรรเทา (Mitigation)** ไม่ใช่การแก้ไข
> ต้นเหตุ — ถ้า Physical Layer มีปัญหาจริง (สาย LAN หลวม, SFP เสื่อม) ต้องแก้ที่ Hardware เสมอ
> Throttling แค่ทำให้ระบบไม่ล่มระหว่างรอซ่อมเท่านั้น

---

## Step 260 — Lab เต็มรูปแบบ: BFD + Reference-Bandwidth + Area Filter-list + Stub-Router Maintenance

Lab สรุปของ Part นี้ — นำทุกเทคนิคจาก Step 251-259 มาประกอบใช้จริงกับ Topology 3-Area จาก
Part 12 (CORE-SW1/CORE-SW2 = ABR, DIST-SW1-4 = Internal Router) โดยไม่แก้ไข Address Plan เดิม

### 260.1 ภาพรวม Lab สุดท้าย

```
                    Area 1                        Area 0                       Area 2
              (BFD 150ms บน Uplink)         (BFD 150ms บน Po1)          (BFD 150ms บน Uplink)
┌──────────┐                          ┌────────────────────────┐                          ┌──────────┐
│ DIST-SW1 │──Te1/1/1──[BFD]──┐      │   Po1 .1 ══[BFD]══ Po1 .2│      ┌──[BFD]──Te1/1/1──│ DIST-SW3 │
└──────────┘                    │      │                          │      │                    └──────────┘
                          ┌─────┴─────┐│ area 1 filter-list       │┌─────┴─────┐
                          │ CORE-SW1  ││  (block 10.10.40.0/24)  ││ CORE-SW2  │
                          │  (ABR)    │└────────────────────────┘│  (ABR)    │
                          └─────┬─────┘                            └─────┬─────┘
┌──────────┐                    │       ref-bw 100000 (ทั้ง 6 ตัว)        │      ┌──────────┐
│ DIST-SW2 │──Te1/1/1──[BFD]──┘       timers throttle spf 10 100 5000    └──[BFD]──Te1/1/1──│ DIST-SW4 │
└──────────┘                                                                    └──────────┘
```

### 260.2 CORE-SW1 — Delta Config (เพิ่มจาก Baseline Part 12 Step 120.2)

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW1(config-if)# ip ospf bfd
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW1(config-if)# ip ospf bfd
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW1(config-if)# ip ospf bfd
CORE-SW1(config-if)# exit

CORE-SW1(config)# ip prefix-list BLOCK-VLAN40 seq 5 deny 10.10.40.0/24
CORE-SW1(config)# ip prefix-list BLOCK-VLAN40 seq 10 permit 0.0.0.0/0 le 32

CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# area 1 filter-list prefix BLOCK-VLAN40 in
CORE-SW1(config-router)# timers throttle spf 10 100 5000
CORE-SW1(config-router)# timers throttle lsa all 10 100 5000
CORE-SW1(config-router)# ispf
CORE-SW1(config-router)# nsf cisco
CORE-SW1(config-router)# end
CORE-SW1# copy running-config startup-config
```

### 260.3 CORE-SW2 — Delta Config (เหมือน CORE-SW1 แต่ฝั่ง Area 2)

```
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW2(config-if)# ip ospf bfd
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/1
CORE-SW2(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW2(config-if)# ip ospf bfd
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/2
CORE-SW2(config-if)# bfd interval 150 min_rx 150 multiplier 3
CORE-SW2(config-if)# ip ospf bfd
CORE-SW2(config-if)# exit

! Area 2 ไม่มี VLAN เฉพาะที่ต้องซ่อนจาก Area 1 ในรอบนี้ — Filter-list ทำฝั่ง CORE-SW1 ฝั่งเดียว
! เพียงพอตาม Requirement (Policy กำหนดให้ VLAN40 ห้ามออกนอก Area 2 เท่านั้น ไม่ใช่สองทาง)

CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# timers throttle spf 10 100 5000
CORE-SW2(config-router)# timers throttle lsa all 10 100 5000
CORE-SW2(config-router)# ispf
CORE-SW2(config-router)# nsf cisco
CORE-SW2(config-router)# end
CORE-SW2# copy running-config startup-config
```

### 260.4 DIST-SW1–4 — Delta Config (BFD บน Uplink + SPF Throttle เหมือนกันทุกตัว)

```
! ===== ตัวอย่าง DIST-SW1 (โครงสร้างเดียวกันทั้ง DIST-SW1/2/3/4 ต่างแค่ Interface/Area) =====
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# bfd interval 150 min_rx 150 multiplier 3
DIST-SW1(config-if)# ip ospf bfd
DIST-SW1(config-if)# exit

DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# timers throttle spf 10 100 5000
DIST-SW1(config-router)# timers throttle lsa all 10 100 5000
DIST-SW1(config-router)# ispf
DIST-SW1(config-router)# end
DIST-SW1# copy running-config startup-config

! ทำซ้ำแบบเดียวกันบน DIST-SW2, DIST-SW3, DIST-SW4 (เปลี่ยนแค่ hostname/interface ตาม IP Plan)
```

### 260.5 Verify #1 — BFD ขึ้นครบทุกคู่

```
CORE-SW1# show bfd neighbors

NeighAddr             LD/RD    RH/RS   State     Int
10.255.0.2            1/1      Up      Up        Po1
10.255.10.2           2/1      Up      Up        Te1/0/1
10.255.20.2           3/1      Up      Up        Te1/0/2

CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:32    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:35    10.255.20.2     TenGigabitEthernet1/0/2
```

### 260.6 Verify #2 — Area Filter-list ทำงานถูกต้อง

```
DIST-SW1# show ip route ospf | include 10.10.40
                                                    <- หายไปตามที่ตั้งใจ (Policy Isolation)

DIST-SW2# show ip route ospf | include 10.10.40
                                                    <- หายไปเช่นกัน (ทั้ง Area 1)

DIST-SW3# show ip route ospf | include 10.10.40
O IA    10.10.40.0/24 [110/10] via 10.255.30.1, TenGigabitEthernet1/1/1
                                                    <- ยังเห็นปกติ (อยู่ Area 2 เดียวกับ VLAN40 เอง)
```

### 260.7 Verify #3 — Stub Router Maintenance Mode บน DIST-SW1 (สาธิตกลไก แม้ Lab ยัง Single-homed)

```
! ก่อนเริ่ม Maintenance บน DIST-SW1 (สมมติจะเปลี่ยน SFP บน Uplink)
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# max-metric router-lsa on-shutdown 300
DIST-SW1(config-router)# end

DIST-SW1# show ip ospf database router 1.1.1.11 | include Metrics
       TOS 0 Metrics: 65535                        <- Router LSA แจ้ง Cost สูงสุดแล้ว

! CORE-SW1 เห็น Route ผ่าน DIST-SW1 มี Cost พุ่งสูงทันที (แม้ยังไม่มี Path สำรองให้เปลี่ยนจริง
! ในสถาปัตยกรรม Single-homed ปัจจุบัน — แต่ Adjacency ยังคง FULL ปกติ ไม่ตัดทันทีเหมือน Shutdown ตรงๆ)
CORE-SW1# show ip route | include 10.10.10
O       10.10.10.0/24 [110/65535] via 10.255.10.2, TenGigabitEthernet1/0/1

! ทำ Maintenance จริง แล้วถอด Max-Metric กลับ
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# no max-metric router-lsa
DIST-SW1(config-router)# end

DIST-SW1# show ip ospf database router 1.1.1.11 | include Metrics
       TOS 0 Metrics: 1                             <- กลับสู่ Cost ปกติ
```

**สรุปผลลัพธ์ของ Lab**: Backbone ทั้งหมดของหลักสูตรตอนนี้มี **BFD Sub-second Failure Detection**
ครบทุก Link, **SPF/LSA Throttling** ป้องกัน CPU พังจาก Flapping, **iSPF** ลดภาระ SPF สำหรับ
Event ที่ไม่กระทบโครงสร้างหลัก, **NSF** พร้อมรองรับ RP Switchover บน CORE Layer, **Area
Filter-list** บังคับ Policy Isolation ระดับ Area ได้จริง, และมี **Runbook Stub-Router
Maintenance** ที่พร้อมใช้งานทันทีที่ Dual-homed Redundancy มาถึงใน Part 33-35 ✅

---

## แบบฝึกหัดทวนความเข้าใจ Part 26

1. เพราะเหตุใด `distribute-list <acl> in` บน OSPF จึงไม่สามารถ "ซ่อน" Subnet จาก Router
   ตัวอื่นในระบบได้ ต้องใช้เครื่องมือใดแทนถ้าต้องการหยุด Type 3 LSA ไม่ให้ข้าม Area จริง?
2. ใน Type 7 LSA, **P-bit** มีหน้าที่อะไร และถ้า P-bit = 0 จะเกิดอะไรขึ้นที่ ABR?
3. `max-metric router-lsa on-shutdown <seconds>` ต่างจาก `max-metric router-lsa
   on-startup wait-for-bgp` อย่างไร ใช้ในสถานการณ์ไหนต่างกัน?
4. BFD กับ SPF Throttling Timer (`timers throttle spf`) แก้ปัญหาคนละเรื่องกันอย่างไร
   ใช้ร่วมกันแล้วได้ประโยชน์อะไรเพิ่ม?
5. ถ้าตั้ง `auto-cost reference-bandwidth` สูงเกินไป (เช่นค่าสูงสุด 4294967) จะเกิดความเสี่ยง
   อะไรกับ OSPF Path Cost ในระบบที่มีจำนวน Hop มาก?

**เฉลย:**

1. เพราะ `distribute-list in` กรองแค่การติดตั้งเข้า **RIB ของ Router ตัวเองเท่านั้น** ไม่ได้
   หยุด LSA Flooding เลย Router อื่นในระบบยังได้รับ LSA และเห็น Route นั้นตามปกติ — ถ้าต้องการ
   หยุด Type 3 LSA ไม่ให้ข้าม Area จริง ต้องใช้ **`area <id> filter-list prefix <name> in|out`**
   ที่ ABR เท่านั้น
2. P-bit (Propagate Bit) บอก ABR ว่า Type 7 LSA ใบนี้**ควรถูกแปลงเป็น Type 5 แล้วส่งออกไป Area
   อื่นหรือไม่** — ถ้า P-bit = 1 (Default) ABR จะแปลงและ Flood ออก แต่ถ้า P-bit = 0 ABR **จะไม่
   แปลง** Route นั้นใช้งานได้เฉพาะภายใน NSSA Area เดียวกันเท่านั้น (มักใช้ป้องกันการแปลงซ้ำซ้อน
   เมื่อมี ABR หลายตัวใน NSSA เดียวกัน)
3. `on-shutdown <seconds>` ใช้ตอน **จะ Reload/ปิด Router** — ตั้ง Max-Metric ชั่วคราวก่อนหยุด
   Process จริงเพื่อ Drain Traffic ออกก่อน ส่วน `on-startup wait-for-bgp` ใช้ตอน **Router เพิ่ง
   บูตขึ้นมา** — ค้าง Max-Metric ไว้จนกว่า BGP จะ Converge เต็มที่ก่อน เพื่อป้องกัน Traffic
   Black-hole ช่วงที่ OSPF ขึ้นแล้วแต่ BGP Table ยังไม่พร้อม
4. BFD ตรวจจับว่า **Path ขาดเร็วแค่ไหน** (Failure Detection ระดับ Sub-second แทน Dead Timer
   40 วินาที) ส่วน `timers throttle spf` ควบคุมว่า **หลังรู้ว่า Topology เปลี่ยนแล้ว จะรัน SPF
   เร็ว/บ่อยแค่ไหน** โดยไม่ให้ CPU พังถ้า Event ถี่เกินไป — ใช้ร่วมกันได้ประโยชน์คือ **รู้เร็ว
   (BFD) และประมวลผลเร็วในครั้งแรกแต่ปลอดภัยจาก Flapping ในครั้งถัดไป (SPF Throttle Exponential
   Backoff)**
5. Cost ต่อ Interface Speed ต่ำ (เช่น GigabitEthernet) จะพุ่งสูงมาก (~4294 ที่ Ref-BW สูงสุด)
   และเพราะ **OSPF Path Cost สะสมมีเพดานที่ 65535 (16-bit)** ระบบที่มีจำนวน Hop มาก (เช่น 15-16
   Hop) อาจ**ชนเพดาน Cost Overflow** ทำให้ Path ที่ควร Reachable กลับกลายเป็น Unreachable โดย
   ไม่มี Error แจ้งเตือนชัดเจน

---

## สรุป Part 26

Part นี้ยกระดับ OSPF ของหลักสูตรจาก Multi-Area Design พื้นฐาน (Part 11-12) ขึ้นสู่ **ความลึก
ระดับ CCNP ENCOR** เต็มรูปแบบ — เรียนรู้ความต่างที่แท้จริงระหว่าง `distribute-list` (RIB-only)
กับ `area filter-list` (LSA-level filtering จริง), กลไก **Stub Router** สำหรับ Maintenance
Mode แบบมืออาชีพ, เจาะลึก **NSSA ถึงระดับ P-bit และ Forwarding Address**, ยกระดับ Convergence
ด้วย **BFD + SPF/LSA Throttling + iSPF**, ทำความเข้าใจ **Network Type ขั้นสูงและ Unnumbered
Interface** สำหรับสถานการณ์ WAN/NBMA, เตรียมความพร้อม **NSF/Graceful Restart** สำหรับ Platform
ระดับ Enterprise Core, ทบทวนและต่อยอด **Reference Bandwidth Tuning** ให้ปลอดภัยจาก Cost
Overflow, และปิดท้ายด้วย **Advanced Verification Toolset** (`show ip ospf statistics`,
`show ip ospf traffic`, `debug ip ospf spf statistic`) ที่ใช้วินิจฉัยปัญหาระดับ Production จริง
✅

**สิ่งที่ Lab หลักสูตรนี้ยังรอ**: ประโยชน์เต็มรูปแบบของ Stub Router และ Redundant ABR จะเห็นผล
ชัดเจนที่สุดเมื่อ **Dual-homed CORE-DIST Redundancy** มาถึงใน **Part 33-35** ตามที่ IP Plan
ระบุไว้ — Part ถัดไปจะพา OSPF ไปสู่มิติใหม่ที่ยังไม่เคยแตะเลยตลอดหลักสูตร: **OSPFv3 สำหรับ
IPv6** ซึ่งมีโครงสร้าง LSA และวิธี Config ที่ต่างจาก OSPFv2 พอสมควรแม้แนวคิด Multi-Area/SPF
จะเหมือนกันก็ตาม

**ไปต่อ:** [Part 27 — OSPFv3 for IPv6 →](part-027-ospfv3-ipv6.md)
