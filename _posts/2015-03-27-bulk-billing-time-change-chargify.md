---
layout: post
title:  "Bulk Billing Time Change with Chargify"
date:   2015-03-27 07:41:00
tags: chargify
---

[Chargify][chargify] has a Bulk Billing Date Change feature that allows you to select multiple subscriptions and then change all of them to a certain billing date and time at once.

But <b>what if you want to change ONLY the billing time for all of your subscriptions</b>, and leave the date alone?

Perhaps you need to print out shipping labels or provide a file to a third party by a certain time each day, or perhaps you simply prefer to have all of your renewals for the day processed by the time you arrive at work in the morning.

At this time the Chargify web interface does not have an option to change only the time for multiple subscriptions<sup>1</sup>, however it's fairly easy to do if you're able to run a bit of Ruby code.

Chargify has provided a Ruby Gem that allows you to interact with the API.  With it you can make make changes to your existing subscriptions (as well as create new ones.)

You can find the source code and installation instructions for the [Chargify API Active Resource Gem on GitHub][chargify-api-ares].

Once you've installed the Gem, take a look at the [Subscriptions examples][subscriptions-examples] to get an idea of how to use it.

### Getting Started

The first thing we'll need to do is find all of the Active subscriptions:

~~~ ruby
subscriptions = Chargify::Subscription.find( :all, params: { state: "active" } )
~~~

By default, this is limited to 20 subscriptions, but you can request up to 200 at a time:

~~~ ruby
subscriptions = Chargify::Subscription.find( :all, params: { per_page: 200, state: "active" } )
~~~

Once you have the list of subscriptions, here's a way to change all of the billing times without changing the date:

~~~ ruby
subscriptions.each { |sub|
    curr_next_billing = sub.next_assessment_at
    new_next_billing = Time.new curr_next_billing.year, curr_next_billing.month, curr_next_billing.day, 6, 0, 0, "-05:00"
    sub.next_billing_at = new_next_billing
    puts "id: #{sub.id} curr: #{curr_next_billing} new: #{new_next_billing}"
    sub.save
  }
~~~

(Be sure to scroll out to the right to see all of the code.)

This will change the billing time for each subscription to 6:00 AM CDT, which has a UTC offset of -05:00.

Since Chargify stores the dates in Universal Coordinated Time (UTC), the billing time will seem to move back and forth by an hour as Daylight Savings Time begins and ends in your time zone.

What if you have more than 200 active subscribers?  Well, then you'll need to page through the results using both the <code>per_page</code> and <code>page</code> parameters.

~~~ ruby
subscriptions = Chargify::Subscription.find( :all, params: { page: 1, per_page: 200, state: "active" } )

subscriptions = Chargify::Subscription.find( :all, params: { page: 2, per_page: 200, state: "active" } )
~~~

### Choosing a Time

Before we move on to a full example, a quick word about choosing a time of day for all your renewals.  If you have a hard deadline such as a third party needing data in order to ship a physical product, be sure to allow PLENTY of time for your renewals to complete.

For example, if you need to produce a file by 9AM, set your renewals at 3AM or 4AM.

It's also a good idea to avoid times from 11PM to 1AM.  If your time zone observes Daylight Savings Time, you may find that your monthly subscriptions renew on a different _day_ for half the year due to the "Spring Forward" or "Fall Back" effect.

### Full Example

Here's a quick Ruby script that pulls it all together:

<https://github.com/wsmoak/chargify/blob/master/ruby/billing-time-change.rb>

This example loops through all of the Active and Trialing subscriptions, five at a time, and changes the billing time to 6:00 AM CDT.

It assumes you have environment variables set for your Subdomain and API Key, however you can pull the values from a config file if you prefer.

To use it, change time and the offset to match your desired renewal time.  You can also change the <code>per_page</code> value to anything up to 200.

And finally, you will need to un-comment line 44 so that it will actually save the values.

### Summary

This example should help you understand how to change your subscribers' billing times without changing the dates.

<b>Use at your own risk!</b> I've run this code against my collection of test subscriptions, however you should thoroughly test and understand the code *before* using it on any live subscriptions.

If something doesn't make sense, leave a comment below or drop me an email at <wsmoak@gmail.com>, and I'll try to clear it up.

### References
* <http://www.timeanddate.com/time/zones/cdt>
* <http://ruby-doc.org/core-2.2.1/Time.html>
* [Chargify Subscriptions API][subscriptions-api] (see next_billing_at)
* [Chargify API Active Resource Gem][chargify-api-ares]
* [Chargify Gem Subscriptions examples][subscriptions-examples]

[chargify]: https://www.chargify.com
[subscriptions-api]: https://docs.chargify.com/api-subscriptions
[chargify-api-ares]: https://github.com/chargify/chargify_api_ares
[subscriptions-examples]: https://github.com/chargify/chargify_api_ares/blob/master/examples/subscriptions.rb

### Footnotes

<sup>1</sup>Actually, that's not *entirely* true.  You must specify a date along with the time.  So, you can filter for the subscriptions that renew on a particular date, and then change the time for all of those subscriptions at once.  If you have monthly subscriptions, you can do this for the next 31 days, one day at a time.  That's still quite a bit of clicking, but it's better than changing each one of the subscriptions individually!

