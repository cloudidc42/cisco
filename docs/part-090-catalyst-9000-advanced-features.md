# Part 90 — Catalyst 9000 Series Advanced Features
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 891–900 จาก 1000**

> ต่อจาก [Part 89 — Enterprise Wireless High-Density Design](part-089-enterprise-wireless-high-density.md)
> ที่เจาะลึกการออกแบบ Wireless สำหรับพื้นที่ความหนาแน่นสูง (Stadium/Auditorium-class Design,
> Co-Channel Interference, Client Load Balancing) ให้กับ WLC-1 และ AP Fleet ของหลักสูตรนี้แล้ว
> Part นี้จะ **ถอยกลับมาที่ Hardware** — ตลอด 89 Part ที่ผ่านมา เราสั่ง config บน `CORE-SW1`,
> `CORE-SW2`, `DIST-SW1–4`, `ACCESS-SW1–4`, และ `WLC-1` นับพันคำสั่งโดยไม่เคยหยุดถามว่า
> **"ทำไม Hardware ตัวนี้ถึงทำสิ่งเหล่านี้ได้"** ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md)
> กำหนดไว้ตั้งแต่ Part 1 ว่า CORE-SW1/CORE-SW2 คือ **Catalyst 9500**, DIST-SW1–4 คือ
> **Catalyst 9300**, ACCESS-SW1–4 คือ **Catalyst 9200/2960**, และ WLC-1 คือ **Catalyst 9800-CL**
> — Part นี้จะเจาะลึกสถาปัตยกรรม **UADP ASIC**, **IOS-XE**, **Application Hosting**,
> **Flexible NetFlow**, **StackWise-1T**, **Perpetual/Fast PoE**, **MACsec**, **Catalyst 9800
> Architecture**, และ **Cisco Licensing Model** ที่ทำให้ทุกอย่างตั้งแต่ Part 1 ถึง Part 89
> เป็นไปได้จริงในระดับ Silicon และ Software ก่อนจะไปสู่ [Part 91 — Advanced Troubleshooting
> Methodology](part-091-advanced-troubleshooting-methodology.md) ที่จะรวมทุกความรู้ Part 1-90
> เข้าเป็นกระบวนการ Troubleshoot แบบมีระบบ

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 891 | UADP ASIC Architecture — ทำไม Catalyst 9000 ต่างจาก Catalyst รุ่นเก่าโดยพื้นฐาน |
| 892 | Cisco IOS-XE Architecture — Linux-based, Containerized OS และความสัมพันธ์กับ NETCONF/RESTCONF |
| 893 | Application Hosting (IOx) บน Catalyst 9000 — รัน Container จริงบน Switch |
| 894 | Flexible NetFlow บน Catalyst 9000 — Hardware-Accelerated Traffic Visibility |
| 895 | StackWise/SVL Deep Dive ฉบับปรับปรุง — StackWise-1T และการออกแบบ Bandwidth ของ SVL Link |
| 896 | Perpetual PoE และ Fast PoE — จ่ายไฟต่อเนื่องแม้ Switch Reload |
| 897 | MACsec (802.1AE) — Hardware-Accelerated Layer 2 Encryption บน CORE-DIST Link |
| 898 | Catalyst 9800 Architecture — ทำไม WLC-1 ใช้ IOS-XE ไม่ใช่ AireOS |
| 899 | Cisco DNA/Catalyst Licensing Model — Essentials, Advantage, Premier และ Smart Licensing |
| 900 | Lab เต็มรูปแบบ: App Hosting + Flexible NetFlow + MACsec + Perpetual PoE ข้ามทุก Platform |

---

## Step 891 — UADP ASIC Architecture: ทำไม Catalyst 9000 ต่างจาก Catalyst รุ่นเก่าโดยพื้นฐาน

### ปัญหาของ Fixed-Function ASIC (Catalyst รุ่นก่อน 9000)

Switch Catalyst รุ่นก่อนหน้า (3750, 3850, 4500, 6500) ใช้ **Fixed-Function ASIC** — ชิปที่ถูก
ออกแบบมาให้ทำงานเฉพาะ Feature Set ที่กำหนดไว้ตั้งแต่วันผลิต **ในระดับ Hardware Logic Gate**
เพิ่ม Feature ใหม่ (เช่น Protocol Encapsulation แบบใหม่) ทำไม่ได้เลยถ้า ASIC ไม่ได้ออกแบบรองรับ
ไว้ล่วงหน้า ทางเดียวคือ **เปลี่ยน Hardware ทั้งตัว** (Forklift Upgrade) — ปัญหานี้ชัดเจนที่สุดตอน
Industry เริ่มต้องการ VXLAN, SD-Access Fabric, และ Micro-segmentation ที่ Switch รุ่นเก่าไม่มีวัน
ทำได้แม้จะ Upgrade Software เท่าไหร่ก็ตาม เพราะ Silicon ไม่มี Logic รองรับ Encapsulation แบบใหม่

### UADP (Unified Access Data Plane) — Programmable/Flexible ASIC

**UADP** คือ ASIC ตระกูลที่ Cisco พัฒนาขึ้นเป็นแกนกลางของ **ทุก Switch ในตระกูล Catalyst 9000**
(9200, 9300, 9400, 9500, 9600) โดยหลักการที่ต่างจาก Fixed-Function โดยสิ้นเชิงคือ **Programmable
Pipeline** — วงจรภายในไม่ได้ถูก "เขียนตายตัว" แต่เป็นชุด **Micro-engine ที่ปรับ Logic ได้ผ่าน
Microcode** ซึ่งอัปเดตได้ผ่าน IOS-XE Software Upgrade ปกติ โดยไม่ต้องเปลี่ยน Hardware เลย

| หัวข้อ | Fixed-Function ASIC (Catalyst รุ่นเก่า) | UADP (Catalyst 9000) |
|---|---|---|
| Logic ภายใน | ตายตัวตั้งแต่วันผลิต (Hard-wired) | Programmable Pipeline ปรับผ่าน Microcode |
| เพิ่ม Feature ใหม่หลัง GA | ทำไม่ได้ ต้องเปลี่ยน Hardware | ทำได้ผ่าน IOS-XE Software Upgrade (ในหลาย Feature) |
| ตัวอย่าง Feature ที่เพิ่มมาทีหลังจาก Software เดียว | ไม่มี — Hardware ต้องรองรับตั้งแต่ต้น | VXLAN Encap/Decap, SD-Access Fabric, MACsec Cipher Suite ใหม่ |
| Recirculation | ไม่มี — Pipeline เดียวจบ | มี **Recirculation Port ภายใน ASIC** — Packet วิ่งผ่าน Pipeline ซ้ำได้เพื่อทำ Encapsulation หลายชั้น |
| Generation ที่ใช้ในหลักสูตรนี้ | - | Catalyst 9300/9500 รุ่น Fixed ใช้ UADP 2.0/3.0 (ตาม SKU) |

### Recirculation — กุญแจของ VXLAN Encap/Decap ใน Hardware

จุดที่ทำให้ UADP รองรับ SD-Access ([Part 38](part-038-sd-access-fundamentals.md),
[Part 77](part-077-sd-access-advanced-design.md)) ได้จริงคือกลไก **Recirculation**:
Packet ที่ต้องถูก **ห่อ VXLAN Header เพิ่มอีกชั้น** (Overlay Encapsulation) จะถูกส่งวนกลับเข้า
Pipeline ของ ASIC เป็นรอบที่สอง **ภายใน Silicon เดียวกัน ไม่ต้อง Punt ขึ้น CPU เลย**:

```
Packet เข้า Port (Underlay, ยังไม่ Encap)
        │
        ▼
┌─────────────────────┐   Pass 1: Lookup Fabric Edge Policy,
│   UADP Pipeline      │   ตัดสินใจว่าต้อง VXLAN Encap
│   (Pass 1)           │
└──────────┬───────────┘
           │ Recirculate (ภายใน ASIC เดียวกัน — ไม่ผ่าน CPU)
           ▼
┌─────────────────────┐   Pass 2: เติม VXLAN Header (UDP 4789)
│   UADP Pipeline      │   + Outer IP/MAC ของ Underlay แล้วส่งออก
│   (Pass 2)           │
└──────────┬───────────┘
           ▼
Packet ออก Port (Overlay, VXLAN Encapsulated) — Wire-speed ทุก Packet
```

ถ้า Switch ใช้ Fixed-Function ASIC รุ่นเก่า การ Encap VXLAN ต้องพึ่ง **Software/CPU** (Punt
ขึ้นไปประมวลผลที่ Route Processor) ซึ่งทำได้ที่ **หลักพัน pps** เท่านั้น — ไม่พอสำหรับ Traffic
ระดับ Enterprise Campus จริง (ที่ต้องการ **หลักล้าน pps ต่อ Port**) นี่คือเหตุผลตรงๆ ว่าทำไม
SD-Access ถึง**ต้อง**ใช้ Catalyst 9000 เท่านั้น ไม่สามารถทำบน Catalyst รุ่นก่อนหน้าได้เลยแม้จะ
Upgrade IOS ทุก Patch ก็ตาม — ข้อจำกัดอยู่ที่ **Silicon** ไม่ใช่ Software

### คำสั่งตรวจสอบ ASIC/Hardware บน Catalyst 9000

```
CORE# show platform
Chassis type: C9500-40X

Slot      Type                State                 Insert time (ago)
--------- ------------------- --------------------- -----------------
1         C9500-40X           ok                    3w2d
2         C9500-40X           ok                    3w2d

CORE# show platform software fed switch active fwd-asic resource tcam utilization
+----------------------------------------------------------------+
|  ASIC# / Table              Max Entries    Used     Free  %Used |
+----------------------------------------------------------------+
|  0 / MAC Address Table         32768        412    32356    1%  |
|  0 / L3 FIB (IPv4)             65536       1284    64252    2%  |
|  0 / Security ACL              4096         188     3908    5%  |
+----------------------------------------------------------------+
```

> **สำคัญ**: `show platform hardware fed switch active fwd-asic resource tcam utilization`
> เป็นคำสั่งที่วิศวกร CCIE Enterprise ใช้บ่อยที่สุดตอน Troubleshoot ปัญหา **"Feature ใหม่ใช้ไม่ได้
> ทั้งที่ Config ถูกต้อง"** — สาเหตุอันดับหนึ่งคือ **TCAM เต็ม** (Space หมดจริง ไม่ใช่ Bug)
> ซึ่งเป็นข้อจำกัดร่วมของ UADP ทุก Generation แม้จะ Programmable ก็ตาม (Programmable ≠ Unlimited
> Memory)

---

## Step 892 — Cisco IOS-XE Architecture: Linux-based, Containerized OS

### IOS Classic (Monolithic) — สถาปัตยกรรมที่หลักสูตรนี้อ้างอิงมาตั้งแต่ Part 1 โดยไม่รู้ตัว

**IOS Classic** (ที่ใช้ใน Catalyst 2960, ISR รุ่นเก่า) เป็น **Monolithic Software** — ทุก Process
(Routing, Switching, CLI, SNMP) รันอยู่ใน **Memory Space เดียวกัน** บน Real-Time OS ตัวเดียว
ไม่มีการแยก Process จริง ผลคือ: (1) Process หนึ่ง Crash มีโอกาสทำให้ **ทั้งระบบ Reload**
(2) ไม่มี Process Isolation ด้าน Memory/Security (3) Patch Feature เดียวต้อง Upgrade
IOS Image ทั้งตัว ไม่มีทาง Patch แค่บางส่วน

### IOS-XE — Linux-based, Modular/Distributed Architecture

**IOS-XE** (ที่ใช้ใน Catalyst 9000 ทุกตัว, ISR4000, ASR1000) เปลี่ยนสถาปัตยกรรมทั้งหมด:
รันบน **Linux Kernel จริง** (Wind River Linux) เป็นฐาน แล้วให้ `IOSd` (IOS Daemon) — ตัว CLI/
Control-Plane ที่วิศวกรคุ้นเคย — ทำงานเป็น **Linux Process ตัวหนึ่ง** ท่ามกลาง Process อื่นๆ
อีกหลายสิบตัวที่รันแยกกันเป็นอิสระ:

```
┌──────────────────────────────────────────────────────────────┐
│                    Linux Kernel (Wind River Linux)             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐ │
│  │   IOSd     │  │    FED     │  │   wcm       │  │  IOx     │ │
│  │ (CLI/      │  │ (Forwarding│  │ (Wireless   │  │ (App     │ │
│  │  Routing   │  │  Engine    │  │  Controller │  │  Hosting │ │
│  │  Protocol) │  │  Driver)   │  │  Manager,   │  │  Container│ │
│  │            │  │            │  │  9800 เท่านั้น)│  │  Runtime)│ │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘ │
│         แต่ละกล่อง = Linux Process/Container แยกกัน            │
│         Crash ตัวหนึ่ง Restart เฉพาะตัวนั้น ไม่ต้อง Reload ทั้งระบบ │
└──────────────────────────────────────────────────────────────┘
```

| หัวข้อ | IOS Classic (Monolithic) | IOS-XE (Linux-based, Modular) |
|---|---|---|
| Kernel พื้นฐาน | ไม่มี Kernel จริง (Run-to-Completion บน RTOS) | Linux Kernel จริง (Wind River Linux) |
| IOSd/CLI | ทั้งระบบคือตัวมันเอง | เป็นแค่ **Process ตัวหนึ่ง** บน Linux |
| Process Isolation | ไม่มี | มี (แต่ละ Daemon แยก Memory Space) |
| Process หนึ่ง Crash | มักทำให้ทั้งระบบ Reload | Restart เฉพาะ Process นั้น (ถ้า Design รองรับ) |
| รัน Container ของบุคคลที่สามได้ไหม | ไม่ได้เลย | ได้ (ผ่าน IOx — ดู Step 893) |
| Patch แบบ Subsystem | ไม่มี | มี (Cisco IOS-XE SMU — Software Maintenance Upgrade แพตช์เฉพาะจุดโดยไม่ต้อง Full Image Upgrade) |

### คำสั่งสำรวจสถาปัตยกรรมจริง

```
CORE# show version | include IOS-XE
Cisco IOS-XE Software, Version 17.09.04a

CORE# show platform software process list | include IOSd|FED|wcm
  1204  IOSd                    S    ppid: 1  status: running
  1301  fed_main                S    ppid: 1  status: running

CORE# show platform software infrastructure inject
CORE# show processes cpu platform sorted | exclude 0.00%   0.00%   0.00%
   PID Runtime(ms)  Invoked   uSecs    5Sec    1Min    5Min TTY Process
  1301      842103   902341     932   3.21%   2.87%   2.75%   0 fed_main
  1204      321044   551200     582   1.10%   0.98%   0.91%   0 IOSd
```

### ทำไม IOS-XE คือเหตุผลที่ NETCONF/RESTCONF ([Part 49](part-049-netconf-restconf-yang-deep-dive.md))
### และ CI/CD Pipeline ([Part 79](part-079-fullstack-automation-cicd.md)) ทำงานได้ดี

[Part 49](part-049-netconf-restconf-yang-deep-dive.md) สอน NETCONF/RESTCONF โดยไม่ได้อธิบาย
ว่า **ทำไม Protocol เหล่านี้ทำงานได้ดีบน Catalyst 9000 โดยเฉพาะ** — คำตอบคือสถาปัตยกรรม IOS-XE
นี่เอง: เพราะ Management Plane (NETCONF/RESTCONF Server, YANG Datastore) รันเป็น **Process
แยกอิสระบน Linux** (ไม่ได้ฝังอยู่ในตัว IOSd โดยตรง) การยิง API เข้ามาจำนวนมาก (เช่น CI/CD
Pipeline ของ Part 79 ที่ Push Config ทั้ง Fleet พร้อมกัน) จึงไม่ไปแย่ง CPU Cycle ของ Routing
Protocol/Forwarding Process โดยตรง — Design แบบ Distributed Process นี้คือ **Foundation**
ที่ทำให้ Model-Driven Programmability เสถียรพอสำหรับ Production Automation ตั้งแต่ต้น
(ต่างจาก IOS Classic ที่การยิง SNMP/API หนักๆ อาจกระทบ CPU Cycle ของ Routing โดยตรง เพราะ
อยู่ใน Process เดียวกันหมด)

---

## Step 893 — Application Hosting (IOx) บน Catalyst 9000

### แนวคิด: รัน Container จริงบน Switch ไม่ใช่แค่ Simulate

**Application Hosting** (Framework ชื่อ **IOx**) คือความสามารถที่ Catalyst 9000 (รุ่นที่มี
Onboard Compute/Storage เพียงพอ เช่น C9300, C9400, C9500 บาง SKU) เปิดให้รัน **Container
จริง** (Docker Image หรือ LXC) ข้างใน Switch เอง โดยแยก CPU/Memory ส่วนหนึ่งออกจากส่วนที่
IOS-XE ใช้ทำ Control/Data Plane — เป็นไปได้เพราะสถาปัตยกรรม Linux-based ของ IOS-XE (Step 892)
ที่มี Container Runtime พร้อมอยู่แล้วในระดับ Kernel

```
┌─────────────────────────────────────────────────────────────┐
│                  Catalyst 9300 (DIST-SW1) Chassis              │
│  ┌───────────────────────────┐   ┌───────────────────────┐   │
│  │   IOS-XE (Control/Data      │   │   IOx Container Runtime │   │
│  │   Plane) — CPU/Memory        │   │   (แยก Resource ชัดเจน) │   │
│  │   ส่วนหลัก                   │   │   ┌─────────────────┐  │   │
│  │                              │   │   │ Monitoring Agent │  │   │
│  │                              │   │   │ Container (เช่น   │  │   │
│  │                              │   │   │ ThousandEyes,     │  │   │
│  │                              │   │   │ Prometheus Node   │  │   │
│  │                              │   │   │ Exporter)         │  │   │
│  └───────────────────────────┘   │   └─────────────────┘  │   │
│                                    └───────────────────────┘   │
│  AppGigabitEthernet1/0/1 (Interface เฉพาะสำหรับ App Traffic)   │
└─────────────────────────────────────────────────────────────┘
```

### Use Case ที่ใช้จริงในองค์กร

| Use Case | อธิบาย | ทำไมต้องรันที่ Switch ไม่ใช่ Server แยก |
|---|---|---|
| **Local Network Monitoring Agent** | รัน Container Synthetic Test/ThousandEyes Enterprise Agent เพื่อวัด Latency/Jitter จาก **มุมมองของ Access Layer จริง** | วัดผลใกล้ End User ที่สุด ไม่ต้องเดินสาย/ใช้ Rack เพิ่ม ไม่มี Extra Hop ที่ทำให้ผลวัดเพี้ยน |
| **Edge Compute IoT Application** | รัน Protocol Gateway (เช่นแปลง Modbus/RS-485 จากเครื่องจักร OT เป็น MQTT ก่อนส่งขึ้น Cloud) | ลด Backhaul Bandwidth, ลด Latency สำหรับ Real-time Control, แยก OT Traffic ออกจาก Core Network ตั้งแต่ Access Layer |
| **Packet Capture as a Service** | รัน `tcpdump`/Container Capture Tool on-demand เพื่อ Troubleshoot โดยไม่ต้อง SPAN ออกไปเครื่องอื่น | ลด Overhead ของการเดิน SPAN Cable ข้าม Layer สำหรับ Case Troubleshoot เร่งด่วน |

### Config เปิด Application Hosting บน DIST-SW1 (Catalyst 9300)

```
DIST-SW1(config)# iox
DIST-SW1(config)# interface AppGigabitEthernet1/0/1
DIST-SW1(config-if)# description ** IOx App-hosting internal interface **
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk allowed vlan 99
DIST-SW1(config-if)# exit

! ติดตั้ง Application (App Package เป็น .tar/.ova ที่ Build มาแล้ว)
DIST-SW1# app-hosting install appid MONITOR-AGENT package flash:monitor-agent.tar
DIST-SW1# app-hosting activate appid MONITOR-AGENT
DIST-SW1# app-hosting start appid MONITOR-AGENT
```

### Verification

```
DIST-SW1# show app-hosting list
App id                           State
---------------------------------------------
MONITOR-AGENT                    RUNNING

DIST-SW1# show app-hosting detail appid MONITOR-AGENT
App id                    : MONITOR-AGENT
Owner                      : iox
State                      : RUNNING
Application
  Type                     : docker
  Name                     : monitor-agent
Resource reservation
  Memory                   : 512 MB
  Disk                     : 100 MB
  CPU                      : 10% (1 core)
Network interface
  eth0
    MAC address            : 52:54:00:aa:bb:cc
    IPv4 address            : 10.10.99.60
    Network name            : Vlan99-App
```

> **ข้อจำกัดสำคัญสำหรับหลักสูตรนี้**: Application Hosting **ไม่รองรับบน ACCESS-SW1–4** ในเคสที่
> เป็น Catalyst 2960 (Fixed-Function, ไม่มี Onboard Compute แยกสำหรับ Container) — รองรับ
> เฉพาะ ACCESS-SW ที่เป็น **Catalyst 9200/9300 รุ่นที่มี DRAM/Flash เพียงพอ** เท่านั้น ในหลักสูตรนี้
> จึงสาธิตบน **DIST-SW1 (Catalyst 9300)** เป็นตัวแทน (ดู Lab เต็มใน Step 900)

---

## Step 894 — Flexible NetFlow บน Catalyst 9000: Hardware-Accelerated Traffic Visibility

### ต่างจาก Model-Driven Telemetry ([Part 54](part-054-network-assurance-telemetry.md)/
### [Part 80](part-080-model-driven-telemetry-streaming.md)) อย่างไร

[Part 54](part-054-network-assurance-telemetry.md) และ [Part 80](part-080-model-driven-telemetry-streaming.md)
สอน **Model-Driven Telemetry (MDT)** — Stream ข้อมูล **สถานะของ Device เอง** (CPU, Memory,
Interface Counter, BGP Neighbor State) ออกมาแบบ Real-time ผ่าน gRPC/gNMI แต่ MDT **ไม่รู้เลย
ว่า Traffic ที่วิ่งผ่าน Interface นั้นเป็น "การสนทนา" (Conversation) ระหว่างใครกับใคร**

**Flexible NetFlow (FNF)** ตอบคำถามคนละแบบ: **"ใครคุยกับใคร ด้วย Protocol อะไร ปริมาณเท่าไหร่"**
โดยจับข้อมูลเป็น **Flow Record** (5-tuple: Source IP, Destination IP, Source Port, Destination
Port, Protocol) — คำตอบที่ MDT ให้ไม่ได้เลยไม่ว่าจะ Stream ถี่แค่ไหนก็ตาม

| หัวข้อ | Model-Driven Telemetry (Part 54/80) | Flexible NetFlow (Step นี้) |
|---|---|---|
| ตอบคำถาม | "Device/Interface นี้อยู่ในสถานะอะไร" | "ใครคุยกับใคร ปริมาณเท่าไหร่" |
| ระดับข้อมูล | Aggregate Counter ต่อ Interface/Process | Per-Flow (5-tuple) รายละเอียดสูงสุด |
| ใช้ทำอะไรต่อ | Dashboard, Anomaly Detection, Auto-Remediation | Top Talker, Capacity Planning, Security Forensics |
| Hardware Path | Punt บางส่วนขึ้น CPU (YANG Push) | **Hardware Flow Cache ใน UADP ASIC โดยตรง** |

### Hardware Acceleration: ทำไม FNF บน Catalyst 9000 ไม่กิน CPU

Switch รุ่นเก่าหรือ Router ทำ NetFlow แบบ **Software-based** (CPU สร้าง/อัปเดต Flow Cache
ทีละ Packet) ซึ่งกิน CPU สูงมากถ้า Traffic หนัก — Catalyst 9000 ใช้ **Hardware Flow Table
ภายใน UADP ASIC เดียวกับที่ทำ TCAM Lookup** (Step 891) การสร้าง/อัปเดต Flow Entry เกิดขึ้น
**พร้อมกับการ Forward Packet ในรอบ Clock เดียวกัน** ไม่ต้อง Punt ขึ้น CPU เลยสำหรับ Traffic
ปกติ — Software (`fed_main` Process จาก Step 892) เข้ามาเกี่ยวข้องแค่ตอน **Export Record
ที่หมดอายุ (Aged-out) ไปยัง Collector** เท่านั้น

### Config เต็มรูปแบบ: Export ไปยัง NOC-SRV (10.10.99.50, VLAN 99 — ตาม [Part 17](part-017-ntp-syslog-snmp.md))

```
! ===== ขั้นตอนที่ 1: Flow Record — กำหนดว่าจะ Match/Collect Field ไหน =====
DIST-SW1(config)# flow record NOC-FLOW-RECORD
DIST-SW1(config-flow-record)# match ipv4 protocol
DIST-SW1(config-flow-record)# match ipv4 source address
DIST-SW1(config-flow-record)# match ipv4 destination address
DIST-SW1(config-flow-record)# match transport source-port
DIST-SW1(config-flow-record)# match transport destination-port
DIST-SW1(config-flow-record)# match interface input
DIST-SW1(config-flow-record)# collect counter bytes long
DIST-SW1(config-flow-record)# collect counter packets long
DIST-SW1(config-flow-record)# collect timestamp sys-uptime first
DIST-SW1(config-flow-record)# collect timestamp sys-uptime last
DIST-SW1(config-flow-record)# exit

! ===== ขั้นตอนที่ 2: Flow Exporter — ปลายทางที่จะส่ง Record ไป (NOC-SRV) =====
DIST-SW1(config)# flow exporter NOC-FLOW-EXPORTER
DIST-SW1(config-flow-exporter)# destination 10.10.99.50
DIST-SW1(config-flow-exporter)# source Vlan99
DIST-SW1(config-flow-exporter)# transport udp 9995
DIST-SW1(config-flow-exporter)# export-protocol ipfix
DIST-SW1(config-flow-exporter)# template data timeout 60
DIST-SW1(config-flow-exporter)# exit

! ===== ขั้นตอนที่ 3: Flow Monitor — ผูก Record + Exporter เข้าด้วยกัน =====
DIST-SW1(config)# flow monitor NOC-FLOW-MONITOR
DIST-SW1(config-flow-monitor)# record NOC-FLOW-RECORD
DIST-SW1(config-flow-monitor)# exporter NOC-FLOW-EXPORTER
DIST-SW1(config-flow-monitor)# cache timeout active 60
DIST-SW1(config-flow-monitor)# exit

! ===== ขั้นตอนที่ 4: Apply เข้า Interface (ทั้ง Ingress/Egress ตามต้องการ) =====
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR input
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR output
DIST-SW1(config-if)# exit
DIST-SW1(config)# end
DIST-SW1# write memory
```

### Verification

```
DIST-SW1# show flow monitor NOC-FLOW-MONITOR
Flow Monitor NOC-FLOW-MONITOR:
  Description:       User defined
  Flow Record:        NOC-FLOW-RECORD
  Flow Exporter:       NOC-FLOW-EXPORTER
  Cache:
    Type:              normal
    Status:             allocated
    Size:               4096 entries
    Inactive Timeout:    15 secs
    Active Timeout:      60 secs

DIST-SW1# show flow exporter statistics
Flow Exporter NOC-FLOW-EXPORTER:
  Packet send statistics (last cleared 00:12:41 ago):
    Successfully sent:         1284       (1112604 bytes)
  Client send statistics:
    Client: Flow Monitor
      Records added:           4821
      Bytes added:             1112604

DIST-SW1# show flow monitor NOC-FLOW-MONITOR cache format table | head
IPV4 SRC ADDR    IPV4 DST ADDR    TRNS SRC PORT  TRNS DST PORT  INTF INPUT   bytes long   pkts long
10.10.10.10      10.10.30.10      51422           443            Vl10         184320       128
10.10.10.15      10.10.99.50      52011           514            Vl10          8192        64
```

> **สำคัญสำหรับ CCIE**: FNF ที่ Apply บน SVI (`interface Vlan10`) แบบนี้จับได้ทุก Traffic ที่ Route
> ผ่าน SVI นั้น (Inter-VLAN) แต่ **ไม่จับ Traffic ที่ Switch ภายใน VLAN เดียวกันแบบ Pure L2**
> (ไม่ผ่าน SVI เลย) — ถ้าต้องการ Visibility ระดับ L2 ล้วนต้อง Apply บน Physical Interface
> โดยตรงแทน ซึ่งบางรุ่น Catalyst 9000 รองรับผ่าน Wireless/Wired Client Flow ที่แยกออกมาอีกชุด

---

## Step 895 — StackWise/SVL Deep Dive ฉบับปรับปรุง: StackWise-1T และ SVL Bandwidth Design

### ทบทวนสิ่งที่ [Part 34](part-034-stackwise-vss-mec.md) ทำไว้กับ CORE-SW1/CORE-SW2

[Part 34](part-034-stackwise-vss-mec.md) รวม CORE-SW1+CORE-SW2 (Catalyst 9500) เป็น
Logical Switch เดียวผ่าน **StackWise Virtual (SVL)** โดยใช้ `TenGigabitEthernet1/0/5-6`
↔ `TenGigabitEthernet2/0/5-6` (รวม 2×10G = 20 Gbps) เป็น SVL Link — Part นี้จะกลับมาตรวจสอบ
**ตัวเลขนี้ว่าเพียงพอจริงหรือไม่** ในเชิง Capacity Planning ระดับ CCIE

### StackWise-1T — เทคโนโลยี Stacking รุ่นใหม่ (Catalyst 9300X)

**StackWise-1T** คือ StackWise (Physical, ใช้ Stack Cable เฉพาะทางแบบ Step 332 ของ Part 34)
เจนเนอเรชันใหม่ล่าสุดบน **Catalyst 9300X** ที่ยกระดับ Bandwidth ของ Stack Ring จาก
**StackWise-480 (480 Gbps)** เป็น **1 Terabit (1,000,000 Mbps)** — สำคัญสำหรับ DIST-SW
ที่ในอนาคตอาจ Upgrade เป็น Catalyst 9300X และทำ Physical Stack จริง (ต่างจาก Lab ปัจจุบันที่
DIST-SW1–4 ยังเป็น Dual-Homed Routed ตาม [Part 33](part-033-advanced-stp-campus-design.md)
ไม่ใช่ Physical Stack) — ยิ่ง Port ความเร็วสูงขึ้น (Multi-Gig, 25G Access Uplink) ยิ่งต้องการ
Stack Bandwidth ที่สูงตาม ไม่ให้ Stack Ring กลายเป็น Bottleneck ของทั้ง Stack

| Generation | Bandwidth รวม | Platform |
|---|---|---|
| StackWise-160 | 160 Gbps | Catalyst 3750/3850 (EOL) |
| StackWise-480 | 480 Gbps | Catalyst 9200/9300 (Fixed) |
| **StackWise-1T** | **1 Tbps (1000 Gbps)** | Catalyst 9300X (รุ่นใหม่, Multi-Gig/25G Port) |

> **ข้อควรระวัง**: StackWise-1T คือ **Physical Stacking** (สาย Stack Cable) ไม่ใช่ SVL —
> **ไม่เกี่ยวกับ CORE-SW1/CORE-SW2 ในหลักสูตรนี้โดยตรง** เพราะ CORE ใช้ SVL (Ethernet-based)
> ไม่ใช่ Stack Cable แต่ต้องรู้จักไว้เพราะเป็นตัวเลือกสำหรับ Distribution/Access Layer รุ่นใหม่
> และมักถูกถามเปรียบเทียบกับ SVL ในข้อสอบ CCIE Enterprise Infrastructure

### Worked Example: ตรวจสอบว่า SVL Link 20 Gbps ของ CORE เพียงพอหรือไม่

จาก [Part 33](part-033-advanced-stp-campus-design.md)/[Part 34](part-034-stackwise-vss-mec.md)
CORE (Logical) มี MEC ไปยัง DIST-SW1–4 รวม 4 เส้นทาง แต่ละเส้นทางเป็น 2×10G MEC (Po19/29/39/49)
= Uplink Capacity รวม **80 Gbps** (4 × 20 Gbps) ที่ CORE ต้อง Forward:

```
กรณีเลวร้ายที่สุด (Worst Case): MEC Hashing เลือก Path ที่ต้องข้าม Chassis เสมอ
(เช่น Frame เข้าที่ Switch 1 แต่ Hashing เลือก Out-interface ที่อยู่บน Switch 2 เท่านั้น)

Traffic ที่ต้องวิ่งข้าม SVL Link (Worst Case)  ≈  สูงสุดถึงครึ่งหนึ่งของ Total Throughput
                                                    ที่ Chassis นั้นรับเข้ามา

ถ้า CORE รับ Traffic รวมจริง (Sales+Voice+Servers+Wifi ทุก VLAN) ที่ 30 Gbps ในช่วง Peak Hour
→ Worst Case ที่ต้องข้าม SVL Link  ≈  15 Gbps  <  20 Gbps (SVL Link ปัจจุบัน)  → ยังพอ (75% Utilization)

ถ้า Traffic โต 2x (Growth Planning ปกติของ CCIE Design)  → 30 Gbps ต้องข้าม SVL Link
                                                             >  20 Gbps ที่มีอยู่ → OVERSUBSCRIBED
```

> **กฎ Capacity Planning สำหรับ SVL Link (Best Practice จาก Cisco Validated Design)**:
> ควรออกแบบ Bandwidth ของ SVL Link ให้ **ไม่น้อยกว่า 25-50% ของ Total Switch Throughput**
> เสมอ และเผื่อ Growth ล่วงหน้าอย่างน้อย 1 รอบ Budget Cycle (2-3 ปี) — สำหรับหลักสูตรนี้ที่
> CORE-SW1/CORE-SW2 เป็น Catalyst 9500 รองรับ 40G/100G Uplink Module ได้ ทางเลือกที่ปลอดภัย
> กว่าในระยะยาวคือ **อัปเกรด SVL Link จาก 2×10G เป็น 2×40G** (จาก 20 Gbps เป็น 80 Gbps)
> เพื่อไม่ให้ SVL Link กลายเป็น Bottleneck ที่ซ่อนอยู่ (Hidden Bottleneck) ซึ่งมองไม่เห็นจาก
> `show interfaces` ตามปกติ เพราะ Traffic ข้าม SVL Link ไม่ได้แสดงเป็น Drop ที่ Interface
> ปลายทางโดยตรง แต่แสดงเป็น **Latency เพิ่มขึ้นผิดปกติ** เท่านั้น

### คำสั่งตรวจสอบ Utilization ของ SVL Link จริง

```
CORE# show stackwise-virtual bandwidth
Switch#   SVL Interface       Bandwidth      Utilization (5-min avg)
------    -----------------   -----------    ------------------------
1         Te1/0/5              10 Gbps        62%
1         Te1/0/6              10 Gbps        58%
2         Te2/0/5              10 Gbps        60%
2         Te2/0/6              10 Gbps        55%
```

Utilization เกิน 60% อย่างต่อเนื่องคือสัญญาณเตือนที่ CCIE ต้องรีบวางแผนเพิ่ม Bandwidth
SVL Link ก่อนที่จะกลายเป็นปัญหา Latency ที่ Users สังเกตเห็นได้เอง

---

## Step 896 — Perpetual PoE และ Fast PoE

### ปัญหาที่แก้: IP Phone/AP ดับตอน Switch Reload

ตั้งแต่ [Part 3](part-003-ethernet-switching-vlan.md) (IP Phone บน ACCESS-SW1/ACCESS-SW2)
และ [Part 20](part-020-wireless-fundamentals.md) (AP-1/AP-2 บน ACCESS-SW3/ACCESS-SW4)
เราสั่ง `power inline auto` เพื่อให้ Switch จ่ายไฟผ่าน PoE ให้ Endpoint เหล่านี้ — แต่ Default
Behavior ของ Cisco Switch คือ **หยุดจ่ายไฟทันทีที่ Switch เข้าสู่ Boot Sequence** (ตอน Reload/
Software Upgrade) ทำให้ Phone/AP **ดับสนิทเป็นเวลาหลายนาที** ทุกครั้งที่ Switch Reload —
ปัญหานี้ร้ายแรงเป็นพิเศษกับ IP Phone (โทรฉุกเฉิน 911/199 ใช้งานไม่ได้ระหว่าง Maintenance
Window) และ Security Camera (จอมืดระหว่างที่ Switch Reload อาจเป็นช่องโหว่ Security)

### Perpetual PoE — จ่ายไฟต่อเนื่องแม้ Switch Reload

**Perpetual PoE** ทำให้ Switch **ยังคงจ่ายไฟผ่าน Port** ต่อเนื่องแม้ระหว่างที่ Software กำลัง
Reload/Upgrade อยู่ (Power Supply/PoE Controller เป็น Hardware Component ที่แยกอิสระจาก
CPU ที่กำลัง Boot IOS-XE — เป็นไปได้เพราะ PoE Controller Chip ไม่ได้ต้องพึ่ง IOS-XE Software
ในการควบคุม Output Power ระดับพื้นฐาน)

### Fast PoE — จ่ายไฟทันทีตั้งแต่ก่อน IOS-XE Boot เสร็จ

**Fast PoE** ทำให้ Switch จ่ายไฟให้ Port **ทันทีที่ได้รับไฟ AC เข้าเครื่อง** โดยไม่ต้องรอให้
IOS-XE Boot เสร็จสมบูรณ์ก่อน (ปกติ Boot IOS-XE เต็มรูปแบบใช้เวลา 3-5 นาที) — สำคัญสำหรับ
สถานการณ์ไฟดับทั้งอาคารแล้วไฟกลับมา (Power Restore) ที่ต้องการให้ Phone/Camera พร้อมใช้งาน
เร็วที่สุดโดยไม่ต้องรอ Switch Boot เต็มรูปแบบ

### Config: เปิด Perpetual PoE + Fast PoE พร้อมกัน (`poe-ha`) บน ACCESS-SW1 (Catalyst 9200)

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 3
ACCESS-SW1(config-if-range)# description ** IP Phone Port — Perpetual+Fast PoE **
ACCESS-SW1(config-if-range)# power inline port poe-ha
ACCESS-SW1(config-if-range)# exit
ACCESS-SW1(config)# end
ACCESS-SW1# write memory
```

> **หมายเหตุ Platform**: `power inline port poe-ha` เป็นคำสั่งที่มีบน **Catalyst 9200/9300/9400/
> 9500** เท่านั้น — **Catalyst 2960 ไม่รองรับ** (ไม่มี Hardware แยก PoE Controller ออกจาก
> CPU Boot Path) ดังนั้นถ้า ACCESS-SW ในหลักสูตรนี้เป็น Catalyst 2960 ตาม
> [`00-ip-address-plan.md`](00-ip-address-plan.md) (ระบุไว้ว่า "9200/2960" — เลือกได้ทั้งคู่)
> ต้องเลือก **Catalyst 9200** โดยเฉพาะถ้าต้องการ Feature นี้ — เป็นตัวอย่างจริงว่า Platform
> Choice ที่ดูเหมือนเล็กน้อยกระทบ Feature ระดับ CCIE ได้โดยตรง

### UPOE และ UPOE+ — สำหรับอุปกรณ์กินไฟสูง (PTZ Camera, Thin Client)

| มาตรฐาน | Power สูงสุดต่อ Port | ใช้กับอุปกรณ์ |
|---|---|---|
| PoE (802.3af) | 15.4W (Switch จ่าย), ~12.95W ถึง Device | IP Phone รุ่นเก่า, Access Point รุ่นเก่า |
| PoE+ (802.3at) | 30W (Switch จ่าย), ~25.5W ถึง Device | **AP-1/AP-2 (Catalyst 9130)** ตาม [Part 20](part-020-wireless-fundamentals.md) |
| **UPOE (Cisco Proprietary, ปูทางสู่ 802.3bt Type 3)** | **60W** | PTZ Camera, Thin Client, Small Retail Kiosk |
| **UPOE+ (802.3bt Type 4)** | **90W** | Laptop Docking Station, Digital Signage ขนาดใหญ่, LED Lighting |

```
! ตัวอย่าง: เปิด UPOE+ (90W) สำหรับ PTZ Camera บน ACCESS-SW3
ACCESS-SW3(config)# interface GigabitEthernet1/0/10
ACCESS-SW3(config-if)# description ** PTZ Camera - UPOE+ 90W **
ACCESS-SW3(config-if)# power inline auto max 90000
ACCESS-SW3(config-if)# exit
```

### Verification

```
ACCESS-SW1# show power inline GigabitEthernet1/0/1
Interface  Admin  Oper       Power(Watts)   Device       Class  Max
                                             detected
---------  -----  ---------  -------------  -----------  -----  ----
Gi1/0/1    auto   on          6.5            Ieee PD      2       30.0

ACCESS-SW1# show power inline
Available:180.0(w)  Used:19.5(w)  Remaining:160.5(w)

Interface Admin  Oper       Power    Device        Class  Perpetual  Fast
                              (Watts)                       PoE        PoE
--------- -----  ---------  -------  ------------  -----  ---------  -----
Gi1/0/1   auto    on          6.5     Ieee PD        2       Y          Y
Gi1/0/2   auto    on          6.5     Ieee PD        2       Y          Y
Gi1/0/3   auto    on          6.5     Ieee PD        2       Y          Y
```

คอลัมน์ `Perpetual PoE` และ `Fast PoE` ที่เป็น `Y` คือหลักฐานยืนยันว่า Port นั้นจะ**ไม่ดับไฟ**
ระหว่าง Switch Reload ครั้งต่อไป — ทดสอบจริงได้ด้วยการ `reload` Switch แล้วสังเกตว่า IP Phone
ยังคง Powered On (แม้ Network จะขาดชั่วคราวเพราะ Switch เอง Reboot ก็ตาม แต่ตัว Phone
ไม่ดับ ไม่ต้อง Boot ตัวเองใหม่)

---

## Step 897 — MACsec (802.1AE): Hardware-Accelerated Layer 2 Encryption

### MACsec คืออะไร — ต่างจาก IPsec ([Part 51](part-051-vrf-lite-gre-ipsec.md)) อย่างไร

**MACsec (802.1AE)** คือมาตรฐานการเข้ารหัส **ทั้ง Ethernet Frame** (รวม L2/L3/L4 Header
ทั้งหมด ยกเว้น Ethernet Header ชั้นนอกสุดที่ต้องเปิดไว้ให้ Switch อ่าน Address ได้) ระหว่าง
**สองปลายที่เชื่อมกันโดยตรง (Point-to-Point, Single Hop)** — ทำงานที่ **Layer 2** และเข้ารหัส/
ถอดรหัสใน **Hardware ASIC โดยตรง (Line-rate, ไม่มี Throughput Penalty)**

| หัวข้อ | IPsec ([Part 51](part-051-vrf-lite-gre-ipsec.md)) | MACsec (802.1AE) |
|---|---|---|
| ทำงานที่ Layer | 3 (Network Layer) | 2 (Data Link Layer) |
| ระยะทาง | ข้ามหลาย Hop ได้ (ผ่าน Internet/WAN) | **Single Hop เท่านั้น** (เชื่อมตรงระหว่าง 2 อุปกรณ์) |
| Overhead ต่อ Packet | ESP Header ~50-73 byte (ขึ้นกับ Mode) + อาจ Fragment | GCM-AES Header คงที่ ~32 byte, ไม่ Fragment |
| การประมวลผล | Crypto Engine/Software (บาง Router มี Hardware Crypto Card) | **Hardware ASIC โดยตรงใน UADP** — ไม่มี Throughput Drop เลย |
| ใช้ป้องกันอะไร | Site-to-Site VPN ข้าม WAN/Internet ที่ไม่น่าเชื่อถือ | Switch-to-Switch/Host-to-Switch บน Link เดียวที่อาจถูก Physical Tap (สายไฟเบอร์ถูกดักฟัง) |
| ตัวอย่างในหลักสูตรนี้ | WAN-EDGE-1↔BRANCH-RTR ([Part 51](part-051-vrf-lite-gre-ipsec.md)) | **CORE↔DIST-SW1 Link (Step นี้)** |

### MKA (MACsec Key Agreement) — กลไกแลกเปลี่ยน Key

MACsec ใช้ **MKA Protocol** ในการเจรจา Session Key ระหว่างสองปลาย โดยรองรับ 2 รูปแบบหลัก:
**802.1X-based** (ใช้ EAP กับ RADIUS Server เช่น ISE — เหมาะกับ Host-facing Port) และ
**Static PSK (Pre-Shared Key)** ผ่าน Key Chain (เหมาะกับ Switch-to-Switch Link ที่ทั้งสองฝั่ง
บริหารจัดการเอง ไม่มี RADIUS Server เกี่ยวข้อง) — หลักสูตรนี้ใช้แบบ **Static PSK** เพราะเป็น
Infrastructure Link ภายในองค์กรเอง

### Config เต็มรูปแบบ: เข้ารหัส CORE↔DIST-SW1 Link (Member ของ Po19 MEC จาก [Part 34](part-034-stackwise-vss-mec.md))

MACsec ถูก Config ที่ **Physical Member Interface** ของ MEC (ไม่ใช่ที่ Port-channel) เพราะ
802.1AE เป็น Feature ระดับ Physical Link — แต่ละ Member เจรจา MKA Session **แยกกันเป็น
อิสระ** แม้จะอยู่ใน Port-channel เดียวกันก็ตาม (ต่างจาก LACP ที่มองทั้ง Bundle เป็นหน่วยเดียว)

```
! ===== ทำบน CORE (Logical, ใช้ Interface TenGigabitEthernet1/0/1 — Member ของ Po19) =====
CORE(config)# key chain CORE-DIST-KC macsec
CORE(config-keychain-macsec)# key 1000
CORE(config-keychain-macsec-key)# cryptographic-algorithm aes-256-cmac
CORE(config-keychain-macsec-key)# key-string 0 12ab34cd56ef78ab90cd12ef34ab56cd78ef90ab12cd34ef56ab78cd90ef12ab
CORE(config-keychain-macsec-key)# exit
CORE(config-keychain-macsec)# exit

CORE(config)# mka policy CORE-DIST-MKA
CORE(config-mka-policy)# key-server priority 1
CORE(config-mka-policy)# macsec-cipher-suite gcm-aes-256
CORE(config-mka-policy)# exit

CORE(config)# interface TenGigabitEthernet1/0/1
CORE(config-if)# description ### MEMBER-OF-Po19-MEC-TO-DIST-SW1 (MACsec Encrypted) ###
CORE(config-if)# macsec network-link
CORE(config-if)# mka policy CORE-DIST-MKA
CORE(config-if)# mka pre-shared-key key-chain CORE-DIST-KC
CORE(config-if)# end
CORE# write memory
```

```
! ===== ทำบน DIST-SW1 (Interface TenGigabitEthernet1/1/1 — Key Chain เดียวกันทุกตัวอักษร) =====
DIST-SW1(config)# key chain CORE-DIST-KC macsec
DIST-SW1(config-keychain-macsec)# key 1000
DIST-SW1(config-keychain-macsec-key)# cryptographic-algorithm aes-256-cmac
DIST-SW1(config-keychain-macsec-key)# key-string 0 12ab34cd56ef78ab90cd12ef34ab56cd78ef90ab12cd34ef56ab78cd90ef12ab
DIST-SW1(config-keychain-macsec-key)# exit
DIST-SW1(config-keychain-macsec)# exit

DIST-SW1(config)# mka policy CORE-DIST-MKA
DIST-SW1(config-mka-policy)# key-server priority 255
DIST-SW1(config-mka-policy)# macsec-cipher-suite gcm-aes-256
DIST-SW1(config-mka-policy)# exit

DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# description ### MEMBER-OF-Po19-MEC-TO-CORE (MACsec Encrypted) ###
DIST-SW1(config-if)# macsec network-link
DIST-SW1(config-if)# mka policy CORE-DIST-MKA
DIST-SW1(config-if)# mka pre-shared-key key-chain CORE-DIST-KC
DIST-SW1(config-if)# end
DIST-SW1# write memory
```

> **สำคัญ**: `key-server priority` ต้อง **ต่างกัน** ระหว่างสองปลาย (ค่าต่ำสุดชนะ กลายเป็น
> Key Server ผู้สร้าง Session Key จริง — เหมือนหลักการ Priority ต่ำชนะของ Spanning-Tree Root
> Bridge ใน [Part 5](part-005-stp-rstp-mstp.md)) ที่นี่ให้ CORE (`priority 1`) เป็น Key Server
> เพราะเป็นฝั่ง Core Layer ที่ Config โดยทีม Network Core โดยตรง

### Verification

```
CORE# show macsec summary
Interface     Xpn      Untagged Traffic    Cipher                  MTU
Te1/0/1        Y        drop                 GCM-AES-256          9198

CORE# show mka session interface TenGigabitEthernet1/0/1
Interface  Local-TxSCI           Policy-Name    Key-Server    Status
Te1/0/1     0011.2233.0001/0001   CORE-DIST-MKA  Yes (Local)   Secured

CORE# show macsec statistics interface TenGigabitEthernet1/0/1
Transmit Secure Channels
  Encrypted Packets:              184203821
Receive Secure Channels
  Decrypted Packets:               182904112
  Validated Packets:               182904112
```

> **ทำไม MACsec ไม่กระทบ Throughput**: ทุก Encrypt/Decrypt Operation เกิดขึ้นใน UADP ASIC
> ตัวเดียวกับที่ทำ TCAM Lookup/Forwarding (Step 891) — Line-rate เต็มความเร็ว Interface เสมอ
> ไม่มี CPU เข้ามาเกี่ยวข้องเลยในสภาวะปกติ ต่างจาก IPsec แบบ Software ที่มักเห็น Throughput
> ลดลงชัดเจนเมื่อเปิด Encryption (ยกเว้น Router รุ่นที่มี Hardware Crypto Module แยกต่างหาก)

---

## Step 898 — Catalyst 9800 Architecture: ทำไม WLC-1 ใช้ IOS-XE ไม่ใช่ AireOS

### AireOS — สถาปัตยกรรมของ WLC รุ่นก่อนหน้า (5508, 8540 — ไม่ใช่ใน Lab นี้)

WLC รุ่นก่อน Catalyst 9800 (Cisco 5508, 8540) รัน **AireOS** — Operating System แยกอิสระ
ที่ไม่มีความสัมพันธ์ทางสถาปัตยกรรมกับ IOS/IOS-XE ของ Switch/Router เลย มี CLI, Syntax,
Debug Philosophy **คนละแบบโดยสิ้นเชิง** วิศวกรที่ชำนาญ IOS ต้องเรียน AireOS CLI ใหม่ทั้งหมด
แยกต่างหาก และไม่มีทาง Reuse ความรู้ NETCONF/RESTCONF/YANG (Part 49) กับ AireOS ได้เลย
เพราะ AireOS ไม่รองรับ Model-Driven Programmability แบบเดียวกัน

### Catalyst 9800 — IOS-XE เดียวกับ Switch/Router ทุกตัวในหลักสูตรนี้

**WLC-1 (Catalyst 9800-CL)** ในหลักสูตรนี้รัน **IOS-XE เวอร์ชันเดียวกัน (Family เดียวกัน)**
กับ CORE-SW1/CORE-SW2 (Catalyst 9500), DIST-SW1–4 (Catalyst 9300) — สถาปัตยกรรมภายใน
เหมือนกับที่อธิบายใน Step 892 ทุกประการ: Linux Kernel, IOSd, FED (ในกรณี 9800 มี Process
เฉพาะเพิ่มชื่อ `wcm` — Wireless Controller Manager ที่ทำหน้าที่ CAPWAP Termination/AP
Management ตาม [Part 36](part-036-wireless-architecture-deep-dive.md))

```
Catalyst 9800-CL (WLC-1)                    Catalyst 9300 (DIST-SW1)
┌─────────────────────────┐                ┌─────────────────────────┐
│   IOS-XE (Linux-based)   │                │   IOS-XE (Linux-based)   │
│   ┌──────┐  ┌──────┐    │   สถาปัตยกรรม  │   ┌──────┐  ┌──────┐    │
│   │ IOSd │  │ wcm  │    │  ◄──เดียวกัน──►│   │ IOSd │  │ FED  │    │
│   └──────┘  └──────┘    │                │   └──────┘  └──────┘    │
│   CLI: show/debug         │                │   CLI: show/debug         │
│   NETCONF/RESTCONF ได้    │                │   NETCONF/RESTCONF ได้    │
└─────────────────────────┘                └─────────────────────────┘
```

### ผลลัพธ์ที่จับต้องได้: Unified Show/Debug Philosophy

| งาน | AireOS (รุ่นก่อน — ไม่ใช้ใน Lab นี้) | Catalyst 9800 (IOS-XE, WLC-1 ในหลักสูตรนี้) |
|---|---|---|
| ดูสถานะ AP | `show ap summary` (Syntax เฉพาะของ AireOS) | `show ap summary` (Syntax ใกล้เคียง `show ip interface brief` ของ Switch) |
| Debug | `debug client <mac>` (Log Format เฉพาะตัว) | `debug platform condition ...` — **Framework เดียวกับที่ใช้ Debug Switch/Router ทุกตัว** |
| Config Backup | Export ผ่าน GUI/TFTP เฉพาะทาง | `copy running-config` เหมือน Switch/Router ทุกตัว |
| Automation API | จำกัดมาก (ไม่มี NETCONF/RESTCONF) | **NETCONF/RESTCONF/YANG เดียวกับ [Part 49](part-049-netconf-restconf-yang-deep-dive.md)** |
| Model-Driven Telemetry | ไม่รองรับ | **รองรับเต็มรูปแบบ ตาม [Part 54](part-054-network-assurance-telemetry.md)/[Part 80](part-080-model-driven-telemetry-streaming.md)** |

นี่คือเหตุผลที่ [Part 36](part-036-wireless-architecture-deep-dive.md) (CAPWAP Deep Dive) และ
[Part 37](part-037-wireless-security-advanced.md) (Wireless Security) สามารถใช้คำสั่ง
`show`/`debug` ในโทนเดียวกับ Part อื่นๆ ของหลักสูตรได้เลยโดยไม่ต้องสอน CLI แยกชุดใหม่ทั้งหมด —
ความรู้ IOS-XE ที่สร้างมาตั้งแต่ Part 1 (แม้จะเป็น Switch/Router) **Transfer ไปใช้กับ Wireless
Controller ได้โดยตรง** ซึ่งเป็นจุดขายสำคัญของ Catalyst 9800 เทียบกับ AireOS รุ่นก่อน — ลด
Learning Curve และ Operational Silo ระหว่างทีม Wired/Wireless ลงอย่างมาก

### รูปแบบ Deployment ของ Catalyst 9800 — Image เดียวกันทุก Form Factor

| Form Factor | อธิบาย | ใช้ใน Lab นี้ |
|---|---|---|
| 9800-CL | Virtual/Cloud (VM หรือ Public Cloud) | **WLC-1 ในหลักสูตรนี้** |
| 9800-40 / 9800-80 | Hardware Appliance ทางกายภาพ | Production ขนาดใหญ่ (ไม่ใช้ใน Lab) |
| Embedded Wireless (EWC) | รันฝังอยู่ใน Switch Catalyst 9300 บางรุ่นโดยตรง (ไม่ต้องมี WLC แยก) | เหมาะกับสาขาขนาดเล็ก (ไม่ใช้ใน Lab นี้เพราะ Topology มี WLC-1 แยกชัดเจน) |

**ทุก Form Factor รัน IOS-XE Image เดียวกัน (Feature Parity เกือบสมบูรณ์)** — ต่างจาก AireOS
ที่แต่ละรุ่น Hardware มักมีข้อจำกัด Feature ไม่เท่ากันชัดเจนกว่ามาก

---

## Step 899 — Cisco DNA/Catalyst Licensing Model: Essentials, Advantage, Premier

### ทำไมสถาปนิก CCIE ต้องรู้เรื่อง Licensing

Feature ทุกตัวที่เรียนมาใน Part 1-90 (แม้ Hardware รองรับในระดับ Silicon ตาม Step 891) **อาจ
ใช้งานไม่ได้จริง** ถ้า License Tier ที่ซื้อไม่ครอบคลุม — การออกแบบ Network ระดับ CCIE ต้อง
คำนวณ License Tier ควบคู่กับ Hardware BOM (Bill of Materials) เสมอ ไม่ใช่แค่เลือก Hardware
ที่ Spec พอแล้วจบ

### สามระดับหลักของ Cisco DNA Licensing (Subscription-based, ผูกกับ Cisco Smart Account)

| Tier | Feature ที่ครอบคลุม (สรุปหลัก) | ตัวอย่างที่เกี่ยวกับหลักสูตรนี้ |
|---|---|---|
| **DNA Essentials** | Switching/Routing พื้นฐาน, ACL, Basic QoS, Basic SNMP/Syslog | VLAN, Trunk, STP, OSPF/EIGRP/BGP พื้นฐาน (Part 1-32 ส่วนใหญ่) |
| **DNA Advantage** | Advanced QoS, **StackWise Virtual/SVL**, Flexible NetFlow เต็มรูปแบบ, SD-Access Fabric Edge Role พื้นฐาน | [Part 34](part-034-stackwise-vss-mec.md) (SVL), Step 894 (FNF) ของ Part นี้ |
| **DNA Premier** | Advanced Security (MACsec 802.1X-based, TrustSec เต็มรูปแบบ), Cisco ISE Integration เต็มรูปแบบ, **DNA Assurance แบบ AI-driven Insight** | [Part 44](part-044-dot1x-trustsec.md) (TrustSec), [Part 78](part-078-security-integration-ise-trustsec-ftd-umbrella.md) (ISE Integration) |

> **หมายเหตุด้าน Branding**: Cisco ปรับชื่อ/โครงสร้าง License Portfolio เป็นระยะ (จาก DNA
> License เดิม สู่การรวมเข้ากับ Portfolio ชื่อ "Cisco Networking" ในภาพรวมที่ใหม่กว่า) — **แนวคิด
> 3 ระดับ (Essentials/Advantage/Premier) และการผูก Feature ขั้นสูงไว้กับ Tier บนยังเป็นหลักการ
> เดียวกัน** ไม่ว่าชื่อ Marketing จะเปลี่ยนไปอย่างไร สถาปนิกควรตรวจสอบ Data Sheet ล่าสุดของ
> Platform ที่จะซื้อจริงเสมอ ไม่อ้างอิงจากความจำเพียงอย่างเดียว

### Smart Licensing Using Policy (SLP) — โมเดล License ปัจจุบันของ Catalyst 9000

Catalyst 9000 ทุกตัวใช้ **Smart Licensing Using Policy (SLP)** ซึ่งมาแทน Smart License/PAK
แบบเดิม — อุปกรณ์ไม่จำเป็นต้อง "Register" ตลอดเวลาแบบบังคับเหมือนก่อน แต่ยัง Report Usage
กลับไปยัง Cisco Smart Software Manager (CSSM) ตาม Policy ที่กำหนด (Online ผ่าน Internet
หรือ Offline ผ่าน Cisco Smart Software Manager On-Prem)

```
CORE# show license status
Smart Licensing is ENABLED

License Usage:
  License                    Entitlement Tag           Count  Status
  ---------------------------------------------------------------------
  network-advantage           (DNA_A_Network)             2    IN USE
  dna-advantage                (DNA_A_Add-on)               2    IN USE

CORE# show license summary
Account Information:
  Smart Account: CCIE-LAB-ORG
  Virtual Account: Enterprise-Campus

License Authorization:
  Status: AUTHORIZED
  Last Communication Attempt: SUCCESS
```

### ตัวอย่าง Impact ทางธุรกิจจริงสำหรับหลักสูตรนี้

| Feature ที่เรียนใน Part นี้ | License Tier ที่ต้องมี | ผล ถ้า License ไม่ถึง Tier |
|---|---|---|
| StackWise Virtual (SVL) — [Part 34](part-034-stackwise-vss-mec.md) | DNA Advantage ขึ้นไป | Config `stackwise-virtual` อาจถูกปฏิเสธหรือ Feature ทำงานแบบจำกัด (ขึ้นกับ Platform/Version) |
| Application Hosting (IOx) — Step 893 | มักรวมอยู่ใน Network Essentials/Advantage (ขึ้นกับ Platform) แต่ต้องตรวจ Data Sheet เฉพาะรุ่น | App-hosting Command อาจใช้ไม่ได้เลยถ้า Platform SKU ไม่รองรับ Onboard Compute |
| MACsec แบบ 802.1X-based เต็มรูปแบบ + ISE — Step 897/[Part 44](part-044-dot1x-trustsec.md) | DNA Premier | ใช้ได้แค่ MACsec แบบ Static PSK เท่านั้น (ไม่มี Dynamic Key Rotation ผ่าน ISE) |
| DNA Assurance (AI-driven Insight, Path Trace) | DNA Premier | Dashboard พื้นฐานใช้ได้ แต่ไม่มี Predictive/AI Analytics |

> **บทเรียนสำหรับ CCIE-level Procurement**: การเสนอ Design SD-Access ([Part 38](part-038-sd-access-fundamentals.md)/[Part 77](part-077-sd-access-advanced-design.md)) ให้ลูกค้าโดยไม่คำนวณ
> License Tier ล่วงหน้า เป็นสาเหตุอันดับต้นๆ ของ Project ที่ **Technical Design ถูกต้องสมบูรณ์
> แต่ Budget บวมเกินคาดตอน Implementation** เพราะทีม Technical มักลืมว่า Feature ระดับ Fabric/
> Assurance ผูกกับ Premier Tier ที่มีค่าใช้จ่าย Subscription ต่อปีสูงกว่า Essentials/Advantage
> อย่างมีนัยสำคัญ — ควรใส่ License Cost เข้า BOM ตั้งแต่ Design Phase แรกเสมอ

---

## Step 900 — Lab เต็มรูปแบบ: App Hosting + Flexible NetFlow + MACsec + Perpetual PoE

### ภาพรวม Lab — ครอบคลุมทุก Platform ของหลักสูตร

Lab นี้รวมทั้ง 4 Feature หลักของ Part นี้เข้าด้วยกัน กระจายไปยัง Platform ที่ถูกต้องตามความ
สามารถของ Hardware จริง:

| Feature | Platform ที่ใช้ | เหตุผล |
|---|---|---|
| Application Hosting | **DIST-SW1** (Catalyst 9300) | มี Onboard Compute เพียงพอ (Step 893) |
| Flexible NetFlow → NOC-SRV | **DIST-SW1, DIST-SW2** (Catalyst 9300) | Hardware Flow Table ใน UADP (Step 894) |
| MACsec | **CORE↔DIST-SW1 Link (Member ของ Po19)** | Hardware MACsec บน UADP ของทั้ง Catalyst 9500 (CORE) และ 9300 (DIST-SW1) (Step 897) |
| Perpetual/Fast PoE | **ACCESS-SW1** (Catalyst 9200, IP Phone Port) | ต้องใช้ 9200 เพราะ 2960 ไม่รองรับ `poe-ha` (Step 896) |

### 1) Application Hosting บน DIST-SW1

```
DIST-SW1(config)# iox
DIST-SW1(config)# interface AppGigabitEthernet1/0/1
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk allowed vlan 99
DIST-SW1(config-if)# exit
DIST-SW1(config)# end
DIST-SW1# app-hosting install appid MONITOR-AGENT package flash:monitor-agent.tar
DIST-SW1# app-hosting activate appid MONITOR-AGENT
DIST-SW1# app-hosting start appid MONITOR-AGENT
DIST-SW1# write memory
```

**Verify:**
```
DIST-SW1# show app-hosting list
App id                           State
---------------------------------------------
MONITOR-AGENT                    RUNNING
```

### 2) Flexible NetFlow — Export ไปยัง NOC-SRV (10.10.99.50) จาก DIST-SW1 และ DIST-SW2

```
! ===== DIST-SW1 =====
DIST-SW1(config)# flow record NOC-FLOW-RECORD
DIST-SW1(config-flow-record)# match ipv4 protocol
DIST-SW1(config-flow-record)# match ipv4 source address
DIST-SW1(config-flow-record)# match ipv4 destination address
DIST-SW1(config-flow-record)# match transport source-port
DIST-SW1(config-flow-record)# match transport destination-port
DIST-SW1(config-flow-record)# collect counter bytes long
DIST-SW1(config-flow-record)# collect counter packets long
DIST-SW1(config-flow-record)# exit
DIST-SW1(config)# flow exporter NOC-FLOW-EXPORTER
DIST-SW1(config-flow-exporter)# destination 10.10.99.50
DIST-SW1(config-flow-exporter)# source Vlan99
DIST-SW1(config-flow-exporter)# transport udp 9995
DIST-SW1(config-flow-exporter)# export-protocol ipfix
DIST-SW1(config-flow-exporter)# exit
DIST-SW1(config)# flow monitor NOC-FLOW-MONITOR
DIST-SW1(config-flow-monitor)# record NOC-FLOW-RECORD
DIST-SW1(config-flow-monitor)# exporter NOC-FLOW-EXPORTER
DIST-SW1(config-flow-monitor)# exit
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR input
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR output
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan20
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR input
DIST-SW1(config-if)# ip flow monitor NOC-FLOW-MONITOR output
DIST-SW1(config-if)# end
DIST-SW1# write memory
```

Config บน **DIST-SW2** ใช้ Pattern เดียวกันทุกประการ (เปลี่ยนแค่ Hostname) — Apply เข้า
`interface Vlan10`/`interface Vlan20` เหมือนกัน เพราะ DIST-SW2 ก็ให้บริการ VLAN 10/20
เช่นเดียวกันตาม [`00-ip-address-plan.md`](00-ip-address-plan.md)

**Verify:**
```
DIST-SW1# show flow exporter statistics
Flow Exporter NOC-FLOW-EXPORTER:
  Packet send statistics (last cleared 00:05:12 ago):
    Successfully sent:         512        (445184 bytes)
```

### 3) MACsec บน CORE↔DIST-SW1 (Member ของ Po19)

```
! ===== CORE =====
CORE(config)# key chain CORE-DIST-KC macsec
CORE(config-keychain-macsec)# key 1000
CORE(config-keychain-macsec-key)# cryptographic-algorithm aes-256-cmac
CORE(config-keychain-macsec-key)# key-string 0 12ab34cd56ef78ab90cd12ef34ab56cd78ef90ab12cd34ef56ab78cd90ef12ab
CORE(config-keychain-macsec-key)# exit
CORE(config-keychain-macsec)# exit
CORE(config)# mka policy CORE-DIST-MKA
CORE(config-mka-policy)# key-server priority 1
CORE(config-mka-policy)# macsec-cipher-suite gcm-aes-256
CORE(config-mka-policy)# exit
CORE(config)# interface TenGigabitEthernet1/0/1
CORE(config-if)# macsec network-link
CORE(config-if)# mka policy CORE-DIST-MKA
CORE(config-if)# mka pre-shared-key key-chain CORE-DIST-KC
CORE(config-if)# end
CORE# write memory
```

```
! ===== DIST-SW1 =====
DIST-SW1(config)# key chain CORE-DIST-KC macsec
DIST-SW1(config-keychain-macsec)# key 1000
DIST-SW1(config-keychain-macsec-key)# cryptographic-algorithm aes-256-cmac
DIST-SW1(config-keychain-macsec-key)# key-string 0 12ab34cd56ef78ab90cd12ef34ab56cd78ef90ab12cd34ef56ab78cd90ef12ab
DIST-SW1(config-keychain-macsec-key)# exit
DIST-SW1(config-keychain-macsec)# exit
DIST-SW1(config)# mka policy CORE-DIST-MKA
DIST-SW1(config-mka-policy)# key-server priority 255
DIST-SW1(config-mka-policy)# macsec-cipher-suite gcm-aes-256
DIST-SW1(config-mka-policy)# exit
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# macsec network-link
DIST-SW1(config-if)# mka policy CORE-DIST-MKA
DIST-SW1(config-if)# mka pre-shared-key key-chain CORE-DIST-KC
DIST-SW1(config-if)# end
DIST-SW1# write memory
```

> **หมายเหตุสำคัญเรื่อง SVL Link**: ทีมงานอาจถูกถามในข้อสอบว่า **"ทำไมไม่เปิด MACsec บน
> SVL Link (Te1/0/5-6) เองไปเลย"** — คำตอบคือ Interface ที่ถูก Config เป็น
> `stackwise-virtual link` แล้วจะ**ไม่รองรับการเปิด MACsec ตรงบน Interface นั้น** เพราะ
> Encapsulation ของ SVL เป็น Proprietary Frame ที่พก Control-Plane Sync + Data-Plane
> Traffic ปนกัน ไม่ใช่ Ethernet Frame ปกติที่ MACsec ออกแบบมาครอบ — ถ้าต้องการเข้ารหัส
> Traffic ระหว่างสองห้อง/อาคารที่ Chassis ของ CORE ตั้งอยู่ (เช่น Traffic ประเภทอื่นที่วิ่งคู่ขนาน
> ไปกับ SVL แต่ไม่ใช่ SVL เอง) ต้องเดินสาย **เพิ่มอีกเส้นแยกจาก SVL/DAD Link โดยเฉพาะ**
> (เช่น `TenGigabitEthernet1/0/7` ↔ `TenGigabitEthernet2/0/7`) แล้วเปิด MACsec บนลิงก์นั้น
> ต่างหาก — Lab นี้จึงเลือกสาธิต MACsec บน **CORE-DIST Link (Physical Member ของ MEC)**
> ซึ่งเป็นจุดที่ Config ได้จริงและตรงกับ Requirement "เข้ารหัส CORE-DIST Link" ตามที่วางแผนไว้

**Verify:**
```
CORE# show mka session interface TenGigabitEthernet1/0/1
Interface  Local-TxSCI           Policy-Name    Key-Server    Status
Te1/0/1     0011.2233.0001/0001   CORE-DIST-MKA  Yes (Local)   Secured

DIST-SW1# show mka session interface TenGigabitEthernet1/1/1
Interface   Local-TxSCI            Policy-Name    Key-Server    Status
Te1/1/1      aabb.cc11.0001/0001    CORE-DIST-MKA  No            Secured
```

### 4) Perpetual PoE + Fast PoE บน ACCESS-SW1 (IP Phone Port)

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 3
ACCESS-SW1(config-if-range)# power inline port poe-ha
ACCESS-SW1(config-if-range)# end
ACCESS-SW1# write memory
```

**Verify (ก่อน Reload):**
```
ACCESS-SW1# show power inline GigabitEthernet1/0/1
Interface Admin  Oper       Power    Device        Class  Perpetual  Fast
                              (Watts)                       PoE        PoE
--------- -----  ---------  -------  ------------  -----  ---------  -----
Gi1/0/1   auto    on          6.5     Ieee PD        2       Y          Y
```

**ทดสอบจริง**: `ACCESS-SW1# reload` แล้วสังเกตว่า IP Phone ที่เสียบอยู่ **ไม่ดับไฟเลย** ตลอด
ช่วงที่ ACCESS-SW1 กำลัง Boot ใหม่ (ตรวจสอบด้วยตาจากไฟหน้าจอ Phone หรือด้วย Power Meter
ถ้ามี) — เมื่อ ACCESS-SW1 Boot เสร็จสมบูรณ์และ Link ขึ้นใหม่ Phone จะ Re-register กับ Call
Manager ทันทีโดยไม่ต้องรอ Boot ตัวเองใหม่ทั้งหมด (เพราะไม่เคยดับไฟ)

### สรุปผลลัพธ์ Lab ทั้งหมด

| Feature | คำสั่ง Verify หลัก | ผลลัพธ์ที่ต้องเห็น |
|---|---|---|
| App Hosting | `show app-hosting list` | `MONITOR-AGENT` สถานะ `RUNNING` บน DIST-SW1 |
| Flexible NetFlow | `show flow exporter statistics` | `Successfully sent` เพิ่มขึ้นต่อเนื่อง ไปยัง 10.10.99.50 |
| MACsec | `show mka session interface ...` | Status `Secured` ทั้งสองฝั่ง CORE และ DIST-SW1 |
| Perpetual/Fast PoE | `show power inline` | คอลัมน์ `Perpetual PoE`/`Fast PoE` เป็น `Y`, Phone ไม่ดับไฟตอน Reload |

Lab นี้ถือว่าสำเร็จเมื่อทั้ง 4 Feature Verify ผ่านพร้อมกันโดยไม่กระทบ Feature เดิมจาก Part
ก่อนหน้าเลย (OSPF Neighbor ของ Po19 ยังคง FULL, VLAN 10/20/30/40/99 ยัง Forward ปกติ,
SVL ของ CORE ยังคง Active/Standby ถูกต้อง)

---

## แบบฝึกหัดทวนความเข้าใจ Part 90

1. UADP ASIC ต่างจาก Fixed-Function ASIC ของ Catalyst รุ่นก่อน 9000 อย่างไร และกลไก
   Recirculation มีส่วนช่วยให้ VXLAN Encap/Decap ทำงานที่ Wire-speed ได้อย่างไร
2. อธิบายว่า IOS-XE (Linux-based, Modular) ต่างจาก IOS Classic (Monolithic) อย่างไร และ
   สถาปัตยกรรมนี้เป็นเหตุผลอย่างไรที่ทำให้ NETCONF/RESTCONF ([Part 49](part-049-netconf-restconf-yang-deep-dive.md)) ทำงานได้เสถียร
3. Flexible NetFlow ตอบคำถามที่ Model-Driven Telemetry ([Part 54](part-054-network-assurance-telemetry.md)/[Part 80](part-080-model-driven-telemetry-streaming.md)) ตอบไม่ได้เรื่องอะไร และทำไม FNF บน
   Catalyst 9000 ไม่กิน CPU สูงเหมือน Software-based NetFlow รุ่นเก่า
4. ทำไมจึงเปิด MACsec บน SVL Link (`stackwise-virtual link`) ของ CORE โดยตรงไม่ได้ และ
   ถ้าต้องการเข้ารหัส Traffic ที่วิ่งคู่ขนานไปกับ SVL Link ต้องทำอย่างไร
5. Perpetual PoE กับ Fast PoE ต่างกันอย่างไร และทำไม ACCESS-SW ที่เป็น Catalyst 2960 (ตาม
   [`00-ip-address-plan.md`](00-ip-address-plan.md)) จึงไม่สามารถใช้คำสั่ง `power inline port poe-ha` ได้

**เฉลย:**

1. Fixed-Function ASIC มี Logic ตายตัวตั้งแต่ผลิต เพิ่ม Feature ใหม่ต้องเปลี่ยน Hardware
   ส่วน UADP เป็น Programmable Pipeline ที่ปรับ Logic ผ่าน Microcode ได้ผ่าน Software
   Upgrade — กลไก Recirculation ทำให้ Packet วิ่งผ่าน Pipeline ของ ASIC ซ้ำได้ภายใน Silicon
   เดียวกัน (ไม่ต้อง Punt ขึ้น CPU) ทำให้เติม VXLAN Header ได้ที่ Wire-speed หลักล้าน pps
   ซึ่ง Fixed-Function ASIC รุ่นเก่าทำไม่ได้เลยเพราะต้องพึ่ง Software/CPU ที่ช้ากว่ามาก
2. IOS Classic รัน Process ทุกตัวใน Memory Space เดียวบน RTOS ไม่มี Process Isolation
   Crash หนึ่งอาจทำให้ทั้งระบบ Reload — IOS-XE รันบน Linux Kernel จริง แยก IOSd/FED/
   Process อื่นเป็น Process อิสระ Crash เฉพาะจุด Restart ได้เฉพาะตัว — เพราะ Management
   Plane (NETCONF/RESTCONF Server) เป็น Process แยกจาก Routing/Forwarding โดยสมบูรณ์
   การยิง API หนักๆ จึงไม่แย่ง CPU Cycle ของ Control/Data Plane โดยตรง ทำให้ Automation
   เสถียรพอสำหรับ Production
3. Model-Driven Telemetry ตอบได้แค่สถานะของ Device/Interface เอง (CPU, Counter) แต่ไม่รู้
   ว่า Traffic เป็นการสนทนาระหว่างใครกับใคร — Flexible NetFlow ตอบคำถามนี้ได้ด้วย 5-tuple
   Flow Record — FNF บน Catalyst 9000 ไม่กิน CPU สูงเพราะสร้าง/อัปเดต Flow Entry ใน
   Hardware Flow Table ภายใน UADP ASIC เดียวกับที่ทำ TCAM Lookup พร้อมกับการ Forward
   Packet เลย ไม่ต้อง Punt ขึ้น CPU สำหรับ Traffic ปกติ (CPU เข้ามาแค่ตอน Export Record
   ที่ Aged-out)
4. Interface ที่ถูก Config เป็น `stackwise-virtual link` ใช้ Proprietary Encapsulation
   เฉพาะสำหรับ SVL Control-Plane Sync + Data-Plane ปนกัน ไม่ใช่ Ethernet Frame ปกติที่
   MACsec ออกแบบมาครอบ จึงเปิด MACsec ตรงบน Interface นั้นไม่ได้ — ถ้าต้องการเข้ารหัส
   Traffic ที่วิ่งคู่ขนานกับ SVL ต้องเดินสายเพิ่มเป็น Physical Link แยกต่างหาก (ไม่ใช่ SVL/DAD
   Link) แล้วเปิด MACsec (`macsec network-link` + MKA Policy) บนลิงก์นั้นโดยเฉพาะ
5. Perpetual PoE ทำให้ Switch จ่ายไฟต่อเนื่องแม้ระหว่าง Software Reload/Upgrade ส่วน
   Fast PoE ทำให้จ่ายไฟทันทีตั้งแต่ก่อน IOS-XE Boot เสร็จสมบูรณ์ (ตอน Power Restore) —
   ทั้งสองต้องพึ่ง PoE Controller Chip ที่แยกอิสระจาก CPU Boot Path ซึ่งมีเฉพาะใน Catalyst
   9200/9300/9400/9500 เท่านั้น Catalyst 2960 (Fixed-Function ASIC รุ่นเก่า, ไม่มี PoE
   Controller แยกแบบนี้) จึงไม่มีคำสั่ง `power inline port poe-ha` ให้ใช้เลย

---

## สรุป Part 90

Part นี้พาย้อนกลับไปมองสิ่งที่ซ่อนอยู่ใต้ทุกคำสั่งที่เคย config มาตั้งแต่ Part 1 — **UADP ASIC**
ที่ทำให้ CORE-SW1/CORE-SW2/DIST-SW1-4 เร็วระดับ Wire-speed และ Programmable พอสำหรับ
SD-Access, **IOS-XE** ที่เป็น Linux-based Modular OS อันเป็นฐานที่ทำให้ NETCONF/RESTCONF/
MDT ([Part 49](part-049-netconf-restconf-yang-deep-dive.md), [Part 54](part-054-network-assurance-telemetry.md), [Part 80](part-080-model-driven-telemetry-streaming.md)) เสถียรพอสำหรับ Production, **Application Hosting** ที่เปิดให้ Switch
กลายเป็น Edge Compute Platform จริง, **Flexible NetFlow** ที่เติม Traffic Visibility ระดับ
Flow ที่ Telemetry ให้ไม่ได้, การทบทวน **SVL/StackWise-1T** เรื่อง Bandwidth Planning ที่
ป้องกัน Hidden Bottleneck, **Perpetual/Fast PoE** ที่รักษาไฟให้ Phone/AP/Camera ไม่ดับตอน
Maintenance, **MACsec** ที่เข้ารหัส Layer 2 แบบ Hardware-Accelerated ไม่กระทบ Throughput,
**Catalyst 9800 Architecture** ที่รวมโลก Wired/Wireless ให้ใช้ Show/Debug Philosophy เดียวกัน,
และปิดท้ายด้วย **Licensing Model** ที่เตือนว่า Feature ทางเทคนิคทั้งหมดนี้ต้องมี Budget รองรับ
ที่ถูก Tier ด้วย — พร้อม Lab เต็มรูปแบบที่รวมทั้ง 4 Feature หลักเข้าด้วยกันข้าม Platform จริง
ของหลักสูตร (Catalyst 9500/9300/9200)

Part 1-90 สร้าง Enterprise Campus Network ที่สมบูรณ์ทั้ง Design (Redundancy ทุก Tier),
Security, Automation, และตอนนี้ Platform-level Deep Dive ครบถ้วน — สิ่งที่ยังไม่ได้ทำอย่างเป็น
ระบบคือ **กระบวนการ Troubleshoot** เวลาที่อะไรบางอย่างในระบบทั้งหมดนี้ผิดพลาดจริง
**Part 91 — Advanced Troubleshooting Methodology** จะรวมความรู้ตั้งแต่ Part 1 ถึง Part 90
เข้าเป็น Framework การไล่หาสาเหตุที่เป็นระบบ (Structured Troubleshooting) ตั้งแต่ Layer 1
ไปจนถึง Application Layer ของ Enterprise Network ทั้ง Topology

**ไปต่อ:** [Part 91 — Advanced Troubleshooting Methodology →](part-091-advanced-troubleshooting-methodology.md)
