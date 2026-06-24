# EHR Connector — General Implementation Guide

This document describes the architecture, flow patterns, and infrastructure used to connect EHR systems to the mycarely platform via Activepieces.

---

## Architecture Overview

```
EHR --->  Webhook  --->  data_stream  --->  EHR Ingest  --->  Catalogue Tables
                                                                    |
                                                              events table
                                                                    |
                                                    Sync data (Live push / Daily retry)
                                                                    |
                                                            mycarely.lol API
```

**Data flows in 3 stages:**

1. **Ingestion** — EHR sends base64-encoded payloads to a webhook. The webhook stores them in the `data_stream` table as pending records.
2. **Processing** — A scheduled flow reads pending data_stream records, decodes them, routes by subtype (patients, events, personnel, locations, activities), and upserts into catalogue/event tables.
3. **Sync** — A scheduled flow reads pending events, looks up all referenced catalogue data (patient, doctor, admin, location, activity), composes a unified package, and POSTs it to the mycarely API. Failed events retry with exponential backoff.

**Multi-tenancy:** Each tenant (clinic/customer) gets its own Activepieces project. Custom Postgres tables (`data_stream`, `events`, `patients`, `kv_store`) are shared across projects and isolated by `project_id`. AP Tables (personnel, locations, activities) are scoped by `tableId` — each project has its own set.

---

## Project Folder Structure

```
Project (e.g. morela)
├── Data stream API              — webhook receiver
├── EHR Ingest Data              — processes data_stream into catalogue tables
├── Sync data - Live push        — sends events to mycarely API
├── Sync data - Daily retry      — retries events with 24h delay
├── Helper flows/
│   ├── Missing patients Odoo request  — auto-requests missing patients from EHR
│   ├── Delete redundant flow logs     — daily cleanup of logs and old data
│   ├── Migrate patients table         — one-time migration tool (manual trigger)
│   └── Patients initial import        — one-time bulk import tool
└── Slack notifications/
    ├── Flow Failure Live Reporting     — real-time failure alerts
    └── Weekly Connector Report         — weekly summary report
```

Core flows sit at the project root. Helper flows and Slack notifications are grouped in subfolders.

---

## Flow Patterns

### 1. Data stream API (Webhook)
**Trigger:** Webhook (HTTP POST)
**Purpose:** Receives base64-encoded payloads from the EHR and stores them in `data_stream`.
**Logic:** Single code step — inserts subtype, content, project_id, dates, and notes.

### 2. EHR Ingest Data (Scheduled, every 1 min)
**Trigger:** Schedule
**Purpose:** Processes pending data_stream records into catalogue tables.
**Logic:**
- Finds pending records (smallest subtype queue first, events last)
- Batch size: 500 data_stream records per run
- Routes by subtype: events, patients, personnel, locations, activities
- Each branch maps EHR-specific fields to the unified schema and upserts
- Marks data_stream records as processed/failed
- Patients and events use custom Postgres tables (batch SQL upserts)
- Personnel, locations, activities use AP Tables (EAV batch SQL upserts)

### 3. Sync data — Live push (Scheduled, every 5 min)
**Trigger:** Schedule
**Purpose:** Sends pending events to the mycarely API.
**Logic:**
- Fetches up to 100 pending events (retry_count < 100)
- For each event: looks up patient, doctor, admin, location, activity
- Composes unified package with all referenced data
- POSTs to the mycarely events ingest API
- On success: marks as `sent`
- On HTTP error: increments retry_count, saves error message
- On missing references: increments retry_count, saves which references are missing
- References with null/empty codes are skipped (not treated as missing)

### 4. Sync data — Daily retry (Scheduled, every 5 min)
**Trigger:** Schedule
**Purpose:** Retries events that exhausted the live push budget.
**Logic:** Same as Live push, but:
- Only picks up events with retry_count >= 100 and < 130
- Respects `next_daily_retry_at` — skips events scheduled for the future
- On failure: sets `next_daily_retry_at = NOW() + 1 day`
- After 130 retries (~30 days of daily retries): events stop being picked up

### 5. Missing patients request (Scheduled, every 10 min)
**Trigger:** Schedule
**Purpose:** Automatically finds patient IDs that events are waiting for and requests them from the EHR.
**Logic:**
- Queries events for distinct customer_ids in error messages
- Calls the EHR's patient lookup endpoint with the missing IDs
- The EHR looks up the patients and sends them back via the webhook
- The webhook -> data_stream -> EHR Ingest pipeline creates the patient records
- Next sync run finds the patient and sends the event successfully

### 6. Delete redundant flow logs (Daily, 03:00 UTC)
**Trigger:** Schedule
**Purpose:** Cleans up flow run logs and old data to prevent disk bloat.
**Logic:**
- Deletes all Data stream API flow run logs (they contain full base64 payloads)
- Deletes webhook payloads older than 3 days
- Deletes processed data_stream records older than 14 days
- Runs VACUUM on affected tables

### 7. Flow Failure Live Reporting (Scheduled, every 5 min)
**Trigger:** Schedule
**Purpose:** Sends Slack alerts for failed production flow runs.
**Logic:**
- Tracks last check timestamp in `kv_store` (per project)
- On first run or after a gap > 5 min, resets checkpoint (prevents flood of old errors)
- Queries for FAILED, TIMEOUT, MEMORY_LIMIT_EXCEEDED, INTERNAL_ERROR, LOG_SIZE_EXCEEDED runs
- Includes health checks with 3-consecutive-failure threshold:
  - Public URL reachability (Cloudflare/DNS/SSL)
  - Internal API reachability (Docker/app container)
  - Disk usage > 90%

### 8. Weekly Connector Report (Every Monday, 06:00 UTC)
**Trigger:** Schedule
**Purpose:** Comprehensive weekly Slack report.
**Covers:** Event counts, daily breakdown, top errors, retry queue, duplicates, data stream throughput by subtype, processing times, table sizes, disk usage, flow failures.

---

## Retry Logic

```
Event created (status: pending, retry_count: 0)
    |
    v
Live push attempts (every 5 min, up to 100 retries)
    |
    v  (retry_count reaches 100)
    |
Daily retry (every 5 min, checks next_daily_retry_at)
    - On failure: next_daily_retry_at = NOW() + 1 day
    - Up to 130 total retries (~30 days)
    |
    v  (retry_count reaches 130)
    |
Event stays pending but is no longer picked up
```

---

## Database Tables

### Custom Postgres Tables

These bypass the Activepieces EAV system for performance at scale. Shared across projects, scoped by `project_id`.

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `data_stream` | Incoming webhook payloads | `id`, `subtype`, `content`, `status`, `project_id` |
| `events` | Event records with retry tracking | `id`, `event_id`, `customer_id`, `status`, `retry_count`, `project_id` |
| `patients` | Patient catalogue | `id`, `customer_id` (unique per project), `project_id` |
| `kv_store` | Flow state (timestamps, counters) | `key`, `value`, `project_id` (composite PK) |

### Activepieces EAV Tables (Small Catalogues)

Each project has its own set. Used for small datasets that benefit from the AP Tables UI.

| Table | Key Field | ~Records |
|-------|-----------|----------|
| Personnel | personnel_code | ~1000 |
| Locations | location_code | ~30 |
| Activities | activity_code | ~1700 |

**Note:** EAV queries require joining `record` -> `cell` -> `field`. An index on `cell ("fieldId", left("value", 255))` is required for performance.

---

## Operations

### Multi-Tenant Rules
- Every query on `data_stream`, `events`, `patients` MUST filter by `project_id`
- `kv_store` uses `(key, project_id)` composite primary key
- Flow failure reporting and weekly reports are scoped by `projectId` input
- Log cleanup is intentionally global (cleans all projects)

### Automated Maintenance (Delete redundant flow logs, daily)
- Deletes Data stream API flow run logs + webhook payloads older than 3 days
- Deletes processed data_stream records older than 14 days
- Runs VACUUM on affected tables

### Manual Maintenance
- `VACUUM FULL <table>` — reclaims disk space after large deletes (locks the table)
- `VACUUM FULL pg_toast.<toast_table>` — reclaims TOAST storage for large text columns

### Configuration
- **DB password:** `process.env.AP_POSTGRES_PASSWORD` (propagated via `AP_SANDBOX_PROPAGATED_ENV_VARS`)
- **Memory limit:** `AP_SANDBOX_MEMORY_LIMIT` in `.env` (KB). Default 1GB, set to 4GB for large batches
- **Flow timeout:** `AP_FLOW_TIMEOUT_SECONDS=600` (10 minutes)
- **Queue mode:** `AP_QUEUE_MODE=REDIS` required for separate app/worker containers
- **Worker frontend URL:** Must be `http://activepieces-app:80` (internal Docker URL), not the public URL

---

## Adding a New EHR Connector

Each tenant gets its own Activepieces project. Custom Postgres tables are shared and scoped by `project_id`.

1. **Create an Activepieces project** — in the platform admin UI, create a new team project for the tenant. Note the project ID.
2. **Create catalogue tables** — run `scripts/create-ehr-tables.sh <base_url> <project_id> <token>` for AP Tables (personnel, locations, activities). Custom Postgres tables already exist.
3. **Create a webhook flow** — Data stream API in the new project, inserting into `data_stream` with the project's ID.
4. **Create the EHR Ingest flow** — scheduled every 1 min. Add router branches for the new EHR's subtypes and write field mapping code steps.
5. **Configure the EHR** — point its data export to the webhook URL.
6. **Create sync flows** — clone Live push and Daily retry. Configure `projectId`, API tokens, target API URL.
7. **Create helper flows** — clone Missing patients request and Delete redundant flow logs. Configure `projectId`.
8. **Create monitoring flows** — clone Flow Failure Live Reporting and Weekly Connector Report. Configure `projectId`, `publicUrl`, Slack channel.

---

## Appendix: Full Table Schemas

### data_stream

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL PK | Auto-increment ID |
| subtype | TEXT | Data type: patients, events, actors_isoz, units, etc. |
| content | TEXT | Base64-encoded JSON payload |
| status | TEXT | pending / processing / processed / failed |
| error_message | TEXT | Error details if failed |
| created_at | TIMESTAMP | When the webhook received it |
| processing_started | TIMESTAMP | When a flow started processing it |
| processed_at | TIMESTAMP | When processing completed |
| project_id | TEXT | Activepieces project ID (multi-tenant) |
| date_from | TEXT | Optional date range from EHR |
| date_to | TEXT | Optional date range from EHR |
| notes | TEXT | Optional metadata (e.g. "patient_lookup: 123, 456") |

Indexes: `status`, `subtype`, `(status, subtype)`

### events

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL PK | Auto-increment ID |
| event_id | TEXT | External correlation ID from EHR |
| event_source | TEXT | Source identifier |
| event_code | TEXT | Activity/service code |
| event_created_timestamp | TEXT | When the EHR created the event |
| event_planned_timestamp | TEXT | Scheduled datetime |
| event_executed_timestamp | TEXT | Actual execution datetime |
| event_description | TEXT | Populated from activity lookup |
| event_duration_planned | TEXT | Populated from activity lookup |
| event_state | TEXT | Raw state code from EHR |
| event_meta | TEXT | JSON blob of unmapped fields |
| customer_id | TEXT | Patient reference |
| doctor_code | TEXT | Doctor reference |
| administrator_code | TEXT | Admin reference |
| location_code | TEXT | Location reference |
| status | TEXT | pending / sent / failed |
| retry_count | INT | Number of send attempts |
| error_message | TEXT | Last error |
| ingested_at | TIMESTAMP | When ingested from data_stream |
| created_at | TIMESTAMP | Record creation |
| updated_at | TIMESTAMP | Last update |
| project_id | TEXT | Multi-tenant scope |
| next_daily_retry_at | TIMESTAMP | Next daily retry (for events with retry_count >= 100) |

Indexes: `status`, `customer_id`, `(status, retry_count)`

Dedup key: event_id + event_code + event_state + customer_id + event_created_timestamp + event_source + event_planned_timestamp + event_executed_timestamp + doctor_code + administrator_code + location_code

### patients

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL PK | Auto-increment ID |
| customer_id | TEXT | Patient identifier from EHR |
| customer_name | TEXT | First name |
| customer_name_2 | TEXT | Middle/second name |
| customer_surname | TEXT | Last name |
| customer_surname_2 | TEXT | Second surname |
| customer_birth_date | TEXT | Date of birth |
| customer_gender | TEXT | Gender |
| customer_address_street | TEXT | Street address |
| customer_address_street_2 | TEXT | Street line 2 |
| customer_address_city | TEXT | City |
| customer_address_postcode | TEXT | Postal code |
| customer_address_country | TEXT | Country code |
| customer_emails | TEXT | Email(s) |
| customer_phones | TEXT | Phone(s) as JSON array |
| customer_contact_consent | TEXT | Contact consent |
| customer_marketing_consent | TEXT | Marketing consent |
| customer_meta | TEXT | JSON blob of extra fields |
| project_id | TEXT | Multi-tenant scope |
| created_at | TIMESTAMP | Record creation |
| updated_at | TIMESTAMP | Last update |

Indexes: `UNIQUE (customer_id, project_id)`, `project_id`
