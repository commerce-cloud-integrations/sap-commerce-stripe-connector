# Stripe Connector for SAP Commerce Cloud

<p align="center">
  <img src="assets/sap-commerce-cloud.webp" alt="SAP Commerce Cloud" height="112">
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img src="assets/stripe.png" alt="Stripe" height="112">
</p>

The Stripe Connector for SAP Commerce Cloud enables SAP Commerce projects to
accept payments through Stripe using a secure, unified integration for hosted
Stripe Checkout, Stripe Payment Elements, webhook processing, payment lifecycle
handling, and SAP Composable Storefront checkout integration.

This project is not affiliated with, sponsored by, or endorsed by Stripe, Inc.
Stripe is a trademark of Stripe, Inc.

## Supported Stripe Capabilities

The connector supports:

- Hosted Stripe Checkout redirect flow.
- Inline Stripe Payment Elements with PaymentIntent confirmation.
- Stripe webhook handling with signature verification.
- Payment lifecycle and SAP Commerce transaction state updates.
- OCC endpoints for checkout session, PaymentIntent, finalization, and refunds.
- Fulfilment-process hooks for post-payment order handling.
- Backoffice visibility for Stripe connector configuration status.
- SAP Composable Storefront checkout and return routes.

## Documentation

Detailed implementation documentation is available in [docs](docs/README.md).
It covers the SAP Commerce extension architecture, configuration model, OCC
API surface, hosted Checkout flow, Payment Elements flow, webhooks, refunds,
transaction state handling, storefront integration, testing, and
troubleshooting.

## Release Compatibility

This release is compatible with:

- SAP Commerce Cloud 2211 JDK21.
- SAP Commerce REST API (OCC).
- SAP Composable Storefront 2211.31.
- Angular 17.3.
- Java 21.
- Stripe Java SDK 33.1.0.
- Stripe.js 5.10.0.

It is advised to use the latest validated release of this connector.

# Installation and Usage

## Installation of the Connector with Stripe payment functionality

Ensure that your SAP Commerce version is supported by this connector. The
connector contains several SAP Commerce extensions under
`hybris/bin/modules/stripe`:

- `stripeservices`
- `stripefacades`
- `stripeocc`
- `stripeevents`
- `stripefulfilmentprocess`
- `stripebackoffice`
- `stripeocctests`
- `stripetest`

Follow these steps to include the connector in your SAP Commerce application:

1. Clone or download this repository.

2. Copy `hybris/bin/modules/stripe` into the `${HYBRIS_BIN_DIR}/modules`
   directory of your SAP Commerce installation.

3. Add the Stripe module path to `localextensions.xml` after the standard
   `${HYBRIS_BIN_DIR}` path:

   ```xml
   <path autoload="true" dir="${HYBRIS_BIN_DIR}/modules/stripe"/>
   ```

4. Configure Java 21 before running SAP Commerce build or startup commands:

   ```bash
   source "$HOME/.sdkman/bin/sdkman-init.sh"
   sdk use java 21.0.2-tem
   java -version
   ```

5. Configure the required Stripe properties in `local.properties`:

   ```properties
   stripe.secret.key=<your Stripe secret key>
   stripe.publishable.key=<your Stripe publishable key>
   stripe.webhook.secret=<your Stripe webhook signing secret>
   stripe.checkout.success.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/checkout/return?session_id={CHECKOUT_SESSION_ID}
   stripe.checkout.cancel.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/checkout/cancel?session_id={CHECKOUT_SESSION_ID}
   stripe.elements.return.url=http://apparel-uk.local:4200/apparel-uk-spa/en/GBP/stripe/elements/return
   ```

6. Run the SAP Commerce build from `hybris/bin/platform`:

   ```bash
   . ./setantenv.sh
   ant clean all
   ```

7. Start SAP Commerce and update the running system.

The connector extensions do not rely on absolute paths. They can also be placed
under another SAP Commerce extension directory if your project uses a different
layout.

## Installing the Connector using recipes

The connector ships with a Gradle recipe for the SAP Commerce installer.

B2C: `b2c_acc_plus_stripe` with SAP Commerce B2C Accelerator, OCC, and Stripe
connector functionality.

The recipe is available under `installer/recipes`. To use the recipe on a clean
installation, copy this repository's `hybris` and `installer/recipes` folders
into the corresponding SAP Commerce installation paths.

Install the connector using the recipe:

```bash
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe setup
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe initialize
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe start
```

For local installations, the recipe can generate local properties for the
connector. For cloud installations, adapt the included `manifest.json` to your
SAP Commerce Cloud repository and environment configuration.

## Installing on SAP Commerce Cloud

Follow SAP Commerce Cloud deployment guidance for custom code repositories.
Include the Stripe connector extensions in the cloud repository and adapt
`manifest.json` to include the Stripe extensions, required properties, and
environment-specific values.

Do not commit real Stripe API keys or webhook signing secrets. Store those
values in the appropriate SAP Commerce Cloud environment configuration.

# SAP Commerce Composable Frontend

SAP Composable Storefront is the Angular-based JavaScript storefront for SAP
Commerce Cloud and communicates with SAP Commerce through OCC.

This repository includes a reusable storefront package under:

```text
js-storefront/stripe-spartacus-connector
```

The package provides the Stripe checkout payment step and return routes:

- `stripe/checkout/return`
- `stripe/checkout/cancel`
- `stripe/elements/return`

Import `StripeCheckoutModule` in the storefront feature wiring and keep guest
checkout enabled when the checkout CMS flow contains guest checkout login
components.

The default local Spartacus 2211.31 OAuth client is `mobile_android` with client
secret `secret`. Ensure the same OAuth client exists in SAP Commerce or align
the storefront authentication configuration with your own OAuth client.

# Webhooks

Forward Stripe CLI events directly to SAP Commerce:

```bash
stripe listen --forward-to http://127.0.0.1:9001/stripeevents/webhooks/stripe
```

If only HTTPS is available locally, use:

```bash
stripe listen --skip-verify --forward-to https://127.0.0.1:9002/stripeevents/webhooks/stripe
```

# Release Notes

- Hosted Stripe Checkout integration through OCC.
- Stripe Payment Elements integration through OCC PaymentIntent endpoints.
- Stripe webhook endpoint with signature verification.
- Payment lifecycle services, transaction state handling, and fulfilment-process
  hooks.
- Backoffice visibility for Stripe connector configuration status.
- SAP Composable Storefront package for checkout and return flows.
- Compatibility with SAP Commerce Cloud 2211 JDK21 and SAP Composable
  Storefront 2211.31.

# Support

Contact your [commercecloudintegrations.com](https://commercecloudintegrations.com)
team if you have any question, technical problem or feature request for the SAP
Commerce Cloud Connector.

# License

This repository is open source and available under the MIT license. See
[LICENSE](LICENSE).
