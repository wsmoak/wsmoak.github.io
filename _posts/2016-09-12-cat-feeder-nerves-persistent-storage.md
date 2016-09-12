---
layout: post
title:  "Cat Feeder: Nerves and Persistent Storage"
date:   2016-09-12 08:35:47
tags: elixir embedded robotics
---

The cat feeder has had a couple of updates recently. First, it's back to being built with Nerves, because who can come back from [ElixirConf][ec] and _not_ want to play with Nerves again?  Second, it now uses a library called Persistent Storage to keep track of interesting bits of information that I don't want to lose between software updates.  Let's see how to use Nerves and Persistent Storage together.

## Nerves

The cat feeder is back on Nerves! I had switched back to Raspbian because my wifi adapter is not supported by the stock Nerves system, and building a custom system was not a project I wanted to take on.  Both of mine are Realtek 8192cu-based and rumor has it that one might make it into the default set of supported adapters, so yay, lots of work (hopefully) avoided.

One reason for switching back to Raspbian was that without wifi, it couldn't connect to the NTP servers, and so the system couldn't magically figure out what time it is.  Every time it booted it would re-start time at the Unix epoch, midnight on Jan 1, 1970.  This is important because the cat feeder has "active hours" of 8am to 8pm, so that it can't be used in the middle of the night and wake me up.  (The stepper motor is rather loud.)

It's still wifi-less for the moment, but since it boots into an iex prompt I can now set the time with `System.cmd("date",["MMDDHHMMYYYY"])` for the current date/time in UTC.

## Persistent Storage

In preparation for adding a web interface to the cat feeder, we'll need some data to display!

Persistent Storage is a small library by Nerves core team member [Garth Hitchens][gh] who is using Elixir in navigation systems for boats.  He had a need to store configuration info and have it persist across software updates.

Once you add :persistent_storage to your project as a dependency and in the applications list, you can do things like:

{% highlight elixir %}
:ok = PersistentStorage.setup path: "/root/storage"

time = Timex.now("America/New_York")
PersistentStorage.put last_fed_at: time

time = PersistentStorage.get :last_fed_at
{% endhighlight %}

So now we're saving the time of the last feeding, and we can retrieve it to display in log messages and (eventually) a web interface.

## Firmware

The first time you burn the firmware, the plain `mix firmware.burn` is fine.  Subsequently, though, you'll need to append `--task upgrade` to avoid overwriting anything you have stored on the data partition.

If you insert the SD card in a reader and look at it with your laptop, you'll see two partitions: BOOT is the read-only partition where the software goes, and APPDATA is a writeable partition that you can use for storing data.  It is mounted as `/root` (the root user's home directory) so you can create subdirectories below that.

## Summary

Thanks to [Garth][gh] for answering questions about Persistent Storage and burning the firmware without overwriting the data!  Any mistakes are my own. :)

You can see the Nerves build configuration and Persistent Storage in use on this branch of the cat feeder code:

<https://github.com/wsmoak/cat_feeder/tree/persistent_storage>

There is a very simple example of using Persistent Data (and Timex) at <https://github.com/wsmoak/storer>.

Both projects are MIT licensed.

Next up for the cat feeder is restructuring into an umbrella project and adding a web UI to display the 'last fed time' that we're storing.  Perhaps it will also let you set the date/time so you don't have to do it at the iex console.

Copyright 2016 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

[ec]: http://elixirconf.com/
[gh]: https://twitter.com/ghitchens
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
