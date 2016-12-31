---
layout: post
title:  "One-Time Pad with GenStage"
date:   2016-12-31 12:35:00
tags: elixir genstage adventofcode
---

Ever since I [read the blog post](http://elixir-lang.org/blog/2016/07/14/announcing-genstage/) and [heard José talk about GenStage](https://www.youtube.com/watch?v=aZuY5-2lwW4) I've wanted to try it out.  Lacking an actual _need_ for it however, I had to wait for one to appear. Finally in 2016's Advent of Code Day 14: One-Time Pad, I found a candidate problem.  Let's see how to use GenStage to solve it.

You can read the problem description here: <http://adventofcode.com/2016/day/14> .  In brief, you need to produce a bunch of md5 hashes and then look for duplicated characters in them.  If a particular hash contains a triplet like "aaa", then you need to look at the next 1000 hashes to see if there is an "aaaaa" quintuplet in any of those.

The overall idea is:

1. generate hashes and check each one in order
2. does this hash contain a triplet?
3. if yes, check the next 1000 hashes for a quintuplet
4. if you find one, add the index of the hash-with-triplet to a list
5. stop at the 64th one that satisfies the triplet-and-quintuplet conditions

(It turns out that you don't need all 64 of them, you just need the index of the 64th one for the answer. But when you are doing Part 1 of an Advent of Code problem, you learn not to throw away interesting information, as it is often asked for in Part 2!)

Initially I set up a HashProducer, a Worker, and a Decider.  The Worker would keep a queue of the hashes and check for triplets.  If it found one, it would send the next 1000 hashes off to the Decider, which would print a message if it found the quintuplet.

This worked, but it got slower and slower... after adding some print statements, I realized that every time through `handle_events` it would ask the producer for more hashes.  Since I was only processing _one_ hash each time through, the queue got huge, and the operations on it were taking longer and longer.

A very simple solution might have been to configure the number of items the HashProducer would send to only 1, however I had seen this section of the docs and wanted to figure out how to manually request items when needed:

<https://hexdocs.pm/gen_stage/Experimental.GenStage.html#module-asynchronous-work-and>

By returning `{:manual, state}` from `handle_subscribe` in the consumer, you are in control of asking for more items from the upstream producer.  Because Worker is both a producer _and_ a consumer, I needed to implement `handle_subscribe` twice and pattern match on the first argument to return the correct thing.

In part 2 of the problem, the Part2HashProducer takes _much_ longer to generate the hashes.  The timing shown in the code could certainly be tuned so that the Worker spends less time waiting for the queue to be replenished.

There is a final process, the Gatherer, that simply receives the items that have passed both checks, and stops when it reaches 64 of them.

Here is the code:  <https://github.com/wsmoak/advent_of_code/tree/2016/one_time_pad>

In particular, look at [this file](https://github.com/wsmoak/advent_of_code/blob/2016/one_time_pad/lib/one_time_pad.ex) to see how the stages are assembled.

There are _many_ potential improvements here, but there are still 11 more days of Advent of Code and I want to finish them first!
