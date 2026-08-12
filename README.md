# KVM Windows VM Provisioning

Ansible automation to provision Windows virtual machines on a Fedora KVM/libvirt host, controlled entirely from an Ansible Controller.

## Architecture

```
Ansible Controller
       |
       | SSH
       v
Fedora KVM Host
       |
       | libvirt/KVM
       v
Windows VM
```

- **Ansible Controller** — the single operational entry point. All provisioning commands are run here.
- **Fedora KVM Host** — the infrastructure target. Ansible connects to it over SSH and manages libvirt/KVM resources remotely.
- **Windows VM** — the workload created and started by Ansible on the KVM host.

The operator never needs to SSH into the Fedora host for normal VM provisioning.

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
# On the Fedora KVM host
sudo virt-host-validate qemu
```

All checks should report `PASS`.

### 1.3 Configure the Ansible user

```bash
# On the Fedora KVM host — replace 'ansible' with your chosen username
sudo useradd -m ansible
sudo usermod -aG libvirt ansible

# Allow passwordless sudo for Ansible operations
echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
```

### 1.4 Configure SSH access

```bash
# On the Controller — copy your SSH key to the Fedora host
ssh-copy-id ansible@fedora-kvm
```

Verify connectivity:

```bash
ssh ansible@fedora-kvm "hostname && virsh version"
```

### 1.5 Configure the default storage pool

```bash
# On the Fedora KVM host (if the default pool doesn't exist)
sudo virsh pool-define-as default dir --target /var/lib/libvirt/images
sudo virsh pool-build default
sudo virsh pool-start default
sudo virsh pool-autostart default
```

### 1.6 Create an ISO directory and place installation media

```bash
# On the Fedora KVM host
sudo mkdir -p /var/lib/libvirt/images/iso

# Copy your Windows ISO and VirtIO driver ISO to this directory
# Windows ISO: e.g., Win11_23H2_English_x64.iso
# VirtIO ISO:  e.g., virtio-win.iso (download from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/)
```

### 1.7 Configure VM networking

The `default` NAT network is usually created automatically by libvirt. Verify:

```bash
# On the Fedora KVM host
sudo virsh net-list --all
```

If it is not active:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

### 1.8 Install Python 3 (usually already present)

```bash
# On the Fedora KVM host
sudo dnf install -y python3
```

**After completing these steps, the Fedora host is ready. Everything else is run from the Controller.**

---

## 2. Controller Preparation

### 2.1 Install Ansible

```bash
# On the Controller
pip install ansible
# or
sudo dnf install -y ansible-core
```

### 2.2 Install required collections

```bash
# From the project root on the Controller
ansible-galaxy collection install -r requirements.yml
```

This installs:

- `community.libvirt` — provides `community.libvirt.virt` for managing VMs.

### 2.3 Python dependencies

The `community.libvirt` collection requires the `libvirt-python` package on the **Controller** (or wherever the Ansible connection plugin runs Python). If you see import errors:

```bash
pip install libvirt-python
```

On Fedora, you can also install it via:

```bash
sudo dnf install -y python3-libvirt
```

---

## 3. Inventory

Edit `inventory/hosts.yml` to match your environment:

```yaml
all:
  children:
    kvm_hosts:
      hosts:
        fedora-kvm:
          ansible_host: 192.168.1.100   # Your Fedora KVM host IP
          ansible_user: ansible          # SSH user
          ansible_python_interpreter: /usr/bin/python3
```

Test connectivity:

```bash
ansible -i inventory/hosts.yml kvm_hosts -m ping
```

---

## 4. Variables

| Variable | Default | Description |
|---|---|---|
| `vm_name` | `windows-vm` | Name of the virtual machine |
| `vm_memory` | `4096` | Memory in MiB |
| `vm_vcpus` | `2` | Number of virtual CPUs |
| `vm_disk_size` | `60G` | Virtual disk size (qemu-img format) |
| `vm_storage_pool` | `default` | libvirt storage pool name |
| `vm_disk_dir` | `/var/lib/libvirt/images` | Directory for VM disk images |
| `windows_iso` | `/var/lib/libvirt/images/iso/windows.iso` | Path to Windows ISO on the KVM host |
| `virtio_iso` | `/var/lib/libvirt/images/iso/virtio-win.iso` | Path to VirtIO driver ISO on the KVM host |
| `vm_network` | `default` | libvirt network name |
| `vm_network_model` | `virtio` | Network interface model |
| `vm_os_variant` | `win10` | OS variant for optimization |
| `vm_machine_type` | `q35` | Machine type (q35 recommended for Windows) |
| `vm_firmware` | `bios` | Firmware type: `bios` or `uefi` |
| `vm_uefi_loader` | `/usr/share/edk2/ovmf/OVMF_CODE.fd` | UEFI firmware path (only used when `vm_firmware: uefi`) |
| `vm_boot_cdrom_first` | `true` | Boot from CD-ROM first for installation |
| `vm_disk_path` | (computed) | Full path to the VM disk image |

---

## 5. Provisioning a Windows VM

Run from the **Controller**:

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/provision_windows.yml \
  -e vm_name=win-demo01 \
  -e vm_memory=8192 \
  -e vm_vcpus=4 \
  -e vm_disk_size=60G
```

With custom ISO paths:

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/provision_windows.yml \
  -e vm_name=win-demo01 \
  -e vm_memory=8192 \
  -e vm_vcpus=4 \
  -e vm_disk_size=80G \
  -e windows_iso=/var/lib/libvirt/images/iso/Win11_23H2_English_x64.iso \
  -e virtio_iso=/var/lib/libvirt/images/iso/virtio-win-0.1.240.iso
```

For UEFI firmware (required for Windows 11):

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/provision_windows.yml \
  -e vm_name=win11-uefi \
  -e vm_firmware=uefi
```

---

## 6. Expected Result

On successful provisioning, the playbook reports:

```
========================================
 Windows VM Provisioning Summary
========================================
 VM Name:        win-demo01
 Status:         running
 Already Existed: False
----------------------------------------
 vCPUs:          4
 Memory:         8192 MiB
 Disk:           /var/lib/libvirt/images/win-demo01.qcow2
 Disk Size:      60G
----------------------------------------
 Windows ISO:    /var/lib/libvirt/images/iso/windows.iso
 VirtIO ISO:     /var/lib/libvirt/images/iso/virtio-win.iso
 Network:        default
 Storage Pool:   default
----------------------------------------
 KVM Host:       fedora-kvm
 Machine Type:   q35
 Firmware:       bios
========================================
 VM created and started successfully.
 Connect via: virt-manager or virt-viewer on the KVM host,
 or use SPICE/VNC remotely to complete Windows installation.
========================================
```

The VM will boot from the Windows ISO. Connect to the VM's SPICE console to complete the interactive Windows installation.

---

## 7. Idempotency

The playbook is safe to run multiple times.

- **VM does not exist**: creates the disk, defines the VM, starts it.
- **VM already exists**: reports that the VM exists and makes no changes. The existing disk, configuration, and state are preserved.

No destructive operations are performed. The playbook will never destroy, undefine, or overwrite an existing VM.

---

## 8. Troubleshooting

### SSH connection fails

```bash
# Test SSH from the Controller
ssh -v ansible@fedora-kvm
```

- Verify the IP/hostname in `inventory/hosts.yml`.
- Verify the SSH key is deployed (`ssh-copy-id`).
- Check that sshd is running on the Fedora host.

### Ansible ping fails

```bash
ansible -i inventory/hosts.yml kvm_hosts -m ping -vvv
```

- Check `ansible_user` and `ansible_host` in the inventory.
- Check that Python 3 is installed on the Fedora host.
- Verify `ansible_python_interpreter` is correct.

### Permission denied / sudo errors

- Verify the sudoers file: `sudo cat /etc/sudoers.d/ansible`
- The Ansible user must have `NOPASSWD: ALL` or at minimum permission to run `virsh`, `qemu-img`, and manage `/var/lib/libvirt/`.
- The Ansible user must be in the `libvirt` group.

### libvirtd is not running

```bash
# On the Fedora host
sudo systemctl status libvirtd
sudo systemctl start libvirtd
```

### Storage pool not found

```bash
# On the Fedora host
sudo virsh pool-list --all
```

Create the pool if missing (see One-Time Preparation above).

### ISO file not found

- Verify the ISO paths exist on the **Fedora KVM host** (not on the Controller).
- Check file permissions — the `qemu` user needs read access.

```bash
# On the Fedora host
ls -la /var/lib/libvirt/images/iso/
```

### KVM not available

```bash
# On the Fedora host
sudo virt-host-validate qemu
lsmod | grep kvm
```

- Ensure hardware virtualization (VT-x/AMD-V) is enabled in BIOS/UEFI.
- Ensure the `kvm` kernel module is loaded.

### Network not found

```bash
# On the Fedora host
sudo virsh net-list --all
```

Start the default network if needed:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

### community.libvirt collection not installed

```bash
# On the Controller
ansible-galaxy collection list | grep libvirt
ansible-galaxy collection install -r requirements.yml
```

### libvirt-python import error

If you see `No module named 'libvirt'`:

```bash
# On the Controller
pip install libvirt-python
# or
sudo dnf install -y python3-libvirt
```

---

## 9. Project Structure

```
kvm-windows/
├── ansible.cfg                    # Ansible configuration
├── requirements.yml               # Required Ansible collections
├── inventory/
│   └── hosts.yml                  # KVM host inventory
├── playbooks/
│   └── provision_windows.yml      # Main provisioning playbook
├── roles/
│   └── windows_vm/
│       ├── defaults/
│       │   └── main.yml           # Default variables
│       ├── tasks/
│       │   ├── main.yml           # Task entry point
│       │   ├── validate.yml       # Variable validation
│       │   ├── preflight.yml      # Prerequisite checks
│       │   ├── provision.yml      # VM creation logic
│       │   └── summary.yml        # Provisioning report
│       ├── templates/
│       │   └── vm.xml.j2          # libvirt VM XML template
│       ├── handlers/
│       │   └── main.yml           # Handlers (reserved)
│       ├── files/                 # Reserved for autounattend.xml
│       └── README.md              # Role documentation
└── README.md                     # This file
```

---

## 10. Shell Commands Used by Ansible

The role uses shell commands only where no Ansible module exists:

| Command | Reason |
|---|---|
| `virsh pool-info` | No Ansible module inspects storage pool details |
| `qemu-img create` | No Ansible module creates qcow2 disk images |

Both commands are executed **remotely on the KVM host** by Ansible — the operator does not run them manually.

---

## 11. Limitations

- Windows installation is **interactive** — the VM boots from the ISO and requires manual console interaction to complete the OS installation.
- No `autounattend.xml` is provided yet for unattended installation.
- No WinRM or SSH bootstrap is configured on the Windows guest.
- The VM uses SPICE for console access; the operator needs network access to the KVM host's SPICE port, or access to virt-manager.

---

## 12. Next Steps

Planned enhancements for future iterations:

1. **Unattended Windows installation** — add `autounattend.xml` as a floppy image to automate the Windows installer.
2. **WinRM bootstrap** — configure WinRM during unattended setup so Ansible can manage the Windows guest after installation.
3. **Post-provisioning configuration** — Ansible playbooks to configure the Windows VM (hostname, network, domain join, etc.).
4. **Windows software installation** — use Ansible's `win_chocolatey` or `win_package` modules to install software.
5. **Certificate deployment** — deploy certificates to the Windows trust store via Ansible.
6. **VM lifecycle management** — playbooks for stop, start, snapshot, and destroy operations.
7. **Multiple VM provisioning** — support provisioning multiple VMs from a single variable file.
