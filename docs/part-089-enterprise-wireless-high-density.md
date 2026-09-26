# Part 89 — Enterprise Wireless at Scale (High-Density Design)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 881–890 จาก 1000**

> ต่อจาก [Part 88 — IPv6 Enterprise Deployment →](part-088-ipv6-enterprise-deployment.md) ที่ปิดท้าย
> เรื่อง Dual-Stack/IPv6 ระดับ Enterprise Part 89 นี้ดึง Wireless กลับมาอีกครั้งหลังจากที่
> [Part 36 — Wireless Architecture Deep Dive](part-036-wireless-architecture-deep-dive.md) สอน
> สถาปัตยกรรม CAPWAP, Roaming, Mobility Group และ WLC High Availability ไปแล้วในระดับ CCNP —
> Part นี้ไม่สอนสถาปัตยกรรมซ้ำ แต่พา RF Design ไปสู่สถานการณ์ที่ Part 36 **จงใจไม่แตะ**:
> **High-Density Design** — สนามกีฬา, Auditorium, ห้องประชุมใหญ่ระดับหลายร้อยคน และ Campus
> 10 อาคารจาก [Part 72](part-072-large-campus-fabric-design.md) ที่ต้องรองรับ Client หนาแน่น
> ในบางพื้นที่พร้อมกันหลายร้อยเครื่อง — ตามที่ Part 72 Step 718 บอกไว้ว่า "รายละเอียดเจาะลึก
> High-Density Area จะเรียนแบบเต็มรูปแบบใน Part 89" นี่คือ Part นั้น
>
> Part นี้อ้างอิงพื้นฐานที่เรียนไปแล้วอย่างเข้มข้น: RRM (DCA/TPC) และ RF Profile จาก Part 36
> Step 353/358, QoS/WMM Mapping จาก [Part 24](part-024-qos-fundamentals.md),
> [Part 40](part-040-advanced-qos.md), และ [Part 82 Step 815](part-082-end-to-end-enterprise-qos-design.md),
> และ Assurance Dashboard จาก [Part 50](part-050-dna-center.md) — ถ้าจำ Step เหล่านี้ไม่ได้
> ควรย้อนทวนก่อนเริ่ม Part นี้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 881 | High-Density Design Fundamentals — ปัญหาพื้นฐานและ Checklist เต็มรูปแบบ |
| 882 | RF Planning for High Density — Cell Size, AP Density, 5GHz-Preference |
| 883 | Client Capacity Management — max-clients, Load-Balancing, Band-Select |
| 884 | High-Density Hardware Considerations — Antenna, AP Model Selection |
| 885 | Airtime Fairness และ QoS at High Density |
| 886 | 6GHz (Wi-Fi 6E) และ High-Density Design |
| 887 | Voice/Video over High-Density Wi-Fi — CAC (Call Admission Control) |
| 888 | Temporary High-Density Events — Stadium/Conference Scenario |
| 889 | Monitoring และ Validating High-Density Performance |
| 890 | Lab เต็มรูปแบบ: High-Density Design Document สำหรับ Auditorium ของ Building 5 |

---

## Step 881 — High-Density Design Fundamentals

### ปัญหาพื้นฐาน: ทำไม High-Density ถึงเป็นปัญหาคนละแบบจาก Office ทั่วไป

Part 36 Step 358 สอนเรื่อง RF Profile ที่แยกตามความหนาแน่น (`RF-HIGH-DENSITY` vs Default) ไปแล้ว
แต่ยังไม่ได้อธิบาย **ตัวเลขจริง** ที่ทำให้ต้องแยก — ตารางด้านล่างเทียบให้เห็นภาพ

| สถานการณ์ | พื้นที่ตัวอย่าง | Client/AP โดยประมาณ | ตัวอย่างในหลักสูตรนี้ |
|---|---|---|---|
| Office ทั่วไป | ห้องทำงาน/Cubicle | 15-30 Client/AP | Building 1-9 ชั้นทำงานทั่วไป (Part 72 Step 718) |
| Conference Room ขนาดกลาง | ห้องประชุม 20-50 ที่นั่ง | 30-60 Client/AP | ห้องประชุมย่อยทั่วอาคาร |
| **High-Density** | Auditorium, Town Hall, Training Center | **150-500 Client/AP ถ้าไม่เพิ่ม AP** | Auditorium ของ Building 5 (Step 890) |
| **Ultra High-Density** | สนามกีฬา, Stadium Bowl, Convention Hall | **หลายพัน Client ต่อ Section** | นอกขอบเขต Physical Lab แต่ต้องเข้าใจหลักการ |

ตัวเลขที่ทำให้ High-Density ต่างจาก Office ไม่ใช่แค่ "จำนวน Client เยอะ" แต่คือ **ความหนาแน่นต่อ
ตารางเมตร (Client Density per m²)** — Auditorium 800 ที่นั่งในพื้นที่ 600 ตร.ม. มีความหนาแน่น
สูงกว่า Office พื้นที่เท่ากันที่มีพนักงานนั่งห่างกันเป็น 10 เท่าตัวขึ้นไป และ Client ส่วนใหญ่ใน
สถานการณ์นี้มักมี**อุปกรณ์มากกว่า 1 เครื่องต่อคน** (Laptop + Phone + Smartwatch) ทำให้ตัวเลข
Endpoint จริงยิ่งสูงกว่าจำนวนคนในห้อง

### ทำไม "เพิ่ม AP อย่างเดียว" ไม่ Scale เป็นเส้นตรง

สัญชาตญาณแรกของ Engineer มือใหม่คือ "Client เยอะก็เพิ่ม AP เข้าไปเรื่อยๆ" — แต่กฎ Physics ของ
RF ไม่ยอมให้ทำแบบนั้นได้ไม่จำกัด เพราะ **จำนวน Non-overlapping Channel มีจำกัด**

```
2.4GHz: มี Non-overlapping Channel ใช้งานจริงแค่ 3 ช่อง (1, 6, 11)
5GHz:   มี Non-overlapping Channel ประมาณ 21-25 ช่อง (ขึ้นกับ Regulatory Domain + DFS)
        (ทวนจาก Part 20 Step 191)

ถ้าเพิ่ม AP เข้าไปในพื้นที่เดิมโดยไม่ลด Cell Size/Power ให้เหมาะสม:
  → AP หลายตัวต้องใช้ Channel ซ้ำกัน (เพราะจำนวน Channel มีจำกัด)
  → เกิด Co-Channel Interference (CCI) สูงขึ้น — AP บน Channel เดียวกันต้อง "ผลัดกันพูด"
    (CSMA/CA Contention) แทนที่จะพูดพร้อมกันได้
  → ผลลัพธ์: Throughput รวมของพื้นที่นั้น "ไม่เพิ่ม" ตามจำนวน AP ที่เพิ่ม (บางกรณีแย่ลงด้วยซ้ำ
    เพราะ Overhead ของ Contention/Management Frame เพิ่มขึ้นเปล่าๆ)
```

หลักการที่ถูกต้องของ High-Density Design จึงไม่ใช่ "เพิ่ม AP" อย่างเดียว แต่คือ **ลด Cell Size
ของ AP แต่ละตัวลง (ลด Power) พร้อมกับเพิ่ม AP จำนวนมากขึ้น** เพื่อให้ AP ข้างเคียงสามารถใช้
Channel ต่างกันได้จริงโดยไม่ทับซ้อนกันเกินไป — เป็นการแลก **Coverage ต่อ AP ที่เล็กลง** กับ
**Capacity รวมของพื้นที่ที่มากขึ้น** ซึ่งขัดกับสัญชาตญาณของ Design แบบ Coverage-only ที่เรียนไว้
ตั้งแต่ Part 20

### High-Density Design Checklist เต็มรูปแบบ

| # | รายการตรวจสอบ | รายละเอียดย่อ | อ้างอิงเจาะลึกใน Step |
|---|---|---|---|
| 1 | คำนวณ Capacity ไม่ใช่แค่ Coverage | ใช้สูตร `max(Coverage AP, Capacity AP)` (ทวนจาก Part 72 Step 718) | 882 |
| 2 | ลด Cell Size + เพิ่ม AP Density | Power ต่ำลง, AP ถี่ขึ้น เพื่อให้ Reuse Channel ได้จริง | 882 |
| 3 | ปิด/จำกัด 2.4GHz, ผลัก Client ไป 5GHz | Band Steering, ปิด Low Data Rate | 882, 883 |
| 4 | จำกัด Client ต่อ AP (`max-clients`) | ป้องกัน AP ตัวเดียวรับภาระเกิน แม้ RF ไหว | 883 |
| 5 | Load-Balance ระหว่าง AP ในพื้นที่เดียวกัน | 802.11v BSS Transition (ทวนจาก Part 36 Step 355) + Aggressive Load Balancing | 883 |
| 6 | เลือก Antenna/AP Model ให้เหมาะกับรูปทรงพื้นที่ | Directional/Sectorized สำหรับ Stadium Bowl vs Omni สำหรับพื้นที่เปิดโล่ง | 884 |
| 7 | เปิด Airtime Fairness | ป้องกัน Client ช้า/เก่าตัวเดียวกิน Airtime เกินสัดส่วน | 885 |
| 8 | พิจารณา 6GHz (Wi-Fi 6E) ถ้า Hardware รองรับ | Channel เพิ่มมหาศาล ลด CCI ได้จริง | 886 |
| 9 | เปิด CAC สำหรับ Voice/Video | ป้องกัน Call Setup เกิน Capacity ที่ RF รองรับได้จริง | 887 |
| 10 | เตรียมแผน Temporary Capacity สำหรับ Event ระยะสั้น | AP เสริม, RF Profile ชั่วคราว, DHCP Scope แยก | 888 |
| 11 | ตั้ง Monitoring/Threshold เฉพาะพื้นที่ High-Density | Client Count, Channel Utilization Trend ผ่าน Assurance | 889 |
| 12 | Manual Override RRM ถ้าจำเป็น | RRM Auto บางครั้งไม่ Aggressive พอในพื้นที่ AP หนาแน่นมาก | 882 |
| 13 | ทดสอบ RF Survey จริงก่อน Go-live | Predictive Survey (แบบจำลอง) + Validation Survey (วัดจริงหลัง Deploy) | 890 |
| 14 | เปิด Fast Roaming (802.11r/k/v) | จำเป็นมากขึ้นในพื้นที่ที่ Client เดินผ่าน Cell เล็กจำนวนมาก | ทวนจาก Part 36 Step 355 |

Checklist นี้คือแกนของทั้ง Part — แต่ละ Step ที่เหลือจะขยายรายการใดรายการหนึ่งให้ลึกถึงระดับ
Config จริง

---

## Step 882 — RF Planning for High Density

### Cell Size Reduction — หลักการที่ตรงข้ามกับ Coverage Design

Coverage Design ทั่วไป (ที่เรียนใน Part 20 และ Part 36 Step 353) ตั้งเป้า RSSI Cell-edge ที่
-67 dBm (Voice) หรือ -70 ถึง -75 dBm (Data) ด้วย Power ที่**สูงพอ**ให้ Cell แต่ละตัวครอบคลุม
พื้นที่ให้มากที่สุดโดยไม่เสีย Overlap 15-20% — **High-Density Design กลับหลักการนี้**: ตั้งเป้า
Cell ให้**เล็กที่สุดที่ยังคุม RSSI เป้าหมายได้** เพื่อบีบให้ AP มากขึ้นอยู่ในพื้นที่เดิมได้โดยไม่ชน
Channel กัน

```
Coverage Design (Office ทั่วไป):        High-Density Design (Auditorium):
                                         
    AP           AP                         AP  AP  AP  AP  AP
  ⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤⬤              ⬤⬤⬤  ⬤⬤⬤  ⬤⬤⬤  ⬤⬤⬤  ⬤⬤⬤
  Cell ใหญ่ ครอบคลุมกว้าง                  Cell เล็ก, Power ต่ำ, AP ถี่
  Client/AP น้อย (พื้นที่ใหญ่ Client น้อย)   Client/AP สูง แต่กระจายให้ทุก AP รับภาระเท่ากัน
```

### Power Planning ที่ปรับจาก Step 358

Part 36 Step 358 สร้าง RF Profile `RF-HIGH-DENSITY` ไว้แล้วด้วยค่า `tx-power-min/max` ที่แคบกว่า
Default — Part นี้ปรับให้เข้มขึ้นอีกระดับสำหรับพื้นที่ระดับ Auditorium/Stadium จริง:

```
WLC-1# configure terminal

! ปรับ RF-HIGH-DENSITY (จาก Step 358) ให้ Power ต่ำลงอีก เพื่อรองรับ AP ที่ถี่ขึ้นมาก
WLC-1(config)# ap dot11 5ghz rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-profile)#  tx-power-min 7
WLC-1(config-rf-profile)#  tx-power-max 11
WLC-1(config-rf-profile)#  channel-width 20
WLC-1(config-rf-profile)#  rx-sop threshold high
WLC-1(config-rf-profile)#  no shutdown
WLC-1(config-rf-profile)#  exit

WLC-1(config)# ap dot11 24ghz rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-profile)#  tx-power-min 1
WLC-1(config-rf-profile)#  tx-power-max 5
WLC-1(config-rf-profile)#  no shutdown
WLC-1(config-rf-profile)#  exit
```

| Parameter ใหม่ที่เพิ่มจาก Step 358 | ความหมาย |
|---|---|
| `channel-width 20` | บังคับ 5GHz ให้ใช้ Channel Width 20MHz (แคบลง) เพื่อให้มี Non-overlapping Channel **มากขึ้น** ในพื้นที่เดิม — ตรงข้ามกับ Office ทั่วไปที่มักใช้ 40/80MHz เพื่อ Throughput สูงสุดต่อ Client |
| `rx-sop threshold high` | **RX-SOP (Receiver Start of Packet)** — ปรับให้ AP "เพิกเฉย" Signal ที่อ่อนเกินไปจาก Client/AP ไกลๆ ไม่พยายามถอดรหัส Frame ที่ไม่เกี่ยวกับตัวเอง ลด CCI effective ที่ AP รับรู้ |

> **ทำไม Channel Width แคบลงถึงช่วย High-Density**: Channel 80MHz ใน 5GHz กินพื้นที่เท่ากับ
> Channel 20MHz 4 ช่องรวมกัน — ถ้าใช้ 80MHz ทุก AP จะเหลือ Non-overlapping Channel ให้เลือก
> น้อยกว่ามาก (Reuse Distance ต้องกว้างขึ้น) ในพื้นที่ Auditorium ที่ AP ต้องอยู่ใกล้กันมาก การ
> ใช้ 20MHz ทำให้มี Channel ให้ Reuse ได้มากพอ แม้ Throughput สูงสุดต่อ Client จะลดลง — ยอมรับ
> ได้เพราะ Client แต่ละตัวใน High-Density ไม่ได้ต้องการ Throughput สูงสุดอยู่แล้ว (เน้น Capacity
> รวม ไม่ใช่ Speed ต่อเครื่อง)

### 5GHz-Preference Design และ Band Steering

เป้าหมายคือ**ผลักดัน Client ที่รองรับ 5GHz ให้ใช้ 5GHz เป็นหลัก** เพื่อลดภาระบน 2.4GHz ที่มี
Channel น้อยกว่ามาก (3 Channel เทียบกับ 20+ Channel) — กลไกหลักคือ **Band Select**

```
WLC-1(config)# ap dot11 24ghz band-select
WLC-1(config)# ap dot11 24ghz band-select cycle-count 3
WLC-1(config)# ap dot11 24ghz band-select cycle-threshold 200
WLC-1(config)# ap dot11 24ghz band-select expire suppression 20
WLC-1(config)# ap dot11 24ghz band-select expire dual-band 60
WLC-1(config)# ap dot11 24ghz band-select client-rssi -80
```

| Parameter | ความหมาย |
|---|---|
| `cycle-count 3` | AP จะไม่ตอบ Probe Request ของ Client บน 2.4GHz จนครบ 3 ครั้ง (บังคับให้ Client "มองไม่เห็น" 2.4GHz ช่วงแรก เพื่อดูว่า Client รองรับ 5GHz ไหม) |
| `cycle-threshold 200` | ระยะเวลา (ms) ที่นับ Probe แต่ละครั้งภายใน Cycle |
| `expire suppression 20` | หลัง Suppress Probe บน 2.4GHz แล้ว 20 วินาที ถ้า Client ยังไม่ Associate ที่ 5GHz ให้เริ่มตอบ 2.4GHz ตามปกติ (ป้องกัน Client ที่รองรับแค่ 2.4GHz เชื่อมต่อไม่ได้เลย) |
| `expire dual-band 60` | ถ้า Client เคย Associate ทั้งสอง Band มาก่อน ให้จำสถานะไว้ 60 วินาที |
| `client-rssi -80` | Threshold ขั้นต่ำของ RSSI ที่ AP จะพิจารณา Band Select เลย (Client ที่ไกลเกินไปไม่ต้องเสียเวลาผลักไป 5GHz เพราะสัญญาณ 5GHz ที่ Attenuate เร็วกว่าจะยิ่งแย่กว่า) |

> **ข้อควรระวัง CCIE-level**: Band Select ทำงานที่ระดับ **Probe Response** เท่านั้น — ถ้า Client
> ส่ง Association Request ตรงมาที่ 2.4GHz โดยไม่ Probe ก่อน (Client บางรุ่นทำแบบนี้) Band Select
> จะไม่มีผล AP ยังต้อง Associate ให้ตามปกติ — Band Select จึงเป็น "แรงจูงใจ" ไม่ใช่ "การบังคับ"
> การบังคับจริงต้องปิด Radio 2.4GHz ทั้งหมดบางส่วนของ AP (ดู Step 884)

### ปิดการใช้งาน Low Data Rate — อีกกลไกที่ช่วยลด Cell Size โดยอ้อม

```
WLC-1(config)# ap dot11 24ghz rate 1 disable
WLC-1(config)# ap dot11 24ghz rate 2 disable
WLC-1(config)# ap dot11 24ghz rate 5.5 disable
WLC-1(config)# ap dot11 24ghz rate 11 mandatory
```

การปิด Data Rate ต่ำ (1/2/5.5 Mbps) บังคับให้ Client ที่อยู่ไกลเกินไป (RSSI อ่อนจนใช้ได้แค่ Rate
ต่ำ) ต้อง Roam ไป AP ตัวอื่นที่ใกล้กว่า แทนที่จะเกาะ AP เดิมด้วย Rate ต่ำซึ่งใช้ **Airtime นาน
กว่าต่อ Byte ข้อมูล** (ทวนรายละเอียดเรื่อง Airtime ใน Step 885) — ผลคือ Cell Size ที่ "มีผลจริง"
เล็กลงโดยไม่ต้องลด Power เพิ่ม และ Roaming เร็วขึ้นเพราะ Client edge-of-cell ถูกดันให้ย้ายก่อน
สัญญาณจะแย่จนหลุด

---

## Step 883 — Client Capacity Management

### `max-clients` — จำกัด Client ต่อ AP ไม่ให้เกิน Capacity ที่ออกแบบไว้

แม้ RF จะรองรับ Client จำนวนมากในทางทฤษฎี แต่ AP แต่ละตัวมี CPU/Memory จำกัดในการประมวลผล
Association/Key Management — Cisco AP รุ่น Enterprise ทั่วไปกำหนด Hard Limit ไว้ที่ **200
Client ต่อ Radio** แต่ค่าที่ใช้งานจริงใน High-Density ควรตั้งให้ต่ำกว่านั้นมาก เพื่อคุม
Experience ต่อ Client ให้ดีอยู่

```
WLC-1# configure terminal

! ตั้งค่า max-clients ระดับ WLAN/Policy Profile
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)#  client association limit 40
WLC-1(config-wireless-policy)#  exit

! ตั้งค่า max-clients ระดับ AP โดยตรง (ต่อ Radio)
WLC-1(config)# ap name AP-AUD-01
WLC-1(config-ap-tag)#  exit
WLC-1(config)# wireless profile policy HIGH-DENSITY-POLICY
WLC-1(config-wireless-policy)#  client association limit per-wlan 40
WLC-1(config-wireless-policy)#  exit
```

| ระดับ Config | ผลลัพธ์ | ใช้เมื่อ |
|---|---|---|
| `client association limit` (Policy Profile) | จำกัด Client ต่อ WLAN บน AP ตัวนั้นๆ | ต้องการคุม Client ต่อ SSID โดยเฉพาะ |
| `client association limit per-wlan` | เหมือนข้างบนแต่ระบุชัดว่าเป็น per-WLAN บน Policy Profile เดียวกัน | AP ตัวเดียวมีหลาย SSID ที่ต้องคุมสัดส่วนต่างกัน |
| Global Client Limit (`wireless client association limit`) | จำกัด Client รวมทั้ง Controller | ใช้ป้องกัน WLC Ceiling (ทวนจาก Part 72 Step 718) |

> **ทำไมต้องจำกัดแม้ RF ยังไหว**: เมื่อ Client ถึง Limit ที่ AP ตัวหนึ่ง WLC จะปฏิเสธ Association
> Request ใหม่ (`Association Response: Reject — AP busy`) บังคับให้ Client Probe หา AP ตัวอื่น
> ที่ยังมี Capacity เหลือ — นี่คือกลไกพื้นฐานที่ทำให้ **Load-Balancing** ทำงานได้จริง เพราะถ้าไม่
> จำกัดไว้ Client จะเลือก Associate กับ AP ที่ RSSI แรงที่สุดเสมอ (ซึ่งมักเป็น AP ตัวเดียวกันสำหรับ
> Client ที่นั่งกลุ่มเดียวกัน) จนตัวนั้นล้นในขณะที่ AP ข้างเคียงยังว่าง

### Load-Balancing ระหว่าง AP ในพื้นที่เดียวกัน

```
WLC-1(config)# wireless load-balancing window 5
WLC-1(config)# wireless load-balancing denial 3
```

| Parameter | ความหมาย |
|---|---|
| `window 5` | AP จะพิจารณาตัวเองว่า "โหลดสูงกว่าเพื่อนบ้าน" เมื่อจำนวน Client มากกว่า AP ข้างเคียงเกิน Window ค่านี้ (Client) |
| `denial 3` | จำนวนครั้งสูงสุดที่ AP จะ "Soft-reject" Client ใหม่ (ส่ง Association Response Status 17 - AP busy กลับไปก่อน) เพื่อบีบให้ Client ลองหา AP ตัวอื่นก่อนจะยอมรับจริงในครั้งถัดไปถ้าไม่มีทางเลือก |

Load-Balancing ทำงานร่วมกับ **802.11v BSS Transition Management** ที่เรียนไปแล้วใน Part 36
Step 355 — ต่างกันที่ BSS Transition คือ WLC "ขอ" ให้ Client ที่ Associate อยู่แล้วย้ายออกไปเชิง
รุก ส่วน Load-Balancing คือ WLC "ปฏิเสธชั่วคราว" ตอน Client กำลังจะ Associate ใหม่ — ใช้คู่กันจึง
ครอบคลุมทั้งสองจังหวะ (ตอนเข้าใหม่ และตอนอยู่แล้วแต่ Load ไม่สมดุล)

### Band-Select ผลักดัน Client ที่รองรับได้ไปยัง 5/6GHz (ทวนจาก Step 882 ในมุม Capacity)

Step 882 อธิบาย Band Select ในมุม RF Planning (ลดภาระ 2.4GHz) — ในมุม Capacity Management
Band Select คือเครื่องมือที่ทำให้ **Client กระจายตัวข้าม 3 Band (2.4/5/6GHz) แทนที่จะกระจุกอยู่
Band เดียว** — ยิ่งมี Band ให้กระจายมากเท่าไหร่ (โดยเฉพาะเมื่อมี 6GHz ตาม Step 886) Capacity
รวมของพื้นที่เดียวกันก็สูงขึ้นตามจำนวน Band ที่ใช้งานได้จริงคูณกับจำนวน Non-overlapping Channel
ของแต่ละ Band

```
สรุปกลไกทั้งหมดของ Step 882-883 ที่ทำงานร่วมกัน:

Client เดินเข้าพื้นที่ High-Density
   → Band Select: พยายามผลักไป 5GHz/6GHz ก่อน (ลดภาระ 2.4GHz)
   → Load-Balancing: ถ้า AP ที่ RSSI แรงสุดโหลดเยอะ ปฏิเสธชั่วคราว ให้ไป AP ข้างเคียง
   → max-clients: AP แต่ละตัวไม่รับเกิน Capacity ที่ออกแบบไว้ ไม่ว่า RF จะไหวหรือไม่
   → 802.11v BSS Transition: ถ้า Client Associate ไปแล้วแต่ AP โหลดไม่สมดุล ขอให้ย้ายเชิงรุก
```

---

## Step 884 — High-Density Hardware Considerations

### รูปทรงพื้นที่กำหนดชนิด Antenna — หลักการที่ต่างจาก Office แบบสิ้นเชิง

AP รุ่น Enterprise ของ Cisco (ตระกูล Catalyst 9100) มีสองแบบหลักตามชนิด Antenna:

| รุ่น (ตัวอย่าง) | ชนิด Antenna | เหมาะกับพื้นที่ | เหตุผล |
|---|---|---|---|
| Catalyst 9130AXI (Internal) | Internal Omni-directional | Office เปิดโล่ง, ห้องประชุมทั่วไป, Corridor | กระจายสัญญาณรอบตัว 360° เท่ากันทุกทิศทาง เหมาะกับพื้นที่ที่ Client กระจายตัวสม่ำเสมอ |
| Catalyst 9130AXE (External) | External — ต่อ Directional/Patch/Sector Antenna ได้ | **Stadium Bowl, Auditorium แถวที่นั่งเป็นแนว, Warehouse ทางแคบยาว** | โฟกัสพลังงาน RF ไปยังทิศทางที่มี Client จริง ไม่กระจายไปยังพื้นที่ที่ไม่มีคนนั่ง (เช่น เพดานสูง หรือ Section ข้างเคียงที่มี AP ของตัวเองอยู่แล้ว) |

### ทำไม Stadium Seating ต้องใช้ Sectorized/Directional Antenna

```
Auditorium ที่นั่งเป็นแถว มองจากด้านบน (Bird's-eye View):

Omni Antenna (ผิด):                    Sectorized/Directional Antenna (ถูก):
                                        
     ╱────AP────╲                          ▼AP▼  (Antenna ยิงลงเป็นมุมแคบ
    ╱   วง360°   ╲                          │    ครอบคลุมเฉพาะ Section
   │  กระจายรอบตัว │                        ▼▼▼    ที่นั่งด้านล่างตัวเอง)
   │  ล้น Section  │                       [ที่นั่ง Section A]
   │  ข้างเคียง    │                       
    ╲             ╱                       [Section B ใช้ AP+Antenna ของตัวเอง
     ╲───────────╱                          แยกกันคนละ Sector ไม่ทับ Section A]
```

Antenna แบบ Directional/Sector "บังคับ" พลังงาน RF ให้ตกอยู่ในพื้นที่ที่นั่งจริงของ Section ตัวเอง
เท่านั้น ไม่ล้นไปกวน Channel ของ Section ข้างเคียง — ทำให้สามารถ **Reuse Channel เดียวกันได้ถี่
กว่า** เมื่อเทียบกับการใช้ Omni Antenna ที่กระจายพลังงานไปทุกทิศทางเท่ากัน (รวมถึงทิศทางที่ไม่มี
Client อยู่เลย เช่น ขึ้นไปในอากาศเหนือหัวคนดู) — เป็นการแปลง "Power ที่เสียเปล่า" เป็น "Cell ที่
เล็กลงตามรูปทรงพื้นที่จริง" ซึ่งตรงกับหลักการ Cell Size Reduction ของ Step 882

### ตารางเปรียบเทียบ Antenna Pattern ที่ใช้บ่อยใน High-Density Design

| Antenna Pattern | มุมกระจาย (Beamwidth) | Gain โดยประมาณ | ใช้เมื่อ |
|---|---|---|---|
| Omni-directional | 360° (รอบตัว) | ~4-6 dBi | Office เปิดโล่ง, พื้นที่ที่ Client กระจายรอบทิศทาง |
| Patch/Panel (Directional) | 60-120° | ~7-10 dBi | ผนังด้านข้าง Auditorium, Corridor ทางเดินยาว |
| Sector Antenna | 60-90° ต่อ Sector | ~8-13 dBi | Stadium Bowl, ติดหลาย Sector รอบ AP ตัวเดียวหรือหลายตัวแบ่งมุม |
| Downtilt Antenna | มุมเงยลงจากเพดาน/เสาสูง | ~10-14 dBi | เพดานสูงมาก (Stadium, Convention Hall ที่ AP ต้องแขวนสูง) เพื่อยิง RF ลงมาที่คนแทนกระจายขึ้นเพดาน |

### ข้อพิจารณาเรื่อง Mounting Height/Tilt

พื้นที่เพดานสูง (Stadium, Convention Hall) ทำให้ AP ต้องแขวนสูงกว่าปกติมาก — ถ้าใช้ Antenna
Pattern แบบ Office ทั่วไป (ออกแบบมาสำหรับเพดานสูง 3-4 เมตร) RF ส่วนใหญ่จะยิงลงมาไม่ถึงระดับ
คนนั่ง หรือยิงเป็นมุมกว้างเกินจนครอบคลุมพื้นที่ที่ไม่มีคน — ต้องเลือก Antenna ที่มี **Downtilt**
ตามความสูงจริงของการติดตั้ง และมักต้องใช้ **RF Predictive Survey Software** (เช่น Ekahau,
iBwave) จำลองมุม Antenna ก่อน Deploy จริงเสมอ ไม่ใช่แค่คำนวณจาก Checklist ทั่วไป — จะกล่าวถึง
ขั้นตอนนี้อีกครั้งใน Step 890

---

## Step 885 — Airtime Fairness และ QoS at High Density

### ปัญหา: Client ช้าตัวเดียวกิน Airtime มากกว่า Client เร็วหลายตัวรวมกัน

หลักการที่มักถูกเข้าใจผิดคือ "AP แบ่ง Bandwidth เท่ากันให้ทุก Client" — ความจริงคือ Wi-Fi
(CSMA/CA) แบ่ง**เวลาการเข้าถึง Channel (Airtime)** ไม่ใช่แบ่ง Bandwidth โดยตรง และ Client ที่ใช้
Data Rate ต่ำ (เช่น 802.11b/g เก่า หรือ Client ที่อยู่ Cell-edge สัญญาณอ่อน) ต้องใช้**เวลานานกว่า
มาก**ในการส่งข้อมูลปริมาณเท่ากันเมื่อเทียบกับ Client 802.11ac/ax ที่ใช้ Data Rate สูง

```
ตัวอย่าง: ส่งข้อมูลขนาดเท่ากัน (เช่น 1 MB)

Client A (802.11ax, MCS สูง, Data Rate ~600 Mbps): ใช้ Airtime ~13ms
Client B (802.11g เก่า/Cell-edge, Data Rate ~6 Mbps): ใช้ Airtime ~1,300ms (100 เท่า!)

ถ้า Client A และ B แข่งกันส่งพร้อมกันบน AP เดียวกัน โดยไม่มี Airtime Fairness:
  → Client B ครอง Channel นานกว่า Client A ถึง 100 เท่าต่อ Byte เท่ากัน
  → Client A (และ Client อื่นๆที่เร็ว) ได้ Throughput ต่ำลงมาก ทั้งที่ตัวเองไม่ได้ทำอะไรผิด
  → นี่คือปัญหาคลาสสิกที่เรียกว่า "One Slow Client Kills Everyone" ซึ่งรุนแรงมากใน
    High-Density ที่มี Client จำนวนมากแข่งกันบน AP เดียว
```

### Airtime Fairness (ATF) Policy — Config

```
WLC-1# configure terminal

! สร้าง ATF Policy กำหนดสัดส่วน Airtime ขั้นต่ำที่ SSID/Client Group นี้ควรได้รับ
WLC-1(config)# wireless profile atf-policy ATF-ENTERPRISE-WIFI
WLC-1(config-atf-policy)#  atf-weight 60
WLC-1(config-atf-policy)#  no shutdown
WLC-1(config-atf-policy)#  exit

! เปิดใช้ ATF บน Radio 5GHz/2.4GHz
WLC-1(config)# ap dot11 5ghz atf
WLC-1(config)# ap dot11 24ghz atf

! ผูก ATF Policy เข้ากับ Policy Profile ของ WLAN
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)#  atf-policy ATF-ENTERPRISE-WIFI
WLC-1(config-wireless-policy)#  exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

| Parameter | ความหมาย |
|---|---|
| `atf-weight 60` | สัดส่วน Airtime (จาก 100 หน่วยรวม) ที่ SSID/Client Group นี้ **ได้รับประกัน** เป็นขั้นต่ำ ไม่ว่า Client ตัวใดใน Group จะช้าเพียงใด Client ตัวอื่นนอก Group จะไม่ถูกแบ่ง Airtime เกินสัดส่วนที่เหลือ |
| `ap dot11 {5ghz\|24ghz} atf` | เปิดกลไก ATF ที่ระดับ Radio — ไม่เปิดคำสั่งนี้ ATF Policy จะไม่มีผลจริงแม้สร้าง Policy ไว้แล้ว |

### ATF ต่างจาก Airtime Fairness แบบ "Strict" อย่างไร

Cisco รองรับ ATF สองโหมด: **Fair-share** (ที่แสดงข้างบน — แบ่งตามสัดส่วน Weight ระหว่าง Group
เท่านั้น ยังปล่อยให้ Client ช้าใน Group เดียวกันแข่งกันตามปกติ) และ **Strict** (จำกัด Airtime
สูงสุดของ Group นั้นไม่ให้เกิน Weight ที่กำหนด แม้ Group อื่นไม่ได้ใช้ Airtime เต็มโควต้าตัวเอง)
— High-Density มักเลือก **Fair-share** เพราะต้องการแค่ป้องกันไม่ให้ SSID ใดครองทั้งหมด แต่ยัง
ต้องการให้ Airtime ที่เหลือ (จาก SSID ที่ใช้ไม่เต็มโควต้า) ไหลไปให้ SSID อื่นที่ต้องการมากกว่าได้

### เชื่อมกับ QoS/WMM ที่เรียนไปแล้วใน Part 24/40/82

ATF แก้ปัญหาที่ระดับ "SSID/Client Group" — แต่ **ภายใน SSID เดียวกัน** ปัญหาการแบ่งลำดับความ
สำคัญของ Traffic (Voice ต้องมาก่อน Data) ยังต้องพึ่ง **WMM/802.11e** ที่เรียนไปแล้วอย่างละเอียด
ใน [Part 82 Step 815](part-082-end-to-end-enterprise-qos-design.md) — ตารางด้านล่างทวนสั้นๆ
เพื่อผูกกับบริบท High-Density:

| Access Category | User Priority | DSCP ที่ WLC-1 ต้อง Map ให้ตรง Enterprise Baseline | ผลใน High-Density |
|---|---|---|---|
| AC_VO (Voice) | 6-7 | EF (46) | ต้องมาก่อนเสมอแม้ Airtime ทั้งพื้นที่ตึงมาก — เชื่อมกับ CAC ใน Step 887 |
| AC_VI (Video) | 4-5 | AF41 (34) | รองจาก Voice — สำคัญมากถ้า Auditorium มีถ่ายทอด Video Conference |
| AC_BE (Best Effort) | 0, 3 | DF (0) | Traffic ทั่วไปส่วนใหญ่ของ Client ใน Auditorium |
| AC_BK (Background) | 1-2 | CS1 (8) | Traffic พื้นหลัง เช่น OS Update — ควรถูกจำกัดมากที่สุดในพื้นที่แน่น |

ATF และ WMM ทำงานคนละ "แกน" ที่ **เสริมกัน**: ATF แบ่ง Airtime ระหว่าง**กลุ่ม Client/SSID**
(แนวนอน) ส่วน WMM แบ่งลำดับความสำคัญของ**ประเภท Traffic ภายในกลุ่มเดียวกัน** (แนวตั้ง) — พื้นที่
High-Density ที่ดีต้องเปิดทั้งสองกลไกพร้อมกัน ไม่ใช่เลือกอย่างใดอย่างหนึ่ง

---

## Step 886 — 6GHz (Wi-Fi 6E) และ High-Density Design

### 6GHz คือ "พื้นที่ Spectrum ใหม่ทั้งหมด" ไม่ใช่แค่ Speed ที่เร็วขึ้น

Wi-Fi 6E เพิ่ม **แถบความถี่ 6GHz (5.925-7.125 GHz)** เข้ามาเป็น Band ที่ 3 (นอกจาก 2.4GHz และ
5GHz) — สิ่งที่สำคัญที่สุดสำหรับ High-Density Design ไม่ใช่ความเร็วที่เพิ่มขึ้น แต่คือ **จำนวน
Non-overlapping Channel ที่เพิ่มขึ้นมหาศาล**

| Band | Spectrum รวม | จำนวน Channel 20MHz Non-overlapping (โดยประมาณ) | สภาพ Spectrum |
|---|---|---|---|
| 2.4GHz | ~80 MHz | 3 Channel | แน่นมาก, มี Interference จากอุปกรณ์ Non-Wi-Fi เยอะ (Bluetooth, Microwave) |
| 5GHz | ~500 MHz (รวม DFS) | ~21-25 Channel | ใช้งานได้ดี แต่ Enterprise ใหญ่เริ่มแน่นในพื้นที่ High-Density |
| **6GHz** | **~1,200 MHz** | **มากถึง ~59 Channel (20MHz)** | **สะอาดสนิท — ไม่มี Legacy Device เก่าค้างอยู่เลย เพราะเป็น Band ใหม่ที่ Wi-Fi 6E/7 เท่านั้นที่เข้าถึงได้** |

จำนวน Channel ที่มากกว่า 2.4GHz+5GHz รวมกันหลายเท่า ทำให้ปัญหาหลักของ High-Density (Channel
ไม่พอ ต้อง Reuse จนเกิด CCI ตาม Step 881) **บรรเทาลงอย่างมีนัยสำคัญ** — ในทางทฤษฎีสามารถวาง AP
ถี่มากในพื้นที่เดียวโดยแต่ละตัวใช้ Channel ที่ไม่ซ้ำกันเลย

### Trade-off ที่ต้องเข้าใจ: Range สั้นลง = ต้องมี AP มากขึ้น (แต่ Interference น้อยลงมาก)

```
กฎ Physics พื้นฐาน: ความถี่สูงขึ้น → Free Space Path Loss สูงขึ้น → ระยะที่สัญญาณใช้งานได้สั้นลง

2.4GHz: ระยะไกลสุด (ทะลุกำแพงได้ดีกว่า)
5GHz:   ระยะกลาง
6GHz:   ระยะสั้นสุด (Attenuate เร็วที่สุด จากทั้ง 3 Band)

ผลลัพธ์สำหรับ High-Density:
  ต้องการ AP มากขึ้น (Cell เล็กลงตามธรรมชาติของ 6GHz อยู่แล้ว)
  แต่แลกมาด้วย Channel ที่เพียงพอมากจนไม่ต้อง "แย่ง" กันแบบ 2.4/5GHz
  → นี่คือ Trade-off ที่ "คุ้มค่า" สำหรับ High-Density โดยเฉพาะ เพราะ High-Density Design
    (Step 882) ต้องการ Cell เล็กอยู่แล้วเป็นปกติ — 6GHz แค่ทำให้สิ่งที่ต้องทำอยู่แล้ว
    "ได้ผลดีขึ้น" โดยไม่มี CCI แถมมาด้วย
```

### ข้อพิจารณาเฉพาะของ 6GHz ที่ CCIE ต้องรู้

| หัวข้อ | รายละเอียด |
|---|---|
| **WPA3 บังคับ** | 6GHz **ไม่รองรับ** WPA2/Open Authentication เดิมเลย — ทุก SSID บน 6GHz ต้องใช้ WPA3 (Personal หรือ Enterprise) เท่านั้น เป็นข้อบังคับตาม Spec ไม่ใช่ตัวเลือก |
| **PSC (Preferred Scanning Channels)** | 6GHz กำหนด Channel ชุดหนึ่งเป็น "PSC" ที่ Client ควร Scan ก่อนเป็นอันดับแรก (ลดเวลา Scan ทั้ง 59 Channel ที่จะช้าเกินไป) — WLC ควรตั้ง DCA ให้เลือก PSC เป็นหลักเพื่อ Client ค้นหาเจอเร็ว |
| **Standard Power vs Low Power Indoor (LPI)** | AP บางรุ่น/ประเทศต้องขึ้นทะเบียนผ่านระบบ **AFC (Automated Frequency Coordination)** ถ้าต้องการใช้ Power ระดับ Standard (Outdoor/ระยะไกล) — ส่วน LPI ใช้ได้ทันทีในอาคารโดยไม่ต้องขึ้นทะเบียน ซึ่งเพียงพอสำหรับ Auditorium/Indoor High-Density ส่วนใหญ่ |
| **AP/Client ต้องรองรับ 6GHz ทั้งคู่** | AP ต้องเป็นรุ่น Wi-Fi 6E/7 (เช่น Catalyst 9136, 9166) และ Client (Laptop/Phone) ต้องมี Chipset รองรับ 6GHz ด้วย — ช่วง Transition ที่ Client ยังไม่รองรับ 6GHz ทั่วถึง ต้องออกแบบ **Hybrid** (2.4/5/6GHz พร้อมกัน) ไปก่อน |

### ผลต่อ Design High-Density โดยรวม

```
สถาปัตยกรรม High-Density แบบ 3-Band (ปัจจุบัน, Hybrid ระยะ Transition):

Client เดินเข้าพื้นที่
   → AP ประกาศ SSID เดียวกันบน 2.4/5/6GHz ทั้ง 3 Band (Same SSID name, Broadcast ทุก Band)
   → Client ที่รองรับ 6GHz (ใหม่) → Band Select ผลักไปที่ 6GHz ก่อน (Channel เยอะสุด, CCI น้อยสุด)
   → Client ที่รองรับแค่ 5GHz (ส่วนใหญ่ปัจจุบัน) → ไปที่ 5GHz ตามปกติ (Step 882)
   → Client เก่าที่รองรับแค่ 2.4GHz → ยังใช้งานได้ (Band Select ไม่บล็อกเด็ดขาด ทวนจาก Step 882)

ผลลัพธ์: ภาระที่เคยกระจุกอยู่บน 5GHz อย่างเดียว ถูกแบ่งออกไปยัง 6GHz เพิ่มอีกชั้น
         → Capacity รวมของพื้นที่สูงขึ้นตามสัดส่วน Client ที่ Migrate ไป 6GHz ได้จริง
```

---

## Step 887 — Voice/Video over High-Density Wi-Fi: CAC (Call Admission Control)

### ปัญหาที่ WMM Priority เพียงอย่างเดียวแก้ไม่ได้

Step 885 อธิบายไปแล้วว่า WMM ทำให้ Voice (AC_VO) มี Priority สูงสุดเสมอเมื่อแข่งกับ Traffic
ประเภทอื่น — แต่ WMM **ไม่ได้จำกัดจำนวน Call ที่เกิดขึ้นพร้อมกัน** ถ้ามี VoWLAN Call จำนวนมาก
เกินกว่าที่ Channel รองรับได้จริง (แม้ Voice จะได้ Priority สูงสุด) **ทุก Call ที่มีอยู่แล้วจะ
เริ่มเสื่อมคุณภาพพร้อมกันหมด** (Jitter/Packet Loss สูงขึ้นทุก Call) เพราะ Airtime รวมที่ Voice
ต้องการเกิน Capacity ของ Channel ไปแล้ว — นี่คือปัญหาที่ **CAC (Call Admission Control)**
ถูกออกแบบมาแก้โดยเฉพาะ: **ปฏิเสธ Call ใหม่ที่จะทำให้เกิน Threshold แต่รักษาคุณภาพของ Call ที่มี
อยู่แล้วไว้ให้ดี** — เป็นหลักการเดียวกับ CAC ของระบบโทรศัพท์ดั้งเดิม (PSTN) ที่ยอมให้สายไม่ผ่าน
("Fast Busy") ดีกว่าปล่อยให้ทุกสายที่กำลังคุยอยู่มีเสียงขาดหายพร้อมกัน

### Static CAC vs Enhanced/Adaptive Voice CAC

| ประเภท | กลไก | ข้อจำกัด |
|---|---|---|
| **Static CAC** | จำกัดจำนวน Call ตาย Fix ต่อ AP (เช่น "ไม่เกิน 10 Call ต่อ AP") | ไม่ปรับตามสภาพ RF จริง — บางช่วง RF ดีอาจรับได้มากกว่า บางช่วง RF แย่ควรรับน้อยกว่า |
| **Enhanced/Adaptive Voice CAC (ใช้ TSPEC)** | Client ที่รองรับ 802.11e **ส่ง TSPEC (Traffic Specification)** แจ้ง Bandwidth ที่ Call นั้นต้องการล่วงหน้า → AP/WLC คำนวณ Airtime ที่ Call ใหม่จะใช้เทียบกับ Airtime ที่เหลือจริงตอนนั้น → อนุมัติ/ปฏิเสธตามสภาพจริง | ต้องการ Client ที่รองรับ 802.11e/TSPEC (VoWLAN Phone รุ่น Enterprise ส่วนใหญ่รองรับ, Softphone บาง Client ไม่รองรับ) |

### Config: เปิด CAC บน 9800

```
WLC-1# configure terminal

! เปิด Voice CAC (ACM - Admission Control Mandatory) บน 5GHz
WLC-1(config)# ap dot11 5ghz cac voice acm
WLC-1(config)# ap dot11 5ghz cac voice max-bandwidth 75
WLC-1(config)# ap dot11 5ghz cac voice roam-bandwidth 6

! เปิด Video CAC เช่นเดียวกัน (ถ้ามี Video Conference ผ่าน WLAN นี้)
WLC-1(config)# ap dot11 5ghz cac video acm
WLC-1(config)# ap dot11 5ghz cac video max-bandwidth 40

! ทำเช่นเดียวกันบน 2.4GHz ถ้ายังมี VoWLAN Client เก่าที่ใช้ Band นี้
WLC-1(config)# ap dot11 24ghz cac voice acm
WLC-1(config)# ap dot11 24ghz cac voice max-bandwidth 75
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

| Parameter | ความหมาย |
|---|---|
| `cac voice acm` | เปิด CAC แบบ **Mandatory** — Client ที่ไม่รองรับ TSPEC จะถูกปฏิเสธไม่ให้ใช้ AC_VO เลย (ตกไปอยู่ Best Effort แทน) เพื่อป้องกัน Client ที่ "โกหก" ว่าเป็น Voice โดยไม่ทำ TSPEC จริง |
| `max-bandwidth 75` | เปอร์เซ็นต์สูงสุดของ Channel Capacity ที่ยอมให้ Voice Call รวมกันใช้ได้ (Cisco Default 75%) — เกินกว่านี้ Call ใหม่จะถูกปฏิเสธ |
| `roam-bandwidth 6` | สำรอง Bandwidth ไว้ 6% สำหรับ Call ที่กำลัง **Roam เข้ามา** จาก AP อื่น (Client ที่คุยอยู่แล้วเดินเข้ามาในพื้นที่ AP นี้) — ให้ Priority สูงกว่า Call ใหม่ที่ยังไม่เคยเริ่ม เพราะการตัด Call ที่กำลังคุยอยู่แย่กว่าการปฏิเสธ Call ใหม่ |

### ทำไม CAC สำคัญเป็นพิเศษใน High-Density

พื้นที่ High-Density มี Client จำนวนมาก แม้จะมีสัดส่วน VoWLAN Client เพียงส่วนน้อย (เช่น Staff
ที่ใช้ Wi-Fi Phone ใน Auditorium ระหว่างจัด Event) แต่ AP ตัวเดียวต้องรองรับ Data Client จำนวน
มากพร้อมกันตาม Step 883 — ถ้าไม่มี CAC จำกัดจำนวน Voice Call ไว้ล่วงหน้า **Burst ของ Data
Traffic จากผู้ชมจำนวนมาก** (เช่น ทุกคนพร้อมใจกัน Upload รูป/Video ตอนพัก Event) จะไปแข่ง
Airtime กับ Voice Call ของ Staff จนคุณภาพเสียงตกฮวบ — CAC ทำให้ระบบรู้ล่วงหน้าว่า "ตอนนี้มี Call
กี่สายอยู่ ใช้ Airtime ไปเท่าไหร่แล้ว" และปฏิเสธ Call ใหม่ก่อนที่จะกระทบ Call เดิม ในขณะที่ WMM
(Step 885) ยังคงทำหน้าที่จัด Priority ของ Call ที่ **ผ่าน CAC มาแล้ว** ให้เหนือกว่า Data Traffic
เสมอ — CAC และ WMM จึงเป็นกลไกที่ทำงาน **คนละจังหวะแต่เสริมกัน**: CAC ตัดสินใจตอน "ขอเข้าใช้"
(Admission) ส่วน WMM ตัดสินใจตอน "กำลังใช้งานแข่งกับ Traffic อื่น" (Scheduling)

---

## Step 888 — Temporary High-Density Events

### สถานการณ์: พื้นที่ปกติความหนาแน่นต่ำ ต้องรองรับ Event ระยะสั้นความหนาแน่นสูง

ปัญหาที่พบบ่อยในทางปฏิบัติมากกว่า Auditorium ที่ Design มาเพื่อ High-Density ตั้งแต่แรกคือ
**พื้นที่ที่ปกติมี Wi-Fi แบบ Coverage-only ธรรมดา** (เช่น Cafeteria, Lobby, ลาน Atrium) แต่ต้อง
รองรับ Event ระยะสั้น (All-hands Meeting, Town Hall, สัมมนาประจำปี) ที่มีคนเข้าร่วมมากกว่าปกติ
หลายเท่าตัวเพียงไม่กี่ชั่วโมง — การ Design พื้นที่นั้นให้เป็น High-Density แบบ Permanent (เพิ่ม
AP ถาวรตาม Step 882-884) **ไม่คุ้มค่า** เพราะ 99% ของเวลา พื้นที่นั้นไม่ต้องการ Capacity ระดับนั้น

### กลยุทธ์ Rapid Temporary Capacity

| กลยุทธ์ | รายละเอียด |
|---|---|
| **Temporary/Portable AP** | AP สำรอง (มักเป็นรุ่นเดียวกับที่ใช้ประจำ) ติดตั้งบน Tripod/Cart เคลื่อนที่ได้ นำมาวางเสริมเฉพาะช่วง Event แล้วเก็บกลับหลังงานจบ — ต้องเตรียม PoE Switch Port และสาย LAN สำรองไว้ล่วงหน้าสำหรับจุดที่คาดว่าจะต้องใช้ |
| **Pre-staged RF Profile + Site/Policy Tag** | สร้าง RF Profile/Site Tag แบบ `RF-HIGH-DENSITY` (ทวนจาก Step 358/882) ที่ **เตรียมไว้ล่วงหน้า** ก่อน Event — วันงานแค่เปลี่ยน Tag ของ AP ที่เกี่ยวข้อง (Existing + Temporary) ให้ชี้ไปที่ Tag นี้ ไม่ต้องสร้าง Config ใหม่หน้างาน |
| **Dedicated Event SSID/VLAN** | สร้าง SSID ชั่วคราวแยกจาก SSID พนักงานปกติ ผูก VLAN/DHCP Scope ของตัวเอง — แยก Traffic ของผู้เข้าร่วม Event ออกจาก Corporate Network หลัก ง่ายต่อการ Monitor/จำกัด Bandwidth และลบทิ้งหลัง Event จบโดยไม่กระทบ SSID ปกติ |
| **Manual RRM Re-tune ทันที** | สั่ง DCA/TPC ให้รันทันที ไม่รอ Interval ปกติ (10 นาที - 1 ชั่วโมงตาม Step 353) เพราะ Event สั้นเกินกว่าจะรอ RRM คำนวณตามรอบปกติ |

```
WLC-1# configure terminal

! สร้าง Site Tag ชั่วคราวสำหรับ Event ผูกกับ RF Profile High-Density ที่เตรียมไว้แล้ว
WLC-1(config)# wireless tag site SITE-EVENT-TEMP
WLC-1(config-site-tag)#  exit

WLC-1(config)# wireless tag rf RF-EVENT-TEMP
WLC-1(config-rf-tag)#  5ghz-rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-tag)#  24ghz-rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-tag)#  exit

! Apply Tag ให้ AP ที่เกี่ยวข้อง (ทั้ง AP ถาวรในพื้นที่ และ AP เสริมชั่วคราว)
WLC-1(config)# ap name AP-CAFETERIA-01
WLC-1(config-ap-tag)#  rf-tag RF-EVENT-TEMP
WLC-1(config-ap-tag)#  site-tag SITE-EVENT-TEMP
WLC-1(config-ap-tag)#  exit
WLC-1(config)# end

! สั่ง DCA รันทันที ไม่รอ Interval — บังคับคำนวณ Channel Plan ใหม่ตอนนี้เลย
WLC-1# wireless config dca-run 5ghz
```

### Capacity Planning จากจำนวนหัวที่คาดการณ์ไว้

ใช้สูตร Capacity เดียวกันจาก Part 72 Step 718 (`max(Coverage, Capacity)`) แต่ปรับตัวเลข
Concurrent User ให้ตรงกับ Event จริง และ**ต้องคูณ Device-per-person** เพราะ Event สมัยใหม่
ผู้เข้าร่วมมักมีมากกว่า 1 Device (Laptop + Phone):

```
ตัวอย่าง: Cafeteria ปกติออกแบบไว้สำหรับ 60 คนเวลาปกติ (2-3 AP, Client/AP ~20-30)
Event All-hands คาดว่าจะมีผู้เข้าร่วม 300 คน พร้อม Device เฉลี่ย 1.5 เครื่อง/คน = 450 Device

ถ้าใช้ Design Target ที่ 30 Client/AP (High-Density ระดับ Conference ทั่วไปตาม Step 881):
   AP ที่ต้องการ ≈ 450 / 30 = 15 ตัว (Coverage AP ปกติมีแค่ 2-3 ตัว — ต้องเสริมอย่างน้อย 12-13 ตัว)

สรุป: ต้องเพิ่ม Temporary AP อย่างน้อย 12 ตัวเข้าไปในพื้นที่ Cafeteria เฉพาะช่วง Event
      (คำนวณล่วงหน้าจากจำนวนหัวที่ผู้จัดงานให้มา ไม่ใช่ประมาณเอาหน้างาน)
```

### แผน Teardown หลัง Event จบ

```
1. ย้าย AP ชั่วคราวกลับไปเก็บ (Physical Removal)
2. Apply Tag เดิม (Site Tag/RF Tag/Policy Tag ปกติ) กลับให้ AP ถาวรในพื้นที่
3. ปิด/ลบ SSID ชั่วคราวและ DHCP Scope ของ Event (ป้องกัน SSID เก่าค้างเป็น Attack Surface)
4. สั่ง DCA/TPC รันใหม่อีกครั้งเพื่อคำนวณ Channel Plan กลับสู่สภาพปกติของพื้นที่
```

> **บทเรียนจาก Real-world Deployment**: องค์กรที่จัด Event บ่อย (Town Hall รายเดือน/รายไตรมาส)
> ควร**เก็บ Config ของ RF/Policy/Site Tag ชั่วคราวไว้ถาวรใน WLC** (ไม่ลบทิ้งทุกครั้ง) แค่ไม่
> Apply ให้ AP เมื่อไม่มี Event — ทำให้ครั้งต่อไปแค่ "สลับ Tag" ได้ทันทีโดยไม่ต้องสร้าง Config
> ใหม่ซ้ำทุกรอบ ลดความเสี่ยงเรื่อง Human Error ตอนเร่งเตรียมงานกระชั้นชิด

---

## Step 889 — Monitoring และ Validating High-Density Performance

### คำสั่งตรวจสอบพื้นฐานบน WLC โดยตรง

```
! ดูจำนวน Client รวมและต่อ AP
WLC-1# show wireless client summary
WLC-1# show ap name AP-AUD-01 config general | include Client

! ดู Channel Utilization ต่อ AP/Radio — ตัวเลขสำคัญที่สุดสำหรับ High-Density Health
WLC-1# show ap dot11 5ghz load-balancing
WLC-1# show ap name AP-AUD-01 dot11 5ghz load

! ดูสถานะ Load-Balancing/ATF ที่กำลังทำงานจริง
WLC-1# show ap name AP-AUD-01 dot11 5ghz atf
```

| Metric | ค่าที่ควรระมัดระวัง | ความหมาย |
|---|---|---|
| Channel Utilization | > 50% ต่อเนื่องนานหลายนาที | เสี่ยง Congestion — Client ต้องรอ Contention นานขึ้น Throughput ต่อ Client จะตกลง |
| Client Count ต่อ AP | เข้าใกล้ `max-clients` ที่ตั้งไว้ (Step 883) | AP กำลังจะเริ่ม Reject Client ใหม่ — สัญญาณว่าต้องเพิ่ม AP ในพื้นที่นี้ |
| Retry Rate | > 10-15% | RF Environment มีปัญหา (Interference/CCI สูง) — Client ต้องส่ง Frame ซ้ำบ่อย กิน Airtime เพิ่ม |
| RF Quality Index (จาก Assurance) | ต่ำกว่าเกณฑ์ที่ตั้ง (ทั่วไป < 7/10) | มีปัญหา SNR/RSSI ที่ Client จำนวนมากรับรู้พร้อมกัน |

### Catalyst Center (DNAC) Assurance สำหรับ High-Density โดยเฉพาะ

ทวนจาก [Part 50 Step 495](part-050-dna-center.md) เรื่อง Assurance — สำหรับพื้นที่ High-Density
ต้องดูมิติที่ Office ทั่วไปไม่ค่อยต้องสนใจ:

| Dashboard/View | ใช้ตรวจอะไรใน High-Density |
|---|---|
| **Client Health (360° View)** | Client Count Trend ต่อ AP รายชั่วโมง/รายวัน — เห็นภาพว่า Peak Time ของพื้นที่นั้นสูงเกิน Design Threshold บ่อยแค่ไหน |
| **Network Health — Wireless** | Channel Utilization Heatmap ต่อ Floor/Area — ระบุ AP ที่ Congestion บ่อยเป็นประจำ (ไม่ใช่แค่ตอน Event) |
| **AI-Driven Issues** | DNAC ตรวจจับ Pattern อัตโนมัติ เช่น "AP Overloaded", "High Client Count", "RF Coverage Hole" แจ้งเตือนก่อนที่ผู้ใช้จะบ่นเข้ามา |
| **Path Trace** | ใช้ตรวจสอบ Client ตัวใดตัวหนึ่งที่ร้องเรียนปัญหาเฉพาะเจาะจงว่า Traffic วิ่งผ่าน Path ไหนและติดขัดที่จุดใด |

### Proactive Capacity Planning จาก Trend Data

หลักการสำคัญของ CCIE-level Operation คือ**ไม่รอให้ User บ่นก่อนแล้วค่อยแก้** — ใช้ข้อมูล Trend
จาก Assurance ย้อนหลัง (เช่น 90 วัน) เพื่อหาพื้นที่ที่ **Channel Utilization/Client Count กำลัง
ไต่ระดับเข้าใกล้ Threshold Design เดิมอย่างสม่ำเสมอ** (ไม่ใช่แค่ Spike ช่วง Event ใดวันหนึ่ง)
แล้ววางแผน**เพิ่ม AP ถาวรหรือปรับ RF Profile ล่วงหน้า** ก่อนที่ปัญหาจะกลายเป็น Incident จริง —
นี่คือความต่างระหว่าง Operation แบบ Reactive (แก้ตอนมีคนร้องเรียน) กับ Proactive (แก้ก่อนมีคน
สังเกตเห็นปัญหา) ซึ่งเป็นกรอบคิดเดียวกับที่ Part 50 สอนไว้สำหรับ Network Health โดยรวม เพียงแต่
Part นี้โฟกัสเฉพาะมิติของ Wireless Capacity

---

## Step 890 — Lab เต็มรูปแบบ: High-Density Design Document สำหรับ Auditorium ของ Building 5

> **หมายเหตุขอบเขต**: เนื่องจากการทดสอบ Client จริงระดับหลายร้อยเครื่องพร้อมกันทำไม่ได้ในรูปแบบ
> Text-based Lab ของหลักสูตรนี้ Step นี้จึงเป็น **Design-level Lab เต็มรูปแบบ** — ผลิตเอกสาร
> ออกแบบจริงที่ใช้ยื่นตรวจสอบได้ (แบบเดียวกับ Part 72 Step 720) ครอบคลุมทุกกลไกที่เรียนไปใน
> Step 881-889 ประยุกต์ใช้กับ **Auditorium/Conference Center ของ Building 5** ในแคมปัส 10 อาคาร
> จาก [Part 72](part-072-large-campus-fabric-design.md) — สมมติว่า Building 5 มีห้อง Auditorium
> ขนาด 800 ที่นั่งอยู่ที่ชั้น 1 (ใหญ่ที่สุดในแคมปัสทั้ง 10 อาคาร ใช้จัด Town Hall/Training ทั้ง
> องค์กร) พื้นที่ 600 ตร.ม. — สมมติเพิ่มนี้ไม่กระทบ `00-ip-address-plan.md` หรือ Config ที่มีอยู่
> ของ Part 72 แต่อย่างใด (Building 5 = 10.50.0.0/16 ตามที่ Part 72 กำหนดไว้แล้ว)

### เอกสารออกแบบ: High-Density Wi-Fi Design — Building 5 Auditorium

**เวอร์ชัน**: 1.0 | **สถานะ**: Draft for Review | **ขอบเขต**: RF/Capacity/QoS Design เฉพาะพื้นที่
Auditorium (ไม่ครอบคลุมชั้นทำงานทั่วไปของ Building 5 ที่ใช้ Design แบบ Office ปกติ)

#### 1. Executive Summary

Auditorium ของ Building 5 (800 ที่นั่ง, 600 ตร.ม.) เป็นพื้นที่ High-Density แห่งเดียวใน Campus
10 อาคารที่ต้องรองรับ Client หนาแน่นระดับ Full-occupancy เป็นประจำ (Town Hall ทุกไตรมาส,
Training เต็มรูปแบบรายเดือน) — เอกสารนี้กำหนด AP Placement, Channel/Power Plan, Band-Steering,
CAC และแผน Monitoring เฉพาะสำหรับพื้นที่นี้ ตาม Checklist Step 881 ทั้ง 14 รายการ

#### 2. Requirement Summary

| ข้อกำหนด | รายละเอียด |
|---|---|
| ความจุที่นั่ง | 800 ที่นั่ง |
| Concurrent Device คาดการณ์ | 800 คน × 1.5 Device/คน ≈ **1,200 Device** (Full-occupancy) |
| พื้นที่ | 600 ตร.ม. |
| WLC ที่ดูแล | WLC Group B (Active ที่ Building 5, Standby ที่ Building 6 — ตาม Part 72 Step 718) |
| Address Block | 10.50.0.0/16 (ตาม Part 72 Section 4) |
| SSID ที่ใช้งาน | `ENTERPRISE-WIFI-PROFILE` (Data/Staff), `EVENT-GUEST` (ผู้เข้าร่วมภายนอกช่วง Event เฉพาะกิจ) |

#### 3. AP Count/Placement Plan

```
คำนวณตามสูตร max(Coverage, Capacity) จาก Part 72 Step 718:

Coverage-only: 600 ตร.ม. / AP ครอบคลุม ~150 ตร.ม./ตัว (Cell เล็กแบบ High-Density) ≈ 4 ตัว
Capacity:      1,200 Device / 30 Client-Device ต่อ AP (Design Target Step 881) ≈ 40 ตัว

ค่าที่ใช้จริง = max(4, 40) = 40 ตัว  ← Capacity เป็นตัวกำหนดอย่างชัดเจน (คนละเรื่องกับ Coverage เดิม)
```

| โซน | จำนวน AP | Antenna Type | เหตุผล |
|---|---|---|---|
| Section A (ที่นั่งด้านหน้า, ใกล้เวที) | 10 ตัว | Sector/Directional (60-90° ตาม Step 884) | ที่นั่งหนาแน่น ต้องแบ่ง Sector ชัดเจนไม่ให้ Cell ทับกัน |
| Section B (ที่นั่งกลาง) | 14 ตัว | Sector/Directional | โซนที่นั่งมากที่สุด ต้องการ AP มากที่สุด |
| Section C (ที่นั่งด้านหลัง/Balcony) | 10 ตัว | Sector/Directional, Downtilt | เพดานสูงกว่าโซนอื่น ต้อง Downtilt ยิงลง |
| Lobby/Foyer หน้า Auditorium | 6 ตัว | Omni-directional | พื้นที่เปิดโล่ง คนเดินผ่าน ไม่ใช่ที่นั่งประจำ — ใช้ Design แบบ Coverage ปกติ |
| **รวม** | **40 ตัว** | Catalyst 9130AXE (External Antenna) ทุกตัว | รองรับ 6GHz (Wi-Fi 6E) ตาม Step 886 ด้วย |

```
Layout แนวคิด (Bird's-eye View, ไม่ตามสัดส่วนจริง):

┌─────────────────────────── Auditorium (600 ตร.ม.) ───────────────────────────┐
│                                    [เวที]                                     │
│   ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼    Section A (หน้า, 10 AP, Sector แคบ)            │
│   ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼                                                    │
│  ▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼  Section B (กลาง, 14 AP, ที่นั่งหนาแน่นสุด)      │
│  ▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼▼AP▼                                                  │
│   ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼    Section C (หลัง/Balcony, 10 AP, Downtilt)      │
│   ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼ ▼AP▼                                                    │
└────────────────────────────────────────────────────────────────────────────┘
   ⬤⬤⬤⬤⬤⬤ (Lobby/Foyer, 6 Omni AP กระจายทั่วพื้นที่เดินผ่าน)
```

#### 4. Channel/Power Plan

| Band | RF Profile | Channel Width | Power Range | หมายเหตุ |
|---|---|---|---|---|
| 2.4GHz | `RF-HIGH-DENSITY` (Step 882) | 20MHz (Fix) | 1-5 dBm | ใช้เป็น Fallback สำหรับ Client เก่าเท่านั้น — Band Select ผลักไป 5/6GHz ก่อนเสมอ |
| 5GHz | `RF-HIGH-DENSITY` (Step 882) | 20MHz (Fix) | 7-11 dBm | Band หลักสำหรับ Client ส่วนใหญ่ในปัจจุบัน |
| 6GHz | `RF-HIGH-DENSITY-6G` (ใหม่, PSC-first DCA) | 20MHz (Fix) | 5-9 dBm | Band ที่มี Channel เหลือมากที่สุด — Client รุ่นใหม่ (Wi-Fi 6E) ถูกผลักมาที่นี่ก่อนเป็นอันดับแรก |

```
WLC-1# configure terminal
WLC-1(config)# ap dot11 6ghz rf-profile RF-HIGH-DENSITY-6G
WLC-1(config-rf-profile)#  tx-power-min 5
WLC-1(config-rf-profile)#  tx-power-max 9
WLC-1(config-rf-profile)#  channel-width 20
WLC-1(config-rf-profile)#  no shutdown
WLC-1(config-rf-profile)#  exit

! บังคับ DCA ให้เลือก PSC (Preferred Scanning Channel) ก่อนเสมอตาม Step 886
WLC-1(config)# ap dot11 6ghz rrm channel dca psc-enforce
WLC-1(config)# end
WLC-1# wireless config dca-run 6ghz
```

#### 5. Band-Steering + CAC Configuration

```
! Band Select (Step 882) — ผลักดัน Client ทุกตัวออกจาก 2.4GHz ก่อนเป็นอันดับแรก
WLC-1(config)# ap dot11 24ghz band-select
WLC-1(config)# ap dot11 24ghz band-select cycle-count 3
WLC-1(config)# ap dot11 24ghz band-select client-rssi -80

! Client Limit ต่อ AP (Step 883) — ยึดตาม Design Target 30 Client-Device/AP
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)#  client association limit 30
WLC-1(config-wireless-policy)#  exit

! Airtime Fairness (Step 885) — แยก Weight ระหว่าง Staff SSID กับ Event Guest SSID
WLC-1(config)# wireless profile atf-policy ATF-STAFF
WLC-1(config-atf-policy)#  atf-weight 70
WLC-1(config-atf-policy)#  no shutdown
WLC-1(config-atf-policy)#  exit
WLC-1(config)# wireless profile atf-policy ATF-EVENT-GUEST
WLC-1(config-atf-policy)#  atf-weight 30
WLC-1(config-atf-policy)#  no shutdown
WLC-1(config-atf-policy)#  exit

! CAC สำหรับ Voice/Video (Step 887) — สำคัญเพราะ Staff ใช้ VoWLAN Phone ระหว่าง Event
WLC-1(config)# ap dot11 5ghz cac voice acm
WLC-1(config)# ap dot11 5ghz cac voice max-bandwidth 75
WLC-1(config)# ap dot11 5ghz cac voice roam-bandwidth 6
WLC-1(config)# ap dot11 5ghz cac video acm
WLC-1(config)# ap dot11 5ghz cac video max-bandwidth 40
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

#### 6. Capacity-Monitoring Dashboard Concept

แนวคิด Dashboard ที่ทีม NOC ควรตั้งไว้ใน Catalyst Center Assurance (ทวนจาก Step 889) เฉพาะสำหรับ
พื้นที่นี้:

```
┌──────────────────────── Building 5 Auditorium — High-Density Health ────────────────────────┐
│                                                                                                 │
│  [Client Count Now]        [Channel Utilization]         [RF Quality Index]                  │
│      847 / 1200                5GHz: 42%  🟢               8.6 / 10  🟢                        │
│      (Design Max)              2.4GHz: 8%  🟢                                                  │
│                                 6GHz: 21%  🟢                                                   │
│                                                                                                 │
│  [Client Distribution by Band]           [Active Voice/Video CAC Sessions]                    │
│   6GHz  ████████░░ 38%                    Voice:  6 / 12 max (50%) 🟢                          │
│   5GHz  ██████████████░ 55%               Video:  1 / 4 max  (25%) 🟢                          │
│   2.4GHz ██░ 7%                                                                                │
│                                                                                                 │
│  [Client Count Trend — Last 90 Days, Event Days Highlighted]                                  │
│   1200 ┤                          ▄█▄                              ▄█▄                        │
│    900 ┤                     ▄▄▄▄███▄▄▄▄                      ▄▄▄▄███▄▄▄▄                     │
│    600 ┤    ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄████████████▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄████████████▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄     │
│    300 ┤                                                                                       │
│      0 └───────────────────────────────────────────────────────────────────────────────────    │
│         สัปดาห์ปกติ (ที่นั่ง+เดินผ่าน ~200-300)    Town Hall (Peak ~1,150)    Training (Peak ~980) │
│                                                                                                 │
│  [AI-Driven Issues ที่เกี่ยวกับพื้นที่นี้]                                                        │
│   - ไม่พบ Issue ("AP Overloaded"/"Coverage Hole") ในช่วง 90 วันที่ผ่านมา ✅                       │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

| Widget | Threshold ที่ตั้ง Alert | เหตุผล |
|---|---|---|
| Client Count Now | > 90% ของ Design Max (1,200) | เตือนล่วงหน้าก่อนถึง `max-clients` Hard Limit จริงของ Section ใดก็ตาม |
| Channel Utilization | > 60% ต่อเนื่อง 15 นาที (ทุก Band) | สัญญาณ Congestion เริ่มกระทบ Experience |
| RF Quality Index | < 7/10 | มีปัญหา SNR/RSSI ที่กระทบ Client จำนวนมากพร้อมกัน |
| Voice/Video CAC Utilization | > 80% ของ max-bandwidth | ใกล้จุดที่ WLC จะเริ่ม Reject Call ใหม่ — แจ้งทีม Event ล่วงหน้าถ้ามีแผนเพิ่ม VoWLAN Phone |
| Client Count Trend (90 วัน) | Peak เพิ่มขึ้นต่อเนื่องทุก Event รอบถัดไป | สัญญาณ Proactive Capacity Planning (Step 889) — ถ้า Trend ไต่ระดับเข้าใกล้ 1,200 ทุกครั้ง ต้องพิจารณาเพิ่ม AP ถาวรก่อนถึง Full Saturation |

#### 7. Verification Checklist ก่อน Go-live

| # | รายการ | คำสั่งตรวจสอบ |
|---|---|---|
| 1 | AP ทุกตัว Join สำเร็จและอยู่ Site Tag ที่ถูกต้อง | `show ap summary` |
| 2 | RF Profile 3 Band ถูก Apply ตรงตาม Design | `show ap dot11 5ghz summary`, `show ap dot11 6ghz summary` |
| 3 | Band Select ทำงาน (Client กระจายไป 5/6GHz จริง) | `show wireless client summary` (ดู Band ต่อ Client) |
| 4 | Client Limit และ ATF Policy ผูกถูก Policy Profile | `show wireless profile policy detailed ENTERPRISE-WIFI-POLICY-FLEX` |
| 5 | CAC เปิดและมี Bandwidth เหลือเพียงพอสำหรับ VoWLAN Staff | `show ap dot11 5ghz cac voice summary` |
| 6 | Predictive RF Survey (Ekahau/iBwave) ตรงกับผลวัดจริงหลัง Deploy (Validation Survey) | เครื่องมือ Survey ภายนอก WLC — เดินวัด RSSI/SNR จริงเทียบกับแบบจำลอง |

> **ทำไมต้องมีทั้ง Predictive และ Validation Survey**: แบบจำลอง (Predictive) ใช้ประเมิน AP Count/
> Placement ก่อน Deploy จริงบนกระดาษ/Software — แต่วัสดุจริงของ Auditorium (เก้าอี้บุนวม, ผู้ชม
> เต็มห้องที่ดูดซับสัญญาณ, โครงสร้างเวที) ทำให้ค่าจริงต่างจากแบบจำลองได้เสมอ **Validation Survey**
> (วัดด้วยเครื่องมือจริงหลัง AP ติดตั้งครบ โดยเฉพาะช่วงที่มีคนเต็มห้องจำลองสถานการณ์จริง) จึงเป็น
> ขั้นตอนบังคับก่อน Go-live เสมอ ไม่ใช่ทำแค่แบบจำลองแล้วถือว่าจบ

---

## แบบฝึกหัดทวนความเข้าใจ Part 89

1. เพราะเหตุใดการ "เพิ่ม AP" ในพื้นที่ High-Density โดยไม่ลด Cell Size/Power ควบคู่กันจึงไม่ทำให้
   Capacity รวมของพื้นที่เพิ่มขึ้นตามสัดส่วน?

2. Airtime Fairness (ATF) และ WMM (802.11e) ทำงานแก้ปัญหาคนละมิติกันอย่างไร เพราะเหตุใดพื้นที่
   High-Density ที่ดีต้องเปิดทั้งสองกลไกพร้อมกัน ไม่ใช่เลือกอย่างใดอย่างหนึ่ง?

3. CAC (Call Admission Control) ต่างจาก WMM Priority (AC_VO) อย่างไร และเพราะเหตุใดการมี WMM
   Priority เพียงอย่างเดียวไม่เพียงพอสำหรับป้องกันปัญหาคุณภาพเสียงใน High-Density Environment?

4. เพราะเหตุใด Antenna แบบ Sectorized/Directional จึงเหมาะกับ Stadium Bowl/Auditorium มากกว่า
   Omni-directional Antenna ทั้งที่ Omni มักใช้เป็น Default ของ AP ทั่วไป?

5. 6GHz (Wi-Fi 6E) ช่วยแก้ปัญหา High-Density ได้อย่างไร และเพราะเหตุใด Range ที่สั้นลงของ 6GHz
   จึงไม่ใช่ข้อเสียร้ายแรงสำหรับ Design ประเภทนี้โดยเฉพาะ (ต่างจาก Coverage Design ทั่วไป)?

### เฉลย

1. เพราะจำนวน Non-overlapping Channel มีจำกัด (2.4GHz มี 3 Channel, 5GHz มีประมาณ 21-25
   Channel) — ถ้าเพิ่ม AP โดย Cell Size เท่าเดิม AP ที่เพิ่มมาต้องใช้ Channel ซ้ำกับ AP ที่มีอยู่
   แล้วในระยะใกล้ ทำให้เกิด Co-Channel Interference (CCI) สูงขึ้น AP ที่ Channel เดียวกันต้อง
   "ผลัดกันพูด" ผ่าน CSMA/CA แทนที่จะทำงานพร้อมกันได้อิสระ — Throughput รวมของพื้นที่จึงไม่เพิ่ม
   ตามสัดส่วน AP ที่เพิ่มขึ้น การเพิ่ม Capacity จริงต้องลด Power/Cell Size ควบคู่กันเพื่อให้ AP
   ที่เพิ่มมาสามารถ Reuse Channel ในระยะที่ไม่รบกวนกันได้จริง

2. ATF แบ่ง **Airtime ระหว่างกลุ่ม Client/SSID** (แนวนอน) เพื่อป้องกัน SSID/Client Group หนึ่ง
   ครองทั้งหมด (เช่น ป้องกัน Client ช้าตัวเดียวกิน Airtime มากกว่า Client เร็วหลายตัวรวมกัน) ส่วน
   WMM แบ่ง **ลำดับความสำคัญของประเภท Traffic ภายในกลุ่มเดียวกัน** (แนวตั้ง) เพื่อให้ Voice/Video
   ได้ Priority เหนือ Data — ทั้งสองแก้ปัญหาคนละแกน พื้นที่ High-Density ที่มี Client จำนวนมาก
   และ Traffic หลากหลายประเภทต้องเปิดทั้งสองอย่างพร้อมกันจึงจะครอบคลุมทั้งปัญหา "แบ่งทรัพยากร
   ระหว่างกลุ่ม" และ "จัดลำดับภายในกลุ่ม" ได้สมบูรณ์

3. WMM Priority (AC_VO) กำหนดแค่ว่า **เมื่อ Traffic แข่งกัน Voice ต้องได้ไปก่อน** แต่ไม่ได้จำกัด
   **จำนวน Call ที่เกิดขึ้นพร้อมกัน** — ถ้ามี Call จำนวนมากเกิน Capacity ของ Channel (แม้ Voice
   จะได้ Priority สูงสุดเสมอ) Airtime รวมที่ Voice ทั้งหมดต้องการก็ยังเกิน Channel รองรับได้ ทำให้
   ทุก Call ที่มีอยู่แล้วเริ่มเสื่อมคุณภาพพร้อมกันหมด (Jitter/Loss สูงขึ้นทุก Call) — CAC แก้ปัญหา
   นี้โดยปฏิเสธ Call ใหม่ที่จะทำให้เกิน Threshold (max-bandwidth) ตั้งแต่ขั้น Admission ก่อนที่
   Call จะเริ่มเลย รักษาคุณภาพของ Call ที่มีอยู่แล้วไว้ให้ดีเสมอ

4. เพราะ Stadium Bowl/Auditorium มีที่นั่งเป็นแนวแบ่ง Section ชัดเจน — Antenna Omni กระจาย
   พลังงาน RF รอบตัว 360° เท่ากันทุกทิศทาง รวมถึงทิศทางที่ไม่มีคนนั่งอยู่ (เช่น ขึ้นไปในอากาศ
   เหนือหัวคนดู หรือล้นไปยัง Section ข้างเคียงที่มี AP ของตัวเองอยู่แล้ว) ทำให้เกิด CCI ระหว่าง
   Section โดยไม่จำเป็น ส่วน Antenna แบบ Sectorized/Directional โฟกัสพลังงานให้ตกอยู่เฉพาะพื้นที่
   ที่นั่งจริงของ Section ตัวเองเท่านั้น ทำให้ Reuse Channel ระหว่าง Section ได้ถี่ขึ้นโดยไม่เพิ่ม
   CCI — ตรงกับหลักการ Cell Size Reduction ตามรูปทรงพื้นที่จริงของ High-Density Design

5. 6GHz เพิ่ม Non-overlapping Channel ให้มากถึงราว 59 Channel (20MHz) ซึ่งมากกว่า 2.4GHz+5GHz
   รวมกันหลายเท่า และเป็น Spectrum ใหม่ที่ไม่มี Legacy Device เก่าค้างอยู่เลย ทำให้ปัญหาหลักของ
   High-Density (Channel ไม่พอต้อง Reuse จนเกิด CCI) บรรเทาลงมาก — ส่วน Range ที่สั้นลง (จาก
   Free Space Path Loss ที่สูงขึ้นตามความถี่) ไม่ใช่ข้อเสียร้ายแรงสำหรับ High-Density เพราะ
   Design ประเภทนี้ต้องการ Cell เล็กอยู่แล้วเป็นปกติ (Step 882) — 6GHz แค่ทำให้สิ่งที่ต้องทำอยู่
   แล้ว (ลด Cell Size, เพิ่ม AP Density) ได้ผลดีขึ้นโดยแถม Channel ที่เพียงพอมาด้วย ต่างจาก
   Coverage Design ทั่วไปที่ Range สั้นลงจะเป็นข้อเสียตรงๆ เพราะต้องการ Cell ใหญ่ครอบคลุมพื้นที่
   กว้างด้วย AP น้อยตัว

---

## สรุป Part 89

Part นี้ต่อยอด Wireless Architecture จาก Part 36 เข้าสู่ **High-Density Design** อย่างเต็มรูปแบบ:
ปัญหาพื้นฐานที่ทำให้ "เพิ่ม AP" ไม่ Scale เป็นเส้นตรง (Step 881), RF Planning ที่กลับหลักการ
Coverage Design ธรรมดา (Cell Size Reduction, 5GHz-Preference, Step 882), Client Capacity
Management (`max-clients`, Load-Balancing, Band-Select, Step 883), Hardware/Antenna Selection
ตามรูปทรงพื้นที่ (Step 884), Airtime Fairness ที่เชื่อมกับ WMM/QoS จาก Part 24/40/82 (Step 885),
6GHz/Wi-Fi 6E ในฐานะทางออกเชิง Spectrum (Step 886), CAC สำหรับปกป้อง Voice/Video Capacity
(Step 887), กลยุทธ์ Temporary Event Capacity (Step 888), Monitoring ผ่าน Assurance ที่โฟกัส
มิติ High-Density โดยเฉพาะ (Step 889) และปิดท้ายด้วยเอกสารออกแบบเต็มรูปแบบสำหรับ Auditorium
800 ที่นั่งของ Building 5 (Step 890) ที่ประยุกต์ใช้ทุกกลไกเข้าด้วยกันจริง

✅ **พร้อมสำหรับ Part 90**: เราออกแบบ Wireless ระดับ High-Density เสร็จสมบูรณ์แล้ว Part 90 จะ
เปลี่ยนโฟกัสกลับไปที่ Hardware Platform — **Catalyst 9000 Advanced Features**: ความสามารถเชิงลึก
ของ Catalyst 9000 Series (StackWise Virtual, SDA-ready ASIC, UADP) ที่รองรับทั้ง Wired Campus
และ Wireless (ผ่าน Embedded Wireless Controller บน Catalyst 9800) ที่เราออกแบบไปตลอด Part นี้

**ไปต่อ:** [Part 90 — Catalyst 9000 Advanced Features →](part-090-catalyst-9000-advanced-features.md)
