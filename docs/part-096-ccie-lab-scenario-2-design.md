# Part 96 — CCIE Full Lab Scenario #2: Design Module
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 951–960 จาก 1000 | Integrated Lab Scenario Block (2/4)**

> ต่อจาก [Part 95 — CCIE Full Lab Scenario #1 (Diagnose Module)](part-095-ccie-lab-scenario-1-diagnose.md)
> ที่ฝึกทักษะ **"เครือข่ายพังแล้ว หา Root Cause ให้เจอ"** แบบเข้มข้นเต็ม 10 Step — Part นี้เปลี่ยน
> โหมดความคิดไปคนละขั้วเลย: จากการ **มองย้อนกลับ** (มีปัญหาอยู่แล้ว → หาสาเหตุ) มาเป็นการ
> **มองไปข้างหน้า** (มี Requirement มา → ต้องสร้าง Solution ที่ยังไม่มีอยู่จริง) นี่คือธรรมชาติของ
> **Design Module** ซึ่งเป็นหนึ่งใน 3 ชั่วโมงแรกของข้อสอบ CCIE EI Lab Exam ตามที่วางกรอบไว้ใน
> [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) — และเป็น Module ที่ Candidate จำนวน
> มาก "ทำได้" ในความหมายที่ว่า Config อะไรไม่ผิด แต่ "ทำไม่ถูก" ในความหมายที่ว่า **ตอบไม่ตรงกับ
> Requirement ที่ให้มาจริงๆ** เพราะเขียน Design ที่ตัวเองคิดว่าดี ไม่ใช่ Design ที่โจทย์ต้องการ
>
> Part นี้จะไม่มี Router/Switch Config แม้แต่บรรทัดเดียว — เหมือน Design Module จริงที่ Candidate
> ตอบผ่านเอกสาร/Diagram/คำอธิบายเท่านั้น สิ่งที่ Part นี้ฝึกคือ **กระบวนการอ่านโจทย์ → จับ Requirement
> ที่ซ่อนอยู่ → เลือกทางที่เหมาะสมที่สุด → เขียนคำอธิบายที่ป้องกันตัวเองได้เมื่อถูกถามซ้ำ** โดยใช้
> เทคนิคหลักที่ Step 953 จะสอน: **Requirements-Traceability Table** — เครื่องมือที่ทำให้ Grader (และ
> ตัวคุณเอง) เห็นชัดว่า Requirement ทุกข้อถูกตอบด้วย Design Element ตัวไหน ไม่มีข้อไหนตกหล่น

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 951 | รูปแบบข้อสอบ Design Module จริง — 3 ประเภทงาน และเกณฑ์การให้คะแนนที่ Grader มองหา |
| 952 | Design Scenario 1 — โจทย์ Business Requirement เต็มรูปแบบ: บริษัทขนาดกลางตั้งใหม่ 3 Site |
| 953 | Design Scenario 1 เฉลย — Model Design พร้อม Requirements-Traceability Table |
| 954 | Design Scenario 2 — โจทย์ Critique: วิเคราะห์ Design ที่มีข้อบกพร่องแอบแฝง |
| 955 | Design Scenario 2 เฉลย — วิเคราะห์ข้อบกพร่องครบทุกจุด พร้อม Design ที่แก้ไขแล้ว |
| 956 | Design Scenario 3 — โจทย์ Prioritization: Campus HA เต็มรูป vs SD-Access เต็มรูป เลือกได้แค่ 1 |
| 957 | Design Scenario 3 เฉลย — Model Justification และการวิเคราะห์ Tradeoff |
| 958 | Design Scenario 4 — โจทย์ Multiple-Choice-with-Justification สไตล์ข้อสอบจริง |
| 959 | Design Scenario 4 เฉลย — วิเคราะห์ทุกตัวเลือกและเหตุผลของคำตอบที่ถูกต้องที่สุด |
| 960 | Full Lab: Design Scenario ผสมข้าม Domain เต็มรูปแบบ + สรุปกลยุทธ์การสอบ Design Module |

---

## Step 951 — รูปแบบข้อสอบ Design Module จริง และเกณฑ์การให้คะแนนที่ Grader มองหา

### 3 ประเภทงานที่ Design Module ใช้จริง

Design Module (~3 ชั่วโมงแรกของ CCIE EI Lab Exam ตาม [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md))
ไม่ได้ถามแบบเดียวซ้ำๆ ตลอด 3 ชั่วโมง — Cisco ผสม **3 รูปแบบงาน** ที่วัดทักษะต่างกันคนละมุม แม้จะใช้
ความรู้ Protocol ฐานเดียวกัน:

| # | รูปแบบงาน | โจทย์ถามแบบไหน | ต้องตอบอะไร | จำลองใน Step |
|---|---|---|---|---|
| 1 | **Design from Requirements** | ให้ Business Requirement Document มาเปล่าๆ ไม่มี Topology/Design ให้ | สร้าง Topology + เลือก Protocol + Diagram + คำอธิบายเหตุผลทั้งหมดเอง | 952-953, 960 |
| 2 | **Critique / Correct a Given Design** | ให้ Design Document ที่มีอยู่แล้วมา (มักมี Diagram+คำอธิบาย) ให้หาจุดที่ไม่ตรง Requirement | ระบุจุดบกพร่องทุกจุด พร้อมอธิบายว่าทำไมผิด และเสนอทางแก้ | 954-955 |
| 3 | **Multiple-Choice with Justification** | ให้ทางเลือก Design 2-4 ทางพร้อม Requirement มา ให้เลือก 1 ทาง | เลือกคำตอบ **พร้อมเหตุผลเชิงเทคนิคที่อ้าง Requirement ทุกข้อ** ไม่ใช่แค่กาช่องคำตอบ | 958-959 |

> **จุดสำคัญของรูปแบบที่ 3**: แม้จะดู "ง่ายกว่า" เพราะไม่ต้องเขียน Design ทั้งหมดเอง แต่ Cisco ให้
> คะแนนจาก **คำอธิบายเหตุผล** เป็นหลัก ไม่ใช่จากตัวเลือกที่กาเพียงอย่างเดียว — กาถูกแต่อธิบายผิด/
> ไม่ครบมักได้คะแนนไม่เต็ม เพราะ Grading Rubric วัด "กระบวนการคิด" ไม่ใช่ "คำตอบปลายทาง" ล้วนๆ

### สิ่งที่ Grader มองหาจริง — ไม่ใช่ "ใช้งานได้ไหม" แต่คือ "ตอบ Requirement ครบไหม"

นี่คือประเด็นที่ Candidate จำนวนมากเข้าใจผิดมากที่สุดเกี่ยวกับ Design Module: **Design ที่ "ใช้งาน
ได้จริงทางเทคนิค" (Technically Correct) ไม่เท่ากับ Design ที่ "ได้คะแนนเต็ม"** เพราะ Design เดียวกัน
อาจ Technically Correct แต่ตอบ Requirement ไม่ครบทุกข้อ ตัวอย่างชัดที่สุด:

```
โจทย์ระบุ 5 Requirement:  R1, R2, R3, R4, R5

Design A (Candidate ตอบ):
  - แก้ R1 ได้สมบูรณ์ แบบ Best Practice ระดับสูง
  - แก้ R2 ได้สมบูรณ์
  - แก้ R3 ได้สมบูรณ์
  - R4  ไม่ได้พูดถึงเลย (ลืม / ไม่เห็นว่าเป็น Requirement)
  - R5  แก้แบบ Technically ใช้ได้ แต่ขัดกับข้อจำกัดที่ระบุใน R2 (Requirement ชนกัน ไม่ได้ Reconcile)

ผลคะแนน: ได้แค่ ~60% แม้ Design จะดูซับซ้อนและ "เก่ง" มาก
เพราะ Grading Rubric ให้คะแนนแยกต่อ Requirement ไม่ใช่ให้คะแนนรวมจาก "ความประทับใจ" ของ Design
```

Grader (มนุษย์ + ระบบ Automated Verification ผสมกัน) ใช้เกณฑ์หลัก 4 ข้อ ไล่ตรวจทีละ Requirement:

| เกณฑ์ | คำถามที่ Grader ถามตัวเอง | ตัวอย่างการเสียคะแนนที่พบบ่อย |
|---|---|---|
| **1. Coverage (ความครบถ้วน)** | Design นี้ตอบ Requirement **ทุกข้อ**ที่ระบุไว้หรือไม่ (รวม Requirement ที่ซ่อนอยู่ใน Constraint) | ตอบ Requirement ที่เห็นชัดครบ แต่ลืม Requirement ที่ซ่อนในประโยคข้อจำกัด (เช่น "งบจำกัด" = ข้อจำกัดที่ต้องสะท้อนใน Design ด้วย) |
| **2. Correctness (ความถูกต้องทางเทคนิค)** | Solution ที่เลือกทำงานได้จริงตามหลัก Protocol/Feature หรือไม่ | เลือก Feature ที่ฟังดูเข้าเรื่องแต่มีข้อจำกัดทางเทคนิคที่ขัดกับ Requirement อื่น (เช่น เลือก Feature ที่ไม่รองรับ Platform ที่โจทย์กำหนด) |
| **3. Fit (ความพอดี ไม่ใช่ความซับซ้อน)** | Solution นี้ "พอดี" กับ Requirement หรือ "เกินความจำเป็น" (Over-engineering) จนขัดกับข้อจำกัดอื่น (งบ/เวลา/ทีมงาน) | เลือก Solution ระดับ Enterprise เต็มรูปแบบ (แพง/ซับซ้อน) ทั้งที่โจทย์ระบุงบจำกัดและทีมงานเล็ก (ตรงกับหลักการ Step 704 ของ Part 71: "เลือก Feature ที่พอดี ไม่ใช่ที่ซับซ้อนที่สุด") |
| **4. Justification (คำอธิบายที่ป้องกันตัวเองได้)** | ถ้าถูกถามซ้ำว่า "ทำไมเลือกทางนี้ ไม่เลือกทางอื่น" ตอบได้ชัดโดยอ้าง Requirement ข้อไหนหรือไม่ | ตอบแค่ "เพราะเป็น Best Practice" โดยไม่ผูกกับ Requirement ข้อใดข้อหนึ่งของโจทย์นี้โดยเฉพาะ |

### ทำไม Requirements-Traceability Table คือเครื่องมือที่ตอบทั้ง 4 เกณฑ์พร้อมกัน

เทคนิคหลักที่ Part นี้จะฝึกซ้ำในทุก Scenario คือการสร้างตาราง 3 คอลัมน์:

```
┌─────────────────────┬──────────────────────────────┬───────────────────────────────┐
│  Requirement (จากโจทย์) │  Design Element (สิ่งที่เลือกใช้) │  เหตุผล (ทำไมสิ่งนี้ตอบ Requirement นี้) │
├─────────────────────┼──────────────────────────────┼───────────────────────────────┤
│  R1: ...             │  ...                          │  ...                            │
│  R2: ...             │  ...                          │  ...                            │
│  R3: ...             │  ...                          │  ...                            │
└─────────────────────┴──────────────────────────────┴───────────────────────────────┘
```

ตารางนี้ทรงพลังเพราะ:

1. **บังคับให้ Coverage ครบ** — เพราะต้อง List Requirement ทุกข้อเป็นแถวก่อนเริ่มออกแบบ ถ้า List
   ไม่ครบ (ลืม Requirement ที่ซ่อนอยู่) จะเห็นช่องว่างทันทีตอนเขียนตาราง
2. **บังคับให้ทุก Design Element มีที่มา** — เพราะแถวจะว่างไม่ได้ ถ้าเขียน Design Element ที่ไม่มี
   Requirement มารองรับ (เช่น ใส่ Feature ที่ "อยากใส่" แต่โจทย์ไม่ได้ขอ) จะเห็นว่ามันไม่ใช่คำตอบต่อ
   Requirement ข้อไหนเลย → สัญญาณของ Over-engineering (ผิดเกณฑ์ที่ 3)
3. **เป็นคำอธิบาย (Justification) สำเร็จรูป** — คอลัมน์ที่ 3 คือคำตอบพร้อมใช้เวลาโจทย์ถามซ้ำว่า "ทำไม
   เลือกทางนี้" เพราะเขียนผูกกับ Requirement เฉพาะเจาะจงไว้แล้วตั้งแต่ตอนออกแบบ
4. **Grader อ่านง่ายและให้คะแนนตรงไปตรงมา** — Grader ไล่ตามแถวได้ทันทีว่า Requirement ไหนถูกตอบ
   ครบหรือไม่ ลด Ambiguity ในการให้คะแนนทั้งสองฝ่าย

> Step 952-953, 954-955, 958-959 และ 960 ของ Part นี้จะใช้เทคนิคเดียวกันนี้ซ้ำทุกครั้ง เพื่อให้กลาย
> เป็นสัญชาตญาณก่อนเข้าห้องสอบจริง — ไม่ใช่แค่รู้ทฤษฎีว่าตารางนี้ดี แต่ต้องเขียนมันได้เร็วภายใน
> เวลาจำกัดจริงด้วย

### ตัวอย่างการถ่วงน้ำหนักคะแนนต่อ Requirement (จำลองจากรูปแบบ Rubric จริง)

Design Task แต่ละข้อในห้องสอบมักมีคะแนนรวมกำหนดไว้ (เช่น 20 คะแนนต่อ Task) และ **แบ่งคะแนนตาม
Requirement ไม่เท่ากันเสมอ** — Requirement ที่เป็น Hard Constraint (คำว่า "ต้อง") มักมีน้ำหนักสูงกว่า
Requirement ที่เป็น Preference ("ควร") ตัวอย่างจำลอง:

| Requirement | น้ำหนักคะแนนสมมติ (จาก 20 คะแนน) | เหตุผลที่น้ำหนักต่างกัน |
|---|---|---|
| Redundancy (Hard Requirement, คำว่า "ต้อง") | 6 คะแนน | เป็น Mandatory ที่ระบุชัดเจนที่สุด — Design ที่ไม่ผ่านข้อนี้มักเสียคะแนนมากที่สุดจากข้อเดียว |
| Legacy Application Compatibility | 5 คะแนน | ผูกกับ Business-critical Application ที่หยุดทำงานไม่ได้ |
| IP Addressing/Scalability | 4 คะแนน | สำคัญแต่ Impact ระยะสั้นน้อยกว่า Redundancy |
| Budget Constraint | 3 คะแนน | มักถูกตรวจแบบ "ผ่าน/ไม่ผ่าน" มากกว่าให้คะแนนไล่ระดับ |
| Justification/เหตุผลที่เขียนอธิบาย | 2 คะแนน | คะแนนเสริมที่ตัดสินเมื่อ Design ทาง Technical เท่ากัน |

> **ข้อคิดเชิงกลยุทธ์**: ถ้าเวลาเหลือน้อยและต้องเลือกว่าจะขัดเกลาส่วนไหนของคำตอบให้สมบูรณ์ที่สุด
> ให้เลือกส่วนที่ตรงกับ Requirement ที่มีน้ำหนักสูงสุดก่อนเสมอ (แม้ Rubric จริงจะไม่เปิดเผยน้ำหนัก
> ตรงๆ แต่คำว่า "ต้อง" ในโจทย์มักเป็นสัญญาณของน้ำหนักสูงเสมอ ตามที่ [Part 71 Step
> 708](part-071-ccie-lab-blueprint-overview.md) อธิบายไว้)

---

## Step 952 — Design Scenario 1: โจทย์ Business Requirement เต็มรูปแบบ

> **หมายเหตุสำคัญ**: บริษัทและ Site ใน Scenario นี้เป็น**กรณีศึกษาสมมติแยกต่างหาก** ไม่ใช่ Lab
> ต่อเนื่องของหลักสูตร (ที่ใช้ Hostname/IP Plan ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md))
> — เพราะ Design Module ของข้อสอบจริงมักให้โจทย์ "บริษัทใหม่" ที่ไม่มี Topology เดิมมาก่อนเสมอ
> ให้อ่านโจทย์ด้านล่างเหมือนเป็นเอกสารที่คุณได้รับในห้องสอบจริง แล้ว**ลงมือออกแบบเองก่อนเปิด Step
> 953** — นี่คือวิธีฝึกที่ได้ผลที่สุด

### เอกสาร Business Requirement — บริษัท สยาม ฟู้ด โปรเซสซิ่ง จำกัด (Siam Food Processing)

**ภูมิหลังธุรกิจ**: Siam Food Processing เป็นผู้ผลิตและจัดจำหน่ายอาหารแปรรูปขนาดกลาง มีพนักงานรวม
~650 คน กำลังลงทุนสร้างระบบ IT Infrastructure ใหม่ทั้งหมด (Greenfield) สำหรับ 3 Site หลัก:

| Site | บทบาท | จำนวนพนักงาน | หมายเหตุ |
|---|---|---|---|
| **HQ (กรุงเทพฯ)** | สำนักงานใหญ่ — Finance, HR, Sales, IT, Call Center | ~250 คน | มี Data Center ขนาดเล็กในอาคารเดียวกัน (Server ~40 ตัว) |
| **Factory (อยุธยา)** | โรงงานผลิต — Production Line, QA, Warehouse | ~300 คน | มีระบบ **MES (Manufacturing Execution System)** ควบคุมสายการผลิตที่ใช้งานมา 5 ปีแล้ว |
| **DC-Hub (ชลบุรี)** | ศูนย์กระจายสินค้า (Distribution Center) | ~100 คน | เชื่อมกับระบบ WMS (Warehouse Management) ที่ HQ |

**Requirement ที่ระบุไว้อย่างชัดเจน (Explicit Requirements)**

1. **R1 — Redundancy**: "ห้ามมี Single Point of Failure ที่ทำให้ Site ใดหลุดจากการเชื่อมต่อกับ Site
   อื่นทั้งหมด (Site Isolation) ไม่ว่าจะเป็น Link เดียวหรืออุปกรณ์เดียวก็ตาม"
2. **R2 — Budget Constraint**: "งบลงทุนโครงสร้างพื้นฐาน Network ทั้งโครงการ (WAN + LAN Core/Dist
   ทุก Site รวมกัน) ต้องไม่เกิน **8 ล้านบาท** ไม่รวมค่า Endpoint/PC/Server"
3. **R3 — Legacy MES Application**: "ระบบ MES ที่ Factory ใช้ License Server ที่ทำงานแบบ **Layer 2
   Adjacency-dependent** (ใช้ Protocol เดิมที่อ้างอิง ARP/Broadcast โดยตรง ไม่รองรับการ Route ผ่าน
   Layer 3) วางแผนจะ Upgrade เป็นเวอร์ชันที่รองรับ Layer 3 ภายใน 2-3 ปีข้างหน้า แต่ **ตอนนี้ยังใช้
   งานแบบเดิมอยู่** และ License Server ตัวหนึ่งอาจต้องอยู่ที่ HQ Data Center ด้วยเหตุผลด้าน Backup"
4. **R4 — แผนควบรวมกิจการ (Merger) ภายใน 18 เดือน**: "ฝ่ายบริหารกำลังเจรจาควบรวมกิจการกับบริษัทคู่แข่ง
   ขนาดใกล้เคียงกัน ('บริษัท B') ซึ่งมี Site อีก 2 แห่ง คาดว่าจะปิดดีลภายใน 18 เดือน — Network Design
   ต้อง**ไม่ปิดกั้นความเป็นไปได้**ที่จะเชื่อมต่อ Site ของบริษัท B เข้ามาในอนาคต โดยไม่ต้อง Renumber
   IP Address ทั้งองค์กรใหม่ทั้งหมด"
5. **R5 — Growth**: "จำนวนพนักงานคาดว่าจะเพิ่มขึ้น ~30% ภายใน 3 ปี (ไม่รวมผลจาก Merger ใน R4)"
6. **R6 — Application ที่ Sensitive ด้านความปลอดภัย**: "ระบบ Call Center ที่ HQ เชื่อมกับระบบ
   Payment Gateway ของธนาคารสำหรับรับชำระเงินลูกค้าองค์กร — ต้องแยก Network Zone จากระบบอื่นตาม
   หลัก PCI-DSS-like Compliance (แม้ไม่ได้บังคับ Certify เต็มรูปแบบ แต่ Auditor ภายในต้องตรวจสอบได้)"
7. **R7 — Operation Team**: "ทีม Network/IT ที่จะดูแลระบบมีพนักงานเพียง **4 คน** (ไม่มีแผนเพิ่มคน
   ในปีนี้) ต้องเลือก Design ที่ทีมขนาดนี้ดูแลได้จริงในระยะยาว"

**ข้อจำกัดเชิงเทคนิคที่ระบุไว้ (Technical Constraints)**

| # | ข้อจำกัด |
|---|---|
| T1 | ผู้ให้บริการ WAN ในพื้นที่ Factory (อยุธยา) มีเฉพาะ MPLS L3VPN จากผู้ให้บริการรายเดียว + Internet Broadband สำรอง (ไม่มี MPLS จากผู้ให้บริการรายที่ 2 ในพื้นที่) |
| T2 | HQ และ DC-Hub อยู่ในพื้นที่ที่มีผู้ให้บริการ MPLS L3VPN 2 รายให้เลือก |
| T3 | Data Center ที่ HQ มีพื้นที่ Rack จำกัด (รองรับเพิ่มอุปกรณ์ได้ไม่เกิน 4U ต่อ Site สำหรับ Network Gear ใหม่) |
| T4 | องค์กรยังไม่มีทีม Wireless/Security เฉพาะทาง — ต้องใช้ Feature ที่ Built-in มาให้มากที่สุด ลด Overhead การจัดการ Appliance แยก |

### สิ่งที่ต้องส่งมอบ (Deliverable)

โจทย์กำหนดให้ตอบครบ 4 อย่าง (เหมือนข้อสอบจริง):

1. WAN Topology Design (เชื่อม 3 Site) พร้อม Routing Protocol ที่เลือกและเหตุผล
2. แนวทางแก้ปัญหา MES Legacy Application (R3) โดยไม่ทำลาย Design โดยรวม
3. IP Addressing Scheme ระดับ Summary (ไม่ต้องลง VLSM ละเอียดทุก Subnet) ที่รองรับ R4 และ R5
4. ตาราง Requirements-Traceability แสดงว่า Requirement ทุกข้อ (R1-R7, T1-T4) ถูกตอบด้วย Design
   Element ใด

> **ลงมือออกแบบเองก่อนอ่าน Step 953** — ลองเขียนคำตอบตัวเองลงกระดาษหรือ Text Editor ก่อน จับเวลา
> ไม่เกิน **45 นาที** (เทียบสัดส่วนเวลาจริงของ Design Module ที่มีหลาย Task ใน 3 ชั่วโมง)

---

## Step 953 — Design Scenario 1 เฉลย: Model Design พร้อม Requirements-Traceability Table

### ขั้นที่ 1 — จับ Requirement ที่ซ่อนอยู่ก่อนออกแบบ

ก่อนเขียน Design ใดๆ ให้ไล่จับ Requirement ที่ไม่ได้เขียนตรงๆ แต่แฝงอยู่ในประโยค:

| Requirement ที่ระบุตรง | Requirement ที่ซ่อนอยู่ (ต้องอนุมานเอง) |
|---|---|
| R1 (Redundancy) | ต้องมี **Dual WAN Router + Dual Carrier ต่อ Site ที่เป็นไปได้** — แต่ T1 บอกว่า Factory มี MPLS แค่รายเดียว → Redundancy ของ Factory ต้องมาจาก **MPLS + Internet Broadband สำรอง (Dual-transport ไม่ใช่ Dual-carrier MPLS)** ไม่ใช่ Design เดียวกันทุก Site |
| R2 (Budget 8 ล้าน) | ห้ามเลือก Solution ที่แพงเกินจำเป็น เช่น SD-WAN Controller-based เต็มรูปแบบสำหรับแค่ 3 Site (Overkill ต่อ R7 ด้วย — ทีมเล็กและงบจำกัด) |
| R3 (MES Legacy L2) | ต้องมี Mechanism **ยืด Layer 2 ระหว่าง HQ ↔ Factory เท่านั้น** (ไม่ใช่ทุก Site) และต้องเป็น Mechanism ที่ **ไม่ผูกกับ WAN Router หลักจนกระทบ Redundancy ของ Traffic อื่น** (แยก Failure Domain) |
| R4 (Merger 18 เดือน) | IP Addressing ต้องเผื่อ **Supernet ที่ยังไม่ได้ใช้** สำหรับ Site บริษัท B ในอนาคต และเลือก Routing Protocol ที่รองรับการเพิ่ม Site ใหม่โดยไม่ต้อง Redesign (เช่น BGP ที่ Summarize ง่าย) |
| R6 (PCI-like) | ต้องมี **VRF-Lite หรือ Zone แยก** สำหรับ Call Center/Payment Segment ไม่ปนกับ LAN ทั่วไป |
| R7 (ทีม 4 คน) | Design ต้อง**ไม่ซับซ้อนเกินความจำเป็น** — เลือก Protocol เดียวที่ทีมเล็กดูแลได้ ไม่ผสมหลาย Routing Protocol โดยไม่จำเป็น |

### ขั้นที่ 2 — Model Design: WAN Topology

```
                          ┌─────────────────────────┐
                          │   MPLS L3VPN Provider A   │ (HQ + DC-Hub + Factory)
                          └──────────┬──────────────┘
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
      ┌───────┴────────┐    ┌────────┴───────┐     ┌────────┴───────┐
      │   HQ-WAN-RTR1/2  │    │ DC-WAN-RTR1/2   │     │ FACT-WAN-RTR1/2 │
      │  (Dual Router,   │    │ (Dual Router,   │     │ (Dual Router,   │
      │   Dual-homed to  │    │  Dual-homed to  │     │  MPLS single-   │
      │   Provider A & B)│    │  Provider A & B)│     │  carrier + IPsec│
      │                  │    │                  │     │  over Internet  │
      │                  │    │                  │     │  backup)        │
      └───────┬──────────┘    └────────┬─────────┘     └────────┬────────┘
              │                        │                         │
      ┌───────┴──────────┐    ┌────────┴─────────┐     ┌─────────┴────────┐
      │ HQ Core/Dist/Access│  │ DC-Hub Core/Access│    │ Factory Core/Access│
      │ + Data Center Zone │  │                    │    │ + MES L2 Extension │
      └────────────────────┘  └────────────────────┘    └────────────────────┘
                    │                                             │
                    └──────────── Dedicated L2 Circuit ───────────┘
                       (Metro-E Point-to-Point หรือ VPLS จาก Provider A
                        เฉพาะ VLAN ของ MES เท่านั้น — แยกจาก MPLS L3VPN หลัก)
```

**เหตุผลของโครงสร้างนี้**: ใช้ **MPLS L3VPN เป็น WAN Backbone หลัก** เชื่อม 3 Site (Provider A ราย
เดียวพอสำหรับ Site ทั้งหมด เพราะ T1 อนุญาต และลดความซับซ้อนตาม R7) แต่ **HQ และ DC-Hub เพิ่ม
Provider B เป็น Dual-carrier** (T2 อนุญาต และ R1 ต้องการ Redundancy สูงสุดที่ Site สำคัญ) ส่วน
**Factory ใช้ Internet Broadband + IPsec เป็น Backup Path** (เพราะ T1 ไม่มี Provider ที่ 2 ในพื้นที่
— นี่คือการยอมรับ Tradeoff ที่สมเหตุสมผลและต้องระบุในคำตอบตรงๆว่าทำไมทำแบบนี้แทน MPLS คู่)

Routing Protocol เลือก **BGP (eBGP ระหว่าง Site ↔ Provider, iBGP หรือ Static+Redistribution ภายใน
แต่ละ Site)** เพราะ: (1) MPLS L3VPN Provider ทุกรายใช้ BGP เป็น Standard อยู่แล้ว (2) BGP Summarize/
Filter ง่ายเมื่อต้องเพิ่ม Site บริษัท B ใน R4 โดยไม่กระทบ Site เดิม (3) เป็น Protocol เดียวที่ทีม 4
คนต้องเรียนรู้เพิ่มจาก OSPF ที่ใช้ภายใน LAN อยู่แล้ว (ตรง R7)

### ขั้นที่ 3 — แก้ปัญหา MES Legacy Application (R3)

**ไม่ยืด Layer 2 ผ่าน MPLS L3VPN หลัก** (จะทำลาย Design L3 ทั้งระบบและเพิ่ม Fault Domain ข้าม Site
โดยไม่จำเป็น ตรงกับหลักการ Failure Domain Isolation) แต่ใช้ **Metro Ethernet Point-to-Point (หรือ
VPLS ถ้า Provider A รองรับ) เฉพาะ VLAN ของ MES ระหว่าง HQ ↔ Factory เท่านั้น** เป็น Circuit แยกต่าง
หากจาก MPLS L3VPN Backbone หลัก — วิธีนี้ทำให้:

- MES ยังคง L2 Adjacency ได้ตาม Requirement เดิม โดยไม่ต้องรอ Upgrade Application
- ถ้า Metro-E Circuit นี้ล้ม **ไม่กระทบ MPLS L3VPN หลักที่ Traffic อื่นทั้งหมดใช้งาน** (Fault Domain
  แยกกันชัดเจน ตรง R1)
- เมื่อ MES Upgrade เป็น L3-capable ใน 2-3 ปีข้างหน้า สามารถยกเลิก Metro-E Circuit นี้ได้โดยไม่กระทบ
  ส่วนอื่นของ Design เลย (Design นี้จึง "Future-proof" ต่อการเปลี่ยนแปลงที่ระบุไว้ในโจทย์)

### ขั้นที่ 4 — IP Addressing Scheme ระดับ Summary (รองรับ R4 + R5)

| Site/Zone | Supernet ที่จัดสรร | เหตุผล |
|---|---|---|
| HQ | 10.10.0.0/16 | เผื่อ Growth 30% (R5) และแยก Zone ย่อยได้เพียงพอ (Data Center, Call Center/Payment แยก /20 ย่อย) |
| Factory | 10.20.0.0/16 | เผื่อขยาย Production Line เพิ่มในอนาคต |
| DC-Hub | 10.30.0.0/16 | เผื่อขยาย Warehouse Zone เพิ่ม |
| **สำรองสำหรับ Merger (R4)** | **10.100.0.0/14 (10.100.0.0–10.103.255.255)** | จองไว้ล่วงหน้าสำหรับ Site บริษัท B ที่จะควบรวมใน 18 เดือน — ไม่ Assign ให้ Site ปัจจุบันแม้จะดูเหมือน "เปลืองที่" เพื่อไม่ต้อง Renumber ทั้งองค์กรตอน Merger จริง |
| Payment/PCI Zone (R6) | 10.10.240.0/22 (VRF แยกภายใน HQ Supernet) | แยกด้วย VRF-Lite ไม่ปนกับ LAN ทั่วไป แม้อยู่ใน Supernet เดียวกันของ HQ |

> **หมายเหตุ**: การจอง Supernet ล่วงหน้าสำหรับ Merger คือตัวอย่างชัดเจนของการตอบ "Requirement ที่
> ยังไม่เกิดขึ้นจริง" (R4 พูดถึงอนาคต 18 เดือน) — Candidate จำนวนมากมองข้าม Requirement แบบนี้เพราะ
> "ยังไม่เกิดขึ้นตอนนี้" แต่ Grading Rubric ให้คะแนน Design ที่เผื่ออนาคตแบบนี้ไว้เสมอ เพราะ CCIE
> วัดความสามารถ **Design เชิงยุทธศาสตร์ ไม่ใช่แค่แก้ปัญหาเฉพาะหน้า**

### ตาราง Requirements-Traceability เต็มรูปแบบ

| Requirement | Design Element ที่ตอบ | เหตุผล |
|---|---|---|
| R1 — ห้าม Site Isolation | Dual WAN Router ทุก Site + Dual-carrier (HQ/DC-Hub) หรือ Dual-transport MPLS+Internet/IPsec (Factory) | ไม่มี Link/Device เดียวที่ล้มแล้วตัดขาด Site ทั้งหมด — ทุก Site มีเส้นทางสำรองอย่างน้อย 1 เส้นทาง |
| R2 — งบไม่เกิน 8 ล้านบาท | เลือก MPLS L3VPN + BGP (Standard, ไม่ใช่ SD-WAN Controller-based ที่มีค่า License/Controller เพิ่ม) | ลดต้นทุนที่ไม่จำเป็นตาม R7 (ทีมเล็ก) และ Requirement ไม่ได้ขอ App-aware Routing ที่ SD-WAN ให้ |
| R3 — MES ต้องการ L2 Adjacency | Metro Ethernet Point-to-Point แยก Circuit เฉพาะ VLAN MES ระหว่าง HQ↔Factory | รักษา L2 Adjacency โดยไม่กระทบ Fault Domain ของ MPLS L3VPN หลัก (แยก Failure Domain ตาม R1) |
| R4 — เผื่อ Merger 18 เดือน | จอง Supernet 10.100.0.0/14 ล่วงหน้า + เลือก BGP ที่ Summarize/เพิ่ม Site ง่าย | เชื่อมต่อ Site บริษัท B ในอนาคตได้โดยไม่ต้อง Renumber องค์กรทั้งหมด |
| R5 — Growth 30% ใน 3 ปี | จัดสรร Supernet ระดับ /16 ต่อ Site (ใหญ่กว่าที่ใช้จริงปัจจุบันมาก) | มี Address Space เหลือสำหรับ VLAN/Subnet ใหม่โดยไม่ต้อง Redesign Addressing |
| R6 — แยก Zone Payment/PCI-like | VRF-Lite แยก Payment/Call Center Segment ภายใน HQ | Isolate Traffic ด้าน Compliance โดยไม่ต้องซื้อ Firewall Appliance แยกเพิ่ม (ตอบ T4 ด้วย) |
| R7 — ทีม 4 คนดูแลได้จริง | Routing Protocol เดียว (BGP ที่ WAN Edge, OSPF ภายใน LAN) ไม่ผสมหลาย Protocol โดยไม่จำเป็น | ลด Learning Curve และ Operational Complexity ให้เหมาะกับขนาดทีม |
| T1 — Factory มี MPLS รายเดียว | Internet Broadband + IPsec เป็น Backup Path ที่ Factory | ยอมรับ Tradeoff ที่มีอยู่จริงในพื้นที่ แทนการเรียกร้อง Provider ที่ไม่มีอยู่ |
| T2 — HQ/DC-Hub มี 2 Provider | Dual-carrier MPLS ที่ 2 Site นี้เท่านั้น | ใช้ Redundancy ระดับสูงสุดที่พื้นที่รองรับได้จริง ไม่เท่ากันทุก Site เพราะข้อจำกัดพื้นที่ต่างกัน |
| T3 — Rack จำกัด 4U ต่อ Site | เลือก WAN Router แบบ ISR/CSR ขนาดเล็ก ไม่ใช่ Chassis-based ขนาดใหญ่ | เข้ากับพื้นที่ Rack ที่มีจริง |
| T4 — ไม่มีทีม Security เฉพาะทาง | ใช้ VRF-Lite (Built-in บน Router, Part 51) แทน Firewall Appliance แยกสำหรับ PCI Zone | ลด Overhead การจัดการ Appliance เพิ่มเติมตามข้อจำกัดทีมงาน |

> สังเกตว่าตารางนี้มี **11 แถว ตอบ Requirement ครบทั้ง R1-R7 และ T1-T4** — นี่คือมาตรฐานที่ต้องทำให้
> ได้ในห้องสอบจริง: ไม่มีแถวไหนขาด ไม่มี Design Element ไหนที่ไม่มี Requirement รองรับ

### ตรวจสอบ Budget Constraint (R2) ด้วยตารางประมาณการคร่าวๆ

โจทย์กำหนดงบไม่เกิน 8 ล้านบาท (R2) — คำตอบที่ดีควรมีตารางประมาณการคร่าวๆ (Order-of-magnitude) แสดง
ว่า Design ที่เลือกอยู่ในกรอบงบจริง ไม่ใช่แค่กล่าวถึงงบลอยๆโดยไม่มีตัวเลขรองรับ:

| รายการ | ประมาณการ (บาท) | หมายเหตุ |
|---|---|---|
| WAN Router (Dual, HQ+DC-Hub+Factory รวม 6 ตัว) | ~2,400,000 | ISR/CSR ระดับกลาง |
| Core/Distribution Switch (3 Site) | ~2,800,000 | รวม HSRP-capable L3 Switch |
| Metro-E Circuit สำหรับ MES (R3) | ~600,000 (ค่าติดตั้ง+ปีแรก) | Circuit แยกเฉพาะ VLAN MES |
| MPLS L3VPN + Internet Broadband (ปีแรกทุก Site) | ~1,500,000 | รวม Dual-carrier ที่ HQ/DC-Hub |
| VRF-Lite Implementation (Software Feature, ไม่มีค่า Hardware เพิ่ม) | 0 | Built-in บน Router ที่มีอยู่แล้ว (ตอบ T4) |
| Access Switch + Cabling ทุก Site | ~600,000 | ตามจำนวน Endpoint ปัจจุบัน+เผื่อ Growth |
| **รวมประมาณการ** | **~7,900,000** | อยู่ในกรอบ 8 ล้านบาทของ R2 (เผื่อ Contingency ~1%) |

> การใส่ตารางประมาณการแบบนี้ (แม้จะเป็นตัวเลขคร่าวๆไม่ใช่ใบเสนอราคาจริง) คือสัญญาณให้ Grader เห็นว่า
> Candidate **ตรวจสอบ Requirement เชิงตัวเลขจริงๆ ไม่ใช่แค่กล่าวถึงผ่านๆ** — ต่างจากการเขียนแค่ว่า
> "Design นี้ประหยัดงบ" โดยไม่มีอะไรรองรับเป็นรูปธรรม

---

## Step 954 — Design Scenario 2: โจทย์ Critique — วิเคราะห์ Design ที่มีข้อบกพร่องแอบแฝง

### บริบทโจทย์

บริษัท **Rungthip Manufacturing** (มีสำนักงาน 2 แห่ง: HQ และ Plant) ได้จ้างวิศวกรเครือข่ายอีกคนออก
แบบ Network ให้เสร็จแล้ว และผู้บริหารขอให้คุณ (ในฐานะ Senior Network Engineer / Reviewer) **ตรวจสอบ
Design Document ก่อนอนุมัติให้ Implement จริง** — นี่คือรูปแบบงาน "Critique" ที่ข้อสอบ CCIE Design
Module ใช้จริงบ่อยครั้ง โจทย์จะไม่บอกตรงๆว่า "ผิดกี่จุด" — ต้องหาให้ครบเอง

**Requirement เดิมของโปรเจกต์** (ที่วิศวกรคนก่อนได้รับมา):

- R1: รองรับพนักงาน HQ 80 คนตอนเริ่มต้น คาดว่าจะเพิ่มเป็น 200 คนภายใน 2 ปี
- R2: ต้องมี Redundancy ระดับพื้นฐาน — WAN ต้องไม่ล้มจาก Router เดียวพัง
- R3: ต้องมี Password/Access Control ที่ปลอดภัยตาม Standard องค์กร (Password Complexity + Encrypted
  Management)
- R4: Site ทั้งสองต้องสื่อสารกันผ่าน MPLS L3VPN ที่จัดหาแล้ว

### Design Document ที่วิศวกรคนก่อนส่งมอบ (มีข้อบกพร่องซ่อนอยู่ — หาให้เจอ)

**IP Addressing Plan**

```
HQ LAN (VLAN 10 - User):      10.1.1.0/27    (30 usable host)
HQ LAN (VLAN 20 - Server):    10.1.2.0/28    (14 usable host)
Plant LAN (VLAN 10 - User):   10.2.1.0/27    (30 usable host)
Management VLAN (ทั้ง 2 Site): 10.1.99.0/24   (Router/Switch Management IP ทุกตัว)
```

**Routing & Redundancy Design**

```
                     ┌──────────────────┐
                     │   MPLS L3VPN      │
                     └─────────┬─────────┘
                               │
                     ┌─────────┴─────────┐
                     │   HQ-WAN-RTR       │  <-- WAN Router ตัวเดียว
                     │  (ทำหน้าที่ทั้ง WAN  │      ทำหน้าที่ Edge Router +
                     │   Edge และ Default │      Default Gateway ของ LAN ทั้งหมด
                     │   Gateway ของ LAN)  │      (ไม่มี Core/Distribution แยก)
                     └─────────┬─────────┘
                               │
                  ┌────────────┴────────────┐
                  │      Access Switch (x3)  │
                  └──────────────────────────┘

หมายเหตุจาก Design Document เดิม: "ใช้ OSPF Totally Stub Area บน HQ-WAN-RTR เพื่อลดขนาด Routing
Table รับ Default Route จาก MPLS Provider เพียงเส้นทางเดียว (ไม่ Redistribute Static สำรอง เพราะ
Provider บอกว่า MPLS Link มี SLA 99.9% อยู่แล้ว เพียงพอต่อ Requirement R2)"
```

**Security Configuration ที่ระบุใน Design Document**

```
- Privileged Access: ใช้ Local Username/Password เดียวกันทุกอุปกรณ์ (username admin
  privilege 15 secret Rungthip@2024) เพื่อความง่ายในการจดจำของทีม Operation
- SNMP: enable snmp-server community public RO สำหรับ Monitoring Tool อ่านค่าพื้นฐาน
- Management VLAN 10.1.99.0/24 เข้าถึงได้จาก VLAN User (10.1.1.0/27) โดยตรงผ่าน Router เดียวกัน
  ไม่มี ACL กั้น เพราะ "ทีม IT ต้อง SSH เข้าไป Manage อุปกรณ์จาก Laptop ตัวเองที่อยู่ใน VLAN User"
```

### งานที่ต้องทำ

หา **ข้อบกพร่องทั้งหมด** ใน Design Document นี้ที่ทำให้ไม่ตรงกับ R1-R4 หรือเป็นความเสี่ยงที่ไม่
สมเหตุสมผล — ระบุ**อย่างน้อย 4 จุด** จากทั้ง 3 หมวด (Addressing / Routing-Redundancy / Security)
พร้อมอธิบายว่าทำไมแต่ละจุดถึงเป็นปัญหา และมันขัดกับ Requirement ข้อไหน

> ลองวิเคราะห์เองก่อนอ่าน Step 955 — ใช้เวลาไม่เกิน 20 นาที

---

## Step 955 — Design Scenario 2 เฉลย: วิเคราะห์ข้อบกพร่องครบทุกจุด พร้อม Design ที่แก้ไขแล้ว

### ตารางวิเคราะห์ข้อบกพร่องทั้งหมด (Flaw Analysis)

| # | หมวด | ข้อบกพร่องที่พบ | ขัดกับ Requirement ข้อไหน | ทำไมถึงเป็นปัญหา |
|---|---|---|---|---|
| **F1** | IP Addressing | HQ User VLAN ใช้ /27 (30 host) แต่ต้องรองรับ 200 คนใน 2 ปี | **R1** | /27 รองรับได้แค่ 30 Host — เมื่อพนักงานเพิ่มถึง ~35-40 คนก็ล้น Subnet ทันที (ยังไม่ถึงครึ่งทางของเป้าหมาย 200 คน) ต้องมาทำ Re-addressing กลางทางซึ่งกระทบ Production |
| **F2** | Routing/Redundancy | HQ-WAN-RTR เป็น Router ตัวเดียวทำหน้าที่ทั้ง WAN Edge และ Default Gateway ของ LAN ทั้งหมด ไม่มี Core/Distribution แยก และไม่มี Router สำรอง | **R2** | ถ้า HQ-WAN-RTR ตัวนี้พังตัวเดียว (Hardware Fail, Software Crash, หรือแค่ Reboot ตอน Upgrade) **ทั้ง LAN และ WAN หลุดพร้อมกันหมด** — นี่คือ Single Point of Failure ที่ตรงข้ามกับ R2 อย่างชัดเจน ("WAN ต้องไม่ล้มจาก Router เดียวพัง" แต่ Design นี้ทำให้ Router เดียวพังแล้วทุกอย่างพังตาม) |
| **F3** | Routing/Redundancy | ใช้ OSPF Totally Stub Area รับ Default Route จาก MPLS Provider **เพียงเส้นทางเดียว** และไม่ Redistribute Static สำรอง โดยอ้างอิงแค่ SLA 99.9% ของ Provider | **R2** | SLA 99.9% ของ Provider คือ**ค่าเฉลี่ยทางสัญญา ไม่ใช่การันตีว่าจะไม่ล้มเลย** — และยิ่งสำคัญกว่านั้น การไม่มี Backup Path ใดๆเลย (ไม่มี Internet Backup, ไม่มี Static Route สำรอง) หมายความว่า**ถ้า MPLS Link นี้ล้มไม่ว่าเหตุผลอะไร Site ทั้งหมดหลุดจาก WAN ทันที** — Design ตีความ R2 ผิดว่า "SLA สูง = ไม่ต้องมี Redundancy จริง" ซึ่งไม่ใช่สิ่งเดียวกัน |
| **F4** | Security | Local Username/Password เดียวกันทุกอุปกรณ์ ไม่มี AAA/TACACS+ Server, ไม่มี Per-user Accountability | **R3** | ขัดกับ "Access Control ที่ปลอดภัยตาม Standard องค์กร" อย่างชัดเจน — Password เดียวกันทุกอุปกรณ์หมายความว่าถ้า Password รั่วที่จุดใดจุดหนึ่ง (เช่น พนักงานลาออกแล้วรู้ Password) **ทุกอุปกรณ์ในองค์กรถูกเข้าถึงได้ทันที** และไม่มีทางรู้ว่า "ใคร" เป็นคนแก้ Config อะไรเมื่อไหร่ (ไม่มี Accountability ตามหลัก AAA ที่สอนใน Part 22) |
| **F5** | Security | SNMP community string เป็น `public` (ค่า Default ที่ทุกคนรู้) แม้จะเป็น Read-only | **R3** | แม้เป็น RO แต่ Community String `public`/`private` คือค่า Default ที่ Attacker Scan หาได้อัตโนมัติเสมอ — SNMP RO ที่เดาได้ยังเปิดเผยข้อมูล Topology/Config จำนวนมากที่ใช้ในการวางแผนโจมตีขั้นต่อไปได้ (Information Disclosure) ควรใช้ **SNMPv3** ที่มี Authentication+Encryption ตาม Standard ที่สอนใน Part 17 |
| **F6** | Security | Management VLAN (10.1.99.0/24) เข้าถึงได้จาก User VLAN โดยตรงไม่มี ACL กั้น | **R3** | ทำให้ Endpoint ทั่วไปในองค์กร (Laptop พนักงาน ที่อาจติด Malware ได้ง่ายกว่า Server) มีเส้นทางตรงไปถึง Management Interface ของทุกอุปกรณ์เครือข่าย — ควรจำกัดด้วย ACL ให้เฉพาะ IP ของทีม IT เท่านั้นที่เข้าถึง Management VLAN ได้ (Jump Host/Bastion Pattern จะดีกว่านี้อีกขั้น) |

> **ข้อสังเกตสำคัญ**: F2 และ F3 เป็นข้อบกพร่องคนละชนิดแต่**เสริมกันจนกลายเป็นความเสี่ยงซ้อนสองชั้น**
> — F2 คือ Single Point of Failure ด้าน Hardware/Platform (Router ตัวเดียว) ส่วน F3 คือ Single
> Point of Failure ด้าน Path/Circuit (Link เดียว) การ Critique ที่ดีต้องแยกแยะทั้งสองชนิดออกจากกัน
> ให้ชัด เพราะวิธีแก้ต่างกันคนละแบบ (F2 แก้ด้วย Redundant Hardware, F3 แก้ด้วย Redundant Path)

### Design ที่แก้ไขแล้ว (Corrected Design)

**IP Addressing ใหม่**

| Zone | Subnet เดิม (ผิด) | Subnet ที่แก้ไข | เหตุผล |
|---|---|---|---|
| HQ User VLAN | 10.1.1.0/27 (30 host) | **10.1.1.0/23 (510 host)** | รองรับ 200 คนใน 2 ปี พร้อมเผื่อ Growth เพิ่มเติมหลังจากนั้นโดยไม่ต้อง Re-address ซ้ำ |
| HQ Server VLAN | 10.1.2.0/28 (14 host) | **10.1.2.0/25 (126 host)** | เผื่อ Server เพิ่มในอนาคต |
| Plant User VLAN | 10.2.1.0/27 (30 host) | **10.2.1.0/24 (254 host)** | เผื่อ Growth ของ Plant ด้วยเช่นกัน แม้ Requirement ไม่ได้ระบุตัวเลขชัดเหมือน HQ |
| Management VLAN | 10.1.99.0/24 (รวมกันทุก Site) | **แยกต่อ Site: 10.1.99.0/26 (HQ) / 10.2.99.0/26 (Plant)** | ลด Broadcast Domain และแยก Fault Domain ของ Management ระหว่าง Site |

**Routing & Redundancy ใหม่**

```
                     ┌───────────────┬───────────────┐
                     │ MPLS L3VPN     │  Internet      │
                     │ (Primary)      │  Broadband     │
                     │                │  + IPsec       │
                     │                │  (Backup Path) │
                     └───────┬────────┴────────┬───────┘
                             │                  │
                   ┌─────────┴──────────────────┴─────────┐
                   │      HQ-WAN-RTR1  ⟷  HQ-WAN-RTR2       │  <-- Dual Router
                   │   (HSRP/VRRP Active-Standby ต่อ LAN)    │      แยกจาก LAN Core
                   └─────────────────┬───────────────────────┘
                                     │
                        ┌────────────┴────────────┐
                        │  HQ-CORE/DIST-SW1/2       │  <-- เพิ่ม Distribution แยก
                        │  (HSRP VIP เป็น Default    │      จาก WAN Router
                        │   Gateway ของ LAN)         │
                        └────────────┬───────────────┘
                                     │
                        ┌────────────┴────────────┐
                        │      Access Switch (x3+)  │
                        └────────────────────────────┘

Routing: OSPF (ภายใน LAN, ไม่ Stub Area — ต้องเรียนรู้ Path สำรองผ่าน Internet Backup ด้วย) +
Static Route สำรอง (Floating Static ผ่าน IPsec Tunnel, Administrative Distance สูงกว่า OSPF)
```

**Security ที่แก้ไขแล้ว**

| หัวข้อ | แก้ไขเป็น |
|---|---|
| Privileged Access | AAA ผ่าน **TACACS+ Server** (Part 22) — แต่ละคนมี Username ของตัวเอง, Local Fallback ไว้เฉพาะกรณี TACACS+ Server ไม่ตอบสนอง |
| SNMP | เปลี่ยนเป็น **SNMPv3** พร้อม Authentication (SHA) + Privacy (AES) แทน Community String |
| Management VLAN Access | เพิ่ม **ACL บน Router/Switch** อนุญาตเฉพาะ IP Range ของทีม IT เข้าถึง Management VLAN ได้ ปิดเส้นทางตรงจาก User VLAN ทั่วไป |

---

## Step 956 — Design Scenario 3: โจทย์ Prioritization — Campus HA เต็มรูป vs SD-Access เต็มรูป

### บริบทโจทย์

บริษัท **Metro Capital Securities** (บริษัทหลักทรัพย์ขนาดกลาง) มี HQ 1 อาคารในกรุงเทพฯ และกำลังอยู่
ระหว่างการเข้าซื้อกิจการสาขาย่อยอีก **5 แห่ง** (จากดีล M&A ที่กำลังจะปิดภายใน 18 เดือน — คล้าย
บริบทของ Scenario 1 แต่คนละบริษัท) แต่ละสาขาที่จะเข้ามาใหม่มีอุปกรณ์เครือข่ายรุ่นเก่าคนละยี่ห้อ
ไม่สอดคล้องกัน (Inconsistent Legacy Gear)

**ทีมผู้บริหารอนุมัติงบลงทุน Network Infrastructure ก้อนใหญ่ก้อนเดียวสำหรับปีนี้เท่านั้น** —
ฝ่ายเทคนิคเสนอ 2 ทางเลือกที่ **ทำได้แค่ทางเดียว** (งบไม่พอทำทั้งสองทางในปีเดียวกัน):

| ทางเลือก | รายละเอียด | อ้างอิง Part ที่เกี่ยวข้อง |
|---|---|---|
| **Option A: Full Campus HA ที่ HQ** | StackWise Virtual/VSS ที่ Core+Distribution, Dual-homed Uplink ทุกชั้น, NSF/SSO เต็มรูปแบบ, Redundant WAN Edge — ยกระดับความทนทานของ HQ ที่มีอยู่แล้วให้แข็งแกร่งที่สุดในระดับ Enterprise | Part 33-35, 81 |
| **Option B: Full SD-Access Rollout** | Fabric ที่ HQ เป็น Fabric Core/Border ก่อน แล้ววางแผน Extend Fabric Edge ไปยัง Site ที่เข้าซื้อใหม่ทั้ง 5 แห่ง ผ่าน DNA Center Template + Automated Onboarding, Micro-segmentation ด้วย SGT ทั่วทั้ง Fabric | Part 38, 50, 77 |

**ข้อมูลเพิ่มเติมที่ระบุในโจทย์ (Business Priorities — ต้องใช้ประกอบการตัดสินใจ)**

1. **P1**: ทีม Network Engineering มีพนักงานเพียง **3 คน** ไม่มีแผนเพิ่มคนภายในปีนี้
2. **P2**: กฎระเบียบกำกับดูแลธุรกิจหลักทรัพย์ (คล้าย ก.ล.ต.) กำหนดให้ต้องมี **Security Policy ที่
   สอดคล้องกันทุก Site ที่ Audit ตรวจสอบได้ง่าย** ก่อนที่สาขาใหม่ทั้ง 5 แห่งจะเริ่มดำเนินธุรกรรมภาย
   ใต้ใบอนุญาตเดียวกันของบริษัท
3. **P3**: Core Network ปัจจุบันที่ HQ มี **Dual-homed Uplink พื้นฐานและ HSRP อยู่แล้ว** (จาก Part
   18/33 ที่ Implement มาตั้งแต่ก่อสร้าง Lab เดิม) — ยังไม่มี StackWise Virtual/NSF-SSO เต็มรูปแบบ
   แต่ **Downtime ที่วัดได้จริงในรอบ 2 ปีที่ผ่านมาอยู่ที่ <0.1% ของเวลาทำการ** (ไม่มี Incident รุนแรง)
4. **P4**: สาขาใหม่ทั้ง 5 แห่งมีอุปกรณ์ Legacy คนละยี่ห้อ การรวมเข้าเป็นมาตรฐานเดียวด้วยมือ (Manual
   Configuration ทีละสาขา) จะใช้เวลาประมาณ 4-6 สัปดาห์ต่อสาขาโดยทีม 3 คนปัจจุบัน (รวม 20-30 สัปดาห์
   สำหรับ 5 สาขา — เกินกรอบเวลา 18 เดือนของดีลถ้านับรวมงานประจำอื่นด้วย)
5. **P5**: Trading Floor ที่ HQ มี Application ที่ Sensitive ต่อ Latency สูง (Order Matching System)
   แต่ **ปัจจุบันยังไม่เคยมี Incident ที่กระทบ Trading เพราะ Network Downtime**

### งานที่ต้องทำ

เลือก **Option A หรือ Option B** พร้อมเขียนคำอธิบายเหตุผลแบบที่ CCIE Design Module ต้องการ — คือ
ต้องวิเคราะห์ **Tradeoff ที่ยอมรับได้ของทางที่ไม่เลือก** ด้วย ไม่ใช่แค่บอกว่าทางที่เลือกดีอย่างไร
(นี่คือทักษะ Prioritization ที่ต่างจาก Scenario 1-2 ซึ่งเป็น "ออกแบบให้ตอบ Requirement ครบ" — Scenario
นี้คือ "ตัดสินใจว่าจะ**ไม่**ทำอะไร และทำไมการไม่ทำนั้นยอมรับได้")

> ลองวิเคราะห์เองก่อนอ่าน Step 957 — ใช้เวลาไม่เกิน 25 นาที

---

## Step 957 — Design Scenario 3 เฉลย: Model Justification และการวิเคราะห์ Tradeoff

### หลักการตัดสินใจ: มองหา "ปัญหาที่ยังไม่ถูกแก้" ก่อน ไม่ใช่ "เทคโนโลยีที่น่าตื่นเต้นกว่า"

จุดที่ Candidate จำนวนมากพลาดในโจทย์ประเภทนี้คือการเลือกโดยใช้เกณฑ์ **"เทคโนโลยีไหนใหม่กว่า/ยากกว่า/
ดูเป็น CCIE มากกว่า"** (มักเอียงไปทาง SD-Access เพราะ "ดูล้ำกว่า") แทนที่จะใช้เกณฑ์ที่ถูกต้องคือ
**"ปัญหาทางธุรกิจข้อไหนที่ Priority สูงสุดและยังไม่มีทางแก้ ณ ตอนนี้"**

### วิเคราะห์แต่ละ Priority ก่อนตัดสินใจ

| Priority | Option A (Campus HA) แก้ปัญหานี้ได้ไหม | Option B (SD-Access) แก้ปัญหานี้ได้ไหม |
|---|---|---|
| P1 (ทีมเล็ก 3 คน) | **ไม่ช่วย** — Campus HA เพิ่ม Complexity ของ HQ เดียวโดยไม่ลด Workload การจัดการ 5 สาขาใหม่เลย | **ช่วยตรงประเด็น** — DNAC Template + Automated Onboarding ลด Manual Work ต่อสาขาลงอย่างมาก ทีม 3 คนดูแล Fabric ที่ขยายได้จริง |
| P2 (Security Policy สอดคล้องกันทุก Site เพื่อ Audit) | **ไม่ช่วยเลย** — Campus HA ปรับปรุงเฉพาะ HQ ไม่มีผลต่อ Policy ของสาขาใหม่แม้แต่น้อย | **ช่วยตรงประเด็นที่สุด** — SD-Access ใช้ SGT/Micro-segmentation ที่ Consistent ทั่ว Fabric โดยธรรมชาติ ตรงกับ Requirement การ Audit ข้าม Site |
| P3 (HQ Downtime <0.1%, ไม่มี Incident รุนแรง) | Campus HA จะปรับปรุงตัวเลขนี้ให้ดีขึ้นอีก แต่**ฐานปัจจุบันดีอยู่แล้ว** — ROI ของการลงทุนเพิ่มต่ำ (Diminishing Returns) | ไม่ใช่ประเด็นที่ Option B ต้องแก้ (ไม่ใช่จุดที่มีปัญหาอยู่) |
| P4 (Manual Integration 20-30 สัปดาห์ เกินกรอบเวลาดีล) | **ไม่ช่วยแก้ปัญหานี้เลย** — Campus HA ไม่เกี่ยวกับการ Integrate สาขาใหม่ | **แก้ปัญหานี้ตรงจุดที่สุด** — Automated Fabric Onboarding คือคำตอบของปัญหา "เวลาจำกัด + คนน้อย" นี้โดยตรง |
| P5 (Trading Floor Sensitive แต่ยังไม่เคย Incident) | Campus HA ช่วยเสริมความมั่นใจ แต่**ยังไม่มีหลักฐานว่าจำเป็นเร่งด่วน** (ไม่มี Incident History) | ไม่ใช่ประเด็นที่ Option B ต้องแก้ |

### บทสรุปการตัดสินใจ: เลือก **Option B — Full SD-Access Rollout**

**เหตุผลหลัก**: จาก 5 Priority ที่ระบุ มี **P1, P2, P4 (3 จาก 5 ข้อ) ที่เป็นปัญหาจริงและยังไม่มีทาง
แก้** และทั้ง 3 ข้อนี้ถูกแก้โดย Option B โดยตรง ในขณะที่ Option A ไม่ได้แก้ปัญหาใดใน 3 ข้อนี้เลย —
ส่วน P3 และ P5 ที่ Option A จะช่วยได้ เป็นประเด็นที่**ฐานปัจจุบันดีอยู่แล้ว (Downtime <0.1%, ไม่มี
Incident)** การลงทุนเพิ่มใน P3/P5 จึงให้ผลตอบแทนต่อบาทลงทุนต่ำกว่ามาก (Diminishing Returns) เทียบกับ
การลงทุนแก้ P1/P2/P4 ที่เป็นปัญหาเปิดอยู่จริงและมี Deadline ผูกกับดีล M&A

### การวิเคราะห์ Tradeoff ของทางที่ไม่เลือก (ส่วนที่ Candidate มักลืมเขียน)

เขียนส่วนนี้ให้ชัดเสมอ — Grader มองหาสัญญาณว่า Candidate **เข้าใจว่าตัวเองยอมรับความเสี่ยงอะไรบ้าง**
ไม่ใช่แค่เลือกทางที่ดูดีแล้วเดินหน้าไปโดยไม่พูดถึงสิ่งที่เสียไป:

> **ความเสี่ยงที่ยอมรับจากการไม่เลือก Option A**: HQ Core ยังคงเป็น Dual-homed + HSRP แบบเดิม
> (ไม่ใช่ StackWise Virtual/NSF-SSO เต็มรูปแบบ) หมายความว่าในเหตุการณ์ Failover ระหว่าง Core Switch
> คู่ จะยังมี **Convergence Time ระดับ Sub-second ถึงไม่กี่วินาที** (ตาม HSRP ปกติ) แทนที่จะเป็น
> **Sub-50ms** แบบ NSF/SSO เต็มรูปแบบ (Part 81) — สำหรับ Trading Floor ที่ Sensitive ต่อ Latency
> (P5) นี่คือความเสี่ยงที่มีอยู่จริง แต่**ยอมรับได้** เพราะ (1) ไม่มี Incident History ที่พิสูจน์ว่า
> เป็นปัญหาจริงในทางปฏิบัติ (2) เป็นความเสี่ยงที่ **มีอยู่แล้วในปัจจุบัน ไม่ได้เพิ่มขึ้นจากการเลือก
> Option B** (Baseline เดิมไม่เปลี่ยน) และ (3) สามารถ**นำมาพิจารณาลงทุนใหม่ในรอบงบปีถัดไป** เมื่อ
> การควบรวมกิจการเสร็จสมบูรณ์และ Priority P1/P2/P4 ถูกแก้ไปแล้ว

> **หลักการสำคัญที่ต้องจำ**: คำตอบ Prioritization ที่ดีไม่ใช่คำตอบที่บอกว่า "ทางที่เลือกไม่มีข้อเสีย
> เลย" (ซึ่งไม่จริงและ Grader มองออกทันที) แต่คือคำตอบที่ **ยอมรับข้อเสียอย่างตรงไปตรงมา พร้อมอธิบาย
> ว่าทำไมข้อเสียนั้นยอมรับได้ภายใต้ Priority ที่ให้มา และมีแผนจะกลับมาแก้เมื่อไหร่**

---

## Step 958 — Design Scenario 4: โจทย์ Multiple-Choice-with-Justification สไตล์ข้อสอบจริง

### บริบทโจทย์

บริษัท **Krungthep Distribution** มีสาขาย่อย (Branch) ทั้งหมด **12 แห่ง** กระจายอยู่ทั่วประเทศ
ปัจจุบันแต่ละสาขาเชื่อมต่อกับ HQ ผ่าน MPLS L3VPN แต่ผู้บริหารต้องการลดค่าใช้จ่าย WAN โดยเปลี่ยนไปใช้
**Internet/Broadband เป็น Underlay หลัก** และสร้าง Overlay VPN เชื่อมทุกสาขาแทน

**Requirement ที่ระบุไว้ (ต้องทำให้ครบ — Mandatory)**

- **M1**: ต้องเป็น **Full-mesh Dynamic-routed Overlay** — สาขาสามารถส่ง Traffic ตรงถึงกันได้โดยไม่
  ต้องผ่าน HQ เสมอไป (ไม่ใช่ Hub-and-spoke แบบบังคับผ่าน HQ ทุก Traffic)
- **M2**: เมื่อมีสาขาใหม่เพิ่มเข้ามาในอนาคต (มีแผนเพิ่มอีก 3-4 สาขาใน 2 ปีข้างหน้า) **ต้องไม่ต้อง
  แก้ไข Configuration ของสาขาเดิมที่มีอยู่แล้วแม้แต่ตัวเดียว** — เพิ่ม Config ที่สาขาใหม่อย่างเดียวพอ
- **M3**: งบประมาณ WAN ต่อปีต้องลดลงจากเดิมอย่างมีนัยสำคัญ (เป้าหมาย ลดลง >40%)

**Requirement เสริม (Bonus — ไม่บังคับในเฟสนี้)**

- **B1**: Application-aware Path Selection (เช่น ส่ง Voice ผ่านเส้นทางที่ Latency ต่ำสุด, ส่ง Bulk
  Data ผ่านเส้นทางที่ถูกที่สุด) — โจทย์ระบุชัดว่า **"เป็นแผนสำหรับเฟสถัดไป ไม่ใช่ Requirement ของ
  เฟสนี้"**

### ตัวเลือก Design ที่ต้องเลือก 1 ทาง

| ตัวเลือก | รายละเอียด |
|---|---|
| **Option A** | สร้าง **IPsec Point-to-Point Tunnel แบบ Full-mesh ด้วยมือ** ระหว่างทุกสาขา (12 สาขา = 66 Tunnel ในปัจจุบัน) ใช้ Static Route หรือ EIGRP/OSPF ผ่าน Tunnel Interface |
| **Option B** | **DMVPN Phase 3 แบบ Dual-hub** — mGRE Tunnel จากทุกสาขาไปยัง Hub 2 ตัว (Redundant), ใช้ EIGRP หรือ BGP over DMVPN, NHRP ทำให้เกิด Spoke-to-Spoke Dynamic Tunnel โดยไม่ต้อง Config ล่วงหน้า |
| **Option C** | **Cisco SD-WAN (Viptela) Fabric เต็มรูปแบบ** — vEdge/cEdge ที่ทุกสาขา, vSmart Controller เป็นศูนย์กลางแจก Policy, Application-aware Routing แบบ Native |

### งานที่ต้องทำ

เลือก 1 ตัวเลือกพร้อมเขียนคำอธิบายว่า **ทำไมตัวเลือกนี้ตอบ M1-M3 ได้ดีที่สุด และทำไมตัวเลือกอื่นไม่
เหมาะสมกว่า** (ต้องอธิบายทั้งสองด้าน ไม่ใช่แค่พูดถึงตัวเลือกที่เลือก) — สังเกตว่าโจทย์แยก Requirement
เป็น Mandatory (M1-M3) กับ Bonus (B1) อย่างชัดเจน ซึ่งเป็นกับดักสำคัญที่ต้องอ่านให้ทัน

> ลองวิเคราะห์เองก่อนอ่าน Step 959 — ใช้เวลาไม่เกิน 15 นาที (ข้อสอบจริงมักให้เวลาต่อข้อ MCQ-with-
> Justification สั้นกว่า Design-from-scratch เต็มรูปแบบ)

---

## Step 959 — Design Scenario 4 เฉลย: วิเคราะห์ทุกตัวเลือกและเหตุผลของคำตอบที่ถูกต้องที่สุด

### ตารางเปรียบเทียบทั้ง 3 ตัวเลือกต่อ Requirement

| Requirement | Option A (Manual Full-mesh IPsec) | Option B (DMVPN Phase 3 Dual-hub) | Option C (SD-WAN Viptela) |
|---|---|---|---|
| **M1** (Full-mesh Dynamic) | ผ่าน — เป็น Full-mesh จริง แต่ "Static" ไม่ใช่ "Dynamic" (Tunnel ถูกสร้างล่วงหน้าตายตัวทั้งหมด ไม่ใช่ตาม Demand) | **ผ่านสมบูรณ์** — Spoke-to-Spoke Tunnel เกิดขึ้น Dynamic ผ่าน NHRP ตาม Demand จริง | ผ่านสมบูรณ์ — SD-WAN Fabric สร้าง Any-to-Any Path ผ่าน vSmart Policy |
| **M2** (สาขาใหม่ไม่กระทบสาขาเดิม) | **ล้มเหลวชัดเจน** — เพิ่ม 1 สาขา = ต้องสร้าง Tunnel ใหม่ไปยัง**ทุกสาขาเดิม 12 แห่ง** หมายความว่าต้องแก้ Config สาขาเดิมทุกตัว (ขัด M2 ตรงๆ) | **ผ่าน** — เพิ่มสาขาใหม่แค่ Config Spoke ตัวใหม่ชี้ไปที่ Hub เดิม ไม่ต้องแก้ไข Spoke เดิมแม้แต่ตัวเดียว | **ผ่าน** — Onboard cEdge ใหม่ผ่าน vManage/Zero-touch Provisioning โดย Sites เดิมไม่ต้องแก้ Config |
| **M3** (ลดงบ >40%) | ผ่าน (Internet ถูกกว่า MPLS) แต่ **Operational Cost สูงจากความซับซ้อนของ O(n²) Tunnel** ที่ต้อง Manage ด้วยมือ (66 Tunnel ตอนนี้ จะเป็น 120 Tunnel เมื่อครบ 16 สาขา) | ผ่าน — Internet ถูกกว่า MPLS และ Operational Overhead ต่ำกว่า Option A มาก (Tunnel Count = จำนวน Spoke ไม่ใช่ O(n²)) | ผ่านด้าน Transport Cost แต่ **มี License/Controller Cost เพิ่มเติม** (vSmart/vManage Subscription) ที่อาจกิน Margin ของการประหยัดค่า WAN ไปบางส่วน |
| **B1** (Application-aware, Bonus) | ไม่รองรับ (ไม่ใช่ปัญหาเพราะเป็น Bonus) | ไม่รองรับ Native (ต้องเสริมด้วย PBR/IP SLA แยกถ้าจะทำ) | **รองรับ Native เต็มรูปแบบ** — จุดแข็งที่สุดของ Option นี้ |

### เหตุผลของคำตอบที่ดีที่สุด: **Option B — DMVPN Phase 3 Dual-hub**

**ทำไมไม่เลือก Option A**: ล้มเหลวตรงๆที่ M2 (Requirement ที่บังคับชัดเจนที่สุดข้อหนึ่ง) — Full-mesh
แบบ Manual หมายความว่าทุกครั้งที่เพิ่มสาขาใหม่ ต้องไปแก้ Config ของสาขาเดิมทั้ง 12 แห่งเพื่อสร้าง
Tunnel ใหม่ ขัดกับ "ต้องไม่ต้องแก้ไข Configuration ของสาขาเดิมที่มีอยู่แล้วแม้แต่ตัวเดียว" อย่าง
ชัดเจนที่สุดในทั้ง 3 ตัวเลือก — นี่คือ Fail แบบ Hard Requirement ไม่ใช่แค่ "ด้อยกว่า"

**ทำไมไม่เลือก Option C ทั้งที่ดูดีที่สุดในตาราง**: นี่คือจุดที่โจทย์ทดสอบหลักการจาก
[Part 71 Step 704](part-071-ccie-lab-blueprint-overview.md) โดยตรง — **"เลือก Feature ที่พอดี ไม่ใช่
ที่ซับซ้อนที่สุด"** Option C ตอบ M1-M2 ได้แบบเดียวกับ Option B และมีจุดแข็งเพิ่มที่ B1 (Application-
aware Routing) แต่**โจทย์ระบุไว้ชัดเจนว่า B1 เป็น Bonus ของเฟสถัดไป ไม่ใช่ Requirement ของเฟสนี้**
— การเลือก Option C เพื่อได้ Feature ที่ยังไม่ต้องใช้ในเฟสนี้ หมายความว่าองค์กรต้องจ่าย **License/
Controller Cost ของ SD-WAN FabIC เต็มรูปแบบ** (M3 ต้องการลดงบ >40% — ยิ่งมี Cost เพิ่มที่ไม่จำเป็น
ยิ่งเสี่ยงทำ M3 ไม่สำเร็จ) เพื่อความสามารถที่ **"เผื่อไว้ก่อนแต่ยังไม่ได้ใช้งานจริง"** — นี่คือ
Over-engineering ตามนิยามในเกณฑ์ที่ 3 ของ Step 951 (Fit) แม้ Option C จะ "เก่งกว่า" ในทางเทคนิคก็ตาม

**สรุปคำตอบที่ถูกต้อง**: **Option B (DMVPN Phase 3 Dual-hub)** เพราะ:

1. ผ่าน M1 อย่างสมบูรณ์ (Dynamic Spoke-to-Spoke ผ่าน NHRP)
2. ผ่าน M2 อย่างสมบูรณ์ (เพิ่ม Spoke ใหม่ไม่กระทบ Spoke เดิม — จุดอ่อนที่สุดของ Option A)
3. ผ่าน M3 ได้ดีที่สุดในบรรดา 3 ตัวเลือก (ต้นทุน Operational ต่ำกว่า A, ไม่มี License Overhead แบบ C)
4. ยอมรับการไม่ได้ B1 อย่างมีเหตุผล เพราะโจทย์บอกตรงๆว่ายังไม่ต้องการในเฟสนี้ — **ถ้า B1 ถูกเลื่อน
   มาเป็น Mandatory ในอนาคต (เฟสถัดไปตามที่โจทย์ Hint ไว้) การ Migrate จาก DMVPN ไปยัง SD-WAN ค่อย
   พิจารณาใหม่ตอนนั้นได้ ไม่ใช่การลงทุนล่วงหน้าที่สูญเปล่าถ้าไม่เกิดขึ้นจริง**

> **บทเรียนสำคัญของ Scenario นี้**: ข้อสอบ MCQ-with-Justification มักใส่ตัวเลือกที่ "ดูเหมือนดีที่สุด
> ทางเทคนิค" (Option C ในที่นี้) ไว้เป็นตัวลวงเสมอ — Grader ให้คะแนนจากการที่ Candidate**อ่านคำว่า
> "Bonus/ไม่บังคับ" ออกและไม่หลงไปกับ Feature ที่ดูน่าตื่นเต้นกว่าแต่ไม่ตรง Requirement ที่บังคับจริง**

---

## Step 960 — Full Lab: Design Scenario ผสมข้าม Domain เต็มรูปแบบ + สรุปกลยุทธ์การสอบ

### บริบทโจทย์ (ยากขึ้นกว่า Scenario 1-4 — ผสม Routing + Security + Automation + Cloud)

บริษัท **TransGlobal Logistics** เป็นผู้ให้บริการโลจิสติกส์ระดับภูมิภาค กำลังขยายจาก 4 Hub เดิม
(กรุงเทพฯ, สิงคโปร์, โฮจิมินห์, จาการ์ตา) เพิ่มอีก **3 Regional Hub ใหม่** ภายใน 12 เดือน พร้อมย้าย
Workload บางส่วนขึ้น **Public Cloud (AWS + Azure)** และมีข้อกำหนดด้าน Compliance ที่ซับซ้อนกว่า
Scenario ก่อนหน้าทั้งหมด — โจทย์นี้จงใจดึงความรู้จาก **Part 71-94 หลายโดเมนมาผสมในโจทย์เดียว**
เพื่อจำลองความยากของ Design Task ท้ายๆของ Design Module จริง

**Requirement เต็มรูปแบบ**

1. **D1 (Routing/Architecture)**: Hub ทั้ง 7 แห่ง (4 เดิม + 3 ใหม่) ต้องเชื่อมต่อกันด้วย Backbone ที่
   รองรับ Traffic Engineering ระดับภูมิภาค — Hub บางคู่มี Traffic ปริมาณสูง (กรุงเทพฯ↔สิงคโปร์) ต้อง
   เลือกเส้นทางที่ Optimize ได้ ไม่ใช่แค่ Routing แบบ Shortest-path เดียว
2. **D2 (Cloud)**: ระบบ Tracking Application (Real-time Shipment Tracking) จะย้ายไปรันบน **AWS
   (Region สิงคโปร์)** ทั้งหมด ต้องมี Connectivity ที่ **Redundant และ Latency ต่ำ** จากทุก Hub ไปยัง
   AWS — ธุรกิจนี้ Sensitive ต่อ Downtime สูง (ระบบ Tracking หยุดทำงาน = ลูกค้าโทรมาร้องเรียนทันที)
3. **D3 (Cloud)**: ระบบ HR/Finance ภายในจะย้ายไปใช้ **Azure (Region ที่ใกล้ HQ กรุงเทพฯ ที่สุด)**
   แยกจาก D2 — ไม่ต้องการ Redundancy ระดับเดียวกับ D2 (ยอมรับ Downtime สั้นๆได้ในกรณีฉุกเฉิน)
4. **D4 (Security/Compliance)**: Hub ที่ Jakarta ต้องเก็บข้อมูลลูกค้าในประเทศอินโดนีเซียตามกฎหมาย
   คุ้มครองข้อมูลท้องถิ่น (**Data Residency Requirement** — คล้าย GDPR ของยุโรปแต่เป็นกฎหมายท้องถิ่น
   อินโดนีเซีย) ห้ามให้ข้อมูลลูกค้าของ Jakarta Hub ถูกส่งไปประมวลผลที่ Region Cloud นอกประเทศ
5. **D5 (Security/Zero Trust)**: ทุก Hub ใหม่ต้องผ่านการ Segment แบบ Zero Trust ตั้งแต่วันแรกที่เปิด
   ใช้งาน — ไม่ยอมรับช่วง "Transition Period" ที่ Security หลวมกว่ามาตรฐาน แม้จะเร่งเปิดใช้งานเร็ว
6. **D6 (Automation)**: ด้วยจำนวน Hub ที่เพิ่มขึ้นและ Config ที่ต้องสอดคล้องกันทุก Hub (Security
   Policy, QoS Baseline, Routing Template) การ Deploy Hub ใหม่แต่ละแห่งต้องใช้ **Automation Pipeline**
   ไม่ใช่ Manual Configuration ทีละอุปกรณ์ — ทีม Network Engineering มีจำกัดเช่นเดิม (Priority แบบ
   เดียวกับ Scenario 3)
7. **D7 (Assurance)**: ผู้บริหารต้องการ **Visibility แบบ Real-time** ว่า Hub ไหนมีปัญหา Latency/
   Packet Loss ระหว่างทางไปยัง Cloud หรือ Hub อื่น เพราะธุรกิจ Logistics Sensitive ต่อ Latency มาก
8. **D8 (Legacy Constraint)**: Hub เดิมที่โฮจิมินห์ยังมีระบบ Multicast-based Video Conferencing รุ่น
   เก่าที่ใช้งานประจำวันในการประชุมข้ามภูมิภาค ต้องรักษาให้ทำงานต่อได้ในทุก Design ใหม่

### Model Design แบบสรุป

**D1 — Backbone**: ใช้ **MPLS L3VPN เป็น Backbone หลัก** ระหว่าง 7 Hub (Part 86-87) เสริมด้วย
**BGP Traffic Engineering** (Local Preference/AS-Path/MED ปรับเส้นทางคู่ กรุงเทพฯ↔สิงคโปร์ให้ใช้ Path
ที่ Optimize แล้ว ตาม Part 74) — เลือก MPLS L3VPN เพราะรองรับ Traffic Engineering ระดับ Provider ได้
ดีกว่า DMVPN สำหรับ Scale ระดับภูมิภาคข้ามประเทศแบบนี้ (ต่างจาก Scenario 4 ที่เป็น Intra-country และ
ไม่มี Requirement ด้าน TE ชัดเจนเท่านี้)

**D2 — AWS Tracking (Latency-sensitive)**: ใช้ **Dual Direct Connect/ExpressRoute-equivalent
(AWS Direct Connect คู่ จาก 2 เส้นทางกายภาพต่างกัน)** จาก Hub สิงคโปร์ (ใกล้ AWS Region สิงคโปร์ที่สุด)
เป็น Primary Cloud Gateway ให้ Hub อื่นทั้งหมด Route ผ่าน Backbone ไปออกที่สิงคโปร์ (Part 93) — เสริม
ด้วย **Site-to-Site VPN สำรอง** ผ่าน Internet เป็น Backup Path กรณี Direct Connect ล้ม (ตรงกับ
หลักการ Redundancy จาก Scenario 1-2: ไม่มี Path เดียวสำหรับ Application ที่ Sensitive)

**D3 — Azure HR/Finance (ยอมรับ Downtime สั้นได้)**: ใช้ **Site-to-Site VPN เดี่ยว (ไม่ Dual)** จาก
HQ กรุงเทพฯ ไปยัง Azure Region ที่ใกล้ที่สุด — **จงใจไม่ทำ Redundancy ระดับเดียวกับ D2** เพราะโจทย์
ระบุชัดว่า Application นี้ยอมรับ Downtime ได้ (Priority ต่างจาก D2 — สอนเรื่อง "ไม่ใช่ทุก Application
ต้องได้ Redundancy ระดับสูงสุดเท่ากันหมด" ซึ่งเป็นการ Over-engineering ถ้าทำ Dual ให้ทุกอย่างเท่ากัน)

**D4 — Jakarta Data Residency**: Jakarta Hub ต้องมี **Local Cloud Presence ในประเทศอินโดนีเซีย
เท่านั้น** (AWS/Azure Region ที่ตั้งอยู่ในอินโดนีเซีย) แยกจาก Workload ของ D2/D3 อย่างเด็ดขาด — ใช้
**VRF-Lite แยก Jakarta Traffic ไม่ให้ปนกับ Backbone หลักที่ส่งไป Region สิงคโปร์** และ Routing Policy
บล็อกไม่ให้ Traffic ของ Jakarta ถูก Route ผ่าน Path ที่ออกนอกประเทศไปยัง Cloud Region อื่นโดยเผลอ
(ป้องกัน Data Residency Violation จาก Routing Design ที่ผิดพลาด ไม่ใช่แค่ Policy บนกระดาษ)

**D5 — Zero Trust ตั้งแต่วันแรก**: ทุก Hub ใหม่ Deploy ด้วย **Zero Trust Segmentation Template** ที่
กำหนดไว้ล่วงหน้า (Part 94) ผ่าน Automation Pipeline (เชื่อมกับ D6 โดยตรง) — ไม่มี "ช่วง Transition"
เพราะ Template ถูกสร้างให้ Deploy พร้อม Security Policy ตั้งแต่ครั้งแรกที่ Provision อุปกรณ์ ไม่ใช่
Deploy ก่อนแล้วมา "ใส่ Security เพิ่มทีหลัง"

**D6 — Automation Pipeline**: ใช้ **Ansible Playbook + Git Repository เป็น Source of Truth** (Part
48, 79) เก็บ Template ของ Routing/QoS/Security Baseline ทุก Hub ไว้เป็น Configuration as Code —
Hub ใหม่แต่ละแห่ง Deploy ผ่าน Pipeline เดียวกัน รับประกันความสอดคล้อง (Consistency) ที่ Manual
Configuration ให้ไม่ได้ และลด Workload ทีมที่มีจำกัด (เชื่อมกับหลักการเดียวกับ Scenario 3: แก้ปัญหา
"ทีมเล็กต้องดูแล Site จำนวนมาก" ด้วย Automation ไม่ใช่ด้วยการเพิ่มคน)

**D7 — Assurance**: ใช้ **Model-driven Telemetry (YANG Push/gRPC)** (Part 80) Stream ข้อมูล Latency/
Packet Loss แบบ Real-time จากทุก Hub เข้าสู่ Dashboard กลาง — เลือก Telemetry แบบ Push ไม่ใช่ SNMP
Polling แบบเดิม เพราะ Requirement ต้องการ "Real-time" ซึ่ง Polling-based Monitoring (Part 17) ตอบ
ไม่ได้ดีพอในระดับ Latency ที่ธุรกิจ Logistics ต้องการเห็นทันที

**D8 — Legacy Multicast**: รักษา **PIM-SM Multicast Routing ข้าม Backbone MPLS L3VPN เดิมที่โฮจิมินห์**
(Part 41-42, 75) ไว้ไม่เปลี่ยนแปลง — ตรวจสอบให้แน่ใจว่า BGP Traffic Engineering ใน D1 ไม่ไปเปลี่ยน
เส้นทาง Unicast จนกระทบ RPF Check ของ Multicast ที่ Hub นี้ (จุดที่ต้อง Cross-check ระหว่าง Requirement
สองข้อที่อาจชนกันโดยไม่ตั้งใจ — D1 กับ D8)

### ตาราง Requirements-Traceability เต็มรูปแบบ

| Requirement | Design Element | เหตุผล |
|---|---|---|
| D1 — Backbone รองรับ TE | MPLS L3VPN + BGP Traffic Engineering (Local Pref/AS-Path/MED) | เลือกเส้นทาง กรุงเทพฯ↔สิงคโปร์ที่ Optimize แยกจาก Shortest-path เดียว |
| D2 — AWS Tracking, Redundant/Low-latency | Dual AWS Direct Connect จาก Hub สิงคโปร์ + Site-to-Site VPN สำรอง | Redundancy ระดับสูงตรงกับ Sensitivity ของ Application |
| D3 — Azure HR/Finance, ยอมรับ Downtime สั้นได้ | Site-to-Site VPN เดี่ยว จาก HQ | ไม่ Over-engineer Redundancy ให้ Application ที่ Priority ต่ำกว่า D2 |
| D4 — Jakarta Data Residency | VRF-Lite แยก Traffic + Local In-country Cloud Region + Routing Policy กันไม่ให้ออกนอกประเทศ | ป้องกัน Compliance Violation ด้วย Routing Design ไม่ใช่แค่ Policy เอกสาร |
| D5 — Zero Trust ตั้งแต่วันแรก | Zero Trust Segmentation Template ฝังใน Automation Pipeline ตั้งแต่ Provisioning | ไม่มีช่วง Transition ที่ Security หลวม |
| D6 — Automation, ทีมเล็ก | Ansible Playbook + Git Source of Truth สำหรับทุก Hub Template | ลด Manual Workload, รับประกัน Consistency |
| D7 — Real-time Visibility | Model-driven Telemetry (YANG Push/gRPC) | Real-time ตรงตาม Requirement ดีกว่า SNMP Polling |
| D8 — Legacy Multicast ที่โฮจิมินห์ | รักษา PIM-SM เดิม + Cross-check RPF กับ BGP TE ใน D1 | ป้องกัน Requirement D1 ทำลาย Requirement D8 โดยไม่ตั้งใจ |

> สังเกตแถวสุดท้าย (D8) — นี่คือระดับความยากที่สูงกว่า Scenario 1-4 ทั้งหมด: **ต้องตรวจสอบว่า Design
> Element ที่ตอบ Requirement หนึ่ง ไม่ไปทำลาย Requirement อีกข้อโดยไม่ตั้งใจ** (Requirement
> Interaction) ซึ่งเป็นทักษะที่ Design Module ระดับยากที่สุดของข้อสอบจริงวัด — ไม่ใช่แค่ตอบแต่ละข้อ
> แยกกันให้ครบ แต่ต้องเห็นว่า Requirement หลายข้อ**เกี่ยวเนื่องกัน**ด้วย

### สรุป: Design Module Exam Strategy — กลยุทธ์การสอบ Design Module ฉบับสมบูรณ์

จาก 4 Scenario + Full Lab ที่ฝึกมาทั้ง Part นี้ สรุปเป็นกลยุทธ์ที่ใช้ได้จริงในห้องสอบ:

| ขั้น | สิ่งที่ต้องทำ | อ้างอิงจาก Step |
|---|---|---|
| 1 | อ่านโจทย์ **ทั้งหมดก่อน** แล้ว List Requirement ทุกข้อเป็นตาราง (รวมข้อที่ซ่อนอยู่ในประโยค Constraint) | 951, 953 |
| 2 | แยก Requirement เป็น **Mandatory vs Bonus/Preference** — ระวังคำว่า "ต้อง" vs "ควร", "บังคับ" vs "เผื่อไว้" | 958-959, Part 71 Step 708 |
| 3 | ออกแบบ Solution ที่ **"พอดี" กับ Requirement** — ไม่เลือกทางที่ซับซ้อน/แพงกว่าที่จำเป็น แม้จะดูเก่งกว่าทางเทคนิค | 951, 959 |
| 4 | ถ้าโจทย์บีบให้เลือกได้ทางเดียว (Prioritization) ให้วิเคราะห์ **ปัญหาไหนยังไม่มีทางแก้และมี Priority สูงสุด** ไม่ใช่เลือกจากความน่าตื่นเต้นของเทคโนโลยี | 957 |
| 5 | เขียน **Requirements-Traceability Table** เสมอ — ทุก Requirement ต้องมีแถว ทุก Design Element ต้องมี Requirement รองรับ | 951, 953, 955, 960 |
| 6 | ตรวจสอบ **Requirement Interaction** — Design Element ที่ตอบข้อหนึ่ง อาจไปกระทบข้ออื่นโดยไม่ตั้งใจ (ยิ่งโจทย์ซับซ้อนข้ามหลาย Domain ยิ่งต้องเช็คจุดนี้) | 960 |
| 7 | เขียน **Tradeoff ของทางที่ไม่เลือกอย่างตรงไปตรงมา** พร้อมเหตุผลว่าทำไมยอมรับได้ — ไม่บอกว่าทางที่เลือก "ไม่มีข้อเสียเลย" | 957 |
| 8 | บริหารเวลา — Design Module มีหลาย Task ในเวลาจำกัด อย่าทำ Task ใด "สมบูรณ์แบบเกินคะแนนที่ให้" จนไม่มีเวลาทำ Task อื่น | Part 71 Step 705, 708 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 96

1. Design Module ของ CCIE EI Lab Exam มีรูปแบบงาน (Task Type) กี่แบบ อะไรบ้าง และรูปแบบไหนที่
   Candidate มักเข้าใจผิดว่า "ง่ายกว่า" ทั้งที่ Grading Rubric ยังเข้มงวดเหมือนกัน?
2. Requirements-Traceability Table มีคอลัมน์อะไรบ้าง และช่วยตอบเกณฑ์การให้คะแนน 4 ข้อของ Grader
   (Coverage, Correctness, Fit, Justification) ได้อย่างไร?
3. ใน Design Scenario 2 (Step 954-955) ข้อบกพร่องเรื่อง Router ตัวเดียวทำหน้าที่ทั้ง WAN Edge และ
   Default Gateway (F2) ต่างจากข้อบกพร่องเรื่อง OSPF Totally Stub รับ Default Route เส้นทางเดียว (F3)
   อย่างไร ทำไมต้องแยกวิเคราะห์เป็นสองจุด?
4. ในโจทย์ Prioritization แบบ Scenario 3 (Step 956-957) หลักการตัดสินใจที่ถูกต้องคืออะไร และทำไม
   "เลือกเทคโนโลยีที่ดูล้ำกว่า" ถึงเป็นเกณฑ์ที่ผิด?
5. อธิบายความหมายของ "Requirement Interaction" ที่ปรากฏใน Design Scenario เต็มรูปแบบของ Step 960
   (D1 กับ D8) พร้อมยกตัวอย่างว่าทำไมการตอบ Requirement แต่ละข้อแยกกันให้ครบ ยังไม่พอสำหรับ Design
   ระดับซับซ้อนสูงสุด?

**เฉลย:**

1. มี **3 รูปแบบ**: (1) Design from Requirements — สร้าง Design เต็มจากโจทย์เปล่า (2) Critique/
   Correct a Given Design — หาจุดบกพร่องใน Design ที่มีอยู่แล้ว (3) Multiple-Choice with
   Justification — เลือกคำตอบพร้อมอธิบายเหตุผล **รูปแบบที่ 3 มักถูกเข้าใจผิดว่าง่ายกว่า** เพราะดูเหมือน
   แค่กาคำตอบ แต่ Cisco ให้คะแนนจาก**คำอธิบายเหตุผล**เป็นหลัก กาถูกแต่อธิบายผิด/ไม่ครบมักไม่ได้คะแนนเต็ม
2. มี 3 คอลัมน์: **Requirement (จากโจทย์) / Design Element (สิ่งที่เลือกใช้) / เหตุผล** — ตอบ
   Coverage เพราะบังคับ List Requirement ครบก่อนออกแบบ, ตอบ Fit เพราะทุก Design Element ต้องมี
   Requirement รองรับ (ป้องกัน Over-engineering), ตอบ Correctness เพราะบังคับอธิบายว่า Design Element
   ทำงานตอบ Requirement นั้นได้จริงอย่างไร, และตอบ Justification โดยตรงเพราะคอลัมน์ที่ 3 คือคำอธิบาย
   สำเร็จรูปที่ผูกกับ Requirement เฉพาะเจาะจงอยู่แล้ว
3. F2 คือ Single Point of Failure ด้าน **Hardware/Platform** (Router ตัวเดียวทำสองบทบาท ถ้าตัวนี้
   พัง ทุกอย่างพังพร้อมกัน) แก้ด้วยการเพิ่ม Redundant Hardware/แยก Layer (Core/Dist แยกจาก WAN Edge)
   ส่วน F3 คือ Single Point of Failure ด้าน **Path/Circuit** (มี Link เดียวไปยัง WAN แม้ Hardware จะ
   Redundant ก็ตาม) แก้ด้วยการเพิ่ม Backup Path (Internet+IPsec) — ต้องแยกวิเคราะห์เพราะแก้คนละวิธี
   ถ้าแก้แค่อย่างเดียว (เช่น เพิ่ม Router สำรองแต่ยังใช้ Link เดียว) ก็ยังมี SPOF เหลืออยู่
4. หลักการที่ถูกต้องคือ **วิเคราะห์ว่า Priority ข้อไหนเป็นปัญหาจริงที่ยังไม่มีทางแก้ ณ ปัจจุบัน และมี
   น้ำหนัก/Deadline ทางธุรกิจสูงสุด** แล้วเลือกทางที่แก้ปัญหานั้นได้ตรงที่สุด — "เลือกเทคโนโลยีที่ดูล้ำ
   กว่า" เป็นเกณฑ์ที่ผิดเพราะวัดจาก "ความน่าตื่นเต้นทางเทคนิค" ไม่ใช่จาก "ผลตอบแทนต่อการลงทุนเทียบกับ
   ปัญหาทางธุรกิจจริง" ซึ่งอาจนำไปสู่การลงทุนซ้ำในจุดที่ Baseline ดีอยู่แล้ว (Diminishing Returns)
   ขณะที่ปัญหาที่มี Priority สูงกว่าจริงยังไม่ถูกแก้
5. Requirement Interaction คือสถานการณ์ที่ **Design Element ซึ่งตอบ Requirement ข้อหนึ่งได้ดี อาจไป
   กระทบหรือทำลาย Requirement อีกข้อโดยไม่ตั้งใจ** — ตัวอย่างจาก D1/D8: การทำ BGP Traffic Engineering
   เพื่อ Optimize เส้นทาง Unicast ระหว่างกรุงเทพฯ↔สิงคโปร์ (ตอบ D1) อาจเปลี่ยนเส้นทาง Routing ที่
   Multicast RPF Check ที่โฮจิมินห์ใช้อ้างอิงอยู่ (กระทบ D8) — การตอบ Requirement แต่ละข้อแยกกันให้ครบ
   (เช่น ทำตาราง Traceability ที่มี 8 แถวครบ) ยังไม่พอ เพราะต้อง **ตรวจสอบผลกระทบข้ามแถวกันด้วย** ซึ่ง
   เป็นทักษะที่ยากขึ้นไปอีกขั้นตามความซับซ้อนของโจทย์ที่ผสมหลาย Domain เข้าด้วยกัน

---

## สรุป Part 96

Part นี้ฝึกทักษะ **Design Module** ของ CCIE EI Lab Exam อย่างเข้มข้นผ่าน 4 Scenario ที่ครอบคลุมทั้ง
3 รูปแบบงานจริง (Design from Requirements, Critique, Multiple-Choice-with-Justification) บวก 1 Full
Lab ที่ผสมข้าม Domain ระดับยากที่สุด — หัวใจของ Part นี้ไม่ใช่การจำ Solution สำเร็จรูป แต่คือ
**กระบวนการคิดที่ทำซ้ำได้**: List Requirement ให้ครบ (รวมที่ซ่อนอยู่), แยก Mandatory กับ Bonus,
เลือก Solution ที่พอดีไม่ใช่ที่ซับซ้อนที่สุด, ตัดสินใจ Prioritization จากปัญหาจริงไม่ใช่จากความน่า
ตื่นเต้นของเทคโนโลยี, เขียน Requirements-Traceability Table ทุกครั้งเพื่อพิสูจน์ Coverage และเป็น
Justification สำเร็จรูป, และตรวจสอบ Requirement Interaction ในโจทย์ที่ซับซ้อนขึ้น — ทักษะทั้งหมดนี้
คือสิ่งที่แยก Candidate ที่ "รู้ทฤษฎีครบ" ออกจาก Candidate ที่ "ทำคะแนน Design Module ได้จริง"

**พร้อมสำหรับ Part 97**: จาก Design Module ที่ตอบผ่านเอกสาร/Diagram เท่านั้น Part ถัดไปจะเปลี่ยนโหมด
ไปสู่ **Deploy Module** — เอา Design ที่ตัดสินใจไว้ (เหมือนที่ฝึกใน Part นี้) มา **Config จริงบน
อุปกรณ์ให้ตรงกับ Requirement ทุกข้อภายในเวลาจำกัด** ซึ่งเป็นด่านที่ใช้เวลามากที่สุดของข้อสอบจริง (~5
ชั่วโมง) และเป็นจุดที่ Design ที่ผิดตั้งแต่ต้นจะส่งผลเสียหายลุกลามมากที่สุด

**ไปต่อ:** [Part 97 — CCIE Full Lab Scenario #3 (Deploy Module) →](part-097-ccie-lab-scenario-3-deploy.md)
