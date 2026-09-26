# Part 48 — Network Automation: Ansible for Network Engineers (Advanced)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 471–480 จาก 1000**

> ต่อจาก [Part 47 — Network Automation: Python Advanced](part-047-automation-python-advanced.md)
> ที่เราเจาะลึก Python ระดับ Advanced (NAPALM, Nornir, Netmiko แบบ concurrent/threading,
> การเขียน Test Framework) มาถึง Part นี้เราจะกลับมาที่ **Ansible** ซึ่งแนะนำแบบพื้นฐานไปแล้วใน
> [Part 23](part-023-automation-basics.md) (Inventory, `ios_command`/`ios_config`, Vault เบื้องต้น,
> Jinja2 เบื้องต้น) แล้วยกระดับขึ้นไปสู่โครงสร้างระดับ **Production-grade Ansible Project** ที่ทีม
> NetOps จริงใช้กันในองค์กร — Roles, Galaxy Collections, Handlers, Tags, Loop/Conditional ขั้นสูง,
> Custom Jinja2 Filter, Vault แบบ Multi-Environment, Check Mode, และภาพรวม AWX/Ansible Automation
> Platform (AAP) ปิดท้ายด้วย Lab เต็มรูปแบบ "day2-ops" ที่รวมทุกเทคนิคเข้าด้วยกัน

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 471 | Ansible Roles — จัดโครงสร้าง Playbook แบบเดิมของ Part 23 ให้เป็น Role ที่ reuse ได้ |
| 472 | Ansible Galaxy — ติดตั้งและ pin เวอร์ชัน Collection ด้วย `requirements.yml` |
| 473 | Handlers และ `notify` — สั่ง `save running-config` เฉพาะเมื่อมีการเปลี่ยนแปลงจริง |
| 474 | Ansible Tags — จัดโครงสร้าง Playbook ใหญ่ให้เลือกรันเฉพาะส่วนได้ (`--tags ntp`, `--tags security`) |
| 475 | Conditionals (`when`) และ Loops (`loop`/`with_items`) — สร้าง VLAN ทั้ง 5 ตัวจาก List เดียว |
| 476 | Custom Jinja2 Filter และ Templating ขั้นสูง — Render Full Device Config จาก `host_vars` |
| 477 | Ansible Vault เจาะลึก — Vault-ID, Multi-Environment (dev/prod), `edit`/`rekey` |
| 478 | Idempotency Verification ด้วย `--check`/`--diff` — Dry-run ก่อน Apply จริงเสมอ |
| 479 | AWX / Ansible Automation Platform (AAP) — จาก Laptop สู่ Platform ระดับทีม NetOps |
| 480 | Lab เต็มรูปแบบ: โปรเจกต์ "day2-ops" รวม Role + Tag + Handler + Loop + Vault ครบวงจร |

---

## Step 471 — Ansible Roles: จัดโครงสร้าง Playbook ให้เป็น Role

### ปัญหาของ Playbook แบบ Flat File (สิ่งที่เราทำใน Part 23)

Playbook อย่าง `push_ntp_config.yaml` ใน Part 23 Step 227 ใช้งานได้ดีสำหรับ Lab ขนาดเล็ก แต่เมื่อ
โปรเจกต์เติบโตขึ้น (ต้อง config ทั้ง NTP, Syslog, AAA, VLAN, ACL ในโปรเจกต์เดียว) การเขียนทุกอย่าง
ไว้ในไฟล์ `site.yaml` ไฟล์เดียวจะเกิดปัญหา:

| ปัญหา | รายละเอียด |
|---|---|
| **Reusability ต่ำ** | อยากเอา task NTP ไปใช้กับโปรเจกต์อื่น ต้อง copy-paste YAML block ทั้งหมด |
| **Testability ต่ำ** | ทดสอบ task เฉพาะเรื่อง NTP อย่างเดียวไม่ได้ ต้องรันทั้งไฟล์ |
| **Readability ต่ำ** | ไฟล์เดียวยาวหลายพันบรรทัด หา task ที่ต้องการแก้ไขยาก |
| **Team Collaboration ยาก** | วิศวกรหลายคนแก้ไฟล์เดียวกันพร้อมกัน = merge conflict บ่อย |
| **ไม่มี Standard Structure** | แต่ละคนจัดโครงสร้างไฟล์ต่างกัน ทีมใหม่เข้าใจโปรเจกต์คนอื่นยาก |

**Ansible Role** คือหน่วยการจัดโครงสร้างมาตรฐานของ Ansible ที่แก้ปัญหาทั้งหมดข้างบน — แบ่ง
Playbook ออกเป็นโฟลเดอร์ตาม "หน้าที่" (function) แต่ละหน้าที่มี Task/Template/Default Variable/
Handler ของตัวเอง แยกจากกันชัดเจน และสามารถ **นำไปใช้ซ้ำ (reuse)** ในโปรเจกต์ไหนก็ได้เพียง
copy โฟลเดอร์ทั้ง Role ไป

### โครงสร้าง Directory มาตรฐานของ Ansible Role

```
roles/
└── ntp-config/
    ├── defaults/
    │   └── main.yaml        # ค่า default variable ของ Role (override ได้จากที่อื่น)
    ├── tasks/
    │   └── main.yaml        # Task ทั้งหมดของ Role นี้ (entry point)
    ├── templates/
    │   └── ntp_syslog.j2     # Jinja2 Template ที่ Role นี้ใช้
    ├── handlers/
    │   └── main.yaml        # Handler ของ Role นี้ (รายละเอียดเต็มใน Step 473)
    ├── meta/
    │   └── main.yaml        # Metadata: dependency กับ Role อื่น, author, platform ที่รองรับ
    └── vars/
        └── main.yaml        # ตัวแปรที่ไม่ควร override (ค่าคงที่ภายใน Role)
```

> **กฎการค้นหาไฟล์ของ Ansible**: เมื่อ Playbook สั่ง `roles: - ntp-config` Ansible จะโหลด
> `tasks/main.yaml` เป็น entry point เสมอ, โหลด `defaults/main.yaml` เป็นตัวแปรเริ่มต้น (priority
> ต่ำสุด — override ได้ง่ายจาก `group_vars`/`host_vars`/`vars:` ใน Playbook), และโหลด
> `handlers/main.yaml` ให้พร้อมใช้กับ `notify:` โดยอัตโนมัติ ไม่ต้อง `include` เอง

### ใช้ `ansible-galaxy` สร้างโครงสร้าง Role ให้อัตโนมัติ

```bash
mkdir -p day2-ops/roles
cd day2-ops/roles
ansible-galaxy role init ntp-config
```

```
- Role ntp-config was created successfully
```

คำสั่งนี้จะสร้างโฟลเดอร์ทั้งหมดที่จำเป็นให้อัตโนมัติ (รวม `README.md`, `tests/` ที่เราจะลบทิ้งได้
ถ้าไม่ใช้) — ประหยัดเวลากว่าสร้างมือทีละโฟลเดอร์

### แปลง NTP/Syslog Config จาก Part 17 ให้เป็น Role `ntp-config`

**`roles/ntp-config/defaults/main.yaml`** — ค่า Default ที่ Role นี้ใช้ (override ได้เสมอ)

```yaml
---
# roles/ntp-config/defaults/main.yaml
# ค่า default ของ Role ntp-config — ตรงกับมาตรฐาน Lab ใน docs/part-017-ntp-syslog-snmp.md
ntp_servers:
  - { ip: "10.10.99.50", prefer: true }    # NOC-SRV (Stratum 2 ของ Lab)
  - { ip: "10.10.99.51", prefer: false }   # Secondary NTP (redundancy)

syslog_server: "10.10.99.50"               # NOC-SRV ทำหน้าที่ Syslog Collector ด้วย (Part 17 Step 165)
syslog_trap_level: "informational"
syslog_source_interface: "Loopback0"

ntp_source_interface: "Loopback0"
```

**`roles/ntp-config/templates/ntp_syslog.j2`** — Template สำหรับ config บรรทัดที่ต้อง render

```jinja2
{# roles/ntp-config/templates/ntp_syslog.j2 #}
{% for server in ntp_servers %}
ntp server {{ server.ip }}{{ ' prefer' if server.prefer else '' }}
{% endfor %}
ntp source {{ ntp_source_interface }}
!
logging host {{ syslog_server }}
logging trap {{ syslog_trap_level }}
logging source-interface {{ syslog_source_interface }}
logging buffered 16384 {{ syslog_trap_level }}
```

**`roles/ntp-config/tasks/main.yaml`** — Task หลักของ Role (ใช้ `src:` ให้ module render Template ให้อัตโนมัติ)

```yaml
---
# roles/ntp-config/tasks/main.yaml
- name: "[ntp-config] Push NTP + Syslog config via rendered template"
  cisco.ios.ios_config:
    src: "ntp_syslog.j2"
  register: ntp_syslog_result
  tags: ["ntp", "syslog"]

- name: "[ntp-config] Debug change status"
  ansible.builtin.debug:
    msg: >
      {{ inventory_hostname }}: NTP/Syslog config
      {{ 'เปลี่ยนแปลง' if ntp_syslog_result.changed else 'ไม่มีการเปลี่ยนแปลง (idempotent)' }}
  tags: ["ntp", "syslog"]
```

> **จุดสำคัญ**: `cisco.ios.ios_config` รองรับ parameter `src:` ที่รับ**พาธของ Jinja2 Template**
> โดยตรง — module จะ render Template ด้วยตัวแปรที่ Ansible มองเห็นอยู่ (`ntp_servers`,
> `syslog_server` จาก `defaults/main.yaml`) ให้อัตโนมัติ **ก่อน** เทียบกับ running-config เพื่อ
> ตัดสินใจว่า `changed` หรือไม่ — ไม่ต้องเขียน task แยกไป render ไฟล์ก่อนเหมือนวิธี Python ล้วน
> ใน Part 23 Step 228 อีกต่อไป

### เรียกใช้ Role จาก Playbook หลัก (`site.yaml`)

```yaml
---
# site.yaml
- name: "Day2-Ops: Push standardized config to all network devices"
  hosts: network_devices
  gather_facts: no
  connection: ansible.netcommon.network_cli

  roles:
    - ntp-config
```

```bash
cd day2-ops/
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

โครงสร้างไฟล์ทั้งโปรเจกต์ ณ จุดนี้:

```
day2-ops/
├── ansible.cfg
├── inventory.yaml
├── group_vars/
│   ├── network_devices.yml
│   └── network_devices_vault.yml
├── roles/
│   └── ntp-config/
│       ├── defaults/main.yaml
│       ├── tasks/main.yaml
│       └── templates/ntp_syslog.j2
└── site.yaml
```

Role เดียวนี้สามารถนำไป `roles: - ntp-config` ในโปรเจกต์อื่นได้ทันที เพียงแค่มีตัวแปร
`ntp_servers`/`syslog_server` ที่เหมาะสมของ Site นั้น — นี่คือหัวใจของการทำ Ansible ให้ **Scale
ได้ในระดับองค์กร** ไม่ใช่แค่ Lab เดียว

---

## Step 472 — Ansible Galaxy: ติดตั้งและ Pin เวอร์ชัน Collection

### Ansible Galaxy คืออะไร

**Ansible Galaxy** (galaxy.ansible.com) คือ Public Registry ของ Ansible Collection/Role ที่ทั้ง
Community และ Vendor (Cisco, Juniper, Arista, AWS ฯลฯ) เผยแพร่ Module สำเร็จรูปให้ใช้งาน — ตั้งแต่
Ansible 2.10 เป็นต้นมา Module สำหรับอุปกรณ์ Network **ไม่รวมมาใน Ansible core อีกต่อไป** ต้อง
ติดตั้ง Collection เพิ่มเองเสมอ (ตามที่แนะนำสั้นๆใน Part 23 Step 224)

### ติดตั้ง Collection แบบ Ad-hoc (วิธีที่ใช้ใน Part 23)

```bash
ansible-galaxy collection install cisco.ios
ansible-galaxy collection install ansible.netcommon
```

ปัญหาของวิธีนี้: ติดตั้ง**เวอร์ชันล่าสุด** เสมอ — ถ้าเพื่อนร่วมทีมรัน `ansible-galaxy collection
install cisco.ios` ในวันอื่น อาจได้เวอร์ชันที่ต่างกัน ทำให้ Playbook ทำงานไม่เหมือนกันระหว่างเครื่อง
(เรียกว่า "Works on my machine" syndrome) — ทางแก้ระดับ Production คือ **Pin เวอร์ชัน** ด้วยไฟล์
`requirements.yml`

### `requirements.yml` — Pin เวอร์ชัน Collection ให้ทุกคนในทีมใช้ตรงกัน

```yaml
---
# requirements.yml
# ระบุ Collection ที่โปรเจกต์นี้ต้องใช้ พร้อม version constraint ที่แน่นอน
collections:
  - name: cisco.ios
    version: "5.3.0"          # Pin เวอร์ชันตายตัว — ทุกเครื่องที่ install จะได้เหมือนกันเสมอ

  - name: ansible.netcommon
    version: ">=5.0.0,<6.0.0"  # หรือระบุเป็น range ก็ได้ (semantic versioning)

  - name: ansible.utils
    version: "3.1.0"           # ใช้ filter เสริม เช่น ipaddr, ipsubnet (ใช้ใน Step 476)

  # ตัวอย่างสำหรับ Platform อื่นที่ทีม NetOps อาจต้องดูแลร่วมกัน (Data Center / ACI)
  - name: cisco.nxos             # ใช้กับ Cisco Nexus (NX-OS) — Data Center Switch
    version: "6.1.0"
  - name: cisco.aci              # ใช้กับ Cisco ACI (Application Centric Infrastructure)
    version: "2.9.0"
```

### ติดตั้งจาก `requirements.yml`

```bash
ansible-galaxy collection install -r requirements.yml
```

```
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Installing 'cisco.ios:5.3.0' to '/home/netadmin/.ansible/collections/ansible_collections/cisco/ios'
Installing 'ansible.netcommon:5.2.1' to '.../ansible_collections/ansible/netcommon'
Installing 'ansible.utils:3.1.0' to '.../ansible_collections/ansible/utils'
Installing 'cisco.nxos:6.1.0' to '.../ansible_collections/cisco/nxos'
Installing 'cisco.aci:2.9.0' to '.../ansible_collections/cisco/aci'
```

### ตรวจสอบว่าติดตั้งตรงตาม Pin จริง

```bash
ansible-galaxy collection list
```

```
# /home/netadmin/.ansible/collections/ansible_collections
Collection         Version
------------------ -------
ansible.netcommon   5.2.1
ansible.utils        3.1.0
cisco.aci             2.9.0
cisco.ios             5.3.0
cisco.nxos            6.1.0
```

> **ทำไม `cisco.nxos`/`cisco.aci` ถึงสำคัญต้องรู้จัก แม้ Lab หลักสูตรนี้ใช้ IOS-XE ล้วน**: ทีม
> NetOps ระดับ Enterprise ส่วนใหญ่ดูแลอุปกรณ์หลายยี่ห้อ/หลาย Platform พร้อมกัน — Campus (IOS-XE)
> ใช้ `cisco.ios`, Data Center (Nexus) ใช้ `cisco.nxos`, ACI Fabric ใช้ `cisco.aci` — โครงสร้าง
> Inventory/Role/Playbook ที่ออกแบบดีจะแยก platform-specific module ไว้ใน Role ของตัวเอง แต่ใช้
> `group_vars` เดียวกันสำหรับ credential/policy กลาง ทำให้ขยายไปดูแล Data Center ในอนาคตได้ง่าย
> โดยไม่ต้องออกแบบโครงสร้างใหม่ทั้งหมด

### เพิ่ม `requirements.yml` เข้า Git และตั้งเป็นขั้นตอนบังคับก่อนรัน Playbook

```
day2-ops/
├── requirements.yml      # <-- commit เข้า Git เสมอ (ไม่มี credential ในไฟล์นี้)
├── inventory.yaml
└── ...
```

```bash
# ขั้นตอนมาตรฐานสำหรับสมาชิกทีมใหม่ หรือ CI/CD Pipeline (Part 47) ก่อนรัน Playbook ใดๆ
git clone https://git.company.local/netops/day2-ops.git
cd day2-ops
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

การ Pin เวอร์ชันด้วย `requirements.yml` คือสิ่งที่ทำให้ Ansible Project กลายเป็น **Reproducible**
— รันบนเครื่องไหน วันไหน ก็ได้ผลลัพธ์เดียวกันเสมอ ซึ่งเป็นข้อกำหนดพื้นฐานของ Network-as-Code
Pipeline ที่จะเรียนเต็มรูปแบบใน Part 47/79

---

## Step 473 — Handlers และ `notify`: Save Config เฉพาะเมื่อเปลี่ยนแปลงจริง

### ปัญหาของการ Save Config ทุกครั้งที่รัน Playbook

Playbook อย่างง่ายมักเขียน task `copy running-config startup-config` ไว้ท้ายสุดเสมอ โดยไม่ตรวจสอบ
ว่ามีการเปลี่ยนแปลง config จริงหรือไม่ — ปัญหาคือ:

| ปัญหา | รายละเอียด |
|---|---|
| **สิ้นเปลือง I/O บน NVRAM/Flash** | `write memory` ทุกครั้งที่รัน แม้ config เหมือนเดิม 100% ทำให้เกิดการเขียน Flash โดยไม่จำเป็น (Flash มีจำนวนรอบเขียนจำกัด) |
| **PLAY RECAP บอกข้อมูลผิด** | ถ้า save ทุกครั้งจะเห็น `changed` เสมอ ทำให้แยกไม่ออกว่า "รอบนี้มีการเปลี่ยน config จริง" หรือ "แค่ save ซ้ำเฉยๆ" |
| **ขัดกับหลัก Idempotency** | Automation ที่ดีควรรายงานผลตรงกับความเป็นจริงของระบบ (Step 227) |

**Handler** คือ Task พิเศษที่ **จะไม่ถูกรันเลย เว้นแต่มี Task อื่นสั่ง `notify` มาที่มัน และ Task
นั้น report ผลเป็น `changed`** — Ansible ฉลาดพอที่จะรวม Handler ที่ถูก notify ซ้ำหลายครั้งใน Play
เดียวให้ **รันแค่ครั้งเดียวท้าย Play** (ไม่ใช่รันซ้ำทุกครั้งที่มี task notify มา)

### `roles/ntp-config/handlers/main.yaml` — เพิ่ม Handler เข้า Role จาก Step 471

```yaml
---
# roles/ntp-config/handlers/main.yaml
- name: save running-config
  cisco.ios.ios_config:
    save_when: always     # handler ถูกเรียกแล้ว แปลว่ามีการเปลี่ยนแปลงแน่นอน ให้ save ตรงๆได้เลย
  listen: "save running-config"
```

> `listen:` ทำให้ Handler นี้รับ notify จาก **หลาย Task/หลาย Role พร้อมกัน** โดยใช้ชื่อ event
> เดียวกัน ("save running-config") ไม่จำเป็นต้องใช้ชื่อ Handler ตรงกับชื่อที่ notify เป๊ะๆ — มี
> ประโยชน์มากเมื่อ Role หลายตัว (NTP, Syslog, AAA, VLAN) ต้องการ save config ตัวเดียวกันท้าย Play

### แก้ `roles/ntp-config/tasks/main.yaml` ให้ `notify` Handler แทนการ save ตรงๆ

```yaml
---
# roles/ntp-config/tasks/main.yaml (ฉบับแก้ไข — ใช้ handler แทน task save ตรงๆ)
- name: "[ntp-config] Push NTP + Syslog config via rendered template"
  cisco.ios.ios_config:
    src: "ntp_syslog.j2"
  register: ntp_syslog_result
  notify: "save running-config"      # <-- แจ้ง handler แค่นี้ ไม่ต้องเขียน when: เอง
  tags: ["ntp", "syslog"]

- name: "[ntp-config] Debug change status"
  ansible.builtin.debug:
    msg: >
      {{ inventory_hostname }}: NTP/Syslog config
      {{ 'เปลี่ยนแปลง (จะ save ท้าย Play)' if ntp_syslog_result.changed else 'ไม่มีการเปลี่ยนแปลง' }}
  tags: ["ntp", "syslog"]
```

สังเกตว่าเราไม่ต้องเขียน `when: ntp_syslog_result.changed` เองอีกต่อไปเหมือน Step 227 —
Ansible **จะเรียก Handler ให้เองโดยอัตโนมัติ ก็ต่อเมื่อ task ด้านบน `changed` เท่านั้น** ถ้า
`ok` (ไม่เปลี่ยน) Handler จะไม่ถูกเรียกเลย

### พฤติกรรมสำคัญ: Handler รันท้าย Play เสมอ (ไม่ใช่ทันทีตำแหน่งที่ notify)

```
Task 1: Push NTP config          -> changed  -> notify "save running-config"
Task 2: Push Syslog config       -> changed  -> notify "save running-config"  (คนละ task แต่ event เดียวกัน)
Task 3: Push AAA config          -> ok       -> ไม่ notify
...
[ท้าย Play ทั้งหมด]
Handler: save running-config     -> รันแค่ "หนึ่งครั้ง" แม้ถูก notify มา 2 ครั้ง (Task 1 และ 2)
```

> ถ้าต้องการให้ Handler รันทันทีก่อน task ถัดไป (ไม่รอท้าย Play) ให้เพิ่ม
> `meta: flush_handlers` ที่ตำแหน่งที่ต้องการ — ใช้บ่อยเมื่อ task ถัดไปต้องพึ่งพา config ที่ save
> ไปแล้วจริง

### ทดสอบผลลัพธ์: รันครั้งแรก (มีการเปลี่ยนแปลง)

```bash
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
...

RUNNING HANDLER [ntp-config : save running-config] *********************************
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=3  changed=2  unreachable=0  failed=0  skipped=0
```

### รันครั้งที่ 2 (ไม่มีอะไรเปลี่ยน — Handler ไม่ถูกเรียกเลย)

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
ok: [CORE-SW1]
ok: [CORE-SW2]
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=2  changed=0  unreachable=0  failed=0  skipped=0
```

สังเกตว่า **ไม่มีบรรทัด `RUNNING HANDLER` เลย** ในรอบที่ 2 — เพราะไม่มี task ใด `notify` มา
(ทุก task เป็น `ok`) — นี่คือพฤติกรรมที่ถูกต้องตามหลัก Idempotency: **ไม่มีการเขียน Flash โดย
ไม่จำเป็นแม้แต่ครั้งเดียว**

---

## Step 474 — Ansible Tags: จัดโครงสร้าง Playbook ใหญ่ให้เลือกรันเฉพาะส่วน

### ทำไมต้องมี Tags

เมื่อ Playbook รวมหลายเรื่องไว้ในโปรเจกต์เดียว (NTP, Syslog, ACL, AAA) วิศวกรจะไม่ต้องการรันทุก
อย่างพร้อมกันเสมอไป — เช่น ถ้าแค่ต้องการแก้ ACL Management ด่วน ไม่ควรต้องรอ Task NTP/Syslog/AAA
รันไปด้วยทั้งหมด (เสียเวลา, เพิ่มความเสี่ยง touch ส่วนที่ไม่เกี่ยวข้อง) **Tags** ให้เราติดป้ายชื่อ
ที่ Task/Play/Role แล้วเลือกรันเฉพาะ Tag ที่ต้องการด้วย `--tags`

### Playbook รวม `day2-ops.yaml` — ครอบคลุม NTP + Syslog + ACL + AAA พร้อม Tag

```yaml
---
# playbooks/day2-ops.yaml
- name: "Day2-Ops Combined Playbook: NTP + Syslog + ACL + AAA"
  hosts: network_devices
  gather_facts: no
  connection: ansible.netcommon.network_cli

  vars:
    ntp_servers:
      - { ip: "10.10.99.50", prefer: true }
      - { ip: "10.10.99.51", prefer: false }
    syslog_server: "10.10.99.50"
    mgmt_admin_subnet: "10.10.99.0 0.0.0.255"

  tasks:
    # ================= NTP Section =================
    - name: "[NTP] Configure NTP servers"
      cisco.ios.ios_config:
        lines:
          - "ntp server {{ item.ip }}{{ ' prefer' if item.prefer else '' }}"
      loop: "{{ ntp_servers }}"
      notify: "save running-config"
      tags: ["ntp"]

    # ================= Syslog Section =================
    - name: "[SYSLOG] Configure Syslog host and trap level"
      cisco.ios.ios_config:
        lines:
          - "logging host {{ syslog_server }}"
          - "logging trap informational"
          - "logging source-interface Loopback0"
      notify: "save running-config"
      tags: ["syslog"]

    # ================= ACL Section (Management Plane Protection, Part 14) =================
    - name: "[ACL] Ensure management-plane ACL exists"
      cisco.ios.ios_config:
        lines:
          - "10 permit {{ mgmt_admin_subnet }}"
          - "20 deny any log"
        parents: "ip access-list extended MGMT-ACCESS-IN"
      notify: "save running-config"
      tags: ["security", "acl"]

    - name: "[ACL] Apply ACL to management VLAN interface"
      cisco.ios.ios_config:
        lines:
          - "ip access-group MGMT-ACCESS-IN in"
        parents: "interface Vlan99"
      notify: "save running-config"
      tags: ["security", "acl"]

    # ================= AAA Section (Part 22) =================
    - name: "[AAA] Ensure AAA new-model and TACACS+ group configured"
      cisco.ios.ios_config:
        lines:
          - "aaa new-model"
          - "tacacs server AAA-SRV1"
          - " address ipv4 10.10.99.60"
          - " key {{ vault_tacacs_key }}"
          - " exit"
          - "aaa group server tacacs+ AAA-TACACS"
          - " server name AAA-SRV1"
          - " exit"
          - "aaa authentication login CONSOLE-IN group AAA-TACACS local"
          - "aaa authentication login VTY-IN group AAA-TACACS local"
      notify: "save running-config"
      tags: ["security", "aaa"]

  handlers:
    - name: save running-config
      cisco.ios.ios_config:
        save_when: always
      listen: "save running-config"
```

### รันเฉพาะ Tag ที่ต้องการ

```bash
# รันเฉพาะส่วน NTP อย่างเดียว
ansible-playbook -i inventory.yaml playbooks/day2-ops.yaml --tags ntp --ask-vault-pass

# รันทุกอย่างที่เกี่ยวกับ security (ครอบคลุมทั้ง acl และ aaa เพราะติด tag "security" ไว้ทั้งคู่)
ansible-playbook -i inventory.yaml playbooks/day2-ops.yaml --tags security --ask-vault-pass

# รันเฉพาะ ACL อย่างเดียว (ไม่เอา AAA แม้จะอยู่ใน "security" เดียวกัน)
ansible-playbook -i inventory.yaml playbooks/day2-ops.yaml --tags acl --ask-vault-pass

# ข้าม Tag ที่ไม่ต้องการ (ใช้ --skip-tags แทน --tags)
ansible-playbook -i inventory.yaml playbooks/day2-ops.yaml --skip-tags aaa --ask-vault-pass

# ดูรายการ Tag ทั้งหมดที่มีใน Playbook โดยไม่ต้องรันจริง
ansible-playbook -i inventory.yaml playbooks/day2-ops.yaml --list-tags
```

```
playbook: playbooks/day2-ops.yaml

  play #1 (network_devices): Day2-Ops Combined Playbook: NTP + Syslog + ACL + AAA
      TAGS: []
      TASK TAGS: [acl, aaa, ntp, security, syslog]
```

### Output ตัวอย่างเมื่อรันด้วย `--tags ntp`

```
PLAY [Day2-Ops Combined Playbook: NTP + Syslog + ACL + AAA] ***********************

TASK [[NTP] Configure NTP servers] *************************************************
changed: [CORE-SW1] => (item={'ip': '10.10.99.50', 'prefer': True})
changed: [CORE-SW1] => (item={'ip': '10.10.99.51', 'prefer': False})
...

RUNNING HANDLER [save running-config] **********************************************
changed: [CORE-SW1]
...

PLAY RECAP **************************************************************************
CORE-SW1  : ok=2  changed=2  unreachable=0  failed=0  skipped=0
```

สังเกตว่า **Task ของ Syslog/ACL/AAA ไม่ปรากฏใน Output เลย** เพราะไม่ตรงกับ `--tags ntp` ที่ระบุ —
Ansible ข้าม (skip) task เหล่านั้นไปโดยสิ้นเชิง ไม่ใช่แค่ไม่แสดงผล

> **แนวทางตั้งชื่อ Tag ที่ดี**: ใช้ทั้ง Tag เฉพาะเจาะจง (`ntp`, `syslog`, `acl`, `aaa`) และ Tag
> รวมกลุ่ม (`security` ครอบคลุม `acl`+`aaa`) พร้อมกัน — Task หนึ่งอันใส่ได้หลาย Tag เป็น list
> (`tags: ["security", "aaa"]`) ทำให้เลือกรันได้ทั้งแบบเจาะจงและแบบกลุ่มตามสถานการณ์จริง

---

## Step 475 — Conditionals (`when`) และ Loops (`loop`): สร้าง VLAN จาก List เดียว

### `when`: รัน Task เฉพาะเมื่อเงื่อนไขเป็นจริง

`when` คือ Conditional ที่ตรวจสอบก่อนว่าจะรัน Task นี้กับ host ปัจจุบันหรือไม่ — ใช้บ่อยเมื่อ
Task บางอย่างควรรันเฉพาะกับกลุ่มอุปกรณ์บางกลุ่มเท่านั้น (เช่น VLAN ควรสร้างบน Distribution Switch
ที่มี SVI เท่านั้น ไม่ควรสร้างบน Core Switch ที่เป็น Routed Port ล้วนตาม Design ใน
`docs/00-ip-address-plan.md`)

```yaml
- name: "[VLAN] Create VLAN only on distribution switches"
  cisco.ios.ios_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
    state: merged
  loop: "{{ lab_vlans }}"
  when: "'dist_switches' in group_names"     # รันเฉพาะ host ที่อยู่ใน group dist_switches เท่านั้น
```

`group_names` คือ Variable พิเศษของ Ansible ที่คืนค่าเป็น list ของทุก group ที่ host ปัจจุบัน
เป็นสมาชิกอยู่ (ตาม Inventory) — เช็คด้วย `in` เพื่อกรองว่า host นี้อยู่ใน group ที่ต้องการหรือไม่

### `loop`: วนซ้ำเพื่อสร้าง VLAN ทั้ง 5 ตัวจาก List เดียว (ไม่ Hardcode)

**`group_vars/dist_switches.yml`** — นิยาม VLAN ทั้งหมดของ Lab เป็น List เดียว (ตาม IP Plan)

```yaml
---
# group_vars/dist_switches.yml
lab_vlans:
  - { id: 10, name: "SALES",   subnet: "10.10.10.0/24" }
  - { id: 20, name: "VOICE",   subnet: "10.10.20.0/24" }
  - { id: 30, name: "SERVERS", subnet: "10.10.30.0/24" }
  - { id: 40, name: "WIFI",    subnet: "10.10.40.0/24" }
  - { id: 99, name: "MGMT",    subnet: "10.10.99.0/24" }
```

**`playbooks/create_lab_vlans.yaml`** — สร้าง VLAN ทั้งหมดด้วย Loop เดียว

```yaml
---
- name: "Create all 5 lab VLANs from a single list (no hardcoding)"
  hosts: dist_switches
  gather_facts: no
  connection: ansible.netcommon.network_cli

  tasks:
    - name: "Ensure VLAN exists with correct name"
      cisco.ios.ios_vlans:
        config:
          - vlan_id: "{{ item.id }}"
            name: "{{ item.name }}"
        state: merged
      loop: "{{ lab_vlans }}"
      loop_control:
        label: "VLAN {{ item.id }} ({{ item.name }})"   # ทำให้ output อ่านง่ายขึ้น ไม่แสดง dict เต็ม
      register: vlan_result
      notify: "save running-config"

    - name: "Report which VLANs actually changed"
      ansible.builtin.debug:
        msg: >
          {{ inventory_hostname }}: VLAN {{ item.item.id }} ({{ item.item.name }})
          {{ 'สร้าง/แก้ไขแล้ว' if item.changed else 'มีอยู่แล้ว ไม่เปลี่ยน' }}
      loop: "{{ vlan_result.results }}"
      loop_control:
        label: "VLAN {{ item.item.id }}"

  handlers:
    - name: save running-config
      cisco.ios.ios_config:
        save_when: always
      listen: "save running-config"
```

### `with_items` เทียบกับ `loop` (รูปแบบเก่าเทียบใหม่)

```yaml
# รูปแบบเก่า (with_items) — ยังใช้ได้ แต่ไม่แนะนำสำหรับ Playbook ใหม่
- name: "Old-style loop syntax (legacy)"
  cisco.ios.ios_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
  with_items: "{{ lab_vlans }}"

# รูปแบบใหม่ (loop) — แนะนำเสมอตั้งแต่ Ansible 2.5+
- name: "Modern loop syntax (recommended)"
  cisco.ios.ios_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
  loop: "{{ lab_vlans }}"
```

> **หมายเหตุ**: `with_items` เป็นส่วนหนึ่งของตระกูล `with_<lookup>` (เช่น `with_dict`,
> `with_nested`) ที่เก่ากว่าและใช้ Lookup Plugin ภายใน ในขณะที่ `loop` เป็น Keyword ใหม่ที่อ่านง่าย
> กว่าและครอบคลุม use case ส่วนใหญ่ — เอกสาร Ansible ทางการแนะนำให้ใช้ `loop` สำหรับ Playbook ใหม่
> ทั้งหมด และเก็บ `with_items` ไว้แค่เพื่อความเข้าใจ Legacy Playbook เท่านั้น

### ผลลัพธ์จริงเมื่อรัน

```bash
ansible-playbook -i inventory.yaml playbooks/create_lab_vlans.yaml --ask-vault-pass
```

```
TASK [Ensure VLAN exists with correct name] *****************************************
changed: [DIST-SW1] => (item=VLAN 10 (SALES))
changed: [DIST-SW1] => (item=VLAN 20 (VOICE))
changed: [DIST-SW1] => (item=VLAN 30 (SERVERS))
changed: [DIST-SW1] => (item=VLAN 40 (WIFI))
changed: [DIST-SW1] => (item=VLAN 99 (MGMT))

TASK [Report which VLANs actually changed] ******************************************
ok: [DIST-SW1] => (item=VLAN 10) => {
    "msg": "DIST-SW1: VLAN 10 (SALES) สร้าง/แก้ไขแล้ว"
}
...

RUNNING HANDLER [save running-config] ***********************************************
changed: [DIST-SW1]

PLAY RECAP ***************************************************************************
DIST-SW1  : ok=6  changed=2  unreachable=0  failed=0  skipped=0
```

การเพิ่ม VLAN ตัวที่ 6 ในอนาคตทำได้เพียง **เพิ่ม 1 บรรทัดใน `lab_vlans` list** โดยไม่ต้องแก้
Playbook เลย — นี่คือประโยชน์หลักของการรวม `loop` เข้ากับ Data-driven Variable ที่แยกออกจาก Logic

---

## Step 476 — Custom Jinja2 Filter และ Templating ขั้นสูง

### ทำไมต้องเขียน Custom Filter เอง

Jinja2 มี Built-in Filter อยู่แล้วมาก (`upper`, `join`, `default` ฯลฯ) และ Ansible เพิ่ม Filter
เฉพาะ Network เข้ามาอีกผ่าน `ansible.utils` (เช่น `ipaddr`, `ipsubnet`) แต่บางครั้งงานของทีม NetOps
มี Logic เฉพาะทางที่ไม่มี Filter สำเร็จรูปให้ใช้ — เช่น การแปลง Prefix Length (`/24`) เป็น
Subnet Mask แบบจุด (`255.255.255.0`) ในรูปแบบเฉพาะที่ IOS Classic ต้องใช้คู่กับคำสั่ง `ip address`
เราจึงเขียน **Custom Filter Plugin** ของเราเองได้

### `filter_plugins/netmask_filters.py` — Custom Filter Plugin

```python
#!/usr/bin/env python3
"""
filter_plugins/netmask_filters.py
Custom Jinja2 Filter สำหรับงาน Network Automation ที่ Ansible ไม่มี Filter สำเร็จรูปให้
"""
import ipaddress


def cidr_to_netmask(prefix_length):
    """แปลง Prefix Length (เช่น 24) เป็น Dotted Decimal Netmask (เช่น 255.255.255.0)"""
    return str(ipaddress.IPv4Network(f"0.0.0.0/{prefix_length}").netmask)


def to_wildcard(prefix_length):
    """แปลง Prefix Length เป็น Wildcard Mask สำหรับใช้กับ ACL (เช่น 24 -> 0.0.0.255)"""
    netmask = ipaddress.IPv4Network(f"0.0.0.0/{prefix_length}").netmask
    wildcard = ipaddress.IPv4Address(int(ipaddress.IPv4Address(netmask)) ^ 0xFFFFFFFF)
    return str(wildcard)


def network_of(cidr):
    """คืนค่าเฉพาะ Network Address จาก CIDR string (เช่น '10.10.10.5/24' -> '10.10.10.0')"""
    return str(ipaddress.IPv4Network(cidr, strict=False).network_address)


class FilterModule(object):
    """ลงทะเบียน Custom Filter ทั้งหมดให้ Ansible/Jinja2 มองเห็น"""

    def filters(self):
        return {
            "cidr_to_netmask": cidr_to_netmask,
            "to_wildcard": to_wildcard,
            "network_of": network_of,
        }
```

> **กฎการค้นหาของ Ansible**: ไฟล์ Custom Filter ต้องอยู่ในโฟลเดอร์ชื่อ `filter_plugins/` ที่ระดับ
> เดียวกับ Playbook (หรือใน Role เป็น `roles/<role>/filter_plugins/`) — Ansible จะโหลด Class
> `FilterModule` และเมธอด `filters()` โดยอัตโนมัติ ไม่ต้อง `import`/register เพิ่มในที่อื่นเลย

### `host_vars/DIST-SW1.yaml` — ตัวแปรเฉพาะอุปกรณ์แบบมีโครงสร้าง (Structured)

```yaml
---
# host_vars/DIST-SW1.yaml
hostname: DIST-SW1
loopback0:
  ip: "1.1.1.11"
  prefix: 32

mgmt_interface:
  name: "Vlan99"
  ip: "10.10.99.12"
  prefix: 24

uplink_interface:
  name: "TenGigabitEthernet1/1/1"
  description: "** Uplink to CORE-SW1 **"

svi_interfaces:
  - { vlan: 10, name: "SALES",   ip: "10.10.10.2",  prefix: 24 }
  - { vlan: 20, name: "VOICE",   ip: "10.10.20.2",  prefix: 24 }
  - { vlan: 99, name: "MGMT",    ip: "10.10.99.12", prefix: 24 }

management_acl:
  name: "MGMT-ACCESS-IN"
  permitted_subnets:
    - "10.10.99.0/24"
```

### `templates/full-device-config.j2` — Render Config เต็มรูปแบบของ DIST Switch

```jinja2
{# templates/full-device-config.j2 #}
! ===============================================================
! Auto-generated FULL configuration for {{ hostname }}
! Generated by Ansible + Jinja2 (Custom Filter) - DO NOT EDIT MANUALLY
! ===============================================================
hostname {{ hostname }}
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address {{ loopback0.ip }} {{ loopback0.prefix | cidr_to_netmask }}
!
interface {{ mgmt_interface.name }}
 description ** Management VLAN **
 ip address {{ mgmt_interface.ip }} {{ mgmt_interface.prefix | cidr_to_netmask }}
 no shutdown
!
interface {{ uplink_interface.name }}
 description {{ uplink_interface.description }}
 no switchport
 speed 10000
 duplex full
 no shutdown
!
{% for vlan in svi_interfaces %}
vlan {{ vlan.vlan }}
 name {{ vlan.name }}
!
interface Vlan{{ vlan.vlan }}
 description ** SVI for VLAN {{ vlan.vlan }} ({{ vlan.name }}) **
 ip address {{ vlan.ip }} {{ vlan.prefix | cidr_to_netmask }}
 no shutdown
!
{% endfor %}
ip access-list extended {{ management_acl.name }}
{% for subnet in management_acl.permitted_subnets %}
 permit ip {{ subnet | network_of }} {{ subnet.split('/')[1] | to_wildcard }} any
{% endfor %}
 deny ip any any log
!
interface {{ mgmt_interface.name }}
 ip access-group {{ management_acl.name }} in
!
end
```

### Render ทดสอบด้วย Ansible `template` Module (Dry-render ไม่ push จริง)

```yaml
---
# playbooks/render_full_config.yaml
- name: "Render full device config locally for review (no push)"
  hosts: dist_switches
  gather_facts: no
  connection: local

  tasks:
    - name: "Render full-device-config.j2 to local file"
      ansible.builtin.template:
        src: "templates/full-device-config.j2"
        dest: "./generated_configs/{{ inventory_hostname }}-full.cfg"
      delegate_to: localhost
```

### ผลลัพธ์ที่ Render ออกมา (`generated_configs/DIST-SW1-full.cfg`, ตัดมาบางส่วน)

```
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.11 255.255.255.255
!
interface Vlan99
 description ** Management VLAN **
 ip address 10.10.99.12 255.255.255.0
 no shutdown
!
vlan 10
 name SALES
!
interface Vlan10
 description ** SVI for VLAN 10 (SALES) **
 ip address 10.10.10.2 255.255.255.0
 no shutdown
!
ip access-list extended MGMT-ACCESS-IN
 permit ip 10.10.99.0 0.0.0.255 any
 deny ip any any log
```

สังเกตว่า `loopback0.prefix | cidr_to_netmask` แปลง `32` เป็น `255.255.255.255` และ
`subnet.split('/')[1] | to_wildcard` แปลง `24` เป็น `0.0.0.255` โดยอัตโนมัติ — เราเก็บข้อมูลใน
`host_vars` เป็น Prefix Length (มาตรฐาน CIDR ที่คนอ่านง่าย) แล้วให้ Custom Filter แปลงเป็นรูปแบบ
ที่ IOS ต้องการตอน Render เท่านั้น ไม่ต้องเก็บข้อมูลซ้ำสองรูปแบบใน `host_vars`

---

## Step 477 — Ansible Vault เจาะลึก: Vault-ID และ Multi-Environment (dev/prod)

### ข้อจำกัดของ Ansible Vault แบบเดียว (Single Password) ที่แนะนำใน Part 23

Part 23 Step 225 ใช้ Vault Password **เพียงรหัสเดียว** สำหรับทั้งโปรเจกต์ — ปัญหาคือถ้าทีมดูแล
ทั้ง Environment **Dev/Lab** และ **Production** พร้อมกัน การใช้รหัส Vault เดียวกันหมายความว่า
**ใครก็ตามที่รู้รหัส Vault ของ Dev จะเปิดไฟล์ Secret ของ Production ได้ด้วย** — เสี่ยงมาก
ในทางปฏิบัติ ควรแยก Vault Password ตาม Environment โดยใช้ฟีเจอร์ **Vault-ID**

### สร้างไฟล์ Password แยกตาม Environment

```bash
mkdir -p ~/.vault_pass
echo "Dev-VaultP@ss-2026!"  > ~/.vault_pass/dev.txt
echo "Prod-VaultP@ss-2026!!" > ~/.vault_pass/prod.txt
chmod 600 ~/.vault_pass/dev.txt ~/.vault_pass/prod.txt

# เพิ่มเข้า .gitignore เสมอ — ห้าม commit ไฟล์รหัส Vault เด็ดขาด
echo ".vault_pass/" >> .gitignore
```

### สร้างไฟล์ Vault ที่ผูกกับ Vault-ID ของแต่ละ Environment

```bash
# สร้าง Vault สำหรับ Environment "dev" — ระบุ vault-id ชื่อ dev
ansible-vault create --vault-id dev@~/.vault_pass/dev.txt group_vars/dev/vault.yml

# สร้าง Vault สำหรับ Environment "prod" — ระบุ vault-id ชื่อ prod
ansible-vault create --vault-id prod@~/.vault_pass/prod.txt group_vars/prod/vault.yml
```

เนื้อหาที่พิมพ์ลงไปในแต่ละไฟล์ (Editor จะเปิดให้พิมพ์ plaintext ก่อน Ansible เข้ารหัสอัตโนมัติ):

```yaml
# group_vars/dev/vault.yml (หลังปลดล็อกดู)
vault_ansible_password: "Dev-Lab-P@ssw0rd!"
vault_ansible_become_password: "Dev-Cisco123!"
vault_tacacs_key: "Dev-TACACS-Key-2026!"
```

```yaml
# group_vars/prod/vault.yml (หลังปลดล็อกดู)
vault_ansible_password: "Pr0d-Str0ng-P@ssw0rd-2026!"
vault_ansible_become_password: "Pr0d-En@ble-Secret-2026!"
vault_tacacs_key: "Prod-TACACS-Key-VeryStrong-2026!"
```

### ไฟล์ที่เข้ารหัสแล้วจะมี Vault-ID ระบุอยู่ใน Header (ต่างจากรูปแบบเดิม)

```
$ANSIBLE_VAULT;1.2;AES256;dev
66386439653236336462626566653063336164663966303231363934653561353966366139
...
```

สังเกตเลขเวอร์ชัน `1.2` (ไม่ใช่ `1.1` แบบ Part 23) และมี `;dev` ต่อท้าย — นี่คือสิ่งที่ทำให้
Ansible รู้ว่าไฟล์นี้ต้องปลดล็อกด้วย Vault-ID ชื่อ `dev` เท่านั้น

### รัน Playbook พร้อมระบุ Vault-ID หลายตัวพร้อมกัน (กรณีต้องใช้ทั้ง dev และ prod ในรอบเดียว)

```bash
# รันกับ Environment dev เท่านั้น
ansible-playbook -i inventory/dev.yaml site.yaml \
  --vault-id dev@~/.vault_pass/dev.txt

# รันกับ Environment prod — ใช้ Vault-ID คนละตัว และให้ระบบถามรหัสสด (ไม่เก็บไฟล์ prod password ไว้บนเครื่อง)
ansible-playbook -i inventory/prod.yaml site.yaml \
  --vault-id prod@prompt

# กรณี Playbook เดียวต้องอ่านค่าจากทั้งสอง Vault พร้อมกัน (เช่น เทียบ config cross-environment)
ansible-playbook -i inventory/all.yaml compare_playbook.yaml \
  --vault-id dev@~/.vault_pass/dev.txt \
  --vault-id prod@prompt
```

`prod@prompt` สั่งให้ Ansible **ถามรหัสผ่านสดหน้าจอ** แทนการอ่านจากไฟล์ — แนวทางที่แนะนำสำหรับ
Production Vault เพื่อไม่ให้มีไฟล์รหัสผ่าน Production เก็บอยู่บน Laptop ของวิศวกรตลอดเวลา

### `ansible-vault edit` — แก้ไข Secret โดยไม่ต้อง Decrypt/Encrypt มือ

```bash
ansible-vault edit --vault-id prod@prompt group_vars/prod/vault.yml
```

คำสั่งนี้จะถามรหัส Vault, decrypt ไฟล์ชั่วคราวใน memory, เปิด Editor (`$EDITOR`) ให้แก้ไข, แล้ว
encrypt กลับให้อัตโนมัติตอนปิด Editor — **ไม่มีช่วงเวลาที่ไฟล์ plaintext ถูกเขียนลง Disk เลย**

### `ansible-vault rekey` — เปลี่ยนรหัสผ่าน Vault (เช่น หลังพนักงานลาออก)

```bash
# เปลี่ยนรหัส Vault ของ Environment prod จากรหัสเก่าเป็นรหัสใหม่
ansible-vault rekey --vault-id prod@~/.vault_pass/prod_old.txt \
  --new-vault-id prod@~/.vault_pass/prod_new.txt \
  group_vars/prod/vault.yml
```

```
New Vault password:
Confirm New Vault password:
Rekey successful
```

> **หลักการ Rotation**: ทุกครั้งที่มีวิศวกรที่รู้รหัส Vault ลาออกจากทีม หรือทุกรอบ Audit ตาม
> นโยบายบริษัท (เช่น ทุก 90 วัน) ต้อง `rekey` และแจกรหัสใหม่ให้เฉพาะสมาชิกทีมที่ยังทำงานอยู่ —
> การมีระบบ Vault-ID แยกตาม Environment ทำให้ Rekey Production ได้โดย**ไม่กระทบ** Vault ของ Dev
> เลย ต่างจาก Single Password Vault ที่ต้อง Rekey ทุกไฟล์ในโปรเจกต์พร้อมกันเสมอ

### เชื่อม Vault-Encrypted Variable เข้ากับ Template จาก Step 476

```yaml
# group_vars/prod/main.yml (ไฟล์ธรรมดา ไม่เข้ารหัส — reference ตัวแปรจาก vault.yml)
ansible_network_os: cisco.ios.ios
ansible_connection: ansible.netcommon.network_cli
ansible_user: admin
ansible_password: "{{ vault_ansible_password }}"
ansible_become_password: "{{ vault_ansible_become_password }}"
ansible_become: yes
ansible_become_method: enable
tacacs_key: "{{ vault_tacacs_key }}"     # ใช้ในบรรทัด AAA ของ day2-ops.yaml (Step 474)
```

ตัวแปร `tacacs_key` (ที่ผูกกับ `vault_tacacs_key` ที่เข้ารหัสไว้) สามารถถูกใช้ตรงใน Template
หรือ `lines:` ของ `ios_config` ได้ทันทีเหมือนตัวแปรปกติทุกประการ — Ansible จะ decrypt ให้อัตโนมัติ
ระหว่างการรัน (in-memory เท่านั้น) โดยที่ Template/Task ไม่ต้องรู้เลยว่าตัวแปรนี้มาจาก Vault

---

## Step 478 — Idempotency Verification ด้วย `--check` และ `--diff`

### ทำไมต้อง Dry-run ก่อนรันจริงเสมอ

แม้ Ansible module อย่าง `ios_config` จะมี Idempotency ในตัว (Step 227) แต่ **ก่อน Apply Config
กับอุปกรณ์ Production จริง** วิศวกรที่ดีควรตรวจสอบก่อนเสมอว่า "ถ้ารัน Playbook นี้จริง จะมีอะไร
เปลี่ยนแปลงบ้าง" — คำตอบนี้หาได้จาก `--check` (Check Mode / Dry-run) ร่วมกับ `--diff` (แสดงผล
ต่างแบบ diff) **โดยไม่ต้อง push อะไรเข้าอุปกรณ์จริงแม้แต่บรรทัดเดียว**

| Flag | ความหมาย |
|---|---|
| `--check` | จำลองการรัน (Dry-run) — บอกว่า task ไหนจะ `changed` ถ้ารันจริง แต่**ไม่ push อะไรเข้าอุปกรณ์จริง** |
| `--diff` | แสดง diff (บรรทัดที่จะถูกเพิ่ม/ลบ) ของ config ที่กำลังจะเปลี่ยน — ใช้คู่กับ `--check` เสมอเพื่อเห็นรายละเอียด |
| `--check --diff` | ใช้คู่กันบ่อยที่สุด — เห็นทั้ง "จะเปลี่ยนไหม" และ "เปลี่ยนอะไรบ้างเป๊ะๆ" |

> **กฎเหล็กของ Part นี้**: **Playbook ทุกตัวในหลักสูตรนี้ ควรรันด้วย `--check --diff` ก่อนเสมอ**
> แล้วอ่านผลให้ละเอียดก่อนรันจริงโดยไม่มี flag ทั้งสองตัว — เป็นขั้นตอนบังคับสำหรับ Production
> Change เสมอ (มักผูกกับ Change Management Ticket ในองค์กรจริง)

### ตัวอย่าง: เพิ่ม NTP Server ตัวใหม่เข้า `ntp_servers` List

```yaml
# group_vars/network_devices.yml (แก้ไข — เพิ่ม NTP server ตัวที่ 3)
ntp_servers:
  - { ip: "10.10.99.50", prefer: true }
  - { ip: "10.10.99.51", prefer: false }
  - { ip: "10.10.99.52", prefer: false }   # <-- เพิ่มใหม่
```

### รันด้วย `--check --diff` ก่อน (ยังไม่ Apply จริง)

```bash
ansible-playbook -i inventory.yaml site.yaml --check --diff --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
--- before
+++ after
@@ -1,5 +1,6 @@
 ntp server 10.10.99.50 prefer
 ntp server 10.10.99.51
+ntp server 10.10.99.52
 ntp source Loopback0
 !
 logging host 10.10.99.50

changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=1  changed=1  unreachable=0  failed=0  skipped=0
```

Diff Output แสดงให้เห็นชัดเจนว่ามีเพียง **1 บรรทัดที่จะถูกเพิ่ม** (`+ntp server 10.10.99.52`)
และไม่มีบรรทัดอื่นถูกกระทบ — วิศวกรตรวจสอบได้ทันทีว่าการเปลี่ยนแปลงตรงกับที่ตั้งใจไว้จริง ก่อนที่
จะกดรันจริงกับอุปกรณ์ Production 6 ตัวพร้อมกัน

> สังเกตว่า **ไม่มีบรรทัด `RUNNING HANDLER`** ปรากฏใน Check Mode แม้ Task จะ report `changed`
> — เพราะ Ansible รู้ว่านี่เป็นแค่การจำลอง จึงไม่ trigger Handler ที่จะไป save config จริงด้วย
> (ป้องกัน side-effect ระหว่าง Dry-run โดยสมบูรณ์)

### รันจริงหลังตรวจสอบ Diff แล้วมั่นใจ

```bash
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
changed: [CORE-SW1]
...

RUNNING HANDLER [ntp-config : save running-config] **********************************
changed: [CORE-SW1]
...
```

### รันซ้ำอีกครั้งด้วย `--check --diff` เพื่อพิสูจน์ Idempotency (ควรไม่มี Diff อีกแล้ว)

```bash
ansible-playbook -i inventory.yaml site.yaml --check --diff --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
ok: [CORE-SW1]
ok: [CORE-SW2]
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=1  changed=0  unreachable=0  failed=0  skipped=0
```

ไม่มี Diff Output เลยเพราะ config ตรงกับที่ต้องการอยู่แล้ว 100% — นี่คือการยืนยัน (verification)
ว่า Playbook ของเรามี Idempotency ที่ถูกต้องจริง ไม่ใช่แค่ "เชื่อว่าถูก" จากการอ่าน YAML เฉยๆ

### ข้อจำกัดของ Check Mode ที่ต้องรู้

| ข้อจำกัด | รายละเอียด |
|---|---|
| ไม่ทุก Module รองรับ Check Mode | Module บางตัว (เช่น Custom Module ที่เขียนเอง หรือ Module เก่าบางตัว) อาจไม่มี logic จำลองผล ให้ตรวจสอบด้วย `ansible-doc <module>` ว่ามี `check_mode: support: full` หรือไม่ |
| Fact ที่ยังไม่เกิดจริง | Task ที่ `register` ผลจาก Task ก่อนหน้าที่ยังไม่ Apply จริง อาจได้ค่าจำลองที่ไม่ตรงกับความเป็นจริง 100% ถ้า Task นั้นดึงข้อมูลจากอุปกรณ์ |
| `delegate_to: localhost` บาง Task | Task ที่เขียนไฟล์บน Control Node (เช่น backup config) จะยังคง "ทำจริง" แม้อยู่ใน Check Mode เว้นแต่จะเช็ค `ansible_check_mode` เอง |

```yaml
# ตัวอย่างการเช็ค ansible_check_mode เองสำหรับ Task ที่ Module ไม่รองรับ Check Mode เต็มรูปแบบ
- name: "Write backup file only when NOT in check mode"
  ansible.builtin.copy:
    content: "{{ running_config_raw.stdout[0] }}"
    dest: "./backups/{{ inventory_hostname }}.cfg"
  delegate_to: localhost
  when: not ansible_check_mode
```

---

## Step 479 — AWX / Ansible Automation Platform (AAP): จาก Laptop สู่ Platform ระดับทีม

### ปัญหาของการรัน Ansible จาก Laptop วิศวกรคนเดียว

ทุก Playbook ที่เขียนมาตลอด Part นี้ (และ Part 23) รันจาก Terminal บน Laptop ของวิศวกรคนเดียว —
ใช้ได้ดีตอนเรียนและตอนทีมเล็ก แต่เมื่อทีม NetOps ขยายใหญ่ขึ้น จะเกิดปัญหา:

| ปัญหา | รายละเอียด |
|---|---|
| **Single Point of Failure** | ถ้า Laptop วิศวกรคนเดียวมีปัญหา (หาย/พัง) Automation ทั้งทีมหยุดชะงัก |
| **ไม่มี RBAC** | ทุกคนที่มี Playbook + Vault Password รันได้หมด ไม่แยกสิทธิ์ว่าใครรัน Playbook ไหนได้ |
| **ไม่มี Audit Trail รวมศูนย์** | ไม่รู้ว่าใครรัน Playbook ไหน เมื่อไหร่ ผลลัพธ์อะไร (มีแต่ log กระจายอยู่ใน Terminal แต่ละคน) |
| **ไม่มี Scheduling** | ต้องมีคนนั่งรัน Playbook ตามเวลาเอง ไม่สามารถตั้งให้รันอัตโนมัติตามรอบ (เช่น Backup Config ทุกคืน) |
| **Credential กระจัดกระจาย** | Vault Password/SSH Key อยู่บน Laptop หลายเครื่อง เสี่ยงหลุด/สูญหายมากกว่าเก็บรวมศูนย์ |
| **Visibility ต่ำ** | ผู้จัดการทีมไม่เห็นภาพรวมว่า Playbook ไหนรันไปแล้วบ้าง สำเร็จ/ล้มเหลวเท่าไหร่ |

### AWX และ Ansible Automation Platform คืออะไร

**AWX** คือโครงการ Open Source (สนับสนุนโดย Red Hat) ที่ครอบ Ansible ด้วย **Web UI + REST API +
Scheduler + RBAC** ทำให้ทีมสามารถรัน Playbook ผ่านหน้าเว็บได้โดยไม่ต้อง SSH เข้าเครื่องใครเลย —
**Ansible Automation Platform (AAP)** คือเวอร์ชัน Commercial/Enterprise ที่ Red Hat ขายพร้อม
Support และฟีเจอร์เพิ่มเติม (Automation Hub ส่วนตัว, Automation Mesh สำหรับ Multi-site) — AWX
คือ Upstream Project ที่ AAP พัฒนาต่อมา (คล้ายความสัมพันธ์ Fedora ↔ RHEL)

### แนวคิดหลักของ AWX/AAP ที่ทีม NetOps ต้องรู้จัก

จาก "รัน `ansible-playbook` ตรงจาก Terminal ด้วย Vault Password ที่อยู่บนเครื่องตัวเอง ไม่มี
Schedule ไม่มี Audit" กลายเป็น Platform เดียวที่มี Web UI/REST API ครอบไว้ทั้ง Project, Inventory,
Credential, Job Template, RBAC, Schedule, Workflow Template และ Job History/Audit Log ครบวงจร:

| แนวคิด | ความหมาย |
|---|---|
| **Project** | เชื่อมต่อกับ Git Repository ที่เก็บ Playbook/Role (เช่น `day2-ops` ที่เราสร้างใน Part นี้) — Sync อัตโนมัติทุกครั้งที่มี commit ใหม่ |
| **Inventory** | Inventory ที่ Sync มาจาก Source จริง (Static file, หรือ Dynamic จาก CMDB/NetBox) แทนการเก็บไฟล์ `.yaml` กระจายบน Laptop แต่ละคน |
| **Credential** | เก็บ Vault Password, SSH Key, TACACS+ Credential ไว้**รวมศูนย์แบบเข้ารหัส** — วิศวกรเลือกใช้ได้โดยไม่เห็นค่าจริงเลย (Credential Injection) |
| **Job Template** | จับคู่ Playbook + Inventory + Credential + Extra Variable เข้าด้วยกันเป็น "ปุ่มกดรัน" ที่ตั้งค่าไว้ล่วงหน้า |
| **RBAC (Role-Based Access Control)** | กำหนดว่า User/Team ไหนเห็น/รัน Job Template ไหนได้ — เช่น Junior Engineer รันได้แค่ Template "Backup Config" ไม่ให้รัน Template "Push AAA Config" |
| **Workflow Template** | ต่อหลาย Job Template เป็น Pipeline เดียว (เช่น Check Mode ก่อน → รอ Approval Node → รันจริง → Notify Slack) |
| **Schedule** | ตั้งให้ Job Template รันอัตโนมัติตามรอบเวลา (เช่น Backup Config ทุกวันตี 2) โดยไม่ต้องมีคนกดรันเอง |

### ตัวอย่าง Flow การทำงานจริงของทีม NetOps ที่ใช้ AWX/AAP

วิศวกร A เขียน Role ใหม่ (เช่น เพิ่ม VLAN) แล้ว commit เข้า Git → AWX Sync Project อัตโนมัติ →
วิศวกร B (RBAC อนุญาตแล้ว) เข้า Web UI กด "Launch" และตอบ Survey (เช่น "VLAN เลขที่เท่าไหร่?") ผ่าน
ฟอร์มโดยไม่ต้องแก้ YAML เอง → AWX รัน Job ด้วย Credential ที่เก็บรวมศูนย์ (วิศวกร B ไม่เห็น Vault
Password จริง) → ผลลัพธ์ถูกบันทึกใน Job History พร้อม timestamp/ผู้รัน → ถ้าตั้ง Notification ไว้
ก็ส่งเข้า Slack/Email/ITSM Ticket ต่ออัตโนมัติ

> **จุดเชื่อมกับ Part อื่นในหลักสูตร**: AWX/AAP คือจุดที่ Ansible Project แบบที่เราสร้างใน Part นี้
> (`day2-ops` พร้อม Role/Tag/Handler/Vault) กลายเป็นส่วนหนึ่งของ **CI/CD Pipeline สำหรับ
> Network-as-Code** ที่จะเจาะลึกใน Part 47/79 — Git repository เดียวกันที่ AWX ดึงมาใช้ ก็คือ
> Source of Truth เดียวกันที่ Pipeline อื่นๆใช้ตรวจสอบ (Lint, Test, Peer Review) ก่อน Merge เข้า
> branch หลัก Part นี้ตั้งใจให้เป็นแค่**ภาพรวมแนวคิด** ไม่ลงรายละเอียดการ Deploy AWX เอง (ต้องใช้
> Kubernetes/OpenShift หรือ Docker Compose ขึ้นอยู่กับเวอร์ชัน) เพราะเป็นหัวข้อระดับ DevNet
> Professional/Platform Engineering ที่แยกออกไปจากแกนหลัก CCNP ENCOR ของหลักสูตรนี้

---

## Step 480 — Lab เต็มรูปแบบ: โปรเจกต์ "day2-ops" ครบวงจร

Lab สรุปของ Part 48: รวมทุกเทคนิคที่เรียนมาตลอด Step 471-479 เข้าเป็น**โปรเจกต์ Ansible เดียว**
ที่ครอบคลุม 3 เรื่องหลักของ Day-2 Operations — **NTP/Syslog** (Part 17), **AAA Hardening**
(Part 22), และ **VLAN Provisioning** (Part 3) — ด้วยโครงสร้าง Role-based เต็มรูปแบบ, Tags,
Handlers, Loops, และ Vault-encrypted Secret ทำงานร่วมกัน แล้วสาธิตการรันแบบ `--check --diff`
ก่อนตามด้วยการรันจริงบน Inventory ทั้ง 6 อุปกรณ์ของ Lab

### โครงสร้าง Directory เต็มรูปแบบของโปรเจกต์ "day2-ops"

```
day2-ops/
├── ansible.cfg
├── requirements.yml                       # Step 472
├── inventory.yaml
├── group_vars/
│   ├── network_devices.yml                # ตัวแปรทั่วไป (ไม่ลับ) — reference vault
│   ├── network_devices_vault.yml           # Step 477 — encrypted ด้วย vault-id "lab"
│   └── dist_switches.yml                   # lab_vlans list (Step 475)
├── filter_plugins/
│   └── netmask_filters.py                  # Step 476
├── roles/
│   ├── ntp-syslog/
│   │   ├── defaults/main.yaml
│   │   ├── tasks/main.yaml
│   │   ├── templates/ntp_syslog.j2
│   │   └── handlers/main.yaml
│   ├── aaa-hardening/
│   │   ├── defaults/main.yaml
│   │   ├── tasks/main.yaml
│   │   └── handlers/main.yaml
│   └── vlan-provisioning/
│       ├── defaults/main.yaml
│       ├── tasks/main.yaml
│       └── handlers/main.yaml
└── site.yaml                               # Playbook หลัก — เรียกทั้ง 3 Role พร้อม tags
```

### `requirements.yml`

```yaml
---
collections:
  - name: cisco.ios
    version: "5.3.0"
  - name: ansible.netcommon
    version: ">=5.0.0,<6.0.0"
  - name: ansible.utils
    version: "3.1.0"
```

### `inventory.yaml`

```yaml
all:
  children:
    network_devices:
      vars:
        ansible_network_os: cisco.ios.ios
        ansible_connection: ansible.netcommon.network_cli
      children:
        core_switches:
          hosts:
            CORE-SW1: { ansible_host: 10.10.99.10 }
            CORE-SW2: { ansible_host: 10.10.99.11 }
        dist_switches:
          hosts:
            DIST-SW1: { ansible_host: 10.10.99.12 }
            DIST-SW2: { ansible_host: 10.10.99.13 }
            DIST-SW3: { ansible_host: 10.10.99.14 }
            DIST-SW4: { ansible_host: 10.10.99.15 }
```

### `group_vars/network_devices_vault.yml` (เข้ารหัสด้วย Vault-ID ชื่อ `lab`)

```bash
ansible-vault create --vault-id lab@prompt group_vars/network_devices_vault.yml
```

เนื้อหาก่อนเข้ารหัส:

```yaml
vault_ansible_password: "StrongP@ssw0rd!"
vault_ansible_become_password: "Cisco123!"
vault_tacacs_key: "CiscoLab-TACACS-Key2026!"
```

### `group_vars/network_devices.yml`

```yaml
---
ansible_user: admin
ansible_password: "{{ vault_ansible_password }}"
ansible_become: yes
ansible_become_method: enable
ansible_become_password: "{{ vault_ansible_become_password }}"
tacacs_key: "{{ vault_tacacs_key }}"

syslog_server: "10.10.99.50"          # NOC-SRV
tacacs_server_ip: "10.10.99.60"       # AAA-SRV1
mgmt_admin_subnet_wildcard: "10.10.99.0 0.0.0.255"
```

### `group_vars/dist_switches.yml`

```yaml
---
lab_vlans:
  - { id: 10, name: "SALES" }
  - { id: 20, name: "VOICE" }
  - { id: 30, name: "SERVERS" }
  - { id: 40, name: "WIFI" }
  - { id: 99, name: "MGMT" }
```

### Role 1 — `roles/ntp-syslog/`

**`defaults/main.yaml`**

```yaml
---
ntp_servers:
  - { ip: "10.10.99.50", prefer: true }
  - { ip: "10.10.99.51", prefer: false }
ntp_source_interface: "Loopback0"
syslog_trap_level: "informational"
```

**`templates/ntp_syslog.j2`**

```jinja2
{% for server in ntp_servers %}
ntp server {{ server.ip }}{{ ' prefer' if server.prefer else '' }}
{% endfor %}
ntp source {{ ntp_source_interface }}
!
logging host {{ syslog_server }}
logging trap {{ syslog_trap_level }}
logging source-interface {{ ntp_source_interface }}
```

**`tasks/main.yaml`**

```yaml
---
- name: "[ntp-syslog] Push NTP + Syslog config"
  cisco.ios.ios_config:
    src: "ntp_syslog.j2"
  register: ntp_syslog_result
  notify: "save running-config"
  tags: ["ntp", "syslog", "day2-ops"]
```

**`handlers/main.yaml`**

```yaml
---
- name: save running-config
  cisco.ios.ios_config:
    save_when: always
  listen: "save running-config"
```

### Role 2 — `roles/aaa-hardening/`

**`defaults/main.yaml`**

```yaml
---
aaa_tacacs_group: "AAA-TACACS"
aaa_tacacs_server_name: "AAA-SRV1"
```

**`tasks/main.yaml`**

```yaml
---
- name: "[aaa-hardening] Ensure AAA new-model is enabled"
  cisco.ios.ios_config:
    lines:
      - "aaa new-model"
  register: aaa_newmodel_result
  notify: "save running-config"
  tags: ["aaa", "security", "day2-ops"]

- name: "[aaa-hardening] Configure TACACS+ server and group"
  cisco.ios.ios_config:
    lines:
      - "tacacs server {{ aaa_tacacs_server_name }}"
      - " address ipv4 {{ tacacs_server_ip }}"
      - " key {{ tacacs_key }}"
      - " timeout 5"
      - " exit"
      - "aaa group server tacacs+ {{ aaa_tacacs_group }}"
      - " server name {{ aaa_tacacs_server_name }}"
      - " exit"
  register: aaa_tacacs_result
  notify: "save running-config"
  tags: ["aaa", "security", "day2-ops"]

- name: "[aaa-hardening] Enforce AAA login methods with local fallback"
  cisco.ios.ios_config:
    lines:
      - "aaa authentication login CONSOLE-IN group {{ aaa_tacacs_group }} local"
      - "aaa authentication login VTY-IN group {{ aaa_tacacs_group }} local"
  register: aaa_login_result
  notify: "save running-config"
  tags: ["aaa", "security", "day2-ops"]

- name: "[aaa-hardening] Ensure management-plane ACL exists"
  cisco.ios.ios_config:
    lines:
      - "10 permit {{ mgmt_admin_subnet_wildcard }}"
      - "20 deny any log"
    parents: "ip access-list extended MGMT-ACCESS-IN"
  register: aaa_acl_result
  notify: "save running-config"
  tags: ["acl", "security", "day2-ops"]

- name: "[aaa-hardening] Apply ACL to management VLAN interface"
  cisco.ios.ios_config:
    lines:
      - "ip access-group MGMT-ACCESS-IN in"
    parents: "interface Vlan99"
  notify: "save running-config"
  tags: ["acl", "security", "day2-ops"]
```

`handlers/main.yaml` ของ Role นี้ใช้ Pattern `listen: "save running-config"` เดียวกันกับ
`roles/ntp-syslog/handlers/main.yaml` ทุกตัวอักษร (Handler ชื่อเดียวกันทำให้ทั้ง 3 Role notify
มารวมกันและ save แค่ครั้งเดียวท้าย Play ตามที่อธิบายใน Step 473)

### Role 3 — `roles/vlan-provisioning/`

**`defaults/main.yaml`**

```yaml
---
vlan_provisioning_target_group: "dist_switches"
```

**`tasks/main.yaml`**

```yaml
---
- name: "[vlan-provisioning] Create all lab VLANs (loop over lab_vlans)"
  cisco.ios.ios_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
    state: merged
  loop: "{{ lab_vlans }}"
  loop_control:
    label: "VLAN {{ item.id }} ({{ item.name }})"
  when: "vlan_provisioning_target_group in group_names"
  register: vlan_result
  notify: "save running-config"
  tags: ["vlan", "day2-ops"]

- name: "[vlan-provisioning] Report per-VLAN change status"
  ansible.builtin.debug:
    msg: >
      {{ inventory_hostname }}: VLAN {{ item.item.id }} ({{ item.item.name }})
      {{ 'สร้าง/แก้ไขแล้ว' if item.changed else 'มีอยู่แล้ว ไม่เปลี่ยน' }}
  loop: "{{ vlan_result.results | default([]) }}"
  loop_control:
    label: "VLAN {{ item.item.id }}"
  when: "vlan_provisioning_target_group in group_names"
  tags: ["vlan", "day2-ops"]
```

Role นี้ก็ใช้ `handlers/main.yaml` Pattern เดียวกันอีกครั้ง (ดู `roles/ntp-syslog/handlers/main.yaml`
ด้านบน) — ทั้ง 3 Role notify มาที่ Handler ชื่อเดียวกัน ทำให้ `site.yaml` save config แค่ครั้งเดียว
ท้าย Play ไม่ว่าจะมีกี่ Role เปลี่ยน config พร้อมกันก็ตาม

### `site.yaml` — Playbook หลักที่รวมทั้ง 3 Role

```yaml
---
- name: "Day2-Ops Full Lab: NTP/Syslog + AAA Hardening + VLAN Provisioning"
  hosts: network_devices
  gather_facts: no
  connection: ansible.netcommon.network_cli

  roles:
    - role: ntp-syslog
      tags: ["ntp", "syslog", "day2-ops"]
    - role: aaa-hardening
      tags: ["aaa", "acl", "security", "day2-ops"]
    - role: vlan-provisioning
      tags: ["vlan", "day2-ops"]
```

### ขั้นตอนที่ 1 — ติดตั้ง Collection ตาม Pin เวอร์ชัน

```bash
cd day2-ops/
ansible-galaxy collection install -r requirements.yml
```

### ขั้นตอนที่ 2 — Dry-run ด้วย `--check --diff` ก่อนเสมอ (กฎเหล็กจาก Step 478)

```bash
ansible-playbook -i inventory.yaml site.yaml --check --diff --vault-id lab@prompt
```

```
Vault password (lab):

PLAY [Day2-Ops Full Lab: NTP/Syslog + AAA Hardening + VLAN Provisioning] ***********

TASK [ntp-syslog : [ntp-syslog] Push NTP + Syslog config] **************************
--- before
+++ after
@@ -0,0 +1,6 @@
+ntp server 10.10.99.50 prefer
+ntp server 10.10.99.51
+ntp source Loopback0
+logging host 10.10.99.50
+logging trap informational
+logging source-interface Loopback0
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

TASK [aaa-hardening : [aaa-hardening] Ensure AAA new-model is enabled] **************
--- before
+++ after
@@ -0,0 +1 @@
+aaa new-model
changed: [CORE-SW1]
...

TASK [aaa-hardening : [aaa-hardening] Configure TACACS+ server and group] ***********
--- before
+++ after
@@ -0,0 +1,7 @@
+tacacs server AAA-SRV1
+ address ipv4 10.10.99.60
+ key CiscoLab-TACACS-Key2026!
+ timeout 5
+aaa group server tacacs+ AAA-TACACS
+ server name AAA-SRV1
changed: [CORE-SW1]
...

TASK [vlan-provisioning : [vlan-provisioning] Create all lab VLANs (loop over lab_vlans)] ***
changed: [DIST-SW1] => (item=VLAN 10 (SALES))
changed: [DIST-SW1] => (item=VLAN 20 (VOICE))
changed: [DIST-SW1] => (item=VLAN 30 (SERVERS))
changed: [DIST-SW1] => (item=VLAN 40 (WIFI))
changed: [DIST-SW1] => (item=VLAN 99 (MGMT))
skipping: [CORE-SW1]   # <-- Core Switch ไม่อยู่ใน group dist_switches ตาม when: (Step 475)
skipping: [CORE-SW2]

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=3  changed=3  unreachable=0  failed=0  skipped=1
DIST-SW1  : ok=3  changed=3  unreachable=0  failed=0  skipped=0
```

Diff ยืนยันชัดเจนว่า Core Switch จะไม่โดน Task VLAN Provisioning เลย (ตรงตาม Design ของ IP Plan
ที่ Core เป็น Routed Port ล้วน) และเห็นทุกบรรทัด config ที่จะถูกเพิ่มก่อนรันจริง 100%

### ขั้นตอนที่ 3 — รันจริงหลัง Review Diff แล้วมั่นใจ

```bash
ansible-playbook -i inventory.yaml site.yaml --vault-id lab@prompt
```

```
PLAY RECAP ***************************************************************************
CORE-SW1  : ok=3  changed=3  unreachable=0  failed=0  skipped=1
CORE-SW2  : ok=3  changed=3  unreachable=0  failed=0  skipped=1
DIST-SW1  : ok=6  changed=6  unreachable=0  failed=0  skipped=0
DIST-SW2  : ok=6  changed=6  unreachable=0  failed=0  skipped=0
DIST-SW3  : ok=6  changed=6  unreachable=0  failed=0  skipped=0
DIST-SW4  : ok=6  changed=6  unreachable=0  failed=0  skipped=0
```

### ขั้นตอนที่ 4 — รันเฉพาะ Tag เดียว (เช่น อัปเดตเฉพาะ VLAN ในภายหลัง)

```bash
ansible-playbook -i inventory.yaml site.yaml --tags vlan --vault-id lab@prompt
```

```
PLAY RECAP ***************************************************************************
DIST-SW1  : ok=2  changed=0  unreachable=0  failed=0  skipped=0
DIST-SW2  : ok=2  changed=0  unreachable=0  failed=0  skipped=0
```

`changed=0` เพราะ VLAN ถูกสร้างไปแล้วในรอบก่อน (idempotent) และ Task ของ Role อื่น (ntp-syslog,
aaa-hardening) ถูก**ข้ามไปทั้งหมด** เพราะไม่ตรงกับ `--tags vlan` — พิสูจน์ว่าการรวม **Role + Tag +
Handler + Loop + Vault** เข้าด้วยกันทำให้โปรเจกต์เดียวนี้ใช้งานได้ทั้งแบบ "รันทุกอย่างครั้งเดียว"
และ "รันเฉพาะส่วนที่ต้องการ" โดยไม่ต้องมี Playbook แยกหลายไฟล์ซ้ำซ้อนกันเลย

### ขั้นตอนที่ 5 — รันซ้ำทั้งโปรเจกต์อีกครั้งเพื่อพิสูจน์ Idempotency เต็มรูปแบบ

```bash
ansible-playbook -i inventory.yaml site.yaml --vault-id lab@prompt
```

```
PLAY RECAP ***************************************************************************
CORE-SW1  : ok=3  changed=0  unreachable=0  failed=0  skipped=1
CORE-SW2  : ok=3  changed=0  unreachable=0  failed=0  skipped=1
DIST-SW1  : ok=6  changed=0  unreachable=0  failed=0  skipped=0
DIST-SW2  : ok=6  changed=0  unreachable=0  failed=0  skipped=0
DIST-SW3  : ok=6  changed=0  unreachable=0  failed=0  skipped=0
DIST-SW4  : ok=6  changed=0  unreachable=0  failed=0  skipped=0
```

`changed=0` ทุกอุปกรณ์ และไม่มี `RUNNING HANDLER` ปรากฏเลย — โปรเจกต์ "day2-ops" นี้พร้อมสำหรับ
การใส่เข้า Schedule ของ AWX/AAP (Step 479) ให้รันซ้ำทุกวันโดยปลอดภัย 100% ซึ่งเป็นเป้าหมายสุดท้าย
ของ Ansible Project ระดับ Production ทุกโปรเจกต์

---

## แบบฝึกหัดทวนความเข้าใจ Part 48

1. Ansible Role กับ Playbook แบบ Flat File (ไฟล์เดียวยาว) ต่างกันอย่างไร และโครงสร้างโฟลเดอร์
   มาตรฐาน 3 อันดับแรกที่ Role ต้องมี (ที่ Ansible ค้นหาโดยอัตโนมัติ) คืออะไร?
2. `requirements.yml` มีไว้แก้ปัญหาอะไร และเพราะเหตุใดการ Pin เวอร์ชัน Collection แบบเจาะจง
   (`version: "5.3.0"`) จึงสำคัญกว่าการ `ansible-galaxy collection install cisco.ios` เฉยๆ?
3. Handler ต่างจาก Task ปกติอย่างไร และเพราะเหตุใดการใช้ `notify` + Handler จึงดีกว่าการเขียน
   `when: <var>.changed` คลุม Task save config เองแบบใน Part 23 Step 227?
4. ในการรัน Playbook ด้วย `--tags security` เทียบกับ `--tags aaa` ต่างกันอย่างไร ถ้า Task หนึ่งมี
   `tags: ["security", "aaa"]` ทั้งสองคำสั่งจะรัน Task นี้เหมือนกันหรือไม่?
5. `--check --diff` ทำอะไรบ้าง และเพราะเหตุใดจึงควรใช้คำสั่งนี้ก่อนรัน Playbook จริงกับอุปกรณ์
   Production ทุกครั้ง (โดยเฉพาะเมื่อมี Handler ที่ทำการ save config ผูกอยู่ด้วย)?

**เฉลย:**

1. Playbook แบบ Flat File เขียนทุก Task รวมไว้ในไฟล์เดียว ทำให้ reuse ไม่ได้ ทดสอบแยกส่วนไม่ได้
   และแก้ไขร่วมกันในทีมยาก ส่วน Role คือการแบ่ง Task/Variable/Template/Handler ออกเป็นโฟลเดอร์ตาม
   หน้าที่ ทำให้ copy ไปใช้ในโปรเจกต์อื่นได้ทั้งโฟลเดอร์ — 3 โฟลเดอร์หลักที่ Ansible ค้นหาอัตโนมัติ
   คือ `tasks/main.yaml` (entry point), `defaults/main.yaml` (ตัวแปรเริ่มต้น priority ต่ำสุด),
   และ `handlers/main.yaml` (พร้อมใช้กับ `notify` โดยไม่ต้อง include เอง)
2. `requirements.yml` แก้ปัญหา "รันบนเครื่องไหน วันไหน ก็ต้องได้ Collection เวอร์ชันเดียวกัน" —
   ถ้าใช้ `ansible-galaxy collection install cisco.ios` เฉยๆจะได้เวอร์ชันล่าสุด ณ วันที่รันเสมอ
   ทำให้เพื่อนร่วมทีมที่ install วันอื่นอาจได้เวอร์ชันต่างกัน และ Playbook อาจทำงานต่างกันโดยไม่
   ตั้งใจ (behavior เปลี่ยนตาม module version) การ Pin เวอร์ชันเจาะจงทำให้โปรเจกต์ Reproducible
   ทุกเครื่อง ทุกครั้งที่รัน
3. Handler เป็น Task พิเศษที่จะไม่ถูกรันเลยเว้นแต่มี Task อื่น `notify` มา และ Task นั้น report
   `changed` เท่านั้น ต่างจาก Task ปกติที่รันตามลำดับเสมอ — การใช้ `notify` ดีกว่าการเขียน
   `when: <var>.changed` เองเพราะ (1) Ansible รวม notify ซ้ำจากหลาย Task/Role ให้ Handler รันแค่
   ครั้งเดียวท้าย Play อัตโนมัติ ไม่ต้องเขียน logic ตรวจสอบ merge เอง และ (2) ทำให้ Task หลาย Role
   (เช่น NTP, Syslog, AAA) เรียก Handler ตัวเดียวกันร่วมกันได้ง่ายด้วย `listen:`
4. `--tags security` จะรันทุก Task ที่มี Tag "security" ติดอยู่ (รวมทั้ง Task ที่ tag เป็น
   `["security", "aaa"]` และ Task อื่นที่ tag เป็น `["security", "acl"]`) ส่วน `--tags aaa` จะรัน
   เฉพาะ Task ที่มี Tag "aaa" ติดอยู่เท่านั้น (ไม่รวม Task ที่ tag แค่ `["security", "acl"]`) — ถ้า
   Task หนึ่งมี `tags: ["security", "aaa"]` ทั้งสองคำสั่งจะรัน Task นี้เหมือนกัน เพราะ Task นี้มี
   ทั้งสอง Tag อยู่ในตัวมันเอง แต่ Task อื่นที่มีแค่ Tag เดียวอาจถูกเลือกต่างกัน
5. `--check` จำลองการรัน Playbook (Dry-run) โดยไม่ push อะไรเข้าอุปกรณ์จริง และไม่ trigger Handler
   จริง ส่วน `--diff` แสดงบรรทัด config ที่จะถูกเพิ่ม/ลบแบบละเอียด (เหมือน `git diff`) — ควรใช้
   คู่กันก่อนรันจริงเสมอเพราะทำให้วิศวกรเห็นผลกระทบที่แน่นอนก่อน Apply Config กับอุปกรณ์
   Production โดยไม่มีความเสี่ยงเลย โดยเฉพาะเมื่อมี Handler ที่ save running-config ผูกอยู่ —
   ถ้ารันแบบ `--check` แล้ว Handler ไม่ถูกเรียกจริง จึงปลอดภัย 100% ที่จะทดสอบก่อนโดยไม่กระทบ
   startup-config ของอุปกรณ์เลย

---

## สรุป Part 48

Part นี้ยกระดับ Ansible จาก "รู้จักพื้นฐาน" (Part 23) ไปสู่ "ใช้งานได้ระดับ Production" อย่างเต็ม
รูปแบบ — เราเริ่มจากการจัดโครงสร้าง Playbook แบบ Flat File ให้เป็น **Role** ที่ reuse ได้จริง,
เรียนรู้การ Pin เวอร์ชัน Collection ด้วย `requirements.yml` ผ่าน **Ansible Galaxy**, ใช้
**Handler + notify** เพื่อ save config เฉพาะเมื่อจำเป็นจริง, จัดระเบียบ Playbook ขนาดใหญ่ด้วย
**Tags** ให้เลือกรันเฉพาะส่วน, ใช้ **Conditional (`when`) และ Loop (`loop`)** สร้าง VLAN ทั้ง 5
ตัวจาก List เดียวโดยไม่ Hardcode, เขียน **Custom Jinja2 Filter** ของตัวเองเพื่อ Render Config
เต็มรูปแบบจาก `host_vars` ที่มีโครงสร้าง, เจาะลึก **Ansible Vault** ด้วย Vault-ID สำหรับแยก
Environment dev/prod อย่างปลอดภัย, ยึดหลัก **`--check --diff`** เป็นขั้นตอนบังคับก่อน Apply Config
จริงเสมอ, มองภาพรวม **AWX/Ansible Automation Platform** ที่พา Automation จาก Laptop คนเดียวไปสู่
Platform ระดับทีม NetOps เต็มรูปแบบ, และปิดท้ายด้วย Lab เต็มรูปแบบ **"day2-ops"** ที่รวมทุกเทคนิค
เข้าเป็นโปรเจกต์เดียวที่ครอบคลุม NTP/Syslog + AAA Hardening + VLAN Provisioning พร้อม Role, Tag,
Handler, Loop, และ Vault ทำงานร่วมกันได้จริง ✅

ทักษะ Ansible ระดับนี้คือมาตรฐานที่ทีม NetOps ระดับ Enterprise ใช้ดูแล Config หลายร้อยอุปกรณ์
พร้อมกันในชีวิตจริง และเป็นพื้นฐานสำคัญก่อนต่อยอดไปสู่ Model-Driven API เต็มรูปแบบ — Part ต่อไปเรา
จะเจาะลึก **NETCONF/RESTCONF และ YANG Data Model** ที่แอบดูไว้สั้นๆใน Part 23 Step 229 อย่าง
ละเอียด รวมถึงการเขียน Python ด้วย `ncclient` และแนวคิด Model-Driven Telemetry

**ไปต่อ:** [Part 49 — NETCONF/RESTCONF & YANG Deep Dive →](part-049-netconf-restconf-yang-deep-dive.md)
