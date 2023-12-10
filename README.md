# Kudos to https://github.com/zimmertr/Bootstrap-Kubernetes-with-QEMU

This project has been inspired by https://github.com/zimmertr/Bootstrap-Kubernetes-with-QEMU

<hr>


## Summary
Declaratively build a 2(with option for more) node Kubernetes cluster on Proxmox using Ansible and QEMU. Optionally enable advanced features including ingress, load balancing, unattended upgrades, the Dashboard, etc.

1. K8 master and workers
2. Metallb with flannel
3. Ingress Class
4. Storage Class

**Approximate deployment time:** 25 minutes


## Requirements
1. Proxmox server
2. DNS Server*
3. Ansible 2.7.0+. Known incompatibility with a previous build.
- For MAC: brew install ansible-lint ansible python-markupsafe
- *A DNS server is not technically required, it is possible to manually add entries corresponding to your node hostnames to your Proxmox's hosts file.
4. ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa to generate SSH keys for your nodes. And copy it to proxmox id_rsa.pub
5. Post installation for demo purposes:
- Run the following:
´´´
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
´´´


## Instructions
**Required:**

1. Modify the `vars.yml` file with values specific to your environment.
2. Provision DNS A records for the IP Addresses & Hostnames you defined for your nodes in the `vars.yml` file.
3. Modify the `inventory.ini` file to reflect your chosen DNS records and the location of the SSH keys used to connect to the nodes.
4. Run the deployment: `ansible-playbook -i inventory.ini site.yml --ask-pass`
5. After deployment, a `~/.kube` directory will be created on your workstation. You can use the `config` file within to interact with your cluster.

**Optional:**

*To enable an optional feature, fill in the additional parameters in `vars.yml` and execute a playbook listed below.*

| Feature | Command | Requirements |
| ------- | ------- | ------------ |
| [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_dashboard.yml` | |
| [NFS backed persistent storage](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_nfs_provisioner.yml` | |
| [MetalLB Load Balancer](https://metallb.universe.tf) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_metallb.yml` | |
| [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_ingress-nginx.yml` | [MetalLB](https://metallb.universe.tf/) or other Load Balancer integration |
| [DataDog agents](https://docs.datadoghq.com/integrations/kubernetes/) | `ansible-playbook -i inventory.ini playbooks/optional/deploy_datadog.yml` | |
| [Unattended Upgrades](https://wiki.debian.org/UnattendedUpgrades) | `ansible-playbook -i inventory.ini playbooks/optional/enable_unattended_upgrades.yml` | |



## TODO
1. Add better support for plugins
2. Add better support for pxe boot of proxmox.
