---
name: cambio-patient-clinical-summary
description: Assemble a read-only clinical summary for one patient from Cambio COSMIC via Cambio Open Services — demographics, attention signals, diagnoses, medications, lab results, journal notes and care contacts.
api: Cambio Open Services (COSMIC REST)
base_url: https://api.openservices.cambio.se
operations:
  - get-patient-information-v2
  - get-attention-signals
  - get-diagnoses-v2
  - getPrescriptionInfo-v2
  - get-chemistry-lab-results-v2
  - get-journal-notes-v2
  - get-contacts-v2
generated: '2026-09-02'
method: generated
source: openapi/ (provider-published operations exported from developer.openservices.cambio.se)
---

# Patient clinical summary from Cambio COSMIC

Read-only. Every call below is a `GET`. Nothing in this skill writes to the health record.

## Before you start

1. You need **both** credentials, on every request:
   - `Ocp-Apim-Subscription-Key: <APIM subscription key>` — from your COS portal profile.
   - `Authorization: Bearer <token>` — from `POST https://api.openservices.cambio.se/auth/realms/COS/protocol/openid-connect/token` (`client_credentials` or `authorization_code`).
2. The token must carry the SMART-on-FHIR scope each operation needs, e.g. `user/Patient.read`, `user/Contact.read`, `user/Diagnosis.read`, `user/JournalNote.read`, `user/AttentionSignal.read`, `user/Medication.read`. A missing scope returns **403 Insufficient scope**, not 401.
3. The patient identifier goes in a **`patient` request header**, not in the path. Every operation below requires it.
4. This is patient data under EU MDR and Swedish patient-data law. Cambio states the consumer is responsible for filtering retrieved data for legal compliance before showing it to any end user, and that these APIs are for direct access — **not** for copying data between care providers without consent handled outside the API.

## Steps

### 1. Attention signals first
```
GET /api/open/attentionsignals
patient: <id>
?fromDateTime=2022-01-01T00:00:00&toDateTime=<now>
```
`get-attention-signals` returns allergies, contagious infections and other alerts that change how everything else should be read. Do this before anything clinical.

### 2. Demographics
```
GET /api/open/patient
patient: <id>
```
`get-patient-information-v2`. No date filter — it returns the current demographic record.

### 3. Diagnoses, medications, labs, notes, contacts
Each of these takes the same `patient` header plus a time window:

| Operation | Path | Window parameters |
|---|---|---|
| `get-diagnoses-v2` | `/api/open/diagnosis` | `fromDateTime`, `toDateTime` |
| `getPrescriptionInfo-v2` | `/api/open/medications` | `fromDateTime`, `toDateTime` |
| `get-chemistry-lab-results-v2` | `/api/open/chemistrylabreports` | `startActionDateTime`, `endActionDateTime` |
| `get-journal-notes-v2` | `/api/open/journalnotes` | `fromDateTime`, `toDateTime` |
| `get-contacts-v2` | `/api/open/contacts` | `fromDateTime`, `toDateTime` |

Note the lab API uses **different parameter names** from every other endpoint. Date format is `1970-01-01T01:00:00`.

### 4. Do not use the deprecated twins
`get-diagnoses`, `getPrescriptionInfo`, `get-chemistry-lab-results`, `get-journal-notes`, `get-patient-information` and `get-contacts` are published under an API version literally labelled `deprecated`. Always call the `-v2` operation.

## Rules

- **No pagination exists.** No page, offset or cursor parameter is published on any of these operations. Narrow the time window instead of trying to page.
- **Errors come in three shapes.** Gateway rejections return `{statusCode, message}`; application errors return `{error, path, status, timestamp}`; the FHIR surface returns a FHIR `OperationOutcome`. Parse defensively.
- **429 means back off 5 seconds.** The gateway body says "Rate limit is exceeded. Try again in 5 seconds." No quota, window or `Retry-After` header is published — treat the 429 itself as the only signal.
- **401 has two causes.** A missing subscription key and an expired bearer token both return 401 with different bodies. Read `message` / `error` before deciding to re-authenticate.
- Test against the COS Sandbox first, using the published synthetic patients (see `sandbox/cambio-healthcare-systems-sandbox.yml`).
