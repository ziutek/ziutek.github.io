---
layout: post
title: Decipher the STM32 naming
tags: mcu stm32
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/stm32f030f4p6.jpg)]({{site.baseur}}2018/05/07/stm32_naming_scheme.html)

How to decipher the name of a STM32 MCU? The above [STM32F030F4P6](http://www.st.com/en/microcontrollers/stm32f030f4.html) will serve us as a model:

<!--more-->

STM32 | Family            | The family of 32-bit MCUs with ARM Cortex-M core
F     | Type              | F: mainstream, L: low power, H: high performance, W: wireless
0     | Core              | 0: M0, 1: M3, 2: M3, 3: M4, 4: M4, 7: M7
30    | Line              | speed, peripherals, silicon process
F     | Numbe of pins     | F: 20, G: 28, K: 32, T: 36, S: 44, C: 48, R: 64,66, V: 100, Z: 144, I: 176
4     | Flash KiB         | 4: 16, 6: 32, 8: 64, B: 128, C: 256, D: 384, E: 512, F: 768, G: 1024, H: 1536, I: 2048
P     | Package           | P: TSOOP, H: BGA, U: VFQFPN, T: LQFP, Y: WLCSP
6     | Temperature range | 6: -40..85°C, 7: -40..105°C

The 8 and B in the Flash size field are often difficult to distinguish. Maybe that's why all known to me STM32F103C**8**T6 have in fact 128 KiB of Flash ;-)

#### Two more examples

The [STM32F103C8T6](http://www.st.com/en/microcontrollers/stm32f103c8.html) you can find on the popular [Blue Pill](https://jeelabs.org/article/1649a/):

F1 | Mainsterm MCU with 32-bit ARM Cortex-M3 core
03 | 72 MHz CPU, up to 20 KiB RAM, 3x USART, 2x SPI/I²S, 2x I²C, 1x USB, 1x CAN, 2x ADC, 7 timers, 7-channel DMA
C  | 48 pins
8  | 64 KiB of Flash
6  | Temperature range from -40 to 85 °C


The [STM32F411CEY6](http://www.st.com/en/microcontrollers/stm32f411ce.html) you can find under the cap of the [EMW3165](http://en.mxchip.com/product/wifi_product/38):

F4 | Mainsterm MCU with 32-bit ARM Cortex-M4F core
11 | 100 MHz CPU, up to 128 KiB RAM, 3x USART, 5x SPI/I²S, 3x I²C, 1x SDIO, 1x USB OTG, 1x ADC, 11 timers, 16-stream DMA
C  | 48 pins
E  | 512 KiB of Flash
Y  | WLCSP package
6  | Temperature range from -40 to 85 °C