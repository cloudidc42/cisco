# Part 21 — Switch Security (Port Security, DHCP Snooping, DAI)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 201–210 จาก 1000**

> ต่อจาก [Part 20 — Wireless Fundamentals (WLC, AP, SSID)](part-020-wireless-fundamentals.md)
> ที่ขยาย Access Layer ของ Enterprise Lab ไปสู่โลกไร้สาย Part นี้จะกลับมาปิดช่องโหว่ด้าน
> ความปลอดภัยที่ **Layer 2** ของ Access Layer ที่ค้างไว้ตั้งใจจากสอง Part ก่อนหน้า: (1)
> [Part 16 — DHCP/DNS](part-016-dhcp-dns.md) **Step 155** แนะนำแนวคิด DHCP Snooping
> (Trusted/Untrusted Port) ไว้แค่ระดับ Concept และบอกไว้ชัดเจนว่า Config เต็มรูปแบบจะอยู่ใน
> Part นี้ และ (2) [Part 4 — Trunking, VTP, DTP](part-004-trunking-vtp-dtp.md) **Step 32**
> ที่อธิบาย Native VLAN Double-Tagging ไว้ในบริบทของ Trunk Security เท่านั้น — Part นี้จะรวม
> ฟีเจอร์ความปลอดภัยระดับ Access Layer ที่เหลือทั้งหมดเข้าด้วยกัน: **Port Security, DHCP
> Snooping, Dynamic ARP Inspection (DAI), IP Source Guard, และ Storm Control** เพื่อสร้าง
> **Layer 2 Security Baseline ฉบับสมบูรณ์** ที่จะถูกอ้างอิงเป็นมาตรฐานของทุก Switch ในหลักสูตร
> ตั้งแต่ Part นี้เป็นต้นไป

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 201 | Layer 2 Threats ที่ต้องรู้ — ตารางสรุป Mechanism / Impact / Mitigation |
| 202 | Port Security พื้นฐาน — `switchport port-security`, Maximum, Violation Mode, Sticky MAC |
| 203 | Port Security Violation Handling เชิงลึก และ Err-Disable Recovery |
| 204 | DHCP Snooping พื้นฐาน — Trusted/Untrusted, Binding Table, Rate-Limiting |
| 205 | Lab เต็มรูปแบบ: DHCP Snooping บน ACCESS-SW1–4 และ DIST-SW1–4 |
| 206 | Dynamic ARP Inspection (DAI) พื้นฐาน — ใช้ Binding Table ตรวจสอบ ARP |
| 207 | คอนฟิก DAI, Rate-Limiting, และ ARP ACL สำหรับ Host ที่ตั้ง Static IP |
| 208 | IP Source Guard — ขาที่สามของ Trio ความปลอดภัยบน DHCP Snooping Binding Table |
| 209 | Storm Control — ป้องกัน Broadcast/Multicast/Unicast Storm |
| 210 | Lab เต็มรูปแบบ: Layer 2 Security Baseline ทุกอุปกรณ์ + การตรวจสอบผลลัพธ์ |

---

## Step 201 — Layer 2 Threats ที่ต้องรู้สำหรับ CCNA/CCNP

### 201.1 ทำไม Layer 2 Security สำคัญไม่แพ้ Layer 3

ตลอด Part 1-20 หลักสูตรนี้ให้ความสำคัญกับ Layer 3 Security (ACL ใน Part 14, NAT ใน Part 15)
และ Layer 2 Availability (STP ใน Part 5, EtherChannel ใน Part 6) เป็นหลัก แต่ยังไม่ได้พูดถึง
**Layer 2 Security** อย่างเป็นระบบ — ทั้งที่ Access Port ของ ACCESS-SW1–4 คือจุดที่ End-user
เสียบสายเข้าเครือข่ายได้โดยตรง ไม่มี Firewall/ACL คั่นเหมือนที่ WAN Edge (FW-1, Part 43-45)

Protocol พื้นฐานของ Layer 2 ส่วนใหญ่ (Ethernet, ARP, DHCP) ถูกออกแบบมาในยุคที่เครือข่ายท้องถิ่น
ถือเป็น "Trusted Environment" ทั้งหมด และ**ไม่มี Authentication ในตัว Protocol เอง** — Switch
มาตรฐาน (ไม่เปิดฟีเจอร์เพิ่ม) จะไม่ตรวจสอบความถูกต้องของ Source MAC, DHCP Message, หรือ ARP
Reply ที่รับเข้ามาเลย ฟีเจอร์ในตารางด้านล่างคือชุดกลไกที่ Cisco เพิ่มเข้ามาเสริมทับ Protocol
เดิมเหล่านี้ เพื่อปิดช่องว่างด้าน Authentication/Validation ที่ Protocol ดั้งเดิมไม่มี

### 201.2 ตารางสรุป Layer 2 Threats หลัก — Mechanism / Impact / Mitigation

| Threat | Mechanism (กลไกทางเทคนิค) | Impact (ผลกระทบ) | Cisco Mitigation |
|---|---|---|---|
| **MAC Flooding (CAM Table Overflow)** | Source MAC Address จำนวนมหาศาลถูกส่งเข้าพอร์ตเดียวในเวลาสั้น ทำให้ CAM Table (MAC Address Table) ของ ASIC เต็ม (Catalyst ทั่วไปรองรับ ~8,000-32,000 entries) | เมื่อ CAM Table เต็ม Switch เข้าสู่ **Fail-Open Mode** — Frame ที่ไม่พบ Destination MAC ในตารางจะถูก **Flood ออกทุกพอร์ตใน VLAN เดียวกัน** (พฤติกรรมเหมือน Hub/Layer-1 Repeater) ทำให้ Traffic ที่ควรไปถึงปลายทางเดียวถูกมองเห็นได้จากทุกพอร์ตในบรอดคาสต์โดเมนนั้น | **Port Security** (Step 202-203) — จำกัดจำนวน MAC Address ที่เรียนรู้ได้ต่อพอร์ต ป้องกันไม่ให้พอร์ตเดียวส่ง Source MAC ได้ไม่จำกัด |
| **Rogue DHCP Server** | อุปกรณ์ที่รัน DHCP Server (ตั้งใจหรือไม่ตั้งใจ) เชื่อมต่อเข้า Access Port แล้วตอบ `DHCPOFFER` แข่งกับ Server ที่ถูกต้อง (DHCP-DNS-SRV) | RFC 2131 กำหนดให้ Client รับ Offer ใบแรกที่มาถึงโดยไม่มีการยืนยันแหล่งที่มา — ถ้า Client ได้รับ Gateway/DNS ที่ชี้ไปยังอุปกรณ์ที่ไม่ถูกต้อง เกิดความเสี่ยง **Man-in-the-Middle** เต็มรูปแบบ หรือ Denial of Service (แจก Scope ไม่ถูกต้องทำให้ Client ใช้งานเครือข่ายไม่ได้) | **DHCP Snooping** (Step 204-205) — บล็อก `DHCPOFFER`/`DHCPACK` ที่มาจากพอร์ตที่ไม่ได้กำหนดเป็น Trusted |
| **ARP Spoofing / ARP Cache Poisoning** | ส่ง ARP Reply (มักเป็น Gratuitous ARP) ที่ประกาศ IP-to-MAC Binding ปลอมซ้ำๆ เพื่อ Poison ARP Cache ของ Host เป้าหมายให้เชื่อว่า MAC ของผู้ส่งคือเจ้าของ IP ของ Default Gateway หรือ Host อื่น | Host เป้าหมายส่ง Traffic ทั้งหมดไปยัง MAC ปลอมก่อน ทำให้เกิด **Man-in-the-Middle** (ดัก/แก้ไข Traffic ก่อนส่งต่อ) หรือ Denial of Service (ไม่ส่งต่อเลย) เพราะ ARP Protocol (RFC 826) ไม่มีการยืนยันตัวตนผู้ส่งเลย | **Dynamic ARP Inspection — DAI** (Step 206-207) — ตรวจสอบ ARP Packet ทุกใบเทียบกับ DHCP Snooping Binding Table |
| **VLAN Hopping — Double Tagging** | ส่ง Frame ที่มี 802.1Q Tag ซ้อนกัน 2 ชั้น (Outer Tag = Native VLAN ของ Trunk ตัวแรก, Inner Tag = VLAN เป้าหมาย) จาก Access Port ที่ Native VLAN ของ Trunk ปลายทางตรงกับ VLAN ของพอร์ตนั้น | Switch ตัวแรกดึง (Strip) Outer Tag ออกเพราะเข้าใจว่าเป็น Native VLAN (ไม่ Tag) แล้วส่ง Frame ที่ยังมี Inner Tag ต่อไปยัง Trunk — Switch ปลายทางตีความ Inner Tag เป็น VLAN จริงทันที ทำให้ Frame เข้าสู่ VLAN เป้าหมายได้แบบ **ทางเดียว (One-way)** โดยไม่ต้องเป็นสมาชิกของ VLAN นั้นจริง (อธิบายไว้แล้วใน [Part 4 Step 32](part-004-trunking-vtp-dtp.md)) | เปลี่ยน Native VLAN เป็น VLAN ที่ไม่มี Access Port ใดสังกัด (VLAN 999 — ทำแล้วทุก Trunk ใน Lab), ห้าม Access Port ใดอยู่ใน Native VLAN ของ Trunk |
| **VLAN Hopping — Switch Spoofing (DTP)** | Host ส่ง DTP (Dynamic Trunking Protocol) Frame ปลอม อ้างว่าตนเองเป็น Switch ที่รองรับ Trunk (จำลอง `dynamic desirable`) เพื่อเจรจาให้ Access Port ของ Switch ที่ตั้งเป็น `dynamic auto`/`desirable` เปลี่ยนเป็น Trunk จริง | ถ้าเจรจาสำเร็จ ผู้ที่เชื่อมต่ออยู่จะได้รับ Trunk Port เต็มรูปแบบ มองเห็นทุก VLAN ที่ Trunk อนุญาต โดยไม่ต้อง Hop ทีละ VLAN แบบ Double Tagging | `switchport mode access` + `switchport nonegotiate` แบบตายตัวบน**ทุก Access Port** (ทำแล้วใน [Part 4 Step 34](part-004-trunking-vtp-dtp.md)) — ห้ามปล่อยเป็น `dynamic auto`/`desirable` |
| **Broadcast / Multicast / Unicast Storm** | Loop ทาง Physical โดยไม่ตั้งใจที่ STP ยังไม่ Converge ทัน หรือ Traffic Broadcast/Multicast/Unknown-Unicast ปริมาณสูงผิดปกติจากอุปกรณ์ตัวใดตัวหนึ่ง | ใช้ Bandwidth ของ Link จนหมดอย่างรวดเร็ว (Broadcast ถูก Flood ทุกพอร์ตใน VLAN เสมอ) ทำให้ Network ทั้ง VLAN ใช้งานไม่ได้ (Broadcast Storm) แม้ STP (Part 5) จะทำหน้าที่ป้องกัน Loop ทาง Topology แต่ไม่ได้จำกัดปริมาณ Traffic ต่อวินาที | **Storm Control** (Step 209) — จำกัด % Bandwidth ของ Broadcast/Multicast/Unicast ต่อพอร์ต ทำงานเป็นเกราะป้องกันชั้นที่สองคู่กับ STP |

> **ตารางนี้คือแผนที่ของ Part นี้ทั้งหมด** — สังเกตว่าคอลัมน์ Mitigation ขวาสุดคือลำดับ Step
> ที่จะอธิบายรายละเอียด Config จริงตั้งแต่ Step 202 เป็นต้นไป และ Step 210 ปิดท้ายด้วยการนำ
> ทุกฟีเจอร์มารวมกันเป็น **Baseline เดียว** ที่ใช้กับทุก Switch ใน Lab Topology

### 201.3 ความสัมพันธ์ระหว่างฟีเจอร์ — ทำไมต้องเรียงลำดับ Config ตาม Step

Port Security, DHCP Snooping, DAI, และ IP Source Guard ไม่ใช่ฟีเจอร์แยกกันโดยสิ้นเชิง —
**DHCP Snooping Binding Table** (สร้างขึ้นจาก Step 204-205) เป็นฐานข้อมูลกลางที่ DAI (Step
206-207) และ IP Source Guard (Step 208) ใช้อ้างอิงในการตรวจสอบ ทำให้เกิดลำดับการ Config ที่
**ต้องทำตามลำดับ**:

```
1. Port Security      →  จำกัดจำนวน MAC ต่อพอร์ต (ทำงานอิสระ ไม่ต้องพึ่งฟีเจอร์อื่น)
2. DHCP Snooping       →  สร้าง Binding Table (MAC + IP + VLAN + Interface + Lease)
                          ██████████████████████████████████████████████
                          █         DHCP Snooping Binding Table          █  <- ฐานข้อมูลกลาง
                          ██████████████████████████████████████████████
                                  │                           │
                                  ▼                           ▼
3. Dynamic ARP Inspection    ตรวจ ARP Packet      4. IP Source Guard    ตรวจ Source IP/MAC
   เทียบกับ Binding Table                            ของ Traffic เทียบกับ Binding Table
```

ด้วยเหตุนี้ **DHCP Snooping ต้องเปิดก่อน DAI และ IP Source Guard เสมอ** — ถ้าเปิด DAI หรือ IP
Source Guard โดยไม่มี DHCP Snooping Binding Table ที่ถูกต้อง Traffic ปกติของ Host ที่ได้ IP
ผ่าน DHCP จะถูกบล็อกไปด้วย (เพราะไม่มี Binding ให้ตรวจสอบ) — Storm Control (Step 209) เป็น
ฟีเจอร์เดียวที่ทำงานอิสระไม่พึ่งฐานข้อมูลใดๆ จึงเปิดเมื่อไหร่ก็ได้

---

## Step 202 — Port Security พื้นฐาน

### 202.1 หลักการทำงาน

**Port Security** จำกัดจำนวน MAC Address ที่ Access Port หนึ่งพอร์ตสามารถ "เรียนรู้" (Learn)
เข้าสู่ CAM Table ได้ ป้องกัน MAC Flooding และป้องกันไม่ให้ผู้ใช้เสียบ Switch/Hub ต่อพ่วง
(เพิ่มจำนวนอุปกรณ์หลัง Access Port โดยไม่ได้รับอนุญาต)

> **ข้อจำกัดสำคัญ**: Port Security ใช้ได้เฉพาะ **Access Port หรือ Trunk Port แบบ Static** เท่านั้น
> (ต้องปิด DTP ด้วย `switchport nonegotiate` ก่อนเสมอ ตามที่ทำไว้ใน Part 4) — ใช้กับ
> Port ที่เป็น `dynamic auto`/`desirable` หรือ EtherChannel member ไม่ได้

### 202.2 คำสั่งพื้นฐานและค่า Default

| คำสั่ง | ผลลัพธ์ | ค่า Default ถ้าไม่กำหนด |
|---|---|---|
| `switchport port-security` | เปิดใช้งาน Port Security บน Interface (ต้องเป็น access/trunk แบบ static ก่อน) | ปิดอยู่ |
| `switchport port-security maximum <n>` | จำกัดจำนวน MAC สูงสุดที่เรียนรู้ได้ต่อพอร์ต | 1 |
| `switchport port-security violation {protect \| restrict \| shutdown}` | กำหนดพฤติกรรมเมื่อเกิน Maximum (รายละเอียดเต็มใน Step 203) | shutdown |
| `switchport port-security mac-address <mac>` | กำหนด MAC แบบ Static ล่วงหน้า (ฝังใน running-config เอง) | ไม่มี |
| `switchport port-security mac-address sticky` | เปิดโหมด Sticky — เรียนรู้ MAC แรกที่เห็นแบบ Dynamic แล้ว "แปลง" เป็น Static อัตโนมัติในarunning-config | ปิดอยู่ |
| `switchport port-security aging time <นาที>` | ตั้งเวลาให้ MAC ที่เรียนรู้แบบ Dynamic (ไม่ sticky) หมดอายุ | Aging ปิดอยู่ (MAC ค้างตลอดไปจนกว่า link down) |

### 202.3 Static MAC vs Sticky MAC vs Dynamic MAC — ความแตกต่าง

| ประเภท | วิธีได้มาซึ่ง MAC | เก็บใน running-config หรือไม่ | เหมาะกับ |
|---|---|---|---|
| **Dynamic** (ค่า Default เมื่อเปิด port-security เฉยๆ) | เรียนรู้จาก Frame แรกที่เข้ามาอัตโนมัติ | **ไม่** — หายไปเมื่อ `no shutdown`/reload | Lab ทดสอบเร็วๆ ไม่ต้องคงอยู่ |
| **Static** | Admin พิมพ์ MAC เองด้วย `switchport port-security mac-address <mac>` | ใช่ — อยู่ใน running-config ทันที | รู้ MAC ของอุปกรณ์แน่นอนล่วงหน้า (เช่น Server, Printer ที่ไม่เปลี่ยน NIC) |
| **Sticky** | เรียนรู้อัตโนมัติเหมือน Dynamic แต่ระบบ "จำ" แล้วเขียนเป็น Static command ให้เอง | ใช่ — ปรากฏใน running-config เป็น `switchport port-security mac-address sticky <mac>` โดยอัตโนมัติหลังเรียนรู้ | **มาตรฐานของหลักสูตรนี้** — ง่ายเหมือน Dynamic แต่คงอยู่หลัง reload เหมือน Static |

> **มาตรฐาน Lab ของหลักสูตรนี้**: ใช้ **Sticky MAC** เป็นค่าหลักสำหรับทุก Access Port ของ
> ผู้ใช้ทั่วไป (PC, Phone, AP) เพราะไม่ต้องรู้ MAC ล่วงหน้า แต่ยัง Persist หลัง `write memory`
> ส่วน Access Port ที่ต่อ Server ที่มี MAC คงที่แน่นอน (เช่น DHCP-DNS-SRV) จะใช้ Static MAC
> ตรงตัวเพื่อความชัดเจนสูงสุด (ดู Step 210)

### 202.4 Worked Config — Port Security บน Access Port ของ ACCESS-SW1

```
! ACCESS-SW1 — Access Port ไปยัง PC1 (VLAN 10 - SALES)
ACCESS-SW1(config)# interface GigabitEthernet1/0/1
ACCESS-SW1(config-if)# description ** Access Port - PC1 (VLAN10 SALES) **
ACCESS-SW1(config-if)# switchport mode access
ACCESS-SW1(config-if)# switchport access vlan 10
ACCESS-SW1(config-if)# switchport nonegotiate
ACCESS-SW1(config-if)# switchport port-security
ACCESS-SW1(config-if)# switchport port-security maximum 2
ACCESS-SW1(config-if)# switchport port-security violation restrict
ACCESS-SW1(config-if)# switchport port-security mac-address sticky
ACCESS-SW1(config-if)# spanning-tree portfast
ACCESS-SW1(config-if)# spanning-tree bpduguard enable
ACCESS-SW1(config-if)# no shutdown
ACCESS-SW1(config-if)# exit
```

**เหตุผลที่ Maximum = 2 (ไม่ใช่ 1)**: พอร์ตแต่ละใบใน Lab นี้ต้องรองรับ PC ต่อผ่าน IP Phone
แบบ Daisy-chain ได้ (VLAN 10 ของ PC + VLAN 20 ของ Phone ผ่าน Voice VLAN บนพอร์ตเดียว ตามที่
Config ไว้ใน [Part 16](part-016-dhcp-dns.md)) — ถ้าพอร์ตใดรองรับ PC เดี่ยวไม่มี Phone
ต่อพ่วง ให้ตั้ง Maximum 1 ได้เพื่อความเข้มงวดสูงสุด

**ทำแบบเดียวกันกับทุก Access Port ของ ACCESS-SW1–4** โดยใช้ `interface range` เพื่อลด
การพิมพ์ซ้ำ:

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# switchport mode access
ACCESS-SW1(config-if-range)# switchport nonegotiate
ACCESS-SW1(config-if-range)# switchport port-security
ACCESS-SW1(config-if-range)# switchport port-security maximum 2
ACCESS-SW1(config-if-range)# switchport port-security violation restrict
ACCESS-SW1(config-if-range)# switchport port-security mac-address sticky
ACCESS-SW1(config-if-range)# spanning-tree portfast
ACCESS-SW1(config-if-range)# spanning-tree bpduguard enable
ACCESS-SW1(config-if-range)# exit
```

> **หมายเหตุ**: `interface range` ตั้งค่าที่เหมือนกันได้ (mode, maximum, violation, sticky)
> แต่ **`switchport access vlan <id>` ต้องตั้งทีละพอร์ตหรือทีละกลุ่ม VLAN เดียวกัน** เพราะพอร์ต
> ในช่วงเดียวกันอาจอยู่คนละ VLAN (เช่นพอร์ตของ PC อยู่ VLAN 10 แต่พอร์ตของ AP อยู่ VLAN 99)

### 202.5 คำสั่ง Verify เบื้องต้น

```
ACCESS-SW1# show port-security interface GigabitEthernet1/0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode              : Restrict
Aging Time                  : 0 mins
Aging Type                  : Absolute
SecureStatic Address Aging  : Disabled
Maximum MAC Addresses        : 2
Total MAC Addresses           : 0
Configured MAC Addresses      : 0
Sticky MAC Addresses           : 0
Last Source Address:Vlan       : 0000.0000.0000:0
Security Violation Count        : 0
```

ก่อนที่ PC1 จะส่ง Frame ใดๆ เข้ามา `Total MAC Addresses` และ `Sticky MAC Addresses` จะเป็น 0
— รอ PC1 เปิดเครื่องและส่ง Frame แรก (เช่น DHCPDISCOVER, ARP Gratuitous) MAC จะถูกเรียนรู้
และ running-config จะปรากฏบรรทัดใหม่โดยอัตโนมัติ (`show running-config interface Gi1/0/1`
จะเห็น `switchport port-security mac-address sticky 0050.7966.6801`)

---

## Step 203 — Port Security Violation Handling เชิงลึก และ Err-Disable Recovery

### 203.1 พฤติกรรมของแต่ละ Violation Mode

| Violation Mode | Drop Traffic ที่เกิน Maximum? | Increment Violation Counter? | ส่ง Syslog/SNMP Trap? | ผลต่อสถานะ Port |
|---|---|---|---|---|
| **Protect** | ใช่ — Drop แบบเงียบ | **ไม่** | **ไม่** | Port ยังคง Up ปกติ — เหมาะกับกรณีที่ไม่ต้องการ Log จำนวนมากแต่ต้องการบล็อกอย่างเดียว |
| **Restrict** | ใช่ | **ใช่** | **ใช่** | Port ยังคง Up ปกติ — เหมาะกับ Production ทั่วไปเพราะยังใช้งานได้ต่อพร้อมมี Log ให้ตรวจสอบ |
| **Shutdown** (Default) | ใช่ (ก่อนที่ Port จะถูกปิด) | ใช่ | ใช่ | **Port เข้าสู่ err-disabled state ทันที** — ปิดสนิท ต้องกู้คืนด้วยมือหรือ `errdisable recovery` |

> **ทำไมหลักสูตรนี้เลือก `restrict` เป็นค่ามาตรฐานของ Lab (Step 202-210)**: `shutdown` (ค่า
> Default ของ IOS) เข้มงวดที่สุดแต่ทำให้ Helpdesk ต้องเข้าไป `shutdown`/`no shutdown` มือทุกครั้ง
> ที่มี False Positive (เช่น ผู้ใช้เปลี่ยน NIC Card ทำให้ MAC เปลี่ยน) — `restrict` ยังคง Drop
> Traffic ที่ไม่ได้รับอนุญาตเหมือนกัน แต่ปล่อยให้ MAC ที่ได้รับอนุญาตอยู่แล้วใช้งานต่อได้โดย
> ไม่ต้อง Manual Intervention ทุกครั้ง เหมาะกับ Enterprise ที่ต้องการ Availability สูง

### 203.2 กรณีศึกษา: เมื่อ Maximum ถูกเกิน (Violation Mode = shutdown)

สมมติพอร์ต `GigabitEthernet1/0/5` ตั้ง Maximum 1 ไว้ (ค่า Default) และ Violation Mode =
`shutdown` — เมื่อมี MAC Address ที่สองพยายามส่ง Frame เข้ามาที่พอร์ตนี้ (เช่น ผู้ใช้เสียบ
Switch เล็กต่อพ่วงเพิ่มอุปกรณ์):

```
%PM-4-ERR_DISABLE: psecure-violation error detected on Gi1/0/5, putting Gi1/0/5 in err-disable state
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 00d0.9878.1122 on port GigabitEthernet1/0/5.
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/5, changed state to down
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/5, changed state to down
```

```
ACCESS-SW1# show interfaces GigabitEthernet1/0/5 status
Port      Name               Status       Vlan       Duplex  Speed Type
Gi1/0/5                       err-disabled 10           auto   auto 10/100/1000BaseTX
```

### 203.3 Err-Disable Recovery — กู้คืน Port อัตโนมัติแทนการ `shutdown`/`no shutdown` ด้วยมือ

การเข้าไป `shutdown` แล้ว `no shutdown` ด้วยมือทุกครั้งไม่สเกลกับ Enterprise ที่มีหลักร้อย
พอร์ต — IOS มีฟีเจอร์ **Err-Disable Recovery** ให้ Port กลับมา Up เองอัตโนมัติหลังเวลาที่กำหนด:

```
ACCESS-SW1(config)# errdisable recovery cause psecure-violation
ACCESS-SW1(config)# errdisable recovery interval 300
```

- `errdisable recovery cause psecure-violation` — เปิดให้ Cause นี้โดยเฉพาะ (Port Security
  Violation) กลับมา Up เองได้ (ค่า Default ของทุก Cause คือ**ไม่**เปิด Recovery อัตโนมัติ)
- `errdisable recovery interval 300` — รอ 300 วินาที (5 นาที) แล้ว IOS จะ `no shutdown`
  ให้เอง (ค่า Default ของ Interval คือ 300 วินาทีอยู่แล้ว แต่ระบุไว้ชัดเจนเพื่อ Documentation)

> **Cause อื่นที่นิยมเปิด Recovery คู่กัน**: `errdisable recovery cause bpduguard`,
> `errdisable recovery cause udld`, `errdisable recovery cause link-flap` — ดูรายการทั้งหมด
> ด้วย `show errdisable recovery`

```
ACCESS-SW1# show errdisable recovery
ErrDisable Reason            Timer Status
-----------------------      --------------
udld                          Disabled
bpduguard                     Disabled
psecure-violation             Enabled
...
Timer interval: 300 seconds

Interfaces that will be enabled at the next timeout:
Interface       Errdisable reason      Time left(sec)
---------------  ---------------------  --------------
Gi1/0/5          psecure-violation      237
```

> **ข้อควรระวัง**: Err-Disable Recovery แก้ปัญหาแค่ "อาการ" (Port ถูกปิด) แต่ไม่ได้แก้ต้นตอ
> ถ้าอุปกรณ์ที่ทำให้เกิด Violation ยังเสียบอยู่ พอร์ตจะกลับเข้า err-disabled อีกครั้งทันทีหลัง
> Recovery — ต้องตรวจสอบ Log (`show logging | include PSECURE`) เพื่อหาสาเหตุจริงเสมอ

### 203.4 คำสั่ง Verify Violation ทั้งระบบ

```
ACCESS-SW1# show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
              (Count)          (Count)        (Count)
---------------------------------------------------------------------------
Gi1/0/1            2               1               0             Restrict
Gi1/0/2            2               2               0             Restrict
Gi1/0/5            1               1               3             Shutdown
---------------------------------------------------------------------------
Total Addresses in System (excluding one mac per port)     : 1
Max Addresses limit in System (excluding one mac per port)  : 6144
```

`SecurityViolation (Count)` ของ `Gi1/0/5` = 3 บอกว่าเกิด Violation ไปแล้ว 3 ครั้งสะสม (ตัวเลข
นี้ไม่รีเซ็ตเองจนกว่า Interface Counter จะถูก Clear ด้วย `clear port-security all`) — เป็น
ตัวเลขสำคัญที่ทีม NOC ใช้ระบุพอร์ตที่มีความผิดปกติเกิดขึ้นซ้ำๆ

---

## Step 204 — DHCP Snooping พื้นฐาน

### 204.1 ทวนจาก Part 16 Step 155 และขยายรายละเอียดเต็มรูปแบบ

**DHCP Snooping** คือ Layer 2 Security Feature ที่ทำให้ Switch ตรวจสอบ DHCP Message ทุกใบที่
วิ่งผ่านตัวเอง และตัดสินใจว่าจะอนุญาต Message ประเภทใดผ่านตามสถานะของพอร์ตที่รับเข้ามา:

| ประเภท Port | Message ที่อนุญาตให้ผ่าน | Message ที่ถูกบล็อก | ใช้กับ Port แบบไหน |
|---|---|---|---|
| **Trusted** | ทั้งหมด — `DHCPDISCOVER`, `DHCPREQUEST` (จาก Client) และ `DHCPOFFER`, `DHCPACK`, `DHCPNAK` (จาก Server) | ไม่มี | Uplink/Trunk ที่นำไปสู่ DHCP Server หรือ DHCP Relay Agent ที่ถูกต้องเท่านั้น |
| **Untrusted** (Default ของทุกพอร์ต) | เฉพาะ Message จาก Client: `DHCPDISCOVER`, `DHCPREQUEST` | `DHCPOFFER`/`DHCPACK`/`DHCPNAK` ที่มาจากพอร์ตนี้ — **Drop ทันทีพร้อม Log** | Access Port ทุกจุดที่ต่อ PC/User/Server (ไม่ควรมี DHCP Server ตัวใดอยู่หลังพอร์ตเหล่านี้) |

### 204.2 DHCP Snooping Binding Table

ทุกครั้งที่ Switch เห็น `DHCPACK` ที่ถูกต้อง (มาจากพอร์ต Trusted) ตอบกลับ Client ที่ขอผ่าน
พอร์ต Untrusted Switch จะบันทึกข้อมูลลง **Binding Table** — ฐานข้อมูลที่ผูก MAC Address, IP
Address, VLAN, Lease Time, และ Interface ของ Client เข้าด้วยกัน:

```
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00:50:79:66:68:01   10.10.10.101     86400       dhcp-snooping   10    GigabitEthernet1/0/1
```

Binding Table นี้คือฐานข้อมูลกลางที่ DAI (Step 206-207) และ IP Source Guard (Step 208) ใช้
อ้างอิงตรวจสอบ ARP Packet และ Source IP ของ Traffic ตามที่อธิบายไว้ใน 201.3

### 204.3 คำสั่งพื้นฐาน

| คำสั่ง | ผลลัพธ์ |
|---|---|
| `ip dhcp snooping` (Global) | เปิดใช้งาน DHCP Snooping ทั้งอุปกรณ์ (ยังไม่มีผลจนกว่าจะระบุ VLAN) |
| `ip dhcp snooping vlan <list>` (Global) | ระบุ VLAN ที่ต้องการให้ DHCP Snooping ทำงาน (ต้องระบุชัดเจน ไม่ Apply ทุก VLAN อัตโนมัติ) |
| `ip dhcp snooping trust` (Interface) | กำหนดพอร์ตนี้เป็น Trusted |
| `ip dhcp snooping limit rate <pps>` (Interface) | จำกัดจำนวน DHCP Packet/วินาทีที่พอร์ต Untrusted รับได้ — ป้องกัน DHCP Exhaustion Attack (ส่ง DHCPDISCOVER จำนวนมากเพื่อให้ Pool หมด) |
| `ip dhcp snooping information option` (Global) | เปิดการแทรก Option 82 (Relay Agent Information) เข้าไปใน DHCP Packet ที่ผ่านพอร์ต Untrusted (Default เปิดอยู่แล้วเมื่อเปิด Snooping) |
| `ip dhcp snooping database flash:/dhcp-snooping.db` (Global) | บันทึก Binding Table ลง Flash เพื่อให้รอดจาก Reload (ถ้าไม่ตั้งค่านี้ Binding Table จะหายหมดทุกครั้งที่ Switch Reload) |

> **ข้อสำคัญที่พลาดบ่อยที่สุด**: การพิมพ์ `ip dhcp snooping` เฉยๆ **ไม่ทำอะไรเลย** ถ้าไม่ตาม
> ด้วย `ip dhcp snooping vlan <list>` — DHCP Snooping ทำงานแบบ **Per-VLAN แบบ Opt-in** ต้อง
> ระบุ VLAN ที่ต้องการป้องกันอย่างชัดเจนเสมอ

### 204.4 การออกแบบ Trust Boundary สำหรับ Topology หลายชั้น (Multi-Tier)

Lab นี้มี Switch สองชั้นที่เกี่ยวข้องกับ DHCP Traffic: **ACCESS-SW1–4** (ต่อ Client โดยตรง)
และ **DIST-SW1–4** (ทำหน้าที่ DHCP Relay Agent ด้วย `ip helper-address` ไปยัง DHCP-DNS-SRV
ตามที่ Config ไว้ใน Part 16) หลักการวาง Trust Boundary ที่ถูกต้องคือ:

```
DHCP-DNS-SRV (10.10.30.20)
        │  (Trunk ขึ้น Core — Trusted)
   DIST-SW1/2/3/4  ◄── ทุกพอร์ต Trunk (ขึ้น Core และลง Access) = Trusted
        │  (Trunk ลง Access — Trusted จากมุมมอง DIST)
   ACCESS-SW1/2/3/4
        │  (Uplink Trunk ไป DIST — Trusted จากมุมมอง ACCESS)
        │
        └── Access Port (PC, Phone, AP, Server) = Untrusted (Default) ◄── จุดเดียวที่ Untrusted
```

**เหตุผล**: Trust Boundary ของ DHCP Snooping ควรอยู่ที่ **ขอบของเครือข่าย (Network Edge)**
เท่านั้น — จุดที่ End-user เสียบสายเข้ามาโดยตรง (Access Port ของ ACCESS-SW) ส่วน Link ที่เชื่อม
ระหว่าง Switch กับ Switch (Infrastructure Link ทั้งหมด ไม่ว่าจะเป็น ACCESS↔DIST หรือ
DIST↔CORE) ถือเป็น **Trusted** เสมอ เพราะ Rogue DHCP Server ตามคำนิยามคืออุปกรณ์ปลายทางที่
เสียบเข้า Access Port ไม่ใช่ Switch ในโครงสร้างพื้นฐานที่ Admin ควบคุมอยู่แล้ว — การตั้งเฉพาะ
Access Port เป็น Untrusted ก็เพียงพอสำหรับปิดช่องโหว่ Rogue DHCP Server ได้ครบถ้วน โดยไม่ต้อง
เปิด DHCP Snooping ที่ทุกพอร์ตของทุก Switch ซึ่งจะซับซ้อนเกินความจำเป็น

---

## Step 205 — Lab เต็มรูปแบบ: DHCP Snooping บน ACCESS-SW1–4 และ DIST-SW1–4

### 205.1 Full Config: ACCESS-SW1 (ตัวแทนของ ACCESS-SW1/2 — VLAN 10,20,99)

```
! ===== ACCESS-SW1 — DHCP Snooping =====
ACCESS-SW1(config)# ip dhcp snooping
ACCESS-SW1(config)# ip dhcp snooping vlan 10,20,99
ACCESS-SW1(config)# ip dhcp snooping information option
ACCESS-SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db

! ----- Uplink ไป DIST-SW1 (Gi0/1) และ DIST-SW2 (Gi0/2) = Trusted -----
ACCESS-SW1(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW1(config-if-range)# ip dhcp snooping trust
ACCESS-SW1(config-if-range)# exit

! ----- Access Port (PC/Phone/AP) = Untrusted (Default) + Rate Limit -----
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ip dhcp snooping limit rate 15
ACCESS-SW1(config-if-range)# exit
```

**ACCESS-SW2 ใช้ Config รูปแบบเดียวกันทุกบรรทัด** (VLAN 10,20,99, Uplink Gi0/1-2 Trusted)
เพียงเปลี่ยน Hostname — **ACCESS-SW3/ACCESS-SW4 เหมือนกันทุกประการ** แต่เปลี่ยน VLAN list
เป็น `30,40,99` เพราะทั้งสองตัวไม่มี VLAN 10/20 ตามแผน Access-Layer Mapping ใน
[00-ip-address-plan.md](00-ip-address-plan.md)

```
! ===== ACCESS-SW3 — DHCP Snooping (VLAN 30,40,99) =====
ACCESS-SW3(config)# ip dhcp snooping
ACCESS-SW3(config)# ip dhcp snooping vlan 30,40,99
ACCESS-SW3(config)# ip dhcp snooping information option
ACCESS-SW3(config)# ip dhcp snooping database flash:/dhcp-snooping.db

ACCESS-SW3(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW3(config-if-range)# ip dhcp snooping trust
ACCESS-SW3(config-if-range)# exit

ACCESS-SW3(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW3(config-if-range)# ip dhcp snooping limit rate 15
ACCESS-SW3(config-if-range)# exit
```

> **หมายเหตุพอร์ตของ DHCP-DNS-SRV**: Server1-3 (รวม DHCP-DNS-SRV) เชื่อมต่อกับ ACCESS-SW3
> ที่ `GigabitEthernet1/0/1` (VLAN 30, Static IP `10.10.30.20`) — พอร์ตนี้ยังคงเป็น
> **Untrusted ตามปกติ** (เพราะเป็น Access Port ปลายทาง ไม่ใช่ Uplink) แต่จะต้องมีการจัดการ
> พิเศษเรื่อง ARP ใน Step 207 เนื่องจาก Server ใช้ Static IP ไม่ผ่าน DHCP เลย จึง**ไม่มี**
> Entry อยู่ใน Binding Table โดยธรรมชาติ

### 205.2 Full Config: DIST-SW1 (ตัวแทนของ DIST-SW1–4)

```
! ===== DIST-SW1 — DHCP Snooping =====
DIST-SW1(config)# ip dhcp snooping
DIST-SW1(config)# ip dhcp snooping vlan 10,20,30,40,99
DIST-SW1(config)# ip dhcp snooping information option
DIST-SW1(config)# ip dhcp snooping database flash:/dhcp-snooping.db

! ----- ขาขึ้น CORE-SW1 (Gi0/1) และ CORE-SW2 (Gi0/2) = Trusted -----
DIST-SW1(config)# interface range GigabitEthernet0/1 - 2
DIST-SW1(config-if-range)# ip dhcp snooping trust
DIST-SW1(config-if-range)# exit

! ----- ขาลง ACCESS-SW1 (Gi0/3) และ ACCESS-SW2 (Gi0/4) = Trusted (Infrastructure Link) -----
DIST-SW1(config)# interface range GigabitEthernet0/3 - 4
DIST-SW1(config-if-range)# ip dhcp snooping trust
DIST-SW1(config-if-range)# exit
```

**DIST-SW2 ใช้ Config เดียวกันทุกบรรทัด** — **DIST-SW3/DIST-SW4** เหมือนกัน แต่ Trunk ลง
Access ของ DIST-SW3/4 นำไปสู่ ACCESS-SW3/ACCESS-SW4 (VLAN 30,40,99) แทน โดยยังคง
`ip dhcp snooping vlan 10,20,30,40,99` เหมือนกันทุกตัว เพราะ DIST-SW ทุกตัวเป็น Trunk ที่
พา VLAN ทั้งหมดผ่าน Core (ตามที่ Config ไว้ใน Part 4/Part 10)

> **ทำไม DIST-SW ต้องเปิด `ip dhcp snooping` ทั้งที่ทุกพอร์ตเป็น Trusted หมด**: (1) DIST-SW
> ทำหน้าที่ DHCP Relay Agent (`ip helper-address`) ซึ่ง Cisco แนะนำให้เปิด DHCP Snooping คู่กัน
> เสมอเพื่อให้ Option 82 (Relay Agent Information) ถูกจัดการถูกต้องตาม Path การ Relay,
> (2) เป็น Defense-in-Depth — ถ้ามีวันหนึ่งมีการเพิ่ม Access Port บน DIST-SW โดยพลาด
> การเปิด Snooping ไว้ล่วงหน้าทำให้พอร์ตใหม่นั้นเป็น Untrusted โดย Default ทันทีโดยไม่ต้อง
> จำเพิ่ม Config อะไรอีก

### 205.3 Verification

```
ACCESS-SW1# show ip dhcp snooping
Switch DHCP snooping is enabled
Switch DHCP gleaning is disabled
DHCP snooping is configured on following VLANs:
10,20,99
DHCP snooping is operational on following VLANs:
10,20,99
DHCP snooping is configured on the following L3 Interfaces:

Insertion of option 82 is enabled
   circuit-id format: vlan-mod-port
   remote-id format: MAC
Option 82 on untrusted port is not allowed
Verification of hwaddr field is enabled
Verification of giaddr field is enabled
DHCP snooping trust/rate is configured on the following Interfaces:

Interface                  Trusted     Allow option    Rate limit (pps)
------------------------   -------     ------------    ----------------
GigabitEthernet0/1          yes         yes              unlimited
GigabitEthernet0/2          yes         yes              unlimited
GigabitEthernet1/0/1        no          no                     15
GigabitEthernet1/0/2        no          no                     15
```

```
ACCESS-SW1# show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00:50:79:66:68:01   10.10.10.101     86345       dhcp-snooping   10    GigabitEthernet1/0/1
0050.7966.6802       10.10.10.102     86201       dhcp-snooping   10    GigabitEthernet1/0/2
Total number of bindings: 2
```

`show ip dhcp snooping binding` บน **DIST-SW1** จะมี Entry น้อยกว่าหรือว่างเปล่า เพราะทุกพอร์ต
ของ DIST-SW1 เป็น Trusted (ตามที่ออกแบบใน 204.4) — Binding ถูกสร้างขึ้นโดย ACCESS-SW ที่เป็น
ผู้เห็น Transaction DORA ผ่านพอร์ต Untrusted ของตัวเองเป็นหลัก ซึ่งเป็นพฤติกรรมที่ถูกต้องตาม
การออกแบบ Trust Boundary แบบ Multi-Tier

---

## Step 206 — Dynamic ARP Inspection (DAI) พื้นฐาน

### 206.1 หลักการทำงาน

**Dynamic ARP Inspection (DAI)** ตรวจสอบ ARP Packet (ทั้ง ARP Request และ ARP Reply) ทุกใบ
ที่ผ่านพอร์ต Untrusted โดยเทียบ **Source MAC + Source IP** ของ ARP Packet กับ **DHCP Snooping
Binding Table** (Step 204-205) — ถ้าไม่ตรงกัน Packet จะถูก Drop ทันที ทำให้ ARP Spoofing/
Poisoning (Step 201) ทำไม่ได้ เพราะผู้โจมตีไม่สามารถประกาศ IP-to-MAC Binding ที่ไม่ตรงกับ
สิ่งที่ Switch เห็นตอน DHCP DORA ได้

```
DAI ตรวจสอบ ARP Packet ทุกใบที่เข้ามาทางพอร์ต Untrusted:

ARP Packet: Sender MAC = 0050.7966.6801, Sender IP = 10.10.10.101
                     │
                     ▼
        เทียบกับ DHCP Snooping Binding Table
   MacAddress: 00:50:79:66:68:01  IpAddress: 10.10.10.101  VLAN: 10  Interface: Gi1/0/1
                     │
              ┌──────┴──────┐
           ตรงกัน           ไม่ตรงกัน
              │                │
          ผ่านปกติ          DROP + Log (%ARP_INSPECTION Denied)
```

### 206.2 คำสั่งพื้นฐาน

| คำสั่ง | ผลลัพธ์ |
|---|---|
| `ip arp inspection vlan <list>` (Global) | เปิด DAI สำหรับ VLAN ที่ระบุ (เหมือน DHCP Snooping ต้องระบุ VLAN ชัดเจน) |
| `ip arp inspection trust` (Interface) | กำหนดพอร์ตเป็น Trusted สำหรับ DAI — **ไม่ตรวจสอบ ARP ที่มาจากพอร์ตนี้เลย** |
| `ip arp inspection validate {src-mac \| dst-mac \| ip} [allow-zeros]` (Global) | เพิ่มความเข้มงวด — ตรวจสอบ MAC ใน Ethernet Header ให้ตรงกับ MAC ใน ARP Payload ด้วย (ป้องกัน MAC Header ปลอมที่ ARP Payload ไม่ปลอม) |
| `ip arp inspection log-buffer entries <n>` (Global) | ควบคุมขนาด Log Buffer สำหรับ ARP Packet ที่ถูก Drop |

> **กฎสำคัญ**: **Trust Boundary ของ DAI ต้องตรงกับ Trust Boundary ของ DHCP Snooping เสมอ**
> เพราะ DAI พึ่ง Binding Table ของ DHCP Snooping โดยตรง — พอร์ตที่เป็น `ip dhcp snooping
> trust` ควรเป็น `ip arp inspection trust` ด้วยเสมอ (มิฉะนั้น ARP ของ Traffic ปกติที่ผ่าน
> Uplink จะถูก Drop เพราะ Uplink ไม่มี Binding Table ของ Client ปลายทางในตัวเอง)

---

## Step 207 — คอนฟิก DAI, Rate-Limiting, และ ARP ACL สำหรับ Host ที่ตั้ง Static IP

### 207.1 Full Config: ACCESS-SW1 — DAI ต่อจาก DHCP Snooping ของ Step 205

```
! ===== ACCESS-SW1 — Dynamic ARP Inspection =====
ACCESS-SW1(config)# ip arp inspection vlan 10,20,99
ACCESS-SW1(config)# ip arp inspection validate src-mac dst-mac ip

! ----- Uplink Gi0/1, Gi0/2 = Trusted (เหมือน DHCP Snooping) -----
ACCESS-SW1(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW1(config-if-range)# ip arp inspection trust
ACCESS-SW1(config-if-range)# exit

! ----- Access Port = Untrusted (Default) + Rate Limit ARP -----
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ip arp inspection limit rate 15 burst interval 1
ACCESS-SW1(config-if-range)# exit
```

`ip arp inspection limit rate 15 burst interval 1` จำกัด ARP Packet ที่พอร์ต Untrusted รับได้
ไม่เกิน 15 Packet ต่อวินาที (เฉลี่ยในช่วง Burst Interval 1 วินาที) — ป้องกัน ARP Flood ที่อาจ
ใช้เป็นการโจมตีแบบ DoS คู่กับ ARP Spoofing เอง ถ้าเกิน Limit พอร์ตจะเข้าสู่ err-disabled
(Cause: `arp-inspection`) เช่นเดียวกับ Port Security — ควรเปิด `errdisable recovery cause
arp-inspection` คู่กันด้วย

```
ACCESS-SW1(config)# errdisable recovery cause arp-inspection
```

**DIST-SW1–4** ทำแบบเดียวกัน โดยใช้ `ip arp inspection vlan 10,20,30,40,99` และ Trust ทุก
Trunk Port (ทั้งขาขึ้น Core และขาลง Access) ตามหลักการเดียวกับ DHCP Snooping ใน 204.4:

```
DIST-SW1(config)# ip arp inspection vlan 10,20,30,40,99
DIST-SW1(config)# interface range GigabitEthernet0/1 - 4
DIST-SW1(config-if-range)# ip arp inspection trust
DIST-SW1(config-if-range)# exit
```

### 207.2 ปัญหา: Host ที่ตั้ง Static IP ไม่มี Entry ใน DHCP Snooping Binding Table

DHCP-DNS-SRV (`10.10.30.20`, VLAN 30) และ Server อื่นๆ ใน VLAN 30 (SERVERS) **ใช้ Static IP
เสมอตาม Best Practice** (ระบุไว้ชัดเจนใน [Part 16](part-016-dhcp-dns.md) ว่า Server ไม่ควร
พึ่ง DHCP) — เมื่อไม่ผ่าน DHCP DORA เลย Server เหล่านี้จะ**ไม่มี Entry ใน DHCP Snooping
Binding Table โดยธรรมชาติ** ผลคือถ้าเปิด DAI บน VLAN 30 โดยไม่ทำอะไรเพิ่ม **ARP ทุกใบที่
DHCP-DNS-SRV ส่งจะถูก DAI Drop ทันที** เพราะไม่พบ Binding ให้ตรวจสอบ — ทำให้ Server
สื่อสารกับ Host อื่นในเครือข่ายไม่ได้เลยหลังเปิด DAI

### 207.3 ทางแก้: ARP Access-List (ARP ACL)

สร้าง ARP ACL ที่ระบุ IP-to-MAC Binding ของ Host ที่ตั้ง Static IP ไว้ล่วงหน้าด้วยมือ แล้ว
ผูกกับ VLAN ที่เกี่ยวข้องแทนการพึ่ง Binding Table อัตโนมัติ:

```
ACCESS-SW3(config)# arp access-list STATIC-SERVERS
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.20 mac host 0010.a5f3.9911
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.21 mac host 0010.a5f3.9922
ACCESS-SW3(config-arp-nacl)# permit ip host 10.10.30.22 mac host 0010.a5f3.9933
ACCESS-SW3(config-arp-nacl)# exit

! ผูก ARP ACL เข้ากับ VLAN 30 — ตรวจสอบ ARP ACL ก่อน แล้วจึง fallback ไปตรวจ Binding Table
ACCESS-SW3(config)# ip arp inspection filter STATIC-SERVERS vlan 30
```

- **ไม่ใส่ keyword `static`** ท้ายคำสั่ง `ip arp inspection filter` → DAI จะตรวจสอบ ARP ACL
  ก่อน ถ้าไม่ Match จะ **fallback ไปตรวจสอบกับ DHCP Snooping Binding Table ต่อ** (เหมาะกับ
  VLAN ที่มีทั้ง Host แบบ Static และแบบ DHCP ผสมกัน เช่น VLAN 30 ที่มี Server Static IP
  ปนกับ PC/Printer บางเครื่องที่อาจใช้ DHCP)
- **ใส่ keyword `static`** ท้ายคำสั่ง (`ip arp inspection filter STATIC-SERVERS vlan 30
  static`) → ตรวจสอบกับ ARP ACL **เท่านั้น** ไม่ fallback ไป Binding Table เลย (เหมาะกับ
  VLAN ที่ Host ทั้งหมดเป็น Static IP ล้วนๆ)

เนื่องจาก VLAN 30 (SERVERS) ตาม Design ของหลักสูตรนี้มี Server ที่ใช้ Static IP **ทั้งหมด**
(ไม่มี Host ใดขอ DHCP เลยตามที่ระบุใน Part 16) จึงใช้ตัวเลือก `static` เพื่อความชัดเจนและ
ประสิทธิภาพสูงสุด:

```
ACCESS-SW3(config)# ip arp inspection filter STATIC-SERVERS vlan 30 static
```

### 207.4 Verification

```
ACCESS-SW1# show ip arp inspection interfaces
 Interface        Trust State     Rate (pps)    Burst Interval
 ---------------  -----------     ----------    --------------
 Gi0/1             Trusted         None           N/A
 Gi0/2             Trusted         None           N/A
 Gi1/0/1           Untrusted       15              1
 Gi1/0/2           Untrusted       15              1

ACCESS-SW1# show ip arp inspection vlan 10
 Vlan     Configuration    Operation    ACL Match         Static ACL
 ----     -------------    ---------    -----------       ----------
   10     Enabled           Active       No                No

ACCESS-SW3# show ip arp inspection vlan 30
 Vlan     Configuration    Operation    ACL Match         Static ACL
 ----     -------------    ---------    -----------       ----------
   30     Enabled           Active       STATIC-SERVERS    Yes
```

```
ACCESS-SW1# show ip arp inspection statistics vlan 10
 Vlan    Forwarded   Dropped   DHCP Drops   ACL Drops
 ----    ---------   -------   ----------   ---------
   10        18420        3            3            0
```

`DHCP Drops = 3` แสดงว่ามี ARP Packet 3 ใบถูก Drop เพราะไม่ตรงกับ DHCP Snooping Binding
Table (ปกติเกิดจาก ARP ที่ยังไม่ผ่าน DHCP เสร็จสมบูรณ์ หรือ ARP ที่ Source IP/MAC ไม่ตรงกับ
Binding จริง)

---

## Step 208 — IP Source Guard

### 208.1 หลักการทำงาน — ขาที่สามของ Trio

**IP Source Guard (IPSG)** คือฟีเจอร์ตัวที่สามที่พึ่ง DHCP Snooping Binding Table (ต่อจาก
Port Security ที่ตรวจ MAC และ DAI ที่ตรวจ ARP) — IPSG ตรวจสอบ **Source IP Address (และ
ทางเลือกคือ Source MAC ด้วย) ของทุก IP Packet** ที่ส่งออกจากพอร์ต Untrusted เทียบกับ Binding
Table ถ้า Source IP ไม่ตรงกับที่เคยได้รับผ่าน DHCP (หรือไม่ตรงกับ Static Binding ที่ Admin
กำหนดไว้) Packet จะถูก Drop ทันที — ป้องกัน **IP Address Spoofing** ที่ Host ปลอม Source IP
ของตัวเองเป็น IP ของ Host อื่น (ใช้ทำ DoS หรือ Bypass ACL ที่กรองตาม Source IP)

| ฟีเจอร์ | ตรวจสอบอะไร | อ้างอิง Binding Table ตัวเดียวกัน |
|---|---|---|
| Port Security (Step 202-203) | **MAC Address** ต่อพอร์ต | ไม่ใช้ — ใช้ CAM Table ของตัวเอง |
| Dynamic ARP Inspection (Step 206-207) | **ARP Packet** (Sender MAC + Sender IP) | ✅ DHCP Snooping Binding Table |
| **IP Source Guard** | **Source IP** (และ MAC ถ้าเปิดโหมดเต็ม) ของ **IP Packet ทุกใบ** ไม่ใช่แค่ ARP | ✅ DHCP Snooping Binding Table |

### 208.2 คำสั่งพื้นฐาน

```
ip verify source                      ! ตรวจสอบ Source IP อย่างเดียว
ip verify source port-security        ! ตรวจสอบทั้ง Source IP และ Source MAC (ต้องเปิด Port Security ด้วย)
```

### 208.3 Worked Config

```
! ACCESS-SW1 — เปิด IP Source Guard บน Access Port ทั้งหมด
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ip verify source port-security
ACCESS-SW1(config-if-range)# exit
```

สำหรับ Host ที่ใช้ Static IP (เช่น DHCP-DNS-SRV) ที่ไม่มี Entry ใน Binding Table เหมือนที่พบ
ปัญหาเดียวกันใน Step 207 ต้องเพิ่ม **Static IP Source Binding** ด้วยมือ:

```
ACCESS-SW3(config)# ip source binding 0010.a5f3.9911 vlan 30 10.10.30.20 interface GigabitEthernet1/0/1
```

### 208.4 Verification

```
ACCESS-SW1# show ip verify source
Interface   Filter-type   Filter-mode   IP-address       Mac-address        Vlan
---------   -----------   -----------   ---------------  -----------------  ----
Gi1/0/1     ip-mac         active        10.10.10.101     0050.7966.6801     10
Gi1/0/2     ip-mac         active        10.10.10.102     0050.7966.6802     10

ACCESS-SW3# show ip source binding
MacAddress          IpAddress        Lease(sec)  Type                       VLAN  Interface
------------------  ---------------  ----------  -------------------------  ----  --------------------
0010.a5f3.9911       10.10.30.20      infinite     static                     30    GigabitEthernet1/0/1
0010.a5f3.6801       10.10.30.101     84532        dhcp-snooping              30    GigabitEthernet1/0/5
```

> **ข้อจำกัดที่ต้องรู้**: IP Source Guard ใช้ได้กับ Access Port เท่านั้น (เหมือน Port Security)
> และถ้า Host มีมากกว่า 1 IP บน Interface เดียว (เช่น เปลี่ยน IP เอง หรือ Secondary IP)
> ต้องมี Binding ครบทุก IP ที่ใช้จริง มิฉะนั้น Traffic ของ IP ที่ไม่มี Binding จะถูก Drop ทันที

---

## Step 209 — Storm Control

### 209.1 หลักการทำงาน

**Storm Control** ตรวจสอบปริมาณ Broadcast, Multicast, และ Unknown-Unicast Traffic ที่เข้า
พอร์ตในหน่วย % ของ Bandwidth ทั้งหมดของพอร์ต (หรือ pps/bps ขึ้นกับรุ่น Platform) — ถ้าเกิน
Threshold ที่ตั้งไว้ (Rising Threshold) Switch จะ **Drop Traffic ประเภทนั้นเกินขนาด**
(ไม่ Drop Traffic ปกติ) หรือ **ปิดพอร์ตทั้งหมด** ขึ้นกับ Action ที่กำหนด — ทำงานเป็นเกราะ
ป้องกันชั้นที่สองคู่กับ STP (Part 5) เพราะ STP ป้องกัน Topology Loop ได้ แต่ไม่ได้จำกัดปริมาณ
Traffic ต่อวินาทีในกรณีที่ Loop เกิดขึ้นชั่วขณะก่อน STP Converge หรือในกรณีที่มีการ Flood
Broadcast/Multicast ปริมาณสูงผิดปกติโดยไม่มี Loop เลย (เช่น Application ผิดพลาด หรือ NIC
เสียแล้วส่ง Broadcast รัว)

### 209.2 คำสั่งพื้นฐาน

```
storm-control broadcast level <rising-%> [<falling-%>]
storm-control multicast level <rising-%> [<falling-%>]
storm-control unicast level <rising-%> [<falling-%>]
storm-control action {shutdown | trap}
```

| Parameter | ความหมาย |
|---|---|
| `<rising-%>` | เมื่อ Traffic ประเภทนั้นเกิน % นี้ของ Bandwidth พอร์ต จะเริ่ม Suppress (Drop ส่วนที่เกิน) |
| `<falling-%>` | (Optional) เมื่อ Traffic ลดลงต่ำกว่า % นี้ จะเลิก Suppress — ถ้าไม่ระบุจะใช้ค่าเดียวกับ Rising |
| `storm-control action shutdown` | เมื่อเกิน Threshold ต่อเนื่อง พอร์ตจะเข้าสู่ err-disabled state (ต้อง errdisable recovery cause storm-control) |
| `storm-control action trap` (Default) | ส่ง SNMP Trap/Syslog แต่พอร์ตยังคง Up (แค่ Drop ส่วนเกิน) |

### 209.3 Worked Config — ป้องกัน Broadcast Storm บน Access Port

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# storm-control broadcast level 50.00 40.00
ACCESS-SW1(config-if-range)# storm-control multicast level 70.00
ACCESS-SW1(config-if-range)# storm-control action trap
ACCESS-SW1(config-if-range)# exit
```

ตัวเลือกใน Lab นี้เลือก `broadcast level 50.00 40.00` (Rising 50%, Falling 40%) เพราะ
Broadcast Traffic ปกติ (ARP, DHCP) ไม่ควรเกิน 50% ของ Bandwidth พอร์ตแม้ในสภาวะ Peak — ถ้าถึง
ระดับนี้แสดงว่าผิดปกติแน่นอน (Loop หรือ Storm จริง) ไม่ตั้ง `unicast level` เพราะ Unicast
Traffic ปริมาณสูงมักเป็น Traffic ปกติ (File Transfer ขนาดใหญ่) ไม่ใช่สัญญาณของปัญหา

**สำหรับ Trunk Port ระหว่าง DIST-SW และ CORE-SW** ควรตั้ง Threshold ที่สูงกว่า Access Port
มาก เพราะ Trunk รวม Traffic จากหลาย VLAN/Access Port เข้าด้วยกัน:

```
DIST-SW1(config)# interface range GigabitEthernet0/1 - 2
DIST-SW1(config-if-range)# storm-control broadcast level 20.00 10.00
DIST-SW1(config-if-range)# storm-control action trap
DIST-SW1(config-if-range)# exit
```

### 209.4 Verification

```
ACCESS-SW1# show storm-control broadcast
Interface     Filter State   Upper       Lower      Current
---------     -------------  ----------  ---------  ----------
Gi1/0/1        Forwarding     50.00%      40.00%     0.00%
Gi1/0/2        Forwarding     50.00%      40.00%     0.02%

ACCESS-SW1# show storm-control history GigabitEthernet1/0/1
```

`Filter State = Forwarding` แสดงว่า Traffic ยังต่ำกว่า Threshold (ปกติ) — ถ้า Traffic พุ่ง
เกิน Rising Threshold สถานะจะเปลี่ยนเป็น `Blocking` (Broadcast/Multicast ส่วนเกินถูก Drop)
พร้อม Log:

```
%PM-4-ERR_DISABLE: storm-control error detected on Gi1/0/1, putting Gi1/0/1 in err-disable state
```
(เกิดเฉพาะเมื่อตั้ง `storm-control action shutdown` เท่านั้น — ถ้าเป็น `trap` จะไม่มี
err-disable แค่มี SNMP Trap/Syslog แจ้งเตือน)

---

## Step 210 — Lab เต็มรูปแบบ: Layer 2 Security Baseline + การตรวจสอบผลลัพธ์

### 210.1 ภาพรวม Baseline ที่รวมทุกฟีเจอร์เข้าด้วยกัน

| ฟีเจอร์ | Access Port (ACCESS-SW) | Uplink/Trunk (ACCESS-SW และ DIST-SW) |
|---|---|---|
| Port Security | `maximum 2`, `violation restrict`, `mac-address sticky` | ไม่ใช้ (Trunk ไม่รองรับ Port Security แบบปกติ) |
| DHCP Snooping | Untrusted (Default) + `limit rate 15` | `ip dhcp snooping trust` |
| DAI | Untrusted (Default) + `limit rate 15 burst interval 1` | `ip arp inspection trust` |
| IP Source Guard | `ip verify source port-security` | ไม่ใช้ (เฉพาะ Access Port) |
| Storm Control | `broadcast level 50.00 40.00`, `action trap` | `broadcast level 20.00 10.00` (Threshold สูงกว่าเพราะรวม Traffic หลาย VLAN) |

### 210.2 Full Running-Config: ACCESS-SW1 (ตัวแทน ACCESS-SW1/2 — VLAN 10,20,99)

```
hostname ACCESS-SW1
!
errdisable recovery cause psecure-violation
errdisable recovery cause arp-inspection
errdisable recovery cause storm-control
errdisable recovery interval 300
!
ip dhcp snooping
ip dhcp snooping vlan 10,20,99
ip dhcp snooping information option
ip dhcp snooping database flash:/dhcp-snooping.db
!
ip arp inspection vlan 10,20,99
ip arp inspection validate src-mac dst-mac ip
!
vlan 999
 name NATIVE-UNUSED
!
interface GigabitEthernet0/1
 description ** Trunk to DIST-SW1 Gi0/3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface GigabitEthernet0/2
 description ** Trunk to DIST-SW2 Gi0/4 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface range GigabitEthernet1/0/1 - 20
 switchport mode access
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 15 burst interval 1
 ip verify source port-security
 storm-control broadcast level 50.00 40.00
 storm-control multicast level 70.00
 storm-control action trap
!
interface GigabitEthernet1/0/1
 switchport access vlan 10
 description ** Access Port - PC1 (VLAN10 SALES) **
!
interface GigabitEthernet1/0/2
 switchport access vlan 10
 description ** Access Port - PC2 (VLAN10 SALES) **
!
interface GigabitEthernet1/0/10
 switchport access vlan 20
 description ** Access Port - IP Phone (VLAN20 VOICE) **
!
interface GigabitEthernet1/0/20
 switchport access vlan 99
 description ** Access Port - AP-1 Management (VLAN99 MGMT) **
!
end
```

### 210.3 Full Running-Config: ACCESS-SW3 (ตัวแทน ACCESS-SW3/4 — VLAN 30,40,99 + ARP ACL)

```
hostname ACCESS-SW3
!
errdisable recovery cause psecure-violation
errdisable recovery cause arp-inspection
errdisable recovery cause storm-control
errdisable recovery interval 300
!
ip dhcp snooping
ip dhcp snooping vlan 30,40,99
ip dhcp snooping information option
ip dhcp snooping database flash:/dhcp-snooping.db
!
arp access-list STATIC-SERVERS
 permit ip host 10.10.30.20 mac host 0010.a5f3.9911
 permit ip host 10.10.30.21 mac host 0010.a5f3.9922
 permit ip host 10.10.30.22 mac host 0010.a5f3.9933
!
ip arp inspection vlan 30,40,99
ip arp inspection filter STATIC-SERVERS vlan 30 static
ip arp inspection validate src-mac dst-mac ip
!
ip source binding 0010.a5f3.9911 vlan 30 10.10.30.20 interface GigabitEthernet1/0/1
!
interface GigabitEthernet0/1
 description ** Trunk to DIST-SW3 Gi0/3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 30,40,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface GigabitEthernet0/2
 description ** Trunk to DIST-SW4 Gi0/3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 30,40,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Access Port - DHCP-DNS-SRV (VLAN30 SERVERS, Static IP) **
 switchport mode access
 switchport access vlan 30
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address 0010.a5f3.9911
 ip verify source port-security
 storm-control broadcast level 50.00 40.00
 storm-control action trap
!
interface range GigabitEthernet1/0/2 - 20
 switchport mode access
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 15 burst interval 1
 ip verify source port-security
 storm-control broadcast level 50.00 40.00
 storm-control multicast level 70.00
 storm-control action trap
!
end
```

> **หมายเหตุพอร์ตของ DHCP-DNS-SRV**: เนื่องจากเป็น Server ที่มี MAC/IP คงที่แน่นอน จึงใช้
> **Static MAC** (ไม่ใช่ Sticky) กับ `maximum 1` เพื่อความเข้มงวดสูงสุด — Server ตัวนี้ไม่ควร
> มี MAC Address อื่นปรากฏบนพอร์ตเดียวกันได้เลยไม่ว่ากรณีใด

### 210.4 Full Running-Config: DIST-SW1 (ตัวแทน DIST-SW1–4)

```
hostname DIST-SW1
!
errdisable recovery cause arp-inspection
errdisable recovery cause storm-control
errdisable recovery interval 300
!
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,99
ip dhcp snooping information option
ip dhcp snooping database flash:/dhcp-snooping.db
!
ip arp inspection vlan 10,20,30,40,99
ip arp inspection validate src-mac dst-mac ip
!
interface GigabitEthernet0/1
 description ** Trunk to CORE-SW1 Gi0/3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 storm-control broadcast level 20.00 10.00
 storm-control action trap
 no shutdown
!
interface GigabitEthernet0/2
 description ** Trunk to CORE-SW2 Gi0/3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 storm-control broadcast level 20.00 10.00
 storm-control action trap
 no shutdown
!
interface GigabitEthernet0/3
 description ** Trunk to ACCESS-SW1 Gi0/1 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 storm-control broadcast level 20.00 10.00
 storm-control action trap
 no shutdown
!
interface GigabitEthernet0/4
 description ** Trunk to ACCESS-SW2 Gi0/1 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,99,999
 switchport nonegotiate
 ip dhcp snooping trust
 ip arp inspection trust
 storm-control broadcast level 20.00 10.00
 storm-control action trap
 no shutdown
!
end
```

**DIST-SW2, DIST-SW3, DIST-SW4** ใช้ Config รูปแบบเดียวกันทุกบรรทัด เปลี่ยนแค่ Description
ปลายทาง (DIST-SW3/4 มี Gi0/3-4 ที่ `switchport trunk allowed vlan 30,40,99,999` แทน เพราะลง
ไปยัง ACCESS-SW3/ACCESS-SW4 ที่ไม่มี VLAN 10/20)

### 210.5 การตรวจสอบผลลัพธ์ — สภาวะปกติและสภาวะผิดปกติ

ต่อไปนี้คือพฤติกรรมของแต่ละฟีเจอร์ที่สังเกตได้จาก Log และ `show` command เมื่อเกิดเงื่อนไข
ที่ระบุ — ใช้สำหรับตรวจสอบว่า Baseline ที่ Deploy ไปทำงานถูกต้องจริงในสภาวะทั้งปกติและผิดปกติ

**(1) สภาวะปกติ — PC1 เสียบเข้า `GigabitEthernet1/0/1` ของ ACCESS-SW1 ครั้งแรก**

```
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/1, changed state to up
```

PC1 ส่ง `DHCPDISCOVER` → DIST-SW1 Relay ไปยัง DHCP-DNS-SRV → ได้รับ `DHCPACK` กลับผ่าน
Trusted Path ทั้งหมด → ACCESS-SW1 บันทึก Binding Table และแปลง Sticky MAC เป็น Static
อัตโนมัติ:

```
ACCESS-SW1# show port-security interface GigabitEthernet1/0/1 | include Sticky|Total
Total MAC Addresses           : 1
Sticky MAC Addresses           : 1
ACCESS-SW1# show ip dhcp snooping binding interface GigabitEthernet1/0/1
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00:50:79:66:68:01   10.10.10.101     86397       dhcp-snooping   10    GigabitEthernet1/0/1
```

**(2) สภาวะผิดปกติ — มีการเชื่อมต่ออุปกรณ์ DHCP Server ที่ไม่ได้รับอนุญาตเข้าพอร์ต Untrusted**

เมื่อ Access Port ใดก็ตามที่เป็น Untrusted ได้รับ `DHCPOFFER`/`DHCPACK` เข้ามา (แสดงว่ามี DHCP
Server ตัวที่สองอยู่หลังพอร์ตนั้น ซึ่งไม่ควรมี) Switch จะ Drop Packet ทันทีพร้อม Generate Log:

```
%DHCP_SNOOPING-5-DHCP_SNOOPING_UNTRUSTED_PORT: DHCP_SNOOPING drop message on untrusted port, message type: DHCPOFFER, MAC sa: 00d0.9988.7766, port GigabitEthernet1/0/12, vlan 10.
```

`show ip dhcp snooping binding` จะ**ไม่มี** Entry ใหม่จาก Offer ปลอมนี้เกิดขึ้นเลย เพราะถูก
Drop ก่อนที่จะไปถึงขั้นตอนบันทึก Binding Table — Client ที่อยู่หลังพอร์ตนั้นจะยังคงได้รับ
`DHCPOFFER` จาก DHCP-DNS-SRV ตัวจริงตามปกติ (ซึ่งมาจากทาง Trusted Uplink) โดยไม่ถูกรบกวน

**(3) สภาวะผิดปกติ — ARP Packet ที่มี Source IP/MAC ไม่ตรงกับ Binding Table**

```
%SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Req) on Gi1/0/12, vlan 10.([00d0.9988.7766/10.10.10.102/0000.0000.0000/0.0.0.0/03:14:22 UTC Thu Sep 24 2026])
```

```
ACCESS-SW1# show ip arp inspection statistics vlan 10
 Vlan    Forwarded   Dropped   DHCP Drops   ACL Drops
 ----    ---------   -------   ----------   ---------
   10        24810        1            1            0
```

ARP Packet ที่ประกาศ IP `10.10.10.102` (ของ PC2) จาก MAC `00d0.9988.7766` (ไม่ใช่ MAC ของ
PC2 ที่บันทึกไว้จริงคือ `0050.7966.6802`) ถูก Drop ทันที ป้องกันไม่ให้ ARP Cache ของ Host อื่น
ในเครือข่ายถูก Poison สำเร็จ

**(4) สภาวะผิดปกติ — MAC Address ที่สองพยายามส่ง Frame ผ่านพอร์ตที่ Maximum เต็มแล้ว**

```
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 00d0.9878.1122 on port GigabitEthernet1/0/1.
```

```
ACCESS-SW1# show port-security interface GigabitEthernet1/0/1 | include Violation
Security Violation Count        : 1
```

เพราะ Violation Mode ตั้งเป็น `restrict` (210.1) พอร์ตยังคง Up ปกติ — เฉพาะ Frame ของ MAC
ใหม่ที่เกิน Maximum เท่านั้นที่ถูก Drop ส่วน MAC ที่ได้รับอนุญาตอยู่แล้ว (PC1 เดิม) ยังใช้งาน
เครือข่ายได้ต่อเนื่องโดยไม่สะดุด

**(5) สภาวะผิดปกติ — Traffic Broadcast พุ่งสูงผิดปกติ (เช่น Loop ชั่วขณะก่อน STP Converge)**

```
%STORM_CONTROL-3-FILTERED: A Broadcast storm detected on Gi1/0/1. Action shown in log, Broadcast traffic has been filtered.
```

```
ACCESS-SW1# show storm-control broadcast
Interface     Filter State   Upper       Lower      Current
---------     -------------  ----------  ---------  ----------
Gi1/0/1        Blocking       50.00%      40.00%     67.40%
```

`Filter State = Blocking` แสดงว่า Broadcast ส่วนที่เกิน 50% ของ Bandwidth ถูก Drop ทันที
โดย Unicast/Multicast Traffic ปกติของพอร์ตยังคงส่งผ่านได้ตามปกติ ไม่ได้รับผลกระทบ

---

## แบบฝึกหัดทวนความเข้าใจ Part 21

1. เมื่อ CAM Table ของ Switch เต็มจาก MAC Flooding Switch จะเปลี่ยนพฤติกรรมอย่างไร และ
   ฟีเจอร์ใดที่ป้องกันปัญหานี้?
2. Violation Mode สามแบบของ Port Security (`protect`, `restrict`, `shutdown`) แตกต่างกัน
   อย่างไรในเรื่องการ Drop Traffic, การนับ Violation Counter, และผลต่อสถานะพอร์ต?
3. ทำไม DHCP Snooping ต้องเปิดใช้งานก่อน Dynamic ARP Inspection และ IP Source Guard เสมอ?
4. ถ้า Server ตัวหนึ่งใน VLAN 30 ใช้ Static IP และไม่เคยขอ DHCP เลย จะเกิดอะไรขึ้นถ้าเปิด DAI
   บน VLAN นั้นโดยไม่ทำอะไรเพิ่ม และแก้ปัญหานี้ด้วยคำสั่งกลุ่มใด?
5. Storm Control ต่างจาก STP (Part 5) อย่างไรในการป้องกันปัญหาที่เกิดจาก Network Loop?

**เฉลย:**
1. Switch เข้าสู่ Fail-Open Mode — Flood Frame ทุกใบออกทุกพอร์ตในบรอดคาสต์โดเมนเหมือน Hub
   (ทำให้ผู้โจมตี Sniff Traffic ได้) — ป้องกันด้วย **Port Security** ที่จำกัดจำนวน MAC ต่อพอร์ต
2. `protect` = Drop เงียบ ไม่นับ Counter ไม่ Log, Port ยัง Up; `restrict` = Drop + นับ Counter
   + ส่ง Log, Port ยัง Up; `shutdown` (Default) = Drop + นับ Counter + ส่ง Log + Port เข้าสู่
   err-disabled state ทันที (ต้องกู้คืนด้วยมือหรือ errdisable recovery)
3. เพราะ DAI และ IP Source Guard ทั้งสองฟีเจอร์อ้างอิง **DHCP Snooping Binding Table** เป็น
   ฐานข้อมูลกลางในการตรวจสอบ ARP Packet และ Source IP ตามลำดับ — ถ้าไม่มี Binding Table ที่
   ถูกต้อง Traffic ปกติของ Host ที่ได้ IP ผ่าน DHCP จะถูกบล็อกไปด้วยเพราะไม่มี Binding ให้ตรวจสอบ
4. ARP ทุกใบของ Server นั้นจะถูก DAI Drop ทันทีเพราะไม่มี Entry ใน Binding Table (ไม่เคยผ่าน
   DHCP) — แก้ด้วย **ARP Access-List** (`arp access-list` + `ip arp inspection filter <acl>
   vlan <id> static`) ที่ระบุ IP-to-MAC Binding ของ Server ไว้ล่วงหน้าด้วยมือ
5. STP ป้องกัน Loop ทาง Topology (ปิด Port ที่ทำให้เกิดวงวน) แต่ไม่จำกัดปริมาณ Traffic ต่อ
   วินาที ส่วน Storm Control จำกัด % Bandwidth ของ Broadcast/Multicast/Unicast ต่อพอร์ตโดยตรง
   — ทำงานเป็นเกราะป้องกันชั้นที่สองในกรณีที่มี Traffic พุ่งสูงผิดปกติแม้ไม่มี Loop ทาง
   Topology เลยก็ตาม (เช่น NIC เสียแล้วส่ง Broadcast รัว)

---

## สรุป Part 21

Part นี้ปิดช่องโหว่ Layer 2 Security ที่ค้างไว้จาก Part 4 และ Part 16 อย่างครบถ้วน ด้วยการสร้าง
**Layer 2 Security Baseline** ที่ประกอบด้วย 5 ฟีเจอร์หลักทำงานร่วมกันเป็นระบบ: **Port
Security** (จำกัด MAC ต่อพอร์ต ป้องกัน MAC Flooding), **DHCP Snooping** (สร้าง Binding Table
กลาง ป้องกัน Rogue DHCP Server), **Dynamic ARP Inspection** (ใช้ Binding Table ตรวจสอบ ARP
ป้องกัน ARP Spoofing), **IP Source Guard** (ใช้ Binding Table เดียวกันตรวจสอบ Source IP
ป้องกัน IP Spoofing), และ **Storm Control** (จำกัดปริมาณ Broadcast/Multicast/Unicast ทำงาน
คู่กับ STP) — Config เต็มรูปแบบถูก Deploy ลงทั้ง **ACCESS-SW1–4** และ **DIST-SW1–4** พร้อม
ตัวอย่าง Log และ Show Command ที่ใช้ตรวจสอบผลลัพธ์ทั้งในสภาวะปกติและผิดปกติ ✅

**พร้อมสำหรับ Part 22**: หลังจากปิดช่องโหว่ Layer 2 ที่ Access Layer ครบถ้วนแล้ว Part ถัดไป
จะยกระดับความปลอดภัยไปสู่ **การควบคุมสิทธิ์การเข้าถึงตัวอุปกรณ์ Cisco เอง** ด้วย **AAA
(Authentication, Authorization, Accounting)** ผ่าน RADIUS/TACACS+ Server กลาง และ Device
Hardening เพิ่มเติมที่จำเป็นสำหรับ Enterprise Network จริง

**ไปต่อ:** [Part 22 — AAA & Device Hardening →](part-022-aaa-device-hardening.md)
