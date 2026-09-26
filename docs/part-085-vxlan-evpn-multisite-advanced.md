# Part 85 — VXLAN/EVPN Multisite Advanced
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 841–850 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> Part นี้ต่อยอดจาก **[Part 84 — VXLAN/EVPN Fundamentals](part-084-vxlan-evpn-fundamentals.md)**
> ซึ่งสร้าง DC Fabric แรก (**DC1**) แบบ Leaf-Spine พื้นฐาน: `DC-SPINE-1/DC-SPINE-2` (Spine),
> `DC-LEAF-1/DC-LEAF-2` (Leaf/VTEP, ต่อ L3 handoff ไปยัง `CORE-SW1/CORE-SW2`), VXLAN/EVPN Type-2/3
> สำหรับ L2 overlay, Distributed Anycast Gateway บน Leaf — ทั้งหมดอยู่ใน **Single Fabric / Single Site**
>
> Part นี้จะ (1) อธิบายว่าทำไม Single Fabric ถึง "ไม่ scale ข้าม Site", (2) สร้าง **DC2**
> (Site สำรอง/DR) ที่มีสถาปัตยกรรมแบบเดียวกัน (`DC2-SPINE-1/2`, `DC2-LEAF-1/2`), และ (3) เชื่อม
> DC1 ↔ DC2 ด้วย **EVPN Multisite** ผ่าน **Border Gateway (BGW)** — เทคโนโลยีระดับ CCIE ที่ทำให้
> Data Center หลาย Site ทำงานเสมือน Fabric เดียวกัน (สำหรับ Workload Mobility) โดยไม่รวม
> Fault Domain ของ Underlay/Control Plane เข้าด้วยกัน

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 841 | ทำไมต้อง Multisite — ข้อจำกัดของ Single Fabric ข้าม Site |
| 842 | Border Gateway (BGW) — บทบาท และสถาปัตยกรรมเต็มรูปแบบ DC1 + DC2 |
| 843 | Multisite Control Plane — eBGP EVPN ระหว่าง BGW ข้าม Site |
| 844 | Multisite Data Plane — VXLAN Tunnel BGW-to-BGW + Config เต็มรูปแบบ |
| 845 | L2 Stretch ข้าม Site — DC-SERVERS VNI และการจัดการ BUM Traffic |
| 846 | L3 Stretch และ Type-5 EVPN Route — Routed Inter-Site Communication |
| 847 | DCI Transport Options — Dark Fiber, DWDM, Routed IP WAN |
| 848 | Failure Scenario และ Site Isolation — Fault Domain แยกจริงหรือไม่ |
| 849 | Multisite และ Disaster Recovery Design |
| 850 | Full Lab — EVPN Multisite DC1 ↔ DC2 (L2 Stretch + L3-only) พร้อม Verification |

---

## ส่วนขยาย IP/Loopback/VNI Plan สำหรับ Part 84-85

> **หมายเหตุสำคัญ**: ตารางนี้เป็น**ส่วนขยายใหม่**สำหรับ DC Fabric (Part 84-85) ที่ยังไม่ถูกรวมเข้า
> `00-ip-address-plan.md` (canonical reference หลักของหลักสูตร) — ออกแบบให้สอดคล้องกับรูปแบบเดิม
> (Loopback0 range `1.1.1.x/32`, private RFC1918) จะถูกรวมเข้า canonical plan อย่างเป็นทางการเมื่อ
> ทีมงานอัปเดต `00-ip-address-plan.md` ในการ revision ครั้งต่อไป — ไม่ใช่ค่าที่แก้ไขในเอกสารนั้นตรงๆ

### Loopback Plan — DC Fabric

| Device | Loopback0 (RID) | Loopback1 (PIP / NVE source) | Loopback100 (Anycast VIP) |
|---|---|---|---|
| DC-SPINE-1 (Part 84) | 1.1.1.101/32 | - | - |
| DC-SPINE-2 (Part 84) | 1.1.1.102/32 | - | - |
| DC-LEAF-1 (Part 84) | 1.1.1.103/32 | (ใช้ Lo0 เป็น NVE source ตรง — single fabric ไม่ต้องแยก PIP/VIP) | - |
| DC-LEAF-2 (Part 84) | 1.1.1.104/32 | (เช่นเดียวกับ DC-LEAF-1) | - |
| **DC-BGW-1** (ใหม่ Part 85) | 1.1.1.105/32 | 1.1.1.106/32 | 1.1.1.109/32 (ร่วมกับ DC-BGW-2) |
| **DC-BGW-2** (ใหม่ Part 85) | 1.1.1.107/32 | 1.1.1.108/32 | 1.1.1.109/32 (ร่วมกับ DC-BGW-1) |
| **DC2-SPINE-1** (ใหม่) | 1.1.1.111/32 | - | - |
| **DC2-SPINE-2** (ใหม่) | 1.1.1.112/32 | - | - |
| **DC2-LEAF-1** (ใหม่) | 1.1.1.113/32 | (ใช้ Lo0) | - |
| **DC2-LEAF-2** (ใหม่) | 1.1.1.114/32 | (ใช้ Lo0) | - |
| **DC2-BGW-1** (ใหม่) | 1.1.1.115/32 | 1.1.1.116/32 | 1.1.1.119/32 (ร่วมกับ DC2-BGW-2) |
| **DC2-BGW-2** (ใหม่) | 1.1.1.117/32 | 1.1.1.118/32 | 1.1.1.119/32 (ร่วมกับ DC2-BGW-1) |

> **จุดสำคัญ**: BGW เท่านั้นที่ต้องมี Loopback 3 ชุด (RID / PIP / Anycast VIP) — Leaf/Spine ธรรมดา
> ใช้ Loopback0 เดียวพอ เพราะ concept "Anycast VTEP ร่วมของหลายอุปกรณ์" มีความหมายเฉพาะตอนต้อง
> ทำ Multisite เท่านั้น

### BGP AS และ Multisite Site-ID

| Site | Underlay/Overlay AS (iBGP ภายใน Site) | EVPN Multisite Site-ID |
|---|---|---|
| DC1 | AS 65001 (Spine = Route-Reflector) | 1 |
| DC2 | AS 65002 (Spine = Route-Reflector) | 2 |
| DC1 ↔ DC2 | **eBGP** ระหว่าง BGW (AS 65001 ↔ AS 65002) | - |

### DCI Transit Link Plan (/30, Routed)

| Link | Subnet | ปลาย A | ปลาย B |
|---|---|---|---|
| DC-BGW-1 ↔ DC2-BGW-1 | 10.85.1.0/30 | DC-BGW-1 .1 | DC2-BGW-1 .2 |
| DC-BGW-2 ↔ DC2-BGW-2 | 10.85.2.0/30 | DC-BGW-2 .1 | DC2-BGW-2 .2 |

### VNI / VRF Plan

| VLAN | ชื่อ | L2 VNI | Subnet DC1 | Subnet DC2 | Stretch ข้าม Site? | Anycast GW |
|---|---|---|---|---|---|---|
| 130 | DC-SERVERS | 30130 | 10.130.30.0/24 | 10.130.30.0/24 (**subnet เดียวกัน**) | ✅ L2 Stretch | 10.130.30.1 |
| 140 | DC-APP-DC1 | 30140 | 10.140.30.0/24 | - | ❌ (L3-only, local DC1) | 10.140.30.1 |
| 141 | DC-APP-DC2 | 30141 | - | 10.141.30.0/24 | ❌ (L3-only, local DC2) | 10.141.30.1 |
| VRF | TENANT-PROD | L3 VNI **50001** | ใช้ร่วมกันทั้งสอง Site เพื่อ route ระหว่าง VLAN140 ↔ VLAN141 ผ่าน Type-5 |

> **Route-Target ข้าม AS**: เพราะ DC1 (AS65001) และ DC2 (AS65002) เป็นคนละ AS, ค่า RT แบบ `auto`
> (ที่ embed AS ไว้ในตัว) จะ**ไม่ match กัน** — Part นี้ใช้ RT แบบ **explicit** ที่เหมือนกันทั้งสอง Site:
> `route-target both 1:30130` (สำหรับ L2 VNI 30130) และ `route-target both 1:50001` (สำหรับ VRF
> TENANT-PROD / L3 VNI 50001) — นี่คือ gotcha ที่ CCIE Lab ชอบเอามาทดสอบ

---

## Step 841 — ทำไมต้อง Multisite: ข้อจำกัดของ Single Fabric ข้าม Site

Part 84 สร้าง VXLAN/EVPN Fabric เดียว (DC1) ที่ Leaf ทุกตัวเป็น VTEP, Spine ทำ Route-Reflector
ให้ BGP EVPN, และทุกอุปกรณ์อยู่ใน **Underlay Convergence Domain เดียวกัน** — คำถามคือ "ทำไมไม่ขยาย
Fabric เดียวกันนี้ให้ครอบคลุมทั้ง DC1 และ DC2 (ลาก Spine ไปอีก Site) เลย?"

### ปัญหาที่ 1 — Fault Domain ไม่ถูกแยก (Blast Radius)

ใน Single Fabric, ทุก Leaf/Spine อยู่ใน BGP AS เดียวกัน, OSPF Area เดียวกัน (หรือ EVPN overlay
เดียวกัน) ถ้าเกิดปัญหาที่ DC1 เช่น:

- Underlay OSPF flap ที่ DC-SPINE-1 (bad cable, microloop)
- BGP EVPN Route-Reflector ที่ DC-SPINE-1/2 ประมวลผล route churn จำนวนมาก (MAC move storm)
- Broadcast/Multicast storm จาก VM ที่ misconfigure

ปัญหานี้จะ**แพร่ (propagate)** ไปยัง DC2 ทันที เพราะ DC2 เป็นสมาชิกของ Control Plane/Underlay
เดียวกัน — นี่คือสิ่งที่ตรงข้ามกับเป้าหมายของการมี Site สำรอง (DR Site) ที่ควรจะ**ไม่ได้รับผลกระทบ**
เมื่อ Site หลักมีปัญหา

### ปัญหาที่ 2 — WAN Bandwidth ≠ DC Fabric Bandwidth

| คุณสมบัติ | DC Fabric ภายใน Site (Leaf-Spine) | WAN ระหว่าง Site (DCI) |
|---|---|---|
| Bandwidth ปกติ | 100G/400G ต่อ link, Non-blocking CLOS | 1G–10G (dark fiber/DWDM) หรือน้อยกว่า (Routed WAN) |
| Latency | < 1 ms (ภายในห้อง Data Center) | 1–50+ ms (ขึ้นกับระยะทาง) |
| BUM Traffic (Broadcast/Unknown-Unicast/Multicast) | รับได้เพราะ bandwidth สูง | **ถ้าไม่ควบคุม จะ saturate WAN link ได้ง่าย** |
| Multicast Underlay (PIM) สำหรับ BUM replication | ใช้ได้ปกติ | ผู้ให้บริการ WAN ส่วนใหญ่ไม่รองรับ PIM ข้าม Site |

ถ้าลาก Fabric เดียวกันข้าม WAN ตรงๆ (extend Spine-Leaf underlay ข้าม Site) เท่ากับเอา BUM
Replication แบบ Data-Center-internal ไปวิ่งบน WAN ที่ bandwidth ต่ำกว่ามาก — Broadcast Storm ครั้งเดียว
ที่ DC1 อาจทำให้ WAN link เต็มและกระทบ Traffic การ Production อื่นที่ใช้ link เดียวกัน

### ปัญหาที่ 3 — BGP/Underlay Convergence Domain ใหญ่เกินไป

จำนวน VTEP, จำนวน MAC/IP ที่ต้อง sync ผ่าน BGP EVPN Type-2/Type-3 เพิ่มเป็น 2x เมื่อรวม 2 Site
เข้า Control Plane เดียวกัน — Convergence Time ในกรณี Failure จะแปรผันตามขนาด Domain
(ยิ่ง Domain ใหญ่ ยิ่งใช้เวลา re-converge นานขึ้น และ blast radius ของ event เดียวก็ใหญ่ขึ้นตามไปด้วย)

### ทางออก — EVPN Multisite Architecture

Cisco แก้ปัญหานี้ด้วยแนวคิด **"Fabric ของ Fabric" (Fabric-of-Fabrics)**: แต่ละ Site ยังคงเป็น
Fabric อิสระของตัวเอง (Underlay/BGP AS/Convergence Domain แยกกันสมบูรณ์) แต่เชื่อมต่อกันผ่าน
อุปกรณ์พิเศษที่เรียกว่า **Border Gateway (BGW)** ซึ่งทำหน้าที่:

1. **Terminate** Local Fabric's EVPN domain ที่ตัวเอง (เป็นจุดสิ้นสุดของ Site)
2. **Re-originate** EVPN Route ไปยัง Site อื่น (ไม่ leak ราย detail ของ Underlay ข้าม Site)
3. **แยก Fault Domain**: ปัญหาภายใน Site หนึ่งจะไม่ทำให้ BGP/Underlay ของอีก Site ไม่มั่นคง

```
   [Single Fabric ข้าม Site — ❌ ไม่แนะนำ]        [EVPN Multisite — ✅ แนวทางที่ถูก]

   DC1 ═══════════ WAN ═══════════ DC2            DC1 [BGW] ══ WAN (eBGP) ══ [BGW] DC2
   (BGP AS เดียว, OSPF Area เดียว,                 (BGP AS65001, Convergence      (BGP AS65002, Convergence
    Convergence Domain เดียวกันทั้งหมด)              Domain แยก)  BGW แยกออก      Domain แยก)  BGW แยกออก
                                                      จาก Fault Domain             จาก Fault Domain
```

Step ถัดไปจะลงรายละเอียด BGW และสถาปัตยกรรมเต็มรูปแบบของ DC1+DC2

---

## Step 842 — Border Gateway (BGW): บทบาท และสถาปัตยกรรมเต็มรูปแบบ

### BGW คืออะไร

**Border Gateway (BGW)** คืออุปกรณ์ (มักเป็น Nexus 9000 คู่ Active/Active) ที่วางอยู่ **"ขอบ"**
ของแต่ละ Site ทำหน้าที่พิเศษ 2 อย่างที่ Leaf/Spine ปกติไม่ทำ:

1. **VTEP ปกติ** — มี NVE interface, encapsulate/decapsulate VXLAN, เป็นสมาชิกของ Local Fabric
   (เชื่อมกับ Local Spine ผ่าน iBGP + OSPF Underlay เหมือน Leaf ทั่วไป)
2. **Multisite Border-Gateway Role** — มี Anycast VIP Loopback ร่วมกันระหว่าง BGW คู่ในไซต์เดียวกัน,
   ทำ **Next-Hop Rewrite** ให้ทุก EVPN Route ที่ส่งออกไป Site อื่น (Local VTEP ของ Leaf ไม่ต้อง
   ถูก expose ให้ Site อื่นรู้จักเลย — Site อื่นเห็นแค่ BGW Anycast VIP เท่านั้น)

> **สำคัญ**: BGW **ไม่ใช่** Border Leaf (อุปกรณ์ที่ทำ L3 handoff ไปยัง Campus/Core เช่น
> `DC-LEAF-1/2` ที่ต่อกับ `CORE-SW1/CORE-SW2` ใน Part 84) — สองบทบาทนี้แยกกันคนละหน้าที่
> (Border Leaf = ทางออกไป Campus, Border Gateway = ทางออกไป Site อื่น) แม้ในทางปฏิบัติบางองค์กร
> จะรวม 2 role ไว้ในอุปกรณ์เดียวกันเพื่อประหยัด hardware ก็ทำได้ แต่ Part นี้ใช้อุปกรณ์แยก
> (`DC-BGW-1/2`) เพื่อความชัดเจนของ Fault Domain ตามแนวทาง CCIE Design Best Practice

### สถาปัตยกรรมเต็มรูปแบบ DC1 + DC2

```
        DC1 — Primary Site (AS 65001, Multisite Site-ID 1)         DC2 — DR Site (AS 65002, Multisite Site-ID 2)
                     [จาก Part 84]                                          [สร้างใหม่ Part 85]

         ┌───────────────┐   ┌───────────────┐
         │   CORE-SW1     │   │   CORE-SW2     │                    (ไม่มี Core/Campus ที่ DC2 ใน Lab นี้
         └───────┬───────┘   └───────┬───────┘                       DC2 เป็น DR Site ที่รับ workload
                 │                   │                                 ผ่าน Multisite เท่านั้น)
         ┌───────┴───────────────────┴───────┐
         │      DC-LEAF-1  /  DC-LEAF-2        │  <- Border Leaf (L3 handoff → Campus)
         │      VTEP: VNI 30130 (DC-SERVERS)   │
         │            VNI 30140 (DC-APP-DC1)   │
         └───────┬───────────────────┬───────┘
         ┌───────┴───────┐   ┌───────┴───────┐
         │  DC-SPINE-1    │===│  DC-SPINE-2    │   <- Underlay OSPF Area0 + iBGP Route-Reflector (AS65001)
         └───────┬───────┘   └───────┬───────┘
         ┌───────┴───────────────────┴───────┐
         │        DC-BGW-1   /   DC-BGW-2       │   <- Border Gateway, Multisite Site-ID 1
         │   Lo100 Anycast VIP = 1.1.1.109/32    │      (VTEP ปกติ + Multisite Next-Hop Rewrite)
         └────────────────┬──────────────────┘
                          │
              DCI Transport (Step 847)
          Dark Fiber / DWDM / Routed IP WAN
                          │
                eBGP Multisite: AS65001 ⟷ AS65002
                          │
         ┌────────────────┴──────────────────┐
         │       DC2-BGW-1  /  DC2-BGW-2        │   <- Border Gateway, Multisite Site-ID 2
         │   Lo100 Anycast VIP = 1.1.1.119/32    │
         └───────┬───────────────────┬───────┘
         ┌───────┴───────┐   ┌───────┴───────┐
         │ DC2-SPINE-1    │===│ DC2-SPINE-2    │   <- Underlay OSPF Area0 + iBGP Route-Reflector (AS65002)
         └───────┬───────┘   └───────┬───────┘
         ┌───────┴───────────────────┴───────┐
         │      DC2-LEAF-1  /  DC2-LEAF-2       │   <- VTEP (DR Workload)
         │      VNI 30130 (DC-SERVERS, stretch) │
         │      VNI 30141 (DC-APP-DC2, local)   │
         └───────┬───────────────────┬───────┘
            Server/VM Rack DC1            Server/VM Rack DC2 (DR)
       (10.130.30.0/24, 10.140.30.0/24)  (10.130.30.0/24 stretch, 10.141.30.0/24)
```

### อุปกรณ์ใหม่ที่เพิ่มเข้าหลักสูตรใน Part 85

| Hostname | Role | Platform แนะนำ |
|---|---|---|
| DC-BGW-1, DC-BGW-2 | Border Gateway — DC1 Multisite (Site-ID 1) | Nexus 9300-EX/FX (BGW-capable) |
| DC2-SPINE-1, DC2-SPINE-2 | Spine — DC2 (Route-Reflector) | Nexus 9500 |
| DC2-LEAF-1, DC2-LEAF-2 | Leaf/VTEP — DC2 (DR Workload) | Nexus 9300 |
| DC2-BGW-1, DC2-BGW-2 | Border Gateway — DC2 Multisite (Site-ID 2) | Nexus 9300-EX/FX (BGW-capable) |

### ทำไมต้อง BGW เป็นคู่ (Redundant Pair) เสมอ

BGW เป็นจุด Single-Point-of-Failure ที่อันตรายมากถ้ามีตัวเดียว (ถ้า BGW ล้ม = Site ทั้งหมดหลุดจาก
Multisite ไปคุยกับ Site อื่นไม่ได้เลย) จึงต้องมีอย่างน้อย 2 ตัวต่อ Site เสมอ และใช้ **Anycast VIP
Loopback (Loopback100)** ร่วมกันเพื่อให้:

- Remote Site เห็น BGW ของ Local Site เป็น **VTEP เดียว** (Anycast IP เดียว) แม้จะมี 2 อุปกรณ์จริง
- Traffic กระจาย (ECMP) ไปยัง BGW ทั้งสองตัวได้ตามปกติของ Underlay Routing
- ถ้า BGW ตัวหนึ่งล้ม อีกตัวยังตอบ Anycast VIP ได้ต่อ — Remote Site **ไม่ต้อง Re-converge** อะไรเลย
  (เห็น next-hop เดิม, แค่ traffic ไปออกที่ BGW ตัวที่เหลือ)

---

## Step 843 — Multisite Control Plane: eBGP EVPN ระหว่าง BGW ข้าม Site

### ทำไมใช้ eBGP ระหว่าง Site (ผูกกับทักษะ Part 29)

จาก **[Part 29 — BGP Fundamentals (eBGP)](part-029-bgp-fundamentals.md)** เราเรียนไปแล้วว่า
"คนละ AS = ใช้ eBGP" — Multisite ใช้หลักการเดียวกันตรงตัว: DC1 อยู่ใน AS 65001, DC2 อยู่ใน AS
65002 → BGW ของทั้งสอง Site จึงเป็น **eBGP Peer** กัน (ไม่ใช่ iBGP) เหตุผลที่เลือกให้แต่ละ Site
มี AS ของตัวเอง (ไม่ใช้ AS เดียวกันทั้งสอง Site):

| เหตุผล | รายละเอียด |
|---|---|
| Loop Prevention แบบ Native | eBGP มี AS-Path loop detection ในตัว ไม่ต้องพึ่ง Split-Horizon เพิ่มเติมสำหรับ Underlay routes |
| แยก Administrative Domain ชัดเจน | สอดคล้องกับแนวคิด "แต่ละ Site เป็น Autonomous Fabric" ตาม Step 841 |
| Design ที่ Cisco แนะนำจริงในเอกสาร CVD (Cisco Validated Design) สำหรับ EVPN Multisite ส่วนใหญ่ | ใช้ AS ต่อ Site (แต่ก็รองรับ same-AS ด้วย `local-as`/`as-override` ถ้าจำเป็น) |

### สิ่งที่ eBGP ระหว่าง BGW ทำ: ส่ง EVPN Route (ไม่ใช่ Underlay เต็มรูปแบบ)

BGP Session ระหว่าง BGW ข้าม Site เปิด 2 Address-Family:

1. **`address-family ipv4 unicast`** — ใช้แค่โฆษณา Loopback ของ BGW เอง (RID, PIP, Anycast VIP)
   ให้ BGW อีกฝั่งรู้จัก (จำเป็นสำหรับ VXLAN tunnel ระหว่าง BGW-to-BGW เท่านั้น) — **ไม่โฆษณา
   Loopback ของ Leaf/Spine ภายใน Site ออกไปเลย**
2. **`address-family l2vpn evpn`** — ส่ง EVPN Route (Type-2 MAC/IP, Type-3 IMET, Type-5 IP-Prefix)
   ที่ BGW **Re-originate** จาก Local Site ไปยัง Site อื่น

### กลไกหัวใจ: Next-Hop Rewrite (สิ่งที่ทำให้ "ซ่อน Underlay" ได้จริง)

นี่คือกลไกที่ตอบคำถาม "ซ่อนรายละเอียด Underlay ของ Site ตัวเองจาก Site อื่นได้อย่างไร":

```
DC-LEAF-1 (VTEP จริง 1.1.1.103) เรียนรู้ Host H1 (VLAN130)
        │  ส่ง EVPN Type-2 Route: Next-Hop = 1.1.1.103 (DC-LEAF-1 loopback)
        ▼
DC-SPINE-1/2 (Route-Reflector, AS65001) — reflect ตรงๆ ไม่แก้ Next-Hop
        │
        ▼
DC-BGW-1/2 (iBGP client ของ Spine)  <-- **จุดที่ Next-Hop ถูก Rewrite**
        │  Re-originate Route เดิม แต่เปลี่ยน Next-Hop เป็น
        │  Loopback100 (Anycast VIP ของ DC1 = 1.1.1.109) ก่อนส่งออก eBGP
        ▼
eBGP ไปยัง DC2-BGW-1/2 (AS65002)
        │  DC2 เห็น Routeของ H1 มี Next-Hop = 1.1.1.109 (DC1 BGW VIP)
        │  **ไม่เคยเห็น 1.1.1.103 (DC-LEAF-1 จริง) เลย**
        ▼
DC2-SPINE-1/2 reflect ต่อไปยัง DC2-LEAF-1/2 (Next-Hop ยังเป็น 1.1.1.109 เดิม
        │  เพราะ Spine ของ DC2 ไม่ rewrite — rewrite ทำที่ BGW เท่านั้น)
        ▼
DC2-LEAF-1/2 เห็น Host H1 อยู่หลัง VTEP = 1.1.1.109 (DC-BGW Anycast VIP)
```

**ผลลัพธ์**: DC2-LEAF ไม่จำเป็นต้องมี Underlay Reachability ไปยัง `1.1.1.103` (DC-LEAF-1 จริง)
เลย — รู้จักแค่ `1.1.1.109` (BGW Anycast VIP ของ DC1) ซึ่งอยู่ห่างแค่ 1 "next-hop" ผ่าน DCI link
ที่มี Underlay Reachability ตรงไปตรงมา นี่คือสิ่งที่ทำให้ Convergence Domain ของ DC1's Leaf/Spine
Underlay **ไม่รั่วไหล** ไปยัง DC2 เลย (ตอบโจทย์ Step 841 ปัญหาที่ 1 และ 3)

### ตัวอย่าง BGP Config ส่วน Control Plane (DC-BGW-1)

```
router bgp 65001
  router-id 1.1.1.105
  ! --- iBGP ไปยัง Local Spine (Route-Reflector) ---
  neighbor 1.1.1.101
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
  neighbor 1.1.1.102
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
  ! --- eBGP Multisite ไปยัง DC2-BGW-1 ---
  neighbor 10.85.1.2
    remote-as 65002
    ebgp-multihop 5
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community both
```

> **หมายเหตุ**: บางแพลตฟอร์ม NX-OS มีคำสั่ง `rewrite-evpn-rt-asn` ภายใต้ neighbor เพื่อ auto-rewrite
> ส่วน ASN ใน Route-Target เมื่อข้าม AS (กรณีใช้ RT แบบ `auto`) — Part นี้เลือกใช้ **explicit
> Route-Target** ที่เหมือนกันทั้งสอง Site แทน (ดู Step 846) เพราะ predictable และ debug ง่ายกว่า
> ในสภาพแวดล้อม Multi-AS

---

## Step 844 — Multisite Data Plane: VXLAN Tunnel BGW-to-BGW + Config เต็มรูปแบบ

### หลักการ Data Plane

ก่อนมี Multisite ถ้า Leaf ของ DC1 ต้องส่ง VXLAN packet ไปยัง Leaf ของ DC2 ตรงๆ (Leaf-to-Leaf
ข้าม WAN) — Underlay ต้องมี Reachability เต็มรูปแบบข้าม Site (ขัดกับ Step 841/843) หลังมี
Multisite, ทุก VXLAN Tunnel ที่ข้าม Site จะ**เดินทางผ่าน BGW ทั้งสองฝั่งเสมอ**:

```
Host H1 (DC1, VLAN130) ──VXLAN──> DC-LEAF-1 ──VXLAN──> DC-BGW-1/2 (decap + re-encap)
                                                              │
                                                    DCI (VXLAN header ใหม่
                                                    src=Lo1 DC-BGW, dst=Lo100 DC2-BGW)
                                                              │
                                    DC2-BGW-1/2 (decap + re-encap) ──VXLAN──> DC2-LEAF-1 ──> Host H2 (DC2)
```

**Leaf-to-Leaf ข้าม Site ไม่มีทาง (ไม่มี VXLAN tunnel ตรง)** — ทุก packet ถูก **Terminate และ
Re-originate ที่ BGW ทั้งสองฝั่ง** เสมอ (2 hop VXLAN encapsulation) นี่คือสิ่งที่ทำให้ BGW เป็น
"single point of policy/visibility" สำหรับ inter-site traffic ทั้งหมด — เหมาะสำหรับใส่ QoS/ACL/
Encryption (MACsec บน DCI link) ได้ที่จุดเดียว

### Full Config — DC-BGW-1 (DC1, Site-ID 1)

```
hostname DC-BGW-1
feature ospf
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
feature evpn

vrf context TENANT-PROD
  vni 50001
  rd auto
  address-family ipv4 unicast
    route-target both 1:50001

interface loopback0
  description RID
  ip address 1.1.1.105/32
  ip router ospf UNDERLAY-DC1 area 0.0.0.0

interface loopback1
  description PIP - NVE source-interface
  ip address 1.1.1.106/32
  ip router ospf UNDERLAY-DC1 area 0.0.0.0

interface loopback100
  description Multisite Anycast VIP (Site-ID 1) - เหมือนกับ DC-BGW-2 ทุกประการ
  ip address 1.1.1.109/32
  ip router ospf UNDERLAY-DC1 area 0.0.0.0

interface Ethernet1/1
  description Uplink to DC-SPINE-1
  no switchport
  ip address 10.84.1.1/30
  ip router ospf UNDERLAY-DC1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description Uplink to DC-SPINE-2
  no switchport
  ip address 10.84.2.1/30
  ip router ospf UNDERLAY-DC1 area 0.0.0.0
  no shutdown

interface Ethernet1/49
  description DCI to DC2-BGW-1 (Step 847: Routed IP WAN / Dark Fiber / DWDM)
  no switchport
  ip address 10.85.1.1/30
  no shutdown

router ospf UNDERLAY-DC1
  router-id 1.1.1.105

evpn multisite border-gateway 1
  delay-restore time 180
  delay-restore interface-vlan time 200

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  multisite border-gateway interface loopback100
  member vni 30130
    ingress-replication protocol bgp
  member vni 30140
    ingress-replication protocol bgp
  member vni 50001 associate-vrf

router bgp 65001
  router-id 1.1.1.105
  neighbor 1.1.1.101
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
  neighbor 1.1.1.102
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
  neighbor 10.85.1.2
    remote-as 65002
    ebgp-multihop 5
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community both
  vrf TENANT-PROD
    address-family ipv4 unicast
      advertise l2vpn evpn
```

### DC-BGW-2 (เหมือน DC-BGW-1 เปลี่ยนเฉพาะ IP ที่ต่างกัน)

```
hostname DC-BGW-2
! --- เหมือน DC-BGW-1 ทุก feature/vrf/evpn multisite border-gateway 1 ---
interface loopback0
  ip address 1.1.1.107/32
interface loopback1
  ip address 1.1.1.108/32
interface loopback100
  description Multisite Anycast VIP (Site-ID 1) - ร่วมกับ DC-BGW-1
  ip address 1.1.1.109/32       ! <-- เหมือนกับ DC-BGW-1 ทุกตัวอักษร
interface Ethernet1/1
  description Uplink to DC-SPINE-1
  ip address 10.84.3.1/30
interface Ethernet1/2
  description Uplink to DC-SPINE-2
  ip address 10.84.4.1/30
interface Ethernet1/49
  description DCI to DC2-BGW-2
  ip address 10.85.2.1/30
router bgp 65001
  neighbor 10.85.2.2 remote-as 65002    ! <-- peer กับ DC2-BGW-2 (เส้น DCI คู่ที่สอง)
```

### DC2-BGW-1 / DC2-BGW-2 (mirror ของ DC1 ฝั่ง Site-ID 2, AS 65002)

```
hostname DC2-BGW-1
! --- feature เดียวกับ DC-BGW-1 ---
vrf context TENANT-PROD
  vni 50001
  rd auto
  address-family ipv4 unicast
    route-target both 1:50001         ! <-- ต้องเหมือนกับ DC1 เพื่อ import Type-5 ข้าม AS ได้

interface loopback0
  ip address 1.1.1.115/32
  ip router ospf UNDERLAY-DC2 area 0.0.0.0
interface loopback1
  ip address 1.1.1.116/32
  ip router ospf UNDERLAY-DC2 area 0.0.0.0
interface loopback100
  description Multisite Anycast VIP (Site-ID 2) - ร่วมกับ DC2-BGW-2
  ip address 1.1.1.119/32
  ip router ospf UNDERLAY-DC2 area 0.0.0.0

interface Ethernet1/1
  description Uplink to DC2-SPINE-1
  ip address 10.84.11.1/30
  ip router ospf UNDERLAY-DC2 area 0.0.0.0
interface Ethernet1/2
  description Uplink to DC2-SPINE-2
  ip address 10.84.12.1/30
  ip router ospf UNDERLAY-DC2 area 0.0.0.0
interface Ethernet1/49
  description DCI to DC-BGW-1
  ip address 10.85.1.2/30

router ospf UNDERLAY-DC2
  router-id 1.1.1.115

evpn multisite border-gateway 2
  delay-restore time 180

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  multisite border-gateway interface loopback100
  member vni 30130
    ingress-replication protocol bgp
  member vni 30141
    ingress-replication protocol bgp
  member vni 50001 associate-vrf

router bgp 65002
  router-id 1.1.1.115
  neighbor 1.1.1.111 remote-as 65002 update-source loopback0
  neighbor 1.1.1.112 remote-as 65002 update-source loopback0
  neighbor 10.85.1.1 remote-as 65001 ebgp-multihop 5
  vrf TENANT-PROD
    address-family ipv4 unicast
      advertise l2vpn evpn
```

`DC2-BGW-2` เหมือน `DC2-BGW-1` เปลี่ยน Loopback0=1.1.1.117/32, Loopback1=1.1.1.118/32,
uplink ไป DC2-SPINE ที่ subnet 10.84.13.0/30, 10.84.14.0/30 และ DCI ไปยัง `DC-BGW-2` ที่
10.85.2.2/30 (peer `10.85.2.1 remote-as 65001`)

---

## Step 845 — L2 Stretch ข้าม Site: DC-SERVERS VNI และการจัดการ BUM Traffic

### Stretch คืออะไร และทำอย่างไร

**L2 Stretch** คือการทำให้ VLAN/Subnet เดียวกัน (เช่น `DC-SERVERS` VLAN130, `10.130.30.0/24`)
มีอยู่พร้อมกันทั้ง DC1 และ DC2 — Host/VM ที่ย้ายจาก DC1 ไป DC2 (Live Migration, DR Failover)
จะ**คง IP/MAC เดิม** และยังคุยกับ Default Gateway (Anycast Gateway `10.130.30.1`) ได้ทันที
เพราะทั้งสอง Site ประกาศ Anycast Gateway IP/MAC **เดียวกัน**

การ Stretch ทำได้ง่ายๆ เพียงกำหนด **VNI เดียวกัน (30130)** และ **Anycast Gateway MAC เดียวกัน**
บน Leaf ของทั้งสอง Site:

```
! ต้องเหมือนกันทุก Site ที่ stretch VNI เดียวกัน
fabric forwarding anycast-gateway-mac 0000.2222.3333

vlan 130
  name DC-SERVERS
  vn-segment 30130

interface Vlan130
  no shutdown
  vrf member TENANT-PROD
  ip address 10.130.30.1/24
  fabric forwarding mode anycast-gateway
  no ip redirects
```

ตั้งค่าเหมือนกันทั้งบน `DC-LEAF-1/2` (DC1, จาก Part 84) และ `DC2-LEAF-1/2` (DC2, ใหม่ใน Part 85)
— BGP EVPN Type-2 (MAC/IP) ของ Host ในแต่ละ Site จะถูก Re-originate ผ่าน BGW ให้อีก Site เห็น
โดยอัตโนมัติ (ตามกลไก Step 843)

### ปัญหา BUM Traffic ข้าม WAN — และวิธีแก้ของ Multisite

BUM = **B**roadcast, **U**nknown-unicast, **M**ulticast — ปกติต้อง flood ไปทุก VTEP ที่เป็นสมาชิก
VNI เดียวกัน ถ้า VNI ถูก stretch ข้าม Site, BUM ก็ต้อง flood ข้าม DCI ด้วย ซึ่งเสี่ยง saturate
WAN bandwidth (ตาม Step 841 ปัญหาที่ 2) Multisite มีกลไกป้องกัน 3 ชั้น:

**1) ARP Suppression (ลด BUM จากต้นตอ)**

```
interface nve1
  member vni 30130
    ingress-replication protocol bgp
    suppress-arp
```

Leaf ที่เรียนรู้ IP↔MAC ของ Host ผ่าน BGP EVPN Type-2 แล้ว จะตอบ ARP Request แทน Host ปลายทาง
เลย (Proxy ARP โดย Local VTEP) — ไม่ต้อง flood ARP Request ข้าม Site อีกต่อไปหลังจาก Host
แรก resolve เสร็จครั้งแรก

**2) Split-Horizon ที่ BGW (ป้องกัน Loop และลด replication ซ้ำซ้อน)**

กฎสำคัญของ Multisite BGW: **Traffic ที่ได้รับจาก DCI (มาจาก Site อื่น) จะไม่ถูกส่งกลับออกไปยัง
DCI อีก (ไปยัง Site ที่สาม)** — BGW ส่งต่อ BUM traffic ที่มาจาก Remote Site ไปยัง **Local Fabric
เท่านั้น** (Local Spine/Leaf) ไม่ส่งย้อนกลับออก DCI:

```
        DC2 (Remote) ──BUM──> DC-BGW-1 (Local)
                                    │
                        ┌───────────┴───────────┐
                        ▼                       ✗ (ห้าม flood กลับออก DCI)
                 DC-SPINE-1/2                 DC3 (ถ้ามี Site ที่ 3)
                        │
                        ▼
                 DC-LEAF-1/2 (Local)
```

**3) Rate-Limit BUM ที่ DCI Interface (เทียบเท่า Storm-Control ของ L2 ปกติ)**

```
class-map type control-plane match-any BUM-CLASS
  match access-group name BUM-MULTICAST-DEST

policy-map type control-plane BUM-RATE-LIMIT
  class BUM-CLASS
    police cir 50 mbps bc 1 mbyte conform transmit exceed drop

interface Ethernet1/49
  description DCI to DC2-BGW-1
  service-policy input BUM-RATE-LIMIT
```

ตั้ง Policer จำกัด BUM traffic ที่ผ่าน DCI link ไม่ให้เกิน threshold ที่กำหนด (ในตัวอย่างนี้
50 Mbps) — แม้เกิด Storm ที่ Local Site ผลกระทบต่อ WAN bandwidth ของ Traffic อื่นก็ยังจำกัดอยู่
ในขอบเขตที่ควบคุมได้ (แนวคิดเดียวกับ `storm-control` บน Access Port ที่เรียนใน Part 3 แต่ประยุกต์
ใช้ระดับ DCI/Multisite)

### หลักการออกแบบ: Stretch เท่าที่จำเป็นเท่านั้น

> **คำแนะนำ CCIE-level**: L2 Stretch มีค่าใช้จ่ายเสมอ (BUM overhead, MAC mobility complexity,
> การ debug Blackhole ที่ยากขึ้นเมื่อ Host เดียวกันมี Path 2 ทาง) — ใช้ L2 Stretch **เฉพาะ Workload
> ที่จำเป็นต้องคง L2 Adjacency จริงๆ** (เช่น VM Cluster ที่ใช้ Heartbeat แบบ L2 Multicast, Live
> Migration ที่ระบบ Hypervisor ต้องการ Subnet เดียวกัน) — สำหรับ Workload อื่นที่ไม่จำเป็น
> ให้ใช้ **L3 Stretch** แทน (Step 846) ซึ่งไม่มีปัญหา BUM ข้าม Site เลย

---

## Step 846 — L3 Stretch และ Type-5 EVPN Route: Routed Inter-Site Communication

### EVPN Route Type ทั้งหมดที่เกี่ยวข้อง (ทวนจาก Part 84 + เพิ่ม Type-5)

| Route Type | ชื่อ | ใช้ทำอะไร | ใช้ใน Multisite เพื่อ |
|---|---|---|---|
| Type-2 | MAC/IP Advertisement | โฆษณา Host MAC + IP (L2 + L3 เดี่ยว) | L2 Stretch (Step 845) |
| Type-3 | Inclusive Multicast Ethernet Tag (IMET) | สร้าง BUM Replication List ต่อ VNI | รู้ว่าใครเป็นสมาชิก VNI (BGW เป็นสมาชิกแทน Leaf ที่มองไม่เห็นจาก Site อื่น) |
| **Type-5** | **IP Prefix Route** | โฆษณา **Subnet/Prefix ทั้งก้อน** (ไม่ผูกกับ MAC ใดๆ) | **L3-only Stretch — ข้อความหลักของ Step นี้** |

### ทำไมต้องมี Type-5 แยกจาก Type-2

Type-2 โฆษณา Host ทีละตัว (ต้องมี MAC จริง) — เหมาะกับ Subnet ที่ Stretch (L2) เพราะ Host
เคลื่อนที่ได้ระหว่าง Site แต่ Subnet เดิม แต่ถ้า Subnet ของ DC1 (`10.140.30.0/24`, VLAN140)
และ Subnet ของ DC2 (`10.141.30.0/24`, VLAN141) เป็น**คนละ Subnet กันจริง** (ไม่ stretch เลย)
การจะให้ Host ใน DC1 คุยกับ Host ใน DC2 ได้ ต้องใช้ **Routing ระหว่าง VRF/Subnet** — Type-5
คือ Route ที่โฆษณา **Prefix ทั้งก้อน** (เหมือน Static Route/BGP Prefix ปกติ) ผ่าน L3 VNI
(ในที่นี้ VNI 50001, VRF `TENANT-PROD`) ไม่ต้องมี MAC ของ Host ปลายทางเลย

```
DC1: VLAN140 "DC-APP-DC1"   10.140.30.0/24   Gateway 10.140.30.1  (VNI 30140, local เท่านั้น)
DC2: VLAN141 "DC-APP-DC2"   10.141.30.0/24   Gateway 10.141.30.1  (VNI 30141, local เท่านั้น)

DC-BGW-1/2 โฆษณา Type-5: Prefix 10.140.30.0/24 → L3 VNI 50001
DC2-BGW-1/2 โฆษณา Type-5: Prefix 10.141.30.0/24 → L3 VNI 50001

ผลลัพธ์: Host ที่ 10.140.30.50 (DC1) ping ไปยัง 10.141.30.50 (DC2) ได้
โดยที่ VLAN140 และ VLAN141 ไม่ต้องมี VNI เดียวกัน ไม่ต้องมี MAC ปลายทางเลย
Packet ถูก Route (ไม่ใช่ Bridge) ข้าม Site ผ่าน Symmetric IRB บน L3 VNI
```

### Config ที่เกี่ยวข้อง (สรุปจาก Step 844 — เน้นส่วน Type-5)

```
! DC-LEAF-1/2 (DC1) — VLAN140 local เท่านั้น ไม่ stretch
vlan 140
  name DC-APP-DC1
  vn-segment 30140
interface Vlan140
  vrf member TENANT-PROD
  ip address 10.140.30.1/24
  fabric forwarding mode anycast-gateway

! DC2-LEAF-1/2 (DC2) — VLAN141 local เท่านั้น ไม่ stretch
vlan 141
  name DC-APP-DC2
  vn-segment 30141
interface Vlan141
  vrf member TENANT-PROD
  ip address 10.141.30.1/24
  fabric forwarding mode anycast-gateway

! ทั้งสอง Site: VRF TENANT-PROD ต้องมี Route-Target แบบ explicit เหมือนกัน (จาก Step 844)
vrf context TENANT-PROD
  vni 50001
  address-family ipv4 unicast
    route-target both 1:50001    ! <-- ทำให้ Type-5 ข้าม AS import กันได้
```

> **หมายเหตุ**: Type-5 ถูกโฆษณาโดย Leaf/BGW ที่มี SVI ของ Subnet นั้น (ตาม Symmetric IRB
> pattern เดียวกับที่ Part 84 สอนไว้สำหรับ Inter-VNI Routing ภายใน Site เดียว) — สิ่งที่ Multisite
> เพิ่มเข้ามาคือ BGW ทำหน้าที่ Re-originate Type-5 (พร้อม Next-Hop Rewrite เหมือน Type-2/3)
> ข้าม Site เท่านั้น

### ตารางเปรียบเทียบ L2 Stretch vs L3 Stretch — แนวทางออกแบบ

| ประเด็น | L2 Stretch (Type-2, Step 845) | L3-only / Type-5 (Step 846) |
|---|---|---|
| Subnet ระหว่าง Site | เดียวกัน | คนละ Subnet |
| BUM Traffic ข้าม DCI | มี (ต้องจัดการตาม Step 845) | **ไม่มีเลย** (Route ไม่ Bridge) |
| MAC Mobility / VM ย้าย Site คง IP เดิม | ✅ รองรับ | ❌ (ต้องเปลี่ยน IP ตาม Subnet ปลายทาง) |
| ความซับซ้อนในการ Troubleshoot | สูงกว่า (ต้องดู MAC move, ARP suppress table) | ต่ำกว่า (คล้าย Routing ปกติที่เรียนมาแล้วทั้งหลักสูตร) |
| **คำแนะนำ** | ใช้เมื่อ **จำเป็นจริงๆ** เท่านั้น (Cluster heartbeat, Live migration) | **ค่าเริ่มต้นที่ควรเลือกก่อน** สำหรับ Workload ทั่วไประหว่าง Site |

---

## Step 847 — DCI Transport Options: Dark Fiber, DWDM, Routed IP WAN

BGW ทั้งสองฝั่งต้องเชื่อมกันด้วย Transport ทางกายภาพบางอย่าง (**DCI — Data Center
Interconnect**) — ตัวเลือกหลักที่ใช้ในงานจริง:

| Transport | ลักษณะ | Latency/ระยะทาง | Bandwidth | เกี่ยวข้องกับ Part ก่อนหน้า |
|---|---|---|---|---|
| **Dark Fiber** | เช่า/เป็นเจ้าของ Fiber เปล่าเชื่อมตรงระหว่าง Site | ต่ำมาก, จำกัดด้วยระยะทางจริง (ปกติ < 80-100 กม. โดยไม่ใช้ Amplifier) | สูงมาก (จำกัดด้วย Transceiver ไม่ใช่ Fiber) | Layer 1 physical — Part 1 Step 6 |
| **DWDM** (Dense Wavelength Division Multiplexing) | ส่งหลาย "สี" (Wavelength) บน Fiber เส้นเดียว, แต่ละสีเป็น circuit อิสระ | ต่ำมาก (ใกล้เคียง Dark Fiber) รองรับระยะไกลกว่าด้วย Amplifier/Repeater | สูงมาก, ขยายได้ทีละ Wavelength | มักใช้คู่กับ Dark Fiber ระยะไกล |
| **Routed IP WAN** | ส่งผ่าน MPLS/Internet/SD-WAN ปกติ (Layer 3) | สูงกว่า 2 แบบข้างต้น, แปรผันตาม ISP/Provider | จำกัดตาม Bandwidth ที่ซื้อจาก Provider | **[Part 19 — WAN Technologies](part-019-wan-technologies.md)**, **[Part 74 — Advanced BGP WAN Edge](part-074-advanced-bgp-wan-edge.md)** |

### ทำไม Transport เลือกอะไรก็ได้ (Multisite ไม่ผูกกับ Layer 1 เฉพาะ)

เพราะ BGW คุยกันด้วย **eBGP over IP** (Layer 3) เท่านั้น (ดู Step 843/844) — ตราบใดที่ BGW ทั้งสอง
ฝั่งมี **IP Reachability** ถึงกัน (ผ่าน `/30` ตรงเชื่อม หรือผ่าน Routed WAN หลาย hop) EVPN
Multisite ก็ทำงานได้ทันที ไม่สนใจว่า Physical Layer ข้างล่างเป็น Fiber, DWDM หรือ Router ของ ISP

| สถานการณ์ | Transport ที่เหมาะสม |
|---|---|
| DC1/DC2 อยู่ในเมืองเดียวกัน/ระยะใกล้ (Metro), องค์กรมี Fiber ของตัวเอง | Dark Fiber หรือ DWDM (Latency ต่ำสุด, เหมาะกับ L2 Stretch ที่ sensitive ต่อ latency) |
| DC1/DC2 อยู่ต่างเมือง/ต่างประเทศ (ตาม DR Site จริงส่วนใหญ่) | Routed IP WAN (MPLS L3VPN — ดู Part 86-87, หรือ SD-WAN — Part 39/76) |
| ต้องการ Redundant Path 2 เส้นทางที่ Diverse กันจริง (ไม่ใช้ Conduit เดียวกัน) | ผสมกัน: เช่น DC-BGW-1 ↔ DC2-BGW-1 ผ่าน Dark Fiber, DC-BGW-2 ↔ DC2-BGW-2 ผ่าน Routed WAN คนละ Provider |

> ใน Lab Part นี้ (Step 844/850) กำหนดให้ DCI เป็น Routed IP WAN แบบ Point-to-Point (`/30`) เพื่อ
> ให้สอดคล้องกับ Lab environment ที่ไม่มี Fiber จริง — ในงานจริงส่วนที่เปลี่ยนคือ Physical/Layer 1-2
> เท่านั้น ส่วน BGP/EVPN Config ทั้งหมดที่เขียนไปแล้วใน Step 844 **ไม่ต้องเปลี่ยนเลย**

---

## Step 848 — Failure Scenario และ Site Isolation: Fault Domain แยกจริงหรือไม่

### Scenario: DC1 Underlay ล้ม (Spine-Leaf OSPF Flap)

จำลองเหตุการณ์ `DC-SPINE-1` มีปัญหา (เช่น Line Card ผิดพลาด ทำให้ OSPF adjacency กับ Leaf/BGW
flap ต่อเนื่อง):

```
DC-SPINE-1# (จำลอง fault)
DC-SPINE-1(config)# interface Ethernet1/1
DC-SPINE-1(config-if)# shutdown
DC-SPINE-1(config-if)# no shutdown
! ทำซ้ำหลายรอบเพื่อจำลอง flapping
```

**สิ่งที่เกิดขึ้นที่ DC1 (ภายใน Site เดียวกัน):**

```
DC-BGW-1# show ip ospf neighbor
 OSPF Process ID UNDERLAY-DC1 VRF default
 Neighbor ID     Pri State            Up Time  Address         Interface
 1.1.1.101         1 FULL/ -  <-- flap ↔ DOWN  10.84.1.2       Eth1/1
```

BGP EVPN Route ที่เรียนผ่าน `DC-SPINE-1` re-converge ผ่าน `DC-SPINE-2` (ECMP/Redundant Path
ที่ Part 84 ออกแบบไว้แล้ว) — Convergence Time อยู่ในระดับปกติของ OSPF/BGP ภายใน Site เดียว
(sub-second ถึงไม่กี่วินาที)

**สิ่งที่เกิดขึ้นที่ DC2 (Site อื่น) — คือหัวใจของคำถามนี้:**

```
DC2-LEAF-1# show bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.111       4 65002    5821    5819       88    0    0 45d02h   142      <- DC2-SPINE-1 ยังปกติ 100%
1.1.1.112       4 65002    5820    5818       88    0    0 45d02h   142      <- DC2-SPINE-2 ยังปกติ 100%

DC2-BGW-1# show bgp l2vpn evpn summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.115       4 65002    9012    9010      210    0    0 45d02h   200
10.85.1.1       4 65001   14203   14198      210    0    0 45d02h   142      <- eBGP ไป DC-BGW-1
                                                                                 ยัง Established
                                                                                 ปกติ (ไม่ flap)
                                                                                 เพราะปัญหาอยู่แค่
                                                                                 DC-SPINE-1 ↔ Leaf/BGW
                                                                                 ภายใน DC1 เท่านั้น
```

**ผลสรุป**: `DC2-BGW-1` ยัง Established กับ `DC-BGW-1` ปกติ (eBGP Session ไม่รับรู้เรื่อง OSPF
ภายใน DC1 เลย เพราะ BGW คุยกันด้วย EVPN routes ที่ผ่านการ Re-originate แล้วเท่านั้น ไม่ใช่ raw
Underlay state) — **BGP Table Version (TblVer) ที่ DC2 ไม่กระเพื่อม**, Host ที่ DC2 ยังคง
ping ข้าม Site ได้ตามปกติ (สมมติว่า DC1 ยังมี ECMP Path ผ่าน DC-SPINE-2 รับ traffic ต่อได้)

### เปรียบเทียบกับกรณี Single Fabric (ถ้าไม่ทำ Multisite)

| ประเด็น | Single Fabric ข้าม Site (ไม่มี BGW) | EVPN Multisite (มี BGW) |
|---|---|---|
| OSPF flap ที่ DC-SPINE-1 กระทบ DC2 หรือไม่ | ✅ กระทบ (Area/Process เดียวกันทั้ง 2 Site) | ❌ ไม่กระทบ (BGW กันไว้) |
| BGP RR (Spine) ที่ DC1 ประมวลผล Route Churn มาก กระทบ RR ที่ DC2 หรือไม่ | ✅ กระทบ (RR เดียวกัน/Full-mesh เดียวกัน) | ❌ ไม่กระทบ (RR แยกคนละ AS) |
| MAC Move Storm ที่ DC1 (VM misconfigure) กระทบ BUM ที่ DC2 หรือไม่ | ✅ กระทบเต็มที่ | ⚠️ กระทบแค่ VNI ที่ Stretch จริง (ผ่าน Rate-Limit Step 845) |

### กลไก `delay-restore` — ป้องกัน Black-hole เมื่อ BGW กลับมาทำงาน

เมื่อ `DC-BGW-1` reload เสร็จและ Interface กลับมา Up, ถ้าปล่อยให้มันเริ่มโฆษณา EVPN Route
ไปยัง DC2 ทันที (ก่อนที่ Local Fabric ของ DC1 จะ re-converge สมบูรณ์) อาจทำให้ Traffic จาก DC2
ถูกส่งเข้ามาที่ DC-BGW-1 ทั้งที่ Local Underlay ยังไม่พร้อม (**Black-hole**) คำสั่ง
`delay-restore time 180` (ที่ตั้งไว้ใน Step 844) แก้ปัญหานี้โดยให้ BGW **หน่วงเวลา 180 วินาที**
ก่อนโฆษณา Route ที่เรียนจาก Local Fabric ออกไปยัง Remote Site หลังจาก Session/Interface กลับมา
Up — ให้เวลา OSPF/BGP ภายใน Site คืนสภาพให้เสถียรก่อน

---

## Step 849 — Multisite และ Disaster Recovery Design

### จาก Multisite สู่ DR Pattern

EVPN Multisite ที่สร้างใน Part นี้ **ไม่ใช่ DR Solution เต็มรูปแบบในตัวเอง** — มันเป็นเพียง
**Network-layer enabler** ที่ทำให้ Pattern ของ DR ทำงานได้จริง Pattern ที่พบบ่อยและ Multisite
รองรับ:

| DR Pattern | ใช้ L2 Stretch หรือ L3-only | รายละเอียด |
|---|---|---|
| **Active-Standby (Cold/Warm DR)** | L3-only ส่วนใหญ่ | DC2 เป็น Standby, Workload ปกติรันที่ DC1, Failover เปลี่ยน DNS/Application config ให้ชี้ไป DC2 (คนละ Subnet ก็ได้เพราะ Failover ทำที่ Application layer) |
| **Live Migration / VM Mobility** | **L2 Stretch จำเป็น** | vMotion/Live Migration ต้องให้ VM คง IP/MAC เดิมระหว่างย้าย Host จาก DC1 ไป DC2 — Anycast Gateway (Step 845) ทำให้ Default Gateway ยังใช้ IP เดิมได้ทันทีหลังย้าย |
| **Active-Active (บาง Tier)** | ผสม: Database tier อาจ L2 stretch (Cluster heartbeat), Web/App tier ใช้ L3-only + Load Balancer/GSLB แจก Traffic 2 Site | ต้องระวัง Split-Brain ของ Database ที่ stretch (นอกเหนือ scope Part นี้ — เป็นเรื่อง Application Architecture) |

### ทำไม Anycast Gateway (Step 845) คือกลไกสำคัญที่สุดสำหรับ Workload Mobility

เมื่อ VM ย้ายจาก DC1 (หลัง `DC-LEAF-1`) ไป DC2 (หลัง `DC2-LEAF-1`) แบบ Live Migration:

1. VM ยังใช้ IP เดิม (`10.130.30.50`) และ Default Gateway เดิม (`10.130.30.1`)
2. เพราะ `DC2-LEAF-1` ประกาศ Anycast Gateway IP/MAC **เดียวกัน**กับ `DC-LEAF-1` (Step 845) —
   VM **ไม่ต้องเปลี่ยน ARP cache ของ Default Gateway เลย** (MAC ของ Gateway เหมือนเดิมทุกตัวอักษร)
3. BGP EVPN Type-2 จาก `DC2-LEAF-1` โฆษณา MAC/IP ของ VM ตัวนี้ใหม่ (MAC Move) — Remote Site
   (DC1 เดิม) ได้รับ Update ผ่าน BGW ว่า VM ย้ายไปแล้ว, MAC Table เดิมที่ DC1 ถูกถอน
4. Traffic ใหม่ที่มุ่งหา VM ตัวนี้จะถูก Route ไปยัง DC2 ทันที (ผ่าน Type-2 update ที่ re-converge)

นี่คือสิ่งที่ VXLAN/EVPN Multisite ทำได้ดีกว่า L3-only DCI แบบเดิม (เช่น GRE/IPsec tunnel ธรรมดา)
ซึ่งไม่มี Distributed Anycast Gateway ในตัว

### เชื่อมต่อไปยัง Part 92

หัวข้อ Disaster Recovery แบบครบวงจร (RPO/RTO, Backup Strategy, Runbook, Site Failover Testing,
Application-layer DR ที่ไม่ใช่แค่ Network) จะอยู่ใน
**[Part 92 — Disaster Recovery & Network Resilience](part-092-disaster-recovery-resilience.md)**
ซึ่งจะใช้ DC1/DC2 Multisite Fabric ที่สร้างไว้ใน Part นี้เป็นฐาน Infrastructure ของ DR Design
เต็มรูปแบบ (รวม Automation ของ Failover, Health-check, และ GSLB/DNS)

---

## Step 850 — Full Lab: EVPN Multisite DC1 ↔ DC2 (L2 Stretch + L3-only) พร้อม Verification

### Lab Objective

1. เชื่อม DC1 (จาก Part 84) และ DC2 (ใหม่) ผ่าน EVPN Multisite (BGW ทั้งสองฝั่ง)
2. Stretch VNI 30130 (`DC-SERVERS`, `10.130.30.0/24`) ข้ามทั้งสอง Site (L2 Stretch)
3. ให้ VLAN140 (DC1-only) และ VLAN141 (DC2-only) คุยกันได้ผ่าน Type-5 (L3-only, VRF `TENANT-PROD`)
4. Verify ด้วย `show bgp l2vpn evpn`, `show nve peers`, `show nve vni`
5. ทดสอบ Workload Mobility จำลอง (ping ต่อเนื่องระหว่างการ "ย้าย" Host จาก DC1-side ไป DC2-side)

### Topology สรุปสุดท้าย (Recap)

```
CORE-SW1/2 ─ DC-LEAF-1/2 ─ DC-SPINE-1/2 ─ DC-BGW-1/2 ══ DCI ══ DC2-BGW-1/2 ─ DC2-SPINE-1/2 ─ DC2-LEAF-1/2
              VNI30130,140                AS65001                            AS65002        VNI30130,141
              (จาก Part84)                Site-ID 1                          Site-ID 2      (ใหม่)
```

### สรุป Config ทั้งหมด (Checklist)

| Device | สิ่งที่ต้อง Config ใหม่ใน Part 85 |
|---|---|
| DC-LEAF-1, DC-LEAF-2 | เพิ่ม VLAN140/VNI30140 + SVI140 เข้า VRF `TENANT-PROD` (VNI30130 มีอยู่แล้วจาก Part 84 ไม่ต้องแก้) |
| DC-BGW-1, DC-BGW-2 | ใหม่ทั้งหมด — Loopback0/1/100, OSPF UNDERLAY-DC1, `evpn multisite border-gateway 1`, NVE (member vni 30130, 30140, 50001 associate-vrf), eBGP ไปยัง DC2-BGW |
| DC2-SPINE-1, DC2-SPINE-2 | ใหม่ทั้งหมด — Underlay OSPF UNDERLAY-DC2, iBGP Route-Reflector AS65002 |
| DC2-LEAF-1, DC2-LEAF-2 | ใหม่ทั้งหมด — VLAN130 (stretch, anycast-gateway-mac **เหมือน DC1**), VLAN141 (local), VRF `TENANT-PROD`, NVE |
| DC2-BGW-1, DC2-BGW-2 | ใหม่ทั้งหมด — เหมือน DC-BGW แต่ Site-ID 2, AS65002 |

### Verification 1 — BGP EVPN Table ที่ DC2-LEAF-1 (เห็น Route จาก DC1 ผ่าน BGW)

```
DC2-LEAF-1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 355, Local Router ID is 1.1.1.113

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1.1.1.103:32867    (RD ของ DC-LEAF-1 เดิมที่ DC1 — ยังเห็นได้ใน field RD
                                          แต่ Next-Hop ถูก Rewrite แล้ว)
*>i[2]:[0]:[0]:[48]:[0050.5601.0001]:[32]:[10.130.30.50]/272
                       1.1.1.109           100          0        i     <-- Next-Hop = DC-BGW VIP
                                                                             (DC1) ไม่ใช่ DC-LEAF-1 จริง!
Route Distinguisher: 1.1.1.115:32867    (RD ของ DC2-BGW-1 เอง — Local Route)
*>l[5]:[0]:[0]:[24]:[10.141.30.0]/224
                       10.85.1.2           100     32768        i     <-- Type-5 จาก DC2 เอง

Route Distinguisher: 1.1.1.109:32867    (RD ที่ DC-BGW ใช้ re-originate จาก DC1)
*>i[5]:[0]:[0]:[24]:[10.140.30.0]/224
                       1.1.1.109           100          0        i     <-- Type-5 จาก DC1 (VLAN140)
                                                                             Next-Hop = DC1 BGW VIP
```

สังเกตว่า Next-Hop ของทุก Route ที่มาจาก DC1 (ทั้ง Type-2 ของ `10.130.30.50` และ Type-5 ของ
`10.140.30.0/24`) เป็น `1.1.1.109` (**DC-BGW Anycast VIP**) เสมอ — ไม่มี Route ใดชี้ไปยัง
`1.1.1.103` (`DC-LEAF-1` จริง) เลย ยืนยันกลไก Next-Hop Rewrite จาก Step 843

### Verification 2 — NVE Peers ที่ DC-BGW-1 (เห็น Peer ทั้ง Local Fabric และ Remote Site)

```
DC-BGW-1# show nve peers
Interface  Peer-IP          State  LearnType  Uptime    Router-Mac
nve1       1.1.1.103        Up     CP         45d02h    n/a           <- DC-LEAF-1 (Local, DC1)
nve1       1.1.1.104        Up     CP         45d02h    n/a           <- DC-LEAF-2 (Local, DC1)
nve1       1.1.1.119        Up     CP         02:14:33  n/a           <- DC2-BGW VIP (Remote Site!)

DC-BGW-1# show nve vni
Codes: CP - Control Plane        DP - Data Plane
       UC - Unconfigured         SA - Suppress ARP
Interface VNI      Multicast-group   State Mode Type [BD/VRF]     Flags
nve1      30130    n/a (IR)          Up    CP   L2   [130]        SA
nve1      30140    n/a (IR)          Up    CP   L2   [140]        SA
nve1      50001    n/a               Up    CP   L3   [TENANT-PROD]
```

`Peer-IP 1.1.1.119` (Anycast VIP ของ DC2-BGW) ปรากฏเป็น NVE Peer ตัวหนึ่งของ `DC-BGW-1` —
ยืนยันว่า VXLAN Tunnel ระหว่าง Site วิ่งจาก BGW ถึง BGW โดยตรง (Step 844) ไม่ใช่ Leaf-to-Leaf

### Verification 3 — L2 Stretch Test (Workload Mobility จำลอง)

**ก่อนย้าย** — Host `10.130.30.50` อยู่หลัง `DC-LEAF-1` (DC1), ping จาก Host ทดสอบที่ DC2
(`10.130.30.99` หลัง `DC2-LEAF-1`, subnet เดียวกันเพราะ stretch):

```
DC2-Test-Host# ping 10.130.30.50 repeat 100
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 1/2/4 ms
```

**จำลองการย้าย** (Live Migration): ปิด Host ที่ DC1, เปิด Host เดียวกัน (IP/MAC เดิม) ที่ port
ของ `DC2-LEAF-2` แทน — สังเกตผลใน `show bgp l2vpn evpn` ที่ `DC-BGW-1` (Remote-site จากมุมของ
DC1):

```
DC-BGW-1# show bgp l2vpn evpn | include 10.130.30.50
*>i[2]:[0]:[0]:[48]:[0050.5601.0001]:[32]:[10.130.30.50]/272
                       1.1.1.119           100          0        i     <-- Next-Hop เปลี่ยนเป็น
                                                                             DC2-BGW VIP แล้ว!
                                                                             (MAC Move ข้าม Site
                                                                             ถูกตรวจจับอัตโนมัติ)
```

**ระหว่างย้าย** — ping ต่อเนื่องจาก Host ทดสอบที่ DC1 (`10.130.30.20`, subnet เดียวกัน) ไปยัง
`10.130.30.50` ระหว่างช่วงเวลาที่ Host ย้าย Site:

```
DC1-Test-Host# ping 10.130.30.50 repeat 200
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!X.!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 99 percent (198/200), round-trip min/avg/max = 1/3/9 ms
! หลุด 2 packet ช่วง MAC Move re-converge (Type-2 withdraw จาก DC1 + advertise ใหม่จาก DC2)
! Default Gateway (10.130.30.1) เดิมทำงานต่อได้ทันที เพราะ Anycast Gateway MAC เหมือนกันทั้ง 2 Site
```

การหลุดแค่ 1-2 packet ช่วง MAC Move ถือว่าเป็นพฤติกรรมปกติของ EVPN MAC Mobility (ไม่ใช่ Bug) —
เทียบกับ DCI แบบเดิม (GRE/L2TPv3 ธรรมดาไม่มี Control-Plane MAC Learning) ที่มักต้องรอ MAC
Aging Timeout (นาทีถึงหลายนาที) กว่าจะ Re-converge นี่คือประโยชน์ของ BGP EVPN Control-Plane
Learning ที่ต่างจาก Data-Plane Learning แบบเดิม

### Verification 4 — L3-only Test (VLAN140 DC1 ↔ VLAN141 DC2 ผ่าน Type-5)

```
DC1-App-Host# ping 10.141.30.10 source 10.140.30.50
!!!!!!!!!!
Success rate is 100 percent (10/10), round-trip min/avg/max = 2/4/7 ms

DC1-App-Host# traceroute 10.141.30.10 source 10.140.30.50
  1  10.140.30.1   1 msec   <- DC-LEAF-1 (Anycast Gateway, Symmetric IRB เข้า VRF)
  2  10.141.30.10  4 msec   <- ปลายทางที่ DC2 โดยตรง (Route ผ่าน L3 VNI/Type-5,
                                 ไม่เห็น Hop ของ BGW เพราะ VXLAN Routing เป็น
                                 Underlay hop เดียวในมุมของ IP TTL)
```

Traceroute แสดงเพียง 2 hop (Local Gateway → ปลายทาง) แม้ว่าจริงๆ packet วิ่งผ่าน `DC-LEAF-1 →
DC-BGW-1/2 → DCI → DC2-BGW-1/2 → DC2-LEAF-1/2` เพราะ VXLAN Encapsulation ทำให้ Underlay Hop
ทั้งหมดโปร่งใสต่อ IP TTL ของ Original Packet (VXLAN ไม่ decrement TTL ของ inner packet ต่อ
underlay hop) — พฤติกรรมนี้เหมือนกับที่อธิบายไว้ตอน L2/L3 VNI พื้นฐานใน Part 84

---

## แบบฝึกหัดทวนความเข้าใจ Part 85

1. Border Gateway (BGW) ต่างจาก Border Leaf อย่างไร และทำไม Part นี้เลือกใช้อุปกรณ์แยกกัน?
2. อธิบายกลไก "Next-Hop Rewrite" ที่ BGW ทำ และทำไมมันคือสิ่งที่ทำให้ Underlay ของแต่ละ Site
   ไม่ถูก expose ให้ Site อื่นรู้จัก?
3. ทำไม Route-Target แบบ `auto` ถึงใช้ไม่ได้กับ VRF/VNI ที่ต้อง Import ข้าม Site ที่มี AS ต่างกัน
   และแก้ปัญหานี้อย่างไรใน Part นี้?
4. L2 Stretch กับ L3-only (Type-5) ต่างกันอย่างไร และควรเลือกใช้แบบไหนเมื่อไร?
5. `delay-restore time` มีไว้แก้ปัญหาอะไร และเกิดอะไรขึ้นถ้าไม่มีการตั้งค่านี้เมื่อ BGW reload กลับมา?

**เฉลย:**

1. Border Leaf ทำ L3 handoff ออกไปยัง Campus/Core (เช่น `DC-LEAF-1/2` ↔ `CORE-SW1/2`) ส่วน
   Border Gateway ทำหน้าที่เชื่อม Site ต่าง DC เข้าด้วยกันผ่าน Multisite (`DC-BGW-1/2` ↔ DCI ↔
   `DC2-BGW-1/2`) — Part นี้แยกอุปกรณ์เพื่อความชัดเจนของ Fault Domain ตาม CCIE Design Best
   Practice แม้จะรวมไว้ในอุปกรณ์เดียวกันได้ในทางปฏิบัติ
2. BGW จะเปลี่ยน Next-Hop ของทุก EVPN Route ที่ Local Fabric เรียนรู้มา (ที่เดิมชี้ไปยัง Loopback
   ของ Leaf จริง) ให้กลายเป็น Anycast VIP Loopback ของตัวเอง (Loopback100) ก่อนส่งออกไปยัง Site
   อื่น — ทำให้ Site อื่นเห็นแค่ BGW VIP เท่านั้น ไม่ต้องมี Underlay Reachability ไปยัง Leaf/Spine
   จริงของ Site ต้นทางเลย
3. RT แบบ `auto` embed ค่า Local-AS ไว้ในตัว (เช่น `65001:VNI`) เมื่อ DC1(AS65001) และ
   DC2(AS65002) ใช้ auto คนละ AS ค่า RT ที่ generate จะไม่ตรงกัน ทำให้ Import ไม่สำเร็จ — Part นี้
   แก้โดยกำหนด RT แบบ explicit ที่เหมือนกันทั้งสอง Site (เช่น `route-target both 1:50001`)
4. L2 Stretch = Subnet เดียวกันทั้ง 2 Site ผ่าน Type-2, รองรับ MAC Mobility/Live Migration แต่มี
   BUM overhead ข้าม DCI; L3-only = คนละ Subnet ผ่าน Type-5, ไม่มี BUM ข้าม Site เลยแต่ Host
   ย้าย Site ต้องเปลี่ยน IP — ควรเลือก L3-only เป็นค่าเริ่มต้น และใช้ L2 Stretch เฉพาะเมื่อ
   Workload ต้องการ L2 Adjacency จริงๆ เท่านั้น
5. `delay-restore time` ป้องกัน Black-hole โดยหน่วงเวลาก่อนที่ BGW ที่กลับมา Up จะเริ่มโฆษณา
   Route ของ Local Fabric ไปยัง Remote Site จนกว่า Local Underlay/EVPN จะ re-converge สมบูรณ์ —
   ถ้าไม่มีการตั้งนี้ Traffic จาก Remote Site อาจถูกดึงเข้ามาที่ BGW ทั้งที่ Local Fabric ยังไม่พร้อม
   รับ ทำให้ Packet loss ยาวนานกว่าที่ควร

---

## สรุป Part 85

Part นี้ต่อยอด VXLAN/EVPN Fabric เดี่ยวจาก Part 84 ให้กลายเป็นสถาปัตยกรรมระดับ CCIE ที่รองรับ
หลาย Data Center ผ่าน **EVPN Multisite**: สร้าง Site ที่สอง (**DC2**) พร้อม Fabric อิสระของตัวเอง
(`DC2-SPINE-1/2`, `DC2-LEAF-1/2`), เพิ่ม **Border Gateway (BGW)** คู่ในแต่ละ Site
(`DC-BGW-1/2`, `DC2-BGW-1/2`) ที่ทำ **eBGP EVPN** ข้าม Site (ผูกกับทักษะ Part 29) พร้อมกลไก
**Next-Hop Rewrite** ที่แยก Fault Domain ของ Underlay แต่ละ Site ออกจากกันจริง เราสาธิตทั้ง
**L2 Stretch** (VNI 30130 `DC-SERVERS` ข้าม Site พร้อมการจัดการ BUM/ARP-Suppression/Split-Horizon)
และ **L3-only ผ่าน Type-5** (VLAN140/141 คนละ Subnet route กันผ่าน VRF `TENANT-PROD`) พร้อม
เปรียบเทียบ DCI Transport (Dark Fiber/DWDM/Routed WAN — ผูกกับ Part 19/74) และทดสอบ Failure
Scenario ที่พิสูจน์ว่า DC2 ไม่ได้รับผลกระทบเมื่อ DC1 มีปัญหา Underlay ✅

**พร้อมสำหรับ Part 86**: เราจะเปลี่ยนโดเมนจาก Data Center Fabric ไปสู่ **MPLS L3VPN** —
เทคโนโลยี WAN ระดับ Provider/Enterprise ที่ใช้ VRF + MP-BGP (คล้ายแนวคิด Type-5/VRF ที่เพิ่งเรียน
ใน Part นี้) เพื่อแยก Routing Table ของลูกค้าหลายรายบน Infrastructure เดียวกัน — รวม PE-CE Routing,
Route-Distinguisher, Route-Target, และ Label Distribution

**ไปต่อ:** [Part 86 — MPLS L3VPN Fundamentals →](part-086-mpls-l3vpn-fundamentals.md)
