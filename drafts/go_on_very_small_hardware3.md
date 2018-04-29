---
layout: post
title: Go on very small hardware (Part 3)
tags: mcu go emgo
permalink: drafts/4
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html)

Most of the examples discussed in the [first]({{site.baseur}}/2018/03/30/go_on_very_small_hardware.html) and [second]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html) part of this article are blinking LEDs in one or another way. It may have been interesting at first, but after a while it has become a bit boring. Let's do something more entertaining...

<!--more-->

 ...let's light more LEDs!?

## WS281x LEDs

The [WS281x](http://www.world-semi.com/solution/list-4-1.html) RGB LEDs (and their clones) are very popular. You can buy them as single elements, chained into long strips or assembled into matrices, rings or other form-factors.

![WS2812B]({{site.baseur}}/images/led/ws2812b.jpg)

They can be connected in series and thanks to this, you can control a long LED strip with only single pin of your MCU. Unfortunately, the phisical protocol used by their internal controller doesn't fit straight into any peripheral you can find in a MCU. You have to use bit-banging or use available peripherals in unusual way.

Which of the available solutions is the most efficient depends on the number of LED strips controlled at the same time. If you have to drive 4 to 16 strips the most efficient way is to [use timers and DMA](http://www.martinhubacek.cz/arm/improved-stm32-ws2812b-library) (don't overlook the links at the end of Martin's article).

If you have to control only one or two strips, use the available SPI or UART peripherals. In case of SPI you can encode only two WS281x bits in one byte sent. UART allows more dense coding thanks to clever use of the start and stop bits: 3 bits per one byte sent.

The best explanation of how the UART protocol fits into WS281x protocol I found on [this site](http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html). If you don't know Polish, here is the [English translation](https://translate.google.pl/translate?sl=pl&tl=en&u=http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html).

## RGB clock

There are many examples of clocks built of RGB LEDs on the Intrnet. Let's make our own using STM32F030 board and this LED ring:

![WS2812B]({{site.baseur}}/images/led/rgbring.jpg)

This ring has 24 individually addressable RGB LEDs (WS2812B) and exposes four terminals: GND, 5V, DI and DO. You can chain more rings or other WS2812 based things by connecting DI (data in) terminal to the DO (data out) terminal of previous one.

We will use UART based driver so the DI should be connected to TXD pin on UART header. The LEDs requires a power supply with at least 3.5 V and can consume quite a lot of current so during programming/debuggin it's best to connect the GND and 5V terminals directly to the GND and 5V pins available on ST-LINK programmer:

![WS2812B]({{site.baseur}}/images/led/ring-stlink-f030.jpg)

Our STM32F030F4P6 MCU, and whole STM32F0, STM32F7, STM32L4 families, have one important thing that the STM32F1, STM32F4, STM32L1 MCUs don't have: it allows to invert UART signals and therefore we can connect this ring directly to the UART TXD pin. If you don't known that we need such inversion you probably didn't read the [article](https://translate.google.pl/translate?sl=pl&tl=en&u=http://mikrokontrolery.blogspot.com/2011/03/Diody-WS2812B-sterowanie-XMega-cz-2.html) I mentioned above.

So you can't use popular [Blue Pill](https://jeelabs.org/article/1649a/) or [STM32F4-DISCOVERY](http://www.st.com/en/evaluation-tools/stm32f4discovery.html) this way. Use their SPI peripheral or an external inverter (see the [Christmas Tree Lights](https://github.com/ziutek/emgo/tree/master/egpath/src/stm32/examples/minidev/treelights) project for more information).

Let's finish this lengthy introduction and go to the code:

```go
package main

import (
	"delay"
	"rtos"

	"led"
	"led/ws281x/wsuart"

	"stm32/hal/dma"
	"stm32/hal/exti"
	"stm32/hal/gpio"
	"stm32/hal/irq"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
	"stm32/hal/usart"
)

var (
	tts   *usart.Driver
	btn   gpio.Pin
	btnev rtos.EventFlag
)

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(true)
	btn = gpio.A.Pin(4)
	tx := gpio.A.Pin(9)

	btn.Setup(&gpio.Config{Mode: gpio.In, Pull: gpio.PullUp})
	ei := exti.Lines(btn.Mask())
	ei.Connect(btn.Port())
	ei.EnableFallTrig()
	ei.EnableRisiTrig()
	ei.EnableIRQ()
	rtos.IRQ(irq.EXTI4_15).Enable()

	tx.Setup(&gpio.Config{Mode: gpio.Alt})
	tx.SetAltFunc(gpio.USART1_AF1)
	d := dma.DMA1
	d.EnableClock(true)

	tts = usart.NewDriver(usart.USART1, d.Channel(2, 0), nil, nil)
	tts.Periph().EnableClock(true)
	tts.Periph().SetBaudRate(3000000000 / 1390)
	tts.Periph().SetConf2(usart.TxInv)
	tts.Periph().Enable()
	tts.EnableTx()
	rtos.IRQ(irq.USART1).Enable()
	rtos.IRQ(irq.DMA1_Channel2_3).Enable()
}

func main() {
	rgb := wsuart.GRB
	strip := make(wsuart.Strip, 24)
	ds := 4 * 60 / len(strip) // Interval between LEDs (quarter-seconds).
	adjust := 0
	adjspeed := ds
	for {
		qs := int(rtos.Nanosec() / 25e7) // Quarter-seconds since reset.
		qa := qs + adjust

		qa %= 12 * 3600 * 4 // Quarter-seconds since 0:00 or 12:00.
		hi := len(strip) * qa / (12 * 3600 * 4)

		qa %= 3600 * 4 // Quarter-seconds in the current hour.
		mi := len(strip) * qa / (3600 * 4)

		qa %= 60 * 4 // Quarter-seconds in the current minute.
		si := len(strip) * qa / (60 * 4)

		hc := led.Color(0x550000)
		mc := led.Color(0x005500)
		sc := led.Color(0x000055)

		// Blend colors if the hands of the clock overlap.
		if hi == mi {
			hc |= mc
			mc = hc
		}
		if mi == si {
			mc |= sc
			sc = mc
		}
		if si == hi {
			sc |= hc
			hc = sc
		}

		// Draw the clock and send to the ring.
		strip.Clear()
		strip[hi] = rgb.Pixel(hc)
		strip[mi] = rgb.Pixel(mc)
		strip[si] = rgb.Pixel(sc)
		tts.Write(strip.Bytes())

		// Sleep until the button pressed or the second hand should be moved.
		if btnWait(0, int64(qs+ds)*25e7) {
			adjust += adjspeed
			// Sleep until the button is released or timeout.
			if !btnWait(1, rtos.Nanosec()+100e6) {
				if adjspeed < 5*60*4 {
					adjspeed += 2 * ds
				}
				continue
			}
			adjspeed = ds
		}
	}
}

func btnWait(state int, deadline int64) bool {
	for btn.Load() != state {
		if !btnev.Wait(1, deadline) {
			return false // timeout
		}
		btnev.Reset(0)
	}
	delay.Millisec(50) // debouncing
	return true
}

func exti4_15ISR() {
	pending := exti.Pending() & 0xFFF0
	pending.ClearPending()
	if pending&exti.Lines(btn.Mask()) != 0 {
		btnev.Signal(1)
	}
}

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
	irq.EXTI4_15:        exti4_15ISR,
}
```

#### The *import* section

The only new thing in the *import* section compared to the previous examples is the *led* package with its *led/ws281x* subtree. Currently the *led* package itself contains only definition of *Color* type. I was wondering about *color* package outside the *led* tree or using *Color* or *RGBA* type from *image/color*. For now I ended with *led.Color* type but not very happy with it. I was also wondering to define LED strip in the way that it will implement *image.Image* interface but because of using of [gamma correction](https://en.wikipedia.org/wiki/Gamma_correction) and big overhead of *image/draw* package I ended with simple `type Strip []Pixel`.

#### The *init* function

There are three new things in the *init* function:

1. The PA4 pin is configured as input with internal pull-up resistor enabled:
```go
btn.Setup(&gpio.Config{Mode: gpio.In, Pull: gpio.PullUp})
```
This pin is connected to the onboard LED but that doesn't hinder anything. More important is that it's located next to the GND pin so we can use any metal object to set the clock. As a bonus we have additional feedback from onboard LED.

2. The EXTI peripheral is configured to track PA4 input and generate an interrupt on any change:
```go
ei := exti.Lines(btn.Mask())
ei.Connect(btn.Port())
ei.EnableFallTrig()
ei.EnableRisiTrig()
ei.EnableIRQ()
```

3. The inversion of UART Tx signal is enabled:
```go
tts.Periph().SetConf2(usart.TxInv)
```

Additionally the baudrate was changed from 115200 to 3000000000/1390 ≈ 2158273 which corresponds to 1390 nanoseconds per WS2812 bit.

#### The *main* function
---->

The *main* function contains whole logic of our clock. The *rgb* variable is set to the color order used by WS2812 controller. This code works also with WS2811 controllers -- the only necessary change is set the color order to RGB. To tell the truth, the baudrate should also be changed but in practice the WS2811 works also with WS2812 timing.

We use the *rtos.Nanosec* function instead of *time.New* to obtain the current time. This saves much of Flash but also reduces our clock to antique device that has no idea about days, months and years and worst of all it doesn't handle daylight saving changes.

Our ring has 24 LEDs, so the second hand can be presented with an accuracy of 2.5 s. To don't sacrifice this accuracy and get smooth operation we use half-second as base interval.

The red, green and blue colors are used respectively for hour, minute and second hands. This allows us to use simple *or* operation for color blending. There is a *Color.Blend* method that can blend arbitrary colors but we are low of Flash so we prefer simplest possible solution.

The *strip* slice acts as a framebuffer. The `strip.Clear()` clears the whole framebuffer to the black color. Next we set the color of three selected pixels to display three clock hands. The `tts.Write(strip.Bytes())` sends the content of the framebuffer to the ring.

Then we have the code that reads the state of PA4 pin to adjust the clock. There is an acceleration when the "button" is held down for some time.

### Interrupts.

The program is ened with the code that handles interrupts, the same as in the [UART example]({{site.baseur}}/2018/04/14/go_on_very_small_hardware2.html#uart).

{::nomarkdown}
<video width=576 height=324 controls preload=auto>
	<source src='{{site.baseur}}/videos/rgb-clock.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
{:/}