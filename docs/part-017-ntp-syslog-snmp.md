# Part 17 — NTP, Syslog, SNMP
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 161–170 จาก 1000**

> ต่อจาก [Part 16 — DHCP & DNS](part-016-dhcp-dns.md) ที่ทำให้อุปกรณ์ปลายทางได้ IP Address และ
> แปลงชื่อโดเมนอัตโนมัติ Part นี้เราจะหันมาดูฝั่ง **Network Management** — สามขาหลักที่ทุก
> Enterprise Network ต้องมีก่อนจะเรียกตัวเองว่า "production-ready" คือ **NTP** (เวลาที่ตรงกัน),
> **Syslog** (log ที่เชื่อถือได้), และ **SNMP** (การ monitor แบบรวมศูนย์) เราจะสร้างเซิร์ฟเวอร์
> จำลองตัวใหม่ชื่อ **NOC-SRV** (10.10.99.50 ใน VLAN 99 MGMT) ให้ทำหน้าที่เป็น NTP stratum
> reference, Syslog collector, และ SNMP/NMS (Network Management System) แบบรวมศูนย์ให้กับทุก
> อุปกรณ์ใน Lab Topology

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 161 | ทำไม Time Synchronization สำคัญ + แนวคิด NTP Hierarchy (Stratum 0-15) |
| 162 | Configuring NTP Client บนทุกอุปกรณ์ (ntp server, clock timezone) |
| 163 | NTP Authentication และคำสั่ง Verify (show ntp status/associations) |
| 164 | Syslog Fundamentals — 8 Severity Level และ Message Format |
| 165 | Configuring Syslog ส่ง log ไปยัง NOC-SRV |
| 166 | Local Logging — Buffer vs Console vs Terminal Monitor |
| 167 | SNMP Fundamentals — SNMPv1/v2c vs SNMPv3 |
| 168 | Configuring SNMPv2c (legacy) และ SNMPv3 (Production) |
| 169 | SNMP Traps vs Polling + แนวคิด MIB/OID |
| 170 | Lab: NTP + Syslog + SNMPv3 แบบสมบูรณ์บนทุกอุปกรณ์ |

---

## Step 161 — ทำไม Time Synchronization สำคัญ + NTP Hierarchy

### ปัญหาที่เกิดขึ้นถ้าเวลาของอุปกรณ์ไม่ตรงกัน

ลองนึกภาพเหตุการณ์จริง: มีปัญหา network ระหว่าง 09:14 - 09:17 คุณต้องไล่ log จาก 5 อุปกรณ์
(CORE-SW1, CORE-SW2, DIST-SW1, WAN-EDGE-1, Firewall) เพื่อหาสาเหตุ แต่ clock ของแต่ละอุปกรณ์
เดินไม่ตรงกัน — บางตัวเร็วไป 3 นาที บางตัวช้าไป 90 วินาที ผลคือ **timeline ของ log ไม่สามารถ
เรียงลำดับเหตุการณ์ได้ถูกต้อง** จะบอกไม่ได้ว่า BGP flap เกิดก่อนหรือหลัง STP topology change

นี่คือเหตุผลที่ **Network Time Protocol (NTP)** เป็นหนึ่งใน "must-have" ของทุก Production Network
ไม่ใช่แค่ "nice to have"

### ผลกระทบเมื่อเวลาไม่ตรงกัน (สรุปเป็นหมวดที่ต้องรู้)

| ผลกระทบ | รายละเอียด |
|---|---|
| **Log Correlation ผิดพลาด** | ไม่สามารถเรียง timeline เหตุการณ์ข้ามอุปกรณ์ได้ — SIEM/Syslog correlation จะพังทั้งระบบ |
| **Certificate Validation ล้มเหลว** | X.509 Certificate มี Validity Period (Not Before / Not After) — ถ้า clock เพี้ยนไปนอกช่วงนี้ TLS/SSH/HTTPS handshake จะ**ล้มเหลว** แม้ certificate ยังไม่หมดอายุจริง |
| **AAA/RADIUS Token หมดอายุผิดเวลา** | TACACS+/RADIUS, OTP (Time-based One-Time Password), Kerberos ใช้ time window ในการตรวจสอบ ถ้าเวลาเพี้ยนเกิน threshold จะ login ไม่ได้ |
| **Debugging Multi-Device ยากขึ้นมาก** | เวลา troubleshoot ปัญหาที่เกี่ยวข้องหลายอุปกรณ์ (เช่น routing flap, STP reconvergence) ต้องใช้ timestamp เทียบ event ข้ามอุปกรณ์เสมอ |
| **Compliance/Audit ล้มเหลว** | มาตรฐานความปลอดภัย (PCI-DSS, ISO 27001) กำหนดให้ต้องมี accurate timestamp บน log ทุกฉบับเพื่อการ audit |
| **Scheduled Task ผิดเวลา** | `kron`, EEM Applet ที่ trigger ตามเวลา, Certificate renewal job จะทำงานผิดเวลาไปด้วย |

### NTP Hierarchy: แนวคิด Stratum

NTP ออกแบบเป็นโครงสร้างลำดับชั้น (Hierarchical) เรียกว่า **Stratum** เพื่อกระจายภาระและ
ลด error สะสมจากการซิงค์เวลาหลายชั้น

```
Stratum 0   ┌─────────────────────────────────────────┐
(Reference) │ GPS Clock / Atomic Clock / Radio Clock    │  <- นาฬิกาอ้างอิงที่แม่นยำที่สุด
            │ (ไม่ใช่อุปกรณ์เครือข่าย ต่อ direct เท่านั้น) │     ไม่มี network device คุยกับมันตรง
            └───────────────────┬───────────────────────┘
                                 │ (ต่อสายตรง เช่น GPS antenna, PPS)
Stratum 1   ┌───────────────────┴───────────────────────┐
(Primary)   │  NTP Server ที่ต่อกับ Stratum 0 โดยตรง       │  <- เช่น pool.ntp.org บางตัว,
            │  เช่น NOC-SRV (ถ้าต่อ GPS receiver จริง)     │     time.google.com, time.nist.gov
            └───────────────────┬───────────────────────┘
                                 │ (คุยผ่าน Network - NTP packet)
Stratum 2   ┌───────────────────┴───────────────────────┐
(Secondary) │  NOC-SRV (sync จาก Internet NTP Pool)       │  <- ในหลักสูตรนี้ NOC-SRV คือ Stratum 2
            └───────────────────┬───────────────────────┘
                                 │
Stratum 3   ┌───────────────────┴───────────────────────┐
            │ CORE-SW1, CORE-SW2, DIST-SW1-4,             │  <- ทุกอุปกรณ์ Lab sync จาก NOC-SRV
            │ WAN-EDGE-1, WAN-EDGE-2, ACCESS-SW1-4         │     กลายเป็น Stratum 3
            └───────────────────┬───────────────────────┘
                                 │
Stratum 4+  ┌───────────────────┴───────────────────────┐
            │ End devices (PC, Server, AP) ที่ sync ต่อจาก  │
            │ Access Switch หรือ Domain Controller         │
            └───────────────────────────────────────────┘
```

| Stratum | ความหมาย |
|---|---|
| 0 | นาฬิกาอ้างอิงจริง (GPS, Atomic, Radio) — ไม่ถือเป็น "NTP Server" ในทางเทคนิค |
| 1 | อุปกรณ์ที่ต่อกับ Stratum 0 โดยตรง (ผ่าน hardware ไม่ใช่ network) |
| 2-14 | อุปกรณ์ที่ sync ผ่าน Network จากอุปกรณ์ stratum สูงกว่าถัดไปทีละชั้น (stratum เพิ่มขึ้น +1 ทุกครั้งที่ผ่าน hop) |
| 15 | ค่าสูงสุดที่ยังถือว่า valid |
| 16 | หมายถึง **unsynchronized** (ยังไม่ sync กับใครเลย) — Cisco แสดงเป็น stratum 16 เสมอก่อน sync สำเร็จ |

> **กฎสำคัญ**: เลข stratum **น้อย = ใกล้ต้นตอ = น่าเชื่อถือกว่า** ไม่ได้แปลว่าเวลาแม่นกว่าเสมอไป
> (เพราะ latency บน network ก็มีผล) แต่โดยทั่วไปยิ่งใกล้ stratum 0 ยิ่งดี ในหลักสูตรนี้เราจะให้
> **NOC-SRV เป็น Stratum 2** (sync จาก NTP Pool บน Internet ผ่าน WAN-EDGE) แล้วให้ทุกอุปกรณ์ใน
> Lab ชี้มาที่ NOC-SRV เพียงจุดเดียว (Single Source of Time) ทำให้อุปกรณ์ทั้งหมดกลายเป็น
> **Stratum 3** และมีเวลาตรงกันทั้งองค์กร

### NTP ใช้ Protocol ใด

NTP ทำงานบน **UDP Port 123** และใช้ Algorithm ที่ซับซ้อน (คำนวณ offset, delay, jitter, dispersion)
เพื่อปรับเวลาแบบ**นุ่มนวล** (gradual adjustment) ไม่กระโดดเวลาแบบทันที ป้องกันปัญหา
timestamp ย้อนกลับ (time going backward) ที่จะทำให้ log/certificate สับสน

---

## Step 162 — Configuring NTP Client

### แนวคิดการ Deploy ในหลักสูตรนี้

เราจะให้ **NOC-SRV (10.10.99.50)** เป็น NTP Source เดียวสำหรับทุกอุปกรณ์ใน Lab (NOC-SRV เองจะ
sync ไปที่ NTP Pool บน Internet ผ่าน WAN-EDGE-1/2 อีกที ซึ่งเป็นสถาปัตยกรรมที่ตรงกับความเป็นจริง
ขององค์กรที่ไม่ต้องการให้อุปกรณ์ทุกตัวออก Internet โดยตรงเพื่อขอเวลา)

```
                    ┌───────────────────────┐
                    │  Internet NTP Pool      │
                    │  (Stratum 1 เทียม)      │
                    └────────────┬───────────┘
                                  │ ผ่าน WAN-EDGE-1/2 (NAT/Internet Access - Part 15)
                    ┌────────────┴───────────┐
                    │  NOC-SRV 10.10.99.50    │  <- Stratum 2 (NTP Server ของ Lab ทั้งหมด)
                    │  VLAN 99 (MGMT)          │     รวม Syslog Collector + SNMP/NMS ไว้ที่นี่ด้วย
                    └────────────┬───────────┘
        ┌─────────────┬──────────┴─────────┬─────────────┐
        ▼             ▼                    ▼             ▼
   CORE-SW1/2     DIST-SW1-4          WAN-EDGE-1/2    ACCESS-SW1-4
   (Stratum 3)    (Stratum 3)         (Stratum 3)     (Stratum 3)
```

### คำสั่งหลักที่ต้องรู้

| คำสั่ง | ความหมาย |
|---|---|
| `ntp server <ip>` | กำหนด NTP Server ที่จะ sync เวลาด้วย (ใส่ได้หลายตัวเพื่อ redundancy) |
| `ntp server <ip> prefer` | บอกให้ device "ชอบ" server ตัวนี้มากกว่าตัวอื่นเมื่อคุณภาพใกล้เคียงกัน |
| `ntp source <interface>` | กำหนด source interface ที่ใช้ส่ง NTP packet (แนะนำใช้ Loopback0 หรือ SVI VLAN99 เพื่อความ stable) |
| `ntp update-calendar` | ให้ NTP sync ค่าไปยัง Hardware Clock (calendar) ด้วย ไม่ใช่แค่ software clock ในหน่วยความจำ (สำคัญกับอุปกรณ์ที่มี RTC เช่น Router) |
| `clock timezone <ชื่อ> <offset-hour> <offset-min>` | กำหนด Time Zone ที่แสดงผล (NTP ภายในคำนวณเป็น UTC เสมอ, `clock timezone` แค่ปรับการแสดงผล) |
| `clock summer-time <ชื่อ> recurring` | ปรับเวลาช่วง Daylight Saving Time (ไม่ใช้ในประเทศไทยเพราะไม่มี DST — ใส่ไว้เพื่อความรู้สำหรับ Lab ที่อยู่ในโซนที่มี DST) |
| `show clock` | แสดงเวลาปัจจุบันของอุปกรณ์ |
| `show clock detail` | แสดงเวลาปัจจุบัน + แหล่งที่มา (NTP / manual) |

> **หมายเหตุ Time Zone ของหลักสูตรนี้**: Lab นี้อยู่ในโซนเวลาประเทศไทย (ICT, UTC+7) ซึ่ง
> **ไม่มี Daylight Saving Time** ดังนั้นเราจะไม่ใช้ `clock summer-time` ในการ config จริง
> แต่จะแสดงตัวอย่าง syntax ไว้ให้ครบสำหรับกรณีที่ deploy ในประเทศที่มี DST (เช่น สหรัฐฯ, ยุโรป)

### Worked Config — ตั้งค่า NTP Client บนทุกอุปกรณ์

**CORE-SW1**
```
CORE-SW1(config)# ntp server 10.10.99.50
CORE-SW1(config)# ntp source Loopback0
CORE-SW1(config)# ntp update-calendar
CORE-SW1(config)# clock timezone ICT 7 0
```

**CORE-SW2**
```
CORE-SW2(config)# ntp server 10.10.99.50
CORE-SW2(config)# ntp source Loopback0
CORE-SW2(config)# ntp update-calendar
CORE-SW2(config)# clock timezone ICT 7 0
```

**DIST-SW1 / DIST-SW2 / DIST-SW3 / DIST-SW4** (ตัวอย่าง DIST-SW1 — ตัวอื่นเปลี่ยนแค่ hostname)
```
DIST-SW1(config)# ntp server 10.10.99.50
DIST-SW1(config)# ntp source Loopback0
DIST-SW1(config)# ntp update-calendar
DIST-SW1(config)# clock timezone ICT 7 0
```

**WAN-EDGE-1 / WAN-EDGE-2** (ตัวอย่าง WAN-EDGE-1)
```
WAN-EDGE-1(config)# ntp server 10.10.99.50
WAN-EDGE-1(config)# ntp source Loopback0
WAN-EDGE-1(config)# ntp update-calendar
WAN-EDGE-1(config)# clock timezone ICT 7 0
```

**ACCESS-SW1-4** (ตัวอย่าง ACCESS-SW1 — Access Switch เป็น L2-only แต่ยังมี VLAN99 management SVI/IP จึงยัง sync NTP ได้ตามปกติ)
```
ACCESS-SW1(config)# ntp server 10.10.99.50
ACCESS-SW1(config)# clock timezone ICT 7 0
```

> ACCESS-SW1-4 ไม่มี Loopback0 (เป็น L2-only switch ตาม Device Platform Reference) จึงไม่ใส่
> `ntp source Loopback0` — จะใช้ Management SVI (VLAN 99) เป็น source โดย default แทน

### ตัวอย่าง Syntax `clock summer-time` (สำหรับความรู้ — ไม่ได้ใช้ใน Lab นี้เพราะไทยไม่มี DST)

```
! ตัวอย่างสมมติถ้า deploy ที่สหรัฐฯ (Eastern Time)
Router(config)# clock timezone EST -5 0
Router(config)# clock summer-time EDT recurring
```

### ตรวจสอบเวลาหลัง config

```
CORE-SW1# show clock detail
15:32:07.481 ICT Fri Sep 26 2026
Time source is NTP
```

`Time source is NTP` คือสิ่งที่ต้องเห็นหลัง config เสร็จและรอ sync สำเร็จ (ใช้เวลาประมาณ
5-15 นาทีสำหรับ IOS ให้ NTP algorithm ยืนยันความน่าเชื่อถือของ server ก่อนเปลี่ยนสถานะเป็น
synchronized เต็มรูปแบบ) ถ้ายังเห็น `Time source is user configuration` หรือ `NTP` ไม่ปรากฏ
ให้ตรวจสอบด้วยคำสั่งใน Step 163

---

## Step 163 — NTP Authentication และการ Verify

### ทำไมต้องมี NTP Authentication

หากไม่มี Authentication ผู้บุกรุกที่เข้าถึง network ได้สามารถปลอมตัวเป็น NTP Server (Spoofing)
และส่งเวลาปลอมให้อุปกรณ์ — ผลที่ตามมาร้ายแรงกว่าที่คิด:

- ทำให้ **Certificate ที่ยังไม่ expire ดูเหมือน expire แล้ว** (หรือกลับกัน — ทำให้ certificate
  ที่ expire ไปแล้วดูเหมือนยังใช้ได้) → เปิดช่องให้ Man-in-the-Middle attack ผ่าน TLS/SSH
- บิดเบือน **timeline ของ log** เพื่อซ่อนหลักฐานการโจมตี (Anti-forensics)
- ทำให้ **Time-based Access Control** (เช่น time-range ACL) ทำงานผิดเวลา
- รบกวน Kerberos/RADIUS ที่พึ่งพา time window ในการ validate token

**NTP Authentication** ใช้ Symmetric Key (MD5 หรือ SHA แล้วแต่ IOS version) เพื่อยืนยันว่า
NTP packet ที่ได้รับมาจาก server ที่เชื่อถือได้จริง ไม่ถูกปลอมหรือแก้ไขระหว่างทาง

### คำสั่งหลักที่ต้องรู้

| คำสั่ง | ความหมาย |
|---|---|
| `ntp authenticate` | เปิดใช้งาน NTP Authentication ทั้งระบบ (global) |
| `ntp authentication-key <id> md5 <key-string>` | สร้าง key พร้อมเลข id และ key string (ต้องตรงกันทั้งฝั่ง client/server) |
| `ntp trusted-key <id>` | บอกให้ device "เชื่อ" key นี้ (ถ้าไม่ trust key จะถูกสร้างไว้แต่ไม่ใช้งานจริง) |
| `ntp server <ip> key <id>` | ผูก NTP server เข้ากับ key id ที่กำหนด — บังคับให้ต้อง authenticate ก่อนยอมรับเวลาจาก server นี้ |

> **ข้อสังเกตสำคัญ**: ต้องทำครบ 3 คำสั่ง (`ntp authenticate` + `ntp authentication-key` +
> `ntp trusted-key`) ควบคู่กับการผูก key เข้ากับ `ntp server ... key <id>` — ถ้าขาดข้อใดข้อหนึ่ง
> NTP จะไม่ authenticate หรือไม่ sync เวลาเลย เป็นจุดที่มือใหม่ผิดพลาดบ่อยที่สุด

### Worked Config — เปิด NTP Authentication (ตัวอย่าง CORE-SW1 กับ NOC-SRV)

สมมติ NOC-SRV configured ให้ต้องการ key id `1` ด้วย key string `NtpAuthKey2026!`
(ฝั่ง NOC-SRV ซึ่งเป็น Linux NTP server ก็ต้องตั้ง key ชุดเดียวกันใน `/etc/ntp.conf` หรือ
`/etc/chrony.conf` — อยู่นอกขอบเขตของ IOS CLI แต่ต้องเข้าใจว่าเป็นค่าที่ต้อง "ตรงกันทั้งสองฝั่ง"
เหมือน pre-shared key ทั่วไป)

```
CORE-SW1(config)# ntp authenticate
CORE-SW1(config)# ntp authentication-key 1 md5 NtpAuthKey2026!
CORE-SW1(config)# ntp trusted-key 1
CORE-SW1(config)# ntp server 10.10.99.50 key 1
```

ทำแบบเดียวกันกับทุกอุปกรณ์ (CORE-SW2, DIST-SW1-4, WAN-EDGE-1/2, ACCESS-SW1-4) — key id และ
key string ต้องเป็นชุดเดียวกันทั้งหมดเพื่อให้ทุกอุปกรณ์ authenticate กับ NOC-SRV ได้สำเร็จ

> **Best Practice**: ใน Production ควรใช้ SHA-based key (บน IOS-XE รุ่นใหม่รองรับ
> `ntp authentication-key <id> sha1 <key-string>` หรือสูงกว่า) แทน MD5 เพราะ MD5 ถือว่า
> cryptographically weak แล้วในปัจจุบัน — ในหลักสูตรนี้ใช้ MD5 เป็นตัวอย่างเพื่อความเข้ากันได้กับ
> IOS เวอร์ชันเก่าที่ยังพบเห็นได้ทั่วไปในสนามจริง แต่แนะนำให้ตรวจสอบ platform ของท่านว่ารองรับ
> SHA authentication หรือไม่ก่อน deploy จริง

### คำสั่ง Verify NTP

```
CORE-SW1# show ntp status
Clock is synchronized, stratum 3, reference is 10.10.99.50
nominal freq is 250.0000 Hz, actual freq is 249.9998 Hz, precision is 2**18
ntp uptime is 245900 (1/100 of seconds), resolution is 4000
reference time is EB2B3C1A.6C3F5A00 (15:32:26.421 ICT Fri Sep 26 2026)
clock offset is 0.0421 msec, root delay is 3.24 msec
root dispersion is 8.19 msec, peer dispersion is 1.05 msec
loopfilter state is 'CTRL' (Normal Controlled Loop), drift is 0.000008192 s/s
system poll interval is 64, last update was 38 sec ago
```

**อ่าน output อย่างไร**:
- `Clock is synchronized` = สำเร็จแล้ว (ถ้าเห็น `Clock is unsynchronized` ต้องตรวจสอบ connectivity/key)
- `stratum 3` = ตรงกับที่คาดไว้ (NOC-SRV เป็น stratum 2, เราจึงเป็น stratum 3)
- `reference is 10.10.99.50` = กำลัง sync จาก NOC-SRV จริง
- `clock offset` = ความต่างของเวลาปัจจุบันเทียบกับ reference (ยิ่งน้อยยิ่งดี หน่วย millisecond)

```
CORE-SW1# show ntp associations

  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.10.99.50     .LOCL.           2     42     64   377   3.241  0.042   1.052
 * sys.peer, # selected, + candidate, - outlyer, x falseticker, ~ configured
```

**อ่าน column สำคัญ**:
- `*` หน้า address = "sys.peer" หมายถึง server ตัวนี้ถูกเลือกเป็นแหล่งเวลาหลักแล้ว (สำคัญที่สุด)
- `st` = stratum ของ server นั้น
- `reach` = ค่า octal ของการติดต่อสำเร็จใน 8 poll ล่าสุด (377 = สำเร็จครบทุกครั้ง)
- `delay` / `offset` / `disp` = ค่าคุณภาพของ link (delay, offset, dispersion) หน่วย millisecond

```
CORE-SW1# show ntp associations detail
10.10.99.50 configured, authenticated, our_master, sane, valid, stratum 2
ref ID .LOCL., time EB2B3B9E.12345678 (15:29:34.070 ICT Fri Sep 26 2026)
our mode client, peer mode server, our poll intvl 64, peer poll intvl 64
root delay 12.00 msec, root disp 45.00, reach 377, sync dist 68.250
delay 3.24 msec, offset 0.0421 msec, dispersion 1.05
```

`authenticated` ในบรรทัดแรกยืนยันว่า NTP Authentication ทำงานสำเร็จจริง — ถ้าเห็นคำว่า
`unauthenticated` แปลว่า key ไม่ตรงกันหรือยังไม่ผูก key กับ server ให้ตรวจสอบ config ใหม่

---

## Step 164 — Syslog Fundamentals: 8 Severity Level

### Syslog คืออะไร

**Syslog** คือ Protocol มาตรฐาน (RFC 5424) สำหรับส่ง log message จากอุปกรณ์ไปยัง collector
กลาง ทำงานบน **UDP Port 514** (เดิม) หรือ **TCP Port 6514 พร้อม TLS** (secure syslog, สำหรับ
production ที่ต้องการความปลอดภัยสูง) — Cisco IOS ผลิต syslog message ทุกครั้งที่มี event สำคัญ
เกิดขึ้น เช่น interface up/down, config change, security violation, routing protocol flap

### 8 Severity Level (ต้องจำให้ขึ้นใจ — ออกสอบ CCNA/CCNP แน่นอน)

| Level | ชื่อ | คำอธิบาย | ตัวอย่างที่ trigger |
|---|---|---|---|
| 0 | **Emergency** | ระบบใช้งานไม่ได้เลย (system unusable) | Hardware ล่มทั้งระบบ, kernel panic |
| 1 | **Alert** | ต้องแก้ไขทันที | Memory หมดจนกระทบการทำงานหลัก, temperature เกิน critical threshold |
| 2 | **Critical** | สภาวะวิกฤต | Power supply ล่ม, module ตัวหนึ่ง fail |
| 3 | **Error** | สภาวะ error | Interface flapping ต่อเนื่อง, ACL deny ผิดปกติจำนวนมาก |
| 4 | **Warning** | สภาวะเตือน | High CPU utilization, spanning-tree topology change บ่อยผิดปกติ |
| 5 | **Notification** | เหตุการณ์ปกติแต่สำคัญ (Notice) | Interface up/down ปกติ, `%LINK-3-UPDOWN`, `%LINEPROTO-5-UPDOWN` |
| 6 | **Informational** | ข้อมูลทั่วไป | Login สำเร็จ, `%SYS-6-LOGGINGHOST_STARTSTOP` |
| 7 | **Debugging** | รายละเอียดระดับ debug | ทุก packet/event แบบละเอียดที่สุด (ใช้เฉพาะช่วง troubleshoot) |

**เทคนิคจำ (Mnemonic)**: **E**very **A**dmin **C**ares **E**specially **W**hen **N**etworks
**I**nvolve **D**owntime (Emergency, Alert, Critical, Error, Warning, Notification,
Informational, Debugging)

> **กฎสำคัญที่สุดของ Syslog Level**: การกำหนด level ใดก็ตาม (เช่น `logging trap warnings`)
> จะ**รวมทุก level ที่ตัวเลขน้อยกว่าหรือเท่ากับ**เข้ามาด้วยเสมอ (ยิ่งเลขน้อยยิ่ง severe และ
> ถูกรวมด้วย) เช่น `warnings` (level 4) = จะได้รับ message ตั้งแต่ level 0-4 ทั้งหมด
> ไม่ใช่แค่ level 4 อย่างเดียว

### Syslog Message Format ของ Cisco IOS

```
seq no:timestamp: %facility-severity-MNEMONIC: description
```

**ตัวอย่างจริง**:
```
000123: Sep 26 2026 15:32:07.481 ICT: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
000124: Sep 26 2026 15:32:10.102 ICT: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
000125: Sep 26 2026 15:35:44.933 ICT: %SYS-5-CONFIG_I: Configured from console by admin on vty0 (10.10.99.100)
000126: Sep 26 2026 15:40:12.771 ICT: %SEC-6-IPACCESSLOGP: list BLOCK_TELNET denied tcp 203.0.113.99(52344) -> 10.10.99.1(23), 1 packet
```

**อ่านแต่ละส่วน**:
- `000123:` = sequence number (ถ้าเปิด `service sequence-numbers`)
- `Sep 26 2026 15:32:07.481 ICT:` = timestamp (ต้องเปิด `service timestamps` — ดู Step 165)
- `%LINK-3-UPDOWN` = **Facility-Severity-Mnemonic**
  - `LINK` = Facility (กลุ่มของ subsystem ที่สร้าง message — เช่น LINK, SYS, SEC, OSPF, BGP)
  - `3` = Severity level (Error ในกรณีนี้)
  - `UPDOWN` = Mnemonic (short code บอกประเภท event)
- `Interface GigabitEthernet0/1, changed state to down` = ข้อความอธิบายเหตุการณ์

> สังเกตว่า `%LINK-3-UPDOWN` (interface **down**) severity เป็น 3 (Error) แต่
> `%LINEPROTO-5-UPDOWN` (line protocol **up**) severity เป็น 5 (Notification) — Cisco ออกแบบให้
> เหตุการณ์ "แย่ลง" (down) มี severity สูงกว่าเหตุการณ์ "กลับมาปกติ" (up) เสมอ

---

## Step 165 — Configuring Syslog ส่ง Log ไปยัง NOC-SRV

### คำสั่งหลักที่ต้องรู้

| คำสั่ง | ความหมาย |
|---|---|
| `logging host <ip>` (หรือ `logging <ip>` บน IOS เก่า) | กำหนดปลายทาง Syslog Server |
| `logging trap <level>` | กำหนดว่าจะส่ง severity level ใดขึ้นไป (level นั้นและที่ severe กว่า) ไปยัง remote syslog server |
| `logging source-interface <interface>` | กำหนด source IP ที่ใช้ส่ง syslog packet (แนะนำ Loopback0 เพื่อ IP คงที่ ไม่เปลี่ยนตาม physical interface) |
| `logging buffered <size> <level>` | เก็บ log ไว้ใน RAM buffer ของอุปกรณ์เอง (ดูรายละเอียด Step 166) |
| `service timestamps log datetime msec` | เปิด timestamp แบบละเอียด (พร้อม millisecond) ในทุก log message — **จำเป็นมาก** เมื่อใช้ร่วมกับ NTP |
| `service timestamps log datetime msec localtime show-timezone` | เพิ่ม timezone abbreviation (เช่น ICT) ต่อท้าย timestamp |
| `service sequence-numbers` | เพิ่ม sequence number หน้าทุก log message (ช่วยตรวจว่า message หายไปหรือไม่จาก buffer) |
| `logging on` | เปิดใช้งาน logging (default เปิดอยู่แล้ว แต่ระบุไว้ชัดเจนเพื่อความรัดกุม) |

### Worked Config — ส่ง Syslog ไปยัง NOC-SRV (ตัวอย่าง CORE-SW1)

```
CORE-SW1(config)# service timestamps log datetime msec localtime show-timezone
CORE-SW1(config)# service sequence-numbers
CORE-SW1(config)# logging host 10.10.99.50
CORE-SW1(config)# logging trap informational
CORE-SW1(config)# logging source-interface Loopback0
CORE-SW1(config)# logging on
```

**คำอธิบายการเลือก `logging trap informational` (level 6)**:
เราเลือกส่งตั้งแต่ level 0-6 (Emergency ถึง Informational) ไปยัง NOC-SRV แต่**ไม่ส่ง level 7
(Debugging)** ผ่าน remote syslog เพราะ debug message มีปริมาณมหาศาลและควรเปิดใช้เฉพาะตอน
troubleshoot สด ๆ ผ่าน console/terminal monitor เท่านั้น (ดู Step 166) — ถ้าส่ง debug level
ไปยัง remote syslog ตลอดเวลา จะทำให้ NOC-SRV และ network bandwidth ล้นด้วย log ที่ไม่จำเป็น

### Config เดียวกันสำหรับทุกอุปกรณ์ใน Lab

ใช้ config block เดียวกันนี้กับ **CORE-SW2, DIST-SW1-4, WAN-EDGE-1/2** (เปลี่ยนแค่ hostname
ใน prompt เท่านั้น เนื้อหาคำสั่งเหมือนกันทั้งหมด):

```
service timestamps log datetime msec localtime show-timezone
service sequence-numbers
logging host 10.10.99.50
logging trap informational
logging source-interface Loopback0
logging on
```

สำหรับ **ACCESS-SW1-4** (ไม่มี Loopback0) ใช้ VLAN 99 SVI/management IP เป็น source แทน:

```
service timestamps log datetime msec localtime show-timezone
service sequence-numbers
logging host 10.10.99.50
logging trap informational
logging on
```

### ตัวอย่าง Log ที่ NOC-SRV จะได้รับ (แสดงจากฝั่ง Syslog Collector)

```
Sep 26 15:32:07 CORE-SW1 000123: Sep 26 2026 15:32:07.481 ICT: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
Sep 26 15:32:10 CORE-SW1 000124: Sep 26 2026 15:32:10.102 ICT: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
Sep 26 15:33:52 DIST-SW1 000891: Sep 26 2026 15:33:52.209 ICT: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet1/1/1 from LOADING to FULL, Loading Done
```

สังเกตว่าแต่ละบรรทัดมี **hostname ต้นทาง** (CORE-SW1, DIST-SW1) นำหน้า — นี่คือประโยชน์หลักของ
การมี Central Syslog Server: engineer สามารถกรอง/ค้นหา log ข้ามอุปกรณ์ทั้งหมดได้จากที่เดียว
และเพราะทุกอุปกรณ์ sync เวลาผ่าน NTP จาก NOC-SRV ตัวเดียวกัน (Step 162) ทำให้ timestamp
ของ log จากทุกอุปกรณ์**เรียงลำดับเหตุการณ์ได้ถูกต้องแม่นยำ** — นี่คือจุดที่ NTP กับ Syslog
ทำงานเสริมกันโดยตรง

---

## Step 166 — Local Logging: Buffer vs Console vs Terminal Monitor

Cisco IOS มีปลายทาง (destination) ของ log message อยู่ 4 แบบหลัก ซึ่งควบคุม level ได้
**อิสระจากกัน**

| ปลายทาง | คำสั่งควบคุม | ใช้เมื่อ |
|---|---|---|
| **Console** | `logging console <level>` | ข้อความที่ขึ้นบนหน้าจอ console โดยตรง (physical console cable) |
| **Terminal (VTY/SSH session)** | `terminal monitor` + `logging monitor <level>` | ข้อความที่ขึ้นบน SSH/Telnet session (ต้องเปิด `terminal monitor` ต่อ session ด้วย) |
| **Buffer (RAM)** | `logging buffered <size> <level>` | เก็บ log ไว้ในหน่วยความจำอุปกรณ์เอง ดูย้อนหลังด้วย `show logging` |
| **Remote Syslog Server** | `logging host <ip>` + `logging trap <level>` | ส่งไปเก็บที่ NOC-SRV (ตามที่ config ใน Step 165) |

### ปัญหา CPU จาก Console Logging (สำคัญมากสำหรับ Production)

**Console logging เป็นปลายทางที่ "แพง" ที่สุดในเชิง CPU** เพราะ:

1. Console port ทำงานที่ความเร็วต่ำมาก (default 9600 bps)
2. IOS ต้อง**หยุดรอ** ให้ log message ถูกส่งออก console port จนเสร็จก่อนทำงานอื่นต่อ (synchronous
   operation แบบเดิม) — ถ้ามี log flooding (เช่น interface flap ถี่ ๆ, spanning-tree loop,
   BGP flap จำนวนมาก) CPU จะถูกอุปกรณ์ก็จะ "ค้าง" ไปกับการพิมพ์ log ออก console จนควบคุมอุปกรณ์
   ไม่ได้เลย — เป็นสาเหตุ **Denial of Service ที่เกิดจาก config ผิดพลาดของตัวเองบ่อยที่สุด**
   ในสนามจริง

### แนวทางแก้ปัญหา — Best Practice สำหรับ Production

**วิธีที่ 1: ลด level ของ console logging**
```
CORE-SW1(config)# logging console warnings
```
จะให้ console แสดงแค่ level 0-4 (Emergency ถึง Warning) ลด volume ของ message ที่ต้องพิมพ์
ออก console อย่างมาก

**วิธีที่ 2 (แนะนำสำหรับ Production จริง): ปิด console logging ไปเลย**
```
CORE-SW1(config)# no logging console
```
แล้วพึ่งพา **Buffer + Remote Syslog** เป็นหลักแทน เพราะทั้งสองทางนี้เร็วกว่ามาก (buffer เขียนลง
RAM, remote syslog ส่งผ่าน UDP แบบ asynchronous ไม่บล็อก CPU) วิธีนี้เป็นแนวทางที่ Cisco และ
วิศวกร Network ระดับ Enterprise ใช้กันเป็นมาตรฐานบนอุปกรณ์ที่ต้อง handle traffic สูง

### Worked Config — ตั้งค่า Local Logging แบบ Production-grade (ตัวอย่าง CORE-SW1)

```
CORE-SW1(config)# no logging console
CORE-SW1(config)# logging buffered 65536 informational
CORE-SW1(config)# logging monitor informational
```

- `no logging console` — ปิด console logging ป้องกัน CPU flooding
- `logging buffered 65536 informational` — เก็บ log ใน RAM buffer ขนาด 64KB ระดับ level 0-6
- `logging monitor informational` — กำหนด level สำหรับ SSH/VTY session ที่เปิด `terminal monitor`

### คำสั่ง Verify และใช้งาน Local Log

```
! ดู log ทั้งหมดที่อยู่ใน buffer
CORE-SW1# show logging

Syslog logging: enabled (0 messages dropped, 3 messages rate-limited,
                 0 flushes, 0 overruns, xml disabled, filtering disabled)
No Active Message Discriminator.
No Inactive Message Discriminator.

    Console logging: disabled
    Monitor logging: level informational, 412 messages logged
    Buffer logging: level informational, 412 messages logged, xml disabled
    Logging Exception size (4096 bytes)
    Count and timestamp logging messages: disabled
    Persistent logging: disabled

No active filter modules.

    Trap logging: level informational, 412 message lines logged
        Logging to 10.10.99.50 (udp port 514, audit disabled,
              link up), 412 message lines logged, 0 message lines rate-limited,
              0 message lines dropped-by-MD

Log Buffer (65536 bytes):
000891: Sep 26 2026 15:33:52.209 ICT: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet1/1/1 from LOADING to FULL, Loading Done
000892: Sep 26 2026 15:35:44.933 ICT: %SYS-5-CONFIG_I: Configured from console by admin on vty0 (10.10.99.100)
```

`Console logging: disabled` และ `Trap logging: ... Logging to 10.10.99.50 ... link up`
คือสองบรรทัดที่ยืนยันว่า config ทำงานถูกต้องตามที่ตั้งใจ

```
! เปิดดู log ผ่าน SSH session แบบ real-time (ต้องเปิดทุกครั้งที่ login ใหม่ เพราะไม่ persist)
CORE-SW1# terminal monitor

! ลบ log buffer ทั้งหมด (ใช้ก่อนเริ่ม troubleshoot ใหม่เพื่อให้เห็นแต่ log ใหม่)
CORE-SW1# clear logging
```

> **ข้อควรระวัง**: `terminal monitor` มีผลเฉพาะ session ปัจจุบันเท่านั้น (ไม่ใช่ global config)
> ถ้า disconnect แล้ว SSH เข้าใหม่ ต้องพิมพ์ `terminal monitor` อีกครั้งเสมอ

---

## Step 167 — SNMP Fundamentals: SNMPv1/v2c vs SNMPv3

### SNMP คืออะไร

**SNMP (Simple Network Management Protocol)** เป็น Protocol มาตรฐานสำหรับให้ NMS
(Network Management System — ในหลักสูตรนี้คือ NOC-SRV ที่จำลองบทบาทคล้าย
Splunk/PRTG/LibreNMS) สามารถ **ดึงข้อมูลสถานะ** (Polling) และ **รับการแจ้งเตือนอัตโนมัติ**
(Traps) จากอุปกรณ์เครือข่ายได้แบบรวมศูนย์ — ทำงานบน **UDP Port 161** (Polling/Query) และ
**UDP Port 162** (Trap)

### SNMP มี 3 เวอร์ชันหลัก

| เวอร์ชัน | Authentication | Encryption | ระดับความปลอดภัย |
|---|---|---|---|
| **SNMPv1** | Community String (plaintext) | ไม่มี | ต่ำมาก — ล้าสมัย ไม่ควรใช้แล้ว |
| **SNMPv2c** | Community String (plaintext) | ไม่มี | ต่ำ — ใช้ได้เฉพาะ Lab/Legacy เท่านั้น |
| **SNMPv3** | Username + Auth (MD5/SHA) | Priv (DES/AES) | สูง — **มาตรฐานที่ต้องใช้ใน Production** |

### ปัญหาของ SNMPv1/v2c: Community String

SNMPv1/v2c ใช้ **Community String** (เหมือน password ที่ใช้ร่วมกันทั้ง read-only และ
read-write) ซึ่งมีปัญหาความปลอดภัยร้ายแรง:

1. **ส่งแบบ Cleartext (Plaintext)** — ไม่มีการเข้ารหัสเลย ใครดัก packet บนสายได้ก็อ่าน
   community string ได้ทันที (packet sniffing ง่ายมาก)
2. **ไม่มี per-user authentication** — community string เดียวใช้ร่วมกันทุกคนที่ต้อง poll
   อุปกรณ์นั้น ไม่สามารถระบุได้ว่า "ใคร" เป็นคน query
3. **Default community string ที่มีชื่อเสียง** — `public` (read-only) และ `private`
   (read-write) เป็นค่า default ที่รู้กันทั่วไป องค์กรจำนวนมากลืมเปลี่ยนจน**เปิดช่องให้ผู้บุกรุก
   เข้าไปอ่านหรือแม้แต่แก้ไข config อุปกรณ์ผ่าน SNMP Read-Write ได้โดยตรง**

### ทำไม SNMPv3 จึงจำเป็นสำหรับ Production ทุกกรณี

SNMPv3 แก้ปัญหาทั้งหมดข้างบนด้วยแนวคิด **USM (User-based Security Model)**:

| คุณสมบัติของ SNMPv3 | อธิบาย |
|---|---|
| **Authentication** | ใช้ username + auth password ที่ hash ด้วย MD5 หรือ **SHA** (แนะนำ SHA เพราะ MD5 ล้าสมัย) — ยืนยันตัวตนผู้ query ได้จริง ไม่ใช่แค่ shared string |
| **Privacy (Encryption)** | เข้ารหัส payload ทั้งหมดด้วย DES (เก่า, ไม่แนะนำ) หรือ **AES** (แนะนำ AES-128/256) — ป้องกันการดัก packet อ่านข้อมูล |
| **Per-user Access Control** | กำหนดสิทธิ์ (read-only, read-write) แยกตาม user แต่ละคนได้ผ่าน SNMP Group |
| **Message Integrity** | ตรวจสอบว่า SNMP message ไม่ถูกแก้ไขระหว่างทาง (tamper detection) |

> **สรุปสำหรับสนามจริง**: SNMPv1/v2c ควรใช้เฉพาะ **Lab/Test Environment ที่ปิด (isolated)**
> หรือกรณี legacy device ที่ไม่รองรับ v3 จริง ๆ เท่านั้น ทุก deployment ที่คำนึงถึงความปลอดภัย
> (security-conscious) **ต้องใช้ SNMPv3** เป็นมาตรฐาน — ในหลักสูตรนี้เราจะสอนทั้งสองแบบ
> (v2c เพื่อความเข้าใจพื้นฐานและความคุ้นเคยกับอุปกรณ์ legacy, v3 เป็นแนวทาง Production จริง)
> แต่จะ**ใช้ SNMPv3 เป็น config สุดท้ายที่ deploy จริงในทุก Lab ต่อจากนี้**

---

## Step 168 — Configuring SNMPv2c และ SNMPv3

### ส่วนที่ 1: SNMPv2c (เพื่อความคุ้นเคยกับ Lab/Legacy — ไม่แนะนำ Production)

```
CORE-SW1(config)# snmp-server community LabReadOnly2026 RO
CORE-SW1(config)# snmp-server community LabReadWrite2026 RW
CORE-SW1(config)# snmp-server location Enterprise-Lab-Bangkok-DC1
CORE-SW1(config)# snmp-server contact netops@lab.local
```

| คำสั่ง | ความหมาย |
|---|---|
| `snmp-server community <string> RO` | Community string สำหรับสิทธิ์ **Read-Only** (GET เท่านั้น) |
| `snmp-server community <string> RW` | Community string สำหรับสิทธิ์ **Read-Write** (GET และ SET) — ควรหลีกเลี่ยงถ้าไม่จำเป็นจริง ๆ |
| `snmp-server location <ข้อความ>` | ระบุตำแหน่งทางกายภาพของอุปกรณ์ (ปรากฏใน MIB `sysLocation`) |
| `snmp-server contact <ข้อความ>` | ระบุผู้รับผิดชอบอุปกรณ์ (ปรากฏใน MIB `sysContact`) |

> **คำเตือน**: ห้ามใช้ community string ว่า `public` หรือ `private` เด็ดขาดในทุกกรณี (แม้แต่ Lab)
> เพราะเป็นค่า default ที่เครื่องมือโจมตีอัตโนมัติ (SNMP scanner) ลองก่อนเสมอ — ในหลักสูตรนี้
> จึงตั้งชื่อ custom (`LabReadOnly2026`, `LabReadWrite2026`) เพื่อสร้างความเคยชินที่ถูกต้อง

### ส่วนที่ 2: SNMPv3 (แนวทาง Production — แนะนำใช้จริง)

SNMPv3 config มี 3 ชั้นที่ต้องเข้าใจความสัมพันธ์กัน:

```
SNMP View  (กำหนดว่า OID/MIB ส่วนใดที่มองเห็นได้)
    │
SNMP Group  (ผูก View เข้ากับ Security Level: noAuth / authNoPriv / authPriv)
    │
SNMP User   (ผูก User เข้ากับ Group + กำหนด Auth/Priv Password จริง)
```

| คำสั่ง | ความหมาย |
|---|---|
| `snmp-server view <name> <oid-tree> included` | กำหนด MIB subtree ที่มองเห็นได้ (`iso` = มองเห็นทั้งหมด) |
| `snmp-server group <name> v3 priv [read <view>] [write <view>]` | สร้าง Group พร้อม Security Level (`priv` = ต้อง auth และ encrypt ทั้งคู่) |
| `snmp-server user <user> <group> v3 auth sha <auth-pass> priv aes 128 <priv-pass>` | สร้าง User พร้อม Authentication (SHA) และ Privacy (AES-128) |

**Security Level ของ SNMPv3 มี 3 ระดับ**:

| Security Level | Authentication | Encryption | คำแนะนำ |
|---|---|---|---|
| `noAuthNoPriv` | ไม่มี | ไม่มี | ไม่แนะนำ — เท่ากับไม่มีความปลอดภัยเลย |
| `authNoPriv` | มี (MD5/SHA) | ไม่มี | พอใช้ได้ถ้า network internal ปิดสนิท แต่ยังดักอ่านข้อมูลได้ |
| `authPriv` | มี (MD5/SHA) | มี (DES/AES) | **แนะนำสำหรับ Production ทุกกรณี** |

### Worked Config — SNMPv3 แบบสมบูรณ์ (ตัวอย่าง CORE-SW1)

```
! 1. สร้าง View ให้มองเห็น MIB tree ทั้งหมด (iso คือ root ของ OID tree ทั้งหมด)
CORE-SW1(config)# snmp-server view FULL-VIEW iso included

! 2. สร้าง Group พร้อม Security Level = authPriv (auth + encrypt) และผูก View
CORE-SW1(config)# snmp-server group NOC-RO-GROUP v3 priv read FULL-VIEW

! 3. สร้าง User ผูกกับ Group พร้อม Authentication (SHA) และ Privacy (AES-128)
CORE-SW1(config)# snmp-server user noc-svc NOC-RO-GROUP v3 auth sha AuthPass2026Strong! priv aes 128 PrivPass2026Strong!

! 4. Location/Contact (ใช้ร่วมกับทั้ง v2c และ v3)
CORE-SW1(config)# snmp-server location Enterprise-Lab-Bangkok-DC1
CORE-SW1(config)# snmp-server contact netops@lab.local

! 5. (แนะนำ) จำกัด SNMP ให้รับ query จาก NOC-SRV เท่านั้นผ่าน ACL
CORE-SW1(config)# ip access-list standard SNMP-ALLOWED
CORE-SW1(config-std-nacl)# permit host 10.10.99.50
CORE-SW1(config-std-nacl)# deny any log
CORE-SW1(config-std-nacl)# exit
CORE-SW1(config)# snmp-server community LabReadOnly2026 RO SNMP-ALLOWED
```

> **หมายเหตุ**: ขั้นตอนที่ 5 (SNMP ACL) ใช้ได้กับทั้ง v2c community และผูกกับ v3 user ผ่าน
> `snmp-server user ... access <acl-name>` ได้เช่นกัน — เป็น **Defense in Depth** ชั้นเพิ่มเติม
> แม้จะใช้ SNMPv3 อยู่แล้วก็ตาม เพราะจำกัดว่า "ใครในเชิง IP" ที่มีสิทธิ์ query ได้เลย ไม่ใช่แค่พึ่ง
> username/password เพียงอย่างเดียว

### ทำแบบเดียวกันกับทุกอุปกรณ์ใน Lab (username/password ชุดเดียวกันทั้งหมดเพื่อให้ NOC-SRV
### บริหารจัดการง่าย — ใน production จริงควรพิจารณาแยก credential ตาม tier ของอุปกรณ์)

Config Block สำหรับ **CORE-SW2, DIST-SW1-4, WAN-EDGE-1/2**:

```
snmp-server view FULL-VIEW iso included
snmp-server group NOC-RO-GROUP v3 priv read FULL-VIEW
snmp-server user noc-svc NOC-RO-GROUP v3 auth sha AuthPass2026Strong! priv aes 128 PrivPass2026Strong!
snmp-server location Enterprise-Lab-Bangkok-DC1
snmp-server contact netops@lab.local
```

### คำสั่ง Verify SNMP

```
CORE-SW1# show snmp user

User name: noc-svc
Engine ID: 800000090300ABCD12345678
storage-type: nonvolatile        active
Authentication Protocol: SHA
Privacy Protocol: AES128
Group-name: NOC-RO-GROUP

CORE-SW1# show snmp group

groupname: NOC-RO-GROUP           security model:v3 priv
readview : FULL-VIEW              writeview: <no writeview specified>
notifyview:  <no notifyview specified>
row status: active

CORE-SW1# show snmp community

Community name: LabReadOnly2026
Community Index: default
Community SecurityName: LabReadOnly2026
storage-type: nonvolatile        active access-list: SNMP-ALLOWED

CORE-SW1# show snmp
Chassis: 12345678
Contact: netops@lab.local
Location: Enterprise-Lab-Bangkok-DC1
0 SNMP packets input
    0 Bad SNMP version errors
    0 Unknown community name
    0 Illegal operation for community name supplied
    0 Encoding errors
    0 Number of requested variables
    0 Number of altered variables
    0 Get-request PDUs
    0 Get-next PDUs
    0 Set-request PDUs
0 SNMP packets output
    0 Too big errors (Maximum packet size 1500)
    0 No such name errors
    0 Bad values errors
    0 General errors
SNMP logging: disabled
```

---

## Step 169 — SNMP Traps vs Polling + แนวคิด MIB/OID

### Polling vs Traps: สองแนวทางในการ Monitor

| แนวทาง | ทิศทาง | ลักษณะ | ตัวอย่างการใช้งาน |
|---|---|---|---|
| **Polling (GET)** | NMS → Device (ถามเข้ามา) | NMS สอบถามสถานะเป็นระยะ (เช่น ทุก 60-300 วินาที) — "Pull" model | Interface utilization graph, CPU/Memory trend, uptime dashboard |
| **Traps (Notifications)** | Device → NMS (แจ้งออกไปเอง) | อุปกรณ์ส่งแจ้งเตือนทันทีที่ event สำคัญเกิดขึ้น — "Push" model | Interface down, power supply fail, config change, high CPU threshold exceeded |

**ข้อดี/ข้อเสียเทียบกัน**:

- **Polling** — ข้อดีคือได้ข้อมูลสม่ำเสมอเพื่อทำ trend/graph ระยะยาว แต่ข้อเสียคือ**มี delay**
  (เหตุการณ์เกิดขึ้นแล้วแต่ NMS ยังไม่รู้จนกว่าจะถึงรอบ poll ถัดไป) และเพิ่ม traffic/load
  บนอุปกรณ์และ network สม่ำเสมอ
- **Traps** — ข้อดีคือ**เกือบ real-time** (แจ้งทันทีที่ event เกิด) แต่ข้อเสียคือ **UDP-based
  จึงไม่รับประกันการส่งถึง** (ถ้า trap packet หายไประหว่างทาง NMS จะไม่รู้เลยว่า event
  เกิดขึ้น) — ด้วยเหตุนี้ Production ที่ดีจึงใช้**ทั้งสองแบบร่วมกัน** (Traps สำหรับ real-time
  alert, Polling สำหรับ trend/graph และเป็น safety-net ที่จะจับความผิดปกติได้แม้ trap หาย)

### Configuring SNMP Traps ส่งไปยัง NOC-SRV

```
! เปิดการส่ง trap ประเภทสำคัญ (เลือกเฉพาะที่เกี่ยวข้อง ไม่เปิดทุกประเภทเพื่อลด noise)
CORE-SW1(config)# snmp-server enable traps snmp linkdown linkup coldstart warmstart
CORE-SW1(config)# snmp-server enable traps config
CORE-SW1(config)# snmp-server enable traps cpu threshold
CORE-SW1(config)# snmp-server enable traps envmon
CORE-SW1(config)# snmp-server enable traps flash insertion removal

! กำหนดปลายทาง trap และ SNMPv3 user ที่ใช้ส่ง (ต้องเป็น user ที่มีอยู่จริงจาก Step 168)
CORE-SW1(config)# snmp-server host 10.10.99.50 version 3 priv noc-svc
```

| คำสั่ง | ความหมาย |
|---|---|
| `snmp-server enable traps <ประเภท>` | เปิดการส่ง trap เฉพาะประเภทที่ระบุ (ไม่ระบุ argument = เปิดทุกประเภทที่รองรับ) |
| `snmp-server host <ip> version 3 <priv\|auth\|noauth> <user>` | กำหนดปลายทาง trap พร้อมระดับความปลอดภัยและ SNMPv3 user ที่ใช้ |

> **คำแนะนำ**: อย่าเปิด `snmp-server enable traps` แบบไม่ระบุ argument (เปิดทุกประเภท) ใน
> Production เพราะจะทำให้ trap flood เข้า NMS จำนวนมากเกินจำเป็น ควรเลือกเฉพาะประเภทที่ทีม
> จะนำไปใช้ตอบสนองจริง (actionable) เท่านั้น เช่น link state, config change, environmental
> (temperature/power/fan), CPU/memory threshold

### แนวคิด MIB และ OID (พื้นฐานที่ต้องเข้าใจก่อนใช้ SNMP จริง)

**MIB (Management Information Base)** คือโครงสร้างข้อมูลแบบ**ลำดับชั้นต้นไม้ (Tree)**
ที่กำหนดว่าอุปกรณ์มีข้อมูลอะไรให้ query ได้บ้าง แต่ละจุดใน tree มีเลขระบุตำแหน่งเรียกว่า
**OID (Object Identifier)** เขียนเป็นชุดเลขคั่นด้วยจุด

```
                          iso (1)
                            │
                      org (1.3)
                            │
                     dod (1.3.6)
                            │
                internet (1.3.6.1)
                            │
              mgmt (1.3.6.1.2)
                            │
              mib-2 (1.3.6.1.2.1)
           ┌────────────────┼────────────────┐
     system (1)         interfaces (2)      ip (4)
  (1.3.6.1.2.1.1)    (1.3.6.1.2.1.2)   (1.3.6.1.2.1.4)
        │                    │
   sysUpTime            ifTable
   (...1.3)             (...1.2.2.1)
```

**ตัวอย่าง OID ที่ใช้บ่อยที่สุด**:

| OID | ความหมาย |
|---|---|
| `1.3.6.1.2.1.1.3.0` | `sysUpTime` — ระยะเวลาอุปกรณ์ทำงานต่อเนื่อง |
| `1.3.6.1.2.1.1.5.0` | `sysName` — hostname ของอุปกรณ์ |
| `1.3.6.1.2.1.2.2.1.10` | `ifInOctets` — จำนวน byte ที่รับเข้าต่อ interface (ใช้คำนวณ bandwidth utilization) |
| `1.3.6.1.4.1.9.9.109.1.1.1.1.5` | Cisco-specific: CPU utilization (5-minute average, จาก CISCO-PROCESS-MIB) |

> Cisco มี **Private Enterprise MIB** ของตัวเอง (OID ขึ้นต้นด้วย `1.3.6.1.4.1.9` ซึ่ง `9` คือ
> Enterprise Number ที่ IANA จัดสรรให้ Cisco) เพื่อเปิดเผยข้อมูลเฉพาะของ Cisco ที่ไม่มีใน
> Standard MIB — NMS อย่าง LibreNMS/PRTG/SolarWinds จะมี MIB file ของ Cisco โหลดไว้ล่วงหน้า
> เพื่อแปลความหมายของ OID เหล่านี้ให้อ่านง่ายเป็นชื่อ (เช่นแปลง `1.3.6.1.4.1.9.9.109.1.1.1.1.5`
> เป็น "CPU 5-min Average") อัตโนมัติ

### ทดสอบ Polling ด้วยตนเองจากเครื่อง Linux (มุมมองฝั่ง NMS)

```
# GET แบบ SNMPv3 (authPriv) จากเครื่อง NOC-SRV ไปยัง CORE-SW1
$ snmpget -v3 -l authPriv -u noc-svc -a SHA -A 'AuthPass2026Strong!' \
    -x AES -X 'PrivPass2026Strong!' 10.255.10.1 1.3.6.1.2.1.1.5.0

SNMPv2-MIB::sysName.0 = STRING: CORE-SW1
```

---

## Step 170 — Lab: NTP + Syslog + SNMPv3 แบบสมบูรณ์บนทุกอุปกรณ์

ในขั้นตอนสุดท้ายนี้ เราจะรวม config ทั้ง 3 ระบบ (NTP, Syslog, SNMPv3) เข้าเป็น **Template
เดียวที่สอดคล้องกัน** เพื่อ apply บน **CORE-SW1, CORE-SW2, DIST-SW1-4, WAN-EDGE-1/2**
ทุกตัวชี้ไปที่ **NOC-SRV (10.10.99.50)** เพียงจุดเดียวสำหรับทั้ง 3 บริการ

### Network Management Template (`nms-services.txt`) — ใช้กับทุกอุปกรณ์ L3 ใน Lab

```
!
! =========================================================
!  Network Management Services Template
!  Part 17 - NTP / Syslog / SNMPv3
!  Target: CORE-SW1/2, DIST-SW1-4, WAN-EDGE-1/2
!  NOC-SRV = 10.10.99.50 (VLAN 99 - MGMT)
! =========================================================
!
! --- 1. NTP (Step 162-163) ---
ntp authenticate
ntp authentication-key 1 md5 NtpAuthKey2026!
ntp trusted-key 1
ntp server 10.10.99.50 key 1
ntp source Loopback0
ntp update-calendar
clock timezone ICT 7 0
!
! --- 2. Syslog (Step 165-166) ---
service timestamps log datetime msec localtime show-timezone
service sequence-numbers
logging host 10.10.99.50
logging trap informational
logging source-interface Loopback0
logging buffered 65536 informational
logging monitor informational
no logging console
logging on
!
! --- 3. SNMPv3 (Step 168-169) ---
snmp-server view FULL-VIEW iso included
snmp-server group NOC-RO-GROUP v3 priv read FULL-VIEW
snmp-server user noc-svc NOC-RO-GROUP v3 auth sha AuthPass2026Strong! priv aes 128 PrivPass2026Strong!
snmp-server location Enterprise-Lab-Bangkok-DC1
snmp-server contact netops@lab.local
ip access-list standard SNMP-ALLOWED
 permit host 10.10.99.50
 deny any log
exit
snmp-server community LabReadOnly2026 RO SNMP-ALLOWED
snmp-server enable traps snmp linkdown linkup coldstart warmstart
snmp-server enable traps config
snmp-server enable traps cpu threshold
snmp-server enable traps envmon
snmp-server host 10.10.99.50 version 3 priv noc-svc
!
end
copy running-config startup-config
```

### ตารางสรุปการ Apply ทั่ว Lab

| Device | Loopback0 (NTP/Syslog source) | NTP Stratum ที่ได้ | Syslog ปลายทาง | SNMPv3 User |
|---|---|---|---|---|
| CORE-SW1 | 1.1.1.1/32 | 3 | 10.10.99.50 | noc-svc |
| CORE-SW2 | 1.1.1.2/32 | 3 | 10.10.99.50 | noc-svc |
| DIST-SW1 | 1.1.1.11/32 | 3 | 10.10.99.50 | noc-svc |
| DIST-SW2 | 1.1.1.12/32 | 3 | 10.10.99.50 | noc-svc |
| DIST-SW3 | 1.1.1.13/32 | 3 | 10.10.99.50 | noc-svc |
| DIST-SW4 | 1.1.1.14/32 | 3 | 10.10.99.50 | noc-svc |
| WAN-EDGE-1 | 1.1.1.21/32 | 3 | 10.10.99.50 | noc-svc |
| WAN-EDGE-2 | 1.1.1.22/32 | 3 | 10.10.99.50 | noc-svc |
| ACCESS-SW1-4 | ไม่มี Loopback0 (ใช้ VLAN99 SVI) | 3 | 10.10.99.50 | noc-svc |
| **NOC-SRV** | 10.10.99.50 (fixed) | **2** (Reference) | (เป็น collector เอง) | (เป็น NMS เอง) |

> ACCESS-SW1-4 apply เฉพาะส่วน NTP+Syslog (ไม่มี `ntp source`/`logging source-interface`
> เนื่องจากไม่มี Loopback0 ตามที่ระบุใน Device Platform Reference — ส่วน SNMPv3 config
> เหมือนกันทุกประการ)

### Verification Checklist ฉบับสมบูรณ์ (รันบนแต่ละอุปกรณ์หลัง Apply)

```
! 1. ตรวจสอบ NTP sync สำเร็จ
CORE-SW1# show ntp status
Clock is synchronized, stratum 3, reference is 10.10.99.50
...
CORE-SW1# show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.10.99.50     .LOCL.           2     18     64   377   3.241  0.042   1.052

! 2. ตรวจสอบ clock ปัจจุบัน (ต้องขึ้น "Time source is NTP")
CORE-SW1# show clock detail
15:58:41.204 ICT Fri Sep 26 2026
Time source is NTP

! 3. ตรวจสอบ Syslog config และสถานะการส่งไป NOC-SRV
CORE-SW1# show logging | include Console|Trap logging|Logging to
    Console logging: disabled
    Trap logging: level informational, 512 message lines logged
        Logging to 10.10.99.50 (udp port 514, audit disabled, link up), 512 message lines logged

! 4. ตรวจสอบ SNMPv3 User และ Group
CORE-SW1# show snmp user
User name: noc-svc
Engine ID: 800000090300ABCD12345678
Authentication Protocol: SHA
Privacy Protocol: AES128
Group-name: NOC-RO-GROUP

CORE-SW1# show snmp group
groupname: NOC-RO-GROUP           security model:v3 priv
readview : FULL-VIEW              writeview: <no writeview specified>

! 5. ตรวจสอบว่า running-config มี block ทั้ง 3 ระบบครบถ้วน
CORE-SW1# show running-config | section ntp
CORE-SW1# show running-config | section logging
CORE-SW1# show running-config | section snmp
```

### ผลลัพธ์สุดท้ายที่ได้จาก Part นี้

หลังจาก apply template นี้ทั่วทั้ง Lab Topology เราจะได้:

1. **เวลาที่ตรงกันทั้งองค์กร** (NTP Stratum 3 ทุกอุปกรณ์, sync จาก NOC-SRV Stratum 2 ตัวเดียว)
   พร้อม Authentication ป้องกัน time-spoofing
2. **Log แบบรวมศูนย์** ที่ NOC-SRV พร้อม timestamp ที่แม่นยำและตรงกันทุกอุปกรณ์ — สามารถ
   เรียง timeline เหตุการณ์ข้ามอุปกรณ์ได้ถูกต้อง 100%
3. **SNMPv3 พร้อมใช้งานเต็มรูปแบบ** สำหรับ NOC-SRV ในการ poll สถานะและรับ trap แบบเข้ารหัส
   (AES) และยืนยันตัวตน (SHA) — ปิดช่องโหว่ของ community string แบบเก่า
4. **CPU ของทุกอุปกรณ์ปลอดภัยจาก log flooding** เพราะปิด console logging และพึ่งพา
   buffer + remote syslog เป็นหลัก

Three-legged Network Management Stack นี้ (NTP + Syslog + SNMP) คือรากฐานที่ทุก Part ถัดไป
ในหลักสูตร (ตั้งแต่ HSRP เป็นต้นไป) จะพึ่งพาในการ verify และ troubleshoot — เมื่อมี HSRP
failover เกิดขึ้นใน Part 18 เราจะใช้ syslog message และ SNMP trap ที่ config ไว้ใน Part นี้
ในการยืนยันว่า failover เกิดขึ้นจริงและเกิดขึ้นเมื่อไหร่

---

## แบบฝึกหัดทวนความเข้าใจ Part 17

1. ถ้า NOC-SRV เป็น NTP Stratum 2 และทุกอุปกรณ์ใน Lab sync เวลาจาก NOC-SRV โดยตรง อุปกรณ์เหล่านั้นจะกลายเป็น Stratum เท่าไหร่ และทำไม?
2. Syslog Severity Level ใดที่ใช้กับเหตุการณ์ปกติอย่าง Interface เปลี่ยนสถานะเป็น "up" (`%LINEPROTO-5-UPDOWN`) และถ้าตั้ง `logging trap warnings` จะได้รับ log ตั้งแต่ level ใดถึงระดับใด?
3. เพราะเหตุใดการเปิด Console Logging ในระดับ debug จึงเป็นอันตรายต่อ CPU ของอุปกรณ์ Production และควรแก้ไขด้วยคำสั่งอะไร?
4. อธิบายความแตกต่างระหว่าง SNMPv2c Community String กับ SNMPv3 USM ในด้านความปลอดภัย และเพราะเหตุใด SNMPv3 จึงจำเป็นสำหรับ Production?
5. SNMP Trap และ SNMP Polling (GET) แตกต่างกันอย่างไรในเชิงทิศทางการสื่อสารและข้อจำกัด และเพราะเหตุใด Production ที่ดีจึงควรใช้ทั้งสองแบบร่วมกัน?

**เฉลย:**
1. **Stratum 3** — เพราะกฎของ NTP คือ stratum จะเพิ่มขึ้น +1 ทุกครั้งที่ผ่าน network hop หนึ่งชั้นจากอุปกรณ์ stratum สูงกว่า (NOC-SRV stratum 2 + 1 hop = stratum 3 สำหรับอุปกรณ์ที่ sync จากมันโดยตรง)
2. Level **5 (Notification)** — และ `logging trap warnings` (level 4) จะได้รับ log ตั้งแต่ **level 0 (Emergency) ถึง level 4 (Warning)** เพราะการตั้ง trap level จะรวมทุก level ที่ severe กว่าหรือเท่ากันเสมอ (เลขน้อยกว่า = severe กว่า = ถูกรวมด้วย)
3. เพราะ Console port ทำงานช้ามาก (default 9600 bps) และ IOS ต้องรอให้ log message พิมพ์ออก console เสร็จก่อนทำงานอื่นต่อ (synchronous) หากมี log flooding (เช่น interface flap ถี่) จะทำให้ CPU ถูกใช้ไปกับการพิมพ์ log จนอุปกรณ์ควบคุมไม่ได้ — แก้ไขด้วย `no logging console` (ปิดไปเลย แนะนำสำหรับ Production) หรือลด level ด้วย `logging console warnings`
4. SNMPv2c ใช้ Community String ที่เป็น**plaintext ไม่เข้ารหัส**และไม่มี per-user authentication (ใครดัก packet อ่านได้ทันที) ส่วน SNMPv3 ใช้ **USM (User-based Security Model)** ที่มี per-user Authentication (SHA/MD5) และ Privacy/Encryption (AES/DES) ทำให้ยืนยันตัวตนผู้ query ได้จริงและป้องกันการดักอ่านข้อมูล — SNMPv3 จึงจำเป็นสำหรับ Production เพราะปิดช่องโหว่ที่ SNMPv1/v2c เปิดไว้ทั้งหมด
5. **Polling (GET)** เป็นทิศทาง NMS→Device (NMS ถามเข้ามาเป็นระยะ) เหมาะกับการทำ trend/graph แต่มี delay ระหว่างรอบ poll ส่วน **Trap** เป็นทิศทาง Device→NMS (อุปกรณ์แจ้งออกไปทันทีที่ event เกิด) เกือบ real-time แต่เป็น UDP-based ไม่รับประกันการส่งถึง (อาจหายได้) — Production ที่ดีจึงใช้ทั้งสองร่วมกัน: Trap สำหรับ alert แบบเร็ว และ Polling เป็น safety-net ที่จับความผิดปกติได้แม้ trap หายไป

---

## สรุป Part 17

Part นี้สร้างรากฐาน **Network Management** ให้กับทุกอุปกรณ์ใน Lab Topology ผ่านเซิร์ฟเวอร์
**NOC-SRV (10.10.99.50)** ที่ทำหน้าที่เป็นทั้ง NTP Stratum 2 Reference, Syslog Collector,
และ SNMP/NMS แบบรวมศูนย์ — เราครอบคลุมตั้งแต่แนวคิด NTP Stratum Hierarchy, การป้องกัน
time-spoofing ด้วย NTP Authentication, 8 Severity Level ของ Syslog, การป้องกัน CPU จาก
log flooding, ไปจนถึงการเปลี่ยนจาก SNMPv2c ที่ไม่ปลอดภัยไปสู่ SNMPv3 ที่มี Authentication
และ Encryption ครบถ้วน ✅ ทุกอุปกรณ์ใน Lab (CORE-SW1/2, DIST-SW1-4, WAN-EDGE-1/2,
ACCESS-SW1-4) ตอนนี้มีเวลาตรงกัน ส่ง log ไปที่เดียวกัน และพร้อมให้ NOC-SRV monitor ผ่าน
SNMPv3 ได้อย่างปลอดภัย

**พร้อมสำหรับ Part 18**: เราจะนำ Network Management Stack ที่สร้างไว้ใน Part นี้ไปใช้ยืนยันผลลัพธ์
ของ **FHRP (First Hop Redundancy Protocol)** — HSRP และ VRRP — ที่จะเพิ่ม Virtual IP (`.1`)
ให้กับทุก VLAN SVI บน DIST-SW1-4 ตามที่ระบุไว้ใน IP Address Plan พร้อมทดสอบ Failover จริง
และอ่านผลผ่าน Syslog/SNMP Trap ที่ config ไว้แล้ว

**ไปต่อ:** [Part 18 — FHRP: HSRP & VRRP →](part-018-fhrp.md)
