# Part 64 — Path Control & PBR Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 631–640 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 63 — Advanced NAT Scenarios](part-063-advanced-nat-scenarios.md) ที่ปิดท้ายด้วยการ
> Troubleshoot NAT ที่พังเพราะ Asymmetric Routing (Step 625) ซึ่งโยงกลับไปที่แนวคิด PBR + IP SLA
> จาก [Part 32](part-032-policy-based-routing.md) — Part นี้จะกลับไปที่หัวใจของปัญหานั้นโดยตรง:
> **Path Control** คือชื่อรวมของทุกกลไกที่ตัดสินใจว่า Packet จะวิ่งไปทางไหน ไม่ว่าจะเป็น
> **Static Route + Administrative Distance** ([Part 9](part-009-static-routing.md)), **Dynamic
> Routing Protocol** (OSPF/EIGRP/BGP), หรือ **Policy-Based Routing (PBR)**
> ([Part 32](part-032-policy-based-routing.md)) — ปัญหาที่ยากที่สุดของ Path Control
> Troubleshooting ไม่ใช่การไม่รู้ว่าแต่ละกลไกทำงานอย่างไร (เรียนมาหมดแล้วทีละตัว) แต่คือการหาให้ได้
> ว่า **ตอนนี้ใครกำลังเป็นคนคุมเส้นทางนี้อยู่กันแน่ เมื่อมีหลายกลไกซ้อนทับกันบนอุปกรณ์เดียวกัน**
> Part นี้จึงเป็น Troubleshooting-Only เต็มรูปแบบ 10 Step ครอบคลุมกับดักที่พบจริงในสนามสอบและใน
> Production ตั้งแต่ PBR ACL พิมพ์ผิด, IP SLA Probe หลอกตา, Floating Static AD สลับกัน, ไปจนถึง
> Route Summarization ที่กลายเป็น Black Hole โดยไม่ตั้งใจ ปิดท้ายด้วย Lab ที่รวม 2 ปัญหาซ้อนกันแบบ
> ที่เกิดขึ้นจริงหลัง Change Window ที่พลาด

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 631 | Path Control Troubleshooting Methodology — ใครคุมเส้นทางนี้อยู่กันแน่ (Priority Order Checklist) |
| 632 | Troubleshoot: PBR ACL Wildcard Mask พิมพ์ผิด — กับดัก "PBR Permit Fallthrough" |
| 633 | Troubleshoot: IP SLA Probe Target พังด้วยเหตุผลที่ไม่เกี่ยวกับ Path จริง |
| 634 | Troubleshoot: Floating Static Route — Administrative Distance สลับกัน (Reversed) |
| 635 | Troubleshoot: PBR Apply ผิด Interface — ลืมกฎ "Inbound เท่านั้น" |
| 636 | Troubleshoot: Local PBR vs Transit PBR สลับกันผิดจุด |
| 637 | Troubleshoot: Recursive Static Route หลุด Resolution กลายเป็น Routing Loop |
| 638 | Troubleshoot: PBR Asymmetric Return Path ทำลาย Stateful Firewall/NAT |
| 639 | Troubleshoot: Route Summarization สร้าง Black Hole Route โดยไม่ตั้งใจ |
| 640 | Lab เต็มรูปแบบ: 2 ปัญหาซ้อนกันหลัง Change Window ที่พลาด (PBR ACL + AD Reversed) |

---

## Step 631 — Path Control Troubleshooting Methodology: ใครคุมเส้นทางนี้อยู่กันแน่

### 631.1 ปัญหาเฉพาะของ Path Control Troubleshooting

ทุก Part ก่อนหน้านี้สอนกลไกควบคุมเส้นทางแยกกันเป็นเรื่องๆ: Static Route + AD
([Part 9](part-009-static-routing.md)), OSPF/EIGRP/BGP Metric ([Part 56-59](part-056-eigrp-troubleshooting.md)),
และ PBR ([Part 32](part-032-policy-based-routing.md)) — ในโลกจริงอุปกรณ์ตัวเดียวมักมี**หลายกลไก
ทำงานพร้อมกัน**บน Interface เดียวกัน และเมื่อ Traffic วิ่งไป "ผิดทาง" คำถามแรกที่ต้องตอบไม่ใช่
"OSPF Config ผิดไหม" หรือ "Static Route AD ผิดไหม" แต่คือ **"อะไรกันแน่ที่กำลังตัดสินใจเส้นทางนี้
อยู่ ณ ขณะนี้"** — เพราะถ้าตอบผิดตัว จะเสียเวลาไล่ Debug ผิดจุดไปนานมาก (เช่น ไล่ OSPF Metric
ทั้งที่ตัวการจริงคือ PBR ที่ Apply อยู่คนละ Interface)

### 631.2 Priority Order: ลำดับที่ Cisco IOS ใช้ตัดสินใจ Forward Packet จริง

```
Packet เข้า Interface
        │
        ▼
┌──────────────────────────────────────────────┐
│ 1) Local PBR (เฉพาะ Traffic ที่ Router สร้างเอง)   │  ip local policy route-map (Step 636)
└──────────────────┬─────────────────────────────┘
        │ ไม่ Match หรือไม่ใช่ Local Traffic
        ▼
┌──────────────────────────────────────────────┐
│ 2) Transit PBR (ตรวจก่อน Routing Table เสมอ)       │  ip policy route-map (Part 32 Step 311.4)
└──────────────────┬─────────────────────────────┘
        │ ไม่ Match ACL (Fallthrough — Step 632)
        ▼
┌──────────────────────────────────────────────┐
│ 3) Longest Prefix Match (LPM)                   │  ตัดสินก่อน AD เสมอถ้า Prefix Length ต่างกัน
└──────────────────┬─────────────────────────────┘
        │ Prefix Length เท่ากันจากหลายแหล่ง
        ▼
┌──────────────────────────────────────────────┐
│ 4) Administrative Distance (AD ต่ำสุดชนะ)          │  Static=1, EIGRP=90, OSPF=110 (Part 9 Step 85)
└──────────────────┬─────────────────────────────┘
        │ แหล่งเดียวกัน (Protocol เดียวกัน) แต่ Metric ต่างกัน
        ▼
┌──────────────────────────────────────────────┐
│ 5) Routing Protocol Metric (ต่ำสุดชนะ)             │  OSPF Cost, EIGRP Composite Metric, BGP Attribute
└──────────────────┬─────────────────────────────┘
        │ AD และ Metric เท่ากันทุกประการ
        ▼
┌──────────────────────────────────────────────┐
│ 6) ECMP / CEF Load-Balancing (per-flow hash)     │  show ip cef exact-route
└──────────────────────────────────────────────────┘
```

> **จุดที่ต้องจำขึ้นใจที่สุดของ Step นี้**: **PBR (ทั้ง Local และ Transit) ถูกตรวจสอบก่อน Layer
> 3-6 ทั้งหมดเสมอ** — ถ้า PBR Apply อยู่และ Match Packet นั้น จะไม่มีทาง "เห็น" ผลลัพธ์ของ Static
> Route, OSPF Metric, หรือ ECMP เลยแม้แต่นิดเดียว เพราะ PBR ตัดสินใจเสร็จสิ้นไปแล้วก่อนที่ Router
> จะแม้แต่เริ่ม Lookup Routing Table — **นี่คือรากของกับดักในหัวข้อ 631.4**

### 631.3 Checklist การ Troubleshoot Path Control — ลำดับที่ต้องทำทุกครั้ง

| ลำดับ | คำถาม | คำสั่งตรวจสอบ | ถ้าใช่ ไปทำอะไรต่อ |
|---|---|---|---|
| 1 | Interface ขาเข้าของ Traffic นี้มี PBR (Transit) Apply อยู่ไหม | `show ip policy` | ไป `show route-map <name>` ดู Hit Counter (Step 632, 635) |
| 2 | Traffic นี้เป็น Traffic ที่ Router สร้างเอง (Syslog/NTP/SSH) ไหม | `show ip local policy` | ตรวจว่า Local PBR ผูกถูกหรือไม่ (Step 636) |
| 3 | ถ้าไม่มี PBR หรือ Fallthrough แล้ว — Routing Table บอกอะไร | `show ip route <dest>` | ดู AD/Metric/Next-Hop ที่แท้จริง |
| 4 | มีหลาย Source ของ Route ปลายทางเดียวกันไหม (Static ปนกับ Protocol) | `show ip route <dest>` (ดู "Known via") | เทียบ AD ตามตาราง Part 9 Step 85 |
| 5 | Next-Hop ที่ Route ชี้ไปจริงๆ "Reachable" อยู่ไหม (Recursive) | `show ip route <next-hop>` | ถ้า Recursive Lookup ล้มเหลว Route จะหายจาก RIB (Step 637) |
| 6 | ถ้าใช้ PBR + `verify-availability` — Track Object เป็นอะไร | `show track <obj>`, `show ip sla statistics` | ตรวจว่า Probe เอง Reliable ไหม (Step 633) |

### 631.4 ทำไม PBR Bug ถึง "ปลอมตัว" เป็น Routing Bug ได้

จาก [Part 32 Step 319.3](part-032-policy-based-routing.md) เรารู้แล้วว่า PBR ถูกออกแบบให้เป็น
**Fail-Open**: ถ้า ACL ไม่ Match เลยสักบรรทัด PBR จะเงียบๆ ปล่อย Packet Fallthrough ไปใช้ Routing
Table ปกติโดยไม่มี Error หรือ Log เตือนใดๆ — ผลคือวิศวกรที่เห็นอาการ "Traffic วิ่งเส้นทางที่ไม่ควร
วิ่ง" มักจะไปเปิด `show ip route` และ `show ip ospf neighbor` เป็นอันดับแรกตามสัญชาตญาณ (เพราะ
"เห็น" Routing Table เป็นสิ่งที่คุ้นเคยที่สุด) และไม่พบอะไรผิดปกติเลย เพราะ **Routing Table ทำงาน
ถูกต้อง 100% ตามที่มันควรจะเป็น — ปัญหาไม่ได้อยู่ที่ Layer นี้เลย** ตัวการจริงคือ PBR ที่ควร
"แทรก" ก่อนแต่กลับไม่ทำงาน (Step 632) หรือ Apply ผิดจุด (Step 635) — **นี่คือเหตุผลที่ Checklist
631.3 ต้องตรวจ `show ip policy` เป็นข้อที่ 1 เสมอ ก่อนจะเริ่มสงสัย Routing Protocol แม้แต่นิดเดียว**
เมื่อ Interface นั้นมี PBR Apply อยู่ Step 632-638 ที่เหลือใน Part นี้คือตัวอย่างจริงของกับดักนี้
ในหลายรูปแบบ

---

## Step 632 — Troubleshoot: PBR ACL Wildcard Mask พิมพ์ผิด (PBR Permit Fallthrough)

### 632.1 อาการที่รายงานเข้ามา

ทีม Network เปลี่ยน Config PBR-VOICE บน **CORE-SW1** (ต่อยอดจาก
[Part 32 Step 313](part-032-policy-based-routing.md)) เพื่อรองรับ VLAN 20 (VOICE) ที่ขยาย DHCP
Scope ใหม่ให้ครอบคลุม Host ตั้งแต่ `.1` ถึง `.254` เต็มวง — หลังเปลี่ยน Config Helpdesk รายงานว่า
**โทรศัพท์บางเครื่องใน VLAN 20 มี Call Quality แย่ลงเป็นบางครั้ง (Jitter สูง)** ในขณะที่โทรศัพท์
เครื่องอื่นในวงเดียวกันใช้งานปกติดี — ไม่มี Pattern ชัดเจนว่าเครื่องไหนมีปัญหา จนทีมสังเกตว่า
เครื่องที่มีปัญหาทั้งหมดได้ IP อยู่ในช่วง `10.10.20.128` ถึง `10.10.20.254`

### 632.2 Config ปัจจุบันบน CORE-SW1 (ที่ถูกแก้ไขไปแล้ว)

```
CORE-SW1# show ip access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255
    20 permit ip 10.10.20.0 0.0.0.63 any (48213 matches)
```

> **สังเกต Wildcard Mask ของบรรทัด 20 ให้ดี**: `0.0.0.63` ไม่ใช่ `0.0.0.255` — วิศวกรที่แก้ไข
> ACL นี้ตั้งใจจะ "ขยาย" Scope แต่พิมพ์ Wildcard Mask ผิดโดยคิดว่า `0.0.0.63` แปลว่า "ครอบคลุม
> มากขึ้น" (สับสนกับตรรกะของ Subnet Mask ที่เลขน้อยกว่า = ครอบคลุมมากกว่า — แต่ Wildcard Mask ตรรกะ
> กลับกัน: **เลขมากกว่า = ครอบคลุมมากกว่า**) ผลคือบรรทัด 20 ตอนนี้ครอบคลุมแค่ `10.10.20.0` ถึง
> `10.10.20.63` เท่านั้น (64 Address แรก) ไม่ใช่ทั้ง `/24`

### 632.3 วินิจฉัย — ทำตาม Checklist Step 631.3

**ขั้นที่ 1 — ยืนยันว่า PBR Apply อยู่ถูก Interface (ตัดความเป็นไปได้แบบ Step 635 ออกก่อน)**

```
CORE-SW1# show ip policy
Interface                Route map
Te1/0/1                  PBR-VOICE
Te1/0/2                  PBR-VOICE

CORE-SW1# show route-map PBR-VOICE
  Policy routing matches: 48213 packets, 6318942 bytes
```

Apply ถูก Interface และ Hit Counter รวมก็เพิ่มขึ้นเรื่อยๆ — **ดูเหมือนทำงานปกติ** นี่คือจุดที่
วิศวกรมือใหม่มักหยุดวินิจฉัยเพราะ Hit Counter ไม่ใช่ 0 แต่ต้องเจาะดูว่า Match **ทุก Host หรือแค่
บางส่วน**

**ขั้นที่ 2 — เจาะ Hit Counter แยกทีละบรรทัดของ ACL เอง**

```
CORE-SW1# show access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (3120 matches)
    20 permit ip 10.10.20.0 0.0.0.63 any (48213 matches)
```

บรรทัด `deny` ยัง Match ทั้ง `/24` เดิม (Wildcard `0.0.0.255` ถูกต้อง) แต่บรรทัด `permit` มี
Wildcard แคบกว่าที่ตั้งใจมาก — Host ที่ IP อยู่นอกช่วง `10.10.20.0-63` (เช่น `10.10.20.130`) จะ
**ไม่ Match บรรทัดใดใน ACL นี้เลย** ซึ่งตาม [Part 32 Step 319.3](part-032-policy-based-routing.md)
หมายความว่า **Clause นี้ไม่ Match → Fallthrough ไป Routing Table ปกติทันที**

**ขั้นที่ 3 — ยืนยันด้วย `debug ip policy` เทียบ Host ในช่วงกับนอกช่วง**

```
CORE-SW1# debug ip policy
! Host 10.10.20.45 (อยู่ในช่วง .0-.63) — Match ตามปกติ
IP: s=10.10.20.45 (TenGigabitEthernet1/0/1), d=8.8.8.8, len 200, policy match
IP: route map PBR-VOICE, item 10, permit

! Host 10.10.20.130 (นอกช่วง .0-.63) — ไม่มีบรรทัด "policy match" เลยแม้แต่บรรทัดเดียว!
```

Log ยืนยันชัดเจนว่ามีแค่ Host ในช่วง `.0-.63` เท่านั้นที่ PBR จับได้ — ส่วนที่เหลือ Fallthrough
แบบเงียบสนิทตามที่ Checklist 631.4 เตือนไว้

### 632.4 Root Cause

Wildcard Mask ของบรรทัด `permit` ใน ACL `VOICE-TO-INTERNET` พิมพ์ผิดจาก `0.0.0.255` (ครอบคลุม
`/24` เต็มวง) เป็น `0.0.0.63` (ครอบคลุมแค่ 64 Address แรก) — Host ในช่วง `.64` ถึง `.254` จึง
Fallthrough ไปใช้ OSPF ปกติแทน PBR ทำให้ Voice Traffic ของ Host เหล่านั้น **สลับไปมาระหว่าง
WAN-EDGE-1/WAN-EDGE-2 ตาม OSPF Metric** ซึ่งไม่มีการันตี Latency ต่ำเหมือนที่ PBR ตั้งใจไว้ — ตรงกับ
อาการ Jitter ที่ Helpdesk รายงานเป๊ะ (เฉพาะ Host ในช่วง IP ที่หลุดจาก ACL)

### 632.5 แก้ไขและ Verify

```
CORE-SW1(config)# ip access-list extended VOICE-TO-INTERNET
CORE-SW1(config-ext-nacl)# no permit ip 10.10.20.0 0.0.0.63 any
CORE-SW1(config-ext-nacl)# permit ip 10.10.20.0 0.0.0.255 any
CORE-SW1(config-ext-nacl)# exit
```

```
CORE-SW1# show access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (3121 matches)
    20 permit ip 10.10.20.0 0.0.0.255 any (48980 matches)

! ทดสอบซ้ำจาก Host 10.10.20.130
CORE-SW1# debug ip policy
IP: s=10.10.20.130 (TenGigabitEthernet1/0/1), d=8.8.8.8, len 200, policy match
IP: route map PBR-VOICE, item 10, permit
IP: TenGigabitEthernet1/0/1 to GigabitEthernet1/0/1 10.10.254.1
```

`10.10.20.130` ถูก PBR จับแล้ว — ยืนยันว่าทุก Host ใน VLAN 20 ถูก Policy Route ผ่าน WAN-EDGE-1
เหมือนกันหมดตาม Design เดิม

> **บทเรียนของ Step นี้**: ทุกครั้งที่แก้ไข ACL ที่ผูกกับ PBR ให้ตรวจสอบ Hit Counter แยกทีละบรรทัด
> เสมอ (`show access-lists`) ไม่ใช่แค่ดู `Policy routing matches` รวมที่ Route-map เพราะตัวเลขรวม
> ที่ "ดูเหมือนทำงาน" (ไม่ใช่ 0) อาจกำลังปกปิดว่ามันทำงาน**ไม่ครบ**อยู่ก็ได้ — ต้องเทียบจำนวน
> Traffic จริงที่ควร Match กับ Hit Counter เสมอ

---

## Step 633 — Troubleshoot: IP SLA Probe Target พังด้วยเหตุผลที่ไม่เกี่ยวกับ Path จริง

### 633.1 อาการที่รายงานเข้ามา

หลังใช้งาน PBR + IP SLA ตาม [Part 32 Step 320](part-032-policy-based-routing.md) มาหลายเดือนโดย
ไม่มีปัญหา ทันใดนั้น Voice Traffic ทั้งหมดของ VLAN 20 **ย้ายไปวิ่งผ่าน WAN-EDGE-2 (Backup Path)
อย่างถาวร** ทั้งที่ `WAN-EDGE-1` (Primary) ยังใช้งานได้ปกติทุกประการ — User ไม่มีใครรายงานปัญหา
Voice โดยตรง (WAN-EDGE-2 ยังพอใช้ได้) แต่ทีม Network สังเกตเห็นจาก Bandwidth Utilization Report
รายเดือนว่า Link WAN-EDGE-2 มี Traffic สูงผิดปกติต่อเนื่องมาหลายวัน

### 633.2 บริบท: มีการเปลี่ยน IP SLA Target เมื่อสัปดาห์ก่อน

ทีม Network Monitoring เสนอให้ปรับ IP SLA ของ Step 314 จาก "Probe แค่ Interface ของ WAN-EDGE-1"
(`10.10.254.1`) เป็น "Probe ให้ลึกกว่านั้น" เพื่อทดสอบ **Internet Reachability จริง** ไม่ใช่แค่
Link ระหว่าง CORE-SW1 กับ WAN-EDGE-1 (เหตุผลฟังดูสมเหตุสมผล — ถ้า Link ถึง WAN-EDGE-1 ยัง Up แต่
WAN-EDGE-1 เองเสีย Uplink ไปยัง ISP-RTR แล้ว Probe แบบเดิมจะไม่รู้เลย) จึงเปลี่ยน Target เป็น
Monitoring Host ภายนอกที่ทีม Cloud Operation ดูแลอยู่ (`198.51.100.50`):

```
CORE-SW1(config)# ip sla 1
CORE-SW1(config-ip-sla)# no icmp-echo 10.10.254.1 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla)# icmp-echo 198.51.100.50 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla-echo)# frequency 5
CORE-SW1(config-ip-sla-echo)# timeout 1000
CORE-SW1(config-ip-sla-echo)# exit
```

### 633.3 วินิจฉัย

**ขั้นที่ 1 — ตรวจ Track Object ตาม Checklist 631.3 ข้อ 6**

```
CORE-SW1# show track 1
Track 1
  IP SLA 1 reachability
  Reachability is Down
    14 changes, last change 3d02h ago
  Tracked by:
    Route Map PBR-VOICE 10
```

`Reachability is Down` มา 3 วันแล้ว — Route-map จึงข้าม Next-Hop แรกไปใช้ WAN-EDGE-2 ตลอดเวลา
ตรงกับอาการที่สังเกต — แต่คำถามคือ **WAN-EDGE-1 จริงๆ เสียหรือไม่**

**ขั้นที่ 2 — ตรวจสอบ WAN-EDGE-1 เองโดยตรง (ไม่ผ่าน SLA) และตรวจ IP SLA Statistics**

```
CORE-SW1# ping 10.10.254.1
!!!!!
Success rate is 100 percent (5/5)

CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Address         Interface
1.1.1.21          1   FULL/  -        10.10.254.1     GigabitEthernet1/0/1

! ทดสอบจาก WAN-EDGE-1 ไปยัง ISP-RTR ตรงๆ — ก็ผ่านปกติ
WAN-EDGE-1# ping 203.0.113.1
!!!!!
Success rate is 100 percent (5/5)

CORE-SW1# show ip sla statistics 1
IPSLA operation id: 1
Latest operation return code: Timeout
Number of successes: 0
Number of failures: 4821
```

**WAN-EDGE-1 สุขภาพดีสมบูรณ์ทุกจุด** (Link Up, OSPF Neighbor FULL, Uplink ไป ISP-RTR ปกติ) แต่
`Number of successes: 0` ที่ IP SLA เองตลอด 3 วัน — ปัญหาต้องอยู่ที่ตัว Probe ไม่ใช่ที่ Path

**ขั้นที่ 3 — Ping ตรงไปยัง Target ของ SLA แล้วสอบถามทีมที่ดูแล**

```
CORE-SW1# ping 198.51.100.50 source GigabitEthernet1/0/1
.....
Success rate is 0 percent (0/5)
```

Ping ตรงไปยัง `198.51.100.50` ล้มเหลว 100% แต่ Path ไป Internet ผ่าน WAN-EDGE-1 ยืนยันแล้วว่าใช้
งานได้ปกติในขั้นที่ 2 จึงต้องเป็นปัญหาที่**ปลายทางเอง** — สอบถามทีม Cloud Operation แล้วยืนยันว่า
**Server `198.51.100.50` ถูก Decommission ย้ายไปยัง Cloud Provider ใหม่เมื่อ 3 วันก่อน** (ตรงกับ
เวลาที่ Track เริ่ม Down พอดี) โดยไม่ได้แจ้งทีม Network เพราะคิดว่าเป็นแค่ "Monitoring Host เฉยๆ
ไม่กระทบใคร" — **นี่คือ "เหตุผลที่ไม่เกี่ยวกับ Path จริง" ตัวจริง**: Target หายไปเพราะการ
เปลี่ยนแปลงของทีมอื่นที่ไม่เกี่ยวกับ WAN Link เลยแม้แต่นิดเดียว

### 633.4 Root Cause

การเลือก IP SLA Probe Target เป็น Host เฉพาะเจาะจงตัวเดียวที่ไม่ได้อยู่ภายใต้การควบคุมของทีม
Network เอง (Server ของทีมอื่น) ทำให้ **สุขภาพของ PBR Failover ทั้งระบบขึ้นอยู่กับความมีชีวิตของ
Host ตัวนั้นโดยตรง** — เมื่อ Host นั้นถูก Decommission ด้วยเหตุผลที่ไม่เกี่ยวกับ Network Path เลย
Track Object ก็ยัง "เชื่อ" ว่า Path เสียอยู่ดี (เพราะมันวัดแค่ "Ping ถึง Target นี้ไหม" ไม่ได้วัด
"Path เส้นนี้ใช้งานได้จริงไหม" โดยตรง) ทำให้ PBR Fail-Over ไปยัง Backup แบบถาวรโดยไม่จำเป็น สิ้นเปลือง
Bandwidth ของ Link สำรองต่อเนื่องหลายวันโดยไม่มีใครรู้จนกว่าจะมาดู Utilization Report

### 633.5 แก้ไขและแนวทางป้องกันในอนาคต

**แก้เร่งด่วน**: เปลี่ยน Target กลับไปเป็นสิ่งที่ทีม Network เองควบคุมได้เต็มที่ — Interface ของ
WAN-EDGE-1 (`10.10.254.1`) เหมือนเดิม เพราะเป็น Address ที่**อยู่ภายใต้ Administrative Boundary
ของทีมเราเอง 100%**:

```
CORE-SW1(config)# ip sla 1
CORE-SW1(config-ip-sla)# no icmp-echo 198.51.100.50 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla)# icmp-echo 10.10.254.1 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla-echo)# exit
```

**แก้แบบยั่งยืนกว่า (ถ้าต้องการทดสอบ End-to-End จริงตามเจตนาเดิมของทีม Monitoring)**: ใช้ **Track
List** (Boolean AND/OR) รวมหลาย IP SLA Operation เข้าด้วยกันแทนการพึ่ง Target เดียว — Probe ทั้ง
Interface ของ WAN-EDGE-1 (Local Link) **และ** ISP-RTR โดยตรง (Uplink จริง) พร้อมกัน ให้ Track
Down เฉพาะเมื่อ**ทั้งสอง Probe ล้มเหลวพร้อมกันเท่านั้น**:

```
CORE-SW1(config)# ip sla 2
CORE-SW1(config-ip-sla)# icmp-echo 203.0.113.1 source-interface GigabitEthernet1/0/1
CORE-SW1(config-ip-sla-echo)# exit
CORE-SW1(config)# ip sla schedule 2 life forever start-time now
CORE-SW1(config)# track 2 ip sla 2 reachability
CORE-SW1(config-track)# exit
!
CORE-SW1(config)# track 100 list boolean and
CORE-SW1(config-track)# object 1
CORE-SW1(config-track)# object 2
CORE-SW1(config-track)# exit
```

`boolean and` หมายถึง Track 100 จะ Up ก็ต่อเมื่อ **ทั้ง Track 1 (Local Link) และ Track 2 (ISP
Uplink) Up พร้อมกัน** — ป้องกันการ Fail-Over จาก Probe เดียวที่ไม่เสถียร โดยยังคงทดสอบ End-to-End
จริงตามเจตนาเดิมของทีม Monitoring แต่ปลอดภัยกว่า

> **บทเรียนสำคัญที่สุดของ Step นี้**: ก่อนเปลี่ยน IP SLA Probe Target ทุกครั้ง ต้องถามคำถามนี้เสมอ
> **"ถ้า Target นี้หายไปด้วยเหตุผลที่ไม่เกี่ยวกับ Network Path ของเราเลย เราจะรู้ทันไหม"** — Target
> ที่ดีที่สุดคือสิ่งที่ทีม Network เองควบคุมวงจรชีวิตได้ทั้งหมด (Interface ของ Router เพื่อนบ้าน)
> ถ้าจำเป็นต้องทดสอบ End-to-End ผ่าน Host ภายนอก ต้องมี Track List ผสมหลาย Probe เพื่อลดจุด
> Single-Point-of-Failure ของการ Monitoring เอง

---

## Step 634 — Troubleshoot: Floating Static Route — Administrative Distance สลับกัน

### 634.1 บริบท: Config เดิมตาม Part 9 Step 90

CORE-SW1 มี Default Route คู่แบบ Primary/Floating Backup ตาม
[Part 9 Step 86](part-009-static-routing.md) มาตั้งแต่แรก (ปรับ Interface ไปยัง Port-channel1
ตาม [00-ip-address-plan.md](00-ip-address-plan.md) ที่เป็นค่า Canonical ตั้งแต่ Part 11):

```
! Config ที่ตั้งใจไว้แต่เดิม (ถูกต้อง)
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2 5
```

- บรรทัดแรก: Default Route หลักผ่าน **WAN-EDGE-1 โดยตรง** (AD=1, ค่า Default)
- บรรทัดสอง: Default Route สำรอง (Floating) ผ่าน **CORE-SW2 → WAN-EDGE-2** (AD=5 — สูงกว่าเจตนา
  ให้ "ลอย" อยู่เฉยๆ จนกว่า Primary จะหาย)

### 634.2 อาการที่รายงานเข้ามาหลัง Change Window

ทีม Network ทำ Change Window เพื่อ "เพิ่มความชัดเจนของ Comment" ในบรรทัด Static Route (ไม่ได้
ตั้งใจเปลี่ยน Logic) โดยลบ Route เดิมแล้วพิมพ์ใหม่ — หลังจากนั้น **User ทั้ง Campus รายงานว่า
Internet ช้าลงเล็กน้อยแต่สม่ำเสมอ** (ไม่ใช่ Intermittent) และทีม Monitoring สังเกตว่า Link ไปยัง
**WAN-EDGE-1 (Primary ที่ควรรับ Traffic เกือบทั้งหมด) มี Utilization ต่ำผิดปกติ** ในขณะที่ Link
Port-channel1 ไปยัง CORE-SW2 (ที่ควรเป็นแค่ Backup) มี Utilization สูงตลอดเวลา

### 634.3 วินิจฉัย

**ขั้นที่ 1 — ตรวจ Routing Table ปัจจุบัน**

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.255.0.2, via Port-channel1
      Route metric is 0, traffic share count is 1
```

**Next-Hop ที่ถูกเลือกคือ `10.255.0.2` (ผ่าน CORE-SW2/WAN-EDGE-2) ที่ AD=1** — นี่คือสิ่งที่ผิด
ทันที: Route ที่ควรเป็น Backup (Floating, ตั้งใจ AD=5) กลับมี AD=1 และเป็น Route ที่ถูกเลือกใช้งาน
จริง ส่วน Route ที่ควรเป็น Primary (ผ่าน WAN-EDGE-1) หายไปจาก Output นี้เลย (เพราะ AD แพ้ ไม่ถูก
ติดตั้งใน RIB — ตรงตามพฤติกรรมที่ [Part 9 Step 85](part-009-static-routing.md) อธิบายไว้)

**ขั้นที่ 2 — ตรวจ Running-Config เพื่อดูว่า AD ที่ตั้งใจพิมพ์จริงๆ คืออะไร**

```
CORE-SW1# show running-config | include ip route 0.0.0.0
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1 5
ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2
```

**พบตัวการ**: บรรทัดที่ควรเป็น Primary (ผ่าน WAN-EDGE-1) ถูกพิมพ์ใหม่พร้อมเลข **`5`** ต่อท้าย
(กลายเป็น AD=5 โดยไม่ตั้งใจ) ส่วนบรรทัดที่ควรเป็น Backup (ผ่าน CORE-SW2) ถูกพิมพ์ใหม่**โดยไม่มี
เลข AD ต่อท้ายเลย** (จึงใช้ค่า Default = 1) — วิศวกรที่พิมพ์ใหม่ตอน Copy-Paste จาก Note สลับ
ตำแหน่งเลข `5` ผิดบรรทัดระหว่างสองคำสั่ง **สลับ (Reverse) ความสัมพันธ์ Primary/Backup ที่ตั้งใจไว้
ทั้งหมด** — Static Route ทั้งสองยัง Valid และ Syntax ถูกต้อง 100% (Config ไม่ Error เตือนอะไรเลย)
เพราะ IOS ไม่มีทางรู้ "เจตนา" ของวิศวกรว่าเส้นไหนควรเป็น Primary

**ขั้นที่ 3 — ยืนยันด้วยการเทียบ AD ทั้งสองบรรทัดตรงๆ**

| Static Route | Next-Hop | AD ที่ตั้งใจ | AD ที่ Config จริง | ผลลัพธ์ |
|---|---|---|---|---|
| ผ่าน WAN-EDGE-1 (ควรเป็น Primary) | `10.10.254.1` | 1 (Default) | **5** | ไม่ถูกติดตั้งใน RIB ตราบใดที่อีกเส้นยัง AD ต่ำกว่า |
| ผ่าน CORE-SW2/WAN-EDGE-2 (ควรเป็น Backup) | `10.255.0.2` | 5 (Floating) | **1** | ถูกติดตั้งเป็น Active Route ตลอดเวลา (AD ต่ำสุด) |

ตารางนี้ยืนยันตรงตัวว่า **AD ของทั้งสองบรรทัดสลับกันโดยสมบูรณ์** — ผลคือ "Floating" Backup Route
ไม่ได้ทำหน้าที่ "Activate เมื่อ Primary ล่ม" ตามที่ออกแบบไว้เลย เพราะมันกลาย เป็น **Active
ตลอดเวลาโดยไม่ต้องรอ Primary ล่มก่อน** (Primary ที่แท้จริงถูกกันไม่ให้ทำงานไปตั้งแต่ต้นด้วย AD=5
ของมันเอง) — Internet Traffic ทั้งหมดวิ่งผ่าน CORE-SW2 → WAN-EDGE-2 ตลอดเวลาโดยไม่จำเป็น ทำให้
Latency สูงกว่าปกติสม่ำเสมอ (ตรงกับอาการ "ช้าแต่สม่ำเสมอ" ไม่ใช่ Intermittent เพราะไม่มี Failover
เกิดขึ้นจริงเลย — มันแค่ "ผิดเส้นทางถาวร" ตั้งแต่ Change Window)

### 634.4 Root Cause

Administrative Distance ของ Static Route คู่ Primary/Floating ถูกพิมพ์สลับกันระหว่าง Change
Window (ค่า AD ที่ควรอยู่กับ Backup ดันไปอยู่กับ Primary และในทางกลับกัน) ทำให้ Route ที่ตั้งใจเป็น
Backup กลายเป็น Route ที่ Active ตลอดเวลา ส่วน Route ที่ตั้งใจเป็น Primary ไม่ถูกใช้งานเลยแม้จะ
Healthy 100% — Static Route AD ไม่มีการ Validate ความสมเหตุสมผลของ "เจตนา Design" ใดๆ ทำให้ Config
ที่ผิดสลับกันแบบนี้ **ไม่ Error และไม่มี Warning ปรากฏเลย** ต้องอาศัยการเทียบ Running-Config
กับ Design Document เท่านั้นจึงจะจับได้

### 634.5 แก้ไขและ Verify

ต้องลบ Route เดิมทิ้งก่อนเสมอ (เปลี่ยนค่า AD ของ Static Route ที่มีอยู่แล้วทำไม่ได้โดยการพิมพ์
ทับตรงๆ — ต้อง `no` ก่อนแล้วพิมพ์ใหม่ เพราะ IOS มองว่า AD ที่ต่างกันคือ Entry คนละตัว):

```
CORE-SW1(config)# no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1 5
CORE-SW1(config)# no ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2
!
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2 5
```

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.10.254.1, via GigabitEthernet1/0/1
      Route metric is 0, traffic share count is 1

CORE-SW1# show running-config | include ip route 0.0.0.0
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1
ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2 5
```

**ทดสอบ Failover จริงเพื่อยืนยันว่าความสัมพันธ์ถูกต้องแล้ว**:

```
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# shutdown
CORE-SW1(config-if)# end

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 5, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.255.0.2, via Port-channel1

CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# no shutdown
```

Route AD=5 เข้ามาแทนที่อัตโนมัติทันทีที่ Primary หาย — ยืนยันว่า Floating Static Route ทำหน้าที่
ถูกต้องตาม Design แล้ว

> **บทเรียนของ Step นี้**: ทุกครั้งที่แก้ไข Static Route คู่ Primary/Floating ให้ Verify ด้วย
> `show ip route <prefix>` **ทันทีหลัง Change** เสมอ (ดูว่า Next-Hop ที่ถูกเลือกจริงตรงกับที่
> ตั้งใจไหม) — อย่าเชื่อแค่ว่า "Syntax ผ่าน ไม่มี Error" เพราะ AD ที่สลับกันจะไม่มี Error ใดๆ
> ทั้งสิ้น และให้ทดสอบ Failover จริง (`shutdown` ชั่วคราวบน Interface Primary) เป็นส่วนหนึ่งของ
> Change Verification มาตรฐานเสมอเมื่อแก้ไข Floating Static Route

---

## Step 635 — Troubleshoot: PBR Apply ผิด Interface — ลืมกฎ "Inbound เท่านั้น"

### 635.1 อาการที่รายงานเข้ามา

ทีม Network ต้องการเพิ่ม PBR ใหม่ชื่อ `PBR-SALES-BACKUP` บน CORE-SW1 เพื่อบังคับให้ Traffic ของ
VLAN 10 (SALES) ที่มุ่งหน้าไปยัง Partner Site แห่งหนึ่ง (ผ่าน Internet, Subnet `198.51.100.0/24`)
วิ่งออกทาง **WAN-EDGE-2** เสมอ (Business Requirement ใหม่: Partner นี้ทำ Peering เฉพาะกับ ISP
ของ WAN-EDGE-2) — หลัง Config เสร็จ ทดสอบแล้ว **Traffic ยังวิ่งออกทาง WAN-EDGE-1 ตามปกติ ไม่ถูก
Policy Route เลยแม้แต่ Packet เดียว**

### 635.2 Config ที่ทีมทำไว้

```
CORE-SW1(config)# ip access-list extended SALES-TO-PARTNER
CORE-SW1(config-ext-nacl)# permit ip 10.10.10.0 0.0.0.255 198.51.100.0 0.0.0.255
CORE-SW1(config-ext-nacl)# exit
!
CORE-SW1(config)# route-map PBR-SALES-BACKUP permit 10
CORE-SW1(config-route-map)# match ip address SALES-TO-PARTNER
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.6
CORE-SW1(config-route-map)# exit
!
! วิศวกร Apply ที่ Interface ที่ Traffic "ออก" ไปยัง WAN (คิดว่าเป็นจุดที่ถูกต้องเพราะเป็นจุดที่
! Traffic นี้กำลังจะไปสู่ Internet)
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# description ** Uplink to WAN-EDGE-1 **
CORE-SW1(config-if)# ip policy route-map PBR-SALES-BACKUP
CORE-SW1(config-if)# exit
```

> **จุดที่ผิดคือตรรกะการเลือก Interface**: วิศวกรคิดว่า "Traffic นี้กำลังจะออกไปยัง Internet
> ผ่าน Uplink ตัวนี้ ดังนั้นควร Apply PBR ที่นี่" — แต่ลืมกฎเหล็กจาก
> [Part 32 Step 312.5](part-032-policy-based-routing.md): **`ip policy route-map` ทำงานเฉพาะ
> ทิศทาง Inbound เท่านั้น** — Traffic ของ VLAN 10 ที่มาจาก DIST-SW1 นั้น **"เข้า" CORE-SW1 ทาง
> Te1/0/1 (Uplink จาก DIST-SW1)** ไม่ใช่ทาง Gi1/0/1 (ซึ่งเป็นจุดที่มันกำลังจะ "ออก" ไปหา
> WAN-EDGE-1) — Gi1/0/1 ในทิศทาง Inbound จะเห็นแค่ Traffic ที่มาจาก WAN-EDGE-1 (เช่น Return
> Traffic จาก Internet) ไม่ใช่ Traffic ขาไปของ VLAN 10 เลย

### 635.3 วินิจฉัยตาม Checklist Step 631.3

**ขั้นที่ 1 — ตรวจว่า PBR ผูกอยู่ Interface ไหน**

```
CORE-SW1# show ip policy
Interface                Route map
Te1/0/1                  PBR-VOICE
Te1/0/2                  PBR-VOICE
Gi1/0/1                  PBR-SALES-BACKUP
```

Route-map ผูกอยู่จริง (ไม่ใช่ปัญหาการ Apply ไม่สำเร็จ) — แต่ผูกอยู่ที่ **Gi1/0/1** ซึ่งเป็น
Interface ไปยัง WAN-EDGE-1 ไม่ใช่ Interface ที่รับ Traffic ของ VLAN 10 เข้ามา (Te1/0/1)

**ขั้นที่ 2 — ตรวจ Hit Counter**

```
CORE-SW1# show route-map PBR-SALES-BACKUP
route-map PBR-SALES-BACKUP, permit, sequence 10
  Match clauses:
    ip address (access-lists): SALES-TO-PARTNER
  Set clauses:
    ip next-hop 10.10.254.6
  Policy routing matches: 0 packets, 0 bytes
```

`0 packets` นิ่งสนิทแม้ทดสอบส่ง Traffic จริงไปแล้วหลายครั้ง — ตรงกับกับดักที่
[Step 631.4](#631-4-ทำไม-pbr-bug-ถึง-ปลอมตัว-เป็น-routing-bug-ได้) เตือนไว้ แต่คราวนี้สาเหตุ
ต่างจาก Step 632 (ACL ไม่ Match) — ต้องตรวจ ACL ด้วยตัวเองต่อ

**ขั้นที่ 3 — ตรวจ ACL ว่า Match ถูกไหม (ตัดความเป็นไปได้ที่เป็นแบบ Step 632)**

```
CORE-SW1# show access-lists SALES-TO-PARTNER
Extended IP access list SALES-TO-PARTNER
    10 permit ip 10.10.10.0 0.0.0.255 198.51.100.0 0.0.0.255 (0 matches)
```

`0 matches` ที่ตัว ACL เองด้วย — นี่คือจุดสำคัญที่แยกกับดัก 2 แบบออกจากกัน: **ถ้า ACL มี Hit
Counter แต่ Route-map ไม่มี = ปัญหาที่ Set Clause หรือ Next-Hop (Step 632/633)** ส่วน **ถ้า ACL
เองก็ 0 Hit เหมือนกัน = Packet ไม่เคยถูกส่งผ่าน ACL นี้เลย** ซึ่งหมายความว่า **Interface ที่ Apply
Policy ไม่เคยเห็น Traffic นี้ "เข้า" มาจริงๆ** — นำไปสู่คำถามข้อ 635.2 ทันที: Interface ไหนกันแน่ที่
Traffic ของ VLAN 10 "เข้า" CORE-SW1

**ขั้นที่ 4 — ยืนยันทิศทางการไหลของ Traffic จริงด้วย `debug ip policy` ที่ Interface ที่ถูกต้อง**

```
CORE-SW1# debug ip policy
CORE-SW1#
IP: s=10.10.10.55 (TenGigabitEthernet1/0/1), d=198.51.100.20, len 200, FIB policy rejected(no match)
```

Log ยืนยันว่า Traffic เข้ามาทาง **`TenGigabitEthernet1/0/1`** (Uplink จาก DIST-SW1) ไม่ใช่
`GigabitEthernet1/0/1` ที่ Apply Policy ไว้ — คำว่า `FIB policy rejected(no match)` ในที่นี้
หมายถึง Interface นี้ (Te1/0/1) **ไม่มี Policy ผูกอยู่เลย** จึงไม่มีการตรวจ ACL ใดๆ เกิดขึ้นตั้งแต่
ต้น (คนละสาเหตุกับ Step 632 ที่ ACL ถูกตรวจแต่ไม่ Match)

### 635.4 Root Cause

`ip policy route-map PBR-SALES-BACKUP` ถูก Apply บน **Interface ขาออกที่ Traffic กำลังจะไปหา
WAN-EDGE-1 (Gi1/0/1)** ซึ่งเป็นทิศทางที่ผิดตามธรรมชาติของ PBR (Inbound เท่านั้น) — Traffic ของ
VLAN 10 ที่มาจาก DIST-SW1 เข้า CORE-SW1 ทาง **Te1/0/1** ต่างหาก ทำให้ Route-map ไม่มีโอกาสได้
ตรวจสอบ Traffic นี้เลยแม้แต่ Packet เดียว — Config ผ่านโดยไม่มี Error เพราะ Syntax ถูกต้องสมบูรณ์
IOS ไม่มีทางรู้ว่า Interface ที่เลือก Apply "ไม่เคยเห็น" Traffic ที่ต้องการ Policy Route

### 635.5 แก้ไขและ Verify

```
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# no ip policy route-map PBR-SALES-BACKUP
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# description ** Uplink to DIST-SW1 (SALES/VOICE block) **
CORE-SW1(config-if)# ip policy route-map PBR-SALES-BACKUP
CORE-SW1(config-if)# exit
!
! อย่าลืม Apply ให้ครบทุก Interface ที่ VLAN 10 อาจเข้ามาได้ (เหมือนเหตุผลใน Part 32 Step 313.3
! — HSRP Active สลับได้ระหว่าง DIST-SW1/DIST-SW2)
CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# ip policy route-map PBR-SALES-BACKUP
CORE-SW1(config-if)# exit
```

```
CORE-SW1# show ip policy
Interface                Route map
Te1/0/1                  PBR-VOICE, PBR-SALES-BACKUP     ! เห็น Warning: 1 Interface ผูกได้แค่ 1 Route-map เท่านั้น
```

> **ข้อควรระวังเพิ่มเติมที่พบระหว่างแก้ไข**: หนึ่ง Interface ผูก `ip policy route-map` ได้**แค่
> Route-map เดียวเท่านั้น** — ถ้า Te1/0/1 มี `PBR-VOICE` อยู่แล้วจาก Part 32 การเพิ่ม
> `PBR-SALES-BACKUP` เข้าไปอีกจะ **แทนที่ Route-map เดิมทันที ไม่ใช่การเพิ่มเสริม** ต้องรวม Logic
> ทั้งสองเข้าไปเป็น Route-map เดียวกันแบบหลาย Sequence แทน:

```
CORE-SW1(config)# route-map PBR-VOICE permit 10
CORE-SW1(config-route-map)# match ip address VOICE-TO-INTERNET
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.10.254.1 10 track 1
CORE-SW1(config-route-map)# set ip next-hop verify-availability 10.10.254.6 20 track 2
CORE-SW1(config-route-map)# exit
CORE-SW1(config)# route-map PBR-VOICE permit 20
CORE-SW1(config-route-map)# match ip address SALES-TO-PARTNER
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.6
CORE-SW1(config-route-map)# exit
!
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# ip policy route-map PBR-VOICE
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# ip policy route-map PBR-VOICE
CORE-SW1(config-if)# exit
```

```
CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  ...
  Policy routing matches: 51204 packets, 6698112 bytes
route-map PBR-VOICE, permit, sequence 20
  Match clauses:
    ip address (access-lists): SALES-TO-PARTNER
  Set clauses:
    ip next-hop 10.10.254.6
  Policy routing matches: 842 packets, 110124 bytes
```

Sequence 20 เริ่มมี Hit Counter เพิ่มขึ้นแล้ว — ยืนยันว่า SALES Traffic ไปยัง Partner ถูก Policy
Route ผ่าน WAN-EDGE-2 สำเร็จ ในขณะที่ Sequence 10 (VOICE เดิม) ยังทำงานต่อเนื่องไม่กระทบกัน

---

## Step 636 — Troubleshoot: Local PBR vs Transit PBR สลับกันผิดจุด

### 636.1 อาการที่รายงานเข้ามา

ทีม Network ต้องการบังคับให้ **NTP Query ที่ CORE-SW1 ส่งออกไปหา NTP Server ภายนอก** (ผ่าน
Internet, ตาม [Part 17](part-017-ntp-syslog-snmp.md)) วิ่งผ่าน **WAN-EDGE-1 เสมอ** (เหตุผล:
NTP Server หลักอยู่ใน Range ที่ Peering กับ ISP ของ WAN-EDGE-1 เท่านั้น Latency ต่ำกว่าชัดเจน) —
วิศวกร Copy Pattern จาก Part 32 Step 313 มาปรับใช้ แต่ทดสอบแล้ว **NTP Traffic ยังคงสลับไปมาระหว่าง
WAN-EDGE-1/WAN-EDGE-2 ตาม OSPF ปกติ ไม่ถูก Policy Route เลย**

### 636.2 Config ที่ทีมทำไว้ (ผิด — ใช้ Transit PBR กับ Traffic ที่ Router สร้างเอง)

```
CORE-SW1(config)# ip access-list extended NTP-QUERY-TRAFFIC
CORE-SW1(config-ext-nacl)# permit udp host 1.1.1.1 any eq 123
CORE-SW1(config-ext-nacl)# exit
!
CORE-SW1(config)# route-map PBR-NTP permit 10
CORE-SW1(config-route-map)# match ip address NTP-QUERY-TRAFFIC
CORE-SW1(config-route-map)# set ip next-hop 10.10.254.1
CORE-SW1(config-route-map)# exit
!
! ผิดจุด: Apply แบบ Transit PBR (ผูกกับ Interface) ทั้งที่ Traffic นี้ Originate จาก Router เอง
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# ip policy route-map PBR-NTP
CORE-SW1(config-if)# exit
```

### 636.3 วินิจฉัยตาม Checklist Step 631.3

**ขั้นที่ 1 — ตอบคำถามข้อ 2 ของ Checklist ก่อน: Traffic นี้ Router สร้างเองหรือเป็น Transit**

NTP Query ที่ CORE-SW1 ส่งออกไปเอง (Source = Loopback0 `1.1.1.1` ของ CORE-SW1 เอง ตาม
[Part 17](part-017-ntp-syslog-snmp.md)) **Originate จากตัว Router เอง 100%** — ไม่ใช่ Traffic ที่
"ผ่าน" Router มาจาก Host อื่น จึงจัดเป็น **Local Traffic** ตามนิยามใน
[Part 32 Step 316.1](part-032-policy-based-routing.md) ทันที — คำตอบของ Checklist ข้อ 1 (Transit
PBR) จึงไม่มีทางถูกเลยตั้งแต่ต้น

**ขั้นที่ 2 — ยืนยันด้วย Hit Counter**

```
CORE-SW1# show route-map PBR-NTP
route-map PBR-NTP, permit, sequence 10
  Match clauses:
    ip address (access-lists): NTP-QUERY-TRAFFIC
  Set clauses:
    ip next-hop 10.10.254.1
  Policy routing matches: 0 packets, 0 bytes

CORE-SW1# show access-lists NTP-QUERY-TRAFFIC
Extended IP access list NTP-QUERY-TRAFFIC
    10 permit udp host 1.1.1.1 any eq 123 (0 matches)
```

`0 matches` ทั้งคู่เหมือน Step 635 — แต่สาเหตุคนละแบบกันโดยสิ้นเชิง

**ขั้นที่ 3 — ตรวจ `show ip local policy` เทียบดู**

```
CORE-SW1# show ip local policy
Local policy routing is disabled
```

**"Local policy routing is disabled"** — คือคำตอบตรงๆ: ไม่มีการ Config Local PBR เลยบน Router
นี้ ทั้งที่ Traffic ที่ต้องการควบคุมเป็น Local Traffic ล้วนๆ — `ip policy route-map` ที่ Apply ไว้
บน Te1/0/1 (Transit PBR) **ไม่มีทางถูกเรียกใช้กับ Traffic ที่ Router สร้างขึ้นเองได้เลย** เพราะ
Traffic นั้นไม่เคย "เข้า" Interface ใดๆ ก่อนจะถูกส่งออก ตรงตามหลักการใน
[Part 32 Step 316.1](part-032-policy-based-routing.md) ที่อธิบายไว้ชัดเจนว่า Transit PBR
ผูกกับ Interface ขาเข้าเท่านั้น

### 636.4 Root Cause

วิศวกรใช้ **Transit PBR** (`ip policy route-map` ผูกกับ Interface) กับ Traffic ที่จัดอยู่ในหมวด
**Local Traffic** (Router สร้างขึ้นเอง ไม่เคยผ่าน Interface ขาเข้าใดๆ) — Transit PBR ตรวจสอบ
Packet เฉพาะตอนที่มันวิ่ง "เข้า" Interface เท่านั้น แต่ NTP Query ที่ CORE-SW1 สร้างขึ้นเอง
Originate จาก Process ภายใน Router โดยตรงและถูกส่งออกทันทีตาม Routing Table ปกติ (Skip ทุก
Interface Inbound Check ไปเลย) — ต้องใช้ **`ip local policy route-map`** (Global Command) ซึ่ง
เป็นกลไกที่ถูกออกแบบมาโดยเฉพาะสำหรับ Traffic ประเภทนี้เท่านั้น

### 636.5 แก้ไขและ Verify

```
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# no ip policy route-map PBR-NTP
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# ip local policy route-map PBR-NTP
```

```
CORE-SW1# show ip local policy
Local policy routing is enabled, using route map PBR-NTP
route-map PBR-NTP, permit, sequence 10
  Match clauses:
    ip address (access-lists): NTP-QUERY-TRAFFIC
  Set clauses:
    ip next-hop 10.10.254.1
  Policy routing matches: 12 packets, 960 bytes

! ทดสอบยิง NTP Query ทันที
CORE-SW1# ntp update-calendar
CORE-SW1# show ip local policy
  Policy routing matches: 13 packets, 1024 bytes
```

Hit Counter เพิ่มขึ้นทันทีหลังสั่ง NTP Sync — ยืนยันว่า Local PBR ทำงานถูกต้องแล้ว

> **วิธีจำกฎนี้แบบไม่ลืม**: ตั้งคำถามเสมอว่า **"Packet นี้ Router เคยเห็นมันวิ่ง 'เข้า' Interface
> ไหนมาก่อนไหม"** — ถ้าตอบว่า "ไม่เคย เพราะ Router เป็นคนสร้าง Packet นี้ขึ้นมาเอง" (Syslog, NTP,
> SNMP Trap, Ping ที่ Admin สั่งจาก CLI, SSH ที่ Router สร้างไป Login เครื่องอื่น) ให้ใช้
> `ip local policy route-map` (Global) เสมอ ไม่ใช่ `ip policy route-map` (Interface)

---

## Step 637 — Troubleshoot: Recursive Static Route หลุด Resolution กลายเป็น Routing Loop

### 637.1 บริบท: Static Host Route ที่พึ่ง Recursive Lookup

CORE-SW1 มี Static Host Route ไปยัง Management Server ตัวหนึ่ง (`172.16.99.50`, อยู่นอก
Campus ผ่าน MPLS Provider แยกที่เชื่อมกับ CORE-SW2) โดยใช้ Next-Hop IP เพียงอย่างเดียวตามรูปแบบ
"Next-hop เท่านั้น" ใน [Part 9 Step 83](part-009-static-routing.md):

```
CORE-SW1(config)# ip route 172.16.99.50 255.255.255.255 10.255.0.2
```

Next-Hop `10.255.0.2` คือ IP ของ CORE-SW2 บน Port-channel1 (`10.255.0.0/30` ตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) — Route นี้ทำงานได้เพราะ CORE-SW1 มี Connected
Route `10.255.0.0/30` อยู่แล้ว (Port-channel1) ทำให้ Recursive Lookup หา Next-Hop สำเร็จเสมอ
ตราบใดที่ Subnet นี้ยังอยู่ใน Table — และ CORE-SW1 มี **Default Route** ไปยัง WAN-EDGE-1 อยู่ด้วย
(`ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1` จาก Step 634)

### 637.2 อาการที่รายงานเข้ามาแบบ "ทันที" หลัง Change ที่ไม่เกี่ยวข้องกันเลย

ทีม Data Center ทำ Change Window แยกกันคนละทีม เพื่อ **Migrate Backbone Link ระหว่าง CORE-SW1
กับ CORE-SW2 จาก Port-channel1 (LACP) ไปเป็น Native Interface เดี่ยว** ชั่วคราวระหว่างเปลี่ยน
Hardware Module (งานที่ดูเหมือนไม่เกี่ยวข้องกับ Static Route ของ Management Server เลย) — ระหว่าง
Migration มีการสั่ง `no interface Port-channel1` บน CORE-SW1 ชั่วคราว (ตั้งใจจะสร้างใหม่ทันทีแต่
เกิดความล่าช้า) ทันทีที่ Interface หายไป **Server Admin รายงานว่าเข้าถึง Management Server
`172.16.99.50` ไม่ได้ทันที** และที่แปลกคือ `traceroute` จาก CORE-SW1 ไปยัง Server ตัวนี้แสดง
Pattern การวิ่งวนซ้ำ:

```
CORE-SW1# traceroute 172.16.99.50
Type escape sequence to abort.
Tracing the route to 172.16.99.50

  1 10.10.254.1 4 msec    <- WAN-EDGE-1
  2 10.255.10.254 4 msec  <- WAN-EDGE-1 ส่งกลับเข้ามาทาง Backbone อีกครั้ง (ผ่าน OSPF Route อื่น)
  3 10.10.254.1 4 msec    <- กลับไป WAN-EDGE-1 อีก
  4 10.255.10.254 4 msec
    ... (วนซ้ำจนกว่า TTL หมด)
```

Pattern นี้ตรงกับ **"Recursive Routing Loop"** ที่
[Part 9 Step 89](part-009-static-routing.md) เตือนไว้เป๊ะ — สองจุดสลับกันไปมาไม่มีที่สิ้นสุด

### 637.3 วินิจฉัย

**ขั้นที่ 1 — ตรวจว่า Static Route ยัง "อยู่" ใน RIB หรือหายไปแล้ว**

```
CORE-SW1# show ip route 172.16.99.50
Routing entry for 172.16.99.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.10.254.1, via GigabitEthernet1/0/1
      Route metric is 0, traffic share count is 1
```

**นี่คือจุดที่ทำให้งงที่สุด**: Route ยัง**อยู่**ใน RIB (ไม่ได้หายไปแบบ Step 637 คาดไว้ตอนแรกว่าน่า
จะเจอ "% Network not in table") แต่ **Next-Hop ที่แสดงกลับเป็น `10.10.254.1` (WAN-EDGE-1)** ซึ่ง
ไม่ใช่ Next-Hop ที่ Config ไว้เลย (`10.255.0.2`) — Static Route ที่ Config ไว้ชี้ไปที่
`10.255.0.2` แต่ตอนนี้แสดงผลว่า Resolve ไปทาง `10.10.254.1` ได้อย่างไร

**ขั้นที่ 2 — ตรวจว่า Next-Hop เดิม (`10.255.0.2`) ยัง Reachable โดยตรงไหม**

```
CORE-SW1# show ip route 10.255.0.2
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.10.254.1, via GigabitEthernet1/0/1
```

**นี่คือ Root Cause ตัวจริง**: เมื่อ `Port-channel1` ถูกลบไปชั่วคราว **Connected Route
`10.255.0.0/30` หายไปจาก RIB ด้วย** — CORE-SW1 จึงไม่มี Route ที่เจาะจงไปยัง `10.255.0.2` อีกเลย
แต่ตาม Longest Prefix Match (Step 631.2 ข้อ 3) **`0.0.0.0/0` (Default Route) ยัง Match
`10.255.0.2` ได้เสมอ** (เพราะ Default Route Match ทุก IP ที่ไม่มี Route เจาะจงกว่า) — IOS จึง
**Recursive-Resolve Next-Hop `10.255.0.2` ผ่าน Default Route โดยอัตโนมัติ** ทำให้ Static Route
ไปยัง `172.16.99.50` ที่ควรจะหายไปตามหลัก Fail-Safe ของ [Part 9 Step 83](part-009-static-routing.md)
กลับ**ไม่หาย** แต่ Resolve ผิดทางไปออก WAN-EDGE-1 (Internet Edge) แทน

**ขั้นที่ 3 — ตรวจสอบทำไม WAN-EDGE-1 ส่ง Packet วนกลับมา**

```
WAN-EDGE-1# show ip route 172.16.99.50
% Network not in table

WAN-EDGE-1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 203.0.113.1, via GigabitEthernet0/0/0

WAN-EDGE-1# show ip route | include 10.255
S    10.255.10.0/30 [1/0] via 10.10.254.2, GigabitEthernet0/0/1
```

WAN-EDGE-1 ไม่รู้จัก `172.16.99.50` เลย แต่มี Static Route ย่อยกลับไปยัง `10.255.10.0/30`
(Backbone Subnet บางส่วนที่ยังเรียนรู้ผ่าน OSPF เดิม) ทำให้ Packet ที่หลงเข้ามาถูกส่งกลับเข้า
Campus อีกรอบผ่านเส้นทางอื่น กลายเป็น Loop ระหว่าง CORE-SW1 ↔ WAN-EDGE-1 จนกว่า TTL จะหมด — ตรงกับ
Pattern ที่เห็นใน `traceroute` ขั้น 637.2 พอดี

### 637.4 Root Cause

Static Host Route ที่ใช้ **Next-Hop IP เพียงอย่างเดียว (Recursive)** ตาม
[Part 9 Step 83](part-009-static-routing.md) พึ่งพา Connected Route ของ Backbone Link
(`10.255.0.0/30` บน Port-channel1) เพื่อ Resolve Next-Hop เสมอ — เมื่อ Interface นั้นถูกลบออก
ชั่วคราวโดยทีมอื่นด้วยเหตุผลที่ไม่เกี่ยวข้องกับ Static Route นี้เลย (Hardware Migration)
**Recursive Lookup ไม่ได้ทำให้ Route หายไปแบบ Fail-Safe อย่างที่คาดหวัง เพราะ Default Route
(`0.0.0.0/0`) เข้ามา "รับหน้า" แทนโดยอัตโนมัติตามกฎ Longest Prefix Match** ทำให้ Static Route
ยัง Valid อยู่ใน RIB แต่ Resolve ผิดทางไปออก Internet Edge แทน Backbone ที่ตั้งใจไว้จริง เมื่อรวม
กับ WAN-EDGE-1 ที่ไม่มี Route กลับที่ถูกต้องสำหรับ `172.16.99.50` จึงเกิด Routing Loop ระหว่างสอง
Router — **นี่คือข้อจำกัดที่สำคัญของ Fail-Safe Behavior ใน Part 9 Step 83: มันจะ "หาย" ให้เห็น
ชัดเจนก็ต่อเมื่อไม่มี Route อื่นใดเลยมา Match Next-Hop นั้น ถ้ามี Default Route ดักอยู่ Fail-Safe
นี้จะไม่ทำงานอย่างที่คาดไว้**

### 637.5 แก้ไขและ Verify

**แก้เร่งด่วน**: เร่งทีม Data Center ให้สร้าง `Port-channel1` กลับคืนโดยเร็ว (แก้ที่ต้นเหตุจริง)

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.0.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

CORE-SW1# show ip route 172.16.99.50
Routing entry for 172.16.99.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.255.0.2, via Port-channel1
```

Route กลับมา Resolve ถูกทางทันทีที่ Connected Subnet คืนกลับมา

**แก้แบบยั่งยืน (ป้องกันปัญหาเดิมซ้ำในอนาคต)**: เปลี่ยน Static Route นี้เป็น **Fully Specified**
(ระบุทั้ง Exit-Interface และ Next-Hop IP พร้อมกัน ตามคำแนะนำของ
[Part 9 Step 83](part-009-static-routing.md)) เพื่อให้ Route นี้ผูกกับ Interface ที่เจาะจงจริง
แทนที่จะพึ่ง Recursive Lookup แบบเปิดกว้างที่เสี่ยงโดน Default Route แซงหน้า:

```
CORE-SW1(config)# no ip route 172.16.99.50 255.255.255.255 10.255.0.2
CORE-SW1(config)# ip route 172.16.99.50 255.255.255.255 Port-channel1 10.255.0.2
```

```
CORE-SW1# show ip route 172.16.99.50
Routing entry for 172.16.99.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.255.0.2, via Port-channel1
```

> **บทเรียนสำคัญที่สุดของ Step นี้**: Static Route แบบ Next-Hop-Only ไม่ได้ "หายไปอย่างปลอดภัย
> เสมอ" เมื่อ Intermediate Route หาย — **ถ้ามี Default Route อยู่ในตาราง มันจะ "รับหน้า" แทนเสมอ
> ตามกฎ Longest Prefix Match** และอาจทำให้ Static Route Resolve ผิดทางไปโดยสิ้นเชิงจนเกิด Routing
> Loop ได้ — Static Route ที่สำคัญ (โดยเฉพาะที่พึ่ง Backbone Link ภายในองค์กร) ควรใช้ **Fully
> Specified** เสมอเพื่อตัดความเสี่ยงนี้ตั้งแต่ต้น ไม่ใช่แค่เพื่อประสิทธิภาพอย่างที่ Part 9 เน้นไว้
> ตอนแรกเท่านั้น

---

## Step 638 — Troubleshoot: PBR Asymmetric Return Path ทำลาย Stateful Firewall/NAT

### 638.1 บริบท: FW-1 (Transparent) อยู่ระหว่าง WAN Edge กับ Core

ตาม [Part 43 Step 421](part-043-security-architecture-firepower-ise.md) **FW-1 (FTD, Transparent
Mode)** วางอยู่ตรงกลางระหว่าง `WAN-EDGE-1/WAN-EDGE-2` กับ `CORE-SW1/CORE-SW2` — ทุก Traffic ที่
ข้ามจาก Campus ไปยัง Internet (และกลับ) ต้องผ่าน FW-1 เสมอ และ FW-1 เป็น **Stateful Firewall**
เต็มรูปแบบ: มันจดจำ Flow ขาไป (SYN) และรอ Traffic ขากลับ (SYN-ACK) ที่ต้อง**สอดคล้องกับ State ที่
บันทึกไว้** — ถ้า Return Traffic มาในลักษณะที่ FW-1 มองว่า "ไม่มี State ที่ตรงกัน" (เช่น เห็น
SYN-ACK แบบไม่มี SYN นำมาก่อนบน Session Table) FW-1 จะ Drop ทันทีตาม Default Behavior ของ NGFW

### 638.2 อาการที่รายงานเข้ามา

หลังเปิด PBR-VOICE (Step 632-636 ทั้งหมด) และ PBR-SALES-BACKUP (Step 635) พร้อมกันบน CORE-SW1
User รายงานว่า **SIP Call Setup ไปยัง Partner Site ล้มเหลวเป็นบางครั้ง** (Ring แล้วไม่มีเสียง หรือ
Call Setup Timeout) โดยเฉพาะ Call ที่เริ่มจากฝั่ง Partner โทรเข้ามา (Inbound Call) ในขณะที่ Call
ที่ User ในองค์กรโทรออกไปก่อน (Outbound) ทำงานได้ปกติดี

### 638.3 วินิจฉัย — เทียบ Path ทั้งสองทิศทาง

**ขั้นที่ 1 — Trace ทิศทางขาไป (Outbound) จาก VOICE VLAN**

```
CORE-SW1# show route-map PBR-VOICE
route-map PBR-VOICE, permit, sequence 10
  Set clauses:
    ip next-hop verify-availability 10.10.254.1 10 track 1  [up]
  Policy routing matches: 88104 packets, ...
```

PBR บังคับ Traffic ขาไปของ VOICE ให้วิ่งออกทาง **WAN-EDGE-1 เสมอ** ตาม Design เดิมของ
[Part 32](part-032-policy-based-routing.md) — ไม่มีอะไรผิดปกติในทิศทางนี้

**ขั้นที่ 2 — Trace ทิศทางขากลับ (Return/Inbound Call จาก Partner)**

Return Traffic (หรือ Inbound Call จาก Partner) เข้ามาจาก ISP ปลายทางไหนก็ได้ที่ Routing แบบ
Public Internet เลือกให้ — **ไม่มี PBR หรือ Policy อะไรควบคุมทิศทางนี้เลย** เพราะ PBR ที่ Apply
ไว้ทั้งหมดใน Part นี้เป็น PBR ที่ตรวจ Traffic **ขาเข้า CORE-SW1 จากฝั่ง DIST-SW เท่านั้น**
(Te1/0/1, Te1/0/2) — Traffic ที่เข้ามาจากฝั่ง WAN (ผ่าน Gi1/0/1 หรือ WAN-EDGE-2) ไม่เคยถูก PBR
ใดๆ แตะเลยตั้งแต่ต้น มันวิ่งตาม **OSPF/Routing Table ปกติเท่านั้น**:

```
CORE-SW2# show ip route 10.10.20.0
Routing entry for 10.10.20.0/24
  Known via "ospf 1", distance 110, metric 3
  Routing Descriptor Blocks:
  * 10.255.0.1, from 1.1.1.1, via Port-channel1
```

Return Traffic ที่เข้าทาง **WAN-EDGE-2 → CORE-SW2** จะถูกส่งต่อไปยัง CORE-SW1 ผ่าน OSPF ตามปกติ
(Metric ต่ำสุด) — **สลับข้าง FW-1 ไปมาได้ตามที่ ISP ปลายทางเลือก Path เข้ามา** ไม่ใช่ Path
เดียวกันกับที่ PBR บังคับไว้ตอนขาไปเลย

**ขั้นที่ 3 — ตรวจ FW-1 ว่าเห็น Traffic สองทิศทางผ่านจุดเดิมหรือคนละจุด**

```
FW-1# show conn address 10.10.20.55
No connections found matching 10.10.20.55 for inbound SIP session id XXXX
```

FW-1 ไม่พบ Session ที่ตรงกันสำหรับ Inbound Call — ยืนยันว่า **Return Traffic ไม่ได้ผ่าน Session
Table เดิมที่ Outbound สร้างไว้** เพราะ Outbound (ขาไป, ผ่าน PBR บังคับ) กับ Inbound Call ครั้งใหม่
(ที่ไม่ผ่าน PBR ใดๆ) เกิดขึ้นจาก Physical Interface หรือ Bridge Group คนละฝั่งของ FW-1 ทำให้ FW-1
มองว่าเป็นการเชื่อมต่อใหม่ที่ไม่มี State รองรับ (โดยเฉพาะ SIP ที่ทำ Call Setup ผ่าน Signaling Flow
แยกจาก Media Flow — Asymmetric Path ที่กระทบแค่บาง Flow ของ Call ทำให้ปัญหาดูเหมือน "บางครั้ง
ทำงาน บางครั้งไม่")

### 638.4 Root Cause

PBR ใน Part นี้ (และ Part 32) ถูกออกแบบมาให้ควบคุม**เฉพาะทิศทางขาออกจาก Campus (Outbound, Traffic
ที่เข้า CORE-SW1 จากฝั่ง DIST-SW)** เท่านั้น — **ไม่มีกลไกใดบังคับทิศทางขากลับ (Return/Inbound)
ให้วิ่งผ่าน Path เดียวกัน** เพราะ PBR ทำงานทิศทาง Inbound ต่อ Interface เท่านั้น (Step 631.2)
ทำให้เกิด **Asymmetric Path** ระหว่าง Outbound กับ Inbound ของ Flow เดียวกัน เมื่อ Path ทั้งสอง
ผ่านจุด Stateful (FW-1) คนละฝั่งกัน FW-1 จะมองว่าไม่มี State ที่ตรงกันและ Drop Traffic — ปัญหานี้
เป็นแนวคิดเดียวกันกับ **Asymmetric NAT** ที่ [Part 63 Step 625](part-063-advanced-nat-scenarios.md)
อธิบายไว้ แต่คราวนี้ตัวการคือ **PBR ที่ควบคุมแค่ครึ่งทาง** ไม่ใช่ ECMP ของ Routing Table ปกติ

### 638.5 แก้ไข — บังคับ Symmetric Path ทั้งสองทิศทาง

ต้องเพิ่ม PBR (หรือ Path Control ที่เทียบเท่า) ให้ครอบคลุม **ทิศทางขากลับด้วย** โดย Apply ที่
Interface ขาเข้าฝั่ง WAN บน CORE-SW2 เพื่อบังคับ Traffic ที่มุ่งหน้าเข้า VLAN 20 ให้ย้อนกลับผ่าน
CORE-SW1/WAN-EDGE-1 เสมอเมื่อ WAN-EDGE-1 Healthy (สมมาตรกับทิศทางขาไป):

```
CORE-SW2(config)# ip access-list extended RETURN-TO-VOICE
CORE-SW2(config-ext-nacl)# permit ip any 10.10.20.0 0.0.0.255
CORE-SW2(config-ext-nacl)# exit
!
CORE-SW2(config)# route-map PBR-VOICE-RETURN permit 10
CORE-SW2(config-route-map)# match ip address RETURN-TO-VOICE
CORE-SW2(config-route-map)# set ip next-hop verify-availability 10.255.0.1 10 track 1
CORE-SW2(config-route-map)# exit
!
! Apply ที่ Interface ขาเข้าจาก WAN-EDGE-2 (ทิศทาง Inbound เข้า CORE-SW2)
CORE-SW2(config)# interface GigabitEthernet1/0/1
CORE-SW2(config-if)# description ** Uplink to WAN-EDGE-2 **
CORE-SW2(config-if)# ip policy route-map PBR-VOICE-RETURN
CORE-SW2(config-if)# exit
```

`set ip next-hop verify-availability 10.255.0.1 10 track 1` บังคับให้ Traffic ที่มุ่งหน้าเข้า
VLAN 20 ที่หลงเข้ามาทาง WAN-EDGE-2/CORE-SW2 **ถูกส่งกลับไปยัง CORE-SW1 ผ่าน Port-channel1 เสมอ**
เมื่อ Track Object (Track เดียวกับที่ตรวจ WAN-EDGE-1) ยัง Up — ทำให้ FW-1 เห็น Outbound และ Inbound
ของ Flow เดียวกันผ่านจุดเดิมเสมอ (Symmetric) แก้ปัญหา Stateful Drop ได้

**Verify**:

```
FW-1# show conn address 10.10.20.55
5 in use, 12 most used
TCP CORE-SW1_side 10.10.20.55:5060 WAN-EDGE-1_side 198.51.100.30:5060, idle 0:00:02, bytes 4820, flags UIO
```

Session ปรากฏใน Connection Table ของ FW-1 พร้อม Flag `UIO` (Up, Inbound, Outbound seen) ยืนยันว่า
ทั้ง Outbound และ Inbound ของ Flow เดียวกันผ่านจุดเดิมของ FW-1 แล้ว SIP Call Setup กลับมาสำเร็จ
ปกติ

> **บทเรียนของ Step นี้**: ทุกครั้งที่ใช้ PBR บังคับ Path ขาไปของ Traffic ที่ต้องผ่าน Stateful
> Device (Firewall, NAT) ต้องถามเสมอว่า **"ทิศทางขากลับของ Flow นี้มีอะไรบังคับให้เดินสวนทาง
> เดียวกันไหม"** ถ้าคำตอบคือ "ไม่มี ปล่อยตาม Routing Table ปกติ" ต้องออกแบบ Path Control ให้
> ครอบคลุมทั้งสองทิศทางเสมอ — ตรวจสอบด้วยการ `traceroute` ทั้งขาไปและขากลับแล้วเทียบ Hop ว่าผ่าน
> จุด Stateful เดียวกันหรือไม่ เป็นขั้นตอน Verification มาตรฐานทุกครั้งที่เพิ่ม PBR ใกล้ Firewall/NAT

---

## Step 639 — Troubleshoot: Route Summarization สร้าง Black Hole Route โดยไม่ตั้งใจ

### 639.1 บริบท: Area Range Summary ที่ Config ไว้ตั้งแต่ Part 12

CORE-SW1 (ABR ของ Area 1) Summarize VLAN 10/20/99 เข้า Area 0 ด้วย Range ที่กว้างมาก
(`10.10.0.0/16`) ตาม [Part 12 Step 115.2](part-012-ospfv2-advanced.md) — ตอนนั้นเอกสารเตือนไว้แล้ว
ว่า IP Plan ของหลักสูตรนี้ "ไม่ได้ถูกออกแบบมาเพื่อ Summarization ตั้งแต่แรก" ทำให้ต้องใช้ Range
กว้างกว่าที่ Subnet จริงต้องการมาก (`/16` แทน `/24` ทีละวง):

```
CORE-SW1(config-router)# area 1 range 10.10.0.0 255.255.0.0
```

ล่าสุด ทีม Security เสนอ Hardening เพิ่มเติมตามคำแนะนำมาตรฐานของ Multi-Area OSPF (ป้องกัน Routing
Loop ระหว่าง Area เมื่อ Component Subnet Flap) โดยเปิด **`discard-route`** อย่างชัดเจน:

```
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# discard-route internal
```

### 639.2 อาการที่รายงานเข้ามา

ทีม Server ตั้งค่า Staging Server ตัวใหม่โดยพิมพ์ IP ผิด — ตั้งใจจะใช้ `10.10.30.12` (VLAN 30
SERVERS ตาม [00-ip-address-plan.md](00-ip-address-plan.md)) แต่พิมพ์เป็น **`10.10.18.12`** โดย
ไม่มีใครสังเกต (ไม่มี VLAN 18 อยู่จริงในหลักสูตรนี้เลย) — เมื่อพยายาม Ping จาก PC1 ไปยัง
`10.10.18.12` **Ping Timeout เงียบสนิท ไม่มี "Destination unreachable" ใดๆ กลับมาเลย** ทั้งที่ปกติ
ถ้า IP ไม่มีอยู่จริง ควรได้ ICMP Unreachable กลับมาอย่างรวดเร็วจาก Router ตัวที่ไม่มี Route

### 639.3 วินิจฉัย

**ขั้นที่ 1 — Traceroute เพื่อดูว่า Packet หายไปที่ Hop ไหน**

```
PC1> traceroute 10.10.18.12

Tracing route to 10.10.18.12 over a maximum of 30 hops:
  1   1 ms   1 ms   1 ms   10.10.10.1        <- CORE-SW1 (SVI Gateway VLAN10)
  2   *      *      *      Request timed out
  3   *      *      *      Request timed out
  ... (ไม่มี Hop ไหนตอบ * ทั้งหมดจนครบ 30 Hop)

Trace complete.
```

**Pattern `* * *` ทุก Hop โดยไม่มี ICMP Unreachable แม้แต่ครั้งเดียว** คือลายเซ็นคลาสสิกของ
**Black Hole Route** — ต่างจาก "ไม่มี Route เลยจริงๆ" ที่ปกติจะเห็น Router ตัวสุดท้ายที่รู้จัก
ปลายทางตอบ `!H` หรือ Message "Destination unreachable" กลับมาทันทีที่ Hop 1 หรือ 2

**ขั้นที่ 2 — ตรวจ Routing Table บน CORE-SW1 (จุดที่ Packet ไปถึง Hop สุดท้ายที่ตอบ)**

```
CORE-SW1# show ip route 10.10.18.12
Routing entry for 10.10.0.0/16, supernet
  Known via "ospf 1", distance 110, metric 0
  Routing Descriptor Blocks:
  * directly connected, via Null0
      Route metric is 0, traffic share count is 1
```

**พบตัวการทันที**: `10.10.18.12` Match กับ Route **`10.10.0.0/16` ที่ชี้ไปยัง `Null0`** — นี่คือ
Discard Route ที่ OSPF สร้างขึ้นอัตโนมัติจากคำสั่ง `area 1 range` ที่ Config ไว้ตั้งแต่ Part 12
เพื่อป้องกัน Routing Loop (มาตรฐานของ OSPF Area Summarization: ถ้ามี Component Subnet ใดๆ ภายใน
Range Active อยู่ ABR จะสร้าง Route แบบ Discard ครอบคลุมทั้ง Range ไว้เสมอ เพื่อไม่ให้ Packet ที่
Match Range แต่ไม่มี Subnet จริงรองรับ ถูกส่งย้อนกลับเข้า Backbone จนเกิด Loop)

**ขั้นที่ 3 — ยืนยันด้วย `show ip ospf summary-address` (เทียบกับ Part 12 ที่เคยเป็น "Not
configured")**

```
CORE-SW1# show ip ospf summary-address
OSPF Process 1, Summary-address
  10.10.0.0/16    Metric -1, Type: Summary, Mask: 255.255.0.0
                  Advertised, Ranges Configured 1, Discard Route Configured
```

`Discard Route Configured` — ต่างจาก [Part 12 Step 115.2](part-012-ospfv2-advanced.md) ที่แสดง
"Discard Route Not configured" ในตอนนั้น ยืนยันว่า Hardening ล่าสุด (`discard-route internal`)
เป็นสาเหตุที่ทำให้ Behavior เปลี่ยนไป

**ขั้นที่ 4 — ตรวจ Interface Null0 เพื่อยืนยันว่า Traffic จริงถูกทิ้งที่นี่**

```
CORE-SW1# show interfaces null0
Null0 is up, line protocol is up
  ...
  5 minutes input rate 0 bits/sec, 0 packets/sec
  5 minutes output rate 0 bits/sec, 0 packets/sec

CORE-SW1# show ip traffic | include discard
  1284 discarded because of no route
```

Counter `discarded because of no route` เพิ่มขึ้นตรงกับจำนวน Ping/Traceroute ที่ทดสอบไป — ยืนยัน
ครบทุกจุดว่า CORE-SW1 **จงใจทิ้ง Packet นี้ตาม Design ของ OSPF Summarization** ไม่ใช่ Bug หรือ
Hardware Failure

### 639.4 Root Cause

**Range การ Summarize (`10.10.0.0/16`) กว้างกว่า Address Space ที่ถูกใช้งานจริงมาก** (ครอบคลุม
Subnet ที่ไม่มีอยู่จริงกว่า 250 /24 จากทั้งหมด 256 ที่เป็นไปได้ใน `/16` นี้) เมื่อรวมกับการเปิด
`discard-route internal` (Hardening ที่ตัวมันเองถูกต้องตามหลักการป้องกัน Routing Loop) ผลข้างเคียง
คือ **ที่อยู่ IP ใดๆ ก็ตามภายใน `/16` นี้ที่ไม่ตรงกับ Subnet ที่ประกาศจริง จะถูก Silent Discard
ทันทีที่ CORE-SW1 แทนที่จะได้รับ ICMP Destination Unreachable ตามปกติ** — Typo ของทีม Server
(`10.10.18.12` แทน `10.10.30.12`) ที่ควรจะถูกจับได้ทันทีจาก Error Message ที่ชัดเจน กลับกลายเป็น
"เงียบหาย" ทำให้วินิจฉัยยากขึ้นมาก เพราะอาการ (Ping Timeout เงียบ) เหมือนกับ Network Down จริงๆ
ทุกประการ

### 639.5 แก้ไข

**แก้เร่งด่วน (Operational)**: แก้ไข IP ของ Staging Server ให้ตรงกับ IP Plan จริง

```
! บน Staging Server: เปลี่ยนจาก 10.10.18.12 เป็น 10.10.30.12 ตาม 00-ip-address-plan.md
```

**แก้เชิง Design (ป้องกันปัญหาเดิมซ้ำ)**: ไม่ควรถอด `discard-route` ออก (เพราะยังจำเป็นต่อการ
ป้องกัน Routing Loop จริงตามที่ Part 12 อธิบาย) แต่ควร **เพิ่มการเฝ้าระวัง (Monitoring) เฉพาะสำหรับ
Null0 Discard Counter** เพื่อให้ทีมรู้ทันทีที่มี Traffic ตกไปโดน Black Hole จำนวนผิดปกติ (ซึ่งมัก
เป็นสัญญาณของ Typo หรือ IP Plan ไม่ตรงกับที่ Deploy จริง ไม่ใช่ Network Failure):

```
CORE-SW1(config)# snmp-server enable traps entity
CORE-SW1(config)# interface null0
CORE-SW1(config-if)# description ** OSPF Area 1 Discard Route (10.10.0.0/16) - Monitor Counter **
```

และเพิ่มขั้นตอนตรวจสอบ `show ip traffic | include discarded because of no route` เข้าไปใน
Checklist Health Check รายวันของทีม NOC เพื่อจับ Pattern ผิดปกตินี้ให้เร็วกว่าการรอ User รายงาน
ปัญหาเข้ามา

**Verify**:

```
PC1> ping 10.10.30.12
Reply from 10.10.30.12: bytes=32 time=1ms TTL=254
```

> **บทเรียนสำคัญที่สุดของ Step นี้**: Route Summarization ที่กว้างเกินความจำเป็น (ปัญหาที่
> [Part 9 Step 87](part-009-static-routing.md) และ [Part 7 Step 69](part-007-ipv4-addressing-subnetting.md)
> เตือนไว้แล้วในเชิงทฤษฎี) เมื่อมาผสมกับ `discard-route` (Best Practice ที่ถูกต้องของมันเอง) จะ
> เปลี่ยน "IP Typo ที่ควรถูกจับได้ง่ายจาก Error Message" ให้กลายเป็น "Black Hole ที่วินิจฉัยยาก
> เหมือน Network Down จริง" — บทเรียนนี้ไม่ได้แปลว่าห้ามใช้ `area range` หรือ `discard-route` แต่
> ต้องรู้ว่ามันมี Trade-off นี้อยู่เสมอ และต้องมี Monitoring คู่กันไปด้วยเพื่อจับปัญหานี้ให้เร็ว

---

## Step 640 — Lab เต็มรูปแบบ: 2 ปัญหาซ้อนกันหลัง Change Window ที่พลาด

### 640.1 บริบท

ในหนึ่ง Change Window เดียวกัน วิศวกรจูเนียร์คนหนึ่งได้รับมอบหมายให้ทำ 2 งานพร้อมกันบน CORE-SW1:

1. เพิ่ม SIP Trunk ใหม่เข้า ACL `VOICE-TO-INTERNET` (ต้องแก้ Wildcard Mask ให้ครอบคลุม Subnet
   VOICE ใหม่ที่ขยายเพิ่ม)
2. "ทำความสะอาด" Comment ของ Static Default Route คู่ Primary/Floating (ลบแล้วพิมพ์ใหม่ให้อ่าน
   ง่ายขึ้น)

ทั้งสองงานดู "เล็กและไม่เกี่ยวกัน" จึงถูกรวมไว้ใน Change เดียวและ Deploy พร้อมกันตอนดึก — เช้าวันรุ่ง
ขึ้น Helpdesk ได้รับ Ticket จำนวนมากพร้อมกัน

### 640.2 อาการรวมที่รายงานเข้ามา

- User หลายคนรายงาน **Voice Call มี Jitter สูง** (คล้าย Step 632) — แต่เฉพาะบางเครื่อง
- ทีม Monitoring รายงาน **Internet Latency สูงขึ้นทั้ง Campus แบบสม่ำเสมอ** (คล้าย Step 634) —
  ไม่ใช่แค่ Voice
- ทั้งสองอาการเกิดขึ้น **พร้อมกันหลัง Change Window เดียวกัน** ทำให้ทีม On-Call ช่วงแรกสงสัยว่าเป็น
  ปัญหาเดียวกัน (เข้าใจผิดว่า "Internet ช้าเลยทำให้ Voice แย่ไปด้วย") ซึ่งทำให้เสียเวลาไล่ผิดทาง
  อยู่พักหนึ่งก่อนจะแยกวินิจฉัยเป็น 2 ปัญหาคนละเรื่องกัน

### 640.3 Systematic Diagnosis — ใช้ Checklist จาก Step 631.3 อย่างเคร่งครัด

**ขั้นที่ 1 (Checklist ข้อ 1): ตรวจ PBR ก่อนเสมอ**

```
CORE-SW1# show ip policy
Interface                Route map
Te1/0/1                  PBR-VOICE
Te1/0/2                  PBR-VOICE
```

PBR ยัง Apply ถูก Interface ตามเดิม — ไปตรวจ Hit Counter ต่อ

**ขั้นที่ 2: ดู Route-map และ ACL Hit Counter แยกทีละบรรทัด**

```
CORE-SW1# show access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (4102 matches)
    15 permit ip 10.10.21.0 0.0.0.255 any (0 matches)
    20 permit ip 10.10.20.0 0.0.0.255 any (52108 matches)
```

พบ **บรรทัดที่ 15 เป็นบรรทัดใหม่ที่เพิ่มเข้ามาสำหรับ SIP Trunk** (`10.10.21.0/24` — Subnet ใหม่ที่
VOICE ขยายมา) แต่มี **0 Matches ตลอด** ทั้งที่ Traffic ของ SIP Trunk ใหม่กำลังวิ่งอยู่จริง — เป็น
สัญญาณเดียวกับ Step 632 (ACL ผิดจน Fallthrough)

**ขั้นที่ 3: ตรวจ Running-Config ของ ACL เพื่อหา Typo**

```
CORE-SW1# show running-config | section VOICE-TO-INTERNET
ip access-list extended VOICE-TO-INTERNET
 deny   ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255
 permit ip 10.10.21.0 0.0.0.15 any
 permit ip 10.10.20.0 0.0.0.255 any
```

พบ **ปัญหาที่ 1**: Wildcard Mask ของบรรทัดใหม่คือ `0.0.0.15` (ครอบคลุมแค่ 16 Address แรกของ
`10.10.21.0/24`) ทั้งที่ต้องการครอบคลุมทั้ง `/24` (`0.0.0.255`) — SIP Trunk Server ส่วนใหญ่มี IP
อยู่นอกช่วง `.0-.15` จึง Fallthrough ไปใช้ OSPF ปกติ เกิด Jitter เหมือน Step 632 เป๊ะ

**ขั้นที่ 4 (Checklist ข้อ 3-4): เมื่อจบเรื่อง PBR แล้ว ไปตรวจ Routing Table แยกส่วน**

เนื่องจากอาการที่ 2 (Internet ช้าทั้ง Campus) **ไม่เกี่ยวกับ VOICE VLAN โดยเฉพาะ** (กระทบทุก VLAN)
ทำให้ตัดความเป็นไปได้ที่จะเป็นเรื่อง PBR-VOICE ได้ทันที (PBR-VOICE Match แค่ VLAN 20 เท่านั้น) — ต้อง
ตรวจ Default Route แยกเป็นอีกปัญหาหนึ่ง:

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.255.0.2, via Port-channel1
```

**ปัญหาที่ 2 ปรากฏชัด**: Next-Hop ที่ Active คือ `10.255.0.2` (ผ่าน CORE-SW2/WAN-EDGE-2 — ควรเป็น
แค่ Backup) ไม่ใช่ `10.10.254.1` (WAN-EDGE-1 — Primary) — สัญญาณเดียวกับ Step 634 เป๊ะ

**ขั้นที่ 5: ยืนยันด้วย Running-Config**

```
CORE-SW1# show running-config | include ip route 0.0.0.0
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1 5
ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2
```

ยืนยัน AD สลับกันแบบเดียวกับ Step 634 ทุกประการ — "ทำความสะอาด Comment" ที่ตั้งใจไว้ ทำให้เกิดการ
พิมพ์เลข AD สลับบรรทัดโดยไม่ได้ตั้งใจอีกครั้ง

### 640.4 สรุปว่าเป็น 2 ปัญหาที่ไม่เกี่ยวข้องกันเลย เกิดพร้อมกันโดยบังเอิญจาก Change เดียวกัน

| ปัญหา | ที่มา | กลไกที่เสีย | อาการ | ตรงกับ Step |
|---|---|---|---|---|
| 1. ACL Wildcard Mask ผิด (SIP Trunk ใหม่) | เพิ่ม ACL Permit ใหม่ผิด Wildcard | PBR (Transit) — Fallthrough | Jitter เฉพาะ SIP Trunk Server บางตัว | Step 632 |
| 2. Default Route AD สลับกัน | พิมพ์ Static Route ใหม่สลับเลข AD | Static Route AD/Floating | Internet ช้าทั้ง Campus สม่ำเสมอ | Step 634 |

### 640.5 แก้ไขทั้งคู่พร้อมกัน

```
! ---------- แก้ปัญหาที่ 1: ACL Wildcard Mask ----------
CORE-SW1(config)# ip access-list extended VOICE-TO-INTERNET
CORE-SW1(config-ext-nacl)# no permit ip 10.10.21.0 0.0.0.15 any
CORE-SW1(config-ext-nacl)# 15 permit ip 10.10.21.0 0.0.0.255 any
CORE-SW1(config-ext-nacl)# exit

! ---------- แก้ปัญหาที่ 2: Default Route AD สลับกัน ----------
CORE-SW1(config)# no ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1 5
CORE-SW1(config)# no ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 Port-channel1 10.255.0.2 5
```

### 640.6 Verify ครบทั้งสองปัญหา

```
! ปัญหาที่ 1 — ACL
CORE-SW1# show access-lists VOICE-TO-INTERNET
Extended IP access list VOICE-TO-INTERNET
    10 deny ip 10.10.20.0 0.0.0.255 10.10.0.0 0.0.255.255 (4102 matches)
    15 permit ip 10.10.21.0 0.0.0.255 any (620 matches)
    20 permit ip 10.10.20.0 0.0.0.255 any (52890 matches)

! ปัญหาที่ 2 — Default Route
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  Routing Descriptor Blocks:
  * 10.10.254.1, via GigabitEthernet1/0/1

! ทดสอบ Failover ของ Default Route ยังทำงานถูกต้องตาม Design
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# shutdown
CORE-SW1(config-if)# end
CORE-SW1# show ip route 0.0.0.0
  Known via "static", distance 5, metric 0
  * 10.255.0.2, via Port-channel1
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# no shutdown
```

```
! ยืนยันจาก User ปลายทาง
SIP-Trunk-Server> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252

PC1> ping 8.8.8.8
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252
```

Latency กลับสู่ระดับปกติทั้ง Voice และ Internet โดยรวม ยืนยันว่าทั้งสองปัญหาที่ไม่เกี่ยวข้องกันเลย
ถูกแก้ไขครบถ้วนแล้ว

### 640.7 สรุปผล Lab

| ประเด็น | รายละเอียด |
|---|---|
| จำนวนปัญหาที่เกิดพร้อมกัน | 2 ปัญหา (คนละกลไก คนละสาเหตุ เกิดพร้อมกันจาก Change Window เดียวกัน) |
| กับดักในการวินิจฉัย | ทีม On-Call เข้าใจผิดว่าเป็นปัญหาเดียวกันในตอนแรก เพราะเกิดพร้อมกันและ Change มาด้วยกัน |
| เครื่องมือที่แยกปัญหาได้ | Checklist Step 631.3: ตรวจ PBR ก่อนเสมอ (ข้อ 1-2) แล้วจึงตรวจ Routing Table แยกส่วน (ข้อ 3-4) — อาการที่กระทบ "เฉพาะ VLAN 20" ชี้ไปที่ PBR ส่วนอาการที่กระทบ "ทุก VLAN" ชี้ไปที่ Path Control ระดับ Routing Table |
| บทเรียน Change Management | Change ที่ดู "เล็กและไม่เกี่ยวกัน" 2 งาน ไม่ควร Deploy รวมกันในหน้าต่างเดียวโดยไม่มี Verification แยกทีละงานก่อน Merge |

---

## แบบฝึกหัดทวนความเข้าใจ Part 64

1. ทำไมการ Troubleshoot Path Control ที่มี PBR เกี่ยวข้อง ต้องตรวจ `show ip policy` และ
   `show route-map` **ก่อน** จะเริ่มสงสัย Routing Protocol หรือ Static Route AD เสมอ?
2. ในกับดัก "PBR Permit Fallthrough" (Step 632) เพราะเหตุใด ACL ที่เขียน Wildcard Mask ผิดจน
   ไม่ Match Traffic ตามที่ตั้งใจ จึงไม่ทำให้ IOS แสดง Error หรือ Log เตือนใดๆ เลย?
3. อธิบายว่าทำไม IP SLA Probe Target ที่เลือกใช้ Host ภายนอกที่ทีมอื่นดูแล (Step 633) มีความเสี่ยง
   มากกว่าการ Probe ไปที่ Interface ของ Router เพื่อนบ้านโดยตรง และวิธีแก้ที่แนะนำ (Track List)
   ทำงานอย่างไร?
4. ในกรณี Floating Static Route ที่ AD สลับกัน (Step 634) เพราะเหตุใดการเปลี่ยนแปลงนี้จึงไม่ทำให้
   `ip route` Command Error หรือแม้แต่ Warning และต้องใช้วิธีใดในการตรวจจับปัญหานี้?
5. อธิบายว่าทำไม Static Route แบบ Next-Hop-Only (Recursive) ที่ Intermediate Route หายไป อาจไม่
   หายไปจาก RIB อย่างที่คาดหวัง (Step 637) และวิธีป้องกันที่แนะนำคืออะไร?

**เฉลย:**

1. เพราะ **PBR ถูกตรวจสอบก่อน Routing Table เสมอ**สำหรับ Packet ที่เข้า Interface ที่มี PBR Apply
   อยู่ (Step 631.2) — ถ้าปัญหาจริงอยู่ที่ PBR (เช่น ACL ไม่ Match หรือ Apply ผิด Interface)
   Routing Table จะทำงาน**ถูกต้องสมบูรณ์**และไม่แสดงอะไรผิดปกติเลย เพราะปัญหาไม่ได้อยู่ที่ Layer
   นั้น การไล่ตรวจ Routing Protocol ก่อนจะเสียเวลาโดยไม่พบสาเหตุจริง
2. เพราะ Route-map ที่ใช้กับ PBR เป็น `permit` โดยธรรมชาติ และถูกออกแบบให้เป็น **Fail-Open**:
   ถ้า ACL ไม่ Match เลยสักบรรทัด (ไม่ว่าเพราะ Wildcard Mask ผิดหรือเหตุผลอื่น) Clause นั้นจะถือว่า
   "ไม่ Match" และ Packet จะ Fallthrough ไปใช้ Routing Table ปกติทันทีโดยไม่มี Error หรือ Log ใดๆ
   (ป้องกันไม่ให้ Config PBR ผิดทำให้ Traffic ทั้งหมด Drop) ต้องตรวจ Hit Counter ของทั้ง Route-map
   และ ACL เองเพื่อจับปัญหานี้
3. เพราะสุขภาพของ Track Object จะขึ้นอยู่กับความมีชีวิตของ Host นั้นโดยตรง ซึ่งอยู่นอกการควบคุมของ
   ทีม Network เอง — ถ้า Host นั้นถูกเปลี่ยนแปลง/ปิดด้วยเหตุผลที่ไม่เกี่ยวกับ Path เลย Track จะ
   รายงานผลผิดพลาดทำให้ PBR ตัดสินใจผิดตามไปด้วย Track List (`boolean and`/`or`) แก้ปัญหาด้วยการ
   รวมหลาย Track Object เข้าด้วยกัน ทำให้ Track โดยรวมไม่ขึ้นกับความล้มเหลวของ Probe เพียงจุดเดียว
4. เพราะ Administrative Distance เป็นแค่ตัวเลขที่ IOS ยอมรับตาม Syntax โดยไม่มีการ Validate
   "เจตนา Design" ใดๆ — ค่า AD ที่สลับกัน (Primary ได้ AD สูง, Backup ได้ AD ต่ำ) ยัง Valid ตาม
   Syntax ทุกประการ ต้องตรวจจับด้วยการเทียบ `show ip route <prefix>` (ดู Next-Hop ที่ Active จริง)
   กับ `show running-config` (ดู AD ที่ Config ไว้) และเทียบกับ Design Document ว่าตรงกับเจตนา
   เดิมหรือไม่ พร้อมทดสอบ Failover จริงเพื่อยืนยัน
5. เพราะ Cisco IOS ทำ Recursive Lookup ตามกฎ **Longest Prefix Match** เหมือน Destination Lookup
   ทั่วไป — ถ้า Route ที่เจาะจงไปยัง Next-Hop นั้นหายไป (เช่น Connected Subnet ถูกลบ) แต่ยังมี
   **Default Route (`0.0.0.0/0`)** อยู่ในตาราง Default Route จะ Match ทุก IP รวมถึง Next-Hop นั้น
   ด้วยเสมอ ทำให้ Static Route ยัง "Resolve ได้" (ไม่หายไปจาก RIB) แต่ Resolve ผิดทางไปตาม Default
   Route แทน อาจนำไปสู่ Routing Loop ได้ — วิธีป้องกันคือใช้ **Fully Specified Route** (ระบุทั้ง
   Exit-Interface และ Next-Hop พร้อมกัน) เพื่อผูก Route กับ Interface ที่เจาะจงจริง ไม่พึ่ง
   Recursive Lookup แบบเปิดกว้าง

---

## สรุป Part 64

Part นี้เป็น Troubleshooting-Only เต็มรูปแบบที่รวม Path Control ทุกกลไกที่เรียนมาตลอดหลักสูตร —
เริ่มจาก **Methodology และ Priority Order Checklist** (Step 631) ที่เป็นกุญแจสำคัญที่สุด: **PBR
ถูกตรวจก่อน Routing Table เสมอ** ตามด้วย 8 Scenario จริงที่พบได้ในสนามสอบและ Production:
PBR ACL Wildcard Mask ผิดจนเกิด Fallthrough แบบเงียบ (Step 632), IP SLA Probe Target ที่พังด้วย
เหตุผลไม่เกี่ยวกับ Path จริง (Step 633), Floating Static Route ที่ AD สลับกัน (Step 634),
PBR Apply ผิดทิศทาง Interface (Step 635), Local PBR vs Transit PBR สลับกันผิดจุด (Step 636),
Recursive Static Route ที่หลุด Resolution จนเกิด Routing Loop (Step 637), PBR Asymmetric Return
Path ที่ทำลาย Stateful Firewall (Step 638), และ Route Summarization ที่กลายเป็น Black Hole
Route (Step 639) ปิดท้ายด้วย Lab เต็มรูปแบบที่รวม 2 ปัญหาซ้อนกันหลัง Change Window เดียว (Step
640) เพื่อฝึกการวินิจฉัยอย่างเป็นระบบด้วย Checklist เดียวกันตั้งแต่ต้น Part ✅

**พร้อมสำหรับ Part 65**: เราจะเปลี่ยนไปที่หัวข้อ **Multicast Troubleshooting** — เจาะลึกปัญหาที่
พบจริงของ IGMP, PIM Sparse/Dense Mode, RP (Rendezvous Point) Failure และ Multicast Routing Table
ที่ผูกกับพื้นฐาน Multicast จาก [Part 41-42](part-041-multicast-fundamentals.md) ให้ลึกขึ้นถึงระดับ
ENARSI เต็มรูปแบบ

**ไปต่อ:** [Part 65 — Multicast Troubleshooting →](part-065-multicast-troubleshooting.md)
