# BookHaven — AWS Architecture

This document records the BookHaven architecture as designed and deployed in the
**ap-southeast-2 (Sydney)** Region.

It distinguishes throughout between the **target design** (what BookHaven would
run in production) and the **proof of concept** (what was actually deployed and
tested). Where the two differ, both are stated. Section 8 lists every difference
in one place.

---

## 1. The problem being solved

BookHaven is an online bookstore. Its web application manages the book
catalogue, customer orders and inventory, and was hosted on a **single
on-premises server**.

During peak trading — Black Friday and Boxing Day — the increase in users made
the application slow or unavailable, causing abandoned carts and lost sales. A
single server is both a capacity limit and a single point of failure.

The solution migrates hosting to AWS and replaces the single-server dependency
with a distributed architecture: multiple application instances behind an
Application Load Balancer, spread across two Availability Zones, with automatic
scaling and a private managed database.

The architecture deliberately keeps the **existing web/application/database
model** rather than rewriting BookHaven into microservices or serverless
functions. This is appropriate for a proof-of-concept migration because it
solves the stated hosting problem without forcing an unnecessary application
rewrite.

---

## 2. Request flow

**Target design**

```
Users
  │
  ▼
Amazon Route 53              custom BookHaven domain, alias record to CloudFront
  │
  ▼
Amazon CloudFront            HTTPS 443 at the edge
  │
  ▼
Application Load Balancer    HTTP 80, public subnets, both AZs
  │
  ├───────────────────────────────┐
  ▼                               ▼
EC2 application server      EC2 application server
ap-southeast-2a             ap-southeast-2b
(private subnet)            (private subnet)
  │                               │
  └───────────────┬───────────────┘
                  ▼
          Amazon RDS for MySQL      port 3306, private subnets
```

**Ports along the path**

```
User ──HTTPS 443──▶ CloudFront ──HTTP 80──▶ ALB ──HTTP 80──▶ EC2 ──MySQL 3306──▶ RDS
```

Visitors connect to CloudFront over HTTPS. CloudFront forwards each request to
the Application Load Balancer, which passes it to a healthy EC2 instance in
either Availability Zone. The PHP application queries RDS over the private
network to read or update catalogue, order and inventory records. The response
returns through the ALB and CloudFront to the user.

Before connecting to the database, the application retrieves its credentials
from **AWS Secrets Manager** through a private interface endpoint and holds them
in memory for five minutes, so no password is stored in code or on disk.

> **Proof of concept:** no domain was registered, so Route 53 was not deployed
> and the application is reached through the CloudFront distribution domain.

---

## 3. AWS service components

| Service | Purpose | Requirement addressed |
|---|---|---|
| **Amazon Route 53** | DNS entry point for a custom BookHaven domain. *Design only — no domain registered for the proof of concept.* | Traffic entry |
| **Amazon CloudFront** | Content delivery layer in front of the ALB. Provides HTTPS to visitors and redirects HTTP to HTTPS. Caching is **disabled for the dynamic PHP pages** so stock levels and orders stay current. | Performance, security |
| **Application Load Balancer** | Internet-facing ALB in the two public subnets. Its HTTP:80 listener forwards to the `BookHaven-WebTG` target group. | Load balancing, availability |
| **Amazon EC2** | Two `t3.micro` Amazon Linux 2023 servers running Apache and PHP, one in each private subnet. | Functional design, performance |
| **EC2 Auto Scaling** | `BookHaven-WebASG` holds two to four instances using target tracking at 50% average CPU. | Scalability, cost optimisation |
| **Amazon RDS for MySQL** | MySQL 8.4 on `db.t4g.micro`, in a private DB subnet group spanning both AZs, storage encrypted with KMS. | Functional design, availability |
| **Amazon VPC** | `BookHaven-vpc` (10.0.0.0/16) with two public and two private subnets across two Availability Zones. | Security, network isolation |
| **Security Groups** | Chain traffic by tier: internet → ALB → EC2 → RDS. No other inbound paths are open. | Security |
| **AWS Secrets Manager** | Holds the RDS-managed database credential. The application reads it at runtime through a private VPC endpoint. | Credential security |
| **AWS IAM** | The `BookHaven-EC2-RDS-Role` instance role lets the servers read the database secret without stored access keys. | Security |
| **Amazon CloudWatch** | Collects EC2 CPU metrics and runs the alarms used by the Auto Scaling target-tracking policy. | Performance, scaling, operations |
| **VPC endpoints** | A Secrets Manager interface endpoint and an S3 gateway endpoint let the private servers reach AWS services **without a NAT gateway**. | Security, cost optimisation |
| **EC2 Instance Connect Endpoint** | Private administrative SSH access to the application servers, with no public IPs and no bastion host. | Security |

---

## 4. Network design

`BookHaven-vpc` uses the CIDR block **10.0.0.0/16** and contains four subnets
across two Availability Zones:

```
BookHaven-vpc  10.0.0.0/16
├── ap-southeast-2a
│   ├── Public subnet   10.0.0.0/20      Application Load Balancer
│   └── Private subnet  10.0.128.0/20    EC2 application server, RDS subnet group
└── ap-southeast-2b
    ├── Public subnet   10.0.16.0/20     Application Load Balancer
    └── Private subnet  10.0.144.0/20    EC2 application server, RDS instance
```

### Public subnets

The public route table sends `0.0.0.0/0` to the BookHaven internet gateway.
These subnets hold **only the Application Load Balancer**. An internet-facing ALB
must be attached to a subnet in each Availability Zone it serves, which is why
there are two.

### Private subnets

The private route tables contain **only the local VPC route**. There is no
internet path in and no internet path out. These subnets hold the EC2
application servers and the RDS database.

The application servers have **no public IP addresses**. Browsing directly to an
application server's private address from the internet times out, confirming
that the servers are not publicly reachable.

Because the private subnets have no outbound internet route, the servers reach
AWS services through **VPC endpoints** instead of a NAT gateway — a Secrets
Manager interface endpoint for credentials, and an S3 gateway endpoint for
operating system packages.

### Why the boundary is drawn here

Each tier is exposed no more than its role requires:

- The load balancer must accept public traffic → public subnet.
- The application must accept traffic from the load balancer but not from the
  internet → private subnet.
- The database must accept traffic from the application only → private subnet,
  with the most restrictive rules of the three tiers.

---

## 5. Security design

The security requirements are that the database must not be directly accessible
from public networks, that the tiers must communicate on appropriate ports, that
the application must remain internet accessible, and that database credentials
must not be hardcoded.

| Control | Implementation and rationale |
|---|---|
| **Public exposure** | Only CloudFront and the internet-facing ALB receive public traffic. EC2 instances have no public IPs and RDS is not publicly accessible. |
| **Network segmentation** | Application servers and the database sit in private subnets whose route tables hold only the local VPC route, so there is no internet path in or out. |
| **Security Groups** | Rules chained by tier — see the table below. |
| **Credential management** | The database password is an RDS-managed secret in Secrets Manager, encrypted with KMS. The application retrieves it at runtime. A search of the web root and Apache configuration found no hardcoded credentials. |
| **IAM** | Application servers use an instance role scoped to the BookHaven database secret, so no AWS access keys are kept on the servers. |
| **Monitoring** | CloudWatch collects the EC2 metrics used by the Auto Scaling policy, and ALB health checks take unhealthy targets out of service. |
| **Administrative access** | Administrators connect through an EC2 Instance Connect Endpoint in the private subnet. There is no bastion host and SSH is not open to the internet. |
| **Encryption** | Visitors use HTTPS through CloudFront, RDS storage is encrypted with KMS, and the application's database connection uses TLS. |

### Security Group chain

Each Security Group references the **Security Group of the tier in front of it**
rather than an IP range, so the rules remain correct when Auto Scaling replaces
instances with new private addresses.

| Security Group | Inbound rule | Source |
|---|---|---|
| `BookHaven-ALB-SG` | HTTP 80 | Internet |
| `BookHaven-EC2-SG` | HTTP 80 | `BookHaven-ALB-SG` |
| `BookHaven-EC2-SG` | SSH 22 | `BookHaven-EICE-SG` |
| `BookHaven-RDS-SG` | MySQL 3306 | `BookHaven-EC2-SG` |
| `BookHaven-VPCE-SG` | HTTPS 443 | `BookHaven-EC2-SG` |

### Permitted and blocked paths

| Path | Status |
|---|---|
| HTTPS 443: Internet → CloudFront | Allowed (viewer traffic) |
| HTTP 80: Internet → ALB | Allowed by `ALB-SG` |
| HTTP 80: ALB → EC2 | Allowed by `EC2-SG`, source `ALB-SG` only |
| SSH 22: EC2 Instance Connect Endpoint → EC2 | Allowed by `EC2-SG`, source `EICE-SG` only |
| MySQL 3306: EC2 → RDS | Allowed by `RDS-SG`, source `EC2-SG` only |
| HTTPS 443: EC2 → Secrets Manager endpoint | Allowed by `VPCE-SG`, source `EC2-SG` only |
| **SSH 22 or MySQL 3306 from the internet** | **Blocked** — no rule, no public IP, no internet route |

---

## 6. Compute and scaling

`BookHaven-WebASG` provides horizontal scaling:

| Setting | Value |
|---|---|
| Minimum instances | 2 (one per Availability Zone) |
| Desired capacity | 2 |
| Maximum instances | 4 |
| Scaling policy | `BookHaven-CPU-Target-Tracking`, target 50% average CPU |
| Health checks | Both EC2 status and ELB status |
| Launch source | BookHaven launch template |

When traffic rises during a sale, new instances launch from the launch template
and register with the load balancer. When demand falls, the group scales back
in. Because health checks use ELB status as well as EC2 status, an instance that
stops serving pages is replaced automatically even if the instance itself is
still running.

---

## 7. Data tier

`bookhaven-db` runs **MySQL 8.4** on a `db.t4g.micro` instance. It is not
publicly accessible, uses a private DB subnet group spanning both Availability
Zones, and its storage is encrypted with KMS.

**Multi-AZ is part of the target design.** A synchronous standby in the second
Availability Zone would take over automatically if the primary failed. Because
the DB subnet group already spans both Zones, enabling Multi-AZ requires **no
network redesign**.

> **Proof of concept:** Multi-AZ could not be enabled on the AWS Free plan used
> for this project, so RDS currently runs as a single instance in
> ap-southeast-2b.

---

## 8. Target design versus proof of concept

| Component | Proof of concept status |
|---|---|
| **Amazon Route 53** | Included in the design for a custom domain. No domain was registered, so the application is reached through the CloudFront distribution domain. |
| **RDS Multi-AZ** | Not available on the AWS Free plan. RDS runs single-AZ inside a two-Zone subnet group, so Multi-AZ can be enabled in production without network changes. |
| **NAT gateway** | Not deployed, to reduce cost. Private servers reach Secrets Manager and S3 through VPC endpoints instead. |
| **HTTPS to the ALB** | Visitors use HTTPS through CloudFront; CloudFront reaches the ALB on HTTP 80. Production would add an ACM certificate to the ALB and accept traffic from CloudFront only. |
| **AWS WAF** | Not enabled, to control cost. Recommended for production. |

---

## 9. Diagrams

The architecture diagrams are maintained in Figma and exported to
[`../diagrams/`](../diagrams/):

- **Final AWS cloud architecture** — the full deployed architecture.
- **Traffic flow, public/private boundaries and security controls** — the
  request path and where each control applies.

See [`../diagrams/README.md`](../diagrams/README.md) for export conventions.

---

## 10. Related documents

- [`requirements-mapping.md`](requirements-mapping.md) — how this architecture
  satisfies each assessment requirement.
- [`implementation-evidence.md`](implementation-evidence.md) — what was deployed
  and what testing confirmed.
- [`unit-context.md`](unit-context.md) — unit background and the IaaS/PaaS
  reasoning behind the tier choices.
- [`references.md`](references.md) — sources cited.
