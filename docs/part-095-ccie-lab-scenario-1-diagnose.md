# Part 95 — CCIE Full Lab Scenario #1: Diagnose Module
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 941–950 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 94 — Zero Trust Network Architecture](part-094-zero-trust-network-architecture.md)
> ที่ปิดท้ายสาย "Technology Deep-Dive" ทั้งหมดของหลักสูตร (Part 1-94) — จากจุดนี้ไปจนถึง Part 100
> จะไม่มีเทคโนโลยีใหม่ให้เรียนอีกแล้ว มีแต่ **CCIE Full Lab Scenario** ที่จำลอง 4 รูปแบบงานจริงของ
> ห้องสอบ CCIE ตามที่ [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) แบ่งไว้ (Design,
> Deploy, Optimize/Operate, Diagnose) โดย **Part นี้เป็น Part แรกและจำลอง Task Type "Diagnose"**
>
> Part นี้ใช้ **กระบวนการคิด** ที่ [Part 91 — Advanced Troubleshooting Methodology](part-091-advanced-troubleshooting-methodology.md)
> สอนไว้ทั้งหมด (Pattern-Matching, Bisection, Scientific Method, Baseline Comparison, การแปล
> คำร้องเรียนคลุมเครือ, Packet Capture อย่างมีเป้าหมาย, Multi-Domain Triage, Time-Boxing, และการ
> บันทึกระหว่างทาง) แต่เปลี่ยนเงื่อนไขให้ตรงกับสภาพห้องสอบจริงที่สุด: **ผู้เรียนจะไม่มีสิทธิ์เข้าถึง
> อุปกรณ์จริงเลยแม้แต่เครื่องเดียว** มีเพียง **เอกสารหลักฐาน (Exhibit)** ที่จำกัดต่อ Ticket เท่านั้น
> ต้องวินิจฉัย Root Cause จากสิ่งที่ให้มาล้วนๆ — Ticket ทั้ง 5 ใบในภาคนี้จงใจ **ไม่บอกโดเมนเทคโนโลยี
> ล่วงหน้า** ผู้เรียนต้องใช้ความรู้จากทุก Part ตั้งแต่ Part 1 ถึง Part 94 ประกอบกับ Methodology จาก
> Part 91 ในการหาคำตอบเอง เหมือนที่ผู้สมัครสอบ CCIE ตัวจริงต้องทำ

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 941 | รูปแบบข้อสอบ CCIE Diagnose Module ตัวจริง — Read-Only, Ticket-Based, ไม่มีสิทธิ์เข้าอุปกรณ์ |
| 942 | Ticket 1 — หลักฐานชุดที่ 1 (โดเมนไม่เปิดเผย) |
| 943 | Ticket 1 — เฉลยและคำอธิบาย Root Cause โดยละเอียด |
| 944 | Ticket 2 — หลักฐานชุดที่ 2 (Topology Diagram + Partial Config) |
| 945 | Ticket 2 — เฉลยและคำอธิบาย Root Cause โดยละเอียด |
| 946 | Ticket 3 — หลักฐานชุดที่ 3 (โดเมนไม่เปิดเผย) |
| 947 | Ticket 3 — เฉลยและคำอธิบาย Root Cause โดยละเอียด |
| 948 | Ticket 4 — หลักฐานชุดที่ 4 (Cross-Domain, พบน้อยในสื่อฝึกทั่วไป) |
| 949 | Ticket 4 — เฉลยและคำอธิบาย Root Cause โดยละเอียด |
| 950 | Full Lab — Ticket 5 (โจทย์ยากที่สุด มี Red Herring) + กลยุทธ์การสอบ Diagnose Module |

---

## Step 941 — รูปแบบข้อสอบ CCIE Diagnose Module ตัวจริง

### 941.1 ความเข้าใจผิดที่พบบ่อยที่สุดเกี่ยวกับ Diagnose Module

ผู้เรียนจำนวนมากคิดว่า "Diagnose" แปลว่าได้ SSH เข้าไปที่อุปกรณ์จริงแล้วรัน `show`/`debug` เองเหมือนที่
ฝึกมาตลอด [Part 56-69](part-056-eigrp-troubleshooting.md) — **นี่ผิดโดยสิ้นเชิงสำหรับ Diagnose
Module** ตามที่ [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) อธิบายไว้ Diagnose
เป็น Task Type ที่ฝังอยู่ใน Deploy, Operate Module ในรูปของ **Ticket** และ Ticket แต่ละใบมาพร้อม
**เอกสารชุดจำกัด (Exhibit)** เท่านั้น — **ไม่มี Console/SSH ไปยังอุปกรณ์จริงให้ในหน้าจอ Diagnose
เลย** ผู้สมัครต้องอ่าน Exhibit ที่ให้มา แล้วเลือก/พิมพ์ **Root Cause ที่ถูกต้องที่สุด** พร้อมบางครั้ง
ต้องระบุ **คำสั่งแก้ไขที่ถูกต้อง** โดยไม่มีโอกาสได้ "ลองรันดูก่อน" แม้แต่ครั้งเดียว

### 941.2 เหตุผลที่ข้อสอบออกแบบมาแบบนี้

| เหตุผล | คำอธิบาย |
|---|---|
| วัดทักษะที่ต่างจาก Deploy โดยตรง | Deploy วัด "พิมพ์ config ถูกไหม" ส่วน Diagnose วัด **"อ่าน Evidence แล้วสรุปถูกไหม"** — เป็นทักษะที่ต้องใช้ตอนอ่าน Log/Ticket จากทีมอื่นในงานจริงที่ไม่มีสิทธิ์ Login เองเสมอไป (เช่น ทีม Security ส่ง Config Snippet มาให้ทีม Network วิเคราะห์) |
| จำลองสถานการณ์ Post-Mortem/Change Review จริง | ในงานจริงหลายครั้งวิศวกรต้องอ่าน Config Diff/Log ที่คนอื่นเก็บมาแล้วตัดสินโดยไม่มี Live Access (เช่น Incident ที่ปิดไปแล้ว ต้องทำ RCA จาก Log เก่า) |
| จำกัดเวลาต่อ Ticket ให้ยุติธรรม | ถ้าให้ Login จริงได้ ผู้สมัครที่ Random ลองคำสั่งเรื่อยๆ (Shotgun ตาม [Part 91 Step 903](part-091-advanced-troubleshooting-methodology.md)) อาจ "เจอโดยบังเอิญ" ได้ — การให้ Exhibit ตายตัวบังคับให้ต้องมี**เหตุผลที่พิสูจน์ได้จาก Evidence ที่ให้มาเท่านั้น** |

### 941.3 รูปแบบ Exhibit ที่พบได้ในข้อสอบจริง (และที่ Part นี้จะจำลอง)

| ประเภท Exhibit | ตัวอย่าง | ใช้บ่อยกับ Ticket ประเภท |
|---|---|---|
| คำร้องเรียนจากลูกค้า/ผู้ใช้ (Trouble Ticket Text) | "ผู้ใช้ที่ Building A เข้า Intranet ไม่ได้บางส่วน" | ทุก Ticket (จุดเริ่มต้นเสมอ) |
| `show` Command Output (ตัดมาเฉพาะส่วนที่เกี่ยวข้อง) | `show ip bgp summary`, `show ip ospf neighbor` | Routing Protocol, L2/L3 |
| Running-config Excerpt (ไม่ใช่ทั้งไฟล์) | `neighbor x.x.x.x route-map ... in` | ทุกโดเมนที่มี Config ผิด |
| Topology Diagram (ASCII/รูปภาพ) | แผนผัง Area/VLAN/Neighbor | OSPF/EIGRP Area Design, Redundancy |
| Log/Timestamp Correlation | Syslog, Ansible Job Log, Change Ticket | Automation, Baseline Comparison |
| Packet Capture Summary (ข้อความสรุป ไม่ใช่ .pcap เต็ม) | สรุป TCP Retransmission Pattern | Application-Network Boundary |

### 941.4 กฎ 3 ข้อที่ต้องยึดตลอด Part นี้ (และในห้องสอบจริง)

1. **ห้ามสมมติข้อมูลที่ไม่ได้ให้มา** — ถ้า Exhibit ไม่ได้บอก MTU ของ Interface ห้ามสมมติว่ารู้ MTU
   (ต่างจาก Part 56-69 ที่ผู้เรียน "รัน show เพิ่มเองได้" ทุกเมื่อที่ต้องการ)
2. **ทุก Root Cause ต้องอธิบายได้ครบทุกอาการที่ Ticket รายงาน** — ถ้า Root Cause ที่เลือกอธิบายได้แค่
   บางอาการ แปลว่ายังไม่ใช่คำตอบที่ถูกที่สุด (หลักการเดียวกับ [Part 91 Step 901.4](part-091-advanced-troubleshooting-methodology.md)
   ข้อ 2: "Layer ไหนอธิบาย Symptom **ทั้งหมด** ได้")
3. **ต้องแยกแยะ Evidence ที่ "เกี่ยวข้อง" กับ "ไม่เกี่ยวข้อง" (Red Herring)** — ไม่ใช่ทุก Exhibit ที่ให้
   มาคือเบาะแสจริงเสมอไป (ดู Ticket 5 ใน Step 950 ที่ทดสอบทักษะนี้ตรงๆ)

> **หมายเหตุสำคัญ**: Ticket 1-4 ใน Part นี้ (Step 942-949) แต่ละใบมี Root Cause เดียวชัดเจนไม่มี
> Red Herring เพื่อฝึกการอ่าน Exhibit ให้คล่องก่อน — **Ticket 5 ใน Step 950 เท่านั้น** ที่จะผสม
> Red Herring เข้ามาเต็มรูปแบบตามสภาพข้อสอบจริงที่ยากที่สุด

---

## Step 942 — Ticket 1: หลักฐานชุดที่ 1

> **คำเตือนก่อนอ่าน**: Ticket นี้ **ไม่ได้บอกชื่อโดเมนเทคโนโลยีที่เกี่ยวข้อง** ห้ามเปิดไปดูเฉลยใน
> Step 943 จนกว่าจะลองวิเคราะห์ด้วยตัวเองจาก Exhibit ที่ให้มาเท่านั้นก่อน — ไม่มีสิทธิ์ "ขอข้อมูล
> เพิ่มเติม" หรือ "ลองรันคำสั่งอื่น" ในสถานการณ์นี้ ตามกฎของ Diagnose Module ที่ Step 941 อธิบายไว้

### Ticket #DIAG-9501

**Exhibit A — คำร้องเรียนจากลูกค้า (ผ่าน Help Desk)**

> "ทีม Partner Integration แจ้งว่าสัปดาห์นี้เพิ่ม Subnet ใหม่ `172.20.60.0/24` (Partner Finance
> System) เข้ามาในการเชื่อมต่อ Site-to-Site ที่มีอยู่แล้ว เจ้าหน้าที่ฝ่ายบัญชีที่ใช้งานผ่าน
> `ACCESS-SW1` (VLAN 10) แจ้งว่าเข้าระบบ Finance System ของ Partner ใหม่นี้ไม่ได้เลย (Connection
> Timeout) แต่ระบบ CRM ของ Partner เดิม (`172.20.50.0/24`) ที่ใช้งานมาหลายเดือนแล้วยังใช้ได้ปกติ
> ทุกจุด ไม่มี Ticket อื่นที่เกี่ยวข้องกับ Internet หรือระบบภายในองค์กรเลยในช่วงเวลานี้"

**Exhibit B — `show ip bgp summary` (เก็บจาก `WAN-EDGE-1`)**

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000   50122   50098    93810    0    0 8d02h          3
1.1.1.22        4 65001   11402   11390     9012    0    0 8d02h        412
198.51.100.14   4 65010    2210    2198     3125    0    0 2w1d           2
```

**Exhibit C — Running-config Excerpt (`WAN-EDGE-1`, ส่วนที่เกี่ยวกับ Neighbor AS 65010 เท่านั้น)**

```
router bgp 65001
 neighbor 198.51.100.14 remote-as 65010
 neighbor 198.51.100.14 description PARTNER-RTR via Tunnel20
 neighbor 198.51.100.14 prefix-list PARTNER-IN in
!
ip prefix-list PARTNER-IN seq 5 permit 172.20.50.0/24
ip prefix-list PARTNER-IN seq 10 permit 172.20.51.0/24
```

**Exhibit D — `show ip bgp` (เก็บจาก `WAN-EDGE-1`, กรองเฉพาะ Path ที่มี AS 65010 ปรากฏ)**

```
WAN-EDGE-1# show ip bgp | include 65010
*> 172.20.50.0/24    198.51.100.14            0             0 65010 i
*> 172.20.51.0/24    198.51.100.14            0             0 65010 i
```

**Exhibit E — สรุปจากทีม Partner (ส่งมาพร้อม Ticket)**

> "ฝั่งเราส่ง `172.20.60.0/24` เข้า BGP Session เดียวกันตั้งแต่เช้าวันจันทร์ที่ผ่านมาแล้ว ยืนยันด้วย
> `show ip bgp neighbors <IP ของ WAN-EDGE-1> advertised-routes` ที่ฝั่งเราเห็น 3 Prefix ถูก
> Advertise ออกไปครบ (`.50.0/24`, `.51.0/24`, `.60.0/24`)"

**คำถาม**: จาก Exhibit ทั้งหมดข้างต้น Root Cause ของปัญหาคืออะไร และควรแก้ไขด้วยคำสั่งใด?

---

## Step 943 — Ticket 1: เฉลยและคำอธิบาย Root Cause โดยละเอียด

### 943.1 กระบวนการอ่าน Exhibit ตาม Methodology ของ Part 91 (แบบไม่มี Live Access)

เนื่องจากไม่มีสิทธิ์รันคำสั่งเพิ่ม ต้องใช้หลักการ [Part 91 Step 905](part-091-advanced-troubleshooting-methodology.md)
แปลง Exhibit A ให้เป็นสมมติฐานที่ทดสอบได้จาก Exhibit ที่เหลือทันที:　**"Prefix เก่าใช้ได้ Prefix ใหม่
ใช้ไม่ได้ ผ่าน Session เดียวกัน"** — คำถามที่ต้องตอบคือ **"ทำไม Session เดียวกันถึงเลือกปฏิบัติ
ระหว่าง Prefix ได้"**

| ข้อสังเกตจาก Exhibit | สิ่งที่ตัดออกได้ทันที |
|---|---|
| Exhibit B: Session กับ 198.51.100.14 (AS 65010) มี State เป็นตัวเลข (`PfxRcd = 2`) ไม่ใช่ `Idle`/`Active` | **ไม่ใช่ปัญหา BGP Neighbor/Session** (Session Established ปกติ) — ตัด TCP/Peering ทิ้งได้ทั้งหมด |
| Exhibit E: ฝั่ง Partner ยืนยันด้วย `advertised-routes` ว่าส่งครบ 3 Prefix แล้ว | ปัญหาไม่ได้อยู่ฝั่ง Partner — ตัดสาเหตุจากฝั่งเขาทิ้งได้ |
| Exhibit D: `show ip bgp` ที่ `WAN-EDGE-1` เห็นแค่ 2 Prefix (`.50.0/24`, `.51.0/24`) สอดคล้องกับ `PfxRcd = 2` ใน Exhibit B | **`172.20.60.0/24` ไม่ถูกรับเข้ามาที่ `WAN-EDGE-1` เลย** ทั้งที่ Partner ยืนยันว่าส่งแล้ว — ต้องมี Filter ขาเข้าบล็อกอยู่ |
| Exhibit C: มี `neighbor ... prefix-list PARTNER-IN in` | นี่คือ Filter ขาเข้าที่ต้องสงสัยตาม Evidence ข้อก่อนหน้า |

### 943.2 วิเคราะห์ Prefix-list `PARTNER-IN` จาก Exhibit C

```
ip prefix-list PARTNER-IN seq 5 permit 172.20.50.0/24
ip prefix-list PARTNER-IN seq 10 permit 172.20.51.0/24
```

Prefix-list นี้เขียนแบบ **Explicit Match ทีละ Prefix** (ไม่มี `ge`/`le` ไม่มี Supernet ครอบคลุม) —
มีเพียง 2 Sequence ที่ Permit `172.20.50.0/24` และ `172.20.51.0/24` เท่านั้น **ไม่มี Sequence ใด
ที่ครอบคลุม `172.20.60.0/24` เลย** ดังนั้น Prefix ใหม่จึงตกไปโดน **Implicit Deny ท้าย Prefix-list**
โดยอัตโนมัติ (หลักการ Implicit Deny เดียวกับที่ [Part 14 Step 142](part-014-access-control-lists.md)
สอนไว้สำหรับ ACL และ [Part 29 Step 291](part-029-bgp-fundamentals.md) ยืนยันว่าใช้กับ Prefix-list
เหมือนกัน)

### 943.3 Root Cause

**Prefix-list `PARTNER-IN` เขียนแบบ Explicit เจาะจงทีละ Subnet โดยไม่มี Supernet/Wildcard
ครอบคลุมล่วงหน้า** — ทุกครั้งที่ทีม Partner เพิ่ม Subnet ใหม่เข้ามาในการเชื่อมต่อเดิม ทีมเราต้องแก้
Prefix-list ฝั่งเราด้วยเสมอ (Manual Process ที่ไม่ได้ Sync กับฝั่ง Partner อัตโนมัติ) — ครั้งนี้ทีม
Partner Integration แจ้งเปลี่ยนแปลงแต่ไม่มีใครอัปเดต Prefix-list ให้ตรงกัน เป็น **Process Gap**
ไม่ใช่ Bug ทางเทคนิคของ Router เลย (Router ทำงาน "ตามที่ Config บอก" อย่างถูกต้อง 100%)

### 943.4 ทำไมนี่ไม่ใช่ปัญหาแบบ Part 69 Step 683 (`ge 1` ตัด `/0`)

| จุดเทียบ | Part 69 Step 683 | Ticket นี้ |
|---|---|---|
| ลักษณะ Bug | `permit 0.0.0.0/0 ge 1` เขียนถูก Syntax แต่ตัด Prefix ต้นฉบับออกไปโดยไม่ตั้งใจ | ไม่มี `ge`/`le` เลย — Explicit List ที่ไม่ได้อัปเดตตามการเปลี่ยนแปลงจริง |
| ประเภทปัญหา | ความเข้าใจผิด Syntax ของ `ge`/`le` | Configuration Drift จาก Process การเปลี่ยนแปลงที่ไม่ Sync กัน |
| Prefix ที่กระทบ | Default Route ตัวเดียว (`/0`) | Prefix ใหม่ที่เพิ่มมาล่าสุดเท่านั้น (Prefix เก่ายังผ่านปกติ) |

### 943.5 การแก้ไขที่ถูกต้อง (คำตอบที่ต้องพิมพ์ในห้องสอบ)

```
WAN-EDGE-1(config)# ip prefix-list PARTNER-IN seq 15 permit 172.20.60.0/24
WAN-EDGE-1(config)# end
WAN-EDGE-1# clear ip bgp 198.51.100.14 soft in
```

> **ทางเลือกเชิง Design ที่ดีกว่าในระยะยาว** (คำตอบเสริมที่ได้คะแนนพิเศษถ้าข้อสอบถามแนวทางป้องกัน):
> เปลี่ยนเป็น Supernet ที่ครอบคลุม Range ที่ทีม Partner ตกลงจะใช้ล่วงหน้าทั้งหมด เช่น
> `ip prefix-list PARTNER-IN seq 5 permit 172.20.48.0/20 ge 24 le 24` (Permit ทุก /24 ภายใน
> `172.20.48.0/20` ที่ Partner อาจประกาศเพิ่มในอนาคต) — ลด Manual Process ที่เป็นต้นเหตุของ Ticket
> นี้ลงได้ทั้งหมด แต่ต้องแลกกับความเสี่ยงด้าน Security ที่ต้องพิจารณาคู่กับทีม Security (เชื่อมกับ
> หลักการ Zero Trust "Never Trust, Always Verify" ของ [Part 94](part-094-zero-trust-network-architecture.md)
> — Supernet กว้างเกินไปอาจ Permit Prefix ที่ Partner ไม่ได้รับอนุญาตจริงเข้ามาด้วย)

### 943.6 Verify (แสดงเป็น Exhibit สมมติหลังแก้ไข เพื่อยืนยันแนวคิด)

```
WAN-EDGE-1# show ip bgp | include 65010
*> 172.20.50.0/24    198.51.100.14            0             0 65010 i
*> 172.20.51.0/24    198.51.100.14            0             0 65010 i
*> 172.20.60.0/24    198.51.100.14            0             0 65010 i

WAN-EDGE-1# show ip bgp summary | include 65010
198.51.100.14   4 65010    2215    2203     3128    0    0 2w1d           3
```

`PfxRcd` เปลี่ยนจาก 2 เป็น 3 ตรงกับที่ Exhibit E ยืนยันว่า Partner ส่งมา 3 Prefix — Root Cause
พิสูจน์ถูกต้องสมบูรณ์

---

## Step 944 — Ticket 2: หลักฐานชุดที่ 2 (Topology Diagram + Partial Config)

> Ticket นี้ใช้รูปแบบ Exhibit ต่างจาก Ticket 1 — มาพร้อม **Topology Diagram** แทนคำร้องเรียนอย่าง
> เดียว ตามที่ข้อสอบจริงมักทำเมื่อปัญหาเกี่ยวข้องกับโครงสร้าง Network มากกว่า Session เดี่ยว

### Ticket #DIAG-9502

**Exhibit A — คำร้องเรียนจากลูกค้า**

> "แผนก Operations ที่ Building D (ตึกใหม่ที่เพิ่งเปิดใช้งานสัปดาห์นี้) แจ้งว่าเข้าใช้งาน File
> Server ในตึกเดียวกันได้ปกติ (เครื่องอยู่ VLAN เดียวกัน) แต่เข้า Intranet ระบบอื่นทั้งหมด (Server
> ที่ Building C, และ Internet) ไม่ได้เลยตั้งแต่เปิดใช้งานตึกวันแรก ทีม Implementation ยืนยันว่า
> Cable และ Physical Layer ทุกจุดตรวจสอบแล้วไม่มีปัญหา (Interface ทุกจุด `up/up`)"

**Exhibit B — Topology Diagram ของการเชื่อมต่อ Building D (ส่วนที่ทีม Implementation เพิ่มใหม่)**

```
                              ┌───────────────────┐
                              │   CORE-SW1          │
                              │  (Area 0 / Area 1)  │
                              └─────────┬───────────┘
                                        │ TenGigabitEthernet1/0/9
                                        │ (สร้างใหม่สำหรับ Building D)
                                        │ Area 3 (New Area — Building D)
                              ┌─────────┴───────────┐
                              │   DIST-SW5           │  <- อุปกรณ์ใหม่ (Building D)
                              │  (Area 3)            │
                              └─────────┬───────────┘
                                        │ VLAN 50 (Building D Operations)
                              ┌─────────┴───────────┐
                              │  ACCESS-SW5           │
                              │  10.10.50.0/24        │
                              └───────────────────────┘
```

**Exhibit C — `show ip ospf neighbor` (เก็บจาก `DIST-SW5`)**

```
DIST-SW5# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   EXSTART/  -     00:00:38    10.255.90.1     TenGigabitEthernet1/1/1
```

**Exhibit D — Running-config Excerpt (`CORE-SW1`, ส่วน Interface ที่เชื่อม `DIST-SW5`)**

```
interface TenGigabitEthernet1/0/9
 description Link to DIST-SW5 (Building D)
 no switchport
 ip address 10.255.90.1 255.255.255.252
 ip ospf 1 area 3
 mtu 9216
```

**Exhibit E — Running-config Excerpt (`DIST-SW5`, ส่วน Interface ที่เชื่อม `CORE-SW1`)**

```
interface TenGigabitEthernet1/1/1
 description Link to CORE-SW1
 no switchport
 ip address 10.255.90.2 255.255.255.252
 ip ospf 1 area 3
```

**Exhibit F — `show interfaces TenGigabitEthernet1/1/1` (เก็บจาก `DIST-SW5`, บางส่วน)**

```
DIST-SW5# show interfaces TenGigabitEthernet1/1/1
TenGigabitEthernet1/1/1 is up, line protocol is up
  MTU 1500 bytes, BW 10000000 Kbit/sec, DLY 10 usec
```

**คำถาม**: จาก Exhibit ทั้งหมดข้างต้น Root Cause ของปัญหาคืออะไร และควรแก้ไขด้วยคำสั่งใด?

---

## Step 945 — Ticket 2: เฉลยและคำอธิบาย Root Cause โดยละเอียด

### 945.1 ใช้ Pattern-Matching จาก Part 91 Step 901 ก่อนอ่านรายละเอียด

Exhibit A บอกว่า **"เครื่องในตึกเดียวกันคุยกันได้ แต่ข้าม Segment ไม่ได้เลย"** และ **Interface
`up/up` ทุกจุด** — ตาม [Part 91 Step 901.2](part-091-advanced-troubleshooting-methodology.md)
ตารางที่ 4 ("Interface ทั้งคู่ `up/up` แต่ Traffic ไม่วิ่งเลย") ชี้ตรงไปที่ **Layer 3 ขึ้นไป**
ทันที — Physical/Data Link ผ่านแล้วแน่นอน ไม่ต้องเสียเวลาสงสัยสายหรือ Duplex อีก

### 945.2 อ่าน Exhibit C — Neighbor State ค้างที่ `EXSTART`

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   EXSTART/  -     00:00:38    10.255.90.1     TenGigabitEthernet1/1/1
```

**`EXSTART`** คือ State ที่ OSPF Router สองตัวกำลังตกลง **Master/Slave** และแลกเปลี่ยน DBD
(Database Description) Packet แรก — ถ้า Neighbor **ค้างอยู่ State นี้ซ้ำๆ ไม่ขยับไป `EXCHANGE`
หรือ `FULL`** นี่คือ Signature คลาสสิกของ **MTU Mismatch** ระหว่าง 2 ฝั่ง (สอนไว้ตั้งแต่
[Part 57 Step 561](part-057-ospf-troubleshooting.md)) — Cisco IOS ตรวจสอบค่า MTU ที่ Neighbor
ประกาศมาใน DBD เทียบกับ MTU ของ Interface ตัวเอง ถ้าไม่ตรงกันและตัวเองมี MTU เล็กกว่าที่ Neighbor
อ้าง จะปฏิเสธไม่ยอมขยับ State ต่อ (ป้องกันไม่ให้ LSA/DBD Packet ใหญ่เกินที่ Interface ตัวเองส่งได้)

### 945.3 ยืนยันด้วย Exhibit D, E, F — เทียบ MTU ทั้ง 2 ฝั่ง

| อุปกรณ์ | MTU ที่ Config (Exhibit D/E) | MTU จริงที่ Interface (Exhibit F) |
|---|---|---|
| `CORE-SW1` (Te1/0/9) | `mtu 9216` (ตั้ง Jumbo Frame ไว้ชัดเจน) | (ไม่มี Exhibit ให้ตรงๆ แต่ Config ยืนยัน 9216) |
| `DIST-SW5` (Te1/1/1) | ไม่มีคำสั่ง `mtu` ใดๆ (ใช้ Default) | `MTU 1500 bytes` (Exhibit F ยืนยันตรงตัว) |

**สรุปตรงเผง**: `CORE-SW1` ตั้ง Jumbo Frame (`mtu 9216`) ไว้ที่ Interface นี้ตามมาตรฐาน Uplink
Backbone ของทีม Data Center (เข้าใจได้ว่าต้องการ Throughput สูงสุดสำหรับ Backbone จริง) แต่ทีม
Implementation ที่ Deploy `DIST-SW5` ใหม่ **ไม่ได้ตั้ง MTU ให้ตรงกัน** ปล่อยไว้ที่ Default 1500 —
เมื่อ `DIST-SW5` ได้รับ DBD จาก `CORE-SW1` ที่ประกาศ MTU 9216 (ใหญ่กว่า MTU ของตัวเอง) จึงปฏิเสธ
ไม่ให้ Adjacency ขยับไปกว่า `EXSTART` ตามพฤติกรรม Default ของ IOS

### 945.4 ทำไม "เครื่องในตึกเดียวกันใช้งานได้ปกติ" — ตัดข้อสงสัยอื่นได้อย่างไร

Traffic ภายใน VLAN 50 (Building D) ทั้งหมดเป็น **Layer 2 Switching ล้วนๆ ภายใน `ACCESS-SW5` และ
`DIST-SW5`** ไม่เกี่ยวกับ OSPF เลย จึงไม่ได้รับผลกระทบจาก Adjacency ที่ `EXSTART` — แต่ Traffic
ที่ต้องออกไปยัง Building C หรือ Internet ต้องพึ่ง **Routing Table ที่ได้จาก OSPF** (Path ไป
`CORE-SW1`) ซึ่งไม่มีเลยเพราะ Adjacency ไม่เคยขึ้นเป็น `FULL` — จึงไม่มี Route ใดๆ นอกจาก Local
Subnet บน `DIST-SW5` ตรงกับอาการที่รายงานมาทุกประการ (Exhibit A อธิบายได้ครบ 100% ตามกฎข้อ 2 ของ
Step 941.4)

### 945.5 Root Cause

**MTU Mismatch ระหว่าง `CORE-SW1` (9216 Byte, Jumbo Frame) และ `DIST-SW5` (1500 Byte, Default)**
บน Link ที่เพิ่ง Deploy ใหม่สำหรับ Building D — ทำให้ OSPF Adjacency ค้างอยู่ที่ State `EXSTART`
ตลอดไป ไม่มีทาง Full Adjacency เกิดขึ้นได้เลยจนกว่าจะแก้ MTU ให้ตรงกัน

### 945.6 การแก้ไข (2 วิธีที่เป็นไปได้ — ต้องเลือกให้ตรงกับ Requirement)

```
! วิธีที่ 1 (แนะนำ ถ้า Requirement ต้องการ Jumbo Frame ทั้ง Backbone จริง)
DIST-SW5(config)# interface TenGigabitEthernet1/1/1
DIST-SW5(config-if)# mtu 9216
DIST-SW5(config-if)# exit

! วิธีที่ 2 (ถ้าไม่จำเป็นต้องใช้ Jumbo Frame ที่ Link นี้ หรือใช้แก้ปัญหาเร็วโดยไม่กระทบ MTU จริง)
CORE-SW1(config)# interface TenGigabitEthernet1/0/9
CORE-SW1(config-if)# ip ospf mtu-ignore
DIST-SW5(config)# interface TenGigabitEthernet1/1/1
DIST-SW5(config-if)# ip ospf mtu-ignore
```

> **ข้อควรระวังสำหรับคำตอบในห้องสอบ**: `ip ospf mtu-ignore` เป็นการ **"บอก OSPF ให้เลิกตรวจ MTU"**
> ไม่ใช่การแก้ MTU จริง — ถ้า MTU จริงยังไม่ตรงกัน (เช่น 9216 vs 1500) และมีการส่ง Packet ขนาดใหญ่
> จริงๆ ผ่าน Link นี้ (ไม่ใช่แค่ OSPF Hello/DBD) จะเกิด Fragmentation หรือ Drop ตามหลักการ PMTUD ที่
> [Part 51 Step 505.3](part-051-vrf-lite-gre-ipsec.md) และ [Part 91 Step 910](part-091-advanced-troubleshooting-methodology.md)
> เตือนไว้แล้ว — **วิธีที่ 1 (ปรับ MTU ให้ตรงกันจริง) คือคำตอบที่ปลอดภัยกว่าเสมอ** เว้นแต่ข้อสอบระบุ
> ข้อจำกัดที่ทำให้ต้องเลือกวิธีที่ 2 (เช่น Hardware บางรุ่นของ `DIST-SW5` ไม่รองรับ Jumbo Frame)

### 945.7 Verify

```
DIST-SW5# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/  -        00:00:33    10.255.90.1     TenGigabitEthernet1/1/1

DIST-SW5# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 1", distance 110, metric 2, candidate default path
```

---

## Step 946 — Ticket 3: หลักฐานชุดที่ 3

### Ticket #DIAG-9503

**Exhibit A — คำร้องเรียนจากลูกค้า**

> "ทีม Learning & Development จัดอบรม Video Training แบบ Live Broadcast ผ่าน Multicast (กลุ่ม
> `239.192.10.50`) จาก Server ที่ `Server2` (VLAN 30, ผ่าน `ACCESS-SW3`) ไปยังผู้ชมทั่วบริษัท พบว่า
> **ผู้ชมที่ต่อสาย LAN (VLAN 30/40 แบบ Wired) เห็นภาพและเสียงชัดเจนสมบูรณ์ทุกคน** แต่ **ผู้ชมที่ต่อ
> Wi-Fi ผ่าน `WLC-1`/`AP-2` ทุกคนไม่เห็นวิดีโอเลย** (จอค้างที่ Loading ตลอด ไม่มี Error Message ใดๆ)
> เหตุการณ์นี้เพิ่งเกิดครั้งแรก เพราะเป็นการจัดอบรมแบบ Multicast ครั้งแรกของบริษัท — ไม่มี Ticket
> อื่นเกี่ยวกับ Wi-Fi ทั่วไป (เช็ก Email/Browse เว็บผ่าน Wi-Fi ตามปกติได้หมด)"

**Exhibit B — `show ip mroute 239.192.10.50` (เก็บจาก `DIST-SW3`)**

```
DIST-SW3# show ip mroute 239.192.10.50
(*, 239.192.10.50), 00:14:22/00:02:58, RP 10.10.30.1, flags: S
  Incoming interface: Vlan30, RPF nbr 10.10.30.1
  Outgoing interface list:
    Vlan40, Forward/Sparse, 00:14:20/00:02:40
    Vlan30, Forward/Sparse, 00:12:05/00:03:11
```

**Exhibit C — `show ip igmp snooping querier vlan 40` (เก็บจาก `DIST-SW4`)**

```
DIST-SW4# show ip igmp snooping querier vlan 40
Vlan  IP Address       IGMP Version   Port
----  --------------   ------------   -----
40    10.10.40.1       v2             Router
```

**Exhibit D — Running-config Excerpt (`DIST-SW4`, Policy-map ที่ Apply บน Trunk ไปยัง `WLC-1`)**

```
interface TenGigabitEthernet1/1/9
 description Uplink to WLC-1
 switchport mode trunk
 switchport trunk allowed vlan 20,40,99
 service-policy output WIRELESS-HARDENING-OUT
!
class-map match-any MULTICAST-FLOOD
 match access-group name MCAST-RANGE
!
policy-map WIRELESS-HARDENING-OUT
 class MULTICAST-FLOOD
  police cir 32000 bc 1500
   conform-action transmit
   exceed-action drop
!
ip access-list extended MCAST-RANGE
 permit ip any 224.0.0.0 15.255.255.255
```

**Exhibit E — บันทึก Change Ticket (อ้างอิงประกอบ)**

> "CHG-90210 (ทีม Wireless Security, ดำเนินการเมื่อ 3 สัปดาห์ก่อน): เพิ่ม Policy `police cir 32000`
> บน Uplink ของ `WLC-1` ทุกตัว เพื่อจำกัด Multicast/Broadcast Storm จาก mDNS/SSDP ของอุปกรณ์ IoT ที่
> เพิ่มจำนวนมากผิดปกติในไตรมาสนี้ — ยังไม่มีรายงานปัญหาใดๆ จนถึงก่อน Ticket นี้"

**คำถาม**: จาก Exhibit ทั้งหมดข้างต้น Root Cause ของปัญหาคืออะไร และควรแก้ไขด้วยคำสั่งใด?

---

## Step 947 — Ticket 3: เฉลยและคำอธิบาย Root Cause โดยละเอียด

### 947.1 Multi-Domain Triage ตาม Part 91 Step 907 — แยก "ปัญหาอยู่ที่ไหน" ก่อน

อาการ "Wired ได้ Wireless ไม่ได้ ทั้งที่เป็น Multicast Group เดียวกัน" ต้องแยกให้ชัดว่าจุดที่แตกต่าง
ระหว่าง Path ทั้งสองอยู่ตรงไหน — ใช้หลัก Bisection ของ [Part 91 Step 902](part-091-advanced-troubleshooting-methodology.md)
วาง Path ทั้งสองเทียบกัน:

```
Server2 → ACCESS-SW3 → DIST-SW3 ─┬→ DIST-SW3 (Vlan30, Wired) → ACCESS-SW3 → Wired Client (สำเร็จ)
                                  └→ DIST-SW4 (Vlan40, OIL) → Trunk to WLC-1 → AP-2 → Wireless Client (ล้มเหลว)
```

Exhibit B ยืนยันว่า **Multicast Routing (PIM/mroute) ทำงานถูกต้องสมบูรณ์แล้ว**: `Vlan40` ปรากฏใน
**Outgoing Interface List (OIL)** ของ `(*, 239.192.10.50)` พร้อม Flag `Forward/Sparse` — แปลว่า
`DIST-SW3`/`DIST-SW4` **ได้ Forward Traffic ไปทาง VLAN 40 แล้วจริง** ตัดปัญหาระดับ PIM/RP/Multicast
Routing ทิ้งได้ทั้งหมด (ไม่ต้องเสียเวลาไปดู RP Mapping, PIM Neighbor หรือ MSDP เลยแม้แต่คำสั่งเดียว)

Exhibit C ยืนยันว่า **IGMP Snooping Querier บน VLAN 40 ทำงานปกติ** (มี Querier ที่ `10.10.40.1`,
Version 2) — ตัดปัญหาการที่ Client "Join Group ไม่สำเร็จเพราะไม่มี Querier" ทิ้งได้ (ปัญหาคลาสสิก
ที่ [Part 65 Step 645](part-065-multicast-troubleshooting.md) สอนไว้ — แต่ในเคสนี้ Querier มีอยู่
จริงจึงไม่ใช่ตัวการ)

### 947.2 ตัดทิ้งได้ว่าอะไรไม่ใช่ Root Cause — เหลืออะไรที่ยังไม่ถูกตรวจ

| สิ่งที่ Exhibit ยืนยันแล้วว่าปกติ | สิ่งที่ยังไม่มี Exhibit ยืนยัน |
|---|---|
| PIM/Multicast Routing ถึง VLAN 40 (Exhibit B) | สิ่งที่เกิดขึ้น **ระหว่าง** `DIST-SW4` กับ `WLC-1` บน Trunk เดียวกัน |
| IGMP Snooping Querier บน VLAN 40 (Exhibit C) | ปริมาณ Traffic จริงที่ Multicast Video Stream ต้องการ (Bandwidth) |

Traffic ที่ Forward ออกจาก `DIST-SW4` ไปทาง VLAN 40 ต้องเดินทางผ่าน **Trunk เดียวกันที่เชื่อมไปยัง
`WLC-1`** (Exhibit D) — จุดนี้เป็นจุดเดียวที่ยังไม่ถูกพิสูจน์ว่าปกติ และ Exhibit D คือหลักฐานที่ให้
มาสำหรับจุดนี้พอดี

### 947.3 วิเคราะห์ Policy-map `WIRELESS-HARDENING-OUT` จาก Exhibit D

```
ip access-list extended MCAST-RANGE
 permit ip any 224.0.0.0 15.255.255.255      ! ครอบคลุม 224.0.0.0/4 ทั้งหมด (ทุก Multicast Group)

policy-map WIRELESS-HARDENING-OUT
 class MULTICAST-FLOOD
  police cir 32000 bc 1500                    ! จำกัดที่ 32 Kbps เท่านั้น
   conform-action transmit
   exceed-action drop
```

`224.0.0.0 15.255.255.255` (Wildcard Mask) ครอบคลุม **ทุก Address ในช่วง `224.0.0.0/4`** ซึ่งคือ
**Multicast Address Space ทั้งหมด** ไม่ได้เจาะจงเฉพาะ mDNS/SSDP (ที่มักใช้ `224.0.0.251` และ
`239.255.255.250` เท่านั้น) ตามเจตนาเดิมของ CHG-90210 — Policer ที่ `police cir 32000` (32 Kbps)
จึงจำกัด **Video Training Stream ที่ต้องใช้ Bandwidth หลัก Mbps** ให้เหลือแค่ 32 Kbps ด้วย ทำให้
Frame ส่วนใหญ่ถูก `exceed-action drop` ทิ้งเงียบๆ — Client เห็นแค่ Buffer ว่างและ Loading ค้างตลอด
โดยไม่มี Error Message ใดๆ ให้เห็นเลย (ตรงกับ Exhibit A ทุกประการ)

### 947.4 Root Cause

**Policy-map `WIRELESS-HARDENING-OUT` ที่ Push เข้ามาโดย CHG-90210 เขียน ACL `MCAST-RANGE` กว้าง
เกินเจตนาเดิม** (ครอบคลุมทั้ง `224.0.0.0/4` ทั้งช่วง ไม่ได้เจาะจงเฉพาะ mDNS/SSDP ตามที่ตั้งใจ) ทำให้
Policer ที่ตั้งไว้สำหรับกันแค่ Broadcast Storm ของ IoT (32 Kbps เพียงพอสำหรับ mDNS จริง) ไปจำกัด
**Legitimate Multicast Video Traffic** ด้วยโดยไม่ตั้งใจ — เป็น Cross-Domain Bug ระหว่าง **QoS
(Policer)** และ **Multicast (Traffic ที่ถูกจำกัด)** ที่ไม่แสดงอาการจนกว่าจะมีคนใช้ Multicast แบบ
Bandwidth สูงจริงเป็นครั้งแรก (อธิบายได้ว่าทำไม "ไม่มีรายงานปัญหาจนถึงก่อน Ticket นี้" ตาม Exhibit E)

### 947.5 การแก้ไข

```
DIST-SW4(config)# ip access-list extended MCAST-RANGE
DIST-SW4(config-ext-nacl)# no permit ip any 224.0.0.0 15.255.255.255
DIST-SW4(config-ext-nacl)# permit ip any host 224.0.0.251        ! mDNS
DIST-SW4(config-ext-nacl)# permit ip any host 239.255.255.250     ! SSDP
DIST-SW4(config-ext-nacl)# exit
```

> **หมายเหตุ**: แก้ที่ ACL ให้เจาะจงตาม Intent เดิม (mDNS/SSDP เท่านั้น) คือคำตอบที่ตรงกับสิ่งที่
> CHG-90210 ตั้งใจจะทำจริง ๆ — **ไม่ใช่การถอด Policy ทั้งหมดออก** (การถอดทั้งหมดจะทำให้ปัญหา IoT
> Broadcast Storm เดิมที่ CHG-90210 แก้ไว้กลับมาใหม่) หลักการเดียวกับ Defense-in-Depth ที่
> [Part 91 Step 910.11](part-091-advanced-troubleshooting-methodology.md) ใช้ตอนแก้ ACL ที่ Deny
> ICMP ทั้งหมดแทนที่จะลบ ACL ทิ้งทั้งเส้น

### 947.6 Verify

```
DIST-SW4# show ip access-lists MCAST-RANGE
Extended IP access list MCAST-RANGE
    10 permit ip any host 224.0.0.251
    20 permit ip any host 239.255.255.250

DIST-SW4# show policy-map interface TenGigabitEthernet1/1/9 | include class|conform|drop
  Class-map: MULTICAST-FLOOD (match-any)
    32000 bps, drop rate 0 bps      ! Video Stream (239.192.10.50) ไม่ Match Class นี้แล้ว จึงไม่ถูก Police
```

---

## Step 948 — Ticket 4: หลักฐานชุดที่ 4 (Cross-Domain)

> Ticket นี้จงใจผสมโดเมน **Security (ACL/Management Plane)** กับ **Automation (CI/CD Pipeline)**
> เข้าด้วยกัน — เป็นรูปแบบที่พบได้น้อยกว่าในสื่อฝึกซ้อมทั่วไป (ซึ่งมักแยก Security กับ Automation
> เป็นโดเมนละบทเรียนไม่ปนกัน) แต่เป็นสิ่งที่เกิดขึ้นจริงบ่อยมากในองค์กรที่ใช้ Config Automation

### Ticket #DIAG-9504

**Exhibit A — คำร้องเรียนจากลูกค้า (ทีม NOC)**

> "ทีม NOC พบว่า SSH เข้า `BRANCH3-RTR` ล้มเหลว (Connection Refused/Timeout) **เป็นช่วงสั้นๆ ทุกคืน
> ประมาณ 02:00-02:02 น.** ติดกันมา 4 คืนแล้ว หลังจากผ่านช่วงเวลานั้นไปแล้ว SSH กลับมาใช้งานได้ปกติ
> ทุกครั้งโดยไม่ต้องทำอะไรเลย ไม่พบอาการนี้ที่ Router สาขาอื่นเลยแม้จะอยู่ภายใต้ Config Template
> เดียวกันทั้งหมด `BRANCH3-RTR` เป็นสาขาที่ไกลที่สุดและใช้ WAN Link ที่มี Latency สูงสุดในบริษัท"

**Exhibit B — Log จาก Ansible AWX (Job: `compliance_push.yml`, Run เมื่อคืนล่าสุด)**

```
02:00:03 PLAY [Push MGMT-ACCESS compliance baseline] ***
02:00:04 TASK [Gather running-config via NETCONF] ***
02:00:05 ok: [BRANCH1-RTR]
02:00:05 ok: [BRANCH2-RTR]
02:00:06 ok: [BRANCH3-RTR]
02:00:07 TASK [Detect drift vs golden template] ***
02:00:08 changed: [BRANCH3-RTR] => ACL MGMT-ACCESS drift detected, remediation required
02:00:09 TASK [Remediate ACL: delete and recreate] ***
02:00:10 changed: [BRANCH3-RTR] => RPC edit-config sent (delete ip access-list extended MGMT-ACCESS)
02:01:34 changed: [BRANCH3-RTR] => RPC edit-config sent (recreate ip access-list extended MGMT-ACCESS, 6 ACE)
02:01:35 ok: [BRANCH1-RTR] (no drift — no action required, completed 02:00:09)
02:01:36 ok: [BRANCH2-RTR] (no drift — no action required, completed 02:00:11)
```

**Exhibit C — Running-config Excerpt (`BRANCH3-RTR`, เก็บช่วงกลางวัน — ดูปกติดี)**

```
ip access-list extended MGMT-ACCESS
 permit tcp 10.10.99.0 0.0.0.255 any eq 22
 permit tcp host 10.10.99.50 any eq 22
 deny   ip any any log
!
line vty 0 4
 access-class MGMT-ACCESS in
 transport input ssh
```

**Exhibit D — บันทึกทีม Automation (ประกอบ Playbook `compliance_push.yml`)**

> "ขั้นตอน Remediation ของ Task 'Delete and Recreate' คือ: (1) ส่ง RPC ลบ ACL ทั้งเส้นก่อนเสมอ
> เพื่อไม่ให้ Sequence Number ตกค้างจาก Manual Edit ที่ทำผ่าน CLI ปนอยู่ (2) ส่ง RPC สร้าง ACE ใหม่
> ทีละบรรทัดตาม Golden Template จนครบ — ออกแบบมาให้ Idempotent ทำงานเหมือนกันทุก Router ไม่ว่า
> Router เดิมจะมี Drift แบบใดมาก่อน"

**Exhibit E — ผลเปรียบเทียบ Latency ของ WAN Link แต่ละสาขา (จาก Network Assurance Dashboard)**

| สาขา | RTT เฉลี่ยไปยัง Ansible Control Node |
|---|---|
| BRANCH1-RTR | 8 ms |
| BRANCH2-RTR | 12 ms |
| BRANCH3-RTR | 210 ms |

**คำถาม**: จาก Exhibit ทั้งหมดข้างต้น Root Cause ของปัญหาคืออะไร และควรแก้ไขด้วยคำสั่ง/แนวทางใด?

---

## Step 949 — Ticket 4: เฉลยและคำอธิบาย Root Cause โดยละเอียด

### 949.1 ใช้ Baseline Comparison (Part 91 Step 904) — เวลาตรงกันเป๊ะคือกุญแจแรก

Exhibit A ระบุว่าปัญหาเกิด **"02:00-02:02 น. ทุกคืน"** และ Exhibit B ยืนยันว่า Ansible Job
`compliance_push.yml` รันเวลา **02:00:03** พอดี — ตรงกับหลักการ [Part 91 Step 904.5](part-091-advanced-troubleshooting-methodology.md)
ข้อ 1: **"ระบบนี้เพิ่งเปลี่ยนแปลงอะไรหรือไม่"** — Correlation ทางเวลาที่ตรงกันขนาดนี้ (นาทีต่อนาที)
เป็น Hypothesis แรกที่แข็งแรงที่สุดทันที ไม่ต้องเดา

### 949.2 อ่าน Exhibit B อย่างละเอียด — จุดที่ต่างจาก Router อื่น

```
02:00:09 changed: [BRANCH3-RTR] => ACL MGMT-ACCESS drift detected, remediation required
02:00:10 changed: [BRANCH3-RTR] => RPC edit-config sent (delete ip access-list extended MGMT-ACCESS)
02:01:34 changed: [BRANCH3-RTR] => RPC edit-config sent (recreate ip access-list extended MGMT-ACCESS, 6 ACE)
```

**ระหว่าง `02:00:10` (สั่งลบ ACL) ถึง `02:01:34` (สร้าง ACE ครบ) คือ 1 นาที 24 วินาที** — ในช่วงนี้
ACL ชื่อ `MGMT-ACCESS` **มีอยู่จริงแต่ไม่มี ACE เลยแม้แต่บรรทัดเดียว** (อยู่ระหว่างการสร้างใหม่ทีละ
บรรทัดผ่าน RPC หลาย Transaction) — พฤติกรรม Cisco IOS ต่อ ACL ที่ตั้งชื่อไว้แต่ **ไม่มี ACE เลย**
คือ **Implicit Deny All** (เทียบเท่ากับ ACL ที่มีแต่ Deny ท้ายบรรทัดเดียว) เมื่อ ACL นี้ถูกอ้างอิงอยู่
กับ `access-class ... in` บน VTY (Exhibit C) การเชื่อมต่อ SSH ใดๆ ที่พยายามเข้ามา **ระหว่างหน้าต่าง
1 นาที 24 วินาทีนี้พอดี** จะถูก Deny ทั้งหมด

Router อื่น (`BRANCH1-RTR`, `BRANCH2-RTR`) **ไม่มี Drift** (`no action required`) จึงไม่เคยเข้าสู่
Cycle "ลบแล้วสร้างใหม่" นี้เลย — ไม่ใช่ว่า Playbook ปลอดภัยกว่าที่สาขาอื่น แต่เป็นเพราะสาขาอื่น
Config ตรงกับ Golden Template อยู่แล้วจึงไม่ถูก Remediate

### 949.3 ทำไมหน้าต่างของ `BRANCH3-RTR` ยาวถึง 1 นาที 24 วินาที (Exhibit E คือคำตอบ)

Exhibit E ยืนยันว่า `BRANCH3-RTR` มี RTT ไปยัง Ansible Control Node สูงถึง **210 ms** เทียบกับ
สาขาอื่นที่ 8-12 ms — RPC ผ่าน NETCONF แต่ละ Transaction (สร้าง ACE ทีละบรรทัด รวม 6 บรรทัด) ต้องรอ
Round-Trip สำหรับแต่ละ Transaction จึงสะสม Latency มากกว่าสาขาอื่นหลายสิบเท่า ทำให้หน้าต่างที่ ACL
ว่างเปล่ายาวนานพอที่จะ **ชนกับ SSH Connection Attempt จริงของทีม NOC พอดีในบางคืน** — ที่สาขาอื่น
หน้าต่างนี้สั้นกว่ามาก (คาดว่าไม่ถึง 1 วินาที) จึงไม่มีใครสังเกตเห็นเลยแม้จะมี Bug เดียวกันแฝงอยู่
ในทุกสาขาก็ตาม

### 949.4 Root Cause

**Playbook `compliance_push.yml` ใช้รูปแบบ "Delete-then-Recreate" กับ ACL ที่ยังถูกอ้างอิงใช้งานจริง
อยู่ (Live-bound ACL บน `access-class` ของ VTY)** — ระหว่างการลบและสร้างใหม่ทีละ Transaction ผ่าน
NETCONF RPC จะมีหน้าต่างเวลาที่ ACL ไม่มี ACE เลย (Implicit Deny All) ซึ่งที่สาขาที่มี Latency สูง
เช่น `BRANCH3-RTR` หน้าต่างนี้ยาวพอที่จะปฏิเสธ SSH Connection จริงที่พยายามเข้ามาในเวลาเดียวกันโดย
บังเอิญ — เป็น **Design Flaw ของ Automation Pipeline** ที่ไม่ Atomic ไม่ใช่ปัญหาที่ ACL/Config
Content เองผิด (Exhibit C ยืนยันว่า Config สุดท้ายถูกต้องสมบูรณ์ทุกครั้ง)

### 949.5 การแก้ไข (ระดับ Pipeline ไม่ใช่ระดับ Device เดียว)

```yaml
# แนวทางที่ถูกต้อง — แก้ ACE แบบ Incremental ด้วย Sequence Number แทนการลบทั้งเส้น
# (Idempotent เหมือนเดิม แต่ไม่มีหน้าต่างที่ ACL ว่างเปล่าเลย)
- name: Remediate ACL incrementally (no delete-then-recreate)
  cisco.ios.ios_acls:
    config:
      - afi: ipv4
        acls:
          - name: MGMT-ACCESS
            aces: "{{ golden_template_aces }}"
    state: replaced   # ใช้ 'replaced' ที่คำนวณ Diff ทีละ ACE แทน 'overridden' ที่ลบทั้งเส้นก่อน
```

```
! ทางเลือกระดับ Manual/Fallback ถ้าต้อง Remediate ผ่าน CLI ตรงในกรณีฉุกเฉิน
BRANCH3-RTR(config)# ip access-list extended MGMT-ACCESS-STAGING
BRANCH3-RTR(config-ext-nacl)# permit tcp 10.10.99.0 0.0.0.255 any eq 22
BRANCH3-RTR(config-ext-nacl)# permit tcp host 10.10.99.50 any eq 22
BRANCH3-RTR(config-ext-nacl)# deny ip any any log
BRANCH3-RTR(config-ext-nacl)# exit
BRANCH3-RTR(config)# line vty 0 4
BRANCH3-RTR(config-line)# access-class MGMT-ACCESS-STAGING in    ! สลับไปเส้นใหม่ที่สมบูรณ์แล้วในคำสั่งเดียว (Atomic)
BRANCH3-RTR(config-line)# exit
BRANCH3-RTR(config)# no ip access-list extended MGMT-ACCESS       ! ลบเส้นเก่าได้อย่างปลอดภัยตอนนี้ (ไม่มีใครอ้างอิงแล้ว)
```

> **หลักการสำคัญที่สุดของ Ticket นี้**: การสลับ (Swap) ไปยัง ACL ใหม่ที่สมบูรณ์แล้ว **ในคำสั่งเดียว**
> (`access-class MGMT-ACCESS-STAGING in`) ไม่มีหน้าต่างเวลาที่ Management Access ไม่มีการป้องกันเลย
> เทียบกับการลบ ACL เดิมก่อนแล้วค่อยสร้างใหม่ทีละบรรทัดที่เปิดช่องว่างอันตรายไว้ — หลักการ **Atomic
> Change** นี้ใช้ได้กับทุก Object ที่ถูกอ้างอิงจาก Live Traffic เสมอ ไม่ใช่แค่ ACL (เช่น Route-map,
> Prefix-list ที่ผูกกับ Neighbor ที่ Active อยู่ก็มีความเสี่ยงแบบเดียวกัน)

### 949.6 Verify (แนวคิดตรวจสอบ ไม่ใช่ Live Access)

ทีม Automation ควรเพิ่ม Assertion ใน Pipeline เพื่อป้องกันปัญหาซ้ำ:

```yaml
- name: Assert ACL is never empty while bound to VTY
  assert:
    that: "acl_ace_count | int > 0"
    fail_msg: "ACL {{ acl_name }} มี 0 ACE ระหว่าง Remediation — ห้าม Apply ทับ Live Reference"
```

---

## Step 950 — Full Lab: Ticket 5 (โจทย์ยากที่สุด) + กลยุทธ์การสอบ Diagnose Module

### 950.1 คำเตือนก่อนเริ่ม Ticket นี้

Ticket นี้คือ Ticket ที่ยากที่สุดของ Part นี้ และจงใจใส่ **Red Herring (หลักฐานที่ดูน่าสงสัยแต่ไม่
เกี่ยวข้อง)** เข้ามาปนกับหลักฐานจริง — ทดสอบโดยตรงว่าผู้เรียนยึดหลักการ [Part 91 Step 903](part-091-advanced-troubleshooting-methodology.md)
(Falsifiable Hypothesis, ไม่ใช่ไล่ตามทุกความผิดปกติที่เห็น) ได้จริงหรือไม่ ภายใต้ความกดดันเรื่องเวลา

### Ticket #DIAG-9505

**Exhibit A — คำร้องเรียนจากลูกค้า**

> "ทีม Call Center รายงานว่าคุณภาพเสียงสนทนาทางโทรศัพท์ (VoIP) ระหว่าง `BRANCH4-RTR` (สาขาใหม่ที่
> เพิ่งเปิด Dual-Homed WAN Link เมื่อสัปดาห์ก่อน) กับสำนักงานใหญ่มีปัญหา **บางสายเสียงขาดๆ หายๆ
> (Choppy) แต่บางสายกลับใช้งานได้ปกติสมบูรณ์ทุกจุด** ไม่สามารถระบุ Pattern ที่แน่นอนได้ว่าสายไหน
> จะมีปัญหา ปัญหาเริ่มพร้อมกับการเปิดใช้ Link ที่สองของสาขานี้เป๊ะ ทีม Monitoring แจ้งเพิ่มเติมว่า
> พบ CRC Error สะสมจำนวนมากที่ Interface `GigabitEthernet0/3` บน `CORE-SW2` และมี Security Alert
> เรื่อง SSH Login จาก IP แปลกใหม่เข้า `WAN-EDGE-2` ในช่วงเวลาใกล้เคียงกันด้วย"

**Exhibit B — `show ip eigrp topology` (เก็บจาก `BRANCH4-RTR`)**

```
BRANCH4-RTR# show ip eigrp topology
EIGRP-IPv4 VR(CAMPUS) Topology Table for AS(100)/ID(1.1.1.40)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 10.10.30.0/24, 2 successors, FD is 30720, serno 812
        via 10.255.140.1 (30720/28160), GigabitEthernet0/1   [Link A — Primary MPLS]
        via 10.255.141.1 (46080/30720), GigabitEthernet0/2   [Link B — Backup Internet-VPN]
```

**Exhibit C — Running-config Excerpt (`BRANCH4-RTR`, ส่วน EIGRP และ QoS)**

```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   variance 3
  exit-af-topology
 exit-address-family
!
interface GigabitEthernet0/1
 description Primary MPLS Link
 service-policy output VOICE-LLQ
!
interface GigabitEthernet0/2
 description Backup Internet-VPN Link (Added last week)
 ip address 172.31.14.2 255.255.255.252
 no service-policy output VOICE-LLQ
```

**Exhibit D — `show policy-map interface GigabitEthernet0/1` (เก็บจาก `BRANCH4-RTR`, บางส่วน)**

```
BRANCH4-RTR# show policy-map interface GigabitEthernet0/1 | section Voice
  Class-map: VOICE (match-any)
    Queueing
    Strict Priority
    30720000 bps, drop rate 0 bps
```

**Exhibit E — `show interfaces GigabitEthernet0/3` (เก็บจาก `CORE-SW2`, ตามที่ Monitoring แจ้งใน
Exhibit A)**

```
CORE-SW2# show interfaces GigabitEthernet0/3
GigabitEthernet0/3 is administratively down, line protocol is down
  ...
  1284 input errors, 1284 CRC, 0 frame, 0 overrun, 0 ignored
  Last input never, output never
```

**Exhibit F — Security Log (เก็บจาก `WAN-EDGE-2`, ตามที่ Alert แจ้งใน Exhibit A)**

```
WAN-EDGE-2# show logging | include SSH
%SEC_LOGIN-5-LOGIN_SUCCESS: Login Success [user: netops-jane] [Source: 10.10.99.87]
```

**Exhibit G — บันทึกจากทีม NOC (ประกอบ Log ข้อ F)**

> "IP `10.10.99.87` คือ Laptop ของวิศวกร NOC คนใหม่ (เริ่มงานสัปดาห์นี้) ที่เพิ่งได้รับสิทธิ์ SSH
> เข้า `WAN-EDGE-2` ตามขั้นตอนปกติ — ยืนยันแล้วว่าเป็น Authorized Access ไม่ใช่การบุกรุก"

**คำถาม**: จาก Exhibit ทั้งหมดข้างต้น (A-G) — (1) ระบุว่า Exhibit ใดเป็น Red Herring และเพราะ
เหตุใด (2) Root Cause ที่แท้จริงคืออะไร (3) ควรแก้ไขด้วยคำสั่งใด

---

### 950.2 เฉลย: แยก Red Herring ออกก่อนตาม Step 941.4 กฎข้อ 3

| Exhibit | เกี่ยวข้องหรือไม่ | เหตุผล |
|---|---|---|
| **E** (CRC Error ที่ `CORE-SW2` Gi0/3) | **Red Herring** | Interface สถานะ `administratively down` (ปิดโดยตั้งใจ) และ `Last input never` — ตัวเลข CRC Error เป็น **ค่าที่สะสมมาจากอดีตก่อนถูกปิด Interface** ไม่ได้ Increment อยู่ ณ ปัจจุบัน และ Interface นี้ไม่ได้อยู่ใน Path ของ `BRANCH4-RTR` เลยแม้แต่ Hop เดียว (ไม่มีความเกี่ยวข้องทาง Topology) |
| **F, G** (SSH Login จาก IP ใหม่) | **Red Herring** | Exhibit G ยืนยันชัดเจนว่าเป็น Authorized Access ของพนักงานใหม่ — ไม่มีความเกี่ยวข้องกับ Voice Quality เลย เป็นคนละโดเมนโดยสิ้นเชิง (Security Alert ที่ไม่ใช่ Incident) |
| **B, C, D** | **เกี่ยวข้องจริง** | ชี้ตรงไปที่ EIGRP Unequal-Cost Load Balancing และ QoS Policy ที่ไม่ครบทั้ง 2 Link |

> **บทเรียนสำคัญของ 950.2**: ผู้เรียนที่เสียเวลาไปกับ Exhibit E และ F/G (เช่น พยายามอธิบายว่า CRC
> Error ทำให้เสียง Choppy หรือสงสัยว่ามีการบุกรุกกระทบ Performance) จะ **หมดเวลาที่จำกัดต่อ Ticket
> ไปกับสิ่งที่ไม่ใช่คำตอบ** — นี่คือสิ่งที่ Step 941.4 เตือนไว้ล่วงหน้าและเป็นทักษะที่แยกผู้สอบผ่าน
> จากผู้สอบไม่ผ่านในสถานการณ์เวลาจำกัดจริง

### 950.3 วิเคราะห์ Exhibit B — EIGRP Unequal-Cost Load Balancing ด้วย `variance 3`

```
P 10.10.30.0/24, 2 successors, FD is 30720, serno 812
        via 10.255.140.1 (30720/28160), GigabitEthernet0/1   [Link A — Primary MPLS]
        via 10.255.141.1 (46080/30720), GigabitEthernet0/2   [Link B — Backup Internet-VPN]
```

`variance 3` (Exhibit C) หมายความว่า EIGRP จะยอมรับ Path ใดๆ ที่ **Metric ไม่เกิน 3 เท่าของ
Feasible Distance ต่ำสุด** เข้ามาเป็น Successor เพิ่มสำหรับ Load Balancing — ในที่นี้ Path ผ่าน
`GigabitEthernet0/2` (Metric 46080) ยังไม่เกิน `30720 × 3 = 92160` จึงถูกรับเข้ามาเป็น **Successor
ตัวที่ 2 จริง** (ปรากฏ `2 successors` ใน Exhibit B) — Traffic ไปปลายทาง `10.10.30.0/24` จะถูก
**Load-Balance ข้าม 2 Link นี้ตาม CEF Hash ต่อ Flow** (Per-Destination หรือ Per-Packet ขึ้นกับ
Config เพิ่มเติมที่ไม่ได้ให้มา แต่ Default ของ IOS คือ Per-Destination) — สายที่ Hash ไปทาง Link A
(MPLS, Primary) จะปกติ ส่วนสายที่ Hash ไปทาง Link B (Backup Internet-VPN) จะมีปัญหา **ตรงกับอาการ
"บางสายปกติ บางสายมีปัญหา โดยไม่มี Pattern ชัดเจน" ทุกประการ**

### 950.4 วิเคราะห์ Exhibit C, D — ช่องว่างที่แท้จริง

```
interface GigabitEthernet0/1
 service-policy output VOICE-LLQ            ! มี LLQ (Low-Latency Queueing) สำหรับ Voice

interface GigabitEthernet0/2
 no service-policy output VOICE-LLQ         ! ไม่มี LLQ เลย — เป็น Best-Effort ทั้งหมด
```

Exhibit D ยืนยันว่า `GigabitEthernet0/1` มี **Strict Priority Queue** สำหรับ Class `VOICE` ทำงาน
อยู่จริง (Drop Rate 0 bps) — แต่ **ไม่มี Exhibit ใดยืนยันว่า `GigabitEthernet0/2` มี Policy เดียวกัน
เลย** และ Config Excerpt ยืนยันตรงตัวว่า **ไม่มี `service-policy output` ใดๆ ผูกอยู่กับ Link B**
เมื่อทีม Implementation เพิ่ม Backup Link ใหม่สัปดาห์ก่อน (Exhibit C บอกชัดว่า "Added last week")
พวกเขา Config IP Address และ Routing (EIGRP) ให้ทำงานได้ถูกต้อง **แต่ไม่ได้ Copy QoS Policy จาก
Primary Link มาด้วย** — เป็น Configuration Gap ที่ไม่กระทบ Reachability เลย (Ping/Routing ผ่าน
ปกติ 100%) จึงไม่มีใครสังเกตจนกว่า Voice Traffic จะถูก Hash ไปทาง Link นี้จริงและเจอ Congestion
จาก Traffic อื่นที่ไม่มี Priority Queueing แยกให้

### 950.5 Root Cause สมบูรณ์

**การเพิ่ม Backup WAN Link (`GigabitEthernet0/2`) ที่ `BRANCH4-RTR` ทำให้เกิด EIGRP Unequal-Cost
Load Balancing (`variance 3`) ข้าม 2 Link ที่มีคุณภาพ QoS ต่างกัน** — Link A (Primary MPLS) มี
Voice LLQ Policy สมบูรณ์ ส่วน Link B (Backup Internet-VPN) ที่เพิ่งเพิ่มไม่มี QoS Policy ใดๆ เลย
Voice RTP Flow ที่ถูก CEF Hash ไปทาง Link B จึงต้องแย่ง Bandwidth กับ Traffic อื่นแบบ Best-Effort
เกิด Congestion/Delay/Jitter เป็น Choppy Voice เฉพาะบางสาย (ตาม Flow ที่ Hash ไปทาง Link B) ตรงกับ
อาการทุกจุดใน Exhibit A ส่วน Exhibit E และ F/G เป็น Red Herring ที่ไม่เกี่ยวข้องตามที่วิเคราะห์ไว้
ใน 950.2

### 950.6 การแก้ไข

```
BRANCH4-RTR(config)# interface GigabitEthernet0/2
BRANCH4-RTR(config-if)# service-policy output VOICE-LLQ
BRANCH4-RTR(config-if)# exit
```

> **ทางเลือกเสริมที่ควรพิจารณาคู่กัน** (คำตอบระดับ Design ที่ได้คะแนนเพิ่มถ้าข้อสอบถามแนวทางป้องกัน
> ระยะยาว): ถ้า Backup Link มี Bandwidth หรือคุณภาพต่ำกว่า Primary มากจริง อาจพิจารณาลด `variance`
> ลง หรือใช้ **EIGRP Offset-list/Delay Tuning** เพื่อไม่ให้ Voice Traffic ถูก Load-Balance ไปทาง
> Backup Link เลยตราบใดที่ Primary ยังพร้อมใช้งานอยู่ (สงวน Backup Link ไว้สำหรับ Failover
> เท่านั้น ไม่ใช่ Load-Sharing ตลอดเวลา) — Trade-off ต้องหารือกับ Requirement ว่าต้องการ
> Load-Sharing เพื่อใช้ Bandwidth เต็มที่ หรือ Consistency ของ Voice Quality เป็นหลัก

### 950.7 Verify

```
BRANCH4-RTR# show policy-map interface GigabitEthernet0/2 | section Voice
  Class-map: VOICE (match-any)
    Queueing
    Strict Priority
    30720000 bps, drop rate 0 bps
```

---

### 950.8 กลยุทธ์การสอบ Diagnose Module (สรุปเฉพาะ Module นี้)

**การจัดสรรเวลา**: Diagnose Module ฝังอยู่ใน Deploy, Operate Module (~5 ชั่วโมงรวม) ตามที่
[Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) อธิบายไว้ — Ticket แบบ Diagnose
มักมีจำนวนไม่มาก (โดยทั่วไป 3-6 Ticket) ปนอยู่กับงาน Deploy/Optimize ในเวลาเดียวกัน หลักการจัดเวลา
ที่แนะนำ:

| ขั้นตอน | เวลาที่แนะนำต่อ Ticket | หลักการ |
|---|---|---|
| อ่าน Exhibit ทั้งหมดรอบแรกแบบเร็ว (ไม่วิเคราะห์ลึก) | 1-2 นาที | จับภาพรวมก่อนว่ามี Exhibit กี่ชิ้น อาการคืออะไร |
| ตั้ง Hypothesis แรกจาก Pattern-Matching (Step 901) | 1 นาที | ใช้ Signature ที่คุ้นเคย (`up/up`, State ค้าง, Correlation เวลา) เลือกจุดสงสัยอันดับ 1 |
| ไล่ Exhibit ที่เหลือเพื่อยืนยัน/หักล้าง Hypothesis | 3-5 นาที | ตัด Exhibit ที่ไม่เกี่ยวข้องออกอย่างมีเหตุผล (Step 950.2) ไม่ใช่แค่ "รู้สึกว่าไม่เกี่ยว" |
| เขียนคำตอบ (Root Cause + คำสั่งแก้ไข) | 2-3 นาที | ใช้โครงสร้างคำตอบตามหัวข้อ 950.9 ด้านล่าง |
| **รวมต่อ Ticket** | **8-12 นาที** | ถ้าเกิน 15 นาทีในหนึ่ง Ticket ควร **บันทึกสิ่งที่พิสูจน์แล้วว่าไม่ใช่ (Step 909)** แล้วข้ามไปก่อน |

**กฎ Time-Boxing เฉพาะ Diagnose Module**: ต่างจาก Deploy Module ที่ทำไม่ทันยังเก็บ Partial Credit
จาก Config ที่พิมพ์ไปได้บ้าง — **Diagnose Module ส่วนใหญ่ให้คะแนนแบบ All-or-Nothing ต่อ Ticket**
(ตอบ Root Cause ผิด = ไม่ได้คะแนน Ticket นั้นทั้งหมด แม้จะระบุ Exhibit ที่ถูกต้องได้บางส่วน) ดังนั้น
**ควรข้าม Ticket ที่ไม่แน่ใจไปก่อนแล้วกลับมาทำหลังจบ Ticket อื่นที่มั่นใจกว่า** เพื่อไม่ให้เวลาที่
จำกัดหมดไปกับ Ticket เดียวที่ตัน (ใช้หลักการ Two-Strikes ของ [Part 91 Step 908](part-091-advanced-troubleshooting-methodology.md)
เป็นตัวช่วยตัดสินใจว่าเมื่อไหร่ควรพักไว้ก่อน)

### 950.9 โครงสร้างคำตอบที่แนะนำเมื่อข้อสอบขอ Written Justification

ถ้า Ticket ขอให้ **อธิบายเหตุผล** ไม่ใช่แค่เลือกคำตอบ ควรเขียนตามโครงสร้าง 4 ส่วนนี้เสมอ (สั้น
กระชับ ตรงประเด็น — ไม่ต้องเขียนเป็นเรียงความ):

```
1. GIVEN (Evidence ที่ใช้)
   ระบุ Exhibit ที่นำมาใช้จริงเป็นข้อๆ (เช่น "Exhibit B แสดง 2 successors จาก variance 3")

2. ROOT CAUSE (สาเหตุที่แท้จริง — ประโยคเดียว ชัดเจน ไม่คลุมเครือ)
   "Backup Link ใหม่ไม่มี QoS Voice Policy ทำให้ Voice ที่ Hash ไปทาง Link นี้ไม่มี Priority Queue"

3. WHY EVIDENCE SUPPORTS THIS (เชื่อมทุก Exhibit ที่เกี่ยวข้องกับ Root Cause)
   "Exhibit C ยืนยันไม่มี service-policy บน Gi0/2 / Exhibit D ยืนยันมีบน Gi0/1 / อาการ 'บางสาย
   ปกติบางสายไม่' ตรงกับพฤติกรรม Per-Flow Load Balancing"

4. FIX + VERIFY (คำสั่งแก้ไข + คำสั่งที่จะยืนยันว่าแก้ถูก)
   "service-policy output VOICE-LLQ บน Gi0/2 / ตรวจด้วย show policy-map interface Gi0/2"
```

โครงสร้างนี้ตรงกับที่ทีม Real-World เขียนใน Post-Incident Review เอกสารจริง ([Part 69 Step
690](part-069-mega-lab-1.md)) — ฝึกใช้รูปแบบเดียวกันตั้งแต่ตอนสอบจะช่วยให้เขียนได้เร็วขึ้นเพราะ
ไม่ต้องคิดโครงสร้างใหม่ทุกครั้งภายใต้ความกดดันเวลา

---

## แบบฝึกหัดทวนความเข้าใจ Part 95

1. เพราะเหตุใด Diagnose Module ของ CCIE Lab Exam จริงจึงไม่ให้ผู้สมัคร Login เข้าอุปกรณ์จริงได้
   ทั้งที่ Deploy Module ในสอบเดียวกันให้ Login ได้เต็มที่? อธิบายทักษะที่ทั้งสอง Module ต้องการวัด
   ต่างกันอย่างไร
2. ใน Ticket 1 (Step 942-943) เพราะเหตุใด Prefix `172.20.60.0/24` ถึงไม่ถูกรับเข้ามาที่ `WAN-EDGE-1`
   ทั้งที่ Session BGP กับ Partner ยังคง Established ปกติและ Prefix เก่ายังใช้งานได้?
3. ใน Ticket 2 (Step 944-945) อธิบายว่าทำไม OSPF Neighbor State ที่ค้างอยู่ที่ `EXSTART` (ไม่ใช่
   `2WAY` หรือ `LOADING`) จึงเป็นสัญญาณที่ชี้ตรงไปที่ MTU Mismatch โดยเฉพาะ?
4. ใน Ticket 4 (Step 948-949) เพราะเหตุใดปัญหา SSH ล้มเหลวถึงเกิดขึ้นเฉพาะที่ `BRANCH3-RTR` เท่านั้น
   ทั้งที่ Playbook เดียวกันรันกับทุกสาขาพร้อมกัน?
5. ใน Ticket 5 (Step 950) จงอธิบายว่าทำไม Exhibit E (CRC Error ที่ `CORE-SW2`) และ Exhibit F/G
   (SSH Login จาก IP ใหม่) จึงถูกจัดเป็น Red Herring — การมี Evidence ที่ "ดูน่าสงสัย" อยู่ในชุด
   Ticket มีจุดประสงค์อะไรในการสอบ?

**เฉลย:**

1. Deploy Module วัดว่า "พิมพ์ Config ถูกต้องไหมภายใต้เวลาจำกัด" ซึ่งจำเป็นต้องมี Live Access
   เพื่อทดสอบ Config จริง ส่วน Diagnose Module วัด "อ่าน Evidence ที่มีจำกัดแล้วสรุป Root Cause ถูก
   ไหม" ซึ่งจำลองสถานการณ์งานจริงที่วิศวกรต้องวิเคราะห์จาก Log/Exhibit ที่คนอื่นเก็บมาให้โดยไม่มี
   สิทธิ์ Login เอง — การให้ Live Access ใน Diagnose จะเปิดช่องให้ Shotgun Troubleshooting (ลองสุ่ม
   จนเจอ) ซึ่งไม่ได้วัดทักษะการวิเคราะห์ที่ต้องการจริง
2. เพราะ Prefix-list `PARTNER-IN` ที่ Apply ขาเข้า (`in`) เขียนแบบ Explicit ทีละ Prefix
   (`172.20.50.0/24` และ `172.20.51.0/24`) โดยไม่มี Sequence ใดครอบคลุม `172.20.60.0/24` เลย —
   Prefix ใหม่จึงตกไปโดน Implicit Deny ท้าย Prefix-list โดยอัตโนมัติ ทั้งที่ Session และ Prefix
   เก่าไม่ได้รับผลกระทบเลยเพราะยัง Match Sequence ที่มีอยู่ได้ปกติ
3. เพราะ `EXSTART` คือ State ที่ Router เริ่มแลกเปลี่ยน DBD Packet แรกและตกลง Master/Slave — Cisco
   IOS ตรวจสอบค่า MTU ที่ Neighbor ประกาศมาเทียบกับ MTU ของตัวเองที่ State นี้ ถ้าไม่ตรงกันจะไม่ยอม
   ขยับ State ต่อเลย ต่างจาก `2WAY` (ค้างเพราะไม่ได้เป็น DR/BDR ที่ต้อง Full กับกัน) หรือ `LOADING`
   (ค้างเพราะ LSU/LSAck ไม่สมบูรณ์ ซึ่งมักเป็นปัญหาอื่น) — Signature ที่ State เฉพาะ `EXSTART` จึง
   ชี้ MTU ได้เจาะจงกว่า State อื่น
4. เพราะ Playbook ใช้รูปแบบ "Delete-then-Recreate" ที่เปิดหน้าต่างเวลาที่ ACL ไม่มี ACE เลย
   (Implicit Deny All) ระหว่างการ Remediate — หน้าต่างนี้เกิดกับทุกสาขาที่มี Drift เหมือนกัน แต่ที่
   `BRANCH3-RTR` มี RTT สูงถึง 210 ms (เทียบกับ 8-12 ms ของสาขาอื่น) ทำให้แต่ละ RPC Transaction ผ่าน
   NETCONF ใช้เวลานานกว่ามาก หน้าต่างที่ ACL ว่างเปล่าจึงยาวพอที่จะชนกับ SSH Connection จริงในบางคืน
   ขณะที่สาขาอื่นหน้าต่างสั้นเกินกว่าจะมีใครสังเกตเห็น
5. Exhibit E ถูกจัดเป็น Red Herring เพราะ Interface อยู่ในสถานะ `administratively down` (ปิดไว้
   ตั้งใจ) และ CRC Error เป็นค่าสะสมจากอดีตที่ไม่ได้ Increment อยู่ ณ ปัจจุบัน ไม่ได้อยู่ใน Path ของ
   ปัญหาเลย ส่วน Exhibit F/G ถูกจัดเป็น Red Herring เพราะ Exhibit G ยืนยันชัดเจนว่าเป็น Authorized
   Access ไม่เกี่ยวกับ Voice Quality — จุดประสงค์ของการใส่ Evidence แบบนี้คือทดสอบว่าผู้สอบยึดหลัก
   "Hypothesis ต้องอธิบาย Symptom ได้ครบและมี Evidence รองรับจริง" (Part 91 Step 903/941.4) ได้ภาย
   ใต้ความกดดันเรื่องเวลา ไม่ใช่ไล่ตามทุกสิ่งที่ดูผิดปกติโดยไม่มีเหตุผลเชื่อมโยง

---

## สรุป Part 95

Part นี้จำลอง **CCIE Diagnose Module** เต็มรูปแบบเป็นครั้งแรกของหลักสูตร — เริ่มจากอธิบายรูปแบบ
ข้อสอบที่แท้จริง (Read-Only, Ticket-Based, ไม่มี Live Access) ใน **Step 941** จากนั้นให้ Ticket
5 ใบที่ไม่เปิดเผยโดเมนล่วงหน้าเลยแม้แต่ใบเดียว: **Ticket 1** (Step 942-943) ทดสอบ BGP Prefix-list
Filtering ที่ไม่ Sync กับการเปลี่ยนแปลงจริง, **Ticket 2** (Step 944-945) ทดสอบ OSPF MTU Mismatch
ผ่านรูปแบบ Topology Diagram, **Ticket 3** (Step 946-947) ทดสอบ Cross-Domain ระหว่าง QoS Policer
กับ Multicast Traffic, **Ticket 4** (Step 948-949) ทดสอบ Cross-Domain ที่พบน้อยระหว่าง Security
ACL กับ Automation Pipeline ที่ไม่ Atomic, และปิดท้ายด้วย **Ticket 5** (Step 950) ที่ยากที่สุด —
รวม EIGRP Unequal-Cost Load Balancing กับ QoS Gap เข้าด้วยกัน พร้อม Red Herring 2 ชุดที่ทดสอบวินัย
ตาม Methodology ของ [Part 91](part-091-advanced-troubleshooting-methodology.md) โดยตรง ปิดท้าย
ด้วยกลยุทธ์การจัดเวลาและโครงสร้างการเขียนคำตอบเฉพาะ Module นี้

ข้อสังเกตสำคัญที่สุดของ Part นี้คือ **ทุก Root Cause ที่ใช้ในทั้ง 5 Ticket ไม่ใช่ความรู้ใหม่เลย** —
ทุกกลไกถูกสอนไปแล้วในหลักสูตรตั้งแต่ Part 1 ถึง Part 94 (Prefix-list จาก Part 29, MTU/OSPF จาก
Part 57, QoS Policer จาก Part 40, Multicast จาก Part 41-42, Automation จาก Part 47-48, EIGRP
Variance จาก Part 28) สิ่งที่ Part นี้ฝึกจริงๆ คือ **การประกอบความรู้ที่กระจัดกระจายเข้าด้วยกันจาก
Evidence ที่จำกัดและไม่บอกโดเมนล่วงหน้า** ซึ่งคือทักษะที่แยกผู้สอบผ่าน CCIE Lab Exam จริงจากผู้ที่
ท่องจำ Checklist แยกโดเมนได้เก่งแต่ประกอบภาพรวมไม่ได้ภายใต้ความกดดันเรื่องเวลา

✅ **พร้อมสำหรับ Part 96**: จากการวินิจฉัยปัญหาที่มีอยู่แล้ว เราจะขยับไปยัง Task Type ที่ตรงข้ามกัน
โดยสิ้นเชิง — **Design Module** ที่ไม่มี Config ผิดให้ตามหา แต่มี **Business Requirement ใหม่**
ให้อ่านแล้วเลือก Topology/Protocol/Technology ที่เหมาะสมที่สุดพร้อมอธิบายเหตุผลและ Trade-off ที่
ยอมรับได้ — ทักษะที่ [Part 71 Step 704](part-071-ccie-lab-blueprint-overview.md) ชี้ไว้ว่าต่างจาก
CCNP โดยพื้นฐานที่สุด

**ไปต่อ:** [Part 96 — CCIE Lab Scenario #2: Design Module →](part-096-ccie-lab-scenario-2-design.md)
