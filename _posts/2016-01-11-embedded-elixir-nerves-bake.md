---
layout: post
title:  "Embedded Elixir with Nerves and Bake"
date:   2016-01-11 10:00:00
tags: elixir embedded nerves bake
---

Now that the [code][cf] and the hardware for the automated cat feeder are coming together, let's see what it takes to build reproducible releases for installation on the Raspberry Pi.

Following on from the last post where I was using the proximity sensor and the servo separately, they are now both connected at once:

<iframe src="https://player.vimeo.com/video/151323242" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="https://vimeo.com/151323242">ElixirProximitySpinny</a> from <a href="https://vimeo.com/user1032254">Wendy Smoak</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

This involved soldering a few more pins into the Servo HAT and attaching the proximity sensor to them with female-to-female jumper wires.

The parts shown in the video are:

* Raspberry Pi 2 Model B
* Adafruit PWM/Servo HAT
* FiTech FS5103R continuous-rotation servo
* VCNL4010 Proximity Sensor
* 5V 2A power supplies (2)
* header pins
* female-to-female jumper wires

The other two cables are the HDMI video and USB to a powered hub which has an Ourlink wireless adapter plugged in.  Neither is required for this example.

See this [Adafruit Wishlist][wish] for the parts list.  As you can see, it's over $100 already and it doesn't do anything useful yet!  This could be done with *much* cheaper components, I'm sure.

Up to this point I've been developing directly on the RPi2, either by connecting a keyboard and monitor, or by SSH'ing into it.  That's one reason the RPi2 makes a great platform for getting started, but with truly embedded applications, it isn't always possible.

In addition to remote development, I wanted a reproducible way to build releases to be installed into the device.  Enter Nerves and Bake (and exrm and buildroot and erlinit and many other things behind the scenes).

I was first introduced to [Nerves][nerves] in [Garth Hitchens][gh]' [Embedded Elixir in Action][eeia] talk at ElixirConf 2015.  If you've never heard of Nerves before, go watch that talk.  I'll wait.

Nerves and Bake are all about building firmware for embedded devices, without the pain of maintaining a virtual machine and cross-compiling for the target platform.

With a few commands we can retrieve a toolchain and system, create a firmware image, and write it to a micro-SD card that can be popped into the RPi.

Starting from the project I had developed directly on the Raspberry Pi, I [cloned the source][cf] on my Mac and followed the instructions on [www.bakeware.io][bake]:

**1:** First, install Bake.  I encourage you to grab the script and review it before just pasting in the command to download and execute it.  A quick read will reveal that it installs the `bake`, `fwup` and `squashfs` utilities.  At the moment, it's a Ruby script but it may get converted to Elixir eventually.

{% highlight bash %}
$ wget https://bakeware.herokuapp.com/bake/install
# examine install file contents, then go ahead:
$ ruby -e "$(curl -fsSL https://bakeware.herokuapp.com/bake/install)"
{% endhighlight %}

Note:  This is currently Mac only, but support for Linux and Windows are planned.

**2:** Add a Bakefile to the root of the project.  In this case we're only targeting one platform, the Raspberry Pi 2.  Specifying a `default_target` in the Bakefile allows us to avoid adding `--target rpi2` on every command.

{% highlight elixir %}
use Bake.Config

platform :nerves
default_target :rpi2

target :rpi2,
  recipe: "nerves/rpi2"
{% endhighlight %}

**3:** Download a "system" and a "toolchain".  Because we included a `default_target` in the Bakefile, we don't have to specify it here.  If you need more than one, use `--target all` with each command.

{% highlight bash %}
$ bake system get
$ bake toolchain get
{% endhighlight %}

You can find the files it downloaded in your home directory under `~/.nerves`.

**4:** Compile the firmware for your target platform, including your project code.  Again it will pick up the default target automatically.

{% highlight bash %}
$ bake firmware
{% endhighlight %}

This will create a file under `_images` named `{your_project}-{platform}.fw`.  For example, I get `_images/cat_feeder-rpi2.fw`.  And it is *tiny*.  Under 18MB for Linux, Erlang, Elixir and my project code:

{% highlight bash %}
$ ll _images
-rw-r--r--   1 wsmoak  staff  17846793 Jan 10 17:23 cat_feeder-rpi2.fw
{% endhighlight %}

This file needs to be written to a micro-SD card and inserted into the Raspberry Pi.

Writing that micro-SD card on your own can be a challenge.  Have a look at the [instructions][image] on the Raspberry Pi website.  The good news is that [@fhunleth][fh] has written a utility called [`fwup`][fwup] (firmware update) to make it very easy.  The only trick is that it needs elevated privileges in order to mount and overwrite the card.

{% highlight bash %}
$ sudo fwup -a -i _images/cat_feeder-rpi2.fw -t complete
{% endhighlight %}

After that, you should have a good image on the micro-SD card that you can insert into the RPi.

After inserting the micro-SD in your RPi, you get another gift when you power it up-- it boots and starts running your program in about five seconds.

<iframe src="https://player.vimeo.com/video/151345328" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="https://vimeo.com/151345328">ElixirProximitySpinnyBoot</a> from <a href="https://vimeo.com/user1032254">Wendy Smoak</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

From my perspective, this is all **AMAZING**. I am *not* an embedded developer.  I expect things to Just Work when I turn them on.  And because of all the hard work by the Nerves and Bakeware teams, it did exactly that!

Interested in learning more?  [Request an Invitation][slack] to Elixir's Slack team and then come find us in the #nerves and #robotics channels.  There is also a [Nerves Google Group][group] but it's pretty quiet.

Now I just need something for the servo to turn to dispense the cat food.  I'm eyeing this [Augur-based Cat Feeder][thing] project on Thingiverse, but also considering just buying one such as this [replacement augur][sephra] for a Sephra chocolate fountain.

Copyright 2016 Wendy Smoak - This post first appeared on [{{ site.url }}][site-url] and is [CC BY-NC][cc-by-nc] licensed.

[eeia]: https://www.youtube.com/watch?v=kpzQrFC55q4
[wish]: https://www.adafruit.com/wishlists/386047
[augur]: https://www.sephra.com/accessories-parts/plastic-auger-for-select-cf16e.html
[bake]: http://www.bakeware.io/
[image]: https://www.raspberrypi.org/documentation/installation/installing-images/mac.md
[cf]: https://github.com/wsmoak/cat_feeder
[fh]: https://twitter.com/fhunleth
[nerves]: http://nerves-project.org/
[gh]: https://twitter.com/ghitchens
[thing]: http://www.thingiverse.com/thing:27854
[sephra]: https://www.sephra.com/accessories-parts/plastic-auger-for-select-cf16e.html
[js]: https://twitter.com/mobileoverlord
[fwup]: https://github.com/fhunleth/fwup
[slack]: https://elixir-slackin.herokuapp.com/
[group]: https://groups.google.com/forum/#!forum/nerves-project
[cc-by-nc]:  http://creativecommons.org/licenses/by-nc/3.0/
[site-url]: {{ site.url }}
