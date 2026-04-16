# Ensuring mkhomedir Feature with authselect — Deep Understanding Through Execution Flow

## What we are trying to achieve

We are working in systems where users may come from external authentication sources like LDAP or SSSD, and when such a user logs in for the first time, the system should automatically create a home directory. This behavior is controlled through authselect using a feature called `with-mkhomedir`, where the system ensures that `/home/username` gets created at first login. The goal here is not just to enable this feature, but to do it safely across multiple servers, only when required, and keep a proper record of what happened on each machine.

## How execution starts and how Ansible selects hosts

```yaml
- name: Ensure authselect mkhomedir feature is enabled
  hosts: "{{ group }}"
  become: true
  gather_facts: true
```

When we run this playbook, we pass a variable like this:

```bash
ansible-playbook playbook.yml -e "group=dev_servers"
```

Ansible reads the inventory, resolves `dev_servers`, and builds a list of machines to target. Instead of hardcoding hostnames, we are making the playbook reusable across environments.

`become: true` means every command will run with elevated privileges, typically using `sudo`, because modifying authentication configuration requires root access. Internally, Ansible wraps commands like:

```bash
sudo authselect current
```

`gather_facts: true` triggers a built-in module that collects system information before any task runs. This includes values like:

```yaml
ansible_hostname
ansible_default_ipv4.address
```

These are later used in reporting, so this step is not optional in this playbook.

## Dynamic report file creation

```yaml
vars:
  report_file: "/tmp/{{ group }}_mkhomedir_report.csv"
```

Here we dynamically generate a file name based on the group. If we pass `group=prod`, then:

```bash
/tmp/prod_mkhomedir_report.csv
```

This allows us to separate reports per environment without changing the playbook.

## Checking whether authselect exists

```yaml
- name: Check if authselect is installed
  command: which authselect
  register: authselect_bin
  ignore_errors: true
  changed_when: false
```

The `command` module executes a shell command on the target machine. Here:

```bash
which authselect
```

If the binary exists, output might be:

```bash
/usr/bin/authselect
```

The result is stored using `register`. Internally, this creates a structured variable:

```yaml
authselect_bin:
  rc: 0
  stdout: /usr/bin/authselect
  stderr: ""
```

If not installed:

```yaml
rc: 1
```

`ignore_errors: true` ensures that even if the command fails, the playbook continues. This is important because we want to handle this condition ourselves instead of letting Ansible stop execution.

`changed_when: false` ensures this task is treated as a check, not a modification.

## Handling systems where authselect is missing

```yaml
- name: Mark authselect missing
  set_fact:
    error_msg: "authselect not installed"
  when: authselect_bin.rc != 0
```

Here we create a runtime variable using `set_fact`. This variable exists only for that host during execution.

Then we stop further execution:

```yaml
- name: Stop further tasks if authselect missing
  meta: end_host
  when: authselect_bin.rc != 0
```

`meta: end_host` tells Ansible to stop processing this host completely but continue with others. This is a controlled exit, not a failure.

## Reading current authselect configuration

```yaml
- name: Get current authselect profile and features
  command: authselect current
  register: authselect_current
  changed_when: false
```

This command shows current configuration:

```bash
authselect current
```

Example output:

```text
Profile ID: sssd
Enabled features:
- with-mkhomedir
```

We capture this output for analysis.

## Checking if mkhomedir is already enabled

```yaml
- name: Check if mkhomedir feature is already enabled
  set_fact:
    mkhomedir_enabled: "{{ 'with-mkhomedir' in authselect_current.stdout }}"
```

We are using a Jinja2 expression to check whether the feature string exists in output. This converts raw text into a boolean:

```yaml
mkhomedir_enabled: true / false
```

Now we can make decisions based on this.

## Enabling mkhomedir only when required

```yaml
- name: Enable mkhomedir feature if missing
  command: authselect enable-feature with-mkhomedir
  when: not mkhomedir_enabled
  register: enable_mkhomedir
  ignore_errors: true
```

If the feature is not enabled, we run:

```bash
authselect enable-feature with-mkhomedir
```

If already enabled, Ansible skips this task.

This conditional execution is what makes the playbook idempotent, meaning it does not make unnecessary changes.

We again capture the result using `register`.

## Applying configuration changes safely

```yaml
- name: Apply authselect changes if feature was enabled
  command: authselect apply-changes
  when:
    - enable_mkhomedir is defined
    - enable_mkhomedir is not skipped
    - enable_mkhomedir.rc == 0
```

After enabling the feature, changes are not active until we run:

```bash
authselect apply-changes
```

But we only run this when:

* The previous task executed
* It was not skipped
* It succeeded

This prevents applying broken or incomplete configurations.

## Preparing final status values

```yaml
- name: Set final status facts
  set_fact:
    already_enabled_flag: "{{ 'YES' if mkhomedir_enabled else 'NO' }}"
    enabled_by_playbook: >-
      {{ 'YES'
         if (enable_mkhomedir is defined
             and enable_mkhomedir is not skipped
             and enable_mkhomedir.rc == 0)
         else 'NO' }}
    error_final: >-
      {{
        enable_mkhomedir.stderr
        if (enable_mkhomedir is defined
            and enable_mkhomedir is not skipped
            and enable_mkhomedir.rc != 0)
        else (error_msg | default(''))
      }}
```

We convert technical values into human-readable flags. Instead of `true/false`, we store `YES/NO`.

We also capture errors:

* If enable command failed → take `stderr`
* If authselect missing → take earlier error

We also sanitize error messages before writing:

```yaml
error_final | replace(',', ';')
```

This ensures CSV format does not break.

## Creating CSV header on control node

```yaml
- name: Create CSV header (once)
  run_once: true
  delegate_to: localhost
  become: false
  copy:
    dest: "{{ report_file }}"
    content: |
      hostname,ip,already_enabled,enabled_by_playbook,error
```

Here we shift execution to the control node using `delegate_to: localhost`. This means instead of running on remote servers, this task runs on the machine where Ansible is executed.

`run_once: true` ensures this runs only once, even if we have 100 hosts.

So we create the file:

```bash
/tmp/dev_servers_mkhomedir_report.csv
```

## Appending results for each host

```yaml
- name: Append host result to CSV
  delegate_to: localhost
  become: false
  lineinfile:
    path: "{{ report_file }}"
    line: >-
      {{ ansible_hostname }},
      {{ ansible_default_ipv4.address | default('NA') }},
      {{ already_enabled_flag }},
      {{ enabled_by_playbook }},
      {{ error_final | replace(',', ';') }}
```

For each host, we append a line to the CSV file.

Even though the task runs per host, execution is delegated to localhost. So the flow becomes:

* Collect data from remote host
* Send values to control node
* Write one line into CSV

Example output:

```text
hostname,ip,already_enabled,enabled_by_playbook,error
server1,10.0.0.1,YES,NO,
server2,10.0.0.2,NO,YES,
server3,10.0.0.3,NO,NO,authselect not installed
```

## What we built in real terms

In real-world execution, we created a full automation workflow where we first validated the system, handled missing dependencies safely, inspected current configuration, applied changes only when required, handled failures without stopping execution, and finally generated a centralized report.

Instead of manually running commands like:

```bash
authselect current
authselect enable-feature with-mkhomedir
authselect apply-changes
```

on every server, we converted the entire process into a controlled, repeatable, and auditable automation.
