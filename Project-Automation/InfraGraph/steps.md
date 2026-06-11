Great! Let me give you the **complete project plan** first, then we'll go step by step.

---

## My Recommendation for Your Setup

| Component | Recommendation | Reason |
|-----------|---------------|--------|
| **OS / Hosting** | **Docker on Windows Server** | Easy, portable, no OS conflicts |
| **PostgreSQL** | Docker Container | No manual install needed |
| **Grafana** | Docker Container | No manual install needed |
| **ETL** | **PowerShell** | You are on Windows, enterprise friendly |
| **Scheduler** | Windows Task Scheduler | Built-in, no extra tool |

---

## Final Architecture

```
vROPs API ──────┐
                ├──► PowerShell ETL ──► PostgreSQL (Docker) ──► Grafana (Docker)
ServiceNow API ─┘        │
                   Task Scheduler
                   (runs every X min)
```

---

## Complete Project Plan — 6 Phases

```
Phase 1 → Install Docker on Windows Server
Phase 2 → Deploy PostgreSQL in Docker
Phase 3 → Deploy Grafana in Docker
Phase 4 → Create Database Schema (Tables)
Phase 5 → Build PowerShell ETL Script
Phase 6 → Connect Grafana to PostgreSQL & Build Dashboard
```

---

---

# PHASE 1 — Install Docker on Windows Server

### Step 1.1 — Check Windows Server Version
```powershell
# Run in PowerShell as Administrator
winver
# Need Windows Server 2016 or higher
```

### Step 1.2 — Install Docker Desktop
```powershell
# Download Docker Desktop installer
# Go to: https://www.docker.com/products/docker-desktop

# OR use winget command
winget install Docker.DockerDesktop
```

### Step 1.3 — Verify Docker Installation
```powershell
docker --version
# Expected output: Docker version 24.x.x

docker ps
# Expected output: Empty container list (no error)
```

### Step 1.4 — Create Project Folder Structure
```powershell
# Create your project directory
mkdir C:\vrops-snow-project
mkdir C:\vrops-snow-project\scripts
mkdir C:\vrops-snow-project\logs
mkdir C:\vrops-snow-project\docker

Write-Host "Project folders created!"
```

---

---

# PHASE 2 — Deploy PostgreSQL in Docker

### Step 2.1 — Create Docker Compose File
```powershell
# Navigate to docker folder
cd C:\vrops-snow-project\docker
```

Create a file called `docker-compose.yml`:
```yaml
version: '3.8'

services:

  postgres:
    image: postgres:15
    container_name: vrops-snow-postgres
    restart: always
    environment:
      POSTGRES_DB: vrops_snow_db
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: Admin@1234
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  grafana:
    image: grafana/grafana:latest
    container_name: vrops-snow-grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: Admin@1234
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - postgres

volumes:
  postgres_data:
  grafana_data:
```

### Step 2.2 — Start Both Containers
```powershell
cd C:\vrops-snow-project\docker

# Start PostgreSQL & Grafana together
docker-compose up -d

# Verify both running
docker ps
```

### Expected Output:
```
CONTAINER ID   IMAGE             STATUS         PORTS
abc123         postgres:15       Up 2 minutes   0.0.0.0:5432->5432/tcp
def456         grafana/grafana   Up 2 minutes   0.0.0.0:3000->3000/tcp
```

---

---

# PHASE 3 — Create Database Schema

### Step 3.1 — Connect to PostgreSQL Container
```powershell
# Open PostgreSQL shell inside container
docker exec -it vrops-snow-postgres psql -U admin -d vrops_snow_db
```

### Step 3.2 — Create Tables
```sql
-- vROPs Inventory Table
CREATE TABLE vrops_inventory (
    id                SERIAL PRIMARY KEY,
    resource_id       VARCHAR(100) UNIQUE,
    resource_name     VARCHAR(200),
    resource_type     VARCHAR(100),
    health_status     VARCHAR(50),
    alert_status      VARCHAR(50),
    cpu_usage         NUMERIC(5,2),
    memory_usage      NUMERIC(5,2),
    collected_at      TIMESTAMP DEFAULT NOW()
);

-- ServiceNow Incidents Table
CREATE TABLE snow_incidents (
    id                SERIAL PRIMARY KEY,
    incident_number   VARCHAR(50) UNIQUE,
    affected_ci       VARCHAR(200),
    priority          VARCHAR(20),
    state             VARCHAR(50),
    category          VARCHAR(100),
    assigned_to       VARCHAR(100),
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP,
    collected_at      TIMESTAMP DEFAULT NOW()
);

-- Unified View (Joining both tables)
CREATE VIEW unified_view AS
SELECT
    v.resource_name,
    v.resource_type,
    v.health_status,
    v.cpu_usage,
    v.memory_usage,
    s.incident_number,
    s.priority        AS incident_priority,
    s.state           AS incident_state,
    s.created_at      AS incident_created
FROM vrops_inventory v
LEFT JOIN snow_incidents s
    ON LOWER(v.resource_name) = LOWER(s.affected_ci);

-- Verify tables created
\dt
```

### Step 3.3 — Exit PostgreSQL Shell
```sql
\q
```

---

---

# PHASE 4 — PowerShell ETL Script

### Step 4.1 — Download Npgsql DLL
```powershell
# Install NuGet & download Npgsql
Install-PackageProvider -Name NuGet -Force
Install-Package Npgsql -Destination C:\vrops-snow-project\lib -Force
```

### Step 4.2 — Create ETL Script
Save this as `C:\vrops-snow-project\scripts\etl_main.ps1`:

```powershell
# ============================================
# ETL Script — vROPs & ServiceNow to PostgreSQL
# ============================================

# Load Npgsql
Add-Type -Path "C:\vrops-snow-project\lib\Npgsql.dll"

# ── CONFIG ───────────────────────────────────
$vropsHost    = "https://your-vrops-host"
$vropsUser    = "your_service_account"
$vropsPass    = "your_password"

$snowHost     = "https://your-instance.service-now.com"
$snowUser     = "your_snow_user"
$snowPass     = "your_snow_password"

$pgConn       = "Host=localhost;Port=5432;Database=vrops_snow_db;Username=admin;Password=Admin@1234"

$logFile      = "C:\vrops-snow-project\logs\etl_log.txt"

# ── LOGGING ──────────────────────────────────
Function Write-Log($msg) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - $msg" | Tee-Object -FilePath $logFile -Append
}

# ── CONNECT TO POSTGRES ───────────────────────
$conn = New-Object Npgsql.NpgsqlConnection($pgConn)
$conn.Open()
Write-Log "PostgreSQL Connected"

# ════════════════════════════════════════════
# PULL FROM vROPs
# ════════════════════════════════════════════
Write-Log "Starting vROPs data pull..."

$authBody = @{
    username   = $vropsUser
    password   = $vropsPass
    authSource = "LOCAL"
} | ConvertTo-Json

$tokenResp = Invoke-RestMethod `
    -Uri "$vropsHost/suite-api/api/auth/token/acquire" `
    -Method POST `
    -ContentType "application/json" `
    -Body $authBody

$token = $tokenResp.token
Write-Log "vROPs Token acquired"

$headers = @{
    Authorization = "vRealizeOpsToken $token"
    Accept        = "application/json"
}

$inventory = Invoke-RestMethod `
    -Uri "$vropsHost/suite-api/api/resources" `
    -Method GET `
    -Headers $headers

Write-Log "vROPs resources pulled: $($inventory.resourceList.Count)"

# PUSH vROPs to PostgreSQL
foreach ($resource in $inventory.resourceList) {
    $query = @"
        INSERT INTO vrops_inventory
            (resource_id, resource_name, resource_type, health_status, collected_at)
        VALUES
            (@rid, @rname, @rtype, @rstatus, @rtime)
        ON CONFLICT (resource_id) DO UPDATE
        SET health_status = EXCLUDED.health_status,
            collected_at  = EXCLUDED.collected_at;
"@
    $cmd = New-Object Npgsql.NpgsqlCommand($query, $conn)
    $cmd.Parameters.AddWithValue("rid",     $resource.identifier)                      | Out-Null
    $cmd.Parameters.AddWithValue("rname",   $resource.resourceKey.name)                | Out-Null
    $cmd.Parameters.AddWithValue("rtype",   $resource.resourceKey.resourceKindKey)     | Out-Null
    $cmd.Parameters.AddWithValue("rstatus", $resource.resourceStatusStates.resourceState) | Out-Null
    $cmd.Parameters.AddWithValue("rtime",   (Get-Date))                                | Out-Null
    $cmd.ExecuteNonQuery() | Out-Null
}
Write-Log "vROPs data pushed to PostgreSQL"

# ════════════════════════════════════════════
# PULL FROM ServiceNow
# ════════════════════════════════════════════
Write-Log "Starting ServiceNow data pull..."

$snowHeaders = @{
    Authorization = "Basic " + [Convert]::ToBase64String(
        [Text.Encoding]::ASCII.GetBytes("${snowUser}:${snowPass}"))
    Accept        = "application/json"
}

$incidents = Invoke-RestMethod `
    -Uri "$snowHost/api/now/table/incident?sysparm_limit=1000" `
    -Method GET `
    -Headers $snowHeaders

Write-Log "ServiceNow incidents pulled: $($incidents.result.Count)"

# PUSH ServiceNow to PostgreSQL
foreach ($inc in $incidents.result) {
    $query = @"
        INSERT INTO snow_incidents
            (incident_number, affected_ci, priority, state, created_at, collected_at)
        VALUES
            (@inum, @aci, @pri, @state, @cat, @ctime)
        ON CONFLICT (incident_number) DO UPDATE
        SET state        = EXCLUDED.state,
            priority     = EXCLUDED.priority,
            collected_at = EXCLUDED.collected_at;
"@
    $cmd = New-Object Npgsql.NpgsqlCommand($query, $conn)
    $cmd.Parameters.AddWithValue("inum",  $inc.number)                    | Out-Null
    $cmd.Parameters.AddWithValue("aci",   $inc.cmdb_ci.display_value)     | Out-Null
    $cmd.Parameters.AddWithValue("pri",   $inc.priority)                  | Out-Null
    $cmd.Parameters.AddWithValue("state", $inc.state)                     | Out-Null
    $cmd.Parameters.AddWithValue("cat",   $inc.sys_created_on)            | Out-Null
    $cmd.Parameters.AddWithValue("ctime", (Get-Date))                     | Out-Null
    $cmd.ExecuteNonQuery() | Out-Null
}
Write-Log "ServiceNow data pushed to PostgreSQL"

# ── CLOSE CONNECTION ──────────────────────────
$conn.Close()
Write-Log "ETL Completed Successfully & Connection Closed"
```

---

---

# PHASE 5 — Schedule ETL via Windows Task Scheduler

### Step 5.1 — Create Scheduled Task
```powershell
# Run as Administrator
$action  = New-ScheduledTaskAction `
    -Execute "PowerShell.exe" `
    -Argument "-ExecutionPolicy Bypass -File C:\vrops-snow-project\scripts\etl_main.ps1"

$trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 15) -Once -At (Get-Date)

$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Minutes 10)

Register-ScheduledTask `
    -TaskName "vROPs-Snow-ETL" `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -RunLevel Highest `
    -Force

Write-Host "Scheduled Task created — runs every 15 minutes!"
```

---

---

# PHASE 6 — Connect Grafana & Build Dashboard

### Step 6.1 — Open Grafana
```
Browser → http://localhost:3000
Login   → admin / Admin@1234
```

### Step 6.2 — Add PostgreSQL Data Source
```
1. Go to → Connections → Data Sources
2. Click → Add Data Source
3. Select → PostgreSQL
4. Fill in:
   Host     : localhost:5432
   Database : vrops_snow_db
   User     : admin
   Password : Admin@1234
5. Click → Save & Test
```

### Step 6.3 — Create Dashboard Panels

**Panel 1 — vROPs Health Status**
```sql
SELECT resource_name, health_status, cpu_usage, memory_usage, collected_at
FROM vrops_inventory
ORDER BY collected_at DESC
```

**Panel 2 — ServiceNow Open Incidents**
```sql
SELECT incident_number, affected_ci, priority, state, created_at
FROM snow_incidents
WHERE state = 'Open'
ORDER BY created_at DESC
```

**Panel 3 — Unified View (Correlated)**
```sql
SELECT * FROM unified_view
ORDER BY incident_created DESC
```

---

---

## Complete Summary Checklist

```
✅ Phase 1 — Docker installed on Windows Server
✅ Phase 2 — PostgreSQL & Grafana running in Docker
✅ Phase 3 — Database tables & unified view created
✅ Phase 4 — PowerShell ETL script pulling & pushing data
✅ Phase 5 — Task Scheduler running ETL every 15 min
✅ Phase 6 — Grafana connected & dashboards built
```

---

> **We will go phase by phase together.** Start with **Phase 1 — Docker installation** and let me know once Docker is installed & running. I'll guide you through each next step!
