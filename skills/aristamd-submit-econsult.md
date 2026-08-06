---
name: Submit an eConsult to a specialist
description: Create a patient if needed, find an available specialist for the right specialty, submit an eConsult, and attach the clinical context — the core AristaMD flow.
api: openapi/aristamd-openapi-original.json
operations:
  - GET /workup-checklists/specialties
  - GET /workup-checklists/specialties/{specialtyCode}/chief-complaints
  - GET /workup-checklists/specialties/{specialtyCode}/chief-complaints/{chiefComplaintCode}
  - GET /patients/search
  - POST /patients
  - GET /specialties/withAvailablePanelists/{filter}
  - GET /panelists/getNextAvailable/{code}/{patient_id}
  - POST /econsults
  - POST /comments
  - GET /econsults/{econsultId}
base_url: https://api.aristamd.com
---

# Submit an eConsult to a specialist

Operating instructions for the AristaMD eConsult submission flow, grounded in the
Swagger 2.0 document AristaMD serves at `https://api.aristamd.com/api-docs`.

> Operations are addressed here by **method + path**, not by `operationId`. The
> published document reuses operationIds across resources (`store` appears on
> four different operations), so ids are not a safe handle.

## Before you start

**This flow handles protected health information.** Patient names, dates of
birth, gender, chronic conditions, insurance coverage and clinical attachments
all cross this API. Do not log request or response bodies, do not place them in
prompt history you retain, and do not run this flow without an appropriate
agreement in place.

**There is no idempotency key.** `POST /patients` and `POST /econsults` have no
replay protection. If either call times out, **do not retry blindly** — search
first (`GET /patients/search`, `GET /econsults/search`) to find out whether the
record was created, then decide.

## Authenticate

Get a token from the OAuth 2.0 server:

```
POST https://api.aristamd.com/oauth/token
Content-Type: application/x-www-form-urlencoded
```

Supported grant types: `authorization_code`, `client_credentials`, `password`,
`refresh_token`. Send the token as a bearer token on every call.

The specification declares no security schemes — that is a documentation gap, not
an open API. Every path returns `401 {"message":"Unauthorized"}` without a token.

## Steps

### 1. Pick the specialty and chief complaint

```
GET /workup-checklists/specialties
GET /workup-checklists/specialties/{specialtyCode}/chief-complaints
```

Then pull the workup checklist so you know what the specialist will need:

```
GET /workup-checklists/specialties/{specialtyCode}/chief-complaints/{chiefComplaintCode}
```

The checklist carries `assessments`, `diagnostics` and a `special_note`. Gather
these before submitting — an eConsult without the expected workup is what causes
a specialist round-trip.

### 2. Resolve the patient

Search before creating:

```
GET /patients/search?data[last_name]=...&data[q]=...
```

Only if there is no match:

```
POST /patients
```

The `Patient` schema carries `organization_id`, `coverage_id`,
`system_of_record`, `reference_id`, `first_name`, `last_name`, `date_of_birth`,
`gender` and `chronic_conditions`. If the patient originates in an EHR, set
`system_of_record` and `reference_id` so the record can be reconciled later.

If the patient is arriving from an HL7 v2 feed, use `POST /HL7/messages` instead.

### 3. Confirm specialist availability

```
GET /specialties/withAvailablePanelists/{filter}
GET /panelists/getNextAvailable/{code}/{patient_id}
```

The second call is the routing engine's own answer for a specialty code and
patient. Prefer it over picking a panelist yourself — AristaMD's routing accounts
for Medicaid certification and daily capacity (visible in the
`EConsultRoutingMetric` and `EConsultRoutingSpecialistMetric` schemas).

### 4. Submit the eConsult

```
POST /econsults
```

The `EConsult` schema takes `organization_id`, `requester_id`,
`requesting_physician_id`, `panelist_id`, `patient_id`, `coverage_id`,
`specialty_id` and `status`.

Expect `201` on success. Capture the returned `id` immediately — without it, and
without idempotency, you cannot safely reconcile a partial failure.

### 5. Attach clinical context

```
POST /comments
```

`Comment` attaches polymorphically via `association_type` + `association_id`. Set
`association_id` to the eConsult id.

### 6. Confirm

```
GET /econsults/{econsultId}
```

Verify `status`, `panelist_id` and `request_date`.

## Handling errors

| Status | Body | What it means |
|---|---|---|
| `400` | `{"message":"Invalid Credentials"}` | Declared on 35 operations. Also used for malformed requests. |
| `401` | `{"message":"Unauthorized"}` | The real auth failure. Not declared on any operation. Refresh the token. |
| `403` | `{"message":"Unauthorized"}` | Authenticated but lacking a role/permission. |
| `404` | `{"message":"..."}` | Resource not found — the message names the resource. |
| `500` | `{"message":"Internal Error"}` | Server failure. **Do not auto-retry writes.** |

Errors are not RFC 9457 and carry no machine-readable code. The HTTP status plus
an English string is the only discriminator available.

## What this API will not do for you

- No webhook or event notification when a specialist responds — you must poll
  `GET /econsults/search?status=` or `GET /econsults/{econsultId}`.
- No pagination on `GET /econsults`. Filter with `status`, `site`,
  `request_date[from]`, `request_date[to]` and `panelistId` rather than listing.
- No versioning, so treat every response shape as subject to change without notice.
