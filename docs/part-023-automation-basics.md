# Part 23 — Network Automation Basics (NETCONF/RESTCONF/Ansible Intro)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 221–230 จาก 1000**

> ต่อจาก [Part 22 — AAA & Device Hardening](part-022-aaa-device-hardening.md) ที่เราปิดประตูความ
> ปลอดภัยของอุปกรณ์ด้วย AAA/TACACS+/RADIUS และ Device Hardening ครบทุกตัวใน Lab แล้ว
> Part นี้คือจุดเปลี่ยนสำคัญของหลักสูตร — เราจะเลิกพิมพ์คำสั่งซ้ำๆบน CLI ทีละอุปกรณ์
> แล้วเริ่ม **เขียนโปรแกรม (script) เพื่อควบคุมเครือข่ายทั้งหมดพร้อมกัน** ด้วย Python (Netmiko),
> Ansible, และแอบดู NETCONF/RESTCONF ซึ่งเป็นรากฐานของ Network Automation ยุคใหม่
> (การเจาะลึก NETCONF/RESTCONF/YANG เต็มรูปแบบจะอยู่ใน **Part 49**, และ CI/CD Pipeline สำหรับ
> Network-as-Code จะอยู่ใน **Part 47-50/79**)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 221 | ทำไมต้อง Network Automation — ปัญหาของการ config มือ และภาพรวม Automation Stack |
| 222 | Python สำหรับวิศวกรเครือข่าย — ติดตั้งและใช้ Netmiko เชื่อมต่อ SSH |
| 223 | Python Script Push Configuration ไปยังหลายอุปกรณ์พร้อม Error Handling |
| 224 | แนะนำ Ansible สำหรับ Network Automation — Inventory, Ad-hoc, YAML, Collection |
| 225 | สร้าง Ansible Inventory File สำหรับ Lab ทั้งหมด |
| 226 | เขียน Ansible Playbook พื้นฐาน — ดึง `show version` ด้วย `ios_command` |
| 227 | เขียน Ansible Playbook Push Config — `ios_config` และแนวคิด Idempotency |
| 228 | Jinja2 Templating สำหรับสร้าง Configuration แบบไดนามิก |
| 229 | แอบดู NETCONF/RESTCONF — Model-Driven API เทียบกับ CLI Screen-Scraping |
| 230 | Lab เต็มรูปแบบ: Automation Project — Backup + Push Config + Verify ด้วย Ansible |

---

## Step 221 — ทำไมต้อง Network Automation

### ปัญหาของการ Config อุปกรณ์ด้วยมือ (Manual CLI)

ตลอด 22 Part ที่ผ่านมา เราพิมพ์คำสั่งบน CLI ทีละอุปกรณ์ ทีละคำสั่ง — วิธีนี้ใช้ได้ดีกับ Lab ขนาด
เล็ก (อุปกรณ์ ~15 ตัว) แต่ใน**สภาพแวดล้อมจริงระดับ Enterprise** ที่มีอุปกรณ์หลักร้อยถึงหลักหมื่นตัว
วิธีนี้ **ไม่สามารถ scale ได้เลย** ปัญหาหลักมีดังนี้:

| ปัญหา | รายละเอียด | ผลกระทบ |
|---|---|---|
| **Configuration Drift** | อุปกรณ์แต่ละตัวถูกแก้ config มือทีละนิดโดยวิศวกรหลายคนไปเรื่อยๆ | Config ไม่ตรงกันระหว่างอุปกรณ์ที่ควรเหมือนกัน (เช่น DIST-SW1-4) |
| **Human Error** | พิมพ์ผิด, ลืมขั้นตอน, ลืม `copy run start` | Network downtime, security hole ที่มองไม่เห็น |
| **เวลาที่เสียไป** | Config อุปกรณ์ 50 ตัวด้วยมือ ใช้เวลาหลายชั่วโมงถึงหลายวัน | Time-to-deploy ช้า, ไม่ทันความต้องการธุรกิจ |
| **ไม่มี Audit Trail** | ไม่รู้ว่าใครแก้อะไร เมื่อไหร่ ทำไม | Troubleshoot ยาก, Compliance ไม่ผ่าน |
| **Repetitive Task** | งานซ้ำๆ เช่น เพิ่ม VLAN ใหม่ 50 Switch, เปลี่ยน NTP Server ทั้ง Site | วิศวกรเสียเวลากับงานที่ไม่สร้างมูลค่า แทนที่จะโฟกัส Design/Architecture |
| **Scale ไม่ได้** | Cloud/Data Center สมัยใหม่มี Switch หลักพัน-หมื่นตัว | เป็นไปไม่ได้ที่จะ config มือทีละตัว |

> **แนวคิดสำคัญ**: Network Automation ไม่ได้มีไว้เพื่อ "แทนที่วิศวกรเครือข่าย" แต่มีไว้เพื่อ
> **เปลี่ยนงานวิศวกรจาก "คนพิมพ์คำสั่ง" ให้เป็น "คนออกแบบระบบและเขียนโปรแกรมที่ควบคุมเครือข่าย"**
> ซึ่งคือทิศทางที่ Cisco ผลักดันมาตั้งแต่ CCNA 200-301 (มี DevNet Associate แยกออกมา) และเป็นหัวข้อ
> บังคับใน CCNP ENCOR/ENARSI สมัยใหม่

### จาก CLI-First สู่ API-First

```
ยุคเก่า (CLI-First)                       ยุคใหม่ (API-First / Model-Driven)
┌─────────────────────┐                  ┌─────────────────────┐
│  วิศวกร → SSH/Telnet  │                  │  Script/Tool → API   │
│  → พิมพ์คำสั่ง CLI     │                  │  → ส่งข้อมูลแบบ        │
│  → อ่าน text output   │                  │    Structured Data   │
│    (ต้อง parse เอง)    │                  │    (JSON/XML/YANG)   │
└─────────────────────┘                  └─────────────────────┘
      Human-optimized                          Machine-optimized
   (อ่านง่ายสำหรับคน)                        (ประมวลผลได้แน่นอน ไม่ต้อง
                                              เดา format, ไม่ parse text)
```

CLI ถูกออกแบบมาให้ **คนอ่าน** ส่วน API (NETCONF/RESTCONF) ถูกออกแบบมาให้ **โปรแกรมอ่าน** — การ
"parse" text output จาก `show` command ด้วย regex เป็นวิธีที่เปราะบาง (fragile) เพราะ format
output อาจเปลี่ยนไปในแต่ละ IOS version ในขณะที่ API แบบ Model-Driven (มี Schema ตายตัวจาก YANG)
จะให้ข้อมูลที่มี structure แน่นอนเสมอ

### ภาพรวม Automation Stack ของหลักสูตรนี้ (Roadmap)

```
Level 1: SSH/Expect Script          ← ง่ายสุด แต่ fragile, เปราะบางต่อการเปลี่ยน prompt/output
   │      (Bash + expect, TCL scripting on-box)
   ▼
Level 2: Python + Netmiko/Paramiko  ← Part 23 (ตอนนี้) — SSH เหมือนเดิม แต่ควบคุมด้วยโปรแกรม
   │      ยังเป็น "CLI-based" อยู่ (ส่งคำสั่ง CLI ผ่าน SSH) แต่ scale ได้ดีกว่ามาก
   ▼
Level 3: Ansible (cisco.ios / ios_config / ios_command)   ← Part 23 (ตอนนี้)
   │      Declarative, Idempotent, Agentless, Inventory-based
   ▼
Level 4: NETCONF/RESTCONF + YANG    ← แอบดูใน Part 23, เจาะลึกใน Part 49
   │      Model-Driven API — ไม่ parse text, ใช้ Structured Data (JSON/XML) ตรงตาม Schema
   ▼
Level 5: CI/CD Pipeline (Git + Pipeline + Testing)   ← Part 47-50, Part 79
          Network-as-Code เต็มรูปแบบ: Config อยู่ใน Git, ผ่าน Automated Test/Pipeline ก่อน Deploy จริง
```

Part นี้ ("Network Automation Basics") จะพาไปที่ **Level 2 และ Level 3** อย่างละเอียด (เขียน
Python script จริงที่รันได้, เขียน Ansible Playbook จริงที่รันได้) และ**แอบดู** Level 4 พอให้เห็น
ภาพรวม — Level 4 (NETCONF/RESTCONF/YANG เต็มรูปแบบ) และ Level 5 (CI/CD) จะเรียนลึกใน Part
หลังๆของหลักสูตร ตามที่ Part 2 Step 17 ได้ forward-reference ไว้ ("Best Practice ระดับ Enterprise
คือทำ config backup อัตโนมัติด้วย Python/Ansible" — นี่คือ Part นั้น)

### เครื่องมือที่ต้องติดตั้งก่อนเริ่ม Part นี้

| เครื่องมือ | ใช้ทำอะไร | คำสั่งติดตั้ง (Ubuntu/Debian/macOS) |
|---|---|---|
| Python 3.9+ | รัน Script อัตโนมัติทั้งหมดใน Part นี้ | มักติดตั้งมาแล้วในระบบ Linux |
| pip | Package Manager ของ Python | `python3 -m ensurepip --upgrade` |
| Netmiko | Python library สำหรับ SSH เข้าอุปกรณ์ Network | `pip install netmiko` |
| Ansible | Automation/Orchestration Engine | `pip install ansible` |
| cisco.ios Collection | Ansible module สำหรับ Cisco IOS/IOS-XE | `ansible-galaxy collection install cisco.ios` |
| Git | เก็บ Script/Playbook เป็น version control | `apt install git` / มีมาแล้วบน macOS |

---

## Step 222 — Python สำหรับวิศวกรเครือข่าย: Netmiko

### Netmiko คืออะไร

**Netmiko** คือ Python library (สร้างต่อจาก Paramiko) ที่ออกแบบมาเพื่อ **SSH เข้าอุปกรณ์
Network โดยเฉพาะ** — จัดการเรื่องยากๆของ CLI อัตโนมัติ เช่น รอ prompt, ตัด banner/paging ทิ้ง
(`terminal length 0`), ส่งคำสั่ง `enable`, และรองรับอุปกรณ์หลายยี่ห้อ (Cisco IOS, IOS-XE, NX-OS,
ASA, Juniper, Arista ฯลฯ) ผ่าน parameter เดียวคือ `device_type`

### ติดตั้ง Netmiko

```bash
pip install netmiko
# ตรวจสอบเวอร์ชันที่ติดตั้ง
python3 -c "import netmiko; print(netmiko.__version__)"
# ตัวอย่าง output: 4.3.0
```

### Netmiko `device_type` ที่ใช้บ่อยในหลักสูตรนี้

| Platform | `device_type` |
|---|---|
| Cisco IOS (Classic, 2960/3560) | `cisco_ios` |
| Cisco IOS-XE (Catalyst 9000, ISR4000, CSR1000v) | `cisco_ios` (ใช้ตัวเดียวกัน) |
| Cisco NX-OS (Nexus) | `cisco_nxos` |
| Cisco ASA/FTD | `cisco_asa` |
| Cisco WLC (AireOS) | `cisco_wlc` |

> ทุกอุปกรณ์ใน Lab หลักสูตรนี้ (CORE-SW1/2, DIST-SW1-4 = Catalyst 9500/9300 บน IOS-XE) จะใช้
> `device_type: "cisco_ios"` เสมอ เพราะ Netmiko ถือว่า IOS-XE เข้ากันได้กับ driver ของ IOS classic

### Script แรก: SSH เข้า CORE-SW1 และรัน `show ip interface brief`

ใช้ IP Management ตามแผนกลาง (`docs/00-ip-address-plan.md`) — CORE-SW1 อยู่ที่ VLAN 99
(`10.10.99.10`) และ user account `admin` ที่สร้างไว้ตั้งแต่ Part 2/22 (AAA local fallback)

```python
#!/usr/bin/env python3
"""
show_ip_int_brief.py
สคริปต์ตัวอย่างแรก: SSH เข้า CORE-SW1 ด้วย Netmiko แล้วรัน show ip interface brief
"""

from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException

# --- Device Dictionary: ข้อมูลการเชื่อมต่อของ CORE-SW1 ---
core_sw1 = {
    "device_type": "cisco_ios",
    "host": "10.10.99.10",          # CORE-SW1 MGMT VLAN 99 (docs/00-ip-address-plan.md)
    "username": "admin",
    "password": "StrongP@ssw0rd!",   # ในงานจริงห้าม hardcode รหัสผ่านแบบนี้ (ดูหมายเหตุด้านล่าง)
    "secret": "Cisco123!",           # enable secret สำหรับเข้าสู่ privileged EXEC
    "port": 22,
    "fast_cli": False,               # ปิด fast_cli ระหว่างเรียนรู้ เพื่อดู behavior ชัดเจนกว่า
}

def main():
    print(f"[*] กำลังเชื่อมต่อไปยัง CORE-SW1 ({core_sw1['host']}) ...")
    try:
        connection = ConnectHandler(**core_sw1)
        connection.enable()  # เข้าสู่ privileged EXEC mode (เทียบเท่าพิมพ์ 'enable' บน CLI)

        output = connection.send_command("show ip interface brief")
        print("\n=== show ip interface brief (CORE-SW1) ===")
        print(output)

        # ตัวอย่างการดึง hostname จริงจาก prompt เพื่อยืนยันว่าต่อถูกเครื่อง
        print(f"\n[*] เชื่อมต่อสำเร็จ prompt ปัจจุบันคือ: {connection.find_prompt()}")

        connection.disconnect()

    except NetmikoAuthenticationException:
        print("[!] Authentication ล้มเหลว — ตรวจสอบ username/password/secret")
    except NetmikoTimeoutException:
        print("[!] เชื่อมต่อไม่ได้ (Timeout) — ตรวจสอบ IP/Routing/ACL/SSH enable บนอุปกรณ์")

if __name__ == "__main__":
    main()
```

### ตัวอย่าง Output จริงเมื่อรัน Script

```
$ python3 show_ip_int_brief.py
[*] กำลังเชื่อมต่อไปยัง CORE-SW1 (10.10.99.10) ...

=== show ip interface brief (CORE-SW1) ===
Interface              IP-Address      OK? Method Status                Protocol
Vlan99                 10.10.99.10     YES manual up                    up
Port-channel1          10.255.0.1      YES manual up                    up
TenGigabitEthernet1/0/1 10.255.10.1    YES manual up                    up
TenGigabitEthernet1/0/2 10.255.20.1    YES manual up                    up
Loopback0              1.1.1.1         YES manual up                    up

[*] เชื่อมต่อสำเร็จ prompt ปัจจุบันคือ: CORE-SW1#
```

> **หมายเหตุความปลอดภัยสำคัญ**: การเขียน password แบบ hardcode ในไฟล์ `.py` เหมือนตัวอย่างข้างบน
> ทำเพื่อความง่ายในการสอน **ห้ามทำแบบนี้ในระบบจริงเด็ดขาด** — ในงานจริงต้องอ่าน credential จาก
> environment variable (`os.environ.get("NET_PASSWORD")`), จาก Ansible Vault (Step 225),
> หรือจาก Secret Manager (HashiCorp Vault, AWS Secrets Manager) เท่านั้น และไฟล์ script/inventory
> ที่มี credential ต้องอยู่ใน `.gitignore` เสมอ ไม่ commit เข้า Git repository

---

## Step 223 — Python Script Push Configuration ไปยังหลายอุปกรณ์

Script ต่อไปนี้จะ**เชื่อมต่อไปยังทุกอุปกรณ์ Core/Distribution ใน Lab** (CORE-SW1, CORE-SW2,
DIST-SW1-4) แล้ว push บรรทัด config ชุดเดียวกัน (เพิ่ม NTP Server ตัวใหม่) โดยมี **Error Handling
แยกทีละอุปกรณ์** — ถ้าอุปกรณ์ตัวใดตัวหนึ่งเชื่อมต่อไม่ได้ script จะบันทึก error แล้ว**ทำงานต่อกับ
อุปกรณ์ตัวถัดไป** ไม่ใช่หยุดทั้งหมด (สำคัญมากสำหรับ Automation ระดับ Production)

```python
#!/usr/bin/env python3
"""
push_ntp_config.py
เชื่อมต่อไปยัง CORE-SW1, CORE-SW2, DIST-SW1-4 และ push NTP server ใหม่แบบ batch
พร้อม error handling แยกทีละอุปกรณ์ (ตัวหนึ่งพังไม่กระทบตัวอื่น)
"""

from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoTimeoutException, NetmikoAuthenticationException
import os
import sys

# --- รายชื่ออุปกรณ์ทั้งหมดที่จะ push config (จาก docs/00-ip-address-plan.md) ---
DEVICES = [
    {"name": "CORE-SW1", "host": "10.10.99.10"},
    {"name": "CORE-SW2", "host": "10.10.99.11"},
    {"name": "DIST-SW1", "host": "10.10.99.12"},
    {"name": "DIST-SW2", "host": "10.10.99.13"},
    {"name": "DIST-SW3", "host": "10.10.99.14"},
    {"name": "DIST-SW4", "host": "10.10.99.15"},
]

# --- Config Set ที่จะ push เหมือนกันทุกอุปกรณ์ ---
CONFIG_COMMANDS = [
    "ntp server 10.10.99.50 prefer",
    "ntp server 10.10.99.51",
]

# ดึง credential จาก Environment Variable แทนการ hardcode (best practice)
USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")

if not PASSWORD or not SECRET:
    sys.exit("[!] กรุณา export NET_PASSWORD และ NET_SECRET ก่อนรัน script นี้")


def push_config(device):
    """เชื่อมต่อและ push config ไปยังอุปกรณ์ 1 ตัว คืนค่า (success: bool, message: str)"""
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

        output = connection.send_config_set(CONFIG_COMMANDS)

        # บันทึก config ทันที (เทียบเท่า copy run start บน CLI)
        save_output = connection.save_config()

        connection.disconnect()
        return True, f"{output}\n{save_output}"

    except NetmikoAuthenticationException:
        return False, "Authentication ล้มเหลว (ตรวจสอบ username/password/secret)"
    except NetmikoTimeoutException:
        return False, "เชื่อมต่อไม่ได้ (Timeout - ตรวจสอบ Network/SSH)"
    except Exception as e:
        return False, f"เกิดข้อผิดพลาดไม่คาดคิด: {str(e)}"


def main():
    results = {"success": [], "failed": []}

    for device in DEVICES:
        print(f"[*] กำลัง push config ไปยัง {device['name']} ({device['host']}) ...")
        ok, message = push_config(device)

        if ok:
            print(f"    [OK] {device['name']} สำเร็จ")
            results["success"].append(device["name"])
        else:
            print(f"    [FAIL] {device['name']}: {message}")
            results["failed"].append((device["name"], message))

    # --- สรุปผลลัพธ์รวม ---
    print("\n" + "=" * 50)
    print(f"สำเร็จ: {len(results['success'])}/{len(DEVICES)} อุปกรณ์")
    print(f"  -> {', '.join(results['success']) if results['success'] else '(ไม่มี)'}")
    if results["failed"]:
        print(f"ล้มเหลว: {len(results['failed'])} อุปกรณ์")
        for name, reason in results["failed"]:
            print(f"  -> {name}: {reason}")
    print("=" * 50)


if __name__ == "__main__":
    main()
```

### วิธีรัน Script (ตั้ง Environment Variable ก่อน)

```bash
export NET_USERNAME="admin"
export NET_PASSWORD="StrongP@ssw0rd!"
export NET_SECRET="Cisco123!"
python3 push_ntp_config.py
```

### ตัวอย่าง Output จริง (จำลองกรณี DIST-SW3 เชื่อมต่อไม่ได้เพราะสาย/interface down)

```
[*] กำลัง push config ไปยัง CORE-SW1 (10.10.99.10) ...
    [OK] CORE-SW1 สำเร็จ
[*] กำลัง push config ไปยัง CORE-SW2 (10.10.99.11) ...
    [OK] CORE-SW2 สำเร็จ
[*] กำลัง push config ไปยัง DIST-SW1 (10.10.99.12) ...
    [OK] DIST-SW1 สำเร็จ
[*] กำลัง push config ไปยัง DIST-SW2 (10.10.99.13) ...
    [OK] DIST-SW2 สำเร็จ
[*] กำลัง push config ไปยัง DIST-SW3 (10.10.99.14) ...
    [FAIL] DIST-SW3: เชื่อมต่อไม่ได้ (Timeout - ตรวจสอบ Network/SSH)
[*] กำลัง push config ไปยัง DIST-SW4 (10.10.99.15) ...
    [OK] DIST-SW4 สำเร็จ

==================================================
สำเร็จ: 5/6 อุปกรณ์
  -> CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2, DIST-SW4
ล้มเหลว: 1 อุปกรณ์
  -> DIST-SW3: เชื่อมต่อไม่ได้ (Timeout - ตรวจสอบ Network/SSH)
==================================================
```

นี่คือเหตุผลที่ **ต้องมี error handling แยกทีละอุปกรณ์** — ถ้า DIST-SW3 พัง โปรแกรมยังต้อง push
สำเร็จให้อีก 5 ตัว และรายงานให้วิศวกรรู้ชัดเจนว่าตัวไหนต้องไปแก้ไขเพิ่มเติมด้วยมือ

---

## Step 224 — แนะนำ Ansible สำหรับ Network Automation

### Netmiko vs Ansible — ต่างกันอย่างไร

| ประเด็น | Netmiko (Python) | Ansible |
|---|---|---|
| แนวคิด | Procedural (บอกทีละขั้นตอนว่าต้องทำอะไร) | Declarative (บอก "สถานะที่ต้องการ" ไม่ต้องบอกทุกขั้นตอน) |
| Idempotency | ต้องเขียน logic ตรวจสอบเองว่ามี config อยู่แล้วหรือยัง | มีในตัว (built-in) โดย module ส่วนใหญ่ |
| การจัดการหลายอุปกรณ์ | เขียน loop เอง (ตัวอย่าง Step 223) | มี Inventory + Parallelism ในตัว |
| Learning Curve | ต้องรู้ Python | เขียน YAML (อ่านง่ายกว่าสำหรับ non-programmer) |
| ความยืดหยุ่น | สูงมาก (เขียน logic อะไรก็ได้) | จำกัดตาม module ที่มี (แต่ครอบคลุมงานส่วนใหญ่) |
| ใช้เมื่อไหร่ | Logic ซับซ้อน, ต้อง parse/ประมวลผลข้อมูลเยอะ | Config Management มาตรฐาน, ทีมงานหลายคนต้อง maintain ร่วมกัน |

> ในงานจริง ทีม NetOps ระดับ Enterprise มักใช้ **ทั้งสองอย่างร่วมกัน**: Ansible สำหรับ Config
> Management ประจำวัน (idempotent, เขียน YAML อ่านง่าย, ทุกคนในทีม maintain ได้) และ Python/Netmiko
> สำหรับงานที่ต้องมี logic ซับซ้อน เช่น parsing output ทำ report, integrate กับระบบอื่น (ITSM,
> Monitoring), หรือเขียน Custom Ansible Module เอง

### Ansible คืออะไร (สรุปสั้น)

**Ansible** เป็น IT Automation Engine แบบ **Agentless** (ไม่ต้องติดตั้ง agent บนอุปกรณ์ปลายทาง —
สำหรับ Network device จะสั่งงานผ่าน SSH โดยตรง) ทำงานโดยอ่านไฟล์ 2 ประเภทหลัก:

1. **Inventory** — รายชื่ออุปกรณ์ที่จะควบคุม (Step 225)
2. **Playbook** (YAML) — ชุดคำสั่ง (Task) ที่จะรันกับอุปกรณ์ใน Inventory (Step 226-227)

### YAML Syntax พื้นฐานที่ต้องรู้ก่อนเขียน Playbook

```yaml
# คอมเมนต์ขึ้นต้นด้วย #
key: value                 # key-value พื้นฐาน
number: 42
boolean_true: true
boolean_false: false
list_example:               # List ใช้ "-" นำหน้าแต่ละ item
  - item1
  - item2
  - item3
dict_example:                # Dictionary/Nested object ใช้ indent (ห้ามใช้ Tab เด็ดขาด!)
  sub_key1: value1
  sub_key2: value2
list_of_dicts:               # รูปแบบที่ใช้บ่อยที่สุดใน Ansible (list ของ task/host)
  - name: item หนึ่ง
    value: 100
  - name: item สอง
    value: 200
```

> **กฎเหล็กของ YAML**: ใช้ **Space เท่านั้น ห้ามใช้ Tab** ในการทำ indent — นี่คือสาเหตุ Syntax
> Error อันดับ 1 ของคนเริ่มเขียน Ansible ควรตั้ง Editor (VS Code) ให้แสดง whitespace/แปลง Tab
> เป็น Space อัตโนมัติเสมอ

### Ansible Ad-hoc Command (ทดสอบเร็วๆ ไม่ต้องเขียน Playbook)

```bash
# ทดสอบ ping (connectivity) ไปยังทุก host ใน group "core_switches"
ansible core_switches -i inventory.yaml -m ansible.netcommon.ping

# รันคำสั่ง show version บนทุก host ใน group "all"
ansible all -i inventory.yaml -m cisco.ios.ios_command -a "commands='show version'"
```

### `cisco.ios` และ `ansible.netcommon` Collection คืออะไร

Ansible ยุคใหม่ (ตั้งแต่ Ansible 2.10+) แยก Module ออกเป็น **Collection** ต้อง install เพิ่มเอง
(ไม่รวมมาใน core อีกต่อไป):

| Collection | มี Module อะไรบ้าง | ใช้ทำอะไร |
|---|---|---|
| `ansible.netcommon` | `netconf_config`, `restconf_get`, `cli_command`, connection plugin `network_cli`/`netconf` | Framework กลางสำหรับ Network Collection ทั้งหมด (ใช้ร่วมกับ Cisco/Juniper/Arista) |
| `cisco.ios` | `ios_command`, `ios_config`, `ios_facts`, `ios_ntp_global`, `ios_vlans`, `ios_interfaces` ฯลฯ | Module เฉพาะสำหรับ Cisco IOS/IOS-XE (ที่ Lab หลักสูตรนี้ใช้) |
| `cisco.nxos` | เทียบเท่าข้างบนแต่สำหรับ Nexus (NX-OS) | ใช้ถ้ามี Nexus Data Center switch |

```bash
# ติดตั้ง Collection ที่ต้องใช้ทั้งหมดสำหรับ Part นี้
ansible-galaxy collection install cisco.ios
ansible-galaxy collection install ansible.netcommon

# ตรวจสอบว่าติดตั้งสำเร็จ
ansible-galaxy collection list | grep -E "cisco.ios|netcommon"
```

---

## Step 225 — Ansible Inventory File สำหรับ Lab

Inventory คือไฟล์ที่บอก Ansible ว่า "มีอุปกรณ์อะไรบ้าง จัดกลุ่มอย่างไร และเชื่อมต่ออย่างไร"
เขียนได้ 2 รูปแบบ: `.ini` (แบบดั้งเดิม) หรือ `.yaml` (แนะนำสำหรับ Playbook ที่ซับซ้อน) — จะแสดง
ทั้งสองรูปแบบให้เห็นความแตกต่าง แต่ Playbook ที่เหลือใน Part นี้จะใช้รูปแบบ YAML เป็นหลัก

### รูปแบบ `.ini` (`inventory.ini`)

```ini
[core_switches]
CORE-SW1 ansible_host=10.10.99.10
CORE-SW2 ansible_host=10.10.99.11

[dist_switches]
DIST-SW1 ansible_host=10.10.99.12
DIST-SW2 ansible_host=10.10.99.13
DIST-SW3 ansible_host=10.10.99.14
DIST-SW4 ansible_host=10.10.99.15

[network_devices:children]
core_switches
dist_switches

[network_devices:vars]
ansible_network_os=cisco.ios.ios
ansible_connection=ansible.netcommon.network_cli
ansible_user=admin
```

### รูปแบบ `.yaml` (`inventory.yaml`) — ใช้ตลอด Part นี้

```yaml
all:
  children:
    network_devices:
      vars:
        ansible_network_os: cisco.ios.ios
        ansible_connection: ansible.netcommon.network_cli
        ansible_user: admin
      children:
        core_switches:
          hosts:
            CORE-SW1:
              ansible_host: 10.10.99.10
            CORE-SW2:
              ansible_host: 10.10.99.11
        dist_switches:
          hosts:
            DIST-SW1:
              ansible_host: 10.10.99.12
            DIST-SW2:
              ansible_host: 10.10.99.13
            DIST-SW3:
              ansible_host: 10.10.99.14
            DIST-SW4:
              ansible_host: 10.10.99.15
```

### Group Variables (`group_vars/network_devices.yml`) — เก็บ Credential แยกจาก Inventory

โครงสร้างไฟล์มาตรฐานของ Ansible Project:

```
lab-automation/
├── inventory.yaml
├── group_vars/
│   └── network_devices.yml       <-- credential และค่า config ร่วม
└── playbooks/
    └── ...
```

**`group_vars/network_devices.yml` (เวอร์ชัน "ผิด" — ห้ามทำแบบนี้ในงานจริง):**

```yaml
# ❌ อย่าทำแบบนี้ — plaintext password ใน Git repository = security incident
ansible_password: "StrongP@ssw0rd!"
ansible_become_password: "Cisco123!"
```

**วิธีที่ถูกต้อง — ใช้ Ansible Vault เข้ารหัสไฟล์ credential**

```bash
# สร้างไฟล์ vault ใหม่ (จะถามรหัสผ่านสำหรับปลดล็อกไฟล์นี้)
ansible-vault create group_vars/network_devices_vault.yml
```

จากนั้นในไฟล์ (ที่ editor จะเปิดให้พิมพ์แบบ plaintext ชั่วคราว ก่อน Ansible เข้ารหัสให้อัตโนมัติ
ตอน save):

```yaml
vault_ansible_password: "StrongP@ssw0rd!"
vault_ansible_become_password: "Cisco123!"
```

ไฟล์ที่ถูกเข้ารหัสจะมีลักษณะแบบนี้เมื่อเปิดดูตรงๆ (ปลอดภัยแม้ถูก commit เข้า Git):

```
$ANSIBLE_VAULT;1.1;AES256
66386439653236336462626566653063336164663966303231363934653561353966366139
3937343934393964656334646263353762613762613262633235373135386566323936303
...
```

จากนั้นใน `group_vars/network_devices.yml` (ไฟล์ปกติ ไม่เข้ารหัส) ให้ reference ตัวแปรจากไฟล์
vault:

```yaml
ansible_network_os: cisco.ios.ios
ansible_connection: ansible.netcommon.network_cli
ansible_user: admin
ansible_password: "{{ vault_ansible_password }}"
ansible_become_password: "{{ vault_ansible_become_password }}"
ansible_become: yes
ansible_become_method: enable
```

**รัน Playbook พร้อมปลดล็อก Vault:**

```bash
# วิธี 1: ให้ Ansible ถามรหัสผ่าน Vault ตอนรัน
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass

# วิธี 2: เก็บรหัสผ่าน Vault ไว้ในไฟล์ (ต้อง chmod 600 และห้าม commit เข้า Git)
ansible-playbook -i inventory.yaml site.yaml --vault-password-file ~/.vault_pass.txt
```

> **หลักการสำคัญ**: `.gitignore` ของทุก Ansible Project ต้องมี `*_vault.yml` password file และ
> `.vault_pass.txt` เสมอ — Vault เข้ารหัสตัว "ข้อมูล" แต่ไฟล์ password ที่ใช้ปลดล็อกต้องไม่ถูก
> commit เข้า Git โดยเด็ดขาด งานจริงระดับ Enterprise มักเก็บ Vault password ไว้ใน Secret Manager
> (HashiCorp Vault, AWS Secrets Manager) แล้วดึงมาใช้ตอน CI/CD Pipeline รัน (Part 47-50)

---

## Step 226 — Ansible Playbook พื้นฐาน: ดึง `show version`

Playbook แรกของเรา: เชื่อมต่อไปยังทุกอุปกรณ์ใน Inventory, รันคำสั่ง `show version`, และบันทึกผล
ลงไฟล์แยกทีละอุปกรณ์บนเครื่อง Control Node (เครื่องที่รัน Ansible)

### `playbooks/gather_show_version.yaml`

```yaml
---
- name: Gather show version from all network devices
  hosts: network_devices
  gather_facts: no          # ปิด fact gathering ปกติ (เป็นของ Linux/Windows host ไม่ใช่ Network)
  connection: ansible.netcommon.network_cli

  tasks:
    - name: Run show version command
      cisco.ios.ios_command:
        commands:
          - show version
      register: version_output

    - name: Create output directory on control node
      ansible.builtin.file:
        path: "./output"
        state: directory
      delegate_to: localhost
      run_once: true

    - name: Save show version output to file per device
      ansible.builtin.copy:
        content: "{{ version_output.stdout[0] }}"
        dest: "./output/{{ inventory_hostname }}-show-version.txt"
      delegate_to: localhost

    - name: Display summary on screen
      ansible.builtin.debug:
        msg: "บันทึก show version ของ {{ inventory_hostname }} ({{ ansible_host }}) เรียบร้อย"
```

### คำอธิบายแต่ละ Keyword สำคัญ

| Keyword | ความหมาย |
|---|---|
| `hosts: network_devices` | รัน Playbook นี้กับทุก host ใน group `network_devices` (ตาม Inventory) |
| `gather_facts: no` | ปิดการเก็บ system facts แบบ default (ใช้กับ Linux/Windows) เพราะ Network device ใช้ `ios_facts` module แยก |
| `register: version_output` | เก็บผลลัพธ์จาก task ไว้ในตัวแปรชื่อ `version_output` เพื่อใช้ใน task ถัดไป |
| `delegate_to: localhost` | สั่งให้ task นี้รันบนเครื่อง Control Node เอง (ไม่ใช่บนอุปกรณ์ Network ที่ไม่มี filesystem แบบ Linux) |
| `run_once: true` | รัน task นี้แค่ครั้งเดียว ไม่ต้องรันซ้ำทุก host (สร้าง directory แค่ครั้งเดียวพอ) |
| `{{ inventory_hostname }}` | Ansible Variable มาตรฐาน = ชื่อ host ปัจจุบันตามที่ระบุใน Inventory (เช่น "CORE-SW1") |

### วิธีรันและ Output ที่คาดหวัง

```bash
ansible-playbook -i inventory.yaml playbooks/gather_show_version.yaml --ask-vault-pass
```

```
PLAY [Gather show version from all network devices] **************************

TASK [Run show version command] ************************************************
ok: [CORE-SW1]
ok: [CORE-SW2]
ok: [DIST-SW1]
ok: [DIST-SW2]
ok: [DIST-SW3]
ok: [DIST-SW4]

TASK [Create output directory on control node] *********************************
changed: [CORE-SW1]

TASK [Save show version output to file per device] *****************************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [Display summary on screen] ************************************************
ok: [CORE-SW1] => {
    "msg": "บันทึก show version ของ CORE-SW1 (10.10.99.10) เรียบร้อย"
}
ok: [CORE-SW2] => {
    "msg": "บันทึก show version ของ CORE-SW2 (10.10.99.11) เรียบร้อย"
}
...

PLAY RECAP **********************************************************************
CORE-SW1  : ok=4  changed=2  unreachable=0  failed=0  skipped=0
CORE-SW2  : ok=3  changed=1  unreachable=0  failed=0  skipped=0
DIST-SW1  : ok=3  changed=1  unreachable=0  failed=0  skipped=0
DIST-SW2  : ok=3  changed=1  unreachable=0  failed=0  skipped=0
DIST-SW3  : ok=3  changed=1  unreachable=0  failed=0  skipped=0
DIST-SW4  : ok=3  changed=1  unreachable=0  failed=0  skipped=0
```

**PLAY RECAP** คือสรุปผลลัพธ์ที่ต้องอ่านให้เป็นทุกครั้งที่รัน Ansible:
- `ok` = task ที่รันสำเร็จ (ไม่ว่าจะเปลี่ยนอะไรหรือไม่)
- `changed` = task ที่รันสำเร็จ **และทำให้เกิดการเปลี่ยนแปลงจริง** บนอุปกรณ์/ไฟล์
- `unreachable` = เชื่อมต่ออุปกรณ์ไม่ได้เลย (Network/SSH ปัญหา)
- `failed` = เชื่อมต่อได้แต่ task ล้มเหลว (เช่น คำสั่งผิด, permission denied)

---

## Step 227 — Ansible Playbook Push Config และแนวคิด Idempotency

### Idempotency คืออะไร และทำไมสำคัญที่สุดใน Automation

**Idempotency** หมายถึง "การรัน operation เดิมซ้ำกี่ครั้งก็ได้ผลลัพธ์สุดท้ายเหมือนกันเสมอ" —
ถ้า config มีอยู่แล้ว Ansible module (เช่น `ios_config`) จะ**ตรวจสอบก่อน** ว่า running-config มี
บรรทัดนั้นอยู่แล้วหรือไม่ ถ้ามีแล้วจะ**ไม่ push ซ้ำ** (รายงานผลเป็น `ok` ไม่ใช่ `changed`) — นี่คือ
ข้อแตกต่างสำคัญจาก Netmiko script ใน Step 223 ที่ถ้ารันซ้ำจะพยายาม push คำสั่งเดิมซ้ำทุกครั้ง
(ซึ่งสำหรับคำสั่ง `ntp server` ไม่เป็นปัญหาเพราะ IOS จัดการ duplicate เองอยู่แล้ว แต่คำสั่ง
บางประเภท เช่น `access-list` แบบ numbered ถ้า push ซ้ำจะเพิ่ม entry ซ้ำซ้อนจริง)

```
รันครั้งที่ 1:  Playbook ตรวจ running-config -> ไม่มี "ntp server ..." -> push เข้าไป -> changed
รันครั้งที่ 2:  Playbook ตรวจ running-config -> มี "ntp server ..." อยู่แล้ว -> ไม่ทำอะไร -> ok
รันครั้งที่ 3:  (เหมือนครั้งที่ 2 ทุกประการ) -> ok
```

นี่คือเหตุผลที่วิศวกรสามารถรัน Playbook เดิม**ซ้ำได้ทุกวันโดยไม่กลัวพัง** (เรียกว่า "Playbook
ปลอดภัยต่อการรันซ้ำ" — safe to re-run) ต่างจาก script Netmiko แบบ Step 223 ที่ควรเขียน logic
ตรวจสอบก่อน push เองถ้าต้องการ idempotency แบบเดียวกัน

### `playbooks/push_ntp_config.yaml` — Push NTP Config เดียวกับ Step 223 แต่ Idempotent

```yaml
---
- name: Push standardized NTP configuration idempotently
  hosts: network_devices
  gather_facts: no
  connection: ansible.netcommon.network_cli

  vars:
    ntp_servers:
      - { ip: "10.10.99.50", prefer: true }
      - { ip: "10.10.99.51", prefer: false }

  tasks:
    - name: Ensure NTP servers are configured
      cisco.ios.ios_config:
        lines:
          - "ntp server {{ item.ip }}{{ ' prefer' if item.prefer else '' }}"
      loop: "{{ ntp_servers }}"
      register: ntp_config_result

    - name: Save running-config to startup-config (write memory)
      cisco.ios.ios_config:
        save_when: modified     # save เฉพาะเมื่อ task ก่อนหน้าทำให้ config เปลี่ยนจริง
      when: ntp_config_result.changed

    - name: Report change status
      ansible.builtin.debug:
        msg: >
          {{ inventory_hostname }}:
          {{ 'มีการเปลี่ยนแปลง config (เพิ่ม NTP ใหม่)' if ntp_config_result.changed
             else 'ไม่มีการเปลี่ยนแปลง (config ตรงกับที่ต้องการอยู่แล้ว)' }}
```

### รันครั้งแรก (มีการเปลี่ยนแปลง)

```bash
$ ansible-playbook -i inventory.yaml playbooks/push_ntp_config.yaml --ask-vault-pass
```

```
TASK [Ensure NTP servers are configured] ****************************************
changed: [CORE-SW1] => (item={'ip': '10.10.99.50', 'prefer': True})
changed: [CORE-SW1] => (item={'ip': '10.10.99.51', 'prefer': False})
changed: [CORE-SW2] => (item={'ip': '10.10.99.50', 'prefer': True})
changed: [CORE-SW2] => (item={'ip': '10.10.99.51', 'prefer': False})
...

TASK [Save running-config to startup-config (write memory)] *********************
changed: [CORE-SW1]
changed: [CORE-SW2]
...

TASK [Report change status] *****************************************************
ok: [CORE-SW1] => { "msg": "CORE-SW1: มีการเปลี่ยนแปลง config (เพิ่ม NTP ใหม่)" }
...

PLAY RECAP ***********************************************************************
CORE-SW1  : ok=3  changed=2  unreachable=0  failed=0  skipped=0
```

### รันครั้งที่ 2 (ไม่มีอะไรเปลี่ยน — พิสูจน์ Idempotency)

```
TASK [Ensure NTP servers are configured] ****************************************
ok: [CORE-SW1] => (item={'ip': '10.10.99.50', 'prefer': True})
ok: [CORE-SW1] => (item={'ip': '10.10.99.51', 'prefer': False})
...

TASK [Save running-config to startup-config (write memory)] *********************
skipping: [CORE-SW1]

TASK [Report change status] *****************************************************
ok: [CORE-SW1] => { "msg": "CORE-SW1: ไม่มีการเปลี่ยนแปลง (config ตรงกับที่ต้องการอยู่แล้ว)" }

PLAY RECAP ***********************************************************************
CORE-SW1  : ok=3  changed=0  unreachable=0  failed=0  skipped=1
```

สังเกตว่า `changed=0` และ task บันทึก config ถูก `skipping` ไปเลยเพราะ `when: ntp_config_result.changed`
เป็น `false` — นี่คือหัวใจของ Idempotency ที่ทำให้ Ansible Playbook ปลอดภัยกว่าการเขียน raw script

---

## Step 228 — Jinja2 Templating สำหรับสร้าง Configuration

### ทำไมต้องใช้ Template

เมื่อต้อง config อุปกรณ์จำนวนมากที่มีโครงสร้างเหมือนกันแต่ค่าต่างกัน (เช่น hostname, IP แต่ละตัว)
การเขียน config ทีละไฟล์ด้วยมือไม่ scale — **Jinja2** คือ Template Engine ของ Python (ที่ Ansible
ใช้อยู่ภายในทุก module) ให้เราเขียน **Template (.j2)** ครั้งเดียว แล้ว render ออกมาเป็น config
จริงสำหรับอุปกรณ์แต่ละตัวโดยอัตโนมัติ

### ไฟล์ตัวแปร (`vars/dist_switches.yml`)

```yaml
devices:
  - hostname: DIST-SW1
    mgmt_ip: 10.10.99.12
    loopback0: 1.1.1.11
    uplink_interface: TenGigabitEthernet1/1/1
    uplink_description: "** Uplink to CORE-SW1 **"
  - hostname: DIST-SW2
    mgmt_ip: 10.10.99.13
    loopback0: 1.1.1.12
    uplink_interface: TenGigabitEthernet1/1/1
    uplink_description: "** Uplink to CORE-SW1 **"
  - hostname: DIST-SW3
    mgmt_ip: 10.10.99.14
    loopback0: 1.1.1.13
    uplink_interface: TenGigabitEthernet1/1/1
    uplink_description: "** Uplink to CORE-SW2 **"
  - hostname: DIST-SW4
    mgmt_ip: 10.10.99.15
    loopback0: 1.1.1.14
    uplink_interface: TenGigabitEthernet1/1/1
    uplink_description: "** Uplink to CORE-SW2 **"
```

### Jinja2 Template (`templates/dist_switch.j2`)

```jinja2
! ===============================================================
! Auto-generated configuration for {{ hostname }}
! Generated by Jinja2 Template - DO NOT EDIT MANUALLY
! ===============================================================
hostname {{ hostname }}
!
interface Vlan99
 description ** Management VLAN **
 ip address {{ mgmt_ip }} 255.255.255.0
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address {{ loopback0 }} 255.255.255.255
!
interface {{ uplink_interface }}
 description {{ uplink_description }}
 no switchport
{% if uplink_interface.startswith("TenGigabitEthernet") %}
 speed 10000
 duplex full
{% endif %}
 no shutdown
!
ntp server 10.10.99.50 prefer
ntp server 10.10.99.51
!
end
```

### คำอธิบาย Jinja2 Syntax ที่ใช้

| Syntax | ความหมาย |
|---|---|
| `{{ variable }}` | แทนค่าตัวแปรตรงๆ (Expression) |
| `{% if condition %} ... {% endif %}` | เงื่อนไข (Control Structure) — เขียนเฉพาะบรรทัดนี้ถ้าเงื่อนไขจริง |
| `{% for item in list %} ... {% endfor %}` | Loop วนซ้ำ (ใช้เมื่อต้อง render list ของบางอย่างซ้ำๆในไฟล์เดียว) |
| `{{ variable.startswith(...) }}` | เรียกใช้ Python string method ได้ตรงๆใน Template |

### Python Script สำหรับ Render Template (ใช้ library `jinja2` ตรงๆ ไม่ผ่าน Ansible)

```python
#!/usr/bin/env python3
"""
render_configs.py
โหลดตัวแปรจาก vars/dist_switches.yml แล้ว render Jinja2 Template
ออกมาเป็น config file แยกทีละอุปกรณ์ใน generated_configs/
"""

import yaml
from jinja2 import Environment, FileSystemLoader
import os

env = Environment(loader=FileSystemLoader("templates/"))
template = env.get_template("dist_switch.j2")

with open("vars/dist_switches.yml") as f:
    data = yaml.safe_load(f)

os.makedirs("generated_configs", exist_ok=True)

for device in data["devices"]:
    rendered_config = template.render(**device)
    out_path = f"generated_configs/{device['hostname']}.cfg"
    with open(out_path, "w") as out_file:
        out_file.write(rendered_config)
    print(f"[*] Generated: {out_path}")
```

```bash
pip install jinja2 pyyaml
python3 render_configs.py
```

```
[*] Generated: generated_configs/DIST-SW1.cfg
[*] Generated: generated_configs/DIST-SW2.cfg
[*] Generated: generated_configs/DIST-SW3.cfg
[*] Generated: generated_configs/DIST-SW4.cfg
```

### ตัวอย่างผลลัพธ์ที่ Render ออกมา (`generated_configs/DIST-SW1.cfg`)

```
! ===============================================================
! Auto-generated configuration for DIST-SW1
! Generated by Jinja2 Template - DO NOT EDIT MANUALLY
! ===============================================================
hostname DIST-SW1
!
interface Vlan99
 description ** Management VLAN **
 ip address 10.10.99.12 255.255.255.0
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.11 255.255.255.255
!
interface TenGigabitEthernet1/1/1
 description ** Uplink to CORE-SW1 **
 no switchport
 speed 10000
 duplex full
 no shutdown
!
ntp server 10.10.99.50 prefer
ntp server 10.10.99.51
!
end
```

> **การใช้งานจริง**: ไฟล์ config ที่ render ออกมาแบบนี้สามารถนำไป push ต่อด้วย Netmiko
> (`send_config_set` จากไฟล์) หรือ Ansible (`cisco.ios.ios_config` กับ parameter `src:`) ได้ทันที
> — เทคนิคนี้คือรากฐานของการทำ **Network-as-Code**: Template + Variable file (เก็บใน Git) คือ
> "Source of Truth" ตัวจริงของ Config ทั้งเครือข่าย ไม่ใช่ running-config บนอุปกรณ์อีกต่อไป

---

## Step 229 — แอบดู NETCONF/RESTCONF (Model-Driven API)

> การเจาะลึก NETCONF/RESTCONF/YANG Data Model แบบเต็มรูปแบบ (รวม `ncclient`, YANG Explorer,
> Telemetry แบบ Model-Driven) จะอยู่ใน **Part 49** — Step นี้แค่แนะนำภาพรวมให้เห็นว่าทำไม
> เทคโนโลยีนี้จึงเป็นอนาคตของ Network Automation

### ปัญหาของ CLI Screen-Scraping (สิ่งที่ Netmiko/Ansible `ios_command` ทำอยู่)

Netmiko และ Ansible module อย่าง `ios_command` (Step 222, 226) ยังทำงานโดย **ส่งคำสั่ง CLI
ผ่าน SSH แล้วอ่าน text output กลับมา** — นี่เรียกว่า **Screen-Scraping**: โปรแกรมต้อง parse
ข้อความที่ถูกออกแบบมาให้ "คนอ่าน" ซึ่งมีปัญหา:

- Format output อาจเปลี่ยนไปตาม IOS version ทำให้ regex parsing พังโดยไม่ทันตั้งตัว
- ไม่มี Schema ตายตัว — ต้องเขียน parser เองสำหรับทุกคำสั่งที่ต้องการข้อมูล
- ช้า (ต้อง spawn SSH session, รอ prompt, ส่งคำสั่งทีละบรรทัด)

### Model-Driven API คืออะไร

**NETCONF** (RFC 6241, port 830) และ **RESTCONF** (RFC 8040, port 443 ผ่าน HTTPS) คือ Protocol
ที่ให้เราสื่อสารกับอุปกรณ์ Cisco IOS-XE ด้วย **Structured Data (XML สำหรับ NETCONF, JSON/XML
สำหรับ RESTCONF)** ที่มี **Schema ตายตัวจาก YANG Data Model** — ไม่ต้อง parse text อีกต่อไป
โปรแกรมอ่าน JSON/XML ได้ตรงๆ

```
CLI/Netmiko แบบเดิม:                        RESTCONF แบบใหม่:
Router# show interfaces Gi0/1               GET /restconf/data/ietf-interfaces:
   ... text หลายบรรทัดผสมกัน ...                interfaces-state/interface=GigabitEthernet0/1
   ต้อง regex เพื่อดึงแค่ speed/duplex           Response (JSON):
                                              {
                                                "ietf-interfaces:interface": {
                                                  "name": "GigabitEthernet0/1",
                                                  "oper-status": "up",
                                                  "speed": 1000000
                                                }
                                              }
                                              -> อ่านตรงๆ ไม่ต้อง parse text เลย
```

### เปิดใช้งาน RESTCONF บน Cisco IOS-XE

```
CORE-SW1(config)# restconf
CORE-SW1(config)# ip http secure-server
CORE-SW1(config)# username apiuser privilege 15 secret ApiP@ssw0rd!
```

### ตัวอย่าง `curl` เรียก RESTCONF เพื่อดึงสถานะ Interface

```bash
curl -k -X GET \
  --header "Accept: application/yang-data+json" \
  --header "Authorization: Basic $(echo -n 'apiuser:ApiP@ssw0rd!' | base64)" \
  https://10.10.99.10/restconf/data/ietf-interfaces:interfaces-state/interface=Vlan99
```

**ตัวอย่าง Response (JSON):**

```json
{
  "ietf-interfaces:interface": {
    "name": "Vlan99",
    "admin-status": "up",
    "oper-status": "up",
    "phys-address": "0050.56b3.aa11",
    "statistics": {
      "in-octets": "18293841",
      "out-octets": "9182734"
    }
  }
}
```

### ตัวอย่าง `curl` แบบ POST เพื่อเปลี่ยน Config (RESTCONF)

```bash
curl -k -X PATCH \
  --header "Content-Type: application/yang-data+json" \
  --header "Authorization: Basic $(echo -n 'apiuser:ApiP@ssw0rd!' | base64)" \
  --data '{"ietf-interfaces:interface": {"name": "Loopback99", "description": "Created via RESTCONF"}}' \
  https://10.10.99.10/restconf/data/ietf-interfaces:interfaces/interface=Loopback99
```

### NETCONF vs RESTCONF สรุปเทียบ

| ประเด็น | NETCONF | RESTCONF |
|---|---|---|
| Transport | SSH (port 830) | HTTPS (port 443) |
| Data Format | XML เท่านั้น | JSON หรือ XML |
| Operation | `<get>`, `<edit-config>`, `<commit>` (มี transaction/candidate config) | HTTP Verb: `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| เหมาะกับ | Config เชิงลึกที่ต้องการ transaction/rollback | Integration กับระบบ Web/REST API ทั่วไป (เข้าถึงง่ายกว่า) |
| YANG Model | ใช้ร่วมกัน | ใช้ร่วมกัน |

> ทั้ง NETCONF และ RESTCONF ใช้ **YANG Data Model** ตัวเดียวกันเป็นแหล่งอ้างอิง Schema —
> รายละเอียดการเขียน YANG Model เอง, การใช้ library `ncclient` (Python สำหรับ NETCONF),
> และ Model-Driven Telemetry จะอยู่ใน **Part 49** ตามที่ประกาศไว้ใน Part 1 Step 9

---

## Step 230 — Lab เต็มรูปแบบ: Automation Project (Backup + Push + Verify)

นี่คือ Lab สรุปของ Part 23: สร้าง **Ansible Project ที่สมบูรณ์** ทำ 3 อย่างอัตโนมัติในการรัน
ครั้งเดียว:

1. **Backup** running-config จาก CORE-SW1, CORE-SW2, DIST-SW1-4 เป็นไฟล์ที่มี timestamp
2. **Push** มาตรฐาน NTP + Syslog Config (อ้างอิงจาก Part 17 — NTP/Syslog/SNMP Monitoring) แบบ
   idempotent
3. **Verify** ผลลัพธ์ด้วยคำสั่ง `show` แล้วบันทึกผลไว้เป็นหลักฐาน

### โครงสร้าง Directory ของ Project

```
lab-automation/
├── inventory.yaml                     # Step 225
├── ansible.cfg                        # ค่า config ของ Ansible เอง
├── group_vars/
│   ├── network_devices.yml            # ตัวแปรทั่วไป (ไม่ลับ)
│   └── network_devices_vault.yml      # credential เข้ารหัสด้วย Ansible Vault
├── backups/                           # ปลายทางเก็บ backup config (สร้างอัตโนมัติ)
│   └── CORE-SW1_2026-09-26_14-30-00.cfg
│   └── ...
├── verify_output/                     # ปลายทางเก็บผล verify (สร้างอัตโนมัติ)
│   └── CORE-SW1_verify.txt
│   └── ...
└── site.yaml                          # Playbook หลักที่รวมทุกขั้นตอน
```

### `ansible.cfg`

```ini
[defaults]
inventory = inventory.yaml
host_key_checking = False
retry_files_enabled = False
timeout = 30

[persistent_connection]
command_timeout = 60
connect_timeout = 60
```

### `site.yaml` — Playbook หลักของ Automation Project

```yaml
---
- name: "Automation Project - Backup, Standardize Config, and Verify"
  hosts: network_devices
  gather_facts: no
  connection: ansible.netcommon.network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y-%m-%d_%H-%M-%S') }}"
    ntp_servers:
      - { ip: "10.10.99.50", prefer: true }
      - { ip: "10.10.99.51", prefer: false }
    syslog_server: "10.10.99.52"

  tasks:
    # ============================================================
    # ขั้นตอนที่ 1: Backup running-config พร้อม timestamp
    # ============================================================
    - name: "[1/3] Ensure backup directory exists"
      ansible.builtin.file:
        path: "./backups"
        state: directory
      delegate_to: localhost
      run_once: true

    - name: "[1/3] Retrieve running-config from device"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: running_config_raw

    - name: "[1/3] Save running-config backup with timestamp"
      ansible.builtin.copy:
        content: "{{ running_config_raw.stdout[0] }}"
        dest: "./backups/{{ inventory_hostname }}_{{ timestamp }}.cfg"
      delegate_to: localhost

    # ============================================================
    # ขั้นตอนที่ 2: Push มาตรฐาน NTP + Syslog Config (Idempotent)
    # ============================================================
    - name: "[2/3] Ensure NTP servers are configured"
      cisco.ios.ios_config:
        lines:
          - "ntp server {{ item.ip }}{{ ' prefer' if item.prefer else '' }}"
      loop: "{{ ntp_servers }}"
      register: ntp_result

    - name: "[2/3] Ensure Syslog server is configured"
      cisco.ios.ios_config:
        lines:
          - "logging host {{ syslog_server }}"
          - "logging trap informational"
          - "logging source-interface Vlan99"
      register: syslog_result

    - name: "[2/3] Save running-config to startup-config if changed"
      cisco.ios.ios_config:
        save_when: modified
      when: ntp_result.changed or syslog_result.changed

    # ============================================================
    # ขั้นตอนที่ 3: Verify การเปลี่ยนแปลงด้วย show command
    # ============================================================
    - name: "[3/3] Ensure verify output directory exists"
      ansible.builtin.file:
        path: "./verify_output"
        state: directory
      delegate_to: localhost
      run_once: true

    - name: "[3/3] Verify NTP associations"
      cisco.ios.ios_command:
        commands:
          - show ntp associations
          - show logging | include Trap logging
      register: verify_output

    - name: "[3/3] Save verification output to file"
      ansible.builtin.copy:
        content: |
          === Verification Report for {{ inventory_hostname }} ({{ timestamp }}) ===
          NTP config changed this run: {{ ntp_result.changed }}
          Syslog config changed this run: {{ syslog_result.changed }}

          --- show ntp associations ---
          {{ verify_output.stdout[0] }}

          --- show logging | include Trap logging ---
          {{ verify_output.stdout[1] }}
        dest: "./verify_output/{{ inventory_hostname }}_verify.txt"
      delegate_to: localhost

    - name: "[3/3] Print final summary to screen"
      ansible.builtin.debug:
        msg: >
          {{ inventory_hostname }}: Backup ✅ |
          NTP {{ 'เปลี่ยน' if ntp_result.changed else 'ไม่เปลี่ยน' }} |
          Syslog {{ 'เปลี่ยน' if syslog_result.changed else 'ไม่เปลี่ยน' }} |
          Verify ✅ บันทึกที่ ./verify_output/{{ inventory_hostname }}_verify.txt
```

### วิธีรัน Automation Project ทั้งหมด

```bash
cd lab-automation/
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

### ตัวอย่าง Terminal Output จริง (สรุปแบบย่อ)

```
Vault password:

PLAY [Automation Project - Backup, Standardize Config, and Verify] ************

TASK [[1/3] Ensure backup directory exists] ************************************
ok: [CORE-SW1]

TASK [[1/3] Retrieve running-config from device] ********************************
ok: [CORE-SW1]
ok: [CORE-SW2]
ok: [DIST-SW1]
ok: [DIST-SW2]
ok: [DIST-SW3]
ok: [DIST-SW4]

TASK [[1/3] Save running-config backup with timestamp] ***************************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [[2/3] Ensure NTP servers are configured] ***********************************
changed: [CORE-SW1] => (item={'ip': '10.10.99.50', 'prefer': True})
ok: [CORE-SW2] => (item={'ip': '10.10.99.50', 'prefer': True})
changed: [DIST-SW1] => (item={'ip': '10.10.99.51', 'prefer': False})
...

TASK [[2/3] Ensure Syslog server is configured] **********************************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [[2/3] Save running-config to startup-config if changed] ********************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [[3/3] Ensure verify output directory exists] *******************************
ok: [CORE-SW1]

TASK [[3/3] Verify NTP associations] *********************************************
ok: [CORE-SW1]
ok: [CORE-SW2]
ok: [DIST-SW1]
ok: [DIST-SW2]
ok: [DIST-SW3]
ok: [DIST-SW4]

TASK [[3/3] Save verification output to file] ************************************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [[3/3] Print final summary to screen] ****************************************
ok: [CORE-SW1] => {
    "msg": "CORE-SW1: Backup ✅ | NTP เปลี่ยน | Syslog เปลี่ยน | Verify ✅ บันทึกที่ ./verify_output/CORE-SW1_verify.txt"
}
ok: [CORE-SW2] => {
    "msg": "CORE-SW2: Backup ✅ | NTP ไม่เปลี่ยน | Syslog เปลี่ยน | Verify ✅ บันทึกที่ ./verify_output/CORE-SW2_verify.txt"
}
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=9  changed=5  unreachable=0  failed=0  skipped=0
CORE-SW2  : ok=9  changed=4  unreachable=0  failed=0  skipped=0
DIST-SW1  : ok=9  changed=5  unreachable=0  failed=0  skipped=0
DIST-SW2  : ok=9  changed=5  unreachable=0  failed=0  skipped=0
DIST-SW3  : ok=9  changed=5  unreachable=0  failed=0  skipped=0
DIST-SW4  : ok=9  changed=5  unreachable=0  failed=0  skipped=0
```

### ตัวอย่างเนื้อหาไฟล์ Verify ที่ได้ (`verify_output/CORE-SW1_verify.txt`)

```
=== Verification Report for CORE-SW1 (2026-09-26_14-30-00) ===
NTP config changed this run: True
Syslog config changed this run: True

--- show ntp associations ---
      address         ref clock       st   when   poll reach  delay  offset   disp
 ~10.10.99.50     .LOCL.           1     12     64   377   0.482   0.023  0.891
 *10.10.99.51     .LOCL.           1     14     64   377   0.501  -0.011  0.734
 * sys.peer, # selected, + candidate, - outlyer, x falseticker, ~ configured

--- show logging | include Trap logging ---
    Trap logging: level informational, 47 message lines logged
```

### เมื่อรันซ้ำครั้งที่ 2 (พิสูจน์ Idempotency ของ Automation Project ทั้งชุด)

```
PLAY RECAP ***************************************************************************
CORE-SW1  : ok=9  changed=3  unreachable=0  failed=0  skipped=0
```

`changed` ลดจาก 5 เหลือ 3 เพราะ task backup และ verify ยังคง `changed` เสมอ (สร้างไฟล์ใหม่ทุก
ครั้งตาม timestamp) แต่ task NTP/Syslog config (`ntp_result`, `syslog_result`) และ save
startup-config จะกลายเป็น `ok` (ไม่เปลี่ยน) เพราะ config มาตรฐานมีอยู่แล้ว — นี่คือพฤติกรรมที่
ต้องการจาก Automation ระดับ Production: **รันซ้ำได้ทุกวันอย่างปลอดภัย (safe to re-run)** โดยไม่
สร้างผลข้างเคียงที่ไม่คาดคิด

---

## แบบฝึกหัดทวนความเข้าใจ Part 23

1. เพราะเหตุใดการ config อุปกรณ์เครือข่ายด้วยมือ (Manual CLI) ใน Enterprise ขนาดใหญ่จึงมีปัญหา
   "Configuration Drift" และ Automation ช่วยแก้ปัญหานี้อย่างไร?
2. ในการเขียน Python Script ด้วย Netmiko เพื่อ push config ไปยังอุปกรณ์หลายตัว เพราะเหตุใดจึง
   ต้องมี Error Handling **แยกทีละอุปกรณ์** (try/except รอบ loop) แทนที่จะครอบ try/except รอบ
   loop ทั้งหมดครั้งเดียว?
3. **Idempotency** ใน Ansible คืออะไร และแตกต่างจากการรัน Netmiko Script ธรรมดาซ้ำๆอย่างไร?
   ยกตัวอย่างผลลัพธ์ `changed` ที่ควรเห็นเมื่อรัน Playbook เดิมเป็นครั้งที่ 2
4. ความแตกต่างหลักระหว่าง NETCONF และ RESTCONF คืออะไร (Transport, Data Format) และทั้งสอง
   Protocol นี้ใช้ Data Model ร่วมกันชื่อว่าอะไร?
5. เพราะเหตุใดจึงไม่ควรเก็บ password ของอุปกรณ์ Network แบบ plaintext ไว้ใน Ansible
   `group_vars` file ที่ commit เข้า Git repository ควรใช้เครื่องมืออะไรแทน?

**เฉลย:**

1. Configuration Drift เกิดจากวิศวกรหลายคนแก้ config มืออุปกรณ์ทีละตัวไปเรื่อยๆโดยไม่มี
   Source of Truth กลาง ทำให้ config อุปกรณ์ที่ควรเหมือนกัน (เช่น DIST-SW1-4) ไม่ตรงกันในที่สุด
   Automation แก้ปัญหานี้โดยให้ Script/Playbook เดียวเป็นตัวกำหนด config ทั้งหมด (Single Source
   of Truth ที่เก็บใน Git) และ push ให้ทุกอุปกรณ์เหมือนกันทุกครั้งที่รัน
2. เพราะถ้าครอบ try/except รอบ loop ทั้งหมดครั้งเดียว การเชื่อมต่อล้มเหลวของอุปกรณ์ตัวแรก (เช่น
   DIST-SW3 timeout) จะทำให้โปรแกรม raise exception และหยุดทำงานทั้งหมด ไม่ push ต่อให้อุปกรณ์
   ตัวที่เหลือเลย ซึ่งไม่เหมาะกับงาน production ที่ต้องการให้อุปกรณ์ที่เชื่อมต่อได้สำเร็จทั้งหมด
   ถูก push แม้อุปกรณ์บางตัวจะมีปัญหา
3. Idempotency คือคุณสมบัติที่รัน operation เดิมซ้ำกี่ครั้งก็ได้ผลลัพธ์สุดท้ายเหมือนกัน — Ansible
   module อย่าง `ios_config` จะตรวจสอบ running-config ก่อนว่ามี config บรรทัดนั้นอยู่แล้วหรือไม่
   ถ้ามีแล้วจะไม่ push ซ้ำ (รายงานเป็น `ok` ไม่ใช่ `changed`) ต่างจาก Netmiko script ธรรมดาที่จะ
   พยายาม push คำสั่งเดิมซ้ำทุกครั้งที่รัน ไม่ตรวจสอบก่อน (เว้นแต่จะเขียน logic ตรวจสอบเอง)
   ตัวอย่างผลลัพธ์: รันครั้งที่ 2 ควรเห็น `changed=0` (หรือน้อยลง) ใน PLAY RECAP สำหรับ task ที่
   ตรวจแล้วว่า config มีอยู่แล้ว
4. NETCONF ใช้ SSH (port 830) และรองรับเฉพาะ XML ส่วน RESTCONF ใช้ HTTPS (port 443) และรองรับ
   ทั้ง JSON และ XML — ทั้งสอง Protocol ใช้ **YANG Data Model** ร่วมกันเป็นแหล่งอ้างอิง Schema
5. เพราะไฟล์ที่ commit เข้า Git repository จะถูกเก็บไว้ใน history ตลอดไป (แม้ลบทิ้งทีหลังก็ยัง
   ค้นเจอใน commit เก่าได้) ทำให้ password รั่วไหลสู่ทุกคนที่มีสิทธิ์เข้าถึง repository —
   ควรใช้ **Ansible Vault** เข้ารหัสไฟล์ credential หรือใช้ **Secret Manager** ระดับ Enterprise
   (HashiCorp Vault, AWS Secrets Manager) แทนการเก็บ plaintext

---

## สรุป Part 23

Part นี้คือจุดเปลี่ยนสำคัญของหลักสูตรจาก "วิศวกรที่พิมพ์คำสั่ง CLI" ไปสู่ "วิศวกรที่เขียนโปรแกรม
ควบคุมเครือข่าย" — เราเริ่มจากปัญหาของ Manual CLI และภาพรวม Automation Stack ทั้ง 5 Level, เขียน
Python Script จริงด้วย Netmiko (ทั้ง read-only และ push config พร้อม error handling),
เรียนรู้ Ansible ตั้งแต่ Inventory, Ansible Vault, Playbook พื้นฐาน (`ios_command`) และ Playbook
Push Config แบบ Idempotent (`ios_config`), ทำ Jinja2 Templating เพื่อ generate config อัตโนมัติ,
แอบดู NETCONF/RESTCONF เป็นตัวอย่าง Model-Driven API, และปิดท้ายด้วย Automation Project เต็ม
รูปแบบที่ Backup + Push Config + Verify ในการรันครั้งเดียว ✅

ทักษะเหล่านี้คือพื้นฐานที่จำเป็นสำหรับ **DevNet Associate/Professional** และเป็นหัวข้อบังคับใน
CCNP ENCOR สมัยใหม่ — Part ต่อไปเราจะกลับไปที่โลก Data Plane เพื่อเรียนรู้ **QoS (Quality of
Service)**: การจัดลำดับความสำคัญของ traffic เพื่อรองรับ VoIP/Video ที่ต้องการ latency ต่ำ
บน Network เดียวกันกับ traffic ทั่วไป

**ไปต่อ:** [Part 24 — QoS Fundamentals →](part-024-qos-fundamentals.md)
