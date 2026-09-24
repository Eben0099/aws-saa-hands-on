# Decision log: Project 01

## Application credentials on EC2

- Kept: IAM role through an instance profile
- Rejected: access keys configured on the instance (`aws configure`)
- Why: access keys are permanent credentials stored on the machine. If the instance is compromised, or if the keys leak (repository, image, logs), they stay valid until someone revokes them manually. The role gives temporary credentials through STS, renewed automatically, and nothing is stored on disk. There is no rotation to manage.

## Adding `s3:ListAllMyBuckets` so that `aws s3 ls` works

- Kept: do not add it
- Rejected: add it
- Why: the application only needs to read one specific bucket. `ListAllMyBuckets` would show the names of all buckets in the account. This is useless for the application and useful for an attacker. Least privilege: I do not grant a permission to make a command work. I grant it only if the business need requires it.

## Policy structure (`ListBucket` and `GetObject`)

- Kept: two separate statements, each with the action and the ARN that fits it
- Rejected: one resource `bucket/*` for both actions (my first version, which failed on `ListBucket`)
- Why: `ListBucket` applies to the bucket and `GetObject` applies to the objects. Each action must be linked to the right ARN level, or no allow matches. Two statements are easier to read: each action has exactly the resource it needs.

## Permissions of `readonly-auditor`

- Kept: permissions attached to a group
- Rejected: permissions attached directly to the user
- Why: a group holds the permissions of one job role. To add a new auditor, I add the user to the group. To change the rights, I change one policy. With permissions per user, policies get duplicated and differences between users appear quickly, which makes audits harder.

## Use of the root account

- Kept: root only for the operations that require it, daily work with the IAM admin user
- Rejected: working as root
- Why: root has powers that no IAM policy can limit, not even an explicit deny (only SCPs from an Organization can). A compromised root session means the whole account is compromised, billing included. Working with an IAM user also makes denials visible, so permission errors can be detected.

## Budget alert

- Kept: budget of $1.00 (see [budget-overview.png](screenshots/budget-overview.png)). Type and thresholds: <!-- TODO: à compléter par Ebene -->
- Rejected: a high threshold ($10 or $100)
- Why: during preparation, almost everything should cost $0. An alert at $1 shows a forgotten resource (NAT Gateway, unattached Elastic IP, orphan EBS volume) before the cost grows. At about 650 FCFA per dollar, a NAT Gateway forgotten for one month costs about 21,000 FCFA. A "forecasted" alert warns before the spending is real.

## Connection to the instance

- Kept: EC2 Instance Connect
- Rejected: SSH with a key pair
- Why: there is no private key to create, store or protect. Instance Connect pushes a temporary key (valid for about 60 seconds) and each connection goes through IAM, so it is controlled and traceable. This fits the goal of the project: no permanent credential.
