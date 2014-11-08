---
layout: post
title:  "Appending Data in mongoDB"
date:   2014-11-07 19:00:00
tags: mongodb ruby
---
There is some documentation on updating mongoDB documents on the [mongo gem](http://api.mongodb.org/ruby/current/) page, and a little more on the wiki [Tutorial](https://github.com/mongodb/mongo-ruby-driver/wiki/Tutorial#updating-documents-with-update) page.

From those I learned how to change the value of a single field using <code>$set</code> (in Ruby):

{% highlight console %}
coll.update( { "doe" => "3BL", "birth_date" => "2014-10-24" }, 
             { "$set" => {"survived" => 9 } } )
{% endhighlight %}

(Note:  Dates are still strings at this stage of the project.)

Nice, but I need to append data, not just replace it. Working with this document:

{% highlight console %}
{"type" => "litter", "litter_id" => "43", "doe" => "3BL", "buck" => "C16", 
  "birth_date" => "2014-10-24", "kindled" => 10, "survived" => 9,
  "weights" => [
    {"weight" => 0.5, "date" => "2014-10-29", "notes" => "well fed"},
    {"weight" => 0.7, "date" => "2014-10-31", "notes" => "doing fine"}
   ],
  "retained" => ["431","436"]
}
{% endhighlight %}

I need to add information to both of the arrays, <code>weights</code> and <code>retained</code>.

The [mongoDB documentation on update](http://docs.mongodb.org/manual/reference/operator/update/) says that the answer is <code>$push</code>, but the  [Modify Document tutorial](http://docs.mongodb.org/manual/tutorial/modify-documents/) has no examples.  I finally found some on [this page](http://docs.mongodb.org/manual/reference/operator/update/push/).

Appending data looks like this:

{% highlight console %}
require 'mongo'
db = Mongo::Connection.new.db("mydb")
coll = db["nivens"]

coll.update( { "doe" => "3BL", "birth_date" => "2014-10-24" }, 
             { "$push" => {"retained" => "434"} } )

new_weight = { "weight" => 0.9, "date" => "2014-11-05", "notes" => "very fat" }
coll.update( { "doe" => "3BL", "birth_date" => "2014-10-24" }, 
             { "$push" => { "weights" => new_weight } } )
{% endhighlight %}

The first parameter to update selects the litter we want to update by giving the doe's identification code along with the birth date.  The second parameter adds a new value to the end of the specified array.

After these changes, the document looks like this:

{% highlight console %}
{"_id"=>BSON::ObjectId('545d155b7e12bd6f08000001'), 
"type"=>"litter", "litter_id"=>"43", "doe"=>"3BL", "buck"=>"C16", 
"birth_date"=>"2014-10-24", "kindled"=>10, "survived"=>9,
   "weights"=>[
      {"weight"=>0.5, "date"=>"2014-10-29", "notes"=>"well fed"}, 
      {"weight"=>0.7, "date"=>"2014-10-31", "notes"=>"doing fine"},
      {"weight"=>0.9, "date"=>"2014-11-05", "notes"=>"very fat"}
	], 
   "retained"=>["431", "436", "434"]}
{% endhighlight %}

You can see the new values at the end of the <code>weights</code> and <code>retained</code> arrays.