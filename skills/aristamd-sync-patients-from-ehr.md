---
name: Sync patients into AristaMD from an EHR
description: Bring patient records in from an HL7 v2 feed or the Greenway Intergy API, reconcile them against existing records, and keep external identifiers straight.
api: openapi/aristamd-openapi-original.json
operations:
  - POST /HL7/messages
  - GET /intergy/patients/{id}
  - GET /patients/search
  - GET /patients
  - POST /patients
  - PUT /patients/{patientId}
  - PATCH /patients/{patientId}
  - GET /patients/{patientId}
  - GET /patients/{patientId}/identifiers
  - GET /patients/{patientId}/history
base_url: https://api.aristamd.com
---

# Sync patients into AristaMD from an EHR

The integration-side flow. Grounded in AristaMD's published Swagger 2.0 document;
operations addressed by **method + path** because operationIds repeat.

**PHI warning.** This flow moves protected health information between systems.
Every body is PHI. No logging, no caching, no retention outside the systems of
record.

## The two intake routes

### HL7 v2 message intake

```
POST /HL7/messages
```

"Creates a Patient from an HL7 message." Returns `201` with the created
`Patient`.

Caveat worth knowing before you build against it: the request body is typed in
the specification as the proprietary `Patient` schema, **not** as an HL7 message,
so the contract does not tell you which HL7 message types, segments or versions
are accepted. AristaMD's own MLLP tooling (`github.com/aristamd/mllparty`,
`github.com/aristamd/elixir-mllp`) indicates real HL7 v2 transport in use.
Confirm the accepted message profile with AristaMD rather than inferring it.

### Greenway Intergy passthrough

```
GET /intergy/patients/{id}?practice_id=<id>
```

Retrieves patient information from the Intergy API. This is a passthrough to a
third-party EHR, not AristaMD's own store — use it to look up, then create or
update in AristaMD.

## Reconcile before you write

Always search before creating. There is **no idempotency key** on `POST
/patients`, so a retried create makes a duplicate patient.

```
GET /patients/search?data[last_name]=...&data[q]=...
```

or the paginated list:

```
GET /patients?start=0&length=50&orderColumn=&orderDir=&searchValue=
```

`start`/`length` is offset pagination in the DataTables convention. No total
count is returned in any declared response schema, so page until you get a short
page — you cannot ask how many there are.

## Keep external identity straight

The `Patient` schema carries the linkage fields:

- `system_of_record` — which upstream system owns this patient
- `reference_id` — that system's id for them
- `version` — AristaMD's own record version

Always set `system_of_record` and `reference_id` on create. Without them you
cannot reconcile later, and duplicate detection has nothing to key on.

To read what identifiers already exist:

```
GET /patients/{patientId}/identifiers?identifier_name=&auth_provider_id=
```

## Create and update

```
POST /patients                       # create
PUT /patients/{patientId}            # full replace
PATCH /patients/{patientId}?operation=<op>&assets=<array>   # partial
```

`PATCH` here is non-standard: it takes an `operation` query parameter and an
`assets` array rather than a JSON Patch or JSON Merge Patch body. Read the
`operation` semantics from AristaMD before using it; do not assume RFC 6902.

Prefer `PATCH` over `PUT` for routine field updates — `PUT` is a full replace and
will drop fields you did not send.

## Domain fields that will surprise you

- `AB109` — a California correctional-realignment population flag. Present on
  every `Patient`. Map it deliberately or leave it alone; do not repurpose it.
- `chronic_conditions` — clinical data, PHI, and not schema-typed in the
  published document.
- `coverage_id` — points at a `Coverage` (insurance plan, provider, member id,
  payor organization). Patient and coverage reference each other; set coverage
  before eConsult submission, because `EConsult` requires `coverage_id`.

## Verify

```
GET /patients/{patientId}
GET /patients/{patientId}/history
```

## Known rough edge

`GET /patients` returns `500 {"message":"An error has occurred while processing
your request"}` when called without credentials, where every other collection
returns `401`. The handler appears to touch the authenticated user before the
auth middleware rejects. Harmless for an authenticated client, but do not treat a
`500` from this path as an outage signal — check your token first.

## Errors

`{"message": "<string>"}`, no machine-readable code, not RFC 9457. `401` is the
real auth failure despite operations declaring `400 "Invalid credentials"`.
Never auto-retry a `500` on a create — search first.
