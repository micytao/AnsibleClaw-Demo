---
name: ansible-package
description: >-
  Generic OS package manager
  Use when managing package resources on remote hosts via Ansible.
---

# ansible.builtin.package

Generic OS package manager

## Execution Mode -- READ THIS FIRST

**AAP mode is active.** You MUST use `scripts/aap_run.py` (located next to this
SKILL.md) for ALL execution. Do NOT run local `ansible` CLI commands.

| Setting | Value |
|---------|-------|
| AAP Controller | `https://aap-aap.apps.cluster-rmf2p-1.dynamic.redhatworkshops.io` |
| Default AAP inventory | `Demo Inventory` |
| Default Credential | `Demo Credential` |
| Default Project | `AnsibleClaw` |
| Default EE | `Default execution environment` |

The **Default AAP inventory** value is a **Controller inventory** (name or numeric ID) defined and maintained in Ansible Automation Platform, not a path to a local file such as `inventory/hosts.yml`.

**IMPORTANT**: The environment variable `AAP_CONTROLLER_TOKEN` MUST be set
before running any command. All other AAP settings are pre-configured.

### Inventory (production)

Hosts and groups for AAP runs come from **inventories managed in AAP** (UI or API). Do not rely on the repo’s static `inventory/` files for production execution. The `--inventory` flag on `aap_run.py` selects **another AAP inventory** (name or ID), not a path on disk.

### Quick Start (FOLLOW THESE STEPS EXACTLY)

1. **Check prerequisites**:
   ```bash
   bash scripts/check.sh
   ```

2. **Run ad-hoc command** (dry-run first, ALWAYS):
   ```bash
   python3 scripts/aap_run.py adhoc "name=ntpdate state=present" --check
   ```

3. **Apply the change** (after reviewing dry-run output):
   ```bash
   python3 scripts/aap_run.py adhoc "name=ntpdate state=present"
   ```

4. **Or launch an existing Job Template**:
   ```bash
   python3 scripts/aap_run.py launch "package"
   ```

5. **If no Job Template exists yet, create one first**:
   ```bash
   python3 scripts/aap_run.py create-jt --name "package"
   ```
   Then launch it with step 4.

## When to Use This Skill

Use the `ansible.builtin.package` Ansible module when you need to manage package on remote hosts. This is preferable to local CLI commands when:

- Targeting one or more **remote** hosts over SSH
- You need **idempotent** state management (ensure a desired state, not just run a command)
- You want **audit trails** and **dry-run** capability via `--check`

Do **not** use this for basic local file operations or CLI tasks that the agent can handle natively.

## Parameters

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `name` | str | yes | - | Package name, or package specifier with version. Syntax varies with package manager. For example V(name-1.0) or... |
| `state` | str | yes | - | Whether to install (V(present)), or remove (V(absent)) a package. You can use other states like V(latest) ONLY if... |
| `use` | str | no | auto | The required package manager module to use (V(dnf), V(apt), and so on). The default V(auto) will use existing facts... |
## How to Execute (AAP)

You MUST use `scripts/aap_run.py` for all commands below. The script
auto-detects the correct API path for both AAP 2.4 (`/api/v2`) and
AAP 2.5+ Gateway (`/api/controller/v2`).

All jobs and ad-hoc commands use **AAP-managed inventories**; Job Templates are created with a Controller inventory attached. Use baked defaults or `--inventory` to refer to an inventory object in AAP, not a local file.

### Ad-Hoc Commands

Run the module directly on AAP-managed hosts:

```bash
# ALWAYS dry-run first
python3 scripts/aap_run.py adhoc "name=ntpdate state=present" --check

# Apply the change after reviewing dry-run output
python3 scripts/aap_run.py adhoc "name=ntpdate state=present"

# Target specific hosts
python3 scripts/aap_run.py adhoc "name=ntpdate state=present" --limit "web1.example.com"
```

### Job Templates

Job Templates provide repeatable, RBAC-controlled execution in AAP.

**Create a Job Template** (if one does not exist yet):

```bash
python3 scripts/aap_run.py create-jt --name "package"
```

**Launch a Job Template**:

```bash
python3 scripts/aap_run.py launch "package"

# With extra variables
python3 scripts/aap_run.py launch "package" --extra-vars '{"target_hosts": "webservers"}'

# Limit to specific hosts
python3 scripts/aap_run.py launch "package" --limit "web1.example.com"
```

**Check job status**:

```bash
python3 scripts/aap_run.py status <job-id>
```

## Examples from Ansible Documentation

```yaml
- name: Install ntpdate
  ansible.builtin.package:
    name: ntpdate
    state: present

# This uses a variable as this changes per distribution.
- name: Remove the apache package
  ansible.builtin.package:
    name: "{{ apache }}"
    state: absent

- name: Install the latest version of Apache and MariaDB
  ansible.builtin.package:
    name:
      - httpd
      - mariadb-server
    state: latest

- name: Use the dnf package manager to install httpd
  ansible.builtin.package:
    name: httpd
    state: present
    use: dnf
```

## Safety

- **ALWAYS dry-run first**: Use `--check` (AAP) or `--check --diff` (CLI) before applying changes
- **Become/sudo**: Most system-level modules require elevated privileges. In AAP mode, this is configured in the credential.
- **Idempotency**: This module is idempotent -- running it multiple times with the same arguments produces the same result
