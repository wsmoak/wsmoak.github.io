---
layout: post
title:  "Validating Stripe Coupon Codes"
date:   2014-11-14 20:30:00
tags: stripe
---

Today's [Stripe](https://stripe.com) how-to is on validating coupon codes entered by the customer.

Here's a subscription page that asks the customer for a coupon code and uses the simple integration of [Stripe Checkout](https://stripe.com/docs/checkout). [1]

{% highlight html %}
<form action="charge-validate-coupon.php" method="post">

  Coupon Code: <input type=text size="6" id="coupon" name="coupon_id" />
  <span id="msg"></span>
  <script src="https://checkout.stripe.com/checkout.js" class="stripe-button"
          data-key="<?php echo $stripe['publishable_key']; ?>"
          data-amount="2995"
          data-description="Monthly Subscription"
          data-label="Subscribe"
          data-allow-remember-me="false">
  </script>
</form>
{% endhighlight %}

Ideally, we'd like to check that the coupon code is valid and give the customer a chance to fix it before they hit the 'Pay' button and Checkout takes over.  

Here is a bit of JavaScript/jQuery to do that.  It communicates with a PHP page that simply responds with true or false.

{% highlight javascript %}
<script type="text/javascript" src="jquery-1.11.1.js"></script>

<script>
$(document).ready(function(){
  $('#coupon').change(function(){
    requestData = "coupon_id="+$('#coupon').val();
    $.ajax({
      type: "GET",
      url: "validate-coupon.php",
      data: requestData,
      success: function(response){
        if (response) {
          $('#msg').html("Valid Code!")
        } else {
          $('#msg').html("Invalid Code!");
        }
      }
    });
  });
});
</script>
{% endhighlight %}

Whenever the Coupon Code text field loses focus and has changed, the script will run.  The client will validate the coupon code by communicating with your server (which in turn communicates with Stripe), and a message will appear next to the text field.

Not unexpectedly, this is _really_ slow. It would be better to use the [custom integration](https://stripe.com/docs/checkout#integration-custom) so that you can disable the 'Pay' button while the validation is taking place.

In any case, you will still need to handle invalid codes on the server side.  While you could put in logic to prevent a form submission if the code is invalid, there's still the case where the coupon becomes invalid after this check, but before the form is submitted.

You can find the files <code>test-validate-coupon.php</code>, <code>charge-validate-coupon.php</code>, and <code>validate-coupon.php</code> in my GitHub repository:  [https://github.com/wsmoak/stripe/tree/master/php](https://github.com/wsmoak/stripe/tree/master/php).

[1] See Pete Keen's advice on [Using Stripe Checkout for Subscriptions](https://www.petekeen.net/using-stripe-checkout-for-subscriptions).