# Purpose:
This Azure Function automates the process of extracting survey responses from TalentLMS and pushing them into a SharePoint list for reporting and analysis. It runs on a timer trigger and ensures that only new, meaningful survey answers are added, avoiding duplicates and filtering out empty or irrelevant responses.
What it accomplishes:

# Connects to TalentLMS API to retrieve:

All courses, categories, and enrolled users.
Survey units within courses and their associated responses.


# Connects to Microsoft Graph API to:

Authenticate using client credentials.
Fetch existing SharePoint list items to prevent duplicates.
Post new survey responses in batches for efficiency.


# Implements data normalization and filtering:

Removes meaningless answers (e.g., “none”, “n/a”).
Builds a CompoundKey for deduplication.


# Logs detailed processing metrics:

Total processed, added, duplicates skipped, and filtered answers.


Handles pagination for large datasets from TalentLMS and SharePoint.
Uses Azure environment variables for secure configuration.

# Key Outcomes:

Keeps SharePoint survey data synchronized with TalentLMS.
Ensures data integrity (no duplicates, no junk answers).
Provides a scalable approach for large volumes of survey responses.


# Function Name
**Identifier:** `SurveyFeedGen`

## 1. Summary / Descriptor
This function retrieves survey responses from TalentLMS courses and posts them into a SharePoint list via Microsoft Graph API. It ensures deduplication, filters meaningless answers, and runs on a timer trigger for scheduled synchronization.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls:**
  - TalentLMS:
    - `GET {TALENTLMS_BASE_URL}/courses`
    - `GET {TALENTLMS_BASE_URL}/users/page_size:{page_size},page_number:{page}`
    - `GET {TALENTLMS_BASE_URL}/categories`
    - `GET {TALENTLMS_BASE_URL}/courses/id:{course_id}`
    - `GET {TALENTLMS_BASE_URL}/getsurveyanswers?survey_id={survey_id}&user_id={user_id}`
  - Microsoft Graph:
    - `POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token` (OAuth token)
    - `GET https://graph.microsoft.com/v1.0/sites/{SITE_ID}/lists/{LIST_ID}/items?expand=fields` (existing items)
    - `POST https://graph.microsoft.com/v1.0/sites/{SITE_ID}/lists/{LIST_ID}/items` (new items)
  - Headers/Auth:
    - OAuth Bearer token for Graph
    - Basic Auth for TalentLMS (API key)
  - Pagination:
    - TalentLMS users fetched in pages of 100
    - SharePoint items paginated via `@odata.nextLink`
  - Payload format:
    - JSON for Graph POST
    - JSON responses from TalentLMS

- **Files / Blobs / SharePoint / Others:**
  - SharePoint List ID: `{LIST_ID}`
  - Site ID: `{SITE_ID}`

### B. Configuration
- **Uses config.json:** No
- **Uses emails.json:** No
- **Uses Azure env vars:** Yes
  - List variables: `AppRegClientID`, `AppRegTenantID`, `AppRegSecretValue`, `TALENTLMS_BASE_URL`, `TALENTLMS_API_KEY`, `SP_SITE_ID`, `SP_LIST_ID`

---

## 3. Outputs
### A. Database Updates
- None (No DB writes; all updates are to SharePoint list)

### B. SharePoint Output
- Writes detected: Yes
  - Methods/verbs: `POST`
  - Endpoints:
    - `https://graph.microsoft.com/v1.0/sites/{SITE_ID}/lists/{LIST_ID}/items`
- Columns populated:
  - `Title`, `CourseName`, `CourseCreator`, `Category`, `User`, `Question`, `Answer`, `SurveyID`, `UserID`, `Date`, `CourseCode`, `Developer`, `CompoundKey`

### C. Other Outputs
- None (No emails, CSV, or PowerBI refresh)

---

## 4. Runtime Behavior
- Imports: `os`, `logging`, `requests`, `datetime`, `time`, `azure.functions`
- Frequency (schedule/cron): Defined in `function.json` (Timer trigger)
- Trigger Type: Timer

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (Configured in Azure)
- Resource Group: (Configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\SurveyFeedGen`

---

## 6. Triggers
- Trigger: Auto
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- Requires valid TalentLMS API key and Microsoft Graph credentials.
- Deduplication relies on `CompoundKey` (user_id + course_code + survey_id).
- Filters meaningless answers (e.g., “none”, “n/a”).
- Large datasets may cause throttling; uses batch posting with pauses.

---

## ## 8. Notes / Freeform
- Ideal for syncing survey feedback for reporting dashboards.
