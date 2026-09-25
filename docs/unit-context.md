# Unit and Assessment Context

This document records the unit this project belongs to, and the background from
earlier work in the unit that informs the BookHaven design.

It is **background and context only**. The agreed BookHaven architecture is
defined in [`architecture.md`](architecture.md), and nothing in this document
adds to or changes the services listed there.

---

## 1. Unit details

| | |
|---|---|
| **Unit code** | COIT20260 |
| **Unit name** | Cloud Computing and Internet of Things for Smart Applications |
| **Institution** | CQUniversity |
| **Campus** | Sydney |
| **Project** | BookHaven — team cloud architecture design |

---

## 2. Where BookHaven sits in the unit

The unit builds up in stages. Earlier individual work covered cloud service
models and a single provider; BookHaven applies that understanding to a team
architecture design on AWS.

| Stage | Focus | Relationship to BookHaven |
|---|---|---|
| **Assessment 1** (individual) | Comparing cloud service providers; IaaS and PaaS service models; deploying a simple web application to a managed platform; ethical and DEI considerations | Provides the service-model vocabulary and the ethical framing used here |
| **Assessment 2** (group — this repository) | Designing and deploying a scalable, load-balanced and secure AWS hosting architecture for BookHaven | Current work |

### Assessment 1 — what it covered

Assessment 1 was an individual report comparing two cloud service providers that
offer both **Infrastructure as a Service (IaaS)** and **Platform as a Service
(PaaS)**. One provider was selected for closer investigation, its IaaS and PaaS
services were described and demonstrated, it was compared with AWS, and a simple
web application was deployed to a managed platform. The report closed with
diversity, equity and inclusion (DEI) considerations in cloud computing.

Three things from that work carry directly into BookHaven: the **IaaS/PaaS
distinction** used in Section 3 below to justify the tier choices, the
**provider comparison** that supports building on AWS, and the **ethical and
sustainability framing** in Section 5.

### Assessment 2 — what this repository documents

Assessment 2 is the group project. BookHaven is an online bookstore whose
existing application ran on a **single on-premises server** and became slow or
unavailable during Black Friday and Boxing Day peaks. The project migrates
hosting to AWS and replaces the single-server dependency with a distributed,
load-balanced, multi-AZ architecture.

The design was built and tested as a proof of concept in the ap-southeast-2
(Sydney) Region. See [`architecture.md`](architecture.md) and
[`implementation-evidence.md`](implementation-evidence.md).

---

## 3. Cloud service models applied to BookHaven

Earlier work in the unit distinguished **Infrastructure as a Service (IaaS)**
from **Platform as a Service (PaaS)**:

- **IaaS** provides raw infrastructure — virtual machines, storage, networking.
  The customer keeps control of the operating system and what runs on it, and
  keeps responsibility for maintaining it.
- **PaaS** provides a managed environment. The provider handles the underlying
  infrastructure, and the customer supplies the application.

The general principle is that **IaaS gives greater control, and PaaS gives less
operational responsibility**. Neither is better in the abstract; the right
choice depends on how much control a given tier actually needs.

BookHaven deliberately uses both, tier by tier:

| BookHaven tier | Service model | Why this model was chosen |
|---|---|---|
| Application tier — **EC2 Auto Scaling** | Closer to **IaaS** | The team keeps control of the operating system and application runtime, which is needed to install and configure the bookshop application. The trade-off accepted is that the team is responsible for maintaining those instances. |
| Database tier — **Amazon RDS Multi-AZ** | Closer to **managed / PaaS** | Replication, failover and backups are handled by AWS. Building the equivalent on EC2 instances would require more instances and considerably more of the team's time, for no additional capability. |

This is the reasoning behind the cost-optimisation argument in
[`requirements-mapping.md`](requirements-mapping.md): managed services are
preferred where the team gains nothing from managing the layer itself.

---

## 4. Provider comparison background

Assessment 1 compared cloud providers at a general level. Two points from that
comparison are directly relevant to BookHaven:

- **AWS offers a broad range of mature enterprise services**, and is well suited
  to enterprise workloads and infrastructure flexibility. This supports the
  team's decision to design BookHaven on AWS.
- **AWS Auto Scaling adjusts EC2 instance counts in response to demand**, which
  is the mechanism BookHaven relies on for scalability.

> **Scope note.** Assessment 1 referred to services from other cloud providers,
> and to AWS services that BookHaven does not use, purely for comparison. Those
> services are **not** part of the BookHaven architecture and must not appear in
> [`architecture.md`](architecture.md), in
> [`requirements-mapping.md`](requirements-mapping.md), or in the diagrams. The
> service list in [`architecture.md`](architecture.md) is authoritative.

---

## 5. Ethical, accessibility and sustainability considerations

Assessment 1 examined diversity, equity and inclusion (DEI) in cloud computing
and identified four themes. They are recorded here because they apply to
BookHaven as an online shop serving the general public, even though Assessment 2
is architectural rather than application-level.

The specific tools named in Assessment 1 belonged to a different cloud provider
and are not used by BookHaven. What transfers is the **principles**, not the
products.

| Theme | What it meant in the earlier work | How it applies to BookHaven |
|---|---|---|
| **Fairness** | Tools that detect and document bias in AI models across demographic groups | Any future recommendation or personalisation feature should be checked for bias before it influences what customers are shown |
| **Accessibility** | Assistive and multilingual technology supporting users who are deaf, hard of hearing or visually impaired | The bookshop interface should be usable with assistive technology; an application-design requirement, not yet addressed |
| **Inclusiveness** | Broad language support preventing exclusion of diverse users | Content and interface language should not assume a single user group |
| **Trust and transparency** | Documenting how automated systems behave | Being clear with customers about what data is collected and how it is used |

**Sustainability** was also identified as an ethical consideration. It connects
to BookHaven's design through cost optimisation: **EC2 Auto Scaling** removes
instances when demand falls, so the architecture consumes compute in proportion
to actual demand rather than running permanently at peak capacity.

> **Scope note.** These are considerations recorded for the application layer.
> They are not yet designed for, and no service has been added to the
> architecture on their account.

---

## 6. Referencing convention

Work in this unit uses the **Harvard (CQUniversity) author–date** style. The same
convention applies to any source cited in this repository, so that the
documentation is consistent with the written assessment submission.

Sources cited in the documentation are listed in
[`references.md`](references.md).

**In-text citation**

```
Auto Scaling adjusts capacity in response to demand (Amazon Web Services 2026).
```

**Reference list entry**

```
Amazon Web Services 2026, *Amazon EC2 Auto Scaling*, Amazon Web Services,
viewed 25 September 2026, <https://docs.aws.amazon.com/autoscaling/>.
```

Add a reference whenever documentation states a factual claim about how an AWS
service behaves that a reader might reasonably want to verify.
