---
layout: post
title:  "How a Data Model Emerges"
date:   2014-11-12 10:30:00
tags: nivens mongodb
---

In [Designing the Data](/2014/11/09/designing-the-data.html) I talked about getting frustrated with Rails and Active Record because it required me to figure out too much about the data model before I could start writing the application.

Here's an example of something that just happened:

~~~ diff
 diff --git a/mongo-ruby/load.rb b/mongo-ruby/load.rb
 index d5147b5..f6b7836 100644
 --- a/mongo-ruby/load.rb
 +++ b/mongo-ruby/load.rb
 @@ -48,7 +48,12 @@ litters.insert(
    {:id => "43", :doe => "3BL", :buck => "C16", :birth_date => to_utc(2014,10,24),
      :kindled => 2, :survived => 2,
      :weights => [
 -      { :weight => 2.24, :count => 2, :date => to_utc(2014,11,10), :notes => "" }
 +      { :date => to_utc(2014,11,10),
 +        :data => [
 +          { :weight => 1.14, :count => 1, :id => "", :notes => "" },
 +          { :weight => 1.10, :count => 1, :id => "", :notes => "" }
 +        ]
 +      }
      ]
    }
  )
~~~

I went out and weighed Doe 3BL's current litter of two kits, and wrote down 1.14 and 1.10 (weights are in pounds).  Originally I stored this as:

~~~
 { :weight => 2.24, :count => 2, :date => to_utc(2014,11,10), :notes => "" }
~~~

It's not unusual to weigh the entire litter together while the kits are very small, but this was the first time I had weighed these guys and they were big enough to do separately.  

On further reflection, I decided I didn't want to throw away the data about individual weights.  So I tried this:

~~~
  { :weight => 1.14, :date => to_utc(2014,11,10), :notes => "" }
  { :weight => 1.10, :date => to_utc(2014,11,10), :notes => "" }
~~~

That doesn't look right.  And it's going to cause problems down the line when I try to query the data and calculate totals and averages for a particular date.

So I pushed that information down under the date as an array of <code>data</code>.

~~~
       { :date => to_utc(2014,11,10),
         :data => [
           { :weight => 1.14, :count => 1, :id = "", :notes => "" }
           { :weight => 1.10, :count => 1, :id = "", :notes => "" }
         ]
       }
~~~

Now I have to re-work the data entry form, but what I _don't_ have to do is mess with any SQL database tables!

([Here](https://github.com/wsmoak/nivens/blob/137e5b5e49340c656660c9aac6b870c8eba9817c/mongo-ruby/load.rb) is the script that loads the mongoDB collections.)
