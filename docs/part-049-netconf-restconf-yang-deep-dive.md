# Part 49 — APIs, YANG, NETCONF/RESTCONF Deep Dive
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 481–490 จาก 1000**

> ต่อจาก [Part 48 — Ansible Advanced & Network CI/CD](part-048-automation-ansible-advanced.md)
> ที่เราขยาย Ansible ไปสู่ Role, Collection ขั้นสูง, และ Pipeline สำหรับ Network-as-Code แล้ว
> Part นี้จะพาเรากลับไปเจาะลึกสิ่งที่ [Part 23 Step 229](part-023-automation-basics.md)
> ได้ "แอบดู" ไว้เพียงสั้นๆ — **Model-Driven Programmability** เต็มรูปแบบ: YANG Data Model,
> NETCONF Protocol (RFC 6241), RESTCONF Protocol (RFC 8040), และการเขียน Python ควบคุมทั้งสองแบบ
> ด้วย `ncclient` และ `requests` เราจะปิดท้ายด้วย Lab เปรียบเทียบทำงานเดียวกันสามวิธี — CLI/Netmiko,
> NETCONF, RESTCONF — เพื่อให้เห็นภาพชัดว่าเบื้องหลังทั้งสาม Protocol คือ config ชุดเดียวกันบน
> อุปกรณ์ Cisco IOS-XE

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 481 | Model-Driven Programmability พื้นฐาน — ทำไมต้อง YANG Data Model |
| 482 | ประเภทของ YANG Model — Native, OpenConfig, IETF Standard |
| 483 | NETCONF Protocol เจาะลึก — SSH Transport, RPC Operations, ตัวอย่าง XML จริง |
| 484 | NETCONF Datastore — running/candidate/startup และ Two-Phase Commit |
| 485 | RESTCONF Protocol เจาะลึก — HTTP Method, URI Structure, ตัวอย่าง `curl` จริง |
| 486 | Python + NETCONF ด้วย `ncclient` |
| 487 | Python + RESTCONF ด้วย `requests` |
| 488 | เครื่องมือสำรวจ YANG Model — `pyang`, `yanglint`, YANG Suite, YANG Library |
| 489 | แอบดู Model-Driven Telemetry (เจาะลึกใน Part 80) |
| 490 | Lab เต็มรูปแบบ: สร้าง Loopback Interface 3 วิธี — CLI/Netmiko vs NETCONF vs RESTCONF |

---

## Step 481 — Model-Driven Programmability พื้นฐาน

### ทบทวนปัญหาจาก Part 23: CLI Screen-Scraping

[Part 23 Step 229](part-023-automation-basics.md) ได้ชี้ปัญหาของ Netmiko/`ios_command` ไว้ว่า
เป็นการ **Screen-Scraping** — โปรแกรมส่งคำสั่ง CLI ผ่าน SSH แล้วต้อง parse text output ที่ถูก
ออกแบบมาให้ "คนอ่าน" ปัญหาหลักมี 3 ข้อ:

| ปัญหาของ CLI Screen-Scraping | รายละเอียด |
|---|---|
| **ไม่มี Schema ตายตัว** | `show interface` output อาจมีฟิลด์ต่างกันระหว่าง IOS 15.x และ IOS-XE 17.x |
| **Fragile ต่อการเปลี่ยน Version** | Regex ที่เขียนไว้อาจพังทันทีถ้า Cisco เปลี่ยน format คำเดียว |
| **ไม่มีการยืนยัน (Validation) ก่อน Apply** | ส่งคำสั่งผิด syntax ไปแล้วค่อยรู้ตอน error กลับมา ไม่มีการตรวจสอบ Schema ล่วงหน้า |

**Model-Driven Programmability** แก้ปัญหานี้ด้วยแนวคิดที่ต่างไปโดยสิ้นเชิง: กำหนด **Data Model**
(Schema) ที่ตายตัวไว้ล่วงหน้าด้วยภาษา **YANG** แล้วให้ NETCONF/RESTCONF ส่งข้อมูลตาม Schema นั้น
เท่านั้น — Client และ Server (อุปกรณ์ Cisco) ทั้งคู่ **รู้ Schema เดียวกัน** ก่อนคุยกัน ทำให้:

- Client สร้าง Request ตรงตาม Structure ที่ Server คาดหวังได้ (ไม่ต้องเดา)
- Server ตรวจสอบ (Validate) ข้อมูลที่รับมาตรงตาม Schema ก่อน Apply จริง
- Tooling (เช่น `pyang`) ตรวจสอบ Syntax ของ Config **ก่อน** ส่งไปที่อุปกรณ์ได้เลย (Step 488)

```
CLI Screen-Scraping (Part 23)                Model-Driven API (Part 49 - Part นี้)
┌─────────────────────────┐                  ┌─────────────────────────┐
│ Script → SSH → CLI text  │                  │ Script → NETCONF/RESTCONF │
│ → Regex parse (เดา format)│                  │ → XML/JSON ตาม YANG Schema │
│ → เปราะบางต่อ IOS version │                  │ → Schema ตายตัว ตรวจสอบได้ │
└─────────────────────────┘                  └─────────────────────────┘
```

### YANG คืออะไร

**YANG (Yet Another Next Generation, RFC 7950)** คือภาษาสร้าง Data Model สำหรับอธิบายโครงสร้าง
ของ Configuration และ State Data ของอุปกรณ์เครือข่าย — YANG **ไม่ใช่ Protocol** (ไม่ได้ส่งข้อมูล
เอง) แต่เป็น "แบบพิมพ์เขียว (Blueprint)" ที่ NETCONF/RESTCONF ใช้อ้างอิงเวลาส่งข้อมูลจริงเป็น
XML (NETCONF) หรือ JSON (RESTCONF)

### องค์ประกอบพื้นฐานของ YANG Module

| องค์ประกอบ | ความหมาย | เทียบเท่ากับ (แนวคิดโปรแกรม) |
|---|---|---|
| `module` | ไฟล์ YANG หนึ่งไฟล์ที่รวม Data Model ทั้งหมดของหัวข้อหนึ่ง | Class/Namespace |
| `container` | กลุ่มข้อมูลที่จัดกลุ่มกัน (ไม่มีค่าตัวเอง เป็นแค่ "กล่อง") | Object/Struct |
| `leaf` | ค่าข้อมูลเดี่ยว (Scalar Value) หนึ่งค่า | Field/Attribute |
| `list` | รายการของ record ที่ซ้ำกันได้ แต่ละ record มี `key` เฉพาะ | Array of Objects (มี Primary Key) |
| `leaf-list` | รายการของค่าเดี่ยวๆ (ไม่มี key, เป็นค่า scalar ซ้ำกันได้) | Array of Scalars |

### ตัวอย่าง YANG Module แบบง่าย (ย่อจากแนวคิดของ `ietf-interfaces`)

```yang
module example-interfaces {
  namespace "urn:example:interfaces";
  prefix "eth-if";

  container interfaces {
    description "Container หลักเก็บ interface ทั้งหมด";

    list interface {
      key "name";                       // "name" คือ key ของ list นี้ — ต้อง unique
      description "List ของ interface หนึ่งตัว ต่อหนึ่ง entry";

      leaf name {
        type string;
        description "ชื่อ interface เช่น GigabitEthernet0/1";
      }

      leaf description {
        type string;
        description "คำอธิบาย interface (เทียบเท่าคำสั่ง IOS 'description')";
      }

      leaf enabled {
        type boolean;
        default "true";
        description "true = no shutdown, false = shutdown";
      }

      leaf-list ip-address {
        type string;
        description "รายการ IP Address บน interface นี้ (secondary ได้หลายค่า)";
      }
    }
  }
}
```

อ่าน Module นี้แบบเทียบกับ Config IOS จริง:

```
interface GigabitEthernet0/1
 description ** Uplink to Core **     <- leaf "description"
 no shutdown                          <- leaf "enabled" = true
 ip address 10.10.99.10 255.255.255.0 <- leaf-list "ip-address"
```

`interfaces` คือ `container`, แต่ละ interface คือ 1 entry ใน `list interface` ที่ key คือ `name` —
โครงสร้างนี้คือรากฐานของทั้ง NETCONF และ RESTCONF ที่จะเห็นตลอด Part นี้

---

## Step 482 — ประเภทของ YANG Model: Native, OpenConfig, IETF

ในโลกจริง YANG Model ที่ใช้กับอุปกรณ์ Cisco IOS-XE แบ่งเป็น 3 ประเภทหลัก โดยทุกประเภท
"อธิบาย config ตัวเดียวกัน" แต่ใช้โครงสร้าง (Path/Namespace) ต่างกัน

| ประเภท | ตัวอย่างชื่อ Module | ดูแลโดย | จุดเด่น | เมื่อไหร่ควรใช้ |
|---|---|---|---|---|
| **Native** | `Cisco-IOS-XE-native` | Cisco เท่านั้น | ครอบคลุม **ทุก Feature** ของ IOS-XE (รวม CLI command ใหม่ล่าสุด) | ต้องการ Config เฉพาะของ Cisco ที่ Model อื่นยังไม่มี (feature ใหม่, platform-specific) |
| **OpenConfig** | `openconfig-interfaces`, `openconfig-bgp` | กลุ่ม Operator/Vendor (Google, AT&T, Cisco ร่วมมือ) | Vendor-neutral — Config เดียวกันใช้ได้กับ Juniper/Arista/Nokia ด้วย | ต้องการ Script ที่ทำงานข้าม Vendor ได้ (Multi-vendor Automation) |
| **IETF Standard** | `ietf-interfaces`, `ietf-routing` | IETF (มาตรฐาน RFC) | มาตรฐานสากล เก่าแก่ที่สุด ครอบคลุมเฉพาะ Feature พื้นฐานร่วมของทุก Vendor | Use Case พื้นฐานที่ทุก Vendor รองรับแน่นอน (interface, VLAN พื้นฐาน) |

### เปรียบเทียบ Path ของ "interface description" เดียวกัน ในแต่ละ Model

**1) Native Model (`Cisco-IOS-XE-native`)** — Path ตรงตามโครงสร้าง CLI ของ Cisco เป๊ะ:

```
/native/interface/GigabitEthernet[name='0/1']/description
```

```xml
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
  <interface>
    <GigabitEthernet>
      <name>0/1</name>
      <description>** Uplink to Core (Native Model) **</description>
    </GigabitEthernet>
  </interface>
</native>
```

**2) IETF Standard Model (`ietf-interfaces`)** — Path เป็น List แบบ Vendor-neutral:

```
/ietf-interfaces:interfaces/interface[name='GigabitEthernet0/1']/description
```

```xml
<interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
  <interface>
    <name>GigabitEthernet0/1</name>
    <description>** Uplink to Core (IETF Model) **</description>
  </interface>
</interfaces>
```

**3) OpenConfig Model (`openconfig-interfaces`)** — โครงสร้างคล้าย IETF แต่ path ลึกกว่า
(แยก `config` container ออกจาก `state` container อย่างชัดเจน — เป็นแนวคิดสำคัญของ OpenConfig):

```
/openconfig-interfaces:interfaces/interface[name='GigabitEthernet0/1']/config/description
```

```xml
<interfaces xmlns="http://openconfig.net/yang/interfaces">
  <interface>
    <name>GigabitEthernet0/1</name>
    <config>
      <description>** Uplink to Core (OpenConfig Model) **</description>
    </config>
  </interface>
</interfaces>
```

> **สังเกตแนวคิด OpenConfig ที่สำคัญ**: แยก `config` (ค่าที่ต้องการ set) ออกจาก `state`
> (ค่าจริงที่อุปกรณ์รายงานกลับ, read-only) อย่างชัดเจนเป็น container แยกกัน — ต่างจาก Native/IETF
> ที่มักใช้ Path เดียวกันสำหรับทั้ง config และ operational state (แยกด้วย datastore
> `running` vs `interfaces-state` ที่จะเห็นใน Step 484 แทน)

### จะรู้ได้อย่างไรว่าอุปกรณ์ Cisco IOS-XE ตัวหนึ่งรองรับ Model ไหนบ้าง

ใช้คำสั่ง `show yang-interfaces` (ดู Step 488 สำหรับรายละเอียดเต็ม) หรือ query ผ่าน RESTCONF
ที่ `/restconf/data/ietf-yang-library:yang-library` เพื่อขอรายชื่อ Module ทั้งหมดที่อุปกรณ์
โหลดไว้จริง — Cisco Catalyst 9000 Series บน IOS-XE 17.x รองรับทั้ง 3 ประเภทพร้อมกัน

### ตารางสรุปเลือก Model ใช้งาน

| สถานการณ์ | Model ที่แนะนำ |
|---|---|
| Config feature เฉพาะของ Cisco ที่ยังไม่มีใน OpenConfig/IETF | Native |
| Automation ข้าม Vendor (มี Juniper/Arista ปนอยู่) | OpenConfig |
| ต้องการความเข้ากันได้สูงสุดกับมาตรฐานเก่า/เครื่องมือ opensource ทั่วไป | IETF |
| Telemetry แบบ Streaming ระดับ Production (Part 80) | OpenConfig (นิยมมากที่สุดในอุตสาหกรรม) |

---

## Step 483 — NETCONF Protocol เจาะลึก

### Transport Layer: SSH Subsystem บน Port 830

NETCONF (RFC 6241) **ไม่ใช่** SSH CLI ธรรมดา — มันรันเป็น **SSH Subsystem** ชื่อ `netconf`
บน **TCP Port 830** (ต่างจาก SSH CLI ปกติที่ port 22) เมื่อ Client เชื่อมต่อ ทั้งสองฝั่งจะ
แลกเปลี่ยนข้อความ `<hello>` เพื่อประกาศ **Capabilities** (Feature/Model ที่แต่ละฝั่งรองรับ) ก่อน
เริ่มคุยกันจริง

### เปิดใช้งาน NETCONF บน Cisco IOS-XE (CORE-SW1)

```
CORE-SW1(config)# netconf-yang
CORE-SW1(config)# username apiuser privilege 15 secret ApiP@ssw0rd!
CORE-SW1(config)# end
CORE-SW1# show netconf-yang status
netconf-yang admin state: enabled
netconf-yang oper state: up
    ssh cipher: [ ecdh-sha2-nistp256, ... ]
```

### RPC (Remote Procedure Call) หลักที่ต้องรู้

| RPC Operation | ความหมาย | เทียบเท่า HTTP Verb ของ RESTCONF |
|---|---|---|
| `<get>` | ดึง Config + Operational State (read-only) | `GET` (ทั้ง config และ state) |
| `<get-config>` | ดึงเฉพาะ Configuration Data จาก Datastore ที่ระบุ | `GET` (เฉพาะ config datastore) |
| `<edit-config>` | แก้ไข Config ใน Datastore ที่ระบุ (merge/replace/create/delete) | `POST`/`PUT`/`PATCH`/`DELETE` |
| `<commit>` | ยืนยัน Config จาก `candidate` datastore เข้า `running` จริง (Step 484) | ไม่มีเทียบเท่าตรง — RESTCONF apply ทันที |
| `<lock>` / `<unlock>` | ล็อก Datastore ป้องกัน Client อื่นแก้พร้อมกัน | ไม่มีเทียบเท่า (RESTCONF ไม่มี lock) |
| `<close-session>` | ปิด NETCONF Session | ไม่มีเทียบเท่า (RESTCONF เป็น Stateless) |

### ตัวอย่างจริงแบบ Manual: เชื่อมต่อ NETCONF ด้วย SSH ตรงๆ (ไม่ผ่าน library)

```bash
# เชื่อมต่อ SSH เข้า NETCONF subsystem โดยตรง เพื่อดู Raw XML (สำหรับเรียนรู้/debug เท่านั้น)
ssh apiuser@10.10.99.10 -p 830 -s netconf
```

**ขั้นตอนที่ 1 — เมื่อเชื่อมต่อสำเร็จ Server (CORE-SW1) จะส่ง `<hello>` มาก่อนทันที**
(ตัดให้สั้นลง เอาแค่ capability สำคัญ — ของจริงมีเป็นร้อยบรรทัด):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
    <capability>http://cisco.com/ns/yang/Cisco-IOS-XE-native?module=Cisco-IOS-XE-native&amp;revision=2022-01-01</capability>
    <capability>urn:ietf:params:xml:ns:yang:ietf-interfaces?module=ietf-interfaces&amp;revision=2018-02-20</capability>
  </capabilities>
  <session-id>42</session-id>
</hello>
]]>]]>
```

> `]]>]]>` คือ **Message Separator** ของ NETCONF 1.0 — ทุก XML message (นอกจาก framing แบบ
> chunked ของ 1.1) จะปิดท้ายด้วย 6 ตัวอักษรนี้เสมอ เพื่อให้ Client รู้ว่า message หนึ่งจบตรงไหน

**ขั้นตอนที่ 2 — Client ต้องตอบ `<hello>` กลับไปประกาศ Capability ของตัวเอง**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
  </capabilities>
</hello>
]]>]]>
```

**ขั้นตอนที่ 3 — Client ส่ง `<get-config>` เพื่อดึง Config ของ Interface Vlan99
(ใช้ `ietf-interfaces` model, filter เฉพาะ interface ที่ต้องการ)**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>
    </source>
    <filter type="subtree">
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>Vlan99</name>
        </interface>
      </interfaces>
    </filter>
  </get-config>
</rpc>
]]>]]>
```

**ขั้นตอนที่ 4 — CORE-SW1 ตอบ Config กลับมาตรง Schema ของ `ietf-interfaces`**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
      <interface>
        <name>Vlan99</name>
        <description>Management VLAN</description>
        <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:l3ipvlan</type>
        <enabled>true</enabled>
        <ietf-ip:ipv4 xmlns:ietf-ip="urn:ietf:params:xml:ns:yang:ietf-ip">
          <ietf-ip:address>
            <ietf-ip:ip>10.10.99.10</ietf-ip:ip>
            <ietf-ip:netmask>255.255.255.0</ietf-ip:netmask>
          </ietf-ip:address>
        </ietf-ip:ipv4>
      </interface>
    </interfaces>
  </data>
</rpc-reply>
]]>]]>
```

สังเกตว่า Response เป็น **XML ที่มี Schema ตายตัว** ตรงตาม YANG Module `ietf-interfaces`
ทุกฟิลด์ (`name`, `description`, `type`, `enabled`, `ipv4/address`) — ไม่ต้องเดา format
เหมือนการ parse `show running-config interface Vlan99` ใน Netmiko แบบ Part 23 เลย

---

## Step 484 — NETCONF Datastore: running/candidate/startup และ Two-Phase Commit

### สาม Datastore หลักที่ต้องเข้าใจ

| Datastore | ความหมาย | เทียบเท่าบน IOS CLI |
|---|---|---|
| `running` | Config ที่ **กำลังทำงานจริง** บนอุปกรณ์ ณ ขณะนี้ | `show running-config` |
| `candidate` | Config **ร่างที่ยังไม่ Apply จริง** — แก้ไขได้อิสระ ยังไม่กระทบอุปกรณ์จนกว่าจะ `commit` | ไม่มีเทียบเท่าตรงบน CLI ทั่วไป (คล้าย config ที่พิมพ์ในโหมด config แต่ยังไม่ `end`) |
| `startup` | Config ที่จะโหลดตอนอุปกรณ์ Boot ครั้งถัดไป | `show startup-config` / `copy run start` |

### สองแนวทางแก้ Config ผ่าน NETCONF

**แนวทางที่ 1 — แก้ `running` ตรงๆ (Direct Edit)**: เหมือนพิมพ์คำสั่งใน CLI ปกติ — Apply ทันที
ไม่มีขั้นตอนตรวจสอบก่อน เสี่ยงถ้า Config ผิดพลาดจะกระทบอุปกรณ์ทันที (คล้าย `configure terminal`
แบบธรรมดาที่เคยใช้ตลอด Part 1-48)

**แนวทางที่ 2 — แก้ `candidate` ก่อน แล้ว `validate` + `commit` (Two-Phase Commit)**: ปลอดภัยกว่า
มาก — เปรียบเสมือนแก้ Config ใน "sandbox" ก่อน ตรวจสอบว่าไม่มี syntax/dependency error แล้วค่อย
สั่ง `commit` ให้มีผลจริงกับ `running` ในขั้นตอนเดียว (Atomic — ถ้า commit สำเร็จคือสำเร็จทั้งหมด
ถ้าล้มเหลวคือไม่มีผลอะไรเปลี่ยนเลย ไม่มีสถานะครึ่งๆกลางๆ)

```
Direct Edit (แนวทาง 1)                    Candidate + Commit (แนวทาง 2 - ปลอดภัยกว่า)
┌──────────────┐                          ┌──────────────┐     ┌───────────┐
│ edit-config   │                          │ edit-config   │     │  validate  │
│ target=running │──► Apply ทันที           │ target=       │──►  │  (ตรวจสอบ)  │
└──────────────┘     (เสี่ยง ถ้าผิดพลาด)     │ candidate     │     └─────┬─────┘
                       กระทบทันที             └──────────────┘           │
                                                                    ┌─────▼─────┐
                                                                    │  commit   │──► Apply เข้า running
                                                                    │ (ยืนยันจริง)│    (ถ้า validate ผ่าน)
                                                                    └───────────┘
```

### ตัวอย่างจริง: ใช้ Candidate Datastore เปลี่ยน Description ของ Loopback0 บน CORE-SW1 อย่างปลอดภัย

**ขั้นตอนที่ 1 — ล็อก Candidate Datastore ก่อนแก้ (ป้องกัน Client อื่นแก้พร้อมกัน)**:

```xml
<rpc message-id="201" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <lock>
    <target>
      <candidate/>
    </target>
  </lock>
</rpc>
]]>]]>
```

```xml
<rpc-reply message-id="201" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
]]>]]>
```

**ขั้นตอนที่ 2 — ส่ง `<edit-config>` แก้ candidate datastore (ยังไม่กระทบ running จริง)**:

```xml
<rpc message-id="202" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <candidate/>
    </target>
    <default-operation>merge</default-operation>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
            <description>Router-ID / Management Loopback (Updated via NETCONF Candidate)</description>
          </Loopback>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
]]>]]>
```

```xml
<rpc-reply message-id="202" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
]]>]]>
```

> ณ จุดนี้ ถ้ารัน `show running-config interface Loopback0` ที่ CORE-SW1 จริง **จะยังเห็น
> description เดิม** — เพราะ candidate ยังไม่ถูก commit เข้า running

**ขั้นตอนที่ 3 — `<validate>` ตรวจสอบ candidate ก่อน commit จริง**:

```xml
<rpc message-id="203" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <validate>
    <source>
      <candidate/>
    </source>
  </validate>
</rpc>
]]>]]>
```

```xml
<rpc-reply message-id="203" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
]]>]]>
```

**ขั้นตอนที่ 4 — `<commit>` ยืนยัน — ตรงนี้เท่านั้นที่ Config มีผลจริงกับ `running`**:

```xml
<rpc message-id="204" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit/>
</rpc>
]]>]]>
```

```xml
<rpc-reply message-id="204" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
]]>]]>
```

**ขั้นตอนที่ 5 — ปลดล็อก candidate datastore**:

```xml
<rpc message-id="205" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <unlock>
    <target>
      <candidate/>
    </target>
  </unlock>
</rpc>
]]>]]>
```

หลัง `commit` สำเร็จ `show running-config interface Loopback0` บน CORE-SW1 จะเห็น description
ใหม่ทันที — และเพราะเป็นการ `commit` แบบ Atomic ถ้า `validate` (ขั้นตอนที่ 3) ล้มเหลว
(เช่น syntax ผิด YANG constraint) Config เดิมจะ**ไม่ถูกกระทบเลยแม้แต่นิดเดียว**

> **หมายเหตุ**: Cisco IOS-XE บางรุ่น/บาง feature set รองรับเฉพาะ `:writable-running` (แก้
> running ตรงๆ) โดย candidate/commit ต้องเปิด capability `:candidate` ก่อน — ตรวจสอบได้จาก
> `<hello>` capability list ที่เห็นใน Step 483 ว่ามี
> `urn:ietf:params:netconf:capability:candidate:1.0` หรือไม่

---

## Step 485 — RESTCONF Protocol เจาะลึก

### แนวคิด: REST-Style ทับบน YANG Model เดียวกัน

**RESTCONF (RFC 8040)** ให้เราเข้าถึง YANG Data Model ตัวเดียวกันกับ NETCONF แต่ผ่าน
**HTTP/HTTPS** แบบ REST API ทั่วไปที่โปรแกรมเมอร์ Web คุ้นเคยอยู่แล้ว — ไม่ต้องเรียนรู้ RPC/Session
แบบ NETCONF เลย

### HTTP Method เทียบกับ Operation ของ NETCONF

| HTTP Method | Operation | เทียบเท่า NETCONF | ใช้เมื่อ |
|---|---|---|---|
| `GET` | อ่านข้อมูล (config หรือ state) | `<get>` / `<get-config>` | ดึงค่าปัจจุบัน |
| `POST` | สร้าง resource ใหม่ (ต้องไม่มีอยู่ก่อน) | `<edit-config>` (`operation="create"`) | เพิ่ม entry ใหม่ใน list |
| `PUT` | สร้างหรือแทนที่ resource ทั้งหมด | `<edit-config>` (`operation="replace"`) | ตั้งค่าทั้ง object ใหม่ทั้งหมด |
| `PATCH` | แก้ไขเฉพาะบางฟิลด์ (merge เข้ากับของเดิม) | `<edit-config>` (`operation="merge"`) | แก้แค่บางฟิลด์ (พบบ่อยที่สุด) |
| `DELETE` | ลบ resource | `<edit-config>` (`operation="delete"`) | ลบ config entry |

### โครงสร้าง URI ของ RESTCONF

```
https://<device-ip>/restconf/data/<yang-module>:<container>/<list>=<key>/<leaf>
        └──────┬──────┘└──┬──┘ └────────┬────────┘ └──┬──┘  └─┬─┘  └──┬──┘
          Base URI     Data root    Module:Container   List   Key    Leaf
```

ตัวอย่างจริง: `GET /restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0/1`
อ่านได้ว่า "ดึง entry ใน list `interface` ที่ key `name` = `GigabitEthernet0/1` จาก container
`interfaces` ของ module `ietf-interfaces`" — Path นี้ตรงกับ YANG Model ที่เห็นใน Step 482 เป๊ะ

### เปิดใช้งาน RESTCONF บน CORE-SW1 (ทบทวนจาก Part 23 พร้อมเพิ่มรายละเอียด)

```
CORE-SW1(config)# restconf
CORE-SW1(config)# ip http secure-server
CORE-SW1(config)# ip http authentication local
CORE-SW1(config)# username apiuser privilege 15 secret ApiP@ssw0rd!
CORE-SW1(config)# end
CORE-SW1# show ip http server status | include restconf
Restconf: enabled
```

### ตัวอย่าง `curl` GET แบบเต็ม — ดึง Config ของ Interface Vlan99 (ตรง Path เดียวกับ Step 483)

```bash
curl -k -s -X GET \
  --header "Accept: application/yang-data+json" \
  --user "apiuser:ApiP@ssw0rd!" \
  https://10.10.99.10/restconf/data/ietf-interfaces:interfaces/interface=Vlan99
```

**Response (JSON — เทียบกับ XML Response ของ NETCONF ใน Step 483 คือข้อมูลชุดเดียวกัน):**

```json
{
  "ietf-interfaces:interface": {
    "name": "Vlan99",
    "description": "Management VLAN",
    "type": "iana-if-type:l3ipvlan",
    "enabled": true,
    "ietf-ip:ipv4": {
      "address": [
        {
          "ip": "10.10.99.10",
          "netmask": "255.255.255.0"
        }
      ]
    }
  }
}
```

### ตัวอย่าง `curl` PATCH — แก้ Description ของ Loopback0 (งานเดียวกับ Step 484 แต่ผ่าน RESTCONF)

```bash
curl -k -s -X PATCH \
  --header "Content-Type: application/yang-data+json" \
  --header "Accept: application/yang-data+json" \
  --user "apiuser:ApiP@ssw0rd!" \
  --data '{
    "Cisco-IOS-XE-native:Loopback": {
      "name": "0",
      "description": "Router-ID / Management Loopback (Updated via RESTCONF)"
    }
  }' \
  https://10.10.99.10/restconf/data/Cisco-IOS-XE-native:native/interface/Loopback=0
```

**Response ที่คาดหวัง (สำเร็จ): HTTP Status `204 No Content` (ไม่มี body — RESTCONF PATCH/PUT
สำเร็จแล้วไม่ต้องส่งข้อมูลกลับ)**

```
HTTP/1.1 204 No Content
```

**ตรวจสอบผลลัพธ์ทันทีด้วย GET ซ้ำ:**

```bash
curl -k -s -X GET \
  --header "Accept: application/yang-data+json" \
  --user "apiuser:ApiP@ssw0rd!" \
  https://10.10.99.10/restconf/data/Cisco-IOS-XE-native:native/interface/Loopback=0
```

```json
{
  "Cisco-IOS-XE-native:Loopback": {
    "name": "0",
    "description": "Router-ID / Management Loopback (Updated via RESTCONF)",
    "ip": {
      "address": {
        "primary": {
          "address": "1.1.1.1",
          "mask": "255.255.255.255"
        }
      }
    }
  }
}
```

สังเกตว่าใน**คำขอเดียว** RESTCONF (PATCH) ทำสิ่งที่ NETCONF ทำผ่าน **5 ขั้นตอน** ใน Step 484
(lock → edit-config → validate → commit → unlock) — เพราะ RESTCONF **ไม่มีแนวคิด
candidate/two-phase-commit** เป็นการ Apply ทันทีเสมอ (คล้ายแนวทาง Direct Edit ของ NETCONF)
นี่คือ trade-off หลักระหว่างสอง Protocol: RESTCONF ง่ายกว่าและเร็วกว่าสำหรับงานทั่วไป
แต่ NETCONF ปลอดภัยกว่าสำหรับ Config ที่มีความเสี่ยงสูงและต้องการ Transaction/Rollback

---

## Step 486 — Python + NETCONF ด้วย `ncclient`

### `ncclient` คืออะไร

**`ncclient`** คือ Python library ที่ครอบ (wrap) การคุย NETCONF ทั้งหมดที่เห็นใน Step 483-484
(hello exchange, RPC message-id, message separator `]]>]]>`) ให้เหลือแค่เรียก method ธรรมดา
ของ Python — ไม่ต้องเขียน Raw XML protocol handling เอง

### ติดตั้ง `ncclient`

```bash
pip install ncclient
python3 -c "import ncclient; print(ncclient.__version__)"
# ตัวอย่าง output: 0.6.15
```

### Script เต็ม: ดึง Config Interface + แก้ Description ผ่าน NETCONF ด้วย `ncclient`

```python
#!/usr/bin/env python3
"""
netconf_interface_description.py
เชื่อมต่อ NETCONF ไปยัง CORE-SW1 ด้วย ncclient เพื่อ:
1. ดึง (get-config) description ปัจจุบันของ interface ที่ระบุ
2. แก้ (edit-config) description ใหม่ผ่าน candidate datastore + commit (two-phase)
3. ยืนยันผลลัพธ์ด้วยการ get-config ซ้ำ
"""

import os
import sys
from ncclient import manager
from ncclient.operations.errors import TimeoutExpiredError
from ncclient.transport.errors import AuthenticationError

# --- Connection parameters: อ่านจาก Environment Variable (best practice จาก Part 23) ---
NETCONF_HOST = "10.10.99.10"      # CORE-SW1 MGMT VLAN 99
NETCONF_PORT = 830
NETCONF_USER = os.environ.get("NET_USERNAME", "apiuser")
NETCONF_PASS = os.environ.get("NET_PASSWORD")

if not NETCONF_PASS:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

# --- Filter แบบ subtree: ระบุเฉพาะ interface ที่ต้องการ (ietf-interfaces model) ---
GET_FILTER = """
<filter>
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>Loopback0</name>
    </interface>
  </interfaces>
</filter>
"""

# --- Payload สำหรับ edit-config: ใช้ Cisco-IOS-XE-native model แก้ description ---
def build_edit_payload(new_description: str) -> str:
    return f"""
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <Loopback>
            <name>0</name>
            <description>{new_description}</description>
          </Loopback>
        </interface>
      </native>
    </config>
    """


def main():
    print(f"[*] กำลังเชื่อมต่อ NETCONF ไปยัง CORE-SW1 ({NETCONF_HOST}:{NETCONF_PORT}) ...")

    try:
        with manager.connect(
            host=NETCONF_HOST,
            port=NETCONF_PORT,
            username=NETCONF_USER,
            password=NETCONF_PASS,
            hostkey_verify=False,      # Lab เท่านั้น — Production ต้องตรวจสอบ host key จริง
            device_params={"name": "iosxe"},
            timeout=30,
        ) as m:

            # --- ขั้นที่ 1: ดึง Config ปัจจุบันของ Loopback0 ---
            print("\n[1] ดึง Config ปัจจุบัน (get-config จาก running)")
            current = m.get_config(source="running", filter=GET_FILTER)
            print(current.data_xml)

            # --- ขั้นที่ 2: แก้ candidate datastore ---
            print("\n[2] แก้ description ผ่าน candidate datastore")
            new_desc = "Router-ID / Management Loopback (Updated via ncclient)"
            edit_payload = build_edit_payload(new_desc)

            m.lock(target="candidate")
            try:
                m.edit_config(target="candidate", config=edit_payload)

                # --- ขั้นที่ 3: validate ก่อน commit จริง ---
                print("[3] validate candidate ก่อน commit")
                m.validate(source="candidate")

                # --- ขั้นที่ 4: commit ---
                print("[4] commit -> apply เข้า running จริง")
                m.commit()
                print("    [OK] commit สำเร็จ")

            finally:
                m.unlock(target="candidate")

            # --- ขั้นที่ 5: ยืนยันผลลัพธ์ ---
            print("\n[5] ยืนยันผลลัพธ์ (get-config ซ้ำ)")
            verify = m.get_config(source="running", filter=GET_FILTER)
            print(verify.data_xml)

    except AuthenticationError:
        print("[!] Authentication ล้มเหลว — ตรวจสอบ username/password")
        sys.exit(1)
    except TimeoutExpiredError:
        print("[!] เชื่อมต่อไม่ได้ (Timeout) — ตรวจสอบ netconf-yang enable และ Network/ACL")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### วิธีรันและ Output ที่คาดหวัง

```bash
export NET_USERNAME="apiuser"
export NET_PASSWORD="ApiP@ssw0rd!"
python3 netconf_interface_description.py
```

```
[*] กำลังเชื่อมต่อ NETCONF ไปยัง CORE-SW1 (10.10.99.10:830) ...

[1] ดึง Config ปัจจุบัน (get-config จาก running)
<?xml version="1.0" encoding="UTF-8"?>
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>Loopback0</name>
      <description>Router-ID / Management Loopback</description>
    </interface>
  </interfaces>
</data>

[2] แก้ description ผ่าน candidate datastore
[3] validate candidate ก่อน commit
[4] commit -> apply เข้า running จริง
    [OK] commit สำเร็จ

[5] ยืนยันผลลัพธ์ (get-config ซ้ำ)
<?xml version="1.0" encoding="UTF-8"?>
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>Loopback0</name>
      <description>Router-ID / Management Loopback (Updated via ncclient)</description>
    </interface>
  </interfaces>
</data>
```

> **จุดสำคัญของ `ncclient`**: `manager.connect(...)` จัดการ `<hello>` exchange, session-id,
> และ message framing ให้อัตโนมัติทั้งหมด — `m.lock()`, `m.edit_config()`, `m.validate()`,
> `m.commit()`, `m.unlock()` คือ method ที่ map ตรงกับ RPC operation ที่เขียนเป็น Raw XML
> ใน Step 484 ทุกตัว การใช้ `with manager.connect(...) as m:` ยังช่วยปิด session (`close-session`)
> ให้อัตโนมัติแม้เกิด Exception ระหว่างทาง

---

## Step 487 — Python + RESTCONF ด้วย `requests`

### เตรียม HTTP Header ที่ถูกต้องสำหรับ RESTCONF

RESTCONF ต้องระบุ Media Type แบบเฉพาะเจาะจง (RFC 8040) — ไม่ใช่ `application/json` ธรรมดา
เหมือน REST API ทั่วไป:

| Header | ค่าที่ต้องใช้ | ใช้เมื่อ |
|---|---|---|
| `Accept` | `application/yang-data+json` | ทุก Request (บอกว่า Client รับ Response แบบ JSON) |
| `Content-Type` | `application/yang-data+json` | Request ที่มี Body (`POST`/`PUT`/`PATCH`) |

### Script เต็ม: ดึง Config + แก้ Description ผ่าน RESTCONF ด้วย `requests` (งานเดียวกับ Step 486)

```python
#!/usr/bin/env python3
"""
restconf_interface_description.py
เชื่อมต่อ RESTCONF ไปยัง CORE-SW1 ด้วย requests เพื่อ:
1. GET description ปัจจุบันของ Loopback0
2. PATCH แก้ description ใหม่
3. GET ยืนยันผลลัพธ์อีกครั้ง
(เทียบเท่างานเดียวกับ netconf_interface_description.py ใน Step 486 ทุกประการ)
"""

import os
import sys
import json
import requests

# ปิด Warning ของ requests เมื่อใช้ verify=False กับ self-signed certificate (Lab เท่านั้น)
requests.packages.urllib3.disable_warnings()

RESTCONF_HOST = "10.10.99.10"
BASE_URL = f"https://{RESTCONF_HOST}/restconf/data"

USERNAME = os.environ.get("NET_USERNAME", "apiuser")
PASSWORD = os.environ.get("NET_PASSWORD")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json",
}

LOOPBACK_URI = f"{BASE_URL}/Cisco-IOS-XE-native:native/interface/Loopback=0"


def get_loopback_description():
    """GET: ดึง Config ปัจจุบันของ Loopback0 คืนค่า dict"""
    response = requests.get(
        LOOPBACK_URI,
        headers=HEADERS,
        auth=(USERNAME, PASSWORD),
        verify=False,      # Lab เท่านั้น — Production ต้องตรวจสอบ CA certificate จริง
        timeout=10,
    )
    response.raise_for_status()   # raise exception ถ้า HTTP status เป็น 4xx/5xx
    return response.json()


def patch_loopback_description(new_description: str):
    """PATCH: แก้เฉพาะ field description (merge เข้ากับ config เดิม)"""
    payload = {
        "Cisco-IOS-XE-native:Loopback": {
            "name": "0",
            "description": new_description,
        }
    }
    response = requests.patch(
        LOOPBACK_URI,
        headers=HEADERS,
        auth=(USERNAME, PASSWORD),
        data=json.dumps(payload),
        verify=False,
        timeout=10,
    )
    response.raise_for_status()
    return response.status_code   # คาดหวัง 204 No Content ถ้าสำเร็จ


def main():
    print(f"[*] เชื่อมต่อ RESTCONF ไปยัง CORE-SW1 ({RESTCONF_HOST}) ...")

    try:
        # --- ขั้นที่ 1: ดึง Config ปัจจุบัน ---
        print("\n[1] GET Config ปัจจุบันของ Loopback0")
        current = get_loopback_description()
        print(json.dumps(current, indent=2))

        # --- ขั้นที่ 2: PATCH แก้ description ---
        print("\n[2] PATCH แก้ description ใหม่")
        new_desc = "Router-ID / Management Loopback (Updated via RESTCONF/requests)"
        status = patch_loopback_description(new_desc)
        print(f"    [OK] HTTP Status: {status}")

        # --- ขั้นที่ 3: ยืนยันผลลัพธ์ ---
        print("\n[3] GET ยืนยันผลลัพธ์อีกครั้ง")
        verify = get_loopback_description()
        print(json.dumps(verify, indent=2))

    except requests.exceptions.HTTPError as e:
        print(f"[!] HTTP Error: {e.response.status_code} - {e.response.text}")
        sys.exit(1)
    except requests.exceptions.ConnectionError:
        print("[!] เชื่อมต่อไม่ได้ — ตรวจสอบ restconf enable, ip http secure-server, Network/ACL")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### วิธีรันและ Output ที่คาดหวัง

```bash
export NET_USERNAME="apiuser"
export NET_PASSWORD="ApiP@ssw0rd!"
pip install requests
python3 restconf_interface_description.py
```

```
[*] เชื่อมต่อ RESTCONF ไปยัง CORE-SW1 (10.10.99.10) ...

[1] GET Config ปัจจุบันของ Loopback0
{
  "Cisco-IOS-XE-native:Loopback": {
    "name": "0",
    "description": "Router-ID / Management Loopback",
    "ip": {
      "address": {
        "primary": {
          "address": "1.1.1.1",
          "mask": "255.255.255.255"
        }
      }
    }
  }
}

[2] PATCH แก้ description ใหม่
    [OK] HTTP Status: 204

[3] GET ยืนยันผลลัพธ์อีกครั้ง
{
  "Cisco-IOS-XE-native:Loopback": {
    "name": "0",
    "description": "Router-ID / Management Loopback (Updated via RESTCONF/requests)",
    "ip": {
      "address": {
        "primary": {
          "address": "1.1.1.1",
          "mask": "255.255.255.255"
        }
      }
    }
  }
}
```

### เปรียบเทียบสั้นๆ: `ncclient` (Step 486) vs `requests` (Step 487)

| ประเด็น | `ncclient` (NETCONF) | `requests` (RESTCONF) |
|---|---|---|
| จำนวนขั้นตอนสำหรับแก้ 1 field | 5 (lock → edit → validate → commit → unlock) | 1 (PATCH เดียว) |
| Rollback ถ้า Config ผิด | มี (candidate ไม่ถูก commit ถ้า validate fail) | ไม่มี (Apply ทันทีทุกครั้ง) |
| Library ที่ต้องเรียนรู้ | เฉพาะทาง (`ncclient`) | ทั่วไป (`requests` — ใช้กับ Web API อะไรก็ได้) |
| เหมาะกับ | Config ที่มีความเสี่ยงสูง ต้องการ Transaction | งานทั่วไป, Integration กับระบบ Web/Dashboard |

---

## Step 488 — เครื่องมือสำรวจ YANG Model

### `pyang` — ตรวจสอบความถูกต้องของ YANG Module

**`pyang`** เป็นเครื่องมือ Python ตรวจสอบ Syntax ของไฟล์ `.yang` เอง (ไม่ต้องต่ออุปกรณ์จริง) —
ใช้ตรวจ Module ที่เขียนเอง หรือตรวจ Module ที่ดาวน์โหลดมาจาก Cisco YANG Suite ก่อนนำไปใช้จริง

```bash
pip install pyang

# ตรวจสอบ syntax ของไฟล์ YANG (จาก Step 481)
pyang example-interfaces.yang
# ไม่มี error/warning = syntax ถูกต้อง

# แปลง YANG module ให้อ่านง่ายเป็นรูปแบบ tree (เห็นโครงสร้าง container/leaf/list ชัดเจน)
pyang -f tree example-interfaces.yang
```

**ตัวอย่าง Output ของ `pyang -f tree`:**

```
module: example-interfaces
  +--rw interfaces
     +--rw interface* [name]
        +--rw name           string
        +--rw description?   string
        +--rw enabled?        boolean
        +--rw ip-address*    string
```

อ่านสัญลักษณ์: `+--rw` = read-write (config), `*` หลังชื่อ = list/leaf-list (มีได้หลาย entry),
`?` = optional (leaf ที่ไม่จำเป็นต้องมีค่า), `[name]` = key ของ list

### `yanglint` — ตรวจสอบ XML/JSON Instance Data ตรงตาม Schema

**`yanglint`** (จากโครงการ `libyang`) ใช้ตรวจว่า **ข้อมูลจริง** (XML/JSON payload ที่จะส่งไป
NETCONF/RESTCONF) ตรงตาม Schema ของ YANG Module หรือไม่ — ก่อนส่งไปที่อุปกรณ์จริง ช่วยจับ error
ตั้งแต่ตอนพัฒนา Script ไม่ต้องรอ error กลับมาจากอุปกรณ์

```bash
# Ubuntu/Debian
apt install yang-tools

# ตรวจสอบว่าไฟล์ JSON payload ตรงตาม Schema ของ example-interfaces.yang หรือไม่
yanglint example-interfaces.yang interface_payload.json
```

### Cisco YANG Suite — เครื่องมือ GUI สำหรับสำรวจ YANG Model แบบครบวงจร

**Cisco YANG Suite** คือ Web-based Tool (Open Source จาก Cisco DevNet) ที่ให้:

- Browse YANG Module ทั้งหมดที่อุปกรณ์รองรับ (Native/OpenConfig/IETF) แบบ Tree View
- ทดสอบส่ง NETCONF RPC / RESTCONF Request ผ่าน GUI โดยไม่ต้องเขียน Script ก่อน
- Generate ตัวอย่าง Python code (`ncclient`/`requests`) จาก Request ที่ทดสอบผ่าน GUI แล้ว

```bash
# ติดตั้ง YANG Suite (Docker แนะนำที่สุด)
git clone https://github.com/CiscoDevNet/yangsuite.git
cd yangsuite
docker compose up -d
# เข้าใช้งานที่ https://localhost (default)
```

### ตรวจสอบ Model ที่อุปกรณ์รองรับจริงด้วย `show yang-interfaces`

```
CORE-SW1# show yang-interfaces
Interface        Yang-interface-name
Loopback0        Cisco-IOS-XE-interfaces-oper:interfaces/interface[name='Loopback0']
Vlan99           Cisco-IOS-XE-interfaces-oper:interfaces/interface[name='Vlan99']
Port-channel1    Cisco-IOS-XE-interfaces-oper:interfaces/interface[name='Port-channel1']
```

### ตรวจสอบผ่าน RESTCONF YANG Library — วิธีที่แนะนำที่สุดสำหรับ Automation จริง

`ietf-yang-library` (RFC 8525) คือ Module มาตรฐานที่**ทุกอุปกรณ์ที่รองรับ RESTCONF ต้องมี**
ใช้ query เพื่อขอรายชื่อ Module ทั้งหมดที่อุปกรณ์โหลดไว้จริง ณ เวลานั้น:

```bash
curl -k -s -X GET \
  --header "Accept: application/yang-data+json" \
  --user "apiuser:ApiP@ssw0rd!" \
  https://10.10.99.10/restconf/data/ietf-yang-library:yang-library
```

**Response (ตัดให้สั้น — ของจริงมีหลายร้อย Module):**

```json
{
  "ietf-yang-library:yang-library": {
    "module-set": [
      {
        "name": "modules",
        "module": [
          {
            "name": "Cisco-IOS-XE-native",
            "revision": "2022-01-01",
            "namespace": "http://cisco.com/ns/yang/Cisco-IOS-XE-native"
          },
          {
            "name": "ietf-interfaces",
            "revision": "2018-02-20",
            "namespace": "urn:ietf:params:xml:ns:yang:ietf-interfaces"
          },
          {
            "name": "openconfig-interfaces",
            "revision": "2021-04-06",
            "namespace": "http://openconfig.net/yang/interfaces"
          }
        ]
      }
    ]
  }
}
```

Script Automation ระดับ Production ควร query path นี้ **ก่อน** ส่ง Config จริง เพื่อตรวจสอบว่า
อุปกรณ์ปลายทาง (ซึ่งอาจเป็น IOS-XE version ต่างกันในแต่ละ Site) รองรับ Module และ Revision
ที่ Script คาดหวังไว้จริงหรือไม่ — ป้องกัน Error ที่จะเกิดตอน `edit-config`/`PATCH` จริง

---

## Step 489 — แอบดู Model-Driven Telemetry (เจาะลึกใน Part 80)

### YANG Model ตัวเดียวกัน แต่ใช้กับ "การรับข้อมูล" แทน "การส่ง Config"

ทุก Protocol ที่เรียนใน Part นี้ (NETCONF Step 483-484, RESTCONF Step 485) เป็นรูปแบบ
**Pull-Based**: Client ต้อง**ร้องขอ** (GET/get-config) ทุกครั้งที่ต้องการข้อมูลใหม่ — เหมาะกับ
Config Management แต่**ไม่เหมาะกับ Monitoring แบบ Real-time** (ถ้า Poll ทุก 5 วินาทีจากอุปกรณ์
นับพันตัว จะสร้าง Load สูงมากทั้งฝั่ง Client และ Device)

**Model-Driven Telemetry (MDT)** แก้ปัญหานี้ด้วยแนวคิด **Push-Based**: อุปกรณ์เป็นฝ่าย
**ส่งข้อมูล Streaming ออกมาเองอย่างต่อเนื่อง** ตาม YANG Path ที่ Subscribe ไว้ล่วงหน้า
ผ่าน **gRPC** (Dial-Out หรือ Dial-In) — ไม่ต้อง Poll ซ้ำๆอีกต่อไป

```
Pull-Based (NETCONF/RESTCONF - Part นี้)         Push-Based (Model-Driven Telemetry - Part 80)
┌──────────┐   GET ทุก 5 วิ    ┌──────────┐      ┌──────────┐   Subscribe ครั้งเดียว   ┌──────────┐
│  Client   │ ───────────────►  │  Device   │      │  Client   │ ─────────────────────►  │  Device   │
│           │ ◄─────────────── │           │      │           │ ◄── stream ต่อเนื่อง ──── │           │
└──────────┘   Response ทีละครั้ง └──────────┘      └──────────┘   (gRPC, ไม่ต้องขอซ้ำ)   └──────────┘
   Client ต้องเดาว่าจะ Poll                            Device ส่งข้อมูลทุกครั้งที่ค่าเปลี่ยน
   ถี่แค่ไหนจึงจะไม่พลาดข้อมูลสำคัญ                        (Event-Driven) หรือตามช่วงเวลาที่กำหนด (Cadence)
```

### ตารางสรุปเปรียบเทียบสั้นๆ (รายละเอียดเต็มใน Part 80)

| ประเด็น | NETCONF/RESTCONF (Part นี้) | Model-Driven Telemetry (Part 80) |
|---|---|---|
| ทิศทาง | Pull (Client ร้องขอ) | Push (Device ส่งเอง) |
| Transport | SSH (830) / HTTPS (443) | gRPC (มักใช้ port 57500 หรือกำหนดเอง) |
| YANG Model ที่ใช้ | Native / IETF / OpenConfig | ตัวเดียวกัน! (มักนิยม OpenConfig สำหรับ Telemetry) |
| Use Case หลัก | Config Management, การเปลี่ยนแปลงเชิงโครงสร้าง | Real-time Monitoring, Dashboard, Anomaly Detection |
| ตัวอย่างระบบปลายทางที่รับข้อมูล | Script/Application ของเราเอง | Pipeline/Collector เช่น Telegraf, Cisco Telemetry Broker |

> **ประเด็นสำคัญที่สุดที่ต้องจำจาก Step นี้**: YANG Data Model คือ "ภาษากลาง" ที่ใช้ร่วมกันได้
> ทั้งฝั่ง Config (NETCONF/RESTCONF) และฝั่ง Telemetry (gRPC Streaming) — เมื่อเข้าใจ YANG อย่าง
> ถ่องแท้จาก Part นี้แล้ว การเรียน Model-Driven Telemetry ใน Part 80 จะเป็นแค่ "เปลี่ยน Transport
> และทิศทางการไหลของข้อมูล" ไม่ต้องเรียนรู้แนวคิด Data Model ใหม่อีกเลย

---

## Step 490 — Lab เต็มรูปแบบ: สร้าง Loopback Interface 3 วิธี

### เป้าหมายของ Lab

สร้าง **Loopback99** บน CORE-SW1 พร้อม IP Address และ Description เดียวกัน โดยทำ **งานเดียวกัน
ด้วย 3 วิธีที่ต่างกัน** เพื่อเปรียบเทียบให้เห็นชัดว่าสุดท้าย Config ที่ได้บนอุปกรณ์**เหมือนกัน
เป๊ะ**ไม่ว่าจะใช้วิธีไหน — สิ่งที่ต่างกันคือ **กระบวนการ (Process)** ในการไปถึงผลลัพธ์นั้น

**Spec ของ Loopback99** (ใช้ Test/Documentation Range ตาม RFC 5737 — ไม่ชนกับ IP Plan จริง
ใน `00-ip-address-plan.md`):

| Field | ค่า |
|---|---|
| Interface | Loopback99 |
| IP Address | 192.0.2.49/32 |
| Description | `TEST - API Comparison Lab (Part 49)` |

### วิธีที่ 1 — Netmiko / CLI ดั้งเดิม (ทบทวนจาก Part 23/47)

```python
#!/usr/bin/env python3
"""
method1_netmiko_cli.py
สร้าง Loopback99 บน CORE-SW1 ด้วย Netmiko (CLI แบบดั้งเดิม)
"""

import os
import sys
from netmiko import ConnectHandler

CORE_SW1 = {
    "device_type": "cisco_ios",
    "host": "10.10.99.10",
    "username": os.environ.get("NET_USERNAME", "admin"),
    "password": os.environ.get("NET_PASSWORD"),
    "secret": os.environ.get("NET_SECRET"),
}

CONFIG_COMMANDS = [
    "interface Loopback99",
    "description TEST - API Comparison Lab (Part 49)",
    "ip address 192.0.2.49 255.255.255.255",
    "no shutdown",
]

if not CORE_SW1["password"] or not CORE_SW1["secret"]:
    sys.exit("[!] กรุณา export NET_PASSWORD และ NET_SECRET ก่อนรัน")

connection = ConnectHandler(**CORE_SW1)
connection.enable()
output = connection.send_config_set(CONFIG_COMMANDS)
connection.save_config()
print(output)
connection.disconnect()
print("[OK] สร้าง Loopback99 ผ่าน Netmiko/CLI สำเร็จ")
```

### วิธีที่ 2 — NETCONF ด้วย `ncclient`

```python
#!/usr/bin/env python3
"""
method2_netconf_ncclient.py
สร้าง Loopback99 บน CORE-SW1 ด้วย NETCONF (ncclient) ผ่าน candidate + commit
"""

import os
import sys
from ncclient import manager

HOST = "10.10.99.10"
USERNAME = os.environ.get("NET_USERNAME", "apiuser")
PASSWORD = os.environ.get("NET_PASSWORD")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน")

EDIT_PAYLOAD = """
<config>
  <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
    <interface>
      <Loopback>
        <name>99</name>
        <description>TEST - API Comparison Lab (Part 49)</description>
        <ip>
          <address>
            <primary>
              <address>192.0.2.49</address>
              <mask>255.255.255.255</mask>
            </primary>
          </address>
        </ip>
      </Loopback>
    </interface>
  </native>
</config>
"""

with manager.connect(
    host=HOST, port=830, username=USERNAME, password=PASSWORD,
    hostkey_verify=False, device_params={"name": "iosxe"}, timeout=30,
) as m:
    m.lock(target="candidate")
    try:
        m.edit_config(target="candidate", config=EDIT_PAYLOAD)
        m.validate(source="candidate")
        m.commit()
        print("[OK] สร้าง Loopback99 ผ่าน NETCONF (ncclient) สำเร็จ")
    finally:
        m.unlock(target="candidate")
```

### วิธีที่ 3 — RESTCONF ด้วย `requests`

```python
#!/usr/bin/env python3
"""
method3_restconf_requests.py
สร้าง Loopback99 บน CORE-SW1 ด้วย RESTCONF (requests) ผ่าน PUT
"""

import os
import sys
import json
import requests

requests.packages.urllib3.disable_warnings()

HOST = "10.10.99.10"
USERNAME = os.environ.get("NET_USERNAME", "apiuser")
PASSWORD = os.environ.get("NET_PASSWORD")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน")

URI = f"https://{HOST}/restconf/data/Cisco-IOS-XE-native:native/interface/Loopback=99"

PAYLOAD = {
    "Cisco-IOS-XE-native:Loopback": {
        "name": "99",
        "description": "TEST - API Comparison Lab (Part 49)",
        "ip": {
            "address": {
                "primary": {
                    "address": "192.0.2.49",
                    "mask": "255.255.255.255",
                }
            }
        },
    }
}

response = requests.put(
    URI,
    headers={
        "Content-Type": "application/yang-data+json",
        "Accept": "application/yang-data+json",
    },
    auth=(USERNAME, PASSWORD),
    data=json.dumps(PAYLOAD),
    verify=False,
    timeout=10,
)
response.raise_for_status()
print(f"[OK] สร้าง Loopback99 ผ่าน RESTCONF (requests) สำเร็จ — HTTP {response.status_code}")
```

> ใช้ `PUT` (ไม่ใช่ `PATCH`) ในวิธีนี้เพราะ Loopback99 **ยังไม่มีอยู่** — `PUT` ทำหน้าที่ทั้ง
> "สร้างใหม่" (ถ้ายังไม่มี) หรือ "แทนที่ทั้งหมด" (ถ้ามีอยู่แล้ว) ตรงตามตาราง Step 485

### ตารางเปรียบเทียบ 3 วิธี — Pros/Cons

| ประเด็น | 1) Netmiko/CLI | 2) NETCONF (`ncclient`) | 3) RESTCONF (`requests`) |
|---|---|---|---|
| **ความเร็วในการเขียน Script** | เร็วที่สุด (คุ้นเคย, เขียน command list ตรงๆ) | ช้าที่สุด (ต้องรู้ XML/YANG Path ให้ถูก) | ปานกลาง (ต้องรู้ JSON/YANG Path แต่ syntax ง่ายกว่า XML) |
| **Schema Validation ก่อน Apply** | ไม่มี (ส่ง command ไปตรงๆ ผิด syntax ค่อยรู้ทีหลัง) | มี (ผ่าน `validate` ก่อน `commit`) | ไม่มี (Apply ทันที ไม่มี validate step แยก) |
| **Rollback อัตโนมัติถ้าผิดพลาด** | ไม่มี | มี (candidate ไม่ commit ถ้า validate fail) | ไม่มี |
| **Parsing Output ที่ได้กลับมา** | ต้อง parse text เอง (เปราะบาง) | Schema ตายตัว (XML ตรง YANG) | Schema ตายตัว (JSON ตรง YANG, อ่านง่ายกว่า XML) |
| **เหมาะกับ Integration ระบบ Web/Dashboard** | ไม่เหมาะ | ไม่เหมาะ (ต้องมี NETCONF client library เฉพาะทาง) | เหมาะที่สุด (เป็น REST API มาตรฐานเดียวกับ Web ทั่วไป) |
| **รองรับ Multi-vendor (Juniper/Arista)** | ต้องเขียน driver แยกทีละยี่ห้อ (`device_type` ต่างกัน) | ได้ (ถ้าใช้ OpenConfig/IETF model แทน Native) | ได้ (เหตุผลเดียวกัน) |
| **ความเสี่ยงต่อ Configuration Drift** | สูง (ไม่มี Schema บังคับ) | ต่ำ (Schema บังคับ + Transaction) | ต่ำ (Schema บังคับ แต่ไม่มี Transaction) |

### ตรวจสอบผลลัพธ์ — ทั้ง 3 วิธีต้องได้ Config เหมือนกันเป๊ะ

รันคำสั่งนี้บน CORE-SW1 หลังจากรันแต่ละวิธี (ทดสอบแยกกันคนละรอบ โดยลบ Loopback99 ทิ้งก่อนทดสอบ
วิธีถัดไปด้วย `no interface Loopback99`):

```
CORE-SW1# show running-config interface Loopback99
Building configuration...

Current configuration : 122 bytes
!
interface Loopback99
 description TEST - API Comparison Lab (Part 49)
 ip address 192.0.2.49 255.255.255.255
end
```

**ผลลัพธ์ที่ยืนยันแล้วจากการทดสอบทั้ง 3 วิธี**:

| วิธี | Description ตรงตาม Spec | IP Address ตรงตาม Spec | `no shutdown` (ค่า default ของ Loopback คือ up) |
|---|---|---|---|
| 1) Netmiko/CLI | ✅ | ✅ | ✅ (สั่ง `no shutdown` ตรงๆ) |
| 2) NETCONF (`ncclient`) | ✅ | ✅ | ✅ (Loopback ไม่มี `shutdown` โดย default บน IOS-XE) |
| 3) RESTCONF (`requests`) | ✅ | ✅ | ✅ (เหตุผลเดียวกับข้อ 2) |

**สรุป Lab**: ไม่ว่าจะสร้างผ่าน CLI, NETCONF, หรือ RESTCONF — `running-config` ที่ CORE-SW1
เก็บไว้จริงคือ**ข้อมูลชุดเดียวกัน** เพราะทั้งสาม Protocol สุดท้ายแล้วล้วนแก้ไข **Configuration
Database ตัวเดียวกัน** ของ IOS-XE เพียงแต่ "ประตูทางเข้า" (Interface) ที่ใช้ต่างกัน — นี่คือ
เหตุผลที่ทีม NetOps ระดับ Enterprise สามารถ**ผสมทั้ง 3 วิธีในระบบเดียวกันได้**: ใช้ RESTCONF
สำหรับ Dashboard ที่ทีมอื่นต้องเรียกดูสถานะ, ใช้ NETCONF สำหรับ Pipeline ที่ต้องการ
Transaction/Rollback (เช่น เปลี่ยน BGP Policy ทั้ง Site), และยังคงมี Netmiko/CLI ไว้สำหรับ
Script เก่าที่ยังไม่ migrate — ทั้งหมดจะไม่ทำให้ Config ขัดแย้งกันเอง ตราบใดที่ทุก Script
แก้ Config ให้ตรงกับ Source of Truth เดียวกัน (แนวคิด Network-as-Code จาก Part 47-48)

---

## แบบฝึกหัดทวนความเข้าใจ Part 49

1. ทำไม Model-Driven API (NETCONF/RESTCONF) จึงแก้ปัญหา "CLI Screen-Scraping" ของ Netmiko/Ansible
   ที่เรียนใน Part 23 ได้ และ YANG มีบทบาทอะไรในเรื่องนี้?
2. อธิบายความแตกต่างระหว่าง YANG Model ประเภท Native, OpenConfig, และ IETF Standard —
   ถ้าต้องเขียน Automation Script ที่ต้องทำงานได้ทั้งกับ Cisco และ Juniper ควรเลือกใช้ Model
   ประเภทใด และเพราะเหตุใด?
3. อธิบายขั้นตอน Two-Phase Commit ของ NETCONF (candidate → validate → commit) และบอกข้อดีของ
   วิธีนี้เทียบกับการแก้ `running` datastore ตรงๆ
4. HTTP Method `PATCH` ของ RESTCONF เทียบเท่ากับ NETCONF Operation อะไร และทำไม RESTCONF
   จึง**ไม่มี**ขั้นตอนเทียบเท่า `validate`/`commit` แบบ NETCONF?
5. ในกรณีที่ทีม NetOps ใช้ทั้ง Netmiko/CLI, NETCONF, และ RESTCONF ปนกันในองค์กรเดียว
   (ตามที่กล่าวใน Step 490) เพราะเหตุใด Config บนอุปกรณ์จึงยังคงตรงกันได้โดยไม่ขัดแย้งกันเอง?

**เฉลย:**

1. CLI Screen-Scraping ต้อง parse text output ที่ไม่มี Schema ตายตัว (เปราะบางต่อการเปลี่ยน
   IOS version) ส่วน Model-Driven API ใช้ **YANG** เป็น Schema กลางที่ทั้ง Client และอุปกรณ์
   รู้ล่วงหน้าร่วมกัน — ข้อมูลที่ส่ง/รับผ่าน NETCONF (XML) หรือ RESTCONF (JSON) จึงมีโครงสร้าง
   ตายตัวเสมอ ไม่ต้องเดา format และตรวจสอบ (Validate) ได้ก่อน Apply จริง
2. Native (`Cisco-IOS-XE-native`) ครอบคลุมทุก Feature ของ Cisco แต่ใช้ได้เฉพาะ Cisco เท่านั้น,
   OpenConfig เป็น Vendor-neutral ออกแบบร่วมกันโดยกลุ่ม Operator หลาย Vendor,
   IETF Standard เป็นมาตรฐานสากลเก่าแก่ที่สุด ครอบคลุมเฉพาะ Feature พื้นฐานร่วม — ถ้าต้องทำงาน
   ข้าม Cisco/Juniper ควรใช้ **OpenConfig** เพราะออกแบบมาให้ Config เดียวกันใช้ได้กับหลาย Vendor
   โดยไม่ต้องเขียน Path แยกทีละยี่ห้อ
3. Two-Phase Commit คือ: (1) แก้ Config ใน `candidate` datastore ก่อน (ยังไม่กระทบ running จริง)
   (2) `validate` ตรวจสอบว่า Config ถูกต้องตาม Schema/Constraint (3) `commit` ยืนยันให้มีผลจริง
   กับ `running` แบบ Atomic — ข้อดีคือถ้า validate ล้มเหลว Config เดิมจะไม่ถูกกระทบเลย
   ต่างจากการแก้ `running` ตรงๆที่ Apply ทันทีและเสี่ยงกระทบอุปกรณ์จริงถ้า Config ผิดพลาด
4. `PATCH` เทียบเท่า `<edit-config>` ที่มี `operation="merge"` (แก้เฉพาะบางฟิลด์ merge เข้ากับ
   ของเดิม) — RESTCONF ไม่มี candidate/validate/commit เพราะออกแบบมาให้เป็น REST API แบบ
   Stateless ที่ Apply ทันทีเสมอ (คล้ายแนวทาง Direct Edit ของ NETCONF) เพื่อความง่ายและเร็ว
   ในการ Integration กับระบบ Web ทั่วไป โดยแลกกับการไม่มี Transaction/Rollback อัตโนมัติ
5. เพราะทั้ง 3 วิธี (CLI, NETCONF, RESTCONF) สุดท้ายแล้วล้วนแก้ไข **Configuration Database
   ตัวเดียวกัน** ของอุปกรณ์ IOS-XE เพียงแต่เป็น "ประตูทางเข้า" (Interface) ที่ต่างกัน — ตราบใดที่
   ทุก Script/Playbook อ้างอิง Source of Truth เดียวกัน (เช่น Git repository ตามแนวคิด
   Network-as-Code จาก Part 47-48) Config ที่ได้จะตรงกันเสมอไม่ว่าจะเขียนผ่านช่องทางใด

---

## สรุป Part 49

Part นี้เจาะลึก **Model-Driven Programmability** เต็มรูปแบบตามที่ [Part 23 Step 229](part-023-automation-basics.md)
ได้ forward-reference ไว้ — เราเริ่มจากพื้นฐาน **YANG Data Model** (container/leaf/list/leaf-list)
และเปรียบเทียบ Model 3 ประเภท (Native/OpenConfig/IETF), เจาะลึก **NETCONF** ทั้ง RPC Operation
และ Datastore แบบ Two-Phase Commit ด้วย Raw XML จริง, เจาะลึก **RESTCONF** ทั้ง HTTP Method
และ URI Structure ด้วย `curl` จริง, เขียน Python ควบคุมทั้งสอง Protocol ด้วย `ncclient` และ
`requests`, สำรวจเครื่องมือ YANG (`pyang`, `yanglint`, YANG Suite, YANG Library), แอบดู
Model-Driven Telemetry เป็นการเตรียมพื้นฐานสำหรับ Part 80, และปิดท้ายด้วย Lab เปรียบเทียบสร้าง
Loopback Interface เดียวกันด้วย 3 วิธี พร้อมพิสูจน์ว่า Config ที่ได้บนอุปกรณ์เหมือนกันเป๊ะ ✅

ทักษะ YANG/NETCONF/RESTCONF เหล่านี้คือรากฐานสำคัญของ **DevNet Professional** และ
**Cisco DNA Center's Intent API** ที่ Part ต่อไปจะพาไปทำความรู้จัก — **DNA Center**
คือ Platform ระดับ Enterprise ของ Cisco ที่รวม Automation, Assurance, และ SD-Access
Controller เข้าไว้ในที่เดียว โดยเบื้องหลังใช้ NETCONF/RESTCONF/YANG ที่เราเพิ่งเรียนจบไปนี่เอง
เป็นกลไกหลักในการสื่อสารกับอุปกรณ์ทุกตัวในเครือข่าย

**ไปต่อ:** [Part 50 — Cisco DNA Center →](part-050-dna-center.md)
