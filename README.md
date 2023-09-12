<p align="center"><img src="./src/icon.svg" width="100" height="100" alt="Stripe for Craft Commerce icon"></p>

<h1 align="center">Stripe for Craft Commerce</h1>

This plugin provides a [Stripe](https://stripe.com/) integration for [Craft Commerce](https://craftcms.com/commerce)
supporting [Payment Intents](https://stripe.com/docs/payments/payment-intents) and all payment methods.

## Requirements

- Craft CMS 4.0 or later
- Craft Commerce 4.3 or later
- Stripe [API version](https://stripe.com/docs/api/versioning) '2022-11-15'

## Installation

You can install this plugin from the Plugin Store or using Composer.

#### From the Plugin Store

Go to the Plugin Store in your project’s control panel, search for “Stripe for Craft Commerce”, and choose **Install**
in the plugin’s modal window.

#### With Composer

Open your terminal and run the following commands:

```bash
# go to the project directory
cd /path/to/my-project.test

# tell Composer to load the plugin
composer require craftcms/commerce-stripe

# tell Craft to install the plugin
php craft install/plugin commerce-stripe
```

## Setup

To add a Stripe payment gateway in the Craft control panel, navigate to **Commerce** → **Settings** → **Gateways**, and click **New gateway**.

Your gateway’s **Name** should make sense to administrators _and_ customers (especially if you’re using the example templates).

### Secrets

From the **Gateway** dropdown, select **Stripe**, then provide the following information:

- Publishable API Key
- Secret API Key
- Webhook Signing Secret

Your **Publishable API Key** and **Secret API Key** can be found in (or generated from) your Stripe dashboard, within the **Developers** &rarr; **API Keys** tab. Read more about [Stripe API keys](https://stripe.com/docs/keys).

> [!NOTE]
> To prevent secrets leaking into project config, put them in your `.env` file, then use the special [environment variable syntax](https://craftcms.com/docs/4.x/config/#control-panel-settings) in the gateway settings.

Stripe provides different keys for testing—use those until you are ready to launch, then replace the testing keys in the live server’s `.env` file.

### Webhooks

Once the gateway has been saved (and it has an ID), revisiting its edit screen will reveal a **Webhook URL** that can be copied into a new webhook in your Stripe dashboard.

#### Local Development

When you've set up that URL in the Stripe dashboard, you can view the signing secret in its settings. Enter this value in your Stripe gateway settings in the Webhook Signing Secret field. To use webhooks, the Webhook Signing Secret setting is required.

We recommend enabling all available events for the webhook. Unnecessary events will be ignored by the plugin.

> [!NOTE]
> We advise using the [Stripe CLI](https://stripe.com/docs/stripe-cli) in development. See Stripe’s [Testing Webhooks](https://stripe.com/docs/webhooks/test) article for more information.

## Uprading from 3.x

Version 4.0 is largely backward-compatible with 3.x. Review the following sections to ensure your site (and any customizations) remain functional.

### Payment Forms

To support the full array of payment methods available via Stripe (like Apple Pay and Google Pay), the plugin makes exclusive use of the Payment Intents API.

Historically, `gateway.getPaymentFormHtml()` has output a basic form for tokenizing a credit card, client-side—it would then submit only the resulting token to the `commerce/payments/pay` action, and capture the payment from the back-end. **Custom payment forms that use this process will continue to work.**

Now, output is a more flexible [Payment Element](https://stripe.com/docs/payments/payment-element) form, which makes use of Stripes modern [Payment Intents](https://stripe.com/docs/api/payment_intents) API. The process looks something like this:

1. A request is submitted in the background to `commerce/payments/pay` (without a payment method);
1. Commerce creates a Payment Intent with some information about the order, then sets up an internal `Transaction` record to track its status;
1. The Payment Intent’s [`client_secret`](https://stripe.com/docs/api/payment_intents/object#payment_intent_object-client_secret) is returned to the front-end;
1. The Stripe JS SDK is initialized with the secret, and the customer is able to select from the available payment methods;

Details on how to configure the new [payment form](#creating-a-stripe-payment-form) are below.

## Configuration Settings

These options are set via `config/commerce-stripe.php`.

### `chargeInvoicesImmediately`

For subscriptions with automatic payments, Stripe creates an invoice 1-2 hours before attempting to charge it. By setting this to `true`, you can force Stripe to charge this invoice immediately.

> [!WARNING]
> This setting affects **all** Stripe gateways in your Commerce installation.

## Subscriptions

### Creating a Subscription Plan

1. Every subscription plan must first be [created in the Stripe dashboard](https://dashboard.stripe.com/test/subscriptions/products).
1. In the Craft control panel, navigate to **Commerce** → **Settings** → **Subscription plans** and create a new subscription plan.

### Subscription Options

In addition to the values you POST to Commerce’s `commerce/subscriptions/subscribe` action, the Stripe gateway supports these options.

#### `trialDays`

The first full billing cycle will start once the number of trial days lapse. Default value is `0`.

### Cancellation Options

#### `cancelImmediately`

If this parameter is set to `true`, the subscription is canceled immediately. Stripe considers this a simultaneous cancellation and “deletion” (as far as webhooks are concerned)—but a record of the subscription remains available. By default, the subscription is marked as canceled and will end along with the current billing cycle. Defaults to `false`.

> [!NOTE]
> Immediately canceling a subscription means it cannot be reactivated.

### Plan-Switching Options

#### `prorate`

If this parameter is set to `true`, the subscription switch will be [prorated](https://stripe.com/docs/billing/subscriptions/upgrade-downgrade#proration). Defaults to `false`.

#### `billImmediately`

If this parameter is set to `true`, the subscription switch is billed immediately. Otherwise, the cost (or credit,
if `prorate` is set to `true` and switching to a cheaper plan) is applied to the next invoice.

> [!WARNING]
> If the billing periods differ, the plan switch will be billed immediately and this parameter will be ignored.

### Reactivation Options

There are no customizations available when reactivating a subscription.

## Events

The plugin provides several events you can use to modify the behavior of your integration.

### Payment Events

#### `buildGatewayRequest`

Plugins get a chance to provide additional metadata when communicating with Stripe in the course of creating a PaymentIntent.

This gives you near-complete control over the data that Stripe sees, with the following considerations:

- Changes to the `Transaction` model (available via the event’s `transaction` property) will not be saved;
- The gateway automatically sets `order_id`, `order_number`, `order_short_number`, `transaction_id`, `transaction_reference`, `description`, and `client_ip` [metadata](https://stripe.com/docs/payments/payment-intents#storing-information-in-metadata) keys;
- Changes to the `amount` and `currency` keys under the `request` property will be ignored, as these are essential to the gateway functioning in a predictable way;

```php
use craft\commerce\models\Transaction;
use craft\commerce\stripe\events\BuildGatewayRequestEvent;
use craft\commerce\stripe\gateways\PaymentIntents;
use yii\base\Event;

Event::on(
    PaymentIntents::class,
    PaymentIntents::EVENT_BUILD_GATEWAY_REQUEST,
    function(BuildGatewayRequestEvent $e) {
        /** @var Transaction $transaction */
        $transaction = $e->transaction;
        $order = $transaction->getOrder();

        $e->request['metadata']['shipping_method'] = $order->shippingMethodHandle;
    }
);
```

> [!NOTE]
> [Subscription events](#subscription-events) are handled separately.

#### `receiveWebhook`

In addition to the generic [`craft\commerce\services\Webhooks::EVENT_BEFORE_PROCESS_WEBHOOK` event](https://docs.craftcms.com/commerce/api/v4/craft-commerce-services-webhooks.html#event-before-process-webhook), you can listen to `craft\commerce\stripe\gateways\PaymentIntents::EVENT_RECEIVE_WEBHOOK`. This event is only emitted after validating a webhook’s authenticity—but it doesn’t make any indication about whether an action was taken in response to it.

```php
use craft\commerce\stripe\events\ReceiveWebhookEvent;
use craft\commerce\stripe\gateways\PaymentIntents;
use yii\base\Event;

Event::on(
    PaymentIntents::class,
    PaymentIntents::EVENT_RECEIVE_WEBHOOK,
    function(ReceiveWebhookEvent $e) {
        if ($e->webhookData['type'] == 'charge.dispute.created') {
            if ($e->webhookData['data']['object']['amount'] > 1000000) {
                // Be concerned that a USD 10,000 charge is being disputed.
            }
        }
    }
);
```

`webhookData` will always have a `type` key, which determines the schema of everything within `data`. Check the Stripe documentation for what kinds of data to expect.

### Subscription Events

#### `createInvoice`

Plugins get a chance to do something when an invoice is created on the Stripe gateway.

```php
use craft\commerce\stripe\events\CreateInvoiceEvent;
use craft\commerce\stripe\gateways\PaymentIntents;
use yii\base\Event;

Event::on(
    PaymentIntents::class, 
    PaymentIntents::EVENT_CREATE_INVOICE,
    function(CreateInvoiceEvent $e) {
        if ($e->invoiceData['billing'] === 'send_invoice') {
            // Forward this invoice to the accounting department.
        }
    }
);
```

#### `beforeSubscribe`

Plugins get a chance to tweak subscription parameters when subscribing.

```php
use craft\commerce\stripe\events\SubscriptionRequestEvent;
use craft\commerce\stripe\gateways\PaymentIntents;
use yii\base\Event;

Event::on(
    PaymentIntents::class,
    PaymentIntents::EVENT_BEFORE_SUBSCRIBE,
    function(SubscriptionRequestEvent $e) {
        /** @var craft\commerce\base\Plan $plan */
        $plan = $e->plan;

        /** @var craft\elements\User $user */
        $user = $e->user;

        // Add something to the metadata:
        $e->parameters['metadata']['name'] = $user->fullName;
        unset($e->parameters['metadata']['another_property']);
    }
);
```

## Billing Portal

You can now generate a link to the Stripe billing portal for customers to manage their credit cards and plans.

```twig
<a href={{ gateway.billingPortalUrl(currentUser) }}">Manage your billing account</a>
```

Pass a `returnUrl` parameter to return the customer to a specific on-site page after they have finished:

```twig
{{ gateway.billingPortalUrl(currentUser, 'myaccount') }}
```

A specific Stripe [customer portal](https://stripe.com/docs/customer-management/activate-no-code-customer-portal) configuration can be chosen with the third `configurationId` argument. This value must agree with a preexisting configuration in the associated Stripe account:

```twig
{{ gateway.billingPortalUrl(currentUser, 'myaccount', 'config_12345') }}
```

> [!NOTE]
> Logged-in users can be redirected via the `commerce-stripe/customers/billing-portal-redirect` action. The `configurationId` parameter is not supported when using this method.

## Syncing Customer Payment Methods

Payment methods created directly in Stripe are now synchronized back to Commerce. Webhooks must be configured for this to work as expected!

To do an initial sync, run the `commerce-stripe/sync/payment-sources` console command:

```bash
php craft commerce-stripe/sync/payment-sources
```

## Creating a Stripe Payment Form

To render a Stripe Elements payment form, get a reference to the gateway, then call its `getPaymentFormHtml()` method:

```twig
{% set cart = craft.commerce.carts.cart %}
{% set gateway = cart.gateway %}

{% namespace gateway.handle|commercePaymentFormNamespace %}
  {{ gateway.getPaymentFormHtml({})|raw }}
{% endnamespace %}
```

This assumes you have provided a means of selecting the gateway in a prior checkout step—or, if your store only uses a single gateway, you may get a static reference to the gateway and set it during payment:

```twig
{% set gateway = craft.commerce.gateways.getGatewayByHandle('myStripeGateway') %}

{# Include *outside* the namespaced form inputs: #}
{{ hiddenInput('gatewayId', gateway.id) }}

{% namespace gateway.handle|commercePaymentFormNamespace %}
  {{ gateway.getPaymentFormHtml({})|raw }}
{% endnamespace %}
```

Regardless of how you use this output, it will automatically register all the necessary Javascript for Stripe Elements to function properly.

### Payment Form Contexts

Commerce uses payment forms in three places:

- `commerce/payments/pay`.
- `commerce/payment-sources/add`.
- `commerce/subscriptions/subscribe`. (You should only output the payment form HTML in this form if the customer does not already have a primary payment source)

If you embed it into a form tag with an action parameter of `commerce/payments/pay`, it will be a payment flow which
will create a transaction in Commerce with a redirect status and create a payment intent in Stripe.

If you want to read how to create the legacy payment form with stripe.js read
the [old README](https://github.com/craftcms/commerce-stripe/blob/e0325e98594cc4824b3e2788ac0573c8d04a71d5/README.md#creating-a-stripe-payment-form-for-the-payment-intents-gateway).

## Customizing the Stripe Payment Form

`getPaymentFormHtml()` accepts one argument, an array with one or more of the following keys:

### `paymentFormType`

#### `elements`

Renders a Stripe Elements form with all the payment method types [enabled in your Stripe Dashboard](https://stripe.com/docs/payments/customize-payment-methods). Some methods may be hidden if the order total or currency don’t meet the method’s criteria—or if they aren’t supported in development environments.

```twig
{% set params = {
  paymentFormType: 'elements',
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
```

Remember, some payment methods like Apple Pay will only show up in the Safari browser.

#### `checkout`

This generates a form ready to redirect to Stripe Checkout. This can only be used inside a `commerce/payments/pay` form.

This option ignores all other params.

```twig
{% set params = {
  paymentFormType: 'checkout',
} %}
{{ gateway.getPaymentFormHtml(params) }}
```

### `appearance`

You can pass an array of appearance options when using the `elements` or `card` payment form type.

This expects data for the [Elements Appearance API](https://stripe.com/docs/elements/appearance-api).

```twig
{% set params = {
  paymentFormType: 'elements',
  appearance: {
    theme: 'stripe'
  }
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
```

```twig
{% set params = {
  paymentFormType: 'elements',
  appearance: {
    theme: 'night',
    variables: {
      colorPrimary: '#0570de'
    }
  }
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
```

### `elementOptions`

This allows you to modify
the [Payment Element options](https://stripe.com/docs/js/elements_object/create_payment_element#payment_element_create-options)

```twig
{% set params = {
  paymentFormType: 'elements',
  elementOptions: {
    layout: {
      type: 'tabs',
      defaultCollapsed: false,
      radios: false,
      spacedAccordionItems: false
    }
  }
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
```

Default value:

```twig
elementOptions: {
    layout: {
      type: 'tabs'
    }
  }
```

### `submitButtonClasses` and `submitButtonText`

These control the button rendered at the bottom of the form.

```twig
{% set params = {
  paymentFormType: 'elements',
  submitButtonClasses: 'cursor-pointer rounded px-4 py-2 inline-block bg-blue-500 hover:bg-blue-600 text-white hover:text-white my-2',
  submitButtonText: 'Pay',
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
```

### `errorMessageClasses`

Above the form is a location where error messages are displayed when an error occurs. You can control the classes of
this element.

```
{% set params = {
  paymentFormType: 'elements',
  submitButtonClasses: 'cursor-pointer rounded px-4 py-2 inline-block bg-blue-500 hover:bg-blue-600 text-white hover:text-white my-2',
  submitButtonText: 'Pay',
  errorMessageClasses: 'bg-red-200 text-red-600 my-2 p-2 rounded',
} %}
{{ cart.gateway.getPaymentFormHtml(params) }}
  
```
