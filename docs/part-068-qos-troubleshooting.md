# Part 68 — QoS Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 671–680 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 67 — Automation Troubleshooting](part-067-automation-troubleshooting.md) ที่สอน
> การ Troubleshoot Network Automation Pipeline (NETCONF/RESTCONF, Ansible, Python) เมื่อ Script
> ที่ควร Push Config ที่ถูกต้องกลับ Push Config ผิดหรือ Drift ไปจากที่ตั้งใจ Part นี้จะนำทักษะ
> Troubleshooting แบบเป็นระบบนั้นมาใช้กับหัวข้อที่ [Part 24](part-024-qos-fundamentals.md) และ
> [Part 40](part-040-advanced-qos.md) วางพื้นฐานไว้เต็มรูปแบบแล้ว — **QoS**: Trust Boundary,
> MQC (`class-map`/`policy-map`/`service-policy`), LLQ/CBWFQ, WRED, Shaping/Policing, Hierarchical
> QoS, และ NBAR ที่ Lab ของหลักสูตรนี้ Deploy ไว้ครบแล้วบน `WAN-EDGE-1` (LLQ+CBWFQ+WRED),
> `ACCESS-SW1`/`ACCESS-SW2` (Trust Boundary + Auto-QoS), และ `DIST-SW3`/`DIST-SW4` (Policing บน
> VLAN 40 WIFI) — **ปัญหาที่ Part นี้เจาะคือ: QoS ที่ Config "ถูกต้องตามทฤษฎี" แต่ยังพังในรูปแบบ
> ที่ผู้เรียนไม่คาดคิด** เพราะ QoS มีธรรมชาติพิเศษที่ทำให้ Troubleshoot ยากกว่าหัวข้ออื่น: **ปัญหา
> ส่วนใหญ่ไม่ปรากฏจนกว่าเครือข่ายจะเกิด Congestion จริง**

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 671 | Methodology การ Troubleshoot QoS — ปัญหาที่โผล่มาเฉพาะตอน Congestion และวิธี Load-Test |
| 672 | สถานการณ์ 1 — เสียงแย่ทั้งที่มี LLQ: Trust Boundary หายไปหลัง RMA เปลี่ยน Switch |
| 673 | สถานการณ์ 2 — Policy-map ไม่ทำงานเลย: `service-policy` ผิดทิศทาง (`input` vs `output`) |
| 674 | สถานการณ์ 3 — Class-map จับ Traffic ผิด: ลำดับ Class ใน Policy-map ผิด (First-Match-Wins) |
| 675 | สถานการณ์ 4 — WRED ทิ้ง Voice/UDP ทั้งที่ WRED ควรใช้กับ TCP เท่านั้น |
| 676 | สถานการณ์ 5 — Hierarchical QoS: Parent Shaper ตั้งผิดหน่วยหลัง WAN Circuit ถูก Downgrade |
| 677 | สถานการณ์ 6 — NBAR จำ Application ไม่ได้หลัง Software Update เปลี่ยน Signature/Port |
| 678 | สถานการณ์ 7 — Policer Drop เกินคาดเพราะคำนวณ Burst Size (Bc) ผิด สำหรับ Traffic ที่ Burst แรง |
| 679 | สถานการณ์ 8 — QoS ทำงานบน Physical Interface แต่ไม่ทำงานบน Tunnel — ต้องใช้ `qos pre-classify` |
| 680 | Lab เต็มรูปแบบ: พบสองปัญหาซ้อนกัน (Trust Boundary + Shaper ผิดหน่วย) จาก Voice Quality Audit |

---

## Step 671 — Methodology การ Troubleshoot QoS

### 671.1 ธรรมชาติที่ทำให้ QoS ต่างจากหัวข้อ Troubleshoot อื่นทั้งหมด

หัวข้อ Troubleshoot ก่อนหน้านี้ (EIGRP/OSPF/BGP ใน Part 56-58, NAT ใน Part 63, DMVPN ใน Part 60)
มีธรรมชาติร่วมกันอย่างหนึ่ง: **ถ้า Config ผิด อาการจะเกิดทันทีและสม่ำเสมอ** — Route ไม่ขึ้นก็ไม่ขึ้น
ตลอด, NAT Translation ผิดก็ผิดทุก Connection ทันที แต่ QoS **ไม่เป็นแบบนั้น**:

```
เครือข่ายที่ไม่มี Congestion เลย (Bandwidth เหลือเฟือ):
┌─────────────────────────────────────────────────────────┐
│  ไม่ว่า Policy-map จะถูก Config ถูกหรือผิด แม้แต่ "ไม่มี Policy  │
│  เลย" — Traffic ทุกประเภทก็ยัง "ผ่านได้หมด ไม่มี Drop เลย"        │
│  เพราะไม่มี Queue ไหนต้องรอ ไม่มี Token Bucket ไหนหมด            │
└─────────────────────────────────────────────────────────┘
                            │
                  เกิด Congestion จริง (Load สูงพอ)
                            ▼
┌─────────────────────────────────────────────────────────┐
│  ตอนนี้เท่านั้นที่ Policy-map/Trust Boundary/WRED/Policer      │
│  จะถูก "ทดสอบจริง" — ถ้ามีจุดผิดพลาดซ่อนอยู่ (Trust Boundary   │
│  หาย, Class-map จับผิด, Shaper ผิดหน่วย) จะปรากฏเป็นอาการ      │
│  "ทันที" ตอนนั้น — และหายไปทันทีเมื่อ Congestion ลดลง             │
└─────────────────────────────────────────────────────────┘
```

**ผลกระทบเชิงปฏิบัติที่ต้องเข้าใจก่อน Troubleshoot ทุกครั้ง**: Config QoS ที่ผิดสามารถ **"อยู่เงียบๆ"
ในระบบได้เป็นสัปดาห์หรือเป็นเดือน** โดยไม่มีใครสังเกต จนกว่าจะมี Event ที่ดัน Bandwidth เข้าใกล้
เพดานจริง (พนักงานกลับมาทำงานพร้อมกันหลัง Holiday, Backup Job รันพร้อมกับ Video Conference,
WAN Circuit ถูก Downgrade) — นี่คือเหตุผลที่ **"Config นี้ใช้งานมาตั้งนานแล้ว ไม่เคยมีปัญหา"
ไม่ได้แปลว่า Config ถูกต้อง** มันอาจแค่ยังไม่เจอ Congestion ที่มากพอจะเผยปัญหาเท่านั้น

### 671.2 หลักการ Load-Testing/Verification ที่ถูกต้อง

เพราะปัญหาซ่อนอยู่จนกว่าจะมี Load จริง การ Verify QoS Policy ด้วยการดู `show policy-map interface`
ตอนเครือข่าย**ไม่มี Traffic**เลยจะให้ผลลัพธ์ที่ทำให้เข้าใจผิดได้ — Counter ทุกตัวจะเป็น 0 ทั้ง
Policy ที่ถูกและผิด แยกไม่ออกเลยว่า Config ใช้งานได้จริงหรือไม่ **หลักการที่ต้องทำเสมอ**:

| ขั้นตอน | รายละเอียด |
|---|---|
| 1. Baseline (ก่อนมี Load) | จด `show policy-map interface` ทุก Class ไว้เป็นค่าเริ่มต้น — Packet Count ที่มีอยู่แล้ว (ไม่มี Command "clear" เฉพาะของ Policy-map Counter ใน IOS แบบ Classic ต้องจดค่าไว้เพื่อคำนวณ Delta เอง) |
| 2. สร้าง Load จำลองให้ใกล้เคียงสภาวะจริงที่สุด | ใช้ Traffic Generator (เช่น `iperf3` รันจาก PC1-PC5/Server1-3) ยิง TCP Bulk Transfer ให้เข้าใกล้ Bandwidth เพดานของ Interface/Shaper พร้อมกับสร้าง Voice Call จริง (หรือ RTP Generator) เพื่อดูผลกระทบต่อ Class VOICE ระหว่างที่ Data Class กำลังแย่ง Bandwidth |
| 3. อ่าน `show policy-map interface` ระหว่างมี Load (ทำซ้ำหลายรอบ) | ใช้คำสั่งซ้ำเป็นช่วงๆ (ทุก 10-30 วินาที) เพื่อดู `30 second offered rate`/`drop rate` ที่เปลี่ยนแปลงตาม Load จริง ไม่ใช่ดูครั้งเดียว |
| 4. เปรียบเทียบ Behavior ระหว่าง Class | Class ที่ควรได้รับความคุ้มครอง (VOICE) ต้องมี `drop rate` ใกล้ 0 เสมอแม้ Class อื่น Drop หนัก — ถ้า VOICE เริ่ม Drop พร้อมกับ Class อื่น แสดงว่า Policy ไม่ได้ทำงานตามที่ตั้งใจ |
| 5. คืนสภาพและยืนยันว่า Drop หายไปเมื่อ Load ลดลง | ยืนยันว่า Drop ที่เห็นมาจาก Congestion จริง ไม่ใช่จาก Hardware/Interface Error (`show interfaces` ต้องไม่มี CRC/Input Error เพิ่มขึ้นคู่กัน — ถ้ามีคือปัญหา Layer 1/2 ไม่ใช่ QoS) |

### 671.3 Counter ที่ต้องจับตาดูใน `show policy-map interface` — Checklist หลักของ Part นี้

| Counter | ความหมาย | สัญญาณเตือนปัญหา |
|---|---|---|
| `30 second offered rate` | Traffic ที่เข้า Class นี้จริงตอนนี้ (bps) | ถ้าเป็น 0 ตลอดขณะที่รู้ว่ามี Traffic ประเภทนั้นวิ่งอยู่จริง = Class-map จับผิด หรือ Trust Boundary ไม่ทำงาน (Step 672/674) |
| `drop rate` | Traffic ที่ถูก Drop ใน Class นี้ (bps) | ถ้า Class VOICE มีค่านี้ > 0 ระหว่าง Congestion = ปัญหาร้ายแรงที่สุดของ Part นี้ (Step 672/675/676) |
| `(queue depth/total drops/no-buffer drops)` | ความยาว Queue ปัจจุบัน / Drop สะสม / Drop จาก Memory Buffer หมด | `no-buffer drops` สูง = ปัญหา Hardware/Memory ไม่ใช่ QoS Policy — แยกออกจาก Policy Drop ให้ถูก |
| `Priority: X% (Y kbps)` (LLQ) | เพดาน Policer จริงของ Priority Queue เป็น kbps | ถ้า Y ต่ำผิดปกติเทียบกับที่คาด = Parent Shaper ผิด (Step 676) หรือคำนวณ % จาก Bandwidth ผิดฐาน |
| `police: cir ... bc ...` | ค่า CIR/Bc จริงที่ Policer ใช้งาน | เทียบกับสูตร Bc = CIR × Tc เสมอ ถ้า `exceeded` สูงผิดคาดทั้งที่ Offered Rate ต่ำกว่า CIR = Bc เล็กเกินไป (Step 678) |
| `Mean queue depth` + ตาราง WRED (`Random drop`) | ผลของ WRED แยกตาม DSCP | ถ้าเห็นตาราง WRED ปรากฏอยู่ใต้ Class ที่ควรเป็น Voice = สัญญาณ Config ผิด Class ทันที (Step 675) |

> **กฎเหล็กของ Part นี้**: ไม่มี Counter ตัวไหนบอกความจริงได้ถ้าไม่มี Congestion ระหว่างที่อ่าน —
> ทุก Scenario ใน Step 672-680 จะแสดงคำสั่ง Verify **ทั้งก่อนและระหว่าง** สภาวะ Load จำลอง
> เพื่อให้เห็นความแตกต่างชัดเจน

---

## Step 672 — สถานการณ์ 1: เสียงแย่ทั้งที่มี LLQ Policy — Trust Boundary หายไปหลัง RMA

### 672.1 อาการที่รายงานเข้ามา

Helpdesk รายงาน: พนักงานฝ่ายขาย (VLAN 10 SALES ที่ `ACCESS-SW1`) โทรออกผ่าน IP Phone (VLAN 20
VOICE) แล้วเสียงสะดุดเป็นช่วงๆ **เฉพาะช่วงเวลาที่มีคนโหลดไฟล์ใหญ่พร้อมกันหลายคน** — ทีมงานยืนยันว่า
`WAN-EDGE-1` มี LLQ Policy (`WAN-EDGE-QOS` จาก [Part 40 Step 400](part-040-advanced-qos.md))
Apply อยู่จริงและถูกต้องทุกตัวอักษรเหมือนเดิมไม่มีใครแก้ไข — ที่น่าสงสัยคือ**ปัญหาเริ่มเกิดขึ้น
หลังจากที่ทีม Hardware เปลี่ยน `ACCESS-SW1` ตัวใหม่ (RMA) ไปเมื่อสัปดาห์ก่อน** เพราะตัวเดิมมี
Hardware Fault ที่ Uplink Port

### 672.2 วินิจฉัย: เช็คที่ WAN-EDGE-1 ก่อน — Policy ทำงานถูก แต่ Class VOICE ไม่มี Traffic

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0
  Service-policy output: WAN-SHAPE-50M
    ...
      Service-policy : WAN-EDGE-QOS

        Class-map: VOICE (match-any)
          182 packets, 11,648 bytes
          30 second offered rate 3,000 bps, drop rate 0 bps
          Match: dscp ef (46)
          Priority: 10% (5,000 kbps), burst bytes 125,000

        Class-map: class-default (match-any)
          892,400 packets, 1,338,600,000 bytes
          30 second offered rate 41,200,000 bps, drop rate 6,100,000 bps
          Match: any
```

**สังเกต**: Class VOICE มี Offered Rate แค่ **3,000 bps** ทั้งที่ตอนนี้มีสาย Voice จาก VLAN 20
กำลังคุยอยู่จริงหลายสาย (แต่ละสาย G.711 ต้องใช้ ~64-106 kbps ตาม [Part 24 Step 232](part-024-qos-fundamentals.md))
— แสดงว่า Voice Traffic ส่วนใหญ่**ไม่ได้เข้า Class VOICE เลย** มันไปกองอยู่ที่ `class-default`
(ซึ่งเป็น Class ที่กำลังโดน Drop หนักที่สุดตามที่ตั้งใจออกแบบให้ Data ทั่วไปโดน) — นี่คือสาเหตุ
ที่เสียงสะดุดพอดีตอน Congestion: **เสียงถูกจัดเป็น Best-Effort ไปแล้วก่อนถึง WAN-EDGE-1 ด้วยซ้ำ**

### 672.3 ตาม Trust Boundary กลับไปที่ ACCESS-SW1 (ตาม Part 24 Step 235)

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: not trusted
trust mode: not trusted
COS override: dis
default COS: 0
DSCP Mutation Map: Default DSCP Mutation Map
qos mode: port-based
```

**พบสาเหตุทันที**: `trust state: not trusted` — Port ที่ต่อ IP Phone ไม่ได้ตั้ง Trust Boundary
เลย ต่างจากที่ [Part 24 Step 240](part-024-qos-fundamentals.md) Config ไว้ (`mls qos trust
device cisco-phone`) ยืนยันด้วย `show running-config interface`:

```
ACCESS-SW1# show running-config interface GigabitEthernet1/0/1
interface GigabitEthernet1/0/1
 switchport access vlan 10
 switchport voice vlan 20
 spanning-tree portfast
end
```

บรรทัด `mls qos trust device cisco-phone` และ `auto qos voip cisco-phone` **หายไปทั้งคู่**
และตรวจ Global ด้วย `show mls qos` ก็พบว่า **แม้แต่ `mls qos` ระดับ Global ก็ไม่ได้เปิดไว้ด้วย**:

```
ACCESS-SW1# show mls qos
QoS is disabled globally
```

### 672.4 Root Cause: RMA ได้ Switch ตัวใหม่ที่ Restore จาก Golden Config เก่ากว่า Part 24

ทีม Hardware เปลี่ยน `ACCESS-SW1` ตัวใหม่แล้วใช้ Automation Script ([Part 48](part-048-automation-ansible-advanced.md))
Push Config จาก **Golden Config Template ที่เก็บไว้ตั้งแต่ก่อน Part 24 Lab จะถูกทำ** — Template
นั้นมีแค่ `switchport access vlan`/`switchport voice vlan`/`spanning-tree portfast` (พื้นฐาน
Layer 2 จาก Part 3-6) แต่ไม่มี QoS Trust Boundary ที่เพิ่มเข้ามาทีหลังใน Part 24 เพราะ**ไม่มีใคร
อัปเดต Golden Config Template ให้ตรงกับ Production Config ปัจจุบันหลังทำ Lab QoS เสร็จ** — Switch
ตัวเดิมก่อน RMA มี Trust Boundary Config อยู่ (Manual Config ทับ Golden Config ไปแล้วนานแล้ว)
แต่ไม่เคยถูก Backup กลับเข้า Template — พอเปลี่ยน Hardware ก็ได้ Config เวอร์ชันเก่ากลับมาแทน

### 672.5 วิธีแก้: Restore Trust Boundary และอัปเดต Golden Config Template

```
ACCESS-SW1(config)# mls qos
ACCESS-SW1(config)# interface GigabitEthernet1/0/1
ACCESS-SW1(config-if)# mls qos trust device cisco-phone
ACCESS-SW1(config-if)# mls qos trust cos
ACCESS-SW1(config-if)# auto qos voip cisco-phone
ACCESS-SW1(config-if)# exit
```

ทำซ้ำกับทุก Port ที่ต่อ IP Phone บน `ACCESS-SW1` (และตรวจ `ACCESS-SW2` ด้วยว่ารอด RMA รอบนี้มา
หรือไม่ — ถ้าไม่ได้ RMA ก็ไม่กระทบ แต่ควร Verify ให้แน่ใจเสมอ) จากนั้น**อัปเดต Golden Config
Template** ใน Ansible Repository ให้รวม QoS Command ไว้ด้วย เพื่อไม่ให้ RMA ครั้งหน้าเจอปัญหา
เดิมซ้ำ — นี่คือ Lesson ที่เชื่อมกับ [Part 67](part-067-automation-troubleshooting.md) ตรงๆ:
**Automation Drift ไม่ได้เกิดจาก Script รันผิดเท่านั้น แต่เกิดจาก "Source of Truth" (Golden
Config) ที่ไม่ถูกอัปเดตให้ตรงกับ Production ได้เหมือนกัน**

### 672.6 Verify หลังแก้ไข

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: trust cos
trust mode: trust cos
Trust device: cisco-phone

WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE
        Class-map: VOICE (match-any)
          188,400 packets, 12,057,600 bytes
          30 second offered rate 452,000 bps, drop rate 0 bps
          Priority: 10% (5,000 kbps), burst bytes 125,000
```

Class VOICE เห็น Offered Rate เพิ่มขึ้นเป็น **452,000 bps** ทันที (ตรงกับปริมาณสาย Voice จริงที่
มีอยู่) — ยืนยันว่า Voice Traffic เดินทางเข้า LLQ Queue ที่ถูกต้องแล้ว ไม่ใช่ไปกอง `class-default`
เหมือนก่อนแก้ไข

---

## Step 673 — สถานการณ์ 2: Policy-map ไม่ทำงานเลย — `service-policy` ผิดทิศทาง

### 673.1 อาการที่รายงานเข้ามา

หลังจากมีการทำ Path Redundancy Failover Test ([Part 58 Step 58x](part-058-bgp-troubleshooting.md)
สลับ Preferred Path ไปทาง `WAN-EDGE-2` ชั่วคราว) พบว่า Voice Quality แย่ลงอย่างเห็นได้ชัดเมื่อ
Traffic วิ่งผ่าน `WAN-EDGE-2` แทน `WAN-EDGE-1` — ทีมงานยืนยันว่าได้ Copy Policy-map ชุดเดียวกัน
(`WAN-EDGE-QOS`/`WAN-SHAPE-50M`) ไปที่ `WAN-EDGE-2` แล้ว "ครบทุกบรรทัด"

### 673.2 วินิจฉัย: `show policy-map interface` ไม่แสดง Counter อะไรเลยที่ทิศทางที่คาดไว้

```
WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0

  Service-policy output: WAN-SHAPE-50M

    Class-map: class-default (match-any)
      0 packets, 0 bytes
      30 second offered rate 0 bps, drop rate 0 bps
      Match: any
      Queueing
      shape (average) cir 50,000,000, bc 1,000,000, be 1,000,000
```

Policy ติดอยู่จริง แต่**ทุก Counter เป็น 0 ทั้งที่มี Traffic วิ่งออก Interface นี้จริงแน่นอน**
(`show interfaces GigabitEthernet0/0/0` ยืนยันว่า Output Rate สูงอยู่) — สิ่งนี้คือสัญญาณเฉพาะ
ของปัญหา "Policy Apply ผิดจุด" เพราะถ้า Class-map จับผิดยังพอมี Traffic ไปโปะที่ `class-default`
บ้าง แต่ถ้า 0 เกลี้ยงแบบนี้แปลว่า **Policy ที่กำลังดูอยู่ไม่ได้ประมวลผล Traffic ทิศทางที่คิดไว้เลย**

### 673.3 ตรวจ Running-Config ให้ครบทุกทิศทาง

```
WAN-EDGE-2# show running-config interface GigabitEthernet0/0/0
interface GigabitEthernet0/0/0
 description ** WAN link to ISP-RTR (Metro-Ethernet, CIR 50M) **
 ip address 203.0.113.6 255.255.255.252
 service-policy input WAN-SHAPE-50M
 no shutdown
```

**พบสาเหตุ**: บรรทัดคือ `service-policy input WAN-SHAPE-50M` — วิศวกรพิมพ์ `input` เพราะเพิ่ง
Config `service-policy input WIFI-POLICE` บน `DIST-SW3`/`DIST-SW4` (ตาม [Part 40 Step 400](part-040-advanced-qos.md))
มาก่อนหน้านี้ในวันเดียวกัน ทำให้ "มือเผลอ" พิมพ์ `input` ซ้ำโดยไม่ได้ตั้งใจ (Muscle Memory Error
ที่พบบ่อยเวลาสลับไปมาระหว่าง Config หลายจุดที่ใช้ Direction คนละแบบในวันเดียวกัน) — เพราะ
`WAN-SHAPE-50M` มี `class-default` ที่ Match "any" อยู่แล้ว มันจึงยัง"ติด"ที่ Interface และแสดงผล
ใน `show policy-map interface` ได้ (ไม่ Error) แต่กำลังจับ **Traffic ทิศทางขาเข้า (จาก ISP เข้า
เรา)** ซึ่งตอนทดสอบมี Traffic น้อยมาก ในขณะที่ Traffic ขาออก (จากเราไป ISP ที่ Voice/Shaping
ต้องทำงานจริง) **ไม่มี Policy อะไรควบคุมเลย** — Voice จึงไปปนกับ Data เต็มๆ เหมือนไม่มี QoS

> **ข้อสังเกตสำคัญ**: สาเหตุอื่นที่ให้ผลลัพธ์เดียวกัน (Counter เป็น 0 ทั้งที่มี Traffic จริง) คือ
> **Apply Policy ผิด Sub-interface** เช่นถ้า `WAN-EDGE-2` ใช้ `GigabitEthernet0/0/0.100` (Dot1Q
> Sub-interface สำหรับ VLAN ทางออกที่สอง) แต่ Traffic จริงวิ่งผ่าน Native `GigabitEthernet0/0/0`
> โดยตรง — วิธีตรวจแยกสองสาเหตุนี้เหมือนกัน: ตรวจ `show running-config interface` ให้ตรงทั้ง
> Interface **และ** Direction เทียบกับที่ Traffic จริงไหลผ่าน

### 673.4 วิธีแก้และ Verify

```
WAN-EDGE-2(config)# interface GigabitEthernet0/0/0
WAN-EDGE-2(config-if)# no service-policy input WAN-SHAPE-50M
WAN-EDGE-2(config-if)# service-policy output WAN-SHAPE-50M
WAN-EDGE-2(config-if)# exit
```

```
WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0
  Service-policy output: WAN-SHAPE-50M
    Class-map: class-default (match-any)
      812,400 packets, 974,880,000 bytes
      30 second offered rate 48,200,000 bps, drop rate 3,100,000 bps
      shape (average) cir 50,000,000, bc 1,000,000, be 1,000,000
      Service-policy : WAN-EDGE-QOS
        Class-map: VOICE (match-any)
          9,200 packets, 588,800 bytes
          30 second offered rate 158,000 bps, drop rate 0 bps
          Priority: 10% (5,000 kbps), burst bytes 125,000
```

ตอนนี้ Counter ขึ้นตรงกับ Traffic ที่ไหลผ่านจริงทุก Class — **หลักการ Verify ที่ต้องจำจาก
Scenario นี้**: `show policy-map interface` ที่ Counter เป็น 0 เกลี้ยงทั้งที่รู้ว่ามี Traffic
วิ่งผ่าน Interface นั้นแน่นอน ให้ตรวจ **Direction** และ **Interface/Sub-interface ที่แท้จริง**
เป็นอันดับแรกเสมอ ก่อนไปสงสัยเรื่อง Class-map หรือกลไก QoS อื่นใดทั้งสิ้น

---

## Step 674 — สถานการณ์ 3: Class-map จับ Traffic ผิดเพราะลำดับ Class ใน Policy-map ผิด

### 674.1 บริบท: ต้องการแยก Voice Signaling (SIP/SCCP) ออกมาให้ Bandwidth การันตีเฉพาะ

ทีม Network ต้องการเพิ่ม Class ใหม่ชื่อ `VOICE-SIGNALING` ให้ Traffic ควบคุมการโทร (DSCP CS3
ตาม [Part 24 Step 234](part-024-qos-fundamentals.md)) ได้รับ Bandwidth การันตีแยกจาก Data
ทั่วไป เพื่อไม่ให้การตั้งสาย/วางสายล่าช้าตอน Congestion แม้ว่า Voice Signaling เองไม่ไวต่อ Delay
เท่าเสียงพูดจริง (RTP) ก็ตาม — วิศวกรเพิ่ม Class เข้าไปใน Policy-map เดิมโดยวางไว้ **ต่อจาก
`CRITICAL-DATA` (Class เดิม)**:

```
policy-map WAN-EDGE-QOS
 class VOICE
  priority percent 10
 class CRITICAL-DATA
  bandwidth percent 30
  random-detect dscp-based
 class VOICE-SIGNALING
  bandwidth percent 5
 class VIDEO
  bandwidth percent 15
 class class-default
  fair-queue
  random-detect dscp-based
```

### 674.2 อาการที่รายงานเข้ามา

การตั้งสาย (Ringback ช้า, บางครั้ง Timeout ตอนวางสาย) แย่ลงตอน Congestion ทั้งที่เพิ่ม Class
`VOICE-SIGNALING` ไปแล้ว ทีมงานยืนยันว่า Class-map `VOICE-SIGNALING` เขียนถูก (`match dscp cs3`)
และ Policy-map ก็ Apply อยู่ที่ทิศทาง Output ถูกต้องตามที่ตรวจใน Step 673

### 674.3 วินิจฉัย: `show policy-map interface` — VOICE-SIGNALING ไม่มี Traffic เลย

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE-SIGNALING|CRITICAL-DATA
        Class-map: CRITICAL-DATA (match-any)
          615,200 packets, 786,900,000 bytes
          30 second offered rate 15,900,000 bps, drop rate 610,000 bps
          Match: dscp af31 (26)
          Match: dscp af21 (18)
          Match: dscp cs3 (24)
        Class-map: VOICE-SIGNALING (match-any)
          0 packets, 0 bytes
          30 second offered rate 0 bps, drop rate 0 bps
          Match: dscp cs3 (24)
```

**พบสาเหตุทันที**: Class-map `CRITICAL-DATA` (ที่นิยามไว้ตั้งแต่ [Part 40 Step 392](part-040-advanced-qos.md))
มี `match dscp cs3` อยู่ในตัวเองอยู่แล้ว (นิยามไว้ให้ครอบคลุม Signaling ตั้งแต่แรกตามตาราง DSCP
Part 24 Step 234) — เพราะ IOS ประมวลผล Class ตาม**ลำดับที่พิมพ์ไว้จากบนลงล่างและหยุดที่ Class
แรกที่ Match (First-Match-Wins)** ตามกฎที่ [Part 40 Step 391](part-040-advanced-qos.md) สอนไว้
Packet ที่มี DSCP CS3 ทุกตัวจึงถูกจัดเข้า `CRITICAL-DATA` **ก่อน**ที่จะไปถึง Class `VOICE-SIGNALING`
ที่วางไว้ทีหลัง — Class ใหม่จึงไม่มีวันได้รับ Traffic แม้แต่ Packet เดียว ทั้งที่ Class-map ของมัน
เองเขียนถูกทุกตัวอักษร

```
WAN-EDGE-1# show policy-map WAN-EDGE-QOS
  Policy Map WAN-EDGE-QOS
    Class VOICE
      priority 10 (%)
    Class CRITICAL-DATA          ← อยู่ก่อน VOICE-SIGNALING แต่ match dscp cs3 (24) ด้วย
      bandwidth 30 (%)
      random-detect dscp-based
    Class VOICE-SIGNALING        ← ไม่มีวัน Match เพราะ cs3 ถูก CRITICAL-DATA เก็บไปหมดแล้ว
      bandwidth 5 (%)
```

### 674.4 Root Cause

การเพิ่ม Class ใหม่เข้า Policy-map เดิมโดยไม่ตรวจสอบว่า **Class ที่มีอยู่แล้วก่อนหน้ามีเงื่อนไข
`match-any` ที่ทับซ้อนกับเงื่อนไขของ Class ใหม่หรือไม่** — Class-map `CRITICAL-DATA` เดิมออกแบบ
มาให้ครอบคลุม CS3 ตั้งแต่ก่อนที่จะมีแนวคิดแยก Signaling ออกมาต่างหาก พอเพิ่ม Class ใหม่ทีหลัง
โดยวางไว้ผิดตำแหน่ง (หลัง Class ที่ Match เงื่อนไขเดียวกันอยู่แล้ว) จึงเกิดปัญหานี้

### 674.5 วิธีแก้ — ต้องแก้ 2 จุดร่วมกันเสมอ (ลำดับ + เงื่อนไขที่ทับซ้อน)

**วิธีที่ถูกต้องที่สุด**: เอา `match dscp cs3` ออกจาก `CRITICAL-DATA` (เพราะตอนนี้ VOICE-SIGNALING
ควรเป็นเจ้าของ CS3 แต่เพียงผู้เดียว) **และ** ย้าย Class `VOICE-SIGNALING` ให้อยู่**ก่อน**
`CRITICAL-DATA` เพื่อความชัดเจนแม้ Class-map จะไม่ทับซ้อนกันแล้วก็ตาม (Best Practice: วาง Class
ที่เจาะจงกว่าไว้บนเสมอ ไม่ให้ต้องพึ่งพาการไม่ทับซ้อนของ Class-map เพียงอย่างเดียว):

```
WAN-EDGE-1(config)# class-map match-any CRITICAL-DATA
WAN-EDGE-1(config-cmap)# no match dscp cs3
WAN-EDGE-1(config-cmap)# exit
!
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# no class VOICE-SIGNALING
WAN-EDGE-1(config-pmap)# class VOICE-SIGNALING
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 5
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit
```

> **หมายเหตุ**: IOS ไม่มีคำสั่ง "ย้ายลำดับ Class" ตรงๆ — วิธีจัดลำดับใหม่ต้องลบ Class ที่ต้องการ
> ย้ายด้วย `no class <name>` แล้ว Config กลับเข้าไปใหม่ (Class ที่ Config ใหม่จะถูกเติมไว้ท้ายสุด
> ก่อน `class-default` เสมอ — ถ้าต้องการลำดับที่แน่นอนสำหรับ Class หลายตัว ต้อง `no class` ทุกตัว
> ที่อยู่หลังจุดที่ต้องการแทรกแล้ว Config กลับตามลำดับใหม่ทั้งหมด)

### 674.6 Verify

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE-SIGNALING
        Class-map: VOICE-SIGNALING (match-any)
          48,200 packets, 3,084,800 bytes
          30 second offered rate 62,000 bps, drop rate 0 bps
          Match: dscp cs3 (24)
          bandwidth 5% (2,500 kbps)
```

Class `VOICE-SIGNALING` เริ่มเห็น Traffic แล้ว — บทเรียนสำคัญของ Scenario นี้: **ทุกครั้งที่เพิ่ม
Class ใหม่เข้า Policy-map เดิม ต้องตรวจ `show class-map` ของทุก Class ที่อยู่ก่อนหน้าเสมอว่ามี
เงื่อนไขทับซ้อนกับ Class ใหม่หรือไม่** — First-Match-Wins ไม่สนใจว่า Class ไหน "ควร" ได้ Traffic
ตามเจตนาของผู้ Config มันสนใจแค่ลำดับที่เขียนไว้เท่านั้น

---

## Step 675 — สถานการณ์ 4: WRED ทิ้ง Voice/UDP ทั้งที่ WRED ควรใช้กับ TCP เท่านั้น

### 675.1 บริบท: WAN-EDGE-2 มี Config Drift จากการ Rebuild แบบเร่งรีบ

หลังจาก `WAN-EDGE-2` ผ่านการซ่อม Hardware ครั้งใหญ่ ทีมงาน Rebuild Policy-map ขึ้นใหม่แบบเร่งรีบ
โดยใช้ Script "เติม WRED ให้ทุก Class ที่เป็น Bandwidth-based" (เขียนไว้ตั้งแต่ก่อนมี LLQ ใน Lab
Part 40) แต่ระหว่าง Rebuild วิศวกรพิมพ์ Class VOICE ด้วย `bandwidth percent 10` (CBWFQ) โดยเผลอ
**ไม่ได้ใช้ `priority percent 10` (LLQ) ตามที่ควรจะเป็น** — เพราะ `bandwidth` ไม่ใช่ `priority`
IOS จึงไม่ปฏิเสธ Script ที่ตามมาทีหลังซึ่งเติม `random-detect dscp-based` ให้ Class นี้เหมือน
Class อื่นๆ ทุกตัว (กฎที่ [Part 40 Step 394](part-040-advanced-qos.md) บอกไว้ — IOS ปฏิเสธ
`random-detect` เฉพาะในคลาสเดียวกันกับ `priority` เท่านั้น — Class ที่ใช้ `bandwidth` ธรรมดา
ไม่ถูกปฏิเสธเลยแม้จะเป็น Class ที่ตั้งใจไว้สำหรับ Voice ก็ตาม):

```
policy-map WAN-EDGE-QOS
 class VOICE
  bandwidth percent 10          ← ผิด! ควรเป็น "priority percent 10"
  random-detect dscp-based      ← IOS ยอมรับเพราะไม่เจอ "priority" ในคลาสนี้
 class CRITICAL-DATA
  bandwidth percent 30
  random-detect dscp-based
 class class-default
  fair-queue
  random-detect dscp-based
```

### 675.2 อาการที่รายงานเข้ามา

Voice Call ผ่าน `WAN-EDGE-2` มีเสียงหายเป็นช่วงสั้นๆ (Choppy) ตอน Congestion แม้ Bandwidth
ที่จัดสรรให้ (10%) ดูเหมือนเพียงพอในทางทฤษฎี — ต่างจาก Step 672 ที่เสียง**หายไปเลยเข้า
Best-Effort** สถานการณ์นี้เสียง**ยังอยู่ใน Class ที่ถูกต้อง** แต่ยังถูก Drop อยู่ดี

### 675.3 วินิจฉัย: `show policy-map interface` เผย WRED Table อยู่ใต้ Class VOICE

```
WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0 | section Class-map: VOICE
        Class-map: VOICE (match-any)
          188,000 packets, 12,032,000 bytes
          30 second offered rate 455,000 bps, drop rate 38,000 bps
          Match: dscp ef (46)
          Queueing
          bandwidth 10% (5,000 kbps)
          Exp-weight-constant: 9 (1/512)
          Mean queue depth: 4 packets
            dscp   Transmitted    Random drop   Tail drop   Minimum   Maximum   Mark
            ef     185600/11.9M   2400/154K     0/0          20        40        1/10
```

**สัญญาณที่ต้องจับได้ทันที**: Class VOICE แสดง `bandwidth 10%` (ไม่ใช่ `Priority: 10%`) และมี
ตาราง WRED (`Random drop`) ปรากฏอยู่ — ทั้งสองอย่างนี้**ไม่ควรมีอยู่ใน Class ที่ตั้งใจให้เป็น
Voice เด็ดขาด** ตามกฎ Part 40 Step 393-394: Voice ต้องใช้ `priority` (LLQ) เท่านั้น และห้ามมี
`random-detect` เด็ดขาดเพราะ RTP/UDP ไม่มี Mechanism ตอบสนองต่อการ Random Drop — ตัวเลข
`Random drop 2400/154K` ยืนยันตรงตัวว่า WRED กำลังสุ่มทิ้ง Voice Packet จริงโดยไม่จำเป็นเลย
(Voice ไม่มี TCP Congestion Control ให้ WRED "สื่อสาร" ด้วยตามที่ Part 40 Step 394 อธิบายไว้)

### 675.4 Root Cause

**สองความผิดพลาดซ้อนกัน**: (1) Class VOICE ถูก Config ด้วย `bandwidth` แทน `priority` ทำให้
ไม่ได้ Strict Priority Treatment ที่ Voice ต้องการจริงๆ ตั้งแต่แรก และ (2) เพราะ IOS ตรวจสอบ
Conflict ระหว่าง `priority`/`random-detect` **เฉพาะภายในคลาสเดียวกัน**เท่านั้น — มันไม่รู้ (และ
ไม่มีทางรู้) ว่า Class-map ที่ใช้อยู่บังเอิญ Match DSCP EF ซึ่งเป็นค่าที่ "ควร" สงวนไว้ให้ Voice
เท่านั้น การป้องกันที่ Level Command เพียงอย่างเดียวไม่พอ ต้องอาศัยความเข้าใจของผู้ Config เอง

### 675.5 วิธีแก้: แปลง Class ให้เป็น LLQ ที่ถูกต้อง และถอด WRED ออก

```
WAN-EDGE-2(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-2(config-pmap)# class VOICE
WAN-EDGE-2(config-pmap-c)# no random-detect dscp-based
WAN-EDGE-2(config-pmap-c)# no bandwidth percent 10
WAN-EDGE-2(config-pmap-c)# priority percent 10
WAN-EDGE-2(config-pmap-c)# exit
```

> **บทเรียนสำหรับ Config Review**: ทุกครั้งที่ Deploy/Rebuild Policy-map ที่มี Class Voice
> ควร Grep หา `random-detect` ที่อยู่ **ใต้ Class เดียวกันกับที่ Class-map Match `dscp ef`**
> เป็นข้อบังคับใน Checklist Review เสมอ ไม่ว่า Class นั้นจะใช้ `bandwidth` หรือ `priority`
> ก็ตาม — เพราะ IOS จะไม่เตือนให้เมื่อ Class ใช้ `bandwidth`

### 675.6 Verify

```
WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0 | section Class-map: VOICE
        Class-map: VOICE (match-any)
          189,400 packets, 12,121,600 bytes
          30 second offered rate 458,000 bps, drop rate 0 bps
          Match: dscp ef (46)
          Priority: 10% (5,000 kbps), burst bytes 125,000
```

`drop rate 0 bps` และไม่มีตาราง WRED ปรากฏอีก — Voice ได้รับ Strict Priority Treatment
ที่ถูกต้องแล้ว

---

## Step 676 — สถานการณ์ 5: Hierarchical QoS — Parent Shaper ตั้งผิดหน่วยหลัง WAN Circuit Downgrade

### 676.1 บริบท: ผู้ให้บริการ Downgrade CIR จาก 50 Mbps เหลือ 20 Mbps

องค์กรตัดสินใจลด CIR ของ Metro-Ethernet ที่ `WAN-EDGE-1` จาก **50 Mbps ลงเหลือ 20 Mbps**
(ปรับสัญญาลดค่าใช้จ่ายรายเดือน) ทีมงานต้องปรับ Parent Shaper ใน HQoS ตาม [Part 40 Step
397](part-040-advanced-qos.md) ให้ตรงกับ CIR ใหม่ — วิศวกรที่รับงานนี้คุ้นเคยกับคำสั่ง
`bandwidth <kbps>` ที่ Interface (หน่วยเป็น **kbps**) มาตลอด จึงเผลอพิมพ์ตัวเลขแบบเดียวกันให้
คำสั่ง `shape average` (ที่ต้องการหน่วยเป็น **bps**):

```
WAN-EDGE-1(config)# policy-map WAN-SHAPE-50M
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)# no shape average 50000000
WAN-EDGE-1(config-pmap-c)# shape average 20000        ← ตั้งใจจะให้ = 20 Mbps แต่หน่วยผิด!
WAN-EDGE-1(config-pmap-c)# exit
```

`shape average 20000` หมายถึง **20,000 bps = 20 kbps** (ไม่ใช่ 20 Mbps ตามที่ตั้งใจ) — ผิดไป
จากที่ต้องการถึง **1,000 เท่า**

### 676.2 อาการที่รายงานเข้ามา

ทันทีหลัง Maintenance Window ปิด **ทุกสายที่โทรผ่าน WAN-EDGE-1 หลุดหรือฟังไม่ออกเลยแม้จะมีสาย
เดียวกำลังคุยอยู่** (ต่างจาก Step 672/675 ที่ยังพอฟังได้บ้าง) — Helpdesk เห็นปริมาณ Complaint
พุ่งขึ้นทันทีหลัง Maintenance ตรงเวลาเป๊ะ ทำให้สงสัย Config ที่เพิ่งเปลี่ยนไปทันที

### 676.3 วินิจฉัย: อ่านค่า `target shape rate` และ `Priority` ที่คำนวณได้จริง

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0
  Service-policy output: WAN-SHAPE-50M
    Class-map: class-default (match-any)
      shape (average) cir 20,000, bc 500, be 500
      target shape rate 20,000

      Service-policy : WAN-EDGE-QOS
        Class-map: VOICE (match-any)
          Priority: 10% (2 kbps), burst bytes 250
```

**พบสาเหตุทันที**: `target shape rate 20,000` (bps) — เทียบกับที่ตั้งใจไว้คือ 20,000,000 bps
(20 Mbps) ต่างกัน 1,000 เท่าตรงตัว และผลกระทบที่ตกลงมาถึง Class VOICE ชัดเจนที่สุด: `Priority:
10% (2 kbps)` — LLQ คำนวณเพดานจาก 10% ของกรอบที่ Shape ไว้ (20 kbps) จึงได้แค่ **2 kbps** ทั้งที่
สาย Voice เดียว (G.711) ต้องการ 64-106 kbps ตามมาตรฐาน [Part 24 Step 232](part-024-qos-fundamentals.md)
— เพดาน Policer ในตัวของ LLQ (ตามหลักการ [Part 40 Step 393](part-040-advanced-qos.md)) จึงทิ้ง
Voice Packet ส่วนใหญ่ทันทีที่มีแม้แต่สายเดียว เพราะ 2 kbps ไม่พอสำหรับสายเดียวด้วยซ้ำ ไม่ต้องพูดถึง
Congestion

### 676.4 Root Cause: ความสับสนหน่วยระหว่าง `bandwidth` (kbps) กับ `shape average` (bps)

| คำสั่ง | หน่วยที่ต้องการ | ตัวอย่างค่าที่ถูกสำหรับ 20 Mbps |
|---|---|---|
| `bandwidth <value>` (Interface-level หรือ CBWFQ) | **kbps** | `bandwidth 20000` |
| `shape average <value>` (Policy-map) | **bps** | `shape average 20000000` |
| `police cir <value>` (Policy-map) | **bps** | `police cir 20000000` |

นี่คือกับดักที่พบบ่อยมากในทางปฏิบัติ เพราะ IOS ใช้หน่วยไม่สอดคล้องกันระหว่างคำสั่งกลุ่ม Interface
(kbps) กับคำสั่งกลุ่ม MQC Rate-based (`shape`/`police`, เป็น bps) — วิศวกรที่คุ้นมือกับ `bandwidth`
มักพลาดจุดนี้เวลาต้องรีบปรับ Shaper ให้ตรงกับสัญญาใหม่ภายใน Maintenance Window ที่มีเวลาจำกัด

### 676.5 วิธีแก้: แก้หน่วยให้ถูก และ Verify สูตรคำนวณ

```
WAN-EDGE-1(config)# policy-map WAN-SHAPE-50M
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)# no shape average 20000
WAN-EDGE-1(config-pmap-c)# shape average 20000000
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# exit
```

> **แนวทางป้องกันสำหรับ Maintenance Window ครั้งต่อไป**: คำนวณและเขียนค่า Target ทั้งหน่วย
> Mbps และ bps ไว้ใน Change Request ล่วงหน้าเสมอ (เช่น "20 Mbps = 20,000,000 bps") แล้ว Copy
> ค่า bps ไปวางตรงในคำสั่งจริง ไม่พิมพ์ตัวเลขจากความจำระหว่าง Maintenance Window

### 676.6 Verify — เปรียบเทียบก่อน/หลังภายใต้ Load จำลอง

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0

  GigabitEthernet0/0/0
  Service-policy output: WAN-SHAPE-50M
    Class-map: class-default (match-any)
      shape (average) cir 20,000,000, bc 1,000,000, be 1,000,000
      target shape rate 20,000,000

      Service-policy : WAN-EDGE-QOS
        Class-map: VOICE (match-any)
          452,800 packets, 28,979,200 bytes
          30 second offered rate 458,000 bps, drop rate 0 bps
          Priority: 10% (2,000 kbps), burst bytes 50,000
```

`Priority: 10% (2,000 kbps)` — คราวนี้คำนวณถูกต้อง (10% ของ 20 Mbps = 2 Mbps) รองรับสาย Voice
ได้หลายสิบสายพร้อมกันสบายๆ และ `drop rate 0 bps` ยืนยันว่า Voice ไม่ถูกกระทบแล้วแม้ Link จะเข้า
ใกล้ CIR ใหม่ 20 Mbps ก็ตาม

---

## Step 677 — สถานการณ์ 6: NBAR จำ Application ไม่ได้หลัง Software Update

### 677.1 อาการที่รายงานเข้ามา

ทีมงานสังเกตว่า Video Conference ผ่าน Webex คุณภาพแย่ลงเป็นระยะๆ ตอน Congestion **ทั้งที่ไม่มี
ใครแก้ Policy-map หรือ Class-map บน `WAN-EDGE-1` เลย** — สิ่งเดียวที่เปลี่ยนคือฝ่าย IT Endpoint
เพิ่ง Force Update Webex Client Version ใหม่ให้พนักงานทุกคนเมื่อสัปดาห์ก่อนตามนโยบาย Security
Patch ปกติ

### 677.2 วินิจฉัย: เปรียบเทียบ NBAR Protocol Discovery ก่อน/หลัง Update

```
WAN-EDGE-1# show ip nbar protocol-discovery interface GigabitEthernet0/0/0 top-n 5

                            Input                    Output
                            -----                    ------
  Protocol                 Packet Count              Packet Count
  -----------------------  ------------------------  ------------------------
  ssl                       892000                    885000
                            580000000                 576000000
  http                      210000                    208000
  unclassified               95000                     94500
  ms-teams                  120000                    118000
  webex-meeting                 800                       780
```

เทียบกับก่อน Update ที่ `webex-meeting` เคยเป็น Top-1 ด้วยปริมาณหลักแสน Packet ตอนนี้เหลือแค่
**~800 Packet** ในขณะที่ `ssl` (Traffic ที่ NBAR แยกไม่ออกว่าเป็น Application อะไร รู้แค่ว่าเป็น
TLS) พุ่งขึ้นมาเป็นอันดับ 1 แทน — ปริมาณ Byte ใกล้เคียงกับที่ Webex เคยใช้พอดี ยืนยันว่า **Traffic
เดิมยังอยู่ (คนยังใช้ Webex เหมือนเดิม) แต่ NBAR "จำ" มันไม่ได้แล้ว**

### 677.3 Confirm ด้วย `show policy-map interface`

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VIDEO-CONFERENCE|class-default
        Class-map: VIDEO-CONFERENCE (match-any)
          820 packets, 524,800 bytes
          30 second offered rate 1,200 bps, drop rate 0 bps
        Class-map: class-default (match-any)
          912,400 packets, 1,368,600,000 bytes
          30 second offered rate 44,600,000 bps, drop rate 8,900,000 bps
```

Class `VIDEO-CONFERENCE` (ที่ Match `protocol webex-meeting`/`webex-media` ตาม [Part 40 Step
398](part-040-advanced-qos.md)) เหลือ Traffic แทบไม่มี — Webex ทั้งหมดตกไปที่ `class-default`
และกำลังโดน Drop ร่วมกับ Data ทั่วไปเหมือนไม่มี QoS สำหรับ Video Conference เลย

### 677.4 Root Cause: Client Version ใหม่เปลี่ยน Traffic Signature ที่ NBAR รู้จัก

Webex Client เวอร์ชันใหม่เปลี่ยนพฤติกรรมการเชื่อมต่อ (ย้ายไปใช้ TLS 1.3 กับ Encrypted Client
Hello, เปลี่ยน Media Relay Endpoint, หรือปรับ Byte Pattern ภายใน Handshake) ทำให้ **NBAR2
Signature ที่ผูกกับ Protocol Pack (PP) เวอร์ชันปัจจุบันในเครื่องไม่ตรงกับรูปแบบ Traffic ใหม่
อีกต่อไป** — NBAR อ่าน Traffic นี้ไม่ออกว่าเป็น `webex-meeting` จึงจัดเป็น `ssl` (รู้แค่ว่าเป็น
TLS ทั่วไป) แทน ยืนยัน Protocol Pack ที่ใช้อยู่ด้วย:

```
WAN-EDGE-1# show ip nbar version
NBAR software version:       26
NBAR minimum backward compatible version: 23
Loaded Protocol Pack(s):
Name:                  Advanced Protocol Pack
Version:               36.0
Publisher:             Cisco Systems Inc.
NBAR Engine Version:   26
```

Protocol Pack 36.0 ถูกโหลดไว้ตั้งแต่ก่อน Webex ปล่อย Client เวอร์ชันใหม่ — Cisco มักออก Protocol
Pack ใหม่ตามหลัง Application ใหญ่ๆ เปลี่ยน Signature แต่**ไม่ใช่ทันที** อาจต้องรอหลายสัปดาห์

### 677.5 วิธีแก้ที่ 1 — อัปเดต NBAR Protocol Pack (ถ้า Cisco ออกตัวใหม่ที่รองรับแล้ว)

```
WAN-EDGE-1# copy tftp://10.10.99.50/pp-nbar-2311.pack flash:
WAN-EDGE-1(config)# ip nbar protocol-pack flash:pp-nbar-2311.pack
WAN-EDGE-1# show ip nbar protocol-pack flash:pp-nbar-2311.pack detail | include webex
  webex-meeting                     Signature updated for TLS 1.3 ECH pattern (v2311)
```

### 677.6 วิธีแก้ที่ 2 — Custom NBAR Classification (ถ้ายังไม่มี Protocol Pack ใหม่รองรับ)

ถ้า Cisco ยังไม่ออก Protocol Pack ที่รู้จัก Signature ใหม่ ให้สร้าง Custom Protocol ด้วยเงื่อนไข
ที่ยังคงที่ (เช่น Destination Port Range ของ Media Relay ที่ Webex เผยแพร่ไว้อย่างเป็นทางการ)
เพื่อใช้แทน Signature-based Matching ชั่วคราว:

```
WAN-EDGE-1(config)# ip nbar custom WEBEX-MEDIA-2026 transport udp range 5004 5090
WAN-EDGE-1(config)# class-map match-any VIDEO-CONFERENCE
WAN-EDGE-1(config-cmap)# match protocol webex-meeting
WAN-EDGE-1(config-cmap)# match protocol webex-media
WAN-EDGE-1(config-cmap)# match protocol WEBEX-MEDIA-2026
WAN-EDGE-1(config-cmap)# exit
```

> **ข้อจำกัดของวิธีที่ 2**: Custom Protocol ที่ Match ด้วย Port Range เพียงอย่างเดียวแม่นยำน้อยกว่า
> Signature-based DPI ของ NBAR (อาจจับ Traffic อื่นที่ใช้ Port Range เดียวกันมาปนด้วย) — ควรใช้
> เป็นวิธี**ชั่วคราว**เท่านั้น และเปลี่ยนกลับไปใช้ `match protocol webex-meeting` แบบมาตรฐานทันที
> ที่ Protocol Pack ใหม่รองรับ Signature ของ Client เวอร์ชันปัจจุบันแล้ว

### 677.7 Verify

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VIDEO-CONFERENCE
        Class-map: VIDEO-CONFERENCE (match-any)
          188,200 packets, 120,448,000 bytes
          30 second offered rate 9,600,000 bps, drop rate 0 bps
```

Traffic Webex กลับเข้า Class `VIDEO-CONFERENCE` แล้ว — บทเรียนสำคัญ: **NBAR ไม่ใช่กลไกที่
"Config ครั้งเดียวจบตลอดไป"** ต้องมี Process ตรวจสอบ `show ip nbar protocol-discovery` เป็น
ประจำ (เชื่อมกับ [Part 40 Step 398](part-040-advanced-qos.md)) โดยเฉพาะหลัง Software/Firmware
Update ของ Application สำคัญที่มี Class QoS เฉพาะของตัวเอง

---

## Step 678 — สถานการณ์ 7: Policer Drop เกินคาดเพราะคำนวณ Burst Size (Bc) ผิด

### 678.1 บริบท: Policing Traffic Backup จาก Server1 ไม่ให้เกิน 8 Mbps

`Server1` (VLAN 30 SERVERS) รัน Backup Job ไป Data Center สำรองผ่าน Site-to-Site VPN บน
`WAN-EDGE-2` ทีมงาน Policing Traffic นี้ที่ Ingress ของ `DIST-SW3` ไม่ให้เกิน **CIR 8 Mbps**
เพื่อไม่ให้แย่ง Bandwidth จาก Business Traffic อื่น — แต่ Backup Application นี้มีลักษณะ Burst
สูงมาก (ส่งเป็นก้อนใหญ่ ~200 KB (1,600,000 bit) ทุกครั้งที่ Flush Buffer แล้วเงียบสนิทระหว่างรอ)
ไม่ใช่ Stream สม่ำเสมอแบบ Voice — วิศวกร Copy Config Policer จาก [Part 40 Step 396](part-040-advanced-qos.md)
(ตัวอย่าง Guest Wi-Fi 20 Mbps ที่ `bc 1000000`) มาปรับแค่ค่า CIR โดยลืมคำนวณ `bc` ใหม่ให้เหมาะกับ
Backup Application:

```
DIST-SW3(config)# policy-map BACKUP-POLICE
DIST-SW3(config-pmap)# class class-default
DIST-SW3(config-pmap-c)# police cir 8000000 bc 8000 conform-action transmit exceed-action drop
```

(`bc 8000` มาจากการหารเลขกลมๆ โดยไม่ได้คำนวณจริง — ไม่ใช่ค่าที่คัดลอกมาจาก Part 40 ตรงๆ
แต่เป็นค่าที่วิศวกร "เดา" เอาเองว่าน่าจะพอ)

### 678.2 อาการที่รายงานเข้ามา

Backup Job ที่ควรใช้เวลา 2 ชั่วโมงกลับใช้เวลานานกว่า 6 ชั่วโมง ทั้งที่ Throughput เฉลี่ยที่วัดได้
(`show policy-map interface` แสดง Offered Rate เฉลี่ย) อยู่ที่ **~7.2 Mbps เท่านั้น — ต่ำกว่า
CIR 8 Mbps ที่ตั้งไว้ด้วยซ้ำ** ทำให้ทีมงานงงว่าทำไม Policer ถึง Drop มากผิดปกติในเมื่อ Traffic
เฉลี่ยยังไม่ถึงเพดานที่อนุญาตไว้เลย

### 678.3 วินิจฉัย: อ่าน `police` Statistics ให้ละเอียดกว่าแค่ Offered Rate เฉลี่ย

```
DIST-SW3# show policy-map interface Vlan30 input

  Vlan30
  Service-policy input: BACKUP-POLICE
    Class-map: class-default (match-any)
      2,150,000 packets, 3,225,000,000 bytes
      30 second offered rate 7,200,000 bps, drop rate 2,850,000 bps
      Match: any
      police:
          cir 8,000,000 bps, bc 8,000 bytes
        conformed 1,380,000 packets, 2,070,000,000 bytes; actions:
          transmit
        exceeded 770,000 packets, 1,155,000,000 bytes; actions:
          drop
        conformed 4,350,000 bps, exceed 2,850,000 bps
```

**ความขัดแย้งที่ต้องอธิบาย**: Offered Rate เฉลี่ย (7.2 Mbps) ต่ำกว่า CIR (8 Mbps) แต่ `exceeded`
สูงถึง **2,850,000 bps (~40% ของ Traffic ทั้งหมด)** — นี่คือลายเซ็นชัดเจนของปัญหา **Burst Size
(Bc) เล็กเกินไปสำหรับลักษณะ Traffic ที่ Burst จริง**: Token Bucket ขนาดเล็ก (`bc 8,000 bytes`
= 64,000 bit) ไม่สามารถรองรับการส่งเป็นก้อนใหญ่ทีเดียว (200 KB ต่อครั้ง) ได้เลย — ก้อนแรกๆ
ของ Burst ผ่านได้ (เท่าที่ Token มีสะสมไว้) แต่พอ Token หมดภายในเสี้ยววินาที ส่วนที่เหลือของก้อน
เดียวกันถูก `exceed`/Drop ทันที **แม้ Traffic เฉลี่ยตลอดทั้งวินาทีจะยังต่ำกว่า CIR ก็ตาม**
เพราะ Policer ตัดสินที่ระดับ Token Bucket ไม่ใช่ค่าเฉลี่ยรายวินาที

### 678.4 คำนวณ Bc ที่ถูกต้องตามสูตร: Bc = CIR × Tc

```
สูตรจาก Part 40 Step 395: Tc (Committed Time Interval) = Bc ÷ CIR
                           →  Bc = CIR × Tc

ค่าที่ Config ไว้ (ผิด):
  Bc = 8,000 bytes = 64,000 bit
  Tc = Bc ÷ CIR = 64,000 bit ÷ 8,000,000 bps = 0.008 วินาที = 8 ms
  ความหมาย: Token Bucket เติมเต็มรอบใหม่ทุก 8 ms เท่านั้น — เก็บ Token ได้สูงสุดแค่ 8,000 byte
            ต่อรอบ ทั้งที่ Backup Job ส่งก้อนเดียวขนาด 200,000 byte (200 KB) ในครั้งเดียว

ค่าที่ควรเป็น (ให้ Bucket รองรับ Burst จริงของ Application ได้เต็มก้อนโดยไม่ Exceed):
  ต้องการ Bc >= ขนาด Burst สูงสุดของ Application = 200,000 byte = 1,600,000 bit
  Bc = 1,600,000 bit
  Tc = Bc ÷ CIR = 1,600,000 bit ÷ 8,000,000 bps = 0.2 วินาที = 200 ms
```

| ค่า | ที่ Config ไว้ (ผิด) | ที่ควรเป็น (ถูก) |
|---|---|---|
| CIR | 8,000,000 bps | 8,000,000 bps (ไม่เปลี่ยน — ตรงตาม Requirement) |
| Bc | 8,000 byte (64,000 bit) | **200,000 byte (1,600,000 bit)** |
| Tc ที่ได้ | 8 ms | 200 ms |
| ผลลัพธ์ | Burst เดียวของ Application ถูก Exceed เกือบทั้งก้อน | Burst เดียวเต็มก้อนผ่านได้ภายใน 1 Token Cycle |

### 678.5 วิธีแก้

```
DIST-SW3(config)# policy-map BACKUP-POLICE
DIST-SW3(config-pmap)# class class-default
DIST-SW3(config-pmap-c)# no police cir 8000000 bc 8000 conform-action transmit exceed-action drop
DIST-SW3(config-pmap-c)# police cir 8000000 bc 200000 conform-action transmit exceed-action drop
DIST-SW3(config-pmap-c)# exit
```

> **ข้อควรระวังเพิ่มเติม**: `Bc` ที่ใหญ่เกินไปก็มีข้อเสีย — มันอนุญาตให้ Burst ผ่านได้เร็วกว่า CIR
> เฉลี่ยในช่วงสั้นๆ มากขึ้น ซึ่งอาจไป Overrun Buffer ของอุปกรณ์ถัดไปได้ถ้าตั้งใหญ่เกินความจำเป็น
> จริง — หลักการคือตั้ง `Bc` ให้ **พอดีกับขนาด Burst สูงสุดตามลักษณะจริงของ Application** (ต้อง
> รู้ Traffic Pattern จริงก่อน Config เสมอ ไม่ใช่ Copy ค่าจาก Policy อื่นที่ออกแบบมาสำหรับ
> Traffic คนละลักษณะ เช่น Guest Wi-Fi ที่เป็น Traffic สม่ำเสมอกว่า Backup Job มาก)

### 678.6 Verify

```
DIST-SW3# show policy-map interface Vlan30 input | include exceeded|conformed
        conformed 2,148,900 packets, 3,223,350,000 bytes; actions:
        exceeded 1,100 packets, 1,650,000 bytes; actions:
        conformed 7,180,000 bps, exceed 3,200 bps
```

`exceed` ลดลงจาก 2.85 Mbps เหลือแค่ **3.2 kbps** (แทบไม่มีนัยสำคัญ) — Backup Job กลับมาใช้เวลา
ปกติที่ ~2 ชั่วโมง เพราะ Token Bucket รองรับ Burst ของ Application ได้เต็มก้อนแล้ว โดย CIR
เฉลี่ยยังคุมไว้ที่ 8 Mbps เหมือนเดิมไม่เปลี่ยนแปลง

---

## Step 679 — สถานการณ์ 8: QoS ทำงานบน Physical Interface แต่ไม่ทำงานบน Tunnel

### 679.1 บริบท: DMVPN Spoke ต้องการ Bandwidth การันตีสำหรับ Subnet วิกฤต

ต่อจาก [Part 53 Step 528](part-053-dmvpn-advanced.md) ที่ Config Hierarchical QoS ต่อ Spoke
ผ่าน NHRP Group บน `Tunnel0` ของ Hub (`WAN-EDGE-1`) ทีมงานต้องการเพิ่มความเจาะจงอีกขั้น: ให้
Traffic จาก Subnet วิกฤตของสาขา (คล้าย `SALES-CRITICAL` ใน [Part 40 Step 392](part-040-advanced-qos.md)
ที่ Match ด้วย `access-group` บน Inner IP Address) ได้รับ Bandwidth เพิ่มเป็นพิเศษเหนือ Class
อื่นในกลุ่มเดียวกัน — วิศวกรเพิ่ม Class-map ใหม่ที่ Match ด้วย ACL บน Inner Subnet ของสาขา
เข้าไปใน Child Policy-map ที่ใช้กับ `Tunnel0` ตาม Step 528

### 679.2 อาการที่รายงานเข้ามา

Class ใหม่ที่ Match ด้วย ACL (Inner Subnet ของ Spoke) **ไม่มี Traffic เข้าเลยตลอดเวลา** ทั้งที่
Subnet นั้นส่ง Traffic จริงต่อเนื่อง — แต่ประหลาดที่ Class อื่นในกลุ่มเดียวกันที่ Match ด้วย DSCP
(`match dscp ef`, `match dscp af31` เป็นต้น) **ยังทำงานได้ปกติดี** บนอุปกรณ์เดียวกัน Policy-map
เดียวกัน

### 679.3 วินิจฉัย: ตรวจสอบว่า IPsec เข้ามาเกี่ยวข้องตั้งแต่เมื่อไหร่

```
WAN-EDGE-1# show crypto ipsec sa peer 203.0.113.50 | include encaps|encrypt
    #pkts encaps: 8823012, #pkts encrypt: 8823012
```

ยืนยันว่า Traffic ของ DMVPN Tunnel นี้ถูก **IPsec เข้ารหัส (ESP)** อยู่จริง (ทีมงาน Enable IPsec
Profile บน `Tunnel0` เพิ่มเข้ามาไม่นานมานี้ตามนโยบาย Security ใหม่ — เชื่อมกับ [Part 51](part-051-vrf-lite-gre-ipsec.md))
— นี่คือจุดเปลี่ยนสำคัญที่ทำให้ Class-map แบบ ACL หยุดทำงาน

### 679.4 Root Cause: ACL/NBAR มองไม่เห็น Inner Header หลัง Encryption แต่ DSCP ยังมองเห็นได้

```
Packet ก่อนเข้า Tunnel0 (Plain):
┌──────────────┬──────────────┬────────┐
│ Inner IP Hdr  │ TCP/UDP Hdr   │ Data   │  ← ACL/NBAR อ่าน Src/Dst IP, Port ได้ที่นี่เท่านั้น
│ (Src/Dst จริง)│               │        │
└──────────────┴──────────────┴────────┘
        │ Encapsulate (GRE) + Encrypt (IPsec ESP)
        ▼
Packet บน Physical Interface ขาออกจริง (สิ่งที่ Policy-map บน Gi0/0/0 เห็น):
┌──────────┬──────────────┬───────────────────────────────┐
│ Outer IP  │ GRE Header    │ ESP[ Inner IP Hdr | TCP/UDP | Data ]  ← เข้ารหัสหมด │
│ Hdr (Hub↔ │               │ ACL/NBAR "มองไม่เห็น" ส่วนนี้เลย         │
│ Spoke NBMA)│              │ DSCP ของ Inner ถูก Copy ขึ้นมาไว้ที่     │
│           │               │ Outer IP Header ด้วย (Uniform Mode      │
│           │               │ Default) — จึงยังมองเห็น/Match ได้        │
└──────────┴──────────────┴───────────────────────────────┘
```

**คำอธิบาย**: DSCP Marking ถูก Copy จาก Inner IP Header ขึ้นไปยัง Outer IP Header โดย Default
(Uniform Mode ของ GRE/IPsec) ทำให้ Class-map ที่ Match ด้วย `dscp` ยังเห็นค่าที่ถูกต้องได้เสมอ
แม้ Payload จะถูกเข้ารหัสแล้วก็ตาม — แต่ **ACL ที่ Match ด้วย Source/Destination IP Address
หรือ Port Number (Layer 3-4 ของ Inner Packet) และ NBAR ที่ต้องทำ Deep Packet Inspection บน
Payload จริง ไม่มีทางมองทะลุผ่าน ESP Encryption ไปอ่าน Inner Header ได้เลย** — ตั้งแต่วินาทีที่
IPsec ถูก Enable บน Tunnel Interface ACL-based/NBAR-based Class-map ที่วางแผน Apply Policy
บน Physical Interface ที่ Traffic เข้ารหัสแล้ววิ่งผ่าน จะ**หยุดทำงานทันที**ไม่ว่า Class-map
จะเขียนถูกแค่ไหนก็ตาม

### 679.5 วิธีแก้: `qos pre-classify`

`qos pre-classify` บอกให้ Router **เก็บสำเนา Header ของ Packet ต้นฉบับ (ก่อน GRE Encapsulate
และก่อน IPsec Encrypt) ไว้ใช้ประกอบการตัดสินใจ QoS** แม้ว่า Packet ตัวจริงที่ส่งออกไปบนสายจะถูก
เข้ารหัสแล้วก็ตาม — Apply ที่ Tunnel Interface (จุดที่ยังเห็น Inner Header ก่อนเข้ารหัส):

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# qos pre-classify
WAN-EDGE-1(config-if)# exit
```

> **ทำไมต้อง Apply ที่ Tunnel Interface ไม่ใช่ Physical Interface**: `qos pre-classify` ต้อง
> อยู่ที่จุดที่ Router **ยังมี Access ถึง Inner Header อยู่** ซึ่งคือ Tunnel Interface (ก่อน
> ที่ Encapsulation/Encryption จะเกิดขึ้น) — ถ้า Apply ผิดที่ (เช่นที่ Physical Interface ที่
> Packet ผ่านไปแล้วในสภาพเข้ารหัสสมบูรณ์) จะไม่มีผลอะไรเลยเพราะ Header ต้นฉบับหายไปแล้วตั้งแต่
> จุดนั้น

### 679.6 Verify

```
WAN-EDGE-1# show policy-map interface Tunnel0 | section BRANCH-CRITICAL-SUBNET
        Class-map: BRANCH-CRITICAL-SUBNET (match-all)
          62,400 packets, 39,936,000 bytes
          30 second offered rate 820,000 bps, drop rate 0 bps
          Match: access-group name BRANCH1-CRITICAL
          bandwidth remaining percent 15
```

Class ที่ Match ด้วย ACL เริ่มมี Traffic เข้าแล้วหลัง Enable `qos pre-classify` — บทเรียนสำคัญ
ของ Scenario นี้: **DSCP-based Classification "รอด" จาก Encryption ได้เสมอเพราะ Copy ขึ้น Outer
Header อัตโนมัติ แต่ ACL-based และ NBAR-based Classification ต้องมี `qos pre-classify` เสมอ
เมื่อมี Encryption เข้ามาเกี่ยวข้อง (IPsec, DMVPN+IPsec, GET VPN)** — เป็นข้อสอบ ENARSI ที่ออก
บ่อยเรื่อง QoS บน VPN/Tunnel Environment

---

## Step 680 — Lab เต็มรูปแบบ: Voice Quality Audit เจอสองปัญหาซ้อนกัน

### 680.1 บริบทของ Lab: Network-Wide Voice Quality Audit

ฝ่ายบริหารสั่งให้ทีม Network ทำ **Voice Quality Audit ทั้งองค์กร** หลังได้รับ Complaint สะสม
จากหลายแผนกในช่วงเดือนที่ผ่านมา (คล้ายกับอาการใน Step 672 และ 676 แต่ยังไม่มีใครเชื่อมจุดทั้งสอง
เข้าด้วยกันว่าเป็นสองปัญหาคนละที่ที่เกิดขึ้นใกล้เคียงกัน) — ทีมงานใช้ Methodology จาก
[Step 671](#step-671--methodology-การ-troubleshoot-qos) เป็นกรอบการทำงาน: Baseline ก่อน,
สร้าง Load จำลอง, อ่าน Counter ที่จุดต่างๆ อย่างเป็นระบบจาก Access Layer ไปจนถึง WAN Edge

### 680.2 Topology ที่ใช้ตรวจสอบ

```
[IP Phone VLAN20]──┐
[PC VLAN10]─────────┤──► ACCESS-SW1 Gi1/0/1 ──► DIST-SW1 ──► CORE-SW1 ──┐
                                                                          │ 10.10.254.1/30
                                                                    ┌─────┴─────┐
                                                                    │ WAN-EDGE-1 │──► ISP-RTR (203.0.113.0/30)
                                                                    │  Gi0/0/0    │    CIR สัญญาปัจจุบัน = 20 Mbps
                                                                    └───────────┘
        จุดที่ Audit ต้องตรวจตามลำดับ:
        1) Trust Boundary ที่ ACCESS-SW1/ACCESS-SW2 (mls qos trust)
        2) Class-map/Policy-map ที่ WAN-EDGE-1 (First-Match-Wins, WRED)
        3) Parent Shaper ของ HQoS ที่ WAN-EDGE-1 (หน่วย bps ถูกหรือไม่)
```

### 680.3 ขั้นที่ 1 — Baseline: สร้าง Load จำลองและสังเกตอาการเบื้องต้น

ทีมงานเริ่ม Backup Job ขนาดใหญ่จาก `Server1` ไปยัง `Server2` พร้อมกับให้ทีม Sales (VLAN 10)
เปิด Video Streaming Bandwidth สูงหลายเครื่องพร้อมกัน (จำลอง Congestion ให้เข้าใกล้ CIR จริง)
แล้วเริ่มสาย Voice Call ทดสอบจาก IP Phone ที่ต่อกับ `ACCESS-SW1`:

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE
        Class-map: VOICE (match-any)
          420 packets, 26,880 bytes
          30 second offered rate 2,100 bps, drop rate 0 bps
          Priority: 10% (2,000 kbps), burst bytes 50,000
```

**สังเกตแรก**: Offered Rate ของ Class VOICE ต่ำผิดปกติ (2,100 bps ทั้งที่ควรมีหลายสายจริง) —
สัญญาณเดียวกับ Step 672 ทันที ทีมงานจึงเริ่ม Checklist จาก Access Layer ก่อนตาม Methodology

### 680.4 ขั้นที่ 2 — พบปัญหาที่ 1: Trust Boundary หายที่ ACCESS-SW1

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: not trusted
```

เหมือนกับ Step 672 เป๊ะ — ตรวจ Log พบว่า `ACCESS-SW1` ผ่านการ RMA เมื่อ 3 สัปดาห์ก่อนและ Golden
Config Template ยังไม่ได้อัปเดต แก้ไขทันที:

```
ACCESS-SW1(config)# mls qos
ACCESS-SW1(config)# interface GigabitEthernet1/0/1
ACCESS-SW1(config-if)# mls qos trust device cisco-phone
ACCESS-SW1(config-if)# mls qos trust cos
ACCESS-SW1(config-if)# auto qos voip cisco-phone
ACCESS-SW1(config-if)# exit
```

**Verify รอบที่ 1 (ยังมี Load เดิมอยู่)**:

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE
        Class-map: VOICE (match-any)
          188,200 packets, 12,044,800 bytes
          30 second offered rate 452,000 bps, drop rate 0 bps
          Priority: 10% (2,000 kbps), burst bytes 50,000
```

Offered Rate ของ VOICE เพิ่มขึ้นเป็น 452,000 bps ตามที่คาด (ปัญหาที่ 1 แก้แล้ว) **แต่ผู้ทดสอบสาย
Voice ที่โทรจริงยังรายงานว่าเสียงยังไม่สมบูรณ์ 100% — มีสะดุดเบาๆ เป็นบางครั้ง** ทีมงานจึงรู้ว่า
ยังมีปัญหาที่สองซ่อนอยู่ (นี่คือเหตุผลที่ Audit ระดับองค์กรสำคัญ — ถ้าหยุดที่ปัญหาแรกแล้วปิดเคส
จะพลาดปัญหาที่สองไปเลย)

### 680.5 ขั้นที่ 3 — พบปัญหาที่ 2: ตรวจ Priority Bandwidth ที่คำนวณได้จริง

สังเกตจากผลลัพธ์ข้างบนอีกครั้ง: `Priority: 10% (2,000 kbps)` — ทีมงานเทียบกับเอกสารสัญญา WAN
ล่าสุดที่ฝ่ายจัดซื้อส่งมา: **CIR ปัจจุบันคือ 20 Mbps** (มีการ Downgrade จาก 50 Mbps ไปเมื่อ
เดือนก่อนตามที่ Step 676 อธิบาย) — คำนวณว่า 10% ของ 20 Mbps ควรเท่ากับ **2 Mbps (2,000 kbps)**
ซึ่ง**ตรงกับตัวเลขที่เห็นพอดี** ทำให้ในตอนแรกดูเหมือนไม่มีปัญหา — แต่ทีมงานตรวจสอบเชิงลึกต่อด้วย
การดู `target shape rate` ตรงๆ เพื่อความมั่นใจ:

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | include target shape rate
      target shape rate 20,000,000
```

**ผลลัพธ์นี้ถูกต้องแล้ว** (20,000,000 bps = 20 Mbps ตรงกับสัญญา) — ทำให้ทีมงานตัดปัญหา Shaper
หน่วยผิดออกจาก WAN-EDGE-1 ได้ แล้วขยายการตรวจไปที่ **WAN-EDGE-2 (เส้นทาง Backup)** เพราะระหว่าง
การทดสอบมี Traffic บางส่วน Failover ไปทาง WAN-EDGE-2 ชั่วครู่ (ตาม BGP Path Selection จาก
[Part 58](part-058-bgp-troubleshooting.md)):

```
WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0 | include target shape rate|Priority
      target shape rate 20,000
        Priority: 10% (2 kbps), burst bytes 250
```

**พบปัญหาที่ 2 ทันที** — `target shape rate 20,000` (bps) บน `WAN-EDGE-2` ผิดไป 1,000 เท่า
เหมือนสถานการณ์ Step 676 ทุกประการ (Router ตัวนี้ถูก Config แยกจาก `WAN-EDGE-1` โดยคนละคนละ
Session ตอนทำ Downgrade CIR เดือนก่อน แล้วเกิดความสับสนหน่วยแบบเดียวกันซ้ำอีกที)

### 680.6 แก้ไขปัญหาที่ 2 บน WAN-EDGE-2

```
WAN-EDGE-2(config)# policy-map WAN-SHAPE-50M
WAN-EDGE-2(config-pmap)# class class-default
WAN-EDGE-2(config-pmap-c)# no shape average 20000
WAN-EDGE-2(config-pmap-c)# shape average 20000000
WAN-EDGE-2(config-pmap-c)# exit
WAN-EDGE-2(config-pmap)# exit
```

### 680.7 Verify สุดท้าย — ทั้งสองปัญหาพร้อมกันภายใต้ Load จำลองต่อเนื่อง

ทีมงานรักษาสภาวะ Load เดิมไว้ (Backup Job + Video Streaming ทดสอบ) และบังคับสลับ Path ไปมา
ระหว่าง `WAN-EDGE-1`/`WAN-EDGE-2` เพื่อทดสอบทั้งสองเส้นทางพร้อมกัน:

```
WAN-EDGE-1# show policy-map interface GigabitEthernet0/0/0 | section VOICE
        Class-map: VOICE (match-any)
          452,900 packets, 28,985,600 bytes
          30 second offered rate 458,000 bps, drop rate 0 bps
          Priority: 10% (2,000 kbps), burst bytes 50,000

WAN-EDGE-2# show policy-map interface GigabitEthernet0/0/0 | section VOICE
        Class-map: VOICE (match-any)
          448,100 packets, 28,678,400 bytes
          30 second offered rate 455,000 bps, drop rate 0 bps
          Priority: 10% (2,000 kbps), burst bytes 50,000

ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1 | include trust state
trust state: trust cos
```

ทั้งสองเส้นทางแสดง `drop rate 0 bps` สำหรับ VOICE พร้อมกัน และ Trust Boundary ที่ Access Layer
ทำงานถูกต้องแล้ว — ทีมงานให้ผู้ทดสอบโทรผ่านทั้งสอง Path ซ้ำอีกครั้งและยืนยันว่าเสียงชัดเจนสมบูรณ์
ไม่มีสะดุดแม้ระหว่างที่ Backup Job และ Video Streaming ยังทำงานเต็มที่อยู่

### 680.8 สรุปผล Audit

| ปัญหา | ตำแหน่งที่พบ | Root Cause | วิธีแก้ | ผลยืนยัน |
|---|---|---|---|---|
| 1. Trust Boundary หาย | `ACCESS-SW1` Gi1/0/1 | RMA Restore จาก Golden Config เก่ากว่า Part 24 Lab (Config Drift) | `mls qos trust device cisco-phone` + อัปเดต Golden Config Template | Offered Rate ของ VOICE ที่ WAN-EDGE-1 เพิ่มจาก 2,100 bps เป็น 452,000 bps |
| 2. Parent Shaper ผิดหน่วย | `WAN-EDGE-2` Gi0/0/0 | สับสนหน่วย `bandwidth` (kbps) กับ `shape average` (bps) ตอน Downgrade CIR | `shape average 20000000` (แทน `20000`) | `target shape rate` ถูกต้อง (20,000,000), `Priority: 10% (2,000 kbps)`, `drop rate 0 bps` |

**บทเรียนสำคัญที่สุดของ Lab นี้**: ปัญหาทั้งสองมีอาการปลายทางคล้ายกัน (เสียงแย่ตอน Congestion)
และเกิดขึ้นในช่วงเวลาใกล้เคียงกันโดยไม่เกี่ยวข้องกันเลย (RMA Hardware กับ Contract Renegotiation)
— การหยุดตรวจสอบทันทีที่เจอปัญหาแรกและ "ปิดเคส" จะทำให้พลาดปัญหาที่สองไปเลย เพราะปัญหาที่สอง
อยู่คนละอุปกรณ์ (`WAN-EDGE-2` ซึ่งไม่ใช่ Path หลักที่ Traffic ส่วนใหญ่วิ่งผ่าน) — Audit ที่เป็น
ระบบ (ตรวจทุกจุดตาม Checklist Step 671 แม้จะเจอปัญหาแรกไปแล้ว) คือสิ่งที่ทำให้เจอปัญหาที่สองได้
ก่อนที่มันจะกลายเป็น Complaint รอบใหม่ตอน Failover ไปทาง WAN-EDGE-2 จริงในอนาคต

---

## แบบฝึกหัดทวนความเข้าใจ Part 68

1. เพราะเหตุใดการดู `show policy-map interface` ตอนเครือข่ายไม่มี Congestion เลยจึง **ไม่**
   เพียงพอต่อการยืนยันว่า QoS Policy ทำงานถูกต้อง และควร Verify อย่างไรให้ได้ผลลัพธ์ที่เชื่อถือได้?

2. ใน Step 672 เพราะเหตุใด Voice Traffic ที่ไม่ผ่าน Trust Boundary จึงไปโดน Drop หนักที่สุด
   ที่ `class-default` แทนที่จะแค่ "ไม่ได้ Priority พิเศษ" เฉยๆ?

3. อธิบายกฎ First-Match-Wins ใน MQC (Step 674) และวิธีป้องกันปัญหานี้เวลาต้องเพิ่ม Class ใหม่
   เข้าไปใน Policy-map ที่มี Class เดิมอยู่แล้ว?

4. ใน Step 676 ทำไมความผิดพลาดเรื่องหน่วย (`shape average 20000` แทน `20000000`) จึงกระทบ
   Class VOICE (LLQ) รุนแรงกว่า Class อื่นที่ใช้ `bandwidth percent` ธรรมดา?

5. เพราะเหตุใด DSCP-based Class-map ยังทำงานได้ปกติบน DMVPN Tunnel ที่เข้ารหัสด้วย IPsec
   (Step 679) แต่ ACL-based และ NBAR-based Class-map กลับต้องพึ่ง `qos pre-classify`?

**เฉลย:**

1. เพราะ QoS Mechanism ส่วนใหญ่ (Queuing, WRED, Policing, Shaping) มีผลเฉพาะตอนที่ Bandwidth
   ไม่พอสำหรับทุก Traffic เท่านั้น — ตอนไม่มี Congestion ทุก Packet ผ่านได้หมดไม่ว่า Policy จะ
   ถูกหรือผิด ทำให้ Counter ทุกตัวดู "ปกติดี" เท่ากันทั้งสองกรณี ต้อง Verify ด้วยการสร้าง Load
   จำลองให้ใกล้เคียงเพดานจริง (Bulk Transfer + Voice Call จริง) แล้วอ่าน `show policy-map
   interface` ระหว่างมี Load นั้น เปรียบเทียบ Behavior ของแต่ละ Class กับ Baseline

2. เพราะ `class-default` ในตัวอย่างของหลักสูตรนี้ถูกออกแบบให้ `fair-queue` + `random-detect`
   สำหรับ Best-Effort Data โดยเฉพาะ — เมื่อ Voice Packet ไม่ได้ถูก Mark DSCP EF มาแล้ว (เพราะ
   Trust Boundary ไม่ทำงาน มัน Reset เป็น 0/Best-Effort) มันจะไม่ Match Class VOICE (`match
   dscp ef`) เลย จึงตกไปที่ `class-default` โดยอัตโนมัติ (Catch-all ของทุก Class-map) ซึ่งเป็น
   Class ที่ถูกออกแบบให้รับ Drop หนักที่สุดตอน Congestion ตามเจตนาเดิม (ปกป้อง Class อื่น) — ผล
   คือ Voice ถูกปฏิบัติเหมือน Traffic ที่สำคัญน้อยที่สุดในระบบทั้งที่ควรได้รับความสำคัญสูงสุด

3. IOS ประมวลผล Class ใน Policy-map ตามลำดับที่พิมพ์จากบนลงล่าง และหยุดที่ Class แรกที่ Match
   เงื่อนไข (`class-default` ถูกย้ายไปท้ายสุดเสมอโดยอัตโนมัติ) — ถ้า Class ที่อยู่บนมีเงื่อนไข
   `match-any` ที่กว้างครอบคลุมเงื่อนไขของ Class ที่อยู่ล่างด้วย Traffic จะไม่มีวันไปถึง Class
   ล่างเลย วิธีป้องกัน: ก่อนเพิ่ม Class ใหม่ ต้องตรวจ `show class-map` ของทุก Class ที่มีอยู่แล้ว
   ว่ามีเงื่อนไขทับซ้อนหรือไม่ และวางลำดับ Class ที่เจาะจงกว่าไว้เหนือ Class ที่กว้างกว่าเสมอ
   (ลบเงื่อนไขที่ทับซ้อนออกจาก Class เดิมด้วยถ้าจำเป็น)

4. เพราะ LLQ (`priority percent`) คำนวณเพดาน Policer ในตัวเป็นเปอร์เซ็นต์ **ของกรอบที่ Parent
   Shaper กำหนดไว้** (ตาม HQoS Part 40 Step 397) — ถ้ากรอบนั้นผิดไป 1,000 เท่า (20,000 bps แทน
   20,000,000 bps) เพดานของ Voice ก็ผิดไป 1,000 เท่าตามกัน (2 kbps แทน 2,000 kbps) ทำให้แม้แต่
   สายเดียวก็เกินเพดานและถูก Policer ภายในของ LLQ ทิ้งทันที ในขณะที่ Class ที่ใช้ `bandwidth
   percent` ธรรมดาไม่มี Policer เข้มงวดแบบนี้ในตัว จึงยัง "พอส่งได้" อยู่บ้างแม้ตัวเลขจะผิดเหมือนกัน
   (แค่ได้ Fair Share ที่น้อยกว่าที่ควร ไม่ถึงกับถูกทิ้งทันที)

5. เพราะ DSCP อยู่ใน IP Header และ Router จะ Copy ค่า DSCP จาก Inner IP Header ขึ้นไปไว้ที่ Outer
   IP Header โดย Default (Uniform Mode) ทำให้ Class-map ที่ Match ด้วย DSCP ยังเห็นค่าที่ถูกต้อง
   ได้แม้ Payload จะถูก IPsec เข้ารหัสไปแล้ว — แต่ ACL ที่ Match ด้วย Source/Destination IP หรือ
   Port ของ Inner Packet และ NBAR ที่ต้องทำ Deep Packet Inspection ต้องอ่าน Header/Payload ของ
   Inner Packet ตรงๆ ซึ่งถูกซ่อนอยู่ใน ESP Encryption หมดแล้ว ไม่มีทางมองเห็นได้เลยจากภายนอก
   ต้องใช้ `qos pre-classify` ที่ Tunnel Interface เพื่อให้ Router เก็บข้อมูล Header ต้นฉบับไว้
   ใช้ตัดสินใจ QoS ก่อนที่ Encryption จะเกิดขึ้น

---

## สรุป Part 68

Part นี้นำ QoS ที่ [Part 24](part-024-qos-fundamentals.md)/[Part 40](part-040-advanced-qos.md)
วางพื้นฐานไว้เต็มรูปแบบมา Troubleshoot ในรูปแบบสถานการณ์จริงระดับ ENARSI ครบ 8 Scenario:
**Trust Boundary หายหลัง RMA** (Step 672), **`service-policy` ผิดทิศทาง** (Step 673),
**First-Match-Wins ทำให้ Class-map จับ Traffic ผิด** (Step 674), **WRED ทำลาย Voice เพราะ Config
ผิด Class** (Step 675), **HQoS Parent Shaper ผิดหน่วยหลัง Downgrade CIR** (Step 676), **NBAR
จำ Application ไม่ได้หลัง Software Update** (Step 677), **Policer Drop เกินเพราะคำนวณ Bc ผิด**
(Step 678), และ **QoS พังบน Encrypted Tunnel ที่ต้องใช้ `qos pre-classify`** (Step 679) — ทุก
Scenario ยึด Methodology เดียวกันจาก Step 671: **ปัญหา QoS ส่วนใหญ่ซ่อนอยู่จนกว่าจะมี Congestion
จริง ต้อง Load-Test เพื่อเผยมันออกมา** และปิดท้ายด้วย Lab เต็มรูปแบบ (Step 680) ที่จำลอง Voice
Quality Audit ทั้งองค์กรซึ่งเจอสองปัญหาที่ไม่เกี่ยวข้องกันซ้อนอยู่พร้อมกัน — ตอกย้ำว่า Audit ที่
เป็นระบบและตรวจครบทุกจุดตาม Checklist สำคัญกว่าการหยุดที่ปัญหาแรกที่เจอเสมอ

✅ **พร้อมสำหรับ Part 69**: เราจะเข้าสู่ **Mega Lab 1** — Lab รวมทุกหัวข้อ Troubleshooting ที่
เรียนมาตั้งแต่ [Part 56](part-056-eigrp-troubleshooting.md) จนถึง Part นี้ (Routing Protocol,
Redistribution, NAT, DMVPN/VPN, Infrastructure Security/Services, และ QoS) เข้าเป็น Lab ขนาดใหญ่
เดียวที่มีปัญหาซ้อนกันหลายชั้นพร้อมกันบน Topology เต็มรูปแบบของหลักสูตร เพื่อฝึกการวินิจฉัยปัญหา
ที่ซับซ้อนแบบที่พบได้จริงในสถานการณ์ทำงานระดับ ENARSI

**ไปต่อ:** [Part 69 — Mega Lab 1 →](part-069-mega-lab-1.md)
