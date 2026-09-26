# Part 47 — Network Automation: Python for Network Engineers (Advanced)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 461–470 จาก 1000**

> ต่อจาก [Part 46 — Control Plane Policing](part-046-control-plane-policing.md) ที่เราปกป้อง
> CPU ของอุปกรณ์ด้วย CoPP แล้ว Part นี้พาเรากลับมาที่สาย Automation อีกครั้ง — [Part 23](part-023-automation-basics.md)
> สอน Netmiko/Ansible/Jinja2 ระดับพื้นฐาน (CCNA-level) ไปแล้ว Part 47 นี้คือ**ภาคขยายฝั่ง Python**
> อย่างจริงจังสำหรับระดับ CCNP ENCOR — เราจะเลิกเขียน script แบบ "ใช้ครั้งเดียวทิ้ง" แล้วเริ่มสร้าง
> Automation Tooling ที่ **multi-vendor, parse structured data ได้, ทนต่อความล้มเหลว, รันแบบ
> concurrent ได้จริง, เก็บ version history, และเปิดเป็น API ให้ระบบอื่นเรียกใช้** — นี่คือทักษะที่
> แยกวิศวกรที่ "เขียน script ได้" ออกจากวิศวกรที่ "สร้างระบบ Automation ระดับ Production ได้"
> (CI/CD Pipeline แบบเต็มรูปแบบและ Ansible ระดับ Advanced จะอยู่ใน **Part 48**, NETCONF/RESTCONF/YANG
> เต็มรูปแบบจะอยู่ใน **Part 49**)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 461 | Beyond Netmiko — แนะนำ NAPALM สำหรับ Multi-Vendor Abstraction |
| 462 | Nornir Framework พื้นฐาน — Inventory, Task, และ Concurrency |
| 463 | Parsing CLI Output ด้วย TextFSM/ntc-templates |
| 464 | สร้าง Network Inventory/CMDB ของตัวเองด้วย Python |
| 465 | Configuration Compliance Checking ด้วย Python |
| 466 | Error Handling และ Retry Logic สำหรับ Production Script |
| 467 | Parallel Execution ด้วย `concurrent.futures.ThreadPoolExecutor` |
| 468 | Git Integration สำหรับ Config Version Control |
| 469 | สร้าง REST API ด้วย Flask ครอบ Automation Script |
| 470 | Lab เต็มรูปแบบ: "NetOps Toolkit" — รวมทุกเทคนิคเข้าด้วยกัน |

---

## Step 461 — Beyond Netmiko: แนะนำ NAPALM

### ทำไม Netmiko ไม่พอสำหรับงานระดับ Enterprise

[Part 23](part-023-automation-basics.md) สอน Netmiko ให้ SSH เข้าอุปกรณ์แล้วรันคำสั่ง CLI ตรงๆ
(`send_command`, `send_config_set`) — วิธีนี้เรียกว่า **CLI-based automation**: เราส่งคำสั่ง
text แล้วได้ text กลับมา ปัญหาคือถ้า Lab มีทั้ง Cisco IOS-XE, Arista EOS, Juniper JunOS
ปนกัน (ซึ่งเกิดขึ้นจริงในองค์กรใหญ่ที่ M&A บริษัทอื่นเข้ามา หรือใช้ ISP หลายเจ้า) คำสั่ง
`show version` ของแต่ละยี่ห้อจะได้ output ที่ format ไม่เหมือนกันเลย ทำให้ script ที่เขียนไว้สำหรับ
Cisco ใช้กับอุปกรณ์อื่นไม่ได้ ต้องเขียน parser ใหม่ทุกครั้ง

**NAPALM (Network Automation and Programmability Abstraction Layer with Multivendor support)**
คือ Python library ที่แก้ปัญหานี้โดยตรง — มันมี **getter methods มาตรฐาน** (เช่น `get_facts()`,
`get_interfaces()`, `get_bgp_neighbors()`) ที่คืนค่าเป็น **Python dict ที่มี structure เดียวกัน
เสมอ ไม่ว่าจะต่อกับอุปกรณ์ยี่ห้อไหนก็ตาม** (Cisco IOS/IOS-XE, Cisco NX-OS, Cisco IOS-XR, Arista
EOS, Juniper JunOS ผ่าน driver ที่ NAPALM เขียนไว้ให้แต่ละยี่ห้อ)

### เปรียบเทียบ Netmiko vs NAPALM vs Nornir

| ประเด็น | Netmiko | NAPALM | Nornir |
|---|---|---|---|
| ระดับ (Layer) | Low-level: SSH/CLI transport | Mid-level: Abstraction บน CLI/API ให้ data เป็น structure มาตรฐาน | High-level: Framework/Orchestration สำหรับรัน task จำนวนมาก |
| Output ที่ได้ | Raw text (ต้อง parse เอง) | **Structured Python dict/list ที่ normalize แล้ว** | ขึ้นกับ task ที่เขียน (มักเรียก Netmiko/NAPALM ข้างในอีกที) |
| Multi-vendor | ต้องรู้ syntax คำสั่งของแต่ละยี่ห้อเอง | **ใช่ — method เดียวกันใช้ได้ทุกยี่ห้อที่มี driver** | ใช่ (ผ่าน plugin เช่น `nornir_napalm`, `nornir_netmiko`) |
| Concurrency (รันหลายอุปกรณ์พร้อมกัน) | ไม่มีในตัว (ต้องเขียน loop/threading เอง) | ไม่มีในตัว (เชื่อมทีละ device object) | **มีในตัว (multi-threaded โดย default)** |
| Config Diff / Rollback | ไม่มี (ต้องเขียนเอง) | **มีในตัว (`compare_config()`, `rollback()`)** | ขึ้นกับ plugin ที่เรียก |
| ใช้เมื่อไหร่ | ต้องส่งคำสั่ง CLI แบบเจาะจง/interactive | ต้องดึงข้อมูล structured จากหลายยี่ห้อ, ทำ config diff/rollback | ต้อง orchestrate task จำนวนมากกับอุปกรณ์จำนวนมาก พร้อม inventory ที่จัดการง่าย |
| ความสัมพันธ์กัน | เป็น transport พื้นฐาน | ใช้ Netmiko/NETCONF เป็น backend ได้ | ใช้ Netmiko หรือ NAPALM เป็น "connection plugin" ได้ทั้งคู่ |

> **แนวคิดสำคัญ**: ทั้งสามตัวไม่ใช่คู่แข่งที่ต้องเลือกอย่างใดอย่างหนึ่ง — Nornir คือ Framework
> ที่ **ครอบ** Netmiko หรือ NAPALM อีกที (ดู Step 462) งานจริงระดับ Enterprise มักใช้ **Nornir
> เป็นตัวจัดการ Concurrency + Inventory** และเลือก **NAPALM หรือ Netmiko เป็น connection plugin**
> ขึ้นกับว่างานนั้นต้องการ structured data (เลือก NAPALM) หรือส่งคำสั่ง CLI ตรงๆ (เลือก Netmiko)

### ติดตั้ง NAPALM

```bash
pip install napalm
# ตรวจสอบ driver ที่รองรับ
python3 -c "from napalm import get_network_driver; print('napalm installed OK')"
```

| Platform | NAPALM `driver` string |
|---|---|
| Cisco IOS / IOS-XE (Catalyst 9500/9300 ใน Lab) | `ios` |
| Cisco NX-OS | `nxos` หรือ `nxos_ssh` |
| Cisco IOS-XR | `iosxr` |
| Arista EOS | `eos` |
| Juniper JunOS | `junos` |

### Script เต็ม: ดึง Facts จาก CORE-SW1 ด้วย NAPALM

```python
#!/usr/bin/env python3
"""
napalm_get_facts.py
เชื่อมต่อไปยัง CORE-SW1 ด้วย NAPALM แล้วดึง facts + interfaces แบบ structured data
เทียบกับ Netmiko (Part 23 Step 222) ที่ต้อง parse text เอง
"""

from napalm import get_network_driver
import json
import os
import sys

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

# CORE-SW1 MGMT VLAN 99 ตาม docs/00-ip-address-plan.md
HOST = "10.10.99.10"


def main():
    driver = get_network_driver("ios")

    device = driver(
        hostname=HOST,
        username=USERNAME,
        password=PASSWORD,
        optional_args={
            "secret": SECRET,          # enable secret สำหรับ privileged EXEC
            "port": 22,
        },
    )

    print(f"[*] กำลังเชื่อมต่อไปยัง CORE-SW1 ({HOST}) ผ่าน NAPALM ...")
    device.open()

    # --- get_facts(): ข้อมูลพื้นฐานของอุปกรณ์ (structure เดียวกันทุกยี่ห้อ) ---
    facts = device.get_facts()
    print("\n=== get_facts() ===")
    print(json.dumps(facts, indent=2, ensure_ascii=False))

    # --- get_interfaces(): สถานะ interface ทั้งหมด (structured, ไม่ต้อง parse text) ---
    interfaces = device.get_interfaces()
    print("\n=== get_interfaces() (แสดงเฉพาะ 3 interface แรก) ===")
    for name, data in list(interfaces.items())[:3]:
        print(f"  {name}: is_up={data['is_up']}, is_enabled={data['is_enabled']}, "
              f"speed={data['speed']} Mbps, mac={data['mac_address']}")

    # --- get_interfaces_ip(): IP Address ของแต่ละ interface ---
    interfaces_ip = device.get_interfaces_ip()
    print("\n=== get_interfaces_ip() ===")
    for name, ip_data in interfaces_ip.items():
        for version, addrs in ip_data.items():
            for ip, meta in addrs.items():
                print(f"  {name} [{version}]: {ip}/{meta['prefix_length']}")

    device.close()
    print("\n[*] ปิดการเชื่อมต่อเรียบร้อย")


if __name__ == "__main__":
    main()
```

### ตัวอย่าง Output จริง (`get_facts()` แสดงเป็น JSON structure มาตรฐาน)

```
$ export NET_PASSWORD="StrongP@ssw0rd!" NET_SECRET="Cisco123!"
$ python3 napalm_get_facts.py
[*] กำลังเชื่อมต่อไปยัง CORE-SW1 (10.10.99.10) ผ่าน NAPALM ...

=== get_facts() ===
{
  "uptime": 5356800,
  "vendor": "Cisco",
  "model": "C9500-40X",
  "hostname": "CORE-SW1",
  "fqdn": "CORE-SW1.lab.local",
  "os_version": "17.09.04a",
  "serial_number": "FXS2145Q0AB",
  "interface_list": [
    "Vlan99", "Port-channel1", "TenGigabitEthernet1/0/1",
    "TenGigabitEthernet1/0/2", "Loopback0"
  ]
}

=== get_interfaces() (แสดงเฉพาะ 3 interface แรก) ===
  Vlan99: is_up=True, is_enabled=True, speed=1000 Mbps, mac=00:1a:2b:3c:63:01
  Port-channel1: is_up=True, is_enabled=True, speed=20000 Mbps, mac=00:1a:2b:3c:63:02
  TenGigabitEthernet1/0/1: is_up=True, is_enabled=True, speed=10000 Mbps, mac=00:1a:2b:3c:63:03

=== get_interfaces_ip() ===
  Vlan99 [ipv4]: 10.10.99.10/24
  Port-channel1 [ipv4]: 10.255.0.1/30
  Loopback0 [ipv4]: 1.1.1.1/32
```

สังเกตว่าไม่มีการเขียน regex หรือ split string เลย — NAPALM parse text output ของ Cisco IOS-XE
ให้เราภายใน driver แล้วคืนกลับมาเป็น dict ที่มี key ชื่อเดียวกันเสมอ (`is_up`, `speed`, `mac_address`)
ไม่ว่าอุปกรณ์จะเป็น Cisco หรือยี่ห้ออื่นที่มี driver รองรับ — นี่คือประโยชน์ที่แท้จริงของ Abstraction
Layer เมื่อ Enterprise มีอุปกรณ์หลายยี่ห้อปนกัน

> **ข้อจำกัดที่ต้องรู้**: NAPALM ไม่ได้ครอบคลุมทุกคำสั่งหรือทุก data ที่อุปกรณ์มี — มันมี getter
> method ที่กำหนดไว้ล่วงหน้าเท่านั้น (`get_facts`, `get_interfaces`, `get_bgp_neighbors`, `get_lldp_neighbors`,
> `get_config`, `get_arp_table` ฯลฯ) ถ้าต้องการข้อมูลที่ NAPALM ไม่มี getter ให้ ยังต้องกลับไปใช้
> `device.cli(["show some command"])` ซึ่งคืนค่าเป็น raw text เหมือน Netmiko เดิม

---

## Step 462 — Nornir Framework พื้นฐาน

### Nornir คืออะไร

**Nornir** คือ Python **Automation Framework** (ไม่ใช่ library เดี่ยวๆแบบ Netmiko/NAPALM) ที่
ออกแบบมาให้เป็น **"Ansible แต่เขียนด้วย Python ทั้งหมด"** — จุดเด่นคือ:

1. **Inventory System** แบบไฟล์ YAML (คล้าย Ansible Inventory ใน Part 23 Step 225) แยก Host/Group/Default
2. **Task-based** — เขียน function ที่รับ `task` object แล้วเรียกใช้ Netmiko/NAPALM/NETCONF ข้างในได้
3. **Concurrency ในตัว** — รัน task กับทุก host พร้อมกันด้วย Thread Pool โดย default (เร็วกว่า loop
   sequential ของ Step 223 ใน Part 23 มาก)
4. **Plugin Ecosystem** — `nornir_netmiko`, `nornir_napalm`, `nornir_utils` (สำหรับ print ผลลัพธ์สวยๆ)

```bash
pip install nornir nornir_netmiko nornir_napalm nornir_utils
```

### โครงสร้าง Inventory ของ Nornir

Nornir แยก Inventory เป็น 3 ไฟล์หลัก (คล้ายแนวคิด Ansible แต่ syntax ต่างกัน):

```
nornir-project/
├── config.yaml
├── inventory/
│   ├── hosts.yaml
│   ├── groups.yaml
│   └── defaults.yaml
└── run_show_version.py
```

**`config.yaml`** — บอก Nornir ว่า Inventory อยู่ที่ไฟล์ไหน และตั้งค่า Thread Pool

```yaml
---
inventory:
  plugin: SimpleInventory
  options:
    host_file: "inventory/hosts.yaml"
    group_file: "inventory/groups.yaml"
    defaults_file: "inventory/defaults.yaml"

runner:
  plugin: threaded
  options:
    num_workers: 10       # จำนวน thread สูงสุดที่รัน task พร้อมกัน
```

**`inventory/defaults.yaml`** — ค่า default ที่ทุก host ใช้ร่วมกัน (credential มาจาก Environment
Variable ตาม best practice เดียวกับ Part 23)

```yaml
---
username: "{{ env('NET_USERNAME') }}"
password: "{{ env('NET_PASSWORD') }}"
port: 22
connection_options:
  netmiko:
    extras:
      secret: "{{ env('NET_SECRET') }}"
      device_type: "cisco_ios"
```

**`inventory/groups.yaml`** — จัดกลุ่มอุปกรณ์ตาม Role (คล้าย `[core_switches]`/`[dist_switches]`
ใน Ansible)

```yaml
---
core_switches:
  data:
    role: core

dist_switches:
  data:
    role: distribution
```

**`inventory/hosts.yaml`** — รายชื่ออุปกรณ์ทั้งหมดใน Lab พร้อม IP (ตาม `docs/00-ip-address-plan.md`)

```yaml
---
CORE-SW1:
  hostname: 10.10.99.10
  groups: [core_switches]
  data:
    loopback0: 1.1.1.1
    model: "C9500-40X"

CORE-SW2:
  hostname: 10.10.99.11
  groups: [core_switches]
  data:
    loopback0: 1.1.1.2
    model: "C9500-40X"

DIST-SW1:
  hostname: 10.10.99.12
  groups: [dist_switches]
  data:
    loopback0: 1.1.1.11
    model: "C9300-48P"

DIST-SW2:
  hostname: 10.10.99.13
  groups: [dist_switches]
  data:
    loopback0: 1.1.1.12
    model: "C9300-48P"

DIST-SW3:
  hostname: 10.10.99.14
  groups: [dist_switches]
  data:
    loopback0: 1.1.1.13
    model: "C9300-48P"

DIST-SW4:
  hostname: 10.10.99.15
  groups: [dist_switches]
  data:
    loopback0: 1.1.1.14
    model: "C9300-48P"
```

### Script: รัน `show version` บนทุกอุปกรณ์พร้อมกันด้วย Nornir + Netmiko plugin

```python
#!/usr/bin/env python3
"""
run_show_version.py
ใช้ Nornir + nornir_netmiko เพื่อรัน 'show version' บนทุกอุปกรณ์ใน inventory
พร้อมกัน (concurrent) แทนการ loop ทีละตัวแบบ Netmiko เดิม (Part 23 Step 223)
"""

import time
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result


def main():
    nr = InitNornir(config_file="config.yaml")

    print(f"[*] จำนวนอุปกรณ์ใน Inventory: {len(nr.inventory.hosts)}")
    print("[*] กำลังรัน 'show version' บนทุกอุปกรณ์แบบ concurrent ...")

    start = time.time()
    result = nr.run(
        task=netmiko_send_command,
        command_string="show version",
    )
    elapsed = time.time() - start

    print_result(result)

    # --- สรุปผลลัพธ์: แยก host ที่สำเร็จ/ล้มเหลว ---
    failed_hosts = [name for name, r in result.items() if r.failed]
    ok_hosts = [name for name, r in result.items() if not r.failed]

    print("\n" + "=" * 60)
    print(f"เสร็จภายใน {elapsed:.2f} วินาที (รัน {len(nr.inventory.hosts)} อุปกรณ์แบบ concurrent)")
    print(f"สำเร็จ: {len(ok_hosts)} -> {', '.join(ok_hosts)}")
    if failed_hosts:
        print(f"ล้มเหลว: {len(failed_hosts)} -> {', '.join(failed_hosts)}")
    print("=" * 60)


if __name__ == "__main__":
    main()
```

### ตัวอย่าง Output จริง (เทียบเวลากับ Netmiko loop เดิม)

```
$ export NET_USERNAME=admin NET_PASSWORD='StrongP@ssw0rd!' NET_SECRET='Cisco123!'
$ python3 run_show_version.py
[*] จำนวนอุปกรณ์ใน Inventory: 6
[*] กำลังรัน 'show version' บนทุกอุปกรณ์แบบ concurrent ...

netmiko_send_command************************************************************
* CORE-SW1 ** changed : False *************************************************
vvvv netmiko_send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
Cisco IOS XE Software, Version 17.09.04a
... (ตัดให้สั้น) ...
^^^^ END netmiko_send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* CORE-SW2 ** changed : False *************************************************
...

============================================================
เสร็จภายใน 3.41 วินาที (รัน 6 อุปกรณ์แบบ concurrent)
สำเร็จ: 6 -> CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2, DIST-SW3, DIST-SW4
============================================================
```

### เทียบความเร็ว: Netmiko Loop (Sequential) vs Nornir (Concurrent)

| วิธีการ | 6 อุปกรณ์ (SSH connect + คำสั่งเดียว ~2.5 วิ/ตัว) | 50 อุปกรณ์ (สมมติ) |
|---|---|---|
| Netmiko `for` loop (Part 23 Step 223) — ทำทีละตัวเรียงกัน | ~15 วินาที (2.5 × 6) | ~125 วินาที |
| Nornir (`num_workers: 10`) — รันหลาย thread พร้อมกัน | **~3.4 วินาที** (จำกัดด้วยตัวที่ช้าสุด ไม่ใช่ผลรวม) | **~12-15 วินาที** (ประมาณ 5 รอบ ถ้า worker=10) |

ยิ่งจำนวนอุปกรณ์มากขึ้น ความแตกต่างของเวลาจะยิ่งชัดเจนขึ้นแบบทวีคูณ — นี่คือเหตุผลที่ Enterprise
ขนาดใหญ่ (อุปกรณ์หลักร้อยถึงหลักพัน) **ต้อง**ใช้ Framework ที่มี Concurrency ในตัวอย่าง Nornir
แทนการ loop แบบ sequential เพียวๆ (รายละเอียดกลไก Concurrency แบบ manual ด้วย `ThreadPoolExecutor`
จะอธิบายเพิ่มใน Step 467)

---

## Step 463 — Parsing CLI Output ด้วย TextFSM/ntc-templates

### ปัญหาของ Unstructured CLI Text

คำสั่ง `show ip interface brief` คืนค่าเป็น text ธรรมดาที่ **ออกแบบมาให้คนอ่าน** ไม่ใช่โปรแกรม
ถ้าต้องการดึงแค่ "รายชื่อ interface ที่ status เป็น down" ออกมา วิธีเดิมคือเขียน `.split()`
หรือ regex เอง ซึ่งเปราะบางมาก (เปลี่ยน IOS version นิดเดียว column อาจเลื่อน)

```
Interface              IP-Address      OK? Method Status                Protocol
Vlan99                 10.10.99.10     YES manual up                    up
Port-channel1          10.255.0.1      YES manual up                    up
TenGigabitEthernet1/0/1 unassigned     YES unset  down                  down
```

### TextFSM คืออะไร

**TextFSM** (สร้างโดย Google) คือ library ที่ parse text ตาม **Template ที่กำหนด Pattern ล่วงหน้า**
(ไฟล์ `.textfsm` เขียนด้วย regex + state machine) — สิ่งที่ทำให้ TextFSM ใช้งานง่ายมากคือ
community ชื่อ **`ntc-templates`** ได้เขียน Template สำหรับคำสั่ง `show` ที่ใช้บ่อยของหลายยี่ห้อ
ไว้ให้แล้วหลายพันไฟล์ — เราแค่เรียกใช้โดยไม่ต้องเขียน Template เองเลย

```bash
pip install textfsm ntc-templates
# ตั้ง Environment Variable ให้ TextFSM หา template ของ ntc-templates เจอ (บางระบบต้องตั้งเอง)
export NET_TEXTFSM=$(python3 -c "import ntc_templates; import os; print(os.path.dirname(ntc_templates.__file__)+'/templates')")
```

### Script เต็ม: Parse `show ip interface brief` ด้วย Netmiko + TextFSM

Netmiko รองรับ TextFSM ในตัวผ่าน parameter `use_textfsm=True` (เรียก ntc-templates ให้อัตโนมัติ)

```python
#!/usr/bin/env python3
"""
parse_ip_int_brief.py
รัน 'show ip interface brief' บน CORE-SW1 แล้ว parse ด้วย TextFSM/ntc-templates
ให้ได้ Python list of dict แทน raw text แบบเดิม
"""

from netmiko import ConnectHandler
import os
import sys

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

core_sw1 = {
    "device_type": "cisco_ios",
    "host": "10.10.99.10",
    "username": USERNAME,
    "password": PASSWORD,
    "secret": SECRET,
}


def print_table(rows, columns):
    """พิมพ์ list of dict ให้เป็นตารางอ่านง่าย"""
    widths = {c: max(len(c), *(len(str(r.get(c, ""))) for r in rows)) for c in columns}
    header = "  ".join(c.ljust(widths[c]) for c in columns)
    print(header)
    print("-" * len(header))
    for row in rows:
        print("  ".join(str(row.get(c, "")).ljust(widths[c]) for c in columns))


def main():
    connection = ConnectHandler(**core_sw1)
    connection.enable()

    # --- use_textfsm=True: Netmiko จะเรียก ntc-templates parse ให้อัตโนมัติ ---
    parsed = connection.send_command("show ip interface brief", use_textfsm=True)
    connection.disconnect()

    print(f"[*] TextFSM parse สำเร็จ ได้ {len(parsed)} interface record")
    print(f"[*] ตัวอย่าง record แรก (raw dict): {parsed[0]}\n")

    print("=== ตารางสรุป Interface Status (จาก Structured Data) ===")
    print_table(parsed, columns=["intf", "ipaddr", "status", "proto"])

    # --- ตัวอย่างการใช้งานจริง: หา interface ที่ down เพื่อแจ้งเตือน ---
    down_interfaces = [r for r in parsed if r["status"].lower() == "down"]
    print(f"\n[!] พบ {len(down_interfaces)} interface ที่สถานะ down:")
    for intf in down_interfaces:
        print(f"    - {intf['intf']}")


if __name__ == "__main__":
    main()
```

### ตัวอย่าง Output จริง

```
$ python3 parse_ip_int_brief.py
[*] TextFSM parse สำเร็จ ได้ 5 interface record
[*] ตัวอย่าง record แรก (raw dict): {'intf': 'Vlan99', 'ipaddr': '10.10.99.10', 'status': 'up', 'proto': 'up'}

=== ตารางสรุป Interface Status (จาก Structured Data) ===
intf                     ipaddr        status  proto
------------------------------------------------------
Vlan99                   10.10.99.10   up      up
Port-channel1            10.255.0.1    up      up
TenGigabitEthernet1/0/1  10.255.10.1   up      up
TenGigabitEthernet1/0/2  10.255.20.1   up      up
Loopback0                1.1.1.1       up      up

[!] พบ 0 interface ที่สถานะ down:
```

### ทำไม TextFSM ยังสำคัญ ในยุคที่มี NETCONF/RESTCONF แล้ว

| สถานการณ์ | ใช้ NETCONF/RESTCONF (Part 49) | ใช้ TextFSM/ntc-templates |
|---|---|---|
| อุปกรณ์รองรับ Model-Driven API (IOS-XE รุ่นใหม่) | ✅ ควรใช้ (structured แต่ต้น ไม่ต้อง parse) | ใช้ได้แต่ไม่จำเป็น |
| อุปกรณ์รุ่นเก่า/ไม่มี license เปิด NETCONF | ❌ ใช้ไม่ได้เลย | **✅ ทางเลือกเดียว** |
| ต้องดึงข้อมูลจากคำสั่ง `show` ที่ไม่มี YANG Model รองรับ | ❌ ไม่มี data model ให้ | **✅ ใช้ได้กับทุกคำสั่ง `show` ที่มี Template** |
| Multi-vendor เก่าที่หลากหลายมาก (Firewall, Load Balancer เก่า) | จำกัด | **✅ ntc-templates ครอบคลุมกว้างมาก** |

> ในงานจริง ทีม NetOps จำนวนมากยังต้องพึ่ง TextFSM เพราะอุปกรณ์เก่าจำนวนมากในองค์กรยังไม่รองรับ
> NETCONF/RESTCONF เต็มรูปแบบ — ทักษะนี้จึงยังจำเป็นแม้ในยุค Model-Driven API

---

## Step 464 — สร้าง Network Inventory/CMDB ของตัวเองด้วย Python

### แนวคิด: "DNAC-lite" ที่เราสร้างเอง

**CMDB (Configuration Management Database)** คือระบบที่เก็บข้อมูล Hardware/Software ของทุก
อุปกรณ์ในองค์กรไว้ที่เดียว — Cisco DNA Center (DNAC) ทำสิ่งนี้แบบ Enterprise-grade แต่เราสามารถ
สร้างเวอร์ชันง่ายๆของตัวเองได้ด้วย Python: SSH เข้าทุกอุปกรณ์, ดึงข้อมูล hostname/model/IOS
version/serial number ผ่าน TextFSM parsing ของ `show version`, แล้วบันทึกลงไฟล์ CSV/JSON
เพื่อใช้เป็น Source of Truth หรือส่งต่อให้ระบบ Asset Management อื่น

### Script เต็ม: `build_inventory.py`

```python
#!/usr/bin/env python3
"""
build_inventory.py
SSH เข้าอุปกรณ์ทุกตัวใน Lab (CORE-SW1/2, DIST-SW1-4), รัน 'show version' ผ่าน TextFSM,
ดึง hostname/model/version/serial แล้วบันทึกเป็น CSV และ JSON เพื่อใช้เป็น CMDB เบื้องต้น
"""

import csv
import json
import os
import sys
from datetime import datetime, timezone

from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

# รายชื่ออุปกรณ์ตาม docs/00-ip-address-plan.md
DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]


def collect_device_facts(device):
    """SSH เข้าอุปกรณ์ 1 ตัว, parse show version, คืนค่า dict ของ facts (หรือ None ถ้าล้มเหลว)"""
    conn_params = {
        "device_type": "cisco_ios",
        "host": device["host"],
        "username": USERNAME,
        "password": PASSWORD,
        "secret": SECRET,
    }
    try:
        connection = ConnectHandler(**conn_params)
        connection.enable()

        parsed = connection.send_command("show version", use_textfsm=True)
        uptime = connection.send_command("show version | include uptime")

        connection.disconnect()

        # ntc-templates คืนค่าเป็น list เสมอ (เผื่อกรณี multi-line match) — เอา record แรก
        facts = parsed[0] if isinstance(parsed, list) and parsed else {}

        return {
            "hostname": device["name"],
            "mgmt_ip": device["host"],
            "model": facts.get("hardware", ["unknown"])[0] if facts.get("hardware") else "unknown",
            "ios_version": facts.get("version", "unknown"),
            "serial_number": facts.get("serial", ["unknown"])[0] if facts.get("serial") else "unknown",
            "uptime": uptime.strip(),
            "collected_at": datetime.now(timezone.utc).isoformat(),
            "status": "reachable",
        }

    except NetmikoAuthenticationException:
        return {"hostname": device["name"], "mgmt_ip": device["host"],
                 "status": "auth_failed", "collected_at": datetime.now(timezone.utc).isoformat()}
    except NetmikoTimeoutException:
        return {"hostname": device["name"], "mgmt_ip": device["host"],
                 "status": "unreachable", "collected_at": datetime.now(timezone.utc).isoformat()}


def save_to_csv(records, filename="inventory.csv"):
    fieldnames = ["hostname", "mgmt_ip", "model", "ios_version", "serial_number",
                  "uptime", "status", "collected_at"]
    with open(filename, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
        writer.writeheader()
        for r in records:
            writer.writerow(r)


def save_to_json(records, filename="inventory.json"):
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(records, f, indent=2, ensure_ascii=False)


def main():
    print(f"[*] เริ่มเก็บ Inventory จาก {len(DEVICES)} อุปกรณ์ ...")
    records = []
    for device in DEVICES:
        print(f"    - กำลังดึงข้อมูล {device['name']} ({device['host']}) ...")
        facts = collect_device_facts(device)
        records.append(facts)

    save_to_csv(records)
    save_to_json(records)

    reachable = [r for r in records if r["status"] == "reachable"]
    print(f"\n[*] เก็บข้อมูลสำเร็จ {len(reachable)}/{len(DEVICES)} อุปกรณ์")
    print("[*] บันทึกไฟล์: inventory.csv, inventory.json")


if __name__ == "__main__":
    main()
```

### ตัวอย่างไฟล์ผลลัพธ์ `inventory.csv`

```csv
hostname,mgmt_ip,model,ios_version,serial_number,uptime,status,collected_at
CORE-SW1,10.10.99.10,C9500-40X,17.09.04a,FXS2145Q0AB,3 weeks,reachable,2026-09-26T02:00:00+00:00
CORE-SW2,10.10.99.11,C9500-40X,17.09.04a,FXS2145Q0AC,3 weeks,reachable,2026-09-26T02:00:03+00:00
DIST-SW1,10.10.99.12,C9300-48P,17.09.04a,FCW2140L0AA,3 weeks,reachable,2026-09-26T02:00:06+00:00
DIST-SW2,10.10.99.13,C9300-48P,17.09.04a,FCW2140L0AB,3 weeks,reachable,2026-09-26T02:00:09+00:00
DIST-SW3,10.10.99.14,,,,unreachable,2026-09-26T02:00:14+00:00
DIST-SW4,10.10.99.15,C9300-48P,17.09.04a,FCW2140L0AD,3 weeks,reachable,2026-09-26T02:00:17+00:00
```

ไฟล์ CSV/JSON นี้สามารถเปิดใน Excel, import เข้า Google Sheets, หรือใช้เป็น Data Source ให้
Dashboard อื่น (Grafana, PowerBI) ต่อได้ทันที — นี่คือรากฐานง่ายๆของ "CMDB ที่สร้างเอง" ก่อนที่จะ
ขยับไปใช้ระบบ Enterprise เช่น DNAC, NetBox, หรือ ServiceNow CMDB ในองค์กรขนาดใหญ่

---

## Step 465 — Configuration Compliance Checking ด้วย Python

### แนวคิด Golden Config

**Golden Config** คือ config มาตรฐานที่ทุกอุปกรณ์ Role เดียวกัน**ต้อง**มี (เช่น NTP Server, Syslog
Server, AAA settings ที่ตั้งไว้ตั้งแต่ [Part 17](part-017-ntp-syslog-snmp.md) และ
[Part 22](part-022-aaa-device-hardening.md)) — **Compliance Checking** คือกระบวนการเทียบ
running-config ของอุปกรณ์จริงกับ Golden Config เพื่อหา **Config Drift** (ความเบี่ยงเบนที่เกิดจาก
การแก้ไขมือแบบไม่ผ่าน process — ปัญหาที่ Part 23 Step 221 พูดถึงไว้)

### Golden Config Template (สิ่งที่ทุกอุปกรณ์ Core/Distribution ต้องมี)

```python
# golden_config.py
"""
Golden Config Rules — สิ่งที่ทุกอุปกรณ์ Core/Distribution ต้องมีในการ config
อ้างอิงจาก Part 17 (NTP/Syslog) และ Part 22 (AAA)
เขียนเป็น "regex pattern ที่ต้องเจอ" แทนการเทียบ string ตรงๆ เพื่อทนต่อความต่างเล็กน้อย
(เช่น ลำดับ argument, ตัวเลข instance)
"""

import re

GOLDEN_RULES = [
    {
        "id": "NTP-01",
        "category": "NTP",
        "description": "ต้องมี NTP server หลัก 10.10.99.50 พร้อม authentication key",
        "pattern": re.compile(r"^ntp server 10\.10\.99\.50 key \d+", re.MULTILINE),
        "severity": "high",
    },
    {
        "id": "SYSLOG-01",
        "category": "Syslog",
        "description": "ต้องส่ง log ไปยัง Syslog server 10.10.99.50",
        "pattern": re.compile(r"^logging host 10\.10\.99\.50", re.MULTILINE),
        "severity": "high",
    },
    {
        "id": "SYSLOG-02",
        "category": "Syslog",
        "description": "Logging trap level ต้องเป็น informational หรือ severe กว่า",
        "pattern": re.compile(r"^logging trap (emergencies|alerts|critical|errors|warnings|notifications|informational)", re.MULTILINE),
        "severity": "medium",
    },
    {
        "id": "AAA-01",
        "category": "AAA",
        "description": "ต้องเปิดใช้งาน aaa new-model",
        "pattern": re.compile(r"^aaa new-model", re.MULTILINE),
        "severity": "critical",
    },
    {
        "id": "AAA-02",
        "category": "AAA",
        "description": "Login ต้องใช้ AAA-TACACS group พร้อม local fallback",
        "pattern": re.compile(r"^aaa authentication login default group AAA-TACACS local", re.MULTILINE),
        "severity": "critical",
    },
    {
        "id": "AAA-03",
        "category": "AAA",
        "description": "Command authorization level 15 ต้อง log ผ่าน AAA",
        "pattern": re.compile(r"^aaa authorization commands 15 default group AAA-TACACS local", re.MULTILINE),
        "severity": "high",
    },
]
```

### Script เต็ม: `compliance_check.py`

```python
#!/usr/bin/env python3
"""
compliance_check.py
ดึง running-config จากทุกอุปกรณ์ แล้วเทียบกับ Golden Config Rules (golden_config.py)
พิมพ์รายงาน Compliance พร้อมระบุ Drift ที่พบ แยกตาม severity
"""

import os
import sys
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

from golden_config import GOLDEN_RULES

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]


def fetch_running_config(device):
    conn_params = {
        "device_type": "cisco_ios",
        "host": device["host"],
        "username": USERNAME,
        "password": PASSWORD,
        "secret": SECRET,
    }
    connection = ConnectHandler(**conn_params)
    connection.enable()
    config = connection.send_command("show running-config")
    connection.disconnect()
    return config


def check_compliance(running_config):
    """เทียบ running_config กับทุก rule ใน GOLDEN_RULES คืนค่า list ของผลลัพธ์แต่ละ rule"""
    results = []
    for rule in GOLDEN_RULES:
        matched = bool(rule["pattern"].search(running_config))
        results.append({
            "id": rule["id"],
            "category": rule["category"],
            "description": rule["description"],
            "severity": rule["severity"],
            "compliant": matched,
        })
    return results


def main():
    report = {}

    for device in DEVICES:
        print(f"[*] กำลังตรวจ Compliance ของ {device['name']} ({device['host']}) ...")
        try:
            config = fetch_running_config(device)
            report[device["name"]] = check_compliance(config)
        except (NetmikoAuthenticationException, NetmikoTimeoutException) as e:
            report[device["name"]] = None
            print(f"    [!] ไม่สามารถเชื่อมต่อได้: {e.__class__.__name__}")

    # --- พิมพ์รายงานสรุป ---
    print("\n" + "=" * 70)
    print("COMPLIANCE REPORT")
    print("=" * 70)

    total_violations = 0
    for hostname, results in report.items():
        if results is None:
            print(f"\n{hostname}: [SKIPPED - unreachable]")
            continue

        violations = [r for r in results if not r["compliant"]]
        total_violations += len(violations)
        status_icon = "✅ COMPLIANT" if not violations else f"❌ {len(violations)} VIOLATION(S)"
        print(f"\n{hostname}: {status_icon}")

        for r in results:
            mark = "✓" if r["compliant"] else "✗"
            print(f"    [{mark}] {r['id']:10s} ({r['severity']:8s}) - {r['description']}")

    print("\n" + "=" * 70)
    print(f"สรุปรวม: พบ Config Drift ทั้งหมด {total_violations} รายการ จาก {len(DEVICES)} อุปกรณ์")
    print("=" * 70)

    # exit code ที่ไม่เท่ากับ 0 เมื่อพบ violation — สำคัญมากสำหรับเอาไปต่อ CI/CD Pipeline ใน Part 48
    sys.exit(1 if total_violations > 0 else 0)


if __name__ == "__main__":
    main()
```

### ตัวอย่าง Output จริง (จำลอง DIST-SW2 ที่มีคนแก้ config มือ ลบ syslog host ทิ้งไป)

```
$ python3 compliance_check.py
[*] กำลังตรวจ Compliance ของ CORE-SW1 (10.10.99.10) ...
[*] กำลังตรวจ Compliance ของ CORE-SW2 (10.10.99.11) ...
[*] กำลังตรวจ Compliance ของ DIST-SW1 (10.10.99.12) ...
[*] กำลังตรวจ Compliance ของ DIST-SW2 (10.10.99.13) ...
[*] กำลังตรวจ Compliance ของ DIST-SW3 (10.10.99.14) ...
[*] กำลังตรวจ Compliance ของ DIST-SW4 (10.10.99.15) ...

======================================================================
COMPLIANCE REPORT
======================================================================

CORE-SW1: ✅ COMPLIANT
    [✓] NTP-01     (high    ) - ต้องมี NTP server หลัก 10.10.99.50 พร้อม authentication key
    [✓] SYSLOG-01  (high    ) - ต้องส่ง log ไปยัง Syslog server 10.10.99.50
    [✓] SYSLOG-02  (medium  ) - Logging trap level ต้องเป็น informational หรือ severe กว่า
    [✓] AAA-01     (critical) - ต้องเปิดใช้งาน aaa new-model
    [✓] AAA-02     (critical) - Login ต้องใช้ AAA-TACACS group พร้อม local fallback
    [✓] AAA-03     (high    ) - Command authorization level 15 ต้อง log ผ่าน AAA

DIST-SW2: ❌ 1 VIOLATION(S)
    [✓] NTP-01     (high    ) - ต้องมี NTP server หลัก 10.10.99.50 พร้อม authentication key
    [✗] SYSLOG-01  (high    ) - ต้องส่ง log ไปยัง Syslog server 10.10.99.50
    [✓] SYSLOG-02  (medium  ) - Logging trap level ต้องเป็น informational หรือ severe กว่า
    [✓] AAA-01     (critical) - ต้องเปิดใช้งาน aaa new-model
    [✓] AAA-02     (critical) - Login ต้องใช้ AAA-TACACS group พร้อม local fallback
    [✓] AAA-03     (high    ) - Command authorization level 15 ต้อง log ผ่าน AAA

======================================================================
สรุปรวม: พบ Config Drift ทั้งหมด 1 รายการ จาก 6 อุปกรณ์
======================================================================
```

Script นี้จบด้วย `sys.exit(1)` เมื่อพบ violation โดยตั้งใจ — ทำให้เอาไปต่อกับ Scheduled Job
(`cron`) หรือ CI/CD Pipeline (Part 48) ได้ทันที: ถ้า exit code ไม่เท่ากับ 0 ระบบสามารถส่ง alert
ไปยัง Slack/Email อัตโนมัติ โดยไม่ต้องมีคนมานั่งอ่าน report ทีละบรรทัด

---

## Step 466 — Error Handling และ Retry Logic สำหรับ Production Script

### ทำไม try/except แบบธรรมดาไม่พอสำหรับ Production

Script ใน Part 23 (Step 223) มี error handling แบบพื้นฐาน (`try/except` แยก Exception type) —
แต่ในสภาพแวดล้อมจริง ปัญหาการเชื่อมต่อ (network glitch ชั่วคราว, SSH server กำลัง busy) มักเป็น
**ปัญหาชั่วคราว (transient)** ที่ถ้า**ลองใหม่อีกครั้งหลังรอสักพัก**ก็มักจะสำเร็จ — การยอมแพ้ทันที
ตั้งแต่ครั้งแรกจึงไม่เหมาะกับ Production Script ที่ต้องรันแบบไม่มีคนเฝ้า (unattended)

### Retry Decorator พร้อม Exponential Backoff

**Exponential Backoff** คือเทคนิคการเพิ่มเวลารอระหว่าง retry แบบทวีคูณ (เช่น 1s → 2s → 4s → 8s)
เพื่อไม่ยิง request ซ้ำถี่เกินไปตอนอุปกรณ์/network กำลังมีปัญหาอยู่แล้ว (ซึ่งจะทำให้แย่ลงไปอีก)

```python
#!/usr/bin/env python3
"""
retry_utils.py
Decorator สำหรับ retry function ที่อาจ fail แบบชั่วคราว (transient failure)
พร้อม Exponential Backoff และ Logging ไปยังไฟล์แทนการ print() ตรงๆ
"""

import functools
import logging
import time

# --- ตั้งค่า Logging ให้เขียนลงไฟล์ พร้อม format ที่มี timestamp + level ---
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[
        logging.FileHandler("automation.log"),
        logging.StreamHandler(),   # ยังแสดงบนหน้าจอด้วย ไม่ใช่แค่เขียนไฟล์
    ],
)
logger = logging.getLogger("netops")


def retry_with_backoff(max_attempts=4, base_delay=1, backoff_factor=2, retry_exceptions=(Exception,)):
    """
    Decorator: retry function ที่ decorate เมื่อเจอ exception ใน retry_exceptions
    - max_attempts: จำนวนครั้งสูงสุดที่จะลอง (รวมครั้งแรก)
    - base_delay: เวลารอครั้งแรก (วินาที)
    - backoff_factor: ตัวคูณเวลารอในแต่ละรอบ (2 = double ทุกครั้ง)
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            delay = base_delay
            last_exception = None

            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except retry_exceptions as e:
                    last_exception = e
                    logger.warning(
                        f"{func.__name__} ล้มเหลวครั้งที่ {attempt}/{max_attempts}: "
                        f"{e.__class__.__name__}: {e}"
                    )
                    if attempt < max_attempts:
                        logger.info(f"รอ {delay} วินาทีก่อนลองใหม่ ...")
                        time.sleep(delay)
                        delay *= backoff_factor
            logger.error(f"{func.__name__} ล้มเหลวครบ {max_attempts} ครั้ง ยอมแพ้")
            raise last_exception

        return wrapper
    return decorator
```

### เวอร์ชัน "Hardened" ของ Push Config Script จาก Part 23 Step 223

```python
#!/usr/bin/env python3
"""
push_ntp_config_hardened.py
เวอร์ชัน hardened ของ Part 23 Step 223 — ใช้ retry decorator + logging ไปไฟล์
แทน print() และแยก error ตามประเภทเพื่อตัดสินใจว่าควร retry หรือไม่
"""

import os
import sys

from netmiko import ConnectHandler
from netmiko.exceptions import (
    NetmikoTimeoutException,
    NetmikoAuthenticationException,
)

from retry_utils import retry_with_backoff, logger

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD or not SECRET:
    sys.exit("[!] กรุณา export NET_PASSWORD และ NET_SECRET ก่อนรัน script นี้")

DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]

CONFIG_COMMANDS = [
    "ntp server 10.10.99.50 prefer",
    "ntp server 10.10.99.51",
]


# --- สำคัญ: retry เฉพาะ NetmikoTimeoutException (ปัญหาชั่วคราว เช่น network glitch) ---
# ไม่ retry NetmikoAuthenticationException เพราะรหัสผ่านผิดจะผิดซ้ำทุกครั้งไม่มีประโยชน์
@retry_with_backoff(
    max_attempts=4,
    base_delay=2,
    backoff_factor=2,
    retry_exceptions=(NetmikoTimeoutException,),
)
def connect_and_push(device):
    conn_params = {
        "device_type": "cisco_ios",
        "host": device["host"],
        "username": USERNAME,
        "password": PASSWORD,
        "secret": SECRET,
        "conn_timeout": 10,
        "auth_timeout": 10,
    }
    connection = ConnectHandler(**conn_params)
    connection.enable()
    output = connection.send_config_set(CONFIG_COMMANDS)
    connection.save_config()
    connection.disconnect()
    return output


def main():
    results = {"success": [], "failed": []}

    for device in DEVICES:
        logger.info(f"เริ่ม push config ไปยัง {device['name']} ({device['host']})")
        try:
            connect_and_push(device)
            logger.info(f"{device['name']}: push config สำเร็จ")
            results["success"].append(device["name"])

        except NetmikoAuthenticationException:
            # ไม่ retry — ต้องแจ้ง engineer ทันทีให้มาตรวจ credential
            logger.error(f"{device['name']}: Authentication ล้มเหลว (ไม่ retry เพราะไม่ใช่ปัญหาชั่วคราว)")
            results["failed"].append((device["name"], "auth_failed"))

        except NetmikoTimeoutException:
            # retry ครบ max_attempts แล้วยังไม่สำเร็จ
            logger.error(f"{device['name']}: Timeout หลัง retry ครบทุกครั้งแล้ว")
            results["failed"].append((device["name"], "timeout_after_retry"))

        except Exception as e:
            logger.exception(f"{device['name']}: เกิด error ที่ไม่คาดคิด")
            results["failed"].append((device["name"], str(e)))

    logger.info(
        f"สรุปผล: สำเร็จ {len(results['success'])}/{len(DEVICES)}, "
        f"ล้มเหลว {len(results['failed'])}"
    )
    if results["failed"]:
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### ตัวอย่างเนื้อหาไฟล์ log (`automation.log`) หลังรัน — เห็น Retry ทำงานจริง

```
2026-09-26 09:15:02 [INFO] netops: เริ่ม push config ไปยัง CORE-SW1 (10.10.99.10)
2026-09-26 09:15:04 [INFO] netops: CORE-SW1: push config สำเร็จ
2026-09-26 09:15:04 [INFO] netops: เริ่ม push config ไปยัง DIST-SW3 (10.10.99.14)
2026-09-26 09:15:14 [WARNING] netops: connect_and_push ล้มเหลวครั้งที่ 1/4: NetmikoTimeoutException: ...
2026-09-26 09:15:14 [INFO] netops: รอ 2 วินาทีก่อนลองใหม่ ...
2026-09-26 09:15:26 [WARNING] netops: connect_and_push ล้มเหลวครั้งที่ 2/4: NetmikoTimeoutException: ...
2026-09-26 09:15:26 [INFO] netops: รอ 4 วินาทีก่อนลองใหม่ ...
2026-09-26 09:15:34 [INFO] netops: DIST-SW3: push config สำเร็จ
2026-09-26 09:15:34 [INFO] netops: สรุปผล: สำเร็จ 6/6, ล้มเหลว 0
```

DIST-SW3 ในตัวอย่างนี้เชื่อมต่อไม่ได้ 2 ครั้งแรก (อาจเป็นเพราะ SSH session pool เต็มชั่วคราว)
แต่ retry ครั้งที่ 3 สำเร็จ — ถ้าใช้ script แบบเดิมที่ไม่มี retry logic (Part 23) อุปกรณ์นี้จะถูก
รายงานว่า "ล้มเหลว" ทั้งที่จริงๆแค่ต้องรอสักพัก

---

## Step 467 — Parallel Execution ด้วย `concurrent.futures.ThreadPoolExecutor`

### ทำไมต้องรู้กลไก Threading เอง ทั้งที่ Nornir ทำให้แล้ว (Step 462)

Nornir สะดวกมากสำหรับงานทั่วไป แต่บางครั้งเราต้องการควบคุม Concurrency แบบละเอียดกว่านั้น
(เช่น จำกัดจำนวน worker แบบไดนามิก, รวม progress bar, จัดการ error แบบ custom) หรือใช้ในโปรเจกต์
ที่ไม่อยาก add dependency ของ Nornir ทั้ง framework — Python มี `concurrent.futures.ThreadPoolExecutor`
ในตัว (standard library ไม่ต้องติดตั้งเพิ่ม) ที่ทำสิ่งเดียวกันได้โดยตรง

> **ทำไมใช้ Thread แทน Process สำหรับงาน Network Automation**: งาน SSH/HTTP เป็น **I/O-bound**
> (เวลาส่วนใหญ่ใช้ไปกับการ**รอ**เครือข่ายตอบกลับ ไม่ใช่การประมวลผล CPU) Python's Global Interpreter
> Lock (GIL) จะถูกปลดปล่อยระหว่างรอ I/O ทำให้ `ThreadPoolExecutor` มีประสิทธิภาพดีมากสำหรับงาน
> ประเภทนี้ ต่างจากงาน CPU-bound (เช่น คำนวณตัวเลขหนักๆ) ที่ต้องใช้ `ProcessPoolExecutor` แทน

### Script เต็ม: เวอร์ชัน Parallel ของ Step 464 (Inventory Collection)

```python
#!/usr/bin/env python3
"""
build_inventory_parallel.py
เวอร์ชัน parallel ของ Step 464 (build_inventory.py) — ใช้ ThreadPoolExecutor
เพื่อ SSH เข้าทุกอุปกรณ์พร้อมกันแทนการ loop ทีละตัว พร้อมวัดเวลาเทียบกันให้เห็นชัด
"""

import csv
import json
import os
import sys
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime, timezone

from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]

MAX_WORKERS = 6   # จำกัดจำนวน thread สูงสุด — ไม่ควรเยอะเกินไปจนอุปกรณ์/SSH server รับไม่ไหว


def collect_device_facts(device):
    """ฟังก์ชันเดียวกับ Step 464 — SSH เข้า 1 อุปกรณ์ คืนค่า dict of facts"""
    conn_params = {
        "device_type": "cisco_ios",
        "host": device["host"],
        "username": USERNAME,
        "password": PASSWORD,
        "secret": SECRET,
        "conn_timeout": 10,
    }
    try:
        connection = ConnectHandler(**conn_params)
        connection.enable()
        parsed = connection.send_command("show version", use_textfsm=True)
        connection.disconnect()

        facts = parsed[0] if isinstance(parsed, list) and parsed else {}
        return {
            "hostname": device["name"],
            "mgmt_ip": device["host"],
            "model": facts.get("hardware", ["unknown"])[0] if facts.get("hardware") else "unknown",
            "ios_version": facts.get("version", "unknown"),
            "serial_number": facts.get("serial", ["unknown"])[0] if facts.get("serial") else "unknown",
            "collected_at": datetime.now(timezone.utc).isoformat(),
            "status": "reachable",
        }
    except (NetmikoAuthenticationException, NetmikoTimeoutException) as e:
        return {
            "hostname": device["name"], "mgmt_ip": device["host"],
            "status": f"failed: {e.__class__.__name__}",
            "collected_at": datetime.now(timezone.utc).isoformat(),
        }


def run_sequential():
    """เวอร์ชัน loop ธรรมดา (เหมือน Step 464 เดิม) — ใช้เทียบเวลา"""
    start = time.time()
    records = [collect_device_facts(d) for d in DEVICES]
    return records, time.time() - start


def run_parallel():
    """เวอร์ชัน ThreadPoolExecutor — ยิง task ทุกอุปกรณ์เข้า pool พร้อมกัน"""
    start = time.time()
    records = []

    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        # submit() คืนค่า Future object ทันที ไม่รอผลลัพธ์ — ส่ง task ทั้งหมดเข้า pool ก่อน
        future_to_device = {
            executor.submit(collect_device_facts, device): device
            for device in DEVICES
        }

        # as_completed() คืนค่า Future ทีละตัวตามลำดับที่ "เสร็จก่อน" ไม่ใช่ลำดับที่ submit
        for future in as_completed(future_to_device):
            device = future_to_device[future]
            try:
                result = future.result()
                records.append(result)
                print(f"    [เสร็จ] {device['name']}")
            except Exception as e:
                print(f"    [ERROR] {device['name']}: {e}")
                records.append({"hostname": device["name"], "status": f"exception: {e}"})

    return records, time.time() - start


def main():
    mode = sys.argv[1] if len(sys.argv) > 1 else "parallel"

    if mode == "sequential":
        print("[*] รันแบบ Sequential (loop ทีละตัว) ...")
        records, elapsed = run_sequential()
    else:
        print(f"[*] รันแบบ Parallel (ThreadPoolExecutor, max_workers={MAX_WORKERS}) ...")
        records, elapsed = run_parallel()

    with open("inventory_parallel.json", "w", encoding="utf-8") as f:
        json.dump(records, f, indent=2, ensure_ascii=False)

    reachable = [r for r in records if r["status"] == "reachable"]
    print(f"\n[*] เก็บข้อมูลสำเร็จ {len(reachable)}/{len(DEVICES)} อุปกรณ์")
    print(f"[*] ใช้เวลาทั้งหมด: {elapsed:.2f} วินาที (mode={mode})")


if __name__ == "__main__":
    main()
```

### ผลลัพธ์เทียบเวลาจริง (สมมติแต่ละ SSH connection ใช้เวลาเฉลี่ย ~2.3 วินาที)

```
$ python3 build_inventory_parallel.py sequential
[*] รันแบบ Sequential (loop ทีละตัว) ...

[*] เก็บข้อมูลสำเร็จ 6/6 อุปกรณ์
[*] ใช้เวลาทั้งหมด: 13.87 วินาที (mode=sequential)

$ python3 build_inventory_parallel.py parallel
[*] รันแบบ Parallel (ThreadPoolExecutor, max_workers=6) ...
    [เสร็จ] CORE-SW2
    [เสร็จ] CORE-SW1
    [เสร็จ] DIST-SW1
    [เสร็จ] DIST-SW4
    [เสร็จ] DIST-SW2
    [เสร็จ] DIST-SW3

[*] เก็บข้อมูลสำเร็จ 6/6 อุปกรณ์
[*] ใช้เวลาทั้งหมด: 2.41 วินาที (mode=parallel)
```

**เร็วขึ้นประมาณ 5.7 เท่า** ในกรณี 6 อุปกรณ์ — สังเกตว่าลำดับที่พิมพ์ "[เสร็จ]" ไม่ตรงกับลำดับใน
`DEVICES` list เลย (CORE-SW2 เสร็จก่อน CORE-SW1) เพราะ `as_completed()` คืนผลตามลำดับที่งาน
เสร็จจริง ไม่ใช่ลำดับที่ submit — เป็นพฤติกรรมปกติของ concurrent execution ที่ต้องเข้าใจ (ถ้าต้อง
เก็บผลลัพธ์ตามลำดับเดิม ให้ใช้ dict ที่ map ด้วย key ของอุปกรณ์ ไม่ใช่ list เรียงตามลำดับที่ได้กลับมา)

> **ข้อควรระวัง**: การเพิ่ม `max_workers` มากเกินไปไม่ได้แปลว่าเร็วขึ้นเสมอ — อุปกรณ์ Cisco มี
> ขีดจำกัดจำนวน VTY session พร้อมกัน (`line vty 0 15` = สูงสุด 16 session, ดู Part 2) ถ้ายิง
> parallel เกิน VTY ที่เหลือของอุปกรณ์ตัวเดียวกัน (กรณีดึงหลายคำสั่งจากอุปกรณ์เดิม) จะเกิด
> connection refused — ควรตั้ง `max_workers` ให้สัมพันธ์กับ VTY line ที่มีอยู่และ CPU ของ Control
> Node ที่รัน script ด้วย

---

## Step 468 — Git Integration สำหรับ Config Version Control

### ทำไม Config Backup ต้องมี Version Control

[Part 17](part-017-ntp-syslog-snmp.md) แนะนำให้ทำ config backup อัตโนมัติเป็นนิสัย — แต่การ
backup แค่ **overwrite ไฟล์เดิมทุกวัน** ทำให้เราไม่รู้ว่า **อะไรเปลี่ยนไปเมื่อไหร่** ถ้าใช้ **Git**
เก็บ backup แต่ละวันเป็น commit แยกกัน เราจะได้ **Full History + Diff Capability** ทันที — เห็นว่า
บรรทัดไหนถูกเพิ่ม/ลบ/แก้ ระหว่างวันที่ A กับวันที่ B ได้ทันทีด้วยคำสั่ง `git diff`

### Script เต็ม: `git_backup.py`

```python
#!/usr/bin/env python3
"""
git_backup.py
ดึง running-config จากทุกอุปกรณ์ บันทึกเป็นไฟล์ .cfg แยกทีละอุปกรณ์
แล้ว commit เข้า local Git repository พร้อม timestamp — ให้ history/diff แบบเต็มรูปแบบ
ใช้ subprocess เรียก git CLI ตรงๆ (ทางเลือกอื่นคือ library 'GitPython': pip install gitpython)
"""

import os
import subprocess
import sys
from datetime import datetime, timezone

from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD:
    sys.exit("[!] กรุณา export NET_PASSWORD ก่อนรัน script นี้")

DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]

BACKUP_DIR = "config-backups"


def run_git(*args, cwd=BACKUP_DIR):
    """helper เรียก git CLI ผ่าน subprocess คืนค่า (returncode, stdout)"""
    result = subprocess.run(
        ["git", *args],
        cwd=cwd,
        capture_output=True,
        text=True,
    )
    return result.returncode, result.stdout.strip(), result.stderr.strip()


def ensure_repo_initialized():
    """สร้าง backup directory + git init ถ้ายังไม่มี (idempotent — รันซ้ำได้ไม่พัง)"""
    os.makedirs(BACKUP_DIR, exist_ok=True)
    if not os.path.isdir(os.path.join(BACKUP_DIR, ".git")):
        print("[*] ยังไม่มี Git repository — กำลังสร้างใหม่ (git init) ...")
        run_git("init")
        run_git("config", "user.name", "NetOps Automation Bot")
        run_git("config", "user.email", "netops-bot@lab.local")


def backup_device_config(device):
    conn_params = {
        "device_type": "cisco_ios",
        "host": device["host"],
        "username": USERNAME,
        "password": PASSWORD,
        "secret": SECRET,
    }
    connection = ConnectHandler(**conn_params)
    connection.enable()
    config = connection.send_command("show running-config")
    connection.disconnect()

    filepath = os.path.join(BACKUP_DIR, f"{device['name']}.cfg")
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(config)
    return filepath


def main():
    ensure_repo_initialized()

    backed_up = []
    failed = []

    for device in DEVICES:
        print(f"[*] กำลัง backup {device['name']} ({device['host']}) ...")
        try:
            backup_device_config(device)
            backed_up.append(device["name"])
        except (NetmikoAuthenticationException, NetmikoTimeoutException) as e:
            print(f"    [!] ล้มเหลว: {e.__class__.__name__}")
            failed.append(device["name"])

    # --- git add ทุกไฟล์ .cfg ที่เปลี่ยน ---
    run_git("add", "-A")

    # --- ตรวจว่ามีอะไรเปลี่ยนจริงหรือไม่ (git diff --cached --quiet คืน non-zero ถ้ามี change) ---
    returncode, _, _ = run_git("diff", "--cached", "--quiet")

    if returncode == 0:
        print("\n[*] ไม่มี config เปลี่ยนแปลงจากการ backup รอบก่อน — ไม่สร้าง commit ใหม่ (idempotent)")
    else:
        timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")
        commit_message = (
            f"Automated config backup - {timestamp}\n\n"
            f"Backed up: {', '.join(backed_up)}\n"
            f"Failed: {', '.join(failed) if failed else 'none'}"
        )
        run_git("commit", "-m", commit_message)
        print(f"\n[*] สร้าง commit ใหม่สำเร็จ: {timestamp}")

    print(f"[*] Backup เสร็จสิ้น: สำเร็จ {len(backed_up)}, ล้มเหลว {len(failed)}")


if __name__ == "__main__":
    main()
```

### ตัวอย่างการใช้ Git ดู History/Diff หลังรัน Script มาหลายวัน

```bash
# ดู commit history ทั้งหมด (แต่ละวันคือ backup แยกกัน)
$ cd config-backups && git log --oneline
a3f9c21 Automated config backup - 2026-09-26 02:00:03 UTC
7e1d004 Automated config backup - 2026-09-25 02:00:05 UTC
f52b8aa Automated config backup - 2026-09-24 02:00:02 UTC

# ดูว่า DIST-SW2 เปลี่ยนอะไรไปบ้างระหว่างวันที่ 24 กับ 26 (พบว่ามีคนลบ logging host ทิ้ง!)
$ git diff f52b8aa a3f9c21 -- DIST-SW2.cfg
diff --git a/DIST-SW2.cfg b/DIST-SW2.cfg
index 8a3f1e2..c91b7d4 100644
--- a/DIST-SW2.cfg
+++ b/DIST-SW2.cfg
@@ -45,8 +45,6 @@
 ntp server 10.10.99.50 key 1
 ntp authentication-key 1 md5 7 105E1A1A0E
 !
-logging host 10.10.99.50
-logging trap informational
-!
 aaa new-model
```

นี่คือ**เหตุผลที่แท้จริง**ของการรวม Config Backup เข้ากับ Git — เราสามารถบอกได้ทันทีว่า
**"ใครลบ syslog config ของ DIST-SW2 ออกไประหว่างวันที่ 24-26 กันยายน"** ซึ่งตรงกับ Config Drift
ที่ Compliance Checker ใน Step 465 ตรวจพบพอดี — Git ให้เราเห็น **เวลาและรายละเอียดของการเปลี่ยนแปลง**
ในขณะที่ Compliance Checker ให้เราเห็น **สถานะปัจจุบัน** — ทั้งสองเครื่องมือทำงานเสริมกัน

> **งานจริงระดับ Enterprise**: มักผลัก backup เหล่านี้ขึ้นไปยัง Git Server กลาง (GitLab/GitHub
> Enterprise/Bitbucket) ด้วย `git push` เพิ่มอีกขั้น เพื่อให้ทีมอื่นเข้าถึง history ได้ และเปิด
> โอกาสให้ทำ Pull Request Review ก่อน apply config จริง (Network-as-Code เต็มรูปแบบ — เรียนต่อ
> ใน Part 48)

---

## Step 469 — สร้าง REST API ด้วย Flask ครอบ Automation Script

### จาก Ad-hoc Script สู่ Internal API

Script ที่เขียนมาทั้งหมดใน Part นี้ต้อง**รันจาก command line** — ถ้าทีมอื่น (เช่น ทีม Monitoring
Dashboard, ทีม ChatOps บน Slack) ต้องการเรียกใช้ข้อมูลเดียวกัน พวกเขาต้องมาเรียนรู้วิธีรัน Python
script ของเรา ซึ่งไม่สะดวก — การครอบ Script ด้วย **REST API** ทำให้ระบบอื่นสามารถเรียกใช้งานผ่าน
HTTP request ธรรมดา (`GET`, `POST`) โดยไม่ต้องรู้เลยว่าข้างในใช้ Netmiko หรือ NAPALM

**Flask** คือ Python web framework แบบ minimal ที่เหมาะมากสำหรับสร้าง Internal API ขนาดเล็ก-กลาง
(สำหรับ API ขนาดใหญ่ระดับ production อาจพิจารณา FastAPI ที่มี async + auto-generated docs
ในตัว แต่ Flask เรียนรู้ง่ายกว่าและเพียงพอสำหรับ use case ส่วนใหญ่ของทีม NetOps)

```bash
pip install flask
```

### Script เต็ม: `app.py` — Flask API ครอบ Netmiko

```python
#!/usr/bin/env python3
"""
app.py
Flask REST API ขั้นต่ำที่ครอบ Netmiko — เปิด endpoint ให้ระบบอื่นเรียกดูข้อมูล interface
ของอุปกรณ์ผ่าน HTTP โดยไม่ต้องรู้เรื่อง SSH/Netmiko เลย

รันด้วย: flask --app app run --host 0.0.0.0 --port 5000
"""

import os
from flask import Flask, jsonify, request
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

app = Flask(__name__)

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

# --- Device Registry: hostname -> mgmt IP (ในระบบจริงควรดึงจาก inventory.json ของ Step 464) ---
DEVICE_REGISTRY = {
    "CORE-SW1": "10.10.99.10",
    "CORE-SW2": "10.10.99.11",
    "DIST-SW1": "10.10.99.12",
    "DIST-SW2": "10.10.99.13",
    "DIST-SW3": "10.10.99.14",
    "DIST-SW4": "10.10.99.15",
}


def get_connection(hostname):
    """helper: เปิด Netmiko connection ไปยัง hostname ที่รู้จักใน DEVICE_REGISTRY"""
    if hostname not in DEVICE_REGISTRY:
        return None
    return ConnectHandler(
        device_type="cisco_ios",
        host=DEVICE_REGISTRY[hostname],
        username=USERNAME,
        password=PASSWORD,
        secret=SECRET,
    )


@app.route("/health", methods=["GET"])
def health_check():
    """endpoint ตรวจสอบว่า API service ยังทำงานอยู่ (ใช้กับ Load Balancer health check ได้)"""
    return jsonify({"status": "ok", "devices_known": len(DEVICE_REGISTRY)})


@app.route("/device/<hostname>/interfaces", methods=["GET"])
def get_interfaces(hostname):
    """
    GET /device/CORE-SW1/interfaces
    คืนค่า interface status ของอุปกรณ์เป็น JSON (parse ด้วย TextFSM ข้างในให้)
    """
    if hostname not in DEVICE_REGISTRY:
        return jsonify({"error": f"ไม่รู้จักอุปกรณ์ '{hostname}'"}), 404

    try:
        connection = get_connection(hostname)
        connection.enable()
        parsed = connection.send_command("show ip interface brief", use_textfsm=True)
        connection.disconnect()

        return jsonify({
            "hostname": hostname,
            "interface_count": len(parsed),
            "interfaces": parsed,
        })

    except NetmikoAuthenticationException:
        return jsonify({"error": "Authentication ล้มเหลว"}), 502
    except NetmikoTimeoutException:
        return jsonify({"error": f"เชื่อมต่อ {hostname} ไม่ได้ (timeout)"}), 504


@app.route("/device/<hostname>/config", methods=["POST"])
def push_config(hostname):
    """
    POST /device/CORE-SW1/config
    Body (JSON): {"commands": ["ntp server 10.10.99.52"]}
    ส่งคำสั่ง config มาจาก client แล้ว push ผ่าน Netmiko — endpoint นี้ต้องมี Authentication
    ป้องกันเพิ่มในงานจริง (ตัวอย่างนี้ตัด Auth ออกเพื่อความกระชับ ดูหมายเหตุด้านล่าง)
    """
    if hostname not in DEVICE_REGISTRY:
        return jsonify({"error": f"ไม่รู้จักอุปกรณ์ '{hostname}'"}), 404

    body = request.get_json(silent=True) or {}
    commands = body.get("commands")
    if not commands or not isinstance(commands, list):
        return jsonify({"error": "ต้องส่ง JSON body แบบ {'commands': [...]}"}), 400

    try:
        connection = get_connection(hostname)
        connection.enable()
        output = connection.send_config_set(commands)
        connection.save_config()
        connection.disconnect()

        return jsonify({
            "hostname": hostname,
            "commands_applied": commands,
            "raw_output": output,
        })

    except NetmikoAuthenticationException:
        return jsonify({"error": "Authentication ล้มเหลว"}), 502
    except NetmikoTimeoutException:
        return jsonify({"error": f"เชื่อมต่อ {hostname} ไม่ได้ (timeout)"}), 504


if __name__ == "__main__":
    # โหมด dev เท่านั้น — งานจริงต้องรันผ่าน WSGI server เช่น gunicorn (ดูหมายเหตุด้านล่าง)
    app.run(host="0.0.0.0", port=5000, debug=False)
```

### ทดสอบ API ด้วย `curl`

```bash
$ export NET_USERNAME=admin NET_PASSWORD='StrongP@ssw0rd!' NET_SECRET='Cisco123!'
$ flask --app app run --host 0.0.0.0 --port 5000 &

$ curl http://localhost:5000/health
{"status": "ok", "devices_known": 6}

$ curl http://localhost:5000/device/CORE-SW1/interfaces
{
  "hostname": "CORE-SW1",
  "interface_count": 5,
  "interfaces": [
    {"intf": "Vlan99", "ipaddr": "10.10.99.10", "status": "up", "proto": "up"},
    {"intf": "Port-channel1", "ipaddr": "10.255.0.1", "status": "up", "proto": "up"}
  ]
}

$ curl -X POST http://localhost:5000/device/DIST-SW1/config \
  -H "Content-Type: application/json" \
  -d '{"commands": ["ntp server 10.10.99.52"]}'
{
  "hostname": "DIST-SW1",
  "commands_applied": ["ntp server 10.10.99.52"],
  "raw_output": "config term\n...\nntp server 10.10.99.52\n...\nend"
}
```

> **คำเตือนด้าน Security (สำคัญมาก)**: API ตัวอย่างข้างบนนี้**ไม่มี Authentication** และรับคำสั่ง
> config ตรงๆจาก request body — **ห้ามนำไป deploy ในระบบจริงแบบนี้เด็ดขาด** งานจริงต้องเพิ่ม
> อย่างน้อย: (1) API Key หรือ OAuth2 Token ตรวจสอบทุก request, (2) รันผ่าน HTTPS เท่านั้น (ไม่ใช่
> HTTP), (3) จำกัดคำสั่งที่ endpoint `/config` รับได้ด้วย Whitelist/Schema Validation ไม่ใช่รับ
> คำสั่งอิสระ, (4) รันผ่าน production WSGI server เช่น `gunicorn` แทน Flask dev server
> (`app.run()` ไม่ได้ออกแบบมาให้รับ traffic จริงจัง), และ (5) ทำ Rate Limiting ป้องกันการยิง
> request รัวๆจน SSH session ของอุปกรณ์เต็ม

---

## Step 470 — Lab เต็มรูปแบบ: "NetOps Toolkit"

### ภาพรวมโปรเจกต์

Lab สุดท้ายของ Part นี้รวมทุกเทคนิคจาก Step 461-469 เข้าเป็นโปรเจกต์เดียวที่รันได้จริง:
**Nornir Inventory** (Step 462) + **TextFSM Parsing** (Step 463) + **Concurrent Execution**
(Step 467) + **Git-versioned Backup** (Step 468) + **Flask API Endpoint** (Step 469)

### โครงสร้างไฟล์ทั้งหมด

```
netops-toolkit/
├── requirements.txt
├── config.yaml                      # Nornir config
├── inventory/
│   ├── hosts.yaml
│   ├── groups.yaml
│   └── defaults.yaml
├── golden_config.py                 # จาก Step 465
├── collectors.py                    # ฟังก์ชันรวม: parse + backup
├── run_backup.py                    # สคริปต์หลัก: เก็บ config ทุกอุปกรณ์ + commit Git
├── api/
│   └── app.py                       # Flask API endpoint
└── config-backups/                  # (Git repo แยก สร้างอัตโนมัติตอนรันครั้งแรก)
```

### `requirements.txt`

```
nornir==3.4.1
nornir-netmiko==1.0.1
nornir-utils==0.2.0
netmiko==4.3.0
ntc-templates==6.4.0
textfsm==1.1.3
flask==3.0.3
```

### `inventory/defaults.yaml`, `groups.yaml`, `hosts.yaml`

ใช้ไฟล์เดียวกับ Step 462 (ดูรายละเอียดเต็มด้านบน) — ไม่ต้องเขียนซ้ำในที่นี้

### `collectors.py` — รวม Logic การ Collect + Parse ไว้ที่เดียว (Reusable)

```python
#!/usr/bin/env python3
"""
collectors.py
รวม Task function ที่ใช้ร่วมกันระหว่าง run_backup.py และ api/app.py
เพื่อไม่ให้ logic การเชื่อมต่อ/parse กระจัดกระจายหลายที่ (DRY principle)
"""

from nornir_netmiko.tasks import netmiko_send_command


def get_running_config(task):
    """Nornir task: ดึง running-config แบบ raw text (ใช้สำหรับ backup)"""
    result = task.run(
        task=netmiko_send_command,
        command_string="show running-config",
    )
    return result


def get_interfaces_structured(task):
    """Nornir task: ดึง show ip interface brief แบบ parse ด้วย TextFSM"""
    result = task.run(
        task=netmiko_send_command,
        command_string="show ip interface brief",
        use_textfsm=True,
    )
    return result
```

### `run_backup.py` — สคริปต์หลักของ NetOps Toolkit

```python
#!/usr/bin/env python3
"""
run_backup.py
สคริปต์หลักของ NetOps Toolkit:
1. ใช้ Nornir ดึง running-config จากทุกอุปกรณ์แบบ concurrent (Step 462, 467)
2. บันทึกไฟล์ .cfg แยกทีละอุปกรณ์
3. ตรวจ Compliance เทียบ Golden Config (Step 465)
4. Commit เข้า Git พร้อม timestamp (Step 468)
5. สรุปผลทั้งหมดเป็น JSON report
"""

import json
import os
import subprocess
import sys
from datetime import datetime, timezone

from nornir import InitNornir
from nornir_utils.plugins.functions import print_result

from collectors import get_running_config
from golden_config import GOLDEN_RULES

BACKUP_DIR = "config-backups"


def run_git(*args, cwd=BACKUP_DIR):
    result = subprocess.run(["git", *args], cwd=cwd, capture_output=True, text=True)
    return result.returncode, result.stdout.strip(), result.stderr.strip()


def ensure_repo_initialized():
    os.makedirs(BACKUP_DIR, exist_ok=True)
    if not os.path.isdir(os.path.join(BACKUP_DIR, ".git")):
        run_git("init")
        run_git("config", "user.name", "NetOps Toolkit Bot")
        run_git("config", "user.email", "netops-toolkit@lab.local")


def check_compliance(config_text):
    violations = []
    for rule in GOLDEN_RULES:
        if not rule["pattern"].search(config_text):
            violations.append({"id": rule["id"], "description": rule["description"],
                                "severity": rule["severity"]})
    return violations


def main():
    ensure_repo_initialized()

    print("[*] เริ่ม NetOps Toolkit: Backup + Compliance Check ทุกอุปกรณ์ (concurrent) ...")
    nr = InitNornir(config_file="config.yaml")
    result = nr.run(task=get_running_config)
    print_result(result, severity_level=40)   # แสดงแค่ error เท่านั้น ไม่ spam log เต็มจอ

    report = {
        "run_at": datetime.now(timezone.utc).isoformat(),
        "devices": {},
    }

    for hostname, multi_result in result.items():
        if multi_result.failed:
            report["devices"][hostname] = {"status": "unreachable", "violations": None}
            continue

        config_text = multi_result[0].result
        filepath = os.path.join(BACKUP_DIR, f"{hostname}.cfg")
        with open(filepath, "w", encoding="utf-8") as f:
            f.write(config_text)

        violations = check_compliance(config_text)
        report["devices"][hostname] = {
            "status": "compliant" if not violations else "drift_detected",
            "violations": violations,
        }

    # --- เขียน compliance report เป็น JSON ---
    with open(os.path.join(BACKUP_DIR, "..", "compliance_report.json"), "w", encoding="utf-8") as f:
        json.dump(report, f, indent=2, ensure_ascii=False)

    # --- git commit เฉพาะเมื่อมีการเปลี่ยนแปลงจริง (idempotent เหมือน Step 468) ---
    run_git("add", "-A")
    returncode, _, _ = run_git("diff", "--cached", "--quiet")
    if returncode != 0:
        timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")
        run_git("commit", "-m", f"NetOps Toolkit automated backup - {timestamp}")
        print(f"[*] Git commit ใหม่ถูกสร้างที่ {timestamp}")
    else:
        print("[*] ไม่มี config เปลี่ยนแปลง — ไม่สร้าง commit ใหม่")

    drift_count = sum(1 for d in report["devices"].values() if d["status"] == "drift_detected")
    print(f"\n[*] สรุปผล: ตรวจ {len(report['devices'])} อุปกรณ์, พบ Config Drift {drift_count} ตัว")
    print("[*] ดูรายละเอียดใน compliance_report.json")

    sys.exit(1 if drift_count > 0 else 0)


if __name__ == "__main__":
    main()
```

### `api/app.py` — Flask Endpoint ที่ผูกกับ Toolkit เดียวกัน

```python
#!/usr/bin/env python3
"""
api/app.py
Flask endpoint ของ NetOps Toolkit — ให้ระบบอื่นดึง compliance report ล่าสุด
และสั่ง trigger backup รอบใหม่ผ่าน HTTP ได้ (เชื่อมกับ run_backup.py โดยตรง)
"""

import json
import os
import subprocess

from flask import Flask, jsonify

app = Flask(__name__)

TOOLKIT_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
REPORT_PATH = os.path.join(TOOLKIT_ROOT, "compliance_report.json")


@app.route("/report/latest", methods=["GET"])
def latest_report():
    """คืนค่า compliance report ล่าสุดที่ run_backup.py สร้างไว้"""
    if not os.path.exists(REPORT_PATH):
        return jsonify({"error": "ยังไม่มีรายงาน — กรุณารัน run_backup.py ก่อน"}), 404

    with open(REPORT_PATH, encoding="utf-8") as f:
        return jsonify(json.load(f))


@app.route("/backup/trigger", methods=["POST"])
def trigger_backup():
    """สั่งให้ run_backup.py ทำงานทันที (synchronous — รอผลจนเสร็จก่อนตอบ)"""
    result = subprocess.run(
        ["python3", "run_backup.py"],
        cwd=TOOLKIT_ROOT,
        capture_output=True,
        text=True,
        timeout=120,
    )
    return jsonify({
        "exit_code": result.returncode,
        "drift_detected": result.returncode == 1,
        "stdout_tail": result.stdout.strip().splitlines()[-5:],
    })


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

### วิธีรันโปรเจกต์เต็มรูปแบบ End-to-End

```bash
# 1) ติดตั้ง dependency ทั้งหมด
cd netops-toolkit
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 2) ตั้ง credential (Environment Variable ตามที่ inventory/defaults.yaml อ้างอิง)
export NET_USERNAME=admin
export NET_PASSWORD='StrongP@ssw0rd!'
export NET_SECRET='Cisco123!'

# 3) รัน Backup + Compliance Check รอบแรก (สร้าง Git repo อัตโนมัติ)
python3 run_backup.py

# 4) เปิด Flask API เพื่อให้ระบบอื่นเรียกดูผลลัพธ์/trigger รอบใหม่
cd api && flask --app app run --host 0.0.0.0 --port 5000
```

### ตัวอย่าง Output จริงของการรันเต็มรูปแบบ

```
$ python3 run_backup.py
[*] เริ่ม NetOps Toolkit: Backup + Compliance Check ทุกอุปกรณ์ (concurrent) ...
[*] Git commit ใหม่ถูกสร้างที่ 2026-09-26 09:30:12 UTC

[*] สรุปผล: ตรวจ 6 อุปกรณ์, พบ Config Drift 1 ตัว
[*] ดูรายละเอียดใน compliance_report.json

$ cat compliance_report.json
{
  "run_at": "2026-09-26T09:30:10.221845+00:00",
  "devices": {
    "CORE-SW1": {"status": "compliant", "violations": []},
    "CORE-SW2": {"status": "compliant", "violations": []},
    "DIST-SW1": {"status": "compliant", "violations": []},
    "DIST-SW2": {
      "status": "drift_detected",
      "violations": [
        {"id": "SYSLOG-01", "description": "ต้องส่ง log ไปยัง Syslog server 10.10.99.50", "severity": "high"}
      ]
    },
    "DIST-SW3": {"status": "compliant", "violations": []},
    "DIST-SW4": {"status": "compliant", "violations": []}
  }
}

$ curl -X POST http://localhost:5000/backup/trigger
{
  "exit_code": 1,
  "drift_detected": true,
  "stdout_tail": [
    "[*] Git commit ใหม่ถูกสร้างที่ 2026-09-26 09:35:44 UTC",
    "",
    "[*] สรุปผล: ตรวจ 6 อุปกรณ์, พบ Config Drift 1 ตัว",
    "[*] ดูรายละเอียดใน compliance_report.json"
  ]
}
```

โปรเจกต์นี้คือ "NetOps Toolkit" ตัวเล็กแต่**ครบวงจร**: Inventory-driven (Nornir), Concurrent
(Thread Pool ในตัว Nornir runner), Structured Data (TextFSM), Version-controlled (Git), และ
เปิดเป็น API ให้ระบบอื่นเรียกใช้ (Flask) — สถาปัตยกรรมแบบนี้คือรากฐานเดียวกันกับที่ทีม NetOps
ระดับ Enterprise ใช้สร้างเครื่องมือภายในของตัวเอง ก่อนที่จะขยับไปสู่ CI/CD Pipeline เต็มรูปแบบ
ที่จะเรียนต่อใน Part 48

---

## แบบฝึกหัดทวนความเข้าใจ Part 47

1. NAPALM ต่างจาก Netmiko อย่างไรในเรื่อง output ที่ได้กลับมา และเมื่อไหร่ที่ยังต้องใช้ Netmiko
   แม้จะมี NAPALM แล้ว?
2. ทำไม Nornir ที่รันแบบ concurrent (multi-threaded) ถึงเหมาะกับงาน SSH/Network Automation
   มากกว่าการใช้ `multiprocessing` (Process-based)?
3. อธิบายหลักการของ Exponential Backoff ใน Retry Decorator (Step 466) — ทำไมการเพิ่มเวลารอ
   แบบทวีคูณจึงดีกว่าการ retry ทันทีซ้ำๆด้วยเวลารอเท่ากันทุกครั้ง?
4. เพราะเหตุใดการ commit config backup เข้า Git จึงมีประโยชน์มากกว่าการ backup แบบ overwrite
   ไฟล์เดิมทุกวัน? ยกตัวอย่างสถานการณ์จริงที่ Git diff ช่วยแก้ปัญหาได้
5. Flask API ตัวอย่างใน Step 469 มีข้อบกพร่องด้าน Security อะไรบ้างที่ต้องแก้ก่อนนำไปใช้งานจริง?

**เฉลย:**

1. Netmiko คืนค่าเป็น raw text ที่ต้อง parse เอง ส่วน NAPALM คืนค่าเป็น Python dict/list ที่มี
   structure มาตรฐานเดียวกันทุกยี่ห้อผ่าน getter method (`get_facts()`, `get_interfaces()` ฯลฯ)
   แต่ยังต้องใช้ Netmiko (หรือ `device.cli()` ของ NAPALM ซึ่งคืนค่า raw text เหมือนกัน) เมื่อ
   ต้องการข้อมูลจากคำสั่งที่ NAPALM ไม่มี getter รองรับ หรือต้อง push คำสั่ง CLI แบบ interactive
   เจาะจง
2. เพราะงาน SSH/Network Automation เป็น I/O-bound (เวลาส่วนใหญ่ใช้รอ network ตอบกลับ ไม่ใช่ใช้
   CPU ประมวลผล) — Python GIL จะถูกปลดปล่อยระหว่างรอ I/O ทำให้ Thread หลายตัวทำงานคาบเกี่ยวกันได้
   ดีอยู่แล้วโดยไม่ต้องเสีย overhead สร้าง Process ใหม่ (ซึ่งหนักกว่าและเสีย memory มากกว่า) แบบที่
   `multiprocessing` ทำ ซึ่งเหมาะกับงาน CPU-bound มากกว่า
3. Exponential Backoff เพิ่มเวลารอเป็นทวีคูณ (เช่น 2s → 4s → 8s) ในแต่ละรอบ retry เพื่อไม่ยิง
   request ซ้ำถี่เกินไปในขณะที่อุปกรณ์/เครือข่ายกำลังมีปัญหาอยู่แล้ว การ retry ทันทีด้วยเวลาเท่ากัน
   ทุกครั้งอาจทำให้ปัญหาแย่ลง (เช่น อุปกรณ์ที่ CPU สูงอยู่แล้วถูกยิง SSH ซ้ำถี่ยิ่งทำให้แย่ลง) ในขณะ
   ที่การรอนานขึ้นเรื่อยๆให้เวลาระบบ "ฟื้นตัว" ก่อนลองใหม่
4. การ backup แบบ overwrite ทำให้เห็นแค่สถานะปัจจุบัน ไม่รู้ว่าเกิดอะไรขึ้นระหว่างทาง Git เก็บทุก
   commit เป็น snapshot ของแต่ละวัน ทำให้ใช้ `git diff` เทียบสองวันที่ต่างกันได้ทันที เช่น กรณี
   ตัวอย่างใน Step 468 ที่ตรวจพบว่ามีคนลบ `logging host`/`logging trap` ของ DIST-SW2 ออกไป
   ระหว่างวันที่ 24-26 กันยายน ซึ่งการ overwrite แบบเดิมจะไม่มีทางรู้เรื่องนี้เลย
5. ไม่มี Authentication/API Key ป้องกัน endpoint, รับคำสั่ง config อิสระจาก request body โดยไม่มี
   Whitelist/Validation, รันผ่าน HTTP ธรรมดาไม่ใช่ HTTPS, ใช้ Flask dev server (`app.run()`)
   ซึ่งไม่เหมาะกับ production traffic (ต้องใช้ WSGI server เช่น gunicorn), และไม่มี Rate Limiting
   ป้องกันการยิง request จนอุปกรณ์ปลายทาง VTY session เต็ม

---

## สรุป Part 47

Part นี้พาเราลงลึกฝั่ง Python สำหรับ Network Automation ระดับ CCNP: จาก **NAPALM** ที่ทำให้ได้
structured data ข้ามยี่ห้อ, **Nornir** ที่จัดการ Inventory + Concurrency ให้ในตัว, **TextFSM/
ntc-templates** ที่แปลง unstructured CLI text เป็น Python object ได้อย่างเป็นระบบ, การสร้าง
**Inventory/CMDB และ Compliance Checker** ของตัวเอง, **Error Handling/Retry ที่ทนต่อความล้มเหลว
ชั่วคราว**, **Parallel Execution** ด้วย `ThreadPoolExecutor`, **Git Version Control** สำหรับ
config history, และปิดท้ายด้วยการเปิด Automation Script เป็น **REST API** ผ่าน Flask — จบด้วย
Lab เต็มรูปแบบที่รวมทุกเทคนิคเข้าเป็น "NetOps Toolkit" ที่ใช้งานได้จริงกับ Lab ทั้ง 6 อุปกรณ์
Core/Distribution ✅ **พร้อมสำหรับ Part 48**: เราจะกลับไปที่ฝั่ง Ansible อีกครั้งในระดับ Advanced
— ยกระดับ Playbook พื้นฐานจาก Part 23 ไปสู่ **Production-grade Ansible Project** ด้วย Roles,
Galaxy Collections, Handlers, Tags, Vault แบบ Multi-Environment, และภาพรวม AWX/Ansible Automation
Platform ซึ่งต่อยอดแนวคิด Version Control จาก Git Integration ใน Step 468 ของ Part นี้โดยตรง

**ไปต่อ:** [Part 48 — Network Automation: Ansible for Network Engineers (Advanced) →](part-048-automation-ansible-advanced.md)
