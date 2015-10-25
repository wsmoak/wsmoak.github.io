---
layout: post
title:  "Connect Four in Elixir (Part 2)"
date:   2015-10-24 22:00:00
tags: elixir game
---

In [Part 1 of Connect Four in Elixir][part-1] we looked at setting up the project and printing out the 7-by-6 board grid.  Now let's look at handling the players' moves and updating the game state.

In Connect Four, two players alternate turns, and a move involves dropping a colored token into the top of the grid, where it falls down to the first empty space.  To process a move, we need to know which player, and the column they are choosing. Typically the players use red and black tokens.

### Game

In IEx, it might look like this:

{% highlight bash %}
> ConnectFour.Game.move(:red,3)
{% endhighlight %}

Let's add that function to `lib/connect_four/game.ex`:

{% highlight elixir %}
  def move(player,column) do
    case GenServer.call(@registered_name, {:move, player, column}) do
      :ok -> "Successful move for #{player} player in column #{column}"
    end
  end
{% endhighlight %}

Recall that Game is a GenServer, and here we're using GenServer.call/2 with the registered name of the Game process, and the message to send.

The `:ok` in the case statement is arbitrary -- you define what the reply from the `handle_call` function will be.

The three-element tuple `{:move, player, column}` is also arbitrary, you can structure the message however you want. It just needs to match in `handle_call`:

{% highlight elixir %}
  def handle_call({:move, player, column}, _from, state) do
    ...
  end
{% endhighlight %}

(The variable names can be different, but this will only match a tuple with the atom `:move` in the first position, and then two additional values.)

We're not doing anything with the process ID of the sending process so it is ignored by adding an underscore: `_from`.

The state is passed to `handle_call` and we can either make a change and return a different state, or just return the same state.

When that call comes in, we need to tell the Board to place the token into one of the spaces.

{% highlight elixir %}
  def handle_call({:move, player, column}, _from, state) do
    ConnectFour.Board.place_token(player, column)
    [...]
  end
{% endhighlight %}

Because this is a `call`, (and not a cast,) we *must* return something.  The allowed return values are shown in [`GenServer.handle_call/3`][handle_call] and we will be sending `{:reply, reply, new_state}`.

Let's assume everything went well and the move was accepted:

{% highlight elixir %}
  def handle_call({:move, player, column}, _from, state) do
    case ConnectFour.Board.place_token(player, column) do
      {:move_accepted} ->
        newstate = Map.put(state, :last_moved, player) # 1
        {:reply, :ok, newstate}                        # 2
    end
  end
{% endhighlight %}

`# 1`: Here we see the state held in the Game being updated.  We'll keep track of the player who moved last, so that later we can do some error handling if the same player tries to move twice in a row.

`# 2`: Here we see the `:ok` that we matched on in the `move/2` function.

This means we'll need to write a `place_token` function in our `Board` module that replies with `:move_accepted` if all goes well. For now we'll just hard-code the return value so we can see this work.

In `lib/connect_four/board.ex`:

{% highlight elixir %}
  def place_token(player,col) do
    :move_accepted
  end
{% endhighlight %}

Let's try it out in IEx:

{% highlight bash %}
$ iex -S mix

> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"
{% endhighlight %}

Go ahead and commit these changes.

{% highlight bash %}
$ git add . && git commit -m "initial code for a player's move"
{% endhighlight %}

What's happening here?  This is what the high level sequence diagram looks like:

{% plantuml %}
IEx -> Game : move(:red,3)
Game -> Board : place_token(:red, 3)
Board -> Game : :move_accepted
Game -> IEx : "Successful move for..."
{% endplantuml %}

But there's more going on -- Game is a GenServer with a client API and server callbacks.

<!-- (the following plantuml code works on http://plantuml.com/plantuml/form but not locally with the jekyll-plantuml plugin)
box "Process" #LightBlue
  participant "IEx"
        participant "Game\n(Client API)"
end box

IEx -> "Game\n(Client API)" : move(:red,3)
"Game\n(Client API)" -> "ConnectFourGame\n(GenServer)" : call( ConnectFourGame,\n{:move, :red, 3} )

box "Process" #LightBlue
  participant "ConnectFourGame\n(GenServer)"
        participant "Game\n(Server Callbacks)"
end box

"ConnectFourGame\n(GenServer)" -> "Game\n(Server Callbacks)" : handle_call( {:move, red, 3} )
"Game\n(Server Callbacks)" -> "Board" : place_token(:red, 3)

box "Process" #LightBlue
  participant "Board"
end box

"Board" -> "Game\n(Server Callbacks)" : :move_accepted
"Game\n(Server Callbacks)" -> "ConnectFourGame\n(GenServer)" : :ok
"ConnectFourGame\n(GenServer)" -> "Game\n(Client API)" : :ok
"Game\n(Client API)" -> IEx : "Successful move for..."
-->

![Game GenServer](/images/2015/10/connect-four-game-genserver-sequence.png)

(With apologies for misusing the symbols in a sequence diagram...)  The code for the Game's Client API and Server Callbacks all lives in the ConnectFour.Game module, but it gets executed in two different processes, in this case the IEx.Evaluator process and the ConnectFourGame process.

To prove it, add some code to the handle_call function in the Game module:

{% highlight diff %}
diff --git a/lib/connect_four/game.ex b/lib/connect_four/game.ex
index bd5820f..babba3a 100644
--- a/lib/connect_four/game.ex
+++ b/lib/connect_four/game.ex
@@ -33,7 +33,12 @@ defmodule ConnectFour.Game do

   # Server Callbacks

-  def handle_call({:move, player, column}, _from, state) do
+  def handle_call({:move, player, column}, from, state) do
+    IO.puts "in Game handle_call, "
+    IO.write "self is "
+    IO.inspect Kernel.self
+    IO.write "and from is "
+    IO.inspect from
     case ConnectFour.Board.place_token(player, column) do
       :move_accepted ->
         newstate = Map.put(state, :last_moved, player)
{% endhighlight %}

{% highlight bash %}
> ConnectFour.Game.move(:red,3)
in Game handle_call,
self is #PID<0.96.0>
and from is {#PID<0.140.0>, #Reference<0.0.7.226>}
"Successful move for red player in column 3"
{% endhighlight %}

If you then look in the Observer, on the Processes tab (click the Pid column to sort by Pid) you'll see that PID 96 is the ConnectFourGame process...

![PID 96](/images/2015/10/connect-four-observer-pid-96.png)

... and PID 140 is the IEx.Evaluator process.

![PID 140](/images/2015/10/connect-four-observer-pid-140.png)

(Revert these changes with `git checkout lib/connect_four/game.ex` -- another option would be to log the messages at the debug level, but I didn't find usage info for Logger at a glance.)

### Board

Now let's look at the `place_token` function and see how to modify the state of the appropriate space on the board.

The player only selects the column.  It's up to us to figure out which row the game piece will fall down to in a vertically suspended grid and determine the row number.

At first, let's just hard-code row number 1 and update the agent for row 1 in the specified column.  We still need to return `:move_accepted` as before.

In `lib/connect_four/board.ex`:

{% highlight diff %}
diff --git a/lib/connect_four/board.ex b/lib/connect_four/board.ex
index e3cb71d..3ca3103 100644
--- a/lib/connect_four/board.ex
+++ b/lib/connect_four/board.ex
@@ -67,6 +67,9 @@ defmodule ConnectFour.Board do
   end

   def place_token(player,col) do
+    agent_name(1,col)
+    |> Process.whereis
+    |> Agent.update(fn _state -> player end)
     :move_accepted
   end
{% endhighlight %}

And try it out in IEx

{% highlight bash %}
$ iex -S mix
Erlang/OTP 18 [erts-7.0.3] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiled lib/connect_four/board.ex
Generated connect_four app
Interactive Elixir (1.1.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"
iex(2)> ConnectFour.Game.print_board
.......
.......
.......
.......
.......
..R....
[:ok, :ok, :ok, :ok, :ok, :ok]
iex(3)>
{% endhighlight %}

Here you can see the 'R' indicating a red game piece in the first (bottom) row of the third column.

But of course we can't always use row 1 -- we need to calculate the lowest empty row for the specified column.  How about this?

{% highlight diff %}
diff --git a/lib/connect_four/board.ex b/lib/connect_four/board.ex
index e3cb71d..ddd600b 100644
--- a/lib/connect_four/board.ex
+++ b/lib/connect_four/board.ex
@@ -67,7 +67,34 @@ defmodule ConnectFour.Board do
   end

   def place_token(player,col) do
+    first_empty(col)
+    |> agent_name(col)
+    |> Process.whereis
+    |> Agent.update(fn _state -> player end)
     :move_accepted
   end

+  def first_empty(col) do
+    first_empty(1,col)           #1
+  end
+
+  def first_empty(row, col) do
+    if empty_space?(row,col) do
+      row
+    else
+      first_empty(row+1,col)     #2
+    end
+  end
+
+  def empty_space?(row,col) do
+    agent_name(row,col)
+    |> Process.whereis
+    |> Agent.get( &(&1) )        #3
+    |> is_empty?
+  end
+
+  def is_empty?(val) do
+    val == Empty                 #4
+  end
+
 end
{% endhighlight %}

`# 1`: While you can have optional parameters with default values, they have to go at the *end*.  Since it's the row we need to default, and everything else is (row,col) it would be too confusing to have this one function be (col, row // 1).

`# 2`: Note the recursion in the `first_empty` function -- if the space is not empty, it calls itself with the next row up.

`# 3`: `&(&1)` is function capture syntax for the identity function, equivalent to `fn x -> x end`.  There is no 'plain' `Agent.get`, you always have to provide a function that produces the value.

`# 4`: Recall that the state of each space was set to `Empty` when it was created.

And try this out:

{% highlight bash %}
$ iex -S mix
Erlang/OTP 18 [erts-7.0.3] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiled lib/connect_four/board.ex
Generated connect_four app
Interactive Elixir (1.1.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> ConnectFour.Game.move(:red,3)
"Successful move for red player in column 3"
iex(2)> ConnectFour.Game.move(:black,3)
"Successful move for black player in column 3"
iex(3)> ConnectFour.Game.move(:red,5)
"Successful move for red player in column 5"
iex(4)> ConnectFour.Game.print_board
.......
.......
.......
.......
..B....
..R.R..
[:ok, :ok, :ok, :ok, :ok, :ok]
iex(5)>
{% endhighlight %}

This shows that the second move in a column correctly detects that the first (bottom) row is filled and places the game piece in the second row up.

And commit the changes:

{% highlight bash %}
$ git add . && git commit -m "Update the space in the first empty row when a player moves in a column"
{% endhighlight %}

### Conclusion

We've seen how a GenServer works behind the scenes, and how to find the first empty row in a column, and how to update the Agent that holds the state of each space on the board.

Next time we'll add some error handling. What if the players don't alternate turns?  What if the column they select is already full? (You can try it by making seven moves in the same column.)  We might also try to get rid of the conditional logic in `first_empty`.  And then we'll need to detect a "win" and stop the game.

The code for this example is available at <https://github.com/wsmoak/connect_four/tree/20151024> and is Apache licensed.

Copyright 2015 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

### References

* [Part 1 of Connect Four in Elixir][part-1]
* [GenServer.handle_call/3][handle_call]

[part-1]: http://wsmoak.net/2015/11/21/connect-four-elixir-part-1.html
[handle_call]: http://elixir-lang.org/docs/master/elixir/GenServer#c:handle_call/3
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
