# Part 53 — DMVPN Advanced (Phase 3, Dual-Hub)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 521–530 จาก 1000**

> ต่อจาก [Part 52 — DMVPN Fundamentals (Phase 1/2)](part-052-dmvpn-fundamentals.md) ที่เราสร้าง
> **DMVPN Cloud** ตัวแรกของหลักสูตรสำเร็จแล้ว — **WAN-EDGE-1** เป็น Hub (NHRP Server), **BRANCH-RTR**
> และ **BRANCH2-RTR** เป็น Spoke ทั้งสอง ใช้ mGRE Tunnel0 ร่วมกัน, **EIGRP AS 200** เป็น Routing
> Protocol ผ่าน Tunnel, และป้องกัน Traffic ด้วย **IPsec Profile (`DMVPN-PROFILE`)** ที่ผูกกับ
> Tunnel0 โดยตรง (`tunnel protection ipsec profile`) — เราได้เห็นว่า Phase 2 DMVPN ทำให้ Spoke
> คุยกันตรง (Spoke-to-Spoke) ได้โดยไม่ต้องผ่าน Hub แต่ต้องแลกกับข้อจำกัดเรื่อง Routing Protocol
> (ต้องปิด Split-Horizon บน EIGRP, ต้องคง Next-Hop เดิมไว้เสมอ) Part นี้จะยกระดับ Lab เดิมไปสู่
> **DMVPN Phase 3** (NHRP Redirect/Shortcut ที่ทำงานได้กับ Routing Protocol ทุกชนิดรวม OSPF) และ
> ขยายเป็น **Dual-Hub Design** ด้วย **WAN-EDGE-2** เป็น Hub สำรอง เพื่อให้ Lab สมบูรณ์ระดับที่
> องค์กรจริงใช้งาน Production — ปิดท้ายด้วยการผสาน **FlexVPN**, **Hierarchical QoS ต่อ Spoke**,
> และ Lab เต็มรูปแบบที่พิสูจน์ทั้ง Spoke-to-Spoke Shortcut และ Hub Failover

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 521 | DMVPN Phase 3 คืออะไร — NHRP Redirect/Shortcut แก้ข้อจำกัดของ Phase 2 |
| 522 | Config Phase 3 — `ip nhrp redirect`/`ip nhrp shortcut` และเปลี่ยนจาก EIGRP เป็น OSPF |
| 523 | Summarization ที่ Hub สำหรับ Phase 3 — ออกแบบ Default/Summary Route ให้ Spoke |
| 524 | Dual-Hub DMVPN Design — เพิ่ม WAN-EDGE-2 เป็น Hub สำรอง, Spoke มี Dual NHS |
| 525 | Dual-Hub Dual-Cloud vs Dual-Hub Single-Cloud — เปรียบเทียบและ Trade-off |
| 526 | ออกแบบ Routing Protocol ให้ Preferred Hub — OSPF Cost / EIGRP Metric Tuning |
| 527 | DMVPN กับ FlexVPN — เมื่อไหร่ควรเลือก FlexVPN Spoke-Hub แทน Classic DMVPN |
| 528 | QoS Integration บน DMVPN — Hierarchical QoS ต่อ Spoke ด้วย NHRP Group |
| 529 | Verification/Troubleshooting เฉพาะ Phase 3 — Shortcut, Redirect, Dual-Hub Failover |
| 530 | Lab เต็มรูปแบบ: Dual-Hub Phase 3 + OSPF Cost + Per-Spoke QoS + Failover Test |

### ทวนโครงสร้าง Lab จาก Part 52 (จุดตั้งต้นของ Part นี้)

| รายการ | ค่าที่ใช้ (สืบทอดจาก Part 52) |
|---|---|
| Hub (Phase 2) | WAN-EDGE-1 — Tunnel0 mGRE, NHRP Server |
| Spoke | BRANCH-RTR, BRANCH2-RTR — Tunnel0 mGRE, NHRP Client |
| DMVPN Tunnel Subnet | `172.16.100.0/24` (NBMA-Independent, ไม่เกี่ยวกับ Internet จริง) |
| NHRP Network-ID / Tunnel Key | `100` |
| IPsec Profile | `DMVPN-PROFILE` (IKEv2 + `esp-aes 256 esp-sha256-hmac`) |
| Routing Protocol (Phase 2) | EIGRP AS 200 บน Tunnel0 (`no ip split-horizon eigrp 200` ที่ Hub) |
| Spoke LAN | BRANCH-RTR = `10.20.1.0/24`, BRANCH2-RTR = `10.20.2.0/24` |
| WAN (NBMA) Underlay | WAN-EDGE-1 = `203.0.113.2` (ISP-RTR), BRANCH-RTR/BRANCH2-RTR = `198.51.100.2/6` |

> ที่อยู่ IP ทั้งหมดข้างต้นเป็นค่าที่สืบทอดมาจาก Lab ของ Part 52 — Part นี้จะ**แก้ไขบางส่วน**
> (เปลี่ยน Routing Protocol, เพิ่ม Hub ตัวที่สอง) โดยจะระบุ diff ให้ชัดเจนทุกจุด

---

## Step 521 — DMVPN Phase 3 คืออะไร

### ทบทวนข้อจำกัดของ Phase 2 (จาก Part 52)

Phase 2 DMVPN ทำให้ Spoke สร้าง Tunnel ตรงถึงกันได้ (Spoke-to-Spoke) แต่ Spoke จะ**รู้ได้ก็ต่อเมื่อ
Routing Protocol บอกเส้นทางที่ถูกต้องมาก่อน** — กลไกคือ Spoke เห็น Next-Hop ของ Prefix ปลายทาง
เป็น NBMA/Tunnel IP ของ **Spoke อีกตัว** (ไม่ใช่ Hub) ในตารางเส้นทาง แล้ว NHRP จึงค่อย Resolve
NBMA Address จริงและสร้าง Tunnel ตรงตามหลัง ปัญหาคือ Routing Protocol ต้อง**คงค่า Next-Hop เดิม
ของผู้ส่งไว้** (ไม่ทำ Next-Hop-Self) ซึ่งนำไปสู่ข้อจำกัดเชิง Routing Protocol ที่ยุ่งยาก:

| Routing Protocol | ปัญหาเมื่อใช้กับ DMVPN Phase 2 |
|---|---|
| EIGRP | ต้องปิด **Split-Horizon** บน Tunnel ที่ Hub (`no ip split-horizon eigrp <AS>`) เพราะ Default จะไม่ Advertise Prefix กลับออก Interface เดียวกันที่เรียนมา — ทำให้ Spoke คุยกันไม่ได้ถ้าไม่ปิด |
| OSPF | Network Type แบบ Broadcast/NBMA เลือก DR/BDR ซึ่งทำให้ Next-Hop ของทุก Prefix ถูกบิดเป็น Hub เสมอ (ผ่าน DR) — ทำให้ Phase 2 Spoke-to-Spoke **ไม่ทำงานกับ OSPF ได้เต็มรูปแบบ** โดยไม่ทำ Hack เพิ่ม |
| BGP | Next-Hop-Self เป็น Default Behavior ของ iBGP อยู่แล้ว — ต้องปิดด้วย `neighbor next-hop-unchanged` ซึ่งซับซ้อนและเสี่ยง Routing Loop |

สรุปคือ **Phase 2 ผูก Feature "Spoke-to-Spoke" เข้ากับพฤติกรรมเฉพาะของ Routing Protocol แต่ละตัว**
ทำให้ Design ยุ่งยากและเปราะบาง (แก้ Routing Protocol ทีก็กระทบ Spoke-to-Spoke ที)

### แนวคิดของ Phase 3 — แยก "Routing" ออกจาก "Forwarding Path Optimization"

**DMVPN Phase 3** แก้ปัญหานี้โดยแยกสองเรื่องออกจากกันอย่างสิ้นเชิง:

1. **Routing Protocol** ทำหน้าที่แค่บอกว่า "ปลายทางนี้ไปได้ผ่าน Hub" เท่านั้น — Hub ทำ
   **Next-Hop-Self ได้ตามปกติ** ไม่ต้องปิด Split-Horizon ไม่ต้องกังวล DR/BDR อีกต่อไป
   ⇒ **ใช้ได้กับทุก Routing Protocol รวม OSPF/BGP โดยไม่ต้อง Hack**
2. **NHRP Redirect/Shortcut** ทำหน้าที่ปรับ Forwarding Path ให้เหมาะสมที่สุด **ภายหลัง** (After
   the Fact) โดยไม่ต้องพึ่ง Routing Protocol เลย — เป็นกลไกที่ทำงานที่ **CEF/Data-Plane Level**

### กลไก NHRP Redirect/Shortcut ทำงานอย่างไร

```
① Spoke-A ส่ง Packet ไปหา Spoke-B โดยดูตารางเส้นทาง (ที่ Hub ทำ Next-Hop-Self ไว้)
   Spoke-A ────────────► HUB ────────────► Spoke-B
                (Traffic วิ่งผ่าน Hub ก่อนเสมอในครั้งแรก = "Hairpin")

② Hub ตรวจ CEF พบว่า Ingress Interface (Tunnel0 จาก Spoke-A) กับ Egress Interface
   (Tunnel0 ไปยัง Spoke-B) เป็น Interface เดียวกัน (Same-Interface Forwarding)
   ⇒ Hub รู้ว่านี่คือ Path ที่ไม่ Optimal (ไม่ควรต้องเสีย Bandwidth วิ่งผ่าน Hub 2 รอบ)

③ Hub ส่ง "NHRP Redirect" (Traffic Indication) กลับไปบอก Spoke-A ว่า
   "ปลายทางนี้มี Path ที่ดีกว่า ลองถามตรงดูสิ"

④ Spoke-A ส่ง "NHRP Resolution Request" ไปยัง Hub เพื่อขอ NBMA Address จริงของ Spoke-B
   Hub ตอบกลับด้วย "NHRP Resolution Reply" (Hub ทำหน้าที่ NHRP Server ยืนยันข้อมูล mapping)

⑤ Spoke-A ได้ NBMA Address ของ Spoke-B แล้ว ⇒ สร้าง Dynamic mGRE/IPsec Session ตรงถึง
   Spoke-B ⇒ CEF ติดตั้ง "NHRP Shortcut Route" (Host Route /32 แบบ Dynamic) แทน Route
   เดิมที่ชี้ผ่าน Hub

Spoke-A ◄─────────────────────────────────────► Spoke-B   (Shortcut Path หลังจากนี้)
```

### เปรียบเทียบ DMVPN 3 Phase

| คุณสมบัติ | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| Tunnel Interface ที่ Hub | mGRE (Multipoint) | mGRE (Multipoint) | mGRE (Multipoint) |
| Tunnel Interface ที่ Spoke | **p2p GRE** (แยก Tunnel ต่อ Hub) | **mGRE** (Tunnel เดียวคุยได้ทุก Peer) | **mGRE** |
| Spoke-to-Spoke Traffic | ผ่าน Hub เสมอ (Hairpin ตลอดไป) | ตรงถึงกัน (ต้องพึ่ง Routing Protocol เห็น Next-Hop เดิม) | ตรงถึงกัน (Redirect/Shortcut ไม่พึ่ง Routing Protocol) |
| Hub ทำ Next-Hop-Self ได้ไหม | ได้ (ไม่กระทบ เพราะไม่มี Spoke-to-Spoke อยู่แล้ว) | **ไม่ได้** (ต้องคง Next-Hop เดิม) | **ได้ตามปกติ** |
| รองรับ Routing Protocol | ทุกตัว | EIGRP (มีเงื่อนไข), OSPF/BGP ยาก | **ทุกตัวโดยไม่ต้อง Hack** |
| Config เพิ่มที่ Hub | - | - | `ip nhrp redirect` |
| Config เพิ่มที่ Spoke | - | - | `ip nhrp shortcut` |
| ใช้เมื่อไหร่ | Lab เก่า/เรียนพื้นฐาน | องค์กรเล็ก ใช้ EIGRP อยู่แล้ว | **Production จริงระดับ Enterprise/CCIE** |

> **ทำไม Phase 3 คือค่า Default ที่แนะนำใน Production ปัจจุบัน**: เพราะมันแยก concern
> Routing Protocol กับ Data-Plane Optimization ออกจากกันอย่างสมบูรณ์ — ทีม Routing ออกแบบ
> OSPF/BGP ตามความถนัดขององค์กรได้เต็มที่ โดยไม่ต้องกังวลเรื่อง Spoke-to-Spoke เลย เพราะ NHRP
> จัดการให้อัตโนมัติที่ Data-Plane

---

## Step 522 — Config Phase 3: `ip nhrp redirect` / `ip nhrp shortcut` และเปลี่ยนไป OSPF

### คำสั่งหลักของ Phase 3

| คำสั่ง | ใช้ที่ | ความหมาย |
|---|---|---|
| `ip nhrp redirect` | **Hub เท่านั้น** (Interface Tunnel0) | เปิดให้ Hub ตรวจ Same-Interface Forwarding และส่ง NHRP Traffic Indication (Redirect) กลับไปยัง Spoke ต้นทาง |
| `ip nhrp shortcut` | **Spoke ทุกตัว** (Interface Tunnel0) | เปิดให้ Spoke ยอมรับ Redirect ที่ได้รับ แล้วทำ NHRP Resolution + ติดตั้ง Shortcut Route ผ่าน CEF |

> ทั้งสองคำสั่งต้องเปิด**คู่กัน** — เปิด `redirect` ที่ Hub อย่างเดียวแต่ Spoke ไม่มี `shortcut`
> จะทำให้ Hub ส่ง Redirect ไปแต่ Spoke เพิกเฉย (Traffic ยัง Hairpin ผ่าน Hub เหมือน Phase 1)

### เปลี่ยน Routing Protocol จาก EIGRP → OSPF เพื่อพิสูจน์ความอิสระของ Phase 3

Part 52 ใช้ EIGRP AS 200 พร้อม `no ip split-horizon eigrp 200` ที่ Hub — Part นี้จะ **ถอด EIGRP
ออกทั้งหมด** แล้วแทนที่ด้วย **OSPF Process 100, Area 51** (Area แยกเฉพาะ DMVPN Cloud ไม่ปนกับ
Backbone Area 0 ของ Campus) โดยใช้ **Network Type `point-to-multipoint`** บน Tunnel0 ซึ่งเป็น
Best Practice มาตรฐานสำหรับ DMVPN เพราะ:

- ไม่มีการเลือก DR/BDR (ตัดปัญหา NBMA/Broadcast Network Type ที่ Phase 2 เจอ)
- แต่ละ Neighbor มองเป็น **Point-to-Point Link ย่อย** ภายใน Tunnel เดียว — Hub ทำ
  Next-Hop-Self ได้ตามปกติของ OSPF (ทุก LSA Type 1 ระบุ Next-Hop เป็นตัวเอง) โดยไม่กระทบ
  Adjacency กับ Spoke ตัวอื่น

### Config เต็ม — WAN-EDGE-1 (Hub, Phase 3 + OSPF)

```
WAN-EDGE-1(config)# router eigrp 200
WAN-EDGE-1(config-router)# no network 172.16.100.0 0.0.0.255
WAN-EDGE-1(config-router)# no network 10.10.254.0 0.0.0.3
WAN-EDGE-1(config)# no router eigrp 200
! ถอด EIGRP ออกทั้ง Process เพราะ Part นี้เปลี่ยนไปใช้ OSPF ทั้งหมดบน WAN Domain

WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# description ** DMVPN Phase 3 Hub (Primary) - mGRE/NHRP/IPsec **
WAN-EDGE-1(config-if)# ip address 172.16.100.1 255.255.255.0
WAN-EDGE-1(config-if)# ip mtu 1400
WAN-EDGE-1(config-if)# ip nhrp authentication DMVPN-KEY
WAN-EDGE-1(config-if)# ip nhrp network-id 100
WAN-EDGE-1(config-if)# ip nhrp redirect
! <-- เพิ่มใหม่: เปิด Phase 3 Redirect ที่ Hub
WAN-EDGE-1(config-if)# ip ospf network point-to-multipoint
WAN-EDGE-1(config-if)# ip ospf 100 area 51
WAN-EDGE-1(config-if)# tunnel source GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# tunnel mode gre multipoint
WAN-EDGE-1(config-if)# tunnel key 100
WAN-EDGE-1(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
WAN-EDGE-1(config-if)# exit

WAN-EDGE-1(config)# router ospf 100
WAN-EDGE-1(config-router)# router-id 1.1.1.21
WAN-EDGE-1(config-router)# area 51 stub no-summary
WAN-EDGE-1(config-router)# passive-interface default
WAN-EDGE-1(config-router)# no passive-interface Tunnel0
WAN-EDGE-1(config-router)# network 172.16.100.0 0.0.0.255 area 51
WAN-EDGE-1(config-router)# network 10.10.254.0 0.0.0.3 area 0
WAN-EDGE-1(config-router)# default-information originate
WAN-EDGE-1(config-router)# exit
```

> **`area 51 stub no-summary`**: ทำให้ WAN-EDGE-1 (ซึ่งเป็น ABR ระหว่าง Area 0 กับ Area 51)
> ฉีด **Default Route (0.0.0.0/0) เพียงเส้นเดียว** เข้า Area 51 แทนที่จะส่ง Inter-Area Route
> ทุกเส้นของ Campus เข้ามาให้ Spoke เห็น — เหตุผลออกแบบละเอียดอยู่ใน Step 523

### Config เต็ม — BRANCH-RTR (Spoke, Phase 3 + OSPF)

```
BRANCH-RTR(config)# no router eigrp 200
! ถอด EIGRP ออกเช่นเดียวกับ Hub

BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# description ** DMVPN Phase 3 Spoke - mGRE/NHRP/IPsec to WAN-EDGE-1 **
BRANCH-RTR(config-if)# ip address 172.16.100.11 255.255.255.0
BRANCH-RTR(config-if)# ip mtu 1400
BRANCH-RTR(config-if)# ip nhrp authentication DMVPN-KEY
BRANCH-RTR(config-if)# ip nhrp network-id 100
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
BRANCH-RTR(config-if)# ip nhrp shortcut
! <-- เพิ่มใหม่: เปิด Phase 3 Shortcut ที่ Spoke
BRANCH-RTR(config-if)# ip ospf network point-to-multipoint
BRANCH-RTR(config-if)# ip ospf 100 area 51
BRANCH-RTR(config-if)# tunnel source GigabitEthernet0/1
BRANCH-RTR(config-if)# tunnel mode gre multipoint
BRANCH-RTR(config-if)# tunnel key 100
BRANCH-RTR(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
BRANCH-RTR(config-if)# exit

BRANCH-RTR(config)# router ospf 100
BRANCH-RTR(config-router)# router-id 172.20.0.1
BRANCH-RTR(config-router)# area 51 stub
BRANCH-RTR(config-router)# passive-interface default
BRANCH-RTR(config-router)# no passive-interface Tunnel0
BRANCH-RTR(config-router)# network 172.16.100.0 0.0.0.255 area 51
BRANCH-RTR(config-router)# network 10.20.1.0 0.0.0.255 area 51
BRANCH-RTR(config-router)# exit
```

BRANCH2-RTR ใช้ config รูปแบบเดียวกัน โดยเปลี่ยน `ip address 172.16.100.12`, `router-id 172.20.1.1`,
และ `network 10.20.2.0 0.0.0.255 area 51`

### ตรวจสอบ OSPF Adjacency ผ่าน Tunnel0 (Point-to-Multipoint)

```
WAN-EDGE-1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.20.0.1        0   FULL/  -        00:01:47    172.16.100.11   Tunnel0
172.20.1.1        0   FULL/  -        00:01:52    172.16.100.12   Tunnel0
```

`Pri = 0` และ `State = FULL/ -` (ไม่มี `/DR` หรือ `/BDR`) ยืนยันว่า Network Type
Point-to-Multipoint ทำงานถูกต้อง — ไม่มี DR/BDR Election เกิดขึ้นเลยบน DMVPN Cloud นี้

---

## Step 523 — Summarization ที่ Hub สำหรับ Phase 3

### ทำไมต้องคิดเรื่อง Summarization ใหม่ใน Phase 3

ใน Phase 3 การที่ Spoke จะ Shortcut ไปยัง Spoke อื่นได้ ต้องมี**การ Hairpin ผ่าน Hub อย่างน้อย
1 Packet แรก** เสมอ (ดู Step 521 ข้อ ①-②) เพื่อให้ Hub ตรวจพบ Same-Interface Forwarding และ
ส่ง Redirect กลับ — นั่นแปลว่า **Spoke ไม่จำเป็นต้องรู้ Route เฉพาะเจาะจง (Specific Prefix)
ของ Spoke อื่นเลยตั้งแต่แรก** เพียงแค่มี Route ใดๆที่ชี้ไปทาง Hub ก็เพียงพอให้ Packet แรกไปถึง
Hub ได้ ⇒ **นี่คือจุดที่ทำให้ Phase 3 Summarize ได้กว้างกว่า Phase 1/2 มาก โดยไม่กระทบ
Reachability** เพราะ NHRP Shortcut จะติดตั้ง Host Route (`/32`) ที่เฉพาะเจาะจงกว่า Summary
เสมอโดยอัตโนมัติหลังจากนั้น (Longest-Prefix-Match ชนะ Summary Route ทันที)

```
ก่อน Shortcut:  Spoke-A มี Route "10.20.0.0/16 via Hub"  (Summary กว้างๆ)
                Packet แรกไป 10.20.2.5 (Spoke-B) → Hairpin ผ่าน Hub → Redirect

หลัง Shortcut:  Spoke-A มี Route "10.20.2.5/32 via Spoke-B (Shortcut, Dynamic)"
                Route นี้ "แคบกว่า" Summary Route เดิม ⇒ CEF เลือก Route นี้เป็นหลัก
                ตารางเส้นทางกลับไปเป็น Summary Route เดิมโดยอัตโนมัติถ้า Shortcut หมดอายุ (Purge)
```

### ออกแบบ Address Block ให้ Summarize ได้ (ทำตั้งแต่ต้น)

| Site | Subnet | อยู่ในบล็อกใหญ่ |
|---|---|---|
| BRANCH-RTR LAN | 10.20.1.0/24 | `10.20.0.0/16` |
| BRANCH2-RTR LAN | 10.20.2.0/24 | `10.20.0.0/16` |
| (สำรองสำหรับ Spoke ใหม่ในอนาคต) | 10.20.3.0/24 – 10.20.255.0/24 | `10.20.0.0/16` |

การจอง `10.20.0.0/16` ให้ Spoke LAN ทั้งหมดตั้งแต่ต้น (แม้ปัจจุบันมีแค่ 2 Spoke) ทำให้ WAN-EDGE-1
สามารถ Summarize เป็นเส้นเดียวไปยัง Backbone Area 0 ได้ในอนาคตเมื่อ Spoke เพิ่มขึ้นเรื่อยๆ:

```
WAN-EDGE-1(config)# router ospf 100
WAN-EDGE-1(config-router)# area 51 range 10.20.0.0 255.255.0.0
```

> `area 51 range` เป็นคำสั่ง Summarization ที่ **ABR** (WAN-EDGE-1) ทำเมื่อส่ง LSA จาก Area 51
> ออกไปยัง Area 0 — เห็นผลเมื่อ Spoke LAN เพิ่มจำนวนมากขึ้น (ในเบื้องต้นแค่ 2 Subnet ยังไม่เห็น
> ความต่างชัดนัก แต่เป็น Design ที่ถูกต้องตั้งแต่ Day 1)

### ทิศทางตรงกันข้าม: ทำให้ Spoke เห็น Route จาก Backbone แบบ Summarize/Default

จาก Step 522 เราตั้ง `area 51 stub no-summary` ที่ Hub และ `area 51 stub` ที่ Spoke ไว้แล้ว —
ผลคือ Spoke จะเห็น**เฉพาะ Default Route (0.0.0.0/0)** สำหรับทุกอย่างที่อยู่นอก Area 51
(Campus VLAN 10/20/30/40/99 ทั้งหมดจาก [00-ip-address-plan.md](00-ip-address-plan.md)) แทนที่จะ
เห็น Inter-Area Route เป็นสิบๆเส้น:

```
BRANCH-RTR# show ip route ospf
     10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O IA    10.20.2.0/24 [110/1001] via 172.16.100.1, 00:14:02, Tunnel0
O*IA 0.0.0.0/0 [110/1] via 172.16.100.1, 00:14:02, Tunnel0
```

สังเกตว่า BRANCH-RTR เห็น `10.20.2.0/24` (Prefix ของ BRANCH2-RTR ซึ่งอยู่ใน Area 51 เดียวกัน —
Intra-Area จึงยังเห็นแบบเฉพาะเจาะจงตามปกติของ Link-State Protocol) และเห็น **Default Route
เพียงเส้นเดียว** แทน Route ของ Campus ทั้งหมด — Design นี้ทำให้ตารางเส้นทางของ Spoke เล็กและ
Stable แม้ Campus ข้างในจะมี VLAN/Subnet เพิ่มขึ้นอีกกี่สิบตัวก็ตาม โดยไม่กระทบ Phase 3
Spoke-to-Spoke เลย เพราะกลไก Shortcut ทำงานที่ NHRP/CEF ไม่เกี่ยวกับว่า Route เดิมเป็น
Specific หรือ Default

---

## Step 524 — Dual-Hub DMVPN Design

### เหตุผลที่ต้องมี Hub สำรอง

Lab จาก Part 52 และ Step 521-523 มี **WAN-EDGE-1 เป็น Hub ตัวเดียว** — ถ้า WAN-EDGE-1 ล่ม
(Hardware Failure, Software Crash, Link ขาด) **DMVPN Cloud ทั้งหมดล่มตามไปด้วย** เพราะ Spoke
ทุกตัว Register NHRP ไปที่ Hub เดียว ไม่มี Path สำรอง — Enterprise Production ต้องมี **Hub
สำรองอย่างน้อย 1 ตัว** ซึ่งใน Lab ของหลักสูตรนี้คือ **WAN-EDGE-2** (มีอยู่แล้วใน Topology หลัก
ตาม [00-ip-address-plan.md](00-ip-address-plan.md) — Loopback0 `1.1.1.22/32`, เชื่อม ISP-RTR ที่
`203.0.113.6`, เชื่อม CORE-SW2)

### แนวคิด NHS (Next Hop Server) หลายตัว

Spoke ที่มี Hub มากกว่า 1 ตัวจะกำหนด `ip nhrp nhs` **ได้หลายบรรทัด** — Spoke จะพยายาม Register
กับทุก NHS ที่ระบุไว้ และเก็บสถานะ **Reachability** ของแต่ละ NHS ไว้แยกกัน ถ้า NHS ตัวใดไม่ตอบ
สนอง (Registration Timeout) Spoke จะ Fail-Over ไปใช้ NHS ตัวถัดไปโดยอัตโนมัติสำหรับ Traffic
ใหม่ (ส่วนการ**เลือกว่าตัวไหนเป็น Primary** ควบคุมผ่าน Routing Protocol Metric ใน Step 526
ไม่ใช่ผ่านลำดับการพิมพ์คำสั่ง NHS)

### Config เต็ม — WAN-EDGE-2 (Hub ตัวที่สอง, Single-Cloud, Phase 3)

WAN-EDGE-2 เข้าร่วม DMVPN Cloud เดียวกัน (Subnet `172.16.100.0/24` เดียวกัน, Network-ID/Tunnel
Key เดียวกัน) ในฐานะ NHRP Server ตัวที่สอง:

```
WAN-EDGE-2(config)# interface Tunnel0
WAN-EDGE-2(config-if)# description ** DMVPN Phase 3 Hub (Backup) - mGRE/NHRP/IPsec **
WAN-EDGE-2(config-if)# ip address 172.16.100.2 255.255.255.0
WAN-EDGE-2(config-if)# ip mtu 1400
WAN-EDGE-2(config-if)# ip nhrp authentication DMVPN-KEY
WAN-EDGE-2(config-if)# ip nhrp network-id 100
WAN-EDGE-2(config-if)# ip nhrp redirect
WAN-EDGE-2(config-if)# ip ospf network point-to-multipoint
WAN-EDGE-2(config-if)# ip ospf 100 area 51
WAN-EDGE-2(config-if)# tunnel source GigabitEthernet0/0/0
WAN-EDGE-2(config-if)# tunnel mode gre multipoint
WAN-EDGE-2(config-if)# tunnel key 100
WAN-EDGE-2(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
WAN-EDGE-2(config-if)# exit

WAN-EDGE-2(config)# router ospf 100
WAN-EDGE-2(config-router)# router-id 1.1.1.22
WAN-EDGE-2(config-router)# area 51 stub no-summary
WAN-EDGE-2(config-router)# area 51 range 10.20.0.0 255.255.0.0
WAN-EDGE-2(config-router)# passive-interface default
WAN-EDGE-2(config-router)# no passive-interface Tunnel0
WAN-EDGE-2(config-router)# network 172.16.100.0 0.0.0.255 area 51
WAN-EDGE-2(config-router)# network 10.10.254.4 0.0.0.3 area 0
WAN-EDGE-2(config-router)# default-information originate
WAN-EDGE-2(config-router)# exit
```

### Config เต็ม — Spoke ปรับให้มี Dual NHS

BRANCH-RTR (และ BRANCH2-RTR ในรูปแบบเดียวกัน) เพิ่มบรรทัด `ip nhrp nhs` ตัวที่สองสำหรับ
WAN-EDGE-2:

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
! NHS #1 = WAN-EDGE-1 (Primary - ดูเหตุผลเรื่อง Preference ใน Step 526)
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast
! NHS #2 = WAN-EDGE-2 (Backup)
BRANCH-RTR(config-if)# exit
```

### ตรวจสอบสถานะ NHS ทั้งสองตัวที่ Spoke

```
BRANCH-RTR# show ip nhrp nhs detail

Legend: E=Expecting Replies, R=Responding, W=Waiting

Tunnel0:
  172.16.100.1  RE priority = 0 cluster = 0    <- WAN-EDGE-1: Registered สำเร็จ
      req-sent 42 req-failed 0 repl-recv 42
  172.16.100.2  RE priority = 0 cluster = 0    <- WAN-EDGE-2: Registered สำเร็จ
      req-sent 40 req-failed 0 repl-recv 40
```

`RE` (Registered) บนทั้งสองแถวยืนยันว่า BRANCH-RTR Register NHRP สำเร็จกับ Hub ทั้งสองตัวพร้อม
กัน — พร้อมสำหรับการ Failover ในกรณีที่ Hub ตัวใดตัวหนึ่งล่ม (พิสูจน์จริงใน Step 530)

---

## Step 525 — Dual-Hub Dual-Cloud vs Dual-Hub Single-Cloud

Design ของ Step 524 ที่ใช้ **Tunnel0 เดียวกัน Subnet เดียวกัน** ทั้งสอง Hub เรียกว่า
**Single-Cloud** — แต่มีอีกแนวทางที่ใช้กันในหลายองค์กรคือ **Dual-Cloud** (แยก Tunnel Interface
และ Subnet ต่อ Hub) ทั้งสองแบบมี Trade-off ต่างกันชัดเจน:

```
DUAL-HUB SINGLE-CLOUD (Step 524 - ที่ Lab นี้ใช้)          DUAL-HUB DUAL-CLOUD (ทางเลือกอื่น)
┌─────────────┐         ┌─────────────┐              ┌─────────────┐         ┌─────────────┐
│ WAN-EDGE-1   │         │ WAN-EDGE-2   │              │ WAN-EDGE-1   │         │ WAN-EDGE-2   │
│ Tunnel0      │         │ Tunnel0      │              │ Tunnel0      │         │ Tunnel1      │
│ .1/24        │         │ .2/24        │              │ 172.16.101.1 │         │ 172.16.102.1 │
└──────┬──────┘         └──────┬──────┘              └──────┬──────┘         └──────┬──────┘
       │   172.16.100.0/24 (Subnet เดียว)   │                     │  Subnet คนละใบ   │
       └──────────┬──────────┘                     ┌──────┴──────┐   ┌──────┴──────┐
              ┌────┴────┐                           │  Tunnel0     │   │  Tunnel1     │
              │ Spoke    │  Tunnel0 เดียว            │ .11/24       │   │ .11/24       │
              │ (1 Intf) │  คุยกับ Hub ทั้งคู่          └─────────────┘   └─────────────┘
              └─────────┘                                  Spoke มี 2 Tunnel Interface
```

| คุณสมบัติ | Dual-Hub **Single-Cloud** | Dual-Hub **Dual-Cloud** |
|---|---|---|
| Tunnel Interface ที่ Spoke | 1 (Tunnel0 เดียวคุยกับ Hub ทั้งคู่) | 2 (Tunnel0→Hub1, Tunnel1→Hub2) |
| NHRP Subnet | ใบเดียว ทั้งสอง Hub อยู่ Subnet เดียวกัน | คนละ Subnet ต่อ Hub |
| Spoke-to-Spoke Shortcut | ทำงานได้ทันทีไม่ว่า Spoke จะ Register กับ Hub ตัวไหนเป็นหลัก (อยู่ Subnet เดียวกัน) | Shortcut ได้เฉพาะ Spoke ที่ใช้ **Tunnel เดียวกัน** (เชื่อม Hub ตัวเดียวกัน) — ถ้าคนละ Hub ต้องผ่าน Hub-to-Hub Link เสมอ |
| Routing Protocol Adjacency ต่อ Spoke | 2 Neighbor บน Interface เดียว | 2 Neighbor บน 2 Interface (Design คุ้นเคยกว่าสำหรับทีมที่ชินกับ Dual-Path แยกกายภาพ) |
| ความซับซ้อน Config | ต่ำกว่า (Interface เดียว) | สูงกว่า (ต้อง Manage 2 Tunnel, 2 IPsec Profile Instance, 2 Routing Adjacency Set) |
| Failure Isolation | Hub ล่ม 1 ตัว กระทบเฉพาะ NHS ฝั่งนั้น แต่ Subnet เดียวกันยังทำงานต่อ | แยกขาดจากกันสมบูรณ์ (Fault Domain แยกเป็น Cloud คนละใบ จริงจังกว่า) |
| เหมาะกับ | องคาพยพ Hub ทั้งสองอยู่ Data Center เดียวกัน/ใกล้กัน, ต้องการ Config เรียบง่าย | Hub ทั้งสองอยู่ Data Center คนละที่ (Geo-Redundancy), ต้องการแยก Fault Domain อย่างเข้มงวด, หรือใช้ ISP คนละราย |

### ตัวอย่าง Config โครงร่าง Dual-Cloud (สำหรับเปรียบเทียบเท่านั้น — Lab หลักของ Part นี้ใช้ Single-Cloud)

```
! Spoke (BRANCH-RTR) แบบ Dual-Cloud - มี 2 Tunnel Interface
interface Tunnel0
 ip address 172.16.101.11 255.255.255.0
 ip nhrp nhs 172.16.101.1 nbma 203.0.113.2 multicast   ! ไปยัง WAN-EDGE-1 เท่านั้น
 ip nhrp shortcut
 tunnel source GigabitEthernet0/1
 tunnel destination 203.0.113.2                          ! Dual-Cloud มักใช้ p2p GRE ต่อ Hub
!
interface Tunnel1
 ip address 172.16.102.11 255.255.255.0
 ip nhrp nhs 172.16.102.1 nbma 203.0.113.6 multicast   ! ไปยัง WAN-EDGE-2 เท่านั้น
 ip nhrp shortcut
 tunnel source GigabitEthernet0/1
 tunnel destination 203.0.113.6
```

> **ข้อสรุปสำหรับ Lab ของหลักสูตรนี้**: เราเลือก **Single-Cloud** ต่อใน Step 526-530 เพราะ
> Topology จริงมี WAN-EDGE-1/WAN-EDGE-2 อยู่ Data Center เดียวกัน (เชื่อมกันด้วย Cross-Link
> `10.10.254.8/30` ตาม IP Plan) ทำให้ Fault Domain ที่แยกจัดกันเข้มงวดแบบ Dual-Cloud ไม่จำเป็น
> เท่ากับความง่ายของ Config ที่ได้จาก Single-Cloud

---

## Step 526 — ออกแบบ Routing Protocol ให้ Preferred Hub

### ปัญหา: Single-Cloud มี Interface เดียว แต่มี Neighbor 2 ตัว

เพราะ Design ที่เลือกใน Step 524-525 คือ Single-Cloud (Spoke มี Tunnel0 ใบเดียวคุยกับ Hub ทั้ง
สอง) คำสั่งปรับ Cost แบบปรับที่ Interface ทั้งใบ (`ip ospf cost`) จะกระทบ Neighbor ทั้งสองตัว
เท่ากัน — ไม่สามารถทำให้ WAN-EDGE-1 "ถูกกว่า" WAN-EDGE-2 ด้วยวิธีนี้ได้ ต้องใช้ความสามารถพิเศษ
ของ **OSPF Network Type Point-to-Multipoint**: คำสั่ง **`neighbor <ip> cost <value>`** ซึ่ง
กำหนด Cost ได้ **แยกตาม Neighbor** แม้อยู่ Interface เดียวกัน

### Config OSPF Cost ให้ WAN-EDGE-1 เป็น Primary, WAN-EDGE-2 เป็น Backup

```
BRANCH-RTR(config)# router ospf 100
BRANCH-RTR(config-router)# neighbor 172.16.100.1 cost 100
! WAN-EDGE-1 = Cost 100 (ต่ำกว่า = Preferred/Primary)
BRANCH-RTR(config-router)# neighbor 172.16.100.2 cost 200
! WAN-EDGE-2 = Cost 200 (สูงกว่า = Backup)
BRANCH-RTR(config-router)# exit
```

ทำเช่นเดียวกันบน BRANCH2-RTR (Cost เท่ากันทุก Spoke เพื่อ Preference สอดคล้องกันทั้ง Cloud)

### ตรวจสอบผล — Route ไปยัง Default/Summary ต้องเลือก WAN-EDGE-1 เป็นหลัก

```
BRANCH-RTR# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 100", distance 110, metric 100, candidate default path
  Last update from 172.16.100.1 on Tunnel0, 00:03:12 ago
  Routing Descriptor Blocks:
  * 172.16.100.1, from 1.1.1.21, 00:03:12 ago, via Tunnel0     <- WAN-EDGE-1 (Metric 100)
      Route metric is 100, traffic share count is 1
    172.16.100.2, from 1.1.1.22, 00:05:40 ago, via Tunnel0      <- WAN-EDGE-2 (Metric 200, Backup)
      Route metric is 200, traffic share count is 1
```

Route หลัก (`*`) ชี้ผ่าน `172.16.100.1` (WAN-EDGE-1, Metric 100) — เส้นไป WAN-EDGE-2 ยังอยู่ใน
LSDB/Topology Table แต่ไม่ถูกติดตั้งเป็น Active Route จนกว่า WAN-EDGE-1 จะ Fail (พิสูจน์ Failover
จริงใน Step 529/530)

### หมายเหตุสำหรับ EIGRP (กรณีใช้ Dual-Cloud แทน)

ถ้าเลือก Design Dual-Cloud จาก Step 525 (มี Tunnel0/Tunnel1 แยกกัน) การ Tune EIGRP Metric จะ
ง่ายกว่ามาก เพราะปรับ **`delay`** ได้ตรงที่ Interface โดยไม่ต้องพึ่ง Neighbor-Level Command:

```
! ตัวอย่างสำหรับ Dual-Cloud เท่านั้น (ไม่ใช่ Config ของ Lab หลัก)
interface Tunnel0
 delay 100        ! ไปยัง WAN-EDGE-1 (Primary) — Delay ต่ำ = Metric ต่ำ = Preferred
interface Tunnel1
 delay 500        ! ไปยัง WAN-EDGE-2 (Backup) — Delay สูง = Metric สูง = Backup
```

สำหรับ Single-Cloud + EIGRP (Interface เดียว 2 Neighbor) EIGRP **ไม่มี** คำสั่งระดับ
Per-Neighbor Metric เทียบเท่า `neighbor cost` ของ OSPF โดยตรง วิธีแก้คือใช้ **`offset-list`**
ร่วมกับ ACL ที่ Match เฉพาะ Prefix ที่มาจาก Neighbor ที่ต้องการ Backup แต่ซับซ้อนกว่า OSPF
มาก — นี่คือเหตุผลเชิงปฏิบัติอีกข้อที่ทำให้ Part นี้เลือกเปลี่ยนไปใช้ **OSPF** ตั้งแต่ Step 522

---

## Step 527 — DMVPN กับ FlexVPN

### ทบทวนจาก Part 51: FlexVPN คืออะไร (สรุปสั้น)

FlexVPN คือ Framework VPN ของ Cisco ที่สร้างขึ้นบน **IKEv2 ล้วน** (ต่างจาก DMVPN Classic ที่ยัง
พึ่งพา ISAKMP/IKEv1 Legacy Command Set ผสมกับ `crypto ipsec profile`) — FlexVPN รวม Feature
หลากหลาย (Site-to-Site, Remote Access/AnyConnect, Hub-Spoke) ไว้ใน Syntax เดียวกันคือ
`crypto ikev2 profile` + `crypto ikev2 authorization policy`

### เปรียบเทียบ DMVPN Classic vs FlexVPN Spoke-Hub

| คุณสมบัติ | DMVPN Classic (ที่ใช้ตลอด Part 52-53) | FlexVPN Spoke-Hub |
|---|---|---|
| IKE Version | IKEv1 (ISAKMP) เป็นค่าพื้นฐาน ปรับเป็น IKEv2 ได้แต่ Syntax ยังลูกผสม | **IKEv2 เท่านั้น** ตั้งแต่ต้น |
| Protection Command | `tunnel protection ipsec profile` (ผูก Static Profile) | `tunnel protection ipsec profile` เหมือนกัน แต่ Profile อ้าง `crypto ikev2 profile` ที่รองรับ **Dynamic Authorization ต่อ Spoke** ผ่าน AAA |
| Routing/Multipoint | GRE mGRE + NHRP (เหมือนกันทั้งคู่) | **เหมือนกันทั้งคู่** — FlexVPN ไม่ได้แทนที่ GRE/NHRP เพียงแทนที่ชั้น IKE/IPsec |
| Per-Spoke Authorization | ยาก (ต้อง Pre-Shared Key/Certificate เดียวกันทั้ง Cloud หรือทำ Per-Peer Crypto Map ที่ซับซ้อน) | **ง่าย** — ผ่าน `crypto ikev2 authorization policy` + RADIUS/AAA ส่ง Attribute (เช่น Route ที่อนุญาต, ACL) ต่อ Spoke แบบ Dynamic |
| Spoke Provisioning ขนาดใหญ่ | Manual ต่อ Spoke | รองรับ **EAP/Certificate-based Auto-Provisioning** เหมาะกับ Spoke จำนวนมาก (ร้อย-พันสาขา) |
| ความเข้ากันได้กับ Non-Cisco | ต่ำ (DMVPN เป็น Cisco Proprietary ทั้ง NHRP) | เท่ากัน (NHRP ยังเป็น Cisco Proprietary เหมือนกัน — FlexVPN ต่างที่ชั้น IKE เท่านั้น) |
| IOS Version ที่รองรับ | ทุก IOS/IOS-XE ที่ทำ GRE ได้ | ต้องการ IOS ที่รองรับ IKEv2 (15.x ขึ้นไปเป็นส่วนใหญ่) |

### เมื่อไหร่ควรเลือก FlexVPN Spoke-Hub แทน Classic DMVPN

- องค์กรมี Compliance/Security Policy บังคับใช้ **IKEv2 เท่านั้น** (IKEv1 ถูกตัดออกจาก Baseline
  ความปลอดภัยตามมาตรฐานใหม่ๆ เช่น NIST/PCI-DSS ฉบับล่าสุด)
- ต้องการ **AAA-Driven Provisioning** — สาขาใหม่เสียบ Router แล้วขอ Config ผ่าน RADIUS ได้เลย
  โดยไม่ต้อง Pre-Config Crypto Map ทีละสาขา (สำคัญมากสำหรับ MSP/SP ที่ Deploy สาขาจำนวนมาก)
- ต้องการรวม Infrastructure เดียวกันระหว่าง **Site-to-Site DMVPN** และ **Remote-Access
  AnyConnect** (FlexVPN รองรับทั้งสองแบบด้วย IKEv2 Profile ชุดเดียวกัน ลด Operational
  Overhead ของทีม Security)

> **สำหรับ Lab ของหลักสูตรนี้**: เราคงใช้ **DMVPN Classic** ต่อไปตลอด Part 52-53 เพราะเป้าหมาย
> คือสอนกลไก NHRP/mGRE/Phase ให้เห็นภาพชัดที่สุด — FlexVPN Spoke-Hub ไม่เปลี่ยนกลไก NHRP/Phase
> เลย เปลี่ยนแค่ชั้น IKE/IPsec ด้านล่าง จึงเป็นเรื่อง "จะเลือกใช้เมื่อไหร่" มากกว่า "ต้องเรียน
> เพิ่มอะไรใหม่"

---

## Step 528 — QoS Integration บน DMVPN: Hierarchical QoS ต่อ Spoke

### ปัญหา: Spoke แต่ละสาขามีขนาดไม่เท่ากัน ต้องการ SLA ต่างกัน

Hub (WAN-EDGE-1/WAN-EDGE-2) มี Interface ทางกายภาพเพียงใบเดียวออกสู่ Internet แต่ต้องบริการ
Spoke หลายสาขาที่มี Bandwidth Contract ต่างกัน (เช่น สาขาใหญ่ 10 Mbps, สาขาเล็ก 2 Mbps) — การ
Apply Service-Policy ตรงที่ Physical Interface หรือ Tunnel0 เพียง Policy เดียวไม่พอ เพราะ
Traffic ของทุก Spoke ไหลผ่าน Tunnel0 ใบเดียวกัน (mGRE) จึงต้องใช้กลไกพิเศษที่แยก Traffic เป็น
**Per-Spoke Sub-Policy** — นี่คือรูปแบบ **Hierarchical QoS (HQoS)** ที่ [Part 40](part-040-advanced-qos.md)
สอนโครงสร้าง Parent/Child ไว้แล้ว ผสานกับกลไกเฉพาะของ DMVPN คือ **NHRP Group**

### กลไก NHRP Group + Per-Tunnel QoS

```
                          WAN-EDGE-1 (Hub)
                    ┌─────────────────────────┐
                    │  Tunnel0                 │
                    │  ip nhrp map group LARGE  │───► service-policy output QOS-LARGE (Shape 10M)
                    │       service-policy ...  │
                    │  ip nhrp map group SMALL  │───► service-policy output QOS-SMALL (Shape 2M)
                    │       service-policy ...  │
                    └───────────┬───────────────┘
             ┌───────────────────┴───────────────────┐
     ip nhrp group LARGE                      ip nhrp group SMALL
     (BRANCH-RTR สาขาใหญ่)                     (BRANCH2-RTR สาขาเล็ก)
```

Spoke ประกาศตัวว่าอยู่กลุ่มไหนด้วย `ip nhrp group <ชื่อ>` บน Tunnel0 ของตัวเอง — เมื่อ Spoke
Register NHRP เข้ามาที่ Hub, Hub จะ**จับคู่ Group Name กับ Service-Policy ที่กำหนดไว้ล่วงหน้า
โดยอัตโนมัติ** (ไม่ต้องรู้ NBMA Address ของ Spoke แต่ละตัวล่วงหน้าเลย — Scale ได้ดีมากเมื่อ
Spoke เพิ่มจำนวน)

### Config เต็ม — Policy-Map แบบ Hierarchical (ใช้ Class-Map จาก Part 40)

```
! Child Policy-Map (ใช้ Class-Map VOICE/VIDEO/CRITICAL-DATA ที่นิยามไว้ตั้งแต่ Part 40)
WAN-EDGE-1(config)# policy-map CHILD-QOS
WAN-EDGE-1(config-pmap)#  class VOICE
WAN-EDGE-1(config-pmap-c)#   priority percent 20
WAN-EDGE-1(config-pmap-c)#  class VIDEO
WAN-EDGE-1(config-pmap-c)#   bandwidth remaining percent 30
WAN-EDGE-1(config-pmap-c)#  class CRITICAL-DATA
WAN-EDGE-1(config-pmap-c)#   bandwidth remaining percent 30
WAN-EDGE-1(config-pmap-c)#  class class-default
WAN-EDGE-1(config-pmap-c)#   bandwidth remaining percent 20
WAN-EDGE-1(config-pmap-c)#   random-detect
WAN-EDGE-1(config-pmap-c)#  exit

! Parent Policy-Map สาขาใหญ่ - Shape 10 Mbps แล้วเรียก Child เข้าไปจัดคิวภายใน
WAN-EDGE-1(config)# policy-map QOS-LARGE
WAN-EDGE-1(config-pmap)#  class class-default
WAN-EDGE-1(config-pmap-c)#   shape average 10000000
WAN-EDGE-1(config-pmap-c)#   service-policy CHILD-QOS
WAN-EDGE-1(config-pmap-c)#  exit

! Parent Policy-Map สาขาเล็ก - Shape 2 Mbps ใช้ Child เดียวกัน (Ratio ปรับตาม % อัตโนมัติ)
WAN-EDGE-1(config)# policy-map QOS-SMALL
WAN-EDGE-1(config-pmap)#  class class-default
WAN-EDGE-1(config-pmap-c)#   shape average 2000000
WAN-EDGE-1(config-pmap-c)#   service-policy CHILD-QOS
WAN-EDGE-1(config-pmap-c)#  exit

! ผูก Policy-Map เข้ากับ NHRP Group บน Tunnel0 ของ Hub
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip nhrp map group BRANCH-LARGE service-policy output QOS-LARGE
WAN-EDGE-1(config-if)# ip nhrp map group BRANCH-SMALL service-policy output QOS-SMALL
WAN-EDGE-1(config-if)# exit
```

### Config ฝั่ง Spoke — ประกาศ Group ตัวเอง

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp group BRANCH-LARGE
! BRANCH-RTR เป็นสาขาใหญ่ (Contract 10 Mbps) -> ได้ QOS-LARGE โดยอัตโนมัติ
BRANCH-RTR(config-if)# exit

BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# ip nhrp group BRANCH-SMALL
! BRANCH2-RTR เป็นสาขาเล็ก (Contract 2 Mbps) -> ได้ QOS-SMALL โดยอัตโนมัติ
BRANCH2-RTR(config-if)# exit
```

> **ข้อดีสำคัญของแนวทางนี้**: Hub ไม่ต้องรู้ NBMA Address หรือ IP ของ Spoke ล่วงหน้าเพื่อ Apply
> QoS เลย — เพียงกำหนด Group Name ให้ตรงกันระหว่าง Hub (`ip nhrp map group`) กับ Spoke
> (`ip nhrp group`) ทำให้เพิ่ม Spoke สาขาใหม่ 100 สาขาโดยแค่เลือกว่าเป็น LARGE หรือ SMALL แล้ว
> Apply Policy ที่ถูกต้องให้อัตโนมัติทันทีที่ Register — Scale ได้ดีกว่าการทำ Per-VC Static
> Mapping แบบเก่ามาก

---

## Step 529 — Verification/Troubleshooting เฉพาะ Phase 3

### `show ip nhrp shortcut` — ตรวจสอบ Shortcut Route ที่เกิดขึ้นจริง

```
BRANCH-RTR# show ip nhrp shortcut

172.16.100.12/32 via 172.16.100.12
   Tunnel0 created 00:00:08, expire 01:59:51
   Type: dynamic, Flags: router rib
   NBMA address: 198.51.100.6
```

Entry นี้ยืนยันว่า BRANCH-RTR ได้ทำ NHRP Shortcut ไปยัง BRANCH2-RTR (`172.16.100.12`) เรียบร้อย
แล้ว, `Flags: router rib` หมายถึง Route นี้ถูกติดตั้งเข้า **RIB (Routing Information Base)**
จริง ไม่ใช่แค่ Cache ชั่วคราว — `expire 01:59:51` คือเวลาที่ Entry จะหมดอายุถ้าไม่มี Traffic
ใหม่มา Refresh (ค่า Default 2 ชั่วโมง)

### `show ip nhrp redirect` — ตรวจสอบว่า Hub ส่ง Redirect ไปแล้วกี่ครั้ง

```
WAN-EDGE-1# show ip nhrp redirect

Src: 172.16.100.11(Tunnel0) Dst: 172.16.100.12
    Type: redirect, Done, Notified: 1, Ack: 1
    Notified time: 00:00:08, Expire: 00:00:52
```

ยืนยันว่า WAN-EDGE-1 ตรวจพบ Same-Interface Forwarding ระหว่าง BRANCH-RTR (`.11`) กับ
BRANCH2-RTR (`.12`) และส่ง Redirect ไปแล้ว 1 ครั้ง (`Notified: 1`) และได้รับการ Acknowledge
กลับมา (`Ack: 1`) — ถ้า `Ack` ไม่ขึ้นหรือค้างที่ 0 แสดงว่า Spoke ปลายทางไม่มี `ip nhrp shortcut`
เปิดอยู่ (ตรวจ Config ตาม Step 522 ทันที)

### ยืนยันว่า Traffic จริงวิ่ง Shortcut ไม่ใช่ Hairpin ผ่าน Hub

```
BRANCH-RTR# show ip cef 10.20.2.0/24 detail
10.20.2.0/24, epoch 2, flags rib defined all rr
  nexthop 172.16.100.12 Tunnel0
  ! <-- ก่อน Shortcut: nexthop 172.16.100.1 (WAN-EDGE-1) ผ่าน Hub เท่านั้น
  ! หลัง Shortcut: nexthop เปลี่ยนเป็น 172.16.100.12 (BRANCH2-RTR) ตรงถึงกันเลย
```

วิธี Lab จริง: `ping 10.20.2.1 source 10.20.1.1` (จาก LAN ของ BRANCH-RTR ไป LAN ของ
BRANCH2-RTR) แล้วเทียบ `show ip cef 10.20.2.0/24` **ก่อน** และ **หลัง** Ping — ครั้งแรก Next-Hop
จะเป็น WAN-EDGE-1 (Hairpin), Ping รอบที่ 2 เป็นต้นไป Next-Hop จะเปลี่ยนเป็น BRANCH2-RTR โดยตรง
(Shortcut สำเร็จ) นอกจากนี้ `traceroute` จาก BRANCH-RTR ไป BRANCH2-RTR **ก่อน** Shortcut จะเห็น
Hop กลางเป็น WAN-EDGE-1 Tunnel IP (`172.16.100.1`) และ**หลัง** Shortcut จะเห็น Traceroute
กระโดดตรงถึงปลายทางเพียง 1 Hop (Latency ต่ำลงชัดเจน — วัดผลได้จริงด้วย `ping` ทั่วไป)

### ตรวจสอบ Dual-Hub Failover — NHS Re-Registration

```
! หลังจาก Shutdown WAN-EDGE-1 Tunnel0 (จำลอง Hub ตัวหลักล่ม)
BRANCH-RTR# show ip nhrp nhs detail

Legend: E=Expecting Replies, R=Responding, W=Waiting

Tunnel0:
  172.16.100.1  W  priority = 0 cluster = 0      <- WAN-EDGE-1: Waiting (ไม่ตอบสนอง)
      req-sent 3 req-failed 3 repl-recv 0
  172.16.100.2  RE priority = 0 cluster = 0       <- WAN-EDGE-2: ยัง Registered ปกติ
      req-sent 45 req-failed 0 repl-recv 45
```

`W` (Waiting) บนแถวของ WAN-EDGE-1 ยืนยันว่า Spoke ตรวจพบว่า NHS ตัวหลักไม่ตอบสนอง (3 ครั้ง
ติดต่อกันไม่มี Reply) ในขณะที่ WAN-EDGE-2 ยังคง `RE` (Registered) ปกติ — ผลที่ตามมาคือ OSPF
Adjacency กับ WAN-EDGE-1 จะ Dead Timer หมดและถอด Neighbor ออก ทำให้ Route ที่เหลือ (ไปทาง
WAN-EDGE-2, Cost 200) กลายเป็น Active Route โดยอัตโนมัติ — Traffic ทุก Flow (รวม Spoke-to-Spoke
Shortcut ที่มีอยู่เดิม) จะถูก Re-establish ผ่าน Path สำรองนี้แทน (รายละเอียด Failover เต็ม
รูปแบบอยู่ใน Step 530)

---

## Step 530 — Lab เต็มรูปแบบ: Dual-Hub DMVPN Phase 3

### สรุป Topology สุดท้ายของ Part นี้

```
                              ┌───────────────┐
                              │   ISP-RTR      │
                              └──┬─────────┬──┘
                       203.0.113.2   203.0.113.6
                              │             │
                      ┌───────┴────┐  ┌─────┴──────┐
                      │ WAN-EDGE-1  │  │ WAN-EDGE-2  │
                      │ (Primary)   │  │ (Backup)    │
                      │ Tunnel0     │  │ Tunnel0     │
                      │ .1/24       │  │ .2/24       │
                      │ redirect ✓  │  │ redirect ✓  │
                      └──────┬──────┘  └──────┬──────┘
                             │   172.16.100.0/24 (Single-Cloud)  │
                    ┌────────┴────────────────────┴────────┐
                    │         mGRE + NHRP + IPsec           │
                    └────┬───────────────────────────┬────┘
             ┌────────────┴───────────┐   ┌───────────┴────────────┐
             │  BRANCH-RTR             │   │  BRANCH2-RTR            │
             │  Tunnel0 .11            │   │  Tunnel0 .12            │
             │  shortcut ✓ group=LARGE │   │  shortcut ✓ group=SMALL │
             │  NHS: .1(cost100)/.2(200)│   │  NHS: .1(cost100)/.2(200)│
             │  LAN 10.20.1.0/24        │   │  LAN 10.20.2.0/24        │
             └─────────────────────────┘   └─────────────────────────┘
```

### สรุป Config Checklist ครบทุกจุด (ทวนจาก Step 521-528)

| จุด Config | WAN-EDGE-1 | WAN-EDGE-2 | BRANCH-RTR | BRANCH2-RTR |
|---|---|---|---|---|
| `ip nhrp redirect` | ✓ | ✓ | - | - |
| `ip nhrp shortcut` | - | - | ✓ | ✓ |
| `ip ospf network point-to-multipoint` | ✓ | ✓ | ✓ | ✓ |
| `area 51 stub (no-summary ที่ Hub)` | ✓ | ✓ | ✓ | ✓ |
| `neighbor <hub-ip> cost` | - | - | ✓ (100/200) | ✓ (100/200) |
| `ip nhrp map group ... service-policy` | ✓ | (เพิ่มเติมได้เหมือนกัน) | - | - |
| `ip nhrp group <name>` | - | - | ✓ (LARGE) | ✓ (SMALL) |
| `tunnel protection ipsec profile DMVPN-PROFILE` | ✓ | ✓ | ✓ | ✓ |

### Running-Config ฉบับย่อ — WAN-EDGE-1 (รวมทุก Feature ของ Part นี้)

```
crypto ikev2 proposal DMVPN-IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy DMVPN-IKEV2-POLICY
 proposal DMVPN-IKEV2-PROPOSAL
!
crypto ikev2 keyring DMVPN-KEYRING
 peer ANY-SPOKE
  address 0.0.0.0 0.0.0.0
  pre-shared-key Cisco123!
!
crypto ikev2 profile DMVPN-IKEV2-PROFILE
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN-KEYRING
!
crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile DMVPN-PROFILE
 set transform-set DMVPN-TSET
 set ikev2-profile DMVPN-IKEV2-PROFILE
!
policy-map CHILD-QOS
 class VOICE
  priority percent 20
 class VIDEO
  bandwidth remaining percent 30
 class CRITICAL-DATA
  bandwidth remaining percent 30
 class class-default
  bandwidth remaining percent 20
  random-detect
!
policy-map QOS-LARGE
 class class-default
  shape average 10000000
  service-policy CHILD-QOS
!
policy-map QOS-SMALL
 class class-default
  shape average 2000000
  service-policy CHILD-QOS
!
interface Tunnel0
 description ** DMVPN Phase 3 Hub (Primary) - mGRE/NHRP/IPsec **
 ip address 172.16.100.1 255.255.255.0
 ip mtu 1400
 ip nhrp authentication DMVPN-KEY
 ip nhrp network-id 100
 ip nhrp redirect
 ip nhrp map group BRANCH-LARGE service-policy output QOS-LARGE
 ip nhrp map group BRANCH-SMALL service-policy output QOS-SMALL
 ip ospf network point-to-multipoint
 ip ospf 100 area 51
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile DMVPN-PROFILE
!
router ospf 100
 router-id 1.1.1.21
 area 51 stub no-summary
 area 51 range 10.20.0.0 255.255.0.0
 passive-interface default
 no passive-interface Tunnel0
 network 172.16.100.0 0.0.0.255 area 51
 network 10.10.254.0 0.0.0.3 area 0
 default-information originate
```

### ขั้นตอนทดสอบทั้งหมด (สรุปเป็น Checklist)

1. **ตรวจ OSPF Adjacency ทั้ง 2 Hub**: `show ip ospf neighbor` บน BRANCH-RTR ต้องเห็น
   `172.16.100.1` และ `172.16.100.2` ทั้งคู่เป็น `FULL/ -`
2. **ตรวจ Preferred Path**: `show ip route 0.0.0.0` ต้องเห็น Active Route ผ่าน `172.16.100.1`
   (Metric 100) เท่านั้น ส่วน `172.16.100.2` (Metric 200) อยู่ใน Topology แต่ไม่ Active
3. **ทดสอบ Spoke-to-Spoke Shortcut**: `ping 10.20.2.1 source 10.20.1.1` จาก BRANCH-RTR ซ้ำ
   2-3 ครั้ง แล้วตรวจ `show ip nhrp shortcut` ต้องเห็น Entry ไปยัง `172.16.100.12`
4. **ทดสอบ Per-Spoke QoS**: `show policy-map interface Tunnel0` บน WAN-EDGE-1 ต้องเห็น
   Sub-Policy `QOS-LARGE` Apply ให้ Session ของ BRANCH-RTR และ `QOS-SMALL` ให้ BRANCH2-RTR
5. **ทดสอบ Hub Failover**: `shutdown` Tunnel0 บน WAN-EDGE-1 แล้วตรวจ:
   - `show ip nhrp nhs detail` บน BRANCH-RTR: WAN-EDGE-1 เปลี่ยนเป็น `W` (Waiting)
   - `show ip ospf neighbor`: Neighbor `172.16.100.1` หายไปหลัง Dead Timer หมด
   - `show ip route 0.0.0.0`: Active Route เปลี่ยนไปใช้ `172.16.100.2` (WAN-EDGE-2) โดยอัตโนมัติ
   - `ping 10.20.2.1 source 10.20.1.1` ยังสำเร็จ (Spoke-to-Spoke ยังทำงานต่อผ่าน Single-Cloud
     เดิม เพราะ Shortcut ที่มีอยู่แล้วไม่ได้ผูกกับ Hub ตัวใดตัวหนึ่งโดยเฉพาะ)
6. **Restore**: `no shutdown` Tunnel0 บน WAN-EDGE-1 แล้วตรวจว่า NHS กลับเป็น `RE` และ OSPF
   Adjacency/Route กลับมา Prefer WAN-EDGE-1 เหมือนก่อน Failover (Cost 100 ชนะ Cost 200 เสมอ
   เมื่อทั้งสอง Path พร้อมใช้งาน)

### ผลลัพธ์ที่คาดหวังจาก Lab นี้

Lab นี้พิสูจน์ครบทั้ง 3 คุณสมบัติหลักของ Part 53: **(1)** DMVPN Phase 3 ทำงานได้เต็มรูปแบบกับ
OSPF โดยไม่ต้อง Hack Routing Protocol เหมือน Phase 2, **(2)** Dual-Hub ให้ High Availability
จริงที่ระดับ Data-Plane (NHS Failover) และ Control-Plane (OSPF Route Re-Convergence) พร้อมกัน,
และ **(3)** QoS แบบ Per-Spoke ผ่าน NHRP Group ทำให้สาขาขนาดต่างกันได้ SLA ที่เหมาะสมโดยไม่ต้อง
Static Mapping ที่ Hub เลย — ครบทุกองค์ประกอบที่ CCIE Enterprise Infrastructure คาดหวังจาก
DMVPN Design ระดับ Production

---

## แบบฝึกหัดทวนความเข้าใจ Part 53

1. อธิบายลำดับขั้นตอนของ NHRP Redirect/Shortcut (Traffic Indication → Resolution → Shortcut)
   ตั้งแต่ Spoke-A ส่ง Packet แรกไปยัง Spoke-B จนกระทั่งเกิด Direct Path
2. เพราะเหตุใด DMVPN Phase 3 จึงทำให้ Hub ทำ Next-Hop-Self ได้ตามปกติ ในขณะที่ Phase 2 ทำไม่ได้?
3. ในการออกแบบ Single-Cloud Dual-Hub ด้วย OSPF Network Type Point-to-Multipoint จะทำให้
   WAN-EDGE-1 เป็น Preferred Hub และ WAN-EDGE-2 เป็น Backup ด้วยคำสั่งอะไร?
4. `area 51 stub no-summary` ที่ Hub ทำงานร่วมกับหลักการ NHRP Shortcut ของ Phase 3 อย่างไร
   จนทำให้ Summarize เส้นทางแบบกว้างๆที่ Spoke ได้โดยไม่กระทบ Spoke-to-Spoke Reachability?
5. `ip nhrp group` ที่ Spoke และ `ip nhrp map group ... service-policy` ที่ Hub ทำงานร่วมกัน
   อย่างไรเพื่อให้ QoS ต่อ Spoke Scale ได้โดยไม่ต้องรู้ NBMA Address ของ Spoke ล่วงหน้า?

**เฉลย:**

1. Spoke-A ส่ง Packet แรกผ่าน Hub (Hairpin) เพราะ Routing Protocol บอก Next-Hop เป็น Hub →
   Hub ตรวจ CEF พบ Same-Interface Forwarding (Ingress/Egress เป็น Tunnel0 เดียวกัน) → Hub ส่ง
   NHRP Redirect (Traffic Indication) กลับไปยัง Spoke-A → Spoke-A ส่ง NHRP Resolution Request
   ไปขอ NBMA Address จริงของ Spoke-B จาก Hub (NHRP Server) → ได้ Resolution Reply กลับมา →
   Spoke-A สร้าง Dynamic mGRE/IPsec ตรงถึง Spoke-B และติดตั้ง Shortcut Route (Host Route)
   ผ่าน CEF แทน Route เดิมที่ผ่าน Hub
2. เพราะ Phase 3 แยก "Routing" กับ "Path Optimization" ออกจากกัน — NHRP Redirect/Shortcut
   ทำงานที่ Data-Plane/CEF โดยไม่พึ่ง Next-Hop ที่ Routing Protocol Advertise เลย ในขณะที่
   Phase 2 อาศัย Routing Protocol เป็นตัวบอก Next-Hop ที่ถูกต้องของ Spoke-to-Spoke โดยตรง
   ถ้า Hub ทำ Next-Hop-Self ค่า Next-Hop จริงของ Spoke ปลายทางจะถูกซ่อนไปเสียก่อน
3. ใช้คำสั่ง `neighbor <hub-tunnel-ip> cost <value>` ภายใต้ `router ospf` ที่ Spoke — กำหนด
   Cost ต่ำให้ WAN-EDGE-1 (เช่น 100) และ Cost สูงให้ WAN-EDGE-2 (เช่น 200) ซึ่งใช้ได้เฉพาะกับ
   Network Type Point-to-Multipoint ที่รองรับการกำหนด Cost แยกตาม Neighbor บน Interface เดียว
4. เพราะ Phase 3 ไม่จำเป็นต้องให้ Spoke รู้ Route เฉพาะเจาะจงของปลายทางเพื่อให้ Spoke-to-Spoke
   ทำงาน — เพียง Packet แรกไปถึง Hub ได้ (ผ่าน Default/Summary Route ก็พอ) Hub ก็จะตรวจพบและ
   ส่ง Redirect ให้ Spoke ไปขอ Shortcut เอง ซึ่ง Shortcut ที่ได้เป็น Host Route (`/32`) ที่
   แคบกว่า Summary เสมอ ทำให้ CEF เลือก Shortcut เป็นหลักโดยอัตโนมัติหลังจากนั้น — Summarize
   จึงไม่กระทบ Reachability ของ Spoke-to-Spoke เลย
5. Spoke ประกาศ Group ของตัวเองด้วย `ip nhrp group <name>` เมื่อ Register NHRP เข้ามาที่ Hub —
   Hub จะจับคู่ Group Name นั้นกับ Service-Policy ที่กำหนดไว้ล่วงหน้าด้วย `ip nhrp map group
   <name> service-policy output <policy>` แล้ว Apply Policy (เช่น Shape Rate ตาม Contract
   ของสาขา) ให้ Session ของ Spoke ตัวนั้นโดยอัตโนมัติ โดย Hub ไม่ต้องรู้ NBMA Address ของ
   Spoke ล่วงหน้าเลย — เพิ่ม Spoke ใหม่แค่กำหนด Group ที่ถูกต้องก็ได้ SLA ที่เหมาะสมทันที

---

## สรุป Part 53

Part นี้ยกระดับ DMVPN Lab จาก Phase 2 (Part 52) ไปสู่ **Phase 3** ด้วย `ip nhrp redirect` และ
`ip nhrp shortcut` ที่ทำให้ Spoke-to-Spoke Traffic ทำงานได้จริงโดยไม่ผูกกับข้อจำกัดของ Routing
Protocol อีกต่อไป (พิสูจน์ด้วยการเปลี่ยนจาก EIGRP เป็น **OSPF Point-to-Multipoint**) จากนั้น
ขยายเป็น **Dual-Hub Design** ด้วย WAN-EDGE-2 เป็น Hub สำรอง เปรียบเทียบ Single-Cloud กับ
Dual-Cloud, ปรับ OSPF Cost ให้ Preferred Hub ชัดเจน, แนะนำ FlexVPN เป็นทางเลือกของชั้น IKE, และ
ปิดท้ายด้วย **Hierarchical QoS ต่อ Spoke ผ่าน NHRP Group** พร้อม Lab เต็มรูปแบบที่พิสูจน์ทั้ง
Shortcut และ Hub Failover จริง ✅ **พร้อมสำหรับ Part 54**: เราจะออกจากโลกของ WAN/VPN ไปสู่
**Network Assurance & Telemetry** — การมอนิเตอร์เครือข่ายยุคใหม่ด้วย Streaming Telemetry,
Cisco DNA Center Assurance, และการวิเคราะห์ Baseline/Anomaly Detection ที่ต่อยอดจากพื้นฐาน
SNMP/Syslog ของ [Part 17](part-017-ntp-syslog-snmp.md)

**ไปต่อ:** [Part 54 — Network Assurance & Telemetry →](part-054-network-assurance-telemetry.md)
