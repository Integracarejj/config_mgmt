
# Executive Overview

# Purpose  
This Azure Function builds and refreshes the **CourseRoleDays** mapping table, which defines how many days each role is allowed for specific courses. It dynamically calculates `DaysAllowed` based on custom fields in `DimCourse` and role names in `DimRole`, ensuring compliance with training timelines.

# Reads Config  
- Loads `config.json` for:
  - **AzureSQL**: `SERVER`, `DB`, `USER`, `PASSWORD`

# Authenticates / Data Sources  
- **Azure SQL (pymssql)**: Reads from `DimCourse` and `DimRole`, writes to `CourseRoleDays` and `ETL_AuditLog`.

# Processing Logic  
1. **Truncate CourseRoleDays**: Clears existing mappings to avoid duplicates.
2. **Populate CourseRoleDays**:
   - Cross-joins `DimCourse` and `DimRole`.
   - Calculates `DaysAllowed`:
     - If role = *Area General Manager* → use `DimCourse.AGM` if present.
     - If role = *Administrative Services Director* → use `DimCourse.ASD`.
     - If role = *EOO* → use `DimCourse.EOO`.
     - Else → default to 90 days.
   - Inserts rows for courses where at least one of `AGM`, `ASD`, or `EOO` is not null.
3. **Audit Logging**:
   - Inserts an entry into `ETL_AuditLog` with row counts and notes.

# Commits & Logging  
- Logs each major step (truncate, populate, audit).
- Commits after inserts and audit logging.
- Closes DB connection gracefully.

# Key Outcomes  
- Provides a dynamic mapping of courses to roles with allowed completion days.
- Supports compliance reporting and training deadline enforcement.
- Maintains an audit trail for ETL transparency.

---

# Function Name
**Identifier:** `CourseRoleDaysHelper`

## 1. Summary / Descriptor
This function refreshes the `CourseRoleDays` table by dynamically calculating allowed days for each role-course combination based on custom fields in `DimCourse`. It ensures accurate training timelines and logs the process in `ETL_AuditLog`.

---

## 2. Inputs
### A. Data Sources
- **Database (Azure SQL via pymssql):**
  - **Reads from**:
    - `dbo.DimCourse` → course metadata and custom fields (`AGM`, `ASD`, `EOO`)
    - `dbo.DimRole` → role names and IDs
  - **Writes to**:
    - `dbo.CourseRoleDays` → inserts calculated mappings
    - `dbo.ETL_AuditLog` → logs ETL activity

### B. Configuration
- **Uses config.json:** Yes (AzureSQL section)
- **Uses emails.json:** No
- **Uses Azure env vars:** No
- **Key config keys:**
  - `AzureSQL.SERVER`, `AzureSQL.DB`, `AzureSQL.USER`, `AzureSQL.PASSWORD`

---

## 3. Outputs
### A. Database Updates
- **Tables Written To & Operations**
  - `dbo.CourseRoleDays` → **TRUNCATE**, then **INSERT**
    - Columns: `CourseID`, `RoleID`, `RoleName`, `CourseName`, `DaysAllowed`
  - `dbo.ETL_AuditLog` → **INSERT**
    - Columns: `TableName`, `RowCount`, `RowsInserted`, `RunDateTime`, `Notes`

### B. SharePoint Output
- None

### C. Other Outputs
- Logging: row counts, ETL completion status

---

## 4. Runtime Behavior
- **Imports**: `logging`, `json`, `pymssql`, `Path`, `azure.functions`
- **Trigger Type**: Timer (defined in `function.json`)
- **Frequency (schedule/cron)**: (from `function.json`)
- **Processing Flow**:
  1. Load config and connect to Azure SQL
  2. Truncate `CourseRoleDays`
  3. Populate mappings via SQL `INSERT ... SELECT` with CASE logic
  4. Log audit entry
  5. Commit and close

---

## 5. Hosting / Deployment
- Hosting Platform: Azure Functions
- Function App Name: (configured in Azure)
- Resource Group: (configured in Azure)
- Python Version: 3.x
- Source Code Repository / Path: `C:\Users\JeremyJoyner\AzureFunctions\TalentLMS\CourseRoleDaysHelper`

---

## 6. Triggers
- Trigger: Auto (Timer)
- If automatic:
  - Cron/Schedule expression: (From `function.json`)

---

## 7. Gotchas / Caveats
- **Hard-coded role logic**: CASE statement only handles `Area General Manager`, `Administrative Services Director`, and `EOO`. Additional roles require code changes.
- **Default DaysAllowed = 90**: Ensure this aligns with compliance requirements.
- **No validation for non-numeric custom fields**: TRY_CAST handles conversion, but invalid values default to NULL.
- **Cross join caution**: Inserts all role-course combinations where any relevant custom field is present; verify this matches business rules.

---

## 8. Notes / Freeform
- Consider extending logic to include other roles and custom fields dynamically.
- Add error handling for invalid or missing config keys.
- Could integrate with reporting dashboards to visualize training timelines by role.
