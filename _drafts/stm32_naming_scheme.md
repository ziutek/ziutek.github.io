---
layout: post
title: STM32 naming scheme
tags: mcu go emgo
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/stm32f030f4p6.jpg)]({{site.baseur}}/2018/04/14/go_on_very_small_hardware4.html)

How to decipher the name on STM32 MCU? The STM32F030F4P6 will be our templeate:

<!--more-->

STM32 | Family            | The family of 32-bit MCUs with ARM Cortex-M CPU
F     | Type              | F: mainstream, L: low power, H: high performance, W: wireless.
0     | CPU               | 0: M0, 1: M3, 2: M3, 3: M3, 4: M4, 7: M7
30    | Line              | speed, peripherals, silicon process
F     | Numbe of pins     | F: 20, G: 28, K: 32, T: 36, S: 44, C: 48, R: 64,66, V: 100, Z: 144, I: 176
4     | Flash KiB         | 4: 16, 6: 32, 8: 64, B: 128, C: 256, D: 384, E: 512, F: 768, G: 1024, H: 1536, I: 2048
P     | Package           | P: TSOOP, H: BGA, U: VFQFPN, T: LQFP, Y: WLCSP
6     | Temperature range | : 6: -40..85°C, 7: -40..105°C

For example the [STM32F411CEY6](http://www.st.com/en/microcontrollers/stm32f411ce.html) you can find in the [EMW3165](http://en.mxchip.com/product/wifi_product/38) is:

F4 | mainsterm MCU with 32-bit ARM Cortex-M4F core
11 | 100 MHz core, 3x USARTs, 3x I²C, 1x SDIO, 1x USB 2.0 OTG, 5x I²S, 12-bit ADC 2.4 MSPS, 11 timers
C  | 48 pins
E  | 512 KiB of Flash
Y  | WLCSP package
6  | temperature range from -40 to 85 °C