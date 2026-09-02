---
name: cambio-appointments-and-referrals
description: Read booked appointments and video meetings from Cambio COSMIC, update an appointment, and create a referral request through Cambio Open Services — including what cannot be undone.
api: Cambio Open Services (COSMIC REST)
base_url: https://api.openservices.cambio.se
operations:
  - get-bookings
  - update-appointment
  - get-videomeetings
  - create-referral
  - register-payment
generated: '2026-09-02'
method: generated
source: openapi/ (provider-published operations exported from developer.openservices.cambio.se)
---

# Appointments, video meetings and referrals in Cambio COSMIC

This skill covers the **write** surface of Cambio Open Services. Read the reversibility warning before you call anything here.

## Credentials

As with every COS call: `Ocp-Apim-Subscription-Key` **and** an OAuth 2.0 bearer token. Writes need write scopes — `user/Appointment.cu` or `patient/Appointment-write` for appointments, `user/Referral.write` for referrals, `user/PaymentNotice.*` for payment notices. A read scope on a write call returns **403**.

## Reversibility — read this first

Cambio publishes **no reversal operation** for any write in Open Services, and **no delete interaction on any of the 24 FHIR resources**. The clinical model handles this by versioning rather than deletion: Cambio states that all information written to the COSMIC database is version handled. In practice:

| Write | Can it be taken back? |
|---|---|
| `update-appointment` (PUT) | Only by writing the previous representation back yourself. Capture the prior state from `get-bookings` **before** you write. |
| `create-referral` (POST) | **No.** No cancel or withdraw operation is published, and FHIR `ServiceRequest` supports read/search/create but not update or delete. |
| `register-payment` (POST) | **No.** No void or reverse operation is published. |

No time window for any of this is stated anywhere in Cambio's public documentation. Do not assume one exists.

## Steps

### 1. List what is already booked
```
GET /api/open/appointments
patient: <id>
?fromDateTime=<start>&toDateTime=<end>
```
`get-bookings`. Keep the full response — it is your only rollback material for step 2.

### 2. Update an appointment
```
PUT /api/open/appointments?id=<appointment id>
patient: <id>
Content-Type: application/json
```
`update-appointment`. This is a full-replacement PUT: send the complete representation, not a patch. There is no idempotency key, but PUT is idempotent by HTTP semantics, so a retried identical request is safe.

### 3. Video meetings
```
GET /open/api/videomeetings
patient: <id>
?fromDateTime=<start>&toDateTime=<end>
```
`get-videomeetings`. Note the path prefix is `/open/api/`, not `/api/open/` — this API and the deprecated medications API are the two that invert it.

### 4. Create a referral
```
POST /api/open/referrals
Content-Type: application/json
```
`create-referral`. Unlike the read operations this one does **not** take a `patient` header — the patient is identified in the body. There is no idempotency key and no cancel operation, so **never retry a failed POST blindly**: re-read with the referral APIs or ask a human before firing again.

### 5. Register a payment notice
```
POST /api/open/paymentnotice
patient: <id>
Content-Type: application/json
```
`register-payment`. Same caution as referrals — no idempotency key, no reversal.

## Rules

- **Never auto-retry an unsafe POST.** With no idempotency key and no reversal path, a duplicate referral or payment notice is a permanent record in a patient's chart.
- On **429**, wait 5 seconds (the gateway's own hint) and retry only safe reads.
- Rehearse every write in the COS Sandbox against the published synthetic patients before touching a real installation.
