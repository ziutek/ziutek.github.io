---
layout: post
title: Go on very small hardware (Part 4)
tags: mcu go emgo
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{site.baseur}}/2018/04/14/go_on_very_small_hardware4.html)

In the last post in this series we will try to draw to a LCD display.

<!--more-->

## LCD display

There are hundreds of different types of displays targeting the MCU market. From simple text displays based on [HD44780](https://en.wikipedia.org/wiki/Hitachi_HD44780_LCD_controller) compatible controllers to [Nextion](https://nextion.itead.cc/) "intelligent" display with its own MCU and up to 8 MB RAM.

I'll present there my favorite one based on [FT811](http://www.ftdichip.com/Products/ICs/FT81X.html) controller.

In my opinion, there are definitely two designs that stand out in the world of microcontrollers. The first one is the Nordic [nRF5 series](https://www.nordicsemi.com/eng/Products/2.4GHz-RF) of Bluetooth enabled MCUs. The second one is the FTDI EVE series of display controllers (FT80x, FT81x), both worth writing a separate articles.

#### Digression about the nRF5 MCUs

To understand what I'm about to write, you should definitely read the Rob Pike's [article](https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html).

Let's take a look at the great STM32 series of MCUs. They have an extremely rich array of peripherals, every with hundred or thousands configuration bits. You can find the right bit for almost your every whim, and if not, then it will definitely appear in the newer family. You can say that in the MCU world the STM32 series is like C++ in the programming language world and the latest STM32 MCUs are the equivalent of the C++14. 

Against the background of STM32 the Nordic nRF5 series can be compared to the Go language.

The nRF5 gives you a set of simple tools (peripherals) and a key concept of events and tasks. To solve a specific problem you can combine these simple peripherals by connecting their events and task. You have to be creative and don't count on the presence of some specific bit in the configuration register.

Additionaly, interrupts, peripherals and its registers are arranged according to the one, general and clear template. For example, you can easily infer the IRQ number of a peripheral from its base address.

#### Shortly about FTDI Embedded Video Engine (EVE)

The EVE provides a drawing interface similar to OpenGL 2D. But the developers of EVE didn't follow the usual path. They found a way to get the maximum effect with the minimum of resources.

If you think about typical GPU based desing you think about the GPU as a specialized processor that executes commands and draws to a large framebuffer. We're just about connecting a relatively big 800x480 display to our MCU. For 32-bit RGBA it requires about 1.5 MB framebuffer. FT811 uses only 6400 bytes for its double-buffered framebuffer (my estimation). How is it possible?

It's posible because the FT811 renders its scene (display list) line by line during the LCD refresh. It probably renders the next line when the previous one is refreshed, so from this I infer that it has two-line framebuffer. 

The display list is double-buffered in RAM of size 2 x 8 kB. The MCU writes basic commands directly to the inactive display list or writes basic and high-level commands to the Graphics Engine (GE) FIFO of size 4 kB. The GE executes higl-level commands like buttons, scroll bars, progress bars, etc. from its FIFO and converts them to the basic commands, written to the inactive display list. When the scene is ready you simply swap buffers.

There is also 1 MB of general purpose RAM where you can store bitmaps, decompress JPEG and PNG images, setup video queues, store audio files, store additional fonts and so on.

As you can see FT811 uses 1050 kB of RAM for a lot of things instead of 1500 kB for only one framebuffer (you need two for double buffering).

The EVE also handles touchscreen. You can assign a tag to any object in display list. The GE will tell you which object is touched by reporting its tag. It also can assists in tracking touches on graphic objects by report angle or distance.

#### The HOTMCU FT811CB-HY50HD display

HAOYU Electronics aka HOTMCU is probably the cheapest source of FT8xx based displays. When I was working on the *display/eve* package I've used two of them: [FT800CB-HY50B](https://www.hotmcu.com/5-graphical-lcd-touchscreen-480x272-spi-ft800-p-124.html?cPath=6_16) and [FT811CB-HY50HD](https://www.hotmcu.com/5-graphical-lcd-capacitive-touch-screen-800x480-spi-ft811-p-301.html). Bellow we will deal with the latter one:

![WS2812B]({{site.baseur}}/images/lcd/ft811cb-hy50hd-front.jpg)

![WS2812B]({{site.baseur}}/images/lcd/ft811cb-hy50hd-back.jpg)

Its TFT panel isn't outstanding, to put it mildly, but good enough for development purposes. Unfortunately, it requires a small modification to work seamlessly with 3.3V logic:

![WS2812B]({{site.baseur}}/images/lcd/ft811cb-mod.jpg)

It has two 74LVC125A that work as level shifters. They probably work well with 5V logic, even with 30 MHz SPI clock, but in case of 3.3V logic they make the screen look damaged. You can avoid these problems without desoldering anything by reducing the SPI speed, unfortunately I don't remember how much.

 This modificatin allows also use EVE2 2-bit QSPI mode which doubles the available bandwidth.