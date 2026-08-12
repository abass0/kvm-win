# windows_vm role

Creates a Windows virtual machine on a KVM/libvirt host using a supplied Windows ISO and optional VirtIO driver ISO.

The role is idempotent: if the VM already exists it reports this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

## Shell commands used

- `virsh pool-info` — no Ansible module inspects storage pool state.
- `qemu-img create` — no Ansible module creates qcow2 disk images.

Both commands are executed remotely on the KVM host by Ansible.

## Future extensions

- `files/autounattend.xml` — unattended Windows installation answer file.
- WinRM bootstrap for post-install configuration.
