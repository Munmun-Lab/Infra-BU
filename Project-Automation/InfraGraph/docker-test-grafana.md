This is actually a very good approach for a POC.

You **don't need Ubuntu VM inside Docker**. That's an extra layer and makes troubleshooting harder.

Instead use:

```text
Windows Jump Server
        |
    Docker Desktop
        |
   ------------------
   |                |
PostgreSQL      Grafana
Container       Container
```

Much simpler.

---

# Architecture

```text
                 vROps
                   |
                   |
            PowerShell Script
                   |
                   v
              PostgreSQL
                   |
                   v
               Grafana
                   |
              Browser Users
```

Later:

```text
ServiceNow
    |
    +---- PowerShell Script
    |
    v
PostgreSQL
```

---

# Step 1 - Install Docker Desktop

On Windows Server (if supported) or Windows Jump Host.

Download:

[Docker Desktop](https://www.docker.com/products/docker-desktop/?utm_source=chatgpt.com)

Verify:

```powershell
docker version
```

---

# Step 2 - Create Project Folder

```powershell
mkdir C:\InfraProject
cd C:\InfraProject
```

Create:

```text
C:\InfraProject
│
├── docker-compose.yml
├── scripts
├── data
└── exports
```

---

# Step 3 - Create Docker Compose

Create:

```yaml
version: '3'

services:

  postgres:
    image: postgres:16
    container_name: postgres
    environment:
      POSTGRES_USER: infraadmin
      POSTGRES_PASSWORD: Password123
      POSTGRES_DB: infra_mapping
    ports:
      - "5432:5432"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    depends_on:
      - postgres
```

Save as:

```text
docker-compose.yml
```

---

# Step 4 - Start Containers

```powershell
docker compose up -d
```

Check:

```powershell
docker ps
```

Expected:

```text
postgres
grafana
```

running.

---

# Step 5 - Access Grafana

Open:

```text
http://jumpserver:3000
```

or

```text
http://localhost:3000
```

Login:

```text
admin
admin
```

Grafana will ask for password change.

---

# Step 6 - Access PostgreSQL

Connect into container:

```powershell
docker exec -it postgres psql -U infraadmin
```

Switch database:

```sql
\c infra_mapping
```

Create tables:

```sql
CREATE TABLE vm_inventory (
vm_name varchar(255),
host_name varchar(255),
cluster_name varchar(255),
datastore varchar(255)
);
```

```sql
CREATE TABLE app_mapping (
vm_name varchar(255),
application_name varchar(255),
owner varchar(255)
);
```

Verify:

```sql
\dt
```

---

# Step 7 - Load Sample Data

Insert test data.

```sql
INSERT INTO vm_inventory VALUES
('SAP-APP01','ESXi05','PROD','DS01');
```

```sql
INSERT INTO app_mapping VALUES
('SAP-APP01','SAP','Finance');
```

---

# Step 8 - Create Merged View

```sql
CREATE VIEW infra_mapping AS
SELECT
v.vm_name,
v.host_name,
v.cluster_name,
v.datastore,
a.application_name,
a.owner
FROM vm_inventory v
LEFT JOIN app_mapping a
ON v.vm_name=a.vm_name;
```

Test:

```sql
SELECT * FROM infra_mapping;
```

Result:

```text
SAP-APP01
ESXi05
DS01
SAP
Finance
```

Now the merge is working.

---

# Step 9 - Connect Grafana to PostgreSQL

Grafana:

```text
Connections
   ↓
Add New Data Source
   ↓
PostgreSQL
```

Use:

```text
Host:
postgres:5432
```

If Grafana is running inside Docker.

Database:

```text
infra_mapping
```

User:

```text
infraadmin
```

Password:

```text
Password123
```

Save & Test.

---

# Step 10 - Create First Dashboard

Panel Query:

```sql
SELECT *
FROM infra_mapping;
```

Dashboard displays:

| VM        | Host   | Application | Owner   |
| --------- | ------ | ----------- | ------- |
| SAP-APP01 | ESXi05 | SAP         | Finance |

Congratulations.

You now have:

```text
ServiceNow Data
        +
VMware Data
        =
Unified View
```

---

# Step 11 - Integrate vROps

For POC:

Export CSV from vROps.

Example:

```csv
vm_name,host_name,cluster_name,datastore
SAP-APP01,ESXi05,PROD,DS01
```

Copy into:

```text
C:\InfraProject\exports
```

PowerShell:

```powershell
Import-Csv vm_inventory.csv
```

Insert into PostgreSQL.

Initially do manual imports.

Later automate.

---

# Step 12 - Integrate ServiceNow

Export:

```csv
vm_name,application_name,owner
SAP-APP01,SAP,Finance
```

Import into:

```sql
app_mapping
```

table.

---

# Step 13 - Impact Analysis Dashboard

Create dropdown variable:

```sql
SELECT DISTINCT host_name
FROM infra_mapping
```

Choose:

```text
ESXi05
```

Panel:

```sql
SELECT *
FROM infra_mapping
WHERE host_name='$host';
```

Output:

```text
Host:
ESXi05

Affected VM:
SAP-APP01

Application:
SAP

Owner:
Finance
```

---

## What I would do first

Don't touch APIs yet.

### Week 1 POC

1. Docker Desktop
2. PostgreSQL Container
3. Grafana Container
4. Create tables manually
5. Import 10 VM records manually
6. Import 10 ServiceNow mappings manually
7. Create merged view
8. Create Grafana dashboard

Once that works, then move to:

```text
vROps API
      ↓
PostgreSQL
      ↓
Grafana

ServiceNow API
      ↓
PostgreSQL
      ↓
Grafana
```

This gives you a working demo within a few days and proves the concept before investing time in automation.
