# Part 35 — Advanced FHRP & Campus High Availability
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 341–350 จาก 1000**

> ต่อจาก [Part 34 — StackWise Virtual, VSS & MEC](part-034-stackwise-vss-mec.md) ที่รวม CORE-SW1
> และ CORE-SW2 ให้กลายเป็น **Logical Switch เดียว** ผ่าน StackWise Virtual (SVL) และเปลี่ยน
> Uplink คู่ของ DIST-SW1–4 (ที่เพิ่งทำ Dual-Homed + OSPF ECMP ไปใน
> [Part 33](part-033-advanced-stp-campus-design.md)) ให้กลายเป็น **Multi-chassis EtherChannel
> (MEC)** เส้นเดียวที่มองเห็น Core เป็นปลายทางเดียว — Part นี้จะกลับไปหา **HSRP** ที่เรียนพื้นฐาน
> ไว้แล้วใน [Part 18](part-018-fhrp.md) แล้วยกระดับให้ลึกถึงระดับ Enterprise/CCIE จริง: **MD5
> Authentication** ป้องกัน Rogue HSRP Speaker, **Multi-Object Tracking** ที่รวม Interface กับ
> IP SLA เข้าด้วยกัน, ข้อจำกัดการ Scale ของ HSRP ในองค์กรใหญ่, แนวคิด **Anycast Gateway** ที่ใช้ใน
> Fabric สมัยใหม่ (Preview ของ Part 84-85), **NSF/SSO** สำหรับ Supervisor Redundancy ภายในเครื่อง,
> และปิดท้ายด้วย Lab ที่รวมทุกเทคโนโลยี HA ของ Campus (STP + FHRP + MEC + ECMP + NSF/SSO) เข้าไว้
> ในการทดสอบ Failover เดียวที่จำลอง **CORE-SW1 ทั้งเครื่องล่ม** หลังทำ StackWise Virtual แล้ว

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 341 | HSRP MD5 Authentication — ป้องกัน Rogue HSRP Speaker |
| 342 | Multi-Object Tracking — รวม Interface + IP SLA พร้อม Priority Math |
| 343 | ข้อจำกัดการ Scale ของ HSRPv2 ในองค์กรขนาดใหญ่ |
| 344 | SVI-based Anycast Gateway — แนวคิดที่ใช้แทน HSRP ใน Fabric สมัยใหม่ |
| 345 | NSF และ SSO — Supervisor Redundancy ภายในเครื่อง |
| 346 | Redundancy Group (RG) สำหรับ Cross-Device NSF (CCIE-level) |
| 347 | Campus HA Design Checklist — รวม STP/FHRP/MEC/ECMP/NSF-SSO |
| 348 | HSRP กับ StackWise Virtual — SVL Failure ส่งผลต่อ Tracking อย่างไร |
| 349 | Verification/Troubleshooting HA ขั้นสูง |
| 350 | Lab เต็มรูปแบบ: MD5 + Multi-Object Tracking ทุก Group พร้อมทดสอบ CORE-SW1 ล่มทั้งเครื่อง |

---

## Step 341 — HSRP MD5 Authentication: ป้องกัน Rogue HSRP Speaker

### 341.1 ปัญหาที่ Authentication แก้ — ใครก็ส่ง Hello ปลอมได้

HSRP Hello เป็น Multicast Packet (`224.0.0.102` สำหรับ v2 ตาม [Step 175, Part 18]
(part-018-fhrp.md)) ที่ทุกอุปกรณ์บน VLAN เดียวกัน**เห็นได้หมด** — และถ้าไม่มี Authentication เลย
**อุปกรณ์ตัวใดก็ได้ที่เสียบเข้ามาในวงเดียวกัน (เช่น Laptop ผู้บุกรุกที่ต่อ Access Port) สามารถส่ง
HSRP Hello ปลอมที่อ้าง Priority สูงกว่าทุกตัวในกลุ่มได้ทันที**

```
        ก่อนมี Authentication (เสี่ยง)

   DIST-SW1 (Priority 150) ◄──Hello──► DIST-SW2 (Priority 100)
        │                                     │
        │         ผู้บุกรุกเสียบเข้า Access Port
        │         ส่ง Hello ปลอม: Group 10, Priority 255, Preempt
        │                     │
        └─────────────────────┴──────► ทั้ง DIST-SW1/DIST-SW2 เห็น Priority 255 สูงกว่าตัวเอง
                                        ยอม "ยกให้" ผู้บุกรุกเป็น Active ทันที (Preempt)

   ❌ ผู้บุกรุกกลายเป็น Active ของ VIP 10.10.10.1 — Traffic ทั้ง VLAN ถูก ARP ตอบด้วย MAC ของ
      อุปกรณ์ผู้บุกรุก → Man-in-the-Middle เต็มรูปแบบ (ดักอ่าน/แก้ไข Traffic ทุก Packet ที่ออกจาก
      VLAN นี้ก่อนส่งต่อไปที่ Gateway จริง หรือปล่อยให้ Black-Hole ไปเลยก็ได้)
```

นี่คือการโจมตีที่เรียกว่า **HSRP Hijacking** — เครื่องมือโจมตีสำเร็จรูป (เช่น Yersinia) ทำสิ่งนี้ได้
โดยไม่ต้องมีความรู้ Cisco ลึกเลย เพียงส่ง Crafted HSRP Packet ที่มี Priority สูงกว่าค่า Default
(100) และ Virtual IP ตรงกับที่ดักฟังมาจาก ARP/Hello ปกติในวง

> **ข้อเท็จจริงที่ต้องรู้**: แม้ไม่ได้ Config Authentication ใดๆ เลย **HSRPv1 มีค่า Authentication
> String พื้นฐานเป็นคำว่า `"cisco"` ที่ส่งเป็น Plaintext อยู่แล้วโดย Default** — ทุก Router/Switch
> Cisco ที่ไม่ได้ตั้งค่าเองจะ**ยอมรับ Hello ที่มีคำว่า "cisco"** เสมอ ทำให้ผู้บุกรุกที่รู้ Fact นี้
> โจมตีได้ง่ายขึ้นไปอีกถ้าไม่ Override ค่านี้ด้วยมือ

### 341.2 Plaintext Authentication (Legacy) — ทำไมไม่พอ

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 authentication text CAMPUS-KEY-2026
DIST-SW1(config-if)# exit
```

Plaintext Authentication ป้องกันได้แค่**ผู้บุกรุกที่ไม่รู้ Key เท่านั้น** แต่ตัว Key เอง **ถูกส่งเป็น
Cleartext อยู่ใน Hello Packet ทุกใบ** — ถ้าผู้บุกรุกดักฟัง Traffic ได้แค่ครั้งเดียว (Wireshark บน
Port ที่ Mirror มา หรือแค่ต่อ Hub/Switch เก่าที่ไม่มี Port Security) ก็อ่าน Key ออกมาได้ทันทีแล้ว
ปลอม Hello ต่อได้เหมือนไม่มี Authentication เลย — **Plaintext จึงกันได้แค่ "คนที่เดา Key สุ่มๆ"
ไม่ได้กันคนที่ Capture Packet ได้จริง**

### 341.3 MD5 Authentication — คำตอบที่ถูกต้องสำหรับ Production

MD5 Authentication ไม่ส่ง Key ตรงๆ ใน Packet แต่ใช้ Key มา**คำนวณ Hash (MD5 Digest)** แนบไปกับ
Hello แทน — อุปกรณ์ที่ได้รับ Hello จะคำนวณ Hash ด้วย Key ของตัวเองแล้วเทียบกับ Hash ที่แนบมา
**ถ้าไม่ตรงกัน (Key ผิด) ถือว่า Hello ใบนี้ไม่น่าเชื่อถือ และ Reject ทันที** โดย Key จริงไม่เคยถูกส่ง
ผ่าน Network เลยแม้แต่ครั้งเดียว

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 authentication md5 key-string 0 C1sco-HSRP-V10-P35!
DIST-SW1(config-if)# exit

DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# standby 10 authentication md5 key-string 0 C1sco-HSRP-V10-P35!
DIST-SW2(config-if)# exit
```

| ส่วนของคำสั่ง | ความหมาย |
|---|---|
| `standby 10 authentication md5` | เปิด MD5 Authentication สำหรับ HSRP Group 10 |
| `key-string 0 <key>` | ใส่ Key เป็น Plaintext ตรงๆ ใน Config (เลข `0` = ไม่ Encrypt ตอนพิมพ์ แต่ `show run` จะซ่อนด้วย `service password-encryption` ถ้าเปิดไว้ตาม [Part 22](part-022-aaa-device-hardening.md)) |
| Key ต้อง**เหมือนกันทุกตัว**ในกลุ่มเดียวกัน | เหมือนหลักการ VIP ที่ต้องตรงกันทุกตัวจาก [Step 173, Part 18](part-018-fhrp.md) |

> **กฎสำคัญ**: Key ควรตั้งให้ **ต่างกันไปในแต่ละ HSRP Group** (ไม่ใช้ Key เดียวกันทุก VLAN) เพื่อ
> จำกัดผลกระทบถ้า Key ของ Group ใด Group หนึ่งรั่ว — ผู้บุกรุกจะปลอม Hello ได้แค่ Group นั้น ไม่ใช่
> ทุก VLAN พร้อมกัน (Blast Radius Containment เช่นเดียวกับหลักการแบ่ง VLAN ของ [Part 3]
> (part-003-ethernet-switching-vlan.md))

### 341.4 Key-Chain — ทางเลือกสำหรับ Key Rotation (ระดับ CCIE)

สำหรับ Production ที่ต้องหมุน Key ตามรอบ (เช่นทุก 90 วันตาม Security Policy) การพิมพ์ Key ตรงๆ
ด้วย `key-string` ทำให้ต้อง Config ใหม่ทุกครั้งที่หมุน Key และมีช่วง Downtime สั้นๆ ระหว่างเปลี่ยน
Key ไม่ตรงกันทั้งสองฝั่ง — Cisco จึงรองรับการอ้าง **Key Chain** (Feature เดียวกับที่ EIGRP/RIP
Authentication ใช้) ซึ่งรองรับ**หลาย Key พร้อมกันโดยมี Accept/Send Lifetime ซ้อนกัน** ทำให้หมุน
Key ได้แบบไม่มี Downtime เลย:

```
DIST-SW1(config)# key chain HSRP-VLAN10-CHAIN
DIST-SW1(config-keychain)# key 1
DIST-SW1(config-keychain-key)# key-string C1sco-HSRP-V10-P35!
DIST-SW1(config-keychain-key)# accept-lifetime 00:00:00 Jan 1 2026 23:59:59 Mar 31 2026
DIST-SW1(config-keychain-key)# send-lifetime 00:00:00 Jan 1 2026 23:59:59 Mar 31 2026
DIST-SW1(config-keychain-key)# exit
DIST-SW1(config-keychain)# key 2
DIST-SW1(config-keychain-key)# key-string N3xt-Qu4rter-HSRP-V10!
DIST-SW1(config-keychain-key)# accept-lifetime 00:00:00 Mar 1 2026 23:59:59 Jun 30 2026
DIST-SW1(config-keychain-key)# send-lifetime 00:00:00 Mar 1 2026 23:59:59 Jun 30 2026
DIST-SW1(config-keychain-key)# exit
DIST-SW1(config-keychain)# exit
!
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 authentication md5 key-chain HSRP-VLAN10-CHAIN
```

> **สำหรับ Lab หลักของหลักสูตรนี้**: ใช้ `key-string` ตรงๆ พอ (Step 350) เพราะไม่มีความจำเป็นต้อง
> หมุน Key ตามรอบใน Lab การศึกษา — แต่ต้องรู้จัก Key-Chain ไว้สำหรับข้อสอบ CCNP ENCOR และงานจริงใน
> องค์กรที่มี Security Policy บังคับ Key Rotation

### 341.5 Verify และอาการเมื่อ Key ไม่ตรงกัน

```
DIST-SW1# show standby vlan 10
Vlan10 - Group 10 (version 2)
  State is Active
  Virtual IP address is 10.10.10.1
  Active virtual MAC address is 0000.0c9f.f00a
  Hello time 3 sec, hold time 10 sec
  Preemption enabled
  Authentication MD5, key-string
  Priority 150 (configured 150)
  Standby router is 10.10.10.3, priority 100 (expires in 8.912 sec)
```

ถ้า Key ไม่ตรงกัน (เช่น พิมพ์ผิดตอน Config DIST-SW2) — ทั้งสองฝั่งจะ**มองไม่เห็น Hello ของกันเลย**
(Hello ถูก Reject เงียบๆ ที่ระดับ Authentication ก่อนถึง State Machine) และ Log แสดง:

```
%HSRP-4-BADAUTH: Bad authentication from 10.10.10.3, group 10, remote state Active
```

ผลคือ**ทั้งสองฝั่งประกาศตัวเป็น Active พร้อมกัน (Split-Brain ระดับ FHRP)** — Host บางเครื่องได้ ARP
Reply จาก DIST-SW1 บางเครื่องได้จาก DIST-SW2 สลับกันไปมา เกิด Duplicate Virtual MAC บน Network
เดียวกัน — **นี่คือสาเหตุที่ต้อง Verify Key ให้ตรงกันทั้งสองฝั่งทุกครั้งที่ Deploy** (เทียบได้กับ
เช็คลิสต์ VIP/Group Number ตรงกันจาก [Step 179, Part 18](part-018-fhrp.md) — เพิ่ม Key เข้าไปเป็น
ข้อที่ 7 ของเช็คลิสต์นั้น)

---

## Step 342 — Multi-Object Tracking: รวม Interface + IP SLA พร้อม Priority Math

### 342.1 ข้อจำกัดของ Interface Tracking เดี่ยว (ทวนจาก Part 18)

[Step 174, Part 18](part-018-fhrp.md) สอน `standby track <interface> decrement <value>` ซึ่งจับ
ได้แค่ **"สายที่ต่อออกจาก Switch ตัวนี้เอง"** ขาดหรือไม่ — แต่ยังมีความล้มเหลวอีกประเภทที่ Interface
Tracking เดี่ยวมองไม่เห็นเลย: **Interface ยัง Up/Up ปกติ แต่ปลายทางที่ไกลออกไป (ผ่าน Core) ใช้งาน
ไม่ได้จริง** เช่น Core มีปัญหา Routing ภายใน, เกิด Micro-Loop, หรือ (อย่างที่จะเห็นใน Step 348)
Core ที่ทำ StackWise Virtual เกิด Dual-Active — สถานการณ์เหล่านี้ **Link Light ยังติดปกติทุก
ประการ** ทำให้ Interface Tracking แบบ Binary (Up/Down) ไม่มีวัน Trigger

### 342.2 แนวคิด: รวม Track Object 2 ประเภทเข้าด้วยกัน

HSRP รองรับการผูก **Track Object หลายตัวพร้อมกันใน Group เดียว** โดยแต่ละตัวมี Decrement ของ
ตัวเอง — ถ้า Track หลายตัว Down พร้อมกัน **Decrement จะถูกบวกรวมกัน (Additive)** เข้ากับ Priority
ปกติ Design นี้ใช้แนวคิดจาก **IP SLA + Track** ที่สอนไว้แล้วใน [Step 314, Part 32]
(part-032-policy-based-routing.md) มาผสมกับ Interface Tracking ของ Part 18:

```
                DIST-SW1 (VLAN 10, Priority ปกติ 150)

   Track 1: interface TenGigabitEthernet1/1/1 line-protocol
            (Physical Member ตัวหนึ่งของ MEC Port-channel1 ไป Core — ดู Part 34)
            Decrement 30   ── สัญญาณ "Local Link เสียครึ่งหนึ่ง แต่ Bundle ยังพอใช้ได้"

   Track 2: ip sla 1 reachability
            (ICMP Echo ข้าม Core ไปยัง Loopback0 ของ DIST-SW2 คู่ตัวเอง — พิสูจน์ Path จริง
             ที่ผ่าน Core ทั้งระบบ ไม่ใช่แค่ตรวจ Link ตรงหน้า)
            Decrement 30   ── สัญญาณ "Path จริงข้าม Core มีปัญหา"

   กฎ: Decrement รวมกันได้ (Additive) เมื่อ Track หลายตัว Down พร้อมกัน
```

### 342.3 คอนฟิกจริง

```
DIST-SW1(config)# track 1 interface TenGigabitEthernet1/1/1 line-protocol
DIST-SW1(config-track)# exit
!
DIST-SW1(config)# ip sla 1
DIST-SW1(config-ip-sla)# icmp-echo 1.1.1.12 source-interface Loopback0
DIST-SW1(config-ip-sla-echo)# frequency 5
DIST-SW1(config-ip-sla-echo)# timeout 1000
DIST-SW1(config-ip-sla-echo)# exit
DIST-SW1(config)# ip sla schedule 1 life forever start-time now
!
DIST-SW1(config)# track 2 ip sla 1 reachability
DIST-SW1(config-track)# delay down 2 up 5
DIST-SW1(config-track)# exit
!
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 track 1 decrement 30
DIST-SW1(config-if)# standby 10 track 2 decrement 30
DIST-SW1(config-if)# exit
```

> **ต่างจาก Part 18 อย่างไร**: Part 18 ใช้รูปแบบย่อ `standby 10 track TenGig... decrement 60`
> (สร้าง Track Object ให้อัตโนมัติ) ซึ่งใช้ได้แค่กับ Interface เท่านั้น — พอต้องผสม IP SLA เข้ามาด้วย
> ต้อง **ประกาศ `track <number>` แบบเต็มที่ Global Config ก่อน** แล้วอ้างด้วยเลข Object ใน `standby
> <group> track <number> decrement <value>` แทน

### 342.4 Priority Math — ตารางคำนวณทุก Scenario

DIST-SW1 Priority ปกติ = 150, DIST-SW2 (Standby) Priority = 100 (preempt เปิดทั้งคู่ตาม Part 18)

| สถานการณ์ | Track 1 (Member Link) | Track 2 (IP SLA ข้าม Core) | คำนวณ Priority | ผลลัพธ์ |
|---|---|---|---|---|
| ปกติ | Up | Up | 150 | **Active** (ไม่เปลี่ยน) |
| Member Link เสีย 1 เส้น เท่านั้น (MEC ยังพอ Forward ผ่านเส้นที่เหลือ) | Down | Up | 150 − 30 = **120** | **ยังเป็น Active** (120 > 100 — ทน Signal เดี่ยวได้ ไม่ Flap เกินความจำเป็น) |
| Path ผ่าน Core มีปัญหา แต่ Local Link ปกติทุกเส้น | Up | Down | 150 − 30 = **120** | **ยังเป็น Active** (เหตุผลเดียวกัน) |
| **ทั้งสอง Track Down พร้อมกัน** (Local Link เสีย + Path ข้าม Core ก็เสียด้วย — สัญญาณยืนยันปัญหาจริงจาก 2 แหล่งอิสระ) | Down | Down | 150 − 30 − 30 = **90** | **Failover!** 90 < 100 → DIST-SW2 ชนะ Election |

> **หลักการออกแบบที่สำคัญที่สุดของ Step นี้**: การตั้ง Decrement ให้ **Track เดี่ยวไม่พอ Trigger
> Failover เอง (120 > 100) แต่ต้องมีสัญญาณยืนยันจาก 2 แหล่งอิสระพร้อมกันก่อนถึงจะ Failover (90 <
> 100)** เป็นการป้องกัน **False Positive จาก Probe ตัวเดียวที่ไม่เสถียร** (เช่น IP SLA Timeout ครั้ง
> เดียวเพราะ CPU Core สูงชั่วคราว ไม่ใช่ปัญหาจริง) ในขณะที่ยังจับปัญหาจริงที่ Interface Tracking
> เดี่ยวจาก Part 18 มองไม่เห็นได้ — Trade-off นี้ต้องปรับ Decrement ให้เหมาะกับ Design แต่ละองค์กร
> เอง ไม่มีค่าตายตัวที่ถูกทุกที่

### 342.5 Verify

```
DIST-SW1# show standby vlan 10
Vlan10 - Group 10 (version 2)
  State is Active
  Priority 150 (configured 150)
  Track object 1 (interface Te1/1/1), state Up, decrement 30
  Track object 2 (ip sla 1), state Up, decrement 30
  Standby router is 10.10.10.3, priority 100 (expires in 8.500 sec)

DIST-SW1# show track brief
Track   Type        Instance      Parameter    State       Last Change
1       interface   Te1/1/1       line-proto   Up          00:45:12
2       ip sla      1             reachability Up          00:45:10
```

---

## Step 343 — ข้อจำกัดการ Scale ของ HSRPv2 ในองค์กรขนาดใหญ่

### 343.1 ทำไม "1 VLAN = 1 HSRP Group" ถึงมีปัญหาเมื่อ VLAN เยอะ

หลักสูตรนี้ใช้ 5 VLAN บน DIST-SW1–4 (Step 180, Part 18) ซึ่งน้อยมากเทียบกับ Enterprise Campus จริง
ที่อาจมี VLAN **หลักร้อยถึงหลักพัน** (แยกตามแผนก, Floor, Tenant) — ถ้าแต่ Design ยังคงเป็น "1 VLAN
= 1 HSRP Group" ต่อไปเรื่อยๆ ปัญหาที่เกิดคือ:

```
   Overhead ต่อ HSRP Group 1 กลุ่ม (2 สมาชิก Active+Standby):
   - Hello Multicast ทุก 3 วินาที × 2 ตัว = ~0.67 packet/sec ต่อ Group

   ถ้ามี 500 VLAN (= 500 HSRP Group) บน Switch คู่เดียว:
   - Hello Multicast รวม = 500 × 0.67 ≈ 335 packet/sec ที่ต้อง Generate/Process ต่อเนื่องตลอดเวลา
   - แต่ละ Group มี State Machine ของตัวเอง (Step 172) ต้องเก็บ Timer/State แยกกันหมด 500 ชุด
   - CPU ใช้ Cycle ประมวลผล Hello Parsing + State Check ต่อ Group เพิ่มขึ้นเป็นเงาตามจำนวน Group
```

### 343.2 ผลกระทบที่เกิดขึ้นจริง

| ผลกระทบ | รายละเอียด |
|---|---|
| **CPU Load สูงขึ้นต่อเนื่อง** | ยิ่ง Group มาก ยิ่งต้อง Process Hello/State Machine มาก แม้ไม่มี Failover เกิดขึ้นเลยก็ตาม (Steady-state Overhead) |
| **CoPP (Control Plane Policing) อาจ Rate-Limit Multicast Hello** | ถ้า Multicast Traffic โดยรวม (รวม HSRP กับ Protocol อื่นที่ใช้ Multicast) เกิน Threshold ที่ CoPP ตั้งไว้เพื่อป้องกัน DoS ต่อ CPU — Hello บาง Packet อาจถูก Drop ทำให้เกิด **False Failover** (Standby คิดว่า Active หายไปทั้งที่จริงยังอยู่ปกติ เพราะ Hello แค่ถูก CoPP Drop) |
| **Group Number ชนกันข้าม VLAN ยากขึ้น** | แม้ HSRPv2 รองรับ 0–4095 Group (Step 175) แต่ถ้า Design ผูก Group Number = VLAN ID ตรงๆ ตามธรรมเนียมหลักสูตรนี้ VLAN ที่เกิน 4095 ก็เป็นไปไม่ได้อยู่แล้วในทางปฏิบัติ (VLAN ID สูงสุด 4094 พอดี ไม่ใช่ปัญหาจริงในทางเลข แต่เป็นภาระบริหาร Documentation ที่ต้อง Track ว่า Group ไหนตรงกับ VLAN ไหนที่ Switch คู่ไหน) |

### 343.3 แนวทางบรรเทา (Mitigation) 3 ระดับ

| แนวทาง | วิธีทำ | Trade-off |
|---|---|---|
| **1. เพิ่ม Hello/Hold Timer บน Network ที่เสถียรแล้ว** | `standby <group> timers 10 30` (จาก Default 3/10 วินาที) บน VLAN ที่ Failover ไม่จำเป็นต้องเร็วมาก (เช่น VLAN Guest/Non-critical) | Failover ช้าลง (30 วินาทีก่อนรู้ว่า Active หาย) แต่ลด Hello Overhead ลงกว่า 3 เท่า |
| **2. ลดจำนวน Group ด้วยการลด VLAN/Subnet** | ออกแบบใหม่ให้ VLAN ใหญ่ขึ้น (Subnet /22-/23 แทน /24 หลายใบ) รวม Host หลาย Segment เดิมเข้าด้วยกัน | ลด Broadcast Domain Isolation, ต้องคิด Security/QoS Policy ใหม่ให้รอบคอบกว่าเดิม |
| **3. เปลี่ยนไปใช้ GLBP หรือ Anycast Gateway** | GLBP ยังมี Group Overhead คล้ายกัน (ไม่ช่วยเรื่อง Scale จริง) ส่วน **Anycast Gateway** (Step 344) **ไม่มี Protocol Overhead เลย** เพราะไม่มี Hello/Election ให้ Process | ต้องเปลี่ยน Design ทั้ง Fabric (SD-Access/VXLAN EVPN) — ไม่ใช่การปรับเล็กๆ แต่เป็นการ Migrate Architecture |

> **ข้อสรุปสำหรับ CCNP ENCOR**: ที่ Scale ของหลักสูตรนี้ (5 VLAN) HSRP Overhead **ไม่ใช่ปัญหาเลย**
> — แต่ข้อสอบและงานจริงในองค์กรระดับ Enterprise/Service Provider ต้องรู้ว่า **เมื่อ VLAN Scale ถึง
> หลักร้อยขึ้นไป ต้อง Plan Timer และพิจารณา Architecture ทางเลือกล่วงหน้า** ไม่ใช่รอให้ CPU สูงแล้ว
> ค่อยแก้ทีหลัง

---

## Step 344 — SVI-based Anycast Gateway: แนวคิดที่ Fabric สมัยใหม่ใช้แทน HSRP

### 344.1 ปัญหาของ Model แบบ "VIP ต่อคู่ Switch" เมื่อมี Gateway หลายสิบตัว

HSRP/VRRP/GLBP ทั้งหมดที่เรียนมา (Part 18, Step 341-343) ถูกออกแบบมาบนสมมติฐานว่า **มี Physical
Gateway แค่ 2-3 ตัวต่อ VLAN** เท่านั้น (เช่น DIST-SW1/DIST-SW2 คู่เดียวต่อ VLAN 10) — แต่ใน Design
แบบ **Spine-Leaf Fabric** (SD-Access, VXLAN/EVPN ที่จะเรียนเต็มรูปแบบใน **Part 84-85** ของ
หลักสูตรนี้) VLAN เดียวอาจกระจายตัวอยู่บน **Leaf Switch หลักสิบตัวทั่วทั้งอาคาร** — ถ้ายังใช้ HSRP
Election แบบเดิม ต้องมี Leaf ทั้งหมดแข่งกันเป็น Active/Standby/Listen ใน Group เดียว (State
`Listen` จาก [Step 172, Part 18](part-018-fhrp.md) ที่บอกว่า "ปกติไม่เห็นใน Lab นี้เพราะมีแค่ 2
ตัว" — แต่ใน Fabric จริงจะเห็น Listen เป็นสิบๆ ตัวพร้อมกัน) ซึ่ง**ไม่ Scale เลย**

### 344.2 แนวคิด Anycast Gateway — ทุก Leaf คือ Active ตัวของตัวเอง

```
        HSRP แบบเดิม (Part 18)                Anycast Gateway (Fabric สมัยใหม่)

    VIP เดียว .1 ── Active/Standby         VIP เดียว .1 ── ทุก Leaf เป็น "Active" ของตัวเอง
    ┌────────┐  ┌────────┐                 ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
    │ DIST-SW1│  │ DIST-SW2│                │ Leaf-1  │ │ Leaf-2  │ │ Leaf-3  │ │ Leaf-N  │
    │ ACTIVE  │  │ STANDBY │                │ .1 SVI  │ │ .1 SVI  │ │ .1 SVI  │ │ .1 SVI  │
    │ ตอบ ARP │  │ รอสำรอง │                │ MAC เดียวกัน ทุกตัว (Anycast MAC) — ตอบ ARP ในพื้นที่│
    └────────┘  └────────┘                 │ ตัวเองทันที ไม่ต้อง Forward ไปหา Leaf อื่นเลย        │
                                            └──────────────────────────────────────────────────┘

    Election/Hello ระหว่าง 2 ตัว                ไม่มี Election, ไม่มี Hello ระหว่าง Leaf เลย
    ต้องรอ Hold Timer ก่อน Failover              First-Hop อยู่ที่ Leaf ที่ Host เสียบอยู่เสมอ
                                                 (Local Gateway ตลอดเวลา ไม่มี "ตัวไหน Active กว่า")
```

**หลักการ**: ทุก Leaf Switch ที่มี VLAN/VNI นั้นอยู่ **Config SVI IP เดียวกัน + Virtual MAC เดียวกัน
เหมือนกันทุกตัวแบบ Static** (ไม่มี Protocol เจรจา ไม่มี Priority ไม่มี Preempt) — Host ที่เสียบอยู่
ที่ Leaf ตัวไหน จะได้รับการตอบ ARP จาก Leaf ตัวนั้นทันที (**First Hop จึงอยู่ใกล้ Host เสมอ**) ส่วน
การรู้ว่า Host แต่ละตัวอยู่ตรงไหนของ Fabric (เพื่อ Route ไปหาได้ถูก) เป็นหน้าที่ของ **Control Plane
ของ Fabric เอง** (เช่น LISP หรือ EVPN Type-2 Route ที่จะเรียนใน Part 84-85) ไม่ใช่หน้าที่ของ FHRP
อีกต่อไป

### 344.3 ทำไม Fabric Design ถึง "เลิกใช้ HSRP" ไปเลย ไม่ใช่แค่ Tune ให้ดีขึ้น

| มุมมอง | HSRP/VRRP/GLBP | Anycast Gateway |
|---|---|---|
| จำนวน Gateway ต่อ VLAN ที่รองรับได้ดี | 2 ตัว (บางที GLBP ทำได้ถึง 4 AVF) | **ไม่จำกัด** (Leaf กี่สิบตัวก็ได้) |
| Overhead ตอน Steady-State | มี Hello ตลอดเวลา (Step 343) | **ไม่มีเลย** — Static Config ล้วนๆ |
| Failover Time เมื่อ Gateway ตัวหนึ่งล่ม | ขึ้นกับ Hold Timer (ไม่กี่วินาที) | **ไม่มี Failover ให้รอ** — Host ที่อยู่ Leaf อื่นไม่เคยพึ่ง Leaf ที่ล่มอยู่แล้วตั้งแต่ต้น |
| ความซับซ้อนที่ต้องแลก | ต่ำ (Election ธรรมดา) | สูงกว่ามาก — ต้องมี Fabric Control Plane (LISP/EVPN) คอย Advertise ตำแหน่ง Host แทน |

> **สรุปสำหรับ Step นี้**: Anycast Gateway ไม่ได้ "ดีกว่า HSRP ในทุกกรณี" — มันคือคำตอบที่ถูกออกแบบ
> มาสำหรับปัญหาคนละแบบ (Scale ระดับ Fabric ที่มี Gateway หลักสิบตัว) ส่วน HSRP ยังคงเป็นคำตอบที่ดี
> ที่สุดสำหรับ Campus แบบ 3-Tier ดั้งเดิมที่หลักสูตรนี้ใช้เป็น Lab หลัก (2 ตัวต่อคู่ Distribution)
> รายละเอียดการ Config Anycast Gateway จริงบน Fabric จะอยู่ใน **Part 84-85** — Step นี้เป็นแค่การ
> ปูพื้นแนวคิดให้เห็นภาพว่า FHRP ที่เรียนมาทั้ง Part 18 และ Part นี้ **ไม่ใช่คำตอบสุดท้ายตลอดไป**
> เมื่อ Scale ขององค์กรเปลี่ยนไป

---

## Step 345 — NSF และ SSO: Supervisor Redundancy ภายในเครื่องเดียว

### 345.1 นี่คือ HA คนละระดับกับ FHRP — ต้องแยกให้ชัด

FHRP (HSRP/VRRP/GLBP) แก้ปัญหา **"อุปกรณ์ทั้งเครื่องล่ม"** โดยมีอุปกรณ์คู่แข่งอีกตัวรับหน้าที่แทน —
แต่ยังมีความล้มเหลวอีกแบบที่ **ไม่จำเป็นต้องให้ทั้งเครื่องล่ม**: Platform ระดับ Modular Chassis
(เช่น Catalyst 9600 Series หรือ Catalyst 6807-XL รุ่นเก่า) รองรับการเสียบ **Supervisor Module 2
ตัวในแชสซีเดียวกัน** — ถ้า Supervisor ตัว Active พังหรือต้อง Reload (เช่น Software Upgrade) แต่
Chassis ตัวเดียวกันยังมี Line Card/Power/Fan ทำงานปกติทุกอย่าง **ไม่มีความจำเป็นต้อง Failover ไป
อุปกรณ์อีกเครื่องเลยด้วยซ้ำ** ถ้า Supervisor สำรองตัวที่สองรับงานต่อได้ทันที

> **หมายเหตุสำคัญเรื่อง Platform ของ Lab หลักสูตรนี้**: CORE-SW1/CORE-SW2 (Catalyst 9500) และ
> DIST-SW1–4 (Catalyst 9300) ตาม [00-ip-address-plan.md](00-ip-address-plan.md) เป็น
> **Fixed-Configuration Platform ที่มี Supervisor ตัวเดียวในตัวเอง** (ไม่มี Slot สำหรับ Supervisor
> สำรองแบบ Modular Chassis) — StackWise Virtual ที่ทำใน [Part 34](part-034-stackwise-vss-mec.md)
> แก้ปัญหา "Chassis ทั้งเครื่องล่ม" ด้วยการรวม **2 เครื่องแยกกัน** ให้ดูเป็น 1 เครื่อง (คนละกลไกกับ
> NSF/SSO ที่ทำงาน **ภายในเครื่องเดียว**) — Step นี้จึงเป็นความรู้ระดับ Concept สำหรับ Platform
> Modular ที่ CCNP ENCOR Exam ถามถึง แม้ Lab ของหลักสูตรนี้จะไม่มี Hardware ที่ Demo ได้จริง

### 345.2 SSO (Stateful Switchover) — กลไกพื้นฐาน

**SSO** คือกลไกที่ Supervisor สำรอง (Standby RP) **Sync State กับ Supervisor หลัก (Active RP)
อยู่ตลอดเวลาแบบ Real-time** (L2 Forwarding Table, Interface State, Chassis Inventory ฯลฯ) ทำให้
เมื่อ Active RP ล่ม Standby RP **เข้ารับหน้าที่ทันทีโดยไม่ต้อง Reload Line Card เลย** — Data Plane
(การ Forward Packet ที่ Hardware ทำอยู่แล้วผ่าน ASIC) **ไม่หยุดทำงานแม้แต่วินาทีเดียว** เพราะ Line
Card ไม่ได้ผ่านการ Reset ไปด้วย

### 345.3 NSF (Non-Stop Forwarding) — ส่วนขยายที่ทำงานร่วมกับ SSO ที่ Layer 3

SSO เองจัดการแค่ระดับ Chassis/L2 — แต่ **Routing Protocol** (OSPF, EIGRP, BGP) ที่รันบน Supervisor
เดิมจะต้อง**เริ่มสร้าง Neighbor Relationship และ Routing Table ใหม่ทั้งหมด**ถ้าไม่มีกลไกพิเศษ ซึ่ง
ระหว่างที่ Routing Table ยังไม่พร้อม **Traffic อาจถูก Drop ได้แม้ Data Plane เดิมจะยังพร้อม
Forward อยู่** — **NSF** แก้ปัญหานี้ด้วย **Graceful Restart (RFC 3623 สำหรับ OSPF)** ที่เรียนไว้แล้ว
ใน [Step 256, Part 26](part-026-advanced-ospf.md):

```
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# nsf cisco
```

หลักการเดียวกับ Step 256: Router ที่กำลัง Switchover ส่ง **Grace-LSA** บอก Neighbor ว่า "อย่าตัด
Adjacency กับฉันนะ ฉันแค่กำลัง Restart Control Plane ภายใน" — Neighbor ที่เป็น **NSF-aware
(Helper)** จะ**คง Routing Information เดิมไว้ให้ชั่วคราว** จนกว่า Router ที่ Restart จะ Sync
Routing Table กลับมาสมบูรณ์ — รวมกับ SSO ที่คง Data Plane ไว้แล้ว **ผลลัพธ์คือ Traffic ไม่หยุดแม้แต่
Packet เดียวระหว่าง Supervisor Switchover ทั้งกระบวนการ**

### 345.4 ความสัมพันธ์กับ FHRP — HA คนละชั้นที่ทำงานร่วมกัน

```
   ลำดับการป้องกันความล้มเหลวแบบเป็นชั้น (Defense in Depth ของ Campus HA)

   ชั้นที่ 1: NSF/SSO         → ป้องกัน "Supervisor เดียวในเครื่องเดียวกันล่ม" (Platform รองรับ)
                                 → ถ้าทำงานสำเร็จ ภายนอกไม่เห็นอะไรเปลี่ยนแปลงเลย ไม่ต้องรอ HSRP
   ชั้นที่ 2: StackWise Virtual → ป้องกัน "Chassis ทั้งเครื่องล่ม" (Part 34, ระดับ 2 กล่องแยกกัน)
                                 → ถ้าทำงานสำเร็จ MEC/OSPF ที่ DIST เห็นก็ไม่เปลี่ยนแปลง
   ชั้นที่ 3: HSRP (FHRP)      → ป้องกัน "Distribution ทั้งคู่ที่ Host พึ่งพาตัวหนึ่งล่มจริง"
                                 → ทำงานเมื่อชั้น 1-2 "เอาไม่อยู่" เท่านั้น (เช่น Part 34 Step 348)

   ยิ่งชั้นบนทำงานสำเร็จมากเท่าไหร่ ยิ่งไม่ต้องพึ่งชั้นล่างเลย — Host ไม่รู้สึกอะไรเลยแม้แต่ครั้งเดียว
```

> **นี่คือหัวใจของ Step 347 (Design Checklist)** — HA ที่ดีไม่ได้พึ่งเทคโนโลยีเดียว แต่ซ้อนหลายชั้น
> ให้แต่ละชั้นจับความล้มเหลวคนละขนาดกัน

---

## Step 346 — Redundancy Group (RG): Cross-Device NSF (CCIE-level Awareness)

NSF/SSO ใน Step 345 ทำงาน**ภายในแชสซีเดียวกันเท่านั้น** (2 Supervisor ใน 1 กล่อง) — Cisco IOS-XE
บน Platform บางกลุ่ม (โดยเฉพาะ **ASR1000 Series** ที่ใช้เป็น WAN Edge/Router ระดับสูง และ SD-WAN
Edge) มี Feature ที่ไปได้ไกลกว่านั้นเรียกว่า **Redundancy Group (RG)** ซึ่งทำ Stateful Failover
**ข้ามแชสซี 2 กล่องที่เป็นอุปกรณ์แยกกันจริง** โดย Sync State ของ Service ที่มี Session อยู่ (เช่น
NAT Translation Table จาก [Part 15](part-015-nat-pat.md), Zone-Based Firewall Session, หรือแม้แต่
FHRP เองบาง Implementation) ผ่าน Link เฉพาะระหว่าง 2 กล่อง คล้ายหลักการ SVL ของ StackWise Virtual
(Part 34) แต่ทำงานที่ระดับ **Service/Session State ไม่ใช่ Chassis Identity**

```
   Box-to-Box (B2B) Redundancy ด้วย Redundancy Group (แนวคิดคร่าวๆ)

   Router-A (Active RG)  ◄──── RG Protocol Link (Sync NAT Table, Session State) ────►  Router-B (Standby RG)
        │                                                                                    │
        └── ถ้า Router-A ล่ม → Router-B รับ Session ต่อ "โดยไม่ต้องให้ Client เริ่ม Connection ใหม่"
            (ต่างจาก HSRP ธรรมดาที่ Gateway เปลี่ยนตัวได้ แต่ Session/NAT Table เดิมหายไปหมด)
```

> **สถานะความรู้ที่ต้องมีสำหรับข้อสอบ**: Redundancy Group เป็นหัวข้อระดับ **CCIE Enterprise
> Infrastructure** ไม่ใช่ CCNP ENCOR โดยตรง — Step นี้จึงเป็นแค่การแนะนำให้รู้จัก **ชื่อและแนวคิด**
> ว่ามีเทคโนโลยีที่ไปไกลกว่า NSF/SSO (ข้ามแชสซี) และไปไกลกว่า HSRP ธรรมดา (Sync Session State ไม่
> ใช่แค่ Gateway Role) — Lab หลักของหลักสูตรนี้ไม่ Config RG เพราะ Platform ในหลักสูตร (Catalyst
> 9300/9500) ไม่ใช่กลุ่ม Platform ที่ใช้ RG เป็นหลัก (RG พบมากใน ASR1000/ISR4000 WAN Edge Role)

---

## Step 347 — Campus HA Design Checklist: รวม STP/FHRP/MEC/ECMP/NSF-SSO

ถึงจุดนี้หลักสูตรได้สอน HA Layer ของ Campus ครบทุกชั้นแล้ว — Step นี้รวบเป็น **ตารางตัดสินใจ
เดียว** ที่ใช้ตรวจสอบ Design จริงว่าครอบคลุม Failure Domain ทุกแบบหรือยัง

### 347.1 ตาราง Failure Domain × เทคโนโลยีที่รับผิดชอบ

| Failure Domain | เทคโนโลยีที่ป้องกัน | Part ที่สอน | เวลา Recovery โดยประมาณ |
|---|---|---|---|
| สาย/Port เดียวขาด (Access↔Distribution) | STP/RSTP Alternate Port หรือ EtherChannel (ถ้า Link นั้นเป็นสมาชิก Port-channel) | Part 5, 6, 33 | RSTP: < 1 วิ (Edge Port ไม่ผ่าน Topology Change) / EtherChannel: ~0 (Sub-second, Member อื่นรับต่อ) |
| Distribution↔Core Link เส้นเดียวขาด | OSPF ECMP (ก่อน Part 34) หรือ MEC Member (หลัง Part 34) | Part 26, 33, 34 | Sub-second (FIB ลบ Path ที่เสียออกจาก ECMP Set ทันที) |
| Distribution Switch ทั้งเครื่องล่ม | HSRP Failover (Priority + Preempt + Track) | Part 18, 35 (Step 341-342) | ~1-10 วิ ขึ้นกับ Timer/Track Delay ที่ตั้งไว้ |
| Core Chassis ทั้งเครื่องล่ม (หลังทำ SVL) | StackWise Virtual Failover | Part 34 | Sub-second ถึงไม่กี่วินาที (ขึ้นกับ SVL Recovery/Dual-Active Detection) |
| Supervisor เดียวในเครื่อง Modular ล่ม | NSF/SSO | Part 26 (Step 256), 35 (Step 345) | ~0 (Non-Stop ถ้าทำสำเร็จสมบูรณ์) |
| Rogue Device ปลอมตัวเป็น Gateway | HSRP MD5 Authentication | Part 35 (Step 341) | ป้องกันไม่ให้เกิดเลย (Preventive ไม่ใช่ Reactive) |
| Black-Hole ที่ Interface ยัง Up แต่ Path จริงเสีย | Multi-Object Tracking (Interface + IP SLA) | Part 32 (Step 314), 35 (Step 342) | ~2-10 วิ ขึ้นกับ `delay down` ที่ตั้งไว้ |

### 347.2 เช็คลิสต์ก่อน Sign-off Design HA ของ Campus

```
[ ] 1. ทุก Access Switch Dual-Homed ไป Distribution 2 ตัว พร้อม STP คุม Loop (Part 5)
[ ] 2. ทุก Distribution↔Core เป็น Routed Uplink คู่ พร้อม OSPF ECMP หรือ MEC (Part 33/34)
[ ] 3. ทุก VLAN มี HSRP VIP ครบ พร้อม Priority/Preempt ตั้งใจให้ Load-Share (Part 18)
[ ] 4. HSRP ทุก Group มี MD5 Authentication (ไม่ใช่ Plaintext หรือไม่มีเลย) (Part 35 Step 341)
[ ] 5. HSRP ทุก Group มี Track Object อย่างน้อย 1 ตัว ครอบคลุมทั้ง Local Link และ End-to-End
       Reachability (Part 35 Step 342)
[ ] 6. Core ที่เป็น Modular Chassis เปิด NSF + SSO ถ้า Hardware รองรับ (Part 35 Step 345)
[ ] 7. Core คู่ที่ทำ StackWise Virtual มี Dual-Active Detection Link แยกจาก SVL Link เสมอ
       (ป้องกัน Split-Brain เต็มรูปแบบ — ดูเพิ่ม Step 348)
[ ] 8. ทดสอบ Failover จริงทุก Layer อย่างน้อยปีละครั้ง ไม่ใช่เชื่อ Design บนกระดาษเฉยๆ (Part 35
       Step 350)
```

---

## Step 348 — HSRP กับ StackWise Virtual: SVL Failure ส่งผลต่อ Tracking อย่างไร

### 348.1 ยืนยันก่อน: DIST Layer ไม่ต้องแก้ HSRP Config เลยหลัง Part 34

คำถามที่พบบ่อยหลังทำ StackWise Virtual (Part 34): **"ในเมื่อ Core กลายเป็น Logical Switch เดียว
แล้ว DIST-SW1–4 ต้องเปลี่ยน HSRP เป็น Group เดียวรวมกันไหม?"** — **คำตอบคือไม่ต้องเลย** เพราะ:

- HSRP ทำงานที่ **SVI ของ DIST-SW1–4** (Layer 3 Boundary ระดับ Distribution) ซึ่ง**ไม่เกี่ยวข้อง
  กับจำนวน Physical Chassis ที่ประกอบเป็น Core เบื้องหลังเลย** — DIST เห็น Core เป็น "ปลายทาง
  Next-Hop เดียว" ทั้งก่อนและหลังทำ SVL (ก่อน SVL เห็นเป็น 2 Next-Hop ผ่าน ECMP, หลัง SVL เห็นเป็น
  1 MEC Interface — แต่ทั้งคู่คือ "จุดเชื่อมต่อขึ้น Core" ไม่ใช่ตัวตัดสินว่า DIST ควร Active/Standby
  ตัวไหน)
- HSRP Election ของ DIST-SW1 vs DIST-SW2 (VLAN 10) **แข่งกันเองระหว่าง DIST-SW1 กับ DIST-SW2**
  ไม่มีความเกี่ยวข้องกับ CORE-SW1/CORE-SW2 เลยแม้แต่ทางใดทางหนึ่ง — Config จาก
  [Step 180, Part 18](part-018-fhrp.md) จึงยัง**ใช้งานได้ทันทีโดยไม่ต้องแก้ไข**

### 348.2 แต่ SVL Failure ส่งผลทางอ้อมต่อ Tracking Object ที่ Step 342 สร้างไว้

แม้ HSRP Config ไม่ต้องเปลี่ยน แต่ **พฤติกรรมของ Track Object ที่ DIST พึ่งพา (Step 342) ได้รับ
ผลกระทบทางอ้อมจากสถานะของ SVL** ในสถานการณ์ที่ซับซ้อนกว่าปกติ — สถานการณ์ที่ต้องเข้าใจให้ชัดคือ
**SVL Dual-Active (Split-Brain ระดับ Core)**:

```
   สถานการณ์ปกติ (SVL ทำงานถูกต้อง)
   CORE-SW1 ◄══ SVL Link ══► CORE-SW2      ทั้งคู่เห็นกันผ่าน SVL ตลอดเวลา
        Active (Control Plane)  Standby (รอ Sync State ตาม Part 34)
   → DIST เห็น Core เป็น 1 Next-Hop เดียว ทำงานถูกต้องสมบูรณ์

   สถานการณ์ผิดปกติ: SVL Link ขาดเอง (แต่ Uplink ไป DIST ทั้งสองฝั่งยังปกติ!)
   CORE-SW1 ✗✗✗ (SVL ขาด) ✗✗✗ CORE-SW2
        คิดว่าตัวเองเป็น Active         คิดว่าตัวเองเป็น Active เหมือนกัน!
        (เพราะไม่เห็น Peer ผ่าน SVL)    (เพราะไม่เห็น Peer ผ่าน SVL เหมือนกัน)

   ❌ Dual-Active: ทั้ง 2 กล่องคิดว่าตัวเองคุม Core ทั้งระบบพร้อมกัน — Duplicate MAC/IP ที่ระดับ
      Core, Routing Table ไม่ตรงกันระหว่าง 2 ฝั่ง (แต่ละฝั่งอาจเห็น Neighbor OSPF คนละชุด)
```

### 348.3 ทำไม Interface Tracking (Track 1) มองไม่เห็นปัญหานี้ แต่ IP SLA (Track 2) เห็น

**Physical Link ระหว่าง DIST กับ CORE-SW1/CORE-SW2 ยังคง Up/Up ปกติทุกเส้นในสถานการณ์ Dual-Active**
(เพราะปัญหาอยู่ที่ SVL Link ภายใน Core เอง ไม่ใช่ Uplink ของ DIST) — **Track 1 (Interface
Line-Protocol) จาก Step 342 จะยังรายงาน "Up" ต่อไปโดยไม่รู้ตัวว่ามีปัญหาเลย** เพราะมันตรวจแค่ Link
ตรงหน้าตัวเอง

แต่ **Track 2 (IP SLA ที่ Ping ข้าม Core ไปยัง Loopback ของ DIST คู่ตัวเอง จาก Step 342.2)** มีโอกาส
สูงที่จะ**เริ่ม Fail หรือ Flap ไม่แน่นอน** เพราะ Routing Table ของ Core ทั้ง 2 ฝั่งไม่ตรงกันแล้ว —
Packet ที่ DIST-SW1 ส่งไปหา DIST-SW2 ผ่าน Core อาจถูก Route ผิดฝั่ง (ไปทาง CORE-SW1 แต่ CORE-SW1 คิด
ว่า Next-Hop ที่ถูกต้องอยู่อีกฝั่ง) จน Drop หรือ Loop ไปมา

> **นี่คือเหตุผลที่แท้จริงที่ Step 342 ออกแบบให้ต้องมี Track 2 (IP SLA End-to-End) เสมอ ไม่ใช่พึ่ง
> Interface Tracking เดี่ยวจาก Part 18 อย่างเดียว** — สถานการณ์ SVL Dual-Active คือตัวอย่างที่ชัด
> ที่สุดของ "ปัญหาที่ Interface Tracking มองไม่เห็นเลย แต่ IP SLA เห็น" ที่กล่าวถึงไว้ตั้งแต่ Step
> 342.1 — แม้ Track 2 เพียงอย่างเดียวจะยังไม่พอ Trigger HSRP Failover ในตารางคำนวณ Step 342.4
> (Decrement 30 อย่างเดียว ไม่พอ) **แต่ก็เพียงพอให้ทีม Network Operation เห็น Alert ผ่าน `show
> track` และ Syslog ได้เร็วกว่าการรอให้ Host Complain** ซึ่งคือคุณค่าของ Defense-in-Depth ตาม
> Step 345.4

### 348.4 การป้องกัน Dual-Active ที่ถูกต้องคือหน้าที่ของ Part 34 ไม่ใช่ Part นี้

ต้องย้ำให้ชัดว่า **การป้องกัน SVL Dual-Active โดยตรง (เช่น Dual-Active Detection Link แยกเฉพาะ,
Fast Hello ระหว่าง Core) เป็นหน้าที่ของการออกแบบ StackWise Virtual ใน Part 34** — Step 342/348
ของ Part นี้เป็นแค่ **ชั้นป้องกันสำรอง (Secondary Safety Net)** ที่ระดับ Distribution เผื่อไว้ในกรณี
ที่การป้องกันหลักที่ Core ล้มเหลวเท่านั้น ตรงตามหลักการ Defense-in-Depth ที่วางไว้ใน Step 345.4

---

## Step 349 — Verification/Troubleshooting HA ขั้นสูง

### 349.1 `show standby brief` ฉบับเต็ม — อ่านคอลัมน์ Authentication และ Tracking

`show standby brief` แบบพื้นฐานจาก [Step 173/179, Part 18](part-018-fhrp.md) ไม่มีคอลัมน์
Authentication/Tracking โดยตรง — ต้องใช้ `show standby <interface>` (Non-brief) เพื่อดูรายละเอียด
เต็มตามที่ Step 341.5 และ 342.5 แสดงไว้ แต่ถ้าต้องการภาพรวมทุก Group พร้อมสถานะ Track ในคำสั่งเดียว
ใช้:

```
DIST-SW1# show standby all
Vlan10 - Group 10 (version 2)
  State is Active
  Authentication MD5, key-string
  Priority 150 (configured 150)
  Track object 1 (interface Te1/1/1), state Up, decrement 30
  Track object 2 (ip sla 1), state Up, decrement 30
Vlan20 - Group 20 (version 2)
  State is Standby
  Authentication MD5, key-string
  ...
```

### 349.2 `show track` — เจาะปัญหาเฉพาะ Track Object

```
DIST-SW1# show track 2
Track 2
  IP SLA 1 reachability
  Reachability is Up
    2 changes, last change 00:12:44
  Delay up 5 secs, down 2 secs
  Tracked by:
    HSRP Vlan10 10
    HSRP Vlan20 20
    HSRP Vlan99 99
```

> **ประโยชน์**: `show track <number>` แสดงด้วยว่า **Track Object ตัวนี้ถูกใช้โดย HSRP Group ไหนบ้าง**
> — สำคัญมากตอน Troubleshoot ว่าทำไมหลาย VLAN Failover พร้อมกันตอน Track ตัวเดียว Down (เพราะใช้
> Track Object ร่วมกันหลาย Group ตามที่ตั้งใจออกแบบไว้ใน Step 350)

### 349.3 ตาราง Root-Cause Checklist สำหรับอาการ HA ที่พบบ่อยที่สุด

| อาการที่เห็น | สาเหตุที่เป็นไปได้ (เรียงจากพบบ่อยสุด) | คำสั่งตรวจสอบ |
|---|---|---|
| DIST ทั้งสองฝั่งอ้างเป็น Active พร้อมกัน (Split-Brain) | (1) MD5 Key ไม่ตรงกัน (Step 341.5) (2) Group Number/VIP ไม่ตรงกัน (Part 18) | `show standby vlan <id>` เทียบ Key/VIP ทั้งสองฝั่ง, ดู Log `%HSRP-4-BADAUTH` |
| Active ไม่ Failover ทั้งที่ Uplink ขาดแล้ว | Decrement ไม่พอ (คำนวณผิดตาม Step 342.4) หรือลืมผูก `standby track` เข้ากับ Group | `show standby vlan <id>` ดู Track Object ว่าผูกครบทุก Group ที่ต้องการหรือไม่ |
| Track Object โชว์ Up แต่ End-to-End Ping จริงไม่ผ่าน | IP SLA Probe เล็งเป้าผิด (เช่น Ping ปลายทางที่ไม่ได้สะท้อนปัญหาจริง) หรือ `frequency`/`timeout` หลวมเกินไปจนตรวจไม่ทัน | `show ip sla statistics` ดู Success/Fail จริง เทียบกับ Ping Manual |
| Failover เกิดถี่ผิดปกติ (Flapping) โดยไม่มี Failure จริง | Decrement รวมน้อยเกินไปจนแค่ Noise เดียวก็ Trigger, หรือ `delay down` ของ Track สั้นเกินไป | `show track <number>` ดู "changes" Counter — ถ้าสูงผิดปกติแสดงว่า Track ไม่เสถียร ต้องปรับ `delay` |
| Core Failure แต่ DIST ไม่มี Alert อะไรเลย (เงียบผิดปกติ) | อาจเป็นสัญญาณดีว่า SVL/MEC (Part 34) ดูดซับ Failure ไว้หมดแล้วจริง (ดู Step 348) — ควร Verify ที่ Core โดยตรงว่า Failover เกิดขึ้นแล้วจริง ไม่ใช่แค่ "ไม่มีอะไรเปลี่ยน" เพราะ Core ยังไม่รู้ตัวว่าเสีย | ตรวจ Log/Status บน Core เอง (StackWise Virtual Status ตาม Part 34) แยกจากการดู DIST เพียงอย่างเดียว |

### 349.4 ลำดับ Troubleshooting HA แบบครบวงจร (ต่อจาก Step 179.3, Part 18)

```
1. show standby all                  → ภาพรวมทุก Group พร้อม Authentication/Track Status
2. show track brief                  → ดูทุก Track Object พร้อมกันว่าตัวไหน Down
3. show ip sla statistics            → ถ้าใช้ IP SLA ตรวจ Root-Cause ที่ต้นทางจริง (ไม่ใช่แค่ Track)
4. show standby vlan <id>             → เจาะ Group เดียว ดู Priority/Decrement คำนวณตรงตามที่ออกแบบไหม
5. show etherchannel summary          → ถ้าสงสัยปัญหาที่ MEC (Part 34) ก่อนที่จะโทษ HSRP ผิดที่
6. show ip ospf neighbor              → ถ้าสงสัย ECMP/Routing (Part 33) เป็นสาเหตุที่แท้จริง
7. debug standby [terse]              → ดู State Transition Real-time (ระวัง CPU เหมือน Part 18)
```

---

## Step 350 — Lab เต็มรูปแบบ: MD5 + Multi-Object Tracking ทุก Group พร้อมทดสอบ CORE-SW1 ล่มทั้งเครื่อง

### 350.1 เป้าหมายของ Lab นี้

เพิ่ม **MD5 Authentication** และ **Multi-Object Tracking (Interface + IP SLA)** เข้าไปในทุก HSRP
Group ที่มีอยู่แล้วจาก [Step 180, Part 18](part-018-fhrp.md) (VLAN 10, 20, 30, 40, 99 ทั้ง 2 บล็อก
บน DIST-SW1–4) แล้วทดสอบ **Failover แบบครบวงจรทุก Layer พร้อมกัน** ด้วยการจำลอง **CORE-SW1 ทั้ง
เครื่องดับสนิท** (ไม่ใช่แค่ Interface เดียว) หลังจากที่ Core ทำ StackWise Virtual แล้วตาม
[Part 34](part-034-stackwise-vss-mec.md)

### 350.2 Key และ Track Target ที่ใช้ในแต่ละ Group

| HSRP Group | Switch คู่ | MD5 Key | IP SLA Ping ไปยัง (Loopback ของคู่ตัวเอง) |
|---|---|---|---|
| 10 (SALES) | DIST-SW1 ↔ DIST-SW2 | `C1sco-HSRP-V10-P35!` | DIST-SW1→1.1.1.12, DIST-SW2→1.1.1.11 |
| 20 (VOICE) | DIST-SW1 ↔ DIST-SW2 | `C1sco-HSRP-V20-P35!` | DIST-SW1→1.1.1.12, DIST-SW2→1.1.1.11 |
| 99 (MGMT บล็อก 1) | DIST-SW1 ↔ DIST-SW2 | `C1sco-HSRP-V99-B1-P35!` | DIST-SW1→1.1.1.12, DIST-SW2→1.1.1.11 |
| 30 (SERVERS) | DIST-SW3 ↔ DIST-SW4 | `C1sco-HSRP-V30-P35!` | DIST-SW3→1.1.1.14, DIST-SW4→1.1.1.13 |
| 40 (WIFI) | DIST-SW3 ↔ DIST-SW4 | `C1sco-HSRP-V40-P35!` | DIST-SW3→1.1.1.14, DIST-SW4→1.1.1.13 |
| 99 (MGMT บล็อก 2) | DIST-SW3 ↔ DIST-SW4 | `C1sco-HSRP-V99-B2-P35!` | DIST-SW3→1.1.1.14, DIST-SW4→1.1.1.13 |

> **Track Object และ IP SLA เป็นของ Switch แต่ละตัว ใช้ร่วมกันได้ทุก Group บนตัวเดียวกัน** — ไม่ต้อง
> สร้าง Track/IP SLA แยกต่อ VLAN เพราะ Track 1 (Member Link) และ Track 2 (IP SLA ไปยัง Loopback คู่
> ตัวเอง) มีความหมายเดียวกันไม่ว่าจะมองจาก VLAN ไหนบนอุปกรณ์ตัวเดียวกัน (ตามที่ `show track`
> Step 349.2 แสดงว่า Track ตัวเดียวถูกใช้โดยหลาย HSRP Group ได้)

### 350.3 คอนฟิกส่วนที่เพิ่มเข้ามา — DIST-SW1 (ส่วนที่เหมือน Part 18 Step 180.2 ไม่คอนฟิกซ้ำ)

```
! ---------- Global: Track Object + IP SLA (ใช้ร่วมกันทุก HSRP Group บนเครื่องนี้) ----------
DIST-SW1(config)# track 1 interface TenGigabitEthernet1/1/1 line-protocol
DIST-SW1(config-track)# exit
!
DIST-SW1(config)# ip sla 1
DIST-SW1(config-ip-sla)# icmp-echo 1.1.1.12 source-interface Loopback0
DIST-SW1(config-ip-sla-echo)# frequency 5
DIST-SW1(config-ip-sla-echo)# timeout 1000
DIST-SW1(config-ip-sla-echo)# exit
DIST-SW1(config)# ip sla schedule 1 life forever start-time now
!
DIST-SW1(config)# track 2 ip sla 1 reachability
DIST-SW1(config-track)# delay down 2 up 5
DIST-SW1(config-track)# exit
!
! ---------- VLAN 10 — เพิ่ม Authentication + เปลี่ยนจาก track เดี่ยว (Part 18) เป็น 2 Track ----------
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# no standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# standby 10 authentication md5 key-string 0 C1sco-HSRP-V10-P35!
DIST-SW1(config-if)# standby 10 track 1 decrement 30
DIST-SW1(config-if)# standby 10 track 2 decrement 30
DIST-SW1(config-if)# exit
!
! ---------- VLAN 20 ----------
DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# no standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# standby 20 authentication md5 key-string 0 C1sco-HSRP-V20-P35!
DIST-SW1(config-if)# standby 20 track 1 decrement 30
DIST-SW1(config-if)# standby 20 track 2 decrement 30
DIST-SW1(config-if)# exit
!
! ---------- VLAN 99 (MGMT บล็อก 1) ----------
DIST-SW1(config)# interface vlan 99
DIST-SW1(config-if)# no standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# standby 99 authentication md5 key-string 0 C1sco-HSRP-V99-B1-P35!
DIST-SW1(config-if)# standby 99 track 1 decrement 30
DIST-SW1(config-if)# standby 99 track 2 decrement 30
DIST-SW1(config-if)# exit
```

> **หมายเหตุ Interface Name**: `TenGigabitEthernet1/1/1` ในที่นี้คือ **Member Link ตัวหนึ่งของ MEC
> Port-channel1** ที่ Part 34 สร้างขึ้น (แต่เดิมคือ Uplink ตรงไป CORE-SW1 ก่อนทำ StackWise Virtual)
> — Port-channel1 เองไม่ต้องถูก Track เพราะ Port-channel จะยัง Up ต่อไปตราบใดที่มี Member เหลืออยู่
> อย่างน้อย 1 เส้น สิ่งที่มีประโยชน์ต่อ HSRP คือ Track "แต่ละ Member" เพื่อรู้ว่า Capacity เหลือครึ่ง
> เดียวหรือไม่ (สัญญาณ Step 342 ไม่ใช่สัญญาณ Failover เดี่ยว)

### 350.4 คอนฟิกส่วนที่เพิ่มเข้ามา — DIST-SW2, DIST-SW3, DIST-SW4 (Pattern เดียวกัน)

```
! ========== DIST-SW2 ==========
DIST-SW2(config)# track 1 interface TenGigabitEthernet1/1/1 line-protocol
DIST-SW2(config)# ip sla 1
DIST-SW2(config-ip-sla)# icmp-echo 1.1.1.11 source-interface Loopback0
DIST-SW2(config-ip-sla-echo)# frequency 5
DIST-SW2(config-ip-sla-echo)# timeout 1000
DIST-SW2(config-ip-sla-echo)# exit
DIST-SW2(config)# ip sla schedule 1 life forever start-time now
DIST-SW2(config)# track 2 ip sla 1 reachability
DIST-SW2(config-track)# delay down 2 up 5
DIST-SW2(config-track)# exit
!
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# no standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# standby 10 authentication md5 key-string 0 C1sco-HSRP-V10-P35!
DIST-SW2(config-if)# standby 10 track 1 decrement 30
DIST-SW2(config-if)# standby 10 track 2 decrement 30
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface vlan 20
DIST-SW2(config-if)# no standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# standby 20 authentication md5 key-string 0 C1sco-HSRP-V20-P35!
DIST-SW2(config-if)# standby 20 track 1 decrement 30
DIST-SW2(config-if)# standby 20 track 2 decrement 30
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface vlan 99
DIST-SW2(config-if)# no standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# standby 99 authentication md5 key-string 0 C1sco-HSRP-V99-B1-P35!
DIST-SW2(config-if)# standby 99 track 1 decrement 30
DIST-SW2(config-if)# standby 99 track 2 decrement 30
DIST-SW2(config-if)# exit

! ========== DIST-SW3 ==========
DIST-SW3(config)# track 1 interface TenGigabitEthernet1/1/1 line-protocol
DIST-SW3(config)# ip sla 1
DIST-SW3(config-ip-sla)# icmp-echo 1.1.1.14 source-interface Loopback0
DIST-SW3(config-ip-sla-echo)# frequency 5
DIST-SW3(config-ip-sla-echo)# timeout 1000
DIST-SW3(config-ip-sla-echo)# exit
DIST-SW3(config)# ip sla schedule 1 life forever start-time now
DIST-SW3(config)# track 2 ip sla 1 reachability
DIST-SW3(config-track)# delay down 2 up 5
DIST-SW3(config-track)# exit
!
DIST-SW3(config)# interface vlan 30
DIST-SW3(config-if)# no standby 30 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# standby 30 authentication md5 key-string 0 C1sco-HSRP-V30-P35!
DIST-SW3(config-if)# standby 30 track 1 decrement 30
DIST-SW3(config-if)# standby 30 track 2 decrement 30
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface vlan 40
DIST-SW3(config-if)# no standby 40 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# standby 40 authentication md5 key-string 0 C1sco-HSRP-V40-P35!
DIST-SW3(config-if)# standby 40 track 1 decrement 30
DIST-SW3(config-if)# standby 40 track 2 decrement 30
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface vlan 99
DIST-SW3(config-if)# no standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# standby 99 authentication md5 key-string 0 C1sco-HSRP-V99-B2-P35!
DIST-SW3(config-if)# standby 99 track 1 decrement 30
DIST-SW3(config-if)# standby 99 track 2 decrement 30
DIST-SW3(config-if)# exit

! ========== DIST-SW4 ==========
DIST-SW4(config)# track 1 interface TenGigabitEthernet1/1/1 line-protocol
DIST-SW4(config)# ip sla 1
DIST-SW4(config-ip-sla)# icmp-echo 1.1.1.13 source-interface Loopback0
DIST-SW4(config-ip-sla-echo)# frequency 5
DIST-SW4(config-ip-sla-echo)# timeout 1000
DIST-SW4(config-ip-sla-echo)# exit
DIST-SW4(config)# ip sla schedule 1 life forever start-time now
DIST-SW4(config)# track 2 ip sla 1 reachability
DIST-SW4(config-track)# delay down 2 up 5
DIST-SW4(config-track)# exit
!
DIST-SW4(config)# interface vlan 30
DIST-SW4(config-if)# no standby 30 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# standby 30 authentication md5 key-string 0 C1sco-HSRP-V30-P35!
DIST-SW4(config-if)# standby 30 track 1 decrement 30
DIST-SW4(config-if)# standby 30 track 2 decrement 30
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface vlan 40
DIST-SW4(config-if)# no standby 40 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# standby 40 authentication md5 key-string 0 C1sco-HSRP-V40-P35!
DIST-SW4(config-if)# standby 40 track 1 decrement 30
DIST-SW4(config-if)# standby 40 track 2 decrement 30
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface vlan 99
DIST-SW4(config-if)# no standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# standby 99 authentication md5 key-string 0 C1sco-HSRP-V99-B2-P35!
DIST-SW4(config-if)# standby 99 track 1 decrement 30
DIST-SW4(config-if)# standby 99 track 2 decrement 30
DIST-SW4(config-if)# exit
```

### 350.5 Verify สถานะก่อนทดสอบ — ทุกอย่างต้องปกติสมบูรณ์ก่อนเริ่ม

```
DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           10.10.10.3      10.10.10.1
Vl20        20   100 P Standby  10.10.20.3       local           10.10.20.1
Vl99        99   150 P Active   local           10.10.99.3      10.10.99.1

DIST-SW1# show standby vlan 10 | include Authentication|Track|Priority
  Authentication MD5, key-string
  Priority 150 (configured 150)
  Track object 1 (interface Te1/1/1), state Up, decrement 30
  Track object 2 (ip sla 1), state Up, decrement 30

DIST-SW1# show ip sla statistics 1 | include Latest
  Latest RTT: 1 milliseconds
  Latest operation return code: OK
```

### 350.6 Failover Test — จำลอง CORE-SW1 ทั้งเครื่องดับสนิทหลังทำ StackWise Virtual

**สถานการณ์**: หลัง Part 34, CORE-SW1 และ CORE-SW2 ทำงานเป็น StackWise Virtual Pair โดย CORE-SW1
รับบท **SVL Active** — ทดสอบด้วยการ**ตัดไฟ CORE-SW1 ทั้งเครื่อง** (ไม่ใช่แค่ Shutdown Interface
เดียวแบบ Part 18 Step 174/180 อีกต่อไป) แล้วไล่ดู HA ทุก Layer ตอบสนองตามลำดับ

```
=== เวลา T+0s: ตัดไฟ CORE-SW1 ทั้งเครื่อง (จำลอง Chassis Failure) ===

--- Layer 1: StackWise Virtual (Part 34) — ตอบสนองก่อนใคร ---
CORE-SW2#
%STACKWISE_VIRTUAL-6-SWO: StackWise Virtual switchover, switch is now active
%STACKWISE_VIRTUAL-2-DUAL_ACTIVE_RECOVERED: CORE-SW2 กลายเป็น Active ของ Logical Switch แทน
   ▲ CORE-SW2 เข้ารับหน้าที่ Active ของ Logical Switch ทั้งระบบภายในไม่กี่วินาที
     (SVI/Route Table ทั้งหมดที่ Sync ไว้ล่วงหน้าจาก Part 34 พร้อมใช้ทันที)

--- Layer 2: MEC ที่ DIST-SW1-4 มองเห็น (Part 34) ---
DIST-SW1#
%LINK-3-UPDOWN: Interface TenGigabitEthernet1/1/1, changed state to down
   (Member Link เส้นที่ต่อไปทาง CORE-SW1 เดิม หลุดตามเครื่องที่ดับ)
%LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel1, changed state to up
   (Port-channel1 ยัง "up" ต่อเนื่อง เพราะ Member อีกเส้นที่ต่อไปทาง CORE-SW2 ยังทำงานปกติ)
   ▲ MEC ดูดซับ Failure ไว้เต็มๆ — DIST ไม่เสีย Uplink ไป Core เลยแม้แต่วินาทีเดียว

--- Layer 3: HSRP Tracking บน DIST-SW1 (Part 35 Step 342) ---
DIST-SW1#
%TRACK-6-STATE: 1 interface Te1/1/1 line-protocol Up -> Down
   (Track 1 เห็น Member Link เส้นนั้น Down ตามจริง)
DIST-SW1# show ip sla statistics 1 | include Latest
  Latest operation return code: OK      <- Track 2 (IP SLA ไป DIST-SW2 ข้าม Core) ยังผ่านปกติ
                                            เพราะ SVL Switchover ของ Core (Layer 1) เร็วมากจนไม่มี
                                            ผลกระทบต่อ Path จริงที่ IP SLA Probe อยู่เลย
DIST-SW1# show standby vlan 10 | include Priority
  Priority 120 (configured 150)          <- ลดแค่ 30 (Track 1 อย่างเดียว) ไม่ถึง Threshold Failover

--- ผลลัพธ์สุทธิ: HSRP ไม่ Failover เลย ---
DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   120 P Active   local           10.10.10.3      10.10.10.1
                  ▲ ยังเป็น Active! (120 > 100 ของ DIST-SW2) แม้ CORE-SW1 ทั้งเครื่องดับไปแล้วจริง
```

**สรุปผล Trace HA ข้ามทุก Layer**: STP (Part 5/33) **ไม่มีผลเกี่ยวข้องเลย** เพราะ Core↔Distribution
เป็น Routed ทั้งหมดตั้งแต่ Part 33 — StackWise Virtual (Part 34) รับ Failure ไว้ที่ต้นตอก่อนเป็น
อันดับแรก, MEC (Part 34) ดูดซับผลกระทบที่ระดับ Physical/Link ไม่ให้ขึ้นไปถึง Routing เลย, OSPF
ECMP เดิม (Part 33) ไม่ต้องคำนวณ Path ใหม่เพราะ MEC เก็บเป็น Next-Hop เดียวไว้ตั้งแต่ต้น และ **HSRP
(Part 18/35) ไม่จำเป็นต้อง Failover เลยด้วยซ้ำ** เพราะ Layer ที่อยู่ต่ำกว่าทำงานได้สมบูรณ์ — Host
ปลายทางไม่รู้สึกอะไรเลยตลอดกระบวนการทั้งหมด ตรงตามเป้าหมาย Defense-in-Depth จาก Step 345.4 ทุก
ประการ

### 350.7 Failover Test ที่ 2 — ทดสอบ Threshold จริงด้วยการตัด MEC ทั้งบันเดิล

เพื่อพิสูจน์ว่า Multi-Object Tracking ยังทำงานได้จริงเมื่อ Failure รุนแรงกว่าเดิม ให้ทดสอบต่อด้วย
การ Shutdown **ทั้ง 2 Member ของ Port-channel1** (จำลองว่า DIST-SW1 เสีย Uplink ไป Core ทั้งหมดจริง
ไม่ใช่แค่ครึ่งเดียว):

```
DIST-SW1(config)# interface range TenGigabitEthernet1/1/1 - 2
DIST-SW1(config-if-range)# shutdown

DIST-SW1#
%TRACK-6-STATE: 1 interface Te1/1/1 line-protocol Up -> Down
%TRACK-6-STATE: 2 ip sla 1 reachability Up -> Down
   (Track 2 ล่มตามจริง เพราะไม่มีเส้นทางไป Core เหลือแม้แต่เส้นเดียวแล้ว)

DIST-SW1# show standby vlan 10 | include Priority
  Priority 90 (configured 150)           <- 150 - 30 - 30 = 90 ตามตาราง Step 342.4

%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Active -> Speak
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Speak -> Standby

DIST-SW2#
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active

DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   90  P Standby  10.10.10.3      local           10.10.10.1

DIST-SW2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Active   local           10.10.10.2      10.10.10.1
```

Failover เกิดขึ้นตรงตามที่คำนวณไว้ทุกประการ — พิสูจน์ว่า Design ของ Step 342 **ทนต่อ Failure
บางส่วน (Single Track) แต่ Failover ทันทีเมื่อ Failure รุนแรงจริง (ทั้ง 2 Track พร้อมกัน)** ตามที่
ตั้งใจออกแบบไว้ตั้งแต่ต้น

---

## แบบฝึกหัดทวนความเข้าใจ Part 35

1. เพราะเหตุใด Plaintext Authentication ของ HSRP (`standby <group> authentication text <string>`)
   จึงถือว่า**ไม่ปลอดภัยเพียงพอ**สำหรับ Production แม้จะปลอดภัยกว่าการไม่มี Authentication เลยก็ตาม?
2. ในตัวอย่าง Multi-Object Tracking ของ Step 342 ถ้า Track 1 (Decrement 30) Down อย่างเดียวโดย
   Track 2 ยัง Up — Priority ของ DIST-SW1 (ปกติ 150) จะเหลือเท่าใด และจะเกิด Failover หรือไม่ (เทียบ
   กับ DIST-SW2 Priority 100)?
3. เพราะเหตุใด SVL Dual-Active (Split-Brain ที่ Core หลังทำ StackWise Virtual) จึง**ไม่ถูกตรวจจับ**
   โดย Interface Tracking (Track 1) แต่มีโอกาสถูกตรวจจับได้โดย IP SLA Reachability Tracking
   (Track 2)?
4. ในสถานการณ์ CORE-SW1 ทั้งเครื่องดับสนิท (Step 350.6) เพราะเหตุใด HSRP บน DIST-SW1–4 จึง**ไม่
   Failover เลย** ทั้งที่ Core ทั้งเครื่องล่มไปจริง?
5. Anycast Gateway (Step 344) ต่างจาก HSRP อย่างไรในเรื่อง "จำนวน Gateway ที่รองรับได้ดีต่อ VLAN"
   และเพราะเหตุใด Fabric Design (SD-Access/VXLAN EVPN) จึงเลือกใช้ Anycast Gateway แทน HSRP?

**เฉลย:**

1. เพราะ Key ของ Plaintext Authentication **ถูกส่งเป็น Cleartext อยู่ใน Hello Packet ทุกใบ** — ถ้า
   ผู้บุกรุก Capture Packet บน Segment เดียวกันได้ (เช่นด้วย Wireshark) จะอ่าน Key ออกมาได้ทันที
   แล้วปลอม Hello ต่อได้เหมือนไม่มี Authentication เลย ต่างจาก MD5 ที่ส่งแค่ Hash ไม่ส่ง Key จริง
   ผ่าน Network เลยแม้แต่ครั้งเดียว
2. Priority = 150 − 30 = **120** — **ไม่ Failover** เพราะ 120 ยังสูงกว่า Priority ของ DIST-SW2
   (100) อยู่ — ออกแบบให้ Track เดี่ยวไม่พอ Trigger Failover เพื่อป้องกัน False Positive จาก
   Probe ตัวเดียวที่ไม่เสถียร
3. เพราะ SVL Dual-Active ทำให้ **Physical Link ระหว่าง DIST กับ Core ยังคง Up/Up ปกติทุกเส้น**
   (ปัญหาอยู่ที่ SVL Link ภายใน Core เอง ไม่ใช่ Uplink ของ DIST) — Interface Tracking ตรวจแค่ Link
   ตรงหน้าตัวเองจึงมองไม่เห็นปัญหานี้เลย ในขณะที่ IP SLA ที่ Ping ข้าม Core ไปยังปลายทางจริงมีโอกาส
   ล้มเหลวหรือ Flap เพราะ Routing Table ของ Core ทั้ง 2 ฝั่งไม่ตรงกันแล้ว
4. เพราะ **StackWise Virtual (Part 34) และ MEC ดูดซับ Failure ไว้ได้สมบูรณ์ก่อนที่จะขึ้นไปถึงระดับ
   HSRP** — CORE-SW2 เข้ารับหน้าที่ Active ของ Logical Switch ทันที และ MEC ที่ DIST เห็นก็ยัง Up
   ต่อเนื่อง (เพราะเหลือ Member อีกเส้นที่ต่อไปทาง CORE-SW2 อยู่) ทำให้ Track 1 ลดแค่ 30 (จาก Member
   เส้นเดียว) และ Track 2 ยัง Up สนิท — Priority เหลือ 120 ยังสูงกว่า Standby อยู่ ไม่ถึง Threshold
   ที่จะ Failover เลย
5. HSRP รองรับ Gateway ที่ Forward Traffic จริงได้ดีแค่ 2 ตัวต่อ Group (GLBP ทำได้ถึง 4 AVF) ในขณะ
   ที่ Anycast Gateway **ไม่จำกัดจำนวน Leaf Switch เลย** เพราะไม่มี Election/Hello ระหว่างกัน — ทุก
   Leaf Config SVI IP และ Virtual MAC เดียวกันแบบ Static แล้วตอบ ARP ในพื้นที่ตัวเองทันที — Fabric
   Design ที่มี Leaf Switch หลักสิบตัวต่อ VLAN เดียวจึงเลือก Anycast Gateway เพราะ HSRP/GLBP ไม่ถูก
   ออกแบบมาให้ Scale ถึงระดับนั้น

---

## สรุป Part 35

Part นี้ยกระดับ HSRP จากพื้นฐานที่เรียนไว้ใน Part 18 ให้ลึกถึงระดับ Enterprise/CCIE จริง: **MD5
Authentication** ป้องกัน Rogue HSRP Speaker และ HSRP Hijacking, **Multi-Object Tracking** ที่รวม
Interface Tracking กับ IP SLA Reachability เข้าด้วยกันพร้อม Priority Math ที่ป้องกัน False
Positive ในขณะที่ยังจับ Black-Hole ที่ Interface Tracking เดี่ยวมองไม่เห็น, ข้อจำกัดการ Scale ของ
HSRPv2 ในองค์กรขนาดใหญ่และแนวทางบรรเทา, แนวคิด **Anycast Gateway** ที่ Fabric สมัยใหม่ใช้แทน HSRP
เมื่อ Scale เกินขอบเขตที่ FHRP แบบ Election รองรับได้ (Preview Part 84-85), **NSF/SSO** สำหรับ
Supervisor Redundancy ภายในเครื่องเดียวและ **Redundancy Group** สำหรับ Cross-Device NSF ระดับ CCIE,
ตารางเช็คลิสต์ Campus HA Design ที่รวมทุก Layer เข้าด้วยกัน, การยืนยันว่า HSRP ไม่ต้องเปลี่ยน Config
เลยหลังทำ StackWise Virtual (Part 34) แต่ต้องเข้าใจผลกระทบทางอ้อมของ SVL Dual-Active ต่อ Tracking
Object และปิดท้ายด้วย Lab เต็มรูปแบบที่พิสูจน์ว่า **HA ที่ออกแบบเป็นชั้น (Defense-in-Depth) ทำให้
Core Chassis ทั้งเครื่องล่มได้โดย Host ไม่รู้สึกอะไรเลยแม้แต่วินาทีเดียว** — และยัง Failover ได้ถูก
ต้องเมื่อ Failure รุนแรงจริงจนชั้นป้องกันด้านล่างเอาไม่อยู่

Campus Network ของหลักสูตรนี้ตอนนี้มี HA ที่ครบทุก Layer ตั้งแต่ Physical Link จนถึง FHRP แล้ว
Part ถัดไปจะเปลี่ยนโฟกัสจาก Wired Campus ไปสู่ **Wireless Architecture** อย่างเต็มรูปแบบ เริ่มจาก
การออกแบบ WLC/AP Deployment Model, CAPWAP Tunneling, และ RF Design ที่ลึกกว่าพื้นฐานที่ Part 20
เคยแนะนำไว้

**ไปต่อ:** [Part 36 — Wireless Architecture Deep Dive →](part-036-wireless-architecture-deep-dive.md)
