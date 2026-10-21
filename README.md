# Docker Role

An Ansible role for installing Docker Engine on Ubuntu 24.04
and newer supported Ubuntu releases.

## Features

- Configures the official Docker APT repository.
- Installs Docker Engine and Docker CLI.
- Installs Docker Buildx and Docker Compose plugins.
- Enables and starts the Docker service.
- Verifies the Docker CLI installation.

## Requirements

- Ansible installed on the control node.
- SSH connectivity to target hosts.
- A user with sudo privileges on target hosts.
- Supported Ubuntu version on target hosts.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| docker_apt_repository_url | https://download.docker.com/linux/ubuntu | Docker APT repository |
| docker_apt_key_url | https://download.docker.com/linux/ubuntu/gpg | Docker signing key URL |
| docker_apt_keyring_path | /etc/apt/keyrings/docker.asc | GPG key location |
| docker_packages | See defaults/main.yml | Packages to install |
| docker_service_name | docker | Docker service name |
| docker_service_enabled | true | Enable service at boot |
| docker_service_state | started | Desired service state |
| docker_apt_cache_valid_time | 3600 | APT cache validity in seconds |

## Example Playbook

```yaml
---
- name: Install Docker
  hosts: vms
  become: true
  roles:
    - docker