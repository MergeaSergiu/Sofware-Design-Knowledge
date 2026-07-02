# AWS IAM — Roles & Policies

**IAM (Identity and Access Management)** is AWS's system for answering one question on every single API call: *is this principal allowed to perform this action on this resource, right now?* It is built from two kinds of building blocks: **identities** (who is asking — users, groups, roles) and **policies** (JSON documents describing what is allowed or denied).

---

## 1. Purpose — Why IAM?

Every interaction with AWS — a console click, a CLI command, an SDK call from code, one service calling another — is an authenticated, authorized API request. IAM addresses three fundamental problems:

| Problem | How IAM helps |
|---|---|
| **Authentication** | Establishes *who* is making the request (a person, an application, an AWS service). |
| **Authorization** | Decides *what* they may do, via policies evaluated on every request. |
| **Credential hygiene** | Replaces long-lived secrets embedded in code with short-lived, auto-rotating credentials (roles + STS). |

> **Rule of thumb:** grant *least privilege* — the minimum actions, on the minimum resources, under the tightest conditions that still let the job get done. Start narrow and widen; never the reverse.

---

## 2. The Building Blocks

```
Identities (WHO)                    Policies (WHAT)
├── User    — permanent identity,  ├── Identity-based — attached to
│             long-lived creds     │   user/group/role: "what can I do?"
├── Group   — bundle of users,     ├── Resource-based — attached to the
│             for policy reuse     │   resource: "who can touch me?"
└── Role    — assumable identity,  └── Guardrails — boundaries, SCPs,
              temporary creds          session policies (limit only)
```

- **User** — a permanent identity for one person or legacy application, with long-lived credentials: a console password and/or **access keys** for the API/CLI.
- **Group** — a collection of users used purely to attach policies once (e.g., a `Developers` group). Groups can't be nested and can't be a principal in trust policies.
- **Role** — an identity with permissions but **no credentials of its own**. Trusted parties *assume* it and receive temporary credentials. This is the centerpiece of modern AWS access (see §4).
- **Policy** — a JSON document listing permissions. Policies do nothing on their own; they take effect only when attached to an identity or resource.

---

## 3. Policies

### 3.1 Anatomy of a policy statement

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAppBucketReadWrite",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*",
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
      }
    }
  ]
}
```

| Element | Meaning |
|---|---|
| **Effect** | `Allow` or `Deny` |
| **Action** | Which API calls (`s3:GetObject`, `ec2:*`, …) |
| **Resource** | Which ARNs the actions apply to |
| **Condition** | Optional context tests: source IP, MFA present, tags, time, VPC endpoint, … |
| **Principal** | *Only in resource-based/trust policies:* who the statement applies to |

### 3.2 Policy types

| Type | Attached to | Question answered | Notes |
|---|---|---|---|
| **Identity-based** | User, group, role | "What can this identity do?" | AWS-managed (`AmazonS3ReadOnlyAccess`), customer-managed, or inline |
| **Resource-based** | The resource itself (S3 bucket, SQS queue, Lambda, KMS key) | "Who can act on this resource?" | Names a `Principal`; the key to cross-account access |
| **Permissions boundary** | User or role | Ceiling on what identity policies may grant | Limits only — never grants |
| **Service Control Policy (SCP)** | Accounts/OUs in an Organization | Org-wide ceiling | Limits only; applies even to account root |
| **Session policy** | A role session at assume time | Further restricts that one session | Limits only |

### 3.3 Evaluation logic — the part worth memorizing

1. Everything is **denied by default** (implicit deny).
2. An applicable **`Allow`** (identity-based *or* resource-based, same account) grants access.
3. An **explicit `Deny` always wins**, no matter how many Allows exist.
4. Guardrails (boundary, SCP, session policy) must *also* allow it — they cap the maximum, they never grant.

For **cross-account** access, both sides must agree: the caller's account must allow the action *and* the resource's policy (or an assumable role) must permit that external principal.

> **Rule of thumb:** use `Allow` statements scoped tightly for day-to-day grants, and reserve explicit `Deny` for non-negotiable guardrails ("never delete backups", "never act outside eu-west-1") — because a Deny cannot be overridden by anything.

---

## 4. Roles — Identities Without Credentials

A role is the answer to "how do I give something permissions **without** giving it a password or permanent keys?" Nobody logs in as a role. Instead, a trusted party calls **STS** (`sts:AssumeRole`) and receives **temporary credentials** — access key + secret + session token — that expire (15 minutes to 12 hours, typically 1 hour).

Every role has **two** policy documents, and keeping them straight is half of understanding IAM:

| Document | Question it answers |
|---|---|
| **Trust policy** (assume-role policy) | *Who may assume this role?* — the only resource-based policy a role has |
| **Permission policies** | *What can the role do once assumed?* |

```json
// Trust policy: "EC2 instances may wear this role"
{
  "Effect": "Allow",
  "Principal": { "Service": "ec2.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Both gates must open: the caller must be trusted **and** the role must have the permission. Assumption flow:

```
Principal (user / service / federated identity)
    │  sts:AssumeRole          ── trust policy checked here
    ▼
AWS STS ──▶ temporary credentials (expire automatically)
    │
    ▼
API calls as the role          ── permission policies checked here
```

---

## 5. How Services Use Roles

The most common real-world pattern: code running on Lambda, EC2, or ECS needs to call other AWS APIs. **Never embed access keys in code** — attach a role instead.

1. Create a role whose **trust policy** trusts the service principal (`lambda.amazonaws.com`, `ec2.amazonaws.com`, `ecs-tasks.amazonaws.com`, …).
2. Attach **permission policies** for exactly what the workload needs (e.g., read one DynamoDB table).
3. Associate the role with the compute resource:

| Service | Mechanism |
|---|---|
| **EC2** | **Instance profile** (a wrapper that binds a role to an instance); credentials served via the instance metadata service (use IMDSv2) |
| **Lambda** | **Execution role** on the function |
| **ECS / Fargate** | **Task role** (your code) + **execution role** (the agent: pulling images, writing logs) |
| **EKS** | **IRSA / Pod Identity** — a role per Kubernetes service account |

4. At runtime the SDK picks up rotating temporary credentials automatically — your code just calls `s3.getObject(...)` and it works, with no keys anywhere in code, config, or environment.

Two related service patterns:

- **Service-linked roles** — created and managed by AWS itself so a service (e.g., Auto Scaling) can act in your account; you generally don't touch these.
- **Confused-deputy protection** — when a *service* is the trusted principal in a cross-account setup, add `Condition` keys like `aws:SourceArn` / `aws:SourceAccount` to the trust policy so the service can only assume the role on behalf of *your* resources.

---

## 6. How Users (Humans) Use Roles

- **Cross-account access** — an admin in account A assumes a role in account B instead of having a second user there. The role in B trusts account A's principal; every assumption is logged in CloudTrail.
- **Privilege escalation on demand** — users carry modest day-to-day permissions but can assume an `Admin` role when needed, often gated by an MFA condition in the trust policy (`"Bool": {"aws:MultiFactorAuthPresent": "true"}`).
- **Federation / SSO — the modern default.** Humans don't get IAM users at all. They authenticate through an identity provider (**IAM Identity Center**, Okta, Entra ID via SAML/OIDC), get mapped to a **permission set** (a role under the hood), and receive fresh temporary credentials each session. Offboarding happens in one place — the IdP.

> **Rule of thumb:** if you find yourself creating an IAM user with access keys in 2026, stop and ask which role should exist instead. Legitimate exceptions are rare (some third-party tools, break-glass accounts).

---

## 7. Users vs. Roles — Mental Model Summary

| | **User** | **Role** |
|---|---|---|
| Credentials | Long-lived (password, access keys) | None of its own — temporary via STS |
| Who is it for | One specific person/app | Anyone/anything the trust policy allows |
| Rotation | Manual (and usually neglected) | Automatic — credentials expire |
| Typical use today | Break-glass, legacy tooling | Everything else: services, humans via SSO, cross-account |

- **Policy** = the *what* (a permission document).
- **User** = a permanent *who* with long-lived keys — discouraged for humans (use SSO) and for apps (use roles).
- **Role** = a *who* that anyone trusted can temporarily become. **Trust policy** controls the *become*; **permission policies** control the *do*.
- Access requires: no explicit Deny anywhere, at least one applicable Allow, and no guardrail (boundary/SCP) capping it out.

---

## 8. Pitfalls & Best Practices

- **Least privilege from day one.** Scope `Action` and `Resource` tightly; `"Action": "*"` on `"Resource": "*"` is a future incident.
- **No access keys in code, config files, or repos — ever.** Roles exist precisely so this is never necessary. If a key leaks, rotate immediately; scanners find keys in public repos within minutes.
- **Prefer roles + temporary credentials for everything** — services, cross-account access, and humans (via IAM Identity Center).
- **Don't confuse the two role policies.** "Access denied assuming the role" → trust policy. "Access denied doing something *as* the role" → permission policies.
- **Require MFA** for human console access and for assuming privileged roles.
- **Never use the account root user** for daily work — lock it down with MFA and reserve it for the few tasks that truly require it.
- **Use conditions as cheap guardrails** — `aws:SourceIp`, `aws:PrincipalOrgID`, `aws:SourceArn` (confused deputy), region restrictions.
- **Audit continuously** — CloudTrail logs every `AssumeRole` and API call; **IAM Access Analyzer** flags resources shared externally and generates least-privilege policies from actual usage.
- **Remember policies are deny-by-default** — when debugging, look for the *missing Allow* or the *hidden explicit Deny/SCP*, in that order.
