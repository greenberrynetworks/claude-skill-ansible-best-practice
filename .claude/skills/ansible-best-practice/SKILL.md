---
name: ansible-best-practice
description: Use when writing, editing, reviewing, or fixing Ansible playbooks, roles, task files, handlers, templates, or inventory in this repository, or documenting how to run them — including new roles, ansible-lint failures, idempotency or --check problems, secrets in task output, and long-running async work.
paths:
  - "ansible/**/*.yml"
  - "ansible/**/*.yaml"
  - "ansible/**/*.md"
---

# Ansible Best Practice

## Overview

Two layers, both mandatory: **house style** (how this repo's Ansible looks) and **engineering practice** (how it behaves: idempotent, check-safe, secret-safe, honest about change). Passing `ansible-lint` covers neither layer fully — lint-clean code that ignores house style is still wrong here.

This skill includes every rule from `ansible-style` plus general best practice. Longer worked examples live in `patterns.md` in this directory — read it when writing a new role, async fan-out, or check-mode-sensitive tasks.

## Every Task File Must Have

| Item | Rule |
|------|------|
| File header | Playbooks: `=` header with Purpose / Prerequisites / Usage. Role task files: `=` header with `TASK:` title, Required Variables, Variables Created. |
| Section headers | Group related tasks under a 74-dash `# ---` comment block with a title. |
| Task names | Every task (including `debug`, `meta`, handlers) has `name:` starting with an UPPERCASE tag + ` \| `. |
| FQCN | `ansible.builtin.*`, `community.general.*`, `openstack.cloud.*` — never bare module names or free-form `key=value` args. |
| Debug tag | Every `ansible.builtin.debug` task carries `tags: [debug]`. |
| Loop label | Every `loop:` over non-trivial items has `loop_control.label`. Use `loop:`, not `with_items:`. |
| `mode:` | Every `copy` / `template` / `file` that creates or modifies a path sets `mode:` explicitly (quoted, e.g. `"0644"`). |

## Task Naming Tags

`<TAG> | <Imperative description>`. Common tags:

| Tag | Use for |
|-----|---------|
| `SET` | `set_fact`, variable assignments |
| `API` / `URI` | HTTP requests (`uri`) |
| `AUTH` | Authentication / token steps |
| `VALIDATE` | Checking required vars / inputs (`assert`, `fail`) |
| `FAIL` | Explicit failure checks mid-play |
| `FIND` / `STAT` | File/resource discovery |
| `INFO` / `DEBUG` / `WARN` | `debug` output |
| `INCLUDE` | `include_role` / `include_tasks` |
| `COPY` / `FILE` / `MOUNT` / `SHELL` / `COMMAND` | Those module families |

When none fits, use a short uppercase noun for the resource or module (`SECGROUP |`, `OPENSTACK |`, `SERVICE |`, `HANDLER |`). Never leave a name untagged.

```yaml
# Good
- name: SET | Build datastore moref → name map
- name: API | Get vApp details
# Bad
- name: Get data
- name: Install NFS client
```

## Headers

Playbook (top of file):

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
```

Role task file (top of file):

```yaml
---
# ==========================================================================
# TASK: Short description of what this task file does
# ==========================================================================
#
# Required Variables:
#   - var_name: description
#
# Variables Created:
#   - fact_name: description
#
# ==========================================================================
```

Section header inside tasks — always exactly 74 dashes, at whatever indent the surrounding tasks use (including inside `block:`):

```yaml
    # --------------------------------------------------------------------------
    # Authentication: Get Session Token
    # --------------------------------------------------------------------------
```

## Validate Inputs First

The first section of any playbook/role that takes required vars validates them, named `VALIDATE |`, with a message telling the caller how to pass the value:

```yaml
    - name: VALIDATE | Ensure nfs_server was provided
      ansible.builtin.fail:
        msg: >-
          The 'nfs_server' variable is required. Pass it as an extra var
          (e.g. -e 'nfs_server="10.0.0.5"').
      when: nfs_server is not defined or nfs_server | length == 0
```

Roles also declare inputs in `meta/argument_specs.yml` (types, `required`, `no_log: true` on secrets). Prefix every role variable with the role name (`glance_image_sync_*`) — `defaults/main.yml` documents each one with a comment.

## Inventory

`hosts.yml` holds **only** group/host membership. All variables — including `ansible_host` and `ansible_user` — go in `inventory/host_vars/<host>/vars.yml` and `inventory/group_vars/<group>/vars.yml`. Secrets go in a separate vaulted `vault.yml` beside `vars.yml`.

## JSON Parsing

Use `community.general.json_query` for filtering/extracting from JSON or lists of dicts. `selectattr` is acceptable **only** for a single-attribute equality filter followed by `first`. Chains like `selectattr|map`, `rejectattr|map|list` are not allowed — rewrite as `json_query`.

```yaml
# Good
not_ready: "{{ health.workers | community.general.json_query(\"[?state!='ready'].name\") }}"
vdc: "{{ vcloud_vdcs | selectattr('name', 'equalto', vcloud) | first }}"
# Bad
not_ready: "{{ health.workers | rejectattr('state', 'equalto', 'ready') | map(attribute='name') | list }}"
```

Add `| default('', true)` / `| default([], true)` when a query may return null. Parse command output with `from_json` inline in `vars:` rather than an extra `set_fact` task.

## Modules Before Shell

Use a module whenever one exists (`apt`, `file`, `find`, `stat`, `mount`, `openstack.cloud.*`). Only use `shell`/`command` for tools with no module. When you must:

- Every `command`/`shell` has an explicit `changed_when`. Read-only → `changed_when: false`. State-changing → a condition on the real output, verified against an actual run (don't invent marker text).
- Read-only commands that must run under `--check` also get `check_mode: false`.
- Any pipe in `shell` → `set -o pipefail` first.
- Pass dynamic values through `environment:` rather than interpolating into the command string.

## Honest Change Reporting

`changed_when: false` is only for tasks that genuinely change nothing (queries, checks, `async_status` polling). **Never** put it on a task that creates/uploads/modifies something just to quiet output — that hides real changes from `--check`, `--diff`, and handlers.

## Errors: No `ignore_errors`

Do not add `ignore_errors` — not even templated (`ignore_errors: "{{ ansible_check_mode }}"`). Instead:

- Tolerate a *specific* known failure with `failed_when` listing the conditions.
- Skip a step that can't work in check mode with `when: not ansible_check_mode` (or a narrower condition).
- Guarantee cleanup with `block:` / `always:`; handle recoverable failures with `rescue:`.

Existing `ignore_errors` in a task you're editing: leave it, don't flag it.

```yaml
# Good — tolerates only "already exists"
  register: sg_rule_result
  failed_when:
    - sg_rule_result is failed
    - "'already exists' not in sg_rule_result.msg | default('')"
```

## Conditionals

Guard registered vars explicitly: `is defined`, `is not skipped`, `is not failed`, `is changed` — not bare truthiness. No `{{ }}` in `when:`.

## Secrets

Tasks that send or receive credentials/tokens use the repo toggle instead of hardcoded `no_log: true`, so they can be debugged deliberately:

- Inside `roles/vmware-to-openstack` (defines it in defaults): `no_log: "{{ redact_sensitive_output }}"`
- Anywhere else (new roles, standalone playbooks): `no_log: "{{ redact_sensitive_output | default(true) }}"`

`no_log: true` on a secret option in `meta/argument_specs.yml` is correct — the toggle rule applies to tasks. Never `debug` a token or password. Passwords come from vault or extra vars, never plaintext in playbooks.

## Variables

Compute intermediate values in a task-level `vars:` block instead of chains of preceding `set_fact` tasks. Only `set_fact` what later tasks need.

## Async / Long-Running Work

Fan out with `async: <timeout>` + `poll: 0`, then wait with `async_status` + `until: <reg>.finished` and `retries` derived from the timeout. Wrap the wait in `block:` with the `mode: cleanup` loop in `always:`. See `patterns.md`.

- Also raise the module's own wait/timeout option (e.g. `openstack.cloud.image` `timeout:` defaults to 180 s) — `async` alone doesn't stop the module timing out.
- `async` tasks can't run under `--check`; guard the fan-out block with `when: not ansible_check_mode`.

## Handlers & Check Mode

- Restart services via `notify` → handler, not an unconditional restart task. Use `meta: flush_handlers` (named, tagged `HANDLER |`) when a later task needs the restarted service.
- A play must complete under `--check`: modules support it natively; read-only commands use `check_mode: false`; tasks depending on files check mode didn't create use `when: not ansible_check_mode`.
- Follow the data: when a task is skipped under `--check` (e.g. `uri` POSTs, `command` without `check_mode: false`), every later task that reads its registered result must also be guarded (`when: not ansible_check_mode` or `when: <reg> is not skipped`). Fixing only the first failing task leaves the next one broken.

## YAML / Lint

| Rule | Requirement |
|------|-------------|
| `truthy` | `true` / `false` only |
| `trailing-spaces` | None |
| `line-length` | ≤ 160 chars; wrap Jinja with `>-` |
| `new-line-at-end-of-file` | Exactly one trailing newline |
| `empty-lines` | Max one consecutive blank line; none before EOF |

API/cloud playbooks against localhost: `gather_facts: false`, `connection: local`.

**Verify before claiming done:** run `ansible-playbook --syntax-check` and `ansible-lint` on the files you touched and report the result.

## Documenting Commands

When docs (READMEs, `USAGE-*.md`, playbook `Usage:` headers) show a command from the Ansible CLI family (`ansible`, `ansible-playbook`, `ansible-vault`, `ansible-galaxy`, `ansible-inventory`, `ansible-lint`, `ansible-doc`, ...), write it one option per line, with each scalar extra var as its own `-e key=value`. Other commands (`curl`, AWX API calls, shell) keep their own format.

```bash
ansible-playbook playbooks/vapp-to-openstack-migrate-vapp.yml \
  -e vcloud=sandbox \
  -e openstack=atc-cloudlab-test \
  -e vcloud_vapp=my-vapp \
  -e automate_qa=true \
  -e '{"vcloud_vapp_vms": ["freenas", "vcenter"]}' \
  -l localhost \
  --vault-password-file .vault
```

- Booleans and numbers go in `key=value` form too: playbooks read them through `| bool` / `| int`.
- A list or dict gets its own JSON `-e '{"name": ...}'`, one variable per `-e`, since `key=value` would make it a string.
- Quote a value only when the shell needs it (`-e 'vapp_pattern=qa-*'`).
- Var files stay `-e @file.yml`.

## Red Flags — Stop and Fix

| You're about to... | Instead |
|--------------------|---------|
| Name a task `Install NFS client` | `APT \| Install NFS client` / `MOUNT \| ...` — tag every name |
| Put `ansible_host` / `ansible_user` in `hosts.yml` | Move to `host_vars/` / `group_vars/` |
| Write `rejectattr(...) \| map(...)` | `json_query` |
| Add `ignore_errors` to survive `--check` | `when: not ansible_check_mode` or `failed_when` |
| Set `changed_when: false` on an upload/create | Let the module report change |
| Hardcode `no_log: true` on a task | `no_log: "{{ redact_sensitive_output \| default(true) }}"` |
| Write a `debug` task without `tags: [debug]` | Add the tag |
| Skip headers because the file is short | Headers are required regardless of size |
| Assume lint-clean means done | Lint doesn't check house style — walk the table above |
| Document an `ansible*` command (`ansible`, `ansible-playbook`, `ansible-vault`, ...) with one `-e '{...}'` JSON blob, or several options on one line | One option per line; `-e key=value` per scalar; JSON only for a list/dict (see Documenting Commands) |
