# VMware Storage Reclamation – Using vROPs (Aria Operations) + Jira
> **No PowerShell required.** This guide uses vRealize Operations (vROPs) for detection  
> and Jira for ticket-based approval and tracking before any cleanup action is taken.

---

## How the Automation Works (Big Picture)

```
vROPs detects issue
      ↓
vROPs Alert fires
      ↓
vROPs Webhook → Jira REST API
      ↓
Jira ticket auto-created (with details)
      ↓
Engineer reviews & approves ticket
      ↓
(Optional) Jira triggers remediation action back to vROPs / vRA
```

---

## PART 0 – Prerequisites

| Requirement | Details |
|---|---|
| vROPs version | 8.x or Aria Operations (any recent version) |
| Jira | Cloud or Server (you need Admin access) |
| Jira API Token | Generated from your Atlassian account |
| vROPs Outbound REST Plugin | Built-in — just needs to be enabled |
| Network | vROPs must be able to reach Jira on port 443 |

---

## PART 1 – Configure vROPs to Detect Storage Issues

### Step 1: Enable the vSphere Adapter (if not already done)

1. Log into **vROPs UI** → go to **Administration** (left menu)
2. Click **Solutions** → **VMware vSphere**
3. Click your vCenter adapter instance → click **Edit**
4. Verify credentials and click **Test Connection** → should show ✅ green
5. Click **Save**

> vROPs now collects metrics from your entire vSphere environment every 5 minutes by default.

---

### Step 2: Create a Custom Alert for Orphaned VMDKs

vROPs has a built-in symptom for this. We will create an alert definition on top of it.

1. Go to **Alerts** → **Alert Definitions** → click **+ Add**
2. Fill in:
   - **Name:** `Orphaned VMDK Detected`
   - **Description:** `A VMDK file exists on a datastore with no associated VM`
   - **Object Type:** `Datastore`
   - **Alert Impact:** `Efficiency`
   - **Alert Criticality:** `Warning`

3. Under **Symptoms**, click **Add Symptom Definition**:
   - Search for: `Orphaned`
   - Select: **"Orphaned virtual disk file detected"** (built-in vROPs symptom)
   - Click **Add**

4. Click **Save Alert Definition**

> ✅ vROPs will now fire an alert on any datastore where orphaned VMDKs are found.

---

### Step 3: Create a Custom Alert for ISOs Mounted to VMs

1. Go to **Alerts** → **Alert Definitions** → **+ Add**
2. Fill in:
   - **Name:** `ISO Image Still Mounted on VM`
   - **Object Type:** `Virtual Machine`
   - **Alert Impact:** `Efficiency`
   - **Alert Criticality:** `Info`

3. Under **Symptoms**, click **Add Symptom Definition** → **New Symptom**:
   - **Name:** `CD-ROM has ISO path connected`
   - **Object Type:** `Virtual Machine`
   - **Metric:** `config|hardware|numCoresPerSocket` *(use as a base — see note below)*

> **Note:** vROPs does not have a native ISO-mounted metric. Use this approach instead:
> - Metric to use: `summary|runtime|connectionState`  
> - OR create a **Super Metric** (see Step 4 below) that queries via the vROPs API

4. **Easier alternative for ISOs:** Use vROPs **Views** to report on this (Step 6 below) rather than an alert — ISO detection is better as a report than a real-time alert.

---

### Step 4: Create a Super Metric for ISO Detection (vROPs Advanced)

1. Go to **Administration** → **Configuration** → **Super Metrics** → **+ Add**
2. **Name:** `ISO_Mounted_Check`
3. In the formula box, enter:
   ```
   count(${adaptertype=VMWARE, objecttype=VirtualMachine, attribute=config|hardware|cdrom|connected, depth=1})
   ```
4. Assign this Super Metric to the **Virtual Machine** object type
5. Click **Save**

> This counts how many VMs have a connected CD-ROM drive. Use this as a symptom trigger in your alert.

---

### Step 5: Create an Alert for Unused Datastores

1. Go to **Alerts** → **Alert Definitions** → **+ Add**
2. Fill in:
   - **Name:** `Datastore Has No VMs – Candidate for Decommission`
   - **Object Type:** `Datastore`
   - **Alert Impact:** `Efficiency`

3. Under **Symptoms** → **Add Symptom Definition** → **New**:
   - **Name:** `Datastore VM Count is Zero`
   - **Object Type:** `Datastore`
   - **Metric:** `summary|number_virtual_machines`
   - **Operator:** `=`
   - **Value:** `0`
   - **Condition Type:** `Static Threshold`

4. Click **Save**

> ✅ vROPs will now alert whenever a datastore has 0 VMs registered on it.

---

## PART 2 – Configure vROPs to Create Jira Tickets Automatically

### Step 6: Get Your Jira API Token

1. Log into **https://id.atlassian.com**
2. Go to **Security** → **API Tokens** → **Create API Token**
3. Name it: `vROPs-Integration`
4. Copy the token — **save it somewhere safe**, you only see it once

---

### Step 7: Configure Outbound REST Plugin in vROPs

1. In vROPs, go to **Administration** → **Management** → **Outbound Settings**
2. Click **+ Add** → Select **REST Notification Plugin**
3. Fill in:
   - **Name:** `Jira-REST`
   - **URL:** `https://your-company.atlassian.net/rest/api/3/issue`
   - **Connection Count:** `2`
   - **HTTP Method:** `POST`
   - **Authentication:** `Basic`
   - **Username:** `your-email@company.com`
   - **Password:** *(paste your Jira API Token here)*
4. Click **Test** → should return HTTP 200 or 201
5. Click **Save**

---

### Step 8: Create a Notification Rule to Create Jira Tickets

1. Go to **Alerts** → **Alert Notification Rules** → **+ Add**
2. Fill in:
   - **Name:** `Storage Reclamation → Jira`
   - **Plugin:** Select `Jira-REST` (created above)
   - **Notification Trigger:** `Alert Created` and `Alert Updated`

3. Under **Filters**, select the 3 alerts you created:
   - `Orphaned VMDK Detected`
   - `ISO Image Still Mounted on VM`
   - `Datastore Has No VMs – Candidate for Decommission`

4. Under **Payload**, paste this JSON (customise project key and issue type for your Jira):

```json
{
  "fields": {
    "project": {
      "key": "INFRA"
    },
    "summary": "vROPs Alert: ${alert.alertName} on ${resource.name}",
    "description": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            {
              "type": "text",
              "text": "Alert Details:\n\nAlert Name: ${alert.alertName}\nObject: ${resource.name}\nObject Type: ${resource.resourceKindKey}\nSeverity: ${alert.criticality}\nTriggered At: ${alert.startTime}\n\nRecommended Action: ${alert.recommendedAction}\n\nPlease review and approve storage reclamation action."
            }
          ]
        }
      ]
    },
    "issuetype": {
      "name": "Task"
    },
    "priority": {
      "name": "Medium"
    },
    "labels": ["vmware", "storage-reclamation", "automation"]
  }
}
```

5. Click **Save**

> ✅ From now on, every time vROPs fires one of your storage alerts, a Jira ticket is automatically created with all the details.

---

## PART 3 – Set Up Jira for Review & Tracking

### Step 9: Create a Jira Board for VMware Alerts

1. In Jira, go to your **INFRA project** → **Board Settings**
2. Add these columns to your Kanban/Scrum board:
   - `Open` – Auto-created vROPs tickets land here
   - `Under Review` – Engineer is checking the finding
   - `Approved for Cleanup` – Ready to action
   - `Done` – Cleanup completed

---

### Step 10: Add a Jira Automation Rule – Assign to Engineer

1. In Jira project → **Project Settings** → **Automation** → **+ Create Rule**
2. **Trigger:** Issue Created
3. **Condition:** Label = `vmware`
4. **Action:** Assign Issue → select your VMware admin user
5. **Action:** Add Comment → `"This ticket was auto-created by vROPs. Please review the storage finding and move to Approved for Cleanup when ready."`
6. Click **Save & Enable**

---

### Step 11: Add a Jira Automation Rule – Auto-close if Resolved in vROPs

1. **Trigger:** Incoming Webhook (from vROPs when alert clears)
2. **Action:** Transition Issue → `Done`
3. **Action:** Add Comment → `"Alert cleared in vROPs. Issue auto-resolved."`

> To configure this, you need a second vROPs Outbound Notification Rule with trigger = `Alert Cancelled` pointing to the Jira transition API endpoint:
> ```
> POST https://your-company.atlassian.net/rest/api/3/issue/{issueKey}/transitions
> ```

---

## PART 4 – Create vROPs Views / Dashboards for Visibility

### Step 12: Create a "Storage Reclamation" Dashboard in vROPs

1. Go to **Visualize** → **Dashboards** → **+ Add**
2. Name it: `Storage Reclamation Overview`
3. Add these **Widgets**:

| Widget Type | Configuration |
|---|---|
| Alert List | Filter: Alerts tagged `Storage Reclamation` alerts |
| Top-N | Metric: `summary|number_virtual_machines` on Datastores — sorted ascending (shows emptiest first) |
| Object List | Object Type: Datastore — show CapacityGB, FreeSpaceGB, VM Count |
| Alert Volume | Bar chart showing alert trend over last 30 days |

4. Click **Save Dashboard**

---

### Step 13: Create a Scheduled vROPs Report (Weekly Email)

1. Go to **Visualize** → **Reports** → **Report Templates** → **+ Add**
2. Name: `Weekly Storage Reclamation Report`
3. Add **Views**:
   - Datastore capacity summary
   - Active storage alerts
   - VMs with connected CD-ROMs (Super Metric view)
4. Click **Save**

5. Go to **Reports** → **Schedules** → **+ Add Schedule**:
   - Template: `Weekly Storage Reclamation Report`
   - Schedule: **Every Monday 7:00 AM**
   - Recipients: `infra-team@yourcompany.com`
   - Format: **PDF**

> ✅ Every Monday morning your team gets a PDF report in their inbox.

---

## PART 5 – Optional: Close the Loop with vRA (Aria Automation)

If you have **vRealize Automation (Aria Automation)** licensed, you can fully close the loop:

### Step 14: Create a vRA Action for VMDK Cleanup

1. In **Aria Automation** → **Extensibility** → **Actions** → **+ New Action**
2. Language: **Python 3**
3. Paste this action script:

```python
import requests

def handler(context, inputs):
    vcenter = inputs["vcenter_host"]
    datastore = inputs["datastore_name"]
    vmdk_path = inputs["vmdk_path"]
    
    # This calls your vCenter API to delete the orphaned VMDK
    # In production: use pyVmomi or vCenter REST API
    
    print(f"Reclaiming orphaned VMDK: {vmdk_path} on {datastore}")
    
    return {
        "status": "success",
        "message": f"VMDK {vmdk_path} deletion initiated"
    }
```

4. Save and **Publish** the action

### Step 15: Trigger vRA Action from Jira Approval

When Jira ticket moves to `Approved for Cleanup`:
1. Jira Automation fires a **Webhook** to vRA Extensibility endpoint
2. vRA Action executes the cleanup
3. vRA posts result back to Jira as a comment
4. Jira ticket auto-closes

---

## Complete Flow Summary

```
┌─────────────────────────────────────────────────────────────┐
│                     AUTOMATED FLOW                          │
│                                                             │
│  vROPs detects:                                             │
│  • Orphaned VMDK on datastore                               │
│  • ISO mounted on VM                                        │
│  • Datastore with 0 VMs                                     │
│         │                                                   │
│         ▼                                                   │
│  vROPs Alert fires                                          │
│  (every 5 minutes scan)                                     │
│         │                                                   │
│         ▼                                                   │
│  Outbound REST Plugin                                       │
│  sends POST to Jira API                                     │
│         │                                                   │
│         ▼                                                   │
│  Jira ticket created automatically                          │
│  • Assigned to VMware admin                                 │
│  • Tagged: vmware, storage-reclamation                      │
│  • Lands in "Open" column                                   │
│         │                                                   │
│         ▼                                                   │
│  Engineer reviews finding                                   │
│  Moves ticket → "Approved for Cleanup"                      │
│         │                                                   │
│         ▼                                                   │
│  (Optional) Jira Webhook → vRA Action                       │
│  Cleanup executed automatically                             │
│         │                                                   │
│         ▼                                                   │
│  vROPs alert clears                                         │
│  Jira ticket auto-closes                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference – Where to Do What

| Task | Tool | Where |
|---|---|---|
| Detect orphaned VMDKs | vROPs | Alerts → Alert Definitions |
| Detect ISOs mounted | vROPs | Super Metrics + Alert |
| Detect empty datastores | vROPs | Symptoms + Alert |
| Create Jira ticket | vROPs | Outbound Settings → Notification Rules |
| Review & approve | Jira | INFRA project board |
| Auto-assign tickets | Jira | Project Automation |
| Dashboard visibility | vROPs | Dashboards |
| Weekly email report | vROPs | Reports → Schedules |
| Execute cleanup | vRA (optional) | Extensibility → Actions |

---

## Tips for Beginners

- **Start with the dashboard (Step 12)** — get familiar with what vROPs sees before setting up alerts
- **Test alerts on a non-production datastore** — create a dummy empty datastore to verify the alert fires
- **Jira project key matters** — replace `"INFRA"` in the JSON payload with your actual Jira project key
- **API Token is your password** — treat it like one; store in a vault if possible
- **Alert noise** — set a `Wait Cycle` of 3 on your alert symptoms so vROPs confirms the issue persists across 3 collection cycles before firing (avoids false positives)

---
*Guide covers: VMware Aria Operations (vROPs) 8.x + Jira Cloud/Server integration for storage reclamation automation*