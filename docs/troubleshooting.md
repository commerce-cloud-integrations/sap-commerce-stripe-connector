# Troubleshooting

Use this page to map common symptoms to the connector layer that owns the
behavior.

## Public Config Fails in Storefront

Symptoms:

- checkout payment component cannot start
- OCC config endpoint fails
- publishable key is empty

Check:

- `stripe.publishable.key`
- site-specific `stripe.publishable.key.<siteUid>`
- current base site in OCC context
- `DefaultStripeConfigurationService`
- `DefaultStripeCheckoutFacade.getPublicConfiguration`

The OCC config endpoint must never return secret keys.

## Hosted Checkout Does Not Redirect

Symptoms:

- `POST /stripe/checkout/session` fails
- session response has no `url`
- Stripe rejects session creation

Check:

- cart has entries, currency, base site, and calculated totals
- `stripe.secret.key`
- absolute success and cancel URLs
- success URL contains `{CHECKOUT_SESSION_ID}`
- Stripe API key mode matches test or live expectations
- `DefaultStripeCheckoutSessionService.buildSessionCreateParams`

## Hosted Checkout Returns But Order Is Not Placed

Symptoms:

- Stripe payment succeeded
- storefront returns to `stripe/checkout/return`
- no order confirmation

Check:

- return URL includes `session_id`
- return URL carries `cartId` or `orderCode` when needed
- storefront calls `POST /stripe/checkout/session/{sessionId}/finalize`
- Checkout Session has paid or complete status
- session metadata has the expected `orderCode`, `siteUid`, and
  `paymentFlow=checkout`
- SAP Commerce has a transaction entry with request id equal to the session id

The generic OCC order placement endpoint is not enough for hosted Checkout
returns; use the Stripe finalize endpoint.

## Payment Elements Does Not Render

Symptoms:

- Payment Element host remains empty
- Stripe.js fails to initialize
- `clientSecret` is missing

Check:

- `POST /stripe/elements/intent` returns `clientSecret`
- `publishableKey` is present
- browser can load Stripe.js
- PaymentIntent was created for the current cart and site
- `StripeCheckoutPaymentComponent.tryMountPaymentElement`
- `StripePaymentElementService.getStripe`

## Payment Elements Confirms But Order Is Not Placed

Symptoms:

- Stripe.js returns `succeeded`
- order confirmation is not reached
- finalize endpoint returns an ownership or readiness error

Check:

- storefront calls `POST /stripe/elements/intent/{paymentIntentId}/finalize`
- `cartId` is present for anonymous return flows
- PaymentIntent metadata has expected `orderCode`, `siteUid`, and
  `paymentFlow=elements`
- PaymentIntent status is `succeeded` or `requires_capture`
- `DefaultStripePaymentElementFacade.resolveCheckoutOwner`
- `DefaultStripePaymentIntentService.validateOwnership`

## Webhook Signature Verification Fails

Symptoms:

- webhook endpoint returns an integration error
- Stripe CLI reports non-2xx responses

Check:

- `stripe.webhook.secret`
- site-specific `stripe.webhook.secret.<siteUid>`
- the webhook secret belongs to the active Stripe CLI listener or dashboard
  endpoint
- `Stripe-Signature` header is present
- forwarded URL is `/stripeevents/webhooks/stripe`

## Webhook Arrives But State Does Not Change

Symptoms:

- webhook returns 200
- order or cart transaction state remains pending

Check:

- event type is supported
- metadata contains the expected `paymentFlow`
- metadata site matches the resolved site
- `orderCode` metadata is present
- SAP Commerce has an order or cart with a transaction entry matching the
  Stripe request id
- `DefaultStripeWebhookService.handleCheckoutSessionEvent`
- `DefaultStripeWebhookService.handlePaymentIntentEvent`

Events with mismatched site or payment flow are intentionally ignored.

## Refund Is Rejected

Symptoms:

- refund endpoint returns a bad request or ownership error
- Stripe refund is not created

Check:

- authenticated user owns the order in the current base store
- request body includes `paymentReference`
- optional `amount` is positive
- payment reference is either a Checkout Session id or a PaymentIntent id
- the order has a transaction entry for the payment reference or the resolved
  PaymentIntent
- `DefaultStripePaymentLifecycleService.validateRefundOwnership`

## Anonymous Cart Cannot Be Restored

Symptoms:

- return route has no active cart
- finalize still needs to resolve owner

Check:

- return URL includes `cartId`
- storefront calls `restoreCartContext(cartId)` for Payment Elements returns
- OCC finalize endpoint passes `cartId`
- Stripe metadata and local transaction entries can still resolve the owner

The Payment Elements finalize controller intentionally tolerates stale cart
reload failures and lets the facade fall back to payment reference and metadata
ownership resolution.

## Backoffice Shows Missing Configuration

Symptoms:

- configuration status is missing or masked unexpectedly

Check:

- global and site-specific Stripe properties
- current base site selected in backoffice context
- `DefaultStripeBackofficeConfigurationService`
- `StripeBackofficeConfigurationUtils`

Secret values should remain masked even when configured.
