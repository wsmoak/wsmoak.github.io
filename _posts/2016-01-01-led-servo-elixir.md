---
layout: post
title:  "Blinking LEDs and Spinning Servos with Elixir"
date:   2016-01-01 08:00:00
tags: elixir robotics
---

I've been looking for an automated cat feeder, and haven't been very happy with the options.  I need it to dispense a very small amount, and only when the cat is present, to avoid it piling up and letting her eat too much at once.  (Bad things happen then.)  The closest thing I found was the [Wireless Whiskers][ww] feeder, but I just wasn't inspired to spend $150 and then fuss with programming it on a tiny screen.  And then [@bitsandhops][bh] posted this:

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">At some point this cat is going to starve - “Automated cat feeder powered by Node.js” <a href="https://t.co/cMM4UzrMT4">https://t.co/cMM4UzrMT4</a></p>&mdash; Richard Bishop (@bitsandhops) <a href="https://twitter.com/bitsandhops/status/671510384520531968">December 1, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Following the link from the [Robokitty source code][robokitty-src] to the [servo][servo] she used, I discovered [Adafruit][adafruit] and promptly sent them all my money.

I knew I wanted to use Elixir (of course!) so I went with a Raspberry Pi starter kit, and then discovered that the RPi doesn't really *do* servos, so I added the Servo HAT.  And a proximity sensor, etc., etc.  Once it works I'll do a post with all the details and a parts list.

## LED

Not having done anything with hardware since college (that course where I spent hours and hours in a windowless basement sticking Motorola chips into breadboards...) I started with the "Hello World" of the embedded space: blinking an LED.

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/10803470@N00/23954136901" title="151225_1686"><img src="https://farm6.staticflickr.com/5672/23954136901_b3958925d1_n.jpg" width="320" height="320" alt="151225_1686"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

(Actually I first wired it directly to power, which is what you see above.  Later I moved that red wire over to GPIO pin #4 as described [here][single-led].)

After reading a bunch of blog posts and with [@fhunleth][fh]'s excellent [elixir_ale][elixir-ale] library, (and his patience in answering LOTS of questions,) it turned out to be as simple as:

{% highlight elixir %}

defmodule BlinkyAle do

  def start do
    {ok,pid} = Gpio.start_link(4,:output)
    blink_forever(pid)
  end

  def blink_forever(pid) do
    Gpio.write(pid,1)
    :timer.sleep 1000
    Gpio.write(pid,0)
    :timer.sleep 1000
    blink_forever(pid)
  end

end
{% endhighlight %}

The elixir_ale library even exports the pin you want to use to make it available, which is described in [this blog post][export-pin].  Under the covers this uses `sysfs` which entails writing a value to a file, after which Linux does the right thing.

## Servo

Having conquered blinking an LED, I turned to the servo.  This meant soldering the header and some pins onto the HAT.  I watched:

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/10803470@N00/24036679745/in/dateposted-public/" title="151226_1701"><img src="https://farm2.staticflickr.com/1512/24036679745_01a7795471_n.jpg" width="320" height="320" alt="151226_1701"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

I spent a frustrating afternoon trying to reverse engineer [the Python example code][adafruit-python] that Adafruit provided, and (after reading more blogs and asking more questions) I figured out that I was *supposed* to have a Datasheet to work from.

Some searching and one [Support Forum post][support] later, I confirmed that the [PCA9685 Datasheet][pca9685] is what I needed, and things got MUCH easier.

With it all plugged in and working:

<iframe src="https://player.vimeo.com/video/150140381" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="https://vimeo.com/150140381">ElixirSpinny</a> from <a href="https://vimeo.com/user1032254">Wendy Smoak</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

The code is here: [Spinny][spinny].

## Proximity

For our last trick, it's back to the LED, this time with the VCNL4010 proximity sensor wired up.

<iframe src="https://player.vimeo.com/video/150217372" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/150217372">ElixirProximityBlinky</a> from <a href="https://vimeo.com/user1032254">Wendy Smoak</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

Here is the [code that makes it work][blinky].

This was a matter of connecting the right pins from the sensor to the RPi, as shown in in the [Adafruit example project][prox-lights] and the [Datasheet][vcnl4010]. [@fhunleth][fh] explained that the SDA and SCL pins are what put something "on the i2c bus" so it can be addressed and written to / read from.

## Next Up

Next up I'll need to get both the servo and the proximity sensor attached at the same time, so that instead of turning on a light when something is near, I can spin the servo.  As soon as the next box from Adafruit arrives, I'll have the additional parts I need to make that happen.

### References

* [Robokitty by Rachel White][robokitty]
* [Robokitty Arduino/Node project details][robokitty-src]
* [A single LED][single-led]
* [Elixir on the Raspberry Pi][export-pin]
* [More notes and links on my wiki][wiki]

[ww]: http://www.wirelesswhiskers.com/ec/index.php
[bh]: https://twitter.com/bitsandhops
[fh]: https://twitter.com/fhunleth
[robokitty]: http://imcool.online/robokitty/
[robokitty-src]: https://github.com/rachelnicole/robokitty
[servo]: https://www.adafruit.com/products/154
[adafruit]: https://www.adafruit.com
[blinky-ale]: https://gist.github.com/wsmoak/c1fd4e95578933e23388#file-blinky-ex
[elixir-ale]: https://github.com/fhunleth/elixir_ale
[support]: https://forums.adafruit.com/viewtopic.php?f=50&t=86471
[pca9685]: https://www.adafruit.com/datasheets/PCA9685.pdf
[spinny]: https://gist.github.com/wsmoak/6da34768fdfd6d4c2d1d
[blinky]: https://gist.github.com/wsmoak/c1fd4e95578933e23388
[export-pin]: http://wtfleming.github.io/2015/12/10/embedded-elixir-raspberry-pi/
[adafruit-python]: https://github.com/adafruit/Adafruit-Raspberry-Pi-Python-Code/blob/master/Adafruit_PWM_Servo_Driver/Adafruit_PWM_Servo_Driver.py
[wiki]: http://wiki.wsmoak.net/cgi-bin/wiki.pl?RaspberryPi
[prox-lights]: https://learn.adafruit.com/festive-feather-holiday-lights
[vcnl4010]: https://www.adafruit.com/images/product-files/466/vcnl4010.pdf
[single-led]: https://projects.drogon.net/raspberry-pi/gpio-examples/tux-crossing/gpio-examples-1-a-single-led/
