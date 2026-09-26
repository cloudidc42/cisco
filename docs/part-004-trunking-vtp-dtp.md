# Part 4 — Trunking, VTP, DTP
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 31–40 จาก 1000**

> ต่อจาก [Part 3 — Ethernet Switching & VLAN](part-003-ethernet-switching-vlan.md) ที่สร้าง VLAN
> 10 (SALES), 20 (VOICE), 30 (SERVERS), 40 (WIFI), 99 (MGMT) และตั้งค่า Access Port บน
> ACCESS-SW1–4 แล้ว Part นี้จะขยายเครือข่ายให้ VLAN เหล่านั้น**เดินทางข้ามหลาย Switch ได้**
> ผ่านลิงก์ **Trunk (802.1Q)** ทั้งหมดใน Lab Topology (CORE↔CORE, CORE↔DIST, DIST↔ACCESS)
> พร้อมเรียนรู้ **DTP** (การเจรจา Trunk อัตโนมัติ) และ **VTP** (การซิงค์ฐานข้อมูล VLAN
> ระหว่าง Switch) รวมถึงข้อควรระวังที่ทำให้เครือข่ายจริงล่มมาแล้วนับไม่ถ้วน

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 31 | แนวคิด 802.1Q Trunking, โครงสร้าง Tag, EtherType 0x8100 |
| 32 | Native VLAN โดยละเอียด และความเสี่ยงด้านความปลอดภัย |
| 33 | Configuring Trunk Port บน Cisco IOS (ทุกลิงก์ใน Topology) |
| 34 | DTP (Dynamic Trunking Protocol) และตาราง Link State ทุก Mode |
| 35 | VTP (VLAN Trunking Protocol) พื้นฐานและ VTP Version 1/2/3 |
| 36 | VTP Modes (Server/Client/Transparent) และหายนะ "VTP Bomb" |
| 37 | VTP Pruning |
| 38 | ข้อดีของ VTP Version 3 |
| 39 | Troubleshooting Trunk/VTP |
| 40 | Lab: Config Trunk ทุกลิงก์ + VTP Transparent Mode ทั้ง Topology |

---

## Step 31 — แนวคิด 802.1Q Trunking, โครงสร้าง Tag, EtherType 0x8100

### ทำไมต้องมี Trunk

VLAN แต่ละใบเปรียบเสมือน Broadcast Domain แยกกันคนละ Switch เดียวก็จริง แต่เครือข่ายจริงมี
Switch หลายสิบ-หลายร้อยตัว ถ้าต้องการให้ VLAN 10 (SALES) ที่ ACCESS-SW1 คุยกับ VLAN 10
ที่ ACCESS-SW2 ได้ ต้องมีวิธีส่ง Frame ของหลาย VLAN ผ่าน**สายเส้นเดียว** ระหว่าง Switch โดยที่
ปลายทางยังแยกออกว่า Frame นั้นเป็นของ VLAN ไหน — นี่คือหน้าที่ของ **Trunk Link**

> **Trunk Link** = ลิงก์ Layer 2 ระหว่าง Switch (หรือ Switch↔Router) ที่ถูกออกแบบให้ขนส่ง
> Frame ได้ **หลาย VLAN พร้อมกัน** โดยใส่ "ป้ายกำกับ" (Tag) บอกว่า Frame นั้นเป็นของ VLAN ใด

Cisco เคยมีสอง Protocol สำหรับทำ Trunk:

| Protocol | เจ้าของ | สถานะปัจจุบัน |
|---|---|---|
| **ISL** (Inter-Switch Link) | Cisco Proprietary | เลิกใช้แล้ว (deprecated) ไม่มีใน Switch รุ่นใหม่ |
| **802.1Q** (dot1q) | IEEE Standard (เปิด) | **มาตรฐานเดียวที่ใช้จริงทุกวันนี้** — หลักสูตรนี้ใช้ 802.1Q ตลอด |

### โครงสร้าง 802.1Q Tag (4 byte ที่ถูกแทรกเข้าไปใน Frame เดิม)

802.1Q ไม่ได้ "ห่อ" Frame เดิมใหม่ทั้งหมดแบบ ISL แต่ **แทรก (insert) 4 byte เข้าไปกลาง Frame**
ระหว่าง Source MAC Address กับ EtherType/Length เดิม ทำให้ Frame ที่เคยมี Max 1518 byte
กลายเป็น Max **1522 byte** (Baby Giant Frame)

```
Ethernet Frame ปกติ (Untagged):
┌──────────┬──────────┬───────────┬──────────────┬─────┐
│ Dest MAC │ Src MAC  │ EtherType │   Payload    │ FCS │
│  6 byte  │  6 byte  │  2 byte   │  46-1500 byte │ 4B  │
└──────────┴──────────┴───────────┴──────────────┴─────┘

Ethernet Frame แบบ 802.1Q Tagged:
┌──────────┬──────────┬═══════════════╗──────────┬──────────────┬─────┐
│ Dest MAC │ Src MAC  │ ║ 802.1Q TAG  ║EtherType │   Payload    │ FCS │
│  6 byte  │  6 byte  │ ╚═══4 byte═══╝  2 byte   │  46-1500 byte │ 4B  │
└──────────┴──────────┴───────────────┴──────────┴──────────────┴─────┘
                        ↑
              แทรกตรงนี้ ก่อน EtherType เดิม
```

### รายละเอียดภายใน 4 byte ของ 802.1Q Tag

```
┌─────────────────────┬──────┬─────┬─────────────────────┐
│   TPID (16 bit)      │ PCP  │ DEI │      VID (12 bit)     │
│  Tag Protocol ID     │(3bit)│(1bit)│      VLAN ID          │
│      0x8100          │      │(CFI)│                       │
└─────────────────────┴──────┴─────┴─────────────────────┘
      2 byte                    2 byte (TCI - Tag Control Info)
```

| Field | ขนาด | ความหมาย |
|---|---|---|
| **TPID** (Tag Protocol Identifier) | 16 bit | ค่าคงที่ **0x8100** — บอกว่า Frame นี้มี 802.1Q Tag (นี่คือ EtherType ที่คนพูดถึงกันบ่อยว่า "0x8100") |
| **PCP** (Priority Code Point) | 3 bit | ค่า CoS (Class of Service) 0-7 สำหรับทำ QoS Layer 2 (เรียนละเอียด Part 24, 40) |
| **DEI/CFI** (Drop Eligible Indicator / Canonical Format Indicator) | 1 bit | บอกความสำคัญของการ drop เมื่อ congestion (Ethernet ปกติมักเป็น 0 เสมอ) |
| **VID** (VLAN Identifier) | 12 bit | หมายเลข VLAN จริง (0-4095, ใช้งานได้จริง 1-4094) |

> **ทำไม VLAN สูงสุดคือ 4094 ไม่ใช่ 4096?** เพราะ VID เป็น field 12 bit = 2^12 = 4096 ค่า
> แต่ VLAN 0 สงวนไว้หมายถึง "ไม่มี VLAN ระบุ ใช้ PCP อย่างเดียว" และ VLAN 4095 สงวนไว้เพื่อ
> implementation ภายใน จึงเหลือใช้งานได้จริง 1-4094 (1-1005 = Normal Range, 1006-4094 =
> Extended Range ซึ่งต้องมี VTP Version ที่รองรับ — ดู Step 35, 38)

### กระบวนการ Tag Insertion / Removal บน Trunk Port

1. **PC/Server ส่ง Frame แบบ Untagged** เข้ามาที่ Access Port ของ Switch ตามปกติ (Client
   ไม่มีทางรู้จัก 802.1Q เลย — นี่คือหน้าที่ของ Switch ล้วนๆ)
2. Switch รับ Frame เข้ามาที่ Access Port ที่ผูกกับ VLAN หนึ่ง (เช่น VLAN 10) → Switch "จำ"
   ในใจว่า Frame นี้เป็นของ VLAN 10 (ไม่ได้ tag จริงตอนนี้ยังอยู่ภายใน switch fabric)
3. เมื่อ Switch ต้องส่ง Frame นั้นออกไปทาง **Trunk Port** → Switch จะ **แทรก 802.1Q Tag
   (VID=10)** เข้าไปใน Frame ก่อนส่งออก (Tag Insertion)
4. Switch ปลายทางที่รับ Frame ทาง Trunk Port อ่านค่า VID ในแท็ก แล้ว **ตัด Tag ออก
   (Tag Removal / Strip)** ก่อนส่งต่อไปยัง Access Port ปลายทางที่อยู่ใน VLAN เดียวกัน
5. ผลลัพธ์: PC ปลายทางเห็น Frame แบบ Untagged เหมือนเดิมทุกประการ — 802.1Q Tag มีชีวิตอยู่
   **เฉพาะระหว่างเดินทางบน Trunk Link เท่านั้น**

```
PC1 (VLAN10)          ACCESS-SW1              DIST-SW1              ACCESS-SW2         PC2 (VLAN10)
   │  Untagged Frame  │                       │                       │  Untagged Frame │
   ├─────────────────►│  [Access Port Gi0/1]  │                       │                 │
   │                   │  Tag Insert VID=10    │                       │                 │
   │                   ├──────────────────────►│  [Trunk Port]         │                 │
   │                   │   (Tagged VID=10)      │  Forward ตาม VID      │                 │
   │                   │                        ├──────────────────────►│ [Trunk Port]    │
   │                   │                        │   (Tagged VID=10)      │  Tag Remove     │
   │                   │                        │                        ├────────────────►│
   │                   │                        │                        │ [Access Gi0/2]  │
```

---

## Step 32 — Native VLAN โดยละเอียด และความเสี่ยงด้านความปลอดภัย

### Native VLAN คืออะไร

802.1Q กำหนดว่า Trunk Port ต้องมี **VLAN หนึ่งใบที่ไม่ถูก Tag เวลาส่งออกทาง Trunk** เรียกว่า
**Native VLAN** ค่า Default ของ Cisco คือ **VLAN 1** เสมอ

- Frame ของ VLAN ที่เป็น Native VLAN → ส่งออกทาง Trunk แบบ **Untagged** (ไม่มี 4 byte tag)
- Frame ของ VLAN อื่นทั้งหมด → ส่งออกทาง Trunk แบบ **Tagged** ตามปกติ

เหตุผลที่มี Native VLAN: เพื่อ backward compatibility กับอุปกรณ์รุ่นเก่าที่ไม่รู้จัก 802.1Q เลย
(เช่น Hub, Switch โบราณบางรุ่น หรือ CDP/PVST BPDU บางชนิดที่ Cisco ออกแบบให้เดินทางแบบ
Untagged บน Native VLAN โดยเจตนา)

### กฎเหล็ก: Native VLAN ต้องตรงกันทั้งสองฝั่งของ Trunk

```
CORE-SW1 Gi0/3 (native vlan 999) ◄────── Trunk ──────► DIST-SW1 Gi0/1 (native vlan 10)
                                     ▲
                          NATIVE VLAN MISMATCH!
```

ถ้า Native VLAN ไม่ตรงกัน:

- **CDP จะแจ้งเตือนทันที** ด้วยข้อความคลาสสิกที่วิศวกร Cisco ทุกคนต้องรู้จัก:
  ```
  %CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on
  GigabitEthernet0/3 (999), with DIST-SW1 GigabitEthernet0/1 (10).
  ```
- Traffic ของ Native VLAN ฝั่งหนึ่ง จะถูกอีกฝั่งตีความว่าเป็น VLAN ของตัวเอง (เพราะไม่มี Tag
  ปลายทางก็สรุปเอาว่าเป็น VLAN native ของตัวเอง) → เกิด **VLAN Leaking** ข้อมูลข้าม VLAN
  โดยไม่ได้ตั้งใจ เป็นทั้งปัญหา Reachability และปัญหาความปลอดภัย
- STP อาจเห็น Topology ผิดเพี้ยนเพราะ BPDU (ที่เดินทางแบบ Untagged บน Native VLAN) ไปโผล่ผิด
  VLAN ได้

### ทำไมใช้ VLAN 1 เป็น Native VLAN จึงเป็นความเสี่ยงด้านความปลอดภัย

1. **VLAN 1 เป็นค่า Default ของทุกอย่าง** — Native VLAN default, Management VLAN default
   (บาง platform เก่า), และเป็น VLAN ที่ Control Plane Traffic จำนวนมาก (CDP, DTP, VTP,
   PAgP, STP BPDU ของ PVST+) วิ่งอยู่โดย default ยิ่งเปิดเผยข้อมูลเยอะยิ่งเสี่ยง
2. **VLAN Hopping ผ่าน Double Tagging Attack**: ผู้โจมตีที่เสียบสายเข้า Access Port ที่อยู่ใน
   Native VLAN สามารถส่ง Frame ที่มี **Tag ซ้อนกัน 2 ชั้น** (Outer tag = Native VLAN,
   Inner tag = VLAN เป้าหมาย) เมื่อ Frame ผ่าน Trunk ตัวแรก Switch จะดึง Outer Tag (Native)
   ออกเพราะคิดว่าเป็น Native VLAN แล้วส่งต่อ Frame ที่เหลือ (ซึ่งยังมี Inner Tag ของ VLAN
   เป้าหมายอยู่) ออกไปยัง VLAN เป้าหมายทันทีโดยไม่ผ่านการตรวจสอบใดๆ — เป็นการ "กระโดด" ข้าม
   VLAN แบบ one-way โดยไม่ต้องเป็นสมาชิก VLAN นั้นเลย
3. VLAN 1 ไม่สามารถลบออกจาก Switch ได้ (เป็น VLAN ที่ Cisco สงวนไว้เสมอ) จึงยิ่งจำเป็นต้อง
   "ลดบทบาท" มันให้เหลือน้อยที่สุด แทนที่จะลบไม่ได้

### Best Practice: เปลี่ยน Native VLAN เป็น VLAN ที่ไม่ได้ใช้งานจริง (Unused VLAN)

แนวทางมาตรฐาน Enterprise (และเป็นสิ่งที่หลักสูตรนี้จะใช้ตลอดทุก Part ถัดไป):

1. สร้าง VLAN ใหม่ที่ **ไม่มี Access Port ใดสังกัดอยู่เลย** ใช้เป็น Native VLAN โดยเฉพาะ
   ในหลักสูตรนี้กำหนดให้เป็น **VLAN 999** ชื่อ `NATIVE-UNUSED`
2. ตั้งค่า `switchport trunk native vlan 999` บน **ทุก Trunk Port** ใน Topology ให้ตรงกันหมด
3. ห้ามให้ VLAN 1 รับ Traffic ของ User ใดๆ เลย (ไม่ Assign VLAN 1 ให้ Access Port ใดๆ)
4. เปิด `service unsupported-transceiver` ไม่เกี่ยวข้อง — สิ่งที่เกี่ยวข้องคือ **ปิด CDP/DTP
   บน Native VLAN Trunk ที่ไม่จำเป็นต้องเจรจา** (ดู Step 34)

```
CORE-SW1(config)# vlan 999
CORE-SW1(config-vlan)# name NATIVE-UNUSED
CORE-SW1(config-vlan)# exit
CORE-SW1(config)# interface GigabitEthernet0/3
CORE-SW1(config-if)# switchport trunk native vlan 999
```

> ตั้งแต่ Part นี้เป็นต้นไป **VLAN 999 = Native VLAN มาตรฐานของทุก Trunk Link ใน Lab Topology**
> จำค่านี้ไว้ให้แม่น เพราะจะปรากฏซ้ำในทุก Part ที่เกี่ยวกับ Layer 2

---

## Step 33 — Configuring Trunk Port บน Cisco IOS (ทุกลิงก์ใน Topology)

### คำสั่งหลักที่ต้องใช้

| คำสั่ง | ความหมาย |
|---|---|
| `switchport trunk encapsulation dot1q` | เลือกใช้ 802.1Q (จำเป็นเฉพาะ Switch รุ่นเก่าที่รองรับทั้ง ISL และ dot1q เช่น Catalyst 3560/3750; Switch รุ่นใหม่เช่น Catalyst 9000 series รองรับ dot1q อย่างเดียวจึงไม่มีคำสั่งนี้ให้เลือก) |
| `switchport mode trunk` | บังคับให้ Port เป็น Trunk ถาวร (ไม่ต้องรอ DTP เจรจา) |
| `switchport trunk native vlan <id>` | กำหนด Native VLAN (ต้องตรงกันทั้งสองฝั่ง) |
| `switchport trunk allowed vlan <list>` | จำกัด VLAN ที่อนุญาตให้วิ่งผ่าน Trunk (default คือ "all" คือทุก VLAN 1-4094) |
| `switchport trunk allowed vlan add <list>` | เพิ่ม VLAN เข้าไปในลิสต์เดิม (ไม่ล้างของเก่า) |
| `switchport trunk allowed vlan remove <list>` | ลบ VLAN ออกจากลิสต์ |
| `switchport nonegotiate` | ปิดการส่ง DTP Frame (ดู Step 34) |

### กำหนด Interface Mapping ของ Lab Topology (ใช้ตลอดหลักสูตรนี้)

เพื่อให้ Config ทุก Part สอดคล้องกัน หลักสูตรนี้กำหนด Physical Interface ของแต่ละลิงก์ดังนี้
(สรุปจาก Topology 3-Tier ใน Part 1 — ทุก Switch เชื่อมลง 2 ทางเพื่อ Redundancy):

| ลิงก์ | ปลาย A | ปลาย B |
|---|---|---|
| Core LACP | CORE-SW1 Gi0/1,Gi0/2 (Po1) | CORE-SW2 Gi0/1,Gi0/2 (Po1) |
| Core→Dist | CORE-SW1 Gi0/3 | DIST-SW1 Gi0/1 |
| Core→Dist | CORE-SW1 Gi0/4 | DIST-SW2 Gi0/1 |
| Core→Dist | CORE-SW1 Gi0/5 | DIST-SW3 Gi0/1 |
| Core→Dist | CORE-SW1 Gi0/6 | DIST-SW4 Gi0/1 |
| Core→Dist | CORE-SW2 Gi0/3 | DIST-SW1 Gi0/2 |
| Core→Dist | CORE-SW2 Gi0/4 | DIST-SW2 Gi0/2 |
| Core→Dist | CORE-SW2 Gi0/5 | DIST-SW3 Gi0/2 |
| Core→Dist | CORE-SW2 Gi0/6 | DIST-SW4 Gi0/2 |
| Dist→Access | DIST-SW1 Gi0/3 | ACCESS-SW1 Gi0/1 |
| Dist→Access | DIST-SW1 Gi0/4 | ACCESS-SW2 Gi0/1 |
| Dist→Access | DIST-SW2 Gi0/3 | ACCESS-SW1 Gi0/2 |
| Dist→Access | DIST-SW2 Gi0/4 | ACCESS-SW2 Gi0/2 |
| Dist→Access | DIST-SW3 Gi0/3 | ACCESS-SW3 Gi0/1 |
| Dist→Access | DIST-SW3 Gi0/4 | ACCESS-SW4 Gi0/1 |
| Dist→Access | DIST-SW4 Gi0/3 | ACCESS-SW3 Gi0/2 |
| Dist→Access | DIST-SW4 Gi0/4 | ACCESS-SW4 Gi0/2 |

VLAN ที่ต้องอนุญาตบนแต่ละ segment:

| Segment | VLAN ที่ต้องผ่าน | เหตุผล |
|---|---|---|
| CORE↔CORE, CORE↔DIST | 10, 20, 30, 40, 99, 999(native) | Core ต้องเห็นทุก VLAN เพื่อรองรับทุก Dist block |
| DIST1/2 ↔ ACCESS1/2 | 10, 20, 99, 999(native) | ACCESS-SW1/2 มีแค่ VLAN SALES/VOICE ตาม Part 3 |
| DIST3/4 ↔ ACCESS3/4 | 30, 40, 99, 999(native) | ACCESS-SW3/4 มีแค่ VLAN SERVERS/WIFI ตาม Part 3 |

### Config เต็ม: CORE-SW1 (ตัวอย่างครบทุกลิงก์ขาออกจาก Core ตัวนี้)

```
CORE-SW1(config)# vlan 999
CORE-SW1(config-vlan)# name NATIVE-UNUSED
CORE-SW1(config-vlan)# exit

! ----- Po1 ไป CORE-SW2 (สร้าง Port-channel เต็มรูปแบบใน Part 6 - ที่นี่โฟกัสแค่ trunk) -----
CORE-SW1(config)# interface range GigabitEthernet0/1-2
CORE-SW1(config-if-range)# description ** LACP Po1 member to CORE-SW2 **
CORE-SW1(config-if-range)# switchport trunk encapsulation dot1q
CORE-SW1(config-if-range)# switchport mode trunk
CORE-SW1(config-if-range)# switchport trunk native vlan 999
CORE-SW1(config-if-range)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW1(config-if-range)# switchport nonegotiate
CORE-SW1(config-if-range)# exit

! ----- Trunk ไป DIST-SW1 -----
CORE-SW1(config)# interface GigabitEthernet0/3
CORE-SW1(config-if)# description ** Trunk to DIST-SW1 Gi0/1 **
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW1(config-if)# switchport nonegotiate
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! ----- Trunk ไป DIST-SW2 -----
CORE-SW1(config)# interface GigabitEthernet0/4
CORE-SW1(config-if)# description ** Trunk to DIST-SW2 Gi0/1 **
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW1(config-if)# switchport nonegotiate
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! ----- Trunk ไป DIST-SW3 -----
CORE-SW1(config)# interface GigabitEthernet0/5
CORE-SW1(config-if)# description ** Trunk to DIST-SW3 Gi0/1 **
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW1(config-if)# switchport nonegotiate
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! ----- Trunk ไป DIST-SW4 -----
CORE-SW1(config)# interface GigabitEthernet0/6
CORE-SW1(config-if)# description ** Trunk to DIST-SW4 Gi0/1 **
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW1(config-if)# switchport nonegotiate
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
```

> **CORE-SW2 ใช้ Config รูปแบบเดียวกันทุกบรรทัด** เพียงเปลี่ยน description ปลายทาง — ดู Full
> Config ทุกอุปกรณ์ใน Step 40

### Config ตัวอย่าง: DIST-SW1 (ฝั่ง Dist มีทั้งขาขึ้น Core และขาลง Access)

```
DIST-SW1(config)# vlan 999
DIST-SW1(config-vlan)# name NATIVE-UNUSED
DIST-SW1(config-vlan)# exit

! ----- ขาขึ้น CORE-SW1 -----
DIST-SW1(config)# interface GigabitEthernet0/1
DIST-SW1(config-if)# description ** Trunk to CORE-SW1 Gi0/3 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk native vlan 999
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW1(config-if)# switchport nonegotiate
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! ----- ขาขึ้น CORE-SW2 -----
DIST-SW1(config)# interface GigabitEthernet0/2
DIST-SW1(config-if)# description ** Trunk to CORE-SW2 Gi0/3 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk native vlan 999
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW1(config-if)# switchport nonegotiate
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! ----- ขาลง ACCESS-SW1 (เฉพาะ VLAN 10,20,99 เพราะ ACCESS-SW1 ไม่มี VLAN 30/40) -----
DIST-SW1(config)# interface GigabitEthernet0/3
DIST-SW1(config-if)# description ** Trunk to ACCESS-SW1 Gi0/1 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk native vlan 999
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
DIST-SW1(config-if)# switchport nonegotiate
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! ----- ขาลง ACCESS-SW2 -----
DIST-SW1(config)# interface GigabitEthernet0/4
DIST-SW1(config-if)# description ** Trunk to ACCESS-SW2 Gi0/1 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk native vlan 999
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
DIST-SW1(config-if)# switchport nonegotiate
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit
```

### Verify

```
DIST-SW1# show interfaces trunk

Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      999
Gi0/2       on               802.1q         trunking      999
Gi0/3       on               802.1q         trunking      999
Gi0/4       on               802.1q         trunking      999

Port        Vlans allowed on trunk
Gi0/1       10,20,30,40,99,999
Gi0/2       10,20,30,40,99,999
Gi0/3       10,20,99,999
Gi0/4       10,20,99,999

Port        Vlans allowed and active in management domain
Gi0/1       10,20,30,40,99,999
Gi0/3       10,20,99,999

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,20,30,40,99
Gi0/3       10,20,99
```

---

## Step 34 — DTP (Dynamic Trunking Protocol) และตาราง Link State ทุก Mode

### DTP คืออะไร

**DTP (Dynamic Trunking Protocol)** เป็น Protocol ของ Cisco (Proprietary) ที่ทำงานอยู่เบื้องหลัง
ทุกครั้งที่ตั้งค่า `switchport mode` — มีหน้าที่ **เจรจาอัตโนมัติ** ว่า Port สองฝั่งควรกลายเป็น
Trunk หรือไม่ โดยส่ง DTP Frame ทุก 30 วินาที ผ่าน VLAN 1 (Native VLAN default) แบบ Untagged

### 5 Mode ของ Switchport ที่เกี่ยวกับ DTP

| Mode | คำสั่ง | พฤติกรรม |
|---|---|---|
| **Access** | `switchport mode access` | บังคับเป็น Access Port ตายตัว ไม่ยอมเป็น Trunk เด็ดขาด |
| **Trunk** | `switchport mode trunk` | บังคับเป็น Trunk ตายตัว (แต่ยังส่ง DTP frame โดย default นอกจากตั้ง nonegotiate) |
| **Dynamic Auto** | `switchport mode dynamic auto` | **รอ** ให้อีกฝั่ง "ชวน" ก่อน ถ้าอีกฝั่งไม่ชวนก็จะเป็น Access (Default บน Switch รุ่นเก่าหลายรุ่น) |
| **Dynamic Desirable** | `switchport mode dynamic desirable` | **เสนอตัว/ชวน** อีกฝั่งให้เป็น Trunก อย่างต่อเนื่อง |
| **Nonegotiate** | `switchport nonegotiate` (ใช้ร่วมกับ mode trunk/access) | ปิดการส่ง DTP Frame ทั้งหมด — ต้องตั้ง mode ให้ตรงกันเองแบบ manual ทั้งสองฝั่ง |

### ตาราง Link State เมื่อจับคู่ Mode ทุกแบบ (ต้องจำให้ขึ้นใจ — ออกข้อสอบทุกระดับ)

| ฝั่ง A \ ฝั่ง B | Access | Trunk | Dynamic Auto | Dynamic Desirable | Nonegotiate |
|---|---|---|---|---|---|
| **Access** | Access | ❌ ไม่ตรงกัน (err/Access) | Access | Access | Access |
| **Trunk** | ❌ ไม่ตรงกัน | **Trunk** | **Trunk** | **Trunk** | **Trunk** (ต้องตั้ง native/allowed ให้ตรงเอง) |
| **Dynamic Auto** | Access | **Trunk** | **Access** (ทั้งคู่รอ ไม่มีใครชวน) | **Trunk** | Access (ไม่มี DTP มาถึง จึงอยู่ Access) |
| **Dynamic Desirable** | Access | **Trunk** | **Trunk** | **Trunk** | Access (ไม่มี DTP มาถึง จึงอยู่ Access) |
| **Nonegotiate** | Access | **Trunk** (ต้องตั้ง native/allowed ให้ตรงเอง) | Access | Access | ต้อง manual ทั้งสองฝั่งให้ตรงกัน |

> ❌ **Access ↔ Trunk (Mode ตรงข้ามแบบ manual ทั้งคู่)** = ปัญหาที่พบบ่อยที่สุดในการสอบ
> Troubleshooting: ฝั่งหนึ่งบังคับ Access ตายตัว อีกฝั่งบังคับ Trunk ตายตัว ผลลัพธ์คือ
> **Trunk ไม่ขึ้น** ฝั่ง Trunk จะเห็น Native VLAN traffic ของตัวเองรั่วเข้าไปใน VLAN ของฝั่ง
> Access (เพราะฝั่ง Access ไม่รู้จัก Tag เลย ตีความทุกอย่างเป็น VLAN ของตัวเอง) — เป็นอีกช่องทาง
> ของ VLAN Leaking ที่ไม่ต้องเป็นการโจมตีเลยด้วยซ้ำ แค่ตั้งค่าไม่ตรงกันก็เกิดได้

### Best Practice ด้านความปลอดภัย: ปิด DTP ด้วย `nonegotiate` เสมอ

DTP เป็นช่องโหว่คลาสสิกของ **VLAN Hopping ผ่าน Switch Spoofing**: ถ้า Access Port ของ User
ถูกปล่อยไว้ที่ Default Mode (`dynamic auto` หรือ `dynamic desirable` บน Switch บางรุ่น) ผู้โจมตี
สามารถเสียบ Laptop ที่รัน Software ปลอม DTP Frame (เช่น Yersinia) หลอกให้ Switch เชื่อว่าอีกฝั่ง
เป็น Switch แล้ว "เจรจา" ให้ Port นั้นกลายเป็น **Trunk** — เมื่อเป็น Trunk แล้ว ผู้โจมตีจะเห็น
Traffic ได้ทุก VLAN ที่ Trunk อนุญาต ทันที

**แนวทางป้องกันมาตรฐาน Enterprise (ใช้ตลอดหลักสูตรนี้):**

```
! บน Trunk Port ระหว่าง Switch กับ Switch (Infrastructure Link)
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport nonegotiate

! บน Access Port ที่ต่อกับ User/PC/Printer (ตั้งจริงใน Part 3, ย้ำที่นี่)
ACCESS-SW1(config-if)# switchport mode access
ACCESS-SW1(config-if)# switchport nonegotiate
ACCESS-SW1(config-if)# spanning-tree portfast
ACCESS-SW1(config-if)# spanning-tree bpduguard enable
```

> **กฎ Enterprise**: Port ทุกใบต้องถูกบังคับ Mode แบบ Manual (`access` หรือ `trunk`) พร้อม
> `nonegotiate` เสมอ **ห้ามปล่อย Default `dynamic auto/desirable`** ไว้บน Production Switch
> โดยเด็ดขาด — Config ทุกอุปกรณ์ใน Step 33 และ 40 ของหลักสูตรนี้จึงมี `switchport nonegotiate`
> กำกับทุกบรรทัดของ Trunk Port

---

## Step 35 — VTP (VLAN Trunking Protocol) พื้นฐานและ VTP Version 1/2/3

### ปัญหาที่ VTP แก้

ลอง imagine ว่า Lab Topology นี้มี Switch 10 ตัว และต้องสร้าง VLAN ใหม่ 1 ใบ (เช่น VLAN 50)
ถ้าไม่มี VTP ต้องเข้าไป `vlan 50` + `name ...` **ทีละเครื่อง 10 ครั้ง** และถ้าลืมเครื่องใดเครื่องหนึ่ง
VLAN นั้นจะใช้งานข้าม Switch ไม่ได้ (Trunk อนุญาต VLAN 50 ผ่านได้ก็จริง แต่ปลายทางไม่มี VLAN 50
ในฐานข้อมูลของตัวเองก็ไร้ประโยชน์)

> **VTP (VLAN Trunking Protocol)** = Protocol ของ Cisco ที่ทำให้ Switch **ทุกตัวในโดเมนเดียวกัน
> (VTP Domain)** ที่เชื่อมกันด้วย Trunk **ซิงค์ฐานข้อมูล VLAN (VLAN Database)** ให้ตรงกัน
> อัตโนมัติ — สร้าง/ลบ/แก้ไข VLAN ที่ Switch ตัวเดียว (VTP Server) แล้วกระจายไปทุกตัวเอง

### VTP Domain

- Switch ทุกตัวต้องอยู่ **VTP Domain Name เดียวกัน** (Case-Sensitive!) จึงจะแลกเปลี่ยนข้อมูลกันได้
- Switch ที่ยังไม่เคยตั้ง Domain Name (`null domain`) จะ **เรียนรู้ Domain Name อัตโนมัติ** จาก
  VTP Advertisement แรกที่ได้รับผ่าน Trunk (เป็นอีกเหตุผลที่ VTP อันตรายถ้าไม่ระวัง — ดู Step 36)
- ควรตั้ง **VTP Password** เพื่อป้องกัน Switch แปลกปลอมเข้าร่วมโดเมนโดยไม่ได้รับอนุญาต

### VTP Advertisement 3 ประเภท

| ประเภท | ส่งเมื่อไร | เนื้อหา |
|---|---|---|
| **Summary Advertisement** | ทุก 300 วินาที หรือทันทีที่มีการเปลี่ยนแปลง | VTP Domain, Version, Revision Number, Timestamp (**ไม่มีรายละเอียด VLAN**) |
| **Subset Advertisement** | ตามหลัง Summary ทันทีเมื่อมีการเปลี่ยนแปลงจริง | รายละเอียด VLAN ที่เปลี่ยน (VLAN ID, Name, MTU, Type) |
| **Advertisement Request** | Switch ใหม่ขอข้อมูลจาก Switch อื่นในโดเมน (เช่นหลัง reset) | ขอให้ VTP Server ส่ง Summary+Subset กลับมาทั้งหมด |

### เปรียบเทียบ VTP Version 1 / 2 / 3

| คุณสมบัติ | VTPv1 | VTPv2 | VTPv3 |
|---|---|---|---|
| ค่า Default บน Switch ใหม่ | v1 (ปิดอยู่จนกว่าจะสร้าง VLAN ครั้งแรก) | - | v1 (ต้อง config เปิดใช้ v3 เอง) |
| รองรับ Token Ring VLAN | ไม่ | ได้ | ได้ (ผ่าน v2 compat) |
| ตรวจสอบความสอดคล้อง (Consistency Check) | น้อยกว่า | มี Consistency Check ที่ดีขึ้น (ตรวจ VLAN name/value ก่อนส่งต่อ) | เข้มงวดสุด |
| ส่งต่อ VTP Advertisement ที่ไม่รู้จัก (Transparent mode) | ตรวจสอบ Domain/Version ก่อนส่งต่อ | ส่งต่อโดยไม่ตรวจสอบ (transparent forwards regardless) | เหมือน v2 |
| รองรับ Extended VLAN (1006-4094) | **ไม่รองรับ** (Server/Client mode) | ไม่รองรับ | **รองรับ** |
| รองรับ Private VLAN | ไม่ | ไม่ | **รองรับ** |
| Primary Server Election | ไม่มี | ไม่มี | **มี** (ดู Step 38) |
| Authentication | Password แบบ plaintext hash เดียว | เหมือน v1 | รองรับ Hidden/Secret password |
| MST (Multiple Spanning Tree) Database | ไม่รองรับ | ไม่รองรับ | **รองรับ** |

> **ข้อสำคัญ**: VTPv1 และ v2 ตัว Version ต้องตรงกันหรือ "compatible" กันเพื่อสื่อสาร (Switch ที่
> ตั้ง v2 จะ auto-negotiate กับ v1 ได้ในบางกรณี) แต่ **VTPv3 ต้อง config เปิดใช้งานเอง** ไม่
> auto-detect และไม่ backward-compatible กับ v1/v2 โดยตรงในแง่ฟีเจอร์ขั้นสูง

---

## Step 36 — VTP Modes (Server/Client/Transparent) และหายนะ "VTP Bomb"

### 3 Mode หลักของ VTP

| Mode | สร้าง/ลบ/แก้ VLAN ได้เอง? | ส่ง VTP Advertisement? | รับและประมวลผล Advertisement? | เก็บ VLAN Database ใน NVRAM (`vlan.dat`)? |
|---|---|---|---|---|
| **Server** (default) | ✅ ได้ | ✅ ส่ง | ✅ รับและ sync ตาม | ✅ เก็บ |
| **Client** | ❌ ไม่ได้ (ต้องรอรับจาก Server) | ✅ ส่งต่อ (forward) | ✅ รับและ sync ตาม | ✅ เก็บ (v2 ลงมา) |
| **Transparent** | ✅ ได้ (แต่มีผลแค่ในตัวเอง ไม่กระจายออก) | ➡️ ส่งต่อ Advertisement ที่รับมาเฉยๆ (ไม่สร้างเอง) | ❌ ไม่ sync ตาม (เก็บ VLAN ของตัวเองแยกต่างหาก) | ✅ เก็บใน running-config โดยตรง |

### คำสั่ง Config

```
! ตั้งเป็น Server (default อยู่แล้ว แต่ระบุชัดเจนเพื่อความรัดกุม)
Switch(config)# vtp mode server
Switch(config)# vtp domain LABCORP
Switch(config)# vtp version 2
Switch(config)# vtp password Cisco@VTP123

! ตั้งเป็น Client
Switch(config)# vtp mode client

! ตั้งเป็น Transparent (แนะนำสำหรับ Enterprise ส่วนใหญ่ - ดูเหตุผลด้านล่าง)
Switch(config)# vtp mode transparent

! Verify
Switch# show vtp status
Switch# show vtp password
```

### VTP Revision Number คืออะไร

ทุกครั้งที่มีการแก้ไข VLAN Database บน VTP Server (เพิ่ม/ลบ/แก้ชื่อ VLAN) **Revision Number
จะเพิ่มขึ้นทีละ 1** (เริ่มจาก 0) ค่านี้เป็นตัวตัดสิน "ใครคือข้อมูลล่าสุด":

> **กฎทอง**: Switch ในโดเมนเดียวกันจะ**ยอมรับ**ฐานข้อมูล VLAN จาก Advertisement ที่มี
> **Revision Number สูงกว่าเท่านั้น** ไม่สนใจว่า Advertisement นั้นมาจาก Server ตัวไหน
> หรือถูกต้องหรือไม่ — **สูงกว่าชนะเสมอ**

### หายนะ "VTP Bomb" (VTP Revision Number Disaster)

นี่คือเหตุการณ์จริงที่เคยทำให้เครือข่าย Enterprise ล่มทั้งองค์กรมาแล้วหลายครั้งในประวัติศาสตร์:

```
สถานการณ์:
1. Lab เดิมมี Switch ทั้งหมดในโดเมน "LABCORP" มี VLAN 10,20,30,40,99 (Revision = 15)
2. วิศวกรเอา Switch สำรอง (เคยใช้ทดสอบใน Lab อื่นมาก่อน) มาเสียบเข้า Trunk ใน Network จริง
   Switch ตัวนี้เคยอยู่โดเมนชื่อ "LABCORP" เหมือนกัน (บังเอิญ หรือ template เดียวกัน)
   แต่มี VLAN Database เก่าที่ถูกทดลองสร้าง/ลบ VLAN ไปมาจนมี Revision = 99 (สูงกว่ามาก)
   แม้ตัว Switch จะถูกตั้งเป็น "VTP Client" ก็ตาม!
3. ทันทีที่เสียบ Trunk เข้าไป Switch ตัวนี้ส่ง Summary Advertisement ออกไปพร้อม Revision=99
4. Switch Server ตัวจริงในเครือข่ายเห็นว่า Revision 99 > 15 → เชื่อว่าเป็นข้อมูลใหม่กว่า
   **ยอมรับและ Overwrite ฐานข้อมูล VLAN ทั้งหมดของตัวเองทันที**
5. ถ้า VLAN Database เก่าของ Switch สำรองนั้นไม่มี VLAN 10,20,30,40,99 เลย (เพราะไปทดลองลบ
   ทิ้งใน Lab อื่น) → **VLAN ทั้งหมดในเครือข่ายจริงหายไปในพริบตา**
6. ผลลัพธ์: ทุก Access Port ที่เคย Assign VLAN ที่หายไป จะกลายเป็น "Inactive" ทันที
   **Network ทั้งองค์กรล่มพร้อมกัน** ภายในไม่กี่วินาที โดยไม่มีใครแก้ไข Config อะไรเลย
```

### วิธีป้องกัน VTP Bomb (Best Practice ที่ต้องทำเสมอ)

1. **ใช้ VTP Mode Transparent เป็นค่าปกติทั่วทั้งองค์กร** (ไม่ให้ VLAN Database ถูก sync
   ข้าม Switch แบบอัตโนมัติเลย) — นี่คือแนวทางที่หลักสูตรนี้เลือกใช้ (ดู Step 40)
2. **ก่อนนำ Switch ตัวใดๆ เข้าเครือข่ายจริงเสมอ** ให้ Reset Revision Number ก่อน โดยวิธีใดวิธีหนึ่ง:
   - เปลี่ยน `vtp domain` เป็นชื่ออื่นชั่วคราวแล้วเปลี่ยนกลับ (Revision จะรีเซ็ตเป็น 0)
   - `vtp mode transparent` แล้วค่อยเปลี่ยนกลับเป็น server/client (รีเซ็ต Revision)
   - ลบไฟล์ `vlan.dat` ออกจาก flash แล้ว reload (`delete flash:vlan.dat`)
3. ตั้ง **VTP Password** ทุกโดเมน (แม้ Transparent Mode ก็ควรตั้งเผื่อเปลี่ยน Mode ในอนาคต)
4. ใช้ **VTPv3** ซึ่งมี Primary Server Election แก้ปัญหานี้ได้อย่างเป็นระบบ (ดู Step 38)
5. เปิด Change Management: ห้ามเสียบ Switch ที่ไม่รู้ประวัติเข้า Trunk Link ของ Production
   โดยไม่ตรวจสอบ `show vtp status` ก่อนเสมอ

---

## Step 37 — VTP Pruning

### VTP Pruning คืออะไร

โดย Default, Trunk Link จะส่ง **Broadcast, Unknown Unicast, และ Multicast Traffic ของทุก VLAN
ที่ Allowed** ออกไปยังทุก Switch ปลายทาง แม้ Switch ปลายทางนั้นจะ**ไม่มี Access Port ใดอยู่ใน
VLAN นั้นเลยก็ตาม** เป็นการสิ้นเปลือง Bandwidth บน Trunk โดยไม่จำเป็น

> **VTP Pruning** = กลไกที่ทำให้ Trunk Link **หยุดส่ง** Broadcast/Unknown-Unicast/Multicast
> ของ VLAN ที่ Switch ปลายทางไม่มี Active Port อยู่เลย โดยอัตโนมัติ ไม่ต้องตั้ง `allowed vlan`
> ด้วยมือ

### ตัวอย่างประโยชน์ในบริบท Lab นี้

```
DIST-SW1 มี Trunk ลง ACCESS-SW1 และ ACCESS-SW2 (มีแค่ VLAN 10,20 ตาม Part 3)
ถ้า Trunk อนุญาต VLAN 10,20,30,40,99 ทั้งหมด (allowed vlan กว้าง) แต่ ACCESS-SW1/2
ไม่มี Access Port ใดอยู่ VLAN 30 (SERVERS) หรือ 40 (WIFI) เลย

ไม่มี Pruning: Broadcast ของ VLAN 30/40 (เช่น ARP Request จำนวนมากจาก Server1-3)
              ยังคงถูกส่งไปที่ ACCESS-SW1/2 ทุกครั้ง แล้วถูก drop ทิ้งเปล่าๆที่ปลายทาง
              (เสีย Bandwidth บน Trunk DIST-SW1↔ACCESS-SW1 ไปฟรีๆ)

มี Pruning:    VTP ตรวจพบว่า ACCESS-SW1 ไม่มี Port ใน VLAN 30/40 → "Prune" VLAN 30/40
              ออกจาก Trunk นั้นโดยอัตโนมัติ Broadcast จะไม่ถูกส่งไปเลย
              ประหยัด Bandwidth ได้จริงโดยไม่ต้องตั้ง allowed vlan เอง
```

### การเปิดใช้งาน

```
! เปิดที่ VTP Server เพียงเครื่องเดียว จะกระจายค่านี้ไปทั้งโดเมนอัตโนมัติ
CORE-SW1(config)# vtp pruning

! ตรวจสอบ
CORE-SW1# show vtp status
CORE-SW1# show interfaces trunk        ! ดูคอลัมน์ "Vlans allowed and active in management domain"
                                        ! เทียบกับ "Vlans in spanning tree forwarding state and not pruned"
```

### VLAN ที่ Prune ได้ (Pruning-Eligible) โดย Default

- **VLAN ที่ Prune ได้โดย Default**: 2–1001
- **VLAN ที่ Prune ไม่ได้เด็ดขาด**: VLAN 1 (Native/Management แบบเก่า), VLAN 1002-1005
  (สงวนสำหรับ Token Ring/FDDI แบบเก่า), และ Extended Range VLAN (1006-4094 ต้อง config
  eligible list เอง)
- ปรับ List ที่ Prune ได้ต่อ Trunk Port ด้วย:
  ```
  DIST-SW1(config-if)# switchport trunk pruning vlan 10,20,30,40
  DIST-SW1(config-if)# switchport trunk pruning vlan remove 99   ! ห้าม prune VLAN 99 (MGMT) ออก
  ```

> **หมายเหตุสำคัญ**: VTP Pruning ทำงานเฉพาะ **VTP Server/Client Mode เท่านั้น** — ถ้าทั้งโดเมน
> ใช้ **Transparent Mode** (ตามที่หลักสูตรนี้แนะนำใน Step 36/40) VTP Pruning จะ**ไม่มีผลใดๆ**
> เพราะ Transparent ไม่ participate ใน VTP Domain เลย ในกรณีนั้นต้องพึ่ง `switchport trunk
> allowed vlan` แบบ Manual แทน (ซึ่งก็คือสิ่งที่ Step 33/40 ทำอยู่แล้ว)

---

## Step 38 — ข้อดีของ VTP Version 3

VTPv3 ถูกออกแบบมาแก้จุดอ่อนสำคัญของ v1/v2 โดยเฉพาะปัญหา "VTP Bomb" ใน Step 36:

### 1. Primary Server Election

ใน VTPv3 มีแนวคิดใหม่คือ **Primary Server** — ต่างจาก v1/v2 ที่ Server ทุกตัวเท่าเทียมกันหมด
(ใครส่ง Revision สูงกว่าก็ชนะ) VTPv3 กำหนดว่า **มีได้แค่ 1 Primary Server ต่อโดเมนต่อ Instance**
เท่านั้นที่สามารถสร้าง/แก้ไข VLAN ได้จริง Switch อื่นแม้ตั้ง Mode เป็น Server ก็ยังเป็นแค่
"Secondary Server" ที่รับข้อมูลอย่างเดียว จนกว่าจะถูกเลือกเป็น Primary อย่างชัดเจนด้วยคำสั่ง:

```
CORE-SW1(config)# vtp version 3
CORE-SW1(config)# vtp domain LABCORP
CORE-SW1(config)# vtp password Cisco@VTP123
CORE-SW1# vtp primary vlan
This system is becoming primary server for feature vlan
No conflicting VTP3 devices found.
Do you want to continue? [confirm]
```

การเลือก Primary ต้องทำ**ผ่านคำสั่ง `vtp primary` โดยตรง** (Manual Action) ไม่ใช่อัตโนมัติจาก
Revision Number ที่สูงกว่าเหมือน v1/v2 อีกต่อไป → **นี่คือการแก้ปัญหา VTP Bomb ที่ต้นเหตุ**
เพราะ Switch แปลกปลอมที่มี Revision สูงแค่ไหนก็ไม่มีสิทธิ์ Overwrite ฐานข้อมูลได้เองอีกแล้ว

### 2. รองรับ Extended VLAN Range (1006-4094)

v1/v2 ในโหมด Server/Client จำกัดที่ VLAN 1-1005 เท่านั้น หากต้องการใช้ VLAN สูงกว่า 1005 (เช่น
ในงาน Service Provider หรือ Data Center ที่ต้องใช้ VLAN ID จำนวนมาก) ต้องเปลี่ยนเป็น Transparent
Mode เท่านั้น — VTPv3 ยกเลิกข้อจำกัดนี้ Server สามารถสร้างและกระจาย VLAN ได้เต็มช่วง 1-4094

### 3. รองรับ Private VLAN (PVLAN)

VTPv3 สามารถกระจายข้อมูล Private VLAN Configuration (Primary/Isolated/Community VLAN
mapping) ไปทั้งโดเมนได้ ซึ่ง v1/v2 ทำไม่ได้เลย (Private VLAN เรียนละเอียดใน Part CCNP)

### 4. รองรับ MST (Multiple Spanning Tree) Instance Database

VTPv3 กระจายได้ทั้ง 2 ฐานข้อมูล (Instance) แยกกัน: **VLAN Database** และ **MST Database**
ผ่าน Feature เดียวกัน ทำให้ MST Region Configuration Sync กันได้ง่ายขึ้น (ดู Part 5)

### สรุปเปรียบเทียบสั้นๆ

| | VTPv1/v2 | VTPv3 |
|---|---|---|
| ใครมีสิทธิ์แก้ VLAN ได้จริง | Server ตัวไหนก็ได้ | เฉพาะ Primary Server เท่านั้น |
| ป้องกัน VTP Bomb | ไม่ป้องกัน | ป้องกันได้จริง |
| Extended VLAN (1006-4094) | ไม่รองรับ (server/client) | รองรับเต็มรูปแบบ |
| Private VLAN | ไม่รองรับ | รองรับ |
| เปิดใช้งาน | Default v1 | ต้อง config เอง (`vtp version 3`) |

> แม้ VTPv3 จะแก้ปัญหาได้ดีกว่า แต่ Enterprise จำนวนมาก (รวมถึงหลักสูตรนี้) ยังเลือกใช้
> **VTP Transparent Mode** เป็นมาตรฐานอยู่ดี เพราะขจัดความเสี่ยงเรื่อง Auto-Sync ได้ 100%
> โดยไม่ต้องพึ่งพา Feature ใดๆ เลย แลกกับต้องสร้าง VLAN เองทีละเครื่อง (ซึ่งใน Production
> จริงจะใช้ Automation เช่น Ansible/Python จัดการแทนการพิมพ์มือ — เรียนใน Part 47-48)

---

## Step 39 — Troubleshooting Trunk/VTP

### คำสั่ง Verify หลักที่ต้องใช้

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show interfaces trunk` | สรุป Port ที่เป็น Trunk ทั้งหมด, Native VLAN, Allowed VLAN, VLAN ที่ Active |
| `show interfaces <if> switchport` | รายละเอียด Mode/Encapsulation/Native/Trunking VLANs ของ Port เดียว |
| `show dtp interface <if>` | สถานะ DTP negotiation ของ Port นั้น |
| `show vtp status` | VTP Mode, Version, Domain, Revision Number, จำนวน VLAN |
| `show vtp password` | ตรวจสอบว่าตั้ง VTP Password ไว้หรือยัง (ไม่แสดง plaintext) |
| `show vtp counters` | นับจำนวน Advertisement ที่ส่ง/รับ/error — ช่วยดูว่า VTP กำลังคุยกันจริงหรือไม่ |
| `show cdp neighbors detail` | เช็ค Native VLAN Mismatch Warning (แสดงใน log ด้วย) |

### ตัวอย่าง Output และการอ่านค่า

```
DIST-SW1# show interfaces GigabitEthernet0/1 switchport
Name: Gi0/1
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 999 (NATIVE-UNUSED)
Administrative Native VLAN tagging: enabled
Trunking VLANs Enabled: 10,20,30,40,99,999
Pruning VLANs Enabled: 2-1001
```

```
CORE-SW1# show vtp status
VTP Version capable             : 1 to 3
VTP version running             : 2
VTP Domain Name                 : LABCORP
VTP Pruning Mode                : Enabled
VTP Traps Generation            : Disabled
Device ID                       : 00aa.bb00.0001
Configuration last modified by 10.10.99.1 at 9-26-26 10:15:22
Local updater ID is 10.10.99.1 on interface Lo0

Feature VLAN:
--------------
VTP Operating Mode                : Transparent
Maximum VLANs supported locally   : 1005
Number of existing VLANs          : 10
Configuration Revision            : 0
MD5 digest                        : 0x1A 0x2B 0x3C 0x4D 0x5E 0x6F 0x70 0x81
```

> สังเกต **Configuration Revision = 0** และ **VTP Operating Mode = Transparent** — นี่คือค่าที่
> ต้องเห็นบน**ทุกเครื่อง**ใน Lab นี้เสมอ ตามมาตรฐานที่กำหนดใน Step 36/40 ถ้าเจอ Revision ไม่ใช่ 0
> หรือ Mode ไม่ใช่ Transparent บนเครื่องใดเครื่องหนึ่งโดยไม่ได้ตั้งใจ = สัญญาณเตือนต้องรีบตรวจสอบ

### ปัญหาที่พบบ่อยที่สุด และวิธีแก้

| อาการ | สาเหตุที่เป็นไปได้ | คำสั่งวินิจฉัย |
|---|---|---|
| Trunk ไม่ขึ้น (`show int trunk` ไม่แสดง Port นั้นเลย) | ฝั่งหนึ่งเป็น `access` อีกฝั่งเป็น `trunk`/`dynamic` ที่เจรจาไม่สำเร็จ | `show interfaces <if> switchport` ทั้งสองฝั่ง เทียบ Administrative Mode |
| Trunk ขึ้นแต่ VLAN บางใบไม่ผ่าน | `switchport trunk allowed vlan` ไม่ได้เพิ่ม VLAN นั้น หรือ VLAN ถูก Prune | `show interfaces trunk` เทียบคอลัมน์ "allowed" กับ "allowed and active" |
| `%CDP-4-NATIVE_VLAN_MISMATCH` ใน log | Native VLAN ตั้งไม่ตรงกันสองฝั่ง | `show interfaces <if> switchport` ดู "Trunking Native Mode VLAN" |
| Switch ใหม่เข้าร่วมแล้ว VLAN หายทั้งหมด | VTP Bomb — Revision Number สูงกว่าจาก Switch ใหม่ overwrite | `show vtp status` ดู Revision ก่อนเสียบ Trunk ทุกครั้ง |
| VTP Advertisement ไม่ sync ข้าม Switch เลย | VTP Domain Name พิมพ์ไม่ตรงกัน (Case-Sensitive) หรือ VTP Password ไม่ตรง | `show vtp status`, `show vtp password` เทียบทุกเครื่อง |
| Switch v1 ไม่ยอมเข้าร่วมโดเมน v3 | VTP Version ไม่ compatible | `show vtp status` ดู "VTP version running" |
| Encapsulation mismatch error | Switch รุ่นเก่าฝั่งหนึ่งตั้ง `encapsulation isl` อีกฝั่งตั้ง `dot1q` | `show interfaces <if> switchport` ดู "Operational Trunking Encapsulation" |

---

## Step 40 — Lab: Config Trunk ทุกลิงก์ + VTP Transparent Mode ทั้ง Topology

Lab นี้รวบยอด Step 31-39 เข้าด้วยกัน: ตั้งค่า **Trunk 802.1Q ทุกลิงก์** ใน Topology ตาม
Interface Mapping ของ Step 33 พร้อม **VTP Transparent Mode** (แนวทาง Enterprise ที่แนะนำ)
ทั่วทั้ง Lab เพื่อป้องกัน VTP Bomb 100% โดยยังคง VTP Password ไว้เผื่ออนาคตต้องการเปิด
Server/Client Mode

### Global Template ที่ใช้กับทุก Switch ก่อนตั้ง Trunk

```
Switch(config)# vlan 999
Switch(config-vlan)# name NATIVE-UNUSED
Switch(config-vlan)# exit
Switch(config)# vtp domain LABCORP
Switch(config)# vtp password Cisco@VTP123
Switch(config)# vtp mode transparent
Switch(config)# vtp pruning
% Pruning mode ignored (VTP is in transparent mode). ← ปกติ: Pruning ไม่มีผลใน Transparent
```

### Full Config: CORE-SW2 (คู่กับ CORE-SW1 ที่ทำไปแล้วใน Step 33)

```
CORE-SW2(config)# vlan 999
CORE-SW2(config-vlan)# name NATIVE-UNUSED
CORE-SW2(config-vlan)# exit
CORE-SW2(config)# vtp domain LABCORP
CORE-SW2(config)# vtp password Cisco@VTP123
CORE-SW2(config)# vtp mode transparent

CORE-SW2(config)# interface range GigabitEthernet0/1-2
CORE-SW2(config-if-range)# description ** LACP Po1 member to CORE-SW1 **
CORE-SW2(config-if-range)# switchport trunk encapsulation dot1q
CORE-SW2(config-if-range)# switchport mode trunk
CORE-SW2(config-if-range)# switchport trunk native vlan 999
CORE-SW2(config-if-range)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW2(config-if-range)# switchport nonegotiate
CORE-SW2(config-if-range)# exit

CORE-SW2(config)# interface GigabitEthernet0/3
CORE-SW2(config-if)# description ** Trunk to DIST-SW1 Gi0/2 **
CORE-SW2(config-if)# switchport trunk encapsulation dot1q
CORE-SW2(config-if)# switchport mode trunk
CORE-SW2(config-if)# switchport trunk native vlan 999
CORE-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW2(config-if)# switchport nonegotiate
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface GigabitEthernet0/4
CORE-SW2(config-if)# description ** Trunk to DIST-SW2 Gi0/2 **
CORE-SW2(config-if)# switchport trunk encapsulation dot1q
CORE-SW2(config-if)# switchport mode trunk
CORE-SW2(config-if)# switchport trunk native vlan 999
CORE-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW2(config-if)# switchport nonegotiate
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface GigabitEthernet0/5
CORE-SW2(config-if)# description ** Trunk to DIST-SW3 Gi0/2 **
CORE-SW2(config-if)# switchport trunk encapsulation dot1q
CORE-SW2(config-if)# switchport mode trunk
CORE-SW2(config-if)# switchport trunk native vlan 999
CORE-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW2(config-if)# switchport nonegotiate
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface GigabitEthernet0/6
CORE-SW2(config-if)# description ** Trunk to DIST-SW4 Gi0/2 **
CORE-SW2(config-if)# switchport trunk encapsulation dot1q
CORE-SW2(config-if)# switchport mode trunk
CORE-SW2(config-if)# switchport trunk native vlan 999
CORE-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
CORE-SW2(config-if)# switchport nonegotiate
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
```

### Full Config: DIST-SW2, DIST-SW3, DIST-SW4 (รูปแบบเดียวกับ DIST-SW1 ใน Step 33)

```
! ===== DIST-SW2 =====
DIST-SW2(config)# vlan 999
DIST-SW2(config-vlan)# name NATIVE-UNUSED
DIST-SW2(config-vlan)# exit
DIST-SW2(config)# vtp domain LABCORP
DIST-SW2(config)# vtp password Cisco@VTP123
DIST-SW2(config)# vtp mode transparent

DIST-SW2(config)# interface GigabitEthernet0/1
DIST-SW2(config-if)# description ** Trunk to CORE-SW1 Gi0/4 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk native vlan 999
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW2(config-if)# switchport nonegotiate
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface GigabitEthernet0/2
DIST-SW2(config-if)# description ** Trunk to CORE-SW2 Gi0/4 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk native vlan 999
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW2(config-if)# switchport nonegotiate
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface GigabitEthernet0/3
DIST-SW2(config-if)# description ** Trunk to ACCESS-SW1 Gi0/2 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk native vlan 999
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,99,999
DIST-SW2(config-if)# switchport nonegotiate
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface GigabitEthernet0/4
DIST-SW2(config-if)# description ** Trunk to ACCESS-SW2 Gi0/2 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk native vlan 999
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,99,999
DIST-SW2(config-if)# switchport nonegotiate
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

! ===== DIST-SW3 (ขาลง ACCESS-SW3/4 อนุญาตเฉพาะ VLAN 30,40,99) =====
DIST-SW3(config)# vlan 999
DIST-SW3(config-vlan)# name NATIVE-UNUSED
DIST-SW3(config-vlan)# exit
DIST-SW3(config)# vtp domain LABCORP
DIST-SW3(config)# vtp password Cisco@VTP123
DIST-SW3(config)# vtp mode transparent

DIST-SW3(config)# interface GigabitEthernet0/1
DIST-SW3(config-if)# description ** Trunk to CORE-SW1 Gi0/5 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk native vlan 999
DIST-SW3(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW3(config-if)# switchport nonegotiate
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet0/2
DIST-SW3(config-if)# description ** Trunk to CORE-SW2 Gi0/5 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk native vlan 999
DIST-SW3(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW3(config-if)# switchport nonegotiate
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet0/3
DIST-SW3(config-if)# description ** Trunk to ACCESS-SW3 Gi0/1 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk native vlan 999
DIST-SW3(config-if)# switchport trunk allowed vlan 30,40,99,999
DIST-SW3(config-if)# switchport nonegotiate
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet0/4
DIST-SW3(config-if)# description ** Trunk to ACCESS-SW4 Gi0/1 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk native vlan 999
DIST-SW3(config-if)# switchport trunk allowed vlan 30,40,99,999
DIST-SW3(config-if)# switchport nonegotiate
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

! ===== DIST-SW4 =====
DIST-SW4(config)# vlan 999
DIST-SW4(config-vlan)# name NATIVE-UNUSED
DIST-SW4(config-vlan)# exit
DIST-SW4(config)# vtp domain LABCORP
DIST-SW4(config)# vtp password Cisco@VTP123
DIST-SW4(config)# vtp mode transparent

DIST-SW4(config)# interface GigabitEthernet0/1
DIST-SW4(config-if)# description ** Trunk to CORE-SW1 Gi0/6 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk native vlan 999
DIST-SW4(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW4(config-if)# switchport nonegotiate
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet0/2
DIST-SW4(config-if)# description ** Trunk to CORE-SW2 Gi0/6 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk native vlan 999
DIST-SW4(config-if)# switchport trunk allowed vlan 10,20,30,40,99,999
DIST-SW4(config-if)# switchport nonegotiate
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet0/3
DIST-SW4(config-if)# description ** Trunk to ACCESS-SW3 Gi0/2 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk native vlan 999
DIST-SW4(config-if)# switchport trunk allowed vlan 30,40,99,999
DIST-SW4(config-if)# switchport nonegotiate
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet0/4
DIST-SW4(config-if)# description ** Trunk to ACCESS-SW4 Gi0/2 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk native vlan 999
DIST-SW4(config-if)# switchport trunk allowed vlan 30,40,99,999
DIST-SW4(config-if)# switchport nonegotiate
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit
```

### Full Config: ACCESS-SW1–4 (ฝั่ง Uplink Trunk ขึ้น Dist — Access Port ทำแล้วใน Part 3)

```
! ===== ACCESS-SW1 (Uplink ไป DIST-SW1 และ DIST-SW2) =====
ACCESS-SW1(config)# vlan 999
ACCESS-SW1(config-vlan)# name NATIVE-UNUSED
ACCESS-SW1(config-vlan)# exit
ACCESS-SW1(config)# vtp domain LABCORP
ACCESS-SW1(config)# vtp password Cisco@VTP123
ACCESS-SW1(config)# vtp mode transparent

ACCESS-SW1(config)# interface GigabitEthernet0/1
ACCESS-SW1(config-if)# description ** Trunk to DIST-SW1 Gi0/3 **
ACCESS-SW1(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW1(config-if)# switchport mode trunk
ACCESS-SW1(config-if)# switchport trunk native vlan 999
ACCESS-SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
ACCESS-SW1(config-if)# switchport nonegotiate
ACCESS-SW1(config-if)# no shutdown
ACCESS-SW1(config-if)# exit

ACCESS-SW1(config)# interface GigabitEthernet0/2
ACCESS-SW1(config-if)# description ** Trunk to DIST-SW2 Gi0/3 **
ACCESS-SW1(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW1(config-if)# switchport mode trunk
ACCESS-SW1(config-if)# switchport trunk native vlan 999
ACCESS-SW1(config-if)# switchport trunk allowed vlan 10,20,99,999
ACCESS-SW1(config-if)# switchport nonegotiate
ACCESS-SW1(config-if)# no shutdown
ACCESS-SW1(config-if)# exit

! ===== ACCESS-SW2 (Uplink ไป DIST-SW1 และ DIST-SW2) =====
ACCESS-SW2(config)# vlan 999
ACCESS-SW2(config-vlan)# name NATIVE-UNUSED
ACCESS-SW2(config-vlan)# exit
ACCESS-SW2(config)# vtp domain LABCORP
ACCESS-SW2(config)# vtp password Cisco@VTP123
ACCESS-SW2(config)# vtp mode transparent

ACCESS-SW2(config)# interface GigabitEthernet0/1
ACCESS-SW2(config-if)# description ** Trunk to DIST-SW1 Gi0/4 **
ACCESS-SW2(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW2(config-if)# switchport mode trunk
ACCESS-SW2(config-if)# switchport trunk native vlan 999
ACCESS-SW2(config-if)# switchport trunk allowed vlan 10,20,99,999
ACCESS-SW2(config-if)# switchport nonegotiate
ACCESS-SW2(config-if)# no shutdown
ACCESS-SW2(config-if)# exit

ACCESS-SW2(config)# interface GigabitEthernet0/2
ACCESS-SW2(config-if)# description ** Trunk to DIST-SW2 Gi0/4 **
ACCESS-SW2(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW2(config-if)# switchport mode trunk
ACCESS-SW2(config-if)# switchport trunk native vlan 999
ACCESS-SW2(config-if)# switchport trunk allowed vlan 10,20,99,999
ACCESS-SW2(config-if)# switchport nonegotiate
ACCESS-SW2(config-if)# no shutdown
ACCESS-SW2(config-if)# exit

! ===== ACCESS-SW3 (Uplink ไป DIST-SW3 และ DIST-SW4) =====
ACCESS-SW3(config)# vlan 999
ACCESS-SW3(config-vlan)# name NATIVE-UNUSED
ACCESS-SW3(config-vlan)# exit
ACCESS-SW3(config)# vtp domain LABCORP
ACCESS-SW3(config)# vtp password Cisco@VTP123
ACCESS-SW3(config)# vtp mode transparent

ACCESS-SW3(config)# interface GigabitEthernet0/1
ACCESS-SW3(config-if)# description ** Trunk to DIST-SW3 Gi0/3 **
ACCESS-SW3(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW3(config-if)# switchport mode trunk
ACCESS-SW3(config-if)# switchport trunk native vlan 999
ACCESS-SW3(config-if)# switchport trunk allowed vlan 30,40,99,999
ACCESS-SW3(config-if)# switchport nonegotiate
ACCESS-SW3(config-if)# no shutdown
ACCESS-SW3(config-if)# exit

ACCESS-SW3(config)# interface GigabitEthernet0/2
ACCESS-SW3(config-if)# description ** Trunk to DIST-SW4 Gi0/3 **
ACCESS-SW3(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW3(config-if)# switchport mode trunk
ACCESS-SW3(config-if)# switchport trunk native vlan 999
ACCESS-SW3(config-if)# switchport trunk allowed vlan 30,40,99,999
ACCESS-SW3(config-if)# switchport nonegotiate
ACCESS-SW3(config-if)# no shutdown
ACCESS-SW3(config-if)# exit

! ===== ACCESS-SW4 (Uplink ไป DIST-SW3 และ DIST-SW4) =====
ACCESS-SW4(config)# vlan 999
ACCESS-SW4(config-vlan)# name NATIVE-UNUSED
ACCESS-SW4(config-vlan)# exit
ACCESS-SW4(config)# vtp domain LABCORP
ACCESS-SW4(config)# vtp password Cisco@VTP123
ACCESS-SW4(config)# vtp mode transparent

ACCESS-SW4(config)# interface GigabitEthernet0/1
ACCESS-SW4(config-if)# description ** Trunk to DIST-SW3 Gi0/4 **
ACCESS-SW4(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW4(config-if)# switchport mode trunk
ACCESS-SW4(config-if)# switchport trunk native vlan 999
ACCESS-SW4(config-if)# switchport trunk allowed vlan 30,40,99,999
ACCESS-SW4(config-if)# switchport nonegotiate
ACCESS-SW4(config-if)# no shutdown
ACCESS-SW4(config-if)# exit

ACCESS-SW4(config)# interface GigabitEthernet0/2
ACCESS-SW4(config-if)# description ** Trunk to DIST-SW4 Gi0/4 **
ACCESS-SW4(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW4(config-if)# switchport mode trunk
ACCESS-SW4(config-if)# switchport trunk native vlan 999
ACCESS-SW4(config-if)# switchport trunk allowed vlan 30,40,99,999
ACCESS-SW4(config-if)# switchport nonegotiate
ACCESS-SW4(config-if)# no shutdown
ACCESS-SW4(config-if)# exit

! ปิดท้ายทุกเครื่อง
ACCESS-SW4(config)# end
ACCESS-SW4# copy running-config startup-config
```

### Verification สุดท้าย — ตรวจสอบทั้ง Topology

```
CORE-SW1# show vtp status
VTP Version running             : 2
VTP Domain Name                 : LABCORP
VTP Operating Mode              : Transparent
Configuration Revision          : 0
Number of existing VLANs        : 6

CORE-SW1# show interfaces trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      999
Gi0/2       on               802.1q         trunking      999
Gi0/3       on               802.1q         trunking      999
Gi0/4       on               802.1q         trunking      999
Gi0/5       on               802.1q         trunking      999
Gi0/6       on               802.1q         trunking      999

DIST-SW1# show interfaces trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      999
Gi0/2       on               802.1q         trunking      999
Gi0/3       on               802.1q         trunking      999
Gi0/4       on               802.1q         trunking      999

ACCESS-SW1# show interfaces trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      999
Gi0/2       on               802.1q         trunking      999

ACCESS-SW1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   SALES                            active    Gi0/3, Gi0/4, Gi0/5
20   VOICE                            active    Gi0/6, Gi0/7
99   MGMT                             active    Gi0/24
999  NATIVE-UNUSED                    active
```

> **สังเกต**: ACCESS-SW1 ไม่มี VLAN 30 (SERVERS) และ 40 (WIFI) ในตัวเองเลย (ถูกจำกัดไว้ตาม
> Part 3 และ `allowed vlan` ของ Trunk ก็ไม่อนุญาตให้ VLAN เหล่านั้นผ่านลงมาด้วย) — ตรงตาม
> Design Intent 100%: Trunk เปิดกว้างเท่าที่จำเป็นเท่านั้น ไม่ใช่เปิด "all" แบบ default

---

## แบบฝึกหัดทวนความเข้าใจ Part 4

1. 802.1Q Tag มีขนาดกี่ byte และ EtherType ที่ใช้ระบุว่า Frame มี Tag คือค่าอะไร?
2. เพราะเหตุใดการใช้ VLAN 1 เป็น Native VLAN จึงเป็นความเสี่ยงด้านความปลอดภัย และหลักสูตรนี้
   แก้ปัญหานี้ด้วยการทำอย่างไร?
3. ถ้า Port ฝั่ง A ตั้งเป็น `switchport mode dynamic desirable` และฝั่ง B ตั้งเป็น
   `switchport mode dynamic auto` ผลลัพธ์ Link State จะเป็นอย่างไร?
4. อธิบายเหตุการณ์ "VTP Bomb" และวิธีป้องกันอย่างน้อย 2 วิธี
5. VTP Pruning ทำงานได้หรือไม่ ถ้าทั้งโดเมนตั้ง VTP Mode เป็น Transparent ทั้งหมด เพราะเหตุใด?

**เฉลย:**
1. 4 byte, EtherType/TPID = **0x8100**
2. เพราะ VLAN 1 เป็นค่า Default ของทุกอย่างและมี Control Plane Traffic จำนวนมากวิ่งอยู่ อีกทั้งเสี่ยงต่อ VLAN Hopping แบบ Double Tagging — หลักสูตรนี้แก้ด้วยการเปลี่ยน Native VLAN ทุก Trunk เป็น **VLAN 999 (NATIVE-UNUSED)** ที่ไม่มี Access Port ใดสังกัดอยู่เลย
3. **Trunk ขึ้น** เพราะ Dynamic Desirable จะ "ชวน" อีกฝั่งอยู่ตลอด และ Dynamic Auto จะตอบรับเมื่อมีฝั่งใดฝั่งหนึ่งชวนก่อน
4. VTP Bomb คือ Switch ที่มี VTP Revision Number สูงกว่า (แม้จะมีฐานข้อมูล VLAN ที่ผิด/ว่างเปล่า) เข้าร่วมโดเมนแล้ว Overwrite ฐานข้อมูล VLAN ของทั้งเครือข่ายทันที เพราะกฎ "Revision สูงกว่าชนะเสมอ" — ป้องกันได้โดย (1) ใช้ VTP Transparent Mode ทั่วทั้งองค์กร (2) Reset Revision Number ของ Switch ใหม่ก่อนเสียบเข้าเครือข่ายจริงเสมอ (หรือใช้ VTPv3 ที่มี Primary Server Election)
5. **ไม่ทำงาน** เพราะ VTP Pruning เป็น Feature ที่ทำงานผ่านกลไก VTP Server/Client Advertisement เท่านั้น เมื่อ Switch ทั้งหมดอยู่ใน Transparent Mode จะไม่มีการ participate ใน VTP Domain เลย ต้องพึ่ง `switchport trunk allowed vlan` แบบ Manual แทน

---

## สรุป Part 4

Part นี้ปูพื้นฐาน **802.1Q Trunking** ตั้งแต่โครงสร้าง Tag ระดับ byte, Native VLAN และความเสี่ยง
ด้านความปลอดภัย, การ Config Trunk จริงทุกลิงก์ใน Topology, **DTP** และเหตุผลที่ต้องปิดด้วย
`nonegotiate`, ไปจนถึง **VTP** ทั้งพื้นฐาน Mode/Version/Pruning และหายนะ "VTP Bomb" ที่ต้อง
ระวังในเครือข่ายจริง ปิดท้ายด้วย Lab เต็มรูปแบบที่ตั้งค่า Trunk + VTP Transparent Mode ครบทุก
อุปกรณ์ใน Lab Topology ✅

**ไปต่อ:** [Part 5 — STP / RSTP / MSTP →](part-005-stp-rstp-mstp.md)
