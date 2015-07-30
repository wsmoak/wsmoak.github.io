---
layout: post
title:  "Re-Ordering Components on Chargify Hosted Signup Pages"
date:   2015-07-25 14:56:00
tags: chargify javascript jquery
---

[Chargify][chargify] offers hosted [Public Signup Pages][psp] so that you don't have to worry about handling credit card data when your customers sign up for your product or service.  If you use [Components][components] as part of your pricing model, you may have noticed that they always appear in alphabetical order, and that there is no option to re-order them.  If you need to display them in a different order, it's not hard to do with a little [Custom JavaScript][custom-js] (JQuery).

Here is part of a signup page listing some types of fruit that you can add to your monthly order:

![Original Component Order](/images/2015/07/chargify-psp-component-order-1.png)

Examining the source of the page, we find that the components are wrapped in `<div>` elements:

{% highlight html%}
  <h2>Configure Your Plan:</h2>
  <div class="component_configuration" id="component_configuration">
    <div class="row" id="component_row_109264">
      <p class="left">
      <input id="components__component_id" name="components[][component_id]" type="hidden" value="109264" />
      <label class="component-label" for="component_allocated_quantity_109264">Apples</label><br>
      [...]
    </div>
    <div class="row" id="component_row_109266">
      <p class="left">
      <input id="components__component_id" name="components[][component_id]" type="hidden" value="109266" />
      <label class="component-label" for="component_allocated_quantity_109266">Bananas</label><br>
      [...]
    </div>
  [...]
  </div>
{% endhighlight %}

<strike>Unfortunately, the `div`'s for each component do not have `id`'s</strike><b>UPDATE: The `div` for each component now has a unique `id`!</b>

Because of this, you can easily select them:

{% highlight javascript %}
var comp1 = $( "#component_row_109264" );
var comp2 = $( "#component_row_109266" );
var comp3 = $( "#component_row_109265" );
var comp4 = $( "#component_row_109267" );
{% endhighlight %}

And having done that, you can re-order them however you like:

{% highlight javascript %}
comp3.insertBefore(comp2);
comp4.insertBefore(comp1);
{% endhighlight %}

This will reverse the order of the second and third components, and then move the fourth one to the top.  Now the components are listed in a different order:

![New Component Order](/images/2015/07/chargify-psp-component-order-2.png)

It can also be done without the intermediate var assignment:

{% highlight javascript %}
$( "#component_row_109265" ).insertBefore( $("#component_row_109266") );
$( "#component_row_109267" ).insertBefore( $("#component_row_109264") );
{% endhighlight %}

As you might expect, in addition to `insertBefore` there is also [`insertAfter`][jquery-insertafter], and you can search the JQuery docs to find out what else is available.

Feel free to fork and experiment with this JSFiddle: <http://jsfiddle.net/wsmoak/1maef2ow/6/>

## Setup

Note that you will need to replace the numbers with the ids of your own components, which can be found on the Setup tab in your Chargify account.

![Chargify Setup Tab](/images/2015/07/chargify-setup-tab.png)

![New Component Order](/images/2015/07/chargify-component-ids.png)

Once you have the code working the way you want it to, return to [Chargify][chargify] and edit your Public Signup Page.  Paste the code into the "Custom JavaScript" area (you may need to un-check the 'Use default' checkbox first) and then save your changes.

Be sure to thoroughly test your signup pages before launching them!

## References

* [How to reorder divs in JQuery][3]
* [JQuery insertAfter][jquery-insertafter]
* [Chargify][chargify]
* [Chargify Custom Javascript & CSS][custom-js]

[3]: http://stackoverflow.com/questions/10088496/jquery-reorder-divs
[chargify]: https://www.chargify.com
[psp]: https://docs.chargify.com/public-pages-intro
[components]: https://docs.chargify.com/product-components
[custom-js]: https://docs.chargify.com/custom-javascript-css
[jquery-insertafter]: http://api.jquery.com/insertafter/
