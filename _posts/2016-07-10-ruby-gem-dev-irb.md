---
layout: post
title:  "Ruby Gem Development with IRB"
date:   2016-07-10 21:45:00
tags: ruby gem irb
---

This weekend I decided to work on a utility to... well, we'll get back to the original purpose in the next post.

I began by staring at an empty console window and trying to remember how to start a project in Ruby, then consulting the [blog post][rps] I wrote the last time I tried something like this.  Oh, right!

{% highlight bash %}
$ bundler gem --test-rspec currency_utils
{% endhighlight%}

Followed by the usual, change into the directory and commit the initial files.

{% highlight bash %}
$ cd currency_utils
$ git commit -m "Initial commit of generated project"
{% endhighlight%}

And just to check that all is well with the setup:

{% highlight bash %}
$ bundle exec rspec

[...]
Finished in 0.00179 seconds (files took 0.43379 seconds to load)
2 examples, 1 failure

Failed examples:

rspec ./spec/currency_utils_spec.rb:8 # CurrencyUtils does something useful
{% endhighlight%}

That's expected. The generated project includes a test to check that the project has a version, which passes, and one that asserts it "does something useful," which fails.

Next I added a class and a method to the generated module, and I wanted to try it out in irb or pry.  But it wasn't there.

{% highlight bash %}
$ irb
> CurrencyUtils
NameError: uninitialized constant CurrencyUtils
{% endhighlight %}

Okay... in the Rails console this all Just Works and I don't have to think about it.  But people develop gems all the time, so this must be a solved problem.

I searched, but Google and Stack Overflow did not have anything that helped.

I tried a few things like `load`ing the `lib/currency_utils.rb` file, but that choked on the `require` statement that is generated into that file.  I tried it in a 'plain' (non gem) project with just the code in a file, and _that_ worked okay.  But I need it to work in a project where the code is under 'lib'.

I have a vague grasp of `require` and `$LOAD_PATH` in Ruby, but not enough to know the answer to this, so I traipsed off to the #ruby channel on Freenode IRC and suffered the usual abuse [1] to learn that what I needed was to use the `-I` switch when starting `irb` and to specify the `lib` directory so that it will be on the loadpath.

Now I can do:

{% highlight bash %}
$ irb -I lib
> require 'currency_utils'
=> true
> CurrencyUtils::VERSION
=> "0.1.0"
{% endhighlight %}

But this still doesn't get me to the point where typing `irb` Just Works the way `rails console` does.  [This article][auto-load] gets us a bit further:

{% highlight bash %}
$ irb -I lib -r currency_utils
{% endhighlight%}

Now `lib` is on the load path and I don't have to type `require 'currency_utils'` after irb starts up.

I have a hard time believing people type that out all the time though.

For now I've got it aliased in `~/.bash_profile` -- this assumes that the directory name matches the project name, meaning if you're in the `whatever` [directory][dirname] there is a file named `lib/whatever.rb`:

{% highlight bash %}
alias irb='irb -I lib -r ${PWD/*\/}'
{% endhighlight %}

How do YOU use irb (or pry) when working on a gem project?

## References

* [Ruby Project Structure][rps]
* [Automatically load project's environment to irb][auto-load]
* [Show only current directory name][dirname]
* [How do gem's work][hdgw]
* [Understanding Ruby's Load Paths][urlp]
* [Exploring how to configure IRB][htc]

[rps]: http://wsmoak.net/2015/02/22/ruby-project-structure.html
[dirname]: http://superuser.com/questions/60555/show-only-current-directory-name-not-full-path-on-bash-prompt
[hdgw]: http://www.justinweiss.com/articles/how-do-gems-work/
[urlp]: http://stackoverflow.com/questions/6671318/understanding-rubys-load-paths
[htc]: http://tagaholic.me/2009/05/29/exploring-how-to-configure-irb.html
[auto-load]: http://stackoverflow.com/questions/5424905/automatically-load-projects-environment-to-irb
