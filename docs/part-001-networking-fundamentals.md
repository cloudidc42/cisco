# Part 1 — Networking Fundamentals & OSI/TCP-IP Model
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 1–10 จาก 1000**

> Part นี้เป็นจุดเริ่มต้นของหลักสูตรทั้งหมด เราจะ (1) ปูพื้นฐานทฤษฎีเครือข่ายที่จำเป็นสำหรับ
> การอ่าน config ทุกบรรทัดในหลักสูตรนี้ และ (2) สร้าง **Enterprise Lab Topology หลัก**
> ที่จะถูกใช้ซ้ำและขยายต่อเนื่องไปตลอด 100 Part ของหลักสูตร

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 1 | เครือข่ายคอมพิวเตอร์คืออะไร และประเภทของเครือข่าย |
| 2 | OSI 7 Layer Model โดยละเอียด |
| 3 | TCP/IP Model และการเทียบกับ OSI |
| 4 | Encapsulation / De-encapsulation และ PDU แต่ละ Layer |
| 5 | อุปกรณ์เครือข่ายและ Layer ที่ทำงาน (Hub, Switch, Router, Firewall, WLC) |
| 6 | Ethernet & Physical Layer พื้นฐาน (Cable, Duplex, Speed, Connector) |
| 7 | MAC Address และ Frame Format |
| 8 | IP Address พื้นฐาน (Class, Public/Private) |
| 9 | TCP vs UDP และ Port Number |
| 10 | ประกาศ Lab Topology หลักของหลักสูตร + เตรียม Lab Environment |

---

## Step 1 — เครือข่ายคอมพิวเตอร์คืออะไร

**เครือข่ายคอมพิวเตอร์ (Computer Network)** คือกลุ่มของอุปกรณ์ที่เชื่อมต่อกันเพื่อแลกเปลี่ยนข้อมูล
ผ่านสื่อกลาง (Media) ไม่ว่าจะเป็นสายทองแดง สายไฟเบอร์ หรือคลื่นวิทยุ

### ประเภทเครือข่ายตามขนาดพื้นที่

| ประเภท | ชื่อเต็ม | ขนาดพื้นที่ | ตัวอย่าง |
|---|---|---|---|
| PAN | Personal Area Network | < 10 เมตร | Bluetooth, USB |
| LAN | Local Area Network | อาคาร/สำนักงาน | Office LAN, Campus Switch |
| CAN | Campus Area Network | กลุ่มอาคารใกล้กัน | มหาวิทยาลัย, โรงงาน |
| MAN | Metropolitan Area Network | เมือง | ISP Metro Ethernet |
| WAN | Wide Area Network | ประเทศ/ทวีป | MPLS, Internet, SD-WAN |

### ประเภทเครือข่ายตามลักษณะการบริการ

- **Client-Server**: Server ให้บริการรวมศูนย์ (Active Directory, DNS Server, Web Server)
- **Peer-to-Peer (P2P)**: อุปกรณ์เท่าเทียมกัน ไม่มี Server กลาง
- **Enterprise Network**: รวม LAN + WAN + Data Center + Cloud ขององค์กรทั้งหมด — **นี่คือสิ่งที่
  หลักสูตรนี้จะสอนให้ออกแบบและ config ตั้งแต่ต้นจนจบ**

### สถาปัตยกรรม Enterprise Network สมัยใหม่ (3-Tier Hierarchical Model)

```
                    ┌─────────────────────┐
                    │   Internet / WAN     │
                    └──────────┬───────────┘
                    ┌──────────┴───────────┐
                    │   Edge / WAN Router   │  <-- WAN Edge Layer
                    └──────────┬───────────┘
          ┌─────────────────────┴─────────────────────┐
          │                CORE LAYER                  │  <-- High-speed backbone
          │        Core-SW1 <===LACP===> Core-SW2       │
          └───────┬─────────────────────────────┬───────┘
          ┌────────┴────────┐             ┌────────┴────────┐
          │ DISTRIBUTION     │             │ DISTRIBUTION     │  <-- Policy, L3 boundary
          │ Dist-SW1/Dist-SW2│             │ Dist-SW3/Dist-SW4│
          └───┬─────────┬────┘             └────┬────────┬───┘
          ┌────┴───┐ ┌───┴────┐             ┌─────┴──┐ ┌───┴────┐
          │ACCESS-1│ │ACCESS-2│             │ACCESS-3│ │ACCESS-4│  <-- End-user ports
          └────────┘ └────────┘             └────────┘ └────────┘
```

โมเดลนี้เรียกว่า **Three-Tier Hierarchical Network Design** (Core / Distribution / Access)
ซึ่งเป็นมาตรฐานการออกแบบ Enterprise Campus ของ Cisco และเป็นพื้นฐานของ CCDA/CCNP/CCIE ทุกระดับ
เราจะใช้โมเดลนี้เป็นแกนกลางของ Lab ตลอดหลักสูตร

---

## Step 2 — OSI 7 Layer Model โดยละเอียด

**OSI (Open Systems Interconnection) Model** พัฒนาโดย ISO เป็น Reference Model แบ่งการสื่อสาร
เครือข่ายออกเป็น 7 Layer เพื่อให้ง่ายต่อการออกแบบ, สอน, และ troubleshoot

| Layer | ชื่อ | หน้าที่หลัก | PDU | ตัวอย่าง Protocol/อุปกรณ์ |
|---|---|---|---|---|
| 7 | Application | โปรแกรมที่ผู้ใช้เห็นและใช้งาน | Data | HTTP, HTTPS, FTP, SSH, DNS, SMTP |
| 6 | Presentation | แปลงรูปแบบข้อมูล, เข้ารหัส/บีบอัด | Data | SSL/TLS, JPEG, ASCII, EBCDIC |
| 5 | Session | เปิด/ปิด/จัดการ session การสื่อสาร | Data | NetBIOS, RPC, PPTP |
| 4 | Transport | ส่งข้อมูลปลายทางถึงปลายทาง, error recovery | Segment | TCP, UDP |
| 3 | Network | Routing, Logical Addressing | Packet | IP, ICMP, OSPF, EIGRP, BGP |
| 2 | Data Link | Physical Addressing, Framing, Error Detection | Frame | Ethernet, PPP, Frame Relay, MAC, Switch |
| 1 | Physical | ส่งสัญญาณดิบ (bit) ผ่านสื่อกลาง | Bit | Cable, Connector, Hub, NIC |

**เทคนิคจำ (Mnemonic)**: All People Seem To Need Data Processing (Layer 7→1)
หรือ Please Do Not Throw Sausage Pizza Away (Layer 1→7)

### รายละเอียดเชิงลึกแต่ละ Layer ที่ต้องรู้สำหรับ Cisco Exam

**Layer 1 (Physical)** — สิ่งที่วิศวกร Cisco ต้อง troubleshoot จริง:
- Attenuation (สัญญาณลดลงตามความยาวสาย)
- Crosstalk / EMI (คลื่นรบกวน)
- Collision Domain (ขอบเขตที่ frame ชนกันได้ — Hub รวม collision domain, Switch แยก)

**Layer 2 (Data Link)** — แบ่งเป็น 2 sub-layer:
- **LLC (Logical Link Control)**: ควบคุม flow และ error, interface กับ Layer 3
- **MAC (Media Access Control)**: MAC Address, CSMA/CD (Ethernet), Framing

**Layer 3 (Network)** — งานหลักของ Router:
- Logical Addressing (IP Address)
- Path Determination (Routing Table, Routing Protocol)
- Packet Forwarding (Switching ภายใน Router — CEF, Process Switching, Fast Switching)

**Layer 4 (Transport)** — งานหลักของ Firewall/Load Balancer/End host:
- Segmentation & Reassembly
- Connection-oriented (TCP: 3-way handshake) vs Connectionless (UDP)
- Flow Control (Windowing), Error Recovery (ACK/Retransmit)

---

## Step 3 — TCP/IP Model (DoD Model) และการเทียบกับ OSI

TCP/IP Model เป็น Model ที่ **ใช้งานจริงบน Internet** (OSI เป็นแค่ Reference/ทฤษฎี)
มี 4 Layer:

| TCP/IP Layer | เทียบเท่า OSI Layer | Protocol หลัก |
|---|---|---|
| Application | Layer 5-6-7 | HTTP, HTTPS, DNS, DHCP, SSH, Telnet, SNMP, FTP, TFTP |
| Transport | Layer 4 | TCP, UDP |
| Internet | Layer 3 | IP (IPv4/IPv6), ICMP, ARP, IGMP |
| Link (Network Access) | Layer 1-2 | Ethernet, Wi-Fi (802.11), PPP, Frame Relay |

> **สำคัญสำหรับข้อสอบ CCNA**: เมื่อโจทย์พูดถึง "Layer 2 switch" หมายถึง OSI Layer 2 เสมอ
> แม้จะทำงานภายใต้ TCP/IP Model ก็ตาม วิศวกรเครือข่ายใช้เลข Layer แบบ OSI ในการสื่อสารกันเป็นสากล

### เปรียบเทียบภาพรวม

```
OSI Model              TCP/IP Model           ตัวอย่างในหลักสูตรนี้
┌──────────────┐
│ Application  │  ┐
├──────────────┤  │
│ Presentation  │  ├──► Application  ───►  HTTP, SSH ที่ใช้ config Router ผ่าน API
├──────────────┤  │
│  Session     │  ┘
├──────────────┤
│  Transport   │  ───►  Transport    ───►  TCP port 22 (SSH), UDP port 161 (SNMP)
├──────────────┤
│  Network     │  ───►  Internet     ───►  IP Routing (OSPF, EIGRP, BGP ใน Part 11-30)
├──────────────┤
│  Data Link   │  ┐
├──────────────┤  ├──►  Link         ───►  VLAN, Trunk, STP (Part 3-6), Ethernet Frame
│  Physical    │  ┘
└──────────────┘
```

---

## Step 4 — Encapsulation / De-encapsulation และ PDU แต่ละ Layer

เมื่อข้อมูลถูกส่งจาก Host A ไป Host B ข้อมูลจะถูก **ห่อหุ้ม (Encapsulate)** ทีละ Layer จากบนลงล่าง
และ **แกะห่อ (De-encapsulate)** จากล่างขึ้นบนที่ปลายทาง

```
                 SENDER (Encapsulation)                    RECEIVER (De-encapsulation)
┌───────────────────────────────────────┐        ┌───────────────────────────────────────┐
│ L7  [        DATA        ]             │        │ L7  [        DATA        ]             │
│ L4  [TCP Hdr[  DATA  ]]  = Segment      │  --->  │ L4  [TCP Hdr[  DATA  ]]  = Segment      │
│ L3  [IP Hdr[TCP[ DATA ]]] = Packet      │  --->  │ L3  [IP Hdr[TCP[ DATA ]]] = Packet      │
│ L2  [Eth Hdr[IP[..]]Eth Trailer]=Frame │  --->  │ L2  [Eth Hdr[IP[..]]Eth Trailer]=Frame │
│ L1  101010110101...  = Bits            │  --->  │ L1  101010110101...  = Bits            │
└───────────────────────────────────────┘        └───────────────────────────────────────┘
```

### PDU (Protocol Data Unit) ชื่อเรียกในแต่ละ Layer — **ต้องจำให้ขึ้นใจ เพราะใช้ตลอดหลักสูตร**

| Layer | ชื่อ PDU |
|---|---|
| Transport (L4) | **Segment** (TCP) / **Datagram** (UDP) |
| Network (L3) | **Packet** |
| Data Link (L2) | **Frame** |
| Physical (L1) | **Bit** |

เวลา troubleshoot ด้วยคำสั่ง `debug` หรืออ่าน Wireshark capture วิศวกรมักพูดว่า "ping ไม่ผ่านที่ Layer 3"
หรือ "frame ถูก drop ที่ switch" — การใช้คำศัพท์ PDU ให้ถูก Layer เป็นสิ่งจำเป็นสำหรับ CCNP/CCIE

---

## Step 5 — อุปกรณ์เครือข่ายและ Layer ที่ทำงาน

| อุปกรณ์ | Layer ที่ทำงาน | หน้าที่ | ตัวอย่างในหลักสูตร |
|---|---|---|---|
| Hub / Repeater | Layer 1 | ขยายสัญญาณ, broadcast ทุกพอร์ต (ล้าสมัยแล้ว) | ไม่ใช้ในหลักสูตรนี้ |
| NIC (Network Interface Card) | Layer 1-2 | เชื่อมต่อ Host เข้าเครือข่าย | PC/Server ทุกเครื่องใน Lab |
| Switch (Layer 2) | Layer 2 | Forward frame ตาม MAC Address Table | ACCESS-SW, DIST-SW (Part 3-6) |
| Multilayer Switch (Layer 3) | Layer 2-3 | Switch + Routing ในตัว (SVI, Inter-VLAN) | CORE-SW, DIST-SW (Part 10, 33) |
| Router | Layer 3 | Routing ระหว่าง Network ต่างๆ | EDGE-RTR, WAN-RTR (Part 9-19) |
| Firewall | Layer 3-7 | กรอง traffic ตาม policy | FW-1 (Part 43-45) |
| WLC (Wireless LAN Controller) | Layer 2-3 | จัดการ Access Point ทั้งหมด | WLC-1 (Part 20, 36-37, 89) |
| Access Point (AP) | Layer 1-2 | ให้บริการ Wi-Fi แก่ Client | AP-1, AP-2 (Part 20) |
| Load Balancer | Layer 4-7 | กระจาย traffic ไปหลาย server | กล่าวถึงใน Part 93 (Cloud) |

---

## Step 6 — Ethernet & Physical Layer พื้นฐาน

### ประเภทสาย (Cable Types)

| ชนิดสาย | มาตรฐาน | ความเร็วสูงสุด | ระยะทางสูงสุด |
|---|---|---|---|
| UTP Cat5e | 100BASE-TX/1000BASE-T | 1 Gbps | 100 เมตร |
| UTP Cat6 | 1000BASE-T/10GBASE-T | 10 Gbps (ระยะสั้น) | 55 เมตร (10G) / 100 เมตร (1G) |
| UTP Cat6a | 10GBASE-T | 10 Gbps | 100 เมตร |
| Fiber Multimode (MMF) | 1000BASE-SX/10GBASE-SR | 10-40 Gbps | 300-550 เมตร |
| Fiber Singlemode (SMF) | 1000BASE-LX/10GBASE-LR | 10-100+ Gbps | 10-80+ กิโลเมตร |

### สายตรง vs สายครอส (สำคัญสำหรับ Physical Lab)

- **Straight-through**: PC↔Switch, Switch↔Router (อุปกรณ์คนละประเภท)
- **Crossover**: PC↔PC, Switch↔Switch (รุ่นเก่า), Router↔Router
- อุปกรณ์สมัยใหม่รองรับ **Auto-MDIX** จึงใช้สายแบบไหนก็ได้ (Cisco Catalyst ทุกรุ่นปัจจุบันรองรับ)

### Duplex Mode

| Mode | คำอธิบาย | ปัญหาที่เกิดถ้า mismatch |
|---|---|---|
| Half-Duplex | ส่ง/รับได้ทีละทาง ต้องใช้ CSMA/CD | Collision, late collision |
| Full-Duplex | ส่ง/รับพร้อมกันได้ ไม่มี collision | ถ้าอีกฝั่ง half → เกิด FCS error, runts |

**คำสั่งตรวจสอบ duplex/speed บน Cisco IOS:**
```
Switch# show interfaces GigabitEthernet0/1 status
Switch# show interfaces GigabitEthernet0/1
Switch# show controllers ethernet-controller GigabitEthernet0/1 | include Duplex
```

---

## Step 7 — MAC Address และ Frame Format

**MAC Address** คือ Physical Address ขนาด 48 bit (6 byte) เขียนเป็น Hexadecimal
เช่น `00:1A:2B:3C:4D:5E` หรือในรูปแบบ Cisco `001a.2b3c.4d5e`

- 3 byte แรก (24 bit) = **OUI (Organizationally Unique Identifier)** — ระบุผู้ผลิต (Cisco OUI เช่น `00:00:0C`)
- 3 byte หลัง = Serial Number ที่ผู้ผลิตกำหนดเอง (unique)
- MAC Address ประเภทพิเศษ: **Broadcast** = `FFFF.FFFF.FFFF`, **Multicast** = bit ต่ำสุดของ byte แรกเป็น 1

### Ethernet II Frame Format (มาตรฐานที่ใช้ทั่วไปปัจจุบัน)

```
┌────────────┬────────────┬──────┬────────────────┬─────┐
│ Preamble   │ Dest MAC   │ Src  │ Type/  │ Data   │ FCS │
│ (7B) +SFD  │ (6 Byte)   │ MAC  │ Length │ (46-   │ (4  │
│ (1B)       │            │(6B)  │ (2B)   │ 1500B) │Byte)│
└────────────┴────────────┴──────┴────────────────┴─────┘
```

- **Preamble + SFD**: sync สัญญาณ (ไม่นับใน frame size)
- **Type/Length field**: ถ้า ≥ 0x0600 = EtherType (เช่น `0x0800`=IPv4, `0x0806`=ARP, `0x86DD`=IPv6)
- **FCS (Frame Check Sequence)**: CRC-32 สำหรับตรวจ error — ถ้าผิดพลาด frame จะถูก drop (ไม่มีการ
  retransmit ที่ Layer 2 ของ Ethernet ธรรมดา ต้องให้ Layer 4 (TCP) จัดการ)
- ขนาด frame มาตรฐาน: **64-1518 byte** (ไม่รวม Preamble/SFD) — เล็กกว่า 64 = **Runt**,
  ใหญ่กว่า 1518 = **Giant** (ยกเว้นเปิด Jumbo Frame)

---

## Step 8 — IP Address พื้นฐาน

**IPv4 Address** มีขนาด 32 bit เขียนเป็น Dotted Decimal 4 ส่วน เช่น `192.168.1.1`

### IPv4 Class (แนวคิดดั้งเดิม — ปัจจุบันใช้ CIDR แต่ยังต้องรู้พื้นฐาน)

| Class | Range แรกของ Octet 1 | Default Mask | ใช้งาน |
|---|---|---|---|
| A | 1–126 | /8 (255.0.0.0) | เครือข่ายขนาดใหญ่มาก |
| B | 128–191 | /16 (255.255.0.0) | เครือข่ายขนาดกลาง |
| C | 192–223 | /24 (255.255.255.0) | เครือข่ายขนาดเล็ก |
| D | 224–239 | - | Multicast |
| E | 240–255 | - | สำรอง/วิจัย |

### Private IP Address (RFC 1918) — ใช้ใน Lab ทั้งหมดของหลักสูตรนี้

| Class | Range |
|---|---|
| A | 10.0.0.0 – 10.255.255.255 (10.0.0.0/8) |
| B | 172.16.0.0 – 172.31.255.255 (172.16.0.0/12) |
| C | 192.168.0.0 – 192.168.255.255 (192.168.0.0/16) |

Public IP คือ IP ที่ไม่อยู่ในช่วงข้างบน ใช้สื่อสารบน Internet โดยตรง (ต้องมี NAT
ถ้า Private IP จะออก Internet — เรียนละเอียดใน Part 15)

---

## Step 9 — TCP vs UDP และ Port Number

| คุณสมบัติ | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | มี (ACK, Retransmit, Sequence) | ไม่มี |
| Ordering | รับประกันลำดับ | ไม่รับประกัน |
| Speed | ช้ากว่า (overhead มาก) | เร็วกว่า |
| Header Size | 20 byte | 8 byte |
| ตัวอย่างการใช้งาน | HTTP/HTTPS, SSH, FTP, Telnet | DNS query, DHCP, SNMP, VoIP, Video Streaming, Syslog |

### TCP 3-Way Handshake

```
Client                              Server
  │  ------ SYN (seq=100) ------->    │
  │  <--- SYN-ACK (seq=300,ack=101)-- │
  │  ------ ACK (ack=301) ------->    │
  │        [Connection Established]   │
```

### Well-Known Port ที่ต้องจำสำหรับ Cisco (ใช้ตลอดหลักสูตร)

| Port | Protocol | ใช้ทำอะไร |
|---|---|---|
| 22/TCP | SSH | Remote config อุปกรณ์ Cisco (ใช้ตลอดหลักสูตร) |
| 23/TCP | Telnet | Remote config (ไม่ปลอดภัย ไม่ควรใช้จริง) |
| 53/TCP,UDP | DNS | แปลงชื่อ ↔ IP |
| 67-68/UDP | DHCP | แจก IP อัตโนมัติ (Part 16) |
| 69/UDP | TFTP | โอนไฟล์ IOS/config (Part 2) |
| 80/TCP | HTTP | Web/GUI management |
| 161-162/UDP | SNMP | Network Monitoring (Part 17) |
| 179/TCP | BGP | Routing Protocol (Part 29-30) |
| 443/TCP | HTTPS | Secure Web/API (RESTCONF ใน Part 49) |
| 514/UDP | Syslog | Log message (Part 17) |
| 830/TCP | NETCONF | Network Automation (Part 49) |

---

## Step 10 — Lab Topology หลักของหลักสูตร + เตรียม Lab Environment

### เครื่องมือที่ต้องติดตั้ง (เลือกอย่างใดอย่างหนึ่ง)

1. **Cisco Packet Tracer** (ฟรีสำหรับผู้เรียน Cisco NetAcad) — เหมาะกับ CCNA
2. **Cisco Modeling Labs (CML) / EVE-NG** — เหมาะกับ CCNP/CCIE (รองรับ IOSv, IOS-XE, NX-OS, ASAv)
3. **GNS3** — ฟรี รองรับ real Cisco IOS image (ต้องมี license IOS เอง)

> หลักสูตรนี้จะเขียน config โดยอ้างอิงจาก **Cisco IOS / IOS-XE CLI** เป็นหลัก (ใช้ได้กับทั้ง
> Packet Tracer, CML, GNS3, และอุปกรณ์จริง) ส่วนที่เป็น NX-OS หรือ WLC AireOS จะระบุไว้ชัดเจน

### Enterprise Lab Topology หลัก (จะขยายต่อเนื่องทุก Part)

```
                                   ┌───────────────┐
                                   │   INTERNET     │
                                   └───────┬───────┘
                                   ┌───────┴───────┐
                                   │  ISP-RTR       │
                                   └───────┬───────┘
                        ┌──────────────────┴──────────────────┐
                        │              WAN-EDGE-1                │  <- Part 19, 39, 76
                        └──────────────────┬──────────────────┘
                  ┌────────────────────────┴────────────────────────┐
                  │                     FW-1 (ASA/FTD)                 │  <- Part 43-45
                  └────────────────────────┬────────────────────────┘
        ┌──────────────────────────────────┴──────────────────────────────────┐
        │                          CORE-SW1 <===LACP Po1===> CORE-SW2           │  <- Part 6, 33
        └────┬─────────────────────────────────────────────────────────┬────┘
   ┌──────────┴──────────┐                                   ┌──────────┴──────────┐
   │  DIST-SW1 / DIST-SW2 │  <- HSRP/VRRP (Part 18)          │  DIST-SW3 / DIST-SW4 │
   └────┬────────────┬───┘                                   └────┬────────────┬───┘
┌────────┴───┐  ┌─────┴──────┐                             ┌────────┴───┐  ┌─────┴──────┐
│ ACCESS-SW1 │  │ ACCESS-SW2  │                             │ ACCESS-SW3 │  │ ACCESS-SW4  │
│ VLAN 10,20 │  │ VLAN 10,20  │                             │ VLAN 30,40 │  │ VLAN 30,40  │
└────────────┘  └────────────┘                             └────────────┘  └────────────┘
      │                │                                          │                │
   PC1-PC5          Phone/AP                                   Server1-3        WLC+AP
  (Sales VLAN10)   (Voice VLAN20)                            (Servers VLAN30)  (Wi-Fi VLAN40)
```

### รายชื่ออุปกรณ์ทั้งหมดใน Lab (จะอ้างอิงชื่อนี้ตลอดหลักสูตร)

| Hostname | Role | Platform แนะนำ |
|---|---|---|
| ISP-RTR | จำลอง ISP | CSR1000v / IOSv |
| WAN-EDGE-1, WAN-EDGE-2 | WAN Edge Router | ISR4451 / CSR1000v |
| FW-1 | Firewall | ASAv / FTDv |
| CORE-SW1, CORE-SW2 | Core Switch | Catalyst 9500 (C9500v) |
| DIST-SW1–4 | Distribution Switch | Catalyst 9300 |
| ACCESS-SW1–4 | Access Switch | Catalyst 9200/2960 |
| WLC-1 | Wireless Controller | Catalyst 9800-CL |
| AP-1, AP-2 | Access Point | Catalyst 9130 |
| PC1–PC5, Server1–3 | End Devices | Windows/Linux VM |

### แผน VLAN เบื้องต้น (จะละเอียดใน Part 3)

| VLAN ID | ชื่อ | Subnet |
|---|---|---|
| 10 | SALES | 10.10.10.0/24 |
| 20 | VOICE | 10.10.20.0/24 |
| 30 | SERVERS | 10.10.30.0/24 |
| 40 | WIFI | 10.10.40.0/24 |
| 99 | MGMT | 10.10.99.0/24 |

### คำสั่งเริ่มต้นตรวจสอบว่าเชื่อมต่ออุปกรณ์ได้ (ทดสอบ Physical/Layer 1 ก่อนเริ่ม Part 2)

```
! บน Router/Switch หลัง console เข้าไปแล้ว
Router> show version
Router> show ip interface brief
Router> show interfaces status
```

---

## แบบฝึกหัดทวนความเข้าใจ Part 1

1. PDU ของ Layer 4 คืออะไร และของ Layer 2 คืออะไร?
2. Switch ทำงานที่ OSI Layer ใด และ Multilayer Switch ต่างจาก Switch ธรรมดาอย่างไร?
3. IP Address `172.20.5.10` อยู่ใน Private range ของ Class ใด?
4. อธิบายขั้นตอน TCP 3-Way Handshake โดยละเอียด
5. Port 179/TCP ใช้สำหรับ Protocol อะไร และจะได้เรียนใน Part ไหน?

**เฉลย:**
1. Layer 4 = Segment (TCP) / Datagram (UDP), Layer 2 = Frame
2. Layer 2 (Data Link) — Multilayer Switch ทำงานทั้ง Layer 2 และ Layer 3 (มี Routing/SVI ในตัว)
3. Class B (172.16.0.0/12)
4. SYN → SYN-ACK → ACK (ดู diagram ใน Step 9)
5. BGP (Border Gateway Protocol) — เรียนละเอียดใน Part 29-30

---

## สรุป Part 1

Part นี้ปูพื้นฐาน OSI/TCP-IP Model, Ethernet, IP, TCP/UDP และประกาศ Lab Topology หลักที่จะใช้
ตลอดหลักสูตร ✅ **พร้อมสำหรับ Part 2**: เราจะเริ่มลงมือจริงกับ **Cisco IOS CLI** — เข้าสู่ Router/Switch
ครั้งแรก, Mode ต่างๆ ของ CLI, และการตั้งค่าเริ่มต้น (hostname, password, banner, save config)

**ไปต่อ:** [Part 2 — Cisco IOS CLI Basics & Initial Device Setup →](part-002-ios-cli-basics.md)
