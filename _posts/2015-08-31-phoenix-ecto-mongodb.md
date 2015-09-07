---
layout: post
title:  "Phoenix and Ecto with MongoDB"
date:   2015-08-31 08:43:00
tags: elixir phoenix ecto mongodb
---

I saw (on [@ElixirStatus][elixirstatus]) that the [Ecto adapter for MongoDB was released][release], and wanted to try it out.  Looking around, I found that while the adapter itself was in the hex repository, the [PR][pull-1161] that would let the Phoenix installer (the phoenix.new mix task) know about mongodb had not yet been merged.  Let's see how to try out someone else's [as-yet-unmerged changes][branch]!

**UPDATE**: As of 2015-09-06 [PR 1161][pull-1161] was merged and this feature is available in Phoenix 1.0.2.  Now `mix phoenix.new my_project --database mongodb` will work with no need to build it yourself.  [Install][install] the latest version and then skip down to [Create A Project](#create-a-project) to try it out. (You'll need to leave off the `--dev` switch.)

**NOTE**: This involves using un-released code on master and a feature branch.  If you're interested in playing on the bleeding edge, follow me!  If not, you may want to wait until this makes its way into a release.

### Clone A Fork

There are [at least] two ways to go about this.  One is to simply clone the other contributor's fork and grab their branch directly.  It looks like this:

{% highlight bash %}
$ git clone https://github.com/michalmuskala/phoenix.git phoenix-michalmuskala
$ cd phoenix-michalmuskala
$ git checkout -b mongodb_installer origin/mongodb_installer
$ git fetch origin mongodb_installer
{% endhighlight %}

Now we have Michał's fork of the Phoenix Framework repository cloned locally in a directory called "phoenix-michalmuskala" * and we can work with it.

* (to avoid stepping on our _own_ fork which is probably in a directory called "phoenix")

### Add A Remote

Another way is to add their repo as a remote to your own fork of Phoenix, and then apply their changes to a branch of your own.

First, make sure your master branch is up to date.  See [syncing a fork][syncing-a-fork] for more info.  If you want to play on the bleeding edge, this is something you'll get in the habit of doing daily:

{% highlight bash %}
$ cd /path/to/phoenix
$ git checkout master
$ git fetch upstream
$ git merge upstream/master
$ git push
{% endhighlight %}

Now if you check your fork on GitHub, it should say "This branch is even with phoenixframework:master".

![GitHub branch ahead of upstream master](/images/2015/08/github-fork-even-with-upstream-master.png)

Now, add Michał's fork as a remote

{% highlight bash %}
$ git remote add michalmuskala https://github.com/michalmuskala/phoenix
{% endhighlight %}

If you check with `git remote -v` you should see it listed:

{% highlight bash %}
$ git remote -v
michalmuskala https://github.com/michalmuskala/phoenix (fetch)
michalmuskala https://github.com/michalmuskala/phoenix (push)
origin  https://github.com/wsmoak/phoenix.git (fetch)
origin  https://github.com/wsmoak/phoenix.git (push)
upstream  https://github.com/phoenixframework/phoenix (fetch)
upstream  https://github.com/phoenixframework/phoenix (push)
{% endhighlight %}

Now create a new local branch, and pull the changes from Michał's branch into it.  I'm using the same branch name, but it's important to note that these are two completely separate branches in different git repositories.

{% highlight bash %}
$ git checkout -b mongodb_installer
$ git status
# check that you are on the mongodb_installer branch so you don't mess up master
$ $ git pull michalmuskala mongodb_installer
{% endhighlight %}

Now you have the absolute latest Phoenix Framework code *plus* Michał's changes.

Optionally, you can push your new branch to GitHub. I usually just type `git push` and let it fail and tell me what to do.  It says:

{% highlight bash %}
$ git push --set-upstream origin mongodb_installer
{% endhighlight %}

Now if you look at your branch you should see it say you are *ahead* of phoenixframework:master.

![GitHub branch ahead of upstream master](/images/2015/08/github-branch-ahead-of-upstream-master.png)

### Verify the Version

However you decided to retrieve the code, change into that directory, and then into the installer subdirectory.

{% highlight bash %}
$ cd /path/to/phoenix
$ cd installer
{% endhighlight %}

In the original [Phoenix on the Bleeding Edge][pbe] post, I learned that typing `mix phoenix.new` will *first* use the version installed into ~/.mix/archives, and then the code in the current directory.

Let's make sure we don't have any other versions of the Phoenix Installer (the phoenix.new mix task) around:

{% highlight bash %}
$ rm ~/.mix/archives/phoenix_new*
{% endhighlight %}

To be absolutely sure, modify the version number in `installer/mix.exs` to something other than the last released version.  I changed mine to "1.0.1-dev" indicating that this is a development or pre-release 1.0.1 version.  You can also just append a dash and your username.  Anything is better than potentially re-building a version number that has already been officially released.  That way lies madness.

{% highlight diff %}
@@ -3,7 +3,7 @@ defmodule Phoenix.New.Mixfile do

   def project do
     [app: :phoenix_new,
-     version: "1.0.0",
+     version: "1.0.1-dev",
      elixir: "~> 1.0-dev"]
   end
{% endhighlight %}

Now check the version that is in use:

{% highlight bash %}
$ mix phoenix.new --version
Phoenix v1.0.1-dev
{% endhighlight %}

If the version it reports matches what you just changed in `installer/mix.exs`, then you can be sure that you're using the code in the current directory, vs. code in an archive somewhere else.

### Create A Project

We're finally ready to try it out!  Remember to use the `--dev` switch as well as the `--database` switch, and to specify `mongodb`.

Double check that you are still in the `installer` directory.

{% highlight bash %}
$ mix phoenix.new my_app --dev --database mongodb
$ cd my_app
{% endhighlight %}

Take a look at `mix.exs` in your generated project and notice that the phoenix dependency is using a relative path to point at the source code two levels up from the project location.  If you move the generated project elsewhere, you'll need to modify this.

**Make sure MongoDB is running locally or the next steps won't work.**

### Generate The Model

Let's go ahead and create the database and a simple model.

{% highlight bash %}
$ mix ecto.create
$ mix phoenix.gen.html User users name:string email:string
# Add the resource to web/router.ex as directed
{% endhighlight %}

Note that there is no need to run `mix ecto.migrate` because there is no table structure to maintain. (And no migration file was created in `priv/repo/migrations`.)  Running it won't hurt, however, it will just tell you "Already up".  I commented on the [PR][pull-1161] mentioning this and there is some discussion about suppressing the message.

### Start the Server

Now for the moment of truth.  Start the server:

{% highlight bash %}
$ mix phoenix.server
{% endhighlight %}

And then visit <http://localhost:4000/users>.  Try out the add/edit/delete functions.

Now for the good part, and why I have been waiting for this before moving on with my side project:  You can add and remove fields at will, without 'migrating' the database structure.

Try adding an "age" field to the model and to the form -- it just works.  Add a column to the index.html.eex template to see the new data displayed.

### MongoDB Console

To see what's been added to the database, fire up the Mongo console

{% highlight bash %}
$ mongo
{% endhighlight %}

{% highlight console %}
> use my_app_dev
switched to db my_app_dev
> db.users.find()
{ "_id" : ObjectId("1d761bdcda362a1e159b0374"), "email" : "sally@example.com", "inserted_at" : ISODate("2015-08-30T19:54:04Z"), "name" : "Sally Jones", "updated_at" : ISODate("2015-08-30T19:54:04Z") }
{ "_id" : ObjectId("1d7634acda362a2add8bbdf4"), "age" : 44, "email" : "kim@example.com", "inserted_at" : ISODate("2015-08-30T21:39:56Z"), "name" : "Kimberly White", "updated_at" : ISODate("2015-08-30T22:32:44Z") }
{ "_id" : ObjectId("1d763670da362a2adde0ce09"), "age" : 52, "email" : "ima@example.com", "inserted_at" : ISODate("2015-08-30T21:47:28Z"), "name" : "Ima Thompson", "updated_at" : ISODate("2015-08-30T21:47:28Z") }
{% endhighlight %}

Notice that one record is missing the "age" attribute because the record was created prior to the new field being added to the form.  And that this causes no problem at all!

This is what I love about MongoDB for development -- I don't have to worry about designing the data store in advance.  I can change it at will until the "right" structure emerges.

### Summary

We've seen how to try out another contributor's as-yet-unmerged changes and gotten a sneak peek at generating a Phoenix project with support for MongoDB.

My branch with the latest (as of 8/30) Phoenix code plus Michał's changes is here:  <https://github.com/wsmoak/phoenix/tree/mongodb_installer>.  A tag with the branch contents as of this post is here: <https://github.com/wsmoak/phoenix/tree/20150830>

The code for this example is available at <https://github.com/wsmoak/my_app_mongodb/tree/20150831> and is MIT licensed.  (Note that the phoenix dependency uses the previously mentioned _branch_, which may have been updated since this was written.)

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

References:

* [Support for Mongodb in installer #1161][pull-1161]
* [Syncing A Fork][syncing-a-fork]
* <http://stackoverflow.com/questions/67699/clone-all-remote-branches-with-git>
* [Adding and Removing Remote Branches](http://www.gitguys.com/topics/adding-and-removing-remote-branches/)
* [Phoenix on the Bleeding Edge][pbe]
* [Getting Started with the Mongo Shell](http://docs.mongodb.org/v3.0/tutorial/getting-started-with-the-mongo-shell/)

[elixirstatus]: http://elixirstatus.com
[release]: http://elixirstatus.com/p/WSat-mongodb-ecto-adapter-released
[branch]: https://github.com/michalmuskala/phoenix/tree/mongodb_installer
[pbe]: http://wsmoak.net/2015/08/17/phoenix-bleeding-edge.html
[pull-1161]: https://github.com/phoenixframework/phoenix/pull/1161
[syncing-a-fork]: https://help.github.com/articles/syncing-a-fork/
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
[install]: http://www.phoenixframework.org/docs/installation
