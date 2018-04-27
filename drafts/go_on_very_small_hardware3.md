---
layout: post
title: Go on very small hardware (Part 3)
tags: mcu go emgo
permalink: drafts/4
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html)

Most of examples in the [first]({{site.baseur}}/2018/03/30/go_on_very_small_hardware.html) and [second]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html) part of this article are blinking LEDs in one or another way. It may have been interesting at first, but after a while it has become a bit boring. Let's do something more entertaining...

<!--more-->

 ...let's light more LEDs!

## WS281x LEDs

The [WS281x](http://www.world-semi.com/solution/list-4-1.html) RGB LEDs (and their clones) are very popular. You can buy them as single elements, chained into long strips or assembled into matrices, rings or other form-factors.

![WS2812B]({{site.baseur}}/images/led/ws2812b.jpg)

They can be connected in series and thanks to this, you can control a long LED strip with only single pin of your MCU. Unfortunately, the phisical protocol used by their internal controller doesn't fit straight into any peripheral you can find in a MCU. You have to use bit-banging or use available peripherals in unusual way.

Which solutions is the most efficient depends on the number of LED strips controlled at the same time. If you have to drive 4 to 16 strips the most efficient way is to [use timers and DMA](http://www.martinhubacek.cz/arm/improved-stm32-ws2812b-library) (don't overlook the links at the end of Martin's article).

If you have to control only one or two strips, use available SPI or UART peripherals. In case of SPI you can encode only two WS281x bits in one byte sent. UART allows more dense coding thanks to clever use of the start and stop bits: 3 bits per one byte sent.

The best explanation of how the UART protocol fits into WS281x protocol I found on [this site](http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html). If you can't read Polish, here is its [English translation](https://translate.google.pl/translate?sl=pl&tl=en&u=http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html).

## RGB clock

There are many examples of clocks built of RGB LEDs on the Intrnet. Let's build our own using our STM32F030 board and this LED ring:

![WS2812B]({{site.baseur}}/images/led/ws2812b.jpg)

It has 24 individually addressable WS2812B LEDs and exposes four terminals: GND, POWER (3.5 - 5.3 V), data IN and data OUT. You can chain more rings or other WS2812 based things by connecting their IN terminals to the OUT terminals of previous ones.

We will use UART driver so the IN terminal should be connected to TxD pin on UART header. The LEDs can consume quite a lot of current so during programming/debuggin it's best to connected the GND and POWER terminals directly to the GND and 5V pins available on ST-LINK programmer:

![WS2812B]({{site.baseur}}/images/led/ws2812b.jpg)

Our STM32F030F4P6 MCU (whole STM32F0 and newer families) has one important thing that the STM32F1, STM32F4, STM32L1 MCUs don't have: it allows to invert UART signals and therefore we can connect this ring directly to the UART TX pin. If you don't known that we need such inversion you probably didn't read the [article](https://translate.google.pl/translate?sl=pl&tl=en&u=http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html) I mentioned above.

So you can't use popular [Blue Pill](https://jeelabs.org/article/1649a/) or [STM32F4-DISCOVERY](http://www.st.com/en/evaluation-tools/stm32f4discovery.html) this way. Use their SPI peripheral or an external inverter (see [Christmas Tree Lights](https://github.com/ziutek/emgo/tree/master/egpath/src/stm32/examples/minidev/treelights) for more information).

Let's finish this lengthy introduction and go to the code:

