---
layout: post
title:  "Same Date Annual Billing with Chargify"
date:   2015-01-26 07:41:00
tags: chargify
---

[Chargify][chargify] offers flexible pricing structures for recurring billing.  For many things, the built-in configuration will work just fine.  Sometimes, though, you'll need to do a bit of work outside the Admin UI to adapt to your situation.

For example, consider <b>a club whose subscriptions all renew on November 1st.  No matter when you join, you pay the full yearly membership dues, plus a startup fee, and then your next payment is due on November 1st</b>.

### Setup

To accomplish this, first we'll set up a Club Membership product that costs $35.00 per year with a $10.00 setup fee.

![Billing Setup](/images/2015/01/26/billing-setup.png "Billing Setup")

After submitting the form, the setup looks like this:

![Product Setup](/images/2015/01/26/product-setup.png "Product Setup")

If we stop here, the member will be charged correctly when they sign up, but their subscription will renew a full year later, instead of on November 1st like we want it to.

One way to fix that is to change the next billing date for each new member in the Admin UI.  That might be fine if you only have a few members, but it would be better to automate the process.

### Webhooks

So, we'll listen for the <code>signup_success</code> [webhook][webhooks], and when we get that, we'll make an [API call to change the next billing date][subscriptions-api] to November 1st at noon.

Here's the code for a simple Webhook Listener which listens for the <code>signup_success</code> webhook and changes the next billing date for the new subscription:

~~~ php
<?php

require_once('./config.php');
  
// see https://github.com/lewsid/chargify-webhook-helper/blob/master/example.php for example code

$subdomain = $_POST['payload']['site']['subdomain'];

if ($_POST["event"] == "signup_success")
{
  $subscription_id = $_POST['payload']['subscription']['id'];

  $json = '{ "subscription": { "next_billing_at": "2015-11-01T12:00:00-05:00" } }';

  // see https://github.com/jforrest/Chargify-PHP-Client/blob/master/lib/ChargifyConnector.php for example code

  $ch = curl_init();

  curl_setopt($ch, CURLOPT_URL, "https://" . $subdomain . ".chargify.com/subscriptions/" . $subscription_id );
  curl_setopt($ch, CURLOPT_USERPWD, $userpwd );
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
  curl_setopt($ch, CURLOPT_HTTPHEADER, array(
      'Content-Type: application/json',
      'Accept: application/json'
    ));
  curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "PUT");
  curl_setopt($ch, CURLOPT_POSTFIELDS, $json);

  $output = curl_exec($ch);

  curl_close($ch);
  
}

?>
~~~

You can find the source for the [webhook listener](https://github.com/wsmoak/chargify/blob/20150126-blog/php/same-date-annual-billing-webhook.php) and the [sample config file](https://github.com/wsmoak/chargify/blob/20150126-blog/php/sample_config.php) in [my Chargify repository on GitHub](https://github.com/wsmoak/chargify).  

Keep in mind that in a real application you'd want to check the signature on the webhook you receive, or use the event id to retrieve it directly from the API, as described in the [Webhooks documentation][webhooks].

And of course you'd want to calculate the year instead of hard-coding it.  Perhaps you don't want to charge new members who sign up during the month of October until November 1 of the following year.

### Public Signup Page

With the [Public Signup Pages][psp] that Chargify offers, club members can securely sign up for a membership without the club having to worry about handling their credit card data.  The top portion of the page looks like this by default:

![Signup Page Before](/images/2015/01/26/psp-before.png "Signup Page Before")

You can see that it says "(then $35.00 at first renewal on 24 Jan 2016)", but that's not correct, because as soon as a new subscription is created, the webhook listener will change the next billing date to November 1.

To fix that, all we need to do is edit the Public Signup Page and enter some custom JavaScript:

~~~ javascript
function changeNextRenewalHtml() {
  var nextRenewal = $('#next-renewal-charge');
  nextRenewal.html("(then $35.00 on November 1st each year)");
};

$(document).bind("afterSummaryRefresh", changeNextRenewalHtml);
~~~

Now the signup page will look like this:

![Signup Page After](/images/2015/01/26/psp-after.png "Signup Page After")

### Results

And here is our first customer!

![Test Subscription](/images/2015/01/26/test-subscription.png "Test Subscription")

You can see that the subscription was activated on Jan 25th, and that rather than renewing on January 25th, _2016_, the next billing date has already been changed to Nov 1st of this year.

In the activity, (which reads in reverse chronological order,) you can see exactly what happened:

![Subscription Activity](/images/2015/01/26/subscription-activity.png "Subscription Activity")

The payment processed successfully, the subscription was created, the statmement settled, and then the billing date was changed.

### Summary

With very little work we've been able to customize a [Chargify][chargify] site to match the business requirements of a club membership. If you'd like to discuss using [Chargify][chargify] for your own subscription-based business, head over to the [Support Site](https://chargify.zendesk.com/hc/en-us) or [Twitter](https://twitter.com/chargify) and ask!

### References

* <https://github.com/lewsid/chargify-webhook-helper/blob/master/example.php>
* <https://github.com/jforrest/Chargify-PHP-Client/blob/master/lib/ChargifyConnector.php> 
* <http://php.net/manual/en/book.curl.php>
* [Chargify Webhooks][webhooks]
* [Chargify Subscriptions API][subscriptions-api] (see next_billing_at)
* [Chargify Public Signup Pages][psp]

[chargify]: https://www.chargify.com
[subscriptions-api]: https://docs.chargify.com/api-subscriptions
[webhooks]: https://docs.chargify.com/webhooks
[psp]: https://docs.chargify.com/public-pages-intro