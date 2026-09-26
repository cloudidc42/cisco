# Part 61 — Infrastructure Security Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE (CCNP ENARSI) | Step 601–610 จาก 1000**

> ต่อจาก [Part 60 — DMVPN & VPN Troubleshooting](part-060-dmvpn-vpn-troubleshooting.md) ที่เราไล่
> Diagnose ปัญหาของ Overlay VPN (DMVPN Phase 3, NHRP, IPsec) กันมาอย่างละเอียด Part นี้ขยับกลับมาที่
> **ฟีเจอร์ความปลอดภัยของ Infrastructure ที่เรา Config ไว้ตลอดหลักสูตรระดับ CCNP** — Port Security,
> DHCP Snooping, DAI ([Part 21](part-021-switch-security.md)), TACACS+ AAA
> ([Part 22](part-022-aaa-device-hardening.md)), 802.1X/TrustSec
> ([Part 44](part-044-dot1x-trustsec.md)), ZBFW ([Part 45](part-045-advanced-acl-zbfw.md)), และ CoPP
> ([Part 46](part-046-control-plane-policing.md)) — ทุกฟีเจอร์เหล่านี้ทำงานถูกต้องดีตอน Deploy ครั้ง
> แรก แต่ในสนามจริง **การเปลี่ยนแปลง Network เพียงเล็กน้อย** (เพิ่ม Server ใหม่ 1 ตัว, ISE Reboot
> เพื่อ Patch, Link Flap ชั่วขณะ) มักทำให้ฟีเจอร์เหล่านี้ **"ทำงานถูกต้องตามที่ Config ไว้" แต่ผลลัพธ์
> กลับกลายเป็นบล็อก Traffic ที่ถูกต้องตามกฎหมาย** — Part นี้คือ Playbook เต็มรูปแบบสำหรับแยกแยะ
> "Security ทำงานถูกต้องแต่ Config ไม่ครอบคลุม Scenario ใหม่" ออกจาก "Security พังจริง" ในสไตล์
> ข้อสอบ ENARSI (Ticket-based Troubleshooting)

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 601 | Infrastructure Security Troubleshooting Methodology — อันตรายของการ "ปิดฟีเจอร์เพื่อแก้ปัญหา" |
| 602 | Scenario 1 — DHCP Snooping บล็อก Server ใหม่ เพราะ Uplink ไม่ได้ Mark Trusted (Part 21) |
| 603 | Scenario 2 — Dynamic ARP Inspection บล็อก Server Static IP เพราะไม่มี ARP ACL Exception (Part 21) |
| 604 | Scenario 3 — 802.1X ค้างที่ Unauthorized เพราะ ISE ล่ม + Critical VLAN พิมพ์ผิด (Part 44) |
| 605 | Scenario 4 — TrustSec SGACL บล็อก Traffic ที่ควรผ่าน เพราะ SXP Propagation ล้มเหลว (Part 44) |
| 606 | Scenario 5 — TACACS+ AAA Lockout เพราะ Server ล่ม + ไม่มี Local Fallback ที่ถูกต้อง (Part 22) |
| 607 | Scenario 6 — ZBFW บล็อก Inter-Zone Traffic ของ Server ใหม่ เพราะ Subnet ไม่อยู่ใน Zone (Part 45) |
| 608 | Scenario 7 — CoPP บล็อก OSPF/BGP Control Traffic จริงระหว่าง Network Event (Part 46) |
| 609 | Scenario 8 — Port Security Violation Shutdown เพราะมี Hub ต่อพ่วงเพิ่มอุปกรณ์เกิน Limit |
| 610 | Lab เต็มรูปแบบ: 3 Layered Outage พร้อมกัน — DHCP Snooping + 802.1X Critical VLAN + CoPP |

---

## Step 601 — Infrastructure Security Troubleshooting Methodology

### 601.1 ทำไม Security Troubleshooting ต่างจาก Routing/Switching Troubleshooting

Part 51-60 ของหลักสูตรนี้ (ENARSI Troubleshooting Track) ส่วนใหญ่ไล่หา **Bug/Misconfiguration** ที่
ทำให้ฟีเจอร์ "ไม่ทำงานตามที่ตั้งใจ" (เช่น OSPF ไม่ Convergence, BGP ไม่ขึ้น Established) — แต่ Security
Feature ที่เรียนมาตั้งแต่ Part 21-46 มีธรรมชาติต่างออกไปอย่างสิ้นเชิง: **มันทำงาน "ถูกต้องตามที่ Config
ไว้" เกือบทุกครั้งที่มีปัญหา** — DHCP Snooping Drop Packet เพราะพอร์ตเป็น Untrusted จริง, DAI Drop ARP
เพราะไม่มี Binding จริง, ZBFW Drop Traffic เพราะไม่มี Zone-Pair/Policy ครอบคลุมจริง — ปัญหาไม่ได้อยู่ที่
"ฟีเจอร์พัง" แต่อยู่ที่ **"Config ไม่ได้ตามทันการเปลี่ยนแปลงของ Network"** เช่น มี Server ใหม่, มี VLAN
ใหม่, ISE ล่มชั่วคราว, Link Flap ทำให้ Control Plane Traffic พุ่งสูงกว่าที่ Baseline คาดไว้

```
Routing/Switching Troubleshooting:          Security Troubleshooting:
"ทำไม Feature ไม่ทำงาน?"                      "ทำไม Feature ทำงาน (ถูกต้อง) แต่ Traffic
                                              ที่ควรผ่านกลับไม่ผ่าน?"

→ หา Bug/Neighbor ไม่ขึ้น/Route หาย           → หา "Gap" ระหว่าง Design เดิม กับ
                                                Traffic/Topology ปัจจุบันที่เปลี่ยนไป
```

### 601.2 อันตรายที่สุดของ Security Troubleshooting: "แก้ปัญหา" ด้วยการปิดฟีเจอร์ทิ้ง

เมื่อ Helpdesk/Junior Engineer เจอ Traffic ที่ถูกบล็อกโดยฟีเจอร์ Security และกำลังถูกกดดันให้แก้ไข
"เดี๋ยวนี้" (User โทรมาต่อว่า, Manager เร่ง) — ทางลัดที่ **ดูเหมือนได้ผลทันที** แต่เป็นหายนะระยะยาวคือ
การปิดฟีเจอร์ Security ทั้งหมดทิ้งไปเลย แทนที่จะหาว่า "ทำไม Traffic ที่ถูกต้องถึงไม่ผ่าน"

| ฟีเจอร์ | ทางลัดที่ "ดูเหมือนแก้ได้" (ผิด) | ผลกระทบที่แท้จริง | สิ่งที่ควรทำแทน |
|---|---|---|---|
| Port Security | `no switchport port-security` ทั้งพอร์ต | เปิดช่องโหว่ MAC Flooding/Rogue Device ทั้งพอร์ตนั้นแบบไม่มีการป้องกันอีกต่อไป | ตรวจ `show port-security interface` หา MAC ที่เกิน Limit จริง แล้วปรับ Maximum/Sticky ให้ตรงกับอุปกรณ์ที่อนุญาตจริง |
| DHCP Snooping | `no ip dhcp snooping` (Global) | เปิดช่องโหว่ Rogue DHCP Server ทั้ง VLAN ทันที — Client ทุกเครื่องเสี่ยง MITM | ตรวจ `show ip dhcp snooping` หาพอร์ตที่ Trust ผิด แล้วแก้ Trust Boundary ให้ตรงกับ Design ([Part 21 Step 204.4](part-021-switch-security.md)) |
| DAI | `no ip arp inspection vlan <n>` | เปิดช่องโหว่ ARP Spoofing/MITM ทั้ง VLAN | ตรวจ `show ip arp inspection statistics` หา Drop Reason จริง แล้วเพิ่ม ARP ACL Exception ที่ขาดไป |
| 802.1X | `no authentication port-control auto` (เปลี่ยนเป็น force-authorized) | พอร์ตเปิดให้เข้าเครือข่ายได้โดยไม่ต้อง Authenticate เลย ถาวร | ตรวจ `show authentication sessions` หาสถานะ RADIUS Server และ Fallback VLAN ที่ Config ผิด |
| TACACS+ AAA | `aaa authentication login default none` | ปิด Authentication ทั้งอุปกรณ์ — ใครก็ Login ได้โดยไม่ต้องมี Password | ตรวจ `show tacacs`/`show aaa servers` หาสาเหตุที่ Server ไม่ตอบ แล้วแก้ที่ต้นเหตุ (Network/Key/Firewall) |
| ZBFW | `no zone-member security` ที่ Interface (เอา Interface ออกจาก Zone ทั้งหมด) | Traffic ทุกประเภทวิ่งผ่านได้แบบไม่มีการ Inspect เลย ไม่ใช่แค่ Traffic ที่มีปัญหา | ตรวจ `show policy-map type inspect zone-pair` หา Class-map/Zone-Pair ที่ไม่ครอบคลุม Subnet ใหม่ |
| CoPP | `no service-policy input COPP-POLICY` ที่ `control-plane` | ปิดการป้องกัน CPU ทั้งหมด — CPU เสี่ยงพังจาก Traffic ผิดปกติใดๆ ในอนาคต | ตรวจ `show policy-map control-plane` หา Class ที่ Drop จริง แล้วปรับ `cir`/`bc` ของ Class นั้นเท่านั้น |

> **หลักคิดที่ต้องยึดเสมอ**: การปิดฟีเจอร์ Security ทั้งหมดคือการ **"รักษาอาการด้วยการทำลายอวัยวะ"**
> — Traffic ที่ควรผ่านจะผ่านได้จริง แต่ Traffic ที่ไม่ควรผ่านก็จะผ่านได้ด้วย ในขณะที่การ Diagnose ที่
> ถูกต้องจะแก้เฉพาะ "Gap" ที่ทำให้ Traffic ที่ถูกต้องติดขัด โดยที่ Security Posture ส่วนที่เหลือทั้งหมด
> ยังคงทำงานปกติ 100%

### 601.3 5-Step Diagnostic Framework สำหรับ Infrastructure Security

```
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 1: ยืนยันอาการ (Confirm Symptom)                                 │
│  → Traffic/User ไหนที่มีปัญหา, เกิดขึ้นเมื่อไหร่, เกิดกับทุกคนหรือบางคน       │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 2: ระบุว่า Security Control ตัวใดกำลัง Drop (Identify the Control) │
│  → ใช้ show/debug เฉพาะฟีเจอร์ (ไม่ใช่เดา) — ตาราง 601.4 ด้านล่าง          │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 3: ตรวจ Dependency Chain ของ Control นั้น (Trace the Data)        │
│  → Binding Table ว่าง? RADIUS Server ตอบไหม? SGT Map มี Entry ไหม?      │
│    CoPP Counter เพิ่มขึ้นตรงกับเวลาที่มีปัญหาไหม?                          │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 4: แยก Misconfiguration ออกจาก Real Attack                       │
│  → ถ้าเป็น Attack จริง (เช่น ARP Flood, ICMP Flood) ห้ามปิด Control       │
│    เด็ดขาด — Control กำลังทำงานถูกต้องอยู่                              │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 5: แก้ที่จุดที่แคบที่สุด (Minimal Correct Fix) + Verify              │
│  → แก้เฉพาะ Interface/ACL/Class-map ที่ขาด ไม่แก้ทั้ง Policy             │
└─────────────────────────────────────────────────────────────────────┘
```

### 601.4 ตาราง Mapping: อาการ → Show Command แรกที่ควรรัน

| อาการที่ Helpdesk รายงาน | Security Control ที่ต้องสงสัยก่อน | Show Command แรก |
|---|---|---|
| PC/Server ใหม่ไม่ได้ IP จาก DHCP | DHCP Snooping | `show ip dhcp snooping` + `show ip dhcp snooping binding` |
| Host ต่อ Static IP คุยกับใครไม่ได้เลย (ARP ไม่ผ่าน) | Dynamic ARP Inspection | `show ip arp inspection statistics vlan <n>` |
| พอร์ตเข้าเครือข่ายไม่ได้ ค้างที่หน้า Login/ไม่มี IP | 802.1X | `show authentication sessions interface <intf> details` |
| Host คุยข้าม Subnet/Zone ไม่ได้ทั้งที่ ACL/Policy อนุญาต | TrustSec SGACL หรือ ZBFW | `show cts role-based sgt-map` / `show policy-map type inspect zone-pair` |
| Admin Login เข้าอุปกรณ์ไม่ได้เลย | TACACS+ AAA | `show tacacs` / `show aaa servers` |
| OSPF/BGP Neighbor Flap โดยไม่มีปัญหา Physical | CoPP | `show policy-map control-plane` |
| พอร์ตเข้า err-disabled ซ้ำๆ | Port Security | `show port-security interface <intf>` + `show logging \| include PSECURE` |

ตารางนี้คือ "Entry Point" ของทุก Scenario ใน Step 602-609 — Step 610 จะรวมทุกอย่างเข้าด้วยกันเป็น Lab
ที่มี 3 ปัญหาเกิดพร้อมกัน ซึ่งเป็นรูปแบบที่ข้อสอบ ENARSI Ticket-based Simulation ชอบใช้มากที่สุด

---

## Step 602 — Scenario 1: DHCP Snooping บล็อก Server ใหม่ เพราะ Uplink ไม่ได้ Mark Trusted

### 602.1 สถานการณ์

ทีม Infrastructure เปลี่ยน **ACCESS-SW3** ตัวเดิม (เสีย Hardware) ด้วยตัวใหม่ในคืนวันศุกร์ โดย Restore
Running-Config จาก **Backup เก่า** (ก่อน Part 21 Baseline จะถูก Deploy เต็มรูปแบบ) แล้วค่อยเพิ่ม DHCP
Snooping กลับเข้าไปด้วยมือ — เช้าวันจันทร์ ทีม Server เพิ่ม **SERVER-4** (`10.10.30.40`, ตั้งให้ขอ IP
ผ่าน DHCP Reservation จาก DHCP-DNS-SRV เพื่อบริหาร IP รวมศูนย์) เข้าที่ `GigabitEthernet1/0/6` ของ
ACCESS-SW3 — SERVER-4 ขึ้น Error "ไม่มี IP Address" (APIPA `169.254.x.x`) ทันที

### 602.2 Config ที่ผิด (Uplink ไม่ได้ Mark Trusted)

```
! ===== ACCESS-SW3 — Config หลัง Restore (ผิดพลาด) =====
ACCESS-SW3(config)# ip dhcp snooping
ACCESS-SW3(config)# ip dhcp snooping vlan 30,40,99
ACCESS-SW3(config)# ip dhcp snooping information option

! ----- Uplink ไป DIST-SW3 (Gi0/1) และ DIST-SW4 (Gi0/2) -----
! *** บรรทัด "ip dhcp snooping trust" ไม่ได้อยู่ใน Backup เก่า และไม่มีใครเพิ่มกลับมา ***
ACCESS-SW3(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW3(config-if-range)# switchport mode trunk
ACCESS-SW3(config-if-range)# exit

ACCESS-SW3(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW3(config-if-range)# ip dhcp snooping limit rate 15
ACCESS-SW3(config-if-range)# exit
```

### 602.3 Symptom — Show Output

```
ACCESS-SW3# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
30,40,99
Interface                  Trusted     Allow option    Rate limit (pps)
------------------------   -------     ------------    ----------------
GigabitEthernet0/1          no          no                unlimited     <-- ควรเป็น "yes"!
GigabitEthernet0/2          no          no                unlimited     <-- ควรเป็น "yes"!
GigabitEthernet1/0/6        no          no                     15
```

```
ACCESS-SW3# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
Total number of bindings: 0
```

```
! Debug ยืนยันชัดเจนว่า DHCPOFFER/ACK จาก Server ถูก Drop ที่ Uplink
ACCESS-SW3# debug ip dhcp snooping event
*Sep 26 08:41:02: DHCP_SNOOPING: received DHCPOFFER, MAC da: 0010.a5f3.9911, MAC sa:
0010.a5f3.6899, if GigabitEthernet0/1.
*Sep 26 08:41:02: DHCP_SNOOPING: DHCPOFFER received on untrusted port, dropping the packet.
```

### 602.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — SERVER-4 ได้ APIPA แสดงว่า `DHCPDISCOVER` ส่งออกไปได้ (ผ่าน Untrusted Port ได้
   ตามปกติ) แต่ `DHCPOFFER`/`DHCPACK` ขากลับไม่ถึง — เป็น Pattern คลาสสิกของ Trust Boundary ผิด
   ไม่ใช่ Pattern ของ Scope หมด (ถ้า Scope หมด DHCP-DNS-SRV จะไม่ตอบเลยตั้งแต่ต้น)
2. **ระบุ Control** — `show ip dhcp snooping` คือคำสั่งแรกตามตาราง 601.4 — พบว่า `Trusted` ของ
   `Gi0/1`/`Gi0/2` เป็น `no` ทั้งคู่ ทั้งที่ตาม Design ใน [Part 21 Step 204.4](part-021-switch-security.md)
   Uplink ทุกเส้นต้องเป็น Trusted
3. **Trace Dependency** — `debug ip dhcp snooping event` ยืนยันชัดเจนว่า `DHCPOFFER` มาถึง `Gi0/1`
   จริง (มาจาก DIST-SW3 relay) แต่ถูก Drop เพราะพอร์ตเป็น Untrusted — ตรงกับ Binding Table ที่ว่างเปล่า
   สนิท (ไม่มี Transaction ใดสำเร็จเลยสักตัวตั้งแต่เปลี่ยนอุปกรณ์)
4. **แยก Misconfig จาก Attack** — ไม่มีสัญญาณ Rogue DHCP Server (ไม่มี Log `DHCPOFFER received on
   untrusted port` จากพอร์ต Access อื่นที่ไม่ควรมี Server) — เป็น Config Gap ล้วนๆ จากการ Restore
   Backup เก่า ไม่ใช่การโจมตี

### 602.5 Root Cause และ Fix

**Root Cause**: Backup ที่ใช้ Restore ACCESS-SW3 เป็นเวอร์ชันก่อน Part 21 Baseline — Uplink ไป
DIST-SW3/DIST-SW4 ไม่มีบรรทัด `ip dhcp snooping trust`

```
ACCESS-SW3(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW3(config-if-range)# ip dhcp snooping trust
ACCESS-SW3(config-if-range)# exit
```

**Verify:**

```
ACCESS-SW3# show ip dhcp snooping | include Gi0/
GigabitEthernet0/1          yes         yes              unlimited
GigabitEthernet0/2          yes         yes              unlimited

ACCESS-SW3# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
0010.a5f3.6899       10.10.30.40      86400       dhcp-snooping   30    GigabitEthernet1/0/6
Total number of bindings: 1
```

### 602.6 ข้อควรระวัง: ความเสี่ยงจากทิศทางตรงข้าม (Access Port ถูก Mark Trusted โดยผิดพลาด)

ถ้าในทางกลับกัน วิศวกร **Mark Access Port เป็น Trusted "เพื่อแก้ปัญหาเร็วๆ"** (เช่น พิมพ์
`ip dhcp snooping trust` ผิดพอร์ตไปที่ `Gi1/0/6` ของ SERVER-4 เอง โดยเข้าใจผิดว่าพอร์ตปลายทาง Server
ต้อง Trust) — Access Port นั้นจะกลาย  เป็น **ช่องโหว่ Rogue DHCP Server เต็มรูปแบบทันที**: ถ้ามีใคร
เสียบ DHCP Server ปลอมเข้าพอร์ตนั้นในอนาคต (หรือ SERVER-4 เองถูกตั้งให้รัน DHCP Service โดยไม่ตั้งใจ)
`DHCPOFFER` ปลอมจะผ่านได้ทันทีโดยไม่ถูกตรวจสอบเลย — นี่คือเหตุผลที่ **Trust Boundary ต้องอยู่ที่ระดับ
Infrastructure Link เท่านั้น** ([Part 21 Step 204.4](part-021-switch-security.md)) ไม่ใช่ที่ Access
Port ไม่ว่ากรณีใดก็ตาม แม้จะเป็น Server ที่เชื่อถือได้ 100% ก็ตาม

---

## Step 603 — Scenario 2: DAI บล็อก Server Static IP เพราะไม่มี ARP ACL Exception

### 603.1 สถานการณ์

**SERVER-4** (`10.10.30.40`, MAC `0010.a5f3.6899`) จาก Step 602 หลังแก้ DHCP Snooping สำเร็จ ทีม
Server เปลี่ยนใจให้ตั้ง **Static IP** แทน (ตาม Best Practice ของ VLAN 30 SERVERS ที่ระบุไว้ใน
[Part 16](part-016-dhcp-dns.md) และย้ำใน [Part 21 Step 207.2](part-021-switch-security.md)) — หลัง
เปลี่ยนเป็น Static IP แล้ว SERVER-4 **คุยกับ Host อื่นในเครือข่ายไม่ได้เลย** ทั้งที่ก่อนหน้านี้ (ตอนใช้
DHCP) ใช้งานได้ปกติ

### 603.2 Config ที่ขาด (ไม่ได้เพิ่ม MAC ของ SERVER-4 เข้า ARP ACL)

```
! ===== ACCESS-SW3 — ARP ACL เดิมจาก Part 21 Step 207.3 (ไม่ได้แก้ไข) =====
ACCESS-SW3(config)# arp access-list STATIC-SERVERS
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.20 mac host 0010.a5f3.9911
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.21 mac host 0010.a5f3.9922
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.22 mac host 0010.a5f3.9933
! *** ไม่มีบรรทัดของ SERVER-4 (10.10.30.40 / 0010.a5f3.6899) เลย ***
ACCESS-SW3(config-arp-nacl)# exit

ACCESS-SW3(config)# ip arp inspection filter STATIC-SERVERS vlan 30 static
```

### 603.3 Symptom — Show Output

```
SERVER-4# ping 10.10.30.1
Sending 5, 100-byte ICMP Echos to 10.10.30.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

```
ACCESS-SW3# show ip arp inspection statistics vlan 30
 Vlan    Forwarded   Dropped   DHCP Drops   ACL Drops
 ----    ---------   -------   ----------   ---------
   30        94210        41            0           41    <-- ACL Drops เท่านั้น (ไม่มี DHCP Drops)
```

```
ACCESS-SW3# show ip arp inspection log
Interface        Vlan    Sender MAC        Sender IP        Num Pkts   Reason
---------------  ----    -----------------  ---------------  --------   ---------
Gi1/0/6          30      0010.a5f3.6899     10.10.30.40      41         Deny
```

### 603.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — SERVER-4 Ping Default Gateway ไม่ผ่านเลย ทั้งที่ Physical Link/VLAN ถูกต้อง 100%
   (`show interfaces status` ยืนยัน Up/Up, VLAN 30 ถูกต้อง) — อาการแบบนี้ (คุยกับ**ทุกคน**ไม่ได้ ไม่ใช่
   แค่บางปลายทาง) ชี้ไปที่ปัญหาระดับ ARP มากกว่าปัญหา Routing
2. **ระบุ Control** — ตามตาราง 601.4 "Host Static IP คุยกับใครไม่ได้เลย" → สงสัย DAI ก่อน — รัน
   `show ip arp inspection statistics vlan 30` พบ **ACL Drops = 41** เพิ่มขึ้นต่อเนื่อง ขณะที่ DHCP
   Drops = 0 (ยืนยันว่าปัญหาไม่เกี่ยวกับ DHCP Snooping Binding Table เลย เพราะ VLAN 30 ใช้ ARP ACL
   แบบ `static` ล้วนตาม Step 207.3 — ไม่มีการ Fallback ไป Binding Table)
3. **Trace Dependency** — `show ip arp inspection log` ระบุชัดเจนว่า Sender MAC/IP ของ SERVER-4 ถูก
   Deny — เทียบกับ `show arp access-list STATIC-SERVERS` พบว่าไม่มีบรรทัดของ `10.10.30.40` เลย
4. **แยก Misconfig จาก Attack** — MAC/IP ที่ถูก Deny ตรงกับ SERVER-4 ที่ทีม Server เพิ่งประกาศเปลี่ยน
   เป็น Static IP จริง (ไม่ใช่ MAC ปลอมหรือ IP ที่ไม่รู้จัก) — เป็น Config Gap ที่คาดเดาได้ ไม่ใช่การ
   โจมตี ARP Spoofing

### 603.5 Root Cause และ Fix

**Root Cause**: ARP ACL `STATIC-SERVERS` ไม่ได้อัปเดตตาม Server ใหม่ที่เพิ่มเข้ามา — DAI (ที่ตั้งเป็น
`static` mode ไม่ Fallback ไป Binding Table) จึงไม่มีทางรู้จัก IP-to-MAC Binding ของ SERVER-4 เลย

```
ACCESS-SW3(config)# arp access-list STATIC-SERVERS
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.40 mac host 0010.a5f3.6899
ACCESS-SW3(config-arp-nacl)# exit
```

> **ไม่ต้อง** `no ip arp inspection filter ... vlan 30 static` แล้ว Config ใหม่ — ARP ACL ทำงานแบบ
> "Live" ทันทีที่แก้ (เหมือน ACL ทั่วไป) ไม่ต้อง Toggle Filter ใหม่

**Verify:**

```
ACCESS-SW3# show ip arp inspection statistics vlan 30
 Vlan    Forwarded   Dropped   DHCP Drops   ACL Drops
 ----    ---------   -------   ----------   ---------
   30        94266         0            0            0    <-- ไม่มี Drop เพิ่มขึ้นอีก

SERVER-4# ping 10.10.30.1
Success rate is 100 percent (5/5)
```

> **ข้อคิดสำหรับ Checklist งาน Onboarding Server ใหม่**: ทุกครั้งที่มี Server Static IP ใหม่เข้า VLAN
> ที่เปิด DAI แบบ `static` ต้อง **เพิ่ม ARP ACL เป็นส่วนหนึ่งของขั้นตอน Onboarding เสมอ** เหมือนกับที่
> ต้องเพิ่ม Static IP Source Binding ของ IP Source Guard คู่กัน ([Part 21 Step 208.3]
> (part-021-switch-security.md)) มิฉะนั้นจะเจอปัญหาเดิมซ้ำกับ IP Source Guard ด้วยในลำดับถัดมา

---

## Step 604 — Scenario 3: 802.1X ค้างที่ Unauthorized เพราะ ISE ล่ม + Critical VLAN พิมพ์ผิด

### 604.1 สถานการณ์

ทีม Security Patch **ISE-1** (RADIUS Server หลักตาม [Part 44](part-044-dot1x-trustsec.md)) ตอนเที่ยง
คืนวันเสาร์ ตามแผน Maintenance Window — แผนคาดว่าจะใช้เวลา 10 นาที แต่ ISE-1 ใช้เวลานานเกินคาด (Patch
ค้าง ต้อง Rollback) จนถึงเช้าวันจันทร์ ISE-1 ยังไม่กลับมา — พนักงานที่มาเสียบ PC เข้า `ACCESS-SW2`
(Uplink ไป DIST-SW1 เหมือน ACCESS-SW1 ตาม [Access-Layer Mapping](00-ip-address-plan.md)) **เข้า
เครือข่ายไม่ได้เลย** ทั้งที่ Design ไว้แล้วว่าต้องมี Critical VLAN รองรับกรณี ISE ล่ม

### 604.2 Config ที่ผิด — Copy จาก ACCESS-SW1 (Step 436) แต่พิมพ์ VLAN ผิด

```
! ===== ACCESS-SW2 — Copy จาก ACCESS-SW1 Step 436 แต่สลับเลขผิด (91 -> 19) =====
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1-3
ACCESS-SW2(config-if-range)# authentication event server dead action authorize vlan 19
ACCESS-SW2(config-if-range)# authentication event server dead action authorize voice
ACCESS-SW2(config-if-range)# authentication event server alive action reinitialize
ACCESS-SW2(config-if-range)# authentication event no-response action authorize vlan 92
ACCESS-SW2(config-if-range)# authentication event fail retry 2 action authorize vlan 93
ACCESS-SW2(config-if-range)# exit
```

**VLAN 19 ไม่มีอยู่จริงในระบบเลย** — DIST-SW1 มีแค่ VLAN 10, 20, 91 (DOT1X-CRITICAL), 92
(DOT1X-GUEST), 93 (DOT1X-AUTHFAIL), 99 ตาม Design ของ Step 436

### 604.3 Symptom — Show Output

```
ACCESS-SW2# show authentication sessions interface GigabitEthernet1/0/1 details
            Interface:  GigabitEthernet1/0/1
          MAC Address:  0050.5601.bbbb
                Status:  Authz Failed
           Domain:  DATA
      Oper host mode:  multi-domain
     Oper control dir:  both
        Authorized By:  N/A
          Vlan Policy:  N/A
     Session timeout:  N/A
          Common Session ID: 0A0A63020000004512345678
      Acct Session ID:  Unknown
              Handle:  0x91000021
      Current Policy:  POLICY_Gi1/0/1
```

```
! Log ที่ ACCESS-SW2 ระหว่างพยายาม Authorize เข้า Critical VLAN
*Sep 28 08:02:11.334: %AUTHMGR-5-FAIL: Authorization failed for client (0050.5601.bbbb) on
Interface Gi1/0/1 AuditSessionID 0A0A63020000004512345678
*Sep 28 08:02:11.335: %DOT1X-5-ERROR_VLAN_INVALID: Vlan 19 configured for authorization does
not exist. Client will remain unauthorized on Interface Gi1/0/1
```

```
! ยืนยันว่า ISE-1 (และ ISE Server Group ทั้งกลุ่ม) Dead จริง — ไม่ใช่ปัญหาแค่ VLAN
ACCESS-SW2# show aaa servers | include State|Server
RADIUS: id 1, priority 1, host 10.10.99.61, auth-port 1812, acct-port 1813
     State: current UP, duration 43212s, previous duration 0s
     State: current DEAD, duration 3892s, previous duration 172800s   <-- ISE-1 DEAD
```

### 604.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — `show authentication sessions ... details` แสดง `Status: Authz Failed` และ
   `Vlan Policy: N/A` — นี่คือสัญญาณว่า "Switch **พยายาม** Authorize เข้า Fallback VLAN แล้วแต่ล้มเหลว"
   ไม่ใช่แค่ "ยังไม่ได้ลองเลย" (ถ้ายังไม่ลองจะเห็น Status เป็น `Running` หรือ `0x2/Running`)
2. **ระบุ Control** — ตามตาราง 601.4 "ค้างที่หน้า Login/ไม่มี IP" → 802.1X — Log
   `%DOT1X-5-ERROR_VLAN_INVALID: Vlan 19 configured for authorization does not exist` ชี้ตรงจุดว่า
   ปัญหาไม่ใช่ที่ RADIUS Reject/Timeout ปกติ แต่เป็นที่ **VLAN ปลายทางของ Fallback Action เอง**
3. **Trace Dependency** — `show aaa servers` ยืนยันว่า ISE-1 อยู่ในสถานะ `DEAD` จริง (ตรงกับสถานการณ์
   Maintenance ที่ยังไม่เสร็จ) แปลว่า `authentication event server dead action authorize vlan 19`
   ถูก Trigger ถูกทาง แต่ล้มเหลวเพราะปลายทางไม่มีจริง — **นี่คือ Two-Fault Scenario**: (1) ISE ล่มจริง
   (คาดการณ์ไว้แล้วในการออกแบบ Step 436) + (2) Config ของ Fallback ผิดเลข (ไม่คาดคิด)
4. **แยก Misconfig จาก Attack** — ไม่มีสัญญาณโจมตีใดๆ เป็นเพียง Human Error ตอน Copy-Paste Config
   ข้ามอุปกรณ์

### 604.5 Root Cause และ Fix

**Root Cause**: พิมพ์ `authorize vlan 19` ผิดจาก `authorize vlan 91` (สลับเลข) ตอน Deploy Config บน
ACCESS-SW2 — VLAN 19 ไม่มีอยู่ในระบบ ทำให้ Authorization Action ล้มเหลวซ้อนกับ ISE ที่ล่มไปแล้ว

```
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1-3
ACCESS-SW2(config-if-range)# no authentication event server dead action authorize vlan 19
ACCESS-SW2(config-if-range)# authentication event server dead action authorize vlan 91
ACCESS-SW2(config-if-range)# exit
```

**Verify:**

```
ACCESS-SW2# show authentication sessions interface GigabitEthernet1/0/1 details | include Status|Vlan
                   Status:  Authorized
              Vlan Policy:  91
```

```
! เมื่อ ISE-1 กลับมา Online จริง (หลัง Rollback สำเร็จ) — Reinitialize อัตโนมัติทันที
*Sep 28 11:15:44.101: %AUTHMGR-5-START: Starting 'dot1x' for client (0050.5601.bbbb) on
Interface Gi1/0/1 AuditSessionID 0A0A63020000004512345679
```

> **บทเรียนสำคัญ**: การ Copy-Paste Config ระหว่างอุปกรณ์ที่ "ควรจะเหมือนกัน" (ACCESS-SW1/2 ที่ทั้งคู่
> Uplink ไป DIST-SW1) เป็นจุดเสี่ยง Human Error ที่พบบ่อยที่สุด — ควร Verify ด้วย `show vlan brief`
> บนอุปกรณ์ปลายทางก่อนเสมอว่า VLAN ที่จะอ้างถึงมีอยู่จริง และควร Config ผ่าน Automation/Template
> ([Part 23](part-023-automation-basics.md)) แทนการพิมพ์มือซ้ำในสถานการณ์ที่มีผลกระทบต่อ Fallback
> Path ของ Security Feature โดยตรง

---

## Step 605 — Scenario 4: TrustSec SGACL บล็อก Traffic ที่ควรผ่าน เพราะ SXP Propagation ล้มเหลว

### 605.1 สถานการณ์

ทีม Security หมุน (Rotate) Shared Password ของ SXP ทุกคู่ Speaker/Listener เป็นประจำทุกไตรมาสตาม
Policy — รอบนี้เปลี่ยน Password บน **DIST-SW1** (Listener) สำเร็จ แต่ลืมอัปเดต **ACCESS-SW1**
(Speaker) ให้ตรงกัน — เช้าวันถัดมา **PC1** (`10.10.10.11`, SALES SGT=100 ตาม
[Part 44 Step 437](part-044-dot1x-trustsec.md)) รายงานว่าเปิดเว็บแอปพลิเคชันที่อยู่บน SERVERS
(`10.10.30.x`, SGT=120) ผ่าน HTTPS ไม่ได้ ทั้งที่ SGACL Matrix อนุญาต SALES→SERVERS Port 443 ไว้แล้ว
([Part 44 Step 438](part-044-dot1x-trustsec.md))

### 605.2 Config ที่ไม่ตรงกัน (Password Mismatch)

```
! ===== DIST-SW1 — หมุน Password ใหม่แล้ว (ทำถูกต้อง) =====
DIST-SW1(config)# cts sxp default password TrustSec-SXP-Key2026-Q4!
```

```
! ===== ACCESS-SW1 — ยังใช้ Password เก่า (ลืมอัปเดต) =====
ACCESS-SW1# show running-config | include sxp default password
cts sxp default password TrustSec-SXP-Key2026!            <-- ไม่ตรงกับ DIST-SW1 อีกแล้ว
```

### 605.3 Symptom — Show Output

```
DIST-SW1# show cts sxp connections
SXP                 : Enabled
Highest Version Supported: 4
Default Password    : Set
Default Source IP   : 10.10.99.2
Connection retry open period: 120 secs
Reconcile period: 120 secs
Retry open timer is not running

Peer IP           : 10.10.99.11
Source IP         : 10.10.99.2
Conn status       : Off                                    <-- ควรเป็น "On" ถ้าปกติ
Conn version      : -
Conn capability    : -
Conn hold time     : -
Local mode         : Listener
Connection inst#   : 3
```

```
! Log ยืนยันสาเหตุชัดเจน
DIST-SW1# show logging | include SXP
*Sep 26 22:04:12.220: %CTS_SXP-3-CONN_FAILED: SXP Connection to peer 10.10.99.11 failed;
reason: password mismatch
```

```
! ผลกระทบที่ปลาย Enforcement Point (DIST-SW3) — ไม่มี Entry ของ PC1 อีกต่อไป
DIST-SW3# show cts role-based sgt-map all | include 10.10.10.11
! ไม่มี Entry — SGT Binding ของ PC1 หายไปตั้งแต่ SXP Chain ขาดตอนที่ DIST-SW1

DIST-SW3# show cts role-based counters | include Unknown
Unknown        Unknown              120         RBACL      Deny-All          4102
```

### 605.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — PC1 คุยกับ SERVERS ไม่ได้ แม้ SGACL Matrix ระบุ Permit ชัดเจน — อาการแบบนี้ (ACL
   ที่ควร Permit แต่ Block จริง) ในบริบท TrustSec ต้องสงสัยว่า **SGT ที่มาถึง Enforcement Point ไม่ใช่
   SGT ที่คาดไว้**
2. **ระบุ Control** — ตามตาราง 601.4 → `show cts role-based sgt-map` ที่ **DIST-SW3** (Enforcement
   Point ตาม [Part 44 Step 440](part-044-dot1x-trustsec.md)) ไม่พบ Entry ของ `10.10.10.11` เลย —
   แปลว่า Traffic จาก PC1 มาถึง DIST-SW3 โดยไม่มี SGT ติดมา (ตกไปเป็น `Unknown`)
3. **Trace Dependency** — ต้นเหตุของ SGT ที่มาจาก ACCESS-SW1 (Speaker) คือ SXP — รัน
   `show cts sxp connections` ที่ DIST-SW1 (Listener ที่รับ SGT จาก ACCESS-SW1 ก่อนส่งต่อผ่าน Inline
   Tagging ไปยัง DIST-SW3) พบ `Conn status: Off` ทันที และ Log ยืนยัน `password mismatch` ตรงกับช่วง
   เวลาที่ทีม Security หมุน Password
4. **แยก Misconfig จาก Attack** — Password Mismatch จาก Change Management ที่ทำไม่ครบ ไม่ใช่การ
   โจมตี — และ `show cts role-based counters` ยืนยันว่า SGACL "ทำงานถูกต้องตามที่ควร" คือ Deny
   Traffic ที่ SGT เป็น `Unknown` (ตาม Default Deny Policy ของ Lab นี้ — สอดคล้องหลัก Zero-Trust ที่
   Traffic ที่ไม่รู้จัก SGT ต้องถูกปฏิเสธไว้ก่อนเสมอ) — **นี่คือหัวใจของ Scenario นี้**: SGACL ไม่ได้
   "พัง" มันทำตามสิ่งที่มันเห็นจริง (SGT ขาดหาย) อย่างถูกต้อง

### 605.5 Root Cause และ Fix

**Root Cause**: `cts sxp default password` ของ ACCESS-SW1 กับ DIST-SW1 ไม่ตรงกันหลังการหมุน Password
ทำให้ SXP Connection ล้มเหลว → SGT Binding ของ Endpoint ทุกตัวใต้ ACCESS-SW1 ไม่ถูกส่งไปถึง DIST-SW3
→ SGACL เห็นเป็น `Unknown` แล้ว Deny ตาม Default Policy

```
ACCESS-SW1(config)# cts sxp default password TrustSec-SXP-Key2026-Q4!
```

> **ไม่ต้อง** `shutdown`/`no shutdown` หรือ Restart SXP Process ใดๆ — SXP จะ Retry การเชื่อมต่อ
> อัตโนมัติตาม `Connection retry open period` (120 วินาทีตาม Default) ทันทีที่ Password ตรงกัน

**Verify:**

```
DIST-SW1# show cts sxp connections | include Peer|Conn status
Peer IP           : 10.10.99.11
Conn status       : On                                     <-- กลับมา On แล้ว

DIST-SW3# show cts role-based sgt-map all | include 10.10.10.11
10.10.10.11              100     SXP

DIST-SW3# show cts role-based counters | include 100.*120
100            120                  120         RBACL      SALES-TO-SERVERS   8821
```

> **ข้อเสนอเชิง Process**: Change Management ที่แก้ Shared Secret ของ SXP/TACACS+/RADIUS ควรมี
> Checklist ยืนยัน **ทั้งสองปลาย** เสมอ (คล้ายกฎ "ห้ามปิด Session เดิมก่อนยืนยัน Session ใหม่" จาก
> [Part 22 Step 213](part-022-aaa-device-hardening.md)) — และควรตั้ง Monitoring/Alert บน
> `show cts sxp connections` State เปลี่ยนจาก `On` เป็น `Off` เพื่อจับ Failure แบบนี้ได้ก่อนที่ User
> จะโทรมาแจ้งเอง

---

## Step 606 — Scenario 5: TACACS+ AAA Lockout เพราะ Server ล่ม + Local Fallback ผิด

### 606.1 สถานการณ์

ทีม Network เพิ่ม **WAN-EDGE-2** เข้าสู่ AAA Baseline โดย Copy Config Template จาก
[Part 22 Step 220](part-022-aaa-device-hardening.md) — แต่วิศวกรที่พิมพ์ Config รีบร้อนและพิมพ์
Method List ผิดโดยไม่ใส่ `local` ต่อท้าย (คิดว่า "TACACS+ ต้อง Reliable อยู่แล้ว ไม่ต้องมี Fallback ก็
ได้") — สองสัปดาห์ต่อมา **AAA-SRV** (`10.10.99.60`) หยุดตอบสนองกะทันหันเพราะ Disk เต็ม — Admin ทุกคน
**Login เข้า WAN-EDGE-2 ไม่ได้เลย** ทั้ง SSH และ Console

### 606.2 Config ที่ผิด (ไม่มี Local Fallback)

```
! ===== WAN-EDGE-2 — Method List ผิด (ไม่มี "local" ต่อท้าย) =====
WAN-EDGE-2(config)# aaa new-model
WAN-EDGE-2(config)# aaa authentication login default group AAA-TACACS
!                                                            ^^^^^^^^^^ ไม่มี "local" ต่อท้าย!
WAN-EDGE-2(config)# aaa authentication enable default group AAA-TACACS
!                                                            ^^^^^^^^^^ ไม่มี "enable" ต่อท้าย!

! Console Line ก็ไม่ได้แยก Method List ที่ไม่พึ่ง TACACS+ ตามกฎเหล็ก Step 213
WAN-EDGE-2(config)# line console 0
WAN-EDGE-2(config-line)# login authentication default        <-- ใช้ default เดียวกันหมด!
WAN-EDGE-2(config-line)# exit
```

### 606.3 Symptom

```
$ ssh admin1@10.10.254.6
Password: ****
% Authentication failed
Password: ****
% Authentication failed

! ลองที่ Console ก็ล้มเหลวเช่นกัน (ไม่มี Method List แยกที่ไม่พึ่ง TACACS+)
WAN-EDGE-2 con0 is now available

Press RETURN to get started.
Username: admin1
Password: ****
% Authentication failed
```

```
! ตรวจสอบสถานะ Server จากอุปกรณ์ข้างเคียงที่ยัง Login ได้ (CORE-SW1 ที่มี Fallback ถูกต้อง)
CORE-SW1# show tacacs
Server: AAA-SRV1/10.10.99.60/49
        Socket opens:               3
        Socket closes:              0
        Socket aborts:               3          <-- Socket เปิดไม่สำเร็จต่อเนื่อง
        Socket errors:               3
        Total packets sent:         3
        Total packets recv:         0            <-- ไม่ได้รับ Response เลย
```

```
CORE-SW1# ping 10.10.99.60
Success rate is 0 percent (0/5)     <-- AAA-SRV ไม่ตอบสนองจริง ยืนยัน Server ล่ม ไม่ใช่ Network ปัญหา
```

### 606.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — Login ล้มเหลวทั้ง SSH และ Console บน WAN-EDGE-2 เท่านั้น (อุปกรณ์อื่นที่ Config
   ถูกต้องยัง Login ได้ผ่าน Local Fallback) — ชี้ว่าปัญหาเฉพาะเจาะจงที่ Method List ของ WAN-EDGE-2
   ไม่ใช่ AAA-SRV ทั้งระบบ (แม้ AAA-SRV จะล่มจริงก็ตาม)
2. **ระบุ Control** — ตามตาราง 601.4 "Admin Login เข้าอุปกรณ์ไม่ได้เลย" → TACACS+ AAA — จาก
   `show tacacs`/`ping` บนอุปกรณ์ข้างเคียงยืนยันว่า AAA-SRV **ไม่ตอบสนองจริง** (Root Trigger)
3. **Trace Dependency** — จุดที่ทำให้ Lockout **สมบูรณ์** (ไม่ใช่แค่ Login ช้า) คือ Method List ของ
   WAN-EDGE-2 ที่ไม่มี `local` — ตามหลักการ Step 213-214: "Fallback ทำงานเฉพาะกรณี Server ไม่ตอบ" —
   ถ้าไม่มี `local` ต่อท้ายเลย เมื่อ Server ไม่ตอบ IOS จะไม่มี Method ใดให้ลองอีกต่อไป → Reject ทุก
   ครั้งไม่มีข้อยกเว้น รวมถึง Console ที่ควรเป็น "ทางหนีสุดท้าย" ตามกฎเหล็ก แต่ถูก Config ให้ใช้
   `default` Method List เดียวกันจนไม่มีทางหนีเหลือเลย
4. **แยก Misconfig จาก Real Outage** — เป็น **Two-Fault Scenario** เหมือน Step 604: (1) AAA-SRV ล่ม
   จริง (Infrastructure Failure ที่คาดเดาไม่ได้ล่วงหน้าเสมอ) + (2) Method List ของ WAN-EDGE-2 ไม่มี
   Fallback ตามกฎเหล็ก (Human Error ที่ป้องกันได้)

### 606.5 Root Cause และ Fix

**Root Cause**: `aaa authentication login default group AAA-TACACS` ไม่มี `local` ต่อท้าย รวมกับ
`line console 0` ที่ไม่มี Method List แยกที่ไม่พึ่ง TACACS+ — เมื่อ AAA-SRV ล่ม ไม่มีทางใด Login เข้า
อุปกรณ์ได้เลยแม้แต่ทาง Console

**ขั้นตอนกู้คืน (ต้องทำผ่าน Physical Console + Password Recovery เพราะไม่มี Fallback เหลือ)**:

```
! ต้อง Interrupt Boot Sequence เข้า ROMMON แล้วข้าม Startup-Config ชั่วคราว (Password Recovery
! Procedure มาตรฐานของ Cisco IOS — Config Register 0x2142) จากนั้นแก้ไข Method List ก่อน Save
Router(config-register)# 0x2142
Router# reload
! ... boot โดยไม่โหลด startup-config ...
Router(config)# aaa authentication login default group AAA-TACACS local
Router(config)# aaa authentication enable default group AAA-TACACS enable
Router(config)# aaa authentication login CONSOLE-IN local
Router(config)# line console 0
Router(config-line)# login authentication CONSOLE-IN
Router(config-line)# exit
Router(config)# config-register 0x2102
Router(config)# end
Router# copy running-config startup-config
```

**Fix ที่ถูกต้อง (สิ่งที่ควรเป็นตั้งแต่แรก ตาม Part 22 Step 213-214):**

```
WAN-EDGE-2(config)# aaa authentication login default group AAA-TACACS local
WAN-EDGE-2(config)# aaa authentication enable default group AAA-TACACS enable
WAN-EDGE-2(config)# aaa authentication login CONSOLE-IN local
WAN-EDGE-2(config)# line console 0
WAN-EDGE-2(config-line)# login authentication CONSOLE-IN
WAN-EDGE-2(config-line)# exit
```

**Verify (จำลองสถานการณ์ AAA-SRV ล่มอีกครั้งแบบควบคุมได้ ก่อนปิดงาน):**

```
! ปิด Route ไป AAA-SRV ชั่วคราวเพื่อทดสอบ Fallback (ทำตามกฎเหล็ก Step 213: ตั้ง reload in ก่อนเสมอ)
WAN-EDGE-2# reload in 10
WAN-EDGE-2(config)# access-list 199 deny tcp any host 10.10.99.60 eq 49
WAN-EDGE-2(config)# access-list 199 permit ip any any
WAN-EDGE-2(config)# interface Loopback0
WAN-EDGE-2(config-if)# ip access-group 199 out
WAN-EDGE-2(config-if)# exit

! ทดสอบ SSH จาก Session ใหม่ (ไม่ปิด Session เดิม)
$ ssh admin-local@10.10.254.6
Password: ****                     <-- ใช้ local fallback สำเร็จ แม้ AAA-SRV ไปไม่ถึง
WAN-EDGE-2#

WAN-EDGE-2(config-if)# no ip access-group 199 out
WAN-EDGE-2# reload cancel
```

> **บทเรียนสำคัญที่สุดของ Scenario นี้**: กฎเหล็ก 3 ข้อจาก [Part 22 Step 213](part-022-aaa-device-hardening.md)
> (Local Fallback เสมอ / Console แยก Method List / `reload in` Safety Net) มีไว้เพื่อป้องกันสถานการณ์
> นี้โดยเฉพาะ — การข้ามขั้นตอนใดขั้นตอนหนึ่งดูเหมือนไม่มีผลกระทบทันที (เพราะ Server ปกติจะ Reachable
> อยู่แล้ว) แต่จะกลายเป็นหายนะทันทีที่ Server ล่มจริงเพียงครั้งเดียว — ต้อง Config ตามกฎเหล็กทุกอุปกรณ์
> โดยไม่มีข้อยกเว้น ไม่ว่าจะ "มั่นใจ" แค่ไหนว่า Server จะไม่ล่ม

---

## Step 607 — Scenario 6: ZBFW บล็อก Inter-Zone Traffic ของ Server ใหม่ เพราะ Subnet ไม่อยู่ใน Zone

### 607.1 สถานการณ์

บริษัทเปิด Server Farm ใหม่สำหรับระบบ ERP โดยเพิ่ม Subnet ใหม่ `10.10.130.0/24` (VLAN 130) เชื่อมต่อ
เข้า **WAN-EDGE-1** ผ่าน Sub-interface ใหม่ `GigabitEthernet0/0/1.130` (Trunk เดิมจาก CORE-SW1 ที่มี
อยู่แล้ว เพิ่มแค่ Sub-interface ใหม่ทาง Logical) — ทีม Server Deploy ERP-SRV (`10.10.130.10`) เสร็จ
แต่ **ออก Internet ไม่ได้เลย** และ **Host ฝั่ง Campus (VLAN 10/20/30) เข้าถึง ERP-SRV ไม่ได้เลย** ทั้งที่
Routing ระหว่าง VLAN ทำงานถูกต้อง (`show ip route` เห็น Subnet ปกติ, `ping` จาก WAN-EDGE-1 เองไปยัง
ERP-SRV สำเร็จ)

### 607.2 Config ที่ขาด (Sub-interface ใหม่ไม่ได้เข้า Zone)

```
! ===== WAN-EDGE-1 — เพิ่ม Sub-interface ใหม่ (แต่ลืม zone-member) =====
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1.130
WAN-EDGE-1(config-subif)# encapsulation dot1Q 130
WAN-EDGE-1(config-subif)# ip address 10.10.130.1 255.255.255.0
WAN-EDGE-1(config-subif)# no shutdown
WAN-EDGE-1(config-subif)# exit
! *** ไม่มีบรรทัด "zone-member security INSIDE" เลย! ***
```

### 607.3 Symptom — Show Output

```
ERP-SRV# ping 8.8.8.8
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)

! แต่ WAN-EDGE-1 เอง Ping ผ่านได้ (เพราง Ping จาก Router เองไม่ผ่าน Zone-Pair เพราะไม่ใช่ Transit
! Traffic ระหว่าง Zone — เป็นกับดักที่ทำให้เข้าใจผิดว่า "Routing ปกติ ต้องไม่ใช่ปัญหา ZBFW")
WAN-EDGE-1# ping 10.10.130.10
Success rate is 100 percent (5/5)
```

```
WAN-EDGE-1# show zone-pair security
Zone-pair name INSIDE-TO-OUTSIDE
    Source-Zone INSIDE  Destination-Zone OUTSIDE
  service-policy INSIDE-TO-OUTSIDE-POLICY

WAN-EDGE-1# show running-config interface GigabitEthernet0/0/1.130
interface GigabitEthernet0/0/1.130
 encapsulation dot1Q 130
 ip address 10.10.130.1 255.255.255.0
!                                          <-- ไม่มี "zone-member security" เลย!

WAN-EDGE-1# show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions
! (ไม่มี Session จาก 10.10.130.x เลยแม้แต่รายการเดียว)
```

### 607.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — ERP-SRV คุยข้าม Zone ไม่ได้เลยทั้งสองทิศทาง (ออก Internet ไม่ได้ + Campus เข้าไม่
   ถึง) แต่ WAN-EDGE-1 เอง Ping ถึง ERP-SRV ได้ปกติ (Routing/L2 ถูกต้อง 100%) — Pattern นี้ (Traffic
   ผ่าน Router เองได้ แต่ Traffic ผ่าน "ทะลุ" Router ไม่ได้) ชี้ไปที่ Policy ระดับ Forwarding Plane
   ไม่ใช่ Routing
2. **ระบุ Control** — ตามตาราง 601.4 "Host คุยข้าม Zone ไม่ได้ทั้งที่ Policy อนุญาต" → ZBFW —
   `show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions` ไม่มี Session จาก
   `10.10.130.x` เลยแม้แต่รายการเดียว (ควรเห็น Session ถ้า Traffic เข้ามาถึงระดับ Inspect จริง)
3. **Trace Dependency** — `show running-config interface GigabitEthernet0/0/1.130` ไม่มีบรรทัด
   `zone-member security` เลย — ตามหลักการ [Part 45 Step 445](part-045-advanced-acl-zbfw.md):
   Interface ที่ไม่ได้เป็นสมาชิกของ Zone ใดเลย (**ไม่ใช่ INSIDE และไม่ใช่ OUTSIDE**) ถือเป็น "Unzoned
   Interface" — Traffic ระหว่าง Unzoned Interface กับ Interface ที่อยู่ใน Zone ใดๆ **ถูก Deny โดย
   Default เสมอ** (คนละกรณีกับ Traffic ระหว่าง Unzoned Interface สองเส้นที่ยังคง Route ได้ปกติแบบไม่
   ผ่าน Inspection เลย)
4. **แยก Misconfig จาก Attack** — ไม่มีสัญญาณ Attack — เป็น Config Gap ล้วนๆ จากการเพิ่ม
   Sub-interface ใหม่โดยไม่ได้ตาม Checklist การเพิ่ม Segment ใหม่เข้า ZBFW ให้ครบ (Zone Membership +
   Class-map/ACL ถ้ามี Exception เฉพาะ)

### 607.5 Root Cause และ Fix

**Root Cause**: Sub-interface ใหม่ `Gi0/0/1.130` ไม่ได้ถูกใส่เข้า Zone `INSIDE` — ตามกฎของ ZBFW
Sub-interface ต้อง Config `zone-member security` **แยกจาก Physical Interface หลักเสมอ** แม้
Physical Interface หลัก (`Gi0/0/1`) จะอยู่ใน Zone `INSIDE` ไปแล้วก็ตาม

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1.130
WAN-EDGE-1(config-subif)# zone-member security INSIDE
WAN-EDGE-1(config-subif)# exit
```

**Verify:**

```
WAN-EDGE-1# show zone-pair security | include INSIDE
Zone-pair name INSIDE-TO-OUTSIDE
    Source-Zone INSIDE  Destination-Zone OUTSIDE

WAN-EDGE-1# show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions
 Zone-pair: INSIDE-TO-OUTSIDE
  Service-policy inspect : INSIDE-TO-OUTSIDE-POLICY
    Class-map: INSIDE-TO-OUTSIDE-CLASS (match-any)
      Match: protocol tcp
        Session 5678900 (10.10.130.10:52344) => (203.0.113.55:443) tcp SIS_OPEN

ERP-SRV# ping 8.8.8.8
Success rate is 100 percent (5/5)
```

> **Checklist สำหรับ "เพิ่ม Subnet/Server Segment ใหม่" ทุกครั้งที่มี ZBFW**: (1) `zone-member
> security` บน Interface/Sub-interface ใหม่เสมอ — ไม่ต่อจาก Physical Interface หลักอัตโนมัติ, (2)
> ตรวจว่า Class-map ที่ใช้ครอบคลุม Protocol ที่ Segment ใหม่ต้องใช้จริง (ถ้าเป็น Protocol แปลกที่ไม่
> อยู่ใน `match protocol tcp/udp/icmp` ต้องเพิ่ม Class-map แยก), (3) ถ้ามี Exception เฉพาะ (เช่น
> NAT/Public IP แบบ [Part 45 Step 445](part-045-advanced-acl-zbfw.md)) ต้องเพิ่ม ACL/Class-map
> Exception คู่กันด้วย — ทำตาม Checklist นี้ **ก่อน** ประกาศ Go-Live ของ Server Segment ใหม่เสมอ

---

## Step 608 — Scenario 7: CoPP บล็อก OSPF/BGP Control Traffic จริงระหว่าง Network Event

### 608.1 สถานการณ์

คืนวันพายุฝนตกหนัก สาย Fiber ระหว่าง **CORE-SW1 ↔ DIST-SW1** และ **CORE-SW1 ↔ DIST-SW2** เกิด
**Flap ซ้ำๆ** (Up/Down สลับกันทุก 10-30 วินาที) นานกว่า 20 นาที เพราะ Connector เสื่อมจากความชื้น — ทีม
NOC สังเกตว่า **OSPF Neighbor ระหว่าง CORE-SW1 กับ DIST-SW1/DIST-SW2 ค้างที่ `INIT` ไม่ยอมขึ้น `FULL`
เลย แม้ Physical Link จะกลับมา Up สำเร็จแล้วก็ตาม** — Symptom เดียวกันกับที่อธิบายไว้ใน
[Part 46 Step 459](part-046-control-plane-policing.md)

### 608.2 สถานการณ์ที่ CoPP ทำงานถูกต้องแต่ CIR ต่ำเกินไปสำหรับ Event จริง

```
! ===== CORE-SW1 — CoPP Policy เดิมจาก Part 46 Step 454-460 (ไม่ได้แก้ไข) =====
CORE-SW1# show running-config | section policy-map COPP-POLICY
policy-map COPP-POLICY
 class COPP-CRITICAL
  police cir 10000000 bc 512000
   conform-action transmit
   exceed-action drop
```

`cir 10000000` (10 Mbps) ถูกออกแบบไว้สำหรับ OSPF Hello/LSA ปริมาณปกติ — แต่ตอน Link Flap 2 เส้นพร้อม
กัน SPF Recalculation + LSA Flooding ของทั้ง Area พุ่งสูงกว่าปริมาณ Baseline หลายเท่าในช่วงเวลาสั้นๆ

### 608.3 Symptom — Show Output

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11          1   INIT/  -        00:00:31    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          1   INIT/  -        00:00:33    10.255.20.2     TenGigabitEthernet1/0/2
```

```
CORE-SW1# show policy-map control-plane class COPP-CRITICAL
    Class-map: COPP-CRITICAL (match-any)
      612400 packets, 78387200 bytes
      5 minute offered rate 14200000 bps, drop rate 4180000 bps
      police:
          cir 10000000 bps, bc 512000 bytes
        conformed 428210 packets, 54810880 bytes; actions:
          transmit
        exceeded 184190 packets, 23576320 bytes; actions:
          drop                                             <-- Drop จริง เพิ่มขึ้นต่อเนื่อง!
```

```
CORE-SW1# show access-list COPP-ACL-ROUTING
Extended IP access list COPP-ACL-ROUTING
    10 permit ospf any host 224.0.0.5 (612400 matches)     <-- ACL Match ถูกต้อง ไม่ใช่ปัญหา Class-map
    20 permit ospf any host 224.0.0.6 (0 matches)
```

### 608.4 ขั้นตอน Diagnose (อ้างอิงกระบวนการเดียวกับ Part 46 Step 459 โดยตรง)

1. **ยืนยันอาการ** — OSPF ค้างที่ `INIT` แม้ Physical Link Up แล้ว — `INIT` แปลว่า CORE-SW1 เห็น Hello
   จาก DIST-SW1/2 (มัน Log ID ของ Neighbor ได้) แต่ Neighbor ไม่เห็น Hello ตอบกลับจาก CORE-SW1 (หรือ
   Two-Way Communication ไม่สำเร็จ) — Pattern นี้ตรงกับ Hello Packet ถูก Drop เป็นบางส่วน ไม่ใช่ Link
   Down จริง (ถ้า Link Down จริง Neighbor จะหายไปจาก Table เลย ไม่ค้างที่ `INIT`)
2. **ระบุ Control** — ตามตาราง 601.4 "OSPF/BGP Flap โดยไม่มีปัญหา Physical" → CoPP —
   `show policy-map control-plane class COPP-CRITICAL` พบ `exceeded 184190 packets` เพิ่มขึ้นต่อเนื่อง
   — Class ที่ควรรับประกัน Routing Protocol กลับมี Drop จริง
3. **Trace Dependency** — `show access-list COPP-ACL-ROUTING` ยืนยันว่า ACL Match ถูกต้อง 100%
   (Traffic OSPF ถูกจัดเข้า `COPP-CRITICAL` แล้วจริง ไม่ได้หลุดไป `class-default`) — ปัญหาจึงไม่ใช่
   Class-map ผิด แต่เป็น **`cir`/`bc` ที่ตั้งไว้ต่ำเกินไปสำหรับ Volume ของ LSA Flooding ตอน Multi-Link
   Flap พร้อมกัน**
4. **แยก Misconfig จาก Attack** — ตรวจสอบเวลาที่ Drop เพิ่มขึ้นตรงกับ Log Link Flap ของ Fiber จริง
   (`show logging | include LINEPROTO`) และไม่มี Source IP ผิดปกติปนเข้ามาใน Traffic — เป็น **Real
   Network Event ที่ Baseline CIR ไม่ได้ออกแบบไว้ให้รองรับ** ไม่ใช่การโจมตี — CoPP กำลังทำงานตามที่
   Config ไว้อย่างถูกต้อง เพียงแต่ Threshold ไม่เพียงพอสำหรับ Worst-Case ของ Topology จริง

### 608.5 Root Cause และ Fix

**Root Cause**: `police cir 10000000` ของ Class `COPP-CRITICAL` ถูกออกแบบตาม Baseline OSPF Hello/LSA
ปกติ แต่ไม่ได้คำนวณ Margin สำหรับ Worst-Case ที่ Link หลายเส้น Flap พร้อมกันจนเกิด LSA Flooding
ปริมาณสูงชั่วขณะ (Transient Burst)

```
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class COPP-CRITICAL
CORE-SW1(config-pmap-c)# police cir 20000000 bc 1024000
CORE-SW1(config-pmap-c-police)#  conform-action transmit
CORE-SW1(config-pmap-c-police)#  exceed-action transmit
!  ^^^^ เปลี่ยนจาก "drop" เป็น "transmit" ชั่วคราวเฉพาะ Class-map นี้เท่านั้น เพื่อยืนยัน Hypothesis
!       ก่อนตัดสินใจเพิ่ม CIR ถาวร (Diagnostic Technique — ไม่ควรเปิดค้างไว้ใน Production เกิน
!       ช่วง Maintenance Window ที่ควบคุมได้)
```

**Verify แบบ 2 ขั้น (ทดสอบก่อนใช้ถาวร แล้วค่อยกลับไปใช้ `drop` พร้อม CIR ใหม่):**

```
! ขั้น 1: ยืนยันว่า OSPF ขึ้น FULL ได้จริงถ้าไม่ถูก Drop เลย (พิสูจน์ Hypothesis)
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11          1   FULL/  -        00:00:38    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          1   FULL/  -        00:00:35    10.255.20.2     TenGigabitEthernet1/0/2

! ขั้น 2: กลับไปใช้ exceed-action drop พร้อม CIR ใหม่ที่สูงพอ (ค่าถาวรที่ผ่านการคำนวณ Margin แล้ว)
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class COPP-CRITICAL
CORE-SW1(config-pmap-c)# police cir 20000000 bc 1024000
CORE-SW1(config-pmap-c-police)#  conform-action transmit
CORE-SW1(config-pmap-c-police)#  exceed-action drop

CORE-SW1# show policy-map control-plane class COPP-CRITICAL
        exceeded 0 packets, 0 bytes; actions:
          drop
```

> **ข้อควรระวังสำคัญ**: การเปลี่ยน `exceed-action` เป็น `transmit` คือเทคนิค **Diagnostic ชั่วคราว
> เท่านั้น** เพื่อพิสูจน์ว่า CoPP คือสาเหตุจริง — ต้องกลับไปเป็น `drop` พร้อม `cir` ใหม่ที่สูงพอเสมอ
> ก่อนปิดงาน มิฉะนั้นเท่ากับปิด CoPP ทิ้งสำหรับ Class ที่สำคัญที่สุด ผิดหลักการ Step 601.2 ทันที — และ
> ควรทดสอบ CIR ใหม่ด้วยการจำลอง Link Flap ซ้ำในช่วง Maintenance Window ที่ควบคุมได้ ก่อนเชื่อว่า
> Margin เพียงพอสำหรับ Worst-Case ในอนาคตจริง

---

## Step 609 — Scenario 8: Port Security Violation Shutdown เพราะมี Hub ต่อพ่วงเพิ่มอุปกรณ์เกิน Limit

### 609.1 สถานการณ์

ห้องประชุมใหญ่ของบริษัทมีเพียง 1 Data Port (`ACCESS-SW1 Gi1/0/18`, ตั้ง Maximum 2 ตาม Baseline ปกติ
สำหรับ PC + Phone) — ทีมจัดประชุมเสียบ **Unmanaged Hub 8 พอร์ต** เข้ากับ Data Port นั้นเพื่อให้ผู้เข้า
ประชุม 6 คน Plug Notebook เข้า LAN ได้พร้อมกัน (ไม่รู้ว่าเป็นการฝ่าฝืน Policy) — พอร์ตเข้าสู่
**err-disabled ซ้ำแล้วซ้ำเล่า** ทุกครั้งที่ Helpdesk `shutdown`/`no shutdown` ให้

### 609.2 Config เดิม — Violation Mode `shutdown` (ไม่ใช่ Baseline ปกติของ Part 21 ที่ใช้ `restrict`)

```
! ===== ACCESS-SW1 — พอร์ตห้องประชุมนี้ตั้งไว้เข้มงวดกว่า Baseline ปกติโดยตั้งใจ =====
ACCESS-SW1(config)# interface GigabitEthernet1/0/18
ACCESS-SW1(config-if)# description ** Meeting Room B - Data Port **
ACCESS-SW1(config-if)# switchport mode access
ACCESS-SW1(config-if)# switchport access vlan 10
ACCESS-SW1(config-if)# switchport port-security
ACCESS-SW1(config-if)# switchport port-security maximum 2
ACCESS-SW1(config-if)# switchport port-security violation shutdown
ACCESS-SW1(config-if)# switchport port-security mac-address sticky
ACCESS-SW1(config-if)# exit
```

### 609.3 Symptom — Show Output

```
%PM-4-ERR_DISABLE: psecure-violation error detected on Gi1/0/18, putting Gi1/0/18 in err-disable state
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address
0024.d7aa.1103 on port GigabitEthernet1/0/18.
```

```
ACCESS-SW1# show port-security interface GigabitEthernet1/0/18
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode              : Shutdown
Maximum MAC Addresses        : 2
Total MAC Addresses           : 2
Security Violation Count        : 14         <-- เพิ่มขึ้นทุกครั้งที่ Helpdesk Recover พอร์ต
```

```
! ตรวจ Log ย้อนหลังพบ MAC หลายตัวต่างกันโดยสิ้นเชิงพยายามเข้าพอร์ตเดียวกันในช่วงเวลาสั้นๆ
ACCESS-SW1# show logging | include PSECURE_VIOLATION
*Sep 26 14:02:11: caused by MAC address 0024.d7aa.1101 on port GigabitEthernet1/0/18.
*Sep 26 14:03:45: caused by MAC address 0024.d7aa.1102 on port GigabitEthernet1/0/18.
*Sep 26 14:05:02: caused by MAC address 0024.d7aa.1103 on port GigabitEthernet1/0/18.
*Sep 26 14:07:18: caused by MAC address 445566.aabbcc on port GigabitEthernet1/0/18.
```

### 609.4 ขั้นตอน Diagnose

1. **ยืนยันอาการ** — พอร์ต err-disabled ซ้ำ**หลายครั้งต่อวัน** ไม่ใช่ครั้งเดียวแล้วหาย — ต่างจาก
   Violation ปกติที่เกิดครั้งเดียวจาก NIC เปลี่ยน (ซึ่งจะเกิด Violation แค่ 1 ครั้งแล้วเงียบ)
2. **ระบุ Control** — ตามตาราง 601.4 "พอร์ตเข้า err-disabled ซ้ำๆ" → Port Security —
   `show port-security interface` ยืนยัน `Security Violation Count: 14` และเพิ่มขึ้นต่อเนื่อง
3. **Trace Dependency** — `show logging | include PSECURE_VIOLATION` เผยให้เห็นสิ่งสำคัญ: **MAC
   Address ที่ทำให้เกิด Violation เปลี่ยนไปเรื่อยๆ** (`...1101`, `...1102`, `...1103`,
   `445566.aabbcc`) — ไม่ใช่ MAC เดิมซ้ำ (ซึ่งจะบ่งบอกว่าเป็นแค่ NIC เดียวที่ Flap) — Pattern นี้ (MAC
   หลายตัวต่างผู้ผลิต OUI กันโดยสิ้นเชิงพยายามเข้าพอร์ตเดียวในเวลาไล่เลี่ยกัน) คือลายเซ็นคลาสสิกของ
   **อุปกรณ์ Layer 1/2 หลายตัวต่อพ่วงผ่าน Hub/Switch เดียวหลังพอร์ตนั้น**
4. **แยก Misconfig จาก Attack** — ไม่มีสัญญาณ MAC Flooding Attack (Rate การเปลี่ยน MAC ไม่เร็วผิด
   ปกติ, MAC ไม่ใช่ Random/Spoofed Pattern) — สอบถามทีม Facility ยืนยันว่ามีการเสียบ Hub จริงเพื่อการ
   ประชุม เป็น **Legitimate Business Need ที่ขัดกับ Policy เดิม** ไม่ใช่การโจมตี

### 609.5 Root Cause และการตัดสินใจที่ถูกต้อง (Security vs Usability Tradeoff)

**Root Cause**: มีการเสียบ Unmanaged Hub เพิ่มจำนวนอุปกรณ์หลัง Data Port เกิน Maximum ที่ Policy ตั้ง
ไว้ (2 — สำหรับ PC+Phone เท่านั้น) จริง — Port Security ทำงาน**ถูกต้องตามที่ออกแบบไว้ทุกประการ**

> **จุดสำคัญที่สุดของ Scenario นี้**: การแก้ปัญหาด้วยการ **`switchport port-security maximum 20`
> (เพิ่มแบบไม่คิด) เป็นทางเลือกที่ผิด** — เพราะ (1) มันไม่ได้แก้ปัญหาที่ต้นเหตุ (Hub ที่ไม่มีการจัดการ
> ยังเสียบอยู่ ไม่มีใครรู้ว่ามีอุปกรณ์กี่ตัวจริงๆ), (2) มันเปิดช่องให้ MAC Flooding ผ่านได้มากขึ้น 10 เท่า
> โดยไม่มีการพิสูจน์ Business Justification ที่เหมาะสม, (3) มันสร้างมาตรฐานที่ไม่ดี — ถ้าทุกพอร์ตที่มี
> ปัญหาแก้ด้วยการเพิ่ม Maximum ตามใจ Baseline ทั้งบริษัทจะค่อยๆ อ่อนลงจนไม่มีการควบคุมจริงเหลือ

**การตัดสินใจที่ถูกต้อง — วิเคราะห์ 3 ทางเลือกตาม Business Need จริง:**

| ทางเลือก | เมื่อไหร่ควรใช้ | Config |
|---|---|---|
| **(A) ไม่อนุญาต Hub เลย — บังคับตาม Policy เดิม** | ห้องประชุมนี้ไม่ควรมีอุปกรณ์เกิน 2 ตัวตาม Design (Wi-Fi ควรใช้แทน Wired สำหรับผู้เข้าประชุม) | คงค่า Maximum 2 ไว้ — แจ้งทีม Facility ให้ถอด Hub ออก แนะนำให้ใช้ Wireless (VLAN 40 WIFI) แทน |
| **(B) อนุญาต แต่ต้องมี Business Justification + ปรับ Maximum ตามจำนวนจริงที่อนุมัติ** | ห้องประชุมนี้ถูกใช้เป็น Training Room บ่อยที่มีความจำเป็นต้องใช้ Wired จริง (Wi-Fi ไม่พอสำหรับ Bandwidth การทดสอบ) | ปรับ Maximum ตามจำนวนที่**อนุมัติแล้วเท่านั้น** (เช่น 8 ถ้าอนุมัติ Hub 8 พอร์ตจริง) พร้อมเปลี่ยน Violation Mode เป็น `restrict` (ไม่ใช่ `shutdown`) เพื่อไม่ให้ห้องประชุมล่มกลางการนำเสนอถ้ามีคนเสียบเกินเล็กน้อย แต่ยัง Log ไว้ตรวจสอบ |
| **(C) เปลี่ยนโครงสร้าง — ติดตั้ง Managed Switch เล็กแทน Hub** | ต้องการทั้ง Capacity สูงและยังคง Visibility/Security ระดับเดิม | ติดตั้ง Access Switch ขนาดเล็ก (เช่น Catalyst 9200L 8-port) ที่รองรับ 802.1X/DHCP Snooping ของตัวเอง แล้ว Uplink เข้า `Gi1/0/18` แทน Hub — คง Maximum เดิมที่ Uplink (เพราะเห็น MAC ของ Switch ใหม่เป็นตัวเดียว หรือปรับตาม Design เดิมของ Part 21) |

**Config ที่เลือกใช้จริง (ทางเลือก B — อนุมัติ Training Room Use Case):**

```
ACCESS-SW1(config)# interface GigabitEthernet1/0/18
ACCESS-SW1(config-if)# switchport port-security maximum 8
ACCESS-SW1(config-if)# switchport port-security violation restrict
ACCESS-SW1(config-if)# exit

ACCESS-SW1(config)# errdisable recovery cause psecure-violation
```

**Verify:**

```
ACCESS-SW1# show port-security interface GigabitEthernet1/0/18
Port Status                : Secure-up
Violation Mode              : Restrict
Maximum MAC Addresses        : 8
Total MAC Addresses           : 6
Security Violation Count        : 0
```

> **สรุปหลักคิด Security-vs-Usability Tradeoff**: คำถามที่ต้องถามเสมอไม่ใช่ "จะเพิ่ม Maximum เท่าไหร่
> ถึงจะพอ" แต่คือ **"จำนวนอุปกรณ์ที่ได้รับอนุญาตจริงคือเท่าไหร่ ตามที่ Business อนุมัติ"** — ตัวเลขที่
> ตั้งควรสะท้อน **Approved Device Count** เสมอ ไม่ใช่ตัวเลขที่ "กว้างพอจะไม่มีใครมาบ่นอีก" — และการ
> เปลี่ยน Violation Mode จาก `shutdown` เป็น `restrict` ไม่ได้ลดความปลอดภัยลง (ยังคง Drop Traffic ของ
> MAC ที่เกิน Limit เหมือนเดิม) แต่ลดผลกระทบต่อ Usability ของอุปกรณ์ที่ได้รับอนุญาตอยู่แล้วเท่านั้น
> ตามหลักการเดียวกับที่อธิบายไว้ใน [Part 21 Step 203.1](part-021-switch-security.md)

---

## Step 610 — Lab เต็มรูปแบบ: 3 Layered Outage พร้อมกัน (DHCP Snooping + 802.1X Critical VLAN + CoPP)

### 610.1 บริบทของ Lab — ทำไมต้องซ้อนกัน 3 ปัญหา

ข้อสอบ ENARSI Ticket-based Simulation (และสถานการณ์จริงในสนาม) มักไม่ให้ปัญหาเดียวโดดๆ — เหตุการณ์
ใหญ่ (Major Incident) มักเกิดจาก **หลายปัจจัยที่ไม่เกี่ยวข้องกันเกิดขึ้นพร้อมกันโดยบังเอิญ** Lab นี้
จำลองคืนวันเสาร์ที่มี 3 เหตุการณ์เกิดพร้อมกันบน Topology เดียวกัน:

```
เวลา 22:00 — พายุฝนทำให้ Fiber CORE-SW2 ↔ DIST-SW3/DIST-SW4 Flap ต่อเนื่อง (เหมือน Step 608)
เวลา 22:05 — ทีม Security เริ่ม Maintenance Window Patch ISE-1 ตามแผน (คาดว่าเสร็จใน 22:15)
เวลา 22:10 — ทีม Network เปลี่ยน ACCESS-SW4 (Hardware ใหม่) ตาม RMA และ Restore Config จาก Backup
             ที่ไม่ครบ (เหมือน Step 602)
เวลา 23:30 — ISE-1 Patch ล้มเหลว ต้อง Rollback (ใช้เวลานานกว่าแผนมาก) — ยังไม่กลับมาจนถึงเช้า
เวลา 06:00 วันอาทิตย์ — พนักงาน Early Shift มาทำงาน รายงานปัญหาพร้อมกัน 3 เรื่อง
```

### 610.2 อาการที่ Helpdesk ได้รับรายงานพร้อมกัน (สร้างความสับสนตอนแรก)

| # | ผู้รายงาน | อาการ |
|---|---|---|
| 1 | ทีม NOC (Monitoring) | OSPF Neighbor ระหว่าง CORE-SW2 ↔ DIST-SW3/DIST-SW4 Flap ต่อเนื่องมาตั้งแต่คืนที่แล้ว |
| 2 | พนักงานใหม่ที่เพิ่งเข้าประจำ ACCESS-SW4 | เสียบ PC แล้วไม่ได้ IP เลย (VLAN 30 SERVERS ฝั่งที่ ACCESS-SW4 ดูแล) |
| 3 | พนักงาน Sales ที่เสียบ Notebook เข้า ACCESS-SW1/2 (VLAN 10) | Login เข้าเครือข่ายไม่ได้ ค้างที่หน้าจอ "Authenticating..." |

> **กับดักสำคัญที่ Junior Engineer มักพลาด**: เห็นปัญหา 3 เรื่องพร้อมกันแล้วสงสัยว่าเป็น "ปัญหาเดียว
> ใหญ่" เช่น "Core ล่มหมด" หรือ "AAA-SRV ทั้งระบบล่ม" — ทั้งที่จริงแล้วเป็น **3 ปัญหาที่แยกกันคนละ
> Root Cause คนละ Device โดยสิ้นเชิง** ที่บังเอิญเกิดพร้อมกันในคืนเดียว ต้อง Diagnose **แยกทีละเรื่อง**
> ตาม Framework Step 601.3 ไม่ใช่หาสาเหตุร่วมเดียวที่ไม่มีจริง

### 610.3 การจัดลำดับความสำคัญ (Triage) ก่อนเริ่ม Diagnose

```
┌────────────────────────────────────────────────────────────────────┐
│  ลำดับ Priority ที่ถูกต้อง (Business Impact สูงสุดก่อน)                  │
├────────────────────────────────────────────────────────────────────┤
│  1st: OSPF Flap (Core Network)     → กระทบทั้งองค์กร ถ้า Convergence   │
│       ไม่เสร็จ Route อาจหายเป็นบาง Segment                             │
│  2nd: 802.1X Login ค้าง (Sales)    → กระทบ Sales Team ทั้งชั้น ธุรกิจ  │
│       เดินไม่ได้ทันที                                                  │
│  3rd: DHCP ไม่ได้ IP (1 พนักงานใหม่) → กระทบคนเดียว ยังรอได้ระยะสั้น      │
└────────────────────────────────────────────────────────────────────┘
```

### 610.4 การ Diagnose ปัญหาที่ 1 — OSPF Flap จาก CoPP (เหมือน Step 608)

```
CORE-SW2# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.13          1   INIT/  -        00:00:29    10.255.30.2     TenGigabitEthernet1/0/1
1.1.1.14          1   EXSTART/ -      00:00:34    10.255.40.2     TenGigabitEthernet1/0/2

CORE-SW2# show policy-map control-plane class COPP-CRITICAL
        exceeded 227410 packets, 29088480 bytes; actions:
          drop
```

**Diagnose (สรุปสั้น — กระบวนการเดียวกับ Step 608 ทุกประการ)**: `show access-list COPP-ACL-ROUTING`
ยืนยัน ACL ถูกต้อง → Root Cause คือ `cir` ของ `COPP-CRITICAL` ต่ำเกินไปสำหรับ LSA Flooding ช่วง Flap
2 เส้นพร้อมกัน

**Fix:**

```
CORE-SW2(config)# policy-map COPP-POLICY
CORE-SW2(config-pmap)# class COPP-CRITICAL
CORE-SW2(config-pmap-c)# police cir 20000000 bc 1024000
CORE-SW2(config-pmap-c-police)#  conform-action transmit
CORE-SW2(config-pmap-c-police)#  exceed-action drop
```

```
CORE-SW2# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.13          1   FULL/  -        00:00:39    10.255.30.2     TenGigabitEthernet1/0/1
1.1.1.14          1   FULL/  -        00:00:36    10.255.40.2     TenGigabitEthernet1/0/2
```

### 610.5 การ Diagnose ปัญหาที่ 2 — 802.1X ค้าง เพราะ ISE-1 ยังล่มอยู่ (เหมือน Step 604)

```
ACCESS-SW1# show authentication sessions interface GigabitEthernet1/0/3 details | include Status|Vlan
                   Status:  Authorized
              Vlan Policy:  91

ACCESS-SW2# show authentication sessions interface GigabitEthernet1/0/2 details | include Status|Vlan
                   Status:  Authz Failed
              Vlan Policy:  N/A
```

**สังเกตความแตกต่างสำคัญ**: ACCESS-SW1 Authorize เข้า VLAN 91 (Critical VLAN) ได้สำเร็จ — แปลว่า
Config ของ ACCESS-SW1 **ถูกต้อง** และ Graceful Degradation ทำงานตามที่ Design ไว้ ส่วน ACCESS-SW2
ล้มเหลว — ต้องสงสัยปัญหาเฉพาะที่ ACCESS-SW2 ทันที ไม่ใช่ปัญหาของ ISE ล่มเพียงอย่างเดียว (เพราะถ้าเป็น
เพราะ ISE ล่มเท่านั้น ACCESS-SW1 ก็ควรมีอาการเดียวกัน)

```
*Sep 27 06:02:11.334: %DOT1X-5-ERROR_VLAN_INVALID: Vlan 19 configured for authorization does
not exist. Client will remain unauthorized on Interface Gi1/0/2
```

**Root Cause**: เหมือน Step 604 ทุกประการ — `authorize vlan 19` พิมพ์ผิดจาก `91` บน ACCESS-SW2

**Fix:**

```
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1-3
ACCESS-SW2(config-if-range)# no authentication event server dead action authorize vlan 19
ACCESS-SW2(config-if-range)# authentication event server dead action authorize vlan 91
ACCESS-SW2(config-if-range)# exit
```

```
ACCESS-SW2# show authentication sessions interface GigabitEthernet1/0/2 details | include Status|Vlan
                   Status:  Authorized
              Vlan Policy:  91
```

> **หมายเหตุ**: ต่างจาก Step 604 ที่แก้ VLAN Typo แล้วจบเลย ในสถานการณ์นี้ **ต้องรอ ISE-1 กลับมา
> Online จริง** ก่อนที่พนักงาน Sales จะได้ VLAN 10 ปกติ — ในระหว่างนี้ VLAN 91 (Critical) คือสถานะ
> ที่ "ถูกต้องตามที่ควรเป็น" แล้ว (จำกัดสิทธิ์ แต่ใช้งานพื้นฐานได้) ไม่ต้องพยายามแก้ให้ได้ VLAN 10 ทันที
> เพราะนั่นหมายถึงการ Bypass 802.1X ทั้งระบบ ซึ่งผิดหลักการ Step 601.2

### 610.6 การ Diagnose ปัญหาที่ 3 — DHCP ไม่ได้ IP เพราะ Uplink ไม่ Trusted (เหมือน Step 602)

```
ACCESS-SW4# show ip dhcp snooping | include Gi0/
GigabitEthernet0/1          no          no                unlimited
GigabitEthernet0/2          no          no                unlimited

ACCESS-SW4# debug ip dhcp snooping event
*Sep 27 06:05:44: DHCP_SNOOPING: DHCPOFFER received on untrusted port, dropping the packet.
```

**Root Cause**: เหมือน Step 602 ทุกประการ — ACCESS-SW4 ถูก Restore จาก Backup เก่าหลัง RMA
Hardware และไม่มีบรรทัด `ip dhcp snooping trust` บน Uplink

**Fix:**

```
ACCESS-SW4(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW4(config-if-range)# ip dhcp snooping trust
ACCESS-SW4(config-if-range)# exit
```

```
ACCESS-SW4# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00aa.bb11.2233       10.10.30.55      86400       dhcp-snooping   30    GigabitEthernet1/0/4
Total number of bindings: 1
```

### 610.7 สรุปผล Lab — ตาราง After-Action Report

| # | ปัญหา | Device | Root Cause | เวลาที่แก้ | Independent จากอีก 2 ปัญหา? |
|---|---|---|---|---|---|
| 1 | OSPF Flap | CORE-SW2 | CoPP `COPP-CRITICAL` CIR ต่ำเกินไปสำหรับ LSA Flooding ตอน Multi-Link Flap | เร่งด่วนสุด (Business Impact สูงสุด) | ใช่ — เกิดจาก Fiber เสื่อมสภาพ ไม่เกี่ยวกับ ISE/DHCP เลย |
| 2 | 802.1X ค้าง | ACCESS-SW2 | ISE-1 ล่มจริง (คาดการณ์ได้ Design รองรับ) + VLAN Typo (`19`→`91`) ที่ทำให้ Fallback ล้มเหลวซ้อน | รองลงมา | ใช่ — Typo เป็นความผิดพลาดแยกจาก Config ของ ACCESS-SW4/CORE-SW2 |
| 3 | DHCP ไม่ได้ IP | ACCESS-SW4 | Restore Backup เก่าหลัง RMA Hardware ไม่มี `ip dhcp snooping trust` | ท้ายสุด (กระทบคนเดียว) | ใช่ — เกิดจาก Hardware RMA คนละเวลา คนละสาเหตุ |

> **บทเรียนสุดท้ายของ Part นี้**: แม้ 3 ปัญหาจะดูน่าตกใจตอนมาพร้อมกัน แต่ **แต่ละปัญหายังตามกระบวนการ
> 5-Step Framework จาก Step 601.3 แยกกันได้อย่างชัดเจน** — Diagnostic Framework ที่ดีไม่ควรเปลี่ยนไป
> ตามจำนวนปัญหาที่เกิดพร้อมกัน สิ่งที่เปลี่ยนคือ **การจัดลำดับความสำคัญ (Triage)** และ **ความอดทนที่จะ
> ไม่ปนสาเหตุของปัญหาหนึ่งเข้ากับอีกปัญหา** แม้ทั้งหมดจะรายงานเข้ามาในช่วงเวลาใกล้เคียงกันก็ตาม — ทักษะ
> นี้คือสิ่งที่ข้อสอบ ENARSI Ticket-based Simulation และ Incident จริงในที่ทำงานทดสอบมากที่สุด

---

## แบบฝึกหัดทวนความเข้าใจ Part 61

1. เพราะเหตุใดการปิดฟีเจอร์ Security ทั้งหมดทิ้ง (เช่น `no ip dhcp snooping` หรือ
   `aaa authentication login default none`) จึงเป็นทางลัดที่อันตราย แม้จะดู "แก้ปัญหาได้ทันที"?
2. ใน Scenario DAI (Step 603) ทำไม `show ip arp inspection statistics` ของ VLAN 30 จึงเห็น
   `ACL Drops` เพิ่มขึ้นแต่ `DHCP Drops` เป็น 0 เสมอ ทั้งที่ Binding Table ก็ไม่มี Entry ของ Server
   นั้นด้วย?
3. ใน Scenario TrustSec (Step 605) ทำไม PC1 จึงยังคง "คุยไม่ได้" แม้ SGACL Matrix จะอนุญาต
   SALES→SERVERS ไว้แล้วอย่างถูกต้อง? อธิบายว่า SGT ที่มาถึง Enforcement Point ตอนนั้นคืออะไร
4. อธิบายความแตกต่างของ ZBFW ระหว่าง (ก) Traffic ระหว่าง Interface สองเส้นที่ **ไม่ได้เข้า Zone ใด
   เลยทั้งคู่** กับ (ข) Traffic ระหว่าง Interface ที่เข้า Zone แล้ว 1 เส้น กับ Interface ที่**ไม่ได้
   เข้า Zone เลย** อีก 1 เส้น — ทำไมพฤติกรรม Default ของทั้งสองกรณีจึงต่างกัน?
5. ใน Scenario Port Security (Step 609) ทำไมการเพิ่ม `switchport port-security maximum` แบบไม่
   พิจารณา Business Justification จึงเป็นทางเลือกที่ผิด แม้จะทำให้พอร์ตไม่ err-disabled อีกก็ตาม?

**เฉลย:**

1. เพราะการปิดฟีเจอร์ทั้งหมดแก้ปัญหาที่ผิวเผิน (Traffic ที่ควรผ่านตอนนี้ผ่านได้) แต่เปิดช่องโหว่ด้าน
   Security ทั้งหมดที่ฟีเจอร์นั้นออกแบบมาป้องกันไปด้วย (เช่น Rogue DHCP Server, Unauthorized Login)
   — วิธีที่ถูกต้องคือหา "Gap" เฉพาะจุดที่ทำให้ Traffic ที่ถูกต้องติดขัด แล้วแก้เฉพาะจุดนั้น โดยให้ Security
   Posture ส่วนที่เหลือทำงานปกติ 100% ตามหลักการ Step 601.2
2. เพราะ VLAN 30 ตั้ง `ip arp inspection filter STATIC-SERVERS vlan 30 static` (มี Keyword
   `static`) — DAI จะตรวจสอบกับ ARP ACL **เท่านั้น** และ**ไม่ Fallback ไป DHCP Snooping Binding
   Table เลย** ตามที่อธิบายใน [Part 21 Step 207.3](part-021-switch-security.md) จึงเห็น Drop
   ทั้งหมดถูกนับเป็น `ACL Drops` แม้ Binding Table จะไม่มี Entry อยู่แล้วก็ตาม เพราะไม่มีการเช็ค
   Binding Table เลยตั้งแต่ต้น
3. เพราะ SGT ของ PC1 ไม่ได้ถูกส่งไปถึง DIST-SW3 (Enforcement Point) เนื่องจาก SXP Connection ระหว่าง
   ACCESS-SW1 ↔ DIST-SW1 ล้มเหลว (Password Mismatch) — Traffic ของ PC1 จึงมาถึง DIST-SW3 โดยไม่มี
   SGT ติดมา (`Unknown`) ซึ่งตรงกับ Default Deny Policy ของ Matrix ไม่ใช่ SGT=100 (SALES) ตามที่
   ควรจะเป็น — SGACL จึง Deny ตาม Policy ของ `Unknown` แทน แม้ Matrix จริงจะอนุญาต 100→120 ก็ตาม
4. (ก) Traffic ระหว่าง Interface ที่ไม่ได้เข้า Zone เลยทั้งคู่ยัง Route ได้ปกติเหมือนไม่มี ZBFW เลย
   (เพราะยังไม่มี Zone Boundary ใดๆ เกี่ยวข้อง) ส่วน (ข) ทันทีที่ Interface หนึ่งเส้นถูกใส่เข้า Zone
   ใดๆ ก็ตาม กฎ Default ของ ZBFW ถือว่า **ทุก Traffic ที่ข้ามจาก Interface ที่มี Zone ไปยัง Interface
   ที่ไม่มี Zone (หรือ Zone อื่น) ต้องถูก Deny เว้นแต่มี Zone-Pair/Policy อนุญาตไว้ชัดเจน** — นี่คือ
   เหตุผลที่ Sub-interface ใหม่ใน Step 607 ต้องเพิ่ม `zone-member security` แยกเสมอ แม้ Physical
   Interface หลักจะอยู่ใน Zone แล้วก็ตาม
5. เพราะการเพิ่ม Maximum แบบไม่พิจารณา Business Justification ไม่ได้แก้ที่ต้นเหตุ (ยังไม่รู้ว่ามี
   อุปกรณ์ไม่ได้รับอนุญาตกี่ตัวเสียบอยู่จริง) และเปิดช่องให้ MAC Flooding ผ่านได้มากขึ้นโดยไม่มีการ
   ควบคุม — ค่า Maximum ที่ถูกต้องควรสะท้อน **Approved Device Count** ที่ผ่านการอนุมัติจริงเท่านั้น
   ไม่ใช่ตัวเลขที่ "กว้างพอจนไม่มีใครมาบ่นอีก" มิฉะนั้น Baseline Security ของทั้งองค์กรจะค่อยๆ อ่อนลง

---

## สรุป Part 61

Part นี้นำฟีเจอร์ Security ทั้งหมดที่ Config ไว้ตลอดหลักสูตร CCNP ENARSI — Port Security, DHCP
Snooping, DAI ([Part 21](part-021-switch-security.md)), TACACS+ AAA
([Part 22](part-022-aaa-device-hardening.md)), 802.1X/TrustSec
([Part 44](part-044-dot1x-trustsec.md)), ZBFW ([Part 45](part-045-advanced-acl-zbfw.md)), และ CoPP
([Part 46](part-046-control-plane-policing.md)) — มาผ่านกระบวนการ Troubleshooting แบบ ENARSI Ticket
เต็มรูปแบบ 8 Scenario บวก Lab สุดท้ายที่รวม 3 ปัญหาเกิดพร้อมกัน หัวใจสำคัญที่สุดที่ต้องจำคือ **Security
Feature ที่บล็อก Traffic ที่ถูกต้องมักไม่ได้ "พัง" — มันทำงานตามที่ Config ไว้อย่างถูกต้อง เพียงแต่
Config ไม่ได้ครอบคลุมการเปลี่ยนแปลงของ Network** และทางแก้ที่ถูกต้องคือการหา Gap นั้นแล้วแก้ให้แคบและ
ตรงจุดที่สุด ไม่ใช่การปิดฟีเจอร์ทั้งหมดทิ้งไป

**พร้อมสำหรับ Part 62**: เราจะขยับไป Troubleshoot **Infrastructure Services** — DNS, DHCP Relay/
Scope Exhaustion, NTP Drift, SNMP/Syslog ที่หยุดทำงาน, และ NAT/PAT Translation ที่ผิดพลาด — กลุ่ม
Service พื้นฐานที่ Network ทั้งองค์กรพึ่งพา แต่มักถูกมองข้ามในการ Troubleshoot เพราะไม่ใช่ Routing/
Switching/Security โดยตรง

**ไปต่อ:** [Part 62 — Infrastructure Services Troubleshooting →](part-062-infra-services-troubleshooting.md)
