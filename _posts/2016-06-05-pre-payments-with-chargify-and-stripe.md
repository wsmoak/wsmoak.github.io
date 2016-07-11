---
layout: post
title:  "Pre-Payments with Chargify and Stripe"
date:   2016-06-05 21:47:00
tags: chargify stripe
---

Chargify offers flexible pricing schemes for recurring billing including add-ons and metered usage.  One thing it doesn't do out of the box, however, is handle pre-payments.  If you're already using the Chargify API, it's easy enough to _also_ use your payment gateway's API to make this happen. Let's see how to do pre-payments with Stripe and Chargify.

This will work with most other gateways, as long as you're allowed to make API calls to the gateway from your server.  (There are some gateways that require certification and use whitelists to limit who they accept API calls from.)

## Prerequisites

You'll need these things before we begin:

* A Chargify account.  [Sign up for a free Developer account][chargify-signup] if you don't already have one.
* A Stripe account for testing.  [Sign up here][stripe-signup] if you don't already have one.
* A reasonable comfort level with developing web applications in your language/framework of choice. The example code below is in Ruby.
* A basic understanding of the [Stripe][stripe-api] and [Chargify][api-intro] APIs. .

## The Plan

Assuming you have a customer with an active subscription, you can get the "vault token" and use that to process a payment directly at the gateway, and then record it as an external payment in Chargify.  This will produce a negative balance to be applied against future charges.

In Chargify, there is no way to cause a payment to be processed without also creating a charge, so you can't get a negative balance to happen.  One workaround is to do a one-time charge, and then add a credit for the same amount, but that will all show up on the customer's statement and is hard to explain and reconcile.

Instead, let's retrieve the "vault token" that Chargify uses to process renewal payments, and use it to charge the customer's card directly at the gateway.

Whatever triggers the need for a pre-payment in your application, when that happens, here's how to create a payment directly at Stripe and then record it in Chargify.

## Step 1: Get the Vault Token

Retrieve the Chargify Subscription and find the "Vault Token" inside the payment profile.  This is what identifies the credit card details that are securely stored in your gateway's vault.  For Stripe, this will be a customer id, which looks like this: `cus_ab1234cd567`.

<https://docs.chargify.com/api-subscriptions>

{% highlight ruby %}

sub = Chargify::Subscription.find 1234567

vault_token = sub.payment_profile.vault_token

{% endhighlight %}

## Step 2: Create a Charge at Stripe

Use the Stripe API to create a charge for the desired amount.

<https://stripe.com/docs/api#create_charge>

{% highlight ruby %}
require "stripe"
Stripe.api_key = "sk_test_YOUR_API_KEY"

Stripe::Charge.create(
  :amount => 400,
  :currency => "usd",
  :customer => vault_token
  :description => "Pre-paid usage for..."
)
{% endhighlight %}

## Step 3: Record an External Payment at Chargify:

If the charge done at Stripe was successful, record it in Chargify.

In a production application, you will of course want better error handling and logging so that you can track down any issues.

<https://docs.chargify.com/api-payments>

{% highlight ruby %}
require "chargify_api_ares"

if chg.paid
  pmt = sub.payment(
    amount_in_cents: 2500,
    memo: "Pre-payment for..."
  )
else
  puts "Charge at Stripe was NOT successful"
end
{% endhighlight %}

## Result:

Now you have a subscription with a credit balance and a clean transaction history, showing the payment that was made in advance of any charges being assessed.

From here, you may record metered usage or set component quantities to be charged for at the end of the billing period.

When the billing period ends, Chargify will calculate what is due, and subtract the credit balance before charging for any remaining amount.

## Summary

I hope that helps you get started collecting pre-payments with Chargify and Stripe.  If you need more help and you have a paid account with Chargify, you can open a support ticket with them.  Otherwise, if you're on the free Developer plan, the best place to ask is on [Stack Overflow][so].

The code for this example is available at <https://github.com/wsmoak/chargify-examples/tree/master/ruby/stripe/pre-payments.rb> and is MIT licensed.

Copyright 2016 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

## Notes

Be sure to use version 1.4.4 or later of the [chargify_api_ares][gem] gem.  Earlier versions do not support external payments.

Adding an external payment _will_ trigger a Receipt Email, if they are enabled for your Chargify Site.

[chargify-signup]: https://app.chargify.com/signup/developer3
[stripe-signup]: https://dashboard.stripe.com/register
[so]: http://stackoverflow.com/questions/tagged/chargify
[api-intro]: https://docs.chargify.com/api-introduction
[stripe-api]: https://stripe.com/docs/api
[gem]: https://rubygems.org/gems/chargify_api_ares/
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}

