# Project 01: Account hardening

## Objective

Give an EC2 instance access to one S3 bucket without any access key stored on the machine. Use an IAM role with a least-privilege policy, and test that everything outside the policy is refused.

## Architecture

<!-- TODO: à compléter par Ebene - ajouter architecture.png puis décommenter la ligne suivante -->
<!-- ![Architecture](architecture.png) -->

The flow works like this:

1. The EC2 instance has an instance profile attached.
2. The instance profile points to the role `BucketListRoleEbene`.
3. The role trusts the EC2 service (see [role-trust-policy.json](policies/role-trust-policy.json)).
4. When the AWS CLI runs, it gets temporary credentials from STS.
5. With these credentials, the instance can list and read `ebene-p1-lab-01` and nothing else.

No access key is configured on the instance.

## What was built

- Root account used for the initial setup (MFA enabled, IAM access to billing)
- IAM admin user with MFA
- Group and user `readonly-auditor` with read-only access
- Bucket `ebene-p1-lab-01` with one test object, and a second bucket `ebene-p1-lab-02` to test denied access
- Customer managed policy `AlowReadObejctEbene`: `s3:ListBucket` and `s3:GetObject` on `ebene-p1-lab-01` only
- Role `BucketListRoleEbene` (trusted entity: EC2) with its instance profile
- Amazon Linux EC2 instance with the role attached, accessed with EC2 Instance Connect
- A test policy with an explicit deny on `s3:GetObject`
- AWS Budgets alert at $1

## Validation

Each check was run from the EC2 instance unless stated otherwise.

| Check | Command | Expected result | Screenshot |
|-------|---------|-----------------|------------|
| The instance uses the role | `aws sts get-caller-identity` | ARN contains `assumed-role/BucketListRoleEbene` | <!-- TODO: capture perdue --> |
| List and read the allowed bucket | `aws s3 ls s3://ebene-p1-lab-01` then `aws s3 cp s3://ebene-p1-lab-01/2026-09-03-audit-hardcode-mock-fleet.md .` | Object listed and downloaded | [terminal](screenshots/s3-cp-syntax-errors-and-lab02-denied.png) |
| Second bucket is refused | `aws s3 ls ebene-p1-lab-02` | `AccessDenied` on `s3:ListBucket` | [terminal](screenshots/s3-cp-syntax-errors-and-lab02-denied.png) |
| Write to `lab-02` is refused | `aws s3 cp s3://ebene-p1-lab-01/<object> s3://ebene-p1-lab-02/` | `AccessDenied` on `s3:PutObject` | [terminal](screenshots/s3-cp-syntax-errors-and-lab02-denied.png) |
| Upload to `lab-01` is refused | <!-- TODO: commande exacte --> | `AccessDenied` (no `s3:PutObject`) | <!-- TODO: capture perdue --> |
| `readonly-auditor` cannot create a bucket | Create bucket in the console as `readonly-auditor` | Access denied | <!-- TODO: capture perdue --> |
| Explicit deny wins over allow | Attach [test-explicit-deny-policy.json](policies/test-explicit-deny-policy.json) to the role, then `aws s3 cp` the object | Access denied even with the allow policy present | <!-- TODO: capture perdue --> |

Other evidence:

- Role and attached policy: [role-summary-and-attached-policy.png](screenshots/role-summary-and-attached-policy.png)
- Policy in the editor: [policy-editor-permission-policy.png](screenshots/policy-editor-permission-policy.png)
- Budget: [budget-overview.png](screenshots/budget-overview.png)

## What went wrong

### `aws s3 ls` without a bucket name

```
An error occurred (AccessDenied) when calling the ListBuckets operation:
User: arn:aws:sts::123456789012:assumed-role/BucketListRoleEbene/i-0123456789abcdef0
is not authorized to perform: s3:ListAllMyBuckets because no identity-based
policy allows the s3:ListAllMyBuckets action
```

Without a bucket name, the command lists all buckets in the account. That needs `s3:ListAllMyBuckets`, which is different from `s3:ListBucket` (list the content of one bucket). I did not allow it, so it was an implicit deny. I chose not to add it (see the [decision log](decision-log.md)). The message also shows `assumed-role`, so the instance really uses the role.

`aws s3 ls -la` gave `Unknown options: -la`. `-la` belongs to the Linux `ls` command, not to `aws s3 ls`. Options are listed with `aws s3 ls help`.

### `ListBucket` refused on my own bucket

```
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:sts::123456789012:assumed-role/BucketListRoleEbene/i-0123456789abcdef0
is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::ebene-p1-lab-01"
because no identity-based policy allows the s3:ListBucket action
```

My first policy had one resource, `arn:aws:s3:::ebene-p1-lab-01/*`, for both actions ([v1](policies/AlowReadObejctEbene-v1-broken.json)). S3 has two resource levels:

- `s3:ListBucket` applies to the bucket: `arn:aws:s3:::ebene-p1-lab-01`
- `s3:GetObject` applies to the objects: `arn:aws:s3:::ebene-p1-lab-01/*`

My ARN only matched objects, so no allow matched `ListBucket`.

Fix: two separate statements, each with its action and its ARN ([final policy](policies/AlowReadObejctEbene.json)). Everything worked after that.

### `aws s3 cp` syntax

Commands tried:

```
aws s3 cp ebene-p1-lab-01/2026-09-03-audit-hardcode-mock-fleet.md s3:/ebene-p1-lab-02/
aws s3 cp ebene-p1-lab-01/2026-09-03-audit-hardcode-mock-fleet.md
```

- The first gave `Error: Invalid argument type`. The source had no `s3://` prefix, so the CLI read it as a local path. The destination had `s3:/` with one slash. The CLI saw local to local, which `aws s3 cp` does not accept.
- The second gave `the following arguments are required: paths`. `cp` always needs a source and a destination.

Correct form:

```
aws s3 cp s3://ebene-p1-lab-01/2026-09-03-audit-hardcode-mock-fleet.md .
```

The valid combinations are local to S3, S3 to local and S3 to S3. A copy from `lab-01` to `lab-02` needs `s3:GetObject` on the source and `s3:PutObject` on the destination. My role has no rights on `lab-02`, so it was refused anyway (see the terminal screenshot).

## Key points

- Policy evaluation: explicit deny > allow > implicit deny. There is no order between policies. All applicable policies are evaluated together, and a deny always wins.
- An implicit deny is not a written policy. It is what remains when no policy matches.
- Identity-based vs resource-based policies: <!-- TODO: à compléter par Ebene -->
- Role vs access keys: a role gives temporary credentials through STS, renewed automatically. Access keys are permanent credentials stored on the machine. They stay valid until someone revokes them.
- Least privilege: `ListBucket` on the bucket ARN, `GetObject` on the object ARN, and no `ListAllMyBuckets`.

## Files

- [decision-log.md](decision-log.md): what I kept, what I rejected, and why
- [cost-report.md](cost-report.md): real cost and cost at scale
- [notes-exam.md](notes-exam.md): exam notes (in French)
- [checklist-nouveau-compte-aws.md](checklist-nouveau-compte-aws.md): reusable checklist for a new AWS account
- [policies/](policies/): IAM policies used in this project
- [screenshots/](screenshots/): screenshots
