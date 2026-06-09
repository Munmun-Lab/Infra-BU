Perfect. If you're starting from scratch, don't think about Grafana, Jira, AI, or impact analysis yet.

Your first milestone is much simpler:

> **Can I create one merged table that shows Application → VM → ESXi → Datastore → Owner?**

If you achieve that, you've completed the foundation of the entire project.

---

# Phase 1 — Discovery & Data Assessment

## Objective

Understand what data already exists.

### Step 1.1: Meet vROps Team

Ask:

```text
Can we access vROps API?

Can we export:
- VM Name
- ESXi Host
- Cluster
- Datastore
- Network
- Power State
```

### Step 1.2: Meet ServiceNow Team

Ask:

```text
Does CMDB contain VMware VMs?

Do Application Services have relationships to VMs?

Can we access ServiceNow REST API?
```

### Step 1.3: Verify Join Key

Most important check:

```text
vROps VM Name

SAP-APP01
```

and

```text
ServiceNow CI Name

SAP-APP01
```

If identical:

```text
JOIN KEY = VM NAME
```

Without this, the project becomes harder.

---

# Phase 2 — Build Lab Environment

Use a VM.

Recommended:

```text
Ubuntu 24.04
4 vCPU
8 GB RAM
100 GB Disk
```

Install:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Verify:

```bash
sudo systemctl status postgresql
```

Expected:

```text
active (running)
```

---

# Phase 3 — Create Central Database

Login:

```bash
sudo -u postgres psql
```

Create database:

```sql
CREATE DATABASE infra_mapping;
```

Create user:

```sql
CREATE USER infraadmin WITH PASSWORD 'StrongPassword';
```

Grant rights:

```sql
GRANT ALL PRIVILEGES ON DATABASE infra_mapping TO infraadmin;
```

Exit:

```sql
\q
```

---

# Phase 4 — Create Tables

Connect:

```bash
psql -U infraadmin -d infra_mapping -h localhost
```

Create VMware inventory table:

```sql
CREATE TABLE vm_inventory (
    vm_name VARCHAR(255),
    host_name VARCHAR(255),
    cluster_name VARCHAR(255),
    datastore VARCHAR(255),
    network VARCHAR(255),
    power_state VARCHAR(50)
);
```

Create ServiceNow mapping table:

```sql
CREATE TABLE app_mapping (
    vm_name VARCHAR(255),
    application_name VARCHAR(255),
    owner VARCHAR(255),
    support_group VARCHAR(255)
);
```

---

# Phase 5 — Export Data from vROps

Since every vROps environment is slightly different, start manually.

Export CSV from vROps.

Example:

```csv
vm_name,host_name,cluster_name,datastore
SAP-APP01,ESXi05,PROD,DS01
SAP-DB01,ESXi06,PROD,DS02
```

Save:

```text
vm_inventory.csv
```

---

# Phase 6 — Export Data from ServiceNow

Export:

```csv
vm_name,application_name,owner
SAP-APP01,SAP,Finance
SAP-DB01,SAP,Finance
```

Save:

```text
app_mapping.csv
```

---

# Phase 7 — Import Data

Import VMware data:

```sql
COPY vm_inventory
FROM '/tmp/vm_inventory.csv'
DELIMITER ','
CSV HEADER;
```

Import ServiceNow data:

```sql
COPY app_mapping
FROM '/tmp/app_mapping.csv'
DELIMITER ','
CSV HEADER;
```

Verify:

```sql
SELECT * FROM vm_inventory;
```

```sql
SELECT * FROM app_mapping;
```

---

# Phase 8 — Merge Data

Create a view:

```sql
CREATE VIEW infra_mapping AS
SELECT
    v.vm_name,
    v.host_name,
    v.cluster_name,
    v.datastore,
    v.network,
    a.application_name,
    a.owner,
    a.support_group
FROM vm_inventory v
LEFT JOIN app_mapping a
ON v.vm_name = a.vm_name;
```

Test:

```sql
SELECT * FROM infra_mapping;
```

Output:

| VM        | Host   | Datastore | Application | Owner   |
| --------- | ------ | --------- | ----------- | ------- |
| SAP-APP01 | ESXi05 | DS01      | SAP         | Finance |

At this point you've completed the most important technical milestone.

---

# Phase 9 — Install Grafana

On Ubuntu:

```bash
sudo apt-get install -y adduser libfontconfig1 musl
```

Download latest Grafana package.

Or use Docker:

```bash
docker run -d \
--name=grafana \
-p 3000:3000 \
grafana/grafana
```

Access:

```text
http://server-ip:3000
```

Login:

```text
admin/admin
```

---

# Phase 10 — Connect PostgreSQL

Grafana:

```text
Connections
   →
Add Data Source
   →
PostgreSQL
```

Provide:

```text
Host: localhost:5432

Database: infra_mapping

User: infraadmin

Password: ********
```

Save & Test.

---

# Phase 11 — First Dashboard

Create table panel.

Query:

```sql
SELECT *
FROM infra_mapping;
```

You now have:

```text
Application
VM
Host
Datastore
Owner
```

on one screen.

---

# Phase 12 — Impact Analysis Dashboard

Create variable:

```text
host_name
```

Query:

```sql
SELECT DISTINCT host_name
FROM infra_mapping
ORDER BY host_name;
```

Dashboard dropdown:

```text
ESXi05
ESXi06
```

Panel query:

```sql
SELECT *
FROM infra_mapping
WHERE host_name='$host_name';
```

Result:

```text
ESXi05

Affected VMs:
SAP-APP01

Affected Applications:
SAP

Owner:
Finance
```

Congratulations.

You now have working impact analysis.

---

# Phase 13 — Automate Data Collection

Only after everything works manually.

Build:

```text
vROps API
      |
      |
PowerShell
      |
PostgreSQL
      |
Grafana
```

and

```text
ServiceNow API
      |
PowerShell
      |
PostgreSQL
```

Schedule:

```bash
crontab -e
```

Example:

```bash
0 2 * * * /scripts/vrops_sync.sh
15 2 * * * /scripts/servicenow_sync.sh
```

---

# What I Would Do This Week

### Day 1

* Verify ServiceNow VM names
* Verify vROps export capability

### Day 2

* Install PostgreSQL
* Create tables

### Day 3

* Export sample CSVs
* Import data

### Day 4

* Create merged view
* Validate relationships

### Day 5

* Install Grafana
* Build first dashboard

Do **not** start with APIs, Jira, AI, Terraform, or automation. First prove that you can manually produce one merged table showing:

```text
Application
     |
VM
     |
ESXi
     |
Datastore
     |
Owner
```

Once that works, the rest of the project becomes incremental rather than overwhelming.
