# Payment Elements Flow

Payment Elements keeps the shopper on the storefront while Stripe.js collects
and confirms payment details. The storefront must still call the connector
finalize endpoint after Stripe confirms the PaymentIntent.

## Prepare and Mount Elements

```mermaid
sequenceDiagram
  autonumber
  actor Shopper
  participant Component as StripeCheckoutPaymentComponent
  participant Flow as StripeCheckoutFlowService
  participant ElementSvc as StripePaymentElementService
  participant OCC as StripePaymentElementController
  participant Facade as DefaultStripePaymentElementFacade
  participant IntentSvc as DefaultStripePaymentIntentService
  participant Stripe as Stripe API
  participant Tx as DefaultStripePaymentTransactionService
  participant StripeJS as Stripe.js

  Shopper->>Component: Select Payment Elements
  Component->>Flow: getActiveCartIdentifier()
  Flow-->>Component: cartId
  Component->>ElementSvc: createPaymentIntent(cartId)
  ElementSvc->>OCC: POST /stripe/elements/intent?cartId=...
  OCC->>Facade: createPaymentIntentForCart()
  Facade->>IntentSvc: createOrUpdatePaymentIntent(cart)
  IntentSvc->>Tx: findLatestOpenPaymentIntentId(cart)
  alt Reusable PaymentIntent exists
    IntentSvc->>Stripe: retrieve or update PaymentIntent
  else No reusable PaymentIntent
    IntentSvc->>Stripe: create PaymentIntent
  end
  Stripe-->>IntentSvc: PaymentIntent with client secret
  IntentSvc->>Tx: registerPaymentIntent(cart, paymentIntentData)
  IntentSvc-->>Facade: StripePaymentIntentData
  Facade-->>OCC: publishable key, client secret, return URL
  OCC-->>ElementSvc: StripePaymentElementWsDTO
  Component->>ElementSvc: getStripe(publishableKey)
  ElementSvc->>StripeJS: loadStripe(publishableKey)
  Component->>StripeJS: elements({ clientSecret }).create("payment")
  Component->>Flow: assignPaymentOptionIfSupported(stripe-elements, cartId)
```

The PaymentIntent service creates or updates the intent with:

- order total in minor units
- currency
- order description
- automatic payment methods enabled
- receipt email when available
- metadata for `orderCode`, `siteUid`, `orderType`, and `paymentFlow=elements`

## Confirm Without Redirect

```mermaid
sequenceDiagram
  autonumber
  actor Shopper
  participant Component as StripeCheckoutPaymentComponent
  participant StripeJS as Stripe.js
  participant ElementSvc as StripePaymentElementService
  participant OCC as StripePaymentElementController
  participant Facade as DefaultStripePaymentElementFacade
  participant IntentSvc as DefaultStripePaymentIntentService
  participant Tx as DefaultStripePaymentTransactionService
  participant StripeAPI as Stripe API
  participant CheckoutFacade as SAP CheckoutFacade
  participant Router as Spartacus routing

  Shopper->>Component: Submit Payment Elements
  Component->>StripeJS: confirmPayment({ redirect: "if_required" })
  StripeJS-->>Component: PaymentIntent status
  alt status is succeeded or requires_capture
    Component->>ElementSvc: finalizePaymentIntent(paymentIntentId, cartId)
    ElementSvc->>OCC: POST /stripe/elements/intent/{paymentIntentId}/finalize
    OCC->>Facade: finalizePaymentIntentForContext(paymentIntentId, cartId)
    Facade->>IntentSvc: getPaymentIntent(owner, paymentIntentId, siteId)
    IntentSvc->>StripeAPI: retrieve PaymentIntent
    IntentSvc->>Tx: registerPaymentIntent(owner, paymentIntentData)
    Facade->>Tx: markPaymentIntentSucceeded(owner, paymentIntentData)
    Facade->>CheckoutFacade: placeOrder()
    CheckoutFacade-->>Facade: OrderData
    OCC-->>Component: OrderWsDTO
    Component->>Router: route to order confirmation
  else status is processing
    Component-->>Shopper: show processing message and rely on return/webhook
  else status is failed
    Component-->>Shopper: show retry message
  end
```

Stripe.js performs the browser-side confirmation. SAP Commerce order placement
still happens server-side through the finalize endpoint.

## Redirect Return Path

```mermaid
sequenceDiagram
  autonumber
  participant Stripe as Stripe redirect
  participant Return as StripePaymentElementReturnComponent
  participant Flow as StripeCheckoutFlowService
  participant ElementSvc as StripePaymentElementService
  participant OCC as StripePaymentElementController
  participant StripeJS as Stripe.js
  participant Router as Spartacus routing

  Stripe-->>Return: Redirect with payment_intent, client_secret, cartId
  Return->>Flow: restoreCartContext(cartId)
  Return->>ElementSvc: getPaymentIntent(paymentIntentId, cartId)
  ElementSvc->>OCC: GET /stripe/elements/intent/{paymentIntentId}
  OCC-->>ElementSvc: PaymentIntent bootstrap data
  Return->>ElementSvc: retrievePaymentIntent(publishableKey, clientSecret)
  ElementSvc->>StripeJS: retrievePaymentIntent(clientSecret)
  StripeJS-->>Return: PaymentIntent status
  alt succeeded or requires_capture
    Return->>ElementSvc: finalizePaymentIntent(paymentIntentId, cartId)
    ElementSvc->>OCC: POST /stripe/elements/intent/{paymentIntentId}/finalize
    OCC-->>Return: OrderWsDTO
    Return->>Router: route to order confirmation
  else processing or requires_action
    Return-->>Return: show pending state
  else failed
    Return-->>Return: show failure state
  end
```

## Finalizable State

The storefront treats `succeeded` and `requires_capture` as successful
PaymentIntent statuses. The facade re-fetches the PaymentIntent server-side and
validates ownership before marking the SAP Commerce transaction as accepted and
placing the order.
