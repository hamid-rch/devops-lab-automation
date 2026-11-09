# MinIO Stack — 4-node / 8-drive Ansible Lab

This role deploys one **distributed MinIO cluster** across exactly four inventory
hosts. Each host runs one MinIO container and contributes two data endpoints.
The cluster therefore has eight MinIO drives total.

This is **Distributed MinIO with Erasure Coding**, not bucket replication
between independent MinIO deployments.

## Lab storage design

Each Vagrant node has a dedicated 2 GiB secondary disk at `/dev/sdb`. The role
creates a GPT table and two 500 MiB XFS partitions on that disk:

- `/dev/sdb1` -> `/srv/minio/data1` -> `/data1`
- `/dev/sdb2` -> `/srv/minio/data2` -> `/data2`

Partition device names, disk path, partition size, filesystem, mount paths,
and container paths are configurable in `defaults/main.yml`.
The role mounts partitions by filesystem UUID so `/etc/fstab` does not depend
only on transient device naming.

**Destructive safety:** on first run, the configured data disk must be blank.
The role refuses to partition a disk with existing signatures, refuses
unexpected partition layouts, and refuses to format a partition containing a
non-XFS filesystem. It does not touch the OS disk. The disk is dedicated to
MinIO and its contents will be used for MinIO data.

These are two partitions on one virtual disk, not two independent physical
disks. A failure of the virtual disk removes both drives on that node. This is
appropriate for a disposable lab, not production fault-isolation testing.

The role uses XFS because MinIO's distributed documentation recommends
filesystems with the required direct-I/O and sync semantics; mountpoint
ownership defaults to UID/GID 1000 and is configurable with
`minio_data_uid` / `minio_data_gid`.

## Topology and availability

- Four hosts in `minio_nodes` (configurable through `minio_node_group`).
- Two drive endpoints per host, eight total.
- MinIO automatically selects the erasure set and parity at initialization.
  For the 4-node / 2-drive-per-node sizing, MinIO's sizing guide lists an
  8-drive set with default EC:4; its documented tolerance is two servers for
  reads and one server for writes. Confirm against the exact MinIO build in
  use; do not manually assume a parity value from a different release.
- All four nodes must use identical root credentials, the same image, the same
  port, and the same drive topology.
- The MinIO cluster endpoints must be reachable bidirectionally between all
  four nodes on TCP 9000.
- TCP 9001 is exposed on all nodes for direct troubleshooting. Normal client
  access should use Traefik.

## Traefik ingress

Traefik is deployed **only on the first host** in `minio_nodes` (the primary
node). It routes:

- `s3.lab` -> all four MinIO API endpoints on port 9000
- `minio.lab` -> all four MinIO Console endpoints on port 9001

This provides one lab ingress address, not a highly available load balancer.
For production, put an external HA load balancer / VIP in front of the MinIO
nodes. The lab Traefik instance itself is a single point of failure. Configure
DNS or `/etc/hosts` so both names resolve to the primary node's IP.

Traefik uses HTTP on port 80 in this lab. Add TLS and an authentication
strategy before exposing it outside a trusted lab network. Do not expose the
MinIO root credentials or the unauthenticated HTTP endpoint to the public
internet.

## Important migration warning

The previous role used `/srv/minio/data` as a single data path. This version
uses two new paths. It deliberately stops if the legacy directory contains
files, rather than silently hiding or abandoning old data. Review and back up
that directory before proceeding. Setting `minio_allow_legacy_data_dir=true`
only bypasses the safety guard; it does **not** migrate old objects.

The drive preparation tasks also refuse to format a backing file with an
unexpected filesystem, resize an existing backing file, or mount over a
non-empty unmounted directory.

## Variables

Image references are preserved from the previous role. Change settings in
`defaults/main.yml` or override them in `group_vars`:

- `minio_node_group`, `minio_expected_nodes`
- `minio_data_disk`, `minio_partition_size_mib`, `minio_partition_start_mib`
- `minio_data_drives`, `minio_drive_fstype`
- `minio_stack_dir`, `minio_api_port`, `minio_console_port`
- `minio_api_hostname`, `minio_console_hostname`
- `minio_traefik_port`, `minio_traefik_entrypoint_address`
- `minio_lab_bucket`

Credentials remain external to this role and should stay in the existing
Vault/group variables as `minio_root_user` and `minio_root_password`.

## Deploy

Run the existing project playbook against all four nodes. Do not run this role
on only one node: all four MinIO processes must start with the same distributed
endpoint list.

Example (adapt to your existing playbook and Vault setup):

```bash
ansible-playbook playbooks/site.yml --limit minio_nodes --ask-become-pass --ask-vault-pass
```

If Vault or become is already configured in `ansible.cfg` / the playbook, use
the project's existing invocation.

## Test harness

Run the included verification playbook after deployment:

```bash
ansible-playbook roles/minio_stack/tests/verify.yml --ask-become-pass --ask-vault-pass
```

It checks:
- Compose configuration parsing on every node.
- MinIO container state on every node.
- Both XFS mountpoints on every node.
- MinIO live health on every node.
- Cluster health and read/write quorum from the first node.
- Bucket versioning status for the configured lab bucket.
- A temporary S3 object upload, stat, and version-aware cleanup.

The verification playbook expects the same inventory group and credential
variables as the deployment role.

## Useful manual checks

```bash
# Container state and logs
ansible minio_nodes -i inventory/hosts.ini -b -m shell -a 'docker compose -f /opt/minio-stack/compose.yml ps'
ansible minio_nodes -i inventory/hosts.ini -b -m shell -a 'docker logs --tail 100 minio'

# Verify mounted XFS filesystems
ansible minio_nodes -i inventory/hosts.ini -b -m shell -a 'findmnt /srv/minio/data1 /srv/minio/data2'

# Cluster health (run against any node)
curl -i http://MINIO_NODE_IP:9000/minio/health/cluster
curl -i http://MINIO_NODE_IP:9000/minio/health/cluster/read
curl -i http://MINIO_NODE_IP:9000/minio/health/cluster/write

# S3 API and Console via the primary node's Traefik ingress
curl -i -H 'Host: s3.lab' http://PRIMARY_NODE_IP/
curl -i -H 'Host: minio.lab' http://PRIMARY_NODE_IP/
```

## Destructive reset

A clean reset destroys all objects, versions, bucket metadata, and IAM data.
Only do this in a disposable lab after stopping MinIO on all four nodes. The
following commands erase the MinIO secondary disk; verify `/dev/sdb` first:

```bash
docker compose -f /opt/minio-stack/compose.yml down
sudo umount /srv/minio/data1 /srv/minio/data2
sudo wipefs --all /dev/sdb1 /dev/sdb2
sudo parted --script /dev/sdb mklabel gpt
```

Repeat on all four hosts. This is destructive and should only be used when you
intend to discard all MinIO data.
