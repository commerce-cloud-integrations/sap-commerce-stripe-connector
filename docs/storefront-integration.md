# Storefront Integration

The storefront package is under:

```text
js-storefront/stripe-spartacus-connector
```

It is an Angular package for SAP Composable Storefront 2211.31 and provides the
checkout payment component, Stripe return routes, OCC client services, and
Stripe.js loading for Payment Elements.

## Module Wiring

Import `StripeCheckoutModule` in the storefront feature wiring. The module:

- declares `StripeCheckoutPaymentComponent`
- declares hosted Checkout return and cancel components
- declares the Payment Elements return component
- registers routes for Stripe redirects
- maps the Spartacus `CheckoutPaymentDetails` CMS component to
  `StripeCheckoutPaymentComponent`
- keeps the standard checkout guards: `CheckoutAuthGuard` and
  `CartNotEmptyGuard`

## Routes

The package defines these storefront routes:

| Route | Component | Purpose |
| --- | --- | --- |
| `stripe/checkout/return` | `StripeCheckoutReturnComponent` | Receives hosted Checkout success redirects. |
| `stripe/checkout/cancel` | `StripeCheckoutCancelComponent` | Receives hosted Checkout cancel redirects. |
| `stripe/elements/return` | `StripePaymentElementReturnComponent` | Receives Payment Elements redirects when Stripe.js requires one. |

## Checkout Payment Component

`StripeCheckoutPaymentComponent` lets the shopper choose between:

- hosted Checkout (`checkout`)
- Payment Elements (`elements`)

For hosted Checkout it gets the active cart identifier, assigns the
`stripe-checkout` payment option when supported, creates a Checkout Session
through OCC, and redirects the browser to the hosted Stripe URL.

For Payment Elements it creates or retrieves a PaymentIntent through OCC, loads
Stripe.js with the publishable key, mounts the Payment Element with the client
secret, and confirms the payment with `redirect: "if_required"`.

## OCC Client Services

`StripeCheckoutService` calls:

- `GET stripe/checkout/config`
- `POST stripe/checkout/session`
- `GET stripe/checkout/session/{sessionId}`
- `POST stripe/checkout/session/{sessionId}/finalize`
- `POST stripe/checkout/session/{sessionId}/expire`
- `PUT carts/{cartId}/paymentoption`

`StripePaymentElementService` calls:

- `POST stripe/elements/intent`
- `GET stripe/elements/intent/{paymentIntentId}`
- `POST stripe/elements/intent/{paymentIntentId}/finalize`
- `POST stripe/elements/intent/{paymentIntentId}/cancel`

It also wraps `loadStripe` and `retrievePaymentIntent`.

## Cart Context

`StripeCheckoutFlowService` is responsible for active cart behavior:

- finding the active cart guid or code
- restoring cart context on return routes
- assigning payment options while ignoring unsupported 400, 401, or 403
  responses
- setting the placed order
- removing the cart after order placement
- routing to order confirmation

## Guest Checkout

Keep guest checkout enabled when the checkout CMS flow includes guest checkout
login components. Return routes rely on the cart id and OCC context so
anonymous checkout can be restored after Stripe redirects back to the
storefront.

## Local Validation URL

The validated local storefront target is:

```text
http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/
```

The return URLs in SAP Commerce properties should point to that storefront
base path unless your local site setup uses a different base URL.
