# Checklist: new AWS account

Built from the decisions in [project 01](README.md).

## Root account

- [ ] Use root only for the operations that require it
- [ ] Enable IAM user and role access to Billing
- [ ] Enable MFA on the root account
- [ ] Check that the root account has no access keys

## Daily work

- [ ] Create an IAM admin user with MFA
- [ ] Work with this user, not with root
- [ ] Put permissions on groups, not on users
- [ ] Create read-only users for auditing and check that write actions are refused

## Applications on EC2

- [ ] Use an IAM role with an instance profile, not access keys
- [ ] Write the policy by hand with the fewest actions possible
- [ ] Match each S3 action to the right ARN: bucket ARN for `ListBucket`, object ARN (`/*`) for `GetObject`
- [ ] Do not add a permission only to make a command work (for example `s3:ListAllMyBuckets`)
- [ ] Connect with EC2 Instance Connect instead of a key pair

## Tests

- [ ] Run `aws sts get-caller-identity` and check for `assumed-role`
- [ ] Test an action that must be refused (another bucket, an upload)
- [ ] Test an explicit deny and remove it afterwards

## Cost

- [ ] Create an AWS Budgets alert at $1
