---
name: cambio-fhir-r4-access
description: Query the Cambio Open Services HL7 FHIR R4 server for COSMIC clinical data — which resources exist, which interactions each supports, which search parameters are declared, and where the profiles live.
api: Cambio Open Services FHIR R4
base_url: https://api.openservices.cambio.se/fhir
operations:
  - fhir-get
  - fhir-post
  - fhir-put
generated: '2026-09-02'
method: generated
source: fhir/cambio-healthcare-systems-capabilitystatement.json (provider-published CapabilityStatement, HTTP 200)
---

# Querying the Cambio Open Services FHIR R4 server

Cambio runs a HAPI FHIR based R4 (4.0.1) server over COSMIC and publishes a real `CapabilityStatement` plus an Implementation Guide of COSMIC-specific profiles. Read the CapabilityStatement before you design a query — the interaction set is narrower than stock FHIR.

## Endpoints

- Base: `https://api.openservices.cambio.se/fhir/<Resource>`
- CapabilityStatement: `https://fhir.openservices.cambio.se/site/CapabilityStatement-CapabilityStatement.json`
- Implementation Guide: `https://fhir.openservices.cambio.se/site/index.html` (canonical `https://fhir.cambio.se/ImplementationGuide/se.cambio.fhir`, version 1.0.0)
- FHIR package: `https://fhir.openservices.cambio.se/site/package.tgz`

## Credentials

`Ocp-Apim-Subscription-Key` plus an OAuth 2.0 bearer token, exactly as for the REST APIs. The CapabilityStatement declares `OAuth` as the security service. Scopes use SMART-on-FHIR syntax: `user/Patient.rus`, `patient/Observation.cds`, `system/Encounter.*` and so on — 723 scopes are advertised in the discovery document across 36 resource names.

## What the server actually supports

Read `fhir/cambio-healthcare-systems-fhir.yml` in this repo for the full table. The shape to plan around:

- **Full CRUD-ish** (create, read, update, search): `Patient`, `Practitioner`, `Organization`.
- **Create + update, no read/search**: `Bundle`, `MedicationDispense`.
- **Search only**: `Condition`, `CarePlan`, `Immunization`, `Invoice`, `MedicationRequest`, `Basic`.
- **Create only**: `ClinicalImpression`, `Provenance`.
- **No `delete` interaction on any resource.** Nothing written through this server can be removed — only superseded, and only where `update` is declared.
- `Encounter` carries the richest search surface (7 declared search parameters); `Observation` has 6.

## Steps

1. Fetch the CapabilityStatement and confirm the interaction you want is declared for that resource. Do not assume stock FHIR behaviour.
2. Acquire a token carrying the matching SMART scope. `403 Insufficient scope` is the failure you will hit most often.
3. Search with the declared parameters only — undeclared parameters are not guaranteed to be honoured.
4. Validate against the Cambio profile for the resource, not the base FHIR profile. The IG ships 40+ COSMIC-specific profiles including many `Observation` variants.

## Rules

- Errors on this surface come back as a FHIR `OperationOutcome` (`issue[].severity`, `issue[].code`, `issue[].details.text`), unlike the REST APIs. Handle both.
- The gateway sits in front of FHIR too: an unauthenticated call to `/fhir/metadata` returns the APIM envelope `{"statusCode": 401, "message": "Access denied due to missing subscription key..."}`, not a FHIR `OperationOutcome`.
- `Provenance` is create-only. If you write on behalf of an agent, you can record provenance but you cannot correct it afterwards.
- There is no `delete`, no bulk-export (`$export`) and no subscription/webhook capability declared. Polling is the only change-detection mechanism available.
