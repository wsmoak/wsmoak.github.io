---
layout: post
title:  "Fun with mongoDB and GeoJSON"
date:   2014-11-24 19:00:00
tags: mongodb geojson javascript node
---


One morning someone asked for help on #mongodb...

<pre>
  dreambox: hey guys, I'm trying to do a within() geometry thingie...
  dreambox: I'm not a DB guy , total noob so please be gentle
  dreambox: I get a bad request.
</pre>

...with this (reformatted, partial) sample code

{% highlight javascript %}
/**
 * List of Messages
 */
var myCoords = [[[ 4.32, 50.84 ],[ 4.01, 50.1 ],[ 4.12, 50.84 ],[ 4.52, 51.84 ]]];
exports.list = function(req, res) { 
  Message.find()
    .where('location')
    .within().geometry({ type: 'Polygon', coordinates: myCoords })
    .sort('-created')
    [...] 
};
{% endhighlight %}

Geometry? Within? I've never seen this before but it looks like fun!

An appeal to Google turned up 
[mquery](https://github.com/aheckmann/mquery#geometry) which matches the syntax and appears to be an interface to the mongoDB  [geoWithin](http://docs.mongodb.org/manual/reference/operator/query/geoWithin/#op._S_geoWithin) query operator.

I still wasn't sure what the data to be queried should look like, until I found this [primer on geospatial data and mongoDB](http://blog.mongolab.com/2014/08/a-primer-on-geospatial-data-and-mongodb/).

Now it's starting to make sense.

I added some locations to documents in the app I'm working on (even though it doesn't really make sense for a rabbit to have a location... they pretty much stay in the cage where you put them.)

{% highlight ruby %}
rabbits.insert ( {:id=>"3BL", :sex=>"F", :birth_date => to_utc(2014,03,04),
  "parent_buck" => "C4", "parent_doe" => "C3",  
  "loc" => {
    "type" => "Point",
    "coordinates" => [ 0.5 , 0.75 ]
  }})
{% endhighlight %}

...and tried the OP's query.  That caused a syntax error...

~~~
  TypeError: Object #<Cursor> has no method 'where'
~~~

...because I'm not using the Mongoose ORM.

I changed my query to plain mongoDB syntax and asked it to find all the rabbits within the one-unit box starting at [0,0]:

{% highlight javascript %}
    rabbits.find( { loc: 
      { $geoWithin: 
        { $geometry: 
          { type: 'Polygon', coordinates : [[[0,0],[0,1],[1,1],[1,0],[0,0]]] }
        }
      }
    });
{% endhighlight %}

It correctly returned the rabbit (above) at [ 0.5, 0.75 ], but none of the other ones. Success!

One of the issues with the OP's query is the coordinates for the polygon.  It needs to be a closed shape with the first point being the same as the last.  (I wonder what happens if you give it something like a figure eight??)

Another possible issue for the OP was the use of a legacy embedded document format for the location data instead of the newer [GeoJSON](http://geojson.org) format shown above.  Since this is new development, there's no reason to use the legacy format.

Later, I created an example app that lets you add and delete items, which have a location, and then query for them.  You can find it on GitHub at [geojson-example](https://github.com/wsmoak/geojson-example), and here are some screen shots:

![list of items](/images/2014/11/geojson-items.png)

![query results](/images/2014/11/geojson-query-results.png)