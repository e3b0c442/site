+++
Description = "BBiQ"
title = "BBiQ"
date = "2018-07-15"
markup = "markdown"
+++

I found a new project earlier this summer.

I had lost my old barbecue thermometer in our move last year. My mother- and father-in-law bought me a new Bluetooth one for my birthday this year. Unfortunately, it had two big drawbacks:

* It could only handle two probe jacks.
* Bluetooth range, while nominally 100 meters, depends on the transmit power and it was clearly obvious that in order to save power that transmit power had been reduced on this device; it would not cover the house.

After considering the problem, I decided that WiFi would be a better connection mechanism to use, because:
* My primary router has line-of-sight to the grill
* it would afford me the ability to run an errand while still being able to monitor the status of my cook (with somebody at home keeping an eye on things... come on!).

So, I started looking around, and found out that there wasn't really much on the market that had 4 probes AND WiFi, and what there was, was quite extensive.

Also in my searching, I found [HeaterMeter](https://store.heatermeter.com), and became intrigued. The price was still a bit high, and it included capabilities that I just didn't need (like a fan controller, for example). However, I thought -- what's keeping me from building my own similar device? This way, I could build it exactly how I wanted, and have complete control over everything. In particular, I could make sure I was using high-quality probes, like [these high-quality probes from ThermoWorks, the same guys that make the ThermaPen](https://www.thermoworks.com/Handheld-Probes/Probes/Pro-Series). 

So I started thinking, and planning, and prototyping. This weekend, I hit two big milestones: I finished the code for everything local to the device (buttons, alarm, display), and I had a successful test of my calibration, with the thermometer reading within 1°F compared to a reference at multiple temperatures.

![BBiQ proto-prototype](/posts/bbiq/bbiq_proto.png)

For this proto-prototype, I'm building with an old [Arduino Uno](https://store.arduino.cc/arduino-uno-rev3) that I've had for awhile, then connecting it to a [Raspberry Pi B+](https://www.raspberrypi.org/products/raspberry-pi-1-model-b-plus/) with an [Edimax N150](https://www.edimax.com/edimax/merchandise/merchandise_detail/data/edimax/in/wireless_adapters_n150/ew-7811un/) mini WiFi adapter. I plan to use the [Blynk](https://blynk.cc) service for the "cloud" functionality, including notifications and history graphs. 

Unfortunately, this means the prototype will be huge and power-hungry, not ideal traits. While I had originally planned on using a Raspberry Pi Zero W in the "final" version, after doing more research I discovered the [ESP8266](https://www.esp8266.com) WiFi microcontroller, and began to hatch a plan.

Once I'm satisfied with the functionality of this initial prototype, I'm going to build a "final" prototype:
* Based on a [Teensy LC](https://www.pjrc.com/teensy/teensyLC.html) dev board (itself based on an ARM Cortex M0+ microcontroller)
* WiFi provided by an ESP8266 "backpack"
* 128x32 monochrome OLED display replacing the large backlight character LCD
* Powered by 2xAA batteries

I'm confident that by carefully managing the power states on both the Teensy and the ESP8266 that I can keep the average power draw under 100mA, which would give 20+ hours of battery life on one pair of rechargeable AAs.

Assuming everything goes well and I feel the urge, I could build my own board based on the Freescale microcontroller that Teensy LC uses... but if I can already get the battery life I'm looking for, what's the point? :)

[Source and schematics](https://github.com/e3b0c442/BBiQ), if you want to follow my progress. Check the prototype branch if there's nothing on master.