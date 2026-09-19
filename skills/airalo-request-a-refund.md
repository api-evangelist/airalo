---
name: Request an Airalo refund
description: >-
  Submit a refund request for up to five Airalo eSIMs with a valid reason code, and set the right
  expectation — the endpoint opens a manually reviewed credit request, it does not reverse a charge.
api: openapi/airalo-partner-api-openapi.yml
operations: [requestAccessToken, getEsimsList, getEsimUsage, requestRefund, getBalance]
---

# Request an Airalo refund

## Know what this is before you call it

`requestRefund` is **not a reversal**. Airalo states the endpoint is enabled by default for every API
partner, but that submitting a request does not guarantee a refund: each one is **manually reviewed by
Airalo Customer Support against the Refund Policy**, and an approved refund is credited to the partner
account as **Airalo credit for future transactions** — not returned to an original payment method.

Tell your own user that, rather than promising money back. There is no published decision SLA and no status
endpoint for a submitted request.

## Submit it — `requestRefund`

`POST /v2/refund`, `multipart/form-data`:

- `iccids` — array, **maximum 5 ICCIDs per request**
- `reason` — one value from the closed list below; an unlisted string is rejected
- `notes` — max 255 characters, **required when `reason` is `OTHERS`**
- `email` — optional, must be a valid address

Valid reasons: `INSTALLATION_FAILURE`, `NO_COVERAGE`, `APN_FAILURE`, `TRIP_CANCELLATION`,
`INTERMITTENT_CONNECTION`, `BLOCKED_NETWORK`, `CHANGE_OF_PLAN`, `DELETED_ESIM`, `EARLY_EXPIRY`,
`HOTSPOT_NOT_WORKING`, `IMSI_CHANGE`, `INCOMPATIBLE_DEVICE`, `LOCKED_DEVICE`, `NO_VOICE_TEXT_SERVICES`,
`OVERCHARGED`, `SLOW_SPEED`, `TOP_UP_PACKAGE_FAILURE`, `UNKNOWN_CHARGES`, `WRONG_PURCHASE`,
`UNABLE_TO_ACCESS_APPS`, `SERVICE_DEGRADATION`, `QR_ISSUE_PARTNERS`, `OTHERS`.

Success is **202 Accepted** — accepted for review, not approved. Rate limited to **5 requests per minute
per IP**, and there is no idempotency key: submitting twice opens two review requests on the same ICCIDs.

## Gather evidence first

A request reviewed against a policy is stronger with facts attached. Before submitting:

- `getEsimsList` (`GET /v2/sims`, filterable on `filter[iccid]` and `filter[created_at]`) to confirm the
  eSIM is yours and when it was ordered.
- `getEsimUsage` (`GET /v2/sims/{sim_iccid}/usage`) to show whether data was ever consumed — a
  `NO_COVERAGE` or `INSTALLATION_FAILURE` claim on an eSIM with zero usage is the clean case.
- Check for error code 73 on the ICCID: a **recycled** eSIM can no longer be used or topped up, which is a
  different conversation than a refund.

## After

`getBalance` (`GET /v2/balance`) shows the partner credit accounts where an approved refund lands. In
Sandbox mode refunds are simulated — a refund object comes back and nothing is actually processed.
