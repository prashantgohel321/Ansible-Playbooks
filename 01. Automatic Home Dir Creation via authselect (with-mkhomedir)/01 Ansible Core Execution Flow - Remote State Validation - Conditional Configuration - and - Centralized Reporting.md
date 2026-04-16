# Day 01 Ansible Core Execution Flow — Remote State Validation, Conditional Configuration, and Centralized Reporting

## Automation starts when we stop thinking in commands and start thinking in system state

When we manage Linux servers manually, the flow is always the same. We SSH into a server, run a command, check output, maybe fix something, then move to the next server. This works fine for one or two machines, but the moment we have 10 or 50 servers, the same process becomes slow, error-prone, and inconsistent.

So instead of thinking “run this command everywhere”, we start thinking “all servers should have this configuration”. This shift is important because Ansible does not focus on commands, it focuses on the final state of the system.

For example, instead of remembering to run:

```bash
authselect enable-feature with-mkhomedir
authselect apply-changes
```

on every server, we define that this feature must be enabled everywhere, and Ansible takes care of checking and applying it.

## How Ansible actually connects and executes internally

When we run Ansible, everything starts from our control machine. This is the system where Ansible is installed.

Suppose we run:

```bash
ansible all -i inventory.ini -m ping
```

What happens internally is not just a simple SSH command. Ansible connects to each server using SSH, then it creates a temporary Python script which represents the module we are using, copies it to the remote server, executes it, captures the result, and deletes it.

If we inspect the remote server during execution, we may notice temporary files in `/tmp` like:

```bash
ls /tmp
ansible-tmp-xxxxx/
```

Inside that directory, Ansible runs its module logic.

This is why Python is required on the target machine. Without Python, Ansible cannot execute modules.

## How we define target machines using inventory

Instead of writing server IPs in commands again and again, we define them in an inventory file.

```ini
[dev_servers]
192.168.1.10
192.168.1.11

[prod_servers]
192.168.1.20
192.168.1.21
```

Now when we run:

```bash
ansible-playbook playbook.yml -e "group=dev_servers"
```

Ansible reads the inventory, finds all machines inside `dev_servers`, and prepares execution for each one.

This makes the same automation reusable across environments.

## Why privilege escalation is required in real scenarios

Many system-level operations require root access. For example, checking authentication configuration or enabling system features.

If we try to run such commands manually as a normal user:

```bash
authselect current
```

we might get permission errors depending on system configuration.

So Ansible uses privilege escalation. When enabled, it internally converts execution into something like:

```bash
sudo authselect current
```

This ensures that tasks run with the required permissions.

## How Ansible gathers system information automatically

Before executing tasks, Ansible can collect system details using fact gathering.

If we run:

```bash
ansible all -m setup
```

we will see a large JSON output containing system information like hostname, IP address, OS version, CPU, memory, etc.

For example:

```json
"ansible_hostname": "server1",
"ansible_default_ipv4": {
    "address": "192.168.1.10"
}
```

This data becomes available automatically during playbook execution, so we can use it without writing extra commands.

## Converting command output into structured data

When we run commands manually, we read output visually. But Ansible does not work like that. It captures output in structured form.

For example, if we run:

```yaml
- name: Check authselect
  command: which authselect
  register: result
```

Internally this becomes:

```bash
which authselect
```

And the output is stored like:

```json
{
  "rc": 0,
  "stdout": "/usr/bin/authselect",
  "stderr": ""
}
```

Now instead of reading output manually, we use this structured data.

If `rc` is 0, command succeeded. If not, it failed.

This is how Ansible makes decisions.

## Handling missing tools in real environments

In real systems, not every server is perfectly configured. Some servers may not have required tools installed.

If we run:

```bash
which authselect
```

On a system without it, we get:

```bash
command not found
```

Instead of failing completely, we handle this situation.

We check the return code, and if it is not 0, we mark that system and skip further steps.

This prevents unnecessary failures and keeps automation running for other servers.

## Understanding idempotency with real behavior

Suppose we run:

```bash
authselect current
```

and we see:

```text
Enabled features:
- with-mkhomedir
```

Now if we run:

```bash
authselect enable-feature with-mkhomedir
```

again, it is unnecessary.

So instead of blindly running commands, we first check the state.

Only if the feature is missing, we run:

```bash
authselect enable-feature with-mkhomedir
```

This ensures that running the same automation multiple times does not change anything unnecessarily.

This concept is called idempotency.

## Applying changes only when required

After enabling a feature, changes may not be active immediately.

We need to run:

```bash
authselect apply-changes
```

But we should not run this blindly. If the previous command failed, applying changes can break the system.

So we only run it when the previous step succeeds.

This creates a dependency between tasks and ensures safe execution.

## Capturing results in a meaningful format

After execution, raw outputs are not useful for reporting. Instead of storing values like true/false, we convert them into readable form.

For example:

```text
already_enabled: YES
enabled_by_playbook: NO
error: authselect not installed
```

If there are commas in error messages, we replace them to avoid breaking CSV format.

```bash
echo "error message, with comma" | sed 's/,/;/g'
```

This ensures clean reporting.

## Writing results to a central location

Instead of saving output on each server, we collect all results and store them in one file on the control machine.

Example CSV file:

```text
hostname,ip,already_enabled,enabled_by_playbook,error
server1,192.168.1.10,YES,NO,
server2,192.168.1.11,NO,YES,
server3,192.168.1.12,NO,NO,authselect not installed
```

This makes it easy to audit and review results across all servers.

## What we are actually building in real terms

If we look at the full flow, we are not just executing commands. We are building a system that connects to multiple servers, checks their current state, decides what needs to be changed, applies only necessary changes, handles failures without stopping everything, and finally generates a clean report.

In manual work, we would run:

```bash
ssh server1
authselect current
authselect enable-feature with-mkhomedir
authselect apply-changes
```

and repeat for every server.

With Ansible, we define this once and let the system handle execution, validation, and reporting automatically.

This is the foundation on which advanced automation is built.
