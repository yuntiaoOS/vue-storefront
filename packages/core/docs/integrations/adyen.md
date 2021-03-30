# Adyen <Badge text="Enterprise" type="info" />

::: warning Paid feature
This feature is part of the Enterprise version. Please [contact our team](https://www.vuestorefront.io/contact/sales) if you'd like to use it in your project.
:::

## Introduction

This package provides integration with [Adyen](https://www.adyen.com/). For more information about this topic, please refer to the [payment providers](../guide/checkout.html#payment-providers) page.

## Installation

1. Install required packages:

```sh
yarn add @vsf-enterprise/adyen
```

2. Add `@vsf-enterprise/adyen` to raw sources:
```js
// nuxt.config.js

export default {
  buildModules: [
    ['@vue-storefront/nuxt', {
      coreDevelopment: true,
      useRawSource: {
        dev: [
          '@vue-storefront/commercetools',
          '@vue-storefront/core',
          '@vsf-enterprise/adyen'
        ],
        prod: [
          '@vue-storefront/commercetools',
          '@vue-storefront/core',
          '@vsf-enterprise/adyen'
        ]
      }
    }]
  ]
};
```

3. Register `@vsf-enterprise/adyen/nuxt` module with following configuration:

```js
// nuxt.config.js

export default {
  modules: [
    ['@vsf-enterprise/adyen/nuxt', {
      availablePaymentMethods: [
        'scheme',
        'paypal'
      ],
      clientKey: '<ADYEN_CLIENT_KEY>',
      environment: 'test',
      methods: {
        paypal: {
          merchantId: '<PAYPAL_MERCHANT_ID>',
          intent: 'capture'
        }
      }
    }]
  ]
};
```

* `availablePaymentMethods` - An array of available payment methods. Here you can find [available values](https://docs.adyen.com/payment-methods).
* `clientKey` - Your client's key. Here you can find information [how to find it](https://docs.adyen.com/development-resources/client-side-authentication#get-your-client-key).
* `environment` - `test` or `live`
* `methods.paypal`:
  * `intent`: `capture` to take cash immediately or `authorize` to delay it.
  * `merchantId`: 13-chars code that identifies your merchant account. Here you can find information [how to find it](https://www.paypal.com/us/smarthelp/article/FAQ3850).


4. Add `@vsf-enterprise/adyen/server` integration to the middleware with following configruation:
```js
// middleware.config.js

adyen: {
  location: '@vsf-enterprise/adyen/server',
  configuration: {
    ctApi: {
      apiHost: '<CT_HOST_URL>',
      authHost: '<CT_AUTH_URL>',
      projectKey: '<CT_PROJECT_KEY>',
      clientId: '<CT_CLIENT_ID>',
      clientSecret: '<CT_CLIENT_SECRET>',
      scopes: [
        'manage_project:<CT_PROJECT_KEY>'
      ]
    },
    adyenMerchantAccount: '<ADYEN_MERCHANT_ACCOUNT>',
    origin: 'http://localhost:3000',
    returnUrl: 'http://localhost:3000/api/adyen/cardAuthAfterRedirect',
    buildRedirectUrlAfter3ds1 (paymentAndOrder, succeed) {
      let redirectUrl = `/checkout/thank-you?order=${paymentAndOrder.order.id}`;
      if (!succeed) {
        redirectUrl += '&error=authorization-failed';
      }
      return redirectUrl;
    }
  }
}
```

* `configuration`:
  * `ctApi` - You need `manage_project` scope to make it work properly, rest information for this property you could find [there](../commercetools/getting-started.html#configuring-your-commercetools-integration).
  * `adyenMerchantAccount` - Name of your Adyen's merchant account
  * `origin` - URL of your frontend. You could check it by printing out `window.location.origin`
  * `returnUrl` - URL where user will be redirected after 3DS1 Auth. Here we are making some payment finalization operations server side and calling `buildRedirectUrlAfter3ds1` which tells the server where the user should be redirect.
  * `buildRedirectUrlAfter3ds1` - `(paymentAndOrder: PaymentAndOrder, succeed: boolean) => string` - A method that tells the server where to redirect the user from `returnUrl`. You can test it with [these cards](https://docs.adyen.com/development-resources/test-cards/test-card-numbers#test-3d-secure-authentication)

```ts
type PaymentAndOrder = Payment & { order: Order }
```

HERE I WILL PUT A DIAGRAM, STAY TUNED


5. Add `origin` to allowed origins in Adyen's dashboard. You can do it in the same place where you looked for the `clientKey`.

6. Commercetools shares [Adyen integration](https://github.com/commercetools/commercetools-adyen-integration) which should be deployed as an independent app. We recommend to deploy it as a Google Function or a AWS Lambda.

::: warning
Make sure to deploy both [extension](https://github.com/commercetools/commercetools-adyen-integration/tree/master/extension) and [notification](https://github.com/commercetools/commercetools-adyen-integration/tree/master/notification) module.
:::

::: warning
You will also need to configure both modules. Check readme of [the repository](https://github.com/commercetools/commercetools-adyen-integration) for details.
:::

7. Use `PaymentAdyenProvider.vue` as a last step of the checkout process.
```vue
<PaymentAdyenProvider
  :afterPay="afterPayAndOrder"
/>
```

`afterPay` is called just after payment is authorized and order placed. Here you could make actions like redirect to the thank you page and clear a cart.
```js
const afterPayAndOrder = async (order) => {
  context.root.$router.push(`/checkout/thank-you?order=${order.id}`);
  setCart(null);
};
```

### Paypal configuration
Configuration of PayPal is well-described in Adyen's documentation, please use [it](https://docs.adyen.com/payment-methods/paypal/web-drop-in).

## API
`@vsf-enterprise/adyen` exports a useAdyen composable.

You can import a [VSF Payment Provider](../commercetools/getting-started.html#configuring-your-commercetools-integration) component from the `@vsf-enterprise/adyen/src/PaymentAdyenProvider`.

## Composable
`useAdyen` composable returns a few properties and methods.

#### Properties
* `error` - _AdyenError_ - errors' state of asynchronous methods.
* `loading` - Boolean that informs if composable is performing some asynchronous method right now.
* `paymentObject` - Object of payment created in the Commercetools. It is updated by `createContext`, `payAndOrder`, `submitAdditionalPaymentDetails` methods.

```ts
interface AdyenError {
  submitAdditionalPaymentDetails: Error | null,
  createContext: Error | null,
  payAndOrder: Error | null
}
```

#### Methods
* `createContext` - Loads a cart, then fetching available payment methods for the loaded cart. At the end, a method stores a response inside `paymentObject`.
* `buildDropinConfiguration` - `(config: AdyenConfigBuilder): any` - Builds a configuration object for Adyen's Web Drop-In
* `payAndOrder` - Setting value of custom field `makePaymentRequest` in the Commercetools' payment. Commercetools will send it to the Adyen and give you the response. At the end, a method stores a response inside `paymentObject`.
* `submitAdditionalPaymentDetails` - Setting value of custom field `submitAdditionalPaymentDetailsRequest` in the Commercetools' payment. Commercetools will send it to the Adyen and give you the response. At the end, a method stores a response inside `paymentObject`.

```ts
interface AdyenConfigBuilder {
  paymentMethodsResponse,
  onChange = (state, component) => {},
  onSubmit = (state, component) => {},
  onAdditionalDetails = (state, component) => {},
  onError = (state) => {}
}
```


## Components
`PaymentAdyenProvider`
Payment provider component that fetches available payment methods, mounts Adyen's Web Drop-In. Component takes care of whole payment flow. It allows to hook into some events by passing functions via props.
* `beforeLoad` - `config => config` - called with built configuration from the `buildDropinConfiguration`, just before creating instance of `AdyenCheckout` and mounting a Drop-In
* `beforePay` - `stateData => stateData` - called inside `onSubmit` of Drop-In, just before calling `payAndOrder`. Here we can modify the payload that will be passed.
* `afterPay` - `paymentAndObject: PaymentAndOrder => void` - called after we got result code `Authorized` from the Adyen, and an order has been placed
* `afterSelectedDetailsChange` - called inside `onChange` of Adyen's Drop-In
* `onError` - `(data: { action: string, error: Error | string }) => void` - called when we got an error from either Adyen or our API

## Placing an order
If transaction is authorized, server's controller for `payAndOrder`/`submitAdditionalPaymentDetails` will place an order in Commercetools and apply `order` object to the response. Thanks to that, we have only one request from the client to make both a payment and an order.