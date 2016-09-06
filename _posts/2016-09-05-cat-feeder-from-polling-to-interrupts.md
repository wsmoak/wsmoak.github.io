---
layout: post
title:  "Cat Feeder: from Polling to Interrupts"
date:   2016-09-05 22:03:29
tags: elixir embedded robotics
---

One of the long-standing TODOs for the cat feeder has been to switch from polling the proximity sensor to having it send an interrupt. The VCNL4010 chip has an interrupt feature, but I needed to learn a few things before I could use it.

Note: the cat feeder really does work!  Several people at ElixirConf asked me about it, thinking it was just an experiment.  While it's definitely not ready (or ever intended) for mass production, it does exactly what it's meant to, which is spread out the cat's meals into many tiny servings.

The [data sheet for the chip][vcnl4010-data-sheet] says that the interrupt pin is an open drain output and will need a pull-up resistor.  Here is the relevant part:

![VCNL4010 Application Information](/images/2016/09/VCNL4010_Application_Information.png)

To someone experienced with electronics, that probably makes perfect sense.  To me, it did not!

I was somewhat hopeful that since resistors are apparently required for SDA/SCL and since I did not add them, they must be already on the chip, and perhaps this pull-up resistor was already there as well.  It was a nice dream anyway, but it wasn't true.

With the interrupt pin connected directly to a GPIO pin on the RPi2, the pin always read 0, even though I could see values changing in the interrupt status register as I touched the proximity sensor. The chip was probably trying to pull the pin low as described, but from the other end at the Raspberry Pi GPIO pin, it seemed to already always be low.

As usual, [Frank Hunleth][fh] and [Justin Schneck][js] on the Nerves team bent over backwards to help, explaining not only how to do it but why things are the way they are, and sketching diagrams for how to wire it up.

Because it's an open drain output, you have to force it high when nothing is happening, so that when something does happen you see the transition as the chip pulls the pin low. Otherwise, I gather its value is more or less undefined and is said to "float".

The circuit is currently on a breadboard while I ponder how to make it more permanent.  The orange wire is +3v from the VCNL4010 chip, then there is a (brown-black-orange-gold) 10k Ohm resistor. The green wire is the interrupt pin from the chip, and the purple wire is connected to GPIO pin 18 on the RPi2, which you can see in the background.

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/10803470@N00/29197157570/in/dateposted-public/" title="160905_2572"><img src="https://c3.staticflickr.com/9/8488/29197157570_ca18000d6e_n.jpg" width="296" height="320" alt="160905_2572"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

It works! And the result is a much more responsive machine, since it sends the interrupt on its next read.  In the polling version, it could potentially take twice as long to 'notice' that something was near depending on how the periodic reading and my polling overlapped.

And there's less code, since I don't have to deal with the message loop of :check_it when polling. I only need to listen for the :gpio_interrupt message and then check whether the state is idle or waiting to decide whether to activate the motor or not.

Here's the new `handle_info` for the custom message sent by `elixir_ale` when it sees the GPIO pin transition from 1 to 0:

{% highlight elixir %}
  def handle_info({:gpio_interrupt, _pin, :falling}, state = %{status: :idle} ) do
    hour = Timex.DateTime.now("America/New_York").hour
    if hour in @active_hours do
      Logger.debug "FEED THE CAT!"
      # turn the motor
      pid = Process.whereis( StepperTurner )
      Process.send(pid, :bump, [])
      # wait before feeding again
      Process.send_after(ProximityChecker, :time_is_up, @wait)
      clear_interrupt_status
      {:noreply, Map.update!(state, :status, fn x -> :waiting end) }
    else
      Logger.debug "Outside of allowed hours, not feeding"
      {:noreply, state}
    end
  end
{% endhighlight %}

You can see the [differences][diff] between the latest master and the interrupt branch, or look at the [interrupt branch][ib] on its own.

Next up for the cat feeder is updating to the latest Nerves configuration, getting wifi working in the Nerves build, and re-structuring it into an umbrella project so that there is a place for a web interface. Oh, and adding some tests...

The code described here is available at <https://github.com/wsmoak/cat_feeder/tree/interrupt> and is MIT licensed.

Copyright 2016 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

[vcnl4010-data-sheet]: https://cdn-shop.adafruit.com/product-files/466/vcnl4010.pdf
[js]: https://twitter.com/mobileoverlord
[fh]: https://twitter.com/fhunleth
[diff]: https://github.com/wsmoak/cat_feeder/compare/interrupt
[ib]: https://github.com/wsmoak/cat_feeder/tree/interrupt

[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
