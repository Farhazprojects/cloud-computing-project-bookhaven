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

BookHaven is an online bookshop serving two groups of users:

- **Customers**, who browse the catalogue, search for titles and place orders.
- **Staff**, who manage stock, update the catalogue and process orders.

The purpose of this project is to design and document an AWS architecture that
supports these users while meeting the assessment criteria for functional
design, load balancing, scalability, high availability, security, cost
optimisation and high performance.

The current stage of work covers:

1. Agreeing the architecture as a team.
2. Documenting that architecture and mapping it to the assessment requirements.
3. Producing an architecture diagram in Figma.
4. Establishing a shared way of working for a remote team.

Building the application and provisioning AWS infrastructure are **out of scope
for this stage** and will follow once the design is agreed and reviewed.

---

## Repository structure

```
bookhaven/
├── README.md                        Project overview and agreed scope
├── CONTRIBUTING.md                  How to contribute changes
├── .gitignore                       Files excluded from version control
├── docs/
│   ├── architecture.md              The agreed architecture, explained
│   ├── requirements-mapping.md      Architecture mapped to assessment criteria
│   ├── team-workflow.md             How the remote team coordinates
│   ├── unit-context.md              Unit details and background from the unit
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

## AWS services currently agreed

The team has agreed the following services. Any addition to this list must be
proposed in an issue and agreed by the team before it appears in the
documentation or the diagram.

| Service | Role in the architecture |
|---|---|
| **Route 53** | Public DNS for the BookHaven domain; resolves user requests to the CloudFront distribution. |
| **CloudFront** | Content delivery network at the edge; caches static content close to users and forwards dynamic requests to the load balancer. |
| **Application Load Balancer** | Distributes incoming application traffic across healthy EC2 instances in both Availability Zones. |
| **EC2 Auto Scaling** | Runs the application tier and adjusts the number of instances in response to demand. |
| **Amazon RDS Multi-AZ** | Managed relational database holding catalogue, customer and order data, with a standby in a second Availability Zone. |
| **VPC and subnets** | Private network boundary, divided into public and private subnets across two Availability Zones. |
| **Security Groups** | Instance-level firewalls controlling traffic between each tier. |
| **IAM** | Identity and access management for users, roles and service permissions. |
| **Secrets Manager** | Secure storage and retrieval of database credentials and other secrets. |
| **CloudWatch** | Metrics, logs and alarms; the source of the scaling signals used by EC2 Auto Scaling. |

Full detail is in [`docs/architecture.md`](docs/architecture.md). The reasoning
behind using EC2 (closer to IaaS) for the application tier and managed RDS for
the database tier is explained in [`docs/unit-context.md`](docs/unit-context.md).

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

Detailed guidance is in [`CONTRIBUTING.md`](CONTRIBUTING.md) and
[`docs/team-workflow.md`](docs/team-workflow.md).

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
