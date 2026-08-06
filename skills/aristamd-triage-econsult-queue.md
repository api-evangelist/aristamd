---
name: Triage and work the eConsult queue as a specialist
description: Find eConsults awaiting review, claim one, record progress, drive its state transitions, and close it out with a review.
api: openapi/aristamd-openapi-original.json
operations:
  - GET /econsults/search
  - GET /econsults
  - GET /econsults/{econsultId}
  - PATCH /econsults/{econsultId}/assign-to-me
  - POST /econsults/{eConsultId}/heartbeat
  - POST /econsults/{eConsultId}/events
  - POST /comments
  - POST /reviews
  - GET /reviews
  - POST /econsults/logAvailability
base_url: https://api.aristamd.com
---

# Triage and work the eConsult queue as a specialist

The panelist-side flow, grounded in AristaMD's published Swagger 2.0 document.
Operations are addressed by **method + path** because operationIds are not unique
in the source document.

**PHI warning.** Everything in this queue is protected health information. No
logging of bodies, no retention outside the system of record.

## 1. Publish your availability

```
POST /econsults/logAvailability
```

The routing engine reads availability when resolving
`GET /panelists/getNextAvailable/{code}/{patient_id}`. If you do not log it, the
engine cannot route to you.

## 2. Find work

```
GET /econsults/search?status=<status>
```

Returns eConsult **ids** by status — the cheap call, use it for polling.

For a richer view:

```
GET /econsults?status=&site=&request_date[from]=&request_date[to]=&panelistId=
```

`request_date[from]` / `request_date[to]` use PHP bracket notation. Note this
operation has **no pagination** — filter narrowly rather than listing broadly.

## 3. Claim it

```
PATCH /econsults/{econsultId}/assign-to-me
```

Assigns the eConsult to the authenticated user. Do this before doing clinical
work, so the record reflects who is responsible.

## 4. Read the case

```
GET /econsults/{econsultId}
```

The `EConsult` response embeds `patient`, `coverage`, `specialty`, `requester`,
`requesting_physician`, `comments[]` and `attachments[]`. There is no expand or
sparse-fields parameter, so you get whatever the server decides to include —
handle both shallow (ids only) and deep (embedded objects) responses.

Attachments are `Asset` objects with `security_profile`, `mime_type`, `hash`,
`size` and `url`. Respect `security_profile`.

## 5. Record that you are working

```
POST /econsults/{eConsultId}/heartbeat
{ "specialistId": <number> }
```

Updates consult-time-progress. Send it periodically while reviewing so the
platform's turnaround metrics are accurate.

## 6. Add your recommendation

```
POST /comments
```

Attach polymorphically: `association_type` + `association_id` (the eConsult id),
plus `text`.

## 7. Transition the eConsult

```
POST /econsults/{eConsultId}/events
{ "action": "<action>" }
```

Returns `204` on a successful transition.

The body takes a bare `action` string and **the specification does not enumerate
the valid values**. This is a real gap: an agent cannot discover the state
machine from the contract. The `StateTransition` schema (`object_type`,
`object_id`, `transition`, `from_status`, `to_status`) confirms transitions are
audited, but the vocabulary is not published. Obtain the valid action list from
AristaMD before automating this step — do not guess action strings against a
clinical state machine.

A `404` here means either the eConsult or the transition was rejected; the
response does not distinguish them.

## 8. Close the loop with a review

```
POST /reviews
GET /reviews?association_type=&association_id=&question_type=
```

`Review` carries `question_type` and `answer`, attached by
`association_type` + `association_id`.

## Polling, because there are no events

AristaMD publishes no webhooks, no AsyncAPI and no event stream. The three
`/events` endpoints are **inbound** state-transition handlers you call — not
notifications you receive.

To detect new work, poll `GET /econsults/search?status=` on an interval. Choose
the interval conservatively: no rate-limit headers are returned, so you cannot
see a limit until you hit it.

## Errors

Same envelope as the rest of the API: `{"message": "<string>"}`, no
machine-readable code, not RFC 9457. `401` is the real auth failure even though
operations declare `400`/`403`. `500` on a transition means the transition may or
may not have applied — re-read `GET /econsults/{econsultId}` before retrying,
since there is no idempotency key.
