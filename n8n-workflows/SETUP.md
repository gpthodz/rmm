# Halo PSA Email Alert to Ticket — Setup Guide

## What This Does

1. **Polls** an Office 365 mailbox every minute for new unread emails
2. **Parses** the subject and body using regex + GPT-4o-mini to extract company name, device/hostname, severity, and alert type
3. **Looks up** the matching client and asset in Halo PSA
4. **Creates a ticket** assigned to that client and device with correct priority
5. **Flags** any alert that couldn't be matched to a client — sends a review email and still creates the ticket

---

## Prerequisites

- n8n instance (self-hosted or cloud)
- Office 365 account / mailbox dedicated to receiving alerts
- OpenAI API key (for GPT-4o-mini parsing)
- Halo PSA API credentials (client ID + secret)

---

## Step 1: Import the Workflow

1. In n8n, go to **Workflows → Import from File**
2. Select `halo-psa-email-alert-to-ticket.json`

---

## Step 2: Configure Credentials

### Office 365
1. Go to **Credentials → Add Credential → Microsoft Outlook OAuth2 API**
2. Register an app in Azure AD with `Mail.Read` and `Mail.Send` permissions
3. Update the credential ID `OUTLOOK_CREDENTIAL_ID` in both Outlook nodes

### Azure OpenAI
- No credential node needed — auth uses n8n Variables (see Step 3)
- Endpoint is pre-configured: `https://dtho-mlc062v8-eastus2.openai.azure.com/`

### Halo PSA
- No credential node needed — auth is handled via HTTP request using n8n Variables (see Step 3)

---

## Step 3: Set n8n Variables

Go to **Settings → Variables** and create:

| Variable | Description | Example |
|---|---|---|
| `AZURE_OPENAI_API_KEY` | API key from Azure OpenAI resource | `BLEl4av3f...` |
| `HALO_INSTANCE` | Your Halo subdomain | `mycompany` |
| `HALO_CLIENT_ID` | Halo PSA API client ID | `abc123` |
| `HALO_CLIENT_SECRET` | Halo PSA API client secret | `secret...` |
| `HALO_TICKET_TYPE_ID` | Ticket type ID for alerts | `1` |
| `ALERT_REVIEW_EMAIL` | Who gets notified on unmatched alerts | `helpdesk@you.com` |

**Azure OpenAI endpoint and deployment** are hardcoded in the workflow:
- Endpoint: `https://dtho-mlc062v8-eastus2.openai.azure.com/`
- Deployment: `gpt-5.4-nano` (most cost-effective, retires Mar 2027)

To get your Halo API credentials:
1. In Halo PSA go to **Configuration → Integrations → Halo PSA API**
2. Create an application with `edit:Tickets`, `read:Clients`, `read:Assets` scopes

---

## Step 4: Customize Email Parsing (Optional)

Open the **"Parse Alert from Email"** Code node to add regex patterns for your specific alert services.

Example services to tailor for:
- **SolarWinds / N-central** — look for `Node:`, `Status:`
- **Datto RMM** — look for `Device:`, `Site:`
- **Webroot** — look for `Endpoint:`, `Threat:`
- **ConnectWise Automate** — look for `Computer:`, `Client:`
- **Veeam** — look for `Job:`, `Server:`

The AI node handles unknown formats automatically.

---

## Step 5: Activate

1. Click the **Active** toggle in the top-right of the workflow
2. The trigger will poll for new emails every minute
3. Test by sending a sample alert email to the monitored mailbox

---

## Workflow Map

```
[Outlook Trigger]
      ↓
[Regex Parse Email]
      ↓
[GPT-4o-mini Extract]
      ↓
[Merge Parsed Data]
      ↓
[Halo Auth Token]
      ↓ ↓
[Find Client] [Find Asset]  ← runs in parallel
      ↓ ↓
[Resolve IDs]
      ↓
[Create Ticket in Halo PSA]
      ↓
[Client Found?]
  YES → Done
  NO  → Send review email + Done
```

---

## Ticket Priority Mapping

| Severity | Halo Priority ID |
|---|---|
| Critical | 1 |
| High | 2 |
| Medium | 3 |
| Low | 4 |

Adjust `priorityId` values in **Resolve Client & Asset IDs** to match your Halo PSA priority configuration.
