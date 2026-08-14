# kvm_vm_linux role

Creates a Linux VM on a KVM/libvirt host.

Idempotent: if a VM with the same name already exists it reports
this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

Set `vm_install_iso` to match your distribution:
- Fedora, Ubuntu, Debian, RHEL, etc.
- Set `vm_os_variant` accordingly (e.g., `fedora40`, `ubuntu24.04`)

## Shared tasks

This role reuses shared task files from `roles/kvm_vm/`.

## Future extensions

- `files/` — reserved for cloud-init configs or kickstart files.
