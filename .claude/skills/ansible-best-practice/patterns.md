# Ansible Patterns (worked examples)

Companion to `SKILL.md`. Each example follows every house-style rule.

## Role Skeleton

```
roles/<role_name>/
  defaults/main.yml         # every input, prefixed <role_name>_, each commented
  meta/main.yml             # galaxy_info
  meta/argument_specs.yml   # typed inputs; required: true; no_log: true on secrets
  tasks/main.yml            # = header, VALIDATE section first, then sections
  handlers/main.yml         # only if the role restarts/reloads something
  templates/*.j2
```

`meta/argument_specs.yml`:

```yaml
---
argument_specs:
  main:
    short_description: Upload qcow2 images to Glance in parallel
    options:
      glance_image_sync_images:
        type: list
        elements: path
        required: true
        description: Absolute qcow2 paths on the conversion host.
      glance_image_sync_password:
        type: str
        required: true
        no_log: true
        description: Keystone password (vault or extra var).
```

## Async Fan-Out With Guaranteed Cleanup

```yaml
    # --------------------------------------------------------------------------
    # Upload: Start all Glance uploads in parallel
    # --------------------------------------------------------------------------
    # async can't run under --check: wrap this and the wait below in a
    # `block:` with `when: not ansible_check_mode` (omitted here for brevity).
    - name: OPENSTACK | Start Glance image uploads
      openstack.cloud.image:
        cloud: "{{ glance_image_sync_cloud }}"
        name: "{{ item | basename | regex_replace('\\.qcow2$', '') }}"
        filename: "{{ item }}"
        disk_format: qcow2
        container_format: bare
        state: present
        timeout: "{{ glance_image_sync_upload_timeout }}"  # module default is 180 s
      loop: "{{ glance_image_sync_images }}"
      loop_control:
        label: "{{ item | basename }}"
      async: "{{ glance_image_sync_upload_timeout }}"
      poll: 0
      register: _upload_jobs
      # no changed_when here: the module's own change status is the truth

    - name: OPENSTACK | Wait for uploads and clean up job files
      block:
        - name: OPENSTACK | Wait for Glance uploads to finish
          ansible.builtin.async_status:
            jid: "{{ item.ansible_job_id }}"
          loop: "{{ _upload_jobs.results }}"
          loop_control:
            label: "{{ item.item | basename }}"
          register: _upload_results
          until: _upload_results.finished
          retries: "{{ (glance_image_sync_upload_timeout / 30) | int }}"
          delay: 30
      always:
        - name: OPENSTACK | Clean up async job files
          ansible.builtin.async_status:
            jid: "{{ item.ansible_job_id }}"
            mode: cleanup
          loop: "{{ _upload_jobs.results }}"
          loop_control:
            label: "{{ item.item | basename }}"
          changed_when: false
```

Build the file name → image ID map (after the block):

```yaml
    - name: SET | Map image file name to Glance image ID
      ansible.builtin.set_fact:
        glance_image_sync_image_ids: >-
          {{ glance_image_sync_image_ids | default({})
             | combine({(item.item.item | basename): item.image.id}) }}
      loop: "{{ _upload_results.results }}"
      loop_control:
        label: "{{ item.item.item | basename }}"
```

## Secret-Handling API Call

```yaml
    # --------------------------------------------------------------------------
    # Authentication: Get project-scoped Keystone token
    # --------------------------------------------------------------------------
    - name: AUTH | Request project-scoped Keystone token
      ansible.builtin.uri:
        url: "{{ keystone_url }}/v3/auth/tokens"
        method: POST
        body_format: json
        body:
          auth:
            identity:
              methods: [password]
              password:
                user:
                  name: "{{ keystone_user }}"
                  domain: { name: Default }
                  password: "{{ keystone_password }}"
        status_code: 201
      register: _keystone_token
      no_log: "{{ redact_sensitive_output | default(true) }}"
      delegate_to: localhost
```

## Vendor CLI Returning JSON (check-safe)

```yaml
    # --------------------------------------------------------------------------
    # Health: Verify v2v workers
    # --------------------------------------------------------------------------
    - name: COMMAND | Run v2v health check
      ansible.builtin.command:
        cmd: /opt/v2v/bin/v2v-healthcheck --json
      register: _v2v_health_raw
      changed_when: false
      check_mode: false

    - name: VALIDATE | Assert v2v status is ok and all workers are ready
      ansible.builtin.assert:
        that:
          - _v2v_health.status | default('') == 'ok'
          - _not_ready | length == 0
        fail_msg: >-
          v2v health check failed on {{ inventory_hostname }}:
          status={{ _v2v_health.status | default('missing') }},
          not ready={{ _not_ready | join(', ') }}
      vars:
        _v2v_health: "{{ _v2v_health_raw.stdout | from_json }}"
        _not_ready: "{{ _v2v_health.workers | default([]) | community.general.json_query(\"[?state!='ready'].name\") }}"
```

## Service From Template, Restart Only On Change, Check-Safe

```yaml
    - name: SERVICE | Install v2v-worker unit file
      ansible.builtin.template:
        src: v2v-worker.service.j2
        dest: /etc/systemd/system/v2v-worker.service
        owner: root
        group: root
        mode: "0644"
      register: _v2v_unit
      notify: HANDLER | Restart v2v-worker

    - name: SERVICE | Enable and start v2v-worker
      ansible.builtin.systemd_service:
        name: v2v-worker
        enabled: true
        state: started
        daemon_reload: "{{ _v2v_unit is changed }}"
      # In --check on a fresh host the unit was never written; skip rather than ignore_errors
      when: not (ansible_check_mode and _v2v_unit is changed)

    - name: HANDLER | Flush handlers before health check
      ansible.builtin.meta: flush_handlers

  handlers:
    - name: HANDLER | Restart v2v-worker
      ansible.builtin.systemd_service:
        name: v2v-worker
        state: restarted
        daemon_reload: true
```

## Inventory Layout

```
inventory/
  hosts.yml                       # membership only
  group_vars/conversion_hosts/
    vars.yml                      # ansible_user: ubuntu, shared settings
    vault.yml                     # ansible-vault encrypted secrets
  host_vars/conv01/vars.yml       # ansible_host: 10.0.0.11
  host_vars/conv02/vars.yml       # ansible_host: 10.0.0.12
```

```yaml
# hosts.yml
---
all:
  children:
    conversion_hosts:
      hosts:
        conv01:
        conv02:
```
