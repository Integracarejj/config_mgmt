
# Executive Overview

# Purpose  
This Azure Function runs a multi-step **ETL pipeline** to synchronize TalentLMS metadata (categories and roles) with your **Azure SQL** model and refresh your **supervisor** dimension from HR data. It normalizes names, merges deduplicated supervisors, updates email/name mappings, flags Area General Managers (AGMs), and sets each user’s `SupervisorID`. The job logs each step to `dbo.ETL_AuditLog` for traceability.

# Reads Config  
- Loads `config.json` (co-located with the function) for:
  - **TalentLMS**: `BASE_URL`, `API_KEY`, `API_PASSWORD` (optional)
  - **AzureSQL**: `SERVER`, `DB`, `USER`, `PASSWORD`

# Authenticates with TalentLMS  
- Connects via **Basic Auth** (`API_KEY` / `API_PASSWORD`) and calls:
  - `GET {BASE_URL}/categories/` → category catalog
  - `GET {BASE_URL}/users/` → user catalog to derive roles (from `custom_field_3`)

# Fetches & Processes Data  
- **DimCategory**: MERGE by `CategoryID` to insert/update `Name`, `ParentCategoryID`.
- **DimRole**: Build unique role set from TLMS users and insert **new** role names.
- **HR_Employees_Staging**: Refresh from `dbo.qry_HR_Employees`, then normalize:
  - Parse `FullName` into `FirstName`/`LastName`
  - `FullName_Normalized` → Proper capitalization
  - `Supervisor_Normalized` → Proper capitalization (comma and no-comma cases)

# SQL Modeling & Business Rules  
- **DimSupervisor**:
  - MERGE distinct `Supervisor_Normalized` as `Name`; insert new supervisors
  - Update `Email` from staging (`Company_Email_Address`) when missing
  - Update display `Name` from `dbo.SupervisorNameMapping`
  - Refresh **AGM flags** (`AGM = 'Y'`) for `DimUser.Role = 'Area General Manager'`
  - Deduplicate supervisors using ROW_NUMBER, keep best record (prefers `@integracare.com` email)
- **DimUser**:
  - Set `SupervisorID` by matching normalized full names from staging to `DimSupervisor`
- **Post-Process**:
  - Fill **missing emails** in staging from inactive/terminated rows in `qry_HR_Employees`
- **Audit**:
  - Write step-level entries to `dbo.ETL_AuditLog` with counts and notes

# Commits & Logging  
- Commits after major phases; logs row counts at each stage (Inserted/Updated/Deleted). Ends with “completed successfully” or raises with detailed error logs.

# Key Outcomes  
- Up-to-date, normalized TLMS **categories** and **roles** in SQL
- Clean, deduplicated **supervisor** dimension with emails and AGM flags
- Accurate **SupervisorID** assignment for active users
- End-to-end **audit trail** in `dbo.ETL_AuditLog`

---

# Function Name
**Identifier:** `RoleCatSuper`

## 1. Summary / Descriptor
Comprehensive ETL for TLMS and HR integration: loads/updates **categories** (DimCategory) and **roles** (DimRole) from TalentLMS; refreshes **HR_Employees_Staging**; normalizes names; merges **DimSupervisor** with emails/mapping and flags AGMs; updates **DimUser.SupervisorID**; performs deduplication; and logs each stage in `ETL_AuditLog`.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls (TalentLMS):**
  - `GET {BASE_URL}/categories/` (category catalog)
  - `GET {BASE_URL}/users/` (user catalog for roles via `custom_field_3`)
  - Auth: Basic (`API_KEY`, `API_PASSWORD` optional)
  - Payloads: JSON

- **Database (Azure SQL via pymssql):**
  - **Reads from**:
    - `dbo.qry_HR_Employees` (source for staging refresh)
    - `dbo.DimRole` (existence check by `Name`)
  - **Writes to**:
    - `dbo.DimCategory` (MERGE by `CategoryID`)
    - `dbo.DimRole` (INSERT `Name` for new roles)
    - `dbo.HR_Employees_Staging` (TRUNCATE → INSERT; multiple UPDATE normalizations)
    - `dbo.DimSupervisor` (MERGE new supervisors; UPDATE `Email`/`Name`; AGM flags; duplicate cleanup)
    - `dbo.DimUser` (UPDATE `SupervisorID`)
    - `dbo.ETL_AuditLog` (INSERT audit records)
    - `dbo.SupervisorNameMapping` (read in join to refresh supervisor display `Name`)
  - **Schema focus**: `dbo` (explicitly used across all objects)

### B. Configuration
- **Uses config.json:** Yes (TalentLMS & AzureSQL sections)
- **Uses emails.json:** No
- **Uses Azure env vars:** No (all SQL credentials are read from `config.json`)
- **Key config keys:**
  - `TalentLMS.BASE_URL`, `TalentLMS.API_KEY`, `TalentLMS.API_PASSWORD`
  - `AzureSQL.SERVER`, `AzureSQL.DB`, `AzureSQL.USER`, `AzureSQL.PASSWORD`

---

## 3. Outputs
### A. Database Updates
- **Tables Written To & Operations**
  - `dbo.DimCategory` → **MERGE** (UPDATE `Name`, `ParentCategoryID`; INSERT on not matched)
  - `dbo.DimRole` → **INSERT** (new role `Name`; skip if exists)
  - `dbo.HR_Employees_Staging` → **TRUNCATE**, **INSERT**, **UPDATE** (name/email normalization)
  - `dbo.DimSupervisor` → **MERGE** (insert new), **UPDATE** (`Email`, mapped `Name`), **DELETE** (duplicate cleanup)
  - `dbo.DimUser` → **UPDATE** (`SupervisorID`)
  - `dbo.ETL_AuditLog` → **INSERT** (audit trail entries)

- **Columns Updated / Inserted (high-level)**
  - **DimCategory**: `CategoryID`, `Name`, `ParentCategoryID`
  - **DimRole**: `Name`
  - **HR_Employees_Staging**: `PersonKey`, `FullName`, `Company_Email_Address`, `Supervisor`, `Employment_Status`, `LastUpdated`, plus derived: `FirstName`, `LastName`, `FullName_Normalized`, `Supervisor_Normalized`
  - **DimSupervisor**: `Name`, `Email`, `AGM` (flag), `SupervisorID` (implicit for joins/cleanup context)
  - **DimUser**: `SupervisorID`
  - **ETL_AuditLog**: `TableName`, `RowCount`, `RowsInserted`, `RowsUpdated`, `RowsDeleted`, `RunDateTime`, `Notes`

### B. SharePoint Output
- **None** (this function does not call Microsoft Graph or write to SharePoint)

### C. Other Outputs
- **Audit entries** in `dbo.ETL_AuditLog` per phase (counts + notes)

---

## 4. Runtime Behavior
- **Imports**: `json`, `requests`, `pymssql`, `Path`, `datetime`, `time`, `logging`, `azure.functions`
- **Trigger Type**: Timer (defined in `function.json`; not shown here)
- **Frequency (schedule/cron)**: (read from `function.json`)
- **ETL Phases** (each commit-separated, with logging):
  1. Categories MERGE
  2. Roles INSERT (new only)
  3. HR staging refresh + normalization
  4. DimSupervisor merge/update/flags/dedup
  5. DimUser supervisor assignment
  6. Post-process email fill-in
  7. Audit logging

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\RoleCatSuper`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **Duplicate code block**: The file contains the **entire function logic repeated twice**, including a second `def main(...)` definition. In Python, the **last definition wins**, so the earlier one is shadowed. This doesn’t change behavior, but it’s redundant and can confuse readers/tools.
- **TLMS roles source**: Roles are derived from `users.custom_field_3`. If TLMS field usage changes, role population may become incomplete.
- **Name normalization**: Depends on consistent `FullName`/`Supervisor` formatting. Unusual name patterns may not parse perfectly.
- **Deduplication policy**: Prefers records with `@integracare.com` email, then lower `SupervisorID`. Validate this matches business expectations.
- **AGM flag**: Based solely on `DimUser.Role = 'Area General Manager'`. If role taxonomy changes, flags may misalign.

---

## 8. Notes / Freeform
- Consider adding row-level counts to `ETL_AuditLog` for each sub-step (e.g., DimSupervisor dedup deletions).
- If HR data is large, batching or indexing on staging/dimension keys can improve performance.
- If you later need SharePoint integration here, you could reuse the Microsoft Graph patterns from `SurveyFeedGen`/`SurveyFeedDB`.
