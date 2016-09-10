---
layout: post
title:  "My 'special' ThinkPad T500"
date:   2016-09-10
categories: technology
tags: thinkpad lenovo t500
---

I'm a big fan of ThinkPads. Over the years I've owned quite a variety. Starting with a T61p  then an X60, X61, T60 4:3, X200, X230, and the subject of this post -- the T500.

My T500 isn't the newest or most powerful, but it's my most unique ThinkPad. Not quite a 'FrankenPad' (a 4:3 T60 with a T61 motherboard, modded BIOS, and a Penryn CPU) but it draws parts from a few different places, and has been through the wars.

![Front][1]


>Windows 10 Pro 64-bit
>
>Intel Core 2 Duo E8335 @ 2.93GHz
>
>8.00GB Dual-Channel DDR3 1066MT/s (7-7-7-20)
>
>Generic PnP Monitor (1920x1200@60Hz)
>
>Intel GMA4500 MHD
>
>ATI Radeon HD3650 256MB VRAM
>
>111GB OCZ-VERTEX3 (SSD)

I picked it up from a seller specialising in ex-corporate computers. It was marked Grade C, so my hopes were not high, but the price was right.

When it arrived, it seemed to be in great condition, with only a slightly conspicuous chunk missing from the rear left corner of the lid. It was overheating quite badly, but after a re-application of thermal paste and a thorough removal of dust, everything was running smoothly. Much later I noticed that the back of the laptop was slightly bowed.

![Rear][4]

The T500's were known for being weak at this corner, with only a sliver of metal running down the rear left side, just over the fan exhaust. I imagine it was dropped onto this corner, leaving the whole thing a bit crooked. The outside of this computer isn't really the interesting part, though.

## Upgrades

I was on the cusp of selling this laptop on when I decided it would be worth more to me as a machine to play around with -- the cosmetic issues weren't big enough to matter to me, but they would devalue the machine. I previously had the notion I might want to build myself a FrankenPad, but unfortunately T60s and T61s are starting to creep up in price. That, and the FlexView 1600x1200 screens aren't cheap either, which is what I'd want to use ideally. Since I'm not a 4:3 zealot, it seemed a bit pointless really. 

### Storage

To start with I stuck an SSD into it, because... why not? SSDs make everything better! There are some mysteries regarding the SATA standard supported. The ICH9 is supposed to support full SATA-II 3.0Gbps, but apparently Lenovo may have limited this with a bandwidth cap -- honestly, I never bothered to check because I was more interested in improving read response times. Fast enough that you probably won't care unless you're doing some serious writing and reading to disk. I 


### Processor

Next, I checked to see which CPUs were compatible with it. It takes Socket P478 CPUs and the fastest one listed that this machine will accept is the T9900. Unofficially, it can take a Core 2 Duo Extreme X9000. Due to BIOS restrictions, it won't accept any Core 2 Quads.

The T9400 that was originally in the computer was no slouch, and I was dubious about the value of upgrading it. Anything above a T9700 started to cost as much as an entire T500, and with indications that the performance improvement would only be in the region of 10-15%, not worth it.

![Front left][2]

So, whilst browsing eBay for Socket P CPUs I found a few with odd model numbers I hadn't seen before, of the format *e8x35*. It turns that these were Intel Core 2 Duos manufactured especially for Apple iMacs, and were never available for retail purchase. A full list of these is available [here][5].

Stranger still, was that there were multiple revisions of each, with different clock speeds and TDP requirements (heat output). After doing a little bit of homework, I discovered that these were perfectly pin-compatible with a standard Socket P, non-Apple computer, and the only real concern was the TDP. The T8x00-9x00 series are all 35W TDP CPUs, but the best e8x35 processor, the E8435 SLGEA(E0), is 45W, and the earlier revision of the same CPU is 55W. 

The T500 CPU cooler is regarded to be one of the best available from that era of laptops, and is compatible the T60s and T61s. Many people who are building a FrankenPad, try to pick up a T500 Heat Sink Fan (HSF) to deal with the added thermal load of the higher-than-original-spec CPUs they generally use. With this in mind, I decided it wasn't going to be a big deal what CPU I ended up settling on -- in any case they were much, much cheaper than the T9700-and-up CPUs on eBay, probably due to relative obscurity.

It was settled by price. You can pick up an E8335 SLAQB(C0) (careful not to get the SLGED (E0) revision, or you might as well get a T9500!) for about £10. I ended up selling the T9400 for more than that, which is a nice demonstration on the economy of buying these more obscure iMac Core 2's. If I see one cheap enough, I'll pick up an E8435 and report back.

![Top][3]

## LCD Panel

The standard LCD with the T500 is a 1680x1050 16:10 panel from a handful of different manufacturers. In my experience, these LCDs are generally dull and mostly the backlights have seen better days. They are generally dim and yellow-hued. My T500's wasn't bad, but it wasn't great either. To make this a computer I'd enjoy using from day to day, this would be a necessary upgrade.

[Panelook.com][6] is an invaluable resource for finding compatible LCDs if all you have to go on is the connection technology (LVDS in this case), size, aspect ratio, and so on. I narrowed my selection down to [this list][7] and through a process of eBay searching each model number to determine the best priced items, I settled on the [Samsung LTN154CT02-002][8].

The only discrepancy with this panel is that required an inverter board that could drive two backlights -- it has two backlight plugs. There exists no ThinkPad inverter which can drive two backlights, so my only choice was to try it on one, see how it worked, and if it worked decide whether it was bright enough. 

Lo and behold it worked perfectly! Whether it could be brighter is immaterial, as it seems perfectly bright, with no bizarre side-effects or uneven backlighting. 

1200p is great on a screen this size, and still manages to feel very spacious even in 2016 when resolutions double that are routinely available on laptops. My X230 by comparison, at 1366x768, feels extremely cramped. Even a 12.5" display would benefit from 1600x900 surely? That's a story for another day though -- Lenovo choosing to use an LVDS 12.5" 16:9 panel when all other 12.5" panels are eDP, so no possible upgrade path exists unfortunately.

## Other upgrades

The keyboard I use on the T500 is from an IBM era T60. It's noticeably heavier, thicker feeling, and much less flex prone. It's probably the nicest keyboard on any machine I own. I use the 'golf tee' TrackPoint.

I installed 8GB of RAM immediately upon arrival. Whilst Windows 8-10 are both quite comfortable on 2-4GB of RAM, nothing makes the experience more smooth than just going all out. 8GB even for light users just means you'll never hit that bottleneck again!

I had some plans to put an mSATA drive in one of the spare mini PCI-e slots, but unfortunately the T520/420 and X220 machines were the first to support mSATA in the ThinkPad lineup. While there is no hardware support for mSATA, these machines could be configured with an Intel Turbo Cache drive between 2-4GB. That's about it in terms of adding any memory of another kind (apart from another SATA drive in the UltraBay). 

The only other substantial upgrade available for a T500 is to swap out the motherboard for a W500 board. The only reason to do this is if you want to se the ATI Mobility FireGL V5700 with 512MB VRAM. Apart from the added RAM on this video adapter, there is no real advantage to using it -- performance wise it will probably be a sidegrade, or barely worth noticing. I personally use the onboard graphics exclusively due to the non-existent support for switchable from this era from either Intel, AMD, or Lenovo (and improved battery life, less heat output). That being said, Lenovo is the one ultimately responsible for supporting their switchable graphics solution, and you can hardly expect them to bother with a machine so far into the sunset even a flat-earther can't see it.

So there it is! I still use it every day, probably more than my X230, despite it outperforming the T500 in every way. It's sort of like a comfy chair, or a vintage car. They have a class all of their own; it's not JUST about performance!



[1]: /assets/front.jpg
[2]: /assets/front-left.jpg
[3]: /assets/top.jpg
[4]: /assets/corner.jpg
[5]: https://en.wikipedia.org/wiki/List_of_Intel_Core_2_microprocessors#.22Penryn.22_.28Apple_iMac_specific.2C_45_nm.29
[6]: http://panelook.com
[7]: http://www.panelook.com/sizmodlist.php?st=&pl=&sizes[]=15.4&resolution_pixels=19201200&lamp_type=CCFL&signal_type_category=LVDS&application=NBPC
[8]: http://www.ebay.co.uk/itm/371476766152