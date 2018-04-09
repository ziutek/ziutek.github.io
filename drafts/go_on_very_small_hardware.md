---
layout: post
title: Go on very small hardware (Part 2)
tags: mcu go emgo
permalink: drafts/1
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{ site.baseur }}/2018/03/30/go_on_very_small_hardware2.html)

At the end of the [first part]({{ site.baseur }}/2018/03/30/go_on_very_small_hardware.html) of this article I promised to write about *interfaces*. I won't give you here a complete or even brief lecture about the interfaces. Instead, I'll show you a simple example how to define and use an interface, and then, how to take advantage of ubiquitous *io.Writer* interfece. At the end I'll show you couple of examples that will push our little board to its borders.

<!--more-->

Interfeces are a crucial part of Go language. If you want to learn more about them, I suggest to read [Effective Go](https://golang.org/doc/effective_go.html#interfaces) and [Russ Cox article](https://research.swtch.com/interfaces).

## Blinky, one more time

When you read the code of *Blinky* example, you probably noticed an counterintuitive way to set the LED on or off. The *Set* method was used to turn the LED off and the *Clear* method was used to turn the LED on. This is due to driving LEDs in open-drain configuration. What we can do to make the code less confusing? Lets define our own *LED* type with *On* and *Off* methods:

```Go
type LED struct {
	pin gpio.Pin
}

func (led LED) On() {
	led.pin.Clear()
}

func (led LED) Off() {
	led.pin.Set()
}
```

In all previous examples, I tried to use the same open-drain configuration to don't complicate the code. But in the last example, it would be easier for me, to connect the thrid LED between GND and PA3, configured in push-pull mode. The next example will use a LED connected this way.

But our new *LED* type doesn't handle this configuration. In fact, we should call it *OpenDrainLED* and define another *PushPullLED* type:

```Go
type PushPullLED struct {
	pin gpio.Pin
}

func (led PushPullLED) On() {
	led.pin.Set()
}

func (led PushPullLED) Off() {
	led.pin.Clear()
}
```

Note, that both types has the same methods, that work the same. It would be nice, if the rest of the code, that operate on LEDs, could use both types without paying attention to which one they use. The interfaces comes to help:

```Go
package main

import (
	"delay"

	"stm32/hal/gpio"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
)

type LED interface {
	On()
	Off()
}

type PushPullLED struct{ pin gpio.Pin }

func (led PushPullLED) On()  { led.pin.Set() }
func (led PushPullLED) Off() { led.pin.Clear() }

func MakePushPullLED(pin gpio.Pin) PushPullLED {
	pin.Setup(&gpio.Config{Mode: gpio.Out, Driver: gpio.PushPull})
	return PushPullLED{pin}
}

type OpenDrainLED struct{ pin gpio.Pin }

func (led OpenDrainLED) On()  { led.pin.Clear() }
func (led OpenDrainLED) Off() { led.pin.Set() }

func MakeOpenDrainLED(pin gpio.Pin) OpenDrainLED {
	pin.Setup(&gpio.Config{Mode: gpio.Out, Driver: gpio.OpenDrain})
	return OpenDrainLED{pin}
}

var led1, led2 LED

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(false)
	led1 = MakeOpenDrainLED(gpio.A.Pin(4))
	led2 = MakePushPullLED(gpio.A.Pin(3))
}

func blinky(led LED, period int) {
	for {
		led.On()
		delay.Millisec(100)
		led.Off()
		delay.Millisec(period - 100)
	}
}

func main() {
	go blinky(led1, 500)
	blinky(led2, 1000)
}
```


```
$ egc
$ arm-none-eabi-size cortexm0.elf 
   text    data     bss     dec     hex filename
  10356     196     212   10764    2a0c cortexm0.elf
```

![Interfaces]({{site.baseur}}/images/mcu/f030-demo-board/interfaces.png)