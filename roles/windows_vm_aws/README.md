# windows_vm_aws role

Creates a Windows EC2 instance on AWS.

The role is idempotent: if an instance with the same Name tag already exists (and is not terminated) it reports this and takes no destructive action.

## Variables

See `defaults/main.yml` for all variables and their defaults.

## AWS authentication

The role does not manage credentials. Provide them via any method the AWS SDK supports:

- Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
- AWS CLI profile: `AWS_PROFILE`
- IAM instance role (if the controller runs on EC2)

## Future extensions

- WinRM bootstrap via user data script.
- Elastic IP allocation.
- Custom VPC and subnet creation.
- Domain join via SSM.
