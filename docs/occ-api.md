# OCC API

The OCC endpoints are implemented by `stripeocc`. All checkout and Payment
Elements endpoints are under the current base site and OCC user:

```text
/{baseSiteId}/users/{userId}
```

Anonymous storefront calls can pass `cartId` when the active cart cannot be
restored from OCC session context.

## Hosted Checkout Endpoints

Base path:

```text
/{baseSiteId}/users/{userId}/stripe/checkout
```

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/config` | Returns frontend-safe public Stripe config. |
| `POST` | `/session` | Creates a hosted Stripe Checkout Session for the current cart. |
| `GET` | `/session/{sessionId}` | Retrieves a Checkout Session owned by the current context. |
| `POST` | `/session/{sessionId}/finalize` | Places or returns the SAP Commerce order for a paid session. |
| `POST` | `/session/{sessionId}/expire` | Expires a current-cart Checkout Session. |

Supported query parameters:

- `cartId`: anonymous cart guid or cart code.
- `orderCode`: original cart or order code for return and read contexts.
- `fields`: OCC field set for order responses on finalize endpoints.

`StripeCheckoutSessionWsDTO` fields:

- `id`
- `url`
- `status`
- `paymentStatus`
- `clientReferenceId`

`StripePublicConfigurationWsDTO` fields:

- `publishableKey`
- `paymentOptionId`
- `paymentMethod`

## Payment Elements Endpoints

Base path:

```text
/{baseSiteId}/users/{userId}/stripe/elements
```

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/intent` | Creates or returns a current-cart PaymentIntent bootstrap payload. |
| `GET` | `/intent/{paymentIntentId}` | Returns the frontend bootstrap payload for a PaymentIntent. |
| `POST` | `/intent/{paymentIntentId}/finalize` | Places or returns the SAP Commerce order for a paid PaymentIntent. |
| `POST` | `/intent/{paymentIntentId}/cancel` | Cancels a current-cart PaymentIntent. |

Supported query parameters:

- `cartId`: anonymous cart guid or cart code.
- `fields`: OCC field set for order responses on finalize endpoints.

`StripePaymentElementWsDTO` fields:

- `id`
- `clientSecret`
- `status`
- `amount`
- `currency`
- `clientReferenceId`
- `publishableKey`
- `paymentOptionId`
- `paymentMethod`
- `formattedAmount`
- `returnUrl`

## Refund Endpoint

Base path:

```text
/{baseSiteId}/users/{userId}/orders/{code}/stripe/refunds
```

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/` | Creates a Stripe refund for an owned order and Stripe payment reference. |

`StripeRefundRequestWsDTO` fields:

- `paymentReference`: required Checkout Session id or PaymentIntent id.
- `amount`: optional positive major-unit refund amount. Omit it for a full
  refund.

`StripeRefundWsDTO` fields:

- `id`
- `paymentIntentId`
- `status`
- `amount`
- `currency`
- `formattedAmount`
- `orderCode`
- `paymentReference`

## Context and Ownership

OCC controllers load cart context before creating, reading, expiring, or
canceling Stripe objects. Finalization endpoints rely on facade ownership
checks so a paid Stripe object can only place or return the SAP Commerce order
that owns the matching Stripe request id and metadata.
