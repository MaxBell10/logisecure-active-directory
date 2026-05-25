# P1 — Active Directory & SIEM Foundation

<img src="./p1_logo.svg" alt="P1 Logo" width="100%"/>

> **LogiSecure SA** · Enterprise Security Programme
> Domain : `lab.local` · DC01 : `10.10.10.10` · WKS01 : `10.10.10.20` · Wazuh : `10.10.10.30`

## Objective

Build the identity and detection infrastructure of a fictional Belgian automated logistics company (**LogiSecure SA**) subject to NIS2, ISO 27001, and IEC 62443, by simulating a complete Active Directory environment with SIEM monitoring via Wazuh.

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                  VirtualBox — lan-network            │
│                   (10.10.10.0/24)                    │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ DC01        │  │ WKS01       │  │ Wazuh OVA   │  │
│  │ Win Srv 2022│  │ Win 10 Pro  │  │ v4.14.5     │  │
│  │ 10.10.10.10 │  │ 10.10.10.20 │  │ 10.10.10.30 │  │
│  │ AD DS · DNS │  │ Domain      │  │ SIEM · XDR  │  │
│  │ GPO · DC    │  │ Member      │  │ Dashboard   │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────┘
        │ Adapter 1 NAT (internet)
        │ Adapter 2 lan-network (isolated)
```

| VM | OS | IP | Role |
|---|---|---|---|
| LOGISECURE-DC01 | Windows Server 2022 | 10.10.10.10 | Domain Controller, DNS, GPO |
| LOGISECURE-WKS01 | Windows 10 Pro | 10.10.10.20 | Domain-joined workstation |
| LOGISECURE-WAZUH | Ubuntu (OVA) | 10.10.10.30 | Wazuh Manager + Dashboard |

---

## What Was Built

### 1. Active Directory — Domain `lab.local`

Complete OU structure simulating a real company:

```
lab.local
└── LogiSecure/
    ├── Users/          (5 business users)
    ├── Admins/         (it.admin — Domain Admin)
    ├── Workstations/   (LOGISECURE-WKS01)
    ├── Servers/
    └── Service Accounts/
```

**Users created:**

| Name | Login | Title |
|---|---|---|
| Sophie Lebrun | s.lebrun | Logistics Operator |
| Thomas Renard | t.renard | WMS Operator |
| Amina Hakimi | a.hakimi | IT Support |
| Jean-Pierre Morel | jp.morel | Operations Manager |
| Clara Devos | c.devos | Finance Analyst |
| IT Admin | it.admin | Domain Admin |

![OU Structure](screenshots/02_ou-users/11_DC01_OU_Structure.png)

---

### 2. GPO Hardening — 4 Active Policies

| GPO | Key Setting | Value | Framework |
|---|---|---|---|
| Account Lockout Policy | Lockout threshold | 5 attempts | CIS / MITRE T1110 |
| Account Lockout Policy | Lockout duration | 30 minutes | CIS Benchmark |
| Password Policy | Minimum length | 12 characters | CIS + NIS2 |
| Password Policy | Complexity | Enabled | CIS Benchmark |
| Audit Policy | Logon events | Success + Failure | MITRE T1078, T1110 |
| Security Hardening | NTLMv2 only | LmCompatibilityLevel=5 | CIS Benchmark |
| Security Hardening | SMB Signing | Enabled | CIS Benchmark |
| Security Hardening | LLMNR disabled | Enabled | MITRE T1557 |

![GPO Account Lockout](screenshots/03_gpo-hardening/14_DC01_GPO_AccountLockout.png)

---

### 3. Wazuh SIEM — 2 Active Agents

Wazuh OVA v4.14.5 deployed with SSH access from the host via NAT port forwarding (2222:22). Dashboard accessible exclusively from the internal `lan-network` — consistent with a zero-trust architecture.

![Wazuh Dashboard](screenshots/04_wazuh-setup/25_Wazuh_Dashboard_Home.png)

**Deployed agents:**

| Agent | IP | OS | Status |
|---|---|---|---|
| DC01 | 10.10.10.10 | Windows Server 2022 | ● active |
| WKS01 | 10.10.10.20 | Windows 10 Pro | ● active |

![Wazuh Agents](screenshots/04_wazuh-setup/34_Wazuh_Agents_Both_Active.png)

---

### 4. Custom MITRE ATT&CK Rules

3 LogiSecure rules created in `/var/ossec/etc/rules/logisecure_rules.xml`:

| Rule ID | Technique | Description | Level |
|---|---|---|---|
| 100001 | T1110 — Brute Force | Failed Windows login (Event 4625) | 10 — High |
| 100002 | T1078 — Valid Accounts | Privileged account logon (Event 4672) | 8 — Medium |
| 100003 | T1087 — Account Discovery | Account/group enumeration (Events 4798, 4799) | 8 — Medium |

![MITRE Rules](screenshots/05_wazuh-rules/35_Wazuh_MITRE_Rules_LogiSecure.png)

---

### 5. PingCastle — AD Security Assessment

**Baseline Scan (2026-05-24):**

![PingCastle Baseline](screenshots/06_pingcastle/20a_DC01_PingCastle_Score.png)

**After Hardening Scan (2026-05-25):**

![PingCastle After](screenshots/06_pingcastle/38_PingCastle_After_Score.png)

**Comparative results:**

| Category | Baseline | After | Delta |
|---|---|---|---|
| Stale Objects | 41/100 | 36/100 | **-5** ✅ |
| Privileged Accounts | 50/100 | 40/100 | **-10** ✅ |
| Trusts | 0/100 | 0/100 | 0 (perfect) |
| Anomalies | 55/100 | 55/100 | 0 (blocks global score) |
| **Domain Risk Level** | **55/100** | **55/100** | blocked by Anomalies |

> Residual anomalies (pass-the-credential, network sniffing) require an internal PKI and advanced configurations beyond P1 scope. Improvements on Privileged Accounts (-10) and Stale Objects (-5) reflect realistic and documented hardening.

---

## Tech Stack

![Windows Server 2022](https://img.shields.io/badge/Windows_Server_2022-0078D4?style=flat&logo=windows&logoColor=white)
![Windows 10](https://img.shields.io/badge/Windows_10_Pro-0078D4?style=flat&logo=windows&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh_4.14.5-00A1E0?style=flat&logo=wazuh&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox_7.x-183A61?style=flat&logo=virtualbox&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=flat&logo=powershell&logoColor=white)

| Tool | Version | Usage |
|---|---|---|
| VirtualBox | 7.x | Hypervisor |
| Windows Server 2022 | Standard Evaluation | Domain Controller |
| Windows 10 Pro | 22H2 | Client workstation |
| Wazuh OVA | 4.14.5 | SIEM / XDR |
| PingCastle | 3.5.1.31 | AD security audit |
| PowerShell | 5.1+ | AD & GPO administration |

---

## Repository Structure

```
logisecure-active-directory/
├── README.md
├── p1_logo.svg
├── lessons_learned.md
└── screenshots/
    ├── 00_lab-setup/          # VirtualBox network config, static IPs
    ├── 01_ad-installation/    # AD DS installation, DC promotion
    ├── 02_ou-users/           # OU structure, users, it.admin
    ├── 03_gpo-hardening/      # 4 active GPOs
    ├── 04_wazuh-setup/        # Wazuh OVA, DC01 and WKS01 agents
    ├── 05_wazuh-rules/        # Custom MITRE ATT&CK rules
    └── 06_pingcastle/         # Baseline and after hardening scans
```

---

## Status

| # | Step | Status |
|---|---|---|
| 1 | DC01 IP config + rename | ✅ Done |
| 2 | AD DS installation + DC promotion | ✅ Done |
| 3 | OU structure + users | ✅ Done |
| 4 | GPO hardening (4 GPOs) | ✅ Done |
| 5 | PingCastle baseline (55/100) | ✅ Done |
| 6 | Wazuh OVA — network config + dashboard | ✅ Done |
| 7 | Wazuh agent on DC01 | ✅ Done |
| 8 | WKS01 — IP config + domain join | ✅ Done |
| 9 | Wazuh agent on WKS01 | ✅ Done |
| 10 | MITRE ATT&CK Wazuh rules | ✅ Done |
| 11 | PingCastle after hardening | ✅ Done |
