# Part 62 — Infra Services (DHCP/NTP/Syslog) Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 611–620 จาก 1000**

> ต่อจาก [Part 61 — Infrastructure Security Troubleshooting](part-061-infra-security-troubleshooting.md)
> ที่ไล่หาสาเหตุปัญหาฝั่ง AAA/ACL/CoPP/ZBFW มาแล้ว Part นี้จะพา Enterprise Engineer มาแก้ปัญหาใน
> อีกมุมที่ต่างออกไปโดยสิ้นเชิง — **Infrastructure Services** อย่าง DHCP, NTP, Syslog, SNMP และ
> DNS ที่สอนพื้นฐานไปแล้วใน [Part 16](part-016-dhcp-dns.md) และ [Part 17](part-017-ntp-syslog-snmp.md)
> ปัญหาของ Service กลุ่มนี้มีลักษณะเฉพาะที่ต่างจาก Routing Protocol หรือ Security Feature ที่เคย
> Troubleshoot มา — **มันมักไม่มี Error Message ชัดเจน ไม่มี Neighbor ที่ Flap ให้เห็น มีแต่
> "ความเงียบ" ที่ Function บางอย่างหายไปเฉยๆ** ทำให้ต้องใช้แนวทาง Checklist ที่เป็นระบบมากกว่า
> การไล่ตาม Error Message เหมือน Part ก่อนๆ — เราจะไล่ทีละ Scenario จริงที่พบได้ในสนามจริง ตั้งแต่
> DHCP ที่แจก IP ผิด, NTP ที่ไม่ยอม Sync, Syslog ที่หายไปกลางทาง, SNMP ที่ทำให้ NMS เข้าใจผิดว่า
> อุปกรณ์ล่ม, ไปจนถึงปัญหาที่ "คลื่นซ่อน" อยู่ใต้น้ำ — **Clock ที่ผิดเพราะ NTP ปัญหา ไปทำให้
> Feature ที่ดูไม่เกี่ยวกันเลยอย่าง Time-based ACL หรือ Certificate Validation พังตามไปด้วย**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 611 | Troubleshooting Methodology สำหรับ Infra Services — ทำไมต้องใช้ Checklist แทนการไล่ Error |
| 612 | Scenario 1 — VLAN ใหม่ไม่มี `ip helper-address` ทำให้ Client ไม่ได้ Lease เลย |
| 613 | Scenario 2 — Client ได้ Lease แต่ Gateway/DNS ผิด เพราะ Scope ไม่ได้ Update ตาม HSRP VIP |
| 614 | Scenario 3 — DHCP Pool Exhaustion เพราะ Excluded-address กว้างเกินไป |
| 615 | Scenario 4 — NTP ไม่ Sync: ACL บล็อก UDP/123 และ Authentication Key ไม่ตรงกัน |
| 616 | Scenario 5 — Syslog ไม่ถึงปลายทาง: Source-interface ผิด ACL และ Severity Filter แคบเกินไป |
| 617 | Scenario 6 — SNMP Polling ล้มเหลว: SNMPv3 User/Group ไม่ตรงกันหลัง Re-config |
| 618 | Scenario 7 — Router เอง Resolve DNS ไม่ได้: `ip domain-lookup` และ `ip name-server` |
| 619 | Scenario 8 — Clock ผิดเพราะ NTP ลาม (Cascade) ไปทำให้ EAP-TLS Certificate Validation ล้มเหลว |
| 620 | Lab เต็มรูปแบบ: 3 ปัญหาซ้อนกัน — Helper-address, NTP Key Mismatch, Time-based ACL ที่พังตาม |

---

## Step 611 — Troubleshooting Methodology สำหรับ Infra Services

### 611.1 ทำไม Infra Services ต่างจากปัญหาที่เคย Troubleshoot มา

ตลอดหลักสูตรที่ผ่านมา เวลา OSPF/EIGRP/BGP มีปัญหา เรามักเห็น **สัญญาณเตือนที่ชัดเจน**: Neighbor
Down, `%OSPF-5-ADJCHG ... from FULL to DOWN`, Interface Flap, STP Topology Change — Log
ฟ้องเราตรงๆ ว่าเกิดอะไรขึ้นและที่ไหน แต่ **DHCP, NTP, Syslog, SNMP, DNS มีพฤติกรรมต่างออกไป
โดยสิ้นเชิง**:

| Service | สิ่งที่เกิดเมื่อพัง | มี Error Message ชัดเจนไหม |
|---|---|---|
| DHCP | Client ไม่ได้ IP เลย หรือได้ IP ผิด — Switch/Router **ไม่รู้ตัวว่ามีปัญหา** เพราะไม่มี Client มาขอสำเร็จให้ Log อะไร | ❌ ไม่มี — Client Fallback เป็น APIPA เงียบๆ |
| NTP | Clock เดินต่อไปเรื่อยๆ ตามปกติ (แค่ไม่ Sync) — ไม่ Crash ไม่ Reboot | ⚠️ มีแค่ Passive State (`unsynchronized`) ต้องไปเช็คเอง |
| Syslog | Message หายไปกลางทาง — อุปกรณ์ต้นทางคิดว่าส่งสำเร็จแล้ว (`link up`) | ❌ ไม่มี — ฝั่งส่งไม่รู้ว่าฝั่งรับไม่ได้รับ (UDP ไม่ยืนยัน) |
| SNMP | NMS แสดงอุปกรณ์เป็น "Down" ทั้งที่ Ping ผ่านและ CLI ใช้งานได้ปกติ | ⚠️ Error อยู่ฝั่ง NMS ไม่ใช่ฝั่งอุปกรณ์ |
| DNS (บน Router เอง) | คำสั่งที่ต้อง Resolve ชื่อจะ Hang หรือ Fail แต่ Feature อื่นที่ไม่ใช้ DNS ทำงานปกติหมด | ⚠️ Error เจาะจงเฉพาะจุดที่เรียกใช้ DNS เท่านั้น |

> **หัวใจของ Step นี้**: เพราะ Infra Services เป็นปัญหาที่ **"เงียบ" (Silent Failure)** — ไม่มี
> Neighbor State ให้ดู ไม่มี Flap ให้นับ — วิศวกรที่ใช้วิธี "รอดู Error ใน Log แล้วไล่ตาม" แบบที่
> ใช้กับ Routing Protocol **จะไม่เจออะไรเลย** ต้องเปลี่ยนมาใช้ **แนวทาง Checklist เชิงระบบ**
> (Systematic Checklist) ที่ไล่ตรวจทุกจุดที่เป็นไปได้ทีละขั้นแทน

### 611.2 Checklist 6 ชั้นสำหรับ Infra Services ทุกตัว

ไม่ว่าจะเป็น DHCP, NTP, Syslog, หรือ SNMP — ปัญหาเกือบทั้งหมดอธิบายได้ด้วย Checklist เดียวกัน
6 ชั้นนี้ (ไล่จากง่ายสุดไปซับซ้อนสุดเสมอ):

```
ชั้น 1: Local Service Config          "Feature เปิดอยู่จริงไหม บน Device นี้"
        (ip helper-address, ntp server, logging host, snmp-server ...)
              │
ชั้น 2: Local Interface/IP Reachability "Ping ไปหา Server ปลายทางได้ไหม"
        (ตรวจ Routing, ตรวจ Source Interface ที่ใช้จริง)
              │
ชั้น 3: Path ACL / Firewall            "มี ACL ระหว่างทางบล็อก Port ที่ Service นี้ใช้ไหม"
        (UDP 67/68 DHCP, UDP 123 NTP, UDP 514 Syslog, UDP 161/162 SNMP)
              │
ชั้น 4: Server-side Config             "ฝั่ง Server ตั้งค่าตรงกับที่ Client คาดหวังไหม"
        (Scope/Pool ตรง Subnet, Key ตรงกัน, Severity Level ตรงกัน, User/Group ตรงกัน)
              │
ชั้น 5: Time/Clock Consistency         "Clock ของทุกฝั่งตรงกันไหม (Foundational Dependency)"
        (NTP พังจะลาม Cert, ACL Time-range, Log Correlation ทั้งหมด)
              │
ชั้น 6: Cross-verification            "เทียบกับอุปกรณ์ตัวอื่นที่ทำงานปกติ (Baseline)"
        (ถ้าอุปกรณ์ข้างเคียง Sync ได้ปกติ แต่ตัวนี้ไม่ได้ = ปัญหาเจาะจงที่ตัวนี้ ไม่ใช่ Server)
```

> **เทคนิคจำ**: ไล่จาก **"เปิดหรือยัง" → "ไปถึงหรือไม่" → "มีอะไรกั้นทางไหม" → "ปลายทางรับได้
> ไหม" → "เวลาตรงไหม" → "เทียบกับตัวอื่นแล้วต่างกันที่ไหน"** — Checklist นี้จะถูกใช้ซ้ำในทุก
> Scenario ของ Part นี้ เพื่อสร้างความเคยชินที่จะพาไปใช้ในข้อสอบ ENARSI จริงได้

### 611.3 กฎเหล็กที่ใช้ตลอด Part นี้: "อย่าเชื่อว่าฝั่งส่งบอกว่าสำเร็จ = ปลายทางได้รับจริง"

เพราะ DHCP Relay, Syslog, SNMP Trap ทั้งหมดวิ่งบน **UDP** (Connectionless, ไม่มี Acknowledgment
ระดับ Transport) — คำสั่งอย่าง `show logging` ที่บอกว่า `Logging to 10.10.99.50 ... link up`
**หมายความแค่ว่า Packet ถูกส่งออกจาก Interface สำเร็จเท่านั้น ไม่ได้ยืนยันว่า NOC-SRV ได้รับจริง**
— นี่คือกับดักที่ทำให้วิศวกรมือใหม่เข้าใจผิดบ่อยที่สุดใน Step 616 ของ Part นี้ ต้อง **ตรวจสอบที่
ปลายทางเสมอ** (เช่น เปิด `debug ip udp` ที่ Server, หรือใช้ Packet Capture) ไม่ใช่เชื่อ Output
ฝั่งต้นทางเพียงอย่างเดียว

---

## Step 612 — Scenario 1: VLAN ใหม่ไม่มี `ip helper-address` ทำให้ Client ไม่ได้ Lease เลย

### 612.1 Background

ทีม Network เพิ่ง Provision **VLAN 60 (R&D-LAB)** เพิ่มเข้ามาให้ทีม R&D ที่ขยายจำนวนคนอย่างรวดเร็ว
(Subnet `10.10.60.0/24` — **เป็น VLAN สาธิตเฉพาะ Scenario นี้ ไม่ได้เพิ่มเข้า
[IP Address Plan หลัก](00-ip-address-plan.md)** เช่นเดียวกับ VLAN 50 Contractor ใน
[Part 45](part-045-advanced-acl-zbfw.md)) โดยตั้ง SVI คู่ Redundant แบบเดียวกับ VLAN 10 SALES
บน **DIST-SW1 (`10.10.60.2`)** และ **DIST-SW2 (`10.10.60.3`)** พร้อม Trunk ลงไปยัง ACCESS-SW1/2
เรียบร้อยแล้ว — ทุกอย่างดูเหมือนพร้อม แต่ Helpdesk แจ้งเข้ามาว่า **"PC ทีม R&D ทุกเครื่องขึ้น
'Obtaining IP Address...' ค้างอยู่ ต่อ Internet ไม่ได้เลย"**

### 612.2 อาการ (Symptom)

```
PC-RND-01> ipconfig /all
Windows IP Configuration
   Autoconfiguration Enabled: Yes
   Autoconfiguration IPv4 Address. . : 169.254.31.204
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :
```

`169.254.x.x` คือ **APIPA** — สัญญาณคลาสสิกที่บอกว่า Client ยิง `DHCPDISCOVER` ออกไปแล้ว
**ไม่มีใครตอบเลย** (ตามที่อธิบายไว้ใน [Part 16 Step 151.5](part-016-dhcp-dns.md))

### 612.3 การไล่ตาม Checklist (Step 611.2)

**ชั้น 1 — Local Service Config**: ตรวจ SVI ที่เป็น Gateway ของ VLAN 60

```
DIST-SW1# show running-config interface vlan 60
interface Vlan60
 description ** Gateway VLAN60-RND-LAB **
 ip address 10.10.60.2 255.255.255.0
end
```

สังเกตว่า **ไม่มีบรรทัด `ip helper-address` เลย** — เทียบกับ VLAN 10 ที่ต้องมี
`ip helper-address 10.10.30.20` ตามที่ Config ไว้ใน [Part 16 Step 153.4](part-016-dhcp-dns.md)
ทันทีที่เห็นสิ่งนี้ก็สามารถสรุปสาเหตุได้เกือบจะแน่นอนแล้ว แต่ยังต้องตรวจ DIST-SW2 คู่กันด้วย
เพราะ VLAN นี้มี Gateway 2 จุด

```
DIST-SW2# show running-config interface vlan 60
interface Vlan60
 description ** Gateway VLAN60-RND-LAB **
 ip address 10.10.60.3 255.255.255.0
end
```

เหมือนกัน — **ไม่มี `ip helper-address` ทั้งสองตัว** ยืนยันว่าตอนสร้าง VLAN ใหม่ ทีมงานทำตาม
Checklist การทำ Inter-VLAN Routing แบบเดิมจาก [Part 10](part-010-inter-vlan-routing.md) (สร้าง
SVI, ใส่ IP, ผูก VLAN ลง Trunk) แต่ **ลืม Step ที่มาใหม่กว่าจาก Part 16** ไปเลย — เป็นรูปแบบ
ความผิดพลาดที่พบบ่อยที่สุดตอน Provision VLAN ใหม่ในสนามจริง เพราะ Checklist เดิมที่ทีมงานคุ้นเคย
ไม่ได้ถูก Update ให้รวม Step ใหม่ๆ เข้าไปด้วย

### 612.4 ยืนยันสาเหตุด้วย `debug ip udp` (ทางเลือกเสริมเพื่อความมั่นใจ)

```
DIST-SW1# debug ip udp
IP UDP debugging is on
DIST-SW1#
*Sep 26 14:02:11.881: UDP: rcvd src=0.0.0.0(68), dst=255.255.255.255(67), length=328
*Sep 26 14:02:11.881: UDP: rcvd src=0.0.0.0(68), dst=255.255.255.255(67), length=328
DIST-SW1# undebug all
```

DIST-SW1 **เห็น** `DHCPDISCOVER` เข้ามาที่ VLAN 60 จริง (แปลว่า Layer 2/VLAN ปกติ ไม่มีปัญหา
Trunk/Access Port) แต่เพราะไม่มี `ip helper-address` ผูกกับ Interface ที่รับ Broadcast นี้
IOS จึงแค่ **Drop Broadcast นั้นไปเงียบๆ ตามพฤติกรรม Default ของ Router ที่ไม่ Forward
Broadcast ข้าม VLAN** (ตามกฎที่อธิบายไว้ใน Part 16 Step 151.5) — ไม่มีการ Log อะไรเลยเพราะ
นี่ไม่ใช่ "Error" ในมุมของ IOS แต่เป็น "พฤติกรรม Default ที่ทำงานตามที่ควรจะเป็น"

### 612.5 Root Cause และการแก้ไข

**Root Cause**: SVI ของ VLAN 60 บนทั้ง DIST-SW1 และ DIST-SW2 ไม่มี `ip helper-address` ชี้ไปยัง
DHCP-DNS-SRV (`10.10.30.20`) — เป็นการตกหล่นตอน Provision VLAN ใหม่ ไม่ใช่ Bug หรือ Hardware
ปัญหาแต่อย่างใด

```
DIST-SW1(config)# interface vlan 60
DIST-SW1(config-if)# ip helper-address 10.10.30.20
DIST-SW1(config-if)# exit

DIST-SW2(config)# interface vlan 60
DIST-SW2(config-if)# ip helper-address 10.10.30.20
DIST-SW2(config-if)# exit
```

### 612.6 ยืนยันผลลัพธ์

```
DIST-SW1# show ip interface vlan 60 | include Helper
  Helper address is 10.10.30.20

DIST-SW1# show ip dhcp binding
IP address       Client-ID/Hardware address     Lease expiration        Type
10.10.60.21      0100.5056.aa11.bb              Sep 27 2026 08:12 AM    Automatic
```

> **ข้อสังเกตสำคัญ**: `show ip dhcp binding` ยืนยัน Binding ได้ก็ต่อเมื่อ Device นั้น **เป็น
> DHCP Server เอง** — กรณีนี้ใช้ Relay ไปหา Server ภายนอก การยืนยันที่แท้จริงต้องเช็คที่ตัว
> Server ปลายทางโดยตรง (`show dhcp lease` บน ISC-DHCP) แต่ Client ที่ได้ IP จริงคือหลักฐาน
> สุดท้ายที่สำคัญที่สุดเสมอ

> **ข้อสอบ ENARSI มักถามแนวนี้**: "Client ใน VLAN ใหม่ไม่ได้ IP เลย แต่ VLAN อื่นปกติทุกอย่าง"
> — คำตอบส่วนใหญ่คือ `ip helper-address` ตกหล่นตอน Provision เพราะเป็นสาเหตุที่พบบ่อยที่สุด
> และไม่มี Error Message ใดๆ เตือนให้รู้ตัว

---

## Step 613 — Scenario 2: Client ได้ Lease แต่ Gateway/DNS ผิด เพราะ Scope ไม่ตาม HSRP VIP

### 613.1 Background

VLAN 10 (SALES) ใช้ DHCP-DNS-SRV กลางผ่าน Relay ตามที่ Config ไว้ใน
[Part 16 Step 153.4](part-016-dhcp-dns.md) — ตอนนั้น Default Gateway ที่แจกให้ Client ยังเป็น
**Real IP** ของ DIST-SW1/DIST-SW2 (`10.10.10.2`/`.3`) เพราะยังไม่มี HSRP ต่อมาใน
[Part 18](part-018-fhrp.md) เราเพิ่ม **HSRP Virtual IP `10.10.10.1`** เข้ามาเป็น Gateway ตัวจริง
ที่ทุก Client ควรใช้ — แต่ทีมงานที่ Deploy HSRP ตอนนั้น **แก้ Config บน DIST-SW1/DIST-SW2 (Router)
สำเร็จ แต่ไม่ได้แจ้งทีม Server ให้ไป Update Scope Option บน DHCP-DNS-SRV ตามด้วย**

### 613.2 อาการ (Symptom)

```
PC-SALES-14> ipconfig /all
   IPv4 Address. . . . . . . . . . . : 10.10.10.87
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.10.2
   DNS Servers . . . . . . . . . . . : 10.10.30.20
```

PC ได้ IP ปกติ (`10.10.10.87`) — DHCP ทำงาน "สำเร็จ" ในสายตา Client แต่ **Default Gateway
ที่ได้คือ `10.10.10.2` (Real IP เดิมของ DIST-SW1) ไม่ใช่ `10.10.10.1` (HSRP VIP)** — ผลลัพธ์คือ
Traffic ทั้งหมดของ PC เครื่องนี้จะพึ่งพา DIST-SW1 ตัวเดียวเสมอ **ไม่มี Redundancy ตามที่ HSRP
ควรให้ได้เลย** — ถ้า DIST-SW1 ล้มหรือ Reload PC เครื่องนี้จะขาดการเชื่อมต่อทันที ทั้งที่ DIST-SW2
ยังทำงานปกติและ HSRP ควร Failover ให้ได้ (แต่ Client ไม่รู้จัก VIP เลยไม่ Failover ตามไปด้วย)

> **จุดที่อันตรายที่สุดของ Scenario นี้**: นี่**ไม่ใช่ปัญหาที่ทำให้ Connectivity ขาดทันที** —
> Client ใช้งานได้ปกติตอนที่ DIST-SW1 ทำงานอยู่ ปัญหานี้จึง **แอบซ่อนอยู่ได้เป็นเดือน** จนกว่าจะ
> มีเหตุการณ์ DIST-SW1 Down จริงถึงจะรู้ตัวว่า Redundancy ที่ตั้งใจทำไว้ไม่ทำงานสำหรับ Client
> กลุ่มนี้เลย

### 613.3 การไล่ตาม Checklist

**ชั้น 6 — Cross-verification ก่อนเลย** (ในกรณีนี้เร็วกว่าไล่ตามลำดับปกติ): ลองเทียบกับ
Client ที่เพิ่งเปิดเครื่องใหม่ (Renew Lease ใหม่ล่าสุด) กับ Client ที่เปิดมานานแล้ว — พบว่า
**ทุกเครื่องได้ Gateway `10.10.10.2` เหมือนกันหมด ไม่มีเครื่องไหนได้ `.1` เลย** แปลว่าไม่ใช่
เรื่อง Cache/Lease เก่าค้าง แต่เป็นค่าที่ Scope กำลังแจกออกมาแบบนี้จริงๆ

**ยืนยัน HSRP ทำงานถูกต้องแล้วบนฝั่ง Router**:

```
DIST-SW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   110 P Active   local           10.10.10.3      10.10.10.1
```

HSRP ทำงานสมบูรณ์ — `Virtual IP` คือ `10.10.10.1` ตามที่ตั้งใจ ปัญหาไม่ได้อยู่ที่ฝั่ง Router
เลย **ปัญหาอยู่ฝั่ง Scope Configuration ของ DHCP-DNS-SRV**

**ตรวจ Scope Option ที่ Server จริง** (DHCP-DNS-SRV เป็น ISC-DHCP ตามตัวอย่างใน
[Part 16 Step 154.2](part-016-dhcp-dns.md)):

```
# /etc/dhcp/dhcpd.conf บน DHCP-DNS-SRV
subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.20 10.10.10.250;
  option routers 10.10.10.2;                  # <-- ค่าเก่าก่อน HSRP ยังไม่ถูกแก้!
  option domain-name-servers 10.10.30.20;
  option domain-name "lab.local";
  default-lease-time 691200;
  max-lease-time 691200;
}
```

พบ `option routers 10.10.10.2` ยังเป็นค่าเดิมตั้งแต่ก่อน Part 18 — นี่คือ Root Cause ที่ชัดเจน
ที่สุด: **ทีม Network เปลี่ยน Gateway จริงบน Router สำเร็จ แต่ไม่มีใครไป Sync ค่านี้กับทีม Server**
ซึ่งเป็นปัญหา Classic ของ Config Management ข้าม Team ในองค์กรจริง (Router ทีมหนึ่งดูแล, DHCP
Server อีกทีมดูแล — Communication Gap ระหว่าง Change)

### 613.4 Root Cause และการแก้ไข

**Root Cause**: Scope Option `routers` บน DHCP-DNS-SRV ยังชี้ไปที่ Real IP เดิม (`10.10.10.2`)
ไม่ได้ Update ตาม HSRP Virtual IP (`10.10.10.1`) ที่เพิ่มเข้ามาใน Part 18

```
# แก้ไขบน DHCP-DNS-SRV
subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.20 10.10.10.250;
  option routers 10.10.10.1;                  # <-- แก้เป็น HSRP VIP
  option domain-name-servers 10.10.30.20;
  option domain-name "lab.local";
  default-lease-time 691200;
  max-lease-time 691200;
}
```

```
root@dhcp-dns-srv:~# systemctl restart isc-dhcp-server
root@dhcp-dns-srv:~# systemctl status isc-dhcp-server
● isc-dhcp-server.service - ISC DHCP IPv4 server
     Active: active (running)
```

> **ทำแบบเดียวกันกับทุก Scope ที่ Redundant ด้วย HSRP**: VLAN 40 (WIFI) ก็ต้องตรวจสอบและแก้
> `option routers` เป็น `10.10.40.1` ด้วยเช่นกัน — ส่วน VLAN 20 (VOICE) ที่ใช้ Local Pool บน
> Cisco IOS เอง (Step 152) ก็ต้องเปลี่ยน `default-router` ให้เหลือแค่ `10.10.20.1` (VIP เดียว)
> ตามที่ [Part 16 Step 152.3](part-016-dhcp-dns.md) เตือนไว้ล่วงหน้าแล้วว่าค่านี้จะต้องถูกเปลี่ยน
> ตอนมี HSRP เข้ามา

### 613.5 ยืนยันผลลัพธ์บน Client

```
PC-SALES-14> ipconfig /release
PC-SALES-14> ipconfig /renew
PC-SALES-14> ipconfig /all
   IPv4 Address. . . . . . . . . . . : 10.10.10.92
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.10.1
   DNS Servers . . . . . . . . . . . : 10.10.30.20
```

Gateway เปลี่ยนเป็น `10.10.10.1` (VIP) ถูกต้องแล้ว — ทดสอบ Failover จริงโดย Shutdown DIST-SW1
ชั่วคราวและยืนยันว่า PC ยัง Ping ผ่าน Gateway ได้ต่อเนื่องผ่าน DIST-SW2 (HSRP Active ใหม่) โดยไม่
Drop Connectivity เลย — พิสูจน์ว่า Redundancy ทำงานสมบูรณ์ตามที่ตั้งใจแล้ว

> **บทเรียน**: ทุกครั้งที่เปลี่ยน Gateway/VIP ของ VLAN ใด ต้องเพิ่ม "ตรวจสอบ/แก้ Scope DHCP
> ที่เกี่ยวข้อง" เป็นหนึ่งใน Change Checklist เสมอ — เป็นจุดที่ตกหล่นบ่อยเพราะ DHCP Server
> มักอยู่นอกความรับผิดชอบของทีม Network โดยตรง

---

## Step 614 — Scenario 3: DHCP Pool Exhaustion เพราะ Excluded-address กว้างเกินไป

### 614.1 Background

VLAN 20 (VOICE) ใช้ Cisco IOS เป็น DHCP Server เอง (Local Pool `VOICE-PHONES` บน DIST-SW2)
ตามที่ตั้งไว้ใน [Part 16 Step 152.3](part-016-dhcp-dns.md) — ตอนนั้น Exclude ไว้แค่
`10.10.20.1 – 10.10.20.9` (กันไว้ให้ Gateway/VIP และ Infra สำรอง) เหลือ Pool ใช้งานจริง
`10.10.20.10 – 10.10.20.254` (245 Address) ซึ่งเพียงพอมากสำหรับ IP Phone ที่มีอยู่ตอนนั้น
(~60 เครื่อง)

ต่อมาทีม Facility เพิ่ม Access Point และ Printer เครือข่ายเข้ามาในตึกอีกหลายตัว วิศวกรที่ดูแล
(คนละคนกับที่ Config ตอนแรก) ต้องการกันที่อยู่ Static เพิ่มให้อุปกรณ์เหล่านี้ แต่ **กันไว้แบบ
"เผื่อไว้ก่อนเยอะๆ กันปัญหา" โดยไม่ได้คำนวณจำนวนจริง**:

```
DIST-SW2(config)# ip dhcp excluded-address 10.10.20.1 10.10.20.200
```

คำสั่งนี้ **แทนที่** Exclude เดิม (`.1-.9`) ด้วย Range ใหม่ที่กว้างกว่ามาก (`.1-.200`) — เหลือ
Pool ใช้งานจริงแค่ `10.10.20.201 – 10.10.20.254` (**54 Address เท่านั้น**) ในขณะที่บริษัทมี
IP Phone เพิ่มขึ้นเป็น 150 เครื่องไปแล้วในช่วงเวลาเดียวกัน (ขยายสาขา/เพิ่มพนักงาน)

### 614.2 อาการ (Symptom)

Helpdesk แจ้งว่า **"IP Phone ตัวใหม่ที่เพิ่งติดตั้งไม่ยอม Register เลย ขึ้นจอ 'Configuring IP'
ค้าง"** ในขณะที่ IP Phone ตัวเก่าที่เสียบมาก่อนหน้ายังใช้งานได้ปกติทุกตัว — สัญญาณนี้ต่างจาก
Scenario 612 ตรงที่ **ไม่ใช่ทุกเครื่องมีปัญหา มีแค่เครื่องที่มาขอ Lease ใหม่/หลังหมด Lease
เท่านั้น**

### 614.3 การไล่ตาม Checklist

**ชั้น 1 (Local Service Config) ผ่านหมด** — Pool มีอยู่จริง, `ip helper-address` ไม่เกี่ยวเพราะ
เป็น Local Pool อยู่แล้ว, ตรวจ Layer 2/VLAN ก็ปกติ — ต้องขยับไปดู **สถานะ Utilization ของ Pool**
โดยตรงด้วยคำสั่งเฉพาะของ DHCP Server บน IOS:

```
DIST-SW2# show ip dhcp pool VOICE-PHONES

Pool VOICE-PHONES :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 54
 Pending event                  : none
 Excluded addresses             : 200
 Available addresses            : 0

  Subnet address                : 10.10.20.0
  Mask                          : 255.255.255.0
  Default router                : 10.10.20.1
  DNS Server                    : 10.10.30.20
  Domain name                   : lab.local
  Lease time                    : 7 days 0 hours 0 minutes
```

**`Available addresses : 0`** และ **`Leased addresses : 54`** (เท่ากับ Pool ที่เหลือจริงพอดี
— 54 Address ถูกใช้ครบ 100%) คือหลักฐานที่ชัดเจนที่สุด: **Pool Exhaustion เต็มแล้ว** — IP Phone
เครื่องใหม่ที่มายิง `DHCPDISCOVER` จะได้รับ**ไม่มี `DHCPOFFER` เลย**เพราะไม่มี Address เหลือให้แจก

**ยืนยันด้วย `show ip dhcp conflict` (ตรวจว่าไม่ใช่ IP Conflict ปนอยู่ด้วย)**:

```
DIST-SW2# show ip dhcp conflict
IP address       Detection method    Detection time            VRF
% No conflict found
```

ไม่มี Conflict — ยืนยันว่าเป็นปัญหา Exhaustion ล้วนๆ ไม่ได้มี IP ซ้ำซ้อนแอบแฝงอยู่

### 614.4 Root Cause และการแก้ไข

**Root Cause**: `ip dhcp excluded-address 10.10.20.1 10.10.20.200` กันที่อยู่กว้างเกินความ
จำเป็นจริง (กันไว้ 200 Address แต่ Static Device จริงมีไม่ถึง 20 ตัว) ทำให้ Pool ใช้งานจริงเหลือ
แค่ 54 Address ซึ่งน้อยกว่าจำนวน IP Phone จริงที่มี (150 เครื่อง) มาก

**แนวทางแก้ไขที่ถูกต้อง**: กลับไปสำรวจจำนวนอุปกรณ์ Static จริงที่ต้องกัน แล้วกำหนด Exclude
Range ให้พอดีกับความต้องการจริง ไม่ใช่ "เผื่อไว้เยอะๆ" แบบไม่มีตัวเลขอ้างอิง:

```
DIST-SW2(config)# no ip dhcp excluded-address 10.10.20.1 10.10.20.200
DIST-SW2(config)# ip dhcp excluded-address 10.10.20.1 10.10.20.29
! กันไว้แค่ 29 Address (Gateway/VIP .1, SVI จริง .2-.3, Static AP/Printer ที่มีจริงไม่เกิน 25 ตัว)
```

### 614.5 ยืนยันผลลัพธ์

```
DIST-SW2# show ip dhcp pool VOICE-PHONES

Pool VOICE-PHONES :
 Total addresses                : 254
 Leased addresses               : 54
 Excluded addresses             : 29
 Available addresses            : 171
```

**`Available addresses : 171`** — เหลือ Pool เพียงพอสำหรับ IP Phone ที่เหลือรวมถึงการเติบโต
ในอนาคตอันใกล้ ทดสอบเสียบ IP Phone เครื่องใหม่และยืนยันว่า Register สำเร็จภายในไม่กี่วินาที

> **Best Practice ป้องกันปัญหานี้ในอนาคต**: ตั้ง **`utilization mark high <percent>`** ผูกกับ
> SNMP Trap เพื่อให้ NOC-SRV แจ้งเตือนล่วงหน้าก่อน Pool เต็มจริง:
> ```
> DIST-SW2(dhcp-config)# utilization mark high 80
> DIST-SW2(dhcp-config)# utilization mark low 60
> ```
> ค่านี้ทำให้ IOS Generate Syslog เตือนอัตโนมัติเมื่อ Pool ใช้งานเกิน 80% ผ่าน Central Syslog
> จาก [Part 17](part-017-ntp-syslog-snmp.md) — รู้ตัว **ก่อน** Pool เต็มจริง ไม่ใช่รู้จาก Ticket

---

## Step 615 — Scenario 4: NTP ไม่ Sync — ACL บล็อก UDP/123 และ Authentication Key ไม่ตรงกัน

### 615.1 Sub-scenario A: ACL บล็อก UDP/123 บน DIST-SW2

**Background**: ทีม Security เพิ่ม ACL รัดกุมขึ้นบน SVI VLAN 99 (MGMT) ของ DIST-SW2 ตามแนวทาง
Infrastructure ACL Hardening (คล้ายกับที่ [Part 61](part-061-infra-security-troubleshooting.md)
พูดถึง) เพื่อจำกัดว่ามีแค่ Traffic จาก NOC-SRV เท่านั้นที่เข้าถึง Management Plane ได้ — แต่
ตอนเขียน ACL อนุญาตแค่ SSH และ SNMP ไป **ลืมอนุญาต NTP (UDP/123)**

```
DIST-SW2(config)# ip access-list extended INFRA-HARDENING
DIST-SW2(config-ext-nacl)# permit tcp host 10.10.99.50 host 10.10.99.12 eq 22
DIST-SW2(config-ext-nacl)# permit udp host 10.10.99.50 host 10.10.99.12 eq 161
DIST-SW2(config-ext-nacl)# deny ip any any log
DIST-SW2(config-ext-nacl)# exit
DIST-SW2(config)# interface vlan 99
DIST-SW2(config-if)# ip access-group INFRA-HARDENING in
DIST-SW2(config-if)# exit
```

**อาการ**: ทุกอุปกรณ์อื่นใน Lab Sync เวลาปกติ มีแค่ **DIST-SW2** ที่ Clock ไม่ตรง

```
DIST-SW2# show ntp status
Clock is unsynchronized, stratum 16, no reference clock
nominal freq is 250.0000 Hz, actual freq is 250.0000 Hz, precision is 2**18
ntp uptime is 892400 (1/100 of seconds), resolution is 4000
```

`stratum 16` และ `unsynchronized` — ยืนยันตามที่อธิบายไว้ใน
[Part 17 Step 161](part-017-ntp-syslog-snmp.md) ว่าคือค่า Default ก่อน Sync สำเร็จ

**ไล่ตาม Checklist**: ชั้น 1 (Config `ntp server 10.10.99.50` มีอยู่ปกติ) → ชั้น 2 (Ping
`10.10.99.50` จาก DIST-SW2 ผ่านปกติ เพราะ ICMP ไม่ได้ถูกบล็อก) → **ชั้น 3 (Path ACL) พบสาเหตุ**:

```
DIST-SW2# show access-lists INFRA-HARDENING
Extended IP access list INFRA-HARDENING
    10 permit tcp host 10.10.99.50 host 10.10.99.12 eq 22 (12 matches)
    20 permit udp host 10.10.99.50 host 10.10.99.12 eq 161 (340 matches)
    30 deny ip any any log (528 matches)
```

**`deny ip any any log` มี 528 matches** และเพิ่มขึ้นเรื่อยๆ ทุก 64 วินาที (รอบ Poll ของ NTP) —
เพราะ Response ของ NTP (UDP/123) จาก NOC-SRV กลับเข้ามาที่ DIST-SW2 ถูก ACL Line 30 Deny ทั้งหมด
(ไม่มี Line ใดอนุญาต UDP/123 เลย)

**Root Cause & Fix**:

```
DIST-SW2(config)# ip access-list extended INFRA-HARDENING
DIST-SW2(config-ext-nacl)# 15 permit udp host 10.10.99.50 host 10.10.99.12 eq 123
DIST-SW2(config-ext-nacl)# exit
```

```
DIST-SW2# show ntp status
Clock is synchronized, stratum 3, reference is 10.10.99.50
```

### 615.2 Sub-scenario B: Authentication Key ไม่ตรงกันบน WAN-EDGE-2

**Background**: ทีม Security กำหนดให้ Rotate NTP Authentication Key ประจำปีตามนโยบาย —
Key ID `1` เปลี่ยน Key String จาก `NtpAuthKey2026!` เป็น `NtpAuthKey2027Secure!` บนทุกอุปกรณ์
รวมถึง NOC-SRV — ตอน Copy-Paste Config ไปยัง WAN-EDGE-2 เกิดการพิมพ์ผิดตัวพิมพ์เล็ก/ใหญ่
(Case-sensitive Key String)

```
WAN-EDGE-2(config)# ntp authentication-key 1 md5 ntpauthkey2027secure!
```

(ตัวพิมพ์เล็กทั้งหมด — ต่างจาก NOC-SRV และอุปกรณ์อื่นที่ใช้ `NtpAuthKey2027Secure!`)

**อาการ**: `show ntp status` บน WAN-EDGE-2 ค้างที่ Reference เดิม (Time source ไม่เปลี่ยนเป็น
NTP ใหม่ หรือ Drift ไปเรื่อยๆ) ต่างจากอุปกรณ์ทุกตัวที่ Sync ปกติ

**ไล่ตาม Checklist**: ชั้น 1-3 ผ่านหมด (Config มี, Ping ผ่าน, ไม่มี ACL กั้น) → ต้องขยับไปดู
Authentication โดยตรงตามที่ [Part 17 Step 163](part-017-ntp-syslog-snmp.md) สอนไว้:

```
WAN-EDGE-2# show ntp associations detail
10.10.99.50 configured, insane, invalid, unauthenticated, stratum 16
ref ID .INIT., time 00000000.00000000 (never)
our mode client, peer mode unspec, our poll intvl 64, peer poll intvl 1024
root delay 0.00 msec, root disp 0.00, reach 0, sync dist 0.00
```

**`unauthenticated`** คือคำตอบที่ชัดเจนที่สุด — Key ไม่ตรงกันระหว่าง WAN-EDGE-2 กับ NOC-SRV
ยืนยันด้วยการเทียบ Running-config ตรงๆ:

```
WAN-EDGE-2# show running-config | include ntp authentication-key
ntp authentication-key 1 md5 0701104C0A18 ,17 scrambled
```

> Key String บน IOS จะถูก Encrypt (Type 7) ในการแสดงผลเสมอ ไม่สามารถเทียบ Plaintext ตรงๆ
> จาก `show run` ได้ — วิธีตรวจสอบที่แน่นอนกว่าคือ **Config ใหม่ทับด้วย Key String ที่ทีม
> Security ยืนยันว่าถูกต้อง** แทนการพยายามเทียบ Encrypted String ด้วยตา

**Root Cause & Fix**:

```
WAN-EDGE-2(config)# no ntp authentication-key 1 md5 ntpauthkey2027secure!
WAN-EDGE-2(config)# ntp authentication-key 1 md5 NtpAuthKey2027Secure!
WAN-EDGE-2(config)# ntp trusted-key 1
WAN-EDGE-2(config)# end

WAN-EDGE-2# show ntp associations detail
10.10.99.50 configured, authenticated, our_master, sane, valid, stratum 2
```

`authenticated` และ `valid` กลับมาแล้ว — Sync สำเร็จ

> **บทเรียนรวม**: NTP ไม่ Sync มีได้แค่ 2 กลุ่มสาเหตุหลักเสมอ — **"ไปไม่ถึง"** (Routing/ACL —
> Sub-scenario A) กับ **"ไปถึงแต่ไม่ได้รับความเชื่อถือ"** (Authentication Mismatch —
> Sub-scenario B) — `show ntp associations detail` แยกสองกลุ่มนี้ได้ทันที

---

## Step 616 — Scenario 5: Syslog ไม่ถึงปลายทาง — Source-interface ผิด ACL และ Severity Filter แคบเกินไป

### 616.1 Sub-scenario A: `logging source-interface` ชี้ผิด Interface ที่ไม่ผ่าน ACL

**Background**: วิศวกรกำลัง Troubleshoot ปัญหาอื่นบน WAN-EDGE-2 และลองเปลี่ยน
`logging source-interface` ไปเป็น Physical Interface ชั่วคราวเพื่อทดสอบอะไรบางอย่าง แล้ว
**ลืมเปลี่ยนกลับเป็น Loopback0**

```
WAN-EDGE-2(config)# logging source-interface GigabitEthernet0/0/1
```

ในขณะเดียวกัน ทีม Security มี ACL Management-plane Hardening บน CORE-SW2 (จุดที่ Traffic
ต้องผ่านไปยัง VLAN 99/NOC-SRV) ที่ **อนุญาตเฉพาะ Traffic จาก Loopback Range (`1.1.1.0/24`)
เท่านั้น** สำหรับ Service ฝั่ง Management — Best Practice ที่ถูกต้อง (Loopback คงที่กว่า
Physical Interface) แต่กลายเป็นปัญหาเมื่อ Source-interface ของ WAN-EDGE-2 ไม่ตรงกับ Policy นี้

```
CORE-SW2# show ip access-lists MGMT-PLANE-ACL
Extended IP access list MGMT-PLANE-ACL
    10 permit udp 1.1.1.0 0.0.0.255 host 10.10.99.50 eq 514
    20 permit udp 1.1.1.0 0.0.0.255 host 10.10.99.50 eq 123
    30 permit udp 1.1.1.0 0.0.0.255 host 10.10.99.50 eq 161
    40 deny ip any any log (91 matches)
```

**อาการ**: WAN-EDGE-2 มองว่า Syslog ทำงานปกติทุกอย่าง — นี่คือกับดักที่อธิบายไว้ใน
Step 611.3 พอดี:

```
WAN-EDGE-2# show logging | include Trap logging|Logging to
    Trap logging: level informational, 892 message lines logged
        Logging to 10.10.99.50 (udp port 514, audit disabled,
              link up), 892 message lines logged, 0 message lines rate-limited,
              0 message lines dropped-by-MD
```

`link up` และ Message Count เพิ่มขึ้นเรื่อยๆ — **ดูเหมือนทุกอย่างสำเร็จ** แต่ที่ NOC-SRV
(Syslog Collector) **ไม่มี Log จาก WAN-EDGE-2 เข้ามาเลยตั้งแต่เมื่อคืน** — Helpdesk/NOC สังเกต
เห็นความผิดปกตินี้ตอนพยายามไล่ Log เพื่อ Correlate เหตุการณ์ Failover ที่เกิดขึ้นบน WAN-EDGE-2

**ไล่ตาม Checklist**: ชั้น 1 (Config มี, ดูเหมือนปกติ) → ชั้น 2 (Ping NOC-SRV จาก WAN-EDGE-2
ผ่านปกติ) → **ชั้น 3 (Path ACL) ต้องเช็คบน CORE-SW2 ที่อยู่กลางทาง** ไม่ใช่แค่บน WAN-EDGE-2
เอง:

```
CORE-SW2# show running-config | include logging source-interface
! (ไม่มี Output — เพราะคำสั่งนี้ต้องดูที่ WAN-EDGE-2)

WAN-EDGE-2# show running-config | include logging source-interface
logging source-interface GigabitEthernet0/0/1
```

พบว่า Source ที่ใช้จริงคือ IP ของ `GigabitEthernet0/0/1` (Transit Interface, `10.10.254.6`)
ไม่ใช่ Loopback0 (`1.1.1.22`) ที่ ACL บน CORE-SW2 อนุญาตไว้ — Traffic ถูก Deny ที่ Line 40
(matches เพิ่มขึ้นตรงกับจำนวน Syslog Message ที่ WAN-EDGE-2 พยายามส่งพอดี)

**Root Cause & Fix**:

```
WAN-EDGE-2(config)# no logging source-interface GigabitEthernet0/0/1
WAN-EDGE-2(config)# logging source-interface Loopback0
```

```
CORE-SW2# show ip access-lists MGMT-PLANE-ACL | include matches
    10 permit udp 1.1.1.0 0.0.0.255 host 10.10.99.50 eq 514 (5 matches)
```

Matches เริ่มขึ้นที่ Line 10 (Permit) แทน Line 40 (Deny) — และ NOC-SRV เริ่มเห็น Log จาก
WAN-EDGE-2 เข้ามาใหม่ทันที

### 616.2 Sub-scenario B: `logging trap` แคบเกินไปบน DIST-SW4

**Background**: ทีมงานทำ "โปรเจกต์ลด Log Noise" บน DIST-SW4 เพราะบ่นว่า Central Syslog มี
Message เข้ามามากเกินไปจนหา Log สำคัญยาก จึงรัด Severity ให้แคบลง:

```
DIST-SW4(config)# logging trap errors
```

(Level 3 — Error) **โดยไม่รู้ว่า `errors` (Level 0-3) จะตัด Level 4-6 ออกไปทั้งหมด** ซึ่งรวมถึง
`%LINEPROTO-5-UPDOWN` (Interface Up/Down), `%HSRP-5-STATECHANGE` (HSRP Failover),
`%OSPF-5-ADJCHG` (OSPF Adjacency Change) — Message ที่สำคัญที่สุดสำหรับ Troubleshoot
กลับถูกกรองออกไปด้วย

**อาการ**: HSRP บน DIST-SW4 Failover จริง (Uplink หลุดชั่วคราว) แต่ตอนทีมงานไปไล่ Central
Syslog เพื่อหาสาเหตุ **ไม่เจอ Message HSRP State Change เลยแม้แต่บรรทัดเดียว** ทั้งที่ Failover
เกิดขึ้นจริงและยืนยันได้จาก `show standby brief`

**ไล่ตาม Checklist**: ตรวจ Local Buffer ก่อน (ชั้น 1) — พบว่า Local Buffer **มี** Message
HSRP อยู่จริง (เพราะ `logging buffered` เป็นคนละ Level กับ `logging trap`):

```
DIST-SW4# show logging | include HSRP
000512: Sep 26 2026 09:14:02.201 ICT: %HSRP-5-STATECHANGE: Vlan40 Grp 40 state Active -> Speak
000513: Sep 26 2026 09:14:12.664 ICT: %HSRP-5-STATECHANGE: Vlan40 Grp 40 state Speak -> Standby
```

Local Buffer เห็น Message ปกติ — แปลว่าไม่ใช่ IOS ไม่ Generate Message แต่เป็นเพราะ **ไม่ถูก
ส่งไป Remote Syslog** เพราะ Severity Filter (`logging trap`) กรองออกไปก่อนถึงขั้นตอน Forward:

```
DIST-SW4# show logging | include Trap logging
    Trap logging: level error, 1204 message lines logged
```

**`level error`** (ไม่ใช่ `informational` ตาม Standard ของ Lab นี้จาก
[Part 17 Step 165](part-017-ntp-syslog-snmp.md)) คือ Root Cause ที่ชัดเจน

**Root Cause & Fix**:

```
DIST-SW4(config)# logging trap informational
```

```
DIST-SW4# show logging | include Trap logging
    Trap logging: level informational, 1204 message lines logged
```

ทดสอบ Failover อีกครั้ง (Manual `standby 40 priority` เปลี่ยนชั่วคราว) และยืนยันว่า NOC-SRV
เห็น `%HSRP-5-STATECHANGE` เข้ามาใหม่ตรงเวลาทันที

> **บทเรียน**: ถ้าต้องการ "ลด Noise" ของ Log ให้ทำที่ **ฝั่ง Collector/SIEM** (Filter/Alert Rule)
> แทนการลด `logging trap` บนอุปกรณ์ต้นทาง — Log ที่ไม่ถูกส่งไปเลยจะไม่มีทางย้อนดูจาก Remote
> Collector ได้อีก ต่างจาก Local Buffer ที่ยังเก็บไว้ชั่วคราว

---

## Step 617 — Scenario 6: SNMP Polling ล้มเหลว — SNMPv3 User/Group ไม่ตรงกันหลัง Re-config

### 617.1 Background

ทีม Security สั่งให้ Rotate SNMPv3 Credential บน CORE-SW2 และเปลี่ยนชื่อ Group ให้สื่อความหมาย
ชัดเจนขึ้นตามมาตรฐานใหม่ (จาก `NOC-RO-GROUP` เดิมตาม
[Part 17 Step 168](part-017-ntp-syslog-snmp.md) เป็น `NOC-MONITOR-GROUP`) วิศวกรที่ทำ Re-config
สร้าง Group ใหม่และ User ใหม่สำเร็จ แต่ **ลืมกำหนด `read`-View ให้ Group ใหม่**

```
CORE-SW2(config)# snmp-server group NOC-MONITOR-GROUP v3 priv
! <-- ลืมใส่ "read FULL-VIEW" ต่อท้าย
CORE-SW2(config)# snmp-server user noc-svc-v2 NOC-MONITOR-GROUP v3 auth sha AuthPass2027! priv aes 128 PrivPass2027!
```

NOC-SRV (NMS) ก็ถูก Update ให้ใช้ Username `noc-svc-v2` ตัวใหม่แล้วเช่นกัน

### 617.2 อาการ (Symptom)

NOC-SRV Dashboard แสดง **CORE-SW2 เป็นสถานะ "Down"** ทั้งที่:

```
NOC-SRV:~$ ping 1.1.1.2
PING 1.1.1.2 (1.1.1.2): 56 data bytes
64 bytes from 1.1.1.2: icmp_seq=0 ttl=254 time=1.2 ms
```

ICMP ผ่านปกติ และ SSH เข้าไป CLI ได้ตามปกติทุกอย่าง — **สัญญาณคลาสสิกของปัญหา SNMP ที่ไม่ใช่
ปัญหา Connectivity จริง** ตามที่ Step 611.1 อธิบายไว้ (NMS ใช้ SNMP Poll เป็นเกณฑ์ตัดสิน "Up/
Down" คู่กับ ICMP — ถ้า SNMP Poll ล้มเหลว NMS บาง Platform จะขึ้น Alert "Down" ทันทีแม้ ICMP
จะผ่าน)

### 617.3 การไล่ตาม Checklist

**ชั้น 4 (Server-side Config)** คือจุดที่ต้องเช็คตรงๆ สำหรับ SNMP เพราะ Layer 1-3 (Physical/
Reachability/ACL) ผ่านหมดแล้ว (Ping ผ่าน, ไม่มี ACL SNMP เพิ่มใหม่) — เริ่มจาก User:

```
CORE-SW2# show snmp user

User name: noc-svc-v2
Engine ID: 800000090300ABCD87654321
storage-type: nonvolatile        active
Authentication Protocol: SHA
Privacy Protocol: AES128
Group-name: NOC-MONITOR-GROUP
```

User มีอยู่จริง ผูกกับ Group `NOC-MONITOR-GROUP` ถูกต้อง Auth/Priv Protocol ก็ตรงกับที่ NOC-SRV
คาดหวัง — ดูเหมือนไม่มีปัญหา ต้องขยับไปดู **Group** ต่อ:

```
CORE-SW2# show snmp group

groupname: NOC-MONITOR-GROUP       security model:v3 priv
readview : <no readview specified>  writeview: <no writeview specified>
notifyview:  <no notifyview specified>
row status: active
```

**`readview: <no readview specified>`** — พบสาเหตุแล้ว: Group `NOC-MONITOR-GROUP` **ไม่มี
Read View ผูกอยู่เลย** แปลว่า User `noc-svc-v2` แม้ Authenticate ผ่านทุกอย่าง แต่เมื่อ NOC-SRV
ส่ง `GET` Request ถาม OID ใดๆ CORE-SW2 จะตอบกลับด้วย **Authorization Error** เพราะไม่มี View
ให้ "มองเห็น" MIB Tree เลย — NMS ตีความ Error แบบนี้เหมือนกับ Device Down

**ยืนยันด้วย Debug (ทำใน Maintenance Window เท่านั้น เพราะ Debug SNMP ส่ง Log ปริมาณมาก)**:

```
CORE-SW2# debug snmp packet
CORE-SW2#
*Sep 26 15:44:02.112: SNMP: Packet received via UDP from 10.10.99.50 on Vlan99
*Sep 26 15:44:02.113: SNMP: Get request, reqid 84621, errstat 0, erridx 0
*Sep 26 15:44:02.114: SNMP: Response, reqid 84621, errstat 13, erridx 1
CORE-SW2# undebug all
```

`errstat 13` คือ **`noAccess`** ใน SNMP Error Status — ยืนยัน 100% ว่าเป็นปัญหา Authorization
(View ไม่ครอบคลุม) ไม่ใช่ปัญหา Authentication (ถ้า Auth ผิดจะไม่เห็น Packet ตอบกลับเลยด้วยซ้ำ
เพราะ Engine จะ Drop ตั้งแต่ก่อนถึงขั้น Process Request)

### 617.4 Root Cause และการแก้ไข

**Root Cause**: Group `NOC-MONITOR-GROUP` ถูกสร้างขึ้นโดยไม่มีการผูก `read`-View — เป็นการตกหล่น
ระหว่าง Partial Re-configuration (สร้าง Group/User ใหม่ตามมาตรฐานชื่อใหม่ แต่ Copy-Paste
Command ไม่ครบ)

```
CORE-SW2(config)# snmp-server view FULL-VIEW iso included
CORE-SW2(config)# snmp-server group NOC-MONITOR-GROUP v3 priv read FULL-VIEW
```

### 617.5 ยืนยันผลลัพธ์

```
CORE-SW2# show snmp group

groupname: NOC-MONITOR-GROUP       security model:v3 priv
readview : FULL-VIEW               writeview: <no writeview specified>
row status: active
```

```
NOC-SRV:~$ snmpget -v3 -u noc-svc-v2 -l authPriv -a SHA -A AuthPass2027! -x AES -X PrivPass2027! \
    1.1.1.2 1.3.6.1.2.1.1.3.0
SNMPv2-MIB::sysUpTime.0 = Timeticks: (89234500) 10:19:05.00
```

Poll สำเร็จ — NOC-SRV Dashboard เปลี่ยนสถานะ CORE-SW2 กลับเป็น "Up" ภายใน 1 รอบ Poll ถัดไป

> **หลักในการวินิจฉัย SNMP**: ไล่เทียบ **User → Group → View** เป็นลำดับเสมอ (`show snmp user`
> → `show snmp group` → `show snmp view <name>`) — ปัญหาส่วนใหญ่ของ SNMPv3 อยู่ที่ "ห่วงโซ่" นี้
> ขาดตอนจุดใดจุดหนึ่ง ไม่ใช่ที่ Network Path เหมือน NTP ใน Step 615

---

## Step 618 — Scenario 7: Router เอง Resolve DNS ไม่ได้

### 618.1 Background

วิศวกรต้องการให้ WAN-EDGE-1 สำรอง Configuration ไปยัง Backup Server ผ่านชื่อ Hostname
(สะดวกกว่าจำ IP) และทดสอบ Connectivity ไปยัง Internet ด้วยชื่อโดเมนแทน IP:

```
WAN-EDGE-1# copy running-config tftp://backup-nms.lab.local/wan-edge-1-confg
```

### 618.2 อาการ — รอบที่ 1: `ip domain-lookup` ถูกปิดจาก Part 2

```
WAN-EDGE-1# copy running-config tftp://backup-nms.lab.local/wan-edge-1-confg
Address or name of remote host [backup-nms.lab.local]?
Translating "backup-nms.lab.local"
% Unrecognized host or address, or protocol not running.
```

`% Unrecognized host or address, or protocol not running.` เป็น Error ที่เจาะจงเวลาที่ IOS
พยายาม Resolve Hostname แต่ **DNS Resolution Function ทั้งระบบไม่ได้เปิดทำงานเลย**

**ไล่ตาม Checklist ชั้น 1**:

```
WAN-EDGE-1# show running-config | include ip domain
! (ไม่มี Output เลย)
```

ไม่พบ `ip domain-lookup` หรือ `ip name-server` ใดๆ — ยืนยันว่า WAN-EDGE-1 ยังอยู่ใน State
เดิมตามที่ [Part 2](part-002-ios-cli-basics.md) Harden ไว้ตอนแรก (`no ip domain-lookup` เพื่อ
ป้องกัน CLI ค้างเวลาพิมพ์คำสั่งผิด ตามที่อธิบายไว้ใน
[Part 16 Step 156.4](part-016-dhcp-dns.md)) — **การ Harden นั้นถูกต้องตอนที่ทำ แต่ตอนนี้มี
Feature ใหม่ (TFTP Backup ด้วย Hostname) ที่ต้องพึ่งพา DNS จริง** ทำให้ต้องเปิดกลับมาเฉพาะจุด

**Fix รอบที่ 1**:

```
WAN-EDGE-1(config)# ip domain-lookup
WAN-EDGE-1(config)# ip domain-name lab.local
WAN-EDGE-1(config)# ip name-server 10.10.30.20
```

### 618.3 อาการ — รอบที่ 2: `ip name-server` ชี้ไปยัง Server ที่ไปไม่ถึง

ทดสอบใหม่หลัง Fix รอบแรก:

```
WAN-EDGE-1# copy running-config tftp://backup-nms.lab.local/wan-edge-1-confg
Address or name of remote host [backup-nms.lab.local]?
Translating "backup-nms.lab.local"...domain server (10.10.30.20)
Translating "backup-nms.lab.local"...domain server (10.10.30.20) [timed out]
% Unrecognized host or address, or protocol not running.
```

รอบนี้ Error เปลี่ยนไปเล็กน้อย — เห็น `Translating ... domain server (10.10.30.20)` แสดงว่า
IOS **พยายาม** Query ไปที่ `10.10.30.20` แล้ว แต่ **`[timed out]`** คือไม่มีการตอบกลับ

**ไล่ตาม Checklist ชั้น 2**:

```
WAN-EDGE-1# ping 10.10.30.20
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/5/8 ms
```

Ping ผ่านปกติ — เครือข่ายไม่มีปัญหา แต่ DHCP-DNS-SRV เองไม่ได้ตอบ Query สำหรับ Record
`backup-nms.lab.local` เพราะ **Record นี้ยังไม่ได้ถูกสร้างไว้ในโซนของ DNS Server เลย**
(Backup Server เพิ่งถูกเพิ่มเข้ามาใหม่ ทีมงานลืมเพิ่ม A Record ให้)

```
root@dhcp-dns-srv:~# host backup-nms.lab.local
Host backup-nms.lab.local not found: 3(NXDOMAIN)
```

`NXDOMAIN` ยืนยันว่า Record ไม่มีอยู่จริงในฝั่ง Server — เป็นสาเหตุที่ต่างจาก 2 คำอธิบายที่ให้มา
เดิม (`ip domain-lookup` ปิด หรือ `ip name-server` ชี้ผิด IP) แต่จัดอยู่ในกลุ่มเดียวกันคือ
"DNS Resolution ล้มเหลว" — Checklist แบบเป็นระบบช่วยให้ไล่เจอสาเหตุที่แท้จริงได้แม้จะไม่ตรงกับ
สมมติฐานแรกเป๊ะ

**Fix รอบที่ 2 (ที่ DNS Server)**:

```
root@dhcp-dns-srv:~# echo "backup-nms  IN  A  10.10.30.25" >> /etc/bind/zones/lab.local.zone
root@dhcp-dns-srv:~# rndc reload
```

### 618.4 ยืนยันผลลัพธ์

```
WAN-EDGE-1# copy running-config tftp://backup-nms.lab.local/wan-edge-1-confg
Address or name of remote host [backup-nms.lab.local]?
Translating "backup-nms.lab.local"...domain server (10.10.30.20) [OK]
Destination filename [wan-edge-1-confg]?
!!
1847 bytes copied in 0.891 secs
```

> **บทเรียน**: ปัญหา "Resolve DNS ไม่ได้" มีได้ 3 ชั้นเสมอ — **(1) Function ปิดอยู่
> (`ip domain-lookup`), (2) ไปหา Server ผิดตัวหรือไปไม่ถึง (`ip name-server`/Routing/ACL),
> (3) Server ไปถึงแล้วแต่ไม่มี Record จริง (`NXDOMAIN`)** — Error Message ของ IOS ต่างกัน
> ชัดเจนในแต่ละชั้น (`Unrecognized host` = ชั้น 1, `[timed out]` = ชั้น 2)

---

## Step 619 — Scenario 8: Clock ผิดเพราะ NTP ลาม (Cascade) ไปทำให้ EAP-TLS Certificate ล้มเหลว

### 619.1 Background — ทำไม Step นี้สำคัญที่สุดของ Part

ทุก Scenario ก่อนหน้านี้ ปัญหาอยู่ "ในตัว Service เดียวกัน" (DHCP พังเพราะ DHCP Config, NTP พัง
เพราะ NTP Config) — แต่ Scenario นี้แสดงให้เห็นว่า **Infra Service ตัวหนึ่งที่พัง (NTP) สามารถ
ทำให้ Feature ที่ดูไม่เกี่ยวข้องกันเลย (Wireless 802.1X Authentication) พังตามไปด้วย** — เป็น
รูปแบบปัญหาที่ยากที่สุดในการ Troubleshoot เพราะทีมที่รับผิดชอบ Wireless มักไม่คิดว่าต้องไปตรวจ
NTP ก่อนเลย

### 619.2 อาการ (Symptom)

ทีม Wireless (ตามความรู้จาก [Part 37](part-037-wireless-security-advanced.md) และ
[Part 44](part-044-dot1x-trustsec.md)) ได้รับแจ้งจากผู้ใช้ทั่วองค์กรพร้อมกันว่า **"ต่อ Wi-Fi
Corporate-SSID ไม่ได้เลย ทุกเครื่อง"** — WLC-1 (Catalyst 9800-CL) ใช้ 802.1X + EAP-TLS
(Client Certificate) ผ่าน RADIUS ไปยัง ISE

```
WLC-1# show wireless client summary
Number of Local Clients: 0

WLC-1# show ap dot1x statistics
AP Name              : AP-1
EAP-TLS Failures      : 214 (last 5 minutes)
```

**EAP-TLS Failures 214 ครั้งในเวลาสั้นๆ** — ผิดปกติมาก เพราะ Certificate ที่ Client ใช้อยู่ยัง
ไม่ Expire ตามกำหนดจริง (ยืนยันจาก CA ว่า Certificate อายุยังเหลืออีก 8 เดือน)

### 619.3 การไล่ตาม Checklist

**ตรวจ Log ที่ WLC-1 โดยตรง**:

```
WLC-1# show logging | include EAP-TLS
%DOT1X-3-CERT_VALIDATION_FAIL: Certificate validation failed for client 00:1A:2B:3C:4D:5E,
  reason: certificate is not yet valid (NotBefore in the future)
```

**`certificate is not yet valid (NotBefore in the future)`** — Error นี้บอกว่า Certificate
"ยังไม่ถึงวันเริ่มใช้งาน" ทั้งที่ Certificate ถูก Issue มาหลายเดือนแล้ว — สัญญาณนี้ชี้ไปที่
**Clock ของ WLC-1 เอง** มากกว่า Certificate จริงๆ (เพราะ Cert Validation เทียบ NotBefore/
NotAfter กับ**เวลาปัจจุบันของอุปกรณ์ที่ตรวจสอบ** ไม่ใช่เวลาจริง)

```
WLC-1# show clock detail
08:14:22.104 UTC Tue Jan 1 2019
Time source is user configuration
```

พบ Root Cause ทันที: **`Time source is user configuration`** (ไม่ใช่ NTP) และวันที่แสดง
**`Jan 1 2019`** — Clock ของ WLC-1 กลับไปเป็นค่า Default หลัง Firmware Upgrade ล่าสุด (Config
ไม่ได้ Save ก่อน Reload) เพราะ Clock ปี 2019 อยู่ **ก่อนหน้า NotBefore ของ Certificate จริง**
(Issue ปี 2026) ทำให้ WLC-1 มองว่า Certificate "ยังไม่ถึงเวลาใช้งาน" ทั้งที่ใช้งานได้สมบูรณ์

```
WLC-1# show ntp status
Clock is unsynchronized, stratum 16, no reference clock

WLC-1# show running-config | include ntp server
! (ไม่มี Output เลย — Config หายไปจริง)
```

### 619.4 Root Cause และการแก้ไข

**Root Cause**: WLC-1 สูญเสีย NTP Client Configuration ไปหลัง Firmware Upgrade/Reload ทำให้
Clock กลับไปเป็นค่า Default ของโรงงาน (ปี 2019) ซึ่งอยู่ **ก่อน** วันที่ Certificate ของ Client
ทุกใบเริ่มมีผล (NotBefore) — EAP-TLS Validation จึง Fail ทุกครั้งเพราะมองว่า Certificate
"มาจากอนาคต"

```
WLC-1(config)# ntp server 10.10.99.50
WLC-1(config)# clock timezone ICT 7 0
WLC-1(config)# end

WLC-1# clock update-calendar
WLC-1# show ntp status
Clock is synchronized, stratum 3, reference is 10.10.99.50

WLC-1# show clock detail
15:52:10.881 ICT Fri Sep 26 2026
Time source is NTP
```

### 619.5 ยืนยันผลลัพธ์

```
WLC-1# show ap dot1x statistics
AP Name              : AP-1
EAP-TLS Failures      : 0 (last 5 minutes)

WLC-1# show wireless client summary
Number of Local Clients: 214
```

Client กลับมา Authenticate สำเร็จทั้งหมดทันที **โดยไม่ต้องแก้ Certificate หรือ RADIUS/ISE
Config ใดๆ เลย** — พิสูจน์ชัดเจนว่า Root Cause คือ Clock เพียงอย่างเดียว

> **บทเรียนสำคัญที่สุดของ Part นี้**: เมื่อเจอปัญหาที่ **"ดูไม่เกี่ยวกับ Infra Service เลย"**
> (Certificate/TLS/Authentication Failure ที่กระทบทุกคนพร้อมกัน) **ให้ตรวจ `show clock detail`
> และ `show ntp status` เป็นหนึ่งในขั้นตอนแรกๆ เสมอ** เพราะ Time-related Failure มักมี
> "ลักษณะพร้อมกันทั้งระบบ" ต่างจากปัญหา Hardware/Config เจาะจงจุดเดียว

---

## Step 620 — Lab เต็มรูปแบบ: 3 ปัญหาซ้อนกัน (Helper-address, NTP Key Mismatch, Time-based ACL)

### 620.1 สถานการณ์เริ่มต้น (Trouble Ticket)

เช้าวันจันทร์ ทีม NOC ได้รับ 2 Ticket พร้อมกันจากคนละแผนก:

**Ticket #1** (จากทีม IoT): "Sensor IoT ที่ติดตั้งใน VLAN ใหม่ (IOT-SENSORS) เมื่อสัปดาห์ก่อน
ไม่มีตัวไหนออนไลน์เลย ทั้งที่ต่อสายและไฟ LED ติดปกติทุกตัว"

**Ticket #2** (จากทีม Contractor): "Contractor ที่เข้าถึง Server1 ผ่าน HTTPS ได้ปกติทุกวันจันทร์-
ศุกร์ วันนี้จันทร์เที่ยงตรง (Business Hours ชัดๆ) แต่กลับเข้าไม่ได้เลย ระบบขึ้น Connection
Refused/Timeout"

สองปัญหาดูไม่เกี่ยวข้องกันเลยในตอนแรก แต่วิศวกรที่รับ Ticket ใช้ Checklist จาก Step 611
ไล่ทั้งสองปัญหาแบบเป็นระบบไปพร้อมกัน — **DIST-SW4** คือ Gateway ของ VLAN 65 (IOT-SENSORS,
Ticket #1) ส่วน **DIST-SW3** คือจุดที่มี `time-range BUSINESS-HOURS` จาก Part 45 อยู่
(Ticket #2) ทั้งสองตัวชี้ NTP ไปที่ NOC-SRV (`10.10.99.50`) และใช้ DHCP-DNS-SRV
(`10.10.30.20`) เป็น Central Server ร่วมกัน

### 620.2 ไล่ Ticket #1 — IoT Sensor ไม่ Online (Independent Issue)

**ชั้น 1 (Local Service Config) บน DIST-SW4**:

```
DIST-SW4# show running-config interface vlan 65
interface Vlan65
 description ** Gateway VLAN65-IOT-SENSORS (New - added last week) **
 ip address 10.10.65.2 255.255.255.0
end
```

ไม่มี `ip helper-address` เหมือนกับ Pattern ที่เจอใน Step 612 เป๊ะ — เป็นปัญหาเดิมที่เกิดขึ้นซ้ำ
เพราะ VLAN 65 ก็เป็น VLAN ที่สร้างขึ้นใหม่เมื่อสัปดาห์ก่อนเช่นกัน และทีมงานที่สร้างก็ทำตาม
Checklist เก่าที่ไม่มี Step DHCP Relay เหมือนเดิม

```
DIST-SW4(config)# interface vlan 65
DIST-SW4(config-if)# ip helper-address 10.10.30.20
DIST-SW4(config-if)# exit
```

```
DIST-SW4# show ip interface vlan 65 | include Helper
  Helper address is 10.10.30.20
```

รอ 2-3 นาทีให้ Sensor Retry DHCP ตามรอบของตัวเอง — Sensor เริ่ม Online ทีละตัว **Ticket #1
ปิดสำเร็จ** — เป็น Root Cause ที่ **ไม่เกี่ยวข้อง**กับ Ticket #2 เลย (Independent Issue ตัวแรก)

### 620.3 ไล่ Ticket #2 — Contractor เข้า Server1 ไม่ได้ในเวลาทำการ

**ทวนความจำ Config เดิมจาก [Part 45 Step 442](part-045-advanced-acl-zbfw.md)**:

```
DIST-SW3# show ip access-lists CONTRACTOR-TO-SERVER1
Extended IP access list CONTRACTOR-TO-SERVER1
    20 permit tcp 10.10.50.0 0.0.0.255 host 10.10.30.10 eq 443 time-range BUSINESS-HOURS (0 matches)
    40 deny ip 10.10.50.0 0.0.0.255 any log (47 matches)
    50 permit ip any any (890 matches)
```

**Line 20 (Permit ในเวลาทำการ) มี 0 matches** ในขณะที่ **Line 40 (Deny) มี 47 matches
เพิ่มขึ้นเรื่อยๆ** — แปลว่า Time-range `BUSINESS-HOURS` **ไม่ Active** ทั้งที่ตอนนี้เป็นวันจันทร์
เที่ยงตรง (อยู่ในช่วง Weekdays 08:00-18:00 แน่นอน)

```
DIST-SW3# show time-range BUSINESS-HOURS
time-range entry: BUSINESS-HOURS (inactive)
  periodic weekdays 8:00 to 18:00
```

**`(inactive)`** ทั้งที่ควรจะ Active — ตาม [Part 45 Step 442](part-045-advanced-acl-zbfw.md)
เตือนไว้ล่วงหน้าแล้วว่า `time-range` อ้างอิงตาม **Clock ของอุปกรณ์เอง** — ต้องตรวจ Clock ของ
DIST-SW3 ทันที (ชั้น 5 ของ Checklist — Time/Clock Consistency):

```
DIST-SW3# show clock detail
21:47:03.552 ICT Thu Sep 24 2026
Time source is user configuration
```

**Clock ของ DIST-SW3 อ่านว่า "21:47 วันพฤหัสฯ ที่ 24 กันยายน"** ทั้งที่เวลาจริงคือ "เที่ยงวันจันทร์
ที่ 28 กันยายน" — คลาดเคลื่อนไปเกือบ 4 วัน และ **`Time source is user configuration`** (ไม่ใช่
NTP) ยืนยันว่า DIST-SW3 ไม่ได้ Sync มานานแล้ว จึง Free-run เพี้ยนไปตาม Internal Clock Drift
ของ Hardware เอง (Drift สะสมแค่ไม่กี่วัน ไม่ใช่ Reset เต็มรูปแบบแบบ Scenario 619)

**ตรวจสาเหตุที่ NTP ไม่ Sync**:

```
DIST-SW3# show ntp status
Clock is unsynchronized, stratum 16, no reference clock

DIST-SW3# show ntp associations detail
10.10.99.50 configured, insane, invalid, unauthenticated, stratum 16
ref ID .INIT., time 00000000.00000000 (never)
```

**`unauthenticated`** — เหมือนกับ Sub-scenario B ของ Step 615 — ตรวจ Key ที่ DIST-SW3:

```
DIST-SW3# show running-config | section ntp
ntp authentication-key 1 md5 0958524B0A18 ,17 scrambled
ntp trusted-key 1
ntp authenticate
ntp server 10.10.99.50 key 1
```

Config ดู "ครบ" ทุกบรรทัดตาม [Part 17 Step 163](part-017-ntp-syslog-snmp.md) — แต่ Key String
ถูก Encrypt เป็น Type 7 เสมอ เทียบด้วยตาไม่ได้ ต้องสอบถามทีม Security ที่ Rotate Key ล่าสุด —
ได้คำตอบว่า **DIST-SW3 ถูกตกหล่นจาก Batch Script Rotate Key ทั่ว Lab เมื่อสัปดาห์ก่อน** (SSH
Session หลุดกลางทางตอนรัน แต่ไม่มีใครสังเกต Error) DIST-SW3 จึงยังใช้ Key เก่า
(`NtpAuthKey2026!`) ในขณะที่ NOC-SRV เปลี่ยนเป็น Key ใหม่ไปแล้วทั้งระบบ

### 620.4 Root Cause Chain แบบสมบูรณ์

```
Root Cause: Batch Script Rotate NTP Key ตกหล่น DIST-SW3
                          │
                          ▼
        DIST-SW3 Authentication Key ไม่ตรงกับ NOC-SRV
                          │
                          ▼
             NTP ไม่ Sync ("unauthenticated")
                          │
                          ▼
        Clock ของ DIST-SW3 Drift ออกไปเรื่อยๆ (Free-run)
                          │
                          ▼
   time-range BUSINESS-HOURS ประเมินผิดว่า "(inactive)"
   ทั้งที่เป็นเวลาทำการจริง (เพราะ Clock ของ DIST-SW3 เอง
   คลาดเคลื่อนไปพ้นช่วง 08:00-18:00 ตามที่มันเข้าใจ)
                          │
                          ▼
   ACE บรรทัด Permit HTTPS ไม่ Active -> ตกไปที่ Deny
                          │
                          ▼
        Contractor เข้าถึง Server1 ไม่ได้ (Ticket #2)
```

นี่คือตัวอย่างสมบูรณ์ของสิ่งที่ Step 619 อธิบายไว้ — **ปัญหา NTP (Infra Service) ที่ "เงียบ"
ลามไปทำให้ Security Policy (Time-based ACL จาก Part 45) ทำงานผิดพลาดโดยที่ไม่มีใครแก้ ACL
เลยแม้แต่บรรทัดเดียว**

### 620.5 การแก้ไขแบบสมบูรณ์

```
DIST-SW3(config)# no ntp authentication-key 1 md5 NtpAuthKey2026!
DIST-SW3(config)# ntp authentication-key 1 md5 NtpAuthKey2027Secure!
DIST-SW3(config)# ntp trusted-key 1
DIST-SW3(config)# end
```

```
DIST-SW3# show ntp associations detail
10.10.99.50 configured, authenticated, our_master, sane, valid, stratum 2

DIST-SW3# show ntp status
Clock is synchronized, stratum 3, reference is 10.10.99.50
```

รอให้ NTP Algorithm ปรับ Clock แบบนุ่มนวลจนเสร็จ (ไม่กระโดดเวลาทันทีตาม
[Part 17 Step 161](part-017-ntp-syslog-snmp.md)) — ใช้เวลาประมาณ 10-15 นาทีสำหรับ Clock ที่
คลาดเคลื่อนหลายวันแบบนี้:

```
DIST-SW3# show clock detail
12:03:47.220 ICT Mon Sep 28 2026
Time source is NTP
```

Clock ถูกต้องแล้ว — ตรวจ Time-range อีกครั้ง:

```
DIST-SW3# show time-range BUSINESS-HOURS
time-range entry: BUSINESS-HOURS (active)
  periodic weekdays 8:00 to 18:00
```

**`(active)`** ตามที่ควรจะเป็น — ตรวจ ACL:

```
DIST-SW3# show ip access-lists CONTRACTOR-TO-SERVER1
Extended IP access list CONTRACTOR-TO-SERVER1
    20 permit tcp 10.10.50.0 0.0.0.255 host 10.10.30.10 eq 443 time-range BUSINESS-HOURS (active) (3 matches)
    40 deny ip 10.10.50.0 0.0.0.255 any log (47 matches)
```

Line 20 เริ่มมี Matches แล้ว — **Ticket #2 ปิดสำเร็จ** โดยที่ **ไม่ได้แก้ ACL แม้แต่บรรทัดเดียว**
พิสูจน์ชัดเจนว่า Root Cause อยู่ที่ NTP Authentication เพียงจุดเดียวเท่านั้น

### 620.6 ตารางสรุป Lab (Full RCA Report ส่งให้ผู้บริหาร)

| # | Ticket | Root Cause | จุดที่แก้ | อิสระจากปัญหาอื่นหรือ Cascade |
|---|---|---|---|---|
| 1 | IoT Sensor ไม่ Online | ไม่มี `ip helper-address` บน SVI Vlan65 (VLAN ใหม่) | DIST-SW4 | Independent — ไม่เกี่ยวกับปัญหาอื่น |
| 2 | Contractor เข้า Server1 ไม่ได้ | NTP Key Mismatch (ตกหล่นจาก Batch Rotate Script) → Clock Drift → Time-range ผิด | DIST-SW3 | Cascade 3 ชั้น (NTP → Clock → ACL) |

### 620.7 Action Item ป้องกันไม่ให้เกิดซ้ำ

| Action | เหตุผล |
|---|---|
| เพิ่ม "ตรวจ `ip helper-address`" เข้า Checklist มาตรฐานการสร้าง VLAN ใหม่ | ป้องกัน Scenario แบบ 620.2 เกิดซ้ำ |
| ให้ Batch Script Rotate Key มี Exit Code Check + Alert ถ้า SSH หลุดกลางทาง | ป้องกันการตกหล่นแบบเงียบๆ อีก |
| ตั้ง SNMP Trap แจ้งเตือนทันทีที่ `show ntp status` เป็น `unsynchronized` เกิน 30 นาที | จับปัญหาก่อนที่ Clock จะ Drift มากจนกระทบ Feature อื่น |

---

## แบบฝึกหัดทวนความเข้าใจ Part 62

1. เพราะเหตุใดปัญหาของ Infra Services (DHCP/NTP/Syslog/SNMP) จึงต้องใช้แนวทาง "Checklist เชิง
   ระบบ" มากกว่าการไล่ตาม Error Message เหมือนที่ใช้กับ Routing Protocol?
2. ในกรณี DHCP Relay ทำงานปกติแต่ Client ได้ Default Gateway ผิด (เป็น Real IP เดิมไม่ใช่ HSRP
   VIP) ต้องไปตรวจสอบและแก้ไขที่จุดใด และทำไมปัญหานี้ถึง "ซ่อน" อยู่ได้นานโดยไม่มีใครรู้ตัว?
3. `show ntp associations detail` ที่แสดงคำว่า `unauthenticated` ต่างจากที่แสดง `insane,
   invalid` แบบไม่มี Response เลยอย่างไร และแต่ละแบบบอกกลุ่มสาเหตุอะไร?
4. อธิบายลำดับการวินิจฉัยปัญหา SNMPv3 Polling ล้มเหลวด้วยแนวคิด "User → Group → View" และ
   `errstat 13` (`noAccess`) จาก `debug snmp packet` บอกอะไรเกี่ยวกับตำแหน่งของปัญหา?
5. จาก Lab ใน Step 620 เพราะเหตุใด Time-based ACL ที่ไม่มีการแก้ไข Config เลยแม้แต่บรรทัดเดียว
   ถึงกลับมาทำงานถูกต้องได้ทันทีหลังแก้ปัญหา NTP Authentication — อธิบาย Chain of Causation
   ทั้งหมด?

**เฉลย:**

1. เพราะ Infra Services ส่วนใหญ่ **ไม่มี Error Message หรือ Neighbor State ที่ Flap ให้เห็น**
   เมื่อพัง (เช่น DHCP Client แค่ Fallback เป็น APIPA เงียบๆ, NTP แค่ค้างที่ Stratum 16 แบบ
   Passive, Syslog หายไปโดยที่ฝั่งส่งไม่รู้ตัวเพราะเป็น UDP) การรอดู Log แล้วไล่ตาม Error
   แบบที่ใช้กับ OSPF/BGP จะไม่พบอะไรเลย ต้องใช้ Checklist 6 ชั้น (Local Config → Reachability
   → Path ACL → Server-side Config → Clock → Cross-verification) ไล่ตรวจทุกจุดที่เป็นไปได้
   อย่างเป็นระบบแทน
2. ต้องไปตรวจและแก้ที่ **Scope Option ของ DHCP Server เอง** (เช่น `option routers` บน
   ISC-DHCP หรือ `default-router` บน IOS Local Pool) ให้ชี้ไปที่ HSRP Virtual IP แทน Real IP
   เดิม — ปัญหานี้ซ่อนอยู่ได้นานเพราะ **Client ยังใช้งานได้ปกติทุกอย่างตราบใดที่ Router ตัวที่
   เป็น Real IP เดิมยังทำงานอยู่** จะรู้ตัวก็ต่อเมื่อ Router ตัวนั้น Down จริงแล้ว Redundancy
   ที่ควรมีจาก HSRP ไม่ทำงานสำหรับ Client กลุ่มนี้เลย
3. `unauthenticated` หมายถึง **Packet ไปถึงปลายทางแล้วจริง** (มี Response กลับมา) แต่
   Authentication Key ไม่ตรงกัน — จัดอยู่ในกลุ่มสาเหตุ "ไปถึงแต่ไม่ได้รับความเชื่อถือ" ส่วน
   `insane, invalid` พร้อม `ref ID .INIT., time 00000000.00000000 (never)` หมายถึง **ไม่มี
   Response กลับมาเลยตั้งแต่ต้น** ซึ่งจัดอยู่ในกลุ่มสาเหตุ "ไปไม่ถึง" (Routing/ACL/Firewall
   บล็อกอยู่ระหว่างทาง)
4. ไล่ตรวจ `show snmp user` ก่อน (User มีอยู่จริงและผูกกับ Group ไหน) → `show snmp group`
   (Group นั้นมี Security Level และ View ที่ถูกต้องหรือไม่) → `show snmp view <name>` (View
   ครอบคลุม OID ที่ต้องการ Poll หรือไม่) — `errstat 13 (noAccess)` จาก `debug snmp packet`
   บอกว่า **Authentication ผ่านสำเร็จแล้ว** (ถ้า Auth ผิดจะไม่มี Response เลย) แต่
   **Authorization ล้มเหลว** เพราะ View ไม่ครอบคลุม ชี้ตรงไปที่ปัญหาอยู่ในขั้นตอน Group/View
   ไม่ใช่ User/Password
5. เพราะ Root Cause ที่แท้จริงไม่ได้อยู่ที่ ACL เลยตั้งแต่ต้น — Chain คือ: NTP Authentication
   Key ไม่ตรงกัน (ตกหล่นจาก Batch Rotate Script) → DIST-SW3 ไม่สามารถ Sync เวลาได้
   (`unauthenticated`) → Clock ของ DIST-SW3 Free-run และ Drift ออกไปจากเวลาจริง (Time source
   เปลี่ยนเป็น user configuration) → `time-range BUSINESS-HOURS` ที่ประเมินผลตาม **Clock ของ
   DIST-SW3 เอง** จึงคำนวณผิดว่าอยู่นอกช่วง 08:00-18:00 ทั้งที่เป็นเวลาทำการจริง → ACE ที่ผูก
   `time-range` นี้ไม่ Active → Traffic ตกไปที่บรรทัด Deny — เมื่อแก้ NTP Authentication ให้
   Sync สำเร็จ Clock กลับมาถูกต้องอัตโนมัติ ทำให้ Time-range กลับมา Active เองโดยไม่ต้องแตะ
   ACL แม้แต่บรรทัดเดียว

---

## สรุป Part 62

Part นี้ครอบคลุมการ Troubleshoot **Infrastructure Services** ครบทั้ง DHCP, NTP, Syslog, SNMP,
และ DNS บนอุปกรณ์เอง — จุดร่วมที่สำคัญที่สุดของทุก Scenario คือ **ปัญหากลุ่มนี้ "เงียบ" โดย
ธรรมชาติ** ไม่มี Neighbor State หรือ Error Message ที่ฟ้องตรงๆ เหมือน Routing Protocol
จึงต้องใช้ **Checklist 6 ชั้น** (Local Config → Reachability → Path ACL → Server-side Config
→ Clock Consistency → Cross-verification) ไล่ตรวจอย่างเป็นระบบแทนการรอดู Error — เราไล่ผ่าน
Scenario จริง 8 แบบ: VLAN ใหม่ที่ลืม `ip helper-address`, Scope DHCP ที่ไม่ตาม HSRP VIP,
Pool Exhaustion จาก Exclude กว้างเกินไป, NTP ที่ถูก ACL บล็อกหรือ Key ไม่ตรงกัน, Syslog ที่หาย
กลางทางเพราะ Source-interface หรือ Severity Filter ผิด, SNMPv3 ที่ User/Group/View ไม่ครบ,
DNS บน Router ที่ Resolve ไม่ได้ 3 ชั้นสาเหตุ, และปิดท้ายด้วยบทเรียนที่สำคัญที่สุด — **NTP ที่พัง
สามารถลามไปทำให้ Certificate Validation และ Time-based ACL พังตามได้แบบมองไม่เห็นความเชื่อมโยง
เลยในตอนแรก** ✅ Lab เต็มรูปแบบใน Step 620 พิสูจน์ Pattern นี้ด้วยสถานการณ์จริงที่ 2 Ticket
ดูไม่เกี่ยวข้องกันเลย แต่แยกออกมาได้ว่าหนึ่งเป็นปัญหา Independent (Helper-address) และอีกหนึ่ง
เป็น Cascade 3 ชั้นเต็มรูปแบบ (NTP Key → Clock Drift → Time-based ACL)

บทเรียนที่สำคัญที่สุดที่ต้องนำติดตัวไปจาก Part นี้คือ **NTP ไม่ใช่แค่ "เรื่องเวลาให้ตรงกันเฉยๆ"
แต่เป็น Foundational Dependency ที่ Feature ความปลอดภัยระดับสูงหลายตัว (Certificate, Time-based
ACL, Kerberos/RADIUS Token) พึ่งพาอยู่โดยตรง** — เมื่อไล่ Troubleshoot ปัญหาที่ดูไม่เกี่ยวกับ
Infra Service เลย ให้ตรวจ `show clock detail` และ `show ntp status` เป็นหนึ่งในขั้นตอนแรกๆ
เสมอ เพราะปัญหาประเภทนี้มักกระทบทุกคนพร้อมกันแบบที่ Hardware/Config เจาะจงจุดเดียวทำไม่ได้

**พร้อมสำหรับ Part 63**: เราจะย้ายไปสู่การ Troubleshoot **NAT/PAT ขั้นสูง** ต่อจากพื้นฐานใน
[Part 15](part-015-nat-pat.md) — ครอบคลุม Scenario ที่ซับซ้อนกว่าเดิมมาก เช่น NAT Overlapping,
Double NAT, PAT Pool Exhaustion, และ NAT ที่ทำงานร่วมกับ VPN/DMVPN ที่เรียนไปแล้วใน
[Part 51-53](part-051-vrf-lite-gre-ipsec.md) พร้อม Diagnostic Process แบบเดียวกันกับ Part นี้

**ไปต่อ:** [Part 63 — Advanced NAT Scenarios →](part-063-advanced-nat-scenarios.md)
