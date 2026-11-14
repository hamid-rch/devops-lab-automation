# kubernetes_stack Ansible role

A role-only deployment for the user's nine-node VirtualBox lab:

- 3 Load Balancers: HAProxy + Keepalived (VRRP unicast), one shared API VIP.
- 3 Control Plane nodes: kubeadm with stacked etcd.
- 3 Worker nodes.
- Kubernetes `v1.37.1` and Calico `v3.33.0`.
- Ubuntu Server 24.04, containerd already installed, Nexus client/proxy configuration already managed elsewhere.

This folder intentionally contains **no inventory, hosts file, or playbook**. The role expects an existing inventory and is designed to be called from a play that includes all nine Kubernetes hosts.

## Inventory group names

Defaults are:

```yaml
k8s_lb_group: loadbalancers
k8s_control_plane_group: masters
k8s_worker_group: workers
k8s_control_plane_leader: master1
```

Override these role variables if the existing inventory uses different group names. The groups must contain exactly three hosts each and be disjoint. `k8s_control_plane_leader` must be a member of the control-plane group.

## Network discovery

By default, each host's address is discovered from its gathered IPv4 addresses using:

```yaml
k8s_node_ip_regex: '^192\.168\.56\.'
```

This matches the Vagrant host-only network used in this lab. If the inventory already defines a per-host `k8s_node_ip`, that value takes precedence. Calico IP autodetection is also pinned to `k8s_node_network_cidr` (`192.168.56.0/24`) so it does not accidentally select VirtualBox's NAT interface. The Keepalived interface defaults to `enp0s8`; override `k8s_lb_interface` if VirtualBox uses another interface name.

API endpoint:

- VIP: `192.168.56.100`
- Port: `6443`

## Nexus assumptions

The role does not recreate Nexus repositories or alter the user's Nexus client configuration.

- Kubernetes APT packages (`kubelet`, `kubeadm`, `kubectl`) must be available from the APT source already configured by the Nexus client role. That APT source should point at the Kubernetes `v1.37` package repository.
- kubeadm core images use `k8s_image_repository`, defaulting to `nexus.lab/k8s-proxy` (the path used in the user's existing tests).
- Calico manifest download defaults to upstream GitHub Raw. Set `k8s_calico_manifest_url` to the exact Nexus Raw Proxy URL if the lab has one.
- Calico v3.33.0 images are hosted at `quay.io/calico`. Ensure containerd's existing `quay.io` registry mirror points to the Nexus Quay proxy and that the Calico images are cached. If the Nexus Docker proxy uses a path-based prefix, set `k8s_calico_image_prefix` to that prefix, for example `nexus.lab/quay-proxy/calico` (only if that is the actual connector path).
- The role requires an existing `/etc/containerd/config.toml` and intentionally preserves the complete Containerd/Nexus registry configuration, including `/etc/containerd/certs.d`. It only changes `SystemdCgroup = false` to `true` and restarts Containerd if needed. It does not rewrite the sandbox/pause image setting because Containerd 1.x and 2.x use different configuration layouts; ensure the existing sandbox image can be pulled through your Nexus `registry.k8s.io` mirror.

## Important variables

| Variable | Default | Purpose |
|---|---|---|
| `k8s_version` | `1.37.1` | Kubernetes patch version expected in APT |
| `k8s_image_repository` | `nexus.lab/k8s-proxy` | kubeadm core image repository |
| `k8s_api_vip` | `192.168.56.100` | Keepalived VIP and kubeadm endpoint |
| `k8s_lb_interface` | `enp0s8` | VirtualBox host-only interface on LB nodes |
| `k8s_node_network_cidr` | `192.168.56.0/24` | Node network used by Calico IP autodetection |
| `k8s_pod_subnet` | `10.244.0.0/16` | Pod network; must not overlap any host network |
| `k8s_service_subnet` | `10.96.0.0/12` | Service network |
| `k8s_calico_version` | `v3.33.0` | Calico manifest/image version |
| `k8s_calico_manifest_url` | upstream raw URL | Override with Nexus Raw Proxy URL |
| `k8s_calico_image_prefix` | `quay.io/calico` | Override if using path-based Quay proxy |
| `k8s_keepalived_auth_pass` | `K8sHA123` | VRRP password; override through Ansible Vault |

## Invocation

Call the role from an existing play that includes the nine Kubernetes nodes. No inventory or playbook is supplied by this role.

```yaml
- name: Configure Kubernetes lab
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: kubernetes_stack
      vars:
        # Change these only if your existing inventory group names differ.
        k8s_lb_group: loadbalancers
        k8s_control_plane_group: masters
        k8s_worker_group: workers
```

If the existing inventory contains unrelated hosts, they are skipped by the role; only the three configured groups are managed. The play still needs to target all nine Kubernetes nodes so the role can orchestrate the bootstrap sequence.

## What the role does

1. Validates group counts, Ubuntu 24.04, node IP discovery, and the LB interface.
2. Configures HAProxy TCP load balancing to all three API servers.
3. Configures Keepalived unicast VRRP with priorities 150/140/130 and HAProxy health tracking.
4. Disables swap, loads `overlay` and `br_netfilter`, and applies Kubernetes sysctls on the six Kubernetes nodes.
5. Verifies the preinstalled containerd service, preserves Nexus registry settings, and enables the systemd cgroup driver.
6. Installs and version-checks `kubelet`, `kubeadm`, and `kubectl` from the already-configured APT source; holds those packages.
7. Runs kubeadm init on the selected first control plane using stacked etcd and the API VIP.
8. Downloads/applies Calico and changes its default IP pool to match `k8s_pod_subnet`.
9. Generates a short-lived join token and uploads control-plane certificates; joins the other two control planes sequentially (explicitly advertising each node's host-only IP), then the three workers.
10. Waits for Calico rollout and all nodes to become Ready.

## Network requirements

At minimum, allow these paths on the cluster network:

- TCP `6443`: all nodes and clients to the API VIP; LB nodes to all control-plane nodes.
- TCP `2379-2380`: between control-plane nodes for stacked etcd.
- TCP `10250`: control-plane components to kubelets as required.
- Calico default manifest uses BGP/IP-in-IP defaults: permit TCP `179` and IP protocol `4` between Kubernetes nodes. If the Calico data plane is changed to VXLAN, allow UDP `4789` instead and configure the manifest accordingly.
- VRRP unicast packets between the three LB nodes (IP protocol `112`).

Ensure UFW or any external firewall does not block these paths. On VirtualBox, all nodes must share the same host-only L2 network for this VIP design.

## Operational notes

- Apply the role to a fresh cluster. It is idempotent for an already initialized leader and nodes already joined, but it is not a cluster reset/rebuild role.
- kubeadm certificate upload keys expire; the role creates a fresh key on each run before joining missing control-plane nodes.
- `k8s_keepalived_auth_pass` is only VRRP protocol authentication, not a substitute for network security. Store a non-default value in Ansible Vault.
- The nine VMs share one physical host, so this demonstrates service/VM failover, not physical-host HA.
- The role does not install Docker Engine, a cloud controller, an Ingress controller, a CSI driver, or application workloads.
- Review `haproxy.cfg`, `keepalived.conf`, containerd config, and the selected Nexus URLs in a lab run before treating the cluster as production-ready.
