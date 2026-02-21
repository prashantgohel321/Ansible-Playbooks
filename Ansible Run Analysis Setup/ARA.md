# ARA (Ansible Run Analysis) – Complete Setup Guide

**Centralized Ansible Execution Logs with Web UI (PostgreSQL backend)**

---

- [ARA (Ansible Run Analysis) – Complete Setup Guide](#ara-ansible-run-analysis--complete-setup-guide)
  - [Why This Setup Exists](#why-this-setup-exists)
  - [What This Setup Provides](#what-this-setup-provides)
  - [High-Level Architecture](#high-level-architecture)
  - [System Users Used](#system-users-used)
  - [Prerequisites (Already Present)](#prerequisites-already-present)
  - [Step 1: Install ARA (Client + Server)](#step-1-install-ara-client--server)
    - [Why this works](#why-this-works)
  - [Step 2: Configure PostgreSQL for ARA](#step-2-configure-postgresql-for-ara)
    - [Create Database](#create-database)
    - [Create User](#create-user)
    - [Grant Permissions](#grant-permissions)
    - [Why PostgreSQL](#why-postgresql)
  - [Step 3: Export ARA Environment Variables](#step-3-export-ara-environment-variables)
    - [Meaning of Each Variable](#meaning-of-each-variable)
  - [Step 4: Initialize ARA Database Schema](#step-4-initialize-ara-database-schema)
  - [Step 5: Create ARA Superuser (UI Login)](#step-5-create-ara-superuser-ui-login)
  - [Step 6: Configure Ansible to Use ARA](#step-6-configure-ansible-to-use-ara)
    - [What a Callback Plugin Is](#what-a-callback-plugin-is)
  - [Step 7: Configure ARA Server Settings](#step-7-configure-ara-server-settings)
    - [Why This Is Required](#why-this-is-required)
  - [Step 8: Start ARA API (Manual Test)](#step-8-start-ara-api-manual-test)
  - [Step 9: Convert ARA into Background Service (Production)](#step-9-convert-ara-into-background-service-production)
    - [Service File](#service-file)
    - [Unit file: `/etc/systemd/system/ara.service`](#unit-file-etcsystemdsystemaraservice)
    - [`[Unit]` — service identity and ordering](#unit--service-identity-and-ordering)
    - [`[Service]` — how the process runs](#service--how-the-process-runs)
    - [Environment configuration (critical section)](#environment-configuration-critical-section)
    - [Process startup](#process-startup)
    - [Reliability and restarts](#reliability-and-restarts)
    - [`[Install]` — boot-time behavior](#install--boot-time-behavior)
    - [How to activate this service (required steps)](#how-to-activate-this-service-required-steps)
    - [One-line summary](#one-line-summary)
  - [Step 10: Verify End-to-End Logging](#step-10-verify-end-to-end-logging)
  - [What Gets Stored in ARA](#what-gets-stored-in-ara)
  - [Why This Setup Is Safe](#why-this-setup-is-safe)
  - [Common Commands (Reference)](#common-commands-reference)
  - [Final Result](#final-result)


<br>
<br>

## Why This Setup Exists

When working with Ansible in real environments, one recurring problem appears:

* Playbook output scrolls past the terminal and is lost
* Failures and unreachable hosts are hard to track later
* No historical record of:
  * What ran
  * When it ran
  * On which servers
  * What failed and why

**ARA (Ansible Run Analysis)** solves this problem by recording every Ansible execution into a database and exposing it through a web UI.

This setup converts Ansible from *fire-and-forget CLI output* into a **searchable, permanent execution history**.

---

<br>
<br>

## What This Setup Provides

* Automatic recording of **every Ansible playbook run**
* Web UI accessible from a browser
* PostgreSQL backend (no SQLite)
* No changes required inside playbooks
* Works with mixed environments (Rocky, RHEL, Ubuntu)
* Runs as a background service (no manual startup)

---

<br>
<br>

## High-Level Architecture

```bash
Ansible Playbook Execution
        |
        v
ARA Callback Plugin
        |
        v
ARA API Server (Django)
        |
        v
PostgreSQL Database
        |
        v
Web UI (Browser)
```

---

<br>
<br>

## System Users Used

Two different Linux users are intentionally used:

| User      | Purpose                               |
| --------- | ------------------------------------- |
| `pginsta` | PostgreSQL installation and ownership |
| `ansible` | All Ansible + ARA operations          |

This separation avoids security and permission issues.

---

<br>
<br>

## Prerequisites (Already Present)

* Python 3
* Ansible installed and working
* PostgreSQL running locally
* Inventory files already in use
* SSH connectivity from ansible controller

---

<br>
<br>

## Step 1: Install ARA (Client + Server)

ARA is installed **only for the `ansible` user**, not system-wide.

```bash
pip install --user "ara[server]" psycopg2-binary
```

### Why this works

* `ara[server]` installs:

  * ARA CLI
  * ARA API server
  * Django dependencies
* `psycopg2-binary` allows Python to talk to PostgreSQL
* `--user` keeps everything inside `~/.local` (safe, clean, reversible)

Verify installation:

```bash
which ara
which ara-manage
```

**Expected output:**

```
~/.local/bin/ara
~/.local/bin/ara-manage
```

---

<br>
<br>

## Step 2: Configure PostgreSQL for ARA

### Create Database

```sql
CREATE DATABASE ara;
```

### Create User

```sql
CREATE USER ara_user WITH PASSWORD 'StrongPassword';
```

### Grant Permissions

```sql
GRANT ALL PRIVILEGES ON DATABASE ara TO ara_user;
```

### Why PostgreSQL

* Persistent storage
* Handles large execution history
* Better concurrency than SQLite
* Already present in your environment

---

<br>
<br>

## Step 3: Export ARA Environment Variables

These variables tell ARA **where and how to store data**.

Run **as ansible user**:

```bash
export ARA_DATABASE=postgresql+psycopg2://ara_user:StrongPassword@localhost:5432/ara
export ARA_API_CLIENT=http
export ARA_API_SERVER=http://127.0.0.1:8000
```

### Meaning of Each Variable

| Variable         | Explanation                |
| ---------------- | -------------------------- |
| `ARA_DATABASE`   | Database connection string |
| `ARA_API_CLIENT` | How Ansible sends data     |
| `ARA_API_SERVER` | Where ARA API listens      |

---

<br>
<br>

## Step 4: Initialize ARA Database Schema

Run migrations (creates tables):

```bash
ara-manage migrate
```

<details>
<summary><b><code>Command Breakdown</code></b></summary>
<br>

**Command:**
`ara-manage migrate`

This command applies **database schema migrations** for **ARA**, ensuring the database structure matches the version of ARA you are running.

* **`ara-manage`** is ARA’s administrative management utility, used for database and internal maintenance tasks.
* **`migrate`** runs the database migration engine, which creates new tables or updates existing ones when ARA is installed or upgraded (a *migration* is a controlled change to database structure).

**What happens internally:**
ARA checks the configured database (defined by `ARA_DATABASE`) for its current schema version, compares it with the required schema for the installed ARA version, and then applies any missing changes in the correct order.

**When this is required:**

* After **installing ARA for the first time** with a non-empty database.
* After **upgrading ARA** to a newer version.
* When switching from SQLite to PostgreSQL or initializing a fresh PostgreSQL backend.

**Operational notes:**

* The database must already exist and be reachable.
* The configured database user must have permissions to create and alter tables.
* The command is safe to run multiple times; migrations are applied only once.

In short, `ara-manage migrate` **prepares and keeps the ARA database schema in sync with the ARA application**, making sure the web UI and callback plugin can store and read playbook run data correctly.


</details>
<br>

This creates all required tables inside PostgreSQL.

---

<br>
<br>

## Step 5: Create ARA Superuser (UI Login)

```bash
ara-manage createsuperuser
```

Example:

```
Username: linuxadmin
Email: admin@example.com
Password: ********
```

This user is used to log into the UI.

---

<br>
<br>

## Step 6: Configure Ansible to Use ARA

Edit **/etc/ansible/ansible.cfg**:

```ini
[defaults]
callback_whitelist = ara
```

Also ensure callback plugin path is visible:

```bash
ansible-config dump | grep -i ara
```

Expected:

```
CALLBACKS_ENABLED = ['ara']
DEFAULT_CALLBACK_PLUGIN_PATH = ~/.local/lib/python3*/site-packages/ara/plugins/callback
```

### What a Callback Plugin Is

A callback plugin hooks into Ansible’s execution lifecycle.

ARA’s callback captures:
* Playbook start
* Task start
* Task result
* Failures
* Summary

No playbook changes are required.

---

<br>
<br>

## Step 7: Configure ARA Server Settings

File location (auto-created):

```bash
~/.ara/server/settings.yaml
```

Edit and ensure this line exists:

```yaml
ALLOWED_HOSTS:
  - 127.0.0.1
  - localhost
  - 192.168.107.114
```

### Why This Is Required

Django blocks external requests unless explicitly allowed.

---

<br>
<br>

## Step 8: Start ARA API (Manual Test)

```bash
ara-manage runserver 0.0.0.0:8000
```

<details>
<summary><b><code>Command Breakdown</code></b></summary>
<br>

**Command:**
`ara-manage runserver 0.0.0.0:8000`

This command starts the **ARA web application server** and makes it accessible over the network.

* **`ara-manage`** is ARA’s administrative control utility, used to manage and operate the ARA application.
* **`runserver`** launches ARA’s built-in web server, which serves the web UI used to browse and analyze Ansible playbook runs.
* **`0.0.0.0`** means the server listens on **all network interfaces**, not just localhost (this allows access from other machines).
* **`8000`** is the TCP port on which the web interface is exposed.

**What happens internally:**
The command starts the ARA application using its configured database backend (defined by `ARA_DATABASE`) and exposes HTTP endpoints for viewing playbook execution data collected by the Ansible callback plugin.

**Operational notes:**

* This built-in server is intended mainly for **development, testing, or internal use**.
* For production setups, ARA is typically run behind a proper web server (such as Gunicorn with Nginx).
* Accessing `http://<server-ip>:8000` in a browser opens the ARA UI.

In short, this command **starts the ARA web UI and makes it reachable on port 8000 from any network interface**, allowing centralized visibility into Ansible automation runs.


</details>
<br>

Verify in browser:

```
http://192.168.107.114:8000
```

You should see the ARA dashboard.

---

<br>
<br>

## Step 9: Convert ARA into Background Service (Production)

Create systemd service:

```bash
sudo vi /etc/systemd/system/ara.service
```

### Service File

```ini
[Unit]
Description=ARA Ansible Records
After=network.target postgresql.service

[Service]
Type=simple
User=ansible
Group=ansible
WorkingDirectory=/datadisk/home/ansible

Environment=ARA_DATABASE=postgresql+psycopg2://ara_user:StrongPassword@localhost:5432/ara
Environment=ARA_API_CLIENT=http
Environment=ARA_API_SERVER=http://127.0.0.1:8000

ExecStart=/datadisk/home/ansible/.local/bin/ara-manage runserver 0.0.0.0:8000

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

<details>
<summary><b><code>Service File Explanation</code></b></summary>
<br>

### Unit file: `/etc/systemd/system/ara.service`

This file tells **systemd** how to start, stop, and supervise the **ARA** web application.

---

### `[Unit]` — service identity and ordering

```ini
[Unit]
Description=ARA Ansible Records
After=network.target postgresql.service
```

* **`Description`** is a human-readable name shown in `systemctl status`.
* **`After=network.target postgresql.service`** ensures the service starts **only after networking is up and PostgreSQL is started**, which is required because ARA depends on a reachable database.

---

### `[Service]` — how the process runs

```ini
[Service]
Type=simple
```

* **`Type=simple`** tells systemd that the process started by `ExecStart` runs in the foreground and does not fork (this matches how `ara-manage runserver` behaves).

```ini
User=ansible
Group=ansible
```

* The service runs as the **`ansible` user and group**, not as root, which is the correct security practice for application services.

```ini
WorkingDirectory=/datadisk/home/ansible
```

* Sets the working directory for the process.
* This matters for relative paths and for predictable runtime behavior.

---

### Environment configuration (critical section)

```ini
Environment=ARA_DATABASE=postgresql+psycopg2://ara_user:StrongPassword@localhost:5432/ara
```

* Defines the **database connection URL** for ARA.
* This points ARA to a **PostgreSQL** backend instead of SQLite.

```ini
Environment=ARA_API_CLIENT=http
```

* Tells the ARA Ansible callback plugin how clients should talk to the API.

```ini
Environment=ARA_API_SERVER=http://127.0.0.1:8000
```

* Defines where the ARA API server is reachable locally.
* Callbacks use this to submit playbook run data.

---

### Process startup

```ini
ExecStart=/datadisk/home/ansible/.local/bin/ara-manage runserver 0.0.0.0:8000
```

* Launches the ARA web server.
* **`0.0.0.0:8000`** binds the service to **all network interfaces**, making the UI accessible remotely.
* Uses the user-local Python installation instead of a system-wide binary.

---

### Reliability and restarts

```ini
Restart=always
RestartSec=5
```

* **`Restart=always`** automatically restarts ARA if it crashes or exits.
* **`RestartSec=5`** waits 5 seconds before restarting, preventing rapid restart loops.

---

### `[Install]` — boot-time behavior

```ini
[Install]
WantedBy=multi-user.target
```

* This enables the service to start automatically in **multi-user mode**, which is the normal server run level.

---

### How to activate this service (required steps)

After saving the file:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ara.service
sudo systemctl start ara.service
```

To verify:

```bash
systemctl status ara.service
```

---

### One-line summary

This unit file runs **ARA as a resilient, non-root, systemd-managed web service**, backed by PostgreSQL, automatically restarted on failure, and started at boot—exactly how a production-grade ARA deployment should be configured.


</details>
<br>

Enable and start:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable ara
sudo systemctl start ara
```

<details>
<summary><b><code>Command Breadown</code></b></summary>
<br>

**Command:**
`sudo systemctl daemon-reexec`

This command **re-executes the systemd manager process itself** while keeping the system running.

* **`sudo`** runs the operation with superuser privileges, which are required to control the init system.
* **`systemctl`** is the control interface for **systemd**, the service manager used by most modern Linux distributions.
* **`daemon-reexec`** instructs systemd to **replace its own process image** with a fresh instance of the same version of systemd, reloading its internal state from disk.

**What actually happens internally:**
Systemd performs a self-restart without stopping services. All running units (services, sockets, timers) continue uninterrupted, but the systemd process itself is reloaded, re-reading its binaries and internal logic.

**When this is used:**

* After **updating the systemd package** to ensure the running PID 1 uses the new binary.
* When systemd behavior appears inconsistent and a full reboot is undesirable.

**Important distinction:**

* `daemon-reexec` reloads **systemd itself**.
* `daemon-reload` reloads **unit files** (`.service`, `.timer`, etc.).
  They solve different problems and are not interchangeable.

In short, `systemctl daemon-reexec` **safely refreshes the running systemd process without rebooting or restarting services**, ensuring the active init system matches the installed version.


</details>
<br>

Verify:

```bash
systemctl status ara
```

---

## Step 10: Verify End-to-End Logging

Run a simple playbook:

```bash
ansible-playbook -i inventory/hosts_test.ini playbooks/test_ara.yml
```

Refresh UI:

```
http://192.168.107.114:8000
```

You will see:

* Playbook listed
* Hosts involved
* Failed/unreachable nodes
* Task-level details

---

## What Gets Stored in ARA

* Playbook name
* Execution time
* User running Ansible
* Inventory hosts
* Task results
* Errors and stack traces
* Unreachable reasons

This data persists across reboots.

---

## Why This Setup Is Safe

* No changes on target servers
* Read-only logging
* No privilege escalation
* Runs under non-root user
* Uses standard Ansible hooks
* Fully reversible

---

## Common Commands (Reference)

| Purpose           | Command                  |
| ----------------- | ------------------------ |
| Start ARA service | `systemctl start ara`    |
| Stop ARA service  | `systemctl stop ara`     |
| Check status      | `systemctl status ara`   |
| List playbooks    | `ara playbook list`      |
| View playbook     | `ara playbook show <id>` |

---

## Final Result

You now have:

* Centralized Ansible execution history
* Searchable UI
* Permanent audit trail
* Production-grade logging
* Zero manual steps during daily usage

This setup mirrors how **real infrastructure teams** track Ansible runs.

---

If you want next:

* HTTPS + Nginx
* Multi-controller support
* Export reports
* Tag-based filtering
* Role-level metrics

Say the word.
