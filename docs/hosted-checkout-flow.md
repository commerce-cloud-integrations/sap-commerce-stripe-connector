# Hosted Checkout Flow

Hosted Checkout redirects the shopper from Spartacus to a Stripe-hosted payment
page. After Stripe redirects back, the storefront calls the connector finalize
endpoint to place or retrieve the SAP Commerce order.

## Create and Redirect

```mermaid
sequenceDiagram
  autonumber
  actor Shopper
  participant Storefront as StripeCheckoutPaymentComponent
  participant Flow as StripeCheckoutFlowService
  participant OCC as StripeCheckoutController
  participant Facade as DefaultStripeCheckoutFacade
  participant Service as DefaultStripeCheckoutSessionService
  participant Stripe as Stripe API
  participant Tx as DefaultStripePaymentTransactionService

  Shopper->>Storefront: Continue with hosted Checkout
  Storefront->>Flow: getActiveCartIdentifier()
  Flow-->>Storefront: cartId
  Storefront->>Flow: assignPaymentOptionIfSupported(stripe-checkout, cartId)
  Storefront->>OCC: POST /stripe/checkout/session?cartId=...
  OCC->>Facade: createCheckoutSessionForCart()
  Facade->>Service: createCheckoutSession(sessionCart)
  Service->>Stripe: create Checkout Session
  Stripe-->>Service: Session id, url, status
  Service->>Tx: registerCheckoutSession(cart, sessionData)
  Service-->>Facade: StripeCheckoutSessionData
  Facade-->>OCC: StripeCheckoutSessionFacadeData
  OCC-->>Storefront: StripeCheckoutSessionWsDTO
  Storefront->>Stripe: window.location.assign(session.url)
```

The service builds the Checkout Session with:

- `mode=payment`
- one line item for the SAP Commerce order total
- `clientReferenceId` set to the cart code
- success and cancel URLs with cart/order context parameters
- metadata for `orderCode`, `siteUid`, `orderType`, and `paymentFlow=checkout`
- card payment method type

## Return and Finalize

```mermaid
sequenceDiagram
  autonumber
  participant Stripe as Stripe hosted Checkout
  participant Return as StripeCheckoutReturnComponent
  participant StorefrontSvc as StripeCheckoutService
  participant OCC as StripeCheckoutController
  participant Facade as DefaultStripeCheckoutFacade
  participant SessionSvc as DefaultStripeCheckoutSessionService
  participant Tx as DefaultStripePaymentTransactionService
  participant CheckoutFacade as SAP CheckoutFacade
  participant Router as Spartacus routing

  Stripe-->>Return: Redirect with session_id, cartId or orderCode
  Return->>StorefrontSvc: pollCheckoutSession(sessionId, context)
  StorefrontSvc->>OCC: GET /stripe/checkout/session/{sessionId}
  OCC->>Facade: getCheckoutSessionForContext(sessionId, reference)
  Facade->>SessionSvc: getCheckoutSession(sessionId, siteId, reference)
  SessionSvc-->>Facade: paid or terminal session data
  Return->>StorefrontSvc: finalizeCheckoutSession(sessionId, context)
  StorefrontSvc->>OCC: POST /stripe/checkout/session/{sessionId}/finalize
  OCC->>Facade: finalizeCheckoutSessionForContext(sessionId, reference)
  Facade->>SessionSvc: getCheckoutSession(sessionId, siteId, reference)
  Facade->>Tx: findOrderByPaymentReference(reference, sessionId)
  alt Existing order owns the session
    Facade-->>OCC: existing OrderData
  else Cart owns a paid session
    Facade->>CheckoutFacade: placeOrder()
    CheckoutFacade-->>Facade: OrderData
  end
  OCC-->>Return: OrderWsDTO
  Return->>Router: route to order confirmation
```

The return component does not call the generic OCC order placement endpoint.
It calls the Stripe finalize endpoint so the connector can re-check Stripe
state and local ownership first.

## Cancel or Restart

```mermaid
sequenceDiagram
  autonumber
  participant Storefront as Cancel or restart action
  participant OCC as StripeCheckoutController
  participant Facade as DefaultStripeCheckoutFacade
  participant Lifecycle as DefaultStripePaymentLifecycleService
  participant Stripe as Stripe API
  participant Tx as DefaultStripePaymentTransactionService

  Storefront->>OCC: POST /stripe/checkout/session/{sessionId}/expire
  OCC->>Facade: expireCheckoutSessionForCart(sessionId)
  Facade->>Lifecycle: expireCheckoutSession(cart, sessionId)
  Lifecycle->>Stripe: retrieve Checkout Session
  Lifecycle->>Lifecycle: validate metadata ownership
  Lifecycle->>Stripe: expire Checkout Session
  Lifecycle->>Tx: markCheckoutSessionExpired(cart, sessionData)
  Tx-->>Lifecycle: AUTHORIZATION rejected if not captured
  Lifecycle-->>OCC: expired session data
  OCC-->>Storefront: StripeCheckoutSessionWsDTO
```

## Finalizable State

The hosted Checkout facade finalizes only when the retrieved session is ready
for order placement. The implementation treats paid or complete Checkout
Session state as finalizable and rejects a session that is still pending or
unpaid.
