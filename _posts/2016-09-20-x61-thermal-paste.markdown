---
layout: post
title:  "The ThinkPad X61: Thermal paste replacement"
date:   2016-09-20
categories: technology
tags: lenovo thinkpad x61 thermal paste teardown
---

The X61 was the last 4:3 aspect ultraportable produced by Lenovo; the end of an era. I've owned a couple of X60's and an X61 and they all seemed to share the same issue -- poor thermal performance. In laymans terms, they run seriously hot. 

![Front][1]

The model used by my partner has an Intel T7100 1.83GHz CPU. A little trick that can be used with these laptops for a little bit more performance, is to enable Dual-IDA by installing [Middleton's BIOS][7]. IDA (Intel Dynamic Acceleration, later Turbo Boost) was, on these CPUs, limited to one core. It basically increments the CPU multiplier by one on one core for a bit of extra performance, shortening the race-to-idle. With Dual-IDA, it allows it to happen on both cores simultaneously (I use [ThrottleStop][8] to manage Core 2's). Unfortunately the X61 gets mighty toasty when making use of this trick, so I decided it might be time to investigate the thermal paste. 

# The teardown

![Keyboard off][2]

Unfortunately the X61 isn't quite as easy to work on as, say, the T61 or T500. Being an ultraportable meant the design had to be a bit more clever to keep everthing fitting inside the limited space. In the X61's case, the CPU and HSF are on the bottom of the board and the only way to access it is to remove the entire board.

![Motherboard removed][3]

As you can see above, there are quite a few cables with tiny connectors which have to be removed before you can lift the board out. I wonder if all the plastic protection (presumably anti-static?) on the top layer of the board might contributing to the generally poor thermal performance? The type of protection is generally only used sparingly on the larger laptops.

![Motherboard bottom][4]

Once the board is out, it's a simple process of unscrewing the heat sink. Note: to remove it completely, there's a screw that attaches it from the other side of the board, inside the ExpressCard slot -- it's easy to miss.

![Old paste][5]

The old paste was fairly caked on, which is fairly common for factory installed paste. It wasn't as dry and brittle as I have come to expect. Unfortunately my thermal paste (Arctic MX-5) was dropped and the nozzle popped off, leaving me with a hassle every time I want to put some paste on something. It's always messy!

![New paste][6]

Not my finest hour in terms of thermal paste application -- I did what I could without making a pig's ear of it. I decided to ditch the thermal pad covering the, what I presume to be, Intel X3100 chip in favour of fairly hefty quantity of paste -- I had to make up the thickness of the pad. I reckon a thick helping of paste is better than any pad though I tend to subscribe to the 'less is more' school of thermal paste (if such a thing exists).

So far the thermals seem marginally better, but I'm going to let the paste cure for a bit before passing final judgment. I wish it had a Penryn!

[1]: /assets/x61/DSC_0222.JPG
[2]: /assets/x61/DSC_0223.JPG
[3]: /assets/x61/DSC_0224.JPG
[4]: /assets/x61/DSC_0225.JPG
[5]: /assets/x61/DSC_0227.JPG
[6]: /assets/x61/DSC_0228.JPG
[7]: http://www.thinkwiki.org/wiki/Middleton's_BIOS#Installation
