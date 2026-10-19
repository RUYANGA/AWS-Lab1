# Discovery Report: Secure Engineer Onboarding with CloudFormation Git Sync

| | |
| --- | --- |
| **Lab** | Lab 1 — Secure Engineer Onboarding with CloudFormation Git Sync |
| **Name** | Merci RUYANGA |
| **Date** | 19 October 2026 |
| **Client** | Onward Logistics (see the Client Profile) |

## 1. What you found

### The client's need

Onward Logistics needs to give each new engineer exactly the access their job
requires on day one, the same way every time, and be able to prove to its
security lead who has access and who changed it.

### Solutions that exist

| Solution | How it works | Trade off for this client |
| --- | --- | --- |
| **Manual setup in the AWS console** | An admin creates each user and ticks the permissions by hand when someone joins. | **Strength:** no setup cost, works on the first day. **Weakness:** every admin does it slightly differently, nobody reviews the change, and access quietly grows over time with no record of why. |
| **Access defined as code and deployed from Git** (CloudFormation + Git sync) | Users, groups and permission limits are written in a template; a change is merged through a pull request and AWS applies it automatically. | **Strength:** repeatable, reviewed before it happens, and every change has an author and a date. **Weakness:** takes effort to set up, and even urgent access changes have to go through a review. |
| **Central single sign-on** (IAM Identity Center with a company identity provider) | Engineers sign in with their company account and receive short-lived AWS access through permission sets. | **Strength:** no long-lived AWS passwords, and leavers lose access when their company account is disabled. **Weakness:** needs AWS Organizations and an identity provider, which is more than Onward needs for a handful of engineers today. |

### What happens with no solution

Without a controlled process, access is granted by habit: engineers get more
than they need, and nobody can say later who gave it to them. Two documented
cases show the cost:

- **Capital One (2019).** An attacker used an over-permissioned AWS role to
  read data on about 100 million customers. In 2020 the US bank regulator
  (OCC) fined Capital One **$80 million** for weak risk management of its
  move to the cloud and ordered a remediation plan.
- **Uber (2016).** Attackers found AWS access keys in a code repository and
  downloaded data on 57 million riders and drivers. Uber paid **$148 million**
  in a 2018 settlement with all 50 US states, mostly because it hid the breach.

For Onward the same risks apply on a smaller scale. An engineer who can do
more than their job, or a password stored in Git, could stop dispatch
systems or expose customer and shipment data. That would mean lost revenue,
lost customer trust, and data protection fines. Without an audit trail,
Onward could not show a customer or regulator what happened.

### Sources

- OCC, *OCC Assesses $80 Million Civil Money Penalty Against Capital One* (2020) — https://www.occ.gov/news-issuances/news-releases/2020/nr-occ-2020-101.html
- Security Affairs, *Uber agrees to pay $148 million in massive 2016 data breach settlement* (2018) — https://securityaffairs.com/76587/security/uber-data-breach-settlement.html
- AWS, *Permissions boundaries for IAM entities* — https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
- AWS, *Working with CloudFormation Git sync* — https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/git-sync.html
- AWS, *What is AWS CloudTrail?* — https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html

## 2. The business value of your solution

### Your chosen approach

I chose to **define access as code and deploy it from Git**. Every engineer
is placed in a job group (storage, compute, or on-call observer), and every
engineer also has a fixed upper limit on what they can ever do (a
*permissions boundary*). Access can only change through a reviewed pull
request, and every change is logged.

This suits Onward better than the alternatives. Manual setup is what causes
the problem. Single sign-on is the better long-term goal, but it needs an
identity provider and AWS Organizations that Onward does not have yet. The
code-based approach gives the control and audit trail now, and the same
groups and limits can be reused later as single sign-on permission sets.

### Services you will use

| Service | The job it does | Why this service, used this way |
| --- | --- | --- |
| **AWS IAM** (users, groups, permissions boundaries) | Creates the three engineers, gives each a job group built from AWS managed policies, and caps each one with a boundary. | Groups make access consistent by job, not by person. Boundaries mean that even if someone wrongly attaches admin rights later, the engineer still cannot go past their limit. |
| **AWS CloudFormation** | Builds all the access resources from one template, as one unit. | The same template gives the same result every time, and it can be rolled back if a change goes wrong. |
| **CloudFormation Git sync + AWS CodeConnections** | Watches the Git repository and deploys the template when a change is merged to `main`. | Makes the pull request the only way to change access, so every change is reviewed and has a named author. |
| **AWS Secrets Manager** | Generates a random first-login password for each engineer. | No person invents or types the password and it never appears in Git. The engineer must change it at first sign-in. |
| **AWS CloudTrail** | Records every access change: who did it, what changed, and when. | Answers the security lead's main question. The trail covers all regions and global IAM events, and its log files are checked for tampering. |
| **Amazon S3** | Stores the CloudTrail logs. | Encrypted, versioned, private, and kept even if the stack is deleted, so the audit history cannot be lost by accident. |

### Who benefits, and how you will know

| Stakeholder | Benefit | How you would know |
| --- | --- | --- |
| **Head of Security** | Can see who has which access and prove who changed it, without asking engineers. | `evidence.sh` shows each user is limited to their job, and CloudTrail names the person and time of each change within minutes. |
| **Platform / IT administrators** | Onboarding is a short pull request instead of a manual checklist. | New-engineer setup takes minutes instead of hours; every IAM change in CloudTrail matches a merged pull request. |
| **New engineers** | Have the right access on day one and do not have to wait or ask for more. | First-login works with a forced password reset; access requests in the first week drop to near zero. |
| **On-call engineers** | Can see compute resources during an incident without the risk of shutting anything down by mistake. | The stress test shows `ec2-user-2` is still denied run, start and terminate even with `AdministratorAccess` attached. |
| **Operations leadership** | Lower risk of an outage or data breach caused by wrong access. | No security findings about over-broad IAM access in the next review; 100% of engineers have a boundary attached. |

### Limits and risks of your design

- **No MFA yet.** The template forces a password reset but does not require
  multi-factor authentication. A stolen password is still enough to sign in.
- **Long-lived IAM users.** These are not short-lived credentials. Leavers
  must be removed through a pull request; nothing happens automatically
  when they leave the company.
- **Git becomes the key to access.** Anyone who can merge to `main` can
  change access. Branch protection and required reviews on the repository
  are essential.
- **Console changes are still possible.** An admin can still change IAM
  directly in the console. CloudTrail records it, but nothing blocks it or
  raises an alert. Drift detection and alarms would close this gap.
- **The deployment role is broad.** The role CloudFormation uses can manage
  all of IAM, S3, Secrets Manager and CloudTrail. It should be limited to
  Onward's own resources before production use.
- **One environment per account.** User names are fixed, so a second copy of
  the stack (for example a test environment) cannot run in the same account.
- **Password hand-over is a manual step.** An admin reads the first password
  from Secrets Manager and must send it to the engineer through a secure
  channel.
