
# Executive Overview

# Purpose  
This Azure Function performs a **DimCourse and FactCourseDevelopment ETL**. It retrieves course metadata from **TalentLMS**, updates or inserts records into `dbo.DimCourse`, and synchronizes `dbo.FactCourseDevelopment` for reporting and analytics. It ensures all course attributes—including custom fields—are normalized and stored for downstream dashboards.

# Reads Config  
- Loads `config.json` for:
  - **TalentLMS**: `BASE_URL`, `API_KEY`, `API_PASSWORD`
  - **AzureSQL**: `SERVER`, `DB`, `USER`, `PASSWORD`

# Authenticates / Data Sources  
- **TalentLMS API**: `GET {BASE_URL}/courses/` using Basic Auth
- **Azure SQL (pymssql)**: Reads/writes across `dbo.DimCourse` and `dbo.FactCourseDevelopment`

# Fetches & Processes Data  
- Pulls all courses from TalentLMS, including:
  - Core fields: `CourseID`, `Code`, `Name`, `CategoryID`, `Description`, `Status`, `Developer`
  - Custom fields: `EOO`, `CRD`, `RWD`, `ASD`, `DED`, `LStaD`, `LStoD`, `SME`, `HEA`, `AGM`, `Other`, `CategoryName`, `DueDate`
- Parses `DueDate` from `custom_field_13` (format: `MM/DD/YYYY`)
- Updates existing courses in `DimCourse` or inserts new ones
- Maintains timestamps: `CreationDate`, `LastUpdate`

# SQL Modeling & Fact Sync  
- After DimCourse updates:
  - Inserts new rows into `FactCourseDevelopment` for courses not yet present
  - Updates existing Fact rows to reflect latest DimCourse attributes
- Ensures Fact table mirrors DimCourse for reporting consistency

# Commits & Logging  
- Logs progress every 20 courses processed
- Commits after DimCourse and FactCourseDevelopment sync
- Closes DB connection gracefully

# Key Outcomes  
- Keeps DimCourse and FactCourseDevelopment aligned with TalentLMS
- Captures all course metadata and custom fields for analytics
- Provides a foundation for dashboards and performance tracking

---

# Function Name
**Identifier:** `DimCourse_FactCourse`

## 1. Summary / Descriptor
This function ingests TalentLMS course data, updates `dbo.DimCourse` with normalized attributes, and synchronizes `dbo.FactCourseDevelopment` for reporting. It handles inserts, updates, and ensures custom fields and due dates are properly stored.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls (TalentLMS):**
  - `GET {BASE_URL}/courses/`
  - Auth: Basic (`API_KEY`, `API_PASSWORD`)
  - Payload: JSON

- **Database (Azure SQL via pymssql):**
  - **Reads from**:
    - `dbo.DimCourse` (existence check by `CourseID`)
    - `dbo.FactCourseDevelopment` (existence check for Fact sync)
  - **Writes to**:
    - `dbo.DimCourse` (INSERT/UPDATE)
    - `dbo.FactCourseDevelopment` (INSERT new rows, UPDATE existing rows)

### B. Configuration
- **Uses config.json:** Yes
- **Uses emails.json:** No
- **Uses Azure env vars:** No (DB credentials come from `config.json`)
- **Key config keys:**
  - `TalentLMS.BASE_URL`, `TalentLMS.API_KEY`, `TalentLMS.API_PASSWORD`
  - `AzureSQL.SERVER`, `AzureSQL.DB`, `AzureSQL.USER`, `AzureSQL.PASSWORD`

---

## 3. Outputs
### A. Database Updates
- **Tables Written To & Operations**
  - `dbo.DimCourse` → **INSERT/UPDATE**
    - Columns: `CourseID, Code, Name, CategoryID, Description, Status, Developer, EOO, CRD, RWD, ASD, DED, LStaD, LStoD, SME, HEA, AGM, Other, CategoryName, DueDate, CreationDate, LastUpdate`
  - `dbo.FactCourseDevelopment` → **INSERT/UPDATE**
    - Mirrors DimCourse attributes for reporting

### B. SharePoint Output
- None

### C. Other Outputs
- Logging: progress every 20 courses, warnings on date parsing, completion summary

---

## 4. Runtime Behavior
- **Imports**: `logging`, `json`, `requests`, `pymssql`, `Path`, `datetime`, `time`, `azure.functions`
- **Trigger Type**: Timer (defined in `function.json`)
- **Frequency (schedule/cron)**: (from `function.json`)
- **Processing Flow**:
  1. Load config; connect to Azure SQL
  2. Fetch TalentLMS courses
  3. Upsert DimCourse records
  4. Sync FactCourseDevelopment (insert new, update existing)
  5. Commit and close

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\DimCourse_FactCourse`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **DueDate parsing**: Expects `MM/DD/YYYY`; invalid formats logged as warnings and skipped.
- **Custom fields**: Hard-coded mapping; changes in TLMS custom field definitions require code updates.
- **Fact sync logic**: Inserts only if CourseID not in Fact; updates all attributes otherwise.
- **No pagination**: If TLMS courses exceed API default page size, consider adding paging logic.

---

## 8. Notes / Freeform
- Consider adding an **audit log** (ETL_AuditLog) for counts inserted/updated.
- If Fact table grows large, optimize UPDATE with indexed joins or batch processing.
- Extend logic to handle
