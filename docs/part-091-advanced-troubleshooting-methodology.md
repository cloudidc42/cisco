# Part 91 — Advanced Troubleshooting Methodology
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 901–910 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 90 — Catalyst 9000 Advanced Features](part-090-catalyst-9000-advanced-features.md)
> ที่ปิดท้ายเนื้อหาเชิง Feature/Platform เฉพาะทางแล้ว **Part นี้เปลี่ยนแนวทางแบบสิ้นเชิง** — Part
> 56-69 สอน Troubleshooting ไปแล้วทีละโดเมน (EIGRP, OSPF, BGP, Redistribution, DMVPN/VPN, Infra
> Security, Infra Services, NAT, Path Control/PBR, Multicast, Wireless, Automation, QoS) และ Part
> 69 เอาทุกโดเมนมาชนกันใน Mega-Lab แบบ Major Incident เต็มรูปแบบ แต่ทั้งหมดนั้นยังคงเป็นการฝึก
> **"รู้โดเมนที่ต้องดู แล้วไล่ตาม Checklist ของโดเมนนั้น"** (เช่น Decision Tree เฉพาะ EIGRP ใน
> [Part 56 Step 551](part-056-eigrp-troubleshooting.md) หรือ Ticket ที่ระบุอาการมาชัดว่าเป็นเรื่อง
> BGP/QoS/Wireless อยู่แล้ว)
>
> **CCIE Diagnose Module (ดู [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md)) ไม่ใจดี
> ขนาดนั้น** — โจทย์จะให้แค่อาการที่ผู้ใช้รายงาน ไม่บอกโดเมนให้ และบางครั้งโดเมนที่แท้จริงก็เป็นสิ่ง
> ที่ไม่มีใครสอนคุณมาก่อนตรงๆ ด้วยซ้ำ สิ่งที่แยกวิศวกรระดับ CCIE ออกจากวิศวกรที่ท่องจำ Checklist
> เก่งคือ **วิธีคิด (Methodology)** ที่ใช้ได้กับปัญหาชนิดใดก็ได้ ไม่ใช่แค่โดเมนที่เคยเจอมาก่อน — Part
> นี้จึงไม่มี Scenario ผูกกับ Technology ใดโดยเฉพาะ แต่สอน **กระบวนการคิดที่อยู่เบื้องหลัง** ทุก Part
> ที่ผ่านมา: เมื่อไหร่ควรเริ่มที่ Layer ไหน, วิธี Bisect ปัญหาให้เจอเร็วที่สุด, วิธีตั้งสมมติฐานแบบ
> วิทยาศาสตร์, การเทียบ Baseline แทนการไล่คิดจากศูนย์, การแปลงคำร้องเรียนคลุมเครือให้เป็นโจทย์ที่
> ทดสอบได้, การใช้ Packet Capture อย่างมีเป้าหมาย, การรู้ขอบเขตว่าใช่ปัญหา Network จริงหรือไม่, วินัย
> เรื่องเวลาและการขอความช่วยเหลือ, และการจดบันทึกระหว่างทาง — ปิดท้ายด้วย Lab ที่ให้แค่ประโยคเดียว
> "ผู้ใช้บอกว่าแอปพลิเคชันใหม่ใช้งานไม่ได้" แล้วให้ผู้เรียนไล่หา Root Cause เองทั้งหมดด้วยกระบวนการ
> จาก Step 901-909

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 901 | OSI Layer-by-Layer ทวนใหม่ระดับ Expert — เมื่อไหร่ควร "ข้าม" Layer แทนไล่จาก Layer 1 เสมอ |
| 902 | Divide-and-Conquer / Bisection Troubleshooting — Test ที่จุดกึ่งกลางของ Path ก่อนปลายทั้งสองฝั่ง |
| 903 | Scientific Method ในงาน Network — Hypothesis → Test → Observe → Refine และการเลี่ยง Shotgun Troubleshooting |
| 904 | Baseline Comparison — "อะไรเปลี่ยนไป" คือทางลัดที่เร็วที่สุดสู่ Root Cause |
| 905 | อ่านนัยของคำร้องเรียนที่คลุมเครือ — แปลง "เน็ตช้า"/"ใช้งานไม่ได้" เป็นสมมติฐานที่ทดสอบได้ |
| 906 | ใช้ Packet Capture อย่างมีเป้าหมาย — SPAN/RSPAN/ERSPAN และการอ่านหา Signal ที่ต้องการ |
| 907 | Multi-Vendor/Multi-Domain Triage — พิสูจน์ว่าใช่ปัญหา Network Infrastructure จริงหรือไม่ |
| 908 | Time-Boxing และวินัยการ Escalate — รู้ว่าเมื่อไหร่ควรขอความช่วยเหลือหรือพักปัญหาไว้ก่อน |
| 909 | บันทึกระหว่างการวินิจฉัย — เขียนสิ่งที่ลองแล้วและตัดออกแล้ว "ไปพร้อมกับที่ทำ" ไม่ใช่ทำเสร็จแล้วเขียน |
| 910 | Full Lab — โจทย์คลุมเครือ "แอปพลิเคชันใหม่ใช้งานไม่ได้" ไล่ทั้งกระบวนการจาก Step 901-909 |

---

## Step 901 — OSI Layer-by-Layer ทวนใหม่ระดับ Expert

### 901.1 ความเข้าใจผิดที่ฝังแน่นที่สุดของคนเริ่มต้น Troubleshooting

ตำราเรียน Networking ทุกเล่ม (รวมถึงหลักสูตรนี้ตั้งแต่ [Part 1 Step 2](part-001-networking-fundamentals.md))
สอนว่า **"เวลาแก้ปัญหาให้ไล่จาก Layer 1 ขึ้นไป"** (Bottom-Up) เพราะ Layer ล่างเป็นเงื่อนไขจำเป็นของ
Layer บน — ถ้าไม่มีไฟที่พอร์ต ก็ไม่มีทางมี IP Routing ทำงานได้เลย นี่คือ**หลักการที่ถูกต้อง 100%**
สำหรับผู้เริ่มต้น เพราะมันไม่พลาดอะไรไปแน่ๆ

แต่ในทางปฏิบัติจริงของวิศวกรระดับ CCIE **ไม่มีใครไล่จาก Layer 1 ทุกครั้ง** — เหตุผลไม่ใช่ว่า
Bottom-Up ผิด แต่เพราะ **อาการ (Symptom) ที่ได้รับมาเองมักบอกอยู่แล้วว่า Layer ล่างต้องผ่านมาแล้ว**
การเสียเวลาไล่เช็คสิ่งที่ Evidence บอกอยู่แล้วว่าปกติ คือการสูญเวลาที่ไม่จำเป็นที่สุดในห้องสอบ CCIE
Diagnose Module ที่มีเวลาจำกัดต่อ Ticket

### 901.2 หลักการ Pattern-Matching: อาการบอก Layer ที่ควรเริ่ม

| อาการที่ได้รับมา | Layer ที่ Evidence ยืนยันว่า "ผ่านแล้วแน่นอน" | Layer ที่ควรเริ่มสงสัยก่อน | เหตุผล |
|---|---|---|---|
| "SSH เข้าไปที่ Router ได้ปกติ แต่ Routing ไม่เห็นปลายทาง" | L1-L4 (TCP 22 สมบูรณ์ทั้ง Handshake) | L3 (Routing Table/Protocol) | ถ้า L1-L4 มีปัญหา SSH เข้าไม่ได้เลย |
| "Ping ผ่าน แต่ Web Browse ไม่ได้" | L1-L3 (ICMP ไปกลับสำเร็จ) | L4-L7 (Port/ACL/Application) | ICMP สำเร็จพิสูจน์ Routing สมบูรณ์ทั้ง 2 ทาง |
| "โทรศัพท์ VoIP มีเสียงแต่ก้อง/แตก" | L1-L3 (Traffic วิ่งถึงกันจริง ไม่ Drop สนิท) | L2 QoS/Trust Boundary, Jitter/Queueing | Part 69 Step 684 คือตัวอย่างจริงของ Pattern นี้ |
| "Interface ทั้งคู่ show ว่า up/up แต่ Traffic ไม่วิ่งเลย" | L1-L2 (Physical + Framing ปกติ) | L3 (IP/Routing/ACL) | up/up ยืนยัน Physical/Data Link เรียบร้อยแล้ว |
| "ใช้งานได้ปกติมาตลอด จู่ๆ ช้าตอน Peak Hour เท่านั้น" | L1-L3 (ไม่มี Total Failure) | L1 เชิง Capacity (Bandwidth/Congestion), QoS | รูปแบบตามเวลาชี้ Capacity ไม่ใช่ Config ผิด |
| "ทุกอย่างเพิ่งเสียหลัง Maintenance Window คืนที่แล้ว" | ไม่มี Layer ใดยืนยันได้ล่วงหน้า | **เริ่มที่ Baseline Comparison (Step 904) ก่อน Layer ใดๆ** | "อะไรเปลี่ยน" สำคัญกว่า Layer ในกรณีนี้ |

### 901.3 กฎการอ่าน "Up/Up" ให้ถูกต้อง — สัญญาณที่ถูกมองข้ามบ่อยที่สุด

```
Router# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/1     10.10.10.1      YES NVRAM  up                    up
```

**`up/up`** (Status/Protocol) หมายความว่า **Layer 1 (Physical) และ Layer 2 (Data Link/Keepalive)
ผ่านแล้วทั้งคู่แน่นอน** — วิศวกรมือใหม่จำนวนมากยังคงรัน `show interfaces` ไล่ดู Duplex/Error
Counter ซ้ำอีกรอบทั้งที่ผลลัพธ์นี้ตอบคำถามนั้นไปแล้ว ระดับ Expert จะข้ามไปที่ Layer 3 ทันทีเมื่อเห็น
`up/up` เว้นแต่ Symptom เป็นแบบ **Intermittent** (Flapping เป็นระยะ ซึ่ง `up/up` ณ เวลาที่เช็คไม่ได้
แปลว่าไม่เคย Down มาก่อน — ต้องเช็ค `show logging | include UPDOWN` ควบคู่เสมอในกรณีนี้)

### 901.4 หลักการ "ตั้งสมมติฐานก่อนเลือก Layer" — สิ่งที่ต้องทำก่อนพิมพ์คำสั่งแรก

ระดับ Expert จะไม่พิมพ์ `show` อะไรเลยจนกว่าจะตอบคำถามนี้ในใจก่อน:

1. **จาก Evidence ที่มีอยู่ตอนนี้ Layer ไหนพิสูจน์ตัวเองแล้วว่าไม่ใช่ปัญหา?**
2. **Layer ที่เหลือ Layer ไหนมีโอกาสสูงสุดที่จะอธิบาย Symptom ทั้งหมดได้ (ไม่ใช่แค่บางส่วน)?**
3. **คำสั่งเดียวที่จะยืนยันหรือหักล้างสมมติฐานนั้นได้เร็วที่สุดคืออะไร?**

> **ข้อเตือนสำคัญ**: การ "ข้าม Layer" ต่างจาก "เดา" — ต้องมี Evidence รองรับการข้ามเสมอ (เช่น
> `up/up` รองรับการข้าม L1/L2, Ping สำเร็จรองรับการข้าม L3) **ห้ามข้าม Layer เพราะความรู้สึกหรือ
> ประสบการณ์เก่าที่ไม่เกี่ยวกับ Evidence ตรงหน้า** — นี่คือเส้นแบ่งระหว่าง "Expert Pattern-Matching"
> กับ "Assumption ที่อันตราย" ซึ่ง Step 903 จะขยายความต่อ

### 901.5 ตารางเปรียบเทียบวิธี Troubleshooting แบบต่างๆ ที่ควรมีติดตัว

| วิธี | หลักการ | ใช้เมื่อไหร่ดีที่สุด | ตัวอย่างใน Part นี้ |
|---|---|---|---|
| **Bottom-Up** | ไล่จาก L1 ขึ้นไป | เพิ่งเริ่มงาน/ไม่มี Evidence เลย, Total Failure ที่ไม่รู้ขอบเขต | Baseline สำหรับผู้เริ่มต้น |
| **Top-Down** | ไล่จาก L7 ลงมา | สงสัย Application เป็นต้นเหตุ (User รายงานเจาะจงที่ Feature หนึ่ง) | Step 907 |
| **Divide-and-Conquer** | Test กึ่งกลาง Path | Total Failure แบบ End-to-End ที่ Path ยาว/ผ่านหลายอุปกรณ์ | Step 902 |
| **Follow-the-Path** | ไล่ตาม Physical/Logical Path ทีละ Hop | เมื่อรู้ Path ชัดแล้วแค่ต้องหาจุด Drop | Part 56-69 หลายจุด |
| **Pattern-Matching (Expert)** | จับคู่ Symptom Signature กับ Root Cause ที่เคยเจอ | มีประสบการณ์สะสมมากพอ, ต้องมี Evidence ยืนยันก่อนข้าม | Step 901.2 |

ทั้ง 5 วิธีนี้**ไม่แข่งกัน** — Expert ตัวจริงใช้ Pattern-Matching เพื่อ**เลือกจุดเริ่ม** แล้วอาจสลับไปใช้
Divide-and-Conquer (Step 902) เพื่อไล่ต่อ และปิดท้ายด้วย Follow-the-Path เพื่อยืนยัน Root Cause
สุดท้าย — นี่คือความหมายของ "Methodology" ในชื่อ Part นี้: **รู้ว่าจะสลับใช้เครื่องมือไหนเมื่อไหร่**

---

## Step 902 — Divide-and-Conquer / Bisection Troubleshooting

### 902.1 ปัญหาของการไล่ทีละ Hop (Linear Search)

สถานการณ์: **PC1 (Sales, VLAN 10, ผ่าน ACCESS-SW1) ไม่สามารถเข้าถึง Server1 (Servers, VLAN 30,
ผ่าน ACCESS-SW3) ได้เลย — Ping ไม่ผ่าน 0%** Path จริงตาม Topology หลักของหลักสูตร
([Part 1 Step 10](part-001-networking-fundamentals.md)) มี 7 Hop:

```
PC1 → ACCESS-SW1 → DIST-SW1 → CORE-SW1 → CORE-SW2 → DIST-SW3 → ACCESS-SW3 → Server1
       (Hop 1)      (Hop 2)    (Hop 3)     (Hop 4)     (Hop 5)     (Hop 6)   (Hop 7)
```

วิธี **Linear/Follow-the-Path** (ไล่ Login เข้าไปทีละอุปกรณ์จาก PC1 ไปทาง Server1 ทีละ Hop) ในกรณี
เลวร้ายที่สุดต้องทำถึง **7 ครั้ง** กว่าจะเจอจุดที่ Drop — ถ้าแต่ละ Hop ใช้เวลา Login+ตรวจ 2-3 นาที
นี่คือ 15-20 นาทีที่เสียไปโดยไม่จำเป็นในสถานการณ์ที่ CCIE Diagnose Module ให้เวลาต่อ Ticket จำกัดมาก

### 902.2 หลักการ Bisection — ทำไม Test ที่กึ่งกลางถึงเร็วกว่าเสมอ

**หลักการทางคณิตศาสตร์**: ถ้า Path มี N จุดที่อาจเป็นต้นเหตุ, การ Test แบบ Linear ต้องการสูงสุด `N`
ครั้ง แต่การ Test แบบ Binary Search (Bisection) ต้องการเพียง `⌈log₂(N)⌉` ครั้งเท่านั้น เพราะ **การ
Test 1 ครั้งที่จุดกึ่งกลางจะตัดครึ่งหนึ่งของความเป็นไปได้ทั้งหมดออกไปในทีเดียว** ไม่ว่าผลจะเป็นสำเร็จ
หรือล้มเหลว

| จำนวน Hop ที่ต้องสงสัย (N) | Linear Search (เลวร้ายสุด) | Bisection (⌈log₂ N⌉) |
|---|---|---|
| 2 | 2 | 1 |
| 4 | 4 | 2 |
| 7 (กรณีนี้) | 7 | 3 |
| 15 | 15 | 4 |
| 31 | 31 | 5 |

ยิ่ง Path ยาว (Enterprise WAN ข้าม Site, DMVPN หลาย Hop) ส่วนต่างของเวลานี้ยิ่งมีนัยสำคัญมาก

### 902.3 Worked Example — Bisect หา Root Cause ใน 3 Test

**Test 1 — กึ่งกลางของ Path ทั้ง 7 Hop คือ CORE-SW1/CORE-SW2 (ตำแหน่งที่ 3-4)**

```
CORE-SW1# ping 10.10.30.10 source Loopback0    ! ทดสอบไปทาง Server1 (ครึ่งหลัง)
Success rate is 100 percent (5/5)

CORE-SW1# ping 10.10.10.5 source Loopback0     ! ทดสอบไปทาง PC1 (ครึ่งแรก)
Success rate is 0 percent (0/5)
```

**ผลลัพธ์ Test 1**: ครึ่งหลัง (CORE-SW1 → Server1) **ผ่าน 100%** — ตัดทิ้งได้ทันที: CORE-SW2,
DIST-SW3, ACCESS-SW3 และ Server1 เอง **ไม่ใช่ต้นเหตุ** ครึ่งแรก (CORE-SW1 → PC1) **ล้มเหลว 100%**
— ปัญหาอยู่ระหว่าง PC1 กับ CORE-SW1 แน่นอน (ตัดปัญหาจาก 7 Hop เหลือ 3 Hop ใน Test เดียว)

**Test 2 — Bisect ครึ่งแรกที่เหลือ (PC1 ↔ ACCESS-SW1 ↔ DIST-SW1 ↔ CORE-SW1) กึ่งกลางคือ DIST-SW1**

```
DIST-SW1# ping 10.255.10.1 source Loopback0    ! ทดสอบไปทาง CORE-SW1
Success rate is 100 percent (5/5)

DIST-SW1# ping 10.10.10.5                       ! ทดสอบไปทาง PC1
Success rate is 0 percent (0/5)
```

**ผลลัพธ์ Test 2**: ตัด DIST-SW1 ↔ CORE-SW1 ออกไปได้อีก — ปัญหาเหลืออยู่ระหว่าง PC1 กับ DIST-SW1
เท่านั้น (คือ PC1 หรือ ACCESS-SW1)

**Test 3 — จุดสุดท้ายที่เหลือ**

```
ACCESS-SW1# show ip interface brief | include Vlan10
Vlan10                 10.10.10.1      YES NVRAM  up                    up

ACCESS-SW1# show interfaces GigabitEthernet1/0/5 | include line protocol
GigabitEthernet1/0/5 is up, line protocol is down
```

**พบ Root Cause ใน 3 Test**: พอร์ตที่ต่อ PC1 มี `line protocol is down` (Layer 2 Keepalive ไม่ผ่าน
แม้ Physical จะ `up`) — สาเหตุจริงที่ยืนยันต่อคือปัญหาระดับ Data Link/Encapsulation Mismatch หรือ
Trunk Native VLAN ไม่ตรงกันที่พอร์ตนี้เพียงจุดเดียว **ไม่ต้องเสียเวลาไปเช็ค CORE-SW2, DIST-SW3,
ACCESS-SW3 หรือ Server1 เลยแม้แต่คำสั่งเดียว**

### 902.4 เงื่อนไขที่ Bisection ใช้ไม่ได้ผลดี — ต้องรู้ข้อจำกัดด้วย

| สถานการณ์ | ทำไม Bisection ไม่เหมาะ | ควรใช้วิธีไหนแทน |
|---|---|---|
| ปัญหาเป็นแบบ Intermittent (ไม่ Fail 100% คงที่) | Test ที่กึ่งกลางอาจ "บังเอิญผ่าน" ทั้งที่จุดนั้นมีปัญหาจริง | Follow-the-Path พร้อม Log ยาวพอ (Step 909) |
| สงสัยว่ามีมากกว่า 1 จุดเสียพร้อมกัน | Bisection สมมติว่ามีจุดเสีย "จุดเดียว" ตามหลัก Occam's Razor | ตรวจแยกทุกสมมติฐานแบบ Part 69 Step 684.1 |
| Path ไม่เป็นเส้นตรง (มี ECMP/Redundant Path) | Traffic จริงอาจไม่วิ่งผ่านอุปกรณ์ที่ Ping ทดสอบเลย | ยืนยัน Path จริงด้วย `traceroute`/`show ip cef` ก่อน Bisect |

---

## Step 903 — Scientific Method ในงาน Network: หลีกเลี่ยง Shotgun Troubleshooting

### 903.1 วงจร Hypothesis → Test → Observe → Refine

```
        ┌─────────────┐
   ┌───►│  Hypothesis  │  ตั้งสมมติฐานเดียว ที่เจาะจง และ "พิสูจน์ผิดได้" (Falsifiable)
   │    └──────┬──────┘
   │           ▼
   │    ┌─────────────┐
   │    │     Test     │  ทำการทดสอบ 1 อย่างที่ยืนยัน/หักล้างสมมติฐานนั้นโดยเฉพาะ
   │    └──────┬──────┘
   │           ▼
   │    ┌─────────────┐
   │    │   Observe    │  บันทึกผลจริงที่เกิดขึ้น (ไม่ตีความล่วงหน้า)
   │    └──────┬──────┘
   │           ▼
   │    ┌─────────────┐
   └────┤    Refine    │  ถ้าไม่ตรงสมมติฐาน → ตั้งสมมติฐานใหม่จาก Evidence ที่ได้ ไม่ใช่ "เดาใหม่"
        └──────┬──────┘
               ▼ (ตรงสมมติฐาน)
         Root Cause ยืนยันแล้ว → แก้ไข → Verify
```

**หลักการที่ขาดไม่ได้**: สมมติฐานที่ดีต้อง **"พิสูจน์ผิดได้" (Falsifiable)** — คือต้องมีผลลัพธ์ที่ชัดเจน
ว่า "ถ้าเห็นแบบนี้ = ผิด, ถ้าเห็นแบบนั้น = ถูก" สมมติฐานที่ตีความได้สองทาง (เช่น "อาจจะเป็นที่ Switch
มั้ง") ไม่มีประโยชน์เพราะทดสอบไม่ได้จริง

### 903.2 Shotgun Troubleshooting คืออะไร และทำไมมันถึงดูน่าดึงดูด

**Shotgun Troubleshooting** คือการเปลี่ยนหลายอย่างพร้อมกันในความหวังว่า "สักอันจะแก้ปัญหาได้" โดยไม่
ทดสอบแยกทีละอย่าง — [Part 69 Step 687](part-069-mega-lab-1.md) เคยเตือนเรื่องนี้ใน**ระดับ
Incident/Change Management** (การ Bundle การเปลี่ยนแปลงหลายจุดไว้ใน Change Window เดียวทำให้ Blast
Radius กว้างเกินจำเป็น) — Step นี้สอนบทเรียนเดียวกัน**ในระดับการวินิจฉัยปัญหาเดี่ยว 1 จุด** ซึ่งเกิด
บ่อยกว่ามากในการทำงานวันต่อวัน

**ตัวอย่างสถานการณ์ Shotgun ที่พบบ่อย**: พอร์ตที่ต่อ IP Phone มีปัญหาเสียงแตกเป็นบางครั้ง วิศวกรที่รีบ
แก้จะทำสิ่งนี้ **พร้อมกันในคำสั่งเดียวชุด**:

```
! วิธีที่ผิด — เปลี่ยนหลายอย่างพร้อมกันโดยไม่ทดสอบทีละจุด
ACCESS-SW2(config-if)# duplex full
ACCESS-SW2(config-if)# speed 1000
ACCESS-SW2(config-if)# shutdown
ACCESS-SW2(config-if)# no shutdown
ACCESS-SW2(config-if)# mls qos trust device cisco-phone
ACCESS-SW2(config-if)# spanning-tree portfast
```

ถ้าปัญหาหายไปหลังจากนี้ **จะไม่มีทางรู้เลยว่าอันไหนคือสาเหตุจริง** — ครั้งถัดไปที่เจอปัญหาคล้ายกันที่
พอร์ตอื่น ต้องเริ่มเดาใหม่ทั้งหมดอีกครั้ง ไม่มี Knowledge สั่งสมขึ้นเลย ยิ่งไปกว่านั้น ถ้าปัญหา**ไม่
หาย** ก็ไม่รู้ด้วยว่าควร Rollback อะไรกลับก่อน (Rollback ทั้งหมดอาจ Rollback สิ่งที่ช่วยแก้ปัญหาบางส่วน
ไปด้วย)

### 903.3 เดินตามวงจรวิทยาศาสตร์กับปัญหาเดียวกัน

| รอบ | Hypothesis (พิสูจน์ผิดได้) | Test | Observe | Refine |
|---|---|---|---|---|
| 1 | "สาย Patch Cord เสีย/หลวม" | `show interfaces ... \| include error` | `0 input errors, 0 CRC` — ไม่มี Physical Error สะสม | **หักล้างแล้ว** ไม่ใช่สาย → ตัด Hypothesis นี้ทิ้ง |
| 2 | "Duplex Mismatch" | `show interfaces status` เทียบ 2 ฝั่ง | ทั้งคู่ `a-full` (Auto เจรจาสำเร็จตรงกัน) | **หักล้างแล้ว** ไม่ใช่ Duplex |
| 3 | "ไม่มี QoS Trust Boundary ที่พอร์ต IP Phone" | `show mls qos interface Gi1/0/5` | `trust state: not trusted` | **ยืนยันตรงสมมติฐาน** — ตรงกับ Signature ของ [Part 69 Step 684](part-069-mega-lab-1.md) |

รอบที่ 3 ใช้ **1 คำสั่งเดียว** แก้ไข (`mls qos trust device cisco-phone`) และรู้แน่ชัด 100% ว่านี่คือ
สาเหตุที่แท้จริง เพราะ Hypothesis ที่ 1-2 ถูกหักล้างด้วย Evidence จริงไปก่อนแล้ว ไม่ใช่การเดาสุ่ม

### 903.4 กฎเหล็กที่ป้องกัน Shotgun Troubleshooting

1. **เปลี่ยนทีละอย่าง แล้ว Test ก่อนเปลี่ยนอย่างต่อไปเสมอ** — แม้จะรีบก็ตาม
2. **Hypothesis ต้องมาจาก Evidence รอบก่อนหน้า ไม่ใช่ "ลองสิ่งที่เหลืออยู่ในหัว"**
3. **ถ้าไม่รู้จะเปลี่ยนอะไรต่อ ให้กลับไป Baseline Comparison (Step 904) แทนการเดา**
4. **บันทึกทุกรอบไว้ (Step 909)** — ทั้งที่ผ่านและไม่ผ่าน เพื่อไม่ต้องเดาซ้ำที่เดิม

---

## Step 904 — Baseline Comparison: "อะไรเปลี่ยนไป" คือทางลัดที่เร็วที่สุด

### 904.1 ทำไมการไล่คิดจากศูนย์ (First Principles) มักช้ากว่าที่ควร

วิศวกรจำนวนมากเมื่อเจอปัญหาจะรีบไล่อ่าน Config ทั้งไฟล์ พยายามเข้าใจ Logic ทั้งหมดใหม่ตั้งแต่ต้น —
วิธีนี้ **ใช้ได้เสมอแต่ช้าที่สุด** เพราะระบบที่ทำงานปกติมาหลายเดือน/ปีมี Logic ที่ถูกต้องอยู่แล้วเป็น
ส่วนใหญ่ สิ่งที่ทำให้มันเสียคือ **การเปลี่ยนแปลง 1 จุดเมื่อเร็วๆ นี้** เกือบทุกครั้ง — คำถามที่เร็วที่สุด
เสมอคือ **"ตั้งแต่เมื่อไหร่ที่มันยังทำงานปกติ และมีอะไรเปลี่ยนไปตั้งแต่ตอนนั้น"** ไม่ใช่ "Logic ทั้งระบบ
นี้ถูกต้องไหม"

### 904.2 เครื่องมือที่ 1 — `show archive config differences` (ในเครื่องเดียว)

```
DIST-SW2# show archive config differences

  ! เทียบ Running-config ปัจจุบันกับ Checkpoint ล่าสุดที่ archive เก็บไว้อัตโนมัติ
+ interface GigabitEthernet1/0/12
+  ip ospf dead-interval 40
-  ip ospf dead-interval 12
```

คำสั่งนี้ (สอนไว้ตั้งแต่ [Part 58 Step 683.4](part-069-mega-lab-1.md) ที่ใช้จริงในการหา Root Cause
ของ Mega-Lab) ตอบคำถาม "อะไรเปลี่ยนไปบนเครื่องนี้" ได้ทันทีโดยไม่ต้องอ่าน Config ทั้งไฟล์เลย —
ข้อจำกัดคือใช้ได้เฉพาะเทียบกับ Checkpoint ล่าสุดบนเครื่องนั้นเครื่องเดียว (ไม่มี History ยาวย้อนหลัง
หลายวัน)

### 904.3 เครื่องมือที่ 2 — `git diff` ผ่าน Config Backup System (Part 47/79)

[Part 47 Step 468](part-047-automation-python-advanced.md) สร้างระบบที่ Backup Running-config ของ
ทุกอุปกรณ์เข้า Git Repository พร้อม Commit ทุกวัน และ [Part 79 Step 787](part-079-fullstack-automation-cicd.md)
ขยายให้ทำงานอัตโนมัติก่อน Deploy ทุกครั้งใน CI/CD Pipeline — ข้อดีที่เหนือกว่า `show archive` คือ
**เทียบระหว่างวันที่ใดก็ได้ ไม่จำกัดแค่ Checkpoint ล่าสุด**:

```bash
$ git log --oneline -- DIST-SW2.cfg
a3f9c21 pre-deploy backup before commit 7e21ab0 at 2026-09-22T16:05:00
f52b8aa pre-deploy backup before commit 5b10ff2 at 2026-09-15T09:00:00

$ git diff f52b8aa a3f9c21 -- DIST-SW2.cfg
diff --git a/DIST-SW2.cfg b/DIST-SW2.cfg
--- a/DIST-SW2.cfg
+++ b/DIST-SW2.cfg
@@ -42,7 +42,7 @@ interface GigabitEthernet1/0/12
 interface GigabitEthernet1/0/12
- ip ospf dead-interval 12
+ ip ospf dead-interval 40
```

เมื่อรู้ว่า Commit `a3f9c21` (22 กันยายน 16:05) คือจุดที่ค่าเปลี่ยน สามารถหาต่อได้ว่า Pipeline Run ไหน
Deploy การเปลี่ยนแปลงนี้ และใคร Approve ผ่าน Audit Trail ของ [Part 79 Step 782](part-079-fullstack-automation-cicd.md)
— **ไม่ต้องเดาเลยว่า "ใครทำ เมื่อไหร่"**

### 904.4 เครื่องมือที่ 3 — Compliance-Check แบบ Part 47 (เทียบกับ Template ที่ "ควรจะเป็น")

Baseline ไม่จำเป็นต้องเป็น "เมื่อวานที่ยังทำงานปกติ" เท่านั้น — บางครั้ง Baseline ที่ถูกต้องคือ
**Template มาตรฐานที่ทุกอุปกรณ์ Role เดียวกันต้องมี** (`compliance_check.py` จาก
[Part 47 Step 741](part-047-automation-python-advanced.md)):

```python
def check_compliance(running_config):
    violations = []
    if "mls qos trust device cisco-phone" not in running_config:
        violations.append("Missing QoS trust boundary on access-voice template")
    if "ip ospf dead-interval 40" not in running_config:
        violations.append("OSPF dead-interval ไม่ตรง Template มาตรฐาน (ต้องเป็น 40)")
    return violations
```

รันกับอุปกรณ์ทุกตัวใน Role เดียวกันพร้อมกัน จะเห็นว่า **เครื่องไหนเบี่ยงเบนจาก Baseline** ได้ทันที
โดยไม่ต้อง Diff ทีละเครื่องด้วยมือ — เหมาะมากเมื่อสงสัยว่าปัญหาเกิดจาก "อุปกรณ์ตัวหนึ่งหลุด Template"
ท่ามกลางอุปกรณ์ Role เดียวกันหลายสิบตัว (เหมือนสถานการณ์ ACCESS-SW2 ใน
[Part 69 Step 684.4](part-069-mega-lab-1.md) ที่หลุด Template ไปจากการเปลี่ยน Hardware)

### 904.5 ลำดับที่แนะนำเมื่อเริ่ม Troubleshoot ปัญหาใดๆ

```
1. ถามก่อนเลย: "ระบบนี้เพิ่งเปลี่ยนแปลงอะไรหรือไม่ (Change Ticket/Maintenance Window)?"
2. ถ้ามี Git History → git diff เทียบวันที่ก่อน/หลังปัญหาเกิด (904.3)
3. ถ้าไม่มี Git แต่มี Archive → show archive config differences (904.2)
4. ถ้าไม่มีทั้งคู่ → Compliance-check เทียบกับอุปกรณ์ Role เดียวกันตัวอื่น (904.4)
5. เฉพาะเมื่อไม่มี Baseline ให้เทียบเลย → จึงไล่คิดจาก First Principles (Bottom-Up ตาม Step 901)
```

> **ข้อควรระวัง**: Baseline Comparison บอกได้ว่า **"อะไรเปลี่ยน"** แต่ไม่ได้แปลว่าสิ่งที่เปลี่ยนคือ
> Root Cause เสมอไป (อาจเป็น Correlation ไม่ใช่ Causation เหมือนที่ [Part 69 Step 681.2](part-069-mega-lab-1.md)
> เตือนไว้) — ยังต้องผ่านวงจร Hypothesis → Test → Observe ของ Step 903 เพื่อพิสูจน์ให้แน่ใจอยู่ดี
> เพียงแต่ Baseline ให้ **Hypothesis ตัวแรกที่ดีที่สุด** แทนการเดาแบบไม่มีทิศทาง

---

## Step 905 — อ่านนัยของคำร้องเรียนที่คลุมเครือ

### 905.1 ทำไมคำร้องเรียนจากผู้ใช้ถึง "แปลไม่ได้ตรงๆ" เสมอ

ผู้ใช้ทั่วไปไม่มีคำศัพท์ทางเทคนิคในการอธิบายปัญหา และมักอธิบายผ่าน**ผลกระทบต่อการทำงาน** ไม่ใช่
สาเหตุทางเทคนิค — คำร้องเรียนอย่าง **"เน็ตช้า"** อาจหมายถึงอะไรก็ได้ตั้งแต่ DNS Resolution ช้า,
Bandwidth ไม่พอ, Application Server ช้า, ไปจนถึง MTU/Fragmentation Blackhole (ดู Step 910)
งานของวิศวกรคือ **แปลง (Translate)** คำร้องเรียนคลุมเครือเป็น**สมมติฐานที่ทดสอบได้** ผ่านการถาม
คำถามที่ถูกต้อง — ไม่ใช่การเดาจากคำร้องเรียนตรงๆ

### 905.2 กรอบคำถาม 6 หมวดที่ต้องถามทุกครั้ง (ก่อนแม้แต่จะเปิด Terminal)

| หมวด | คำถามตัวอย่าง | สิ่งที่คำตอบช่วยตัดขอบเขต |
|---|---|---|
| **What (อะไรที่เจาะจง)** | "ช้า" หมายถึงหน้าจอโหลดช้า, หรือค้างไปเลย, หรือ Error Message? มี Error Code ไหม? | แยก Timeout จริงออกจาก Perceived Slowness |
| **When (เมื่อไหร่)** | เริ่มเป็นตั้งแต่เมื่อไหร่แน่ๆ? เกิดตลอดเวลาหรือเฉพาะบางช่วง (เช้า/บ่าย/Peak Hour)? | ชี้ Capacity/Congestion vs Config ผิดถาวร |
| **Who (ใครบ้าง)** | เกิดกับคนเดียวหรือหลายคน? แผนกเดียวหรือทั้งอาคาร? เครื่องเดิมหรือทุกเครื่อง? | บอก Scope: Endpoint เดียว, VLAN, Site, หรือทั้งองค์กร |
| **Where (จากที่ไหนไปที่ไหน)** | ปัญหาเกิดตอนเข้า Internal App หรือ External/Internet ด้วย? Wi-Fi หรือสาย LAN? | บอกว่า Path ไหนที่เกี่ยวข้อง |
| **What Changed (มีอะไรเปลี่ยน)** | มีการอัปเดต/ติดตั้ง/ย้ายระบบอะไรก่อนหน้านี้ไหม (แม้จะดูไม่เกี่ยวก็ตาม)? | เชื่อมกับ Baseline Comparison (Step 904) ทันที |
| **Reproducibility (ทำซ้ำได้ไหม)** | ทำตามขั้นตอนเดิมซ้ำแล้วเป็นทุกครั้งหรือเป็นบางครั้ง? | Consistent = Config ผิด, Intermittent = Capacity/Race Condition |

### 905.3 ตัวอย่างการแปลจริง — "เน็ตช้า" กลายเป็นสมมติฐานทดสอบได้

```
ผู้ใช้: "เน็ตช้ามากเลยตอนนี้"

วิศวกร: "ช้าตอนเปิดเว็บ หรือช้าตอนโหลดไฟล์ หรือทั้งคู่ครับ?"
ผู้ใช้: "เปิดเว็บทั่วไปเร็วปกตินะ แต่พอดาวน์โหลดไฟล์จาก Server ที่ทำงานแล้วมันช้ามาก"

วิศวกร: "เกิดกับทุกคนในทีมหรือแค่พี่คนเดียวครับ?"
ผู้ใช้: "เห็นพี่ข้างๆ (เดสก์ติดกัน คนละสาย LAN) ก็บ่นเหมือนกันเมื่อเช้า"

วิศวกร: "เริ่มเป็นตั้งแต่เมื่อไหร่ครับ?"
ผู้ใช้: "จำไม่ได้แน่ๆ แต่ไม่กี่วันนี้แหละ"
```

**ผลจากการถาม**: "เน็ตช้า" ที่คลุมเครือ กลายเป็นสมมติฐานที่เจาะจงและทดสอบได้ทันที: **"Throughput ไป
Internal File Server ต่ำผิดปกติ เกิดกับหลายคนใน Segment เดียวกัน เพิ่งเริ่มไม่กี่วันนี้ — Internet
Browsing ปกติดี (ตัด DNS/Internet Path ทิ้งได้)"** — Hypothesis แรกที่สมเหตุสมผลคือ **Bandwidth
Bottleneck หรือ Duplex Mismatch บน Uplink ของ Segment นั้น ไม่ใช่ Internet/WAN** (เพราะ Internet
Browsing ปกติดีอยู่ ตัด Path ไป ISP ทิ้งได้ทั้งหมด)

### 905.4 คำเตือน: อย่าถามนำ (Leading Question) จนได้คำตอบที่อยากได้ยิน

```
❌ "เป็นเพราะ WiFi หลุดใช่ไหมครับ?"        (ถามนำ — ผู้ใช้อาจตอบ "ใช่" ทั้งที่ไม่แน่ใจ)
✅ "ตอนที่เป็นปัญหา ต่อผ่าน WiFi หรือสาย LAN ครับ?"   (คำถามเปิด ให้ผู้ใช้ตอบข้อเท็จจริง)
```

คำถามที่ดีต้องเป็น **Open-ended คำถามเชิงข้อเท็จจริง** ที่ผู้ใช้ตอบได้โดยไม่ต้องเดาสาเหตุทางเทคนิคเอง
— หน้าที่วิเคราะห์เป็นของวิศวกร ไม่ใช่ของผู้ใช้

---

## Step 906 — ใช้ Packet Capture อย่างมีเป้าหมาย

### 906.1 เมื่อไหร่ที่ `show`/`debug` ไม่พอ

คำสั่ง `show` และ `debug` บอกได้ว่า **Router/Switch "คิดว่า" มันทำอะไรอยู่** แต่บางปัญหาต้องเห็น
**Bit ที่วิ่งอยู่บนสายจริง** ถึงจะวินิจฉัยได้ เช่น:

- สงสัยว่า Payload ถูกส่งผิด Format (Application Bug ปลอมตัวเป็นปัญหา Network — ดู Step 907)
- ต้องพิสูจน์ว่า Packet ถูก Drop **ระหว่างทาง** จริงหรือไม่ (ไม่ใช่แค่ปลายทางไม่ตอบ)
- ต้องเห็น TCP Retransmission/Window Size จริงเพื่อวินิจฉัยปัญหา Performance
- สงสัย ICMP Message บางประเภทถูก Filter หายไปเงียบๆ (ดู Step 910)

### 906.2 SPAN — Local Switched Port Analyzer

```
DIST-SW1(config)# monitor session 1 source interface GigabitEthernet1/0/5
DIST-SW1(config)# monitor session 1 destination interface GigabitEthernet1/0/24
```

ใช้เมื่อ Source และจุดที่ต่อ Laptop วิเคราะห์ (Wireshark) อยู่บน **Switch ตัวเดียวกัน** — ง่ายที่สุด
แต่จำกัดในขอบเขตอุปกรณ์เดียว

### 906.3 RSPAN — Remote SPAN ข้าม Switch ผ่าน VLAN

```
! บน Switch ต้นทาง (มี Traffic ที่ต้องการ Capture)
SOURCE-SW(config)# vlan 999
SOURCE-SW(config-vlan)# remote-span
SOURCE-SW(config)# monitor session 1 source interface GigabitEthernet1/0/5
SOURCE-SW(config)# monitor session 1 destination remote vlan 999

! บน Switch ปลายทาง (ต่อ Laptop Wireshark จริง)
DEST-SW(config)# monitor session 1 source remote vlan 999
DEST-SW(config)# monitor session 1 destination interface GigabitEthernet1/0/24
```

ใช้เมื่อ Source Traffic อยู่ Switch หนึ่ง แต่วิศวกร/Laptop วิเคราะห์อยู่อีกที่ (เช่น NOC กลาง) — ต้อง
มี VLAN `remote-span` เดินทางผ่าน Trunk เชื่อมทุก Switch ระหว่างทางได้

### 906.4 ERSPAN — ข้าม Layer 3/ข้าม Site ทั้งหมด

```
DIST-SW1(config)# monitor session 2 type erspan-source
DIST-SW1(config-mon-erspan-src)# source interface GigabitEthernet1/0/5
DIST-SW1(config-mon-erspan-src)# destination
DIST-SW1(config-mon-erspan-src-dst)# erspan-id 1
DIST-SW1(config-mon-erspan-src-dst)# ip address 10.10.99.50
DIST-SW1(config-mon-erspan-src-dst)# origin ip address 10.10.10.1
```

ERSPAN ห่อ Traffic ที่ Capture ด้วย GRE แล้วส่งข้าม IP Network ไปยังปลายทางที่ไหนก็ได้ (เช่น
NOC ที่อยู่คนละ Site, หรือ Server วิเคราะห์ Traffic กลางที่รัน Wireshark/IDS) — Platform ต้องรองรับ
(Catalyst 9000 รองรับตาม [Part 90](part-090-catalyst-9000-advanced-features.md))

### 906.5 หลักการเลือกตำแหน่ง SPAN — ผูกกับ "คำถามที่ต้องการคำตอบ"

| คำถามที่ต้องการคำตอบ | ตำแหน่งที่ควร SPAN |
|---|---|
| "Client ส่ง Request จริงหรือไม่" | Access Port ที่ต่อ Client |
| "Server ตอบกลับจริงหรือไม่ / ตอบช้าแค่ไหน" | Access Port ที่ต่อ Server (SPAN สองฝั่งพร้อมกันเพื่อเทียบ Timestamp) |
| "Traffic ถูก Drop ระหว่างทางที่จุดไหน" | SPAN ที่ Uplink ก่อน/หลังจุดที่สงสัย (Bisection แนวคิดเดียวกับ Step 902) |
| "Firewall/ACL บล็อกอะไรอยู่จริง" | SPAN ทั้งขาเข้าและขาออกของจุดที่ ACL/Policy Apply อยู่ |

### 906.6 อ่าน Capture แบบมีเป้าหมาย — ไม่ใช่ไล่ดูทุก Packet

ความผิดพลาดที่พบบ่อยที่สุดคือเปิด Capture ไฟล์ใหญ่แล้ว **Scroll ไล่ดูทุกบรรทัด** — วิธีที่ถูกคือ
**ตั้ง Display Filter ให้ตรงกับ Signal ที่ต้องการเห็นเท่านั้น** ก่อนเริ่มดู:

| ต้องการหา | Wireshark Display Filter |
|---|---|
| TCP Retransmission (Performance/Packet Loss) | `tcp.analysis.retransmission` |
| TCP Session ที่ไม่จบ Handshake สมบูรณ์ | `tcp.flags.syn==1 and tcp.flags.ack==0` |
| ICMP Fragmentation Needed (PMTUD) | `icmp.type==3 && icmp.code==4` |
| Traffic เฉพาะ Host/Port ที่สงสัย | `ip.addr==10.10.30.10 && tcp.port==443` |
| DNS Query ที่ไม่มี Response กลับมา | `dns && !dns.flags.response` |

> **หลักการ**: ตั้ง Hypothesis (Step 903) ก่อนเปิด Capture เสมอ — "ฉันกำลังหา Signal อะไร" ต้องตอบได้
> ก่อนกด Start Capture ไม่ใช่ Capture ก่อนแล้วค่อยคิดว่าจะหาอะไรทีหลัง (Capture ไม่มีเป้าหมายคือ
> Shotgun Troubleshooting เวอร์ชัน Packet-level ตาม Step 903.2)

---

## Step 907 — Multi-Vendor/Multi-Domain Triage: ใช่ปัญหา Network จริงหรือไม่

### 907.1 กับดักที่พบบ่อยที่สุดของทีม Network

เมื่อผู้ใช้รายงานว่า "ใช้งานแอปพลิเคชันไม่ได้" ทีม Network มักถูกเรียกเป็นทีมแรกเสมอ (เพราะ Network
คือสิ่งที่ "อยู่ตรงกลาง" ระหว่างทุกระบบ) แต่ Root Cause จริงมีโอกาสสูงที่จะอยู่ในโดเมนอื่นที่ทีม
Network **ไม่ได้ถูกฝึกมาโดยตรง** — Server Linux, Cloud Provider, หรือ Application Bug เอง — งานแรก
ที่ต้องทำไม่ใช่ "แก้ปัญหา Network" แต่คือ **พิสูจน์ด้วย Evidence ว่านี่ใช่โดเมน Network จริงหรือไม่**

### 907.2 กรอบ Triage แบบเร็ว — ทดสอบที่ขอบเขตของแต่ละ Layer/Domain

```
Client ──► [1] Network Path ──► [2] Server OS/Network Stack ──► [3] Application Process ──► [4] Backend/Cloud
```

| จุดทดสอบ | คำสั่ง/วิธี | ผลที่แยกโดเมนได้ |
|---|---|---|
| **[1] Network Path** | `ping`/`traceroute` จาก Client ถึง Server IP | ถ้าสำเร็จ 100% ไม่มี Loss/Latency ผิดปกติ → ตัด Routing/Switching ออกได้ทั้งหมด |
| **[2] Server Network Stack** | บน Linux Server: `ip addr`, `ethtool eth0`, `ss -tlnp \| grep <port>` | ยืนยันว่า NIC/Driver ปกติ และ **Process จริงกำลัง Listen อยู่บน Port ที่ควร** |
| **[3] Application Process เอง** | `curl -v http://localhost:<port>` **จากบน Server ตัวเอง** | ถ้าสำเร็จจาก Localhost แต่ไม่สำเร็จจากภายนอก → ปัญหาอยู่ระหว่าง Network กับ Server ไม่ใช่ตัว App |
| **[4] Backend/Cloud** | ตรวจ Cloud Security Group/NSG, Load Balancer Health Check | ปัญหา Cloud มักมี Signature เฉพาะ (Timeout เท่ากันทุกครั้งพอดีตาม Cloud Timeout Default) |

### 907.3 ตัวอย่าง Signature ที่ชี้ว่า "ไม่ใช่ Network"

| Signature ที่สังเกตได้ | ความหมาย |
|---|---|
| `ping`/ICMP สำเร็จ 100% แต่ `curl`/Browser Timeout | ปัญหาอยู่ที่ L4-L7 ไม่ใช่ L3 (Port ปิด, Firewall Rule เฉพาะ Port, หรือ Application ค้าง) |
| Connection สำเร็จเร็ว (`tcp.flags.syn-ack` มาเร็ว) แต่ Response Body ช้ามาก | Application Processing ช้า (Database Query ช้า, Thread Pool เต็ม) — Network ทำหน้าที่ตัวเองเสร็จแล้ว |
| ปัญหาเกิดเฉพาะ Cloud Region เดียว หรือเฉพาะช่วง Auto-Scaling | ไม่ใช่ On-Prem Network เลย เป็นเรื่อง Cloud Capacity/Config |
| `netstat`/`ss` บน Linux Server ไม่เห็น Process Listen บน Port ที่คาด | Application ไม่ได้ทำงาน/Crash — ไม่ใช่ Network แม้ว่า Symptom จะดู "เข้าไม่ได้" เหมือนกัน |

### 907.4 การทำงานร่วมกับทีมอื่นอย่างมีประสิทธิภาพ — ไม่ใช่แค่ "โยนปัญหา"

หลักการที่ถูกต้องไม่ใช่การพูดว่า **"Ping ผ่าน ไม่ใช่ปัญหา Network"** แล้วปิด Ticket ทันที (นี่คือการ
โยนปัญหาไม่ใช่ Triage) — แต่คือการ**ส่งต่อพร้อม Evidence ที่เป็นประโยชน์**:

```
สรุปที่ควรส่งให้ทีม Application/Server:
"Network Path จาก Client ถึง Server ปกติ 100% (Ping/Traceroute ไม่มี Loss, Latency 3ms)
TCP Port 8443 เปิดรับ Connection ได้ (SYN-ACK มาภายใน 5ms) — Handshake สมบูรณ์
แต่หลัง Handshake สำเร็จ Server ไม่ส่ง HTTP Response กลับมาเลยภายใน 30 วินาที (Capture แนบมาด้วย)
ดูเหมือนปัญหาอยู่ที่ตัว Application Process เอง ไม่ใช่ Network — ขอให้ทีม App ตรวจ Log/Thread ที่ Server"
```

การส่งต่อแบบนี้ **ประหยัดเวลาของทีมถัดไปมหาศาล** เพราะพวกเขาไม่ต้องเริ่มตรวจ Network ซ้ำเอง — และ
สร้างความน่าเชื่อถือให้ทีม Network ในระยะยาว (ตรงข้ามกับการโยนโดยไม่มี Evidence ซึ่งทำให้ทีมอื่นต้อง
เสียเวลาพิสูจน์ย้อนกลับว่า "จริงๆ Network ปกติหรือไม่")

---

## Step 908 — Time-Boxing และวินัยการ Escalate

### 908.1 Tunnel Vision —ศัตรูที่มองไม่เห็นของวิศวกรทุกระดับ

**Tunnel Vision** คือภาวะที่วิศวกรจมอยู่กับ Hypothesis เดียวนานเกินไป มองไม่เห็นความเป็นไปได้อื่น
ทั้งที่ Evidence เริ่มขัดแย้งกับ Hypothesis นั้นแล้ว — ยิ่งลงทุนเวลาไปมากเท่าไหร่กับสมมติฐานหนึ่ง
(Sunk Cost) ยิ่งยากที่จะยอมรับว่ามันผิดและเปลี่ยนทาง [Part 70 Step 698](part-070-enarsi-capstone-mock-exam.md)
เคยเตือนเรื่องนี้ในบริบทเฉพาะข้อสอบ ENARSI (การจัดสรรเวลาต่อ Ticket ในห้องสอบ Lab) — Step นี้ขยาย
เป็นวินัยระดับอาชีพที่ใช้ได้ทั้งใน CCIE Diagnose Module และงาน NOC จริง

### 908.2 กฎ Time-Box ตาม Severity — ยืมแนวคิด Severity Matrix จาก Part 69

[Part 69 Step 682.2](part-069-mega-lab-1.md) นิยาม Severity Matrix สำหรับจัดลำดับ Incident —
Step นี้ต่อยอดเป็น **Time-Box ต่อรอบการวินิจฉัย** (ไม่ใช่ต่อ Incident ทั้งก้อน):

| Severity/บริบท | Time-Box ต่อ Hypothesis 1 รอบ | เมื่อครบเวลาแล้วไม่คืบหน้า ให้ทำอะไร |
|---|---|---|
| Sev-1 (Business ทั้งหมดกระทบ) | 10-15 นาที | Escalate ทันที ดึงคนเพิ่มเข้า War Room |
| Sev-2 (กระทบผู้ใช้จำนวนมาก) | 20-30 นาที | ปรึกษา Senior/เพื่อนร่วมทีมก่อน Escalate เป็นทางการ |
| Sev-3 (กระทบกลุ่มเล็ก) | 45-60 นาที | ลองมุมมองใหม่ด้วยตัวเองอีกรอบ ก่อนขอความเห็นที่สอง |
| CCIE Diagnose Module (ต่อ Ticket) | ตามเวลาที่แบ่งสรรไว้ล่วงหน้า (ดู [Part 70 Step 698](part-070-enarsi-capstone-mock-exam.md)) | ข้ามไป Ticket อื่นก่อน แล้วย้อนกลับมาถ้ามีเวลาเหลือ |

### 908.3 กฎ "Two-Strikes" — สัญญาณที่บอกว่าควร Escalate จริงๆ

ไม่ใช่แค่เวลาหมดเท่านั้นที่ควร Escalate — สัญญาณที่ชัดเจนกว่าคือ:

> **ถ้า Hypothesis ที่สมเหตุสมผลที่สุด 2 ข้อถูกทดสอบและหักล้างไปแล้วทั้งคู่ (ตามวงจร Step 903) และ
> ไม่มี Hypothesis ที่ 3 ที่มี Evidence รองรับจริง (ไม่ใช่แค่ "เดาไปเรื่อยๆ") — นี่คือจุดที่ควร
> Escalate หรือขอความเห็นที่สอง ไม่ใช่พยายามคิด Hypothesis ที่ 4-5-6 ต่อไปคนเดียว**

เหตุผลเชิงหลักการ: การที่ Hypothesis ที่ดีที่สุด 2 ข้อผิดทั้งคู่ มักแปลว่า **มุมมองที่ใช้อยู่ (Frame)
อาจผิดตั้งแต่ต้น** ไม่ใช่แค่ "ยังไม่เจอ Hypothesis ที่ถูก" — คนอื่นที่มามองด้วยมุมมองใหม่ (Fresh Eyes)
มีโอกาสสูงกว่าที่จะเห็นสิ่งที่ถูกมองข้ามไป

### 908.4 Escalate ไม่ใช่ความล้มเหลว — มันคือทักษะ

| ความเชื่อผิดๆ | ความจริง |
|---|---|
| "Escalate แปลว่าฉันทำไม่ได้" | Escalate แปลว่าฉันรู้จักขอบเขตความรู้/เวลาของตัวเอง — เป็นสัญญาณของ Senior Engineer ไม่ใช่ Junior |
| "ต้องหา Root Cause ให้เจอเองก่อนถึงจะขอความช่วยเหลือได้" | การขอความช่วยเหลือพร้อม Evidence ที่มี (แม้ยังไม่เจอ Root Cause) มีประโยชน์กว่าการเงียบจนหมดเวลา |
| "ถ้าข้ามไปทำ Ticket อื่นก่อน แปลว่ายอมแพ้ Ticket นี้" | การพักปัญหาไว้ (Park) แล้วกลับมาด้วยมุมมองใหม่ทีหลัง มักได้ผลดีกว่าจมอยู่กับที่เดิมต่อเนื่อง |

> **สำหรับ CCIE Diagnose Module โดยเฉพาะ**: ไม่มี Ticket ใดควรกิน "เวลาที่แบ่งสรรไว้" ของ Ticket
> อื่นเด็ดขาด — ถ้า Time-Box ของ Ticket หนึ่งหมดแล้วยังไม่เจอ ให้บันทึกสถานะปัจจุบัน (Step 909)
> แล้วข้ามไปก่อน คะแนนจาก Ticket ที่ทำได้เร็วหลายใบมักมากกว่าการทุ่มเวลาทั้งหมดกับ Ticket ยากใบเดียว

---

## Step 909 — บันทึกระหว่างการวินิจฉัย: เขียนไปพร้อมกับที่ทำ

### 909.1 ทำไมการเขียนสรุป "ทีหลัง" ถึงไม่ได้ผล

วิศวกรจำนวนมากตั้งใจจะ "จำไว้ก่อน แล้วสรุปทีหลังตอนจบ" — ปัญหาคือ:

1. **ความจำผิดพลาดได้** โดยเฉพาะเมื่อทดสอบไปแล้วหลายรอบและผลลัพธ์คล้ายกัน
2. **ถ้าต้อง Handoff กลางทาง** (หมดเวร, Escalate ตาม Step 908) คนถัดไปไม่มีข้อมูลอะไรเลยนอกจาก
   คำพูดปากเปล่า
3. **มีความเสี่ยงสูงที่จะ Test ซ้ำสิ่งที่ทำไปแล้ว** โดยไม่รู้ตัว เพราะจำไม่ได้ว่าทำอะไรไปแล้วบ้าง
4. **[Part 69 Step 690](part-069-mega-lab-1.md) (Post-Incident Review)** ต้องใช้ Timeline ที่
   แม่นยำ — ถ้าไม่มี Log ระหว่างทาง PIR จะกลายเป็นการเดาย้อนหลังว่าเกิดอะไรขึ้นบ้าง

### 909.2 Template บันทึกที่ใช้งานได้จริง — เขียนสั้น เขียนทันที

| เวลา | Hypothesis | Test/คำสั่งที่รัน | คาดว่าจะเห็น | เห็นจริง | สรุป |
|---|---|---|---|---|---|
| 09:14 | สายเสีย/หลวม | `show interfaces Gi1/0/5 \| include error` | CRC/Error สูง ถ้าใช่สายเสีย | `0 input errors, 0 CRC` | ❌ ตัดทิ้ง — ไม่ใช่สาย |
| 09:17 | Duplex Mismatch | `show interfaces status` (ทั้ง 2 ฝั่ง) | Duplex ไม่ตรงกัน | ทั้งคู่ `a-full` ตรงกัน | ❌ ตัดทิ้ง |
| 09:21 | QoS Trust Boundary หาย | `show mls qos interface Gi1/0/5` | `trust state: not trusted` ถ้าใช่ | `trust state: not trusted` ตรงเป๊ะ | ✅ **ยืนยัน Root Cause** |
| 09:24 | (แก้ไข) | `mls qos trust device cisco-phone` + `auto qos voip cisco-phone` | เสียงต้องกลับมาชัด | ทดสอบโทรซ้ำ เสียงชัดปกติ | ✅ **Verified — ปิด Ticket** |

**หลักการของ Template นี้**: แต่ละแถวคือ **1 รอบของวงจร Hypothesis → Test → Observe (Step 903)**
เขียนทันทีหลังได้ผลแต่ละรอบ ไม่ใช่รอจบแล้วค่อยเขียนย้อนหลัง — คอลัมน์ "คาดว่าจะเห็น" **ต้องเขียนก่อน
รันคำสั่ง** (นี่คือหลักฐานว่า Hypothesis นั้น Falsifiable จริงตาม Step 903.1 ไม่ใช่เขียนย้อนหลังให้
เข้ากับผลที่เห็นแล้ว)

### 909.3 ประโยชน์ที่ได้คืนมาจริง

| ประโยชน์ | อธิบาย |
|---|---|
| **ไม่ Test ซ้ำ** | เห็นชัดจาก Log ว่า Hypothesis ไหนถูกตัดทิ้งไปแล้ว ไม่เสียเวลาลองใหม่โดยไม่รู้ตัว |
| **Handoff ได้ทันที** | คนที่มารับต่อ (Escalate ตาม Step 908, เปลี่ยนเวร) อ่าน Log แล้วเริ่มจากรอบที่ 5 ต่อได้เลย ไม่ต้องเริ่มจากรอบ 1 |
| **เป็น Input ให้ PIR** | [Part 69 Step 690](part-069-mega-lab-1.md) ใช้ Timeline แบบนี้ตรงๆ ในการเขียน Post-Incident Review |
| **ฝึกวินัย Falsifiable Hypothesis** | การต้องเขียน "คาดว่าจะเห็น" ก่อนรันคำสั่งบังคับให้ตั้งสมมติฐานที่ทดสอบได้จริงเสมอ (ป้องกัน Step 903.2 Shotgun) |

> **หมายเหตุ**: ไม่จำเป็นต้องใช้ตารางเป๊ะแบบนี้เสมอ — Text ธรรมดาที่มี Timestamp + สิ่งที่ทำ + ผล
> ก็เพียงพอ สิ่งที่สำคัญที่สุดคือ **ความถี่ (เขียนทุกรอบ ไม่ใช่ทุกชั่วโมง)** ไม่ใช่รูปแบบของบันทึก

---

## Step 910 — Full Lab: "ผู้ใช้บอกว่าแอปพลิเคชันใหม่ใช้งานไม่ได้"

### 910.1 โจทย์ที่ได้รับมา (ทั้งหมดที่มีในตอนแรก)

> **Ticket INC-51102** — "ทีม Sales รายงานว่าแอปพลิเคชัน CRM ตัวใหม่ใช้งานไม่ได้"

ไม่มีรายละเอียดอื่นเลย — งานของเราคือใช้ Step 901-909 ทั้งหมดไล่หา Root Cause ให้เจอเอง

### 910.2 ใช้ Step 905 — แปลงคำร้องเรียนให้เป็นสมมติฐานที่ทดสอบได้

```
วิศวกร: "ใช้งานไม่ได้ หมายถึง Login ไม่ผ่าน หรือเข้าได้แต่ทำอะไรไม่ได้ครับ?"
ผู้ใช้:  "Login เข้าได้ปกติ เร็วดีด้วยซ้ำ แต่พอกดเปิดหน้า 'รายชื่อลูกค้า' หรือดาวน์โหลดรายงาน
         ที่มีไฟล์แนบ มันจะหมุนค้างแล้วสุดท้าย Timeout"

วิศวกร: "เริ่มเป็นตั้งแต่เมื่อไหร่ครับ?"
ผู้ใช้:  "เช้าวันนี้เอง — ทีม IT บอกว่าเมื่อคืนวันศุกร์มีการย้าย CRM ไปโฮสต์ที่ใหม่"

วิศวกร: "เกิดกับทุกคนในทีม Sales ไหม รวมถึงสาขาด้วยหรือเปล่าครับ?"
ผู้ใช้:  "ทุกคนที่ HQ เจอเหมือนกันหมด เพื่อนที่ BRANCH2 ก็เพิ่งทักมาบอกว่าเจอปัญหาเดียวกัน"
```

**สมมติฐานที่ทดสอบได้จาก 3 คำถามนี้**: "Login (Payload เล็ก, HTTP Response สั้น) สำเร็จเสมอ — แต่
หน้าที่ต้องโหลดข้อมูลจำนวนมาก/ไฟล์แนบ (Payload ใหญ่) ค้าง/Timeout เสมอ เกิดกับทุก Site ที่ไปถึง CRM
ตัวใหม่ (ทั้ง HQ และ BRANCH2) เริ่มพร้อมกันตั้งแต่ Deploy คืนวันศุกร์" — Scope คือ**ทุก Path ที่ไปถึง
CRM ใหม่** ไม่ใช่ปัญหาเฉพาะ User คนเดียวหรือ Site เดียว

### 910.3 ใช้ Step 901 — Pattern-Matching เลือก Layer ที่ควรเริ่ม

Login สำเร็จ = พิสูจน์แล้วว่า DNS, TCP 3-Way Handshake, TLS Handshake, Authentication ทั้งหมด
**ผ่านสมบูรณ์** (L1-L7 ทำงานได้กับ Payload เล็ก) — ตัด DNS/Firewall-block-ทั้งหมด/Routing-ขาด
ทิ้งไปได้เลยทันที ไม่ต้องเสียเวลาไล่เช็ค เพราะถ้า Routing/DNS/TCP มีปัญหาจริง Login จะไม่ผ่านเลยด้วย
สมมติฐานเดียวที่อธิบาย **"เล็กผ่าน ใหญ่ไม่ผ่าน"** ได้ครบตาม Pattern ใน Step 901.2 คือ **ปัญหาที่
เกี่ยวกับขนาด Packet** — วงศ์ตระกูล MTU/Fragmentation คือ Hypothesis แรกที่ต้อง Test

### 910.4 ใช้ Step 904 — Baseline Comparison ยืนยันว่ามีอะไรเปลี่ยนจริง

```
WAN-EDGE-1# show archive config differences

  ! เทียบกับ Checkpoint ก่อน Maintenance Window คืนวันศุกร์
+ interface Tunnel20
+  description ** GRE-over-IPsec to PARTNER-DC-RTR (New CRM Hosting) **
+  ip address 172.16.220.1 255.255.255.252
+  tunnel source GigabitEthernet0/0/0
+  tunnel destination 198.51.100.20
+  tunnel protection ipsec profile IPSEC-PROFILE-PARTNERDC
+ ip access-list extended WAN-EDGE-1-HARDENING-IN
+  deny icmp any any
+  permit ip any any
+ interface GigabitEthernet0/0/0
+  ip access-group WAN-EDGE-1-HARDENING-IN in
```

พบการเปลี่ยนแปลง 2 จุดพร้อมกันในหน้าต่าง Maintenance เดียวกัน (Pattern เดียวกับ
[Part 69 Step 687.2](part-069-mega-lab-1.md) ที่ Bundle การเปลี่ยนแปลงหลายจุดไว้ Window เดียว):
**(1)** Tunnel20 ใหม่สำหรับ CRM ที่ Partner Data Center และ **(2)** ACL Hardening ใหม่ที่ Deny
ICMP ทั้งหมดขาเข้า (ไม่มีการเจาะจง Type/Code เลย — Deny ทุกชนิดของ ICMP)

### 910.5 ใช้ Step 902 — Bisection ยืนยันตำแหน่งที่ Packet ใหญ่ตายจริง

```
! Test จาก WAN-EDGE-1 (กึ่งกลาง Path) ด้วย Packet ขนาดใหญ่ + DF-bit
WAN-EDGE-1# ping 172.20.50.10 size 1500 df-bit source Tunnel20
Type escape sequence to abort.
Sending 5, 1500-byte ICMP Echos to 172.20.50.10, timeout is 2 seconds:
Packet sent with the DF bit set
.....
Success rate is 0 percent (0/5)

WAN-EDGE-1# ping 172.20.50.10 size 1400 df-bit source Tunnel20
Success rate is 100 percent (5/5)
```

**Packet ขนาด 1500 Byte ผ่าน DF-bit ไม่สำเร็จ แต่ 1400 Byte สำเร็จ** — ยืนยันว่า Effective Path
MTU ของ Tunnel20 อยู่ระหว่าง 1400-1500 Byte (ตามที่คาดจาก Overhead ของ GRE + IPsec) และ WAN-EDGE-1
เองไม่ได้รับ ICMP อะไรกลับมาเลยตอนที่ Packet ใหญ่ Fail (ไม่มี "Fragmentation Needed" แจ้งกลับ)

### 910.6 ใช้ Step 906 — Packet Capture ยืนยัน PMTUD Black Hole แบบเห็นภาพ

```
DIST-SW1(config)# monitor session 1 source interface GigabitEthernet1/0/5
DIST-SW1(config)# monitor session 1 destination interface GigabitEthernet1/0/24
```

เปิด Wireshark ที่ปลาย Capture ด้วย Filter `tcp.port==443 && ip.addr==172.20.50.10`:

```
Frame 1042: TCP SYN         PC1 → APP-SRV-CRM (172.20.50.10)
Frame 1043: TCP SYN-ACK     APP-SRV-CRM → PC1
Frame 1044: TCP ACK         PC1 → APP-SRV-CRM         (Handshake สำเร็จ — ตรงกับที่ Login ผ่าน)
Frame 1050: TCP GET /customers/report.pdf (Payload ใหญ่, DF=1, Len=1460)
             (ไม่มี Response กลับมาเลย)
Frame 1085: [TCP Retransmission] GET /customers/report.pdf   (ส่งซ้ำหลัง 1 วินาที)
Frame 1130: [TCP Retransmission] GET /customers/report.pdf   (ส่งซ้ำอีกครั้ง — Exponential Backoff)
             (ไม่มี ICMP Type 3 Code 4 ปรากฏใน Capture เลยแม้แต่เฟรมเดียว)
```

**หลักฐานชัดเจน**: TCP Segment ขนาดใหญ่ (ต้องการ Fragment เพราะเกิน Path MTU ที่ Tunnel20 รับได้)
ถูกส่งซ้ำไปเรื่อยๆ โดยไม่มี ICMP "Fragmentation Needed" ส่งกลับมาบอก Sender ให้ลดขนาดเลย — ตรงกับ
กลไก **PMTUD Black Hole** ที่อธิบายไว้แล้วใน [Part 51 Step 505.3](part-051-vrf-lite-gre-ipsec.md)
ทุกประการ: Client รอ ACK ที่ไม่มาถึง ส่งซ้ำเรื่อยๆ จนกว่า Application เองจะ Timeout — อธิบาย
"หมุนค้างแล้ว Timeout" ที่ผู้ใช้รายงานได้ตรงเป๊ะ

### 910.7 ใช้ Step 907 — พิสูจน์ว่าไม่ใช่ปัญหาฝั่ง Application/Server

ก่อนสรุป Root Cause ต้องพิสูจน์ให้แน่ใจว่าไม่ใช่ Application Bug ปลอมตัวมา:

```
$ curl -v --max-time 5 https://172.20.50.10/health
* Connected to 172.20.50.10 (172.20.50.10) port 443
< HTTP/1.1 200 OK                          (Endpoint เล็กสำเร็จ — Server ทำงานปกติ)

$ curl -v --max-time 30 https://172.20.50.10/customers/report.pdf
* Connected to 172.20.50.10 (172.20.50.10) port 443
* Operation timed out after 30000 milliseconds with 0 bytes received

$ curl -v --max-time 30 --tcp-nodelay --interface eth0 \
      -H "Range: bytes=0-1000" https://172.20.50.10/customers/report.pdf
< HTTP/1.1 206 Partial Content              (บังคับให้ Response เล็กลง → สำเร็จทันที!)
```

การบังคับให้ Server ตอบเป็นชิ้นเล็ก (`Range` Header) แล้วสำเร็จทันที **พิสูจน์ว่า Application/Server
ทำงานถูกต้องสมบูรณ์** — ปัญหาอยู่ที่การส่ง Payload **ขนาดใหญ่** ผ่าน Network เท่านั้น ไม่เกี่ยวกับ
Logic ของ CRM เลยแม้แต่นิดเดียว (ทีม Application ไม่ต้องแก้ Code อะไรทั้งสิ้น)

### 910.8 ใช้ Step 908 — Time-Box และ Escalate ให้ทีม Security (ผู้ดูแล ACL)

Hypothesis 2 ข้อแรกที่ทดสอบไปพร้อมกับทีม Security ในช่วง 20 นาทีแรก ("CRM Server เพิ่ง Deploy อาจ
ยังไม่ Config เสร็จ" และ "Bandwidth ไม่พอที่ Tunnel20") ถูกหักล้างไปแล้วด้วย Evidence จาก 910.5-910.7
— ตาม กฎ Two-Strikes (Step 908.3) ทีมจึงดึง **Security Engineer ผู้เป็นเจ้าของ ACL
`WAN-EDGE-1-HARDENING-IN`** เข้ามาร่วมทันที เพราะ Evidence ทั้งหมดชี้ไปที่ ACL ที่ทีม Security เป็น
คน Push โดยตรง — ไม่ใช่การนั่งเดาต่อคนเดียวว่า "ทำไม ACL ตัวนี้ถึงเขียนแบบนี้"

### 910.9 บันทึกระหว่างทางแบบ Step 909 (สรุปสั้นของทั้งกระบวนการ)

| เวลา | Hypothesis | Test | คาดว่าจะเห็น | เห็นจริง | สรุป |
|---|---|---|---|---|---|
| 08:40 | CRM Server ยัง Config ไม่เสร็จ | `curl /health` | 200 OK ถ้า Server ปกติ | 200 OK | ❌ ตัดทิ้ง — Server ทำงาน |
| 08:52 | Bandwidth ไม่พอที่ Tunnel20 | `show interfaces Tunnel20 \| include rate` | Utilization สูงมาก ถ้าใช่ Bandwidth | Utilization < 5% | ❌ ตัดทิ้ง |
| 09:05 | MTU/Fragmentation ปัญหา (จาก Pattern Step 901) | `ping ... size 1500 df-bit` เทียบ `size 1400` | 1500 Fail, 1400 Success ถ้าใช่ MTU | ตรงเป๊ะตามคาด | ✅ ยืนยันตำแหน่ง |
| 09:15 | ICMP ถูกบล็อกจน PMTUD ทำงานไม่ได้ | SPAN + Wireshark หา `icmp.type==3` | ไม่มี ICMP เลยถ้าใช่ | ไม่มี ICMP แม้แต่เฟรมเดียว | ✅ ยืนยัน Root Cause แบบสมบูรณ์ |
| 09:30 | (Escalate) ดึง Security Engineer มาดู ACL | ตรวจ `WAN-EDGE-1-HARDENING-IN` ร่วมกัน | พบ `deny icmp any any` | พบตรงตามคาด | ✅ พร้อมแก้ไข |

### 910.10 Root Cause สมบูรณ์

**สาเหตุมี 2 ชั้นซ้อนกัน** (ต้องแก้ทั้งคู่ ไม่ใช่แก้แค่จุดเดียว):

1. **Tunnel20 (GRE-over-IPsec ใหม่ไปยัง Partner Data Center) มี Effective Path MTU ต่ำกว่า 1500
   Byte** (Overhead ของ GRE Header + IPsec ESP Header) แต่ **ไม่มีการ Config `ip tcp adjust-mss`**
   ไว้ที่ Tunnel Interface ตามที่ [Part 51 Step 505.3](part-051-vrf-lite-gre-ipsec.md) แนะนำไว้เป็น
   Best Practice สำหรับ GRE-over-IPsec ทุกเส้น
2. **ACL `WAN-EDGE-1-HARDENING-IN`** ที่ทีม Security Push เข้ามาในหน้าต่าง Maintenance เดียวกัน
   เขียนแบบ **`deny icmp any any`** (Deny ICMP ทุกชนิดแบบไม่เจาะจง) ซึ่งบล็อก ICMP Type 3 Code 4
   (Fragmentation Needed) ที่ควรไหลกลับมาบอก Sender ให้ลด Segment Size ไปด้วย — ทำให้ **PMTUD
   ทำงานไม่ได้เลย** (Black Hole ตามหลักการที่ [Part 51 Step 505.3](part-051-vrf-lite-gre-ipsec.md)
   เตือนไว้ล่วงหน้าแล้วว่าเป็นปัญหาที่รู้จักกันดีในวงการ)

ผลลัพธ์: HTTP Request ขนาดเล็ก (Login, health-check) ไม่แตะขอบ MTU เลยจึงผ่านปกติ — แต่ Response
ขนาดใหญ่ (รายชื่อลูกค้า, ไฟล์แนบ PDF) ต้องใช้ TCP Segment ที่ใหญ่กว่า Path MTU และตั้ง DF-bit (ตาม
Default ของ OS สมัยใหม่ทุกตัว) จึงถูก Drop เงียบๆ โดยไม่มีทางรู้ตัวว่าต้องลดขนาด — Retransmit วน
ไปจนกว่า Application จะ Timeout เอง

### 910.11 การแก้ไข (2 ชั้น ตามสาเหตุที่ซ้อนกัน)

```
! ชั้นที่ 1 — TCP MSS Clamping บน Tunnel Interface (แก้ปัญหาโดยไม่พึ่ง ICMP เลย)
WAN-EDGE-1(config)# interface Tunnel20
WAN-EDGE-1(config-if)# ip tcp adjust-mss 1360
WAN-EDGE-1(config-if)# exit

PARTNER-DC-RTR(config)# interface Tunnel20
PARTNER-DC-RTR(config-if)# ip tcp adjust-mss 1360
PARTNER-DC-RTR(config-if)# exit

! ชั้นที่ 2 — แก้ ACL ให้ยอมรับ ICMP ที่จำเป็นต่อ PMTUD (Defense-in-Depth สำหรับ Traffic ที่ไม่ใช่ TCP)
WAN-EDGE-1(config)# ip access-list extended WAN-EDGE-1-HARDENING-IN
WAN-EDGE-1(config-ext-nacl)# no deny icmp any any
WAN-EDGE-1(config-ext-nacl)# permit icmp any any unreachable
WAN-EDGE-1(config-ext-nacl)# permit icmp any any echo-reply
WAN-EDGE-1(config-ext-nacl)# deny icmp any any
WAN-EDGE-1(config-ext-nacl)# permit ip any any
WAN-EDGE-1(config-ext-nacl)# exit
```

> **หมายเหตุ**: แก้ทั้ง 2 ชั้นแม้ Layer แรกอย่างเดียวก็เพียงพอให้ TCP ของ CRM ใช้งานได้แล้ว — เพราะ
> `ip tcp adjust-mss` ไม่พึ่งพา ICMP เลย แต่การเปิด ICMP Unreachable กลับมาด้วยยังจำเป็นสำหรับ
> Traffic ประเภทอื่นที่ไม่ใช่ TCP (เช่น UDP-based Application อื่นในอนาคตที่ไม่มี MSS ให้ Clamp)
> — หลักการ Defense-in-Depth เดียวกับที่ [Part 69 Step 686.6](part-069-mega-lab-1.md) ใช้ตอนแก้ CoPP
> (คืนค่าเดิมที่ปลอดภัย ไม่ใช่ปิด Security ทิ้งทั้งหมด)

### 910.12 Verify

```
WAN-EDGE-1# ping 172.20.50.10 size 1500 df-bit source Tunnel20
Success rate is 100 percent (5/5)     ! (ผ่านเพราะ MSS Clamp ทำให้ TCP ไม่มีทางส่ง Segment ใหญ่
                                          เกิน 1360+40 อยู่แล้ว — Ping ขนาดนี้ทดสอบ Path MTU ตรงๆ
                                          ซึ่งตอนนี้ ICMP unreachable ก็ไหลกลับมาปกติแล้วด้วย)

$ curl -v --max-time 10 https://172.20.50.10/customers/report.pdf
< HTTP/1.1 200 OK
* Connection #0 to host 172.20.50.10 left intact    (ดาวน์โหลดสำเร็จภายใน 2 วินาที)
```

ผู้ใช้ทดสอบเปิดหน้า "รายชื่อลูกค้า" และดาวน์โหลดรายงานซ้ำ — ใช้งานได้ปกติทุกจุด **INC-51102 ปิด**

### 910.13 สรุปบทเรียนที่ใช้ครบทุก Step ของ Part นี้

| Step ที่ใช้ | บทบาทในการไล่หา Root Cause ครั้งนี้ |
|---|---|
| 905 | แปลง "ใช้งานไม่ได้" เป็น "เล็กผ่าน ใหญ่ไม่ผ่าน ทุก Site ตั้งแต่ Deploy คืนวันศุกร์" |
| 901 | Pattern-Match ทันทีว่า "เล็กผ่าน ใหญ่ไม่ผ่าน" = ตระกูล MTU/Fragmentation ข้าม Layer อื่นทั้งหมด |
| 904 | `show archive config differences` เจอ Tunnel20 + ACL ใหม่พร้อมกันในหน้าต่างเดียว |
| 902 | Bisection ด้วย `ping size ... df-bit` หาขอบ MTU ที่แท้จริง (1400 ผ่าน, 1500 ไม่ผ่าน) |
| 906 | SPAN + Wireshark Filter เฉพาะเจาะจง (`icmp.type==3`) ยืนยัน Black Hole แบบเห็นภาพ |
| 907 | `curl --Range` พิสูจน์ว่า Application/Server ไม่มีบัคเลย ไม่ต้องรอทีม App แก้ Code |
| 908 | Two-Strikes Rule ดึง Security Engineer มาร่วมทันทีหลัง Hypothesis อื่นถูกหักล้าง |
| 909 | Log ทุกรอบทำให้ไล่จาก 08:40-09:30 ได้อย่างไม่ Test ซ้ำ และพร้อมเป็น Input ของ PIR |
| 903 | ทุกรอบเดินตาม Hypothesis → Test → Observe → Refine ไม่มีรอบใดเป็นการเดาสุ่ม |

**ข้อสังเกตสำคัญที่สุดของ Lab นี้**: Root Cause จริง (PMTUD Black Hole) **ไม่ใช่เรื่องใหม่**
— หลักสูตรสอนกลไกนี้ไปแล้วเต็มรูปแบบตั้งแต่ [Part 51 Step 505.3](part-051-vrf-lite-gre-ipsec.md)
สิ่งที่ Lab นี้ฝึกจริงๆ **ไม่ใช่ความรู้เรื่อง PMTUD** แต่คือ **กระบวนการค้นพบว่าใช่ PMTUD** จาก
ประโยคเดียวที่คลุมเครือที่สุดเท่าที่จะเป็นไปได้ ("ใช้งานไม่ได้") — นี่คือทักษะที่แท้จริงของ CCIE
Diagnose Module: **ความรู้ Domain อย่างเดียวไม่พอ ต้องมี Methodology ที่พาไปหา Domain ที่ถูกให้เจอ
ด้วยตัวเอง**

---

## แบบฝึกหัดทวนความเข้าใจ Part 91

1. เพราะเหตุใด `show ip interface brief` ที่รายงาน `up/up` จึงทำให้ Expert "ข้าม" การไล่เช็ค Layer
   1-2 ได้ทันที และมีข้อยกเว้นกรณีใดที่ยังต้องตรวจสอบเพิ่มแม้เห็น `up/up` แล้ว?
2. ถ้า Path ที่ต้องสงสัยมี 15 จุด (Hop) การใช้ Bisection Troubleshooting ต้องใช้ Test สูงสุดกี่ครั้ง
   เทียบกับ Linear Search ที่ต้องใช้สูงสุด 15 ครั้ง?
3. อธิบายความแตกต่างระหว่าง "เปลี่ยนหลายอย่างพร้อมกันแล้วปัญหาหาย" กับการเดินตามวงจร Hypothesis →
   Test → Observe → Refine ทีละรอบ — ทำไมวิธีแรกจึงเป็นปัญหาแม้ผลลัพธ์สุดท้ายจะเหมือนกัน (ปัญหาหาย)?
4. เพราะเหตุใดการทดสอบ `curl` แบบขอ Partial Content (`Range` Header) ใน Step 910.7 จึงเพียงพอที่จะ
   พิสูจน์ว่า Application ไม่ใช่ต้นเหตุ โดยไม่ต้องรอให้ทีม Application เข้ามาตรวจ Code เลย?
5. อธิบายกฎ "Two-Strikes" ของ Step 908.3 — เพราะเหตุใดการที่ Hypothesis ที่ดีที่สุด 2 ข้อผิดพลาดทั้งคู่
   จึงเป็นสัญญาณที่บอกว่าควร Escalate มากกว่าการพยายามคิด Hypothesis ข้อที่ 3-4-5 ต่อไปคนเดียว?

**เฉลย:**

1. เพราะ `up/up` (Status/Protocol) พิสูจน์แล้วว่า Physical Layer (Signal/Cable) และ Data Link
   Layer (Keepalive/Encapsulation) ทำงานถูกต้อง ณ ขณะที่เช็ค — การไล่เช็ค Duplex/Error Counter ซ้ำ
   จึงไม่ได้ข้อมูลใหม่ ข้อยกเว้นคือกรณี **Intermittent/Flapping**: `up/up` ที่เห็นตอนนี้ไม่ได้แปลว่า
   ไม่เคย Down มาก่อน ต้องเช็ค `show logging | include UPDOWN` ควบคู่เสมอเพื่อดู History การ Flap
2. ต้องใช้ **⌈log₂(15)⌉ = 4 ครั้ง** เท่านั้น (เทียบกับ Linear สูงสุด 15 ครั้ง) — เพราะ Bisection
   ตัดครึ่งหนึ่งของความเป็นไปได้ที่เหลือออกไปในทุก Test เดียว (15→7-8→3-4→1-2→เจอจุดเดียว)
3. การเปลี่ยนหลายอย่างพร้อมกัน (Shotgun) ทำให้**ไม่รู้ว่าอันไหนคือสาเหตุจริง** แม้ปัญหาจะหายไปก็ตาม
   — ไม่มี Knowledge สั่งสมสำหรับครั้งถัดไป, ถ้าปัญหากลับมาไม่รู้จะ Rollback อะไรก่อน, และเสี่ยงที่
   การเปลี่ยนบางอย่างจะสร้างผลข้างเคียงใหม่ที่ยังไม่ถูกตรวจพบ ส่วนวงจร Hypothesis→Test→Observe→
   Refine ยืนยันสาเหตุที่แท้จริงทีละจุด ทำให้รู้แน่ชัด 100% ว่าอะไรแก้ปัญหาได้จริง
4. เพราะ `Range` Header บังคับให้ Server ตอบกลับเป็น Payload ขนาดเล็กแทน Payload เต็มขนาดใหญ่ — ถ้า
   Request แบบนี้สำเร็จทันที (200/206 OK) แปลว่า Application Logic, Database Query, Authentication
   ทั้งหมดทำงานถูกต้องสมบูรณ์ (ไม่มี Bug ใน Business Logic) ปัญหาที่เหลือจึงต้องอยู่ที่การส่ง Payload
   **ขนาดใหญ่** ผ่าน Network เท่านั้น ซึ่งเป็นขอบเขตนอก Application ไปแล้ว
5. เพราะ Hypothesis ที่ดีที่สุด 2 ข้อผิดพลาดทั้งคู่มักบ่งบอกว่า **มุมมอง (Frame) ที่ใช้วิเคราะห์ปัญหา
   อยู่อาจผิดตั้งแต่ต้น** ไม่ใช่แค่ "ยังไม่เจอคำตอบที่ถูก" — การพยายามคิด Hypothesis ที่ 3-4-5 ต่อไป
   คนเดียวด้วย Frame เดิมมีโอกาสสูงที่จะวนอยู่ในกับดักความคิดเดิม ในขณะที่คนอื่นที่มาด้วยมุมมองใหม่
   (Fresh Eyes) มีโอกาสสูงกว่าที่จะเห็นสิ่งที่ถูกมองข้ามไปจากมุมที่แตกต่าง

---

## สรุป Part 91

Part นี้ไม่ได้สอน Technology ใหม่แม้แต่ตัวเดียว แต่สอน **กระบวนการคิด** ที่อยู่เบื้องหลังทุก Part
Troubleshooting ที่ผ่านมาตั้งแต่ [Part 56](part-056-eigrp-troubleshooting.md) ถึง
[Part 69](part-069-mega-lab-1.md): **Step 901** สอนให้เลือก Layer เริ่มต้นจาก Evidence/Pattern
ไม่ใช่ไล่ Bottom-Up เสมอไป, **Step 902** สอน Bisection ที่ลดจำนวน Test จาก O(N) เป็น O(log N),
**Step 903** สอนวงจรวิทยาศาสตร์ที่ป้องกัน Shotgun Troubleshooting ในระดับการวินิจฉัยเดี่ยว, **Step
904** สอนให้ใช้ Baseline Comparison (Git/Archive/Compliance-Check) แทนการไล่คิดจากศูนย์, **Step
905** สอนแปลงคำร้องเรียนคลุมเครือเป็นสมมติฐานที่ทดสอบได้ผ่านกรอบคำถาม 6 หมวด, **Step 906** สอนวาง
Packet Capture อย่างมีเป้าหมายด้วย SPAN/RSPAN/ERSPAN, **Step 907** สอน Triage เพื่อพิสูจน์ขอบเขต
ว่าใช่ปัญหา Network จริงหรือไม่ก่อนลงมือแก้, **Step 908** สอนวินัย Time-Boxing และกฎ Two-Strikes
สำหรับการ Escalate, **Step 909** สอนบันทึกระหว่างทางที่ป้องกันการ Test ซ้ำและเป็น Input ของ PIR,
และ **Step 910** รวมทุกอย่างเข้าด้วยกันใน Lab เดียวที่เริ่มจากประโยคคลุมเครือที่สุด แล้วไล่จนเจอ
Root Cause ที่แท้จริง (PMTUD Black Hole จาก Tunnel MTU + ACL Hardening ที่ Deny ICMP ทั้งหมด) โดย
ไม่มีใครบอกโดเมนให้ล่วงหน้าเลย — นี่คือทักษะที่ [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md)
เรียกว่า **Diagnose Module** และ **Part 95 (CCIE Full Lab Scenario #1 — Diagnose Module)** จะจำลอง
เต็มรูปแบบอีกครั้งด้วย Scenario ที่ไม่มีการเปิดเผยโดเมนล่วงหน้าเช่นกัน

✅ **พร้อมสำหรับ Part 92**: จาก Methodology การวินิจฉัยปัญหาที่เกิดขึ้นแล้ว เราจะขยับไปมองอีกฝั่งของ
เหรียญเดียวกัน — **Disaster Recovery & Network Resilience** คือการออกแบบระบบให้ **รับมือกับ Failure
ก่อนที่มันจะเกิด** และมี Plan ที่ชัดเจนสำหรับสถานการณ์ที่ร้ายแรงที่สุด (Site Failure, Data Center
ล่มทั้งหมด) ไม่ใช่แค่ Component เดียวเสียแบบที่ HA/FHRP ในภาคก่อนๆ รับมือได้อยู่แล้ว

**ไปต่อ:** [Part 92 — Disaster Recovery & Network Resilience →](part-092-disaster-recovery-network-resilience.md)
