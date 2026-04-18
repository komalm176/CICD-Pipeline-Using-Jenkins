# Logic App — Email Processor
## Deployment & Testing Guide

---

## Prerequisites

Before deploying, ensure the following exist in Azure:

| Resource | Notes |
|---|---|
| Azure Resource Group | Where Logic App will be deployed |
| Office 365 Outlook Mailbox | The inbox to be polled |
| Azure Storage Account | With `email-staging` container created |
| Azure Service Bus Namespace | With a queue created |
| `FailedEmails` folder in Outlook | Create manually in Outlook before testing |

---

## Pre-Deployment Steps

### 1. Create `FailedEmails` folder in Outlook
- Open Outlook (web or desktop)
- Right-click on your mailbox → **Create new folder**
- Name it exactly: `FailedEmails`

### 2. Create `email-staging` container in Azure Storage
- Go to Azure Portal → Storage Account → Containers
- Click **+ Container**
- Name: `email-staging`
- Access level: **Private**

### 3. Create Service Bus Queue
- Go to Azure Portal → Service Bus Namespace → Queues
- Click **+ Queue**
- Set the following **manually**:
  - `Max delivery count` = **10**
  - `Message lock duration` = **5 minutes**
  - `Message time to live` = **14 days** (or as required)

---

## Deployment Steps

### Option A — Azure Portal

1. Go to **Azure Portal** → search **Deploy a custom template**
2. Click **Build your own template in the editor**
3. Paste contents of `LogicApp_EmailProcessor.json`
4. Click **Save**
5. Fill in parameters or upload `LogicApp_EmailProcessor.parameters.json`
6. Click **Review + Create** → **Create**

### Option B — Azure CLI

```bash
# Login to Azure
az login

# Set your subscription
az account set --subscription "<YOUR-SUBSCRIPTION-ID>"

# Deploy
az deployment group create \
  --resource-group "<YOUR-RESOURCE-GROUP>" \
  --template-file LogicApp_EmailProcessor.json \
  --parameters LogicApp_EmailProcessor.parameters.json
```

---

## Post-Deployment Steps

### 1. Authorise Office 365 Connection
- Go to Azure Portal → **API Connections** → `office365`
- Click **Edit API connection**
- Click **Authorise** → sign in with the mailbox account
- Click **Save**

### 2. Enable Diagnostic Settings (Azure Monitor)
- Go to Logic App → **Diagnostic settings** → **+ Add diagnostic setting**
- Name: `LogicApp-Monitor`
- Check: **allLogs** and **AllMetrics**
- Destination: **Send to Log Analytics workspace**
- Select your Log Analytics workspace
- Click **Save**

### 3. Verify Logic App is enabled
- Go to Logic App → Overview
- Ensure status shows **Enabled**

---

## Configuration — App Settings

All configurable values are ARM template parameters:

| Parameter | Default | Description |
|---|---|---|
| `emailPollCount` | `50` | Emails fetched per poll |
| `maxAttachmentSizeMB` | `25` | Max attachment size in MB |
| `stagingContainer` | `email-staging` | Blob staging container name |
| `failedEmailsFolder` | `FailedEmails` | Outlook folder for failed emails |
| `serviceBusQueueName` | *(required)* | Service Bus queue name |

To change any value after deployment:
- Go to Logic App → **Logic app designer** → update the relevant parameter
- Or redeploy the ARM template with updated parameters

---

## Testing

### Test 1 — Happy path (email with attachment)
1. Send an email with a PDF attachment to the monitored mailbox
2. Wait up to 5 minutes for Logic App to poll
3. **Expected:**
   - Blob appears in `email-staging/{messageId}_{datetime}/` folder
   - Service Bus message appears in queue
   - Email marked as **read** in Inbox

### Test 2 — Email with no attachment
1. Send an email with no attachment
2. Wait up to 5 minutes
3. **Expected:**
   - No blob uploaded (nothing to upload)
   - Service Bus message sent with `EmailAttachmentPath = null`
   - Email marked as **read**

### Test 3 — Large attachment (> 25MB)
1. Send an email with an attachment larger than 25MB
2. Wait up to 5 minutes
3. **Expected:**
   - Email moved to `FailedEmails` folder
   - Azure Monitor log entry with severity `Critical` and message `LogicAppFailed - Attachment exceeds size limit`
   - Email **not** marked as read in Inbox

### Test 4 — Multiple attachments
1. Send an email with 3 attachments (all under 25MB)
2. Wait up to 5 minutes
3. **Expected:**
   - All 3 blobs appear in same staging folder
   - One Service Bus message sent
   - Email marked as read

### Test 5 — Email with ZIP attachment
1. Send an email with a ZIP file attachment
2. Wait up to 5 minutes
3. **Expected:**
   - ZIP uploaded to staging folder as-is (Function App handles extraction)
   - Service Bus message sent
   - Email marked as read

### Test 6 — Concurrency (2 emails arrive simultaneously)
1. Send 2 emails at the same time
2. Wait for first poll
3. **Expected:**
   - Both emails processed sequentially (not in parallel)
   - Both marked as read
   - No duplicate blobs

### Test 7 — Logic App downtime recovery
1. Disable the Logic App
2. Send 3 emails to the mailbox
3. Re-enable the Logic App
4. Wait up to 5 minutes
5. **Expected:**
   - All 3 emails processed on next poll
   - All marked as read

---

## Monitoring

### View Logic App run history
- Go to Logic App → **Overview** → **Runs history**
- Click any run to see step-by-step execution details

### View Azure Monitor logs
- Go to Log Analytics Workspace → **Logs**
- Query Logic App logs:
```kusto
AzureDiagnostics
| where ResourceType == "WORKFLOWS"
| where resource_workflowName_s == "email-processor-logic-app"
| where status_s == "Failed"
| order by TimeGenerated desc
```

### Check for failed emails
- Open Outlook → navigate to `FailedEmails` folder
- Each email here represents a processing failure
- Check Azure Monitor for the corresponding Critical log entry

---

## Troubleshooting

| Issue | Solution |
|---|---|
| Logic App not triggering | Check Office 365 connection is authorised |
| Blobs not appearing in staging | Check Storage connection string and container name |
| Service Bus messages not appearing | Check Service Bus connection string and queue name |
| Emails not being marked as read | Check Office 365 connection permissions |
| All emails going to FailedEmails | Check body size and attachment size limits |
