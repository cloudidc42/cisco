# Part 72 — Large Campus Fabric Design (Layer 2 at Scale)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 711–720 จาก 1000**

> ต่อจาก [Part 71 — CCIE Lab Blueprint Overview](part-071-ccie-lab-blueprint-overview.md) ที่วางภาพรวม
> ของ CCIE Enterprise Infrastructure Lab Exam Blueprint ไว้แล้ว **Part นี้คือจุดเปลี่ยนสำคัญของหลักสูตร**
> ตั้งแต่ [Part 1](part-001-networking-fundamentals.md) จนถึง [Part 70](part-070-enarsi-capstone-mock-exam.md)
> เราสร้างและขยาย Lab เดียวกันอย่างต่อเนื่อง — CORE-SW1/2, DIST-SW1–4, ACCESS-SW1–4 รวมประมาณ
> 10 อุปกรณ์ Switching (ดู [`00-ip-address-plan.md`](00-ip-address-plan.md)) ซึ่งแทน **"1 อาคาร"**
> ของ Enterprise Campus จริงเท่านั้น — ทุก Design Pattern ที่เรียนมา (Dual-Homed Routed Uplink
> จาก [Part 33](part-033-advanced-stp-campus-design.md), StackWise Virtual/MEC จาก
> [Part 34](part-034-stackwise-vss-mec.md), SD-Access Fabric จาก
> [Part 38](part-038-sd-access-fundamentals.md)) ถูกพิสูจน์แล้วว่าใช้ได้ผลดีที่สเกลของ **1 อาคาร
> ~10-14 Switch** เท่านั้น
>
> **Part 72 จะยกระดับคำถามทั้งหมดขึ้นไปอีกขั้น**: ถ้า Headquarters ขององค์กรมี **10 อาคาร**
> แต่ละอาคารมี Access Switch **50 ตัว** (รวม 500 ตัวทั่วองค์กร) รองรับ Endpoint **หลักหมื่นตัว**
> — Design Pattern เดิมยังใช้ได้ไหม? ตรงไหนพังก่อน? และต้องเปลี่ยนแนวคิดอะไรบ้าง — นี่คือ
> **CCIE-level Design Thinking** ที่ไม่ใช่แค่ "Config เพิ่ม" แบบ Part ก่อนๆ แต่เป็นการ **ตัดสินใจ
> เชิงสถาปัตยกรรมที่มี Tradeoff จริง ไม่มีคำตอบเดียวที่ถูกต้อง 100%** — ทุก Step ในนี้จะจบด้วยคำถาม
> "แล้วจะเลือกอะไร และเพราะอะไร" ซึ่งเป็นรูปแบบคำถามที่ข้อสอบ CCIE Enterprise Infrastructure
> Written และ Design ส่วนของ Lab Exam ใช้จริง

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 711 | การขยายสเกลเกินกว่า 1 อาคาร — จาก Lab เดี่ยว (Part 1–70) สู่ Multi-Building Campus นับสิบอาคาร |
| 712 | ข้อจำกัดของ STP ที่สเกลใหญ่ — ทำไมการพึ่ง STP Blocking ใช้ไม่ได้อีกต่อไป |
| 713 | Routed Access ที่สเกลใหญ่ — ผัน Layer 3 ลงไปถึง Access Switch ทุกตัว, ข้อดี-ข้อเสีย |
| 714 | ออกแบบ VLAN ที่สเกลใหญ่ — "1 VLAN ต่อตู้ Rack" vs "ต่อชั้น" vs "ต่ออาคาร" และกฎ Broadcast Domain |
| 715 | ออกแบบรับ Traffic แบบ East-West vs North-South ในยุค Application สมัยใหม่ |
| 716 | การเชื่อมต่อระหว่างอาคาร (Multi-Building Interconnection) — Dark Fiber, Metro Ethernet, Microwave |
| 717 | Spine-Leaf ในฐานะ Campus Core ทางเลือก — เมื่อไหร่ควรใช้แทน 3-Tier Model ดั้งเดิม |
| 718 | การขยาย Wireless ระดับ Campus — WLC Placement Strategy และ AP Density Planning |
| 719 | ออกแบบ Failure Domain Isolation — จำกัดขอบเขตความเสียหายของทุก Failure |
| 720 | Lab ระดับ Design เต็มรูปแบบ: เอกสารออกแบบ Campus HQ 10 อาคาร ต่อจาก Lab เดิม (Building 1) |

---

## Step 711 — การขยายสเกลเกินกว่า 1 อาคาร

### ทบทวน: Lab ของเราคือ "1 อาคาร" มาตลอด โดยไม่มีใครพูดตรงๆ

ตั้งแต่ [Part 1 Step 10](part-001-networking-fundamentals.md) เราสร้าง Enterprise Lab Topology
ที่มี CORE-SW1/2, DIST-SW1–4, ACCESS-SW1–4 — ทั้งหมดนี้ **ไม่ได้ระบุไว้ชัดเจนว่าอยู่ที่ไหน**
แต่ในทางปฏิบัติ ขนาด Switch จำนวนนี้ (~10 ตัว, ผู้ใช้ไม่กี่ร้อยคน) ตรงกับสิ่งที่วิศวกร Design จริง
เรียกว่า **"1 Building" หรือ "1 IDF/MDF Zone"** ของ Enterprise Campus — คือขนาดที่ 3-Tier Model
เดี่ยวหนึ่งชุดรองรับได้พอดี

คำถามที่ CCIE-level Design ต้องตอบคือ: **ถ้าองค์กรเดียวกันมีอาคารแบบนี้ 10 อาคาร บนพื้นที่
Campus เดียวกัน (Headquarters) จะทำอย่างไร?** คำตอบที่ผิดที่สุดคือ **"Copy-Paste Design เดิม
10 รอบ แล้วเอา Core ของทุกอาคารมาต่อกัน"** — Part นี้จะแสดงให้เห็นทีละ Step ว่าทำไมคำตอบนั้น
ผิด และต้องแก้ไขตรงไหนบ้าง

### นิยามสเกลของ Part นี้: 10 อาคาร × 50 Access Switch

| พารามิเตอร์ | ค่า | ที่มาของตัวเลข |
|---|---|---|
| จำนวนอาคาร | 10 | Headquarters Campus ขนาดกลาง-ใหญ่ (ระดับ CCIE Lab Design scenario) |
| Access Switch ต่ออาคาร | 50 | ตู้ Rack/IDF ประมาณ 50 จุดต่ออาคาร (พื้นที่สำนักงานหลายชั้น) |
| Access Switch ทั้ง Campus | **500 ตัว** | 10 × 50 |
| พอร์ตต่อ Access Switch (ใช้งานจริง) | ~35 จาก 48 พอร์ต (~73%) | Provisioning จริงไม่เต็ม 100% เสมอ (สำรองสำหรับขยาย) |
| Endpoint แบบ Wired | **~17,500 ตัว** | 500 × 35 |
| Endpoint แบบ Wireless (โน้ตบุ๊ก/มือถือ, เฉลี่ย 1.5 ต่อ User) | **~15,000–20,000 ตัว** | ประเมินจาก Wired User Count |
| Endpoint IoT/Building System (กล้อง, Sensor, Access Control) | ~3,000–5,000 ตัว | ทั่วทั้ง 10 อาคาร |
| **รวม Endpoint ทั้ง Campus** | **~35,000–42,000 ตัว** | ตรงตามเป้าหมายของ Part นี้ ("หลักหมื่นตัว") |

### ทำไม "Copy-Paste 10 รอบ" ไม่ใช่ Design ที่ถูกต้อง

```
Design ที่ผิด (Naive Replication):

  Building1-CORE ── Building2-CORE ── Building3-CORE ── ... ── Building10-CORE
       (ต่อกันแบบ Ad-hoc ไม่มี Tier ใหม่ ไม่มี Summarization ไม่มีขอบเขต Failure Domain)

ปัญหาที่เกิดทันที:
1. ไม่มี Tier ใหม่รองรับการเชื่อมระหว่างอาคาร → Core ของอาคารหนึ่งต้องแบก Routing Table
   ของอีก 9 อาคารเต็มๆ โดยไม่มีการสรุป (Summarization)
2. ถ้า VLAN ใดเผลอถูก Trunk ข้ามอาคาร (ตามความเคยชินจาก Part 1-32) → STP Domain ทั้ง Campus
   กลายเป็นวงเดียว 500+ Switch (ดู Step 712 ว่าทำไมนี่คือหายนะ)
3. IP Addressing ชนกัน ถ้าแต่ละอาคาร Copy VLAN 10/20/30/40/99 = 10.10.x.0/24 เดิมทุกอาคาร
4. ไม่มีใครออกแบบ Failure Domain Boundary ไว้เลย — Broadcast Storm ที่อาคาร 7 กระทบอาคาร 1 ได้
```

### แนวคิดที่ถูกต้อง: เพิ่ม Tier ใหม่ + ขอบเขตชัดเจนทุกชั้น

Part นี้จะแนะนำ **Tier ใหม่** ที่ Part 1–70 ไม่มี เรียกว่า **Campus Backbone (หรือ "HQ Core")**
ซึ่งทำหน้าที่เชื่อม Core ของแต่ละอาคารเข้าด้วยกัน — โครงสร้างลำดับชั้นที่สมบูรณ์ของ Campus 10 อาคาร
จึงกลายเป็น **4 Tier** (เดิมมี 3 Tier ต่ออาคาร):

```
                         ┌─────────────────────────────┐
                         │   CAMPUS BACKBONE (ใหม่)       │  <- Tier ใหม่ที่ Part นี้เพิ่ม (Step 716-717)
                         │   เชื่อม Core ของทุกอาคาร      │     Routed ล้วน, Summarize ทุกเส้น
                         └──┬──────┬──────┬───────┬──────┘
                    ┌───────┘      │      │       └────────┐
              ┌─────┴─────┐  ┌─────┴─────┐ ...        ┌─────┴─────┐
              │ Building1  │  │ Building2  │            │ Building10 │
              │  CORE      │  │  CORE      │            │  CORE      │
              │ (= Lab เดิม│  │ (Replica   │            │ (Replica   │
              │  Part1-70) │  │  Pattern)  │            │  Pattern)  │
              └─────┬─────┘  └─────┬─────┘            └─────┬─────┘
              ┌──────┴──────┐┌──────┴──────┐            ┌──────┴──────┐
              │ DIST-SW×4    ││ DIST-SW×4    │            │ DIST-SW×4    │
              └──────┬──────┘└──────┬──────┘            └──────┬──────┘
              ┌──────┴──────┐┌──────┴──────┐            ┌──────┴──────┐
              │ ACCESS-SW×50 ││ ACCESS-SW×50 │            │ ACCESS-SW×50 │
              └─────────────┘└─────────────┘            └─────────────┘
```

**Building 1 ในแผนภาพนี้ = Lab เดิมทั้งหมดที่สร้างมาตั้งแต่ Part 1–70 พอดี** (CORE-SW1/2,
DIST-SW1–4, เพียงแต่ Access Layer ขยายจาก 4 ตัวเป็น 50 ตัวตามสัดส่วนของอาคารจริง) — ทุก Step
ถัดไปในนี้จะอ้างอิง Building 1 เป็นฐาน แล้วขยายแนวคิดออกไปอีก 9 อาคาร

> **ข้อสังเกตสำคัญสำหรับ CCIE Written/Design**: ตัวเลข "500 Switch" ไม่ใช่แค่ตัวเลขสวยๆ — มันคือ
> จุดที่ทำให้ Failure Domain, MAC Address Table Size (ค่า Default ของ Catalyst มักอยู่ที่
> 32,000–64,000 entry ต่อตัว — ไม่ใช่ปัญหาต่อ Switch เดี่ยว แต่เป็นปัญหาถ้า L2 Domain เดียวต้อง
> เรียนรู้ MAC ของ Endpoint ทั้ง 35,000+ ตัว), และ STP BPDU Processing Overhead (Step 712)
> กลายเป็นตัวแปรที่ต้อง Design อย่างจงใจ ไม่ใช่แค่ "เสียบสายแล้วมันทำงานได้" แบบ Lab ขนาดเล็ก

---

## Step 712 — ข้อจำกัดของ STP ที่สเกลใหญ่

### ทบทวน: STP ทำงานได้ดีมากที่สเกลของ Part 5/33 — แต่ทำไมจะพังที่สเกลนี้

[Part 5](part-005-stp-rstp-mstp.md) และ [Part 33 Step 325](part-033-advanced-stp-campus-design.md)
สรุปไว้ว่า STP (RSTP/Rapid-PVST+) ยังจำเป็นแค่ที่ **ขอบ Access↔Distribution** ของ **1 อาคาร**
(4 Access Switch, 4 Dist Switch) — ที่สเกลนี้ Convergence เร็ว (Sub-second ด้วย RSTP) และ
Diameter (จำนวน Hop สูงสุดจาก Root ถึงปลาย) มีแค่ 2-3 Hop เท่านั้น ไม่มีปัญหาอะไรเลย

ที่สเกล **50 Access Switch ต่ออาคาร** (ยังไม่ต้องพูดถึง 500 ตัวทั้ง Campus) ปัญหาไม่ใช่แค่
"Convergence ช้าลง" อย่างที่มักเข้าใจผิด — RSTP Proposal/Agreement Handshake ทำงานแบบ
Hop-by-Hop เร็วในตัวเองไม่ว่า Topology จะใหญ่แค่ไหน **ปัญหาจริงคือ Topology Change
Notification (TCN) และ MAC Table Flush ที่กระจายไปทั่วทั้ง L2 Domain**

### กลไกที่พังจริง: Topology Change กระเพื่อมไปทั่ว Domain

```
เหตุการณ์: Port บน Access Switch ตัวที่ 47 (สุดปลาย Domain) Flap (Up/Down ซ้ำๆ จาก NIC เสีย)

1. Access-47 ส่ง BPDU with Topology Change (TC) bit ขึ้นไปยัง Root ทันที
2. ทุก Switch บนเส้นทางไปยัง Root (Dist, ...) forward TC bit ต่อขึ้นไปเรื่อยๆ
3. Root Bridge ประกาศ TC ออกไป "ทุก Switch ใน STP Domain เดียวกันทั้งหมด" (Flood ทั่ว Domain)
4. ทุก Switch ที่ได้รับ TC จะ "ลด MAC Address Aging Timer ลงเหลือ Forward Delay (ปกติ 15 วิ)
   ชั่วคราว" เพื่อเรียนรู้ MAC ใหม่เร็วขึ้น — ผลข้างเคียงคือ MAC Table เดิมถูกล้างเร็วกว่าปกติ
5. ระหว่างที่ MAC Table ถูกล้าง Switch ทุกตัวใน Domain (ไม่ใช่แค่ Access-47) จะ Flood Frame
   แบบ Unknown Unicast ออกทุกพอร์ตชั่วคราว จนกว่าจะเรียนรู้ MAC ใหม่ครบ
```

**ผลลัพธ์**: ถ้า Access-47 Flap ซ้ำๆ (NIC เสียแบบไม่ตายสนิท, สาย LAN หลวม) **ทุก Switch ทั้ง
50 ตัวในอาคารเดียวกันที่อยู่ใน STP Domain เดียวกัน (Rapid-PVST+ Instance เดียวกัน) จะเจอ
Unknown-Unicast Flood ซ้ำๆ พร้อมกันหมด** แม้ Endpoint ส่วนใหญ่จะไม่ได้เกี่ยวข้องกับ Access-47
เลยก็ตาม — นี่คือ **"Broadcast/Flood Storm จาก TCN"** ที่ไม่มีทางเกิดขึ้นแบบมีนัยสำคัญที่สเกล
4 Access Switch ของ Lab เดิม แต่จะเกิดขึ้นจริงและกระทบผู้ใช้จำนวนมากที่สเกล 50 ตัว

### ตัวเลข BPDU Processing Overhead ที่ต้องคิดตอน Design

| รายการ | ที่สเกล Lab เดิม (4 Access) | ที่สเกล 1 อาคารใหม่ (50 Access) |
|---|---|---|
| จำนวน Switch ใน STP Domain เดียวกัน (ถ้า VLAN spanning เต็มอาคาร) | ~10 (4 Access + 4 Dist + 2 Core เดิม ก่อน Part 33 ตัด Core ออก) | ~54 (50 Access + 4 Dist) |
| BPDU Hello ที่แต่ละ Switch ต้องประมวลผลต่อ VLAN ต่อ 2 วินาที | น้อยมาก (2-3 พอร์ต Trunk/Switch) | มากขึ้นตามจำนวน Uplink/Downlink พอร์ตที่มี STP Active |
| Diameter จาก Root ถึงปลายสุด | 2-3 Hop | ยังคง 2-3 Hop เท่ากัน (Access→Dist ชั้นเดียว) — **นี่คือจุดที่มักเข้าใจผิด: Diameter ไม่ได้แย่ลงเพราะจำนวน Switch มากขึ้นในแนวขวาง** |
| ขอบเขตที่ TCN กระทบเมื่อ Port ใดก็ได้ Flap | Switch ทั้งหมดใน Domain (~10 ตัว) | **Switch ทั้งหมดใน Domain (~54 ตัว) — นี่คือสิ่งที่แย่ลงจริง** |
| CPU Load สำหรับคำนวณ STP ใหม่ทั้ง Domain เมื่อ Topology เปลี่ยน | ต่ำ | สูงขึ้นตามจำนวน Switch/Port ที่ต้องคำนวณ Role ใหม่พร้อมกัน |

> **ประเด็นสำคัญที่ CCIE ต้องเข้าใจให้ถูก**: Diameter (จำนวน Hop) ไม่ใช่ตัวแปรที่แย่ลงจากการเพิ่ม
> Access Switch ในแนวขวาง (เพิ่ม Access Switch คู่ขนานกันที่ Hop เดียวกัน) — ตัวแปรที่แย่ลงจริงคือ
> **"ขนาดของ Failure Domain / Blast Radius"** เมื่อ TCN เกิดขึ้น เพราะ TCN ถูก Flood ไปยัง **ทุก
> Switch ใน STP Instance เดียวกันเสมอ ไม่ว่า Switch ที่ Flap จะอยู่ห่างจาก Switch อื่นแค่ไหน**

### สรุปกฎการตัดสินใจ: STP Domain ต้อง "เล็กพอ" เสมอ ไม่ว่าจำนวน Switch ทั้ง Campus จะมากแค่ไหน

Cisco Best Practice (และแนวทางที่ข้อสอบ CCIE ใช้ตัดสิน) คือ **STP Domain หนึ่งชุดไม่ควรใหญ่กว่า
ขนาดที่ Convergence/TCN Blast Radius ยอมรับได้ — โดยทั่วไปหมายถึงไม่เกินขอบเขตของ Wiring
Closet กลุ่มเดียว หรืออย่างมากคือ 1 ชั้น/1 Zone ของอาคาร ไม่ใช่ทั้งอาคาร และไม่ใช่ทั้ง Campus
เด็ดขาด** ที่สเกล 50 Access Switch ต่ออาคารในหลักสูตรนี้ ถ้ายังใช้ Traditional Design (Part 33)
แบบเดิมทุกจุด (VLAN Trunk เต็มอาคาร) STP Domain จะมีขนาด ~54 Switch ซึ่งอยู่ในระดับที่เสี่ยงเกิน
ไปแล้วสำหรับสถานการณ์ Production จริงที่มี Port หลายพันจุด (ความเสี่ยงของ Human Error/สาย
เสีย ณ จุดใดจุดหนึ่งสูงขึ้นตามจำนวนพอร์ต) — นี่คือแรงจูงใจที่ทำให้ต้องพิจารณา **Routed Access**
อย่างจริงจังใน Step 713 ต่อไป ไม่ใช่แค่เป็นตัวเลือกทางทฤษฎีแบบที่ [Part 33 Step 322](part-033-advanced-stp-campus-design.md)
เคยบอกว่า "ยังไม่จำเป็นสำหรับ Lab นี้"

---

## Step 713 — Routed Access ที่สเกลใหญ่

### ทบทวนตารางเปรียบเทียบจาก Part 33 — แล้วใส่ตัวแปร "สเกล" เข้าไป

[Part 33 Step 322](part-033-advanced-stp-campus-design.md) เปรียบเทียบ Traditional Design กับ
Routed Access ไว้แล้ว และสรุปว่า Lab นี้ยังใช้ Traditional เพราะ **Hardware ของ ACCESS-SW1–4
เป็น Catalyst 9200/2960 (L2-only)** — ที่สเกล 500 Access Switch ทั้ง Campus คำตอบไม่เปลี่ยนแค่
เพราะ "อยากได้ Feature ใหม่" แต่เปลี่ยนเพราะ **STP Domain Blast Radius จาก Step 712 บีบให้ต้อง
พิจารณาใหม่อย่างจริงจัง**

| หัวข้อ | Traditional (VLAN Trunk เต็มอาคาร) ที่สเกล 50 Access/อาคาร | Routed Access ที่สเกล 50 Access/อาคาร |
|---|---|---|
| STP Domain Size | ~54 Switch ต่ออาคาร (เสี่ยงสูงตาม Step 712) | **0 — ไม่มี STP Domain ที่มีความหมายเลย** (ทุก Access Switch เป็น L3 Boundary ของตัวเอง) |
| Blast Radius เมื่อ Port ใดก็ได้ Flap | ทั้งอาคาร (54 Switch) | **เฉพาะ Access Switch ตัวนั้นตัวเดียว** |
| VLAN ID สามารถซ้ำกันข้าม Access Switch ได้ไหม | ไม่ได้ (ต้อง Unique ถ้า Trunk ถึงกัน) | **ได้เสมอ** — VLAN Significant แค่ในตัว Switch เดียว (ดู Step 714) |
| Hardware ที่ต้องใช้ | L2-only ราคาถูกได้ (Catalyst 9200/2960 แบบ Lab เดิม) | **ต้องเป็น Multilayer (Catalyst 9300 ขั้นต่ำ)** — เพิ่มต้นทุนต่อ Access Switch |
| VM Live Migration / Legacy L2 App | ทำได้ตรงไปตรงมา (Trunk เดิม) | **ทำไม่ได้ตรงๆ** — ต้องพึ่ง Overlay (SD-Access VXLAN/LISP, [Part 38](part-038-sd-access-fundamentals.md)) |
| Wireless Client Roaming ข้าม Access Switch | ทำได้ (Client อยู่ VLAN เดิมเสมอผ่าน CAPWAP กลับ WLC) | ต้องพึ่ง Overlay เช่นกัน มิฉะนั้น Client ต้องเปลี่ยน Subnet ตอน Roam |
| Multicast-Heavy Application (AV-over-IP, Building Automation) | ทำงานง่ายในวง L2 เดียว | ต้อง Config PIM/IGMP ข้าม Subnet ทุกจุด — ซับซ้อนขึ้น |

### ตัวอย่าง Config เชิงแนวคิด: Access Switch แบบ Routed Access

สมมติ Access Switch (Catalyst 9300) รับ Endpoint ของ VLAN เดียว (เช่น DATA) ทั้งตู้ Rack แล้ว
Route ออกไปเป็น Uplink แบบ Routed Port ทันที (ไม่มี Trunk ไปยัง Access Switch ตัวอื่นเลย):

```
! Access Switch ระดับ Routed Access — ไม่มี VLAN ใดถูก Trunk ออกไปยัง Access ตัวอื่นอีกต่อไป
ACCESS-B2-47(config)# vlan 10
ACCESS-B2-47(config-vlan)# name DATA
ACCESS-B2-47(config-vlan)# exit
ACCESS-B2-47(config)# interface vlan 10
ACCESS-B2-47(config-if)# ip address 10.20.147.1 255.255.255.0
ACCESS-B2-47(config-if)# no shutdown
ACCESS-B2-47(config-if)# exit

! Uplink ไป Distribution เป็น Routed Port เสมอ (หลักการเดียวกับ Part 10/33 ที่ Dist↔Core)
ACCESS-B2-47(config)# interface TenGigabitEthernet1/1/1
ACCESS-B2-47(config-if)# no switchport
ACCESS-B2-47(config-if)# ip address 10.20.247.2 255.255.255.252
ACCESS-B2-47(config-if)# no shutdown
ACCESS-B2-47(config-if)# exit

ACCESS-B2-47(config)# router ospf 1
ACCESS-B2-47(config-router)# network 10.20.147.0 0.0.0.255 area 20
ACCESS-B2-47(config-router)# network 10.20.247.0 0.0.0.3 area 20
```

สังเกตว่า **VLAN 10 บน ACCESS-B2-47 ตัวนี้ไม่มีทาง Trunk ไปยัง ACCESS-B2-48 ตัวข้างๆ ได้เลย
ตามหลักการ** — แต่ละ Access Switch คือ L3 Boundary ที่สมบูรณ์ในตัวเอง ผลคือ **ไม่มี STP
Instance ที่มีความหมายระหว่าง Access Switch เลยแม้แต่คู่เดียว** (PortFast/BPDU Guard ยังต้อง
เปิดที่พอร์ต User-facing เพื่อความปลอดภัยเท่านั้น ไม่ใช่เพื่อป้องกัน Loop ข้าม Access อีกต่อไป)

### Tradeoff ที่ต้องรับผิดชอบเมื่อเลือก Routed Access: Mobility ที่หายไป

ปัญหาที่แท้จริงของ Routed Access แบบ "ล้วนๆ" (ไม่มี Overlay) คือ **Endpoint ที่ต้องการ IP เดิม
แต่ย้ายที่เชื่อมต่อ** — สองกรณีที่พบบ่อยที่สุดในองค์กรจริง:

1. **VM vMotion/Live Migration**: Hypervisor Cluster ที่กระจาย VM ข้าม Physical Host หลายตัว
   (อาจอยู่ต่าง Rack/ต่าง Access Switch) ต้องให้ VM คง IP เดิมแม้ Host ย้าย — Routed Access ล้วน
   ทำไม่ได้เพราะ Subnet ผูกกับ Access Switch ตัวเดียว
2. **Wireless Client Roaming**: Client เดินข้าม AP ที่ต่อกับ Access Switch คนละตัว ต้องคง IP เดิม
   ระหว่าง Session (VoIP Call ค้าง, VPN Session ค้าง) — Routed Access ล้วนทำให้ Client ต้อง
   ขอ IP ใหม่ทุกครั้งที่ Roam ข้าม Subnet

**คำตอบสำหรับปัญหานี้ไม่ใช่ "กลับไปใช้ Traditional Design"** แต่คือ **SD-Access Fabric
(LISP + VXLAN)** ที่ [Part 38](part-038-sd-access-fundamentals.md) สอนไว้แล้ว — LISP แยก EID
(Identity) จาก RLOC (Location) ทำให้ Endpoint คง IP เดิมได้แม้ RLOC (ตำแหน่งทางกายภาพ) เปลี่ยน
โดยไม่ต้องมี VLAN Trunk ข้าม Access Switch เลยแม้แต่บรรทัดเดียว — **Routed Access + LISP/VXLAN
Overlay จึงเป็นคำตอบที่สมบูรณ์ที่สุดสำหรับสเกลนี้** ในขณะที่ Routed Access ล้วนๆ ไม่มี Overlay
เหมาะกับ Segment ที่ไม่มีความต้องการ Mobility เท่านั้น (เช่น Desktop สำนักงานทั่วไปที่ไม่ย้าย)

> **กฎการเลือกสำหรับ Design ที่สเกลนี้**: ใช้ **Routed Access เป็นค่า Default ของทุก Access
> Switch** (แก้ปัญหา STP Blast Radius จาก Step 712 ได้เด็ดขาด) แล้ว **เสริม Overlay (SD-Access)
> เฉพาะ Segment ที่ต้องการ Mobility จริง** (Wireless VLAN, Server Cluster ที่มี VM Migration)
> — ไม่ใช่การเลือกอย่างใดอย่างหนึ่งแบบสุดโต่งทั้ง Campus

---

## Step 714 — ออกแบบ VLAN ที่สเกลใหญ่

### สามแนวทางหลักในการกำหนดขอบเขต VLAN

| แนวทาง | คำอธิบาย | ขนาด Broadcast Domain โดยประมาณ |
|---|---|---|
| **1 VLAN ต่อตู้ Rack/IDF** | แต่ละ Access Switch (หรือ Stack) มี VLAN Data เป็นของตัวเอง | ~30-48 Host (เล็กที่สุด, ปลอดภัยที่สุด) |
| **1 VLAN ต่อชั้น (Floor)** | รวม Access Switch ทั้งชั้นเข้า VLAN เดียว (ต้อง Trunk ข้าม Access ในชั้นเดียวกัน) | ~150-400 Host ต่อชั้น (ขึ้นกับขนาดชั้น) |
| **1 VLAN ต่ออาคาร** | รวมทั้งอาคารเป็น VLAN เดียว (แบบที่ Lab เดิม Part 1-33 ทำ ตอนยังมีแค่ 4 Access Switch) | 1,000+ Host ที่สเกล 50 Access Switch — **เกินกฎ 500 Host มาก** |

### กฎ Broadcast Domain Sizing (Rule of Thumb)

> **กฎที่ใช้ในงาน Design จริงและข้อสอบ CCIE**: พยายามรักษาจำนวน Host ที่ Active ใน 1 Broadcast
> Domain (1 VLAN/1 Subnet) ให้ **ไม่เกินประมาณ 500 Host** — เหตุผลคือ ARP/Neighbor Discovery
> Broadcast, DHCP Broadcast, และ Unknown-Unicast Flood (จาก Step 712) ทั้งหมดนี้แปรผันตรงกับ
> จำนวน Host ในวงเดียวกัน ยิ่งวงใหญ่ ยิ่งมี Broadcast/Flood รบกวน CPU ของทุก Host และทุก Switch
> ใน Domain มากขึ้นตามสัดส่วน — Subnet ขนาด **/24 (254 Host ใช้งานได้จริง)** จึงเป็นจุดสมดุลที่
> นิยมมากที่สุดในทางปฏิบัติ (มีพื้นที่เผื่อเติบโตโดยยังไม่เกิน 500) ส่วน **/23 (510 Host)** ใช้ได้
> แต่เข้าใกล้เพดานแล้ว ควรใช้เฉพาะเมื่อมีเหตุผลชัดเจน (เช่น Wireless SSID ที่กระจาย AP จำนวนมาก
> ต้องการ Subnet ใหญ่กว่าปกติเพื่อลด Overhead การจัดการ)

### ตัวอย่างการวางแผน Capacity: 1 อาคาร มีผู้ใช้ 2,000 คน (Building ทั่วไปในหลักสูตรนี้)

| กลุ่ม Endpoint | จำนวนประมาณ | จำนวน Subnet /24 ที่ต้องการ (Rule: <500 Host/Subnet) |
|---|---|---|
| พนักงาน (Wired Desktop) | 800 | 2 Subnet (400/Subnet) หรือแยกตามชั้น 5 Subnet (~160/ชั้น) |
| Wireless Client (โน้ตบุ๊ก/มือถือ, 2 ต่อคน + Guest) | ~2,400 | 6 Subnet (400/Subnet) กระจายตามโซน AP |
| VoIP Phone (Voice VLAN) | 800 | 2 Subnet |
| IoT/Building System (กล้อง, Sensor, Access Control) | 200 | 1 Subnet |
| Server (Local ต่ออาคาร) | 50 | 1 Subnet |
| Management (OOB) | ~60 (1 ต่อ Switch) | 1 Subnet |
| **รวม VLAN/Subnet ต่ออาคาร** | | **~13 VLAN** (ปัดขึ้นเผื่อขยาย ~15-20 VLAN ต่ออาคาร) |

ด้วย Routed Access จาก Step 713 **VLAN ID เดิม (เช่น 10=DATA, 20=VOICE, 30=SERVERS, 40=WIFI,
99=MGMT ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md) กำหนดไว้แล้ว) สามารถใช้ซ้ำได้
ทุกชั้น ทุกอาคาร โดยไม่ชนกันเลย** เพราะ VLAN ไม่เคย Trunk ข้าม Access Switch แม้แต่คู่เดียว —
สิ่งที่ต้อง Unique จริงคือ **Subnet (IP Address)** เท่านั้น ไม่ใช่ VLAN ID (ดู Step 720 สำหรับ
IP Address Plan เต็มรูปแบบที่ใช้หลักการนี้)

### เทียบกับ Lab เดิม: ทำไม VLAN ต่ออาคาร (แบบ Part 1-33) ใช้ไม่ได้อีกแล้วที่สเกลนี้

Lab เดิมของหลักสูตร (VLAN 10 SALES ครอบคลุมทั้ง ACCESS-SW1+ACCESS-SW2 ผ่าน DIST-SW1/2) ใช้ได้
ดีที่สเกล **4 Access Switch** (รวมกันไม่เกิน ~200 Host) แต่ถ้าขยายเป็น 50 Access Switch ต่ออาคาร
โดยยังคง "VLAN เดียวทั้งอาคาร" แบบเดิม จะได้ Broadcast Domain ขนาด **1,000+ Host** ซึ่งเกินกฎ
500 Host ไปมาก — นี่คือเหตุผลเชิงตัวเลขที่สนับสนุนการเปลี่ยนไปใช้ Routed Access (Step 713) และ
แบ่ง VLAN ระดับ "ต่อชั้น" หรือ "ต่อตู้ Rack" (Step 714 นี้) ไปพร้อมกัน — ทั้งสอง Step นี้แก้ปัญหา
คนละมุมแต่เสริมกัน: Step 713 แก้ปัญหา **STP/Failure Domain**, Step 714 แก้ปัญหา **Broadcast
Domain Sizing** — องค์กรที่ทำถูกทั้งสองอย่างจะไม่มี VLAN ใดในทั้ง Campus ที่มี Host เกิน ~500 ตัว
เลยแม้แต่วงเดียว

---

## Step 715 — ออกแบบรับ Traffic แบบ East-West vs North-South

### สมมติฐานดั้งเดิมของ 3-Tier Model: North-South คือ Traffic หลัก

[Part 1 Step 1](part-001-networking-fundamentals.md) วาง 3-Tier Model (Core/Distribution/Access)
โดยมีสมมติฐานที่ไม่ได้พูดตรงๆ ว่า **Traffic ส่วนใหญ่วิ่งแนวตั้ง (North-South)**: Client ที่ Access
Layer คุยกับ Server ที่อยู่หลัง Core/Data Center — Uplink จาก Access→Dist→Core จึงถูกออกแบบให้
"หนาขึ้นเรื่อยๆ" ตามลำดับชั้น (Oversubscription แบบ 3-Tier ดั้งเดิม สมมติว่า Access↔Access ไม่ค่อย
คุยกันเอง)

```
Traditional Assumption (North-South Dominant):

        [Server/Data Center]
                 │
              CORE (Bandwidth หนาที่สุด)
                 │
          DISTRIBUTION (หนารองลงมา)
                 │
        ┌────────┼────────┐
     ACCESS1  ACCESS2  ACCESS3   <- Client ส่วนใหญ่คุยขึ้นไปหา Server อย่างเดียว
        │        │        │         ไม่ค่อยคุยกันเองข้าม Access (East-West น้อย)
       PC       PC       PC
```

### ความจริงในยุคปัจจุบัน: East-West เพิ่มขึ้นมาก

Application สมัยใหม่เปลี่ยนสัดส่วน Traffic ไปมาก ด้วยเหตุผลหลายอย่าง:

| แหล่งที่มาของ East-West Traffic | ตัวอย่าง |
|---|---|
| Microservices/API เรียกกันเองระหว่าง Service | Web Tier → App Tier → Database Tier ที่กระจายอยู่หลาย Server/Container |
| Storage Traffic | iSCSI/NFS ระหว่าง Compute กับ Storage Array (มักอยู่ใน Data Center Building) |
| Container/Kubernetes Pod-to-Pod | Pod บน Node ต่างกัน คุยกันผ่าน Overlay Network ตลอดเวลา |
| Distributed Database Replication | Database Cluster Sync ข้าม Node หลายตัวพร้อมกัน |
| Video Collaboration (SFU Mesh) | Client คุยผ่าน Media Server ที่อาจอยู่ On-prem แทน Cloud ทั้งหมด |
| IoT/Building Automation | Sensor → Controller → Actuator คุยกันเองในอาคารเดียวกัน ไม่ผ่าน Core เลยในทางทฤษฎี แต่ Traditional Design บีบให้ต้องผ่าน |

### ผลกระทบต่อ Design ที่สเกล Campus 10 อาคาร

ปัญหาคือ **3-Tier Model ดั้งเดิมบีบให้ East-West Traffic ทุกเส้นต้อง "Hairpin" ขึ้นไปถึง
Distribution หรือ Core ก่อนแล้วค่อยวกกลับลงมา** แม้ Endpoint สองตัวจะอยู่ใกล้กันมากทางกายภาพ
(เช่น Server สองตัวอยู่ Rack ติดกัน แต่เสียบคนละ Access Switch) — ที่สเกลเล็ก (Lab เดิม) Overhead
นี้เล็กน้อยจนไม่มีใครสังเกต แต่ที่สเกล 500 Access Switch ถ้ามี Traffic Pattern แบบ Data
Center (Server-to-Server จำนวนมาก) อยู่ในบางอาคาร (เช่นอาคารที่เป็น Data Center ของ Campus)
การ Hairpin ผ่าน 3-Tier ทำให้:

1. **Latency เพิ่มขึ้นไม่จำเป็น** — ทุก East-West Flow ต้องผ่าน Dist+Core (4-6 Hop) แทนที่จะ
   ผ่านแค่ 1-2 Hop ถ้าออกแบบให้ Server อยู่ใกล้กันในเชิง Topology ด้วย
2. **Core/Distribution รับ Load เกินความจำเป็น** — Bandwidth ที่ควรใช้แค่ระหว่าง Access
   สองตัวถูกบีบให้วิ่งผ่าน Uplink ที่ออกแบบไว้สำหรับ North-South เท่านั้น

### แนวทางแก้: แยก Topology ตาม Traffic Pattern ของแต่ละอาคาร ไม่ใช่ใช้สูตรเดียวทั้ง Campus

```
อาคารที่เป็น Office ทั่วไป (Traffic ส่วนใหญ่ North-South)   อาคารที่เป็น Data Center/Server Farm
       → ใช้ 3-Tier + Routed Access ตามเดิม (Step 713)         (Traffic ส่วนใหญ่ East-West)
                                                                  → ควรพิจารณา Spine-Leaf (Step 717)
```

นี่คือเหตุผลที่ Step 717 จะแนะนำ **Spine-Leaf** ในฐานะ Topology ทางเลือกสำหรับอาคารที่มี
Traffic Pattern แบบ East-West จำนวนมาก (โดยเฉพาะ "Building 10" ในเอกสาร Design ของ Step 720
ที่จะกำหนดให้เป็น Data Center Building โดยเฉพาะ) แทนที่จะบังคับใช้ 3-Tier แบบเดียวกันทุกอาคาร

---

## Step 716 — การเชื่อมต่อระหว่างอาคาร (Multi-Building Interconnection)

### สามตัวเลือกหลักในการเชื่อม Core ของแต่ละอาคารเข้ากับ Campus Backbone

| เทคโนโลยี | Bandwidth ทั่วไป | Latency | ค่าใช้จ่าย | ความเป็นเจ้าของ/ควบคุม | ข้อจำกัด |
|---|---|---|---|---|---|
| **Dark Fiber** (เช่าหรือเป็นเจ้าของสายไฟเบอร์ดิบ) | 10G-100G+ (จำกัดแค่ Optic ที่เลือก) | ต่ำที่สุด (ไม่มี Switching กลางทาง) | สูง (ค่าเดินสาย/ค่าเช่า Conduit) แต่คงที่ระยะยาว | สูงสุด — องค์กรควบคุม Layer 1-2 เต็มรูปแบบ | ต้องมี Conduit/สิทธิ์เดินสายระหว่างอาคารจริง (มักมีอยู่แล้วในพื้นที่ Campus เดียวกัน) |
| **Metro Ethernet** (บริการจาก ISP เช่น E-Line/E-LAN) | 1G-10G (ตาม SLA ที่ซื้อ) | ปานกลาง (ผ่าน Provider Network) | ค่าบริการรายเดือนต่อเนื่อง (OPEX) | ต่ำ — พึ่ง SLA ของ Provider | เหมาะกับกรณีอาคารอยู่ไกลกันเกินระยะ Fiber ที่องค์กรเดินเองได้ หรือข้ามที่ดินคนอื่น |
| **Point-to-Point Microwave** | 100M-10G (ตามคลื่นความถี่/ระยะทาง) | ต่ำ (Line-of-Sight ตรง) | ปานกลาง (ค่าอุปกรณ์ครั้งเดียว + License คลื่นความถี่บางกรณี) | สูง (องค์กรควบคุมเอง) | ขึ้นกับสภาพอากาศ (ฝน/หมอกลดทอนสัญญาณ), ต้อง Line-of-Sight ตรง (ตึกบัง = ใช้ไม่ได้) |

> **คำแนะนำสำหรับ Campus ในพื้นที่เดียวกัน (เช่น 10 อาคารในบริเวณ Headquarters เดียวกัน)**:
> **Dark Fiber คือตัวเลือกที่ถูกต้องที่สุดในเกือบทุกกรณี** เพราะองค์กรมักมี Conduit ระหว่างอาคาร
> อยู่แล้ว (หรือขุดเพิ่มได้ในพื้นที่ของตัวเอง) และให้ Bandwidth/Latency ดีที่สุดโดยไม่มีค่าบริการ
> รายเดือนระยะยาว — **Metro Ethernet เหมาะกับกรณีอาคารอยู่นอกพื้นที่ Campus เดียวกัน** (เช่น
> สาขาอีกฝั่งเมือง) และ **Microwave เหมาะเป็น Backup Path เสริม** ให้ Dark Fiber ในกรณีฉุกเฉิน
> (สาย Fiber ถูกขุดขาดจากอุบัติเหตุ) มากกว่าใช้เป็น Primary Path

### กฎเหล็ก: Conduit Diversity — ห้าม Physical Path ซ้ำกันเด็ดขาด

ปัญหาที่พบบ่อยที่สุดในงานจริง (และเป็นคำถาม CCIE Design ที่ชอบถามสับสน) คือ องค์กรซื้อสาย
Fiber "สองเส้น" เพื่อ Redundancy แต่ **เดินสายทั้งสองเส้นไปในท่อร้อยสาย (Conduit) เดียวกัน**
ทำให้ถ้ามีการขุดถนน/ก่อสร้างตัดท่อนั้นโดยไม่ตั้งใจ **สาย Fiber ทั้งสองเส้นขาดพร้อมกัน** —
Redundancy ที่ซื้อมาไม่มีประโยชน์อะไรเลย

```
ผิด (Conduit เดียวกัน):                       ถูก (Conduit แยกกันจริง — Physical Diversity):

Building A ══[Fiber1+Fiber2]══ Building B     Building A ══[Fiber1]══(เส้นทางเหนือ)══ Building B
            (Conduit เดียวกัน)                Building A ══[Fiber2]══(เส้นทางใต้)════ Building B
   ขุดท่อนี้ตัด = ขาดสื่อสารทั้งคู่                ขุดท่อเส้นเหนือ = ยังมี Fiber2 เส้นใต้ใช้งานได้
```

### ออกแบบ Topology ระหว่างอาคาร: Ring ดีกว่า Hub-and-Spoke ที่สเกลนี้

| Topology | ข้อดี | ข้อเสีย |
|---|---|---|
| **Hub-and-Spoke** (ทุกอาคารต่อเข้า Hub อาคารเดียว) | เข้าใจง่าย, Config น้อยจุด | **Hub คือ Single Point of Failure ของทั้ง Campus** — ถ้า Hub ล่ม อาคารทั้งหมดขาดการเชื่อมต่อกัน |
| **Full Mesh** (ทุกอาคารต่อกันหมดทุกคู่) | ทนทานที่สุด, ไม่มี Single Point of Failure เลย | ค่าใช้จ่ายสูงมาก — 10 อาคาร = C(10,2) = 45 เส้น Fiber |
| **Ring (แต่ละอาคารต่อ 2 อาคารข้างเคียง)** | ทนทานดี (Fiber ขาด 1 จุด = Reroute ผ่านอีกทาง), ค่าใช้จ่ายสมเหตุสมผล (10 เส้นเท่านั้นสำหรับ 10 อาคาร) | ถ้า Fiber ขาด 2 จุดพร้อมกันคนละที่ อาจมีอาคารถูกตัดขาดได้ (ความเสี่ยงต่ำ ยอมรับได้ในทางปฏิบัติ) |

**คำแนะนำสำหรับ 10 อาคาร**: ใช้ **Ring Topology** เป็นฐาน (แต่ละอาคารต่อ Dark Fiber ไปยัง
อาคารข้างเคียง 2 อาคารเสมอ ปิดเป็นวงแหวนสมบูรณ์) แล้วเสริมด้วย **Chord Link** (เส้นลัดผ่านกลาง
Ring) 2-3 เส้นระหว่างอาคารที่มี Traffic สูงเป็นพิเศษ (เช่น Data Center Building) เพื่อลด Hop
Count และเพิ่ม Bandwidth เฉพาะจุด — ดู Diagram เต็มรูปแบบใน Step 720

```
                    Building1 ══════ Building2
                   ╱                        ╲
            Building10                  Building3
                 ║                            ║
            Building9                    Building4
                   ╲                        ╱
                    Building8 ══════ Building5
                          ╲          ╱
                       Building7══Building6

  (Ring หลัก 10 เส้น + Chord เสริม เช่น Building1↔Building6 สำหรับ Traffic สูง — ดู Step 720)
```

แต่ละเส้นในวงแหวนนี้ควรทำ **Dual-Homed Routed Uplink + OSPF ECMP แบบ Part 33** ในระดับ
Building-to-Backbone (ไม่ใช่แค่ Dist↔Core ภายในอาคารเดียวอีกต่อไป) เพื่อให้ Core ของแต่ละอาคาร
มีเส้นทางสำรองไปยัง Campus Backbone อย่างน้อย 2 เส้นทางเสมอ — รายละเอียด IP Addressing ของ
เส้นเหล่านี้อยู่ใน Step 720

---

## Step 717 — Spine-Leaf ในฐานะ Campus Core ทางเลือก

### ทบทวนแนวคิด Spine-Leaf จาก Data Center Design (สรุปสั้น — เจาะลึกที่ Part 84-85)

**Spine-Leaf** คือสถาปัตยกรรมที่ยืมมาจาก Data Center Design: มี **Spine Switch** จำนวนหนึ่ง
(มักเป็นเลขคู่ 2-4 ตัว) ที่ **Leaf Switch ทุกตัวต้องต่อไปหา Spine ทุกตัว** (Full Mesh ระหว่าง
สอง Layer นี้เท่านั้น — Leaf ไม่ต่อกันเองโดยตรงเด็ดขาด, Spine ไม่ต่อกันเองโดยตรงเด็ดขาด)

```
                    SPINE1        SPINE2
                   ╱  │  ╲       ╱  │  ╲
                  ╱   │   ╲     ╱   │   ╲
              LEAF1  LEAF2  LEAF3  LEAF4    <- ทุก Leaf ต่อทุก Spine (Full Mesh 2 Layer)
                                              ไม่มี Leaf↔Leaf โดยตรง, ไม่มี Spine↔Spine โดยตรง
```

คุณสมบัติสำคัญ: **ทุก Path จาก Leaf หนึ่งไปยัง Leaf อื่นมี Hop Count เท่ากันเสมอ (Leaf→Spine→
Leaf = 2 Hop พอดี ไม่ว่า Leaf จะอยู่ตำแหน่งไหน)** ทำให้ไม่มีปัญหา Oversubscription สะสมแบบ
3-Tier (ที่ต้อง Hairpin ผ่าน Distribution→Core→Distribution เมื่อ Traffic ข้าม Zone) และรองรับ
ECMP เต็มรูปแบบทุกคู่ Leaf โดยธรรมชาติ — เป็น Topology ที่ Design มาสำหรับ **East-West Traffic
จำนวนมาก** ตรงตามที่ Step 715 พูดถึง

### เมื่อไหร่ Spine-Leaf เหมาะกับ Campus (ไม่ใช่แค่ Data Center)

| ปัจจัย | เหมาะกับ 3-Tier (Core/Dist/Access) | เหมาะกับ Spine-Leaf |
|---|---|---|
| Traffic Pattern หลัก | North-South (Client→Server ผ่าน Hierarchy) | East-West จำนวนมาก (Server↔Server, Storage) |
| จำนวนจุดที่ต้อง Aggregate (Leaf/Building) | น้อย-กลาง (ไม่กี่สิบจุด) | มาก และมีแผนขยายต่อเนื่อง (เพิ่ม Leaf ทีละตัวโดยไม่กระทบ Topology เดิม) |
| ความต้องการ Latency คงที่ (Predictable) ทุกคู่ปลายทาง | ไม่จำเป็นมาก (Client ทนความหน่วงต่างกันได้) | จำเป็นมาก (Database Cluster, Storage Replication ไวต่อ Latency ที่ไม่สม่ำเสมอ) |
| ความคุ้นเคยของทีมงาน/ค่าใช้จ่าย License | สูง (ใช้มาตั้งแต่ Part 1 ของหลักสูตร, Skill หาง่าย) | ต่ำกว่า (ต้องมี Skill Data Center Fabric เพิ่ม, มักต้องใช้ Platform ระดับ Data Center) |
| STP Involvement | มีที่ขอบ Access เท่านั้น (Step 712-713 ลดขนาดแล้ว) | **ไม่มีเลย** — ทุกลิงก์ Routed ล้วน (เหมือน Part 33/34 แต่ขยายทั้ง 2 Layer) |

### คำแนะนำสำหรับ Campus 10 อาคารของหลักสูตรนี้

**อาคารส่วนใหญ่ (สำนักงานทั่วไป, Traffic North-South เป็นหลัก) ควรใช้ 3-Tier + Routed Access
ตามที่ Step 711-714 วางไว้แล้ว** เพราะ Traffic Pattern ไม่ได้ต้องการ Spine-Leaf จริงๆ และทีมงาน
คุ้นเคยกับ 3-Tier มากกว่า (ต้นทุนการเรียนรู้ต่ำกว่า) — **แต่สำหรับอาคารที่ทำหน้าที่เป็น Data
Center/Server Farm ของ Campus (ในเอกสาร Design ของ Step 720 จะกำหนดให้เป็น "Building 10")
ควรใช้ Spine-Leaf แทน** เพราะ Traffic ภายในอาคารนั้นเป็น East-West (Server-to-Server,
Storage) เป็นหลักตามที่ Step 715 อธิบายไว้

```
Campus Backbone (Step 716)
        │
   ┌────┴─────────────────────────────────┐
   │                                        │
Building 1-9 (3-Tier ตามเดิม)      Building 10 = Data Center
Core→Dist→Access                    ┌──────────────────┐
(North-South Dominant)               │  SPINE1    SPINE2  │
                                      │  ╱ │ ╲    ╱ │ ╲    │
                                      │LEAF1 LEAF2 LEAF3   │  <- แต่ละ Leaf = Server Rack
                                      └──────────────────┘     (East-West Dominant)
```

> **สรุปสำหรับข้อสอบ CCIE Design**: คำถามแนว "จะใช้ 3-Tier หรือ Spine-Leaf" **ต้องดูที่ Traffic
> Pattern ของพื้นที่นั้นเป็นหลัก** เหมือนที่ Step 322 ของ Part 33 บอกว่าต้องดู Hardware ก่อนเลือก
> Routed Access — **ไม่มี Topology ใดที่ถูกต้อง 100% สำหรับทั้ง Campus** ในองค์กรขนาดใหญ่จริง
> มักเห็น 3-Tier และ Spine-Leaf อยู่ร่วมกันในเครือข่ายเดียวกันคนละโซนแบบนี้เสมอ

---

## Step 718 — การขยาย Wireless ระดับ Campus

### สองโมเดลหลักของ WLC Placement

| โมเดล | หลักการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Centralized** | WLC คู่เดียว (Active/Standby SSO) วางที่ Campus Backbone ให้บริการ AP ทุกตัวทั้ง 10 อาคารผ่าน CAPWAP Tunnel | Config/Management จุดเดียว, RRM (Radio Resource Management) เห็นภาพรวมทั้ง Campus ช่วยจัด Channel/Power ได้ดีกว่า | **Traffic Tromboning** ทวีความรุนแรงขึ้นตามสเกล (ตามที่ [Part 38 Step 378](part-038-sd-access-fundamentals.md) เคยเตือนไว้) — Client Data Traffic ทุก Byte จากทุกอาคารต้องเดินทางไป-กลับ Campus Backbone แม้ปลายทางจะอยู่อาคารเดียวกัน |
| **Distributed** | WLC หลายคู่ กระจายตามกลุ่มอาคาร (เช่น 1 คู่ WLC ต่อทุก 3-4 อาคาร) | ลด Tromboning — Traffic ส่วนใหญ่ Local ในกลุ่มอาคารเดียวกัน, Blast Radius เล็กลงถ้า WLC คู่หนึ่งล่ม (กระทบแค่กลุ่มอาคารของตัวเอง) | Management ซับซ้อนขึ้น (หลายจุดต้อง Sync RF/Policy), ต้นทุน License/Hardware เพิ่มตามจำนวนคู่ |

### ตัวเลข Ceiling ของ WLC ที่ต้องคำนวณตอน Design

| Platform (ตัวอย่าง) | AP สูงสุดที่รองรับ | Client สูงสุดที่รองรับ |
|---|---|---|
| Catalyst 9800-40 | ~2,000 AP | ~40,000 Client |
| Catalyst 9800-80 | ~6,000 AP | ~64,000 Client |

**คำนวณสำหรับ Campus 10 อาคาร**: ถ้าแต่ละอาคารมี AP เฉลี่ย ~100 ตัว (ครอบคลุมพื้นที่สำนักงาน
ทั่วไป) รวมทั้ง Campus = **~1,000 AP** และ Client รวม (ตามตัวเลข Step 711) = **~35,000-42,000
ตัว** — ตัวเลขนี้ **อยู่ใต้ Ceiling ของ WLC ระดับใหญ่ 1 คู่ (9800-80) ได้พอดี ในทางทฤษฎี** แต่
Cisco ไม่แนะนำให้ใช้ WLC เพียง 1 คู่รับภาระทั้ง Campus จริง เพราะ:

1. **Traffic Tromboning ยังเป็นปัญหาอยู่** แม้ Ceiling ทางเทคนิครองรับได้
2. **Single Point of Software Risk** — ถ้า WLC Pair เดียวมีปัญหา (Bug, Reload) ทั้ง 10 อาคาร
   ไม่มี Wi-Fi พร้อมกันหมด

### คำแนะนำสำหรับ Campus 10 อาคารของหลักสูตรนี้

ใช้ **Distributed WLC** แบ่งเป็น **3 กลุ่ม** (ประมาณ 3-4 อาคารต่อกลุ่ม, 3 คู่ WLC ทั้ง Campus)
โดยแต่ละคู่ WLC วาง **Active ที่อาคารหนึ่งในกลุ่ม, Standby (SSO) ที่อาคารอื่นในกลุ่มเดียวกัน**
(Geographic Redundancy ระดับอาคาร — ถ้าอาคารที่มี Active WLC ไฟดับทั้งอาคาร Standby ที่อาคาร
อื่นยัง Take Over ได้) — โมเดลนี้สอดคล้องกับแนวทาง **Fabric-mode WLC** ของ SD-Access
([Part 38 Step 372](part-038-sd-access-fundamentals.md)) ที่แนะนำให้ WLC ทำงานใกล้ชิดกับกลุ่ม
Edge Node ของตัวเองอยู่แล้ว

### AP Density Planning — สูตรคร่าวๆ (รายละเอียดเต็มที่ Part 89)

AP Density ไม่ได้ออกแบบจาก "พื้นที่ครอบคลุมสัญญาณ (Coverage)" เพียงอย่างเดียวอีกต่อไปในงาน
สมัยใหม่ — ต้องคิดเรื่อง **Capacity (จำนวน Client ต่อ AP)** ควบคู่กันเสมอ:

```
ตัวอย่าง: ชั้นสำนักงานขนาด 2,000 ตร.ม. มีผู้ใช้พร้อมกัน 300 คน (Concurrent)

ถ้าคิดแบบ Coverage-only (AP ครอบคลุมพื้นที่ ~150-200 ตร.ม./ตัว):
   ต้องการ AP ≈ 2,000 / 175 ≈ 12 ตัว

ถ้าคิดแบบ Capacity (Voice/Data-grade แนะนำ ~20-25 Client/AP สำหรับงานทั่วไป):
   ต้องการ AP ≈ 300 / 20 ≈ 15 ตัว

ค่าที่ใช้จริง = max(Coverage, Capacity) = 15 ตัว (Capacity เป็นตัวกำหนดในกรณีนี้)
```

> **หมายเหตุ**: การคำนวณ AP Density แบบเต็มรูปแบบต้องพิจารณา RF Survey จริง (Attenuation จาก
> ผนัง/เฟอร์นิเจอร์, Channel Planning ลด Co-Channel Interference, ความหนาแน่นพิเศษสำหรับพื้นที่
> High-Density เช่น ห้องประชุม/Auditorium) ซึ่งเป็นเนื้อหาเจาะลึกที่ **จะเรียนแบบเต็มรูปแบบใน
> Part 89** — Step นี้ให้แค่หลักการระดับ Design overview เพื่อวางแผน WLC Capacity (Step ข้างต้น)
> ให้ถูกต้องเท่านั้น

---

## Step 719 — ออกแบบ Failure Domain Isolation

### หลักการ: จำกัด "Blast Radius" ของทุก Failure โดยจงใจ

Step 712 แสดงให้เห็นแล้วว่า TCN Storm จาก STP กระทบทั้ง Domain — หลักการทั่วไปที่ CCIE-level
Design ต้องยึดถือคือ **ทุก Failure ที่เป็นไปได้ (Bad BPDU, Broadcast Storm, DHCP Server
ปลอมที่พนักงานเสียบผิด, Loop จาก Human Error) ต้องถูกจำกัดขอบเขตความเสียหายไว้ด้วย "กำแพง"
ที่ออกแบบไว้ล่วงหน้า ไม่ใช่หวังว่าจะไม่เกิดขึ้น**

### เครื่องมือ 4 ชนิดที่ใช้สร้างกำแพงเหล่านี้

| เครื่องมือ | จำกัด Failure ประเภทไหน | ระดับที่ Enforce |
|---|---|---|
| **L3 Boundary (Routed Access, Step 713)** | STP TCN Storm, Broadcast Storm, Loop จาก Human Error | 1 Access Switch หรือเล็กกว่า |
| **Storm-Control** | Broadcast/Multicast/Unknown-Unicast Flood ที่ล้น (แม้อยู่ใน L2 Domain เดียวเดิม) | 1 พอร์ตต่อพอร์ต |
| **Private VLAN** | จำกัดไม่ให้ Endpoint ในวงเดียวกันคุยกันเอง (แม้อยู่ Subnet เดียวกัน) | 1 พอร์ตต่อพอร์ตภายใน 1 VLAN |
| **DHCP Snooping + IP Source Guard** | Rogue DHCP Server, IP Spoofing | 1 พอร์ต/1 VLAN |

### Config จริง: Storm-Control บนพอร์ต Access ทุกจุด

```
ACCESS-B2-47(config)# interface range GigabitEthernet1/0/1 - 48
ACCESS-B2-47(config-if-range)# storm-control broadcast level 1.00 2.00
ACCESS-B2-47(config-if-range)# storm-control multicast level 5.00 10.00
ACCESS-B2-47(config-if-range)# storm-control action trap
ACCESS-B2-47(config-if-range)# storm-control action shutdown
```

- `storm-control broadcast level 1.00 2.00` — เริ่ม Drop เมื่อ Broadcast Traffic เกิน 1% ของ
  Bandwidth พอร์ต และกลับสู่ปกติเมื่อลดลงต่ำกว่า 2% (Rising/Falling Threshold)
- `action trap` ส่ง SNMP Trap แจ้งเตือนทันทีที่ Threshold ถูกกระทบ (สำคัญมากสำหรับทีม NOC
  ให้รู้ตัวก่อนที่ผู้ใช้จะโทรมาบ่น)
- `action shutdown` (เสริม) — ปิดพอร์ตอัตโนมัติถ้า Storm รุนแรงต่อเนื่อง เพื่อป้องกันไม่ให้
  Access Switch ตัวนั้นส่ง Storm ต่อขึ้นไปยัง Uplink (แม้ Uplink เป็น Routed Port ที่ไม่มี STP
  แล้วก็ตาม — Storm ที่ Flood เป็น Broadcast ยังวิ่งผ่าน Routed Port ได้ถ้าไม่ตัดที่ต้นทาง)

### Config จริง: Private VLAN แยก IoT Device ไม่ให้คุยกันเอง

สถานการณ์: กล้อง IoT 30 ตัวอยู่ Subnet เดียวกัน (10.20.180.0/24) แต่ต้องการห้ามไม่ให้กล้องคุยกันเอง
โดยตรง (ป้องกัน Malware กระจายตัวข้ามกล้อง) ให้คุยได้แค่กับ Gateway/NVR Server เท่านั้น:

```
ACCESS-B2-IOT(config)# vlan 180
ACCESS-B2-IOT(config-vlan)# private-vlan primary
ACCESS-B2-IOT(config-vlan)# exit
ACCESS-B2-IOT(config)# vlan 181
ACCESS-B2-IOT(config-vlan)# private-vlan isolated
ACCESS-B2-IOT(config-vlan)# exit
ACCESS-B2-IOT(config)# vlan 180
ACCESS-B2-IOT(config-vlan)# private-vlan association 181
ACCESS-B2-IOT(config-vlan)# exit

! พอร์ตกล้อง = Isolated Host (คุยได้แค่กับ Promiscuous Port เท่านั้น)
ACCESS-B2-IOT(config)# interface range GigabitEthernet1/0/1 - 30
ACCESS-B2-IOT(config-if-range)# switchport mode private-vlan host
ACCESS-B2-IOT(config-if-range)# switchport private-vlan host-association 180 181

! พอร์ตของ NVR/Gateway = Promiscuous Port (คุยกับทุก Isolated Port ได้)
ACCESS-B2-IOT(config)# interface GigabitEthernet1/0/48
ACCESS-B2-IOT(config-if)# switchport mode private-vlan promiscuous
ACCESS-B2-IOT(config-if)# switchport private-vlan mapping 180 181
```

### เปรียบเทียบ "Blast Radius" ก่อน-หลังใช้เครื่องมือเหล่านี้ร่วมกัน

```
ก่อน (Flat L2 ทั้งอาคาร, ไม่มี Storm-Control):
  Port 1 เสีย/ถูกโจมตี → Broadcast Storm/TCN → กระทบทั้ง 50 Access Switch ในอาคาร (Step 712)

หลัง (Routed Access + Storm-Control + Private VLAN):
  Port 1 เสีย/ถูกโจมตี → Storm-Control ตัดที่พอร์ตนั้นภายในไม่กี่วินาที
                       → แม้ Storm หลุดผ่านไปได้ ก็ติดอยู่แค่ VLAN เดียวบน Access Switch ตัวเดียว
                       → Routed Uplink (Step 713) กันไม่ให้ Broadcast วิ่งขึ้นไปยัง Distribution เลย
                       → 49 Access Switch ที่เหลือในอาคาร + อีก 9 อาคาร ไม่รู้ตัวว่าเกิดอะไรขึ้น
```

> **ข้อสรุปสำหรับ CCIE Design**: Failure Domain Isolation ไม่ใช่ Feature เดียว แต่เป็น
> **ผลลัพธ์ที่เกิดจากการเลือก Topology ที่ถูกต้อง (Routed Access, Step 713) ร่วมกับ Feature
> เสริมที่ Enforce ที่ขอบ (Storm-Control, Private VLAN, DHCP Snooping)** — ถ้าเลือก Topology
> ผิดตั้งแต่ต้น (Flat L2 ทั้งอาคาร) Feature เสริมเหล่านี้ช่วยได้แค่บางส่วน ไม่สามารถทดแทนการ
> ออกแบบ L3 Boundary ที่ถูกต้องได้เลย

---

## Step 720 — Lab ระดับ Design เต็มรูปแบบ: เอกสารออกแบบ Campus HQ 10 อาคาร

> Step นี้คือ Deliverable หลักของ Part 72 — เอกสารออกแบบฉบับสมบูรณ์ที่วิศวกร CCIE ระดับ Design
> จริงจะผลิตให้ผู้บริหาร/ทีม Implementation ใช้อ้างอิง โดยต่อยอดจาก Lab ที่สร้างมาตลอด Part 1-70
> (= **Building 1** ในเอกสารนี้) และใช้หลักการทั้งหมดจาก Step 711-719 ข้างต้น

---

### เอกสารออกแบบ: Headquarters Campus Network — 10-Building Design

**เวอร์ชัน**: 1.0 | **สถานะ**: Draft for Review | **ขอบเขต**: Layer 2/3 Topology & Addressing
(Routing Protocol Detail ดูต่อใน [Part 73](part-073-large-scale-routing-design.md))

#### 1. Executive Summary

องค์กรมีแผนขยาย Headquarters จาก 1 อาคาร (Building 1 ปัจจุบัน = Lab ที่สร้างตั้งแต่ Part 1-70)
เป็น **Campus 10 อาคาร** รองรับพนักงาน/อุปกรณ์รวม **~35,000-42,000 Endpoint** เอกสารนี้กำหนด
สถาปัตยกรรม Layer 2/3, IP Addressing, และแผนการเชื่อมต่อระหว่างอาคาร โดยยึดหลัก **No Single
Point of Failure** ทุก Tier (ต่อจาก [Part 33](part-033-advanced-stp-campus-design.md)) และ
**Failure Domain Isolation** ทุกขอบเขต (Step 719)

#### 2. Requirement Summary

| ข้อกำหนด | รายละเอียด |
|---|---|
| จำนวนอาคาร | 10 (Building 1 = อาคารปัจจุบัน, Building 2-10 = อาคารใหม่) |
| Access Switch ต่ออาคาร | 50 |
| Endpoint รวม | ~35,000-42,000 |
| เป้าหมาย Availability | No Single Point of Failure ทุก Tier |
| ข้อจำกัด Legacy | Building 1 มี Hardware Access Layer เดิม (Catalyst 9200/2960, L2-only) ที่ยัง Deploy อยู่จริง |

#### 3. Physical/Logical Topology Overview

```
                              CAMPUS BACKBONE (Ring + Chord, Step 716)
                     ┌──────────────────────────────────────────────┐
                     │  B1══B2══B3══B4══B5══B6══B7══B8══B9══B10══B1  │ (Ring)
                     │        B1════════════════B6  (Chord เสริม)     │
                     └──────────────────────────────────────────────┘
        ┌────────────────────┬──────────────────┬─────────────────────┐
   Building 1-9 (Office, 3-Tier)                              Building 10 (Data Center, Spine-Leaf)
   CORE→DIST×4→ACCESS×50                                       SPINE×2→LEAF×8
```

#### 4. IP Addressing Scheme — ภาพรวมทั้ง Campus

> **หลักการ**: Building 1 **คงค่า Legacy ทั้งหมดตาม [`00-ip-address-plan.md`](00-ip-address-plan.md)
> ไว้ไม่เปลี่ยนแปลง** (10.10.0.0/16, Loopback 1.1.1.x) เพราะเป็น Production จริงที่ Deploy ไปแล้ว
> ส่วน Building 2-10 (Greenfield) ใช้ Scheme ใหม่ที่ Summarize ง่ายกว่า

| อาคาร | VLAN/Subnet Block | Loopback0 Block | หมายเหตุ |
|---|---|---|---|
| Building 1 (Legacy) | **10.10.0.0/16** | 1.1.1.x (ตาม `00-ip-address-plan.md`) | คงเดิมทั้งหมด — เอกสารนี้ไม่แก้ไข |
| Building 2 | 10.20.0.0/16 | 1.2.0.x (Core), 1.2.1.x (Dist) | Greenfield, ใช้ Scheme ใหม่ |
| Building 3 | 10.30.0.0/16 | 1.3.0.x, 1.3.1.x | Greenfield |
| Building 4 | 10.40.0.0/16 | 1.4.0.x, 1.4.1.x | Greenfield |
| Building 5 | 10.50.0.0/16 | 1.5.0.x, 1.5.1.x | Greenfield |
| Building 6 | 10.60.0.0/16 | 1.6.0.x, 1.6.1.x | Greenfield |
| Building 7 | 10.70.0.0/16 | 1.7.0.x, 1.7.1.x | Greenfield |
| Building 8 | 10.80.0.0/16 | 1.8.0.x, 1.8.1.x | Greenfield |
| Building 9 | 10.90.0.0/16 | 1.9.0.x, 1.9.1.x | Greenfield |
| Building 10 (Data Center) | 10.100.0.0/16 | 1.10.0.x (Spine), 1.10.1.x (Leaf) | Greenfield, Spine-Leaf (Step 717) |
| Campus Backbone (Inter-Building Link) | 10.253.0.0/16 (`10.253.<เลขอาคาร>.0/30` ×2 ต่ออาคาร) | — | ดู Step 716 |

**Block ต่ออาคาร (10.X0.0.0/16) แต่ละบล็อกแบ่งย่อยตามหลักการ Step 714** — ตัวอย่าง Building 2:

| VLAN ID | ชื่อ | Subnet ตัวอย่าง (ต่อชั้น/ต่อโซน) |
|---|---|---|
| 10 (DATA) | พนักงาน Wired | 10.20.10.0/24, 10.20.11.0/24, ... (1 ต่อชั้น) |
| 20 (VOICE) | VoIP Phone | 10.20.20.0/24, 10.20.21.0/24, ... |
| 30 (SERVERS) | Server ประจำอาคาร | 10.20.30.0/24 |
| 40 (WIFI) | Wireless Client | 10.20.40.0/24 – 10.20.45.0/24 (6 Subnet ตาม Step 714) |
| 99 (MGMT) | Out-of-Band Management | 10.20.99.0/24 |
| 180 (IOT) | IoT/Building System | 10.20.180.0/24 |

VLAN ID ชุดเดียวกัน (10/20/30/40/99/180) **ใช้ซ้ำได้ทุกอาคาร ทุกชั้น ทุกตู้ Rack** ตามหลักการ
Routed Access (Step 713) — สิ่งที่ Unique คือ 2 Octet แรกของ Subnet (10.20.x.x = Building 2
เท่านั้น) เท่านั้น

#### 5. VLAN Allocation Strategy (สรุปจาก Step 714)

- ระดับ Granularity: **1 VLAN ต่อกลุ่ม Access Switch ในโซนเดียวกัน (ไม่เกิน ~10-15 Access
  Switch ต่อ VLAN, ไม่เกิน 500 Host ตามกฎ Step 714)**
- VLAN ไม่มีวัน Trunk ข้าม Access Switch ที่คนละ Uplink Routed Port (บังคับโดย Design ของ
  Step 713 อยู่แล้ว)
- Building 1 (Legacy) ยังคง VLAN Trunk ระหว่าง ACCESS-SW คู่กัน (dual-homed) ตามที่ Part 5/33
  ออกแบบไว้ เพราะ Hardware ไม่รองรับ Routed Access — **ถือเป็น Documented Exception** ไม่ใช่
  ความผิดพลาดที่มองข้าม

#### 6. Layer 2/3 Boundary Decision: เลือก Routed Access สำหรับ Building 2-10 — พร้อมเหตุผล

| เหตุผล | อ้างอิง Step |
|---|---|
| STP Domain Blast Radius ที่สเกล 50 Access Switch/อาคารเสี่ยงเกินไป | Step 712 |
| ต้องการ Failure Domain แคบที่สุดเท่าที่เป็นไปได้ (ต่อ Access Switch เดียว) | Step 713, 719 |
| Broadcast Domain ต้องเล็กกว่า 500 Host ตามกฎ Sizing | Step 714 |
| รองรับการขยายในอนาคตสู่ SD-Access Fabric ได้ทันที (Underlay พร้อมอยู่แล้วตาม [Part 38 Step 377](part-038-sd-access-fundamentals.md)) | Step 713 |

**ข้อยกเว้นที่ต้องระบุชัดเจน**: **Building 1 ยังคง Traditional Design** (ACCESS-SW1-4 เป็น
Catalyst 9200/2960, L2-only) เพราะเป็น Hardware ที่ Deploy ไปแล้วจริงตาม
[`00-ip-address-plan.md`](00-ip-address-plan.md) — แผน Migration ระยะยาว (นอกขอบเขตเอกสารนี้)
คือทดแทน Hardware Access Layer ของ Building 1 เป็น Catalyst 9300 เมื่อครบวงจร Refresh ปกติ
เพื่อให้ตรงตาม Standard เดียวกับ Building 2-10

#### 7. Inter-Building Connectivity Design (สรุปจาก Step 716)

| เส้นทาง | เทคโนโลยี | Redundancy |
|---|---|---|
| Ring หลัก (10 เส้น รอบ Campus) | Dark Fiber (องค์กรเป็นเจ้าของ Conduit) | แต่ละอาคารมี 2 เส้นทางไปยังอาคารข้างเคียงคนละฝั่งเสมอ, Conduit แยกกันจริง (Step 716) |
| Chord Link (Building1↔Building6) | Dark Fiber, ระยะทางไกลกว่า Ring ปกติ | เสริม Bandwidth และลด Hop Count สำหรับ Traffic ข้าม Campus |
| Backup Path ฉุกเฉิน | Point-to-Point Microwave (2-3 คู่ตามความเสี่ยงพื้นที่) | ใช้เมื่อ Dark Fiber ทั้ง Ring ขาดพร้อมกันในบางจุด (Rare Case) |

แต่ละอาคารเชื่อมกับ Campus Backbone ด้วย **Dual-Homed Routed Uplink + OSPF ECMP** (หลักการ
เดียวกับ [Part 33](part-033-advanced-stp-campus-design.md) แต่ยกระดับจาก Dist↔Core ภายในอาคาร
ไปเป็น Building-Core↔Campus-Backbone)

#### 8. Campus Core Architecture Decision

| อาคาร | Topology | เหตุผล (อ้าง Step) |
|---|---|---|
| Building 1-9 | 3-Tier (Core/Dist/Access) + Routed Access (2-9) | Traffic North-South เป็นหลัก (สำนักงานทั่วไป) — Step 715, 717 |
| Building 10 | Spine-Leaf | Data Center Building, Traffic East-West เป็นหลัก — Step 715, 717 |

#### 9. Wireless Design Summary (สรุปจาก Step 718)

- **3 กลุ่ม WLC** (Distributed): กลุ่ม A (Building 1-3), กลุ่ม B (Building 4-7), กลุ่ม C
  (Building 8-10) — แต่ละกลุ่มมี WLC Pair (Active/Standby SSO) วางคนละอาคารในกลุ่มเดียวกัน
- AP รวมทั้ง Campus ประมาณ 1,000 ตัว (~100 ต่ออาคาร Office, มากกว่านั้นสำหรับ Building 10 ถ้ามี
  พื้นที่ Warehouse/Floor สูง)

#### 10. Failure Domain & Resiliency Checklist (สรุปจาก Step 719)

| ควบคุม | Apply ที่ | สถานะ |
|---|---|---|
| Routed Access (L3 Boundary ทุก Access Switch) | Building 2-10 | ✅ Standard |
| Storm-Control ทุกพอร์ต User-facing | ทุกอาคาร (รวม Building 1) | ✅ Standard |
| Private VLAN สำหรับ IoT/Building System | ทุกอาคาร | ✅ Standard |
| DHCP Snooping + IP Source Guard | ทุกอาคาร | ✅ Standard |
| Dual-Homed Routed Uplink ทุก Tier | ทุกอาคาร + Campus Backbone | ✅ Standard |
| Conduit Diversity ระหว่างอาคาร | Campus Backbone Ring | ✅ Standard |

#### 11. Routing Protocol Note (ส่งต่อไป Part 73)

เอกสารนี้จงใจ **ไม่ลงรายละเอียด Routing Protocol** (จำนวน OSPF Area, การ Summarize, หรือการ
พิจารณา BGP/IS-IS ที่สเกลนี้) เพราะเป็นขอบเขตของ [Part 73 — Large-Scale Routing
Design](part-073-large-scale-routing-design.md) โดยตรง — สิ่งที่กำหนดไว้แน่นอนแล้วในเอกสารนี้
คือ **ขอบเขตของ Address Block ต่ออาคาร (ตาม Section 4)** ซึ่งถูกออกแบบให้ **Summarize เป็น 1
Route ต่ออาคารได้พอดี** (แต่ละอาคารคือ 1 บล็อก `/16`) — เป็นการวาง "วัตถุดิบ" ที่ Part 73 จะนำไป
ออกแบบ Area Boundary/Route Summarization ต่อ

#### 12. Migration Path จาก Lab ปัจจุบัน (Building 1) สู่ Design นี้

1. **Building 1 คงเดิมทั้งหมด** — ไม่มีการแก้ Config ใดๆ บน CORE-SW1/2, DIST-SW1-4,
   ACCESS-SW1-4 ที่มีอยู่ (ตรงตามข้อกำหนดของ Part นี้ที่ไม่แก้ไข Lab เดิม)
2. เพิ่ม **Campus Backbone Tier ใหม่** (Section 3/7) เชื่อม CORE-SW1/2 ของ Building 1 เข้ากับ
   Backbone ผ่าน Dual-Homed Uplink ใหม่
3. Build Building 2-10 ตาม Standard ใหม่ (Routed Access ตั้งแต่ Day 1, ไม่ต้องผ่านขั้น
   Traditional Design ก่อนแบบ Building 1)
4. วางแผน Hardware Refresh ของ ACCESS-SW1-4 (Building 1) เป็น Catalyst 9300 ในรอบ Budget ถัดไป
   เพื่อให้ Building 1 ตรงตาม Standard เดียวกับอาคารอื่นในระยะยาว (ไม่บังคับทันที)

#### 13. Open Items / Risks

- Chord Link Building1↔Building6 ต้องสำรวจ Conduit จริงว่ามีเส้นทางที่ Diverse จาก Ring
  หลักหรือไม่ (ยังไม่ยืนยัน ณ เวลาที่เขียนเอกสาร)
- Building 10 (Data Center) ต้องมี Power/Cooling Study แยกต่างหาก (นอกขอบเขต Network Design)
- Routing Protocol Design (Part 73) อาจส่งผลย้อนกลับต่อ Address Block ถ้าพบว่าต้อง Subnet
  เพิ่มสำหรับ Route Summarization ที่ Area Boundary — ต้อง Review Section 4 อีกครั้งหลัง Part 73

---

## แบบฝึกหัดทวนความเข้าใจ Part 72

1. เพราะเหตุใด "Diameter" ของ STP (จำนวน Hop จาก Root) จึงไม่ใช่ตัวแปรที่แย่ลงเมื่อเพิ่ม Access
   Switch คู่ขนานกันในแนวขวาง แต่ "Blast Radius" ของ TCN กลับแย่ลง?
2. ระหว่าง Routed Access แบบ "ล้วนๆ" กับ Routed Access + SD-Access Overlay (LISP/VXLAN) แบบไหน
   แก้ปัญหา VM Live Migration และ Wireless Roaming ได้ และเพราะอะไร?
3. กฎ Broadcast Domain Sizing "ไม่เกิน ~500 Host" มีเหตุผลเชิงเทคนิคอะไรรองรับ (ตอบอย่างน้อย
   2 เหตุผล)?
4. เพราะเหตุใด Building ที่เป็น Data Center/Server Farm ในหลักสูตรนี้จึงแนะนำให้ใช้ Spine-Leaf
   แทน 3-Tier ทั้งที่ Building อื่นในแคมปัสเดียวกันยังใช้ 3-Tier?
5. ในการเชื่อมต่อระหว่างอาคารด้วย Dark Fiber 2 เส้นเพื่อ Redundancy เหตุใด "Conduit Diversity"
   จึงสำคัญกว่าจำนวนเส้น Fiber ที่มี?

### เฉลย

1. Diameter วัดจากจำนวน Hop ในแนวตั้ง (Root→ปลายทาง) ซึ่งไม่เปลี่ยนเมื่อเพิ่ม Switch คู่ขนานที่
   Hop เดียวกัน — แต่ Blast Radius วัดจาก "จำนวน Switch ทั้งหมดใน STP Instance เดียวกัน" ซึ่ง
   TCN ถูก Flood ไปถึงทุกตัวเสมอไม่ว่าจะอยู่ Hop ไหน ดังนั้นยิ่งมี Switch ใน Domain มาก (แนวขวาง)
   Blast Radius ก็ใหญ่ขึ้นตามจำนวนนั้น แม้ Diameter จะเท่าเดิม (ดู Step 712)
2. Routed Access + SD-Access Overlay แก้ได้ เพราะ LISP แยก EID (Identity) จาก RLOC (Location)
   ทำให้ Endpoint คง IP เดิมได้แม้ตำแหน่งทางกายภาพ (RLOC) เปลี่ยน — Routed Access ล้วนไม่มี
   Overlay ทำไม่ได้เพราะ Subnet ผูกติดกับ Access Switch ตัวเดียวตายตัว (Step 713)
3. (ก) ยิ่งวง Broadcast Domain ใหญ่ ยิ่งมี ARP/Neighbor Discovery Broadcast และ DHCP Broadcast
   มากตามสัดส่วนจำนวน Host รบกวน CPU ของทุก Host/Switch ในวงเดียวกัน (ข) Unknown-Unicast
   Flood จาก MAC Table Flush (เช่นจาก TCN) จะกระทบ Host จำนวนมากขึ้นตามขนาดวง (ดู Step 714/712)
4. เพราะ Traffic Pattern ของ Data Center/Server Farm เป็น East-West เป็นหลัก (Server-to-Server,
   Storage) ซึ่ง Spine-Leaf ให้ Hop Count คงที่ (2 Hop ทุกคู่ Leaf) และ ECMP เต็มรูปแบบโดยไม่มี
   Oversubscription สะสมแบบ 3-Tier ในขณะที่ Building สำนักงานทั่วไป Traffic เป็น North-South
   เป็นหลักซึ่ง 3-Tier ออกแบบมาให้พอดีอยู่แล้ว (ดู Step 715, 717)
5. เพราะถ้า Fiber ทั้ง 2 เส้นเดินอยู่ใน Conduit เดียวกัน อุบัติเหตุจุดเดียว (เช่น ขุดถนนตัดท่อ)
   จะทำให้ทั้ง 2 เส้นขาดพร้อมกัน ทำให้ Redundancy ที่ตั้งใจซื้อไม่มีประโยชน์จริง — ต้องแยก
   Physical Path (Conduit) ให้จริงเพื่อให้ Redundancy มีความหมายจริงเมื่อเกิด Single Physical
   Failure (ดู Step 716)

---

## สรุป Part 72

Part นี้ยกระดับหลักสูตรจาก "Config Lab เดี่ยว" สู่ **CCIE-level Design Thinking ที่สเกลระดับ
Campus จริง**: เราแสดงให้เห็นว่า STP Blocking ใช้ไม่ได้ที่สเกล 500 Access Switch (Step 712),
Routed Access กลายเป็น Default ที่จำเป็นไม่ใช่ทางเลือก (Step 713), VLAN ต้องออกแบบตามกฎ
Broadcast Domain Sizing (Step 714), Traffic Pattern สมัยใหม่ (East-West) เปลี่ยนวิธีคิดเรื่อง
Topology (Step 715), การเชื่อมต่อระหว่างอาคารต้องคำนึงถึง Conduit Diversity (Step 716),
Spine-Leaf เป็นทางเลือกที่ถูกต้องสำหรับพื้นที่ East-West Heavy (Step 717), Wireless ต้อง
Distributed ที่สเกลนี้ (Step 718), และ Failure Domain Isolation คือผลลัพธ์ของการเลือก Topology
ที่ถูกต้องร่วมกับ Feature เสริม (Step 719) — ปิดท้ายด้วยเอกสารออกแบบเต็มรูปแบบสำหรับ Campus
HQ 10 อาคาร (Step 720) ที่ต่อยอดจาก Lab เดิมทั้งหมดโดยไม่แก้ไข Config ที่มีอยู่แม้แต่บรรทัดเดียว

✅ **พร้อมสำหรับ Part 73**: เราวาง Address Block ต่ออาคารไว้แล้วให้ Summarize ได้ง่าย
(Section 4/11 ของ Step 720) แต่ยังไม่ได้ตัดสินใจเรื่อง **Routing Protocol Design ที่สเกลนี้**
เลย — Part 73 จะเจาะลึก **Large-Scale Routing Design**: กี่ OSPF Area ถึงจะพอดี, เมื่อไหร่ต้อง
เปลี่ยนไปใช้ IS-IS หรือ BGP ในแกนกลาง Campus, และการ Summarize ที่ Area/Domain Boundary
อย่างถูกต้องสำหรับ Campus ระดับหลายหมื่น Endpoint

**ไปต่อ:** [Part 73 — Large-Scale Routing Design →](part-073-large-scale-routing-design.md)
