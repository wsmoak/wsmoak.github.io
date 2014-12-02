---
layout: post
title:  "Listing Stripe Customers in Python"
date:   2014-11-26 18:37:00
tags: stripe python
---

The [Stripe API docs](https://stripe.com/docs/api) are *awesome*.  All the attributes and arguments are explained in the middle column, and you can copy and paste from working examples in several languages in the right-hand column.  Sometimes, though, the example code doesn't do exactly what you need, and the description doesn't have quite enough information.

Recently, someone in the #stripe channel on Freenode wanted a list of customers that were created before a certain date, using Python.

The example code for [https://stripe.com/docs/api/python#list_customers](https://stripe.com/docs/api/python#list_customers) simply shows: 

{% highlight python %}
import stripe
stripe.api_key = "sk_test_XXXXXXXXXXXX"

stripe.Customer.all()
{% endhighlight %}

So let's try that just to make sure everything is working...

{% highlight python %}
import os
import stripe

stripe.api_key = os.environ['STRIPE_SECRET_KEY']

# list all customers (10 by default) 
print stripe.Customer.all()
{% endhighlight %}

...and we get back ten customer records (head and tail of output shown below):

~~~
{
  "data": [
    {
      "account_balance": 0, 
      "cards": {
        "data": [
          {
            "address_city": null, 
            "address_country": null, 
            "address_line1": null, 
...
        "object": "list", 
        "total_count": 0, 
        "url": "/v1/customers/cus_XXXXXXXXXXX/subscriptions"
      }
    }
  ], 
  "has_more": true, 
  "object": "list", 
  "url": "/v1/customers"
}
~~~

Note that you don't just get an array of customers, you get an object with a ```data``` property, the value of which is an array of customer objects.  Make sure you access the ```data``` property if you want to iterate over the results.

At the end you get some additional properties that tell you there are more records, that it is a list, and the url of the api endpoint.

Now let's use the ```created``` parameter.  

{% highlight python %}
# list customers created on 2014-10-01 at midnight
print stripe.Customer.all(created=1412121600)
{% endhighlight %}

This doesn't return any results for me, because I don't have any customers that were created at the stroke of midnight UTC on October 1, 2014.

And finally, what the person was looking for.  The ```created``` parameter can either be a "...string with an integer Unix timestamp, or it can be a dictionary with the following options...".  (Switch languages in the right hand column, and you'll see that for Java it says Map instead of dictionary, and for Ruby it says hash.)  Here is an example using two options to show the syntax:

{% highlight python %}
# list customers created between 2014-10-01 and 2014-10-31
print stripe.Customer.all( limit=100, created={"gte": 1412121600,"lte": 1414713600} )
{% endhighlight %}

In addition, the docs say "You can optionally request that the response include the total count of all customers that match your filters. To do so, specify `include[]=total_count` in your request."  The syntax for that is:

{% highlight python %}
# include the total count
print stripe.Customer.all( limit=25, include=["total_count"] )
{% endhighlight %}

And as described, the result contains a total_count attribute:

{% highlight console %}
[...]
    }
  ], 
  "has_more": true, 
  "object": "list", 
  "total_count": 221, 
  "url": "/v1/customers"
}
{% endhighlight %}

If you're using [Stripe](https://stripe.com), be sure to read the API docs carefully -- there are far more options available than are shown in the example code.