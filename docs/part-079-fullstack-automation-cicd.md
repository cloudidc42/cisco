# Part 79 — Full-Stack Automation & CI/CD for Network Infrastructure
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 781–790 จาก 1000**

> ต่อจาก [Part 78 — Security Integration (ISE, TrustSec, FTD, Umbrella) →](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
> ที่เราประกอบร่าง Security Stack ทั้งองค์กรให้ทำงานร่วมกันเป็น Ecosystem เดียวแล้ว Part นี้จะพาเรา
> กลับไปที่แกน **Automation** ของหลักสูตรอีกครั้งเป็นครั้งสุดท้ายก่อนเข้าสู่ Telemetry ระดับ Production
> — [Part 23](part-023-automation-basics.md) สอน Netmiko/Ansible/Jinja2 พื้นฐาน,
> [Part 47](part-047-automation-python-advanced.md) ยกระดับ Python ด้วย NAPALM/Nornir/TextFSM/
> Compliance Checking, [Part 48](part-048-automation-ansible-advanced.md) ยกระดับ Ansible ด้วย
> Role/Vault/Idempotency, [Part 49](part-049-netconf-restconf-yang-deep-dive.md) เจาะ NETCONF/
> RESTCONF/YANG, และ [Part 67](part-067-automation-troubleshooting.md) สอนวิธี Troubleshoot
> เมื่อ Automation Stack ทั้งหมดนี้พัง — สิ่งที่ยังขาดหายไปตลอดมาคือคำถามระดับ **CCIE**: **"สคริปต์
> เหล่านี้รันอย่างไรในโลกจริงที่มีความเสี่ยงสูง โดยไม่มีใครคนเดียวเป็นจุดล้มเหลว (Single Point of
> Failure) และไม่มีการเปลี่ยนแปลงใดหลุดเข้า Production โดยไม่มีใครตรวจสอบก่อน"** — Part นี้จะยก
> Script/Playbook ที่มีอยู่แล้วทั้งหมดขึ้นไปสู่ **Network CI/CD Pipeline** เต็มรูปแบบ: Git เป็น
> Single Source of Truth, Automated Testing บน Digital Twin, Peer Approval, Canary Rollout,
> Automated Rollback, และ Secrets Management ระดับ Enterprise ด้วย HashiCorp Vault — นี่คือ
> **Infrastructure as Code (IaC) ที่ประยุกต์ใช้กับเครือข่ายจริง**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 781 | ทำไม Ad-hoc Script (Part 23/47) ไม่พอสำหรับ Production — แรงจูงใจสู่ CI/CD Pipeline |
| 782 | สถาปัตยกรรม Network CI/CD Pipeline — Git as Source of Truth และ 7 Stage หลัก |
| 783 | Pre-commit Validation — `ansible-lint`, `yamllint`, Jinja2 Lint, และ Custom Config Validator |
| 784 | Digital Twin / Test Lab — ทดสอบการเปลี่ยนแปลงบน CML/pyATS Testbed ก่อนแตะ Production |
| 785 | Jenkinsfile เต็มรูปแบบ — Lint → Test → Plan → Peer Approval → Deploy เรียก Playbook จาก Part 48 |
| 786 | Canary/Staged Rollout — Deploy ทีละ Batch พร้อม Automated Health-check Gate |
| 787 | Automated Rollback — ใช้ Git History + Config Backup System (Part 47) Revert อัตโนมัติ |
| 788 | Observability Integration — ส่ง Deployment Marker เข้า Telemetry Stack (Part 54/80) |
| 789 | Secrets Management ระดับ Enterprise — จาก Ansible Vault สู่ HashiCorp Vault |
| 790 | Lab เต็มรูปแบบ: Pipeline ครบวงจร Push ACL ใหม่ทั้ง Fleet พร้อม Successful/Failed Run Walkthrough |

---

## Step 781 — ทำไม Ad-hoc Script ไม่พอสำหรับ Production

### 781.1 ทบทวนสิ่งที่เรามีอยู่แล้ว

ตลอด Part 23/47/48 เราสร้างเครื่องมือ Automation ที่ทรงพลังมาก — Netmiko/NAPALM Script,
Nornir Concurrent Task, Ansible Role พร้อม Vault/Handler/Idempotency — แต่ทุกอย่างมี**รูปแบบการ
ใช้งานเดียวกัน**: วิศวกรคนหนึ่งเปิด Terminal บน Laptop ของตัวเอง แล้วรันคำสั่งตรงๆ

```bash
# วิธีที่เราใช้มาตลอด Part 23-48
$ ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
$ python3 compliance_check.py
```

วิธีนี้ใช้ได้ดีใน Lab หรือทีมขนาดเล็กมาก แต่ในองค์กรที่มีวิศวกรหลายสิบคนดูแลอุปกรณ์หลักร้อยถึง
หลักพันตัว วิธีนี้มีจุดอ่อนที่ร้ายแรงสี่ข้อ ซึ่งล้วนเป็นจุดอ่อนแบบเดียวกับ "การแก้ Config มือ" ที่
Part 23 Step 221 เคยเตือนไว้ — เพียงแต่คราวนี้ **แม้จะ Automate แล้วก็ยังเจอปัญหาเดิม** ถ้าไม่มี
Process ควบคุมการรัน Automation เองอีกที

### 781.2 สี่ปัญหาหลักของ Ad-hoc Automation

| ปัญหา | รายละเอียด | ตัวอย่างที่เกิดขึ้นจริง |
|---|---|---|
| **1. ไม่มี Peer Review** | วิศวกรคนเดียวแก้ Playbook/Template แล้วรันได้ทันทีจาก Laptop ตัวเอง ไม่มีใครตรวจสอบ Logic ก่อน | Engineer-B แก้ `ntp_syslog.j2` ด้วย CRLF (Part 67 Step 663) แล้ว Commit เข้า Git ตรง — ไม่มีใคร Review เห็นปัญหาก่อน Deploy จริง |
| **2. ไม่มี Testing ก่อน Deploy** | Playbook รันตรงกับ Production ทันที ไม่มีขั้นตอนทดสอบบนสภาพแวดล้อมอื่นก่อน | `compliance_check.py` (Part 47 Step 465) ตรวจพบ Drift **หลังจาก** Config ผิดถูก Apply ไปแล้ว ไม่ใช่ **ก่อน** |
| **3. ไม่มี Audit Trail ที่สมบูรณ์** | `git log` (Part 47 Step 468) บอกได้ว่า "ไฟล์เปลี่ยนอะไร" แต่บอกไม่ได้ว่า **"ใครสั่งรัน Playbook นี้กับ Production เมื่อไหร่ ผลลัพธ์เป็นอย่างไร ใครอนุมัติ"** | เกิด Incident กลางดึก ไม่มีใครรู้แน่ชัดว่า Playbook เวอร์ชันไหนถูกรันจริง เพราะรันจาก Laptop คนละเครื่องกัน |
| **4. ไม่มี Rollback Mechanism** | เมื่อ Deploy แล้วพัง วิธีแก้คือ "จำเอาว่า Config เดิมเป็นอย่างไร" แล้วพิมพ์กลับมือ | ไม่มี Process กำหนดตายตัวว่า Rollback ต้องทำอย่างไร ขึ้นกับความจำและความเร็วมือของวิศวกรที่ On-call ในขณะนั้น |

### 781.3 แนวคิด Infrastructure as Code (IaC) ประยุกต์ใช้กับ Network

**Infrastructure as Code** ในโลก Server/Cloud หมายถึงการเขียน Infrastructure (VM, Network, Storage)
เป็น Code ที่ผ่าน Pipeline เดียวกับ Software ทั่วไป — **Network as Code** คือแนวคิดเดียวกันที่นำมาใช้
กับ Config อุปกรณ์เครือข่าย: Config/Playbook/Template ทุกไฟล์ต้องอยู่ใน **Git** และการเปลี่ยนแปลง
ทุกครั้งต้องผ่าน **Pipeline อัตโนมัติ** ที่บังคับให้เกิด Lint → Test → Review → Deploy → Verify
เสมอ ไม่มีทางลัดให้วิศวกรคนเดียวรัน Playbook ตรงกับ Production จาก Laptop ได้อีกต่อไป

| ประเด็น | Ad-hoc Automation (Part 23-48) | Network CI/CD Pipeline (Part 79) |
|---|---|---|
| จุดที่รัน Playbook | Laptop วิศวกรแต่ละคน | CI/CD Runner กลาง (Jenkins/GitLab Runner) เท่านั้น |
| ก่อน Deploy | ไม่มีขั้นตอนบังคับ | Lint + Test บน Digital Twin + Peer Approval บังคับผ่านทุกครั้ง |
| Credential | Vault Password ที่วิศวกรพิมพ์มือ (`--ask-vault-pass`) | Inject จาก Secrets Manager อัตโนมัติ ไม่มีใครเห็น Plaintext (Step 789) |
| Audit Trail | `git log` เท่านั้น (ไม่รู้ว่าใครรันจริง) | Pipeline Log ทุก Run ผูกกับ Commit + ผู้ Approve + ผลลัพธ์ + Timestamp |
| Rollback | จำ Config เดิมเอง/แก้มือ | Automated — Trigger จาก Post-deploy Verification ที่ Fail (Step 787) |
| Deploy Scope | ทุกอุปกรณ์พร้อมกัน (Big-bang) | Canary ก่อน แล้วขยายเป็น Batch (Step 786) |

> **ข้อคิดสำคัญ**: Pipeline ไม่ได้มาแทนที่ Ansible/Python ที่เราสร้างมาตลอด Part 23-49 เลย —
> **Pipeline คือ "กรอบ" ที่ครอบ Playbook/Script เดิมทั้งหมดให้รันอย่างปลอดภัย มีการตรวจสอบ และ
> ทำซ้ำได้เสมอ** — ทุก Ansible Role จาก Part 48 (`ntp-config`, `day2-ops`) และทุก Python Script จาก
> Part 47 (`compliance_check.py`, `build_inventory.py`) จะถูกเรียกใช้จากภายใน Pipeline นี้ทั้งหมด
> ไม่ต้องเขียนใหม่แม้แต่บรรทัดเดียว

---

## Step 782 — สถาปัตยกรรม Network CI/CD Pipeline

### 782.1 Git เป็น Single Source of Truth

หลักการแรกและสำคัญที่สุดของ Network CI/CD คือ: **สิ่งที่อยู่ใน Git คือความจริงเพียงหนึ่งเดียว** —
ไม่ใช่ Running-config บนอุปกรณ์ ถ้าใครแก้ Config มือบนอุปกรณ์โดยไม่ผ่าน Git ถือว่าเป็น **Config
Drift** ที่ต้องถูกจับได้ (ด้วย `compliance_check.py` จาก Part 47 Step 465) และแก้กลับให้ตรงกับ Git

โครงสร้าง Repository มาตรฐานที่ต่อยอดจาก `day2-ops` ใน Part 48 Step 480:

```
network-cicd/
├── Jenkinsfile                    # Pipeline Definition (Step 785)
├── .gitlab-ci.yml                  # ทางเลือกถ้าใช้ GitLab CI แทน Jenkins
├── requirements.yml                # Pin Ansible Collection (Part 48 Step 472)
├── requirements.txt                # Pin Python Library (pyats, requests, napalm)
├── ansible.cfg
├── .ansible-lint                   # กฎ Lint ของ Ansible (Step 783)
├── .yamllint                       # กฎ Lint ของ YAML (Step 783)
├── .pre-commit-config.yaml         # ผูก Lint ทั้งหมดเข้า Git Hook (Step 783)
├── inventory/
│   ├── production.yaml             # Inventory จริง (จาก docs/00-ip-address-plan.md)
│   └── staging.yaml                # Inventory ของ Digital Twin/CML Lab (Step 784)
├── group_vars/
│   ├── network_devices.yml
│   ├── canary_switches.yml         # กลุ่ม Canary (Step 786)
│   └── vault.yml                   # Encrypted (หรือดึงจาก HashiCorp Vault, Step 789)
├── roles/
│   ├── ntp-config/                 # จาก Part 48 Step 471
│   ├── acl-management/             # ตัวอย่างที่จะใช้ใน Step 790
│   └── ...
├── playbooks/
│   ├── day2-ops.yaml                # จาก Part 48 Step 480
│   └── deploy_acl_change.yaml       # Playbook สำหรับ Step 790
├── scripts/
│   ├── validate_config.py           # Custom Validator (Step 783)
│   ├── compliance_check.py          # จาก Part 47 Step 465
│   ├── backup_configs.py            # จาก Part 47 Step 468 (Git-based Backup)
│   ├── rollback.py                  # Automated Rollback (Step 787)
│   └── post_deployment_marker.py    # Observability Hook (Step 788)
└── tests/
    └── pyats/
        ├── testbed_staging.yaml     # pyATS Testbed ของ Digital Twin (Step 784)
        └── verify_change.py         # pyATS Testscript (Step 784/790)
```

### 782.2 เจ็ด Stage หลักของ Pipeline

```
┌─────────┐   ┌─────────┐   ┌───────────────┐   ┌───────────────┐   ┌─────────┐   ┌──────────┐   ┌──────────────────┐
│  LINT   │──►│  TEST   │──►│ PLAN / DRY-RUN │──►│ PEER APPROVAL │──►│ DEPLOY  │──►│  VERIFY  │──►│ ROLLBACK-IF-FAILED │
└─────────┘   └─────────┘   └───────────────┘   └───────────────┘   └─────────┘   └──────────┘   └──────────────────┘
 Syntax/Style   Digital Twin   --check --diff      Human Gate         Canary→Fleet   pyATS/         Auto-revert จาก
 (ansible-lint, (CML + pyATS)  เทียบกับ Production  (Jenkins input     (Step 786)     Compliance     Git History +
  yamllint,     Step 784       จริง ไม่ Apply       step / GitLab                    Check          Config Backup
  custom valid.)               จริง                 manual gate)                    (Step 787)      (Step 787)
```

| Stage | เครื่องมือ | เกณฑ์ผ่าน (Pass Criteria) | ถ้าล้มเหลว |
|---|---|---|---|
| **1. Lint** | `ansible-lint`, `yamllint`, `validate_config.py` | Exit code 0, ไม่มี Syntax/Style Error | Pipeline หยุดทันที ไม่ไปต่อ Stage อื่น |
| **2. Test** | pyATS บน Digital Twin (CML) | Testscript ทุกตัว `PASSED` | Pipeline หยุด แจ้งวิศวกรว่า Logic มีปัญหาบน Lab ก่อนแตะ Production |
| **3. Plan/Dry-run** | `ansible-playbook --check --diff` เทียบกับ Production จริง (ไม่ Apply) | Diff ตรงตามที่ตั้งใจ ไม่มี Side-effect ที่ไม่คาดคิด | แจ้งเตือน, รอวิศวกรตรวจ Diff ก่อนอนุมัติ |
| **4. Peer Approval** | Manual Gate ใน Jenkins/GitLab | Reviewer อีกคน (ไม่ใช่ผู้เขียน Commit) กด Approve | Pipeline หยุดรอ ไม่ Timeout ไปเอง (หรือ Timeout แล้ว Abort) |
| **5. Deploy** | Ansible Playbook (Canary → Full Fleet, Step 786) | ทุก Batch `changed`/`ok` ไม่มี `failed`/`unreachable` | หยุดที่ Batch ที่ล้มเหลว ไม่ไป Batch ถัดไป |
| **6. Verify** | pyATS + `compliance_check.py` บน Production จริงหลัง Deploy | State ตรงกับที่คาดหวัง, Compliance 100% | Trigger Stage 7 ทันที |
| **7. Rollback-if-failed** | `rollback.py` + Git History (Step 787) | Config กลับสู่ Known-good State ที่ยืนยันได้ | แจ้งเตือน Severity สูงสุด ให้ Human เข้ามาดูทันที |

> **หลักการสำคัญ**: Pipeline **หยุดทันทีที่ Stage ไหน Fail** (Fail Fast) — ไม่มีการ "ข้าม Stage ที่
> Fail ไปก่อนแล้วมาแก้ทีหลัง" เด็ดขาด เพราะ Stage หลังทุกตัวสมมติว่า Stage ก่อนหน้าผ่านแล้วจริง
> การข้าม Stage คือการทำลายสมมติฐานนี้และเปิดช่องให้ปัญหาที่ Part 67 เจอ (Idempotency พัง,
> Parsing ผิดเงียบๆ) หลุดเข้า Production ได้อีกครั้ง

### 782.3 Branching Model — GitOps แบบ Network

```
feature/add-new-acl  (วิศวกรแก้ไฟล์ที่นี่)
      │
      │ git push + เปิด Pull Request/Merge Request
      ▼
   [Pipeline รัน Stage 1-3 อัตโนมัติทันทีที่เปิด PR]
      │
      │ Reviewer เห็นผล Lint/Test/Diff ใน PR โดยตรง แล้ว Approve PR
      ▼
   merge เข้า main
      │
      │ [Pipeline รัน Stage 4-7 อัตโนมัติ — Stage 4 (Peer Approval) ยังต้องมี
      │  Human กดยืนยันอีกครั้งก่อน Deploy จริงกับ Production]
      ▼
   Production ถูก Deploy พร้อม Audit Trail สมบูรณ์: Commit -> PR -> Reviewer -> Pipeline Run -> Deploy Log
```

โมเดลนี้แยกความรับผิดชอบชัดเจน: **PR Review** ตรวจ "Logic ของ Code" (Diff ของ Playbook/Template)
ส่วน **Peer Approval Gate ใน Pipeline** ตรวจ "ความพร้อม Deploy จริง" (ดู Diff เทียบ Production
สดๆ ณ เวลา Deploy ซึ่งอาจเปลี่ยนไปจากตอนเปิด PR แล้ว) — ทั้งสองขั้นตอนจำเป็นทั้งคู่ ไม่ใช่ Redundant

---

## Step 783 — Pre-commit Validation

### 783.1 `yamllint` — ตรวจ Syntax/Style ของทุกไฟล์ YAML

```yaml
# .yamllint
---
extends: default

rules:
  line-length:
    max: 160
    level: warning
  indentation:
    spaces: 2
    indent-sequences: true
  comments:
    min-spaces-from-content: 1
  truthy:
    allowed-values: ["true", "false"]   # บังคับใช้ true/false ไม่ใช่ yes/no/True/False ปนกัน
```

```bash
$ yamllint -c .yamllint inventory/ group_vars/ playbooks/ roles/
```

```
group_vars/network_devices.yml
  12:81     warning  line too long (172 > 160 characters)  (line-length)

roles/ntp-config/defaults/main.yaml
  8:1       error    wrong indentation: expected 2 but found 4  (indentation)
```

### 783.2 `ansible-lint` — ตรวจ Best Practice ของ Playbook/Role

```yaml
# .ansible-lint
---
profile: production          # เข้มงวดที่สุด — เหมาะกับ Pipeline Production
exclude_paths:
  - .cache/
  - tests/pyats/
skip_list:
  - yaml[line-length]        # ให้ yamllint จัดการเรื่อง line-length แทนซ้ำ

warn_list:
  - experimental
```

```bash
$ ansible-lint --project-dir . playbooks/day2-ops.yaml
```

```
WARNING  Listing 3 violation(s) that are fatal
playbooks/day2-ops.yaml:37: no-changed-when
Commands should not change things if nothing needs doing.

roles/acl-management/tasks/main.yaml:12: fqcn[action-core]
Use FQCN for builtin module actions (ansible.builtin.debug).

roles/acl-management/tasks/main.yaml:25: risky-file-permissions
File permissions unset or incorrect.
```

`ansible-lint` ตรวจสิ่งที่ `yamllint` มองไม่เห็นเลย — เช่น การใช้ FQCN (Fully Qualified Collection
Name) ที่จำเป็นสำหรับ Ansible ยุคใหม่ (Part 48 Step 472 พูดถึง Collection ที่ Module ไม่รวมมาใน
Core อีกต่อไป) และ Idempotency Anti-pattern อย่าง `no-changed-when` ซึ่งตรงกับปัญหา Category 3
ที่ Part 67 Step 663 เจอมาแล้ว — `ansible-lint` ช่วยจับปัญหาแบบนี้**ก่อน**ที่จะกลาย Incident จริง

### 783.3 Lint Jinja2 Template ด้วย `j2lint`

```bash
pip install j2lint
j2lint roles/*/templates/*.j2
```

```
[j2lint]: Issues found in file: roles/ntp-config/templates/ntp_syslog.j2

jinja2-statements-indentation
  Line: 3 Statement should be indented as parent statement

single-space-decorator
  Line: 5 Jinja statement should have single space before and after decorator: {{ }}
```

> **เชื่อมกับ Part 67 Step 663**: `j2lint` ยังตรวจ Style ไม่ใช่ Line Ending — ปัญหา CRLF ที่ทำให้
> Idempotency พังใน Step 663 ต้องจับด้วย `.gitattributes` (ตามที่แก้ไปแล้ว) ควบคู่กับ `j2lint`
> เพื่อป้องกันปัญหาคนละมุมพร้อมกัน — Lint Stage ที่ดีต้องซ้อนหลายเครื่องมือ ไม่ใช่เครื่องมือเดียว

### 783.4 Custom Config Validator — จับ "ความผิดพลาดเฉพาะ Network" ที่ Lint ทั่วไปมองไม่เห็น

`ansible-lint`/`yamllint` ตรวจ Syntax ของ YAML/Jinja2 ได้ดี แต่ **ไม่รู้จักความหมายของ Network
Config เลย** — มันไม่รู้ว่า ACL Sequence Number ซ้ำกันคือปัญหา หรือ VLAN ID เกินขอบเขตที่องค์กร
กำหนด เราจึงต้องเขียน Validator เฉพาะทางเองที่ต่อยอดจาก `golden_config.py` (Part 47 Step 465)

**เครื่องมือระดับ Enterprise ที่ทำสิ่งนี้แบบเต็มรูปแบบคือ Batfish** — Network Configuration
Analysis Tool แบบ Open Source ที่ Parse Config จริงของหลาย Vendor แล้วจำลอง Control Plane/Data
Plane ทั้ง Network เพื่อตรวจหา Routing Loop, ACL ที่ขัดแย้งกัน, หรือ Unreachable Subnet **ก่อน
Deploy จริง** — สำหรับ Lab ขนาดนี้เราจะเขียน **Custom Python Validator** ที่ครอบคลุมกฎที่พบบ่อย
ที่สุดแทน ซึ่งใช้แนวคิดเดียวกับ Batfish แต่เบากว่ามาก

```python
#!/usr/bin/env python3
"""
scripts/validate_config.py
Pre-commit Custom Validator — ตรวจ "ความผิดพลาดเฉพาะ Network" ในไฟล์ config/template
ที่จะถูก Render ก่อนส่งเข้า Pipeline Stage อื่น (Lint ทั่วไปตรวจ Syntax, ตัวนี้ตรวจ Semantic)
รันเป็นส่วนหนึ่งของ pre-commit hook และ Stage 1 (Lint) ของ Jenkinsfile (Step 785)
"""

import re
import sys
import argparse
from pathlib import Path

# ช่วง VLAN ที่องค์กรอนุญาต (ตาม docs/00-ip-address-plan.md: 10,20,30,40,99 + สำรองสำหรับขยาย)
ALLOWED_VLAN_RANGE = range(1, 1000)
RESERVED_VLAN_IDS = {1, 1002, 1003, 1004, 1005}   # VLAN สำรองของ Cisco ห้ามใช้งานจริง


class ValidationError:
    def __init__(self, filename, line_no, rule_id, message):
        self.filename = filename
        self.line_no = line_no
        self.rule_id = rule_id
        self.message = message

    def __str__(self):
        return f"{self.filename}:{self.line_no}: [{self.rule_id}] {self.message}"


def check_duplicate_acl_sequence(filename, lines):
    """ACL-VAL-01: หา ACL Sequence Number ที่ซ้ำกันภายใน access-list เดียวกัน"""
    errors = []
    current_acl = None
    seen_sequences = {}

    for i, line in enumerate(lines, start=1):
        acl_match = re.match(r"^\s*ip access-list extended (\S+)", line)
        if acl_match:
            current_acl = acl_match.group(1)
            seen_sequences[current_acl] = set()
            continue

        seq_match = re.match(r"^\s*(\d+)\s+(permit|deny)", line)
        if seq_match and current_acl:
            seq_no = int(seq_match.group(1))
            if seq_no in seen_sequences[current_acl]:
                errors.append(ValidationError(
                    filename, i, "ACL-VAL-01",
                    f"ACL '{current_acl}' มี Sequence Number {seq_no} ซ้ำกัน — "
                    f"บรรทัดหลังจะถูก Cisco IOS ปฏิเสธหรือ Overwrite แบบไม่ตั้งใจ",
                ))
            seen_sequences[current_acl].add(seq_no)
    return errors


def check_acl_missing_explicit_deny(filename, lines):
    """ACL-VAL-02: ACL Extended ทุกตัวต้องมี explicit 'deny any log' ปิดท้ายเสมอ (ตาม Golden Config Part 47)"""
    errors = []
    current_acl = None
    acl_start_line = {}
    has_explicit_deny = {}

    for i, line in enumerate(lines, start=1):
        acl_match = re.match(r"^\s*ip access-list extended (\S+)", line)
        if acl_match:
            current_acl = acl_match.group(1)
            acl_start_line[current_acl] = i
            has_explicit_deny[current_acl] = False
            continue

        if current_acl and re.search(r"deny\s+ip\s+any\s+any\s+log", line):
            has_explicit_deny[current_acl] = True

        if current_acl and not line.strip():
            current_acl = None

    for acl_name, has_deny in has_explicit_deny.items():
        if not has_deny:
            errors.append(ValidationError(
                filename, acl_start_line[acl_name], "ACL-VAL-02",
                f"ACL '{acl_name}' ไม่มี explicit 'deny ip any any log' ปิดท้าย — "
                f"ขัดกับ Golden Config Rule (docs/part-047, golden_config.py)",
            ))
    return errors


def check_vlan_id_range(filename, lines):
    """VLAN-VAL-01: VLAN ID ต้องอยู่ในช่วงที่อนุญาต และห้ามเป็น VLAN สำรองของ Cisco"""
    errors = []
    for i, line in enumerate(lines, start=1):
        vlan_match = re.match(r"^\s*vlan\s+(\d+)\s*$", line)
        if vlan_match:
            vlan_id = int(vlan_match.group(1))
            if vlan_id in RESERVED_VLAN_IDS:
                errors.append(ValidationError(
                    filename, i, "VLAN-VAL-01",
                    f"VLAN {vlan_id} เป็น VLAN สำรองของ Cisco (Reserved) ห้ามใช้งานจริง",
                ))
            elif vlan_id not in ALLOWED_VLAN_RANGE:
                errors.append(ValidationError(
                    filename, i, "VLAN-VAL-01",
                    f"VLAN {vlan_id} อยู่นอกช่วงที่องค์กรอนุญาต ({min(ALLOWED_VLAN_RANGE)}-{max(ALLOWED_VLAN_RANGE)})",
                ))
    return errors


VALIDATORS = [
    check_duplicate_acl_sequence,
    check_acl_missing_explicit_deny,
    check_vlan_id_range,
]


def validate_file(filepath: Path):
    lines = filepath.read_text(encoding="utf-8").splitlines()
    all_errors = []
    for validator in VALIDATORS:
        all_errors.extend(validator(str(filepath), lines))
    return all_errors


def main():
    parser = argparse.ArgumentParser(description="Custom Network Config Validator (Pre-commit)")
    parser.add_argument("files", nargs="+", help="ไฟล์ config/generated template ที่จะตรวจ")
    args = parser.parse_args()

    total_errors = 0
    for file_str in args.files:
        filepath = Path(file_str)
        if not filepath.exists():
            continue
        errors = validate_file(filepath)
        for err in errors:
            print(err)
        total_errors += len(errors)

    if total_errors > 0:
        print(f"\n[FAIL] พบปัญหาทั้งหมด {total_errors} รายการ — Pipeline หยุดที่ Stage Lint")
        sys.exit(1)

    print("[PASS] Custom Config Validator ผ่านทั้งหมด ไม่พบปัญหา")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

### 783.5 ตัวอย่าง Output เมื่อพบปัญหาจริง

```bash
$ python3 scripts/validate_config.py generated_configs/ACCESS-SW1-full.cfg
```

```
generated_configs/ACCESS-SW1-full.cfg:14: [ACL-VAL-01] ACL 'MGMT-ACCESS-IN' มี Sequence Number 20 ซ้ำกัน — บรรทัดหลังจะถูก Cisco IOS ปฏิเสธหรือ Overwrite แบบไม่ตั้งใจ
generated_configs/ACCESS-SW1-full.cfg:22: [VLAN-VAL-01] VLAN 1002 เป็น VLAN สำรองของ Cisco (Reserved) ห้ามใช้งานจริง

[FAIL] พบปัญหาทั้งหมด 2 รายการ — Pipeline หยุดที่ Stage Lint
```

### 783.6 ผูกทุก Lint เข้า `.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml
# รันอัตโนมัติทุกครั้งที่ 'git commit' บนเครื่อง Local — ให้ Feedback เร็วที่สุด
# ก่อนที่จะไปเจอ Error ช้าๆที่ Stage 1 ของ Jenkins Pipeline อีกที (Defense in Depth)
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
        args: ["-c", ".yamllint"]

  - repo: https://github.com/ansible/ansible-lint
    rev: v24.2.0
    hooks:
      - id: ansible-lint

  - repo: local
    hooks:
      - id: custom-network-validator
        name: Custom Network Config Validator
        entry: python3 scripts/validate_config.py
        language: system
        files: '\.(cfg|j2)$'
```

```bash
pip install pre-commit
pre-commit install       # ผูก Hook เข้า .git/hooks/pre-commit อัตโนมัติ
git commit -m "Add new ACL rule"
```

```
yamllint.................................................................Passed
ansible-lint..............................................................Passed
Custom Network Config Validator...........................................Failed
- hook id: custom-network-validator
- exit code: 1
generated_configs/ACCESS-SW1-full.cfg:14: [ACL-VAL-01] ACL 'MGMT-ACCESS-IN' มี Sequence Number 20 ซ้ำกัน
```

`git commit` ถูกปฏิเสธ**ตั้งแต่บนเครื่อง Local** — วิศวกรแก้ปัญหาแล้ว Commit ใหม่ได้ทันที ไม่ต้อง
รอ Pipeline บน CI Server (ที่ช้ากว่าและกิน Resource ของทีมมากกว่า) มาบอกปัญหาเดียวกัน

---

## Step 784 — Digital Twin / Test Lab: ทดสอบก่อนแตะ Production

### 784.1 แนวคิด Digital Twin สำหรับ Network

**Digital Twin** คือสภาพแวดล้อมจำลองที่มีโครงสร้างเหมือน Production มากที่สุด (เดียวกับ IOS-XE
Version, Topology แบบเดียวกัน, Config Baseline เดียวกัน) ใช้สำหรับทดสอบการเปลี่ยนแปลง **ก่อนที่จะ
แตะอุปกรณ์จริงแม้แต่ตัวเดียว** — สำหรับ Network มักสร้างด้วย **Cisco Modeling Labs (CML)** หรือ
EVE-NG (Part 1 Step 10 แนะนำไว้แล้วสำหรับ Lab ของหลักสูตรนี้ทั้งหมด) รัน IOS-XE Virtual Image
เวอร์ชันเดียวกับ Production เป๊ะ

```
                     Production Network                          Digital Twin (CML, "staging")
        ┌─────────────────────────────────┐          ┌─────────────────────────────────┐
        │  CORE-SW1 ── CORE-SW2            │          │  CORE-SW1-STG ── CORE-SW2-STG     │
        │     │             │              │  mirror  │     │             │              │
        │  DIST-SW1..4                     │ ───────► │  DIST-SW1-STG..4-STG              │
        │     │                            │ (Topology│     │                            │
        │  ACCESS-SW1..4                   │  + IOS   │  ACCESS-SW1-STG..4-STG            │
        └─────────────────────────────────┘  Version) └─────────────────────────────────┘
              ↑ ห้าม Deploy ตรงที่นี่ก่อน                    ↑ Pipeline Stage 2 ทดสอบที่นี่ก่อนเสมอ
                Stage 5 ทดสอบผ่านแล้วเท่านั้น
```

### 784.2 pyATS/Genie คืออะไร (แนะนำสั้นๆก่อนใช้จริง)

**pyATS (Python Automated Test System)** คือ Test Framework ของ Cisco เอง (Open Source) ออกแบบมา
สำหรับ Test Network โดยเฉพาะ — มี **Genie** เป็นส่วนเสริมที่ให้ Parser สำเร็จรูปสำหรับคำสั่ง `show`
หลายร้อยคำสั่ง (คล้าย TextFSM/ntc-templates ที่ใช้ใน Part 47 Step 463 แต่ออกแบบมาเพื่องาน Testing
โดยเฉพาะ พร้อม Test Report แบบมาตรฐาน)

```bash
pip install pyats[full] genie
```

### 784.3 Testbed File — นิยาม Digital Twin เป็น YAML

```yaml
# tests/pyats/testbed_staging.yaml
# pyATS Testbed ของ Digital Twin (CML) — โครงสร้างเหมือน production.yaml ทุกอย่าง
# ต่างกันเฉพาะ IP (ชี้ไปที่ CML VM แทนอุปกรณ์จริง)
testbed:
  name: staging_digital_twin
  credentials:
    default:
      username: "%ENV{NET_USERNAME}"
      password: "%ENV{NET_PASSWORD}"
    enable:
      password: "%ENV{NET_SECRET}"

devices:
  CORE-SW1-STG:
    os: iosxe
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 172.20.100.10        # CML Management Network — แยกจาก Production 10.10.99.0/24 เสมอ
        port: 22

  DIST-SW1-STG:
    os: iosxe
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 172.20.100.12
        port: 22

  ACCESS-SW1-STG:
    os: iosxe
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 172.20.100.16
        port: 22

topology:
  CORE-SW1-STG:
    interfaces:
      TenGigabitEthernet1/0/1:
        link: core_dist_link1
  DIST-SW1-STG:
    interfaces:
      TenGigabitEthernet1/1/1:
        link: core_dist_link1
```

### 784.4 pyATS Testscript เต็มรูปแบบ — ทดสอบการเปลี่ยนแปลงบน Digital Twin

```python
#!/usr/bin/env python3
"""
tests/pyats/verify_change.py
pyATS Testscript — ตรวจสอบว่าการเปลี่ยนแปลง Config (ที่ Ansible Deploy ไปยัง Digital Twin แล้ว)
ไม่ทำให้ Operational State พังก่อน (Interface ยัง up, OSPF Neighbor ยัง Full, ACL Apply ถูกต้อง)
ใช้ pyats.aetest แบบ Class-based ตามมาตรฐานของ pyATS
"""

import logging
from pyats import aetest
from pyats.topology import loader
from genie.testbed import load

log = logging.getLogger(__name__)


class CommonSetup(aetest.CommonSetup):
    """เชื่อมต่อทุกอุปกรณ์ใน Testbed ก่อนเริ่ม Test จริง"""

    @aetest.subsection
    def load_testbed(self, testbed):
        self.parent.parameters["testbed"] = load(testbed)

    @aetest.subsection
    def connect_all_devices(self, testbed):
        genie_testbed = self.parent.parameters["testbed"]
        for device in genie_testbed.devices.values():
            log.info(f"[*] กำลังเชื่อมต่อ {device.name} ...")
            device.connect(log_stdout=False, learn_hostname=True)


class VerifyInterfacesUp(aetest.Testcase):
    """ตรวจว่า Interface สำคัญ (Uplink/SVI) ยัง up/up หลัง Deploy Config ใหม่"""

    EXPECTED_UP_INTERFACES = {
        "CORE-SW1-STG": ["TenGigabitEthernet1/0/1", "Vlan99"],
        "DIST-SW1-STG": ["TenGigabitEthernet1/1/1", "Vlan99"],
    }

    @aetest.test
    def check_interfaces(self, testbed):
        genie_testbed = self.parent.parameters["testbed"]
        failed_interfaces = []

        for device_name, interfaces in self.EXPECTED_UP_INTERFACES.items():
            device = genie_testbed.devices[device_name]
            parsed = device.parse("show interfaces")   # Genie Parser — structured data เหมือน Part 47 TextFSM

            for intf in interfaces:
                status = parsed.get(intf, {}).get("oper_status", "unknown")
                if status != "up":
                    failed_interfaces.append(f"{device_name}/{intf}: {status}")
                    log.error(f"[FAIL] {device_name} interface {intf} = {status} (คาดหวัง: up)")
                else:
                    log.info(f"[OK] {device_name} interface {intf} = up")

        if failed_interfaces:
            self.failed(f"พบ Interface ที่ไม่ up: {failed_interfaces}")


class VerifyOspfNeighbors(aetest.Testcase):
    """ตรวจว่า OSPF Neighbor ยัง FULL หลัง Deploy — ยืนยันว่า Config ไม่กระทบ Routing"""

    @aetest.test
    def check_ospf_full(self, testbed):
        genie_testbed = self.parent.parameters["testbed"]
        device = genie_testbed.devices["DIST-SW1-STG"]

        parsed = device.parse("show ip ospf neighbor")
        neighbors = parsed.get("interfaces", {})

        non_full = []
        for intf, data in neighbors.items():
            for neighbor_id, nbr_data in data.get("neighbors", {}).items():
                state = nbr_data.get("state", "")
                if "FULL" not in state:
                    non_full.append(f"{intf}/{neighbor_id}: {state}")

        if non_full:
            self.failed(f"พบ OSPF Neighbor ที่ไม่ FULL: {non_full}")
        else:
            log.info("[OK] OSPF Neighbor ทุกตัวอยู่สถานะ FULL")


> **หมายเหตุ**: Testscript นี้ตรวจ Operational Health ทั่วไป (Interface/OSPF) เท่านั้น — การตรวจ
> เนื้อหา ACL แบบละเอียด (Sequence Order, ACE ที่เพิ่มใหม่) จะเขียนเป็น Testscript แยกเฉพาะทางใน
> Step 790 (`verify_acl_change.py`) เพราะกฎที่ต้องตรวจเปลี่ยนไปตาม Config Change แต่ละครั้ง
> ไม่ควรฝังไว้ใน Testscript ทั่วไปที่ใช้ซ้ำได้กับการเปลี่ยนแปลงทุกประเภท

class CommonCleanup(aetest.CommonCleanup):
    """ปิด Connection ของทุกอุปกรณ์เมื่อ Test จบ (สำเร็จหรือล้มเหลวก็ตาม)"""

    @aetest.subsection
    def disconnect_all(self):
        genie_testbed = self.parent.parameters.get("testbed")
        if genie_testbed:
            for device in genie_testbed.devices.values():
                if device.is_connected():
                    device.disconnect()


if __name__ == "__main__":
    aetest.main()
```

### 784.5 รัน Testscript บน Digital Twin

```bash
python3 tests/pyats/verify_change.py --testbed tests/pyats/testbed_staging.yaml
```

**ตัวอย่าง Output กรณีผ่านทั้งหมด (สำเร็จ):**

```
+------------------------------------------------------------------------------+
|                              Detailed Results                                 |
+------------------------------------------------------------------------------+
 SECTIONS/TESTCASES                                                      RESULT
--------------------------------------------------------------------------------
.
|-- CommonSetup                                                           PASSED
|   |-- load_testbed                                                      PASSED
|   `-- connect_all_devices                                               PASSED
|-- VerifyInterfacesUp                                                    PASSED
|   `-- check_interfaces                                                  PASSED
|-- VerifyOspfNeighbors                                                   PASSED
|   `-- check_ospf_full                                                   PASSED
`-- CommonCleanup                                                         PASSED
    `-- disconnect_all                                                    PASSED
--------------------------------------------------------------------------------
Total Passed: 3    Total Failed: 0    Total Errored: 0

Overall Result: PASSED
```

pyATS คืนค่า **Exit Code 0 เมื่อ `PASSED`** และ **Exit Code ไม่เท่ากับ 0 เมื่อมี Testcase `FAILED`**
— เช่นเดียวกับหลักการที่ `compliance_check.py` ใช้มาตั้งแต่ Part 47 Step 465 นี่คือสิ่งที่ทำให้
Jenkins Pipeline ใน Step 785 ตัดสินใจ "ไปต่อ Stage ถัดไป" หรือ "หยุด Pipeline" ได้อัตโนมัติ
โดยไม่ต้องมีคนอ่าน Log ทีละบรรทัด

---

## Step 785 — Jenkinsfile เต็มรูปแบบ

### 785.1 เตรียม Jenkins Credentials ก่อนเขียน Pipeline

```
Jenkins > Manage Jenkins > Credentials
┌─────────────────────────────┬──────────────────────┬─────────────────────────┐
│ Credential ID                │ Type                  │ ใช้เก็บอะไร              │
├─────────────────────────────┼──────────────────────┼─────────────────────────┤
│ vault-password-prod          │ Secret text           │ Ansible Vault Password (ชั่วคราว — Step 789 จะย้ายไป Vault) │
│ net-device-creds             │ Username with password│ NET_USERNAME/NET_PASSWORD │
│ grafana-api-token             │ Secret text           │ Token สำหรับ Annotation API (Step 788) │
└─────────────────────────────┴──────────────────────┴─────────────────────────┘
```

### 785.2 Jenkinsfile — Declarative Pipeline ครบ 7 Stage

```groovy
// Jenkinsfile
// Network CI/CD Pipeline — Lint -> Test -> Plan -> Peer Approval -> Deploy -> Verify -> Rollback-if-failed
// เรียก Ansible Playbook จาก Part 48 (day2-ops) และ Python Script จาก Part 47

pipeline {
    agent { label 'network-automation-runner' }

    environment {
        ANSIBLE_CONFIG   = "${WORKSPACE}/ansible.cfg"
        NET_CREDS        = credentials('net-device-creds')
        VAULT_PASS_FILE  = credentials('vault-password-prod')
    }

    options {
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()   // ป้องกัน 2 Pipeline Run ชนกันบนอุปกรณ์เดียวกัน
    }

    stages {

        stage('1. Lint') {
            steps {
                sh '''
                    python3 -m venv .venv && . .venv/bin/activate
                    pip install -q -r requirements.txt
                    ansible-galaxy collection install -r requirements.yml
                    yamllint -c .yamllint inventory/ group_vars/ playbooks/ roles/
                    ansible-lint --project-dir . playbooks/*.yaml
                    j2lint roles/*/templates/*.j2
                    python3 scripts/validate_config.py $(find generated_configs -name '*.cfg')
                '''
            }
        }

        stage('2. Test on Digital Twin') {
            steps {
                sh '''
                    . .venv/bin/activate
                    # Deploy การเปลี่ยนแปลงไปยัง Digital Twin (CML Staging) ก่อนแตะ Production
                    ansible-playbook -i inventory/staging.yaml playbooks/deploy_acl_change.yaml \
                        --vault-password-file "$VAULT_PASS_FILE"
                    # รัน pyATS Testscript ยืนยัน Operational State บน Digital Twin
                    python3 tests/pyats/verify_change.py --testbed tests/pyats/testbed_staging.yaml
                '''
            }
        }

        stage('3. Plan / Dry-run against Production') {
            steps {
                sh '''
                    . .venv/bin/activate
                    ansible-playbook -i inventory/production.yaml playbooks/deploy_acl_change.yaml \
                        --vault-password-file "$VAULT_PASS_FILE" \
                        --check --diff | tee plan_output.txt
                '''
                archiveArtifacts artifacts: 'plan_output.txt', fingerprint: true
            }
        }

        stage('4. Peer Approval') {
            steps {
                script {
                    def diffSummary = readFile('plan_output.txt')
                    input message: "ตรวจสอบ Diff ด้านล่างก่อนอนุมัติ Deploy จริงเข้า Production:\n\n${diffSummary.take(3000)}",
                          ok: 'อนุมัติ Deploy',
                          submitter: 'network-leads'   // ต้องเป็นสมาชิกกลุ่มนี้เท่านั้นที่กด Approve ได้
                }
            }
        }

        stage('5. Deploy (Canary -> Full Fleet)') {
            steps {
                sh '''
                    . .venv/bin/activate
                    ansible-playbook -i inventory/production.yaml playbooks/deploy_acl_change.yaml \
                        --vault-password-file "$VAULT_PASS_FILE" \
                        --limit canary_switches
                '''
                sh 'python3 scripts/health_check.py --group canary_switches'
                sh '''
                    . .venv/bin/activate
                    ansible-playbook -i inventory/production.yaml playbooks/deploy_acl_change.yaml \
                        --vault-password-file "$VAULT_PASS_FILE" \
                        --limit 'all:!canary_switches'
                '''
            }
        }

        stage('6. Verify') {
            steps {
                sh '''
                    . .venv/bin/activate
                    python3 scripts/compliance_check.py
                    python3 tests/pyats/verify_change.py --testbed inventory/production_pyats.yaml
                '''
            }
        }
    }

    post {
        success {
            sh '. .venv/bin/activate && python3 scripts/post_deployment_marker.py --status success --tag "${GIT_COMMIT}"'
        }
        failure {
            echo "[!] Pipeline ล้มเหลว — เริ่ม Automated Rollback (Step 787)"
            sh '. .venv/bin/activate && python3 scripts/rollback.py --reason "pipeline_failed_build_${BUILD_NUMBER}"'
            sh '. .venv/bin/activate && python3 scripts/post_deployment_marker.py --status rollback --tag "${GIT_COMMIT}"'
        }
        always {
            archiveArtifacts artifacts: '*.log,plan_output.txt', allowEmptyArchive: true
        }
    }
}
```

### 785.3 สิ่งที่ Jenkinsfile นี้บังคับให้เกิดเสมอ

| จุดที่ Pipeline บังคับ | ป้องกันปัญหาจาก Step 781 ข้อไหน |
|---|---|
| Stage 4 ต้องมี Human ใน `network-leads` group กด Approve | ปัญหา #1 (ไม่มี Peer Review) — แก้แล้ว |
| Stage 2 ต้องผ่าน pyATS บน Digital Twin ก่อนถึง Stage 3 | ปัญหา #2 (ไม่มี Testing ก่อน Deploy) — แก้แล้ว |
| Jenkins Build History ผูก Commit + ผู้ Approve + Log ทุก Stage | ปัญหา #3 (ไม่มี Audit Trail) — แก้แล้ว |
| `post { failure { ... rollback.py ... } }` | ปัญหา #4 (ไม่มี Rollback) — แก้แล้ว (รายละเอียด Step 787) |

---

## Step 786 — Canary/Staged Rollout

### 786.1 ทำไมห้าม Deploy ทุกอุปกรณ์พร้อมกัน (Big-bang Deployment)

แม้ Config จะผ่าน Lint, ผ่าน Test บน Digital Twin, และผ่าน Peer Approval มาแล้ว **ก็ยังมีความเสี่ยง
ที่ Digital Twin จำลองไม่ครบ 100%** — เช่น Hardware Platform ต่างรุ่นกันเล็กน้อย (Part 47 Step 464
เจอ Serial/Model ต่างกันแม้ Family เดียวกัน), Traffic Pattern จริงที่ Lab ไม่มี, หรือ Software
Defect เฉพาะบางรุ่น — ถ้า Deploy ผิดพร้อมกันทั้ง Fleet ความเสียหายจะเกิดกับ**ทุกอุปกรณ์ในเวลา
เดียวกัน** ซึ่งกู้คืนยากกว่าการเสียหายเฉพาะจุดมาก

**Canary Deployment** (ยืมชื่อจาก "Canary in a coal mine") คือการ Deploy ไปยัง**อุปกรณ์กลุ่มเล็ก
ที่สุดก่อน** สังเกตผลกระทบจริงในสภาพแวดล้อม Production ก่อนขยายไปยังอุปกรณ์ที่เหลือ

### 786.2 กำหนดกลุ่ม Canary ใน Inventory

```yaml
# inventory/production.yaml
---
all:
  children:
    network_devices:
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
        access_switches:
          hosts:
            ACCESS-SW1: { ansible_host: 10.10.99.16 }
            ACCESS-SW2: { ansible_host: 10.10.99.17 }
            ACCESS-SW3: { ansible_host: 10.10.99.18 }
            ACCESS-SW4: { ansible_host: 10.10.99.19 }

    # กลุ่ม Canary: เลือก Access Switch ที่กระทบผู้ใช้น้อยที่สุด 1 ตัว ไปก่อนเสมอ
    canary_switches:
      hosts:
        ACCESS-SW1:
```

### 786.3 Ansible Playbook พร้อม `serial` — จำกัดจำนวน Host ต่อ Batch

```yaml
---
# playbooks/deploy_acl_change.yaml
- name: "Deploy new ACL rule across fleet (Canary-aware)"
  hosts: all
  gather_facts: no
  connection: ansible.netcommon.network_cli
  serial:
    - 1              # Batch 1: 1 อุปกรณ์เท่านั้น (ใช้คู่กับ --limit canary_switches จาก Jenkinsfile)
    - "50%"          # Batch 2 (เมื่อรันแบบ --limit all:!canary_switches): ครึ่งแรกของที่เหลือ
    - "100%"         # Batch 3: ที่เหลือทั้งหมด
  max_fail_percentage: 0   # แม้แต่ Host เดียว Fail ก็หยุด Batch ถัดไปทันที

  roles:
    - acl-management
```

> **`serial` ทำงานอย่างไร**: Ansible จะรัน Task ทั้งหมดกับ Host ใน Batch แรกให้ครบ (รวม Handler)
> ก่อนเริ่ม Batch ถัดไป — ถ้า `max_fail_percentage` เกินที่กำหนดใน Batch ใดๆ **Ansible จะหยุดทันที
> ไม่ไป Batch ถัดไป** ทำงานร่วมกับ `--limit` ที่ Jenkinsfile Step 785 ใช้แยก Canary ออกจาก Fleet
> ที่เหลืออย่างชัดเจนอีกชั้นหนึ่ง

### 786.4 Health-check Script — Gate ก่อนขยายไป Batch ถัดไป

```python
#!/usr/bin/env python3
"""
scripts/health_check.py
ตรวจสุขภาพของอุปกรณ์ในกลุ่มที่ระบุ (เช่น canary_switches) หลัง Deploy
ก่อนอนุญาตให้ Pipeline ไปยัง Batch ถัดไป — ใช้ NAPALM (Part 47 Step 461) ดึง Structured Data
"""

import sys
import argparse
import yaml
from napalm import get_network_driver
import os

USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")


def load_group_hosts(group_name, inventory_file="inventory/production.yaml"):
    with open(inventory_file) as f:
        inv = yaml.safe_load(f)
    group = inv["all"]["children"].get(group_name, {})
    return group.get("hosts", {})


def check_device_health(hostname, ip):
    driver = get_network_driver("ios")
    device = driver(hostname=ip, username=USERNAME, password=PASSWORD,
                     optional_args={"secret": SECRET})
    issues = []
    try:
        device.open()
        facts = device.get_facts()
        interfaces = device.get_interfaces()
        env = device.get_environment()

        # Health Check 1: Uptime ต้องไม่ Reset (แปลว่าอุปกรณ์ไม่ได้ Reload กลางอากาศตอน Deploy)
        if facts["uptime"] < 60:
            issues.append(f"Uptime ต่ำผิดปกติ ({facts['uptime']}s) — อุปกรณ์อาจ Reload ระหว่าง Deploy")

        # Health Check 2: Interface หลักต้อง up ทั้งหมด
        down_interfaces = [name for name, data in interfaces.items()
                            if data["is_enabled"] and not data["is_up"]]
        if down_interfaces:
            issues.append(f"พบ Interface ที่ down ผิดปกติ: {down_interfaces}")

        # Health Check 3: CPU ต้องไม่พุ่งเกิน Threshold ทันทีหลัง Deploy
        cpu_percent = env.get("cpu", {}).get(0, {}).get("%usage", 0)
        if cpu_percent > 80:
            issues.append(f"CPU Usage สูงผิดปกติ ({cpu_percent}%) ทันทีหลัง Deploy")

        device.close()
    except Exception as e:
        issues.append(f"เชื่อมต่อไม่ได้หลัง Deploy: {e.__class__.__name__}: {e}")

    return issues


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--group", required=True, help="ชื่อ Group ใน Inventory ที่จะตรวจ")
    args = parser.parse_args()

    hosts = load_group_hosts(args.group)
    print(f"[*] Health-check กลุ่ม '{args.group}' ({len(hosts)} อุปกรณ์) หลัง Deploy ...")

    total_issues = 0
    for name, data in hosts.items():
        ip = data.get("ansible_host")
        print(f"    - ตรวจ {name} ({ip}) ...")
        issues = check_device_health(name, ip)
        if issues:
            total_issues += len(issues)
            for issue in issues:
                print(f"        [!] {issue}")
        else:
            print(f"        [OK] ปกติทุกอย่าง")

    if total_issues > 0:
        print(f"\n[FAIL] Health-check พบปัญหา {total_issues} รายการ — ห้ามขยาย Deploy ไป Batch ถัดไป")
        sys.exit(1)

    print(f"\n[PASS] กลุ่ม '{args.group}' สุขภาพปกติ — Pipeline ไปต่อ Batch ถัดไปได้")
    sys.exit(0)


if __name__ == "__main__":
    main()
```

### 786.5 Timeline การขยาย Canary → Full Fleet

```
เวลา 00:00  Deploy ACL ใหม่ -> ACCESS-SW1 (Canary, 1 อุปกรณ์, กระทบ User น้อยที่สุด)
เวลา 00:02  health_check.py ตรวจ ACCESS-SW1 -> PASS (Uptime ปกติ, Interface up, CPU ปกติ)
เวลา 00:05  รอสังเกตผล 5 นาที (Soak Time) — ดู Grafana Dashboard (Step 788) ว่ามี Anomaly ไหม
เวลา 00:10  ไม่พบ Anomaly -> Deploy ต่อไปยัง 50% ที่เหลือ (DIST-SW1/2, ACCESS-SW2/3)
เวลา 00:15  health_check.py ตรวจกลุ่มนี้ -> PASS
เวลา 00:18  Deploy Batch สุดท้าย (100% ที่เหลือ: CORE-SW1/2, DIST-SW3/4, ACCESS-SW4)
เวลา 00:22  health_check.py + compliance_check.py ตรวจทั้ง Fleet -> PASS -> Pipeline สำเร็จ
```

---

## Step 787 — Automated Rollback

### 787.1 สองแหล่งข้อมูลที่ใช้ Rollback

| แหล่งข้อมูล | ใช้ทำอะไร | มาจาก Part ไหน |
|---|---|---|
| **Git History** | บอกว่า Commit ก่อนหน้า (Known-good) คือ Playbook/Template เวอร์ชันไหน | ใช้ `git log`/`git show` เหมือน Part 47 Step 468 |
| **Config Backup System** | บอกว่า Running-config จริงบนอุปกรณ์ ณ ก่อน Deploy คือ Text อะไร (ใช้ Restore ตรงๆได้ทันที) | ต่อยอดจาก `build_inventory.py`/Git-committed backup ของ Part 47 Step 468 |

Config Backup System จาก Part 47 Step 468 (Git Integration) ถูกขยายให้รันอัตโนมัติ **ก่อน** ทุก
Deploy เสมอ ไม่ใช่แค่ Manual Backup เป็นครั้งๆ:

```python
#!/usr/bin/env python3
"""
scripts/backup_configs.py
Backup Running-config ของทุกอุปกรณ์ไปยัง Git Repo แยก (config-backups/) ก่อน Deploy ทุกครั้ง
Commit ด้วย Message ที่มี Timestamp + Git SHA ของ Pipeline Run ที่กำลังจะ Deploy
เพื่อให้ rollback.py หาเวอร์ชัน "ก่อนหน้า" ได้แน่นอน 100%
"""

import os
import subprocess
from datetime import datetime, timezone
import yaml
from netmiko import ConnectHandler

BACKUP_DIR = "config-backups"
USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")


def load_all_devices(inventory_file="inventory/production.yaml"):
    """อ่านรายชื่ออุปกรณ์ทั้งหมดจาก network_devices group เดียวกับ inventory ที่ Step 786 ใช้จริง
    (แทนการ Hardcode List ซ้ำในทุกไฟล์ — ถ้าเพิ่ม/ลบอุปกรณ์ แก้ที่ Inventory ที่เดียวพอ)"""
    with open(inventory_file) as f:
        inv = yaml.safe_load(f)
    children = inv["all"]["children"]["network_devices"]["children"]
    devices = []
    for group in children.values():
        for name, data in group.get("hosts", {}).items():
            devices.append({"name": name, "host": data["ansible_host"]})
    return devices


def backup_one(device):
    conn = ConnectHandler(device_type="cisco_ios", host=device["host"],
                           username=USERNAME, password=PASSWORD, secret=SECRET)
    conn.enable()
    config = conn.send_command("show running-config")
    conn.disconnect()

    path = os.path.join(BACKUP_DIR, f"{device['name']}.cfg")
    with open(path, "w", encoding="utf-8") as f:
        f.write(config)
    return path


def main():
    os.makedirs(BACKUP_DIR, exist_ok=True)
    devices = load_all_devices()
    pipeline_sha = os.environ.get("GIT_COMMIT", "manual-run")
    timestamp = datetime.now(timezone.utc).isoformat()

    for device in devices:
        print(f"[*] Backup {device['name']} ...")
        backup_one(device)

    subprocess.run(["git", "-C", BACKUP_DIR, "add", "."], check=True)
    subprocess.run(
        ["git", "-C", BACKUP_DIR, "commit", "-m",
         f"pre-deploy backup before commit {pipeline_sha} at {timestamp}"],
        check=True,
    )
    # Tag เพื่อให้หา "จุด Known-good ล่าสุดก่อน Deploy นี้" ได้ตรงเสมอ ไม่ต้องเดาจาก Log
    subprocess.run(
        ["git", "-C", BACKUP_DIR, "tag", f"pre-deploy-{pipeline_sha[:8]}"],
        check=True,
    )
    print(f"[OK] Backup ครบ {len(devices)} อุปกรณ์ และ Tag 'pre-deploy-{pipeline_sha[:8]}' แล้ว")


if __name__ == "__main__":
    main()
```

### 787.2 Rollback Script — ดึง Config Known-good ล่าสุดกลับไป Restore

```python
#!/usr/bin/env python3
"""
scripts/rollback.py
Automated Rollback — เรียกจาก post { failure { ... } } ของ Jenkinsfile (Step 785)
ดึง Config Known-good ล่าสุดจาก config-backups/ (Git Tag ที่ backup_configs.py สร้างไว้ก่อน Deploy)
แล้ว Restore กลับด้วย NAPALM load_replace_candidate + compare_config + commit_config
(ใช้แนวคิด Two-Phase Commit เดียวกับ NETCONF candidate/commit ที่เรียนใน Part 49 Step 484
แต่ NAPALM ทำ Abstraction ให้ผ่าน 'configure replace' ของ IOS-XE)
"""

import os
import sys
import argparse
import subprocess
from napalm import get_network_driver
from scripts.backup_configs import load_all_devices   # ใช้ Helper เดียวกับ Step 787.1 (DRY)

BACKUP_DIR = "config-backups"
USERNAME = os.environ.get("NET_USERNAME", "admin")
PASSWORD = os.environ.get("NET_PASSWORD")
SECRET = os.environ.get("NET_SECRET")


def find_latest_pre_deploy_tag():
    """หา Git Tag 'pre-deploy-*' ล่าสุด (ตาม Commit Time) ใน config-backups/"""
    result = subprocess.run(
        ["git", "-C", BACKUP_DIR, "tag", "--sort=-creatordate", "-l", "pre-deploy-*"],
        capture_output=True, text=True, check=True,
    )
    tags = result.stdout.strip().splitlines()
    if not tags:
        sys.exit("[FATAL] ไม่พบ Git Tag 'pre-deploy-*' เลย — ไม่มีจุดอ้างอิงให้ Rollback")
    return tags[0]


def checkout_known_good_config(device_name, tag):
    """ดึงเนื้อหาไฟล์ Config ของอุปกรณ์นั้น ณ Tag ที่ระบุ (ไม่กระทบ Working Tree ปัจจุบัน)"""
    result = subprocess.run(
        ["git", "-C", BACKUP_DIR, "show", f"{tag}:{device_name}.cfg"],
        capture_output=True, text=True, check=True,
    )
    return result.stdout


def rollback_device(device, known_good_config: str):
    driver = get_network_driver("ios")
    napalm_device = driver(hostname=device["host"], username=USERNAME, password=PASSWORD,
                            optional_args={"secret": SECRET})
    napalm_device.open()

    tmp_path = f"/tmp/rollback_{device['name']}.cfg"
    with open(tmp_path, "w") as f:
        f.write(known_good_config)

    napalm_device.load_replace_candidate(filename=tmp_path)   # เทียบเท่า 'configure replace'
    diff = napalm_device.compare_config()

    if not diff:
        print(f"    [SKIP] {device['name']}: ไม่มีความต่างจาก Known-good อยู่แล้ว (อาจยังไม่ถูก Deploy ผิดจริง)")
        napalm_device.discard_config()
    else:
        print(f"    [DIFF] {device['name']}:\n{diff}")
        napalm_device.commit_config()     # Apply Rollback จริง — Atomic เหมือน NETCONF commit (Part 49 Step 484)
        print(f"    [OK] {device['name']}: Rollback สำเร็จ")

    napalm_device.close()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--reason", required=True, help="เหตุผลของการ Rollback (บันทึกลง Log)")
    args = parser.parse_args()

    tag = find_latest_pre_deploy_tag()
    print(f"[*] Rollback Reason: {args.reason}")
    print(f"[*] ใช้ Known-good Config จาก Tag: {tag}")

    failed_rollbacks = []
    for device in load_all_devices():
        print(f"[*] กำลัง Rollback {device['name']} ...")
        try:
            known_good = checkout_known_good_config(device["name"], tag)
            rollback_device(device, known_good)
        except Exception as e:
            failed_rollbacks.append(device["name"])
            print(f"    [ERROR] {device['name']}: Rollback ล้มเหลว — {e}")

    if failed_rollbacks:
        print(f"\n[CRITICAL] Rollback ล้มเหลวบางอุปกรณ์: {failed_rollbacks} — ต้องมี Human เข้าแก้ไขทันที")
        sys.exit(1)

    print(f"\n[OK] Rollback สำเร็จทุกอุปกรณ์ กลับสู่ Known-good ที่ Tag {tag}")


if __name__ == "__main__":
    main()
```

### 787.3 Trigger การ Rollback ในบริบทของ Pipeline

Rollback จะถูกเรียกโดยอัตโนมัติจากสองจุดเท่านั้น (ตาม Jenkinsfile Step 785):

1. **Stage 5 (Deploy) ล้มเหลว** — Batch ใด Batch หนึ่งของ `ansible-playbook` คืน `failed`/`unreachable`
2. **Stage 6 (Verify) ล้มเหลว** — `compliance_check.py` หรือ `pyATS` พบว่า Post-deploy State
   ไม่ตรงตามที่คาดหวัง แม้ Ansible จะรายงาน `changed` สำเร็จก็ตาม (Config Apply สำเร็จ แต่ผลลัพธ์
   ทาง Operational ผิด — Category ที่ Part 67 Step 665 เจอกับ YANG `must` Constraint)

> **จุดสำคัญ**: Rollback ไม่ได้ผูกกับ "Ansible Exit Code" อย่างเดียว — มันผูกกับ **Stage 6
> (Verify) เป็นหลัก** เพราะ Ansible อาจรายงาน `ok`/`changed` สำเร็จทุกอย่าง แต่ Operational State
> จริงพังก็ได้ (เหมือนที่ TextFSM เงียบๆคืนค่าผิดใน Part 67 Step 664) — **Verify Stage ที่ตรวจ
> State จริงบนอุปกรณ์คือ Safety Net ตัวสุดท้ายก่อนตัดสินใจ Rollback**

---

## Step 788 — Observability Integration

### 788.1 ทำไม Deployment ต้องเป็น "Event" ใน Telemetry Stack

Part 54 Step 535 สร้าง TIG Stack (Telegraf → InfluxDB → Grafana) ไว้แล้วสำหรับ Streaming
Telemetry — ปัญหาคือถ้า Metric พุ่งผิดปกติตอน 00:15 น. วิศวกรที่เห็น Dashboard จะไม่รู้ทันทีว่า
**"นี่เกิดจาก Deployment ที่เพิ่งรันไปเมื่อ 00:10 หรือเป็นปัญหาอื่นที่ไม่เกี่ยวกัน"** — การส่ง
**Deployment Marker** (Annotation) เข้า Grafana ทุกครั้งที่ Pipeline Deploy คือสิ่งที่ปิดช่องว่างนี้

```
Timeline บน Grafana Dashboard:
─────┬─────────────────────┬───────────────────────────┬─────────────────►
     │                     │                           │
  00:00                 00:10                        00:15
                           │ 🔵 Deploy: commit a3f9c2   │ 🔴 CPU Spike บน DIST-SW2
                           │    (จาก post_deployment_    │    (Grafana Alert Rule)
                           │     marker.py)              │
                           └─────────────────────────────┘
                        วิศวกรเห็นทันทีว่า Spike เกิดขึ้น "5 นาทีหลัง Deploy"
                        ไม่ต้องเดาหรือไปเปิด Jenkins Log เทียบเวลาเอง
```

### 788.2 Grafana Annotations API — ส่ง Marker จาก Pipeline

```python
#!/usr/bin/env python3
"""
scripts/post_deployment_marker.py
ส่ง Deployment Event เข้า Grafana ผ่าน Annotations API (ต่อยอดจาก TIG Stack ของ Part 54 Step 535)
ให้ Deploy ทุกครั้งกลายเป็นเส้นแนวตั้งบน Dashboard พร้อม Label สถานะ (success/rollback)
เรียกจาก post { success { ... } } และ post { failure { ... } } ของ Jenkinsfile (Step 785)
"""

import os
import sys
import time
import argparse
import requests

GRAFANA_URL = "https://noc-srv.lab.local:3000"      # NOC-SRV เดียวกับ TIG Stack (Part 54)
GRAFANA_API_TOKEN = os.environ.get("GRAFANA_API_TOKEN")

STATUS_COLOR = {
    "success": ["deploy", "success"],
    "rollback": ["deploy", "rollback", "alert"],
    "failed": ["deploy", "failed", "alert"],
}


def post_annotation(status: str, commit_sha: str, text: str):
    if not GRAFANA_API_TOKEN:
        sys.exit("[!] กรุณา export GRAFANA_API_TOKEN ก่อนรัน script นี้")

    payload = {
        "time": int(time.time() * 1000),     # Grafana ต้องการ Epoch Millisecond
        "tags": STATUS_COLOR.get(status, ["deploy"]),
        "text": f"[{status.upper()}] Commit {commit_sha[:8]} — {text}",
    }

    response = requests.post(
        f"{GRAFANA_URL}/api/annotations",
        headers={"Authorization": f"Bearer {GRAFANA_API_TOKEN}",
                 "Content-Type": "application/json"},
        json=payload,
        verify=False,
        timeout=10,
    )
    response.raise_for_status()
    return response.json()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--status", required=True, choices=["success", "rollback", "failed"])
    parser.add_argument("--tag", required=True, help="Git Commit SHA ของ Pipeline Run นี้")
    args = parser.parse_args()

    text_map = {
        "success": "Deploy สำเร็จทุก Stage รวม Canary + Full Fleet + Verify",
        "rollback": "Deploy ล้มเหลว — Automated Rollback ถูก Trigger และสำเร็จแล้ว",
        "failed": "Deploy ล้มเหลว และ Rollback ก็ล้มเหลวด้วย — ต้องการ Human ทันที",
    }

    result = post_annotation(args.status, args.tag, text_map[args.status])
    print(f"[OK] ส่ง Annotation เข้า Grafana สำเร็จ (Annotation ID: {result.get('id')})")


if __name__ == "__main__":
    main()
```

### 788.3 เชื่อมกับ Syslog Annotation ที่มีอยู่แล้วใน Part 54

Part 54 Step 539 ตั้งค่า Telegraf `syslog` Input Plugin ไว้บน NOC-SRV แล้วให้ Grafana ดึง Syslog
Event มาเป็น Annotation อัตโนมัติอยู่แล้ว — เราจึงเพิ่ม Path ที่สองที่ทำสิ่งเดียวกันแบบไม่ต้องพึ่ง
Grafana API โดยตรง (เผื่อ Pipeline ไม่มี Network Reach ถึง Grafana แต่มี Reach ถึง Syslog Server):

```python
import logging
import logging.handlers

syslog_handler = logging.handlers.SysLogHandler(address=("10.10.99.50", 514))
pipeline_logger = logging.getLogger("network-cicd")
pipeline_logger.addHandler(syslog_handler)
pipeline_logger.warning(
    f"AUTOMATION-EVENT: deploy status=success commit={os.environ.get('GIT_COMMIT', '')[:8]}"
)
```

ทั้งสองแนวทาง (Grafana Annotation API ตรง และ Syslog ที่ Telegraf Scrape) ให้ผลลัพธ์เดียวกันบน
Dashboard — องค์กรจริงมักใช้ทั้งสองแบบซ้อนกันเพื่อ Redundancy ของ Observability เอง

---

## Step 789 — Secrets Management ระดับ Enterprise: จาก Ansible Vault สู่ HashiCorp Vault

### 789.1 ข้อจำกัดของ Ansible Vault ที่ Scale ไม่ได้

Part 48 Step 477 สอน Ansible Vault แบบ Multi-Environment (Vault-ID) ไว้อย่างละเอียด และ Part 67
Step 666 แสดงให้เห็นแล้วว่าแม้แต่ระบบที่ออกแบบมาดีก็ยังเจอปัญหาการ Map ตัวแปร CI/CD ผิด — ปัญหา
เชิงโครงสร้างที่ Ansible Vault ยังแก้ไม่ได้มีดังนี้:

| ข้อจำกัดของ Ansible Vault | ผลกระทบระดับ Enterprise |
|---|---|
| Password เป็น **Static** ไม่หมดอายุเอง | ถ้า Password รั่ว ต้อง Rotate มือทุกที่ที่ใช้ Password นั้น |
| ไม่มี **Audit Log** ว่าใคร Decrypt เมื่อไหร่ | ตรวจสอบย้อนหลังไม่ได้ว่าใครเคยเห็น Secret บ้าง |
| Credential เดียวใช้กับทุกอุปกรณ์ในกลุ่มเดียวกัน | ถ้า Credential รั่ว กระทบทุกอุปกรณ์ในกลุ่มพร้อมกัน |
| ต้องมีไฟล์ Password วางไว้ที่ใดที่หนึ่งบน CI Runner เสมอ | เพิ่ม Attack Surface — ไฟล์นั้นเองอาจรั่วได้ (Step 666) |

### 789.2 HashiCorp Vault — ภาพรวมสถาปัตยกรรม

**HashiCorp Vault** คือ Secrets Manager ที่ออกแบบมาสำหรับ Dynamic Secrets, Short-lived
Credential, และ Audit Log แบบสมบูรณ์ — แนวคิดหลักที่ต่างจาก Ansible Vault คือ **Secret ไม่ใช่
"ไฟล์ที่เข้ารหัสแล้ววางไว้เฉยๆ" แต่เป็น "สิ่งที่ต้องขอ (Request) จาก Vault Server ทุกครั้งที่จะใช้"**

```
┌─────────────┐    1. Authenticate (AppRole:      ┌──────────────────────┐
│   Jenkins    │       role_id + secret_id)         │   HashiCorp Vault     │
│   Runner     │ ──────────────────────────────────►│      Server           │
│              │                                     │                        │
│              │    2. ได้ Client Token (TTL สั้น)    │  ┌──────────────────┐  │
│              │◄────────────────────────────────────┤  │ KV Secrets Engine │  │
│              │                                     │  │ secret/network/*  │  │
│              │    3. ขอ Secret (device creds/       │  └──────────────────┘  │
│              │       TACACS+ key) ด้วย Token นี้    │  ┌──────────────────┐  │
│              │ ──────────────────────────────────►│  │ Audit Log Device   │  │
│              │                                     │  │ (ทุก Request ถูก   │  │
│              │    4. ได้ Secret กลับมา              │  │  บันทึกถาวร)        │  │
│              │◄────────────────────────────────────┤  └──────────────────┘  │
│              │                                     └──────────────────────┘
│  Ansible     │    5. Secret ถูก Inject เข้า Environment Variable/File ชั่วคราว
│  Playbook    │       เท่านั้น ไม่เคย Commit เข้า Git หรือเขียนถาวรบน Disk
│  รันจริง     │
└─────────────┘
```

### 789.3 องค์ประกอบสำคัญที่ทำให้ Vault Scale ได้จริง

| องค์ประกอบ | หน้าที่ | เทียบกับ Ansible Vault |
|---|---|---|
| **AppRole Auth Method** | ให้ CI Runner Authenticate ด้วย `role_id` (Public, เก็บใน Jenkins Credential ได้) + `secret_id` (Private, หมดอายุได้และใช้ครั้งเดียวได้) แทน Password ตายตัว | Ansible Vault ใช้ Password เดียวตายตัวไม่มีวันหมดอายุ |
| **Dynamic Secrets** | Vault สร้าง Credential ใหม่ทุกครั้งที่ขอ (เช่น TACACS+ Key ชั่วคราว) แล้วเพิกถอน (Revoke) อัตโนมัติเมื่อ Lease หมดอายุ | Ansible Vault เก็บ Credential ตัวเดิมถาวรจนกว่าจะมีคน Rotate มือ |
| **Lease TTL** | Secret ที่ขอมามีอายุจำกัด (เช่น 15 นาที) พอสำหรับ 1 Pipeline Run เท่านั้น | ไม่มีแนวคิดนี้ใน Ansible Vault |
| **Audit Log Device** | บันทึกทุก Request ว่า "ใคร (Token ไหน) ขอ Secret อะไร เมื่อไหร่" ถาวร | Ansible Vault ไม่มี Log ระดับนี้เลย |
| **Policy (ACL)** | กำหนดได้ละเอียดว่า Token จาก AppRole นี้เข้าถึง Path ไหนได้บ้าง (เช่น `secret/network/dev/*` เท่านั้น อ่าน `prod/*` ไม่ได้) | Ansible Vault-ID (Part 48 Step 477) ทำได้แค่แยกไฟล์ ไม่ได้แยกระดับ Path ละเอียดขนาดนี้ |

### 789.4 เก็บ Credential ของอุปกรณ์เครือข่ายไว้ใน Vault (KV Secrets Engine)

```bash
# เปิด KV Secrets Engine เวอร์ชัน 2 (มี Versioning ในตัว)
vault secrets enable -path=secret/network kv-v2

# เก็บ Credential ของ Network Devices (แทนที่ group_vars/vault.yml เดิมจาก Part 48)
vault kv put secret/network/production \
    net_username="apiuser" \
    net_password="Pr0d-Str0ng-P@ssw0rd-2026!" \
    net_secret="Pr0d-En@ble-Secret-2026!" \
    tacacs_key="Prod-TACACS-Key-VeryStrong-2026!"

# สร้าง Policy จำกัดสิทธิ์ — AppRole ของ Jenkins อ่านได้เฉพาะ Path นี้เท่านั้น
vault policy write network-cicd-policy - <<EOF
path "secret/data/network/production" {
  capabilities = ["read"]
}
EOF

# เปิด AppRole Auth Method และผูกกับ Policy ข้างบน
vault auth enable approle
vault write auth/approle/role/network-cicd-runner \
    token_policies="network-cicd-policy" \
    token_ttl=15m \
    token_max_ttl=30m
```

```bash
# ดึง role_id (เก็บใน Jenkins Credential แบบ Plaintext ได้ — ไม่ Sensitive คนเดียว)
vault read auth/approle/role/network-cicd-runner/role-id
# role_id = 8f2a1c3d-...

# สร้าง secret_id ใหม่ (Sensitive — สร้างครั้งเดียว ให้ Jenkins เก็บแบบ Secret Credential)
vault write -f auth/approle/role/network-cicd-runner/secret-id
# secret_id = 7e8b1a0c-...
```

### 789.5 Jenkinsfile ที่แก้ให้ดึง Secret จาก Vault แทน Ansible Vault Password File

```groovy
// เพิ่มเข้า Jenkinsfile ก่อน Stage 'Deploy' — แทนที่การใช้ VAULT_PASS_FILE (Ansible Vault) เดิม
stage('Fetch Secrets from HashiCorp Vault') {
    steps {
        withVault(
            configuration: [vaultUrl: 'https://vault.lab.local:8200', vaultCredentialId: 'vault-approle'],
            vaultSecrets: [[
                path: 'secret/data/network/production',
                secretValues: [
                    [envVar: 'NET_USERNAME', vaultKey: 'net_username'],
                    [envVar: 'NET_PASSWORD', vaultKey: 'net_password'],
                    [envVar: 'NET_SECRET',   vaultKey: 'net_secret'],
                ]
            ]]
        ) {
            sh '''
                . .venv/bin/activate
                # ansible-playbook อ่าน NET_USERNAME/NET_PASSWORD/NET_SECRET จาก Environment
                # เหมือนที่ทำมาตลอดตั้งแต่ Part 23-49 — เปลี่ยนแค่ "แหล่งที่มา" ของค่านี้เท่านั้น
                ansible-playbook -i inventory/production.yaml playbooks/deploy_acl_change.yaml
            '''
        }
        // Environment Variable ถูกล้างออกจาก Memory ทันทีที่ withVault block จบ
        // ไม่มีไฟล์ Plaintext Password เหลือบน Disk ของ Runner เลย
    }
}
```

> **จุดเปลี่ยนสำคัญ**: สังเกตว่า Ansible Playbook เองไม่ต้องแก้อะไรเลย — มันยังอ่าน
> `NET_USERNAME`/`NET_PASSWORD`/`NET_SECRET` จาก Environment Variable เหมือนเดิมทุกประการ
> (Best Practice ที่ยึดมาตั้งแต่ Part 23) **HashiCorp Vault เปลี่ยนแค่ "วิธีที่ค่าเหล่านี้เข้ามาอยู่
> ใน Environment Variable"** จาก "อ่านจากไฟล์ Vault Password ที่วิศวกรพิมพ์มือ" เป็น "ดึงจาก Vault
> Server แบบ Ephemeral ที่มี Audit Log ครบและหมดอายุอัตโนมัติ" — นี่คือเหตุผลที่การย้ายไป Secrets
> Manager ระดับนี้ **ไม่กระทบ Automation Code ที่มีอยู่แล้วเลย**

---

## Step 790 — Lab เต็มรูปแบบ: Pipeline ครบวงจร Push ACL ใหม่ทั้ง Fleet

### 790.1 สถานการณ์ของ Lab

ทีม Security ขอให้เพิ่ม ACL Rule ใหม่บน `MGMT-ACCESS-IN` (ACL เดิมจาก Part 48 Step 474) เพื่อ
อนุญาต Subnet ของทีม NetOps ชุดใหม่ (`10.10.199.0/24`) ให้เข้าถึง Management VLAN ได้ — เราจะรัน
Pipeline เต็มรูปแบบทั้ง 7 Stage สองครั้ง: **ครั้งแรกสำเร็จ**, **ครั้งที่สองล้มเหลวและ Rollback
อัตโนมัติ** (จำลอง ACL ที่เขียนผิดจนกระทบ Connectivity)

### 790.2 การเปลี่ยนแปลงที่ต้อง Deploy

```yaml
# roles/acl-management/vars/main.yaml
---
mgmt_acl_new_rule:
  sequence: 15
  action: permit
  protocol: ip
  source: "10.10.199.0 0.0.0.255"
  destination: any
```

```yaml
# roles/acl-management/tasks/main.yaml
---
- name: "[acl-management] Insert new ACE into MGMT-ACCESS-IN"
  cisco.ios.ios_config:
    lines:
      - "{{ mgmt_acl_new_rule.sequence }} {{ mgmt_acl_new_rule.action }} {{ mgmt_acl_new_rule.protocol }} {{ mgmt_acl_new_rule.source }} {{ mgmt_acl_new_rule.destination }}"
    parents: "ip access-list extended MGMT-ACCESS-IN"
    before: "20 deny any log"     # แทรกก่อน explicit deny เสมอ — ไม่งั้น ACL จะ deny ก่อนถึง permit ใหม่
  notify: "save running-config"
  tags: ["acl", "security"]

- name: "[acl-management] Verify ACL still ends with explicit deny"
  cisco.ios.ios_command:
    commands:
      - "show ip access-lists MGMT-ACCESS-IN"
  register: acl_verify
  tags: ["acl", "security"]
```

### 790.3 pyATS Testscript สำหรับ Lab นี้โดยเฉพาะ (ต่อยอดจาก Step 784)

```python
#!/usr/bin/env python3
"""
tests/pyats/verify_acl_change.py
ทดสอบเฉพาะการเปลี่ยนแปลง ACL ของ Lab นี้ — ใช้ทั้งบน Digital Twin (Stage 2) และ Production (Stage 6)
โดยเปลี่ยนแค่ Testbed File ที่ชี้เข้ามา (--testbed) ไม่ต้องแก้ Testscript เลย
"""

import logging
from pyats import aetest
from genie.testbed import load

log = logging.getLogger(__name__)

EXPECTED_NEW_ACE_SOURCE = "10.10.199.0"


class CommonSetup(aetest.CommonSetup):
    @aetest.subsection
    def connect(self, testbed):
        tb = load(testbed)
        self.parent.parameters["testbed"] = tb
        for device in tb.devices.values():
            device.connect(log_stdout=False)


class VerifyNewAceExists(aetest.Testcase):
    """ACE ใหม่ (10.10.199.0/24) ต้องปรากฏใน ACL จริง"""

    @aetest.test
    def check_new_ace(self, testbed):
        tb = self.parent.parameters["testbed"]
        device = tb.devices["ACCESS-SW1"] if "ACCESS-SW1" in tb.devices else tb.devices["ACCESS-SW1-STG"]

        parsed = device.parse("show ip access-lists MGMT-ACCESS-IN")
        aces = parsed.get("acls", {}).get("MGMT-ACCESS-IN", {}).get("aces", {})

        found = any(EXPECTED_NEW_ACE_SOURCE in str(ace) for ace in aces.values())
        if not found:
            self.failed(f"ไม่พบ ACE ใหม่ที่มี Source {EXPECTED_NEW_ACE_SOURCE} ใน MGMT-ACCESS-IN")
        else:
            log.info(f"[OK] พบ ACE ใหม่สำหรับ {EXPECTED_NEW_ACE_SOURCE}/24 แล้ว")


class VerifyExplicitDenyStillLast(aetest.Testcase):
    """explicit deny ต้องยังอยู่บรรทัดสุดท้ายเสมอ (ไม่ใช่ถูก ACE ใหม่แทรกไปอยู่หลัง)"""

    @aetest.test
    def check_deny_order(self, testbed):
        tb = self.parent.parameters["testbed"]
        device = tb.devices["ACCESS-SW1"] if "ACCESS-SW1" in tb.devices else tb.devices["ACCESS-SW1-STG"]

        parsed = device.parse("show ip access-lists MGMT-ACCESS-IN")
        aces = parsed.get("acls", {}).get("MGMT-ACCESS-IN", {}).get("aces", {})
        sequences = sorted(int(seq) for seq in aces.keys())

        last_seq = sequences[-1]
        last_ace = aces[str(last_seq)]
        if "deny" not in str(last_ace.get("actions", {}).get("forwarding", "")):
            self.failed("Sequence Number สุดท้ายไม่ใช่ deny — ACL Order ผิดพลาดร้ายแรง (Security Gap)")
        else:
            log.info(f"[OK] Sequence {last_seq} (deny) ยังคงอยู่ท้ายสุดตามที่คาดหวัง")


class VerifyMgmtConnectivityStillWorks(aetest.Testcase):
    """ต้องยัง SSH เข้า Management ได้จาก Subnet เดิม — ยืนยันว่า ACL ใหม่ไม่ Block ของเดิมโดยไม่ตั้งใจ"""

    @aetest.test
    def check_existing_access_not_broken(self, testbed):
        tb = self.parent.parameters["testbed"]
        device = tb.devices["ACCESS-SW1"] if "ACCESS-SW1" in tb.devices else tb.devices["ACCESS-SW1-STG"]

        # ถ้า Connect สำเร็จมาถึงจุดนี้ได้ (ผ่าน CommonSetup มาแล้ว) แปลว่า Management Access เดิมยังทำงาน
        if not device.is_connected():
            self.failed("Management Access เดิมถูก Block ไปแล้ว — ACL ใหม่ทำให้เข้าอุปกรณ์ไม่ได้")
        else:
            log.info("[OK] Management Access เดิมยังใช้งานได้ปกติ")


class CommonCleanup(aetest.CommonCleanup):
    @aetest.subsection
    def disconnect(self):
        tb = self.parent.parameters.get("testbed")
        if tb:
            for device in tb.devices.values():
                if device.is_connected():
                    device.disconnect()


if __name__ == "__main__":
    aetest.main()
```

### 790.4 Walkthrough #1 — SUCCESSFUL RUN

```
========================================================================
JENKINS BUILD #142 — feature/add-netops-subnet-acl merged to main
========================================================================

[Stage 1: Lint]                                                    PASSED (18s)
  yamllint ................................................ 0 error
  ansible-lint ............................................. 0 error
  j2lint .................................................... 0 error
  validate_config.py ........................................ 0 issue

[Stage 2: Test on Digital Twin]                                    PASSED (94s)
  ansible-playbook -i inventory/staging.yaml deploy_acl_change.yaml
    changed: [ACCESS-SW1-STG]
  pyATS verify_acl_change.py --testbed testbed_staging.yaml
    VerifyNewAceExists ............................. PASSED
    VerifyExplicitDenyStillLast ..................... PASSED
    VerifyMgmtConnectivityStillWorks ................. PASSED
    Overall Result: PASSED

[Stage 3: Plan / Dry-run against Production]                       PASSED (12s)
  --check --diff:
    + 15 permit ip 10.10.199.0 0.0.0.255 any
  Diff ตรงตามที่ตั้งใจ — ไม่มี Side-effect

[Stage 4: Peer Approval]                                           APPROVED
  ผู้ Approve: netops-lead-somchai (สมาชิก group 'network-leads')
  เวลา: 2026-09-26 09:42:11 +07:00

[Stage 5: Deploy (Canary -> Full Fleet)]                            PASSED (41s)
  Canary (ACCESS-SW1): changed=1
  health_check.py --group canary_switches ................ PASS
  Full Fleet (9 อุปกรณ์ที่เหลือ): changed=9

[Stage 6: Verify]                                                   PASSED (22s)
  compliance_check.py ...................................... 0 violation
  pyATS verify_acl_change.py --testbed production_pyats.yaml
    Overall Result: PASSED

[post { success }]
  post_deployment_marker.py --status success --tag a3f9c2e1
  [OK] ส่ง Annotation เข้า Grafana สำเร็จ

========================================================================
BUILD #142: SUCCESS (Total: 3m 47s)
========================================================================
```

### 790.5 Walkthrough #2 — FAILED RUN ที่ Trigger Automated Rollback

จำลองสถานการณ์: วิศวกรอีกคน Deploy การเปลี่ยนแปลงชุดที่สอง (แก้ ACL Sequence ที่มีอยู่แล้วผิดจน
`before: "20 deny any log"` ชี้ผิดตำแหน่ง ทำให้ ACE ใหม่ไปแทรก **หลัง** explicit deny โดยไม่ตั้งใจ
— ปัญหาแบบเดียวกับที่ Custom Validator (Step 783) ควรจับได้ แต่สมมติว่าหลุดผ่าน Lint มาได้เพราะ
Sequence Number ไม่ซ้ำกัน จึงเป็นตัวอย่างที่ดีว่า **ทำไม Pipeline ต้องมีหลาย Stage ซ้อนกัน ไม่ใช่
พึ่ง Lint อย่างเดียว**):

```
========================================================================
JENKINS BUILD #143 — feature/add-partner-subnet-acl merged to main
========================================================================

[Stage 1: Lint]                                                    PASSED (17s)
[Stage 2: Test on Digital Twin]                                     PASSED (91s)
  (บน Digital Twin บังเอิญไม่มี ACE เก่าให้ชนตำแหน่งผิด — Testbed ไม่ครบ 100%
   เหมือน Production เป๊ะ ตามที่เตือนไว้ใน Step 786.1)
[Stage 3: Plan / Dry-run against Production]                       PASSED (11s)
[Stage 4: Peer Approval]                                            APPROVED

[Stage 5: Deploy (Canary -> Full Fleet)]                            PASSED (39s)
  Canary (ACCESS-SW1): changed=1
  health_check.py --group canary_switches ................ PASS
      (Health-check ตรวจ Interface/CPU/Uptime — ไม่ตรวจ Content ของ ACL โดยตรง
       จึงยังผ่านแม้ ACL Order จะผิดแล้ว)
  Full Fleet: changed=9

[Stage 6: Verify]                                                   FAILED (24s)
  compliance_check.py ...................................... 0 violation (Golden Config ไม่ครอบคลุม ACE นี้)
  pyATS verify_acl_change.py --testbed production_pyats.yaml
    VerifyNewAceExists .............................. PASSED
    VerifyExplicitDenyStillLast ...................... FAILED
        AssertionError: Sequence Number สุดท้ายไม่ใช่ deny — ACL Order ผิดพลาดร้ายแรง (Security Gap)
    Overall Result: FAILED

[post { failure }]
  [!] Pipeline ล้มเหลว — เริ่ม Automated Rollback (Step 787)
  rollback.py --reason "pipeline_failed_build_143"
    [*] ใช้ Known-good Config จาก Tag: pre-deploy-a3f9c2e1
    [*] กำลัง Rollback ACCESS-SW1 ...
        [DIFF] ACCESS-SW1: -15 permit ip 10.10.50.0 ...  (ACE ที่ผิดถูกลบออก)
        [OK] ACCESS-SW1: Rollback สำเร็จ
    [*] กำลัง Rollback DIST-SW1 ... [OK]
    ... (Rollback ครบทั้ง 10 อุปกรณ์) ...
  [OK] Rollback สำเร็จทุกอุปกรณ์ กลับสู่ Known-good ที่ Tag pre-deploy-a3f9c2e1

  post_deployment_marker.py --status rollback --tag b7c1d4f2
  [OK] ส่ง Annotation เข้า Grafana สำเร็จ (Tag: deploy, rollback, alert)

========================================================================
BUILD #143: FAILURE (Rollback สำเร็จ) — แจ้งเตือนทีม NetOps ทันที (Total: 5m 12s)
========================================================================
```

### 790.6 บทเรียนจาก Walkthrough ทั้งสอง

| ประเด็น | Build #142 (สำเร็จ) | Build #143 (ล้มเหลว+Rollback) |
|---|---|---|
| Stage ที่ตัดสินผลลัพธ์สุดท้าย | ทุก Stage ผ่าน | **Stage 6 (Verify)** จับปัญหาที่ Stage 1-5 มองไม่เห็น |
| ทำไม Digital Twin (Stage 2) ไม่จับปัญหานี้ | — | Digital Twin ไม่ได้จำลอง ACE เดิมที่มีอยู่ครบ 100% (ตามคำเตือน Step 786.1) |
| ผลกระทบต่อผู้ใช้จริง | ไม่มี — Canary ตรวจแล้วค่อยขยาย | จำกัดอยู่ที่ช่วงเวลาสั้นๆ ก่อน Verify จับได้ และ Rollback ทันที ไม่ใช่ทิ้งไว้ค้างคืน |
| Audit Trail | Build #142, Approver, Diff, ผลลัพธ์ครบใน Jenkins | Build #143, Failure Reason, Rollback Log, Grafana Marker ครบเช่นกัน |

> **ข้อคิดปิด Lab**: Pipeline ไม่ได้ทำให้ "ไม่มีวันเกิดปัญหา" — มันทำให้ **เมื่อเกิดปัญหา ปัญหานั้น
> ถูกจับได้เร็ว ผลกระทบถูกจำกัดด้วย Canary และแก้กลับได้อัตโนมัติโดยไม่ต้องรอ Human ตอนกลางดึก**
> นี่คือความแตกต่างหลักระหว่าง "Automation ที่ทำงานได้" (Part 23-49) กับ "Automation ที่ทำงานได้
> อย่างปลอดภัยในระดับ Production" (Part 79) — ซึ่งคือหัวใจของคำถามระดับ CCIE ที่ Part นี้ตั้งไว้
> ตั้งแต่ต้น

---

## แบบฝึกหัดทวนความเข้าใจ Part 79

1. Ad-hoc Automation จาก Part 23/47/48 ขาดสิ่งใดไปสี่อย่าง ที่ทำให้ไม่เหมาะกับ Production และ
   Network CI/CD Pipeline แก้ปัญหาแต่ละข้ออย่างไร?
2. ทำไมต้องมี Stage "Test on Digital Twin" (Step 784) ก่อน Stage "Deploy" ทั้งที่ Stage "Lint"
   (Step 783) ก็ตรวจ Config แล้ว? ยกตัวอย่างปัญหาที่ Lint จับไม่ได้แต่ pyATS บน Digital Twin จับได้
3. Canary Deployment (Step 786) ต่างจากการ Deploy พร้อมกันทั้ง Fleet (Big-bang) อย่างไร และ
   `serial`/`max_fail_percentage` ใน Ansible ช่วยบังคับ Canary Behavior นี้ได้อย่างไร?
4. ใน Walkthrough Step 790.5 (Build #143) ทำไม Health-check Script (Step 786.4) จึงผ่าน (PASS)
   ทั้งที่ ACL Order ผิดพลาดไปแล้ว และ Stage ไหนที่จับปัญหานี้ได้จริง เพราะอะไร?
5. HashiCorp Vault (Step 789) แก้ข้อจำกัดของ Ansible Vault (Part 48 Step 477) ในเรื่องอะไรบ้าง
   และทำไมการเปลี่ยนไปใช้ Vault จึง**ไม่ต้องแก้ไข** Ansible Playbook ที่มีอยู่แล้วเลย?

**เฉลย:**

1. ขาด (1) Peer Review — แก้ด้วย Stage 4 Manual Approval Gate, (2) Testing ก่อน Deploy — แก้ด้วย
   Stage 2 Digital Twin + pyATS, (3) Audit Trail สมบูรณ์ — แก้ด้วย Pipeline Log ที่ผูก Commit +
   Approver + ผลลัพธ์ทุก Run เข้าด้วยกัน, (4) Rollback Mechanism — แก้ด้วย `rollback.py` ที่ Trigger
   อัตโนมัติจาก `post { failure { ... } }` ใน Jenkinsfile
2. Lint (`ansible-lint`/`yamllint`/Custom Validator) ตรวจได้แค่ Syntax/Style/Pattern ที่รู้ล่วงหน้า
   เท่านั้น (เช่น ACL Sequence ซ้ำ, VLAN ผิดช่วง) แต่ **ไม่รู้ Operational State จริงของอุปกรณ์**
   เช่น OSPF Neighbor จะยัง FULL หรือไม่หลัง Deploy, Interface จะยัง up หรือไม่ — สิ่งเหล่านี้ต้อง
   ทดสอบด้วยการ Deploy จริงบน Digital Twin แล้วใช้ pyATS ตรวจ State จริงเท่านั้นจึงจะรู้ได้
3. Big-bang Deploy ทุกอุปกรณ์พร้อมกันหมายความว่าถ้า Config มีปัญหาที่ Digital Twin จำลองไม่ครบ
   (เช่น ACE เดิมที่ต่างกันในแต่ละอุปกรณ์) ทุกอุปกรณ์จะพังพร้อมกันทันที — Canary Deploy ไปยัง
   อุปกรณ์กลุ่มเล็กที่สุดก่อน (`serial: [1, "50%", "100%"]`) แล้วให้ `max_fail_percentage: 0`
   บังคับให้ Ansible หยุดทันทีถ้า Batch ใดมี Host ล้มเหลวแม้แต่ตัวเดียว ไม่ขยายไป Batch ถัดไป
4. Health-check Script (Step 786.4) ตรวจแค่ Uptime/Interface Status/CPU Usage เท่านั้น — มันไม่ได้
   ตรวจ**เนื้อหา**ของ ACL ว่า Order ถูกหรือผิด ดังนั้น ACE ที่แทรกผิดตำแหน่งจึงไม่กระทบ Metric ที่
   Health-check ดู (CPU/Interface ยังปกติ) — Stage ที่จับปัญหานี้ได้จริงคือ **Stage 6 (Verify)**
   ผ่าน pyATS Testcase `VerifyExplicitDenyStillLast` ที่ Parse เนื้อหา ACL จริงและตรวจ Sequence
   Order โดยเฉพาะ ซึ่งเป็นการตรวจ Semantic ที่ Health-check ทั่วไปไม่ครอบคลุม
5. HashiCorp Vault แก้ (1) Password ตายตัวไม่มีวันหมดอายุ → เปลี่ยนเป็น Dynamic Secret/Lease TTL
   สั้น, (2) ไม่มี Audit Log → มี Audit Log Device บันทึกทุก Request, (3) ไม่มีการแยกสิทธิ์ละเอียด
   ตาม Path → มี Policy (ACL) ผูกกับ AppRole จำกัด Path ที่เข้าถึงได้ — และเพราะ Ansible Playbook
   ยังอ่าน Credential จาก **Environment Variable** (`NET_USERNAME`/`NET_PASSWORD`/`NET_SECRET`)
   เหมือนที่ทำมาตั้งแต่ Part 23 ทุกประการ Vault จึงเปลี่ยนแค่ "แหล่งที่มา" ของค่าเหล่านี้ก่อนที่
   Ansible จะเริ่มรัน โดยไม่ต้องแก้ Logic ภายใน Playbook/Role เลยแม้แต่บรรทัดเดียว

---

## สรุป Part 79

Part นี้ยก Automation Script/Playbook ทั้งหมดที่สร้างมาตลอด Part 23/47/48/49/67 ขึ้นสู่ **Network
CI/CD Pipeline** ระดับ Production จริง — เราเรียนรู้ว่า Git คือ Single Source of Truth เพียงหนึ่ง
เดียว, การเปลี่ยนแปลงทุกครั้งต้องผ่าน Lint → Test บน Digital Twin → Dry-run → Peer Approval →
Canary Deploy → Verify → Rollback-if-failed เสมอไม่มีทางลัด, และ Secret ต้องมาจาก Secrets Manager
ที่มี Audit Log และ Lease TTL แทนไฟล์ Password ตายตัว — ทั้งหมดนี้คือคำตอบของคำถาม **"Automation
รันอย่างไรในโลกจริงที่มีความเสี่ยงสูง โดยไม่มีจุดล้มเหลวเดียวและไม่มีอะไรหลุดเข้า Production โดย
ไม่มีใครตรวจสอบ"** ✅ **พร้อมสำหรับ Part 80**: เราจะเจาะลึก **Model-Driven Telemetry Streaming**
เต็มรูปแบบ — ขยายจาก TIG Stack ของ Part 54 และ Deployment Marker ที่สร้างใน Step 788 ไปสู่
gNMI Dial-out/Dial-in แบบ Production-grade, Subscription Model, และการออกแบบ Telemetry Pipeline
ที่รองรับ Scale ระดับ Enterprise จริง

**ไปต่อ:** [Part 80 — Model-Driven Telemetry Streaming →](part-080-model-driven-telemetry-streaming.md)
