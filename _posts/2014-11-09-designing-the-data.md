---
layout: post
title:  "Designing the Data"
date:   2014-11-09 18:30:00
tags: mongodb nosql
---

Designing the Data, or, Why I Ditched Rails (for now).

In the quest to check off the "Learn Ruby" task on my todo list (as well as the "Learn Rails" sub-task) I decided to work with the data from rabbit breeding.

After setting up a simple Rails app with scaffold I was happy to be able to add, view and delete rabbit records.  But of course the domain is not that simple.  There are litters, which have an associated parent doe and buck, and for which we need to track a series of weights with a date and short note.

I quickly found myself surrounded by drawings of how the database tables needed to look to make this all happen, and then reading up on Active Record and database migrations.  A couple of hours went by... and I had not written any Ruby code.  This wasn't going to work!

I've been keeping an eye on the NoSQL movement because I have a history in Multi-Value or "extended relational" databases. (IBM UniData, anyone?)  CouchDB was the first one I played with, years ago, but I never found time to really use it.

Now, though, I really needed something I could just stuff data into without worrying about defining the schema beforehand.  Enter mongoDB.  And since Rails is pretty opinionated about using ActiveRecord and a normal database, I ditched it in favor of Sinatra.

I whipped up a script to load some sample data, copied and pasted the sample Sinatra app, and I was back in business.

Now when the model changes, I just clear out the database, re-load it with my new idea of how things should look, and continue on.

Once the data model stabilizes, I will probably re-implement this in Rails (and Express.js, and Flask, and...) as a learning experience.