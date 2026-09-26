# Part 83 — Enterprise Multicast Deep Dive: Advanced Features
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 821–830 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 82 — End-to-End Enterprise QoS Design](part-082-end-to-end-enterprise-qos-design.md)
> ที่ปิดท้ายเรื่อง QoS Design ทั้ง Enterprise แบบ End-to-End แล้ว Part นี้วกกลับมาที่ **Multicast**
> อีกครั้งเป็นครั้งที่ 3 ต่อจาก [Part 41](part-041-multicast-fundamentals.md) (IGMP, PIM-DM/SM
> พื้นฐาน), [Part 42](part-042-multicast-advanced.md) (Auto-RP, BSR, Anycast RP + MSDP, SSM,
> Bidir-PIM ภาพรวม) และ [Part 75](part-075-advanced-multicast-enterprise.md) (Scaling ระดับ
> Enterprise, MVPN Concept, Boundary Filtering, QoS Integration, Multicast over DMVPN, IGMP
> Scaling, MSDP Mesh-group)
>
> **จุดสำคัญที่ต้องเข้าใจก่อนอ่าน Part นี้**: Part 75 ตอบคำถามเรื่อง **"จะ Design/Scale Multicast
> ทั้ง Enterprise อย่างไร"** ไปครบแล้ว (RPF ที่ Scale, MVPN Preview, Boundary, QoS, DMVPN, IGMP
> Limit, MSDP Mesh-group) — **Part นี้ไม่ทำซ้ำเรื่องเดิมเลยแม้แต่หัวข้อเดียว** แต่จะ **เติมเต็ม
> Feature/Protocol เชิงลึกที่ยังไม่ได้ Config จริงในหลักสูตรเลยจนถึงจุดนี้**: Bidir-PIM แบบ Full
> Deep-dive (Part 42 ให้แค่ Concept), MVPN ทั้ง 2 รุ่น (Rosen GRE เต็มรูปแบบ + BGP-based รุ่นใหม่ที่
> Part 75 ยังไม่ได้แตะ), IPv6 Multicast (MLD/PIM6 — ยังไม่เคย Config เลยตลอดหลักสูตร), MSDP SA-filter
> ระดับ (S,G), Multicast Rate-Limiting, PIM-DM Legacy Feature ที่เหลือ, Multicast+NAT Incompatibility,
> และ mDNS/Bonjour Gateway ซึ่งเป็น "Multicast" อีกประเภทที่ไม่เกี่ยวกับ PIM เลยแต่พบบ่อยที่สุดใน
> Enterprise Wireless จริง — ถือเป็น Part สุดท้ายของสาย Multicast ในหลักสูตรนี้ก่อนจะไปสู่
> [Part 84 — VXLAN/EVPN Fundamentals](part-084-vxlan-evpn-fundamentals.md) ที่ Multicast (หรือ
> Ingress Replication) จะกลับมาเป็นกลไกสำคัญของ BUM Traffic ใน VXLAN Underlay อีกครั้ง

## Use Case ที่จะขับเคลื่อน Part นี้: 4 โจทย์ที่ Part 75 ยังไม่ครอบคลุม

| โจทย์ใหม่ | ทำไม Part 75 ไม่ครอบคลุม | Step ที่ตอบ |
|---|---|---|
| **Trading Floor Order-Update** — Trader ทุกคนใน VLAN 10 SALES ต้องส่ง/รับ Order Update ของกันและกันแบบ Many-to-Many กับ Matching Engine ใน VLAN 30 SERVERS (คนละแบบกับ Market-Data SSM `232.50.1.1` ที่เป็น One-to-Many และ Part 75/42 คุยแค่ Concept) | Part 42 Step 418 ให้ Concept Bidir-PIM ผิวๆ ไม่มี DF Election เชิงลึก ไม่มี Phantom RP ไม่มี Config เต็ม | 821 |
| **Guest Portal Advertisement Video ข้าม Site** — VRF `GUEST` (Part 51) ต้องกระจาย Video เดียวกันข้าม Site ผ่าน Core ที่ไม่มี MPLS จริง | Part 75 Step 742 ให้แค่ Concept MVPN 4-5 บรรทัด Config ไม่ได้ลง Lab จริงเลย | 822, 823 |
| **All-Hands Video ต้องมี IPv6 ด้วย** — บริษัทเริ่ม Dual-Stack (Part 8) เต็มรูปแบบ ผู้บริหารต้องการดู Stream ผ่าน IPv6 ด้วยเช่นกัน | Part 41/42/75 ทั้งหมด Config เฉพาะ IPv4 Multicast เท่านั้น ไม่เคยแตะ MLD/PIM6 เลย | 824 |
| **รอง Laptop ปลอมตัวเป็น Source** — Guest Wi-Fi (VLAN 40) มีเครื่องแปลกส่ง Traffic ไปที่ Group All-Hands (`239.1.1.1`) หลอกเป็น STREAM-SRV แล้ว MSDP อาจกระจาย SA ไปทั้ง Regional DC/Branch | Part 75 Step 748 ตั้ง MSDP Mesh-group เพื่อ Topology/Loop-prevention เท่านั้น ไม่ได้กรองที่ระดับ (S,G) เลย | 825 |
| **AirPlay/Bonjour ข้าม VLAN บน Wireless** — พนักงานเชื่อม Wi-Fi (VLAN 40) อยากเห็น Apple TV/Printer ที่อยู่ VLAN 10/30 แต่ mDNS Multicast (`224.0.0.251`) มี TTL=1 ข้าม VLAN ไม่ได้โดย Design | ไม่มี Part ไหนแตะ mDNS/Bonjour มาก่อนเลยตลอดหลักสูตร | 829, 830 |

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 821 | Bidirectional PIM (Bidir-PIM) Deep Dive — DF Election เชิงลึก, Phantom RP, Config เต็มสำหรับ Trading Floor |
| 822 | Multicast VPN (MVPN) Rosen GRE Model — Default MDT / Data MDT เต็มรูปแบบ ต่อจาก VRF-Lite Part 51 |
| 823 | mVPN BGP-based (MDT-SAFI / NG-MVPN) — ทางเลือกยุคใหม่แทน GRE, เปรียบเทียบ + Config เชิง Concept |
| 824 | IPv6 Multicast — MLD, PIM6 (`ipv6 pim`), เปิด Dual-Stack ให้ All-Hands Video ต่อจาก Part 8/41 |
| 825 | MSDP SA-filter — กรอง (S,G) ระดับ Source เพื่อความปลอดภัย ป้องกัน Rogue Source กระจายทั้ง Enterprise |
| 826 | Multicast Rate-Limiting ต่อ Source — `ip multicast rate-limit` ป้องกัน Source ที่ Flood ผิดปกติ |
| 827 | PIM Join/Prune Suppression และ PIM State Refresh — ส่วนที่เหลือของ PIM-DM สำหรับความสมบูรณ์ CCIE |
| 828 | Multicast และ NAT — ทำไม Multicast ข้าม NAT Boundary ไม่ได้ดี และผลต่อ Design |
| 829 | Multicast Auto-Discovery — mDNS/Bonjour (`224.0.0.251`) และ Cisco mDNS Gateway ข้าม VLAN |
| 830 | Lab เต็มรูปแบบ: Bidir-PIM Trading Floor + IPv6 Multicast Dual-Stack + MSDP SA-filter + mDNS Gateway |

---

## Step 821 — Bidirectional PIM (Bidir-PIM) Deep Dive

### 821.1 ทวนจุดที่ Part 42 Step 418 ทิ้งไว้

[Part 42 Step 418](part-042-multicast-advanced.md#step-418--bidirectional-pim-bidir-pim) สอนแค่
**แนวคิด**: Bidir-PIM ไม่มี `(S,G)`, ไม่มี PIM Register, ใช้ **DF (Designated Forwarder)** แทน RPF
Check ปกติ และ RP ไม่จำเป็นต้องเป็น Device จริง (Phantom RP ได้) — แต่ปิดท้ายด้วยประโยคตรงๆ ว่า
**"ไม่ลง Lab เต็มรูปแบบ (เหมาะกับ CCIE มากกว่า ENCOR)"** — Part นี้คือจุดที่หลักสูตรกลับมาทำตาม
สัญญานั้น ด้วย 3 เรื่องที่ Part 42 ยังไม่ได้อธิบาย: **(1)** กลไก DF Election ทีละขั้นตอนจริง
**(2)** วิธี Config Phantom RP ให้มี Redundancy จริง (ไม่ใช่แค่พูดถึงแนวคิด) **(3)** Config เต็ม
รูปแบบสำหรับ Use Case Many-to-Many จริง (Trading Floor)

### 821.2 DF (Designated Forwarder) Election — กลไกทีละขั้นตอน

ทุก **Link** (ไม่ใช่ทุก Router) ที่มี PIM Router มากกว่า 1 ตัวต่ออยู่ ต้องมี DF ของตัวเองต่อ 1 RP
Address — DF คือ Router เดียวที่ได้รับอนุญาตให้ Forward Traffic ขึ้น/ลง Link นั้นไปทาง RP Tree
เพื่อป้องกัน Loop (ทำหน้าที่แทน RPF Check ของ PIM-SM ปกติ เพราะ Bidir-PIM ไม่มี Source Tree ให้
RPF เทียบ)

**ตัวอย่างที่เห็นภาพชัดที่สุดใน Topology ของเรา**: VLAN 10 SALES (`10.10.10.0/24`) เป็น
**Multi-access Link ที่มี PIM Router 2 ตัวพร้อมกัน** — DIST-SW1 (`.2`) และ DIST-SW2 (`.3`) ตาม
HSRP Pair จาก [Part 18](part-018-fhrp.md) — นี่คือ Link ที่ต้องมี DF Election จริง (ต่างจาก
Point-to-point Link ที่มี PIM Neighbor แค่ 2 ตัวซึ่ง DF ตกเป็นของฝั่งที่ Metric ดีกว่าเสมอโดยไม่มี
อะไรซับซ้อน)

```
VLAN 10 SALES (Multi-access Segment)
        │
   ┌────┴────┐         ┌─────────┐
   │ DIST-SW1 │         │ DIST-SW2 │     ทั้งสองเป็น PIM Neighbor บน Vlan10 เดียวกัน
   │  .2      │         │  .3      │     ทั้งสองมีเส้นทาง (Unicast Route) ไปยัง Phantom RP
   └────┬────┘         └────┬────┘     คนละเส้นทาง (ผ่าน CORE-SW1 กับ CORE-SW2 ตามลำดับ)
        │  Cost ไป RP = 20        │  Cost ไป RP = 15
        └───────────┬───────────┘
                     ▼
         ต้องมี DF ตัวเดียวสำหรับ Link นี้ — DIST-SW2 ชนะ (Cost ต่ำกว่า)
```

### 821.3 State Machine ของ DF Election (Offer / Winner / Backoff / Lose)

```
เริ่มต้น: ทั้ง DIST-SW1 และ DIST-SW2 Boot ขึ้นพร้อมกัน ไม่มี DF บน Vlan10 เลย

Step 1 — OFFER
  DIST-SW1 ส่ง DF-Offer(Metric=20) ออก Vlan10
  DIST-SW2 ส่ง DF-Offer(Metric=15) ออก Vlan10
  (ทั้งคู่ประกาศ Metric ของตัวเองไปยัง RP Address พร้อมกัน — ยังไม่มีใครเป็น DF)

Step 2 — เปรียบเทียบ Metric
  DIST-SW1 เห็น Offer ของ DIST-SW2 (Metric=15) ดีกว่าตัวเอง (Metric=20)
       → DIST-SW1 หยุด Offer, เข้าสถานะ "Lose" (ยอมรับว่าไม่ชนะ)
  DIST-SW2 เห็น Offer ของ DIST-SW1 (Metric=20) แย่กว่าตัวเอง (Metric=15)
       → DIST-SW2 เข้าสถานะ "Backoff" (รอสักพักก่อนประกาศชนะ เผื่อมี Router Metric ดีกว่าเข้ามาอีก)

Step 3 — WINNER (หลัง Backoff Timer หมด ไม่มีใคร Offer แข่งอีก)
  DIST-SW2 ประกาศ DF-Winner(Metric=15) → กลายเป็น DF ตัวจริงของ Link นี้
  DIST-SW1 ยืนยันสถานะ "Lose" ต่อไป (ไม่ Forward Traffic เข้า/ออก Vlan10 สำหรับ Group นี้เลย
  ยกเว้นทำหน้าที่รับ Multicast จาก Vlan10 ไปส่งต่อ downstream อื่นถ้ามี)

Step 4 — Metric เปลี่ยน (เช่น Link CORE-SW2↔DIST-SW2 ล่ม ทำให้ DIST-SW2 Metric แย่ลงเป็น 30)
  DIST-SW1 เห็นว่า Metric ตัวเอง (20) ดีกว่า DF ปัจจุบัน (30) แล้ว
       → DIST-SW1 ส่ง Offer ใหม่ทันที → DIST-SW2 (DF เดิม) เห็น Offer ที่ดีกว่า
       → DIST-SW2 สละสถานะ DF, เข้า "Lose"
       → DIST-SW1 เข้า "Backoff" แล้ว "Winner" ตามลำดับเดิม (DF เปลี่ยนมืออัตโนมัติ)
```

| สถานะ | ความหมาย | Trigger ที่ทำให้เปลี่ยนสถานะ |
|---|---|---|
| **Offer** | Router ประกาศ Metric ของตัวเองไปยัง RP บน Link นี้ | Boot ใหม่ หรือเห็น DF ปัจจุบันมี Metric แย่กว่าตัวเอง |
| **Backoff** | Router ที่ Metric ดีที่สุด ณ ขณะนี้ รอ Timer สั้นๆ ก่อนประกาศชนะ (ป้องกัน Flap ถ้ามี Router อื่น Metric ดีกว่าเข้ามาอีก) | ชนะการเปรียบเทียบ Offer ทุกตัวบน Link ชั่วขณะ |
| **Winner** | DF ตัวจริงของ Link นี้ ส่ง Winner Message เป็นระยะเพื่อยืนยันสถานะ | Backoff Timer หมดโดยไม่มี Offer ที่ดีกว่าเข้ามาแข่ง |
| **Lose** | Router ที่ไม่ใช่ DF บน Link นี้ — เงียบ ไม่ Forward Traffic ขึ้น/ลง Link | เห็น Offer/Winner อื่นที่ Metric ดีกว่าตัวเองเสมอ |

> **จุดสำคัญสำหรับข้อสอบ CCIE**: DF Election **ทำงานแยกกันทุก Link และทุก RP Address** (ไม่ใช่
> ทำครั้งเดียวทั้ง Domain) — Router ตัวเดียวอาจเป็น DF บน Link หนึ่งแต่ Lose บน Link อื่นพร้อมกันได้
> เพราะ Metric ไปยัง RP วัดจาก **แต่ละ Link แยกกัน** (คนละ Interface คนละ Cost) — ต่างจาก RPF
> Check ของ PIM-SM ที่ตัดสินใจจาก Global Unicast Routing Table เพียงครั้งเดียวต่อ Source

### 821.4 Phantom RP — RP ที่ "ไม่มีตัวตนจริง" แต่ให้ Redundancy ได้

Bidir-PIM RP **ไม่ต้อง Process Data Plane เลย** (ต่างจาก PIM-SM ที่ RP ต้องรับ Register และ
(ในช่วงก่อน SPT Switchover) Forward Traffic จริง) — RP ทำหน้าที่แค่เป็น **"จุดอ้างอิงของ Tree"**
ให้ทุก Router คำนวณ Metric/DF Election เทียบกับ Address นี้ ดังนั้น **RP Address ไม่จำเป็นต้อง
ถูก Config เป็น IP จริงบน Interface ใดเลย** — เทคนิคที่ใช้กันทั่วไปคือ **Phantom RP**: ให้ Router
2 ตัว (เพื่อ Redundancy) Advertise Prefix ที่ "ครอบคลุม" RP Address เดียวกัน แต่ด้วย **Subnet Mask
ยาวไม่เท่ากัน** เพื่ออาศัย Longest-Prefix-Match ของ Unicast Routing ทำ Failover อัตโนมัติ

```
RP Address ที่ทุก Router ใน Domain Config เหมือนกัน:  10.255.199.1  (ไม่มีใครใช้ Address นี้จริง)

CORE-SW1 (Primary)                          CORE-SW2 (Backup)
Loopback199: 10.255.199.1/30                Loopback199: 10.255.199.2/24
   → Advertise 10.255.199.0/30 (ยาวกว่า)        → Advertise 10.255.199.0/24 (สั้นกว่า)
   → ครอบคลุม .1 ด้วยแต่ "ยาวกว่า" จึงชนะ           → ครอบคลุม .1 ด้วยแต่ "สั้นกว่า" จึงเป็น Backup
     Longest-Prefix-Match ตราบใดที่ CORE-SW1
     ยัง Advertise Prefix /30 อยู่

สถานะปกติ (CORE-SW1 Up)              CORE-SW1 ล่ม (Loopback199 Down)
ทุก Router มองเห็น 10.255.199.1        /30 หายจาก OSPF LSDB ทันที
ผ่าน /30 (Longest Match ชนะ)          เหลือแค่ /24 จาก CORE-SW2
→ Metric คำนวณผ่านเส้นทางไป CORE-SW1  → ทุก Router Re-converge อัตโนมัติผ่าน CORE-SW2
                                        (ไม่ต้องแก้ ip pim rp-address สักตัวอักษรเดียวทั้ง Domain)
```

### 821.5 Config เต็มรูปแบบ: Bidir-PIM สำหรับ Trading Floor Many-to-Many

**Use Case**: ทีม Trading Desk (Client Terminal อยู่ใน **VLAN 10 SALES**) ต้องส่ง/รับ Order Update
ของกันและกันกับ **Matching Engine Application** (อยู่ใน **VLAN 30 SERVERS** ตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) แบบ Real-time — สมมติมี Trader Terminal ~300
เครื่องกระจายทั้ง VLAN 10 ทุกคนเป็นทั้ง Publisher และ Subscriber ของ Group `239.9.9.9` พร้อมกัน
(**คนละ Group กับ Market-Data SSM `232.50.1.1`** ที่เป็น One-to-Many Feed จาก Exchange ซึ่ง
[Part 75 Step 741](part-075-advanced-multicast-enterprise.md#step-741--multicast-scaling-ระดับ-enterprise)
นับรวมไว้แล้ว — Group นี้เป็นการสื่อสารระหว่าง Trader **ภายในบริษัทเท่านั้น**)

```
! ===== ACL กำหนด Group ที่จะทำ Bidir (แยกจาก Group PIM-SM ปกติอื่นๆ) =====
CORE-SW1(config)# ip access-list standard BIDIR-TRADING-GROUP
CORE-SW1(config-std-nacl)# permit 239.9.9.9
CORE-SW1(config-std-nacl)# exit

! ===== Phantom RP: CORE-SW1 (Primary, Prefix ยาวกว่า) =====
CORE-SW1(config)# interface Loopback199
CORE-SW1(config-if)# description ** Phantom RP - Bidir-PIM Trading Floor (Primary) **
CORE-SW1(config-if)# ip address 10.255.199.1 255.255.255.252
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# network 10.255.199.0 0.0.0.3 area 0
CORE-SW1(config-router)# exit
CORE-SW1(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
CORE-SW1(config)# ip pim bidir-enable

! ===== Phantom RP: CORE-SW2 (Backup, Prefix สั้นกว่า) =====
CORE-SW2(config)# interface Loopback199
CORE-SW2(config-if)# description ** Phantom RP - Bidir-PIM Trading Floor (Backup) **
CORE-SW2(config-if)# ip address 10.255.199.2 255.255.255.0
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 10.255.199.0 0.0.0.255 area 0
CORE-SW2(config-router)# exit
CORE-SW2(config)# ip access-list standard BIDIR-TRADING-GROUP
CORE-SW2(config-std-nacl)# permit 239.9.9.9
CORE-SW2(config-std-nacl)# exit
CORE-SW2(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
CORE-SW2(config)# ip pim bidir-enable

! ===== DIST-SW1/DIST-SW2 (VLAN 10 SALES — Trading Desk) และ DIST-SW3/DIST-SW4 (VLAN 30 SERVERS — Matching Engine) =====
DIST-SW1(config)# ip access-list standard BIDIR-TRADING-GROUP
DIST-SW1(config-std-nacl)# permit 239.9.9.9
DIST-SW1(config-std-nacl)# exit
DIST-SW1(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
DIST-SW1(config)# ip pim bidir-enable
! (ทำเหมือนกันบน DIST-SW2, DIST-SW3, DIST-SW4 — ACL + rp-address ต้องเหมือนกันทุกตัวอักษรทุกตัว
!  เหมือนกฎ Static RP ปกติจาก Part 41 Step 407, ต่างกันแค่เพิ่ม keyword "bidir")
```

| คำสั่งที่เพิ่มใหม่ในหัวข้อนี้ | ความหมาย |
|---|---|
| `ip pim rp-address <RP> <acl> bidir` | ระบุว่า RP นี้ทำงานแบบ Bidirectional สำหรับ Group ที่ ACL `permit` เท่านั้น (Group อื่นยัง PIM-SM ปกติได้ในเวลาเดียวกันบน RP Address คนละตัว) |
| `ip pim bidir-enable` | **Global Command บังคับ** — ต้องเปิดก่อน Router จะยอมรับ Bidir-PIM RP เลย (ถ้าลืมคำสั่งนี้ Router จะปฏิเสธ RP Bidir แม้ตั้ง `ip pim rp-address ... bidir` ไว้แล้วก็ตาม — จุด Troubleshoot ที่พบบ่อยที่สุดของ Bidir-PIM) |

### 821.6 Verification

```
CORE-SW1# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is a candidate RP (v2)

Group(s) 239.9.9.9/32
  RP 10.255.199.1 (?), v2, bidir
    Info source: 10.255.199.1 (?), via static

CORE-SW1# show ip pim interface df
* implies this system is the DF
Interface                RP Address        DF Winner         Metric       Uptime
Vlan10                    10.255.199.1      10.10.10.3(*)     [110/15]     01:12:40
Port-channel1             10.255.199.1      10.255.0.1(*)     [110/0]      02:40:11

! (แสดงตัวอย่างจาก DIST-SW2 มุมมองของตัวเอง — DIST-SW2 คือ DF ของ Vlan10 ตามที่คำนวณใน 821.3)

CORE-SW1# show ip mroute 239.9.9.9
(*, 239.9.9.9), 00:20:11/00:02:59, RP 10.255.199.1, flags: BC
  Bidir-Upstream: Port-channel1, RPF nbr 10.255.0.2
  Outgoing interface list:
    Port-channel1, Forward/Sparse, 00:20:11/00:02:59

! Flag "B" = Bidir Group — ไม่มี Flag "T" (SPT) หรือ "F" (Register) ปรากฏเลย ยืนยันว่าไม่มี
! (S,G) หรือ Register Process เกิดขึ้นตามที่ตารางใน 821.1/Part 42 อธิบายไว้
```

---

## Step 822 — Multicast VPN (MVPN) Rosen GRE Model: Config เต็มรูปแบบ

### 822.1 ต่อยอดจาก Part 75 Step 742 ที่ให้แค่ Concept

[Part 75 Step 742](part-075-advanced-multicast-enterprise.md#step-742--multicast-vpn-mvpn-default-mdt--data-mdt)
อธิบายแนวคิด MVPN Rosen GRE ไว้แล้ว (Default MDT ที่มีเสมอ + Data MDT ที่สร้างเมื่อ Traffic เกิน
Threshold) พร้อม Config ตัวอย่าง 6 บรรทัดแบบ Conceptual — Part นี้จะ **ลง Lab เต็มรูปแบบจริง** โดย
ต่อยอดจาก **VRF `GUEST`** ที่สร้างไว้แล้วใน [Part 51](part-051-vrf-lite-gre-ipsec.md) ให้กลายเป็น
MVPN ข้าม 2 Site จริง

> **ข้อจำกัดของ Lab นี้ที่ต้องเข้าใจก่อน**: MVPN Rosen แบบดั้งเดิมมักอธิบายในบริบท MPLS L3VPN
> (PE-CE ผ่าน P-Network) แต่ **GRE-based MDT ไม่จำเป็นต้องมี MPLS เลย** — สิ่งที่จำเป็นจริงๆ มีแค่
> **PIM-SM Domain ในระดับ Global/Provider Table ที่ PE ทุกตัวเข้าถึงกันได้** (ในหลักสูตรนี้คือ OSPF
> Backbone ที่มีมาตั้งแต่ Part 11-12) — Lab นี้จึงให้ **WAN-EDGE-1** ทำหน้าที่ PE ของ Site "HQ
> Campus" และ **WAN-EDGE-2** ทำหน้าที่ PE ของ Site "Regional DC Edge" (สมมติสำหรับ Lab นี้ ต่างจาก
> Part 51 ที่ VRF `GUEST` อยู่จริงที่ DIST-SW3 — ทำให้เห็นภาพ 2-PE MVPN ชัดที่สุดโดยไม่ต้องพะวง
> เรื่อง Hop ระหว่างทาง) — Full MPLS L3VPN Integration จะเรียนจริงใน
> [Part 86-87](part-086-mpls-l3vpn-fundamentals.md)

### 822.2 Topology ของ Lab นี้

```
        VRF GUEST (Customer Multicast Domain — Guest Portal Ad Video)
        ┌─────────────────────┐                    ┌─────────────────────┐
        │  WAN-EDGE-1 (PE-HQ)  │                    │ WAN-EDGE-2 (PE-DC)   │
        │  Lo0: 1.1.1.21       │                    │ Lo0: 1.1.1.22        │
        └──────────┬──────────┘                    └──────────┬──────────┘
                   │  Global Table (Provider PIM Domain — ยืม OSPF Backbone เดิม)
                   │  RP ของ Provider = Anycast RP 1.1.1.100 (CORE-SW1/2, จาก Part 42/75)
                   └───────────────── GRE MDT Tunnel (Default MDT 239.192.1.1) ─────────────────┘
                            (P-Router/CORE-SW เห็นแค่ 239.192.1.1 ธรรมดา ไม่รู้จัก VRF GUEST เลย)
```

### 822.3 Config เต็มรูปแบบ

```
! ===== WAN-EDGE-1 (PE-HQ) =====
WAN-EDGE-1(config)# vrf definition GUEST
WAN-EDGE-1(config-vrf)# rd 65001:40
WAN-EDGE-1(config-vrf)# address-family ipv4
WAN-EDGE-1(config-vrf-af)# route-target export 65001:40
WAN-EDGE-1(config-vrf-af)# route-target import 65001:40
WAN-EDGE-1(config-vrf-af)# mdt default 239.192.1.1
WAN-EDGE-1(config-vrf-af)# mdt data 232.1.1.0 0.0.0.255 threshold 500
WAN-EDGE-1(config-vrf-af)# exit-address-family
WAN-EDGE-1(config-vrf)# exit

WAN-EDGE-1(config)# ip multicast-routing                 ! Provider (Global) PIM Domain
WAN-EDGE-1(config)# ip multicast-routing vrf GUEST        ! Customer PIM Domain (คนละ Domain กับ Global โดยสิ้นเชิง)
WAN-EDGE-1(config)# ip pim rp-address 1.1.1.100 10        ! Provider RP — Anycast เดิมจาก Part 42/75
WAN-EDGE-1(config)# ip pim vrf GUEST rp-address 1.1.1.21  ! Customer RP — ใช้ WAN-EDGE-1 เองสำหรับ Lab นี้
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1.40
WAN-EDGE-1(config-subif)# description ** Customer-facing: Guest Wi-Fi Ad Video Source Segment **
WAN-EDGE-1(config-subif)# vrf forwarding GUEST
WAN-EDGE-1(config-subif)# ip pim sparse-mode
WAN-EDGE-1(config-subif)# exit
WAN-EDGE-1(config)# interface Port-channel2               ! ทางไป Core/WAN-EDGE-2 (Provider-facing)
WAN-EDGE-1(config-if)# ip pim sparse-mode
WAN-EDGE-1(config-if)# exit

! ===== WAN-EDGE-2 (PE-DC) — Config คู่กันทุกตัวอักษร ยกเว้น RP ฝั่ง Customer =====
WAN-EDGE-2(config)# vrf definition GUEST
WAN-EDGE-2(config-vrf)# rd 65001:40
WAN-EDGE-2(config-vrf)# address-family ipv4
WAN-EDGE-2(config-vrf-af)# route-target export 65001:40
WAN-EDGE-2(config-vrf-af)# route-target import 65001:40
WAN-EDGE-2(config-vrf-af)# mdt default 239.192.1.1
WAN-EDGE-2(config-vrf-af)# mdt data 232.1.1.0 0.0.0.255 threshold 500
WAN-EDGE-2(config-vrf-af)# exit-address-family
WAN-EDGE-2(config-vrf)# exit
WAN-EDGE-2(config)# ip multicast-routing
WAN-EDGE-2(config)# ip multicast-routing vrf GUEST
WAN-EDGE-2(config)# ip pim rp-address 1.1.1.100 10
WAN-EDGE-2(config)# ip pim vrf GUEST rp-address 1.1.1.21   ! ชี้ไปยัง Customer RP เดียวกับ PE-HQ เสมอ
WAN-EDGE-2(config)# interface GigabitEthernet0/0/2.40
WAN-EDGE-2(config-subif)# vrf forwarding GUEST
WAN-EDGE-2(config-subif)# ip pim sparse-mode
WAN-EDGE-2(config-subif)# exit
WAN-EDGE-2(config)# interface Port-channel2
WAN-EDGE-2(config-if)# ip pim sparse-mode
```

### 822.4 Verification — เห็น GRE MDT Tunnel ที่ถูกสร้างอัตโนมัติ

```
WAN-EDGE-1# show ip pim vrf GUEST mdt interface
Flags: T - Threshold set, S - Statically encoded, D - Data MDT interface
Interface   Source            Group             Flags       VRF
Tunnel0     1.1.1.21          239.192.1.1                    GUEST
                              <-- Tunnel0 คือ GRE MDT ที่ IOS สร้างขึ้นอัตโนมัติจากคำสั่ง
                                  "mdt default" — ไม่ปรากฏใน running-config เพราะเป็น Virtual
                                  Interface ที่ระบบสร้างเอง

WAN-EDGE-1# show ip pim vrf GUEST mdt send
MDT-data send list for VRF: GUEST
  (source, group)                     MDT-data group/num   ref_count
  (10.10.40.30, 239.20.1.1)           232.1.1.1/1           1
                                       <-- Data MDT ถูกสร้างจริงเมื่อ Guest Ad Video Server
                                           (10.10.40.30) ส่ง Traffic เกิน 500 kbps ตามที่ Config

WAN-EDGE-2# show ip mroute vrf GUEST
(*, 239.20.1.1), 00:05:40/00:02:47, RP 1.1.1.21, flags: SJC
  Incoming interface: Tunnel0, RPF nbr 1.1.1.21
  Outgoing interface list:
    GigabitEthernet0/0/2.40, Forward/Sparse, 00:05:40/00:02:47

WAN-EDGE-2# show ip mroute 239.192.1.1
(1.1.1.21, 239.192.1.1), 00:12:03/00:02:58, flags: T
  Incoming interface: Port-channel2, RPF nbr 10.10.254.9
  Outgoing interface list:
    Tunnel0-decapsulator     <-- Global (Provider) Table เห็นแค่ Provider Group ธรรมดา
                                  ไม่รู้จัก 239.20.1.1 ของ Customer เลยแม้แต่นิดเดียว ตามที่
                                  Part 75 Step 742.3 อธิบายไว้ตั้งแต่ต้น
```

> **สรุป**: `mdt default`/`mdt data` สร้าง **GRE Tunnel เสมือนอัตโนมัติ** (`Tunnel0` ในตัวอย่างนี้)
> ที่ Encapsulate Customer Multicast Packet ทั้งหมดของ VRF `GUEST` ไว้ข้างใน แล้วส่งผ่าน Provider
> PIM-SM Domain ปกติ (RP `1.1.1.100` เดิม) — Customer PIM Domain (RP `1.1.1.21`) และ Provider PIM
> Domain (RP `1.1.1.100`) เป็นคนละ Instance กันโดยสิ้นเชิง เหมือนที่ VRF-Lite ปกติแยก Routing Table
> — ต่างกันแค่ตรงนี้มี "PIM Tunnel" คั่นกลางแทนที่จะเป็น Interface จริง

---

## Step 823 — mVPN BGP-based (MDT-SAFI / Next-Generation MVPN)

### 823.1 ปัญหาของ Rosen GRE ที่ BGP-based MVPN แก้

Rosen GRE Model (Step 822) มีข้อจำกัดที่ยิ่งชัดเจนขึ้นเมื่อ Scale ใหญ่ขึ้น: **ต้องมี PIM-SM Domain
เต็มรูปแบบวิ่งอยู่ใน P-Network** (Provider ต้องดูแล PIM Neighbor/RP ของตัวเองแยกจาก Customer VPN
Routing ที่ใช้ MP-BGP อยู่แล้ว) — Service Provider ยุคใหม่ที่ Migrate ทุกอย่างไปใช้ MP-BGP ล้วนๆ
(Unicast VPNv4/VPNv6 อยู่แล้ว) จึงมองว่า **การต้องเปิด PIM ควบคู่กับ MP-BGP ในระดับ Core เป็นภาระ
Operation ซ้ำซ้อน** — **NG-MVPN (RFC 6513/6514)** แก้ปัญหานี้ด้วยการเปลี่ยนวิธี Signal Membership
Information จาก "PIM Join ใน P-Network" ไปเป็น **BGP Update ธรรมดา** (ผ่าน Address-Family ใหม่
คือ **MDT-SAFI** และ MVPN NLRI Type 1-7)

### 823.2 เปรียบเทียบ Rosen GRE vs BGP-based (NG-MVPN)

| มิติ | Rosen GRE (Step 822) | BGP-based / NG-MVPN |
|---|---|---|
| Signaling Membership | **PIM** ใน P-Network (PE ทุกตัว Join Provider Group ผ่าน PIM Join จริง) | **MP-BGP** (MDT-SAFI, AFI 1/SAFI 66 หรือ MVPN NLRI Type 1-7) — ไม่มี PIM ใน Core เลย |
| P-Tunnel ที่ Carry Traffic จริง | GRE Tunnel (Point-to-multipoint ผ่าน Provider Multicast Tree) | เลือกได้หลายแบบ: MPLS P2MP/MP2MP LSP (mLDP), Ingress Replication (Unicast ซ้ำที่ Ingress PE), หรือยังใช้ GRE ก็ได้ (BGP แค่เปลี่ยนวิธี Signal ไม่ได้บังคับ P-Tunnel Type) |
| ต้องมี PIM Domain ใน P-Network หรือไม่ | **ต้องมี** | **ไม่ต้อง** ถ้าเลือก P-Tunnel แบบ mLDP/Ingress Replication |
| Convergence เมื่อ PE เพิ่ม/ลด | ช้ากว่า — ต้องรอ PIM Join/Prune Converge | เร็วกว่า — ใช้ BGP Update/Withdraw แบบเดียวกับ VPNv4 Route ปกติ |
| Scale (จำนวน VRF/PE) | จำกัดกว่า — ทุก VRF ต้องมี Provider Group ของตัวเอง (Data MDT Pool มีขนาดจำกัด) | Scale ดีกว่า — ใช้ BGP Route Reflector Infrastructure เดิมที่มีอยู่แล้วสำหรับ VPNv4 |
| ปัจจุบันใช้งานจริงมากกว่าแบบไหน | Legacy — ยังเจอใน Network เก่า/Design ง่ายๆ | **แบบที่ Cisco/Juniper แนะนำสำหรับ Deployment ใหม่** ตั้งแต่ปลายทศวรรษ 2010 เป็นต้นมา |

### 823.3 Config เชิง Concept (ไม่ลง Lab เต็ม — เหตุผลในหมายเหตุท้าย Step)

```
! ===== เปิด MDT-SAFI บน BGP Session ระหว่าง PE (ตัวอย่าง WAN-EDGE-1 ↔ WAN-EDGE-2) =====
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# address-family ipv4 mdt
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 activate
WAN-EDGE-1(config-router-af)# neighbor 1.1.1.22 send-community extended
WAN-EDGE-1(config-router-af)# exit-address-family

! ===== VRF GUEST: เปลี่ยนวิธี Auto-Discovery จาก PIM (Rosen) ไปเป็น BGP =====
WAN-EDGE-1(config)# vrf definition GUEST
WAN-EDGE-1(config-vrf)# address-family ipv4
WAN-EDGE-1(config-vrf-af)# mdt auto-discovery bgp
WAN-EDGE-1(config-vrf-af)# mdt default mpls mldp        ! เลือก mLDP เป็น P-Tunnel แทน GRE
WAN-EDGE-1(config-vrf-af)# exit-address-family
```

> **สรุปสำหรับ CCIE Enterprise Infrastructure**: NG-MVPN เป็นหัวข้อที่ **อยู่ในเนื้อหา SP/MPLS
> Track เข้มข้นกว่า Enterprise Track มาก** (ต้องมี mLDP หรือ RSVP-TE P2MP รองรับในระดับ P-Router
> ซึ่งไม่ใช่ Scope ของหลักสูตรนี้จนกว่าจะถึง [Part 86-87](part-086-mpls-l3vpn-fundamentals.md))
> ข้อสอบระดับ Enterprise Infrastructure จึงมักถามแค่ **"BGP-based MVPN ต่างจาก Rosen GRE
> อย่างไร"** (คำตอบ: เปลี่ยน Signaling จาก PIM เป็น BGP, ไม่ต้องมี PIM Domain ใน Core) และ
> **"ทำไม Enterprise ใหม่มักเลือกแบบนี้"** (Scale/Convergence ดีกว่า, ใช้ BGP Infrastructure
> เดิมที่มีอยู่แล้ว) — หลักสูตรนี้ให้ภาพเปรียบเทียบที่เพียงพอตรงนี้โดยไม่ลง Lab เต็มเพราะต้องมี
> mLDP/RSVP-TE ที่ยังไม่ได้เรียนจนกว่าจะถึง Part 86-87

---

## Step 824 — IPv6 Multicast: MLD และ PIM6

### 824.1 ทวนพื้นฐาน: MLD คือ IGMP เวอร์ชัน IPv6

[Part 41 Step 403](part-041-multicast-fundamentals.md#step-403--igmp-internet-group-management-protocol)
สอน IGMP สำหรับ IPv4 — IPv6 ใช้กลไกเดียวกันทุกประการแต่เปลี่ยนชื่อเป็น **MLD (Multicast Listener
Discovery)** และเปลี่ยนจาก IPv4 Packet ไปเป็น **ICMPv6 Message Type** (ต่างจาก IGMP ที่เป็น IP
Protocol Number 2 ของตัวเอง) — ทำงานเป็น Host↔Router (Last-Hop) เหมือนกันเป๊ะ

| คุณสมบัติ | IGMP (IPv4, Part 41) | MLD (IPv6) |
|---|---|---|
| Transport | IP Protocol 2 (ของตัวเอง) | ICMPv6 (Type 130-132 สำหรับ v1, 143 สำหรับ v2) |
| เวอร์ชันที่เทียบเท่ากัน | IGMPv1 | MLDv1 |
| เวอร์ชันที่เทียบเท่ากัน | IGMPv2 (มี Leave Group) | MLDv1 (มี Done Message เทียบเท่า Leave) |
| เวอร์ชันที่เทียบเท่ากัน | IGMPv3 (Source Filtering, รองรับ SSM) | **MLDv2** (Source Filtering, รองรับ SSM) |
| Default บน Cisco IOS | IGMPv2 | **MLDv2** |
| Snooping บน Switch | `ip igmp snooping` | `ipv6 mld snooping` |
| Query ไปที่ | Group Address หรือ `224.0.0.1` | `ff02::1` (All-Nodes Link-Local) |

### 824.2 PIM6 — PIM สำหรับ IPv6 (`ipv6 pim`)

PIM6 ใช้แนวคิดเดียวกับ PIM ทุกอย่างจาก [Part 41 Step 404-408](part-041-multicast-fundamentals.md#step-404--pim-คืออะไร-ทำไมต้องมี-multicast-routing-protocol-แยกจาก-igmp)
(RPF Check, `(*,G)`/`(S,G)`, Sparse-mode Explicit Join, RP) — เปลี่ยนแค่ Syntax คำสั่งจาก `ip pim`
เป็น `ipv6 pim` และ Hello ไปที่ `ff02::d` (เทียบเท่า `224.0.0.13` ของ IPv4) **ข้อแตกต่างที่สำคัญ
ที่สุด**: IOS **ไม่รองรับ PIM Dense Mode สำหรับ IPv6 เลย** (PIM6 ต้องเป็น Sparse-mode หรือ
SSM เท่านั้น — ตัดปัญหา Flood-and-Prune จาก [Part 41 Step 405](part-041-multicast-fundamentals.md#step-405--pim-dense-mode-pim-dm-flood-and-prune-model)
ออกไปตั้งแต่ต้นสำหรับ IPv6)

### 824.3 Config เต็มรูปแบบ: เปิด IPv6 Multicast ให้ STREAM-SRV คู่กับ IPv4 เดิม

ต่อยอดจาก **Dual-Stack Lab ของ [Part 8](part-008-ipv6-fundamentals.md)** (ULA Prefix
`fd00:10:10:X0::/64` ตาม VLAN) และ **STREAM-SRV** จาก [Part 41](part-041-multicast-fundamentals.md)
— สมมติ STREAM-SRV มี IPv6 Address `fd00:10:10:30::30` (ตามรูปแบบ ULA ของ VLAN 30 SERVERS,
Host ID `::30` ให้ตรงกับ Octet สุดท้ายของ IPv4 `10.10.30.30` เพื่อจดจำง่าย) และต้องการกระจาย
Video Stream ตัวเดียวกันผ่าน IPv6 Group `ff05::239:1:1` (Site-local Scope, ตั้งชื่อ Group ให้จำง่าย
โดยอ้างอิงเลข `239.1.1.1` เดิม)

> **หมายเหตุเรื่อง Loopback0 IPv6**: [00-ip-address-plan.md](00-ip-address-plan.md) ไม่ได้กำหนด
> IPv6 Address ของ Loopback0 ไว้ (มีแค่ VLAN ULA Prefix) — Part นี้จึงกำหนดส่วนขยายเพิ่มตาม
> Convention **`fd00:<Loopback0 IPv4 แต่ละ Octet>::<Octet สุดท้าย>`** เช่น CORE-SW1
> (`1.1.1.1` → `fd00:1:1:1::1`) เพื่อใช้เป็น RP Address ของ PIM6 เท่านั้น (Convention นี้ใช้เฉพาะ
> Part นี้ ไม่กระทบ IP Plan หลัก)

```
! ===== CORE-SW1 (Static RP ของ PIM6 — ใช้ Static RP ธรรมดาเหมือน Part 41 Step 407 ไม่ต้องซับซ้อน) =====
CORE-SW1(config)# ipv6 multicast-routing
CORE-SW1(config)# interface Loopback0
CORE-SW1(config-if)# ipv6 address fd00:1:1:1::1/128
CORE-SW1(config-if)# ipv6 pim
CORE-SW1(config-if)# exit
CORE-SW1(config)# ipv6 pim rp-address fd00:1:1:1::1
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ipv6 pim
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# ipv6 pim

! ===== CORE-SW2 =====
CORE-SW2(config)# ipv6 multicast-routing
CORE-SW2(config)# ipv6 pim rp-address fd00:1:1:1::1
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# ipv6 pim
CORE-SW2(config-if)# exit
CORE-SW2(config)# interface TenGigabitEthernet1/0/1
CORE-SW2(config-if)# ipv6 pim

! ===== DIST-SW3 (VLAN 30 SERVERS — ต้นทาง STREAM-SRV IPv6) =====
DIST-SW3(config)# ipv6 multicast-routing distributed
DIST-SW3(config)# ipv6 pim rp-address fd00:1:1:1::1
DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# ipv6 pim
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ipv6 mld version 2                  ! ระบุ MLDv2 ให้ชัดเจน (Default อยู่แล้วแต่ Best Practice ต้อง Config ให้เห็นเสมอตามแนวทาง Part 41 Step 403)
DIST-SW3(config-if)# ipv6 pim

! ===== DIST-SW1 (VLAN 10 SALES) และ DIST-SW4 (VLAN 40 WIFI) — Receiver =====
DIST-SW1(config)# ipv6 multicast-routing distributed
DIST-SW1(config)# ipv6 pim rp-address fd00:1:1:1::1
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ipv6 mld version 2
DIST-SW1(config-if)# ipv6 pim

DIST-SW4(config)# ipv6 multicast-routing distributed
DIST-SW4(config)# ipv6 pim rp-address fd00:1:1:1::1
DIST-SW4(config)# interface Vlan40
DIST-SW4(config-if)# ipv6 mld version 2
DIST-SW4(config-if)# ipv6 pim
```

### 824.4 Verification

```
DIST-SW1# show ipv6 mld groups
MLD Connected Group Membership
Group Address                             Interface     Uptime    Expires
ff05::239:1:1                              Vlan10        00:03:20  00:02:41

CORE-SW1# show ipv6 pim neighbor
PIM Neighbor Table
Neighbor Address                         Interface              Uptime/Expires  Ver  DR Prio/Mode
FE80::A8BB:CCFF:FE00:1200                  Port-channel1          00:45:10/00:01:32  v2   1 / DR

DIST-SW1# show ipv6 mroute ff05::239:1:1
(*, FF05::239:1:1), 00:03:22/00:02:37, RP fd00:1:1:1::1, flags: SJC
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr FE80::A8BB:CCFF:FE00:1100
  Outgoing interface list:
    Vlan10, Forward/Sparse, 00:03:22/00:02:37

! เห็น (*, G) เกิดขึ้นคู่กันสมบูรณ์กับ (*, 239.1.1.1) ของ IPv4 เดิม — Dual-Stack Multicast
! ทำงานพร้อมกันได้จริงโดยไม่ต้องเลือกอย่างใดอย่างหนึ่ง ตรงกับหลักการ Dual-Stack ของ Part 8
```

---

## Step 825 — MSDP SA-filter: กรอง (S,G) ระดับ Source เพื่อความปลอดภัย

### 825.1 ช่องว่างที่ MSDP Mesh-group (Part 75) ไม่ได้ปิด

[Part 75 Step 748](part-075-advanced-multicast-enterprise.md#step-748--multicast-high-availability-anycast-rp-failure-ระดับ-scale-และ-msdp-mesh-group-ป้องกัน-sa-loop)
ตั้ง `ip msdp mesh-group HQ-DC-MESH` ระหว่าง Anycast RP คู่ HQ (`1.1.1.1`/`1.1.1.2`) กับ DC
(`2.2.2.1`/`2.2.2.2`) เพื่อป้องกัน **SA Message Loop จาก Topology Full-mesh** — แต่ Mesh-group
**ไม่ได้ตรวจสอบเนื้อหาของ SA เลยว่า Source ที่ประกาศมานั้น "ควรมีสิทธิ์" ประกาศ Group นั้นหรือไม่**
— ถ้ามีเครื่องแปลกปลอมใน VLAN 40 WIFI Guest (เช่น `10.10.40.99`) เริ่มส่ง Traffic ไปที่ Group
All-Hands (`239.1.1.1`) หลอกเป็น STREAM-SRV, Local RP (CORE-SW1/2) จะ **Register Source นี้เข้า
Domain โดยไม่มีการตรวจสอบ** แล้ว MSDP จะ **กระจาย SA Message ของ Source ปลอมนี้ไปทั้ง Regional
DC และ Branch ผ่าน Mesh-group ที่ตั้งไว้** ทันที — Mesh-group แก้ปัญหา Loop ได้ แต่ไม่แก้ปัญหา
"ใครมีสิทธิ์เป็น Source ของ Group ไหน" เลย

```
ก่อน Config SA-filter                              หลัง Config SA-filter

Rogue Laptop (10.10.40.99, VLAN 40)                Rogue Laptop (10.10.40.99, VLAN 40)
       │ ส่ง Traffic ไป 239.1.1.1                          │ ส่ง Traffic ไป 239.1.1.1
       ▼                                                   ▼
   CORE-SW1 (RP) Register Source ปลอม                  CORE-SW1 (RP) Register Source ปลอม
       │                                                   │ MSDP ตรวจ SA-filter (in) ก่อน
       ▼ MSDP SA กระจายไปทุก Mesh-group Peer                ▼ ACL ไม่ permit (10.10.40.99, 239.1.1.1)
   DC-CORE-1/2, Branch เห็น Source ปลอมทันที             ❌ SA ถูก Drop ที่ต้นทาง ไม่กระจายออกไปเลย
   (Traffic อาจไหลไปถึงจริงถ้ามี Receiver Join)          Regional DC/Branch ไม่เห็น Source นี้เลย
```

### 825.2 คำสั่งหลัก: `ip msdp sa-filter`

```
Router(config)# ip msdp sa-filter {in | out} <peer-address> [list <acl>] [route-map <name>]
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| `in` | กรอง SA Message ที่ **รับเข้ามา** จาก Peer นี้ ก่อนบันทึกเข้า SA-Cache ของตัวเอง |
| `out` | กรอง SA Message ที่ **จะส่งออกไป** ให้ Peer นี้ (ควบคุมทิศทางตรงข้าม) |
| `list <acl>` | Extended ACL ที่ Match แบบ `permit <source> <group>` — SA ที่ตรง `permit` ผ่านได้, ตรง `deny` (หรือ Implicit Deny) ถูกทิ้ง |
| ระดับความละเอียด | ทำงานที่ **ระดับ (Source, Group) จริง** — ละเอียดกว่า `ip multicast boundary` (Part 75 Step 744 กรองแค่ระดับ Group ไม่สนใจ Source) และละเอียดกว่า Mesh-group (คุม Topology ไม่ใช่เนื้อหา) |

### 825.3 Config เต็มรูปแบบ: อนุญาตเฉพาะ STREAM-SRV เป็น Source ของ All-Hands

```
! ===== CORE-SW1: จำกัด Source ที่ถูกต้องของแต่ละ Group =====
CORE-SW1(config)# ip access-list extended MSDP-VALID-SOURCES
CORE-SW1(config-ext-nacl)# remark ==== All-Hands: อนุญาตเฉพาะ STREAM-SRV จริงเท่านั้น ====
CORE-SW1(config-ext-nacl)# permit ip host 10.10.30.30 host 239.1.1.1
CORE-SW1(config-ext-nacl)# remark ==== Group อื่นที่รู้จักแล้วและถูกต้อง (Signage/Training ต่ออาคาร) ====
CORE-SW1(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 239.50.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 239.60.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# remark ==== ปิดกั้นทุก Source อื่นที่ไม่ได้อยู่ใน VLAN 30 SERVERS โดยเด็ดขาด ====
CORE-SW1(config-ext-nacl)# deny   ip any any
CORE-SW1(config-ext-nacl)# exit

! ===== Apply เข้ากับทุก MSDP Peer (ทั้ง Anycast Pair ภายใน HQ และ Mesh-group ไป DC) =====
CORE-SW1(config)# ip msdp sa-filter in 1.1.1.2 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter in 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter in 2.2.2.2 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter out 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter out 2.2.2.2 list MSDP-VALID-SOURCES

! ===== CORE-SW2: Config เดียวกันทุกตัวอักษร (ต้อง Sync กันเสมอเหมือน Mesh-group) =====
CORE-SW2(config)# ip access-list extended MSDP-VALID-SOURCES
CORE-SW2(config-ext-nacl)# permit ip host 10.10.30.30 host 239.1.1.1
CORE-SW2(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 239.50.0.0 0.0.255.255
CORE-SW2(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 239.60.0.0 0.0.255.255
CORE-SW2(config-ext-nacl)# deny   ip any any
CORE-SW2(config-ext-nacl)# exit
CORE-SW2(config)# ip msdp sa-filter in 1.1.1.1 list MSDP-VALID-SOURCES
CORE-SW2(config)# ip msdp sa-filter in 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW2(config)# ip msdp sa-filter in 2.2.2.2 list MSDP-VALID-SOURCES
CORE-SW2(config)# ip msdp sa-filter out 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW2(config)# ip msdp sa-filter out 2.2.2.2 list MSDP-VALID-SOURCES
```

### 825.4 Verification: ก่อน/หลัง Config

```
! ก่อน Config SA-filter — Rogue Source หลุดเข้า SA-Cache จริง
CORE-SW1# show ip msdp sa-cache 239.1.1.1
MSDP Source-Active Cache - 2 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.100, MBGP/AS 65001, 00:20:11/00:05:32
(10.10.40.99, 239.1.1.1), RP 1.1.1.100, MBGP/AS 65001, 00:00:45/00:05:58   <-- Rogue Source!

! หลัง Config SA-filter — Rogue Source ถูกกรองตั้งแต่ต้น ไม่เข้า Cache เลย
CORE-SW1# show ip msdp sa-cache 239.1.1.1
MSDP Source-Active Cache - 1 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.100, MBGP/AS 65001, 00:00:08/00:05:59

CORE-SW1# debug ip msdp
MSDP0: Received SA (10.10.40.99, 239.1.1.1) length 12 from peer 1.1.1.2
MSDP0: SA (10.10.40.99, 239.1.1.1) filtered by MSDP-VALID-SOURCES (in) - discarded
```

---

## Step 826 — Multicast Rate-Limiting ต่อ Source

### 826.1 ปัญหาที่ยังไม่มี Feature ไหนใน Part 75 ป้องกันได้

[Part 75 Step 747](part-075-advanced-multicast-enterprise.md#step-747--igmpmld-scaling-igmpv3mldv2-source-filtering-และ-ip-igmp-limit-ป้องกัน-router-ล้น)
ใช้ `ip igmp limit` ป้องกัน **จำนวน Group/State ล้น** — แต่ไม่ได้ป้องกัน **1 Source ตัวเดียวส่ง
Traffic Rate สูงผิดปกติ** เช่น Security Camera Feed (`239.70.3.5`) ที่ปกติควรส่งแค่ ~2 Mbps
(H.264 Resolution ทั่วไป) แต่ถ้า Firmware เสีย/ถูก Compromise แล้วเริ่มส่งที่ 200+ Mbps จะไม่มี
Mechanism ไหนที่เรียนมาป้องกันเลย — `ip multicast rate-limit` แก้ปัญหานี้โดยตรง

### 826.2 คำสั่งและ Config

```
Router(config-if)# ip multicast rate-limit {in | out} [group-list <acl>] [source-list <acl>] <kbps>
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| `in` / `out` | จำกัด Traffic ที่ **เข้า** หรือ **ออก** จาก Interface นี้ |
| `group-list` | จำกัดเฉพาะ Group ที่ ACL Match (ไม่ระบุ = ทุก Group) |
| `source-list` | จำกัดเฉพาะ Source ที่ ACL Match (ไม่ระบุ = ทุก Source) |
| `<kbps>` | เพดาน Bandwidth รวมของ (S,G) ที่ Match — Packet ส่วนเกินจะถูก **Drop เงียบๆ** (ไม่มี ICMP แจ้งกลับ เหมือน Policer ทั่วไป) |

### 826.3 Config เต็มรูปแบบ: ป้องกัน Camera Feed อาคาร 3 Flood

```
! ===== ACL ระบุ Source (Camera ตัวที่สงสัยว่า Compromised) และ Group ของมัน =====
DIST-SW3(config)# access-list 50 permit 10.10.30.55        ! IP ของ Camera-3 ในอาคาร
DIST-SW3(config)# access-list 60 permit 239.70.3.5          ! Group Feed ของ Camera-3 ตัวนี้

! ===== Apply Rate-Limit ที่ Interface ทางเข้าของ Camera Segment =====
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ip multicast rate-limit in group-list 60 source-list 50 8000
! จำกัดไม่ให้เกิน 8000 kbps (8 Mbps — เผื่อ Burst 4x จาก Baseline ~2 Mbps) ต่อ (S,G) นี้เท่านั้น
! Camera/Group อื่นที่ไม่ Match ACL 50/60 ไม่ได้รับผลกระทบเลย
```

### 826.4 Verification

```
DIST-SW3# show ip mroute count
IP Multicast Statistics
Group: 239.70.3.5, Source count: 1
  Source: 10.10.30.55,  Forwarding: 1/0/1450/850, Other: 1/0/0
                              ^pkts/pps/bytes/bps      ^dropped by rate-limit

! ค่า "bps" ที่เห็นควรไม่เกิน 8000 kbps ที่ตั้งไว้ — ถ้า Traffic จริงส่งเกิน จะเห็นตัวเลขใน
! คอลัมน์ "Other" (Dropped) เพิ่มขึ้นต่อเนื่อง ยืนยันว่า Rate-Limit ทำงานจริงและ Drop ส่วนเกินอยู่
```

> **ข้อสังเกตสำคัญ**: `ip multicast rate-limit` ทำงานที่ **Traffic ระดับ Data Plane เท่านั้น**
> ไม่เกี่ยวกับ (S,G) State เลย (ต่างจาก `ip igmp limit` ที่จำกัดจำนวน State) — ใช้คู่กับ Boundary
> (Part 75 Step 744) และ IGMP Limit (Part 75 Step 747) เป็น **Defense-in-depth 3 ชั้น**: Boundary
> คุมทิศทาง (Group ไหนไปไหนได้), IGMP Limit คุมปริมาณ State, Rate-Limit คุมปริมาณ Bandwidth ต่อ
> Source — ครบทั้ง 3 มิติของ Resource ที่ Multicast อาจใช้เกินจำเป็น

---

## Step 827 — PIM Join/Prune Suppression และ PIM State Refresh

### 827.1 Join/Prune Suppression บน Multi-access Link

ทบทวนจาก [Part 41 Step 405-406](part-041-multicast-fundamentals.md#step-405--pim-dense-mode-pim-dm-flood-and-prune-model)
— PIM Join/Prune Message ถูกส่งแบบ Multicast ไปที่ `224.0.0.13` (All-PIM-Routers) เสมอ **ไม่ใช่
Unicast ไปยัง Upstream Router โดยตรง** — ผลคือบน **Multi-access Link ที่มี Downstream Router
มากกว่า 1 ตัว** (เช่น VLAN 10 SALES ที่มี DIST-SW1/DIST-SW2 ทั้งคู่เป็น PIM Neighbor ตาม Step
821.2) **ทุก Router บน Link นั้นได้ยิน Join/Prune ของกันและกันโดยอัตโนมัติ** — นี่คือที่มาของ
"Suppression" ตามธรรมชาติ: **ถ้า Router A ส่ง Join ไปแล้ว Router B ที่ต้องการ Group เดียวกันไม่
จำเป็นต้องส่ง Join ซ้ำ** (Upstream Router เห็น Join จาก A ก็ Forward ให้ทั้ง Link อยู่แล้ว)

### 827.2 Prune Override — กลไกป้องกัน Suppression ทำงานผิดทาง

ปัญหาตรงข้ามที่ต้องระวัง: ถ้า Router A **ไม่ต้องการ** Group แล้วส่ง **Prune**, แต่ Router B บน
Link เดียวกัน **ยังต้องการ** Group นั้นอยู่ — ถ้า Upstream หยุดส่งทันทีตาม Prune ของ A, Router B
จะเสีย Traffic ไปด้วย (ทั้งที่ยังต้องการ) — PIM แก้ด้วย **Prune Override Timer**: เมื่อ Router B
ได้ยิน Prune ของ A บน Link เดียวกัน จะรอ **3 วินาที** (Override Interval มาตรฐาน) ก่อน Prune
มีผลจริง — ถ้าภายใน 3 วินาทีนี้ Router B ส่ง **Join** แข่งเข้ามา (Override) Upstream จะ **ไม่ Prune
Link นั้นเลย** (ยกเลิก Prune ของ A ไปเอง)

```
Link มี 2 Downstream Router (A ไม่ต้องการ Group, B ยังต้องการ)

  A ──Prune(G)──┐
                ├──► Upstream Router ได้ยินทั้งคู่บน Link เดียวกัน (Multicast to 224.0.0.13)
  B ──Join(G)───┘     รอ 3 วินาที (Override Window) แล้วเห็น Join ของ B เข้ามาทัน
                       → Upstream "ยกเลิก" Prune ของ A โดยอัตโนมัติ ยังคง Forward ต่อ Link นี้
```

### 827.3 PIM State Refresh — Feature เสริมของ PIM-DM เท่านั้น (Legacy)

[Part 41 Step 405](part-041-multicast-fundamentals.md#step-405--pim-dense-mode-pim-dm-flood-and-prune-model)
อธิบายว่า PIM-DM ต้อง **Re-flood ทุก 3 นาที** เพื่อ Refresh Prune State — **PIM State Refresh**
(Cisco Extension, RFC Draft ที่ไม่ได้ Standardize เต็มรูปแบบ) แก้ปัญหานี้โดยให้ Router ใกล้ Source
ที่สุด (First-Hop) ส่ง **State Refresh Message** ลง Tree เป็นระยะ (Default 60 วินาที) เพื่อ
**ต่ออายุ Prune State เดิมโดยไม่ต้อง Flood ใหม่ทั้ง Tree**

```
Router(config-if)# ip pim state-refresh origination-interval <1-100>   ! ตั้งความถี่ (Default 60s)
Router(config-if)# ip pim state-refresh disable                        ! ปิด ถ้ามี Router รุ่นเก่าที่ไม่รองรับ
```

| คำสั่ง | ความหมาย |
|---|---|
| `ip pim state-refresh origination-interval` | ตั้งบน Router ที่ **ใกล้ Source ที่สุด** เท่านั้นที่มีผล (Router อื่นแค่รับและ Forward State Refresh ต่อ) |
| `ip pim state-refresh disable` | ปิด State Refresh บน Interface นั้น — จำเป็นถ้ามี Router Vendor อื่นหรือ IOS รุ่นเก่าที่ไม่เข้าใจ Message นี้ (จะทำให้ Prune Timeout ปกติทุก 3 นาทีแทน ซึ่งยังทำงานได้แต่ไม่มีประสิทธิภาพเท่า) |

> **สรุปสำหรับ CCIE Enterprise Infrastructure**: หัวข้อนี้ถูกถามใน Written/Lab Exam เพื่อทดสอบ
> ความรู้ **Feature ที่เหลือของ PIM-DM ทั้งหมด** มากกว่าจะเป็น Feature ที่ต้องใช้จริง — เพราะ
> [Part 41 Step 405](part-041-multicast-fundamentals.md#step-405--pim-dense-mode-pim-dm-flood-and-prune-model)
> ยืนยันไปแล้วว่า Enterprise สมัยใหม่ใช้ PIM-SM เกือบ 100% — หลักสูตรนี้จึงให้ครบเพื่อความสมบูรณ์
> ของทฤษฎีสำหรับข้อสอบ CCIE เท่านั้น ไม่มี Lab เต็มรูปแบบเพราะไม่ตรงกับ Design จริงของ Topology นี้

---

## Step 828 — Multicast และ NAT: ความเข้ากันไม่ได้ที่ต้องรู้

### 828.1 ทำไม Multicast ข้าม NAT Boundary ไม่ได้ดี

ทบทวนจาก [Part 15](part-015-nat-pat.md)/[Part 63](part-063-advanced-nat-scenarios.md) — NAT/PAT
ทำงานบนสมมติฐานว่า **Traffic เป็น Unicast แบบมี Connection ที่ระบุตัวตนได้ชัดเจน** (Source IP +
Port ↔ Translated IP + Port แบบ 1:1 หรือ N:1 ผ่าน PAT Table) — Multicast **ทำลายสมมติฐานนี้ทันที**
ด้วย 2 เหตุผลหลัก:

| ปัญหา | รายละเอียด |
|---|---|
| **RPF Check พังถ้า Source ถูก NAT** | [Part 41 Step 408](part-041-multicast-fundamentals.md#step-408--multicast-forwarding-และ-rpf-check-sg-และ-g) กำหนดว่า RPF Check ต้องเทียบ **Source Address ใน Packet จริง** กับ Unicast Routing Table เพื่อหา Incoming Interface ที่ถูกต้อง — ถ้า Source ถูก NAT เปลี่ยน Address ไปแล้วก่อนถึง Router ที่ทำ PIM, Router นั้นจะ RPF Check โดยเทียบกับ **Address ที่ถูกแปลงแล้ว** ซึ่งอาจไม่ตรงกับ Interface ที่ Multicast Tree ควรใช้เลย — เกิด RPF Failure และ Traffic ถูก Drop |
| **Group Address ไม่ควรถูกแปลงเลย** | Destination Group Address (เช่น `239.1.1.1`) ไม่ใช่ Address ของอุปกรณ์ตัวใดตัวหนึ่งที่จะ "แปล" ได้ตรงไปตรงมาแบบ Unicast — Receiver ทุกตัว Join Group เดิมผ่าน IGMP/MLD; ถ้า Group ถูก NAT เปลี่ยนเป็น Group อื่นระหว่างทาง, IGMP Membership ที่ Client ประกาศไว้ (Group เดิม) จะไม่ตรงกับ Traffic จริงที่ไหลมา (Group ใหม่) ทำให้ Client ไม่เห็น Stream เลย |

### 828.2 ตัวอย่างจริงในหลักสูตร: ทำไม MVPN (Step 822) ต้องเลี่ยง NAT Path โดยเจตนา

**WAN-EDGE-1** ทำ PAT ให้ Internal Host ออก Internet มาตั้งแต่ [Part 15](part-015-nat-pat.md)
(`ip nat inside`/`ip nat outside` บน Interface ฝั่ง Internet) — สังเกตว่า **MVPN Lab ใน Step 822
จงใจให้ GRE MDT Tunnel วิ่งผ่าน Port-channel2 (ทางไป Core/WAN-EDGE-2 ภายในองค์กร) ไม่ใช่ทางออก
Internet ที่ทำ NAT** — นี่ไม่ใช่ความบังเอิญ แต่เป็นหลักการ Design ที่ต้องยึดเสมอ:

```
Design ที่ใช้ได้ (Step 822)                       Design ที่จะพังถ้าลองทำ

GUEST VRF Multicast                               GUEST VRF Multicast
    │                                                  │
    ▼ ผ่าน Provider PIM Domain                          ▼ ผ่าน NAT Boundary (ip nat inside/outside)
  Port-channel2 (ไม่มี NAT)                          Interface ฝั่ง Internet (มี NAT/PAT)
    │                                                  │
    ▼                                                  ▼ Source Address ถูกแปลงก่อน RPF Check
  WAN-EDGE-2 (RPF Check ตรง — ไม่มี NAT คั่น)          RPF Check เทียบ Address ที่ผิด → Drop
```

### 828.3 แนวทาง Design เมื่อต้องให้ Multicast ไปถึง Site ที่อยู่หลัง NAT

| แนวทาง | คำอธิบาย |
|---|---|
| **หลีกเลี่ยง NAT Boundary ให้อยู่คนละ Path กับ Multicast RPF Boundary เสมอ** | เหมือนที่ Step 822 ทำ — ให้ Multicast วิ่งผ่าน Path ภายใน (VPN/GRE/DMVPN ตาม Part 75 Step 746) ที่ไม่มี NAT คั่นระหว่างทาง |
| **Tunnel ก่อน NAT เสมอ (Encapsulate ก่อนแปล Address)** | ถ้าจำเป็นต้องวิ่งผ่าน Network ที่มี NAT จริงๆ ให้ Encapsulate Multicast Packet ไว้ใน GRE/IPsec ก่อน (เหมือน MDT Tunnel ใน Step 822 หรือ DMVPN ใน Part 75 Step 746) แล้วให้ NAT แปล Address ของ **Tunnel Endpoint (Unicast)** เท่านั้น ไม่ใช่ Address ข้างในที่เป็น Multicast — Payload ข้างใน Tunnel ไม่ถูก NAT แตะเลย |
| **Application-level Relay/Proxy แทนการ Route Multicast ข้าม NAT ตรงๆ** | ถ้า Site ปลายทางอยู่หลัง NAT ของผู้ให้บริการ Internet จริง (ควบคุมไม่ได้) ให้ตั้งอุปกรณ์ Relay ที่รับ Multicast Stream ทางฝั่งที่ไม่มี NAT แล้ว Re-originate เป็น Unicast (หรือ Multicast ใหม่ในโดเมนท้องถิ่น) ฝั่ง Remote Site แทน — หลักการเดียวกับ mDNS Gateway ที่จะเรียนใน Step 829 |

---

## Step 829 — Multicast Auto-Discovery: mDNS/Bonjour และ Cisco mDNS Gateway

### 829.1 "Multicast" อีกแบบที่ Enterprise เจอบ่อยที่สุดแต่ไม่เกี่ยวกับ PIM เลย

ตลอดหลักสูตรที่ผ่านมา Multicast ที่เราเรียนคือ Traffic ที่ต้องข้าม VLAN/Subnet ผ่าน PIM — แต่มี
Multicast อีกประเภทที่ **ถูก Design มาให้อยู่แค่ Local Segment เดียวโดยเจตนา (TTL=1 เสมอ)** ตาม
[Part 41 Step 401](part-041-multicast-fundamentals.md#step-401--multicast-คืออะไร-unicast-vs-broadcast-vs-multicast)
Local Network Control Block: **mDNS (Multicast DNS)** ที่ Apple เรียกว่า **Bonjour** (และ Android/
Linux เรียก Avahi) ใช้ Group `224.0.0.251` (IPv4) / `ff02::fb` (IPv6) UDP Port 5353 สำหรับ
**Zero-configuration Service Discovery** — AirPlay, AirPrint, Chromecast, Network Printer,
Spotify Connect ล้วนใช้กลไกนี้ในการ "ประกาศตัวตน" ให้อุปกรณ์ข้างเคียงเห็น

### 829.2 ปัญหาจริงที่ Enterprise Wireless เจอบ่อยที่สุด

```
พนักงานเชื่อม Wi-Fi (VLAN 40 WIFI)                    Apple TV ในห้องประชุม (VLAN 10 SALES, ต่อสาย)
        │                                                      │
        │  mDNS Query ("มี AirPlay Receiver ไหนบ้าง?")          │  mDNS Response (224.0.0.251, TTL=1)
        ▼  ส่งไปที่ 224.0.0.251                                 ▼  ตอบกลับแค่ใน VLAN 10 เท่านั้น
   ┌─────────┐                                              ┌─────────┐
   │ AP/WLC  │──X── ไม่ข้าม VLAN ไปหา Apple TV เลย            │ Switch  │
   └─────────┘     (TTL=1 บังคับให้ Router ไม่ Forward         └─────────┘
                    ข้าม Interface โดย Design — เหมือน
                    OSPF/EIGRP Hello ตาม Part 41 Step 401)

ผลลัพธ์: พนักงานเปิด AirPlay บนมือถือ ไม่เห็น Apple TV ในห้องประชุมเลย ทั้งที่อยู่ Network เดียวกัน
         (นี่คือ Enterprise Wireless Ticket ที่พบบ่อยที่สุดเรื่องหนึ่งในทุกองค์กรที่มี BYOD)
```

**การใช้ PIM/IGMP ทำ Multicast Routing ให้ `224.0.0.251` ข้าม VLAN ไม่ใช่คำตอบที่ถูก** — เพราะ
Group นี้อยู่ใน Local Network Control Block โดยเจตนา (RFC 6762 กำหนด TTL=1 ตายตัว) การพยายาม
"ฝืน Route" TTL=1 Traffic ข้าม Subnet จะทำให้ mDNS Query/Response Flood ไปทั่วทั้ง Multicast
Domain (ทุก Service จากทุก VLAN ประกาศไปทุกที่พร้อมกัน) เสี่ยง Broadcast Storm ระดับ Application
Layer และเสียหลักการ Scope ที่ RFC ตั้งใจไว้

### 829.3 ทางออกที่ถูกต้อง: mDNS Gateway (Service Discovery Relay ไม่ใช่ Multicast Routing)

**Cisco mDNS Gateway** (มีบน Catalyst 9000 Series และ Catalyst 9800 WLC) ทำงานต่างจาก PIM
โดยสิ้นเชิง: มันคือ **Application-layer Proxy** ที่ **"ดักฟัง" mDNS Message, เก็บ Service Record
ไว้ใน Cache ของตัวเอง (ไม่ใช่ Multicast Tree)** แล้ว **"ตอบแทน" (Re-originate) Response นั้นให้กับ
VLAN อื่นที่ Config ไว้ว่าให้ Bridge ถึงกัน** — เสมือนมี Router "แกล้งเป็น" Apple TV ตอบ mDNS
Query จากอีกฝั่งแทน โดยที่ Traffic จริงของ AirPlay (Video/Audio Stream หลังจาก Discover เจอกันแล้ว)
ยังเป็น Unicast ปกติ ไม่เกี่ยวกับ Multicast เลย

```
Config เต็มรูปแบบ: เปิด mDNS Gateway เชื่อม VLAN 10 (Apple TV) กับ VLAN 40 (Wi-Fi Client)

CORE-SW1(config)# mdns-sd gateway
CORE-SW1(config-mdns-sd-gw)# service-list AIRPLAY-SERVICES service-policy-name AIRPLAY-POLICY
CORE-SW1(config-mdns-sd-gw)# service-policy AIRPLAY-POLICY IN
CORE-SW1(config-mdns-sd-gw)# service-instance-suffix-string HQ-Campus
CORE-SW1(config-mdns-sd-gw)# exit

CORE-SW1(config)# mdns-sd service-list AIRPLAY-SERVICES SERVICE-LIST-TYPE
CORE-SW1(config)# mdns-sd service-list AIRPLAY-SERVICES service-permit AirPlay
CORE-SW1(config)# mdns-sd service-list AIRPLAY-SERVICES service-permit AirTunes
CORE-SW1(config)# mdns-sd service-list AIRPLAY-SERVICES service-permit AirPrint

! ===== Apply เข้ากับ VLAN ที่ต้องการ Bridge Service Discovery ระหว่างกัน =====
CORE-SW1(config)# interface Vlan10
CORE-SW1(config-if)# mdns-sd gateway
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface Vlan40
CORE-SW1(config-if)# mdns-sd gateway
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| `mdns-sd gateway` (Global) | เปิด Gateway Process หลัก — เริ่ม Cache Service Record ที่ Learn จากทุก VLAN ที่เปิด `mdns-sd gateway` ไว้ |
| `service-list` | กำหนดว่า Service **ประเภทไหน** (AirPlay, AirPrint, Chromecast ฯลฯ) ที่จะถูก Relay ข้าม VLAN — ควบคุมไม่ให้ Service ที่ไม่เกี่ยวข้อง (เช่น Printer ของแผนกบัญชี) รั่วไปหา Guest Wi-Fi โดยไม่จำเป็น |
| `mdns-sd gateway` (Interface) | Apply Gateway บน SVI ของ VLAN ที่ต้องการให้เข้าร่วม (เฉพาะ VLAN ที่เปิดไว้เท่านั้นที่ Relay ถึงกัน — VLAN 99 MGMT ที่ไม่เปิดจะไม่เห็น Service เหล่านี้เลย ตามหลัก Security Segmentation) |

> **จุดสำคัญที่ต้องจำสำหรับข้อสอบ**: mDNS Gateway **ไม่ใช่ Multicast Routing** และ **ไม่ใช้ PIM/
> IGMP เลยแม้แต่นิดเดียว** — มันคือ Cache-and-Relay Proxy ที่ทำงานที่ Layer เนื้อหา (DNS-SD Record
> Parsing) ไม่ใช่ Layer 3 Forwarding — คำถามข้อสอบมักถามแยกให้ชัดว่า **"ทำไมไม่ใช้ PIM ทำให้
> 224.0.0.251 ข้าม VLAN ได้เลย"** คำตอบคือ TTL=1 เป็นเจตนาการ Design ของ RFC 6762 เพื่อจำกัด Scope
> ไว้ที่ Link เดียวเสมอ ไม่ใช่ข้อจำกัดทางเทคนิคที่ PIM จะมาแก้ได้อย่างเหมาะสม

---

## Step 830 — Lab เต็มรูปแบบ: รวม 4 Feature ของ Part นี้เข้าด้วยกัน

Lab นี้รวม **Bidir-PIM (Step 821)**, **IPv6 Multicast Dual-Stack (Step 824)**, **MSDP SA-filter
(Step 825)** และ **mDNS Gateway (Step 829)** เข้าไว้ในระบบเดียวกัน — 4 เรื่องนี้ **ไม่ทับซ้อนกัน
เลย** (ทำงานคนละ Layer/Protocol กันโดยสิ้นเชิง) และทั้งหมดทำงาน**ควบคู่**กับทุกอย่างที่ Part 41/42/
75 Config ไว้แล้วโดยไม่ต้องแก้ Config เดิมแม้แต่บรรทัดเดียว

### 830.1 ภาพรวม Topology สุดท้ายของ Part นี้

```
                                   CORE-SW1 ◄──MSDP + Mesh-group──► CORE-SW2
                                   Lo1: 1.1.1.100 (Anycast RP, PIM-SM เดิม)     Lo1: 1.1.1.100
                                   Lo199: 10.255.199.1/30 (Phantom RP Bidir)    Lo199: 10.255.199.2/24
                                   Lo0-v6: fd00:1:1:1::1 (RP ของ PIM6)
                                   mdns-sd gateway (Vlan10, Vlan40)
                                        │                    │
                       ┌────────────────┴──────┐   ┌─────────┴────────────────┐
                       │                        │   │                         │
                  DIST-SW1/2                DIST-SW3/4                    (ไป DC-CORE-1/2
                  VLAN 10 SALES              VLAN 30 SERVERS                ผ่าน Mesh-group
                  - Bidir Trader Terminal    - Matching Engine (Bidir)       ที่ Hardening
                  - IPv6 MLD/PIM6            - STREAM-SRV (IPv4+IPv6)        แล้วด้วย SA-filter)
                  - mDNS Gateway (AirPlay)   VLAN 40 WIFI
                                             - mDNS Gateway (AirPlay Client)
```

### 830.2 ส่วนที่ 1 — Bidir-PIM Trading Floor (สรุปจาก Step 821)

```
! Global bidir-enable + Phantom RP บน CORE-SW1/CORE-SW2 (เต็มรูปแบบใน Step 821.5)
CORE-SW1(config)# ip pim bidir-enable
CORE-SW1(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
CORE-SW2(config)# ip pim bidir-enable
CORE-SW2(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
DIST-SW1(config)# ip pim bidir-enable
DIST-SW1(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
DIST-SW2(config)# ip pim bidir-enable
DIST-SW2(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
DIST-SW3(config)# ip pim bidir-enable
DIST-SW3(config)# ip pim rp-address 10.255.199.1 BIDIR-TRADING-GROUP bidir
```

**Verification**:
```
DIST-SW1# show ip pim rp mapping 239.9.9.9
Group(s) 239.9.9.9/32
  RP 10.255.199.1 (?), v2, bidir
    Info source: 10.255.199.1 (?), via static

DIST-SW1# show ip mroute 239.9.9.9
(*, 239.9.9.9), 00:40:02/00:02:59, RP 10.255.199.1, flags: BC
  Bidir-Upstream: TenGigabitEthernet1/1/1, RPF nbr 10.255.10.1
  Outgoing interface list:
    Vlan10, Forward/Sparse, 00:40:02/00:02:59
```

### 830.3 ส่วนที่ 2 — IPv6 Multicast Dual-Stack (สรุปจาก Step 824)

```
CORE-SW1(config)# ipv6 multicast-routing
CORE-SW1(config)# ipv6 pim rp-address fd00:1:1:1::1
DIST-SW3(config)# ipv6 multicast-routing distributed
DIST-SW3(config)# ipv6 pim rp-address fd00:1:1:1::1
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ipv6 mld version 2
DIST-SW3(config-if)# ipv6 pim
```

**Verification**:
```
DIST-SW1# show ipv6 mroute ff05::239:1:1 | include Outgoing -A2
Outgoing interface list:
    Vlan10, Forward/Sparse, 00:12:40/00:02:44

! STREAM-SRV Video ไหลถึง VLAN 10 SALES ทั้ง IPv4 (239.1.1.1) และ IPv6 (ff05::239:1:1) พร้อมกัน
```

### 830.4 ส่วนที่ 3 — MSDP SA-filter Hardening (สรุปจาก Step 825)

```
CORE-SW1(config)# ip msdp sa-filter in 1.1.1.2 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter in 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter in 2.2.2.2 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter out 2.2.2.1 list MSDP-VALID-SOURCES
CORE-SW1(config)# ip msdp sa-filter out 2.2.2.2 list MSDP-VALID-SOURCES
```

**Verification**:
```
CORE-SW1# show ip msdp sa-filter
SA Filter list applied to peers:
Peer 1.1.1.2 : in MSDP-VALID-SOURCES : out -
Peer 2.2.2.1 : in MSDP-VALID-SOURCES : out MSDP-VALID-SOURCES
Peer 2.2.2.2 : in MSDP-VALID-SOURCES : out MSDP-VALID-SOURCES
```

### 830.5 ส่วนที่ 4 — mDNS Gateway (สรุปจาก Step 829)

```
CORE-SW1(config)# mdns-sd gateway
CORE-SW1(config-mdns-sd-gw)# service-policy AIRPLAY-POLICY IN
CORE-SW1(config)# interface Vlan10
CORE-SW1(config-if)# mdns-sd gateway
CORE-SW1(config)# interface Vlan40
CORE-SW1(config-if)# mdns-sd gateway
```

**Verification**:
```
CORE-SW1# show mdns-sd cache
Service Instance Name           Service Type        VLAN    TTL
Conference-Room-A._airplay      _airplay._tcp        10      120
Conference-Room-A._raop         _raop._tcp            10      120

! Wi-Fi Client (VLAN 40) เห็น "Conference-Room-A" เป็นตัวเลือก AirPlay ได้แล้ว แม้ Apple TV
! ตัวจริงอยู่คนละ VLAN — mDNS Gateway Relay Record นี้ให้โดยไม่ต้องมี PIM/IGMP เกี่ยวข้องเลย
```

### 830.6 สรุปผลการทดสอบทั้ง 4 ส่วน

| Feature | ผลก่อน Config | ผลหลัง Config |
|---|---|---|
| Bidir-PIM | Many-to-Many ต้องใช้ (S,G) แยกทุก Trader — Scale ไม่ได้กับ 300 เครื่อง | มีแค่ `(*, 239.9.9.9)` เดียวทั้ง Domain — Scale ได้จริง |
| IPv6 Multicast | STREAM-SRV มีแค่ IPv4 — ผู้ใช้ IPv6-only เห็นไม่ได้ | Dual-Stack สมบูรณ์ ผ่าน MLDv2 + PIM6 |
| MSDP SA-filter | Rogue Source ปลอมตัวได้ กระจาย SA ไปทั้ง DC/Branch | SA ของ Rogue Source ถูกกรองตั้งแต่ Local RP ไม่กระจายออกเลย |
| mDNS Gateway | AirPlay/Bonjour ไม่ข้าม VLAN 10↔40 เลย (TTL=1 by design) | Client เห็น Apple TV ข้าม VLAN ได้ โดยไม่กระทบ Scope Design ของ RFC 6762 |

---

## Quiz ทบทวน Part 83

**Q1.** อธิบายกลไก DF Election ของ Bidir-PIM ทีละขั้นตอน (Offer → Backoff → Winner → Lose) และ
บอกว่าทำไม DF ต้องเลือกแยกกันทุก Link ไม่ใช่เลือกครั้งเดียวทั้ง Domain

**Q2.** Phantom RP ทำงานอย่างไรให้เกิด Redundancy ได้ทั้งที่ RP Address ไม่ได้ถูก Config เป็น
Address จริงบน Interface ใดเลย — อธิบายด้วยหลัก Longest-Prefix-Match

**Q3.** MVPN Rosen GRE (Step 822) กับ BGP-based/NG-MVPN (Step 823) ต่างกันที่ Signaling Mechanism
อย่างไร และทำไม Enterprise ใหม่มักเลือกแบบ BGP-based มากกว่า

**Q4.** ทำไม `ip multicast boundary` (Part 75), `ip msdp sa-filter` (Step 825) และ
`ip multicast rate-limit` (Step 826) ถึงต้องใช้ **ทั้ง 3 อย่างร่วมกัน** ในการป้องกัน Multicast
Resource Abuse ให้ครบทุกมิติ — แต่ละอย่างป้องกันมิติไหน

**Q5.** ทำไมการพยายามใช้ PIM/IGMP ทำ Multicast Routing ให้ mDNS (`224.0.0.251`) ข้าม VLAN ถึงไม่ใช่
วิธีที่ถูกต้อง และ mDNS Gateway แก้ปัญหานี้ด้วยกลไกที่ต่างออกไปอย่างไร

---

## เฉลย Quiz

**A1.** **Offer**: Router ที่ยังไม่ใช่ DF (หรือเห็น DF ปัจจุบัน Metric แย่กว่าตัวเอง) ประกาศ Metric
ของตัวเองไปยัง RP บน Link — **Backoff**: Router ที่ชนะการเปรียบเทียบ Offer ทั้งหมด ณ ขณะนั้นรอ
Timer สั้นๆ ก่อนประกาศชนะจริง (เผื่อมี Router Metric ดีกว่าเข้ามาแข่งอีก) — **Winner**: หลัง Backoff
หมดโดยไม่มีคู่แข่งดีกว่า ประกาศตัวเป็น DF และส่ง Winner Message เป็นระยะยืนยันสถานะ — **Lose**:
Router อื่นทั้งหมดบน Link ที่ไม่ชนะ เงียบไม่ Forward Traffic ขึ้น/ลง Link นั้น — ต้องเลือกแยกกันทุก
Link เพราะ **Metric ไปยัง RP วัดจาก Interface/Link นั้นโดยเฉพาะ** (คนละ Interface มี Cost ต่างกัน)
Router ตัวเดียวอาจเป็น DF บน Link หนึ่งแต่ Lose บน Link อื่นในเวลาเดียวกันได้

**A2.** Router 2 ตัวถูกกำหนดให้ Advertise Prefix ที่ "ครอบคลุม" RP Address เดียวกันด้วย Subnet
Mask ยาวไม่เท่ากัน (เช่น CORE-SW1 Advertise `/30` ที่มี RP Address นี้เป็น Address จริงบน Loopback,
CORE-SW2 Advertise `/24` ที่ครอบคลุม Address เดียวกันแต่เป็น Supernet) — ตราบใดที่ CORE-SW1 (เจ้า
ของ Prefix ที่ยาวกว่า) ยัง Up, ทุก Router ทั้ง Domain จะเลือกเส้นทางผ่าน CORE-SW1 เสมอเพราะ
Longest-Prefix-Match ชนะ Metric/Cost — ถ้า CORE-SW1 ล่ม Prefix `/30` หายจาก IGP ทันที เหลือแค่
`/24` จาก CORE-SW2 ที่ยังครอบคลุม Address เดิม ทำให้ทุก Router Re-converge อัตโนมัติผ่าน CORE-SW2
โดยไม่ต้องแก้ `ip pim rp-address` เลยแม้แต่ตัวอักษรเดียว

**A3.** Rosen GRE Signal Membership ผ่าน **PIM Join/Prune ใน P-Network จริง** (PE ทุกตัว Join
Provider Multicast Group ผ่าน PIM) — BGP-based/NG-MVPN Signal ผ่าน **MP-BGP Update** (MDT-SAFI/
MVPN NLRI Type 1-7) โดยไม่ต้องมี PIM Domain ใน Core เลยถ้าเลือก P-Tunnel แบบ mLDP/Ingress
Replication — Enterprise ใหม่เลือกแบบ BGP-based เพราะ Scale/Convergence ดีกว่า และใช้
BGP Route Reflector Infrastructure เดิมที่มีอยู่แล้วสำหรับ VPNv4/VPNv6 Unicast อยู่แล้ว ไม่ต้อง
ดูแล PIM Domain แยกซ้ำซ้อนใน Core

**A4.** `ip multicast boundary` ป้องกันมิติ **Direction/Scope** (Group ไหนข้าม Administrative
Boundary ได้หรือไม่) — `ip msdp sa-filter` ป้องกันมิติ **Source Legitimacy** (Source ไหนมีสิทธิ์
เป็นเจ้าของ Group ไหนบ้าง ที่ระดับ (S,G) จริง ละเอียดกว่า Boundary ที่ดูแค่ Group) —
`ip multicast rate-limit` ป้องกันมิติ **Bandwidth Consumption** (Source ที่ถูกต้องแล้วแต่ Flood
เกิน Baseline ผิดปกติ) — ทั้ง 3 ทำงานคนละ Layer ของปัญหา ต้องใช้ร่วมกันเพื่อครอบคลุม Security/
Resource Exhaustion ครบทุกมุม

**A5.** mDNS (`224.0.0.251`) ถูก RFC 6762 กำหนด TTL=1 ตายตัวโดยเจตนา เพื่อจำกัด Scope ไว้ที่ Link
เดียวเสมอ (เหมือน OSPF/EIGRP Hello ตาม Part 41 Step 401) — การฝืนทำ Multicast Routing ให้ข้าม VLAN
จะทำให้ Query/Response จากทุก Service ทุก VLAN Flood ไปทั่ว Multicast Domain พร้อมกัน เสียหลักการ
Scope และเสี่ยง Storm ระดับ Application — mDNS Gateway แก้ปัญหาด้วยกลไกที่ต่างไปโดยสิ้นเชิง: มันคือ
**Application-layer Cache-and-Relay Proxy** ที่ดักฟัง Record, เก็บ Cache ไว้เอง แล้ว Re-originate
Response ให้ VLAN อื่นที่ Config ไว้ให้ Bridge ถึงกันเท่านั้น (ควบคุมด้วย Service-list) ไม่ใช้ PIM/
IGMP หรือ Multicast Routing จริงแม้แต่นิดเดียว

---

## สรุป Part 83

Part นี้เติมเต็มช่องว่างที่ [Part 75](part-075-advanced-multicast-enterprise.md) ตั้งใจทิ้งไว้ให้
Part หลังมาสาน — เริ่มจาก **Bidir-PIM Deep Dive** (DF Election ทีละ State, Phantom RP ด้วย
Longest-Prefix-Match, Config เต็มสำหรับ Trading Floor Many-to-Many — Step 821) ที่ Part 42 เคยให้
แค่ Concept, ต่อด้วย **MVPN ทั้ง 2 รุ่น**: Rosen GRE เต็มรูปแบบต่อยอดจาก VRF `GUEST` ของ
[Part 51](part-051-vrf-lite-gre-ipsec.md) (Step 822) และ BGP-based/NG-MVPN เชิงเปรียบเทียบสำหรับ
อนาคต (Step 823), เปิดโลก **IPv6 Multicast** ที่ไม่มี Part ไหนแตะมาก่อนเลย (MLD, PIM6, Dual-Stack
เต็มรูปแบบคู่กับ STREAM-SRV เดิม — Step 824), เพิ่มความปลอดภัยระดับ (S,G) ด้วย **MSDP SA-filter**
(Step 825) และปกป้อง Bandwidth ด้วย **Rate-Limit ต่อ Source** (Step 826), ปิดท้ายทฤษฎี PIM-DM ที่
เหลือด้วย **Join/Prune Suppression และ State Refresh** (Step 827), อธิบาย **ความเข้ากันไม่ได้ของ
Multicast กับ NAT** และผลต่อ Design จริง (Step 828), แนะนำ **mDNS/Bonjour Gateway** สำหรับปัญหา
Enterprise Wireless ที่พบบ่อยที่สุดแต่ไม่เกี่ยวกับ PIM เลย (Step 829) และปิดท้ายด้วย **Lab เต็ม
รูปแบบที่รวมทั้ง 4 Feature เข้าด้วยกันจริง** (Step 830)

จุดที่สำคัญที่สุดที่ต้องพาไปใช้ต่อ: **Multicast ระดับ CCIE ไม่ได้จบแค่ PIM-SM/Anycast RP/Scaling
Design เท่านั้น — ยังมี Feature เฉพาะทาง (Bidir สำหรับ Many-to-Many, MVPN สำหรับข้าม VPN Boundary,
IPv6 สำหรับ Dual-Stack, SA-filter/Rate-limit สำหรับ Security, และ mDNS Gateway สำหรับ Use Case
Wireless ที่ไม่เกี่ยวกับ PIM เลย) ที่ต้องเลือกใช้ให้ตรงกับโจทย์จริง ไม่ใช่ Tool เดียวที่ใช้ได้กับ
ทุกสถานการณ์** — Part 83 คือ Part สุดท้ายของสาย Multicast ในหลักสูตรนี้ หัวข้อถัดไปจะย้ายไปสู่
[Part 84 — VXLAN/EVPN Fundamentals](part-084-vxlan-evpn-fundamentals.md) ที่ **BUM Traffic
(Broadcast, Unknown-unicast, Multicast)** ของ VXLAN Overlay จะต้องเลือกระหว่าง **Multicast
Underlay** (ใช้ PIM-SM ที่เรียนมาทั้งหมดตั้งแต่ Part 41 ทำหน้าที่ Replicate BUM Traffic ข้าม VTEP)
กับ **Ingress Replication** (Unicast ล้วน ไม่ต้องมี PIM เลย คล้ายแนวคิด NG-MVPN ที่ Step 823
เกริ่นไว้) — แนวคิด Bidir-PIM, MVPN, และ RP Design ที่เรียนใน Part นี้จะกลับมาเป็นพื้นฐานสำคัญของ
การเลือก Underlay ที่ถูกต้องใน Part 84 ทันที
