---
layout: post
title:  "Presenting Source Code with Reveal JS"
date:   2015-09-16 20:32:00
tags: revealjs highlightjs speaking
---

In "Instantly Better Presentations" Damian Conway shows a great way to step through a block of code by highlighting the exact bit you want the audience to focus on at each moment. You can see him describe it at starting at [1:14:58 of this talk][video].  Now let's learn how to do it with [reveal.js][revealjs].

[reveal.js][revealjs] is an HTML based presentation framework.  Check out <http://lab.hakim.se/reveal-js/> for an example of what it can do.

[reveal.js][revealjs] uses [highlight.js][highlightjs] for syntax highlighting.  This will automatically detect the language and highlight anything you put inside `<pre>` and `<code>` elements.  For example you might have a slide like this:

<pre>
&lt;section>
  &lt;pre>&lt;code>
defmodule Math do
  def sum(a, b) do
    a + b
  end
end
  &lt;/code>&lt;/pre>
&lt;/section>
</pre>

This code will appear on a slide in your presentation, nicely syntax highlighted.

It's common to use the `data-trim` attribute on the `<code>` element to trim any surrounding whitespace.

<pre>
&lt;section>
  &lt;pre>&lt;code data-trim>
defmodule Math do
  def sum(a, b) do
    a + b
  end
end
  &lt;/code>&lt;/pre>
&lt;/section>
</pre>

I found [this issue][1133] where someone requested the feature of highlighting a specific line of code be added to reveal.js.  The author said it really belonged in highlight.js.  And the author of highlight.js [said it's already possible][740] by using standard HTML markup.

In short, to highlight some code, surround it with a [`<mark>`][mark] element:

<pre>
&lt;section>
  &lt;pre>&lt;code>
defmodule Math do
  &lt;mark>def sum(a, b) do
    a + b
  end&lt;/mark>
end
  &lt;/code>&lt;/pre>
&lt;/section>
</pre>

Unfortunately, back in reveal.js, this will result in the `<mark>`ed code flashing yellow and then the literal text `<mark>` and `</mark>` appearing in your presentation.

After some investigation, I found that [reveal.js will escape HTML inside the `<code>` block][L14] unless you tell it not to.  And the way to tell it not to is to add a `data-noescape` attribute to the `<code>` element.

<pre>
&lt;section>
  &lt;pre>&lt;code data-trim data-noescape>
defmodule Math do
  &lt;mark>def sum(a, b) do
    a + b
  end&lt;/mark>
end
  &lt;/code>&lt;/pre>
&lt;/section>
</pre>

Now the code will be syntax highlighted, plus the code surrounded by `<mark>` will have a yellow background.

That doesn't look quite right though.  It looks better if you `<mark>` each line separately:

<pre>
&lt;section>
  &lt;pre>&lt;code>
defmodule Math do
  &lt;mark>def sum(a, b) do&lt;/mark>
    &lt;mark>a + b&lt;/mark>
  &lt;mark>end&lt;/mark>
end
  &lt;/code>&lt;/pre>
&lt;/section>
</pre>

![Highlighted Source Code](/images/2015/09/presenting-source-code.png)

If you don't like the yellow background, the CSS can be modified.

And to get the effect Damian describes, you can repeat the same block of code on subsequent slides, each with different parts `<mark>`ed for emphasis.

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

The source for the example presentation is available [in GitHub][source] (see the index.html file) and can be viewed [on GitHub pages][view]. It is MIT licensed.

### References

* [Instantly Better Presentations video at 1:14:58][video]
* [reveal.js][revealjs]
* [highlight.js][highlightjs]
* [HTML mark element][mark]
* [reveal.js issue 1133][1133]
* [highlight.js issue 740][740]

[revealjs]: http://lab.hakim.se/reveal-js/#/
[highlightjs]: https://highlightjs.org/
[video]: https://www.youtube.com/watch?v=W_i_DrWic88&t=1h14m58s
[L14]: https://github.com/hakimel/reveal.js/blob/master/plugin/highlight/highlight.js#L14
[mark]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/mark
[1133]: https://github.com/hakimel/reveal.js/issues/1133
[740]: https://github.com/isagalaev/highlight.js/issues/740
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[site-url]: {{ site.url }}
[source]: https://github.com/wsmoak/presenting-source-code/tree/master
[view]: http://wsmoak.github.io/presenting-source-code
