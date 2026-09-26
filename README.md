# Cisco CCNA / CCNP / CCIE — หลักสูตรเขียนสคริปต์และคอนฟิกฉบับสมบูรณ์

หลักสูตรนี้สอน **การเขียนและพัฒนา config / script สำหรับอุปกรณ์ Cisco** ตั้งแต่ระดับพื้นฐาน (CCNA)
ไปจนถึงระดับมืออาชีพ (CCNP) และระดับโลก (CCIE Enterprise Infrastructure) แบบ **step-by-step
ตั้งแต่ขั้นตอนที่ 1 ถึง 1000** โดยแบ่งเป็น **100 Part** (Part ละ 10 Step)

ทุก Part เป็นไฟล์ Markdown แยกในโฟลเดอร์ [`docs/`](docs/) เนื้อหาแต่ละ Part ใช้งานได้จริง
ประกอบด้วย: ทฤษฎี, IOS/IOS-XE config เต็มรูปแบบ, คำสั่ง verify/troubleshoot, ASCII topology,
Lab scenario, และคำถามทวนความเข้าใจพร้อมเฉลย

> **สถานะ:** โครงการนี้เป็นงานต่อเนื่องขนาดใหญ่ — เนื้อหาจะถูกเพิ่มเข้ามาเรื่อยๆ ทีละ Part
> จนครบ 100 Part (1000 Step) ดูตาราง roadmap ด้านล่างเพื่อดูสถานะความคืบหน้าล่าสุด
> (✅ = เขียนแล้ว, ⏳ = กำลังจะเขียนต่อไป)

> **เอกสารอ้างอิงกลาง**: [docs/00-ip-address-plan.md](docs/00-ip-address-plan.md) — IP Address Plan,
> Loopback0, Transit Link, Device Platform ที่ใช้อ้างอิงสอดคล้องกันตั้งแต่ Part 11 เป็นต้นไป

## โครงสร้างหลักสูตร

| ระดับ | Part | Step | หัวข้อหลัก |
|---|---|---|---|
| **CCNA (200-301)** | 1–25 | 1–250 | พื้นฐานเครือข่าย, Switching, Routing, Services, Security, Wireless, Automation พื้นฐาน |
| **CCNP ENCOR (350-401)** | 26–55 | 251–550 | Advanced Routing/Switching, Wireless, SD-Access/SD-WAN, QoS, Multicast, Security, Automation |
| **CCNP ENARSI (300-410)** | 56–70 | 551–700 | Advanced Troubleshooting ทุกโดเมน |
| **CCIE Enterprise Infrastructure** | 71–100 | 701–1000 | Design/Deploy/Optimize ระดับ Enterprise, VXLAN/EVPN, MPLS L3VPN, Automation เต็มสาย, Capstone |

## Roadmap ฉบับเต็ม (100 Part / 1000 Step)

### ระดับ CCNA — พื้นฐาน (Part 1–25)

| Part | Step | ไฟล์ | หัวข้อ | สถานะ |
|---|---|---|---|---|
| 1 | 1–10 | [part-001-networking-fundamentals.md](docs/part-001-networking-fundamentals.md) | Networking Fundamentals, OSI/TCP-IP Model | ✅ |
| 2 | 11–20 | [part-002-ios-cli-basics.md](docs/part-002-ios-cli-basics.md) | Cisco IOS CLI, Initial Device Setup | ✅ |
| 3 | 21–30 | [part-003-ethernet-switching-vlan.md](docs/part-003-ethernet-switching-vlan.md) | Ethernet Switching, VLAN | ✅ |
| 4 | 31–40 | [part-004-trunking-vtp-dtp.md](docs/part-004-trunking-vtp-dtp.md) | Trunking, VTP, DTP | ✅ |
| 5 | 41–50 | [part-005-stp-rstp-mstp.md](docs/part-005-stp-rstp-mstp.md) | STP / RSTP / MSTP | ✅ |
| 6 | 51–60 | [part-006-etherchannel.md](docs/part-006-etherchannel.md) | EtherChannel (LACP/PAgP) | ✅ |
| 7 | 61–70 | [part-007-ipv4-addressing-subnetting.md](docs/part-007-ipv4-addressing-subnetting.md) | IPv4 Addressing & Subnetting | ✅ |
| 8 | 71–80 | [part-008-ipv6-fundamentals.md](docs/part-008-ipv6-fundamentals.md) | IPv6 Fundamentals | ✅ |
| 9 | 81–90 | [part-009-static-routing.md](docs/part-009-static-routing.md) | Static Routing | ✅ |
| 10 | 91–100 | [part-010-inter-vlan-routing.md](docs/part-010-inter-vlan-routing.md) | Inter-VLAN Routing (RoaS, SVI) | ✅ |
| 11 | 101–110 | part-011-ospfv2-fundamentals.md | OSPFv2 Fundamentals | ⏳ |
| 12 | 111–120 | part-012-ospfv2-advanced.md | OSPFv2 Advanced (Areas, LSA, Auth) | ⏳ |
| 13 | 121–130 | part-013-eigrp-fundamentals.md | EIGRP Fundamentals | ⏳ |
| 14 | 131–140 | part-014-access-control-lists.md | Standard/Extended ACL | ⏳ |
| 15 | 141–150 | part-015-nat-pat.md | NAT / PAT | ⏳ |
| 16 | 151–160 | part-016-dhcp-dns.md | DHCP & DNS Services | ⏳ |
| 17 | 161–170 | part-017-ntp-syslog-snmp.md | NTP, Syslog, SNMP | ⏳ |
| 18 | 171–180 | part-018-fhrp.md | HSRP / VRRP / GLBP | ⏳ |
| 19 | 181–190 | part-019-wan-technologies.md | PPP, MPLS Intro, VPN Intro | ⏳ |
| 20 | 191–200 | part-020-wireless-fundamentals.md | Wireless LAN Fundamentals (WLC/AP) | ⏳ |
| 21 | 201–210 | part-021-switch-security.md | Port Security, DHCP Snooping, DAI | ⏳ |
| 22 | 211–220 | part-022-aaa-device-hardening.md | AAA & Device Hardening | ⏳ |
| 23 | 221–230 | part-023-automation-basics.md | NETCONF/RESTCONF/Ansible เบื้องต้น | ⏳ |
| 24 | 231–240 | part-024-qos-fundamentals.md | QoS Fundamentals | ⏳ |
| 25 | 241–250 | part-025-ccna-capstone-lab.md | CCNA Capstone Lab & Exam Prep | ⏳ |

### ระดับ CCNP ENCOR (Part 26–55)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 26 | 251–260 | Advanced OSPF (Multi-area, LSA Type 1-7) | ⏳ |
| 27 | 261–270 | OSPFv3 for IPv6 | ⏳ |
| 28 | 271–280 | EIGRP Advanced (Named Mode, Summarization) | ⏳ |
| 29 | 281–290 | BGP Fundamentals (eBGP) | ⏳ |
| 30 | 291–300 | BGP Advanced (iBGP, RR, Attributes, Path Selection) | ⏳ |
| 31 | 301–310 | Route Redistribution & Route-maps | ⏳ |
| 32 | 311–320 | Policy-Based Routing | ⏳ |
| 33 | 321–330 | Advanced STP & Campus Multilayer Design | ⏳ |
| 34 | 331–340 | StackWise / VSS / Multichassis EtherChannel | ⏳ |
| 35 | 341–350 | Advanced FHRP & Campus HA | ⏳ |
| 36 | 351–360 | Wireless Architecture Deep Dive (CAPWAP, Roaming) | ⏳ |
| 37 | 361–370 | Wireless Security Advanced | ⏳ |
| 38 | 371–380 | SD-Access Fundamentals | ⏳ |
| 39 | 381–390 | SD-WAN Fundamentals (Viptela) | ⏳ |
| 40 | 391–400 | Advanced QoS (MQC, Queuing, Shaping/Policing) | ⏳ |
| 41 | 401–410 | Multicast Fundamentals (IGMP, PIM-DM/SM) | ⏳ |
| 42 | 411–420 | Multicast Advanced (RP, MSDP, Anycast RP) | ⏳ |
| 43 | 421–430 | Security Architecture (Firepower, ISE Intro) | ⏳ |
| 44 | 431–440 | 802.1X & Cisco TrustSec | ⏳ |
| 45 | 441–450 | Advanced ACL & Zone-Based Firewall | ⏳ |
| 46 | 451–460 | Control Plane Policing & Infra Security | ⏳ |
| 47 | 461–470 | Automation: Python for Network Engineers | ⏳ |
| 48 | 471–480 | Automation: Ansible for Network Engineers | ⏳ |
| 49 | 481–490 | APIs, YANG, NETCONF/RESTCONF Deep Dive | ⏳ |
| 50 | 491–500 | Cisco DNA Center / Catalyst Center | ⏳ |
| 51 | 501–510 | VRF-Lite, GRE, IPsec VPN | ⏳ |
| 52 | 511–520 | DMVPN Fundamentals | ⏳ |
| 53 | 521–530 | DMVPN Advanced (Phase 3, Dual-Hub) | ⏳ |
| 54 | 531–540 | Network Assurance & Telemetry | ⏳ |
| 55 | 541–550 | ENCOR Capstone Lab & Exam Prep | ⏳ |

### ระดับ CCNP ENARSI (Part 56–70)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 56 | 551–560 | Advanced EIGRP Troubleshooting | ⏳ |
| 57 | 561–570 | Advanced OSPF Troubleshooting | ⏳ |
| 58 | 571–580 | Advanced BGP Troubleshooting | ⏳ |
| 59 | 581–590 | Route Redistribution Troubleshooting | ⏳ |
| 60 | 591–600 | DMVPN & VPN Troubleshooting | ⏳ |
| 61 | 601–610 | Infrastructure Security Troubleshooting | ⏳ |
| 62 | 611–620 | Infra Services (DHCP/NTP/Syslog) Troubleshooting | ⏳ |
| 63 | 621–630 | Advanced NAT Scenarios | ⏳ |
| 64 | 631–640 | Path Control & PBR Troubleshooting | ⏳ |
| 65 | 641–650 | Multicast Troubleshooting | ⏳ |
| 66 | 651–660 | Wireless Troubleshooting | ⏳ |
| 67 | 661–670 | Automation Troubleshooting | ⏳ |
| 68 | 671–680 | QoS Troubleshooting | ⏳ |
| 69 | 681–690 | Full Troubleshooting Mega-Lab #1 | ⏳ |
| 70 | 691–700 | ENARSI Capstone & Mock Exam | ⏳ |

### ระดับ CCIE Enterprise Infrastructure (Part 71–100)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 71 | 701–710 | CCIE Lab Blueprint Overview | ⏳ |
| 72 | 711–720 | Large Campus Fabric Design (L2) | ⏳ |
| 73 | 721–730 | Large-Scale Routing Design (OSPF+BGP+EIGRP) | ⏳ |
| 74 | 731–740 | Advanced BGP for Enterprise WAN Edge | ⏳ |
| 75 | 741–750 | Advanced Multicast for Enterprise | ⏳ |
| 76 | 751–760 | SD-WAN Advanced Design (Viptela Deep Dive) | ⏳ |
| 77 | 761–770 | SD-Access Advanced Design & Deployment | ⏳ |
| 78 | 771–780 | Security Integration (ISE, TrustSec, FTD, Umbrella) | ⏳ |
| 79 | 781–790 | Full-Stack Automation (Python+Ansible+CI/CD) | ⏳ |
| 80 | 791–800 | Model-Driven Telemetry & Streaming | ⏳ |
| 81 | 801–810 | High Availability (NSF/SSO, ISSU) | ⏳ |
| 82 | 811–820 | End-to-End Enterprise QoS Design | ⏳ |
| 83 | 821–830 | Enterprise Multicast Deep Dive Advanced | ⏳ |
| 84 | 831–840 | VXLAN/EVPN Fundamentals | ⏳ |
| 85 | 841–850 | VXLAN/EVPN Multisite Advanced | ⏳ |
| 86 | 851–860 | MPLS L3VPN Fundamentals | ⏳ |
| 87 | 861–870 | MPLS L3VPN Advanced & WAN Integration | ⏳ |
| 88 | 871–880 | IPv6 Enterprise Deployment Deep Dive | ⏳ |
| 89 | 881–890 | Enterprise Wireless at Scale (High Density) | ⏳ |
| 90 | 891–900 | Catalyst 9000 Advanced Features | ⏳ |
| 91 | 901–910 | Advanced Troubleshooting Methodology | ⏳ |
| 92 | 911–920 | Disaster Recovery & Network Resilience | ⏳ |
| 93 | 921–930 | Cloud Connectivity (AWS/Azure/GCP, Cloud onRamp) | ⏳ |
| 94 | 931–940 | Zero Trust Network Architecture | ⏳ |
| 95 | 941–950 | CCIE Full Lab Scenario #1 (Diagnose Module) | ⏳ |
| 96 | 951–960 | CCIE Full Lab Scenario #2 (Design Module) | ⏳ |
| 97 | 961–970 | CCIE Full Lab Scenario #3 (Deploy Module) | ⏳ |
| 98 | 971–980 | CCIE Full Lab Scenario #4 (Optimize Module) | ⏳ |
| 99 | 981–990 | Capstone: Build Enterprise Network From Scratch | ⏳ |
| 100 | 991–1000 | Career Mastery: Exam Strategy & Real-World Scenarios | ⏳ |

## วิธีใช้หลักสูตรนี้

1. เริ่มจาก Part 1 ไปตามลำดับ ห้ามข้าม เพราะแต่ละ Part อ้างอิง topology/config จาก Part ก่อนหน้า
2. ทำ Lab ทุก Part จริงด้วย GNS3 / EVE-NG / Cisco Packet Tracer / Cisco Modeling Labs (CML)
3. ตอบคำถามทวนความเข้าใจท้ายแต่ละ Part ก่อนไป Part ถัดไป
4. ระดับ CCNP/CCIE ให้ย้อนกลับไปทวน Part CCNA ที่เกี่ยวข้องเสมอเมื่อไม่แน่ใจพื้นฐาน

## Lab Environment ที่ใช้ตลอดหลักสูตร

ตั้งแต่ Part 1 เราจะสร้างและใช้ **Enterprise Lab Topology เดียวกันต่อเนื่องทั้งหลักสูตร**
(ขยายเพิ่มอุปกรณ์ทีละ Part) ประกอบด้วย: 2x Core Switch (Catalyst 9500 rec.), 4x Distribution
Switch, 8x Access Switch, 3x Edge Router (ISR 4000 / CSR1000v), 2x WAN Router, 1x WLC + AP,
1x Firewall (ASA/FTD) — ดูรายละเอียด topology เต็มใน [part-001](docs/part-001-networking-fundamentals.md)
