# Method Call Sequences

This page focuses on method-level calls between the main classes. The diagrams
use implementation names so maintainers can move from the docs to the code
quickly.

## Hosted Checkout Session Creation

```mermaid
sequenceDiagram
  autonumber
  participant Component as StripeCheckoutPaymentComponent.continueToStripe
  participant Flow as StripeCheckoutFlowService
  participant StoreSvc as StripeCheckoutService
  participant Controller as StripeCheckoutController.createCheckoutSession
  participant CartCtx as AbstractStripeCartContextController
  participant Facade as DefaultStripeCheckoutFacade.createCheckoutSessionForCart
  participant Service as DefaultStripeCheckoutSessionService.createCheckoutSession
  participant Client as DefaultStripeClientFactory.createCheckoutSession
  participant Tx as DefaultStripePaymentTransactionService.registerCheckoutSession

  Component->>Flow: getActiveCartIdentifier()
  Component->>Flow: assignPaymentOptionIfSupported(paymentOptionId, cartId)
  Component->>StoreSvc: createCheckoutSession(cartId)
  StoreSvc->>Controller: POST /stripe/checkout/session
  Controller->>CartCtx: loadCartForContext(userId, cartId, message)
  Controller->>Facade: createCheckoutSessionForCart()
  Facade->>Service: createCheckoutSession(getSessionCart())
  Service->>Service: buildSessionCreateParams(order, siteId)
  Service->>Client: createCheckoutSession(secretKey, params)
  Service->>Tx: registerCheckoutSession(order, sessionData)
  Tx-->>Service: authorization entry persisted
  Service-->>Facade: StripeCheckoutSessionData
  Facade-->>Controller: StripeCheckoutSessionFacadeData
  Controller-->>Component: StripeCheckoutSessionWsDTO
```

## Hosted Checkout Finalize

```mermaid
sequenceDiagram
  autonumber
  participant Return as StripeCheckoutReturnComponent.placeOrder
  participant StoreSvc as StripeCheckoutService.finalizeCheckoutSession
  participant Controller as StripeCheckoutController.finalizeCheckoutSession
  participant Facade as DefaultStripeCheckoutFacade.finalizeCheckoutSessionForContext
  participant Service as DefaultStripeCheckoutSessionService.getCheckoutSession
  participant Tx as DefaultStripePaymentTransactionService.findOrderByPaymentReference
  participant Support as AbstractStripeCheckoutFacadeSupport.placeOrder
  participant CheckoutFacade as CheckoutFacade.placeOrder

  Return->>StoreSvc: finalizeCheckoutSession(sessionId, cartId, orderCode)
  StoreSvc->>Controller: POST /stripe/checkout/session/{sessionId}/finalize
  Controller->>Facade: finalizeCheckoutSessionForContext(sessionId, reference)
  Facade->>Facade: requireContextCode(reference, sessionId)
  Facade->>Service: getCheckoutSession(sessionId, siteId, contextCode)
  Service->>Service: validateSessionOwnership(sessionId, session, siteId, contextCode)
  Facade->>Facade: resolveCheckoutOwner(sessionId, contextCode, sessionData)
  Facade->>Tx: findOrderByPaymentReference(contextCode, sessionId)
  alt Owner is OrderModel
    Facade-->>Controller: getOrderData(orderModel)
  else Owner is CartModel
    Facade->>Support: placeOrder(cart)
    Support->>CheckoutFacade: placeOrder()
    CheckoutFacade-->>Support: OrderData
    Support-->>Facade: OrderData
  end
```

## PaymentIntent Creation or Reuse

```mermaid
sequenceDiagram
  autonumber
  participant Component as StripeCheckoutPaymentComponent.ensurePaymentElementsReady
  participant ElementSvc as StripePaymentElementService.createPaymentIntent
  participant Controller as StripePaymentElementController.createPaymentIntent
  participant Facade as DefaultStripePaymentElementFacade.createPaymentIntentForCart
  participant IntentSvc as DefaultStripePaymentIntentService.createOrUpdatePaymentIntent
  participant Tx as DefaultStripePaymentTransactionService
  participant Client as DefaultStripeClientFactory

  Component->>ElementSvc: createPaymentIntent(cartId)
  ElementSvc->>Controller: POST /stripe/elements/intent
  Controller->>Facade: createPaymentIntentForCart()
  Facade->>IntentSvc: createOrUpdatePaymentIntent(cart)
  IntentSvc->>Tx: findLatestOpenPaymentIntentId(order)
  alt reusable existing PaymentIntent
    IntentSvc->>Client: getPaymentIntent(secretKey, existingId)
  else updateable existing PaymentIntent
    IntentSvc->>Client: updatePaymentIntent(secretKey, existingId, updateParams)
  else no reusable PaymentIntent
    IntentSvc->>Client: createPaymentIntent(secretKey, createParams)
  end
  IntentSvc->>Tx: registerPaymentIntent(order, paymentIntentData)
  IntentSvc-->>Facade: StripePaymentIntentData
  Facade->>Facade: convert(order, paymentIntentData)
  Facade-->>Controller: StripePaymentElementFacadeData
```

## PaymentIntent Finalize

```mermaid
sequenceDiagram
  autonumber
  participant Component as StripeCheckoutPaymentComponent.finalizePaymentIntent
  participant ElementSvc as StripePaymentElementService.finalizePaymentIntent
  participant Controller as StripePaymentElementController.finalizePaymentIntent
  participant CartCtx as AbstractStripeCartContextController
  participant Facade as DefaultStripePaymentElementFacade.finalizePaymentIntentForContext
  participant IntentSvc as DefaultStripePaymentIntentService.getPaymentIntent
  participant Tx as DefaultStripePaymentTransactionService
  participant Support as AbstractStripeCheckoutFacadeSupport.placeOrder

  Component->>ElementSvc: finalizePaymentIntent(paymentIntentId, cartId)
  ElementSvc->>Controller: POST /stripe/elements/intent/{paymentIntentId}/finalize
  Controller->>CartCtx: loadCartForFinalizeContextIfPossible(userId, cartId, message)
  Controller->>Facade: finalizePaymentIntentForContext(paymentIntentId, cartId)
  Facade->>Facade: resolveCheckoutOwner(paymentIntentId, cartId)
  Facade->>IntentSvc: getPaymentIntent(owner, paymentIntentId, currentSiteId)
  IntentSvc->>IntentSvc: validateOwnership(owner, paymentIntent, siteId)
  IntentSvc->>Tx: registerPaymentIntent(owner, paymentIntentData)
  Facade->>Facade: isFinalizablePaymentIntent(paymentIntentData)
  Facade->>Tx: markPaymentIntentSucceeded(owner, paymentIntentData)
  Facade->>Support: placeOrder(cart)
```

## Webhook Dispatch

```mermaid
sequenceDiagram
  autonumber
  participant Controller as StripeWebhookController.handleWebhook
  participant Service as DefaultStripeWebhookService.handleWebhook
  participant Client as DefaultStripeClientFactory.constructEvent
  participant Handler as DefaultStripeWebhookService.handleEvent
  participant Tx as DefaultStripePaymentTransactionService

  Controller->>Service: handleWebhook(payload, signature, siteId)
  Service->>Client: constructEvent(payload, signature, webhookSecret)
  Client-->>Service: verified Event
  Service->>Handler: handleEvent(event, resolvedSiteId)
  alt Checkout Session event
    Handler->>Handler: handleCheckoutSessionEvent(event, session, siteId)
    Handler->>Tx: markCheckoutSessionCompleted or markCheckoutSessionExpired
  else PaymentIntent event
    Handler->>Handler: handlePaymentIntentEvent(event, paymentIntent, siteId)
    Handler->>Tx: markPaymentIntentSucceeded or markPaymentIntentFailed
  end
```

## Refund Creation

```mermaid
sequenceDiagram
  autonumber
  participant Controller as StripeRefundController.createRefund
  participant Facade as DefaultStripeRefundFacade.createRefundForOrder
  participant Lifecycle as DefaultStripePaymentLifecycleService.createRefund
  participant Client as DefaultStripeClientFactory
  participant Tx as DefaultStripePaymentTransactionService.registerRefund

  Controller->>Controller: validateRequest(request)
  Controller->>Facade: createRefundForOrder(code, paymentReference, amount)
  Facade->>Facade: resolveOrder(code)
  Facade->>Lifecycle: createRefund(order, paymentReference, amount)
  Lifecycle->>Lifecycle: resolveRefundPaymentIntentId(secretKey, paymentReference)
  Lifecycle->>Lifecycle: validateRefundOwnership(order, reference, paymentIntentId, secretKey)
  Lifecycle->>Client: createRefund(secretKey, refundParams)
  Client-->>Lifecycle: Refund
  Lifecycle->>Tx: registerRefund(order, paymentReference, refundData)
  Lifecycle-->>Facade: StripeRefundData
```
