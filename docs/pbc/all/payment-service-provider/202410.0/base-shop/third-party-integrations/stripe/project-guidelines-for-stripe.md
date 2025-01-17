---
title: Project guidelines for Stripe
description: Learn the guidelines that are needed for your Spryker projects when it comes to using Stripe throught the Spryker Stripe App Integration.
last_updated: Jul 22, 2024
template: howto-guide-template
related:
   - title: Stripe
     link: docs/pbc/all/payment-service-provider/page.version/base-shop/third-party-integrations/stripe/stripe.html
redirect_from:
   - /docs/pbc/all/payment-service-provider/202311.0/third-party-integrations/stripe/install-stripe.html
   - /docs/pbc/all/payment-service-provider/202311.0/base-shop/third-party-integrations/stripe/install-stripe.html
   - /docs/pbc/all/payment-service-provider/202311.0/base-shop/third-party-integrations/stripe/integrate-stripe.html

---

This document provides guidelines for projects using Stripe through the Stripe app.

## OMS configuration

The complete default payment OMS configuration is available at `vendor/spryker/sales-payment/config/Zed/Oms/ForeignPaymentStateMachine01.xml`.

The payment flow of the default OMS involves authorizing the initial payment. The amount is temporarily blocked when the payment method permits. Then, the OMS sends requests to capture, that is, transfer of the previously blocked amount from the customer's account to the store account.

The `Payment/Capture` command initiates the capture action. By default, this command is initiated when a Back Office user clicks **Ship** on the **Order Overview** page.

Optionally, you can change and configure your own payment OMS based on `ForeignPaymentStateMachine01.xml` from the core package and change this behavior according to your business flow. For more information about OMS configuration, see [Install the Order Management feature](/docs/pbc/all/order-management-system/{{page.version}}/base-shop/install-and-upgrade/install-features/install-the-order-management-feature.html).

To configure your payment OMS based on `ForeignPaymentStateMachine01.xml`, copy `ForeignPaymentStateMachine01.xml` with the `Subprocess` folder to the project root `config/Zed/oms`. Then, change the file's name and the value of `<process name=` in the file.

The following example shows how to configure the order state machine transition from `ready for dispatch` to `payment capture pending`:

<details>
  <summary>State machine example</summary>

```xml
<?xml version="1.0"?>
<statemachine
        xmlns="spryker:oms-01"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="spryker:oms-01 https://static.spryker.com/oms-01.xsd"
>

   <process name="SomeProjectProcess" main="true">

      <!-- other configurations -->

      <states>

         <!-- other states -->

         <state name="payment capture pending" display="oms.state.in-progress"/>

         <!-- other states -->

      </states>

      <transitions>

         <!-- other transitions -->

         <transition happy="true">
            <source>ready for dispatch</source>
            <target>payment capture pending</target>
            <event>capture payment</event>
         </transition>

         <!-- other transitions -->

      </transitions>

      <events>

         <!-- other events -->

         <event name="capture payment" onEnter="true" command="Payment/Capture"/>

         <!-- other events -->

      </events>

   </process>

</statemachine>
```

</details>

By default, the timeout for the payment authorization action is set to seven days. If the order is in the `payment authorization pending` state, after a day the order state is changed to `payment authorization failed`. Another day later, the order is transitioned to the `payment authorization canceled` state.

To decrease or increase timeouts or change the states, update `config/Zed/oms/Subprocess/PaymentAuthorization01.xml`.

For more information on the integration of ACP payment methods with OMS configuration, see [Integrate ACP payment apps with Spryker OMS configuration](/docs/dg/dev/acp/integrate-acp-payment-apps-with-spryker-oms-configuration.html).


## Implementing Stripe for checkout in a headless application

Use this approach for headless applications with third-party frontends.

### Install modules

Install or upgrade the modules to the specified or later versions:
- `spryker/kernel-app:1.2.0`
- `spryker/payment:5.24.0`
- `spryker/payments-rest-api:1.3.0`

### Pre-order payment flow

When Stripe is integrated into a headless application, orders are processed using a pre-order payment flow:

1. The customer either selects Stripe as the payment method or [Stripe Elements](https://docs.stripe.com/payments/elements) is loaded by default.
2. The `InitializePreOrderPayment` Glue API endpoint (`glue.mysprykershop.com/payments`) is called with the following parameters:
  * Payment provider name: Stripe
  * Payment method name: Stripe
  * Payment amount
  * Quote data
3. Back Office sends the quote data and the payment amount to Stripe app through an API.
4. The payment with the given data is persisted in the Stripe app.
5. Stripe app requests `ClientSecret` and `PublishableKey` keys from Stripe through an API.
6. Stripe returns a JSON response with the following parameters:
  * TransactionId
  * ClientSecret
  * PublishableKey
  * Only for marketplaces: AccountId
7. Stripe Elements is rendered on the order summary page. See [Rendering the Stripe Elements on the summary page](#rendering-the-stripe-elements-on-the-summary-page) for rendering Stripe Elements.
8. The customer selects a payment method in Stripe Elements and submits the data.
9. The customer is redirected to the configured `return_url`, which makes an API request to persist the order in the Back Office: `glue.mysprykershop.com/checkout`.
10. The customer is redirected to the application's success page.
11. Stripe app confirms the pre-order payment using the plugin: `\Spryker\Zed\Payment\Communication\Plugin\Checkout\PaymentConfirmPreOrderPaymentCheckoutPostSavePlugin`.
  The `order_reference` is passed to the Stripe app to be connected with `transaction_id`.
12. Stripe app processes the payment and sends a `PaymentUpdated` message to Spryker.
13. Depending on payment status, one of the following messages is returned through an asynchronous API:
  *  Payment is successful: `PaymentAuthorized` message.
  *  Payment is failed: `PaymentAuthorizationFailed` message.
14. Further on, the order is processed through the OMS.

All payment related messages mentioned above are handled by `\Spryker\Zed\Payment\Communication\Plugin\MessageBroker\PaymentOperationsMessageHandlerPlugin`, which is registered in `MessageBrokerDependencyProvider`.


### Example of the headless checkout with Stripe

The Payment selection in this example will be used on the Summary page. The following examples show how to implement the Stripe Payment App in a headless application.

Before the customer is redirected to the summary page, all required data is collected: customer data, addresses, and selected shipment method. When the customer goes to the summary page, to get the data required for rendering the Stripe Elements, the application needs to call the `InitializePreOrderPayment` Glue API endpoint.

#### Pre-order payment initialization

```JS

async initializePreOrderPayment() {
    const requestData = {
      data: {
        type: 'payments',
        attributes: {
          quote: QUOTE_DATA,
          payment: {
            amount: GRAND_TOTAL, // You will get it through the `/checkout-data?include=carts` endpoint
            paymentMethodName: 'stripe', // taken from /checkout-data?include=payment-methods
            paymentProviderName: 'stripe',  // taken from /checkout-data?include=payment-methods
          },
          preOrderPaymentData: {
            "transactionId": this.transactionId, // This is empty in the first request but has to be used in further requests
          },
        },
      },
    };

    const responseData = await this.fetchHandler(`GLUE_APPLICATION_URL/payments`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ACCESS_TOKEN`,
      },
      body: JSON.stringify(requestData),
    });

    const paymentProviderData =
      responseData.data.attributes.preOrderPaymentData;

    this.transactionId = paymentProviderData.transactionId;
    this.accountId = paymentProviderData.accountId; // only be used on the Direct business model. When using a Marketplace business model this will not be present.

    await this.setupStripe();
  }

```

To identify the customer when initiating a request using Glue API, use the `Authorization` header for a logged-in customer and `X-Anonymous-Customer-Unique-Id` for a guest user.

After a `PaymentIntent` is created using the Stripe API, a payment is created in the Stripe app. The response looks as follows:

```JSON
{
  "data": {
    "type": "payments",
    "attributes": {
      "isSuccessful": true,
      "error": null,
      "preOrderPaymentData": {
        "transactionId": "pi_3Q3............",
        "clientSecret": "pi_3Q3............_secret_R3WC2........",
        "publishableKey": "pk_test_51OzEfB..............."
      }
    }
  }
}
```

#### Rendering the Stripe Elements on the summary page

The `preOrderPaymentData` from the previous example is used to render Stripe Elements on the summary page:

```JAVASCRIPT
async setupStripe() {
    const paymentElementOptions = {
      layout: 'accordion', // Change this to your needs
    };

    let stripeAccountDetails = {};

    if (this.accountId) { // Only in Direct business model not in the Marketplace business model
      stripeAccountDetails = { stripeAccount: this.accountId }
    }

    const stripe = Stripe(this.publishableKey, stripeAccountDetails);

    const elements = stripe.elements({
      clientSecret: this.clientSecret,
    });

    const paymentElement = elements.create('payment', paymentElementOptions);
    paymentElement.mount('#payment-element'); // Change this to the id of the HTML element you want to render the Stripe Elements to

    SUBMIT_BUTTON.addEventListener('click', async () => {
      const { error } = await stripe.confirmPayment({
        elements,
        confirmParams: {
          return_url: `APPLICATION_URL/return-url?id=${idCart}`, // You need to pass the id of the cart to this request
        },
      });
      if (error) {
        // Add your error handling to this block.
      }
    });
  }
```

This sets up Stripe Elements on the summary page of your application. The customer can now select the Payment Method in Stripe Elements and submit the data. Then, the customer is redirected to the configured `return_url`, which makes another Glue API request to persist the order in the Back Office. After this, the customer should see the success page.

When the customer submits the order, the payment data is sent to Stripe. Stripe may redirect them to another page, for example — PayPal, or redirect the customer to the specified `return_url`. The `return_url` must make another Glue API request to persist the order in the Back Office.

#### Persisting orders in the Back Office through the return URL

Because an order can be persisted in the Back Office only after a successful payment, the application needs to handle the `return_url` and make a request to the Glue API to persist the order.

<details>
  <summary>Request example</summary>

```JAVASCRIPT
app.get('/return-url', async (req, res) => {
  const paymentIntentId = req.query.payment_intent;
  const clientSecret = req.query.payment_intent_client_secret;
  const idCart = req.query.id;

  if (paymentIntentId) {
    try {
      const data = await fetchHandler(`GLUE_APPLICATION_URL/checkout`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ACCESS_TOKEN`
        },
        body: JSON.stringify({
          data: {
            type: 'checkout',
            attributes: {
              customer: CUSTOMER_DATA,
              idCart: idCart,
              billingAddress: BILLING_ADDRESS,
              shippingAddress: SHIPPING_ADDRESS,
              payments: [
                {
                  paymentMethodName: 'Stripe',
                  paymentProviderName: 'Stripe',
                },
              ],
              shipment: SHIPMENT_DATA,
              preOrderPaymentData: {
                transactionId: paymentIntentId,
                clientSecret: clientSecret,
              },
            },
          },
        }),
      });

      if (data) {
        res.send('<h2>Order Successful!</h2>');
      } else {
        res.send('<h2>Order Failed!</h2>');
      }
    } catch (error) {
      console.error(error);
      res.send('<h2>Order Failed!</h2>');
    }
  } else {
    res.send('<h2>Invalid Payment Intent!</h2>');
  }
});
```

</details>

After this, the customer should be redirected to the success page or in case of a failure to an error page.


{% info_block infoBox %}
- When the customer reloads the summary page, which renders `PaymentElements`, an extra unnecessary API request is sent to initiate `preOrder Payment`. Stripe can handle these without issues. However, you can also prevent unnecessary API calls from being sent on the application side by, for example, checking if relevant data has changed.
- When the customer leaves the summary page, the payment is created in Stripe app and Stripe. However, in the Back Office, there is a stale payment without an order.
- To enable the customer to abort the payment process, you can implement the cancellation of payments through Glue API.

{% endinfo_block %}



#### Cancelling payments through Glue API

The following request cancels a PaymentIntent on the Stripe side and shows a `canceled` PaymentIntent in the Stripe Dashboard. You can implement this in your application to enable the customer to cancel the payment process.

<details>
  <summary>Cancel a payment through Glue API</summary>

```JAVASCRIPT
async cancelPreOrderPayment() {
    const requestData = {
      data: {
        type: 'payment-cancellations',
        attributes: {
          payment: {
            paymentMethodName: 'stripe',
            paymentProviderName: 'stripe',
          },
          preOrderPaymentData: {
            transactionId: this.transactionId,
          },
        },
      },
    };

    const url = `GLUE_APPLICATION_URL/payment-cancellations`;

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ACCESS_TOKEN`,
      },
      body: JSON.stringify(requestData),
    });

    if (!response.ok) {
      throw new Error('Network response was not ok');
    }

    const responseData = await response.json();

    if (responseData.data.attributes.isSuccessful === true) {
      // Add your logic here when the payment cancellation was successful
    } else {
        // Add your logic here when the payment cancellation has failed
    }
  }
```

</details>


### Implementing Stripe checkout as a hosted payment page

If you have rewritten `@CheckoutPage/views/payment/payment.twig` on the project level, do the following:

1. Make sure that a form molecule uses the following code for the payment selection choices:

```twig
{% raw %}

{% for name, choices in data.form.paymentSelection.vars.choices %}
    ...
    {% embed molecule('form') with {
        data: {
            form: data.form[data.form.paymentSelection[key].vars.name],
            ...
        }
    {% endembed %}
{% endfor %}
{% endraw %}       
```

2. If you want to change the default payment provider or method names, do the following:
   1. Make sure the names are translated in your payment step template:

```twig
{% raw %}
{% for name, choices in data.form.paymentSelection.vars.choices %}
    ...
    <h5>{{ name | trans }}</h5>
{% endfor %}
{% endraw %}
```

    2. Add translations to your glossary data import file:

```csv
...
Stripe,Pay Online with Stripe,en_US
```
    3. Run the data import command for the glossary:

```bash
console data:import glossary
```

## Processing refunds

In the default OMS configuration, a refund can be done for an order or an individual item. The refund action is initiated by a Back Office user triggering the `Payment/Refund` command. The selected item enters the `payment refund pending` state, awaiting the response from Stripe.

During this period, Stripe attempts to process the request, which results in success or failure:
* Success: the items transition to the `payment refund succeeded` state, although the payment isn't refunded at this step.
* Failure: the items transition to the `payment refund failed` state.

These states are used to track the refund status and inform the Back Office user. In a few days after an order is refunded in the Back Office, Stripe finalizes the refund, causing the item states to change accordingly. Previously successful refunds may be declined and the other way around.

If a refund fails, the Back Office user can go to the Stripe Dashboard to identify the cause of the failure. After resolving the issue, the item can be refunded again.

In the default OMS configuration, seven days are allocated to Stripe to complete successful payment refunds. This is reflected in the Back Office by transitioning items to the `payment refunded` state.

## Retrieving and using payment details from Stripe

For instructions on using payment details, like the payment reference, from Stripe, see [Retrieve and use payment details from third-party PSPs](/docs/pbc/all/payment-service-provider/{{page.version}}/base-shop/retrieve-and-use-payment-details-from-third-party-psps.html)

## Embed the Stripe payment page using iframe

By default, the Stripe App payment flow assumes that the payment page is on another domain. When users place an order, they're redirected to the Stripe payment page. To improve the user experience, you can embed the Stripe payment page directly into your website as follows:


1. Create or update `src/Pyz/Zed/Payment/PaymentConfig.php` with the following configuration:
```php
namespace Pyz\Zed\Payment;

class PaymentConfig extends \Spryker\Zed\Payment\PaymentConfig
{
    public function getStoreFrontPaymentPage(): string
    {        
        // Please make sure that domain is whitelisted in the config_default.php `$config[KernelConstants::DOMAIN_WHITELIST]`
        return '/payment'; //or any other URL on your storefront domain e.g. https://your-site.com/payment-with-stripe
    }
}
```

In this setup, the redirect URL will be added as a `url` query parameter to the URL you've specified in the `getStoreFrontPaymentPage()` method; the value of the parameter is base64-encoded.


2. Depending on your frontend setup, create a page to render the Stripe payment page in one of the following ways:

* Use the following minimal page regardless of the frontend technology used.
* If your Storefront is based on a third-party frontend, follow the documentation of your framework to create a page to render the Stripe payment page using query parameters from the redirect URL provided in the Glue API `POST /checkout` response.
* If your Storefront is based on Yves, follow [Create an Yves page for rendering the Stripe payment page](#create-an-yves-page-for-rendering-the-stripe-payment-page).

```php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Order payment page</title>
</head>
<body>
<iframe src="<?php echo base64_decode($_GET['url']) ?>" width="100%" height="700px" style="border:0"></iframe>
</body>
</html>
```


### Create an Yves page for rendering the Stripe payment page


1. Create a controller to render the payment page:

**src/Pyz/Yves/PaymentPage/Controller/PaymentController.php**
```php

namespace Pyz\Yves\PaymentPage\Controller;

use Spryker\Yves\Kernel\Controller\AbstractController;
use Spryker\Yves\Kernel\View\View;
use Symfony\Component\HttpFoundation\Request;

class PaymentController extends AbstractController
{
    /**
     * @return \Spryker\Yves\Kernel\View\View
     */
    public function indexAction(Request $request): View
    {
        return $this->view(
            [
                'iframeUrl' => base64_decode($request->query->getString('url', '')),
            ],
            [],
            '@PaymentPage/views/payment.twig',
        );
    }
}

```

2. Create a template for the page:

**src/Pyz/Yves/PaymentPage/Theme/default/views/payment.twig**
```twig
{% raw %}
{% extends template('page-layout-checkout', 'CheckoutPage') %}

{% define data = {
    iframeUrl: _view.iframeUrl,
    title: 'Order Payment' | trans,
} %}

{% block content %}
    <iframe  src="{{ data.iframeUrl }}" class="payment-iframe" style="border:0; display:block; width:100%; height:700px"></iframe>
{% endblock %}
{% endraw %}
```

3. Create a route for the controller:

**src/Pyz/Yves/PaymentPage/Plugin/Router/EmbeddedPaymentPageRouteProviderPlugin.php**
```php
namespace Pyz\Yves\PaymentPage\Plugin\Router;

use Spryker\Yves\Router\Plugin\RouteProvider\AbstractRouteProviderPlugin;
use Spryker\Yves\Router\Route\RouteCollection;

class EmbeddedPaymentPageRouteProviderPlugin extends AbstractRouteProviderPlugin
{
    /**
     * @param \Symfony\Component\Routing\RouteCollection $routeCollection
     *
     * @return \Symfony\Component\Routing\RouteCollection
     */
    public function addRoutes(RouteCollection $routeCollection): RouteCollection
    {
        $route = $this->buildRoute('/payment', 'PaymentPage', 'Payment', 'indexAction');
        $routeCollection->add('payment-page', $route);

        return $routeCollection;
    }
}
```

4. In `src/Pyz/Yves/Router/RouterDependencyProvider.php`, add a router plugin to `RouterDependencyProvider::getRouteProvider()`.


## Sending additional data to Stripe

Stripe accepts metadata passed using API calls. To send additional data to Stripe, the `QuoteTransfer::PAYMENT::ADDITIONAL_PAYMENT_DATA` field is used; the field is a key-value array. When sending requests using Glue API, pass the `additionalPaymentData` field in the `POST /checkout` request.

```text
POST https://api.your-site.com/checkout
Content-Type: application/json
Accept-Language: en-US
Authorization: Bearer {{access_token}}

{
    "data": {
        "type": "checkout",
        "attributes": {
            "customer": {
                ...
            },
            "idCart": "{{idCart}}",
            "billingAddress": {  
                ...
            },
            "shippingAddress": {
                ...
            },
            "payments": [
                {
                    "paymentMethodName": "Stripe",
                    "paymentProviderName": "Stripe",
                    "additionalPaymentData": {
                         "custom-field-1": "my custom value 1",
                         "custom-field-2": "my custom value 2"
                    }
                }
            ],
            "shipment": {
                "idShipmentMethod": {{idMethod}}
            }
        }    
    }
}
```

The metadata sent using the field must meet the following criteria:

| ATTRIBUTE | MAXIMUM VALUE |
| - | - |
| Key length | 40 characters |
| Value length | 500 characters |
| Key-value pairs | 50 pairs |

When you pass metadata to Stripe, it's stored in the payment object and can be retrieved later. For example, this way you can pass an external ID to Stripe.

When a `PaymentIntent` is created on the Stripe side, you can see your passed `additionalPaymentData` in the Stripe Dashboard.
