# Deploying Talos Linux Cluster in Proxmox using Ansible
## Motivation
I wanted to an automated way to produce a Kubernetes cluster to learn more about deploying services on K8s. While tools like kubespray exist, I also wanted to try my hand at tinkering with Talos Linux as my K8s host. 

I knew that any cluster I build would get mucked up eventually, trial by error and all, so I wanted to be able to quickly redeploy a cluster rather than following manual steps repeatedly.

## Contents

### `update-talos.yml`
Updates `talosctl` on your local machine by downloading the latest `talosctl` binary for your platform and moving into your path. Also downloads the latest bare metal ISO for Talos Linux and stores it onto Proxmox.

Variables to update:
  - `talos_version_offset` - _default: 0_ - how many versions prior to latest you want to go back for talos. 0 == latest.
  - `arch` - _default: amd64_ - update this for both Proxmox and localhost depending on your platform
  - `distro` - _default: arm64_ - distribution target for localhost

### `config-network.yml`
Generates static IP addresses for the Talos cluster and configures static DHCP leases in pfSense. Very crude string generation. Requires `pfsensible` Ansible collection. 

Variables to update:
  - `subnet_24` - _default: 192.168.1._  - the /24 subnet to place your nodes on
  - `reset_network` - _default: no_ - whether or not you want to refresh the static DHCP leases in pfSense

### `deploy-talos.yml`
Deletes old Talos VMs from Proxmox, and redeploys Talos VMs using the bare metal iso downloaded in `update-talos.yml`. Generates Talos configurations, applies them to all nodes, then bootstraps `etcd` and verifies that the K8s cluster is up and running.

Edit `vars/talosctl.yml` to configure where to store configurations and what the Talos cluster name. 

### `build-cluster.yml`
Runs all three previous playbooks but only if needed (i.e. we don't know what version to use or we don't have IP addresses assigned). 

This relies on fact caching a JSON file, edit `ansible.cfg` to modify the timeout (_default is 1 hour_).

## Requirements
Prepare a Python virtualenv and install the `requirements.txt`:

```bash
python3 -m virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
```


Install `pfsensible` collection:

```bash
ansible-galaxy collection install pfsensible.core -p ./collections
```

## pfSense Integration
In order to run Ansible against pfSense, you need to enable `sudo` via the Package Manager in the Web GUI. I also recommend creating a new user named `ansible` to use instead of your own user or admin account.

## Running the Playbooks

To build a new cluster, edit the variables described above and run the following command:

```
ansible-playbook -kK build-cluster.yml
```

This assumes that both `proxmox` and `pfsense` have users as described in `inventory.yml`, and that their SSH passwords are the same. Alternatively, you can configure SSH keys. 