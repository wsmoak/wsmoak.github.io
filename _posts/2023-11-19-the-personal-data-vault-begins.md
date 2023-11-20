---
layout: post
title:  "The Personal Data Vault Begins"
date:   2023-11-19 19:30:00 -0400
tags:   data-vault dbt python sql
---

Ever since I learned about the Data Vault 2.0 methodology earlier this year, I've been thinking about implementing it for myself as a learning project.

While I don't have a company that needs a data warehouse, I do have a great deal of personal data to organize, and some ideas for
interesting projects.  I'd like to collect all of my financial data in my own database rather than use something like Personal Capital.
And I'd like to gather pricing data so I can tell whether a certain store has the best price on an item, and perhaps do some predictions
about when to stock up on things based on a store's historical pattern of discounts.

Data Vault encourages you to model "the business" first.  In my case that's going to be "the family" and all the related entities such as organizations
which might be stores or employers, and then transactions such as credit card charges and paycheck deposits.  I want to use Ellie.ai to do the modeling, 
alternately Mermaid in Github markdown or Lucidchart will work.

To move the data around, for example from the CSV files I download from my credit union into the raw tables in Postgres, I plan to use a 
combination of SQL and Python, neither of which I am an expert in.  I'm also interested in learning DBT.

One day, there may be a Django web application to view and edit the data, for example to classify expenses and split transactions.
This means it will turn into an operational data vault, such as the one Dan Linstedt talks about doing at [Cendant Timeshare](https://www.youtube.com/watch?v=FS4IERBV3G0).
The records will not be edited, rather new ones will be inserted.

Once it all works locally, I may move it to Snowflake and try out that platform.

This is the first side project I've done in a very long time, follow along to see how it goes!
