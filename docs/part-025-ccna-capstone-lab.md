# Part 25 — CCNA Capstone Lab & Exam Prep
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 241–250 จาก 1000**

> ต่อจาก [Part 24 — QoS Fundamentals](part-024-qos-fundamentals.md) นี่คือ **Part สุดท้ายของระดับ
> CCNA (200-301)** — Part นี้**ไม่มีเทคโนโลยีใหม่**ให้เรียน แต่เป็น **Capstone Lab** ที่รวบรวม
> ทุกอย่างที่สร้างมาตั้งแต่ Part 1 เข้าด้วยกันเป็นภาพเดียว, ฝึกลำดับการ Bring-up จริง,
> ฝึก Troubleshoot แบบ Systematic, และเตรียมสอบ **Cisco CCNA 200-301** ให้พร้อมที่สุด ปิดท้ายด้วย
> Final Running-Config Appendix ที่เป็นเอกสารอ้างอิงสมบูรณ์ของทั้ง 24 Part ที่ผ่านมา

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 241 | Full Topology Recap — ภาพ Enterprise Lab ฉบับสมบูรณ์ (Part 1-24) |
| 242 | CCNA 200-301 Exam Blueprint Mapping — Domain ไหนอยู่ Part ไหน |
| 243 | Capstone Scenario A: "Day 1 Bring-up" — Runbook คอนฟิกจากเครื่องเปล่า |
| 244 | Capstone Scenario B: "Security Hardening Day 2" — ลำดับ ACL/Port Security/AAA |
| 245 | Troubleshooting Scenario 1 — PC1 (VLAN10) ping Server1 (VLAN30) ไม่ผ่าน |
| 246 | Troubleshooting Scenario 2 — HSRP Failover ทำงานไม่ตรงตามที่คาด |
| 247 | กับดักข้อสอบ CCNA ที่พบบ่อย + 5 คำถามฝึกฝนพร้อมเฉลยละเอียด |
| 248 | Hands-on Lab Exam Simulation — โจทย์จับเวลา + Model Answer เต็มรูปแบบ |
| 249 | เคล็ดลับวันสอบจริง + แผนฝึกต่อเนื่องหลัง Part นี้ |
| 250 | Final Running-Config Appendix — CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2 ฉบับสมบูรณ์ |

---

## Step 241 — Full Topology Recap: ภาพ Enterprise Lab ฉบับสมบูรณ์ (Part 1-24)

ก่อนสอบและก่อนลง Lab Scenario ต่อไปในนี้ ต้อง**หยุดแล้วมองภาพรวมทั้งหมดอีกครั้ง** นี่คือ Diagram
ที่สมบูรณ์ที่สุดในหลักสูตรระดับ CCNA — รวมทุก Layer, ทุก Protocol, ทุก Security Feature ที่ผ่านมา
ทั้ง 24 Part เข้าด้วยกันเป็นภาพเดียว

```
                                          ┌───────────────────┐
                                          │     INTERNET       │
                                          └──────────┬──────────┘
                                          ┌──────────┴──────────┐
                                          │      ISP-RTR         │  1.1.1.254/32
                                          └────┬─────────────┬───┘
                             203.0.113.0/30│               │203.0.113.4/30
                                          ┌────┴───┐     ┌────┴───┐
                                          │WAN-EDGE-1│═════│WAN-EDGE-2│  10.10.254.8/30 (cross-link)
                                          │1.1.1.21  │     │1.1.1.22  │  OSPF Area 0 + ASBR
                                          │(Part 19) │     │(Part 19) │  (redistribute static default)
                                          └────┬───┘     └────┬───┘
                                     10.10.254.0/30│       │10.10.254.4/30
                                          ┌────┴─────────────┴───┐
                                          │        FW-1 (ASA/FTD)      │  <- อ้างอิงถึงใน Part 43-45 (CCIE)
                                          └────┬─────────────────┴───┘
   ═══════════════════════════ OSPF Area 0 (Backbone) ═══════════════════════════
                          ┌───────────────────────────────────────┐
                          │  CORE-SW1 ⇄ CORE-SW2  (Po1, LACP, no switchport, routed) │
                          │  1.1.1.1      10.255.0.0/30      1.1.1.2 │
                          │  root: n/a (Core = Routed only, ไม่มี VLAN ข้าม Core)   │
                          │  ABR (Area0+Area1)         ABR (Area0+Area2)            │
                          └───┬───────────────────────────────┬───┘
             10.255.10.0/30 │Te1/0/1              Te1/0/2│ 10.255.20.0/30      10.255.30.0/30│Te1/0/1  Te1/0/2│10.255.40.0/30
   ═════ OSPF Area 1 ═══════│═══════════════════════════│═════        ═════│═══════════════════════════│═════ OSPF Area 2 ═════
                          ┌───┴───┐                    ┌───┴───┐        ┌───┴───┐                    ┌───┴───┐
                          │DIST-SW1│◄══HSRP/STP══════►│DIST-SW2│        │DIST-SW3│◄══HSRP/STP══════►│DIST-SW4│
                          │1.1.1.11│  (Gi1/0/3 L2 link) │1.1.1.12│        │1.1.1.13│  (Gi1/0/3 L2 link) │1.1.1.14│
                          │VLAN10 Active│  VLAN10 Standby│        │VLAN30 Active│  VLAN30 Standby│
                          │VLAN20 Standby│ VLAN20 Active │        │VLAN40 Standby│ VLAN40 Active │
                          │VLAN99 Active│  VLAN99 Standby│        │VLAN99 Active│  VLAN99 Standby│
                          │STP Root(10,20,99)│STP 2nd Root│        │STP Root(30,40,99)│STP 2nd Root│
                          └──┬──────┬──┘                └──┬──────┬──┘        └──┬──────┬──┘                └──┬──────┬──┘
                     Trunk│      │Trunk                Trunk│      │Trunk    Trunk│      │Trunk                Trunk│      │Trunk
                          │      └────────────┐   ┌─────────┘      │             │      └────────────┐   ┌─────────┘      │
                       ┌──┴───┐            ┌──┴───┴──┐          ┌──┴───┐      ┌──┴───┐            ┌──┴───┴──┐          ┌──┴───┐
                       │ACCESS│            │ ACCESS  │          │      │      │ACCESS│            │ ACCESS  │          │      │
                       │ SW1  │            │  SW2    │          │      │      │ SW3  │            │  SW4    │          │      │
                       └──┬───┘            └──┬──────┘          └──────┘      └──┬───┘            └──┬──────┘          └──────┘
    Port-Security│DHCP Snoop│DAI              │                                   │                    │
                       PC1-PC2                PC3-PC5+Phone                  Server1-3            WLC-1 ⇄ AP-1/AP-2
                     VLAN10(.10-.14)         VLAN10/20 (.15-.30)          VLAN30(.10-.30)          VLAN40 (WLAN SSID)
                     SALES 10.10.10.0/24     VOICE 10.10.20.0/24          SERVERS 10.10.30.0/24    WIFI 10.10.40.0/24

   MGMT-Only Devices ใน VLAN99 (10.10.99.0/24) กระจายอยู่ทั้ง 2 Block (ผ่าน DIST↔Core ปกติ):
   NOC-SRV .50 (NTP/Syslog/SNMPv3 - Part 17) | AAA-SRV .51 (TACACS+ - Part 22) | DHCP-DNS-SRV = 10.10.30.20 (Part 16, อยู่ VLAN30)
```

### สรุปสิ่งที่ประกอบเข้าเป็น Topology นี้ — อ้างอิงย้อนแต่ละ Layer/Feature ไปยัง Part ที่สอน

| Layer/Feature | อุปกรณ์ที่เกี่ยวข้อง | สอนละเอียดที่ |
|---|---|---|
| Physical/OSI/TCP-IP baseline | ทุกอุปกรณ์ | Part 1 |
| IOS CLI, SSH, Initial Setup, Loopback0 | ทุกอุปกรณ์ | Part 2 |
| VLAN, Access Port | ACCESS-SW1-4, DIST-SW1-4 | Part 3 |
| Trunk (802.1Q), Native VLAN, DTP | ทุกลิงก์ระหว่าง Switch | Part 4 |
| STP/RSTP root election ที่ DIST Layer (หลัง Core เป็น Routed) | DIST-SW1(root vlan10/20/99), DIST-SW3(root vlan30/40/99) | Part 5 |
| EtherChannel (LACP) — Po1 CORE↔CORE | CORE-SW1 ↔ CORE-SW2 | Part 6 |
| IPv4 Addressing/Subnetting/VLSM | ทุก Subnet ใน `00-ip-address-plan.md` | Part 7 |
| IPv6 (ULA fd00:.../64 คู่กับทุก VLAN) | ทุก VLAN SVI | Part 8 |
| Static Route (ก่อนมี OSPF) | ทุก Router/L3 Switch (เริ่มถูกแทนที่ใน Part 11) | Part 9 |
| Inter-VLAN Routing: SVI(DIST) + Routed Port(Core) | DIST-SW1-4, CORE-SW1-2 | Part 10 |
| OSPFv2 Single-Area (ต้นแบบ) | ทุก L3 device | Part 11 |
| OSPFv2 Multi-Area: Area0(Core)/Area1(DIST1-2)/Area2(DIST3-4), ABR, Summarization | CORE-SW1/2 (ABR), DIST-SW1-4 | Part 12 |
| EIGRP (สอนคู่ขนานเพื่อเปรียบเทียบ AD/Metric — ไม่ใช้จริงใน Production Lab) | Lab แยกสาธิต | Part 13 |
| ACL: `SALES-TO-SERVERS` (extended), `MGMT-ONLY-VTY` (standard) | DIST-SW3 (Vlan30 SVI in), ทุกอุปกรณ์ (VTY) | Part 14 |
| NAT/PAT (WAN Edge ออก Internet) | WAN-EDGE-1/2 | Part 15 |
| DHCP Server/Relay (`ip helper-address` → 10.10.30.20), DNS | DIST-SW1-4, DHCP-DNS-SRV | Part 16 |
| NTP/Syslog/SNMPv3 → NOC-SRV (10.10.99.50) | ทุกอุปกรณ์ | Part 17 |
| HSRP (v2), Interface Tracking, Preempt | DIST-SW1-4 (Group 10/20/30/40/99) | Part 18 |
| WAN: PPP/MPLS Intro/VPN Intro, WAN-EDGE เป็น ASBR (redistribute default) | WAN-EDGE-1/2, ISP-RTR | Part 19 |
| Wireless: WLC-1 + AP-1/AP-2, SSID map เข้า VLAN40 | WLC-1, AP-1/2, ACCESS-SW3/4 | Part 20 |
| Switch Security: Port Security, DHCP Snooping, DAI | ACCESS-SW1-4 | Part 21 |
| AAA (TACACS+ → AAA-SRV 10.10.99.51), Device Hardening | ทุกอุปกรณ์ | Part 22 |
| Automation พื้นฐาน: NETCONF/RESTCONF/Ansible เบื้องต้น | NOC-SRV (Ansible Control), ทุกอุปกรณ์ (NETCONF agent) | Part 23 |
| QoS: Trust Boundary, Auto QoS VoIP, DSCP marking | ACCESS-SW1-4 (phone ports), DIST-SW1-4, CORE-SW1-2 | Part 24 |

> **ข้อสังเกตสำคัญสำหรับ Capstone**: สังเกตว่า STP Root ของแต่ละ VLAN ในภาพนี้อยู่ที่ **Distribution
> Layer** (DIST-SW1 เป็น Root ของ VLAN10/20/99, DIST-SW3 เป็น Root ของ VLAN30/40/99) ไม่ใช่ที่ Core
> อีกต่อไป — นี่คือผลตามธรรมชาติของการเปลี่ยน Core เป็น **Routed Core ล้วน** ตั้งแต่ Part 10
> (ไม่มี VLAN ใดวิ่งข้าม Core Layer เลย จึง Core ไม่มีส่วนร่วมใน STP Topology ของ VLAN ผู้ใช้งานอีก)
> เป็นตัวอย่างที่ดีว่า Design การ์ตอนหนึ่งเปลี่ยน (Part 10) มีผลกระทบย้อนไปยัง Part ก่อนหน้า (Part 5)
> — วิศวกรจริงต้องตรวจสอบผลกระทบข้าม Layer แบบนี้เสมอเวลา Redesign

---

## Step 242 — CCNA 200-301 Exam Blueprint Mapping

Cisco แบ่งข้อสอบ CCNA 200-301 เป็น 6 Domain พร้อมสัดส่วนน้ำหนักคะแนน (% อย่างเป็นทางการจาก Cisco)
ตารางนี้ Mapping แต่ละ Domain เข้ากับ Part ที่หลักสูตรนี้สอนไปแล้ว:

| # | Domain (ชื่อทางการ) | น้ำหนัก | Part ที่ครอบคลุม |
|---|---|---|---|
| 1.0 | Network Fundamentals | **20%** | Part 1 (OSI/TCP-IP), Part 6 (EtherChannel), Part 7 (IPv4), Part 8 (IPv6) |
| 2.0 | Network Access | **20%** | Part 3 (VLAN), Part 4 (Trunk/DTP), Part 5 (STP/RSTP), Part 6 (EtherChannel), Part 20 (Wireless) |
| 3.0 | IP Connectivity | **25%** | Part 9 (Static Route), Part 10 (Inter-VLAN), Part 11-12 (OSPFv2), Part 18 (HSRP/FHRP) |
| 4.0 | IP Services | **10%** | Part 15 (NAT/PAT), Part 16 (DHCP/DNS), Part 17 (NTP/Syslog/SNMP), Part 19 (WAN) |
| 5.0 | Security Fundamentals | **15%** | Part 2 (SSH/Password), Part 14 (ACL), Part 21 (Port Security/DHCP Snooping/DAI), Part 22 (AAA) |
| 6.0 | Automation and Programmability | **10%** | Part 23 (NETCONF/RESTCONF/Ansible) |
| — | **รวม** | **100%** | Part 1-24 ครบทุก Domain |

### รายละเอียดหัวข้อย่อยที่ Cisco ระบุใน Blueprint จริง เทียบกับ Part ที่สอน

| Blueprint Sub-topic (ตัวอย่างสำคัญ) | Part ที่ตรงกัน |
|---|---|
| 1.1 Explain the role and function of network components | Part 1 (Step 5) |
| 1.5 Compare IPv4 and IPv6 addressing | Part 7, Part 8 |
| 1.9 Compare and contrast collapsed core and three-tier architectures | Part 1 (Step 1), Part 10 (Step 98) |
| 2.4 Configure and verify Layer 2 discovery protocols (CDP/LLDP) | Part 2 (Step 18) |
| 2.5 Configure and verify (Layer 2/Layer 3) EtherChannel (LACP) | Part 6 |
| 2.8 Describe WLAN, WLC, AP modes, and RF concepts | Part 20 |
| 3.1 Interpret the components of a routing table | Part 9, Part 11-12 |
| 3.2 Determine how a router makes a forwarding decision by default | Part 9 |
| 3.3 Configure and verify IPv4 and IPv6 static routing | Part 9 |
| 3.4 Configure and verify single area/multiarea OSPFv2 | Part 11-12 |
| 3.5 Describe the purpose of FHRP | Part 18 |
| 4.1 Configure and verify inside source NAT using static/pools | Part 15 |
| 4.2 Configure and verify NTP operating in client/server mode | Part 17 |
| 4.3 Explain the role of DHCP and DNS | Part 16 |
| 4.5 Describe the capabilities and functions of SNMP | Part 17 |
| 5.1 Define key security concepts (threats, vulnerabilities, exploits...) | Part 22 |
| 5.4 Describe device access control using AAA | Part 22 |
| 5.5 Configure and verify access control lists | Part 14 |
| 5.6 Configure Layer 2 security features (DHCP snooping, DAI, port security) | Part 21 |
| 5.8 Configure WLAN using WPA2 PSK | Part 20 |
| 6.1 Explain how automation impacts network management | Part 23 |
| 6.4 Compare traditional networks with controller-based/SDA networking | Part 23 (ต่อยอดใน Part 38 CCNP) |
| 6.6 Interpret JSON encoded data | Part 23 |

> **ข้อสังเกตสำคัญ**: Domain 3.0 (IP Connectivity, 25%) มีน้ำหนักสูงสุดในข้อสอบ — สอดคล้องกับที่
> หลักสูตรนี้ใช้เวลาสอน Static Route + Inter-VLAN + OSPF + FHRP รวมกันถึง 4 Part เต็ม (Part 9,10,
> 11-12,18) มากกว่ากลุ่มอื่นใด **ควรทวน Part เหล่านี้ให้แม่นที่สุดก่อนสอบจริง**

---

## Step 243 — Capstone Scenario A: "Day 1 Bring-up" (Runbook จากเครื่องเปล่า)

สมมติสถานการณ์: วิศวกรได้รับอุปกรณ์ **Factory-Default ทั้งหมด** (CORE-SW1/2, DIST-SW1-4,
ACCESS-SW1-4, WAN-EDGE-1/2) ต้อง Build ทั้ง Enterprise Lab ให้ใช้งานได้ภายใน "วันเดียว" —
นี่คือ Runbook ลำดับที่ถูกต้องตามหลัก Dependency จริง (ทำผิดลำดับ = เสียเวลา Debug เพิ่มมาก)

### Runbook ฉบับสมบูรณ์ (Checklist ตามลำดับ Phase)

```
PHASE 0 — Physical & Console (ก่อนเริ่ม CLI ใดๆ)
  [ ] เดินสาย Physical ตาม Topology (Step 241) ให้ครบทุกลิงก์
  [ ] ต่อ Console เข้าทุกอุปกรณ์ตามลำดับ เพื่อยืนยัน show version บูตขึ้นสำเร็จ

PHASE 1 — Initial Device Setup (อ้างอิง Part 2)
  [ ] ตั้ง hostname ตรงกับ Topology ทุกตัว (CORE-SW1, CORE-SW2, DIST-SW1-4, ACCESS-SW1-4, ...)
  [ ] no ip domain-lookup, ip domain-name lab.local
  [ ] enable secret + service password-encryption
  [ ] crypto key generate rsa modulus 2048 + ip ssh version 2 (เตรียม SSH ไว้ก่อน AAA ใน Phase 6)
  [ ] username admin privilege 15 secret ... (local user สำรองไว้เสมอ ก่อนเปิด AAA)
  [ ] line vty 0 15 → transport input ssh, login local
  [ ] interface Loopback0 → ใส่ IP ตาม `00-ip-address-plan.md` (ใช้เป็น Router-ID ทุก Routing Protocol)
  [ ] copy running-config startup-config ทุกตัวก่อนไป Phase ถัดไป (กันไฟดับ/reload พลาด)

PHASE 2 — Layer 2 Foundation (อ้างอิง Part 3-6) — ทำที่ ACCESS-SW1-4 และ DIST-SW1-4 ก่อน
  [ ] สร้าง VLAN Database ให้ตรงกันทุกตัว: vlan 10 SALES / 20 VOICE / 30 SERVERS / 40 WIFI / 99 MGMT
  [ ] ตั้ง Access Port บน ACCESS-SW1-4 (switchport mode access + switchport access vlan <id>)
  [ ] ตั้ง Trunk ระหว่าง ACCESS↔DIST (switchport mode trunk, allowed vlan ตาม Access-Layer Mapping)
  [ ] ตั้ง STP: DIST-SW1 root primary (VLAN10/20/99), DIST-SW2 root secondary,
      DIST-SW3 root primary (VLAN30/40/99), DIST-SW4 root secondary
  [ ] ตั้ง EtherChannel Po1 (LACP) ระหว่าง CORE-SW1 ↔ CORE-SW2
  [ ] Verify: show vlan brief, show interfaces trunk, show spanning-tree summary, show etherchannel summary

PHASE 3 — Layer 3 Foundation (อ้างอิง Part 7-10)
  [ ] เปิด ip routing บน CORE-SW1/2 และ DIST-SW1-4 ทุกตัว (ลืมขั้นนี้ = Route ไม่เกิดแม้ SVI up/up)
  [ ] สร้าง SVI ทุก VLAN บน DIST-SW1-4 ตาม IP Plan (`.2`/`.3` ก่อน HSRP)
  [ ] ตั้ง Routed Port (no switchport) สำหรับลิงก์ DIST↔CORE และ CORE↔CORE ตาม Transit Link Plan
  [ ] ตั้ง Static Route ชั่วคราวเป็น Default Route ชี้ขึ้น Core ก่อน (จะถูกแทนที่ด้วย OSPF ใน Phase 4)
  [ ] เพิ่ม IPv6 ULA ตาม Plan คู่กับทุก SVI (Part 8)
  [ ] Verify: ping ข้าม VLAN ภายใน Block เดียวกันให้ผ่านก่อน (ยังไม่ต้องข้าม Core)

PHASE 4 — Dynamic Routing (อ้างอิง Part 11-12)
  [ ] ลบ Static Default Route เดิมของ Phase 3 ออกทุกตัว
  [ ] router ospf 1 บนทุก L3 Device พร้อม router-id อ้าง Loopback0
  [ ] ใส่ ip ospf 1 area <id> ตาม Area Plan: Area0=Po1(Core↔Core), Area1=DIST-SW1/2 block,
      Area2=DIST-SW3/4 block
  [ ] passive-interface default ใต้ router ospf แล้ว no passive-interface เฉพาะ Transit Link จริง
  [ ] area 1 authentication message-digest / area 2 authentication message-digest (ถ้าใช้ MD5)
  [ ] area 1 range 10.10.0.0 255.255.0.0 / area 2 range 10.10.0.0 255.255.0.0 (Summarization บน ABR)
  [ ] Verify: show ip ospf neighbor (ครบทุกคู่ FULL state), show ip route ospf, ping ข้าม Core ทั้ง Topology

PHASE 5 — FHRP: HSRP (อ้างอิง Part 18)
  [ ] standby version 2 ก่อนคำสั่งอื่นเสมอ บนทุก SVI ของ DIST-SW1-4
  [ ] standby <group> ip <VIP .1> ตาม VLAN Plan
  [ ] ตั้ง Priority สลับ Active/Standby ตาม Block (ดู Step 241 diagram) + standby preempt
  [ ] standby <group> track <transit-interface> decrement 60 (กัน Active ค้างบน Uplink ตาย)
  [ ] เปลี่ยน Default Gateway ของ Host ทุกเครื่องจาก `.2`/`.3` เป็น `.1` (VIP)
  [ ] Verify: show standby brief ทุกตัว, ทดสอบ shutdown uplink Active แล้วดู failover จริง

PHASE 6 — Services เสริม (อ้างอิง Part 15-17, 19-20)
  [ ] DHCP Server Pool ที่ DHCP-DNS-SRV + ip helper-address บน SVI ที่ไม่มี DHCP Server ในบล็อกเดียวกัน
  [ ] ntp server 10.10.99.50 ทุกอุปกรณ์ + clock timezone
  [ ] logging host 10.10.99.50 + snmp-server host 10.10.99.50 version 3 priv
  [ ] NAT/PAT ที่ WAN-EDGE-1/2 ออก Internet ผ่าน ISP-RTR
  [ ] WLC-1 + AP-1/AP-2 ขึ้นทะเบียน, Map SSID เข้า VLAN40

PHASE 7 — Save ทุกอุปกรณ์
  [ ] copy running-config startup-config ทุกตัวอีกครั้ง — ปิด Day 1
```

> **กฎเหล็กของ Runbook**: **ห้ามเปิด Dynamic Routing (Phase 4) ก่อนที่ Layer 2 (Phase 2) เสถียร**
> เพราะ OSPF Neighbor จะไม่มีทางขึ้น Full ถ้า Trunk/VLAN ยังมีปัญหาอยู่ใต้ชั้น — และ**ห้ามตั้ง HSRP
> (Phase 5) ก่อน OSPF (Phase 4)** เพราะถ้า Uplink ตาย HSRP ต้อง Track ให้ Priority ลดแล้วให้
> Routing Protocol Reroute ตามจริง ถ้า OSPF ยังไม่ Converge การทำ Failover จะไม่มีประโยชน์

---

## Step 244 — Capstone Scenario B: "Security Hardening Day 2"

Day 2 คือการนำ Lab ที่ Bring-up สำเร็จจาก Step 243 มา **Harden ด้านความปลอดภัย** โดยรวม 3 เรื่อง
จาก Part 14 (ACL), Part 21 (Port Security/DHCP Snooping/DAI), และ Part 22 (AAA) เข้าด้วยกัน —
**ลำดับผิดได้ = Lock ตัวเองออกจากอุปกรณ์ หรือบล็อก DHCP/ARP ที่จำเป็นโดยไม่ตั้งใจ**

### เช็คลิสต์ตามลำดับที่ถูกต้อง พร้อมเหตุผล Dependency

```
DAY 2 — SECURITY HARDENING CHECKLIST

[ ] ขั้น 1: Layer 2 Access-Edge Security ก่อนเสมอ (ACCESS-SW1-4)
    เหตุผล: ต้องปิดรูรั่วที่ปลายทางผู้ใช้งานก่อน เพราะเป็นจุดเข้าออกที่ Attacker เข้าถึงง่ายที่สุด

    1a. switchport port-security (ทุก Access Port ผู้ใช้งาน)
        - switchport port-security maximum 2
        - switchport port-security violation restrict
        - switchport port-security mac-address sticky
        ⚠️ ทำก่อน DHCP Snooping เสมอ เพราะไม่พึ่งพา Snooping Binding Table เลย

    1b. ip dhcp snooping (Global) + ip dhcp snooping vlan 10,20,30,40,99
        - Trust เฉพาะ Uplink ไปยัง DIST (ip dhcp snooping trust)
        - Access Port ทั้งหมด = Untrusted (ค่า Default อยู่แล้ว ไม่ต้องสั่งเพิ่ม)
        ⚠️ ต้องทำก่อน DAI เสมอ เพราะ DAI ใช้ DHCP Snooping Binding Table เป็นฐานข้อมูล
        ตรวจสอบ ARP — ถ้าเปิด DAI ก่อนมี Binding Table ที่สมบูรณ์ Host ที่ได้ IP มาแล้วจะ ARP ไม่ผ่าน

    1c. ip arp inspection vlan 10,20,30,40,99 (Dynamic ARP Inspection)
        - Trust บน Uplink เดียวกับที่ Trust DHCP Snooping (ip arp inspection trust)
        - รอให้ Binding Table จาก 1b โตพอสมควรก่อน Apply จริงใน Production (หรือใช้ arp access-list
          static เสริมสำหรับ Server ที่มี Static IP)

[ ] ขั้น 2: Distribution-Layer Traffic Filtering (DIST-SW3)
    เหตุผล: ทำหลัง Layer 2 Edge มั่นคงแล้ว เพราะ ACL ที่ Distribution ควบคุม "ทิศทางที่อนุญาต"
    ระหว่าง VLAN/Department ซึ่งเป็นนโยบายระดับสูงกว่า Edge Security

    2a. ip access-list extended SALES-TO-SERVERS (ตาม Part 14) — Apply "in" บน SVI Vlan30 ของ
        DIST-SW3 (permit tcp 443/22 จาก SALES ไปยัง Server1 เท่านั้น, deny+log ส่วนที่เหลือ)
    2b. ip access-list standard MGMT-ONLY-VTY — Apply บน line vty 0 15 ทุกอุปกรณ์
        (จำกัดว่าใครมี IP ใน VLAN99 เท่านั้นที่ SSH เข้ามาได้)
    ⚠️ ทดสอบด้วย show access-lists (ดู match counter) ก่อนประกาศ "เสร็จ" เสมอ — เคยพบว่า ACL
    Deny ผิดโดยไม่ตั้งใจแล้วบล็อก DHCP Relay/NTP/Syslog ที่วิ่งข้าม Subnet เดียวกันไปด้วย

[ ] ขั้น 3: AAA & Device Hardening (ทุกอุปกรณ์ - ทำเป็นลำดับสุดท้ายเสมอ)
    เหตุผล: เป็น Attack Surface ที่ "ควบคุมการเข้าถึงอุปกรณ์เอง" — ถ้าทำผิดลำดับ/พลาด
    เสี่ยง Lock ตัวเองออกจากอุปกรณ์ทั้งหมดพร้อมกัน (ต้อง Console เข้าไปแก้ทีละตัว)

    3a. ยืนยันก่อนเสมอว่า AAA-SRV (10.10.99.51) reachable และ TACACS+ Service พร้อมแล้ว
        - ping 10.10.99.51 จากทุกอุปกรณ์ให้ผ่านก่อน
        - test aaa group tacacs+ <user> <pass> legacy (ทดสอบ authen จริงก่อนเปลี่ยน default)
    3b. tacacs server AAA-SRV → address ipv4 10.10.99.51, key <shared-key>
    3c. aaa new-model
    3d. aaa authentication login default group tacacs+ local
        ⚠️ ต้องมี "local" เป็น Fallback เสมอ ไม่งั้นถ้า AAA-SRV ล่ม จะ Login เข้าอุปกรณ์ไม่ได้เลย
    3e. aaa authorization exec default group tacacs+ local
    3f. aaa accounting exec default start-stop group tacacs+
    3g. ทดสอบ Login ผ่าน Session ใหม่ (เปิด SSH คนละหน้าต่างค้างไว้ก่อนตัด Session เดิม!)
        กฎเหล็ก: ห้ามปิด Session ปัจจุบันจนกว่าจะยืนยันว่า Login รอบใหม่ผ่าน AAA สำเร็จแล้วจริง
```

### Dependency Graph สรุปภาพเดียว

```
Port Security ──► DHCP Snooping ──► Dynamic ARP Inspection      (ลำดับตายตัว ห้ามสลับ)
                        │
                        ▼ (Binding Table พร้อมแล้ว)
              ACL ที่ Distribution (SALES-TO-SERVERS, MGMT-ONLY-VTY)
                        │
                        ▼ (Policy ระดับ Traffic วางเสร็จ)
              AAA (ทดสอบ Reachability + Fallback local ก่อนเปลี่ยน Default)
```

---

## Step 245 — Troubleshooting Scenario 1: PC1 (VLAN10) ping Server1 (VLAN30) ไม่ผ่าน

### อาการที่รายงานเข้ามา

> "PC1 (10.10.10.10, VLAN10/SALES) ping ไป Server1 (10.10.30.10, VLAN30/SERVERS) ไม่ผ่านเลย
> (`Request timed out`) แต่ **PC-WIFI (VLAN40/WIFI) ping Server1 ได้ปกติ** และ **PC1 เปิด HTTPS
> ไป Server1 ได้ปกติ (Browser โหลดหน้าเว็บสำเร็จ)**"

### ขั้นตอน Troubleshoot แบบ Systematic (Layer-by-Layer ตามหลัก Part 1)

```
Layer 1 (Physical) — ตัดออกทันที เพราะ PC1 ใช้ Trunk/Access เดียวกันกับ Traffic อื่นที่ผ่านได้
   (HTTPS ผ่าน = Layer 1-4 ของ Path เดียวกันทำงานได้จริง)

Layer 2 (Data Link) — ตัดออก ด้วยเหตุผลเดียวกัน (Frame ไปถึง Gateway ได้แน่นอน เพราะ HTTPS ผ่าน)

Layer 3 (Network) — จุดที่ต้องสงสัยอันดับแรก: Routing มีปัญหาหรือไม่?
```

**Step A — ตรวจ Routing Table ก่อน (แยกให้ออกว่าเป็นปัญหา Route หรือปัญหา Filter)**

```
DIST-SW1# show ip route 10.10.30.0
Routing entry for 10.10.30.0/24
  Known via "ospf 1", distance 110, metric 3
  ...
```
→ **Route มีอยู่ปกติ** (ผ่าน OSPF, Area 1 → Area 0 → Area 2) แปลว่า**ไม่ใช่ปัญหา Routing/OSPF**
ต้องดูขั้นถัดไป: ปัญหาน่าจะอยู่ที่ **Filtering (ACL)** เพราะอาการ "TCP ผ่าน แต่ ICMP ไม่ผ่าน"
เป็นลายเซ็นชัดเจนของ ACL ที่กรองเฉพาะ Protocol

**Step B — ตรวจ ACL ที่ Apply บน Path ปลายทาง (DIST-SW3, SVI Vlan30 — ตาม Part 14)**

```
DIST-SW3# show ip interface vlan 30 | include access list
  Outgoing access list is not set
  Inbound  access list is SALES-TO-SERVERS

DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443 (142 matches)
    30 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22 (0 matches)
    40 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log (37 matches)   <-- ICMP โดน Deny ที่นี่!
    60 remark ==== Explicit permit for everything else ====
    70 permit ip any any (881 matches)
```

**พบ Root Cause แล้ว**: ACL `SALES-TO-SERVERS` ที่ Apply แบบ `in` บน SVI Vlan30 ของ DIST-SW3
อนุญาตเฉพาะ **TCP port 443 และ 22** จาก SALES VLAN (10.10.10.0/24) ไปยัง Server1 เท่านั้น
บรรทัด 50 (`deny ip ... log`) จับ Protocol IP ทั้งหมดที่ไม่ผ่าน 2 บรรทัดแรก **รวมถึง ICMP
(Echo Request ของ ping)** ด้วย — ตรงกับ Match Counter ที่เห็น (37 matches) และตรงกับอาการที่
HTTPS (TCP/443) ผ่านได้ปกติ (ตรงบรรทัด 20)

**ทำไม PC-WIFI (VLAN40) ping ผ่าน?** เพราะ ACL นี้ระบุ Source เป็น `10.10.10.0 0.0.0.255`
(เฉพาะ SALES) เท่านั้น — VLAN40 (WIFI, 10.10.40.0/24) **ไม่ตรงกับ Source ใน ACE บรรทัดไหนเลย**
จึงตกไปที่บรรทัด 70 (`permit ip any any`) ผ่านได้ปกติทุก Protocol

### วิธีแก้ (ตามหลัก Part 14 — แก้ไขด้วย Sequence Number ไม่ต้องลบทั้งชุด)

```
DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW3(config-ext-nacl)# 15 permit icmp 10.10.10.0 0.0.0.255 host 10.10.30.10 echo
DIST-SW3(config-ext-nacl)# 16 permit icmp 10.10.10.0 0.0.0.255 host 10.10.30.10 echo-reply
DIST-SW3(config-ext-nacl)# exit
DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    15 permit icmp 10.10.10.0 0.0.0.255 host 10.10.30.10 echo
    16 permit icmp 10.10.10.0 0.0.0.255 host 10.10.30.10 echo-reply
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    ...

! ทดสอบยืนยันจาก PC1
PC1> ping 10.10.30.10
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125    <-- แก้แล้ว
```

> **บทเรียนสำหรับข้อสอบ**: อาการ "Protocol หนึ่งผ่าน อีก Protocol ไม่ผ่าน ระหว่าง Host คู่เดียวกัน"
> คือลายเซ็นคลาสสิกของ **ACL ที่กรองเฉพาะ Protocol/Port** เกือบทุกครั้ง — ให้ตรวจ `show ip access-
> lists` (ดู Match Counter เพิ่มขึ้นตรงกับเวลาที่ทดสอบ) เป็นคำสั่งแรกๆที่ต้องนึกถึงเมื่อ Routing Table
> ปกติแต่ Traffic บางประเภทไม่ผ่าน

---

## Step 246 — Troubleshooting Scenario 2: HSRP Failover ทำงานไม่ตรงตามที่คาด

### อาการที่รายงานเข้ามา

> ทีม NOC ทดสอบ Resiliency โดยตัดสาย Uplink `TenGigabitEthernet1/1/1` ของ DIST-SW1 (Active
> HSRP ของ VLAN10) — HSRP Failover ไป DIST-SW2 **สำเร็จตามคาด** (PC1 ไม่รู้สึกสะดุดเลย) แต่พอ
> **เสียบสาย Uplink คืน**กลับพบว่า **PC1 หลุดการเชื่อมต่อ TCP Session ไปประมาณ 15-20 วินาที**
> ก่อนจะกลับมาใช้งานได้ปกติ — คำถาม: ทำไม Fail-back ถึงแย่กว่า Fail-over?

### ขั้นตอน Troubleshoot

**Step A — ตรวจสถานะ HSRP ปัจจุบันหลังเสียบสายคืน**

```
DIST-SW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   150 P Active  local           10.10.10.3      10.10.10.1
```

DIST-SW1 กลับมาเป็น **Active ทันที**ที่ Uplink Up (Priority 150 > 100 ของ DIST-SW2 และมี
`preempt` เปิดอยู่ — ตรงตามที่ตั้งใจออกแบบไว้ใน Part 18) แต่นี่คือจุดที่ทำให้เกิดปัญหา

**Step B — ตรวจ OSPF ที่ CORE-SW1 ในช่วงเวลาเดียวกัน**

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11         1   INIT/DROTHER    00:00:38    10.255.10.2     TenGigabitEthernet1/0/1
```

พบว่า Neighbor Adjacency ระหว่าง CORE-SW1 ↔ DIST-SW1 (ผ่าน Te1/0/1↔Te1/1/1 ที่เพิ่งเสียบคืน)
**ยังอยู่ใน State INIT** ไม่ถึง FULL — เพราะ OSPF ต้องผ่านลำดับ Down→Init→2-Way→ExStart→Exchange
→Loading→Full ซึ่งใช้เวลาหลักวินาที**ไม่ใช่ทันที**

### Root Cause

```
Timeline ของปัญหา:

t=0s    Uplink เสียบคืน → Interface Up ทันที (Layer 1/2)
t=0-1s  DIST-SW1 เห็น Track Object กลับมา Up → Priority คืนจาก 90 เป็น 150 ทันที
        → HSRP Preempt ยึด Active คืนทันที (HSRP ไม่รอ Routing Protocol เลย)
t=0-1s  แต่ OSPF Neighbor ระหว่าง DIST-SW1↔CORE-SW1 ยังอยู่ระหว่าง Init→Full (ยังไม่ Full จนถึง
        t=8-10s โดยประมาณ ตาม Dead Timer/LSDB Sync ปกติของ OSPF)
t=0-10s ช่วงนี้ DIST-SW1 เป็น HSRP Active แล้ว (รับ Traffic จาก PC1 เข้ามา) แต่ยังไม่มี Route
        ที่สมบูรณ์ไปยัง VLAN30/40 ผ่าน Backbone เลย (Area 1 ↔ Area 0 ยังไม่ Sync LSDB เสร็จ)
        → Traffic จาก PC1 ที่วิ่งผ่าน DIST-SW1 ถูก **Black Hole ชั่วคราว** จนกว่า OSPF Full
```

**สรุป Root Cause**: HSRP `preempt` ทำงานเร็วกว่า OSPF Convergence มาก (HSRP วัดจาก Track
Interface ธรรมดา ไม่รอ Routing Protocol) ทำให้ DIST-SW1 ยึด Active คืนและรับ Traffic ก่อนที่
เส้นทาง Routing ทั้งระบบจะพร้อมจริง เกิดเป็น **Transient Black Hole** ที่ Fail-over ปกติไม่มีปัญหา
นี้เลย (เพราะตอน Fail-over DIST-SW2 ที่รับ Active ต่อ **มี Route พร้อมอยู่แล้วตั้งแต่ต้น** ไม่ต้องรอ
Convergence ใหม่)

### วิธีแก้: `standby <group> preempt delay minimum <seconds>`

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 preempt delay minimum 60
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# standby 20 preempt delay minimum 60
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface vlan 99
DIST-SW1(config-if)# standby 99 preempt delay minimum 60
DIST-SW1(config-if)# exit
```

คำสั่งนี้บอกให้ DIST-SW1 **รอ 60 วินาทีหลัง Interface/Track กลับมา Up ก่อนค่อย Preempt ยึด
Active คืน** — ให้เวลา OSPF Converge จนสมบูรณ์ (Full state + LSDB Sync ทั่ว Area) ก่อนที่ DIST-SW1
จะกลับมารับ Traffic จริง ตัวเลข 60 วินาทีควรตั้งให้มากกว่าเวลา Convergence จริงของ OSPF ใน Lab
นี้พอสมควร (วัดจริงด้วย `debug ip ospf adj` ก่อนกำหนดค่าถาวร)

```
! ยืนยันผลลัพธ์หลังแก้ไข
DIST-SW1# show standby vlan10
Vlan10 - Group 10
  ...
  Preempt enabled, delay min 60 secs
  Active router is 10.10.10.3, priority 100 (expires in 8.928 sec)   <-- ยังไม่ยึดคืนทันที รอครบ 60s ก่อน
```

> **บทเรียนสำหรับข้อสอบ**: `standby preempt` เพียวๆ (ไม่มี `delay minimum`) ปลอดภัยพอสำหรับ Lab
> เล็กที่ Convergence เร็ว แต่ใน Enterprise จริงที่มี Multi-Area OSPF/BGP ซับซ้อน **ควรใส่ `preempt
> delay minimum` เสมอทุกครั้งที่ใช้ Interface Tracking** เพื่อป้องกัน Transient Black Hole ที่เกิด
> จาก HSRP เร็วกว่า Routing Protocol Convergence — เป็นหัวข้อที่ข้อสอบ CCNA ชอบทดสอบด้วยคำถาม
> เชิงสถานการณ์ (Scenario-based) มากกว่าจำ Syntax เฉยๆ

---

## Step 247 — กับดักข้อสอบ CCNA ที่พบบ่อย + 5 คำถามฝึกฝน

### 247.1 Subnetting Speed-Solving Tips (ทวนจาก Part 7)

- จำ **Power of 2** ให้ขึ้นใจ (2,4,8,16,32,64,128,256) — ใช้แปลง CIDR ↔ จำนวน Host/Subnet ทันที
- เทคนิค **"Magic Number"**: Magic Number = 256 - Subnet Mask Octet สุดท้าย เช่น /27 → Magic
  Number = 256-224 = 32 → Subnet Boundary คือ 0,32,64,96,128,160,192,224 (นับทวีคูณ 32 ไปเรื่อยๆ)
- อ่านคำถาม "จำนวน Usable Host" อย่าลืมหัก 2 (Network + Broadcast) เสมอ ยกเว้นถามตรงว่า
  "จำนวน Address ทั้งหมดใน Subnet" (ไม่หัก)
- ข้อสอบชอบถามแบบ "IP นี้อยู่ Subnet เดียวกับ IP นี้หรือไม่" → **AND** IP กับ Subnet Mask แล้ว
  เทียบ Network Address ตรงๆ เร็วกว่าคำนวณ Range เต็ม

### 247.2 OSPF vs EIGRP Administrative Distance Tie-Break Trap (ทวนจาก Part 9, 13)

| Source ของ Route | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| EIGRP Summary Route | 5 |
| **eBGP** | 20 |
| **EIGRP (internal)** | **90** |
| **OSPF** | **110** |
| **IS-IS** | 115 |
| **RIP** | 120 |
| **EIGRP (external)** | 170 |
| iBGP | 200 |
| Unknown | 255 |

> **กับดักคลาสสิก**: ถ้า Router เรียนรู้ Subnet ปลายทางเดียวกันจากทั้ง OSPF และ EIGRP พร้อมกัน
> **EIGRP (AD 90) ชนะเสมอ** เพราะ AD ต่ำกว่า **ไม่เกี่ยวกับ Metric เลย** (AD เทียบกันก่อน Metric
> ทุกครั้ง — Metric ใช้เทียบเฉพาะเมื่อมาจาก Routing Protocol **เดียวกัน** เท่านั้น) ข้อสอบมักเขียน
> โจทย์ให้ EIGRP Metric แย่กว่า OSPF มาก แล้วถามว่า Router จะเลือกใช้ Path ไหน — คำตอบคือ
> **ยังเลือก EIGRP อยู่ดี** เพราะ AD ตัดสินก่อน Metric เสมอ

### 247.3 ACL Implicit Deny Trap (ทวนจาก Part 14, ดู Step 245 ด้วย)

> ทุก ACL (Standard/Extended) มี **`deny any` ที่มองไม่เห็น (Implicit) อยู่บรรทัดสุดท้ายเสมอ**
> แม้จะไม่ได้พิมพ์เอง — ถ้า Apply ACL ที่มีแต่ `permit` บางเงื่อนไข โดยไม่ปิดท้ายด้วย `permit ip
> any any` **Traffic อื่นทั้งหมดที่ไม่ตรงเงื่อนไขจะถูก Deny ทันทีอัตโนมัติ** ข้อสอบชอบโจทย์ประเภท
> "ACL มีแค่ 1 บรรทัด permit ... แล้ว Traffic อื่นที่ไม่ตรงจะเกิดอะไรขึ้น" → คำตอบคือ **ถูก Deny
> โดย Implicit Deny เสมอ** (ไม่ใช่ "ผ่านได้เพราะไม่มีเขียน deny ไว้ชัดเจน" ซึ่งเป็นตัวเลือกหลอกที่พบบ่อย)

### 247.4 5 คำถามฝึกฝนสไตล์ CCNA พร้อมเฉลยละเอียด

**คำถามที่ 1**
โจทย์: Host มี IP `192.168.10.70/27` Subnet ของ Host นี้คือ Subnet ใด และ Broadcast Address คือ?

<details>
<summary>เฉลย</summary>

/27 = 255.255.255.**224** → Magic Number = 32 → Boundary: 0,32,64,**96**,128...
70 อยู่ระหว่าง 64-95 → **Network = 192.168.10.64/27**, **Broadcast = 192.168.10.95**
</details>

**คำถามที่ 2**
โจทย์: Router มี Route ไปยัง `10.0.0.0/8` จาก 3 แหล่งพร้อมกัน: Static Route, OSPF (metric 20),
RIP (metric 2) — Router จะเลือกใช้ Route จากแหล่งใดเป็น Best Path?

<details>
<summary>เฉลย</summary>

**Static Route** เพราะ AD ของ Static = 1 ต่ำที่สุดในสามตัวเลือก (OSPF=110, RIP=120) — Metric
ของ OSPF/RIP ไม่มีผลใดๆเลยในการเปรียบเทียบข้าม Routing Source ที่ต่างกัน
</details>

**คำถามที่ 3**
โจทย์: Switch Port ตั้ง `switchport port-security violation restrict` ไว้ ถ้ามี MAC Address
ที่ไม่ได้รับอนุญาต (เกิน Maximum) ส่ง Frame เข้ามา จะเกิดอะไรขึ้นกับ Port นี้?

<details>
<summary>เฉลย</summary>

Port **ไม่ Shutdown** (ต่างจาก Mode `shutdown` ซึ่งเป็น Default) — Frame ที่ผิดกฎเพียง**ถูก Drop**
เท่านั้น พอร์ตยังเปิดใช้งานได้ปกติสำหรับ MAC ที่ได้รับอนุญาตอยู่แล้ว และ **จะเพิ่ม Security
Violation Counter** (ดูได้ด้วย `show port-security interface`) แต่ไม่สร้าง Syslog/ปิดพอร์ต
เหมือน Mode `shutdown`
</details>

**คำถามที่ 4**
โจทย์: ในสถาปัตยกรรม OSPF Multi-Area ต่อไปนี้ — Area 5 ต้องการติดต่อกับ Area 7 โดยตรง ไม่ผ่าน
Area 0 ได้หรือไม่? เพราะเหตุใด?

<details>
<summary>เฉลย</summary>

**ไม่ได้** — ตามกฎ Backbone Rule ของ OSPF, Area ที่ไม่ใช่ Area 0 ทุกคู่ **ต้องสื่อสารผ่าน Area 0
เท่านั้น** ห้าม Area-to-Area โดยตรงเด็ดขาด (ยกเว้นทำ Virtual Link ซึ่งเป็นเทคนิคเฉพาะกรณีที่ไม่มี
Physical Link ตรงไปยัง Area 0 — เรียนละเอียดใน CCNP ENARSI)
</details>

**คำถามที่ 5**
โจทย์: DHCP Client ในเครือข่ายที่เปิด **DHCP Snooping** แต่ยังไม่เปิด Trust บน Uplink ไปยัง
DHCP Server จะเกิดอะไรขึ้น?

<details>
<summary>เฉลย</summary>

Client **จะไม่ได้รับ IP Address เลย** เพราะ DHCP Snooping มองว่า Uplink เป็น **Untrusted Port
โดย Default** และจะ**บล็อก DHCP Reply (OFFER/ACK) ที่มาจากทิศทาง Untrusted** ทันที ต้องตั้ง
`ip dhcp snooping trust` บน Uplink ที่ต่อไปยัง DHCP Server (หรือ Relay Agent ที่ถูกต้อง) เสมอ
ก่อนใช้งานจริง
</details>

---

## Step 248 — Hands-on Lab Exam Simulation (จับเวลา 45 นาที)

### โจทย์ (อ่านให้ครบก่อนเริ่ม Config)

> ให้ Topology ย่อ 4 อุปกรณ์: `SW-CORE` (L3, เชื่อม `SW-DIST-A` และ `SW-DIST-B`), `SW-DIST-A`,
> `SW-DIST-B` (ทั้งคู่เป็น L3, ต่อ Access Switch คนละตัว), เชื่อมกันด้วย Routed Link ล้วน
> จงคอนฟิกตามข้อกำหนดต่อไปนี้:
>
> 1. สร้าง VLAN 100 (`USERS`, 172.16.100.0/24) และ VLAN 999 (`MGMT`, 172.16.999.0/24 — ใช้
>    third octet 200 จริงเพราะ VLAN ID ไม่เกิน 4094 แต่ IP กำหนดเป็น 172.16.200.0/24) บน
>    `SW-DIST-A` และ `SW-DIST-B`
> 2. Trunk ระหว่าง Access Switch ↔ SW-DIST-A/B พก VLAN 100,200 เท่านั้น (Native VLAN = 999
>    ไม่มีจริงในโจทย์นี้ ใช้ Native = default VLAN1 ตามปกติ)
> 3. เปิด OSPF Area 0 เดียวทั้งระบบ, Router-ID ตรงกับ Loopback0 (SW-CORE=9.9.9.9,
>    SW-DIST-A=9.9.9.11, SW-DIST-B=9.9.9.12), `passive-interface default` ทุกตัว
> 4. ตั้ง HSRP VLAN100 บน SW-DIST-A/B: VIP = 172.16.100.1, SW-DIST-A เป็น Active (Priority 150),
>    SW-DIST-B Standby, ทั้งคู่ต้อง `preempt`
> 5. สร้าง Extended ACL ชื่อ `USERS-OUTBOUND` บน SW-DIST-A: permit เฉพาะ TCP port 80/443 จาก
>    VLAN100 ไปยังปลายทางใดๆ, deny ip อื่นทั้งหมด (ไม่ต้อง log), Apply "in" ที่ SVI VLAN100

### Model Answer — SW-DIST-A

```
hostname SW-DIST-A
!
ip routing
!
vlan 100
 name USERS
vlan 200
 name MGMT
!
interface Loopback0
 ip address 9.9.9.11 255.255.255.255
!
interface Vlan100
 description ** Gateway VLAN100-USERS **
 ip address 172.16.100.2 255.255.255.0
 standby version 2
 standby 100 ip 172.16.100.1
 standby 100 priority 150
 standby 100 preempt
 no shutdown
!
interface Vlan200
 description ** Gateway VLAN200-MGMT **
 ip address 172.16.200.2 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to Access Switch **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,200
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to SW-CORE **
 no switchport
 ip address 172.16.254.2 255.255.255.252
 no shutdown
!
ip access-list extended USERS-OUTBOUND
 permit tcp 172.16.100.0 0.0.0.255 any eq 80
 permit tcp 172.16.100.0 0.0.0.255 any eq 443
 deny ip any any
!
interface Vlan100
 ip access-group USERS-OUTBOUND in
!
router ospf 1
 router-id 9.9.9.11
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
 network 172.16.100.0 0.0.0.255 area 0
 network 172.16.200.0 0.0.0.255 area 0
 network 172.16.254.0 0.0.0.3 area 0
!
end
```

### Model Answer — SW-DIST-B (ต่างเฉพาะ Priority/IP)

```
hostname SW-DIST-B
!
ip routing
vlan 100
 name USERS
vlan 200
 name MGMT
!
interface Loopback0
 ip address 9.9.9.12 255.255.255.255
!
interface Vlan100
 description ** Gateway VLAN100-USERS **
 ip address 172.16.100.3 255.255.255.0
 standby version 2
 standby 100 ip 172.16.100.1
 standby 100 priority 100
 standby 100 preempt
 no shutdown
!
interface Vlan200
 ip address 172.16.200.3 255.255.255.0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to Access Switch **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,200
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to SW-CORE **
 no switchport
 ip address 172.16.254.6 255.255.255.252
 no shutdown
!
router ospf 1
 router-id 9.9.9.12
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
 network 172.16.100.0 0.0.0.255 area 0
 network 172.16.200.0 0.0.0.255 area 0
 network 172.16.254.4 0.0.0.3 area 0
!
end
```

### Model Answer — SW-CORE (สรุปย่อ)

```
hostname SW-CORE
ip routing
interface Loopback0
 ip address 9.9.9.9 255.255.255.255
interface TenGigabitEthernet1/0/1
 description ** Routed link to SW-DIST-A **
 no switchport
 ip address 172.16.254.1 255.255.255.252
 no shutdown
interface TenGigabitEthernet1/0/2
 description ** Routed link to SW-DIST-B **
 no switchport
 ip address 172.16.254.5 255.255.255.252
 no shutdown
router ospf 1
 router-id 9.9.9.9
 passive-interface default
 no passive-interface TenGigabitEthernet1/0/1
 no passive-interface TenGigabitEthernet1/0/2
 network 172.16.254.0 0.0.0.3 area 0
 network 172.16.254.4 0.0.0.3 area 0
end
```

### เกณฑ์ตรวจให้คะแนน (Self-Check)

| ข้อกำหนด | วิธี Verify |
|---|---|
| VLAN/Trunk ถูกต้อง | `show vlan brief`, `show interfaces trunk` |
| OSPF Neighbor ขึ้น FULL ทั้งสองคู่ | `show ip ospf neighbor` |
| Router-ID ตรงกับ Loopback0 ที่กำหนด | `show ip ospf` (บรรทัดแรก) |
| HSRP: SW-DIST-A เป็น Active, Priority ถูก | `show standby brief` |
| ACL บล็อก Traffic อื่นที่ไม่ใช่ 80/443 จาก VLAN100 จริง | `show ip access-lists USERS-OUTBOUND` (match counter เพิ่ม) |

---

## Step 249 — เคล็ดลับวันสอบจริง + แผนฝึกต่อเนื่อง

### 249.1 การบริหารเวลาในห้องสอบ (Time Management)

- ข้อสอบ CCNA 200-301 มีเวลา **120 นาที** ประมาณ 90-110 ข้อ (สัดส่วนจริงแตกต่างกันไปตามรอบสอบ)
  เฉลี่ยประมาณ **1-1.3 นาทีต่อข้อ** — **ห้ามติดอยู่ข้อเดียวนานเกิน 2 นาที** ถ้าไม่แน่ใจให้ Flag
  ไว้ ("Mark for review") แล้วข้ามไปก่อนเสมอ
- **Simulation Question (Lab จำลอง)** ใช้เวลานานกว่าข้อ Multiple Choice มาก (5-10 นาทีต่อข้อ)
  ควรทำข้อ Multiple Choice ให้เร็วที่สุดก่อน เพื่อเก็บเวลาไว้ให้ Simulation ที่มีน้ำหนักคะแนนสูง
- อ่านคำถามให้ครบ**ก่อน**ดู Choice เสมอ — Cisco มักใส่คำว่า "**NOT**", "**EXCEPT**",
  "**BEST**" ซ่อนอยู่ในโจทย์ ซึ่งเปลี่ยนความหมายคำถามทั้งหมด

### 249.2 กลยุทธ์ทำข้อสอบ Simulation (Lab จำลองในห้องสอบ)

1. **อ่านโจทย์ทั้งหมดก่อนพิมพ์คำสั่งแรก** — Sim มักมี Requirement ซ่อนอยู่หลายข้อในกำแพงข้อความ
2. ใช้ `show running-config` ตรวจสอบ Baseline ของอุปกรณ์ก่อนเริ่มแก้ไขเสมอ (บาง Sim มี Config
   เดิมผิดที่ต้องแก้ ไม่ใช่ Config จากเครื่องเปล่า)
3. พิมพ์คำสั่ง `end` และ `copy running-config startup-config` ทุกครั้งที่ทำ Requirement หนึ่ง
   เสร็จ (กัน Session Timeout หรือ Sim ค้างแล้วเสีย Config ที่ทำไปแล้ว)
4. ถ้า Sim ให้คะแนนบางส่วน (Partial Credit) — ทำ Requirement ที่มั่นใจให้ครบก่อน อย่าเสียเวลา
   กับ Requirement ที่ไม่แน่ใจจนกระทบ Requirement อื่นที่ทำได้แน่นอน

### 249.3 หัวข้อที่ควรทวนก่อนสอบวันสุดท้าย (High-Yield Topics)

- Subnetting/VLSM Speed-solving (Part 7) — ยังคงเป็นพื้นฐานของโจทย์แทบทุกหมวด
- OSPF Neighbor States, DR/BDR Election, LSA Type 1-5 (Part 11-12)
- HSRP/VRRP/GLBP เปรียบเทียบ Syntax และ State Machine (Part 18)
- ACL Wildcard Mask การคำนวณ และ Implicit Deny (Part 14, Step 247.3)
- Port Security Violation Mode ทั้ง 3 แบบ (Protect/Restrict/Shutdown) ต่างกันอย่างไร (Part 21)
- AAA Authentication vs Authorization vs Accounting ต่างกันอย่างไร (Part 22)

### 249.4 แผนฝึกต่อเนื่องหลังจบ Part นี้ (ก่อนไป Part 26)

> **คำแนะนำที่สำคัญที่สุด**: อย่าอ่าน Part 1-24 ซ้ำเฉยๆ — ให้**สร้าง Lab นี้ขึ้นมาใหม่ทั้งหมดด้วย
> ตัวเองใน Packet Tracer หรือ Cisco Modeling Labs (CML) โดยไม่เปิดหลักสูตรนี้ดูเลย** แล้วค่อยกลับมา
> เทียบกับ Final Running-Config ใน Step 250 ว่าตรงกันหรือไม่ — นี่คือวิธีฝึกที่สร้าง "ความจำกล้ามเนื้อ
> (Muscle Memory)" ของ CLI ได้จริงที่สุด ดีกว่าอ่านซ้ำหลายรอบมาก
>
> ทำซ้ำ Lab เต็มรูปแบบนี้อย่างน้อย **2-3 รอบ** ก่อนเข้าสอบจริง โดยรอบหลังๆให้จับเวลาตัวเองด้วย
> (เทียบเวลาที่ใช้ Bring-up ทั้ง Topology ให้ใช้งานได้ ควรลดลงทุกรอบ)

---

## Step 250 — Final Running-Config Appendix (CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2)

นี่คือ **Running-Config ฉบับสมบูรณ์ที่สุด** ของ 4 อุปกรณ์ตัวแทน หลังรวมทุก Feature จาก Part 1-24
เข้าด้วยกัน — ใช้เป็นเอกสารอ้างอิงหลักเวลาทวนทั้งหลักสูตร หรือเทียบกับ Lab ที่สร้างเองตาม Step 249.4

> **หมายเหตุ**: Config ชุดนี้เป็น "Steady State สุดท้าย" หลัง Part 24 สมบูรณ์แล้ว — ตัด Static
> Default Route ของ Part 9 ออกแล้ว (ถูกแทนที่ด้วย OSPF), เปลี่ยน Gateway จาก `.2/.3` เป็น HSRP
> VIP `.1` แล้ว (Part 18), และรวม ACL/Port Security/AAA/QoS ของ Part 14, 21, 22, 24 ครบทุกตัว

### CORE-SW1 (Catalyst 9500, ABR: Area 0 + Area 1)

```
hostname CORE-SW1
!
no ip domain-lookup
ip domain-name lab.local
enable secret Cisco123!
service password-encryption
!
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
!
username admin privilege 15 secret StrongP@ssw0rd!
tacacs server AAA-SRV
 address ipv4 10.10.99.51
 key CCNA-LAB-KEY
!
banner motd #
========================================================
  Enterprise Lab - Cisco CCNA/CCNP/CCIE Training Course
  Unauthorized access is prohibited.
========================================================
#
!
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
spanning-tree mode rapid-pvst
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.1 255.255.255.255
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 (Area 0) **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf 1 area 0
 ip ospf message-digest-key 1 md5 Ar3aZeroKey!
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Po1 member - to CORE-SW2 **
 no switchport
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Po1 member - to CORE-SW2 **
 no switchport
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to DIST-SW1 (Area 1) **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 no shutdown
!
interface TenGigabitEthernet1/1/2
 description ** Routed uplink to DIST-SW2 (Area 1) **
 no switchport
 ip address 10.255.20.1 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 no shutdown
!
mls qos
mls qos trust dscp
!
ntp server 10.10.99.50
logging host 10.10.99.50
logging trap informational
snmp-server host 10.10.99.50 version 3 priv noc-svc
!
router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 1 range 10.10.0.0 255.255.0.0
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface TenGigabitEthernet1/1/1
 no passive-interface TenGigabitEthernet1/1/2
!
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny any log
!
line console 0
 password Cisco123!
 login authentication default
 exec-timeout 10 0
 logging synchronous
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login authentication default
 exec-timeout 10 0
!
no cdp run
lldp run
!
end
```

### CORE-SW2 (Catalyst 9500, ABR: Area 0 + Area 2)

```
hostname CORE-SW2
!
no ip domain-lookup
ip domain-name lab.local
enable secret Cisco123!
service password-encryption
!
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
!
username admin privilege 15 secret StrongP@ssw0rd!
tacacs server AAA-SRV
 address ipv4 10.10.99.51
 key CCNA-LAB-KEY
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.2 255.255.255.255
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW1 (Area 0) **
 no switchport
 ip address 10.255.0.2 255.255.255.252
 ip ospf 1 area 0
 ip ospf message-digest-key 1 md5 Ar3aZeroKey!
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Po1 member - to CORE-SW1 **
 no switchport
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Po1 member - to CORE-SW1 **
 no switchport
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to DIST-SW3 (Area 2) **
 no switchport
 ip address 10.255.30.1 255.255.255.252
 ip ospf 1 area 2
 ip ospf message-digest-key 1 md5 Area2Key!
 no shutdown
!
interface TenGigabitEthernet1/1/2
 description ** Routed uplink to DIST-SW4 (Area 2) **
 no switchport
 ip address 10.255.40.1 255.255.255.252
 ip ospf 1 area 2
 ip ospf message-digest-key 1 md5 Area2Key!
 no shutdown
!
mls qos
mls qos trust dscp
!
ntp server 10.10.99.50
logging host 10.10.99.50
snmp-server host 10.10.99.50 version 3 priv noc-svc
!
router ospf 1
 router-id 1.1.1.2
 area 0 authentication message-digest
 area 2 authentication message-digest
 area 2 range 10.10.0.0 255.255.0.0
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface TenGigabitEthernet1/1/1
 no passive-interface TenGigabitEthernet1/1/2
!
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny any log
!
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login authentication default
 exec-timeout 10 0
!
no cdp run
lldp run
!
end
```

### DIST-SW1 (Catalyst 9300, Internal Router Area 1, HSRP Active VLAN10/99)

```
hostname DIST-SW1
!
no ip domain-lookup
ip domain-name lab.local
enable secret Cisco123!
service password-encryption
!
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
!
username admin privilege 15 secret StrongP@ssw0rd!
tacacs server AAA-SRV
 address ipv4 10.10.99.51
 key CCNA-LAB-KEY
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
spanning-tree vlan 10,20,99 root primary
!
ip dhcp snooping
ip dhcp snooping vlan 10,20,99
ip arp inspection vlan 10,20,99
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.11 255.255.255.255
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ip helper-address 10.10.30.20
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 track TenGigabitEthernet1/1/1 decrement 60
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
 ip helper-address 10.10.30.20
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 100
 standby 20 preempt delay minimum 60
 standby 20 track TenGigabitEthernet1/1/1 decrement 60
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
 standby version 2
 standby 99 ip 10.10.99.1
 standby 99 priority 150
 standby 99 preempt delay minimum 60
 standby 99 track TenGigabitEthernet1/1/1 decrement 60
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW1 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW2 (redundant, STP handles path) **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW1 (Area 1) **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 no shutdown
!
mls qos
mls qos trust dscp
auto qos voip trust
!
ntp server 10.10.99.50
logging host 10.10.99.50
!
router ospf 1
 router-id 1.1.1.11
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny any log
!
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login authentication default
 exec-timeout 10 0
!
no cdp run
lldp run
!
end
```

### DIST-SW2 (Catalyst 9300, Internal Router Area 1, HSRP Active VLAN20 / Standby VLAN10/99)

```
hostname DIST-SW2
!
no ip domain-lookup
ip domain-name lab.local
enable secret Cisco123!
service password-encryption
!
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
!
username admin privilege 15 secret StrongP@ssw0rd!
tacacs server AAA-SRV
 address ipv4 10.10.99.51
 key CCNA-LAB-KEY
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
spanning-tree vlan 10,20,99 root secondary
!
ip dhcp snooping
ip dhcp snooping vlan 10,20,99
ip arp inspection vlan 10,20,99
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.12 255.255.255.255
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.3 255.255.255.0
 ip helper-address 10.10.30.20
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 100
 standby 10 preempt delay minimum 60
 standby 10 track TenGigabitEthernet1/1/1 decrement 60
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.3 255.255.255.0
 ip helper-address 10.10.30.20
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 150
 standby 20 preempt delay minimum 60
 standby 20 track TenGigabitEthernet1/1/1 decrement 60
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.3 255.255.255.0
 standby version 2
 standby 99 ip 10.10.99.1
 standby 99 priority 100
 standby 99 preempt delay minimum 60
 standby 99 track TenGigabitEthernet1/1/1 decrement 60
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW1 (redundant, STP handles path) **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW2 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW1 (Area 1) **
 no switchport
 ip address 10.255.20.2 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 no shutdown
!
mls qos
mls qos trust dscp
auto qos voip trust
!
ntp server 10.10.99.50
logging host 10.10.99.50
!
router ospf 1
 router-id 1.1.1.12
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny any log
!
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login authentication default
 exec-timeout 10 0
!
no cdp run
lldp run
!
end
```

> **หมายเหตุปิด Appendix**: DIST-SW3 และ DIST-SW4 ใช้โครงสร้างเดียวกันทุกประการกับ DIST-SW1/DIST-SW2
> ข้างต้น เพียงเปลี่ยน VLAN เป็น 30(SERVERS)/40(WIFI)/99(MGMT), เปลี่ยน Area เป็น `area 2`,
> เปลี่ยน MD5 Key เป็น `Area2Key!`, เปลี่ยน Loopback0 เป็น `1.1.1.13`/`1.1.1.14`, และสลับ Priority
> HSRP ตามรูปแบบเดียวกับ DIST-SW1/DIST-SW2 (Active/Standby สลับกันคนละ VLAN) — ตาม Pattern ที่
> วางไว้ตั้งแต่ Step 241

---

## แบบฝึกหัดทวนความเข้าใจ Part 25

1. ในสถาปัตยกรรม Routed Core ของ Lab นี้ (ตั้งแต่ Part 10) เพราะเหตุใด STP Root Bridge ของ VLAN
   ผู้ใช้งานทุกใบจึงย้ายไปอยู่ที่ Distribution Layer แทน Core Layer?
2. ตามลำดับ Runbook ใน Step 244 เพราะเหตุใดต้องเปิด Port Security ก่อน DHCP Snooping และเปิด
   DHCP Snooping ก่อน Dynamic ARP Inspection เสมอ ห้ามสลับลำดับ?
3. ใน Troubleshooting Scenario 1 (Step 245) เพราะเหตุใด PC1 (VLAN10) เปิด HTTPS ไป Server1 ได้
   ปกติ แต่ ping (ICMP) ไม่ผ่าน ทั้งที่เป็น Path เดียวกันและ ACL เดียวกัน?
4. ใน Troubleshooting Scenario 2 (Step 246) คำสั่งใดที่แก้ปัญหา Transient Black Hole ตอน HSRP
   Fail-back และคำสั่งนั้นทำงานอย่างไร?
5. ถ้า Router เรียนรู้ Subnet ปลายทางเดียวกันจากทั้ง Static Route, OSPF, และ EIGRP พร้อมกัน
   Router จะเลือกใช้ Route จากแหล่งใด และเพราะเหตุใด (ไม่ต้องดู Metric)?

**เฉลย:**
1. เพราะ Core (CORE-SW1/CORE-SW2) เปลี่ยนเป็น Routed Port ล้วนตั้งแต่ Part 10 (ไม่มี VLAN ใด
   Trunk ข้าม Core Layer เลย) ทำให้ Core ไม่มีส่วนร่วมใน STP Topology ของ VLAN ผู้ใช้งานอีกต่อไป
   Root Bridge จึงตกไปอยู่ที่ Distribution Layer โดยธรรมชาติ (DIST-SW1/DIST-SW3 ตามที่ตั้ง Priority
   ต่ำสุดไว้)
2. Port Security ทำงานอิสระไม่พึ่งพา Snooping Binding Table จึงทำก่อนได้ทันที ส่วน Dynamic ARP
   Inspection ต้องใช้ DHCP Snooping Binding Table เป็นฐานข้อมูลตรวจสอบ ARP — ถ้าเปิด DAI ก่อนที่
   Binding Table จะมีข้อมูลสมบูรณ์ Host ที่ได้ IP ไปแล้วจะ ARP ไม่ผ่านทันที
3. เพราะ ACL `SALES-TO-SERVERS` อนุญาตเฉพาะ TCP port 443/22 จาก SALES ไปยัง Server1 (บรรทัด
   permit เฉพาะ Protocol/Port นั้น) ส่วน ICMP ไม่ตรงกับบรรทัด permit ใดเลย จึงตกไปโดนบรรทัด
   `deny ip ... log` ที่ดักจับ Protocol IP ทั้งหมดที่เหลือ
4. `standby <group> preempt delay minimum <seconds>` — คำสั่งนี้ทำให้ Router ที่ Priority สูงกว่า
   รอเวลาที่กำหนดหลัง Track Interface กลับมา Up ก่อนค่อย Preempt ยึด Active คืน เปิดโอกาสให้ OSPF
   Converge จน Full/LSDB Sync สมบูรณ์ก่อนที่ Router นั้นจะกลับมารับ Traffic จริง ป้องกัน Black Hole
5. **Static Route** เพราะ Administrative Distance ของ Static = 1 ต่ำที่สุดในสามตัวเลือก (EIGRP=90,
   OSPF=110) — AD เป็นเกณฑ์แรกที่ใช้ตัดสินเสมอ ไม่เกี่ยวกับ Metric เลย (Metric ใช้เทียบเฉพาะ
   Route ที่มาจาก Routing Protocol เดียวกันเท่านั้น)

---

## จบระดับ CCNA! 🎓

**ยินดีด้วย — คุณผ่าน Part 1 ถึง Part 25 ครบทั้ง 250 Step ของระดับ CCNA (Cisco 200-301) แล้ว**

### สิ่งที่สร้างและเรียนรู้มาตลอด 25 Part

ตลอดหลักสูตรที่ผ่านมา เราสร้าง **Enterprise Lab Topology เดียวกัน** ต่อเนื่องตั้งแต่ Part 1 และ
ขยายเพิ่มความสามารถทีละ Part จนกลายเป็นเครือข่ายที่มีองค์ประกอบครบทุกด้านของ Enterprise จริง:

- **Layer 1-2**: Ethernet, VLAN, Trunk, STP/RSTP, EtherChannel (Part 1, 3-6)
- **Layer 3**: IPv4/IPv6 Addressing, Static Route, Inter-VLAN Routing, OSPF Multi-Area (Part 7-12)
- **Routing Protocol เปรียบเทียบ**: EIGRP เพื่อความเข้าใจ AD/Metric ควบคู่กับ OSPF (Part 13)
- **Security**: ACL, Port Security, DHCP Snooping, DAI, AAA (Part 14, 21-22)
- **Services**: NAT/PAT, DHCP/DNS, NTP/Syslog/SNMP, WAN Technologies (Part 15-17, 19)
- **High Availability**: HSRP/FHRP พร้อม Interface Tracking (Part 18)
- **Wireless**: WLC/AP Fundamentals (Part 20)
- **Automation**: NETCONF/RESTCONF/Ansible เบื้องต้น (Part 23)
- **QoS**: Trust Boundary, Auto QoS (Part 24)
- **Capstone**: รวมทุกอย่างเป็น Runbook, Troubleshooting Methodology, และ Exam Readiness (Part 25)

นี่คือพื้นฐานที่**ครบถ้วนและลึกพอ**สำหรับการสอบ **Cisco CCNA 200-301** จริง และสำคัญยิ่งกว่านั้น
คือเป็นพื้นฐานที่วิศวกรเครือข่าย Enterprise ใช้ทำงานจริงทุกวัน — ไม่ใช่แค่ท่องจำเพื่อสอบผ่าน

### ก้าวต่อไป: ระดับ CCNP ENCOR (350-401)

หลักสูตรจะดำเนินต่อเข้าสู่ **ระดับ CCNP Enterprise Core (ENCOR 350-401)** ตั้งแต่ **Part 26**
เป็นต้นไป โดย Lab Topology เดียวกันนี้จะถูก**ขยายให้ซับซ้อนขึ้นอีกขั้น** — เพิ่ม Dual-homed
DIST↔Core Redundancy, BGP, Route Redistribution, SD-Access, SD-WAN, Advanced QoS, Multicast,
และ Automation เต็มรูปแบบ

**Part 26 — Advanced OSPF (Multi-area, LSA Type 1-7)** จะไม่ใช่การทวน Part 12 ซ้ำ แต่เป็นการ
เจาะลึกในระดับ CCNP: การออกแบบ Multi-Area OSPF ขนาดใหญ่ที่มีมากกว่า 2 Area ที่ไม่ใช่ Area 0,
Virtual Link, LSA Type 6/7 แบบละเอียด, Route Filtering ด้วย Distribute-list/Prefix-list บน ABR,
และ Advanced Troubleshooting ของ LSDB Synchronization ที่ Scale ระดับ Enterprise จริง — ใช้ความรู้
พื้นฐาน OSPF จาก Part 11-12 เป็นฐาน แต่ไปในเชิงลึกและซับซ้อนกว่ามาก

**ไปต่อ:** [Part 26 — Advanced OSPF (Multi-area, LSA Type 1-7) →](part-026-advanced-ospf.md)
(จุดเริ่มต้นของระดับ CCNP ENCOR)
