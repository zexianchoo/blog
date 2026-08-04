---
title: "ESP32 Commute Display"
date: 2026-07-18
summary: "i got tired of checking three transit apps every morning"
tags: ["ESP32", "E-Ink", "C++", "APIs", "Hardware"]
author: "Sean"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: true
description: "using an esp32, onebusaway and google routes to tell me when to leave home"
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

My morning commute has several legs, which means checking one transit app is apparently not enough.

I need to know when to leave home, which train I should catch, and whether that train actually lines up with the final bus. Doing this every morning on my phone was not difficult, but it was annoying enough that I decided to spend significantly more time building a dedicated device instead.

Very reasonable behavior.

The result is an ESP32-S3 commute display using a 2.9-inch black / white / red e-paper panel.

# What it shows

The screen gives me the next three complete commute options:

1. when to leave home
2. which Link train to catch
3. the matching bus departure
4. the last successful sync time

I show the important times in red because the display only has three colors and I might as well use the exciting one.

The final bus is the main constraint, so the program starts from its upcoming departures and works backward through the train itinerary to calculate a leave-home time.

I also add a five-minute bus-catching buffer. Google may think a two-minute transfer is technically possible, but Google does not have to watch me sprint up the stairs.

# APIs

I use two data sources:

- OneBusAway for scheduled or live bus predictions
- Google Routes for the transit itinerary

Google's response can be pretty large compared to what an ESP32 actually needs, so I use a field mask and only parse the relevant itinerary data.

The code also checks that the response contains the expected rail segment. A route arriving at roughly the correct time through some completely different combination of buses is not useful just because the timestamp matches.

This was one of those places where microcontrollers make waste very obvious. On a server, downloading a massive JSON response and ignoring 99% of it is merely ugly. On an ESP32, it can just run out of memory and die.

# Refresh schedule / Deep sleep

The display refreshes every ten minutes during my morning travel window, Monday through Saturday.

Outside of that schedule, it shows a calm off-hours page and deep-sleeps until the next active period.

This saves power, reduces API calls, and avoids flashing the e-paper screen all night for absolutely no reason.

Errors behave slightly differently. If an API request fails, the device keeps USB debugging available so I can see what happened, then restarts and retries later.

There is also a test mode which fetches once and stays awake. This was essential because trying to debug a microcontroller which immediately goes to sleep is a really good way to lose your mind.

# Hardware

The hardware is:

1. ESP32-S3
2. WeAct 2.9-inch black / white / red e-paper display
3. SSD1680 controller
4. 128x296 resolution

E-paper is perfect for this because the screen keeps the last successful result visible without constant power. It also does not glow or animate, which forced me to keep the information simple.

The repository contains placeholders for Wi-Fi, API keys, and the home address. None of those values are committed.

One security compromise is still documented: the current hobby build disables TLS certificate verification for the API calls. The better version should install and maintain the correct root certificates.

It is not ideal, but hiding the problem inside the code would not make it safer. At least it is very clearly listed as something I need to fix before pretending this is a hardened device.

# Conclusion

Yes, I could look at my phone.

But the nice part about this screen is that it is ambient. I can glance at it and immediately know whether I should leave now or whether I have time to continue drinking coffee.

I think smart displays are usually filled with way too much information. The actual work in this project happens behind the scenes so the final screen can show less.

Three commute options. One decision. No app to open.

Honestly, this might be one of the most practically useful tiny things I have built.
