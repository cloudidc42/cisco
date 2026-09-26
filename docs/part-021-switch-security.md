# Part 21 — Switch Security (Port Security, DHCP Snooping, DAI)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 201–210 จาก 1000**

> ต่อจาก [Part 20 — Wireless Fundamentals (WLC, AP, SSID)](part-020-wireless-fundamentals.md)
> ที่เราขยาย Access Layer ของ Enterprise Lab ไปสู่โลกไร้สายแล้ว Part นี้จะพาเรากลับมาปิด
> **ช่องโหว่ด้านความปลอดภัยที่ Layer 2 ของ Access Layer** ที่ค้างไว้จากสอง Part ก่อนหน้าอย่าง
> ตั้งใจ: (1) [Part 16 — DHCP/DNS](part-016-dhcp-dns.md) **Step 155** ที่แนะนำ DHCP Snooping
> ไว้แค่ระดับ Concept (Trusted/Untrusted Port) และบอกไว้ชัดเจนว่า *"Config เต็มรูปแบบจะอยู่ใน
> Part 21"* และ (2) [Part 4 — Trunking, VTP, DTP](part-004-trunking-vtp-dtp.md) **Step 32**
> ที่พูดถึง **Native VLAN Double-Tagging Attack** ไว้แค่ผ่านๆ ก่อนจะสรุปเป็น Best Practice
> "เปลี่ยน Native VLAN เป็น VLAN 999" — Part นี้จะอธิบายกลไกการโจมตีทั้งสองแบบนี้โดยละเอียด
> พร้อม Config ป้องกันจริงบน **ACCESS-SW1–4** และ **DIST-SW1–4** ร่วมกับฟีเจอร์ความปลอดภัย
> ระดับ Access Layer ที่เหลือทั้งหมด — **Port Security, DHCP Snooping, Dynamic ARP Inspection
> (DAI), IP Source Guard, Storm Control** — เพื่อสร้าง **Layer 2 Security Baseline ฉบับสมบูรณ์**
> ที่จะใช้เป็นมาตรฐานของทุก Switch ในหลักสูตรนี้ตั้งแต่ Part นี้เป็นต้นไป

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 201 | Layer 2 Attacks ภาพรวม — MAC Flooding, Rogue DHCP, ARP Spoofing, VLAN Hopping |
| 202 | Port Security พื้นฐาน — `switchport port-security`, Maximum, Violation Mode, Sticky MAC |
| 203 | Port Security Violation Handling เชิงลึก และ Err-Disable Recovery |
| 204 | DHCP Snooping พื้นฐาน — Trusted/Untrusted, Binding Table, Rate-Limiting |
| 205 | Lab เต็มรูปแบบ: DHCP Snooping บน ACCESS-SW1–4 และ DIST-SW1–4 |
| 206 | Dynamic ARP Inspection (DAI) พื้นฐาน — ป้องกัน ARP Spoofing ด้วย Binding Table |
| 207 | คอนฟิก DAI, Rate-Limiting, และ ARP ACL สำหรับ Host ที่ตั้ง Static IP |
| 208 | IP Source Guard — ขาที่ 3 ของ Trio ความปลอดภัยที่พึ่ง DHCP Snooping Binding Table |
| 209 | Storm Control — ป้องกัน Broadcast/Multicast/Unicast Storm |
| 210 | Lab เต็มรูปแบบ: Layer 2 Security Baseline ทุกอุปกรณ์ + จำลองการโจมตีจริง |

---

## Step 201 — Layer 2 Attacks ภาพรวม

### 201.1 ทำไม Layer 2 Security สำคัญไม่แพ้ Layer 3

ตลอด Part 1-20 เราใช้เวลาส่วนใหญ่กับ Layer 3 Security (ACL ใน Part 14, NAT ใน Part 15) และ
Layer 2 Availability (STP ใน Part 5, EtherChannel ใน Part 6) แต่ยังไม่ได้พูดถึง **Layer 2
Security** อย่างจริงจังเลย — ทั้งที่ Layer 2 คือจุดที่ **End-user เสียบสายเข้าเครือข่ายได้โดยตรง**
(Access Port ของ ACCESS-SW1-4) ต่างจาก Layer 3 ที่มักมี Firewall/ACL คั่นอยู่แล้ว

**ข้อเท็จจริงสำคัญ**: Layer 2 Protocol ส่วนใหญ่ (Ethernet, ARP, DHCP, STP, CDP) ถูกออกแบบมา
ในยุคที่เครือข่ายเป็น "Trusted Environment" ทั้งหมด — **ไม่มี Authentication ในตัว Protocol
เองเลย** ผู้โจมตีที่เสียบสายเข้า Access Port เพียงจุดเดียวสามารถส่ง Frame ปลอมได้อย่างเสรี
โดย Switch เองไม่ตรวจสอบอะไรเลยถ้าไม่เปิดฟีเจอร์ความปลอดภัยเพิ่มเติม — นี่คือเหตุผลที่
Cisco สร้างฟีเจอร์กลุ่มนี้ขึ้นมาเสริมทับ Protocol เดิมทั้งหมด

### 201.2 ตารางสรุป Layer 2 Attacks หลักที่ต้องรู้สำหรับ CCNA/CCNP

| Attack | วิธีการโจมตี | ผลกระทบ | Mitigation (สอนใน Part นี้) |
|---|---|---|---|
| **MAC Flooding (CAM Table Overflow)** | ส่ง Frame ที่มี Source MAC ปลอมสุ่มจำนวนมหาศาลอย่างรวดเร็ว (เครื่องมือเช่น `macof`) เพื่อให้ CAM Table (MAC Address Table) ของ Switch เต็ม | เมื่อ CAM Table เต็ม Switch **Fail-Open** — เปลี่ยนพฤติกรรมจาก Switch เป็นเหมือน Hub คือ Flood ทุก Frame ออกทุกพอร์ต ทำให้ผู้โจมตี Sniff Traffic ของคนอื่นในเครือข่ายเดียวกันได้หมด (แม้ไม่ได้อยู่ VLAN เดียวกับเหยื่อในทาง Physical ก็ตาม ถ้าอยู่ VLAN เดียวกัน) | **Port Security** (Step 202-203) จำกัดจำนวน MAC ต่อพอร์ต |
| **Rogue DHCP Server** | เสียบอุปกรณ์ที่รัน DHCP Server (ตั้งใจหรือไม่ตั้งใจ เช่น เอา Home Router มาเสียบผิดที่) เข้า Access Port แล้วแจก `DHCPOFFER` แข่งกับ Server จริง | Client รับ Offer ใบแรกที่มาถึงโดยไม่ตรวจสอบ — ได้ Gateway/DNS ปลอม เกิด **Man-in-the-Middle** เต็มรูปแบบ หรือ DoS (แจก Scope ผิด ทำให้ต่อเน็ตไม่ได้) | **DHCP Snooping** (Step 204-205) |
| **ARP Spoofing / ARP Poisoning** | ส่ง Gratuitous ARP Reply ปลอมอ้างว่าตัวเองคือ IP ของ Gateway (หรือของเครื่องอื่น) ซ้ำๆ เพื่อ Poison ARP Cache ของเหยื่อ | เหยื่อส่ง Traffic ทั้งหมดไปที่เครื่องผู้โจมตีก่อน (คิดว่าเป็น Gateway) → **Man-in-the-Middle**, ดู/แก้ไข Traffic ได้ทั้งหมดก่อนส่งต่อจริง (หรือไม่ส่งต่อเลย = DoS) | **Dynamic ARP Inspection (DAI)** (Step 206-207) |
| **VLAN Hopping — Double Tagging** | ส่ง Frame ที่มี 802.1Q Tag ซ้อนกัน 2 ชั้น (Outer Tag = Native VLAN ของ Trunk ตัวแรก, Inner Tag = VLAN เป้าหมาย) จาก Access Port ที่อยู่ใน Native VLAN | Trunk ตัวแรกดึง Outer Tag (Native) ออกเพราะคิดว่าไม่ต้อง Tag แล้วส่งต่อ Frame ที่เหลือ (ยังมี Inner Tag ของ VLAN เป้าหมาย) เข้าสู่ VLAN เป้าหมายทันทีแบบ **One-way โดยไม่ต้องเป็นสมาชิก VLAN นั้นเลย** — รายละเอียดเต็มด้านล่าง (201.3) | เปลี่ยน Native VLAN เป็น VLAN ที่ไม่มีใครใช้ (VLAN 999 — ทำแล้วใน Part 4), Tag Native VLAN ด้วย (`vlan dot1q tag native`), ห้าม Access Port ใดอยู่ใน Native VLAN |
| **VLAN Hopping — Switch Spoofing** | เครื่อง Host ส่ง DTP Frame ปลอมอ้างว่าตัวเองเป็น Switch ที่รองรับ Trunk (`dynamic desirable`) เพื่อเจรจาให้ Access Port เปลี่ยนเป็น Trunk จริง | ถ้าเจรจาสำเร็จ ผู้โจมตีได้ Trunk Port เต็มรูปแบบ มองเห็นทุก VLAN ที่ Trunk อนุญาต ไม่ต้อง Hop ทีละ VLAN เหมือน Double Tagging | `switchport mode access` + `switchport nonegotiate` แบบตายตัวบน**ทุก Access Port** — ห้ามปล่อยเป็น `dynamic auto/desirable` เด็ดขาด (ดูรายละเอียด 201.4) |
| **Broadcast/Multicast Storm** | Loop ทาง Physical โดยไม่ตั้งใจ (STP ล้มเหลว) หรือผู้โจมตีส่ง Broadcast/Multicast Flood ตั้งใจ | ใช้ Bandwidth ทั้ง Link จนเครือข่ายทั้ง VLAN ใช้งานไม่ได้ (Broadcast Storm) | **Storm Control** (Step 209) ทำงานร่วมกับ STP (Part 5) |

> **ตารางนี้คือแผนที่ของ Part นี้ทั้งหมด** — สังเกตว่าฟีเจอร์ป้องกันแต่ละตัว (คอลัมน์ขวาสุด)
> จะถูกอธิบายเรียงตาม Step อย่างเป็นระบบ และ Step 210 ปิดท้ายด้วยการเอาทุกฟีเจอร์มารวมกันเป็น
> **Baseline เดียว** ที่ใช้กับทุก Switch ใน Lab

### 201.3 กลไกโดยละเอียดของ Native VLAN Double-Tagging Attack (ตามที่ค้างไว้จาก Part 4 Step 32)

```
ผู้โจมตี (เสียบที่ Access Port ซึ่งบังเอิญ/ตั้งใจอยู่ใน Native VLAN ของ Trunk ถัดไป)
   │
   │  Frame ที่ผู้โจมตีส่ง: [Outer 802.1Q Tag = VLAN 1 (Native)] [Inner 802.1Q Tag = VLAN 30] [Payload]
   ▼
┌─────────────────────┐
│   ACCESS-SW (ตัวแรก)   │  รับ Frame เข้ามาที่ Access Port (VLAN 1) → ส่งออก Trunk
└──────────┬───────────┘
           │  Trunk Port ของ ACCESS-SW เห็นว่า Frame นี้ปลายทางต้อง "ไม่ Tag" เพราะ VLAN 1
           │  คือ Native VLAN ของ Trunk ตัวเอง → **ดึง (Strip) Outer Tag ออก** แล้วส่งต่อ
           ▼        Frame ที่เหลือ = [Inner 802.1Q Tag = VLAN 30] [Payload]  (ยังมี Tag ซ้อนอยู่!)
┌─────────────────────┐
│   Switch ตัวที่สอง     │  รับ Frame ที่ "เหลือ Tag เดียว" เข้ามาทาง Trunk → **ตีความ Inner Tag
│  (DIST-SW / อีก VLAN)  │  (VLAN 30) เป็น VLAN จริงของ Frame ทันที** เพราะดูเหมือน Frame ปกติ
└──────────┬───────────┘  ของ VLAN 30 ที่ Tag มาแบบมาตรฐาน
           ▼
   Frame ถูกส่งเข้า VLAN 30 (SERVERS) — ผู้โจมตี "กระโดด" จาก VLAN 1 (ห