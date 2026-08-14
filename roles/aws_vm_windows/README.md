# aws_vm_windows role

Creates a Windows EC2 instance on AWS.

Idempotent: if an instance with the same Name tag already exists
(and is not terminated) it reports this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

## Shared tasks

This role reuses shared task files from `roles/aws_vm/`.

## AWS authentication

Provide credentials via any method the AWS SDK supports:
environment variables, AWS CLI profile, or IAM instance role.
