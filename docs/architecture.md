# Architecture

Engineer access to the Onward Logistics AWS account is defined in Git, deployed by
CloudFormation Git sync, capped by permission boundaries and recorded by CloudTrail.
The architecture has three parts: how a change gets deployed, what each engineer can do,
and how every change is recorded.

## Deploy path

A merge to `main` is the only way access changes. Nobody edits IAM by hand in the console.

```text
  Platform engineer
  opens a pull request, a reviewer approves it
            │
            │  merge to main
            ▼
┌────────────────────────────────────────────┐
│ GitHub · RUYANGA/AWS-Lab1 · main           │
│   templates/onboarding.yaml                │
│   deployment-file.yaml                     │
└──────────────────────┬─────────────────────┘
                       │  watches main
═══════════════════════╪═══ AWS account ══════
                       ▼
┌────────────────────────────────────────────┐
│ CodeConnections                            │  ◀ made by bootstrap stack
│   onward-onboarding-repo                   │
└──────────────────────┬─────────────────────┘
                       │  pulls template + deployment file
                       ▼
┌────────────────────────────────────────────┐
│ CloudFormation Git sync                    │  ◀ role from bootstrap stack
│   role: onward-gitsync-role                │
└──────────────────────┬─────────────────────┘
                       │  change set: create → execute
                       ▼
┌────────────────────────────────────────────┐
│ CloudFormation stack: onward-onboarding    │  ◀ role from bootstrap stack
│   role: onward-onboarding-stack-role       │
└──────────────────────┬─────────────────────┘
                       │  builds
                       ▼
    IAM · Secrets Manager · CloudTrail · S3
```

| Part | Created by | Job |
| --- | --- | --- |
| `onward-onboarding-repo` | Bootstrap stack | Links the AWS account to the GitHub repository |
| `onward-gitsync-role` | Bootstrap stack | Lets Git sync create and run change sets on the stack |
| `onward-onboarding-stack-role` | Bootstrap stack | Lets CloudFormation build IAM, Secrets Manager, S3 and CloudTrail resources |
| `onward-onboarding` stack | Git sync, from [`deployment-file.yaml`](../deployment-file.yaml) | Holds every resource in [`templates/onboarding.yaml`](../templates/onboarding.yaml) |

The bootstrap stack is [`bootstrap/gitsync-bootstrap.yaml`](../bootstrap/gitsync-bootstrap.yaml).
An admin deploys it once and approves the GitHub connection in the console.

## Access model

Each engineer gets permissions from a **group** and is limited by a **boundary**.
They can only do what both allow:

> **effective access = group ∩ boundary**

```text
 GROUP gives                               USER               BOUNDARY limits

  Storage engineer
┌─────────────────────────┐           ┌────────────┐         ┌───────────────────────────┐
│ storage-engineers       │─ grants ─▶│  s3-user   │◀─ caps ─│ boundary-storage          │
│ AmazonS3FullAccess      │           │            │         │ S3 + own password         │
└─────────────────────────┘           └────────────┘         └───────────────────────────┘

  Compute engineer
┌─────────────────────────┐           ┌────────────┐         ┌───────────────────────────┐
│ compute-engineers       │─ grants ─▶│ ec2-user-1 │◀─ caps ─│ boundary-compute          │
│ AmazonEC2FullAccess     │           │            │         │ EC2 · ELB · Auto Scaling  │
└─────────────────────────┘           └────────────┘         └───────────────────────────┘

  On-call engineer
┌─────────────────────────┐           ┌────────────┐         ┌───────────────────────────┐
│ oncall-observers        │─ grants ─▶│ ec2-user-2 │◀─ caps ─│ boundary-compute-observer │
│ AmazonEC2ReadOnlyAccess │           │            │         │ read-only + explicit deny │
└─────────────────────────┘           └────────────┘         └───────────────────────────┘
```

| User | Group policy | Boundary allows at most |
| --- | --- | --- |
| `s3-user` | AmazonS3FullAccess | S3, own password |
| `ec2-user-1` | AmazonEC2FullAccess | EC2, load balancers, Auto Scaling, read CloudWatch, own password |
| `ec2-user-2` | AmazonEC2ReadOnlyAccess | Read-only compute and CloudWatch, own password. Starting, creating and terminating instances is explicitly denied. |

- All group and boundary names start with `onward-prod-`, set by `Environment` in
  `deployment-file.yaml`.
- Every group also has `IAMUserChangePassword`.
- If someone attaches `AdministratorAccess` to a user or group, the boundary still blocks
  anything outside its list.

## First-login passwords

Secrets Manager generates one 20-character password per user, stored at
`onward/prod/onboarding/<user>`. CloudFormation reads it when it creates the user, so no
password is ever written in Git. An admin sends it to the engineer outside Git, with the
sign-in URL from the stack outputs, and the engineer must set a new password at first sign-in.

## Audit trail

Created when `CreateAuditTrail` is `true` in `deployment-file.yaml`.

```text
  Every access change (users, groups, boundaries, roles)
            │
            ▼
┌──────────────────────────────┐                ┌────────────────────────────┐
│ CloudTrail                   │  writes logs   │ S3 audit bucket            │
│ onward-prod-access-audit     │ ────────────▶  │ encrypted · versioned      │
│ all regions + IAM events     │  HTTPS only    │ public access blocked      │
│ log file validation on       │                │ kept if stack is deleted   │
└──────────────────────────────┘                └────────────────────────────┘
```

- The trail covers every region and includes global IAM events, so every access change is
  recorded.
- Log file validation proves whether a log file was edited or deleted.
- The bucket policy lets only this trail write, only into its own folder, and refuses any
  request that isn't over HTTPS.

## How a change reaches AWS

1. An admin deploys the bootstrap stack once and approves the GitHub connection.
2. An engineer edits `templates/onboarding.yaml` or `deployment-file.yaml`, opens a pull
   request, and a reviewer approves the merge to `main`.
3. Git sync sees the new commit through CodeConnections, assumes `onward-gitsync-role`, and
   creates and runs a change set on `onward-onboarding`.
4. CloudFormation builds or updates the groups, users, boundaries, secrets and audit trail
   using `onward-onboarding-stack-role`.
5. An admin sends each new engineer their first-login password and the sign-in URL.
6. CloudTrail records the change, and [`evidence.sh`](../evidence.sh) shows the boundaries
   still hold.
