# Part 87 — MPLS L3VPN Advanced & WAN Integration
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 861–870 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 86 — MPLS L3VPN Fundamentals](part-086-mpls-l3vpn-fundamentals.md) ที่สร้าง SP Core
> จำลอง (**SP-P1/SP-P2** เป็น P Router, **SP-PE1/SP-PE2** เป็น PE Router) และพิสูจน์ **Any-to-Any
> Connectivity** ระหว่าง **WAN-EDGE-1** (CE, HQ) กับ **REMOTE-SITE-CE** (CE, Branch) ผ่าน eBGP
> PE-CE + MP-BGP VPNv4 + Label Stacking สำเร็จไปแล้ว — Part 86 ปิดท้ายด้วยการสัญญาไว้ว่า Part 87
> จะยกระดับขึ้นสู่ประเด็นระดับ **Production/CCIE SP** จริง: PE-CE Routing Protocol อื่นนอกจาก eBGP
> (OSPF + Sham-Link, EIGRP + SoO), การทำ **Multi-Homing** ลูกค้าเข้ากับ PE สองตัว, **Internet
> Access** ผ่าน MPLS VPN, **MPLS TE**, **Carrier Supporting Carrier/Inter-AS VPN**, การเปรียบเทียบ
> กับ **SD-WAN** ([Part 39](part-039-sd-wan-fundamentals.md)/[Part 76](part-076-sdwan-advanced-design.md))
> และการนำ **MVPN** ([Part 75](part-075-advanced-multicast-enterprise.md)/
> [Part 83](part-083-enterprise-multicast-deep-dive.md)) มาวิ่งข้าม MPLS Core ตัวจริงที่สร้างไว้
> — Part นี้คือจุดที่ทำตามสัญญานั้นทั้งหมด และเพิ่มอุปกรณ์ตัวใหม่ **SP-PE3** เข้าไปใน SP Core
> เพื่อสอน Multi-Homing อย่างสมบูรณ์

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 861 | OSPF เป็น PE-CE Routing Protocol — Sham-Link และ DN-bit ป้องกัน Routing Loop เมื่อลูกค้ามี Backdoor Link |
| 862 | EIGRP เป็น PE-CE Routing Protocol — Site of Origin (SoO) ป้องกัน Loop ในลูกค้าที่ Multi-Home |
| 863 | Multi-Homing ลูกค้าเข้ากับ PE สองตัว — WAN-EDGE-1 ต่อทั้ง SP-PE1 และ SP-PE3 พร้อม BGP Path Selection ที่ถูกต้อง |
| 864 | Internet Access ผ่าน MPLS L3VPN — Internet VRF + Route Leaking เทียบกับ Internet Circuit แยก |
| 865 | MPLS Traffic Engineering (RSVP-TE) ภาพรวมเชิงแนวคิดสำหรับ Enterprise WAN Architect |
| 866 | Carrier Supporting Carrier (CsC) และ Inter-AS MPLS VPN — Option A/B/C |
| 867 | Migration Framework จาก MPLS L3VPN สู่ SD-WAN — เมื่อไหร่ควรอยู่ เมื่อไหร่ควรย้าย และ Hybrid Design |
| 868 | MPLS L3VPN + Multicast — Default MDT วิ่งข้าม SP-P1/SP-P2 ตัวจริง (ต่อจาก MVPN Rosen GRE Part 83) |
| 869 | Verification จากมุมมองลูกค้า — IP SLA End-to-End Monitoring แทนการมองเข้าไปใน SP Core |
| 870 | Lab เต็มรูปแบบ: Multi-Homing + OSPF Sham-Link + IP SLA Monitoring ครบวงจร |

---

## Step 861 — OSPF เป็น PE-CE Routing Protocol: Sham-Link และ DN-bit

### 861.1 ทำไมลูกค้าบางรายไม่ใช้ eBGP เป็น PE-CE

Part 86 เลือก eBGP เป็น PE-CE Protocol เพราะเป็นตัวเลือกที่ Scale ดีที่สุดและ RFC 4364 แนะนำเป็น
ค่า Default ของอุตสาหกรรม แต่ลูกค้าองค์กรจำนวนมาก **มี OSPF รันอยู่ทั้งองค์กรอยู่แล้ว** (เหมือน
Campus ของหลักสูตรนี้เองที่ใช้ `router ospf 1` มาตั้งแต่ Part 11) และไม่ต้องการเรียน BGP เพิ่มแค่
เพื่อต่อ MPLS VPN — SP จึงต้องรองรับ **OSPF เป็น PE-CE Protocol** ได้ด้วยตาม RFC 4577

> **ขอบเขตของ Step นี้**: เพื่อสอน Sham-Link อย่างสมบูรณ์ Step นี้จะสมมติว่า **ทั้ง WAN-EDGE-1
> และ REMOTE-SITE-CE เปลี่ยนจาก eBGP (Part 86) มาใช้ OSPF Area 0 เป็น PE-CE Protocol** — เป็น
> Design ทางเลือก (Alternative) สำหรับสอนแนวคิดเฉพาะ Step นี้ ไม่ได้ทับ Lab หลักของ Part 86
> (เหมือนที่ Part 86 Step 857 สอน Hub-and-Spoke แบบ VRF สมมติแยกจาก Lab หลักเช่นกัน) — Step 870
> จะนำ Design นี้ไปใช้จริงกับ REMOTE-SITE-CE ในภาพ Lab เต็มรูปแบบ

### 861.2 ปัญหา Backdoor Link ที่ทำให้ต้องมี Sham-Link

สมมติว่าก่อนซื้อ MPLS L3VPN นี้ WAN-EDGE-1 (HQ) และ REMOTE-SITE-CE (Branch) มี **Leased Line
สำรองเดิม (Backdoor Link)** เชื่อมกันตรงอยู่แล้ว (พบบ่อยมากในสถานการณ์ M&A ที่บริษัทมี Circuit
เก่าค้างอยู่ก่อนซื้อ MPLS VPN ใหม่) — Backdoor Link นี้ **ไม่ผ่าน SP Core เลย**

```
        WAN-EDGE-1 (HQ)                                    REMOTE-SITE-CE (Branch)
        AS/OSPF Area 0 ฝั่งลูกค้า                            AS/OSPF Area 0 ฝั่งลูกค้า
             │                                                       │
             │ Gi0/0/2 (192.0.2.2/30)                Gi0/0/1 (192.0.2.6/30) │
        ┌────┴─────┐                                            ┌────┴─────┐
        │ SP-PE1    │◄──────── ผ่าน MPLS Core (SP-P1/SP-P2) ─────►│ SP-PE2    │
        └───────────┘        (เป็น Inter-Area LSA — Type 3)       └───────────┘
             │                                                       │
             └─────────────── Gi0/0/4 ↔ Gi0/0/2 (10.99.0.0/30) ───────┘
                    Backdoor Link (Leased Line ของลูกค้าเอง — Intra-Area LSA Type 1)
                    ** OSPF ชอบ Intra-Area (Type 1) มากกว่า Inter-Area (Type 3) เสมอ
                       แม้ Cost รวมของ Backdoor อาจแย่กว่า MPLS Core จริง! **
```

ถ้าไม่มี Sham-Link, OSPF จะมองว่า Route ที่มาจาก MPLS Core (ผ่าน SP-PE1/SP-PE2) เป็น **LSA Type 3
(Inter-Area Summary)** เพราะ PE Redistribute BGP VPNv4 กลับเข้า OSPF เป็น Inter-Area เสมอ — ในขณะ
ที่ Backdoor Link เป็น **LSA Type 1 (Intra-Area)** ธรรมดา ซึ่ง OSPF **ให้ความสำคัญกับ Intra-Area
ก่อน Inter-Area เสมอไม่ว่า Cost จะเท่าไหร่** (ตามกฎ OSPF Route Preference มาตรฐาน) ทำให้ Traffic
ทั้งหมดวิ่งผ่าน Backdoor Link ที่อาจช้ากว่า/ไม่มี SLA รับประกันแทนที่จะใช้ MPLS Core ที่ลูกค้าเสีย
เงินซื้อมา — และในบางกรณี Backdoor ที่ไม่ผ่าน Any-to-Any Design ที่ถูกต้องอาจทำให้เกิด **Routing
Loop** เมื่อ Route เดินทางวนซ้ำระหว่าง Backdoor กับ MPLS Core

### 861.3 DN-bit (Down Bit) — กลไกป้องกัน Loop อัตโนมัติ

เมื่อ PE (SP-PE2) Redistribute Route ที่เรียนจาก MP-BGP VPNv4 (มาจาก SP-PE1) กลับเข้า OSPF PE-CE
ของ VRF, IOS-XE จะ **ติด DN-bit (Down Bit)** ให้ LSA Type 3/5 นั้นโดยอัตโนมัติเสมอ (ไม่มีคำสั่ง
Manual ต้องพิมพ์) — ความหมายคือ "Route นี้มาจาก BGP VPN ไม่ใช่ OSPF จริงของ Area นี้" ผลคือ **ถ้า
LSA นี้เดินทางย้อนกลับไปถึง PE อื่นอีกทีผ่าน Backdoor Link, PE ตัวนั้นจะ "ไม่" Redistribute LSA
ที่มี DN-bit กลับเข้า BGP VPNv4 อีก** — ตัดวงจร Loop ที่อาจเกิดขึ้นได้ 100%

```
SP-PE2# show ip ospf database summary 10.10.0.0
  ...
  LS Age: 45
  Options: (No TOS-capability, DC)
  LS Type: Summary Links (Network)
  Link State ID: 10.10.0.0 (summary Network Number)
  Advertising Router: 100.64.0.12
  ...
  Downward Bit Set          <-- DN-bit — ยืนยันว่า Route นี้มาจาก BGP VPN ไม่ใช่ OSPF จริง
```

> **DN-bit แก้ปัญหา Loop แต่ไม่แก้ปัญหา Path Preference** — ต้องใช้ **Sham-Link** แก้อีกชั้นหนึ่ง

### 861.4 Sham-Link คืออะไร และแก้ปัญหาอย่างไร

**Sham-Link** คือ **OSPF Virtual P2P Adjacency ระหว่าง PE สองตัวภายใน VRF เดียวกัน** ที่ทำให้
เส้นทางผ่าน MPLS Core "ดูเหมือน" เป็น **Intra-Area (Type 1)** เช่นเดียวกับ Backdoor Link — ทำให้
OSPF เปรียบเทียบ **Cost** ของทั้งสองเส้นทางอย่างเป็นธรรม (Apples-to-Apples) แทนที่จะให้ Backdoor
ชนะอัตโนมัติเพราะเป็นแค่ Intra-Area

Sham-Link ต้องมี **Loopback Address ที่อยู่ *ภายใน VRF*" (ไม่ใช่ Loopback0 Global ที่ใช้เพื่อ LDP/
MP-BGP)** เป็น Endpoint เพราะ Sham-Link ต้อง Reachable ผ่าน **MP-BGP VPNv4 เท่านั้น** (ไม่ใช่ผ่าน
OSPF เอง — ป้องกัน Chicken-and-Egg Problem ที่ OSPF ต้องพึ่ง Sham-Link แต่ Sham-Link ต้องพึ่ง OSPF)

**บน SP-PE1:**

```
SP-PE1(config)# interface Loopback100
SP-PE1(config-if)# description ** Sham-Link Source Address (อยู่ใน VRF, Reachable ผ่าน BGP VPNv4 เท่านั้น) **
SP-PE1(config-if)# vrf forwarding CUST-ENTLAB
SP-PE1(config-if)# ip address 100.64.100.11 255.255.255.255
SP-PE1(config-if)# exit
!
SP-PE1(config)# router bgp 65000
SP-PE1(config-router)# address-family ipv4 vrf CUST-ENTLAB
SP-PE1(config-router-af)# network 100.64.100.11 mask 255.255.255.255
SP-PE1(config-router-af)# exit-address-family
!
SP-PE1(config)# router ospf 2 vrf CUST-ENTLAB
SP-PE1(config-router)# router-id 100.64.0.11
SP-PE1(config-router)# area 0 sham-link 100.64.100.11 100.64.100.12
SP-PE1(config-router)# redistribute bgp 65000 subnets
SP-PE1(config-router)# network 192.0.2.0 0.0.0.3 area 0
SP-PE1(config-router)# exit
!
SP-PE1(config)# router bgp 65000
SP-PE1(config-router)# address-family ipv4 vrf CUST-ENTLAB
SP-PE1(config-router-af)# redistribute ospf 2 match internal external 1 external 2
SP-PE1(config-router-af)# exit-address-family
```

**บน SP-PE2** (สลับ IP/Neighbor ตามฝั่งตัวเอง — Loopback100 = `100.64.100.12`, `sham-link
100.64.100.12 100.64.100.11`, `network 192.0.2.4 0.0.0.3 area 0`) — รูปแบบคำสั่งเหมือนกันทุก
ประการ

| คำสั่ง | ความหมาย |
|---|---|
| `router ospf 2 vrf CUST-ENTLAB` | OSPF Process แยกเฉพาะ PE-CE ของ VRF นี้ (คนละ Process จาก `router ospf 100` ของ SP Core Step 851 และ `router ospf 1` ของ Enterprise Campus เดิม) |
| `area 0 sham-link <src> <dst>` | สร้าง Virtual Adjacency ระหว่าง Loopback100 ของ PE ทั้งสอง — ทำให้ Route ที่เรียนจาก BGP VPNv4 กลาย "Intra-Area" ในมุมมอง OSPF ของ VRF |
| `redistribute bgp 65000 subnets` (เข้า OSPF) | ดึง Route จาก BGP VPNv4 (ที่เรียนจาก SP-PE ฝั่งตรงข้าม) เข้ามาประกาศใน OSPF ของ VRF — IOS ติด DN-bit ให้อัตโนมัติ |
| `redistribute ospf 2 match ...` (เข้า BGP) | ดึง Route ที่เรียนจาก CE (WAN-EDGE-1) ผ่าน OSPF เข้า BGP VPNv4 เพื่อส่งข้าม Core ไปยัง PE อื่น |

### 861.5 Verify Sham-Link และผลลัพธ์ Path Selection

```
SP-PE1# show ip ospf sham-links

Sham Link OSPF_SL0 to address 100.64.100.12 is up
Area 0, source address 100.64.100.11
  Run as demand circuit
  DoNotAge LSA allowed. Cost of using 1 State POINT_TO_POINT,
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

SP-PE1# show ip route vrf CUST-ENTLAB ospf
O IA 172.30.10.0/24 [110/74] via 100.64.100.12, 00:02:11, OSPF_SL0
```

- `via 100.64.100.12, ... OSPF_SL0` = Route เดินทางผ่าน Sham-Link (MPLS Core) แม้จะยังเห็นเป็น
  `O IA` (Inter-Area — เพราะ Loopback100 อยู่ Area 0 คนละ Area กับ Interface จริง) แต่ **Cost
  [110/74] เทียบได้ตรงกับ Cost ของ Backdoor Link แบบ Apples-to-Apples แล้ว** — ถ้า Backdoor Link
  มี Cost แพงกว่า (เช่น Bandwidth ต่ำกว่า) OSPF จะเลือกเส้นทางผ่าน Sham-Link (MPLS Core) เป็น Best
  Path โดยอัตโนมัติตาม Cost จริง ไม่ใช่เพราะ LSA Type แต่อย่างเดียวอีกต่อไป

---

## Step 862 — EIGRP เป็น PE-CE Routing Protocol: Site of Origin (SoO)

### 862.1 EIGRP PE-CE — ทางเลือกสำหรับลูกค้าที่ใช้ EIGRP อยู่แล้ว

Enterprise ที่ยังใช้ EIGRP เป็น IGP หลัก (แทน OSPF) สามารถขอให้ SP รองรับ **EIGRP เป็น PE-CE
Protocol** ได้เช่นกัน โดย SP ต้องเปิด **EIGRP VRF-Aware** บน PE (`address-family ipv4 vrf ...
autonomous-system <n>` ภายใต้ `router eigrp <name>`) — กลไก Redistribute เข้า/ออก BGP VPNv4
เหมือนหลักการเดียวกับ OSPF (Step 861) ทุกประการ เพียงไม่มี DN-bit (EIGRP ไม่มี LSA) แต่ EIGRP มี
กลไกป้องกัน Loop ของตัวเองที่ทำหน้าที่คล้ายกันคือ **SoO (Site of Origin)**

### 862.2 ปัญหาที่ SoO แก้: ลูกค้า Multi-Home ด้วย EIGRP

ถ้าลูกค้าไซต์เดียว Multi-Home เข้า PE สองตัว (เหมือนสถานการณ์ WAN-EDGE-1 ↔ SP-PE1/SP-PE3 ใน
Step 863 แต่ใช้ EIGRP แทน BGP) — Route ที่ Site นี้ Advertise ออกไปทาง SP-PE1 อาจถูก SP ส่งกลับมา
เข้า Site เดิมผ่าน SP-PE3 (เพราะ MP-BGP VPNv4 มองว่าเป็น Route ปกติ ไม่รู้ว่า "อย่าส่งกลับไปที่มา")
ทำให้ Site เห็น Route ของตัวเองย้อนกลับมาเป็น External Route ที่ดูสมเหตุสมผล (Loop ไม่ชัดเจนแบบ
OSPF Type 1 vs 3) และอาจสร้าง Suboptimal Route หรือ Micro-Loop ได้

### 862.3 คอนฟิก SoO เพื่อแก้ (ตัวอย่างแนวคิดย่อ)

```
SP-PE1(config)# route-map TAG-SOO-SITE-A permit 10
SP-PE1(config-route-map)# set extcommunity soo 65000:1001
!
SP-PE1(config)# router bgp 65000
SP-PE1(config-router)# address-family ipv4 vrf CUST-ENTLAB
SP-PE1(config-router-af)# neighbor 192.0.2.2 route-map TAG-SOO-SITE-A in
SP-PE1(config-router-af)# exit-address-family
```

บน **SP-PE3** (PE อีกตัวที่ต่อ Site เดียวกัน) ต้อง **ติด SoO Tag เดียวกัน (`65000:1001`)** บน
Neighbor ที่รับ Route จาก Site นี้ด้วย — ผลคือ **ทุก PE ที่เห็น Route ที่มี SoO Tag ตรงกับ Site
ปลายทางที่กำลังจะ Advertise ออกไป จะ "ปฏิเสธ" การส่ง Route นั้นกลับเข้า Site เดิมโดยอัตโนมัติ**
(กลไกคล้าย Split-Horizon แต่ทำงานข้าม PE ผ่าน BGP Extended Community แทนที่จะเป็นแค่ Interface
เดียว) — ป้องกัน Loop ได้แม่นยำกว่า Split-Horizon ปกติมากในสถานการณ์ Multi-PE

> **สรุป**: SoO คือ EIGRP-World Equivalent ของสิ่งที่ Step 863 จะใช้ **RD ต่างกันต่อ Attachment
> Point** แก้ปัญหาคล้ายกันในโลก BGP — ทั้งสองกลไกมีเป้าหมายเดียวกันคือ "บอกให้ระบบรู้ว่า Route นี้
> มาจาก Site ไหน เพื่อไม่ Advertise กลับไปที่เดิม"

---

## Step 863 — Multi-Homing ลูกค้าเข้ากับ PE สองตัว: SP-PE3 และ BGP Path Selection

### 863.1 ทำไมต้อง Multi-Home CE เข้า PE สองตัว

WAN-EDGE-1 (Part 86) ต่อ SP-PE1 เพียงเส้นทางเดียว — ถ้า Link นี้ล่ม หรือ SP-PE1 ล่มทั้งตัว, HQ
จะขาดการเชื่อมต่อ MPLS VPN ทั้งหมด **Multi-Homing** (ต่อ CE เข้า PE 2 ตัวที่ต่างกัน) คือมาตรฐาน
Design สำหรับ Site สำคัญ (HQ/Data Center) เพื่อความ Redundant ระดับ PE ไม่ใช่แค่ระดับ Link — เรา
เพิ่ม **SP-PE3** (PE ตัวใหม่ ต่อกับ SP-P2) เข้าสู่ SP Core เพื่อสอน Pattern นี้

```
                                  SP Core (AS 65000)
                    SP-PE1 ── SP-P1 ── SP-P2 ── SP-PE2 ── REMOTE-SITE-CE
                       │                  │
                       │ (Primary)        │ Gi0/0/3 (100.64.1.12/30)
                       │                  └── SP-PE3 (PE ใหม่)
                       │                       Lo0: 100.64.0.13/32
                  WAN-EDGE-1                    │ (Secondary/Backup)
                  Gi0/0/2 → SP-PE1               │
                  Gi0/0/3 → SP-PE3 ──────────────┘
                  (192.0.2.10/30)
```

### 863.2 IP Addressing เพิ่มเติมสำหรับ SP-PE3

| Device | Interface | IP Address | หมายเหตุ |
|---|---|---|---|
| SP-P2 | GigabitEthernet0/0/3 (→ SP-PE3, ใหม่) | 100.64.1.13/30 | `mpls ip` เปิด |
| SP-PE3 | Loopback0 | 100.64.0.13/32 | MPLS Router-ID / MP-iBGP Source |
| SP-PE3 | GigabitEthernet0/0/1 (Core → SP-P2) | 100.64.1.14/30 | `mpls ip` เปิด |
| SP-PE3 | GigabitEthernet0/0/2 (CE → WAN-EDGE-1, สำรอง) | 192.0.2.9/30 | `vrf forwarding CUST-ENTLAB`, ไม่มี `mpls ip` |
| WAN-EDGE-1 | GigabitEthernet0/0/3 (→ SP-PE3, ใหม่) | 192.0.2.10/30 | Secondary/Backup Uplink |

### 863.3 ปัญหา RD เดิม (`65000:100`) เมื่อ Multi-Home

ถ้า SP-PE3 ใช้ VRF `CUST-ENTLAB` ด้วย **RD เดียวกัน (`65000:100`)** เหมือน SP-PE1/SP-PE2 (ตาม
Step 853 เดิม) — VPNv4 Prefix ที่ SP-PE1 กับ SP-PE3 ทั้งคู่ Advertise สำหรับ `10.10.0.0/16` จะกลาย
เป็น **NLRI เดียวกันเป๊ะ** (`65000:100:10.10.0.0/16`) ผลคือ BGP มองว่าเป็น Prefix ซ้ำกันและเก็บ
แค่ **Best Path เดียว** ทิ้ง Path ที่เหลือทันที — SP-PE2/PE อื่นจะ **ไม่รู้จัก Path สำรองผ่าน
SP-PE3 เลย** แม้ Physical Link จะพร้อมอยู่ก็ตาม (Failover ต้องรอ BGP Re-converge ทั้งกระบวนการ
ใหม่หมด ช้ากว่าที่ควร)

**วิธีแก้ตาม Best Practice** (ตามที่ Part 86 Step 857.4 ทิ้งปมไว้): ให้ SP-PE3 ใช้ **RD ที่ไม่ซ้ำ
กัน** แต่ **RT (Import/Export) ค่าเดิม** เพื่อให้ยังอยู่ใน VPN เดียวกัน (Any-to-Any) แต่ BGP มองว่า
เป็น **Path คนละเส้นทาง** ที่ต้องเก็บทั้งคู่:

```
SP-PE3(config)# vrf definition CUST-ENTLAB
SP-PE3(config-vrf)# description ** Secondary/Backup Attachment Point ของ WAN-EDGE-1 (Multi-Homing) **
SP-PE3(config-vrf)# rd 65000:101
SP-PE3(config-vrf)# address-family ipv4
SP-PE3(config-vrf-af)# route-target export 65000:100
SP-PE3(config-vrf-af)# route-target import 65000:100
SP-PE3(config-vrf-af)# exit-address-family
```

| RD | Export/Import RT | ผล |
|---|---|---|
| SP-PE1: `65000:100` | `65000:100` | Path หลักของ WAN-EDGE-1 |
| SP-PE3: `65000:101` (**ต่างจาก SP-PE1**) | `65000:100` (**เหมือนกัน**) | Path สำรองของ WAN-EDGE-1 — RD ต่างพอให้ BGP เก็บทั้งสอง Path, RT เหมือนพอให้ยังอยู่ใน VPN เดิม |

### 863.4 คอนฟิก eBGP บน WAN-EDGE-1 (Dual-Homed) และควบคุม Path Selection

Problem สำคัญของ Multi-Homing คือ **Asymmetric Routing**: ถ้าไม่ควบคุม, Traffic ขาไปอาจวิ่งผ่าน
SP-PE1 แต่ขากลับวิ่งผ่าน SP-PE3 (Router ทั้งสองฝั่งเลือก Best Path อิสระจากกัน) — แม้จะยัง Reachable
แต่ทำให้ Troubleshoot ยาก, Latency ไม่คงที่, และ Stateful Firewall/NAT อาจ Drop Traffic เพราะเห็น
แค่ครึ่งทาง (Asymmetric Path ผ่าน Firewall คือปัญหาคลาสสิกของ Part 45 Best Practice)

**หลักการแก้ให้ Symmetric**: ควบคุมทั้ง 2 ทิศทางให้เลือก **SP-PE1 เป็น Primary เสมอ**

```
! ===== WAN-EDGE-1: ทิศทาง "ขาเข้า" (เลือก Path ไหนไปยัง REMOTE-SITE-CE) =====
WAN-EDGE-1(config)# route-map PREFER-PE1-IN permit 10
WAN-EDGE-1(config-route-map)# set local-preference 200
WAN-EDGE-1(config)# route-map PREFER-PE3-IN permit 10
WAN-EDGE-1(config-route-map)# set local-preference 100
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/3
WAN-EDGE-1(config-if)# description ** Secondary MPLS L3VPN Access Link to SP-PE3 (Multi-Homing) **
WAN-EDGE-1(config-if)# ip address 192.0.2.10 255.255.255.252
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 192.0.2.9 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 192.0.2.9 description ** eBGP to SP-PE3 (Backup Uplink) **
WAN-EDGE-1(config-router)# neighbor 192.0.2.1 route-map PREFER-PE1-IN in
WAN-EDGE-1(config-router)# neighbor 192.0.2.9 route-map PREFER-PE3-IN in
!
! ===== WAN-EDGE-1: ทิศทาง "ขาออก" (ทำให้ Remote Site เลือกวิ่งเข้า SP-PE1 เป็น Primary) =====
WAN-EDGE-1(config)# route-map PREPEND-TO-PE3 permit 10
WAN-EDGE-1(config-route-map)# set as-path prepend 65001 65001
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 192.0.2.9 route-map PREPEND-TO-PE3 out
WAN-EDGE-1(config-router)# exit
```

| Route-Map | ทิศทาง | ผล |
|---|---|---|
| `PREFER-PE1-IN` (Local-Pref 200) เข้า Neighbor SP-PE1 | Inbound ที่ WAN-EDGE-1 | WAN-EDGE-1 **เลือก Path ผ่าน SP-PE1 เสมอ** เมื่อจะส่ง Traffic ออกไปหา REMOTE-SITE-CE (ตราบใด SP-PE1 Up) |
| `PREPEND-TO-PE3` (AS-Path Prepend 2 ครั้ง) ออก Neighbor SP-PE3 | Outbound จาก WAN-EDGE-1 | Route `10.10.0.0/16` ที่ SP-PE3 เห็นจาก WAN-EDGE-1 มี AS-Path ยาวกว่า Path ที่ SP-PE1 เห็น — ทำให้ **REMOTE-SITE-CE/PE อื่นเลือก Path ผ่าน SP-PE1 เป็น Best Path เสมอ** (AS-Path สั้นกว่า) ตาม BGP Decision Process มาตรฐานจาก [Part 29](part-029-bgp-fundamentals.md) |

ผลลัพธ์: **ทั้งสองทิศทางเลือก SP-PE1 เป็น Primary Path พร้อมกัน** — Traffic วิ่ง Symmetric ตลอด
เวลาที่ SP-PE1 Up และ Failover ไป SP-PE3 อัตโนมัติเมื่อ SP-PE1/Link ล่ม (RD ต่างกันตาม 863.3 ทำให้
Path ผ่าน SP-PE3 พร้อมอยู่ใน BGP Table แล้วตลอดเวลา ไม่ต้องรอ Re-learn)

### 863.5 หมายเหตุ: iBGP VPNv4 Full-Mesh เริ่มไม่ Scale เมื่อมี 3 PE

ตอนนี้ SP Core มี PE 3 ตัว (SP-PE1, SP-PE2, SP-PE3) ทำให้ Full-Mesh MP-iBGP ต้องมี 3 Session
(n(n-1)/2 = 3) — ยังจัดการได้ แต่ถ้า SP ขยายเป็นหลักร้อย PE จำนวน Session จะโตแบบ O(n²) จนดูแลไม่
ได้จริง แนวทาง Production คือใช้ **Route Reflector (RR)**: กำหนด PE (หรือ P Router ที่รัน BGP)
ตัวหนึ่งเป็น RR แล้วให้ PE อื่นทำ iBGP แค่กับ RR ตัวเดียว (`neighbor <RR-ip> route-reflector-client`
สั่งที่ RR) — ลด Session ที่ต้องดูแลจาก O(n²) เหลือ O(n) โดย VPN Route ยังกระจายครบทุก PE เหมือนเดิม
เพราะ RR ทำหน้าที่ "Reflect" Route ให้ Client ทุกตัวแทน Full-Mesh

---

## Step 864 — Internet Access ผ่าน MPLS L3VPN: Route Leaking เทียบกับ Internet Circuit แยก

### 864.1 ปัญหา: VRF ของลูกค้าไม่มี Default Route ไปอินเทอร์เน็ตในตัวเอง

VRF `CUST-ENTLAB` (Part 86) มีแค่ Route ของลูกค้าเอง (ระหว่าง WAN-EDGE-1 ↔ REMOTE-SITE-CE) — ไม่มี
เส้นทางไปอินเทอร์เน็ตเลย เพราะ VRF ถูกออกแบบให้ Isolate จาก Global Table ของ SP โดยเจตนา (Step
853) — ลูกค้าที่ต้องการ Internet Access มีให้เลือก 2 แนวทางหลัก

### 864.2 แนวทางที่ 1: Internet VRF บน SP + Route Leaking

SP สร้าง **VRF `INTERNET`** แยกต่างหาก (มี Default Route ไปยัง Internet Peering ของ SP เอง) แล้ว
"รั่ว" (Leak) เฉพาะ Default Route เข้าไปยัง VRF ของลูกค้าทุกราย ผ่าน RT เฉพาะสำหรับ Internet:

```
! บน SP-PE1
SP-PE1(config)# vrf definition INTERNET
SP-PE1(config-vrf)# rd 65000:999
SP-PE1(config-vrf)# address-family ipv4
SP-PE1(config-vrf-af)# route-target export 65000:999
SP-PE1(config-vrf-af)# exit-address-family
!
SP-PE1(config)# ip route vrf INTERNET 0.0.0.0 0.0.0.0 <Next-Hop ไปยัง Internet Peering ของ SP>
!
SP-PE1(config)# vrf definition CUST-ENTLAB
SP-PE1(config-vrf)# address-family ipv4
SP-PE1(config-vrf-af)# route-target import 65000:999   ! ลูกค้า Import แค่ Default Route จาก INTERNET VRF
SP-PE1(config-vrf-af)# exit-address-family
```

- ลูกค้า **Import** RT `65000:999` เข้าไปเพื่อรับ Default Route แต่ **ไม่ Export** RT ของตัวเอง
  (`65000:100`) กลับให้ VRF `INTERNET` — ป้องกัน Route ภายในของลูกค้ารั่วเข้าไปปนกับ VRF Internet
  ของ SP หรือลูกค้ารายอื่นที่ Import RT เดียวกันนี้ (ทุกลูกค้าที่ซื้อบริการ Internet Add-on จาก SP
  รายนี้จะ Import RT `65000:999` เหมือนกันหมด — Asymmetric RT แบบเดียวกับ Hub-and-Spoke Step 857)

### 864.3 แนวทางที่ 2: Internet Circuit แยกจาก MPLS VPN โดยสิ้นเชิง

หลักสูตรนี้มี **WAN-EDGE-2 ↔ ISP-RTR** ([Part 19](part-019-wan-technologies.md)) เป็น Internet
Circuit แยกอยู่แล้วตั้งแต่ต้น — ลูกค้าใช้ MPLS L3VPN (WAN-EDGE-1 ↔ SP Core) **เฉพาะ Site-to-Site
Traffic ภายในองค์กร** และใช้ WAN-EDGE-2/ISP-RTR/FW-1 (Part 63 NAT) **เฉพาะ Internet-bound
Traffic** — ทั้งสอง Path แยกกันสมบูรณ์ ไม่ต้องพึ่ง SP เลยสำหรับ Internet Access

### 864.4 เปรียบเทียบและคำแนะนำ

| มิติ | (1) Internet VRF + Route Leaking (ผ่าน SP) | (2) Internet Circuit แยก (WAN-EDGE-2/ISP-RTR) |
|---|---|---|
| ความซับซ้อนของ RT Design | สูง (ต้องคุม Asymmetric RT ระวัง Leak ผิดพลาด) | ต่ำ (ไม่เกี่ยว MPLS VPN เลย) |
| Security Inspection รวมศูนย์ที่ SP | ทำได้ (SP อาจเสนอ Managed Firewall/Cloud Security ที่ Internet VRF) | ต้องทำเอง (FW-1 ของ Enterprise เอง) |
| ค่าใช้จ่าย | มักแพงกว่า (SP คิดค่า Internet Add-on ต่อ Mbps) | ถูกกว่า (Internet Circuit ทั่วไปราคาต่ำกว่า MPLS มาก) |
| ความเร็วในการเปลี่ยน ISP | ช้า (ผูกกับ SP รายเดียว) | เร็ว (เปลี่ยน ISP-RTR ได้อิสระ) |
| เข้ากับ SD-WAN/Hybrid WAN ในอนาคต | ยาก (ผูก Internet เข้ากับ MPLS Vendor) | ง่าย (Internet Circuit คือ Transport ที่ SD-WAN ต้องการอยู่แล้ว — Step 867) |

**คำแนะนำของหลักสูตรนี้**: แนวทาง (2) **เหมาะกับ Enterprise ส่วนใหญ่มากกว่า** เพราะยืดหยุ่นกว่า
ราคาถูกกว่า และสอดคล้องกับทิศทาง Hybrid WAN/SD-WAN ที่ Step 867 จะแนะนำ — แนวทาง (1) เหมาะกับ
องค์กรที่ต้องการ **Single Provider Accountability** (SLA เดียวครอบคลุมทั้ง Site-to-Site และ
Internet, มัก Regulated Industry ที่ต้องการ Audit Trail จาก Provider รายเดียว) เท่านั้น

---

## Step 865 — MPLS Traffic Engineering (RSVP-TE): ภาพรวมเชิงแนวคิด

### 865.1 ปัญหาที่ TE แก้: LDP เดินตาม IGP Best Path เสมอ ไม่สนใจ Bandwidth

LDP (Part 86 Step 851) สร้าง LSP ตาม **IGP Best Path เท่านั้น** — ถ้า Link บน Best Path นั้น
Congest (Bandwidth เต็ม) LDP **ไม่มีทางเลี่ยงไปใช้ Link อื่นที่ว่างกว่าได้เลย** แม้จะมี Path สำรอง
อยู่จริงก็ตาม เพราะ LDP ไม่รู้จักแนวคิด "Bandwidth Reservation" ใดๆ

**MPLS TE (Traffic Engineering)** ใช้ **RSVP-TE (Resource Reservation Protocol - Traffic
Engineering)** แทน LDP สำหรับบาง LSP ที่ SP ต้องการควบคุมเส้นทางและจองปริมาณ Bandwidth ไว้ล่วงหน้า
บน Link เฉพาะเจาะจง — ไม่ใช่แค่ "ตาม IGP Cost ต่ำสุด" เหมือน LDP

### 865.2 กลไกทำงานแบบย่อ (Provider-Side Concept)

```
   1) SP กำหนด TE Tunnel: SP-PE1 → SP-PE2 ต้องการ Bandwidth 500 Mbps ตลอดเวลา
   2) RSVP-TE ส่ง PATH Message ไปตามเส้นทางที่เลือก (Explicit Path หรือ CSPF-computed)
      ผ่าน SP-P1 → SP-P2 พร้อมข้อมูล Bandwidth ที่ต้องการ
   3) แต่ละ Hop ตรวจสอบ Bandwidth ที่เหลือบน Link (Available Bandwidth จาก IGP-TE Extension:
      OSPF Opaque LSA Type 10 หรือ IS-IS TE Extension) — ถ้าไม่พอ Reject
   4) ถ้าทุก Hop มี Bandwidth พอ, ส่ง RESV Message ย้อนกลับเพื่อยืนยันการจอง (Reservation)
   5) LSP ที่ได้ (TE Tunnel) การันตี 500 Mbps ตลอดเวลาบน Path นี้ — Traffic อื่นที่ไม่ใช้ TE Tunnel
      นี้ "มองไม่เห็น" Bandwidth ที่ถูกจองไว้เลย (เหมือน Link นั้นแคบลงสำหรับ Traffic ทั่วไป)
```

### 865.3 ทำไม Enterprise WAN Architect ควรเข้าใจ TE แม้จะเป็น Feature ฝั่ง SP

- **ประเมิน SLA ที่ SP เสนอขาย**: ถ้า SP บอกว่า Premium Tier (Step 858) "การันตี Bandwidth" นั่น
  หมายความว่า SP น่าจะใช้ MPLS TE ภายใน Core ของตัวเองเพื่อ Reserve เส้นทางให้ Traffic Tier นั้น
  จริง — ถ้า SP ไม่มี TE เลย คำว่า "Guaranteed Bandwidth" อาจเป็นแค่ Best-Effort ที่ Mark EXP สูง
  เฉยๆ ไม่มีการันตีทางเทคนิคจริง
- **เข้าใจ Fast Reroute (FRR)**: RSVP-TE รองรับ **MPLS TE Fast Reroute** ที่สร้าง Backup LSP
  ล่วงหน้า (Pre-computed) ทำให้ Failover ระดับ 50ms เมื่อ Link ล่ม — เร็วกว่า IGP/LDP Re-converge
  ธรรมดามาก ถ้า SP รับรอง FRR ในสัญญาถือเป็นจุดขายสำคัญสำหรับ Traffic Real-time
- **Interpret เมื่อ SP Blame ปัญหา Congestion**: ถ้า SP บอกว่า "Core ไม่ Congest เพราะมี TE" แต่
  ลูกค้ายังเห็น Latency สูงผ่าน [Part 87 Step 869](part-087-mpls-l3vpn-advanced-wan-integration.md)
  IP SLA — เป็นหลักฐานที่ใช้ต่อรอง/เรียกร้อง SLA Credit ได้อย่างมีข้อมูลสนับสนุน

> **ขอบเขต CCIE Enterprise Infrastructure**: การ Config RSVP-TE เต็มรูปแบบ (`mpls traffic-eng`,
> Explicit Path, CSPF) เป็นเนื้อหาของ CCIE Service Provider โดยเฉพาะ — หลักสูตรนี้ให้แค่ระดับ
> Concept ที่ต้องรู้เพื่อประเมิน SP Contract เท่านั้น ไม่ลง Config เต็มรูปแบบ

---

## Step 866 — Carrier Supporting Carrier (CsC) และ Inter-AS MPLS VPN

### 866.1 เมื่อไหร่ที่ MPLS VPN ต้องข้าม SP มากกว่า 1 ราย

สถานการณ์ที่พบจริงในองค์กรขนาดใหญ่:

- **M&A (Merger & Acquisition)**: บริษัทควบรวมกัน แต่ Site เดิมของแต่ละบริษัทซื้อ MPLS VPN จาก
  SP คนละราย — ต้องเชื่อม MPLS VPN ทั้งสองเข้าด้วยกันโดยไม่ทุบทิ้งของเดิมทั้งหมด
- **Global Enterprise ใช้ Regional SP หลายราย**: SP รายหนึ่งอาจไม่มี Coverage ครบทุกประเทศ ต้องใช้
  SP ภูมิภาคต่างๆ ที่ Peer กันเอง (Inter-Provider)
- **Enterprise กลายเป็น "Carrier" เอง (CsC)**: บริษัทลูก/Subsidiary ที่มี MPLS Network ของตัวเอง
  (เช่น Telco ลูกในกลุ่ม) ต้องซื้อ "Transport เปล่า" จาก SP รายใหญ่กว่าเพื่อเชื่อม Site ของตัวเอง
  ข้ามเมือง — SP มองบริษัทลูกนี้เป็น "ลูกค้าที่เป็น Carrier" (Carrier Supporting Carrier)

### 866.2 Carrier Supporting Carrier (CsC) — ภาพรวม

```
   Enterprise-as-Carrier Site A          SP รายใหญ่ (Backbone Transport)      Enterprise-as-Carrier Site B
   ┌─────────────────┐                  ┌──────────────────────────┐         ┌─────────────────┐
   │ CsC-CE (ลูกค้า    │── labeled BGP ──►│  SP-PE (ทำหน้าที่ CsC-PE) │◄── labeled BGP ──│ CsC-CE            │
   │ ที่มี MPLS ของตัวเอง)│   + IGP/LDP     │   ส่ง Label ของลูกค้า      │                 │                   │
   └─────────────────┘                  │   ผ่าน Core โดยไม่ต้องรู้จัก│                 └─────────────────┘
                                          │   VPN ภายในของลูกค้าเลย   │
                                          └──────────────────────────┘
```

จุดต่างจาก MPLS L3VPN ปกติ (Step 852): CE ที่เป็น CsC-CE **ต้องเปิด `mpls ip` และรัน Label**
กับ SP-PE (ไม่ใช่ Plain IP เหมือน CE ทั่วไป) — เพราะ SP ต้องส่ง **Labeled Route** (ไม่ใช่แค่ IP
Route ธรรมดา) ข้าม Backbone ของตัวเองไปให้ CsC-CE อีกฝั่ง (ใช้ `neighbor <x> send-label` ใน BGP
ระหว่าง CsC-CE ↔ SP-PE) — SP ทำหน้าที่ "ขนส่ง Label" เป็นบริการ ไม่ใช่ "เป็นเจ้าของ VPN" แบบ
MPLS L3VPN ปกติ

### 866.3 Inter-AS MPLS VPN — Option A/B/C

| Option | ชื่อเต็ม | กลไก | ข้อดี | ข้อเสีย |
|---|---|---|---|---|
| **Option A** | VRF-to-VRF (Back-to-Back) | ASBR ของ SP ทั้งสองฝั่งต่อกันแบบ **VRF-Lite ธรรมดา** (แต่ละ VRF ทำตัวเป็น "CE" ให้อีกฝั่ง) — ไม่มี Label ข้าม AS เลย | ง่ายที่สุด, ปลอดภัยสุด (ไม่แชร์ Label Space ข้าม AS) | Scale แย่สุด — ต้องมี Sub-Interface/VRF แยกต่อ Customer VPN 1 คู่เสมอ |
| **Option B** | eBGP Redistribution of Labeled VPNv4 | ASBR ทั้งสองฝั่งแลก **VPNv4 Route ที่มี Label ติดมาด้วย** ผ่าน eBGP ตรงระหว่าง ASBR (Next-Hop ถูก Rewrite เป็น ASBR เอง) | Scale ดีกว่า A มาก (ไม่ต้องแยก VRF ต่อ Customer) | ASBR ต้องเก็บ VPNv4 Route **ทุก VPN ที่ข้าม AS** (Load บน ASBR สูง) |
| **Option C** | Multihop MP-eBGP ระหว่าง RR ข้าม AS | RR ของแต่ละ AS คุยกันตรงผ่าน **Multihop eBGP VPNv4** (ASBR แค่ Redistribute Labeled IPv4 Loopback ของ PE ข้าม AS พอ ไม่ต้องเห็น VPN Route เลย) | Scale ดีที่สุด (ASBR เบาที่สุด, RR จัดการ VPN Route แทน) | ซับซ้อนสุด, ต้อง Trust กันสูงระหว่าง 2 AS (Next-Hop-Unchanged, Label ยังต้อง Reachable ข้าม AS) |

### 866.4 คำแนะนำเชิง Design

- **M&A ระยะสั้น (ต้องเชื่อมเร็ว, จำนวน VPN น้อย)**: เลือก **Option A** ก่อนเสมอ — เสี่ยงน้อยสุด,
  Debug ง่ายสุด แม้จะ Scale ไม่ดี
- **Global Enterprise ระยะยาว (VPN จำนวนมาก ข้ามหลาย SP ต่อเนื่อง)**: ควรผลักดันให้ SP ทั้งสองฝั่ง
  ใช้ **Option C** เพื่อ Scale ในระยะยาว แม้ Setup ครั้งแรกจะซับซ้อนกว่า
- ไม่ว่าเลือก Option ใด, **Enterprise WAN Architect ควรถามหา SLA ของ "Inter-AS Link" เป็นพิเศษ**
  เพราะเป็นจุดที่ Failure Domain ของ 2 SP มาบรรจบกัน มักเป็นจุดอ่อนที่สุดของ End-to-End SLA ทั้งหมด

---

## Step 867 — Migration Framework: MPLS L3VPN สู่ SD-WAN

### 867.1 ทวนความจากหลักสูตรเดิม

[Part 39](part-039-sd-wan-fundamentals.md) แนะนำ SD-WAN Fundamentals และ [Part 76](part-076-sdwan-advanced-design.md)
ขยายเป็น Advanced Design ไปแล้ว — Part นี้ (86-87) แสดงให้เห็นความซับซ้อนจริงของ MPLS L3VPN ทั้ง
ฝั่ง SP — คำถามที่ CCIE Enterprise Infrastructure ต้องตอบให้ลูกค้าได้คือ **"เมื่อไหร่ควรอยู่กับ
MPLS ต่อ และเมื่อไหร่ควรย้ายไป SD-WAN-over-Internet"**

### 867.2 กรอบการตัดสินใจ (Decision Framework)

| เงื่อนไข | MPLS L3VPN ยังคือคำตอบที่ถูก | SD-WAN-over-Internet เพียงพอ/ดีกว่า |
|---|---|---|
| ต้องการ SLA ที่รับประกันด้วยสัญญาจริง (Latency/Jitter/Packet Loss การันตี) | ✅ MPLS TE (Step 865) + CoS Tier (Part 86 Step 858) ให้ SLA จริงที่ฟ้องร้องได้ | ⚠️ Internet ไม่มีใครการันตี SLA ได้ 100% (Best-Effort โดยธรรมชาติ) |
| ข้อกำหนดกฎหมาย/Regulatory (การเงิน, สุขภาพ, ราชการ) ต้องใช้ Private Circuit เท่านั้น | ✅ หลายกฎหมายยังกำหนดให้ต้องมี Dedicated Private Link | ❌ Internet-based มักไม่ผ่าน Compliance บางประเภท |
| Real-time Application ที่ Sensitive สูง (Trading Platform, SCADA/OT Control) | ✅ Deterministic Latency สำคัญกว่าต้นทุน | ⚠️ Jitter/Loss ของ Internet อาจกระทบ Real-time Control Loop |
| จำนวน Site มาก, กระจายทั่วโลก, ต้องขยาย Site ใหม่บ่อย | ⚠️ MPLS Provisioning ใช้เวลาหลักสัปดาห์-เดือนต่อ Site ใหม่ | ✅ SD-WAN Zero-Touch Provisioning ใช้เวลาหลักชั่วโมง-วัน |
| ต้นทุนต่อ Mbps | ⚠️ แพงกว่า Internet มาก (หลักสิบเท่า) | ✅ ถูกกว่ามาก — เพิ่ม Bandwidth ได้อิสระ |
| Application-Aware Routing ข้าม Transport หลายเส้น (MPLS+Internet+4G/5G) | ⚠️ MPLS เดี่ยวไม่มี Path Selection ในตัว | ✅ SD-WAN ทำ Per-App Path Selection ข้าม Transport ได้โดยธรรมชาติ |

### 867.3 Hybrid WAN — คำตอบที่ Enterprise ส่วนใหญ่เลือกจริงในช่วง Transition

แทนที่จะเลือก "ทิ้ง MPLS ทั้งหมดทันที" หรือ "อยู่กับ MPLS ตลอดไป" องค์กรส่วนใหญ่เลือก **Hybrid
WAN**: รัน MPLS L3VPN (Part 86-87) และ SD-WAN-over-Internet ([Part 76](part-076-sdwan-advanced-design.md))
**พร้อมกันเป็น Transport คู่ขนาน** โดยให้ SD-WAN Overlay ทำหน้าที่ Application-Aware Path Selection
เลือกว่า Traffic ประเภทไหนควรวิ่งผ่าน MPLS (Latency-Sensitive, ต้องการ SLA) และประเภทไหนวิ่งผ่าน
Internet (Bulk/Non-Critical) — MPLS กลายเป็นแค่ **"Transport ตัวหนึ่ง" ในหลาย Transport ที่ SD-WAN
Overlay มองเห็นเท่าเทียมกัน** ไม่ใช่ "WAN ทั้งหมดขององค์กร" อีกต่อไป

```
                          SD-WAN Overlay (Fabric — Part 76)
                    ┌─────────────────────────────────────────┐
                    │  Application-Aware Path Selection Policy  │
                    └───────┬─────────────────────────┬────────┘
                            │                          │
                   Transport 1: MPLS L3VPN     Transport 2: Internet (Broadband/4G)
                   (Part 86-87 — WAN-EDGE-1     (WAN-EDGE-2 ↔ ISP-RTR — Part 19/63)
                    ↔ SP-PE1 ↔ SP Core)
                            │                          │
                   ── Voice/ERP/Trading:            ── Bulk Backup/Guest/Web Browsing:
                      บังคับใช้ MPLS เสมอ              ใช้ Internet เสมอ (ประหยัด MPLS Bandwidth)
```

**Timeline การเปลี่ยนผ่านทั่วไป**: (1) Phase 1 — ติดตั้ง SD-WAN Overlay คู่กับ MPLS เดิม ยังไม่ยกเลิก
สัญญา MPLS ใดๆ (2) Phase 2 — ค่อยๆ ย้าย Application ที่ไม่ Critical ไปวิ่งบน Internet ผ่าน SD-WAN
Policy ทดสอบผลกระทบทีละ App (3) Phase 3 — ลด Bandwidth ของ MPLS Circuit ลง (ประหยัดค่าใช้จ่าย) เมื่อ
มั่นใจว่า Internet Path รองรับ Traffic ส่วนใหญ่ได้ (4) Phase 4 (บาง Site เท่านั้น) — ยกเลิก MPLS
Circuit ทั้งหมดสำหรับ Site ที่ไม่มีข้อกำหนด Regulatory/SLA พิเศษ คงไว้เฉพาะ Site ที่จำเป็นจริง (Data
Center หลัก, Site ที่มีข้อกำหนดกฎหมาย)

---

## Step 868 — MPLS L3VPN และ Multicast: Default MDT ข้าม SP Core ตัวจริง

### 868.1 ต่อยอดจาก MVPN Rosen GRE (Part 83 Step 822)

[Part 83 Step 822](part-083-enterprise-multicast-deep-dive.md#step-822--multicast-vpn-mvpn-rosen-gre-model-config-เต็มรูปแบบ)
สอน MVPN Rosen GRE เต็มรูปแบบไปแล้ว แต่ระบุชัดว่าใช้ **VRF-Lite (WAN-EDGE-1/WAN-EDGE-2 เป็น "PE"
เอง)** ยังไม่มี MPLS Core จริงคั่นกลาง — ตอนนี้ Part 86-87 สร้าง SP Core จริงแล้ว (SP-P1/SP-P2 เป็น
LSR จริง, LDP จริง) จึงถึงเวลาย้าย MVPN Rosen GRE ให้วิ่งผ่าน **VRF `CUST-ENTLAB` บน SP-PE1/SP-PE2**
(ตำแหน่งที่ VRF อยู่จริงตาม Step 853) แทน

### 868.2 ข้อสังเกตสำคัญ: PIM ยัง "Hop-by-Hop" แม้ Data Plane จะ Label-Switched แล้ว

Unicast VPNv4 Traffic (Part 86 Step 855) ถูก Label-Switched ข้าม SP-P1/SP-P2 โดยที่ P Router
**ไม่ต้องรัน PIM หรือรู้จัก Multicast เลย** — แต่ **Default MDT GRE Tunnel ของ MVPN Rosen ยังต้อง
พึ่ง PIM Join/Prune แบบ Hop-by-Hop ในทุก Router ที่อยู่บนเส้นทาง (รวม SP-P1/SP-P2 ด้วย)** เพราะ PIM
Signaling ไม่ใช่ Label Switching — **นี่คือเหตุผลที่แท้จริงที่ NG-MVPN/BGP-based (Part 83 Step
823) ถูกคิดค้นขึ้น**: เพื่อกำจัด Requirement นี้ ให้ P Router ไม่ต้องรัน PIM เลยแม้แต่ตัวเดียว
(ใช้ MP-BGP MDT-SAFI แทนทั้งหมด) — Step นี้จะแสดงให้เห็น Requirement ของ Rosen GRE Model ชัดเจน
ในทางปฏิบัติจริงเป็นครั้งแรกในหลักสูตร (Part 83 ไม่มี P Router คั่นกลางให้เห็นปัญหานี้)

```
   Data Plane ของ GRE MDT Packet (Encapsulate แล้ว, Dst = Loopback0 ของ PE ปลายทาง):
        ── Label-Switched ปกติผ่าน SP-P1/SP-P2 (ใช้ LSP เดิมจาก LDP, Step 851, ไม่ต้องคอนฟิกเพิ่ม)

   Control Plane ของ PIM Join/Prune (สร้าง (S,G) Tree ให้ MDT Tunnel):
        ── ต้องมี "ip pim sparse-mode" + "ip multicast-routing" บน SP-P1/SP-P2 ด้วย!
           (Native Multicast จริงในทุก Hop ของ P-Network — Requirement ที่ Rosen GRE เพิ่มเข้ามา)
```

### 868.3 คอนฟิกเต็มรูปแบบ — Provider PIM Domain (Global Table ของ SP Core)

```
! ===== SP-PE1 =====
SP-PE1(config)# ip multicast-routing
SP-PE1(config)# ip pim rp-address 100.64.0.1 10          ! RP ของ Provider Domain = SP-P1 Loopback0
SP-PE1(config)# interface GigabitEthernet0/0/1
SP-PE1(config-if)# ip pim sparse-mode
SP-PE1(config-if)# exit
SP-PE1(config)# interface Loopback0
SP-PE1(config-if)# ip pim sparse-mode
SP-PE1(config-if)# exit

! ===== SP-P1 (P Router — ต้องเปิด PIM เพิ่มเป็นครั้งแรกในหลักสูตรนี้) =====
SP-P1(config)# ip multicast-routing
SP-P1(config)# ip pim rp-address 100.64.0.1 10
SP-P1(config)# interface range GigabitEthernet0/0/1 , GigabitEthernet0/0/2 , Loopback0
SP-P1(config-if-range)# ip pim sparse-mode
SP-P1(config-if-range)# exit

! ===== SP-P2 และ SP-PE2 — รูปแบบเดียวกันทุกอินเทอร์เฟซฝั่ง Core =====
```

### 868.4 คอนฟิกเต็มรูปแบบ — VRF `CUST-ENTLAB` Multicast (Rosen GRE, ตำแหน่งจริงบน PE)

```
! ===== SP-PE1 =====
SP-PE1(config)# vrf definition CUST-ENTLAB
SP-PE1(config-vrf)# address-family ipv4
SP-PE1(config-vrf-af)# mdt default 239.192.2.1
SP-PE1(config-vrf-af)# mdt data 232.2.1.0 0.0.0.255 threshold 500
SP-PE1(config-vrf-af)# exit-address-family
SP-PE1(config-vrf)# exit
!
SP-PE1(config)# ip multicast-routing vrf CUST-ENTLAB
SP-PE1(config)# ip pim vrf CUST-ENTLAB rp-address 1.1.1.21   ! Customer RP = WAN-EDGE-1 Loopback0
SP-PE1(config)# interface GigabitEthernet0/0/2
SP-PE1(config-if)# ip pim sparse-mode
SP-PE1(config-if)# exit

! ===== SP-PE2 — Config คู่กันทุกตัวอักษร =====
SP-PE2(config)# vrf definition CUST-ENTLAB
SP-PE2(config-vrf)# address-family ipv4
SP-PE2(config-vrf-af)# mdt default 239.192.2.1
SP-PE2(config-vrf-af)# mdt data 232.2.1.0 0.0.0.255 threshold 500
SP-PE2(config-vrf-af)# exit-address-family
SP-PE2(config-vrf)# exit
SP-PE2(config)# ip multicast-routing vrf CUST-ENTLAB
SP-PE2(config)# ip pim vrf CUST-ENTLAB rp-address 1.1.1.21
SP-PE2(config)# interface GigabitEthernet0/0/2
SP-PE2(config-if)# ip pim sparse-mode

! ===== WAN-EDGE-1 (CE — เปิด PIM ฝั่งลูกค้าล้วนๆ ไม่รู้จัก MPLS/VRF เลยเหมือนเดิม) =====
WAN-EDGE-1(config)# ip multicast-routing
WAN-EDGE-1(config)# ip pim rp-address 1.1.1.21 20
WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# ip pim sparse-mode
WAN-EDGE-1(config-if)# exit
WAN-EDGE-1(config)# interface Loopback0
WAN-EDGE-1(config-if)# ip pim sparse-mode

! ===== REMOTE-SITE-CE — รูปแบบเดียวกัน (ชี้ RP ไปยัง 1.1.1.21 เดียวกัน) =====
REMOTE-SITE-CE(config)# ip multicast-routing
REMOTE-SITE-CE(config)# ip pim rp-address 1.1.1.21 20
REMOTE-SITE-CE(config)# interface GigabitEthernet0/0/1
REMOTE-SITE-CE(config-if)# ip pim sparse-mode
```

### 868.5 Verify: Default MDT Tunnel ถูก Label-Switched จริงข้าม SP-P1/SP-P2

```
SP-PE1# show ip pim vrf CUST-ENTLAB mdt interface
Interface   Source            Group             VRF
Tunnel0     100.64.0.11       239.192.2.1        CUST-ENTLAB

SP-PE1# show mpls forwarding-table 100.64.0.12
Local  Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label  Label      or Tunnel Id     Switched      interface
17     19         100.64.0.12/32   84520         Gi0/0/1    100.64.1.2
                              ▲
                              └── Bytes Switched เพิ่มขึ้นจริง — พิสูจน์ว่า GRE MDT Packet (สร้างจาก
                                  Tunnel0 ปลายทาง Loopback0 ของ SP-PE2) ถูก Label-Switched ผ่าน LSP
                                  เส้นเดิมที่ LDP สร้างให้ Unicast VPNv4 (Step 851/855) ไม่ต้องสร้าง
                                  LSP ใหม่แยกต่างหากสำหรับ Multicast เลย

SP-P1# show ip pim neighbor
Neighbor          Interface                Uptime/Expires    Ver   DR
100.64.1.1         GigabitEthernet0/0/1     00:10:22/00:01:40 v2    1 / DR
100.64.1.6         GigabitEthernet0/0/2     00:10:15/00:01:35 v2    1 / DR
                              ▲
                              └── SP-P1 (P Router) มี PIM Neighbor จริง — ยืนยัน Requirement 868.2
                                  ว่า Rosen GRE ต้องมี Native Multicast ทุก Hop ของ Core จริง
```

---

## Step 869 — Verification จากมุมมองลูกค้า: IP SLA End-to-End Monitoring

### 869.1 ปัญหา: ลูกค้ามองไม่เห็นข้างใน SP Core เลย

ตลอด Part 86-87 เราคอนฟิกและ Verify จากมุมมอง **SP** (`show mpls forwarding-table`, `show ip bgp
vpnv4`, `show ip ospf sham-links` ฯลฯ) — แต่ **ลูกค้าตัวจริง (WAN-EDGE-1) ไม่มีสิทธิ์ Login เข้า
SP-PE1/SP-P1/SP-P2 เลย** สิ่งที่ WAN-EDGE-1 มองเห็นได้มีแค่ **ผลลัพธ์ปลายทาง** (Reachability/
Latency/Jitter/Loss ไปยัง REMOTE-SITE-CE) — [Part 32 Step 314](part-032-policy-based-routing.md)
สอน IP SLA + Track Object พื้นฐานไว้แล้วสำหรับ PBR Failover — Step นี้ขยายทักษะเดิมไปใช้เป็น
**เครื่องมือหลักตรวจสอบ SLA ที่ SP ขายให้จริง**

### 869.2 IP SLA `icmp-echo` — วัด Reachability/RTT พื้นฐาน

```
WAN-EDGE-1(config)# ip sla 10
WAN-EDGE-1(config-ip-sla)# icmp-echo 172.30.0.1 source-interface Loopback0
WAN-EDGE-1(config-ip-sla-echo)# frequency 30
WAN-EDGE-1(config-ip-sla-echo)# threshold 100
WAN-EDGE-1(config-ip-sla-echo)# timeout 1000
WAN-EDGE-1(config-ip-sla-echo)# exit
WAN-EDGE-1(config)# ip sla schedule 10 life forever start-time now
```

### 869.3 IP SLA `udp-jitter` — วัด Jitter/Packet Loss/MOS เทียบกับ SLA ของ Voice Tier

ตาม CoS Tier ของ SP (Part 86 Step 858) ที่ขาย "Premium Tier" สำหรับ Voice — ต้อง Verify ว่า Jitter
จริงอยู่ในเกณฑ์ที่สัญญาไว้ (เช่น < 10ms) ด้วย `udp-jitter` (ต้องมี **SLA Responder** ที่ปลายทางเพื่อ
ผลลัพธ์แม่นยำ — คำสั่งเดียว ไม่ต้องมี Config Operation เต็มรูปแบบฝั่ง Responder):

```
! บน REMOTE-SITE-CE (ปลายทาง — เปิด Responder เท่านั้น)
REMOTE-SITE-CE(config)# ip sla responder

! บน WAN-EDGE-1 (ต้นทาง)
WAN-EDGE-1(config)# ip sla 11
WAN-EDGE-1(config-ip-sla)# udp-jitter 172.30.0.1 16384 source-interface Loopback0 codec g711alaw
WAN-EDGE-1(config-ip-sla-jitter)# frequency 60
WAN-EDGE-1(config-ip-sla-jitter)# exit
WAN-EDGE-1(config)# ip sla schedule 11 life forever start-time now
!
WAN-EDGE-1(config)# track 10 ip sla 10 reachability
WAN-EDGE-1(config-track)# delay down 10 up 5
```

### 869.4 Verify และตีความผลเทียบกับ SLA ที่สัญญาไว้

```
WAN-EDGE-1# show ip sla statistics 11
IPSLA operation id: 11
   Latest RTT: 14 milliseconds
   Latest operation start time: 14:22:05.104
   Latest operation return code: OK
   RTT Values:
     Number Of RTT: 10       RTT Min/Avg/Max: 12/14/18 milliseconds
   Latency one-way time:
     Number of Latency one-way Samples: 10
     Source to Destination Latency one way Min/Avg/Max: 6/7/9 milliseconds
     Destination to Source Latency one way Min/Avg/Max: 6/7/9 milliseconds
   Jitter Time:
     Number of SD Jitter Samples: 9
     Source to Destination Jitter Min/Avg/Max: 1/2/4 milliseconds
     Number of DS Jitter Samples: 9
     Destination to Source Jitter Min/Avg/Max: 1/2/3 milliseconds
   Packet Loss Values:
     Loss Source to Destination: 0        Loss Destination to Source: 0
     Out Of Sequence: 0   Tail Drop: 0   Packet Late Arrival: 0   Packet Skipped: 0
```

| Metric ที่วัดได้ | ค่าที่ SP สัญญาไว้ (Premium Tier ตัวอย่าง) | ผลจริง | สรุป |
|---|---|---|---|
| RTT/Latency | ≤ 50 ms One-Way | 7 ms One-Way | ✅ ผ่านเกณฑ์มาก |
| Jitter | ≤ 10 ms | 2-4 ms | ✅ ผ่านเกณฑ์ |
| Packet Loss | ≤ 0.5% | 0% | ✅ ผ่านเกณฑ์ |

> **ประโยชน์เชิงธุรกิจ**: ผลลัพธ์ IP SLA เก็บสะสมนี้คือ **หลักฐานเดียวที่ลูกค้ามีในมือ** สำหรับ
> เรียกร้อง SLA Credit หาก SP ทำผิดสัญญา (เช่น Latency พุ่งสูงต่อเนื่องเกินเกณฑ์) — ผูก `track`
> Object เข้ากับ EEM Script หรือ Syslog/SNMP Trap เพื่อแจ้งเตือนทันทีที่ SLA เริ่มเสีย ไม่ต้องรอ
> ผู้ใช้ Complain ก่อน (แนวคิดเดียวกับ Part 32 Step 320 Failover เพียงเปลี่ยนจาก "Failover Action"
> เป็น "Alerting Action")

---

## Step 870 — Lab เต็มรูปแบบ: Multi-Homing + OSPF Sham-Link + IP SLA ครบวงจร

### 870.1 ภาพรวม Lab นี้ — แบ่งเป็น 3 ส่วนที่ประกอบกันเป็นภาพเดียว

```
   Lab A: BGP Multi-Homing            Lab B: OSPF + Sham-Link              Lab C: IP SLA
   (ต่อยอด Part 86 จริง)              (Alternative PE-CE Protocol           (ทำงานได้ไม่ว่า Lab A/B
   WAN-EDGE-1 ↔ SP-PE1 + SP-PE3        สำหรับ REMOTE-SITE-CE + Backdoor)     ใช้ Protocol ไหนก็ตาม)
```

### 870.2 Running-Config — WAN-EDGE-1 (Lab A: Dual-Homed CE)

```
hostname WAN-EDGE-1
!
interface Loopback0
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/2
 description ** Primary MPLS L3VPN Access Link to SP-PE1 **
 ip address 192.0.2.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/3
 description ** Secondary/Backup MPLS L3VPN Access Link to SP-PE3 (Multi-Homing) **
 ip address 192.0.2.10 255.255.255.252
 no shutdown
!
route-map PREFER-PE1-IN permit 10
 set local-preference 200
!
route-map PREFER-PE3-IN permit 10
 set local-preference 100
!
route-map PREPEND-TO-PE3 permit 10
 set as-path prepend 65001 65001
!
router bgp 65001
 bgp router-id 1.1.1.21
 neighbor 192.0.2.1 remote-as 65000
 neighbor 192.0.2.1 description ** eBGP to Provider PE SP-PE1 (Primary) **
 neighbor 192.0.2.1 route-map PREFER-PE1-IN in
 neighbor 192.0.2.9 remote-as 65000
 neighbor 192.0.2.9 description ** eBGP to Provider PE SP-PE3 (Backup) **
 neighbor 192.0.2.9 route-map PREFER-PE3-IN in
 neighbor 192.0.2.9 route-map PREPEND-TO-PE3 out
 network 10.10.0.0 mask 255.255.0.0
!
end
```

### 870.3 Running-Config — SP-PE3 (Lab A: PE ตัวใหม่)

```
hostname SP-PE3
!
vrf definition CUST-ENTLAB
 description ** Secondary/Backup Attachment Point ของ WAN-EDGE-1 (Multi-Homing) **
 rd 65000:101
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface Loopback0
 ip address 100.64.0.13 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Core Link to SP-P2 **
 ip address 100.64.1.14 255.255.255.252
 mpls ip
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** CE-Facing Access Link to WAN-EDGE-1 (Secondary) **
 vrf forwarding CUST-ENTLAB
 ip address 192.0.2.9 255.255.255.252
 no shutdown
!
router ospf 100
 router-id 100.64.0.13
 network 100.64.0.13 0.0.0.0 area 0
 network 100.64.1.12 0.0.0.3 area 0
!
router bgp 65000
 bgp router-id 100.64.0.13
 neighbor 100.64.0.11 remote-as 65000
 neighbor 100.64.0.11 description ** MP-iBGP VPNv4 to SP-PE1 **
 neighbor 100.64.0.11 update-source Loopback0
 neighbor 100.64.0.12 remote-as 65000
 neighbor 100.64.0.12 description ** MP-iBGP VPNv4 to SP-PE2 **
 neighbor 100.64.0.12 update-source Loopback0
 !
 address-family vpnv4
  neighbor 100.64.0.11 activate
  neighbor 100.64.0.11 send-community extended
  neighbor 100.64.0.12 activate
  neighbor 100.64.0.12 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUST-ENTLAB
  neighbor 192.0.2.10 remote-as 65001
  neighbor 192.0.2.10 description ** eBGP to Customer CE WAN-EDGE-1 (Backup Path) **
  neighbor 192.0.2.10 activate
 exit-address-family
!
end
```

> **หมายเหตุ SP-P2**: ต้องเพิ่ม `interface GigabitEthernet0/0/3` (`100.64.1.13/30`, `mpls ip`) และ
> `network 100.64.1.12 0.0.0.3 area 0` เข้าไปใน `router ospf 100` เดิม เพื่อรับ Link ใหม่ไปยัง
> SP-PE3 — ส่วนที่เหลือของ SP-P2 ไม่เปลี่ยนแปลงจาก Part 86

### 870.4 Running-Config — REMOTE-SITE-CE และ SP-PE2 (Lab B: OSPF + Sham-Link)

```
! ===== REMOTE-SITE-CE: เปลี่ยนจาก eBGP (Part 86) เป็น OSPF PE-CE =====
hostname REMOTE-SITE-CE
!
interface Loopback0
 ip address 172.30.0.1 255.255.255.255
!
interface Loopback1
 ip address 172.30.10.1 255.255.255.0
!
interface GigabitEthernet0/0/1
 description ** MPLS L3VPN Access Link to SP-PE2 (OSPF PE-CE) **
 ip address 192.0.2.6 255.255.255.252
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Backdoor Leased Line to WAN-EDGE-1 (Customer-owned, pre-existing) **
 ip address 10.99.0.2 255.255.255.252
 no shutdown
!
router ospf 3
 router-id 172.30.0.1
 network 172.30.0.1 0.0.0.0 area 0
 network 172.30.10.0 0.0.0.255 area 0
 network 192.0.2.4 0.0.0.3 area 0
 network 10.99.0.0 0.0.0.3 area 0
!
end

! ===== SP-PE2: เพิ่ม OSPF PE-CE + Sham-Link (แทน eBGP Address-Family เดิมสำหรับ REMOTE-SITE-CE) =====
SP-PE2(config)# interface Loopback100
SP-PE2(config-if)# vrf forwarding CUST-ENTLAB
SP-PE2(config-if)# ip address 100.64.100.12 255.255.255.255
SP-PE2(config-if)# exit
!
SP-PE2(config)# router bgp 65000
SP-PE2(config-router)# address-family ipv4 vrf CUST-ENTLAB
SP-PE2(config-router-af)# network 100.64.100.12 mask 255.255.255.255
SP-PE2(config-router-af)# redistribute ospf 2 match internal external 1 external 2
SP-PE2(config-router-af)# exit-address-family
!
SP-PE2(config)# router ospf 2 vrf CUST-ENTLAB
SP-PE2(config-router)# router-id 100.64.0.12
SP-PE2(config-router)# area 0 sham-link 100.64.100.12 100.64.100.11
SP-PE2(config-router)# redistribute bgp 65000 subnets
SP-PE2(config-router)# network 192.0.2.4 0.0.0.3 area 0
SP-PE2(config-router)# exit
```

> **WAN-EDGE-1 ในฉากของ Lab B**: ต้องเปลี่ยนมาใช้ OSPF Area 0 กับ SP-PE1 ด้วย (`router ospf 3`
> รูปแบบเดียวกับ REMOTE-SITE-CE ข้างบน) พร้อม `interface GigabitEthernet0/0/4` เชื่อม Backdoor Link
> ไปยัง REMOTE-SITE-CE (`10.99.0.1/30`) — และ SP-PE1 ต้องมี `Loopback100 (100.64.100.11)` +
> `router ospf 2 vrf CUST-ENTLAB` แบบเดียวกับ SP-PE2 ข้างบน (สลับ Neighbor ของ Sham-Link) — Lab B
> นี้เป็น **Design ทางเลือกคู่ขนาน** กับ Lab A (ไม่ได้รันพร้อมกันในอุปกรณ์จริงชุดเดียว) ตามที่ Step
> 861 อธิบายไว้

### 870.5 Running-Config — IP SLA Monitoring (Lab C: ใช้ได้กับทั้ง Lab A และ Lab B)

```
! บน WAN-EDGE-1
ip sla 10
 icmp-echo 172.30.0.1 source-interface Loopback0
 frequency 30
 threshold 100
 timeout 1000
ip sla schedule 10 life forever start-time now
!
ip sla 11
 udp-jitter 172.30.0.1 16384 source-interface Loopback0 codec g711alaw
 frequency 60
ip sla schedule 11 life forever start-time now
!
track 10 ip sla 10 reachability
 delay down 10 up 5
!
! บน REMOTE-SITE-CE
ip sla responder
```

### 870.6 Verification Checklist ครบวงจรของ Lab นี้

```
! (1) Multi-Homing: WAN-EDGE-1 ต้องเห็น BGP Neighbor ทั้งสองเส้นทาง Up
WAN-EDGE-1# show ip bgp summary | include 192.0.2
192.0.2.1       4 65000     120     118       12    0    0 00:30:11        1
192.0.2.9       4 65000      45      44       12    0    0 00:15:02        1

! (2) Path Selection: Best Path ต้องเป็น SP-PE1 (Local-Pref 200 ชนะ)
WAN-EDGE-1# show ip bgp 172.30.10.0
BGP routing table entry for 172.30.10.0/24
  Advertised to update-groups: 1
  65000 65002
    192.0.2.1 from 192.0.2.1 (100.64.0.11)
      Origin IGP, localpref 200, valid, external, best     <-- Best Path ผ่าน SP-PE1
  65000 65002
    192.0.2.9 from 192.0.2.9 (100.64.0.13)
      Origin IGP, localpref 100, valid, external           <-- Backup ผ่าน SP-PE3 (พร้อมใช้ทันที)

! (3) Sham-Link (Lab B): ต้อง Up และ Intra-Area Cost ถูกต้อง
SP-PE1# show ip ospf sham-links | include is up
Sham Link OSPF_SL0 to address 100.64.100.12 is up

! (4) IP SLA (Lab C): ต้องผ่านเกณฑ์ SLA ที่ SP สัญญาไว้
WAN-EDGE-1# show track 10
Track 10
  IP SLA 10 reachability
  Reachability is Up
    2 changes, last change 00:45:12
WAN-EDGE-1# show ip sla statistics 10 | include Latest RTT
   Latest RTT: 12 milliseconds

! (5) End-to-End Reachability สุดท้าย — ยังต้องสำเร็จเหมือน Part 86 ไม่ว่าจะผ่าน Path ไหน
WAN-EDGE-1# ping 172.30.10.1 source Loopback0 repeat 20
Success rate is 100 percent (20/20), round-trip min/avg/max = 5/8/14 ms
```

Lab นี้พิสูจน์ครบทุกแนวคิดหลักของ Part 87: **Multi-Homing** ที่ Failover ได้จริงโดยไม่ Suboptimal
Route (Checklist 1-2), **OSPF Sham-Link** ที่แก้ปัญหา Backdoor Link ได้จริง (Checklist 3), และ
**IP SLA** ที่ให้ลูกค้าเห็นผล SLA ได้เองโดยไม่ต้องพึ่ง SP (Checklist 4) — ทั้งหมดยืนอยู่บนพื้นฐาน
Any-to-Any Connectivity ที่ Part 86 สร้างไว้ (Checklist 5)

---

## แบบฝึกหัดทวนความเข้าใจ Part 87

1. เพราะเหตุใด Sham-Link ต้องใช้ Loopback Address **ภายใน VRF** (ไม่ใช่ Loopback0 Global ของ PE)
   เป็น Endpoint? และทำไมต้อง Advertise ผ่าน BGP VPNv4 (`network ... mask ...`) แทนที่จะพึ่ง OSPF?
2. ในสถานการณ์ Multi-Homing (Step 863) ถ้า SP-PE3 ใช้ RD **เดียวกัน** กับ SP-PE1 (`65000:100`)
   จะเกิดผลอย่างไรกับ Path สำรอง และทำไมการใช้ RD ต่างกันแต่ RT เหมือนกันจึงแก้ปัญหานี้ได้?
3. อธิบายว่าทำไม MVPN Rosen GRE (Step 868) ต้องเปิด PIM บน P Router (SP-P1/SP-P2) ด้วย ในขณะที่
   Unicast VPNv4 Traffic ไม่ต้องให้ P Router รู้จัก BGP/VPN เลยแม้แต่นิดเดียว?
4. ตามกรอบการตัดสินใจ Step 867 องค์กรที่เป็น Trading Platform (ต้องการ Latency ต่ำที่คาดเดาได้
   แน่นอน) ควรเลือก MPLS L3VPN, SD-WAN-over-Internet, หรือ Hybrid WAN? อธิบายเหตุผล
5. Inter-AS Option A, B, และ C ต่างกันที่จุดใดในเรื่อง "ใครเก็บ VPN Route ของลูกค้าบ้าง" และ Option
   ไหนที่ ASBR แบกภาระ (Load) มากที่สุด?

**เฉลย:**

1. Sham-Link ทำงานเป็น Virtual OSPF Adjacency ที่ "ซ้อน" อยู่บนโครงสร้าง MP-BGP VPNv4 ที่มีอยู่แล้ว
   — ถ้าใช้ Loopback0 Global (ที่ Reachable ผ่าน IGP ของ SP Core/LDP) จะทำให้ OSPF ของ VRF ไปพึ่งพา
   Infrastructure ของ Global Table ซึ่งไม่ถูกต้องตามสถาปัตยกรรม VRF Isolation (Step 853) และจะเกิด
   Chicken-and-Egg Problem ถ้า Reachability ของ Sham-Link เอง ต้องพึ่ง OSPF ของ VRF ที่ Sham-Link
   กำลังจะสร้างขึ้น จึงต้อง Advertise ผ่าน BGP VPNv4 (Independent Path) เพื่อให้ Sham-Link มี
   Reachability ที่มั่นคงไม่ขึ้นกับ OSPF ของตัวเอง
2. RD เดียวกันทำให้ VPNv4 NLRI ของ SP-PE1 และ SP-PE3 กลายเป็น Prefix เดียวกันเป๊ะ (`RD:Prefix`
   ซ้ำกัน) — BGP จะเก็บแค่ **Best Path เดียว** ทิ้ง Path ที่เหลือจาก BGP Table ของ PE อื่นทันที ทำให้
   ไม่มี Path สำรองพร้อมใช้เลยแม้ Link จะพร้อมอยู่จริง (Failover ต้องรอ BGP Convergence ใหม่ทั้งหมด)
   — RD ต่างกันทำให้ทั้งสอง Path ถูกมองเป็น NLRI คนละตัว จึงถูกเก็บไว้ทั้งคู่ ส่วน RT เหมือนกันทำให้
   ทั้งสอง Path ยังถูก Import เข้า VRF เดียวกัน (VPN เดิม) ไม่หลุดออกจากกัน
3. Unicast VPNv4 Traffic ใช้ **Label Switching** ที่ P Router แค่ Swap Label ตาม LFIB ที่ LDP สร้างให้
   (Data Plane ล้วนๆ ไม่ต้องอ่าน VPN Information) แต่ Default MDT ของ MVPN Rosen ต้องสร้าง (S,G)
   Multicast Tree ผ่าน **PIM Join/Prune ที่เป็น Control-Plane Signaling แบบ Hop-by-Hop** ซึ่งไม่ใช่
   กลไก Label Switching เลย — ทุก Router บนเส้นทาง (รวม P Router) ต้องเข้าใจ PIM และมี Multicast
   Routing Table (mroute) ของตัวเองเพื่อส่ง Join/Prune ต่อไปยัง Hop ถัดไปได้ถูกต้อง นี่คือข้อจำกัดที่
   ทำให้ NG-MVPN/BGP-based (Part 83 Step 823) ถูกออกแบบมาแทนที่ในภายหลัง
4. ควรเลือก **Hybrid WAN** เป็นคำตอบที่สมดุลที่สุด — ใช้ MPLS L3VPN (พร้อม MPLS TE ถ้า SP รองรับ)
   สำหรับ Traffic ของ Trading Platform ที่ Sensitive สูงเพื่อรับประกัน Latency ที่คาดเดาได้แน่นอน
   ตามสัญญา แต่ยังใช้ SD-WAN-over-Internet เป็น Transport คู่ขนานสำหรับ Traffic อื่นที่ไม่ Critical
   เพื่อประหยัดต้นทุนและเพิ่มความคล่องตัว — MPLS อย่างเดียวแพงเกินไปสำหรับ Traffic ทั้งหมด ในขณะที่
   SD-WAN-over-Internet อย่างเดียวไม่มีการันตี Latency ที่เพียงพอสำหรับ Trading Platform
5. Option A (Back-to-Back VRF) ทำให้ **ASBR ของทั้งสองฝั่งไม่เก็บ VPN Route ของอีก AS เลย** (มองว่า
   เป็นแค่ CE ปกติของกันและกัน) Option B ทำให้ **ASBR ต้องเก็บ VPNv4 Route ของทุก VPN ที่ข้าม AS**
   (Load สูงสุดในสาม Option เพราะ ASBR ต้องแบก Full VPN Table ของ Customer ทุกรายที่ข้าม Border)
   Option C ทำให้ **ASBR เก็บแค่ Labeled IPv4 Route ของ Loopback PE เท่านั้น** ปล่อยให้ Route
   Reflector ของแต่ละ AS เป็นคนเก็บ VPN Route จริงและคุยกันเองผ่าน Multihop MP-eBGP — ดังนั้น
   **Option B คือ Option ที่ ASBR แบกภาระมากที่สุด**

---

## สรุป Part 87

Part นี้ยกระดับ MPLS L3VPN ที่ Part 86 สร้างไว้ขึ้นสู่ประเด็นระดับ **Production/CCIE SP** ครบทุกมิติ
ที่ Enterprise WAN Architect ต้องเจอจริง: **PE-CE Protocol ทางเลือก** (OSPF + Sham-Link/DN-bit,
EIGRP + SoO) นอกเหนือจาก eBGP, **Multi-Homing** ที่แก้ปัญหา Suboptimal/Asymmetric Routing ด้วย
RD/RT Design และ BGP Attribute อย่างถูกต้อง (เพิ่ม **SP-PE3** เข้าสู่ SP Core), **Internet Access**
ผ่าน MPLS VPN สองแนวทางพร้อมคำแนะนำ, ภาพรวม **MPLS TE** และ **CsC/Inter-AS VPN** สำหรับสถานการณ์
ข้าม SP หลายราย, กรอบการตัดสินใจ **MPLS vs SD-WAN** ที่เชื่อมกับ [Part 39](part-039-sd-wan-fundamentals.md)/
[Part 76](part-076-sdwan-advanced-design.md), การนำ **MVPN Rosen GRE** ([Part 83](part-083-enterprise-multicast-deep-dive.md))
มาวิ่งข้าม MPLS Core ตัวจริงเป็นครั้งแรก, และปิดท้ายด้วย **IP SLA** ([Part 32](part-032-policy-based-routing.md))
เป็นเครื่องมือ Verify SLA จากมุมมองลูกค้าที่มองไม่เห็นข้างในของ SP เลย — Lab เต็มรูปแบบใน Step 870
ผสานทุกแนวคิดเข้าด้วยกันและพิสูจน์ว่าใช้งานได้จริงทุกจุด

MPLS L3VPN (Part 86-87) ถือเป็น **WAN Underlay Technology ตัวสุดท้ายที่หลักสูตรนี้ลง Config เต็ม
รูปแบบทั้งฝั่ง Provider และ Enterprise** — ต่อจากนี้หลักสูตรจะเปลี่ยนมุมกลับสู่ฝั่ง Enterprise
Network เต็มตัวอีกครั้งเพื่อปิดหัวข้อ Fundamental ที่เหลือให้ครบก่อนเข้าสู่ Capstone Lab สุดท้ายของ
ระดับ CCIE

✅ **พร้อมสำหรับ Part 88**: หลักสูตรจะเปลี่ยนโฟกัสไปที่ **IPv6 Enterprise Deployment** — การนำ IPv6
มาใช้จริงทั่วทั้ง Enterprise Network ที่สร้างมาตลอด 87 Part (Campus, Data Center, WAN, และ MPLS
L3VPN ที่เพิ่งสร้างนี้) ครอบคลุม Dual-Stack Design, IPv6 Routing (OSPFv3/BGP), และ Transition
Mechanism ที่จำเป็นสำหรับ Enterprise ที่ยังต้องอยู่กับ IPv4 คู่ขนานไปอีกนาน

**ไปต่อ:** [Part 88 — IPv6 Enterprise Deployment →](part-088-ipv6-enterprise-deployment.md)
