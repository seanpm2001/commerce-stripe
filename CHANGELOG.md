# Release Notes for Stripe for Craft Commerce

## 4.0.1 - 2023-09-28
- Fixed a PHP error that occurred when switching a subscription’s plan.

## 4.0.0 - 2023-09-13

- Added support for all of Stripe’s payment methods, including Apple Pay and Google Wallet. ([#223](https://github.com/craftcms/commerce-stripe/issues/223), [#222](https://github.com/craftcms/commerce-stripe/issues/222),[#212](https://github.com/craftcms/commerce-stripe/issues/212))
- Added support for [Stripe Billing](https://stripe.com/billing).
- Added support for [Stripe Checkout](https://stripe.com/payments/checkout).
- Added support for syncing customer payment methods.
- Plans are now kept in sync with Stripe plans. ([#240](https://github.com/craftcms/commerce-stripe/issues/240))
- Customer information is now kept in sync with Stripe customers.
- Improved logging.
- Stripe now uses the `2022-11-15` version of the Stripe API.
- Added the `commerce-stripe/customers/billing-portal-redirect` action.
- Added the `commerce-stripe/customers/create-setup-intent` action.
- Added the `commerce-stripe/sync/payment-methods` command.
- Added `craft\commerce\stripe\events\BuildSetupIntentRequestEvent`.
- Added `craft\commerce\stripe\gateways\PaymentIntents::getBillingPortalUrl()`.
- Removed `craft\commerce\stripe\base\Gateway::normalizePaymentToken()`.
- Removed `craft\commerce\stripe\events\BuildGatewayRequestEvent::$metadata`. `BuildGatewayRequestEvent::$request` should be used instead.
- Deprecated the `commerce-stripe/default/fetch-plans` action.
- Deprecated creating new payment sources via the `commerce/subscriptions/subscribe` action. 
- Fixed a bug where `craft\commerce\stripe\base\SubscriptionGateway::getSubscriptionPlans()` was returning incorrectly-formatted data.

## 3.1.1 - 2023-05-10

- Stripe customers’ default payment methods are now kept in sync with Craft users’ primary payment sources. ([#235](https://github.com/craftcms/commerce-stripe/issues/235))
- Added `craft\commerce\stripe\services\Customers::EVENT_BEFORE_CREATE_CUSTOMER`. ([#233](https://github.com/craftcms/commerce-stripe/pull/233))
- Added `craft\commerce\stripe\events\SubscriptionRequestEvent::$plan`, which will be set to the plan being subscribed to. ([#141](https://github.com/craftcms/commerce-stripe/pull/141))

## 3.1.0 - 2022-01-29

- Added the `commerce-stripe/reset-data` command.

## 3.0.3 - 2022-11-23

### Fixed
- Fixed a PHP error that occurred when switching a subscription’s plan. ([#3035](https://github.com/craftcms/commerce/issues/3035))

## 3.0.2 - 2022-11-17

### Added
- Added `craft\commerce\stripe\gateways\PaymentIntents::EVENT_BEFORE_CONFIRM_PAYMENT_INTENT`. ([#221](https://github.com/craftcms/commerce-stripe/pull/221))

### Fixed
- Fixed a PHP error that occurred where switching a subscription plan.

## 3.0.1 - 2022-06-16

### Fixed
- Fixed a bug where billing address wasn’t being sent to Stripe.
- Fixed incorrect type on `craft\commerce\stripe\models\forms\SwitchPlans::$billingCycleAnchor`.

## 3.0.0 - 2022-05-04

### Added
- Added Craft CMS 4 and Craft Commerce 4 compatibility.
- Payment Intents gateway settings now support environment variables.

### Removed
- Removed the Charge gateway.

### Fixed
- Fixed an error that would occur when calling `craft\commerce\stripe\models\PaymentIntent::getTransaction()`.
