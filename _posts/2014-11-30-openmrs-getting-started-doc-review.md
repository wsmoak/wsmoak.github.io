---
layout: post
title:  "OpenMRS Getting Started Documentation Review"
date:   2014-11-30 12:30:00
tags: openmrs documentation
---

<blockquote class="twitter-tweet" lang="en"><p>Patching &quot;How To Contribute&quot; docs for open source projects as a service. Seriously, ping me if you want to work on one and don&#39;t see how.</p>&mdash; Wendy Smoak (@wsmoak) <a href="https://twitter.com/wsmoak/status/538860866452856832">November 30, 2014</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-conversation="none" lang="en"><p><a href="https://twitter.com/wsmoak">@wsmoak</a> <a href="https://twitter.com/nearyd">@nearyd</a> Thanks! Any suggestions on how we could improve <a href="http://t.co/gqcbAqeJgQ">http://t.co/gqcbAqeJgQ</a> ?</p>&mdash; Paul Biondich (@pbiondich) <a href="https://twitter.com/pbiondich/status/539073435675418624">November 30, 2014</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Sure!  First, A+ on making the Get Involved link on [http://openmrs.org](http://openmrs.org) both prominent and welcoming.  Having a great 'how to help' page doesn't matter if no one can find it.

Here are some observations and suggestions:

<h2>Help</h2>

The [help page](http://openmrs.org/help/) itself is well done.  I like the fact that you don't make people join a mailing list in order to talk to you -- there's a contact form where you make it easy to send in one's thoughts.  Hopefully that results in a prompt response and some personal attention to help the potential contributor find their way in.

I read the whole [help page](http://openmrs.org/help), and when I got to "Create a profile" at the bottom... I initially could not figure out how to do that.  There was nothing that looked like a link to click.  I checked the links in the footer.  Still no luck.  At some point I waved my mouse around and figured out that the orange headings are actually links, and clicking on "Get an OpenMRS ID" was what I needed to do.

Consider making the text "Create a profile" into a link, and also adding id.openmrs.org to the list of 'Other OpenMRS sites' in the footer.

The same issue exists for the text "let us know!" under the Suggest heading.  Since it's not immediately obvious that the orange headings are, in fact, links, I suggest making a bit of text under each of the headings clickable as well.

<h2>Document</h2>

Moving on, the first sentence under Document is, "Do you have experience and skills with technical writing?"  This makes it sound like you only want experienced technical writers as contributors, which I'm sure is not the case.  Clicking through, the [/help/document](http://openmrs.org/help/document) page is much more welcoming.

Consider something like, "OpenMRS needs assistance maintaining its documentation and resources. It is important to keep everything up to date and well-organized. We welcome anyone with an interest in writing and organizing information. Don't worry if you're not an experienced technical writer, there's a place for everyone!"

Also on [/help/document](http://openmrs.org/help/document), it would be nice if "Let us know" linked to the Contact form with the 'Getting Involved & Volunteering' value pre-filled and the Documentation checkbox already checked.  The fewer options the person has to think about and decide upon, the more likely they'll finish it and submit the form.

Finally, I could not find any information on how to patch [http://openmrs.org/help](http://openmrs.org/help) itself, or I would have. :)

<h2>Develop</h2>

While skimming the docs for new developers, I joined the #openmrs IRC channel on Freenode.  As luck would have it, one of the Google Code-In participants, was having trouble building the project.  Unable to resist, I went back to the docs to see if I could help.

I had no trouble finding and cloning the openmrs-core code, and after noting that there was a pom.xml, tried a naive 'mvn install' on it.

Sure enough, I had the same two test errors the student did.  (He also had two test *failures* that I did not see.)

And another person [reported the same thing](https://groups.google.com/a/openmrs.org/forum/#!msg/dev/zwQ5bsngpks/qfUpFqnyI3EJ) last week on the OpenMRS Developers group.

With three of us experiencing it, there's some kind of an issue here.  A build that doesn't work for a new user is a *huge* barrier to entry, because they may not know the tools well enough to get around it.

I replied to the thread on the [OpenMRS Developers group](https://groups.google.com/a/openmrs.org/forum/#!forum/dev) so hopefully something can be done to improve things.  Perhaps those problematic tests could be skipped by default, and the [set up docs](http://en.flossmanuals.net/openmrs-developers-guide/get-set-up/) could be improved to explain what to do if the build fails.

All the text on the wiki and the channel /topic says the IRC channel is logged, but The [logs after March 2014 are missing](https://wiki.openmrs.org/display/IRC/2014+-+OpenMRS).  If this is not something the community wants to work on, consider asking [BotBot.me](https://botbot.me) or another service to log the channel for you.

<h3>Summary</h3>

The getting started docs for OpenMRS are great.  Except for the build failure, these are all minor suggested improvements.  I don't think I've seen another project that publishes a [*book*](http://en.flossmanuals.net/openmrs-developers-guide/) on how to get started!
