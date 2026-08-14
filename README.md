# VM Provisioning Automation

Ansible automation to provision Windows and Linux virtual machines, controlled entirely from an Ansible Controller.

Four provisioning paths are supported:

| Playbook | Target | OS |
|---|---|---|
| `provision_windows.yml` | Fedora KVM/libvirt | Windows |
| `provision_linux.yml` | Fedora KVM/libvirt | Linux |
| `provision_windows_aws.yml` | AWS EC2 | Windows |
| `provision_linux_aws.yml` | AWS EC2 | Linux |

## Architecture

### Local KVM

```
Ansible Controller
       |
       | SSH
       v
Fedora KVM Host
       |
       | libvirt/KVM
       v
Windows / Linux VM
```

### AWS EC2

```
Ansible Controller
       |
       | AWS API (boto3)
       v
Amazon EC2
       |
       v
Windows / Linux Instance
```

- **Ansible Controller** — the single operational entry point.
- **Fedora KVM Host** (KVM path) — managed over SSH by Ansible.
- **Amazon EC2** (AWS path) — managed via API calls from the controller.

---

## 1. One-Time Fedora KVM Host Preparation

These steps are performed **once** to bootstrap the host. After this, all provisioning is done from the Controller.

### 1.1 Install KVM and libvirt

```bash
# On the Fedora KVM host
sudo dnf install -y @virtualization
sudo systemctl enable --now libvirtd
```

### 1.2 Verify KVM support

```bash
sudo virt-host-validate qemu
```

### 1.3 Configure the Ansible user

```bash
sudo useradd -m ansible
sudo usermod -aG libvirt ansible
echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
```

### 1.4 Configure SSH access

```bash
# On the Controller
ssh-copy-id ansible@fedora-kvm
ssh ansible@fedora-kvm "hostname && virsh version"
```

### 1.5 Configure the default storage pool

```bash
sudo virsh pool-define-as default dir --target /var/lib/libvirt/images
sudo virsh pool-build default
sudo virsh pool-start default
sudo virsh pool-autostart default
```

### 1.6 Place installation media

```bash
sudo mkdir -p /var/lib/libvirt/images/iso
# Copy Windows ISO, VirtIO driver ISO, and/or Linux ISOs here
```

### 1.7 Configure VM networking

```bash
sudo virsh net-list --all
# If not active:
sudo virsh net-start default
sudo virsh net-autostart default
```

### 1.8 Install Python 3

```bash
sudo dnf install -y python3
```

**After these steps, the Fedora host is ready. Everything else runs from the Controller.**

---

## 2. Controller Preparation

### 2.1 Install Ansible

```bash
pip install ansible
```

### 2.2 Install required collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2.3 Python dependencies

**For KVM provisioning:**

```bash
pip install libvirt-python
```

**For AWS provisioning:**

```bash
pip install boto3 botocore
```

---

## 3. Inventory

Edit `inventory/hosts.yml`:

```yaml
all:
  children:
    kvm_hosts:
      hosts:
        fedora-kvm:
          ansible_host: 192.168.1.100
          ansible_user: ansible
          ansible_python_interpreter: /usr/bin/python3
    local:
      hosts:
        localhost:
          ansible_connection: local
```

Test KVM connectivity:

```bash
ansible -i inventory/hosts.yml kvm_hosts -m ping
```

---

## 4. Variables

### 4.1 KVM Variables (shared by `kvm_vm_windows` and `kvm_vm_linux`)

| Variable | Windows Default | Linux Default | Description |
|---|---|---|---|
| `vm_name` | `windows-vm` | `linux-vm` | VM name |
| `vm_memory` | `4096` | `2048` | Memory in MiB |
| `vm_vcpus` | `2` | `2` | Virtual CPUs |
| `vm_disk_size` | `60G` | `30G` | Disk size |
| `vm_storage_pool` | `default` | `default` | libvirt storage pool |
| `vm_disk_dir` | `/var/lib/libvirt/images` | `/var/lib/libvirt/images` | Disk directory |
| `vm_install_iso` | `.../iso/windows.iso` | `.../iso/linux.iso` | Installation ISO path |
| `vm_extra_iso` | `.../iso/virtio-win.iso` | (empty) | Extra ISO (VirtIO drivers for Windows) |
| `vm_network` | `default` | `default` | libvirt network |
| `vm_network_model` | `virtio` | `virtio` | NIC model |
| `vm_os_variant` | `win10` | `generic` | OS variant hint |
| `vm_machine_type` | `q35` | `q35` | Machine type |
| `vm_firmware` | `bios` | `bios` | `bios` or `uefi` |
| `vm_boot_cdrom_first` | `true` | `true` | Boot CD-ROM first |

### 4.2 AWS Variables (shared by `aws_vm_windows` and `aws_vm_linux`)

| Variable | Windows Default | Linux Default | Description |
|---|---|---|---|
| `aws_instance_name` | `windows-vm` | `linux-vm` | Instance Name tag |
| `aws_region` | `us-east-2` | `us-east-2` | AWS region |
| `aws_instance_type` | `t3.large` | `t3.medium` | Instance type |
| `aws_ami_id` | (auto) | (auto) | Specific AMI ID |
| `aws_ami_name_filter` | `Windows_Server-2022-*` | `al2023-ami-*-x86_64` | AMI lookup filter |
| `aws_key_name` | `my-keypair` | `my-keypair` | EC2 key pair |
| `aws_vpc_id` | (default VPC) | (default VPC) | VPC ID |
| `aws_subnet_id` | (default) | (default) | Subnet ID |
| `aws_security_group_name` | `windows-vm-sg` | `linux-vm-sg` | Security group |
| `aws_allowed_rdp_cidrs` | `["0.0.0.0/0"]` | n/a | RDP access CIDRs |
| `aws_allowed_ssh_cidrs` | n/a | `["0.0.0.0/0"]` | SSH access CIDRs |
| `aws_root_volume_size` | `60` | `30` | Root volume GB |
| `aws_root_volume_type` | `gp3` | `gp3` | EBS volume type |
| `aws_assign_public_ip` | `true` | `true` | Public IP |

---

## 5. Provisioning

### 5.1 KVM Windows

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/provision_windows.yml \
  -e vm_name=win-demo01 \
  -e vm_memory=8192 \
  -e vm_vcpus=4 \
  -e vm_disk_size=60G
```

### 5.2 KVM Linux

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/provision_linux.yml \
  -e vm_name=fedora-dev01 \
  -e vm_memory=4096 \
  -e vm_vcpus=2 \
  -e vm_install_iso=/var/lib/libvirt/images/iso/Fedora-Server-dvd-x86_64-40.iso \
  -e vm_os_variant=fedora40
```

### 5.3 AWS Windows

```bash
ansible-playbook playbooks/provision_windows_aws.yml \
  -e aws_instance_name=win-demo01 \
  -e aws_instance_type=t3.large \
  -e aws_key_name=my-keypair
```

### 5.4 AWS Linux

```bash
ansible-playbook playbooks/provision_linux_aws.yml \
  -e aws_instance_name=linux-demo01 \
  -e aws_instance_type=t3.medium \
  -e aws_key_name=my-keypair
```

For Ubuntu instead of Amazon Linux:

```bash
ansible-playbook playbooks/provision_linux_aws.yml \
  -e aws_instance_name=ubuntu-demo01 \
  -e aws_key_name=my-keypair \
  -e aws_ami_owner=099720109477 \
  -e 'aws_ami_name_filter=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*'
```

---

## 6. Idempotency

All playbooks are safe to run multiple times.

- **VM/instance does not exist**: creates it.
- **VM/instance already exists**: reports it and makes no changes.

No destructive operations are performed. Playbooks never destroy, terminate, or overwrite existing resources.

For AWS, idempotency is based on the `Name` tag.

---

## 7. Project Structure

```
kvm-windows/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── provision_windows.yml
│   ├── provision_linux.yml
│   ├── provision_windows_aws.yml
│   └── provision_linux_aws.yml
├── roles/
│   ├── aws_vm/                        # Shared AWS tasks
│   │   └── tasks/
│   │       ├── validate.yml
│   │       ├── ami.yml
│   │       ├── vpc.yml
│   │       └── provision.yml
│   ├── aws_vm_windows/                # AWS Windows entry point
│   │   ├── defaults/main.yml
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── network.yml
│   │   │   └── summary.yml
│   │   └── README.md
│   ├── aws_vm_linux/                  # AWS Linux entry point
│   │   ├── defaults/main.yml
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── network.yml
│   │   │   └── summary.yml
│   │   └── README.md
│   ├── kvm_vm/                        # Shared KVM tasks
│   │   └── tasks/
│   │       ├── validate.yml
│   │       ├── preflight.yml
│   │       └── provision.yml
│   ├── kvm_vm_windows/                # KVM Windows entry point
│   │   ├── defaults/main.yml
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   └── summary.yml
│   │   ├── templates/vm.xml.j2
│   │   ├── handlers/main.yml
│   │   ├── files/
│   │   └── README.md
│   └── kvm_vm_linux/                  # KVM Linux entry point
│       ├── defaults/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   └── summary.yml
│       ├── templates/vm.xml.j2
│       ├── handlers/main.yml
│       ├── files/
│       └── README.md
└── README.md
```

**Base roles** (`aws_vm`, `kvm_vm`) contain only shared task files — they are not called directly.

**Entry-point roles** (`aws_vm_windows`, `aws_vm_linux`, `kvm_vm_windows`, `kvm_vm_linux`) own the defaults, templates, and OS-specific tasks. They pull in shared tasks via `include_tasks`.

---

## 8. OS-Specific Differences

| Aspect | Windows | Linux |
|---|---|---|
| **KVM XML** | Hyper-V enlightenments, `localtime`, `hypervclock`, QXL video | No Hyper-V, `utc` clock, VirtIO video |
| **KVM extra ISO** | VirtIO driver ISO | None (kernel has VirtIO built-in) |
| **AWS AMI** | Windows Server 2022 | Amazon Linux 2023 |
| **AWS SG ports** | RDP 3389, WinRM 5986 | SSH 22 |
| **AWS access** | `get-password-data` + RDP | SSH with key pair |

---

## 9. Troubleshooting

### KVM: SSH connection fails

```bash
ssh -v ansible@fedora-kvm
```

### KVM: libvirtd is not running

```bash
sudo systemctl start libvirtd
```

### KVM: Storage pool / network / ISO not found

```bash
sudo virsh pool-list --all
sudo virsh net-list --all
ls -la /var/lib/libvirt/images/iso/
```

### KVM: community.libvirt or libvirt-python missing

```bash
ansible-galaxy collection install -r requirements.yml
pip install libvirt-python
```

### AWS: boto3 missing

```bash
pip install boto3 botocore
```

### AWS: Credentials not found

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
# or: aws configure
```

### AWS: Key pair not found

Key pairs are per-region. Create one in the target region:

```bash
aws ec2 create-key-pair --key-name my-keypair \
  --query 'KeyMaterial' --output text \
  --region us-east-2 > my-keypair.pem
chmod 400 my-keypair.pem
```

### AWS: No default VPC

Provide `aws_vpc_id` and `aws_subnet_id` explicitly via `-e`.

### AWS: Cannot retrieve Windows password

Wait 5-10 minutes after launch, then:

```bash
aws ec2 get-password-data \
  --instance-id i-0abc123... \
  --priv-launch-key ./my-keypair.pem \
  --region us-east-2
```

---

## 10. Shell Commands Used by Ansible

| Command | Reason |
|---|---|
| `virsh pool-info` | No Ansible module inspects storage pool state |
| `qemu-img create` | No Ansible module creates qcow2 disk images |

Both run remotely on the KVM host via Ansible.

---

## 11. Limitations

**KVM:** OS installation is interactive (boots from ISO). No unattended install or post-install configuration yet.

**AWS:** Instances use stock AMIs. Security groups default to open access — restrict CIDRs in production. No Elastic IP — public IP changes on stop/start.

---

## 12. Next Steps

1. **Unattended Windows install** — `autounattend.xml` as floppy image
2. **Cloud-init for Linux** — automated Linux setup via cloud-init ISO
3. **WinRM bootstrap** — enable WinRM for Ansible management of Windows guests
4. **Post-provisioning** — hostname, networking, domain join, software
5. **VM lifecycle** — stop, start, snapshot, destroy playbooks
6. **Multi-VM provisioning** — batch creation from a variable file
