# Part 86 — MPLS L3VPN Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 851–860 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 85 — VXLAN EVPN Multi-Site Advanced](part-085-vxlan-evpn-multisite-advanced.md) ที่ปิด
> เรื่อง Data Center Fabric แบบ VXLAN EVPN ข้าม Site ไปแล้ว Part นี้จะพาเรากลับไปที่มุมมอง **WAN**
> อีกครั้ง แต่คราวนี้เปลี่ยนจากมุมมอง "ลูกค้า (Enterprise)" ไปเป็นมุมมอง **"Service Provider (SP)"**
> ที่แท้จริง — [Part 19 Step 186-187](part-019-wan-technologies.md) เคยแนะนำแนวคิด MPLS L3VPN
> แบบผ่านๆ จากมุมมอง CE (WAN-EDGE-1 ต่อ eBGP เข้ากับ PE เส้นเดียว) และ**ตั้งใจค้างรายละเอียดฝั่ง
> SP ไว้ทั้งหมด** โดยระบุชัดว่า "จะสอนคอนฟิก PE/P Router แบบเต็มรูปแบบใน Part 86-87" — Part นี้
> คือจุดที่เราทำตามสัญญานั้น
>
> เราจะสร้าง **SP Core จำลองขนาดเล็ก** ขึ้นมาใหม่ 4 ตัว: **SP-P1/SP-P2** (Provider Core Router
> — MPLS LSR) และ **SP-PE1/SP-PE2** (Provider Edge Router — MPLS LER) เพื่อแทนที่ **ISP-RTR**
> ตัวจำลอง Internet เดิม **เฉพาะสำหรับขอบเขตการสอนของ Part นี้เท่านั้น** (ISP-RTR ยังคงเป็น
> ตัวแทน Internet Access ปกติในภาพรวมหลักสูตร — SP Topology ใหม่นี้ใช้เพื่อสอนแนวคิด MPLS L3VPN
> แบบ Provider-grade อย่างละเอียดโดยเฉพาะ) **WAN-EDGE-1** จะยังคงทำหน้าที่ **CE (Customer Edge)**
> เดิมตาม Part 19 — ซื้อบริการ MPLS L3VPN จาก SP รายนี้ — และเราจะเพิ่ม Site ที่สองของลูกค้าเข้ามา
> คือ **REMOTE-SITE-CE** เพื่อพิสูจน์โมเดล **Any-to-Any Connectivity** ที่เป็นจุดขายหลักของ
> MPLS L3VPN จริง เนื้อหานี้เชื่อมโยงกับ **VRF-Lite ([Part 51](part-051-vrf-lite-gre-ipsec.md))**
> และ **eBGP ([Part 29](part-029-bgp-fundamentals.md))** ที่เรียนไปแล้ว — Part นี้จะแสดงให้เห็นว่า
> เมื่อ VRF ผูกกับ MPLS Label Switching และ MP-BGP จริง มันจะกลายเป็น "VRF เวอร์ชันเต็มรูปแบบ"
> ที่ Scale ได้ระดับ Service Provider

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 851 | MPLS ทวนความเข้าใจเชิงลึก — Label Switching Mechanics และ LDP (Label Distribution Protocol) เต็มรูปแบบใน SP Core |
| 852 | ความสัมพันธ์ PE-CE — ลูกค้าเห็นแค่ IP Routing ปกติ ไม่รู้จัก MPLS เลย (eBGP PE-CE เต็มรูปแบบ) |
| 853 | VRF บน PE Router — Multi-Tenancy ที่ผูกกับ MPLS อย่างแท้จริง (`ip vrf`/`rd`/`route-target`) |
| 854 | MP-BGP VPNv4 — วิธีที่ SP Core แจก VPN Route ระหว่าง PE ด้วย MP-iBGP + Route Distinguisher |
| 855 | Label Stacking — 2 Label ต่อ Packet (Transport Label ชั้นนอก + VPN Label ชั้นใน) |
| 856 | Lab เต็มรูปแบบ End-to-End — WAN-EDGE-1 ↔ SP-PE1 ↔ MPLS Core ↔ SP-PE2 ↔ REMOTE-SITE-CE (Any-to-Any) |
| 857 | Route Target Import/Export เชิงลึก — Hub-and-Spoke VPN ด้วย Asymmetric RT |
| 858 | MPLS L3VPN QoS — CoS Tier ของ SP และการ Map DSCP ลูกค้าเข้ากับ MPLS EXP Bit |
| 859 | Verification/Troubleshooting — LDP Neighbor, MPLS Forwarding Table, BGP VPNv4, Route ใน VRF |
| 860 | Lab เต็มรูปแบบสุดท้าย — Full Configuration ทุกอุปกรณ์ + พิสูจน์ Any-to-Any Reachability |

---

## Step 851 — MPLS ทวนความเข้าใจเชิงลึก และ LDP เต็มรูปแบบใน SP Core

### 851.1 ทวนแนวคิด Label Switching จาก Part 19 แบบเจาะลึกกว่าเดิม

[Part 19 Step 186](part-019-wan-technologies.md) อธิบายแนวคิด Label Switching แบบภาพรวมไว้แล้วว่า
Router Core (LSR) แค่ **Swap Label** โดยไม่ต้องดู IP Header เต็มรูปแบบ ทำให้ Forward เร็วกว่า
Traditional Routing — Part นี้จะลงรายละเอียดว่า **Label เกิดขึ้นได้อย่างไร** และ **Router รู้ว่า
ต้อง Swap เป็น Label อะไรได้อย่างไร**

MPLS แบ่งกระบวนการทำงานเป็น 2 ส่วนที่แยกจากกันชัดเจน (Control Plane vs Data Plane):

| Plane | ทำหน้าที่ | Protocol ที่เกี่ยวข้อง |
|---|---|---|
| **Control Plane** | ตัดสินใจว่า Label ไหนคู่กับ Prefix ไหน และแจก (Distribute) ค่านี้ให้ Router ข้างเคียงรู้ | **LDP** (Label Distribution Protocol) — Part นี้ / RSVP-TE (Traffic Engineering — นอกขอบเขต Part นี้) |
| **Data Plane** | Forward Packet จริงตาม Label โดยดู **LFIB (Label Forwarding Information Base)** ที่สร้างจาก Control Plane | Label Swapping ที่ Hardware ASIC/CEF |

### 851.2 LDP (Label Distribution Protocol) คืออะไร

**LDP (RFC 5036)** คือ Protocol ที่ Router MPLS (LSR/LER) ใช้เพื่อ **แจก Label ให้ Router
ข้างเคียงที่เป็น LDP Neighbor** โดยอัตโนมัติ ครอบคลุมทุก Prefix ที่มีอยู่ใน **IGP** (OSPF/EIGRP/
IS-IS) ของ SP Core — LDP **ไม่ใช่ Routing Protocol** มันไม่คำนวณ Best Path เอง แต่ "ยืม" ผลลัพธ์
จาก IGP ที่มีอยู่แล้วมาผูก Label ให้ทุก Prefix จึงต้องมี **IGP รันอยู่ก่อนเสมอ** (LDP พึ่งพา
Routing Table ของ IGP 100%)

```
                     ลำดับการทำงานของ MPLS Core (ต้องเรียงตามนี้เสมอ)

   1) IGP (OSPF Area 0)  ──►  ทุก Router รู้จัก Loopback0 ของกันและกัน (Reachability พื้นฐาน)
   2) LDP                ──►  ผูก Label ให้ทุก Prefix ที่ IGP เรียนรู้ (สร้าง LSP หรือ Label
                                Switched Path ตาม IGP Best Path โดยอัตโนมัติ)
   3) MP-BGP VPNv4        ──►  แจก VPN Route ระหว่าง PE (Step 854) — ใช้ LSP จาก Step 1-2
                                เป็น "ถนน" ขนส่ง Label ของ VPN
```

### 851.3 LDP Session และ Message Type ที่ต้องรู้

| ขั้นตอน | Message | หน้าที่ |
|---|---|---|
| 1. Discovery | **LDP Hello** (UDP/646, Multicast 224.0.0.2) | ค้นหา LDP Neighbor บน Link — ส่งเป็นระยะทุก Interface ที่เปิด `mpls ip` |
| 2. Session Establishment | **TCP/646** | เมื่อเจอกันจาก Hello แล้ว สร้าง TCP Session เพื่อแลกข้อมูล Label แบบ Reliable |
| 3. Label Advertisement | **Label Mapping Message** | แจกคู่ **(Prefix (FEC), Label)** ให้ Neighbor — ค่า Default คือ "Unsolicited Downstream" (แจกให้เองโดยไม่ต้องขอ) |
| 4. Label Withdraw | **Label Withdraw Message** | ถอน Label คืนเมื่อ Prefix หายไปจาก IGP (Link ล่ม, Route เปลี่ยน) |

> **FEC (Forwarding Equivalence Class)**: กลุ่มของ Packet ที่ถูก Forward แบบเดียวกัน (ปกติคือ
> Prefix ปลายทางเดียวกันจาก IGP) — LDP แจก 1 Label ต่อ 1 FEC เสมอ

### 851.4 Topology ของ SP Core ที่จะสร้างใน Part นี้

```
                     SP Core (MPLS Backbone) — AS 65000 (Single-AS Provider)

        WAN-EDGE-1                                              REMOTE-SITE-CE
        (CE, HQ, AS 65001)                                      (CE, Branch, AS 65002)
             │ 192.0.2.0/30                                           │ 192.0.2.4/30
             │ .2                                                      │ .6
       ┌─────┴──────┐   100.64.1.0/30    ┌──────────┐  100.64.1.4/30  ┌──────────┐  100.64.1.8/30   ┌─────┴──────┐
       │  SP-PE1     ├────────────────────┤  SP-P1    ├─────────────────┤  SP-P2    ├──────────────────┤  SP-PE2     │
       │  (PE / LER) │ .1              .2 │ (P / LSR) │ .5           .6 │ (P / LSR) │ .9            .10│  (PE / LER) │
       └─────────────┘                    └──────────┘                  └──────────┘                  └─────────────┘
       Lo0 100.64.0.11/32                 Lo0 100.64.0.1/32             Lo0 100.64.0.2/32              Lo0 100.64.0.12/32
```

> **หมายเหตุ IP Addressing**: Address Space `100.64.0.0/10` (RFC 6598 — Shared Address Space
> สำหรับ CGN) ถูกเลือกใช้เฉพาะ **ภายใน SP Core Backbone** ของ Scenario นี้ เพื่อไม่ให้ชนกับ
> `10.0.0.0/8` ที่ Enterprise Lab ใช้อยู่แล้วตาม [00-ip-address-plan.md](00-ip-address-plan.md) —
> สอดคล้องกับแนวทางที่ [Part 19](part-019-wan-technologies.md) เคยใช้ IP วงแยกสำหรับ Scenario
> BRANCH-RTR เช่นกัน ส่วน Link ระหว่าง CE-PE ยังคงใช้ `192.0.2.0/30` เดิมจาก
> [Part 19 Step 187](part-019-wan-technologies.md) เพื่อความต่อเนื่อง (PE=.1, WAN-EDGE-1=.2)

### 851.5 คอนฟิก IGP (OSPF) พื้นฐานของ SP Core — Prerequisite ก่อน LDP

LDP ต้องพึ่ง IGP เสมอ — เราใช้ **OSPF Process 100** (แยกเลข Process จาก `router ospf 1` ของ
Enterprise Campus เพื่อไม่ให้สับสนว่าเป็นคนละ Routing Domain กันโดยสิ้นเชิง)

**บน SP-PE1:**

```
SP-PE1(config)# interface Loopback0
SP-PE1(config-if)# description ** MPLS Router-ID / MP-BGP Update Source **
SP-PE1(config-if)# ip address 100.64.0.11 255.255.255.255
SP-PE1(config-if)# exit
!
SP-PE1(config)# interface GigabitEthernet0/0/1
SP-PE1(config-if)# description ** Core Link to SP-P1 **
SP-PE1(config-if)# ip address 100.64.1.1 255.255.255.252
SP-PE1(config-if)# no shutdown
SP-PE1(config-if)# exit
!
SP-PE1(config)# router ospf 100
SP-PE1(config-router)# router-id 100.64.0.11
SP-PE1(config-router)# network 100.64.0.11 0.0.0.0 area 0
SP-PE1(config-router)# network 100.64.1.0 0.0.0.3 area 0
SP-PE1(config-router)# exit
```

**บน SP-P1:**

```
SP-P1(config)# interface Loopback0
SP-P1(config-if)# ip address 100.64.0.1 255.255.255.255
SP-P1(config-if)# exit
!
SP-P1(config)# interface GigabitEthernet0/0/1
SP-P1(config-if)# description ** Core Link to SP-PE1 **
SP-P1(config-if)# ip address 100.64.1.2 255.255.255.252
SP-P1(config-if)# no shutdown
SP-P1(config-if)# exit
!
SP-P1(config)# interface GigabitEthernet0/0/2
SP-P1(config-if)# description ** Core Link to SP-P2 **
SP-P1(config-if)# ip address 100.64.1.5 255.255.255.252
SP-P1(config-if)# no shutdown
SP-P1(config-if)# exit
!
SP-P1(config)# router ospf 100
SP-P1(config-router)# router-id 100.64.0.1
SP-P1(config-router)# network 100.64.0.1 0.0.0.0 area 0
SP-P1(config-router)# network 100.64.1.0 0.0.0.3 area 0
SP-P1(config-router)# network 100.64.1.4 0.0.0.3 area 0
SP-P1(config-router)# exit
```

**บน SP-P2 และ SP-PE2**: คอนฟิกแบบเดียวกัน (สลับ Neighbor และ IP ให้ตรงตาม Diagram 851.4) —
SP-P2 ต่อกับ SP-P1 (`100.64.1.4/30`) และ SP-PE2 (`100.64.1.8/30`); SP-PE2 ต่อกับ SP-P2 เท่านั้น
ที่ฝั่ง Core (ฝั่งลูกค้าจะคอนฟิกใน Step 852-853)

### 851.6 เปิด MPLS + LDP บนทุก Interface ของ SP Core

คำสั่งเดียวที่เปิด MPLS Forwarding และ LDP พร้อมกันคือ `mpls ip` บน Interface (สั่งบน Interface
ฝั่ง **Core เท่านั้น** — Interface ฝั่ง CE **ห้ามเปิด MPLS** เพราะ CE ต้องไม่รู้จัก MPLS เลย
ตามหลักการ Step 852)

```
! บน SP-PE1
SP-PE1(config)# mpls label protocol ldp
SP-PE1(config)# mpls ldp router-id Loopback0 force
SP-PE1(config)# interface GigabitEthernet0/0/1
SP-PE1(config-if)# mpls ip
SP-PE1(config-if)# exit

! บน SP-P1 (ทั้งสอง Core Interface)
SP-P1(config)# mpls label protocol ldp
SP-P1(config)# mpls ldp router-id Loopback0 force
SP-P1(config)# interface range GigabitEthernet0/0/1 , GigabitEthernet0/0/2
SP-P1(config-if-range)# mpls ip
SP-P1(config-if-range)# exit

! บน SP-P2 (ทั้งสอง Core Interface) และ SP-PE2 (Interface เดียวฝั่ง Core) — รูปแบบเดียวกัน
```

| คำสั่ง | ความหมาย |
|---|---|
| `mpls label protocol ldp` | ระบุให้ใช้ LDP เป็น Label Distribution Protocol (ทางเลือกอื่นคือ TDP แบบเก่า — ไม่ใช้แล้วในปัจจุบัน) |
| `mpls ldp router-id Loopback0 force` | บังคับให้ LDP ใช้ Loopback0 เป็น Router-ID เสมอ (`force` = ใช้ค่านี้แม้ Interface จะ Down ชั่วคราว ป้องกัน LDP Session หลุดเวลา Physical Link กระพริบ) |
| `mpls ip` (บน Interface) | เปิด MPLS Forwarding + LDP Hello บน Interface นั้น — Interface นี้จะกลายเป็น Label Switching Interface ทันที |

### 851.7 Verify LDP Session

```
SP-PE1# show mpls ldp neighbor
    Peer LDP Ident: 100.64.0.1:0; Local LDP Ident 100.64.0.11:0
        TCP connection: 100.64.0.1.646 - 100.64.0.11.53024
        State: Oper; Msgs sent/rcvd: 15/14; Downstream
        Up time: 00:12:03
        LDP discovery sources:
          GigabitEthernet0/0/1, Src IP addr: 100.64.1.2
        Addresses bound to peer LDP Ident:
          100.64.0.1        100.64.1.1        100.64.1.2        100.64.1.5

SP-PE1# show mpls interfaces
Interface              IP            Tunnel   BGP Static Operational
GigabitEthernet0/0/1    Yes (ldp)     No       No  No     Yes
```

- `State: Oper` = LDP Session สำเร็จสมบูรณ์ (คล้าย `LCP: Open` ของ PPP ใน Part 19)
- `Downstream` = ทิศทางการแจก Label แบบ Unsolicited Downstream (ค่า Default ของ Cisco IOS LDP)

### 851.8 ผลลัพธ์: LFIB ถูกสร้างขึ้นเองอัตโนมัติ

หลังจาก LDP Session ขึ้นครบทุก Hop ของ SP Core, ทุก Router จะมี **LFIB (Label Forwarding
Information Base)** ที่ผูก Label เข้ากับทุก Prefix ของ IGP โดยอัตโนมัติ — ตัวอย่างที่ SP-P1:

```
SP-P1# show mpls forwarding-table
Local  Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label  Label      or Tunnel Id     Switched      interface
16     Pop Label  100.64.0.11/32   0             Gi0/0/1    100.64.1.1
17     19         100.64.0.12/32   0             Gi0/0/2    100.64.1.6
18     Pop Label  100.64.0.1/32    0             aggregate
```

- แถวที่สอง (`Prefix 100.64.0.12/32` = Loopback ของ SP-PE2) แสดงให้เห็นว่า SP-P1 **Swap Label
  17 (ที่ตัวเองแจกให้ Neighbor ทางซ้าย) เป็น Label 19 (ที่เรียนมาจาก SP-P2 ทางขวา)** — นี่คือ
  แก่นแท้ของ Label Switching ที่ Step 855 จะขยายรายละเอียดต่อ
- แถวแรก (`Pop Label`) คือกรณีที่ Prefix นั้นอยู่ **ห่างจาก Router นี้เพียง 1 Hop** (Penultimate
  Hop Popping — PHP) จึงไม่ต้องแปะ Label เปล่าๆ ให้ Egress Router รับภาระถอด Label โดยไม่จำเป็น

---

## Step 852 — ความสัมพันธ์ PE-CE: ลูกค้าเห็นแค่ IP Routing ปกติ

### 852.1 หลักการสำคัญที่สุดของ MPLS L3VPN

**CE Router ของลูกค้า (WAN-EDGE-1) ไม่รู้จัก MPLS แม้แต่คำสั่งเดียว** — Interface ที่ WAN-EDGE-1
ต่อไปยัง SP-PE1 เป็น **IP Link ปกติ 100%** ไม่มี `mpls ip`, ไม่มี Label ปรากฏบน CE เลย นี่คือสิ่งที่
[Part 19 Step 186-187](part-019-wan-technologies.md) พิสูจน์ไปแล้วในทางปฏิบัติ — Part นี้เพียง
เติมส่วนที่ขาด: **สิ่งที่เกิดขึ้นจริงฝั่ง SP (SP-PE1) ที่ทำให้ภาพลวงนี้เป็นไปได้**

```
              มุมมองของ WAN-EDGE-1 (CE)                     ความเป็นจริงฝั่ง SP
   ┌─────────────────────────────────────┐      ┌──────────────────────────────────────┐
   │  "ฉันมี eBGP Neighbor 1 ตัว คือ       │      │  SP-PE1 มี VRF แยกต่อลูกค้า, ทำ MP-BGP │
   │   192.0.2.1 (AS 65000) — เขาส่ง       │ ◄──► │  VPNv4 กับ SP-PE2, ใส่ Label 2 ชั้น    │
   │   Route ของ Site อื่นๆ มาให้เอง"      │      │  ก่อนส่งเข้า MPLS Core — ทั้งหมดนี้     │
   │   ไม่มี MPLS/Label/VRF อะไรให้เห็นเลย  │      │  "ล่องหน" จากมุมมอง CE โดยสมบูรณ์      │
   └─────────────────────────────────────┘      └──────────────────────────────────────┘
```

### 852.2 คอนฟิก eBGP PE-CE — ฝั่ง WAN-EDGE-1 (CE)

คอนฟิกนี้ **เหมือนกับ [Part 19 Step 187](part-019-wan-technologies.md) ทุกประการ** (พิสูจน์ว่า
CE ไม่ต้องเปลี่ยนคอนฟิกอะไรเลย แม้ SP จะเปลี่ยนจาก ISP-RTR ตัวเดียวมาเป็น Full SP Core 4 ตัว
ก็ตาม) — ใช้ทักษะ eBGP เดียวกันกับที่เรียนใน [Part 29](part-029-bgp-fundamentals.md)

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# description ** MPLS L3VPN Access Link to Provider PE (SP-PE1) **
WAN-EDGE-1(config-if)# ip address 192.0.2.2 255.255.255.252
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp router-id 1.1.1.21
WAN-EDGE-1(config-router)# neighbor 192.0.2.1 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 192.0.2.1 description ** eBGP to Provider PE SP-PE1 (MPLS L3VPN) **
WAN-EDGE-1(config-router)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-1(config-router)# exit
```

- **AS 65001** = Customer AS ของ WAN-EDGE-1 (ใช้ต่อจาก Part 29/19)
- **AS 65000** = Provider AS ของ SP ทั้ง Core (SP-PE1, SP-PE2, SP-P1, SP-P2 ทุกตัวอยู่ AS เดียวกัน
  — SP มักใช้ **Single-AS Design** สำหรับ Backbone ของตัวเอง แล้วรัน iBGP/MP-iBGP ภายใน)
- `network 10.10.0.0 mask 255.255.0.0` = ประกาศ Aggregate ของทุก VLAN ใน Campus (`10.10.0.0/16`
  ครอบคลุมทั้ง VLAN 10/20/30/40/99 ตาม [00-ip-address-plan.md](00-ip-address-plan.md)) เข้าไปใน
  MPLS VPN เพื่อให้ Site อื่น (REMOTE-SITE-CE) เห็น Route ของ HQ

### 852.3 คอนฟิก eBGP PE-CE — ฝั่ง SP-PE1 (ภายใน VRF)

ฝั่ง PE ต้องรัน eBGP **ภายใน VRF ของลูกค้า** (ไม่ใช่ Global Table) — นี่คือจุดที่ VRF (Step 853)
กับ eBGP PE-CE มาบรรจบกัน รายละเอียดการสร้าง VRF จะอธิบายเต็มใน Step 853 ที่นี่ขอโฟกัสที่ eBGP:

```
SP-PE1(config)# interface GigabitEthernet0/0/2
SP-PE1(config-if)# description ** CE-Facing Access Link to WAN-EDGE-1 (Customer: ENTLAB) **
SP-PE1(config-if)# vrf forwarding CUST-ENTLAB
SP-PE1(config-if)# ip address 192.0.2.1 255.255.255.252
SP-PE1(config-if)# no shutdown
SP-PE1(config-if)# exit
!
SP-PE1(config)# router bgp 65000
SP-PE1(config-router)# address-family ipv4 vrf CUST-ENTLAB
SP-PE1(config-router-af)# neighbor 192.0.2.2 remote-as 65001
SP-PE1(config-router-af)# neighbor 192.0.2.2 description ** eBGP to Customer CE WAN-EDGE-1 (HQ) **
SP-PE1(config-router-af)# neighbor 192.0.2.2 activate
SP-PE1(config-router-af)# exit-address-family
```

> **สำคัญ**: Interface `GigabitEthernet0/0/2` ของ SP-PE1 **ไม่มี `mpls ip`** — เพราะเป็น
> Interface ฝั่ง CE ล้วนๆ MPLS ทำงานเฉพาะฝั่ง Core (`GigabitEthernet0/0/1` ที่ต่อกับ SP-P1)
> นี่คือกฎเหล็กที่ทำให้ CE "มองไม่เห็น" MPLS ได้จริงในทางเทคนิค — Label จะถูกเติมเข้า Packet
> **หลังจาก** SP-PE1 รับ Packet จาก CE และตัดสินใจ Forward เข้า Core เท่านั้น

### 852.4 หลักการ AS ของแต่ละ CE Site — เมื่อไหร่ต้องใช้ `as-override`

ใน Lab นี้ WAN-EDGE-1 ใช้ AS 65001 และ REMOTE-SITE-CE (Step 856) จะใช้ AS 65002 — **คนละ AS กัน**
ทำให้ไม่มีปัญหา AS_PATH Loop Prevention ของ BGP เลย แต่ในโลกจริง **ลูกค้าองค์กรใหญ่มักใช้ AS
เดียวกันทุก Site** (เพื่อบริหารง่าย) ซึ่งจะชน AS_PATH Loop Prevention เพราะ PE ที่ Advertise
Route กลับไปยัง CE อีก Site จะเห็น AS ของตัวเองอยู่ใน AS_PATH แล้ว BGP จะ Drop Route นั้นทิ้งเอง
— วิธีแก้คือคำสั่ง `neighbor <ip> as-override` บน PE (แทนที่ AS ของ Customer ในสำเนาที่ส่งกลับไป
ยัง Site อื่นด้วย AS ของ SP เอง) หรือ `neighbor <ip> allowas-in <n>` ฝั่ง CE (อนุญาตให้ AS ของ
ตัวเองปรากฏใน Path ได้ไม่เกิน n ครั้ง) — Part นี้ไม่ต้องใช้เทคนิคนี้เพราะออกแบบ AS ต่างกันไว้แล้ว
แต่เป็นประเด็น Troubleshoot ที่พบบ่อยที่สุดของ MPLS L3VPN ระดับ CCIE SP

---

## Step 853 — VRF บน PE Router: Multi-Tenancy ที่ผูกกับ MPLS อย่างแท้จริง

### 853.1 ทวนความแตกต่างจาก VRF-Lite (Part 51)

[Part 51 Step 501.3](part-051-vrf-lite-gre-ipsec.md) เคยตั้งตารางเปรียบเทียบ **VRF-Lite** กับ
**Full/MPLS VRF** ไว้แล้ว และบอกว่า Full VRF "จะสอนใน Part 86-87" — นี่คือ Part นั้น สิ่งที่ต่างจาก
VRF-Lite อย่างเป็นรูปธรรมคือ:

| คุณสมบัติ | VRF-Lite (Part 51, DIST-SW3) | Full MPLS VRF (Part นี้, SP-PE1) |
|---|---|---|
| การแลก Route ข้าม PE | ไม่มี (VRF-Lite ทำงานแค่ Local Router เดียว หรือ Hop-by-Hop) | **MP-iBGP VPNv4** ข้าม PE ทั้งหมดอัตโนมัติ (Step 854) |
| `rd`/`route-target` | จองไว้ล่วงหน้า **ไม่มีผลจริง** เพราะไม่มี MP-BGP | **มีผลจริง 100%** — ใช้แยก VPN และควบคุม Import/Export จริง |
| Data Plane | IP Routing ปกติ ไม่มี Label | **MPLS Label 2 ชั้น** (Step 855) ขนส่งผ่าน Core |
| Scale | เหมาะ Campus/DC ขนาดกลาง | เหมาะ SP Backbone นับพัน PE, นับหมื่น VPN |

### 853.2 คอนฟิก VRF บน SP-PE1 — `vrf definition` (IOS-XE Syntax ปัจจุบัน)

VRF ของลูกค้า ENTLAB (WAN-EDGE-1 + REMOTE-SITE-CE) ถูกตั้งชื่อ **`CUST-ENTLAB`** บน SP-PE1
(และ SP-PE2 ใน Step 854/856 ด้วยชื่อเดียวกัน — ชื่อ VRF เป็น Local Significance เท่านั้น ไม่ต้อง
เหมือนกันข้าม PE ก็ได้ในทางเทคนิค แต่ตั้งชื่อเดียวกันเพื่อให้อ่าน Config ง่ายและลดความสับสน)

```
SP-PE1(config)# vrf definition CUST-ENTLAB
SP-PE1(config-vrf)# description ** MPLS L3VPN Customer: Enterprise Lab (WAN-EDGE-1 HQ Site) **
SP-PE1(config-vrf)# rd 65000:100
SP-PE1(config-vrf)# address-family ipv4
SP-PE1(config-vrf-af)# route-target export 65000:100
SP-PE1(config-vrf-af)# route-target import 65000:100
SP-PE1(config-vrf-af)# exit-address-family
SP-PE1(config-vrf)# exit
```

| คำสั่ง | ความหมาย — ต่างจาก VRF-Lite อย่างไร |
|---|---|
| `rd 65000:100` | **ใช้จริง**: ทำให้ Prefix ของลูกค้า ENTLAB (เช่น `10.10.0.0/16`) กลายเป็น VPNv4 Prefix ที่ไม่ซ้ำใครในตาราง MP-BGP ของ SP แม้ลูกค้ารายอื่นจะใช้ Subnet `10.10.0.0/16` เหมือนกันก็ตาม (**Overlapping Address Space** — จุดขายสำคัญของ MPLS L3VPN) — รูปแบบ `<SP-ASN>:<Number ใดๆ ที่ SP กำหนด>` |
| `route-target export/import 65000:100` | **ใช้จริง**: ควบคุมว่า Route จาก VRF นี้จะถูก "แปะป้าย" Community อะไรก่อนส่งเข้า MP-BGP (`export`) และ Route ที่มี Community อะไรบ้างที่จะถูก "รับเข้า" VRF นี้ (`import`) — ค่าเดียวกันทั้ง Export/Import แบบนี้คือ **Any-to-Any Model** (Step 856); ถ้าต่างกันคือ **Hub-and-Spoke** (Step 857) |

> **ทำไมใช้ `vrf definition` ไม่ใช่ `ip vrf`**: `ip vrf` เป็น Syntax รุ่นเก่า (IOS Classic) ที่
> เลิกใช้แล้วบน IOS-XE — [Part 51 Step 502.2](part-051-vrf-lite-gre-ipsec.md) อธิบายไว้แล้วว่า
> `vrf definition` คือ Syntax มาตรฐานปัจจุบันของ Catalyst/ISR/ASR ทุกรุ่นตาม Baseline ของหลักสูตรนี้
> — แนวคิด RD/RT เหมือนกันทั้งสอง Syntax เพียงคำสั่งเขียนต่างกัน

### 853.3 ผูก Interface CE-Facing เข้ากับ VRF

```
SP-PE1(config)# interface GigabitEthernet0/0/2
SP-PE1(config-if)# vrf forwarding CUST-ENTLAB
SP-PE1(config-if)# ip address 192.0.2.1 255.255.255.252
SP-PE1(config-if)# no shutdown
SP-PE1(config-if)# exit
```

> **ข้อสังเกตสำคัญ**: การสั่ง `vrf forwarding` บน Interface ที่มี `ip address` ตั้งไว้แล้ว จะทำให้
> IP Address นั้น **ถูกล้างออกจาก Global Table โดยอัตโนมัติ** ต้องพิมพ์ `ip address` ใหม่อีกครั้ง
> เสมอหลังสั่ง `vrf forwarding` (Cisco IOS Behavior — พบบ่อยที่สุดในข้อผิดพลาดของผู้เริ่มต้น VRF)

### 853.4 Multi-Tenancy บน SP-PE1 ตัวเดียว — ตัวอย่างแนวคิด (ไม่ใช้จริงใน Lab หลัก)

จุดแข็งของ Full VRF คือ **PE ตัวเดียวรองรับลูกค้าหลายรายพร้อมกันโดยแยก Routing Table สมบูรณ์**
ตัวอย่างสมมติถ้า SP-PE1 มีลูกค้าอีกราย "ACME-CORP" ที่ใช้ Subnet `10.10.0.0/16` เหมือนกับ ENTLAB
โดยบังเอิญ (Overlapping IP):

```
SP-PE1(config)# vrf definition CUST-ACME
SP-PE1(config-vrf)# rd 65000:200
SP-PE1(config-vrf)# address-family ipv4
SP-PE1(config-vrf-af)# route-target export 65000:200
SP-PE1(config-vrf-af)# route-target import 65000:200
SP-PE1(config-vrf-af)# exit-address-family
```

- แม้ `10.10.0.0/16` ของ ENTLAB (RD `65000:100`) และของ ACME-CORP (RD `65000:200`) จะซ้ำกัน
  Prefix เดิม แต่ **RD ที่ต่างกันทำให้ทั้งสองกลายเป็น VPNv4 Prefix ที่ไม่ซ้ำกันเลย** ในตาราง
  MP-BGP ของ SP — เก็บอยู่ใน `RIB` คนละชุดกัน (`show ip route vrf CUST-ENTLAB` เห็นแค่ของ ENTLAB,
  `show ip route vrf CUST-ACME` เห็นแค่ของ ACME) **ไม่มีทางรั่วไหลถึงกันได้เลยแม้ Subnet ชนกัน** —
  นี่คือสิ่งที่ VRF-Lite ธรรมดา (ไม่มี RD) ทำไม่ได้ในสถานการณ์ Overlapping IP ข้าม PE

---

## Step 854 — MP-BGP VPNv4: กระดูกสันหลังของการแจก VPN Route ข้าม PE

### 854.1 ทำไมต้องมี Address-Family ใหม่ (VPNv4)

BGP มาตรฐาน (Address-Family `ipv4 unicast`) แจกแค่ **Prefix + Next-Hop + Path Attribute**
ไม่มีที่เก็บ RD หรือบอกว่า Route นี้เป็นของ VRF ไหน — MPLS L3VPN จึงต้องใช้ **VPNv4 Address-Family**
ซึ่งเป็นรูปแบบ Prefix พิเศษ:

```
VPNv4 NLRI (Network Layer Reachability Information) Format:

   ┌────────────────────────┬─────────────────────────┐
   │   RD (8 byte)           │   IPv4 Prefix (4 byte)   │   = 12 byte รวม
   │   65000:100              │   10.10.0.0/16            │
   └────────────────────────┴─────────────────────────┘

   ตัวอย่าง VPNv4 Prefix ที่ส่งจริงผ่าน MP-BGP:  65000:100:10.10.0.0/16
```

พร้อมด้วย **Extended Community Attribute** ที่แนบ **Route Target** และ **MPLS Label** (VPN
Label ที่จะอธิบายใน Step 855) ติดไปกับ Prefix นี้ด้วยเสมอ — ทำให้ PE ปลายทางรู้ทั้ง (1) VRF ไหน
ควร Import Route นี้ และ (2) Label อะไรที่ต้องใช้ส่งกลับไปยัง PE ต้นทาง

### 854.2 คอนฟิก MP-iBGP ระหว่าง SP-PE1 ↔ SP-PE2 (VPNv4 Address-Family)

Lab นี้มี PE เพียง 2 ตัว จึงทำ **iBGP แบบ Direct Peering ระหว่าง Loopback** ได้เลย (SP ขนาดใหญ่
จริงจะใช้ **Route Reflector** กลาง เพื่อไม่ต้อง Full-Mesh ระหว่าง PE ทุกตัว — Part 87 จะกล่าวถึง)

**บน SP-PE1:**

```
SP-PE1(config)# router bgp 65000
SP-PE1(config-router)# bgp router-id 100.64.0.11
SP-PE1(config-router)# neighbor 100.64.0.12 remote-as 65000
SP-PE1(config-router)# neighbor 100.64.0.12 description ** MP-iBGP VPNv4 to SP-PE2 **
SP-PE1(config-router)# neighbor 100.64.0.12 update-source Loopback0
!
SP-PE1(config-router)# address-family vpnv4
SP-PE1(config-router-af)# neighbor 100.64.0.12 activate
SP-PE1(config-router-af)# neighbor 100.64.0.12 send-community extended
SP-PE1(config-router-af)# exit-address-family
```

**บน SP-PE2:** (กลับด้าน Neighbor IP)

```
SP-PE2(config)# router bgp 65000
SP-PE2(config-router)# bgp router-id 100.64.0.12
SP-PE2(config-router)# neighbor 100.64.0.11 remote-as 65000
SP-PE2(config-router)# neighbor 100.64.0.11 description ** MP-iBGP VPNv4 to SP-PE1 **
SP-PE2(config-router)# neighbor 100.64.0.11 update-source Loopback0
!
SP-PE2(config-router)# address-family vpnv4
SP-PE2(config-router-af)# neighbor 100.64.0.11 activate
SP-PE2(config-router-af)# neighbor 100.64.0.11 send-community extended
SP-PE2(config-router-af)# exit-address-family
```

| คำสั่ง | ความหมาย |
|---|---|
| `neighbor <lo> remote-as 65000` (เท่ากับ AS ตัวเอง) | เป็น **iBGP** โดยอัตโนมัติ (ตามหลักการ [Part 29](part-029-bgp-fundamentals.md)) — ต่างจาก eBGP PE-CE ใน Step 852 |
| `update-source Loopback0` | ใช้ Loopback0 เป็น Source IP ของ TCP/179 Session — ทำให้ Session ไม่ล่มแม้ Physical Interface หนึ่งเส้นทางล่ม (มี Path สำรองผ่าน IGP) |
| `address-family vpnv4` | เปิด Address-Family พิเศษสำหรับแลก VPNv4 Prefix (แยกจาก `ipv4 unicast` ปกติที่ Neighbor นี้ไม่ได้ใช้เลยเพราะเป็น Core-only Session) |
| `send-community extended` | **บังคับต้องมี** — Route Target เป็น **Extended Community** ถ้าไม่ส่งค่านี้ Neighbor จะไม่เห็น RT เลย และจะ Import Route ผิดหรือไม่ Import เลย |

### 854.3 SP-PE1 ทำอะไรกับ Route ที่ได้จาก WAN-EDGE-1 ก่อนส่งเข้า VPNv4

BGP Route ที่ SP-PE1 เรียนจาก WAN-EDGE-1 ผ่าน eBGP (Step 852, ภายใน `address-family ipv4 vrf
CUST-ENTLAB`) จะถูก **Import เข้า VPNv4 Table โดยอัตโนมัติ** ทันทีที่:

1. VRF `CUST-ENTLAB` มี `rd` กำหนดไว้แล้ว (Step 853.2)
2. VRF มี `route-target export 65000:100` กำหนดไว้แล้ว
3. Neighbor VPNv4 (SP-PE2) ถูก Activate แล้ว (Step 854.2)

**ไม่ต้องมีคำสั่ง `redistribute` เพิ่มเติมใดๆ** — นี่คือกลไก Built-in ของ IOS-XE: ทุก Route ที่
เข้ามาใน BGP Table ของ VRF (ไม่ว่าจะจาก eBGP PE-CE, Static Route ที่ Redistribute เข้า BGP ของ
VRF, หรือ Connected Route) จะถูกแปะ RD + Route Target แล้วส่งเข้า VPNv4 Address-Family อัตโนมัติ
ตราบใดที่ VRF นั้นมี RT Export กำหนดไว้

### 854.4 Verify VPNv4 Route ที่ SP-PE2 ได้รับ

```
SP-PE2# show ip bgp vpnv4 vrf CUST-ENTLAB
BGP table version is 8, local router ID is 100.64.0.12
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 65000:100 (default for vrf CUST-ENTLAB)
*>i 10.10.0.0/16     100.64.0.11              0    100      0 65001 i
```

- `Next Hop 100.64.0.11` = Loopback ของ **SP-PE1** (ไม่ใช่ WAN-EDGE-1!) — เพราะ SP-PE1 ทำหน้าที่
  "แทนตัว" WAN-EDGE-1 ก่อนส่งเข้า MP-BGP (BGP Next-Hop Rewrite ที่ PE ทำโดยอัตโนมัติเสมอ)
- `Path 65001 i` = AS_PATH ยังเก็บ AS ของลูกค้า (65001) ไว้ครบ แม้จะข้าม SP Core (AS 65000) มา
  แล้วก็ตาม — Provider AS **ไม่ปรากฏใน AS_PATH ของ VPNv4** (Transparent ต่อลูกค้า)

---

## Step 855 — Label Stacking: 2 Label ต่อ Packet ใน MPLS L3VPN

### 855.1 ทำไมต้องมี 2 Label

Packet ของลูกค้าที่วิ่งผ่าน MPLS L3VPN ต้องแก้ปัญหา 2 อย่างพร้อมกัน:

| ปัญหา | แก้ด้วย Label ชั้นไหน | ใครเป็นคนดู/แก้ Label นี้ |
|---|---|---|
| **"จะไปถึง PE ปลายทางได้อย่างไร"** (ข้าม Core ที่มีหลาย Hop) | **Label ชั้นนอก (Outer / Transport Label)** | P Router ทุกตัว (Swap ทุก Hop ตาม LDP) |
| **"เมื่อไปถึง PE ปลายทางแล้ว จะรู้ได้อย่างไรว่าเป็นของ VRF ไหน/ส่งต่อไป CE ไหน"** | **Label ชั้นใน (Inner / VPN Label)** | เฉพาะ PE ปลายทางเท่านั้น (P Router "มองไม่เห็น" ชั้นนี้เลย เพราะไม่แกะ Label ชั้นนอกออก) |

```
โครงสร้าง Packet ที่วิ่งอยู่กลาง MPLS Core (ระหว่าง SP-P1 ↔ SP-P2):

   ┌──────────────────┬──────────────────┬────────────────────────────┐
   │ Outer Label       │ Inner (VPN) Label │ Original Customer IP Packet │
   │ (Transport, Swap  │ (คงเดิมตลอดทาง    │  (Src: 172.30.10.x,          │
   │  ทุก Hop โดย LDP)  │  ไม่ถูก Swap เลย) │   Dst: 10.10.10.x)            │
   └──────────────────┴──────────────────┴────────────────────────────┘
        แก้ปัญหา "ไปถึง PE           แก้ปัญหา "PE ปลายทาง
         ปลายทางได้อย่างไร"           รู้ว่าเป็นของ VRF ไหน"
```

### 855.2 Walkthrough เต็มรูปแบบ: Packet จาก REMOTE-SITE-CE ไปยัง WAN-EDGE-1 (HQ, VLAN SALES)

สมมติ PC ที่ REMOTE-SITE-CE (`172.30.10.0/24`) ส่ง Packet ไปยัง Server ที่ VLAN SALES ของ HQ
(`10.10.10.x`) — Ingress PE = **SP-PE2**, Egress PE = **SP-PE1**

```
Hop 0: REMOTE-SITE-CE → SP-PE2 (eBGP link, ไม่มี Label เลย — Packet เป็น Plain IP ปกติ)
       [ Dst: 10.10.10.5 ]

Hop 1: SP-PE2 (Ingress PE) รับ Packet เข้า VRF CUST-ENTLAB, Lookup VPNv4 Table
       → พบ Route 10.10.0.0/16 มี Next-Hop = SP-PE1 (100.64.0.11), VPN Label = 42
       → เติม Label 2 ชั้น:  [Outer=24][Inner=42][ IP Packet ]
       → ส่งออก Gi0/0/1 (Core) มุ่งหน้า SP-P2 (Outer Label 24 เรียนจาก LDP ของ SP-P2)

Hop 2: SP-P2 (P Router) รับ Packet, ดูแค่ Outer Label 24 ใน LFIB
       → Swap 24 → 31 (Label ใหม่ที่เรียนจาก LDP ของ SP-P1 สำหรับ FEC 100.64.0.11/32)
       → [Outer=31][Inner=42][ IP Packet ]  — ไม่แตะ Inner Label เลย, ไม่ดู IP Header เลย
       → ส่งออกมุ่งหน้า SP-P1

Hop 3: SP-P1 (P Router, Penultimate Hop ก่อนถึง Egress PE)
       → SP-PE1 แจก Label แบบ Implicit-Null (PHP) ให้ FEC 100.64.0.11/32 (เพราะเป็นเจ้าของ
         Prefix นี้เอง — Loopback ของตัวเอง)
       → SP-P1 จึง "Pop" Outer Label ทิ้งไปเลย (ไม่ต้อง Swap) แล้วส่งต่อ
       → [Inner=42][ IP Packet ]  — เหลือ Label เดียว (VPN Label) ตอนถึง SP-PE1

Hop 4: SP-PE1 (Egress PE) รับ Packet ที่มี Label เดียว (42)
       → Lookup Label 42 ใน LFIB → รู้ทันทีว่าเป็นของ VRF CUST-ENTLAB, ต้อง Pop Label แล้ว
         Lookup IP ต่อใน VRF Table (ไม่ต้องเดา, ไม่ต้อง Broadcast — Label ชี้ตรงเข้า VRF/Interface)
       → [ IP Packet ]  — Label หลุดหมดแล้ว เหลือ Plain IP
       → Lookup 10.10.10.5 ใน VRF CUST-ENTLAB RIB → Next-Hop 192.0.2.2 (WAN-EDGE-1)
       → ส่งออก Gi0/0/2 (CE-Facing) เป็น Plain IP Packet — WAN-EDGE-1 ไม่เห็น Label เลยแม้แต่ Bit เดียว
```

### 855.3 ข้อสังเกตสำคัญที่ต้องจำ

- **VPN Label (Inner) ไม่ถูก Swap เลยตลอดทาง** — เปลี่ยนแค่ครั้งเดียวตอน Pop ที่ Egress PE
  เท่านั้น เพราะ P Router (SP-P1, SP-P2) ไม่มีความรู้เรื่อง VPN/VRF ใดๆ เลย พวกเขาเห็นแค่ Outer
  Label — นี่คือเหตุผลที่ **P Router Scale ได้มหาศาล** (ไม่ต้องเก็บ VPN Route นับล้าน Prefix
  ของลูกค้าทุกราย เก็บแค่ Loopback ของ PE ไม่กี่ร้อย-พันตัวพอ)
- **PHP (Penultimate Hop Popping)** ลด Load ของ Egress PE — ให้ Router ก่อนหน้าทำ Pop แทน แต่
  ในกรณี MPLS L3VPN Packet ยังเหลือ VPN Label 1 ชั้นอยู่ดี (PHP popป็แค่ Outer เท่านั้น) เพื่อให้
  Egress PE รู้ทันทีว่าเป็น VRF ไหนโดยไม่ต้อง Lookup IP เต็มรูปแบบก่อน

---

## Step 856 — Lab เต็มรูปแบบ End-to-End: Any-to-Any Connectivity

### 856.1 Topology สมบูรณ์ของ Part นี้

```
   WAN-EDGE-1 (CE, HQ)                                              REMOTE-SITE-CE (CE, Branch)
   AS 65001, Lo0=1.1.1.21/32                                        AS 65002, Lo0=172.30.0.1/32
   LAN: 10.10.0.0/16 (Campus)                                       LAN: 172.30.10.0/24 (Lo1)
        │ Gi0/0/2                                                        │ Gi0/0/1
        │ 192.0.2.2/30                                                   │ 192.0.2.6/30
   ─────┴─────                                                      ─────┴─────
        │ .1                                                             │ .5
   ┌────┴─────┐  Gi0/0/1    100.64.1.0/30    Gi0/0/1 ┌────────┐ Gi0/0/2   ┌────────┐ Gi0/0/1  ┌────┴─────┐
   │ SP-PE1    ├──────────────────────────────────────┤ SP-P1   ├───────────┤ SP-P2   ├────────────┤ SP-PE2    │
   │ (PE, LER) │ .1                                .2 │(P, LSR)│ .5     .6 │(P, LSR)│ .9    .10  │ (PE, LER) │
   └───────────┘                                       └────────┘           └────────┘             └───────────┘
   Lo0=100.64.0.11/32           MPLS Core, AS 65000 (SP)          100.64.1.4/30      Lo0=100.64.0.12/32
   VRF CUST-ENTLAB                                                                    VRF CUST-ENTLAB
   RD 65000:100                                                                       RD 65000:100
                                                                    100.64.1.8/30
   ── LDP ทุก Core Link (SP-PE1↔SP-P1, SP-P1↔SP-P2, SP-P2↔SP-PE2) ──
   ── MP-iBGP VPNv4 ตรงระหว่าง SP-PE1 Lo0 ↔ SP-PE2 Lo0 (ผ่าน LSP ที่ LDP สร้างให้) ──
   ── eBGP PE-CE: SP-PE1↔WAN-EDGE-1 (AS 65000↔65001) และ SP-PE2↔REMOTE-SITE-CE (AS 65000↔65002) ──
```

### 856.2 ตาราง IP Addressing สรุปทั้งหมดของ Part นี้

| Device | Interface | IP Address | หมายเหตุ |
|---|---|---|---|
| SP-PE1 | Loopback0 | 100.64.0.11/32 | MPLS Router-ID / MP-iBGP Source |
| SP-PE1 | Gi0/0/1 (Core → SP-P1) | 100.64.1.1/30 | `mpls ip` เปิด, Global Table |
| SP-PE1 | Gi0/0/2 (CE → WAN-EDGE-1) | 192.0.2.1/30 | `vrf forwarding CUST-ENTLAB`, ไม่มี `mpls ip` |
| SP-P1 | Loopback0 | 100.64.0.1/32 | MPLS Router-ID |
| SP-P1 | Gi0/0/1 (→ SP-PE1) | 100.64.1.2/30 | `mpls ip` เปิด |
| SP-P1 | Gi0/0/2 (→ SP-P2) | 100.64.1.5/30 | `mpls ip` เปิด |
| SP-P2 | Loopback0 | 100.64.0.2/32 | MPLS Router-ID |
| SP-P2 | Gi0/0/1 (→ SP-P1) | 100.64.1.6/30 | `mpls ip` เปิด |
| SP-P2 | Gi0/0/2 (→ SP-PE2) | 100.64.1.9/30 | `mpls ip` เปิด |
| SP-PE2 | Loopback0 | 100.64.0.12/32 | MPLS Router-ID / MP-iBGP Source |
| SP-PE2 | Gi0/0/1 (Core → SP-P2) | 100.64.1.10/30 | `mpls ip` เปิด, Global Table |
| SP-PE2 | Gi0/0/2 (CE → REMOTE-SITE-CE) | 192.0.2.5/30 | `vrf forwarding CUST-ENTLAB`, ไม่มี `mpls ip` |
| WAN-EDGE-1 | Loopback0 | 1.1.1.21/32 | ตาม [00-ip-address-plan.md](00-ip-address-plan.md) เดิม |
| WAN-EDGE-1 | Gi0/0/2 (→ SP-PE1) | 192.0.2.2/30 | ตาม Part 19 เดิม |
| REMOTE-SITE-CE | Loopback0 | 172.30.0.1/32 | BGP Router-ID |
| REMOTE-SITE-CE | Loopback1 | 172.30.10.1/24 | จำลอง LAN ของสาขา — Advertise เข้า eBGP |
| REMOTE-SITE-CE | Gi0/0/1 (→ SP-PE2) | 192.0.2.6/30 | eBGP PE-CE |

### 856.3 Running-Config เต็มรูปแบบ — SP-PE1

```
hostname SP-PE1
!
vrf definition CUST-ENTLAB
 description ** MPLS L3VPN Customer: Enterprise Lab (HQ Site via WAN-EDGE-1) **
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface Loopback0
 description ** MPLS Router-ID / MP-BGP Update Source **
 ip address 100.64.0.11 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Core Link to SP-P1 **
 ip address 100.64.1.1 255.255.255.252
 mpls ip
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** CE-Facing Access Link to WAN-EDGE-1 **
 vrf forwarding CUST-ENTLAB
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
router ospf 100
 router-id 100.64.0.11
 network 100.64.0.11 0.0.0.0 area 0
 network 100.64.1.0 0.0.0.3 area 0
!
router bgp 65000
 bgp router-id 100.64.0.11
 neighbor 100.64.0.12 remote-as 65000
 neighbor 100.64.0.12 description ** MP-iBGP VPNv4 to SP-PE2 **
 neighbor 100.64.0.12 update-source Loopback0
 !
 address-family vpnv4
  neighbor 100.64.0.12 activate
  neighbor 100.64.0.12 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUST-ENTLAB
  neighbor 192.0.2.2 remote-as 65001
  neighbor 192.0.2.2 description ** eBGP to Customer CE WAN-EDGE-1 **
  neighbor 192.0.2.2 activate
 exit-address-family
!
end
```

### 856.4 Running-Config เต็มรูปแบบ — SP-P1 (P Router, ไม่มี VRF/BGP เลย)

```
hostname SP-P1
!
mpls label protocol ldp
mpls ldp router-id Loopback0 force
!
interface Loopback0
 ip address 100.64.0.1 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Core Link to SP-PE1 **
 ip address 100.64.1.2 255.255.255.252
 mpls ip
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Core Link to SP-P2 **
 ip address 100.64.1.5 255.255.255.252
 mpls ip
 no shutdown
!
router ospf 100
 router-id 100.64.0.1
 network 100.64.0.1 0.0.0.0 area 0
 network 100.64.1.0 0.0.0.3 area 0
 network 100.64.1.4 0.0.0.3 area 0
!
end
```

> **จุดสำคัญ**: SP-P1/SP-P2 **ไม่มี `router bgp` เลย** — P Router ไม่เกี่ยวข้องกับ VPN Route
> ของลูกค้าแม้แต่นิดเดียว รู้จักแค่ Loopback ของ PE ทุกตัวผ่าน OSPF+LDP เท่านั้น พิสูจน์คำกล่าวใน
> Step 855.3 ว่า P Router Scale ได้เพราะไม่แบก VPN State เลย

### 856.5 Running-Config เต็มรูปแบบ — SP-P2 และ SP-PE2

**SP-P2**: เหมือน SP-P1 ทุกประการ เพียงสลับ IP ตามตาราง 856.2 (`Gi0/0/1 = 100.64.1.6/30` ไปทาง
SP-P1, `Gi0/0/2 = 100.64.1.9/30` ไปทาง SP-PE2, `Loopback0 = 100.64.0.2/32`)

**SP-PE2**: เหมือน SP-PE1 ทุกประการ เพียงเปลี่ยน:

```
hostname SP-PE2
!
vrf definition CUST-ENTLAB
 rd 65000:100
 address-family ipv4
  route-target export 65000:100
  route-target import 65000:100
 exit-address-family
!
interface Loopback0
 ip address 100.64.0.12 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Core Link to SP-P2 **
 ip address 100.64.1.10 255.255.255.252
 mpls ip
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** CE-Facing Access Link to REMOTE-SITE-CE **
 vrf forwarding CUST-ENTLAB
 ip address 192.0.2.5 255.255.255.252
 no shutdown
!
router ospf 100
 router-id 100.64.0.12
 network 100.64.0.12 0.0.0.0 area 0
 network 100.64.1.8 0.0.0.3 area 0
!
router bgp 65000
 bgp router-id 100.64.0.12
 neighbor 100.64.0.11 remote-as 65000
 neighbor 100.64.0.11 update-source Loopback0
 !
 address-family vpnv4
  neighbor 100.64.0.11 activate
  neighbor 100.64.0.11 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUST-ENTLAB
  neighbor 192.0.2.6 remote-as 65002
  neighbor 192.0.2.6 description ** eBGP to Customer CE REMOTE-SITE-CE **
  neighbor 192.0.2.6 activate
 exit-address-family
!
end
```

### 856.6 Running-Config เต็มรูปแบบ — REMOTE-SITE-CE (Site ใหม่ของลูกค้า)

```
hostname REMOTE-SITE-CE
!
interface Loopback0
 description ** BGP Router-ID **
 ip address 172.30.0.1 255.255.255.255
!
interface Loopback1
 description ** Simulated Branch LAN Subnet **
 ip address 172.30.10.1 255.255.255.0
!
interface GigabitEthernet0/0/1
 description ** MPLS L3VPN Access Link to Provider PE (SP-PE2) **
 ip address 192.0.2.6 255.255.255.252
 no shutdown
!
router bgp 65002
 bgp router-id 172.30.0.1
 neighbor 192.0.2.5 remote-as 65000
 neighbor 192.0.2.5 description ** eBGP to Provider PE SP-PE2 (MPLS L3VPN) **
 network 172.30.10.0 mask 255.255.255.0
!
end
```

### 856.7 Verify Any-to-Any Connectivity

```
! บน WAN-EDGE-1 (HQ) — ต้องเห็น Route ของ REMOTE-SITE-CE ผ่าน eBGP
WAN-EDGE-1# show ip route bgp
     172.30.10.0/24 [20/0] via 192.0.2.1, 00:05:12

WAN-EDGE-1# ping 172.30.10.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.30.10.1, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.21
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/7/12 ms

! บน REMOTE-SITE-CE — ต้องเห็น Route ของ HQ กลับไปด้วย (Any-to-Any สมบูรณ์)
REMOTE-SITE-CE# show ip route bgp
     10.10.0.0/16 [20/0] via 192.0.2.5, 00:05:09

REMOTE-SITE-CE# traceroute 10.10.10.5 source Loopback1
Type escape sequence to abort.
Tracing the route to 10.10.10.5

  1 192.0.2.5 4 msec 2 msec 2 msec
  2 10.10.10.5 6 msec 5 msec 4 msec
```

> **สิ่งที่ต้องสังเกต**: `traceroute` จาก REMOTE-SITE-CE **ไม่เห็น Hop ของ SP-P1/SP-P2 เลย**
> แม้ Packet จะวิ่งผ่านจริง 4 Hop ภายใน MPLS Core — เพราะ P Router ไม่ทำการ Decrement TTL แบบ
> IP-Header ปกติเมื่อ Forward ด้วย Label เท่านั้น (Cisco Default ยังคง Copy TTL ของ IP ไปที่ Label
> TTL เพื่อป้องกัน Routing Loop แต่ P Router **ไม่ Generate ICMP TTL Exceeded กลับมาแบบเห็น IP
> Hop เหมือน CE** — SP ส่วนใหญ่ตั้งใจซ่อน Internal Topology จากลูกค้าด้วยเหตุผลด้าน Security ผ่าน
> `no mpls ip propagate-ttl` เพิ่มเติมได้ด้วย) — นี่คือหลักฐานเชิงประจักษ์อีกชิ้นว่า **MPLS Core
> "ล่องหน" จากมุมมองลูกค้าจริง**

---

## Step 857 — Route Target Import/Export เชิงลึก: Hub-and-Spoke VPN

### 857.1 ทวนกลไก RT จาก Step 853-854

Any-to-Any Model (Step 856) ใช้ RT เดียวกันทั้ง Export/Import บนทุก PE — ทำให้ทุก Site เห็น
Route ของทุก Site อื่นเท่าเทียมกัน แต่ Enterprise จำนวนมากไม่ต้องการแบบนี้ — สถานการณ์ที่พบบ่อย
ที่สุดคือ **Hub-and-Spoke**: มี Data Center กลาง (Hub) ที่ทุก Branch (Spoke) ต้องคุยด้วยเสมอ
แต่ **Branch ไม่ควรคุยกันเองโดยตรง** (ต้องผ่าน Hub เท่านั้น เพื่อให้ Hub ทำ Security
Inspection/Firewall ได้ทุก Traffic)

### 857.2 การออกแบบด้วย Asymmetric Route Target (ตัวอย่างประกอบเพื่อสอนแนวคิด)

สมมติ VRF ใหม่ (แนวคิดเพิ่มเติม ไม่ใช่ Physical Device จริงใน Lab หลักของ Part นี้) สำหรับลูกค้า
รายที่มี Data Center กลาง 1 แห่ง (HUB-DC) และ Branch 3 แห่ง (BRANCH-1, BRANCH-2, BRANCH-3)
ใช้ **2 Route Target แยกทิศทาง**:

| Route Target | ความหมาย | ใครใช้ |
|---|---|---|
| **RT 65000:900** (HUB export) | "นี่คือ Route ของ Hub-DC" | HUB-DC PE **Export** เท่านั้น |
| **RT 65000:901** (SPOKE export) | "นี่คือ Route ของ Branch แต่ละแห่ง" | Branch PE ทุกตัว **Export** เท่านั้น |

```
                              HUB-DC (Data Center กลาง)
                            RT Export: 65000:900
                            RT Import: 65000:901   <── รับ Route จาก Branch ทุกแห่ง
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
              BRANCH-1            BRANCH-2            BRANCH-3
         RT Export: 65000:901  RT Export: 65000:901  RT Export: 65000:901
         RT Import: 65000:900  RT Import: 65000:900  RT Import: 65000:900
         (รับเฉพาะ Route ของ HUB-DC เท่านั้น — Import ไม่มี 65000:901 จึงไม่เห็น Branch อื่นเลย)
```

### 857.3 คอนฟิกตัวอย่างบน PE (สมมติชื่อ VRF `CUST-HUBSPOKE`)

**PE ที่ต่อ HUB-DC:**

```
PE-HUB(config)# vrf definition CUST-HUBSPOKE
PE-HUB(config-vrf)# rd 65000:900
PE-HUB(config-vrf)# address-family ipv4
PE-HUB(config-vrf-af)# route-target export 65000:900
PE-HUB(config-vrf-af)# route-target import 65000:901
PE-HUB(config-vrf-af)# exit-address-family
```

**PE ที่ต่อ BRANCH-1/2/3 (คอนฟิกเหมือนกันทุกตัว):**

```
PE-BRANCH(config)# vrf definition CUST-HUBSPOKE
PE-BRANCH(config-vrf)# rd 65000:9<เลข Branch เช่น 01, 02, 03>
PE-BRANCH(config-vrf)# address-family ipv4
PE-BRANCH(config-vrf-af)# route-target export 65000:901
PE-BRANCH(config-vrf-af)# route-target import 65000:900
PE-BRANCH(config-vrf-af)# exit-address-family
```

- **HUB-DC**: Export 900, Import 901 → ส่ง Route ของตัวเอง (Tag 900) และรับ Route ทุก Branch
  (Tag 901) ทั้งหมด — HUB เห็นทุก Site
- **BRANCH ทุกตัว**: Export 901, Import 900 → ส่ง Route ของตัวเอง (Tag 901) แต่ **รับแค่ Route
  ที่มี Tag 900 เท่านั้น** (คือของ HUB-DC เพียงแห่งเดียว) — Branch **ไม่เห็น Route ของ Branch
  อื่นเลย** เพราะไม่มี RT Import ใดตรงกับ Tag 901 ของกันและกัน

### 857.4 ทำไมต้องมี RD ต่างกันทุก Site (ในตัวอย่างนี้)

ต่างจาก Step 856 ที่ใช้ RD เดียว (`65000:100`) ทุก PE เพราะ Any-to-Any ไม่จำเป็นต้องแยก Path —
แต่ **Hub-and-Spoke ที่มี Branch หลายแห่งประกาศ Prefix เดียวกัน (เช่น Default Route จาก HUB)
เข้ามาซ้ำกัน จำเป็นต้องมี RD ต่างกันต่อ Site** เพื่อให้ BGP มองว่าเป็น Prefix คนละตัว (ไม่ถูก
มองว่าเป็น Duplicate แล้วเก็บ Best Path ไว้ Path เดียว) — นี่คือเหตุผลเชิงเทคนิคที่แท้จริงว่า
"ทำไม RD ควร Unique ต่อ PE-VRF แม้จะเป็น VPN เดียวกัน" ซึ่งเป็น Best Practice ที่ Part 87 จะเจาะลึก

### 857.5 เปรียบเทียบ Any-to-Any กับ Hub-and-Spoke

| คุณสมบัติ | Any-to-Any (Step 856) | Hub-and-Spoke (Step นี้) |
|---|---|---|
| RT Export/Import | ค่าเดียวกันทุก Site | แยก 2 ค่า (Hub-tag / Spoke-tag) |
| Site คุยกันเองได้หรือไม่ | ได้ทุก Site ↔ ทุก Site | Spoke ↔ Spoke **ไม่ได้** (ต้องผ่าน Hub) |
| ใช้เมื่อไหร่ | Site เท่าเทียมกัน ไม่มี Central Control | มี Data Center/Firewall กลางที่ต้อง Inspect Traffic ทุกเส้นทาง |
| จำนวน RT ที่ต้องจัดการ | 1 ค่า | อย่างน้อย 2 ค่า (ซับซ้อนขึ้นตามจำนวน Tier) |

---

## Step 858 — MPLS L3VPN QoS: CoS Tier ของ SP และการ Map DSCP ของลูกค้า

### 858.1 ปัญหา: SP Core เป็น Shared Infrastructure

MPLS Core ของ SP ถูกใช้ร่วมกันโดยลูกค้าหลายราย (Multi-Tenant) — ถ้าไม่มี QoS ลูกค้ารายหนึ่งที่ส่ง
Traffic จำนวนมาก (เช่น Backup ข้าม Site) อาจไป Congest Link จนกระทบ Traffic Real-time (Voice/
Video) ของลูกค้ารายอื่นได้ SP ส่วนใหญ่จึงขาย **CoS Tier (Class of Service)** เป็นส่วนหนึ่งของ
บริการ MPLS L3VPN โดยลูกค้าเลือกได้ว่า Traffic ประเภทไหนควรได้ Priority สูงสุด

### 858.2 CoS Tier ตัวอย่างที่ SP มักเสนอขาย

| Tier | ชื่อ | ใช้กับ Traffic ประเภทไหน | % Bandwidth ที่รับประกัน (ตัวอย่าง) |
|---|---|---|---|
| **Premium** | Real-Time | Voice (VoIP), Video Conference | 10-20% (Priority Queue, Low Latency) |
| **Business Critical** | Assured Forwarding สูง | ERP, Database Replication, Video Streaming | 30-40% |
| **Standard** | Assured Forwarding ปานกลาง | Email, File Transfer ภายในองค์กร | 30% |
| **Best-Effort** | Default | Internet Browsing ทั่วไป, Traffic ที่ไม่ได้ Mark อะไรเลย | เหลือเท่าไหร่ใช้เท่านั้น |

### 858.3 EXP Bit — "DSCP เวอร์ชัน MPLS"

MPLS Header มี Field ชื่อ **EXP (Experimental — ปัจจุบันเรียกว่า Traffic Class)** ขนาด 3 Bit
(รองรับค่า 0-7) ทำหน้าที่เหมือน DSCP/CoS แต่ **อยู่ใน MPLS Label Header** (ไม่ใช่ IP Header) —
เพราะ P Router **ไม่ดู IP Header เลย** (ตาม Step 855) จึงต้องมีที่เก็บ QoS Marking แยกไว้ใน
Label Header เอง เพื่อให้ P Router ทำ Queueing/Scheduling ได้โดยไม่ต้องแกะ Label ออกดู IP ข้างใน

```
MPLS Label Header (4 byte):

   ┌─────────────────────┬─────┬───┬─────────────────┐
   │  Label (20 bit)       │ EXP │ S │  TTL (8 bit)     │
   │                        │(3bit)│(1)│                  │
   └─────────────────────┴─────┴───┴─────────────────┘
                              │
                              └──► ใช้เก็บ QoS Marking (เทียบเท่า DSCP ของ IP Header)
                                   S = Bottom of Stack bit (1 = เป็น Label ชั้นล่างสุดแล้ว)
```

### 858.4 การ Map DSCP ของลูกค้า → EXP ของ SP (ที่ Ingress PE)

[Part 24 Step 234](part-024-qos-fundamentals.md) และ [Part 40](part-040-advanced-qos.md) สอน
เรื่อง DSCP Marking ที่ Trust Boundary ของ Enterprise ไปแล้ว (และ Part 82 ต่อยอดเรื่อง QoS ที่
WAN Edge) — เมื่อ Traffic ของลูกค้าที่ Mark DSCP มาแล้วเดินทางเข้าสู่ MPLS Core ของ SP, **SP-PE1
(Ingress PE) ต้อง "แปล" DSCP ของลูกค้าเป็น EXP ของตัวเอง** ตอน Impose Label — ตารางการ Map
มาตรฐานที่ใช้กันทั่วไป:

| DSCP ของลูกค้า (จาก Part 24/40) | Traffic ประเภท | EXP ที่ SP Map ให้ | CoS Tier ของ SP |
|---|---|---|---|
| EF (46) | Voice | 5 | Premium |
| AF41/AF42/AF43 (34/36/38) | Video Conference | 4 | Premium |
| AF31/AF32/AF33 (26/28/30) | Business Critical App | 3 | Business Critical |
| AF21/AF22/AF23 (18/20/22) | Business Data ทั่วไป | 2 | Standard |
| AF11/AF12/AF13 (10/12/14) | Bulk/Backup | 1 | Standard (Low) |
| Default (0) | Best-Effort | 0 | Best-Effort |
| CS6/CS7 (48/56) | Network Control (Routing Protocol) | 6/7 | สำรองสำหรับ Control Plane ของ SP เอง |

### 858.5 คอนฟิกตัวอย่างบน SP-PE1 (Ingress PE) — MQC Marking EXP ตาม DSCP ขาเข้า

```
SP-PE1(config)# class-map match-any SP-PREMIUM
SP-PE1(config-cmap)# match dscp ef
SP-PE1(config-cmap)# match dscp af41 af42 af43
SP-PE1(config)# class-map match-any SP-BUSINESS
SP-PE1(config-cmap)# match dscp af31 af32 af33
SP-PE1(config)# class-map match-any SP-STANDARD
SP-PE1(config-cmap)# match dscp af21 af22 af23 af11 af12 af13
!
SP-PE1(config)# policy-map SP-MPLS-EXP-MARKING
SP-PE1(config-pmap)# class SP-PREMIUM
SP-PE1(config-pmap-c)# set mpls experimental topmost 5
SP-PE1(config-pmap)# class SP-BUSINESS
SP-PE1(config-pmap-c)# set mpls experimental topmost 3
SP-PE1(config-pmap)# class SP-STANDARD
SP-PE1(config-pmap-c)# set mpls experimental topmost 2
SP-PE1(config-pmap)# class class-default
SP-PE1(config-pmap-c)# set mpls experimental topmost 0
!
SP-PE1(config)# interface GigabitEthernet0/0/1
SP-PE1(config-if)# service-policy output SP-MPLS-EXP-MARKING
```

- `set mpls experimental topmost <value>` = Mark EXP Bit ของ **Label ชั้นบนสุด (Topmost —
  คือ Outer/Transport Label)** เท่านั้น เพราะ P Router ดูแค่ Label บนสุดในการ Queueing เสมอ
  (สอดคล้องกับ Step 855.1 ที่บอกว่า P Router ไม่แกะ Inner Label ออกดูเลย)
- Policy นี้ Apply เป็น **`service-policy output`** บน Interface ฝั่ง Core (ตอน Impose Label
  พอดี — จุดเดียวที่ทำให้ Mark EXP ได้ถูกจังหวะ)
- P Router (SP-P1, SP-P2) ใช้ Policy แยกอีกชุด (`match mpls experimental topmost <value>`) เพื่อ
  จัดคิว (Queueing) ตาม EXP ที่ Mark มาแล้ว — ไม่ต้อง Classify ใหม่จาก IP Header เลย (Core ไม่ควร
  แกะ IP Header ด้วยเหตุผลด้าน Performance)

---

## Step 859 — Verification/Troubleshooting คำสั่งสำคัญของ MPLS L3VPN

### 859.1 ตรวจ LDP Neighbor (ทวนจาก Step 851 พร้อมมุม Troubleshoot)

```
SP-P1# show mpls ldp neighbor
    Peer LDP Ident: 100.64.0.11:0; Local LDP Ident 100.64.0.1:0
        TCP connection: 100.64.0.11.646 - 100.64.0.1.52011
        State: Oper; Msgs sent/rcvd: 22/23; Downstream
        Up time: 00:20:41
    Peer LDP Ident: 100.64.0.2:0; Local LDP Ident 100.64.0.1:0
        TCP connection: 100.64.0.2.646 - 100.64.0.1.51987
        State: Oper; Msgs sent/rcvd: 20/21; Downstream
```

> **ถ้าไม่เห็น Neighbor เลย**: เช็ค (1) `mpls ip` ลืมสั่งบน Interface หรือไม่ (2) IGP
> (`show ip ospf neighbor`) ขึ้นหรือไม่ — LDP ต้องพึ่ง IGP เสมอ ถ้า OSPF ไม่ขึ้น LDP ก็ไม่มีทาง
> Discover กันได้เลยแม้ Physical Link จะ up

### 859.2 ตรวจ MPLS Forwarding Table (LFIB)

```
SP-PE1# show mpls forwarding-table
Local  Outgoing   Prefix              Bytes Label   Outgoing   Next Hop
Label  Label      or Tunnel Id        Switched      interface
16     Pop Label  100.64.1.4/30       0             Gi0/0/1    100.64.1.2
17     18         100.64.0.12/32      15420         Gi0/0/1    100.64.1.2
18     Pop Label  100.64.0.1/32       0             Gi0/0/1    100.64.1.2
19     Aggregate  192.0.2.0/30[V]     0
```

- แถวสุดท้าย (`[V]` = VRF) แสดง Prefix ของ VRF `CUST-ENTLAB` — ไม่มี Outgoing Label เพราะเป็น
  Local Interface ของ VRF เอง (ไม่ต้องส่งผ่าน MPLS Core)

### 859.3 ตรวจ BGP VPNv4 Table

```
SP-PE1# show ip bgp vpnv4 all summary
BGP router identifier 100.64.0.11, local AS number 65000
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
100.64.0.12     4 65000      40      41        6    0    0 00:18:02        1

SP-PE1# show ip bgp vpnv4 vrf CUST-ENTLAB labels
   Network          Next Hop      In label/Out label
Route Distinguisher: 65000:100 (CUST-ENTLAB)
   10.10.0.0/16     0.0.0.0       42/nolabel
   172.30.10.0/24   100.64.0.12   nolabel/42
```

- `In label 42` = Label ที่ SP-PE1 **แจกให้คนอื่นใช้** ส่ง Traffic เข้ามาหา `10.10.0.0/16`
  (ของ WAN-EDGE-1) — ตรงกับตัวเลขที่ใช้ใน Walkthrough Step 855.2
- `Out label 42` (แถวที่สอง) = Label ที่ SP-PE1 **ต้องใช้เอง** ตอนส่ง Traffic ไปหา
  `172.30.10.0/24` (ของ REMOTE-SITE-CE ผ่าน SP-PE2)

### 859.4 ตรวจ Route ภายใน VRF (มุมมอง PE)

```
SP-PE1# show ip route vrf CUST-ENTLAB
Routing Table: CUST-ENTLAB
Codes: ... (ตัด Legend มาตรฐานออก)

B    172.30.10.0/24 [200/0] via 100.64.0.12, 00:22:10
C    192.0.2.0/30 is directly connected, GigabitEthernet0/0/2
B    10.10.0.0/16 [20/0] via 192.0.2.2, 00:22:15
```

- `B ... via 100.64.0.12` (AD 200 = iBGP) = Route ที่มาจาก MP-iBGP VPNv4 (ข้าม MPLS Core มา)
- `B ... via 192.0.2.2` (AD 20 = eBGP) = Route ที่มาจาก CE ตรงๆ (WAN-EDGE-1)
- **ไม่มี Route ของ VRF อื่น (`CUST-ACME` ตัวอย่าง Step 853.4) ปรากฏในตารางนี้เลย** — พิสูจน์
  Isolation สมบูรณ์

### 859.5 ตรวจสอบ TTL/Traceroute เพื่อดู Label Switching Path จริง

```
SP-PE1# traceroute mpls ipv4 10.10.0.0/16 192.0.2.2

Type escape sequence to abort.
Tracing MPLS Label Switched Path to 10.10.0.0/16, timeout is 2 seconds

  0 100.64.1.1 MRU 1500 [Labels: implicit-null Exp: 0]
L  1 100.64.1.2 MRU 1500 [Labels: implicit-null Exp: 0] 4 ms
```

- คำสั่งชุด `traceroute mpls` เป็นเครื่องมือเฉพาะสำหรับ Debug LSP โดยตรง (ต่างจาก `traceroute`
  ปกติที่ใช้ ICMP) — ใช้ตรวจว่า Label Switched Path เดินทางถูก Hop ตามที่ IGP คำนวณไว้จริงหรือไม่

### 859.6 สรุปคำสั่ง Troubleshoot ที่ต้องจำ

| คำสั่ง | ใช้ตรวจอะไร |
|---|---|
| `show mpls ldp neighbor` | LDP Session ขึ้นหรือไม่ (Control Plane ของ Label) |
| `show mpls interfaces` | Interface ไหนเปิด MPLS/LDP อยู่บ้าง |
| `show mpls forwarding-table` | LFIB — Label Mapping จริงที่ใช้ Forward Packet |
| `show ip bgp vpnv4 all summary` | MP-iBGP VPNv4 Session ระหว่าง PE ขึ้นหรือไม่ |
| `show ip bgp vpnv4 vrf <name> labels` | VPN Label (Inner) ของแต่ละ Prefix |
| `show ip route vrf <name>` | Routing Table ของ VRF นั้นๆ (มุมมอง Native IP ไม่มี Label) |
| `show ip vrf` | รายชื่อ VRF ทั้งหมดที่มีบน Router พร้อม RD |
| `traceroute mpls ipv4 <prefix> <mask>` | ตรวจ LSP จริงทีละ Hop |

---

## Step 860 — Lab เต็มรูปแบบสุดท้าย: Full MPLS L3VPN Deployment

### 860.1 ภาพรวม Checklist การ Deploy MPLS L3VPN ที่ครบทุกองค์ประกอบ

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  1) IGP (OSPF 100) ทุกตัวใน SP Core (SP-P1/P2, SP-PE1/2)         │  <- Step 851.5
   │  2) mpls ip + LDP ทุก Core Interface                              │  <- Step 851.6
   │  3) VRF Definition + RD + RT บน PE ทุกตัว                        │  <- Step 853
   │  4) MP-iBGP VPNv4 ระหว่าง PE ทุกตัว (update-source Loopback0)     │  <- Step 854
   │  5) eBGP PE-CE ภายใน Address-Family ของ VRF                       │  <- Step 852
   │  6) (ถ้ามี) QoS EXP Marking ที่ Ingress PE                        │  <- Step 858
   │  7) Verify: LDP Neighbor → LFIB → BGP VPNv4 → Route ใน VRF → Ping │  <- Step 859
   └─────────────────────────────────────────────────────────────────┘
```

รายการ Config เต็มรูปแบบของทุกอุปกรณ์ (SP-PE1, SP-P1, SP-P2, SP-PE2, WAN-EDGE-1,
REMOTE-SITE-CE) ถูกให้ไว้ครบแล้วใน **Step 856.3–856.6** — Step นี้จะรวบรวมผลการ Verify ทั้งหมด
เป็น Checklist เดียวเพื่อพิสูจน์ว่า Lab สมบูรณ์ 100%

### 860.2 Verification Checklist แบบครบวงจร

**(1) IGP + LDP ของ SP Core ขึ้นครบทุก Link:**

```
SP-PE1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
100.64.0.1        1   FULL/  -        00:00:38    100.64.1.2      GigabitEthernet0/0/1

SP-PE1# show mpls ldp neighbor | include Peer
    Peer LDP Ident: 100.64.0.1:0; Local LDP Ident 100.64.0.11:0
```

**(2) MP-iBGP VPNv4 Session ระหว่าง SP-PE1 ↔ SP-PE2 ขึ้น:**

```
SP-PE1# show ip bgp vpnv4 all summary | include 100.64.0.12
100.64.0.12     4 65000      55      54        9    0    0 00:25:40        1
```

**(3) eBGP PE-CE ทั้งสองฝั่งขึ้น:**

```
SP-PE1# show ip bgp vpnv4 vrf CUST-ENTLAB summary | include 192.0.2.2
192.0.2.2       4 65001      30      29        9    0    0 00:15:03        1

SP-PE2# show ip bgp vpnv4 vrf CUST-ENTLAB summary | include 192.0.2.6
192.0.2.6       4 65002      28      27        9    0    0 00:14:55        1
```

**(4) VRF Route ครบทั้งสองฝั่ง (Any-to-Any สมบูรณ์):**

```
SP-PE1# show ip route vrf CUST-ENTLAB bgp
B    172.30.10.0/24 [200/0] via 100.64.0.12, 00:20:11

SP-PE2# show ip route vrf CUST-ENTLAB bgp
B    10.10.0.0/16 [200/0] via 100.64.0.11, 00:20:14
```

**(5) End-to-End Reachability ระหว่าง CE ทั้งสอง Site:**

```
WAN-EDGE-1# ping 172.30.10.1 source 1.1.1.21 repeat 20
Success rate is 100 percent (20/20), round-trip min/avg/max = 5/8/14 ms

REMOTE-SITE-CE# ping 10.10.10.1 source 172.30.10.1 repeat 20
Success rate is 100 percent (20/20), round-trip min/avg/max = 5/8/13 ms
```

**(6) ยืนยันว่า Label 2 ชั้นถูกใช้จริงตอน Forward (ตรวจที่ SP-PE2, Ingress สำหรับทิศทางนี้):**

```
SP-PE2# show ip bgp vpnv4 vrf CUST-ENTLAB labels | include 10.10.0.0
   10.10.0.0/16     100.64.0.11   nolabel/42
SP-PE2# show mpls forwarding-table labels 24
Local  Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label  Label      or Tunnel Id     Switched      interface
24     17         100.64.0.11/32   28840         Gi0/0/1    100.64.1.9
```

> ผลลัพธ์นี้ยืนยันครบทั้ง 2 ชั้น Label ที่อธิบายใน Step 855: **Outer Label 24 → 17** (Transport,
> Swap ตาม LDP) และ **Inner/VPN Label 42** (คงที่ ไม่ถูก Swap จนกว่าจะถึง SP-PE1) — สอดคล้องกับ
> Walkthrough ที่อธิบายไว้ทุกประการ

### 860.3 สรุปผลลัพธ์ของ Lab

Lab นี้พิสูจน์ครบทุกแนวคิดหลักของ MPLS L3VPN ใน 1 Topology เดียว:

| แนวคิด | หลักฐานที่พิสูจน์ใน Lab |
|---|---|
| CE ไม่รู้จัก MPLS เลย | WAN-EDGE-1/REMOTE-SITE-CE ไม่มี `mpls ip` แม้แต่บรรทัดเดียว |
| VRF แยก Multi-Tenant บน PE | `CUST-ENTLAB` แยกจาก Global Table ของ SP-PE1/PE2 สมบูรณ์ |
| MP-BGP VPNv4 แจก Route ข้าม PE | SP-PE2 เห็น `10.10.0.0/16` ผ่าน MP-iBGP โดยไม่ต้องมี eBGP ตรงกับ WAN-EDGE-1 |
| Label Stacking 2 ชั้น | `show mpls forwarding-table` + `show ip bgp vpnv4 ... labels` ยืนยันตรงกับ Walkthrough |
| Any-to-Any Connectivity | Ping/Traceroute สำเร็จทั้งสองทิศทางระหว่าง HQ ↔ Branch โดยไม่ต้องคอนฟิก Site-to-Site เอง |

---

## แบบฝึกหัดทวนความเข้าใจ Part 86

1. เพราะเหตุใด LDP จึงต้องพึ่งพา IGP เสมอ และถ้า IGP ไม่ Converge จะเกิดผลอย่างไรกับ LDP?
2. อธิบายความแตกต่างระหว่าง **Outer (Transport) Label** กับ **Inner (VPN) Label** — ใครเป็นคน
   Swap/Pop Label แต่ละชั้น?
3. RD (Route Distinguisher) แก้ปัญหาอะไร และทำไม Route-Target ถึงเป็นคนละแนวคิดกับ RD?
4. ในสถานการณ์ Hub-and-Spoke VPN (Step 857) ถ้า Spoke ตั้ง `route-target export` ผิดเป็นค่า
   เดียวกับ Spoke อื่น (ไม่ใช่ Import ของ Hub) จะเกิดผลอย่างไร?
5. เพราะเหตุใด P Router (SP-P1, SP-P2) ไม่มีคำสั่ง `router bgp` เลยในคอนฟิกทั้ง Part นี้?

**เฉลย:**

1. LDP ไม่คำนวณ Path เอง แต่ "ยืม" ผลลัพธ์จาก IGP Routing Table มาผูก Label ให้แต่ละ Prefix
   (FEC) — ถ้า IGP ไม่ Converge (Neighbor ไม่ขึ้น หรือ Route หาย) LDP จะไม่มี Prefix ให้ผูก Label
   เลย ส่งผลให้ LFIB ไม่สมบูรณ์และ Label Switched Path ขาดตอนทันที แม้ LDP Session เองจะ "Oper"
   อยู่ก็ตาม
2. Outer (Transport) Label ใช้พา Packet ข้าม Hop ต่างๆ ของ MPLS Core ไปให้ถึง PE ปลายทาง —
   P Router ทุกตัวเป็นคน Swap Label ชั้นนี้ตาม LDP (และ Router ก่อนถึง Egress PE มักทำ PHP คือ
   Pop ทิ้งไปเลย) ส่วน Inner (VPN) Label ใช้บอกว่า Packet เป็นของ VRF ไหน — มีแค่ PE ต้นทาง
   (Impose) และ PE ปลายทาง (Pop) เท่านั้นที่แตะ Label ชั้นนี้ ไม่ถูก Swap โดย P Router เลยตลอดทาง
3. RD ทำให้ Prefix ที่อาจซ้ำกันของลูกค้าหลายราย (Overlapping IP Address) กลายเป็น VPNv4 Prefix
   ที่ไม่ซ้ำกันในตาราง MP-BGP ของ SP เสมอ (แก้ปัญหา **Uniqueness**) ส่วน Route-Target ควบคุมว่า
   Route ที่ไม่ซ้ำกันแล้วนั้นควรถูก **Import เข้า VRF ไหนบ้าง** (แก้ปัญหา **Reachability/Policy**)
   — RD ทำให้ Route "ไม่ชนกัน" RT ทำให้ Route "ไปถึงคนที่ควรเห็น"
4. Spoke นั้นจะ Export Route ของตัวเองด้วย Tag ที่ Spoke อื่น "บังเอิญ" Import อยู่ (ถ้า Import
   ของ Spoke อื่นตรงกับ Tag ที่ตั้งผิดนี้) ทำให้เกิด **Route Leak ที่ไม่ได้ตั้งใจ**: Spoke สองแห่ง
   อาจเห็น Route ของกันและกันโดยตรง ทำลาย Security Model ของ Hub-and-Spoke ที่ต้องการให้ Traffic
   ผ่าน Hub เท่านั้น — นี่คือเหตุผลที่ต้องตรวจสอบ RT Export/Import ให้ตรงตาม Design เสมอ
5. เพราะ P Router ทำหน้าที่แค่ Label Switching (Data Plane) ตาม LFIB ที่ LDP+IGP สร้างให้เท่านั้น
   — ไม่มีส่วนเกี่ยวข้องกับ VPN Route ของลูกค้าเลย (VPN Route ถูกแลกกันเฉพาะระหว่าง PE ผ่าน
   MP-iBGP VPNv4) การไม่ให้ P Router รู้จัก BGP/VPN เลยคือสิ่งที่ทำให้ SP Scale Backbone ได้ถึง
   ระดับหลายพัน VPN โดยที่ P Router ไม่ต้องแบก State ของลูกค้าแม้แต่รายเดียว

---

## สรุป Part 86

Part นี้เติมเต็มสัญญาที่ [Part 19](part-019-wan-technologies.md) ค้างไว้ตั้งแต่ Step 186-187:
เราสร้าง **SP Core จำลองที่สมบูรณ์** (SP-P1/SP-P2 เป็น P Router, SP-PE1/SP-PE2 เป็น PE Router)
และเดินผ่านทุกองค์ประกอบของ MPLS L3VPN อย่างครบวงจร — **LDP** สร้าง Label Switched Path ข้าม
Core, **VRF** แยก Routing Table ของลูกค้าบน PE แบบ Multi-Tenant จริง, **MP-BGP VPNv4** แจก VPN
Route ข้าม PE ด้วย RD/RT, **Label Stacking 2 ชั้น** ทำให้ P Router Forward ได้โดยไม่ต้องรู้จัก
VPN เลย และ Lab เต็มรูปแบบพิสูจน์ **Any-to-Any Connectivity** ระหว่าง WAN-EDGE-1 (HQ) กับ
REMOTE-SITE-CE (Branch) สำเร็จ — ทั้งหมดนี้เกิดขึ้นโดยที่ CE ทั้งสองฝั่ง **ไม่ต้องรู้จัก MPLS
แม้แต่คำสั่งเดียว** ตามหลักการที่ Step 852 พิสูจน์ไว้

✅ **พร้อมสำหรับ Part 87**: เราจะยกระดับ MPLS L3VPN ที่สร้างไว้ใน Part นี้ขึ้นสู่ระดับ Production
จริง — **Route Reflector** (แทน Full-Mesh iBGP เมื่อมี PE จำนวนมาก), **Inter-AS/Inter-Provider
VPN** (Option A/B/C เมื่อ VPN ต้องข้าม SP หลายราย), **MPLS Traffic Engineering (RSVP-TE)**, และ
การ **Integrate MPLS L3VPN เข้ากับ SD-WAN** ([Part 39](part-039-sd-wan-fundamentals.md)/
[Part 76](part-076-sdwan-advanced-design.md)) เป็น Underlay Transport แบบผสม (Hybrid WAN)

**ไปต่อ:** [Part 87 — MPLS L3VPN Advanced & WAN Integration →](part-087-mpls-l3vpn-advanced-wan-integration.md)
