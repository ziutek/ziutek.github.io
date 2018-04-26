---
layout: post
title: Go on very small hardware (Part 3)
tags: mcu go emgo
permalink: drafts/4
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html)

Most of examples in the [first]({{site.baseur}}/2018/03/30/go_on_very_small_hardware.html) and [second]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html) part of this article are blinking LEDs in one or another way. It may have been interesting at first, but after a while it has become a bit boring. Let's do something more interesting...

<!--more-->

...let's light more LEDs!

## WS281x LEDs

The [WS281x](http://www.world-semi.com/solution/list-4-1.html) RGB LEDs (and their clones) are very popular. You can buy them as single elements, chained into longer strips or assembled into matrices, rings or other form-factors.

![WS2812B]({{site.baseur}}/images/led/ws2812b.jpg)

They can be connected in series and thanks to this, you can control a long LED strip with only single pin of your MCU. Unfortunately the phisical protocol used by their internal controller doesn't fit straight into any peripheral you can find in a MCU. You have to use bit-banging or use available peripherals in unusual way.

Which solutions is the most efficient depends on the number of LED strips controlled at the same time. If you have to drive 8 to 16 strips the most efficient way is to [use timers and DMA](http://www.martinhubacek.cz/arm/improved-stm32-ws2812b-library).

If you have to control only one strip, SPI or UART are best. In case of SPI you can encode two WS281x bits in one byte sent. UART allows more dense coding thanks to clever use of the start and stop bits: 3 bits per one byte sent.

The best explanation how the UART protocol fits to WS281x protocol I found on [this site](http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html). If you can't read Polish, here is its [English translation](https://translate.google.pl/translate?sl=pl&tl=en&u=http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html).

## RGB clock




