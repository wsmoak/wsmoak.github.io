---
layout: post
title:  "Connect Four in Elixir (Part 3)"
date:   2015-10-26 08:55:00
tags: elixir game
---

In [Part 2 of Connect Four in Elixir][part-2] we updated the board for the players' moves. Now let's see how to prevent errors like the same player moving twice in a row, or attempting to move in a column that is full.  After that, we'll consider how to detect a win.

### Column is Full

To check whether the column is full, we only need to look at the space in the 'last' (topmost) row and see if it is not empty.

In `lib/connect_four/board.ex:`

{% highlight elixir %}
  def place_token(player,col) do
    if is_full?(col) do
      :column_full
    else
      first_empty(col)
      |> agent_name(col)
      |> Process.whereis
      |> Agent.update(fn _state -> player end)
      :move_accepted
    end
  end

  def is_full?(col) do
    agent_name(@last_row,col)
    |> Process.whereis
    |> Agent.get( &(&1) )
    |> (&(&1 != Empty)).()         # See Appendix A below
  end
  {% endhighlight %}

And handle the new possibility in game.ex:

{% highlight diff %}
diff --git a/lib/connect_four/game.ex b/lib/connect_four/game.ex
index bd5820f..281cef5 100644
--- a/lib/connect_four/game.ex
+++ b/lib/connect_four/game.ex
@@ -28,6 +28,7 @@ defmodule ConnectFour.Game do
   def move(player,column) do
     case GenServer.call(@registered_name, {:move, player, column}) do
       :ok -> "Successful move for #{player} player in column #{column}"
+      :full -> "Column #{column} is full.  Please choose another."
     end
   end

@@ -38,6 +39,8 @@ defmodule ConnectFour.Game do
       :move_accepted ->
         newstate = Map.put(state, :last_moved, player)
         {:reply, :ok, newstate}
+      :column_full ->
+        {:reply, :full, state}
     end
   end
{% endhighlight %}

And try it out in IEx:

{% highlight bash%}
$ iex -S mix

> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"
> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

[...repeat...]

> ConnectFour.Game.move(:red,3)
"Column 3 is full.  Please choose another."

> ConnectFour.Game.print_board
..R....
..R....
..R....
..R....
..R....
..R....
[:ok, :ok, :ok, :ok, :ok, :ok]
>
{% endhighlight %}

And commit these changes.

{% highlight bash %}
$ git add . && git commit -m "Detect when a column is full"
{% endhighlight %}

Now we can detect when the column is full, but we're allowing the same player to move over and over.  They need to alternate.

# Alternate Player Moves

Recall that we're updating the state in the Game GenServer with the player who moved last.  In a two-player game, it's sufficient to check that that player who moved last isn't trying to move again.

(If there were more players, we might want to keep track of who is expected to move _next_ instead. Or we might want to remove the player from the incoming message altogether, and just assume that the move is intended for player who should go next.)

Initially I started trying to add another condition to the existing `handle_call` function, but then I realized... PATTERN MATCHING!  We can match on the state being passed into handle_call, like this:

{% highlight elixir %}
  def handle_call({:move, player, _column}, _from, %{last_moved: player} = state) do
    {:reply, :wrong_player, state}
  end
{% endhighlight %}

If the Game's state (recall it was initialized as an empty map and then updated for each successful move) contains a key of `:last_moved` and the value is the same as the player attempting to move now, then there is a problem.  It's not their turn; the other player needs to move first.

Note that this needs to go *above* the original handle_call, otherwise that one will always match.  (Try it below and see the compiler warning.)

We also need to handle the new case:

{% highlight diff %}
@@ -29,11 +29,16 @@ defmodule ConnectFour.Game do
     case GenServer.call(@registered_name, {:move, player, column}) do
       :ok -> "Successful move for #{player} player in column #{column}"
       :full -> "Column #{column} is full.  Please choose another."
+      :wrong_player -> "It's not your turn!"
     end
   end
{% endhighlight %}

Now in IEx we get:

{% highlight bash %}
> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

> ConnectFour.Game.move(:red,3)
"It's not your turn!"
{% endhighlight %}

There is more error handling we could do, such as restricting the players to a defined list of `:red` and `:black`, but we'll commit this change and move on to detecting a win.

{% highlight bash %}
$ git add . && git commit -m "Prevent same player from moving again"
{% endhighlight %}

### Detect a Win

A winning move is one that connects four pieces of the same color in a vertical, horizontal or diagonal line.  Starting from the most recently updated space, we need to look at most three spaces in all directions in order to check all the possible winning patterns.

We'll start by detecting a vertical win in the current column, because that's the easiest.  There can't be any pieces *above* the last one, so we only need to look down and see if there are three more of the same color.

Here's what placing a token looks like now in `lib/connect_four/game.ex/`:

{% highlight elixir %}
  def place_token(player,col) do
    if is_full?(col) do
      :column_full
    else
      row = first_empty(col)
      place_token(player,row,col)
    end
  end

  def place_token(player,row,col) do
    agent_name(row,col)
    |> Process.whereis
    |> Agent.update(fn _state -> player end)

    if winner?(row,col) do
      :winner
    else
      :move_accepted
    end
  end

  def winner?(row,col) do
    agent_name(row,col)
    |> Process.whereis
    |> Agent.get( &(&1) )
    |> column_winner?(row,col,1)
  end

  def column_winner?(player,row,col,4) do                 #2
    true
  end

  def column_winner?(player,row,col,count) when row > 1 and row <= @last_row do
    neighbor = agent_name(row-1,col)
    |> Process.whereis
    |> Agent.get( &(&1) )

    if player == neighbor do
      column_winner?(player,row-1,col,count+1)
    else
      false                                                #3
    end
  end

  def column_winner?(player,row,col,count) when row == 1 do  #1
    false
  end
{% endhighlight %}

`#1`: there is no neighbor *below* row 1, so if we've gotten here without finding four adjacent pieces, it's not going to happen.

`#2`: the base case -- we've found four adjacent pieces and this player wins

`#3`: the neighbor is not the same, and we haven't yet found 4, so it's not a win.

I still don't like all the conditional logic in this.  If you see a better way to do it, add a comment!

Let's see this work in IEx:

{% highlight bash %}
$ iex -S mix

iex(1)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

iex(2)> ConnectFour.Game.move(:black,3)
"Successful move for black player in column 3"

iex(3)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

iex(4)> ConnectFour.Game.move(:black,4)
"Successful move for black player in column 4"

iex(5)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

iex(6)> ConnectFour.Game.move(:black,4)
"Successful move for black player in column 4"

iex(7)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"

iex(8)> ConnectFour.Game.move(:black,4)
"Successful move for black player in column 4"

iex(9)> ConnectFour.Game.move(:red,3)
"Player red wins!"

iex(10)> ConnectFour.Game.print_board
..R....
..R....
..R....
..RB...
..BB...
..RB...
[:ok, :ok, :ok, :ok, :ok, :ok]
iex(11)>
{% endhighlight %}

Detecting a win on the row is more complicated because you have to look both left and right along the row.  This is left as an exercise for the reader. :)

One final commit:

{% highlight bash %}
$ git add . && git commit -m "Detect a winner in the column of the last move"
{% endhighlight %}

### Conclusion

We've added some error handling and seen how to detect the simplest winning pattern, a vertical win in a column.

The code for this example is available at <https://github.com/wsmoak/connect_four/tree/20151026> and is Apache licensed.

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

### References

* [Part 2 of Connect Four in Elixir][part-2]
* [How to pass an anonymous function to the pipe in Elixir][pipe-anon-fn]

### Appendix A: Anonymous Functions in the Pipeline

In the `is_full?` function I originally had `|> &(&1 != Empty)` as the last line of pipeline.  This is the shortcut function capture syntax for `fn x -> x != Empty end`.  But if you try to use this in a pipeline, you get:

{% highlight bash %}
== Compilation error on file lib/connect_four/board.ex ==
** (ArgumentError) cannot pipe Agent.get(Process.whereis(agent_name(
   @last_row, col)), &&1) into &&1 != Empty.(), can only pipe into
   local calls foo(), remote calls Foo.bar() or anonymous functions
   calls foo.()
    (elixir) lib/macro.ex:113: Macro.bad_pipe/2
    (stdlib) lists.erl:1262: :lists.foldl/3
    (elixir) expanding macro: Kernel.|>/2
    lib/connect_four/board.ex:85: ConnectFour.Board.is_full?/1
{% endhighlight %}

Misreading the error, I tried `|> &(&1 != Empty).()` but that didn't make it happy either.  [Stack Overflow to the Rescue!][pipe-anon-fn] and the answer is `|> (&(&1 != Empty)).()`.

I opened [PR 3916][pr-3916] to see about improving the error message -- José replied that this syntax shouldn't be encouraged, and a private method ought to be used instead.  So, don't do this! :)

[part-2]: http://wsmoak.net/2015/10/24/connect-four-elixir-part-2.html
[pipe-anon-fn]: http://stackoverflow.com/questions/24593967/how-to-pass-an-anonymous-function-to-the-pipe-in-elixir
[pr-3916]: https://github.com/elixir-lang/elixir/pull/3916
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
