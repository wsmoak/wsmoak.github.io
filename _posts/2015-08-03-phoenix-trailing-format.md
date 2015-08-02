---
layout: post
title:  "Phoenix and the Trailing Format Plug"
date:   2015-08-03 08:50:00
tags: elixir phoenix
---

Many frameworks use the `Accept` [header][rfc-2616-14] to determine what type of content to send. For API's you're often expected to set a header such as `Accept:application/json` to indicate that you want a response in JSON format.  But what if you're re-writing an API where clients expect to specify the format as an 'extension' such as http://example.com/api/tasks/1.json ?

Let's set up a simple example and see what happens in Phoenix.

### Generate Phoenix App and API

Step 1: Generate the usual Phoenix example app:

{% highlight bash %}
$ mix phoenix.new my_app_802337
Fetch and install dependencies? [Yn] Y
$ cd my_app_802306
$ git init && git add . && git commit -m "Initial commit of generated Phoenix app"
$ curl http://[your-id].mit-license.org > LICENSE
# See https://github.com/remy/mit-license for more info
$ git add LICENSE && git commit -m "Add MIT License"
{% endhighlight %}

Step 2: Generate a simple JSON API for some tasks:

{% highlight bash %}
$ mix phoenix.gen.json Task tasks title:string due_at:datetime
$ git add . && git commit -m "Add generated controller and model for tasks json api"
{% endhighlight %}

Step 3: Edit web/router.ex

In this case we need to un-comment the api scope and add an extra line:

{% highlight diff %}
@@ -18,8 +18,9 @@ defmodule MyApp_802306.Router do
     get "/", PageController, :index
   end

-  # Other scopes may use custom stacks.
-  # scope "/api", MyApp_802306 do
-  #   pipe_through :api
-  # end
+  scope "/api", MyApp_802306 do
+    pipe_through :api
+
+    resources "/tasks", TaskController
+  end
 end
{% endhighlight %}

{% highlight bash %}
$ git add . && git commit -m "Add tasks resources to api scope"
{% endhighlight %}

Step 4: Create and Migrate the database

{% highlight bash %}
$ mix ecto.create && mix ecto.migrate
{% endhighlight %}

### Add a Task

Let's start the server inside `iex` so we can add a task directly to the database as described in [Ecto Models][ecto-models].

{% highlight bash %}
$ iex -S mix phoenix.server
{% endhighlight %}

At the iex prompt, we'll use `alias` to shorten the commands we must type, and then create a changeset and insert it in the repo.

{% highlight elixir %}
iex> alias MyApp_802337.Task

iex> changeset = Task.changeset(%Task{}, %{title: "Very Important Task", due_at: {%raw%}{{2015,8,5},{12,0,0,0}}{%endraw%} })
%Ecto.Changeset{action: nil,
 changes: %{due_at: #Ecto.DateTime<2015-08-05T12:00:00Z>,
   title: "Very Important Task"}, errors: [], filters: %{},
 model: %MyApp_802337.Task{__meta__: %Ecto.Schema.Metadata{source: {nil,
    "tasks"}, state: :built}, due_at: nil, id: nil, inserted_at: nil,
  title: nil, updated_at: nil}, optional: [],
 params: %{"due_at" => {%raw%}{{2015, 8, 5}, {12, 0, 0, 0}}{%endraw%},
   "title" => "Very Important Task"}, repo: nil, required: [:title, :due_at],
 types: %{due_at: Ecto.DateTime, id: :id, inserted_at: Ecto.DateTime,
   title: :string, updated_at: Ecto.DateTime}, valid?: true, validations: []}

iex> changeset.valid?
true

> alias MyApp_802337.Repo

> Repo.insert!(changeset)
[debug] BEGIN [] OK query=77.6ms queue=4.0ms
[debug] INSERT INTO "tasks" ("due_at", "inserted_at", "title", "updated_at") VALUES ($1, $2, $3, $4) RETURNING "id" [{%raw%}{{2015, 8, 5}, {12, 0, 0, 0}}{%endraw%}, {%raw%}{{2015, 8, 2}, {19, 46, 36, 0}}{%endraw%}, "Very Important Task", {%raw%}{{2015, 8, 2}, {19, 46, 36, 0}}{%endraw%}] OK query=1.7ms
[debug] COMMIT [] OK query=0.8ms
%MyApp_802337.Task{__meta__: %Ecto.Schema.Metadata{source: {nil, "tasks"},
  state: :loaded}, due_at: #Ecto.DateTime<2015-08-05T12:00:00Z>, id: 1,
 inserted_at: #Ecto.DateTime<2015-08-02T19:46:36Z>,
 title: "Very Important Task", updated_at: #Ecto.DateTime<2015-08-02T19:46:36Z>}
{% endhighlight %}

Now we have a task in the database, and we can see it has the `id` of `1`.

### Examine Routes

In a separate console window (be sure to leave the app running, we'll need it in a moment,) we can see what routes are available:

{% highlight bash %}
$ mix phoenix.routes
page_path  GET     /                    MyApp_802306.PageController :index
task_path  GET     /api/tasks           MyApp_802306.TaskController :index
task_path  GET     /api/tasks/:id/edit  MyApp_802306.TaskController :edit
task_path  GET     /api/tasks/new       MyApp_802306.TaskController :new
task_path  GET     /api/tasks/:id       MyApp_802306.TaskController :show
task_path  POST    /api/tasks           MyApp_802306.TaskController :create
task_path  PATCH   /api/tasks/:id       MyApp_802306.TaskController :update
           PUT     /api/tasks/:id       MyApp_802306.TaskController :update
task_path  DELETE  /api/tasks/:id       MyApp_802306.TaskController :delete
{% endhighlight %}

### Use API

That means we should be able to GET <http://localhost:4000/api/tasks/1>, right?  Let's try it, either with `curl` or in a browser:

{% highlight bash %}
$ curl http://localhost:4000/api/tasks/1
{"data":{"id":1}}
{% endhighlight %}

Hmm... that's not what I expected.  I should see my Very Important Task and its due date!

I puzzled over this for a while, and finally figured out that while the fields you specify in `mix phoenix.gen.json` get added to the model, they do NOT get added to the view.

To fix this, we need to add those fields to the view in `web/views/task_view.ex`

{% highlight diff %}
@@ -10,6 +10,6 @@ defmodule MyApp_802337.TaskView do
   end

   def render("task.json", %{task: task}) do
-    %{id: task.id}
+    %{id: task.id, title: task.title, due_at: task.due_at}
   end
 end
 {% endhighlight %}

{% highlight bash %}
$ git add . && git commit -m "Add fields to the json view."
{% endhighlight %}

Now let's try that again.  Either with curl or in a browser, request <http://localhost:4000/api/tasks/1>

{% highlight bash %}
$ curl http://localhost:4000/api/tasks/1
{"data":{"title":"Very Important Task","id":1,"due_at":"2015-08-05T12:00:00Z"}}
{% endhighlight %}

That's better!

### Legacy Clients

But what about those legacy clients who insist on appending the format as an extension?  If you try...

{% highlight bash %}
$ curl http://localhost:4000/api/tasks/1.json
{% endhighlight %}

... you get a bunch of html -- a text version of the lovely purple error page I'm sure you've seen before.  The error is:

{% highlight bash %}
Ecto.CastError at GET /api/tasks/1.json
deps/ecto/lib/ecto/repo/queryable.ex:178: value `"1.json"` in `where` cannot be cast to type :id in query:
{% endhighlight %}

The error is coming from line 27 in `task_controller.ex`:

{% highlight elixir %}
27    task = Repo.get!(Task, id)
{% endhighlight %}

It's trying to use "1.json" as the 'id' to find a record in the database, which is causing an error.

This was the topic of a [question on phoenix-talk][qpt] the other morning that I started to research, but didn't have time to finish.  I had gotten as far as:

> By adding a route and inspecting `conn` in the controller (just playing with the generated Phoenix app) I can see that given a simple GET with
>
>  `curl http://localhost:4000/api/v1.0/resource/1.json`
>
> there's a
>
>   `path_info: ["api", "v1.0", "resource", "1.json"]`
>
> that you might be able to use.  And then instead of the `:accepts` plug, which seems to be working off the Accepts header, you might have an `:extension` plug of your own that figures out what they're requesting.

Shortly thereafter, Chris McCord said:

> What you are after is the [trailing_format_plug][tfp]

### Trailing Format Plug

So! As usual, someone else has already solved the problem I have. Let's have a look at this [trailing_format_plug][tfp] and see how to use it.

A quick look at the README shows it is MIT licensed, so we're good there, however the instructions are somewhat sparse.  We're meant to add the trailing_format_plug dependency to mix.exs, which is easy enough:

{% highlight diff %}
@@ -34,6 +34,7 @@ defmodule MyApp_802337.Mixfile do
      {:postgrex, ">= 0.0.0"},
      {:phoenix_html, "~> 1.4"},
      {:phoenix_live_reload, "~> 0.5", only: :dev},
-     {:cowboy, "~> 1.0"}]
+     {:cowboy, "~> 1.0"},
+     {:trailing_format_plug, "~> 0.0.4"}]
   end
 end
{% endhighlight %}

Since we've modified the dependencies, let's make sure everything is present locally, and then commit the changes to mix.exs and mix.lock:

{% highlight bash %}
$ mix deps.get
$ git add . && git commit -m "Add trailing_format_plug dependency"
{% endhighlight %}

Then, since we are using Phoenix, the docs say "Add the plug to the :before pipeline in your router.ex".

Well, we don't _have_ a :before pipeline, and adding one didn't seem to work.  (I later found out it's something that has been removed from the framework in favor of endpoints.)

After trying several things I [asked for help in #elixir-lang][irc] on Freenode, and it turns out that the usage instructions are not correct for the lastest version of Phoenix. For the plug to work as-is, it must be placed in the Endpoint.

Let's make this change to `lib\my_app_802337\endpoint.ex`

{% highlight diff %}
@@ -35,5 +35,7 @@ defmodule MyApp_802337.Endpoint do
     key: "_my_app_802337_key",
     signing_salt: "SWCT6HSq"

+  plug TrailingFormatPlug
+
   plug MyApp_802337.Router
 end
{% endhighlight%}

Now we're back in business, and those legacy clients can make their requests with a trailing format 'extension':

{% highlight bash %}
$ curl http://localhost:4000/api/tasks/1.json
{"data":{"title":"Very Important Task","id":1,"due_at":"2015-08-05T12:00:00Z"}}
{% endhighlight %}

*However* this has broken other routes -- you can no longer visit http://localhost:4000 for example.  As Chris explained (and patched) on [irc][irc], it would be better to fix the plug so that it can work in a pipeline in router.ex and does not have to be run on every request.

### Patch

I've forked the plugin and branched to apply Chris' patch:

<https://github.com/wsmoak/trailing_format_plug/tree/phoenix_0_15_update>

To use this version in our example app, we can update mix.exs to point at that branch on GitHub:

{% highlight diff %}
@@ -35,6 +35,6 @@ defmodule MyApp_802337.Mixfile do
      {:phoenix_html, "~> 1.4"},
      {:phoenix_live_reload, "~> 0.5", only: :dev},
      {:cowboy, "~> 1.0"},
-     {:trailing_format_plug, "~> 0.0.4"}]
+     {:trailing_format_plug, github: "wsmoak/trailing_format_plug", branch: "phoenix_0_15_update"}]
   end
 end
{% endhighlight %}

Then run `mix deps.get` and we should see it cloning the code locally:

{% highlight bash %}
$ mix deps.get
* Getting trailing_format_plug (git://github.com/wsmoak/trailing_format_plug.git)
Cloning into '/Users/wsmoak/projects/my_app_802337/deps/trailing_format_plug'...
[...]
$ git add mix.* && git commit -m "Use patched version of trailing_format_plug"
{% endhighlight %}

And then move the plug (again!) from `lib/my_app_802337/endpoint.ex` back to `web/router.ex`, this time in the `:api` pipeline:

{% highlight diff %}
@@ -35,7 +35,5 @@ defmodule MyApp_802337.Endpoint do
     key: "_my_app_802337_key",
     signing_salt: "SWCT6HSq"

-  plug TrailingFormatPlug
-
   plug MyApp_802337.Router
 end

@@ -9,6 +9,7 @@ defmodule MyApp_802337.Router do
   end

   pipeline :api do
+    plug TrailingFormatPlug
     plug :accepts, ["json"]
   end
{% endhighlight %}

{% highlight bash %}
$ git add . && git commit -m "Move (patched version of) trailing_format_plug back to the :api pipeline"
{% endhighlight %}

Now the *only* time the [TrailingFormatPlug][tfp] will be used is when a request comes in that matches the `/api` path, so it's no longer breaking the other routes.

I can see that <http://localhost:4000/api/tasks/1.json> works again, but with all our changes and without tests I'm not *really* sure.  A quick way to find out is to comment out the `plug TrailingFormatPlug` line in `web/router.ex` (add a `#` in front) and confirm that you get the error again.

Next up is to make sure the patched version of the plug works with all the routes we saw earlier with `mix phoenix.routes`, write some tests for the new behavior, and submit a PR.

### Conclusion

We've learned that while there are lots of resources available in the community, they're not all kept up to date with the latest changes in _other_ resources!  My guess is that the author of this plug is not using Phoenix, or hasn't upgraded lately, and so hasn't noticed the problem.

We've also seen how to use a patched version of a dependency by pointing at a branch on GitHub.  (If you need to work with it locally, add the dependency as `{:trailing_format_plug, path: "../path/to"}`.)

The code for this example is available at <https://github.com/wsmoak/my_app_802337/tree/20150803> and is licensed MIT.

Copyright 2015 Wendy Smoak - This article first appeared on [{{ site.url }}]({{ site.url }}) and is licensed [CC BY-NC][cc-by-nc].

### References

* <http://www.phoenixframework.org/docs/understanding-plug>
* [RFC2616 Header Field Definitions][rfc-2616-14]
* <http://learnelixir.com/blog/2014/10/08/playing-with-model-in-elixir-phoenix-console/>
* <https://robots.thoughtbot.com/testing-a-phoenix-elixir-json-api>
* [Ecto Models][ecto-models]
* [Trailing Format Plug][tfp]

[ecto-models]: http://www.phoenixframework.org/docs/ecto-models
[qpt]: https://groups.google.com/d/msg/phoenix-talk/vW8M9Nc4Uik/LR-YZHCkBgAJ
[tfp]:  https://github.com/mschae/trailing_format_plug
[irc]: http://wiki.wsmoak.net/cgi-bin/wiki.pl?TrailingFormatPlug
[rfc-2616-14]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/

