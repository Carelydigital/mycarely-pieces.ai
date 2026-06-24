# ISOZ Connector — Morela Implementation

This document covers the Morela-specific configuration of the EHR connector, including ISOZ field mappings, Odoo integration, and production details.

For general architecture, see [CONNECTOR-GUIDE.md](CONNECTOR-GUIDE.md).

---

## Overview

- **EHR System:** ISOZ (via Odoo data.stream model)
- **Tenant:** Morela
- **Project ID:** `xTkMHPv1gXRpUvn38A4tL`
- **Webhook URL:** `https://pieces.mycarely.lol/api/v1/webhooks/sc8yaVzCFIxu3iI6hVVR2`
- **Target API:** `https://mycarely.lol/api/v1/events/ingest`

---

## Odoo Integration

### Data Export
Odoo runs a scheduled action that queries `data.stream` records and POSTs them to the webhook.

```python
subtypes = ['units', 'actors_isoz', 'users_isoz', 'activities', 'patients', 'events']
last_id = env['data.stream'].send_data_to_api(webhook_url, subtype=subtypes, limit=100, last_id=last_id_r.index)
```

- Sends records ordered by ID, tracks `last_id` to resume after failures
- Each record is base64-encoded with a `type` and `data` array inside
- On failure: retries 3 times with exponential backoff, `last_id` stays at last success

### Webhook Payload Format
```json
{
  "type": "catalogue",
  "subtype": "events",
  "system": "2",
  "status": "new",
  "content": "<base64-encoded JSON>",
  "content_text": "",
  "date_from": "2026-01-01 00:00:00",
  "date_to": "2026-06-01 00:00:00"
}
```

The `content` field decodes to:
```json
{
  "type": "events",
  "data": [
    { "external_id": "107", "source_code": "SUHE", ... },
    { "external_id": "108", "source_code": "OPH", ... }
  ]
}
```

### Patient Lookup Endpoint
When events reference patients that don't exist in the catalogue, the Missing patients flow calls Odoo:

```
POST https://morela-digital.odoo.com/api/patient_lookup
{
  "jsonrpc": "2.0",
  "params": {
    "medical_indexes": ["123604", "124136"],
    "webhook_url": "https://pieces.mycarely.lol/api/v1/webhooks/sc8yaVzCFIxu3iI6hVVR2",
    "limit": 5000
  }
}
```

Odoo searches its `data.stream` records for the requested patient IDs, base64-encodes the results, and POSTs them back to the webhook. The normal ingestion pipeline then creates the patient records.

---

## ISOZ Subtypes

| Subtype | Description | Target Table | Key Field |
|---------|-------------|-------------|-----------|
| `events` | Appointment/activity events | `events` (custom) | Composite dedup key |
| `patients` | Patient demographics | `patients` (custom) | `customer_id` (medical_index) |
| `actors_isoz` | Doctors/performers | Personnel (AP Table) | `personnel_code` |
| `users_isoz` | System users/administrators | Personnel (AP Table) | `personnel_code` |
| `users_helios` | Helios system administrators | Personnel (AP Table) | `personnel_code` |
| `units` | Clinic locations/units | Locations (AP Table) | `location_code` |
| `activities` | Service/activity definitions | Activities (AP Table) | `activity_code` |

---

## Field Mappings

### Events

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| activity_external_id | event_id | Correlation ID, not unique |
| source_identifier | event_source | |
| source_code | event_code | Links to activity catalogue |
| activity_timestamp | event_created_timestamp | |
| planned_timestamp | event_planned_timestamp | |
| executed_timestamp | event_executed_timestamp | |
| activity_state | event_state | Mapped via state codes |
| external_id | customer_id | Patient reference (medical_index) |
| performer_code | doctor_code | |
| activity_user_code | administrator_code | |
| performer_unit_code | location_code | |

**Unmapped fields go to event_meta:** requestor_unit_code, requestor_unit_name, requestor_code, requestor_name, datumipl, contact_state, contact_number, doc_first_visit, zav_code, in_package, send_confirm, zav_desc, services, contact_hc

### Patients

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| medical_index | customer_id | Primary identifier |
| name_1 | customer_name | |
| name_2 | customer_name_2 | |
| surname_1 | customer_surname | |
| surname_2 | customer_surname_2 | |
| birth_date | customer_birth_date | |
| gender | customer_gender | |
| street | customer_address_street | |
| street_number | customer_address_street_2 | |
| city | customer_address_city | |
| post_code | customer_address_postcode | Trimmed of whitespace |
| country | customer_address_country | Country code (e.g. "705") |
| email | customer_emails | |
| phone + mobile | customer_phones | Deduplicated, stored as JSON array |
| my_morela | customer_contact_consent | |
| my_morela | customer_marketing_consent | |

**Meta fields:** surname_prefix, temporary_user, added_isoz_timestamp, my_morela_timestamp

### Personnel (Doctors — actors_isoz)

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| code | personnel_code | |
| name | name | |
| lastname | lastname | |
| full_name | full_name | |
| prefix | prefix | e.g. "dr. med." |
| suffix1 + suffix2 | suffix | Joined with ", " |

**Meta fields:** zzs_code, datumipl

### Personnel (Admins — users_isoz, users_helios)

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| code | personnel_code | Trimmed of whitespace |
| full_name | full_name | Split into name/lastname (first space) |
| username | (meta) | |
| valid_from | (meta) | |
| valid_to | (meta) | |

The `role` field is set to the raw subtype (`actors_isoz`, `users_isoz`, `users_helios`).

### Locations

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| code | location_code | |
| name | location_name | Also used for display_name |
| address | location_address_street | |
| city | location_address_city | |
| country | location_address_country | |
| phone | location_phones | |
| email | location_emails | |

**Meta fields:** post (postal code)

### Activities

| ISOZ Field | Unified Field | Notes |
|-----------|---------------|-------|
| code | activity_code | |
| description | activity_description | |
| duration | activity_duration | |
| type | activity_type | |

**Meta fields:** duration_patient, services

---

## Event State Mapping

| ISOZ Code | State Name |
|-----------|-----------|
| 3 | Planned |
| 5 | Executed |
| 6 | Authorized |
| 8 | Cancelled |
| C0 | Vpisovanje v CK |
| C2 | Potrjena nap. dok. |
| C3 | Vabljen |
| C5 | Sprejet v obravnavo |
| C6 | Zaključen |

---

## Data Transformations

Applied in the Build Package step when composing the unified package:

- Names are converted from FULL CAPS to Title Case (handles Slovenian characters: š, č, ž)
- `event_meta` and all `_meta` fields are parsed from JSON strings to objects
- `_emails` and `_phones` fields are parsed to arrays
- `contact_consent` and `marketing_consent` are converted to booleans ("DA" -> true)
- `event_duration_planned` is converted to a number
- `event_state` is mapped from raw codes to human-readable names
- `invoice_amount` is extracted from `event_meta.services[0].pay` (fallback: `price_inc_disc`)
- `__NONE__` sentinel values are cleaned to empty strings or null

---

## Production IDs

### Activepieces Project
- **Project ID:** `xTkMHPv1gXRpUvn38A4tL`

### Custom Postgres Tables
- `data_stream` — shared across projects (filtered by `project_id`)
- `events` — shared across projects (filtered by `project_id`)
- `patients` — shared across projects (filtered by `project_id`)
- `kv_store` — shared, keyed by `(key, project_id)`

### AP Tables (Morela-specific IDs)
| Table | Table ID |
|-------|----------|
| Patients (legacy EAV) | `B5nD21XjZdrCtzL16cVuJ` |
| Personnel | `UqFCEVPZN8bMV9yDq0oRz` |
| Locations | `lIxzfMdhz24GX1cC0Rj1v` |
| Activities | `ukJbmG354n99ZDsBnqODP` |

### External Services
| Service | URL |
|---------|-----|
| Webhook | `https://pieces.mycarely.lol/api/v1/webhooks/sc8yaVzCFIxu3iI6hVVR2` |
| mycarely API | `https://mycarely.lol/api/v1/events/ingest` |
| Odoo patient lookup | `https://morela-digital.odoo.com/api/patient_lookup` |
| Slack channel | `C09R5QNN8TY` |

---

## Known Quirks

1. **Full-caps names** — ISOZ sends all names in UPPERCASE. The package builder converts to Title Case at output time. Raw data in tables stays as-is from the EHR.

2. **Duplicate events** — The same event can arrive multiple times with different states (planned, executed, cancelled). Events are deduplicated on insert by an 11-field composite key. A periodic duplicate check runs in the weekly report.

3. **Missing patients** — Events can arrive before the patient record exists in the catalogue. These events retry until the patient appears. The Missing patients flow automatically requests unknown patients from Odoo every 10 minutes (best working temporary solution).

4. **Personnel name splitting** — `users_isoz` and `users_helios` subtypes only have `full_name` (no separate first/last name). The code splits on the first space: everything before = lastname, everything after = name. This may be incorrect for compound surnames.

5. **Empty admin/doctor codes** — Events may have empty `administrator_code` or `doctor_code`. These are not treated as missing references — the package is sent without that data.

6. **Post code whitespace** — ISOZ sends post codes with trailing spaces (e.g. "1000      "). These are trimmed during ingestion.

7. **Country codes** — ISOZ uses numeric country codes (e.g. "705" for Slovenia), not ISO alpha-2 codes.

---

## Appendix 1: Unified Package Example

The full package sent to `mycarely.lol/api/v1/events/ingest`:

```json
{
  "event_id": "410361",
  "event_source": "pr260ak0_410361",
  "event_code": "SIVMRE",
  "event_created_timestamp": "2025-11-06 13:07:00",
  "event_planned_timestamp": "2026-07-14 10:30:00",
  "event_executed_timestamp": "",
  "event_description": "SIVA MRENA - OPERACIJA KONCESIJA",
  "event_duration_planned": 15,
  "event_state": "Planned",
  "event_meta": { ... },
  "invoice_amount": 161.5,

  "contact_external_id": "119533",
  "contact_name": "Doroteja",
  "contact_surname": "Logar",
  "contact_name_2": "",
  "contact_surname_2": "",
  "contact_birth_date": "1946-09-03",
  "contact_gender": "F",
  "contact_address_street": "SEČA 83 A",
  "contact_address_street_2": "",
  "contact_address_city": "PORTOROŽ - PORTOROSE",
  "contact_address_postcode": "6320",
  "contact_address_country": "705",
  "contact_emails": ["tine_logar@t-2.net"],
  "contact_phones": ["041316729"],
  "contact_contact_consent": false,
  "contact_marketing_consent": false,
  "contact_meta": { ... },

  "doctor_external_id": "08505",
  "doctor_name": "Kristina",
  "doctor_lastname": "Mikek",
  "doctor_full_name": "Mikek Kristina",
  "doctor_prefix": "dr.med.",
  "doctor_suffix": "mag., spec. oftalmolog",
  "doctor_phones": ["01 510 23 40"],
  "doctor_emails": ["kmikek@morela.si"],
  "doctor_meta": { ... },

  "administrator_external_id": "JAN",
  "administrator_name": "Janja",
  "administrator_lastname": "Markelj",
  "administrator_full_name": "Janja Markelj",
  "administrator_prefix": "",
  "administrator_suffix": "",
  "administrator_emails": ["janjak@morela.si"],
  "administrator_phones": ["01 510 23 40"],
  "administrator_meta": { ... },

  "location_code": "OK02",
  "location_name": "OPERACIJA KATARAKTE",
  "location_display_name": "OPERACIJA KATARAKTE",
  "location_address_street": "TRR: 10100-0044831552",
  "location_address_city": "Ljubljana",
  "location_address_country": "705",
  "location_phones": [],
  "location_emails": [],
  "location_meta": { "post": null }
}
```

---

## Appendix 2: Operational Queries

All queries run via SSH on the AWS server. Replace project_id as needed.

### Queue / Redis Status
```bash
echo "=== Flow Runs ===" && docker exec postgres psql -U postgres -d activepieces -c "SELECT status, COUNT(*) FROM flow_run GROUP BY status ORDER BY count DESC;" && echo "=== Redis Queue ===" && docker exec redis redis-cli KEYS "bull:workerJobs:*" | wc -l
```

### Data Stream Stats
```bash
docker exec postgres psql -U postgres -d activepieces -c "SELECT status, subtype, COUNT(*) FROM data_stream GROUP BY status, subtype ORDER BY status, subtype;"
```

### Event Stream Stats
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT status, COUNT(*),
  SUM(CASE WHEN retry_count >= 100 THEN 1 ELSE 0 END) AS daily_retry,
  MIN(event_created_timestamp) AS oldest_event,
  MAX(event_created_timestamp) AS newest_event,
  ROUND(AVG(retry_count)::numeric, 2) AS avg_retries
FROM events
GROUP BY status
ORDER BY count DESC;
"
```

### Event Error Messages
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT error_message, COUNT(*), AVG(retry_count)::int AS avg_retries
FROM events
WHERE status = 'pending' AND error_message != ''
GROUP BY error_message
ORDER BY count DESC
LIMIT 20;
"
```

### Event Duplicate Detection
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT event_id, customer_id, event_state, event_created_timestamp, COUNT(*) AS duplicates
FROM events
GROUP BY event_id, event_code, event_state, customer_id, event_created_timestamp,
  event_source, event_planned_timestamp, event_executed_timestamp, event_description,
  event_duration_planned, doctor_code, administrator_code, location_code
HAVING COUNT(*) > 1
ORDER BY duplicates DESC
LIMIT 20;
"
```

### Missing Patient IDs (copy-pasteable list)
```bash
docker exec postgres psql -U postgres -d activepieces -t -c "
SELECT STRING_AGG('\"' || medical_index || '\"', ', ') FROM (
  SELECT DISTINCT SUBSTRING(error_message FROM 'customer_id: ([0-9]+)') AS medical_index
  FROM events
  WHERE status = 'pending' AND error_message LIKE '%customer_id%'
  AND SUBSTRING(error_message FROM 'customer_id: ([0-9]+)') IS NOT NULL
) t;
"
```

### Odoo Patient Lookups History
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT id, created_at, notes FROM data_stream WHERE notes IS NOT NULL ORDER BY id DESC LIMIT 10;
"
```

### Last 20 Flow Runs with Duration
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT
  fv.\"displayName\" AS flow,
  fr.status,
  fr.created,
  EXTRACT(EPOCH FROM (fr.updated - fr.created))::int AS seconds
FROM flow_run fr
JOIN flow_version fv ON fr.\"flowVersionId\" = fv.id
ORDER BY fr.created DESC
LIMIT 20;
"
```

### Patient Count (custom table)
```bash
docker exec postgres psql -U postgres -d activepieces -c "SELECT COUNT(*) FROM patients;"
```

### Patient Count (legacy EAV)
```bash
docker exec postgres psql -U postgres -d activepieces -c "SELECT COUNT(*) FROM record WHERE \"tableId\" = 'B5nD21XjZdrCtzL16cVuJ';"
```

### Reset Failed Events for Re-processing
```bash
docker exec postgres psql -U postgres -d activepieces -c "
UPDATE events SET retry_count = 0, error_message = '', next_daily_retry_at = NULL
WHERE status = 'pending' AND error_message LIKE '%HTTP%'
AND project_id = 'xTkMHPv1gXRpUvn38A4tL';
"
```

### Reset Stuck Processing Records
```bash
docker exec postgres psql -U postgres -d activepieces -c "UPDATE data_stream SET status = 'pending', error_message = '' WHERE status = 'processing';"
```

### Delete Duplicate Events (keep oldest)
```bash
docker exec postgres psql -U postgres -d activepieces -c "
DELETE FROM events WHERE id IN (
  SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER (
      PARTITION BY event_id, event_code, event_state, customer_id, event_created_timestamp,
        event_source, event_planned_timestamp, event_executed_timestamp,
        doctor_code, administrator_code, location_code
      ORDER BY id ASC
    ) AS rn
    FROM events
  ) t
  WHERE rn > 1
);
"
```

### Cancel Queued/Running Flow Runs
```bash
docker exec postgres psql -U postgres -d activepieces -c "UPDATE flow_run SET status = 'CANCELLED' WHERE status = 'QUEUED';"
```

### DB Disk Usage
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT relname AS table_name, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;
"
```

### Flow Run Log Sizes by Flow
```bash
docker exec postgres psql -U postgres -d activepieces -c "
SELECT fv.\"displayName\" AS flow, COUNT(f.id)::int AS logs,
  pg_size_pretty(SUM(pg_column_size(f.data))::bigint) AS compressed_size
FROM file f
JOIN flow_run fr ON fr.\"logsFileId\" = f.id
JOIN flow_version fv ON fr.\"flowVersionId\" = fv.id
WHERE f.type = 'FLOW_RUN_LOG'
GROUP BY fv.\"displayName\"
ORDER BY SUM(pg_column_size(f.data)) DESC;
"
```

### Check Active Postgres Queries
```bash
docker exec postgres psql -U postgres -d activepieces -c "SELECT pid, state, left(query, 80) AS query, now() - query_start AS duration FROM pg_stat_activity WHERE state = 'active' AND pid != pg_backend_pid();"
```

### Kill Stuck Queries
```bash
docker exec postgres psql -U postgres -d activepieces -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE query LIKE '%DELETE%' AND state = 'active' AND pid != pg_backend_pid();"
```

### Production Table IDs

**morela** (project: `xTkMHPv1gXRpUvn38A4tL`)
- Patients (legacy EAV): `B5nD21XjZdrCtzL16cVuJ`
- Personnel: `UqFCEVPZN8bMV9yDq0oRz`
- Locations: `lIxzfMdhz24GX1cC0Rj1v`
- Activities: `ukJbmG354n99ZDsBnqODP`