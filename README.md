# Two-node OpenStack VPS lab

Run a small VPS provider from the command line. As the operator, you publish two server plans. As a customer, you choose a plan, launch an instance, and log in with your own SSH key. The deployment is complete when the customer can run a command on the VPS and the operator can confirm which compute host runs it.

## Topology

| Role | What it does | What you provide |
|---|---|---|
| Ansible control machine | Prepares both OpenStack nodes over SSH | A system with Bash, OpenSSH and `ansible-core` |
| `controller` | Runs the OpenStack APIs, scheduler, image service and OVN network gateway | A dedicated Ubuntu Server 24.04 node |
| `compute01` | Runs customer instances using Nova Compute and KVM | A dedicated Ubuntu Server 24.04 node with a working `/dev/kvm` |

The two Ubuntu nodes can be physical servers or VMs. For a virtual compute node, enable nested virtualization in the outer hypervisor. As a starting allocation for one small instance, allow 3 vCPUs, 8 GiB RAM and 45 GiB disk for the controller, and 2 vCPUs, 4 GiB RAM and 35 GiB disk for the compute node. Guest memory comes from the compute node’s allocation. DevStack modifies networking and system services on both nodes; use hosts dedicated to the lab.

## 1. Define the management network

Give both nodes stable IPv4 addresses on a mutually reachable network, outbound access to package repositories, and an SSH account with sudo privileges. Set their **short hostnames** to `controller` and `compute01`. The playbook checks each hostname and management address before making changes.

Edit [inventory.ini](inventory.ini). Keep the host aliases and group names used by the playbook. Set `ansible_host` to each node’s actual management IPv4 address and `ansible_user` to your SSH login. The shipped addresses and login are examples.

| Setting | Repository default | Where to change it |
|---|---|---|
| Controller and compute management IPs | `192.168.56.11`, `192.168.56.12` | `inventory.ini` and the nodes’ network configuration |
| SSH login | `lab` | `inventory.ini` |
| Tenant address range | `10.77.0.0/22` | `IPV4_ADDRS_SAFE_TO_USE` in [templates/local.conf.j2](templates/local.conf.j2) |
| Floating IP range | `172.24.4.0/24` | `FLOATING_RANGE` in the same template |

Choose ranges that do not overlap with each other or with routes already in use. The supplied configuration puts the external network gateway on the controller. The customer SSH test runs **from the controller**, where the floating IP is reachable through `br-ex`. Internet-routable addresses are unnecessary for this lab.

## 2. Prepare both nodes with Ansible

On the **Ansible control machine**, authorize your management SSH public key on both nodes and verify their host key fingerprints through a trusted console. Run the following from the repository root. Enter the path to your existing management **private** key at the prompt. If it has a passphrase, load it into your SSH agent first.

```bash
read -rp 'Management SSH key path (absolute): ' management_key
ansible-inventory -i inventory.ini --graph
ansible all -i inventory.ini --private-key "$management_key" -m ping
ansible workers -i inventory.ini --private-key "$management_key" -m command -a 'test -c /dev/kvm'
```

Both nodes should return `pong`; the KVM check should exit with code 0. Run [prepare.yml](prepare.yml):

```bash
ansible-playbook -i inventory.ini prepare.yml --private-key "$management_key" -K
```

`-K` prompts for the managed nodes’ sudo password. It assumes the same sudo password on both nodes. Omit it for passwordless sudo, or configure per-host become credentials if they differ. At the separate DevStack prompt, enter a shared password consisting of ASCII letters and digits. Keep it for repeat runs.

The playbook creates the `stack` account, checks out DevStack `stable/2026.2` in `/opt/stack/devstack`, and writes each node’s `local.conf`. Continue when the recap shows `failed=0` and `unreachable=0` for both nodes. Keep generated configurations and installation logs containing credentials out of Git.

## 3. Install the controller, then the compute node

Open an **SSH session on the controller** using its inventory address and management login. Switch to the DevStack account:

```bash
sudo -iu stack
```

In the new shell, run:

```bash
cd /opt/stack/devstack
./stack.sh
```

Wait for **`stack.sh completed`**. Then open an SSH session on **compute01** and repeat the same commands there. Each installer reads the configuration for its own role. If installation stops, check that node’s timestamped file under `/opt/stack/logs/` for the first failing operation before continuing.

For the remaining steps, use **one Bash session on the controller** as `stack`, in `/opt/stack/devstack`. This keeps the server ID and floating IP available through cleanup.

## 4. Register the compute node and publish plans

Load the operator credentials and check the compute services:

```bash
source openrc admin admin
openstack compute service list --service nova-compute
```

Wait until `compute01` reports `up`. Map the host into the Nova cell so the scheduler can place instances on it. DevStack also registers a local compute service on the controller. Disable scheduling there so customer instances land on `compute01`. If the service list uses a fully qualified controller hostname, use that exact value in the `set` command.

```bash
./tools/discover_hosts.sh
openstack hypervisor list
openstack compute service set --disable controller nova-compute
openstack compute service list --service nova-compute
```

Check that `compute01` is **enabled / up** and the controller is **disabled / up**. Then create these public flavors once. A flavor defines the resources for an instance; billing is outside the lab’s scope.

| Plan | vCPUs | RAM | Root disk |
|---|---:|---:|---:|
| `host-mini` | 1 | 512 MiB | 2 GiB |
| `host-small` | 1 | 1,024 MiB | 5 GiB |

```bash
openstack flavor create --public --vcpus 1 --ram 512 --disk 2 host-mini
openstack flavor create --public --vcpus 1 --ram 1024 --disk 5 host-small
openstack flavor list
```

## 5. Launch a customer VPS

Switch to DevStack’s `demo` project. Confirm that the project can see both flavors, an active CirrOS image, the tenant network (`private`) and the external network (`public`). Substitute your network names below if they differ.

```bash
source openrc demo demo
openstack flavor list
openstack image list
openstack network list
```

Enter the ID of the active CirrOS image:

```bash
read -rp 'CirrOS image ID: ' image_id
```

Create a dedicated instance key on the controller and register its public half. These commands assume a fresh run. When resuming, reuse the existing key and named resources after verifying ownership.

```bash
mkdir -p -m 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/first-vps
openstack keypair create --public-key ~/.ssh/first-vps.pub first-vps-key
openstack security group create first-vps-ssh
fip=$(openstack floating ip create public -f value -c floating_ip_address)
ip -4 route get "$fip"
```

The route should use `br-ex`. Copy the IPv4 address following **`src`** from the route output. Use it as the source of the inbound SSH rule. For the default topology it is typically `172.24.4.1`. Enter the address without a prefix length:

```bash
read -rp 'Controller source IPv4 for SSH: ' ssh_source
```

Create the rule, launch the instance and associate its floating IP:

```bash
openstack security group rule create --ingress --protocol tcp --dst-port 22 \
  --remote-ip "${ssh_source}/32" first-vps-ssh
openstack server create --wait --image "$image_id" --flavor host-mini \
  --network private --key-name first-vps-key --security-group first-vps-ssh \
  --use-config-drive first-vps
vps_id=$(openstack server show first-vps -f value -c id)
openstack server add floating ip "$vps_id" "$fip"
openstack server show "$vps_id" -c id -c status -c flavor -c addresses
```

## 6. Verify delivery

**Customer check.** Once the instance is `ACTIVE` and CirrOS has finished booting, connect from the controller. Verify the guest’s SSH host key through a trusted console before accepting it on first use.

```bash
ssh -o IdentitiesOnly=yes -o PreferredAuthentications=publickey \
  -i ~/.ssh/first-vps "cirros@$fip" 'hostname; id; uptime'
```

**Operator check.** Use the same instance ID to confirm placement:

```bash
source openrc admin admin
openstack server show "$vps_id" -c name -c OS-EXT-SRV-ATTR:host
```

Delivery succeeds when the instance shows the requested flavor, the SSH command returns guest output, and Nova reports `compute01` as the host. For a deployment record, save the date, DevStack commit (`git rev-parse HEAD` on each node), selected flavor, placement and SSH result. Publish sanitized output from an actual run.

## 7. Release the customer resources

Return to the customer project. Confirm that the instance and floating IP match this run before deleting them. If the shell was closed, recover the two values with `openstack server list` and `openstack floating ip list` after loading the demo credentials.

```bash
source openrc demo demo
openstack server show "$vps_id" -c name -c addresses
openstack floating ip show "$fip" -c floating_ip_address -c port_id
```

```bash
openstack server delete --wait "$vps_id"
openstack floating ip delete "$fip"
openstack security group delete first-vps-ssh
openstack keypair delete first-vps-key
```

The two public flavors remain available for the next customer. Deleting the OpenStack keypair removes its registration; the SSH key files remain on the controller.

