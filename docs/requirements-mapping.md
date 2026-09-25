# BookHaven — Requirements Mapping

This document maps the BookHaven architecture to the seven assessment
requirements. For each requirement it states what is required, the components
that satisfy it, the evidence from testing, and the limits of what the proof of
concept demonstrates.

The architecture itself is described in [`architecture.md`](architecture.md),
and deployment evidence in
[`implementation-evidence.md`](implementation-evidence.md).

---

## Summary

| Requirement | How the architecture satisfies it |
|---|---|
| **Functional design** | EC2 hosts the BookHaven PHP application; RDS stores the book and order tables. Viewing, adding, modifying and deleting books, placing and updating orders, and stock checks were all tested successfully. |
| **Load balanced** | The ALB distributes requests across healthy EC2 instances in two Availability Zones; testing showed responses returning from both servers. |
| **Scalable** | `BookHaven-WebASG` scales between two and four instances using a 50% CPU target-tracking policy. |
| **Highly available** | EC2 capacity spans two Availability Zones and unhealthy instances are replaced automatically. RDS uses a two-Zone subnet group, with Multi-AZ planned for production. |
| **Secure** | EC2 and RDS are private; Security Groups chain ALB → EC2 → RDS; credentials come from Secrets Manager through a private endpoint; administrator access uses an EC2 Instance Connect Endpoint. |
| **Cost optimised** | Auto Scaling, small instance classes, and VPC endpoints in place of a NAT gateway keep running costs low. |
| **High performing** | CloudFront, load distribution and scalable application capacity support peak workloads; pages were generated in under 15 milliseconds during testing. |

---

## 1. Functional design

**Requirement.** The architecture must support what the bookshop actually does:
manage the book catalogue, customer orders and inventory.

**How it is met.** EC2 instances host the BookHaven PHP application and Amazon
RDS stores the book and order tables. The application tier handles business
operations — catalogue display, stock checks, order placement and order status
updates — while RDS holds the data that must persist and stay consistent.

**Evidence.** All core functions were tested against the deployed environment:
viewing the catalogue with prices and stock levels, adding a book record,
modifying a book's price, deleting a book record, placing an order (which
reduced stock automatically), updating an order status, and rejecting an order
larger than the available stock.

**Limits.** The application is a proof-of-concept PHP application, not a
production bookshop. Payment processing, customer accounts and fulfilment are
out of scope.

---

## 2. Load balancing

**Requirement.** Traffic must be distributed across the application tier so that
no single instance is a bottleneck or a single point of failure.

**How it is met.** The **Application Load Balancer** spreads incoming requests
across the healthy EC2 instances registered in the `BookHaven-WebTG` target
group, which removes the single-server bottleneck that caused the original
problem. Stickiness is **disabled**, so consecutive requests can be served from
different Availability Zones. Health checks stop traffic reaching an instance
that has stopped serving pages.

**Evidence.** Repeated requests to the same address were answered by the server
in ap-southeast-2a and by the server in ap-southeast-2b. The application footer
reports which instance and Availability Zone handled each request, which is how
distribution was confirmed.

**Limits.** Load distribution was confirmed by repeated manual requests, not by
a formal load test.

---

## 3. Scalability

**Requirement.** The system must absorb the Black Friday and Boxing Day peaks
that made the original single server slow or unavailable.

**How it is met.** `BookHaven-WebASG` provides horizontal scaling between a
minimum of two instances, one per Availability Zone, and a maximum of four. A
target-tracking policy holds average CPU utilisation at 50%: when traffic rises,
new instances launch from the BookHaven launch template and register with the
load balancer; when demand falls, the group scales back in.

```
Demand rises → CloudWatch CPU metric crosses target → target-tracking policy →
Auto Scaling launches instances → ALB registers and health checks them →
load shared across more instances
```

Scalability therefore depends on **Auto Scaling and CloudWatch together** —
without CloudWatch metrics the group has no signal telling it when to act.

**Limits.** The maximum of four instances was chosen for a proof of concept on a
Free plan; production sizing would be based on measured peak demand. No load
test was run to confirm behaviour at the scaling boundary.

---

## 4. High availability

**Requirement.** The service must survive the failure of an instance or an
entire Availability Zone.

**How it is met.** Application instances run in **two Availability Zones**. If a
server or a Zone fails, the load balancer stops routing to the unhealthy target
while the other instance keeps serving users, and the Auto Scaling group
launches a replacement. Recovery is automatic; no administrator action is
required.

The database sits in a DB subnet group spanning both Zones.

| Failure | Response |
|---|---|
| One EC2 instance fails | ALB health check fails → traffic stops → Auto Scaling replaces it |
| An entire Availability Zone is lost | ALB routes to the instance in the surviving Zone |
| RDS primary fails *(target design)* | Multi-AZ promotes the standby; the endpoint is unchanged |

**Limits — stated plainly.** Multi-AZ could not be enabled on the AWS Free plan,
so **in the proof of concept RDS is a single instance in ap-southeast-2b and is
a single point of failure**. The application tier is genuinely highly available;
the database tier is not yet. Because the DB subnet group already spans both
Zones, enabling Multi-AZ in production needs no network redesign.

This is also a single-region design. A region-wide failure is not covered.

---

## 5. Security

**Requirement.** The database must not be reachable from public networks, tiers
must communicate on appropriate ports, the application must stay internet
accessible, and credentials must not be hardcoded.

**How it is met — in layers.**

- **Network.** EC2 and RDS sit in private subnets whose route tables hold only
  the local VPC route. The servers have no public IP addresses. Direct access to
  a private address from the internet times out.
- **Traffic.** Security Groups chain by tier: `ALB-SG` accepts HTTP 80 from the
  internet, `EC2-SG` accepts HTTP 80 only from `ALB-SG` and SSH 22 only from
  `EICE-SG`, and `RDS-SG` accepts MySQL 3306 only from `EC2-SG`. Because rules
  reference Security Groups rather than IP addresses, they stay correct as Auto
  Scaling replaces instances.
- **Credentials.** The database password is an RDS-managed secret in Secrets
  Manager, encrypted with KMS, retrieved at runtime through a private interface
  endpoint. A search of the web root and Apache configuration found **no
  hardcoded credentials**.
- **Identity.** The `BookHaven-EC2-RDS-Role` instance role is scoped to the
  BookHaven secret, so no AWS access keys exist on the servers.
- **Administrative access.** An EC2 Instance Connect Endpoint replaces a bastion
  host. SSH is not open to the internet.
- **Encryption.** HTTPS for visitors through CloudFront, KMS-encrypted RDS
  storage, TLS on the database connection.

**Limits.** AWS WAF was not enabled, to control cost, and is recommended for
production. CloudFront reaches the ALB over HTTP 80; production would add an ACM
certificate to the ALB and restrict it to CloudFront traffic only.
Application-level security — input validation, session handling — is a
development concern outside this architecture.

---

## 6. Cost optimisation

**Requirement.** Avoid paying for capacity that is not being used, while still
meeting the availability requirement.

**How it is met.**

- **Capacity follows demand.** Auto Scaling removes instances when demand falls,
  so BookHaven does not run permanently at peak size. This is the direct
  cost consequence of solving the scalability requirement.
- **Right-sized instances.** The proof of concept uses `t3.micro` for EC2 and
  `db.t4g.micro` for RDS.
- **VPC endpoints instead of a NAT gateway.** A free S3 gateway endpoint
  provides operating-system package access and a Secrets Manager interface
  endpoint provides credential access. This costs considerably less than running
  a NAT gateway in each Availability Zone, which would otherwise be needed to
  give private subnets outbound access.
- **Managed services.** RDS removes the need to administer database servers
  directly, which would require additional instances and staff time.
- **Deferred extras.** AWS WAF and a custom domain were left out of the proof of
  concept to control cost, and are recorded as production enhancements.

**The trade-off, stated honestly.** High availability is not free — running
across two Availability Zones costs more than one Zone, and production Multi-AZ
will add a standby that serves no traffic. The design optimises cost *within*
the availability requirement rather than pursuing the cheapest possible
architecture.

**Limits.** No costing model or budget analysis was produced, and no purchasing
options such as Savings Plans were compared.

---

## 7. High performance

**Requirement.** Pages should respond quickly under normal, variable and peak
load.

**How it is met.**

- **CloudFront** terminates HTTPS at edge locations close to users, shortening
  the round trip. Caching **can** be enabled for static assets such as images
  and stylesheets; it is **deliberately disabled for the dynamic PHP pages** so
  that stock levels and order data are always current. This is a correctness
  decision taken over a performance one.
- **The ALB** spreads requests so none queue behind a busy instance while
  another sits idle.
- **Auto Scaling**, driven by CloudWatch, adds capacity as load rises so
  response times stay within range instead of degrading as a fixed fleet
  saturates.
- **RDS** runs on a dedicated instance in the same Region and private subnets as
  the application, keeping database round trips inside the AWS network.

**Evidence.** In testing, each page was generated in approximately **8 to 15
milliseconds**, including the database query — no perceivable delay for users.

**Limits.** Those timings are from light manual testing on a proof of concept,
not from a load test at peak volume. No formal performance targets were agreed.

---

## 8. Traceability

Every requirement above is satisfied by components documented in
[`architecture.md`](architecture.md) and shown in the architecture diagrams.

Where the proof of concept differs from the target design — Route 53, RDS
Multi-AZ, NAT gateway, HTTPS to the ALB, and AWS WAF — the difference is
recorded in Section 8 of the architecture document and repeated in the relevant
requirement above, rather than being left implicit.
