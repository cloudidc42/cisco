# Part 5 — STP / RSTP / MSTP
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 41–50 จาก 1000**

> ต่อจาก [Part 4 — Trunking, VTP, DTP](part-004-trunking-vtp-dtp.md) ที่เราทำให้ Trunk เชื่อม
> VLAN ระหว่าง CORE/DIST/ACCESS ได้ครบทุกลิงก์แล้ว Part นี้จะแก้ปัญหาสำคัญที่สุดของการออกแบบ
> Layer 2 ที่มี Redundant Link คือ **Loop** — เราจะเรียนรู้ **Spanning Tree Protocol (STP)**
> ทั้งเวอร์ชันคลาสสิก 802.1D, **Rapid STP (RSTP) 802.1w** ที่ใช้งานจริงในปัจจุบัน, และ
> **Multiple STP (MSTP) 802.1s** สำหรับ Enterprise ขนาดใหญ่ พร้อม config STP เต็มรูปแบบให้
> Lab Topology ของเราทำงานได้อย่างเสถียรและคาดเดาได้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 41 | ทำไมต้องมี STP — Broadcast Storm, MAC Table Instability, Duplicate Frame |
| 42 | STP พื้นฐาน — Bridge ID, Root Bridge Election, Timer, Port State, Convergence |
| 43 | STP Port Role และ STP Cost |
| 44 | Configuring Root Bridge (Primary/Secondary) |
| 45 | RSTP (802.1w) — Port Role/State ใหม่, Edge Port, Link Type |
| 46 | PortFast และ BPDU Guard |
| 47 | Root Guard, Loop Guard, UplinkFast/BackboneFast, BPDU Filter |
| 48 | MSTP (802.1s) — Region, Instance, Mapping VLAN |
| 49 | Troubleshooting STP |
| 50 | Lab เต็มรูปแบบ: ออกแบบและ Config STP ทั้ง Topology |

---

## Step 41 — ทำไมต้องมี STP

### ทบทวน Topology ของเรา (จาก Part 1–4)

Lab Topology หลักของหลักสูตรนี้ถูกออกแบบให้มี **Redundant Link ในทุกชั้น** เพื่อความทนทาน
(High Availability) ซึ่งที่ Layer 2 ล้วนๆ (ก่อนมี Routing เข้ามาช่วยใน Part 10) นี่คือปัญหาใหญ่:

```
                    ┌─────────────┐         ┌─────────────┐
                    │  CORE-SW1   │◄═══════►│  CORE-SW2   │
                    │ (Root ตั้งใจ) │         │ (Secondary) │
                    └──┬───────┬──┘         └──┬───────┬──┘
                       │       └────────────────┼───────┘
                       │       ┌────────────────┘       │
                    ┌──┴───┐ ┌─┴────┐              ┌─────┴─┐ ┌───┴──┐
                    │DIST-1│ │DIST-2│              │DIST-3 │ │DIST-4│
                    └┬────┬┘ └┬────┬┘              └┬────┬─┘ └┬────┬┘
                     │    └───┼────┘                │    └────┼────┘
                     └────┬───┘                     └────┬────┘
                  ┌───────┴───────┐              ┌────────┴──────┐
                  │  ACCESS-SW1   │              │  ACCESS-SW3   │
                  │ (dual-homed   │              │ (dual-homed   │
                  │ ไป DIST-1/2)  │              │ ไป DIST-3/4)  │
                  └───────────────┘              └───────────────┘
```

ทุก DIST switch มีสายไป **ทั้ง CORE-SW1 และ CORE-SW2** และทุก ACCESS switch มีสายไป
**ทั้ง DIST switch คู่ของตัวเอง** (ACCESS-SW1 → DIST-SW1 + DIST-SW2, ACCESS-SW3 → DIST-SW3 +
DIST-SW4) นี่คือรูปแบบ **Redundant Triangle/Square** ที่ทำให้เกิด **Physical Loop** ที่ Layer 2
ในทุกๆ สามเหลี่ยม/สี่เหลี่ยมของ Topology

### ถ้าไม่มี STP จะเกิดอะไรขึ้น — 3 ปัญหาหลัก

**1. Broadcast Storm**

Switch (Layer 2) เมื่อได้รับ Broadcast Frame (Dest MAC = `FFFF.FFFF.FFFF`, เช่น ARP Request)
จะ **flood ออกทุกพอร์ตยกเว้นพอร์ตที่รับเข้ามา** ในโครงสร้างที่มี Loop เช่น
`ACCESS-SW1 → DIST-SW1 → CORE-SW1 → CORE-SW2 → DIST-SW2 → ACCESS-SW1` (วนกลับมาที่เดิม)
Frame เดียวกันจะถูก flood วนไปเรื่อยๆ **ไม่มีวันหมดอายุ** (ไม่มี TTL แบบ Layer 3) จนเต็ม
bandwidth ทุกลิงก์ 100% → เรียกว่า **Broadcast Storm** — ผลคือทั้งเครือข่ายล่ม (CPU ของทุก
switch พุ่งขึ้นไปจัดการ flood traffic จนตอบสนองคำสั่งอื่นไม่ได้)

**2. MAC Address Table Instability**

เมื่อ Frame จาก Host ตัวเดียวกันวิ่งวนเข้ามาที่ switch จากหลายทิศทางพร้อมกัน (เพราะมี Loop)
switch จะเห็น Source MAC เดียวกันโผล่เข้ามาจากพอร์ตที่ต่างกันสลับไปมาอย่างรวดเร็ว
(MAC ปรากฏที่ Gi0/1 แล้วอีกเสี้ยววินาทีปรากฏที่ Gi0/2) ทำให้ MAC Address Table
**flap ตลอดเวลา** (เรียกว่า **MAC Table Instability** หรือ **MAC Address Flapping**)
ผลคือ switch ไม่รู้ว่าจะ forward unicast frame ไปทางไหนแน่ ทำให้เกิดการ flood
unicast แทนที่จะ forward ตรง (ยิ่งซ้ำเติมปัญหา bandwidth)

**3. Duplicate Frame**

เพราะมีเส้นทางมากกว่า 1 เส้นทางไปยังปลายทางเดียวกัน (ผ่าน DIST-SW1 หนึ่งชุด ผ่าน DIST-SW2
อีกชุด) Host ปลายทางอาจได้รับ **Frame เดียวกันซ้ำหลายชุด** (Duplicate Unicast Frame)
ซึ่งทำให้ Application ชั้นบน (เช่น TCP) ต้องจัดการข้อมูลซ้ำซ้อน หรือในกรณีเลวร้ายที่สุด
เมื่อรวมกับ Broadcast Storm และ MAC Instability แล้ว จะทำให้ Network **ใช้งานไม่ได้เลย
ภายในไม่กี่วินาที** หลังจากเปิดสายที่สร้าง Loop

### สรุปตารางปัญหา

| ปัญหา | สาเหตุ | ผลกระทบ |
|---|---|---|
| Broadcast Storm | Broadcast/Multicast/Unknown-Unicast วนไม่รู้จบ | Bandwidth เต็ม 100%, CPU switch พุ่ง, network ล่ม |
| MAC Table Instability | Source MAC เดียวกันเข้ามาจากหลายพอร์ตสลับกัน | Switch ไม่รู้ทางที่แท้จริง เกิด flood แทน forward |
| Duplicate Frame | มีหลายเส้นทางไปปลายทางเดียวกัน | Host ได้รับข้อมูลซ้ำ, Upper layer protocol สับสน |

### ทางแก้: Spanning Tree Protocol (STP)

**STP (IEEE 802.1D)** คือ Protocol ที่ทำงานที่ Layer 2 มีหน้าที่ **ตรวจจับ Loop ทาง Topology
และปิด (Block) พอร์ตส่วนเกินโดยอัตโนมัติ** เพื่อให้เหลือเส้นทางแบบ **Loop-Free Logical
Topology (รูปต้นไม้ — Tree)** ในสภาวะปกติ แต่ยังคง **สาย Physical ทั้งหมดไว้เป็น Backup**
เมื่อลิงก์ที่ใช้งานอยู่ล่ม STP จะ **unblock** ลิงก์สำรองให้กลับมาใช้งานอัตโนมัติ

หลักการสำคัญ: **STP ไม่ได้ลบสายที่ต่อจริง มันแค่ปิดกั้น (Block) ทางตรรกะที่พอร์ตบางพอร์ต
ไม่ให้ forward user traffic แต่ยังคงส่ง/รับ STP control frame (BPDU) เพื่อ monitor
สถานะของ topology ตลอดเวลา**

---

## Step 42 — STP พื้นฐาน: Bridge ID, Root Election, Timer, Port State

### BPDU (Bridge Protocol Data Unit)

Switch ที่รัน STP จะส่ง **BPDU** ออกทุกพอร์ตทุก **2 วินาที (Hello Timer)** เพื่อแลกเปลี่ยน
ข้อมูล Topology กับ switch ข้างเคียง BPDU มาตรฐาน 802.1D เรียกว่า **Configuration BPDU**
มีข้อมูลสำคัญ ได้แก่ Root Bridge ID, Sender Bridge ID, Root Path Cost, Port ID, และ Timer ต่างๆ

### Bridge ID (BID)

ทุก switch มี **Bridge ID** ขนาด 8 byte ใช้ตัดสินว่าใครจะเป็น **Root Bridge**:

```
┌──────────────────────┬──────────────────────┐
│   Bridge Priority     │      MAC Address       │
│      (2 byte)          │        (6 byte)         │
│   ค่า default = 32768  │  MAC ของ switch (unique) │
└──────────────────────┴──────────────────────┘
```

- **Priority**: ค่าเริ่มต้น (default) = `32768` ปรับได้เป็นทวีคูณของ 4096 (0, 4096, 8192, ...,
  61440) — **ยิ่งค่าน้อย ยิ่งมีสิทธิ์เป็น Root Bridge มากกว่า**
- ตั้งแต่ 802.1t (Extended System ID) เป็นต้นมา Priority field 2 byte แบ่งเป็น
  **Priority (4 bit สูงสุด = ทวีคูณ 4096) + System ID Extension (12 bit = VLAN ID)**
  เพื่อให้ทุก VLAN มี Bridge ID ต่างกันแม้ใช้ MAC เดียวกัน (สำหรับ PVST+)
- **MAC Address**: ใช้ตัดสินเมื่อ Priority เท่ากัน (ยิ่ง MAC น้อย ยิ่งชนะ) — เป็นเหตุผลว่าทำไม
  **switch รุ่นเก่าที่มี MAC ต่ำ อาจถูกเลือกเป็น Root Bridge โดยไม่ตั้งใจ** ถ้าไม่ได้ config
  priority ไว้ล่วงหน้า (ปัญหานี้จะพูดถึงลึกใน Step 49)

### ขั้นตอน Root Bridge Election

1. เริ่มต้น **ทุก switch คิดว่าตัวเองเป็น Root Bridge** และส่ง BPDU ที่มี Root BID = BID ตัวเอง
2. เมื่อ switch ได้รับ BPDU ที่มี Root BID **ต่ำกว่า** ของตัวเอง จะเปลี่ยนไปเชื่อ Root BID ใหม่นั้น
   และส่งต่อ (forward) BPDU นั้นออกไปพร้อมอัปเดต Root Path Cost
3. ทำซ้ำจนทั้งเครือข่ายเห็นพ้องกันว่า **switch ที่มี Bridge ID ต่ำสุดในทั้ง Layer 2 Domain
   คือ Root Bridge** — ในหลักสูตรนี้เราจะ config ให้ **CORE-SW1 เป็น Root Bridge เสมอ**
   ด้วยการตั้ง Priority ต่ำสุด (ดู Step 44)

### STP Timer มาตรฐาน (802.1D)

| Timer | ค่า Default | ความหมาย |
|---|---|---|
| **Hello Timer** | 2 วินาที | ความถี่ที่ Root Bridge ส่ง Configuration BPDU |
| **Max Age (Age Timer)** | 20 วินาที | เวลาที่ non-root switch จะรอก่อนตัดสินว่า BPDU จาก neighbor หายไป (Root หรือ link ล่ม) |
| **Forward Delay** | 15 วินาที | เวลาที่พอร์ตต้องอยู่ใน Listening และ Learning state (state ละ 15 วิ) ก่อนเข้าสู่ Forwarding |

> Timer เหล่านี้ถูกกำหนดค่าไว้ที่ **Root Bridge เท่านั้น** แล้วกระจาย (propagate) ผ่าน BPDU
> ไปยัง switch ทุกตัวในเครือข่าย — ห้าม config Timer ต่างกันในแต่ละ switch เพราะจะทำให้
> STP คำนวณผิดพลาดได้

### 5 Port State ของ 802.1D Classic STP

| State | คำอธิบาย | Forward User Data? | Learn MAC? | ระยะเวลา |
|---|---|---|---|---|
| **Disabled** | พอร์ต shutdown (administratively down) | ไม่ | ไม่ | - |
| **Blocking** | รับ BPDU เพื่อ monitor เท่านั้น | ไม่ | ไม่ | จนกว่าจะเปลี่ยน role |
| **Listening** | เริ่มมีส่วนร่วมในการเลือก Root/Port Role | ไม่ | ไม่ | 15 วินาที (Forward Delay) |
| **Learning** | เริ่มเรียนรู้ MAC Address เข้า MAC table | ไม่ | ใช่ | 15 วินาที (Forward Delay) |
| **Forwarding** | ทำงานปกติเต็มรูปแบบ | ใช่ | ใช่ | ตลอดจนกว่า topology เปลี่ยน |

> โดยทั่วไปเมื่อพูดถึง "3 port state หลัก" ของ 802.1D จะหมายถึง **Blocking → Listening
> → Learning → Forwarding** (ไม่นับ Disabled ที่เป็น administrative state) การเปลี่ยนผ่าน
> ทุก state (ยกเว้นจาก Blocking→Listening ซึ่งไม่มี Timer บังคับ) ใช้เวลารวมตามสูตร:

```
Convergence Time (Classic 802.1D) ≈ Max Age (20s) + 2×Forward Delay (2×15s) = 50 วินาที
```

**นี่คือจุดอ่อนใหญ่ที่สุดของ 802.1D** — เมื่อลิงก์ที่กำลังใช้งานอยู่ล่ม ต้องรอถึง **~50 วินาที**
กว่าลิงก์สำรองจะ converge เป็น Forwarding ทำให้ Application สำคัญ (VoIP, Video Call) หลุดได้ง่าย
นี่คือเหตุผลที่อุตสาหกรรมพัฒนา **RSTP (802.1w)** ขึ้นมาแทนที่ (ดู Step 45)

### ตัวอย่างคำสั่งดู STP เบื้องต้น

```
CORE-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    24586
             Address     0019.aa11.2233
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24586  (priority 24576 sys-id-ext 10)
             Address     0019.aa11.2233
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1              Desg FWD 4         128.1    P2p
Gi1/0/2              Desg FWD 4         128.2    P2p
```

---

## Step 43 — STP Port Role และ STP Cost

### Port Role ทั้ง 3 ประเภทใน 802.1D

| Port Role | ความหมาย | อยู่ที่ switch ใด |
|---|---|---|
| **Root Port (RP)** | พอร์ตที่มี **Cost ต่ำสุด** ไปยัง Root Bridge — มีได้ **1 พอร์ตต่อ switch เท่านั้น** (ยกเว้น Root Bridge เองที่ไม่มี Root Port) | ทุก switch ที่ไม่ใช่ Root |
| **Designated Port (DP)** | พอร์ตที่ forward traffic ให้กับ **segment** นั้นๆ — มีได้ **1 พอร์ตต่อ segment (สายหรือ collision domain)** | ทุกพอร์ตของ Root Bridge เป็น DP เสมอ + พอร์ตอื่นที่ชนะการแข่งขันในแต่ละ segment |
| **Blocking Port (Non-Designated)** | พอร์ตที่ **ไม่ forward** user traffic เพื่อตัด Loop — ยังคงรับ BPDU เพื่อ monitor | switch ที่มีสายซ้ำซ้อนไปยัง segment เดียวกัน |

### วิธีตัดสิน Port Role (ลำดับการเปรียบเทียบ 4 ขั้นตอน)

เมื่อ switch มีหลายพอร์ตแข่งกัน STP จะตัดสินตามลำดับนี้ (ถ้าขั้นก่อนหน้าเท่ากันจึงดูขั้นถัดไป):

1. **Lowest Root Bridge ID** — ตัดสินว่าใครคือ Root (ทั้งเครือข่าย)
2. **Lowest Root Path Cost** — ผลรวม Cost ของทุกลิงก์จาก switch นั้นไปถึง Root — ตัดสิน Root Port
3. **Lowest Sender Bridge ID** — เมื่อ Cost เท่ากัน ให้เชื่อ switch ที่มี BID ต่ำกว่าเป็น Designated
4. **Lowest Sender Port ID** — เมื่อทุกอย่างเท่ากัน (เช่น switch เดียวกันมี 2 สายขนานไป segment
   เดียวกัน) ใช้ Port ID (Priority.PortNumber) ต่ำสุดตัดสิน

### STP Cost ตามความเร็วลิงก์ (ค่ามาตรฐาน IEEE — ใช้จริงในหลักสูตรนี้)

| ความเร็วลิงก์ | STP Cost (ค่า Cisco default, IEEE ปรับปรุงปี 1998+) | STP Cost (ค่าเก่า 802.1D ดั้งเดิม) |
|---|---|---|
| 10 Mbps | **100** | 100 |
| 100 Mbps (Fast Ethernet) | **19** | 10 |
| 1 Gbps (GigabitEthernet) | **4** | 1 |
| 10 Gbps (TenGigabitEthernet) | **2** | 1 |
| 100 Gbps | 2 (ต่ำสุดตามมาตรฐานใหม่) | - |

> Cisco IOS ใช้ค่า **"short" path cost method (ค่า default บนอุปกรณ์ปัจจุบัน)** ตามตารางคอลัมน์
> ซ้าย เป็นค่าที่หลักสูตรนี้ใช้ตลอด สามารถดูค่าที่ใช้งานจริงด้วย `show spanning-tree pathcost method`

### ตัวอย่างการคำนวณ Root Path Cost ใน Topology ของเรา

```
CORE-SW1 (Root) ---- Gi (Cost 4) ---- DIST-SW1 ---- Gi (Cost 4) ---- ACCESS-SW1
                                         │
CORE-SW2 --------- Gi (Cost 4) -------- DIST-SW1 (เส้นทางสำรอง)
```

- DIST-SW1 ได้รับ BPDU จาก CORE-SW1 โดยตรง → Root Path Cost = 0 (จาก Root) + 4 (ลิงก์ตัวเอง) = **4**
- DIST-SW1 ยังได้รับ BPDU จาก CORE-SW2 → ต้องคำนวณ Cost ผ่าน CORE-SW2 ไป Root (ผ่าน
  CORE-SW1↔CORE-SW2 อีกที) ซึ่งจะมากกว่า 4 เสมอ (เพราะมีลิงก์เพิ่มอีก 1 เส้น)
- ดังนั้น DIST-SW1 เลือกพอร์ตที่ต่อไป **CORE-SW1 เป็น Root Port** (Cost ต่ำสุด = 4)
  ส่วนพอร์ตที่ต่อไป CORE-SW2 จะกลายเป็น **Blocking Port** (เพราะ Cost สูงกว่า)

### แผนภาพสรุป Port Role ในสามเหลี่ยม CORE–DIST หนึ่งชุด

```
                    ┌────────────┐
                    │  CORE-SW1  │  Root Bridge (ทุกพอร์ต = DP)
                    └──┬──────┬──┘
                   DP  │      │  DP
                       │      │
              Cost=4   │      │  Cost=4
                       │      │
                     RP│      │DP (ชนะ sender BID ต่ำกว่า)
                    ┌──┴──┐ ┌─┴──────┐
                    │DIST1│ │ CORE-SW2│
                    └──┬──┘ └────┬────┘
                       │         │ Blocking (Cost ผ่าน CORE-SW2 มากกว่า RP โดยตรง)
                    DP │      RP/Blk (ตัดสินตาม cost แต่ละ switch)
                       └────┬────┘
                       (segment DIST1-CORE2: ฝั่งใดฝั่งหนึ่ง Block)
```

---

## Step 44 — Configuring STP Root Bridge (Primary/Secondary)

การปล่อยให้ STP เลือก Root Bridge เองเป็นความเสี่ยง (อาจได้ switch รุ่นเก่าที่ MAC ต่ำ
หรือ switch ที่อยู่ไกลสุดของ topology กลายเป็น Root ทำให้เส้นทางไม่เหมาะสม) ดังนั้น
**Best Practice คือต้อง config Root Bridge ให้ชัดเจนเสมอ**

### หลักการออกแบบของหลักสูตรนี้

- **CORE-SW1** = Root Bridge หลัก (Primary) สำหรับ **ทุก VLAN** (10, 20, 30, 40, 99)
- **CORE-SW2** = Root Bridge สำรอง (Secondary) สำหรับ **ทุก VLAN** — ถ้า CORE-SW1 ล่ม
  CORE-SW2 จะกลายเป็น Root ทันที

### วิธีที่ 1 — ใช้คำสั่ง `root primary` / `root secondary` (แนะนำ ง่ายและปลอดภัยที่สุด)

```
! บน CORE-SW1
CORE-SW1(config)# spanning-tree vlan 10,20,30,40,99 root primary
```

คำสั่งนี้ทำให้ IOS **คำนวณ Priority ให้อัตโนมัติ**:
- ตรวจสอบ Priority ของ Root Bridge ปัจจุบันในเครือข่าย (จาก BPDU ที่ได้ยิน)
- ตั้ง Priority ของตัวเองให้ **ต่ำกว่า** ค่าที่ต่ำที่สุดที่เจอ โดยปัดลงเป็นทวีคูณของ 4096
  ที่ใกล้ที่สุด ถ้าค่าที่คำนวณได้ต่ำกว่า 4096 อยู่แล้ว จะตั้งเป็น **4096**
- ถ้าเป็นเครือข่ายใหม่ (ไม่มี switch อื่นประกาศ Root มาก่อน) จะตั้ง Priority = **4096** ทันที

```
! บน CORE-SW2
CORE-SW2(config)# spanning-tree vlan 10,20,30,40,99 root secondary
```

คำสั่งนี้ตั้ง Priority = **28672** (ค่าคงที่ที่ IOS ใช้เสมอสำหรับ secondary) ซึ่งต่ำกว่า
default (32768) แต่สูงกว่า Primary (4096) เสมอ ทำให้ CORE-SW2 เป็นตัวเลือกอันดับ 2

### วิธีที่ 2 — กำหนด Priority เอง (ใช้เมื่อต้องการควบคุมละเอียด เช่นมี Root หลาย tier)

```
CORE-SW1(config)# spanning-tree vlan 10 priority 4096
CORE-SW1(config)# spanning-tree vlan 20 priority 4096
CORE-SW1(config)# spanning-tree vlan 30 priority 4096
CORE-SW1(config)# spanning-tree vlan 40 priority 4096
CORE-SW1(config)# spanning-tree vlan 99 priority 4096

CORE-SW2(config)# spanning-tree vlan 10 priority 8192
CORE-SW2(config)# spanning-tree vlan 20 priority 8192
CORE-SW2(config)# spanning-tree vlan 30 priority 8192
CORE-SW2(config)# spanning-tree vlan 40 priority 8192
CORE-SW2(config)# spanning-tree vlan 99 priority 8192
```

> **ข้อควรระวัง**: Priority ต้องเป็นทวีคูณของ 4096 เท่านั้น (0, 4096, 8192, ..., 61440)
> ถ้าใส่ค่าอื่น IOS จะ reject ทันที:
> ```
> CORE-SW1(config)# spanning-tree vlan 10 priority 5000
> % Bridge Priority must be in increments of 4096.
> % Allowed values are:
> 0 4096 8192 12288 16384 20480 24576 28672 32768 36864 40960 45056 49152 53248 57344 61440
> ```

### Config เต็มรูปแบบสำหรับ CORE-SW1 และ CORE-SW2 (ใช้ `root primary/secondary` — แนวทางหลักสูตร)

```
! ============ CORE-SW1 (Root Primary ทุก VLAN) ============
CORE-SW1(config)# spanning-tree mode rapid-pvst
CORE-SW1(config)# spanning-tree vlan 10,20,30,40,99 root primary
CORE-SW1(config)# spanning-tree vlan 10,20,30,40,99 hello-time 2
CORE-SW1(config)# end
CORE-SW1# write memory

! ============ CORE-SW2 (Root Secondary ทุก VLAN) ============
CORE-SW2(config)# spanning-tree mode rapid-pvst
CORE-SW2(config)# spanning-tree vlan 10,20,30,40,99 root secondary
CORE-SW2(config)# end
CORE-SW2# write memory
```

### ตรวจสอบผลลัพธ์

```
CORE-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0019.aa11.2233
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
             Address     0019.aa11.2233
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec
```

```
DIST-SW1# show spanning-tree vlan 10 | include Root
  Root ID    Priority    4106
             Address     0019.aa11.2233
```

DIST-SW1 (และ switch อื่นทุกตัว) ควรเห็น **Root ID เดียวกัน** คือ Bridge ID ของ CORE-SW1
ในทุก VLAN — ถ้าไม่ตรงกัน แปลว่า config ยังไม่ถูก propagate หรือมี switch อื่นแอบมี priority
ต่ำกว่า (ดู troubleshooting ใน Step 49)

---

## Step 45 — RSTP (802.1w): Rapid Spanning Tree Protocol

### ทำไมต้องใช้ RSTP แทน 802.1D

802.1D มี Convergence Time ~50 วินาทีซึ่งช้าเกินไปสำหรับเครือข่ายสมัยใหม่ **RSTP (IEEE
802.1w)** ถูกออกแบบมาให้ **Converge ได้ในระดับต่ำกว่า 1 วินาทีถึงไม่กี่วินาที** โดยอาศัยกลไก
ใหม่ 3 อย่างหลัก: **Proposal/Agreement Handshake**, **Edge Port**, และ **Alternate/Backup
Port ที่พร้อมใช้งานทันที (ไม่ต้องรอ Listening/Learning)**

### Port Role ใหม่ใน RSTP (เพิ่มจาก 802.1D)

| Port Role (RSTP) | เทียบเท่า 802.1D | ความหมาย |
|---|---|---|
| **Root Port** | Root Port | เหมือนเดิม — เส้นทาง Cost ต่ำสุดไป Root |
| **Designated Port** | Designated Port | เหมือนเดิม — พอร์ตหลักของ segment |
| **Alternate Port** | (รวมอยู่ใน Blocking) | เส้นทางสำรองไป **Root Bridge** — พร้อมเปลี่ยนเป็น Root Port ทันทีถ้า RP เดิมล่ม |
| **Backup Port** | (รวมอยู่ใน Blocking) | เส้นทางสำรองของ **Designated Port บน segment เดียวกัน** (เกิดจาก switch มี 2 สายไป hub/segment เดียวกัน) |

> **Alternate Port** เกิดจากการได้ยิน BPDU ที่ดีกว่าจาก switch **อื่น** (คล้าย Blocking ใน
> 802.1D ที่ DIST-SW1 บล็อกพอร์ตไป CORE-SW2) ส่วน **Backup Port** เกิดขึ้นเมื่อ switch
> ตัวเดียวกันมีสาย 2 เส้นไปยัง segment เดียวกัน (พบได้เมื่อต่อผ่าน Hub เก่า หรือกรณี
> switch ต่อ 2 สายขนานเข้าพอร์ตของอีก switch ตัวเดียวกัน) — ไม่ค่อยพบใน Topology
> จุดต่อจุด (point-to-point) แบบที่หลักสูตรนี้ใช้

### Port State ใหม่ใน RSTP (ลดจาก 5 state เหลือ 3 state)

| RSTP State | เทียบเท่า 802.1D | Forward Data? | Learn MAC? |
|---|---|---|---|
| **Discarding** | รวม Disabled + Blocking + Listening | ไม่ | ไม่ |
| **Learning** | Learning | ไม่ | ใช่ |
| **Forwarding** | Forwarding | ใช่ | ใช่ |

RSTP ยุบ Blocking/Listening/Disabled ให้เหลือ state เดียวคือ **Discarding** เพราะทั้ง 3
กรณีมีพฤติกรรมเดียวกันคือไม่ forward และไม่ learn MAC — ทำให้ state machine ง่ายขึ้นและ
เปลี่ยนผ่านได้เร็วขึ้น

### Edge Port (คือ PortFast ใน RSTP)

**Edge Port** คือพอร์ตที่ต่อกับ **End Device เท่านั้น** (PC, Server, Printer) ไม่มีทาง
เกิด Loop ได้ RSTP จะให้ Edge Port **ข้าม Discarding/Learning ทันที เข้าสู่ Forwarding
โดยไม่ต้องรอ Proposal/Agreement** เทียบเท่ากับการเปิด PortFast ใน 802.1D
(ดูรายละเอียดเต็มใน Step 46)

### Link Type — กลไกสำคัญที่ทำให้ RSTP เร็ว (Proposal/Agreement)

RSTP แบ่งลิงก์เป็น 2 ประเภทตาม Duplex:

| Link Type | เงื่อนไข | รองรับ Rapid Transition? |
|---|---|---|
| **Point-to-Point** | Full-Duplex link ระหว่าง switch กับ switch (ปกติทุกลิงก์ในหลักสูตรนี้) | ✅ ใช้ Proposal/Agreement Handshake ได้ |
| **Shared** | Half-Duplex (ผ่าน Hub เก่า) | ❌ ต้อง fallback เป็นพฤติกรรมแบบ 802.1D (ช้า) |

**Proposal/Agreement Handshake** (กลไกที่ทำให้ Root Port ใหม่ Forward ได้เร็ว):

```
DIST-SW1 (ต้องการเป็น Designated)         ACCESS-SW1 (Downstream)
      │  --- Proposal BPDU (RSTP) --->     │
      │                                     │  (Sync: บล็อก non-edge port อื่นชั่วคราว)
      │  <--- Agreement BPDU ------------   │
      │  [DIST-SW1 พอร์ตนี้ Forward ทันที]  │
```

ทั้งกระบวนการใช้เวลาระดับ **มิลลิวินาทีถึงไม่กี่วินาที** ต่างจาก 802.1D ที่ต้องรอ
Forward Delay 15 วินาที x2 เต็มๆ ไม่ว่ากรณีใด

### Configuring RSTP บน Cisco IOS

Cisco ใช้ชื่อโหมด **Rapid PVST+ (Per-VLAN Rapid Spanning Tree)** ซึ่งรัน RSTP แยก instance
ต่อ VLAN (คล้าย PVST+ ของ 802.1D แต่เร็วกว่า):

```
CORE-SW1(config)# spanning-tree mode rapid-pvst
CORE-SW2(config)# spanning-tree mode rapid-pvst
DIST-SW1(config)# spanning-tree mode rapid-pvst
DIST-SW2(config)# spanning-tree mode rapid-pvst
DIST-SW3(config)# spanning-tree mode rapid-pvst
DIST-SW4(config)# spanning-tree mode rapid-pvst
ACCESS-SW1(config)# spanning-tree mode rapid-pvst
ACCESS-SW2(config)# spanning-tree mode rapid-pvst
ACCESS-SW3(config)# spanning-tree mode rapid-pvst
ACCESS-SW4(config)# spanning-tree mode rapid-pvst
```

> **สำคัญ**: `spanning-tree mode` ต้องตั้งให้ **เหมือนกันทุก switch ใน Layer 2 Domain**
> เดียวกัน ถ้า switch บางตัวยังเป็น `pvst` (802.1D) จะทำให้ลิงก์นั้น fallback เป็น
> compatibility mode (ช้าลง) — Cisco Catalyst รุ่นใหม่ (9200/9300/9500) ส่วนใหญ่ default
> เป็น `rapid-pvst` อยู่แล้ว แต่ควรใส่คำสั่งชัดเจนเสมอเพื่อไม่ให้พลาด

### ตรวจสอบ RSTP

```
DIST-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0019.aa11.2233
             Cost             4
             Port            1 (GigabitEthernet1/0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     001b.cc44.5566
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1              Root FWD 4         128.1    P2p
Gi1/0/2              Altn BLK 4         128.2    P2p
Gi1/0/3              Desg FWD 4         128.3    P2p
Gi1/0/4              Desg FWD 4         128.4    P2p
```

สังเกตคอลัมน์ **Role** ที่เห็น `Altn` (Alternate) แทน `Blk` แบบใน 802.1D และคอลัมน์ **Type**
ที่บอกว่าเป็น `P2p` (Point-to-Point) — ทั้งสองคือคำศัพท์เฉพาะของ RSTP

---

## Step 46 — PortFast และ BPDU Guard

### ปัญหาที่ PortFast แก้

พอร์ตที่ต่อกับ **End Device** (PC, Printer, Server, IP Phone) ปกติจะต้องผ่าน
Listening → Learning (15+15 วินาทีใน 802.1D) ก่อนเข้า Forwarding ทุกครั้งที่ Host
เปิดเครื่องหรือเสียบสายใหม่ — ปัญหาคือ **PC ที่ทำ DHCP Request หรือ PXE Boot ระหว่างรอ
30 วินาทีนี้อาจ timeout** เพราะพอร์ตยังไม่ forward

**PortFast** สั่งให้พอร์ตที่แน่ใจว่าไม่มีทางเกิด Loop (ต่อ End Device เท่านั้น)
**ข้าม Listening/Learning ทันที เข้าสู่ Forwarding State ทันทีที่ link up**

### คำสั่ง Configure PortFast

```
! แบบรายพอร์ต (แนะนำสำหรับ ACCESS-SW ทุกพอร์ตที่ต่อ PC/Phone)
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 24
ACCESS-SW1(config-if-range)# spanning-tree portfast
%Warning: portfast should only be enabled on ports connected to a single
 host. Connecting hubs, concentrators, switches, bridges, etc... to this
 interface  when portfast is enabled, can cause temporary bridging loops.
 Use with CAUTION

! แบบ global (เปิด default ให้ทุก access port ที่ไม่ใช่ trunk)
ACCESS-SW1(config)# spanning-tree portfast default
```

> **คำเตือนสำคัญ**: **ห้ามเปิด PortFast บนพอร์ตที่ต่อกับ switch ตัวอื่น** เด็ดขาด
> เพราะพอร์ตนั้นจะข้ามการตรวจสอบ Loop ทันที ถ้าบังเอิญมีคนต่อสาย loop เข้าพอร์ต
> PortFast โดยไม่ตั้งใจ (เช่น พนักงานเอาสาย LAN ทั้งสองปลายเสียบเข้า wall jack เดียวกัน)
> จะเกิด **Broadcast Storm ทันที** โดยไม่มี STP มาป้องกัน (ดู Step 49 กรณีศึกษาจริง)

### BPDU Guard — เกราะป้องกัน PortFast

เพราะ PortFast เสี่ยงตามที่กล่าวข้างต้น Cisco จึงมี **BPDU Guard** มาคู่กันเสมอ:
หากพอร์ตที่เปิด PortFast **ได้รับ BPDU เข้ามา** (แปลว่ามี switch/bridge อื่นต่ออยู่
ปลายสาย ซึ่งผิดข้อสันนิษฐานว่าเป็น end-device port) พอร์ตนั้นจะถูก **err-disable**
(ปิดพอร์ตทันทีเพื่อป้องกัน Loop)

```
! แบบรายพอร์ต
ACCESS-SW1(config-if-range)# spanning-tree bpduguard enable

! แบบ global (ทำงานเฉพาะพอร์ตที่เปิด portfast ไว้)
ACCESS-SW1(config)# spanning-tree portfast bpduguard default
```

### ตัวอย่างเหตุการณ์เมื่อ BPDU Guard ทำงาน

```
*Mar  1 10:15:22.123: %SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port
GigabitEthernet1/0/5 with BPDU Guard enabled. Disabling port.
*Mar  1 10:15:22.129: %PM-4-ERR_DISABLE: bpduguard error detected on Gi1/0/5,
putting Gi1/0/5 in err-disable state

ACCESS-SW1# show interfaces GigabitEthernet1/0/5 status
Port      Name               Status       Vlan       Duplex  Speed Type
Gi1/0/5                      err-disabled 10         a-full  a-100 10/100/1000BaseTX
```

### Err-Disable Recovery — ให้พอร์ต Auto-Recover แทนการเข้าไปเปิดเอง

ปกติพอร์ตที่ err-disable ต้อง `shutdown` แล้ว `no shutdown` ด้วยมือ แต่สามารถตั้งให้
**auto-recover อัตโนมัติหลังพักช่วงเวลาหนึ่ง** ได้ (ใช้เมื่อมั่นใจว่าจะแก้ปัญหาที่ต้นเหตุ
เช่น ถอดสาย loop ออกแล้ว):

```
ACCESS-SW1(config)# errdisable recovery cause bpduguard
ACCESS-SW1(config)# errdisable recovery interval 300
```

- `errdisable recovery cause bpduguard` — เปิดให้พอร์ตที่ err-disable จากสาเหตุ
  bpduguard กลับมา `no shutdown` เองอัตโนมัติ
- `errdisable recovery interval 300` — รอ 300 วินาที (5 นาที) ก่อน auto-recover
  (ค่า default cause ที่รองรับคือรวมกันหมด ใช้ timer เดียวกัน)

ตรวจสอบ cause ที่เปิด auto-recovery:

```
ACCESS-SW1# show errdisable recovery
ErrDisable Reason            Timer Status
-----------------------      --------------
udld                          Disabled
bpduguard                     Enabled
security-violation            Disabled
channel-misconfig             Disabled
...
Timer interval: 300 seconds

Interfaces that will be enabled at the next timeout:
Interface       Errdisable reason      Time left(sec)
---------       -----------------      --------------
Gi1/0/5          bpduguard              187
```

---

## Step 47 — Root Guard, Loop Guard, UplinkFast/BackboneFast, BPDU Filter

### Root Guard — ป้องกัน switch แปลกปลอมกลายเป็น Root Bridge

**ปัญหา**: ถ้ามีใครนำ switch ใหม่ (หรือ switch ที่ยังไม่ได้ config priority) มาต่อเข้ากับ
พอร์ตของ DIST switch ที่หันไปทาง ACCESS layer และ switch ใหม่นั้นดันมี Priority ต่ำกว่า
Root ปัจจุบันโดยบังเอิญ (หรือ MAC ต่ำกว่า) → **switch แปลกปลอมจะกลายเป็น Root Bridge ใหม่
ทันที** ทำให้ Topology ทั้งหมดคำนวณ STP ใหม่หมด (Traffic ไหลผิดเส้นทาง อาจทำให้ Network ล่ม
ชั่วขณะ)

**Root Guard** แก้ปัญหานี้: config บนพอร์ตที่ **ไม่ควรมี Root Bridge อยู่ปลายสาย**
(เช่น พอร์ตของ DIST switch ที่หันไปทาง ACCESS switch) — ถ้าพอร์ตนั้นได้รับ Superior BPDU
(ประกาศตัวเป็น Root ที่ดีกว่า) พอร์ตจะเข้าสู่สถานะ **root-inconsistent** (คล้าย blocking)
ทันที แทนที่จะยอมรับเป็น Root ใหม่

```
DIST-SW1(config)# interface range GigabitEthernet1/0/1 - 4
DIST-SW1(config-if-range)# spanning-tree guard root
```

เมื่อเกิดเหตุการณ์จริง:

```
*Mar  1 11:02:10.551: %SPANTREE-2-ROOTGUARD_BLOCK: Root guard blocking port
GigabitEthernet1/0/2 on VLAN0010.

DIST-SW1# show spanning-tree inconsistentports

Name                 Interface           Inconsistency
-------------------- ------------------- ------------------
VLAN0010             GigabitEthernet1/0/2 Root Inconsistent
```

พอร์ตจะกลับมาปกติอัตโนมัติทันทีที่ไม่ได้ยิน Superior BPDU อีกต่อไป (ไม่ต้องรอ err-disable
recovery แบบ BPDU Guard)

### Loop Guard — ป้องกัน Loop จาก Unidirectional Link Failure

**ปัญหา**: Alternate/Root Port ที่กำลัง Block อยู่ อาศัยการ**ได้ยิน BPDU อย่างต่อเนื่อง**
เพื่อรู้ว่ายังต้อง Block อยู่ ถ้าลิงก์เกิด **Unidirectional Failure** (สายส่งได้แต่รับไม่ได้
ทางเดียว — พบได้จาก fiber ที่ทิศทางหนึ่งขาด, GBIC เสีย) พอร์ตจะไม่ได้ยิน BPDU อีกต่อไป
และ **เข้าใจผิดว่าลิงก์อื่นล่มจริง** จึงเปลี่ยนตัวเองจาก Blocking เป็น Designated/Forwarding
→ เกิด **Loop ทันที** ทั้งที่ physical link จริงยังเชื่อมต่ออยู่บางส่วน

**Loop Guard** ป้องกันโดย: ถ้าพอร์ตที่เคยได้ยิน BPDU แล้วจู่ๆ **หยุดได้ยิน BPDU ไปเลย**
(ไม่ใช่ได้ยิน Superior BPDU แบบ Root Guard) พอร์ตจะถูกใส่ในสถานะ **loop-inconsistent**
(บล็อกต่อไป) แทนที่จะเปลี่ยนเป็น Forwarding

```
! แนะนำใส่บนพอร์ต Root Port และ Alternate Port ทั้งหมด (พอร์ตที่ต่อ switch-to-switch)
DIST-SW1(config)# interface range GigabitEthernet1/0/1 - 2
DIST-SW1(config-if-range)# spanning-tree guard loop

! หรือเปิด global ให้ครอบคลุมทุกพอร์ต non-designated อัตโนมัติ (แนะนำในหลักสูตรนี้)
DIST-SW1(config)# spanning-tree loopguard default
```

> **ข้อแตกต่างสำคัญ**: Root Guard ใช้กับพอร์ตที่ **คาดว่าจะเป็น Designated** (ไม่ควรมี Root
> โผล่มาจากทางนั้น) ส่วน Loop Guard ใช้กับพอร์ตที่ **คาดว่าจะเป็น Root/Alternate Port**
> (ไม่ควรหยุดได้ยิน BPDU) — ทั้งสองจึงมักถูกใช้คนละตำแหน่งกันในทาง practice

### UplinkFast และ BackboneFast (คุณสมบัติเก่าของ 802.1D — Cisco Proprietary)

ก่อนที่ RSTP จะแพร่หลาย Cisco สร้างส่วนขยาย proprietary ให้ 802.1D convergence เร็วขึ้น
โดยไม่ต้องเปลี่ยน protocol ทั้งหมด — **ปัจจุบันไม่จำเป็นต้องใช้แล้วถ้า config RSTP
(rapid-pvst) เพราะ RSTP เร็วกว่าและเป็นมาตรฐานสากล** แต่ยังต้องรู้ไว้สำหรับข้อสอบและ
เครือข่ายเก่าที่ยังรัน PVST+ (802.1D):

| Feature | แก้ปัญหา | ความเร็วหลัง fix |
|---|---|---|
| **UplinkFast** | Root Port ล่ม แล้วต้องรอ Listening/Learning เต็ม 30 วิ ก่อน Alternate ขึ้นมาแทน | เปลี่ยนเป็น Forwarding เกือบทันที (~1-3 วิ) โดยข้าม state ปกติ (ใช้ที่ Access/Distribution layer เท่านั้น ไม่ใช้ที่ Root) |
| **BackboneFast** | Indirect Link Failure (ลิงก์ที่ไม่ได้ต่อกับ switch ตัวเองล่ม) ทำให้ต้องรอ Max Age (20 วิ) เต็มก่อนรู้ตัว | ลด Max Age wait เหลือทันที โดยส่ง RLQ (Root Link Query) ถามหา Root โดยตรง |

```
! Config แบบ Legacy (ใช้เฉพาะกรณีเครือข่ายยังรัน PVST+ 802.1D ล้วนๆ)
DIST-SW1(config)# spanning-tree uplinkfast
DIST-SW1(config)# spanning-tree backbonefast
```

> **ในหลักสูตรนี้เราใช้ Rapid PVST+ ทั้งเครือข่าย (Step 45) จึงไม่จำเป็นต้อง config
> UplinkFast/BackboneFast** เพราะ RSTP มีกลไก Proposal/Agreement ที่ทำงานได้ดีกว่าอยู่แล้ว
> ใส่ไว้ในหลักสูตรเพื่อให้เข้าใจประวัติศาสตร์และรู้จักเมื่อเจอเครือข่ายเก่า

### BPDU Filter — ปิดการส่ง/รับ BPDU ทั้งหมดบนพอร์ต (ใช้ระมัดระวังสูงสุด)

**BPDU Filter** ต่างจาก BPDU Guard ตรงที่ **ไม่ err-disable พอร์ต แต่หยุดส่งและหยุดรับ
BPDU ไปเลย** ทำให้พอร์ตนั้น "มองไม่เห็น" STP โดยสิ้นเชิง — อันตรายมากถ้าใช้ผิดที่ เพราะ
ถ้าบังเอิญมี Loop เกิดขึ้นจริงบนพอร์ตนั้น STP จะไม่มีทางรู้และป้องกันได้เลย

```
! แบบรายพอร์ต (ไม่แนะนำ ยกเว้นมีเหตุผลเฉพาะเจาะจงมาก)
Switch(config-if)# spanning-tree bpdufilter enable

! แบบ global (ทำงานเฉพาะพอร์ตที่เปิด portfast — ปลอดภัยกว่าแบบรายพอร์ต)
Switch(config)# spanning-tree portfast bpdufilter default
```

> **คำแนะนำหลักสูตรนี้**: ให้ใช้ **BPDU Guard คู่กับ PortFast เสมอ** (Step 46) และ
> **หลีกเลี่ยง BPDU Filter** ในทุก access port เพราะ BPDU Guard ปลอดภัยกว่ามาก
> (err-disable พอร์ตที่ผิดปกติ ดีกว่าปล่อยให้เกิด Loop เงียบๆ)

### สรุปตาราง STP Protection Feature ทั้งหมด

| Feature | ป้องกันอะไร | ใช้ตรงไหน | ผลเมื่อ trigger |
|---|---|---|---|
| PortFast | Delay การ converge บน edge port | ACCESS port ที่ต่อ end-device | ข้าม Listening/Learning |
| BPDU Guard | Switch/Loop ที่มาต่อผิดที่บน edge port | คู่กับ PortFast เสมอ | Err-disable พอร์ต |
| Root Guard | Switch แปลกปลอมแย่งเป็น Root | พอร์ตหันไปทาง Access/Downstream | root-inconsistent (block ต่อ) |
| Loop Guard | Unidirectional link failure ทำ loop | พอร์ต Root/Alternate (switch-to-switch) | loop-inconsistent (block ต่อ) |
| UplinkFast | Root Port ล่มช้าไป (802.1D เก่า) | Access/Distribution (ไม่ใช้ที่ Root) | Forward เร็วขึ้น (~1-3 วิ) |
| BackboneFast | Indirect failure รอ Max Age นาน (802.1D เก่า) | ทุก switch ที่ไม่ใช่ Root | ข้าม Max Age wait |
| BPDU Filter | ปิด STP ทั้งหมดบนพอร์ต (เสี่ยงสูง) | ใช้เฉพาะกรณีพิเศษเท่านั้น | ไม่ส่ง/รับ BPDU เลย |

---

## Step 48 — MSTP (802.1s): Multiple Spanning Tree Protocol

### ปัญหาของ PVST+/Rapid-PVST+ เมื่อ VLAN เยอะ

**PVST+ (Per-VLAN Spanning Tree Plus)** และ **Rapid-PVST+** รัน **STP Instance แยกกัน
1 instance ต่อ 1 VLAN** — ข้อดีคือ load-balance ได้ละเอียด (แต่ละ VLAN block คนละลิงก์ได้)
แต่ข้อเสียคือถ้ามี VLAN 100 VLAN บน switch เดียว จะมี **100 STP instance รันพร้อมกัน**
ซึ่งกิน CPU และ Memory มาก (แต่ละ instance ต้องส่ง/ประมวลผล BPDU ของตัวเอง)

Topology ของหลักสูตรนี้มี 5 VLAN (10, 20, 30, 40, 99) จึงยังไม่เจอปัญหานี้ชัดเจน แต่ใน
Enterprise ขนาดใหญ่ที่มี VLAN หลักร้อย MSTP คือทางแก้มาตรฐาน

### แนวคิด MSTP: Region + Instance

**MSTP (IEEE 802.1s)** แก้ปัญหาด้วยการ **จัดกลุ่มหลาย VLAN ให้ใช้ STP Instance ร่วมกัน**
แทนที่จะแยก 1 instance ต่อ 1 VLAN:

- **MST Region**: กลุ่มของ switch ที่มี config เดียวกันทุกประการ (Region Name + Revision
  Number + VLAN-to-Instance Mapping) — switch ที่อยู่ Region เดียวกันจะเห็นกันเป็น
  "switch เดียว" ในมุมมองของ Region อื่น (เรียกว่า **Internal Spanning Tree - IST** ภายใน
  และ **Common Spanning Tree - CST** ระหว่าง Region)
- **MST Instance (MSTI)**: กลุ่ม VLAN ที่ map เข้าด้วยกันให้รันบน STP instance เดียว
  เช่น VLAN 10,20 → Instance 1 และ VLAN 30,40,99 → Instance 2 (จากเดิมต้องมี 5 instance
  เหลือเพียง 2 instance)

### ตัวอย่างการออกแบบ MST สำหรับหลักสูตรนี้ (ถ้าเลือกใช้ MSTP แทน Rapid-PVST+)

| MST Instance | VLAN ที่ map | เหตุผล |
|---|---|---|
| **Instance 0 (IST)** | ทุก VLAN ที่ไม่ได้ map ไปที่อื่น (ค่า default) | เป็น instance บังคับที่มีเสมอ |
| **Instance 1** | VLAN 10 (SALES), 20 (VOICE) | กลุ่ม User/Voice traffic — ให้ CORE-SW1 เป็น Root |
| **Instance 2** | VLAN 30 (SERVERS), 40 (WIFI), 99 (MGMT) | กลุ่ม Infra/Server traffic — ให้ CORE-SW2 เป็น Root (load-balance ระหว่าง Core) |

### Configuration MST เต็มรูปแบบ

```
! ต้อง config MST Region ให้ "เหมือนกันทุกตัวอักษร" บนทุก switch ใน Region เดียวกัน
! (Region Name, Revision, VLAN Mapping ต้องตรงกัน 100% ไม่งั้นจะถูกมองเป็นคนละ Region)

CORE-SW1(config)# spanning-tree mode mst
CORE-SW1(config)# spanning-tree mst configuration
CORE-SW1(config-mst)# name CAMPUS-REGION-1
CORE-SW1(config-mst)# revision 1
CORE-SW1(config-mst)# instance 1 vlan 10,20
CORE-SW1(config-mst)# instance 2 vlan 30,40,99
CORE-SW1(config-mst)# show pending

Pending MST configuration
        Name      [CAMPUS-REGION-1]
        Revision  1
        Instance  Vlans mapped
        --------  ---------------------------------------------------------------
        0         1,50-4093,4095
        1         10,20
        2         30,40,99
        --------  ---------------------------------------------------------------

CORE-SW1(config-mst)# exit
CORE-SW1(config)# spanning-tree mst 1 root primary
CORE-SW1(config)# spanning-tree mst 2 root secondary
CORE-SW1(config)# end
```

```
CORE-SW2(config)# spanning-tree mode mst
CORE-SW2(config)# spanning-tree mst configuration
CORE-SW2(config-mst)# name CAMPUS-REGION-1
CORE-SW2(config-mst)# revision 1
CORE-SW2(config-mst)# instance 1 vlan 10,20
CORE-SW2(config-mst)# instance 2 vlan 30,40,99
CORE-SW2(config-mst)# exit
CORE-SW2(config)# spanning-tree mst 2 root primary
CORE-SW2(config)# spanning-tree mst 1 root secondary
CORE-SW2(config)# end
```

> ทำแบบเดียวกันบน **DIST-SW1–4** และ **ACCESS-SW1–4** (mode mst + region config เหมือนกัน
> ทุกตัวอักษร) โดยไม่ต้องตั้ง root primary/secondary ที่ device เหล่านั้น (ให้เป็นแค่
> non-root member ของ region)

### ตรวจสอบ MST

```
CORE-SW1# show spanning-tree mst configuration

Name      [CAMPUS-REGION-1]
Revision  1     Instances configured 3

Instance  Vlans mapped
--------  ---------------------------------------------------------------
0         1,50-4093,4095
1         10,20
2         30,40,99
--------  ---------------------------------------------------------------

CORE-SW1# show spanning-tree mst 1

##### MST1    vlans mapped:   10,20
Bridge        address 0019.aa11.2233  priority      24577 (24576 sysid 1)
Root          this switch for MST1

Interface        Role Sts Cost      Prio.Nbr  Type
---------------- ---- --- --------- --------  --------------------------------
Gi1/0/1          Desg FWD 20000     128.1     P2p
Gi1/0/2          Desg FWD 20000     128.2     P2p
```

### เมื่อไหร่ควรใช้ MSTP แทน Rapid-PVST+

| เกณฑ์ | Rapid-PVST+ | MSTP |
|---|---|---|
| จำนวน VLAN | น้อย (< ~20 VLAN) | มาก (หลักร้อย VLAN) |
| ความละเอียดในการ load-balance ต่อ VLAN | สูงสุด (block ต่างกันได้ทุก VLAN) | จำกัดตามจำนวน instance ที่แบ่ง |
| CPU/Memory Overhead | สูงขึ้นตามจำนวน VLAN | ต่ำ (คงที่ตามจำนวน instance) |
| Interoperability กับ non-Cisco | Cisco proprietary (PVST+) | มาตรฐาน IEEE เปิด ใช้ร่วมกับ vendor อื่นได้ |
| ความซับซ้อนในการ config | ง่าย (ต่อ VLAN ตรงไปตรงมา) | ต้องวางแผน Region/Instance ให้ตรงกันทุก switch |

> หลักสูตรนี้ (5 VLAN) จะยังคงใช้ **Rapid-PVST+** เป็นหลักในการ config Lab ต่อเนื่องต่อไป
> (Step 50 และ Part ถัดๆ ไป) และหยิบ MSTP มาใช้อีกครั้งอย่างละเอียดใน **Part 33 — Advanced
> STP & Campus Multilayer Design**

---

## Step 49 — Troubleshooting STP

### คำสั่ง Verify หลักที่ต้องจำ

```
Switch# show spanning-tree                          ! แสดงทุก VLAN/Instance
Switch# show spanning-tree vlan 10                   ! แสดงเฉพาะ VLAN 10
Switch# show spanning-tree summary                    ! สรุปภาพรวมทุก VLAN
Switch# show spanning-tree interface Gi1/0/1 detail   ! รายละเอียดพอร์ตเดียว
Switch# show spanning-tree root                        ! แสดงเฉพาะข้อมูล Root ของทุก VLAN
Switch# show spanning-tree bridge                       ! แสดง Bridge ID ของตัวเองทุก VLAN
Switch# show spanning-tree inconsistentports            ! แสดงพอร์ตที่ root/loop-inconsistent
Switch# show errdisable recovery                         ! สถานะ auto-recovery
```

### ตัวอย่าง `show spanning-tree summary`

```
CORE-SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0010, VLAN0020, VLAN0030, VLAN0040, VLAN0099
EtherChannel misconfig guard is enabled
Extended system ID           is enabled
Portfast Default              is disabled
PortFast BPDU Guard Default    is disabled
Portfast BPDU Filter Default    is disabled
Loopguard Default                is enabled
UplinkFast                        is disabled
BackboneFast                       is disabled
Pathcost method used                short

Name                   Blocking Listening Learning Forwarding STP Active
----------------------  -------- --------- -------- ---------- ----------
VLAN0010                       0         0        0          2          2
VLAN0020                       0         0        0          2          2
VLAN0030                       0         0        0          2          2
VLAN0040                       0         0        0          2          2
VLAN0099                       0         0        0          2          2
----------------------  -------- --------- -------- ---------- ----------
5 vlans                         0         0        0         10         10
```

### ตัวอย่าง `show spanning-tree interface ... detail`

```
DIST-SW1# show spanning-tree interface GigabitEthernet1/0/2 detail
 Port 2 (GigabitEthernet1/0/2) of VLAN0010 is alternate blocking
   Port path cost 4, Port priority 128, Port Identifier 128.2.
   Designated root has priority 4106, address 0019.aa11.2233
   Designated bridge has priority 32778, address 00c1.dd77.8899
   Designated port id is 128.2, designated path cost 4
   Timers: message age 2, forward delay 0, hold 0
   Number of transitions to forwarding state: 1
   Link type is point-to-point by default
   BPDU: sent 3, received 9021
```

### ปัญหาที่พบบ่อยที่สุดในสนามจริง (3 กรณีศึกษา)

**1. STP Loop เกิดจริง (ไม่มี Loop Guard/Root Guard ป้องกันไว้)**

อาการ: CPU switch พุ่งสูง (ดูด้วย `show processes cpu sorted`), `show interfaces` เห็น
input/output rate เต็ม 100% ทุกพอร์ต, MAC Address Table flap ต่อเนื่อง

```
DIST-SW1# show mac address-table | include 0019.aa11.0001
 10    0019.aa11.0001    DYNAMIC     Gi1/0/1
DIST-SW1# show mac address-table | include 0019.aa11.0001
 10    0019.aa11.0001    DYNAMIC     Gi1/0/3     <!-- MAC เดิมย้ายพอร์ตไปมาเร็วผิดปกติ -->
```

**แนวทางแก้**: ตรวจว่า STP enable ครบทุก switch (`show spanning-tree summary` ต้องไม่ขึ้น
"disabled"), ตรวจว่าไม่มีใครเปิด `bpdufilter` โดยไม่ตั้งใจบนพอร์ต uplink, และเปิด
Loop Guard + Root Guard ตาม Step 47 เพื่อป้องกันไม่ให้เกิดซ้ำ

**2. Root Bridge ถูกเลือกผิดตัวโดยไม่ตั้งใจ (Unintended Root — MAC ต่ำกว่าโดยบังเอิญ)**

อาการ: ตรวจ `show spanning-tree vlan 10` บน CORE-SW1 แล้วพบว่า **"This bridge is the
root" ไม่ปรากฏ** ทั้งที่ตั้งใจให้ CORE-SW1 เป็น Root:

```
CORE-SW1# show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    32778
             Address     0050.ba99.1122      <!-- ไม่ใช่ MAC ของ CORE-SW1 -->
             Cost             4
             Port            3 (GigabitEthernet1/0/3)
```

สาเหตุที่พบบ่อย: มี ACCESS switch รุ่นเก่า (Priority default 32768 แต่ MAC ต่ำกว่า
CORE-SW1 โดยบังเอิญ) หลุดรอดไม่ได้ config priority ไว้ หรือมีคนต่อ switch ใหม่/switch
ทดสอบ (lab switch, unmanaged switch ที่ยังไม่ reset) เข้าไปใน network โดยไม่ผ่านการ
ตรวจสอบ

**แนวทางแก้**: ตรวจสอบทุก switch ด้วย `show spanning-tree vlan 10 | include Bridge ID`
ว่า Priority ตรงตามที่ออกแบบ (Step 44), และเปิด **Root Guard** (Step 47) บนพอร์ตทุกพอร์ต
ที่หันไปทาง Access Layer เพื่อป้องกันไม่ให้เกิดซ้ำอีกในอนาคตแม้จะมี switch แปลกปลอมมาต่อ

**3. เปิด PortFast บนพอร์ตที่ต่อ switch จริง (Human Error) ทำให้เกิด Loop**

อาการ: ผู้ดูแลระบบเข้าใจผิดคิดว่าพอร์ตนั้นต่อ PC จึงเปิด `spanning-tree portfast` ไว้
แต่จริงๆ พอร์ตนั้นต่อกับ **switch สำรอง/switch เก่าที่ยังเสียบสายค้างอยู่** — เมื่อ
เปิดพอร์ตขึ้นมา (link up) พอร์ตจะข้าม Listening/Learning ทันทีเข้า Forwarding
**ก่อนที่ STP จะทันคำนวณ Topology ใหม่** → ถ้าปลายทางสร้าง Loop ขึ้นมาพอดี
(เช่น switch ปลายทางมีสายอีกเส้นวนกลับมาที่ segment เดิม) จะเกิด **Broadcast Storm
ทันทีในเสี้ยววินาที** ก่อนที่ STP ปกติจะทันบล็อกได้

**แนวทางแก้ที่ถูกต้อง**: **เปิด BPDU Guard คู่กับ PortFast เสมอ (ไม่มีข้อยกเว้น)** —
เมื่อ BPDU Guard ตรวจพบ BPDU เข้ามาที่พอร์ต PortFast (แปลว่ามี switch จริงต่ออยู่)
จะ err-disable พอร์ตทันทีก่อนเกิด Loop นี่คือเหตุผลที่ Step 46 เน้นย้ำว่า **ต้องเปิด
คู่กันเสมอ ห้ามเปิดแค่ PortFast อย่างเดียว**

### Checklist Troubleshooting STP แบบเป็นระบบ

1. `show spanning-tree summary` — เช็คโหมด (ต้องเป็น `rapid-pvst` หรือ `mst` ตรงกันทุก switch)
2. `show spanning-tree vlan X` — เช็คว่า Root ID ตรงกับที่ออกแบบไว้ (CORE-SW1)
3. `show spanning-tree vlan X` บนทุก switch — เช็คว่าเห็น **Root ID เดียวกันหมด**
4. `show interfaces status err-disabled` — เช็คพอร์ตที่โดน BPDU Guard ปิดไปแล้ว
5. `show spanning-tree inconsistentports` — เช็ค Root Guard/Loop Guard ที่ trigger อยู่
6. `show logging | include SPANTREE` — ดู log การเปลี่ยน Topology (TOPOLOGY_CHANGE)
   ย้อนหลัง เพื่อดูว่ามี Flapping เกิดขึ้นบ่อยแค่ไหน

---

## Step 50 — Lab เต็มรูปแบบ: STP Design ทั้ง Topology

### เป้าหมายของ Lab นี้

1. **CORE-SW1** = Root Primary ทุก VLAN, **CORE-SW2** = Root Secondary ทุก VLAN
2. ทุก switch ใช้ **spanning-tree mode rapid-pvst**
3. **PortFast + BPDU Guard** บนทุกพอร์ต access (user-facing) ของ ACCESS-SW1–4
4. **Root Guard** บนพอร์ตของ DIST switch ที่หันไปทาง ACCESS layer
5. **Loop Guard** บนพอร์ตของ DIST switch ที่หันไปทาง CORE layer (Root/Alternate Port)
6. Verify ผลลัพธ์สุดท้าย: Root Port / Designated Port / Blocking Port ของแต่ละ switch

### Topology อ้างอิงเต็มรูปแบบ (สาย Physical ทั้งหมด — ยังไม่ผ่าน STP)

```
                         ┌───────────────┐         ┌───────────────┐
                         │   CORE-SW1     │◄═══════►│   CORE-SW2     │
                         │ Pri 4096(+VLAN)│  Po/Trk │ Pri 28672      │
                         └──┬────┬───┬───┘         └──┬───┬────┬───┘
              ┌─────────────┘    │   └──────┐  ┌──────┘   │    └─────────────┐
              │             ┌────┘          │  │          └────┐             │
        ┌─────┴──┐    ┌─────┴──┐      ┌─────┴──┴┐      ┌───────┴─┐    ┌──────┴──┐
        │DIST-SW1│    │DIST-SW2│      │DIST-SW3 │      │DIST-SW4 │    │(สาย     │
        └─┬────┬─┘    └─┬────┬─┘      └─┬─────┬─┘      └─┬─────┬─┘    │ไขว้ครบ  │
          │    └────┬───┘    │          │     └────┬─────┘     │      │ทุกคู่)  │
          │  ┌───────┴────┐  │          │   ┌───────┴────┐     │      └─────────┘
          │  │(DIST1-2    │  │          │   │(DIST3-4     │     │
          │  │ ไขว้กัน)   │  │          │   │ ไขว้กัน)     │     │
          │  └────────────┘  │          │   └─────────────┘     │
    ┌─────┴──────┐     ┌─────┴──────┐  ┌─────┴──────┐    ┌──────┴─────┐
    │ ACCESS-SW1 │     │ ACCESS-SW2 │  │ ACCESS-SW3 │    │ ACCESS-SW4 │
    │(dual->D1/D2)│     │(dual->D1/D2)│  │(dual->D3/D4)│    │(dual->D3/D4)│
    └────────────┘     └────────────┘  └────────────┘    └────────────┘
```

DIST-SW1/DIST-SW2 มีสายไปทั้ง CORE-SW1 และ CORE-SW2 (dual-homed) เช่นเดียวกับ
DIST-SW3/DIST-SW4 — และ ACCESS-SW1 มีสายไปทั้ง DIST-SW1 และ DIST-SW2 (dual-homed)
เช่นเดียวกับ ACCESS-SW2 (ไปยัง DIST-SW1/DIST-SW2), ACCESS-SW3/ACCESS-SW4 (ไปยัง
DIST-SW3/DIST-SW4) — สร้าง Loop ในทุกสามเหลี่ยม/สี่เหลี่ยมตามที่อธิบายไว้ใน Step 41

### Config เต็ม — CORE-SW1

```
CORE-SW1(config)# spanning-tree mode rapid-pvst
CORE-SW1(config)# spanning-tree vlan 10,20,30,40,99 root primary
CORE-SW1(config)# spanning-tree portfast bpduguard default
CORE-SW1(config)# spanning-tree loopguard default
CORE-SW1(config)# end
CORE-SW1# write memory
```

### Config เต็ม — CORE-SW2

```
CORE-SW2(config)# spanning-tree mode rapid-pvst
CORE-SW2(config)# spanning-tree vlan 10,20,30,40,99 root secondary
CORE-SW2(config)# spanning-tree portfast bpduguard default
CORE-SW2(config)# spanning-tree loopguard default
CORE-SW2(config)# end
CORE-SW2# write memory
```

### Config เต็ม — DIST-SW1 (ตัวแทน DIST-SW2/3/4 ใช้ pattern เดียวกัน เปลี่ยนแค่ interface)

```
DIST-SW1(config)# spanning-tree mode rapid-pvst

! พอร์ตขึ้นไปหา CORE (Root/Alternate Port) -> ใส่ Loop Guard
DIST-SW1(config)# interface range GigabitEthernet1/0/1 - 2
DIST-SW1(config-if-range)# description ** UPLINK TO CORE-SW1/CORE-SW2 **
DIST-SW1(config-if-range)# switchport mode trunk
DIST-SW1(config-if-range)# spanning-tree guard loop
DIST-SW1(config-if-range)# exit

! พอร์ตลงไปหา ACCESS-SW (Designated Port เสมอ เพราะ DIST อยู่ใกล้ Root กว่า) -> ใส่ Root Guard
DIST-SW1(config)# interface range GigabitEthernet1/0/3 - 4
DIST-SW1(config-if-range)# description ** DOWNLINK TO ACCESS-SW1/ACCESS-SW2 **
DIST-SW1(config-if-range)# switchport mode trunk
DIST-SW1(config-if-range)# spanning-tree guard root
DIST-SW1(config-if-range)# exit

DIST-SW1(config)# end
DIST-SW1# write memory
```

### Config เต็ม — ACCESS-SW1 (ตัวแทน ACCESS-SW2/3/4 ใช้ pattern เดียวกัน)

```
ACCESS-SW1(config)# spanning-tree mode rapid-pvst

! Uplink ไป DIST-SW1/DIST-SW2 -> เป็น Trunk ธรรมดา ไม่เปิด PortFast/Guard พิเศษ
ACCESS-SW1(config)# interface range GigabitEthernet1/0/23 - 24
ACCESS-SW1(config-if-range)# description ** UPLINK TO DIST-SW1/DIST-SW2 **
ACCESS-SW1(config-if-range)# switchport mode trunk
ACCESS-SW1(config-if-range)# exit

! พอร์ต user-facing ทั้งหมด -> PortFast + BPDU Guard
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 22
ACCESS-SW1(config-if-range)# switchport mode access
ACCESS-SW1(config-if-range)# switchport access vlan 10
ACCESS-SW1(config-if-range)# spanning-tree portfast
ACCESS-SW1(config-if-range)# spanning-tree bpduguard enable
ACCESS-SW1(config-if-range)# exit

ACCESS-SW1(config)# errdisable recovery cause bpduguard
ACCESS-SW1(config)# errdisable recovery interval 300
ACCESS-SW1(config)# end
ACCESS-SW1# write memory
```

> ทำ config ชุดเดียวกัน (ปรับเลข interface และ VLAN ตามจริง) กับ **ACCESS-SW2, ACCESS-SW3,
> ACCESS-SW4** และ **DIST-SW2, DIST-SW3, DIST-SW4**

### ผลลัพธ์ที่คาดหวัง — Logical Topology หลัง STP Converge (RSTP)

```
                       ┌──────────────┐              ┌──────────────┐
                       │  CORE-SW1     │══════════════│  CORE-SW2    │
                       │  ROOT BRIDGE  │  DP ---- RP  │  Secondary   │
                       └──┬────────┬──┘              └──┬────────┬──┘
                       DP │        │DP                DP│        │DP
                          │        └──────────┐   ┌──────┘        │
                       RP │                Blk│   │RP              │Blk (Altn)
                    ┌─────┴──┐          ┌─────┴───┴┐         ┌──────┴──┐
                    │DIST-SW1 │          │ DIST-SW2  │         │DIST-SW3/4│ (pattern เดียวกัน)
                    └──┬───┬─┘          └──┬────┬───┘         └──────────┘
                    DP │   │DP           DP│    │DP
                       │   └────┐    ┌─────┘    │
                    RP │     Blk│    │RP         │Blk
                 ┌─────┴──┐  ┌──┴────┴──┐
                 │ACCESS-1│  │ ACCESS-2  │  (pattern เดียวกันสำหรับ ACCESS-3/4 บน DIST-3/4)
                 └────────┘  └──────────┘
```

**สรุปพฤติกรรม**:
- ทุกพอร์ตของ **CORE-SW1** เป็น **Designated Port (DP)** เสมอ เพราะเป็น Root Bridge
- **CORE-SW2** มี Root Port (RP) ไปทาง CORE-SW1 โดยตรง (Cost ต่ำสุด) ส่วนพอร์ตที่ต่อลงมา
  DIST เป็น DP
- **DIST-SW1** เลือก Root Port ไปทาง **CORE-SW1 โดยตรง** (Cost 4) และ **Block** พอร์ตที่ไป
  CORE-SW2 (Cost ผ่านทางอ้อมสูงกว่า) — เช่นเดียวกับ DIST-SW2/3/4
- **ACCESS-SW1** เลือก Root Port ไปทาง DIST switch ที่ Cost ต่ำสุด (ปกติคือ DIST-SW1 ที่
  ใกล้ Root กว่าเล็กน้อยถ้า topology ไม่สมมาตร หรือใช้ Bridge ID/Port ID ตัดสินถ้า cost
  เท่ากันสนิท) และ **Block** พอร์ตที่ไป DIST-SW2

### ตรวจสอบผลลัพธ์จริงด้วย `show spanning-tree` ทุก switch

```
CORE-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0019.aa11.2233
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
             Address     0019.aa11.2233
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1              Desg FWD 4         128.1    P2p
Gi1/0/2              Desg FWD 4         128.2    P2p
Gi1/0/3              Desg FWD 4         128.3    P2p
Gi1/0/4              Desg FWD 4         128.4    P2p
Po1                  Desg FWD 3         128.48   P2p
```

```
DIST-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0019.aa11.2233
             Cost             4
             Port            1 (GigabitEthernet1/0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     001b.cc44.5566
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1              Root FWD 4         128.1    P2p     <- ไปยัง CORE-SW1
Gi1/0/2              Altn BLK 4         128.2    P2p     <- ไปยัง CORE-SW2 (Blocking)
Gi1/0/3              Desg FWD 4         128.3    P2p     <- ไปยัง ACCESS-SW1 (Root Guard)
Gi1/0/4              Desg FWD 4         128.4    P2p     <- ไปยัง ACCESS-SW2 (Root Guard)
```

```
ACCESS-SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     0019.aa11.2233
             Cost             8
             Port            23 (GigabitEthernet1/0/23)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0022.ee88.99aa
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1              Desg FWD 4         128.1    Edge P2p    <- PortFast (user port)
Gi1/0/2              Desg FWD 4         128.2    Edge P2p    <- PortFast (user port)
...
Gi1/0/23             Root FWD 4         128.23   P2p         <- ไปยัง DIST-SW1
Gi1/0/24             Altn BLK 4         128.24   P2p         <- ไปยัง DIST-SW2 (Blocking)
```

> สังเกตคอลัมน์ **Type** ของพอร์ต user-facing ที่ขึ้น **`Edge P2p`** — ยืนยันว่า PortFast
> ทำงานถูกต้อง (ถ้าขึ้น `P2p` เฉยๆ แปลว่ายังไม่ได้เปิด PortFast บนพอร์ตนั้น ให้ตรวจสอบซ้ำ)

### ตรวจสอบ BPDU Guard และ Root Guard พร้อมกันทั้ง Topology

```
ACCESS-SW1# show interfaces status err-disabled
Port      Name               Status       Vlan       Duplex  Speed Type
                                (ว่างเปล่า = ไม่มีพอร์ตไหน err-disable — สถานะปกติ)

DIST-SW1# show spanning-tree inconsistentports
                                (ว่างเปล่า = ไม่มี root/loop-inconsistent — สถานะปกติ)
```

Lab นี้ถือว่าสำเร็จเมื่อ: (1) ทุก switch เห็น Root ID เดียวกัน = CORE-SW1 ทุก VLAN,
(2) ไม่มีพอร์ตไหน err-disabled หรือ inconsistent ในสภาวะปกติ, (3) เมื่อทดสอบถอดสาย
Uplink หลักของ DIST-SW1 ไป CORE-SW1 ออก จะเห็น Alternate Port (Gi1/0/2 ไปยัง CORE-SW2)
เปลี่ยนเป็น `Root FWD` ภายในไม่กี่วินาที (ทดสอบด้วยการจับเวลา `ping` ต่อเนื่องระหว่าง
ทดสอบถอดสาย ควรเห็น packet loss เพียง 1-3 วินาทีเท่านั้น ต่างจาก ~50 วินาทีถ้ายังใช้
802.1D แบบ classic)

---

## แบบฝึกหัดทวนความเข้าใจ Part 5

1. อธิบาย 3 ปัญหาหลักที่เกิดขึ้นถ้าเครือข่าย Layer 2 ที่มี Redundant Link ไม่ได้รัน STP
2. Bridge ID ประกอบด้วยอะไรบ้าง และ Root Bridge Election ตัดสินจากอะไรเป็นลำดับแรก?
3. STP Cost ของลิงก์ GigabitEthernet (1 Gbps) คือเท่าไหร่ตามค่ามาตรฐานที่ Cisco ใช้ปัจจุบัน
   (short path cost method)?
4. ทำไมต้องเปิด **BPDU Guard** คู่กับ **PortFast** เสมอ ถ้าเปิดแค่ PortFast อย่างเดียวจะเกิด
   ความเสี่ยงอะไร?
5. Root Guard กับ Loop Guard ต่างกันอย่างไร และแต่ละอันควรใส่บนพอร์ตประเภทไหน?

**เฉลย:**

1. (1) **Broadcast Storm** — broadcast frame วนไม่รู้จบจนกิน bandwidth 100% (2) **MAC
   Address Table Instability** — MAC เดียวกันปรากฏสลับพอร์ตอย่างรวดเร็วทำให้ switch
   ตัดสินใจ forward ผิดพลาด (3) **Duplicate Frame** — host ปลายทางได้รับ frame ซ้ำเพราะมี
   หลายเส้นทางไปถึงพร้อมกัน
2. Bridge ID = **Priority (2 byte, default 32768)** + **MAC Address (6 byte)** — Root
   Bridge Election ตัดสินจาก **Priority ต่ำสุดก่อน** ถ้าเท่ากันจึงดู **MAC Address ต่ำสุด**
   เป็นลำดับถัดไป
3. **4** (Cost ตาม short path cost method: 10Mbps=100, 100Mbps=19, 1Gbps=4, 10Gbps=2)
4. เพราะ PortFast ทำให้พอร์ตข้าม Listening/Learning เข้า Forwarding ทันที ถ้าพอร์ตนั้นดัน
   ต่อกับ switch จริง (ไม่ใช่ end-device) โดยไม่ตั้งใจ (Human Error) จะทำให้เกิด Loop ได้
   ทันทีโดยไม่มีการตรวจสอบก่อน — BPDU Guard จะ err-disable พอร์ตทันทีที่ตรวจพบ BPDU
   เข้ามา (แปลว่ามี switch อื่นต่ออยู่จริง) ป้องกัน Loop ก่อนที่จะเกิดขึ้น
5. **Root Guard** ใส่บนพอร์ตที่หันไปทาง Downstream/Access (ไม่ควรมี Root Bridge โผล่มาจาก
   ทางนั้น) — trigger เมื่อได้ยิน **Superior BPDU** ทำให้พอร์ต root-inconsistent
   **Loop Guard** ใส่บนพอร์ต Root/Alternate ที่ต่อไปทาง switch อื่น (คาดว่าจะได้ยิน BPDU
   ต่อเนื่อง) — trigger เมื่อ **หยุดได้ยิน BPDU กะทันหัน** (เช่น unidirectional link
   failure) ทำให้พอร์ต loop-inconsistent (ยังคง block แทนที่จะเปลี่ยนเป็น forwarding)

---

## สรุป Part 5

Part นี้ทำให้ Lab Topology ที่มี Redundant Link เต็มรูปแบบ (CORE↔CORE, DIST↔CORE คู่,
ACCESS↔DIST คู่) ทำงานได้อย่างปลอดภัยโดยไม่เกิด Loop — เราได้เรียนรู้ตั้งแต่ปัญหาพื้นฐาน
ของ Layer 2 Loop, กลไก 802.1D คลาสสิก, การ config Root Bridge อย่างมีการควบคุม
(CORE-SW1 = Primary, CORE-SW2 = Secondary ทุก VLAN), RSTP ที่ทำให้ converge เร็วขึ้นจาก
~50 วินาทีเหลือไม่กี่วินาที, feature ป้องกัน (PortFast, BPDU Guard, Root Guard, Loop Guard),
ไปจนถึง MSTP สำหรับ Enterprise ขนาดใหญ่ และปิดท้ายด้วย Lab เต็มรูปแบบที่ config STP
ครบทั้ง Topology พร้อม verify ผลลัพธ์จริง

ตอนนี้ Layer 2 Topology ของเราเสถียรและปลอดภัยจาก Loop แล้ว **แต่ยังใช้ Bandwidth ของ
Redundant Link ได้ไม่เต็มที่** เพราะ STP ยังคง Block ลิงก์สำรองไว้เฉยๆ (ไม่ได้ใช้งานจริง
จนกว่าลิงก์หลักจะล่ม) — Part ถัดไปเราจะแก้ปัญหานี้ด้วย **EtherChannel** ที่รวมหลายสาย
กายภาพให้กลายเป็นลิงก์เดียวทาง Logical (ทำให้ STP มองเห็นเป็นสายเดียว ไม่ต้อง Block และ
ใช้ Bandwidth ได้เต็มทุกเส้น) พร้อมโปรโตคอลเจรจา **LACP และ PAgP**

**ไปต่อ:** [Part 6 — EtherChannel (LACP/PAgP) →](part-006-etherchannel.md)
