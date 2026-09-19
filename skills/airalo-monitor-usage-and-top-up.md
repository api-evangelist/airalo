---
name: Monitor eSIM usage and sell a top-up
description: >-
  Watch a live Airalo eSIM's remaining data, subscribe to the low-data webhook, and sell the traveller a
  top-up before they run out.
api: openapi/airalo-partner-api-openapi.yml
operations: [requestAccessToken, getEsimUsage, optInNotification, getTopUpPackageList, submitTopUpOrder, getEsimPackageHistory]
---

# Monitor eSIM usage and sell a top-up

The retention loop. Everything here needs a bearer token from `requestAccessToken` (24h, cache it).

## Poll or subscribe — pick subscribe

`getEsimUsage` (`GET /v2/sims/{sim_iccid}/usage`) returns remaining and total data, text and voice plus
`expired_at`. It is **cached 20 minutes in Production** (60 in Sandbox), so polling harder than that
returns the same numbers, and the limits are strict: 10 requests per minute for the same ICCID and 5 per
second per client_id.

The better pattern is the **low-data webhook**. `optInNotification`
(`POST /v2/notifications/opt-in`, JSON) with `type: webhook_low_data` and a `webhook_url`, and Airalo
pushes an alert at 75% and 90% of the allowance and at 3 days and 1 day before expiry. Your endpoint must
answer **HEAD with 200** (Airalo checks it at opt-in) and accept POST.

Verify every delivery: the `airalo-signature` header is an HMAC-SHA512 of the raw JSON body keyed on your
API secret. Reject anything that does not match. Airalo retries a failed delivery **20 times at 15-minute
intervals**, so return 200 for anything you have accepted — including events you cannot act on — and
reserve a non-2xx for genuine processing failures.

Webhooks do not fire in Sandbox mode. Use `simulateWebhook` (`POST /v2/simulator/webhook`) with
`event: low_data_notification` and `type: data_75` to test the handler; passing `webhook_url` there also
bypasses the opt-in check so you can test a new endpoint before subscribing it.

## Sell the top-up — `getTopUpPackageList` then `submitTopUpOrder`

`GET /v2/sims/{iccid}/topups` lists what this specific eSIM can be extended with — the catalogue is
per-eSIM, so never offer a package from `getPackages` as a top-up without checking here first. 10 requests
per minute for the same ICCID.

`POST /v2/orders/topups` (`multipart/form-data`) with the `package_id` and the `iccid`. Known 422s: missing
required fields, invalid package id, and a **recycled sim** — error code 73, terminal, the ICCID can never
be used or topped up again. Sell the traveller a new eSIM instead.

As with ordering, there is **no idempotency key**. A blind retry sells a second top-up.

## Reconcile — `getEsimPackageHistory`

`GET /v2/sims/{iccid}/packages` returns the original package plus every top-up applied to that eSIM, which
is what you reconcile billing and support tickets against.
