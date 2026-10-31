# gitlab_stack

Deploys GitLab CE using Docker Compose, preserves existing `/data/gitlab` bind mounts, adds Docker `extra_hosts` for `mimir.lab`, and manages Prometheus `remote_write` and external labels in persistent `/data/gitlab/config/gitlab.rb`.

## Example inventory

```ini
[gitlab]
ubuntu2404E ansible_host=192.168.56.34 ansible_user=vagrant
```

## Example playbook entry

```yaml
- name: Install and configure GitLab
  hosts: gitlab
  become: true
  roles:
    - docker
    - gitlab_stack
```

## Run

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --limit gitlab --check --diff
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --limit gitlab
```

**Important:** The Omnibus attributes `prometheus['remote_write']` and `prometheus['external_labels']` must be validated against the exact GitLab image and configuration already proven in your lab. Before applying to an existing instance, compare the managed block with your current `gitlab.rb`; if those keys are already defined elsewhere, remove the duplicate assignments. Do not run in check mode on a fresh host that has no GitLab configuration yet (first-boot creation requires an actual run).

Compose changes can recreate the container and briefly interrupt GitLab. This role does not delete persistent data. If the existing GitLab container was created outside this Compose project, inspect its labels and backup before adoption.
