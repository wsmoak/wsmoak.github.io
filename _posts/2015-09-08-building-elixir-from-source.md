---
layout: post
title:  "Building Elixir From Source"
date:   2015-09-08 08:43:00
tags: elixir
---

With Elixir 1.1.0-beta I made the switch from whatever version of Elixir was available in Homebrew, to bleeding edge compiled from source.  It was surprisingly easy!

### Get the Latest Source

Following the [installation][elixir-lang-install] instructions:

{% highlight bash %}
$ git clone https://github.com/elixir-lang/elixir.git
$ cd elixir
{% endhighlight %}

In this case I want to clone directly from the elixir-lang repo, not from my fork.  This is what I'll use to execute most of my Elixir programs locally, so unless I'm specifically testing a change I'm making, I don't want to use my own fork, where I may be swapping branches back and forth.

### Change the Version Number

The convention seems to be to leave the version number alone after a release is tagged, which means that if there have been any commits since the release, and you simply build from the source on the master branch, you will build something *different* than already exists for that name.  This can complicate bug reporting if you say you were using "1.1.0-beta" and yours isn't the same as someone else's.

Since I know the latest tagged release was 1.1.0-beta, let's see where that appears in the source:

{% highlight bash %}
$ git grep 1.1.0-beta
CHANGELOG.md:## v1.1.0-beta (2015-09-05)
VERSION:1.1.0-beta
src/elixir.app.src: {vsn, "1.1.0-beta"},
{% endhighlight %}

Based on this I need to change the version number in two places, the `VERSION` file and the `src/elixir.app.src` file:

It's not a good idea to make changes directly on master of someone else's repository, so I'm going to make a branch.  I often use the date for the branch name if I'm not working on something specific.

{% highlight bash %}
$ git checkout -b 20150907
# edit src/elixir.app.src and VERSION to: 1.1.0-20150907
{% endhighlight %}

Now review the changes.  In particular MAKE SURE that your editor does not add a newline to the `VERSION` file, or you may see failing tests in IEx.

If you are a `vim` user you can avoid this by editing the file in binary mode:  `vim -b VERSION`.

{% highlight diff %}
$ git diff
diff --git a/VERSION b/VERSION
index 2ff3577..ec10cd1 100644
--- a/VERSION
+++ b/VERSION
@@ -1 +1 @@
-1.1.0-beta
\ No newline at end of file
+1.1.0-20150907
\ No newline at end of file
diff --git a/src/elixir.app.src b/src/elixir.app.src
index 3c099d4..e63cf7e 100644
--- a/src/elixir.app.src
+++ b/src/elixir.app.src
@@ -1,6 +1,6 @@
 {application, elixir,
 [{description, "elixir"},
- {vsn, "1.1.0-beta"},
+ {vsn, "1.1.0-20150907"},
  {modules, [
  elixir
   ]},
{% endhighlight %}

### Build From Source

And then build and test it:

{% highlight bash %}
$ make clean test
{% endhighlight %}

If all the dots are green, carry on!

### Modify Environment

Next I modified my `~/.bash_profile` so that the newly built `elixir` and `iex` scripts would be **first** on my path:

{% highlight bash %}
export ELIXIR_HOME=/Users/wsmoak/projects/elixir/bin
export PATH=$ELIXIR_HOME:$PATH
{% endhighlight %}

It's important to make sure that $ELIXIR_HOME appears in the PATH *before* `/usr/local/bin` where Homebrew puts the things it is managing.

I used to always start a new terminal session to get bash profile changes to take effect, but eventually I learned about:

{% highlight bash %}
$ source ~/.bash_profile
{% endhighlight%}

And now

{% highlight bash %}
$ elixir -v
Elixir 1.1.0-20150907

$ iex -v
Erlang/OTP 18 [erts-7.0.3] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Elixir 1.1.0-20150907

$ which elixir
/Users/wsmoak/projects/elixir/bin/elixir
{% endhighlight %}

### Updating and Re-Building

When you come back for your next Elixir programming session, here's how to update to the lastest code on master.

**NOTE:**  It's probably a good idea to shut down anything that is using `elixir` or `iex`.  In my brief test, when the `elixir/bin` directory was yanked out from under a running process, a Phoenix app started throwing errors, while iex seemed to survive.

{% highlight bash %}
$ cd /path/to/elixir
$ git checkout master
$ git pull
$ git checkout -b YYYYMMDD
# edit VERSION and src/elixir.app.src
$ make clean test
{% endhighlight %}

It's up to you whether to commit the version number change on the branch, or just revert it:

{% highlight bash %}
$ git checkout VERSION src/elixir.app.src
ingenii:elixir wsmoak$ git status
On branch YYYYMMDD
nothing to commit, working directory clean
{% endhighlight %}

Either way, the branch is there to remind me what revision I was working with on a given day, in case things start going wrong.

If I needed to update again the same day, I would just add the time to the branch name so it is unique.

### Switch back to Homebrew'ed Elixir

If something goes wrong, simply revert your changes to ~/.bash_profile, making sure that `/usr/local/bin` is on your PATH.

Start a new terminal session and check `which elixir` and `elixir -v` to see what is in use.

### Uninstall Homebrew'ed Elixir

I uninstalled the Homebrew version of Elixir I had been using because I'd rather be in control of all the bits. :)

{% highlight bash %}
$ brew uninstall elixir
# it complained that there was still another version installed
$ brew uninstall --force elixir
{% endhighlight %}

### Build From Source Distribution

If something goes wrong using the latest compiled version from master we may want to drop back to the most recent release to compare behavior.

Since I uninstalled Elixir 1.0.5 from homebrew, how do I get it back?

There are precompiled and source archives listed on the release page:

<https://github.com/elixir-lang/elixir/releases/tag/v1.0.5>

I'm going to download elixir-1.0.5.tar.gz into ~/Downloads folder and then expand and build it in my /Applications directory because that's where I keep software that I use as-is.

{% highlight bash %}
$ cd /Applications
$ tar -xzvf ~/Downloads/elixir-1.0.5.tar.gz
$ cd elixir-1.0.5
$ make
{% endhighlight %}

Now I'll add a line to ~/.bash_profile so that I can easily switch versions by un-commenting one or the other:

{% highlight bash %}
export ELIXIR_HOME=/Applications/elixir-1.0.5/bin
{% endhighlight %}

Recall that earlier we also added `ELIXIR_HOME` to the `PATH`:

{% highlight bash %}
export PATH=$ELIXIR_HOME:$PATH
{% endhighlight %}

**Note:**  This also works with the 1.1.0-beta source distribution, which you can download from <https://github.com/elixir-lang/elixir/releases/tag/v1.1.0-beta>

### DIY Source Archive

If for some reason there isn't a source distribution available for a given tag, you can make one with `git archive`.  Here's an example using the v1.0.5 tag:

Make sure you're in the same directory as the clone of elixir-lang/elixir from above:

{% highlight bash %}
$ cd /path/to/elixir
$ git archive --prefix=elixir-1.0.5/ -o elixir-1.0.5.tar v1.0.5
{% endhighlight %}

This will create an uncompressed tar with the contents of the v1.0.5 tag, with a top-level directory of elixir-1.0.5.

### Clone the v1.0.5 Tag

Alternately, here's how to clone the v1.0.5 tag
directly from the repository.  This will take up more space on disk because it has all the git metadata included, but is preferable if you ever need to make changes, so that `git diff` will work.

{% highlight bash %}
$ git clone --branch v1.0.5 https://github.com/elixir-lang/elixir.git  elixir-1.0.5
Cloning into 'elixir-1.0.5'...
remote: Counting objects: 81849, done.
remote: Compressing objects: 100% (60/60), done.
remote: Total 81849 (delta 17), reused 0 (delta 0), pack-reused 81787
Receiving objects: 100% (81849/81849), 29.59 MiB | 1010.00 KiB/s, done.
Resolving deltas: 100% (45201/45201), done.
Checking connectivity... done.
Note: checking out '3eb938a0ba7db5c6cc13d390e6242f66fdc9ef00'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

$
{% endhighlight %}

Since I'm not going to make changes, being in a 'detached HEAD' state is fine.

Now build it.  Remember to check the README file of the branch/tag you are working with.  In this case the instructions are the same.

{% highlight bash %}
$ cd elixir-1.0.5
$ make clean test
{% endhighlight %}

In this case it's not necessary to change the version number, because this is the official tagged source for the 1.0.5 version.

Now I can edit ~/.bash_profile to switch the location of ELIXIR_HOME and control which version is used.

Remember to start a new terminal session or execute `source ~/.bash_profile` to reload changes.

### Wrap Up

We've seen how to build multiple versions of Elixir from source and how to switch between them. This will allow us to

* test out beta versions,
* reproduce bugs that people report on specific versions, and
* check whether bug are still there on master.

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

### References

* [Elixir Installation Instructions][elixir-lang-install]
* <http://git-scm.com/docs/git-archive>
* <http://stackoverflow.com/questions/791959/download-a-specific-tag-with-git>

[elixir-lang-install]: http://elixir-lang.org/install.html
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/4.0/
[site-url]: {{ site.url }}
