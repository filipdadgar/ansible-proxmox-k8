This project has been inspired by https://github.com/zimmertr/Bootstrap-Kubernetes-with-QEMU

<hr>

## This is half of a two-repo setup

This repo provisions the infrastructure layer: Proxmox VMs -> base OS -> a bare `kubeadm` +
Flannel cluster. Once that's up, **[k8gitops](https://github.com/filipdadgar/k8gitops)** (a
separate, private repo) takes over everything above that layer via Flux GitOps - all apps,
MetalLB, DNS, storage classes, etc. Don't use this repo's optional playbooks for things
k8gitops already manages (see the table below) - they install older, disconnected copies of
the same things with placeholder config that doesn't match the live network.

## Summary
Declaratively build a Kubernetes cluster on Proxmox using Ansible and QEMU - the real cluster
this repo provisions is a 4-node cluster (1 control-plane + 3 workers) spread across a
**2-node Proxmox cluster** (two separate physical hypervisors). `vars.yml`'s `k8s_workers`
list can hold any number of worker entries; it isn't hardcoded to a fixed count.

1. K8s control-plane and worker nodes (`kubeadm`)
2. Flannel CNI
3. Per-node hardware passthrough support (vGPU, raw PCI GPU, USB) where a node's `vars.yml`
   entry needs it

**Approximate deployment time:** 25-30 minutes, plus a few more for `configure_node3_bluetooth.yml`
if your node needing USB/Bluetooth passthrough is part of the build.


## Requirements
1. A Proxmox server, or (as in this repo's actual deployment) a Proxmox cluster. List every
   hypervisor host under `[proxmox_server]` in `inventory.ini`, not just one, and make sure
   each node's `vars.yml` entry sets `host` to the hypervisor it should actually live on.
2. A DNS server for your nodes' resolvers (see the `ns` field in `vars.yml`'s node entries) -
   not technically required, but domain-based features elsewhere in your stack may depend on
   it resolving correctly for host-networked workloads (see k8gitops `CLAUDE.md`'s
   "Node-level DNS is not GitOps-managed" section for why this matters more than it looks).
3. An NFS (or similar) server if any app depends on `vars.yml`'s `nfs_mounts` - this repo
   mounts those at the OS level on every node via `install_base_packages.yml`, independent of
   Kubernetes.
4. Ansible 2.7.0+. Known incompatibility with a previous build.
- For MAC: ``` brew install ansible-lint ansible python-markupsafe ```
5. ```ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa to generate SSH keys for your nodes. And copy it to proxmox id_rsa.pub ```
6. Post installation for demo purposes:
- Run the following from the master node:

```

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

## Instructions
**Required:**

1. Modify the `vars.yml` file with values specific to your environment - in particular, make
   sure `k8s_master_node` and every entry in `k8s_workers` has the right `host` (which
   hypervisor), `id` (VM ID), `ip`, and `stg` (storage pool - pool names can differ per
   hypervisor in a Proxmox cluster, as they do here).
2. Provision DNS A records for the IP addresses & hostnames you defined for your nodes.
3. Modify `inventory.ini` to reflect your real Proxmox hosts, master/worker IPs, and the
   location of the SSH keys used to connect to the nodes.
4. Run the deployment: `ansible-playbook -i inventory.ini site.yml --ask-pass`
5. If any node needs hardware passthrough support beyond what `extra_qm_args` in `vars.yml`
   already covers (e.g. this repo's own `prox-node3` Bluetooth dongle), run its dedicated
   playbook next - e.g. `ansible-playbook -i inventory.ini playbooks/main/configure_node3_bluetooth.yml`.
6. After deployment, a `~/.kube` directory will be created on your workstation. You can use
   the `config-proxmox` file within to interact with your cluster.

**Destructive playbooks require explicit confirmation**, since this repo targets a real,
persistent cluster, not a disposable test environment: `playbooks/main/bootstrap_kubernetes.yml`
(unconditionally runs `kubeadm reset -f` on the master) and
`playbooks/optional/delete_all_resources.yml` (destroys every node VM) both require
`-e confirm_destroy=true` or they refuse to run. Don't set that flag against a cluster you
don't actually intend to wipe.

**Optional:**

*To enable an optional feature, fill in the additional parameters in `vars.yml` and execute a playbook listed below.*

| Feature | Command | Status |
| ------- | ------- | ------ |
| [MetalLB Load Balancer](https://metallb.universe.tf) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_metallb.yml` | **Superseded** - MetalLB is Flux-managed from k8gitops with the real IP pool; don't run this against the live cluster |
| [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_ingress-nginx.yml` | **Superseded** - no app in k8gitops uses Ingress; every app is LoadBalancer + external-dns |
| [NFS backed persistent storage](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_nfs_provisioner.yml` | **Superseded** (as a StorageClass) - every PVC in k8gitops uses `synology-iscsi-storage-v2`. Unrelated to the static NFS fstab mounts this repo still actively manages. |
| [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_dashboard.yml` | Likely unused - nothing in k8gitops references it; verify before running |
| [DataDog agents](https://docs.datadoghq.com/integrations/kubernetes/) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_datadog.yml` | Unverified against current cluster |
| [Unattended Upgrades](https://wiki.debian.org/UnattendedUpgrades) | `ansible-playbook -i inventory.ini playbooks/optional/enable_unattended_upgrades.yml` | Unverified against current cluster |

Kept all of the above rather than deleted, even the superseded ones, in case they're useful
reference for a from-scratch cluster that hasn't been pointed at k8gitops yet - just don't run
them against this cluster.


## TODO
1. Add better support for plugins and later version of k8
2. Add better support for pxe boot of proxmox.
3. `files/templates/{Datadog,NFS,NGINX-Ingress,Service_Account}/*.j2` haven't been
   individually re-verified against the live cluster the way MetalLB's and the Dashboard's
   were - treat as unverified until checked.
