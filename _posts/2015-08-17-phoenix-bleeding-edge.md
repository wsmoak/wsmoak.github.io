---
layout: post
title:  "Phoenix on the Bleeding Edge"
date:   2015-08-17 08:55:00
tags: elixir phoenix
---

As the Phoenix Framework nears a 1.0 release, more and more people are going to want to contribute. That means being able to test changes against the latest code on the master branch. There's no sense reporting a bug if it's already been fixed! Let's find out how to do it.

To get started, I forked the [phoenixframework/phoenix][github-phoenix] repo on GitHub, cloned it locally, and looked at the top-level README file.  It says:

> ### Running a Phoenix master app
>
> `$ cd installer`<br/>
> `$ mix phoenix.new path/to/your/app --dev`

So I tried it:

{% highlight bash %}
$ cd ~/projects/phoenix
$ cd installer
$ mix phoenix.new ~/projects/my_bleeding_edge_project --dev
** (Mix) --dev version must be inside Phoenix directory
{% endhighlight %}

Well... that didn't work.  What does it mean?

After some poking around and `IO.inspect`ing, I figured out that when you create a new Phoenix project with the `--dev` switch, that project must be created _inside_ the directory containing the Phoenix source code.

The relevant bit of code is at
<https://github.com/phoenixframework/phoenix/blob/master/installer/lib/phoenix_new.ex#L412>

{% highlight elixir %}
  defp phoenix_path(path, true) do
    absolute = Path.expand(path)
    relative = Path.relative_to(absolute, @phoenix)

    if absolute == relative do
      Mix.raise "--dev version must be inside Phoenix directory"
    end
{% endhighlight %}

To be fair, the instructions DO say `path/to/my/project` which _is_ a *relative* path.

So, trying again (remember we're in the `phoenix/installer` directory)

{% highlight bash %}
$ mkdir projects
$ mix phoenix.new projects/my_bleeding_edge_project --dev
{% endhighlight %}

And that works!

### What Version?

But... what version of the `phoenix.new` mix task are we actually using? Presumably before you decided to move to the bleeding edge, you were tracking the latest release, which you have installed locally:

{% highlight bash %}
$ ls ~/.mix/archives/phoenix*
/Users/wsmoak/.mix/archives/phoenix_new-0.16.1.ez
{% endhighlight %}

So when the version says...

{% highlight bash %}
$ mix phoenix.new --version
Phoenix v0.16.1
{% endhighlight %}

...is that the one in `~/.mix/archives`, or is the one in the directory we're sitting in?  Because `phoenix/installers/mix.exs` says...

{% highlight elixir %}
   def project do
     [app: :phoenix_new,
      version: "0.16.1",
      elixir: "~> 1.0-dev"]
   end
{% endhighlight %}

...so it's not clear.

### Pre-Release Version Numbers

To get a better idea of what's going on, and to avoid ever re-building a changed version of an official release, let's change the version number in that `mix.exs` file.

But first, since we've forked someone else's repository, we need to move off of the master branch.  If I'm not working on anything specific I'll often just use the date for the branch name.

{% highlight bash %}
$ git checkout -b 20150816
Switched to a new branch '20150816'
{% endhighlight %}

And now change the version number:

{% highlight diff %}
@@ -3,7 +3,7 @@ defmodule Phoenix.New.Mixfile do

   def project do
     [app: :phoenix_new,
-     version: "0.16.1",
+     version: "0.16.2-pre",
      elixir: "~> 1.0-dev"]
   end
{% endhighlight %}

Now we'll be able to tell which one we're using, and we won't confuse ourselves by re-building a local 0.16.1 with behavior that differs from the official release.

At this point, `mix phoenix.new --version` still reports 0.16.1, so we must be using the released version under `~/.mix/archives`.

Let's delete that file.

{% highlight bash %}
$ rm ~/.mix/archives/phoenix_new-0.16.1.ez
{% endhighlight %}

What version does mix see now?

{% highlight bash %}
$ mix phoenix.new --version
Phoenix v0.16.2-pre
{% endhighlight %}

Ah ha!  Now we're actually using the _code_ in the installer directory, which means if we want to quickly modify the `phoenix.new` task itself, we can do so.

Let's create another project, just in case the `phoenix.new` task has changed since the 0.16.1 release.

{% highlight bash %}
$ mix phoenix.new projects/another_bleeding_edge_project --dev
"/Users/wsmoak/projects/phoenix-wsmoak/installer/projects/another_bleeding_edge_project"
"installer/projects/another_bleeding_edge_project"
mix.exs:1: warning: redefining module Phoenix.New.Mixfile
Error while loading project :umbrella_check at /Users/wsmoak/projects/phoenix-wsmoak/installer
{% endhighlight %}

(Oops.  It thought we were trying to create an umbrella project with multiple modules.  Despite the error, it doesn't seem to have hurt anything.)

### Building PhoenixNewTask

Having to delete the official release from `~/.mix/archives` isn't ideal.  Can we build the latest version and use it locally without that?  There is a README in the `installer` directory we're sitting in that says how.

{% highlight bash %}
$ MIX_ENV=prod mix archive.build
Generated archive phoenix_new-0.16.2-pre.ez with MIX_ENV=prod

$ mix archive.install
Are you sure you want to install archive phoenix_new-0.16.2-pre.ez? [Yn] Y
* creating /Users/wsmoak/.mix/archives/phoenix_new-0.16.2-pre.ez
{% endhighlight %}

Well, that puts our development version in `~/.mix/archive`, so as expected,

{% highlight bash %}
$ mix phoenix.new --version
Phoenix v0.16.2-pre
{% endhighlight %}

Anecdotally, if there is an archive in `~/.mix/archives`, mix uses that first. Next it looks in `lib` in the current directory. I'm looking forward to learning more about this in the training class at ElixirConf!

### Why

The reason the bleeding edge project has to be created *inside* the phoenix source code directory can be seen in its (the newly created project's) mix.exs:

{% highlight elixir %}
  defp deps do
    [{:phoenix, path: "../../..", override: true},
{% endhighlight %}

It wants to use a relative path to find the phoenix dependency.  From our location underneath the `installer/projects` directory we created, it's backing up three directory levels... which is the top-level of the phoenix source code.

It also overrides any other definition of a dependency on phoenix.

### Recommendations

If you're going to be modifying the phoenix.new task itself, then you'll probably want to `rm ~/.mix/archive/phoenix_new*` to make sure you're *only* working with the local source code, and do your work from the `installer` directory.

If you're patching something in Phoenix outside the installer, then it either may not matter which installer you're using, or you may wish to build the latest one and install the archive locally.

(It also seems possible to move the newly-created project elsewhere, and modify the path to the Phoenix source code, but I haven't tried it.)

Now, off to IRC to ask and to either [do a PR][pr] to improve that error message I got initially, or else find out why it is the way it is!

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

### References

* <https://github.com/phoenixframework/phoenix/blob/master/README.md>
* <https://github.com/phoenixframework/phoenix/blob/master/CONTRIBUTING.md>
* <https://github.com/phoenixframework/phoenix/blob/master/installer/README.md>
* [PR 1137 Improve docs for potential contributors][pr]

[github-phoenix]: https://github.com/phoenixframework/phoenix
[pr]: https://github.com/phoenixframework/phoenix/pull/1137
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
