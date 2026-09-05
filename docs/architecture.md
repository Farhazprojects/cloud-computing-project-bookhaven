# BookHaven — Agreed Architecture

This document records the architecture agreed by the team. It is the written
counterpart to the Figma diagram, and the two must always describe the same
design.

Only the AWS services listed in the [README](../README.md) appear here. Any
additional service must be proposed in an issue and agreed by the team before it
is added to this document or to the diagram.

---

## 1. Request flow

The agreed path a request takes through the system:

```
Customers and Staff
        │
        ▼
    Internet
        │
        ▼
   Route 53                    DNS resolution for the BookHaven domain
        │
        ▼
   CloudFront                  Edge caching and entry point for all traffic
        │
        ▼
Application Load Balancer      Distributes traffic across healthy instances
        │
        ├──────────────────────────────┐
        ▼                              ▼
  EC2 Auto Scaling             EC2 Auto Scaling      Application tier,
  Availability Zone A          Availability Zone B   spread across two AZs
        │                              │
        └──────────────┬───────────────┘
                       ▼
              Amazon RDS Multi-AZ        Primary database with a
              (primary + standby)        standby in the second AZ
```

### Stage by stage

**Customers and Staff → Internet.** Both user groups reach BookHaven over the
public internet through a web browser. Staff use the same entry point as
customers; their additional privileges are handled inside the application and
through IAM for any AWS-level access, not by a separate network path.

**Internet → Route 53.** Route 53 provides authoritative DNS for the BookHaven
domain. A request for the site is resolved by Route 53 to the CloudFront
distribution. Route 53 health checks allow DNS-level failover if the endpoint
becomes unavailable.

**Route 53 → CloudFront.** CloudFront is the single public entry point. It
serves cached static content — images, stylesheets, scripts, book cover art —
from the edge location nearest the user. Requests that cannot be served from
cache, such as searches, basket updates and order submissions, are forwarded to
the origin. Because CloudFront terminates the connection at the edge, the origin
is not exposed directly to end users.

**CloudFront → Application Load Balancer.** The load balancer is the origin for
the CloudFront distribution. It receives forwarded requests and distributes them
across the registered EC2 instances. It performs health checks on each instance
and stops sending traffic to any instance that fails, which is what allows a
failing instance to be replaced without a visible outage.

**Application Load Balancer → EC2 Auto Scaling.** The application runs on EC2
instances managed by an Auto Scaling group that spans two Availability Zones.
The group has a minimum, desired and maximum size. It adds instances when demand
rises, removes them when demand falls, and replaces any instance that fails a
health check.

**EC2 Auto Scaling → Amazon RDS Multi-AZ.** The application tier reads and
writes catalogue, customer and order data through the RDS endpoint. RDS is
deployed Multi-AZ: a primary database instance in one Availability Zone with a
synchronously replicated standby in the other. If the primary fails, RDS
promotes the standby and the endpoint continues to resolve, so the application
does not need to be reconfigured.

---

## 2. Network boundaries: public and private subnets

The VPC spans two Availability Zones. Each Availability Zone contains one public
subnet and one private subnet, giving four subnets in total.

```
VPC
├── Availability Zone A
│   ├── Public subnet A     Application Load Balancer node
│   └── Private subnet A    EC2 instances, RDS primary
└── Availability Zone B
    ├── Public subnet B     Application Load Balancer node
    └── Private subnet B    EC2 instances, RDS standby
```

### Public subnets

The public subnets are the only subnets with a route to the internet. They
contain **only the Application Load Balancer nodes**. Nothing that stores or
processes BookHaven data is placed here.

Two public subnets are required because an Application Load Balancer must be
attached to a subnet in each Availability Zone it serves.

### Private subnets

The private subnets have **no route allowing inbound connections from the
internet**. They contain:

- The **EC2 instances** running the application.
- The **RDS primary and standby** database instances.

An EC2 instance in a private subnet cannot be reached directly from the
internet. The only way a user request arrives at the application is through
CloudFront and then the load balancer. This means an attacker cannot bypass
those layers to reach the application or the database, however the request is
constructed.

### Why the boundary is drawn here

The design follows the principle that a resource should be no more exposed than
its role requires:

- The load balancer must accept public traffic, so it sits in the public subnet.
- The application must accept traffic from the load balancer, but not from the
  internet, so it sits in a private subnet.
- The database must accept traffic from the application only, so it also sits in
  a private subnet with the most restrictive rules of the three tiers.

Spreading each tier across two Availability Zones means the loss of a single
Availability Zone removes capacity but does not remove the service.

---

## 3. Security Groups

Security Groups act as virtual firewalls attached to each resource. They are
**stateful**, so a permitted inbound request is automatically allowed to return,
and they **deny everything that is not explicitly allowed**.

Three Security Groups are used, each referencing the one above it rather than an
IP range. This is the key design decision in this section: because the rules
name the *source Security Group* rather than a set of addresses, they remain
correct when Auto Scaling replaces instances with new private IP addresses.

| Security Group | Attached to | Inbound rule | Source |
|---|---|---|---|
| `bookhaven-alb-sg` | Application Load Balancer | HTTPS (443) | Internet |
| `bookhaven-app-sg` | EC2 application instances | Application port | `bookhaven-alb-sg` |
| `bookhaven-db-sg` | RDS instances | Database port | `bookhaven-app-sg` |

The effect is a chain in which each tier will only accept traffic from the tier
directly in front of it:

```
Internet ──HTTPS──▶ alb-sg ──app port──▶ app-sg ──db port──▶ db-sg
```

The database Security Group has no rule permitting traffic from the load
balancer or from the internet, so there is no configuration in which the
database can be reached without first passing through the application tier.

---

## 4. IAM

IAM controls who and what may act within the AWS account.

**For team members.** Each member of the team has an individual IAM identity
with permissions appropriate to their role. Credentials are never shared between
members, so any action in the account can be attributed to one person. The root
account is not used for routine work, and multi-factor authentication is
required.

**For EC2 instances.** The application instances are granted an **IAM role**
rather than stored credentials. The role is assumed by the instance, and AWS
supplies temporary credentials that are rotated automatically. This is the
mechanism that allows the application to retrieve its database credentials from
Secrets Manager and to publish logs and metrics to CloudWatch **without any
long-lived access key existing anywhere in the system**, including in this
repository.

**Least privilege.** Every policy grants only the permissions the identity
actually needs. The instance role, for example, is limited to reading the
specific BookHaven secret rather than reading all secrets in the account.

---

## 5. Secrets Manager

Secrets Manager stores the database credentials used by the application to
connect to RDS.

The sequence at runtime is:

1. The application starts on an EC2 instance that carries the IAM instance role.
2. The application requests the BookHaven database secret from Secrets Manager.
3. IAM checks the role's permissions and allows the request.
4. Secrets Manager returns the credentials, which are held in memory and used to
   open the database connection.

The credentials therefore exist only in Secrets Manager and in the memory of a
running instance. They are never written into application configuration files,
never stored on an instance disk, and never committed to this repository. When a
credential needs to change, it is updated in Secrets Manager and instances pick
up the new value, rather than requiring a code change and redeployment.

---

## 6. CloudWatch

CloudWatch provides the monitoring layer and closes the loop that makes the
architecture responsive rather than merely redundant.

**Metrics.** CloudWatch collects metrics from the components of the
architecture — CPU utilisation and instance health from EC2 Auto Scaling,
request counts, response times and healthy host counts from the Application Load
Balancer, connection counts and storage from RDS, and cache and error rates from
CloudFront.

**Logs.** Application and system logs are sent to CloudWatch Logs, giving a
single searchable location. This matters because Auto Scaling terminates
instances; logs written only to an instance disk are lost when that instance is
replaced.

**Alarms and scaling.** Alarms watch metrics against thresholds. The important
connection is between CloudWatch and EC2 Auto Scaling:

```
CloudWatch metric  ──▶  CloudWatch alarm  ──▶  Auto Scaling action
(e.g. average CPU)      (threshold crossed)     (add or remove instances)
```

Without CloudWatch the Auto Scaling group has no signal telling it when to act,
so scalability depends on this link rather than on the Auto Scaling group alone.
Alarms also notify the team when a condition needs human attention, such as a
sustained rise in error responses or an Availability Zone losing healthy hosts.

---

## 7. Summary of responsibilities

| Component | Responsibility | Placement |
|---|---|---|
| Route 53 | DNS resolution and health-checked failover | AWS global |
| CloudFront | Edge caching, single public entry point | AWS edge locations |
| Application Load Balancer | Traffic distribution, instance health checks | Public subnets, both AZs |
| EC2 Auto Scaling | Runs the application, adjusts capacity | Private subnets, both AZs |
| Amazon RDS Multi-AZ | Data storage with automatic failover | Private subnets, both AZs |
| VPC and subnets | Network isolation and the public/private boundary | Regional |
| Security Groups | Tier-to-tier traffic control | Attached to each tier |
| IAM | Identity, roles and least-privilege permissions | Account-wide |
| Secrets Manager | Database credential storage and retrieval | Regional |
| CloudWatch | Metrics, logs, alarms and the scaling signal | Regional |

---

## 8. Scope and open points

**Agreed and documented.** The request flow, the subnet boundaries, the Security
Group chain, and the roles of IAM, Secrets Manager and CloudWatch as set out
above.

**Not yet decided.** Instance sizes, Auto Scaling thresholds and cooldown
periods, RDS engine and instance class, CloudFront cache behaviours, and the
specific CloudWatch alarm thresholds. These require agreement as the design is
refined and should each be raised as an issue.

**Deliberately out of scope at this stage.** Application code, deployment
scripts and infrastructure-as-code. Any AWS service not listed in the README.
