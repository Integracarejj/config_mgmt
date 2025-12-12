
# Executive Overview

# Purpose  
This Azure Function (file: **DimEnrollment**) performs a **DimUser ETL** rather than an enrollment ETL. It pulls users from **TalentLMS**, enriches with **HR staging** (PersonKey & Supervisor), resolves **RoleID** and **SupervisorID** from dimensions, and **upserts** into `dbo.DimUser`. It also applies targeted data fixes (one-off for *Leslie Oliveras*) and manual overrides, and attempts to backfill supervisor emails from a name-mapping table.

# Reads Config  
- Loads `config.json` for:
  - **TalentLMS**: `BASE_URL`, `API_KEY`, `API_PASSWORD` (optional)
  - **AzureSQL**: `SERVER`, `DB`, `USER`, `PASSWORD`

# Authenticates / Data Sources  
- **TalentLMS API**: `GET {BASE_URL}/users/` with Basic Auth
- **Azure SQL (pymssql)**: reads/writes across `dbo.DimUser`, `dbo.DimRole`, `dbo.DimSupervisor`, `dbo.HR_Employees_Staging`, `dbo.SupervisorNameMapping`

# Fetches & Normalizes Data  
- **TLMS users**: ID, login, names, status, custom fields:
  - `custom_field_1` → HireDate (parsed `MM/DD/YYYY`)
  - `custom_field_2` → Location
  - `custom_field_3` → Role (joined to DimRole for `RoleID`)
  - `custom_field_8` → Exclusion flag (`"Exclude"` → `FL_Exclude = True`)
- **HR staging (active only)**: provides `PersonKey` and `Supervisor_Normalized`
- **Supervisor resolution**:
  - Direct match `Supervisor_Normalized` → `DimSupervisor.Name`
  - Fallback via `SupervisorNameMapping` (`SupervisorName_HR` → `SupervisorFullName_TLMS`)

# SQL Modeling & Upsert Rules  
- If `DimUser.UserID` exists → **UPDATE** attributes (login, names, dates, role, status, flags, keys)
- Else → **INSERT** new row
- **Post-processing**:
  - One-off fix for *Leslie Oliveras* (sets `SupervisorID` via staging and DimSupervisor)
  - Backfill supervisor emails from `SupervisorNameMapping` (**caveat below**)
  - Manual overrides for seven named employees (`PersonKey`, `SupervisorID`)

# Commits & Logging  
- Commits after major phases, logs progress every 25 users, and closes SQL connection cleanly. Warns when supervisor resolution fails.

# Key Outcomes  
- Keeps `dbo.DimUser` aligned with TalentLMS and HR supervisor relationships
- Populates `RoleID` and `SupervisorID` for reporting/joins
- Applies targeted corrections where automated matching is insufficient

---

# Function Name
**Identifier:** `DimEnrollment`

## 1. Summary / Descriptor
Despite its name, this function implements **DimUser ETL**: it ingests TalentLMS users, enriches with HR and dimension data, and upserts into `dbo.DimUser`. It also applies a one-off fix, manual overrides, and a supervisor email backfill from `SupervisorNameMapping`.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls (TalentLMS):**
  - `GET {BASE_URL}/users/`
  - Auth: Basic (`API_KEY`, optional `API_PASSWORD`)
  - Payload: JSON
  - **Note**: No pagination implemented here; if TLMS paginates, add paging.

- **Database (Azure SQL via `pymssql`):**
  - **Reads from**:
    - `dbo.HR_Employees_Staging` → `PersonKey`, `FullName_Normalized`, `Supervisor_Normalized`, `Employment_Status='A'`
    - `dbo.DimRole` → `RoleID` by role `Name`
    - `dbo.DimSupervisor` → `SupervisorID` by `Name`
    - `dbo.SupervisorNameMapping` → map HR supervisor name → TLMS supervisor name
  - **Writes to**:
    - `dbo.DimUser` (INSERT/UPDATE core attributes)
    - `dbo.DimSupervisor` (email backfill step)

### B. Configuration
- **Uses config.json:** Yes (TalentLMS & AzureSQL sections)
- **Uses emails.json:** No
- **Uses Azure env vars:** No (DB credentials come from `config.json`)
- **Key config keys:**
  - `TalentLMS.BASE_URL`, `TalentLMS.API_KEY`, `TalentLMS.API_PASSWORD`
  - `AzureSQL.SERVER`, `AzureSQL.DB`, `AzureSQL.USER`, `AzureSQL.PASSWORD`

---

## 3. Outputs
### A. Database Updates
- **Tables Written To & Operations**
  - `dbo.DimUser` → **INSERT/UPDATE**
    - Columns: `UserID, Login, FirstName, LastName, FullName, HireDate, Location, Role, Status, CreatedOn, LastUpdated, FL_Exclude, PersonKey, SupervisorID, RoleID`
  - `dbo.DimSupervisor` → **UPDATE**
    - Tries to set `Email` from `SupervisorNameMapping` (see caveat)

### B. SharePoint Output
- None

### C. Other Outputs
- Logging: progress every 25 users, warnings on supervisor resolution, completion summary

---

## 4. Runtime Behavior
- **Imports**: `json`, `requests`, `pymssql`, `Path`, `datetime`, `time`, `re`, `logging`, `azure.functions`
- **Trigger Type**: Timer (defined in `function.json`)
- **Frequency (schedule/cron)**: (from `function.json`)
- **Processing Flow**:
  1. Load config; connect to Azure SQL
  2. Fetch TLMS users
  3. Load HR staging, roles, supervisors, and name-mapping
  4. Normalize names (`clean_name`)
  5. Resolve `RoleID`, `SupervisorID` (with fallback)
  6. Upsert `DimUser`
  7. Post-fixes: one-off update, supervisor email backfill, manual overrides
  8. Commit and close

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\DimEnrollment`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **Function mislabeling**: Log message says *“DimUser ETL started”* while the file is **DimEnrollment**. The code implements DimUser logic—not enrollment. Consider renaming the file/function or adjusting the logic to match enrollment needs.
- **Supervisor email backfill likely incorrect**:  
  The query sets `DimSupervisor.Email = SupervisorFullName_TLMS` (a **name**, not an email). If the mapping table has an email column, use that (e.g., `SupervisorEmail_TLMS`); otherwise, remove/replace this step.
- **No TLMS pagination**: Other functions use page-size/page-number; without paging you may miss users in large tenants.
- **Date parsing strictness**: HireDate, CreatedOn, LastUpdated expect fixed string formats; unexpected formats become `None`.
- **Manual overrides**: Hard-coded fixes for seven employees; keep current or migrate to a managed override table.
- **String matching sensitivity**: Supervisor resolution depends on normalized strings; irregular name formats may fail.

---

## 8. Notes / Freeform
- If this function is intended to handle **enrollment** (e.g., `FactEnrollment`/`DimEnrollment`), consider extracting enrollment-specific logic (courses, enrollments, event tables) and moving current user ETL to `DimUser` exclusively.
- Add an **audit log** (like `ETL_AuditLog`) for counts inserted/updated to improve observability for this job.
- Consider a diagnostic report of users without resolved `SupervisorID` or missing `RoleID`.
