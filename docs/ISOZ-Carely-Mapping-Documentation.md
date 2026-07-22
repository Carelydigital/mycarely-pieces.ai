# EHR Live Push Mapping Quick Reference

General cleanup:

- Empty strings and `null` are normalized internally and sent as empty strings, empty arrays, empty objects, or `null` depending on the target field.
- Codes/postcodes/countries are trimmed where the flow explicitly does so.
- JSON strings are parsed before sending. Invalid or missing JSON becomes `{}`.
- Array-like values are parsed from JSON where possible; plain non-empty strings become single-item arrays.

## EHR Ingest: Events

| Source attribute | Internal attribute |
| --- | --- |
| `activity_external_id` | `event_id` |
| `source_identifier` | `event_source` |
| `source_code` | `event_code` |
| `activity_timestamp` | `event_created_timestamp` |
| `planned_timestamp` | `event_planned_timestamp` |
| `executed_timestamp` | `event_executed_timestamp` |
| `activity_state` | `event_state` |
| `external_id` | `customer_id` |
| `performer_code` | `doctor_code` |
| `activity_user_code` | `administrator_code` |
| `performer_unit_code` | `location_code` |

Moved into `event_meta`:

| Source attribute | Internal attribute |
| --- | --- |
| `requestor_unit_code` | `event_meta.requestor_unit_code` |
| `requestor_unit_name` | `event_meta.requestor_unit_name` |
| `requestor_code` | `event_meta.requestor_code` |
| `requestor_name` | `event_meta.requestor_name` |
| `datumipl` | `event_meta.datumipl` |
| `contact_state` | `event_meta.contact_state` |
| `contact_number` | `event_meta.contact_number` |
| `doc_first_visit` | `event_meta.doc_first_visit` |
| `zav_code` | `event_meta.zav_code` |
| `in_package` | `event_meta.in_package` |
| `send_confirm` | `event_meta.send_confirm` |
| `zav_desc` | `event_meta.zav_desc` |
| `services` | `event_meta.services` |
| `contact_hc` | `event_meta.contact_hc` |

Value mappings:

| Value/source | Result |
| --- | --- |
| new ingested event | `status = "pending"` |
| new ingested event | `retry_count = 0` |
| missing `executed_timestamp` | `event_executed_timestamp = null` |

## EHR Ingest: Patients

| Source attribute | Internal attribute |
| --- | --- |
| `medical_index` | `customer_id` |
| `name_1` | `customer_name` |
| `name_2` | `customer_name_2` |
| `surname_1` | `customer_surname` |
| `surname_2` | `customer_surname_2` |
| `birth_date` | `customer_birth_date` |
| `gender` | `customer_gender` |
| `street` | `customer_address_street` |
| `street_number` | `customer_address_street_2` |
| `city` | `customer_address_city` |
| `post_code` | `customer_address_postcode` |
| `country` | `customer_address_country` |
| `email` | `customer_emails` |
| unique `[phone, mobile]` | `customer_phones` |
| `my_morela` | `customer_marketing_consent` |

Moved into `customer_meta`:

| Source attribute | Internal attribute |
| --- | --- |
| `surname_prefix` | `customer_meta.surname_prefix` |
| `temporary_user` | `customer_meta.temporary_user` |
| `added_isoz_timestamp` | `customer_meta.added_isoz_timestamp` |
| `my_morela_timestamp` | `customer_meta.my_morela_timestamp` |

Value mappings:

| Value/source | Result |
| --- | --- |
| every ingested patient | `customer_contact_consent = "true"` |
| `phone` and `mobile` | duplicates removed, stored as JSON array |

## EHR Ingest: Personnel

| Source attribute | Internal attribute |
| --- | --- |
| `code` | `personnel_code` |
| incoming subtype | `role` |
| `name` | `name` |
| `lastname` | `lastname` |
| `full_name` | `full_name` |
| `prefix` | `prefix` |
| `suffix1` + `suffix2` | `suffix` |

Moved into `meta`:

| Source attribute | Internal attribute |
| --- | --- |
| `zzs_code` | `meta.zzs_code` |
| `datumipl` | `meta.datumipl` |
| `username` | `meta.username` |
| `valid_from` | `meta.valid_from` |
| `valid_to` | `meta.valid_to` |

Value mappings:

| Value/source | Result |
| --- | --- |
| missing `name` and `lastname`, with multi-part `full_name` | first token -> `name`, remaining tokens -> `lastname` |
| missing `name` and `lastname`, with single-part `full_name` | `full_name` -> `lastname` |
| `suffix1`, `suffix2` | non-empty values joined with `, ` |
| no image/phone/email data | `image_url = null`, `phones = null`, `emails = null` |

## EHR Ingest: Locations

| Source attribute | Internal attribute |
| --- | --- |
| `code` | `location_code` |
| `name` | `location_name` |
| `name` | `location_display_name` |
| `address` | `location_address_street` |
| `city` | `location_address_city` |
| `country` | `location_address_country` |
| `phone` | `location_phones` |
| `email` | `location_emails` |
| `post` | `location_meta.post` |

## EHR Ingest: Activities

| Source attribute | Internal attribute |
| --- | --- |
| `code` | `activity_code` |
| `description` | `activity_description` |
| `duration` | `activity_duration` |
| `type` | `activity_type` |
| `duration_patient` | `activity_meta.duration_patient` |
| `services` | `activity_meta.services` |

## Live Push: Event Package

| Internal attribute | Outgoing attribute |
| --- | --- |
| `event_id` | `event_id` |
| constant `"ehr"` | `event_source` |
| `event_code` | `event_code` |
| `event_created_timestamp` | `event_created_timestamp` |
| `event_planned_timestamp` | `event_planned_timestamp` |
| `event_executed_timestamp` | `event_executed_timestamp` |
| `activity.activity_description` | `event_description` |
| `activity.activity_duration` | `event_duration_planned` |
| `event_state` | `event_state` |
| `event_meta` | `event_meta` |
| `event_meta.services[0].pay` | `invoice_amount` |
| `event_meta.services[0].price_inc_disc` | `invoice_amount`, fallback when `pay` is missing |

Value mappings:

| Source value | Outgoing value |
| --- | --- |
| `event_state = "3"` | `Planned` |
| `event_state = "5"` | `Executed` |
| `event_state = "6"` | `Authorized` |
| `event_state = "8"` | `Cancelled` |
| `event_state = "C0"` | `Vpisovanje v CK` |
| `event_state = "C2"` | `Potrjena nap. dok.` |
| `event_state = "C3"` | `Vabljen` |
| `event_state = "C5"` | `Sprejet v obravnavo` |
| `event_state = "C6"` | `Zaključen` |
| unknown `event_state` | original cleaned value |
| `YYYY-MM-DD HH:MM:SS` timestamps | UTC ISO strings, interpreted from `Europe/Ljubljana` |
| date-like strings inside metadata | recursively converted to UTC ISO strings |

## Live Push: Contact Package

| Internal attribute | Outgoing attribute |
| --- | --- |
| `customer_id` | `contact_external_id` |
| `customer_name` | `contact_name` |
| `customer_name_2` | `contact_name_2` |
| `customer_surname` | `contact_surname` |
| `customer_surname_2` | `contact_surname_2` |
| `customer_birth_date` | `contact_birth_date` |
| `customer_gender` | `contact_gender` |
| `customer_address_street` | `contact_address_street` |
| `customer_address_street_2` | `contact_address_street_2` |
| `customer_address_city` | `contact_address_city` |
| `customer_address_postcode` | `contact_address_postcode` |
| `customer_address_country` | `contact_address_country` |
| `customer_emails` | `contact_emails` |
| `customer_phones` | `contact_phones` |
| `customer_contact_consent` | `contact_contact_consent` |
| `customer_marketing_consent` | `contact_marketing_consent` |
| `customer_meta` | `contact_meta` |

Value mappings:

| Source value | Outgoing value |
| --- | --- |
| names and surnames | title-cased |
| `customer_address_country` | converted with country-code mapping below |
| `customer_contact_consent = true`, `"true"`, or `"DA"` | `true` |
| other `customer_contact_consent` values | `false` |
| `customer_marketing_consent = true`, `"true"`, or `"DA"` | `true` |
| other `customer_marketing_consent` values | `false` |

## Live Push: Doctor Package

| Internal attribute | Outgoing attribute |
| --- | --- |
| `personnel_code` | `doctor_external_id` |
| `name` | `doctor_name` |
| `lastname` | `doctor_lastname` |
| `full_name` | `doctor_full_name` |
| `prefix` | `doctor_prefix` |
| `suffix` | `doctor_suffix` |
| `phones` | `doctor_phones` |
| `emails` | `doctor_emails` |
| `meta` | `doctor_meta` |

Value mappings:

| Source value | Outgoing value |
| --- | --- |
| doctor names | title-cased |
| date-like strings inside `meta` | UTC ISO strings, interpreted from `Europe/Ljubljana` |

## Live Push: Administrator Package

| Internal attribute | Outgoing attribute |
| --- | --- |
| `personnel_code` | `administrator_external_id` |
| `name` | `administrator_name` |
| `lastname` | `administrator_lastname` |
| `full_name` | `administrator_full_name` |
| `prefix` | `administrator_prefix` |
| `suffix` | `administrator_suffix` |
| `emails` | `administrator_emails` |
| `phones` | `administrator_phones` |
| `meta` | `administrator_meta` |

Value mappings:

| Source value | Outgoing value |
| --- | --- |
| administrator names | title-cased |
| date-like strings inside `meta` | UTC ISO strings, interpreted from `Europe/Ljubljana` |

## Live Push: Location Package

| Internal attribute | Outgoing attribute |
| --- | --- |
| `location_code` | `location_code` |
| `location_name` | `location_name` |
| `location_display_name` | `location_display_name` |
| `location_address_street` | `location_address_street` |
| `location_address_city` | `location_address_city` |
| `location_address_country` | `location_address_country` |
| `location_phones` | `location_phones` |
| `location_emails` | `location_emails` |
| `location_meta` | `location_meta` |

Value mappings:

| Source value | Outgoing value |
| --- | --- |
| `location_address_country` | converted with country-code mapping below |
| date-like strings inside `location_meta` | UTC ISO strings, interpreted from `Europe/Ljubljana` |

## Country Code Mapping

| Source value | Outgoing value |
| --- | --- |
| `40` | `AT` |
| `380` | `IT` |
| `705` | `SI` |
| ... | ... |
| already-known abbreviation | uppercased abbreviation |
| unknown country value | original cleaned value |

## Send Status

| Condition | Internal result |
| --- | --- |
| live-push POST returns `2xx` | `events.status = "sent"` |
| live-push POST does not return `2xx` | `events.status = "pending"`, retry count increments |
| required patient/doctor/admin/location reference is missing | `events.status = "pending"`, retry count increments |
