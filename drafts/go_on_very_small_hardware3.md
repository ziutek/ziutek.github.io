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

Let's finish this lengthy introduction and go to the code. This time I will put the code and description in pieces.

Let's start with the *import* section:

```go
package main

import (
	"delay"
	"rtos"

	"led"
	"led/ws281x/wsuart"

	"stm32/hal/dma"
	"stm32/hal/gpio"
	"stm32/hal/irq"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
	"stm32/hal/usart"
)
```

The only new thing compared to previous examples is *led* package with its *led/ws281x* subtree. Currently *led* contains only definition of *Color* type. I was wondering about *color* package outside the *led* tree or using *Color* or *RGBA* type from *image/color*. For now I ended with *led.Color* type but I'm not very happy about it. I was also wondering to define LED strip in the way that it will implement *image.Image* interface but because of *gamma correction* and big overhead of *image/draw* package I ended with simple slice.

There are also not many novelties in the *init* function:

```go
var (
	tts *usart.Driver
	btn gpio.Pin
)

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(true)
	btn = gpio.A.Pin(4)
	tx := gpio.A.Pin(9)

	btn.Setup(&gpio.Config{Mode: gpio.In, Pull: gpio.PullUp})

	tx.Setup(&gpio.Config{Mode: gpio.Alt})
	tx.SetAltFunc(gpio.USART1_AF1)
	d := dma.DMA1
	d.EnableClock(true)

	// 1390 ns/WS2812bit = 3 * 463 ns/UARTbit -> BR = 3 * 1e9 ns/s / 1390 ns/bit

	tts = usart.NewDriver(usart.USART1, d.Channel(2, 0), nil, nil)
	tts.Periph().EnableClock(true)
	tts.Periph().SetBaudRate(3000000000 / 1390)
	tts.Periph().SetConf2(usart.TxInv) // STM32F0 need no external inverter.
	tts.Periph().Enable()
	tts.EnableTx()

	rtos.IRQ(irq.USART1).Enable()
	rtos.IRQ(irq.DMA1_Channel2_3).Enable()
}
```

```go
func main() {
	var setClock, setSpeed int
	rgb := wsuart.GRB
	strip := make(wsuart.Strip, 24)
	for {
		hs := int(rtos.Nanosec() / 5e8) // Half-seconds elapsed since reset.
		hs += setClock

		hs %= 12 * 3600 * 2 // Half-seconds from the last 0:00 or 12:00.
		h := len(strip) * hs / (12 * 3600 * 2)

		hs %= 3600 * 2 // Half-second from the beginning of the current hour.
		m := len(strip) * hs / (3600 * 2)

		hs %= 60 * 2 // Half-second from the beginning of the current minute.
		s := len(strip) * hs / (60 * 2)

		hc := led.Color(0x550000)
		mc := led.Color(0x005500)
		sc := led.Color(0x000055)

		// Blend colors if the hands of the clock overlap.
		if h == m {
			hc |= mc
			mc = hc
		}
		if m == s {
			mc |= sc
			sc = mc
		}
		if s == h {
			sc |= hc
			hc = sc
		}

		strip.Clear()
		strip[h] = rgb.Pixel(hc)
		strip[m] = rgb.Pixel(mc)
		strip[s] = rgb.Pixel(sc)
		tts.Write(strip.Bytes())

		// Set colck.
		if btn.Load() == 0 {
			setClock += setSpeed
			i := 0
			for btn.Load() == 0 && i < 10 {
				delay.Millisec(20)
				i++
			}
			if i == 10 && setSpeed < 10*60*2 {
				setSpeed += 10
			}
			continue
		}
		setSpeed = 5
		delay.Millisec(50)
	}
}
```

```go
func ttsISR() {
	tts.ISR()
}

func ttsDMAISR() {
	tts.TxDMAISR()
}

//c:__attribute__((section(".ISRs")))
var ISRs = [...]func(){
	irq.USART1:          ttsISR,
	irq.DMA1_Channel2_3: ttsDMAISR,
}
```