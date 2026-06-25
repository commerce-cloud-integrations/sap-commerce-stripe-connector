# Testing

The repository includes unit and integration test surfaces for the main Stripe
connector behavior. Run the relevant SAP Commerce build and test suites for
your target project after changing connector code or configuration.

## Unit Test Surfaces

| Area | Test examples | Coverage focus |
| --- | --- | --- |
| Webhooks | `DefaultStripeWebhookServiceTest` | Completed sessions, succeeded PaymentIntents, site mismatch handling, missing metadata fallback, unsupported payload failures. |
| Hosted Checkout facade | `DefaultStripeCheckoutFacadeTest` | Session creation, public config, session retrieval, expiration, existing order finalization, paid cart finalization, missing context failures. |
| Payment Elements facade | `DefaultStripePaymentElementFacadeTest` | Create, get, cancel, finalize paid cart, loaded cart resolution, guid lookup, expired anonymous cart fallback, Stripe metadata fallback. |
| Fulfilment payment checks | `DefaultStripeCheckoutSessionPaymentStatusServiceTest` and action tests | OK, NOK, and WAIT transitions based on SAP Commerce transaction entries. |
| Backoffice configuration | `DefaultStripeBackofficeConfigurationServiceTest` and utility tests | Site/global property resolution and masked configuration display. |

## OCC Integration Tests

The `stripeocctests` extension contains controller integration tests for:

- hosted Checkout endpoints
- Payment Elements endpoints
- refund endpoint
- authentication and ownership failures
- invalid request cases

## Storefront Tests

The storefront package contains service tests for:

- OCC endpoint construction
- checkout flow service behavior
- active cart and payment option handling

## Manual End-to-End Scenarios

Validate these flows against a running SAP Commerce and storefront setup:

1. Hosted Checkout paid order:
   - create cart
   - continue with hosted Checkout
   - pay in Stripe hosted page
   - return to storefront
   - finalize through OCC
   - land on order confirmation
   - verify SAP Commerce order and Stripe transaction entries

2. Hosted Checkout cancel:
   - create hosted Checkout Session
   - cancel or restart checkout
   - expire the session
   - verify rejected authorization when not captured

3. Payment Elements paid order:
   - create or update PaymentIntent
   - mount Payment Element
   - confirm with Stripe.js
   - finalize through OCC
   - land on order confirmation
   - verify SAP Commerce order and PaymentIntent transaction entries

4. Webhook state update:
   - forward Stripe CLI to `/stripeevents/webhooks/stripe`
   - trigger supported events
   - verify accepted or rejected transaction state on the owning cart or order

5. Refund:
   - place a paid order
   - call the refund endpoint with a Checkout Session or PaymentIntent
     reference
   - verify Stripe refund and SAP Commerce `REFUND_FOLLOW_ON` entry

## Suggested Verification Commands

From `hybris/bin/platform` after setting the SAP Commerce environment:

```bash
. ./setantenv.sh
ant clean all
```

Project-specific test commands vary by SAP Commerce setup. Run the unit and
integration suites that include `stripeservices`, `stripefacades`, `stripeocc`,
`stripeevents`, `stripefulfilmentprocess`, and `stripebackoffice`.
