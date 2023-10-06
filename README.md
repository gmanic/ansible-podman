# sleif.podman

This role installs and manages Podman.

The role supports different `podman_operations`.

```yaml
podman_operations:
  - podman_generate_systemd
  - podman_init_vars
  - podman_install
  - podman_pod_create
```

- podman_install: installs Podman and prepares rootfull and rootless operations
- podman_init_vars: provides Podman environments for other roles; it will provide:
  - _container_storage_dir_base_local
  - _container_storage_dir_base
  - _group, _owner
  - _systemd_scope
  - _systemd_service_files_dir
  - _xdg_runtime_dir
- podman_generate_systemd: provides systemd unit file management; to ensure that Podman containers will work smoothly with systemd, it is required to just create `state: created` the containers with `containers.podman.podman_container` and call `podman_generate_systemd` after the pod creation
- podman_pod_create: creates a Podman pod to be used from other roles

## Requirements

None

## Role Variables

- podman_operation

## Dependencies

None

## Example Playbook

- install Podman:

```yaml
- name: VM caddy.example.com
  hosts: "caddy.example.com"
  user: root

  vars:
    podman_networks:
      podman_network_root:
        podman_network_name: 'podman_custom'
        podman_network_subnet: '10.0.0.0/24'
        podman_network_gateway: '10.0.0.1'
        podman_network_iprange: '10.0.0.128/25'
      podman_network_rootless:
        podman_network_name: 'podman_custom'
        podman_network_subnet: '10.0.1.0/24'
        podman_network_gateway: '10.0.1.1'
        podman_network_iprange: '10.0.1.128/25'
    podman_rootless: true
    podman_network_name: "{{ podman_networks.podman_network_rootless.podman_network_name }}"

  roles:
    - {role: sleif.podman, tags: "podman_role",
       podman_operation: "podman_install"}
```

- Calls from inside other roles:
  - Initialize the Podman environment

  ```yaml
  # the next task will return:
  # - _container_storage_dir_base_local
  # - _container_storage_dir_base
  # - _group, _owner
  # - _systemd_scope
  # - _systemd_service_files_dir
  # - _xdg_runtime_dir
  - name: Include podman_init_vars.yml from sleif.podman
    ansible.builtin.include_tasks: "{{ lookup('first_found', params) }}"
    vars:
      params:
        files: sleif.podman/tasks/includes/podman_init_vars.yml
        paths: "{{ lookup('config', 'DEFAULT_ROLES_PATH') }}"
    tags: always
  ```

  - Create a Podman pod:

  ```yaml
  - name: Include role sleif.podman podman_operation podman_pod_create
    ansible.builtin.include_role:
      name: sleif.podman
      apply:
        tags:
          - podman_pod_create
    vars:
      podman_operation: podman_pod_create
    tags: always
  ```

  - Trigger systemd unit file creation and service enable/start

  ```yaml
  - name: Include role sleif.podman podman_operation podman_generate_systemd
    ansible.builtin.include_role:
      name: sleif.podman
      apply:
        tags:
          - podman_generate_systemd
    vars:
      podman_operation: podman_generate_systemd
      target: "{{ pod_name if pod_name | d('') is truthy else container_name }}"
    tags: always
  ```

## License

MIT

## Author Information

Created in 2023 by Sebastian Berthold
