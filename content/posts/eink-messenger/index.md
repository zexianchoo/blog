---
title: "E-Ink Messenger"
date: 2026-08-03
summary: "sending cute little drawings through the most complicated route possible"
tags: ["ESP32", "SwiftUI", "Golang", "Bluetooth", "E-Ink"]
author: "Sean"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: true
description: "an iphone, a go server, apns, bluetooth and an esp32 walk into a bar"
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

I wanted to make a small display where someone could send text and drawings from their phone and have it appear beside the other person.

That was the cute and simple idea.

The actual implementation somehow became a Go server, a SwiftUI app, APNs, Core Bluetooth, ESP32 firmware, two different display backends, a browser client, image rendering, recovery codes, widgets, and a delivery state machine.

So yeah, **E-Ink Messenger** ended up being a lot more complicated than I expected.

# Technical Overview - Architecture

The full message path is:

```text
sender iPhone/browser
        |
        v
Go server + SQLite
        |
       APNs
        |
        v
recipient iPhone
        |
       BLE
        |
        v
ESP32-S3 -> LCD / e-paper display
```

The recipient iPhone acts as the relay between the server and the display.

I could have made the ESP32 poll a public API directly, but then the display would need internet credentials, more complicated auth, and enough logic to understand the entire application. Keeping the ESP32 behind BLE felt much safer and also made the firmware smaller.

The server stores the message, sends a content-free APNs notification, and the recipient phone renders the correct frame for its paired display. The iPhone then transfers the frame through BLE to the ESP32.

Sounds simple enough when written in one paragraph.

It was not.

# Delivery receipts - “sent” means nothing

One of the most annoying bugs was that a message could say `push_sent` forever and only appear after pressing **Retry pending message**.

At first, it was tempting to assume BLE was broken. However, the manual retry worked, which proved that the payload, the phone, the Bluetooth link, and the display were actually capable of completing the transfer.

The real unreliable part was waking the iPhone in the background.

I ended up splitting delivery into several explicit states:

```text
push_sent -> phone_received -> frame_committed -> displayed
```

Each state means something different:

1. Apple accepted the background push
2. The iPhone actually received it
3. The complete frame was transferred
4. The display acknowledged and showed it

APNs background pushes are best-effort. Low Power Mode can disable Background App Refresh, iOS may decide not to wake the app, and having the phone physically beside the display does not mean the process is running.

This kinda sucked because I wanted the display to magically update unattended. Unfortunately, I cannot bully iOS into making guarantees it does not make.

The recovery path is therefore very explicit: open the app, let BLE reconnect, or retry the pending message. At least the UI can now explain exactly where the message stopped instead of showing one meaningless checkmark.

# Display backends

The original target was a 2.9-inch black / white / red e-paper display.

Later, I also added a 2.8-inch ILI9341 color LCD at 320x240. I first tested the screen using a completely separate breadboard sketch because I did not want to debug my renderer, BLE protocol, firmware, and wiring simultaneously.

This was also where I nearly connected to a pin which was not actually ground. Thankfully, I checked the printed labels and wiring before powering it, which is probably the only reason this section is not about how I killed an ESP32.

## Rendering

The displays behave very differently.

The e-paper panel only has black, white, and red, refreshes slowly, and keeps its image without continuous power. The ILI9341 is a normal color LCD, so I render a dithered 64-color RGB222 palette while preserving the named drawing colors.

I kept a common document model above both backends. Text, strokes, and photos are composed once, and the renderer converts the result into the exact frame required by the paired display.

Photo metadata is stripped during conversion. Images are fitted with white letterboxing instead of randomly cropping out someone's face, which happened enough times that I had to fix it properly.

# The iOS editor rabbit hole

The first editor was basically just a drawing canvas.

Then I added:

- zooming
- camera and photo imports
- multiple movable and resizable photos
- multi-stroke selection
- message history
- full-screen saved message previews
- favorite drawings
- a randomized favorite carousel for the display
- recent-message and streak widgets

The hardest part of these features was not usually rendering them. It was keeping the exact same document stable while the user zoomed, moved things, imported another image, reopened history, or rendered it for a completely different display.

I think making visual editors is one of those things that seems easy until you have more than one coordinate system. Then everything starts drifting for reasons which feel personally targeted.

# Web access and recovery

I also made a linked browser composer.

The phone creates a one-use link which expires after a short time. Redeeming it creates a separate browser session with an expiring secure cookie. The browser never receives the iPhone's native bearer token, and the browser session can be revoked independently.

This was important because sharing one credential between every client is convenient right until a laptop or phone gets lost.

Phone recovery works similarly. An active partner can create a one-use replacement code which rotates only the lost phone's session while keeping the shared space and message history.

There is also a local reset when the server is down, although the app very clearly warns that it cannot revoke the old server session until connectivity comes back.

Honestly, recovery flows are boring to demo but they are what makes the app feel like something I can actually keep using.

# Display sleep

The firmware sleeps the display after 30 minutes and resets the timer whenever a valid message or retry completes.

E-paper is amazing here because the image remains on screen while the controller hibernates. The LCD controller can also enter sleep, but there is one funny hardware problem: if the backlight is wired directly to 3.3V, it stays lit anyway.

Software can put the controller to sleep. Software cannot magically rewire the backlight.

To make the LCD fully dark, I still need a safe transistor / MOSFET controlled backlight circuit and a proper physical observation test. I am leaving that boundary honest instead of claiming that a passing firmware test proves the room is dark.

# Conclusion

I originally thought the difficult part would be rendering a drawing on an ESP32 display.

It turns out the difficult part was knowing whether the drawing actually reached the other person.

APNs accepting a push does not mean the phone received it. The phone receiving it does not mean BLE finished. BLE finishing does not mean the display committed the frame.

I had to make every one of those boundaries visible before the system started feeling reliable.

This is probably the most complicated route I could have chosen to send a cute little drawing to someone, but seeing the exact same picture appear on a tiny physical screen is still incredibly satisfying.

Worth it hehe.
