---
layout: post_with_comments
title:  "Media Server with Jellyfin with remote power up"
description: Power Up Jellyfin PC remotely with Home Assistant
date:   2023-11-30 16:28:00 +0200
---

![media_center]({{ site.baseurl }}/assets/images/jellyfin-esp-power-up/me_and_my_media_center.jpeg "media center")

If you're like me, with a desktop PC that isn't always online but houses your media server powered by [Jellyfin](https://jellyfin.org/), you've probably faced the challenge of waking it up quickly when needed. Jellyfin is an open-source media server that allows you to manage and stream your media collection effortlessly.

I initially experimented with the traditional magic packet approach, only to find that my motherboard didn't fully support it, and reliability was a bit of an issue.

Enter the [ESP32](https://www.espressif.com/en/products/socs/esp32), a versatile microcontroller I've used in other projects, and [ESPHOME](https://esphome.io/). The ESP32 is a powerful and flexible microcontroller widely used in the maker community. ESPHOME is a framework for creating custom firmwares for ESP8266/ESP32 boards. After stumbling upon [this project from Erriez](https://github.com/Erriez/ESPHomePCPowerControlHomeAssistant), I discovered a solution where an ESP32 could intercept connections for the power and reset buttons of the desktop PC.

![prototype]({{ site.baseurl }}/assets/images/jellyfin-esp-power-up/IMG_3374.jpg "prototype")

My initial prototype, though in a rather "temporary fashion," successfully allowed me to power up the PC. You can find the details on how to build the circuit in the GitHub project page.


![HA_Device]({{ site.baseurl }}/assets/images/jellyfin-esp-power-up/Captura.PNG "HA Device")

This is how the device turns out in HA.

However, reading the reset pin to determine the computer's status proved to be a bit trickier than expected. Undeterred, I sought an alternative solution and found a [Home Assistant companion app for Windows](https://lab02-research.org/hassagent/). This app, coupled with Mosquitto MQTT, allowed me to create a sensor in Home Assistant that could accurately report whether the computer was sleeping, off, or available. This way, I could make informed decisions about when to press the power button remotely before diving into a Jellyfin binge.

As I explored further, I discovered additional possibilities for improvement. Leveraging Google TV integration, which provides a sensor indicating the currently opened application, I could refine my setup. By identifying when Jellyfin was in use, I could automatically check if the computer needed powering up, avoided the need to wake up the computer myself.

If you're facing a similar challenge or love tinkering with home automation, give this approach a try. 

ESP32 ftw

chao!