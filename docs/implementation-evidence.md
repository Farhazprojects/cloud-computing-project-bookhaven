# BookHaven — Implementation Evidence

What was actually deployed in the **ap-southeast-2 (Sydney)** Region, and what
testing confirmed. This is the record behind the claims made in
[`requirements-mapping.md`](requirements-mapping.md).

Screenshots and figures referenced here are held in the assessment report rather
than in this repository.

---

## 1. Deployed configuration

| Component | Deployed as |
|---|---|
| VPC | `BookHaven-vpc`, 10.0.0.0/16 |
| Public subnets | 10.0.0.0/20, 10.0.16.0/20 |
| Private subnets | 10.0.128.0/20, 10.0.144.0/20 |
| Availability Zones | ap-southeast-2a, ap-southeast-2b |
| Load balancer | `BookHaven-ALB` — internet-facing, active, both public subnets |
| Target group | `BookHaven-WebTG` — HTTP:80 listener, both servers registered and healthy |
| Application servers | Two `BookHaven-AppServer` instances, `t3.micro`, Amazon Linux 2023, Apache + PHP |
| Auto Scaling group | `BookHaven-WebASG` — min 2, desired 2, max 4 |
| Scaling policy | `BookHaven-CPU-Target-Tracking` — 50% average CPU |
| Database | `bookhaven-db` — MySQL 8.4, `db.t4g.micro`, KMS-encrypted, private DB subnet group across both AZs |
| Instance role | `BookHaven-EC2-RDS-Role` |
| Admin access | `BookHaven-EICE` — EC2 Instance Connect Endpoint |
| VPC endpoints | Secrets Manager interface endpoint, S3 gateway endpoint |

---

## 2. What testing confirmed

### Web application

BookHaven is served over HTTPS through CloudFront. The page lists the catalogue
with prices and stock levels, flags low stock, and provides forms for adding
books and placing orders. **The footer reports which EC2 instance and
Availability Zone handled the request and how long the page took to generate** —
this is what made load distribution and shared-database behaviour observable.

### EC2 application servers

Two instances run in private subnets, one per Availability Zone, with **no
public IP addresses**. Both run Apache and PHP, start automatically after
reboot, listen on port 80 and serve the application.

### Load balancing

The ALB is internet-facing and active across both Availability Zones, with both
servers registered and healthy in the target group. **Repeated requests to the
same address were answered by different servers** — once by the ap-southeast-2a
instance and once by ap-southeast-2b — confirming traffic is shared across both
Zones with stickiness disabled.

### CloudFront and Route 53

A CloudFront distribution uses the ALB as its origin, redirects visitors from
HTTP to HTTPS, and forwards to the ALB on port 80 with caching disabled so
dynamic pages stay current. **Route 53 was not deployed**: no domain was
registered for the proof of concept, so no hosted zone was created and the
application is reached through the CloudFront distribution domain.

### Network segmentation

The public route table sends `0.0.0.0/0` to the BookHaven internet gateway;
the private route tables contain only the local route. **Browsing directly to an
application server's private address from the internet timed out**, confirming
the servers are not publicly reachable.

### Administrative access and Security Groups

Administrators reach the private servers through the EC2 Instance Connect
Endpoint rather than a bastion host. `BookHaven-EC2-SG` allows HTTP only from
the ALB security group and SSH only from the endpoint's security group;
`BookHaven-RDS-SG` allows MySQL 3306 only from `BookHaven-EC2-SG`.

### Database

The database is not publicly accessible, uses a private DB subnet group spanning
both Availability Zones, and its storage is encrypted with KMS. **A connection
test from an application server to the RDS endpoint on port 3306 succeeded.**
The Multi-AZ option was unavailable on the account's Free plan, so the proof of
concept runs single-AZ.

### Credential management

The database credential is an RDS-managed secret in Secrets Manager. The servers
use the instance role and the Secrets Manager interface endpoint to retrieve it
at runtime — **without internet access and without storing the password in
code**.

### Auto Scaling and monitoring

The Auto Scaling group manages both servers across the two private subnets, with
the target-tracking policy holding average CPU at 50% using CloudWatch metrics
and alarms.

---

## 3. Application functional tests

All of the following were tested against the deployed environment and worked as
expected:

| Test | Result |
|---|---|
| View the catalogue with prices and stock levels | Displayed correctly |
| Add a new book record | Records added successfully |
| Modify a book's price | Price updated from 29.99 to 35.00 |
| Delete a book record | Record removed, confirmation message shown |
| Place an order | Order accepted and **stock reduced automatically** from 20 to 17 for a quantity of 3 |
| Update an order status | Status changed to Delivered |
| Order more than available stock | **Correctly rejected** — 31 requested against 30 in stock returned an insufficient-stock message |
| Data added on one server visible from the other | Confirmed via the footer, showing both servers share one database |

The last test matters most architecturally: it confirms the two application
servers are genuinely stateless with respect to data, which is what makes
horizontal scaling behind a load balancer valid.

### Performance observed

Each page was generated in approximately **8 to 15 milliseconds**, including the
database query.

---

## 4. Demonstration video

A recorded walkthrough of the working application through the CloudFront
endpoint is available at <https://youtu.be/c9QRsO2iuDU>.

---

## 5. Limits of this evidence

- Testing was **manual and functional**, not a load test. Behaviour at the
  scaling boundary and under peak concurrency has not been measured.
- Failover was **not** exercised: no instance or Availability Zone was
  deliberately terminated to observe recovery.
- The database is **single-AZ** in this deployment, so database failover could
  not be tested at all.
- Timings come from a lightly loaded proof of concept and should not be read as
  production performance figures.
