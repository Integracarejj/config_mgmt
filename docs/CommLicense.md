
# Executive Overview

# Purpose  
This Azure Function automates **license expiration tracking** for items in a SharePoint list. It calculates the number of days until each license expires (`DaysTillExpire`) and assigns a corresponding **status** (e.g., Active, Renew Soon, Expired). It updates these fields in SharePoint via **Microsoft Graph API**, ensuring compliance visibility and proactive renewal management.

# Reads Config  
- Loads `config.json` for:
  - **Azure AD / Graph**: `tenant_id`, `client_id`, `client_secret`
  - **SharePoint**: `site_id`, `list_id`
  - **test_toggle**: Enables safe dry-run mode (logs updates without applying them)

# Authenticates / Data Sources  
- **Microsoft Graph API**:
  - Acquires OAuth token using **client credentials flow**
  - Fetches all list items with expanded fields
  - Updates item fields via PATCH requests

# Processing Logic  
1. **Fetch all items** from SharePoint list using Graph API with pagination.
2. For each item:
   - Parse `ExpireDate` (supports multiple Graph formats).
   - Calculate `DaysTillExpire` (difference between today and expiration date).
   - Determine `Status`:
     - `Expired` → days ≤ 0
     - `Renew Soon` → 1–30 days
     - `Renewal Stage` → 31–120 days
     - `Active` → ≥121 days
     - `Non-Expire` → no expiration date
3. Compare calculated values with current fields:
   - If different, prepare update payload.
   - In **TEST MODE**, log intended changes without applying.
   - Otherwise, PATCH updates to SharePoint via Graph API.
4. Log summary of items updated.

# Commits & Logging  
- Logs each update (or simulated update in test mode).
- Logs total items processed and updated.
- Handles errors gracefully with detailed logs.

# Key Outcomes  
- Keeps SharePoint license list current with expiration countdown and status.
- Enables proactive renewal workflows and compliance reporting.
- Supports safe testing before production updates.

---

# Function Name
**Identifier:** `CommLicense`

## 1. Summary / Descriptor
This function updates `DaysTillExpire` and `Status` fields for licenses stored in a SharePoint list. It calculates expiration intervals, assigns statuses, and applies changes via Microsoft Graph API. Includes a test mode for safe validation.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls (Microsoft Graph):**
  - `POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token` → OAuth token
  - `GET https://graph.microsoft.com/v1.0/sites/{site_id}/lists/{list_id}/items?expand=fields&top=200` → fetch list items
  - `PATCH https://graph.microsoft.com/v1.0/sites/{site_id}/lists/{list_id}/items/{item_id}/fields` → update fields
  - Headers/Auth:
    - Bearer token from OAuth
    - Content-Type: `application/json`
  - Payload format:
    - JSON for PATCH updates (e.g., `{ "DaysTillExpire": 45, "field_4": "Renew Soon" }`)

- **Files / Config:**
  - `config.json` with keys:
    - `tenant_id`, `client_id`, `client_secret`
    - `site_id`, `list_id`
    - `test_toggle` (Y/N)

### B. Configuration
- **Uses config.json:** Yes
- **Uses emails.json:** No
- **Uses Azure env vars:** No
- **Key config keys:** tenant_id, client_id, client_secret, site_id, list_id, test_toggle

---

## 3. Outputs
### A. SharePoint Updates
- **Fields Updated:**
  - `DaysTillExpire` → integer or null
  - `field_4` (Status) → one of:
    - `Expired`
    - `Renew Soon`
    - `Renewal Stage`
    - `Active`
    - `Non-Expire`
- **Operations:** PATCH via Graph API for items needing updates

### B. Database Updates
- None

### C. Other Outputs
- Logging:
  - Items fetched
  - Items updated (or simulated in test mode)
  - Summary counts

---

## 4. Runtime Behavior
- **Imports:** `logging`, `json`, `requests`, `datetime`, `date`, `timezone`, `Path`, `azure.functions`
- **Trigger Type:** Timer (defined in `function.json`)
- **Frequency (schedule/cron):** (from `function.json`)
- **Processing Flow:**
  1. Load config
  2. Authenticate to Graph
  3. Fetch SharePoint items
  4. Calculate days and status
  5. Compare and update if needed
  6. Log summary

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\CommLicense`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **Field name mismatch risk**: Status field uses `field_4` internally; confirm this matches SharePoint schema.
- **Date parsing**: Handles multiple Graph formats but may fail on unexpected custom formats.
- **Test mode**: Ensure `test_toggle` is set correctly; otherwise, updates may apply unintentionally.
- **Large lists**: Pagination handled via `@odata.nextLink`, but performance may degrade for very large lists.
- **Non-expiring licenses**: Marked as `Non-Expire`; verify business logic aligns with compliance rules.

---

## 8. Notes / Freeform
- Consider adding an **audit log** in SharePoint or a database for tracking changes.
- Extend logic to send notifications for items nearing expiration.
