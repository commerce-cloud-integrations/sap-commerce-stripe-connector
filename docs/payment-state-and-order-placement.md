# Payment State and Order Placement

The connector persists Stripe payment state in SAP Commerce
`PaymentTransactionModel` and `PaymentTransactionEntryModel` records. Stripe
request identifiers are stored in transaction entry request ids so checkout
finalization, webhook processing, and refunds can resolve the owning cart or
order.

## Transaction Entry Model

| Stripe action | Entry type | Request id | Status |
| --- | --- | --- | --- |
| Checkout Session created | `AUTHORIZATION` | `cs_...` | `PENDING` |
| Checkout Session completed | `AUTHORIZATION` and `CAPTURE` | `cs_...` | `ACCEPTED` / `SUCCESSFUL` |
| Checkout Session expired | `AUTHORIZATION` | `cs_...` | `REJECTED` |
| PaymentIntent created or retrieved | `AUTHORIZATION` | `pi_...` | `PENDING` with Stripe status details |
| PaymentIntent succeeded | `AUTHORIZATION` and `CAPTURE` | `pi_...` | `ACCEPTED` with Stripe status details |
| PaymentIntent failed | `AUTHORIZATION` | `pi_...` | `REJECTED` |
| PaymentIntent canceled | `AUTHORIZATION` and `CANCEL` | `pi_...` | `REJECTED` then `ACCEPTED` cancel entry |
| Refund created | `REFUND_FOLLOW_ON` | `re_...` | mapped from Stripe refund status |

## Order Status Updates

When a capture entry exists with accepted status, the order is marked
`PAYMENT_CAPTURED`. When a rejected authorization exists without capture, the
order can be marked `PAYMENT_NOT_CAPTURED`.

## Cart to Order Synchronization

```mermaid
sequenceDiagram
  autonumber
  participant CheckoutFacade as SAP CheckoutFacade.placeOrder
  participant Hook as StripePaymentPlaceOrderMethodHook
  participant Tx as DefaultStripePaymentTransactionService
  participant Cart as Source Cart
  participant Order as Target Order

  CheckoutFacade-->>Hook: afterPlaceOrder(cart, order)
  Hook->>Tx: synchronizeStripePaymentsToOrder(cart, order)
  Tx->>Cart: read Stripe payment transaction entries
  loop Stripe payment entry
    Tx->>Order: copy entry if missing or update existing entry
  end
  Tx->>Order: update payment status from copied entries
```

The order placement hook exists because Stripe payment entries are usually
created while the checkout is still a cart. Once SAP Commerce places the order,
the relevant Stripe entries must also exist on the order.

## Finalization State Machine

```mermaid
stateDiagram-v2
  [*] --> CartPrepared
  CartPrepared --> StripePending: Create Session or PaymentIntent
  StripePending --> StripeAccepted: Stripe paid or PaymentIntent succeeded
  StripePending --> StripeRejected: expired, failed, or canceled
  StripeAccepted --> OrderPlaced: finalize endpoint calls placeOrder
  StripeAccepted --> ExistingOrderReturned: finalization sees order already exists
  StripeRejected --> [*]
  OrderPlaced --> [*]
  ExistingOrderReturned --> [*]
```

## Fulfilment Process Payment Check

The `stripefulfilmentprocess` extension includes a checkout payment process
that waits for Stripe payment updates. The payment status service checks
whether relevant payment transaction entries are accepted, rejected, or still
pending and returns an OK, NOK, or WAIT transition.

```mermaid
flowchart TD
  Start([Start checkout payment process]) --> Check[StripeCheckCheckoutSessionPaymentAction]
  Check --> Status[DefaultStripeCheckoutSessionPaymentStatusService]
  Status -->|accepted capture or authorization| Ok[OK]
  Status -->|rejected authorization| Nok[NOK]
  Status -->|pending or missing| Wait[WAIT for STRIPE_PAYMENT_UPDATED]
  Wait --> Check
  Ok --> Success([success end])
  Nok --> Failed([failed end])
```

## Idempotency Behavior

Transaction update methods check for existing accepted or rejected entries
before creating new ones. This keeps browser finalization and webhook updates
from duplicating capture, cancel, or refund entries when the same Stripe event
is observed through multiple paths.
