---
layout: post
title:  "Joining CSV Files"
date:   2015-07-18 18:31:00
tags: csvkit csv chargify
---

Have you ever needed to combine data from two csv (comma-separated value) files?  Maybe you have transaction data in one, so you know who made payments, and shipping addresses in the other?

I'm sure it's possible to do by importing the files into Excel and... perhaps writing some formulas or macros and clicking buttons.  I'll leave that as an exercise for the reader. :)

But what if you could join the files, pick out the columns you want, even filter the rows based on certain criteria, all at the command line?

As long as the two csv files have some sort of identifier in common, you can!  There is an amazing project called [csvkit][csvkit] that provides utilities to do all of this.

From the [csvkit documentation][csvkit]: "csvkit is a suite of utilities for converting to and working with CSV, the king of tabular file formats."

### Example

In this example we're going to join up the Transactions and Customers CSV exports from [Chargify][chargify].

(If you're not already using [Chargify][chargify] you can [sign up for a free Developer account][chargify-signup], but it will be quite a bit of work to populate a test site with enough data to produce these exports. You'll probably have more fun using csv files you already have available. :) )

First let's use the [`csvcut`][csvcut] command to look at the column or field names in each file. Here is the transactions file:

<pre>
$ csvcut -n transactions.csv
  1: id
  2: created_at
  3: type
  4: memo
  5: amount_in_cents
  6: ending_balance_in_cents
  7: subscription_id
  8: customer_id
  9: customer_name
 10: product_id
 11: success
 12: kind
 13: payment_id
 14: gateway_transaction_id
 15: customer_organization
 16: gateway_order_id
</pre>

And the customers file:

<pre>
$ csvcut -n customers.csv
  1: id
  2: first_name
  3: last_name
  4: organization
  5: email
  6: created_at
  7: reference
  8: address
  9: address_2
 10: city
 11: state
 12: zip
 13: country
 14: phone
 15: created_at
 16: updated_at
</pre>

Note that field #8 in the transactions file is called `customer_id`.  That's going to match up with field #1 in the customers file, called simply `id`.

The goal for this step is to create a new csv file that contains all of the lines in the transactions.csv file, with each line having the fields from the appropriate line of the customers.csv file appended to it.

Have a look at the [documentation for `csvjoin`][csvjoin], which we're going to use to combine these files.

Based on this, we'll need to specify our filenames, transactions.csv and customers.csv and the two field names, customer_id and id.  Note that the order is important -- the first field name must be found in the first csv file, and so on.

We should also do a LEFT join, so that all the rows in the transactions file are preserved.  In theory, you can't have a transaction without a customer, but if for some reason the customers file is truncated you wouldn't want to miss one of the payments simply because it didn't match a line in the customers file.

So far, our command looks like this:

<code>
csvjoin -c "customer_id,id" --left transactions.csv customers.csv
</code>

Note that the transactions export contains *all* of the transactions that were exported. We're assuming that only Payments were exported, but if we only want the _successful_ payments, we'll need to filter it further.  We can do that by "piping" the output of the join into another [csvkit][csvkit] utility called [`csvgrep`][csvgrep].  Again, have a look at the [documentation for `csvgrep`][csvgrep].

We'll use the pipe symbol `|` which sends the output of one command into another, and then add the csvgrep command:

<code>
csvjoin -c "customer_id,id" --left transactions.csv customers.csv | csvgrep -c "success" -m "true" |
</code>

Not only can we filter for the successful payments, we can also trim the file down to only the columns we need.  You guessed it, there's a utility for that -- it's `csvcut`, which we already used to print the names of the columns.  Here is the [documentation for `csvcut`][csvcut] again.

Examine the column names above and decide which ones you need.

Once again we'll use the pipe symbol `|` to feed the output of one command into the next, and add the csvcut command to the end:

<code>
csvjoin -c "customer_id,id" --left transactions.csv customers.csv | csvgrep -c "success" -m "true" | csvcut -c "customer_name,address,address_2,city,state,zip,amount_in_cents,success"
</code>

I included the name and address fields, and also the amount and success fields, just as a sanity check that nothing went wrong with the filtering.

Note that I picked the customer_name field from the transactions file, then some address fields from the customers file, then went back to transactions for the amount and success fields.  You can re-order the fields in any way you want.

To send the results to a file, append `> paid_orders.csv`.

<code>
csvjoin -c "customer_id,id" --left transactions.csv customers.csv | csvgrep -c "success" -m "true" | csvcut -c "customer_name,address,address_2,city,state,zip,amount_in_cents,success" > paid_orders.csv
</code>

You can then use that file for whatever you need, for example a mail-merge, or to send to a fulfillment service.

## History

I know I learned about csvkit on Twitter, but I didn't remember the details. Looking back, I see that Joe Germuska was one of the authors of the original `csvcut` utility.  I 'know' Joe from way back at Apache Struts.  Never underestimate the power of your social network -- without that connection, I might not have heard about this amazing tool!

## Next Steps

This is only a small part of what [csvkit][csvkit] can do.  The next thing I'd encourage you to check out is [`csvstat`][csvstat] which will give you a summary overview of your data.  Here is a [tutorial on examining data][tutorial].

Have fun!

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site.url] and is licensed [CC BY-NC][cc-by-nc].

## References

* [csvkit][csvkit]
* [csvcut][csvcut]
* [csvjoin][csvjoin]
* [csvgrep][csvgrep]
* [csvstat][csvstat] and the [tutorial][tutorial]
* [Chargify][chargify]

[chargify]: https://www.chargify.com
[chargify-signup]: https://app.chargify.com/signup/developer3
[csvkit]: https://csvkit.readthedocs.org/en/0.9.1/index.html
[csvjoin]: http://csvkit.readthedocs.org/en/latest/scripts/csvjoin.html
[csvcut]: http://csvkit.readthedocs.org/en/latest/scripts/csvcut.html
[csvgrep]: http://csvkit.readthedocs.org/en/latest/scripts/csvgrep.html
[csvstat]: http://csvkit.readthedocs.org/en/latest/scripts/csvstat.html
[tutorial]: https://csvkit.readthedocs.org/en/0.9.1/tutorial/2_examining_the_data.html
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site.url]: {{ site.url }}
