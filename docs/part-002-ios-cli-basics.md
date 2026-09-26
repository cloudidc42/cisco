# Part 2 — Cisco IOS CLI Basics & Initial Device Setup
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 11–20 จาก 1000**

> ต่อจาก [Part 1](part-001-networking-fundamentals.md) ที่ปูทฤษฎีและประกาศ Lab Topology หลัก
> Part นี้เราจะลงมือจริงกับ **Cisco IOS CLI** ตั้งแต่การเข้าใช้งานครั้งแรก จนถึงการเขียน
> Initial Configuration Script ที่ใช้กับอุปกรณ์ทุกตัวใน Lab Topology

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 11 | IOS CLI Modes & Navigation |
| 12 | Show Commands พื้นฐานสำหรับ Verify |
| 13 | Hostname, Banner, Password Security |
| 14 | Configuring SSH Access (แทน Telnet) |
| 15 | Interface Configuration Basics |
| 16 | IOS File System & Config Management |
| 17 | Backup/Restore Config และ IOS Image ผ่าน TFTP/SCP |
| 18 | CDP และ LLDP Neighbor Discovery |
| 19 | คำสั่ง Troubleshooting พื้นฐาน |
| 20 | Lab: Initial Setup Script สำหรับทุกอุปกรณ์ใน Topology |

---

## Step 11 — IOS CLI Modes & Navigation

Cisco IOS CLI มีโครงสร้าง Mode แบบเป็นลำดับชั้น (Hierarchical):

```
User EXEC Mode          Router>
      │  enable
      ▼
Privileged EXEC Mode     Router#
      │  configure terminal
      ▼
Global Configuration     Router(config)#
      │  interface GigabitEthernet0/1        │  router ospf 1        │  line vty 0 4
      ▼                                        ▼                       ▼
Interface Config      Router Config Mode    Line Config Mode
Router(config-if)#    Router(config-router)# Router(config-line)#
```

| Mode | Prompt | เข้าด้วยคำสั่ง | ออกด้วยคำสั่ง |
|---|---|---|---|
| User EXEC | `Router>` | (เริ่มต้น) | `exit` (ออกจาก session) |
| Privileged EXEC | `Router#` | `enable` | `disable` |
| Global Config | `Router(config)#` | `configure terminal` | `exit` / `end` |
| Interface Config | `Router(config-if)#` | `interface <type><number>` | `exit` |
| Sub-interface Config | `Router(config-subif)#` | `interface Gi0/1.10` | `exit` |
| Line Config | `Router(config-line)#` | `line console 0` / `line vty 0 4` | `exit` |
| Router Config | `Router(config-router)#` | `router ospf 1` | `exit` |
| VLAN Config | `Router(config-vlan)#` | `vlan 10` | `exit` |

### Editing Shortcuts ที่ใช้บ่อยที่สุด

| Shortcut | ผลลัพธ์ |
|---|---|
| `Tab` | เติมคำสั่งให้สมบูรณ์ (auto-complete) |
| `?` | แสดง help ของคำสั่ง ณ ตำแหน่งนั้น (context-sensitive help) |
| `Ctrl+A` | ไปต้นบรรทัด |
| `Ctrl+E` | ไปท้ายบรรทัด |
| `Ctrl+U` | ลบทั้งบรรทัด |
| `Ctrl+W` | ลบคำก่อนหน้า |
| `Ctrl+Shift+6` | หยุดคำสั่งที่กำลังรัน (เช่น ping ค้าง) |
| `↑ / Ctrl+P` | เรียกคำสั่งก่อนหน้าจาก history |
| `show history` | แสดงคำสั่งที่ใช้ล่าสุด (default เก็บ 10 บรรทัด) |

### ตัวอย่างการใช้ Context-Sensitive Help

```
Router# show ?
  aaa                Show AAA values
  access-lists        List access lists
  interfaces          Interface status and configuration
  ip                   IP information
  running-config       Current operating configuration
  version              System hardware and software status
  ...

Router# show ip ?
  interface     IP interface status and configuration
  ospf          OSPF related information
  route         IP routing table
  ...

Router# configure terminal
Router(config)# int g0/0/1<Tab>
Router(config)# interface GigabitEthernet0/0/1
```

### คำสั่งลบ Configuration: `no` และ `default`

```
Router(config-if)# no shutdown          ! เปิด interface (ยกเลิกการปิด)
Router(config-if)# no ip address        ! ลบ IP ออกจาก interface
Router(config)# no router ospf 1        ! ลบ OSPF process ทั้งหมด
Router(config-if)# default interface GigabitEthernet0/1   ! คืนค่า interface กลับเป็น default ทั้งหมด
```

---

## Step 12 — Show Commands พื้นฐานสำหรับ Verify

คำสั่ง `show` (verify command) เป็นคำสั่งที่ใช้มากที่สุดในการทำงานจริงและในข้อสอบ Cisco ทุกระดับ

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show version` | Hardware, IOS version, uptime, config register |
| `show running-config` (`sh run`) | Config ที่ทำงานอยู่ใน RAM ปัจจุบัน |
| `show startup-config` (`sh start`) | Config ที่จะโหลดตอน boot (เก็บใน NVRAM) |
| `show ip interface brief` (`sh ip int br`) | สถานะ IP/Up-Down ของทุก interface (คำสั่งที่ใช้บ่อยที่สุด) |
| `show interfaces` (`sh int`) | รายละเอียด interface: error, drop, duplex, speed |
| `show interfaces status` | สถานะ port แบบสรุป (นิยมใช้บน Switch) |
| `show cdp neighbors` | อุปกรณ์ Cisco ที่เชื่อมต่อโดยตรง |
| `show mac address-table` | MAC table บน Switch |
| `show vlan brief` | รายการ VLAN บน Switch |
| `show ip route` | Routing table |
| `show processes cpu` | การใช้งาน CPU |
| `show logging` | Log message ที่เก็บใน buffer |
| `show clock` | เวลาปัจจุบันของอุปกรณ์ |

### ตัวอย่าง Output จริงที่ต้องอ่านให้เป็น

```
Router# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     10.10.99.1      YES manual up                    up
GigabitEthernet0/1     unassigned      YES unset   administratively down down
Loopback0               1.1.1.1         YES manual up                    up
```

- **Status = administratively down** → interface ถูก `shutdown` ไว้ (แก้ด้วย `no shutdown`)
- **Status = down, Protocol = down** → ปัญหา Layer 1 (สายหลุด, ไม่ได้เสียบ)
- **Status = up, Protocol = down** → ปัญหา Layer 2 (encapsulation mismatch, keepalive)
- **Status = up, Protocol = up** → ใช้งานได้ปกติทั้ง L1/L2

---

## Step 13 — Hostname, Banner, Password Security

นี่คือ config พื้นฐานที่ควรทำกับ**อุปกรณ์ Cisco ทุกตัวทันทีหลัง unbox**:

```
Router> enable
Router# configure terminal

! 1. ตั้งชื่ออุปกรณ์ให้ตรงกับ Lab Topology (Part 1)
Router(config)# hostname CORE-SW1

! 2. ตั้ง Enable Secret (เข้ารหัสแบบ MD5/scrypt เสมอ ห้ามใช้ enable password แบบเก่า)
CORE-SW1(config)# enable secret Cisco123!

! 3. เข้ารหัส password ทั้งหมดที่เก็บใน running-config (type 7 - weak แต่ดีกว่าไม่เข้ารหัส)
CORE-SW1(config)# service password-encryption

! 4. Banner แจ้งเตือนก่อน login (legal notice) - ใช้ delimiter ที่ไม่ซ้ำกับข้อความ เช่น #
CORE-SW1(config)# banner motd #
Unauthorized access to this device is strictly prohibited.
All activities are logged and monitored. - Enterprise Lab
#

! 5. ตั้ง password สำหรับ console
CORE-SW1(config)# line console 0
CORE-SW1(config-line)# password Cisco123!
CORE-SW1(config-line)# login
CORE-SW1(config-line)# exec-timeout 10 0
CORE-SW1(config-line)# logging synchronous
CORE-SW1(config-line)# exit

! 6. ปิด DNS lookup อัตโนมัติ (ป้องกัน CLI ค้างเวลาพิมพ์คำสั่งผิด)
CORE-SW1(config)# no ip domain-lookup
```

> **หมายเหตุความปลอดภัย**: `enable secret` ใช้ hashing (irreversible) ส่วน `enable password`
> เป็น plaintext/weak encryption เท่านั้น — **ห้ามใช้ `enable password` ในระบบจริงเด็ดขาด**
> ถ้าตั้งทั้งสองอย่าง `enable secret` จะมีผลเหนือกว่าเสมอ

---

## Step 14 — Configuring SSH Access (แทน Telnet)

Telnet ส่งข้อมูล (รวม password) แบบ plaintext — **ห้ามใช้ในระบบจริง** ต้องใช้ SSH เท่านั้น

```
CORE-SW1(config)# hostname CORE-SW1
CORE-SW1(config)# ip domain-name lab.local

! สร้าง RSA key อย่างน้อย 1024 bit (แนะนำ 2048 bit สำหรับความปลอดภัยระดับ Enterprise)
CORE-SW1(config)# crypto key generate rsa modulus 2048
% The key modulus size is 2048 bits
% Generating 2048 bit RSA keys, keys will be non-exportable...
[OK] (elapsed time was 3 seconds)

! บังคับใช้ SSH version 2 เท่านั้น (v1 มีช่องโหว่)
CORE-SW1(config)# ip ssh version 2
CORE-SW1(config)# ip ssh time-out 60
CORE-SW1(config)# ip ssh authentication-retries 3

! สร้าง local user account สำหรับ login
CORE-SW1(config)# username admin privilege 15 secret StrongP@ssw0rd!
CORE-SW1(config)# username netops privilege 5 secret NetOpsP@ss123!

! ตั้งค่า VTY line ให้รับเฉพาะ SSH และใช้ local database ในการ authen
CORE-SW1(config)# line vty 0 15
CORE-SW1(config-line)# transport input ssh
CORE-SW1(config-line)# login local
CORE-SW1(config-line)# exec-timeout 10 0
CORE-SW1(config-line)# exit
```

**ทดสอบจากเครื่อง Client:**
```
ssh -l admin 10.10.99.10
```

**คำสั่ง verify SSH:**
```
CORE-SW1# show ip ssh
CORE-SW1# show ssh
```

> Privilege Level ของ Cisco IOS มีตั้งแต่ 0-15 (0=จำกัดสุด, 15=เทียบเท่า enable เต็ม)
> ค่า default: User EXEC = level 1, Privileged EXEC = level 15
> จะเรียนการปรับแต่ง privilege level แบบละเอียดร่วมกับ AAA ใน **Part 22**

---

## Step 15 — Interface Configuration Basics

```
CORE-SW1(config)# interface GigabitEthernet0/1
CORE-SW1(config-if)# description ** Uplink to CORE-SW2 - Po1 member **
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! ตัวอย่างตั้ง IP บน Router interface (Layer 3)
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** Link to ISP-RTR **
WAN-EDGE-1(config-if)# ip address 203.0.113.2 255.255.255.252
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit

! กำหนด speed/duplex แบบ manual (ปกติใช้ auto แต่บางกรณีต้อง fix)
CORE-SW1(config-if)# speed 1000
CORE-SW1(config-if)# duplex full
```

**กฎสำคัญ**: `description` เป็นแค่ comment (ไม่มีผลต่อการทำงาน) แต่ **จำเป็นมาก**สำหรับ
การทำ documentation และ troubleshooting — ทุก interface ที่ใช้งานจริงในหลักสูตรนี้จะมี
description ที่บอกว่าเชื่อมต่อกับอะไร ตาม convention: `** <คำอธิบาย> **`

---

## Step 16 — IOS File System & Config Management

Cisco IOS แยกไฟล์เก็บ config เป็น 2 ที่หลัก:

| Location | ชื่อ | เก็บที่ | หายเมื่อ reload? |
|---|---|---|---|
| RAM | `running-config` | Memory | หาย (volatile) |
| NVRAM | `startup-config` | Non-volatile flash | ไม่หาย |

```
! ดู config ปัจจุบันที่กำลังทำงาน
CORE-SW1# show running-config

! ดู config ที่จะโหลดตอน boot ครั้งถัดไป
CORE-SW1# show startup-config

! บันทึก running-config ลง startup-config (ต้องทำทุกครั้งหลัง config เสร็จ!)
CORE-SW1# copy running-config startup-config
Destination filename [startup-config]?
Building configuration...
[OK]

! ทางลัดที่ใช้บ่อยกว่า (เทียบเท่าคำสั่งข้างบน)
CORE-SW1# write memory
CORE-SW1# wr

! ลบ startup-config (คืนค่าโรงงาน - ระวังมาก!)
CORE-SW1# erase startup-config
CORE-SW1# reload
```

### IOS File System Commands

```
CORE-SW1# show flash:
CORE-SW1# dir flash:
CORE-SW1# dir all-filesystems
CORE-SW1# show file systems
```

**ตัวอย่าง output `dir flash:`**
```
Directory of flash:/
    1  -rw-  623421685   cat9k_iosxe.17.09.04a.SPA.bin
    2  -rw-       1847   private-config.text
```

---

## Step 17 — Backup/Restore Config และ IOS Image ผ่าน TFTP/SCP

### สำรอง config ไปเก็บที่ TFTP Server

```
CORE-SW1# copy running-config tftp:
Address or name of remote host []? 10.10.99.50
Destination filename [core-sw1-confg]? CORE-SW1-backup-2026-09-26.cfg
!!
1847 bytes copied in 0.512 secs
```

### กู้คืน config จาก TFTP

```
CORE-SW1# copy tftp: running-config
Address or name of remote host []? 10.10.99.50
Source filename []? CORE-SW1-backup-2026-09-26.cfg
Destination filename [running-config]?
```

### ใช้ SCP แทน TFTP (ปลอดภัยกว่า - แนะนำสำหรับ Production)

```
CORE-SW1(config)# ip scp server enable
CORE-SW1(config)# aaa new-model
CORE-SW1(config)# aaa authentication login default local
CORE-SW1(config)# aaa authorization exec default local
! จากเครื่อง Linux/Mac client:
!   scp admin@10.10.99.10:running-config ./core-sw1-backup.cfg
```

### อัปเกรด IOS Image ผ่าน TFTP

```
CORE-SW1# copy tftp: flash:
Address or name of remote host []? 10.10.99.50
Source filename []? cat9k_iosxe.17.12.01.SPA.bin
Destination filename [cat9k_iosxe.17.12.01.SPA.bin]?

! ตั้งให้ boot ด้วย image ใหม่
CORE-SW1(config)# boot system flash:cat9k_iosxe.17.12.01.SPA.bin
CORE-SW1# copy running-config startup-config
CORE-SW1# reload
```

> **Best Practice ระดับ Enterprise**: ทำ config backup อัตโนมัติทุกวันด้วย script (Python/Ansible)
> ไปเก็บที่ Git repository หรือ Configuration Management Database — เรียนละเอียดใน **Part 47-48**

---

## Step 18 — CDP และ LLDP Neighbor Discovery

**CDP (Cisco Discovery Protocol)** — Cisco proprietary, เปิด default บนอุปกรณ์ Cisco
**LLDP (Link Layer Discovery Protocol)** — มาตรฐานเปิด (IEEE 802.1AB) ใช้ได้กับอุปกรณ์ทุกยี่ห้อ

```
! CDP
CORE-SW1(config)# cdp run                    ! เปิดทั้งอุปกรณ์ (default เปิดอยู่แล้ว)
CORE-SW1(config-if)# cdp enable              ! เปิดเฉพาะ interface
CORE-SW1# show cdp neighbors
CORE-SW1# show cdp neighbors detail
CORE-SW1# show cdp interface GigabitEthernet0/1

! LLDP
CORE-SW1(config)# lldp run
CORE-SW1(config-if)# lldp transmit
CORE-SW1(config-if)# lldp receive
CORE-SW1# show lldp neighbors
CORE-SW1# show lldp neighbors detail
```

**ตัวอย่าง output `show cdp neighbors`:**
```
Capability Codes: R - Router, T - Trans-Bridge, B - Source-Route-Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone

Device ID        Local Intrfce    Holdtme    Capability   Platform   Port ID
CORE-SW2         Gig 0/1          178              S I    C9500     Gig 0/1
DIST-SW1         Gig 0/2          165              S I    C9300     Gig 0/1
```

> **ข้อควรระวังด้านความปลอดภัย**: บน interface ที่ต่อออก Internet/WAN ควรปิด CDP/LLDP
> (`no cdp enable`, `no lldp transmit`/`no lldp receive`) เพื่อไม่เปิดเผยข้อมูลอุปกรณ์ภายในให้บุคคลนอก

---

## Step 19 — คำสั่ง Troubleshooting พื้นฐาน

```
! ทดสอบ connectivity ระดับ Layer 3
Router# ping 10.10.99.1
Router# ping 10.10.99.1 repeat 100 size 1500 timeout 2      ! extended options แบบสั้น

! Extended ping (ใช้บ่อยใน CCNP/CCIE lab เพื่อระบุ source interface)
Router# ping
Protocol [ip]:
Target IP address: 10.10.30.5
Repeat count [5]: 10
Datasize [100]: 1500
Source address or interface: Loopback0

! Traceroute เพื่อดู path
Router# traceroute 10.10.30.5

! ดูการใช้ CPU/Memory
Router# show processes cpu sorted
Router# show memory statistics

! Debug (ใช้อย่างระมัดระวังใน Production - กิน CPU สูง)
Router# debug ip icmp
Router# terminal monitor          ! ให้เห็น debug message ผ่าน SSH/Telnet session ด้วย
Router# undebug all               ! ปิด debug ทั้งหมดทันที (คำสั่งฉุกเฉินที่ต้องจำ!)

! ดู log
Router# show logging
Router(config)# logging buffered 16384
```

> **กฎเหล็ก**: ทุกครั้งที่เปิด `debug` ต้องมีแผนปิดเสมอ ให้พิมพ์ `undebug all` เตรียมไว้ก่อนเปิด debug
> จริงเสมอ เพราะ debug message จำนวนมากสามารถทำให้ CPU อุปกรณ์พุ่งจนควบคุมไม่ได้

---

## Step 20 — Lab: Initial Setup Script สำหรับทุกอุปกรณ์ใน Topology

ต่อไปนี้คือ **Initial Configuration Script ฉบับสมบูรณ์** ที่ใช้กับทุกอุปกรณ์ใน Lab Topology
(Part 1) ตาม naming convention และ IP Plan ที่กำหนดไว้ — คัดลอกไปวางใน CLI ได้ทันที
(เป็น template สำหรับทุก part ถัดไปในหลักสูตร)

### Template: `initial-setup.txt` (ใช้กับทุกอุปกรณ์ เปลี่ยนแค่ hostname/IP)

```
enable
configure terminal
!
hostname CORE-SW1
no ip domain-lookup
ip domain-name lab.local
!
enable secret Cisco123!
service password-encryption
!
banner motd #
========================================================
  Enterprise Lab - Cisco CCNA/CCNP/CCIE Training Course
  Unauthorized access is prohibited.
========================================================
#
!
username admin privilege 15 secret StrongP@ssw0rd!
!
line console 0
 password Cisco123!
 login
 exec-timeout 10 0
 logging synchronous
!
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
!
interface Loopback0
 description ** Management/Router-ID Loopback **
 ip address 0.0.0.0 255.255.255.255
!
no cdp run
lldp run
!
end
copy running-config startup-config
```

### IP Loopback0 Plan (Router-ID) สำหรับทุกอุปกรณ์ Layer 3 ในหลักสูตร

| Device | Loopback0 IP |
|---|---|
| CORE-SW1 | 1.1.1.1/32 |
| CORE-SW2 | 1.1.1.2/32 |
| DIST-SW1 | 1.1.1.11/32 |
| DIST-SW2 | 1.1.1.12/32 |
| DIST-SW3 | 1.1.1.13/32 |
| DIST-SW4 | 1.1.1.14/32 |
| WAN-EDGE-1 | 1.1.1.21/32 |
| WAN-EDGE-2 | 1.1.1.22/32 |

> ทุก Part ถัดไปที่ทำ config Routing Protocol (OSPF/EIGRP/BGP) จะใช้ Loopback0 นี้เป็น
> Router-ID เสมอ เพื่อให้ debug/verify routing table อ่านง่ายและสอดคล้องกันทั้งหลักสูตร

---

## แบบฝึกหัดทวนความเข้าใจ Part 2

1. ความแตกต่างระหว่าง `enable password` และ `enable secret` คืออะไร และควรใช้อันไหน?
2. คำสั่งใดใช้บันทึก running-config ไปยัง NVRAM อย่างถาวร?
3. `show ip interface brief` แสดง Status = up แต่ Protocol = down หมายถึงปัญหาที่ Layer ใด?
4. เพราะเหตุใดจึงไม่ควรใช้ Telnet ในการบริหารอุปกรณ์จริง?
5. คำสั่งฉุกเฉินที่ต้องพิมพ์ทันทีเมื่อเปิด debug แล้วอุปกรณ์ทำงานหนักเกินไปคืออะไร?

**เฉลย:**
1. `enable secret` เข้ารหัสแบบ hash (ปลอดภัย) ส่วน `enable password` plaintext/weak — ควรใช้ `enable secret` เท่านั้น
2. `copy running-config startup-config` หรือ `write memory`
3. Layer 2 (Data Link) — เช่น encapsulation mismatch หรือ keepalive ไม่ตรงกัน
4. Telnet ส่ง traffic รวม password แบบ plaintext ไม่เข้ารหัส เสี่ยงถูกดักจับข้อมูล (sniffing)
5. `undebug all`

---

## สรุป Part 2

Part นี้ครอบคลุมการใช้งาน IOS CLI, การตั้งค่าความปลอดภัยเบื้องต้น, SSH, การจัดการไฟล์ config/IOS,
และปิดท้ายด้วย Initial Setup Script ที่จะใช้เป็นฐานของทุกอุปกรณ์ตลอดหลักสูตร ✅

**ไปต่อ:** [Part 3 — Ethernet Switching & VLAN →](part-003-ethernet-switching-vlan.md)
