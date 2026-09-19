---
name: Order and install an Airalo eSIM
description: >-
  Authenticate against the Airalo Partner API, find a package for a destination, place an order, and hand
  the traveller a working installation — QR, manual SM-DP+ details, or the iOS 17.4+ direct install link.
api: openapi/airalo-partner-api-openapi.yml
operations: [requestAccessToken, getPackages, submitOrder, getEsim, getInstallationInstructions]
---

# Order and install an Airalo eSIM

The happy path a partner integration runs on every sale. Base URL `https://partners-api.airalo.com`.
Sandbox and Production share this URL and the same credentials — the mode is set on the company in the
Partner Platform, so **check which mode the account is in before placing an order**: in Production an
order provisions a real eSIM and deducts real credit.

## 1. Get a token — `requestAccessToken`

`POST /v2/token`, `application/x-www-form-urlencoded`, with `client_id`, `client_secret` and
`grant_type=client_credentials`.

The token is valid for **24 hours** and the endpoint is limited to **3 requests per minute**. Cache the
token and reuse it; do not call this per request. Send it on everything else as
`Authorization: Bearer <access_token>`.

A 401 later on means the token expired — mint a new one. If you get **HTTP 403 with body code 89**, the
token is fine and your calling IP is simply not on the company's API allowlist.

## 2. Find a package — `getPackages`

`GET /v2/packages`. Filter with `filter[country]=US` (ISO 3166-1 alpha-2) or
`filter[type]=local|global`; add `include=topup` to get top-up packages alongside. Pass
`Accept-Language` for a localised response.

The response nests **country → operators[] → packages[]**. Each package carries `id` (the slug you order,
e.g. `kallur-digital-7days-1gb`), `net_price` (your wholesale cost), `recommended_retail_price`, `amount`
in MB, `day`, and `is_unlimited` — use `is_unlimited`, not `amount`, to decide whether a plan is unlimited.

Limits: **80 requests per minute** per token. Airalo's own guidance is to sync the full catalogue
(~3,200 packages) once an hour with a high `limit` rather than paginating — and note that at `page >= 2`
the response shape changes.

## 3. Place the order — `submitOrder`

`POST /v2/orders`, `multipart/form-data`, with `quantity` (max 50) and `package_id`. Optional:
`description` (put your internal order ID here — it is echoed back and appears in the Partner Platform),
`brand_settings_name` (a brand configured in the Partner Platform; `null` = unbranded), and
`to_email` + `sharing_option[]` (`link` or `pdf`) to have Airalo email the eSIM to the traveller.

**There is no idempotency key on this endpoint.** A retried request places a second order and bills for it.
Before retrying a request whose response you did not see, call `getOrderList` filtered on your
`description` value and check whether the order already landed.

For high volume, use `submitOrderAsync` (`POST /v2/orders-async`) instead: it returns a `request_id`
nanoid immediately and delivers the completed order to your webhook. You must be opted in to
`async_orders` or pass `webhook_url` in the request, or it fails with a 422.

## 4. Deliver the install — `getEsim` / `getInstallationInstructions`

The order response already contains the sims. To fetch one later, `GET /v2/sims/{sim_iccid}` — but only
eSIMs ordered via the API are retrievable here.

An eSIM carries three install routes, and a good integration offers all three:

- **QR** — `qrcode` is a GSMA SGP.22 activation string, `LPA:1$<SM-DP+ address>$<MatchingID>`. Render it
  as a QR code, or use the ready-made `qrcode_url`.
- **Direct (iOS 17.4+)** — `direct_apple_installation_url` is an Apple universal link that installs the
  profile with one tap. Offer it when you can detect the device.
- **Manual** — `GET /v2/sims/{sim_iccid}/instructions` returns language-specific steps (pass
  `Accept-Language`) including `smdp_address_and_activation_code` and, where `apn_type` is `manual`, the
  `apn_value` the traveller must enter.

## Errors worth handling explicitly

`422` with `code` 11 (insufficient credit), 33 (insufficient stock), 34 (package invalid or out of stock),
13 (operator maintenance), 73 (recycled eSIM). `429` is Too Many Attempts — back off exponentially and
read `x-ratelimit-remaining`. Full list: `errors/airalo-problem-types.yml`.

## Rehearse it first

In Sandbox mode the order is simulated, no credit is deducted, and the returned test ICCID cannot be
installed on a device. Force failures with the published test package IDs —
`test-insufficient-balance-7days-1gb`, `test-out-of-stock-7days-1gb`,
`test-maintenance-mode-7days-1gb` — see `sandbox/airalo-sandbox.yml`.
