---
layout: post
title: Go on very small hardware (Part 2)
tags: mcu go emgo
permalink: drafts/1
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/f030-demo-board/board.jpg)]({{ site.baseur }}/2018/03/30/go_on_very_small_hardware2.html)

At the end of the [first part]({{ site.baseur }}/2018/03/30/go_on_very_small_hardware.html) of this article I promised to write about *interfaces*. I don't want to write here a complete or even brief lecture about the interfaces. Instead, I'll show a simple example how to define and use an interface, and then, how to take advantage of ubiquitous *io.Writer* interfece. At the end of this article I'll show a couple of examples that will push our little board to its borders.

<!--more-->

Interfeces are a crucial part of Go language. If you want to learn more about them, I suggest to read [Effective Go](https://golang.org/doc/effective_go.html#interfaces) and [Russ Cox article](https://research.swtch.com/interfaces).

## Concurrent Blinky - revisited

When you read the code of previous examples you probably noticed a counterintuitive way to turn the LED on or off. The *Set* method was used to turn the LED off and the *Clear* method was used to turn the LED on. This is due to driving LEDs in open-drain configuration. What we can do to make the code less confusing? Let's define the *LED* type with *On* and *Off* methods:

```go
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

Now we can simply call `led.On()` and `led.Off()` which no longer raises any doubts.

In all previous examples I tried to use the same open-drain configuration to don't complicate the code. But in the last example, it would be easier for me to connect the thrid LED between GND and PA3 pins and configure PA3 in push-pull mode. The next example will use a LED connected this way.

But our new *LED* type doesn't support the push-pull configuration. In fact, we should call it *OpenDrainLED* and define another *PushPullLED* type:

```go
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

Note, that both types have the same methods that work the same. It would be nice if the code that operates on LEDs could use both types, without paying attention to which one it uses at the moment. The *interface type* comes to help:

```go
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

We've defined *LED* interface that has two methods: *On* and *Off*. The *PushPullLED* and *OpenDrainLED* types represents two ways of driving LEDs. We also defined two *Make***LED* functions which act as constructors. Both types implement the *LED* interface, so the values of these types can be assigned to the variables of type *LED*: 

```go
led1 = MakeOpenDrainLED(gpio.A.Pin(4))
led2 = MakePushPullLED(gpio.A.Pin(3))
```

In this case the assignability is checked at compile time. After the assignment the *led1* variable contains `OpenDrainLED{gpio.A.Pin(4)}` value and a pointer to the method set of the *OpenDrainLED* type. The `led1.On()` call roughly corresponds to the following C code: `led1.methods->On(led1.value)`. As you can see, this is quite inexpensive abstraction if only consider the function call overhead.

But any assigment to an interface causes to include a lot of information about the assigned type:

```
$ egc
$ arm-none-eabi-size cortexm0.elf 
   text    data     bss     dec     hex filename
  10356     196     212   10764    2a0c cortexm0.elf
```

If we don't use [reflection](https://blog.golang.org/laws-of-reflection) we can save some bytes by avoid to include the names of types and struct fields:

```
$ egc -nf -nt
$ arm-none-eabi-size cortexm0.elf 
   text    data     bss     dec     hex filename
  10312     196     212   10720    29e0 cortexm0.elf
```

The resulted binary still contains some necessary information about types and full information about all exported methods (with names). These information are need for checking assignability at runtime, mainly when you assign one value stored in the interface variable to an other variable.

We can also remove type and field names from imported packages by recompile them all:

```
$ egc
$ cd $HOME/emgo
$ ./clean.sh
$ cd -
$ egc -nf -nt
$ arm-none-eabi-size cortexm0.elf 
   text    data     bss     dec     hex filename
  10272     196     212   10680    29b8 cortexm0.elf
```

Lets see does it work as expected. This time we'll use [st-flash](https://github.com/texane/stlink):

```
$ arm-none-eabi-objcopy -O binary cortexm0.elf cortexm0.bin
$ st-flash write cortexm0.bin 0x8000000
st-flash 1.4.0-33-gd76e3c7
2018-04-10T22:04:34 INFO usb.c: -- exit_dfu_mode
2018-04-10T22:04:34 INFO common.c: Loading device parameters....
2018-04-10T22:04:34 INFO common.c: Device connected is: F0 small device, id 0x10006444
2018-04-10T22:04:34 INFO common.c: SRAM size: 0x1000 bytes (4 KiB), Flash: 0x4000 bytes (16 KiB) in pages of 1024 bytes
2018-04-10T22:04:34 INFO common.c: Attempting to write 10468 (0x28e4) bytes to stm32 address: 134217728 (0x8000000)
Flash page at addr: 0x08002800 erased
2018-04-10T22:04:34 INFO common.c: Finished erasing 11 pages of 1024 (0x400) bytes
2018-04-10T22:04:34 INFO common.c: Starting Flash write for VL/F0/F3/F1_XL core id
2018-04-10T22:04:34 INFO flash_loader.c: Successfully loaded flash loader in sram
 11/11 pages written
2018-04-10T22:04:35 INFO common.c: Starting verification of write complete
2018-04-10T22:04:35 INFO common.c: Flash written and verified! jolly good!
```

I havn't connected the NRST signal to the programmer so the *---reset* option can't be used and the reset button must be pressed after programming.

![Interfaces]({{site.baseur}}/images/mcu/f030-demo-board/interfaces.png)

It seems that the *st-flash* works a bit unreliably with this board (often requires reseting the ST-LINK dongle). Additionally, the current version doesn't issue the reset command over SWD (uses only NRST signal). It isn't a big problem, especially that the software reset isn't realiable, however it introduces inconvenience. For this programmer--board pair the *OpenOCD* works much better.

## UART

UART (Universal Aynchronous Receiver-Transmitter) is still one of the most important peripherals of today's microcontrollers. Its advantage is unique combination of the following properties:

- relatively high speed,
- only two signal lines (even one in case of half-duplex communication),
- symmetry of roles,
- synchronous in-band signaling about new data (start bit),
- accurate timing inside transmitted word.

This causes that UART, originally intedned to transmit asynchronous messages consisting of 7--9 bit words, is also used to efficiently implement various other phisical protocols such as used by [WS28xx LEDs](http://www.world-semi.com/products/index.html) or [1-wire](https://pl.wikipedia.org/wiki/1-Wire) devices.

However, for now, we will use the UART in its usual role - as text output from our program:

```go
package main

import (
	"rtos"

	"stm32/hal/dma"
	"stm32/hal/gpio"
	"stm32/hal/irq"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
	"stm32/hal/usart"
)

var tts *usart.Driver

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(true)
	port, tx := gpio.A, gpio.Pin9

	port.Setup(tx, &gpio.Config{Mode: gpio.Alt})
	port.SetAltFunc(tx, gpio.USART1_AF1)
	d := dma.DMA1
	d.EnableClock(true)
	tts = usart.NewDriver(usart.USART1, d.Channel(2, 0), nil, nil)
	tts.Periph().EnableClock(true)
	tts.Periph().SetBaudRate(115200)
	tts.Periph().Enable()
	tts.EnableTx()

	rtos.IRQ(irq.USART1).Enable()
	rtos.IRQ(irq.DMA1_Channel2_3).Enable()
}

func main() {
	tts.WriteString("Hello, World!\r\n")
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
}
```

You can find this code slightly complicated because in addtion to USART peripheral you see DMA and interrupts. But for now there is no simple polling UART driver in STM32 HAL (it would be probably useful sometimes). The *usart.Driver* is efficient driver that uses DMA and interrupts to ofload the CPU.

STM32 USART peripheral provides traditional UART and its synchronous version. I this article I use the UART word rather as the name of phisical protocol and USART as the name of STM32 peripheral.

To use USART perpheral we must connect its signals to selected pins of GPIO:

```go
port.Setup(tx, &gpio.Config{Mode: gpio.Alt})
port.SetAltFunc(tx, gpio.USART1_AF1)
```

We configuring the *usart.Driver* in Tx only mode (rxdma and rxbuf are set to nil):

```go
tts = usart.NewDriver(usart.USART1, d.Channel(2, 0), nil, nil)
```


