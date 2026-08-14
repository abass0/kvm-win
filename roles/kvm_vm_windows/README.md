# kvm_vm_windows role

Creates a Windows VM on a KVM/libvirt host.

Idempotent: if a VM with the same name already exists it reports
this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

## Shared tasks

This role reuses shared task files from `roles/kvm_vm/`.

## Shell commands used

- `virsh pool-info` — no Ansible module inspects storage pool state.
- `qemu-img create` — no Ansible module creates qcow2 disk images.

Both are executed remotely on the KVM host by Ansible.
