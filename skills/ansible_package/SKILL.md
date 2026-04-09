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
SKILL.md) for ALL execution. Do NOT run local `ansible` CLI commands or
`scripts/run.sh` — that wrapper invokes the local `ansible` CLI only, not AAP.

| Setting | Value |
|---------|-------|
| AAP Controller | `https://aap-aap.apps.cluster-rmf2p-1.dynamic.redhatworkshops.io` |
| Default AAP inventory | `Demo Inventory` |
| Default Credential | `Demo Credential` |
| Default Project | `AnsibleClaw` |
| Default EE | `Default execution environment` |

The **Default AAP inventory** value is a **Controller inventory** (name or numeric ID) defined and maintained in Ansible Automation Platform, not a path to a local file such as `inventory/hosts.yml`.

**IMPORTANT**: `AAP_CONTROLLER_TOKEN` is required only for Controller API calls
(`create-jt`, `launch`, `status`, `adhoc`). You can still edit `assets/playbook.yml`
and push changes to the Project Git repo without the token.

### Inventory (production)

Hosts and groups for AAP runs come from **inventories managed in AAP** (UI or API). Do not rely on the repo’s static `inventory/` files for production execution. The `--inventory` flag on `aap_run.py` selects **another AAP inventory** (name or ID), not a path on disk.

### Git and the AAP Project (production)

- The Job Template **playbook path** is **relative to the AAP Project’s Git repository** (the SCM URL on the Project in Controller), **not** to a path under `~/.cursor/skills`, `~/.gemini/skills`, or other local agent install directories.
- When you change `assets/playbook.yml` (including from the bundled example), **commit and push** to **that same Git remote** so the next **project sync** in AAP picks up the file. AnsibleClaw **Deploy to AAP** in the web UI is another way to push skill trees into that repo.
- **SCM URL hint** (from AnsibleClaw `AAP_DEFAULT_SCM_URL` / `.ansibleclaw.yml`): `https://github.com/micytao/AnsibleClaw-Demo.git`
- If you only have this skill under an agent skills folder, copy or merge into a checkout of the Project repository before pushing; **Git credentials and branch policy are your responsibility**.

**Recommended Git flow for assistants (do this first):**
Use the helper script:

```bash
bash scripts/publish_playbook.sh
```

This skill has an SCM hint baked in (`https://github.com/micytao/AnsibleClaw-Demo.git`), so the script can clone automatically if needed.

### Quick Start (FOLLOW THESE STEPS EXACTLY)

**Preferred path: Job Templates** (repeatable, RBAC-friendly, typical production flow).

1. **Prepare and push playbook changes first**:
   - Update `assets/playbook.yml` for the user request.
   - Run `bash scripts/publish_playbook.sh` to copy/update the skill under `skills/ansible_package/` in the AAP Project repo and push.

2. **Optional prerequisite check** (useful before API calls):
   ```bash
   bash scripts/check.sh
   ```

3. **Create a Job Template** (skip when **Deploy to AAP** already created it):
   - Use the same project-relative playbook path that you pushed in step 1.
   - AnsibleClaw **Deploy to AAP** can also push, sync, and create the Job Template.

4. **Create a Job Template** via helper script:
   ```bash
   python3 scripts/aap_run.py create-jt --name "package"
   ```
   Use your site’s Job Template naming convention if a prefix is configured (e.g. `AnsibleClaw: …`).

**Agent / automation stop point:** For production, treat **Job Template creation** as the usual handoff. **Do not** launch jobs from the assistant unless an operator has explicitly approved it. **AAP administrators** run or schedule the Job Template in the Controller UI (security and audit).

#### Operators: launch and job status (after approval)

```bash
python3 scripts/aap_run.py launch "package"
# With extra variables, limits, inventory overrides — see **How to Execute (AAP)** below
python3 scripts/aap_run.py status <job-id>
```

#### Optional: ad-hoc (diagnostics only; avoid for normal changes)

Use ad-hoc only when an operator explicitly asks for a quick probe.
For normal changes, use the Job Template path above.

For `ansible.builtin.package`, prefer playbook + Job Template flow.
Some AAP environments reject ad-hoc requests for this meta-module.

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

**Do not use `scripts/run.sh` in AAP mode** — it is for local CLI execution only.

**Prefer Job Templates** for normal work. Treat ad-hoc as diagnostics-only.
If ad-hoc fails due to Controller module restrictions, use Job Templates.

All launches use **AAP-managed inventories**; Job Templates are created with a Controller inventory attached. Use baked defaults or `--inventory` to refer to an inventory object in AAP, not a local file.

### Job Templates

Job Templates provide repeatable, RBAC-controlled execution in AAP.

**Create a Job Template** (if one does not exist yet):

```bash
python3 scripts/aap_run.py create-jt --name "package"
```

**Launch** (typically **AAP administrators** or approved operators after the Job Template exists):

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

### Optional: ad-hoc commands (diagnostics only)

Use only for operator-directed probes; do not use as the default execution path.
If ad-hoc is rejected by AAP policy or module allow-lists, continue with Job Templates.

For `ansible.builtin.package`, many Controllers reject ad-hoc usage of `package`.
Prefer playbook + Job Template for package install/remove workflows.

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

- **ALWAYS dry-run first**: In AAP mode, run the Job Template in **check mode** from the Controller UI when available (or validate in a non-production inventory first). Use ad-hoc `--check` only for diagnostics. In CLI mode use `--check --diff` before applying changes
- **Become/sudo**: Most system-level modules require elevated privileges. In AAP mode, this is configured in the credential.
- **Idempotency**: This module is idempotent -- running it multiple times with the same arguments produces the same result
