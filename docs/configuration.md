# Configuration

The connector reads Stripe configuration from SAP Commerce properties through
`DefaultStripeConfigurationService`. The property constants live in
`StripeServicesConstants`.

## Required Properties

```properties
stripe.secret.key=<your Stripe secret key>
stripe.publishable.key=<your Stripe publishable key>
stripe.webhook.secret=<your Stripe webhook signing secret>
stripe.checkout.success.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/checkout/return?session_id={CHECKOUT_SESSION_ID}
stripe.checkout.cancel.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/checkout/cancel?session_id={CHECKOUT_SESSION_ID}
stripe.elements.return.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/elements/return
```

The hosted Checkout success URL must include `{CHECKOUT_SESSION_ID}` so Stripe
can append the created session identifier to the return route.

## Site-Specific Overrides

The configuration service supports base-site overrides by appending the site id
to the property name. For example, a site-specific publishable key can use the
same separator pattern as other connector properties:

```properties
stripe.publishable.key.apparel-uk=<site publishable key>
stripe.secret.key.apparel-uk=<site secret key>
stripe.webhook.secret.apparel-uk=<site webhook signing secret>
```

When a site-specific value is absent, the service falls back to the global
property.

## Public Versus Secret Configuration

Only public configuration is returned through OCC:

- `publishableKey`
- `paymentOptionId`
- `paymentMethod`

Secret keys and webhook signing secrets are used only server-side by
`stripeservices` and must not be exposed to the storefront or committed to the
repository.

## Return URL Validation

Configured return URLs must be absolute HTTP or HTTPS URLs. This protects the
Stripe redirect and Payment Elements return flows from incomplete storefront
paths.

## Payment Identifiers

The connector uses these payment identifiers:

| Identifier | Value | Usage |
| --- | --- | --- |
| Payment provider | `Stripe` | SAP Commerce payment transaction provider. |
| Hosted Checkout payment option | `stripe-checkout` | Storefront payment option for hosted Checkout. |
| Payment Elements payment option | `stripe-elements` | Storefront payment option for Payment Elements. |
| Hosted Checkout payment method | `card` | Stripe Checkout Session card payment method. |
| Payment Elements payment method | `payment_element` | Payment Elements facade response value. |

## Recipe Defaults

The installer recipe `installer/recipes/b2c_acc_plus_stripe/build.gradle`
contains local-development defaults and includes the Stripe extensions in the
generated setup. Treat those values as local placeholders, not production
secrets.

## Backoffice Visibility

`stripebackoffice` resolves the same global and site-specific properties for
operational visibility. Secret values are masked before display.
