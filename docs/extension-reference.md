# Extension Reference

All SAP Commerce extensions are under `hybris/bin/modules/stripe`.

## stripeservices

Primary service-layer extension. It contains:

- Stripe SDK wrapper methods in `DefaultStripeClientFactory`.
- Provider configuration resolution in `DefaultStripeConfigurationService`.
- Checkout Session creation and retrieval in
  `DefaultStripeCheckoutSessionService`.
- PaymentIntent creation, reuse, update, retrieval, and ownership validation in
  `DefaultStripePaymentIntentService`.
- Provider lifecycle operations in `DefaultStripePaymentLifecycleService`.
- Webhook verification and dispatch in `DefaultStripeWebhookService`.
- Payment transaction persistence in `DefaultStripePaymentTransactionService`.

Required SAP Commerce extensions: `acceleratorservices`, `commerceservices`,
`opfservices`, and `webhookservices`.

## stripefacades

Facade orchestration layer. It depends on `stripeservices` and shields OCC and
storefront callers from provider and model-level details.

Important classes:

- `DefaultStripeCheckoutFacade`
- `DefaultStripePaymentElementFacade`
- `DefaultStripeRefundFacade`
- `AbstractStripeCheckoutFacadeSupport`

Required SAP Commerce extensions: `commercefacades`, `opffacades`, and
`stripeservices`.

## stripeocc

OCC API layer for storefront and API clients.

Controllers:

- `StripeCheckoutController`
- `StripePaymentElementController`
- `StripeRefundController`
- `AbstractStripeCartContextController`

DTOs are declared in `stripeocc/resources/stripeocc-beans.xml`.

Required SAP Commerce extensions: `commercewebservices`, `cmsocc`, and
`stripefacades`.

## stripeevents

Webhook HTTP module. Its webroot is `/stripeevents`, and the Stripe webhook
path is `/webhooks/stripe`, producing the full local path:

```text
/stripeevents/webhooks/stripe
```

Required SAP Commerce extensions: `cms2`, `stripeservices`,
`webhookservices`, and `webservicescommons`.

## stripefulfilmentprocess

Fulfilment process integration. It contains the checkout payment process and
the payment status action used to wait for accepted Stripe payment entries.

Important classes and resources:

- `StripeCheckCheckoutSessionPaymentAction`
- `DefaultStripeCheckoutSessionPaymentStatusService`
- `resources/stripefulfilmentprocess/process/stripe-checkout-payment-process.xml`

Required SAP Commerce extensions: `acceleratorservices`, `ticketsystem`, and
`stripeservices`.

## stripebackoffice

Backoffice visibility for connector configuration.

Important classes:

- `DefaultStripeBackofficeConfigurationService`
- `StripeBackofficeConfigurationUtils`
- `StripeBackofficeConfigurationStatus`

Required SAP Commerce extensions: `backoffice`, `acceleratorbackoffice`,
`customersupportbackoffice`, `stripeservices`, and `stripeevents`.

## stripeocctests

OCC integration tests for the connector API surface. It depends on
`commercewebservicestests` and `stripeocc`.

## stripetest

Test support extension for service and event surfaces. It depends on
`odata2services`, `stripeservices`, and `stripeevents`.

## Storefront Package

The storefront integration is a separate Angular package:

```text
js-storefront/stripe-spartacus-connector
```

It contains `StripeCheckoutModule`, the checkout payment component, return and
cancel components, OCC endpoint helpers, and Stripe.js integration.
