# Part 60 — DMVPN & VPN Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 591–600 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 59 — Route Redistribution Troubleshooting](part-059-redistribution-troubleshooting.md)
> ที่เจาะปัญหา Redistribution/Routing Loop ระดับ Control-Plane ล้วนๆ Part นี้จะดึงกลับไปที่
> **DMVPN Dual-Hub Phase 3 Lab** ที่สร้างไว้เต็มรูปแบบใน [Part 51](part-051-vrf-lite-gre-ipsec.md)
> (GRE-over-IPsec พื้นฐาน), [Part 52](part-052-dmvpn-fundamentals.md) (DMVPN Phase 1/2, NHRP,
> mGRE) และ [Part 53](part-053-dmvpn-advanced.md) (DMVPN Phase 3, Dual-Hub, NHRP
> Redirect/Shortcut) — คราวนี้เราจะ**ทำให้ Lab นั้นพังทีละจุดโดยตั้งใจ** แล้วฝึก Diagnose แบบที่
> ข้อสอบ **CCNP ENARSI (300-410)** ชอบถามที่สุด: given ปัญหาที่ปลายทาง, ให้หา Root Cause จาก
> `show`/`debug` output แล้วแก้ให้ถูกจุดโดยไม่แก้อะไรเกินความจำเป็น
>
> ทุก Scenario ใน Part นี้ใช้ **Topology และค่า Config เดิมทุกประการจาก Part 51-53** —
> **WAN-EDGE-1** (Hub หลัก, `Tunnel0` = `172.16.100.1/24`, NBMA = `203.0.113.2`),
> **WAN-EDGE-2** (Hub สำรอง, `Tunnel0` = `172.16.100.2/24`, NBMA = `203.0.113.6`),
> **BRANCH-RTR** (Spoke1, `Tunnel0` = `172.16.100.11/24`, NBMA = `198.51.100.2`, LAN
> `10.20.1.0/24`), **BRANCH2-RTR** (Spoke2, `Tunnel0` = `172.16.100.12/24`, NBMA =
> `198.51.100.6`, LAN `10.20.2.0/24`) — NHRP Network-ID/Tunnel Key = `100`, IPsec Profile =
> `DMVPN-PROFILE` (Wildcard PSK `Cisco@DMVPN123`, Transform-Set `DMVPN-TSET` = `esp-aes 256
> esp-sha256-hmac` mode transport), Routing Protocol = **OSPF Process 100, Area 51,
> point-to-multipoint** บน Tunnel0 — ถ้าจำ Config เดิมไม่ได้ ให้เปิด Part 51-53 ควบคู่กันไปด้วย

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 591 | DMVPN/VPN Troubleshooting Methodology — Layered Checks จาก Underlay ถึง Data-Plane |
| 592 | Scenario 1 — IKE Phase 1 (ISAKMP) ล้มเหลวจาก Pre-Shared Key/Policy Mismatch |
| 593 | Scenario 2 — IPsec Phase 2 ล้มเหลวจาก Transform-Set Mismatch |
| 594 | Scenario 3 — NHRP Registration ล้มเหลวเพราะ NHS Address ผิด |
| 595 | Scenario 4 — mGRE/Point-to-Point Tunnel Mode Mismatch |
| 596 | Scenario 5 — Phase 3 Spoke-to-Spoke ไม่ Shortcut (Redirect/Shortcut ขาดหาย) |
| 597 | Scenario 6 — Routing Protocol ไม่ Form Neighbor เพราะ Network-Type Mismatch |
| 598 | Scenario 7 — Asymmetric Encryption จาก Route ที่ชี้ผิด Interface |
| 599 | Scenario 8 — Dual-Hub Failover ไม่ทำงานเพราะ NHRP Timer/Backup NHS ไม่ครบ |
| 600 | Lab เต็มรูปแบบ — 3 ปัญหาซ้อนกันใน Dual-Hub DMVPN, Diagnose แบบ Systematic |

---

## Step 591 — DMVPN/VPN Troubleshooting Methodology

### 591.1 ทำไมต้อง Diagnose แบบ "Layered" เสมอ

DMVPN/VPN เป็น**การเรียงชั้น Technology ซ้อนกันหลายชั้น** (Layered Stack) — Underlay IP →
IKE/IPsec → NHRP → Routing Protocol → Application Data — แต่ละชั้น**พึ่งพาชั้นที่อยู่ล่างกว่า
เสมอ**: ถ้า Underlay ไปไม่ถึงกัน IKE จะไม่มีทาง Form; ถ้า IKE ไม่ Form, IPsec SA จะไม่มีวันเกิด;
ถ้า IPsec ไม่มี, mGRE Packet (ที่ต้องถูกเข้ารหัสก่อนส่ง) จะถูก Drop ทำให้ NHRP Registration
ไปไม่ถึง Hub; ถ้า NHRP ไม่ลงทะเบียน Routing Protocol ก็ไม่มีทาง Neighbor เกิดผ่าน Tunnel ได้

**บทเรียนที่สำคัญที่สุดของ Step นี้**: วิศวกรมือใหม่มักเริ่ม Diagnose จาก Symptom ที่เห็นชัดสุด
(เช่น "OSPF Neighbor ไม่ขึ้น") แล้วรีบไปแก้ Routing Config ทันที — แต่ถ้า Root Cause จริงๆอยู่ที่
IKE Phase 1 (ชั้นล่างกว่ามาก) การแก้ Routing Config จะไม่มีผลอะไรเลย และเสียเวลาโดยเปล่าประโยชน์
**หลักการ ENARSI ที่ถูกต้องคือ Diagnose จากล่างขึ้นบน (Bottom-Up) เสมอ**

### 591.2 6 Layer ที่ต้องไล่ตรวจตามลำดับ

```
Layer 6: Data Plane (Traffic จริงไปถึงปลายทางไหม, สมมาตรไหม)
              ▲  ping / traceroute ผ่าน Tunnel, show crypto ipsec sa (encaps=decaps?)
Layer 5: Routing Protocol over Tunnel (Neighbor/Adjacency ขึ้นไหม, Route ถูกเรียนไหม)
              ▲  show ip ospf neighbor, show ip route
Layer 4: NHRP (Spoke Register กับ NHS สำเร็จไหม, Mapping ถูกไหม)
              ▲  show ip nhrp, show dmvpn, show ip nhrp nhs detail
Layer 3: IPsec Phase 2 / Tunnel Protection (ESP SA เกิดไหม, เข้ารหัสจริงไหม)
              ▲  show crypto ipsec sa
Layer 2: IKE Phase 1 / ISAKMP (Management Channel เกิดไหม)
              ▲  show crypto isakmp sa
Layer 1: Underlay IP Reachability (NBMA ถึงกันไหม, Physical up/up ไหม)
              ▲  show ip interface brief, ping <NBMA>, traceroute <NBMA>
   (จุดเริ่มต้นของการ Diagnose ทุกครั้ง — เริ่มที่นี่เสมอ)
```

### 591.3 ตาราง Checklist คำสั่งต่อ Layer

| Layer | คำถามหลัก | คำสั่งตรวจสอบ | อาการเมื่อ Layer นี้พัง |
|---|---|---|---|
| 1. Underlay | Physical Interface up/up ไหม, ping NBMA ถึงกันไหม, มี ACL/Firewall บล็อกอยู่ระหว่างทางไหม | `show ip interface brief`, `ping <NBMA-IP>`, `traceroute <NBMA-IP>` | Tunnel Interface อาจยัง `up/up` ได้ (เพราะ GRE Line-Protocol ไม่ผูกกับ Underlay โดยตรงถ้าไม่มี `keepalive`) แต่ทุก Layer บนนี้จะไม่ทำงานเลย |
| 2. IKE Phase 1 | ISAKMP SA เกิดไหม, State เป็น `QM_IDLE` (สมบูรณ์) หรือค้างที่ State อื่น | `show crypto isakmp sa`, `debug crypto isakmp` | State ค้างที่ `MM_KEY_EXCH`/`MM_SA_SETUP`/ไม่มี SA เลย |
| 3. IPsec Phase 2 | ESP SA เกิดไหม, `pkts encaps`/`pkts decaps` เพิ่มขึ้นไหม | `show crypto ipsec sa`, `debug crypto ipsec` | `pkts encaps #pkts encaps: 0`, ไม่มี Inbound/Outbound SPI |
| 4. NHRP | Spoke ขึ้น `RE` (Registered) กับ NHS ไหม, `show dmvpn` เห็น Peer ไหม | `show ip nhrp`, `show dmvpn`, `show ip nhrp nhs detail` | Entry ว่างเปล่า, ค้างที่ `W` (Waiting) หรือ `E` (Expecting) |
| 5. Routing over Tunnel | Neighbor ขึ้นไหม, Route เรียนมาไหม, Next-Hop ถูกไหม | `show ip ospf neighbor`, `show ip route`, `show ip protocols` | ไม่มี Neighbor, ไม่มี Route แม้ NHRP จะสมบูรณ์แล้ว |
| 6. Data Plane | Ping/Traceroute ผ่านจริงไหม, ทางไป-กลับสมมาตรไหม | `ping`, `traceroute`, เทียบ `show crypto ipsec sa` ทั้งสองฝั่ง | Traffic ทางเดียวผ่าน อีกทางไม่ผ่าน (Asymmetric), Shortcut ไม่เกิด |

> **Golden Rule ของ Part นี้**: ทุก Scenario ด้านล่างจะเดินตาม Checklist นี้เป๊ะๆ — เริ่มจาก
> `show ip interface brief` เสมอ ก่อนไล่ขึ้นไปทีละ Layer จนเจอจุดที่ Symptom เริ่มปรากฏ **นั่นคือ
> Layer ที่มี Root Cause อยู่**

---

## Step 592 — Scenario 1: IKE Phase 1 (ISAKMP) ล้มเหลวจาก Pre-Shared Key Mismatch

### 592.1 อาการที่ Helpdesk รายงาน

Engineer เพิ่ม Spoke สาขาใหม่กลับเข้าวง DMVPN หลังส่ง RMA เปลี่ยน Router ที่ **BRANCH2-RTR** —
พิมพ์ Config กลับจาก Backup File ทั้งหมด แต่ **BRANCH2-RTR ping LAN ของ BRANCH-RTR ไม่ผ่านเลย**
ขณะที่ BRANCH-RTR ↔ WAN-EDGE-1 ยังใช้งานได้ปกติ

### 592.2 Config ที่ Restore มา (มี Bug ซ่อนอยู่)

```
BRANCH2-RTR(config)# crypto isakmp policy 10
BRANCH2-RTR(config-isakmp)# encryption aes 256
BRANCH2-RTR(config-isakmp)# hash sha256
BRANCH2-RTR(config-isakmp)# authentication pre-share
BRANCH2-RTR(config-isakmp)# group 14
BRANCH2-RTR(config-isakmp)# exit
BRANCH2-RTR(config)# crypto isakmp key Cisc0@DMVPN123 address 0.0.0.0 0.0.0.0
```

> สังเกตให้ดี: `Cisc0@DMVPN123` — ตัว `o` ตัวที่สองใน `Cisco` ถูกพิมพ์เป็นเลข `0` โดย OCR ตอน
> Restore จาก Printed Backup (Typo ที่พบบ่อยมากในทางปฏิบัติจริง) ค่าที่ถูกต้องตาม Part 53 คือ
> `Cisco@DMVPN123` — ต่างกันแค่ 1 ตัวอักษรแต่ทำให้ Authentication ล้มเหลวทั้งหมด

### 592.3 Diagnose ตาม Methodology (Step 591) — เริ่มจาก Layer 1

```
BRANCH2-RTR# show ip interface brief | include Tunnel0
Tunnel0                172.16.100.12   YES manual up                    up

BRANCH2-RTR# ping 203.0.113.2 source GigabitEthernet0/0/1
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/11/15 ms
```

**Layer 1 ผ่าน**: Tunnel0 ยัง `up/up` (เพราะไม่มี `keepalive` ผูกกับ IKE) และ NBMA Reachability
ไปยัง WAN-EDGE-1 ปกติ — ปัญหาไม่ได้อยู่ที่ Underlay แน่นอน ไปดู Layer 2 ต่อ

```
WAN-EDGE-1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.2     198.51.100.2    QM_IDLE           1001 ACTIVE
203.0.113.2     198.51.100.6    MM_KEY_EXCH       1002 ACTIVE
```

`198.51.100.2` (BRANCH-RTR) อยู่ที่ `QM_IDLE` = Phase 1 สมบูรณ์แล้ว แต่ `198.51.100.6`
(BRANCH2-RTR) **ค้างอยู่ที่ `MM_KEY_EXCH`** (Main Mode Message ที่ 5/6 — ขั้นตอนที่ต้อง Verify
Hash ที่คำนวณจาก Pre-Shared Key) — นี่คือหลักฐานชัดว่า **Phase 1 ล้มเหลวเฉพาะคู่นี้**

```
WAN-EDGE-1# debug crypto isakmp
*Sep 26 10:14:02.112: ISAKMP (1002): processing HASH payload. message ID = 0
*Sep 26 10:14:02.113: ISAKMP (1002): notify HASH not authenticated
*Sep 26 10:14:02.114: %CRYPTO-4-IKMP_BAD_MESSAGE: IKE message from 198.51.100.6 failed
  its sanity check or is malformed
```

`notify HASH not authenticated` = **ยืนยัน Root Cause ตรงตัว**: WAN-EDGE-1 คำนวณ Hash จาก
Pre-Shared Key ของตัวเอง (`Cisco@DMVPN123`) แล้วเทียบกับ Hash ที่ BRANCH2-RTR ส่งมา (คำนวณจาก
`Cisc0@DMVPN123`) — **ไม่ตรงกัน** จึง Reject ที่ขั้นตอนสุดท้ายของ Main Mode

### 592.4 Root Cause และการแก้ไข

| รายการ | ค่าที่ผิด (BRANCH2-RTR) | ค่าที่ถูกต้อง |
|---|---|---|
| `crypto isakmp key` | `Cisc0@DMVPN123` | `Cisco@DMVPN123` |

```
BRANCH2-RTR(config)# no crypto isakmp key Cisc0@DMVPN123 address 0.0.0.0 0.0.0.0
BRANCH2-RTR(config)# crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
```

### 592.5 Verify

```
WAN-EDGE-1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.2     198.51.100.2    QM_IDLE           1001 ACTIVE
203.0.113.2     198.51.100.6    QM_IDLE           1003 ACTIVE
```

ทั้งสอง Peer อยู่ที่ `QM_IDLE` แล้ว — Phase 1 สมบูรณ์ พร้อมให้ Phase 2 (IPsec) เจรจาต่อ

> **จุดที่ต้องระวังสำหรับข้อสอบ ENARSI**: `MM_KEY_EXCH` ที่ค้างอยู่ไม่ได้แปลว่า PSK ผิดเสมอไป —
> อาจเป็น `crypto isakmp policy` (Encryption/Hash/DH Group) ไม่ Match กันก็ได้เช่นกัน วิธีแยกคือ
> ดู `debug crypto isakmp` ต่อ: ถ้า Log บอก `atts are not acceptable` = Policy Parameter ไม่ตรง
> (ต้องแก้ `encryption`/`hash`/`group`), ถ้าบอก `HASH not authenticated`/`INVALID_KEY_INFORMATION`
> = PSK ไม่ตรง (ต้องแก้ `crypto isakmp key`) — **อย่าเดา ต้องอ่าน debug message ให้ตรงประเด็น**

---

## Step 593 — Scenario 2: IPsec Phase 2 ล้มเหลวจาก Transform-Set Mismatch

### 593.1 อาการ

หลังจากแก้ Scenario 1 เสร็จ (PSK ตรงกันแล้ว) BRANCH2-RTR ยัง **Ping ข้าม Site ไม่ผ่านอยู่ดี**
แต่ตอนนี้ `show crypto isakmp sa` ขึ้น `QM_IDLE` เรียบร้อยแล้ว — แสดงว่าปัญหาขยับขึ้นไปอีก Layer

### 593.2 Config ที่มี Bug (Transform-Set ไม่ตรงกัน)

Engineer คนเดิมที่ Restore Config ผิดใน Scenario 1 ดันคอนฟิก Transform-Set ผิดตามไปด้วย
โดยพิมพ์ Algorithm ตาม Template เก่าที่เคยใช้กับ Site อื่นที่ Bandwidth จำกัดกว่า (ตั้งใจลด
Encryption Overhead) โดยไม่รู้ว่า DMVPN Cloud นี้ต้อง**ตรงกันทุกอุปกรณ์**:

```
BRANCH2-RTR(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 128 esp-sha-hmac
BRANCH2-RTR(cfg-crypto-trans)# mode transport
BRANCH2-RTR(cfg-crypto-trans)# exit
```

ขณะที่ WAN-EDGE-1/WAN-EDGE-2/BRANCH-RTR ทุกตัวยังใช้ค่าเดิมจาก Part 53:

```
WAN-EDGE-1(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
WAN-EDGE-1(config-crypto-trans)# mode transport
```

> **จุดที่ทำให้งงในทางปฏิบัติ**: ชื่อ Transform-Set (`DMVPN-TSET`) **เหมือนกันทุกตัวอักษร** —
> แต่ Algorithm ภายใน (`esp-aes 128 esp-sha-hmac` vs `esp-aes 256 esp-sha256-hmac`) **ต่างกัน**
> — IOS ไม่ Validate ว่าชื่อ Transform-Set ต้อง Map กับ Algorithm เดียวกันข้าม Router เพราะ
> Transform-Set เป็นแค่ Local Object ที่ใช้ชื่ออ้างอิงในเครื่องตัวเอง **ชื่อไม่จำเป็นต้องตรงกัน
> ระหว่าง Router เลยด้วยซ้ำ — Cisco ตรวจแค่ Algorithm ที่เจรจากันจริงตอน Quick Mode/IKEv2
> CREATE_CHILD_SA เท่านั้น**

### 593.3 Diagnose

**Layer 1-2 ผ่านแล้ว** (Scenario 1 ยืนยันแล้ว) ไปตรวจ Layer 3 ต่อ:

```
WAN-EDGE-1# show crypto ipsec sa peer 198.51.100.6

interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 203.0.113.2

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (203.0.113.2/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (198.51.100.6/255.255.255.255/47/0)
   current_peer 198.51.100.6 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0
    #pkts decaps: 0, #pkts decrypt: 0, #pkts verify: 0
    #pkts compressed: 0, #pkts decompressed: 0
    #send errors 12, #recv errors 0

     inbound esp sas:
     outbound esp sas:
```

**หลักฐานชัดเจนที่สุด**: `#pkts encaps: 0` ทุกตัว และ **ไม่มี `inbound esp sas`/`outbound esp
sas` เลยสักบรรทัด** (Section ว่างเปล่า) — แปลว่า **ไม่มี IPsec SA ตั้งขึ้นมาได้จริงสักคู่**
แม้ Phase 1 จะผ่านแล้วก็ตาม — `#send errors 12` ที่เพิ่มขึ้นเรื่อยๆ ยืนยันว่า Router พยายาม
เจรจา Phase 2 ซ้ำแล้วซ้ำเล่าแต่ล้มเหลวทุกครั้ง

```
WAN-EDGE-1# debug crypto ipsec
*Sep 26 10:22:17.881: IPSEC(validate_proposal_request): proposal part #1,
  (key eng. msg.) INBOUND local= 203.0.113.2, remote= 198.51.100.6,
    local_proxy= 203.0.113.2/255.255.255.255/47/0,
    remote_proxy= 198.51.100.6/255.255.255.255/47/0,
    protocol= ESP, transform= NONE  (Transport),
*Sep 26 10:22:17.882: IPSEC(ipsec_process_proposal): proposal part #1,
  transform proposals not supported
```

`transform proposals not supported` = **ยืนยัน Root Cause**: WAN-EDGE-1 เสนอ `esp-aes 256
esp-sha256-hmac` แต่ BRANCH2-RTR เสนอ `esp-aes 128 esp-sha-hmac` กลับมา — ไม่มี Transform ตัว
ไหนที่ทั้งสองฝั่งเห็นตรงกันเลย Phase 2 (Quick Mode) จึงล้มเหลว

### 593.4 Root Cause และการแก้ไข

| อุปกรณ์ | Transform-Set เดิม (ผิด) | Transform-Set ที่ถูกต้อง |
|---|---|---|
| BRANCH2-RTR | `esp-aes 128 esp-sha-hmac` | `esp-aes 256 esp-sha256-hmac` |

```
BRANCH2-RTR(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
BRANCH2-RTR(cfg-crypto-trans)# mode transport
BRANCH2-RTR(cfg-crypto-trans)# exit
```

> การพิมพ์ `crypto ipsec transform-set <ชื่อเดิม>` ซ้ำด้วย Algorithm ใหม่ **จะ Overwrite
> ค่าเดิมทันทีโดยไม่ต้อง `no` ก่อน** (ต่างจาก ACL ที่ต้อง `no` แต่ละ Line) — แต่ถ้ามี Session
> ที่ใช้ Transform-Set นี้ Active อยู่ IOS จะเตือนว่า Session เดิมจะไม่ถูกกระทบจนกว่าจะ Renegotiate

### 593.5 Verify

```
WAN-EDGE-1# show crypto ipsec sa peer 198.51.100.6 | include pkts encaps|inbound|outbound
    #pkts encaps: 148, #pkts encrypt: 148, #pkts digest: 148
     inbound esp sas:
      spi: 0xC3A21F08(3282817800)
     outbound esp sas:
      spi: 0x9F41A2E0(2673868512)
```

มี `inbound esp sas`/`outbound esp sas` พร้อม SPI จริง และ `#pkts encaps` เพิ่มขึ้นเรื่อยๆ —
Phase 2 สมบูรณ์แล้ว

> **หมายเหตุสำหรับข้อสอบ**: DMVPN ใช้ `tunnel protection ipsec profile` ซึ่งไม่มี Crypto ACL
> (`match address`) แบบ Crypto Map ดั้งเดิมใน Part 51 — Traffic Selector ของ DMVPN เป็นแบบ
> Implicit (`protocol 47` ระหว่าง NBMA IP คู่นั้นเท่านั้น) ดังนั้น **สาเหตุ ACL/Traffic-Selector
> Mismatch แทบไม่เกิดกับ DMVPN โดยตรง** แต่ยังเป็นสาเหตุคลาสสิกกับ IPsec แบบ Crypto Map ตัวต่อตัว
> (Point-to-Point GRE-over-IPsec ของ Part 51) — ถ้าเจอ `#pkts encaps: 0` แบบเดียวกันบน Crypto
> Map แบบเก่า ให้ตรวจ `show crypto map` เทียบ ACL (`match address`) ทั้งสองฝั่งว่า Mirror กัน
> พอดี (Source/Destination สลับข้างกัน) ด้วย เพราะเป็นสาเหตุที่พบบ่อยพอกับ Transform-Set Mismatch

---

## Step 594 — Scenario 3: NHRP Registration ล้มเหลวเพราะ NHS Address ผิด

### 594.1 อาการ

Phase 1/2 (IKE/IPsec) สมบูรณ์แล้วตาม Scenario 1-2 แต่ **BRANCH2-RTR ยัง Ping ข้าม Site ไม่ผ่าน**
— WAN-EDGE-1 มองไม่เห็น BRANCH2-RTR ใน DMVPN Cloud เลย

### 594.2 Config ที่มี Bug

ตรวจ Config ของ Tunnel0 ที่ BRANCH2-RTR พบว่า `ip nhrp nhs` ชี้ไปที่ **Tunnel IP ผิด**
(พิมพ์เลขตัวสุดท้ายผิดจาก `.1` เป็น `.10` — ไม่มี Device ไหนถือ IP `172.16.100.10` อยู่จริงเลย)

```
BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# ip nhrp nhs 172.16.100.10 nbma 203.0.113.2 multicast
```

ค่าที่ถูกต้องตาม Part 53 คือ `ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast`
(Tunnel IP ของ WAN-EDGE-1 คือ `172.16.100.1` ไม่ใช่ `.10`)

### 594.3 Diagnose

**Layer 1-3 ผ่านแล้ว** ไปตรวจ Layer 4 (NHRP):

```
WAN-EDGE-1# show ip nhrp
172.16.100.11/32 via 172.16.100.11
   Tunnel0 created 00:45:12, expire 01:59:03
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.2
```

**WAN-EDGE-1 เห็นแค่ BRANCH-RTR (`172.16.100.11`) เท่านั้น — ไม่มี Entry ของ BRANCH2-RTR
(`172.16.100.12`) เลยแม้แต่บรรทัดเดียว** ทั้งที่ Phase 1/2 ที่ Scenario 1-2 แก้ไปแล้วสมบูรณ์แล้ว
— นี่คือหลักฐานว่า NHRP Registration Request จาก BRANCH2-RTR **ไม่เคยไปถึง Hub เลย**

```
BRANCH2-RTR# show ip nhrp nhs detail

Legend: E=Expecting Replies, R=Responding, W=Waiting

Tunnel0:
  172.16.100.10  W  priority = 0 cluster = 0
      req-sent 24 req-failed 24 repl-recv 0
```

`W` (Waiting) แปลว่า BRANCH2-RTR **ส่ง Registration Request ไปเรื่อยๆ แต่ไม่มีการตอบกลับ**
(`req-sent 24 req-failed 24 repl-recv 0` = ส่งไป 24 ครั้ง ล้มเหลวครบ 24 ครั้ง ไม่มี Reply เลย)
และ **NHS ที่ระบุคือ `172.16.100.10` ซึ่งไม่ใช่ IP ของอุปกรณ์ไหนในวงจริง** — Request จึงถูกส่ง
ไปยัง Tunnel IP ที่ไม่มีใครรับ (ทาง mGRE จะพยายาม Resolve ผ่าน NHS ตัวเดิมที่ระบุไว้ใน `nbma`
คือ `203.0.113.2` แต่ Payload ภายใน NHRP Packet เองยังอ้าง Target Protocol Address เป็น
`172.16.100.10` ทำให้ WAN-EDGE-1 ที่รับ Packet เข้ามาไม่ยอมรับว่าเป็น Registration ที่ตรงกับ
ตัวเอง — Behavior จริงขึ้นกับ IOS Version แต่ผลลัพธ์ที่สังเกตได้คือ Registration ไม่สำเร็จเสมอ)

### 594.4 Root Cause และการแก้ไข

| รายการ | ค่าที่ผิด | ค่าที่ถูกต้อง |
|---|---|---|
| `ip nhrp nhs` (Tunnel IP ของ Hub) | `172.16.100.10` | `172.16.100.1` |

```
BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# no ip nhrp nhs 172.16.100.10 nbma 203.0.113.2 multicast
BRANCH2-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
BRANCH2-RTR(config-if)# exit
```

### 594.5 Verify

```
BRANCH2-RTR# show ip nhrp nhs detail
Tunnel0:
  172.16.100.1  RE priority = 0 cluster = 0
      req-sent 3 req-failed 0 repl-recv 3

WAN-EDGE-1# show ip nhrp
172.16.100.11/32 via 172.16.100.11
   Tunnel0 created 00:52:40, expire 01:59:18
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.2
172.16.100.12/32 via 172.16.100.12
   Tunnel0 created 00:00:18, expire 01:59:42
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.6
```

`RE` (Registered) และ Entry ของ BRANCH2-RTR (`172.16.100.12 ↔ 198.51.100.6`) ปรากฏที่ Hub
แล้ว — NHRP สมบูรณ์

> **หมายเหตุ**: อีกสาเหตุที่พบบ่อยพอกันของ NHRP Registration ล้มเหลวคือ **Firewall/ACL
> ระหว่างทางบล็อก GRE (Protocol 47) หรือ UDP 500/4500** อย่างที่ Part 52 Step 519 กล่าวถึง —
> วิธีแยกจากสาเหตุ NHS ผิดคือดูที่ `show crypto isakmp sa`: ถ้า **Phase 1/2 ผ่านแล้ว** (แบบ
> Scenario นี้) แสดงว่า GRE/UDP 500 ไหลผ่านได้อยู่แล้ว ปัญหาต้องอยู่ที่ **NHRP Config เอง**
> ไม่ใช่ Firewall แต่ถ้า Phase 1 ยังไม่ผ่านด้วย ต้องสงสัย Firewall/ACL เป็นอันดับแรก

---

## Step 595 — Scenario 4: mGRE/Point-to-Point Tunnel Mode Mismatch

### 595.1 อาการ

Site สาขาที่ 3 (`BRANCH3-RTR`, สาขาใหม่ที่กำลังเพิ่มเข้าวง DMVPN เป็นครั้งแรก) คอนฟิกตาม
Template ที่ Copy มาจาก Part 52 (ยุค Phase 1/2 เดิม) โดยไม่ได้ปรับให้ตรงกับ Phase 3 ปัจจุบัน —
Tunnel0 ของ BRANCH3-RTR ขึ้น **`down/down`** ทันทีหลัง `no shutdown` และไม่มีทางเชื่อมกับ Hub
ได้เลยแม้แต่ Ping ไปยัง Tunnel IP ของ Hub เอง

### 595.2 Config ที่มี Bug (Legacy P2P GRE ค้างอยู่)

```
BRANCH3-RTR(config)# interface Tunnel0
BRANCH3-RTR(config-if)# ip address 172.16.100.13 255.255.255.0
BRANCH3-RTR(config-if)# ip nhrp authentication DMVPN-KEY
BRANCH3-RTR(config-if)# ip nhrp network-id 100
BRANCH3-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
BRANCH3-RTR(config-if)# ip nhrp shortcut
BRANCH3-RTR(config-if)# tunnel source GigabitEthernet0/1
BRANCH3-RTR(config-if)# tunnel destination 203.0.113.2
BRANCH3-RTR(config-if)# tunnel key 100
BRANCH3-RTR(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
BRANCH3-RTR(config-if)# exit
```

> สังเกต: มี `tunnel destination 203.0.113.2` ค้างอยู่ — Copy มาจาก Template Phase 1 (P2P GRE)
> โดยไม่ได้ลบออก ขณะที่ Hub (WAN-EDGE-1) เป็น `tunnel mode gre multipoint` แบบ Phase 3 อยู่แล้ว

### 595.3 Diagnose

**Layer 1**: ตรวจ Tunnel0 ที่ BRANCH3-RTR ก่อนเลย เพราะ Symptom ("down/down") ปรากฏตั้งแต่
Interface State — เป็นข้อยกเว้นของ Methodology ที่ให้เริ่มดู Interface ก่อนไล่ Layer อื่น
เมื่อ Symptom อยู่ที่ Interface State โดยตรง:

```
BRANCH3-RTR# show interface Tunnel0
Tunnel0 is up, line protocol is down
  Internet address is 172.16.100.13/24
  ...
  Tunnel source 198.51.100.10 (GigabitEthernet0/1), destination 203.0.113.2
  Tunnel protocol/transport GRE/IP
    Key 0x64, sequencing disabled
  Tunnel TTL 255
```

`Tunnel source ... destination 203.0.113.2` = **ยืนยันว่า BRANCH3-RTR ยังเป็น Point-to-Point
GRE** (มี `tunnel destination` ตายตัว) — คำสั่ง `tunnel mode gre multipoint` ที่ควรมี **หายไป
จาก Config เลย** (ไม่ปรากฏใน Output นี้ ต่างจาก Phase 3 Spoke ตัวอื่นที่ต้องขึ้น
`Tunnel protocol/transport multi-GRE/IP`)

เทียบกับ BRANCH-RTR (Spoke ที่ทำงานถูกต้อง) เพื่อยืนยัน Baseline:

```
BRANCH-RTR# show interface Tunnel0 | include Tunnel protocol
  Tunnel protocol/transport multi-GRE/IP
```

`multi-GRE/IP` (BRANCH-RTR, ถูกต้อง) เทียบกับ `GRE/IP` ธรรมดา (BRANCH3-RTR, ผิด) — นี่คือความ
ต่างที่ชัดเจนที่สุดระหว่าง mGRE กับ P2P GRE ใน Output ของ `show interface`

`line protocol is down` เกิดขึ้นเพราะ IOS ตรวจพบว่า **`tunnel destination` ที่ระบุ
(`203.0.113.2`) แต่ Peer (WAN-EDGE-1) ตอบกลับด้วย Encapsulation แบบ mGRE ที่ไม่ตรงกับที่ P2P
GRE คาดหวัง** ทำให้ Line Protocol ไม่ยอมขึ้น — นี่คือ Classic Symptom ของ mGRE/P2P Mismatch

### 595.4 Root Cause และการแก้ไข

| รายการ | ค่าที่ผิด (BRANCH3-RTR) | ค่าที่ถูกต้อง (Phase 3) |
|---|---|---|
| Tunnel Mode | P2P GRE (`tunnel destination` ระบุตายตัว, ไม่มี `tunnel mode gre multipoint`) | mGRE (`no tunnel destination` + `tunnel mode gre multipoint`) |

```
BRANCH3-RTR(config)# interface Tunnel0
BRANCH3-RTR(config-if)# no tunnel destination
BRANCH3-RTR(config-if)# tunnel mode gre multipoint
BRANCH3-RTR(config-if)# exit
```

> **ลำดับคำสั่งสำคัญ**: ต้อง `no tunnel destination` **ก่อน** ใส่ `tunnel mode gre multipoint`
> เสมอ — ถ้าใส่ `tunnel mode gre multipoint` ทั้งที่ยังมี `tunnel destination` ค้างอยู่ IOS
> รุ่นใหม่หลายรุ่นจะ Reject คำสั่งด้วย Error `% Tunnel destination address required` (สำหรับ
> P2P) กลับด้าน หรือทำงานไม่ถูกต้องแบบเงียบๆ (Silent Failure) ขึ้นกับ Platform/Version — ปลอดภัย
> ที่สุดคือ `no tunnel destination` ก่อนเสมอ

### 595.5 Verify

```
BRANCH3-RTR# show interface Tunnel0
Tunnel0 is up, line protocol is up
  Internet address is 172.16.100.13/24
  Tunnel source 198.51.100.10 (GigabitEthernet0/1)
  Tunnel protocol/transport multi-GRE/IP
    Key 0x64, sequencing disabled

WAN-EDGE-1# show dmvpn brief
Interface       Peer NBMA Address  Peer Tunnel Add  State  Attrb
--------------- ------------------ ---------------- ------ -----
Tu0             198.51.100.2       172.16.100.11     UP     D
Tu0             198.51.100.6       172.16.100.12     UP     D
Tu0             198.51.100.10      172.16.100.13     UP     D
```

`line protocol is up` และ `multi-GRE/IP` ยืนยันว่า Tunnel0 เป็น mGRE สมบูรณ์แล้ว, `show dmvpn
brief` ที่ Hub เห็น BRANCH3-RTR ครบ (`172.16.100.13`, `Attrb = D`)

---

## Step 596 — Scenario 5: Phase 3 Spoke-to-Spoke ไม่ Shortcut (Redirect/Shortcut ขาดหาย)

### 596.1 อาการ

ทุก Site ขึ้นปกติหมดแล้ว (Phase 1/2/NHRP/mGRE ผ่านทุก Layer) — Ping จาก LAN ของ BRANCH-RTR
(`10.20.1.x`) ไปยัง LAN ของ BRANCH2-RTR (`10.20.2.x`) **ผ่านได้ตลอด** แต่ทีม Network
Operations สังเกตว่า **Latency สูงคงที่ตลอดเวลา (ไม่ลดลงเลยแม้ Ping ต่อเนื่องนานหลายนาที)**
ต่างจาก Behavior ปกติของ Phase 3 ที่ Latency ควรลดลงหลัง Shortcut เกิดขึ้น (เพราะ Traffic
ไม่ต้องผ่าน Hub 2 รอบอีกต่อไป) — นี่คือสัญญาณว่า **Spoke-to-Spoke ไม่เกิด Direct Tunnel เลย
ทั้งที่ Design ตั้งใจให้เป็น Phase 3**

### 596.2 Config ที่มี Bug (คำสั่งขาดหายหลัง Maintenance Window)

ตรวจสอบพบว่าช่วง Maintenance Window ล่าสุด Engineer อีกคน `no ip nhrp redirect` ออกจาก
WAN-EDGE-1 โดยเข้าใจผิดว่าเป็นคำสั่งที่ไม่จำเป็น (สับสนกับ `no ip redirects` ที่เป็นคำสั่ง
ICMP Redirect ทั่วไปที่มักปิดใน DMVPN Design เดิม)

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# no ip nhrp redirect
```

Config ที่เหลือของ Tunnel0 ยังถูกต้องครบทุกอย่าง (`ip nhrp network-id 100`, `tunnel protection
ipsec profile DMVPN-PROFILE`, `ip ospf network point-to-multipoint` ฯลฯ) — ขาดไปแค่บรรทัดนี้
บรรทัดเดียว

### 596.3 Diagnose

**Layer 1-5 ผ่านหมดแล้ว** (Ping ผ่าน, OSPF Neighbor ขึ้นปกติ) — ต้องไปตรวจ Layer 6 (Data Plane)
โดยเฉพาะจุดที่ Part 53 Step 529 สอนไว้คือ `show ip nhrp shortcut`:

```
BRANCH-RTR# show ip nhrp shortcut

BRANCH-RTR#
```

**คำสั่งไม่แสดงผลอะไรเลย (ว่างเปล่าสมบูรณ์)** แม้จะ Ping ข้าม Site ไปแล้วหลายสิบ Packet — เทียบ
กับ Behavior ที่ถูกต้องตาม Part 53 ที่ควรเห็น Entry `172.16.100.12/32 via 172.16.100.12 ... Type:
dynamic, Flags: router rib` ปรากฏขึ้นหลัง Traffic กระตุ้น

ตรวจต่อว่า Hub ส่ง Redirect ออกไปหรือไม่ด้วย `debug nhrp` ที่ WAN-EDGE-1 ระหว่างสั่ง Ping จาก
BRANCH-RTR:

```
WAN-EDGE-1# debug nhrp
BRANCH-RTR# ping 10.20.2.1 source 10.20.1.1 repeat 20
!!!!!!!!!!!!!!!!!!!!

WAN-EDGE-1#
(ไม่มี Log NHRP Traffic Indication/Redirect ปรากฏขึ้นเลยระหว่าง Ping)
```

**ไม่มี Log การส่ง NHRP Redirect เกิดขึ้นเลย** ทั้งที่ CEF บน WAN-EDGE-1 ตรวจพบ Same-Interface
Forwarding (Ingress = Egress = Tunnel0) อย่างแน่นอน (เพราะ Traffic ไป-กลับข้าม Site ทั้งคู่ผ่าน
Tunnel0 เดียวกัน) — สิ่งเดียวที่อธิบายได้คือ **Hub ไม่ได้เปิด Redirect Feature อยู่**

```
WAN-EDGE-1# show run interface Tunnel0 | include redirect
```

คำสั่งนี้**ไม่คืนผลลัพธ์อะไรเลย** — ยืนยัน Root Cause ชัดเจนที่สุด: **`ip nhrp redirect` ไม่มี
อยู่ใน Config ของ Tunnel0 บน Hub**

### 596.4 Root Cause และการแก้ไข

| รายการ | สถานะที่ผิด | สถานะที่ถูกต้อง |
|---|---|---|
| `ip nhrp redirect` บน WAN-EDGE-1 Tunnel0 | ไม่มี (ถูกลบออกโดยไม่ตั้งใจ) | ต้องมี |

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip nhrp redirect
WAN-EDGE-1(config-if)# exit
```

> **ทำไมยังไม่มี Symptom ที่ Ping ล้มเหลว**: เพราะ Routing Protocol (OSPF Phase 3) ยังคง
> Advertise Route ที่ Next-Hop เป็น Hub ได้ปกติ (Next-Hop-Self ทำงานได้เสมอใน Phase 3 ไม่ว่า
> Redirect จะเปิดหรือไม่) — Traffic จึง**ยัง Hairpin ผ่าน Hub สำเร็จอยู่ดี** เพียงแต่**ไม่มีทาง
> Shortcut ให้ตรงขึ้นได้** ซึ่งเป็นปัญหาด้าน Performance/Scale ไม่ใช่ Reachability — จุดนี้คือ
> เหตุผลที่ข้อสอบ ENARSI มักตั้งโจทย์ "Ping ผ่านปกติ แต่..." เพื่อทดสอบว่าเข้าใจ Phase 3 ลึกพอ
> ที่จะรู้ว่าต้องดู `show ip nhrp shortcut` ไม่ใช่แค่ Ping ผ่านแล้วสรุปว่าไม่มีปัญหา

### 596.5 Verify

```
BRANCH-RTR# ping 10.20.2.1 source 10.20.1.1 repeat 20
!!!!!!!!!!!!!!!!!!!!

BRANCH-RTR# show ip nhrp shortcut
172.16.100.12/32 via 172.16.100.12
   Tunnel0 created 00:00:04, expire 01:59:55
   Type: dynamic, Flags: router rib
   NBMA address: 198.51.100.6

WAN-EDGE-1# show dmvpn detail | include Shortcut
```

Entry ของ Shortcut Route (`172.16.100.12/32`, `Flags: router rib`) ปรากฏขึ้นทันทีหลัง Ping
รอบใหม่ — Phase 3 Redirect/Shortcut ทำงานสมบูรณ์แล้ว

---

## Step 597 — Scenario 6: Routing Protocol ไม่ Form Neighbor เพราะ Network-Type Mismatch

### 597.1 อาการ

BRANCH2-RTR ผ่าน IKE/IPsec/NHRP ครบทุก Layer (`show dmvpn brief` เห็น `UP`, `Attrb = D`
ตามปกติ) **แต่ OSPF ไม่มี Neighbor กับ Hub เลย** — Route ของ BRANCH2-RTR ไม่ปรากฏที่ไหนในวง
DMVPN ทั้งที่ NHRP บอกว่า Tunnel เชื่อมกันสมบูรณ์แล้ว

### 597.2 Config ที่มี Bug

Engineer แก้ Config OSPF บน Tunnel0 ของ BRANCH2-RTR หลัง Copy Template มาจาก Lab อื่นที่ใช้
Network Type แบบ Broadcast (คุ้นเคยกับ Ethernet ปกติมากกว่า) โดยไม่รู้ว่า DMVPN Phase 3 ต้องใช้
Point-to-Multipoint เท่านั้นตามที่ Part 53 Step 522 กำหนดไว้:

```
BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# ip ospf network broadcast
BRANCH2-RTR(config-if)# ip ospf 100 area 51
BRANCH2-RTR(config-if)# exit
```

ขณะที่ WAN-EDGE-1 (Hub) ยังเป็น `ip ospf network point-to-multipoint` ตามค่าเดิมจาก Part 53

### 597.3 Diagnose

**Layer 1-4 ผ่านหมด** (`show dmvpn brief` ยืนยัน UP/D) — ไปตรวจ Layer 5:

```
WAN-EDGE-1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.20.0.1        0   FULL/  -        00:01:44    172.16.100.11   Tunnel0
```

**เห็น Neighbor ของ BRANCH-RTR (`172.20.0.1`) แต่ไม่มี Neighbor ของ BRANCH2-RTR
(`172.20.1.1`) เลย** ทั้งที่ NHRP บอกว่า Tunnel ขึ้นแล้ว — ยืนยันว่าปัญหาอยู่ที่ OSPF โดยเฉพาะ

```
WAN-EDGE-1# debug ip ospf adj
*Sep 26 11:03:55.201: OSPF-100 ADJ  Tu0: 2-Way Communication to 172.16.100.12
  on Tunnel0, state 2WAY
*Sep 26 11:03:55.203: OSPF-100 ADJ  Tu0: Nbr 172.16.100.12 has larger interface
  MTU/Network-Type mismatch: my type POINT_TO_MULTIPOINT, hello Network Type BROADCAST
```

Log บอกตรงตัวเลย: **`Network-Type mismatch: my type POINT_TO_MULTIPOINT, hello Network Type
BROADCAST`** — WAN-EDGE-1 ส่ง Hello แบบ Point-to-Multipoint แต่ได้รับ Hello จาก BRANCH2-RTR
ที่ประกาศตัวเป็น Broadcast Network Type ใน Packet — OSPF Reject การ Form Adjacency ทันทีเมื่อ
Network Type ไม่ตรงกัน (เป็นกฎที่ OSPF บังคับเสมอ ไม่ว่า Network Type ไหนก็ตาม)

ตรวจยืนยันที่ BRANCH2-RTR:

```
BRANCH2-RTR# show ip ospf interface Tunnel0 | include Network Type
  Process ID 100, Router ID 172.20.1.1, Network Type BROADCAST, Cost: 1
```

`Network Type BROADCAST` ยืนยัน Root Cause ชัดเจน — ต่างจาก WAN-EDGE-1 ที่ควรเป็น
`POINT_TO_MULTIPOINT`

### 597.4 Root Cause และการแก้ไข

| อุปกรณ์ | Network Type ที่ผิด | Network Type ที่ถูกต้อง |
|---|---|---|
| BRANCH2-RTR (Tunnel0) | `broadcast` | `point-to-multipoint` |

```
BRANCH2-RTR(config)# interface Tunnel0
BRANCH2-RTR(config-if)# ip ospf network point-to-multipoint
BRANCH2-RTR(config-if)# exit
```

> **ทำไม Broadcast ใช้ไม่ได้กับ DMVPN**: Network Type Broadcast ต้องมี **DR/BDR Election**
> (เหมือน Ethernet Segment ปกติ) ซึ่งต้องพึ่งพา Link-Layer Multicast/Broadcast ที่ NBMA Cloud
> ไม่รองรับแบบ Native (ทุก Spoke ไม่เห็นกันเองโดยตรงในทาง Physical แม้จะอยู่ใน mGRE Subnet
> เดียวกันก็ตาม) และยิ่งขัดกับหลักการ Phase 3 ที่ต้องการ Adjacency แบบ Point-to-Point ย่อยต่อ
> Neighbor เพื่อให้ Hub ทำ Next-Hop-Self ได้อย่างถูกต้องตามที่ Part 53 Step 522 อธิบายไว้ —
> **Point-to-Multipoint คือ Network Type เดียวที่ถูกต้องสำหรับ DMVPN mGRE Tunnel เสมอ**
> (ไม่นับ Non-Broadcast แบบ Manual Neighbor ที่ซับซ้อนกว่าและไม่ใช้ใน Lab นี้)

### 597.5 Verify

```
WAN-EDGE-1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.20.0.1        0   FULL/  -        00:01:38    172.16.100.11   Tunnel0
172.20.1.1        0   FULL/  -        00:01:49    172.16.100.12   Tunnel0
```

Neighbor ของ BRANCH2-RTR (`172.20.1.1`) ขึ้น `FULL/ -` แล้ว (ไม่มี `/DR`/`/BDR` ยืนยัน
Point-to-Multipoint ทำงานถูกต้องเหมือน Part 53)

---

## Step 598 — Scenario 7: Asymmetric Encryption จาก Route ที่ชี้ผิด Interface

### 598.1 อาการ

BRANCH-RTR รายงานว่า **Ping จาก LAN ของตัวเองไปยัง LAN ของ BRANCH2-RTR ส่งออกได้ (Echo Request
ออกไปแน่นอน) แต่ไม่ได้รับ Echo Reply กลับมาเลย** — ทุก Layer ของ DMVPN (IKE/IPsec/NHRP/OSPF)
ที่ BRANCH-RTR ตรวจสอบแล้วดูปกติทุกอย่าง ปัญหาต้องอยู่ที่อีกฝั่ง

### 598.2 Config ที่มี Bug (Static Route แซง Route ของ Tunnel)

ตรวจสอบที่ BRANCH2-RTR พบว่ามี Static Route เก่าที่ค้างมาจากการทดสอบ Backup Link แบบ Manual
เมื่อหลายเดือนก่อน **ไม่ได้ถูกลบออกหลัง Test เสร็จ**:

```
BRANCH2-RTR(config)# ip route 10.20.1.0 255.255.255.0 GigabitEthernet0/1 198.51.100.5
```

Route นี้ชี้ตรงไปยัง Subnet ของ BRANCH-RTR LAN (`10.20.1.0/24`) โดยออกทาง **Physical Interface
(`GigabitEthernet0/1`) ตรงๆ** (Underlay/NBMA Path) แทนที่จะผ่าน `Tunnel0` — Static Route มี
Administrative Distance เริ่มต้น (`1`) **ดีกว่า OSPF (`110`) เสมอ** จึงถูกเลือกใน RIB ก่อน
Route ที่เรียนจาก OSPF ผ่าน Tunnel0 โดยอัตโนมัติ แม้ Next-Hop `198.51.100.5` จะไม่มี Router
ตัวไหนถืออยู่จริงก็ตาม (Test Address ที่ไม่มีอยู่แล้วในปัจจุบัน)

### 598.3 Diagnose

**Layer 1-6 ที่ BRANCH-RTR ปกติทุกอย่าง** (ตรวจแล้วตาม Methodology ไม่พบปัญหา) — จุดสำคัญของ
Scenario นี้คือ**ต้องข้ามไปตรวจฝั่งตรงข้าม (BRANCH2-RTR) เพราะ Asymmetric แปลว่าปัญหาอยู่ที่
"ทางกลับ" ซึ่งควบคุมโดยอีกฝั่งเสมอ**:

```
BRANCH2-RTR# show ip route 10.20.1.0
Routing entry for 10.20.1.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 198.51.100.5
      Route metric is 0, traffic share count is 1
```

**พบ Root Cause ทันที**: `Known via "static"` ที่ Next-Hop `198.51.100.5` — **ไม่ใช่ Tunnel0
เลย** ทั้งที่ OSPF ควรจะเรียน Route นี้ผ่าน `Tunnel0` ตามปกติ (เทียบกับ `show ip route ospf`
ที่จะเห็น Route เดียวกันแต่ผ่าน Tunnel0 หากไม่มี Static Route มาบัง):

```
BRANCH2-RTR# show ip route ospf | include 10.20.1.0
O IA    10.20.1.0/24 [110/1001] via 172.16.100.11, 00:45:02, Tunnel0
```

OSPF **มี** Route นี้อยู่จริงในฐาน Topology Table (มองเห็นผ่าน `show ip route ospf`) — แต่ Route
ที่ถูกติดตั้งจริงใน RIB ที่ใช้ Forward Traffic กลับกลายเป็น Static Route ที่ AD ดีกว่า (`1` < `110`)

### 598.4 อธิบายว่าทำไมเป็น Asymmetric Encryption

```
ทางไป (BRANCH-RTR → BRANCH2-RTR):
  BRANCH-RTR ── OSPF Route ผ่าน Tunnel0 (ถูกต้อง, มี IPsec ป้องกัน) ──► ถึง BRANCH2-RTR ปกติ

ทางกลับ (BRANCH2-RTR → BRANCH-RTR):
  BRANCH2-RTR ── Static Route ผ่าน Gi0/1 ตรง (Bypass Tunnel0 = Bypass IPsec) ──► 198.51.100.5
                          (Next-Hop นี้ไม่มี Router ถืออยู่จริง ⇒ Packet ถูก Drop กลางทาง)
```

Echo Request จาก BRANCH-RTR ไปถึง BRANCH2-RTR ได้ปกติ (เพราะ BRANCH-RTR ไม่มี Static Route
แซง Route ของ Tunnel เลย) แต่ Echo Reply ที่ BRANCH2-RTR ต้องส่งกลับ **ถูก Route ผิดทาง**
ออกไปยัง Physical Interface ตรงๆ ตาม Static Route ที่ค้างอยู่ — Traffic ที่ออกทาง Physical
Interface ตรงนี้**ไม่ผ่าน Tunnel Protection เลย** (ไม่ถูก GRE Encapsulate, ไม่ถูก IPsec
เข้ารหัส) และยิ่งไปกว่านั้น Next-Hop `198.51.100.5` ไม่มีตัวตนจริง ทำให้ Packet ถูก Drop ก่อน
จะไปถึงปลายทางเลยด้วยซ้ำ — **นี่คือทั้ง Asymmetric Routing และ Asymmetric Encryption ในปัญหา
เดียวกัน**

> **บทเรียนสำคัญของ Scenario นี้**: อาการ "Encrypt ทางเดียว, อีกทางเป็น Cleartext/Drop" **ไม่ใช่
> ปัญหาของ IKE/IPsec Config เลย** (ทั้งสองฝั่งอาจ Config IPsec ถูกทุกอย่าง) — Root Cause ที่แท้
> จริงอยู่ที่ **Routing Table** เท่านั้น: ถ้ามี Route ที่ Administrative Distance ดีกว่าชี้ออก
> Interface อื่นที่ไม่ใช่ Tunnel Interface สำหรับ Subnet ปลายทางเดียวกัน Traffic นั้นจะ **Bypass
> IPsec โดยสมบูรณ์** โดยไม่มี Error หรือ Warning อะไรเตือนเลย ต้องอ่าน `show ip route <subnet>`
> ที่ Field `Known via` เสมอเพื่อยืนยันว่า Route ที่ถูกใช้จริงมาจาก Protocol/Source ไหน

### 598.5 การแก้ไข

```
BRANCH2-RTR(config)# no ip route 10.20.1.0 255.255.255.0 GigabitEthernet0/1 198.51.100.5
```

### 598.6 Verify

```
BRANCH2-RTR# show ip route 10.20.1.0
Routing entry for 10.20.1.0/24
  Known via "ospf 100", distance 110, metric 1001, type inter area
  Routing Descriptor Blocks:
  * 172.16.100.11, from 1.1.1.21, 00:00:03 ago, via Tunnel0
      Route metric is 1001, traffic share count is 1

BRANCH-RTR# ping 10.20.2.1 source 10.20.1.1 repeat 10
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

Route กลับมาเป็น `Known via "ospf 100"` ผ่าน `Tunnel0` แล้ว — Ping ผ่านทั้งสองทางสมบูรณ์

---

## Step 599 — Scenario 8: Dual-Hub Failover ไม่ทำงาน

### 599.1 อาการ

ทีม Network Operations จำลอง WAN-EDGE-1 (Hub หลัก) ล่มโดยตั้งใจ (`shutdown` Physical Interface
เพื่อทดสอบ DR) ตามแผน — คาดหวังว่า Spoke ทุกตัวจะ Failover ไปใช้ WAN-EDGE-2 (Hub สำรอง) แทนโดย
อัตโนมัติตาม Design ของ Part 53 Step 524/526 แต่ **BRANCH-RTR ใช้เวลานานเกินคาด (มากกว่า 5
นาที) กว่า Traffic จะไปทาง WAN-EDGE-2 ได้** ขณะที่ BRANCH2-RTR Failover ได้เร็วตามที่ออกแบบไว้
(ไม่กี่สิบวินาที)

### 599.2 Config ที่มี Bug — 2 จุดซ้อนกัน

**จุดที่ 1 — NHRP Holdtime ยาวเกินไปที่ BRANCH-RTR** (ค้างมาจากการปรับแต่งเพื่อลด NHRP Traffic
โดยไม่ได้พิจารณาผลกระทบต่อ Failover Time):

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp holdtime 7200
```

**จุดที่ 2 — Backup NHS ไม่ครบที่ BRANCH-RTR** — ตรวจ Config เพิ่มพบว่า Engineer ที่เพิ่ม
WAN-EDGE-2 เข้ามาทำ Config ไม่ครบทุก Spoke (Human Error ตอน Deploy):

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
! ขาด NHS ตัวที่สอง (WAN-EDGE-2) ไปเลย — มีแค่บรรทัดเดียว
```

### 599.3 Diagnose

```
BRANCH-RTR# show ip nhrp nhs detail

Legend: E=Expecting Replies, R=Responding, W=Waiting

Tunnel0:
  172.16.100.1  E priority = 0 cluster = 0
      req-sent 48 req-failed 40 repl-recv 8
```

**เห็น NHS แค่ตัวเดียว** (`172.16.100.1` = WAN-EDGE-1 ที่ตอนนี้ล่มไปแล้ว, State = `E`
Expecting) — **ไม่มี Entry ของ WAN-EDGE-2 (`172.16.100.2`) เลยแม้แต่บรรทัดเดียว** — นี่คือ
Root Cause จุดที่ 1: **BRANCH-RTR ไม่รู้จัก WAN-EDGE-2 เป็น Backup NHS ตั้งแต่แรก** เมื่อ
WAN-EDGE-1 ล่ม จึงไม่มีทาง Register กับ Hub ตัวอื่นได้เลย ไม่ว่าจะรอนานแค่ไหนก็ตาม

เทียบกับ BRANCH2-RTR ที่ Config ถูกต้อง (Failover ได้เร็วตามคาด):

```
BRANCH2-RTR# show ip nhrp nhs detail
Tunnel0:
  172.16.100.1  E priority = 0 cluster = 0
      req-sent 45 req-failed 40 repl-recv 5
  172.16.100.2  RE priority = 0 cluster = 0
      req-sent 45 req-failed 0 repl-recv 45
```

BRANCH2-RTR มี **NHS 2 ตัวครบ** — `172.16.100.1` (WAN-EDGE-1, ตอนนี้ `E` เพราะล่มไปแล้ว) และ
`172.16.100.2` (WAN-EDGE-2, ยัง `RE` ปกติ) จึง Failover ได้ทันทีเพราะมี Path สำรองที่ Register
ไว้อยู่แล้วล่วงหน้า (ไม่ต้องรอ Registration ใหม่เลย)

**ต่อกรณีสมมติที่ BRANCH-RTR จะมี NHS ครบ 2 ตัวแต่ Holdtime ยาวเกินไปด้วย** ให้ตรวจ Layer 4
เพิ่มด้วย `ip nhrp holdtime` เพื่อดูว่าการ Detect Failure ของ NHS เดิมช้าไปด้วยหรือไม่:

```
BRANCH-RTR# show run interface Tunnel0 | include holdtime
 ip nhrp holdtime 7200
```

`ip nhrp holdtime 7200` (ค่าเดิมจาก Part 52/53 คือ Default 7200 วินาทีเช่นกันในหลาย Config —
แต่สำหรับ Design ที่ต้องการ Failover เร็ว ค่านี้สูงเกินไปมาก) — Holdtime ที่ยาวหมายถึง **Spoke
เชื่อว่า Hub เดิมยัง Reachable อยู่นานเกินจริง** (รอจนกว่า Registration จะหมดอายุตาม Holdtime
ก่อนจะยอมรับว่า NHS ตัวนั้น "หายไปแล้ว" อย่างสมบูรณ์) ทำให้แม้จะมี Backup NHS Config อยู่ครบ
ก็ยังโอนไปใช้ Backup ช้ากว่าที่ควรจะเป็น — ค่าที่แนะนำสำหรับ Design ที่ต้องการ Fast Failover
คือค่าเดียวกับที่ใช้ตอน Lab (Part 52 ตั้งไว้ที่ `300`) หรือต่ำกว่านั้นตามความเหมาะสมของ
Environment จริง

### 599.4 Root Cause สรุป

| จุดที่ผิด | อุปกรณ์ | ค่าที่ผิด | ค่าที่ถูกต้อง |
|---|---|---|---|
| Backup NHS ไม่ครบ | BRANCH-RTR | มี NHS แค่ WAN-EDGE-1 ตัวเดียว | ต้องมี WAN-EDGE-1 **และ** WAN-EDGE-2 ครบทั้งคู่ |
| NHRP Holdtime สูงเกินไป | BRANCH-RTR | `7200` วินาที | ปรับให้เหมาะกับ RTO ที่องค์กรต้องการ (Lab นี้ใช้ `300`) |

### 599.5 การแก้ไข

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast
BRANCH-RTR(config-if)# ip nhrp holdtime 300
BRANCH-RTR(config-if)# exit
```

### 599.6 Verify

```
BRANCH-RTR# show ip nhrp nhs detail
Tunnel0:
  172.16.100.1  E priority = 0 cluster = 0
      req-sent 52 req-failed 44 repl-recv 8
  172.16.100.2  RE priority = 0 cluster = 0
      req-sent 6 req-failed 0 repl-recv 6

BRANCH-RTR# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 100", distance 110, metric 1, candidate default path
  Routing Descriptor Blocks:
  * 172.16.100.2, from 1.1.1.22, 00:00:41 ago, via Tunnel0
      Route metric is 1, traffic share count is 1
```

BRANCH-RTR Register กับ WAN-EDGE-2 สำเร็จ (`RE`) และ Default Route Failover ไปใช้
`172.16.100.2` (WAN-EDGE-2) แล้วโดยอัตโนมัติ

> **ข้อควรจำสำหรับข้อสอบ**: Dual-Hub Design ต้อง**ตรวจ Config ของทุก Spoke ให้เหมือนกันเป๊ะ**
> เสมอ (`ip nhrp nhs` ต้องมีครบทุกตัวของ Hub ทุกสาขา, `ip nhrp holdtime` ต้องตรงกันทั้งวง) —
> ความผิดพลาดแบบ "Deploy ไม่ครบทุก Spoke" เป็นสาเหตุอันดับ 1 ของ Dual-Hub Failover ที่ทำงาน
> ไม่สมบูรณ์ในทางปฏิบัติจริง มากกว่าปัญหาทาง Technical เสียอีก

---

## Step 600 — Lab เต็มรูปแบบ: 3 ปัญหาซ้อนกันใน Dual-Hub DMVPN

### 600.1 สถานการณ์: Site สาขาใหม่ "BRANCH4-RTR" เข้าวงไม่สมบูรณ์

**BRANCH4-RTR** เป็น Spoke สาขาที่ 4 ที่กำลัง Deploy เข้าวง Dual-Hub DMVPN Phase 3 (NBMA =
`198.51.100.14`, Tunnel0 = `172.16.100.14/24`, LAN = `10.20.4.0/24`) — Engineer ที่ Deploy
Config รายงานว่า "Ping ไม่ผ่านจากสาขา 4 ไปสาขาอื่นเลย" และขอให้ตรวจสอบทั้งหมด **โจทย์นี้มี
3 ปัญหาซ้อนกันพร้อมกัน** (ตาม Brief ของ Part นี้): **(1)** Transform-Set Mismatch (แบบ
Scenario 2), **(2)** NHRP Redirect/Shortcut ขาดหาย (แบบ Scenario 5), **(3)** OSPF Network-Type
Mismatch (แบบ Scenario 6) — ให้ Diagnose ตาม Methodology (Step 591) แบบ Bottom-Up ทีละ Layer
จนเจอครบทั้ง 3 จุด

### 600.2 Config ทั้งหมดของ BRANCH4-RTR (ก่อนแก้ — มี Bug ซ่อนอยู่ 3 จุด)

```
BRANCH4-RTR(config)# crypto isakmp policy 10
BRANCH4-RTR(config-isakmp)# encryption aes 256
BRANCH4-RTR(config-isakmp)# hash sha256
BRANCH4-RTR(config-isakmp)# authentication pre-share
BRANCH4-RTR(config-isakmp)# group 14
BRANCH4-RTR(config-isakmp)# exit
BRANCH4-RTR(config)# crypto isakmp key Cisco@DMVPN123 address 0.0.0.0 0.0.0.0
!
BRANCH4-RTR(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha-hmac
BRANCH4-RTR(cfg-crypto-trans)# mode transport
BRANCH4-RTR(cfg-crypto-trans)# exit
! <-- Bug #1: esp-sha-hmac ไม่ตรงกับ esp-sha256-hmac ของ Hub/Spoke อื่น
BRANCH4-RTR(config)# crypto ipsec profile DMVPN-PROFILE
BRANCH4-RTR(ipsec-profile)# set transform-set DMVPN-TSET
BRANCH4-RTR(ipsec-profile)# exit
!
BRANCH4-RTR(config)# interface Loopback0
BRANCH4-RTR(config-if)# ip address 172.20.3.1 255.255.255.255
BRANCH4-RTR(config-if)# exit
!
BRANCH4-RTR(config)# interface Tunnel0
BRANCH4-RTR(config-if)# ip address 172.16.100.14 255.255.255.0
BRANCH4-RTR(config-if)# ip mtu 1400
BRANCH4-RTR(config-if)# ip nhrp authentication DMVPN-KEY
BRANCH4-RTR(config-if)# ip nhrp network-id 100
BRANCH4-RTR(config-if)# ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
BRANCH4-RTR(config-if)# ip nhrp nhs 172.16.100.2 nbma 203.0.113.6 multicast
BRANCH4-RTR(config-if)# ip ospf network broadcast
! <-- Bug #3: ต้องเป็น point-to-multipoint เหมือน Hub/Spoke อื่นทั้งหมด
BRANCH4-RTR(config-if)# ip ospf 100 area 51
BRANCH4-RTR(config-if)# tunnel source GigabitEthernet0/1
BRANCH4-RTR(config-if)# tunnel mode gre multipoint
BRANCH4-RTR(config-if)# tunnel key 100
BRANCH4-RTR(config-if)# tunnel protection ipsec profile DMVPN-PROFILE
BRANCH4-RTR(config-if)# exit
! <-- Bug #2 (แยกเป็นการขาดหาย ไม่ใช่บรรทัดผิด): ไม่มี "ip nhrp shortcut" เลยใน Interface นี้
!
BRANCH4-RTR(config)# router ospf 100
BRANCH4-RTR(config-router)# router-id 172.20.3.1
BRANCH4-RTR(config-router)# area 51 stub
BRANCH4-RTR(config-router)# passive-interface default
BRANCH4-RTR(config-router)# no passive-interface Tunnel0
BRANCH4-RTR(config-router)# network 172.16.100.0 0.0.0.255 area 51
BRANCH4-RTR(config-router)# network 10.20.4.0 0.0.0.255 area 51
BRANCH4-RTR(config-router)# exit
```

### 600.3 Diagnose ขั้นที่ 1 — Layer 1-2 (Underlay + IKE)

```
BRANCH4-RTR# show ip interface brief | include Tunnel0
Tunnel0                172.16.100.14   YES manual up                    up

BRANCH4-RTR# ping 203.0.113.2 source GigabitEthernet0/1
Success rate is 100 percent (5/5)

WAN-EDGE-1# show crypto isakmp sa | include 198.51.100.14
203.0.113.2     198.51.100.14   QM_IDLE           2011 ACTIVE
```

**Layer 1-2 ผ่านทั้งคู่** — Underlay Reachable, ISAKMP `QM_IDLE` สมบูรณ์ ไปตรวจ Layer 3 ต่อ

### 600.4 Diagnose ขั้นที่ 2 — Layer 3 (IPsec Phase 2) → พบ Bug #1

```
WAN-EDGE-1# show crypto ipsec sa peer 198.51.100.14 | include pkts encaps|esp sas
    #pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0

WAN-EDGE-1# debug crypto ipsec
*Sep 26 14:02:10.552: IPSEC(ipsec_process_proposal): proposal part #1,
  transform proposals not supported
```

**พบ Bug #1**: `#pkts encaps: 0` และ `transform proposals not supported` — ตรวจ Transform-Set
ทั้งสองฝั่งเทียบกัน:

```
WAN-EDGE-1# show crypto ipsec transform-set DMVPN-TSET
Transform set DMVPN-TSET: { esp-256-aes esp-sha256-hmac }
   will negotiate = { Transport, },

BRANCH4-RTR# show crypto ipsec transform-set DMVPN-TSET
Transform set DMVPN-TSET: { esp-256-aes esp-sha-hmac }
   will negotiate = { Transport, },
```

`esp-sha256-hmac` (WAN-EDGE-1) vs `esp-sha-hmac` (BRANCH4-RTR) — **ยืนยัน Bug #1** ตรงตามที่
คาดไว้ **แก้ทันที** ก่อนไปตรวจ Layer ถัดไป (เพราะ Layer บนกว่าจะ Diagnose ไม่ได้ผลจริงจนกว่า
Layer นี้จะผ่านก่อน):

```
BRANCH4-RTR(config)# crypto ipsec transform-set DMVPN-TSET esp-aes 256 esp-sha256-hmac
BRANCH4-RTR(cfg-crypto-trans)# mode transport
BRANCH4-RTR(cfg-crypto-trans)# exit
```

Verify:

```
WAN-EDGE-1# show crypto ipsec sa peer 198.51.100.14 | include pkts encaps
    #pkts encaps: 12, #pkts encrypt: 12, #pkts digest: 12
```

Layer 3 ผ่านแล้ว

### 600.5 Diagnose ขั้นที่ 3 — Layer 4-5 (NHRP + Routing) → พบ Bug #3

```
WAN-EDGE-1# show ip nhrp | include 172.16.100.14
172.16.100.14/32 via 172.16.100.14
   Tunnel0 created 00:01:02, expire 01:58:58
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 198.51.100.14
```

**NHRP ผ่านแล้ว** (Layer 4 ปกติ — เพราะ NHRP Registration ไม่ได้ผูกกับ OSPF Network Type
เลย) ไปตรวจ Layer 5 ต่อ:

```
WAN-EDGE-1# show ip ospf neighbor | include 172.20.3.1
(ไม่มีผลลัพธ์ — ไม่พบ Neighbor นี้เลย)

WAN-EDGE-1# debug ip ospf adj
*Sep 26 14:08:33.400: OSPF-100 ADJ  Tu0: Nbr 172.16.100.14 has larger interface
  MTU/Network-Type mismatch: my type POINT_TO_MULTIPOINT, hello Network Type BROADCAST
```

**พบ Bug #3**: Network-Type Mismatch ตรงตามที่คาดไว้ — **แก้ทันที**:

```
BRANCH4-RTR(config)# interface Tunnel0
BRANCH4-RTR(config-if)# ip ospf network point-to-multipoint
BRANCH4-RTR(config-if)# exit
```

Verify:

```
WAN-EDGE-1# show ip ospf neighbor | include 172.20.3.1
172.20.3.1        0   FULL/  -        00:01:52    172.16.100.14   Tunnel0
```

Layer 5 ผ่านแล้ว — ทดสอบ Ping ข้าม Site:

```
BRANCH4-RTR# ping 10.20.1.1 source 10.20.4.1 repeat 10
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

**Ping ผ่านสมบูรณ์แล้ว** — ดูเผินๆเหมือนจบ Lab แล้ว แต่ตาม Brief ยังมี **Bug #3 ที่ 3** ที่ยัง
ไม่ถูกตรวจพบ เพราะไม่ส่งผลต่อ Reachability (Ping ผ่านได้) — ต้องไปตรวจ Layer 6 ต่อให้ครบตาม
Methodology เสมอ ไม่หยุดแค่ Ping ผ่านแล้วสรุปว่าจบ (บทเรียนเดียวกับ Scenario 5)

### 600.6 Diagnose ขั้นที่ 4 — Layer 6 (Data Plane / Shortcut) → พบ Bug #2

```
BRANCH4-RTR# ping 10.20.1.1 source 10.20.4.1 repeat 50
(ping ผ่านครบ 50/50 — ผ่านตลอด)

BRANCH4-RTR# show ip nhrp shortcut

BRANCH4-RTR#
```

**ว่างเปล่า** — ทั้งที่ Ping ไปแล้ว 50 ครั้ง ไม่มี Shortcut Route เกิดขึ้นเลย ตรวจ Config ของ
Tunnel0:

```
BRANCH4-RTR# show run interface Tunnel0 | include shortcut

BRANCH4-RTR#
```

**ยืนยัน Bug #2**: ไม่มี `ip nhrp shortcut` อยู่ใน Config ของ BRANCH4-RTR เลย (Hub มี `ip nhrp
redirect` อยู่ครบตามปกติของ Part 53 — ตรวจแยกยืนยันด้วย `show run interface Tunnel0` ที่
WAN-EDGE-1/WAN-EDGE-2 ก็เจอว่ามีอยู่ครบ ไม่ใช่ปัญหาฝั่ง Hub) **แก้ทันที**:

```
BRANCH4-RTR(config)# interface Tunnel0
BRANCH4-RTR(config-if)# ip nhrp shortcut
BRANCH4-RTR(config-if)# exit
```

### 600.7 Verify สุดท้าย — ยืนยันทั้ง 3 ปัญหาถูกแก้ครบ

```
BRANCH4-RTR# ping 10.20.1.1 source 10.20.4.1 repeat 20
!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (20/20)

BRANCH4-RTR# show ip nhrp shortcut
172.16.100.11/32 via 172.16.100.11
   Tunnel0 created 00:00:03, expire 01:59:56
   Type: dynamic, Flags: router rib
   NBMA address: 198.51.100.2

WAN-EDGE-1# show crypto ipsec sa peer 198.51.100.14 | include pkts encaps
    #pkts encaps: 87, #pkts encrypt: 87, #pkts digest: 87

WAN-EDGE-1# show ip ospf neighbor | include 172.20.3.1
172.20.3.1        0   FULL/  -        00:05:20    172.16.100.14   Tunnel0

WAN-EDGE-1# show dmvpn brief | include 198.51.100.14
Tu0             198.51.100.14      172.16.100.14     UP     D
```

ครบทั้ง 4 จุดยืนยัน: **IPsec เข้ารหัสจริง (`pkts encaps` เพิ่มขึ้น), OSPF Neighbor `FULL`,
NHRP Shortcut เกิดขึ้นจริง (`Flags: router rib`), และ DMVPN Brief แสดง `UP/D`** — Site
BRANCH4-RTR เข้าวง Dual-Hub Phase 3 DMVPN สมบูรณ์ครบทุก Layer ตาม Methodology ของ Step 591

### 600.8 สรุปลำดับการ Diagnose ทั้ง Lab

| ลำดับ | Layer ที่ตรวจ | คำสั่งที่ใช้เจอปัญหา | Bug ที่พบ | การแก้ |
|---|---|---|---|---|
| 1 | Underlay + IKE (L1-2) | `ping`, `show crypto isakmp sa` | ไม่พบ (ผ่านปกติ) | - |
| 2 | IPsec Phase 2 (L3) | `show crypto ipsec sa`, `debug crypto ipsec` | Transform-Set Mismatch (`esp-sha-hmac` vs `esp-sha256-hmac`) | แก้ `crypto ipsec transform-set` ให้ตรงกัน |
| 3 | NHRP (L4) | `show ip nhrp` | ไม่พบ (ผ่านปกติ) | - |
| 4 | Routing (L5) | `show ip ospf neighbor`, `debug ip ospf adj` | OSPF Network-Type Mismatch (`broadcast` vs `point-to-multipoint`) | แก้ `ip ospf network point-to-multipoint` |
| 5 | Data Plane (L6) | `show ip nhrp shortcut` | ขาด `ip nhrp shortcut` บน Spoke | เพิ่ม `ip nhrp shortcut` |

> **บทเรียนสุดท้ายของ Part นี้**: การ Diagnose แบบ Bottom-Up ตาม Methodology ทำให้เจอปัญหาที่
> ซ้อนกันได้ทีละชั้นอย่างเป็นระบบ โดยไม่พลาดจุดใดจุดหนึ่งไป — ถ้าหยุดที่ "Ping ผ่านแล้ว" (หลัง
> แก้ Bug #1 และ #3) โดยไม่ไล่ตรวจ Layer 6 ต่อให้ครบ จะพลาด Bug #2 (Shortcut ขาดหาย) ไปเลย
> ซึ่งเป็นปัญหาเชิง Performance/Scale ที่ไม่กระทบ Reachability ทันทีแต่ส่งผลระยะยาวต่อ Hub
> Bandwidth เมื่อจำนวน Spoke เพิ่มขึ้น — **Methodology ที่ครบทุก Layer สำคัญกว่าการรีบสรุปว่า
> "Ping ผ่านแล้วจบ" เสมอ**

---

## แบบฝึกหัดทวนความเข้าใจ Part 60

1. `show crypto isakmp sa` แสดง State ค้างอยู่ที่ `MM_KEY_EXCH` ควรสงสัยสาเหตุอะไรเป็นอันดับแรก
   และต้องใช้คำสั่งอะไรต่อเพื่อยืนยัน Root Cause ให้ชัดเจน?
2. เพราะเหตุใด DMVPN ที่ใช้ `tunnel protection ipsec profile` จึงไม่มีปัญหา Crypto ACL/Traffic
   Selector Mismatch แบบเดียวกับ IPsec Crypto Map ดั้งเดิม?
3. `show ip nhrp shortcut` ว่างเปล่าทั้งที่ Ping ข้าม Spoke ผ่านได้ปกติ แปลว่าอะไร และต้อง
   ตรวจสอบ Config บรรทัดไหนที่ Hub และ Spoke?
4. อธิบายว่าทำไม Route ที่มี Administrative Distance ดีกว่า (เช่น Static Route) ที่ชี้ออก
   Physical Interface ตรงๆ ถึงทำให้เกิด Asymmetric Encryption ได้ ทั้งที่ Config IPsec ทั้งสอง
   ฝั่งถูกต้องสมบูรณ์?
5. Dual-Hub DMVPN ที่ Spoke ตัวหนึ่ง Failover ช้ากว่าตัวอื่นมาก ควรตรวจ Config อะไร 2 อย่าง
   เป็นอันดับแรก และคำสั่งอะไรที่ใช้ยืนยันแต่ละอย่าง?

**เฉลย:**

1. สงสัย **Pre-Shared Key ไม่ตรงกัน** เป็นอันดับแรก (หรือ ISAKMP Policy Parameter ไม่ Match) —
   ใช้ `debug crypto isakmp` แล้วอ่าน Log: ถ้าเจอ `HASH not authenticated` = PSK ผิด, ถ้าเจอ
   `atts are not acceptable` = Policy (Encryption/Hash/Group) ไม่ตรงกัน
2. เพราะ `tunnel protection ipsec profile` ไม่ใช้ Crypto ACL (`match address`) แบบ Crypto Map
   เดิม — Traffic Selector เป็นแบบ Implicit ที่ผูกกับคู่ NBMA IP โดยตรง (`protocol 47` ระหว่าง
   Tunnel Source/Destination เท่านั้น) จึงไม่มีทางที่ ACL สองฝั่งจะไม่ Mirror กันได้เหมือน
   Crypto Map แบบเดิม
3. แปลว่า Phase 3 NHRP Redirect/Shortcut ไม่ทำงาน (Traffic ยัง Hairpin ผ่าน Hub อยู่ แม้ Ping
   จะผ่านปกติ) — ต้องตรวจ `ip nhrp redirect` บน Interface Tunnel ของ **Hub** และ `ip nhrp
   shortcut` บน Interface Tunnel ของ **Spoke** ว่ามีครบทั้งคู่หรือไม่
4. เพราะ Route Selection ใน RIB เลือกตาม Administrative Distance ก่อนเสมอ ไม่ว่า Routing
   Protocol/Tunnel Config จะถูกต้องแค่ไหน — ถ้ามี Static Route (AD=1) ชี้ Subnet ปลายทางเดียว
   กันออก Physical Interface Traffic จะถูก Forward ผ่านทางนั้นแทน OSPF Route ที่ชี้ผ่าน Tunnel
   (AD=110) โดยอัตโนมัติ ทำให้ Traffic ทางนั้น Bypass GRE/IPsec ไปเลยโดยไม่มี Error เตือน
5. ตรวจ **(1)** `show ip nhrp nhs detail` ว่ามี NHS ครบทุกตัวของทุก Hub หรือไม่ (Backup NHS
   อาจ Config ไม่ครบ) และ **(2)** `show run interface Tunnel0 | include holdtime` ว่า
   `ip nhrp holdtime` ตั้งไว้สูงเกินไปหรือไม่ (ทำให้รอนานก่อนตัดสินว่า NHS เดิมตายแล้ว)

---

## สรุป Part 60

Part นี้นำ **DMVPN Dual-Hub Phase 3 Lab** จาก Part 51-53 มาผ่านกระบวนการ Systematic
Troubleshooting แบบ Layered (Underlay → IKE → IPsec → NHRP → Routing → Data Plane) ครบ 8
Scenario ที่ครอบคลุมทุกจุดที่ ENARSI Exam ชอบทดสอบ: PSK/Policy Mismatch, Transform-Set
Mismatch, NHRP NHS ผิด, mGRE/P2P Mismatch, Redirect/Shortcut ขาดหาย, OSPF Network-Type
Mismatch, Asymmetric Encryption จาก Routing Table, และ Dual-Hub Failover ที่ไม่สมบูรณ์ — ปิด
ท้ายด้วย Lab ที่รวม 3 ปัญหาซ้อนกันในโจทย์เดียว พิสูจน์ว่า Methodology แบบ Bottom-Up คือกุญแจ
สำคัญที่สุดในการ Diagnose VPN/DMVPN ทุกกรณี ไม่ว่าจะซับซ้อนแค่ไหนก็ตาม

**พร้อมสำหรับ Part 61**: เราจะขยับจาก DMVPN/VPN ไปสู่ **Infrastructure Security
Troubleshooting** — ปัญหาที่พบบ่อยใน AAA, Device Hardening, Control Plane Policing, ACL/ZBFW
และ 802.1X/TrustSec ที่เรียนไว้ตั้งแต่ Part 21-22, 43-46 ของหลักสูตร มาผ่านกระบวนการ Diagnose
แบบ ENARSI เดียวกันกับ Part นี้

**ไปต่อ:** [Part 61 — Infrastructure Security Troubleshooting →](part-061-infra-security-troubleshooting.md)
