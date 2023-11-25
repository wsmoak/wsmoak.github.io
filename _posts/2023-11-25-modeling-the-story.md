---
layout: post
title:  "Modeling the Story"
date:   2023-11-25 11:00:00 -0400
tags:   data-vault modeling
---

I'm using [Ellie.ai](https://ellie.ai) to model my domain for a Data Vault project.  They have some excellent blog posts on how to get started.  Here are a couple of them:

[Identifying Entities](https://www.ellie.ai/blogs/modelers-corner-1-identifying-entities)


[Modeling Verbs and Nouns](https://www.ellie.ai/blogs/modelers-corner-3-modeling-verbs-and-nouns)

In Ellie, you add items to the glossary, and then you can use them in a model.  This makes sure everyone is in agreement about what the entities are before they get used in a model.  Just definining what a "Customer" is can take quite a while at many places! Ellie helps tremendously with this.

As described in the 'Modeling Verbs and Nouns' post linked above, the first thing to do is write some sentences about what is happening, and pick out the nouns which will be the main entities in the model.

I am modeling my personal finances rather than a business, but since I interact with businesses, many of the same entities I would see in a work project are going to be present, just perhaps from the other side.  Then again, businesses are customers of other businesses just like individuals are customers.  Let's see what happens.

A person is an employee who works for a company and receives a paycheck.

A person shops at a store and receives a receipt detailing the items purchased, discounts, taxes, and the total amount paid.

A person receives a credit card statement detailing the purchases, credits, payments, and refunds that happened during a specific time period.

A person receives a bank statement detailing the withdrawals and deposits that happened during a specific period of time.

A person splits a financial transaction into multiple categories such as income, groceries, taxes, hobbies, or books.

That's enough to start with.

From this (and from what I already know) I can see that there are a few main entities: Person, Financial Transaction, Item, Category.

After some pondering and rearranging, here's what I have so far:

![Store Purchase Model](/images/2023/11/wendy-Store-Purchase-2023-11-25-1035.png)

This started as the model for purchasing items at a store, but it also covers the employee receiving a paycheck with no changes other than the relationship at the top left which could be 'is employed by' instead.
 
Item will eventually get more complicated as I want to be able to track pricing data from advertisements and purchase receipts, but this will work for now.
