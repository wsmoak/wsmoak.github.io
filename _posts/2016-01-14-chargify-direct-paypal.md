---
layout: post
title:  "Accepting PayPal with Chargify Direct"
date:   2016-01-14 08:41:00
tags: chargify paypal
---

When Chargify originally announced [support for customers to pay with PayPal][paypal-blog], it was limited to the Chargify-hosted Public Signup and Self-Service pages.  If you are using Braintree Blue as your gateway, you can check a box, and your customers will have the option to either pay with a credit card or with PayPal. Recently, the ability to accept PayPal with Chargify Direct was added.  Let's see how it works.

## Prerequisites

You'll need these things before we begin:

* A Chargify account.  [Sign up for a free Developer account][chargify-signup] if you don't already have one.
* A Braintree Blue Sandbox account for testing.  [Sign up here][braintree-signup] if you don't already have one.
* A reasonable comfort level with developing web applications in your language/framework of choice.
* A basic understanding of Chargify Direct. Read [this introduction][chargify-direct-intro] and take a look at the [Ruby Sinatra example app][cd-ruby].

## Create

To create new subscriptions that are paid with PayPal using Chargify Direct, you will need to design your own signup form and include the Braintree widget.  The widget is what displays the blue PayPal button and the pop-up that your customers will use to authenticate and then authorize the recurring transactions.

Here's what the PayPal flow looks like on a Public Signup Page:

![the gif](/images/2016/01/chargify-psp-flow.gif)

I found these articles helpful when I was creating my example application:

* <https://developers.braintreepayments.com/start/hello-client/javascript/v2>

* <https://developers.braintreepayments.com/start/hello-server/ruby>

* <https://developers.braintreepayments.com/guides/paypal/client-side/javascript/v2>

In addition to the usual name, address and email fields, the page needs these additional things:

### Braintree JavaScript

{% highlight html %}
  <script src="https://js.braintreegateway.com/v2/braintree.js"></script>
{% endhighlight %}

### Braintree Setup

{% highlight html %}
  <script type="text/javascript">
    braintree.setup( "<%= @client_token %>" , "paypal", {
      container: "paypal-container",
      locale: "de_de",
      paymentMethodNonceInputField: "paypal_nonce",
      onPaymentMethodReceived: function (obj) {
        doSomethingWithTheNonce(obj.nonce);
      }
    });
  </script>
{% endhighlight%}

Note that you can optionally set the locale if you would like the PayPal popup to appear in a certain language.

Note that the client token needs to be generated.  This is explained in the Braintree articles linked above.

### PayPal Container

Place this where you want the PayPal button to appear.  Note that the `id` of the `<div>` needs to match what you configured as the `container` in the `braintree.setup` above.

{% highlight html %}
  <div id="paypal-container"></div>
{% endhighlight %}

### Form Fields

And then add the (hidden) fields to your signup form.

Make sure that the fields are named exactly `signup[payment_profile][payment_method_nonce]`, `signup[payment_profile][paypal_email]` and `signup[payment_profile][payment_type]` and that the latter has a value of `paypal_account`, like this:

{% highlight html %}
<input type="hidden" id="paypal_nonce" type="text" name="signup[payment_profile][payment_method_nonce]" />
<input type="hidden" name="signup[payment_profile][paypal_email]" />
<input type="hidden" name="signup[payment_profile][payment_type]" value="paypal_account" />
{% endhighlight %}

Again, the `id` of the field for the nonce needs to match the value for the `paymentMethodNonceInputField` in the `braintree.setup` above.

You will probably want to fill in the paypal_email field with JavaScript. Unfortunately the Braintree widget won't fill it in for you.  On the Chargify side, it's only for display on the Payment Profiles tab, but it may cause confusion at some point if it doesn't match the email that the customer gave to PayPal when signing up.

When the form submits, all of the form fields including the hidden ones will be sent to Chargify and used to create the new subscription.

## Signup Flow

The customer navigates to your page, fills out the form, authenticates with PayPal, authorizes the payment, and submits the form (direct to Chargify). Chargify processes it, creates a subscription, and redirects the customer back to your website.

## Update

There is a second endpoint in Chargify Direct called `card_update`.  Despite the name, it also works for updating PayPal type payment profiles if you include the PayPal widget and the form fields as above.

## Summary

I hope that helps you get started accepting PayPal with Chargify Direct!  If you need more help and you have a paid account with Chargify, you can open a support ticket with them.  Otherwise, if you're on the free Developer plan, the best place to ask is on [Stack Overflow][so].

## References

* [Chargify + Braintree + PayPal = Pay with PayPal!][paypal-blog]
* [Chargify Direct Introduction][chargify-direct-intro]

[paypal-blog]: https://www.chargify.com/blog/paypal/
[chargify-signup]: https://app.chargify.com/signup/developer3
[braintree-signup]: https://www.braintreepayments.com/get-started
[chargify-direct-intro]: https://docs.chargify.com/chargify-direct-introduction
[cd-ruby]: https://github.com/chargify/chargify_direct_example
[so]: http://stackoverflow.com/questions/tagged/chargify
