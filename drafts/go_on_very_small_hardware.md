---
layout: post
title: Go on very small hardware
permalink: /drafts/1
---

How low we can *Go* and still do something useful?

I recently bought this ridiculously cheap board:

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/board.jpg)

and now I can share my experiences.

## The Hardware

The [STM32F030F4P6](http://www.st.com/content/st_com/en/products/microcontrollers/stm32-32-bit-arm-cortex-mcus/stm32-mainstream-mcus/stm32f0-series/stm32f0x0-value-line/stm32f030f4.html) is impresive piece of hardware:

* CPU: [Cortex M0](https://en.wikipedia.org/wiki/ARM_Cortex-M#Cortex-M0) 48 MHz (only 12000 logic gates in minimal configuration),
* RAM: 4 KB,
* Flash: 16 KB,
* ADC, SPI, I2C, USART and a couple of timers,

all enclosed in TSSOP20 package. As you can see, it is very small 32-bit system.

## The software

If you hoped to read how to use [genuine Go](https://golang.org/) to program this board, you need to read the hardware specification one more time. You must face the truth: there is a negligible chance that someone will ever add support for Cortex-M0 (ARMv6-M) to the Go compiler and this is just the beginning of work.

We'll use [Emgo](https://github.com/ziutek/emgo), but don't worry, you will see that it gives you as much Go as it can on such small system.

There was no support for any F0 MCU in [stm32/hal](https://github.com/ziutek/emgo/tree/master/egpath/src/stm32/hal) before this board arrived to me. After brief study of [RM](http://www.st.com/resource/en/reference_manual/dm00091010.pdf), the STM32F0 series appeared to be striped down STM32F3 series, which made work on new port easier.

If you want to follow subsequent steps, you need to install Emgo

```
cd $HOME
git clone https://github.com/ziutek/emgo/
cd emgo/egc
go install
```

and set a couple environment variables

```
export EGCC=path_to_arm_gcc      # eg. /usr/local/arm/bin/arm-none-eabi-gcc
export EGLD=path_to_arm_linekr   # eg. /usr/local/arm/bin/arm-none-eabi-ld
export EGAR=path_to_arm_archiver # eg. /usr/local/arm/bin/arm-none-eabi-ar

export EGROOT=$HOME/emgo/egroot
export EGPATH=$HOME/emgo/egpath

export EGARCH=cortexm0
export EGOS=noos
export EGTARGET=f030x6
```

A more detailed description can be found on the [Emgo website](https://github.com/ziutek/emgo).

Ensure that egc is on your PATH. You can use `go build` instead of `go install` and copy egc to your $HOME/bin or /usr/local/bin.

Now create new directory for your first Emgo program and copy example linker script there:
```
mkdir $HOME/firstemgo
cd $HOME/firstemgo
cp $EGPATH/src/stm32/examples/f030-demo-board/blinky/script.ld .
```

## Minimal program

Lets create minimal program in main.go file:

```go
package main

func main() {
}
```

It's actually minimal and compiles witout any problem:

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
   7452     172     104    7728    1e30 cortexm0.elf
```

It takes 7728 bytes of Flash, quite a lot for a program that does nothing. There are 8656 free bytes left to do something useful.

What about traditional *Hello, World!* code:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

Unfortunately, this time it went worse:

```
$ egc
/usr/local/arm/bin/arm-none-eabi-ld: /home/michal/P/go/src/github.com/ziutek/emgo/egpath/src/stm32/examples/f030-demo-board/blog/cortexm0.elf section `.text' will not fit in region `Flash'
/usr/local/arm/bin/arm-none-eabi-ld: region `Flash' overflowed by 10880 bytes
exit status 1
```

*Hello, World!* requires at last STM32F030x6, with its 32 KB of Flash.

Fmt is very heavy package, even a slimmed-down version in Emgo. We must forget about it. There are many applications that don't  require fancy formatted text output. Often one or more LEDs are enough. However, at the end of this post, I'll try to use strconv package to print something over UART. 

## Blinky

Our board has one LED connected between PA4 pin and VCC. This time we need a bit more code:

```go
package main

import (
	"delay"

	"stm32/hal/gpio"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
)

var led gpio.Pin

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(false)
	led = gpio.A.Pin(4)

	cfg := &gpio.Config{Mode: gpio.Out, Driver: gpio.OpenDrain}
	led.Setup(cfg)
}

func main() {
	for {
		led.Clear()
		delay.Millisec(100)
		led.Set()
		delay.Millisec(900)
	}
}
```

By convention, the init function is used to initialize the runtime and peripherals.

`system.SetupPLL(8, 1, 48/8)` configures RCC to use PLL with external 8 MHz oscilator as system clock source. PLL divider is set to 1, multipler to 48/8 = 6 which gives 48 MHz system clock.

`systick.Setup(2e6)` setups Cortex-M SYSTICK timer as system timer, which runs scheduler every 2e6 nanoseconds (500 times per second).

`gpio.A.EnableClock(false)` enables clock for GPIO port A. `false` means that this clock should be disabled in low-power mode, but this is not implemented int STM32F0 series.

`led.Setup(cfg)` setups PA4 pin as open-drain output.

`led.Clear()` sets PA4 pin low, which in open-drain configuration turns on the LED.

`led.Set()` sets PA4 to high-impedance state, which turns off the LED.

Lets compile this code:

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
   9772     172     168   10112    2780 cortexm0.elf
```

As you can see, blinky takes 2384 bytes more than minimal program. There are still 6272 bytes left for more code.

Let's see if it works:

```
$ openocd -d0 -f interface/stlink.cfg -f target/stm32f0x.cfg -c 'init; reset init; program cortexm0.elf; reset run; exit'
Open On-Chip Debugger 0.10.0+dev-00319-g8f1f912a (2018-03-07-19:20)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
debug_level: 0
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
adapter speed: 950 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0xc1000000 pc: 0x08000ef8 msp: 0x20000a20
adapter speed: 4000 kHz
adapter speed: 950 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0xc1000000 pc: 0x08000ef8 msp: 0x20000a20
adapter speed: 4000 kHz
** Programming Started **
auto erase enabled
target halted due to breakpoint, current mode: Thread 
xPSR: 0x61000000 pc: 0x2000003a msp: 0x20000a20
wrote 10240 bytes from file cortexm0.elf in 0.822888s (12.152 KiB/s)
** Programming Finished **
adapter speed: 950 kHz
```

For this article, first time in my life, I converted short video to [animated PNG](https://en.wikipedia.org/wiki/APNG) sequence. I'm impressed, goodbye YouTube and sorry IE users. See [apngasm](http://apngasm.sourceforge.net/) for more info.

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/blinky.png)

