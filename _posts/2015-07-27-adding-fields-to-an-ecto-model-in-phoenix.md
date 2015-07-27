---
layout: post
title:  "Adding Fields to an Ecto Model in Phoenix"
date:   2015-07-27 08:57:00
tags: elixir phoenix ecto
---

I recently needed to add some fields to an Ecto model that I had generated with `mix phoenix.gen.html [...]`, and to use the new fields in a Phoenix app.  Google did not immediately serve up a simple tutorial, so here it is!

This assumes you have generated a new Phoenix app and then generated a Users model, like so:

{% highlight bash %}
$ mix phoenix.new my_app
Fetch and install dependencies? [Yn] Y
$ cd my_app
$ git init && git add . && git commit -m "Initial commit of generated Phoenix app"
$ curl http://[your-id].mit-license.org > LICENSE
$ git add LICENSE && git commit -m "Add MIT License"
$ mix phoenix.gen.html User users name:string email:string
$ git add . && git commit -m "Add generated User model"
# edit web/router.ex as instructed in output
$ git add . && git commit -m "Add users resources to browser scope"
$ mix ecto.create
$ mix ecto.migrate
$ mix phoenix.server
{% endhighlight %}

If you need more information on any of the steps, they are covered in detail in [Phoenix and Ecto from mix new to Heroku][20150712], or feel free to ask in the comments below.

Before moving on, visit <http://localhost:4000/users> and make sure you can add, edit, and delete a user.

After discovering that I needed to add some fields to the User model, I checked to see if there was something like `rails generate migration NAME [field[:type]...]`.

Not quite (yet?), but there is `mix ecto.gen.migration` which will generate a skeleton for you to fill in.  You can read about it in the [Phoenix Mix Tasks docs][phx-mix-tasks] docs, or in [the Ecto docs][ecto-gen-migration].

Note:  If you search for something related to Phoenix and end up at a URL that contains a version number such as `/v0.10.0/`, be sure to delete that bit of the URL and reload the page so that you are looking at the _latest_ version of the docs.

## Generate a Migration

Let's generate a migration to add fields to our User model.

{% highlight bash %}
$ mix ecto.gen.migration add_fields_to_users
* creating priv/repo/migrations
* creating priv/repo/migrations/20150727000247_add_fields_to_users.exs
$ git add . && git commit -m "Add generated add_fields_to_users migration"
{% endhighlight %}

If we open the `[datetime]_add_fields_to_users.exs` file it says it created, we'll see this:

{% highlight elixir %}
defmodule MyApp_726605.Repo.Migrations.AddFieldsToUsers do
  use Ecto.Migration

  def change do
  end
end
{% endhighlight %}

It's up to us to tell it what needs to be changed.  We can get a hint by looking at the other file in the migrations directory, named `[datetime]_create_user.exs`:

{% highlight elixir %}
defmodule MyApp_726605.Repo.Migrations.CreateUser do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :name, :string
      add :email, :string

      timestamps
    end

  end
end
{% endhighlight %}

We want essentially the same thing with `alter table` instead of `create`, and with our new fields and types.  Let's update the add_fields_to_users migration with this:

{% highlight diff %}
@@ -2,5 +2,11 @@ defmodule MyApp_726605.Repo.Migrations.AddFieldsToUsers do
   use Ecto.Migration

   def change do
+    alter table(:users) do
+      add :user_id, :string
+      add :access_token, :binary
+      add :access_token_expires_at, :datetime
+      add :refresh_token, :binary
+    end
   end
 end
{% endhighlight %}

{% highlight bash %}
$ git add . && git commit -m "Update add_fields_to_users migration"
{% endhighlight %}

## Run the Migration

Before we run the migration, let's have a look at the database in the Postgres `psql` console:

{% highlight sql %}
# \list
# \connect my_app_726605_dev
# \d
# \d users
                                      Table "public.users"
   Column    |            Type             |                     Modifiers
-------------+-----------------------------+----------------------------------------------------
 id          | integer                     | not null default nextval('users_id_seq'::regclass)
 name        | character varying(255)      |
 email       | character varying(255)      |
 inserted_at | timestamp without time zone | not null
 updated_at  | timestamp without time zone | not null
{% endhighlight %}

Now we can run the migration...

{% highlight bash %}
$ mix ecto.migrate

20:22:42.939 [info]  == Running MyApp_726605.Repo.Migrations.AddFieldsToUsers.change/0 forward

20:22:42.939 [info]  alter table users

20:22:42.956 [info]  == Migrated in 0.1s
{% endhighlight %}

... and have another look at the users table:

{% highlight sql %}
# \d users
                                            Table "public.users"
         Column          |            Type             |                     Modifiers
-------------------------+-----------------------------+----------------------------------------------------
 id                      | integer                     | not null default nextval('users_id_seq'::regclass)
 name                    | character varying(255)      |
 email                   | character varying(255)      |
 inserted_at             | timestamp without time zone | not null
 updated_at              | timestamp without time zone | not null
 user_id                 | character varying(255)      |
 access_token            | bytea                       |
 access_token_expires_at | timestamp without time zone |
 refresh_token           | bytea                       |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
{% endhighlight %}

What if we made a mistake? If so, we can roll back with:

{% highlight bash %}
$ mix ecto.rollback
{% endhighlight %}

If you now look at the database table, the additional fields will be gone.  (If you did the rollback, re-run the migration to put the fields back in place.)

## Update Phoenix App

Now that our new fields exist in the database, let's look at what's needed to use them in our Phoenix app.

These aren't fields that will need to be edited directly, (they will be coming from another application after the user grants us access to their data via OAuth,) but we can display the user_id on the 'show' page.

Let's edit the template at `web/templates/user/show.html.eex`:

{% highlight diff %}
@@ -12,6 +12,12 @@
     <%= @user.email %>
   </li>

+  <li>
+    <strong>User ID:</strong>
+    <%= @user.user_id %>
+  </li>
+
+
 </ul>

 <%= link "Back", to: user_path(@conn, :index) %>
{% endhighlight %}

If we add a user and then click the 'Show' link, we'll get an error.  We need to add the user_id to the model. While we're at it, let's add the other fields and make them all optional. In `web/models/user.ex`:

{% highlight diff %}
@@ -4,12 +4,16 @@ defmodule MyApp_726605.User do
   schema "users" do
     field :name, :string
     field :email, :string
+    field :user_id, :string
+    field :access_token, :binary
+    field :access_token_expires_at, Ecto.DateTime
+    field :refresh_token, :binary

     timestamps
   end

-  @required_fields ~w(name email)
-  @optional_fields ~w()
+ @required_fields ~w()
+ @optional_fields ~w(name email user_id access_token access_token_expires_at refresh_token)
 {% endhighlight %}

(Note that if you do not add the new fields to either `@required_fields` or `@optional_fields`, they will be ignored when you attempt to update the database.)

Now, visiting <http://localhost:4000/users>, adding a user and clicking 'Show' will work -- the <b>User ID</b> label will be displayed along with no value since the field is empty.

{% highlight bash %}
git add . && git commit -m "Add new fields to model. Make all fields optional. Add user_id to show template."
{% endhighlight %}

Next let's simulate adding a user after they return from the OAuth flow and have granted us access to their data.  In reality there will be other libraries and a separate controller involved, but we'll just add a new function to the page controller.

In `web/page_controller.ex`, add this *above* the current `index` function:

{% highlight elixir %}
  alias MyApp_726605.User

  def index(conn, %{"test" => _}) do
    changeset = User.changeset(%User{},
      %{name: "Amy Smith",
        email: "amy@example.com",
        user_id: "ABC123",
        access_token: "fjlsfj2l34h2lh2l432lj",
        refresh_token: "l4l2k34h2l234k2h97sf",
        access_token_expires_at: {%raw%}{{2015, 12, 31}, {12, 00, 00}}{%endraw%}
      })
    Repo.insert!(changeset)

    render conn, "index.html"
  end
{% endhighlight %}

This must be added *above* the current `index` method due to the pattern match.  This function definition will match on a request that contains 'test' as a parameter. (The underscore means that we don't care what the value is, just that it is present.) If there is no 'test' parameter, then it will continue on and run the original `index` function that does not care about the parameters.

Now if you visit <http://localhost:4000/?test> you will get routed to this function instead of the default index function, and the console log will show:

{% highlight bash %}
[debug] Processing by MyApp_726605.PageController.index/2
  Parameters: %{"format" => "html", "test" => nil}
  Pipelines: [:browser]
[debug] BEGIN [] OK query=0.3ms
[debug] INSERT INTO "users" ("access_token", "access_token_expires_at", "email", "inserted_at", "name", "refresh_token", "updated_at", "user_id") VALUES ($1, $2, $3, $4, $5, $6, $7, $8) RETURNING "id" ["fjlsfj2l34h2lh2l432lj", {%raw%}{{2015, 12, 31}, {12, 0, 0, 0}}{%endraw%}, "amy@example.com", {%raw%}{{2015, 7, 27}, {11, 18, 47, 0}}{%endraw%}, "Amy Smith", "l4l2k34h2l234k2h97sf", {%raw%}{{2015, 7, 27}, {11, 18, 47, 0}}{%endraw%}, "ABC123"] OK query=0.7ms
[debug] COMMIT [] OK query=0.7ms
{% endhighlight %}

You can see that values for all of the fields are filled in, and there were no errors.  If you return to <http://localhost:4000/users> and click 'Show' for the last one in the list, you should see that user id displayed:

![Show User page with User ID](/images/2015/07/show-user-with-id.png)

{% highlight bash %}
$ git add . && git commit -m "Simulate adding a user to the database after the OAuth flow"
{% endhighlight %}

The code for this example is available at <https://github.com/wsmoak/my_app_726605/tree/20150727>

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}]({{ site.url }}) and is licensed [CC BY-NC][cc-by-nc].

## References
* [Phoenix and Ecto: From mix phoenix.new to Heroku][20150712]
* [Phoenix Mix Tasks: ecto.gen.migration][phx-mix-tasks]
* [Ecto Mix Tasks: ecto.gen.migration][ecto-gen-migration]
* <http://stackoverflow.com/questions/28506589/default-datetime-with-ecto-elixir>
* <http://hexdocs.pm/ecto/Ecto.Schema.html>

[20150712]: http://wsmoak.net/2015/07/12/phoenix-and-ecto-from-mix-new-to-heroku.html
[phx-mix-tasks]: http://www.phoenixframework.org/docs/mix-tasks#section--ecto-gen-migration-
[ecto-gen-migration]: http://hexdocs.pm/ecto/Mix.Tasks.Ecto.Gen.Migration.html
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
