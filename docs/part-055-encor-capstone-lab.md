# Part 55 — ENCOR Capstone Lab & Exam Prep
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 541–550 จาก 1000**

> ต่อจาก [Part 54 — Network Assurance & Telemetry](part-054-network-assurance-telemetry.md) นี่คือ
> **Part สุดท้ายของระดับ CCNP ENCOR (350-401)** — เช่นเดียวกับ [Part 25](part-025-ccna-capstone-lab.md)
> ที่ปิดระดับ CCNA, Part นี้**ไม่มีเทคโนโลยีใหม่**ให้เรียน แต่เป็น **Capstone Lab** ที่รวบรวม
> ทุก Feature ตั้งแต่ Part 26 ถึง Part 54 (30 Part เต็ม, Step 251-540) เข้าเป็นภาพเดียว, ฝึกลำดับ
> การ Bring-up จริงของทั้ง Enterprise Campus + WAN Edge, ฝึก Troubleshoot ข้าม Layer แบบที่ข้อสอบ
> จริงชอบออก, และเตรียมสอบ **Cisco 350-401 ENCOR** ให้พร้อมที่สุด ปิดท้ายด้วย Final Running-Config
> Appendix ที่เป็นเอกสารอ้างอิงสมบูรณ์ของทั้งระดับ CCNP ENCOR

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 541 | Full Topology Recap — ภาพ Enterprise Lab ฉบับสมบูรณ์ที่สุด (Part 1-54 ซ้อนกันทุก Layer) |
| 542 | ENCOR 350-401 Exam Blueprint Mapping — Domain ไหนอยู่ Part ไหน พร้อมน้ำหนักคะแนนจริง |
| 543 | Capstone Integration Checklist — Master Build-Order Runbook ของ ENCOR Feature ทั้งหมด |
| 544 | Troubleshooting Scenario 1 — BGP Redistribute เข้า OSPF ผิด ทำให้ Remote Site เสีย Internet Failover |
| 545 | Troubleshooting Scenario 2 — DMVPN Spoke-to-Spoke วิ่งผ่าน Hub (Hairpin) ไม่ใช้ Phase 3 Shortcut |
| 546 | กับดักข้อสอบ ENCOR ที่พบบ่อย + 5 คำถามฝึกฝนพร้อมเฉลยละเอียด |
| 547 | Hands-on Lab Exam Simulation — BGP+Redistribution+QoS+Multicast+Security รวมในโจทย์เดียว |
| 548 | เคล็ดลับวันสอบ ENCOR + แผนฝึกต่อเนื่อง |
| 549 | Career Context — CCNP ENCOR ปลดล็อกอะไรได้บ้าง และก้าวต่อไปคือทางไหน |
| 550 | Final Running-Config Appendix — CORE-SW1, DIST-SW1 ฉบับสมบูรณ์ที่สุดของระดับ ENCOR |

---

## Step 541 — Full Topology Recap: ภาพ Enterprise Lab ฉบับสมบูรณ์ที่สุด (Part 1-54)

ก่อนสอบและก่อนลง Lab Scenario ต่อไปในนี้ ต้อง**หยุดแล้วมองภาพรวมทั้งหมดอีกครั้ง** — Diagram นี้คือ
ภาพที่สมบูรณ์ที่สุดในหลักสูตรทั้งหมดจนถึงจุดนี้ ต่อยอดจาก Diagram ของ [Part 25 Step 241]
(part-025-ccna-capstone-lab.md) โดยซ้อนทุก Layer/Technology ของระดับ CCNP ENCOR (Part 26-54)
เข้าไปบน Enterprise Lab เดิม

```
                                        ┌───────────────────────┐
                                        │   INTERNET (ISP1/ISP2)  │
                                        └──┬─────────────────┬───┘
                          AS 65000        │                 │        AS 65002
                    ┌──────────┴──────┐  ┌──────────┴──────┐
                    │   ISP-RTR (ISP1) │  │  ISP2-RTR (ISP2) │
                    └────────┬─────────┘  └────────┬─────────┘
             203.0.113.0/30 │                       │ 203.0.113.4/30
   DMVPN Hub1/NHRP ┌────────┴────────┐  mGRE Tu0 ┌──┴──────────────┐ DMVPN Hub2/NHRP
   BGP eBGP AS65000│   WAN-EDGE-1    │═══════════│    WAN-EDGE-2    │BGP eBGP AS65002
   ZBFW INSIDE/OUT │ cEdge (SD-WAN)  │10.10.254.8/30│  cEdge (SD-WAN) │ZBFW INSIDE/OUT
   default-info    │ VRF: GLOBAL     │  cross-link │  VRF: GLOBAL    │default-info
   originate always│ GRE+IPsec→BRANCH│            │                  │originate always
   metric 1 type1  │ 1.1.1.21        │            │  1.1.1.22        │metric 5 type1
                    └───┬─────────┬───┘            └────┬─────────┬──┘
        10.10.254.0/30 │         │ Tunnel0 mGRE 172.16.220.1  │   │10.10.254.4/30
                        │         └──────────────┐  ┌─────────┘   │
                        │                        │  │Tunnel0 mGRE 172.16.220.2
                        │           ┌─────────────┴──┴──────┐
                        │           │  DMVPN Phase 3 Cloud   │  NHRP net-id 1
                        │           │  BRANCH-RTR (.11)      │  Spoke-to-Spoke Shortcut
                        │           │  BRANCH-RTR-2 (.12)    │  (Part 52-53)
                        │           └────────────────────────┘
                        │
   ═══════ OSPF Area 0 (Backbone) + BGP iBGP AS 65001 + BFD + CoPP ═══════════
                        ┌───────────────────────────────────────┐
                        │  CORE-SW1 ⇄ CORE-SW2  **StackWise Virtual (SVL Domain 100)**  │
                        │  ครั้งนี้เป็น "Logical Switch เดียว" ไม่ใช่ 2 Chassis แยกกันอีกต่อไป │
                        │  1.1.1.1/1.1.1.2  Po1(SVL, 2x40G) 10.255.0.0/30  ABR A0+A1/A2      │
                        │  iBGP AS65001 peer WAN-EDGE-1/2  Anycast RP 1.1.1.100 + MSDP       │
                        │  CoPP: COPP-CRITICAL/IMPORTANT/NORMAL  gRPC Telemetry → NOC-SRV    │
                        └───┬───────────────────────────────────────────────────────┬───┘
        10.255.10.0/30 MEC │ Po1(spans SVL pair)              MEC │ Po1(spans SVL pair) 10.255.30.0/30
   ══ OSPF Area 1 + OSPFv3 │═══════════════════════════════════════│══ OSPF Area 2 (NSSA) + OSPFv3 ══
        (Dual-homed MEC — ทุก DIST ต่อทั้ง 2 Chassis ของ SVL Pair พร้อมกัน ตั้งแต่ Part 33-34)
                        ┌───┴───┐                              ┌───┴───┐
                        │DIST-SW1│◄══HSRP(2 Track Obj)/STP══►│DIST-SW3│  ★ VRF-Lite: PARTNER-VRF
                        │1.1.1.11│  PBR (SLA-tracked next-hop) │1.1.1.13│  ★ EIGRP↔OSPF Mutual
                        │VLAN10 Active/VLAN20 Standby/VLAN99   │VLAN30 Active/VLAN40 Standby   Redistribution (route-map+tag,
                        │Fabric Edge Node (SD-Access overlay)  │Fabric Edge Node, SGACL Enforce Part 31, ที่นี่กับ DIST-SW4)
                        └──┬──────┬──┘        (คู่กับ)        └──┬──────┬──┘
                   Trunk│      │Trunk   ┌───┴───┐    ┌───┴───┐  Trunk│      │Trunk
                        │      └────────┤DIST-SW2│    │DIST-SW4├──────┘      │
                     ┌──┴───┐        └──┬──────┬──┘    └──┬──────┬──┘      ┌──┴───┐
                     │ACCESS│           │      │           │      │        │      │
                     │ SW1  │           │ACCESS│           │ACCESS│        │      │
                     │802.1X│           │ SW2  │           │ SW3  │        │      │
                     │IBNS2.0│          │IBNS2.0│          │+ SGT │        │      │
                     └──┬───┘           └──┬───┘           └──┬───┘        └──────┘
    dot1x+MAB│DHCP Snoop│DAI                │                  │ SXP→DIST-SW3          WLC-1 ⇄ AP-1/AP-2
                     PC1-PC2               PC3-PC5+Phone   Server1-3               Fabric WLC (SD-Access)
                   VLAN10(.10-.14)        VLAN10/20(.15-30) VLAN30(.10-.30)         CAPWAP + FlexConnect
                   SALES 10.10.10.0/24    VOICE 10.10.20.0/24 SERVERS 10.10.30.0/24 WIFI 10.10.40.0/24
                   TrustSec SGT: EMPLOYEE                                          802.1X/WPA2-Ent (Part37)

   MGMT VLAN99 (10.10.99.0/24): NOC-SRV .50(NTP/Syslog/SNMP/Ansible/Telemetry Collector) | AAA-SRV .51(TACACS+)
   ISE-1(PAN+MnT) .65 | ISE-2(PSN) .66 | FTD-1 (NGFW, ตรวจ Traffic เข้า-ออก WAN Edge) | DNAC-VM .60 (Catalyst Center)
```

### สรุปสิ่งที่ประกอบเข้าเป็น Topology นี้ — Mapping ทุก Layer/Feature ของระดับ ENCOR ไปยัง Part ที่สอน

| Layer/Feature | อุปกรณ์ที่เกี่ยวข้อง | สอนละเอียดที่ |
|---|---|---|
| Advanced OSPF: NSSA (Area2), BFD, Network Type ทางเลือก, Fast-Hello | CORE-SW1/2 (ABR), DIST-SW3/4 (NSSA), WAN-EDGE-1/2 | Part 26 |
| OSPFv3 for IPv6 (Per-link, ไม่มี `network` statement) | ทุก L3 Device (คู่กับ IPv6 ULA เดิมจาก Part 8) | Part 27 |
| EIGRP Advanced (Named Mode, Summarization) — Demo Site แยก | Lab สาธิตแยก (ไม่ใช่ Production Topology นี้โดยตรง) | Part 28 |
| BGP Fundamentals (eBGP): WAN-EDGE-1↔ISP1(AS65000), WAN-EDGE-2↔ISP2(AS65002) | WAN-EDGE-1/2, ISP-RTR | Part 29 |
| BGP Advanced: iBGP AS65001 (WAN-EDGE-1/2↔CORE-SW1/2), Attributes, Path Selection | WAN-EDGE-1/2, CORE-SW1/2 | Part 30 |
| Redistribution: `default-information originate` (BGP→OSPF, Default เท่านั้น), EIGRP↔OSPF Mutual (route-map+tag) | WAN-EDGE-1/2 (BGP→OSPF), DIST-SW3/4 (EIGRP↔OSPF) | Part 31 |
| Policy-Based Routing: `set ip next-hop verify-availability` + IP SLA Track | DIST-SW1/2 | Part 32 |
| Advanced STP & Campus Multilayer Design | ทุก Switch (พื้นฐานของ MEC ด้านล่าง) | Part 33 |
| StackWise Virtual (SVL Domain 100) + MEC (Dual-homed DIST↔CORE) | CORE-SW1/2 (SVL Pair), DIST-SW1-4 (MEC Uplink) | Part 34 |
| Advanced HSRP: หลาย Track Object, Preempt Delay | DIST-SW1-4 | Part 35 |
| Wireless Architecture Deep Dive: CAPWAP, Roaming | WLC-1, AP-1/2 | Part 36 |
| Wireless Security Advanced: 802.1X/WPA2-3-Enterprise บน WLAN | WLC-1, ISE-1/2 | Part 37 |
| SD-Access Design Overlay: Fabric Edge/Border/Control-Plane Node, LISP/VXLAN (Conceptual) | CORE-SW1/2(Border+CP), DIST-SW1-4(Edge), WLC-1(Fabric WLC) | Part 38 |
| SD-WAN Overlay: cEdge, vManage/vSmart/vBond (Conceptual/Architectural) | WAN-EDGE-1/2, BRANCH-RTR/-2 | Part 39 |
| Advanced QoS: MQC 3-Class (VOICE/VIDEO/CRITICAL-DATA), LLQ, Shaping | ทุก Switch/Router (Trust Boundary ที่ Access, Enforce ที่ Core/WAN) | Part 40 |
| Multicast Fundamentals: IGMP, PIM-SM | DIST-SW1-4 (IGMP), CORE-SW1/2 (PIM) | Part 41 |
| Multicast Advanced: Anycast RP (1.1.1.100) + MSDP | CORE-SW1/2 | Part 42 |
| Security Architecture: FTD-1 (NGFW), ISE-1/2 (PAN/MnT/PSN) | FTD-1, ISE-1(.65)/ISE-2(.66) | Part 43 |
| 802.1X (IBNS 2.0) + TrustSec (SGT/SGACL/SXP) | ACCESS-SW1-4, DIST-SW3 (SGACL Enforce) | Part 44 |
| Advanced ACL & Zone-Based Firewall (ZBFW) | WAN-EDGE-1/2 (Zone INSIDE/OUTSIDE/self) | Part 45 |
| Control Plane Policing (CoPP): COPP-CRITICAL/IMPORTANT/NORMAL | CORE-SW1/2 (หลักๆ), ทุก Router/Switch (ควรมี) | Part 46 |
| Automation: Python (Netmiko/NAPALM/Nornir) | NOC-SRV (Automation Control Node) | Part 47 |
| Automation: Ansible (Playbook/Role/Inventory) | NOC-SRV | Part 48 |
| APIs: NETCONF/RESTCONF/YANG Deep Dive | ทุกอุปกรณ์ (NETCONF/RESTCONF Agent) | Part 49 |
| Cisco DNA Center / Catalyst Center (Underlay Automation, Assurance) | DNAC-VM (10.10.99.60) | Part 50 |
| VRF-Lite, GRE, IPsec VPN | DIST-SW3 (PARTNER-VRF), WAN-EDGE-1↔BRANCH-RTR (GRE+IPsec) | Part 51 |
| DMVPN Fundamentals (Hub-Spoke, mGRE, NHRP) | WAN-EDGE-1 (Hub1), BRANCH-RTR | Part 52 |
| DMVPN Advanced: Dual-Hub Phase 3 + Spoke-to-Spoke Shortcut | WAN-EDGE-1/2 (Dual Hub), BRANCH-RTR/-2 (Spoke) | Part 53 |
| Network Assurance & Telemetry: Model-Driven Telemetry (gRPC), Flexible NetFlow, IP SLA, EEM | ทุกอุปกรณ์ → NOC-SRV (Telemetry Collector) | Part 54 |

> **ข้อสังเกตสำคัญสำหรับ Capstone**: สังเกตว่า CORE-SW1/CORE-SW2 ที่เคยเป็น "2 Chassis แยกกันเชื่อม
> ด้วย Po1" ตั้งแต่ Part 1 กลายเป็น **1 Logical Switch เดียว (StackWise Virtual)** ตั้งแต่ Part 34
> เป็นต้นไป — ผลกระทบสำคัญคือ EtherChannel ระหว่าง DIST↔CORE ที่เคย Single-homed (DIST ต่อ CORE
> ตัวเดียว ตาม IP Plan เดิม) กลายเป็น **MEC (Multichassis EtherChannel)** ที่ DIST แต่ละตัวต่อสาย
> ไปทั้ง 2 Chassis ของ SVL Pair พร้อมกันได้จริง โดยที่ฝั่ง DIST มองเห็นเป็น Port-channel เดียว
> เหมือน STP/LACP มองเห็น Peer เดียว — นี่คือคำตอบของ "Dual-homed DIST↔CORE" ที่ `00-ip-address-
> plan.md` บอกไว้ล่วงหน้าตั้งแต่ Part 10 ว่าจะมาใน Part 33-35 ของ ENCOR

---

## Step 542 — ENCOR 350-401 Exam Blueprint Mapping

Cisco แบ่งข้อสอบ ENCOR 350-401 เป็น 6 Domain พร้อมสัดส่วนน้ำหนักคะแนนอย่างเป็นทางการ ตารางนี้
Mapping แต่ละ Domain เข้ากับ Part ที่หลักสูตรนี้สอนไปแล้ว (Part 26-54 รวมกับพื้นฐาน CCNA ที่ยังใช้
ต่อเนื่อง):

| # | Domain (ชื่อทางการ) | น้ำหนัก | Part หลักที่ครอบคลุม |
|---|---|---|---|
| 1.0 | Architecture | **15%** | Part 33-34 (Campus Design/SVL/MEC), Part 38 (SD-Access), Part 39 (SD-WAN), Part 51-53 (VRF/DMVPN WAN Architecture) |
| 2.0 | Virtualization | **10%** | Part 51 (VRF-Lite), Part 52-53 (DMVPN Tunnel/GRE Overlay) |
| 3.0 | Infrastructure | **30%** | Part 26-28 (OSPF/OSPFv3/EIGRP), Part 29-32 (BGP/Redistribution/PBR), Part 35 (HA/FHRP), Part 36-37 (Wireless), Part 40-42 (QoS/Multicast) |
| 4.0 | Network Assurance | **10%** | Part 54 (Telemetry, NetFlow, IP SLA, EEM) |
| 5.0 | Security | **20%** | Part 43-46 (Firepower/ISE, 802.1X/TrustSec, ZBFW, CoPP) |
| 6.0 | Automation | **15%** | Part 47-49 (Python/Ansible/NETCONF-RESTCONF-YANG), Part 50 (DNA Center) |
| — | **รวม** | **100%** | Part 26-54 ครบทุก Domain |

### รายละเอียดหัวข้อย่อยที่ Cisco ระบุใน Blueprint จริง เทียบกับ Part ที่สอน

| Blueprint Sub-topic (ตัวอย่างสำคัญ) | Part ที่ตรงกัน |
|---|---|
| 1.1 Explain the different design principles used in an enterprise network (QoS, HA, etc.) | Part 33-35, 40 |
| 1.2 Analyze design principles of a WLAN deployment | Part 36-37 |
| 1.3 Differentiate hardware and software redundancy (SSO, StackWise) | Part 34 |
| 1.5 Explain the working principles of the Cisco SD-WAN solution | Part 39 |
| 1.6 Explain the working principles of the Cisco SD-Access solution | Part 38 |
| 2.4 Describe the components and benefits of VRF-Lite | Part 51 |
| 2.5 Describe the working principles of DMVPN (single hub) | Part 52 |
| 3.1 Compare EIGRP and OSPF (advanced concepts) | Part 26, 28 |
| 3.2 Configure and verify simple OSPF environments (NSSA, filtering) | Part 26 |
| 3.3 Describe, configure, and verify BGP (peering, path attributes) | Part 29-30 |
| 3.4 Configure and verify RIP2 for IPv4/simple redistribution | Part 31 |
| 3.5 Configure and verify policy-based routing | Part 32 |
| 3.6 Identify suboptimal routing issues | Part 31, 32, Step 544-545 |
| 3.7 Explain how VXLAN operates (data plane in SD-Access) | Part 38 |
| 3.8 Describe HSRP/VRRP (advanced tracking) | Part 35 |
| 3.9 Describe device management using AAA with TACACS+ and RADIUS | Part 43 (ISE), Part 22 (CCNA base) |
| 3.10 Describe network events with syslog (advanced) | Part 54 |
| 3.11-3.14 Multicast (IGMP/PIM/Rendezvous Point) | Part 41-42 |
| 3.16 QoS (marking, shaping, policing, queuing) | Part 40 |
| 3.17 Wireless QoS, roaming | Part 36-37, 40 |
| 4.1 Diagnose network problems using tools (NetFlow, IP SLA, model-driven telemetry) | Part 54 |
| 4.4 Configure NetFlow/Flexible NetFlow | Part 54 |
| 4.5 Configure network devices for monitoring via SNMP/syslog | Part 17 (CCNA), Part 54 |
| 5.1 Configure and verify device access control (802.1X, MAB, WebAuth) | Part 44 |
| 5.2 Configure and verify infrastructure security features (CoPP, ACL) | Part 45-46 |
| 5.3 Describe REST APIs for security appliances (Firepower/ISE) | Part 43 |
| 5.4 Explain TrustSec (SGT/SGACL) | Part 44 |
| 5.5 Describe security capabilities within SD-Access | Part 38, 44 |
| 6.1-6.4 Interpret/Construct Python for API-based management | Part 47 |
| 6.5 Construct EEM applet | Part 54 |
| 6.6 Compare agent vs agentless (Chef/Puppet/Ansible) | Part 48 |
| 6.7 Interpret REST API responses in Python (error codes) | Part 47, 49 |
| 6.8 Explain the widespread use of YANG as a data modeling language | Part 49 |
| 6.9 Describe the components of NETCONF/RESTCONF | Part 49 |
| 6.10 Explain the concepts of Cisco DNA Center Assurance/workflows | Part 50 |

> **ข้อสังเกตสำคัญ**: Domain 3.0 (Infrastructure, 30%) ยังคงมีน้ำหนักสูงสุด เหมือนกับที่ Domain
> "IP Connectivity" เคยมีน้ำหนักสูงสุดในข้อสอบ CCNA — แต่สิ่งที่ต่างจาก CCNA คือ **Domain 5.0
> (Security, 20%) และ Domain 6.0 (Automation, 15%) มีน้ำหนักรวมกันสูงถึง 35%** ซึ่งมากกว่าที่
> CCNA เคยให้ความสำคัญมาก — สะท้อนว่า ENCOR คาดหวังให้วิศวกรมองเครือข่ายทั้งระบบ (Security +
> Automation) ไม่ใช่แค่ Routing/Switching เพียวๆอีกต่อไป **ควรจัดสรรเวลาทวน Part 43-49 ให้มาก
> พอๆกับ Part 26-32 ก่อนสอบจริง**

---

## Step 543 — Capstone Integration Checklist: Master Build-Order Runbook

นี่คือ Runbook ระดับ ENCOR เต็มรูปแบบ — สมมติว่าเรามี Enterprise Lab ที่ Bring-up ครบระดับ CCNA
(Part 1-24, ตาม Runbook ของ [Part 25 Step 243](part-025-ccna-capstone-lab.md#step-243)) อยู่แล้ว
และต้อง **Layer เพิ่ม Feature ระดับ ENCOR ทั้งหมด (30 Part) เข้าไปทีละชั้นตามลำดับ Dependency ที่
ถูกต้อง** — ทำผิดลำดับในระดับนี้ผลกระทบรุนแรงกว่า CCNA มาก (เช่น เปิด SVL ก่อนเช็ค Physical
Cabling ให้ครบ อาจทำให้ Core ทั้งคู่ Split-Brain/Dual-Active ทันที)

### Runbook ฉบับสมบูรณ์ (Checklist ตามลำดับ Phase, ต่อจาก Phase 7 ของ Part 25)

```
PHASE 8 — Campus Physical Consolidation (อ้างอิง Part 33-34) — ทำก่อนสิ่งอื่นใดในระดับ ENCOR
  [ ] ตรวจสอบ Physical Cabling ระหว่าง CORE-SW1↔CORE-SW2 มี SVL Link อย่างน้อย 2 เส้น 10G/40G ขึ้นไป
  [ ] ตั้งค่า DAD (Dual-Active Detection) Link แยกจาก SVL Link เสมอ ก่อนเปิด SVL จริง
  [ ] stackwise-virtual → domain 100 (ตรงกันทั้ง 2 Chassis) → stackwise-virtual link 1 บน
      Interface ที่เลือก → Reload พร้อมกันทั้งคู่
  [ ] Verify: show switch stack-ports summary, show stackwise-virtual, ยืนยันเห็น "1 Switch"
      ใน show version (ไม่ใช่ 2 Chassis แยกกันอีกต่อไป)
  [ ] เดินสาย MEC เพิ่มจาก DIST-SW1-4 แต่ละตัวไปยัง**ทั้ง 2 Chassis**ของ SVL Pair (Dual-homed)
  [ ] แปลง Port-channel เดิม (Single-homed) เป็น MEC บน DIST-SW1-4 ทีละตัว — ห้ามทำทุกตัวพร้อมกัน
      (ทำทีละตัว Verify ก่อนไปตัวต่อไป กัน Traffic ทั้ง Block ขาดพร้อมกันถ้าทำผิด)
  [ ] Verify: show etherchannel summary (เห็น Member มาจาก 2 Chassis Physical แต่เป็น Po เดียว)

PHASE 9 — Advanced Routing Layer (อ้างอิง Part 26-32) — ทำหลัง SVL/MEC เสถียรแล้วเท่านั้น
  [ ] OSPF Advanced: เปลี่ยน Area 2 (DIST-SW3/4) เป็น NSSA (`area 2 nssa`) — ทำที่ ABR (CORE-SW2)
      ก่อน แล้วค่อยทำที่ DIST-SW3/4 (ABR ต้องรับรู้ก่อน ไม่งั้น Router ในพื้นที่จะเห็น Area
      Type ไม่ตรงกันชั่วคราว — "Area Mismatch")
  [ ] เปิด BFD ผูกกับ OSPF (`ip ospf bfd` ทุก Backbone Interface) — ทำหลัง OSPF Adjacency Full
      เสถียรแล้วเท่านั้น (BFD ที่เปิดตอน Adjacency ยังไม่ Full อาจทำให้ Flap ถี่กว่าที่ตั้งใจ)
  [ ] เปิด OSPFv3 (IPv6) คู่กับ OSPFv2 บนทุก Interface ที่มี IPv6 ULA อยู่แล้ว (Part 8) —
      ตั้ง router-id แบบ Manual เสมอ (Loopback0 เดียวกับ OSPFv2)
  [ ] เปิด eBGP ที่ WAN-EDGE-1(→ISP1 AS65000) และ WAN-EDGE-2(→ISP2 AS65002) — ทดสอบ Session
      Establish ก่อนไปขั้นถัดไป
  [ ] เปิด iBGP AS65001 ระหว่าง WAN-EDGE-1/2 ↔ CORE-SW1/2 (Full-Mesh 4 Session) — Verify
      `show ip bgp summary` ทุกตัวก่อน
  [ ] Redistribute BGP→OSPF ด้วย `default-information originate always metric ...` **เท่านั้น**
      (ห้าม `redistribute bgp ... subnets` แบบไม่กรอง — ดู Step 544 ว่าทำไมสำคัญ)
  [ ] ทำ EIGRP↔OSPF Mutual Redistribution ที่ DIST-SW3/4 ด้วย Route-map+Tag (กัน Feedback Loop)
  [ ] ตั้ง PBR ที่ DIST-SW1/2 พร้อม `set ip next-hop verify-availability ... track <obj>` —
      ต้องตั้ง IP SLA Track **ก่อน** เขียน Route-map เสมอ (Track Object ต้องมีอยู่แล้วให้ Route-map
      อ้างถึง ไม่ใช่สร้างพร้อมกัน)

PHASE 10 — Advanced HA & Wireless (อ้างอิง Part 35-37)
  [ ] เพิ่ม Track Object ตัวที่ 2 ให้ HSRP ทุก Group บน DIST-SW1-4 (Uplink ไป CORE ทั้ง 2 เส้นของ
      MEC) — ต้องทำหลัง Phase 8 (MEC) เสร็จเท่านั้น เพราะต้อง Track ทั้ง 2 Physical Path จริง
  [ ] ตั้ง Preempt Delay Minimum ทุก Group (ทวนจาก Part 18/25 Step 246 — ยังคงสำคัญที่ระดับนี้)
  [ ] Deploy WLC-1 Feature ขั้นสูง: CAPWAP Redundancy, Roaming (Part 36), WPA2/3-Enterprise
      ผ่าน 802.1X กับ ISE (Part 37) — ต้องมี ISE พร้อมใช้งานจริงก่อน (ดู Phase 12)

PHASE 11 — QoS & Multicast (อ้างอิง Part 40-42)
  [ ] MQC 3-Class (VOICE/VIDEO/CRITICAL-DATA) — Class-map ก่อน, Policy-map ทีหลัง, Apply
      (`service-policy`) ที่ Interface **ขาออก**เสมอสำหรับ Queuing/Shaping (ทวน Part 24)
  [ ] เปิด `ip multicast-routing` + `ip pim sparse-mode` ทุก Interface ที่เกี่ยวข้อง **ก่อน**
      ตั้ง RP ใดๆ (Multicast Routing ต้องเปิด Global ก่อนเสมอ ไม่งั้น PIM Neighbor จะไม่ขึ้น)
  [ ] ตั้ง Anycast RP (1.1.1.100) บน CORE-SW1/2 พร้อม MSDP Peer ระหว่างกัน — ต้องตั้งทั้งสองข้าง
      พร้อมกัน (Anycast RP ที่มีแค่ตัวเดียวประกาศ = Traffic ครึ่งหนึ่งหา RP ไม่เจอ)

PHASE 12 — Security Stack (อ้างอิง Part 43-46) — ทำก่อน Automation เสมอ (Security ต้องมาก่อน API เปิด)
  [ ] Deploy ISE-1(PAN+MnT, .65)/ISE-2(PSN, .66) ให้ Online และ Sync กันก่อน Config อะไรที่พึ่งพา
  [ ] Deploy FTD-1 (NGFW) ที่ WAN Edge — Policy พื้นฐานก่อน (Allow ที่จำเป็น) ยังไม่ต้อง Deep
      Inspection เต็มรูปแบบในรอบแรก
  [ ] 802.1X (IBNS 2.0, `authentication port-control auto`) บน ACCESS-SW1-4 — ทดสอบกับ 1 Port
      ก่อนขยายทุก Port (กัน Lock-out ผู้ใช้งานทั้ง Access Switch พร้อมกัน)
  [ ] TrustSec: เปิด SGT Assignment จาก ISE → SXP ไปยัง DIST-SW3 → SGACL Enforce (ลำดับนี้ตายตัว
      — SGACL ที่ Enforce ก่อนมี SGT Binding ที่สมบูรณ์ = Block Traffic ที่ควร Allow โดยไม่ตั้งใจ)
  [ ] ZBFW ที่ WAN-EDGE-1/2 (Zone INSIDE/OUTSIDE/self) — ทำหลัง FTD Rule พื้นฐานเสถียรแล้ว
      (ทั้งสองเป็น Stateful Inspection คนละชั้น อย่าเปิดพร้อมกันในรอบแรกที่ยังไม่ Baseline Traffic)
  [ ] CoPP (COPP-CRITICAL/IMPORTANT/NORMAL) บน CORE-SW1/2 **เป็นลำดับสุดท้ายของ Security Stack**
      เสมอ — เพราะ CoPP กระทบ Control-Plane Traffic ของ Routing Protocol ที่ตั้งไว้แล้วทั้งหมด
      (ดู Step 546 กับดัก CoPP Policy Update ทำ OSPF Hello หลุด)

PHASE 13 — Automation & Assurance (อ้างอิง Part 47-50, 54) — ทำหลัง Security Stack เสถียรเท่านั้น
  [ ] เปิด NETCONF/RESTCONF (`netconf-yang`, `restconf`) ทุกอุปกรณ์ — ต้องมี AAA/Local User
      พร้อม Privilege 15 ที่ใช้ Authenticate API ได้ก่อน (Part 22 พื้นฐาน + Part 43 ISE)
  [ ] Deploy Python/Ansible Automation Control Node ที่ NOC-SRV — ทดสอบ Read-only Task
      (`show version` ผ่าน Netmiko/NAPALM) ก่อน Push Config จริงเสมอ
  [ ] Onboard อุปกรณ์เข้า DNAC-VM (.60) — Discovery ผ่าน SNMP/NETCONF ที่เปิดไว้แล้ว
  [ ] เปิด Model-Driven Telemetry (gRPC Dial-out) ส่งไปยัง NOC-SRV Collector — ตั้ง Subscription
      เฉพาะ YANG Path ที่จำเป็นก่อน (CPU/Interface/BGP) ไม่เปิดทุก Path พร้อมกันในรอบแรก

PHASE 14 — WAN Overlay (อ้างอิง Part 51-53) — ทำคู่ขนานกับ Phase 9-13 ได้ (WAN Edge เป็นคนละ
Physical Path จาก Campus Core) แต่ต้องรอ BGP (Phase 9) เสถียรก่อนเสมอ เพราะ DMVPN NHRP พึ่งพา
Underlay Routing ที่ใช้ไปหา NBMA Address ของ Peer
  [ ] VRF-Lite (`PARTNER-VRF`) ที่ DIST-SW3 — สร้าง VRF ก่อน ค่อย Assign Interface เข้า VRF
      (ลำดับผิด = Interface หลุดจาก Global Table ไปอยู่ VRF เปล่าที่ยังไม่มี Route)
  [ ] GRE Tunnel (WAN-EDGE-1↔BRANCH-RTR) จาก Part 19 เดิม → เพิ่ม IPsec Protection (Crypto
      ISAKMP/IPsec ครอบ Tunnel Interface เดิม) — ต้องมี GRE Reachable ปกติก่อนค่อยเพิ่ม Encryption
  [ ] DMVPN Hub1 (WAN-EDGE-1) ก่อน Hub2 (WAN-EDGE-2) — ตั้ง mGRE + NHRP บน Hub1 ให้ Spoke
      Register สำเร็จก่อน แล้วค่อยเพิ่ม Hub2 เข้ามาเป็น Dual-Hub
  [ ] เปิด NHRP Shortcut (`ip nhrp shortcut`) + Redirect (`ip nhrp redirect`) เพื่อให้เกิด
      Phase 3 Spoke-to-Spoke จริง — ต้องเปิดทั้งที่ Hub (redirect) และ Spoke (shortcut) พร้อมกัน
      ไม่งั้น Traffic จะยัง Hairpin ผ่าน Hub (ดู Root Cause เต็มที่ Step 545)
```

### Dependency Graph สรุปภาพเดียว (ระดับ ENCOR)

```
SVL/MEC (Phase 8) ──► Advanced Routing: OSPF/BGP/Redistribution/PBR (Phase 9)
                              │
                              ▼
                     Advanced HA/Wireless (Phase 10) ──► QoS/Multicast (Phase 11)
                                                                  │
                                                                  ▼
                                              Security Stack: ISE→802.1X→TrustSec→ZBFW→CoPP (Phase 12)
                                                                  │
                                                                  ▼
                                        Automation/Assurance: NETCONF→Ansible→DNAC→Telemetry (Phase 13)

Phase 14 (VRF/GRE-IPsec/DMVPN) วิ่งคู่ขนานได้ แต่ DMVPN ต้องรอ Phase 9 (BGP Underlay เสถียร) ก่อน
```

> **กฎเหล็กของ Runbook ระดับ ENCOR**: สังเกตว่า **CoPP มาเป็นลำดับสุดท้ายของ Security Stack เสมอ**
> และ **Automation มาหลัง Security Stack เสมอ** — เหตุผลเดียวกันทั้งคู่: ทั้ง CoPP และการเปิด
> API (NETCONF/RESTCONF) ส่งผลกระทบต่อ **สิ่งที่ทำงานอยู่แล้วทั้งระบบ** (CoPP กระทบทุก Control-
> Plane Protocol ที่ Config ไว้แล้ว, API ที่เปิดโดยไม่มี AAA/ACL ป้องกันดีพอ = ช่องโหว่ใหม่) —
> จึงต้องทำเป็นลำดับท้ายสุดหลัง Baseline ทุกอย่างเสถียรแล้วเท่านั้น ไม่ใช่ทำตั้งแต่ต้น

---

## Step 544 — Troubleshooting Scenario 1: BGP Redistribute เข้า OSPF ผิด ทำ Remote Site เสีย Internet Failover

### อาการที่รายงานเข้ามา

> "BRANCH-RTR (เชื่อมผ่าน WAN-EDGE-1 ด้วย Leased Line/GRE) รายงานว่า **Internet ใช้งานไม่ได้เลย
> ตั้งแต่เมื่อคืนนี้** ทีม NOC ยืนยันว่า **ISP1 (Link ของ WAN-EDGE-1) ล่มจริงตั้งแต่เมื่อคืน** แต่
> ที่ควรเกิดขึ้นคือ Traffic ต้อง Failover ไปใช้ **ISP2 (ผ่าน WAN-EDGE-2)** โดยอัตโนมัติเหมือนที่
> เคยทดสอบไว้ตอน Design — แต่รอบนี้ **ไม่ Failover เลย ทุก Site ยังส่ง Traffic ไปทาง WAN-EDGE-1
> เหมือนเดิม (Black Hole)**"

### ขั้นตอน Troubleshoot แบบ Systematic

**Step A — ยืนยันสถานะ BGP ที่ WAN-EDGE-1 ก่อน (จุดที่ ISP1 ล่มจริง)**

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 65000       0       0        1    0    0 never    Idle    <- ISP1 eBGP ล่มจริง
1.1.1.1         4 65001    5821    5819       12    0    0 3d02h          4
1.1.1.2         4 65001    5820    5818       12    0    0 3d02h          4
```

BGP Session ไป ISP1 (203.0.113.1) อยู่ใน **Idle** จริงตามที่ NOC รายงาน — WAN-EDGE-1 **ไม่มี
Default Route จาก BGP อีกต่อไป** (BGP Table ควรไม่มี Route จาก AS65000 หลงเหลือแล้ว)

```
WAN-EDGE-1# show ip route 0.0.0.0
% Network not in table
```

**ยืนยันแล้ว**: WAN-EDGE-1 **ไม่มี** Default Route ใน RIB ของตัวเองเลย (ตรงตามคาด เพราะ ISP1 ล่ม)

**Step B — ตรวจสอบว่า WAN-EDGE-1 ยัง "โฆษณา" Default Route เข้า OSPF อยู่หรือไม่ (จุดที่คาดไม่ถึง)**

```
WAN-EDGE-1# show run | section router ospf
router ospf 1
 router-id 1.1.1.21
 default-information originate always metric 1 metric-type 1
 passive-interface default
 no passive-interface TenGigabitEthernet0/0/0
```

**พบ Root Cause แล้ว**: Keyword **`always`** ทำให้ WAN-EDGE-1 **โฆษณา Default Route (`0.0.0.0/0`)
เข้า OSPF ต่อไปเรื่อยๆ โดยไม่สนใจว่าตัวเองมี Default Route จริงใน RIB หรือไม่** — ต่างจากไม่มี
`always` ที่จะหยุดโฆษณาทันทีถ้า Default Route หายไปจาก RIB (พฤติกรรม Conditional ตามปกติ)

```
CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 1", distance 110, metric 1, candidate default path
  Last update from 10.255.10.2 on TenGigabitEthernet1/1/1, 00:00:12 ago
  Routing Descriptor Blocks:
  * 10.255.10.2, from 1.1.1.21, 00:00:12 ago, via TenGigabitEthernet1/1/1
      Route metric is 1, traffic share count is 1
```

CORE-SW1 ยังเห็น Default Route จาก WAN-EDGE-1 (metric **1**) เท่านั้น — **ไม่เห็น Route จาก
WAN-EDGE-2 เลย** (แม้ WAN-EDGE-2 ก็ตั้ง `default-information originate always metric 5
metric-type 1` เหมือนกัน) เพราะ metric ของ WAN-EDGE-1 (1) ยังต่ำกว่า WAN-EDGE-2 (5) เสมอ — OSPF
จึงยังเลือก Path ไป WAN-EDGE-1 เป็น Best Path ทั้งที่ Path นั้น**ไม่มี Internet จริงรออยู่ปลายทาง
เลย** เกิดเป็น **Black Hole เต็มรูปแบบ**

### Root Cause สรุป

```
Timeline ของปัญหา:

t=0      ISP1 Link ล่ม → BGP Session WAN-EDGE-1↔ISP-RTR หลุด (Idle) → WAN-EDGE-1 ไม่มี
         0.0.0.0/0 ใน BGP Table/RIB ของตัวเองอีกต่อไป

t=0+     WAN-EDGE-1 ยังคง Advertise 0.0.0.0/0 เข้า OSPF ต่อไป เพราะใช้ `always` — OSPF ไม่รู้เลย
         ว่า BGP ข้างในตายไปแล้ว (default-information originate always = "โฆษณาเสมอ ไม่เช็ค RIB")

t=0+     CORE-SW1/CORE-SW2 ยังเห็น O*E1 Default Route จาก WAN-EDGE-1 (metric 1) ต่ำกว่าจาก
         WAN-EDGE-2 (metric 5) เสมอ → เลือก WAN-EDGE-1 เป็น Best Path ต่อไปไม่เปลี่ยน

t=0+     Traffic ทุก Site (รวม BRANCH-RTR) ที่ต้องออก Internet ถูกส่งไปที่ WAN-EDGE-1 → ตกไปที่
         Interface ที่ไม่มี BGP Session จริงรออยู่ → Drop เงียบๆ (Black Hole)
```

**สรุป Root Cause**: `default-information originate always` เหมาะกับกรณีที่ Router ต้องการเป็น
"Default Gateway ของ OSPF Domain เสมอไม่ว่าอะไรจะเกิด" (เช่น Router ที่มี Static Default Route
ของตัวเองเป็น Fallback) แต่ **ไม่เหมาะกับ ASBR ที่ Default Route มาจาก BGP Dynamic** เพราะ
`always` ตัดการเชื่อมโยงระหว่าง "สถานะจริงของ BGP" กับ "สิ่งที่ OSPF เห็น" ออกจากกันโดยสมบูรณ์

### วิธีแก้: ถอด `always` ออก + ปรับให้ Metric สะท้อนความจริงของ Path

```
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# no default-information originate always metric 1 metric-type 1
WAN-EDGE-1(config-router)# default-information originate metric 1 metric-type 1
WAN-EDGE-1(config-router)# exit

WAN-EDGE-2(config)# router ospf 1
WAN-EDGE-2(config-router)# no default-information originate always metric 5 metric-type 1
WAN-EDGE-2(config-router)# default-information originate metric 5 metric-type 1
WAN-EDGE-2(config-router)# exit
```

โดยไม่มี `always`, `default-information originate` จะ**โฆษณาเข้า OSPF ก็ต่อเมื่อ Router ตัวเองมี
`0.0.0.0/0` อยู่ใน RIB จริงเท่านั้น** — พอ ISP1 ล่มและ BGP ไม่มี Default Route จริง WAN-EDGE-1 จะ
**หยุดโฆษณาเข้า OSPF โดยอัตโนมัติ** ทำให้ CORE-SW1/2 เหลือแค่ Route จาก WAN-EDGE-2 (metric 5)
เพียงตัวเดียว และ Failover เกิดขึ้นจริงตามที่ควรจะเป็น

```
! ทดสอบยืนยันหลังแก้ไข (จำลอง ISP1 ล่มอีกครั้ง)
WAN-EDGE-1# show ip route 0.0.0.0
% Network not in table
WAN-EDGE-1# show ip ospf database | include 0.0.0.0
                                                          <- ไม่เห็น Type-5 LSA 0.0.0.0/0 จาก WAN-EDGE-1 แล้ว

CORE-SW1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 1", distance 110, metric 5, candidate default path
  * 10.255.20.2, from 1.1.1.22, 00:00:03 ago, via TenGigabitEthernet1/1/2   <- Failover ไป WAN-EDGE-2 สำเร็จ
```

> **บทเรียนสำหรับข้อสอบ**: `default-information originate always` เป็นกับดักคลาสสิกของ ENCOR —
> ข้อสอบชอบให้โจทย์ "Redistribute BGP Default เข้า OSPF ที่ WAN Edge คู่ (Dual-ISP)" แล้วถามว่า
> ทำไม Failover ไม่เกิดขึ้นจริง คำตอบมักวนอยู่ที่ `always` (ทำให้ Advertise ไม่สนใจ RIB จริง) หรือ
> Metric ตั้งผิด (ทำให้ Path ที่ควรเป็น Backup กลับถูกเลือกเป็น Primary เสมอ) — ให้จำหลักไว้ว่า
> **`always` ควรใช้เฉพาะ ASBR ที่มี Static Default Route ของตัวเองเป็นฐานเท่านั้น ไม่ใช่กับ BGP
> Dynamic Default Route**

---

## Step 545 — Troubleshooting Scenario 2: DMVPN Spoke-to-Spoke Hairpin ผ่าน Hub

### อาการที่รายงานเข้ามา

> "BRANCH-RTR และ BRANCH-RTR-2 (ทั้งคู่เป็น DMVPN Spoke ของ Dual-Hub WAN-EDGE-1/WAN-EDGE-2)
> ต้องคุยกันบ่อยเพื่อ Replicate ข้อมูลระหว่างสาขา — ทีม NOC สังเกตว่า **Latency ระหว่าง 2 สาขาสูง
> กว่าที่คาดมาก และ Traffic ทั้งหมดยังวิ่งผ่าน WAN-EDGE-1 (Hub) เสมอ** ทั้งที่ตั้งใจ Design ให้ใช้
> **DMVPN Phase 3 Spoke-to-Spoke Shortcut** (คุยตรงกันโดยไม่ผ่าน Hub) มาตั้งแต่ต้น"

### ขั้นตอน Troubleshoot

**Step A — ยืนยัน Phase ที่ใช้จริงก่อน (Phase 1/2 ไม่รองรับ Spoke-to-Spoke Shortcut อัตโนมัติ)**

```
BRANCH-RTR# show ip nhrp | include Type
172.16.220.1/32 via 172.16.220.1, Tunnel0 created 3d10h, never expire Type: static

BRANCH-RTR# show dmvpn
Interface: Tunnel0, IPv4 NHRP Details
Type:Spoke, NHRP Peers:2

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 203.0.113.2        172.16.220.1    UP 3d10h     S   <- Hub1 (Static NHRP mapping, ปกติ)
     1 203.0.113.6        172.16.220.2    UP 3d10h     S   <- Hub2 (Static NHRP mapping, ปกติ)
```

Config `Tunnel0` เป็น mGRE (`tunnel mode gre multipoint`) และ NHRP Map ไปยัง Hub ทั้งคู่ถูกต้อง —
**Phase ที่ใช้คือ Phase 3 จริง** (ไม่ใช่ Phase 1/2 ที่ออกแบบมาไม่รองรับ Shortcut อยู่แล้ว)
ต้องดูขั้นถัดไปว่าทำไม Shortcut ไม่เกิดขึ้นจริง

**Step B — ตรวจ Config ที่ Hub (WAN-EDGE-1) ว่ามี `ip nhrp redirect` หรือไม่**

```
WAN-EDGE-1# show run interface Tunnel0
interface Tunnel0
 ip address 172.16.220.1 255.255.255.0
 no ip redirects
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 tunnel source TenGigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 1
```

**พบ Root Cause แล้ว**: Hub ไม่มีคำสั่ง **`ip nhrp redirect`** — คำสั่งนี้คือสิ่งที่บอกให้ Hub
"แจ้งกลับ (Redirect) ไปยัง Spoke ต้นทางว่ามี Path ที่ดีกว่า (ตรงไปยัง Spoke ปลายทาง) ให้ใช้แทน"
ทุกครั้งที่ Hub เห็นว่าตัวเองต้อง Forward Traffic ระหว่าง 2 Spoke (นั่นคือสัญญาณว่ากำลัง Hairpin
อยู่) — ถ้าไม่มีคำสั่งนี้ **Hub จะ Forward Traffic ต่อไปเรื่อยๆโดยไม่บอกอะไร Spoke เลย แม้จะเป็น
mGRE Phase 3 ที่ถูกต้องแล้วก็ตาม**

**Step C — ตรวจ Config ที่ Spoke (BRANCH-RTR) ว่ามี `ip nhrp shortcut` หรือไม่**

```
BRANCH-RTR# show run interface Tunnel0
interface Tunnel0
 ip address 172.16.220.11 255.255.255.0
 ip nhrp network-id 1
 ip nhrp nhs 172.16.220.1 nbma 203.0.113.2
 ip nhrp nhs 172.16.220.2 nbma 203.0.113.6
 tunnel source GigabitEthernet0/0/1
 tunnel mode gre multipoint
 tunnel key 1
```

**พบ Root Cause ส่วนที่ 2**: Spoke ก็ไม่มีคำสั่ง **`ip nhrp shortcut`** เช่นกัน — แม้ Hub จะส่ง
NHRP Redirect มาให้ (ถ้าแก้ Step B แล้ว) แต่ถ้า Spoke ไม่มี `ip nhrp shortcut` **Spoke จะเพิกเฉย
ต่อ Redirect นั้น และยังคงใช้ Route เดิมที่ชี้ผ่าน Hub อยู่ดี** (CEF ที่ Spoke จะไม่สร้าง Shortcut
Entry ใหม่เข้า Routing Table)

### Root Cause สรุป

```
DMVPN Phase 3 ต้องมี 2 องค์ประกอบพร้อมกันเสมอ ขาดอย่างใดอย่างหนึ่ง = Hairpin ต่อไปเหมือน Phase 2:

  Hub:    ip nhrp redirect     -> "แจ้ง Spoke ว่ามีทางลัดให้ใช้"  (ขาดใน Scenario นี้)
  Spoke:  ip nhrp shortcut     -> "ยอมรับทางลัดที่ Hub แจ้งมา และสร้าง CEF Entry ใหม่" (ขาดด้วย)

ผลคือ Traffic BRANCH-RTR -> BRANCH-RTR-2 ยังคงเดินตาม Routing Table เดิม (ผ่าน Hub เสมอ)
เพราะไม่มีฝั่งไหนเริ่ม "แจ้ง" หรือ "รับ" กระบวนการ Shortcut เลย — เป็น Phase 3 แค่ในเชิง mGRE/
NHRP Mapping (รองรับได้) แต่ไม่ได้ "เปิดใช้งานจริง" (ไม่ได้ตั้งคำสั่งเปิด Shortcut Behavior)
```

### วิธีแก้: เปิด NHRP Redirect ที่ Hub ทั้งคู่ + Shortcut ที่ทุก Spoke

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip nhrp redirect
WAN-EDGE-1(config-if)# exit

WAN-EDGE-2(config)# interface Tunnel0
WAN-EDGE-2(config-if)# ip nhrp redirect
WAN-EDGE-2(config-if)# exit

BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp shortcut
BRANCH-RTR(config-if)# exit

BRANCH-RTR-2(config)# interface Tunnel0
BRANCH-RTR-2(config-if)# ip nhrp shortcut
BRANCH-RTR-2(config-if)# exit
```

```
! ทดสอบยืนยันผลลัพธ์ — ยิง Traffic ต่อเนื่องจาก BRANCH-RTR ไป BRANCH-RTR-2 แล้วเช็คอีกครั้ง
BRANCH-RTR# show ip nhrp
172.16.220.12/32 via 172.16.220.12, Tunnel0 created 00:00:08, expire 01:59:51
    Type: dynamic, Flags: router rib nho          <- Shortcut Entry ใหม่เกิดขึ้นจริง!
    NBMA address: 198.51.100.20

BRANCH-RTR# show dmvpn
     2 198.51.100.20      172.16.220.12   UP 00:00:08     ND    <- "ND" = NHRP Dynamic (Spoke-to-Spoke จริง)
```

Flag `ND` (NHRP Dynamic) ที่ปรากฏใหม่ใน `show dmvpn` คือหลักฐานว่า Spoke-to-Spoke Shortcut
เกิดขึ้นจริงแล้ว — Traffic ระหว่าง BRANCH-RTR ↔ BRANCH-RTR-2 จากนี้ไปวิ่งตรงถึงกันทาง Internet
โดยไม่ผ่าน Hub อีกต่อไป (Latency กลับมาปกติ)

> **บทเรียนสำหรับข้อสอบ**: จำคู่คำสั่งนี้ให้แม่น — **`ip nhrp redirect` ต้องอยู่ที่ Hub เท่านั้น,
> `ip nhrp shortcut` ต้องอยู่ที่ Spoke เท่านั้น** สลับข้างกันจะไม่มีผลอะไรเลย (Hub ไม่ต้องการ
> Shortcut เพราะมันเป็นศูนย์กลางอยู่แล้ว, Spoke ไม่ต้อง Redirect เพราะมันไม่ใช่คนกลาง) และ
> ทั้งสองฝั่งต้องมีครบเสมอ — มีแค่ฝั่งเดียวจะยัง Hairpin เหมือนไม่ได้ทำอะไรเลย เป็นคำถาม
> Scenario-based ที่ ENCOR ชอบออกคู่กับหัวข้อ DMVPN Phase 1/2/3 เปรียบเทียบกัน

---

## Step 546 — กับดักข้อสอบ ENCOR ที่พบบ่อย + 5 คำถามฝึกฝน

### 546.1 BGP Best Path Selection Order — ตัวช่วยจำ

ข้อสอบ ENCOR ชอบให้โจทย์ BGP Table ที่มีหลาย Route ไปปลายทางเดียวกัน แล้วถามว่า Router จะเลือก
Path ไหนเป็น Best Path ลำดับการเทียบ (ย่อ 9 ขั้นแรกที่พบบ่อยที่สุด) ท่องจำด้วยคำย่อ
**"W-A-L-O-M-N-P-R-A"** (Weight-AS Path length... จำเป็นวลี "**Wise Angry Lions Occasionally
Meet Nasty Rude Aardvarks**" ก็ได้ตามสไตล์ที่ถูกใจ):

| ลำดับ | เกณฑ์ | ค่าที่ชนะ |
|---|---|---|
| 1 | **Weight** (Cisco Proprietary, Local เท่านั้น) | สูงกว่าชนะ |
| 2 | **Local Preference** | สูงกว่าชนะ |
| 3 | Route ที่ Router สร้างเอง (Locally Originated) | ชนะ Route ที่เรียนมาจากที่อื่น |
| 4 | **AS Path length** (สั้นกว่า) | สั้นกว่าชนะ (ใช้หลักการ Prepend โกงข้อนี้ได้ - Part 30) |
| 5 | **Origin Code** | IGP < EGP < Incomplete |
| 6 | **MED** (Multi-Exit Discriminator, ต่ำกว่าชนะ) | **เทียบกันเฉพาะ Neighbor AS เดียวกันเท่านั้น** (กับดัก Step 546.2) |
| 7 | eBGP > iBGP | eBGP ชนะ |
| 8 | **IGP Metric ต่ำสุดไปยัง Next-hop** | ต่ำกว่าชนะ |
| 9 | **Router-ID ต่ำสุด** (ตัวตัดสินสุดท้าย) | ต่ำกว่าชนะ |

> **กับดักคลาสสิก**: ข้อสอบมักเขียนโจทย์ให้ Weight สูงกว่าใน Path ที่ AS Path ยาวกว่ามาก แล้วถาม
> ว่า Router เลือก Path ไหน — คำตอบคือ **ยังเลือก Path ที่ Weight สูงกว่าอยู่ดี** เพราะ Weight
> เป็นขั้นที่ 1 ตัดสินก่อน AS Path (ขั้นที่ 4) เสมอ — หลักการเดียวกับ AD ที่ตัดสินก่อน Metric ใน CCNA

### 546.2 Redistribution Seed-Metric Trap (ทวนจาก Part 31)

> **กับดักที่พบบ่อยที่สุดในข้อสอบ Redistribution**: Redistribute เข้า **EIGRP** โดยไม่ระบุ
> `metric` (หรือ `default-metric`) — **Route จะไม่ถูกติดตั้งเข้า Topology Table เลยแบบเงียบๆ
> ไม่มี Error/Warning ใดๆ** (ต่างจาก OSPF ที่มี Default Seed Metric = 20 ให้อัตโนมัติถ้าไม่ระบุ)
> ข้อสอบชอบให้ Config ที่ดูสมบูรณ์แล้วถามว่า "ทำไม Route จาก OSPF ไม่ปรากฏใน `show ip route
> eigrp`" — คำตอบมักเป็นเรื่องนี้เสมอ ไม่ใช่ AD/Filter อื่น

### 546.3 QoS Policy-Map Ordering Trap (ทวนจาก Part 40)

> `policy-map` ประเมิน Class ตามลำดับที่พิมพ์ **จากบนลงล่าง เสมอ** และ**หยุดที่ Class แรกที่
> Match** — ถ้าเขียน Class ที่กว้างกว่า (เช่น Match ACL ที่ครอบคลุม Traffic เยอะ) ไว้**ก่อน**
> Class ที่เจาะจงกว่า (เช่น Match DSCP EF ของ VoIP) Traffic ที่ควรได้ Priority Queue จะถูกจับ
> เข้า Class กว้างก่อนแล้วไม่มีทางไปถึง Class VoIP อีกเลย — ข้อสอบชอบให้ `policy-map` ที่เรียง
> Class ผิดลำดับแล้วถามว่า VoIP Traffic ได้ Priority Queue จริงหรือไม่ — ต้องเรียง **เจาะจงที่สุด
> ไปกว้างที่สุด** เสมอ (`class-default` ต้องอยู่ล่างสุดเสมอเพราะเป็น "จับทุกอย่างที่เหลือ")

### 546.4 5 คำถามฝึกฝนสไตล์ ENCOR พร้อมเฉลยละเอียด

**คำถามที่ 1**

โจทย์: WAN-EDGE-1 เรียนรู้ Prefix `192.168.50.0/24` เดียวกันจาก 2 Path: Path A มี Local
Preference 200, AS Path ยาว 3 AS. Path B มี Local Preference 100, AS Path ยาว 1 AS (Prepend
มาจากอีกฝั่ง) จะเลือก Path ใดเป็น Best Path?

<details>
<summary>เฉลย</summary>

**Path A** เพราะ **Local Preference (ลำดับที่ 2)** ตัดสินก่อน **AS Path Length (ลำดับที่ 4)**
เสมอ — แม้ Path B จะมี AS Path สั้นกว่ามาก ก็ยังแพ้ตั้งแต่ขั้น Local Preference ก่อนที่จะได้
เทียบ AS Path เลย (Weight ไม่มีผลในที่นี้เพราะโจทย์ไม่ได้ระบุ = ใช้ Default 0 ทั้งคู่)
</details>

**คำถามที่ 2**

โจทย์: Engineer ตั้ง `redistribute ospf 1` ใต้ `router eigrp 100` โดยไม่ใส่ `metric` หรือ
`default-metric` ใดๆ ผลลัพธ์ที่คาดว่าจะเกิดคืออะไร?

<details>
<summary>เฉลย</summary>

**Route จาก OSPF จะไม่ถูกติดตั้งเข้า EIGRP Topology Table เลย** (ไม่มี Error แจ้งเตือนใดๆ) เพราะ
EIGRP ไม่มี Default Seed Metric ให้เหมือน OSPF/RIP — ต้องระบุ `metric <bw> <delay> <rel> <load>
<mtu>` ต่อท้าย `redistribute` โดยตรง หรือใช้ `default-metric` ใต้ `router eigrp` แยกต่างหาก
อย่างใดอย่างหนึ่งเสมอ
</details>

**คำถามที่ 3**

โจทย์: DMVPN Spoke ตัวหนึ่งมี `ip nhrp shortcut` ตั้งไว้ถูกต้อง แต่ Hub ทั้งคู่ไม่มี `ip nhrp
redirect` เลย Traffic ระหว่าง 2 Spoke จะเป็นอย่างไร?

<details>
<summary>เฉลย</summary>

**ยัง Hairpin ผ่าน Hub เหมือนเดิม** (เหมือน Phase 2) เพราะ `ip nhrp shortcut` ที่ Spoke เป็นแค่
"ความยินดีที่จะรับ Redirect" — ถ้า Hub ไม่ส่ง Redirect มาให้เลย (เพราะไม่มี `ip nhrp redirect`)
Spoke จะไม่มีอะไรให้ตอบรับ ต้องมีทั้งสองฝั่งพร้อมกันเสมอ (ดู Step 545 เต็มรูปแบบ)
</details>

**คำถามที่ 4**

โจทย์: CORE-SW1 ตั้ง CoPP Policy ใหม่ที่มี `class-default` เป็น `drop` (Deny ทุกอย่างที่ไม่ Match
Class ที่ตั้งไว้ก่อนหน้า) แต่ลืมสร้าง Class-map ให้ครอบคลุม OSPF Hello Packet เกิดอะไรขึ้น?

<details>
<summary>เฉลย</summary>

**OSPF Hello Packet ที่ไม่ตรง Class-map ใดๆที่ตั้งไว้ล่วงหน้าจะถูก Drop ทันทีที่ `class-default`**
เพราะ CoPP ทำงานกับ Traffic ที่มุ่งหน้า **CPU/Control-Plane ของอุปกรณ์เอง** โดยตรง (ไม่ใช่ Traffic
Transit ปกติ) — OSPF Neighbor จะ Dead Timer หมดแล้วหลุด Adjacency ทั้งหมด แนวทางแก้คือต้องมี
Class-map ที่ Match Routing Protocol Traffic ที่จำเป็น (OSPF/BGP/EIGRP Multicast+Unicast) ไว้ใน
Class ที่ Permit/ Rate-limit อย่างเหมาะสม **ก่อน** Apply Policy ใหม่เข้า `control-plane` เสมอ
(ทดสอบด้วย `show policy-map control-plane` ดู Drop Counter ก่อนประกาศใช้งานจริง)
</details>

**คำถามที่ 5**

โจทย์: Anycast RP ตั้งค่า `ip pim rp-address 1.1.1.100 10` บน CORE-SW1/CORE-SW2 ถูกต้องทั้งคู่
แต่ลืมตั้ง `ip msdp peer` ระหว่างกัน จะเกิดผลกระทบอย่างไรกับ Multicast Source ที่อยู่คนละครึ่ง
ของ Domain (เช่น Source ใกล้ CORE-SW1 แต่ Receiver ใกล้ CORE-SW2)?

<details>
<summary>เฉลย</summary>

**Receiver ที่ Join ผ่าน CORE-SW2 จะไม่เห็น Source ที่ Register ผ่าน CORE-SW1 เลย** เพราะ
Anycast RP (RP Address เดียวกันหลายตัว) ทำให้ Traffic จาก Source กับ Receiver **ไปหา RP คนละ
ตัวกันได้** (ตาม Path ที่ใกล้ที่สุดของแต่ละฝั่ง) — ถ้าไม่มี **MSDP** เชื่อม RP ทั้งสองตัวเข้าด้วย
กัน RP แต่ละตัวจะไม่รู้จัก Source-Active ของอีกฝั่งเลย ทำให้ Multicast Stream ไปไม่ถึง Receiver
ฝั่งตรงข้าม — Anycast RP กับ MSDP ต้องมาคู่กันเสมอ ไม่สามารถใช้ตัวใดตัวหนึ่งเดี่ยวๆได้
</details>

---

## Step 547 — Hands-on Lab Exam Simulation (จับเวลา 60 นาที)

### โจทย์ (อ่านให้ครบก่อนเริ่ม Config)

> ให้ Topology ย่อ 3 อุปกรณ์: `EDGE-A` (ต่อ ISP1 AS 64500 ที่ `192.0.2.1/30`, ต่อ Core ภายใน),
> `EDGE-B` (ต่อ ISP2 AS 64501 ที่ `192.0.2.5/30`, ต่อ Core ภายใน), `CORE-X` (L3, ต่อทั้ง EDGE-A
> และ EDGE-B, ทำ OSPF Area 0 กับทั้งคู่) ทั้ง 3 ตัวอยู่ใน iBGP AS 65100 ร่วมกัน จงคอนฟิกตาม
> ข้อกำหนดต่อไปนี้:
>
> 1. เปิด eBGP ที่ `EDGE-A`↔ISP1(AS64500) และ `EDGE-B`↔ISP2(AS64501), เปิด iBGP AS65100 แบบ
>    Full-Mesh ระหว่าง `EDGE-A`/`EDGE-B`/`CORE-X` (Loopback0 เป็น Router-ID/Update-Source)
> 2. Redistribute BGP Default Route เข้า OSPF Area 0 **เฉพาะ Default Route เท่านั้น** โดยที่
>    `EDGE-A` ต้องเป็น Primary (Metric ต่ำกว่า) และ `EDGE-B` เป็น Backup — **ต้อง Failover ได้
>    จริงถ้า ISP1 ล่ม** (ห้ามใช้ `always` แบบไม่ระมัดระวัง — ทวน Step 544)
> 3. สร้าง Class-map/Policy-map MQC บน `CORE-X`: Class `VOICE-RTP` (match dscp ef) ให้ Priority
>    Queue 20% ของ Bandwidth, Class `CRITICAL-DATA` (match dscp af31) ให้ Bandwidth Remaining
>    Percent 30, ส่วนที่เหลือเป็น `class-default` (fair-queue), Apply ที่ Interface ขาออกไปยัง
>    EDGE-A/EDGE-B ทั้งคู่
> 4. เปิด Multicast Routing + PIM Sparse-mode ทุก Interface ของ `CORE-X`, ตั้ง Static RP ที่
>    `CORE-X` เอง (Loopback0) เพื่อรองรับ Video Streaming ภายใน
> 5. ตั้ง CoPP บน `CORE-X`: Class `COPP-ROUTING` (match OSPF+BGP Traffic) ให้ Priority (ไม่ Drop),
>    Class-default ให้ Police ที่ 500kbps (ไม่ Drop ทั้งหมด แค่จำกัด Rate)

### Model Answer — EDGE-A

```
hostname EDGE-A
!
interface Loopback0
 ip address 9.9.9.1 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** eBGP to ISP1 (AS 64500) **
 ip address 192.0.2.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** OSPF Area0 to CORE-X **
 ip address 10.0.0.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 9.9.9.1
 default-information originate metric 10 metric-type 1
 passive-interface default
 no passive-interface GigabitEthernet0/0/1
!
router bgp 65100
 bgp router-id 9.9.9.1
 neighbor 192.0.2.2 remote-as 64500
 neighbor 9.9.9.2 remote-as 65100
 neighbor 9.9.9.2 update-source Loopback0
 neighbor 9.9.9.3 remote-as 65100
 neighbor 9.9.9.3 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 192.0.2.2 activate
  neighbor 9.9.9.2 activate
  neighbor 9.9.9.2 next-hop-self
  neighbor 9.9.9.3 activate
  neighbor 9.9.9.3 next-hop-self
 exit-address-family
!
end
```

### Model Answer — EDGE-B (ต่างเฉพาะ ISP/Metric)

```
hostname EDGE-B
!
interface Loopback0
 ip address 9.9.9.2 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** eBGP to ISP2 (AS 64501) **
 ip address 192.0.2.5 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** OSPF Area0 to CORE-X **
 ip address 10.0.0.5 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 9.9.9.2
 default-information originate metric 20 metric-type 1
 passive-interface default
 no passive-interface GigabitEthernet0/0/1
!
router bgp 65100
 bgp router-id 9.9.9.2
 neighbor 192.0.2.6 remote-as 64501
 neighbor 9.9.9.1 remote-as 65100
 neighbor 9.9.9.1 update-source Loopback0
 neighbor 9.9.9.3 remote-as 65100
 neighbor 9.9.9.3 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 192.0.2.6 activate
  neighbor 9.9.9.1 activate
  neighbor 9.9.9.1 next-hop-self
  neighbor 9.9.9.3 activate
  neighbor 9.9.9.3 next-hop-self
 exit-address-family
!
end
```

### Model Answer — CORE-X (QoS + Multicast + CoPP)

```
hostname CORE-X
!
ip multicast-routing
!
interface Loopback0
 ip address 9.9.9.3 255.255.255.255
 ip pim sparse-mode
!
interface GigabitEthernet0/0/0
 description ** to EDGE-A **
 ip address 10.0.0.2 255.255.255.252
 ip ospf 1 area 0
 ip pim sparse-mode
 service-policy output CORE-QOS-POLICY
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** to EDGE-B **
 ip address 10.0.0.6 255.255.255.252
 ip ospf 1 area 0
 ip pim sparse-mode
 service-policy output CORE-QOS-POLICY
 no shutdown
!
ip pim rp-address 9.9.9.3
!
class-map match-any VOICE-RTP
 match dscp ef
class-map match-any CRITICAL-DATA
 match dscp af31
!
policy-map CORE-QOS-POLICY
 class VOICE-RTP
  priority percent 20
 class CRITICAL-DATA
  bandwidth remaining percent 30
 class class-default
  fair-queue
!
router ospf 1
 router-id 9.9.9.3
 passive-interface default
 no passive-interface GigabitEthernet0/0/0
 no passive-interface GigabitEthernet0/0/1
!
router bgp 65100
 bgp router-id 9.9.9.3
 neighbor 9.9.9.1 remote-as 65100
 neighbor 9.9.9.1 update-source Loopback0
 neighbor 9.9.9.2 remote-as 65100
 neighbor 9.9.9.2 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 9.9.9.1 activate
  neighbor 9.9.9.2 activate
 exit-address-family
!
class-map match-any COPP-ROUTING
 match access-group name COPP-ROUTING-ACL
!
policy-map COPP-POLICY
 class COPP-ROUTING
  priority
 class class-default
  police 500000
   conform-action transmit
   exceed-action drop
!
ip access-list extended COPP-ROUTING-ACL
 permit ospf any any
 permit tcp any any eq bgp
 permit tcp any eq bgp any
!
control-plane
 service-policy input COPP-POLICY
!
end
```

### เกณฑ์ตรวจให้คะแนน (Self-Check)

| ข้อกำหนด | วิธี Verify |
|---|---|
| eBGP/iBGP Full-Mesh ขึ้นครบ | `show ip bgp summary` (State = ตัวเลข Prefix ไม่ใช่ Idle/Active) |
| EDGE-A เป็น Primary Default จริง | `show ip route 0.0.0.0` ที่ CORE-X → metric 10 ต้องมาก่อน |
| Failover ทำงานจริงถ้า ISP1 ล่ม | Shutdown eBGP Neighbor ที่ EDGE-A แล้วเช็ค `show ip route 0.0.0.0` เปลี่ยนไป EDGE-B |
| QoS Priority Queue ทำงานถูก Class | `show policy-map interface Gi0/0/0` (ดู Packet ที่ Match แต่ละ Class) |
| Multicast RP ตั้งถูก | `show ip pim rp mapping` |
| CoPP ไม่ Drop OSPF/BGP | `show policy-map control-plane` (Drop Counter ของ Class COPP-ROUTING ต้องเป็น 0) |

---

## Step 548 — เคล็ดลับวันสอบ ENCOR + แผนฝึกต่อเนื่อง

### 548.1 การบริหารเวลาในห้องสอบ (Time Management ระดับ ENCOR)

- ข้อสอบ ENCOR 350-401 มีเวลา **120 นาที** ประมาณ 90-110 ข้อ เหมือน CCNA แต่ **Blueprint กว้างกว่า
  มาก** (6 Domain ครอบคลุมทั้ง Routing, Wireless, Security, Automation) — ควรเผื่อเวลาหลังทำ
  Multiple Choice เสร็จไว้ทวนข้อที่ Flag ไว้อย่างน้อย 10-15 นาที
- **Simulation Question ระดับ ENCOR ซับซ้อนกว่า CCNA มาก** (มักผสมหลาย Technology ในโจทย์เดียว
  เช่น BGP+Redistribution+QoS พร้อมกันแบบ Step 547) — ให้ประเมินความยากตั้งแต่อ่านโจทย์รอบแรก
  ถ้าซับซ้อนเกินไปให้ Flag ข้ามไปทำข้อง่ายก่อนเสมอ อย่าติดอยู่ข้อเดียวจนหมดเวลา
- อ่านคำถามที่มีคำว่า "**MOST likely**", "**LEAST**", "**without disrupting**" ให้ละเอียด —
  ข้อสอบ ENCOR ชอบถามเชิง Design/Trade-off มากกว่า CCNA (ไม่ใช่แค่ "ใช้คำสั่งไหน" แต่ถาม "ทำไม
  วิธีนี้ดีกว่าวิธีอื่นในสถานการณ์นี้")

### 548.2 กลยุทธ์ทำข้อสอบ Simulation ที่ซับซ้อนขึ้น

1. **แยกโจทย์เป็น Checklist ก่อนพิมพ์คำสั่งแรกเสมอ** (เหมือน Step 547) — โจทย์ ENCOR มักมี 5-8
   Requirement ซ้อนกัน ถ้าไม่แยกเป็นรายการก่อนจะลืมบาง Requirement แน่นอน
2. Verify ทีละ Requirement ทันทีที่ทำเสร็จ **ไม่ใช่รอทำครบทุกอย่างแล้วค่อย Verify รวด** — เพราะ
   Requirement หลังอาจพึ่งพา Requirement ก่อนหน้า (เช่น Step 547 ข้อ 3 QoS ต้อง Apply บน
   Interface ที่ OSPF ทำงานแล้วจากข้อ 1-2)
3. ระมัดระวัง Requirement ที่เขียนว่า "ต้องไม่กระทบ Traffic ที่ทำงานอยู่แล้ว" (เช่น CoPP,
   Redistribution) — เป็นสัญญาณว่าข้อสอบต้องการเห็นว่าคุณเข้าใจ**ผลกระทบข้าม Layer** ไม่ใช่
   แค่ Syntax ถูก (ตรงกับ Troubleshooting Scenario ทั้ง Step 544-545)

### 548.3 หัวข้อที่ควรทวนก่อนสอบวันสุดท้าย (High-Yield Topics ระดับ ENCOR)

- BGP Best Path Selection Order เต็ม 9-13 ขั้น (Part 30, Step 546.1) — ข้อสอบออกบ่อยที่สุดใน
  Domain 3.0
- Redistribution Seed-Metric + Route-map/Tag ป้องกัน Feedback Loop (Part 31, Step 546.2)
- DMVPN Phase 1/2/3 เปรียบเทียบ + `nhrp redirect`/`shortcut` (Part 52-53, Step 545)
- 802.1X IBNS 2.0 State Machine + TrustSec SGT/SGACL Flow (Part 44)
- CoPP Class Design ที่ต้อง Permit Routing Protocol เสมอก่อน Deny/Police (Part 46, Step 546.4)
- YANG Data Model พื้นฐาน + ความต่าง NETCONF/RESTCONF (Part 49)

### 548.4 แผนฝึกต่อเนื่องหลังจบ Part นี้ (ก่อนไป Part 56 — ENARSI)

> **คำแนะนำเดียวกับ Part 25**: อย่าอ่าน Part 26-54 ซ้ำเฉยๆ — ให้**สร้าง Lab นี้ขึ้นมาใหม่ทั้งหมด
> ด้วยตัวเองใน Cisco Modeling Labs (CML) หรือ EVE-NG** (Packet Tracer ไม่รองรับ Feature ระดับ
> ENCOR หลายตัว เช่น BGP เต็มรูปแบบ, DMVPN, ZBFW, StackWise Virtual — ต้องใช้ CML/EVE-NG กับ
> IOS-XE Image จริง) แล้วเทียบกับ Final Running-Config ใน Step 550
>
> ทำซ้ำ Capstone Lab เต็มรูปแบบนี้อย่างน้อย **2-3 รอบ** ก่อนเข้าสอบจริง โดยเน้นจับเวลา
> Bring-up ทั้ง Runbook (Step 543) ให้เร็วขึ้นทุกรอบ และฝึก Troubleshoot Scenario ใหม่ที่ไม่ได้
> อยู่ใน Step 544-545 ด้วยตัวเอง (ลอง Break Config แบบสุ่มแล้วหา Root Cause เอง) — เป็นวิธีฝึก
> ที่ใกล้เคียงกับ Simulation Question ในห้องสอบจริงมากที่สุด

---

## Step 549 — Career Context: CCNP ENCOR ปลดล็อกอะไรได้บ้าง

### 549.1 CCNP ENCOR คือ "แกนกลาง" ของทุก CCNP Concentration

Cisco ออกแบบให้ **ENCOR (350-401) เป็นข้อสอบบังคับร่วม (Core Exam)** ของ CCNP ทุกสาย — สอบผ่าน
ENCOR เพียงตัวเดียว **ใช้ร่วมกับข้อสอบ Concentration ตัวใดก็ได้** จาก 7 สาย เพื่อได้ Certification
CCNP เต็มใบในสายนั้น:

| CCNP Concentration | ข้อสอบ Concentration | เหมาะกับ |
|---|---|---|
| Enterprise | 300-410 ENARSI (Part 56-70 ของหลักสูตรนี้) | Routing/Switching เชิงลึก — สายที่หลักสูตรนี้เดินต่อ |
| Data Center | 300-610 DCCOR | Data Center Fabric, ACI, UCS |
| Security | 300-710 SNCF (หรือเทียบเท่า) | Firewall/VPN/Security เชิงลึก (ต่อยอดจาก Part 43-46) |
| Collaboration | 300-810 CLCOR | Voice/Video/UC |
| Service Provider | 300-510 SPRI | MPLS/Service Provider Network |
| Wireless | 300-430 ENWLSD/ENWLSI | Wireless Design/Implementation (ต่อยอดจาก Part 36-37) |
| Cloud | 300-635 (DevNet Cloud) | Cloud-Native Automation |

> **นั่นแปลว่า**: ถ้าในอนาคตต้องการเปลี่ยนสายไป Security หรือ Data Center **ไม่ต้องสอบ ENCOR ใหม่
> เลย** — เอา ENCOR ที่ผ่านแล้วไปจับคู่กับ Concentration ใหม่ได้ทันที นี่คือเหตุผลที่ ENCOR สำคัญ
> ที่สุดในกลุ่ม CCNP ทั้งหมด และเป็นเหตุผลที่หลักสูตรนี้ให้เวลาสอน ENCOR ยาวถึง 30 Part

### 549.2 ENCOR คือ Foundation Exam ของ CCIE Enterprise Infrastructure ด้วย

**Cisco ยกเว้นข้อสอบ Written (Qualification Exam) ของ CCIE Enterprise Infrastructure ให้ทันที
ถ้าสอบผ่าน ENCOR 350-401 แล้ว** — พูดง่ายๆคือ **ทุก Part ที่เรียนใน Part 26-55 นี้ ใช้ต่อยอด
ตรงไปยัง CCIE Lab Exam ได้เลย** โดยไม่ต้องสอบ Written ซ้ำอีกรอบ (ถ้า ENCOR ยังไม่หมดอายุ)

```
เส้นทาง Certification จากจุดนี้ (หลังจบ Part 55):

  ทาง A: CCNP ENCOR (ผ่านแล้ว) + ENARSI (Part 56-70) ──► CCNP Enterprise เต็มใบ
                                                              │
                                                              ▼ (ฝึกฝีมือ Hands-on ต่อ)
                                                    CCIE Enterprise Infrastructure Lab
                                                    (Part 71-100 ของหลักสูตรนี้)

  ทาง B: CCNP ENCOR (ผ่านแล้ว) ──► ข้าม ENARSI ──► เตรียม CCIE Lab โดยตรง
         (ได้ Waiver Written Exam ของ CCIE แล้ว แต่ยังต้องมีพื้นฐาน Troubleshooting เชิงลึก
         ที่ ENARSI สอน — หลักสูตรนี้ยังแนะนำให้ผ่าน Part 56-70 ก่อนเสมอ ไม่ใช่ข้าม)
```

### 549.3 คำแนะนำของหลักสูตรนี้: ไปทางไหนต่อ?

> **แนะนำเส้นทาง A เสมอ (ENCOR → ENARSI → CCIE Lab)** แม้ ENCOR จะปลดล็อก CCIE Written Waiver
> ให้แล้วก็ตาม เพราะ **ENARSI (Part 56-70) สอน Troubleshooting Methodology เชิงลึกที่ CCIE Lab
> คาดหวังให้มีอยู่แล้วเป็นพื้นฐาน** — การข้ามตรงไป CCIE Lab โดยไม่ผ่าน ENARSI มักทำให้ติดปัญหา
> เรื่อง "รู้ Syntax แต่ Debug ไม่เป็นระบบ" ในห้องสอบ Lab จริงซึ่งมีเวลาจำกัดมาก (8 ชั่วโมง) —
> ความสามารถ Troubleshoot อย่างเป็นระบบสำคัญกว่าความรู้ Syntax เดี่ยวๆมากในระดับนี้
>
> ผู้ที่มีเป้าหมาย CCNP Enterprise เต็มใบ (ไม่ได้มุ่ง CCIE ทันที) ก็ควรไป ENARSI ต่อเช่นกัน
> เพราะเป็นข้อสอบบังคับของ CCNP Enterprise พอดี — **ทุกเส้นทางจากจุดนี้ นำไปสู่ Part 56 เสมอ**

---

## Step 550 — Final Running-Config Appendix (CORE-SW1, DIST-SW1)

นี่คือ **Running-Config ฉบับสมบูรณ์ที่สุด** ของ 2 อุปกรณ์ตัวแทน (Core + Distribution) หลังรวม
ทุก Feature จาก Part 26-54 เข้ากับ Baseline ระดับ CCNA (Part 1-25) ที่มีอยู่แล้ว — ใช้เป็นเอกสาร
อ้างอิงหลักเวลาทวนทั้งระดับ ENCOR หรือเทียบกับ Lab ที่สร้างเองตาม Step 548.4

> **หมายเหตุ Scope**: Feature ที่เป็นของ **WAN Edge โดยเฉพาะ** (BGP eBGP, ZBFW, VRF-Lite, GRE/
> IPsec, DMVPN Hub/Spoke) อยู่บน **WAN-EDGE-1/2 และ BRANCH-RTR** ไม่ใช่ CORE-SW1/DIST-SW1 —
> Config ชุดนี้แสดงเฉพาะ Feature ที่ตรงกับ**บทบาทจริง**ของ Core/Distribution Switch เท่านั้น
> (Step 541 มี Diagram เต็มที่แสดงว่า Feature ไหนอยู่อุปกรณ์ไหน) เช่นเดียวกับที่ Part 25 Step 250
> แสดงเฉพาะ CORE-SW1/2 และ DIST-SW1/2 ไม่ใช่ทุกอุปกรณ์ในหลักสูตร

### CORE-SW1 (Catalyst 9500, StackWise Virtual Pair กับ CORE-SW2, Domain 100)

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
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
! ===== Part 34: StackWise Virtual (SVL Domain 100) =====
stackwise-virtual
 domain 100
!
ip routing
ip multicast-routing
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
 description ** Router-ID / Management Loopback (Fabric Border+CP Node - Part 38) **
 ip address 1.1.1.1 255.255.255.255
 ip pim sparse-mode
!
! ===== Part 34: SVL Link (Control+Data Plane ระหว่าง Chassis) =====
interface TenGigabitEthernet1/0/1
 description ** SVL Link 1 to CORE-SW2 **
 stackwise-virtual link 1
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** SVL Link 2 to CORE-SW2 (redundant) **
 stackwise-virtual link 1
 no shutdown
!
interface TenGigabitEthernet1/0/3
 description ** Dual-Active Detection Link **
 stackwise-virtual dual-active-detection
 no shutdown
!
! ===== Part 26: OSPF Backbone (Area 0) + BFD =====
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 (Area 0, Legacy Po - เหลือไว้เพื่อ Reference) **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf 1 area 0
 ip ospf message-digest-key 1 md5 Ar3aZeroKey!
 bfd interval 150 min_rx 150 multiplier 3
 ip ospf bfd
 ip pim sparse-mode
 no shutdown
!
! ===== Part 34: MEC ไปยัง DIST-SW1 (Dual-homed — spans ทั้ง 2 Chassis ของ SVL Pair) =====
interface Port-channel11
 description ** MEC to DIST-SW1 (Dual-homed, spans SVL Pair) - Area 1 **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 ip ospf network point-to-point
 bfd interval 150 min_rx 150 multiplier 3
 ip ospf bfd
 ip pim sparse-mode
 service-policy output CORE-QOS-POLICY
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** MEC member 1 - to DIST-SW1 **
 no switchport
 channel-group 11 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/2
 description ** MEC member 2 - to DIST-SW1 (ผ่าน CORE-SW2 Chassis ทาง SVL) **
 no switchport
 channel-group 11 mode active
 no shutdown
!
! ===== Part 27: OSPFv3 (IPv6) - Per-link, Manual Router-ID =====
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 1.1.1.1
!
interface Port-channel1
 ipv6 address fd00:255:0::1/64
 ipv6 ospf 1 area 0
!
interface Port-channel11
 ipv6 address fd00:255:10::1/64
 ipv6 ospf 1 area 1
!
! ===== Part 40: MQC QoS (3-Class: VOICE/VIDEO/CRITICAL-DATA) =====
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41
class-map match-any CRITICAL-DATA
 match dscp af31
!
policy-map CORE-QOS-POLICY
 class VOICE
  priority percent 15
 class VIDEO
  bandwidth remaining percent 25
 class CRITICAL-DATA
  bandwidth remaining percent 30
 class class-default
  fair-queue
!
mls qos
mls qos trust dscp
!
! ===== Part 42: Multicast Advanced - Anycast RP + MSDP =====
ip pim rp-address 1.1.1.100 10
ip msdp peer 1.1.1.2 connect-source Loopback0
ip msdp originator-id Loopback0
!
! ===== Part 30: BGP iBGP AS65001 (peer WAN-EDGE-1/2 + CORE-SW2) =====
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 update-source Loopback0
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.21 activate
  neighbor 1.1.1.22 activate
 exit-address-family
!
! ===== Part 26: OSPF Advanced (NSSA ที่ Area 2 - ประกาศจากฝั่ง ABR) =====
router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 1 range 10.10.0.0 255.255.0.0
 area 2 nssa
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface Port-channel11
 default-information originate metric 1 metric-type 1
!
! ===== Part 46: Control Plane Policing =====
class-map match-any COPP-CRITICAL
 match access-group name COPP-ROUTING-ACL
class-map match-any COPP-IMPORTANT
 match access-group name COPP-MGMT-ACL
class-map match-any COPP-NORMAL
 match access-group name COPP-ICMP-ACL
!
policy-map COPP-POLICY
 class COPP-CRITICAL
  police 2000000 conform-action transmit exceed-action transmit
 class COPP-IMPORTANT
  police 1000000 conform-action transmit exceed-action drop
 class COPP-NORMAL
  police 500000 conform-action transmit exceed-action drop
 class class-default
  police 250000 conform-action transmit exceed-action drop
!
ip access-list extended COPP-ROUTING-ACL
 permit ospf any any
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit pim any any
!
ip access-list extended COPP-MGMT-ACL
 permit tcp any any eq 22
 permit udp host 10.10.99.51 any eq tacacs
 permit udp any any eq snmp
!
ip access-list extended COPP-ICMP-ACL
 permit icmp any any
!
control-plane
 service-policy input COPP-POLICY
!
! ===== Part 49: NETCONF/RESTCONF (สำหรับ Part 47-50 Automation/DNAC) =====
netconf-yang
restconf
ip http secure-server
!
! ===== Part 54: Model-Driven Telemetry (gRPC Dial-out ไปยัง NOC-SRV Collector) =====
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization
 source-address 1.1.1.1
 stream yang-push
 update-policy periodic 3000
 receiver ip address 10.10.99.50 57500 protocol grpc-tcp
!
ntp server 10.10.99.50
logging host 10.10.99.50
logging trap informational
snmp-server host 10.10.99.50 version 3 priv noc-svc
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

### DIST-SW1 (Catalyst 9300, Fabric Edge Node - MEC Dual-homed สู่ SVL Pair)

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
ip multicast-routing
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
spanning-tree vlan 10,20,99 priority 4096
!
interface Loopback0
 description ** Router-ID / Fabric Edge Node (SD-Access overlay - Part 38) **
 ip address 1.1.1.11 255.255.255.255
!
! ===== Part 34: MEC ไปยัง SVL Pair (Dual-homed แทน Single-homed เดิม) =====
interface Port-channel1
 description ** MEC to CORE SVL Pair (spans CORE-SW1 + CORE-SW2) - Area 1 **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 ip ospf network point-to-point
 bfd interval 150 min_rx 150 multiplier 3
 ip ospf bfd
 ip pim sparse-mode
 service-policy output DIST-QOS-POLICY
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** MEC member 1 - to CORE-SW1 **
 no switchport
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/2
 description ** MEC member 2 - to CORE-SW2 (ผ่าน SVL - มองเห็นเป็น Peer เดียวกับด้านบน) **
 no switchport
 channel-group 1 mode active
 no shutdown
!
! ===== Part 27: OSPFv3 =====
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 1.1.1.11
!
interface Port-channel1
 ipv6 address fd00:255:10::2/64
 ipv6 ospf 1 area 1
!
! ===== Part 8/18/35: SVI + Advanced HSRP (2 Track Object - MEC ทั้ง 2 เส้น) =====
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ipv6 address fd00:10:10:10::2/64
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 150
 standby 10 preempt delay minimum 60
 standby 10 track 1 decrement 30
 standby 10 track 2 decrement 30
 ip pim sparse-mode
 ip igmp version 3
 no shutdown
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
 ipv6 address fd00:10:10:20::2/64
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 100
 standby 20 preempt delay minimum 60
 standby 20 track 1 decrement 30
 standby 20 track 2 decrement 30
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
 standby version 2
 standby 99 ip 10.10.99.1
 standby 99 priority 150
 standby 99 preempt delay minimum 60
 standby 99 track 1 decrement 30
 standby 99 track 2 decrement 30
 no shutdown
!
track 1 interface TenGigabitEthernet1/1/1 line-protocol
track 2 interface TenGigabitEthernet1/1/2 line-protocol
!
! ===== Part 32: Policy-Based Routing (SLA-tracked Next-hop) =====
ip sla 1
 icmp-echo 10.255.20.1 source-interface Port-channel1
 frequency 10
ip sla schedule 1 life forever start-time now
track 10 ip sla 1 reachability
!
ip access-list extended SALES-VIA-DIST2-BACKUP
 permit ip 10.10.10.0 0.0.0.255 any
!
route-map PBR-SALES-BACKUP permit 10
 match ip address SALES-VIA-DIST2-BACKUP
 set ip next-hop verify-availability 10.255.20.1 10 track 10
!
interface Vlan10
 ip policy route-map PBR-SALES-BACKUP
!
! ===== Part 40: MQC QoS (Trust Boundary จาก Access, Enforce ที่นี่) =====
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41
class-map match-any CRITICAL-DATA
 match dscp af31
!
policy-map DIST-QOS-POLICY
 class VOICE
  priority percent 15
 class VIDEO
  bandwidth remaining percent 25
 class CRITICAL-DATA
  bandwidth remaining percent 30
 class class-default
  fair-queue
!
mls qos
mls qos trust dscp
!
! ===== Part 41-42: Multicast (IGMP ที่ SVI, Anycast RP แบบเดียวกับ Core) =====
ip pim rp-address 1.1.1.100 10
!
! ===== Part 26: OSPF Advanced =====
router ospf 1
 router-id 1.1.1.11
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 1 range 10.10.0.0 255.255.0.0
 passive-interface default
 no passive-interface Port-channel1
!
! ===== Part 44: TrustSec SXP (ส่ง SGT ไปยัง DIST-SW3 ที่ Enforce SGACL จริง) =====
cts sxp enable
cts sxp default password Cisco123!
cts sxp connection peer 1.1.1.13 source 1.1.1.11 password default mode local speaker
!
! ===== Part 49: NETCONF/RESTCONF =====
netconf-yang
restconf
ip http secure-server
!
! ===== Part 54: Model-Driven Telemetry =====
telemetry ietf subscription 102
 encoding encode-kvgpb
 filter xpath /ospf-oper:ospf-oper-data/ospf-neighbor
 source-address 1.1.1.11
 stream yang-push
 update-policy on-change
 receiver ip address 10.10.99.50 57500 protocol grpc-tcp
!
ntp server 10.10.99.50
logging host 10.10.99.50
snmp-server host 10.10.99.50 version 3 priv noc-svc
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

### เกณฑ์ Self-Check ก่อนปิด Part 55

| Feature Group | คำสั่ง Verify หลัก | ควรเห็นอะไร |
|---|---|---|
| StackWise Virtual | `show switch stack-ports summary`, `show stackwise-virtual` | CORE-SW1/2 เห็นเป็น 1 Switch |
| MEC | `show etherchannel summary` | Po Member มาจาก 2 Chassis กายภาพ แต่เป็น Po เดียว |
| OSPF Advanced + BFD | `show ip ospf neighbor`, `show bfd neighbors detail` | FULL ทุกคู่, BFD Session UP |
| iBGP AS65001 | `show ip bgp summary` | Full-Mesh, State = ตัวเลข Prefix |
| Redistribution (Default เท่านั้น) | `show ip route ospf \| include ^O\*E` | เห็น O*E1 เดียว ไม่เห็น Full Internet Table |
| Advanced HSRP | `show standby brief` | Track 2 Object ต่อ Group, Preempt Delay ตั้งไว้ |
| PBR | `show route-map PBR-SALES-BACKUP` | Policy Matches เพิ่มขึ้นตาม Traffic |
| Multicast Anycast RP | `show ip pim rp mapping` | RP = 1.1.1.100 ทั้ง CORE/DIST |
| CoPP | `show policy-map control-plane` | Drop Counter Class Routing = 0 |
| TrustSec SXP | `show cts sxp connections` | Connection State = On |
| Telemetry | `show telemetry ietf subscription all` | Subscription State = Active |

---

## จบระดับ CCNP ENCOR!

**ยินดีด้วย — คุณเรียนครบ Part 26-55 (Step 251-550) ของระดับ CCNP ENCOR (350-401) แล้ว**

ทวนภาพรวมสิ่งที่ผ่านมาทั้ง 30 Part:

- **Advanced Routing** (Part 26-32): OSPF Multi-area เชิงลึก, OSPFv3, EIGRP Advanced, BGP
  (eBGP/iBGP/Attributes), Redistribution, Policy-Based Routing
- **Advanced Campus Design** (Part 33-40): STP/Multilayer Design, StackWise Virtual/MEC,
  Advanced FHRP, Wireless Deep Dive x2, SD-Access, SD-WAN, Advanced QoS
- **Multicast** (Part 41-42): IGMP/PIM Fundamentals, Anycast RP + MSDP
- **Security Stack** (Part 43-46): Firepower/ISE, 802.1X+TrustSec, ZBFW, CoPP
- **Automation & Assurance** (Part 47-50): Python, Ansible, NETCONF/RESTCONF/YANG, DNA Center
- **WAN Overlay & Telemetry** (Part 51-54): VRF-Lite/GRE/IPsec, DMVPN Dual-Hub Phase 3,
  Network Assurance & Telemetry
- **Capstone** (Part 55 — Part นี้): รวมทุกอย่างเป็นภาพเดียว, Exam Blueprint Mapping,
  Troubleshooting Scenario ข้าม Layer, Hands-on Simulation, Career Roadmap

จาก Enterprise Lab เดิมที่เริ่มต้นด้วย 2 Core Switch ต่อกันด้วย EtherChannel ธรรมดาใน Part 1
ตอนนี้กลายเป็น **Enterprise Network เต็มรูปแบบ**: StackWise Virtual Core ที่ดูเหมือน 1 Switch,
Dual-ISP BGP พร้อม Failover ที่ Verify แล้วว่าทำงานจริง, Security Stack ครบวงจรตั้งแต่ Access
Port ถึง Control-Plane ของ Core, Automation Pipeline ที่พร้อม Integrate กับ DNA Center, และ
WAN Overlay ที่ขยายไปถึงสาขาผ่าน DMVPN Dual-Hub — นี่คือระดับความซับซ้อนที่ตรงกับงานวิศวกร
เครือข่าย Enterprise ระดับ Senior จริงในตลาด

**Part 56 จะเริ่มระดับ CCNP ENARSI (300-410)** — ไฟล์ถัดไปคือ
[part-056-eigrp-troubleshooting.md](part-056-eigrp-troubleshooting.md) หัวข้อ **Advanced EIGRP
Troubleshooting** ซึ่งจะพา EIGRP Demo Site ที่วางฐานไว้ตั้งแต่ Part 13/28/31 กลับมาเจาะลึกเป็น
โจทย์ Troubleshooting แบบเข้มข้นที่สุด เปิดฉากระดับ ENARSI ที่เน้น **"อะไรเสีย ทำไมเสีย แก้อย่างไร
เร็วที่สุด"** เป็นแกนหลักตลอด 15 Part ถัดไป (Part 56-70) ก่อนไปสู่ CCIE Enterprise Infrastructure
ใน Part 71-100

**เจอกันที่ Part 56 — CCNP ENARSI เริ่มต้นแล้ว**
