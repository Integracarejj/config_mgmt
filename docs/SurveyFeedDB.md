
# Executive Overview

# Purpose  
This Azure Function performs an **ETL (Extract, Transform, Load)** process that moves survey feedback data from a **SharePoint list** into an **Azure SQL database**, ensuring data consistency, audit tracking, and snapshotting for trend analysis.

# Reads Config  
- Loads `config.json` for Microsoft Graph credentials and SharePoint site/list details.
- Reads Azure SQL connection details from **environment variables** (`DB_Server`, `DB_Name`, `DB_User`, `DB_Pass`).

# Authenticates with Microsoft Graph  
- Uses **client credentials flow** to obtain an OAuth token.
- Required for accessing SharePoint list items via Graph API.

# Fetches SharePoint Data  
- Retrieves all items from the specified SharePoint list using **Graph API**.
- Handles pagination with `@odata.nextLink`.
- Extracts relevant fields such as `CompoundKey`, `ActionTaken`, `Answer`, `Developer`, `CourseName`, etc.

# Connects to Azure SQL  
- Establishes a connection using **pymssql**.
- Targets tables:
  - `dbo.DimSurveyFeed` (main dimension table)
  - `dbo.FactSurveyStatusChange` (audit table for status changes)
  - `dbo.FactSurveyStatusSnapshot` (daily snapshot for trend analysis)

# Processes Each Item  
- Checks if `CompoundKey` exists in `DimSurveyFeed`:
  - If exists → Updates record if status, answer, or notes changed.
  - Logs status changes in `FactSurveyStatusChange`.
- If new → Inserts into `DimSurveyFeed`.
- Inserts a snapshot into `FactSurveyStatusSnapshot` if not already present for today.

# Commits Changes  
- Commits all inserts/updates.
- Logs counts for inserted, updated, audit entries, and snapshots.
- Closes SQL connection gracefully.

# Key Outcomes  
- Synchronizes SharePoint survey feedback with Azure SQL.
- Maintains historical audit trail of status changes.
- Enables trend reporting via daily snapshots.

---

# Function Name
**Identifier:** `SurveyFeedDB`

## 1. Summary / Descriptor
This function extracts survey feedback items from a SharePoint list and loads them into Azure SQL tables for reporting and analytics. It ensures deduplication, updates changed records, logs status changes, and creates daily snapshots for trend analysis.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls:**
  - Microsoft Graph:
    - `POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token` (OAuth token)
    - `GET https://graph.microsoft.com/v1.0/sites/integracare.sharepoint.com:/sites/{SITE_NAME}:/lists/{LIST_NAME}/items?expand=fields&$top=100` (fetch list items)
  - Headers/Auth:
    - OAuth Bearer token for Graph
  - Pagination:
    - Handles `@odata.nextLink` for large lists
  - Payload format:
    - JSON responses from Graph API

- **Files / Blobs / SharePoint / Others:**
  - Config file: `config.json`
  - SharePoint List: `{LIST_NAME}` in site `{SITE_NAME}`

### B. Configuration
- **Uses config.json:** Yes
- **Uses emails.json:** No
- **Uses Azure env vars:** Yes
  - `DB_Server`, `DB_Name`, `DB_User`, `DB_Pass`

---

## 3. Outputs
### A. Database Updates
- **Tables Written To:**
  - `dbo.DimSurveyFeed` → Inserts new survey feedback or updates existing records.
  - `dbo.FactSurveyStatusChange` → Logs old vs. new status when changed.
  - `dbo.FactSurveyStatusSnapshot` → Inserts daily snapshot for trend analysis.
- **Operations detected:** INSERT, UPDATE
- **Columns updated:**
  - Developer, CourseCode, CourseName, Category, SurveyID, CourseID, UserID, Status, CourseCreator, User, Question, Answer, Notes, timestamps, audit fields.

### B. SharePoint Output
- None (Read-only from SharePoint list)

### C. Other Outputs
- None (No emails, CSV, or PowerBI refresh)

---

## 4. Runtime Behavior
- Imports: `logging`, `azure.functions`, `requests`, `json`, `os`, `datetime.date`, `pymssql`
- Frequency (schedule/cron): Defined in `function.json` (Timer trigger)
- Trigger Type: Timer

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (Configured in Azure)
- Resource Group: (Configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\SurveyFeedDB`

---

## 6. Triggers
- Trigger: Auto
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- Requires valid Microsoft Graph credentials in `config.json`.
- Requires Azure SQL credentials in environment variables.
- Large SharePoint lists may require pagination handling.
- Updates only occur if status, answer, or notes differ from existing record.

---

## 8. Notes## 8. Notes / Freeform
- Ideal for building a historical data warehouse for survey feedback.
