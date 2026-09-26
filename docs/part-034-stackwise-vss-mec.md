# Part 34 — StackWise / VSS / Multichassis EtherChannel (MEC)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 331–340 จาก 1000**

> ต่อจาก [Part 33 — Advanced STP & Campus Multilayer Design](part-033-advanced-stp-campus-design.md)
> ที่ปิดช่องโหว่ Distribution↔Core ด้วย Full-Mesh Routed Uplink + OSPF ECMP — DIST-SW1–4 แต่ละตัว
> ตอนนี้มี **2 เส้นทาง Active จริง** ไปยัง CORE-SW1 และ CORE-SW2 พร้อมกัน แต่ Part 33 เองก็ปิดท้าย
> ด้วยการชี้ช่องโหว่ที่เหลืออยู่ไว้ชัดเจนว่า **"ความซ้ำซ้อนในระดับ Physical Chassis เอง"** ยังไม่ได้แตะ —
> ทุกอย่างที่ทำไปคือ **L3 ECMP** (สองเส้นทางที่ต่างกัน ต้องพึ่ง OSPF ในการเลือก/สลับเส้นทาง) ไม่ใช่
> **EtherChannel ตัวเดียวจริงๆ** เพราะ [Part 6](part-006-etherchannel.md) วางกฎไว้ตั้งแต่ต้นว่า
> Member Link ของ Standard EtherChannel **ต้องจบที่ Switch ตัวเดียวกันเท่านั้น** และทิ้งท้ายไว้ว่า
> การจะให้ Member Link เส้นหนึ่งจบที่ CORE-SW1 อีกเส้นจบที่ CORE-SW2 ได้ใน 1 Port-channel เดียว
> ต้องรอ **"Part 34 — StackWise / VSS / Multichassis EtherChannel"** — **Part นี้คือ Part นั้น**
> เราจะรวม CORE-SW1 และ CORE-SW2 ให้เป็น **Logical Switch ตัวเดียว** ด้วย StackWise Virtual (SVL)
> แล้วเปลี่ยน OSPF ECMP 2 เส้นทางของ Part 33 ให้กลายเป็น **Multichassis EtherChannel (MEC)
> ตัวเดียว** ที่เร็วกว่า ง่ายกว่า และมองไม่เห็นการเปลี่ยนแปลง Topology เลยจากมุมของ Routing Protocol

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 331 | ปัญหาที่ MEC แก้ — ทำไม Standard EtherChannel ข้าม Physical Switch ไม่ได้ |
| 332 | StackWise (Physical Stacking) — Catalyst 9300/9200, Stack Cable, Stack Master Election |
| 333 | StackWise Virtual (SVL) — Catalyst 9500/9600, Virtual Stacking ผ่าน 10G/40G, Dual-Active Detection |
| 334 | VSS (Virtual Switching System) — Catalyst 6500/6800 รุ่นเก่า, Virtual Switch Link (VSL) |
| 335 | Multichassis EtherChannel (MEC) — แปลง CORE-SW1+CORE-SW2 เป็น SVL Pair เต็มรูปแบบ |
| 336 | ผลกระทบต่อการออกแบบ — เทียบ Routed ECMP (Part 33) กับ L3 MEC (Part 34) |
| 337 | Dual-Active Detection (DAD) เจาะลึก — Split-Brain, Fast-Hello Link, BFD, Enhanced PAgP |
| 338 | พฤติกรรมเมื่อเกิด Failure และการ Recovery — Member ล่ม, Chassis ล่มทั้งตัว, Rejoin |
| 339 | Verification/Troubleshooting — `show switch stack-ports`, `show stackwise-virtual`, `show etherchannel summary` |
| 340 | Lab เต็มรูปแบบ: SVL + DAD บน CORE-SW1/CORE-SW2 พร้อม MEC บน DIST-SW1–4 และทดสอบ Chassis Failure |

---

## Step 331 — ปัญหาที่ MEC แก้: ทำไม Standard EtherChannel ข้าม Physical Switch ไม่ได้

### ทบทวนกฎเหล็กจาก Part 6

[Part 6 Step 51](part-006-etherchannel.md) วางกฎไว้ชัดเจนตั้งแต่ต้นหลักสูตร:

> **"EtherChannel ปกติ (Standard/Regular EtherChannel) ทุก Member Link ต้องเชื่อมระหว่าง Switch
> คู่เดียวกันเท่านั้น... จะเอาสายหนึ่งไปจบที่ CORE-SW2 อีกสายไปจบที่ DIST-SW1 แบบ Standard
> EtherChannel ทำไม่ได้"**

และคำเฉลยข้อ 5 ของแบบฝึกหัด Part 6 ก็ทิ้งสัญญาไว้ตรงๆ ว่าเรื่องนี้ "จะเรียนละเอียดใน **Part 34 —
StackWise / VSS / Multichassis EtherChannel**" — เหตุผลเชิงเทคนิคที่กฎนี้มีอยู่คือ **LACP/PAgP
เป็น Protocol ที่ผูกกับ Control Plane ของ Switch ตัวเดียว**: การตัดสินใจว่า Port ไหนจะถูก Bundle,
ตาราง Hashing ที่ใช้เลือก Member Link, และ System ID ที่ใช้เจรจากับฝั่งตรงข้าม — ทั้งหมดนี้เก็บอยู่ใน
**Supervisor/Control Plane ตัวเดียว** ของ Switch ตัวนั้น ถ้า Member Link 2 เส้นไปจบที่ Switch
คนละตัว (CORE-SW1 และ CORE-SW2 ซึ่งมี Control Plane แยกกันคนละตัวโดยสิ้นเชิง) ฝั่ง DIST-SW1
จะเห็น LACP System ID **ไม่ตรงกัน** จากสองปลาย และปฏิเสธการ Bundle ทันที

### ทำไม Part 33 ต้องใช้ L3 ECMP แทน ไม่ใช่ MEC ตรงๆ

Part 33 แก้ปัญหา "Distribution↔Core Single-Homed" ด้วยการเพิ่มลิงก์ Routed ที่สองไปยัง Core
อีกตัว แล้วให้ **OSPF** เป็นผู้จัดการ Redundancy (ECMP) — นี่คือทางออกที่ถูกต้อง ณ เวลานั้น เพราะ
CORE-SW1 และ CORE-SW2 ยังเป็น **2 Control Plane อิสระกัน 100%** (คนละ Router-ID, คนละ
OSPF Process, คนละ CLI Session) ทำให้ Regular EtherChannel ข้ามทั้งสองไม่มีทางเป็นไปได้เลย
ไม่ว่าจะ Config ยังไงก็ตาม

```
Part 33 (L3 ECMP) — 2 Control Plane อิสระ, 2 OSPF Neighbor       Part 34 (MEC) — 1 Control Plane, 1 Neighbor
                                                                   ทั้ง CORE-SW1+CORE-SW2 มองจาก DIST-SW1

        CORE-SW1 (Control Plane A)      CORE-SW2 (Control Plane B)         "CORE" (Control Plane เดียว)
              ▲                                ▲                          ▲              ▲
              │ Te1/1/1                        │ Te1/1/2                  │Te1/1/1       │Te1/1/2
              │ (OSPF Nbr #1)                  │ (OSPF Nbr #2)             └──────┬───────┘
         ┌────┴────┐                                                          Po19 (LACP)
         │DIST-SW1 │◄────────────────────────────────────────┐              1 OSPF Neighbor
         └─────────┘   2 เส้นทาง, ECMP ต้องคำนวณ Hash/SPF    ┘         ┌─────┴─────┐
                                                                        │  DIST-SW1  │
                                                                        └────────────┘
```

### เงื่อนไขที่ทำให้ MEC เป็นไปได้: ต้องมี Control Plane เดียวก่อน

สิ่งที่ Part 34 ต้องทำก่อนสร้าง MEC คือ **รวม Control Plane ของ CORE-SW1 และ CORE-SW2 ให้เหลือ
ตัวเดียว** — เมื่อทำสำเร็จ ทั้งสอง Chassis จะมี LACP System ID เดียวกัน, OSPF Process เดียวกัน,
CLI Session เดียวกัน (Login ที่ใครคนหนึ่งก็เห็นทั้งระบบ) เมื่อถึงจุดนั้น DIST-SW1 จะมองเห็น "CORE"
เป็น **Switch ตัวเดียว** ทำให้ Member Link เส้นหนึ่งไปจบที่ Chassis ทางกายภาพของ CORE-SW1 อีก
เส้นไปจบที่ Chassis ทางกายภาพของ CORE-SW2 **ถูกมองว่าเป็น Switch เดียวกัน** และ Bundle เข้า
Port-channel เดียวได้จริงตามกฎ LACP ปกติ — เทคนิคที่ทำให้ 2 Chassis กลายเป็น 1 Control Plane
มี 3 แบบหลักที่ต้องรู้จัก (Step 332-334) และ EtherChannel ที่เกิดขึ้นตามมาเรียกว่า
**Multichassis EtherChannel (MEC)** (Step 335)

---

## Step 332 — StackWise (Physical Stacking)

### แนวคิด

**StackWise** คือการรวม Switch หลายตัว (Access/Distribution-class เช่น **Catalyst 9300, 9200**)
ให้เป็น Logical Switch เดียวโดยใช้ **สาย Stack เฉพาะทาง (Stack Cable)** เสียบระหว่าง Stack Port
ด้านหลังของแต่ละตัว ต่อเป็นวงแหวน (Ring) — สาย Stack นี้ไม่ใช่สาย Ethernet ปกติ แต่เป็น Bus
ความเร็วสูงมาก (StackWise-480 = 480 Gbps, StackWise-1T = 1 Terabit บน Catalyst 9300 รุ่นใหม่)
ที่ใช้ส่งทั้ง Control Plane Message และ Data Plane Traffic ระหว่าง Member

```
        ┌──────────┐   Stack Cable   ┌──────────┐   Stack Cable   ┌──────────┐
        │ Switch 1  │◄───────────────►│ Switch 2  │◄───────────────►│ Switch 3  │
        │ (Master)  │                 │ (Member)  │                 │ (Member)  │
        └────┬─────┘                 └──────────┘                 └────┬─────┘
             └──────────────────── Stack Cable (ปิดวงแหวน) ──────────────┘

     ผลลัพธ์: 3 Physical Chassis = 1 Logical Switch, 1 Running-Config, 1 Management IP
```

### Stack Master Election

ทุก Stack ต้องมี **Stack Master** ตัวเดียวที่ทำหน้าที่ Control Plane หลัก (รัน CLI, คำนวณ
Routing/STP, Sync Config ไปยังสมาชิกที่เหลือ) — การเลือก Master ใช้ลำดับเกณฑ์ดังนี้ (สูงสุดชนะ):

| ลำดับ | เกณฑ์ | คำอธิบาย |
|---|---|---|
| 1 | **Switch Priority สูงสุด** | ตั้งด้วยคำสั่ง `switch <num> priority <1-15>` (Default = 1) — ค่ายิ่งสูงยิ่งมีสิทธิ์เป็น Master ก่อน |
| 2 | **มี Startup-config ที่มี "ประวัติ" เป็น Master มาก่อน** | ป้องกันการสลับ Master โดยไม่จำเป็นหลัง Reboot |
| 3 | **Software Version ใหม่กว่า/สมบูรณ์กว่า** | Member ที่รัน Image เก่ากว่าจะไม่ถูกเลือกเป็น Master |
| 4 | **MAC Address ต่ำสุด** | Tie-breaker สุดท้ายถ้าเกณฑ์อื่นเท่ากันหมด |

```
DIST-SW1(config)# switch 1 priority 15
DIST-SW1(config)# switch 2 priority 14
```

> **Best Practice**: ตั้ง Priority ของตัวที่ต้องการให้เป็น Master **สูงกว่าตัวอื่นอย่างชัดเจน**
> เสมอ (เช่น 15 กับ 14) เพื่อไม่ให้เกิดการสลับ Master โดยไม่ตั้งใจเมื่อ Reload พร้อมกันทั้งคู่

### คุณสมบัติสำคัญของ StackWise

- **Single Control Plane**: มี Running-config ชุดเดียว, IP Management ชุดเดียว, SSH/Console
  เข้าที่ Master ตัวเดียวเห็นภาพรวมทั้ง Stack
- **Config Sync อัตโนมัติ**: เมื่อ Config ที่ Master เปลี่ยน จะถูก Sync ไปยัง Standby/Member
  ทันที (สำหรับกรณี Master ล่ม Standby จะ Takeover ได้เร็วมากเพราะมี Config พร้อมอยู่แล้ว)
- **Interface Numbering แบบ `<switch#>/<slot>/<port>`**: เช่น `GigabitEthernet1/0/5` หมายถึง
  Switch หมายเลข 1 ในสแตก, `GigabitEthernet2/0/5` หมายถึง Switch หมายเลข 2 — **หลักสูตรนี้ใช้
  รูปแบบนี้มาตั้งแต่ Part 10 กับ DIST-SW (`Te1/1/1`) แม้ยังไม่ได้ Stack จริงก็ตาม** เพราะ Catalyst
  9300 ทุกตัวมี Stack-ready Numbering ติดมาเป็น Default อยู่แล้ว
- **`switch <num> renumber <new-num>`**: เปลี่ยนหมายเลข Switch ใน Stack (มีผลหลัง Reload)

### Verification พื้นฐาน

```
DIST-SW1# show switch
Switch/Stack Mac Address : 0011.2233.0001 - Local Mac Address
Mac persistency wait time: Indefinite
                                             H/W   Current
Switch#  Role     Mac Address     Priority Version  State
------------------------------------------------------------
*1       Active   0011.2233.0001     15     V02     Ready
 2       Standby  0011.2233.0002     14     V02     Ready

DIST-SW1# show switch detail
Switch/Stack Mac Address : 0011.2233.0001 - Local Mac Address
                                             H/W   Current
Switch#  Role     Mac Address     Priority Version  State
------------------------------------------------------------
*1       Active   0011.2233.0001     15     V02     Ready
 2       Standby  0011.2233.0002     14     V02     Ready

           Stack Port Status             Neighbor
Switch#  Port#   Status       Speed       Switch#  Port#
--------------------------------------------------------
  1        1        OK          480G         2         2
  1        2        OK          480G         2         1
  2        1        OK          480G         1         2
  2        2        OK          480G         1         1
```

> **หมายเหตุสำคัญ**: StackWise เหมาะกับ **Access/Distribution Layer** (Catalyst 9200/9300) ที่
> Chassis ตั้งอยู่ **ใกล้กันในตู้ Rack เดียวกัน** เพราะสาย Stack Cable มีระยะทางจำกัดมาก (ปกติ
> ไม่กี่เมตร) — **ไม่เหมาะกับ Core Layer** ที่มักต้องการวาง Chassis คนละ MDF/ห้องกัน (Physical
> Redundancy ระดับห้อง/อาคาร) ซึ่งเป็นเหตุผลที่ **CORE-SW1/CORE-SW2 (Catalyst 9500) ในหลักสูตร
> นี้ใช้ StackWise Virtual (Step 333) แทน ไม่ใช่ StackWise ธรรมดา**

---

## Step 333 — StackWise Virtual (SVL)

### แนวคิด: "Stack เสมือน" ผ่านสาย Fiber ธรรมดา ไม่ใช่สาย Stack เฉพาะทาง

**StackWise Virtual (SVL)** ใช้แนวคิดเดียวกับ StackWise (รวม Control Plane เดียว) แต่ **ไม่ใช้
Stack Cable เฉพาะทาง** — ใช้ Interface Ethernet ธรรมดาความเร็วสูง (10G/40G/100G ขึ้นกับรุ่น)
ที่มีอยู่แล้วบน Chassis มาตั้งเป็น **Stackwise Virtual Link (SVL)** แทน ทำให้ Chassis ทั้งสอง
**ไม่จำเป็นต้องอยู่ใกล้กันเป็นเมตร** เหมือน StackWise ธรรมดา — วางคนละห้อง คนละชั้น หรือคนละ
อาคารในระยะที่ Fiber ไปถึงได้ก็ทำได้ (ข้อจำกัดคือ Latency ของ SVL Link ต้องต่ำมากพอสำหรับ
Control Plane Sync) รองรับบน **Catalyst 9500, 9600** (Core-class) เป็นหลัก

```
                CORE-SW1 (Switch 1)                    CORE-SW2 (Switch 2)
           ┌─────────────────────────┐            ┌─────────────────────────┐
           │  Te1/0/5 ────────────────┼────────────┼──────────── Te2/0/5    │
           │  Te1/0/6 ────────────────┼────────────┼──────────── Te2/0/6    │
           │      (SVL Link 1, 2x10G) │            │  (SVL Link 1, 2x10G)   │
           │                          │            │                        │
           │  Gi0/1 ──────────────────┼────────────┼──────────── Gi2/0/1    │
           │      (Dual-Active         │            │  (Dual-Active          │
           │       Detection Link)     │            │   Detection Link)      │
           └─────────────────────────┘            └─────────────────────────┘
                    ผลลัพธ์: มองจากภายนอกเป็น "CORE" ตัวเดียว, 1 Control Plane, 1 Router-ID
```

### SVL Link ทำหน้าที่อะไรบ้าง (สำคัญกว่าที่คิด — ไม่ใช่แค่ Heartbeat)

SVL Link ไม่ได้ใช้แค่ส่ง Keepalive/Config Sync (Control Plane) แต่ยังต้องรับ **Data Plane
Traffic จริง** ในกรณีที่ Frame เข้ามาที่ Chassis หนึ่ง แต่ Hashing Algorithm ของ MEC (ดู Step
335) เลือก Out-interface ที่อยู่บน **อีก Chassis** — Traffic นั้นต้องวิ่งข้าม SVL Link ไปออกอีกฝั่ง
ก่อน ดังนั้น **ต้องออกแบบ Bandwidth ของ SVL Link ให้เพียงพอ** (ใช้ 2 เส้น 10G ขึ้นไปเสมอใน
Production จริง ไม่ใช่แค่ 1 เส้น) เพื่อไม่ให้ SVL Link กลายเป็น Bottleneck ของทั้งระบบ

### คำสั่ง Config หลัก

```
! ขั้นตอนที่ 1 — เปิดโหมด StackWise Virtual และกำหนด Domain (ต้องตรงกันทั้ง 2 Chassis)
CORE-SW1(config)# stackwise-virtual
CORE-SW1(config-stackwise-virtual)# domain 100
CORE-SW1(config-stackwise-virtual)# exit

! ขั้นตอนที่ 2 — กำหนด Physical Interface ที่จะใช้เป็น SVL Link (ต้องเป็น 10G ขึ้นไป)
CORE-SW1(config)# interface TenGigabitEthernet1/0/5
CORE-SW1(config-if)# stackwise-virtual link 1
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/6
CORE-SW1(config-if)# stackwise-virtual link 1
CORE-SW1(config-if)# exit

! ขั้นตอนที่ 3 — กำหนด Dual-Active Detection Link (แนะนำให้เป็นพอร์ตแยกต่างหาก ความเร็วต่ำก็ได้)
CORE-SW1(config)# interface GigabitEthernet0/1
CORE-SW1(config-if)# stackwise-virtual dual-active-detection
CORE-SW1(config-if)# exit
CORE-SW1(config)# exit
CORE-SW1# write memory
CORE-SW1# reload
```

> **สำคัญ**: ทั้ง `stackwise-virtual link` และ `stackwise-virtual dual-active-detection` มีผล
> "หลัง Reload" เท่านั้น (คล้ายกับการเปลี่ยน Boot Variable) — ต้องตั้งค่าเหมือนกันทั้ง 2 Chassis
> (Domain Number ต้องตรงกัน) แล้ว **Reload พร้อมกันทั้งคู่** เพื่อให้เกิดการ Form SVL Pair ครั้งแรก

### Dual-Active Detection (DAD) — แนะนำเบื้องต้น (เจาะลึกใน Step 337)

DAD Link คือลิงก์แยกต่างหาก (ไม่ใช่ SVL Link) ที่ใช้ตรวจสอบว่า "ถ้า SVL Link หลุดทั้งหมด
Chassis อีกฝั่งยังมีชีวิตอยู่จริงหรือไม่" — จำเป็นมากเพื่อป้องกัน **Split-Brain** (ทั้งสอง Chassis
คิดว่าตัวเองเป็น Active พร้อมกัน) รายละเอียดวิธีการทำงานเต็มรูปแบบอยู่ใน Step 337

### หลังฟอร์ม SVL สำเร็จ — สิ่งที่เปลี่ยนไปทันที

| ก่อน SVL | หลัง SVL |
|---|---|
| CORE-SW1, CORE-SW2 = 2 Console/SSH Session แยกกัน | Login ที่ Active ตัวเดียว เห็นภาพรวมทั้งคู่ (Standby ไม่มี CLI แยก) |
| Interface `Te1/0/1` (บน CORE-SW2) ชนกับ `Te1/0/1` ของ CORE-SW1 | Interface ของ CORE-SW2 ถูก Renumber เป็น `Te2/0/1` โดยอัตโนมัติ (Switch Number 2) |
| OSPF Process/Router-ID 2 ตัว (1.1.1.1 และ 1.1.1.2) | OSPF Process เดียว — Router-ID ของ Chassis Active เท่านั้นที่ Active จริง |
| LACP System ID ต่างกัน (ทำ MEC ไม่ได้) | LACP System ID เดียวกันทั้งคู่ — **MEC ทำได้แล้ว (Step 335)** |

---

## Step 334 — VSS (Virtual Switching System) — รุ่นเก่าบน Catalyst 6500/6800

### แนวคิดเดียวกัน แต่มาก่อน SVL

**VSS (Virtual Switching System)** คือเทคโนโลยีรุ่นก่อนหน้าของ StackWise Virtual แนวคิด
เหมือนกันทุกอย่าง (รวม 2 Chassis เป็น 1 Control Plane, ทำ MEC ได้) แต่ใช้บน **Catalyst 6500/6800**
(Chassis-based, การ์ด Supervisor แยก) ซึ่งเป็น Platform รุ่นเก่าที่ Cisco เริ่มเลิกทำตลาดแล้ว

### ศัพท์เทียบเท่าระหว่าง VSS กับ SVL

| แนวคิด | VSS (Catalyst 6500/6800) | StackWise Virtual (Catalyst 9500/9600) |
|---|---|---|
| ลิงก์เชื่อม Control Plane ระหว่าง Chassis | **VSL (Virtual Switch Link)** | **SVL (Stackwise Virtual Link)** |
| หน่วยจัดกลุ่ม Domain | **Virtual Switch Domain** | **Stackwise Virtual Domain** |
| คำสั่งเปิดโหมด | `switch virtual domain <id>` แล้ว `switch mode virtual` | `stackwise-virtual` แล้ว `domain <id>` |
| กลไกป้องกัน Split-Brain | Fast-Hello Link หรือ **Enhanced PAgP (ePAgP)** ผ่าน MEC เดิม | Dedicated DAD Link หรือ BFD |
| Platform | Catalyst 6500, 6800 (Chassis-based, EOL กำลังใกล้เข้ามา) | Catalyst 9500, 9600 (Fixed/Modular รุ่นใหม่, Cisco แนะนำปัจจุบัน) |
| MEC หลังฟอร์มสำเร็จ | ได้เหมือนกัน | ได้เหมือนกัน |

### ตัวอย่าง Config VSS (สำหรับความรู้/ข้อสอบเท่านั้น — หลักสูตรนี้ไม่ใช้ Platform 6500/6800)

```
! บน Switch ที่จะเป็น Switch 1
Switch(config)# switch virtual domain 10
Switch(config-vs-domain)# switch 1
Switch(config-vs-domain)# exit
Switch(config)# interface range TenGigabitEthernet1/1/1 - 2
Switch(config-if-range)# channel-group 1 mode on
Switch(config-if-range)# no shutdown
Switch(config-if-range)# exit
Switch(config)# interface Port-channel1
Switch(config-if)# switch virtual link 1
Switch(config-if)# exit
Switch(config)# switch convert mode virtual
```

> **หมายเหตุสำคัญสำหรับหลักสูตรนี้**: VSS ถูกนำมาพูดถึงเพื่อ **ความสมบูรณ์ของความรู้ CCNP/CCIE**
> เท่านั้น เพราะยังปรากฏในข้อสอบและองค์กรจำนวนมากที่ยังมี Catalyst 6500/6800 ค้างอยู่ในระบบ —
> Cisco **ไม่แนะนำให้ Design ใหม่ด้วย VSS อีกต่อไป** สำหรับ Deployment ใหม่ทุกกรณีให้ใช้
> **StackWise Virtual บน Catalyst 9500/9600** แทน (ตามที่ Lab ของหลักสูตรนี้ใช้ CORE-SW1/
> CORE-SW2 = Catalyst 9500 มาตั้งแต่ [`00-ip-address-plan.md`](00-ip-address-plan.md)) ตั้งแต่
> Step 335 เป็นต้นไปเราจะใช้ SVL เพียงอย่างเดียวใน Config จริงทั้งหมด

---

## Step 335 — Multichassis EtherChannel (MEC): แปลง CORE-SW1+CORE-SW2 เป็น SVL Pair เต็มรูปแบบ

### สถานะก่อนหน้า (สรุปจาก Part 6 + Part 33)

| องค์ประกอบ | สถานะก่อน Part 34 |
|---|---|
| CORE-SW1 ↔ CORE-SW2 | `Port-channel1` (Gi0/1-2, LACP, **L3 Routed** ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md), 10.255.0.0/30) จาก Part 6 |
| DIST-SW1 ↔ CORE | Te1/1/1→CORE-SW1 (10.255.10.2/30, Part 10) + Te1/1/2→CORE-SW2 (10.255.11.2/30, Part 33) — **2 OSPF Neighbor, ECMP** |
| DIST-SW2/3/4 ↔ CORE | Pattern เดียวกับ DIST-SW1 (ดูตาราง Subnet เต็มใน Part 33 Step 321) |

### แผนผังฉบับ MEC (เป้าหมายของ Part นี้)

```
                       ┌─────────────────────────────────────────┐
                       │        "CORE" (Logical, SVL Domain 100)  │
                       │  CORE-SW1 (Switch 1) ◄══SVL══► CORE-SW2 (Switch 2) │
                       └──┬────┬────┬────┬──────┬────┬────┬────┬─┘
                    Te1/0/1 │ Te1/0/2│Te1/0/3│Te1/0/4│Te2/0/1│Te2/0/2│Te2/0/3│Te2/0/4
                          │        │       │       │      │       │       │
              Po19 ┌──────┘        │       │       │      │       │       │
                    └───────────────────────────────────────────────┘ (Te2/0/3)
              Po29 ┌───────────────┘                               │
                    └───────────────────────────────────────────────┘ (Te2/0/4)
              Po39 ┌───────────────────────┐        ┌───────────────┘ (Te2/0/1)
                    └────────────────────────────────┘
              Po49 ┌────────────────────────────────┐ ┌──────────────┘ (Te2/0/2)
                    └──────────────────────────────────┘
        ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
        │  DIST-SW1  │  │  DIST-SW2  │  │  DIST-SW3  │  │  DIST-SW4  │
        │ Po19 (MEC) │  │ Po29 (MEC) │  │ Po39 (MEC) │  │ Po49 (MEC) │
        └───────────┘  └───────────┘  └───────────┘  └───────────┘
```

**หลักการจับคู่ Member Link ของแต่ละ Po**: นำลิงก์เดิม (Part 10, จบที่ Chassis เดียว) และลิงก์ใหม่
(Part 33, จบที่ Chassis อีกตัว) ของ DIST ตัวเดียวกัน มา Bundle เป็น Po เดียว — **ทุกเส้นที่มีอยู่แล้ว
ถูกใช้ซ้ำทั้งหมด ไม่ต้องเพิ่มสายใหม่แม้แต่เส้นเดียว** เปลี่ยนแค่วิธี "มองเห็น" จาก OSPF ให้กลายเป็น
EtherChannel

| Po (MEC) | DIST | Member 1 (เดิม, Part 10) | Member 2 (ใหม่, Part 33) | Subnet ใหม่ (Part 34) |
|---|---|---|---|---|
| Po19 | DIST-SW1 | Te1/0/1 (CORE-SW1) ↔ Te1/1/1 | Te2/0/3 (CORE-SW2) ↔ Te1/1/2 | 10.255.19.0/30 |
| Po29 | DIST-SW2 | Te1/0/2 (CORE-SW1) ↔ Te1/1/1 | Te2/0/4 (CORE-SW2) ↔ Te1/1/2 | 10.255.29.0/30 |
| Po39 | DIST-SW3 | Te2/0/1 (CORE-SW2) ↔ Te1/1/1 | Te1/0/3 (CORE-SW1) ↔ Te1/1/2 | 10.255.39.0/30 |
| Po49 | DIST-SW4 | Te2/0/2 (CORE-SW2) ↔ Te1/1/1 | Te1/0/4 (CORE-SW1) ↔ Te1/1/2 | 10.255.49.0/30 |

> **หมายเหตุ Subnet Plan**: เอกสารนี้ **ขยาย** [`00-ip-address-plan.md`](00-ip-address-plan.md)
> ต่อจากรูปแบบที่ Part 33 เริ่มไว้ (`10.255.X0.0/30` = ลิงก์เดิม, `10.255.X1.0/30` = ลิงก์ใหม่ Part 33)
> — Part 34 เพิ่มรูปแบบ `10.255.X9.0/30` = **Subnet ของ MEC Port-channel ที่รวม 2 ลิงก์เดิมเป็น
> เส้นทางเดียว** (เลข 9 สื่อถึง "รวมเสร็จสมบูรณ์แล้ว") โดย CORE (Logical) ใช้ `.1` เสมอ และ DIST
> ใช้ `.2` เสมอ ตามรูปแบบเดิมทุกประการ

### Config เต็ม — ยกเลิก Po1 เดิม (Part 6) และตั้ง SVL

Po1 (Part 6, Gi0/1-2 ระหว่าง CORE-SW1↔CORE-SW2 แบบ Routed EtherChannel) **หมดความจำเป็น**
ทันทีที่ทั้งสอง Chassis กลายเป็น Control Plane เดียว เพราะไม่มี "อีกฝั่ง" ให้ Route ไปหาอีกต่อไป
(เป็น Switch ตัวเดียวกันแล้ว) — เราจะยกเลิก Po1 และนำ Physical Port ทั้งสองไปใช้งานใหม่:
`Gi0/1` กลายเป็น **DAD Link**, ส่วน `Gi0/2` เก็บเป็น Spare (สำรองไว้สำหรับ DAD Link ที่ 2 หรือ
Out-of-Band Management ในอนาคต)

```
! ===== ทำบน CORE-SW1 (ก่อน Reload) =====
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# shutdown
CORE-SW1(config-if)# exit
CORE-SW1(config)# no interface Port-channel1

CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# no channel-group 1 mode active
CORE-SW1(config-if-range)# no switchport trunk allowed vlan 10,20,30,40,99
CORE-SW1(config-if-range)# no switchport mode trunk
CORE-SW1(config-if-range)# exit

CORE-SW1(config)# stackwise-virtual
CORE-SW1(config-stackwise-virtual)# domain 100
CORE-SW1(config-stackwise-virtual)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/5
CORE-SW1(config-if)# description ### SVL-LINK-TO-CORE-SW2 ###
CORE-SW1(config-if)# stackwise-virtual link 1
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/6
CORE-SW1(config-if)# description ### SVL-LINK-TO-CORE-SW2 ###
CORE-SW1(config-if)# stackwise-virtual link 1
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface GigabitEthernet0/1
CORE-SW1(config-if)# description ### DUAL-ACTIVE-DETECTION-LINK ###
CORE-SW1(config-if)# stackwise-virtual dual-active-detection
CORE-SW1(config-if)# exit
CORE-SW1(config)# end
CORE-SW1# write memory
CORE-SW1# reload
```

```
! ===== ทำบน CORE-SW2 (ก่อน Reload — Domain Number ต้องตรงกับ CORE-SW1 = 100) =====
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# shutdown
CORE-SW2(config-if)# exit
CORE-SW2(config)# no interface Port-channel1

CORE-SW2(config)# interface range GigabitEthernet0/1 - 2
CORE-SW2(config-if-range)# no channel-group 1 mode active
CORE-SW2(config-if-range)# no switchport trunk allowed vlan 10,20,30,40,99
CORE-SW2(config-if-range)# no switchport mode trunk
CORE-SW2(config-if-range)# exit

CORE-SW2(config)# stackwise-virtual
CORE-SW2(config-stackwise-virtual)# domain 100
CORE-SW2(config-stackwise-virtual)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/5
CORE-SW2(config-if)# description ### SVL-LINK-TO-CORE-SW1 ###
CORE-SW2(config-if)# stackwise-virtual link 1
CORE-SW2(config-if)# exit
CORE-SW2(config)# interface TenGigabitEthernet1/0/6
CORE-SW2(config-if)# description ### SVL-LINK-TO-CORE-SW1 ###
CORE-SW2(config-if)# stackwise-virtual link 1
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface GigabitEthernet0/1
CORE-SW2(config-if)# description ### DUAL-ACTIVE-DETECTION-LINK ###
CORE-SW2(config-if)# stackwise-virtual dual-active-detection
CORE-SW2(config-if)# exit
CORE-SW2(config)# end
CORE-SW2# write memory
CORE-SW2# reload
```

> **สำคัญ**: Reload ทั้งสอง Chassis **พร้อมกัน** (หรือใกล้เคียงกันที่สุด) เพื่อให้ SVL Negotiation
> เกิดขึ้นตั้งแต่ต้น — หลัง Reload เสร็จ CLI ของ CORE-SW2 จะ**หายไป** (ไม่มี Console/SSH แยกอีก
> ต่อไป) และ Interface ของมันทั้งหมดจะถูก Renumber จาก `1/0/x` เป็น `2/0/x` โดยอัตโนมัติ — จาก
> จุดนี้ไปเราจะเรียก Session เดียวที่เหลือว่า `CORE#` (Login ผ่าน Management IP ของ Switch 1 ซึ่ง
> เป็น Active)

### Config MEC บน "CORE" (Logical, หลัง SVL Form สำเร็จ) — สำหรับ DIST-SW1 (Po19)

```
CORE# configure terminal
CORE(config)# interface range TenGigabitEthernet1/0/1 , TenGigabitEthernet2/0/3
CORE(config-if-range)# description ### MEMBER-OF-Po19-MEC-TO-DIST-SW1 ###
CORE(config-if-range)# no switchport
CORE(config-if-range)# channel-group 19 mode active
CORE(config-if-range)# no shutdown
CORE(config-if-range)# exit
CORE(config)# interface Port-channel19
CORE(config-if)# description ### MEC-TO-DIST-SW1 ###
CORE(config-if)# no switchport
CORE(config-if)# ip address 10.255.19.1 255.255.255.252
CORE(config-if)# no shutdown
CORE(config-if)# exit
CORE(config)# router ospf 1
CORE(config-router)# network 10.255.19.0 0.0.0.3 area 0
CORE(config-router)# end
CORE# write memory
```

สังเกตคำสั่ง `interface range TenGigabitEthernet1/0/1 , TenGigabitEthernet2/0/3` — นี่คือหัวใจ
ของ MEC: **Interface หนึ่งอยู่บน Switch 1 (Chassis เดิมของ CORE-SW1) อีก Interface อยู่บน
Switch 2 (Chassis เดิมของ CORE-SW2) แต่ถูก Config พร้อมกันในคำสั่งเดียว และ Bundle เข้า
`channel-group 19` เดียวกันได้จริง** — สิ่งนี้เป็นไปไม่ได้เลยก่อน Step 335 (ก่อน SVL Form สำเร็จ)

Config บน CORE สำหรับ Po29/Po39/Po49 (DIST-SW2/3/4) ใช้ Pattern เดียวกันทุกประการ เปลี่ยนแค่
เลข Interface/Port-channel/Subnet ตามตาราง Step 335 ด้านบน

### Config MEC บน DIST-SW1 (ฝั่ง Distribution — Pattern สำหรับ DIST-SW2-4)

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# no ip address
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# interface TenGigabitEthernet1/1/2
DIST-SW1(config-if)# no ip address
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

DIST-SW1(config)# interface range TenGigabitEthernet1/1/1 - 2
DIST-SW1(config-if-range)# description ### MEMBER-OF-Po19-MEC-TO-CORE ###
DIST-SW1(config-if-range)# channel-group 19 mode active
DIST-SW1(config-if-range)# no shutdown
DIST-SW1(config-if-range)# exit

DIST-SW1(config)# interface Port-channel19
DIST-SW1(config-if)# description ### MEC-TO-CORE (Logical, was CORE-SW1+CORE-SW2) ###
DIST-SW1(config-if)# no switchport
DIST-SW1(config-if)# ip address 10.255.19.2 255.255.255.252
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! ลบ Network Statement เดิมของ Part 10/Part 33 ออก เพราะรวมเป็น Subnet ใหม่แล้ว
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# no network 10.255.10.0 0.0.0.3 area 0
DIST-SW1(config-router)# no network 10.255.11.0 0.0.0.3 area 0
DIST-SW1(config-router)# network 10.255.19.0 0.0.0.3 area 0
DIST-SW1(config-router)# end
DIST-SW1# write memory
```

### Verify ทันทีหลัง Config

```
DIST-SW1# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/1/1(P)   Te1/1/2(P)

DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1         1     FULL/  -        00:00:38    10.255.19.1     Po19
```

**สังเกตความแตกต่างจาก Part 33**: ตอนนี้ DIST-SW1 เห็น OSPF Neighbor เพียง **1 ตัว** (Router-ID
ของ Chassis Active ของ "CORE") ผ่าน Interface **เดียว** (`Po19`) แทนที่จะเป็น 2 Neighbor ผ่าน
2 Interface แยกกันแบบ Part 33 — นี่คือผลลัพธ์ที่ตรงตามชื่อ Step 336 ที่จะอธิบายต่อไป

---

## Step 336 — ผลกระทบต่อการออกแบบ: เทียบ Routed ECMP (Part 33) กับ L3 MEC (Part 34)

### ภาพรวมของการเปลี่ยนแปลง

Part 33 แก้ปัญหา Redundancy ด้วย **L3 ECMP** (2 เส้นทาง, 2 Neighbor, Routing Protocol เป็น
ผู้จัดการ Failover) ส่วน Part 34 แก้ปัญหาเดิมด้วย **L3 MEC** (1 เส้นทางเชิงตรรกะ ที่ข้างในมี 2
Physical Link คนละ Chassis, EtherChannel เป็นผู้จัดการ Failover) — ทั้งสองแบบให้ผลลัพธ์ปลายทาง
เดียวกันคือ **"ไม่มี Single Point of Failure"** แต่กลไกภายในต่างกันโดยสิ้นเชิง

```
Part 33 — Routed ECMP                              Part 34 — MEC (บน SVL)

DIST-SW1                                            DIST-SW1
  │ Te1/1/1 (Nbr #1, cost10) ──► CORE-SW1              │ Po19 (Nbr #1 เดียว, cost10)
  │ Te1/1/2 (Nbr #2, cost10) ──► CORE-SW2              │   ├─ Te1/1/1 ──► CORE (Chassis1)
  │                                                      │   └─ Te1/1/2 ──► CORE (Chassis2)
  2 OSPF Adjacency, SPF ต้องรันเมื่อ Path ใดตาย         1 OSPF Adjacency, EtherChannel
                                                          จัดการ Member ล่มโดย OSPF ไม่รู้ตัวเลย
```

### ตารางเทียบ Pros/Cons โดยละเอียด

| ประเด็น | Part 33 — Routed ECMP | Part 34 — L3 MEC (บน SVL) |
|---|---|---|
| จำนวน OSPF Neighbor ต่อ DIST | 2 | 1 |
| Failover เมื่อ Link เดียวล่ม | ต้องรอ OSPF ลบ Path จาก RIB/FIB (เร็ว แต่ยังต้อง "รู้ตัว" ที่ Layer 3) | EtherChannel ลบ Member ออกจาก Hash Table ทันที — **OSPF ไม่รู้ตัวด้วยซ้ำว่ามีอะไรเปลี่ยน** (Interface Po ยัง Up เหมือนเดิม) |
| ความเร็ว Failover | Sub-second (ตามที่พิสูจน์ใน Part 33 Step 330) | **เร็วกว่าอีก** — ไม่มี Routing Protocol Reconvergence เกี่ยวข้องเลยแม้แต่ก้าวเดียว |
| ความซับซ้อนของ Config | ต้องดูแล `maximum-paths`, Network Statement 2 เส้นต่อ DIST | Network Statement เส้นเดียวต่อ DIST, ไม่ต้องยุ่งกับ ECMP Tuning |
| Dependency ต่อ Software เดียว (Single Point of Software Risk) | **ไม่มี** — CORE-SW1/CORE-SW2 เป็น Control Plane อิสระกัน 100% บั๊ก/ปัญหา Software ของตัวหนึ่งไม่กระทบอีกตัว | **มี** — CORE-SW1/CORE-SW2 กลายเป็น Control Plane เดียว ถ้า SVL/StackWise Software มีบั๊กร้ายแรง (เช่น Crash Loop) อาจกระทบทั้งคู่พร้อมกัน |
| ความชัดเจนของ Control-Plane Separation | ชัดเจนมาก (คนละ Router-ID, คนละ Process, Debug ง่าย) | Control Plane ผูกกันเป็นเนื้อเดียว — Debug ต้องเข้าใจ SVL Role (Active/Standby) เพิ่มเติม |
| Upgrade Software (IOS-XE) | ทำ 1 ตัวได้โดยไม่กระทบอีกตัว (Maintenance Window แยกกันได้) | ต้อง Coordinate ทั้ง Pair (แต่ Cisco รองรับ In-Service Software Upgrade — ISSU บางรุ่น) |
| ใช้ Bandwidth ของทุกเส้นจริงหรือไม่ | ใช้จริงทั้งคู่ (ECMP Active-Active) | ใช้จริงทั้งคู่ (MEC Active-Active ผ่าน LACP Hashing) — **เหมือนกัน** |
| เหมาะกับ Access↔Distribution ด้วยหรือไม่ | ใช้ได้ (แต่ Part 33 focus ที่ Distribution↔Core) | ใช้ได้เหมือนกัน — แนวคิดเดียวกันขยายไปทำ ACCESS-SW dual-home เข้า "Distribution ตัวเดียว" (SVL Pair ของ DIST-SW1+DIST-SW2) ได้ในอนาคต |

### ข้อสรุปสำหรับการเลือกใช้งานจริง (คำแนะนำระดับ CCNP Design)

> **กฎการเลือกที่ใช้ในข้อสอบและงานออกแบบจริง**: ถ้าองค์กรมี Platform ที่รองรับ SVL/VSS อยู่แล้ว
> (Catalyst 9500/9600) และทีมงานมีความชำนาญพอที่จะดูแล Software Dependency ที่เพิ่มขึ้น
> **MEC มักถูกเลือกที่ Distribution↔Core** เพราะ Failover เร็วกว่าและ Config ง่ายกว่าในระยะยาว
> — แต่ถ้าองค์กรให้ความสำคัญกับ **Control-Plane Isolation สูงสุด** (เช่น Environment ที่ Regulator
> กำหนดว่าต้องมี Software Fault Domain แยกกันชัดเจน หรือทีมงานยังไม่มั่นใจกับการดูแล SVL Pair)
> **Routed ECMP แบบ Part 33 ยังเป็นตัวเลือกที่สมบูรณ์แบบและปลอดภัยกว่า** ไม่มีข้อใดผิดทาง
> เทคนิค 100% — เป็นการ Trade-off ระหว่าง **ความเร็ว/ความง่าย (MEC)** กับ **Fault Isolation
> (ECMP)** เท่านั้น

---

## Step 337 — Dual-Active Detection (DAD) เจาะลึก

### ปัญหาที่ DAD แก้: Split-Brain

สมมติ **SVL Link ทั้งหมด (Te1/0/5, Te1/0/6) ขาดพร้อมกัน** แต่ **ทั้ง CORE-SW1 และ CORE-SW2
ยังเปิดเครื่องทำงานปกติทั้งคู่** (ไม่ใช่กรณี Chassis ล่ม — คนละ Case กับ Step 338) — ถ้าไม่มีกลไป
ป้องกันใดๆ สิ่งที่จะเกิดขึ้นคือ:

```
                    SVL Link (Te1/0/5, Te1/0/6) ขาดทั้งคู่ — X ตรงกลาง
        CORE-SW1 (Switch 1)          X          CORE-SW2 (Switch 2)
        เดิม: Standby หรือ Active?              เดิม: Standby หรือ Active?
              │                                        │
        "ไม่เห็น Heartbeat จาก Switch 2 เลย"    "ไม่เห็น Heartbeat จาก Switch 1 เลย"
              │                                        │
        คิดว่า Switch 2 ตายแล้ว                  คิดว่า Switch 1 ตายแล้ว
              │                                        │
        เลื่อนตัวเองเป็น ACTIVE                  เลื่อนตัวเองเป็น ACTIVE ด้วย
              └──────────────► ทั้งคู่เป็น ACTIVE พร้อมกัน = SPLIT-BRAIN ◄──────────────┘
```

**ผลลัพธ์ของ Split-Brain**: ทั้งสอง Chassis ใช้ **IP Address, Router-ID, MAC Address ของ
Port-channel เดียวกันซ้ำกันทั้งคู่** (เพราะ Config ถูก Sync มาเป็นชุดเดียวกันตั้งแต่ก่อน SVL Link
ขาด) เกิด **IP/MAC Duplication** ทันที — OSPF ทั้งสองฝั่งจะพยายาม Advertise Route เดียวกันด้วย
Router-ID เดียวกัน ทำให้ Neighbor ฝั่งอื่น (DIST-SW1-4) สับสนอย่างรุนแรง Traffic อาจถูก Forward
ไปผิด Chassis หรือเกิด Loop ได้ — **นี่คือ Failure Mode ที่ร้ายแรงที่สุดของ SVL/VSS ทั้งระบบ**

### วิธีที่ DAD ตรวจจับและแก้ปัญหา

DAD ทำงานโดยใช้ **ลิงก์ที่ไม่ใช่ SVL Link** (เพราะถ้า SVL Link เองขาด DAD ต้องมีทางอื่นถามยืนยัน)
เพื่อเช็คว่า **"อีกฝั่งตายจริงหรือแค่ SVL Link ขาด"**:

```
ถ้า SVL Link ขาด  แต่ DAD Link (Gi0/1) ยังส่ง/รับ Hello ได้ปกติ
        │
        ▼
  สรุปได้ว่า: "อีก Chassis ยังมีชีวิตอยู่ — แค่ SVL Link ขาดเฉยๆ"
        │
        ▼
  Chassis ที่ตรวจพบ (ปกติคือฝั่งที่กลายเป็น Standby ใหม่ตาม Algorithm ของ IOS-XE)
  จะเข้าสู่ "Recovery Mode": Shutdown ทุก Interface ที่ไม่ใช่ SVL/DAD/Management ทันที
  (รวม MEC Member ทั้งหมดที่อยู่บน Chassis นั้น) เพื่อไม่ให้ Forward Traffic ซ้ำกับฝั่ง Active เดิม
        │
        ▼
  รอจนกว่า SVL Link จะกลับมา Up แล้ว Chassis นั้นจะ Reload ตัวเองอัตโนมัติเพื่อ Rejoin
  เป็น Standby ที่ถูกต้องอีกครั้ง (ไม่ใช่ Active ซ้ำ)
```

### วิธีทำ DAD ที่รองรับ (ต้องรู้จักทั้ง 3 แบบสำหรับข้อสอบ)

| วิธี | ใช้กับ | หลักการ |
|---|---|---|
| **Dedicated Fast-Hello Link** | SVL (แนะนำ Default) | ลิงก์แยกต่างหาก (เช่น `GigabitEthernet0/1` ในหลักสูตรนี้) รัน Protocol Hello ความถี่สูงเฉพาะสำหรับตรวจ Dual-Active — เรียบง่าย เชื่อถือได้ ไม่ต้องพึ่ง Routing Protocol อื่น |
| **BFD (Bidirectional Forwarding Detection)** | SVL (ทางเลือกสำหรับ Design ที่มีลิงก์ L3 อื่นเชื่อมสองฝั่งอยู่แล้ว เช่น WAN Uplink คู่ขนาน) | ใช้ Session BFD ที่วิ่งผ่านลิงก์ L3 อื่นที่มีอยู่แล้ว (ไม่ต้องเดินสายใหม่) ตรวจจับไวมาก (Sub-second) แต่ต้องมี Path L3 อื่นให้ใช้ก่อน |
| **Enhanced PAgP (ePAgP)** | VSS (Legacy, Catalyst 6500/6800 เท่านั้น) | ไม่ต้องมีลิงก์แยก — ฝัง TLV พิเศษไปกับ PAgP Message บน MEC Link ที่มีอยู่แล้ว ถ้า Active ตัวจริงเห็น TLV ของตัวเองไม่ตรงกับที่คาด แสดงว่ามี Active อีกตัวเกิดขึ้น — ใช้ได้เฉพาะ VSS ไม่มีบน SVL |

### Verify ก่อนเกิดปัญหา — ต้องเห็น DAD Link "up" เสมอ

```
CORE# show stackwise-virtual dual-active-detection
Dual-Active-Detection Configuration:
--------------------------------------
Dual-Active-Detection Enabled : Yes
Recovery Reload : Enabled

Switch      Dual-Active-Detection Link                   Status
------      -----------------------------------------      -------
1                GigabitEthernet1/0/1                        up
2                GigabitEthernet2/0/1                        up
```

> **คำเตือนสำคัญ**: ถ้า DAD Link เอง "down" พร้อมกับ SVL Link ก็ตกอยู่ในสถานการณ์เดียวกับ
> Step 338 (Chassis ล่มทั้งตัว) เพราะ Algorithm จะสรุปว่า **ทั้งสองลิงก์ขาดพร้อมกัน = สัญญาณของ
> Chassis Failure จริง ไม่ใช่แค่ SVL Link Failure** — ด้วยเหตุนี้ **DAD Link ต้องเดินสายคนละ
> Physical Path กับ SVL Link เสมอ** (คนละการ์ด LAN, คนละ Conduit ถ้าเป็นไปได้) เพื่อไม่ให้เกิด
> Single Point of Failure ที่ทำให้ทั้ง SVL และ DAD ขาดพร้อมกันจากอุบัติเหตุเดียว (เช่น สาย Fiber
> Bundle เดียวกันถูกตัด)

---

## Step 338 — พฤติกรรมเมื่อเกิด Failure และการ Recovery

### กรณีที่ 1 — Member Link เดียวของ MEC ล่ม (ปกติที่สุด)

```
DIST-SW1 --- Te1/1/1 ---X (ขาด)---> CORE (Chassis1, Te1/0/1)
DIST-SW1 --- Te1/1/2 -------------> CORE (Chassis2, Te2/0/3)   <- ยัง Up
```

```
%LINK-3-UPDOWN: Interface TenGigabitEthernet1/1/1, changed state to down
%EC-5-UNBUNDLE: Interface TenGigabitEthernet1/1/1 left the port-channel Port-channel19

DIST-SW1# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/1/1(D)   Te1/1/2(P)
```

Po19 **ยังคง Up (RU)** เหมือนกับพฤติกรรม EtherChannel ปกติที่ Part 6 สอนไว้ทุกประการ — OSPF
Neighbor ผ่าน `Po19` **ไม่หลุดเลย** เพราะ Interface ระดับ Layer 3 (Po19) ไม่ได้ Down ตาม —
Traffic ทั้งหมดเปลี่ยนไปใช้ `Te1/1/2` ในระดับ Sub-second โดย OSPF ไม่ต้องทำอะไรเลยแม้แต่ก้าวเดียว

### กรณีที่ 2 — Chassis ทั้งตัวล่ม (เช่น CORE-SW2 ไฟดับ/Hardware พัง)

**ข้อแตกต่างสำคัญจาก Step 337**: กรณีนี้ **ทั้ง SVL Link และ DAD Link ขาดพร้อมกันในเวลาเดียวกัน**
(เพราะ Chassis ทั้งตัวตาย ไม่ใช่แค่สายขาด) — Algorithm ของ SVL แยกแยะกรณีนี้จาก Split-Brain
ได้อย่างชัดเจน และ**ไม่มีความเสี่ยง Split-Brain เลย** เพราะ Chassis ที่ตายไปจริงๆไม่สามารถอ้างตัว
เป็น Active ได้อีกต่อไป

```
                    CORE-SW2 (Switch 2) ไฟดับกะทันหัน
        SVL Link (Te1/0/5,6) ขาด  +  DAD Link (Gi0/1) ขาด  (พร้อมกันทั้งคู่)
                          │
                          ▼
        CORE-SW1 (Switch 1) ตรวจพบว่าขาดทั้ง 2 ทาง = Chassis จริงตาย ไม่ใช่แค่ Link
                          │
                          ▼
        CORE-SW1 คงสถานะ Active ต่อไปทันที (ไม่มีอะไรต้อง Promote เพราะเป็น Active อยู่แล้ว)
        MEC Member ที่อยู่บน Chassis 2 (Te2/0/1 - Te2/0/4) หายไปจาก Po ทั้ง 4 ตัวพร้อมกัน
        Member ที่เหลือบน Chassis 1 (Te1/0/1 - Te1/0/4) รับ Traffic ทั้งหมด 100% ต่อทันที
```

```
%STACKMGR-1-STANDBY_LOST: Standby switch 2 has left the stack
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/1 left the port-channel Port-channel39
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/2 left the port-channel Port-channel49
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/3 left the port-channel Port-channel19
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/4 left the port-channel Port-channel29

CORE# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/0/1(P)   Te2/0/3(D)
29     Po29(RU)        LACP     Te1/0/2(P)   Te2/0/4(D)
39     Po39(RU)        LACP     Te1/0/3(P)   Te2/0/1(D)
49     Po49(RU)        LACP     Te1/0/4(P)   Te2/0/2(D)
```

ทั้ง 4 Port-channel **ยัง Up ทุกตัว** — DIST-SW1–4 ทุกตัวยังคงมี OSPF Neighbor เดิมผ่าน Po เดิม
เหมือนไม่มีอะไรเกิดขึ้นเลยในมุมของ Layer 3 (เทียบกับ Part 33 ที่ถ้า CORE-SW1 ทั้งตัวล่ม DIST-SW1
จะต้องเสีย 1 OSPF Neighbor ไปจริงๆ และรอ SPF คำนวณ Path ที่เหลือใหม่ — แม้จะ Sub-second
เหมือนกัน แต่ยังมี "งาน" ให้ OSPF ทำ ในขณะที่ MEC ไม่มีงานให้ OSPF ทำเลยแม้แต่นิดเดียว)

### การ Recovery/Rejoin เมื่อ Chassis ที่ล่มกลับมา

```
! CORE-SW2 เปิดเครื่องกลับมาใหม่ (Power Restore)
1. CORE-SW2 Boot ขึ้นมาในสถานะเดิมที่ Save ไว้ก่อนล่ม (มี stackwise-virtual domain 100
   Config อยู่แล้วจาก Step 335)
2. ตรวจพบ SVL Link (Te1/0/5,6) กลับมา Up และ DAD Link (Gi0/1) กลับมา Up ทั้งคู่
3. เจรจากับ CORE-SW1 (Active) ว่าตัวเองคือ "Switch 2" เดิม — CORE-SW1 ยังคง Active ต่อไป
   (ไม่มีการสลับ Role โดยไม่จำเป็น — Preemption ไม่เกิดขึ้นอัตโนมัติเพื่อความเสถียร)
4. Sync Running-Config ทั้งหมดจาก Active มายัง Standby ใหม่ (Config, VLAN, Route Table
   สำหรับ Display เท่านั้น เพราะ RIB/FIB จริงคำนวณที่ Active)
5. Interface Te2/0/1 - Te2/0/4 กลับมา Up และถูก Bundle เข้า Po39/Po49/Po19/Po29 อัตโนมัติ
   (เหมือนพฤติกรรม EtherChannel ปกติเวลาสาย Cable เสียบกลับ)
```

```
%STACKMGR-1-STANDBY_JOINED: Switch 2 has joined the stack as Standby
%EC-5-BUNDLE: Interface TenGigabitEthernet2/0/3 joined the port-channel Port-channel19

CORE# show switch virtual
Switch Mode : Stackwise Virtual
                     Switch 1        Switch 2
------------------------------------------------
Role                 Active           Standby
Domain Number        100              100
```

ไม่ต้อง Config อะไรเพิ่มเลยหลัง Chassis กลับมา — ระบบ **Self-heal เต็มรูปแบบ** เหมือนหลักการ
EtherChannel/OSPF ที่สอนไว้ตั้งแต่ Part 6/Part 33

---

## Step 339 — Verification/Troubleshooting

### ตารางคำสั่งหลักที่ต้องใช้เป็นประจำ

| คำสั่ง | ใช้กับ | ใช้ตรวจสอบ |
|---|---|---|
| `show switch` | StackWise (Physical) | Role (Active/Standby/Member), Priority, State ของทุก Switch ใน Stack |
| `show switch stack-ports` | StackWise (Physical) | สถานะสาย Stack Cable ระหว่าง Member และ Neighbor Port ที่เชื่อมกัน |
| `show switch virtual` | SVL/VSS | Role (Active/Standby), Domain Number, Switch Number ของแต่ละ Chassis |
| `show switch virtual link` | SVL | สถานะ SVL Link แต่ละเส้น (Up/Down) พร้อม Switch ที่เป็นเจ้าของ |
| `show stackwise-virtual` | SVL | สรุปภาพรวม Domain, Role, จำนวน SVL Link ที่ Configured |
| `show stackwise-virtual dual-active-detection` | SVL | สถานะ DAD Link — **ต้องเห็น "up" ทุก Switch เสมอ** |
| `show etherchannel summary` | MEC (และ EtherChannel ทั่วไป) | สถานะ Bundle ของ Po — สังเกต Member ที่มี Switch Number ต่างกัน (`Te1/0/x` + `Te2/0/x`) ในกลุ่มเดียว = MEC ทำงานจริง |
| `show lacp neighbor` | MEC | System ID ที่ DIST เห็นจาก "CORE" ต้องเป็น **ค่าเดียว** (ไม่ใช่ 2 ค่าที่ต่างกันแบบก่อน SVL) |
| `show ip ospf neighbor` | ทั้งระบบ | ยืนยันว่าเหลือ 1 Neighbor ต่อ MEC Link (ไม่ใช่ 2 แบบ Part 33) |

### ตัวอย่างการอ่านผล `show switch virtual` แบบละเอียด

```
CORE# show switch virtual
Switch Mode : Stackwise Virtual
Switch Virtual Link (SVL) Domain Number : 100

                     Switch 1                Switch 2
--------------------------------------------------------------
Role                 Active                   Standby
Session ID           Local:0   Peer:1         Local:1   Peer:0
```

- **Role = Active/Standby**: ต้องมี Active ตัวเดียวเสมอ ถ้าเห็น "Active" ทั้งสองตัวพร้อมกัน (แม้จะ
  เป็นแค่ชั่วขณะ) = สัญญาณของ Split-Brain ที่ DAD ควรจับได้ก่อนหน้านี้แล้ว
- **SVL Domain Number ต้องตรงกันทั้งคู่**: ถ้าไม่ตรง SVL จะไม่ Form ขึ้นเลยตั้งแต่ต้น (คล้ายกับ
  `auto-cost reference-bandwidth` ที่ต้องตรงกันทุกตัวใน OSPF Domain ตาม Part 33 Step 324)

### ตัวอย่างการอ่านผล `show switch virtual link`

```
CORE# show switch virtual link
Flags: A - Alive, D - Dead, S - Standby, H - Hello not received

Switch      Stackwise Virtual Link Port                  Status
------      ----------------------------------------      ------
1               1                TenGigabitEthernet1/0/5     up
1               1                TenGigabitEthernet1/0/6     up
2               1                TenGigabitEthernet2/0/5     up
2               1                TenGigabitEthernet2/0/6     up
```

ต้องเห็น **ทั้ง 4 บรรทัด "up"** เสมอ (2 เส้นต่อฝั่ง) — ถ้าเหลือแค่เส้นเดียว "up" (อีกเส้น "down")
ถือว่า SVL Link ยังทำงานได้ (Redundant ภายในตัวมันเอง) แต่ Bandwidth ของ SVL ลดลงครึ่งหนึ่ง
ควรรีบซ่อมเส้นที่ขาดก่อนที่เส้นที่เหลือจะขาดตามไปด้วย (ซึ่งจะกลายเป็น Trigger ของ DAD ทันที)

### Checklist การไล่ปัญหา MEC ไม่ Bundle (เรียงตามลำดับที่ควรเช็ค)

1. `show switch virtual` — SVL Form สำเร็จหรือไม่ (Role ต้องมี Active 1 + Standby 1 เท่านั้น)
2. `show switch virtual link` — SVL Link ทุกเส้น "up" หรือไม่ (ถ้า SVL ไม่ Up เลย จะไม่มีทางเห็น
   Interface ของ Switch 2 ในระบบด้วยซ้ำ)
3. `show interfaces status` — Physical Link ของ MEC Member ทั้ง 2 เส้น Up จริงหรือไม่
4. Mode Combination ของ LACP ตรงกันหรือไม่ (`active`/`active` ตาม Part 6 Step 53)
5. `show etherchannel summary` — Flag ของสมาชิกคืออะไร (`P`/`I`/`s`/`D` — อ่านตาม Part 6 Step 59)
6. `show lacp neighbor` — System ID ที่เห็นจากทั้ง 2 Physical Link (ของ Switch 1 และ Switch 2)
   ต้องเป็นค่าเดียวกัน ถ้าไม่ตรง = SVL ยังไม่ Form สำเร็จจริง (แม้ `show switch virtual` อาจดูปกติ)

---

## Step 340 — Lab เต็มรูปแบบ: SVL + DAD + MEC พร้อมทดสอบ Chassis Failure

### เป้าหมายของ Lab

1. แปลง CORE-SW1 + CORE-SW2 เป็น **StackWise Virtual Pair** (Domain 100) พร้อม DAD Link
2. ยกเลิก Po1 (Part 6) และ OSPF ECMP 2 เส้นทางของ Part 33 ทั้งหมด
3. แปลงทั้ง 4 คู่ลิงก์ DIST↔CORE ให้เป็น **MEC Port-channel เดียว** (Po19/29/39/49)
4. ทดสอบ **Chassis Failure** (ปิด CORE-SW2 ทั้งตัว) พร้อม Ping ต่อเนื่อง ยืนยัน Sub-second Failover

### Topology สรุปหลัง Lab เสร็จ

```
                       ┌───────────────────────────────────────────┐
                       │     "CORE" (SVL Domain 100, Logical 1 ตัว)   │
                       │  CORE-SW1(SW1) ◄══SVL 2x10G══► CORE-SW2(SW2) │
                       │  DAD: Gi1/0/1 ◄─────────────► Gi2/0/1        │
                       └──┬────────┬────────┬────────┬───────────────┘
                     Po19 │   Po29 │   Po39 │   Po49 │
              ┌───────────┘        │        │        └───────────┐
        ┌─────┴─────┐        ┌─────┴─────┐  │                ┌─────┴─────┐
        │ DIST-SW1  │        │ DIST-SW2  │  │                │ DIST-SW4  │
        └───────────┘        └───────────┘  │                └───────────┘
                                       ┌─────┴─────┐
                                       │ DIST-SW3  │
                                       └───────────┘
        แต่ละ Po มี 2 Member: 1 เส้นไป Chassis1 (Te1/0/x), 1 เส้นไป Chassis2 (Te2/0/x)
```

### ขั้นตอนที่ 1 — Config CORE-SW1/CORE-SW2 เป็น SVL (ใช้ Config เต็มจาก Step 335)

ใช้ Config Block "ทำบน CORE-SW1" และ "ทำบน CORE-SW2" จาก Step 335 ทุกบรรทัด แล้ว `reload`
ทั้งคู่พร้อมกัน รอจนขึ้น Prompt `CORE#` (ผ่าน Management IP ของ Switch 1)

### ขั้นตอนที่ 2 — ยืนยัน SVL Form สำเร็จก่อน Config MEC

```
CORE# show switch virtual
Switch Mode : Stackwise Virtual
                     Switch 1        Switch 2
------------------------------------------------
Role                 Active           Standby
Domain Number        100              100

CORE# show stackwise-virtual dual-active-detection
Switch      Dual-Active-Detection Link                   Status
------      -----------------------------------------      -------
1                GigabitEthernet1/0/1                        up
2                GigabitEthernet2/0/1                        up
```

**ต้องเห็นผลลัพธ์แบบนี้ก่อนเท่านั้น** ถึงจะไปทำขั้นตอนต่อไปได้ — ถ้า Role ไม่ครบ Active/Standby
หรือ DAD Link ไม่ "up" ห้าม Config MEC ต่อโดยเด็ดขาด (เสี่ยง Config ค้างผิดสถานะ)

### ขั้นตอนที่ 3 — Config MEC ทั้ง 4 Port-channel บน CORE (ใช้ Config Po19 จาก Step 335 เป็น Pattern)

```
CORE(config)# interface range TenGigabitEthernet1/0/2 , TenGigabitEthernet2/0/4
CORE(config-if-range)# description ### MEMBER-OF-Po29-MEC-TO-DIST-SW2 ###
CORE(config-if-range)# no switchport
CORE(config-if-range)# channel-group 29 mode active
CORE(config-if-range)# no shutdown
CORE(config-if-range)# exit
CORE(config)# interface Port-channel29
CORE(config-if)# no switchport
CORE(config-if)# ip address 10.255.29.1 255.255.255.252
CORE(config-if)# no shutdown
CORE(config-if)# exit

CORE(config)# interface range TenGigabitEthernet2/0/1 , TenGigabitEthernet1/0/3
CORE(config-if-range)# description ### MEMBER-OF-Po39-MEC-TO-DIST-SW3 ###
CORE(config-if-range)# no switchport
CORE(config-if-range)# channel-group 39 mode active
CORE(config-if-range)# no shutdown
CORE(config-if-range)# exit
CORE(config)# interface Port-channel39
CORE(config-if)# no switchport
CORE(config-if)# ip address 10.255.39.1 255.255.255.252
CORE(config-if)# no shutdown
CORE(config-if)# exit

CORE(config)# interface range TenGigabitEthernet2/0/2 , TenGigabitEthernet1/0/4
CORE(config-if-range)# description ### MEMBER-OF-Po49-MEC-TO-DIST-SW4 ###
CORE(config-if-range)# no switchport
CORE(config-if-range)# channel-group 49 mode active
CORE(config-if-range)# no shutdown
CORE(config-if-range)# exit
CORE(config)# interface Port-channel49
CORE(config-if)# no switchport
CORE(config-if)# ip address 10.255.49.1 255.255.255.252
CORE(config-if)# no shutdown
CORE(config-if)# exit

CORE(config)# router ospf 1
CORE(config-router)# network 10.255.19.0 0.0.0.3 area 0
CORE(config-router)# network 10.255.29.0 0.0.0.3 area 0
CORE(config-router)# network 10.255.39.0 0.0.0.3 area 0
CORE(config-router)# network 10.255.49.0 0.0.0.3 area 0
CORE(config-router)# end
CORE# write memory
```

### ขั้นตอนที่ 4 — Config DIST-SW2, DIST-SW3, DIST-SW4 (Pattern ตาม DIST-SW1 ใน Step 335)

| Device | Po (MEC) | Member 1 | Member 2 | IP บน Po | OSPF Network เพิ่ม | OSPF Network ที่ลบ (Part 10/33) |
|---|---|---|---|---|---|---|
| DIST-SW2 | Po29 | Te1/1/1 | Te1/1/2 | 10.255.29.2/30 | `network 10.255.29.0 0.0.0.3 area 0` | `10.255.20.0` และ `10.255.21.0` |
| DIST-SW3 | Po39 | Te1/1/1 | Te1/1/2 | 10.255.39.2/30 | `network 10.255.39.0 0.0.0.3 area 0` | `10.255.30.0` และ `10.255.31.0` |
| DIST-SW4 | Po49 | Te1/1/1 | Te1/1/2 | 10.255.49.2/30 | `network 10.255.49.0 0.0.0.3 area 0` | `10.255.40.0` และ `10.255.41.0` |

ทุกตัวใช้คำสั่ง Pattern เดียวกับ Config เต็มของ DIST-SW1 ใน Step 335 ทุกบรรทัด (เปลี่ยนแค่เลข
Interface/Port-channel/Subnet ตามตารางนี้)

### ขั้นตอนที่ 5 — Verify ทั้งระบบก่อนทดสอบ Failure

```
CORE# show etherchannel summary
Number of channel-groups in use: 4
Number of aggregators:            4

Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/0/1(P)   Te2/0/3(P)
29     Po29(RU)        LACP     Te1/0/2(P)   Te2/0/4(P)
39     Po39(RU)        LACP     Te1/0/3(P)   Te2/0/1(P)
49     Po49(RU)        LACP     Te1/0/4(P)   Te2/0/2(P)

CORE# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/  -        00:00:37    10.255.19.2     Po19
1.1.1.12        1     FULL/  -        00:00:39    10.255.29.2     Po29
1.1.1.13        1     FULL/  -        00:00:35    10.255.39.2     Po39
1.1.1.14        1     FULL/  -        00:00:38    10.255.49.2     Po49
```

ยืนยันสำเร็จ: ทุก Po มี Member จาก **Switch 1 และ Switch 2 คนละตัว** ในกลุ่มเดียวกัน (พิสูจน์ว่า
MEC ทำงานจริง) และ OSPF เหลือ **Neighbor เดียวต่อ DIST** (ลดลงจาก 2 ใน Part 33)

### ขั้นตอนที่ 6 — ทดสอบ Chassis Failure: ปิด CORE-SW2 ทั้งตัวระหว่าง Ping ต่อเนื่อง

```
! เปิด ping ต่อเนื่องจาก PC1 (VLAN10, หลัง DIST-SW1) ไป Server1 (VLAN30, หลัง DIST-SW3) ค้างไว้ก่อน
PC1> ping 10.10.30.10 -t

Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
Reply from 10.10.30.10: bytes=32 time=1ms TTL=125
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
                                                        <-- ปิดเครื่อง CORE-SW2 (Chassis 2) ตรงนี้
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125      <-- แทบไม่มี packet loss เลย แม้แต่ 1 reply
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
```

```
%STACKMGR-1-STANDBY_LOST: Standby switch 2 has left the stack
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/1 left the port-channel Port-channel39
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/2 left the port-channel Port-channel49
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/3 left the port-channel Port-channel19
%EC-5-UNBUNDLE: Interface TenGigabitEthernet2/0/4 left the port-channel Port-channel29

CORE# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/0/1(P)   Te2/0/3(D)
29     Po29(RU)        LACP     Te1/0/2(P)   Te2/0/4(D)
39     Po39(RU)        LACP     Te1/0/3(P)   Te2/0/1(D)
49     Po49(RU)        LACP     Te1/0/4(P)   Te2/0/2(D)

CORE# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/  -        00:00:34    10.255.19.2     Po19
1.1.1.12        1     FULL/  -        00:00:31    10.255.29.2     Po29
1.1.1.13        1     FULL/  -        00:00:36    10.255.39.2     Po39
1.1.1.14        1     FULL/  -        00:00:33    10.255.49.2     Po49
```

**สังเกตสิ่งที่สำคัญที่สุดของ Lab นี้**: `show ip ospf neighbor` **ไม่มีอะไรเปลี่ยนแปลงเลยแม้แต่
บรรทัดเดียว** ทั้ง 4 Neighbor ยัง FULL อยู่ครบ ผ่าน Interface เดิมทุกตัว (`Po19`-`Po49`) — OSPF
**ไม่รู้ตัวด้วยซ้ำว่า Chassis ทั้งตัวตายไปแล้วครึ่งหนึ่งของ "CORE"** เพราะมองเห็นแค่ Port-channel
Interface ที่ยัง Up อยู่เหมือนเดิม การ Failover ทั้งหมดเกิดขึ้นที่ระดับ EtherChannel/MEC ล้วนๆ
เร็วกว่าและโปร่งใสกว่า Routed ECMP ของ Part 33 อย่างชัดเจน (ที่ยังต้องเห็น 1 Neighbor หายไปจริง)

### ขั้นตอนที่ 7 — เปิด CORE-SW2 กลับมา ยืนยัน Rejoin อัตโนมัติ

```
%STACKMGR-1-STANDBY_JOINED: Switch 2 has joined the stack as Standby

CORE# show switch virtual
                     Switch 1        Switch 2
------------------------------------------------
Role                 Active           Standby

CORE# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+-----------------------------------------------
19     Po19(RU)        LACP     Te1/0/1(P)   Te2/0/3(P)
29     Po29(RU)        LACP     Te1/0/2(P)   Te2/0/4(P)
39     Po39(RU)        LACP     Te1/0/3(P)   Te2/0/1(P)
49     Po49(RU)        LACP     Te1/0/4(P)   Te2/0/2(P)
```

ทุก Member กลับมา `(P)` ครบทุกตัวโดยไม่ต้อง Config อะไรเพิ่มเติมเลย — Lab นี้ถือว่าสำเร็จเมื่อ:
(1) SVL Form สำเร็จพร้อม DAD Link "up" ทั้งคู่, (2) ทุก Po มี Member จาก Switch 1+Switch 2
ครบ 4 ตัว, (3) OSPF เหลือ 1 Neighbor ต่อ DIST, และ (4) ทดสอบปิด Chassis ตัวใดตัวหนึ่งแล้ว OSPF
Neighbor ไม่มีการเปลี่ยนแปลงเลย พร้อม Ping ต่อเนื่องไม่มี Packet Loss

---

## แบบฝึกหัดทวนความเข้าใจ Part 34

1. เพราะเหตุใด Standard EtherChannel ตามกฎของ Part 6 จึงไม่สามารถมี Member Link เส้นหนึ่งจบที่
   CORE-SW1 อีกเส้นจบที่ CORE-SW2 ได้ และเทคโนโลยีอะไรที่ทำให้ข้อจำกัดนี้หมดไป
2. StackWise (Physical) กับ StackWise Virtual (SVL) เหมือนและต่างกันอย่างไร และทำไมหลักสูตรนี้
   เลือกใช้ SVL กับ CORE-SW1/CORE-SW2 แทน StackWise ธรรมดา
3. Split-Brain คืออะไร เกิดขึ้นได้อย่างไรกับ SVL/VSS และ Dual-Active Detection Link ป้องกันปัญหา
   นี้ได้อย่างไร (อธิบายกรณีที่ SVL Link ขาดแต่ DAD Link ยัง Up)
4. เปรียบเทียบพฤติกรรมเมื่อ CORE-SW1 ทั้งตัวล่มระหว่าง Part 33 (Routed ECMP) กับ Part 34
   (MEC บน SVL) — ทำไม Part 34 จึงทำให้ OSPF "ไม่รู้ตัว" ว่ามี Chassis ล่ม ในขณะที่ Part 33
   OSPF ต้องเสีย Neighbor ไปจริง
5. ในสถานการณ์ที่องค์กรต้องการ Software Fault Domain แยกกันชัดเจนที่สุด (Regulator กำหนด)
   ควรเลือก Routed ECMP (Part 33) หรือ MEC (Part 34) ที่ Distribution↔Core และเพราะเหตุใด

**เฉลย:**

1. เพราะ LACP/PAgP ผูกกับ Control Plane ของ Switch ตัวเดียว (System ID, Hashing Table, การ
   ตัดสินใจ Bundle ทั้งหมดเก็บอยู่ใน Supervisor ตัวเดียว) ถ้า Member Link ไปจบที่ Switch คนละตัว
   ที่มี Control Plane อิสระกัน ฝั่งที่เจรจาจะเห็น System ID ไม่ตรงกันและปฏิเสธการ Bundle — ต้องใช้
   StackWise, StackWise Virtual (SVL), หรือ VSS เพื่อรวม Control Plane ของทั้งสอง Chassis ให้
   เหลือตัวเดียวก่อน ถึงจะทำ Multichassis EtherChannel (MEC) ได้จริง
2. เหมือนกันในแง่ที่ทั้งคู่รวม Control Plane หลาย Chassis เป็นตัวเดียว แต่ StackWise (Physical)
   ใช้สาย Stack Cable เฉพาะทางที่มีระยะทางจำกัดมาก เหมาะกับ Access/Distribution (Catalyst
   9200/9300) ที่ Chassis อยู่ใกล้กันใน Rack เดียวกัน ส่วน SVL ใช้ Interface Ethernet ความเร็วสูง
   ธรรมดา (10G/40G) ทำให้วาง Chassis คนละห้อง/อาคารได้ — CORE-SW1/CORE-SW2 (Catalyst
   9500) ต้องการ Physical Redundancy ระดับห้อง/อาคาร จึงต้องใช้ SVL ไม่ใช่ StackWise ธรรมดา
3. Split-Brain คือภาวะที่ทั้งสอง Chassis คิดว่าตัวเองเป็น Active พร้อมกัน เกิดเมื่อ SVL Link ขาด
   ทั้งหมดแต่ทั้งสอง Chassis ยังทำงานอยู่จริง ทำให้แต่ละฝั่งคิดว่าอีกฝั่งตายแล้วและเลื่อนตัวเองเป็น
   Active — DAD Link (ลิงก์แยกต่างหากจาก SVL) ใช้ตรวจสอบว่าถ้า SVL Link ขาดแต่ DAD Link ยัง
   Up แสดงว่าอีกฝั่งยังมีชีวิตอยู่จริง (แค่ SVL Link ขาดเฉยๆ) ระบบจะให้ Chassis ที่ตรวจพบเข้าสู่
   Recovery Mode (Shutdown Interface ที่ไม่ใช่ SVL/DAD ทั้งหมด) เพื่อไม่ให้ Forward Traffic ซ้ำ
4. Part 33 (Routed ECMP): DIST-SW เห็น CORE-SW1 และ CORE-SW2 เป็น 2 OSPF Neighbor แยกกัน
   ถ้า CORE-SW1 ล่ม OSPF ต้องตรวจจับ Neighbor หายไปจริง แล้วลบ Path นั้นออกจาก ECMP Set
   (ยังเร็วแบบ Sub-second แต่ยังมี "งาน" ให้ OSPF ทำ) — Part 34 (MEC): DIST-SW เห็น "CORE"
   เป็น 1 OSPF Neighbor ผ่าน Interface Port-channel เดียว ถ้า Chassis หนึ่งของ CORE ล่ม MEC
   แค่ตัด Member Link ที่อยู่บน Chassis นั้นออกจาก Hash Table (เหมือน EtherChannel ปกติเวลาสาย
   ขาด) — Interface Po ยังคง Up และ OSPF Neighbor ยังคง FULL อยู่เหมือนเดิมทุกประการ ไม่มี
   Routing Protocol Reconvergence เกี่ยวข้องเลยแม้แต่ก้าวเดียว
5. ควรเลือก **Routed ECMP (Part 33)** เพราะ CORE-SW1 และ CORE-SW2 ยังเป็น Control Plane
   อิสระกัน 100% (คนละ Router-ID, คนละ OSPF Process, คนละ Software Instance) บั๊กหรือปัญหา
   Software ร้ายแรงของตัวหนึ่งไม่มีทางกระทบอีกตัวได้เลย ในขณะที่ MEC (Part 34) ทำให้ทั้งสอง
   Chassis กลายเป็น Control Plane เดียว (ผ่าน SVL) ซึ่งเพิ่ม Software Fault Domain ที่ใช้ร่วมกัน
   แม้จะให้ Failover ที่เร็วและ Config ที่ง่ายกว่าก็ตาม — เป็นการ Trade-off ที่ Regulator ที่เข้มงวด
   เรื่อง Fault Isolation มักไม่ยอมรับ

---

## สรุป Part 34

Part นี้ปิดสัญญาที่ [Part 6](part-006-etherchannel.md) ทิ้งไว้ตั้งแต่ต้นหลักสูตร (ข้อจำกัดของ
Standard EtherChannel ที่ Member Link ต้องจบที่ Switch ตัวเดียวกันเท่านั้น) ด้วยการรวม
**CORE-SW1 และ CORE-SW2 ให้เป็น Logical Switch เดียว** ผ่าน **StackWise Virtual (SVL)**
พร้อม **Dual-Active Detection (DAD)** เพื่อป้องกัน Split-Brain, แปลง OSPF ECMP 2 เส้นทางของ
[Part 33](part-033-advanced-stp-campus-design.md) ให้กลายเป็น **Multichassis EtherChannel
(MEC) ตัวเดียว** ต่อ DIST-SW แต่ละตัว (Po19/29/39/49), เปรียบเทียบ Trade-off ระหว่าง Routed
ECMP กับ MEC อย่างละเอียด (ความเร็ว/ความง่าย vs Fault Isolation), และพิสูจน์ด้วย Lab จริงว่า
การปิด Chassis ทั้งตัวไม่ทำให้ OSPF Neighbor หลุดแม้แต่ตัวเดียว — Campus Network ของเราตอนนี้
มี Redundancy ครบทุกระดับตั้งแต่ Physical Chassis (StackWise Virtual), Link (MEC/EtherChannel),
จนถึง Routing Protocol (OSPF ECMP ที่ยังเหลือใช้ในกรณีอื่น)

สิ่งที่ยังไม่ได้แตะเลยตลอด 34 Part ที่ผ่านมาคือ **Default Gateway ของ End User** — ตั้งแต่
[Part 18](part-018-fhrp.md) เราใช้ HSRP แบบพื้นฐานเป็น Virtual IP ให้ PC/Server เท่านั้น โดยไม่ได้
เจาะลึกกลไกภายใน, การปรับแต่ง Timer, Load-Balancing แบบ GLBP, หรือ Interaction ระหว่าง FHRP
กับ Topology ใหม่ที่ซับซ้อนขึ้นมากจาก Part 33-34 นี้ **Part 35 — Advanced FHRP & Campus HA**
จะพาไปเจาะลึกทั้งหมดนี้ เพื่อปิดวงจร High Availability ของ Campus Network ให้สมบูรณ์ทุกมิติ

**ไปต่อ:** [Part 35 — Advanced FHRP & Campus HA →](part-035-advanced-fhrp-campus-ha.md)
