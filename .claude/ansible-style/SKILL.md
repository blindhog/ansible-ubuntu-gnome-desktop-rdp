---
name: ansible-style
description: Ansible style and conventions for this repository — task naming tags, section comment headers, json_query usage, FQCN, shell command restrictions, loop_control labels, playbook/inventory structure, and ansible-lint rule fixes (truthy, trailing-spaces, line-length, file permissions, changed_when, ignore-errors). Use when writing or editing Ansible playbooks, roles, tasks, or inventory files.
paths:
  - "**/*.yml"
  - "**/*.yaml"
---

# Ansible Style

Ansible style guidelines and best practices for this project.

---

## Inventory Structure

Store host and group variables in dedicated files, not inline in `hosts.yml`:

- Host variables → `inventory/host_vars/<host_name>/`
- Group variables → `inventory/group_vars/<group_name>/`

```
# Good
inventory/
  hosts.yml               # only host/group membership
  host_vars/
    web01/
      vars.yml
  group_vars/
    all/
      vars.yml

# Bad — vars defined inline in hosts.yml
all:
  hosts:
    web01:
      ansible_host: 10.0.0.1
      my_var: value
```

---

## Task Naming

Prefix every task name with a category tag in uppercase followed by a pipe and space:

| Tag | Use for |
|-----|---------|
| `SET \|` | `set_fact`, variable assignments |
| `API \|` | HTTP requests (`uri` module) |
| `AUTH \|` | Authentication steps |
| `INFO \|` | Informational `debug` output |
| `DEBUG \|` | Diagnostic `debug` output (tag with `debug`) |
| `FIND \|` | File/resource discovery |
| `FAIL \|` | Explicit failure checks |
| `VALIDATE \|` | Input/variable validation |
| `INCLUDE \|` | `include_role` / `include_tasks` |

```yaml
# Good
- name: SET | Build datastore moref → name map
- name: API | Get vApp details
- name: FAIL | Abort if VM directory not found

# Bad
- name: Get data
- name: Set the variable
```

Every task must have a `name:` — this includes `debug` tasks. Never use the bare free-form form:

```yaml
# Good
- name: DEBUG | Show resolved cloud name
  ansible.builtin.debug:
    var: openstack_cloud.name

# Bad — missing name
- debug: var=openstack_cloud.name
```

---

## Debug Tags

Every `ansible.builtin.debug` task must carry `tags: [debug]`, regardless of its name prefix (`DEBUG |` or `INFO |`), so debug output can be filtered with `--skip-tags debug` / `--tags debug`:

```yaml
# Good
- name: DEBUG | Show resolved cloud name
  ansible.builtin.debug:
    var: openstack_cloud.name
  tags:
    - debug

# Bad — debug module with no debug tag
- name: DEBUG | Show resolved cloud name
  ansible.builtin.debug:
    var: openstack_cloud.name
```

---

## Section Comment Blocks

Group related tasks under a section header comment block. Use this exact format — 74 dashes:

```yaml
    # --------------------------------------------------------------------------
    # Authentication: Get Session Token
    # --------------------------------------------------------------------------
```

At the top of role task files, use the wider `=` style header (74 `=` chars):

```yaml
# ==========================================================================
# TASK: Short description of what this task file does
# ==========================================================================
#
# Longer description, required variables, and output facts documented here.
#
# Required Variables:
#   - var_name: description
#
# Output Facts:
#   - fact_name: description
#
# ==========================================================================
```

---

## JSON Parsing

**Always use `json_query` (via `community.general.json_query`) for JSON filtering and extraction.** Do not use chained `select` / `selectattr` / `map` / `list` filters for complex nested queries.

```yaml
# Good — json_query for complex nested access
vm_moref: >-
  {{ vm_data.section
     | community.general.json_query("[?_type=='GuestCustomizationSectionType'].virtualMachineId | [0]") }}

cpu_count: >-
  {{ vm_data.section
     | community.general.json_query("[?_type=='VirtualHardwareSectionType'].item[] | [?resourceType==`3`].virtualQuantity | [0]") }}

# Acceptable — selectattr for simple single-attribute equality filters on lists
vdc: "{{ vcloud_vdcs | selectattr('name', 'equalto', vcloud) | first }}"

# Bad — chained select/map for nested or multi-step extraction
result: "{{ data | selectattr('type', 'equalto', 'Foo') | map(attribute='value') | first }}"
```

Use `| default('', true)` or `| default('N/A', true)` when the query may return empty/null.

---

## Fully Qualified Collection Names (FQCN)

Always use FQCN for all modules:

```yaml
# Good
ansible.builtin.set_fact:
ansible.builtin.uri:
ansible.builtin.debug:
ansible.builtin.shell:
ansible.builtin.fail:
ansible.builtin.include_role:
community.general.json_query

# Bad
set_fact:
uri:
debug:
```

---

## Loop Control

Always include `loop_control.label` for any loop over non-trivial items:

```yaml
loop: "{{ vm_hrefs }}"
loop_control:
  label: "{{ item.vm_name }}"
```

---

## Shell Commands

**Do not use `ansible.builtin.shell` or `ansible.builtin.command` when an Ansible module exists that can complete the task.** Only reach for shell/command when no suitable module covers the operation (e.g., complex glob searches, custom CLI tools).

For read-only shell tasks, always set `changed_when: false`. When failures are handled downstream, use `failed_when: false`:

```yaml
- name: FIND | Locate VM directory
  ansible.builtin.shell: |
    find {{ paths | map('quote') | join(' ') }} -maxdepth 1 -type d -iname "${NAME}-????"
  environment:
    NAME: "{{ item }}"
  register: result
  changed_when: false
  failed_when: false
```

Pass dynamic values through `environment:` rather than shell interpolation to prevent injection.

Every `command`/`shell` task needs an explicit `changed_when`, even when it isn't read-only. Don't leave it unset and let the module report "changed" unconditionally — check the actual result. Verify the marker text against real output before relying on it; don't guess:

```yaml
# Good — reflects the real outcome, confirmed against actual apt-get output
- name: Install Playwright Chromium system dependencies
  ansible.builtin.command:
    cmd: "{{ virtualenv_path }}/bin/playwright install-deps chromium"
  register: playwright_deps_install
  changed_when: "'0 newly installed' not in playwright_deps_install.stdout"

# Bad — always reports changed, even when there was nothing to do
- name: Install Playwright Chromium system dependencies
  ansible.builtin.command:
    cmd: "{{ virtualenv_path }}/bin/playwright install-deps chromium"
```

Some commands are inherently idempotent but print no distinguishing text for the "nothing to do" case (verify this with a real run before assuming it). For those, `changed_when: false` is more honest than fabricating a text check that doesn't actually appear in the output:

```yaml
# Good — playwright's browser installer skips silently either way; no text to key off of
- name: Install Playwright Chromium browser binary
  ansible.builtin.command:
    cmd: "{{ virtualenv_path }}/bin/playwright install chromium"
  changed_when: false
```

When a `shell` command pipes output through another command, set `pipefail` first so a failure earlier in the pipe isn't masked by the exit code of the last command:

```yaml
# Good
- name: SHELL | Replace Windows boot images with virt-v2v output
  ansible.builtin.shell:
    cmd: |
      set -o pipefail
      output=$(find {{ _workdir }} -maxdepth 1 -type f ! -name '*.xml' | head -1)
      mv -f "$output" {{ _dest }}

# Bad — pipefail unset; a failed `find` is silently swallowed by `head`
- name: SHELL | Replace Windows boot images with virt-v2v output
  ansible.builtin.shell:
    cmd: |
      output=$(find {{ _workdir }} -maxdepth 1 -type f ! -name '*.xml' | head -1)
      mv -f "$output" {{ _dest }}
```

---

## Error Handling

**Do not add `ignore_errors: true` in new task suggestions.** Use explicit `failed_when` / `rescue` blocks instead so failures are handled deliberately rather than silently swallowed.

If `ignore_errors` already exists in a task you're editing, leave it in place — do not remove it or flag it as a violation.

---

## Variable Scoping with `vars:`

Use inline `vars:` blocks on tasks to compute intermediate values rather than preceding `set_fact` tasks:

```yaml
- name: SET | Enhance VM info
  ansible.builtin.set_fact:
    enhanced_vm_info: "{{ enhanced_vm_info | default([]) + [enhanced_info] }}"
  vars:
    vmx_resolved: "{{ (_vmx_path_map | default({}))[item.vmx_file_path] | default('') }}"
    ds_name: "{{ vmx_resolved | regex_search('\\[([^\\]]+)\\]', '\\1') | default([''], true) | first }}"
    enhanced_info: "{{ item | combine({'datastore_name': ds_name}) }}"
  loop: "{{ vm_vcenter_info }}"
```

---

## Conditional Guards

Prefer explicit `is defined` / `is not skipped` / `is not failed` guards over truthy checks for registered variables:

```yaml
when:
  - storage_containers_response is not skipped
  - storage_containers_response is not failed
  - storage_containers_response.json is defined
```

---

## Error Handling

**Do not add `ignore_errors: true` in new task suggestions.** Use explicit `failed_when` / `rescue` blocks instead so failures are handled deliberately rather than silently swallowed.

If `ignore_errors` already exists in a task you're editing, leave it in place — do not remove it or flag it as a violation.


```yaml
# Good — tolerates only the known "already exists" case
- name: SECGROUP | Add SSH rule to Linux security group
  openstack.cloud.security_group_rule:
    cloud: "{{ openstack_cloud.name }}-{{ vcloud_vapp }}"
    security_group: "{{ vcloud_vapp }}-linux-sg"
    protocol: tcp
    port_range_min: 22
    port_range_max: 22
  register: sg_rule_result
  failed_when:
    - sg_rule_result is failed
    - "'already exists' not in sg_rule_result.msg | default('')"

# Bad — swallows every kind of failure, not just "already exists"
- name: SECGROUP | Add SSH rule to Linux security group
  openstack.cloud.security_group_rule:
    cloud: "{{ openstack_cloud.name }}-{{ vcloud_vapp }}"
    security_group: "{{ vcloud_vapp }}-linux-sg"
    protocol: tcp
    port_range_min: 22
    port_range_max: 22
  ignore_errors: true
```

---

## File Permissions

Always set `mode:` explicitly on `copy`, `template`, and `file` tasks that create or modify a file or directory — don't rely on the module's default:

```yaml
# Good
- name: COPY requirements.apt to host /tmp/
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "/tmp/{{ item | basename }}"
    mode: "0644"

# Bad — mode omitted
- name: COPY requirements.apt to host /tmp/
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "/tmp/{{ item | basename }}"
```

---

## YAML Formatting

`ansible-lint`/`yamllint` enforce these formatting rules — check for them before committing:

| Rule | Requirement |
|------|-------------|
| `truthy` | Use lowercase `true` / `false` for booleans — not `yes`, `no`, `True`, `False`, `on`, `off` |
| `trailing-spaces` | No trailing whitespace at the end of any line |
| `line-length` | Max 160 characters per line — wrap long Jinja expressions with `>-` or `\|` block scalars |
| `new-line-at-end-of-file` | Every file must end with exactly one newline |
| `empty-lines` | No blank line immediately before end-of-file; no more than one consecutive blank line elsewhere |

```yaml
# Good
update_cache: true

# Bad
update_cache: yes
```

---

## Playbook Structure

Standard playbook header:

```yaml
---
# =============================================================================
# Playbook: <Title>
# =============================================================================
#
# Purpose: <what it does>
#
# Prerequisites:
#   - <requirement>
#
# Usage:
#   ansible-playbook <playbook-name>.yml -e 'var=value'
#
# =============================================================================

- name: <Playbook Name>
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    output_dir: "./output"
  tasks:
```

Use `gather_facts: false` for API/cloud playbooks that run against localhost.

---

## Validation Block

Always validate required extra vars before any work:

```yaml
    # --------------------------------------------------------------------------
    # Validate required variables
    # --------------------------------------------------------------------------
    - name: VALIDATE | Ensure <var> was provided
      ansible.builtin.fail:
        msg: >-
          The '<var>' variable is required. Pass it as an extra var
          (e.g. -e '<var>="value"').
      when: var_name is not defined or var_name | length == 0
```
