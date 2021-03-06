---
layout: post
title: Copy memory on STM32
tags: stm32 mcu go emgo
---

[![STM32F030F4P6]({{site.baseur}}/images/mcu/stm32dma.jpg)]({{site.baseur}}/2018/05/18/copy_memory_on_stm32.html)

Interesting results of different ways of copying memory:

<!--more-->

[STM32F411RE](https://github.com/ziutek/emgo/blob/master/egpath/src/stm32/examples/nucleo-f411re/dma/main.go) 96 MHz

```
Initialize src                          27422 kB/s
for i := range src { dst[i] = src[i] }  17330 kB/s
copy(dst, src)                          38367 kB/s
DMA                                    121004 kB/s
DMA FT1                                122148 kB/s
DMA FT2                                122086 kB/s
DMA FT3                                121303 kB/s
DMA FT4                                 98422 kB/s
DMA FT4 PB4 MB4                        122543 kB/s
```

[STM32F407VG](https://github.com/ziutek/emgo/blob/master/egpath/src/stm32/examples/f4-discovery/dma/main.go) 168 MHz

```
Initialize src                          47989 kB/s
for i := range src { dst[i] = src[i] }  29798 kB/s
copy(dst, src)                          64153 kB/s
DMA                                    162141 kB/s
DMA FT1                                162325 kB/s
DMA FT2                                161654 kB/s
DMA FT3                                161897 kB/s
DMA FT4                                138937 kB/s
DMA FT4 PB4 MB4                        187397 kB/s
```

The *src* and *dst* are both of type *[]uint32*.

`Initialize src` is simple:

```go
for i := range src {
	src[i] = uint32(i)
}
```

`for i := range src { dst[i] = src[i] }` doesn't require explanation.

`copy(dst, src)` uses builtin *copy* function that is implemented in [assembly](https://github.com/ziutek/emgo/blob/master/egroot/src/internal/copy-cortexm.s).

`DMA` uses DMA peripheral in direct mode (FIFO not used).

`DMA FT1` uses DMA with internal 4-word FIFO enabled. Memory write is trigered immediately when FIFO is filled in 1/4 (one word).

`DMA FT2` also uses DMA in FIFO mode. Memory write triggered when FIFO filled in 2/4 (two words).

`DMA FT3` - memory write triggered at 3/4 (three words).

`DMA FT4` - memory write triggered when FIFO is full.

`DMA FT4 PB4 MB4` uses DMA in 4-word burst mode.

As you can see DMA is ca. three times faster than *copy* (but only *copy* handles correctly any overlapping *src* and *dst*).

You can also see that it's not good to wait for write until the FIFO is full.

The best performance is achieved using AHB burst mode. There is a warning about burst transfers:

>> The burst configuration has to be selected in order to respect the AHB protocol, where bursts must not cross the 1 KB address boundary because the minimum address space that can be allocated to a single slave is 1 KB. This means that the 1 KB address boundary should not be crossed by a burst block transfer, otherwise an AHB error would be generated, that is not reported by the DMA registers.

I don't quite understand this warning but despite the deliberate attempts to generate this error it doesn't occur in case of memory to memory transfer.