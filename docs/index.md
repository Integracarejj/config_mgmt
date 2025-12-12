
# Scheduled Jobs Overview

This page lists all automated scripts, their execution schedule, and purpose.

---

## 📋 Job Summary Table

| Order | Script Name               | Cron Expression   | Frequency | Purpose Summary                                                                                                                             |
| ----- | ------------------------- | ----------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | **CommLicense**           | `0 0 18,23 * * *` | Daily     | Updates SharePoint license items with DaysTillExpire and Status via Graph API.                                                              |
| 2     | **SurveyFeedGen**         | `0 0 4 * * 6`     | Weekly    | Pulls survey responses from TalentLMS; posts to SharePoint list; deduplicates and filters meaningless answers.                              |
| 3     | **SurveyFeedReports**     | `0 0 7 * * 6`     | Weekly    | Sends email notifications for survey feedback marked “Awaiting Review”; includes HTML tables and SharePoint link.                           |
| 4     | **SurveyFeedDB**          | `0 30 7 * * 6`    | Weekly    | ETL from SharePoint survey list to Azure SQL; updates DimSurveyFeed; logs status changes; snapshots for trend analysis.                     |
| 5     | **RoleCatSuper**          | `0 0 16 * * 6`    | Weekly    | ETL for TalentLMS categories and roles; refreshes HR staging; normalizes names; updates DimSupervisor and DimUser supervisor relationships. |
| 6     | **DimUser**               | `0 30 16 * * 6`   | Weekly    | Ingests TalentLMS users; enriches with HR data; upserts into DimUser; applies manual overrides.                                             |
| 7     | **FactEnrollment1**       | `0 17 * * 6`      | Weekly    | Pulls course enrollment data for each user from TalentLMS; populates SQL database.                                                          |
| 8     | **DimCourse_FactCourse2** | `30 17 * * 6`     | Weekly    | Ingests TalentLMS courses; updates DimCourse; syncs FactCourseDevelopment for reporting.                                                    |
| 9     | **CourseRoleDaysHelper**  | `0 0 18 * * 6`    | Weekly    | Builds CourseRoleDays mapping table dynamically from DimCourse and DimRole.                                                                 |

---

## 🔍 Details for Each Script

### 1. CommLicense
- **Runs:** Daily at 18:00 and 23:00
- **Purpose:** Updates SharePoint license items with `DaysTillExpire` and `Status` using Graph API.

### 2. SurveyFeedGen
- **Runs:** Weekly on Saturday at 04:00
- **Purpose:** Pulls survey responses from TalentLMS, posts to SharePoint list, deduplicates, and filters meaningless answers.

### 3. SurveyFeedReports
- **Runs:** Weekly on Saturday at 07:00
- **Purpose:** Sends email notifications for survey feedback marked “Awaiting Review”; includes HTML tables and SharePoint link.

### 4. SurveyFeedDB
- **Runs:** Weekly on Saturday at 07:30
- **Purpose:** ETL from SharePoint survey list to Azure SQL; updates DimSurveyFeed; logs status changes; snapshots for trend analysis.

### 5. RoleCatSuper
- **Runs:** Weekly on Saturday at 16:00
- **Purpose:** ETL for TalentLMS categories and roles; refreshes HR staging; normalizes names; updates DimSupervisor and DimUser supervisor relationships.

### 6. DimUser
- **Runs:** Weekly on Saturday at 16:30
- **Purpose:** Ingests TalentLMS users; enriches with HR data; upserts into DimUser; applies manual overrides.

### 7. FactEnrollment1
- **Runs:** Weekly on Saturday at 17:00
- **Purpose:** Pulls course enrollment data for each user from TalentLMS; populates SQL database.

### 8. DimCourse_FactCourse2
- **Runs:** Weekly on Saturday at 17:30
- **Purpose:** Ingests TalentLMS courses; updates DimCourse; syncs FactCourseDevelopment for reporting.

### 9. CourseRoleDaysHelper
- **Runs:** Weekly on Saturday at 18:00
- **Purpose:** Builds CourseRoleDays mapping table dynamically from DimCourse and DimRole.

---

## ✅ Notes
- All times are in **UTC** unless otherwise specified.
- Cron expressions follow the standard format: `minute hour day month weekday`.

---

``
