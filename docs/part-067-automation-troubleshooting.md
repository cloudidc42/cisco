# Part 67 — Automation Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 661–670 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 66 — Wireless Troubleshooting](part-066-wireless-troubleshooting.md) ที่เจาะ
> Troubleshoot ปัญหา RF/WLC/AP ระดับ ENARSI Part นี้เปลี่ยนโฟกัสไปที่แกนที่สองของหลักสูตรนี้ทั้งหมด
> คือ **Automation** — [Part 23](part-023-automation-basics.md) สอน Netmiko/Ansible/Jinja2
> พื้นฐาน, [Part 47](part-047-automation-python-advanced.md) ยกระดับ Python ด้วย NAPALM/Nornir/
> TextFSM/Compliance Checking/Concurrency, [Part 48](part-048-automation-ansible-advanced.md)
> ยกระดับ Ansible ด้วย Role/Vault/Idempotency Verification, และ
> [Part 49](part-049-netconf-restconf-yang-deep-dive.md) เจาะ NETCONF/RESTCONF/YANG เต็มรูปแบบ —
> ทุก Tool ที่สร้างมาทั้งหมดนี้ **พังได้เหมือนกับ Protocol ทั่วไป** เพียงแต่พังในรูปแบบที่ไม่มี
> `debug ip ospf` หรือ `show ip route` ให้ดู วิศวกร Automation ต้อง Troubleshoot **Script, Pipeline,
> Credential, และ Data Parsing** ด้วยวิธีคิดที่เป็นระบบเหมือนกับที่ใช้ Troubleshoot Routing
> Protocol — Part นี้จะสอนวิธีวินิจฉัยปัญหา Automation ที่พบจริงในงาน NetOps ทีละ Scenario
> พร้อม Root Cause และวิธีแก้ที่ผูกกลับไปยังทุก Part ของสาย Automation ที่เรียนมา

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 661 | Automation Troubleshooting Methodology — Failure Category และ Diagnostic Checklist |
| 662 | Scenario 1: Netmiko เชื่อมต่อไม่ได้ทุกอุปกรณ์หลัง IOS Upgrade — SSH Key Exchange Mismatch |
| 663 | Scenario 2: Ansible Playbook `changed` ทุกรอบ — Idempotency พังจาก Line Ending/Whitespace |
| 664 | Scenario 3: TextFSM Parse ได้ค่าว่าง/ผิดเงียบๆ หลัง IOS Upgrade เปลี่ยนคำใน `show` Output |
| 665 | Scenario 4: NETCONF `edit-config` ล้มเหลวด้วย YANG Validation Error ที่อ่านไม่เข้าใจ |
| 666 | Scenario 5: Ansible Vault Decrypt ไม่ผ่านบน CI/CD Runner แต่รันได้ปกติบนเครื่อง Local |
| 667 | Scenario 6: Compliance Check รายงาน False-Positive Drift — Golden Config ไม่ทันการเปลี่ยนแปลงจริง |
| 668 | Scenario 7: RESTCONF คืนค่า 401 Unauthorized เป็นพักๆ — Session/AAA Limit ใน Long-Running Script |
| 669 | Scenario 8: Nornir/ThreadPoolExecutor ทำ CPU อุปกรณ์รุ่นเก่าพุ่ง — ต้องจำกัด Concurrency ตาม Platform |
| 670 | Lab เต็มรูปแบบ: วินิจฉัย "Automation Pipeline พังทั้งคืน" — Vault Fail + Cipher Mismatch พร้อมกัน |

---

## Step 661 — Automation Troubleshooting Methodology

### 661.1 ทำไม Automation Troubleshooting ต่างจาก Protocol Troubleshooting

Troubleshooting OSPF/BGP/NAT ที่เรียนมาตลอด Part 56-66 มี "จุดตรวจ" ที่ชัดเจนตาม OSI Layer —
Layer 1 → Layer 2 → Layer 3 → Application เสมอ แต่ **Automation ไม่ใช่ Protocol ตัวเดียว**
มันคือ **Stack ของหลายเลเยอร์ที่ประกอบกัน**: Script/Playbook ของเรา → Library (Netmiko/NAPALM/
Ansible Module) → Transport (SSH/HTTPS/NETCONF) → อุปกรณ์ปลายทาง (IOS-XE) → และในหลายกรณี
มี **Credential Store** (Vault) กับ **CI/CD Runner** เป็นเลเยอร์เพิ่มเข้ามาอีก — ปัญหาจึงเกิดได้จาก
"เลเยอร์ของ Automation" ไม่ใช่แค่เลเยอร์ของ Network

> **หลักคิดสำคัญที่สุดของ Part นี้**: เมื่อ Automation Job ล้มเหลว คำถามแรกที่ต้องถามไม่ใช่
> "OSPF/BGP มีปัญหาไหม" แต่คือ **"ล้มเหลวที่ Category ไหนของ Automation Stack"** — เพราะวิธีแก้และ
> คนที่ต้องเข้ามาช่วย (Network Engineer, DevOps, Security) ต่างกันโดยสิ้นเชิงในแต่ละ Category

### 661.2 สี่ Failure Category ของ Automation

| Category | อาการที่พบ | ตัวอย่าง Exception/Error | อยู่ในความรับผิดชอบของ |
|---|---|---|---|
| **1. Connectivity/Auth** | Connect ไม่ได้เลย, Timeout, Authentication ล้มเหลว | `NetmikoTimeoutException`, `NetmikoAuthenticationException`, SSH `kex` error, HTTP 401/403 | Network + Security (Credential/ACL/Firewall) |
| **2. Parsing/Data** | Connect ได้ปกติ แต่ข้อมูลที่ได้ผิด/ว่าง/รูปแบบเพี้ยน | TextFSM คืน `[]`, KeyError, YANG validation error, JSON schema mismatch | Automation Engineer (ต้อง sync กับ Vendor Output ที่เปลี่ยน) |
| **3. Logic/Idempotency** | รันได้ผลลัพธ์ปกติ แต่ "ตัดสินใจ" ผิด — เช่น `changed` ทุกรอบ, Compliance False-Positive | Ansible `changed=6` ตลอด, Compliance Report ไม่ตรงความจริง | Automation Engineer (Logic/Template Bug) |
| **4. API/Version/Scale** | ใช้งานได้ช่วงแรกแล้วเสื่อมลงเรื่อยๆ, พังเฉพาะบางอุปกรณ์/บางช่วงเวลา | Session Limit, CPU Spike จาก Concurrency, Version Drift ของ Collection/Module | Automation Engineer + Platform Capacity |

### 661.3 Diagnostic Checklist — ลำดับการถามคำถามเมื่อ Automation Job ล้มเหลว

```
Automation Job ล้มเหลว
    │
    ▼
Q1: ล้มเหลว "ทุกอุปกรณ์" หรือ "เฉพาะบางอุปกรณ์"?
    │                               │
   ทุกอุปกรณ์                    เฉพาะบางตัว
    │                               │
    ▼                               ▼
Q2: เพิ่งพังตอนนี้             Q3: อุปกรณ์ที่พังมีอะไร
   หรือพังมานาน?                  ร่วมกัน? (Platform เดียวกัน,
    │                             IOS Version เดียวกัน, IP Range
   เพิ่งพัง → เช็ค Changelog       เดียวกัน, รันพร้อมกันช่วงเวลาเดียว)
   (IOS Upgrade? Credential            │
   Rotation? CI Config เปลี่ยน?)       ▼
    │                             มักเป็น Category 1 (เฉพาะกลุ่มที่
    ▼                             เพิ่ง Upgrade) หรือ Category 4
Q4: Error เกิดที่ Connection      (เฉพาะกลุ่มที่ทรัพยากรจำกัด)
   หรือเกิดหลัง Connect สำเร็จ?
    │                    │
  ที่ Connection        หลัง Connect สำเร็จ
    │                    │
    ▼                    ▼
 Category 1          Category 2/3 (ดู error
 (SSH/HTTPS/Auth)     เป็น Parsing หรือ Logic)
```

### 661.4 คำสั่ง/เทคนิคตรวจสอบพื้นฐานที่ใช้ซ้ำได้ทุก Scenario

| เครื่องมือ | ใช้ตรวจอะไร |
|---|---|
| `ssh -vvv user@host` | ดู SSH Handshake ดิบ (kex algorithm, cipher, auth method) — ไม่ผ่าน library ใดๆ เลย |
| `ansible-playbook --check --diff -vvv` | ดู Diff จริงและ Log การตัดสินใจของ Module แบบละเอียด (Part 48 Step 478) |
| `python3 -c "import netmiko, paramiko, cryptography; print(...)"` | เทียบเวอร์ชัน Library ที่ติดตั้งจริงกับที่ Pin ไว้ใน `requirements.txt` |
| `git log -p <file>` | ดูว่า Template/Script/Golden Config เปลี่ยนอะไรไปเมื่อไหร่ (Part 47 Step 468) |
| `show ip ssh` / `show running-config \| include ssh` | ดู Algorithm ที่อุปกรณ์ IOS-XE เปิดรับจริง ณ ขณะนี้ |
| Diff CI Runner Log vs Local Run Log | หา "สิ่งที่ต่างกันระหว่าง Environment" — Category 1 (Vault/Credential) ส่วนใหญ่ซ่อนอยู่ที่นี่ |

Checklist นี้จะถูกใช้ซ้ำใน Step 662-670 ทุก Scenario — สังเกตว่าขั้นแรกของทุก Scenario คือ
**"แยก Category ก่อนแก้"** ไม่ใช่การรีบแก้ตามอาการที่เห็นทันที

---

## Step 662 — Scenario 1: Netmiko เชื่อมต่อไม่ได้ทุกอุปกรณ์หลัง IOS Upgrade

### 662.1 อาการ

Cron Job ที่รัน `push_ntp_config_hardened.py` (Part 47 Step 466) ทุกคืนมาหลายเดือนโดยไม่มีปัญหา
เช้าวันนี้ทีม NetOps พบว่า Log แจ้ง **ล้มเหลวทั้ง 6 อุปกรณ์พร้อมกัน** — ตรงกับ Category 1
(Connectivity/Auth) และ "ล้มเหลวทุกอุปกรณ์พร้อมกัน" ตาม Checklist Step 661.3

```
2026-09-26 02:00:04 [WARNING] netops: connect_and_push ล้มเหลวครั้งที่ 1/4: SSHException: ...
2026-09-26 02:00:14 [WARNING] netops: connect_and_push ล้มเหลวครั้งที่ 2/4: SSHException: ...
2026-09-26 02:00:30 [WARNING] netops: connect_and_push ล้มเหลวครั้งที่ 3/4: SSHException: ...
2026-09-26 02:00:58 [ERROR] netops: connect_and_push ล้มเหลวครบ 4 ครั้ง ยอมแพ้
2026-09-26 02:00:58 [ERROR] netops: CORE-SW1: เกิด error ที่ไม่คาดคิด
Traceback (most recent call last):
  ...
paramiko.ssh_exception.SSHException: Incompatible ssh peer (no acceptable kex algorithm)
```

สังเกตว่า Retry Decorator จาก Step 466 **ไม่ retry Exception ประเภทนี้เลย** เพราะเราตั้งใจให้
retry เฉพาะ `NetmikoTimeoutException` (ปัญหาชั่วคราว) — `SSHException` แบบนี้เป็นปัญหาถาวร (จะพัง
เหมือนเดิมทุกครั้งไม่ว่า retry กี่รอบ) ซึ่งเป็นการออกแบบที่ถูกต้องแล้ว

### 662.2 ตอบคำถาม Checklist: "เพิ่งพังตอนนี้ หรือพังมานาน?"

```bash
$ git log --oneline -- push_ntp_config_hardened.py requirements.txt
# ไม่มีการแก้ script/requirements.txt เลยในช่วง 3 เดือนที่ผ่านมา -> ไม่ใช่ script เปลี่ยน

$ # เช็ค Change Log ของ Network Team (Change Management Ticket)
CHG0004821: Upgrade IOS-XE บน CORE-SW1/CORE-SW2/DIST-SW1-4 จาก 17.09.04a -> 17.12.02
            ทำ Maintenance Window เมื่อคืนนี้ 01:00-01:45 (ก่อน Cron Job รัน 02:00 พอดี)
```

**เพิ่งพังตอนนี้ + มี IOS Upgrade ตรงเวลาก่อนพังเป๊ะ** — นี่คือหลักฐานสำคัญที่สุด: ปัญหาไม่ได้อยู่ที่
Script แต่อยู่ที่ **สิ่งที่เปลี่ยนไปบนอุปกรณ์**

### 662.3 ยืนยัน Root Cause ด้วย `ssh -vvv` (ตัดผ่าน Netmiko/Paramiko ออกทั้งหมด)

```bash
$ ssh -vvv admin@10.10.99.10
...
debug1: SSH2_MSG_KEXINIT sent
debug1: kex: algorithm: (no match)
debug1: kex_input_kexinit: SSH2_MSG_KEXINIT
debug1: kex: server: diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,
        ecdh-sha2-nistp256,ecdh-sha2-nistp384
debug1: kex: client: diffie-hellman-group14-sha1,diffie-hellman-group1-sha1,
        diffie-hellman-group-exchange-sha1
Unable to negotiate with 10.10.99.10 port 22: no matching key exchange method found.
Their offer: diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,ecdh-sha2-nistp256,ecdh-sha2-nistp384
```

พฤติกรรม Default ของ IOS-XE 17.12 (เทียบกับ 17.09) **เข้มงวดขึ้นด้าน Cryptographic Hardening**
— ปิด KEX Algorithm รุ่นเก่าที่อ่อนแอ (`diffie-hellman-group1-sha1`, `diffie-hellman-group14-sha1`
บาง Build) โดย Default ทันทีหลัง Upgrade **โดยไม่มีการแจ้งใน Release Note ระดับที่ทีม NetOps
ทันสังเกต** — ฝั่ง Client คือเวอร์ชัน Paramiko ที่ Pin ไว้ใน `requirements.txt` ของโปรเจกต์
(เก่ากว่า 1 ปี) ไม่รู้จัก `diffie-hellman-group16-sha512`/`group18-sha512` เลย ทำให้ต่อไม่ได้
สักตัวเดียวพร้อมกันทันทีที่ Maintenance Window เสร็จ

### 662.4 ตรวจสอบเวอร์ชัน Library ที่ติดตั้งจริงเทียบกับที่ Pin ไว้

```bash
$ pip show paramiko | grep Version
Version: 2.7.2          # <-- เก่ามาก ไม่รองรับ group16/group18-sha512

$ cat requirements.txt
netmiko==3.4.0
paramiko==2.7.2          # <-- Pin ไว้นานแล้ว ไม่มีใครอัปเดต
textfsm==1.1.3
ntc-templates==3.1.0
```

### 662.5 ตรวจสอบฝั่ง IOS-XE ว่ารองรับ Algorithm อะไรบ้าง

```
CORE-SW1# show ip ssh
SSH Enabled - version 2.0
Authentication methods:publickey,keyboard-interactive,password
Authentication Timeout: 120 secs; Retry Count: 3
Diffie-Hellman Group  : dh-group14-sha256, dh-group16-sha512, dh-group18-sha512
Encryption Algorithms : aes128-ctr aes192-ctr aes256-ctr aes128-gcm@openssh.com aes256-gcm@openssh.com
```

### 662.6 แก้ไข — วิธีที่ถูกต้อง (อัปเกรด Library) vs วิธีที่ไม่ควรทำ (ลด Security อุปกรณ์)

| วิธีแก้ | รายละเอียด | ควรใช้เมื่อไหร่ |
|---|---|---|
| ❌ **สั่งให้ IOS-XE เปิด KEX เก่ากลับมา** (`ip ssh server algorithm kex diffie-hellman-group14-sha1`) | ลด Security ของอุปกรณ์ทั้งองค์กรเพียงเพื่อให้ Script เก่าทำงานได้ | **ไม่ควรใช้เด็ดขาด** ยกเว้นเป็นการแก้ปัญหาชั่วคราวสุดๆก่อนย้ายไปวิธีที่ถูกต้อง |
| ✅ **อัปเกรด `paramiko`/`cryptography` ในโปรเจกต์** | แก้ที่ต้นเหตุจริง (Client เก่าเกินไป) โดยไม่ลด Security ของอุปกรณ์เลย | วิธีมาตรฐานเสมอ |

```bash
# แก้ requirements.txt ให้ทันสมัยขึ้น (ตรวจ Compatibility กับ Netmiko เวอร์ชันที่ใช้ก่อน pin จริง)
pip install --upgrade paramiko cryptography netmiko
pip freeze | grep -E "paramiko|cryptography|netmiko"
```

```
paramiko==3.4.0
cryptography==42.0.5
netmiko==4.3.0
```

```bash
$ python3 push_ntp_config_hardened.py
```

```
2026-09-26 09:10:02 [INFO] netops: เริ่ม push config ไปยัง CORE-SW1 (10.10.99.10)
2026-09-26 09:10:04 [INFO] netops: CORE-SW1: push config สำเร็จ
2026-09-26 09:10:04 [INFO] netops: เริ่ม push config ไปยัง CORE-SW2 (10.10.99.11)
2026-09-26 09:10:06 [INFO] netops: CORE-SW2: push config สำเร็จ
...
2026-09-26 09:10:20 [INFO] netops: สรุปผล: สำเร็จ 6/6, ล้มเหลว 0
```

### 662.7 มาตรการป้องกันไม่ให้เกิดซ้ำ

- เพิ่ม **Pre-flight Check** เข้า Pipeline: ก่อนรัน Job จริงทุกครั้ง ให้ทดสอบ `ssh -vvv` (หรือ
  Netmiko `ConnectHandler` แบบ dry connect ที่ไม่ push อะไร) ไปยังอุปกรณ์ตัวแทน 1 ตัวก่อน — ถ้า
  ล้มเหลวด้วย `SSHException` (ไม่ใช่ Timeout) ให้ Job หยุดทันทีและแจ้งเตือนแยกจาก Timeout ปกติ
- ผูก Cron Job/CI Pipeline เข้ากับ **Change Calendar เดียวกันกับ Network Team** — ถ้ามี
  Maintenance Window (IOS Upgrade) ให้ Pipeline หยุดพักอัตโนมัติ 1 รอบเพื่อให้คนตรวจสอบ
  Compatibility ก่อน แทนที่จะรันชนกันแบบ Unattended ทันที
- ตั้ง Policy Pin เวอร์ชัน `paramiko`/`cryptography` ให้ **ไม่เก่ากว่า N เดือน** เสมอ (คล้ายแนวคิด
  `requirements.yml` Pin ของ Ansible ใน Part 48 Step 472 แต่กลับด้าน — ต้อง Pin ไม่ให้เก่าเกินไป
  ไม่ใช่ Pin ให้ตายตัวตลอดกาล)

---

## Step 663 — Scenario 2: Ansible Playbook `changed` ทุกรอบ — Idempotency พัง

### 663.1 อาการ

Playbook `site.yaml` ที่ใช้ Role `ntp-config` (Part 48 Step 471-473) เคยรันแล้ว `changed=0`
มาตลอดหลายสัปดาห์ (พิสูจน์ Idempotency ผ่านแล้วตาม Step 478) จนกระทั่งมีวิศวกรอีกคนแก้ไข
`templates/ntp_syslog.j2` เพื่อเพิ่ม Comment อธิบาย Template แล้ว Commit เข้า Git — หลังจากนั้น
Playbook รายงาน `changed=6` **ทุกครั้งที่รัน** แม้ Config บนอุปกรณ์จะไม่มีอะไรผิดปกติเลย

```bash
$ ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
changed: [CORE-SW1]
changed: [CORE-SW2]
changed: [DIST-SW1]
changed: [DIST-SW2]
changed: [DIST-SW3]
changed: [DIST-SW4]

RUNNING HANDLER [ntp-config : save running-config] *********************************
changed: [CORE-SW1]
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=3  changed=2  unreachable=0  failed=0  skipped=0
```

รันซ้ำอีกรอบทันที (ไม่มีใครแก้ Config หรือ Template เพิ่มเลย) — **ยัง `changed=6` เหมือนเดิม**
ทุกครั้ง นี่คือสัญญาณคลาสสิกของ Category 3 (Logic/Idempotency) ไม่ใช่ Category 1

### 663.2 ใช้ `--check --diff` (Part 48 Step 478) เพื่อดูว่า "เปลี่ยนอะไร" จริงๆ

```bash
$ ansible-playbook -i inventory.yaml site.yaml --check --diff --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
--- before
+++ after
@@ -1,6 +1,6 @@
 ntp server 10.10.99.50 prefer
 ntp server 10.10.99.51
-ntp source Loopback0
+ntp source Loopback0
 !
 logging host 10.10.99.50
 logging trap informational

changed: [CORE-SW1]
```

**บรรทัดที่ลบและเพิ่มดูเหมือนกันทุกตัวอักษร** (`ntp source Loopback0` ทั้งสองบรรทัด) — นี่คือ
สัญญาณว่ามี**อักขระที่มองไม่เห็น**อยู่ในบรรทัดหนึ่ง (Whitespace, Tab, หรือ Carriage Return)

### 663.3 ขุดลึกด้วย `cat -A` เพื่อเผยอักขระที่ซ่อนอยู่

```bash
# Render Template ออกมาเป็นไฟล์เพื่อตรวจสอบตรงๆ (เหมือน Step 476 dry-render)
$ ansible-playbook -i inventory.yaml playbooks/render_full_config.yaml
$ cat -A generated_configs/DIST-SW1-full.cfg | grep -A1 "ntp source"
```

```
ntp source Loopback0^M$
!$
```

`^M` คือสัญลักษณ์ที่ `cat -A` ใช้แสดง **Carriage Return (`\r`)** — เทียบกับ Running-config ของ
อุปกรณ์จริงที่ใช้ Line Ending แบบ `\n` (LF) ล้วน ไม่มี `\r` เลย

### 663.4 Root Cause: วิศวกรแก้ไฟล์ Template บน Windows Editor ที่บันทึกเป็น CRLF

```bash
$ git log -p templates/ntp_syslog.j2 | head -30
```

```
commit 8f2a1c3 (HEAD -> main)
Author: Engineer-B <engineer-b@company.local>
Date:   Wed Sep 24 14:20:11 2026 +0700

    เพิ่ม comment อธิบาย NTP source interface ใน template

diff --git a/templates/ntp_syslog.j2 b/templates/ntp_syslog.j2
index 3a1f9c2..7e8b1a0 100644
--- a/templates/ntp_syslog.j2
+++ b/templates/ntp_syslog.j2
@@ -1,7 +1,7 @@
-{% for server in ntp_servers %}
-ntp server {{ server.ip }}{{ ' prefer' if server.prefer else '' }}
-{% endfor %}
-ntp source {{ ntp_source_interface }}
+{% for server in ntp_servers %}^M
+ntp server {{ server.ip }}{{ ' prefer' if server.prefer else '' }}^M
+{% endfor %}^M
+ntp source {{ ntp_source_interface }}^M
```

Engineer-B เปิดไฟล์ `.j2` ด้วย Text Editor บน Windows ที่ตั้งค่า Line Ending เป็น CRLF (`\r\n`)
Default แล้ว Save กลับ — Git commit เข้ามาโดยไม่มี guard ป้องกัน ทำให้**ทุกบรรทัดของไฟล์**มี `\r`
ต่อท้าย Jinja2 render ค่านี้ผ่านไปตรงๆ (`\r` ไม่ใช่ Whitespace ที่ Jinja `trim_blocks`/`lstrip_blocks`
ตัดออกให้) เมื่อ `cisco.ios.ios_config` เทียบบรรทัดที่ Render ได้ (`ntp source Loopback0\r`) กับ
บรรทัดจริงบนอุปกรณ์ (`ntp source Loopback0` ไม่มี `\r`) **มันคือคนละ String กันในสายตาโปรแกรม**
แม้จะดูเหมือนกันตอนพิมพ์ออกหน้าจอ — module จึงเห็นว่า "ไม่ match" แล้วสั่ง push ใหม่ทุกครั้ง
(แต่ push ไปแล้วก็ยัง mismatch อีกในรอบถัดไป เพราะ Template แหล่งต้นทางยังมี `\r` อยู่เหมือนเดิม
กลายเป็นวนลูป `changed` ตลอดกาล)

### 663.5 แก้ไข — ทำความสะอาดไฟล์ + ป้องกันไม่ให้เกิดซ้ำด้วย `.gitattributes`

```bash
# แปลง CRLF -> LF ในไฟล์ที่มีปัญหา
$ sed -i 's/\r$//' templates/ntp_syslog.j2
$ cat -A templates/ntp_syslog.j2 | head -3
ntp server {{ server.ip }}{{ ' prefer' if server.prefer else '' }}$
$
ntp source {{ ntp_source_interface }}$
```

```bash
# .gitattributes — บังคับ LF สำหรับไฟล์ Template/YAML ทุกไฟล์ในโปรเจกต์ ไม่ว่า Editor/OS ไหน commit
cat >> .gitattributes << 'EOF'
*.j2 text eol=lf
*.yaml text eol=lf
*.yml text eol=lf
EOF
git add .gitattributes templates/ntp_syslog.j2
git commit -m "Fix: normalize line endings to LF, add .gitattributes to prevent CRLF regressions"
```

```bash
$ ansible-playbook -i inventory.yaml site.yaml --check --diff --ask-vault-pass
```

```
TASK [ntp-config : [ntp-config] Push NTP + Syslog config via rendered template] ***
ok: [CORE-SW1]
ok: [CORE-SW2]
...

PLAY RECAP ***************************************************************************
CORE-SW1  : ok=1  changed=0  unreachable=0  failed=0  skipped=0
```

`changed=0` กลับมาแล้ว — Idempotency ได้รับการยืนยันอีกครั้งด้วยกระบวนการเดียวกับ Step 478

> **บทเรียนสำคัญ**: ปัญหา Idempotency ที่พบบ่อยที่สุดในทีมจริงไม่ใช่ Logic ผิด แต่คือ **Whitespace/
> Line Ending/ลำดับบรรทัดที่ไม่ตรงกัน** เพราะสิ่งเหล่านี้ "ดูเหมือนกัน" บนหน้าจอเสมอ `--check --diff`
> ที่ดูเหมือนกันเป๊ะระหว่าง before/after (ไม่ใช่ไม่มี diff เลย) คือสัญญาณเตือนที่ต้องสงสัยอักขระ
> ที่มองไม่เห็นทันที

---

## Step 664 — Scenario 3: TextFSM Parse ได้ค่าว่าง/ผิดเงียบๆ หลัง IOS Upgrade

### 664.1 อาการ

หลัง Maintenance Window อัปเกรด IOS-XE เป็น 17.12.02 (เหตุการณ์เดียวกับ Step 662 แต่คนละมุม)
`build_inventory.py` (Part 47 Step 464) ยังรันจนจบโดยไม่ Error เลย — แต่ CMDB ที่ได้กลับมีข้อมูล
ผิดปกติ

```csv
hostname,mgmt_ip,model,ios_version,serial_number,uptime,status,collected_at
CORE-SW1,10.10.99.10,unknown,unknown,unknown,3 weeks,reachable,2026-09-26T09:15:00+00:00
CORE-SW2,10.10.99.11,unknown,unknown,unknown,3 weeks,reachable,2026-09-26T09:15:03+00:00
```

**นี่คือ Category 2 (Parsing/Data) ที่อันตรายที่สุด** เพราะ Script รายงาน `status: reachable`
(ดูเหมือนสำเร็จ) — ไม่มี Exception ให้ Alert ทำงาน ไม่มีใครรู้ว่าข้อมูลผิดจนกว่าจะมีคนเปิด CMDB
ดูเอง (ในเคสจริงมักถูกพบช้าไปหลายวันเพราะ Dashboard ที่ดึงจากไฟล์นี้ไม่มี Sanity Check)

### 664.2 ตรวจสอบ Raw CLI Output ก่อน — เทียบกับที่ TextFSM คาดหวัง

```
CORE-SW1# show version
Cisco IOS XE Software, Version 17.12.02
Cisco IOS Software [Dublin], Catalyst L3 Switch Software (CAT9K_IOSXE), Version 17.12.2, RELEASE SOFTWARE (fc3)
...
Cisco Catalyst 9500-40X (X86) processor with 3963557K/6147K bytes of memory.
Processor board ID FXS2145Q0AB
...
```

เทียบกับ Output เดิมก่อน Upgrade (จาก Step 461):

```
Cisco IOS XE Software, Version 17.09.04a
...
cisco C9500-40X (X86) processor with 3963557K/6147K bytes of memory.
Processor board ID FXS2145Q0AB
```

**สังเกตความต่างที่ดูเล็กน้อยแต่ร้ายแรง**: บรรทัด Hardware เปลี่ยนจาก
`cisco C9500-40X (X86) processor ...` (ตัวพิมพ์เล็ก `cisco` + ไม่มีคำว่า "Catalyst") เป็น
`Cisco Catalyst 9500-40X (X86) processor ...` (ตัวพิมพ์ใหญ่ `Cisco` + เพิ่มคำว่า "Catalyst")
ใน IOS-XE 17.12 — Cisco เปลี่ยนคำบรรยาย Hardware String ใน Release ใหม่โดยไม่ถือว่าเป็น
Breaking Change (เพราะมันเป็นแค่ Text อธิบายสำหรับคนอ่าน) แต่สำหรับ TextFSM ที่ Parse ด้วย Regex
ตายตัว **นี่คือ Breaking Change เต็มรูปแบบ**

### 664.3 เปิด Template `ntc-templates` ที่ใช้จริงเพื่อดู Regex ที่คาดหวัง

```bash
$ python3 -c "import ntc_templates, os; print(os.path.dirname(ntc_templates.__file__))"
/home/netadmin/.venv/lib/python3.11/site-packages/ntc_templates/templates

$ grep -A3 "^Value HARDWARE" \
    /home/netadmin/.venv/lib/python3.11/site-packages/ntc_templates/templates/cisco_ios_show_version.textfsm
```

```
Value HARDWARE (\S+.*)
Value SERIAL (\S+)
Value CONFIG_REGISTER (\S+)

...
^cisco (${HARDWARE}) \(.*\) processor.* -> Continue
```

Regex เดิมคาดหวังบรรทัดที่ขึ้นต้นด้วย `cisco ` (ตัวพิมพ์เล็ก, มี space ต่อท้าย) ตรงตัว — เมื่อ
IOS-XE 17.12 เปลี่ยนเป็น `Cisco Catalyst 9500-40X ...` (ตัวพิมพ์ใหญ่ `Cisco` และมีคำว่า
`Catalyst` แทรกมา) **บรรทัดนี้ไม่ Match Regex เดิมอีกต่อไป** State Machine ของ TextFSM จึงไม่เข้า
State ที่ Capture ค่า `HARDWARE`/`SERIAL` เลย — ไม่ Error แต่คืนค่า Field เป็นค่า Default (String
ว่าง) ทำให้ `facts.get("hardware")` ใน `build_inventory.py` ได้ List ว่าง แล้ว fallback ไปเป็น
`"unknown"` ตาม Logic ที่เขียนไว้ (Step 464) — **นี่คือเหตุผลที่ Script ไม่ Error แม้ Parse พัง**

### 664.4 ทดสอบยืนยันด้วยการรัน TextFSM แยกจาก Netmiko โดยตรง

```python
#!/usr/bin/env python3
"""debug_textfsm.py — ทดสอบ Template กับ Raw Output แยกจาก Netmiko เพื่อยืนยัน Root Cause"""
import textfsm

with open("raw_show_version_17_12.txt") as raw_file:
    raw_output = raw_file.read()

with open("cisco_ios_show_version.textfsm") as template_file:
    fsm = textfsm.TextFSM(template_file)
    result = fsm.ParseText(raw_output)

print(f"จำนวน record ที่ parse ได้: {len(result)}")
print(f"Header: {fsm.header}")
print(f"ค่าที่ได้: {result}")
```

```
$ python3 debug_textfsm.py
จำนวน record ที่ parse ได้: 1
Header: ['VERSION', 'HARDWARE', 'SERIAL', 'CONFIG_REGISTER', 'UPTIME', ...]
ค่าที่ได้: [['17.12.2', '', '', '0x2102', '3 weeks', ...]]
```

ยืนยันชัดเจน: field `VERSION`/`UPTIME` ยัง Parse ได้ (Regex ของบรรทัดพวกนั้นไม่เปลี่ยน) แต่
`HARDWARE`/`SERIAL` เป็น String ว่างตรงตามที่คาดไว้

### 664.5 แก้ไข — สร้าง Local Template Override โดยไม่ต้องรอ Community อัปเดต Upstream

Cisco Community ที่ดูแล `ntc-templates` มักจะแก้ปัญหานี้ในไม่กี่สัปดาห์ แต่ทีม NetOps ต้องใช้งาน
วันนี้ — วิธีที่ถูกต้องคือสร้าง **Local Template Directory ของตัวเอง** ที่ TextFSM มองหาก่อน
Directory ของ `ntc-templates` เสมอ

```bash
mkdir -p custom_templates
cp $(python3 -c "import ntc_templates,os;print(os.path.dirname(ntc_templates.__file__))")/templates/cisco_ios_show_version.textfsm \
   custom_templates/
```

แก้ Regex ให้รองรับทั้งสองรูปแบบ (Backward-Compatible กับ Version เก่าด้วย):

```diff
- ^cisco (${HARDWARE}) \(.*\) processor.* -> Continue
+ ^[Cc]isco (Catalyst )?(${HARDWARE}) \(.*\) processor.* -> Continue
```

```bash
# custom_templates/index — ไฟล์บอก TextFSM ว่า Command นี้ใช้ Template ไฟล์ไหน
cat > custom_templates/index << 'EOF'
Template, Hostname, Platform, Command
cisco_ios_show_version.textfsm, .*, cisco_ios, sh[[ow]] ver[[sion]]
EOF

# ชี้ NET_TEXTFSM ไปที่ Directory ของเราก่อน (Priority สูงกว่า ntc-templates เดิม)
export NET_TEXTFSM=$(pwd)/custom_templates
python3 build_inventory.py
```

```csv
hostname,mgmt_ip,model,ios_version,serial_number,uptime,status,collected_at
CORE-SW1,10.10.99.10,9500-40X,17.12.2,FXS2145Q0AB,3 weeks,reachable,2026-09-26T09:40:00+00:00
```

> **มาตรการป้องกัน**: เพิ่ม **Sanity Check ใน Script เอง** — ถ้า `facts.get("hardware")` ว่าง หรือ
> `serial_number == "unknown"` ทั้งที่ `status == "reachable"` ให้ Log เป็น `WARNING` ระดับสูง
> แยกจาก "ปกติ" ทันที (ไม่ควรรอให้คนเปิด CMDB มาเจอเอง) — และเก็บ Raw CLI Output ดิบไว้เสมอ
> (ไม่ใช่แค่ Parsed Data) เพื่อให้ Debug กรณีนี้ทำได้เร็วโดยไม่ต้องรอ SSH เข้าไปเก็บใหม่

---

## Step 665 — Scenario 4: NETCONF `edit-config` ล้มเหลวด้วย YANG Validation Error

### 665.1 อาการ

Script ที่ใช้ NETCONF (Part 49 Step 483-484) เพื่อเพิ่ม Secondary IP Address ให้กับ
`GigabitEthernet0/0/3` บน `WAN-EDGE-1` (สำหรับรองรับ Subnet เดิมของ Partner ที่ต้อง Migrate)
ล้มเหลวตอน `<validate>` ด้วย Error ที่อ่านแล้วงงทันที

```xml
<rpc message-id="310" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target><candidate/></target>
    <default-operation>replace</default-operation>
    <config>
      <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
        <interface>
          <GigabitEthernet>
            <name>0/0/3</name>
            <ip>
              <address>
                <secondary>
                  <address>172.20.10.254</address>
                  <mask>255.255.255.0</mask>
                </secondary>
              </address>
            </ip>
          </GigabitEthernet>
        </interface>
      </native>
    </config>
  </edit-config>
</rpc>
```

Response ที่ได้กลับมา:

```xml
<rpc-reply message-id="310" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <rpc-error>
    <error-type>application</error-type>
    <error-tag>operation-failed</error-tag>
    <error-severity>error</error-severity>
    <error-path xmlns:ios="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
      /ios:native/ios:interface/ios:GigabitEthernet[ios:name='0/0/3']/ios:ip/ios:address
    </error-path>
    <error-message xml:lang="en">
      Need gateway to be configured before setting up static route
    </error-message>
  </rpc-error>
</rpc-reply>
```

### 665.2 อ่าน `error-path` และ `error-message` อย่างเป็นระบบ (แทนการเดา)

| ส่วนของ Error | ความหมาย |
|---|---|
| `error-type: application` | ปัญหาไม่ใช่ Syntax XML ผิด (จะเป็น `error-type: protocol`) — เป็นปัญหาระดับ**ความหมาย/Constraint ของข้อมูล** |
| `error-tag: operation-failed` | Operation เข้าใจ Syntax ได้ปกติ แต่ Apply แล้วขัดกับกฎของ YANG Model |
| `error-path` | บอกตำแหน่ง**เป๊ะ**ใน Data Tree ที่ทำให้เกิดปัญหา — ในเคสนี้คือ `.../GigabitEthernet[name='0/0/3']/ip/address` |
| `error-message` | ข้อความจาก IOS-XE เอง (มักสั้นและไม่ได้อธิบาย "ทำไม" แบบเข้าใจง่าย — ต้องไปดู YANG Model ต่อ) |

`error-message` ในเคสนี้ทำให้เข้าใจผิดได้ง่ายว่าเกี่ยวกับ Route/Gateway — แต่ `error-path` ชี้ตรง
มาที่ `ip/address` ไม่ใช่ Routing เลย เป็นสัญญาณว่าต้องดู **YANG `must` Statement ของ Container
`address`** โดยตรง ไม่ใช่เชื่อ Error Message ตรงตัว

### 665.3 ตรวจสอบ YANG Constraint จริงด้วย `pyang`/`yanglint`

```bash
$ pyang -f tree --tree-path /native/interface/GigabitEthernet/ip/address \
    Cisco-IOS-XE-native.yang
```

```
module: Cisco-IOS-XE-native
  +--rw native
     +--rw interface
        +--rw GigabitEthernet* [name]
           +--rw ip
              +--rw address
                 +--rw primary!
                 |  +--rw address?   inet:ipv4-address
                 |  +--rw mask?      yang:dotted-quad
                 +--rw secondary* [address mask]
                    +--rw address    inet:ipv4-address
                    +--rw mask       yang:dotted-quad
```

```bash
$ yanglint -f tree Cisco-IOS-XE-native.yang | grep -B2 -A2 "must"
```

```
      +--rw address
         +--rw primary!
         |    ...
         +--rw secondary* [address mask]
              ...
         # must "not(secondary) or primary" {
         #   error-message "A primary address must be configured
         #                   before any secondary address can exist";
         # }
```

**พบ Root Cause แล้ว**: YANG Model กำหนด `must` Constraint ไว้ชัดเจนว่า **ต้องมี `primary`
address อยู่แล้ว (หรืออยู่ใน `edit-config` เดียวกัน) ก่อนที่จะเพิ่ม `secondary` ได้เสมอ** — Script
ของเราส่งเฉพาะ `secondary` ไปตัวเดียวโดยตั้ง `<default-operation>replace</default-operation>`
ที่ระดับ `native` (ไม่ใช่ `merge`) ซึ่งไปทับ (Replace) ทั้ง Container `ip/address` ของ Interface
นั้นทิ้งไปหมด **รวม `primary` ที่มีอยู่แล้วในอุปกรณ์จริงด้วย** เหลือแค่ `secondary` ตัวเดียวในผลลัพธ์
ที่ Validate — ทำให้ชน `must` Constraint ทันที

### 665.4 แก้ไข — เปลี่ยน `default-operation` เป็น `merge` และ/หรือส่ง `primary` มาด้วย

```xml
<!-- แก้ที่ 1: เปลี่ยนจาก replace เป็น merge — ไม่ลบ primary ที่มีอยู่แล้วบนอุปกรณ์ -->
<edit-config>
  <target><candidate/></target>
  <default-operation>merge</default-operation>   <!-- <-- เปลี่ยนจุดนี้ -->
  <config>
    <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
      <interface>
        <GigabitEthernet>
          <name>0/0/3</name>
          <ip>
            <address>
              <secondary>
                <address>172.20.10.254</address>
                <mask>255.255.255.0</mask>
              </secondary>
            </address>
          </ip>
        </GigabitEthernet>
      </interface>
    </native>
  </config>
</edit-config>
```

```
<rpc-reply message-id="311" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>
```

`<validate>` ผ่านทันทีเพราะตอนนี้ `must` Constraint เห็นทั้ง `primary` (ที่ยังอยู่บนอุปกรณ์ ไม่ถูก
ลบเพราะ `merge` ไม่ทับของเดิม) และ `secondary` (ที่เพิ่งเพิ่ม) พร้อมกัน — ตรงตามกฎที่ YANG Model
กำหนดไว้พอดี

> **บทเรียนสำคัญ**: NETCONF `default-operation: replace` มีความหมายว่า "แทนที่ทั้ง Subtree ณ
> ระดับที่ระบุ" ไม่ใช่แค่ "แก้ Field ที่ส่งมา" — Script ที่ตั้ง `replace` ที่ระดับสูงเกินไป (เช่น
> ที่ `native` ทั้งก้อน) จะลบข้อมูลที่ไม่ได้ตั้งใจแก้ทิ้งไปโดยไม่รู้ตัว แล้วมาปะทะกับ `must`
> Constraint ที่ตรวจ Field หลายตัวพร้อมกันแบบนี้เสมอ — ค่า Default ที่ปลอดภัยกว่าสำหรับ Script
> ส่วนใหญ่คือ `merge` เว้นแต่ตั้งใจล้าง Config เก่าทิ้งจริงๆ (ทบทวน Two-Phase Commit เต็มรูปแบบใน
> Part 49 Step 484)

---

## Step 666 — Scenario 5: Ansible Vault Decrypt ไม่ผ่านบน CI/CD Runner

### 666.1 อาการ

ทีม NetOps ตั้ง CI/CD Pipeline (GitLab CI Runner) ให้รัน `day2-ops` Playbook (Part 48 Step 480)
อัตโนมัติทุกคืนตี 2 บน Branch `dev` — Engineer-C ทดสอบบนเครื่อง Local ด้วยคำสั่งเดียวกันก่อน Commit
แล้วรันผ่านสำเร็จ

```bash
# บนเครื่อง Local ของ Engineer-C — สำเร็จ
$ ansible-playbook -i inventory/dev.yaml site.yaml \
    --vault-id dev@~/.vault_pass/dev.txt
PLAY RECAP: DIST-SW1 : ok=6  changed=0  ...
```

แต่ Pipeline Job บน CI Runner คืนคืนนี้ Fail ด้วย Error:

```
$ ansible-playbook -i inventory/dev.yaml site.yaml \
    --vault-id dev@$CI_PROJECT_DIR/.ci_vault_pass/dev.txt
ERROR! Decryption failed (no vault secrets were found that could decrypt) on
group_vars/dev/vault.yml.
```

**Category 1 (Connectivity/Auth) แต่คนละแบบ** — ปัญหาไม่ใช่ SSH ไปอุปกรณ์เลย เป็น Auth ระดับ
Ansible เอง เกิดเฉพาะบน CI Runner ไม่เกิด Local — ตรงกับ "ล้มเหลวเฉพาะ Environment หนึ่ง" ตาม
Checklist Step 661.3 (Q1: ล้มเหลวทุกที่ หรือเฉพาะบางที่)

### 666.2 ตรวจสอบไฟล์ Vault Password บน CI Runner โดยตรง (ไม่เชื่อ Log อย่างเดียว)

```bash
# เพิ่ม Debug Step ชั่วคราวเข้า .gitlab-ci.yml เพื่อตรวจไฟล์ก่อนเรียก ansible-playbook
- echo "ขนาดไฟล์ vault password:"
- wc -c "$CI_PROJECT_DIR/.ci_vault_pass/dev.txt"
- echo "md5sum ของไฟล์ vault password:"
- md5sum "$CI_PROJECT_DIR/.ci_vault_pass/dev.txt"
```

```
ขนาดไฟล์ vault password:
0 .ci_vault_pass/dev.txt
md5sum ของไฟล์ vault password:
d41d8cd98f00b204e9800998ecf8427e  .ci_vault_pass/dev.txt
```

**ไฟล์มีขนาด 0 byte** — MD5 `d41d8cd98f00b204e9800998ecf8427e` คือค่า Hash ของไฟล์เปล่าพอดี
Password ไม่ได้ถูกเขียนเข้าไปเลย ทั้งที่ Step ก่อนหน้าใน Pipeline เขียนว่า "สำเร็จ" (Exit Code 0)

### 666.3 ตรวจสอบ `.gitlab-ci.yml` ว่าไฟล์นี้ถูกสร้างอย่างไร

```yaml
# .gitlab-ci.yml (ส่วนที่เกี่ยวข้อง)
nightly_dev_run:
  stage: deploy
  script:
    - mkdir -p .ci_vault_pass
    - echo "$VAULT_PASSWORD_DEV" > .ci_vault_pass/dev.txt
    - ansible-galaxy collection install -r requirements.yml
    - ansible-playbook -i inventory/dev.yaml site.yaml
        --vault-id dev@.ci_vault_pass/dev.txt
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
```

Script ดูถูกต้องทุกอย่าง — ปัญหาจึงต้องอยู่ที่ **ตัวแปร `$VAULT_PASSWORD_DEV` เอง** ไม่ใช่ที่ Script

### 666.4 ตรวจสอบการตั้งค่า CI/CD Variable ใน GitLab (จุดที่พบ Root Cause จริง)

```
Settings > CI/CD > Variables
┌────────────────────┬──────────┬───────────┬─────────────────────────┐
│ Key                 │ Protected│ Masked    │ Environment Scope       │
├────────────────────┼──────────┼───────────┼─────────────────────────┤
│ VAULT_PASSWORD_DEV  │   ✅ Yes  │   ✅ Yes  │ production   <-- ผิด!    │
│ VAULT_PASSWORD_PROD │   ✅ Yes  │   ✅ Yes  │ production               │
└────────────────────┴──────────┴───────────┴─────────────────────────┘
```

**พบสาเหตุแท้จริง**: ตัวแปร `VAULT_PASSWORD_DEV` ถูกตั้ง **Environment Scope เป็น `production`**
โดยผิดพลาดตอน Setup Pipeline ครั้งแรก (Copy-paste จาก `VAULT_PASSWORD_PROD` แล้วลืมแก้ Scope) —
GitLab CI จะ **ไม่ Inject ค่าตัวแปรเข้า Job เลยถ้า Environment Scope ไม่ตรงกับ Job ที่กำลังรัน**
Job `nightly_dev_run` รันใน Context ของ `dev` (ไม่ตรงกับ Scope `production` ที่ตั้งไว้) ดังนั้น
`$VAULT_PASSWORD_DEV` จึงเป็น**ค่าว่าง**เสมอในบริบทนี้ — `echo "" > dev.txt` จึงได้ไฟล์ 0 byte
พอดี ไม่ใช่ Error ทันที (เพราะ `echo`/redirect ไม่ตรวจสอบว่าตัวแปรว่างหรือไม่) แต่ไป Fail ที่ขั้น
`ansible-vault decrypt` ในภายหลังแทน ซึ่งทำให้ Debug ยากเพราะ Error Message ไม่ได้บอกตรงๆว่า
"Password ที่ใช้เป็นค่าว่าง"

### 666.5 แก้ไข — แก้ Environment Scope ให้ตรง + เพิ่ม Guard ตรวจไฟล์ก่อนใช้งานจริง

```
Settings > CI/CD > Variables
VAULT_PASSWORD_DEV : Environment Scope = dev      <-- แก้ให้ตรงกับ Job จริง
```

เพิ่ม Guard เข้า Pipeline เพื่อให้ Fail แบบชี้ชัดตั้งแต่ต้น แทนการรอไป Fail ที่ `ansible-vault`
ด้วย Error ที่อ่านไม่รู้ต้นเหตุ:

```yaml
nightly_dev_run:
  stage: deploy
  script:
    - mkdir -p .ci_vault_pass
    - echo "$VAULT_PASSWORD_DEV" > .ci_vault_pass/dev.txt
    # --- Guard: ตรวจก่อนใช้งานจริงว่าไฟล์ไม่ใช่ค่าว่าง ---
    - |
      if [ ! -s .ci_vault_pass/dev.txt ]; then
        echo "[FATAL] VAULT_PASSWORD_DEV เป็นค่าว่าง — ตรวจ Environment Scope ของ CI/CD Variable"
        exit 1
      fi
    - ansible-galaxy collection install -r requirements.yml
    - ansible-playbook -i inventory/dev.yaml site.yaml
        --vault-id dev@.ci_vault_pass/dev.txt
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
```

```bash
$ ansible-playbook -i inventory/dev.yaml site.yaml --vault-id dev@.ci_vault_pass/dev.txt
PLAY RECAP: DIST-SW1 : ok=6  changed=0  unreachable=0  failed=0  skipped=0
```

> **บทเรียนสำคัญ**: Vault-ID/Multi-Environment (Part 48 Step 477) แก้ปัญหา "ใครรู้ Password
> Production ได้อ่าน Dev ด้วย" แต่มันเปิดช่องให้เกิด **ความผิดพลาดในการ Map ตัวแปรให้ตรง
> Environment ในระบบ CI/CD** ซึ่งเป็นปัญหาคนละชั้นแต่ Error Message ที่ Ansible แสดง
> ("no vault secrets were found") เหมือนกันทั้งสองกรณี — จึงต้อง**ตรวจไฟล์ Password ตรงๆก่อน
> เสมอ** อย่าเชื่อ Error Message ของ Ansible Vault อย่างเดียวว่าปัญหาคือ "รหัสผิด" เพราะบางครั้ง
> ปัญหาคือ "ไม่มีรหัสอยู่ในไฟล์เลย"

---

## Step 667 — Scenario 6: Compliance Check รายงาน False-Positive Drift

### 667.1 อาการ

`compliance_check.py` (Part 47 Step 465) ที่รันทุกเช้าผ่าน Cron รายงาน **`DIST-SW3` มี Violation
ของ Rule `SYSLOG-02`** ต่อเนื่องมาแล้ว 11 วัน

```
DIST-SW3: ❌ 1 VIOLATION(S)
    [✓] NTP-01     (high    ) - ต้องมี NTP server หลัก 10.10.99.50 พร้อม authentication key
    [✗] SYSLOG-02  (medium  ) - Logging trap level ต้องเป็น informational หรือ severe กว่า
    [✓] AAA-01     (critical) - ต้องเปิดใช้งาน aaa new-model
```

วิศวกรที่ดู Report ทุกวันเริ่มมองว่า "DIST-SW3 มันเป็นแบบนี้อยู่แล้ว ไม่ต้องสนใจ" (Alert Fatigue)
— จนกระทั่งวันที่ 12 Compliance Check เริ่มรายงาน Violation ใหม่ที่ `AAA-02` (Critical) บน
`DIST-SW3` ด้วย แต่ **ไม่มีใครสังเกตเพราะทุกคนมองข้าม Report ของอุปกรณ์นี้ไปแล้วจากความเคยชิน**
กว่าจะพบว่า AAA Login Fallback ถูกลบออกจริง (จาก Config เดิมมือใครบางคนที่ Rollback ไม่สมบูรณ์)
ก็ผ่านไปแล้ว 4 วัน

### 667.2 ตรวจสอบว่า Violation ของ `SYSLOG-02` เป็น Drift จริงหรือ False-Positive

```
DIST-SW3# show running-config | include logging trap
logging trap notifications
```

Config จริงคือ `logging trap notifications` — ตรวจตาม `GOLDEN_RULES` (Step 465):

```python
{
    "id": "SYSLOG-02",
    "pattern": re.compile(
        r"^logging trap (emergencies|alerts|critical|errors|warnings|notifications|informational)",
        re.MULTILINE,
    ),
},
```

Regex นี้ **`notifications` อยู่ในกลุ่มที่ Match อยู่แล้ว** — แปลว่า Rule ควรจะผ่าน แล้วทำไม Report
ถึงบอกว่า Violation? ตรวจ Golden Config Rule ตัวเดิมอีกครั้งให้ละเอียด (เช็ค Git History)

```bash
$ git log -p golden_config.py | grep -B5 "SYSLOG-02" | head -20
```

```
commit a91f2e4 (5 เดือนก่อน)
    "pattern": re.compile(
-       r"^logging trap (informational|debugging)$",
+       r"^logging trap (emergencies|alerts|critical|errors|warnings|notifications|informational)",
        re.MULTILINE,
    ),
```

Regex ถูกแก้ไขให้กว้างขึ้นแล้ว **5 เดือนที่แล้ว** — Rule ปัจจุบันจึงถูกต้องแล้วจริงๆ ไม่มี
False-Positive ที่ Regex — ต้องหาสาเหตุอื่น

### 667.3 ตรวจสอบว่า Regex Match แต่ทำไม `compliant` ยังเป็น `False`

```
DIST-SW3# show running-config | include logging trap
logging trap notifications level 5
```

**พบจริง**: Config บนอุปกรณ์จริงมี Argument เพิ่มต่อท้าย (`level 5`) ที่ Regex `^logging trap
(...)` จับได้แค่คำแรกหลัง `logging trap ` (`notifications`) เพราะ Regex ไม่ได้ใส่ `$` ปิดท้าย —
ที่จริง Regex **ควร Match ได้** (ไม่ต้อง Match ทั้งบรรทัด) เพราะ `re.search` ไม่ใช่ `re.fullmatch`
— ทดสอบตรงๆ:

```python
>>> import re
>>> pattern = re.compile(r"^logging trap (emergencies|alerts|critical|errors|warnings|notifications|informational)", re.MULTILINE)
>>> bool(pattern.search("logging trap notifications level 5"))
True
```

**Match ได้จริง** — แสดงว่าปัญหาไม่ได้อยู่ที่ Regex เลยด้วยซ้ำ ต้องดูที่ `running_config` ตัวแปร
ที่ Script ดึงมาจริงว่าตรงกับที่เห็นบน CLI หรือไม่

### 667.4 พบ Root Cause แท้จริง: Golden Config ไม่ได้ Sync กับ Terminal Width Wrap

```python
config = connection.send_command("show running-config")
```

```
DIST-SW3# show running-config
...
logging trap notifica
tions level 5
...
```

**Root Cause แท้จริง**: อุปกรณ์ `DIST-SW3` ตัวเดียวใน Lab ที่ยังตั้งค่า `terminal width` เป็นค่า
Default เก่า (`terminal width 80` จากการ Provisioning ครั้งแรกที่ไม่ผ่าน Automation) ทำให้
Output ของ `show running-config` **ตัดขึ้นบรรทัดใหม่กลางคำ** (`notifica` / `tions level 5`
คนละบรรทัด) เพราะไม่มีการปิด Terminal Pagination/Width ก่อนรัน Command ผ่าน Netmiko — บรรทัด
`notifica` (ไม่ครบคำ) จึงไม่ Match Regex ที่คาดหวังคำเต็ม `notifications`

### 667.5 แก้ไข — ปิด Terminal Width ให้ถูกต้องทุกอุปกรณ์ (จุดที่ Golden Config ควบคุมไม่ถึง)

```python
# แก้ collect config ให้ปิด wrap ก่อนดึง running-config เสมอ (ไม่พึ่งพา Default ของอุปกรณ์)
connection = ConnectHandler(**conn_params)
connection.enable()
connection.send_command("terminal width 511")   # กว้างพอไม่ให้ IOS wrap บรรทัดเอง
config = connection.send_command("show running-config")
```

```bash
$ python3 compliance_check.py
```

```
DIST-SW3: ✅ COMPLIANT
```

### 667.6 การจัดการ False-Positive อย่างเป็นระบบ — Waiver File + Scripted Diff-Review

ปัญหาจริงของ Scenario นี้ไม่ใช่แค่ Bug เดียว (`terminal width`) แต่คือ **Operational Discipline**:
ทีมปล่อยให้ Compliance Report มี Noise สะสมจนเกิด Alert Fatigue นานเกินไป — วิธีป้องกันเชิงระบบ
คือแยก **"Known/Waived Exception"** ออกจาก **"New/Unwaived Violation"** อย่างชัดเจน แทนการดู
Report ดิบทุกวัน

```yaml
# exceptions.yaml — Known Exception ที่ผ่านการอนุมัติ พร้อม Expiry (บังคับ Renew เป็นระยะ)
- device: DIST-SW3
  rule_id: "SYSLOG-02"
  reason: "Terminal width legacy — รอ Field Change Window เพื่อ Standardize (CHG0004790)"
  approved_by: "netops-lead@company.local"
  expires: "2026-10-15"
```

```python
#!/usr/bin/env python3
"""
diff_review.py
เทียบ Compliance Report วันนี้กับ exceptions.yaml — แจ้งเตือนเฉพาะ Violation ที่ "ใหม่" หรือ
"Waiver หมดอายุแล้ว" เท่านั้น ไม่ทำให้ทีม Alert Fatigue จาก Known Issue ที่รอแก้อยู่แล้ว
"""
import sys
import yaml
from datetime import date

from compliance_check import DEVICES, fetch_running_config, check_compliance


def load_exceptions(path="exceptions.yaml"):
    with open(path) as f:
        return yaml.safe_load(f) or []


def main():
    exceptions = load_exceptions()
    today = date.today()
    new_violations = []

    for device in DEVICES:
        config = fetch_running_config(device)
        results = check_compliance(config)

        for r in results:
            if r["compliant"]:
                continue

            waiver = next(
                (e for e in exceptions
                 if e["device"] == device["name"] and e["rule_id"] == r["id"]),
                None,
            )

            if waiver is None:
                new_violations.append((device["name"], r["id"], "ไม่มี Waiver — Violation ใหม่"))
            elif date.fromisoformat(waiver["expires"]) < today:
                new_violations.append((device["name"], r["id"], f"Waiver หมดอายุตั้งแต่ {waiver['expires']}"))

    if new_violations:
        print("[!] พบ Violation ที่ต้องสนใจจริง (ใหม่ หรือ Waiver หมดอายุ):")
        for hostname, rule_id, reason in new_violations:
            print(f"    - {hostname} / {rule_id}: {reason}")
        sys.exit(1)

    print("[*] ไม่มี Violation ใหม่ — Known Exception ทั้งหมดยังอยู่ใน Waiver ที่ยังไม่หมดอายุ")


if __name__ == "__main__":
    main()
```

```
$ python3 diff_review.py
[!] พบ Violation ที่ต้องสนใจจริง (ใหม่ หรือ Waiver หมดอายุ):
    - DIST-SW3 / AAA-02: ไม่มี Waiver — Violation ใหม่
```

ด้วยวิธีนี้ Violation ของ `AAA-02` (ปัญหาจริง — AAA Fallback ถูกลบ) จะถูกเห็น**ทันทีในวันแรก**
แยกออกจาก Noise ของ `SYSLOG-02` ที่ Waive ไว้แล้วอย่างมีเงื่อนไข/มีวันหมดอายุชัดเจน — บังคับให้ทีม
ต้อง Renew Waiver เป็นระยะ (ไม่ปล่อยให้ Exception ค้างตลอดกาลจนกลายเป็น "ปกติถาวร")

---

## Step 668 — Scenario 7: RESTCONF คืนค่า 401 Unauthorized เป็นพักๆ

### 668.1 อาการ

Script Python ที่ Poll ข้อมูล Interface ผ่าน RESTCONF (Part 49 Step 485) จาก `CORE-SW1` ทุก 5
นาทีแบบ Long-Running Process (รันต่อเนื่องหลายชั่วโมงไม่หยุด) เริ่มได้ผลลัพธ์ `401 Unauthorized`
เป็นพักๆ หลังรันไปแล้วประมาณ 2 ชั่วโมง — Username/Password เดิมที่ Hardcode ไว้ถูกต้องแน่นอน
(ทดสอบ `curl` ตรงๆ ด้วย Credential เดียวกันตอนนั้นก็ผ่านได้ปกติ)

```python
import requests
from requests.auth import HTTPBasicAuth

def poll_interfaces():
    response = requests.get(
        "https://10.10.99.10/restconf/data/ietf-interfaces:interfaces",
        auth=HTTPBasicAuth("admin", "StrongP@ssw0rd!"),
        headers={"Accept": "application/yang-data+json"},
        verify=False,
    )
    return response.json()

while True:
    data = poll_interfaces()
    ...
    time.sleep(300)
```

```
[09:00:00] Poll สำเร็จ (200 OK)
[09:05:00] Poll สำเร็จ (200 OK)
...
[11:05:00] Poll ล้มเหลว: 401 Unauthorized
[11:10:00] Poll สำเร็จ (200 OK)
[11:15:00] Poll ล้มเหลว: 401 Unauthorized
[11:20:00] Poll ล้มเหลว: 401 Unauthorized
```

### 668.2 วิเคราะห์: Username/Password ถูกต้อง แต่บางครั้งถูก 401 — สงสัย Session/AAA Limit

เพราะ Credential ถูกต้องแน่นอน (Test ตรงด้วย `curl` ผ่าน) แต่ 401 เกิดเฉพาะบางรอบ **ไม่ใช่ปัญหา
Credential ผิด** — ต้องดูที่ Layer ของ AAA/Session ตาม Checklist Category 1 ย่อยที่เจาะจงกว่า
(Session Exhaustion) แทน "รหัสผิด" ตรงตัว

```
CORE-SW1# show users
    Line       User       Host(s)              Idle       Location
   0 con 0                idle                 00:00:12
*  2 vty 0     admin      idle                 00:00:03   10.10.99.20
   3 vty 1     admin      idle                 00:01:47   10.10.99.20
   4 vty 2     admin      idle                 00:03:02   10.10.99.20
   5 vty 3     admin      idle                 00:04:15   10.10.99.20
```

พบ Session `admin` จาก IP เดียวกัน (`10.10.99.20` — เครื่องที่รัน Script) **ค้างอยู่ 4 Session
พร้อมกัน** ทั้งที่ Script ควรจะเปิด-ปิด Connection ทีละรอบ Poll เท่านั้น (ไม่ควรมี Session ค้าง
เลยระหว่างรอ `time.sleep(300)`)

### 668.3 ตรวจสอบ AAA Server (TACACS+) และ Configuration ฝั่ง Login

```
CORE-SW1# show running-config | include session-limit
aaa authorization exec default group AAA-TACACS local
line vty 0 15
 session-limit 4
```

`session-limit 4` (ตั้งไว้ตาม Hardening Baseline ของ Part 22 Step 214 เพื่อจำกัดจำนวน Concurrent
Session ต่อ Group/Line เพื่อความปลอดภัย) — เมื่อ Script สร้าง Connection ใหม่ทุกรอบโดยไม่ปิด
Connection เดิมให้เรียบร้อย จะสะสม Session ค้างจนถึง Limit ที่ 4 พอดี **รอบ Poll ที่ 5 เป็นต้นไป
IOS-XE ปฏิเสธ Authentication ใหม่ทันที** (เพราะ Session-Limit เต็ม) ซึ่ง RESTCONF บน IOS-XE
มักคืนค่าเป็น `401 Unauthorized` แทนข้อความที่ชัดเจนกว่าอย่าง "Session limit exceeded" ทำให้
วิศวกรเข้าใจผิดว่าเป็นปัญหา Credential

### 668.4 ตรวจสอบ Root Cause ที่ตัว Script — ไม่ปิด Connection/ไม่ใช้ Session Reuse

```python
def poll_interfaces():
    response = requests.get(...)   # <-- สร้าง TCP Connection ใหม่ทุกครั้งที่เรียกฟังก์ชันนี้
    return response.json()
    # ไม่มี response.close() หรือการ reuse connection เลย
```

`requests.get()` แบบเดี่ยวๆ (ไม่ผ่าน `requests.Session()`) เปิด TCP/TLS Connection ใหม่ทุกครั้ง —
แม้ HTTP Connection จะถูกปิดที่ระดับ TCP ตามปกติของ `requests`, **RESTCONF บน IOS-XE ผูก
Authentication เข้ากับ AAA Session ที่นับที่ Line VTY** ซึ่งบางครั้งไม่ได้ถูกเคลียร์ทันทีที่ TCP
ปิด (ต้องรอ Idle Timeout) — เมื่อ Poll ทุก 5 นาทีต่อเนื่องนานพอ Session ค้างที่ยังไม่ Timeout
จะสะสมชนกับ Session ใหม่ที่เพิ่มเข้ามาเรื่อยๆ จนถึง `session-limit 4`

### 668.5 แก้ไข — ใช้ `requests.Session()` (Connection Reuse) + Retry/Backoff + ปรับ Timeout

```python
#!/usr/bin/env python3
"""
poll_interfaces_fixed.py
แก้ปัญหา Session สะสมด้วยการใช้ requests.Session() (TCP Connection เดียว reuse ตลอด)
แทนการเปิด Connection ใหม่ทุกรอบ Poll — ลดจำนวน AAA Session ที่ค้างบนอุปกรณ์ให้เหลือ 1 เสมอ
"""
import time
import requests
from requests.auth import HTTPBasicAuth

session = requests.Session()          # <-- สร้าง Session เดียว ใช้ตลอดอายุของ Script
session.auth = HTTPBasicAuth("admin", "StrongP@ssw0rd!")
session.headers.update({"Accept": "application/yang-data+json"})
session.verify = False


def poll_interfaces():
    response = session.get(
        "https://10.10.99.10/restconf/data/ietf-interfaces:interfaces",
        timeout=10,
    )
    response.raise_for_status()
    return response.json()


def main():
    while True:
        try:
            data = poll_interfaces()
            print(f"[{time.strftime('%H:%M:%S')}] Poll สำเร็จ (200 OK)")
        except requests.exceptions.HTTPError as e:
            print(f"[{time.strftime('%H:%M:%S')}] Poll ล้มเหลว: {e}")
        except requests.exceptions.RequestException as e:
            print(f"[{time.strftime('%H:%M:%S')}] Connection error: {e}")
        finally:
            time.sleep(300)


if __name__ == "__main__":
    main()
```

```
CORE-SW1# show users
    Line       User       Host(s)              Idle       Location
*  2 vty 0     admin      idle                 00:00:02   10.10.99.20
```

หลังแก้ไข เหลือ Session ค้างเพียง 1 ตัวเสมอ (Reuse Connection เดิม) ไม่ชน `session-limit 4`
อีกต่อไป — เพิ่มมาตรการป้องกันอีกชั้นด้วยการเพิ่ม `line vty` Timeout ให้สั้นลงเพื่อเคลียร์
Session ค้างเร็วขึ้นในกรณีที่มี Client อื่นทำผิดพลาดแบบเดียวกันในอนาคต:

```
CORE-SW1(config)# line vty 0 15
CORE-SW1(config-line)# exec-timeout 5 0
```

> **จุดเชื่อมกับ Part อื่น**: นี่คือตัวอย่างที่ชัดเจนว่าปัญหา Automation (Category 4: API/Version/
> Scale) มักซ่อนอยู่ที่**จุดตัดระหว่าง Automation กับ AAA** (Part 22/43) — วิศวกร Automation
> ที่ไม่รู้จัก `session-limit`/`aaa authorization exec` จะเสียเวลาสงสัย Credential นานเกินจำเป็น
> ในขณะที่วิศวกร Network ที่ไม่รู้ว่า Script ไม่ปิด Connection ก็จะสงสัยว่า AAA Server ผิดปกติ —
> ต้องอ่าน `show users`/AAA Config ควบคู่กับ Code ของ Script เสมอ

---

## Step 669 — Scenario 8: Nornir/ThreadPoolExecutor ทำ CPU อุปกรณ์รุ่นเก่าพุ่ง

### 669.1 อาการ

ทีม NetOps ขยาย Inventory ของ `run_show_version.py` (Part 47 Step 462) ให้ครอบคลุม
`ACCESS-SW1-4` (Catalyst 2960, ตาม `docs/00-ip-address-plan.md`) เพิ่มจากเดิมที่มีแค่
CORE-SW1/2 และ DIST-SW1-4 — ใช้ `num_workers: 10` เดิมที่เคยทำงานได้ดี

```yaml
# config.yaml (เดิม ไม่เปลี่ยน)
runner:
  plugin: threaded
  options:
    num_workers: 10
```

ผลลัพธ์: CORE/DIST ทำงานปกติเหมือนเดิม แต่ `ACCESS-SW1-4` เริ่มมีปัญหา — SSH Session หลุดกลาง
คำสั่ง, Ping Monitoring แจ้ง Latency สูงขึ้นช่วงที่ Job รัน, และบางครั้งได้ Error แปลกๆกลับมา

```
netmiko_send_command************************************************************
* ACCESS-SW2 ** changed : False ** failed : True *******************************
vvvv netmiko_send_command ** changed : False ** failed : True vvvvvvvvvvvvvvvvv ERROR
% Invalid input detected at '^' marker.
^^^^ END netmiko_send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

### 669.2 ตอบคำถาม Checklist: "อุปกรณ์ที่พังมีอะไรร่วมกัน?"

ทั้ง 4 ตัวที่มีปัญหาคือ `ACCESS-SW1-4` (Catalyst 2960) เท่านั้น — CORE-SW1/2 (Catalyst 9500) และ
DIST-SW1-4 (Catalyst 9300) ไม่มีปัญหาแม้รันพร้อมกันในกลุ่ม Thread เดียวกันเป๊ะ — สัญญาณชัดเจนว่า
เป็นเรื่อง **Platform/Resource ไม่ใช่ Script Logic** (Category 4)

### 669.3 ตรวจสอบ CPU ของ `ACCESS-SW2` ระหว่างที่ Job กำลังรัน

```
ACCESS-SW2# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 97%/2%; one minute: 68%; five seconds: 97%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
 142      812234       98211       8271  61.23%  38.10%  22.40%   0 SSH Process
 118      120044        8821      13608  19.02%  11.30%   6.10%   0 IP Input
```

`ACCESS-SW2` เป็น Catalyst 2960 (Fixed-Config, CPU รุ่นเก่า, ไม่มี Multi-Core แบบ Catalyst 9300/
9500) — เมื่อ Nornir เปิด SSH Connection **พร้อมกันหลาย Thread ไปยัง Platform รุ่นเก่าตัวเดียวกัน
ในเวลาไล่เลี่ยกัน** (เพราะ `num_workers: 10` ไม่ได้แยกตาม Platform เลย ถ้า Job เดียวมีถึง `ACCESS-
SW1-4` พร้อมกันภายใน Batch เดียวกัน) CPU ของ SSH Process บนอุปกรณ์รุ่นเก่าพุ่งสูงจนกระทบ Process
อื่น รวมถึงการ Render คำสั่งกลับผ่าน VTY (ทำให้ Netmiko อ่าน Output ผิดจังหวะ ได้ Prompt/Echo
ปนกัน กลายเป็น `% Invalid input`)

### 669.4 แก้ไข — แยก Concurrency ตาม Platform ด้วย Nornir Filter + Runner คนละชุด

```python
#!/usr/bin/env python3
"""
run_show_version_tiered.py
แก้ปัญหา CPU Spike บน Platform รุ่นเก่า ด้วยการแยก Concurrency เป็น 2 ระดับ:
- Core/Distribution (Catalyst 9500/9300): num_workers สูง ตามเดิม
- Access (Catalyst 2960): num_workers ต่ำมาก + หน่วงเวลาระหว่างคำสั่ง
"""
import time
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir.core.task import Task, Result


def send_command_with_delay(task: Task, command_string: str, delay: float = 0.0) -> Result:
    """Wrapper รอบ netmiko_send_command — เพิ่ม delay เล็กน้อยก่อนส่งคำสั่งจริง
    สำหรับอุปกรณ์ CPU ต่ำ ลด Burst ที่กระทบ SSH Process พร้อมกันหลาย Session"""
    if delay:
        time.sleep(delay)
    return task.run(task=netmiko_send_command, command_string=command_string)


def main():
    nr = InitNornir(config_file="config.yaml")

    # --- กลุ่ม High-Power: Core/Distribution — เดิมทำงานได้ดีอยู่แล้ว ไม่ต้องแก้ ---
    high_power = nr.filter(role="core") | nr.filter(role="distribution")
    print(f"[*] รัน High-Power Group ({len(high_power.inventory.hosts)} อุปกรณ์) "
          f"ด้วย num_workers เดิม (10) ...")
    result_high = high_power.run(task=netmiko_send_command, command_string="show version")

    # --- กลุ่ม Low-Power: Access Switch (Catalyst 2960) — จำกัด Concurrency ให้ต่ำมาก ---
    low_power = nr.filter(role="access")
    low_power.config.runner.options["num_workers"] = 2   # ลดจาก 10 -> 2 เฉพาะกลุ่มนี้
    print(f"[*] รัน Low-Power Group ({len(low_power.inventory.hosts)} อุปกรณ์) "
          f"ด้วย num_workers=2 + delay 1.5s ...")
    result_low = low_power.run(
        task=send_command_with_delay,
        command_string="show version",
        delay=1.5,
    )

    failed_low = [name for name, r in result_low.items() if r.failed]
    print(f"\n[*] High-Power: สำเร็จ {len(result_high) - len([r for r in result_high.values() if r.failed])}/{len(result_high)}")
    print(f"[*] Low-Power: สำเร็จ {len(result_low) - len(failed_low)}/{len(result_low)}"
          + (f" (ล้มเหลว: {failed_low})" if failed_low else ""))


if __name__ == "__main__":
    main()
```

```
inventory/groups.yaml (เพิ่ม group access_switches)
---
access_switches:
  data:
    role: access
```

```bash
$ python3 run_show_version_tiered.py
[*] รัน High-Power Group (6 อุปกรณ์) ด้วย num_workers เดิม (10) ...
[*] รัน Low-Power Group (4 อุปกรณ์) ด้วย num_workers=2 + delay 1.5s ...

[*] High-Power: สำเร็จ 6/6
[*] Low-Power: สำเร็จ 4/4
```

```
ACCESS-SW2# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 14%/1%; one minute: 9%; five seconds: 14%
```

CPU ของ `ACCESS-SW2` กลับสู่ระดับปกติ (14% เทียบกับ 97% เดิม) — เวลาที่เพิ่มขึ้นของกลุ่ม Access
(จำกัด Concurrency + Delay) น้อยกว่าความเสี่ยงที่จะทำให้อุปกรณ์เหล่านั้น Drop Session/CPU พุ่งจน
กระทบ Traffic จริงของ VLAN 30/40 ที่ผ่านอุปกรณ์เหล่านี้

> **หลักการเลือก `num_workers`**: ไม่มีค่าตายตัวที่ใช้ได้กับทุกอุปกรณ์ — ต้องพิจารณาทั้ง (1) จำนวน
> VTY Line ที่เหลือของอุปกรณ์ (Step 467), (2) กำลัง CPU ของ Platform (Fixed-Config รุ่นเก่า vs
> Modular/Stackable รุ่นใหม่), และ (3) ความสำคัญของ Traffic ที่วิ่งผ่านอุปกรณ์ตัวนั้นระหว่าง Job
> รัน — Inventory ที่มี Platform หลากหลาย (ตามจริงของ Enterprise) ควร**แบ่ง Runner Config ตาม
> Group เสมอ** ไม่ใช่ใช้ `num_workers` เดียวทั้ง Inventory

---

## Step 670 — Lab เต็มรูปแบบ: วินิจฉัย "Automation Pipeline พังทั้งคืน"

### 670.1 สถานการณ์เริ่มต้น

เช้าวันจันทร์ ทีม NetOps เปิด Dashboard พบว่า **Nightly Automation Run ของคืนวันอาทิตย์ (ตี 2)
ล้มเหลวทั้งสอง Pipeline พร้อมกัน**:

```
Pipeline A: "day2-ops" Ansible Playbook (Vault-based) via CI/CD Runner   -> FAILED
Pipeline B: "nightly_inventory" Nornir/Netmiko Python Script (Cron)     -> PARTIAL FAILURE (4/6 อุปกรณ์)
```

Chat ทีมเริ่ม Panic — มีคนเสนอให้ Rollback ทุกอย่างกลับ Commit เมื่อสัปดาห์ก่อน แต่หัวหน้าทีมสั่งให้
วินิจฉัยตาม Checklist ก่อน (Step 661) เพราะ **สอง Pipeline ล้มเหลวพร้อมกันไม่ได้แปลว่าสาเหตุ
เดียวกัน**

### 670.2 ขั้นที่ 1 — แยก Pipeline ก่อนวินิจฉัย อย่ามองเป็นปัญหาเดียว

| คำถาม | Pipeline A (Ansible) | Pipeline B (Nornir/Netmiko) |
|---|---|---|
| ล้มเหลวทุกอุปกรณ์ หรือบางส่วน? | ทุกอุปกรณ์ (Playbook หยุดตั้งแต่ก่อนต่ออุปกรณ์เลย) | บางส่วน (4/6 — เฉพาะบางตัว) |
| เพิ่งพังตอนนี้ หรือพังมานาน? | เพิ่งพังคืนนี้เป็นครั้งแรก | เพิ่งพังคืนนี้เป็นครั้งแรก |
| Error เกิดที่ Connection หรือหลัง Connect สำเร็จ? | ไม่ถึงขั้น Connect เลย (Fail ที่ขั้นเตรียม Vault) | เกิดที่ Connection (SSH) |

จาก Checklist Step 661.3: Pipeline A "ล้มเหลวทุกอุปกรณ์" + "Fail ก่อน Connect" → ชี้ไปที่ปัญหา
ระดับ **ก่อน Automation แม้แต่จะเริ่มทำงาน** (Credential/Environment) — Pipeline B "ล้มเหลว
เฉพาะบางตัว" + "Fail ที่ Connection" → ชี้ไปที่ปัญหาเฉพาะกลุ่มอุปกรณ์ (Category 1 เหมือนกันแต่
คนละอาการย่อย) — **สอง Pipeline นี้น่าจะเป็นสองปัญหาที่แยกกันจริง ไม่ใช่ปัญหาเดียวลุกลาม**

### 670.3 ขั้นที่ 2 — วินิจฉัย Pipeline A (Ansible Vault)

```bash
$ # ดึง Log เต็มจาก CI/CD Runner
$ cat ci_job_log_pipeline_a.txt
```

```
$ echo "$VAULT_PASSWORD_DEV" > .ci_vault_pass/dev.txt
$ ansible-playbook -i inventory/dev.yaml site.yaml --vault-id dev@.ci_vault_pass/dev.txt
ERROR! Decryption failed (no vault secrets were found that could decrypt) on
group_vars/dev/vault.yml.
```

เหมือน Symptom ใน Step 666 เป๊ะ — ตรวจไฟล์ Password ตามขั้นตอนเดิม:

```bash
$ wc -c .ci_vault_pass/dev.txt
0 .ci_vault_pass/dev.txt
```

```
Settings > CI/CD > Variables
VAULT_PASSWORD_DEV   Environment Scope = production   <-- ยังไม่ได้แก้จาก Incident ก่อนหน้า!
```

**Root Cause Pipeline A**: การแก้ไขจาก Step 666 (แก้ Environment Scope เป็น `dev`) **ถูก Revert
กลับโดยไม่ตั้งใจ** — ตรวจ Git History ของการตั้งค่า Infra-as-Code (CI/CD Variable จัดการผ่าน
Terraform ในกรณีนี้):

```bash
$ git log --oneline -- terraform/gitlab_ci_variables.tf
c3d8f21 Revert "Fix VAULT_PASSWORD_DEV environment scope" (สัปดาห์ก่อน)
8f2a1c3 Fix VAULT_PASSWORD_DEV environment scope (Step 666)
```

Commit `c3d8f21` เป็นการ Revert ที่เกิดจากวิศวกรอีกคน Merge Branch เก่าที่ยังไม่มี Fix กลับเข้า
`main` โดยไม่ตรวจสอบ Conflict อย่างละเอียด (Merge ทับค่าที่ถูกต้องด้วยค่าเก่าที่ผิด) — เป็น
ปัญหาด้าน Process ไม่ใช่ Technical ล้วนๆ

**แก้ไข Pipeline A**:

```bash
$ git revert c3d8f21 -m "Re-apply: fix VAULT_PASSWORD_DEV environment scope (regression from merge)"
$ terraform apply
```

```
Settings > CI/CD > Variables
VAULT_PASSWORD_DEV   Environment Scope = dev   <-- แก้กลับให้ถูกต้องแล้ว
```

```bash
$ ansible-playbook -i inventory/dev.yaml site.yaml --vault-id dev@.ci_vault_pass/dev.txt
PLAY RECAP: DIST-SW1 : ok=6  changed=0  unreachable=0  failed=0  skipped=0
```

Pipeline A กลับมาทำงานปกติ — เพิ่ม Guard จาก Step 666.5 เข้าไปด้วยเพื่อป้องกัน Regression แบบนี้
ในอนาคต (ถ้าไฟล์ Password ว่าง Job จะ Fail ทันทีตั้งแต่ต้น พร้อมข้อความชี้ชัด ไม่ต้องมารอ Vault
Error ที่คลุมเครือ)

### 670.4 ขั้นที่ 3 — วินิจฉัย Pipeline B (Nornir/Netmiko — เฉพาะ 4 อุปกรณ์)

```bash
$ cat nightly_inventory_log.txt
```

```
[02:00:04] CORE-SW1: สำเร็จ
[02:00:06] CORE-SW2: สำเร็จ
[02:00:08] DIST-SW1: ล้มเหลว — SSHException: Incompatible ssh peer (no acceptable kex algorithm)
[02:00:10] DIST-SW2: ล้มเหลว — SSHException: Incompatible ssh peer (no acceptable kex algorithm)
[02:00:12] DIST-SW3: ล้มเหลว — SSHException: Incompatible ssh peer (no acceptable kex algorithm)
[02:00:14] DIST-SW4: ล้มเหลว — SSHException: Incompatible ssh peer (no acceptable kex algorithm)
```

**อาการตรงกับ Step 662 เป๊ะ** แต่ครั้งนี้เกิดเฉพาะ `DIST-SW1-4` ไม่ใช่ทุกอุปกรณ์ — เพราะ
Maintenance Window ล่าสุด (ตรวจ Change Calendar) **อัปเกรดเฉพาะ `DIST-SW1-4`** (Phased Rollout —
อัปเกรด CORE ไปแล้วเมื่อเดือนก่อนพร้อมกับ Fix Paramiko ตาม Step 662 แล้ว แต่ DIST ยังไม่ถึงรอบ
จนกระทั่งสัปดาห์นี้)

```bash
CHG0004902: Upgrade IOS-XE บน DIST-SW1-4 จาก 17.09.04a -> 17.12.02
            Maintenance Window: วันอาทิตย์ 01:00-01:40 (ก่อน Cron 02:00 พอดี — Pattern เดิม)
```

```bash
$ ssh -vvv admin@10.10.99.12
...
Unable to negotiate with 10.10.99.12 port 22: no matching key exchange method found.
Their offer: diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,ecdh-sha2-nistp256,ecdh-sha2-nistp384
```

**Root Cause Pipeline B**: เหมือน Step 662 — แต่คราวนี้เกิดเพราะ `requirements.txt` ของ
**Environment ที่รัน Pipeline B คนละตัวกับที่ใช้ Fix ใน Step 662** (Pipeline B รันบน Cron ของ
เครื่อง Physical Server ตัวเก่าที่ทีมยังไม่ได้อัปเดต `paramiko`/`cryptography` เหมือนกับ Virtual
Environment ที่ใช้ทดสอบ Pipeline A) — เป็นตัวอย่างชัดเจนว่า **Fix ที่ทำใน Environment เดียว
ไม่ได้แปลว่า Environment อื่นที่รัน Automation Job เดียวกันจะได้รับ Fix นั้นไปด้วย** ถ้าไม่มี
กระบวนการ Sync Dependency ระหว่าง Environment ที่ Standardize จริง

```bash
$ ssh cron-server "pip show paramiko | grep Version"
Version: 2.7.2   # <-- ยังเป็นเวอร์ชันเก่า ไม่ได้ Sync กับ Fix ของ Step 662
```

**แก้ไข Pipeline B**:

```bash
$ ssh cron-server "pip install --upgrade paramiko cryptography netmiko"
$ ssh cron-server "pip freeze | grep -E 'paramiko|cryptography|netmiko'"
paramiko==3.4.0
cryptography==42.0.5
netmiko==4.3.0

$ ssh cron-server "python3 build_inventory_parallel.py"
[*] เก็บข้อมูลสำเร็จ 6/6 อุปกรณ์
```

### 670.5 ขั้นที่ 4 — ยืนยัน Full Re-run ทั้งสอง Pipeline พร้อมกัน

```bash
$ ansible-playbook -i inventory/dev.yaml site.yaml --check --diff --vault-id dev@.ci_vault_pass/dev.txt
PLAY RECAP: DIST-SW1 : ok=6  changed=0  unreachable=0  failed=0  skipped=0

$ python3 build_inventory_parallel.py
[*] เก็บข้อมูลสำเร็จ 6/6 อุปกรณ์
[*] ใช้เวลาทั้งหมด: 2.38 วินาที (mode=parallel)
```

ทั้งสอง Pipeline กลับสู่สถานะปกติ — **ทั้งสองปัญหาไม่ได้เกี่ยวข้องกันเลยในเชิงเทคนิค** (Vault
Environment Scope Regression กับ SSH KEX Mismatch) แต่บังเอิญปะทะกันคืนเดียวกันเพราะ **ทั้งสอง
Trigger มาจาก 2 เหตุการณ์ที่เกิดในสัปดาห์เดียวกันโดยไม่มีการประสานงาน**: การ Merge Branch เก่าเข้า
`main` (Pipeline A) และ Maintenance Window Phase 2 ของ IOS Upgrade (Pipeline B)

### 670.6 Postmortem — สรุป Root Cause และ Guardrail ที่เพิ่มเข้าไปกันไม่ให้เกิดซ้ำ

| Pipeline | Root Cause | Fix ทันที | Guardrail ระยะยาว |
|---|---|---|---|
| A (Ansible Vault) | CI/CD Variable Environment Scope ถูก Revert กลับผิดโดยไม่ตั้งใจจากการ Merge Branch เก่า | Revert-the-Revert + Terraform Apply ใหม่ | Guard Script ตรวจไฟล์ Vault Password ไม่ว่างก่อนใช้งานเสมอ (Step 666.5) + เพิ่ม Terraform State Check ใน CI ก่อน Merge เข้า `main` |
| B (Netmiko/Nornir) | `paramiko`/`cryptography` ไม่ได้ Sync เวอร์ชันข้าม Environment (Cron Server เก่ายังไม่ได้ Fix) | อัปเดต Library บน Cron Server ให้ตรงกับ Environment อื่น | รวม Environment ทั้งหมดที่รัน Automation Job เดียวกันเข้า **Dependency Management ระบบเดียว** (เช่น `pip-compile`/Container Image เดียวที่ทุก Environment ใช้ Build จากเดียวกัน แทนการติดตั้ง `pip install` แยกแต่ละเครื่อง) |

| มาตรการป้องกันร่วม (ครอบทั้งสอง Pipeline) | รายละเอียด |
|---|---|
| **Pre-flight Health Check ก่อนรัน Nightly Job จริง** | เพิ่ม Step แรกของทุก Pipeline: ทดสอบ Vault Decrypt 1 ตัวแปร + ทดสอบ SSH Connect 1 อุปกรณ์ตัวแทน — ถ้า Fail ให้หยุด Job ทันทีพร้อม Alert ที่ชี้ชัดว่า Fail ที่ขั้นไหน (ไม่ต้องรอให้ Job รันไปครึ่งทางแล้วพังแบบกำกวม) |
| **ผูก Automation Pipeline เข้า Change Calendar เดียวกับ Network Team** | ถ้ามี Maintenance Window ในคืนนั้น ให้ Pipeline แจ้งเตือน/หยุดพักอัตโนมัติ 1 รอบเพื่อรอการยืนยัน Compatibility ก่อน (จาก Step 662.7) |
| **Postmortem ทุกครั้งที่ Automation Fail แบบไม่คาดคิด ต้องระบุ "จะป้องกันไม่ให้เกิดซ้ำอย่างไร" เสมอ ไม่ใช่แค่ "แก้ปัญหาให้ผ่าน"** | หลักการเดียวกับ Postmortem ของ Incident เครือข่ายทั่วไป — นำมาปรับใช้กับ Automation Pipeline เพราะ Automation ที่พังโดยไม่มีคนเฝ้า (Unattended) มีความเสี่ยงสูงกว่า Manual Process ตรงที่ **ไม่มีคนสังเกตทันทีที่เกิดปัญหา** |

---

## แบบฝึกหัดทวนความเข้าใจ Part 67

1. อธิบาย 4 Failure Category ของ Automation Troubleshooting (Step 661) และบอกว่า Scenario ใน
   Step 662 (SSH KEX Mismatch) กับ Step 667 (Compliance False-Positive) จัดอยู่ใน Category ใด
   ตามลำดับ พร้อมเหตุผล
2. ใน Step 663 ทำไม `ansible-playbook --check --diff` ที่แสดง Diff โดยบรรทัด "before" และ "after"
   ดูเหมือนกันทุกตัวอักษรถึงยังถือเป็นสัญญาณเตือนที่ต้องสงสัยทันที และควรตรวจสอบด้วยเครื่องมืออะไร
   ต่อ?
3. ใน Step 665 (NETCONF YANG Validation Error) เพราะเหตุใดการตั้ง `<default-operation>replace
   </default-operation>` ที่ระดับ Container สูงเกินไปจึงทำให้ชน `must` Constraint ได้ และวิธีแก้
   ที่ถูกต้องคืออะไร?
4. ใน Step 666 เพราะเหตุใด Error `"Decryption failed (no vault secrets were found that could
   decrypt)"` จึงอาจไม่ได้หมายความว่า "รหัส Vault ผิด" เสมอไป และควรตรวจสอบอะไรก่อนเชื่อ Error
   Message ตรงตัว?
5. ใน Step 669 เพราะเหตุใดการตั้ง `num_workers` ค่าเดียวสำหรับทั้ง Inventory ที่มี Platform
   หลากหลาย (Catalyst 9500/9300 ปนกับ 2960) จึงเป็นแนวทางที่เสี่ยง และวิธีแก้ที่นำเสนอใน Step
   นั้นทำงานอย่างไร?

**เฉลย:**

1. **Category 1 (Connectivity/Auth)** สำหรับ Step 662 — เพราะปัญหาอยู่ที่การเจรจา SSH ก่อนแม้แต่
   จะ Authenticate สำเร็จ (Layer ต่ำสุดของ Automation Stack) **Category 3 (Logic/Idempotency)**
   สำหรับ Step 667 — เพราะ Script Connect และ Parse ข้อมูลได้ปกติทุกอย่าง แต่ "ตัดสินใจ" ผิดเพราะ
   Golden Config/Terminal Width ไม่ตรงกับความเป็นจริง ทำให้รายงานผลผิดโดยไม่มี Exception ใดๆเกิดขึ้น
2. เพราะ Diff ที่ดูเหมือนกันทุกตัวอักษรแต่ Ansible ยังตัดสินว่า "เปลี่ยน" แปลว่ามี**อักขระที่มองไม่
   เห็นอยู่ในบรรทัดหนึ่ง** (Whitespace, Tab, หรือ Carriage Return `\r`) — ควรตรวจสอบต่อด้วย
   `cat -A` (หรือ `od -c`) กับไฟล์ที่ Render ออกมาจริง เพื่อเผยอักขระที่ Terminal ปกติไม่แสดงให้เห็น
3. เพราะ `default-operation: replace` มีความหมายว่า "แทนที่ทั้ง Subtree ณ ระดับที่ระบุ" ไม่ใช่แค่
   แก้ Field ที่ส่งมาเท่านั้น — ถ้าตั้งที่ระดับสูง (เช่น Container `ip/address` ทั้งก้อน) มันจะลบ
   ข้อมูลอื่นที่มีอยู่แล้ว (เช่น `primary` address) ทิ้งไปโดยไม่ตั้งใจ ทำให้ผลลัพธ์ที่ Validate มี
   แค่ `secondary` เพียงตัวเดียว ชนกับ `must` Constraint ที่บังคับว่าต้องมี `primary` อยู่ก่อนเสมอ
   วิธีแก้คือเปลี่ยนเป็น `merge` (หรือส่ง `primary` มาพร้อมกันใน `edit-config` เดียวกัน)
4. เพราะ Error Message นี้เกิดขึ้นทั้งกรณี "รหัสผิด" และกรณี "ไม่มีรหัสอยู่ในไฟล์เลย" (ไฟล์ว่าง 0
   byte) — ก่อนเชื่อว่ารหัสผิด ควรตรวจสอบไฟล์ Password โดยตรงด้วย `wc -c`/`md5sum` เพื่อดูว่าไฟล์
   มีเนื้อหาอยู่จริงหรือไม่ เพราะสาเหตุที่พบจริงมักเป็นเรื่อง Environment Scope/Variable Injection
   ของ CI/CD ผิดพลาด ไม่ใช่รหัสที่ตั้งไว้ผิดเอง
5. เพราะ Platform รุ่นเก่า (Catalyst 2960) มี CPU/ทรัพยากรจำกัดกว่า Platform รุ่นใหม่มาก — การเปิด
   SSH Connection พร้อมกันหลาย Thread แบบเดียวกับที่ใช้กับ Core/Distribution (`num_workers: 10`)
   ทำให้ SSH Process บนอุปกรณ์รุ่นเก่า CPU พุ่งสูงจนกระทบการทำงานปกติและทำให้ Netmiko อ่าน Output
   ผิดจังหวะ วิธีแก้คือแยก Inventory เป็นกลุ่มตาม Platform ด้วย `nr.filter(role=...)` แล้วกำหนด
   `num_workers` ต่ำ (พร้อม delay เพิ่ม) เฉพาะกลุ่ม Platform รุ่นเก่า โดยกลุ่ม Platform ที่แรงพอยัง
   ใช้ค่าเดิมได้ตามปกติ

---

## สรุป Part 67

Part นี้เปลี่ยนมุมมอง Troubleshooting จาก Protocol Layer ที่คุ้นเคยมาตลอดหลักสูตร ไปสู่
**Automation Stack** ที่สร้างมาตั้งแต่ [Part 23](part-023-automation-basics.md),
[Part 47](part-047-automation-python-advanced.md), [Part 48](part-048-automation-ansible-advanced.md),
และ [Part 49](part-049-netconf-restconf-yang-deep-dive.md) — เริ่มจาก **Methodology** ที่แบ่ง
ปัญหาเป็น 4 Category (Connectivity/Auth, Parsing/Data, Logic/Idempotency, API/Version/Scale)
แล้วไล่วินิจฉัย 8 Scenario จริงที่พบได้ในงาน NetOps: SSH KEX Mismatch หลัง IOS Upgrade,
Idempotency พังจาก CRLF ที่มองไม่เห็น, TextFSM Parse เงียบหลัง Vendor เปลี่ยนคำใน Output,
YANG `must` Constraint ที่ต้องอ่าน `error-path` อย่างเป็นระบบ, Ansible Vault ที่พังเฉพาะบน CI/CD
Runner จาก Environment Scope ผิด, Compliance False-Positive ที่นำไปสู่ Alert Fatigue, RESTCONF
401 จาก Session/AAA Limit ที่ Script ไม่ปิด Connection, และ CPU Spike จาก Concurrency ที่ไม่แยก
ตาม Platform ปิดท้ายด้วย Lab เต็มรูปแบบที่รวมสองปัญหาจริง (Vault Regression + Cipher Mismatch)
เข้าไว้ในคืนเดียวกัน เพื่อฝึกแยกวิเคราะห์หลาย Pipeline ที่ล้มเหลวพร้อมกันโดยไม่ปนกันเป็นปัญหาเดียว
บทเรียนที่สำคัญที่สุดของ Part นี้คือ **Automation ที่ดีต้องมี Guardrail ที่ตรวจจับปัญหาได้เร็วกว่า
คนสังเกตเอง** เพราะ Automation ทำงานแบบ Unattended เป็นค่าเริ่มต้น

**ไปต่อ:** [Part 68 — QoS Troubleshooting →](part-068-qos-troubleshooting.md)
