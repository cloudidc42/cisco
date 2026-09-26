# Part 24 — QoS Fundamentals (Introductory)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 231–240 จาก 1000**

> ต่อจาก [Part 23 — Automation Basics](part-023-automation-basics.md) ที่สอนพื้นฐาน Network
> Automation (NETCONF/RESTCONF, Python, Ansible เบื้องต้น) Part นี้เราจะเปลี่ยนมาที่หัวข้อ
> **QoS (Quality of Service)** — เรื่องที่ทำให้ traffic ประเภทต่างกัน (เสียง, วิดีโอ, data) ได้รับ
> การดูแลที่ต่างกันเวลาที่ Network เกิด congestion

> **สำคัญมาก — ขอบเขตของ Part นี้**: Part 24 นี้เป็น **QoS ระดับ CCNA (Introductory QoS)**
> เท่านั้น เราจะเรียนแนวคิดพื้นฐาน, Classification/Marking, Trust Boundary, และ Auto-QoS
> ซึ่งเพียงพอสำหรับข้อสอบ CCNA และงาน Access-Layer ทั่วไป **เนื้อหาเชิงลึกระดับ CCNP ENCOR**
> — MQC (Modular QoS CLI) แบบ manual เต็มรูปแบบ, Class-Based Weighted Fair Queuing (CBWFQ),
> Low-Latency Queuing (LLQ), Traffic Shaping (GTS/CBS), Traffic Policing (CB Policing, dual-rate
> policing), และ WRED (Weighted Random Early Detection) แบบละเอียด — จะถูกสอนเต็มรูปแบบใน
> **Part 40 (QoS ระดับ CCNP ENCOR)** Part นี้จะพูดถึงหัวข้อเหล่านี้แค่ "แนะนำให้รู้จัก" (preview)
> เพื่อไม่ให้ผู้เรียนงงว่าทำไมยังไม่ได้เรียนคำสั่งพวกนี้แบบเต็ม

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 231 | ทำไมต้องมี QoS — Convergence ของ Voice/Video/Data บนเครือข่ายเดียว |
| 232 | คุณลักษณะของ QoS — Bandwidth, Delay, Jitter, Packet Loss (พร้อมตัวอย่างคำนวณ) |
| 233 | QoS Models — Best Effort, IntServ, DiffServ |
| 234 | Classification & Marking — CoS, IP Precedence, DSCP |
| 235 | Trust Boundary — จุดที่ควร Classify Traffic และคำสั่ง `mls qos trust` |
| 236 | 4 กลุ่มกลไกของ QoS — ภาพรวมก่อนเจาะลึกใน Part 40 |
| 237 | Auto-QoS — `auto qos voip cisco-phone` |
| 238 | Congestion Management เบื้องต้น — FIFO vs Priority Queuing |
| 239 | คำสั่ง Verify QoS — `show mls qos`, `show queueing`, `show policy-map interface` |
| 240 | Lab: Trust Boundary + Auto-QoS บน ACCESS-SW1/ACCESS-SW2 (VLAN 20 VOICE) |

---

## Step 231 — ทำไมต้องมี QoS

### Convergence: จุดเริ่มต้นของปัญหา

ในอดีต Voice ใช้ระบบโทรศัพท์แยก (PBX/PSTN สาย copper) และ Data ใช้เครือข่าย IP แยกกันคนละ
Infrastructure วันนี้ทุกอย่างถูกรวมเป็น **Converged Network** เดียว — เสียง (VoIP), วิดีโอ
(Video Conference), และ Data (Web, File Transfer, Email) เดินทางบนสาย Ethernet เส้นเดียวกัน,
ผ่าน Switch/Router ตัวเดียวกัน, แข่งกันใช้ Bandwidth เดียวกัน

ใน Lab ของหลักสูตรนี้ ตัวอย่างที่ชัดที่สุดคือ **ACCESS-SW1/ACCESS-SW2** (ดู Part 3 Step 26):
IP Phone (VLAN 20 VOICE, subnet `10.10.20.0/24`) เสียบสาย Ethernet เข้ากับ PC (VLAN 10 SALES,
subnet `10.10.10.0/24`) แล้วต่อสายเดียวเข้า switch port — Voice traffic กับ Data traffic
วิ่งบน **สายเดียวกัน (physical link เดียวกัน)** แค่แยกด้วย 802.1Q tag คนละ VLAN เท่านั้น

```
        [IP Phone VLAN20]───┐
                             ├──(สายเดียว 1 Gbps)──→ ACCESS-SW1 Gi1/0/1 ──→ Uplink to DIST-SW1
        [PC VLAN10]─────────┘
```

เมื่อ Uplink จาก ACCESS-SW1 ไป DIST-SW1 เกิด **Congestion** (เช่น มีคนหลาย PC โหลดไฟล์ใหญ่พร้อมกัน
จนเต็ม Bandwidth ของ Uplink) — packet ที่มาไม่พร้อมกันจะต้องเข้า **Queue** ของ Interface นั้น
ถ้า Switch ไม่มี QoS เลย มันจะใช้นโยบาย **Best Effort**: ทุก packet เท่าเทียมกัน, เข้า Queue เดียว
(FIFO — First In First Out), ใครมาก่อนได้ไปก่อน — **ไม่สนใจว่า packet นั้นเป็นเสียงคนคุยโทรศัพท์
อยู่หรือเป็นแค่ไฟล์ที่กำลัง download เบื้องหลัง**

### ความต้องการของ Voice ต่างจาก Data อย่างสิ้นเชิง

| คุณสมบัติที่ต้องการ | VoIP (เสียง) | Data ทั่วไป (HTTP, FTP, Email) |
|---|---|---|
| ความไวต่อ Delay (Latency) | สูงมาก — ต้อง **< 150 ms** (one-way, ตามมาตรฐาน ITU-T G.114) | ต่ำ — รอ 1-2 วินาทีไม่มีผลกระทบมาก |
| ความไวต่อ Jitter (ความแปรปรวนของ Delay) | สูงมาก — ต้อง **< 30 ms** | ไม่สนใจ (TCP มี buffer/reorder จัดการให้) |
| ความไวต่อ Packet Loss | สูงมาก — ต้อง **< 1%** ไม่งั้นเสียงขาดๆ หายๆ | รับได้มากกว่า — TCP retransmit ให้อัตโนมัติ |
| ลักษณะ Traffic | Steady stream เล็กๆ ต่อเนื่อง (เช่น G.711 = 64 kbps คงที่ตลอดสาย) | Bursty — บางช่วงเงียบ บางช่วงพุ่งเต็ม bandwidth |
| Protocol ชั้น Transport | UDP (RTP) — **ไม่มี retransmit** ถ้าหายไปคือหายไปเลย | TCP ส่วนใหญ่ — มี retransmit อัตโนมัติ |
| ผลเมื่อ QoS ไม่ดี | เสียงสะดุด, หน่วง, echo, คุยไม่ทัน (garbled voice) | หน้าเว็บโหลดช้าลง — ผู้ใช้รำคาญแต่ใช้งานได้ |

**ประเด็นสำคัญ**: Voice ใช้ RTP over UDP เพราะไม่ต้องการ retransmit (จะช้าเปล่าๆ — พูดซ้ำสดกว่า)
แต่นั่นหมายความว่า **ถ้า packet เสียงหายหรือมาช้าเกินไป จะไม่มีการแก้ไขที่ Layer 4** เสียงนั้นจะขาด
หายไปเลย (หรือถูก Jitter Buffer ที่ปลายทางทิ้งไปเพราะมาไม่ทันจังหวะ) — นี่คือเหตุผลที่ Voice
ต้อง**ได้รับความสำคัญพิเศษ (Priority Treatment)** ตั้งแต่ที่ Switch/Router ทุกตัวที่มันเดินทางผ่าน

### สิ่งที่เกิดขึ้นถ้าไม่มี QoS (Best Effort ล้วนๆ)

```
Uplink ACCESS-SW1 → DIST-SW1 (1 Gbps) เกิด Congestion ชั่วครู่ (burst จาก PC หลายเครื่อง)

ไม่มี QoS:                              มี QoS (แยก Queue ให้ Voice):
┌─────────────────────────┐             ┌─────────────────────────┐
│ FIFO Queue เดียว          │             │ Priority Queue (Voice)   │──→ ออกก่อนเสมอ (แทบไม่มี delay)
│ [Data][Voice][Data][Data]│──→ ส่งตามลำดับ│ [Voice][Voice][Voice]    │
│ [Voice][Data]...         │   ที่มาถึง    ├─────────────────────────┤
└─────────────────────────┘             │ Normal Queue (Data)       │──→ ส่งเมื่อ Priority Queue ว่าง
   ผลลัพธ์: Voice packet ต้องรอ           │ [Data][Data][Data]...    │
   ต่อคิวหลัง Data packet ใหญ่ๆ           └─────────────────────────┘
   → Delay/Jitter พุ่ง → เสียงสะดุด        ผลลัพธ์: Voice ไม่ต้องรอ Data เลย
```

นี่คือเหตุผลทั้งหมดว่าทำไมองค์กรที่ deploy VoIP (เช่น Cisco IP Phone ใน Lab ของเรา) **ต้อง**
Config QoS ที่ Access Switch เป็นอย่างน้อย — ไม่ทำก็ยังใช้งานได้ในสภาวะเครือข่ายไม่แน่น
แต่ทันทีที่มี Congestion คุณภาพเสียงจะแย่ลงทันที

---

## Step 232 — คุณลักษณะของ QoS: Bandwidth, Delay, Jitter, Packet Loss

QoS พูดถึง 4 มิติหลักของคุณภาพการส่ง packet ผ่านเครือข่าย โดยแต่ละมิติมีนิยามที่ต้องรู้แม่นสำหรับ
ข้อสอบ CCNA/CCNP และงานจริง

### 1. Bandwidth (แบนด์วิดท์)

ความสามารถในการส่งข้อมูลสูงสุดของ link ในหน่วยเวลา (bps) — เช่น Uplink ACCESS-SW1↔DIST-SW1
เป็น 1 Gbps QoS ไม่ได้เพิ่ม Bandwidth ให้ แต่ช่วย **จัดสรร (Allocate)** ว่าใครได้ใช้ก่อน/มากแค่ไหน
เวลา Bandwidth ไม่พอสำหรับทุกคน

### 2. Delay / Latency (ความหน่วง)

เวลาที่ packet ใช้เดินทางจากต้นทางถึงปลายทาง แบ่งเป็นส่วนประกอบย่อย:

| ประเภท Delay | คำอธิบาย | ตัวอย่างจริงใน Lab |
|---|---|---|
| **Propagation Delay** | เวลาที่สัญญาณเดินทางในสาย (ระยะทาง ÷ ความเร็วแสงในสื่อกลาง) | Fiber ระหว่างอาคาร ~5 microsecond/km — ผลกระทบน้อยมากในระยะ Campus |
| **Serialization Delay** | เวลาที่ใช้ "แปลง" bit เป็นสัญญาณส่งออก interface ตามความเร็ว link | packet 1500 byte บน link 1 Gbps ≈ 12 microsecond, บน link 64 kbps (WAN เก่า) ≈ 187 ms! |
| **Queuing Delay** | เวลาที่ packet ต้องรอใน Queue ก่อนถูกส่งออก (ส่วนที่ QoS ควบคุมได้มากที่สุด) | ยิ่ง congestion มาก ยิ่งรอนาน — นี่คือจุดที่ QoS เข้ามาช่วย |
| **Processing Delay** | เวลาที่ device ใช้ตัดสินใจ forward (ตรวจ routing table, ACL, ฯลฯ) | บน Cisco switch/router สมัยใหม่ใช้ CEF (Hardware-based) จึง < 1 ms |

**มาตรฐานสำหรับ VoIP (ITU-T G.114)**: One-way delay ควร **≤ 150 ms** ถ้าเกิน 150-400 ms
คุณภาพเสียงจะเริ่มแย่ลงจนสังเกตได้ (คุยแล้วสวนกัน พูดชนกัน) เกิน 400 ms แทบใช้งานไม่ได้เลย

### 3. Jitter (ความแปรปรวนของ Delay)

Jitter คือ **ความแตกต่างของ Delay ระหว่าง packet ต่อ packet** ในสาย Voice stream เดียวกัน
— ไม่ใช่ Delay สูงหรือต่ำ แต่คือ **ความไม่คงที่**

**ตัวอย่างคำนวณ**: สมมติ Codec G.711 ส่ง packet เสียงทุก 20 ms (ปกติของ VoIP)

```
เวลาที่ควรมาถึง (ตามทฤษฎี):     0ms   20ms   40ms   60ms   80ms
เวลาที่มาถึงจริง (มี Jitter):    0ms   35ms   38ms   65ms   79ms
ผลต่าง (Inter-packet delay variation):  -    35ms   3ms    27ms   14ms
```

Jitter Buffer ที่ปลายทาง (IP Phone) จะพยายามเก็บ packet ไว้ชั่วครู่แล้วเรียงลำดับใหม่ให้เสียง
ฟังต่อเนื่อง แต่ Jitter Buffer มีขนาดจำกัด (โดย default IP Phone Cisco ใช้ ~20-50 ms) —
ถ้า Jitter สูงเกิน Buffer จะรับไม่ทัน ต้อง**ทิ้ง packet** ทำให้เสียงสะดุด (drop-out)

**ค่ามาตรฐานที่ยอมรับได้สำหรับ VoIP**: Jitter ควร **< 30 ms** (ยิ่งน้อยยิ่งดี ค่าที่ดีมากคือ < 10 ms)

### 4. Packet Loss (การสูญเสียแพ็กเก็ต)

Packet ที่ถูกส่งแต่ไม่ถึงปลายทาง (ถูก drop ที่ Queue เต็ม, CRC error, หรือ congestion)

**ผลกระทบต่อเสียงตามเปอร์เซ็นต์ (ใช้ Codec G.711 เป็นตัวอย่าง)**:

| Packet Loss % | ผลกระทบต่อคุณภาพเสียงที่ได้ยิน |
|---|---|
| 0% | เสียงชัดเจนสมบูรณ์ |
| < 1% | แทบไม่รู้สึก (Concealment algorithm ของ Codec ซ่อนได้) — **นี่คือเกณฑ์มาตรฐานที่ยอมรับได้** |
| 1-3% | เริ่มมี "click" หรือเสียงกระตุกเป็นครั้งคราว พอทนได้ |
| 5% | เสียงขาดเป็นช่วงๆ ชัดเจน คุยลำบาก |
| > 10% | คุยไม่รู้เรื่อง แทบใช้งานไม่ได้ |

**เกณฑ์รวมสำหรับ VoIP ที่ต้องจำ (จะออกข้อสอบ CCNA แน่นอน)**:

| Metric | เกณฑ์มาตรฐาน (Cisco / ITU-T) |
|---|---|
| One-way Latency | < 150 ms |
| Jitter | < 30 ms |
| Packet Loss | < 1% |
| Bandwidth ต่อสาย G.711 call | 17.6 kbps (payload) + Layer 2 overhead ≈ 64-106 kbps รวม header ทั้งหมด |

---

## Step 233 — QoS Models: Best Effort, IntServ, DiffServ

มี 3 แนวทาง (Model) หลักในการจัดการ QoS บนเครือข่าย IP:

### 1. Best Effort (ไม่มี QoS เลย)

- ทุก packet ได้รับการปฏิบัติแบบเดียวกันหมด ไม่มีการแยกประเภท
- ใช้ FIFO Queue เดี่ยว ใครมาก่อนไปก่อน
- **ข้อดี**: ง่ายที่สุด ไม่ต้อง config อะไร, ไม่มี overhead
- **ข้อเสีย**: ไม่มีการันตีอะไรเลย — Voice จะแย่ลงทันทีที่มี congestion (ตามที่อธิบายใน Step 231)
- นี่คือค่า default ของทุก Interface บน Cisco IOS ถ้าไม่ config QoS อะไรเพิ่ม

### 2. IntServ (Integrated Services) — RFC 1633

แนวคิด: **จองทรัพยากร (Reserve Bandwidth) ล่วงหน้า** ให้กับแต่ละ Flow (การสนทนาหนึ่งคู่)
ก่อนที่จะส่งข้อมูลจริง โดยใช้ Protocol ชื่อ **RSVP (Resource Reservation Protocol)**

```
Host A ──RSVP PATH msg──→ Router1 ──→ Router2 ──→ Host B
Host A ←──RSVP RESV msg── Router1 ←── Router2 ←── Host B
      (จองสำเร็จ = ทุก Router บนเส้นทางกัน Bandwidth ไว้ให้ Flow นี้โดยเฉพาะ)
```

- **ข้อดี**: การันตี Bandwidth ที่แม่นยำที่สุด (Hard QoS guarantee) ต่อ flow
- **ข้อเสีย (จุดตายที่ทำให้ไม่ใช้จริงในองค์กรใหญ่)**: **ไม่ Scale** — ทุก Router บนเส้นทางต้องเก็บ
  **State ของทุก Flow** (Per-flow state) ไว้ใน memory ถ้ามี 10,000 การสนทนาผ่าน Router
  ตัวเดียวกัน Router ต้องจำสถานะของทั้ง 10,000 flow — ใน ISP Backbone ที่มี flow เป็นล้านๆ
  ต่อวินาที นี่เป็นไปไม่ได้ในทางปฏิบัติ
- **สรุปสำหรับข้อสอบ**: IntServ = "RSVP, per-flow, ไม่ scale" — แทบไม่ถูกใช้ใน Enterprise/ISP
  จริง ยกเว้นบางกรณีเฉพาะทาง (เช่น บาง MPLS TE Tunnel Reservation)

### 3. DiffServ (Differentiated Services) — RFC 2474/2475 — **Model ที่ใช้จริงในทางปฏิบัติ**

แนวคิด: **ไม่จองอะไรล่วงหน้า และไม่เก็บ State ต่อ Flow** แต่ทำ 2 อย่างแทน:

1. **Classify & Mark** traffic ที่ Edge ของเครือข่าย (ใส่ "ป้ายกำกับ" ลงใน packet เช่น DSCP value)
2. ทุก Router/Switge ที่ traffic ผ่าน จะดู "ป้ายกำกับ" นั้นแล้วให้บริการตาม **Per-Hop Behavior (PHB)**
   ที่กำหนดไว้ — โดยไม่ต้องรู้จัก Flow นั้นเป็นรายตัว แค่ดู class ของมันพอ

```
        Classify+Mark ที่ Edge                 ทุก Hop ดูแค่ "ป้าย" แล้วปฏิบัติตาม Policy ของตัวเอง
┌──────────────┐        ┌───────────┐  DSCP=EF  ┌───────────┐  DSCP=EF  ┌───────────┐
│ IP Phone     │──packet│ ACCESS-SW │══════════▶│ DIST-SW   │══════════▶│ CORE-SW   │
│ (มาร์คด้วยตัวเอง)│       │ (Trust/Mark)│  (ไม่ต้องรู้ว่าเป็น  │ (PHB: Priority Q) │ (PHB: Priority Q)
└──────────────┘        └───────────┘   flow ไหน แค่เห็น    └───────────┘            └───────────┘
                                          class = EF ก็พอ)
```

**เปรียบเทียบ IntServ vs DiffServ (ต้องแยกให้ได้แม่นสำหรับข้อสอบ)**:

| คุณสมบัติ | IntServ | DiffServ |
|---|---|---|
| หน่วยที่จัดการ | ต่อ Flow (per-flow) | ต่อ Class (per-class, กลุ่ม traffic) |
| Protocol หลัก | RSVP | ไม่มี Protocol พิเศษ — ใช้ field ใน IP Header (DSCP) |
| State ที่ต้องเก็บ | ทุก Router ต้องจำทุก Flow | ไม่ต้องเก็บ State ของ Flow เลย — Stateless ต่อ flow |
| Scalability | ต่ำ (ไม่ scale กับจำนวน flow มาก) | สูง (scale ได้ดีมาก — นี่คือเหตุผลที่ใช้จริง) |
| ความแม่นยำของการันตี | Hard Guarantee (แน่นอน) | Soft Guarantee (ดีกว่า Best Effort มาก แต่ไม่ 100% แน่นอน) |
| ใช้จริงในองค์กร/ISP ปัจจุบัน | แทบไม่ใช้ | **ใช้เป็นมาตรฐานหลักทุกที่** (นี่คือ Model ที่หลักสูตรนี้จะสอนทั้งหมด) |

> **สรุป Step 233**: หลักสูตรนี้ (และ Cisco QoS ทั้งหมดตั้งแต่ CCNA ถึง CCIE) ใช้ **DiffServ Model**
> เป็นหลัก ทุกกลไกที่จะเรียนต่อจากนี้ (CoS, DSCP, Trust Boundary, Queuing, Policing, Shaping)
> ล้วนเป็นส่วนประกอบของ DiffServ ทั้งสิ้น

---

## Step 234 — Classification & Marking: CoS, IP Precedence, DSCP

**Classification** คือการ "ตรวจสอบ" ว่า packet/frame นี้เป็น traffic ประเภทไหน (Voice, Video, Data
สำคัญ, Data ทั่วไป) ส่วน **Marking** คือการ "เขียนป้ายกำกับ" ลงใน Header ของ frame/packet
เพื่อให้ Router/Switch ตัวถัดไปรู้ประเภทของมันทันทีโดยไม่ต้องตรวจสอบซ้ำ (re-classify)

### Layer 2 Marking: CoS (Class of Service) — 802.1p

- อยู่ใน **802.1Q VLAN Tag** (Tag ที่ใส่ตอนทำ Trunk/VLAN — ดู Part 4)
- ขนาด **3 bit** ในฟิลด์ที่เรียกว่า **PCP (Priority Code Point)** ภายใน 802.1Q Tag
- ค่าได้ตั้งแต่ **0-7** (2³ = 8 ระดับ)
- **ข้อจำกัดสำคัญ**: CoS มีความหมายเฉพาะภายใน **Layer 2 Segment เดียวกัน** เท่านั้น — เมื่อ packet
  ถูก Route ข้าม Layer 3 (เช่นผ่าน SVI ของ DIST-SW) 802.1Q Tag (และ CoS ที่อยู่ในนั้น) **จะหายไป**
  เพราะ Routing คือการสร้าง Frame ใหม่ทุกครั้ง — นี่คือเหตุผลที่ต้องมี Marking ที่ Layer 3 ด้วย
  (DSCP) เพื่อให้ QoS Information **เดินทางข้าม Layer 3 ได้**

```
802.1Q Tag Format (4 byte):
┌────────────┬─────┬───┬──────────────┐
│ TPID       │ PCP │DEI│ VLAN ID (12b) │
│ (0x8100)   │(3b) │(1b)│              │
│ 2 byte     │        2 byte           │
└────────────┴─────┴───┴──────────────┘
                ↑
          นี่คือ CoS (802.1p) — 3 bit, ค่า 0-7
```

### Layer 3 Marking: IP Precedence (Legacy) vs DSCP (มาตรฐานปัจจุบัน)

ทั้งสองใช้ฟิลด์เดียวกันใน IPv4 Header คือ **ToS (Type of Service) byte** (byte ที่ 2 ของ IP Header)
แต่ตีความต่างกัน:

```
IPv4 Header — ToS Byte (8 bit):

แบบเดิม (IP Precedence, RFC 791):        แบบใหม่ (DSCP, RFC 2474 — ใช้จริงปัจจุบัน):
┌─────────────┬───────────────┐         ┌─────────────────────┬─────┐
│ IP Prec (3b)│  ToS bits (5b)│         │  DSCP (6 bit)         │ ECN │
│  0-7        │  (ไม่ค่อยใช้)  │         │  0-63                 │(2b) │
└─────────────┴───────────────┘         └─────────────────────┴─────┘
```

**IP Precedence** (Legacy, 3 bit, ค่า 0-7) — เป็นระบบเก่า ยังใช้ได้ (backward compatible)
แต่ให้ความละเอียดน้อย (แค่ 8 class)

**DSCP (Differentiated Services Code Point)** — มาตรฐานปัจจุบัน, **6 bit** (0-63 = 64 ระดับ
ความละเอียดสูงกว่า IP Precedence ถึง 8 เท่า) ออกแบบให้ **Backward Compatible** กับ IP Precedence:
3 bit บนของ DSCP (bit ที่ 5-3) ตรงกับค่า IP Precedence เดิม ทำให้ Router รุ่นเก่าที่รู้จักแค่
IP Precedence ยังอ่านค่าได้ถูกต้อง (แค่ไม่เห็นความละเอียดที่เพิ่มมา)

### ตาราง DSCP Value ที่ต้องจำ (ออกข้อสอบบ่อยที่สุดของหัวข้อ QoS)

| DSCP Name | Binary (6 bit) | Decimal | IP Precedence เทียบเท่า | ใช้กับ Traffic ประเภท |
|---|---|---|---|---|
| **CS0 / Default (BE)** | 000000 | 0 | 0 (Routine) | Best Effort — traffic ทั่วไปที่ไม่ได้ mark อะไร (default) |
| CS1 | 001000 | 8 | 1 | Scavenger class (traffic ที่ไม่สำคัญ เช่น P2P, backup ที่ไม่เร่งรีบ) |
| AF11 | 001010 | 10 | 1 | Bulk Data, Low Drop Precedence |
| AF12 | 001100 | 12 | 1 | Bulk Data, Medium Drop Precedence |
| AF13 | 001110 | 14 | 1 | Bulk Data, High Drop Precedence |
| CS2 | 010000 | 16 | 2 | Network Management (SNMP, Syslog) |
| AF21 | 010010 | 18 | 2 | Transactional/Interactive Data, Low Drop |
| AF22 | 010100 | 20 | 2 | Transactional/Interactive Data, Medium Drop |
| AF23 | 010110 | 22 | 2 | Transactional/Interactive Data, High Drop |
| CS3 | 011000 | 24 | 3 | Signaling (Call Control — SIP/H.323 signaling, ไม่ใช่ตัวเสียง) |
| AF31 | 011010 | 26 | 3 | Multimedia Streaming, Low Drop |
| AF32 | 011100 | 28 | 3 | Multimedia Streaming, Medium Drop |
| AF33 | 011110 | 30 | 3 | Multimedia Streaming, High Drop |
| CS4 | 100000 | 32 | 4 | Realtime Interactive (Video Conference control) |
| AF41 | 100010 | 34 | 4 | Multimedia Conferencing, Low Drop |
| AF42 | 100100 | 36 | 4 | Multimedia Conferencing, Medium Drop |
| AF43 | 100110 | 38 | 4 | Multimedia Conferencing, High Drop |
| **CS5** | 101000 | 40 | 5 | Broadcast Video |
| **EF (Expedited Forwarding)** | **101110** | **46** | 5 | **Voice (VoIP RTP payload) — priority สูงสุดสำหรับเสียงพูด** |
| CS6 | 110000 | 48 | 6 | Network Control (Routing Protocol: OSPF, EIGRP, BGP Hello/Update) |
| CS7 | 111000 | 56 | 7 | Reserved สำหรับ Internal Network Control สูงสุด |

**ค่าที่ต้องจำแม่นที่สุดสำหรับ Voice QoS (จะใช้ใน Step 240 Lab)**:

| Traffic | DSCP ที่แนะนำโดย Cisco (Best Practice) | CoS ที่สัมพันธ์กัน |
|---|---|---|
| Voice bearer (RTP — เสียงพูดจริง) | **EF (46)** | CoS 5 |
| Voice/Video Signaling (SIP, SCCP, H.323 call setup) | **CS3 (24)** | CoS 3 |
| Video Conferencing | AF41 (34) | CoS 4 |
| Data ทั่วไป (VLAN 10 SALES) | **CS0 / Default (0)** | CoS 0 |

**AF (Assured Forwarding) Class ทำงานอย่างไร**: ตัวเลขแรก (1-4) คือ Class (Queue กลุ่มไหน)
ตัวเลขที่สอง (1-3) คือ **Drop Precedence** — ยิ่งเลขสูง ยิ่งถูก drop ก่อนตอนเกิด congestion
(เช่น AF13 จะถูก drop ก่อน AF11 ถ้า Queue ของ Class 1 เต็ม) กลไกนี้ทำงานคู่กับ WRED
(จะเรียนใน Step 236 แบบ preview และเต็มรูปแบบใน Part 40)

---

## Step 235 — Trust Boundary: จุดที่ควร Classify Traffic

### แนวคิด Trust Boundary

**Trust Boundary** คือ**จุดในเครือข่ายที่ Switch/Router ตัดสินใจว่าจะ "เชื่อ" Marking (CoS/DSCP)
ที่ติดมากับ packet หรือไม่** — คำถามสำคัญคือ: **ใครควรเป็นคนทำ Classification/Marking ครั้งแรก?**

หลักการที่ Cisco แนะนำ (Best Practice) คือ **Classify ให้เร็วที่สุด ใกล้ Source มากที่สุด**
เพราะยิ่ง Mark เร็ว อุปกรณ์ที่อยู่ปลายทางถัดไปก็ทำงานง่ายขึ้น (แค่ดู tag ที่มีอยู่แล้ว ไม่ต้องเปิด
packet มาวิเคราะห์ใหม่ ซึ่งกิน CPU/ASIC resource)

```
Trust Boundary ที่แนะนำที่สุด (Ideal — Endpoint ทำเอง):

┌──────────────┐  CoS=5, DSCP=EF   ┌──────────────┐   Trust CoS/DSCP    ┌──────────┐
│ Cisco IP Phone│──(mark เอง)─────▶│ ACCESS-SW1    │──(เชื่อ ไม่ mark ใหม่)─▶│ DIST-SW1  │
│ (Trusted      │                   │ Trust Boundary │                     │           │
│  Endpoint)    │                   │ อยู่ที่นี่      │                     │           │
└──────────────┘                   └──────────────┘                     └──────────┘

Trust Boundary ที่ไม่ปลอดภัย (ถ้า PC ธรรมดา mark เอง):

┌──────────────┐  CoS=5 (โกง!)     ┌──────────────┐
│ PC (User      │──(mark ปลอมเป็น──▶│ ACCESS-SW1    │  ต้องไม่เชื่อ PC — ต้อง Classify/Re-mark ใหม่
│  ทั่วไป)       │   Voice traffic)  │ (ต้อง Reclass) │  ที่ Switch เอง (หรือปฏิเสธ Trust จาก PC port)
└──────────────┘                   └──────────────┘
```

**ประเด็นความปลอดภัย**: ถ้า PC ธรรมดา (ที่ไม่ใช่ IP Phone) สามารถ mark CoS=5/DSCP=EF ปลอมตัวเองว่า
เป็น Voice ได้ ผู้ใช้ที่ไม่ดีสามารถ "แซงคิว" ทำให้ traffic ของตัวเอง (เช่น download บิตทอร์เรนต์)
ได้ Priority เท่า Voice จริง ซึ่งจะทำลาย QoS Policy ทั้งระบบ — ดังนั้น Trust Boundary
**ต้องอยู่ในตำแหน่งที่เหมาะสม และต้อง Verify ว่าอุปกรณ์ที่ส่ง Marking มาเป็นอุปกรณ์ที่เชื่อถือได้จริง**

### คำสั่งบน Cisco Catalyst Switch: `mls qos trust`

Cisco Catalyst switch (MLS = MultiLayer Switching hardware) มีคำสั่งระดับ Interface
เพื่อกำหนดว่าจะ **เชื่อ (Trust)** Marking ที่มาจาก device ปลายสายหรือไม่:

| คำสั่ง | ความหมาย |
|---|---|
| `mls qos trust cos` | เชื่อค่า **CoS** (802.1p) ที่ frame ส่งเข้ามา แล้ว switch จะ map ค่า CoS นั้นไปเป็น DSCP ภายในให้เองตาม CoS-to-DSCP map (ใช้ตอน Part 3 Step 26 กับ IP Phone แล้ว) |
| `mls qos trust dscp` | เชื่อค่า **DSCP** ที่ packet มาพร้อมอยู่แล้วโดยตรง (ใช้บน Trunk ระหว่าง switch หรือจาก Router ที่ mark มาแล้ว) |
| `mls qos trust device cisco-phone` | **Conditional Trust** — เชื่อ CoS **เฉพาะเมื่อ** switch ตรวจสอบผ่าน **CDP** แล้วว่าอุปกรณ์ปลายสายเป็น Cisco IP Phone ตัวจริง (ปลอดภัยกว่า `trust cos` เปล่าๆ) |
| `no mls qos trust` (default) | **ไม่เชื่อ** — reset CoS/DSCP ทั้งหมดเป็น 0 (Best Effort) ก่อนส่งต่อ — นี่คือค่า default ของทุก port ที่ไม่ config อะไร |

### `mls qos trust cos` vs `mls qos trust device cisco-phone` — ต่างกันอย่างไร

`mls qos trust cos` (ที่ config ไว้ใน Part 3) จะ **เชื่อ CoS ของทุก frame ที่เข้ามาที่ port นั้น
โดยไม่สนใจว่าอุปกรณ์ปลายสายเป็นอะไร** — ถ้ามีคนเอา PC มาต่อแทน Phone แล้วส่ง frame ที่ mark
CoS=5 มา switch ก็จะเชื่อทันที (ช่องโหว่ตามที่อธิบายด้านบน)

`mls qos trust device cisco-phone` ปลอดภัยกว่า เพราะ switch จะใช้ **CDP (Cisco Discovery
Protocol)** ตรวจสอบก่อนว่า device ที่เสียบอยู่จริงๆ เป็น Cisco IP Phone หรือไม่ (ดูจาก CDP
capability ที่ device ส่งมา) **ถ้าไม่ใช่ Phone จริง (เช่นถอด Phone ออกแล้วเสียบ PC ตรงเข้า port
เดิม) switch จะไม่เชื่อ CoS ที่ PC ส่งมาอีกต่อไป** — เป็น Best Practice ที่แนะนำมากกว่า
`trust cos` เปล่าๆ ในสถานการณ์จริง (จะใช้ใน Lab Step 240)

### Boundary ที่แนะนำในภาพรวมของ Enterprise Campus

```
[IP Phone]──[PC]──┐
                    │ Access Port                    Trunk (เชื่อ DSCP ที่ mark มาแล้ว)
              ┌─────┴──────┐  mls qos trust      ┌──────────┐  mls qos trust dscp   ┌──────────┐
              │ ACCESS-SW1  │  device cisco-phone │ DIST-SW1  │──────────────────────▶│ CORE-SW1  │
              │ (Classify/  │─────────────────────▶│ (เชื่อสิ่งที่│                        │           │
              │  Mark ครั้งแรก)│  (Trust Boundary)    │  ACCESS mark มา)│                       │           │
              └────────────┘                      └──────────┘                       └──────────┘
                     ↑
        จุดนี้คือ "Trust Boundary" ที่แท้จริงขององค์กร — Classify/Verify ที่นี่ครั้งเดียว
        ที่เหลือ (DIST, CORE) แค่ "เชื่อและปฏิบัติตาม PHB" (DiffServ model จาก Step 233)
```

---

## Step 236 — 4 กลุ่มกลไกของ QoS (ภาพรวมก่อนเจาะลึกใน Part 40)

QoS mechanism ทั้งหมดบน Cisco IOS แบ่งเป็น 4 กลุ่มใหญ่ ตามลำดับที่ packet จะเจอเมื่อวิ่งผ่าน
interface หนึ่งๆ:

```
Packet เข้า Interface
        │
        ▼
┌───────────────────┐
│ 1. CLASSIFICATION   │  ← "นี่คือ traffic ประเภทไหน?" (ACL match, DSCP match, NBAR)
│    & MARKING        │  ← "ติดป้ายกำกับ" (set CoS/DSCP) — เรียนแล้วใน Step 234-235
└─────────┬──────────┘
          ▼
┌───────────────────┐
│ 2. CONGESTION       │  ← ทำงานที่ Egress Queue "ก่อน" packet ถูกส่งออก — จัดการว่า
│    AVOIDANCE        │    packet ไหนควรถูก "ทิ้งล่วงหน้า" (drop) เพื่อป้องกัน Queue ล้น
│    (เช่น WRED)       │    ก่อนที่จะเกิด Tail Drop รุนแรง — **Preview เท่านั้น, เต็มรูปแบบ Part 40**
└─────────┬──────────┘
          ▼
┌───────────────────┐
│ 3. CONGESTION       │  ← "ใครออกก่อน เมื่อมีหลาย Queue รอส่งพร้อมกัน?" — กลไก Queuing
│    MANAGEMENT       │    (FIFO, PQ, CBWFQ, LLQ) — Step 238 สอนแค่แนวคิดเบื้องต้น
│    (Queuing)        │    **เต็มรูปแบบ (CBWFQ/LLQ ด้วย MQC) อยู่ใน Part 40**
└─────────┬──────────┘
          ▼
┌───────────────────┐
│ 4. POLICING /       │  ← "จำกัดปริมาณ traffic ไม่ให้เกินที่กำหนด"
│    SHAPING          │  ← Policing = ทิ้ง/mark-down ส่วนที่เกิน (ทันที, ไม่มี buffer)
│                     │  ← Shaping = "ชะลอ" ส่วนที่เกินไว้ใน buffer แล้วค่อยส่งทีหลัง (ไม่ drop ทันที)
└────────────────────┘  ← **Preview เท่านั้น, เต็มรูปแบบ (CIR/Bc/Be, Token Bucket) อยู่ใน Part 40**
```

### สรุปสั้นๆ แต่ละกลุ่ม (สำหรับจดจำระดับ CCNA)

| กลุ่มกลไก | ทำงานตรงไหน | เปรียบเทียบง่ายๆ | รายละเอียดเต็มอยู่ที่ |
|---|---|---|---|
| Classification & Marking | Ingress (ตอน packet เข้า) | "ติดป้ายว่าใครเป็นใคร" | Step 234-235 (Part นี้) |
| Congestion Avoidance | Egress Queue (ก่อนล้น) | "กันไม่ให้รถติดสนิท โดยทยอยเบรกก่อนถึงจุดอุดตัน" (WRED) | Part 40 |
| Congestion Management | Egress Queue (ตอนมีคิว) | "ตำรวจจราจรตัดสินว่ารถคันไหนไปก่อน" (Queuing) | Step 238 (เบื้องต้น) / Part 40 (เต็ม) |
| Policing / Shaping | Ingress หรือ Egress | "จำกัดความเร็วรถไม่ให้เกินที่กำหนด" | Part 40 |

**เหตุผลที่ Part 24 (CCNA) ไม่ลงคำสั่ง MQC เต็มรูปแบบ**: การ config `class-map`, `policy-map`,
`service-policy` แบบ manual (Modular QoS CLI) เป็นทักษะระดับ CCNP ENCOR ที่ต้องเข้าใจ Queuing
Algorithm เชิงลึก (Weighted Fair Queuing คำนวณ weight อย่างไร, Token Bucket คำนวณ Bc/Be อย่างไร)
ระดับ CCNA แค่ต้องเข้าใจ **แนวคิด + ใช้ Auto-QoS ได้** ก็เพียงพอสำหรับข้อสอบและงาน Access-Layer
พื้นฐาน — Part 40 จะสอน MQC เต็มรูปแบบทั้งหมด

---

## Step 237 — Auto-QoS: `auto qos voip cisco-phone`

### ทำไมต้องมี Auto-QoS

การ config QoS แบบ manual เต็มรูปแบบ (MQC) มีความซับซ้อนสูง ต้องเข้าใจ Queue, Bandwidth
Allocation, Trust Boundary อย่างละเอียด — สำหรับ Deployment พื้นฐานที่ต้องการแค่ "รองรับ VoIP
ให้ดีพอ" อย่างรวดเร็ว Cisco จึงสร้างฟีเจอร์ **Auto-QoS** ที่ generate configuration ที่แนะนำ
(Best Practice template) ให้อัตโนมัติเพียงคำสั่งเดียว

### คำสั่งหลัก (บน Interface ที่ต่อกับ Cisco IP Phone)

```
ACCESS-SW1(config)# interface GigabitEthernet1/0/1
ACCESS-SW1(config-if)# auto qos voip cisco-phone
```

คำสั่งนี้จะบอก IOS ว่า: "Port นี้ต่อกับ Cisco IP Phone ที่ต่อ PC ต่อพ่วง (Cascade) ให้ตั้งค่า
QoS ที่เหมาะสมให้อัตโนมัติ"

### สิ่งที่ `auto qos voip cisco-phone` ทำให้อัตโนมัติ (แปลงเป็นคำสั่งจริงที่เห็นได้ด้วย `show running-config`)

เมื่อรันคำสั่งเดียวนี้ IOS จะ generate คำสั่งเทียบเท่าประมาณนี้โดยอัตโนมัติ (ตัวเลขจริงต่างกันไป
ตามรุ่น IOS/Switch แต่แนวคิดเหมือนกันทุกรุ่น):

```
! คำสั่งที่ Auto-QoS สร้างให้อัตโนมัติ (ตัวอย่างค่าที่ IOS มักจะแทรกให้)
ACCESS-SW1(config)# mls qos                                    ! เปิด QoS ระดับ Global (บังคับต้องมี)
ACCESS-SW1(config-if)# mls qos trust device cisco-phone        ! Conditional Trust Boundary (ตาม Step 235)
ACCESS-SW1(config-if)# mls qos trust cos
ACCESS-SW1(config-if)# auto qos voip cisco-phone
ACCESS-SW1(config-if)# priority-queue out                      ! เปิด Strict Priority Queue สำหรับ Voice
ACCESS-SW1(config-if)# wrr-queue bandwidth ...                 ! จัดสรร Bandwidth ให้ Queue อื่นๆ (WRR)
ACCESS-SW1(config-if)# srr-queue bandwidth ...                 ! (บางรุ่นใช้ SRR แทน WRR — ขึ้นกับ platform)
```

### สรุปสิ่งที่ Auto-QoS ทำ (แนวคิดที่ต้องเข้าใจ ไม่ต้องจำ syntax เป๊ะทุกรุ่น)

| สิ่งที่ Auto-QoS Config ให้ | เหตุผล |
|---|---|
| เปิด `mls qos` ระดับ Global | QoS ทั้งหมดบน Catalyst switch ถูก**ปิดโดย default** ต้องเปิด global ก่อนคำสั่ง trust ระดับ interface จะมีผลจริง |
| ตั้ง Conditional Trust Boundary (`trust device cisco-phone`) | ตาม Best Practice Step 235 — เชื่อเฉพาะ Cisco Phone จริง |
| ตั้งค่า CoS-to-DSCP Map และ DSCP-to-Queue Map มาตรฐาน | Map CoS 5 (Voice) ไปยัง DSCP EF และ Queue ที่มี priority สูงสุดให้อัตโนมัติ |
| เปิด Priority Queue (Strict Priority) สำหรับ Voice traffic | Voice ได้ queue พิเศษที่ถูกส่งก่อน queue อื่นเสมอ (preview ของ Step 238) |
| จัดสรร Bandwidth ให้ Queue อื่นๆ ตามสัดส่วนที่ Cisco แนะนำ | ป้องกันไม่ให้ Data traffic (VLAN 10) ถูก "อด" Bandwidth จนหมด |

### Auto-QoS vs Manual MQC — ควรใช้แบบไหน

| ด้าน | Auto-QoS | Manual MQC (Part 40) |
|---|---|---|
| ความง่าย | ง่ายมาก — คำสั่งเดียวจบ | ซับซ้อน ต้อง config class-map/policy-map เอง |
| ความละเอียดในการปรับแต่ง | จำกัด (ใช้ template ที่ Cisco กำหนดไว้) | ปรับแต่งได้ทุกรายละเอียด (Bandwidth %, Queue depth, ฯลฯ) |
| เหมาะกับ | Access-Layer พื้นฐาน, องค์กรขนาดเล็ก-กลาง, deploy เร็ว | Enterprise ขนาดใหญ่ที่ต้องการ Policy ละเอียด, WAN Edge/Core ที่ต้อง fine-tune |
| ระดับที่สอน | **CCNA (Part 24 — Part นี้)** | **CCNP ENCOR (Part 40)** |

> **ข้อควรระวังสำคัญ**: Auto-QoS เป็น "จุดเริ่มต้นที่ดี" แต่**ควรตรวจสอบ config ที่มันสร้างให้เสมอ
> ด้วย `show running-config`** ก่อนนำไปใช้งานจริงในองค์กรขนาดใหญ่ — เพราะค่า default ของ
> Auto-QoS อาจไม่เหมาะกับทุกสถานการณ์ (เช่น ถ้ามี Video Conference จำนวนมากอาจต้องปรับ Bandwidth
> Allocation เพิ่มเติมแบบ manual ด้วย MQC ใน Part 40)

---

## Step 238 — Congestion Management เบื้องต้น: FIFO vs Priority Queuing

**Congestion Management (Queuing)** ตอบคำถาม: **เมื่อมีหลาย packet รอส่งออก Interface พร้อมกัน
(Queue มีมากกว่า 1 คิว) ใครควรถูกส่งออกก่อน?**

### FIFO (First In, First Out) — Default ของทุก Interface ที่ไม่ config อะไร

```
Queue เดียว:  [P1][P2][P3][P4][P5] ──→ ส่งออกตามลำดับที่มาถึง (ไม่สนใจประเภท)
```

- ไม่มีการแยกประเภท packet เลย — ทุก packet เข้า Queue เดียวกัน เรียงคิวตามเวลาที่มาถึง
- **ข้อดี**: ง่ายที่สุด ไม่กิน CPU/Resource ในการตัดสินใจ
- **ข้อเสีย**: Voice packet ที่มาทีหลัง Data packet ขนาดใหญ่ ต้องรอต่อคิวเหมือนกันทั้งหมด
  — นี่คือสาเหตุของปัญหาที่อธิบายไว้ใน Step 231

### Priority Queuing (PQ) — Strict Priority

```
┌─────────────────────┐
│ High Priority Queue  │──→ ส่งก่อนเสมอ ถ้ามี packet ค้างอยู่ใน queue นี้ (Voice ไปที่นี่)
│ [Voice][Voice]       │
├─────────────────────┤
│ Medium Priority Queue│──→ ส่งเมื่อ High Priority Queue "ว่างเปล่า" เท่านั้น
│ [Video]              │
├─────────────────────┤
│ Normal Priority Queue│──→ ส่งเมื่อ High + Medium ว่างทั้งคู่
│ [Data ทั่วไป]         │
├─────────────────────┤
│ Low Priority Queue    │──→ ส่งเมื่อ Queue อื่นว่างหมด (Scavenger class)
│ [Backup/P2P]          │
└─────────────────────┘
```

- แบ่งเป็นหลาย Queue ตาม Priority ระดับต่างๆ (สูงสุด 4 ระดับใน PQ แบบดั้งเดิม: High/Medium/
  Normal/Low)
- **กฎเหล็ก**: Queue ที่ Priority สูงกว่าจะถูกส่งออก**ก่อนเสมอ** ถ้ามี packet รออยู่ใน Queue นั้น
  — Queue ที่ต่ำกว่าจะได้ส่งก็ต่อเมื่อ Queue สูงกว่าทั้งหมด "ว่างเปล่า" ในขณะนั้นเท่านั้น
- **ข้อเสียใหญ่ (Starvation)**: ถ้า High Priority Queue มี traffic เข้ามาต่อเนื่องไม่หยุด (เช่น
  ถูกโจมตีหรือ misconfiguration ทำให้ traffic ผิดประเภทถูก mark เป็น High) Queue ที่ต่ำกว่า
  **จะไม่ได้ส่งเลย** (Bandwidth Starvation) — Data traffic ปกติอาจค้างคาไม่ได้ไปไหนตลอดกาล

### เปรียบเทียบสั้นๆ (แนวคิดเบื้องต้นที่ CCNA ต้องรู้)

| Queuing Method | หลักการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| FIFO | ไม่แยกประเภท | ง่ายที่สุด | Voice ปนกับ Data เต็มๆ |
| Priority Queuing (PQ) | Strict Priority ตาม Queue | Voice ได้ Priority สูงสุดแน่นอน | Starvation กับ Queue ต่ำ ถ้า High Queue traffic ล้น |
| **CBWFQ / LLQ** (Part 40) | รวม Fair-share Bandwidth (CBWFQ) + Strict Priority เฉพาะ Voice (LLQ) | แก้ปัญหา Starvation — Voice ได้ Priority Queue เล็กๆ ที่จำกัด bandwidth (ป้องกันการ "กิน" ทั้งหมด) ส่วน Data ที่เหลือแบ่ง Bandwidth อย่างเป็นธรรมด้วย CBWFQ | ซับซ้อนกว่า ต้อง config MQC เต็มรูปแบบ (Part 40) |

> **สิ่งที่ Auto-QoS (Step 237) ทำจริงๆ เบื้องหลัง**: บน Catalyst switch สมัยใหม่ Auto-QoS
> ไม่ได้ใช้ PQ แบบ Strict ล้วนๆ (ที่มีปัญหา Starvation) แต่ใช้กลไกที่ผสมระหว่าง **Priority Queue
> ขนาดจำกัด สำหรับ Voice** กับ **Weighted Round Robin (WRR/SRR) สำหรับ Queue อื่น** ซึ่งเป็น
> แนวคิดเดียวกันกับ LLQ (Low Latency Queuing) ที่จะเรียนเต็มรูปแบบใน Part 40 — Part 24 นี้
> แค่ให้เข้าใจว่า "ทำไมต้องมี Priority Queue" และ "Strict PQ อย่างเดียวมีความเสี่ยง Starvation"
> ก็เพียงพอสำหรับ CCNA

---

## Step 239 — คำสั่ง Verify QoS

### `show mls qos` — สถานะ QoS ระดับ Global

```
ACCESS-SW1# show mls qos
QoS is enabled globally
QoS ip packet dscp rewrite is enabled
Wireless QoS is disabled globally
```

ถ้าเห็น `QoS is disabled globally` แปลว่า **ทุกคำสั่ง `mls qos trust ...` ที่ config ไว้ระดับ
Interface จะไม่มีผลอะไรเลย** — ต้องเปิด global ด้วยคำสั่ง `mls qos` ที่ `(config)#` ก่อนเสมอ
(นี่คือจุดที่มือใหม่พลาดบ่อยที่สุด: config `mls qos trust cos` ที่ interface แล้วสงสัยว่าทำไม
ยังไม่ทำงาน — สาเหตุคือลืมเปิด `mls qos` แบบ global)

### `show mls qos interface` — สถานะ Trust Boundary ต่อ Interface

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: trust cos
trust mode: trust cos
trust enabled flag: ena
COS override: dis
default COS: 0
DSCP Mutation Map: Default DSCP Mutation Map
Trust device: cisco-phone
qos mode: port-based
```

ข้อมูลสำคัญที่ต้องอ่าน: **trust state** (ปัจจุบันเชื่ออะไร), **trust device** (ถ้ามี conditional
trust จะบอกว่าเชื่อเฉพาะ device ประเภทไหน), **default COS** (ค่า CoS ที่ port จะ mark ให้ ถ้า
frame ที่เข้ามาไม่มี CoS ติดมาเลย)

### `show queueing interface` — สถานะ Queue ของ Interface (Preview — เต็มรูปแบบ Part 40)

```
ACCESS-SW1# show queueing interface GigabitEthernet1/0/1
  Interface GigabitEthernet1/0/1 queueing strategy: Class-based queueing
  Port QoS is enabled
  Trust boundary disabled
  Trust state: trust cos
  Extend trust state: not trusted [COS = 0]
  Default COS is 0
  Queueing Mode In Tx direction: mode-cos
  Transmit queues [type = 4]:
    Queue Id    Scheduling  Num of thresholds
    -----------------------------------------
       01         WRR                 02
       02         PRIORITY            01
       03         WRR                 02
       04         WRR                 02
```

จะเห็นว่า Queue 02 ถูกกำหนดให้เป็น **PRIORITY** (นี่คือ Queue ที่ Voice traffic ถูก map เข้าไป
จาก Auto-QoS หรือ manual config) ส่วน Queue อื่นใช้ **WRR (Weighted Round Robin)** — รายละเอียด
วิธีคำนวณ weight และการปรับแต่ง threshold แต่ละ Queue จะสอนเต็มรูปแบบใน Part 40

### `show policy-map interface` — Preview (จะใช้จริงเมื่อมี MQC service-policy ใน Part 40)

```
ACCESS-SW1# show policy-map interface GigabitEthernet1/0/1
% No policy currently attached to this interface's input/output
```

ที่ Step 240 เราใช้ Auto-QoS + `mls qos trust` ล้วนๆ (ไม่ใช่ MQC-based `service-policy`) ดังนั้น
`show policy-map interface` จะไม่แสดงอะไรตอนนี้ — คำสั่งนี้จะมีข้อมูลให้ดูเต็มรูปแบบ
(match statistics, class hit counters, queue drop counters) หลังจากที่เราเรียนสร้าง `class-map`
+ `policy-map` + `service-policy` ด้วยตัวเองใน **Part 40** — ที่แสดงคำสั่งนี้ไว้ตอนนี้ก็เพื่อให้
คุ้นชื่อคำสั่งล่วงหน้า

### สรุปคำสั่ง Verify ที่ต้องจำสำหรับ CCNA

| คำสั่ง | ใช้ตรวจอะไร |
|---|---|
| `show mls qos` | QoS เปิด/ปิดระดับ Global หรือไม่ |
| `show mls qos interface <if>` | Trust state ของ interface นั้น (trust cos/dscp/device) |
| `show queueing interface <if>` | โครงสร้าง Queue ของ interface (กี่ queue, queue ไหนเป็น priority) |
| `show policy-map interface <if>` | สถิติของ MQC Policy ที่ attach ไว้ (เต็มรูปแบบใน Part 40) |
| `show interfaces <if> switchport` | Voice VLAN ที่ config ไว้ (ทวนจาก Part 3) |

---

## Step 240 — Lab: Trust Boundary + Auto-QoS บน ACCESS-SW1/ACCESS-SW2

### เป้าหมาย Lab

ต่อยอดจาก Part 3 Step 26 (ที่ config `switchport voice vlan 20` และ `mls qos trust cos` ไว้แล้ว
บน ACCESS-SW1/ACCESS-SW2) — คราวนี้เราจะ:

1. เปิด **`mls qos` ระดับ Global** (ที่ Part 3 ยังไม่ได้เปิด — เป็นสาเหตุที่ `trust cos`
   ที่ config ไว้ก่อนหน้านี้ **ยังไม่มีผลจริง** จนกว่าจะเปิด global switch นี้)
2. เปลี่ยนจาก `mls qos trust cos` (เชื่อทุกอย่างไม่มีเงื่อนไข) เป็น **`mls qos trust device
   cisco-phone`** (Conditional Trust ตาม Best Practice Step 235)
3. ใช้ **`auto qos voip cisco-phone`** เพื่อให้ IOS generate Queuing/Priority config ที่แนะนำ
   ให้อัตโนมัติ (ตาม Step 237)
4. Verify ผลลัพธ์ด้วยชุดคำสั่งจาก Step 239
5. อธิบายการเปรียบเทียบ Marking ที่เกิดขึ้นจริงระหว่าง VLAN 20 (Voice) กับ VLAN 10 (Data)
   ด้วยลักษณะ Packet Capture

### Topology ที่ใช้ใน Lab นี้ (ทวนจาก Part 1 และ Part 3)

```
                              DIST-SW1
                                 │ Trunk (VLAN 10,20,99)
                                 │
                          ┌──────┴──────┐
                          │ ACCESS-SW1  │
                          └──┬───┬───┬──┘
                    Gi1/0/1  │   │   │  Gi1/0/3
                             │   │   │
                       ┌─────┘   │   └─────┐
                 ┌─────┴────┐    │    ┌─────┴────┐
                 │IP Phone+PC│   │    │IP Phone+PC│  ← Phone เสียบสาย, PC เสียบผ่าน Phone (daisy-chain)
                 │VLAN20+VLAN10│  │    │VLAN20+VLAN10│    (โครงสร้างเดียวกันทุก port ที่ต่อ Phone)
                 └──────────┘    │    └──────────┘
                                  │
                            (ACCESS-SW2 โครงสร้างเดียวกัน ต่อ DIST-SW2)
```

### ขั้นที่ 1 — เปิด QoS ระดับ Global (ทำครั้งเดียวต่อ switch)

```
ACCESS-SW1(config)# mls qos
ACCESS-SW1(config)# exit
ACCESS-SW1# show mls qos
QoS is enabled globally
```

> **ทำไม Part 3 ไม่ได้ error แม้ไม่ได้เปิด `mls qos` global**: IOS ยอมให้ config
> `mls qos trust cos` ที่ interface ได้แม้ global QoS ยังปิดอยู่ (ไม่มี error message)
> แต่คำสั่งนั้นจะ **"ยังไม่ทำงานจริง"** จนกว่าจะเปิด `mls qos` แบบ global — นี่เป็นกับดักที่พบบ่อย
> ในการ deploy จริง ต้องเช็ค `show mls qos` เสมอเป็นขั้นแรกเวลา troubleshoot QoS ไม่ทำงาน

### ขั้นที่ 2-3 — แก้ Trust Boundary เป็น Conditional Trust + เปิด Auto-QoS (ACCESS-SW1)

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 3
ACCESS-SW1(config-if-range)# switchport mode access
ACCESS-SW1(config-if-range)# switchport access vlan 10
ACCESS-SW1(config-if-range)# switchport voice vlan 20
ACCESS-SW1(config-if-range)# no mls qos trust cos              ! ถอด trust cos แบบไม่มีเงื่อนไขออกก่อน
ACCESS-SW1(config-if-range)# mls qos trust device cisco-phone  ! Conditional trust — ตรวจผ่าน CDP
ACCESS-SW1(config-if-range)# auto qos voip cisco-phone         ! ให้ IOS generate Queuing config ให้
ACCESS-SW1(config-if-range)# spanning-tree portfast
ACCESS-SW1(config-if-range)# exit
ACCESS-SW1(config)# exit
ACCESS-SW1# copy running-config startup-config
```

### ACCESS-SW2 (โครงสร้างเดียวกัน)

```
ACCESS-SW2(config)# mls qos
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1 - 3
ACCESS-SW2(config-if-range)# switchport mode access
ACCESS-SW2(config-if-range)# switchport access vlan 10
ACCESS-SW2(config-if-range)# switchport voice vlan 20
ACCESS-SW2(config-if-range)# no mls qos trust cos
ACCESS-SW2(config-if-range)# mls qos trust device cisco-phone
ACCESS-SW2(config-if-range)# auto qos voip cisco-phone
ACCESS-SW2(config-if-range)# spanning-tree portfast
ACCESS-SW2(config-if-range)# exit
ACCESS-SW2(config)# exit
ACCESS-SW2# copy running-config startup-config
```

### Running-Config ฉบับสมบูรณ์ของ ACCESS-SW1 (ส่วนที่เกี่ยวกับ QoS — เพิ่มจาก Part 3)

```
hostname ACCESS-SW1
!
mls qos
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust device cisco-phone
 auto qos voip cisco-phone
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust device cisco-phone
 auto qos voip cisco-phone
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/3
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust device cisco-phone
 auto qos voip cisco-phone
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
end
```

### Verification 1 — `show mls qos interface` (ยืนยัน Trust Boundary ทำงานถูกต้อง)

```
ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: not trusted
trust mode: trust device cisco-phone
trust enabled flag: ena
COS override: dis
default COS: 0
DSCP Mutation Map: Default DSCP Mutation Map
Trust device: cisco-phone
qos mode: port-based
```

**การอ่านผลลัพธ์**: `trust mode: trust device cisco-phone` ยืนยันว่า config ถูกต้อง แต่
`trust state: not trusted` **ในตอนนี้** เพราะยังไม่มี Cisco Phone จริงเสียบเข้ามา (CDP ยัง
ไม่ negotiate เสร็จ) — เมื่อ Phone จริงเสียบเข้ามาและ CDP exchange สำเร็จ ค่านี้จะเปลี่ยนเป็น
`trust state: trust cos` โดยอัตโนมัติ

### Verification 2 — หลังเสียบ IP Phone จริงเข้า Gi1/0/1

```
ACCESS-SW1# show cdp neighbors GigabitEthernet1/0/1 detail
Device ID: SEP001A2B3C4D5E
Entry address(es):
  IP address: 10.10.20.11
Platform: Cisco IP Phone 8865,  Capabilities: Host Phone
Interface: GigabitEthernet1/0/1,  Port ID (outgoing port): Port 1

ACCESS-SW1# show mls qos interface GigabitEthernet1/0/1
GigabitEthernet1/0/1
trust state: trust cos                    ← เปลี่ยนจาก "not trusted" เป็น "trust cos" แล้ว
trust mode: trust device cisco-phone
trust enabled flag: ena
Trust device: cisco-phone
qos mode: port-based
```

### Verification 3 — `show queueing interface` (ยืนยันว่า Auto-QoS สร้าง Priority Queue ให้)

```
ACCESS-SW1# show queueing interface GigabitEthernet1/0/1
  Interface GigabitEthernet1/0/1 queueing strategy: Class-based queueing
  Port QoS is enabled
  Trust state: trust cos
  Trust device: cisco-phone
  Queueing Mode In Tx direction: mode-cos
  Transmit queues [type = 4]:
    Queue Id    Scheduling  Num of thresholds
    -----------------------------------------
       01         WRR                 02
       02         PRIORITY            01     ← Voice traffic (CoS 5) ถูก map มา queue นี้
       03         WRR                 02
       04         WRR                 02
```

### Verification 4 — เปรียบเทียบ Marking จริงด้วย Packet Capture (คำอธิบายลักษณะ capture)

สมมติจับ traffic ที่ Uplink Trunk `GigabitEthernet1/0/24` ของ ACCESS-SW1 (ไปยัง DIST-SW1) ด้วย
Wireshark ผ่าน SPAN port ในขณะที่ IP Phone กำลังโทรออก (RTP stream) และ PC (VLAN 10) กำลัง
browse เว็บพร้อมกัน จะเห็นความแตกต่างของ Marking ชัดเจนดังนี้:

**Frame จาก IP Phone (VLAN 20 — RTP Voice payload)**:

```
Frame 118: 214 bytes on wire
Ethernet II
IEEE 802.1Q Virtual LAN
    .001 .... = Priority: 5 (CoS 5)             ← 802.1p CoS mark โดย Phone เอง
    VLAN ID: 20
Internet Protocol Version 4
    Differentiated Services Field: 0xb8 (DSCP: EF, ECN: Not-ECT)
        1011 10.. = Differentiated Services Codepoint: Expedited Forwarding (46)   ← DSCP EF
User Datagram Protocol
    Source Port: 16384 (RTP dynamic range)
RTP (Real-Time Transport Protocol)
    Payload Type: ITU-T G.711 PCMU
```

**Frame จาก PC (VLAN 10 — HTTP Data)**:

```
Frame 119: 1518 bytes on wire
Ethernet II
IEEE 802.1Q Virtual LAN
    .000 .... = Priority: 0 (CoS 0)              ← ไม่ได้ mark อะไร (default)
    VLAN ID: 10
Internet Protocol Version 4
    Differentiated Services Field: 0x00 (DSCP: CS0/Default, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)                 ← DSCP CS0/BE
Transmission Control Protocol
    Source Port: 51422
    Destination Port: 443
```

**สรุปผลจาก Capture**: Voice frame จาก IP Phone มี **CoS 5 + DSCP EF (46)** ติดมาตั้งแต่ต้นทาง
(Phone mark เอง) ส่วน Data frame จาก PC มี **CoS 0 + DSCP CS0 (Default/Best Effort)** — ACCESS-SW1
ที่ config `mls qos trust device cisco-phone` แล้ว **เชื่อ marking ของ Voice frame และส่งต่อ
โดยไม่แก้ไข**, ส่วน Data frame ที่ไม่มี Marking ก็ถูกส่งต่อด้วย DSCP Default เช่นเดิม (ไม่ต้อง
re-mark เพราะ CS0 คือค่า default อยู่แล้ว) — เมื่อทั้งสอง frame ไปถึง Queue ขาออกที่ Uplink
Gi1/0/24 ระบบ Queue ที่ Auto-QoS ตั้งไว้จะดู CoS/DSCP นี้แล้วส่ง Voice frame ผ่าน **PRIORITY
Queue** ก่อน Data frame ที่อยู่ใน **WRR Queue** เสมอเมื่อเกิด Congestion — ทำให้ Voice call
ไม่สะดุดแม้ตอนที่ PC หลายเครื่องกำลังโหลดข้อมูลหนักพร้อมกัน

### สรุป Lab Step 240

| รายการที่ทำ | คำสั่งหลัก | ผลลัพธ์ |
|---|---|---|
| เปิด QoS ระดับ Global | `mls qos` | QoS engine ของ switch เริ่มทำงานจริง |
| Trust Boundary แบบปลอดภัย | `mls qos trust device cisco-phone` | เชื่อ CoS เฉพาะเมื่อ CDP confirm เป็น Cisco Phone จริง |
| Auto-QoS Template | `auto qos voip cisco-phone` | Priority Queue + WRR Queue อื่นถูกสร้างอัตโนมัติ |
| Verify Trust | `show mls qos interface` | ยืนยัน trust state เปลี่ยนเป็น `trust cos` เมื่อ Phone เสียบจริง |
| Verify Queue | `show queueing interface` | ยืนยันมี Queue ประเภท PRIORITY สำหรับ Voice |
| Verify Marking จริง | Packet Capture (SPAN) | Voice = CoS5/DSCP EF, Data = CoS0/DSCP CS0 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 24

1. ตามมาตรฐาน ITU-T/Cisco ค่า One-way Latency, Jitter, และ Packet Loss ที่ยอมรับได้สำหรับ VoIP
   คือเท่าไหร่ตามลำดับ?

2. อธิบายความแตกต่างระหว่าง IntServ และ DiffServ — เพราะเหตุใด DiffServ จึงเป็น Model ที่ใช้จริง
   ในองค์กร/ISP ปัจจุบัน ในขณะที่ IntServ แทบไม่ถูกใช้?

3. DSCP ต่างจาก IP Precedence อย่างไร และค่า DSCP ที่ Cisco แนะนำให้ใช้กับ Voice bearer (RTP)
   คือค่าอะไร (ชื่อและ decimal)?

4. คำสั่ง `mls qos trust cos` กับ `mls qos trust device cisco-phone` ต่างกันอย่างไร และเพราะเหตุใด
   คำสั่งหลังจึงปลอดภัยกว่า?

5. ถ้า config `mls qos trust cos` ที่ interface ไปแล้ว แต่ตรวจสอบด้วย `show mls qos` แล้วพบว่า
   "QoS is disabled globally" จะเกิดผลอย่างไร และต้องแก้ไขด้วยคำสั่งอะไร?

### เฉลย

1. One-way Latency < 150 ms, Jitter < 30 ms, Packet Loss < 1%

2. IntServ ใช้ RSVP จองทรัพยากรแบบ per-flow ทำให้ทุก Router บนเส้นทางต้องเก็บ State ของทุก Flow
   ไว้ใน memory — เมื่อจำนวน flow เพิ่มขึ้นมาก (เป็นล้านๆ ใน ISP Backbone) จะไม่ scale
   ส่วน DiffServ ใช้การ Classify/Mark ที่ Edge แล้วให้แต่ละ Hop ปฏิบัติตาม Per-Hop Behavior (PHB)
   ตาม Class โดยไม่ต้องเก็บ State ต่อ Flow เลย (Stateless) จึง Scale ได้ดีกว่ามาก — นี่คือเหตุผล
   ที่ DiffServ ถูกใช้จริงในทางปฏิบัติทุกที่

3. DSCP มีขนาด 6 bit (0-63, 64 ระดับ) ในขณะที่ IP Precedence มีขนาด 3 bit (0-7, 8 ระดับ) เท่านั้น
   โดย DSCP ถูกออกแบบให้ backward compatible กับ IP Precedence (3 bit บนของ DSCP ตรงกับค่า
   IP Precedence เดิม) ค่า DSCP ที่แนะนำสำหรับ Voice bearer (RTP) คือ **EF (Expedited
   Forwarding) = decimal 46**

4. `mls qos trust cos` เชื่อค่า CoS ของทุก frame ที่เข้ามาที่ port นั้นโดยไม่มีเงื่อนไข (ถ้ามี PC
   ปลอมตัวส่ง CoS สูงมาก็เชื่อทันที) ส่วน `mls qos trust device cisco-phone` เป็น Conditional
   Trust ที่ switch จะตรวจสอบผ่าน CDP ก่อนว่าอุปกรณ์ปลายสายเป็น Cisco IP Phone ตัวจริงหรือไม่
   ถ้าไม่ใช่ (เช่นถอด Phone ออกแล้วเสียบ PC ตรงเข้า port เดิม) switch จะไม่เชื่อ Marking นั้นอีก
   ต่อไป — จึงปลอดภัยกว่าในการป้องกันการปลอมตัวของ Traffic

5. คำสั่ง `mls qos trust cos` ที่ config ไว้ระดับ interface **จะไม่มีผลจริง** (QoS engine ของ
   switch ทั้งหมดถูกปิดอยู่ระดับ Hardware) ต้องแก้ไขด้วยการเปิด QoS ระดับ Global ก่อนด้วยคำสั่ง
   `mls qos` ที่ Global Configuration Mode แล้วคำสั่ง trust ระดับ interface จึงจะเริ่มทำงานจริง

---

## สรุป Part 24

Part นี้ปูพื้นฐาน **QoS ระดับ CCNA** ทั้งหมด: เหตุผลที่ต้องมี QoS เมื่อ Voice/Video/Data มาบรรจบ
กันบนเครือข่ายเดียว (Convergence), นิยามที่แม่นยำของ Bandwidth/Delay/Jitter/Packet Loss พร้อม
เกณฑ์มาตรฐานสำหรับ VoIP, สามแนวคิดของ QoS Model (Best Effort/IntServ/DiffServ) และเหตุผลที่
DiffServ ถูกใช้จริง, ระบบ Classification/Marking ทั้ง Layer 2 (CoS) และ Layer 3 (IP Precedence/
DSCP) พร้อมตารางค่า DSCP มาตรฐาน, แนวคิด Trust Boundary และคำสั่ง `mls qos trust`, ภาพรวม 4 กลุ่ม
กลไก QoS, ฟีเจอร์ Auto-QoS ที่ช่วย deploy ได้เร็ว, แนวคิดพื้นฐานของ Queuing (FIFO vs PQ), และปิดท้าย
ด้วย Lab จริงบน ACCESS-SW1/ACCESS-SW2 ที่ใช้ VLAN 20 (VOICE) และ VLAN 10 (SALES) เป็นตัวอย่าง
พร้อม Verification และการเปรียบเทียบ Marking จริงผ่าน Packet Capture

จำไว้ว่า Part นี้เป็นเพียง**จุดเริ่มต้น**ของ QoS — เมื่อถึงระดับ CCNP ENCOR ใน **Part 40** เราจะ
กลับมาเรื่อง QoS อีกครั้งแบบเจาะลึกเต็มรูปแบบ: Modular QoS CLI (MQC) แบบ manual ทั้งหมด, CBWFQ,
Low-Latency Queuing (LLQ), Traffic Shaping (CB Shaping), Traffic Policing (CB Policing แบบ
single-rate และ dual-rate), และ WRED (Weighted Random Early Detection) สำหรับ Congestion
Avoidance — ทุกกลไกที่ Part นี้แค่ "แนะนำให้รู้จัก" จะถูกสอนให้ config ได้เองอย่างละเอียดที่นั่น

✅ **พร้อมสำหรับ Part 25**: เราจะเข้าสู่ **CCNA Capstone Lab** — Lab สรุปรวมทุกความรู้ตั้งแต่
Part 1-24 (Switching, VLAN, Routing, ACL, NAT, DHCP, FHRP, Security เบื้องต้น, Automation
เบื้องต้น, และ QoS) มาประกอบเป็น Enterprise Network เต็มรูปแบบหนึ่งชุด เพื่อเตรียมความพร้อมก่อน
เข้าสู่เนื้อหาระดับ CCNP ENCOR ตั้งแต่ Part 26 เป็นต้นไป

**ไปต่อ:** [Part 25 — CCNA Capstone Lab →](part-025-ccna-capstone-lab.md)
