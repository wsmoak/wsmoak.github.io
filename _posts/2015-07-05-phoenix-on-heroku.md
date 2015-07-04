---
layout: post
title:  "Deploying a Phoenix app to Heroku"
date:   2015-07-05 14:41:00
tags: elixir phoenix heroku
---

Having (finally) gotten the [Phoenix Framework][phoenix] for [Elixir] installed (/me *stares accusingly at Norton Firewall*) and played with the Hello Phoenix app a bit locally, I decided to deploy it to Heroku.

I was happy to find what looked like a complete set of instructions...

* <http://learnelixir.com/blog/2014/10/15/deploy-phonenix-application-to-heroku-server/>

... but apparently that blog post is outdated.  It *is* over six months old. :) Let's start over.

You will need at _least_ Elixir, Phoenix, NodeJS and the Heroku Toolbelt installed.  Also git, and... the list is long.  If something doesn't work, ask on #elixir-lang or in the comments below, or email me and I'll try to help.

### Step 1: Create a sample app and get it under version control

<pre>
$ mix phoenix.new hello_phoenix_heroku
[...]
Fetch and install dependencies? [Yn] Y
[...]
$ cd hello_phoenix_heroku
$ git init
$ git add .
$ git commit -m "Initial commit of Hello Phoenix app"
</pre>

### Step 2: Create the Heroku application

<pre>
$ heroku create
Creating afternoon-reef-1857... done, stack is cedar-14
https://afternoon-reef-1857.herokuapp.com/ | https://git.heroku.com/afternoon-reef-1857.git
Git remote heroku added
</pre>

(You can specify an application name after `heroku create`, but I enjoy the ones Heroku generates.)

### Step 3: Add [buildpacks][buildpacks] to the Heroku application

A newly created Heroku application does not know anything about the language and frameworks used by the app. [Buildpacks][buildpacks] are used for this configuration.

Execute the commands shown in the [Usage](https://github.com/gjaldon/heroku-buildpack-phoenix-static#usage) instructions for the [Phoenix Static Buildpack][phoenix-static-buildpack]:

<pre>
$ heroku buildpacks:set https://github.com/gjaldon/phoenix-static-buildpack
Buildpack set. Next release on afternoon-reef-1857 will use https://github.com/gjaldon/phoenix-static-buildpack.
Run `git push heroku master` to create a new release using this buildpack.

$ heroku buildpacks:add --index 1 https://github.com/HashNuke/heroku-buildpack-elixir
Buildpack added. Next release on afternoon-reef-1857 will use:
  1. https://github.com/HashNuke/heroku-buildpack-elixir
  2. https://github.com/gjaldon/phoenix-static-buildpack
Run `git push heroku master` to create a new release using these buildpacks.
</pre>

This adds the [Phoenix Static buildpack][phoenix-static-buildpack], and then puts the [Elixir buildpack][elixir-buildpack] in the first position, which pushes Phoenix down to second.  See this article on [using multiple buildpacks for an app][using-multiple-buildpacks] for more info.

### Step 4: Try to deploy

Let's try to deploy and see what happens. (This will take a while the first time because it has to install all the dependencies.)

<pre>
$ git push heroku master
[...]
remote: -----> Fetching app dependencies with mix
remote: ** (Mix.Config.LoadError) could not load config config/prod.secret.exs
[...]
remote:  !     Push rejected, failed to compile elixir app
</pre>

It's complaining that a config file is missing.  If you look in your local directory structure, you will find that the `config/prod.secret.exs` file is present... but if you look in the `.gitignore` file, you will find that it is listed, which means it will not be pushed to Heroku by git.  The comment in .gitignore says:

<pre>
# The config/prod.secret.exs file by default contains sensitive
# data and you should not commit it into version control.
#
# Alternatively, you may comment the line below and commit the
# secrets file as long as you replace its contents by environment
# variables.
/config/prod.secret.exs
</pre>

The `config/prod.secret.exs` file contains information about database configuration that you would not want to commit to your source code repository.
For security reasons this information needs to be kept separate, and setting environment variables is one way to go about it.

### Step 5: Modify prod.secret.exs and .gitignore

So, let's replace the sensitive values in `config/prod.secret.exs` with calls to read the values from the environment, and then comment out that line in `.gitignore` by adding a `#` in front.

`config/prod.secret.exs`
<pre>
use Mix.Config

# In this file, we keep production configuration that
# you likely want to automate and keep it away from
# your version control system.
config :hello_phoenix_heroku, HelloPhoenixHeroku.Endpoint,
  secret_key_base: System.get_env("SECRET_KEY_BASE")

# Configure your database
config :hello_phoenix_heroku, HelloPhoenixHeroku.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: System.get_env("DATABASE_USERNAME"),
  password: System.get_env("DATABASE_PASSWORD"),
  database: "hello_phoenix_heroku_prod",
  size: 20 # The amount of database connections in the pool
</pre>

`.gitignore`
<pre>
[...]
# /config/prod.secret.exs
</pre>

See the Elixir language docs for [System.get_env/0](http://elixir-lang.org/docs/v1.0/elixir/System.html#get_env/0) for more information.

Now commit your changes:

<pre>
$ git add .
$ git commit -m "Include prod.secret.exs and use environment variables"
</pre>

### Step 6: Deploy to Heroku

Since this app does not actually use the database, if we deploy it now, it should work.  Let's see it in action:

<pre>
$ git push heroku master
[...]
remote: -----> Compressing... done, 82.0MB
remote: -----> Launching... done, v4
remote:        https://afternoon-reef-1857.herokuapp.com/ deployed to Heroku
remote:
remote: Verifying deploy.... done.
To https://git.heroku.com/afternoon-reef-1857.git
   853a1d0..c207cc8  master -> master
</pre>

... and success! <https://afternoon-reef-1857.herokuapp.com>

Screen shot for posterity, since an unused app won't stay running for long on Heroku:

![Hello Phoenix on Heroku](/images/2015/07/hello-phoenix-heroku.png)

### Further Configuration

However at some point you *are* going to need a database, so here is a bit of info on setting environment variables and exporting them in Heroku:

<pre>
$ heroku config:set SECRET_KEY_BASE=[long.string.of.chars]
$ heroku config:set DATABASE_USERNAME=[your.database.username]
$ heroku config:set DATABASE_PASSWORD=[your.database.password]
</pre>

See <https://devcenter.heroku.com/articles/config-vars> for more information on configuration variables in Heroku.

Edit (create if necessary) either `elixir_buildpack.config` or `phoenix_static_buildpack.config` in the root of your app and specify the environment variables to be exported.

Note that a line with a matching key will *override* the one from the buildpack, so make sure to include any existing values, such as DATABASE_URL.  You can see the default config files for the two buildpacks at

* <https://github.com/HashNuke/heroku-buildpack-elixir/blob/master/elixir_buildpack.config> and
* <https://github.com/gjaldon/heroku-buildpack-phoenix-static/blob/master/phoenix_static_buildpack.config>

My elixir_buildpack.config file now contains:
<pre>
config_vars_to_export=(DATABASE_URL SECRET_KEY_BASE DATABASE_USERNAME DATABASE_PASSWORD)
</pre>

This doesn't actually add a database to the Heroku environment, but we'll leave that for a future (or someone else's) post.

Note: secret_key_base is not database related, it's for cookie session storage. See the [Sessions][sessions] Guide for more info.

Thanks to ericmj, HashNuke, and chrismccord in #elixir-lang on freenode as well as gjaldon for the buildpack.

The code for this example is available at <https://github.com/wsmoak/hello_phoenix_heroku>

### References
* [Elixir][elixir]
* [Phoenix Framework][phoenix]
* [Heroku Buildpacks][buildpacks]
* [Elixir Buildpack][elixir-buildpack]
* [Phoenix Static Buildpack][phoenix-static-buildpack]
* [Heroku Using Multiple Buildpacks][using-multiple-buildpacks]
* <http://learnelixir.com/blog/2014/10/15/deploy-phonenix-application-to-heroku-server/>
* <http://maxwellholder.com/blog/build-a-blog-with-phoenix-and-ember>

[elixir]: http://elixir-lang.org
[phoenix]: http://www.phoenixframework.org
[buildpacks]: https://devcenter.heroku.com/articles/buildpacks
[using-multiple-buildpacks]: https://devcenter.heroku.com/articles/using-multiple-buildpacks-for-an-app
[phoenix-static-buildpack]: https://github.com/gjaldon/heroku-buildpack-phoenix-static
[elixir-buildpack]: https://github.com/HashNuke/heroku-buildpack-elixir
[sessions]: http://www.phoenixframework.org/v0.14.0/docs/sessions
