---
layout: post
title:  "SICP in Clojure"
date:   2014-11-06 21:26:00
tags: sicp clojure
---
Working through [SICP](http://mitpress.mit.edu/sicp/) seems to be a popular thing for developers. I was exposed to it briefly in school, but but didn't finish it.  I've picked it up occasionally since then, but never made much progress.

Now with a nascent interest in Clojure, I'm starting over.  And I wondered, can I do the exercises in Clojure?  I gave it a try:

{% highlight console %}
user=> 486
486
user=> (+ 137 349)
486
user=> (- 1000 334)
666
user=> (/ 10 5)
2
{% endhighlight %}

So far, so good, but...

{% highlight console %}
user=> (define size 2)

CompilerException java.lang.RuntimeException: Unable to resolve symbol: define
in this context, compiling:(NO_SOURCE_PATH:1:1) 
{% endhighlight %}

Okay, that's not so good.  A look at the docs shows that in Clojure it's "def" rather than "define", so

{% highlight console %}
 user=> (def size 2)
 #'user/size
user=> size
2
{% endhighlight %}

Yay!  Moving on...

{% highlight console %}
user=> (def (square x) (* x x))

CompilerException java.lang.RuntimeException: First argument to def must 
be a Symbol, compiling:(NO_SOURCE_PATH:1:1) 
{% endhighlight %}

I don't know enough yet to get past that one.  

I looked to see if anyone else was doing this, and Google turned up [SICP in Clojure](http://sicpinclojure.com) and an [SICP Distilled Kickstarter](https://www.kickstarter.com/projects/1751759988/sicp-distilled).
The former looks to be abandoned, but the Kickstarter looks promising!
It was funded at over 3.5 times the goal, and promises to make all the resources free after 12 months.

Meanwhile, I'll go back to Scheme for my work on the SICP exercises.