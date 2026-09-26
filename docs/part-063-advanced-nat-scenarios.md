# Part 63 — Advanced NAT Scenarios
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 621–630 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 62 — Infrastructure Services Troubleshooting](part-062-infra-services-troubleshooting.md)
> ที่เจาะ Troubleshoot DHCP/DNS/NTP/SNMP ให้ลึกขึ้นในระดับ ENARSI Part นี้จะกลับมาที่หัวข้อ **NAT**
> ที่ [Part 15](part-015-nat-pat.md) ปูพื้นฐานไว้ตั้งแต่ระดับ CCNA (Static NAT, Dynamic NAT, PAT,
> Order of Operations) แล้วยกระดับขึ้นสู่ **สถานการณ์ขั้นสูงที่ ENARSI Blueprint คาดหวัง**:
> Overlapping Address Space หลัง M&A, การผสม NAT กับ VRF-Lite (ต่อจาก
> [Part 51](part-051-vrf-lite-gre-ipsec.md)), และที่สำคัญที่สุดคือ **การ Troubleshoot NAT ที่พัง
> ในรูปแบบที่ซับซ้อนกว่าการลืม mark inside/outside** — ตั้งแต่ Translation Table เต็ม, ACL ที่แอบ
> บล็อก Traffic หลัง/ก่อน NAT, ไปจนถึง Asymmetric Routing ที่ทำให้ NAT State ไม่ตรงกันระหว่าง
> Router สองตัว (เชื่อมกับแนวคิด PBR จาก [Part 32](part-032-policy-based-routing.md)) ทุก Config
> ใน Part นี้ยังคงอยู่บน `WAN-EDGE-1`/`WAN-EDGE-2` ตาม Topology หลักของหลักสูตร เพื่อให้ต่อยอดจาก
> Static NAT + PAT ที่ Part 15 วางไว้ได้โดยตรง

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 621 | NAT กับ Overlapping Address Space — Merger & Acquisition ที่สอง Network ใช้ Subnet ซ้ำกัน |
| 622 | NAT กับ VRF Interaction — แปลง Address ข้ามจาก VRF ไปยัง Global Routing Table |
| 623 | Troubleshoot: PAT ล้มเหลวเฉพาะ Connection ใหม่ — Translation Table/Port Exhaustion |
| 624 | Troubleshoot: Static NAT Server1 (HTTPS) เข้าถึงไม่ได้หลังแก้ ACL — กับดัก NAT Order of Operations |
| 625 | Troubleshoot: Asymmetric NAT — WAN-EDGE-1/WAN-EDGE-2 ไม่มี NAT State ตรงกัน |
| 626 | NAT64 / NAT-PT — แนวคิดสำหรับ IPv4-to-IPv6 Transition (Exam Awareness) |
| 627 | NVI (NAT Virtual Interface) — ทางเลือกสมัยใหม่แทน Inside/Outside แบบดั้งเดิม |
| 628 | Debug NAT เชิงลึก — อ่าน `debug ip nat detailed` ทีละบรรทัด |
| 629 | Performance & Scale ของ NAT — Process-Switching, CEF Punt, และ CGN (Carrier-Grade NAT) |
| 630 | Lab เต็มรูปแบบ: แก้ปัญหา Exhaustion + NAT/ACL Ordering พร้อมกันบน WAN-EDGE-1 |

---

## Step 621 — NAT กับ Overlapping Address Space: Merger & Acquisition

### 621.1 ปัญหาที่เกิดขึ้นจริงหลัง M&A

สมมติสถานการณ์: องค์กรของเรา (ตาม Topology หลักของหลักสูตร) เข้าซื้อบริษัทขนาดเล็กชื่อ
**Acquired Corp** และต้องเชื่อมเครือข่ายทั้งสองเข้าด้วยกันผ่าน Leased Line ใหม่ระหว่าง
`WAN-EDGE-1` กับ Router ของ Acquired Corp (`ACQ-RTR`) — ปัญหาคือ **Acquired Corp ใช้ Subnet
`10.10.10.0/24` เหมือนกับ VLAN 10 (SALES) ของเราเป๊ะ** (เหตุการณ์แบบนี้เกิดขึ้นจริงบ่อยมากหลัง
M&A เพราะทั้งสององค์กรเลือก RFC 1918 Range แบบเดียวกันโดยไม่ได้ตกลงกันไว้ก่อน)

**ผลกระทบ**: `WAN-EDGE-1` ไม่สามารถมี Routing Table Entry สอง Entry ที่ชี้ไปยัง Prefix
`10.10.10.0/24` เดียวกันแต่มีความหมายคนละเครือข่ายได้เลย (Entry หนึ่งต้องไปทาง CORE-SW1 สำหรับ
VLAN 10 ของเราเอง อีก Entry ต้องไปทาง `ACQ-RTR` สำหรับ Acquired Corp) — Longest-Prefix-Match
จะเลือกได้แค่ Entry เดียว ทำให้อีกฝั่งไปไม่ถึงตลอดกาล **นี่ไม่ใช่ปัญหาที่ Routing Protocol ตัวไหน
แก้ได้ — ต้องใช้ NAT แปลง Address ทั้งสองฝั่งเพื่อให้ Prefix ไม่ชนกันอีกต่อไป**

### 621.2 หลักการ: ต้องแปลงทั้งสองทิศทาง (Double-Sided NAT)

Overlapping Address ต้องใช้ NAT ทั้ง 2 รูปแบบพร้อมกันบนอุปกรณ์ตัวเดียว (`WAN-EDGE-1`):

| ทิศทาง | คำสั่ง | ทำหน้าที่ |
|---|---|---|
| **Inside source static network** | `ip nat inside source static network ...` | แปลง Address จริงของเรา (`10.10.10.0/24`) ให้เป็น Address ปลอม (Inside Global) ที่ Acquired Corp มองเห็นแทน — เพราะเราไม่สามารถส่ง Address จริงของเราไปให้ Acquired Corp ได้ (มันซ้อนกับ Subnet ของเขาเอง) |
| **Outside source static network** | `ip nat outside source static network ...` | แปลง Address จริงของ Acquired Corp (`10.10.10.0/24` ฝั่งเขา) ให้เป็น Address ปลอม (Outside Local) ที่เราใช้อ้างอิงถึงเขาแทน — ทิศทางนี้คือทิศทางที่ Part 15 Step 141.4 บอกว่า "พบไม่บ่อย" เพราะ 90% ของ Lab ทั่วไปไม่มี Outside NAT เลย |

**Keyword `network`** (ต่างจาก Static NAT ปกติใน Part 15 Step 142 ที่แปลงทีละ Host) แปลง
**ทั้ง Subnet พร้อมกันในคำสั่งเดียว** โดยใช้ Offset เดียวกันกับ Host ทุกตัวในนั้น (เช่น
`10.10.10.50` → `172.20.20.50` คือ Host เดิม บวก Offset ของ Network ใหม่)

### 621.3 ออกแบบ Address Mapping

| Address จริง (Real) | ความหมาย | Address ปลอมที่ใช้แทน |
|---|---|---|
| `10.10.10.0/24` (ของเรา, VLAN 10 SALES) | Inside Local | `172.20.20.0/24` → **Inside Global** (Acquired Corp ใช้ Range นี้อ้างถึงเรา) |
| `10.10.10.0/24` (ของ Acquired Corp) | Outside Global (Address จริงของเขา) | `172.20.10.0/24` → **Outside Local** (เราใช้ Range นี้อ้างถึงเขา) |

Link ใหม่ระหว่าง `WAN-EDGE-1` กับ `ACQ-RTR` ใช้ `192.0.2.0/30` (ตัวอย่าง Documentation Range
ตาม RFC 5737 เพื่อไม่ชนกับ Address จริงของหลักสูตร) — `WAN-EDGE-1` = `192.0.2.1`,
`ACQ-RTR` = `192.0.2.2`

### 621.4 Worked Config เต็มรูปแบบบน WAN-EDGE-1

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/3
WAN-EDGE-1(config-if)# description ** OUTSIDE ** to ACQ-RTR (Acquired Corp - overlapping 10.10.10.0/24)
WAN-EDGE-1(config-if)# ip address 192.0.2.1 255.255.255.252
WAN-EDGE-1(config-if)# ip nat outside
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# ip nat inside source static network 10.10.10.0 172.20.20.0 /24
WAN-EDGE-1(config)# ip nat outside source static network 10.10.10.0 172.20.10.0 /24
!
WAN-EDGE-1(config)# ip route 172.20.10.0 255.255.255.0 192.0.2.2
```

> **ทำไมต้องมี `ip route 172.20.10.0 ... 192.0.2.2`**: `WAN-EDGE-1` **ห้าม** Route ไปยัง Prefix
> จริง `10.10.10.0/24` ผ่าน `192.0.2.2` เด็ดขาด (จะชนกับ Route เดิมของ VLAN 10 ตัวเองทันที) —
> ต้อง Route ไปยัง **Prefix ที่แปลงแล้ว (`172.20.10.0/24`)** เท่านั้น แล้วปล่อยให้ NAT เปลี่ยน
> Destination กลับเป็น Address จริงตอนที่ Packet ออกทาง Interface Outside (`Gi0/0/3`) — Interface
> `Gi0/0/1`/`Gi0/0/2` (Inside เดิมของ Part 15) ไม่ต้องแก้ไขอะไรเลย

Interface ฝั่ง `ACQ-RTR` (จำลองไว้เพื่อความสมบูรณ์ ไม่ใช่อุปกรณ์ของเรา) จะต้องมี Static Route
กลับมาที่ `172.20.20.0/24 via 192.0.2.1` แบบเดียวกัน (ทีมของ Acquired Corp ต้อง Config ฝั่งเขา
เองด้วยหลักการเดียวกัน — NAT แบบนี้ต้อง**ตกลงกันทั้งสองฝั่ง**ว่าใครใช้ Range ปลอมอะไร)

### 621.5 ทดสอบและ Verify

```
! PC1 (10.10.10.50 จริง) ping ไปยัง File Server ของ Acquired Corp ผ่าน Address ปลอม
PC1> ping 172.20.10.20

WAN-EDGE-1# show ip nat translations
Pro Inside global       Inside local        Outside local        Outside global
icmp 172.20.20.50:1     10.10.10.50:1       172.20.10.20:1        10.10.10.20:1
```

สังเกตว่า **ทุกคอลัมน์มีค่าครบทั้ง 4 ช่อง** — ต่างจากทุก Example ใน Part 15 ที่ `Outside local`
= `Outside global` เสมอ (เพราะไม่มี Outside NAT) นี่คือกรณีพิเศษที่ Part 15 Step 141.4 บอกไว้ว่า
"เป็นกรณีพิเศษที่ใช้ตอนมี Overlapping Address ระหว่างสอง Site" — และนี่คือ Worked Example ที่
สมบูรณ์ของกรณีนั้น: `10.10.10.50` (จริง) ↔ `172.20.20.50` (ที่ Acquired Corp มองเห็น) และ
`10.10.10.20` (จริงของเขา) ↔ `172.20.10.20` (ที่เรามองเห็น)

### 621.6 ข้อจำกัดและข้อควรระวัง

1. **ต้องตกลง Address Mapping กับอีกฝั่งล่วงหน้าเสมอ** — NAT ฝั่งเราแก้ปัญหาแค่ครึ่งทาง
   ถ้าอีกฝั่งไม่ Route ไปยัง Range ปลอมที่เราประกาศให้ Traffic กลับจะไปไม่ถึง
2. **Application ที่ Embed IP Address ไว้ใน Payload** (เช่น FTP Active Mode, SIP, บาง Protocol
   ที่ส่ง IP ของตัวเองในข้อมูล ไม่ใช่แค่ Header) จะพังกับ Overlapping NAT แบบนี้เสมอ เพราะ NAT
   ปกติแก้ไขแค่ Header ไม่แก้ Payload (ต้องพึ่ง ALG — Application Layer Gateway ซึ่งมีจำกัดเฉพาะ
   บาง Protocol เท่านั้น)
3. เป็น Solution ระยะสั้นเสมอ — ทางออกระยะยาวที่ถูกต้องคือ **Renumber (เปลี่ยน IP Address จริง)**
   ของฝั่งใดฝั่งหนึ่งให้ไม่ชนกัน แต่ Renumber ทั้งองค์กรใช้เวลาเป็นเดือน/ปี ในระหว่างนั้น
   Overlapping NAT คือสิ่งที่ทำให้ธุรกิจสองฝั่งคุยกันได้ทันทีหลัง M&A

---

## Step 622 — NAT กับ VRF Interaction: แปลง Address ข้ามจาก VRF ไปยัง Global Table

### 622.1 ทวนปัญหาจาก Part 51

[Part 51 Step 504.3](part-051-vrf-lite-gre-ipsec.md) ให้ VRF `GUEST` (แยก VLAN 40 Wi-Fi ออกจาก
Global Table ที่ `DIST-SW3`/`DIST-SW4`) ออก Internet ได้ด้วยวิธี **Static Route Leaking**
(`ip route vrf GUEST 0.0.0.0 0.0.0.0 ... global`) — Traffic รั่วเข้า Global Table ที่ CORE
แล้วค่อยไปโดน PAT ปกติที่ `WAN-EDGE-1` (Part 15) ข้อเสียของวิธีนี้ที่ Part 51 Step 504.4 เตือน
ไว้คือ: **การ Leak Default Route เปิดทางให้ VRF `GUEST` มองเห็น Global Table ทั้งก้อนในทางทฤษฎี**
ต้องพึ่ง ACL ป้องกันซ้ำเสมอ

### 622.2 ทางเลือกที่สะอาดกว่า: ทำ NAT ตรงที่จุดขอบ VRF เลย

Cisco IOS มี Feature **"NAT Support for Multiple VRFs"** ที่ให้ `ip nat inside source ...`
ระบุ Keyword **`vrf <name>`** เพิ่มเข้ามาได้ — ความหมายคือ: "จับคู่ ACL นี้กับ Traffic **ที่มาจาก
Routing Table ของ VRF ที่ระบุเท่านั้น** แล้วแปลงไปเป็น Address จาก Pool ที่อยู่ **นอก VRF นั้น**
(ปกติคือ Global Table)" — วิธีนี้ทำให้ NAT **เป็นจุดควบคุมทางออกจุดเดียว** ของ VRF นั้นในตัวเอง
โดยไม่ต้อง Leak Default Route เข้า Global Table เลย (Global Table ไม่รู้จัก VRF `GUEST` เลย
แม้แต่นิดเดียว — ตรงข้ามกับวิธี Static Leak ที่ Part 51 ใช้)

> สมมติว่าในอนาคตองค์กรขยาย VRF `GUEST` แบบ Hop-by-Hop มาจนถึง `WAN-EDGE-1` แล้ว (ตามที่
> [Part 51 Step 503.3](part-051-vrf-lite-gre-ipsec.md) เกริ่นไว้ว่าต้องคอนฟิก VRF ซ้ำทุก Hop
> ที่ Traffic ต้องเดินทางผ่าน) — Part นี้จะสาธิตว่าเมื่อถึงจุดนั้นแล้ว NAT ที่ `WAN-EDGE-1`
> ควรถูกปรับให้ "รู้จัก" VRF `GUEST` โดยตรงอย่างไร แทนที่จะพึ่ง Static Leak เพียงอย่างเดียว

### 622.3 Worked Config — NAT ที่ WAN-EDGE-1 รับรู้ VRF GUEST โดยตรง

```
WAN-EDGE-1(config)# vrf definition GUEST
WAN-EDGE-1(config-vrf)# rd 65001:40
WAN-EDGE-1(config-vrf)# address-family ipv4
WAN-EDGE-1(config-vrf-af)# route-target export 65001:40
WAN-EDGE-1(config-vrf-af)# route-target import 65001:40
WAN-EDGE-1(config-vrf-af)# exit-address-family
WAN-EDGE-1(config-vrf)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1.40
WAN-EDGE-1(config-subif)# description ** GUEST VRF dedicated logical link from CORE-SW1 **
WAN-EDGE-1(config-subif)# encapsulation dot1Q 40
WAN-EDGE-1(config-subif)# vrf forwarding GUEST
WAN-EDGE-1(config-subif)# ip address 10.10.254.14 255.255.255.252
WAN-EDGE-1(config-subif)# ip nat inside
WAN-EDGE-1(config-subif)# exit
!
WAN-EDGE-1(config)# ip nat pool GUEST-PAT-POOL 203.0.113.13 203.0.113.13 netmask 255.255.255.248
WAN-EDGE-1(config)# ip access-list standard NAT-GUEST-ACL
WAN-EDGE-1(config-std-nacl)# permit 10.10.40.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# exit
WAN-EDGE-1(config)# ip nat inside source list NAT-GUEST-ACL pool GUEST-PAT-POOL vrf GUEST overload
```

| ส่วนของคำสั่ง | ความหมาย |
|---|---|
| `interface Gi0/0/1.40` + `vrf forwarding GUEST` | จุดที่ `WAN-EDGE-1` เข้าร่วม VRF `GUEST` (Interface ต้อง Mark `ip nat inside` เหมือนเดิมทุกประการ) |
| `ip nat pool GUEST-PAT-POOL 203.0.113.13 ...` | Public IP แยกเฉพาะสำหรับ Guest — **ไม่ใช้ Public IP ตัวเดียวกันกับ PAT ของ VLAN 10/20/30/40 ปกติ** (Traceability: Log ที่ปลายทางจะแยก Guest Traffic ออกจาก Internal Traffic ได้ทันทีจาก Source IP) |
| `... pool GUEST-PAT-POOL vrf GUEST overload` | **Keyword `vrf GUEST`** คือหัวใจของ Step นี้ — บอก IOS ว่า ACL `NAT-GUEST-ACL` ต้อง Match กับ Routing Table ของ VRF `GUEST` เท่านั้น (ไม่ใช่ Global Table) ส่วน Pool `GUEST-PAT-POOL` อยู่ใน **Global Table** ตามปกติ (Interface `Gi0/0/0` Outside ยังเป็น Global Table เดิม ไม่ต้องแก้ไข) |

### 622.4 Verify

```
WAN-EDGE-1# show vrf
  Name                             Default RD            Protocols   Interfaces
  GUEST                            65001:40               ipv4        Gi0/0/1.40

WAN-EDGE-1# show ip nat translations vrf GUEST
Pro Inside global        Inside local         Outside local        Outside global
tcp 203.0.113.13:1024    10.10.40.55:52301    93.184.216.34:80     93.184.216.34:80

WAN-EDGE-1# show ip nat statistics vrf GUEST
Total translations: 1 (0 static, 1 dynamic; 1 extended)
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1.40
```

คำสั่ง `show` ของ NAT ทุกตัวรองรับ Keyword `vrf <name>` เพิ่มเข้ามาเช่นกัน (เหมือนที่
[Part 51 Step 503.4](part-051-vrf-lite-gre-ipsec.md) แสดงให้เห็นกับ `show ip route vrf GUEST`)

### 622.5 เทียบข้อดี-ข้อเสียกับวิธี Static Leak เดิม (Part 51)

| ประเด็น | Static Route Leak (Part 51) | NAT + VRF Keyword (Step นี้) |
|---|---|---|
| Global Table รู้จัก VRF GUEST หรือไม่ | รู้จักทางอ้อม (ผ่าน Route Leak) | **ไม่รู้จักเลย** — Isolation แน่นกว่า |
| ต้องพึ่ง ACL ป้องกันซ้ำ (Defense-in-Depth) | ต้อง (ตามคำเตือน Step 504.4) | ยังแนะนำให้มี แต่ความเสี่ยงเริ่มต้นต่ำกว่า |
| ความซับซ้อนของ Config | ต่ำกว่า (Static Route ธรรมดา) | สูงกว่า (ต้องมี VRF ที่ NAT Boundary เอง) |
| เหมาะกับ Topology แบบไหน | VRF Egress Point ≠ NAT Boundary (หลาย Hop) | VRF Egress Point = NAT Boundary (Hop เดียวกัน) |

---

## Step 623 — Troubleshoot: PAT ล้มเหลวเฉพาะ Connection ใหม่ (Translation Table/Port Exhaustion)

### 623.1 อาการที่รายงานเข้ามา

Helpdesk รายงาน: User หลายคนใน VLAN 40 (WIFI) **เปิด Web ใหม่ไม่ได้** แต่ Session ที่เปิดอยู่ก่อน
แล้ว (เช่น Video Call ที่ค้างอยู่) **ยังทำงานปกติ** — Ping ไป Default Gateway ผ่านปกติทุกอย่าง,
`show ip nat statistics` ยืนยันว่า Interface Inside/Outside Mark ถูกต้องครบ (ตัด Step 145.4 ออก
จาก List of Suspects ได้ทันที)

### 623.2 วินิจฉัย: ดู show ip nat statistics ให้ลึกกว่าปกติ

```
WAN-EDGE-1# show ip nat statistics
Total translations: 63980 (1 static, 63979 dynamic; 63979 extended)
Peak translations: 64021, occurred 00:02:14 ago
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1, GigabitEthernet0/0/2
Hits: 8823012  Misses: 4127
CEF Translated packets: 8823012, CEF Punted packets: 118
Expired translations: 8759998
Dynamic mappings:
-- Inside Source
[Id: 1] access-list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 refcount 63979
```

**จุดที่ต้องอ่านให้ออก**:

1. `Total translations: 63980` — PAT ผ่าน 1 Public IP มี Port ใช้ได้จริงประมาณ **64,512 ค่า**
   (Ephemeral Port Range ของ IOS) ตัวเลข `63980` คือ **ใกล้เต็ม Pool มากแล้ว**
2. `Misses: 4127` และเพิ่มขึ้นเรื่อยๆ — Traffic พยายามสร้าง Translation ใหม่แต่ **หา Port ว่าง
   ไม่ได้** จึงถูกนับเป็น Miss แทน Hit
3. `refcount 63979` ยืนยันว่าทุก Entry มาจาก ACL/Interface เดียวกัน (`NAT-INTERNAL-ACL` +
   `Gi0/0/0`) — สอดคล้องกับอาการที่ทุก VLAN กระทบพร้อมกัน ไม่ใช่ปัญหาเฉพาะ VLAN ใดวลาน หนึ่ง

### 623.3 ยืนยันด้วย debug ip nat detailed สำหรับ Connection ใหม่ที่ล้มเหลว

```
WAN-EDGE-1# debug ip nat detailed
IP NAT detailed debugging is on
WAN-EDGE-1#
*Sep 26 11:16:40.881: NAT: i: tcp (10.10.40.203, 51820) -> (203.0.113.2, ?)
*Sep 26 11:16:40.881: NAT: no available port in pool/interface for translation
*Sep 26 11:16:40.882: NAT*: translation failed, dropping packet s=10.10.40.203:51820 d=93.184.216.34:443
```

Log ยืนยันตรงตัวว่า **หาPort ว่างให้ Source ใหม่ไม่ได้เลย** — ไม่ใช่ปัญหา Routing, ACL, หรือ
Interface Mark เลยแม้แต่นิดเดียว เป็น **Resource Exhaustion ล้วนๆ**

### 623.4 สาเหตุที่แท้จริง

Root Cause มักเป็นหนึ่งในสองแบบนี้เสมอ:

1. **จำนวน Session พร้อมกันมากผิดปกติจริง** (Event พิเศษ, User เพิ่มขึ้นเยอะ, Application ที่เปิด
   Connection จำนวนมากต่อ User เช่น Streaming/Ads ที่เปิด Parallel Connection เกินจำเป็น)
2. **Session เก่าไม่ถูกปล่อย Port คืน** — Timeout ยาวเกินไป (Default TCP Timeout ของ NAT คือ
   `86400` วินาที = 24 ชั่วโมง!) ทำให้ Session ที่ Client ปิดไปแล้วจริง (แต่ไม่ได้ส่ง FIN/RST
   ครบ เช่น Client หลุด Wi-Fi กะทันหัน) ยังค้างอยู่ใน Table โดยไม่จำเป็น

### 623.5 วิธีแก้ที่ 1 — เพิ่ม Public IP เข้า Pool (ขยาย Port Space)

เปลี่ยนจาก PAT แบบ `interface ... overload` (ใช้ IP เดียว) เป็น `pool ... overload` (ใช้หลาย IP)
เพื่อเพิ่ม Port Space รวมให้มากขึ้นตามจำนวน Public IP:

```
WAN-EDGE-1(config)# ip nat pool INTERNET-PAT-POOL 203.0.113.9 203.0.113.11 netmask 255.255.255.248
WAN-EDGE-1(config)# no ip nat inside source list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 overload
WAN-EDGE-1(config)# ip nat inside source list NAT-INTERNAL-ACL pool INTERNET-PAT-POOL overload
```

3 Public IP × ~64,512 Port ต่อ IP = **Port Space รวมกว่า 193,000 ค่า** (เพิ่มขึ้น 3 เท่า)
รองรับจำนวน Session พร้อมกันที่มากขึ้นได้ทันที โดยไม่ต้องรอ Session เก่าหมดอายุก่อน

> **ข้อควรระวัง**: Public IP ที่นำมาใช้ต้องมาจาก Block ที่ ISP Route มาให้จริง (เช่นก้อน
> `203.0.113.8/29` ที่ Part 15 ใช้ทำ Static NAT ของ Server1 อยู่แล้ว — ต้องเช็คว่า `.9`-`.11`
> ไม่ชนกับ Static NAT Entry อื่นตาม Step 149.4 เรื่อง Overlapping NAT Pool)

### 623.6 วิธีแก้ที่ 2 — ปรับ Timeout ให้สอดคล้องกับพฤติกรรม Traffic จริง

```
WAN-EDGE-1(config)# ip nat translation tcp-timeout 3600
WAN-EDGE-1(config)# ip nat translation udp-timeout 120
WAN-EDGE-1(config)# ip nat translation finrst-timeout 5
```

| Timeout | Default | ค่าที่ปรับ | เหตุผล |
|---|---|---|---|
| `tcp-timeout` | 86400s (24 ชม.) | `3600` (1 ชม.) | Session TCP ที่ Idle นานเกิน 1 ชม. ส่วนใหญ่คือ Session ตายแล้วโดยไม่ได้ปิดสะอาด |
| `udp-timeout` | 300s (5 นาที) | `120` (2 นาที) | UDP ไม่มี Connection State ให้เช็ค ปล่อย Port คืนเร็วขึ้นลดความเสี่ยง Exhaustion |
| `finrst-timeout` | 60s | `5` | เมื่อเห็น FIN/RST ของ TCP แล้ว (Session ปิดแบบสมบูรณ์) ควรปล่อย Port คืนเกือบทันที ไม่ต้องรอ 60 วินาที |

**สองวิธีนี้ใช้ร่วมกันได้** และแนะนำให้ทำทั้งคู่ในสถานการณ์จริง — เพิ่ม Pool แก้ปัญหาทันที
ส่วนปรับ Timeout ป้องกันไม่ให้ปัญหาเกิดซ้ำจาก Session ค้างในอนาคต

### 623.7 Verify หลังแก้ไข

```
WAN-EDGE-1# show ip nat statistics
Total translations: 812 (1 static, 811 dynamic; 811 extended)
Peak translations: 64021, occurred 1d02h ago
Hits: 8925501  Misses: 0
```

`Misses: 0` และ `Total translations` ลดลงกลับสู่ระดับปกติ — ยืนยันว่าปัญหา Exhaustion หายไปแล้ว

---

## Step 624 — Troubleshoot: Static NAT Server1 เข้าถึงไม่ได้หลังแก้ ACL (กับดัก Order of Operations)

### 624.1 สถานการณ์

Server1 (`10.10.30.10`, Static NAT เป็น `203.0.113.10:443` ตาม Part 15 Step 147/150) ใช้งานได้ดี
มาตลอด จนกระทั่งทีม Security ขอเพิ่มเงื่อนไข "จำกัดให้เฉพาะ Partner Range `198.51.100.0/24`
เข้าถึง Server1 ได้เท่านั้น" — หลังจากทีมอื่น (ไม่ใช่ทีมที่ทำ NAT เดิม) เพิ่ม ACL ใหม่เข้าไป
**HTTPS ไปยัง Server1 ใช้งานไม่ได้เลยจาก Partner Range ที่ควรจะผ่านได้**

### 624.2 ขั้นที่ 1 — ยืนยันว่า NAT เองไม่มีปัญหา

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---

WAN-EDGE-1# debug ip nat detailed
*Sep 26 13:02:11.100: NAT: o: tcp (198.51.100.20, 51902) -> (203.0.113.10, 443)
*Sep 26 13:02:11.101: NAT: match found using static entry 203.0.113.10:443 10.10.30.10:443
*Sep 26 13:02:11.102: NAT*: TCP s=198.51.100.20, d=203.0.113.10->10.10.30.10 [51902]
```

`debug ip nat detailed` ยืนยันชัดเจนว่า **NAT แปลง Destination สำเร็จ** (`203.0.113.10` →
`10.10.30.10`) — ปัญหาไม่ได้อยู่ที่ NAT เลย ต้องมี Packet ถูกบล็อกที่จุดอื่นหลังจากนี้

### 624.3 ขั้นที่ 2 — พบ ACL ใหม่ที่ทีม Security เพิ่มเข้ามา

ตรวจสอบ Config พบว่าทีม Security เพิ่ม ACL ใหม่ที่ **Inside Interface ทิศทาง `out`** (คิดว่าเป็น
จุดที่ "ปลอดภัยกว่า" เพราะใกล้ Server มากกว่า) โดยเขียน ACL อ้างอิง **Public IP** ของ Server1
(เข้าใจว่า Address ที่ Partner เห็นคือ Address ที่ต้องใช้กรอง):

```
WAN-EDGE-1(config)# ip access-list extended SERVER1-PARTNER-ACL
WAN-EDGE-1(config-ext-nacl)# permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# deny ip any host 10.10.30.10 log
WAN-EDGE-1(config-ext-nacl)# permit ip any any
WAN-EDGE-1(config-ext-nacl)# exit
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# ip access-group SERVER1-PARTNER-ACL out
WAN-EDGE-1(config-if)# exit
```

### 624.4 วิเคราะห์ตาม NAT Order of Operations (ทวนจาก Part 15 Step 146)

```
Outside -> Inside:
  [Inbound ACL บน Outside Interface]   <- ใช้ Public IP (ถูก - นี่คือ OUTSIDE-IN-ACL เดิม)
        │
        ▼
  [NAT แปลง Destination: 203.0.113.10 -> 10.10.30.10]   <- NAT ทำงานตรงนี้ สำเร็จแล้ว
        │
        ▼
  [Routing Decision]
        │
        ▼
  [Outbound ACL บน Inside Interface]   <- ณ จุดนี้ Destination = 10.10.30.10 (Private) แล้ว!
        │                                  ACL ใหม่เขียนด้วย Public IP 203.0.113.10
        ▼                                  จึงไม่มีวัน Match บรรทัด permit แรกได้เลย!
  ส่งออกทาง Inside Interface
```

**บทเรียนตรงนี้คือกับดักคลาสสิกที่ Part 15 Step 146.2-146.3 เตือนไว้**: ACL ที่ Apply บน
**Inside Interface ทิศทาง `out`** สำหรับ Traffic ขาเข้า (Outside → Inside) จะเห็น Packet
**หลัง NAT แปลง Destination แล้วเสมอ** — ทีม Security เข้าใจผิดคิดว่า Public IP ยังเป็น
Destination อยู่ ทำให้บรรทัด `permit` แรกไม่ Match อะไรเลย Traffic ทั้งหมด (แม้จาก Partner Range
ที่ถูกต้อง) จึงตกไปโดน `deny ip any host 10.10.30.10 log` แทน

### 624.5 หลักฐานยืนยันจาก Log

```
WAN-EDGE-1#
*Sep 26 13:05:44.220: %SEC-6-IPACCESSLOGP: list SERVER1-PARTNER-ACL denied tcp
  198.51.100.20(51903) -> 10.10.30.10(443), 1 packet
```

**นี่คือหลักฐานสำคัญที่สุด**: Log แสดง Destination เป็น `10.10.30.10` (Private) ไม่ใช่
`203.0.113.10` ที่ทีม Security ตั้งใจเขียนไว้ใน ACL เลย — ยืนยันตรงตัวว่า ACL กำลังมองเห็น
Packet ที่ NAT แปลงไปแล้ว ไม่ใช่ Packet ดิบที่ทีม Security คิดไว้

### 624.6 แก้ไข — 2 ทางเลือก

**ทางเลือกที่ 1 (แก้เร็วที่สุด)**: แก้ ACL ให้ตรงกับ Address ที่จุดนั้นมองเห็นจริง (Private)

```
WAN-EDGE-1(config)# ip access-list extended SERVER1-PARTNER-ACL
WAN-EDGE-1(config-ext-nacl)# no permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# 5 permit tcp 198.51.100.0 0.0.0.255 host 10.10.30.10 eq 443
WAN-EDGE-1(config-ext-nacl)# exit
```

**ทางเลือกที่ 2 (แนะนำกว่า — ตาม Part 15 Step 148.3 "หนึ่ง Boundary หนึ่งบทบาท")**: ยกเลิก ACL
ใหม่นี้ทั้งหมด แล้วรวม Requirement ของ Partner Range เข้าไปที่ `OUTSIDE-IN-ACL` เดิม (จุดเดียวที่
เห็น Public IP อย่างถูกต้องตามธรรมชาติของมัน — Outside Interface ทิศทาง `in`):

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# no ip access-group SERVER1-PARTNER-ACL out
WAN-EDGE-1(config-if)# exit
WAN-EDGE-1(config)# no ip access-list extended SERVER1-PARTNER-ACL
!
WAN-EDGE-1(config)# ip access-list extended OUTSIDE-IN-ACL
WAN-EDGE-1(config-ext-nacl)# no permit tcp any host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# 5 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# exit
```

ทางเลือกที่ 2 ดีกว่าเพราะ**ไม่มี Policy สองจุดที่ต้องดูแลพร้อมกัน** — ลด Attack Surface ของความ
ผิดพลาดแบบนี้ในอนาคตด้วย

### 624.7 Verify

```
Partner-Host> curl -I https://203.0.113.10
HTTP/1.1 200 OK

WAN-EDGE-1# show access-lists OUTSIDE-IN-ACL | include permit tcp
    5 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443 (234 matches)
```

---

## Step 625 — Troubleshoot: Asymmetric NAT ระหว่าง WAN-EDGE-1/WAN-EDGE-2

### 625.1 บริบท: Dual-ISP Failover ที่เพิ่มเข้ามา

หลังจาก [Part 51 Step 505.4](part-051-vrf-lite-gre-ipsec.md) เพิ่ม Redundancy ระดับ ISP ด้วย
`WAN-EDGE-2` เป็น Backup Path วิศวกรอีกทีมเพิ่ม Default Route สำรองบน `CORE-SW1` ไปยัง
`WAN-EDGE-2` (ผ่าน Cross-link `10.10.254.8/30`) **โดยไม่ได้ตั้งใจให้ Administrative Distance
ต่างกัน** — ผลคือ `CORE-SW1` มี Default Route **สองเส้นทางที่ Metric เท่ากัน (ECMP)** ไปยัง
ทั้ง `WAN-EDGE-1` และ `WAN-EDGE-2` พร้อมกัน:

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.255.10.254 (WAN-EDGE-1 via backbone), via GigabitEthernet1/0/1
    10.10.254.9 (WAN-EDGE-2 via cross-link), via GigabitEthernet1/0/2
```

### 625.2 อาการที่เกิดขึ้น

User รายงาน **Connection หลุดเป็นระยะแบบสุ่ม** โดยเฉพาะ Session ที่ใช้เวลานาน (Download ไฟล์
ใหญ่, Video Call) — Ping ปกติไม่มีปัญหา (Ping สั้นเกินกว่าจะเจอ CEF Re-hash) แต่ TCP Session
ที่ยาวๆ มักถูก RST กลางทาง

### 625.3 วินิจฉัย — เปรียบเทียบ NAT State ระหว่างสอง Router

```
WAN-EDGE-1# show ip nat translations | include 10.10.10.51
tcp 203.0.113.2:52301      10.10.10.51:52301    198.51.100.20:443    198.51.100.20:443

WAN-EDGE-2# show ip nat translations | include 10.10.10.51
! (ไม่มีผลลัพธ์ - WAN-EDGE-2 ไม่มี Entry ของ Session นี้เลย)

WAN-EDGE-2# show ip nat statistics | include Misses
Hits: 442  Misses: 891
```

**Misses สูงผิดปกติที่ `WAN-EDGE-2`** ทั้งที่ Traffic ทั้งหมดของ Session นี้**ควรจะ**วิ่งผ่าน
`WAN-EDGE-1` ตลอด — ยืนยันว่า Packet บางส่วนของ Session เดียวกันถูก CEF Load-Balance ไปทาง
`WAN-EDGE-2` โดยไม่ได้ตั้งใจ (จาก Equal-Cost Default Route ใน 625.1) แต่ `WAN-EDGE-2` **ไม่มี
Translation Entry** ของ Session นั้น (เพราะ SYN แรกไปทาง `WAN-EDGE-1` และถูกสร้าง Entry ไว้ที่
`WAN-EDGE-1` เท่านั้น) — Packet ที่หลุดไปทาง `WAN-EDGE-2` จึงถูก Miss/Drop หรือถูกสร้าง Entry
ใหม่ด้วย Source Public IP **คนละตัว** (`203.0.113.6` ของ `WAN-EDGE-2`) ทำให้ Server ปลายทาง
เห็น Source IP ของ Session เดียวกันเปลี่ยนกลางทาง → TCP Reset

```
CORE-SW1# show ip cef 0.0.0.0
0.0.0.0/0, epoch 0
  nexthop 10.255.10.254 GigabitEthernet1/0/1
  nexthop 10.10.254.9 GigabitEthernet1/0/2
```

`show ip cef` ยืนยันว่า CEF ถือทั้งสอง Next-Hop เป็น Valid Path พร้อมกัน (Load-Sharing) — นี่คือ
Root Cause ตัวจริง: **NAT เป็น Stateful ต่อ Router (Router-Local State) แต่ Routing (CEF ECMP)
ไม่รู้จัก NAT State เลย** เมื่อทั้งสองไม่ประสานกัน Session ที่ยาวพอจะเจอปัญหานี้เสมอ

### 625.4 ทางแก้ที่ 1 — ทำให้ Path เป็น Primary/Backup ที่ชัดเจน (ตัด ECMP)

แก้ที่ต้นเหตุแท้จริง: `WAN-EDGE-2` ควรเป็น **Backup เท่านั้น** ไม่ใช่ Load-Share — ใช้ Floating
Static Route ด้วย Administrative Distance สูงกว่า:

```
CORE-SW1(config)# no ip route 0.0.0.0 0.0.0.0 10.10.254.9
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 10.10.254.9 200
```

`200` ทำให้ Route นี้ถูกใช้ **เฉพาะเมื่อ Path หลัก (ผ่าน `WAN-EDGE-1`, AD=1) หายไปจาก Routing
Table เท่านั้น** — ตัด ECMP ออกไปโดยสิ้นเชิง Session ทั้งก้อนจะ Fail-Over ไปพร้อมกันเป็นเหตุการณ์
เดียว (Binary) ไม่ใช่แยก Packet ทีละใบแบบเดิม แก้ปัญหา Asymmetric NAT ได้ทันทีเพราะตอนนี้มีแค่
Path เดียว Active ในเวลาใดเวลาหนึ่งเท่านั้น

### 625.5 ทางแก้ที่ 2 — ใช้ PBR + IP SLA Tracking เพื่อคุม Symmetric Path อย่างเข้มงวด

ถ้าต้องการ Fail-Over ที่ตรวจสอบ Health จริง (ไม่ใช่แค่รอ Interface `down`) ใช้แนวคิดจาก
[Part 32 Step 314](part-032-policy-based-routing.md) — `set ip next-hop verify-availability`:

```
WAN-EDGE-1(config)# ip sla 10
WAN-EDGE-1(config-ip-sla)# icmp-echo 203.0.113.1
WAN-EDGE-1(config-ip-sla-echo)# frequency 5
WAN-EDGE-1(config-ip-sla-echo)# exit
WAN-EDGE-1(config)# ip sla schedule 10 life forever start-time now
WAN-EDGE-1(config)# track 10 ip sla 10 reachability
WAN-EDGE-1(config-track)# delay down 2 up 5
WAN-EDGE-1(config-track)# exit
!
CORE-SW1(config)# route-map PBR-INTERNET-PRIMARY permit 10
CORE-SW1(config-route-map)# match ip address INTERNET-TRAFFIC
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.255.10.254 10 track 10
CORE-SW1(config-route-map)# exit
CORE-SW1(config)# interface GigabitEthernet1/0/24
CORE-SW1(config-if)# ip policy route-map PBR-INTERNET-PRIMARY
CORE-SW1(config-if)# exit
```

วิธีนี้บังคับให้ Traffic ทั้งหมดยึด `WAN-EDGE-1` เป็น Next-Hop **ตราบใดที่ Track Object ยัง
Reachable** เท่านั้นจึงจะ Fallthrough ไป Routing Table ปกติ (ซึ่งควรมีแค่ Path ไป `WAN-EDGE-2`
เป็นตัวเลือกเดียวที่เหลือ) — ผลลัพธ์เดียวกับ 625.4 (Fail-Over แบบ Binary ไม่ใช่ ECMP) แต่ตรวจจับ
ปัญหาได้เร็วกว่าและละเอียดกว่า (Tracking ผ่าน IP SLA จริง ไม่ใช่แค่ Interface State)

> **บทเรียนสำคัญที่สุดของ Step นี้**: NAT ไม่มี State Synchronization ระหว่าง Router โดย Default
> (Cisco IOS ไม่มี NAT High-Availability แบบ Stateful Failover ใน Feature พื้นฐาน) — การออกแบบ
> Routing (ECMP, Redundancy) **ต้องคำนึงถึงว่ามี NAT อยู่ที่จุดไหนเสมอ** และต้องบังคับให้ Path
> ไป-กลับผ่าน NAT Device ตัวเดียวกันตลอด Session (Symmetric) ไม่ใช่ปล่อยให้ Routing เลือกเอง
> อย่างอิสระ — ตรงกับที่ Part 15 Step 149.4 เตือนไว้สั้นๆ ("Asymmetric Routing") ส่วน Step นี้คือ
> การขยายปัญหานั้นให้เห็นเต็มรูปแบบพร้อมวิธีแก้จริง

---

## Step 626 — NAT64 / NAT-PT: แนวคิดสำหรับ IPv4-to-IPv6 Transition

### 626.1 ปัญหาที่ NAT64 แก้ไข

Lab ของหลักสูตรนี้เป็น **Dual-Stack** (ทุก VLAN มีทั้ง IPv4 `10.10.x.0/24` และ IPv6 ULA
`fd00:10:10:x::/64` ควบคู่กันตาม [00-ip-address-plan.md](00-ip-address-plan.md)) จึงไม่จำเป็น
ต้องใช้ NAT64 จริงในหลักสูตรนี้ — แต่ CCNP ENARSI Blueprint ยังคาดหวังให้เข้าใจแนวคิดนี้ไว้
สำหรับสถานการณ์ที่ต่างออกไป: **เครือข่ายที่เป็น IPv6-Only ล้วนๆ** (ไม่มี IPv4 เหลือให้ใช้แล้ว
เช่น Mobile Carrier Network สมัยใหม่) แต่ยังต้องเข้าถึง Resource บน Internet ที่เป็น
**IPv4-Only** อยู่ (เว็บไซต์เก่าจำนวนมากยังไม่รองรับ IPv6)

### 626.2 หลักการทำงานของ NAT64 (Stateful, RFC 6146)

```
IPv6-Only Client                NAT64 Gateway                    IPv4-Only Server
fd00::100                 ──►    แปลง IPv6 <-> IPv4              ──►    198.51.100.20
                                  Well-Known Prefix: 64:ff9b::/96
```

- Client ส่ง Packet ไปยัง Address สังเคราะห์ `64:ff9b::198.51.100.20` (ฝัง IPv4 Address จริง
  ไว้ใน 32 bit ท้ายของ IPv6 Address ตาม Prefix มาตรฐาน `64:ff9b::/96`)
- NAT64 Gateway แปลง Packet ทั้ง Header (IPv6 → IPv4) และทำ PAT ตามปกติสำหรับ IPv4 Address
  ปลายทางจริง — เป็น Stateful เหมือน PAT (ต้องเก็บ Translation Table)
- ต้องคู่กับ **DNS64**: DNS Resolver พิเศษที่สังเคราะห์ AAAA Record ปลอม (`64:ff9b::` + A Record
  จริง) ให้กับ Client ที่ Query โดเมนที่มีแค่ A Record (IPv4-only) เท่านั้น

### 626.3 NAT-PT (Legacy — Deprecated)

**NAT-PT (Network Address Translation - Protocol Translation, RFC 2766)** เป็นเทคนิครุ่นก่อน
NAT64 ที่ IETF **Deprecate ไปแล้วอย่างเป็นทางการ (RFC 4966)** เพราะมีปัญหาเรื่อง Scalability
และ Application Compatibility จำนวนมาก — ข้อสอบ CCNP อาจถามถึงชื่อนี้เพื่อทดสอบว่ารู้จัก
Timeline ของเทคโนโลยี Transition แต่**ไม่ควรใช้งานจริงในสถาปัตยกรรมใหม่**

### 626.4 IOS-XE Syntax (แนวคิดเท่านั้น — ไม่มีใน Lab นี้)

Platform ระดับ Provider (ASR1000, CSR1000v บางรุ่นที่มี License เฉพาะ) รองรับ Feature นี้ผ่าน
Syntax แนวคิด (ไม่ลง Lab เต็มรูปแบบเพราะ Lab นี้เป็น Dual-Stack ไม่ต้องพึ่ง NAT64):

```
Router(config)# nat64 prefix stateful 64:ff9b::/96
Router(config)# interface GigabitEthernet0/0/0
Router(config-if)# nat64 enable
```

> **สรุปสำหรับข้อสอบ**: NAT64 = IPv6-Only Client เข้าถึง IPv4-Only Server, ต้องคู่กับ DNS64,
> เป็น Stateful เหมือน PAT — ใช้มากในเครือข่าย Mobile/Carrier ที่เป็น IPv6-Only จริงๆ ไม่ใช่
> Enterprise Dual-Stack แบบหลักสูตรนี้

---

## Step 627 — NVI (NAT Virtual Interface): ทางเลือกสมัยใหม่แทน Inside/Outside

### 627.1 ข้อจำกัดของ Model Inside/Outside แบบดั้งเดิม

ตลอดหลักสูตรนี้ (Part 15 และ Step 621-625 ข้างบน) ใช้ Model ดั้งเดิมเสมอ: ทุก Interface ต้องถูก
Mark ชัดเจนว่าเป็น `ip nat inside` หรือ `ip nat outside` — Model นี้มีข้อจำกัดในบางสถานการณ์:

1. **ต้องรู้ทิศทางล่วงหน้าเสมอ** — ถ้า Topology มี Traffic ที่ทิศทาง Inside/Outside ไม่ตายตัว
   (เช่น NAT ระหว่างสอง VRF ที่ไม่มีฝั่งไหนเป็น "Internet" ชัดเจนแบบ Step 622) การเลือกว่า
   Interface ไหนเป็น Inside/Outside อาจดูขัดกับสามัญสำนึก
2. **Routing ที่ซับซ้อน (Multiple Path, VRF)** อาจทำให้ Interface เดียวต้องทำหน้าที่ทั้ง Inside
   และ Outside พร้อมกันสำหรับ Traffic ต่างกลุ่ม — Model ดั้งเดิมทำแบบนั้นไม่ได้ (1 Interface
   = 1 Role เท่านั้น)

### 627.2 แนวคิดของ NVI

**NVI (NAT Virtual Interface)** เปลี่ยนวิธีคิดทั้งหมด: **ไม่มี Inside/Outside อีกต่อไป** —
ทุก Interface ที่เกี่ยวข้องกับ NAT ใช้คำสั่งเดียวกันหมด (`ip nat enable`) แล้ว IOS จะตัดสินใจ
ทิศทางการแปลง Address เอง **ตาม Routing Decision ของ Packet นั้นๆ ณ ขณะนั้น** ไม่ใช่ตาม Role
ที่ Fix ไว้ล่วงหน้าบน Interface

### 627.3 Syntax เปรียบเทียบ

| รูปแบบ | Model ดั้งเดิม (ใช้ในหลักสูตรนี้) | NVI (ทางเลือกสมัยใหม่) |
|---|---|---|
| Mark Interface | `ip nat inside` / `ip nat outside` | `ip nat enable` (ทุก Interface ใช้คำสั่งเดียวกัน) |
| Static NAT | `ip nat inside source static <local> <global>` | `ip nat source static <local> <global>` (ไม่มี inside/outside นำหน้า) |
| Dynamic/PAT | `ip nat inside source list <acl> ... overload` | `ip nat source list <acl> ... overload` |
| ดู Translation Table | `show ip nat translations` | `show ip nat nvi translations` |
| ดู Statistics | `show ip nat statistics` | `show ip nat nvi statistics` |

### 627.4 Worked Example — แปลง Static NAT ของ Server1 (Part 15) เป็นแบบ NVI

เพื่อการเปรียบเทียบเท่านั้น (**หลักสูตรนี้ยังคงใช้ Model ดั้งเดิมกับ `WAN-EDGE-1` ทุก Step
เพื่อความสอดคล้องกับ Part 15**) — ถ้าจะเขียนแบบ NVI จะมีรูปหน้าตาดังนี้:

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# ip nat enable
WAN-EDGE-1(config-if)# exit
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# ip nat enable
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# ip nat source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
```

สังเกตว่า**ทั้งสอง Interface ใช้คำสั่งเดียวกัน (`ip nat enable`)** ไม่มีการระบุว่าใครเป็น
Inside ใครเป็น Outside เลย — IOS จะดูจาก Routing Table ว่า Traffic วิ่งไปทางไหนแล้วเลือก
ทิศทางแปลงให้เองโดยอัตโนมัติ

### 627.5 เมื่อไรควรใช้ NVI จริง

NVI เหมาะกับสถานการณ์ที่ Model ดั้งเดิมทำได้ยาก โดยเฉพาะ **NAT ที่เกี่ยวข้องกับ VRF หลายตัว
พร้อมกัน** (ตรงกับ Step 622 ข้างบน — ถ้า WAN-EDGE-1 ต้องทำ NAT ให้ทั้ง Global Table และ VRF
`GUEST` ผ่าน Interface ที่มีบทบาทไม่ตายตัว NVI จะลด Config ที่ต้องคิดเรื่อง Inside/Outside ลง
ได้มาก) — อย่างไรก็ตาม **หลักสูตรนี้เลือกใช้ Model ดั้งเดิมตลอดเพื่อความสอดคล้องกับ Part 15**
และเพราะ Topology ของเรามี Inside/Outside ที่ชัดเจนอยู่แล้วในทุกกรณี (LAN ชัดเจนว่าเป็น Inside,
ISP ชัดเจนว่าเป็น Outside) — NVI จึงเป็นความรู้สำหรับข้อสอบ CCNP และสำหรับ Design ที่ซับซ้อน
กว่านี้ มากกว่าจะเป็นสิ่งที่ต้องใช้ในทุก Deployment

---

## Step 628 — Debug NAT เชิงลึก: อ่าน debug ip nat detailed ทีละบรรทัด

### 628.1 ความแตกต่างจาก debug ip nat ธรรมดา (ทวนจาก Part 15 Step 149.5)

`debug ip nat` (ธรรมดา) แสดงแค่ **ผลลัพธ์สุดท้าย** ของการแปลง (`s=...->...`) — `debug ip nat
detailed` แสดง **กระบวนการตัดสินใจทุกขั้นตอน** ก่อนถึงผลลัพธ์นั้น ทำให้เห็นว่า IOS "คิดอะไร"
ในแต่ละ Step ซึ่งจำเป็นเวลา Troubleshoot กรณีที่ซับซ้อนกว่าการแปลงสำเร็จ/ไม่สำเร็จแบบตรงไปตรงมา

### 628.2 อ่าน Trace ของ Session ที่สำเร็จทีละบรรทัด

```
WAN-EDGE-1# debug ip nat detailed
IP NAT detailed debugging is on
WAN-EDGE-1#
*Sep 26 11:15:02.115: NAT: i: tcp (10.10.10.51, 52301) -> (203.0.113.2, 52301) [top]
*Sep 26 11:15:02.116: NAT: found dynamic entry for src, insert directly, addr 203.0.113.2, port 52301
*Sep 26 11:15:02.117: NAT*: TCP s=10.10.10.51->203.0.113.2, d=198.51.100.20 [52301]
*Sep 26 11:15:02.140: NAT: o: tcp (198.51.100.20, 443) -> (10.10.10.51, 52301) [top]
*Sep 26 11:15:02.141: NAT: match found using entry 203.0.113.2:52301 10.10.10.51:52301 198.51.100.20:443 198.51.100.20:443
*Sep 26 11:15:02.142: NAT*: TCP s=198.51.100.20, d=203.0.113.2->10.10.10.51 [52301]
```

| บรรทัด | ความหมาย |
|---|---|
| `NAT: i: tcp (10.10.10.51, 52301) -> (203.0.113.2, 52301)` | **`i:`** = Inbound-to-NAT-Engine (Packet เข้าสู่กระบวนการพิจารณา NAT จากฝั่ง Inside) — กำลังจะแปลง Source `10.10.10.51:52301` เป็น `203.0.113.2:52301` |
| `NAT: found dynamic entry for src, insert directly` | IOS หา Entry ใน Table ไม่พบ (Session ใหม่) จึง **สร้าง Entry ใหม่ทันที** ด้วย Port ที่ระบุ (แสดงว่า Port ยังว่างอยู่ ไม่มีปัญหา Exhaustion) |
| `NAT*: TCP s=10.10.10.51->203.0.113.2, d=198.51.100.20 [52301]` | ผลลัพธ์สุดท้ายแบบเดียวกับ `debug ip nat` ธรรมดา — Source ถูกแปลงสำเร็จ |
| `NAT: o: tcp (198.51.100.20, 443) -> (10.10.10.51, 52301)` | **`o:`** = Outbound-from-NAT-Engine มุมมอง (Packet ขากลับที่เข้ามาทาง Outside กำลังจะถูกแปลง Destination กลับ) |
| `NAT: match found using entry ...` | IOS **ค้นเจอ Entry เดิม** ที่ Session ขาไปสร้างไว้ (Match ทั้ง 4 Field: Inside Global/Local + Outside Local/Global) — ยืนยันว่า Return Traffic วิ่งผ่าน Router ตัวเดียวกับ Session ขาไป (Symmetric — ไม่มีปัญหาแบบ Step 625) |
| `NAT*: TCP s=198.51.100.20, d=203.0.113.2->10.10.10.51 [52301]` | Destination ถูกแปลงกลับสำเร็จ — Session สมบูรณ์ทั้งไปและกลับ |

### 628.3 อ่าน Trace ของ Session ที่ล้มเหลว (เทียบกับ Step 623)

```
*Sep 26 11:16:40.881: NAT: i: tcp (10.10.40.203, 51820) -> (203.0.113.2, ?)
*Sep 26 11:16:40.881: NAT: no available port in pool/interface for translation
*Sep 26 11:16:40.882: NAT*: translation failed, dropping packet s=10.10.40.203:51820 d=93.184.216.34:443
```

สังเกตว่าบรรทัดที่ 2 **ไม่มี "found dynamic entry" หรือ "insert directly"** เหมือน Trace ที่
สำเร็จ — ขึ้น `no available port` ตรงตัวทันที นี่คือจุดที่แยก "NAT Config ผิด" ออกจาก "NAT
Resource หมด" ได้ชัดเจนที่สุด: ถ้า Config ผิด (เช่น ACL ไม่ Match) Trace จะไม่มีบรรทัด `i:`
ปรากฏขึ้นเลยตั้งแต่ต้น (เพราะ Packet ไม่ถูกพิจารณาว่าต้อง NAT) แต่ถ้า Resource หมด Trace จะเห็น
บรรทัด `i:` (เข้าสู่กระบวนการ) แต่ **ล้มเหลวตอนหา Port**

### 628.4 จำกัดขอบเขต Debug เพื่อไม่ให้กิน CPU (ต่อจากคำเตือน Part 15 Step 149.5)

`debug ip nat detailed` กิน CPU **มากกว่า** `debug ip nat` ธรรมดาอย่างมีนัยสำคัญ (ต้อง Print
Log หลายบรรทัดต่อ Packet) — บน Production **ห้ามเปิดแบบไม่จำกัดขอบเขตเด็ดขาด** ใช้ Keyword
`list <acl>` จำกัดให้ Debug แสดงผลเฉพาะ Traffic ที่สนใจจริงๆ:

```
WAN-EDGE-1(config)# ip access-list extended DEBUG-FILTER
WAN-EDGE-1(config-ext-nacl)# permit tcp host 10.10.40.203 any
WAN-EDGE-1(config-ext-nacl)# exit
WAN-EDGE-1# debug ip nat detailed list DEBUG-FILTER
```

และปิดทันทีหลังใช้งานเสร็จเสมอ:

```
WAN-EDGE-1# undebug all
```

---

## Step 629 — Performance & Scale ของ NAT: Process-Switching, CEF Punt, และ CGN

### 629.1 CEF (Fast Path) vs Process-Switching สำหรับ NAT

Cisco IOS พยายามทำ NAT ผ่าน **CEF (Cisco Express Forwarding) Fast Path** เสมอเมื่อเป็นไปได้
(Hardware/ASIC-Accelerated บนอุปกรณ์ที่รองรับ) — แต่มีบางสถานการณ์ที่ NAT **ต้อง Process-Switch
เสมอ** (ตกไปที่ CPU ตัดสินใจทีละ Packet ช้ากว่า Fast Path มาก):

| สถานการณ์ที่ทำให้ NAT ต้อง Process-Switch | เหตุผล |
|---|---|
| Packet ที่ถูก Fragment (IP Fragmentation) | ต้องรอ Fragment ทั้งหมดมารวมก่อนแปลง Port ได้ (Port อยู่แค่ Fragment แรก) |
| Traffic ที่ต้องใช้ **ALG (Application Layer Gateway)** เช่น FTP Active Mode, SIP, H.323 | ต้องอ่าน/แก้ไข Payload ไม่ใช่แค่ Header — งานหนักเกินกว่า Fast Path จะทำได้ |
| Static NAT แบบ `network` (Step 621) ร่วมกับ ACL ที่ซับซ้อนมาก | เพิ่ม Overhead การตัดสินใจต่อ Packet |
| ICMP บางประเภทที่ต้องแปลง Payload ภายใน (เช่น ICMP Error ที่ Embed Header เดิมไว้ข้างใน) | ต้องแก้ไขข้อมูลที่ซ้อนอยู่ลึกกว่าปกติ |

### 629.2 อ่านค่า CEF Translated/Punted จาก show ip nat statistics

```
WAN-EDGE-1# show ip nat statistics
...
CEF Translated packets: 8823012, CEF Punted packets: 118
```

- **CEF Translated packets**: จำนวน Packet ที่ NAT ผ่าน Fast Path สำเร็จ (ปกติควรเป็นตัวเลข
  ส่วนใหญ่เกือบทั้งหมด)
- **CEF Punted packets**: จำนวน Packet ที่ต้อง "Punt" (ส่งขึ้นไปให้ CPU/Process-Switch จัดการ
  แทน) — ถ้าตัวเลขนี้ **สูงเทียบเป็นสัดส่วนกับ Translated** (เช่นเกิน 1-2% ของ Total Traffic)
  ควรตรวจสอบว่ามี ALG หรือ Fragment จำนวนมากผิดปกติหรือไม่ เพราะจะกิน CPU สูงกว่าปริมาณ
  Bandwidth ที่ดูเหมือนน้อยมาก

### 629.3 ผลกระทบต่อ CPU ที่ Scale สูง

Enterprise ขนาดกลาง (แบบ Lab หลักสูตรนี้) แทบไม่ชน Limit นี้เลยเพราะปริมาณ ALG Traffic
(FTP/SIP) เป็นสัดส่วนเล็กมากของ Traffic รวม — แต่องค์กรที่มี Traffic ประเภทนี้เป็นสัดส่วนสูง
(เช่น Call Center ที่ใช้ SIP Trunk จำนวนมากผ่าน NAT เดียวกัน) ควรพิจารณาแยก ALG Traffic ออกไปใช้
Path เฉพาะ (เช่น VPN/Direct Peering ที่ไม่ต้อง NAT เลย) เพื่อลด Process-Switching Load บน Router
ตัวเดียวกันกับ Traffic ทั่วไป

### 629.4 CGN (Carrier-Grade NAT) — Exam Awareness สำหรับระดับ ISP

**CGN** (หรือเรียก **NAT44** เมื่อเทียบกับ NAT64) คือ NAT ที่ ISP ทำเพื่อแก้ปัญหา IPv4
Address Exhaustion ของ**ตัวเอง** — ต่างจาก Enterprise NAT (แบบ `WAN-EDGE-1` ในหลักสูตรนี้) ที่
แปลง Private ↔ Public IP **ครั้งเดียว**, CGN ทำให้ Customer หลายพัน/หลายหมื่นราย **ที่แต่ละราย
ก็มี Private/RFC 1918 Address ของตัวเองอยู่แล้ว (Router บ้านทำ PAT ของตัวเองรอบแรก) ถูก NAT
ซ้ำอีกรอบที่ ISP** ก่อนออก Internet จริง (Double NAT)

| คุณสมบัติ | Enterprise PAT (หลักสูตรนี้) | CGN (ISP-Scale) |
|---|---|---|
| จำนวน Session ที่ต้องรองรับ | หลักพัน-หลักหมื่น Session | หลักสิบ-หลักร้อยล้าน Session |
| Address Range ที่ใช้ตรงกลาง | Public IP ปกติ | **RFC 6598 Shared Address Space (`100.64.0.0/10`)** — จองไว้เฉพาะสำหรับ CGN ไม่ใช่ RFC 1918 |
| Hardware ที่ใช้จริง | ISR/CSR ทั่วไป (Software-based) | Platform เฉพาะ (ASR1000 + CGv6, หรือ ASIC/Network Processor เฉพาะ) เพื่อรองรับ Scale ระดับสิบล้าน Session พร้อม Logging ตามกฎหมาย |
| ปัญหาที่ตามมา | น้อย (Defense-in-Depth ยังทำได้ปกติ) | **Double NAT ทำให้ Peer-to-Peer/Gaming/บาง VPN ใช้งานยากขึ้น**, ต้องเก็บ Log การแปลง Address จำนวนมหาศาลตามข้อกำหนดทางกฎหมายในหลายประเทศ |

> **สรุปสำหรับข้อสอบ**: CGN ไม่ใช่ Feature ที่ Enterprise ทั่วไปต้อง Config เอง (เป็นเรื่องของ
> ISP) แต่ CCNP ต้องเข้าใจว่ามันมีอยู่และมีผลกระทบต่อ Traffic ของลูกค้าปลายทาง (เช่น ถ้า Customer
> อยู่หลัง CGN ของ ISP, Static NAT/Port Forwarding ที่ Customer ทำเองที่ Router บ้านจะใช้งานไม่
> ได้เลย เพราะ Public IP ที่ Router บ้านได้รับจริงๆ เป็นแค่ Address ภายในของ CGN ไม่ใช่ Public
> IP จริงที่ Routable บน Internet)

---

## Step 630 — Lab เต็มรูปแบบ: แก้ปัญหา Exhaustion + NAT/ACL Ordering พร้อมกันบน WAN-EDGE-1

### 630.1 โจทย์ของ Lab

`WAN-EDGE-1` มีปัญหาซ้อนกัน **2 อย่างพร้อมกัน** ที่ทีม NOC รายงานเข้ามาในเช้าวันเดียวกัน:

1. User จาก VLAN 40 (WIFI) เปิด Connection ใหม่ไม่ได้ (ปัญหาแบบ Step 623 — Translation
   Exhaustion) แต่ VLAN อื่นยังใช้งานได้ตามปกติ
2. Partner ภายนอก (Range `198.51.100.0/24`) รายงานว่าเข้าถึง Server1 HTTPS ไม่ได้เลย ทั้งที่
   เมื่อวานยังใช้งานได้ (ปัญหาแบบ Step 624 — NAT/ACL Order of Operations)

### 630.2 Running-Config "ก่อนแก้" (Before) — จำลองสภาพที่มีปัญหาทั้งสองอย่าง

```
hostname WAN-EDGE-1
!
ip nat translation timeout 86400
ip nat translation tcp-timeout 86400
ip nat translation udp-timeout 300
!
interface GigabitEthernet0/0/0
 description ** OUTSIDE ** to ISP-RTR
 ip address 203.0.113.2 255.255.255.252
 ip access-group OUTSIDE-IN-ACL in
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** INSIDE ** to CORE-SW1
 ip address 10.10.254.1 255.255.255.252
 ip access-group SERVER1-PARTNER-ACL out
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** INSIDE ** cross-link to WAN-EDGE-2
 ip address 10.10.254.9 255.255.255.252
 ip nat inside
 no shutdown
!
ip access-list standard NAT-INTERNAL-ACL
 permit 10.10.10.0 0.0.0.255
 permit 10.10.20.0 0.0.0.255
 permit 10.10.30.0 0.0.0.255
 permit 10.10.40.0 0.0.0.255
!
ip access-list extended OUTSIDE-IN-ACL
 permit tcp any host 203.0.113.10 eq 443
 deny ip any host 203.0.113.10
 permit icmp any host 203.0.113.2 echo-reply
 deny ip any any log
!
ip access-list extended SERVER1-PARTNER-ACL
 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443
 deny ip any host 10.10.30.10 log
 permit ip any any
!
ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
ip nat inside source list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 overload
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### 630.3 ขั้นการวินิจฉัย (ตามลำดับที่ Engineer ควรทำจริง)

```
! 1) ตรวจสอบ Translation Table โดยรวมก่อน
WAN-EDGE-1# show ip nat statistics
Total translations: 63988 (1 static, 63987 dynamic; 63987 extended)
Hits: 9012344  Misses: 5920
```

`Total translations` ใกล้เต็ม Pool + `Misses` สูง → ยืนยันปัญหาที่ 1 (Exhaustion) ตรงกับ
Step 623

```
! 2) ตรวจสอบ NAT ของ Server1 (ยืนยันว่า Static NAT ยังทำงานถูกต้อง)
WAN-EDGE-1# show ip nat translations | include 203.0.113.10
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---

! 3) ตรวจสอบ ACL บน Inside Interface ที่ทีม Security เพิ่มมาโดยไม่รู้ผลกระทบ
WAN-EDGE-1# show access-lists SERVER1-PARTNER-ACL
Extended IP access list SERVER1-PARTNER-ACL
    10 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443 (0 matches)
    20 deny ip any host 10.10.30.10 log (412 matches)
    30 permit ip any any
```

`(0 matches)` บนบรรทัด `permit` ตัวแรก แต่ `(412 matches)` บนบรรทัด `deny` — ยืนยันปัญหาที่ 2
(NAT/ACL Ordering) ตรงกับ Step 624 ทุกประการ: ACL เขียนด้วย Public IP บน Interface ที่เห็น
Private IP แล้ว จึงไม่มีวัน Match บรรทัดที่ตั้งใจปล่อยผ่าน

### 630.4 Running-Config "หลังแก้" (After) — แก้ทั้งสองปัญหาพร้อมกัน

```
hostname WAN-EDGE-1
!
ip nat translation timeout 3600
ip nat translation tcp-timeout 3600
ip nat translation udp-timeout 120
ip nat translation finrst-timeout 5
!
ip nat pool INTERNET-PAT-POOL 203.0.113.9 203.0.113.11 netmask 255.255.255.248
!
interface GigabitEthernet0/0/0
 description ** OUTSIDE ** to ISP-RTR
 ip address 203.0.113.2 255.255.255.252
 ip access-group OUTSIDE-IN-ACL in
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** INSIDE ** to CORE-SW1
 ip address 10.10.254.1 255.255.255.252
 ip nat inside
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** INSIDE ** cross-link to WAN-EDGE-2
 ip address 10.10.254.9 255.255.255.252
 ip nat inside
 no shutdown
!
ip access-list standard NAT-INTERNAL-ACL
 permit 10.10.10.0 0.0.0.255
 permit 10.10.20.0 0.0.0.255
 permit 10.10.30.0 0.0.0.255
 permit 10.10.40.0 0.0.0.255
!
ip access-list extended OUTSIDE-IN-ACL
 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443
 deny ip any host 203.0.113.10
 permit icmp any host 203.0.113.2 echo-reply
 deny ip any any log
!
ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
ip nat inside source list NAT-INTERNAL-ACL pool INTERNET-PAT-POOL overload
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**สรุปการเปลี่ยนแปลง**:

1. **แก้ Exhaustion (Step 623)**: เปลี่ยนจาก `interface GigabitEthernet0/0/0 overload`
   (1 Public IP) เป็น `pool INTERNET-PAT-POOL overload` (3 Public IP) + ลด Timeout ทั้งหมด
2. **แก้ NAT/ACL Ordering (Step 624)**: ลบ `SERVER1-PARTNER-ACL` และ `ip access-group ...
   out` ออกจาก `Gi0/0/1` ทั้งหมด แล้วย้าย Requirement ของ Partner Range เข้าไปรวมกับ
   `OUTSIDE-IN-ACL` เดิม (จุดเดียวที่ถูกต้องตาม Order of Operations) — เหลือ ACL ที่ทำหน้าที่
   นี้แค่จุดเดียวในระบบ

### 630.5 Verification เต็มรูปแบบหลังแก้

```
! Exhaustion หายไปแล้ว
WAN-EDGE-1# show ip nat statistics
Total translations: 890 (1 static, 889 dynamic; 889 extended)
Peak translations: 64021, occurred 1d04h ago
Hits: 9102211  Misses: 0
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1, GigabitEthernet0/0/2

! User VLAN 40 เปิด Connection ใหม่ได้ตามปกติ
WiFi-Client40> curl -I http://93.184.216.34
HTTP/1.1 200 OK

! Partner เข้าถึง Server1 HTTPS ได้แล้ว
Partner-Host> curl -I https://203.0.113.10
HTTP/1.1 200 OK

WAN-EDGE-1# show access-lists OUTSIDE-IN-ACL
Extended IP access list OUTSIDE-IN-ACL
    10 permit tcp 198.51.100.0 0.0.0.255 host 203.0.113.10 eq 443 (318 matches)
    20 deny ip any host 203.0.113.10 (0 matches)
    30 permit icmp any host 203.0.113.2 echo-reply
    40 deny ip any any log

! ยืนยันว่า Non-Partner Range ยังถูกบล็อกตามเดิม (ไม่ได้เปิดกว้างเกินความจำเป็นระหว่างแก้ปัญหา)
Random-Internet-Host> curl -I https://203.0.113.10
curl: (7) Failed to connect: Connection timed out
```

### 630.6 สรุปผล Lab

| ปัญหา | สาเหตุ | วิธีแก้ | ผลตรวจสอบ |
|---|---|---|---|
| PAT ล้มเหลวเฉพาะ Connection ใหม่ | Port Space ของ PAT ใกล้เต็ม (1 Public IP + Timeout ยาวเกินไป) | ขยาย Pool เป็น 3 Public IP + ลด Timeout ทั้งหมด | `Misses: 0`, `Total translations` กลับสู่ระดับปกติ |
| Server1 HTTPS เข้าถึงไม่ได้จาก Partner | ACL ใหม่ Apply บน Inside Interface `out` แต่เขียนด้วย Public IP (ประเมิน Address ผิดจุดตาม NAT Order of Operations) | รวม Policy กลับไปที่ `OUTSIDE-IN-ACL` จุดเดียว (Public IP ถูกต้องตามธรรมชาติของจุดนั้น) | `curl` จาก Partner สำเร็จ, Non-Partner ยังถูกบล็อกตามเดิม |

---

## แบบฝึกหัดทวนความเข้าใจ Part 63

1. ในสถานการณ์ Overlapping Address (Step 621) ทำไมต้องใช้ทั้ง `ip nat inside source static
   network` และ `ip nat outside source static network` พร้อมกัน — ถ้าใช้แค่ตัวเดียวจะเกิด
   ปัญหาอะไร?
2. คำสั่ง `ip nat inside source list <acl> pool <pool> vrf <vrf-name> overload` (Step 622)
   ต่างจาก Static Route Leaking แบบ `ip route vrf ... global` (Part 51) อย่างไร และข้อดีของ
   วิธีแรกคืออะไร?
3. อธิบายว่าทำไม `show ip nat statistics` ที่มี `Misses` สูงขึ้นเรื่อยๆ พร้อมกับ `Total
   translations` ใกล้ค่าสูงสุด บ่งบอกถึงปัญหาประเภทใด และมีวิธีแก้ 2 แนวทางอะไรบ้าง?
4. ในกับดัก NAT/ACL Order of Operations (Step 624) เพราะเหตุใด ACL ที่ apply บน **Inside
   Interface ทิศทาง `out`** ต้องเขียนด้วย Private IP ไม่ใช่ Public IP สำหรับ Traffic ขาเข้า
   (Outside → Inside)?
5. อธิบายว่าทำไม Equal-Cost Default Route (ECMP) ไปยัง `WAN-EDGE-1` และ `WAN-EDGE-2` พร้อมกัน
   ถึงทำให้ NAT พังได้ (Step 625) และวิธีแก้ 2 แนวทางที่กล่าวถึงคืออะไร?

**เฉลย:**

1. เพราะ Overlapping Address ต้องแก้ปัญหาทั้งสองทิศทาง — ถ้าใช้แค่ `inside source static
   network` (แปลง Address ของเราเท่านั้น) Routing Table ของเรายังมี Prefix `10.10.10.0/24`
   ชนกันสองความหมายอยู่ดี (ของเราเองและของ Acquired Corp) ยังส่ง Traffic ไปหา Acquired Corp
   ไม่ได้เลยเพราะไม่รู้จะ Route ไปทางไหน ต้องมี `outside source static network` แปลง Address
   ของ Acquired Corp ให้เป็น Prefix ที่ไม่ชนกันด้วย จึงจะ Route ได้ทั้งสองทิศทาง
2. Static Route Leak เปิดทางให้ VRF มองเห็น Global Table ทั้งก้อนผ่าน Default Route ที่ Leak
   มา (ต้องพึ่ง ACL ป้องกันซ้ำ) ส่วน NAT+VRF Keyword ทำให้ Global Table ไม่รู้จัก VRF นั้นเลย
   — Isolation แน่นกว่าเพราะ NAT เป็นจุดควบคุมทางออกเดียวของ VRF โดยไม่ต้องเปิดทางแบบ Default
   Route กว้างๆ
3. บ่งบอกถึง **Translation Table/Port Exhaustion** — Port Space ของ PAT ผ่าน 1 Public IP
   เกือบเต็ม ทำให้ Connection ใหม่หา Port ว่างไม่ได้ (แต่ Connection เดิมที่มี Entry อยู่แล้ว
   ยังทำงานปกติ) วิธีแก้: (1) เพิ่ม Public IP เข้า Pool เพื่อขยาย Port Space, (2) ลด NAT
   Translation Timeout เพื่อปล่อย Port คืนเร็วขึ้นจาก Session ที่ตายแล้วแต่ยังค้างอยู่
4. เพราะตาม NAT Order of Operations ของ Traffic ขาเข้า (Outside → Inside) **NAT แปลง
   Destination กลับเป็น Private IP ก่อน Routing Decision เสมอ** — เมื่อ Packet เดินทางมาถึง
   Inside Interface (เพื่อออกไปทาง `out`) มันผ่าน NAT ไปแล้ว Destination จึงเป็น Private IP
   แล้วเสมอ ณ จุดนั้น ACL ที่เขียนด้วย Public IP จะไม่มีวัน Match Packet ที่จุดนี้ได้เลย
5. เพราะ NAT เป็น Stateful ต่อ Router ตัวเดียว (ไม่มี State Sync ระหว่าง Router) แต่ ECMP ทำให้
   CEF เลือก Next-Hop สลับไปมาระหว่างสอง Router สำหรับ Packet ของ Session เดียวกันได้ — ถ้า
   Packet บางส่วนของ Session ไปทาง Router ที่ไม่มี NAT Entry ของ Session นั้น Packet จะถูก Drop
   หรือถูกแปลงด้วย Source IP คนละตัว ทำให้ Session พัง วิธีแก้: (1) ตัด ECMP ออกด้วย Floating
   Static Route (Administrative Distance ต่างกัน) ให้เหลือ Path Active ทีละเส้นทางเท่านั้น,
   (2) ใช้ PBR ร่วมกับ IP SLA Tracking (`set ip next-hop verify-availability`) เพื่อบังคับ
   Symmetric Path อย่างเข้มงวดกว่า Routing Table ปกติ

---

## สรุป Part 63

Part นี้ยกระดับ NAT จากพื้นฐานที่ [Part 15](part-015-nat-pat.md) วางไว้ ขึ้นสู่สถานการณ์ขั้นสูง
ระดับ CCNP ENARSI ครบทั้งสองด้านตามที่ Blueprint คาดหวัง — **ด้าน Advanced Scenario**: Overlapping
Address Space หลัง M&A ด้วย Double-Sided Static Network NAT, และการผสาน NAT เข้ากับ VRF-Lite
(ต่อจาก [Part 51](part-051-vrf-lite-gre-ipsec.md)) ผ่าน Keyword `vrf` ที่ให้ Isolation แน่นกว่า
Static Route Leak เพียงอย่างเดียว **ด้าน Troubleshooting**: Translation/Port Exhaustion,
กับดัก NAT/ACL Order of Operations แบบคลาสสิก, และ Asymmetric NAT ที่เชื่อมกับแนวคิด PBR/IP SLA
Tracking จาก [Part 32](part-032-policy-based-routing.md) ปิดท้ายด้วยความรู้ระดับ Exam Awareness
(NAT64, NVI, Process-Switching/CGN) และ Lab เต็มรูปแบบที่รวมปัญหาสองอย่างเข้าด้วยกันบน
`WAN-EDGE-1` เพื่อฝึกวินิจฉัยหลายปัญหาซ้อนกันพร้อมกันแบบที่พบได้จริงในสถานการณ์ทำงาน

**ไปต่อ:** [Part 64 — Path Control & PBR Troubleshooting →](part-064-path-control-pbr-troubleshooting.md)
