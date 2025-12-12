Executive Overview
# Purpose
This Azure Function automates sending email notifications to course developers when survey feedback items in SharePoint are marked as “Awaiting Review.” It ensures timely follow-up on user feedback collected via TalentLMS surveys.

# What It Does
script compiles any survey results by developer then sends email to individual developer in a table display, including a link to the surveyfeedback list.

# Reads configuration from config.json and an email mapping file to determine:

Azure credentials for Microsoft Graph API.
SharePoint site and list details.
Email sender and recipient mappings.


# Authenticates with Microsoft Graph using client credentials.
Fetches all items from a SharePoint list and filters those with ActionTaken = "Awaiting Review".
Groups pending feedback items by Developer and generates HTML tables summarizing questions and answers.
Sends personalized emails to each developer (or a test email if TEST_MODE is enabled).
Includes a SharePoint link for developers to review and update feedback status.
Logs success/failure for each email and provides a summary.

# Key Outcomes

Ensures developers are promptly notified of pending survey feedback.
Provides a structured, styled HTML summary of items requiring attention.
Supports test mode for safe validation before production runs.


# Function Name
**Identifier:** `SurveyFeedReports`

## 1. Summary / Descriptor
This function sends email notifications to course developers when survey feedback items in SharePoint are marked as “Awaiting Review.” It authenticates via Microsoft Graph, retrieves SharePoint list items, groups them by developer, and sends HTML-formatted emails with actionable links.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls:**
  - Microsoft Graph:
    - `POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token` (OAuth token)
    - `GET https://graph.microsoft.com/v1.0/sites/integracare.sharepoint.com:/sites/{SITE_NAME}:/lists/{LIST_NAME}/items?expand=fields` (fetch list items)
    - `POST https://graph.microsoft.com/v1.0/users/{FROM_ADDRESS}/sendMail` (send email)
  - Headers/Auth:
    - OAuth Bearer token for Graph
  - Payload format:
    - JSON for Graph POST
    - HTML email body with embedded table

- **Files / Blobs / SharePoint / Others:**
  - Config file: `config.json`
  - Email mapping file: defined in `config["survey"]["email_mapping_file"]`
  - SharePoint List: `{LIST_NAME}` in site `{SITE_NAME}`

### B. Configuration
- **Uses config.json:** Yes
- **Uses emails.json:** No (uses email mapping JSON)
- **Uses Azure env vars:** No (reads from config.json)
- **Test Mode:** Controlled by `config["testemail"]["test_toggle"]`
- **Key Config Variables:**
  - `tenant_id`, `client_id`, `client_secret`
  - `site_name`, `list_name`
  - `from_address`

---

## 3. Outputs
### A. Database Updates
- None (No DB writes)

### B. SharePoint Output
- None (Read-only from SharePoint list)

### C. Other Outputs
- **Emails sent via Microsoft Graph**
  - Subject: `Items Awaiting Review - TLMS Survey Feedback`
  - Body: HTML table of pending items + SharePoint link
  - Recipients: Developer email from mapping file (or test email if TEST_MODE)

---

## 4. Runtime Behavior
- Imports: `logging`, `azure.functions`, `requests`, `json`, `pandas`, `os`
- Frequency (schedule/cron): Defined in `function.json` (Timer trigger)
- Trigger Type: Timer

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (Configured in Azure)
- Resource Group: (Configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\SurveyFeedReports`

---

## 6. Triggers
- Trigger: Auto
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- Requires valid Microsoft Graph credentials in `config.json`.
- Email mapping file must include all developer names.
- TEST_MODE routes all emails to a single address for validation.
- SharePoint link hardcoded in HTML body; ensure it matches production environment.

---

## 8. Notes / Freeform
- Ideal for workflow automation to ensure timely review of survey feedback.
- Extendable for additional filtering or custom email
