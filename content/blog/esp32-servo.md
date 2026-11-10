---
title: "Automating My Range Hood with an ESP32 (and a Loose Pin)"
date: 2026-08-28T17:44:00+09:00
draft: false
categories: ["Hardware"]
tags: ["esp32", "esp32-c3", "servo", "iot", "home-automation", "debugging", "gemini"]
summary: "An ESP32-C3 that answered every HTTP request while the servo refused to move. Six software fixes later, the problem was a pin I never soldered."
---

Wanted my range hood to turn itself on when the air in my studio gets bad. Cooking in a small officetel, the air gets bad pretty fast, and I kept forgetting to turn the hood on. I already had a CO2/VOC sensor reporting to a server, so the missing part was something that could physically press the hood's button.

The hood has one rocker switch and that's it. No smart plug, no API. So the plan: an SG90 servo that pokes the button, driven by an ESP32-C3 SuperMini over Wi-Fi. Both the cheapest versions I could find ㅋㅋ

This is about the evening I spent debugging the servo in software when the actual problem was a pin I never soldered.

## The setup

```text
                    Wi-Fi (2.4GHz)
  server ---- GET /hood/on ----> ESP32-C3 SuperMini
                                   |  5V  ---- red    +
                                   |  GND ---- brown  |- SG90 -> pushes hood button
                                   |  GPIO --- orange +
                                   v
                                 USB power
```

Firmware is about as simple as it gets:

```cpp
WebServer server(80);
Servo hoodServo;

void pressButton() {
  hoodServo.write(135);  // swing the arm into the button
  delay(400);
  hoodServo.write(90);   // back to rest
  delay(400);
}

void handleOn() {
  pressButton();
  server.send(200, "text/plain", "Hood ON triggered");
}
```

`GET /hood/on` came back with `Hood ON triggered` every single time. The servo didn't move once.

## Six fixes that weren't

HTTP obviously worked, so I figured the problem was the motor code, and so did Gemini. Roughly what we went through, in order:

| Theory | Change | Result |
|---|---|---|
| ESP32-C3 needs PWM timers allocated explicitly | `ESP32PWM::allocateTimer(0..3)`, `setPeriodHertz(50)` | no movement |
| Detaching right after attaching cuts the pulse | attach once in `setup()`, add a boot-time test sweep | a small ticking sound |
| Sweeping to 0° jams a clone servo against its end stop | standalone sketch, sweep 90° ↔ 135°, narrower pulse widths | silence |
| Library default pulse widths don't suit clones | `attach(pin)` instead of `attach(pin, 500, 2400)` | silence |
| GPIO 4 doubles as a JTAG pin | move data to GPIO 8 | a blue light came on |
| GPIO 8 has the onboard LED, which drags the signal down | move data to GPIO 7 | blue light gone, still nothing |

Some of these are real ESP32-C3 quirks, and the ticking did mean *some* signal was getting to the servo.

The blue light is where I should've stopped. When I moved the wire to GPIO 8 the onboard LED came on and Gemini called it "great news", proof the PWM signal was working. Two answers later, the same LED was the reason the signal was too weak and the wire had to move again. Same LED, opposite explanations, both said with the same confidence... at that point we weren't diagnosing anything, just guessing.

## Wiggling the cable

What finally cracked it was me messing with the wires, not code:

> wait. it worked for a very short moment. when I kind of turned the cable here and there?? is it the loose connection?

And then:

> I am using the friction pins? I am not using soldering

I'd pushed the header pins into the SuperMini's holes and never soldered them, then plugged jumper wires onto those pins. The only thing holding each pin against the copper ring was friction.

Once you know that the whole evening makes sense. An LED or a logic input needs like a milliamp, which gets across a bad contact fine. An SG90 under load pulls a few hundred milliamps. Through a loose pin the contact acts like a resistor, voltage drops, motor gets nothing. The ticking was just barely enough current to twitch. The ticking going quiet "after changing the pulse width" was really the pin shifting. And every pin move seemed to change *something* because moving a pin disturbed the contacts.

No soldering iron, so the fix was pretty crude: cut the pins off the jumper wires, strip 2–3cm of copper, thread it through the 5V, GND and GPIO 7 holes, twist it tight against the ring. After that:

> everything is good.

Six rounds of code changes, and the fix was twisted wire. Kinda embarrassing honestly.

## Wi-Fi that takes like 100 tries

The other problem was connecting at all. The board needed dozens of attempts and the serial log was full of this:

```text
E (8912) wifi:CCMP replay detected: A1=... A2=... PN=2, RSC=3 seq=464
```

This made it usable, not good:

```cpp
WiFi.mode(WIFI_STA);
WiFi.setSleep(false);                 // keep the radio awake (the usual fix for the CCMP replay drops)
WiFi.setTxPower(WIFI_POWER_8_5dBm);   // lower TX power = smaller current spikes on a weak power path
WiFi.disconnect(true);
WiFi.begin(SSID, PASSWORD);
```

Two non-code things helped too. The C3 only does 2.4GHz b/g/n, so I set the router's 2.4GHz band to g only. And the SuperMini's tiny ceramic antenna is really directional, so which way the board faces actually matters.

At one point I asked for a static IP and the board came up on a different DHCP address. Gemini said the router "rejected" the static IP. That's not a thing — a static IP never asks the router for anything. More likely my own code: `WiFi.config()` ran before a `WiFi.disconnect(true)` I added later, and the disconnect reset it. Just dropped the static config and used DHCP.

Still not fixed though. It connects, but pings from about 3 meters away still fail. Gemini blamed that on the twisted wires too — once the loose pin turned out to be real, every symptom after that got blamed on the wiring. Might be right, haven't checked.

## Mounting it, and the light switch problem

With the motor working, two problems showed up that have nothing to do with code.

Mounting. The servo has to push a lever and stay put while pushing, otherwise it just shoves itself backwards. And there's a gap between anything I could attach it to and the button. Options I'm thinking about: zip ties through the servo's mounting ears and the holes in the hood bracket, automotive double-sided tape (after degreasing, hoods are covered in cooking oil film), epoxy putty as a custom pillar, or just a longer arm out of a stiff stick so the motor can sit further back.

Power. In my officetel, turning off the kitchen light also cuts power to the hood. The automation has to work while I'm asleep with the light off.

Thought about covering the hood lights with a black sheet so I could leave that circuit on. Bad idea, hood lights can get hot and that's a fire risk. Also thought about rewiring the hood to another outlet, but that's 220V mains in a rented place, so no. Safer options: a lot of hoods in Korean studios plug into a hidden outlet in the cabinet above, which could go onto an always-on outlet with a proper power strip. Or maybe the wall panel lets outlets stay on with the lights off.

Still need to open that cabinet and look.

## Where it's at

Servo moves now. Not something I'd leave running unattended yet:

- The wiring is twisted copper, not solder. It's going to work loose from pushing a button every day.
- `/hood/on` and `/hood/off` do the same thing: press the button once. It's a toggle and the ESP32 has no idea if the hood is running, so after one missed press the whole thing is inverted forever. Needs some kind of feedback, maybe a current sensor on the hood's power cord.
- The endpoints are unauthenticated GETs, so anyone on my Wi-Fi can turn on my range hood lol. Needs at least a shared token.
- `pressButton()` sits in `delay()` for 800ms inside the request handler, so the web server is blocked during every press.
- Servo and radio share USB power. A separate 5V supply for the servo (common ground) would stop motor spikes from starving the radio. Might be part of the Wi-Fi problem too.

## Next time

Test the motor alone first. A standalone sweep sketch with no Wi-Fi should've been step one, not step five. Check every joint with a multimeter before touching code — would've found this in like a minute. And change one thing per attempt. Several rows in that table changed a pin *and* the pulse widths at the same time, and with an intermittent contact any change can look like it worked.

Gemini is fast at listing possible causes, but it can't feel a loose pin. The most useful debugging step all evening was me wiggling a cable.

Also, buy a soldering iron.
