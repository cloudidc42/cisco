# Part 45 — Advanced ACL & Zone-Based Firewall (ZBFW)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 441–450 จาก 1000**

> ต่อจาก [Part 44 — 802.1X & Cisco TrustSec](part-044-dot1x-trustsec.md) ที่เราปิดท้ายเรื่อง
> **Identity-Based Access Control** ที่ Layer 2 (ใครเสียบสาย/ต่อ Wi-Fi ได้บ้าง) Part นี้จะดึง
> ความรู้เรื่อง **ACL** จาก [Part 14](part-014-access-control-lists.md) กลับมาต่อยอดให้ทันสมัยและ
> ใช้งานจริงในระดับ Enterprise มากขึ้น (Object-Group, Time-based ACL, Reflexive ACL) ก่อนจะ
> ก้าวข้ามไปสู่แนวคิดที่**เปลี่ยนกระบวนทัศน์ของ Security บน Router/Switch โดยสิ้นเชิง** —
> **IOS Zone-Based Firewall (ZBFW)** ซึ่งเป็นก้าวสำคัญของ CCNP ENCOR ที่แยก Router ธรรมดา
> ออกจาก Router ที่ทำหน้าที่ **Stateful Firewall** ได้ในตัว

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 441 | ทวนและขยาย ACL ขั้นสูง — Object-Group (`object-group network`/`service`) เพื่อความอ่านง่าย/ดูแลง่าย |
| 442 | Time-based ACL — `time-range` จำกัดสิทธิ์ตามช่วงเวลา (เช่น Contractor VLAN เข้าได้เฉพาะเวลาทำการ) |
| 443 | Reflexive ACL — ACL ที่จำ session ได้ (`reflect`/`evaluate`) เพื่อพฤติกรรมกึ่ง Stateful บน Router |
| 444 | IOS Zone-Based Firewall (ZBFW) พื้นฐาน — Zone, Zone-Pair, Self-Zone, Default-Deny-Between-Zones |
| 445 | คอนฟิก ZBFW เต็มรูปแบบ — `zone security`, `zone-member`, `class-map`/`policy-map type inspect`, `zone-pair` |
| 446 | Stateful Inspection ของ ZBFW — `inspect` action ติดตาม Session และ Application-Layer Inspection |
| 447 | ปกป้อง Self-Zone — Management Plane Security ผ่าน Self-Zone เทียบกับ Part 22 |
| 448 | เปรียบเทียบ ZBFW vs Reflexive ACL vs Plain ACL — ใช้อะไรเมื่อไหร่ |
| 449 | Verification/Troubleshooting ZBFW และ Reflexive ACL |
| 450 | Lab ใหญ่: Deploy ZBFW เต็มรูปแบบบน WAN-EDGE-1 (INSIDE/OUTSIDE/SELF) |

---

## Step 441 — ทวนและขยาย ACL ขั้นสูงด้วย Object-Group

### ปัญหาของ ACL แบบดั้งเดิมเมื่อ Enterprise Lab โตขึ้น

ACL ที่เราเขียนใน [Part 14](part-014-access-control-lists.md) (เช่น `SALES-TO-SERVERS`) ใช้
IP Address และ Port ตรงๆ ในทุกบรรทัด ซึ่งใช้งานได้ดีตอน Policy ยังเล็ก แต่พอ Enterprise Lab
มี Server เพิ่มขึ้น (Server1, Server2, Server3, ...) หรือมี Port Service หลายตัวที่ต้องอนุญาต
พร้อมกัน (HTTPS + SSH + RDP) ACL จะ**ยาวและซ้ำซ้อน**อย่างรวดเร็ว — ทุกครั้งที่เพิ่ม Server ใหม่
ต้องกลับไปแก้ทุกบรรทัดที่เกี่ยวข้อง เสี่ยงพิมพ์ผิดหรือลืมแก้บางบรรทัด

**Object-Group** คือ Cisco IOS Feature ที่ให้เรา**จัดกลุ่ม IP Address หรือ Port หลายตัวไว้ในชื่อ
เดียว** แล้วอ้างอิงชื่อกลุ่มนั้นใน ACL แทนการพิมพ์ IP/Port ซ้ำๆ ทุกบรรทัด — ผลลัพธ์คือ ACL ที่
**อ่านง่ายขึ้นมาก อัปเดตง่ายขึ้นมาก** (แก้ที่ Object-Group จุดเดียว มีผลกับทุกบรรทัด ACL ที่อ้างอิง
กลุ่มนั้นทันที โดยไม่ต้องไปไล่แก้ ACE ทีละบรรทัด)

### Object-Group มี 2 ประเภทหลัก

| ประเภท | คำสั่ง | ใช้จัดกลุ่ม |
|---|---|---|
| `object-group network` | จัดกลุ่ม IP Address/Subnet/Host | Server1, Server2, Server3 → กลุ่มเดียว |
| `object-group service` | จัดกลุ่ม Protocol + Port | HTTPS, SSH, RDP → กลุ่มเดียว |

### Syntax

```
Router(config)# object-group network <name>
Router(config-network-group)# host <ip>
Router(config-network-group)# <network> <subnet-mask หรือ /prefix>
Router(config-network-group)# group-object <name-of-another-object-group>

Router(config)# object-group service <name>
Router(config-service-group)# tcp eq <port>
Router(config-service-group)# udp eq <port>
Router(config-service-group)# tcp range <port1> <port2>
```

> **ข้อสังเกตสำคัญ**: `object-group network` ใช้ **Subnet Mask ปกติ** (ไม่ใช่ Wildcard Mask
> แบบ ACE ทั่วไป) หรือใช้ Prefix Length ก็ได้ — เป็นจุดที่มักสร้างความสับสนให้วิศวกรที่คุ้นกับ
> Wildcard Mask ของ ACL มาตลอด

### Worked Example: แปลง ACL จาก Part 14 ให้ใช้ Object-Group

**สถานการณ์ต่อยอด**: ทีม Security ของ Enterprise Lab ขยาย Requirement เดิม (Part 14 Step 134)
ให้ครอบคลุมมากขึ้น — SALES ต้องเข้าถึง **Server1 และ Server3** (ไม่ใช่แค่ Server1) ด้วย
**HTTPS, SSH, และ RDP** (3 Port พร้อมกัน) แทนที่จะเขียน ACE 6 บรรทัด (2 Host × 3 Port) เรา
จัดกลุ่มด้วย Object-Group แทน:

```
DIST-SW3(config)# object-group network SERVERS-SALES-ALLOWED
DIST-SW3(config-network-group)# host 10.10.30.10
DIST-SW3(config-network-group)# host 10.10.30.30
DIST-SW3(config-network-group)# exit

DIST-SW3(config)# object-group service ADMIN-PORTS
DIST-SW3(config-service-group)# tcp eq 443
DIST-SW3(config-service-group)# tcp eq 22
DIST-SW3(config-service-group)# tcp eq 3389
DIST-SW3(config-service-group)# exit

DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS-V2
DIST-SW3(config-ext-nacl)# remark ==== SALES -> Server1/Server3 via HTTPS/SSH/RDP (Object-Group) ====
DIST-SW3(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 object-group SERVERS-SALES-ALLOWED object-group ADMIN-PORTS
DIST-SW3(config-ext-nacl)# remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
DIST-SW3(config-ext-nacl)# deny   ip  10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
DIST-SW3(config-ext-nacl)# permit ip any any
DIST-SW3(config-ext-nacl)# exit
```

หนึ่งบรรทัด `permit tcp ... object-group SERVERS-SALES-ALLOWED object-group ADMIN-PORTS`
เทียบเท่ากับ ACE **6 บรรทัด** ถ้าเขียนแบบดั้งเดิม (2 Host × 3 Port) — และถ้าอนาคตต้องเพิ่ม
Server4 เข้ามาในสิทธิ์นี้ ทีมงานเพียงแก้ที่ `object-group network SERVERS-SALES-ALLOWED`
จุดเดียว (`host 10.10.30.40`) โดยไม่ต้องแก้ ACL เลยแม้แต่บรรทัดเดียว

### ตรวจสอบผลลัพธ์

```
DIST-SW3# show object-group
Network object group SERVERS-SALES-ALLOWED
 host 10.10.30.10
 host 10.10.30.30
Service object group ADMIN-PORTS
 tcp eq 443
 tcp eq 22
 tcp eq 3389

DIST-SW3# show ip access-lists SALES-TO-SERVERS-V2
Extended IP access list SALES-TO-SERVERS-V2
    10 remark ==== SALES -> Server1/Server3 via HTTPS/SSH/RDP (Object-Group) ====
    20 permit tcp 10.10.10.0 0.0.0.255 object-group SERVERS-SALES-ALLOWED object-group ADMIN-PORTS
    30 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    40 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    50 permit ip any any
```

> **ข้อจำกัดที่ต้องรู้**: Object-Group ACL (บางครั้งเรียก "Object-Group ACL" หรือใน Platform อื่น
> เรียก "Object-Group-Based ACE") ต้องการ IOS/IOS-XE ที่รองรับ Feature นี้ (Catalyst 9000/ISR4000
> รุ่นใหม่รองรับครบ) และ `show access-lists` จะแสดง ACE เป็นชื่อกลุ่ม **ไม่ขยาย (expand)** ให้เห็น
> รายชื่อ IP/Port ทั้งหมด — ถ้าต้องการดู Effective Rule จริงต้องเช็ค `show object-group` ควบคู่กันไป
> เสมอ

---

## Step 442 — Time-based ACL: จำกัดสิทธิ์ตามช่วงเวลาด้วย `time-range`

### แนวคิด

ACL ปกติที่เรียนมาทั้งหมดจะ **permit/deny ตลอด 24 ชั่วโมง 7 วัน** โดยไม่สนใจเวลา — แต่ใน
Enterprise จริงมักมี Requirement แบบ **"อนุญาตเฉพาะช่วงเวลาทำการ"** เช่น Contractor ที่มาทำงาน
ชั่วคราวควรเข้าถึงระบบได้แค่ในเวลางาน ไม่ใช่ตลอด 24 ชั่วโมง (ลดพื้นที่โจมตีในช่วงเวลาที่ไม่มีใคร
เฝ้าระบบ) — Cisco IOS แก้ปัญหานี้ด้วยคำสั่ง **`time-range`** ที่ผูกเข้ากับ ACE แต่ละบรรทัดได้

### Syntax

```
Router(config)# time-range <name>
Router(config-time-range)# periodic <days> <start-time> to <end-time>
Router(config-time-range)# absolute start <hh:mm> <date> [end <hh:mm> <date>]
```

| รูปแบบ | ใช้เมื่อ | ตัวอย่าง |
|---|---|---|
| `periodic` | เกิดซ้ำเป็นประจำ (รายวัน/รายสัปดาห์) | `periodic weekdays 08:00 to 18:00` |
| `absolute` | ช่วงเวลาครั้งเดียว มีวันเริ่ม/วันหมดอายุ | `absolute start 00:00 1 Jan 2027 end 23:59 31 Dec 2027` |

> **ข้อควรระวังเรื่อง Clock**: `time-range` อ้างอิงตาม **Clock ของอุปกรณ์เอง** — ถ้า Router/Switch
> ยังไม่ Sync เวลาให้ถูกต้องด้วย **NTP** (สอนไว้ใน [Part 17](part-017-ntp-syslog-snmp.md))
> `time-range` จะทำงานผิดเวลาโดยสิ้นเชิง (คลาดเคลื่อนตามเวลาของอุปกรณ์ที่อาจไม่ตรงกับความเป็นจริง)
> — **NTP คือ Prerequisite ที่ขาดไม่ได้ก่อนใช้ Time-based ACL ใน Production เสมอ**

### Worked Example: Contractor VLAN เข้าถึง SERVERS ได้เฉพาะเวลาทำการ

**สถานการณ์**: ทีม Facility เพิ่ม Network สำหรับ Contractor ชั่วคราวเข้ามา (สมมติเพิ่มเพื่อสาธิต
Time-based ACL เท่านั้น — **ไม่ได้เพิ่มเข้า [IP Address Plan หลัก](00-ip-address-plan.md)**
เพราะเป็น Scenario ประกอบการสอนเฉพาะ Step นี้) ใช้ Subnet `10.10.50.0/24` โดย Requirement คือ
Contractor เข้าถึง Server1 (`10.10.30.10`) ผ่าน HTTPS ได้เฉพาะ **จันทร์-ศุกร์ 08:00-18:00**
เท่านั้น นอกเวลานี้ต้อง Deny ทั้งหมด แม้จะพยายามเชื่อมต่อ Protocol/Port เดียวกันก็ตาม

```
DIST-SW3(config)# time-range BUSINESS-HOURS
DIST-SW3(config-time-range)# periodic weekdays 08:00 to 18:00
DIST-SW3(config-time-range)# exit

DIST-SW3(config)# ip access-list extended CONTRACTOR-TO-SERVER1
DIST-SW3(config-ext-nacl)# remark ==== Contractor VLAN -> Server1 HTTPS, business hours only ====
DIST-SW3(config-ext-nacl)# permit tcp 10.10.50.0 0.0.0.255 host 10.10.30.10 eq 443 time-range BUSINESS-HOURS
DIST-SW3(config-ext-nacl)# remark ==== Deny everything else from Contractor VLAN, log attempts ====
DIST-SW3(config-ext-nacl)# deny   ip  10.10.50.0 0.0.0.255 any log
DIST-SW3(config-ext-nacl)# permit ip any any
DIST-SW3(config-ext-nacl)# exit

DIST-SW3(config)# interface Vlan50
DIST-SW3(config-if)# ip access-group CONTRACTOR-TO-SERVER1 in
DIST-SW3(config-if)# exit
```

**หลักการทำงาน**: บรรทัด `permit ... time-range BUSINESS-HOURS` จะ **Active เฉพาะช่วงเวลาที่
กำหนดเท่านั้น** — นอกเวลานั้น IOS จะข้ามบรรทัดนี้ไปเหมือนไม่มีอยู่ (ไม่ match เลย ไม่ว่า
Source/Destination/Port จะตรงแค่ไหนก็ตาม) แล้วตกไปที่บรรทัด `deny` แทน

### ตรวจสอบสถานะ Time-range และผลลัพธ์

```
DIST-SW3# show time-range BUSINESS-HOURS
time-range entry: BUSINESS-HOURS (inactive)
  periodic weekdays 8:00 to 18:00

DIST-SW3# show ip access-lists CONTRACTOR-TO-SERVER1
Extended IP access list CONTRACTOR-TO-SERVER1
    10 remark ==== Contractor VLAN -> Server1 HTTPS, business hours only ====
    20 permit tcp 10.10.50.0 0.0.0.255 host 10.10.30.10 eq 443 time-range BUSINESS-HOURS (inactive) (0 matches)
    30 remark ==== Deny everything else from Contractor VLAN, log attempts ====
    40 deny ip 10.10.50.0 0.0.0.255 any log (14 matches)
    50 permit ip any any (203 matches)
```

**ทดสอบ 2 ช่วงเวลา**:

| ช่วงเวลาทดสอบ | สถานะ Time-range | ผลลัพธ์การเชื่อมต่อ HTTPS |
|---|---|---|
| วันจันทร์ 10:00 น. | `(active)` | สำเร็จ (match line 20) |
| วันจันทร์ 20:00 น. หรือวันเสาร์ | `(inactive)` | ถูก deny — ตกไปที่ line 40, เห็น log |

> **ข้อควรระวังที่พบบ่อย**: `time-range` ไม่ได้ "ปิด Interface" หรือ "Shutdown VLAN" — มันแค่ทำให้
> **ACE บรรทัดนั้นไม่ Active** ชั่วคราว ดังนั้นต้องออกแบบบรรทัด `deny`/`permit` ที่เหลือให้ครอบคลุม
> พฤติกรรมที่ต้องการทั้งในและนอกเวลาทำการเสมอ (เหมือน Pitfall เรื่อง Implicit Deny ใน
> [Part 14 Step 138](part-014-access-control-lists.md))

---

## Step 443 — Reflexive ACL: ACL ที่จำ Session ได้ (`reflect`/`evaluate`)

### ปัญหาที่ Reflexive ACL แก้ไข

[Part 14 Step 138](part-014-access-control-lists.md) แนะนำ keyword **`established`** เพื่อ
อนุญาต Return Traffic ของ Session ที่มีอยู่แล้ว แต่ `established` มีข้อจำกัดสำคัญ 2 อย่าง:

1. ใช้ได้กับ **TCP เท่านั้น** (ดู ACK/RST flag) — UDP และ ICMP ใช้ไม่ได้เลย
2. เป็นการตรวจสอบแบบ **Stateless จริงๆ** — ดูแค่ flag ของ packet เดี่ยวๆ ไม่ได้สร้าง "รายการ
   Session ที่กำลัง Active อยู่จริง" ผู้โจมตีสามารถปลอม ACK flag ขึ้นมาเองแล้วผ่าน ACL ได้โดยไม่
   ต้องมี Session จริงเกิดขึ้นก่อนเลย

**Reflexive ACL** แก้ทั้งสองข้อนี้: มันจะ **สร้าง Temporary ACE ขึ้นมาโดยอัตโนมัติ** ทุกครั้งที่มี
Traffic ขาออก (Outbound) ที่ Match บรรทัดที่มี keyword `reflect` — Temporary ACE นี้จะอนุญาต
เฉพาะ Return Traffic ที่ตรงกับ **Session จริง** (Source/Destination IP+Port สลับกัน) เท่านั้น
และจะ **หมดอายุอัตโนมัติ** (Timeout) เมื่อ Session ปิดหรือไม่มี Traffic นานเกินกำหนด — ทำงานได้ทั้ง
TCP, UDP, และ ICMP (ต่าง กับ `established` ที่ใช้ได้แค่ TCP)

### Syntax

```
Router(config)# ip access-list extended <outbound-acl-name>
Router(config-ext-nacl)# permit <protocol> <source> <destination> reflect <reflexive-name> [timeout <seconds>]

Router(config)# ip access-list extended <inbound-acl-name>
Router(config-ext-nacl)# evaluate <reflexive-name>
```

**หลักการ**: ACL สองตัวทำงานคู่กันเสมอ — ตัวหนึ่ง apply ที่ทิศทาง**ขาออก** (มี `reflect`
สร้าง Temporary Entry) อีกตัว apply ที่ทิศทาง**ขาเข้า** (มี `evaluate` ไปตรวจสอบ Temporary
Entry ที่ถูกสร้างไว้)

### Worked Example: ป้องกัน WAN-EDGE-1 แบบ Stateful-like โดยไม่มี Firewall เต็มรูปแบบ

**สถานการณ์**: ก่อนจะนำ ZBFW (Step 444 เป็นต้นไป) มาใช้งานจริง ทีม Network ต้องการโซลูชัน
ชั่วคราวที่ให้ **Campus (Inside) เริ่ม Connection ออกไป Internet ได้ตามปกติ และรับ Return
Traffic ของ Session นั้นกลับเข้ามาได้เท่านั้น** — Traffic ที่ Internet เริ่มเข้ามาเองโดยไม่มี
Session ก่อนหน้าให้ Deny ทั้งหมด (ยกเว้น Static NAT ของ Server1 ตาม
[Part 15](part-015-nat-pat.md)) — ทำบน Interface `GigabitEthernet0/0/0` (OUTSIDE ไปยัง
ISP-RTR) ของ WAN-EDGE-1

```
WAN-EDGE-1(config)# ip access-list extended OUTBOUND-REFLECT
WAN-EDGE-1(config-ext-nacl)# remark ==== Outbound campus traffic -> reflect for stateful return ====
WAN-EDGE-1(config-ext-nacl)# permit tcp 10.10.0.0 0.0.255.255 any reflect TCP-SESSIONS timeout 300
WAN-EDGE-1(config-ext-nacl)# permit udp 10.10.0.0 0.0.255.255 any reflect UDP-SESSIONS timeout 60
WAN-EDGE-1(config-ext-nacl)# permit icmp 10.10.0.0 0.0.255.255 any reflect ICMP-SESSIONS timeout 10
WAN-EDGE-1(config-ext-nacl)# exit

WAN-EDGE-1(config)# ip access-list extended INBOUND-REFLECT
WAN-EDGE-1(config-ext-nacl)# remark ==== Static NAT exception: Server1 HTTPS from Part 15 ====
WAN-EDGE-1(config-ext-nacl)# permit tcp any host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# remark ==== Evaluate reflexive sessions created by outbound ACL ====
WAN-EDGE-1(config-ext-nacl)# evaluate TCP-SESSIONS
WAN-EDGE-1(config-ext-nacl)# evaluate UDP-SESSIONS
WAN-EDGE-1(config-ext-nacl)# evaluate ICMP-SESSIONS
WAN-EDGE-1(config-ext-nacl)# remark ==== Deny all other unsolicited inbound traffic, log ====
WAN-EDGE-1(config-ext-nacl)# deny   ip any any log
WAN-EDGE-1(config-ext-nacl)# exit

WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# ip access-group OUTBOUND-REFLECT out
WAN-EDGE-1(config-if)# ip access-group INBOUND-REFLECT in
WAN-EDGE-1(config-if)# exit
```

**อธิบายลำดับการทำงาน**:

```
Campus Host (10.10.10.11) เปิด HTTPS ไป Internet Server (203.0.113.50:443)
     │
     ▼
[1] Packet ออกทาง Gi0/0/0 (out) → match OUTBOUND-REFLECT permit tcp ... reflect TCP-SESSIONS
     │
     ▼
[2] IOS สร้าง Temporary ACE อัตโนมัติใน TCP-SESSIONS:
     permit tcp host 203.0.113.50 eq 443 host <NAT-public-IP> eq <source-port> (สลับ src/dst)
     │
     ▼
[3] Return Traffic (SYN-ACK) จาก Internet Server กลับเข้า Gi0/0/0 (in)
     → match INBOUND-REFLECT evaluate TCP-SESSIONS → เจอ Temporary ACE ที่สร้างไว้ → PERMIT
     │
     ▼
[4] Session ปิด (FIN/RST) หรือไม่มี Traffic เกิน timeout (300 วินาที) → Temporary ACE ถูกลบอัตโนมัติ
```

### ตรวจสอบ Temporary Entry ที่ถูกสร้างขึ้นจริง

```
WAN-EDGE-1# show ip access-lists INBOUND-REFLECT
Reflexive IP access list TCP-SESSIONS
    permit tcp host 203.0.113.50 eq 443 host 203.0.113.2 eq 51022 (4 matches) (time left 287)

Extended IP access list INBOUND-REFLECT
    10 remark ==== Static NAT exception: Server1 HTTPS from Part 15 ====
    20 permit tcp any host 203.0.113.10 eq 443 (312 matches)
    30 remark ==== Evaluate reflexive sessions created by outbound ACL ====
    40 evaluate TCP-SESSIONS
    50 evaluate UDP-SESSIONS
    60 evaluate ICMP-SESSIONS
    70 remark ==== Deny all other unsolicited inbound traffic, log ====
    80 deny ip any any log (211 matches)
```

สังเกตว่า `show ip access-lists` แสดง **Reflexive IP access list TCP-SESSIONS** เป็น Block
แยกต่างหาก พร้อม `(time left <seconds>)` บอกเวลาที่เหลือก่อน Entry นี้จะถูกลบอัตโนมัติ — นี่คือ
สิ่งที่ทำให้ Reflexive ACL ต่างจาก ACL ปกติอย่างชัดเจน: **มันมี State ที่เปลี่ยนแปลงตามเวลาจริง**

### ข้อจำกัดของ Reflexive ACL (เหตุผลที่ยังไม่ใช่ Firewall เต็มรูปแบบ)

| ข้อจำกัด | รายละเอียด |
|---|---|
| ไม่มี Application-Layer Awareness | ไม่รู้จัก Protocol ระดับ Application (HTTP method, FTP data channel แยก port) |
| จัดการ Session แบบง่าย | ไม่ Track TCP State Machine เต็มรูปแบบ (SYN → SYN-ACK → ESTABLISHED → FIN) เหมือน Firewall จริง |
| ต้องคู่ ACL 2 ตัวเสมอ | บริหารจัดการยากขึ้นเมื่อมี Zone มากกว่า 2 ฝั่ง (Inside/Outside/DMZ) |
| ไม่มี Zone Concept | ไม่สามารถกำหนด Policy ตาม "โซนความปลอดภัย" ได้ ต้องคิดเป็น Interface + ACL ทีละคู่ |

> ข้อจำกัดเหล่านี้คือเหตุผลที่ Cisco พัฒนา **Zone-Based Firewall (ZBFW)** ขึ้นมาเป็นขั้นต่อไป —
> เริ่มตั้งแต่ Step 444

---

## Step 444 — IOS Zone-Based Firewall (ZBFW) พื้นฐาน

### จุดเปลี่ยนกระบวนทัศน์ (Paradigm Shift) จาก ACL สู่ ZBFW

นี่คือแนวคิดที่ **สำคัญที่สุดของ Part นี้** และเป็นสิ่งที่ทำให้วิศวกรที่คุ้นกับ ACL สับสนบ่อยที่สุด:

| แนวคิด | ACL (Part 14, Step 441-443) | Zone-Based Firewall (ZBFW) |
|---|---|---|
| Default Behavior | **Permit ทุกอย่าง** ยกเว้นที่ Deny ไว้ (Implicit Deny อยู่ท้ายสุด แต่ต้อง apply ACL ก่อนถึงมีผล) | **Deny ทุกอย่างระหว่าง Zone ที่ต่างกัน** โดยอัตโนมัติ ทันทีที่สร้าง Zone และใส่ Interface เข้า Zone แล้ว แม้ยังไม่มี Zone-Pair ใดๆ เลย |
| หน่วยพื้นฐานของ Policy | Interface + Direction (`in`/`out`) | **Zone** (กลุ่ม Interface ที่ Trust Level เดียวกัน) |
| การอนุญาต Traffic | ต้องเขียน `permit` ทีละบรรทัดครอบคลุมทุกกรณี | ต้องสร้าง **Zone-Pair** + **Policy-Map type inspect** อย่างชัดเจนระหว่าง Zone สองฝั่ง |
| State Awareness | Stateless (ยกเว้น `established`/Reflexive ที่กึ่ง Stateful) | **Stateful เต็มรูปแบบ** ผ่าน `inspect` action |

> **กฎเหล็กที่ต้องจำ**: ทันทีที่ Interface ถูกใส่เข้า Zone ด้วย `zone-member security` — Traffic
> ที่จะไปยัง Interface ใน **Zone อื่น** จะถูก **Deny โดยอัตโนมัติ (Default Deny Between Zones)**
> จนกว่าจะมี **Zone-Pair + Policy-Map** ที่อนุญาตไว้อย่างชัดเจน (ตรงข้ามกับ ACL ที่ Default คือ
> Permit จนกว่าจะเขียน Deny)

### องค์ประกอบหลักของ ZBFW

| องค์ประกอบ | คำสั่ง | ความหมาย |
|---|---|---|
| **Zone** | `zone security <name>` | กลุ่ม Interface ที่มี Trust Level เดียวกัน (เช่น INSIDE = Campus ทั้งหมด) |
| **Zone Membership** | `zone-member security <name>` (บน Interface) | กำหนดว่า Interface นี้เป็นสมาชิกของ Zone ใด |
| **Zone-Pair** | `zone-pair security <name> source <zoneA> destination <zoneB>` | กำหนดทิศทาง Traffic ที่จะตรวจสอบ (จาก Zone A ไป Zone B) |
| **Class-Map type inspect** | `class-map type inspect [match-any\|match-all] <name>` | จับคู่ Traffic ที่ต้องการ (ตาม Protocol หรือ ACL) |
| **Policy-Map type inspect** | `policy-map type inspect <name>` | กำหนด Action (`inspect`/`pass`/`drop`) สำหรับแต่ละ Class |
| **Self Zone** | `zone security self` (มีอยู่แล้วในตัว ไม่ต้องสร้าง) | หมายถึง Router เอง (Traffic ที่ปลายทาง/ต้นทางคือ Router ตัวมันเอง) |

### 3 Action หลักใน Policy-Map type inspect

| Action | ความหมาย | ใช้เมื่อ |
|---|---|---|
| `inspect` | **Stateful** — ตรวจสอบและจดจำ Session, อนุญาต Return Traffic อัตโนมัติ | Traffic ที่ต้องการให้ตอบกลับได้ (ส่วนใหญ่ของ Traffic ทั่วไป) |
| `pass` | **Stateless** — อนุญาตให้ผ่านตรงๆ โดยไม่จดจำ Session (ต้องเปิด Return Path เองถ้าต้องการ) | Traffic ทางเดียวที่ไม่มี Return หรือมี Zone-Pair อีกฝั่งจัดการ Return เอง (เช่น Routing Protocol บางกรณี) |
| `drop` (+ `log` ทางเลือก) | บล็อก Traffic ทั้งหมดใน Class นี้ | Traffic ที่ต้องการปฏิเสธชัดเจน |

### ภาพรวม Architecture ของ ZBFW บน WAN-EDGE-1

```
                       WAN-EDGE-1 (Zone-Based Firewall)
   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │    ZONE: INSIDE            ZONE: OUTSIDE           ZONE: SELF      │
   │   Gi0/0/1, Gi0/0/2          Gi0/0/0             (Router ตัวมันเอง)  │
   │  (Campus + cross-link)   (ไปยัง ISP-RTR)         (CPU/Control Plane)│
   │        │                       │                       │           │
   │        │  zone-pair             │                       │           │
   │        │  INSIDE→OUTSIDE        │                       │           │
   │        ├─────inspect (Stateful)─►                       │           │
   │        │◄────Return Traffic─────┤ (auto-permitted        │           │
   │        │                       │  by inspect state)     │           │
   │        │                       │                       │           │
   │        │  zone-pair             │                       │           │
   │        │  OUTSIDE→INSIDE (deny-all + Server1 exception)  │           │
   │        │◄──────drop (default)──┤                       │           │
   │        │                       │                       │           │
   │        │                       │  zone-pair             │           │
   │        │                       │  OUTSIDE→SELF          │           │
   │        │                       ├──limited pass──────────►           │
   │        │                       │  (management exception) │           │
   └──────────────────────────────────────────────────────────────────┘

  หมายเหตุ: Self Zone มีพฤติกรรม Default ต่างจาก Zone อื่น — ดูรายละเอียดใน Step 447
```

> **ทำไมต้องมี Self Zone แยก?** เพราะ Traffic ที่มุ่งไปยัง "Router เอง" (เช่น SSH เข้ามา config,
> SNMP Poll, NTP Sync) ไม่ได้ "ผ่าน (Transit)" Router แต่เป็น "ปลายทาง (Destination)" ของ Router
> เอง — ZBFW จึงต้องมี Zone พิเศษเพื่อแยกแยะ Traffic 2 ประเภทนี้ออกจากกันอย่างชัดเจน

---

## Step 445 — คอนฟิก ZBFW เต็มรูปแบบบน WAN-EDGE-1

### ลำดับขั้นตอนการคอนฟิก ZBFW (ต้องทำตามลำดับนี้เสมอ)

```
1. สร้าง Zone (zone security)
2. ใส่ Interface เข้า Zone (zone-member security) -- ผลข้างเคียง: Traffic ข้าม Zone ถูก Deny ทันที
3. สร้าง Class-Map type inspect (จับคู่ Traffic ที่ต้องการ)
4. สร้าง Policy-Map type inspect (กำหนด Action ต่อ Class)
5. สร้าง Zone-Pair (กำหนดทิศทาง source -> destination)
6. Apply Policy-Map เข้า Zone-Pair (service-policy type inspect)
```

### ขั้นที่ 1-2: สร้าง Zone และใส่ Interface เข้า Zone

WAN-EDGE-1 มี Interface ตาม [IP Address Plan](00-ip-address-plan.md) และ Part 15/19: `Gi0/0/0`
(ไปยัง ISP-RTR, Outside), `Gi0/0/1` (ไปยัง CORE-SW1, Inside), `Gi0/0/2` (Cross-link ไปยัง
WAN-EDGE-2, ยังเป็น Inside เพราะอยู่ในองค์กร) — เพิ่ม `Serial0/0/0` (ไปยัง BRANCH-RTR ตาม
[Part 19](part-019-wan-technologies.md)) เข้าเป็น Inside ด้วย เพราะเป็น Private WAN Link
ภายในองค์กร ไม่ใช่ Internet

```
WAN-EDGE-1(config)# zone security INSIDE
WAN-EDGE-1(config-sec-zone)# description Campus network + private WAN links (trusted)
WAN-EDGE-1(config-sec-zone)# exit

WAN-EDGE-1(config)# zone security OUTSIDE
WAN-EDGE-1(config-sec-zone)# description Internet-facing towards ISP-RTR (untrusted)
WAN-EDGE-1(config-sec-zone)# exit

WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# zone-member security INSIDE
WAN-EDGE-1(config-if)# exit

WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# zone-member security INSIDE
WAN-EDGE-1(config-if)# exit

WAN-EDGE-1(config)# interface Serial0/0/0
WAN-EDGE-1(config-if)# zone-member security INSIDE
WAN-EDGE-1(config-if)# exit

WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# zone-member security OUTSIDE
WAN-EDGE-1(config-if)# exit
```

> **ผลข้างเคียงทันทีหลังคำสั่งชุดนี้**: ทันทีที่ `Gi0/0/0` เข้า Zone `OUTSIDE` และ `Gi0/0/1`/`Gi0/0/2`
> เข้า Zone `INSIDE` — Traffic ทั้งหมดระหว่าง Zone ทั้งสอง **จะถูก Drop โดยอัตโนมัติ** แม้จะยังไม่มี
> Zone-Pair ใดๆ เลยก็ตาม (ตรงข้ามกับตอนที่ยังไม่มี Zone — Router ทำงานแบบ Routing ปกติ) —
> นี่คือสาเหตุที่ **ต้องวางแผน Policy ให้พร้อมก่อน** จะ apply `zone-member` จริงใน Production
> (คล้ายกับความเสี่ยง Pitfall เรื่องล็อกตัวเองออกใน [Part 14 Step 138](part-014-access-control-lists.md)
> — ควรใช้ `reload in 10` เป็น Safety Net เช่นเดียวกัน)

### ขั้นที่ 3-4: Class-Map และ Policy-Map สำหรับ INSIDE → OUTSIDE (Campus ออก Internet)

```
WAN-EDGE-1(config)# class-map type inspect match-any INSIDE-TO-OUTSIDE-CLASS
WAN-EDGE-1(config-cmap)# match protocol tcp
WAN-EDGE-1(config-cmap)# match protocol udp
WAN-EDGE-1(config-cmap)# match protocol icmp
WAN-EDGE-1(config-cmap)# exit

WAN-EDGE-1(config)# policy-map type inspect INSIDE-TO-OUTSIDE-POLICY
WAN-EDGE-1(config-pmap)# class type inspect INSIDE-TO-OUTSIDE-CLASS
WAN-EDGE-1(config-pmap-c)#  inspect
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  drop log
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit
```

`class-default` คือ Class ที่ IOS สร้างให้อัตโนมัติเสมอ (ครอบคลุม Traffic ที่ไม่ Match Class
ใดๆ ที่กำหนดไว้ — คล้าย Implicit Deny ของ ACL แต่ในที่นี้เรา **เขียน `drop log` อย่างชัดเจน** เพื่อ
ให้เห็น Log และเจตนาชัดเจนในการอ่าน Config)

### ขั้นที่ 3-4 (ต่อ): Class-Map และ Policy-Map สำหรับ OUTSIDE → INSIDE (พร้อม Exception Server1)

```
WAN-EDGE-1(config)# ip access-list extended SERVER1-HTTPS-EXCEPTION
WAN-EDGE-1(config-ext-nacl)# remark ==== Match traffic to Server1 static-NAT public address, HTTPS only ====
WAN-EDGE-1(config-ext-nacl)# permit tcp any host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# exit

WAN-EDGE-1(config)# class-map type inspect match-all OUTSIDE-TO-INSIDE-EXCEPTION-CLASS
WAN-EDGE-1(config-cmap)# match access-group name SERVER1-HTTPS-EXCEPTION
WAN-EDGE-1(config-cmap)# exit

WAN-EDGE-1(config)# policy-map type inspect OUTSIDE-TO-INSIDE-POLICY
WAN-EDGE-1(config-pmap)# class type inspect OUTSIDE-TO-INSIDE-EXCEPTION-CLASS
WAN-EDGE-1(config-pmap-c)#  inspect
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  drop log
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit
```

> **หมายเหตุเรื่อง NAT Order of Operations**: `class-map` ที่ Match ด้วย ACL ตัวนี้ต้องอ้างถึง
> **Public IP (`203.0.113.10`)** ไม่ใช่ **Private IP (`10.10.30.10`)** เพราะ ZBFW ตรวจสอบ
> Traffic ที่ Interface Outside **ก่อน** ที่ NAT จะแปลง Destination กลับเป็น Inside Local —
> หลักการเดียวกับที่อธิบายไว้ใน [Part 15 Step 147](part-015-nat-pat.md) สำหรับ ACL ทั่วไป
> (`OUTSIDE-IN-ACL`) ที่ Match Public IP เช่นกัน

### ขั้นที่ 5-6: สร้าง Zone-Pair และ Apply Policy-Map

```
WAN-EDGE-1(config)# zone-pair security INSIDE-TO-OUTSIDE source INSIDE destination OUTSIDE
WAN-EDGE-1(config-sec-zone-pair)# service-policy type inspect INSIDE-TO-OUTSIDE-POLICY
WAN-EDGE-1(config-sec-zone-pair)# exit

WAN-EDGE-1(config)# zone-pair security OUTSIDE-TO-INSIDE source OUTSIDE destination INSIDE
WAN-EDGE-1(config-sec-zone-pair)# service-policy type inspect OUTSIDE-TO-INSIDE-POLICY
WAN-EDGE-1(config-sec-zone-pair)# exit
```

> สังเกตว่าเราสร้าง **Zone-Pair 2 ทิศทาง** (`INSIDE→OUTSIDE` และ `OUTSIDE→INSIDE`) แยกกันโดย
> สิ้นเชิง แม้จะเป็น Zone คู่เดียวกันก็ตาม — ZBFW ถือว่า **ทิศทางคือส่วนหนึ่งของ Identity ของ
> Zone-Pair** ไม่เหมือน ACL ที่ทิศทาง `in`/`out` ผูกกับ Interface เดี่ยวๆ

### ตรวจสอบ Config ทั้งหมดที่เกี่ยวข้อง

```
WAN-EDGE-1# show running-config | section zone
zone security INSIDE
 description Campus network + private WAN links (trusted)
zone security OUTSIDE
 description Internet-facing towards ISP-RTR (untrusted)
zone-pair security INSIDE-TO-OUTSIDE source INSIDE destination OUTSIDE
 service-policy type inspect INSIDE-TO-OUTSIDE-POLICY
zone-pair security OUTSIDE-TO-INSIDE source OUTSIDE destination INSIDE
 service-policy type inspect OUTSIDE-TO-INSIDE-POLICY
```

**ผลลัพธ์ที่คาดหวัง**: Campus (INSIDE) ออก Internet ได้ตามปกติ (TCP/UDP/ICMP ผ่าน `inspect`)
พร้อม Return Traffic กลับเข้ามาได้อัตโนมัติ, ส่วน Traffic ที่ Internet เริ่มเข้ามาเองจะถูก Drop
ทั้งหมด ยกเว้น HTTPS ไปยัง Server1 ที่อนุญาตไว้เป็น Exception

---

## Step 446 — Stateful Inspection: `inspect` Action ทำงานอย่างไร

### ความแตกต่างจาก Reflexive ACL (Step 443)

`inspect` action ทำงาน**คล้าย** Reflexive ACL ในหลักการพื้นฐาน (สร้าง State ให้ Return Traffic
กลับเข้ามาได้อัตโนมัติ) แต่ **ลึกและสมบูรณ์กว่ามาก**:

| คุณสมบัติ | Reflexive ACL | ZBFW `inspect` |
|---|---|---|
| ติดตาม TCP State Machine | ไม่ (ดูแค่ Source/Dest/Port ตรงกันหรือไม่) | **ใช่** — ติดตามทั้ง SYN → SYN-ACK → ESTABLISHED → FIN/RST ตาม RFC 793 จริง |
| ตรวจจับ Session ที่ผิดปกติ | ไม่ | ใช่ — Half-open connection มากเกินไป (SYN Flood pattern) ถูกจับได้ |
| Application-Layer Awareness | ไม่ | ใช่ — เข้าใจ Protocol ระดับ Application (ดูหัวข้อถัดไป) |
| จำนวน Zone ที่จัดการได้ | ต้องคู่ ACL ทีละ Interface | ไม่จำกัด — ขยาย Zone เพิ่มได้ (เช่น DMZ) โดยไม่กระทบ Zone-Pair เดิม |

### กลไกการทำงานของ `inspect`

```
Session TCP ใหม่เริ่มจาก INSIDE ไป OUTSIDE
     │
     ▼
[1] SYN ออกจาก Gi0/0/1 -> Gi0/0/0 -> zone-pair INSIDE-TO-OUTSIDE, class match tcp -> inspect
     │  IOS สร้าง Session Entry ใน State Table: {src, dst, sport, dport, state=SYN_SENT}
     ▼
[2] SYN-ACK ตอบกลับจาก OUTSIDE -> INSIDE
     │  ZBFW ตรวจสอบ zone-pair OUTSIDE-TO-INSIDE ก่อน -- แต่เพราะมี Session Entry ที่ inspect
     │  สร้างไว้แล้ว (Stateful Match) -> PERMIT ทันที โดยไม่ต้องพึ่ง Policy ของ zone-pair
     │  OUTSIDE-TO-INSIDE เลย (Session Entry มีสิทธิ์เหนือกว่า Static Policy)
     │  State เปลี่ยนเป็น SYN_RECEIVED
     ▼
[3] ACK จาก INSIDE -> OUTSIDE -> State เปลี่ยนเป็น ESTABLISHED
     ▼
[4] Data Traffic ไหลทั้งสองทิศทางได้ตามปกติตลอด Session
     ▼
[5] FIN/RST หรือ Idle Timeout -> Session Entry ถูกลบออกจาก State Table
```

> **ข้อสำคัญที่สุด**: Return Traffic (ขั้นที่ 2) **ไม่ต้องพึ่งพา** Zone-Pair `OUTSIDE-TO-INSIDE`
> ที่เรา Config เป็น `drop` ไว้เลย — เพราะ **Session Entry ที่ถูกสร้างจาก `inspect`** มีความสำคัญ
> เหนือกว่า Static Policy ของ Zone-Pair ปกติ นี่คือสิ่งที่ทำให้ ZBFW เป็น Firewall **Stateful
> เต็มรูปแบบ** — ตรงกับพฤติกรรมของ Firewall จริงในโลกการทำงาน (ASA, FTD, Palo Alto, Fortinet)

### Application-Layer Inspection (Layer 7 Awareness)

`class-map type inspect` ไม่ได้จำกัดแค่ `match protocol tcp/udp/icmp` (Layer 3/4) เท่านั้น —
ยังรองรับ `match protocol <application>` สำหรับ Protocol ระดับ Application ที่ IOS "เข้าใจ"
โครงสร้างภายในของมันด้วย

| Protocol | `match protocol` Keyword | ประโยชน์ของ Application-Layer Inspection |
|---|---|---|
| HTTP | `http` | ตรวจจับ Traffic ที่ปลอมเป็น HTTP บน Port อื่น (Protocol Anomaly Detection) |
| FTP | `ftp` | เปิด Data Channel (Port สุ่ม) อัตโนมัติให้ตรงกับ Control Channel (Port 21) — ไม่ต้องเปิด Port Range กว้างๆ ล่วงหน้า |
| DNS | `dns` | ตรวจสอบ Query/Response ให้ตรงคู่กัน |
| ESMTP | `smtp` (esmtp) | ตรวจสอบโครงสร้าง Mail Command ให้ถูกต้องตาม Protocol |
| SIP | `sip` | รองรับ VoIP Signaling ที่มีการเปิด RTP Media Channel แบบ Dynamic Port (คล้ายกับ Requirement VOICE VLAN ใน [Part 14](part-014-access-control-lists.md)) |

**ตัวอย่าง Class-Map ที่ใช้ Application-Layer Inspection**:

```
WAN-EDGE-1(config)# class-map type inspect match-any APPLICATION-AWARE-CLASS
WAN-EDGE-1(config-cmap)# match protocol http
WAN-EDGE-1(config-cmap)# match protocol ftp
WAN-EDGE-1(config-cmap)# match protocol dns
WAN-EDGE-1(config-cmap)# exit
```

> **ประโยชน์เชิง Security ที่ ACL ทำไม่ได้เลย**: ACL ธรรมดา Match แค่ Port Number — ถ้ามี
> Malware สื่อสารผ่าน Port 443 แต่ไม่ได้เป็น HTTPS จริง (ปลอม Port) ACL จะปล่อยผ่านโดยไม่รู้ตัว
> แต่ Application-Layer Inspection ของ ZBFW จะตรวจ **โครงสร้าง Protocol จริง** และสามารถ Drop
> Traffic ที่ผิดปกติได้ แม้จะมาบน Port ที่ "ดูเหมือน" ถูกต้องก็ตาม (รายละเอียดเชิงลึกระดับ
> Application Firewall/IPS แบบสมบูรณ์จะอยู่ในหลักสูตร Security เฉพาะทางถัดไป)

---

## Step 447 — ปกป้อง Self-Zone: Management Plane Security

### พฤติกรรม Default ของ Self-Zone ที่ต่างจาก Zone อื่นโดยสิ้นเชิง

นี่คือรายละเอียดที่ CCNP ENCOR สอบถามบ่อยและสร้างความสับสนมากที่สุด:

> **กฎพิเศษของ Self-Zone**: Traffic ที่มีต้นทางหรือปลายทางเป็น **Self Zone (Router เอง)** จะถูก
> **PERMIT โดยอัตโนมัติเป็น Default** — **ตรงข้าม** กับกฎ Default-Deny-Between-Zones ของ Zone
> ทั่วไป (Step 444) เหตุผลคือ Router ต้องสามารถทำงานพื้นฐานได้ (Routing Protocol, NTP Sync,
> Ping ตัวเอง) แม้จะยังไม่ได้ Config Zone-Pair ใดๆ เกี่ยวกับ Self-Zone เลย — **แต่ทันทีที่มีการ
> สร้าง Zone-Pair ที่เกี่ยวข้องกับ Self-Zone อย่างชัดเจน (เช่น `source OUTSIDE destination self`)
> Behavior จะเปลี่ยนไปใช้ Policy ที่ Config ไว้ทันที** (Class ที่ไม่ Match จะตกไปที่ `class-default`
> ตาม Policy-Map นั้น)

### ทำไมต้องปกป้อง Self-Zone

Management Plane ของ Router (SSH, SNMP, NTP) ที่ [Part 22](part-022-aaa-device-hardening.md)
ปกป้องด้วย `access-class` บน VTY Line — เป็นการปกป้อง **เฉพาะ VTY Line** เท่านั้น แต่ Traffic
ที่มุ่งไปยัง Self-Zone ยังรวมถึง Protocol อื่นที่ **ไม่ผ่าน VTY** เลย เช่น SNMP (UDP 161), NTP
(UDP 123), หรือแม้แต่ ICMP Echo Request ไปยัง IP ของ Router เอง — `access-class` **ไม่ครอบคลุม
Traffic เหล่านี้** เพราะไม่ได้เข้าทาง VTY Line

ZBFW Self-Zone จึงเป็น**ชั้นป้องกันที่ครอบคลุมกว่า** เพราะทำงานที่ระดับ Zone (ครอบคลุมทุก
Protocol ที่ปลายทางเป็น Router) ไม่ใช่แค่ Line-level แบบ `access-class`

### Worked Example: จำกัด Traffic จาก OUTSIDE เข้า Self-Zone

**Requirement**: อนุญาตให้ ISP/Internet ping Router (`Gi0/0/0`, `203.0.113.2`) ได้เพื่อการ
Monitoring พื้นฐาน แต่**ห้าม SSH, SNMP, หรือ Protocol Management อื่นเข้าจาก OUTSIDE เด็ดขาด**
(Management ที่แท้จริงต้องมาจาก MGMT VLAN ผ่าน INSIDE Zone เท่านั้น ตามหลักการ
[Part 22](part-022-aaa-device-hardening.md))

```
WAN-EDGE-1(config)# ip access-list extended OUTSIDE-TO-SELF-ALLOWED
WAN-EDGE-1(config-ext-nacl)# remark ==== Allow only ICMP monitoring from Internet to router itself ====
WAN-EDGE-1(config-ext-nacl)# permit icmp any host 203.0.113.2 echo
WAN-EDGE-1(config-ext-nacl)# permit icmp any host 203.0.113.2 echo-reply
WAN-EDGE-1(config-ext-nacl)# permit icmp any host 203.0.113.2 time-exceeded
WAN-EDGE-1(config-ext-nacl)# exit

WAN-EDGE-1(config)# class-map type inspect match-all OUTSIDE-TO-SELF-CLASS
WAN-EDGE-1(config-cmap)# match access-group name OUTSIDE-TO-SELF-ALLOWED
WAN-EDGE-1(config-cmap)# exit

WAN-EDGE-1(config)# policy-map type inspect OUTSIDE-TO-SELF-POLICY
WAN-EDGE-1(config-pmap)# class type inspect OUTSIDE-TO-SELF-CLASS
WAN-EDGE-1(config-pmap-c)#  pass
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  drop log
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit

WAN-EDGE-1(config)# zone-pair security OUTSIDE-TO-SELF source OUTSIDE destination self
WAN-EDGE-1(config-sec-zone-pair)# service-policy type inspect OUTSIDE-TO-SELF-POLICY
WAN-EDGE-1(config-sec-zone-pair)# exit
```

> **สังเกตการใช้ `pass` แทน `inspect`**: Traffic ICMP ที่นี่ใช้ `pass` (Stateless) เพราะ ICMP
> Echo Request/Reply เป็นการสื่อสารแบบ Request-Response สั้นๆ ไม่มี Session ยาวนานที่ต้อง
> Track State — ใช้ `inspect` ก็ได้เช่นกันแต่ `pass` เพียงพอและประหยัด Resource กว่าในกรณีนี้

### เปรียบเทียบ Self-Zone vs Part 22's Management-Plane ACL

| มุมมอง | `access-class` บน VTY ([Part 22](part-022-aaa-device-hardening.md)) | Self-Zone ZBFW |
|---|---|---|
| ครอบคลุม Protocol | เฉพาะที่เข้าทาง VTY Line (SSH/Telnet) | **ทุก Protocol** ที่ปลายทาง/ต้นทางคือ Router (SSH, SNMP, NTP, ICMP, Routing Protocol) |
| Layer ที่ทำงาน | Line-level (ผูกกับ VTY Line โดยเฉพาะ) | Zone-level (ผูกกับ Interface ที่เป็นสมาชิก Zone) |
| Stateful | ไม่ (Stateless ACL ปกติ) | ได้ (ถ้าใช้ Action `inspect` แทน `pass`) |
| ความซับซ้อนในการ Config | ง่าย (2-3 บรรทัด) | ซับซ้อนกว่า (ต้อง Class-Map + Policy-Map + Zone-Pair) |
| ใช้ทดแทนกันได้หรือไม่ | **ใช้ควบคู่กัน (Defense-in-Depth)** — ไม่ใช่ Feature ที่แข่งกัน แต่ป้องกันคนละมุมของ Management Plane เดียวกัน |

> **Best Practice**: ใช้ทั้งสองชั้นพร้อมกันเสมอ — `access-class` ป้องกัน VTY โดยเฉพาะ (เร็ว,
> ง่าย, เบา) และ Self-Zone ของ ZBFW ป้องกัน Protocol Management อื่นที่ VTY ไม่ครอบคลุม — ตรงตาม
> หลักการ **Security-in-Depth** ที่เรียนมาตั้งแต่ [Part 22](part-022-aaa-device-hardening.md)
> (Management Plane / Control Plane / Data Plane) และจะสมบูรณ์ยิ่งขึ้นเมื่อเพิ่ม **Control Plane
> Policing (CoPP)** ใน [Part 46](part-046-control-plane-policing.md)

---

## Step 448 — เปรียบเทียบ ZBFW vs Reflexive ACL vs Plain ACL

### ตารางเปรียบเทียบเต็มรูปแบบ

| มุมมอง | Plain Extended ACL | Reflexive ACL | Zone-Based Firewall (ZBFW) |
|---|---|---|---|
| **Statefulness** | Stateless (ยกเว้น `established` ที่กึ่ง Stateful เฉพาะ TCP) | กึ่ง Stateful (จำ Session ตาม Src/Dst/Port แต่ไม่ Track TCP State Machine เต็มรูปแบบ) | **Stateful เต็มรูปแบบ** (Track TCP State Machine, Session Table จริง) |
| **รองรับ Protocol** | ทุก Protocol แต่ `established` ใช้ได้แค่ TCP | TCP, UDP, ICMP | TCP, UDP, ICMP + Application-Layer (HTTP, FTP, DNS, SIP, ฯลฯ) |
| **ความซับซ้อนในการ Config** | ต่ำที่สุด (บรรทัดเดียวต่อ Policy) | กลาง (ต้องคู่ ACL 2 ตัวเสมอ, จัดการยากขึ้นเมื่อมีหลาย Zone) | สูงที่สุด (ต้อง Zone + Class-Map + Policy-Map + Zone-Pair) แต่ **Scale ได้ดีที่สุด** เมื่อ Zone เพิ่มขึ้น (DMZ, Guest, ฯลฯ) |
| **Default Behavior** | Permit จนกว่าจะ Deny (ต้องคิดครบทุกกรณีเอง) | เหมือน ACL (Permit จนกว่าจะ Deny) แต่มี State เสริม | **Deny ทุกอย่างระหว่าง Zone โดยอัตโนมัติ** (ปลอดภัยกว่าโดย Design — Fail-Safe) |
| **Application-Layer Awareness** | ไม่มี (ดูแค่ Port Number) | ไม่มี | มี (ตรวจ Protocol Anomaly, เปิด Dynamic Port ให้ FTP/SIP อัตโนมัติ) |
| **Performance Overhead** | น้อยที่สุด (ไม่มี State ต้องจัดการ) | ปานกลาง (ต้องจัดการ Temporary ACE + Timeout) | มากที่สุด (ต้องจัดการ Session Table + Inspection Engine) แต่ Hardware สมัยใหม่ (ISR4000, ASR) รองรับได้ดีด้วย Dedicated Processing |
| **เหมาะกับงาน** | Traffic Filtering พื้นฐาน, Route-map/NAT Matching, VTY Access-Class | Router ขนาดเล็ก/กลางที่ต้องการ Stateful-like โดยไม่ลงทุน Firewall เพิ่ม | Enterprise Edge/Branch ที่ต้องการ Firewall เต็มรูปแบบในตัว Router โดยไม่ต้องซื้อ Firewall แยก (ASA/FTD) |

### แนวทางการเลือกใช้ (Decision Guide)

```
ต้องการ Filter Traffic ง่ายๆ ระหว่าง VLAN, จำกัด VTY, หรือ Match ให้ Route-map/NAT?
     │
     ▼ ใช่
   Plain Extended/Standard ACL (Part 14) เพียงพอ — ง่ายที่สุด เร็วที่สุด
     │
     ▼ ไม่ใช่ (ต้องการ Stateful Return Traffic โดยไม่มี Firewall เต็มรูปแบบ)
   Reflexive ACL — เหมาะกับ Router ขนาดเล็กที่ยังไม่มี ZBFW License/Resource เพียงพอ
     │
     ▼ ต้องการ Firewall เต็มรูปแบบ, หลาย Zone (Inside/Outside/DMZ), Application-Layer Inspection?
   Zone-Based Firewall (ZBFW) — มาตรฐานของ Enterprise Edge Router ยุคปัจจุบัน
```

> **ข้อคิดสำคัญสำหรับ CCNP ENCOR**: ข้อสอบมักถาม **Default Behavior ที่ต่างกัน** ระหว่าง ACL
> (Permit-by-default) กับ ZBFW (Deny-by-default ระหว่าง Zone) เป็นจุดที่ Distractor (ตัวเลือกหลอก)
> มักใช้สร้างความสับสน — จำให้แน่นว่า **แค่สร้าง Zone และใส่ Interface เข้า Zone ก็เพียงพอที่จะ
> Block Traffic ข้าม Zone ได้แล้ว** แม้ยังไม่มี Zone-Pair เลยก็ตาม

---

## Step 449 — Verification/Troubleshooting ZBFW และ Reflexive ACL

### คำสั่ง Verify หลักของ ZBFW

```
WAN-EDGE-1# show zone security
zone self
  Description: System defined zone

zone INSIDE
  Description: Campus network + private WAN links (trusted)
  Member Interfaces:
    GigabitEthernet0/0/1
    GigabitEthernet0/0/2
    Serial0/0/0

zone OUTSIDE
  Description: Internet-facing towards ISP-RTR (untrusted)
  Member Interfaces:
    GigabitEthernet0/0/0
```

```
WAN-EDGE-1# show zone-pair security
Zone-pair name INSIDE-TO-OUTSIDE
  Source-Zone INSIDE  Destination-Zone OUTSIDE
  service-policy INSIDE-TO-OUTSIDE-POLICY

Zone-pair name OUTSIDE-TO-INSIDE
  Source-Zone OUTSIDE  Destination-Zone INSIDE
  service-policy OUTSIDE-TO-INSIDE-POLICY

Zone-pair name OUTSIDE-TO-SELF
  Source-Zone OUTSIDE  Destination-Zone self
  service-policy OUTSIDE-TO-SELF-POLICY
```

```
WAN-EDGE-1# show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions
Zone-pair: INSIDE-TO-OUTSIDE

  Service-policy inspect : INSIDE-TO-OUTSIDE-POLICY

    Class-map: INSIDE-TO-OUTSIDE-CLASS (match-any)
      Match: protocol tcp
      Match: protocol udp
      Match: protocol icmp
      Inspect
        Established Sessions
         Session 6BC91A20 (10.10.10.11:52210)=>(203.0.113.50:443) tcp SIS_OPEN
          Created 00:02:14, Last heard 00:00:02
          Bytes sent (initiator:responder) [4520:98204]

    Class-map: class-default (match-any)
      Match: any
      Drop (log)
        20 packets, 1240 bytes
```

`show policy-map type inspect zone-pair <name> sessions` คือคำสั่งที่**สำคัญที่สุด**สำหรับ
Verify ว่า Session จริงถูก Track อยู่หรือไม่ — เห็น Source/Destination IP:Port, State (`SIS_OPEN`),
เวลาที่สร้าง และปริมาณ Byte ที่ผ่านไปแล้ว

```
WAN-EDGE-1# show class-map type inspect
 Class Map type inspect match-any INSIDE-TO-OUTSIDE-CLASS (id 1)
   Match protocol tcp
   Match protocol udp
   Match protocol icmp

 Class Map type inspect match-all OUTSIDE-TO-INSIDE-EXCEPTION-CLASS (id 2)
   Match access-group name SERVER1-HTTPS-EXCEPTION
```

### Reflexive ACL: `show ip access-lists` แสดง Temporary Entry

ตามที่กล่าวไว้ใน Step 443 — Reflexive Entry จะปรากฏใน `show ip access-lists` เป็น Block
`Reflexive IP access list <name>` แยกจาก Extended ACL ปกติ พร้อม `(time left <วินาที>)`
เป็นหลักฐานว่า Session นั้นยัง Active อยู่จริง (ถ้าไม่มี Session ผ่าน Block นี้จะไม่ปรากฏเลย)

### สถานการณ์ Troubleshoot: Zone-Pair ที่ Drop Traffic แบบไม่มีสาเหตุชัดเจน

**อาการ**: Config Zone-Pair + Policy-Map ดูถูกต้องครบทุกอย่าง แต่ Traffic ที่ควร Pass ยังถูก
Drop อยู่ (ไม่มี Error ใดๆ ตอน Config)

**ลำดับการ Troubleshoot ที่ถูกต้อง (เรียงจากสาเหตุที่พบบ่อยที่สุดไปหาน้อยที่สุด)**:

```
1. ตรวจสอบ Zone-Membership ของ Interface ก่อนเสมอ
   WAN-EDGE-1# show zone security
   -> Interface ที่คาดว่าอยู่ใน Zone นั้น ปรากฏใน "Member Interfaces" จริงหรือไม่?
      (สาเหตุที่พบบ่อยที่สุด: ลืม zone-member security หรือใส่ผิด Zone)

2. ตรวจสอบว่า Interface ทั้งสองฝั่งอยู่ใน Zone ที่ถูกต้องตามที่ Zone-Pair คาดหวัง
   -> ถ้า Interface ไม่ได้เป็นสมาชิกของ Zone ใดเลย (Unzoned) แต่ Router มี Zone อื่นอยู่แล้ว
      Traffic ระหว่าง Interface ที่ Unzoned กับ Interface ที่อยู่ใน Zone จะถูก Drop เช่นกัน
      (พฤติกรรม Default-Deny ใช้ครอบคลุมทั้ง "ระหว่าง Zone" และ "Zone กับ Unzoned Interface")

3. ตรวจสอบทิศทาง Zone-Pair (source/destination) ตรงกับทิศทาง Traffic จริงหรือไม่
   WAN-EDGE-1# show zone-pair security
   -> Traffic ไหลจาก OUTSIDE ไป INSIDE แต่ Zone-Pair ที่ Config ไว้มีแค่ INSIDE->OUTSIDE ทิศทางเดียว?

4. ตรวจสอบ Class-Map จริงๆ Match Traffic ตามที่ตั้งใจหรือไม่
   WAN-EDGE-1# show policy-map type inspect zone-pair <name> sessions
   -> ดูว่า Traffic ตกไปที่ class-default (Drop) แทน Class ที่ตั้งใจให้ Match หรือไม่
      (สาเหตุที่พบบ่อย: match access-group อ้าง ACL ผิดชื่อ, หรือ Match Private IP แทน Public
      IP ในกรณีที่มี NAT ทำงานร่วมด้วย — ดู Step 445)

5. ถ้าเป็น Static NAT Exception ตรวจสอบ Order of Operations ระหว่าง ZBFW กับ NAT
   -> ACL ที่ใช้ใน Class-Map ต้อง Match ที่ Global Address (Public IP) เสมอสำหรับ Traffic
      ทิศทางเข้า (Inbound) เพราะ ZBFW ตรวจสอบก่อน NAT แปลง Address
```

> **บทเรียนสำคัญ**: **ตรวจ Zone-Membership ก่อนเสมอ** ก่อนจะไปดู Class-Map/Policy-Map ที่ซับซ้อน
> กว่า — ประสบการณ์จริงในสนามพบว่ากรณี "ลืม `zone-member security`" หรือ "ใส่ Interface ผิด Zone"
> คือสาเหตุอันดับ 1 ของปัญหา ZBFW ที่ดู "ไม่มีเหตุผล" ทั้งที่ Policy ถูกต้องสมบูรณ์แบบ

### สรุปคำสั่ง Verify/Troubleshoot ZBFW ทั้งหมด

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show zone security` | Zone ทั้งหมดและ Interface สมาชิกของแต่ละ Zone |
| `show zone-pair security` | Zone-Pair ทั้งหมด, ทิศทาง, และ Policy-Map ที่ Apply |
| `show policy-map type inspect zone-pair <name> sessions` | Session จริงที่กำลัง Active ใน Zone-Pair นั้น พร้อม Byte Counter |
| `show class-map type inspect` | รายการ Class-Map ทั้งหมดและเงื่อนไข Match |
| `show ip access-lists` | ACL ปกติ + Reflexive Entry (Temporary ACE พร้อม Timeout ที่เหลือ) |
| `show running-config \| section zone` | Config ของ Zone/Zone-Pair ทั้งหมดใน Running-Config |

---

## Step 450 — Lab ใหญ่: Deploy ZBFW เต็มรูปแบบบน WAN-EDGE-1

### Requirement เต็มรูปแบบของ Lab นี้

| # | Requirement |
|---|---|
| 1 | สร้าง 3 Zone: **INSIDE** (Campus + Private WAN), **OUTSIDE** (ISP-RTR), และ **SELF** (Router เอง — มีอยู่แล้วในตัว) |
| 2 | Campus (INSIDE) ออก Internet (OUTSIDE) ได้ตามปกติทุก Protocol หลัก (TCP/UDP/ICMP) พร้อม Stateful Return Traffic |
| 3 | Internet (OUTSIDE) เข้า Campus (INSIDE) โดยไม่ได้รับอนุญาต ต้องถูก Block ทั้งหมด **ยกเว้น** HTTPS ไปยัง Server1 ผ่าน Static NAT (`203.0.113.10:443`) ตาม [Part 15](part-015-nat-pat.md) |
| 4 | Internet (OUTSIDE) เข้า Router เอง (SELF) ได้เฉพาะ ICMP สำหรับ Monitoring เท่านั้น — SSH/SNMP/Management อื่นห้ามเข้าจาก OUTSIDE เด็ดขาด |

### Full Running-Config บน WAN-EDGE-1 (รวมทุกส่วนที่เกี่ยวข้อง)

```
hostname WAN-EDGE-1
!
interface Loopback0
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** OUTSIDE ** to ISP-RTR
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 zone-member security OUTSIDE
 negotiation auto
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** INSIDE ** to CORE-SW1
 ip address 10.10.254.1 255.255.255.252
 ip nat inside
 zone-member security INSIDE
 negotiation auto
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** INSIDE ** cross-link to WAN-EDGE-2
 ip address 10.10.254.9 255.255.255.252
 ip nat inside
 zone-member security INSIDE
 negotiation auto
 no shutdown
!
interface Serial0/0/0
 description ** INSIDE ** Leased Line to BRANCH-RTR
 ip address 172.16.199.1 255.255.255.252
 encapsulation ppp
 ip nat inside
 zone-member security INSIDE
 no shutdown
!
! ==================== ZBFW: Zone Definitions ====================
zone security INSIDE
 description Campus network + private WAN links (trusted)
zone security OUTSIDE
 description Internet-facing towards ISP-RTR (untrusted)
!
! ==================== ZBFW: ACL used as Class-Map matcher ====================
ip access-list extended SERVER1-HTTPS-EXCEPTION
 remark ==== Match traffic to Server1 static-NAT public address, HTTPS only ====
 permit tcp any host 203.0.113.10 eq 443
!
ip access-list extended OUTSIDE-TO-SELF-ALLOWED
 remark ==== Allow only ICMP monitoring from Internet to router itself ====
 permit icmp any host 203.0.113.2 echo
 permit icmp any host 203.0.113.2 echo-reply
 permit icmp any host 203.0.113.2 time-exceeded
!
! ==================== ZBFW: Class-Maps ====================
class-map type inspect match-any INSIDE-TO-OUTSIDE-CLASS
 match protocol tcp
 match protocol udp
 match protocol icmp
!
class-map type inspect match-all OUTSIDE-TO-INSIDE-EXCEPTION-CLASS
 match access-group name SERVER1-HTTPS-EXCEPTION
!
class-map type inspect match-all OUTSIDE-TO-SELF-CLASS
 match access-group name OUTSIDE-TO-SELF-ALLOWED
!
! ==================== ZBFW: Policy-Maps ====================
policy-map type inspect INSIDE-TO-OUTSIDE-POLICY
 class type inspect INSIDE-TO-OUTSIDE-CLASS
  inspect
 class class-default
  drop log
!
policy-map type inspect OUTSIDE-TO-INSIDE-POLICY
 class type inspect OUTSIDE-TO-INSIDE-EXCEPTION-CLASS
  inspect
 class class-default
  drop log
!
policy-map type inspect OUTSIDE-TO-SELF-POLICY
 class type inspect OUTSIDE-TO-SELF-CLASS
  pass
 class class-default
  drop log
!
! ==================== ZBFW: Zone-Pairs ====================
zone-pair security INSIDE-TO-OUTSIDE source INSIDE destination OUTSIDE
 service-policy type inspect INSIDE-TO-OUTSIDE-POLICY
!
zone-pair security OUTSIDE-TO-INSIDE source OUTSIDE destination INSIDE
 service-policy type inspect OUTSIDE-TO-INSIDE-POLICY
!
zone-pair security OUTSIDE-TO-SELF source OUTSIDE destination self
 service-policy type inspect OUTSIDE-TO-SELF-POLICY
!
! ==================== NAT (ตาม Part 15) ====================
ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
ip nat inside source list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 overload
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### ขั้นตอน Deploy พร้อม Safety Net (สำคัญมากสำหรับ ZBFW)

```
WAN-EDGE-1# reload in 10
Reload scheduled for 16:40:00 UTC Sat Sep 26 2026 (in 10 minutes) by admin
Proceed with reload? [confirm]

WAN-EDGE-1# configure terminal
! -- ใส่ Config ทั้งหมดข้างต้นตามลำดับ --
WAN-EDGE-1(config)# end

! ทดสอบ Management Access ผ่าน Session ใหม่ (ไม่ใช่ Session เดิม) ว่ายังเข้าได้ปกติจาก MGMT VLAN
! ถ้าเข้าได้ปกติ -> ยกเลิก reload
WAN-EDGE-1# reload cancel
Reload scheduled for 16:40:00 UTC Sat Sep 26 2026 is cancelled.

WAN-EDGE-1# copy running-config startup-config
```

> **เหตุผลที่ต้องใช้ `reload in` กับ ZBFW มากกว่า ACL ปกติ**: เพราะ `zone-member security`
> ทำให้ Traffic ข้าม Zone ถูก Deny **ทันที** ตั้งแต่บรรทัดแรกที่พิมพ์ (ก่อนจะพิมพ์ Zone-Pair
> ครบด้วยซ้ำ) — ถ้า Session ที่ใช้ Config อยู่วิ่งผ่าน Interface ที่กำลังจะเข้า Zone (เช่น
> ผ่าน Gi0/0/1 มาจาก Core) และยังไม่มี Zone-Pair ที่อนุญาต Traffic นั้น จะเกิดการล็อกตัวเองออก
> ทันทีระหว่างพิมพ์ Config เลย รุนแรงกว่า Pitfall ของ ACL ปกติใน
> [Part 14 Step 138](part-014-access-control-lists.md) เสียอีก

### Verification เต็มรูปแบบ

**Test 1: Campus (INSIDE) ออก Internet — ควรผ่านพร้อม Return Traffic**

```
PC1 (10.10.10.11)> curl -k https://www.example.com
HTTP/1.1 200 OK ...   <- สำเร็จ

WAN-EDGE-1# show policy-map type inspect zone-pair INSIDE-TO-OUTSIDE sessions
  Class-map: INSIDE-TO-OUTSIDE-CLASS (match-any)
    Inspect
      Established Sessions
       Session A1B2C3D4 (203.0.113.2:52301)=>(93.184.216.34:443) tcp SIS_OPEN
```

**Test 2: Internet (OUTSIDE) พยายาม SSH เข้า Server2 (ไม่มี Exception) — ควรถูก Block**

```
Internet-Host> ssh admin@203.0.113.2
% Connection timed out

WAN-EDGE-1# show policy-map type inspect zone-pair OUTSIDE-TO-INSIDE sessions
  Class-map: class-default (match-any)
    Drop (log)
     3 packets, 186 bytes
```

**Test 3: Internet (OUTSIDE) → Server1 HTTPS (Exception) — ควรผ่าน**

```
Internet-Host> curl -k https://203.0.113.10
HTTP/1.1 200 OK ...   <- สำเร็จ (NAT แปลงเป็น 10.10.30.10:443 แล้ว Inspect อนุญาตผ่าน Exception)
```

**Test 4: Internet (OUTSIDE) → Router เอง ด้วย ICMP — ควรผ่าน**

```
Internet-Host> ping 203.0.113.2
Reply from 203.0.113.2: bytes=32 time=18ms TTL=253   <- สำเร็จ (pass ผ่าน OUTSIDE-TO-SELF-POLICY)
```

**Test 5: Internet (OUTSIDE) → Router เอง ด้วย SSH — ควรถูก Block**

```
Internet-Host> ssh admin@203.0.113.2
% Connection timed out

WAN-EDGE-1# show policy-map type inspect zone-pair OUTSIDE-TO-SELF sessions
  Class-map: class-default (match-any)
    Drop (log)
     1 packets, 60 bytes
```

### สรุปผล Lab เทียบกับ Requirement

| # | Requirement | ผลการทดสอบ | สถานะ |
|---|---|---|---|
| 1 | 3 Zone (INSIDE/OUTSIDE/SELF) | `show zone security` เห็นครบทั้ง INSIDE, OUTSIDE, และ self (built-in) | ผ่าน |
| 2 | Campus ออก Internet พร้อม Stateful Return | Test 1 สำเร็จ, Session ปรากฏใน `sessions` state `SIS_OPEN` | ผ่าน |
| 3 | Block Internet เข้า Campus ยกเว้น Server1 HTTPS | Test 2 ถูก Block, Test 3 ผ่านสำเร็จตาม Exception | ผ่าน |
| 4 | Internet เข้า Router เองได้เฉพาะ ICMP | Test 4 ผ่าน, Test 5 ถูก Block | ผ่าน |

Lab นี้พิสูจน์ให้เห็นว่า WAN-EDGE-1 ทำหน้าที่เป็น **Stateful Firewall เต็มรูปแบบในตัว Router**
โดยไม่ต้องพึ่งพา Firewall แยกต่างหาก (ASA/FTD) — เหมาะสำหรับ Branch/Edge Site ขนาดเล็กถึงกลาง
ที่ต้องการ Security ระดับ Firewall แต่ไม่มี Budget ซื้ออุปกรณ์เพิ่มเติม

---

## แบบฝึกหัดทวนความเข้าใจ Part 45

1. Object-Group Network และ Object-Group Service ต่างจากการเขียน ACE แบบดั้งเดิมอย่างไร และ
   มีข้อดีอะไรในระยะยาวเมื่อ Enterprise Lab มี Server/Service เพิ่มขึ้น?
2. เพราะเหตุใด `time-range` จึงต้องพึ่งพา NTP ให้ทำงานถูกต้อง และถ้า Clock ของอุปกรณ์ผิดจะเกิด
   ผลอะไรกับ ACE ที่ผูก `time-range` ไว้?
3. Reflexive ACL ต่างจาก keyword `established` อย่างไร และทำไม Reflexive ACL จึงรองรับ UDP/ICMP
   ได้ ในขณะที่ `established` ทำไม่ได้?
4. อธิบาย Default Behavior ของ ZBFW ทันทีหลัง `zone-member security` ถูก Apply เข้า Interface
   — ต่างจาก Default Behavior ของ ACL ทั่วไปอย่างไร?
5. Self-Zone มีพฤติกรรม Default ต่างจาก Zone อื่นอย่างไร และจะเปลี่ยนพฤติกรรมนั้นได้อย่างไร?

**เฉลย:**

1. Object-Group ให้เราจัดกลุ่ม IP Address (`network`) หรือ Port/Protocol (`service`) ไว้ในชื่อ
   เดียว แล้วอ้างอิงในหลาย ACE โดยไม่ต้องพิมพ์ IP/Port ซ้ำทุกบรรทัด — ข้อดีคือเมื่อต้องเพิ่ม/ลบ
   สมาชิกในกลุ่ม (เช่น เพิ่ม Server ใหม่) แก้ที่ Object-Group จุดเดียว มีผลกับทุก ACE ที่อ้างอิงกลุ่ม
   นั้นทันที ลดความเสี่ยงพิมพ์ผิด/ลืมแก้บางบรรทัด และทำให้ ACL อ่านง่ายขึ้นมากเมื่อ Policy ซับซ้อน
2. `time-range` (ทั้ง `periodic` และ `absolute`) อ้างอิงตาม Clock ภายในของอุปกรณ์เองโดยตรง ถ้า
   Clock ไม่ได้ Sync ด้วย NTP ให้ถูกต้อง ACE ที่ผูก `time-range` จะ Active/Inactive ผิดเวลาไปจาก
   ความเป็นจริงทั้งหมด (เช่น อาจ Active ตอนเที่ยงคืนแทนตอนเช้า) ทำให้ Security Policy ทำงานผิด
   จากที่ตั้งใจโดยสิ้นเชิง
3. `established` ตรวจสอบแค่ ACK/RST Flag ของ Packet เดี่ยวๆ (Stateless) ใช้ได้กับ TCP เท่านั้น
   เพราะ UDP/ICMP ไม่มี Flag ให้ตรวจ ส่วน Reflexive ACL สร้าง Temporary ACE ที่จำ Source/
   Destination/Port ของ Session จริง (ไม่สนใจ Flag) จึงใช้ได้กับ TCP, UDP, และ ICMP ได้ทั้งหมด
   และยากต่อการปลอมกว่า `established` ที่ปลอม Flag ได้
4. ทันทีที่ Interface ถูกใส่เข้า Zone ด้วย `zone-member security` — Traffic ที่จะไปยัง Zone อื่น
   (หรือ Interface ที่ยังไม่มี Zone) จะถูก **Deny โดยอัตโนมัติ** แม้ยังไม่มี Zone-Pair ใดๆ เลย
   (Default-Deny-Between-Zones) ตรงข้ามกับ ACL ทั่วไปที่ Default คือ Permit ทุกอย่างจนกว่าจะมี
   การเขียน `deny` ไว้อย่างชัดเจน
5. Self-Zone (Router เอง) มี Default Behavior เป็น **Permit** โดยอัตโนมัติสำหรับ Traffic ที่มี
   ต้นทาง/ปลายทางเป็น Router เอง แม้ยังไม่มี Zone-Pair ใดๆ ที่เกี่ยวข้องกับ Self-Zone เลยก็ตาม —
   ตรงข้ามกับ Zone ทั่วไป พฤติกรรมนี้จะเปลี่ยนไปใช้ Policy ที่กำหนดเองทันทีที่มีการสร้าง Zone-Pair
   ที่เกี่ยวข้องกับ Self-Zone อย่างชัดเจน (เช่น `zone-pair security ... source OUTSIDE destination self`)

---

## สรุป Part 45

Part นี้ยกระดับความรู้เรื่อง Security ระดับ Layer 3/4 จาก ACL พื้นฐานใน
[Part 14](part-014-access-control-lists.md) ไปสู่เทคนิคที่ใช้งานจริงใน Enterprise ระดับสูงขึ้น
— **Object-Group** ทำให้ ACL ดูแลรักษาง่ายขึ้นเมื่อ Policy ซับซ้อน, **Time-based ACL** เพิ่มมิติ
เรื่องเวลาเข้าไปในการควบคุมสิทธิ์, และ **Reflexive ACL** ให้พฤติกรรมกึ่ง Stateful บน Router
โดยไม่ต้องมี Firewall เต็มรูปแบบ ก่อนจะข้ามไปสู่จุดเปลี่ยนสำคัญที่สุดของ Part นี้คือ **IOS
Zone-Based Firewall (ZBFW)** — แนวคิด Zone/Zone-Pair ที่เปลี่ยน Default Behavior จาก
"Permit-by-default" ของ ACL ไปเป็น "Deny-by-default ระหว่าง Zone" พร้อม Stateful Inspection
เต็มรูปแบบผ่าน `inspect` action, Application-Layer Awareness, และ Self-Zone สำหรับปกป้อง
Management Plane ของ Router เอง ปิดท้ายด้วย Lab ใหญ่ที่ Deploy ZBFW จริงบน WAN-EDGE-1 ครบทั้ง
3 Zone พร้อม Static NAT Exception และการทดสอบยืนยันผลลัพธ์ครบทุก Requirement ✅

ZBFW คือรากฐานสำคัญของ Firewall-in-Router ที่ใช้จริงในงาน Enterprise Edge/Branch จำนวนมาก และ
เป็นพื้นฐานที่จะเชื่อมต่อกับแนวคิด **Control Plane Policing (CoPP)** ใน Part ต่อไป ซึ่งจะปกป้อง
**CPU ของอุปกรณ์เอง** จาก Traffic ที่มากเกินไป (Control Plane Layer ที่ Part 22 พูดถึงไว้เป็น
Preview) — เติมเต็มการป้องกันครบทั้ง 3 Plane (Management/Control/Data) ของ Enterprise Network

**พร้อมสำหรับ Part 46**: เราจะเรียนรู้ **Control Plane Policing (CoPP)** — วิธีใช้ QoS Policy
(`class-map`/`policy-map`/`service-policy`) จำกัด Rate ของ Traffic ที่เข้าสู่ CPU ของอุปกรณ์เอง
เพื่อป้องกัน DoS Attack ที่มุ่งเป้าทำให้ Control Plane ล่ม (เช่น ปลอม Routing Protocol Packet
จำนวนมาก, ARP Flood) แบบละเอียดเต็มรูปแบบ ต่อจาก Preview ที่แนะนำไว้ใน
[Part 22](part-022-aaa-device-hardening.md)

**ไปต่อ:** [Part 46 — Control Plane Policing (CoPP) →](part-046-control-plane-policing.md)
