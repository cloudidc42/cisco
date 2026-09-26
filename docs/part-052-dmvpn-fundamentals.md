# Part 52 — DMVPN Fundamentals (Dynamic Multipoint VPN)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 511–520 จาก 1000**

> ต่อจาก [Part 51 — VRF-Lite, GRE-over-IPsec & VTI](part-051-vrf-lite-gre-ipsec.md) ที่สอนการสร้าง
> **Site-to-Site VPN แบบ Point-to-Point** ด้วย GRE-over-IPsec และ VTI (Virtual Tunnel Interface)
> ระหว่าง **WAN-EDGE-1** กับ **BRANCH-RTR** ไปแล้ว Part นี้จะตอบคำถามที่ตามมาโดยธรรมชาติ: ถ้า
> องค์กรมี Site สาขา**หลายสิบหลายร้อยแห่ง** การคอนฟิก GRE Tunnel คู่ต่อคู่แบบ Part 51 ทีละ Site
> จะกลายเป็นปัญหาด้าน Scale ทันที — **DMVPN (Dynamic Multipoint VPN)** คือคำตอบที่ Cisco ออกแบบ
> มาเพื่อแก้ปัญหานี้โดยเฉพาะ
>
> Part นี้จะแนะนำอุปกรณ์จำลองใหม่อีก 1 ตัวคือ **BRANCH2-RTR** — Router สาขาที่สองที่เชื่อมต่อ
> Internet ผ่าน ISP คนละเจ้ากับ BRANCH-RTR (Part 19) เพื่อให้ Topology เป็น **Hub-and-Spoke ที่มี
> มากกว่า 1 Spoke อย่างสมเหตุสมผล** (1 Hub + 2 Spoke) ซึ่งจำเป็นต่อการสอนแนวคิด Spoke-to-Spoke
> ของ DMVPN Phase 2 ให้เห็นภาพจริง — ยังคงเป็นอุปกรณ์จำลองสำหรับ Scenario WAN/VPN เท่านั้น
> ไม่ได้อยู่ใน Core Campus Topology หลักที่ประกาศใน Part 1

## Topology ของ Part นี้

```
                              ┌───────────────────┐
                              │      ISP-RTR        │  (จำลอง Internet Backbone)
                              └──┬────────┬────────┘
                    203.0.113.0/30       198.51.100.0/30 , 198.51.100.4/30
                                  │                │
                    ┌─────────────┴───┐    ┌───────┴──────────┐    ┌──────────────────┐
                    │   WAN-EDGE-1     │    │    BRANCH-RTR     │    │   BRANCH2-RTR     │
                    │  (HUB / NHS)     │    │   (SPOKE 1)       │    │   (SPOKE 2)       │
                    │  Gi0/0/0:        │    │  Gi0/0/1:         │    │  Gi0/0/1:         │
                    │  203.0.113.2/30  │    │  198.51.100.2/30  │    │  198.51.100.6/30  │
                    │  Lo0: 1.1.1.21/32│    │  Lo0:172.20.0.1/32│    │  Lo0:172.20.1.1/32│
                    └─────────┬────────┘    └─────────┬─────────┘    └─────────┬─────────┘
                              │                        │                        │
                              │       Tunnel10 — mGRE DMVPN Cloud (172.16.201.0/24)
                              │ .1                     │ .2                     │ .3
                              └────────────┬───────────┴────────────┬───────────┘
                                           NHRP Network-ID 100 / EIGRP AS 200
                                    (Phase 2: Spoke ↔ Spoke สร้าง Tunnel ตรงกันได้เอง)
```

### สรุป IP Addressing ของ DMVPN Lab นี้

| Device | Role | NBMA (Physical/Real) IP | Tunnel10 IP (DMVPN) | Loopback0 (จำลอง LAN/Router-ID) |
|---|---|---|---|---|
| WAN-EDGE-1 | Hub / NHS (NHRP Server) | 203.0.113.2 (Gi0/0/0 ไปยัง ISP-RTR — ค่าเดิมจาก [00-ip-address-plan.md](00-ip-address-plan.md)) | 172.16.201.1/24 | 1.1.1.21/32 (Router-ID เดิม) |
| BRANCH-RTR | Spoke 1 | 198.51.100.2 (Gi0/0/1 — ค่าเดิมจาก [Part 19](part-019-wan-technologies.md)) | 172.16.201.2/24 | 172.20.0.1/32 (จำลอง LAN สาขา 1) |
| BRANCH2-RTR | Spoke 2 (**ใหม่ใน Part นี้**) | 198.51.100.6 (Gi0/0/1, จำลอง Internet ผ่าน ISP อีกเจ้า) | 172.16.201.3/24 | 172.20.1.1/32 (จำลอง LAN สาขา 2) |

> **หมายเหตุ**: NBMA ของ WAN-EDGE-1 และ BRANCH-RTR ใช้ค่าเดิมทุกประการจาก Part 19/51 (Physical
> Interface เดิม, IP เดิม) — Part นี้เพียงแค่**เปลี่ยนวิธีสร้าง Tunnel** จาก Point-to-Point GRE
> ไปเป็น DMVPN mGRE เท่านั้น ไม่แตะ Physical Layer เดิมแม้แต่บรรทัดเดียว ส่วน BRANCH2-RTR เป็น
> Router สาขาใหม่ที่ประกาศครั้งแรกใน Part นี้ ใช้ Subnet `198.51.100.4/30` (ใหม่ ไม่ชนกับ
> `198.51.100.0/30` ของ BRANCH-RTR)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 511 | ปัญหาการ Scale ของ GRE Full-Mesh — ทำไมต้องมี DMVPN |
| 512 | เทคโนโลยีหลักของ DMVPN — mGRE, NHRP, Dynamic IPsec |
| 513 | DMVPN Phase 1 — Hub-and-Spoke (Spoke-to-Spoke ผ่าน Hub เท่านั้น) |
| 514 | NHRP Registration Process — Spoke ลงทะเบียน NBMA IP กับ Hub |
| 515 | DMVPN Phase 2 — mGRE ทั้ง Hub และ Spoke, เปิดทาง Spoke-to-Spoke ตรง |
| 516 | Routing over DMVPN — ทำไมต้องใช้ Dynamic Routing Protocol (EIGRP) |
| 517 | ข้อจำกัด Phase 1 vs Phase 2 — Split-Horizon และทางไปสู่ Phase 3 |
| 518 | เพิ่ม IPsec ให้ DMVPN — `tunnel protection ipsec profile` |
| 519 | Verification/Troubleshooting — `show dmvpn`, `show ip nhrp`, ปัญหาที่พบบ่อย |
| 520 | Lab เต็มรูปแบบ: DMVPN Phase 2 + IPsec + EIGRP บน 1 Hub + 2 Spoke |

---

## Step 511 — ปัญหาการ Scale ของ GRE Full-Mesh

### ทบทวนจาก Part 51: GRE Tunnel คู่ต่อคู่ต้องคอนฟิกทีละคู่

Part 51 สอนการสร้าง GRE-over-IPsec/VTI ระหว่าง **WAN-EDGE-1 ↔ BRANCH-RTR** เพียงคู่เดียว — ทำงาน
ได้ดีเมื่อมี Site น้อย แต่ปัญหาจะปรากฏทันทีที่องค์กรขยายตัวและต้องการให้ **ทุก Site คุยกันได้แบบ
Full-Mesh** (Site A ↔ B, A ↔ C, B ↔ C, ...) เพราะ GRE เป็น Protocol แบบ **Point-to-Point ล้วนๆ**
— หนึ่ง Tunnel Interface ผูกกับปลายทางเดียวเท่านั้น (`tunnel destination` ต้องเป็น IP เดียว)

### สูตรคณิตศาสตร์ที่พิสูจน์ปัญหา: N(N-1)/2

ถ้ามี **N Site** ที่ต้องการ Full-Mesh กันทุกคู่ จำนวน Tunnel ที่ต้องคอนฟิกทั้งหมดคือ:

```
จำนวน Tunnel ที่ต้องคอนฟิกทั้งหมด = N(N-1) / 2
```

| จำนวน Site (N) | จำนวน Tunnel ที่ต้องคอนฟิก (N(N-1)/2) | จำนวน Tunnel Interface ต่อ Router (N-1) |
|---|---|---|
| 3 | 3 | 2 |
| 5 | 10 | 4 |
| 10 | 45 | 9 |
| 50 | 1,225 | 49 |
| 100 | 4,950 | 99 |

```
Full-Mesh GRE แบบ Manual เมื่อ N=4 (HQ + 3 Branch) — ต้องมี 6 Tunnel แยกกัน:

        HQ ═══════════════ Branch1
        ║  ╲              ╱   ║
        ║    ╲          ╱     ║
        ║      ╲      ╱       ║
        ║        ╲  ╱         ║
      Branch3 ═══ ╳ ═══ Branch2
         (แต่ละ ═/║/╲/╱ คือ Tunnel Interface ที่ต้องคอนฟิกเอง 1 คู่ tunnel source/destination)
```

### ผลกระทบที่แท้จริงในทางปฏิบัติ ไม่ใช่แค่จำนวน Tunnel

- **เมื่อเพิ่ม Site ใหม่ 1 แห่ง** ต้องไปแก้ Config **ทุก Router ที่มีอยู่เดิมทั้งหมด** (เพิ่ม Tunnel
  Interface ใหม่ชี้ไปที่ Site ใหม่) — ไม่ใช่แค่คอนฟิกที่ Site ใหม่ Site เดียว
- **NBMA (Real/Physical) IP ของ Spoke อาจเปลี่ยนได้** (เช่น Branch ใช้ DHCP จาก ISP) — GRE
  Point-to-Point ต้องระบุ `tunnel destination` แบบ Static เท่านั้น เปลี่ยน IP เมื่อไหร่ Tunnel ล่ม
  ทันทีจนกว่าจะเข้าไปแก้ Config มือ
- **Routing Table โตตามจำนวน Tunnel** และการ Design Routing Protocol ให้ Scale (Split-Horizon,
  Summarization) ยิ่งซับซ้อนขึ้นเรื่อยๆ ตามจำนวน Neighbor ที่เพิ่ม

### DMVPN แก้ปัญหานี้อย่างไร (ภาพรวมก่อนลงรายละเอียด)

**DMVPN (Dynamic Multipoint VPN)** เปลี่ยนแนวคิดจาก "คอนฟิก Tunnel คู่ต่อคู่ล่วงหน้า" มาเป็น
**"Hub คอนฟิกครั้งเดียว รองรับ Spoke ได้ไม่จำกัดจำนวนโดยไม่ต้องแก้ Config Hub อีก"** และ
**"Spoke ค้นหาเส้นทางไปยัง Spoke อื่นแบบ Dynamic เมื่อมี Traffic จริงเกิดขึ้น"** — ไม่ต้องรู้ IP
ของ Spoke อื่นล่วงหน้าเลย เมื่อเพิ่ม Site ใหม่แค่คอนฟิกที่ Site ใหม่ฝั่งเดียว แล้ว "ลงทะเบียน" กับ
Hub เท่านั้น — จำนวน Tunnel ที่ต้อง**คอนฟิกด้วยมือ** ลดจาก N(N-1)/2 เหลือแค่ **N-1** (Spoke แต่ละตัว
คอนฟิกแค่ 1 ครั้งชี้ไปที่ Hub) ส่วน Tunnel ระหว่าง Spoke-to-Spoke ที่เหลือ **ระบบสร้างให้อัตโนมัติ**

---

## Step 512 — เทคโนโลยีหลักของ DMVPN: mGRE, NHRP, Dynamic IPsec

DMVPN ไม่ใช่ Protocol ตัวใหม่ตัวเดียว แต่เป็น**การผสาน 3 เทคโนโลยี**เข้าด้วยกันเพื่อให้เกิด
พฤติกรรม "Dynamic Multipoint" ที่ GRE ธรรมดาทำไม่ได้:

| เทคโนโลยี | บทบาท | เปรียบเทียบให้เข้าใจง่าย |
|---|---|---|
| **mGRE** (Multipoint GRE) | Tunnel Interface **เดียว** แทน Logical GRE Tunnel ได้**หลายเส้น**พร้อมกัน โดยไม่ต้องระบุ `tunnel destination` ตายตัว | เหมือน Switchport เดียวที่คุยกับหลาย MAC Address ได้ (ต่างจาก P2P GRE ที่เหมือนสาย Crossover เชื่อมได้แค่ 2 ปลาย) |
| **NHRP** (Next Hop Resolution Protocol) | Protocol ที่ทำหน้าที่ **Map ระหว่าง Tunnel IP (Virtual) กับ NBMA IP (Real/Physical)** ของ Router แต่ละตัวในวง DMVPN | **"DNS สำหรับ Tunnel"** — เหมือนที่ DNS แปลงชื่อ Domain เป็น IP จริง, NHRP แปลง Tunnel IP (เช่น 172.16.201.2) เป็น NBMA IP จริง (เช่น 198.51.100.2) ให้ mGRE รู้ว่าจะห่อหุ้ม Packet ส่งไปที่ IP จริงไหน |
| **Dynamic IPsec** | สร้าง IPsec SA แบบไม่ต้องรู้ Peer IP ล่วงหน้า (ใช้ Wildcard Pre-shared Key + Tunnel Protection) เพราะ Spoke ใหม่อาจมี NBMA IP ที่ไม่รู้ล่วงหน้า | ต่างจาก Crypto Map แบบเดิมที่ต้องระบุ `set peer <IP>` ตายตัวทีละ Peer — DMVPN ผูก IPsec Profile ไว้ที่ Tunnel Interface แทน แล้วให้ครอบคลุมทุก Peer ที่ NHRP Resolve มาได้โดยอัตโนมัติ |

### mGRE ในภาพเปรียบเทียบกับ P2P GRE

```
P2P GRE (Part 19/51):                    mGRE (DMVPN):
Tunnel0 ── tunnel destination A ─── Peer A     Tunnel10 (mGRE, ไม่มี tunnel destination ตายตัว)
Tunnel1 ── tunnel destination B ─── Peer B         │
Tunnel2 ── tunnel destination C ─── Peer C         ├──── Peer A (NHRP resolve ได้)
   (ต้องมี 1 Tunnel Interface ต่อ 1 Peer)          ├──── Peer B (NHRP resolve ได้)
                                                    └──── Peer C (NHRP resolve ได้)
                                        (1 Tunnel Interface เดียว คุยกับ Peer ได้ไม่จำกัดจำนวน)
```

### NHRP: บทบาท NHS (NHRP Server) และ NHC (NHRP Client)

| Role | อยู่ที่ Device ไหนใน Lab นี้ | หน้าที่ |
|---|---|---|
| **NHS** (Next Hop Server) | WAN-EDGE-1 (Hub) | เก็บ **NHRP Cache** ของทุก Spoke ที่ลงทะเบียนเข้ามา (Tunnel IP ↔ NBMA IP) — เปรียบเสมือน **DNS Server** ตัวหลักของวง DMVPN |
| **NHC** (Next Hop Client) | BRANCH-RTR, BRANCH2-RTR (Spoke) | ส่ง **NHRP Registration Request** ไปลงทะเบียนกับ NHS ตอน Tunnel Up และส่ง **NHRP Resolution Request** เมื่อต้องการรู้ NBMA IP ของ Spoke อื่น (ใน Phase 2) |

### ทำไม Dynamic IPsec ถึงจำเป็นสำหรับ DMVPN โดยเฉพาะ

IPsec แบบดั้งเดิมที่ใช้ Crypto Map (`set peer <IP>`) **ต้องรู้ IP ของ Peer ล่วงหน้าเสมอ** — ใช้ไม่ได้
กับ DMVPN เพราะ Spoke ใหม่ที่จะเข้าร่วมวงในอนาคตยังไม่มี IP ให้รู้ล่วงหน้า และ Spoke-to-Spoke
Tunnel ใน Phase 2 ก็เกิดขึ้นแบบ Dynamic เช่นกัน วิธีแก้คือใช้ **Tunnel Protection** (`tunnel
protection ipsec profile`) ผูก IPsec Profile ไว้กับ Tunnel Interface โดยตรง (ไม่ผูกกับ Peer IP)
ร่วมกับ **Wildcard Pre-shared Key** (`crypto isakmp key ... address 0.0.0.0 0.0.0.0`) เพื่อให้
Peer ที่มี NBMA IP อะไรก็ได้สามารถต่อ IPsec เข้ามาได้ — รายละเอียดคอนฟิกเต็มรูปแบบอยู่ใน Step 518

---

## Step 513 — DMVPN Phase 1: Hub-and-Spoke (Spoke-to-Spoke ผ่าน Hub เท่านั้น)

### หลักการของ Phase 1

**DMVPN Phase 1** คือรูปแบบเริ่มต้นที่ง่ายที่สุด: **mGRE ใช้เฉพาะที่ Hub เท่านั้น** ส่วน Spoke
ยังคงใช้ **Point-to-Point GRE แบบเดิม** (ชี้ `tunnel destination` ไปที่ Hub ตายตัว) — ผลคือ
Spoke ทุกตัวคุยกับ Hub ได้โดยตรง แต่ **Spoke-to-Spoke ต้อง Hairpin ผ่าน Hub เสมอ** (Traffic
จาก BRANCH-RTR ไปยัง BRANCH2-RTR ต้องวิ่งเข้า Hub ก่อนแล้วจึงส่งต่อออกไป ไม่มีทางลัดตรง)

```
DMVPN Phase 1 — Spoke-to-Spoke ต้องผ่าน Hub เสมอ:

              BRANCH-RTR (Spoke1)  ←──── P2P GRE ────→  WAN-EDGE-1 (Hub, mGRE)
                    │                                          │  ▲
                    │         Traffic Spoke1→Spoke2            │  │
                    └──────────────────────────────────────────┘  │
                                    (ต้อง Hairpin ผ่าน Hub)         │
              BRANCH2-RTR (Spoke2) ←──── P2P GRE ─────────────────┘

  * Hub: mGRE (Tunnel10 ไม่มี tunnel destination ตายตัว, รับได้หลาย Spoke)
  * Spoke: ยังเป็น P2P GRE ปกติ (tunnel destination = Hub NBMA เท่านั้น)
```

- ข้อดี: ง่ายที่สุด ใกล้เคียง Config GRE เดิมที่เรียนใน Part 19/51 มากที่สุด, Hub ควบคุม/มองเห็น
  Traffic ทุกเส้นทางได้ 100% (เหมาะกับ Security Policy ที่ต้อง Inspect ทุก Traffic ที่ Hub)
- ข้อเสีย: **Hub กลายเป็นคอขวด (Bottleneck)** ทั้ง Bandwidth และ CPU เมื่อ Spoke คุยกันเยอะ
  (Voice/Video ระหว่างสาขาต้องเดินทาง 2 เท่าของระยะทางจริง)

### คอนฟิก Phase 1 เต็มรูปแบบ — WAN-EDGE-1 (Hub)

```
WAN-EDGE-1(config)# interface Tunnel10
WAN-EDGE-1(config-if)# description ** DMVPN Hub - mGRE Cloud (Phase 1) **
WAN-EDGE-1(config-if)# ip address 172.16.201.1 255.255.255.0
WAN-EDGE-1(config-if)# no ip redirects
WAN-EDGE-1(config-if)# ip nhrp authentication DMVPN-KEY
WAN-EDGE-1(config-if)# ip nhrp map multicast dynamic
WAN-EDGE-1(config-if)# ip nhrp network-id 100
WAN-EDGE-1(config-if)# ip nhrp holdtime 300
WAN-EDGE-1(config-if)# tunnel source GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# tunnel mode gre multipoint
WAN-EDGE-1(config-if)# tunnel key 5152
WAN-EDGE-1(config-if)# exit
```

- `tunnel mode gre multipoint` = คำสั่งหลักที่เปิดโหมด mGRE (ไม่ระบุ `tunnel destination` เลย
  ต่างจาก P2P GRE ใน Part 19 ที่ต้องมี `tunnel destination` เสมอ)
- `ip nhrp map multicast dynamic` = อนุญาตให้ Hub Replicate Multicast/Broadcast (เช่น EIGRP
  Hello) ไปยังทุก Spoke ที่ลงทะเบียนเข้ามาแบบ Dynamic โดยไม่ต้องระบุ NBMA IP ของ Spoke ล่วงหน้า
- `tunnel key 5152` = แยก GRE Session ออกจากกันในกรณีมีหลาย mGRE Tunnel ใช้ Physical Interface
  เดียวกัน (Best Practice แม้ Lab นี้มี Tunnel เดียว)

### คอนฟิก Phase 1 เต็มรูปแบบ — BRANCH-RTR (Spoke 1, ยังเป็น P2P GRE)

```
BRANCH-RTR(config)# interface Tunnel10
BRANCH-RTR(config-if)# description ** DMVPN Spoke1 - P2P GRE to Hub (Phase 1) **
BRANCH-RTR(config-if)# ip address 172.16.201.2 255.255.255.0
BRANCH-RTR(config-if)# no ip redirects
BRANCH-RTR(config-if)# ip nhrp authentication DMVPN-KEY
BRANCH-RTR(config-if)# ip nhrp map 172.16.201.1 203.0.113.2
BRANCH-RTR(config-if)# ip nhrp map multicast 203.0.113.2
BRANCH-RTR(config-if)# ip nhrp network-id 100
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.201.1
BRANCH-RTR(config-if)# tunnel source GigabitEthernet0/0/1
BRANCH-RTR(config-if)# tunnel destination 203.0.113.2
BRANCH-RTR(config-if)# tunnel key 5152
BRANCH-RTR(config-if)# exit
```

### คอนฟิก Phase 1 เต็มรูปแบบ — BRANCH2-RTR (Spoke 2, ใหม่ใน Part นี้)

```
BRANCH2-RTR(config)# interface GigabitEthernet0/0/1
BRANCH2-RTR(config-if)# description ** Internet Link (จำลอง Public IP สาขา 2) **
BRANCH2-RTR(config-if)# ip address 198.51.100.6 255.255.255.252
BRANCH2-RTR(config-if)# no shutdown
BRANCH2-RTR(config-if)# exit
!
BRANCH2-RTR(config)# interface Loopback0
BRANCH2-RTR(config-if)# description ** จำลอง LAN สาขา 2 **
BRANCH2-RTR(config-if)# ip address 172.20.1.1 255.255.255.255
BRANCH2-RTR(config-if)# exit
!
BRANCH2-RTR(config)# interface Tunnel10
BRANCH2-RTR(config-if)# description ** DMVPN Spoke2 - P2P GRE to Hub (Phase 1) **
BRANCH2-RTR(config-if)# ip address 172.16.201.3 255.255.255.0
BRANCH2-RTR(config-if)# no ip redirects
BRANCH2-RTR(config-if)# ip nhrp authentication DMVPN-KEY
BRANCH2-RTR(config-if)# ip nhrp map 172.16.201.1 203.0.113.2
BRANCH2-RTR(config-if)# ip nhrp map multicast 203.0.113.2
BRANCH2-RTR(config-if)# ip nhrp network-id 100
BRANCH2-RTR(config-if)# ip nhrp nhs 172.16.201.1
BRANCH2-RTR(config-if)# tunnel source GigabitEthernet0/0/1
BRANCH2-RTR(config-if)# tunnel destination 203.0.113.2
BRANCH2-RTR(config-if)# tunnel key 5152
BRANCH2-RTR(config-if)# exit
```

### Verify Phase 1 เบื้องต้น

```
WAN-EDGE-1# show dmvpn brief
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        # Ent --> Number of NHRP entries with same NBMA peer

Interface       Peer NBMA Address  Peer Tunnel Add  State  Attrb
--------------- ------------------ ---------------- ------ -----
Tu10            198.51.100.2       172.16.201.2      UP     D
Tu10            198.51.100.6       172.16.201.3      UP     D

WAN-EDGE-1# ping 172.16.201.2
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/7/11 ms
WAN-EDGE-1# ping 172.16.201.3
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/8/12 ms
```

`Attrb = D` หมายถึง Entry นี้เกิดจากการ**ลงทะเบียนแบบ Dynamic** (NHRP Registration จาก Spoke)
ไม่ใช่จากการพิมพ์ Static Mapping — พิสูจน์ว่า Hub เรียนรู้ Spoke ทั้งสองได้เองโดยไม่ต้องคอนฟิก
ล่วงหน้าที่ Hub แม้แต่บรรทัดเดียว (รายละเอียดกระบวนการนี้อยู่ใน Step 514)

---

## Step 514 — NHRP Registration Process

### ลำดับขั้นตอนที่เกิดขึ้นตอน Tunnel10 Up ที่ฝั่ง Spoke

```
   BRANCH-RTR (Spoke, NHC)                        WAN-EDGE-1 (Hub, NHS)
        │                                                  │
        │  1. NHRP Registration Request                     │
        │     (Tunnel IP=172.16.201.2, NBMA IP=198.51.100.2)│
        ├────────────────────────────────────────────────►  │
        │                                                  │  2. บันทึกลง NHRP Cache
        │                                                  │     (172.16.201.2 ↔ 198.51.100.2)
        │  3. NHRP Registration Reply (Success)             │
        │  ◄────────────────────────────────────────────── ┤
        │                                                  │
   (ทำซ้ำทุก ip nhrp holdtime/2 เพื่อ Refresh ไม่ให้ Entry หมดอายุที่ Hub)
```

- **NHRP Registration Request**: Spoke (NHC) ส่งข้อความบอก Hub (NHS) ว่า "Tunnel IP ของฉันคือ
  172.16.201.2 และ NBMA IP จริงตอนนี้คือ 198.51.100.2" — คำสั่งที่ทำให้เกิดพฤติกรรมนี้คือ
  `ip nhrp nhs 172.16.201.1` (บอก Spoke ว่าใครคือ NHS ที่ต้องลงทะเบียนด้วย)
- **`ip nhrp map 172.16.201.1 203.0.113.2`**: คำสั่ง Static Mapping ที่ Spoke **ต้องรู้ IP ของ
  Hub ล่วงหน้าเสมอ** (Hub เป็นจุดเดียวที่ต้องรู้ IP แน่นอนตายตัว — ต่างจาก Spoke อื่นที่ Resolve
  แบบ Dynamic ได้ใน Phase 2)
- **`ip nhrp network-id 100`**: ต้อง**ตรงกันทุกอุปกรณ์**ในวง DMVPN เดียวกัน (คล้าย VLAN ID —
  ถ้าไม่ตรงกัน NHRP จะไม่ยอมสร้าง Adjacency กันเลย แม้ Tunnel IP จะ Reachable ก็ตาม)
- **Hub ไม่มีการคอนฟิก `ip nhrp map` ชี้ไปที่ Spoke แม้แต่ตัวเดียว** — นี่คือจุดสำคัญที่สุดของ
  Step นี้: Hub เรียนรู้ Mapping ของ Spoke **ทั้งหมดแบบ Dynamic ผ่าน Registration เท่านั้น**

### Verify NHRP ที่ฝั่ง Hub (Dynamic Entries)

```
WAN-EDGE-1# show ip nhrp
172.16.201.2/32 via 172.16.201.2
   Tunnel10 created 00:12:33, expire 00:02:41
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.2
172.16.201.3/32 via 172.16.201.3
   Tunnel10 created 00:08:05, expire 00:06:55
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.6
```

### Verify NHRP ที่ฝั่ง Spoke (Static Entry ของ Hub เท่านั้น)

```
BRANCH-RTR# show ip nhrp
172.16.201.1/32 via 172.16.201.1
   Tunnel10 created 00:12:35, never expire
   Type: static, Flags: used
   NBMA address: 203.0.113.2
```

- **`expire 00:02:41`** ที่ฝั่ง Hub = เวลาที่เหลือก่อน Entry นี้หมดอายุถ้า Spoke ไม่ Refresh
  (ค่า Default `ip nhrp holdtime` = 7200 วินาที, Lab นี้ตั้งไว้ 300 วินาทีเพื่อให้เห็นการหมดอายุ
  ได้ง่ายใน Lab)
- **`Type: static, never expire`** ที่ฝั่ง Spoke = Mapping ของ Hub ที่ Spoke พิมพ์ไว้ตายตัว
  ไม่มีวันหมดอายุ (เพราะ Hub ต้อง Reachable ตลอดเวลา)
- ถ้า `show ip nhrp` ที่ Hub **ไม่เห็น Entry ของ Spoke เลย** = NHRP Registration ล้มเหลว
  (ดูสาเหตุที่พบบ่อยใน Step 519)

---

## Step 515 — DMVPN Phase 2: mGRE ทั้ง Hub และ Spoke

### หลักการของ Phase 2

**DMVPN Phase 2** อัปเกรด Spoke จาก P2P GRE ให้เป็น **mGRE เหมือน Hub** — ผลคือ Spoke สามารถ
สร้าง **Tunnel ตรงถึง Spoke อื่นได้เองแบบ Dynamic** โดยไม่ต้อง Hairpin ผ่าน Hub อีกต่อไป (หลังจาก
Traffic รอบแรกกระตุ้นให้เกิดการ Resolve เท่านั้น)

```
DMVPN Phase 2 — ขั้นตอนสร้าง Spoke-to-Spoke Tunnel แบบ Dynamic:

Step 1: Traffic แรกจาก Spoke1→Spoke2 ยังไม่รู้ NBMA ของ Spoke2 เลย จึงส่งผ่าน Hub ไปก่อน (เหมือน Phase1)

   BRANCH-RTR ────(1. Data Packet + NHRP Resolution Request)────► WAN-EDGE-1 (Hub)
                                                                        │
                                          (2. Hub รู้ NBMA ของ Spoke2 จาก NHRP Cache ที่ลงทะเบียนไว้)
                                                                        │
                              (3. Hub ส่ง NHRP Resolution Reply กลับไปที่ Spoke1 พร้อม NBMA ของ Spoke2)
   BRANCH-RTR ◄─────────────────────────────────────────────────── WAN-EDGE-1

Step 2: Spoke1 มี NBMA ของ Spoke2 แล้ว → สร้าง mGRE Session ตรงไปยัง Spoke2 ทันที (Dynamic)

   BRANCH-RTR ═══════════════ Direct Spoke-to-Spoke mGRE Tunnel ═══════════════ BRANCH2-RTR
                     (Traffic รอบต่อไปวิ่งตรง ไม่ผ่าน Hub อีกต่อไป)
```

### คอนฟิก Phase 2 — เปลี่ยนเฉพาะฝั่ง Spoke (Hub คอนฟิกเดิมจาก Step 513 ไม่ต้องแก้)

**BRANCH-RTR (Spoke1) — จาก P2P เปลี่ยนเป็น mGRE:**

```
BRANCH-RTR(config)# interface Tunnel10
BRANCH-RTR(config-if)# description ** DMVPN Spoke1 - mGRE Cloud (Phase 2) **
BRANCH-RTR(config-if)# no tunnel destination
BRANCH-RTR(config-if)# tunnel mode gre multipoint
BRANCH-RTR(config-if)# exit
```

**BRANCH2-RTR (Spoke2) — เปลี่ยนแบบเดียวกัน:**

```
BRANCH2-RTR(config)# interface Tunnel10
BRANCH2-RTR(config-if)# description ** DMVPN Spoke2 - mGRE Cloud (Phase 2) **
BRANCH2-RTR(config-if)# no tunnel destination
BRANCH2-RTR(config-if)# tunnel mode gre multipoint
BRANCH2-RTR(config-if)# exit
```

> **สิ่งที่ไม่เปลี่ยน**: `ip nhrp map 172.16.201.1 203.0.113.2` (ชี้ไปที่ Hub) และ
> `ip nhrp nhs 172.16.201.1` **ยังต้องมีอยู่เหมือนเดิม** — Spoke ยังต้องรู้จัก Hub แบบ Static
> เสมอ เพราะ Hub คือจุดเริ่มต้นของทุก NHRP Resolution Request ที่จะพา Spoke ไปหากันเอง Phase 2
> เปลี่ยนแค่ **ความสามารถในการรับ/ส่ง Tunnel ตรงกับ Spoke อื่น** เท่านั้น ไม่ได้ตัด Hub ออกจาก
> สมการเลย (Hub ยังต้อง Up ตลอดเวลาเพื่อให้ Resolution ทำงานได้)

### Verify Phase 2: ก่อนกับหลังมี Traffic ระหว่าง Spoke

**ก่อนมี Traffic ระหว่าง Spoke (เห็นแค่ Hub-Spoke ปกติ):**

```
BRANCH-RTR# show dmvpn
Interface       Peer NBMA Address  Peer Tunnel Add  State  Attrb
--------------- ------------------ ---------------- ------ -----
Tu10            203.0.113.2        172.16.201.1      UP     S
```

**หลัง Ping จาก BRANCH-RTR LAN (172.20.0.1) ไปยัง BRANCH2-RTR LAN (172.20.1.1) ครั้งแรก:**

```
BRANCH-RTR# show dmvpn
Interface       Peer NBMA Address  Peer Tunnel Add  State  Attrb
--------------- ------------------ ---------------- ------ -----
Tu10            203.0.113.2        172.16.201.1      UP     S
Tu10            198.51.100.6       172.16.201.3      UP     D
```

Entry ที่สอง (`198.51.100.6 ↔ 172.16.201.3`, `Attrb = D`) คือ**หลักฐานว่า Spoke-to-Spoke Tunnel
เกิดขึ้นแบบ Dynamic สำเร็จ** — เกิดขึ้นเองหลัง Traffic จริงกระตุ้น NHRP Resolution โดยไม่มีการ
คอนฟิกอะไรเพิ่มที่ BRANCH-RTR เลยสักบรรทัด

### เงื่อนไขสำคัญที่มักถูกมองข้าม: Next-Hop ต้องไม่ถูกเปลี่ยนระหว่างทาง

Phase 2 จะทำงานได้ก็ต่อเมื่อ **Hub ไม่เปลี่ยน Next-Hop ของ Route ที่ส่งต่อระหว่าง Spoke**
(ห้ามใช้ `next-hop-self` แบบที่มักทำกับ BGP/EIGRP Hub ทั่วไป) เพราะ Spoke ต้องเห็น **Tunnel IP
ของ Spoke อื่นเป็น Next-Hop จริง** ในตาราง Routing เพื่อกระตุ้นให้เกิด NHRP Resolution Request
ไปหา Spoke ตัวนั้นตรงๆ — ถ้า Hub เปลี่ยน Next-Hop เป็นตัวเอง Traffic จะวิ่งผ่าน Hub ตลอดไปแม้จะ
เปิด Phase 2 ไว้แล้วก็ตาม (ค่า Default ของ EIGRP บน Multipoint Interface ไม่ทำ Next-Hop-Self
อยู่แล้ว จึงไม่ต้องคอนฟิกเพิ่มสำหรับ Lab นี้)

---

## Step 516 — Routing over DMVPN: ทำไมต้องใช้ Dynamic Routing Protocol

### ทำไม Static Route ใช้ไม่ได้ผลกับ DMVPN

Static Route ต้องระบุ Next-Hop หรือ Exit Interface **ตายตัว** — ขัดกับธรรมชาติของ DMVPN ที่:

1. **จำนวน Spoke เพิ่มขึ้นได้เรื่อยๆ แบบ Dynamic** โดย Hub ไม่รู้ล่วงหน้าว่าจะมี Spoke กี่ตัว
   หรือ Subnet LAN ของ Spoke ใหม่คืออะไร — ถ้าใช้ Static ต้องกลับไปแก้ Hub ทุกครั้งที่เพิ่ม Site
   (ย้อนกลับไปเจอปัญหาแบบเดียวกับ Step 511 อีก)
2. **Spoke-to-Spoke Tunnel ใน Phase 2 เกิดขึ้น "หลังจากมี Traffic"** — ไม่มีทางรู้ล่วงหน้าว่า
   Next-Hop ที่จะใช้คือ Tunnel ตรงหรือผ่าน Hub จนกว่า NHRP จะ Resolve เสร็จจริง
3. Spoke อาจมี **NBMA IP ที่เปลี่ยนได้** (DHCP จาก ISP) — Static Route ที่ผูกกับ Interface/
   Next-Hop คงที่จะพังทันทีที่ ISP เปลี่ยน IP ให้

**Dynamic Routing Protocol** (EIGRP หรือ OSPF) แก้ปัญหานี้เพราะเรียนรู้ Neighbor และ Route
ผ่าน Tunnel10 แบบอัตโนมัติ ไม่ต้องรู้จำนวน Spoke หรือ Subnet ล่วงหน้าเลย — Lab นี้เลือก **EIGRP**
เพราะ Cisco แนะนำ EIGRP เป็น IGP ที่ทำงานร่วมกับ DMVPN ได้ดีที่สุดในทางปฏิบัติ (Convergence เร็ว,
รองรับ Summarization/Stub ได้ดี ต่อยอดจากทักษะที่เรียนไปแล้วใน [Part 13](part-013-eigrp-fundamentals.md)
และ [Part 28](part-028-eigrp-advanced.md))

### คอนฟิก EIGRP over DMVPN — WAN-EDGE-1 (Hub, แจก Default Route)

```
WAN-EDGE-1(config)# router eigrp 200
WAN-EDGE-1(config-router)# eigrp router-id 1.1.1.21
WAN-EDGE-1(config-router)# network 172.16.201.0 0.0.0.255
WAN-EDGE-1(config-router)# network 1.1.1.21 0.0.0.0
WAN-EDGE-1(config-router)# no auto-summary
WAN-EDGE-1(config-router)# exit
!
! Hub ไม่แจก Route ปลีกย่อยของ Campus ทั้งหมดเข้าไปใน DMVPN Cloud
! แต่แจกแค่ Default Route แทน — ให้ Spoke ส่ง Traffic ทุกอย่างที่ไม่รู้จักมาที่ Hub เป็นทางออกเดียว
WAN-EDGE-1(config)# interface Tunnel10
WAN-EDGE-1(config-if)# ip summary-address eigrp 200 0.0.0.0 0.0.0.0
WAN-EDGE-1(config-if)# exit
```

### คอนฟิก EIGRP over DMVPN — BRANCH-RTR / BRANCH2-RTR (Spoke, แจก LAN ของตัวเอง)

```
BRANCH-RTR(config)# router eigrp 200
BRANCH-RTR(config-router)# eigrp router-id 172.20.0.1
BRANCH-RTR(config-router)# network 172.16.201.0 0.0.0.255
BRANCH-RTR(config-router)# network 172.20.0.1 0.0.0.0
BRANCH-RTR(config-router)# no auto-summary
BRANCH-RTR(config-router)# exit
```

```
BRANCH2-RTR(config)# router eigrp 200
BRANCH2-RTR(config-router)# eigrp router-id 172.20.1.1
BRANCH2-RTR(config-router)# network 172.16.201.0 0.0.0.255
BRANCH2-RTR(config-router)# network 172.20.1.1 0.0.0.0
BRANCH2-RTR(config-router)# no auto-summary
BRANCH2-RTR(config-router)# exit
```

> ใช้ **EIGRP AS 200** แยกจาก **AS 100** ที่เป็น Standalone Demo Topology ของ Part 13/28
> โดยเจตนา — DMVPN Cloud นี้เป็น WAN Scenario แยกส่วนกันคนละบริบทกับ EIGRP Demo เดิม

### Verify Routing ที่ฝั่ง Spoke

```
BRANCH-RTR# show ip route eigrp
D*   0.0.0.0/0 [90/26880256] via 172.16.201.1, 00:04:12, Tunnel10
D    172.20.1.1/32 [90/28288000] via 172.16.201.3, 00:00:42, Tunnel10
```

- `0.0.0.0/0` เรียนจาก Hub (`172.16.201.1`) = Default Route ที่ Hub Summarize ไว้ตาม Step นี้
- `172.20.1.1/32` (LAN ของ BRANCH2-RTR) เรียนจาก **`172.16.201.3` โดยตรง** (Tunnel IP ของ
  BRANCH2-RTR เอง ไม่ใช่ของ Hub) — พิสูจน์ว่า Routing Protocol มองเห็น Next-Hop เป็น Spoke อีกตัว
  ตรงๆ ตามเงื่อนไขที่อธิบายไว้ท้าย Step 515

---

## Step 517 — ข้อจำกัด Phase 1 vs Phase 2: Split-Horizon และทางไปสู่ Phase 3

### ปัญหา Split-Horizon บน Multipoint Interface

**EIGRP Split-Horizon Rule (ค่า Default)**: Router จะ**ไม่ Advertise Route กลับออกไปยัง
Interface เดียวกันกับที่เรียนรู้ Route นั้นเข้ามา** — กฎนี้ใช้ได้ดีบน P2P Link เสมอ (Route ที่
เรียนจาก Peer ฝั่งหนึ่งไม่มีประโยชน์ที่จะส่งกลับไปทาง Interface เดิม) **แต่ Tunnel10 บน Hub เป็น
Multipoint Interface** ที่มี Spoke หลายตัวอยู่หลัง Interface เดียวกัน — ผลคือ:

```
ปัญหา: Hub เรียน Route ของ BRANCH-RTR (172.20.0.1/32) เข้ามาทาง Tunnel10
       แล้วต้อง Advertise Route นี้ต่อไปให้ BRANCH2-RTR — แต่ BRANCH2-RTR ก็อยู่หลัง Tunnel10
       เดียวกัน! Split-Horizon จะ**บล็อกการ Advertise นี้ไว้** ทำให้ BRANCH2-RTR ไม่เห็น
       Route ของ BRANCH-RTR เลย แม้ Hub จะรู้ Route นั้นแล้วก็ตาม
```

### วิธีแก้: ปิด Split-Horizon บน Tunnel10 ของ Hub

```
WAN-EDGE-1(config)# interface Tunnel10
WAN-EDGE-1(config-if)# no ip split-horizon eigrp 200
WAN-EDGE-1(config-if)# exit
```

- คำสั่งนี้ **ต้องใส่ที่ Hub เท่านั้น** (จุดที่ทำหน้าที่ "ส่งต่อ" Route ระหว่าง Spoke สองตัวผ่าน
  Interface เดียวกัน) — Spoke ไม่จำเป็นต้องปิด Split-Horizon เพราะ Spoke มี Neighbor ตัวเดียว
  คือ Hub ผ่าน Tunnel10 เท่านั้น (ในกรณี Phase 1) ไม่มีสถานการณ์ต้อง Relay Route ระหว่าง
  Neighbor สองตัวผ่าน Interface เดียวกันแบบ Hub

### สรุปเปรียบเทียบ Phase 1 vs Phase 2

| ประเด็น | Phase 1 | Phase 2 |
|---|---|---|
| mGRE | Hub เท่านั้น | Hub + Spoke ทุกตัว |
| Spoke-to-Spoke Traffic | ผ่าน Hub เสมอ (Hairpin) | ตรงถึงกันได้ (หลัง NHRP Resolve) |
| Hub เป็นคอขวด Bandwidth | ใช่ (ทุก Traffic วิ่งผ่าน) | น้อยลงมาก (Traffic ตรงหลัง Resolve) |
| Routing ที่ Hub Summarize ได้ไหม | ทำได้เต็มที่ (Spoke ไม่ต้องรู้ Route ของ Spoke อื่นเลย เพราะยังไงก็ต้องผ่าน Hub) | **ทำได้จำกัด** — ต้องปล่อย Route เฉพาะเจาะจงของ Spoke อื่นผ่านไปด้วย (ต้อง `no ip split-horizon`) เพื่อให้ Next-Hop ถูกต้อง |
| ความซับซ้อนของ Config | ต่ำ | สูงขึ้น (ต้องดูแล Split-Horizon) |
| CEF/Switching Path ตอน Traffic เปลี่ยนจาก Hub ไป Spoke ตรง | N/A | ต้องรอ NHRP Resolve และปรับ CEF ใหม่ (มี Delay เล็กน้อยตอนเปลี่ยน Path) |

### ทำไมยังไม่จบแค่นี้ — สิ่งที่ Phase 2 ยังทำได้ไม่สมบูรณ์

Phase 2 แก้ปัญหา Hairpin ได้ แต่ยังมีข้อจำกัดที่สำคัญ 2 อย่างที่ยังไม่ลงตัว:

1. **Hub ไม่สามารถ Summarize Route ของ Spoke ได้อีกต่อไป** เพราะต้องคง Next-Hop เดิมของ Spoke
   ไว้เสมอ (Route ที่ Advertise ต่อต้องเป็น Specific Route ของ Spoke ต้นทางเท่านั้น) — เมื่อมี
   Spoke จำนวนมาก Routing Table ของทุก Spoke จะโตตามจำนวน Spoke ทั้งหมดในวง DMVPN
2. **Spoke ยังต้องรอ Traffic รอบแรก "เจ็บตัว" ผ่าน Hub ก่อนเสมอ** ก่อนจะมี Direct Tunnel
   (Sub-optimal Routing ชั่วขณะ) และ NHRP Resolution ต้องพึ่งพา Routing Protocol เป็นตัวกระตุ้น
   (ถ้า Design Routing ไม่ดี Resolution อาจไม่เกิดขึ้นเลย)

**DMVPN Phase 3** (สอนใน [Part 53 — DMVPN Advanced](part-053-dmvpn-advanced.md)) แก้ทั้งสอง
ข้อจำกัดนี้ด้วยแนวคิด **NHRP Redirect/Shortcut** — Hub สามารถ Summarize Route ได้เต็มที่เหมือน
Phase 1 (ลด Routing Table Spoke ได้จริง) ในขณะที่ยังคงความสามารถสร้าง Spoke-to-Spoke Tunnel
โดยตรงแบบ Phase 2 ไว้ได้ครบ — ผ่านกลไก Hub ส่ง **NHRP Traffic Indication** บอก Spoke ต้นทางว่า
"มี Shortcut ที่ดีกว่าไปทาง Spoke ปลายทางโดยตรง" แทนที่จะพึ่งพา Routing Protocol อย่างเดียว

---

## Step 518 — เพิ่ม IPsec ให้ DMVPN Cloud

### ทำไม DMVPN ยังต้องมี IPsec

เหมือนที่ Part 51 อธิบายไว้กับ GRE ธรรมดา — **mGRE เพียงอย่างเดียวไม่เข้ารหัสข้อมูล** Traffic
ทั้งหมดที่วิ่งผ่าน Tunnel10 (รวม EIGRP Hello/Update) จะเป็น Cleartext บน Internet สาธารณะถ้าไม่
เพิ่ม IPsec เข้ามา — DMVPN ใช้ **Tunnel Protection** (ผูก IPsec Profile ที่ Tunnel Interface
โดยตรง) แทน Crypto Map แบบเดิม เพราะ Crypto Map ต้องรู้ Peer IP ตายตัว ใช้กับ Peer ที่เกิดขึ้น
แบบ Dynamic (ทั้ง Spoke ใหม่ที่เข้าร่วมวง และ Spoke-to-Spoke Tunnel ใน Phase 2) ไม่ได้

### คอนฟิก IPsec (เหมือนกันทั้ง 3 อุปกรณ์ ต่างกันแค่ IP ใน `crypto isakmp key`)

**ISAKMP Policy + Wildcard Pre-shared Key (Wildcard เพราะ NBMA IP ของ Spoke ไม่ตายตัว):**

```
WAN-EDGE-1(config)# crypto isakmp policy 10
WAN-EDGE-1(config-isakmp)# encryption aes 256
WAN-EDGE-1(config-isakmp)# hash sha256
WAN-EDGE-1(config-isakmp)# authentication pre-share
WAN-EDGE-1(config-isakmp)# group 14
WAN-EDGE-1(config-isakmp)# exit
!
WAN-EDGE-1(config)# crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
```

> `address 0.0.0.0 0.0.0.0` = Wildcard PSK ที่ยอมรับ Peer ที่มี IP อะไรก็ได้ (จำเป็นสำหรับ
> DMVPN เพราะ Hub ไม่รู้ NBMA IP ของ Spoke ใหม่ล่วงหน้า) — ทุกอุปกรณ์ในวง DMVPN ต้องใช้
> Pre-shared Key **ค่าเดียวกัน** (`Cisco@DMVPN123`) เพราะเป็น Symmetric Secret เหมือนหลักการ
> CHAP ที่เรียนไปแล้วใน [Part 19 Step 184](part-019-wan-technologies.md)

**IPsec Transform-Set + Profile (ใช้ Transport Mode ลด Overhead เพราะ GRE ทำ Encapsulation
ให้แล้วชั้นหนึ่ง):**

```
WAN-EDGE-1(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
WAN-EDGE-1(config-crypto-trans)# mode transport
WAN-EDGE-1(config-crypto-trans)# exit
!
WAN-EDGE-1(config)# crypto ipsec profile DMVPN-PROFILE
WAN-EDGE-1(ipsec-profile)# set transform-set DMVPN-TSET
WAN-EDGE-1(ipsec-profile)# exit
```

**ผูก IPsec Profile เข้ากับ Tunnel10 (คำสั่งเดียวที่ทำให้ mGRE Cloud ทั้งหมดถูกเข้ารหัส):**

```
WAN-EDGE-1(config)# interface Tunnel10
WAN-EDGE-1(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
WAN-EDGE-1(config-if)# exit
```

ต้องทำคำสั่งชุดเดียวกันนี้ (ISAKMP Policy, Wildcard Key, Transform-Set, Profile, `tunnel
protection`) ที่ **BRANCH-RTR และ BRANCH2-RTR ด้วยทุกอุปกรณ์** — Pre-shared Key ต้องตรงกันทั้งวง

### Verify IPsec Session

```
WAN-EDGE-1# show crypto session
Crypto session current status

Interface: Tunnel10
Session status: UP-ACTIVE
Peer: 198.51.100.2 port 500
  IKEv1 SA: local 203.0.113.2/500 remote 198.51.100.2/500 Active
  IPSEC FLOW: permit 47 host 203.0.113.2 host 198.51.100.2

Interface: Tunnel10
Session status: UP-ACTIVE
Peer: 198.51.100.6 port 500
  IKEv1 SA: local 203.0.113.2/500 remote 198.51.100.6/500 Active
  IPSEC FLOW: permit 47 host 203.0.113.2 host 198.51.100.6
```

`IPSEC FLOW: permit 47 ...` = IPsec ทำหน้าที่เข้ารหัส **GRE (Protocol Number 47)** ทั้งก้อน —
ยืนยันว่านี่คือ GRE-over-IPsec แบบ Tunnel Protection ไม่ใช่ IPsec ตรงกับ Traffic ต้นฉบับ
(สอดคล้องกับแนวคิด GRE-over-IPsec ที่วางไว้ตั้งแต่ [Part 19 Step 189](part-019-wan-technologies.md))

---

## Step 519 — Verification / Troubleshooting DMVPN

### คำสั่ง Verify หลักที่ใช้บ่อยที่สุด

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show dmvpn` / `show dmvpn detail` | สถานะ Tunnel ทั้งหมด, Peer NBMA/Tunnel IP, Attrb (S/D/N/I/L/X), IPsec Status ต่อ Peer |
| `show ip nhrp` / `show ip nhrp detail` | NHRP Cache ทั้งหมด (Mapping Tunnel IP ↔ NBMA IP), Static vs Dynamic, เวลาหมดอายุ |
| `show crypto session` / `show crypto session detail` | สถานะ IKE/IPsec SA ต่อ Peer, Encapsulation ที่ถูกป้องกัน (Protocol 47 = GRE) |
| `show crypto isakmp sa` | สถานะ ISAKMP/IKE Phase 1 SA แต่ละคู่ |
| `debug nhrp` / `debug nhrp packet` | ดู NHRP Registration/Resolution Request-Reply แบบ Real-time (ใช้ตอน Troubleshoot เท่านั้น) |

### ตารางปัญหาที่พบบ่อยที่สุดในสภาพแวดล้อมจริง

| อาการ | สาเหตุที่พบบ่อยที่สุด | วิธีแก้ |
|---|---|---|
| `show ip nhrp` ที่ Hub ไม่เห็น Entry ของ Spoke เลย (NHRP Registration ล้มเหลว) | Firewall/ACL ระหว่างทาง (เช่นที่ ISP-RTR หรือ Internet Edge) บล็อก **GRE (Protocol 47)** หรือ **UDP/500 + UDP/4500** (IKE/NAT-T) ไม่ให้ผ่าน | ตรวจ ACL/Firewall Policy ให้อนุญาต Protocol 47 และ UDP 500/4500 ระหว่าง NBMA ของ Hub-Spoke ทุกคู่ |
| Tunnel10 ขึ้น `up/up` แต่ NHRP ไม่ทำงาน, `ip nhrp network-id` ไม่ Error แต่ Adjacency ไม่เกิด | **`ip nhrp network-id` ไม่ตรงกัน** ระหว่าง Hub-Spoke (เช่น Hub ใช้ 100 แต่ Spoke ใช้ 200 โดยไม่ตั้งใจ) | ตรวจให้ `network-id` ตรงกัน**ทุกอุปกรณ์**ในวง DMVPN เดียวกัน |
| Spoke ตัวหนึ่งขึ้น Tunnel ไม่ได้เลย ขณะที่ Spoke อื่นปกติ | **mGRE/P2P Mismatch** — เผลอตั้ง `tunnel destination` ค้างไว้ที่ Spoke ตัวนั้นหลัง Migrate เป็น Phase 2 (ยังเป็น P2P อยู่ ขณะ Hub เป็น mGRE) หรือ Hub ยังไม่ได้เปิด mGRE จริง | `show interface Tunnel10` แล้วดู Tunnel Mode/Destination เทียบทั้งสองฝั่ง ต้องเป็น `tunnel mode gre multipoint` ทั้ง Hub และ Spoke ใน Phase 2, ใช้ `no tunnel destination` ให้ครบ |
| Spoke-to-Spoke ไม่เกิด Direct Tunnel เลยแม้ Ping ข้าม Site หลายรอบแล้ว (ยัง Hairpin ผ่าน Hub ตลอด) | ยังอยู่ใน **Phase 1** (Spoke เป็น P2P GRE) โดยไม่รู้ตัว หรือ Routing Protocol ทำ **Next-Hop-Self** เปลี่ยน Next-Hop ที่ Hub จนบัง NHRP Resolution ไม่ให้เกิด | ยืนยัน `tunnel mode` เป็น multipoint ทั้งสองฝั่ง (Phase 2) และตรวจ `show ip route` ที่ Spoke ว่า Next-Hop ของ Route อีก Spoke เป็น Tunnel IP ของ Spoke นั้นจริง ไม่ใช่ IP ของ Hub |
| IKE/IPsec ไม่ขึ้น, `show crypto isakmp sa` ว่างเปล่า | **Pre-shared Key ไม่ตรงกัน** หรือ ISAKMP Policy (Encryption/Hash/DH Group) ไม่ Match กันทั้งสองฝั่ง | ตรวจ `crypto isakmp key` ให้ค่าตรงกันทุกอุปกรณ์ และ `crypto isakmp policy` ให้ Parameter ตรงกันทั้งหมด (เหมือนหลักการ Match PPP LCP ที่เรียนใน Part 19) |

### ตัวอย่าง Debug NHRP ตอน Registration ล้มเหลว (ACL บล็อก GRE)

```
BRANCH-RTR# debug nhrp packet
NHRP: Send Registration Request via Tunnel10 vrf 0, packet size: 88
NHRP: 65 bytes out Tunnel10
   (ไม่มี Reply กลับมาเลยหลังจากนี้ — timeout ซ้ำๆ)
NHRP: Retransmitting Registration Request (retry 1)
NHRP: Retransmitting Registration Request (retry 2)
```

ถ้าเห็น **Retransmit ซ้ำๆ ไม่มี Reply กลับมาเลย** ให้สงสัยปัญหา Path ระหว่าง NBMA ก่อนเสมอ
(ACL/Firewall บล็อก, NAT ทำ Port/Protocol ผิดเพี้ยน) มากกว่าปัญหาการคอนฟิก NHRP บน Router เอง
เพราะ Syntax ของ `ip nhrp` คำสั่งค่อนข้าง Forgiving (ไม่ Error ตอน Apply ถ้า Argument ผิด)

---

## Step 520 — Lab เต็มรูปแบบ: DMVPN Phase 2 + IPsec + EIGRP (1 Hub + 2 Spoke)

Lab นี้รวมทุกแนวคิดของ Part 52 เข้าด้วยกันเป็น Running-Config ที่สมบูรณ์ทั้ง 3 อุปกรณ์ —
**DMVPN Phase 2** (mGRE ทั้ง Hub-Spoke), **IPsec Tunnel Protection**, และ **EIGRP AS 200** เป็น
Routing Protocol เหนือ DMVPN Cloud

```
                              ┌───────────────────┐
                              │      ISP-RTR        │
                              └──┬────────┬────────┘
                    203.0.113.0/30       198.51.100.0/30 , 198.51.100.4/30
                    ┌─────────────┴───┐    ┌───────┴──────────┐    ┌──────────────────┐
                    │   WAN-EDGE-1     │    │    BRANCH-RTR     │    │   BRANCH2-RTR     │
                    │  HUB / NHS       │    │   SPOKE 1         │    │   SPOKE 2         │
                    └─────────┬────────┘    └─────────┬─────────┘    └─────────┬─────────┘
                              │ .1                     │ .2                     │ .3
                              └────────── Tunnel10 mGRE + IPsec (172.16.201.0/24) ─────────┘
                                          EIGRP AS 200, NHRP network-id 100
```

### Running-Config เต็มรูปแบบ — WAN-EDGE-1 (Hub)

```
hostname WAN-EDGE-1
!
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile DMVPN-PROFILE
 set transform-set DMVPN-TSET
!
interface Loopback0
 description ** Router-ID / Management **
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to ISP-RTR (NBMA / Tunnel Source) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface Tunnel10
 description ** DMVPN Hub - mGRE Cloud (Phase 2) **
 ip address 172.16.201.1 255.255.255.0
 no ip redirects
 no ip split-horizon eigrp 200
 ip nhrp authentication DMVPN-KEY
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp holdtime 300
 ip summary-address eigrp 200 0.0.0.0 0.0.0.0
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 5152
 tunnel protection ipsec profile DMVPN-PROFILE
!
router eigrp 200
 eigrp router-id 1.1.1.21
 network 172.16.201.0 0.0.0.255
 network 1.1.1.21 0.0.0.0
 no auto-summary
```

### Running-Config เต็มรูปแบบ — BRANCH-RTR (Spoke 1)

```
hostname BRANCH-RTR
!
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile DMVPN-PROFILE
 set transform-set DMVPN-TSET
!
interface Loopback0
 description ** จำลอง LAN สาขา 1 **
 ip address 172.20.0.1 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Internet Link (NBMA / Tunnel Source) **
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Tunnel10
 description ** DMVPN Spoke1 - mGRE Cloud (Phase 2) **
 ip address 172.16.201.2 255.255.255.0
 no ip redirects
 ip nhrp authentication DMVPN-KEY
 ip nhrp map 172.16.201.1 203.0.113.2
 ip nhrp map multicast 203.0.113.2
 ip nhrp network-id 100
 ip nhrp nhs 172.16.201.1
 tunnel source GigabitEthernet0/0/1
 tunnel mode gre multipoint
 tunnel key 5152
 tunnel protection ipsec profile DMVPN-PROFILE
!
router eigrp 200
 eigrp router-id 172.20.0.1
 network 172.16.201.0 0.0.0.255
 network 172.20.0.1 0.0.0.0
 no auto-summary
```

### Running-Config เต็มรูปแบบ — BRANCH2-RTR (Spoke 2)

```
hostname BRANCH2-RTR
!
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
 mode transport
!
crypto ipsec profile DMVPN-PROFILE
 set transform-set DMVPN-TSET
!
interface Loopback0
 description ** จำลอง LAN สาขา 2 **
 ip address 172.20.1.1 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Internet Link (NBMA / Tunnel Source) **
 ip address 198.51.100.6 255.255.255.252
 no shutdown
!
interface Tunnel10
 description ** DMVPN Spoke2 - mGRE Cloud (Phase 2) **
 ip address 172.16.201.3 255.255.255.0
 no ip redirects
 ip nhrp authentication DMVPN-KEY
 ip nhrp map 172.16.201.1 203.0.113.2
 ip nhrp map multicast 203.0.113.2
 ip nhrp network-id 100
 ip nhrp nhs 172.16.201.1
 tunnel source GigabitEthernet0/0/1
 tunnel mode gre multipoint
 tunnel key 5152
 tunnel protection ipsec profile DMVPN-PROFILE
!
router eigrp 200
 eigrp router-id 172.20.1.1
 network 172.16.201.0 0.0.0.255
 network 172.20.1.1 0.0.0.0
 no auto-summary
```

### Verification ครบทุกจุด — เริ่มจาก Neighbor และ Routing

```
WAN-EDGE-1# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(200)
H   Address         Interface   Hold Uptime   SRTT   RTO  Q  Seq
1   172.16.201.3    Tu10          12 00:14:02    8    100  0  15
0   172.16.201.2    Tu10          14 00:14:10    7    100  0  22

BRANCH-RTR# show ip route eigrp
D*   0.0.0.0/0 [90/26880256] via 172.16.201.1, 00:14:10, Tunnel10
D    172.20.1.1/32 [90/28288000] via 172.16.201.3, 00:01:15, Tunnel10

BRANCH2-RTR# show ip route eigrp
D*   0.0.0.0/0 [90/26880256] via 172.16.201.1, 00:14:02, Tunnel10
D    172.20.0.1/32 [90/28288000] via 172.16.201.2, 00:01:15, Tunnel10
```

### Verification — DMVPN แสดง Direct Spoke-to-Spoke Tunnel หลัง Traffic จริง

```
BRANCH-RTR# ping 172.20.1.1 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.20.1.1, timeout is 2 seconds:
Packet sent with a source address of 172.20.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 9/13/18 ms

BRANCH-RTR# show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket

Interface       Peer NBMA Address  Peer Tunnel Add  State  Attrb
--------------- ------------------ ---------------- ------ -----
Tu10            203.0.113.2        172.16.201.1      UP     S
Tu10            198.51.100.6       172.16.201.3      UP     D
```

### Verification — สรุปสถานะ IPsec ครบทั้ง 3 คู่ Peer (Hub-Spoke1, Hub-Spoke2, Spoke1-Spoke2)

```
WAN-EDGE-1# show crypto session brief
interface        peer          IKEv1 SA   IPSEC FLOW
Tunnel10          198.51.100.2  Active     permit 47
Tunnel10          198.51.100.6  Active     permit 47

BRANCH-RTR# show crypto session brief
interface        peer          IKEv1 SA   IPSEC FLOW
Tunnel10          203.0.113.2   Active     permit 47
Tunnel10          198.51.100.6  Active     permit 47
```

> **Lab นี้พิสูจน์ครบทุกเป้าหมายของ Part 52**: WAN-EDGE-1 คอนฟิกครั้งเดียวรองรับ Spoke ได้โดยไม่
> ต้องรู้จัก BRANCH2-RTR ล่วงหน้าแม้แต่บรรทัดเดียว (แก้ปัญหา N(N-1)/2 จาก Step 511), NHRP
> ลงทะเบียนและ Resolve อัตโนมัติ (Step 512-515), EIGRP เรียนรู้ Route ผ่าน DMVPN Cloud แบบ
> Dynamic เต็มรูปแบบ (Step 516-517), และ IPsec เข้ารหัสทุก Path รวมถึง Path ตรงระหว่าง Spoke
> ที่เพิ่งเกิดขึ้นเองแบบ Dynamic (Step 518) — เหลือเพียงข้อจำกัดเรื่อง Summarization ที่ Hub
> ซึ่งจะแก้ใน Phase 3 ของ Part 53

---

## แบบฝึกหัดทวนความเข้าใจ Part 52

1. ถ้าองค์กรมี 8 Site ที่ต้องการ Full-Mesh กันด้วย GRE แบบ Manual ตาม Part 51 ต้องคอนฟิก Tunnel
   ทั้งหมดกี่เส้น และ DMVPN ลดจำนวน Tunnel ที่ต้อง**คอนฟิกด้วยมือ**ลงเหลือกี่เส้น?
2. NHRP Registration กับ NHRP Resolution ต่างกันอย่างไร และเกิดขึ้นใน DMVPN Phase ไหนบ้าง
   (Phase 1 เท่านั้น, Phase 2 เท่านั้น, หรือทั้งสอง Phase)?
3. เพราะเหตุใด Spoke ใน DMVPN Phase 2 จึงยังต้องมีคำสั่ง `ip nhrp map` และ `ip nhrp nhs` ชี้ไปที่
   Hub อยู่ ทั้งที่เปลี่ยนมาใช้ `tunnel mode gre multipoint` เหมือน Hub แล้ว?
4. `no ip split-horizon eigrp 200` ต้องใส่ที่อุปกรณ์ไหนใน Lab นี้ และทำไมใส่ที่ Hub เท่านั้นจึง
   เพียงพอ ไม่ต้องใส่ที่ Spoke?
5. DMVPN Phase 2 แก้ปัญหา Hairpin ของ Phase 1 ได้ แต่ยังมีข้อจำกัดอะไรเหลืออยู่ที่ทำให้ต้องมี
   Phase 3 ในเรื่อง Route Summarization ที่ Hub?

**เฉลย:**

1. ต้องคอนฟิก **8×7/2 = 28 Tunnel** แบบ Manual ถ้าใช้ GRE Full-Mesh ตาม Part 51 — DMVPN ลด
   จำนวน Tunnel ที่ต้องคอนฟิกด้วยมือลงเหลือ **N-1 = 7 เส้น** (Spoke แต่ละตัวคอนฟิกแค่ 1 ครั้งชี้
   ไปที่ Hub เท่านั้น ส่วน Tunnel ระหว่าง Spoke-to-Spoke ที่เหลือทั้งหมดระบบสร้างให้อัตโนมัติผ่าน
   NHRP โดยไม่ต้องคอนฟิกเพิ่ม)
2. **NHRP Registration** คือขั้นตอนที่ Spoke ลงทะเบียน Mapping ของตัวเอง (Tunnel IP ↔ NBMA IP)
   กับ Hub (NHS) — เกิดขึ้น**ทุก Phase** ตอน Tunnel Up (ทั้ง Phase 1 และ Phase 2) **NHRP
   Resolution** คือขั้นตอนที่ Router ตัวหนึ่งถาม Hub ว่า NBMA IP ของ Peer อีกตัวคืออะไร เพื่อสร้าง
   Tunnel ตรงถึงกัน — เกิดขึ้น**เฉพาะ Phase 2 เท่านั้น** เพราะ Phase 1 ไม่มี Spoke-to-Spoke Tunnel
   ให้ต้อง Resolve เลย (Spoke คุยกับ Hub ตรงๆ อย่างเดียว)
3. เพราะ Hub เป็นจุดเดียวใน DMVPN Cloud ที่**ต้องรู้จักแบบ Static เสมอ** — ไม่มี Router ตัวใดจะไป
   ถาม Hub ว่า "NBMA ของ Hub คืออะไร" ได้ (จะถามใครถ้าไม่รู้จัก Hub มาก่อน) ดังนั้น Hub จึงเป็น
   ข้อยกเว้นเดียวที่ต้องคอนฟิกแบบ Static เสมอ ไม่ว่าจะ Phase ไหนก็ตาม ส่วน Spoke อื่นๆถึงจะ
   Resolve แบบ Dynamic ได้ในภายหลัง
4. ต้องใส่ที่ **WAN-EDGE-1 (Hub) เท่านั้น** เพราะ Split-Horizon จะเป็นปัญหาเฉพาะจุดที่ Router
   ต้อง **Relay Route ระหว่าง Neighbor สองตัวที่อยู่หลัง Interface Multipoint เดียวกัน** — มีแค่
   Hub เท่านั้นที่อยู่ในสถานการณ์นี้ (มี BRANCH-RTR และ BRANCH2-RTR อยู่หลัง Tunnel10 เดียวกัน)
   ส่วน Spoke มี Neighbor ตัวเดียวคือ Hub ผ่าน Tunnel10 จึงไม่มีสถานการณ์ต้อง Relay ระหว่าง
   Neighbor สองตัวผ่าน Interface เดียวกันเลย
5. Phase 2 ต้องคง **Next-Hop เดิมของ Spoke ต้นทาง** ไว้เสมอเมื่อ Advertise Route ผ่าน Hub (ห้าม
   Next-Hop-Self) เพื่อให้ Spoke ปลายทางเห็น Next-Hop เป็น Tunnel IP จริงและกระตุ้น NHRP
   Resolution ได้ — ผลคือ Hub **ไม่สามารถ Summarize Route ของ Spoke เป็น Route เดียวได้อีก**
   (ต้องส่งต่อ Route แบบ Specific ของ Spoke ต้นทางเสมอ) ทำให้ Routing Table ของทุก Spoke โตตาม
   จำนวน Spoke ทั้งหมดในวง DMVPN — Phase 3 แก้ด้วย NHRP Redirect/Shortcut ที่แยกกลไก "บอกทาง
   ลัด" ออกจากกลไก "Advertise Route" ทำให้ Hub Summarize ได้เต็มที่โดยไม่กระทบความสามารถสร้าง
   Spoke-to-Spoke Tunnel โดยตรง

---

## สรุป Part 52

Part นี้แก้ปัญหาการ Scale ของ GRE Full-Mesh ที่ Part 51 ทิ้งไว้ (N(N-1)/2 Tunnel แบบ Manual)
ด้วย **DMVPN** ซึ่งผสาน 3 เทคโนโลยีเข้าด้วยกัน: **mGRE** (Tunnel Interface เดียวแทน Logical
Tunnel ได้หลายเส้น), **NHRP** ("DNS สำหรับ Tunnel" ที่ Map Tunnel IP กับ NBMA IP แบบ Dynamic),
และ **Dynamic IPsec** (Tunnel Protection แทน Crypto Map แบบเดิม) — เดินตาม DMVPN Phase 1
(Hub-and-Spoke, Spoke คุยกันผ่าน Hub เท่านั้น) ไปสู่ Phase 2 (mGRE ทั้ง Hub-Spoke, เปิดทาง
Spoke-to-Spoke Tunnel ตรงแบบ Dynamic) พร้อมอธิบายกลไก NHRP Registration/Resolution, ปัญหา
Split-Horizon บน Multipoint Interface, การวาง EIGRP เป็น Routing Protocol เหนือ DMVPN Cloud,
การเพิ่ม IPsec ป้องกัน Traffic ทั้งหมด, และปิดท้ายด้วย Lab เต็มรูปแบบที่มี **1 Hub (WAN-EDGE-1)
+ 2 Spoke (BRANCH-RTR, BRANCH2-RTR ที่ประกาศใหม่ใน Part นี้)** พิสูจน์ทั้ง NHRP Dynamic Mapping,
EIGRP Convergence, และ Direct Spoke-to-Spoke Tunnel ที่เกิดขึ้นเองหลัง Traffic จริง

Part นี้ยังทิ้งข้อจำกัดสำคัญไว้ 1 ข้อ: **Hub ไม่สามารถ Summarize Route ของ Spoke ได้อีกต่อไปใน
Phase 2** เพราะต้องคง Next-Hop เดิมไว้เสมอเพื่อให้ NHRP Resolution ทำงานได้ — นี่คือจุดเริ่มต้น
ของ Part 53

✅ **พร้อมสำหรับ Part 53**: **DMVPN Advanced** — DMVPN Phase 3 (NHRP Redirect/Shortcut ที่แก้
ข้อจำกัด Summarization ของ Phase 2 ได้สมบูรณ์), Dual-Hub/Dual-Cloud สำหรับ Redundancy ระดับ
Enterprise, DMVPN กับ IPv6, และการ Troubleshoot DMVPN ระดับลึกที่ข้อสอบ CCNP ENCOR ชอบถาม

**ไปต่อ:** [Part 53 — DMVPN Advanced →](part-053-dmvpn-advanced.md)
