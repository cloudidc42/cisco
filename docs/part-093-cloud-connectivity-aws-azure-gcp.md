# Part 93 — Cloud Connectivity (AWS/Azure/GCP, Cloud OnRamp)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 921–930 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 92 — Disaster Recovery & Network Resilience →](part-092-disaster-recovery-network-resilience.md)
> ที่ออกแบบหลักการ **RTO/RPO, Active-Active/Active-Passive Data Center, และ Network-Level DR**
> ให้กับ Enterprise Network ของหลักสูตรนี้ — Part นี้ขยายหลักการเดียวกันไปสู่ทิศทางที่ Workload
> ขององค์กรสมัยใหม่ไหลไปอยู่มากขึ้นทุกปี: **Public Cloud (AWS, Azure, GCP)** เราจะตอบคำถามที่
> ค้างไว้จากสอง Part ก่อนหน้า: [Part 74 Step 735](part-074-advanced-bgp-wan-edge.md#step-735--bgp-สำหรับ-multi-cloud-connectivity-preview)
> Preview เรื่อง eBGP บน Direct Connect/ExpressRoute-style Circuit ไว้ และ
> [Part 76 Step 755-756](part-076-sdwan-advanced-design.md#step-755--cloud-onramp-for-saas-เชิงลึก-เลือก-local-breakout-ที่ดีที่สุดต่อแอปโดยอัตโนมัติ)
> Preview เรื่อง Cloud OnRamp for SaaS/IaaS ไว้โดยบอกตรงๆ ว่ารายละเอียดเต็มรูปแบบจะสอนที่ Part นี้
>
> Part นี้คือจุดที่เราเชื่อม **WAN-EDGE-1/WAN-EDGE-2** ของหลักสูตร เข้ากับ **AWS VPC, Azure VNet,
> และ GCP VPC** ด้วยวิธีที่ Enterprise จริงใช้งาน — ทั้งแบบ **Dedicated Private Connectivity**
> (AWS Direct Connect, Azure ExpressRoute, GCP Cloud Interconnect) และแบบ **SD-WAN Automated
> Cloud OnRamp** — พร้อมออกแบบ **Multi-Cloud Resilience** ตามหลักการ DR ที่ Part 92 วางไว้

> **หมายเหตุเรื่อง Lab**: Direct Connect/ExpressRoute/Cloud Interconnect ต้องมี **Physical Cross
> Connect จริงที่ Colocation Facility** (Equinix, Digital Realty ฯลฯ) ซึ่งอยู่นอกเหนือขอบเขตของ
> Lab CML/EVE-NG/Packet Tracer — ทุก Config ในหลักสูตรนี้เป็น **Syntax จริงของ Cisco IOS-XE**
> (ฝั่ง WAN-EDGE) ที่นำไปใช้กับ Direct Connect/ExpressRoute จริงได้ทันที ส่วนฝั่ง Cloud
> (Console/CLI ของ AWS/Azure/GCP) แสดงเป็น **Console-style Reference** เพื่อให้เห็นภาพ End-to-End
> ตาม Concept เดียวกับที่ Part 76 ใช้กับ SD-WAN Policy ระดับ Enterprise Scale

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 921 | ทำไม Enterprise ต้องมี Dedicated Cloud Connectivity — Internet-based VPN vs Dedicated Connection |
| 922 | AWS Direct Connect — Direct Connect Gateway, Virtual Interface (Public/Private/Transit), Worked Config eBGP บน WAN-EDGE-1 |
| 923 | Azure ExpressRoute — Circuit, Peering Type (Private/Microsoft), เทียบกับ AWS Direct Connect |
| 924 | GCP Cloud Interconnect/Partner Interconnect — ตาราง 3-Way Comparison AWS/Azure/GCP |
| 925 | Cloud Router/Transit Gateway — AWS Transit Gateway และ Azure Virtual WAN คือ "Hub" ฝั่ง Cloud |
| 926 | Routing Design สำหรับ Hybrid Cloud — BGP Route Propagation จาก Campus ถึง VPC/VNet |
| 927 | Security สำหรับ Cloud Connectivity — ขยาย FTD/Umbrella Stack (Part 78) เข้าสู่ Cloud Workload |
| 928 | Cloud OnRamp เจาะลึก (ต่อจาก Part 76 Step 755-756) — เมื่อไหร่พอ เมื่อไหร่ต้อง Direct Connect/ExpressRoute จริง |
| 929 | Multi-Cloud Resilience Design — Dual Connectivity ข้าม Region/Provider ตามหลัก DR ของ Part 92 |
| 930 | Lab เต็มรูปแบบ: Hybrid-Cloud Connectivity Design สำหรับ Enterprise Lab พร้อม Verification |

---

## Step 921 — ทำไม Enterprise ต้องมี Dedicated Cloud Connectivity

### 921.1 จุดเริ่มต้น: ทุก Enterprise เชื่อมต่อ Cloud ผ่าน Internet ได้อยู่แล้ว แล้วทำไมยังไม่พอ

ตั้งแต่ [Part 15](part-015-nat-pat.md) (NAT/PAT) หลักสูตรนี้สอนไปแล้วว่า Enterprise ออก Internet
ได้ผ่าน WAN-EDGE-1/2 อยู่แล้ว ซึ่งในทางเทคนิค **เพียงพอ**ที่จะสร้าง Site-to-Site VPN (IPsec, ตาม
[Part 51](part-051-vrf-lite-gre-ipsec.md)) ไปยัง AWS Virtual Private Gateway หรือ Azure VPN
Gateway ได้ทันทีโดยไม่ต้องรอ Part นี้เลย — คำถามคือ **แล้วทำไมองค์กรขนาดใหญ่ยังลงทุนหลักแสนถึง
หลักล้านบาทต่อปีกับ Direct Connect/ExpressRoute ทั้งที่ Internet VPN ก็ "ใช้งานได้" เหมือนกัน**

คำตอบอยู่ที่ **ลักษณะของ Internet Path เอง** ไม่ใช่ที่ Protocol IPsec:

| ประเด็น | Internet-based VPN (IPsec ผ่าน Internet) | Dedicated Connection (Direct Connect/ExpressRoute/Interconnect) |
|---|---|---|
| เส้นทางกายภาพ | ไม่แน่นอน — ผ่าน ISP หลายเจ้า, Peering Point ที่ไม่รู้จัก, เปลี่ยนแปลงได้ตาม BGP ของ Internet | คงที่ — Physical Cross Connect ที่ Colocation Facility เดียวกันกับ Cloud Provider (จองไว้ล่วงหน้า) |
| Bandwidth ที่รับประกัน | ไม่มี SLA เรื่อง Bandwidth จริง (Best-Effort ตาม ISP) | รับประกัน Bandwidth ตาม Port Speed ที่จอง (เช่น 1G/10G/100G Dedicated) |
| Latency | ผันแปรตามเส้นทาง Internet ขณะนั้น (Congestion ของ ISP กลางทาง) | ต่ำและสม่ำเสมอ (Physical Path เดียว ไม่ผ่าน Peering หลายจุด) |
| Jitter/Packet Loss | สูงกว่าและไม่แน่นอน — กระทบ Application ที่ Sensitive (Database Replication, Real-time Analytics) | ต่ำกว่ามาก — เหมาะกับ Traffic ที่ต้องการ Consistency |
| ความปลอดภัย | ข้อมูลวิ่งผ่าน Public Internet เสมอ (ต้อง Encrypt ด้วย IPsec ทุกครั้ง) | ไม่ผ่าน Public Internet เลย (แต่ยัง **ควร** เข้ารหัสอยู่ดีสำหรับข้อมูล Sensitive ตาม Defense-in-Depth) |
| Compliance | บาง Regulation (PCI-DSS, ข้อบังคับการเงินบางประเทศ) มองว่า Internet-based Path เป็นความเสี่ยงเพิ่ม | ตอบโจทย์ Regulation ที่ต้องการ "Private Path" ระหว่าง Data Center กับ Cloud โดยตรง |
| ต้นทุน | ต่ำ (ใช้ Internet Circuit ที่มีอยู่แล้ว) | สูงกว่า (ค่า Port + ค่า Cross Connect + ค่า Data Transfer ที่มักถูกกว่า Internet ต่อ GB) |
| เวลาติดตั้ง | เร็ว (เปิด IPsec Tunnel ได้ในไม่กี่นาที) | ช้า (สั่ง Circuit ใหม่ใช้เวลาหลายสัปดาห์ถึงหลายเดือน ถ้าไม่มี Port ว่างอยู่แล้ว) |

### 921.2 กรณีที่ Dedicated Connectivity คุ้มค่าการลงทุนจริง

```
                    เส้นทางที่ 1: Internet-based Site-to-Site VPN (เทคนิคเดิมจาก Part 51)

  WAN-EDGE-1 ──biz-internet (ISP-RTR)──► [ ISP หลายเจ้า, Peering Point ไม่รู้จัก ] ──► AWS VGW
        │                                          │
        │                          Latency/Jitter ผันแปรตาม Internet ขณะนั้น
        │                          Bandwidth ไม่มี SLA รับประกัน
        ▼
  เหมาะกับ: Workload ที่ไม่ Sensitive ต่อ Latency, Traffic ปริมาณไม่สูง, Budget จำกัด


                    เส้นทางที่ 2: Dedicated Connection (Direct Connect/ExpressRoute/Interconnect)

  WAN-EDGE-1 ──Cross Connect ที่ Colocation──► [ Cloud Provider Edge Router ] ──► VPC/VNet
        │              (Physical, จองไว้ล่วงหน้า, Bandwidth การันตี)
        ▼
  เหมาะกับ: Database Replication, ERP แบบ Real-time, Migration ข้อมูลปริมาณสูง,
            องค์กรที่ต้อง Compliance ที่ห้าม Traffic วิ่งผ่าน Public Internet
```

### 921.3 หลักการที่ยังใช้ต่อจาก Part ก่อนหน้าเสมอ

**Dedicated Connectivity ไม่ได้แทนที่ IPsec VPN** — ทั้งสองเทคนิคยังใช้งานคู่กันในองค์กรจริงเสมอ:

| การใช้งาน | เทคนิคที่เหมาะสม |
|---|---|
| Branch เล็กที่ไม่มี Traffic ไป Cloud มาก | Internet VPN (Part 51) หรือ SD-WAN Cloud OnRamp (Step 928) |
| Data Center หลักที่มี Traffic ไป Cloud ปริมาณสูงต่อเนื่อง | Direct Connect/ExpressRoute (Step 922-924) |
| Backup Path เผื่อ Dedicated Connection ล่ม | Internet VPN เป็น Failover เสมอ (ไม่ใช่ Single Path ที่พึ่งได้ทางเดียว — ตามหลักการ Redundancy ที่ [Part 92](part-092-disaster-recovery-network-resilience.md) วางไว้) |

> **จุดสำคัญสำหรับ CCIE**: ข้อสอบมักถามว่า "ทำไมองค์กร X ยังต้องมี IPsec VPN สำรองทั้งที่มี Direct
> Connect แล้ว" — คำตอบคือ **Direct Connect เป็น Physical Circuit เส้นเดียว** (แม้จะมี Redundant
> VIF ตาม Step 922 ก็ยังอาจล่มทั้งคู่ถ้า Colocation Facility มีปัญหา Power/Fiber Cut) การมี
> Internet VPN เป็น Backup คือการทำ **Diverse Path** ตามหลักการ DR พื้นฐานที่สุด

---

## Step 922 — AWS Direct Connect: Direct Connect Gateway, Virtual Interface, Worked Config

### 922.1 สถาปัตยกรรมของ AWS Direct Connect

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS Direct Connect Location (Colocation)              │
│                                                                                │
│   WAN-EDGE-1 ══Cross Connect (Fiber)══► AWS Direct Connect Router             │
│   (Customer/Partner Router)                    │                              │
│                                                  │  Virtual Interface (VIF)     │
│                                    ┌─────────────┴─────────────┐              │
│                              Public VIF                  Private VIF          │
│                          (เข้าถึง AWS Public               (เข้าถึง VPC              │
│                           Service เช่น S3               ผ่าน Private IP โดยตรง)     │
│                           โดยไม่ผ่าน Internet)                  │              │
│                                                          Transit VIF          │
│                                                     (เข้าถึงหลาย VPC ผ่าน           │
│                                                      Direct Connect Gateway         │
│                                                      + Transit Gateway)             │
└─────────────────────────────────────────────────────────────────────────────┘
                                                          │
                                            ┌─────────────┴─────────────┐
                                            │   Direct Connect Gateway     │  <- Global Resource
                                            │   (dxgw-11223344)             │     เชื่อมหลาย VIF
                                            └─────────────┬─────────────┘     เข้ากับหลาย VPC/Region
                                            ┌─────────────┴─────────────┐
                                            │      Virtual Private Gateway   │  <- ต่อ VPC เดียว
                                            │      หรือ Transit Gateway       │     หรือหลาย VPC (Step 925)
                                            └─────────────┬─────────────┘
                                                    ┌──────┴──────┐
                                                    │   VPC (Prod)   │  10.50.0.0/16
                                                    └─────────────┘
```

### 922.2 ความหมายของ Virtual Interface (VIF) ทั้ง 3 ประเภท

| VIF Type | ใช้เข้าถึงอะไร | BGP Session แยกจากกันหรือไม่ | ตัวอย่างการใช้งาน |
|---|---|---|---|
| **Public VIF** | AWS Public Service (S3, DynamoDB) ผ่าน Public IP แต่ไม่ผ่าน Internet จริง | แยก Session, ต้องประกาศ Public Prefix ของตัวเอง | Backup ข้อมูลขึ้น S3 โดยไม่กิน Internet Bandwidth |
| **Private VIF** | VPC เดียว ผ่าน Private IP โดยตรง (ผ่าน Virtual Private Gateway) | แยก Session ต่อ VPC | Workload ปกติที่อยู่ใน VPC เดียว |
| **Transit VIF** | หลาย VPC/หลาย Region ผ่าน Direct Connect Gateway + Transit Gateway | แยก Session แต่ Advertise ครอบคลุมทุก VPC ที่ Attach กับ Transit Gateway | Enterprise ที่มีหลาย VPC ตาม Business Unit (Step 925) |

> **เทียบกับหลักสูตรนี้**: VIF แต่ละแบบทำงานเหมือน **Sub-interface + VLAN** ที่ Part
> [Part 10](part-010-inter-vlan-routing.md) สอน (Router on a Stick) — Physical Cross Connect
> เส้นเดียวถูกแบ่งด้วย 802.1Q VLAN Tag ออกเป็นหลาย Logical Interface แต่ละ Interface มี BGP
> Session ของตัวเอง คนละความหมายของ Traffic ที่วิ่งผ่าน

### 922.3 Worked Config เต็มรูปแบบ: WAN-EDGE-1 ตั้ง Private VIF ไปยัง AWS VPC

**สถานการณ์**: WAN-EDGE-1 มี Cross Connect ไปยัง AWS Direct Connect Location แล้ว (งานที่ทำผ่าน
AWS Console/Partner ไม่ใช่ CLI ของ WAN-EDGE-1) — AWS สร้าง Private VIF ชื่อ `vif-0a1b2c3d4e5f67890`
ให้แล้ว โดยกำหนด VLAN 100, BGP Peering Subnet `169.254.100.0/30` (Amazon-assigned ตามค่า
Default), Amazon Router IP `169.254.100.2`, Amazon Side ASN `64512` (Virtual Private Gateway)

```
! ---------- ขั้นที่ 1: สร้าง Sub-interface สำหรับ Private VIF (VLAN 100) ----------
WAN-EDGE-1(config)# interface TenGigabitEthernet0/0/3
WAN-EDGE-1(config-if)# description ** Cross-Connect ไปยัง AWS Direct Connect Location (Equinix) **
WAN-EDGE-1(config-if)# no ip address
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# interface TenGigabitEthernet0/0/3.100
WAN-EDGE-1(config-subif)# description ** AWS Direct Connect Private VIF - vif-0a1b2c3d4e5f67890 (Prod-VPC) **
WAN-EDGE-1(config-subif)# encapsulation dot1Q 100
WAN-EDGE-1(config-subif)# ip address 169.254.100.1 255.255.255.252
WAN-EDGE-1(config-subif)# exit
!
! ---------- ขั้นที่ 2: Prefix-list ควบคุมว่าจะประกาศ/รับ Prefix อะไร (หลักการเดิมจาก Part 29 Step 286) ----------
WAN-EDGE-1(config)# ip prefix-list PL-TO-AWS-VPC seq 10 permit 10.10.0.0/16
! ประกาศแค่ Summary ของ On-Prem (VLAN 10/20/30/40/99 รวมกัน) ไม่ปล่อยทุก /24 ย่อยออกไปให้ AWS เห็น
!
WAN-EDGE-1(config)# ip prefix-list PL-FROM-AWS-VPC seq 10 permit 10.50.0.0/16 le 24
! รับเฉพาะ Prefix ของ Prod-VPC (10.50.0.0/16) และ Subnet ย่อยไม่เกิน /24 เท่านั้น
! ป้องกัน AWS ฝั่งโน้น Advertise Prefix ผิดพลาด (เช่น Default Route 0.0.0.0/0) เข้ามาโดยไม่ตั้งใจ
!
! ---------- ขั้นที่ 3: BGP Neighbor สำหรับ Private VIF ----------
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 169.254.100.2 remote-as 64512
WAN-EDGE-1(config-router)# neighbor 169.254.100.2 description ** eBGP to AWS VGW via Private VIF (Prod-VPC) **
WAN-EDGE-1(config-router)# neighbor 169.254.100.2 timers 10 30
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 169.254.100.2 activate
WAN-EDGE-1(config-router-af)# neighbor 169.254.100.2 prefix-list PL-TO-AWS-VPC out
WAN-EDGE-1(config-router-af)# neighbor 169.254.100.2 prefix-list PL-FROM-AWS-VPC in
WAN-EDGE-1(config-router-af)# neighbor 169.254.100.2 soft-reconfiguration inbound
WAN-EDGE-1(config-router-af)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-1(config-router-af)# exit-address-family
!
! ---------- ขั้นที่ 4: Aggregate เพื่อให้ network statement ข้างบนทำงาน (ต้องมี Route ที่ครอบคลุมจริง) ----------
WAN-EDGE-1(config)# ip route 10.10.0.0 255.255.0.0 Null0 250
! Static Route แบบ Discard (AD 250 = ต่ำสุด ไม่แทรกแซง Route จริงถ้ามี Path ที่เฉพาะเจาะจงกว่า)
! ใช้เป็น "หลักฐาน" ให้ BGP เชื่อว่า Summary Prefix นี้มีอยู่จริงในตาราง Routing เพื่อประกาศได้
```

- **`encapsulation dot1Q 100`** — Cisco IOS-XE ใช้ 802.1Q Tagging เดียวกันกับที่สอนใน
  [Part 10](part-010-inter-vlan-routing.md) เพื่อแยก VIF แต่ละตัวบน Physical Circuit เส้นเดียว
- **`169.254.100.1/30`** — AWS ใช้ Link-Local Range (RFC 3927) เป็นค่า Default สำหรับ BGP Peering
  IP ของ Private VIF (ลูกค้าสามารถระบุ Subnet ของตัวเองแทนได้ถ้าต้องการ)
- **`ip route ... Null0 250`** — เทคนิคเดียวกับที่ [Part 29 Step 285](part-029-bgp-fundamentals.md)
  ใช้กับ `network` statement — ต้องมี Route ที่ตรง (Exact Match) กับ Mask ที่ระบุอยู่ใน RIB ก่อน
  BGP จึงจะยอมประกาศ Prefix นั้นออกไป

### 922.4 Verification

```
WAN-EDGE-1# show ip bgp neighbors 169.254.100.2

BGP neighbor is 169.254.100.2,  remote AS 64512, external link
  BGP version 4, remote router ID 10.50.0.1
  BGP state = Established, up for 00:14:22
  Neighbor capabilities:
    Route refresh: advertised and received
    Address family IPv4 Unicast: advertised and received
  Address family IPv4 Unicast: passive
    Prefixes Current: 1 (Consumes 44 bytes), 1 (Consumes 24 bytes)

WAN-EDGE-1# show ip route bgp | include 10.50
B        10.50.0.0/16 [20/0] via 169.254.100.2, 00:14:22

WAN-EDGE-1# show interfaces TenGigabitEthernet0/0/3.100
TenGigabitEthernet0/0/3.100 is up, line protocol is up
  Hardware is Ten Gigabit Ethernet, address is 0050.56b1.0203
  Description: ** AWS Direct Connect Private VIF - vif-0a1b2c3d4e5f67890 (Prod-VPC) **
  Internet address is 169.254.100.1/30
  Encapsulation 802.1Q Virtual LAN, Vlan ID 100.
```

`BGP state = Established` ยืนยันว่า eBGP Session ผ่าน Private VIF ทำงานจริง และ
`show ip route bgp` แสดงว่า WAN-EDGE-1 เรียนรู้ `10.50.0.0/16` (Prod-VPC CIDR) มาจาก AWS แล้ว —
กลไกทั้งหมดนี้คือ eBGP แบบเดียวกับที่ [Part 29](part-029-bgp-fundamentals.md) สอนไปกับ ISP-RTR
เพียงแค่เปลี่ยนปลายทางจาก "ISP" เป็น "Cloud Provider Edge Router" ตามที่ Part 74 Step 735
Preview ไว้

---

## Step 923 — Azure ExpressRoute: Circuit, Peering Type, เทียบกับ AWS Direct Connect

### 923.1 สถาปัตยกรรมของ Azure ExpressRoute

Azure ใช้คำศัพท์ต่างจาก AWS แต่แนวคิดเดียวกัน: **ExpressRoute Circuit** คือหน่วยการเชื่อมต่อหลัก
(เทียบเท่า "1 Physical Cross Connect" ของ AWS Direct Connect) ภายใน 1 Circuit แบ่งเป็น
**Peering** ได้หลายแบบ:

| Peering Type | ใช้เข้าถึงอะไร | เทียบเท่า AWS VIF |
|---|---|---|
| **Private Peering** | Azure VNet ผ่าน Private IP โดยตรง | Private VIF |
| **Microsoft Peering** | Microsoft 365, Azure PaaS Public Endpoint (Storage, SQL Database) โดยไม่ผ่าน Internet | Public VIF |

> Azure เคยมี "Public Peering" (เข้าถึง Azure Public IP ทั่วไป) แต่ **ถูก Deprecate ไปแล้ว** โดย
> รวมเข้ากับ Microsoft Peering ปัจจุบัน — ข้อสอบรุ่นใหม่จะเจอแค่ Private Peering และ Microsoft
> Peering เท่านั้น

### 923.2 Redundancy ภายใน 1 Circuit — Primary/Secondary Connection

ทุก ExpressRoute Circuit ที่สั่งใหม่ Azure **บังคับ**ให้มี 2 BGP Session เสมอ (Primary +
Secondary) ผ่าน 2 Subnet /30 แยกกัน — ต่างจาก AWS ที่ Redundancy เกิดจากการสั่ง **2 VIF/2
Connection แยกกันเอง**:

```
                         Azure ExpressRoute Circuit (1 หน่วย)
                   ┌─────────────────────────────────────────┐
                   │  Primary Connection    Secondary Connection │
                   │  (Subnet /30 ที่ 1)      (Subnet /30 ที่ 2)     │
                   │       │                        │              │
                   └───────┼────────────────────────┼──────────────┘
                           │                        │
                    WAN-EDGE-2.300           WAN-EDGE-2.301
                    (Primary BGP)             (Secondary BGP)
```

### 923.3 ตารางเทียบคำศัพท์ AWS Direct Connect ↔ Azure ExpressRoute

| แนวคิด | AWS Direct Connect | Azure ExpressRoute |
|---|---|---|
| หน่วยการเชื่อมต่อหลัก | Connection (1 Physical/Hosted Port) | Circuit |
| การแบ่ง Logical Traffic | Virtual Interface (VIF) | Peering (Private/Microsoft) |
| Redundancy ภายในหน่วยเดียว | ต้องสั่งหลาย VIF/Connection เอง | บังคับมี Primary+Secondary ในตัว |
| เข้าถึง VPC/VNet โดยตรง | Private VIF → VGW | Private Peering → ExpressRoute Gateway (ในVNet) |
| เข้าถึง Public Cloud Service โดยไม่ผ่าน Internet | Public VIF | Microsoft Peering |
| เชื่อมหลาย VPC/VNet พร้อมกัน | Direct Connect Gateway + Transit Gateway | ExpressRoute Gateway ผ่าน Virtual WAN Hub |
| ASN ฝั่ง Cloud (ตัวอย่าง) | 64512 (VGW) | 12076 (Microsoft) |

### 923.4 Worked Config ตัวอย่างสั้น: WAN-EDGE-2 ตั้ง Private Peering ไปยัง Azure VNet

```
! Primary Connection (VLAN 300)
WAN-EDGE-2(config)# interface TenGigabitEthernet0/0/3.300
WAN-EDGE-2(config-subif)# description ** Azure ExpressRoute Private Peering - Primary (Corp-VNet) **
WAN-EDGE-2(config-subif)# encapsulation dot1Q 300
WAN-EDGE-2(config-subif)# ip address 192.168.10.1 255.255.255.252
WAN-EDGE-2(config-subif)# exit
!
! Secondary Connection (VLAN 301)
WAN-EDGE-2(config)# interface TenGigabitEthernet0/0/3.301
WAN-EDGE-2(config-subif)# description ** Azure ExpressRoute Private Peering - Secondary (Corp-VNet) **
WAN-EDGE-2(config-subif)# encapsulation dot1Q 301
WAN-EDGE-2(config-subif)# ip address 192.168.10.5 255.255.255.252
WAN-EDGE-2(config-subif)# exit
!
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# neighbor 192.168.10.2 remote-as 12076
WAN-EDGE-2(config-router)# neighbor 192.168.10.2 description ** Primary - Microsoft Edge Router **
WAN-EDGE-2(config-router)# neighbor 192.168.10.6 remote-as 12076
WAN-EDGE-2(config-router)# neighbor 192.168.10.6 description ** Secondary - Microsoft Edge Router **
WAN-EDGE-2(config-router)# address-family ipv4 unicast
WAN-EDGE-2(config-router-af)# neighbor 192.168.10.2 activate
WAN-EDGE-2(config-router-af)# neighbor 192.168.10.6 activate
WAN-EDGE-2(config-router-af)# neighbor 192.168.10.6 route-map DEPREF-SECONDARY in
WAN-EDGE-2(config-router-af)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-2(config-router-af)# exit-address-family
!
WAN-EDGE-2(config)# route-map DEPREF-SECONDARY permit 10
WAN-EDGE-2(config-route-map)# set local-preference 90
! ทำให้ Path ผ่าน Secondary Connection มี Local-Preference ต่ำกว่า Default (100)
! WAN-EDGE-2 จะเลือกใช้ Primary Connection ก่อนเสมอ เว้นแต่ Primary ล่ม
```

`set local-preference 90` คือเทคนิคเดียวกับที่ [Part 30](part-030-bgp-advanced.md) สอนเรื่อง Best
Path — ใช้บังคับ Path Selection ระหว่าง Primary/Secondary Connection ของ ExpressRoute Circuit
เดียวกัน (Azure แนะนำให้ทำแบบนี้เพื่อไม่ให้ Traffic กระจาย ECMP ข้าม 2 Session โดยไม่ตั้งใจ)

---

## Step 924 — GCP Cloud Interconnect/Partner Interconnect: 3-Way Comparison เต็มรูปแบบ

### 924.1 สถาปัตยกรรมของ GCP โดยย่อ

GCP มี 2 รูปแบบของ Dedicated Connectivity:

| รูปแบบ | ลักษณะ | เหมาะกับ |
|---|---|---|
| **Dedicated Interconnect** | Cross Connect ตรงเข้า Google Edge Location (ต้องมี Physical Presence ที่ Facility ของ Google) | องค์กรที่มี Traffic ปริมาณสูงมาก (10G/100G) และอยู่ใกล้ Google Edge Location |
| **Partner Interconnect** | เชื่อมผ่าน Service Provider ที่เป็น Partner ของ Google (ไม่ต้องมี Physical Presence ที่ Google เอง) | องค์กรทั่วไปที่ต้องการ Bandwidth ระดับ 50Mbps–50Gbps โดยไม่ลงทุน Physical Circuit ตรง |

การเชื่อมต่อทำผ่าน **VLAN Attachment** (เทียบเท่า VIF ของ AWS/Peering ของ Azure) เข้าสู่
**Cloud Router** ซึ่งเป็นจุดที่ GCP รัน eBGP จริง (Google สงวน ASN สาธารณะ **16550** ไว้ใช้เป็น
ASN Default ของ Cloud Router ทุกโปรเจกต์)

### 924.2 ตาราง 3-Way Comparison เต็มรูปแบบ — AWS / Azure / GCP

| แนวคิด | AWS Direct Connect | Azure ExpressRoute | GCP Cloud/Partner Interconnect |
|---|---|---|---|
| ชื่อบริการ Dedicated Connectivity | Direct Connect | ExpressRoute | Cloud Interconnect (Dedicated) / Partner Interconnect |
| หน่วยการเชื่อมต่อหลัก | Connection | Circuit | Interconnect Attachment |
| การแบ่ง Logical Traffic บน Physical เดียวกัน | Virtual Interface (VIF: Public/Private/Transit) | Peering (Private/Microsoft) | VLAN Attachment |
| Cloud-side Router/Gateway ต่อ VPC เดียว | Virtual Private Gateway (VGW) | ExpressRoute Gateway (ในVNet) | Cloud Router |
| เชื่อมหลาย VPC/VNet พร้อมกัน (Step 925) | Direct Connect Gateway + Transit Gateway | Virtual WAN Hub | Network Connectivity Center (NCC) + Cloud Router หลายตัว |
| Routing Protocol | eBGP | eBGP | eBGP |
| ASN ฝั่ง Cloud (ตัวอย่าง/Default) | 64512 (VGW) | 12076 (Microsoft) | 16550 (Cloud Router Default) |
| Redundancy ที่บังคับมาให้ | ไม่บังคับ (ลูกค้าสั่งเพิ่มเอง) | บังคับ Primary+Secondary ในทุก Circuit | ไม่บังคับ (แนะนำ 2 Attachment ต่อ Region) |
| Range IP สำหรับ BGP Peering ตัวอย่าง | 169.254.0.0/16 (Amazon-assigned) | Customer-assigned /30 (เช่น 192.168.x.x) | 169.254.0.0/16 (Google-assigned ได้เช่นกัน) |
| Bandwidth Tier ตัวอย่าง | 50Mbps – 100Gbps | 50Mbps – 10Gbps | 50Mbps – 50Gbps (Partner) / สูงกว่าสำหรับ Dedicated |
| เข้าถึง Public Service โดยไม่ผ่าน Internet | Public VIF | Microsoft Peering | Cloud Interconnect + Private Google Access / Restricted Google APIs |

### 924.3 Worked Config ตัวอย่างสั้น: WAN-EDGE-1 ตั้ง VLAN Attachment ไปยัง GCP VPC

```
WAN-EDGE-1(config)# interface TenGigabitEthernet0/0/4.500
WAN-EDGE-1(config-subif)# description ** GCP Partner Interconnect VLAN Attachment - gcp-prod-vpc **
WAN-EDGE-1(config-subif)# encapsulation dot1Q 500
WAN-EDGE-1(config-subif)# ip address 169.254.10.1 255.255.255.248
WAN-EDGE-1(config-subif)# exit
!
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 169.254.10.2 remote-as 16550
WAN-EDGE-1(config-router)# neighbor 169.254.10.2 description ** eBGP to GCP Cloud Router (gcp-prod-vpc) **
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 169.254.10.2 activate
WAN-EDGE-1(config-router-af)# neighbor 169.254.10.2 prefix-list PL-TO-AWS-VPC out
WAN-EDGE-1(config-router-af)# exit-address-family
```

> **สังเกต**: ใช้ Prefix-list ตัวเดียวกัน (`PL-TO-AWS-VPC`) ซ้ำได้ทันที เพราะ Policy ที่ต้องการคือ
> "ประกาศ Summary On-Prem 10.10.0.0/16 เท่านั้น" ซึ่งเหมือนกันไม่ว่าปลายทางจะเป็น AWS, Azure,
> หรือ GCP — นี่คือประโยชน์ของการออกแบบ Prefix-list ตาม **ความหมายของ Traffic** (Part 29 Step
> 286) แทนที่จะผูกชื่อกับปลายทางเฉพาะเจาะจง

---

## Step 925 — Cloud Router/Transit Gateway: "Hub" ฝั่ง Cloud ที่ Mirror แนวคิด Enterprise Hub

### 925.1 ปัญหาที่ Transit Gateway/Virtual WAN แก้: การเชื่อมหลาย VPC/VNet เข้ากับ On-Prem

องค์กรขนาดใหญ่แทบไม่มี Workload อยู่ใน VPC/VNet เดียว — มักแยกตาม Business Unit, Environment
(Prod/Dev/Test), หรือ Compliance Boundary เป็นสิบๆ VPC ต่อ Account/Region การให้ Direct Connect
Private VIF **แยกกันทุก VPC** (VIF ละ VPC) ทำให้จำนวน VIF/BGP Session โตแบบไม่มีเพดานควบคุมได้
เหมือนปัญหา Tunnel Count ของ SD-WAN Full-Mesh ที่ [Part 76 Step 751](part-076-sdwan-advanced-design.md#step-751--sd-wan-fabric-design-ระดับ-enterprise-scale-hub-and-spoke-vs-full-mesh-vs-regional-mesh)
เจอ

**AWS Transit Gateway** และ **Azure Virtual WAN Hub** คือคำตอบ — ทำหน้าที่เป็น **"Hub" ฝั่ง
Cloud** ที่รวบ VPC/VNet หลายตัวเข้าด้วยกัน แล้วเปิดจุดเชื่อมต่อไปยัง On-Prem **แค่จุดเดียว**
(ผ่าน Direct Connect Gateway หรือ VPN Attachment) แทนที่จะต้องเชื่อมทุก VPC แยกกัน

```
                    ก่อนมี Transit Gateway: เชื่อมทุก VPC แยกกันเอง (N Direct Connect Gateway/VIF)

  WAN-EDGE-1 ══VIF-1══► VPC-A (Prod)          WAN-EDGE-1 ══VIF-2══► VPC-B (Data)
  WAN-EDGE-1 ══VIF-3══► VPC-C (Dev)           WAN-EDGE-1 ══VIF-4══► VPC-D (Shared Services)
  (จำนวน VIF/Session โตตามจำนวน VPC — ไม่ Scale)


                    มี Transit Gateway: On-Prem เห็น "Hub" เดียว ที่ Aggregate ทุก VPC ไว้แล้ว

                              WAN-EDGE-1
                                  │  Transit VIF เดียว (ผ่าน Direct Connect Gateway)
                                  ▼
                    ┌───────────────────────────┐
                    │   AWS TRANSIT GATEWAY (TGW)    │   <- เทียบเท่า "CORE-SW" ของหลักสูตร
                    │      tgw-0a1b2c3d4e5f67890      │      (Part 1 Step 1: 3-Tier Model)
                    └──┬──────┬──────┬──────┬─────┘
                  Attach   Attach  Attach  Attach
                    ┌─┴──┐ ┌─┴──┐ ┌─┴──┐ ┌─┴──┐
                    │VPC-A│ │VPC-B│ │VPC-C│ │VPC-D│
                    │Prod │ │Data │ │Dev  │ │Shared│
                    └────┘ └────┘ └────┘ └────┘
```

### 925.2 การ Mirror แนวคิด Core/Distribution ของ Enterprise Campus

จุดที่น่าสนใจสำหรับ CCIE คือ Transit Gateway/Virtual WAN Hub **ใช้หลักการ Hierarchical เดียวกัน**
กับ 3-Tier Model ที่ [Part 1](part-001-networking-fundamentals.md) สอนไว้ตั้งแต่ Part แรก:

| Enterprise Campus (On-Prem) | Cloud (AWS/Azure) | บทบาทที่เหมือนกัน |
|---|---|---|
| CORE-SW1/CORE-SW2 | Transit Gateway / Virtual WAN Hub | จุดรวม Traffic กลาง, ทำ Route Propagation ระหว่างกลุ่มย่อย |
| DIST-SW1–4 | VPC/VNet แต่ละตัว (Attachment) | กลุ่มย่อยที่มี Policy/Routing Boundary ของตัวเอง |
| WAN-EDGE-1/2 | Direct Connect Gateway | จุดเชื่อมต่อออกไปสู่ภายนอก (On-Prem ↔ Cloud) |
| OSPF/EIGRP ภายใน Core-Distribution | Transit Gateway Route Table (Static + Propagated จาก Attachment) | กลไก Route Exchange ภายในกลุ่ม |

### 925.3 Transit Gateway Route Table — แนวคิด Route Isolation แบบ VRF

Transit Gateway รองรับ **หลาย Route Table** ในตัวเดียว — ทำให้ VPC บางกลุ่มมองเห็นกันได้
(Route Propagate เข้า Route Table เดียวกัน) และบางกลุ่มถูกแยก Isolate จากกันโดยสมบูรณ์ (อยู่คนละ
Route Table) หลักการเดียวกับ **VRF** ที่ [Part 51](part-051-vrf-lite-gre-ipsec.md) สอนไว้
เพียงแค่ย้ายมาทำที่ TGW แทน Physical Router:

```
TGW Route Table: "PROD-RT"          TGW Route Table: "DEV-RT"
  - VPC-A (Prod) attachment           - VPC-C (Dev) attachment
  - Direct Connect attachment         - (ไม่มี Direct Connect — Dev เข้าไม่ถึง On-Prem โดยตรง)
  → VPC-A คุยกับ On-Prem ได้            → VPC-C ถูก Isolate จาก On-Prem โดยสมบูรณ์
```

### 925.4 Azure Virtual WAN — แนวคิดเดียวกัน คำศัพท์ต่างกัน

| แนวคิด | AWS Transit Gateway | Azure Virtual WAN |
|---|---|---|
| ชื่อ Hub กลาง | Transit Gateway | Virtual Hub |
| การเชื่อม VPC/VNet เข้า Hub | Attachment | Hub Virtual Network Connection |
| การเชื่อม On-Prem เข้า Hub | Direct Connect Gateway Attachment / VPN Attachment | ExpressRoute Gateway / VPN Gateway (อยู่ใน Hub เอง) |
| Route Isolation แบบ VRF | หลาย Route Table ต่อ TGW | Routing Intent + Route Table ต่อ Hub |
| รองรับหลาย Region/Hub เชื่อมกันเอง | Transit Gateway Peering (ข้าม Region) | Virtual WAN แบบ Standard (Hub-to-Hub ผ่าน Microsoft Backbone อัตโนมัติ) |

---

## Step 926 — Routing Design สำหรับ Hybrid Cloud: BGP Route Propagation เต็มเส้นทาง

### 926.1 เส้นทางเต็มรูปแบบจาก Campus ถึง Cloud VPC

รวมทุก Layer ของ Routing ที่หลักสูตรนี้สอนมาตั้งแต่ [Part 73](part-073-large-scale-routing-design.md)
(Large-Scale Routing: OSPF ที่ Campus + BGP เป็น Glue) เข้ากับ Direct Connect ของ Step 922:

```
  Server1 (VLAN 30, 10.10.30.11)
        │  OSPF Area (Campus, Part 10/33)
        ▼
  DIST-SW3 ──OSPF──► CORE-SW2
        │  Redistribute OSPF → iBGP (Part 73 Step 73x)
        ▼
  CORE-SW2 (iBGP RR-Client ของ Tier1, Part 74 Step 731)
        │  iBGP (Route-Reflector)
        ▼
  WAN-EDGE-1 (iBGP RR ของ Tier1 + eBGP ไปยัง AWS)
        │  eBGP ผ่าน Private VIF (Step 922)
        ▼
  AWS Virtual Private Gateway (ASN 64512)
        │  Propagate เข้า VPC Route Table
        ▼
  VPC Prod (10.50.0.0/16) → Subnet 10.50.1.0/24 (Application Tier)
```

### 926.2 Route-Table Walkthrough แบบ Hop-by-Hop

| ตำแหน่ง | คำสั่งตรวจสอบ | ผลลัพธ์ที่คาดหวัง (ทิศทาง On-Prem → Cloud) |
|---|---|---|
| DIST-SW3 | `show ip route ospf` | เห็น Route ไป Cloud (`10.50.0.0/16`) ผ่าน OSPF ที่ Redistribute มาจาก CORE-SW2 |
| CORE-SW2 | `show ip bgp` | เห็น `10.50.0.0/16` มาจาก iBGP (Learned ผ่าน RR ที่ WAN-EDGE-1) |
| WAN-EDGE-1 | `show ip bgp` | เห็น `10.50.0.0/16` เป็น eBGP Path (Learned ตรงจาก AWS VGW ผ่าน Private VIF) — **Best Path ตัวจริง** |
| WAN-EDGE-1 | `show ip route bgp` | `B  10.50.0.0/16 [20/0] via 169.254.100.2` |

ทิศทางกลับ (Cloud → On-Prem) ดูจาก **AWS VPC Route Table** (Console-style):

```
VPC Route Table: rtb-0a1b2c3d4e5f67890 (Attach กับ Subnet 10.50.1.0/24)

Destination            Target                          หมายเหตุ
10.50.0.0/16            local                           Traffic ภายใน VPC เอง
10.10.0.0/16            vgw-0a1b2c3d (Virtual Private Gateway)   <- Propagate มาจาก eBGP อัตโนมัติ
                                                          (ต้องเปิด "Route Propagation" ที่ VGW เอง)
0.0.0.0/0               igw-0a1b2c3d4e5f67890 (Internet Gateway) เส้นทาง Internet ปกติของ Instance
```

> **จุดที่ต้องระวังเรื่อง Cloud Console**: AWS **ไม่ได้ใส่ Route ของ On-Prem เข้า VPC Route Table
> ให้อัตโนมัติ**เพียงเพราะ BGP Session Established แล้ว — ต้องไปเปิด **"Route Propagation"**
> ที่หน้า Virtual Private Gateway ผูกกับ Route Table ที่ต้องการอีกขั้นหนึ่งเสมอ (คล้ายกับที่
> OSPF ต้อง `redistribute` อย่างชัดเจนก่อน Route จาก Protocol อื่นจะเข้ามาในตาราง — BGP รู้จัก
> Prefix ไม่ได้แปลว่า VPC Route Table จะเห็น Prefix นั้นทันที)

### 926.3 Full BGP Table ที่ WAN-EDGE-1 (มุมมองรวมทุก Path)

```
WAN-EDGE-1# show ip bgp
BGP table version is 44, local router ID is 1.1.1.21
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network            Next Hop            Metric LocPrf Weight Path
*> 10.10.0.0/16        0.0.0.0                  0         32768 i
*> 10.50.0.0/16        169.254.100.2            0             0 64512 i
*  10.60.0.0/16        192.168.10.2             0             0 12076 i    ! (ExpressRoute ผ่าน WAN-EDGE-2, เรียนผ่าน iBGP)
*> 203.0.113.0/24      1.1.1.254                0             0 65000 i
```

`10.50.0.0/16` มี Origin AS `64512` (AWS) และ `10.60.0.0/16` มี Origin AS `12076` (Azure) — ยืนยัน
ว่า WAN-EDGE-1 มองเห็นทั้งสอง Cloud ผ่าน BGP Table เดียวกัน เหมือนมองเห็น ISP รายอื่นตามที่
[Part 30](part-030-bgp-advanced.md) สอนเรื่อง Multihoming ไว้ทุกประการ

---

## Step 927 — Security สำหรับ Cloud Connectivity: ขยาย FTD/Umbrella Stack เข้าสู่ Cloud

### 927.1 หลักการ: Dedicated Connectivity ไม่ได้แทนที่ Security Stack ที่มีอยู่

Direct Connect/ExpressRoute ทำให้ Traffic ไม่ผ่าน Public Internet แล้ว **ไม่ได้แปลว่าปลอดภัย
โดยอัตโนมัติ** — Threat จากภายในองค์กรเอง (Insider Threat, Compromised Credential, Lateral
Movement) ยังเดินทางผ่าน Private Path นี้ได้เหมือนกัน [Part 78](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
สร้าง Security Stack แบบบูรณาการ (ISE/TrustSec/FTD/Umbrella) ไว้แล้วสำหรับ On-Prem — Step นี้
ขยาย Stack เดียวกันให้ครอบคลุม Cloud Workload

### 927.2 รูปแบบการวาง Inspection Point สำหรับ Cloud Traffic

| รูปแบบ | อธิบาย | เหมาะกับ |
|---|---|---|
| **Centralized Inspection ที่ HQ** | Traffic ไป/มา Cloud ทุก Path ยังบังคับผ่าน FW-1 (FTD) ที่ HQ ก่อนออก WAN-EDGE-1 | องค์กรที่ต้องการ Policy จุดเดียว, ไม่ต้องดูแล Firewall หลายที่ |
| **Distributed Inspection ด้วย FTDv ใน Cloud** | Deploy Cisco Secure Firewall Threat Defense Virtual (FTDv) เข้าไปใน VPC/VNet เอง เป็น Inspection VPC แยก (คล้าย Firewall Insertion ของ Transit Gateway) | องค์กรที่มี Traffic ระหว่าง VPC ปริมาณสูง ไม่อยากให้ Hairpin กลับ On-Prem ทุกครั้ง |
| **Umbrella สำหรับ Cloud Workload** | Instance/VM บน Cloud ส่ง DNS Query ไปตรวจกับ Cisco Umbrella เหมือน Endpoint บน On-Prem | ทุกกรณี — DNS-Layer Security ทำงานได้โดยไม่ขึ้นกับว่า Workload อยู่ที่ไหน |

```
                    Traffic ระหว่าง VPC-A ↔ VPC-B ผ่าน Inspection VPC (Firewall Insertion Pattern)

                              ┌───────────────────┐
                              │   Transit Gateway     │
                              └──┬──────┬──────┬────┘
                            Attach   Attach   Attach
                          ┌─┴──┐   ┌──┴───┐  ┌─┴──┐
                          │VPC-A│   │Inspect│  │VPC-B│
                          │Prod │   │ VPC   │  │Data │
                          └────┘   │(FTDv) │  └────┘
                                   └───────┘
                    TGW Route Table บังคับ Traffic ข้าม VPC-A↔VPC-B
                    ให้เดินทางผ่าน Inspection VPC ก่อนเสมอ (Route Table Trick)
```

### 927.3 Security Group / NSG — ชั้นเสริม ไม่ใช่ตัวแทน FTD

**Security Group (AWS)** และ **Network Security Group — NSG (Azure)** คือ Firewall ระดับ Instance/
Subnet ของ Cloud Provider เอง — ทำงานเป็น **Stateful L3/L4 ACL** ธรรมดา (Allow/Deny ตาม
Source/Dest IP, Port, Protocol) **ไม่มี** Deep Packet Inspection, IPS Signature, หรือ URL
Filtering แบบที่ FTD ทำได้

| คุณสมบัติ | Security Group/NSG (Cloud-native) | FTD (Part 43/78) |
|---|---|---|
| ระดับที่ Enforce | L3/L4 (IP/Port) | L3–L7 (Deep Packet Inspection) |
| IPS/Signature-based Detection | ไม่มี | มี (Cisco Talos) |
| ทำงานที่ | Instance/Subnet (มาพร้อม Cloud Platform) | Inspection VPC/On-Prem (ต้อง Deploy เพิ่ม) |
| ค่าใช้จ่ายเพิ่ม | ไม่มี (รวมในราคา Cloud Platform) | มี (License + Compute) |
| บทบาทที่ถูกต้อง | **ชั้นแรก** — Micro-segmentation พื้นฐานราคาถูกที่สุด | **ชั้นลึก** — วิเคราะห์ Traffic ที่ผ่าน Security Group แล้ว |

> **หลักการ Defense-in-Depth**: ใช้ Security Group/NSG กรอง Traffic ที่ไม่เกี่ยวข้องออกไปก่อน
> (ลด Load ที่ต้องส่งเข้า FTD) แล้วให้ FTD ทำ Deep Inspection กับ Traffic ที่เหลือ — **ไม่ใช่
> เลือกอย่างใดอย่างหนึ่ง** เหมือนที่ [Part 78 Step 774](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
> สอนเรื่อง SGT + Firewall Policy ทำงานร่วมกันเป็นชั้นๆ

### 927.4 กรอบคิด Shared Responsibility Model

| ผู้รับผิดชอบ | AWS/Azure/GCP รับผิดชอบ (Security **of** the Cloud) | องค์กรลูกค้ารับผิดชอบ (Security **in** the Cloud) |
|---|---|---|
| Physical Infrastructure | Data Center, Hardware, Network Backbone ของ Cloud Provider | - |
| Hypervisor/Virtualization Layer | Isolation ระหว่าง Tenant | - |
| Network Configuration | Physical Switch/Router ของ Cloud Provider | **VPC/VNet Design, Security Group/NSG Rule, Route Table** |
| Direct Connect/ExpressRoute Circuit ตัวมันเอง | Physical Circuit, BGP Peering Infrastructure | **BGP Config ฝั่งตัวเอง (WAN-EDGE), Prefix-list Filtering (Step 922)** |
| Data/Application | - | **ข้อมูล, OS Patching, Application Security, IAM, Encryption Key** |

> **จุดสำคัญสำหรับ CCIE/Cloud Networking**: ทุก Incident ที่เกิดจาก "Security Group เปิด 0.0.0.0/0
> โดยไม่ตั้งใจ" หรือ "S3 Bucket เปิด Public" **ไม่ใช่ความผิดของ Cloud Provider** — อยู่ในโซน
> "Security **in** the Cloud" ที่ลูกค้ารับผิดชอบเต็มเสมอ ไม่ว่าจะต่อผ่าน Internet หรือ Direct
> Connect ก็ตาม

---

## Step 928 — Cloud OnRamp เจาะลึก: เมื่อไหร่พอ เมื่อไหร่ต้อง Direct Connect/ExpressRoute จริง

### 928.1 ทบทวน Cloud OnRamp for IaaS จาก Part 76

[Part 76 Step 756](part-076-sdwan-advanced-design.md#step-756--cloud-onramp-for-iaas-ขยาย-fabric-เข้าสู่-awsazuregcp)
สอนไว้ว่า Cloud OnRamp for IaaS ให้ vManage Deploy **cEdge เสมือน (Catalyst 8000V)** เข้าไปใน
VPC/VNet โดยอัตโนมัติ แล้วให้ cEdge เสมือนนั้นเข้าร่วม SD-WAN Fabric เดียวกันกับ Branch/HQ ทุกตัว
ผ่าน **IPsec Tunnel ที่ Auto-built** — ข้อดีคือ **ไม่ต้องรอ Physical Cross Connect ที่ Colocation
Facility เลย** เปิดใช้งานได้ภายในไม่กี่ชั่วโมงถึงหนึ่งวัน เทียบกับ Direct Connect/ExpressRoute
ที่มักใช้เวลาหลายสัปดาห์ถึงหลายเดือน

### 928.2 ตารางตัดสินใจ: Cloud OnRamp พอ vs ต้อง Dedicated Connectivity

| เกณฑ์ | Cloud OnRamp for IaaS เพียงพอ | ต้อง Direct Connect/ExpressRoute/Interconnect |
|---|---|---|
| **Bandwidth ที่ต้องการต่อเนื่อง** | < 1 Gbps โดยประมาณ (ขึ้นกับ Platform ของ cEdge เสมือนที่เลือก) | หลาย Gbps ขึ้นไปอย่างต่อเนื่อง (Data Migration ปริมาณสูง, Video/Media Workload) |
| **SLA เรื่อง Bandwidth** | ไม่มี (วิ่งผ่าน Internet Transport เดิมของ Fabric) | มี (Port Speed ที่จองรับประกันจริง) |
| **Compliance ห้าม Traffic ผ่าน Public Internet** | ไม่ผ่านเกณฑ์ (แม้ Encrypt ด้วย IPsec, Traffic ยังวิ่งบน Internet Transport ทางกายภาพ) | ผ่านเกณฑ์ (Physical Path ไม่แตะ Public Internet เลย) |
| **ความเร็วในการเปิดใช้งาน** | เร็ว (ชั่วโมง–วัน ผ่าน vManage Automation) | ช้า (สัปดาห์–เดือน รอ Physical Cross Connect) |
| **ความซับซ้อนในการดูแล** | ต่ำ (จัดการผ่าน Single Pane of Glass เดียวกับ Fabric ทั้งหมด) | สูงกว่า (ต้องดูแล BGP Config, VIF/Peering แยกจาก SD-WAN Policy) |
| **Application-Aware Routing บน Cloud Link** | มี (Cloud Link เป็น TLOC ปกติ, AAR วัดคุณภาพได้ตาม Part 76 Step 756) | ไม่มีในตัว (ต้องพึ่ง BGP Path Selection แบบเดิมของ Step 923/AS-Path Prepend) |
| **Multi-VPC/VNet Aggregation** | ทำได้ผ่าน SD-WAN Fabric เอง (Site หลาย Site) | ทำได้ผ่าน Transit Gateway/Virtual WAN (Step 925) |

### 928.3 Pattern ที่พบบ่อยในองค์กรจริง: ใช้ทั้งสองแบบร่วมกัน

```
                    HQ/Data Center (Traffic ปริมาณสูงต่อเนื่อง, Compliance สูง)
                         │
                    ══Direct Connect/ExpressRoute (Dedicated, Step 922-923)══► Cloud VPC/VNet
                         │
                    Branch หลายสิบ-ร้อยแห่ง (Traffic ต่อ Cloud ปริมาณน้อย, กระจัดกระจาย)
                         │
                    ══Cloud OnRamp for IaaS (ผ่าน SD-WAN Fabric เดิม)══► Cloud VPC/VNet (เดียวกัน)

  ทั้งสองเส้นทางเข้า VPC/VNet เดียวกันได้ — Direct Connect รับ Traffic หลักจาก HQ,
  Cloud OnRamp รับ Traffic กระจายจาก Branch โดยไม่ต้อง Backhaul ทุก Branch ไป HQ ก่อน
```

> **คำแนะนำสำหรับข้อสอบ Design**: คำถามแบบ "องค์กร Retail 200 สาขาต้องการเชื่อม Cloud" มักมี
> คำตอบที่ถูกคือ **ใช้ทั้งสองแบบผสมกันตาม Traffic Pattern** ไม่ใช่เลือกแบบใดแบบหนึ่งแบบสัมบูรณ์ —
> HQ/DC ที่มี Traffic เยอะใช้ Direct Connect, Branch ที่กระจัดกระจายใช้ Cloud OnRamp ผ่าน Fabric
> ที่มีอยู่แล้ว

---

## Step 929 — Multi-Cloud Resilience Design: Dual Connectivity ตามหลัก DR ของ Part 92

### 929.1 นำหลักการ DR จาก Part 92 มาใช้กับ Cloud Dependency

[Part 92](part-092-disaster-recovery-network-resilience.md) สอนหลักการ RTO/RPO และ Redundant
Path สำหรับ Data Center — Cloud Connectivity ก็ต้องมองด้วยกรอบคิดเดียวกัน: **Direct Connect
เส้นเดียวคือ Single Point of Failure** ไม่ว่าจะมี Redundant VIF ภายในก็ตาม เพราะยังพึ่ง
Colocation Facility เดียวกัน (Power, Fiber Path, แม้กระทั่ง Region เดียวกันของ Cloud Provider)

### 929.2 3 ระดับของ Redundancy สำหรับ Cloud Connectivity

| ระดับ | คำอธิบาย | ป้องกันความล้มเหลวแบบไหน |
|---|---|---|
| **ระดับ 1 — Redundant VIF/Connection** | 2 Private VIF ผ่าน 2 Connection ที่ Colocation Facility เดียวกัน | Port/Router ฝั่ง Cloud Provider ล่มตัวเดียว |
| **ระดับ 2 — Redundant Location** | 2 Direct Connect Connection ที่ **Colocation Facility คนละที่** เชื่อมเข้า VPC เดียวกัน | Facility ทั้งหมดล่ม (Power Outage, Fiber Cut ทั้ง Building) |
| **ระดับ 3 — Multi-Region/Multi-Cloud** | Workload เดียวกัน Deploy ซ้ำใน **คนละ Region** ของ Provider เดียวกัน หรือ **คนละ Provider** เลย (AWS + Azure) พร้อม Connectivity แยกกัน | Region/Provider ทั้งก้อนมีปัญหา (Outage ระดับ Global ของ Cloud Provider เอง) |

```
                    Multi-Region Resilience (Provider เดียวกัน — AWS)

  WAN-EDGE-1 ══DX Connection A (Location 1)══► AWS ap-southeast-1 (Primary VPC)
  WAN-EDGE-2 ══DX Connection B (Location 2)══► AWS ap-southeast-2 (DR VPC, Data ซิงค์ผ่าน Cross-Region Replication)

                    Multi-Cloud Resilience (คนละ Provider — สำหรับ Workload สำคัญที่สุด)

  WAN-EDGE-1 ══Direct Connect══► AWS VPC (Primary Database)
  WAN-EDGE-2 ══ExpressRoute═══► Azure VNet (Standby Read Replica)
  (ถ้า AWS ทั้ง Region มีปัญหา — Application สลับไปอ่าน/เขียนที่ Azure ได้ตาม RTO ที่ Part 92 กำหนด)
```

### 929.3 ตารางออกแบบตาม Critical Tier ของ Workload (เชื่อมกับแนวคิด RTO/RPO ของ Part 92)

| Critical Tier | ตัวอย่าง Workload | ระดับ Redundancy ที่แนะนำ | RTO ที่ยอมรับได้ |
|---|---|---|---|
| **Tier 0 — Mission Critical** | Core Banking Database, Payment Gateway | ระดับ 3 (Multi-Region หรือ Multi-Cloud) | วินาที–นาที (Active-Active) |
| **Tier 1 — Business Critical** | ERP, CRM หลัก | ระดับ 2-3 (Multi-Region อย่างน้อย) | นาที–ชั่วโมง (Active-Passive) |
| **Tier 2 — Important** | Internal Tools, Reporting | ระดับ 1-2 (Redundant Connection พอ) | ชั่วโมง–วัน |
| **Tier 3 — Non-Critical** | Dev/Test Environment | ระดับ 1 (Single Connection + Cloud OnRamp Backup) | วัน (ยอมรับ Downtime ได้) |

> **หลักการสำคัญที่ตรงกับ Part 92**: การลงทุน Redundancy ต้อง **สัดส่วนกับ Critical Tier** ไม่ใช่
> ทำ Multi-Cloud ให้ทุก Workload เพราะต้นทุนและความซับซ้อนของ Multi-Cloud (ต้องดูแล Data
> Consistency ข้าม Provider, Network Policy ที่ต้อง Duplicate) สูงมากพอที่จะไม่คุ้มค่าถ้าใช้กับ
> Workload ที่ไม่ได้ Critical จริง — เกณฑ์การเลือก Tier ต้องมาจาก Business Impact Analysis เดียวกับ
> ที่ Part 92 ใช้กำหนด RTO/RPO ของ Data Center

---

## Step 930 — Lab เต็มรูปแบบ: Hybrid-Cloud Connectivity Design สำหรับ Enterprise Lab

### 930.1 โจทย์ Design

Enterprise Lab ของหลักสูตร (ตาม [00-ip-address-plan.md](00-ip-address-plan.md)) ต้องการเชื่อมต่อ
Cloud ดังนี้:

1. **WAN-EDGE-1** สร้าง Direct Connect Private VIF ไปยัง **AWS Prod-VPC** (`10.50.0.0/16`) เป็น
   เส้นทางหลักสำหรับ Traffic ที่ต้องการ Bandwidth/Latency สม่ำเสมอ
2. **Multi-VPC Aggregation** ฝั่ง AWS ใช้ **Transit Gateway** รวม Prod-VPC และ Data-VPC
   (`10.51.0.0/16`) เข้าด้วยกัน
3. **Resilience**: ถ้า Direct Connect ล่มทั้งหมด (Physical Circuit/Colocation Facility มีปัญหา)
   ต้องมี Backup Path ผ่าน **WAN-EDGE-2 ด้วย Cloud OnRamp for IaaS** (ผ่าน IPsec บน Internet
   Transport) ไปยัง VPC เดียวกัน
4. ต้อง Verify ได้ว่า Server1 (VLAN 30, `10.10.30.11`) เข้าถึง Application Tier บน AWS
   (`10.50.1.0/24`) ได้จริงผ่านเส้นทางหลัก และสลับไปเส้นทางสำรองได้เมื่อ Primary ล่ม

### 930.2 Diagram สถาปัตยกรรมเต็มรูปแบบ

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                              ENTERPRISE LAB (On-Prem)                              │
│                                                                                    │
│   Server1 (VLAN 30, 10.10.30.11) ── DIST-SW3 ── CORE-SW2 ─┬─ WAN-EDGE-1 (Primary) │
│                                                              │        │             │
│                                                              └─ WAN-EDGE-2 (Backup)  │
└──────────────────────────────────────────┬───────────────────────┬─────────────────┘
                                            │                       │
                          Direct Connect Private VIF        IPsec Tunnel over Internet
                          (169.254.100.0/30, Primary)        (Cloud OnRamp for IaaS, Backup)
                                            │                       │
                                            ▼                       ▼
                              ┌─────────────────────────────────────────────┐
                              │            AWS TRANSIT GATEWAY                  │
                              │           tgw-0a1b2c3d4e5f67890                 │
                              └──────┬───────────────────────┬────────────────┘
                                Attach                    Attach
                              ┌──────┴──────┐          ┌──────┴──────┐
                              │  Prod-VPC     │          │  Data-VPC     │
                              │  10.50.0.0/16 │          │  10.51.0.0/16 │
                              │  App Subnet:   │          │              │
                              │  10.50.1.0/24 │          │              │
                              └──────────────┘          └──────────────┘
```

### 930.3 Config เต็มรูปแบบ — WAN-EDGE-1 (Primary Path: Direct Connect via Transit VIF)

```
! ---------- Sub-interface: Transit VIF (เพื่อเข้าถึงทั้ง Prod-VPC และ Data-VPC ผ่าน TGW) ----------
WAN-EDGE-1(config)# interface TenGigabitEthernet0/0/3.200
WAN-EDGE-1(config-subif)# description ** AWS Direct Connect Transit VIF - via Transit Gateway **
WAN-EDGE-1(config-subif)# encapsulation dot1Q 200
WAN-EDGE-1(config-subif)# ip address 169.254.200.1 255.255.255.252
WAN-EDGE-1(config-subif)# exit
!
! ---------- Prefix-list: ประกาศ Summary On-Prem, รับทั้ง Prod-VPC และ Data-VPC ----------
WAN-EDGE-1(config)# ip prefix-list PL-TO-CLOUD seq 10 permit 10.10.0.0/16
WAN-EDGE-1(config)# ip prefix-list PL-FROM-CLOUD-VPC seq 10 permit 10.50.0.0/16 le 24
WAN-EDGE-1(config)# ip prefix-list PL-FROM-CLOUD-VPC seq 20 permit 10.51.0.0/16 le 24
!
! ---------- BGP: eBGP ไปยัง Direct Connect Gateway (ผ่าน Transit Gateway) ----------
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 169.254.200.2 remote-as 64512
WAN-EDGE-1(config-router)# neighbor 169.254.200.2 description ** eBGP to AWS DXGW/TGW - Transit VIF (Primary) **
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 169.254.200.2 activate
WAN-EDGE-1(config-router-af)# neighbor 169.254.200.2 prefix-list PL-TO-CLOUD out
WAN-EDGE-1(config-router-af)# neighbor 169.254.200.2 prefix-list PL-FROM-CLOUD-VPC in
WAN-EDGE-1(config-router-af)# neighbor 169.254.200.2 soft-reconfiguration inbound
WAN-EDGE-1(config-router-af)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-1(config-router-af)# exit-address-family
WAN-EDGE-1(config-router)# exit
!
WAN-EDGE-1(config)# ip route 10.10.0.0 255.255.0.0 Null0 250
```

### 930.4 Config เต็มรูปแบบ — WAN-EDGE-2 (Backup Path: Cloud OnRamp for IaaS ผ่าน IPsec)

```
! ---------- IPsec Tunnel ไปยัง cEdge เสมือนของ Cloud OnRamp for IaaS (Auto-built โดย vManage ปกติ
! แต่แสดง Manual Equivalent ที่นี่เพื่อให้เห็นกลไกจริงตามหลักการ Part 51/52) ----------
WAN-EDGE-2(config)# crypto ikev2 proposal CLOUD-ONRAMP-PROPOSAL
WAN-EDGE-2(config-ikev2-proposal)# encryption aes-cbc-256
WAN-EDGE-2(config-ikev2-proposal)# integrity sha256
WAN-EDGE-2(config-ikev2-proposal)# group 19
WAN-EDGE-2(config-ikev2-proposal)# exit
!
WAN-EDGE-2(config)# interface Tunnel200
WAN-EDGE-2(config-if)# description ** Backup Path to AWS via Cloud OnRamp for IaaS (IPsec over Internet) **
WAN-EDGE-2(config-if)# ip address 169.254.201.1 255.255.255.252
WAN-EDGE-2(config-if)# tunnel source GigabitEthernet0/0/1
WAN-EDGE-2(config-if)# tunnel mode ipsec ipv4
WAN-EDGE-2(config-if)# tunnel destination 198.51.100.10
! IP นี้คือ Public IP ของ cEdge เสมือน (Catalyst 8000V) ที่ vManage Deploy ไว้ใน AWS แล้ว
WAN-EDGE-2(config-if)# tunnel protection ipsec profile CLOUD-ONRAMP-PROFILE
WAN-EDGE-2(config-if)# exit
!
! ---------- BGP: eBGP ผ่าน Tunnel เดียวกัน เพื่อเรียนรู้ Prefix ของ Cloud VPC สำรอง ----------
WAN-EDGE-2(config)# router bgp 65001
WAN-EDGE-2(config-router)# neighbor 169.254.201.2 remote-as 64512
WAN-EDGE-2(config-router)# neighbor 169.254.201.2 description ** eBGP to AWS via Cloud OnRamp IPsec (Backup) **
WAN-EDGE-2(config-router)# address-family ipv4 unicast
WAN-EDGE-2(config-router-af)# neighbor 169.254.201.2 activate
WAN-EDGE-2(config-router-af)# neighbor 169.254.201.2 route-map DEPREF-BACKUP-PATH in
WAN-EDGE-2(config-router-af)# neighbor 169.254.201.2 prefix-list PL-TO-CLOUD out
WAN-EDGE-2(config-router-af)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-2(config-router-af)# exit-address-family
WAN-EDGE-2(config-router)# exit
!
WAN-EDGE-2(config)# route-map DEPREF-BACKUP-PATH permit 10
WAN-EDGE-2(config-route-map)# set local-preference 80
! ตั้ง Local-Preference ต่ำกว่า Default (100) ของ Path หลักที่ WAN-EDGE-1 เรียนมา
! iBGP ระหว่าง WAN-EDGE-1/WAN-EDGE-2 (Part 29 Step 287) จะเลือก Path ผ่าน WAN-EDGE-1 ก่อนเสมอ
! ยกเว้น Path นั้นหายไปจากตาราง BGP จริง (Direct Connect ล่ม) จึงจะ Fallback มาที่ Backup Path นี้
!
WAN-EDGE-2(config)# ip route 10.10.0.0 255.255.0.0 Null0 250
```

### 930.5 Verification — ยืนยัน BGP Session ฝั่ง Cloud ทั้งสองเส้นทาง

```
WAN-EDGE-1# show ip bgp neighbors 169.254.200.2
BGP neighbor is 169.254.200.2,  remote AS 64512, external link
  BGP state = Established, up for 02:10:05
  Address family IPv4 Unicast: passive
    Prefixes Current: 2 (Consumes 88 bytes), 1 (Consumes 24 bytes)
                       ^-- ได้รับ 2 Prefix: Prod-VPC (10.50.0.0/16) + Data-VPC (10.51.0.0/16)

WAN-EDGE-2# show ip bgp neighbors 169.254.201.2
BGP neighbor is 169.254.201.2,  remote AS 64512, external link
  BGP state = Established, up for 02:09:58
  Address family IPv4 Unicast: passive
    Prefixes Current: 2 (Consumes 88 bytes), 1 (Consumes 24 bytes)
```

ทั้งสอง Session `Established` พร้อมกัน — Primary (Direct Connect) และ Backup (Cloud OnRamp IPsec)
ทำงานคู่ขนาน ต่างกันแค่ Local-Preference ที่ทำให้ BGP Best Path Algorithm เลือก Primary ก่อนเสมอ

### 930.6 Route-Table Trace: Server1 (VLAN 30) → AWS App Subnet (10.50.1.0/24)

```
! ขั้นที่ 1: DIST-SW3 (Server1 อยู่ VLAN 30 ที่นี่)
DIST-SW3# show ip route 10.50.1.0
Routing entry for 10.50.0.0/16
  Known via "ospf 1", distance 110, metric 20, type extern 2
  Last update from 10.255.30.1 on Vlan30, 02:15:00 ago
  * 10.255.30.1, via GigabitEthernet1/1/1

! ขั้นที่ 2: CORE-SW2 (Redistribute OSPF -> iBGP ตาม Part 73)
CORE-SW2# show ip bgp 10.50.0.0/16
BGP routing table entry for 10.50.0.0/16
  Paths: (1 available, best #1)
    64512
      1.1.1.21 from 1.1.1.21 (1.1.1.21)     ! เรียนจาก WAN-EDGE-1 ผ่าน iBGP/RR
      Origin IGP, localpref 100, valid, internal, best

! ขั้นที่ 3: WAN-EDGE-1 (Best Path จริง - eBGP ตรงจาก Direct Connect)
WAN-EDGE-1# show ip route 10.50.1.0
Routing entry for 10.50.0.0/16
  Known via "bgp 65001", distance 20, metric 0
  * 169.254.200.2, via TenGigabitEthernet0/0/3.200

! ขั้นที่ 4: จำลอง Direct Connect ล่ม แล้วตรวจสอบว่าสลับไป Backup Path จริง
WAN-EDGE-1# show ip bgp neighbors 169.254.200.2 | include state
  BGP state = Idle    ! Direct Connect ล่มจริงตามสมมติฐาน

CORE-SW2# show ip bgp 10.50.0.0/16
BGP routing table entry for 10.50.0.0/16
  Paths: (1 available, best #1)
    64512
      1.1.1.22 from 1.1.1.22 (1.1.1.22)     ! เปลี่ยนมาเรียนจาก WAN-EDGE-2 แทน (Backup Path)
      Origin IGP, localpref 80, valid, internal, best
                       ^-- localpref 80 คือ Path สำรองที่ตั้งไว้ใน Step 930.4 กลายเป็น Best Path
                           ทันทีที่ Path หลัก (localpref 100 จาก WAN-EDGE-1) หายไปจากตาราง
```

Trace นี้พิสูจน์ **ทุกหลักการของ Part นี้ในเส้นทางเดียว**: Route Redistribution จาก OSPF เข้า BGP
(Part 73), iBGP ผ่าน Route Reflector (Part 74 Step 731), eBGP ไปยัง Cloud ผ่าน Direct Connect
(Step 922/926), และ Automatic Fallback ไปยัง Cloud OnRamp Backup Path เมื่อ Primary ล่ม (Step
928-929) — ทั้งหมดทำงานร่วมกันโดยไม่ต้องแก้ Config ใดๆ ที่ On-Prem เพิ่มเติมเลยตอนเกิด Failure จริง

---

## แบบฝึกหัดทวนความเข้าใจ Part 93

1. Internet-based VPN กับ Dedicated Connection (Direct Connect/ExpressRoute) ต่างกันหลักๆ ที่
   ประเด็นใดบ้าง และเพราะเหตุใดองค์กรที่ทำ PCI-DSS Compliance มักเลือก Dedicated Connection?
2. VIF ทั้ง 3 ประเภทของ AWS Direct Connect (Public/Private/Transit) ต่างกันอย่างไร และ Transit
   VIF ต้องใช้ร่วมกับ Resource อะไรของ AWS จึงเข้าถึงหลาย VPC ได้?
3. Azure ExpressRoute Circuit บังคับให้มี Primary/Secondary Connection เสมอ — ต่างจาก Redundancy
   ของ AWS Direct Connect อย่างไร และ Local-Preference มีบทบาทอะไรในการควบคุมว่าจะใช้เส้นทางไหน
   ก่อน?
4. AWS Transit Gateway และ Azure Virtual WAN Hub เทียบเคียงได้กับอุปกรณ์ใดใน 3-Tier Hierarchical
   Model ของ Enterprise Campus (Part 1) และเพราะเหตุใด?
5. เมื่อไหร่ Cloud OnRamp for IaaS เพียงพอ และเมื่อไหร่ต้องใช้ Direct Connect/ExpressRoute แบบ
   Dedicated จริง? ยกตัวอย่างสถานการณ์ที่ควรใช้ทั้งสองแบบร่วมกัน

**เฉลย:**

1. ต่างกันที่ Bandwidth SLA, Latency/Jitter ที่คงที่กว่า, และการไม่ผ่าน Public Internet เลย —
   PCI-DSS และ Regulation หลายฉบับมองว่า Traffic ที่ผ่าน Public Internet (แม้เข้ารหัสด้วย IPsec)
   มีความเสี่ยงเพิ่มเติมที่ Dedicated Connection ไม่มี เพราะ Physical Path คงที่และไม่ผ่าน Peering
   Point ที่ควบคุมไม่ได้
2. Public VIF เข้าถึง AWS Public Service (S3 ฯลฯ) โดยไม่ผ่าน Internet, Private VIF เข้าถึง VPC
   เดียวผ่าน Virtual Private Gateway, Transit VIF เข้าถึงหลาย VPC/Region ผ่าน **Direct Connect
   Gateway + Transit Gateway** ร่วมกัน
3. Azure บังคับ Primary+Secondary มาให้ในทุก Circuit โดย Design (ไม่ต้องสั่งเพิ่ม) ต่างจาก AWS ที่
   ต้องสั่งหลาย VIF/Connection เอง — Local-Preference ใช้กำหนดว่า BGP จะเลือก Path ไหนเป็น Best
   Path ก่อน (ค่าสูงกว่าชนะ) ทำให้ควบคุมทิศทาง Primary/Secondary ได้โดยไม่ต้องพึ่ง AS-Path
   Prepend
4. เทียบเคียงกับ **CORE-SW1/CORE-SW2** เพราะทำหน้าที่เป็นจุดรวม Traffic กลางที่ Aggregate หลาย
   VPC/VNet (เทียบเท่า DIST-SW) เข้าด้วยกัน ก่อนส่งต่อออกไปสู่ WAN-EDGE (เทียบเท่า Direct Connect
   Gateway) — เป็นสถาปัตยกรรม Hierarchical แบบเดียวกับ 3-Tier Model ที่ Part 1 สอนไว้ทุกประการ
5. Cloud OnRamp พอเมื่อ Bandwidth ไม่เกิน ~1Gbps, ไม่มีข้อบังคับ Compliance ที่ห้าม Traffic ผ่าน
   Internet Transport, และต้องการเปิดใช้งานเร็ว — ต้องใช้ Dedicated Connection เมื่อ Bandwidth สูง
   ต่อเนื่อง หรือ Compliance บังคับ Private Path จริง — Pattern ที่พบบ่อยคือใช้ Direct Connect ที่
   HQ/Data Center (Traffic เยอะ) ร่วมกับ Cloud OnRamp ที่ Branch (Traffic กระจัดกระจาย น้อยกว่า)
   เข้า VPC/VNet เดียวกัน

---

## สรุป Part 93

Part นี้ปิดช่องว่างที่ [Part 74 Step 735](part-074-advanced-bgp-wan-edge.md#step-735--bgp-สำหรับ-multi-cloud-connectivity-preview)
และ [Part 76 Step 755-756](part-076-sdwan-advanced-design.md#step-755--cloud-onramp-for-saas-เชิงลึก-เลือก-local-breakout-ที่ดีที่สุดต่อแอปโดยอัตโนมัติ)
Preview ไว้ทั้งคู่ — เราสร้าง **Dedicated Cloud Connectivity เต็มรูปแบบ**ด้วย eBGP บน AWS Direct
Connect (Virtual Interface, Direct Connect Gateway), เทียบเคียงกับ Azure ExpressRoute (Circuit,
Peering Type) และ GCP Cloud/Partner Interconnect (VLAN Attachment, Cloud Router) ผ่านตาราง
3-Way Comparison ที่ใช้ได้ทั้งการเลือก Provider และการสอบ CCIE ออกแบบ **Transit Gateway/Virtual
WAN Hub** เป็น "Hub ฝั่ง Cloud" ที่ Mirror แนวคิด Core/Distribution ของ Enterprise Campus, เดิน
**BGP Route Propagation** เต็มเส้นทางจาก Campus ถึง VPC, ขยาย **Security Stack ของ Part 78**
เข้าสู่ Cloud Workload ด้วยกรอบคิด Shared Responsibility, เจาะลึก **Cloud OnRamp** ว่าเมื่อไหร่
พอและเมื่อไหร่ต้อง Dedicated Connection จริง และปิดท้ายด้วย **Multi-Cloud Resilience Design**
ที่นำหลักการ RTO/RPO จาก [Part 92](part-092-disaster-recovery-network-resilience.md) มาประยุกต์
ใช้กับ Cloud Dependency โดยตรง — Lab เต็มรูปแบบใน Step 930 พิสูจน์ว่าทุกกลไก (OSPF Redistribution,
iBGP Route Reflector, eBGP บน Direct Connect, Automatic Fallback ไปยัง Cloud OnRamp) ทำงาน
ร่วมกันเป็นเส้นทางเดียวได้จริงโดยไม่ต้องแก้ Config เพิ่มตอนเกิด Failure

✅ **พร้อมสำหรับ Part 94**: เราจะยกระดับแนวคิด Security ทั้งหมดที่สร้างมา (ISE/TrustSec/FTD/
Umbrella ใน Part 78, Cloud Security ใน Part นี้) ขึ้นสู่ **สถาปัตยกรรม Zero Trust Network**
เต็มรูปแบบ — หลักการ "Never Trust, Always Verify", Continuous Authentication, และการออกแบบ
Enterprise Network ที่ไม่พึ่ง Perimeter-based Security แบบเดิมอีกต่อไป

**ไปต่อ:** [Part 94 — Zero Trust Network Architecture →](part-094-zero-trust-network-architecture.md)
