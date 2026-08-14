# aws_vm_linux role

Creates a Linux EC2 instance on AWS.

Idempotent: if an instance with the same Name tag already exists
(and is not terminated) it reports this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

The default AMI filter discovers the latest Amazon Linux 2023 AMI.
Override `aws_ami_name_filter` for other distributions:

- Ubuntu: `ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*`
  with `aws_ami_owner: "099720109477"`
- RHEL: `RHEL-9.*_HVM-*-x86_64-*`
  with `aws_ami_owner: "309956199498"`

## Shared tasks

This role reuses shared task files from `roles/aws_vm/`.
