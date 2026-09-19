---
name: Handle async and future-dated Airalo orders
description: >-
  Place high-volume asynchronous orders and date-scheduled future orders against the Airalo Partner API,
  correlate them through the webhook, and cancel a scheduled order inside its window.
api: openapi/airalo-partner-api-openapi.yml
operations: [requestAccessToken, optInNotification, submitOrderAsync, submitFutureOrder, getFutureOrders, cancelFutureOrders, getOrder]
---

# Handle async and future-dated Airalo orders

Two order shapes that do not return an eSIM in the response. Both resolve on your webhook, and both
correlate through a server-generated `request_id`. Token first (`requestAccessToken`, 24h, cached).

## Before either: subscribe — `optInNotification`

`POST /v2/notifications/opt-in` with `type: async_orders` and your `webhook_url`. Email delivery is not
supported for this event type. Your endpoint must answer HEAD with 200 at opt-in and accept POST.

Either order endpoint also accepts a per-request `webhook_url` that **overrides** the opted-in URL for that
order. A future order with neither an opt-in nor a `webhook_url` is rejected.

## Asynchronous orders — `submitOrderAsync`

`POST /v2/orders-async` (`multipart/form-data`), same fields as `submitOrder` plus the optional
`webhook_url`. It returns a 25-character nanoid in `request_id` immediately.

**Store that `request_id` before you do anything else.** It is the only key that ties the webhook payload
back to the order you placed — the webhook body is the standard order response with `request_id` added. A
failure payload arrives with a `reason` and no `sims`; that is a terminal Airalo error (out of stock, for
example), so acknowledge it with 200 and do not retry.

Known 422s: quantity not available, and "not opted in and no webhook_url provided".

## Future-dated orders — `submitFutureOrder`

`POST /v2/future-orders` (JSON) with `quantity` (1–50), `package_id` and `due_date` as
`YYYY-MM-DD HH:MM` — **minimum two days ahead, maximum one year**. Optional `to_email` +
`sharing_option` (`link` / `pdf`) emails the traveller when it fires.

It returns a `request_id`. At the due date Airalo processes the order and POSTs the completed order — again
with `request_id` — to your webhook.

Track the queue with `getFutureOrders` (`GET /v2/future-orders`), filtering on `status`, `limit`,
`from_due_date` and `to_due_date`. Statuses you will see: pending, retry, failed.

## Cancel inside the window — `cancelFutureOrders`

`POST /v2/cancel-future-orders` (JSON) with `request_ids` — an array, **up to 10 per request**.

**The window is 24 hours before the due date.** After that the order cannot be cancelled and will provision.
This is the one reversal path with a hard published deadline, so build against it: a scheduled order is
recoverable, a placed order is not (the only route back is a manually reviewed refund request — see
`airalo-request-a-refund.md`).

Expect 422 "Already processed request ID" and "Request Id does not exist".

## Rehearse in Sandbox

Future-order statuses are forceable with published test package slugs:
`test-create-pending-future-order-7days-1gb`, `test-create-retry-future-order-7days-1gb`,
`test-create-failed-future-order-7days-1gb`. Any non-test package processes normally. Note that webhook
notification settings do not function in Sandbox — use `simulateWebhook` with
`event: async_order_notification` to exercise the handler.
