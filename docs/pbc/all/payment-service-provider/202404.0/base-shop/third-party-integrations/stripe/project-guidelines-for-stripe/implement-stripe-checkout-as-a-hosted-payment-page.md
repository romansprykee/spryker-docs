---
title: Implement Stripe checkout as a hosted payment page
description: Learn how to implement Stripe using ACP
last_updated: Nov 8, 2024
template: howto-guide-template
related:
   - title: Stripe
     link: docs/pbc/all/payment-service-provider/page.version/base-shop/third-party-integrations/stripe/stripe.html
---

Implementing Stripe checkout as a hosted payment page is usually needed if `@CheckoutPage/views/payment/payment.twig` is overwritten on the project level. To implement Stripe checkout as a hosted payment page, follow the steps:

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

   ```
   Stripe,Pay Online with Stripe,en_US
   ```

    3. Run the data import command for the glossary:

   ```bash
   console data:import glossary
   ```