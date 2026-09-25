# Cloud Computing Project — BookHaven

A team project for the design of a highly available, secure and cost-aware cloud
architecture for an online bookshop.

This repository holds the **documentation and collaboration files** for the
project. No application code, deployment scripts or infrastructure-as-code has
been added at this stage.

**Unit:** COIT20260 Cloud Computing and Internet of Things for Smart Applications
**Institution:** CQUniversity, Sydney
**Team project:** BookHaven cloud architecture design

Unit background, the IaaS and PaaS reasoning behind the tier choices, and the
ethical and sustainability considerations carried forward from earlier work in
the unit are recorded in [`docs/unit-context.md`](docs/unit-context.md).

---

## Project purpose

BookHaven is an online bookstore. Its web application manages the book
catalogue, customer orders and inventory, and was hosted on a **single
on-premises server**. During Black Friday and Boxing Day, large increases in
users made the application slow or unavailable, causing abandoned carts and lost
sales.

This project migrates hosting to AWS and replaces the single-server dependency
with a distributed architecture: multiple application instances behind an
Application Load Balancer, spread across two Availability Zones, with Auto
Scaling and a private managed database.

The design meets the assessment criteria for functional design, load balancing,
scalability, high availability, security, cost optimisation and high
performance, and was built and tested as a proof of concept in the
**ap-southeast-2 (Sydney)** Region.

This repository contains:

1. The architecture as designed and as deployed.
2. A mapping of the architecture to each assessment requirement.
3. The implementation evidence from the deployed proof of concept.
4. Architecture diagrams produced in Figma.
5. The team's way of working.

**Target design versus proof of concept.** Route 53 and RDS Multi-AZ are part of
the target design but were not deployed — no domain was registered, and Multi-AZ
is unavailable on the AWS Free plan. Every difference is listed in
[`docs/architecture.md`](docs/architecture.md).

---

## Repository structure

```
bookhaven/
├── README.md                        Project overview and agreed scope
├── CONTRIBUTING.md                  How to contribute changes
├── .gitignore                       Files excluded from version control
├── docs/
│   ├── architecture.md              The architecture, as designed and deployed
│   ├── requirements-mapping.md      Architecture mapped to assessment criteria
│   ├── implementation-evidence.md   What was deployed and what testing showed
│   ├── unit-context.md              Unit details and background from Assessment 1
│   └── references.md                Harvard reference list for cited sources
├── diagrams/
│   └── README.md                    Where exported Figma diagrams are stored
└── .github/
    ├── pull_request_template.md     Structure for every pull request
    └── ISSUE_TEMPLATE/
        ├── task.md                  Template for a planned piece of work
        └── bug_report.md            Template for reporting a defect
```

---

## Architecture diagram

The architecture diagram is maintained in Figma and exported into
[`diagrams/`](diagrams/) for assessment submission.

**Figma file:** `https://www.figma.com/board/IdgZQlzcqfqZntz5PJtZpy`

Exported images must be committed to the repository so the diagram is available
without a Figma account. See [`diagrams/README.md`](diagrams/README.md) for
naming and export conventions.

---

## AWS services used

| Service | Role in the architecture |
|---|---|
| **Amazon Route 53** | DNS entry point for a custom domain. *Target design only — no domain registered.* |
| **Amazon CloudFront** | HTTPS at the edge in front of the ALB; caching disabled for dynamic PHP pages. |
| **Application Load Balancer** | Internet-facing, in two public subnets; forwards to the `BookHaven-WebTG` target group. |
| **Amazon EC2** | Two `t3.micro` Amazon Linux 2023 servers running Apache and PHP, one per AZ. |
| **EC2 Auto Scaling** | `BookHaven-WebASG`, 2–4 instances, target tracking at 50% average CPU. |
| **Amazon RDS for MySQL** | MySQL 8.4 on `db.t4g.micro`, KMS-encrypted, private DB subnet group across both AZs. *Multi-AZ is target design.* |
| **Amazon VPC** | `BookHaven-vpc` (10.0.0.0/16), two public and two private subnets across two AZs. |
| **Security Groups** | Traffic chained by tier: internet → ALB → EC2 → RDS. |
| **AWS IAM** | `BookHaven-EC2-RDS-Role` instance role; no access keys stored on servers. |
| **AWS Secrets Manager** | RDS-managed database credential, read at runtime through a private endpoint. |
| **Amazon CloudWatch** | EC2 metrics and the alarms driving the Auto Scaling policy. |
| **VPC endpoints** | Secrets Manager interface endpoint and S3 gateway endpoint, used instead of a NAT gateway. |
| **EC2 Instance Connect Endpoint** | Private administrative SSH access; no bastion host, no public IPs. |

**Not deployed:** AWS WAF and an ACM certificate on the ALB were left out of the
proof of concept to control cost, and are recorded as production enhancements.

Full detail is in [`docs/architecture.md`](docs/architecture.md), the
requirement mapping in
[`docs/requirements-mapping.md`](docs/requirements-mapping.md), and the
deployment evidence in
[`docs/implementation-evidence.md`](docs/implementation-evidence.md). The
reasoning behind using EC2 (closer to IaaS) for the application tier and managed
RDS for the database tier is explained in
[`docs/unit-context.md`](docs/unit-context.md).

---

## Team collaboration

All work is planned, discussed and reviewed through this repository so that
contributions from every team member are visible and recorded.

**Before starting work**

1. Check the issue list to see what is already claimed.
2. Open an issue, or assign yourself an existing one, so the team knows what you
   are working on.
3. Discuss anything that changes the agreed architecture **before** making the
   change, not in the pull request.

**While working**

4. Work on your own feature branch. Do not commit directly to `main`.
5. Keep changes focused on the issue you are addressing.
6. Post a short progress update on your issue if work runs across several days.

**When finished**

7. Open a pull request using the repository template.
8. Request a review from at least one other team member.
9. Merge only after approval.

Detailed guidance is in [`CONTRIBUTING.md`](CONTRIBUTING.md).

---

## Branching and pull-request workflow

`main` is the protected, submission-ready branch. It should always contain
documentation the team is willing to be assessed on.

**Branch naming**

| Prefix | Used for | Example |
|---|---|---|
| `docs/` | Documentation changes | `docs/architecture-security-section` |
| `diagram/` | Diagram exports and updates | `diagram/architecture-v2` |
| `fix/` | Corrections to existing content | `fix/broken-figma-link` |
| `chore/` | Repository housekeeping | `chore/update-gitignore` |

**Workflow**

```
main
 └── docs/architecture-security-section     ← create a branch from main
      ├── commit                            ← make focused changes
      ├── commit
      └── pull request → review → approval → merge into main
```

1. Create a branch from an up-to-date `main`.
2. Make focused changes that address one issue.
3. Commit with clear, descriptive messages.
4. Push the branch to GitHub.
5. Open a pull request and complete the template.
6. Request a review from a team member.
7. Respond to review comments and update the branch.
8. Merge once the pull request has been approved.
9. Delete the branch after merging.

Pull requests are never merged by their own author without a review, so that
every change has been seen by at least two people.

---

## Repository conventions

- **No credentials of any kind** are stored in this repository. No passwords,
  AWS access keys, `.pem` files or `.env` files. Database credentials belong in
  AWS Secrets Manager, as described in the architecture document.
- **No application code or infrastructure-as-code** at this stage. The
  repository is documentation only until the design is agreed.
- **Documentation and diagram must agree.** If one changes, the other changes in
  the same pull request.
- **Sources are cited in Harvard (CQUniversity) author–date style**, and listed
  in [`docs/references.md`](docs/references.md).
