
# CommLicense Azure Function

This function:
- Fetches items from a SharePoint list via Microsoft Graph.
- Upserts data into an Azure SQL Database.
- Includes retry logic for transient SQL errors (e.g., 40613).

---

## Full Python Script

```python
import logging
import json
import requests
import pymssql
from datetime import datetime
import os
import azure.functions as func

# NEW imports for retry/jitter
import time
import random

# ------------------------------
# Hardcoded SharePoint site/list info
# ------------------------------
SITE_ID = "integracare.sharepoint.com,b211e7da-ecb7-4ce4-8752-1174c21313e7,cd65ec56-38dd-40e2-b08a-a77716603c9f"
LIST_ID = "ee823721-1b2e-44df-880c-0337b8588b7c"

# ------------------------------
# Load secrets from environment variables (defensive)
# ------------------------------
def _get_env_required(name: str) -> str:
    val = os.getenv(name)
    if not val:
        raise RuntimeError(f"Required app setting '{name}' is missing. "
                           f"Check Function App > Configuration (Linux is case-sensitive).")
    return val

# Use the exact casing as in your Function App settings
CLIENT_ID = os.getenv("AppRegClientID")
TENANT_ID = os.getenv("AppRegTenantID")
CLIENT_SECRET = os.getenv("AppRegSecretValue")

SQL_SERVER = _get_env_required("DB_Server")
SQL_DATABASE = _get_env_required("DB_Name")
SQL_USER = _get_env_required("DB_User")
SQL_PASSWORD = _get_env_required("DB_Pass")

# ------------------------------
# Helpers
# ------------------------------
def safe_value(val, field_type="str"):
    if val is None:
        return None
    if field_type == "int":
        try:
            return int(val)
        except:
            return None
    if field_type == "float":
        try:
            return float(val)
        except:
            return None
    if isinstance(val, dict) or isinstance(val, list):
        return json.dumps(val)
    if isinstance(val, bool):
        return int(val)
    if isinstance(val, str):
        return val.strip()
    return val

def parse_datetime(val):
    if not val:
        return None
    try:
        return datetime.fromisoformat(val.replace("Z", ""))
    except:
        return None

# ------------------------------
# Get Microsoft Graph token
# ------------------------------
def get_access_token():
    url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    payload = {
        "client_id": CLIENT_ID,
        "scope": "https://graph.microsoft.com/.default",
        "client_secret": CLIENT_SECRET,
        "grant_type": "client_credentials"
    }
    r = requests.post(url, data=payload)
    r.raise_for_status()
    logging.info("Obtained Microsoft Graph access token")
    return r.json()["access_token"]

# ------------------------------
# Fetch all SharePoint list items
# ------------------------------
def get_list_items(access_token):
    items = []
    url = f"https://graph.microsoft.com/v1.0/sites/{SITE_ID}/lists/{LIST_ID}/items?expand=fields&$top=999"
    headers = {"Authorization": f"Bearer {access_token}"}

    while url:
        r = requests.get(url, headers=headers)
        r.raise_for_status()
        data = r.json()
        items.extend(data.get("value", []))
        url = data.get("@odata.nextLink")

    logging.info(f"Fetched {len(items)} items from SharePoint list")
    return [i["fields"] for i in items]

# ------------------------------
# Resilient Azure SQL connection with exponential backoff
# ------------------------------
TRANSIENT_SQL_ERROR_CODES = {40613, 40197, 40501, 10928, 10929, 11001}

def _extract_sql_error_code(exc: Exception):
    try:
        if exc.args and isinstance(exc.args[0], int):
            return exc.args[0]
    except Exception:
        pass
    return None

def connect_sql_with_retry(
    server: str,
    user: str,
    password: str,
    database: str,
    max_attempts: int = 6,
    base_delay_seconds: float = 2.0
):
    for attempt in range(1, max_attempts + 1):
        try:
            conn = pymssql.connect(
                server=server,
                user=user,
                password=password,
                database=database,
                charset="UTF-8",
                login_timeout=15,
                timeout=30
            )
            logging.info("SQL connection established.")
            return conn
        except Exception as e:
            code = _extract_sql_error_code(e)
            should_retry = (code in TRANSIENT_SQL_ERROR_CODES) or isinstance(e, pymssql.OperationalError)
            delay = base_delay_seconds * (2 ** (attempt - 1)) + random.uniform(0, 0.5)

            if attempt < max_attempts and should_retry:
                logging.warning(f"Transient SQL connect error (attempt {attempt}/{max_attempts}, code={code}). Retrying in {delay:.1f}s. Error: {e}")
                time.sleep(delay)
                continue

            logging.error(f"SQL connect failed (final attempt {attempt}, code={code}): {e}", exc_info=True)
            raise

# ------------------------------
# Fully optimized upsert
# ------------------------------
def upsert_items(items):
    conn = connect_sql_with_retry(
        server=SQL_SERVER,
        user=SQL_USER,
        password=SQL_PASSWORD,
        database=SQL_DATABASE
    )
    cursor = conn.cursor(as_dict=True)

    cursor.execute("SELECT * FROM dbo.DimCommLicense")
    existing_rows = {row["id"]: row for row in cursor.fetchall()}

    inserts = 0
    updates = 0

    for f in items:
        row = {
            "id": safe_value(f.get("id"), "int"),
            "Title": safe_value(f.get("Title")),
            "State": safe_value(f.get("field_1")),
            "LIc Occupany (PC/SL/AL)": safe_value(f.get("field_2"), "float"),
            "Lic Occupancy (MC)": safe_value(f.get("field_3"), "float"),
            "Status": safe_value(f.get("field_4")),
            "Licensure number": safe_value(f.get("field_5"), "int"),
            "Facility Address 1": safe_value(f.get("field_7")),
            "Facility City/State": safe_value(f.get("field_8")),
            "Facility Zipcode": safe_value(f.get("field_9"), "float"),
            "Facility Tax ID": safe_value(f.get("field_10")),
            "Taxpayer ID": safe_value(f.get("field_11")),
            "LegalName": safe_value(f.get("LegalName")),
            "ExpireDate": parse_datetime(f.get("ExpireDate")),
            "DaysTillExpire": safe_value(f.get("DaysTillExpire"), "float"),
            "RenewalStage": safe_value(f.get("RenewalStage")),
            "LicenseInspectionSummary": safe_value(f.get("LicenseInspectionSummary")),
            "Certificate": safe_value(f.get("Certificate")),
            "CommunityDocs": safe_value(f.get("CommunityDocs")),
            "ContentType": safe_value(f.get("ContentType")),
            "Modified": parse_datetime(f.get("Modified")),
            "Created": parse_datetime(f.get("Created")),
            "AuthorLookupId": safe_value(f.get("AuthorLookupId"), "int"),
            "EditorLookupId": safe_value(f.get("EditorLookupId"), "int"),
            "_UIVersionString": safe_value(f.get("_UIVersionString"), "float"),
            "Attachments": safe_value(f.get("Attachments"), "int"),
            "Edit": safe_value(f.get("Edit")),
            "LinkTitleNoMenu": safe_value(f.get("LinkTitleNoMenu")),
            "ItemChildCount": safe_value(f.get("ItemChildCount"), "int"),
            "FolderChildCount": safe_value(f.get("FolderChildCount"), "int"),
            "_ComplianceFlags": safe_value(f.get("_ComplianceFlags")),
            "_ComplianceTag": safe_value(f.get("_ComplianceTag")),
            "_ComplianceTagWrittenTime": parse_datetime(f.get("_ComplianceTagWrittenTime")),
            "_ComplianceTagUserId": safe_value(f.get("_ComplianceTagUserId"), "int"),
            "AppAuthorLookupId": safe_value(f.get("AppAuthorLookupId"), "int"),
            "AppEditorLookupId": safe_value(f.get("AppEditorLookupId"), "int")
        }

        existing = existing_rows.get(row["id"])

        if existing:
            changed = False
            for col, val in row.items():
                if col == "id":
                    continue
                existing_val = existing.get(col)
                if isinstance(val, datetime) and isinstance(existing_val, datetime):
                    if val != existing_val:
                        changed = True
                        break
                elif val != existing_val:
                    changed = True
                    break

            if changed:
                set_clause = ", ".join([f"[{col}] = %s" for col in row if col != "id"])
                values = [row[col] for col in row if col != "id"]
                values.append(row["id"])
                update_sql = f"UPDATE dbo.DimCommLicense SET {set_clause} WHERE id = %s"
                cursor.execute(update_sql, values)
                updates += 1
        else:
            cols = ", ".join([f"[{col}]" for col in row])
            placeholders = ", ".join(["%s"] * len(row))
            insert_sql = f"INSERT INTO dbo.DimCommLicense ({cols}) VALUES ({placeholders})"
            cursor.execute(insert_sql, list(row.values()))
            inserts += 1

    conn.commit()
    cursor.close()
    conn.close()
    logging.info(f"Inserted {inserts} new rows, updated {updates} rows.")

# ------------------------------
# Azure Function entry point
# ------------------------------
def main(mytimer: func.TimerRequest) -> None:
    logging.info("CommLicenseFunction started")
    try:
        token = get_access_token()
        items = get_list_items(token)
        upsert_items(items)
        logging.info("CommLicenseFunction completed successfully")
    except Exception as e:
        logging.error        logging.error(f"Function failed: {e}", exc_info=True)
