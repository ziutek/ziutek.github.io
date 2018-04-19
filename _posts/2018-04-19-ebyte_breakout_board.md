---
layout: post
title: Breakout board for Ebyte module
tags: mcu nrf5
---

[![Ebyte breakout board]({{site.baseur}}/images/mcu/ebyte/ebyte-breakout.jpg)]({{site.baseur}}/2018/04/19/ebyte_breakout_board.html)

This is my hand made breakout board for Ebyte [E73-2G4M04S](http://www.cdebyte.com/en/product-view-news.aspx?id=243) module. 

<!--more-->

Don't try to do it yourself, its not worth the time spent on it.

I needed something to test purchased E73-2G4M04S modules. I attached the module with double-sided tape and started soldering with the idea to expose only few pins. I ended with something like this. It will take much less time, less frazzle of nerves and less flux in your lungs if you design a PCB and send it to some factory.

However, the pin layout is worth attention:

[![Ebyte breakout pins]({{site.baseur}}/images/mcu/ebyte/ebyte-bpins.jpg)]

It makes pin finding easier thanks to double GND pins between Px4-Px5 and Px9-Px0. It also exposes many GND and 3.3V pins for direct connections. Its SWD connector is compatible with ST-LINK V2 programmer (straight connections). At the same time it stays breadboard friendly:

[![Ebyte breadboard]({{site.baseur}}/images/mcu/ebyte/ebyte-breadboard.jpg)]

The pins with yellow jumper allows to connect microamp meter to mesure the module power consumption. 