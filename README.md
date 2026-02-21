# Production-Ready Ansible Playbooks

Managing Linux servers at scale can be complex, but it doesn't have to be a headache. This repository is a collection of structured, "production-style" Ansible playbooks designed for real-world server environments.

Built to solve actual operational challenges—like hardening security, managing patches across different distros, and keeping your infrastructure audits clean.

### What's the goal?
It's simple: Provide safe, reusable automation workflows that you can drop into an enterprise environment and run with confidence. 

---

<br>
<br>

## What can you do with this?

This isn't just a list of syntax examples. These playbooks handle the "daily grind" of sysadmin and DevOps work:

*   **Security & Hardening:** Lock down `sshd`, manage `sudoers`, apply PAM templates, and set file immutability.
*   **Identity & Access:** Automate Active Directory joins and distribute SSH keys safely.
*   **Visibility & Monitoring:** Deploy Zabbix agents, audit active checks, and collect live hardware/OS facts.
*   **Lifecycle Management:** Generate Ubuntu patch reports and perform controlled upgrades.
*   **Infrastructure Management:** Sync Foreman `/etc/hosts`, distribute custom scripts, and handle RPM deployments.
*   **New Server Setup:** Run a full bootstrap that handles disk provisioning (LVM), AD integration, and security hardening in one go.

---

<br>
<br>

## The "Safe-to-Run" Philosophy

I follow a few strict principles to make sure these playbooks are safe for production:

*   **Idempotency is king:** If the system is already in the right state, the playbook won't touch it.
*   **No "ghost" changes:** Clear logging and controller-side reporting so you know exactly what changed and where.
*   **Safety Guards:** Built-in OS checks and existence checks mean the playbook stops before it can break anything.
*   **Inspection First:** I prioritize "reporting mode" so you can audit your systems before you enforce a new configuration.

---

<br>
<br>

## A Tour of the Playbooks

### 01–04: Authentication & Security
*   **Authselect & mkhomedir:** Enforces home directory creation for domain users and generates a compliance report.
*   **SSHD & Access Hardening:** Applies PAM/sudoers templates and cleans up insecure defaults.
*   **Revert Access:** A "safety net" playbook to roll back hardening changes using backups if needed.

### 05–08: Visibility & Monitoring
*   **Zabbix Agent 2:** Standardizes agent deployment across Ubuntu and RHEL safely.
*   **Foreman /etc/hosts Sync:** Manages a specific block in your hosts file without touching the rest of the system.
*   **Live Fact Aggregation:** Grabs CPU, RAM, and disk details and dumps them into a clean JSON file for you.
*   **Active Directory Audit:** Checks domain membership and join status across your fleet.

### 09–12: Patching & Package Management
*   **Ubuntu Patch Reports:** Simulates upgrades to show you what's pending without actually changing anything.
*   **Controlled Upgrades:** Performs production-safe package upgrades with transactional logging.
*   **Remote Distribution:** A clean way to copy, run, and clean up scripts or custom RPMs across multiple nodes.

### 13–15: System Hardening & Maintenance
*   **File Immutability:** Uses `chattr +i` to lock down critical system binaries.
*   **SELinux Fixes:** Automatically corrects context labels for custom home directories to prevent login failures.
*   **Zabbix Config Audit:** Scans your fleet for missing `ServerActive` settings.

### 16: The Master Bootstrap
A comprehensive "Day 1" playbook for new servers. It handles:
*   Disk partitioning (fdisk) and LVM setup.
*   Active Directory / SSSD integration.
*   Security hardening (PAM, Authselect, SELinux).
*   Full structured logging for the entire setup process.

---

<br>
<br>

## Getting Started

### What you'll need
*   A Linux control node (Ubuntu or RHEL preferred).
*   SSH access and sudo privileges on your target servers.
*   Ansible (2.9+) and Python 3.

### Quick Install

**RHEL / Rocky / Fedora:**
```bash
sudo dnf install -y ansible python3 sshpass
```

**Ubuntu:**
```bash
sudo apt update && sudo apt install -y ansible python3 sshpass
```

### How to run them

To run a specific task (like enabling `mkhomedir`):
```bash
ansible-playbook playbooks/authselect_mkhomedir.yml -e "group=linux_servers"
```

To bootstrap a brand new server with a data disk:
```bash
ansible-playbook server_setup.yml -e "setup_datadisk=true disk_name=sdb"
```

---

<br>
<br>

## Design Approach
I don't believe in "black box" automation. Every playbook here follows a logical flow:
1.  **Validate** prerequisites.
2.  **Inspect** the current state.
3.  **Change** only what's necessary.
4.  **Log** everything centrally.

