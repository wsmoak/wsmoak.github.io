---
layout: post
title:  "Ordering Components on Chargify Hosted Signup Pages"
date:   2015-07-25 14:56:00
tags: chargify javascript jquery
---

[Chargify][chargify] offers hosted [Public Signup Pages][psp] so that you don't have to worry about handling credit card data when your customers sign up for your product or service.  If you use [Components][components] as part of your pricing model, you may have noticed that they always appear in alphabetical order, and that there is no option to re-order them.  If you need to display them in a different order, it's not hard to do with a little [Custom JavaScript][custom-js] (JQuery).

Here is part of a signup page listing some fruits that you can add to your monthly order:

![Original Component Order](/images/2015/07/chargify-psp-component-order-1.png)

Examining the source of the page, we find that the components are wrapped in `<div>` elements:

{% highlight html%}
  <h2>Configure Your Plan:</h2>
  <div class="component_configuration">
    <div class="row">
      <p class="left">
      <input id="components__component_id" name="components[][component_id]" type="hidden" value="109264" />
      <label class="component-label" for="component_allocated_quantity_109264">Apples</label><br>
      [...]
    </div>
    <div class="row">
      <p class="left">
      <input id="components__component_id" name="components[][component_id]" type="hidden" value="109266" />
      <label class="component-label" for="component_allocated_quantity_109266">Bananas</label><br>
      [...]
    </div>
  [...]
  </div>
{% endhighlight %}

Unfortunately, the `div`'s for each component do not have `id`'s, so they can't be selected directly.  However, we do know that they are wrapped in a `div` with the `class="component_configuration"` that appears nowhere else on the page, and that the `div` for each component has a `class="row"`.

With some [help][1] [from][2] [Google][3], I determined that you can identify the `div`s containing the components as follows:

{% highlight javascript %}
var comp1 = $( ".component_configuration > .row:nth-child(1)" );
var comp2 = $( ".component_configuration > .row:nth-child(2)" );
var comp3 = $( ".component_configuration > .row:nth-child(3)" );
var comp4 = $( ".component_configuration > .row:nth-child(4)" );
{% endhighlight %}

And having done that, you can re-order them however you like:

{% highlight javascript %}
comp3.insertBefore(comp2);
comp4.insertBefore(comp1);
{% endhighlight %}

This will reverse the order of the second and third components, and then move the fourth one to the top.  Now the components are listed in a different order:

![New Component Order](/images/2015/07/chargify-psp-component-order-2.png)


You can fork and experiment with this JSFiddle: <http://jsfiddle.net/wsmoak/1maef2ow/4/>

Once you have the code working the way you want it to, return to [Chargify][chargify] and edit your Public Signup Page.  Paste the code into the "Custom JavaScript" area (you may need to un-check the 'Use default' checkbox first) and then save your changes.

Be sure to thoroughly test your signup pages before launching them!

## References

* [JQuery nth-child selector][1]
* [JQuery class selector][2]
* [How to reorder divs in JQuery][3]
* [Chargify][chargify]
* [Chargify Custom Javascript & CSS][custom-js]

[1]: https://api.jquery.com/nth-child-selector/
[2]: https://api.jquery.com/class-selector/
[3]: http://stackoverflow.com/questions/10088496/jquery-reorder-divs
[chargify]: https://www.chargify.com
[psp]: https://docs.chargify.com/public-pages-intro
[components]: https://docs.chargify.com/product-components
[custom-js]: https://docs.chargify.com/custom-javascript-css
