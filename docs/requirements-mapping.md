# BookHaven — Requirements Mapping

This document maps the agreed architecture to the assessment criteria. For each
criterion it states the requirement, the components that satisfy it, how they do
so, and the limits of what the current design achieves.

The architecture itself is described in [`architecture.md`](architecture.md).
Every component named here appears in that document and in the Figma diagram.

**Summary**

| Criterion | Primary components |
|---|---|
| Functional design | Route 53, CloudFront, ALB, EC2 Auto Scaling, RDS Multi-AZ |
| Load balancing | Application Load Balancer, EC2 Auto Scaling |
| Scalability | EC2 Auto Scaling, CloudWatch, CloudFront |
| High availability | Two Availability Zones, ALB, EC2 Auto Scaling, RDS Multi-AZ, Route 53 |
| Security | VPC and subnets, Security Groups, IAM, Secrets Manager, CloudFront |
| Cost optimisation | EC2 Auto Scaling, CloudFront, RDS Multi-AZ, CloudWatch |
| High performance | CloudFront, ALB, EC2 Auto Scaling, RDS Multi-AZ |

---

## 1. Functional design

**Requirement.** The architecture must support what customers and staff actually
need to do: browse and search the catalogue, place orders, and manage stock and
catalogue content.

**How the architecture meets it.**

The design is a conventional three-tier web architecture, with each tier matched
to the kind of work it performs.

| User action | Path through the architecture |
|---|---|
| Loading a page or book cover image | Route 53 → CloudFront (served from cache at the edge) |
| Searching the catalogue | Route 53 → CloudFront → ALB → EC2 → RDS |
| Placing an order | Route 53 → CloudFront → ALB → EC2 → RDS (write) |
| Staff updating stock | Route 53 → CloudFront → ALB → EC2 → RDS (write) |

Route 53 gives the site a usable domain name. CloudFront serves the static parts
of the shop directly. The Application Load Balancer routes dynamic requests to
the application tier. EC2 instances run the business logic — search, basket,
checkout, stock management. RDS holds the data that must persist and stay
consistent, such as stock levels and order records, which is why a relational
database is appropriate rather than a simple file store.

Staff and customers share one entry point. Staff functions are separated inside
the application, and any staff access to AWS itself is controlled through IAM.

**Limits.** The functional design is documented at architecture level. Detailed
application design — data model, endpoints, screens — is not part of this stage.

---

## 2. Load balancing

**Requirement.** Incoming traffic must be distributed across the application
tier so that no single instance is overwhelmed and no single instance is a point
of failure.

**How the architecture meets it.**

The **Application Load Balancer** is the component responsible for distribution.
It:

- Distributes requests across all registered EC2 instances in both Availability
  Zones, so load is shared rather than concentrated.
- Runs **health checks** against each instance and stops sending traffic to any
  instance that fails, so a failing instance does not receive requests while it
  is being replaced.
- Works with **EC2 Auto Scaling** so that instances launched by scaling are
  registered automatically and instances being removed are drained first,
  allowing in-flight requests to complete.

It is an *Application* Load Balancer specifically because it operates at the
HTTP layer, which is what allows it to health check an application path rather
than only a network port.

**Limits.** Health check paths, thresholds and intervals are not yet agreed.

---

## 3. Scalability

**Requirement.** The system must handle changes in demand — quiet weekday
evenings, and peaks such as a promotion or the run-up to Christmas — without
manual intervention.

**How the architecture meets it.**

Scaling happens at two levels.

**Application tier.** The **EC2 Auto Scaling** group has a minimum, desired and
maximum size. **CloudWatch** metrics, such as average CPU utilisation or request
count per instance, are watched by alarms. When an alarm threshold is crossed,
the Auto Scaling group adds instances; when demand falls, it removes them. New
instances are registered with the load balancer automatically and begin
receiving traffic once they pass health checks.

```
Demand rises → CloudWatch metric crosses threshold → alarm →
Auto Scaling adds instances → ALB registers and health checks them →
traffic is shared across more instances
```

**Edge.** **CloudFront** absorbs demand for static content at the edge. A cached
book cover served from an edge location never reaches the load balancer or the
EC2 instances at all, so a traffic spike on browsing pages does not
automatically become a spike on the application tier.

The dependency worth stating explicitly is that scalability comes from Auto
Scaling **and** CloudWatch together. Without CloudWatch the group has no signal
telling it when to act.

**Limits.** RDS is not horizontally scaled in this design; the Multi-AZ standby
provides availability, not additional read capacity. Scaling thresholds and
cooldown periods are still to be agreed.

---

## 4. High availability

**Requirement.** The service must remain available when individual components
fail, including the loss of an entire Availability Zone.

**How the architecture meets it.**

Every tier is duplicated across **two Availability Zones**, which are physically
separate data centres within the region.

| Failure | What happens |
|---|---|
| One EC2 instance fails | ALB health check fails, traffic stops going to it, Auto Scaling replaces it |
| Demand exceeds capacity | Auto Scaling adds instances in both AZs |
| The RDS primary fails | RDS promotes the Multi-AZ standby; the endpoint is unchanged |
| An entire Availability Zone is lost | The ALB routes to instances in the surviving AZ; RDS fails over if the primary was in the lost AZ |
| The application endpoint becomes unreachable | Route 53 health checks support DNS-level failover |

The important characteristic is that recovery is **automatic** in each case. No
member of the team needs to intervene for the service to keep running, which is
what distinguishes high availability from simply having a backup.

**Limits.** This is a single-region design. A region-wide failure is not covered,
and multi-region operation would require services beyond the agreed list.
Recovery time and recovery point objectives have not yet been defined.

---

## 5. Security

**Requirement.** Customer and order data must be protected, and access to both
the system and the AWS account must be controlled.

**How the architecture meets it.**

Security is applied in layers, so no single control is the only thing preventing
an incident.

**Network layer.** The **VPC** divides the environment into public and private
subnets. Only the Application Load Balancer sits in a public subnet. The EC2
instances and RDS databases sit in private subnets with no route allowing
inbound connections from the internet, so they cannot be reached directly
however a request is constructed.

**Traffic layer.** Three **Security Groups** form a chain in which each tier
accepts traffic only from the tier in front of it: the internet reaches the load
balancer, the load balancer reaches the application, and only the application
reaches the database. Because the rules reference Security Groups rather than IP
addresses, they stay correct as Auto Scaling replaces instances.

**Identity layer.** **IAM** gives each team member an individual identity with
least-privilege permissions and multi-factor authentication, and gives EC2
instances a **role** rather than stored credentials. Actions in the account are
attributable to a specific identity.

**Credential layer.** **Secrets Manager** holds the database credentials. The
application retrieves them at runtime using its IAM role, so no long-lived
credential is stored on an instance, in application configuration, or in this
repository.

**Edge layer.** **CloudFront** serves as the single public entry point over
HTTPS and keeps the origin from being addressed directly by end users.

**Monitoring layer.** **CloudWatch** records logs and metrics centrally, so
unusual activity is visible and evidence survives the termination of an
instance.

**Limits.** Encryption settings, certificate management and formal access
reviews are still to be agreed. Application-level security, such as input
validation and session handling, is a development concern outside this
architecture.

---

## 6. Cost optimisation

**Requirement.** The design should avoid paying for capacity that is not being
used, while still meeting the availability requirement.

**How the architecture meets it.**

**Paying for demand, not for peaks.** **EC2 Auto Scaling** removes instances
when demand falls. Without it, the team would have to run enough instances for
the busiest expected hour at all times, and pay for that capacity overnight and
in quiet periods.

**Reducing work that reaches the paid tiers.** **CloudFront** serves cached
content from the edge. Requests answered from cache do not consume load balancer
capacity, EC2 compute or database connections, so caching reduces the size the
application tier needs to be.

**Managed services instead of self-managed ones.** **RDS Multi-AZ** provides
replication, failover and backups as part of the service. Building the
equivalent on EC2 instances would require additional instances and considerably
more of the team's time.

**Visibility.** **CloudWatch** shows actual utilisation, which is what allows
over-provisioning to be identified rather than assumed. A consistently
low-utilisation Auto Scaling group is evidence that the minimum size or the
instance type can be reduced.

**The trade-off, stated honestly.** High availability is not free. Multi-AZ RDS
runs a standby instance that serves no traffic, and running instances in two
Availability Zones costs more than concentrating them in one. The team has
accepted this cost because availability is a stated requirement; the design
optimises cost *within* that constraint rather than pursuing the cheapest
possible architecture.

**Limits.** No pricing analysis, budget or purchasing model comparison has been
carried out at this stage.

---

## 7. High performance

**Requirement.** Pages and searches should respond quickly, and response times
should not degrade sharply as demand rises.

**How the architecture meets it.**

**Reducing distance.** **CloudFront** serves cached content from the edge
location nearest the user. This shortens the physical round trip, which is
usually the largest single contributor to page load time for static assets.

**Reducing queueing.** The **Application Load Balancer** spreads requests across
instances so that requests are not waiting behind others on one busy instance
while another sits idle.

**Adding capacity before performance degrades.** **EC2 Auto Scaling**, driven by
**CloudWatch** metrics, adds instances as load increases, so response times stay
within range rather than climbing as a fixed number of instances saturate.

**Keeping data access efficient.** **RDS** is a managed relational database on
dedicated instances, sized for the workload. Placing it in the same region and
private subnets as the application keeps database round trips within the AWS
network.

**Measuring rather than assuming.** **CloudWatch** records response times and
request counts, so performance claims can be evidenced and regressions detected.

**Limits.** No load testing has been carried out and no performance targets have
been agreed. Cache behaviours, instance types and database sizing all remain
open, and performance in practice will depend on those decisions.

---

## 8. Traceability

Every criterion above is satisfied by components in the agreed list. No service
appears in this mapping that is not documented in
[`architecture.md`](architecture.md) and shown in the architecture diagram.

If the architecture changes, this mapping and the diagram must be updated in the
same pull request, so that the three remain consistent.
