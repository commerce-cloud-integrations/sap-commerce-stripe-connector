# Installation and Deployment

This release targets SAP Commerce Cloud 2211 JDK21 and SAP Composable
Storefront 2211.31.

## Manual SAP Commerce Installation

1. Copy the Stripe SAP Commerce extensions:

   ```text
   hybris/bin/modules/stripe
   ```

   into your SAP Commerce installation under:

   ```text
   ${HYBRIS_BIN_DIR}/modules/stripe
   ```

2. Add the module path to `localextensions.xml`:

   ```xml
   <path autoload="true" dir="${HYBRIS_BIN_DIR}/modules/stripe"/>
   ```

3. Configure Java 21:

   ```bash
   source "$HOME/.sdkman/bin/sdkman-init.sh"
   sdk use java 21.0.2-tem
   java -version
   ```

4. Configure the Stripe properties in `local.properties`.

5. Build SAP Commerce:

   ```bash
   cd hybris/bin/platform
   . ./setantenv.sh
   ant clean all
   ```

6. Start SAP Commerce and update or initialize the system as appropriate for
   the target environment.

## Installer Recipe

The repository includes a Gradle recipe:

```text
installer/recipes/b2c_acc_plus_stripe/build.gradle
```

On a clean installation, copy this repository's `hybris` and
`installer/recipes` folders into the corresponding SAP Commerce installation
paths, then run:

```bash
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe setup
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe initialize
HYBRIS_HOME/installer$ ./install.sh -r b2c_acc_plus_stripe start
```

## SAP Commerce Cloud Deployment

For SAP Commerce Cloud, include the Stripe connector extensions in the cloud
repository and adapt `manifest.json` for your environment. Keep real Stripe
keys and webhook signing secrets in environment configuration, not in git.

The release manifest declares:

- SAP Commerce 2211 JDK21 compatibility
- Java 21
- SAP Composable Storefront 2211.31
- Angular 17.3
- Stripe Java SDK 33.1.0
- Stripe.js 5.10.0
- extension list under `hybris/bin/modules/stripe`
- storefront package path
- local webhook forwarding command

## Storefront Package Installation

The storefront package lives at:

```text
js-storefront/stripe-spartacus-connector
```

Wire `StripeCheckoutModule` into the target composable storefront and ensure
that the OCC base site, OAuth client, and guest checkout settings match the SAP
Commerce system.

## Webhook Forwarding

Use Stripe CLI for local validation:

```bash
stripe listen --forward-to http://127.0.0.1:9001/stripeevents/webhooks/stripe
```

Fallback for local HTTPS:

```bash
stripe listen --skip-verify --forward-to https://127.0.0.1:9002/stripeevents/webhooks/stripe
```

After starting the listener, copy the webhook signing secret into
`stripe.webhook.secret` or the site-specific override.

## Deployment Checklist

- SAP Commerce runs on Java 21.
- Stripe extensions are present in `localextensions.xml` or the cloud manifest.
- `stripe.secret.key`, `stripe.publishable.key`, and `stripe.webhook.secret`
  are configured.
- Hosted Checkout success and cancel URLs point to valid storefront routes.
- Payment Elements return URL points to the storefront elements return route.
- Storefront imports `StripeCheckoutModule`.
- Stripe webhook endpoint is reachable from Stripe for the target environment.
- No real Stripe secrets are committed.
