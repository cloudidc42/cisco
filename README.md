# Cisco CCNA / CCNP / CCIE — หลักสูตรเขียนสคริปต์และคอนฟิกฉบับสมบูรณ์

หลักสูตรนี้สอน **การเขียนและพัฒนา config / script สำหรับอุปกรณ์ Cisco** ตั้งแต่ระดับพื้นฐาน (CCNA)
ไปจนถึงระดับมืออาชีพ (CCNP) และระดับโลก (CCIE Enterprise Infrastructure) แบบ **step-by-step
ตั้งแต่ขั้นตอนที่ 1 ถึง 1000** โดยแบ่งเป็น **100 Part** (Part ละ 10 Step)

ทุก Part เป็นไฟล์ Markdown แยกในโฟลเดอร์ [`docs/`](docs/) เนื้อหาแต่ละ Part ใช้งานได้จริง
ประกอบด้วย: ทฤษฎี, IOS/IOS-XE config เต็มรูปแบบ, คำสั่ง verify/troubleshoot, ASCII topology,
Lab scenario, และคำถามทวนความเข้าใจพร้อมเฉลย

> **สถานะ: เขียนจบครบสมบูรณ์แล้ว — 100/100 Part (Step 1–1000) ✅**
> หลักสูตรนี้ครอบคลุมตั้งแต่ CCNA (Part 1–25) → CCNP ENCOR (Part 26–55) → CCNP ENARSI
> (Part 56–70) → CCIE Enterprise Infrastructure (Part 71–100) แบบต่อเนื่องเป็น Lab เดียวกัน
> ตั้งแต่ต้นจนจบ รวมเนื้อหากว่า 108,000 บรรทัด ดูตาราง roadmap ด้านล่างสำหรับลิงก์ไปยังทุก Part

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
| 11 | 101–110 | [part-011-ospfv2-fundamentals.md](docs/part-011-ospfv2-fundamentals.md) | OSPFv2 Fundamentals | ✅ |
| 12 | 111–120 | [part-012-ospfv2-advanced.md](docs/part-012-ospfv2-advanced.md) | OSPFv2 Advanced (Areas, LSA, Auth) | ✅ |
| 13 | 121–130 | [part-013-eigrp-fundamentals.md](docs/part-013-eigrp-fundamentals.md) | EIGRP Fundamentals | ✅ |
| 14 | 131–140 | [part-014-access-control-lists.md](docs/part-014-access-control-lists.md) | Standard/Extended ACL | ✅ |
| 15 | 141–150 | [part-015-nat-pat.md](docs/part-015-nat-pat.md) | NAT / PAT | ✅ |
| 16 | 151–160 | [part-016-dhcp-dns.md](docs/part-016-dhcp-dns.md) | DHCP & DNS Services | ✅ |
| 17 | 161–170 | [part-017-ntp-syslog-snmp.md](docs/part-017-ntp-syslog-snmp.md) | NTP, Syslog, SNMP | ✅ |
| 18 | 171–180 | [part-018-fhrp.md](docs/part-018-fhrp.md) | HSRP / VRRP / GLBP | ✅ |
| 19 | 181–190 | [part-019-wan-technologies.md](docs/part-019-wan-technologies.md) | PPP, MPLS Intro, VPN Intro | ✅ |
| 20 | 191–200 | [part-020-wireless-fundamentals.md](docs/part-020-wireless-fundamentals.md) | Wireless LAN Fundamentals (WLC/AP) | ✅ |
| 21 | 201–210 | [part-021-switch-security.md](docs/part-021-switch-security.md) | Port Security, DHCP Snooping, DAI | ✅ |
| 22 | 211–220 | [part-022-aaa-device-hardening.md](docs/part-022-aaa-device-hardening.md) | AAA & Device Hardening | ✅ |
| 23 | 221–230 | [part-023-automation-basics.md](docs/part-023-automation-basics.md) | NETCONF/RESTCONF/Ansible เบื้องต้น | ✅ |
| 24 | 231–240 | [part-024-qos-fundamentals.md](docs/part-024-qos-fundamentals.md) | QoS Fundamentals | ✅ |
| 25 | 241–250 | [part-025-ccna-capstone-lab.md](docs/part-025-ccna-capstone-lab.md) | CCNA Capstone Lab & Exam Prep | ✅ |

### ระดับ CCNP ENCOR (Part 26–55)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 26 | 251–260 | [part-026-advanced-ospf.md](docs/part-026-advanced-ospf.md) | Advanced OSPF (Filtering, Stub Router, BFD) | ✅ |
| 27 | 261–270 | [part-027-ospfv3-ipv6.md](docs/part-027-ospfv3-ipv6.md) | OSPFv3 for IPv6 | ✅ |
| 28 | 271–280 | [part-028-eigrp-advanced.md](docs/part-028-eigrp-advanced.md) | EIGRP Advanced (Named Mode, Summarization) | ✅ |
| 29 | 281–290 | [part-029-bgp-fundamentals.md](docs/part-029-bgp-fundamentals.md) | BGP Fundamentals (eBGP) | ✅ |
| 30 | 291–300 | [part-030-bgp-advanced.md](docs/part-030-bgp-advanced.md) | BGP Advanced (iBGP, RR, Attributes, Path Selection) | ✅ |
| 31 | 301–310 | [part-031-route-redistribution.md](docs/part-031-route-redistribution.md) | Route Redistribution & Route-maps | ✅ |
| 32 | 311–320 | [part-032-policy-based-routing.md](docs/part-032-policy-based-routing.md) | Policy-Based Routing | ✅ |
| 33 | 321–330 | [part-033-advanced-stp-campus-design.md](docs/part-033-advanced-stp-campus-design.md) | Advanced STP & Campus Multilayer Design | ✅ |
| 34 | 331–340 | [part-034-stackwise-vss-mec.md](docs/part-034-stackwise-vss-mec.md) | StackWise / VSS / Multichassis EtherChannel | ✅ |
| 35 | 341–350 | [part-035-advanced-fhrp-campus-ha.md](docs/part-035-advanced-fhrp-campus-ha.md) | Advanced FHRP & Campus HA | ✅ |
| 36 | 351–360 | [part-036-wireless-architecture-deep-dive.md](docs/part-036-wireless-architecture-deep-dive.md) | Wireless Architecture Deep Dive (CAPWAP, Roaming) | ✅ |
| 37 | 361–370 | [part-037-wireless-security-advanced.md](docs/part-037-wireless-security-advanced.md) | Wireless Security Advanced | ✅ |
| 38 | 371–380 | [part-038-sd-access-fundamentals.md](docs/part-038-sd-access-fundamentals.md) | SD-Access Fundamentals | ✅ |
| 39 | 381–390 | [part-039-sd-wan-fundamentals.md](docs/part-039-sd-wan-fundamentals.md) | SD-WAN Fundamentals (Viptela) | ✅ |
| 40 | 391–400 | [part-040-advanced-qos.md](docs/part-040-advanced-qos.md) | Advanced QoS (MQC, Queuing, Shaping/Policing) | ✅ |
| 41 | 401–410 | [part-041-multicast-fundamentals.md](docs/part-041-multicast-fundamentals.md) | Multicast Fundamentals (IGMP, PIM-DM/SM) | ✅ |
| 42 | 411–420 | [part-042-multicast-advanced.md](docs/part-042-multicast-advanced.md) | Multicast Advanced (RP, MSDP, Anycast RP) | ✅ |
| 43 | 421–430 | [part-043-security-architecture-firepower-ise.md](docs/part-043-security-architecture-firepower-ise.md) | Security Architecture (Firepower, ISE Intro) | ✅ |
| 44 | 431–440 | [part-044-dot1x-trustsec.md](docs/part-044-dot1x-trustsec.md) | 802.1X & Cisco TrustSec | ✅ |
| 45 | 441–450 | [part-045-advanced-acl-zbfw.md](docs/part-045-advanced-acl-zbfw.md) | Advanced ACL & Zone-Based Firewall | ✅ |
| 46 | 451–460 | [part-046-control-plane-policing.md](docs/part-046-control-plane-policing.md) | Control Plane Policing & Infra Security | ✅ |
| 47 | 461–470 | [part-047-automation-python-advanced.md](docs/part-047-automation-python-advanced.md) | Automation: Python for Network Engineers | ✅ |
| 48 | 471–480 | [part-048-automation-ansible-advanced.md](docs/part-048-automation-ansible-advanced.md) | Automation: Ansible for Network Engineers | ✅ |
| 49 | 481–490 | [part-049-netconf-restconf-yang-deep-dive.md](docs/part-049-netconf-restconf-yang-deep-dive.md) | APIs, YANG, NETCONF/RESTCONF Deep Dive | ✅ |
| 50 | 491–500 | [part-050-dna-center.md](docs/part-050-dna-center.md) | Cisco DNA Center / Catalyst Center | ✅ |
| 51 | 501–510 | [part-051-vrf-lite-gre-ipsec.md](docs/part-051-vrf-lite-gre-ipsec.md) | VRF-Lite, GRE, IPsec VPN | ✅ |
| 52 | 511–520 | [part-052-dmvpn-fundamentals.md](docs/part-052-dmvpn-fundamentals.md) | DMVPN Fundamentals | ✅ |
| 53 | 521–530 | [part-053-dmvpn-advanced.md](docs/part-053-dmvpn-advanced.md) | DMVPN Advanced (Phase 3, Dual-Hub) | ✅ |
| 54 | 531–540 | [part-054-network-assurance-telemetry.md](docs/part-054-network-assurance-telemetry.md) | Network Assurance & Telemetry | ✅ |
| 55 | 541–550 | [part-055-encor-capstone-lab.md](docs/part-055-encor-capstone-lab.md) | ENCOR Capstone Lab & Exam Prep | ✅ |

### ระดับ CCNP ENARSI (Part 56–70)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 56 | 551–560 | [part-056-eigrp-troubleshooting.md](docs/part-056-eigrp-troubleshooting.md) | Advanced EIGRP Troubleshooting | ✅ |
| 57 | 561–570 | [part-057-ospf-troubleshooting.md](docs/part-057-ospf-troubleshooting.md) | Advanced OSPF Troubleshooting | ✅ |
| 58 | 571–580 | [part-058-bgp-troubleshooting.md](docs/part-058-bgp-troubleshooting.md) | Advanced BGP Troubleshooting | ✅ |
| 59 | 581–590 | [part-059-redistribution-troubleshooting.md](docs/part-059-redistribution-troubleshooting.md) | Route Redistribution Troubleshooting | ✅ |
| 60 | 591–600 | [part-060-dmvpn-vpn-troubleshooting.md](docs/part-060-dmvpn-vpn-troubleshooting.md) | DMVPN & VPN Troubleshooting | ✅ |
| 61 | 601–610 | [part-061-infra-security-troubleshooting.md](docs/part-061-infra-security-troubleshooting.md) | Infrastructure Security Troubleshooting | ✅ |
| 62 | 611–620 | [part-062-infra-services-troubleshooting.md](docs/part-062-infra-services-troubleshooting.md) | Infra Services (DHCP/NTP/Syslog) Troubleshooting | ✅ |
| 63 | 621–630 | [part-063-advanced-nat-scenarios.md](docs/part-063-advanced-nat-scenarios.md) | Advanced NAT Scenarios | ✅ |
| 64 | 631–640 | [part-064-path-control-pbr-troubleshooting.md](docs/part-064-path-control-pbr-troubleshooting.md) | Path Control & PBR Troubleshooting | ✅ |
| 65 | 641–650 | [part-065-multicast-troubleshooting.md](docs/part-065-multicast-troubleshooting.md) | Multicast Troubleshooting | ✅ |
| 66 | 651–660 | [part-066-wireless-troubleshooting.md](docs/part-066-wireless-troubleshooting.md) | Wireless Troubleshooting | ✅ |
| 67 | 661–670 | [part-067-automation-troubleshooting.md](docs/part-067-automation-troubleshooting.md) | Automation Troubleshooting | ✅ |
| 68 | 671–680 | [part-068-qos-troubleshooting.md](docs/part-068-qos-troubleshooting.md) | QoS Troubleshooting | ✅ |
| 69 | 681–690 | [part-069-mega-lab-1.md](docs/part-069-mega-lab-1.md) | Full Troubleshooting Mega-Lab #1 | ✅ |
| 70 | 691–700 | [part-070-enarsi-capstone-mock-exam.md](docs/part-070-enarsi-capstone-mock-exam.md) | ENARSI Capstone & Mock Exam | ✅ |

### ระดับ CCIE Enterprise Infrastructure (Part 71–100)

| Part | Step | หัวข้อ | สถานะ |
|---|---|---|---|
| 71 | 701–710 | [part-071-ccie-lab-blueprint-overview.md](docs/part-071-ccie-lab-blueprint-overview.md) | CCIE Lab Blueprint Overview | ✅ |
| 72 | 711–720 | [part-072-large-campus-fabric-design.md](docs/part-072-large-campus-fabric-design.md) | Large Campus Fabric Design (L2) | ✅ |
| 73 | 721–730 | [part-073-large-scale-routing-design.md](docs/part-073-large-scale-routing-design.md) | Large-Scale Routing Design (OSPF+BGP+EIGRP) | ✅ |
| 74 | 731–740 | [part-074-advanced-bgp-wan-edge.md](docs/part-074-advanced-bgp-wan-edge.md) | Advanced BGP for Enterprise WAN Edge | ✅ |
| 75 | 741–750 | [part-075-advanced-multicast-enterprise.md](docs/part-075-advanced-multicast-enterprise.md) | Advanced Multicast for Enterprise | ✅ |
| 76 | 751–760 | [part-076-sdwan-advanced-design.md](docs/part-076-sdwan-advanced-design.md) | SD-WAN Advanced Design (Viptela Deep Dive) | ✅ |
| 77 | 761–770 | [part-077-sd-access-advanced-design.md](docs/part-077-sd-access-advanced-design.md) | SD-Access Advanced Design & Deployment | ✅ |
| 78 | 771–780 | [part-078-security-integration-ise-trustsec-ftd-umbrella.md](docs/part-078-security-integration-ise-trustsec-ftd-umbrella.md) | Security Integration (ISE, TrustSec, FTD, Umbrella) | ✅ |
| 79 | 781–790 | [part-079-fullstack-automation-cicd.md](docs/part-079-fullstack-automation-cicd.md) | Full-Stack Automation (Python+Ansible+CI/CD) | ✅ |
| 80 | 791–800 | [part-080-model-driven-telemetry-streaming.md](docs/part-080-model-driven-telemetry-streaming.md) | Model-Driven Telemetry & Streaming | ✅ |
| 81 | 801–810 | [part-081-high-availability-nsf-sso-issu.md](docs/part-081-high-availability-nsf-sso-issu.md) | High Availability (NSF/SSO, ISSU) | ✅ |
| 82 | 811–820 | [part-082-end-to-end-enterprise-qos-design.md](docs/part-082-end-to-end-enterprise-qos-design.md) | End-to-End Enterprise QoS Design | ✅ |
| 83 | 821–830 | [part-083-enterprise-multicast-deep-dive.md](docs/part-083-enterprise-multicast-deep-dive.md) | Enterprise Multicast Deep Dive Advanced | ✅ |
| 84 | 831–840 | [part-084-vxlan-evpn-fundamentals.md](docs/part-084-vxlan-evpn-fundamentals.md) | VXLAN/EVPN Fundamentals | ✅ |
| 85 | 841–850 | [part-085-vxlan-evpn-multisite-advanced.md](docs/part-085-vxlan-evpn-multisite-advanced.md) | VXLAN/EVPN Multisite Advanced | ✅ |
| 86 | 851–860 | [part-086-mpls-l3vpn-fundamentals.md](docs/part-086-mpls-l3vpn-fundamentals.md) | MPLS L3VPN Fundamentals | ✅ |
| 87 | 861–870 | [part-087-mpls-l3vpn-advanced-wan-integration.md](docs/part-087-mpls-l3vpn-advanced-wan-integration.md) | MPLS L3VPN Advanced & WAN Integration | ✅ |
| 88 | 871–880 | [part-088-ipv6-enterprise-deployment.md](docs/part-088-ipv6-enterprise-deployment.md) | IPv6 Enterprise Deployment Deep Dive | ✅ |
| 89 | 881–890 | [part-089-enterprise-wireless-high-density.md](docs/part-089-enterprise-wireless-high-density.md) | Enterprise Wireless at Scale (High Density) | ✅ |
| 90 | 891–900 | [part-090-catalyst-9000-advanced-features.md](docs/part-090-catalyst-9000-advanced-features.md) | Catalyst 9000 Advanced Features | ✅ |
| 91 | 901–910 | [part-091-advanced-troubleshooting-methodology.md](docs/part-091-advanced-troubleshooting-methodology.md) | Advanced Troubleshooting Methodology | ✅ |
| 92 | 911–920 | [part-092-disaster-recovery-network-resilience.md](docs/part-092-disaster-recovery-network-resilience.md) | Disaster Recovery & Network Resilience | ✅ |
| 93 | 921–930 | [part-093-cloud-connectivity-aws-azure-gcp.md](docs/part-093-cloud-connectivity-aws-azure-gcp.md) | Cloud Connectivity (AWS/Azure/GCP, Cloud onRamp) | ✅ |
| 94 | 931–940 | [part-094-zero-trust-network-architecture.md](docs/part-094-zero-trust-network-architecture.md) | Zero Trust Network Architecture | ✅ |
| 95 | 941–950 | [part-095-ccie-lab-scenario-1-diagnose.md](docs/part-095-ccie-lab-scenario-1-diagnose.md) | CCIE Full Lab Scenario #1 (Diagnose Module) | ✅ |
| 96 | 951–960 | [part-096-ccie-lab-scenario-2-design.md](docs/part-096-ccie-lab-scenario-2-design.md) | CCIE Full Lab Scenario #2 (Design Module) | ✅ |
| 97 | 961–970 | [part-097-ccie-lab-scenario-3-deploy.md](docs/part-097-ccie-lab-scenario-3-deploy.md) | CCIE Full Lab Scenario #3 (Deploy Module) | ✅ |
| 98 | 971–980 | [part-098-ccie-lab-scenario-4-optimize.md](docs/part-098-ccie-lab-scenario-4-optimize.md) | CCIE Full Lab Scenario #4 (Optimize Module) | ✅ |
| 99 | 981–990 | [part-099-capstone-build-from-scratch.md](docs/part-099-capstone-build-from-scratch.md) | Capstone: Build Enterprise Network From Scratch | ✅ |
| 100 | 991–1000 | [part-100-career-mastery-exam-strategy.md](docs/part-100-career-mastery-exam-strategy.md) | Career Mastery: Exam Strategy & Real-World Scenarios | ✅ |

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
