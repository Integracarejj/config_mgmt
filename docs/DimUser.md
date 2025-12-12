
# Executive Overview

# Purpose  
This Azure Function performs the **DimUser ETL** for your TalentLMS integration. It pulls user profiles from **TalentLMS**, enriches them with **HR staging** data (PersonKey and Supervisor), resolves **RoleID** and **SupervisorID** from dimension tables, and **upserts** records into `dbo.DimUser`. It then applies targeted data fixes (a one-off for *Leslie Oliveras* and several manual overrides) and attempts to backfill supervisor emails from a name-mapping table.

# Reads Config  
- Loads `config.json` (co-located with the function) for:
  - **TalentLMS**: `BASE_URL`, `API_KEY`, `API_PASSWORD` (optional)
  - **AzureSQL**: `SERVER`, `DB`, `USER`, `PASSWORD`

# Authenticates / Data Sources  
- **TalentLMS API**: `GET {BASE_URL}/users/` using Basic Auth (`API_KEY`, `API_PASSWORD`)
- **Azure SQL (pymssql)**: Reads/writes across `dbo.DimUser`, `dbo.DimRole`, `dbo.DimSupervisor`, `dbo.HR_Employees_Staging`, `dbo.SupervisorNameMapping`

# Fetches & Normalizes Data  
- **TalentLMS users**: pulls IDs, login, names, status, custom fields:
  - `custom_field_1` → Hire Date (parsed `MM/DD/YYYY`)
  - `custom_field_2` → Location
  - `custom_field_3` → Role (joined to `DimRole` for `RoleID`)
  - `custom_field_8` → Exclusion flag (`"Exclude"` → `FL_Exclude = True`)
- **HR staging**: active rows with normalized names, provides:
  - `PersonKey` by `FullName_Normalized`
  - `Supervisor_Normalized` for Supervisor matching
- **Supervisor resolution**:
  - Directly match `Supervisor_Normalized` against `DimSupervisor.Name` (normalized)
  - Fallback: use `SupervisorNameMapping.SupervisorFullName_TLMS` to map HR name to TLMS name, then match `DimSupervisor`

# SQL Modeling & Upsert Rules  
- If `DimUser.UserID` exists → **UPDATE** all user attributes (login, names, dates, location, role, status, exclusion flag, `PersonKey`, `SupervisorID`, `RoleID`)
- Else → **INSERT** new user row with the same attributes
- **Post-processing**:
  - **One-off fix**: sets `SupervisorID` for *Leslie Oliveras* by joining HR staging and `DimSupervisor`
  - **Backfill supervisor emails**: updates `DimSupervisor.Email` **from** `SupervisorNameMapping.*` (**Caveat** below)
  - **Manual overrides**: explicitly sets `PersonKey` and `SupervisorID` for named employees

# Commits & Logging  
- Logs ETL progress and durations, commits after major phases, and closes the SQL connection cleanly.
- Warning logs when supervisor name resolution fails for a user.

# Key Outcomes  
- Keeps `dbo.DimUser` in sync with TalentLMS and HR supervisor relationships.
- Populates `RoleID` and `SupervisorID` for reporting and downstream joins.
- Applies targeted corrections where automated matching is insufficient.

---

# Function Name
**Identifier:** `DimUser`

## 1. Summary / Descriptor
This function ingests TalentLMS users and merges them into `dbo.DimUser` with enriched attributes from HR staging (`PersonKey`, `Supervisor_Normalized`) and dimension tables (`RoleID`, `SupervisorID`). It performs upserts, targeted fixes (Leslie Oliveras), manual overrides, and attempts supervisor-email backfill via `SupervisorNameMapping`.

---

## 2. Inputs
### A. Data Sources
- **HTTP/API Calls (TalentLMS):**
  - `GET {BASE_URL}/users/`
  - Auth: Basic (`API_KEY`, `API_PASSWORD` optional)
  - Payloads: JSON
  - **Note**: No pagination implemented here; if TLMS returns paged results, consider adding paging.

- **Database (Azure SQL via pymssql, `as_dict` cursor for reads):**
  - **Reads from**:
    - `dbo.HR_Employees_Staging` → `PersonKey`, `FullName_Normalized`, `Supervisor_Normalized`, `Employment_Status='A'`
    - `dbo.DimRole` → `RoleID` by `Name`
    - `dbo.DimSupervisor` → `SupervisorID` by `Name` (normalized)
    - `dbo.SupervisorNameMapping` → mapping `SupervisorName_HR` → `SupervisorFullName_TLMS`
  - **Writes to**:
    - `dbo.DimUser` (INSERT/UPDATE core attributes)
    - `dbo.DimSupervisor` (email backfill step)

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
  - `dbo.DimUser` → **INSERT/UPDATE**
    - Columns: `UserID, Login, FirstName, LastName, FullName, HireDate, Location, Role, Status, CreatedOn, LastUpdated, FL_Exclude, PersonKey, SupervisorID, RoleID`
  - `dbo.DimSupervisor` → **UPDATE**
    - Attempt to set `Email` from `SupervisorNameMapping` (see caveat below)

### B. SharePoint Output
- **None** (no Microsoft Graph or SharePoint interactions in this function)

### C. Other Outputs
- Logging: progress, warnings on supervisor resolution, ETL completion

---

## 4. Runtime Behavior
- **Imports**: `json`, `requests`, `pymssql`, `Path`, `datetime`, `time`, `re`, `logging`, `azure.functions`
- **Trigger Type**: Timer (defined in `function.json`; not shown here)
- **Frequency (schedule/cron)**: (read from `function.json`)
- **Processing flow**:
  1. Load config and connect to Azure SQL
  2. Fetch TLMS users
  3. Load HR staging, roles, supervisors, and name-mapping
  4. Normalize names (`clean_name`)
  5. Resolve `RoleID` and `SupervisorID` (with fallback mapping)
  6. Upsert `DimUser`
  7. Post-fixes: one-off update, supervisor email backfill, manual overrides
  8. Commit and close

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\DimUser`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **Supervisor email backfill likely incorrect**:  
  The query:
  ```sql
   UPDATE ds
  SET ds.Email = snm.SupervisorFullName_TLMS
  FROM dbo.DimSupervisor ds
  INNER JOIN dbo.SupervisorNameMapping snm
      ON ds.Name = snm.SupervisorName_HR
