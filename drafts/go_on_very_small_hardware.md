---
layout: post
title: Go on very small hardware
permalink: /drafts/1
---

How low we can Go and still do something useful? I recently bought this board:

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/board.jpg)

and now I can share my experiences.

## The Hardware

The [STM32F030F4P6](http://www.st.com/content/st_com/en/products/microcontrollers/stm32-32-bit-arm-cortex-mcus/stm32-mainstream-mcus/stm32f0-series/stm32f0x0-value-line/stm32f030f4.html) is impresive piece of hardware:

* CPU: [Cortex M0](https://en.wikipedia.org/wiki/ARM_Cortex-M#Cortex-M0) 48 MHz (only 12000 logic gates in minimal configuration),
* RAM: 4 KB,
* Flash: 16 KB,
* ADC, SPI, I2C, USART and a couple of timers,

all enclosed in TSSOP20 package. As you can see, this is very small 32-bit system.

## The software

If you hoped to read how to use [genuine Go](https://golang.org/) to program this board, you need to read the hardware specification one more time. You must face the truth: there is a negligible chance that someone will ever port real Go to Cortex-M0 (ARMv6-M) architecture and this is just the beginning of work.

We will use [Emgo](https://github.com/ziutek/emgo), but don't worry, you will see that it gives you as much Go as it can on such small system.

There was no support for any F0 MCU in [stm32/hal](https://github.com/ziutek/emgo/tree/master/egpath/src/stm32/hal) before this board arrived to me. After brief study of [RM](http://www.st.com/resource/en/reference_manual/dm00091010.pdf), the STM32F0 series appeared to be striped down series STM32F3 which made work on new port easier.

## The minimal program

```Go
package main

func main() {
}
```