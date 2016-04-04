---
layout: post
title:  "Seeding the Database from a CSV file in Phoenix"
date:   2016-02-16 08:27:00
tags: elixir phoenix ecto
---

When you generate a Phoenix project, there is a `priv/repo/seeds.exs` file that you can use to seed the database. Let's see how to seed it from a CSV file.

We'll be working with the `transactions.csv` file from [Mint][mint], and the project name is `Minty`. This assumes `phoenix.gen.html` has been used to create a model and the resources line has been added to `web/router.ex`.

{% highlight bash %}
$ mix phoenix.gen.html Transaction transactions date:string description:string original_description:string amount:string transaction_type:string category:string account_name:string labels:string notes:string
{% endhighlight %}

First, consult the [documentation for the CSV library][csv-docs] and add the dependency to mix.exs:

{% highlight elixir %}
  defp deps do
    [ ...
     {:csv, "~> 1.2.3"}
    ]
  end
{% endhighlight %}

Then in `priv/repo/seeds.exs` we'll use the CSV library to generate a Map for each row of the CSV file with keys that EXACTLY match our Ecto model.

{% highlight elixir %}
# Script for populating the database. You can run it as:
#
#     mix run priv/repo/seeds.exs
#
# Inside the script, you can read and write to any of your
# repositories directly:
#
#     Minty.Repo.insert!(%Minty.SomeModel{})
#
# We recommend using the bang functions (`insert!`, `update!`
# and so on) as they will fail if something goes wrong.

alias Minty.Transaction
alias Minty.Repo

defmodule Minty.Seeds do

  def store_it(row) do
    changeset = Transaction.changeset(%Transaction{}, row)
    Repo.insert!(changeset)
  end

end

File.stream!("/Users/wsmoak/Downloads/transactions.csv")
  |> Stream.drop(1)
  |> CSV.decode(headers: [:date, :description, :original_description, :amount, :transaction_type, :category, :account_name, :labels, :notes])
  |> Enum.each(&Minty.Seeds.store_it/1)
{% endhighlight %}

Note that we are [dropping the first line of the CSV][csv-issue] file. It contains headers, but we need to define different ones that match the Ecto model to make insertion easier.

As the comments indicate, you can run this with `mix run priv/repo/seeds.exs`.

This works, but it assumes everything is a string.  To match up the types we need to do a bit more work.

### Date

The date column in the Mint.com CSV file is unfortunately in M[M]/DD/YYYY format.  Additionally, the days are zero-padded, but not the months. (?!!)  Ecto.Date can work with several formats, but that is not one of them.  Let's flip this around to be YYYY-MM-DD instead.

{% highlight elixir %}
  def fix_date(%{date: <<m0,"/",d1,d0,"/",y3,y2,y1,y0>>} = row) do
    date = <<y3,y2,y1,y0,"-","0",m0,"-",d1,d0>>
    Map.update!(row,:date,fn _ -> date end)
  end

  def fix_date(%{date: <<m1,m0,"/",d1,d0,"/",y3,y2,y1,y0>>} = row) do
    date = <<y3,y2,y1,y0,"-",m1,m0,"-",d1,d0>>
    Map.update!(row,:date,fn _ -> date end)
  end
{% endhighlight %}

This pattern matches on the one- and two-digit months and then rearranges the bits with `-`'s in between, also zero-padding the one-digit months.

An earlier version of this used String.split and String.rjust, but I think I like it better with the pattern matching.

Now the migration can have `add :date, :date` and the model can have `field :date, Ecto.Date`.

### Amount

All of the amounts in the CSV are positive numbers, and there is a separate transaction_type field that contains either "debit" or "credit".  Mint doesn't really *do* double-entry bookeeping so I have no idea why they use this concept.  Let's keep income as positive numbers and change all the expenses ("debits") to negative.

We're still working with Strings at this point, so all we need to do is prepend a dash to indicate that it's negative.

{% highlight elixir %}
  def fix_amount(%{transaction_type: "debit"} = row) do
    Map.update!(row,:amount,&("-"<>&1))
  end

  def fix_amount(%{transaction_type: "credit"} = row) do
    row
  end
{% endhighlight %}

This pattern matches on a Map containing a key of `transaction_type`, and either updates the value under the `:amount` key, (or not.)

Now the amount field can be :float in both the migration and the model.

## Re-load

Drop the database, re-create it, and run the (edited) migration again:

{% highlight bash %}
$ mix ecto.drop
$ mix ecto.create
$ mix ecto.migrate
$ mix run "priv/repo/seeds.exs"
{% endhighlight %}

Now you should have data with a proper date field and positive/negative numbers so that math will work.

## Query

Now that the data is loaded, let's try some queries.  I have this in `priv/repo/queries.exs`

{% highlight elixir %}
defmodule Minty.Queries do
  import Ecto.Query

  alias Minty.Repo
  alias Minty.Transaction

  def bignum do
    Repo.all(
      from txn in Transaction,
      where: txn.amount > 1000
    )
  end

  def summary do
    Repo.all(
      from txn in Transaction,
      group_by: txn.category,
      select: [txn.category, sum(txn.amount)]
    )
  end
end
{% endhighlight %}

Play with it in `iex -S mix`:

{% highlight bash %}
> c("priv/repo/queries.exs")
# large transactions
> Minty.Queries.bignum

# summarized by category, sorted by category name
> Minty.Queries.summary |> Enum.sort |> Enum.each(&(IO.inspect &1))

# summarized by category, sorted by amount
> Minty.Queries.summary |> Enum.sort(&(Enum.at(&1,1)>Enum.at(&2,1))) |> Enum.each(&(IO.puts("#{Enum.at(&1,0)} #{Float.to_string(Enum.at(&1,1),[decimals: 2])}")))
{% endhighlight %}

## Conclusion

We've seen how to seed the database from a CSV file in a Phoenix project including date and float type fields, and some simple queries.

The code for this example is available at <https://github.com/wsmoak/minty> and is MIT licensed.

Copyright 2016 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

## References

* [Elixir CSV][csv-docs]
* [Mint][mint]
* [Populating Database Tables from a CSV in Elixir][prior-art]

[mint]: https://www.mint.com/
[csv-docs]: https://github.com/beatrichartz/csv
[csv-issue]: https://github.com/beatrichartz/csv/issues/27
[prior-art]: http://www.rymai.me/2015/12/08/populating-database-tables-from-a-csv-in-elixir/
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
