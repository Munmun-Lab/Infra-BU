This is the architecture question you need to solve.

Grafana **does not merge data well by itself** for this type of dependency mapping. You need an intermediate layer.

# Option 1 (Recommended): PostgreSQL as the Central Repository

```text
vROps API
    |
    |
    +---- Inventory Export
    |
PostgreSQL
    |
ServiceNow API
    |
    +---- CMDB Export
    |
Grafana
```

---

## Step 1: Export from vROps

Daily or hourly pull:

| VM        | Host   | Datastore | Cluster |
| --------- | ------ | --------- | ------- |
| SAP-APP01 | ESXi05 | DS01      | PROD    |
| SAP-DB01  | ESXi06 | DS02      | PROD    |

Store into table:

```sql
vm_inventory
```

---

## Step 2: Export from ServiceNow

Pull:

| VM        | Application | Owner   |
| --------- | ----------- | ------- |
| SAP-APP01 | SAP         | Finance |
| SAP-DB01  | SAP         | Finance |

Store into table:

```sql
application_mapping
```

---

## Step 3: Create a View

This is where the merge happens.

```sql
CREATE VIEW infra_mapping AS
SELECT
    v.vm_name,
    v.host_name,
    v.datastore,
    a.application,
    a.owner
FROM vm_inventory v
LEFT JOIN application_mapping a
ON v.vm_name = a.vm_name;
```

Result:

| VM        | Host   | Datastore | Application | Owner   |
| --------- | ------ | --------- | ----------- | ------- |
| SAP-APP01 | ESXi05 | DS01      | SAP         | Finance |
| SAP-DB01  | ESXi06 | DS02      | SAP         | Finance |

---

## Step 4: Grafana Reads Only the Final View

Grafana datasource:

```text
PostgreSQL
```

Query:

```sql
SELECT *
FROM infra_mapping
```

Dashboard:

```text
Application: SAP

VMs:
 SAP-APP01
 SAP-DB01

Hosts:
 ESXi05
 ESXi06

Owner:
 Finance Team
```

---

# Impact Analysis Example

Host failure:

```text
ESXi05
```

Grafana query:

```sql
SELECT *
FROM infra_mapping
WHERE host_name='ESXi05';
```

Result:

| Application | VM        | Owner   |
| ----------- | --------- | ------- |
| SAP         | SAP-APP01 | Finance |

Manager immediately sees:

```text
Affected Application:
SAP

Affected VM:
SAP-APP01

Owner:
Finance Team
```

---

# Option 2: Elasticsearch

If your organization already has EFK/ELK.

```text
vROps
   |
   +---- Logstash
                |
ServiceNow -----+
                |
         Elasticsearch
                |
             Grafana
```

The merge happens inside:

* Logstash
* Ingest Pipeline

But this is more complex.

---

# Option 3: CSV Proof of Concept

For an MVP:

```text
vROps Export CSV
+
ServiceNow Export CSV
```

Merge using:

* Python script
* PowerShell script

Output:

```text
infra_mapping.csv
```

Grafana can read it through:

* PostgreSQL
* MySQL
* CSV plugin

This is often how successful POCs start.

---

# What I Would Do as a VMware/Infrastructure Team

Since your team knows automation but not software development:

### Build

```text
vROps API
      |
PowerShell
      |
      +---- PostgreSQL
      |
ServiceNow API
      |
Grafana
```

### Why?

* Easy to learn
* Free/Open Source
* Grafana supports PostgreSQL natively
* Simple SQL joins
* No Neo4j knowledge required
* Easy Jira integration later

Think of PostgreSQL as your **single source of truth** where VMware data and ServiceNow data meet. Once the merged table exists, Grafana becomes very easy because it only needs to visualize the already-joined data.
