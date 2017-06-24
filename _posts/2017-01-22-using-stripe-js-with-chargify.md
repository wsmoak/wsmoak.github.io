---
layout: post
title:  "Using Stripe JS with Chargify"
date:   2017-01-22 11:59:00
tags: chargify stripe
---

Stripe offers a JavaScript library called Stripe JS that you can use to collect payment details.  It sends the customer's credit card data directly to Stripe and gives you a token so that you don't have to handle card data on your own server.  Let's see how to integrate this with Chargify subscription signups and card updates.

This is an alternative to Chargify Direct, in which you post the _entire_ form to Chargify's servers and they redirect the customer back to you. Stripe Checkout still allows you to avoid extensive PCI compliance work (though [some is still required!][stripe-pci]) and it gives you more control over the customer experience.

## Prerequisites

You'll need these things before we begin:

* A Chargify account.  [Sign up for a free Developer account][chargify-signup] if you don't already have one.
* A Stripe account for testing.  [Sign up here][stripe-signup] if you don't already have one.
* A reasonable comfort level with developing web applications in your language/framework of choice.
* A basic understanding of the Chargify API. Start [here][api-intro].

## Create

To create a new subscription using credit card data that was captured by Stripe JS, you will need to design your own signup page and include the JavaScript provided by Stripe.  When the customer clicks Submit, you'll first send the sensitive data directly to Stripe to tokenize it before allowing the form to submit the token and the rest of the fields to your own server.

### Form

The most important thing to remember is to _not_ add `name` elements to the sensitive fields like the card number, cvv, and expiration date.

This page in the Stripe documentation has an example form and more information:  <https://stripe.com/docs/custom-form>

## Server Side

When the form is submitted to your server, you'll need to use the token that Stripe Checkout inserted to create a customer record in Stripe.

{% highlight ruby %}
  customer = Stripe::Customer.create(
    :email => params[:stripeEmail],
    :card  => params[:stripeToken]
  )
{% endhighlight %}

Assuming it is successful, the result will contain a customer. The customer's id is what Chargify needs in order to process payments and renewals, and is supplied as the 'vault_token'.

{% highlight ruby %}
  subscription = Chargify::Subscription.create(
    product_handle: 'monthly-plan',
    customer_attributes: {
      first_name: "Valued",
      last_name: "Customer",
      email: params[:stripeEmail]
    },
    credit_card_attributes: {
      current_vault: 'stripe',
      vault_token: customer.id,
      first_name: "Valued",
      last_name: "Customer",
      card_type: card.brand.downcase,
      last_four: card.last4,
      expiration_month: card.exp_month,
      expiration_year: card.exp_year
    }
  )
{% endhighlight %}

## Signup Flow

The customer navigates to your page, fills out the form and clicks the submit button. The Stripe JavaScript tokenizes the card and inserts some fields into the form, then browser submits the form to your server.  You use the token to create a customer in Stripe, and then use the ID of the Stripe customer as the `vault_token` to create a subscription using the Chargify API.

## Card Update

How you update cards will depend on how you use the data in your Stripe account.  If you don't mind duplicate customer records in Stripe, simply collect the card details, make a new Stripe customer-with-card, then make a payment profile in Chargify and make it the default for the subscription.

<http://help.chargify.com/announcements/change-payment-profile.html>

Stripe does allow partial updates, so you can update, for example, the expiration date without supplying the full card number again.  This can be done directly through the Chargify API:

<https://docs.chargify.com/api-payment-profiles#update>

Alternately, since Chargify always charges the default card belonging to the Stripe customer, you can add a card to the Stripe customer on the Stripe side, and create a new payment profile using the same vault token on the Chargify side so that the card number and expiration date will match.  Remember to make that card the default for the subscription since that doesn't happen automatically when adding a new payment profile.

Other options to update the payment details on an existing subscription are: You may direct customers to the Chargify-hosted Self-Service page, the Billing Portal, or you may use the Chargify Direct `card_update` endpoint.

## Summary

I hope that helps you get started using Stripe JS with Chargify!  If you need more help and you have a paid account with Chargify, you can open a support ticket with them.  Otherwise, if you're on the free Developer plan, the best place to ask API questions is on [Stack Overflow][so].  There is also a [list of consultants][lc] who are familiar with Chargify integrations if you need help with application design and development.

The code for this example is available at <https://github.com/wsmoak/chargify-examples/blob/master/ruby/stripe/sinatra-stripejs.rb> and is MIT licensed.

[chargify-signup]: https://app.chargify.com/signup/developer3
[stripe-signup]: https://dashboard.stripe.com/register
[so]: http://stackoverflow.com/questions/tagged/chargify
[lc]: https://www.chargify.com/consultants/
[api-intro]: https://docs.chargify.com/api-introduction
[stripe-pci]: https://stripe.com/docs/security
[v1-subscriptions]: https://docs.chargify.com/api-subscriptions
[checkout-docs]: https://stripe.com/docs/checkout#integration-simple-options
