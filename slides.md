---
theme: default
title: Redshift Data Sharing & Workload Isolation
background: https://cover.sli.dev
info: |
  AWS Redshift lecture
class: text-center
drawings:
  persist: false
transition: slide-left
---

# Redshift Data Sharing  
### Solving Workload Contention at Scale

---

# Data sharing in Redshift

> "Data sharing lets you share live data, without having to create a copy or move it."
>
> Source: [AWS Redshift Developer Guide — Data sharing in Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/dg/datashare-overview.html)

---

# Data sharing in Redshift

```text
  +----------------------+           +----------------------+
  | Producer Cluster     |           | Consumer Cluster     |
  |                      |           |                      |
  | Local tables/views   |           | CREATE DATABASE ...  |
  |                      |           | FROM DATASHARE       |
  +----------+-----------+           +----------+-----------+
             |                                  |
             | creates                          | references
             v                                  |
        +--------------------------+            |
        | Datashare: Sales Data    |<-----------+
        | Thin metadata layer      |
        | No copied table data     |
        +------------+-------------+
                     |
                     | points to
                     v
        +--------------------------+
        | Redshift Managed Storage |
        | S3-backed live data      |
        +--------------------------+
```

- `Sales Data` is the thin sharing layer, not a copied dataset
- The consumer references the datashare while the producer remains the data owner

---

# Setup — Producer Cluster

```sql
CREATE DATASHARE shared_sales_data
WITH (WRITE_ACCESS = TRUE);

-- aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
SELECT current_namespace;

GRANT USAGE ON DATASHARE shared_sales_data
TO NAMESPACE '12345678-1234-1234-1234-123456789012';

ALTER DATASHARE shared_sales_data
ADD SCHEMA sales;

ALTER DATASHARE shared_sales_data
ADD ALL TABLES IN SCHEMA sales;

ALTER DATASHARE shared_sales_data
SET INCLUDENEW = TRUE FOR SCHEMA sales;
```

---

# Setup — Consumer Cluster

```sql
CREATE DATABASE shared_sales_data
FROM DATASHARE shared_sales_data
OF NAMESPACE 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';

CREATE EXTERNAL SCHEMA sales
FROM REDSHIFT DATABASE 'shared_sales_data' SCHEMA shared_sales_data
```

---

# Setup - Additional Resources

- [Amazon Redshift Data Sharing: Setup Guide 2024](https://awsforengineers.com/blog/amazon-redshift-data-sharing-setup-guide-2024/)  
  Step-by-step walkthrough covering datashare creation, permissions, consumer setup, monitoring, and troubleshooting.

- [YouTube video walkthrough](https://www.youtube.com/watch?v=1DKulnmczW4)  
  Supplemental video resource for seeing the Redshift data sharing flow explained visually.
---
layout: center
class: text-center
---

# Why bother with Data Sharing?

### Isolation, burst compute, testing, and redundancy


---

# The Problem - One Cluster Rules Them All

```text
              ETL                 Audit
               \                    /
                \                  /
                 v                v
        +----------------------------------+
        |     Amazon Redshift Cluster      |
        |                                  |
        |   +---------+    +---------+     |
        |   | Leader  |----| Compute |     |
        |   |  Node   |    |  Nodes  |     |
        |   +---------+    +---------+     |
        |                                  |
        +----------------------------------+
                 ^                ^
                /                  \
               /                    \
        Marketing                Finance
```

- One shared Redshift cluster often becomes the center of the entire data platform
- Very different teams and workloads all converge on the same compute at the same time
- This is the contention problem the rest of the presentation will unpack

---

# Stepping Stone - Workload Management

<div style="display:flex; gap:24px; align-items:flex-start;">
<div style="flex:1;">

- Control concurrency
- Allocate memory
- Prioritize workloads
- Evicts lower priority queries when needed

## WLM Types

### Manual
- Static queues
- Fixed allocation

### Automatic
- ML-based
- Dynamic memory + slots

</div>
<div style="flex:1;">

```text
 ETL ----------- Priority 1 ---\
 Audit --------- Priority 1 ----\
 Marketing ----- Priority 2 -----+--> [ WLM ] --> +------------------+
 Finance ------- Priority 3 ----/               | Redshift Cluster |
                                                +------------------+
```

</div>
</div>

---



# Stepping Stone - Workload Management

<div style="display:flex; gap:24px; align-items:flex-start;">
<div style="flex:1;">

- You hit a classic **chicken and egg** problem:
  - ETL pipelines need steady throughput
  - Client-facing queries need low latency
- Both are business-critical, so contention becomes an architecture problem, not just a queueing problem

</div>
<div style="flex:1;">

```text
 ETL ----------- Priority 1 ---\
 Audit --------- Priority 1 ----\
 Marketing ----- Priority 2 -----+--> [ WLM ] --> +------------------+
 Finance ------- Priority 3 ----/               | Redshift Cluster |
                                                +------------------+
```

</div>
</div>

---

# Why One Cluster Stops Working

## Traffic patterns and isolation requirements keep growing:
  - ETL prococess
  - Marketing and business analytics
  - Audit and compliance workloads with especially high sensitivity

## Types of traffic
  - Continuous baseline traffic sits on provisioned hardware all day
  - But some workloads are bursty and should just finish as fast as possible, even if they only run once per day


---

# Why One Cluster Stops Working

> ### A single shared cluster forces these very different workloads to fight over the same compute

---

# The Goal - Isolated Clusters Per Workload

```text
   +------------------+    +------------------+
   | ETL Cluster      |    | Audit Cluster    |
   | Redshift         |    | Redshift         |
   +------------------+    +------------------+

   +------------------+    +------------------+
   | Marketing Cluster|    | Finance Cluster  |
   | Redshift         |    | Redshift         |
   +------------------+    +------------------+
```

- The target state is workload isolation instead of one shared compute plane
- ETL, Audit, Marketing, and Finance each get their own Redshift cluster

---

# More Than Just Performance

- Separate environments are often needed for testing:
  - pre-production validation
  - production-like testing
  - testing newer Redshift versions along the upgrade path

---

# More Than Just Performance

- Redundancy also matters:
  - avoid a single point of failure
  - reduce blast radius during service outages
  - protect against version-specific bugs

---

# More Than Just Performance

- Redshift Data Sharing wins:
  - Traffic isolation
  - Performance baseline per Redshift Cluster
  - Workloads can be isolated without duplicating storage
  - Internal billing
  - Redundancy
  - Decentralization

---
layout: center
class: text-center
---

# Redshift Architecture Crash Course

Pre-RA3 constraints, RA3 decoupling, and the path to Data Sharing

---

# Pre-RA3 Architecture

<div style="display:flex; gap:24px; align-items:flex-start;">
<div style="flex:1;">

```text
        +-----------------------------------+
        |         Single Cluster            |
        |                                   |
        |     +-----------------------+     |
        |     | Leader                |     |
        |     |   Node                |     |
        |     +-----------------------+     |
        |        |          |               |
        |        v          v               |
        |   +-----------+  +-----------+    |
        |   | Node 1    |  | Node 2    |    |
        |   |           |  |           |    |
        |   | Compute   |  | Compute   |    |
        |   | Storage   |  | Storage   |    |
        |   +-----------+  +-----------+    |
        +-----------------------------------+
```

</div>
<div style="flex:1;">

- Compute and storage tightly coupled
- Painful resizing causing downtimes (data shuffling between nodes)

</div>
</div>

---

# Pre-RA3 Resizing

<div style="display:flex; gap:24px; align-items:flex-start;">
<div style="flex:1.2;">

<svg viewBox="0 0 760 360" width="100%" xmlns="http://www.w3.org/2000/svg">
  <rect x="8" y="8" width="744" height="344" rx="20" fill="#f8fafc" stroke="#cbd5e1" stroke-width="2"/>

  <g>
    <rect x="28" y="34" width="190" height="180" rx="16" fill="#ffffff" stroke="#334155" stroke-width="3"/>
    <text x="123" y="62" text-anchor="middle" fill="#0f172a" style="font: 700 20px Arial, sans-serif;">2-Node Cluster</text>
    <rect x="48" y="84" width="150" height="44" rx="10" fill="#e2e8f0" stroke="#64748b" stroke-width="2"/>
    <rect x="48" y="140" width="150" height="44" rx="10" fill="#e2e8f0" stroke="#64748b" stroke-width="2"/>
    <text x="123" y="111" text-anchor="middle" fill="#0f172a" style="font: 600 16px Arial, sans-serif;">Node 1</text>
    <text x="123" y="167" text-anchor="middle" fill="#0f172a" style="font: 600 16px Arial, sans-serif;">Node 2</text>
  </g>

  <g>
    <rect x="542" y="34" width="190" height="236" rx="16" fill="#ffffff" stroke="#334155" stroke-width="3"/>
    <text x="637" y="62" text-anchor="middle" fill="#0f172a" style="font: 700 20px Arial, sans-serif;">3-Node Cluster</text>
    <rect x="562" y="84" width="150" height="44" rx="10" fill="#e2e8f0" stroke="#64748b" stroke-width="2"/>
    <rect x="562" y="140" width="150" height="44" rx="10" fill="#e2e8f0" stroke="#64748b" stroke-width="2"/>
    <rect x="562" y="196" width="150" height="44" rx="10" fill="#e2e8f0" stroke="#64748b" stroke-width="2"/>
    <text x="637" y="111" text-anchor="middle" fill="#0f172a" style="font: 600 16px Arial, sans-serif;">Node 1</text>
    <text x="637" y="167" text-anchor="middle" fill="#0f172a" style="font: 600 16px Arial, sans-serif;">Node 2</text>
    <text x="637" y="223" text-anchor="middle" fill="#0f172a" style="font: 600 16px Arial, sans-serif;">Node 3</text>
  </g>

  <text x="380" y="118" text-anchor="middle" fill="#b91c1c" style="font: 800 18px Arial, sans-serif;">DATA SHUFFLING</text>
  <line x1="224" y1="170" x2="536" y2="170" stroke="#fca5a5" stroke-width="8" stroke-linecap="round" opacity="0.55"/>

  <rect x="242" y="159" width="18" height="18" rx="4" fill="#ef4444">
    <animate attributeName="x" values="242;510" dur="2.2s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0;1;1;0" dur="2.2s" repeatCount="indefinite"/>
  </rect>
  <rect x="242" y="159" width="18" height="18" rx="4" fill="#ef4444">
    <animate attributeName="x" values="242;510" dur="2.2s" begin="0.7s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0;1;1;0" dur="2.2s" begin="0.7s" repeatCount="indefinite"/>
  </rect>
  <rect x="242" y="159" width="18" height="18" rx="4" fill="#ef4444">
    <animate attributeName="x" values="242;510" dur="2.2s" begin="1.4s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0;1;1;0" dur="2.2s" begin="1.4s" repeatCount="indefinite"/>
  </rect>

  <rect x="160" y="286" width="440" height="38" rx="19" fill="#fee2e2"/>
  <text x="380" y="311" text-anchor="middle" fill="#991b1b" style="font: 700 13px Arial, sans-serif;">
    Resize in progress: blocks redistributed across the new node layout
  </text>
</svg>

</div>
<div style="flex:0.8; text-align:left;">

- Pre-RA3 storage lived inside compute nodes
- Resizing meant redistributing data blocks across the new node layout
- This created long shuffle windows and operational disruption

</div>
</div>

---

# RA3 Architecture

<div style="display:flex; gap:24px; align-items:flex-start;">
<div style="flex:1;">

```text
  +------------------------------+
  |     RA3 Compute Cluster      |
  |                              |
  |  +---------+                 |
  |  | Leader  |                 |
  |  |  Node   |                 |
  |  +----+----+                 |
  |       |                      |
  |    +--+-------+              |
  |    |          |              |
  | +--v------+ +--v------+      |
  | | Node 1  | | Node 2  |      |
  | | Compute | | Compute |      |
  | +---------+ +---------+      |
  +-------------+----------------+
                |
                v
  +------------------------------+
  | Redshift Managed Storage     |
  | (S3 + cache + optimizations  |
  +------------------------------+
```

</div>
<div style="flex:1;">

- Compute and storage decoupled
- Independent scaling (elastic resize)

</div>
</div>

---

# One Data Layer, Multiple Compute

```
  +----------------------+   +----------------------+   +----------------------+
  | RA3 ETL Cluster      |   | RA3 BI Cluster       |   | Serverless for Audit |
  |                      |   |                      |   |                      |
  | +--------+           |   | +--------+           |   |  On-demand compute   |
  | | Leader |           |   | | Leader |           |   |  no fixed cluster    |
  | +---+----+           |   | +---+----+           |   |                      |
  |     |                |   |     |                |   |                      |
  |  +--+-----+          |   |  +--+-----+          |   |                      |
  |  | Compute |         |   |  | Compute |         |   |                      |
  |  +--------+          |   |  +--------+          |   |                      |
  +----------+-----------+   +----------+-----------+   +----------+-----------+
             |                            |                            |
             v                            v                            v
       +--------------------------------------------------------------------+
       | Redshift Managed Storage                                           |
       | (S3 + cache + optimizations)                                       |
       +--------------------------------------------------------------------+

```

- Same data
- No duplication
- Independent compute

---

# Redshift Offerings

## Provisioned
- Fixed clusters
- Predictable performance

## Serverless
- Auto scaling
- Bursty / unpredictable workloads
- High priority use cases

---

# Serverless Cost Model

- Billed per **RPU-seconds**, at least 60 :)
- ⚠️ Easy to overspend => Monitoring

## Serverless Scaling
- RPU ~ 1 vCPU, 8 GB Memory ???
- Base RPUs, Max RPUs
- [Intelligent Scaling in Amazon Redshift](https://www.amazon.science/publications/intelligent-scaling-in-amazon-redshift)
  - Comples queries cost less and finish faster on properly scaled RPU
  - The goal is to avoid disk spills
  - AI adjusts over ~1–2 weeks



---

# What Happens Under the Hood

- Metadata shared  
- Data stays in S3  
- Queries executed on consumer  

👉 Zero-copy architecture  

---

# Pattern: Hub & Spoke

```
        +----------------------+
        |   ETL (Producer)     |
        +----------+-----------+
                   |
                   v
        ========================
               Data Share
        ========================
          /        |         \
         v         v          v
     BI Team   Analytics   Reporting
```

- Producer cluster owns the data
- Exposes it through data shares

---

# Pattern: Mesh

```
   +---------+     +---------+
   | Dept A  |<--->| Dept B  |
   +----+----+     +----+----+
        |               |
        v               v
   +---------+     +---------+
   | Dept C  |<--->| Dept D  |
   +---------+     +---------+
```

- Clusters have their own data
- Act both as Producers and Consumers

---

# Pattern: Hybrid

- Mix of hub + mesh  

👉 Reality  

---

# Lesson learned

## Must match:
- Encryption settings
- Collation
- SUPER attribute settings
  - enable_case_sensitive_identifier
  - enable_case_sensitive_super_attribute

## Helper objects
- UDFs

## Schema naming across clusters
- sales.table1 on consumer
- sales.table1 on producer

---

# Compatibility Issues

- Version mismatches (too much)

👉 Keep clusters aligned  

---

# Monitoring

- Know your baseline traffic
- RPUs consumed (Serverless)
- Health checks
  - Test Redshift availability
  - Test Datashare availability from consumers
👉 Heartbeat  

---

# Operations

**There's no time for reverse engineering when shit hits the fan**

- Script everything  
- Document setup  
- Automate deployment  
- Runbooks
  - What to do when ...


---

# Final Thought

<div style="display:flex; justify-content:space-between; align-items:center; gap:32px;">
<div style="flex:1; text-align:left;">

**Stop fighting over one cluster.**

<br>

[hubacek.xyz](https://hubacek.xyz)

</div>
<div style="flex:0 0 220px; text-align:center;">

<img src="/hubacek-qr.svg" alt="QR code for hubacek.xyz" width="220" />

</div>
</div>
