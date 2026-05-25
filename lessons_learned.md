# Lessons Learned — P1 Active Directory & SIEM Foundation

## Overview

This document reflects my honest experience building P1 of the LogiSecure SA portfolio from scratch. The goal is not to pretend everything went smoothly, but to document what I actually learned — including the friction points, dead ends, and unexpected discoveries. This lab was built on VirtualBox 7.x using Windows Server 2022, Windows 10 Pro, and Wazuh OVA v4.14.5.

---

## Main Challenges

### Navigating Windows Server 2022 for the first time

The biggest initial challenge was simply working within Windows Server 2022. Server Manager, the AD DS role installation wizard, promoting the server to a Domain Controller, and understanding the difference between local and domain policies — none of this was intuitive at the start.

Understanding the relationship between the DC, the domain, and the client machine took time — particularly why certain settings apply at the domain level via GPO rather than locally on each machine, and why the DNS server must point to itself (`127.0.0.1`) before AD DS promotion.

One concrete example: the French Windows Server uses **"Admins du domaine"** instead of **"Domain Admins"** for the group name in PowerShell commands. This caused a first failed attempt when trying to add `it.admin` to the group.

This is exactly the kind of hands-on familiarity that no course can fully replace.

---

### Wazuh OVA — keyboard layout and console access

The Wazuh OVA boots into a Linux console inside VirtualBox with a **QWERTY keyboard layout**, while the physical machine uses **AZERTY**. This made direct console interaction extremely painful — every special character required mental remapping, and commands with symbols like `{`, `/`, or `|` were nearly impossible to type reliably.

**Fix applied:** Instead of fighting the console, a **port forwarding rule** was added to the NAT adapter (host port 2222 → guest port 22) before first boot. This allowed SSH access from the Windows host via PowerShell:

```powershell
ssh -p 2222 wazuh-user@127.0.0.1
```

From that point, all configuration was done over SSH with full AZERTY support and copy-paste. This turned out to be a significantly better workflow than the VirtualBox console for any Linux VM.

**Production difference:** In a real environment, console access to security infrastructure is typically reserved for emergency break-glass access. SSH (or a dedicated management plane) is the standard operational interface.

---

### Wazuh OVA — no netplan, wrong network stack

The OVA uses **Amazon Linux / RHEL-based networking** (`/etc/sysconfig/network-scripts/ifcfg-*`), not Ubuntu's netplan. The first instinct was to look for `/etc/netplan/` and `/etc/network/` — neither existed. This caused a failed reimport of the OVA before understanding the underlying OS.

**Fix applied:** Once the correct path was identified, the static IP was configured via:

```bash
sudo tee /etc/sysconfig/network-scripts/ifcfg-eth0 << 'EOF'
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
TYPE=Ethernet
IPADDR=10.10.10.30
PREFIX=24
DNS1=10.10.10.10
USERCTL=yes
EOF
sudo systemctl restart network
```

**Lesson:** Never assume the network stack of a Linux VM without checking first. `ls /etc/netplan/`, `ls /etc/network/`, and `ls /etc/sysconfig/network-scripts/` are the three common paths depending on the distro.

---

### Wazuh admin password — no install file on OVA

The standard Wazuh documentation says to recover the admin password from `wazuh-install-files.tar`. That file **does not exist** on the pre-built OVA. The password hash in `internal_users.yml` is set at build time and not recoverable through that method.

**Fix applied:** The password was reset manually in three steps:

1. Generate a new bcrypt hash using the bundled Java:
```bash
sudo JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh \
  -p LogiSecure2024!
```

2. Replace the old hash in `internal_users.yml` using `sed`:
```bash
sudo sed -i 's|[old hash]|[new hash]|' \
  /etc/wazuh-indexer/opensearch-security/internal_users.yml
```

3. Apply the configuration live via `securityadmin.sh` without restarting the service:
```bash
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
sudo -E /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -f /etc/wazuh-indexer/opensearch-security/internal_users.yml \
  -icl -nhnv \
  -cacert /etc/wazuh-indexer/certs/root-ca.pem \
  -cert /etc/wazuh-indexer/certs/admin.pem \
  -key /etc/wazuh-indexer/certs/admin-key.pem
```

**Why it matters:** Understanding OpenSearch security internals — and the fact that `securityadmin.sh` pushes changes live to the cluster without a restart — is directly applicable to any Wazuh or OpenSearch production environment.

---

### Wazuh dashboard access — network isolation by design

The initial assumption was that the Wazuh dashboard (`https://10.10.10.30`) should be accessible from the VirtualBox host machine. It is not — and intentionally so.

The `lan-network` internal network in VirtualBox is isolated from the host. Only VMs on that network can reach each other. The dashboard is only accessible from **DC01** or **WKS01** inside the lab — which is the correct architecture for a SIEM in a real enterprise (management plane isolated from the host network).

This was initially perceived as a bug; it is actually a feature.

---

### WKS01 domain join — NetBIOS name truncation

When running:

```powershell
Add-Computer -DomainName "lab.local" -NewName "LOGISECURE-WKS01" -Credential lab\it.admin -Restart
```

PowerShell warned that the NetBIOS name would be truncated to `LOGISECURE-WKS0` (15-character limit). The machine joined the domain successfully but the rename failed on first attempt because the Directory Service was busy during the join operation.

**Fix applied:** Rename was done separately after reboot:

```powershell
Rename-Computer -NewName "LOGISECURE-WKS01" -DomainCredential lab\it.admin -Restart
```

**Lesson:** The NetBIOS 15-character limit is a legacy constraint. In AD, the full DNS name (`LOGISECURE-WKS01.lab.local`) is what matters for modern authentication. The truncated NetBIOS name has no functional impact in this lab.

---

### PingCastle score — anomalies block global improvement

After applying all hardening corrections (MachineAccountQuota=0, Recycle Bin, AES256 on all accounts, NTLMv2 enforcement, LDAP signing, it.admin marked as sensitive), the **Domain Risk Level remained at 55/100**.

The reason: PingCastle takes the **maximum** of the four category scores. The Anomalies category (55/100) was not improved by these corrections — it requires configurations beyond the scope of P1 such as a PKI for LDAP over SSL, elimination of pass-the-credential vectors, and disabling legacy protocols at the OS level.

**What did improve:**
- Privileged Accounts: 50 → 40 (-10 pts)
- Stale Objects: 41 → 36 (-5 pts)

**Lesson:** A security score is not always a linear reflection of effort. Understanding *why* a score does not move — and being able to explain it — is more valuable than chasing the number itself.

---

## Positive Surprises

### Active Directory structure is more approachable than expected

Creating the OU hierarchy, adding users, assigning groups, and linking GPOs was more straightforward than anticipated once the underlying logic was clear. The separation — OUs for organisation, groups for permissions, GPOs for enforcement — is clean and well-designed.

### Wazuh out-of-the-box detection capability

Without any custom rule configuration, Wazuh already correlated security events across the domain:
- Individual logon failures (rule 60122, level 5)
- Brute-force pattern detection (rule 60204, level 10)
- Automatic MITRE ATT&CK mapping (T1110, T1078)

The depth of visibility available from a free, open-source SIEM on a home lab was genuinely surprising.

### SSH port forwarding as a workflow pattern

The decision to use SSH port forwarding instead of the VirtualBox console for Wazuh turned out to be the right call not just for keyboard reasons — it also enables scripting, copy-paste, and proper terminal behavior. This pattern (NAT + port forwarding for management access) is directly reusable for any future Linux VM in the lab.

---

## What I Would Do Differently in Production

I am currently in a learning phase — theoretical and lab-based — with the goal of moving into a hands-on GRC + Technical role. I do not yet have production experience, and I think it is important to say that clearly rather than pretend otherwise.

That said, based on what I have learned building P1:

- **Wazuh would sit on a dedicated management VLAN**, fully isolated from monitored endpoints, with firewall rules restricting who can reach the dashboard and the manager port (1514/1515)
- **Agent enrollment would use certificate-based authentication** rather than the default shared key, to ensure integrity of agent-manager communication
- **GPOs would be more granular** — separate policies per OU rather than domain-root linked, allowing targeted enforcement and easier rollback
- **The Wazuh admin password would never be reset via `sed`** on a production system — a proper secrets management tool (HashiCorp Vault, AWS Secrets Manager) would handle credential rotation
- **PingCastle scans would run on a schedule** (monthly or after any AD change) as part of a continuous compliance posture, not just as point-in-time baseline/after snapshots
- **LDAP signing and channel binding** would be enforced from day one — in this lab they were added as a hardening step, but in production they should be baseline requirements

---

## Next Steps

- Deploy pfSense for network segmentation (P2) — DMZ, VLAN isolation, Suricata IDS/IPS
- Build a custom Wazuh rule for Event ID 4728 (user added to Domain Admins group)
- Add a custom rule for Event ID 4720 (user account created) for insider threat detection
- Configure Wazuh File Integrity Monitoring (FIM) on DC01 for SYSVOL and critical registry keys
- Run a simulated password spray attack to validate the T1110 detection rule in a controlled scenario

---

*Part of the [LogiSecure SA Enterprise Security Programme](https://github.com/MaxBell10/logisecure-enterprise-security-program) — a 16-project cybersecurity portfolio.*

---


