---
title: "Rover: An ESP32 DevKit V1 Omni-Wheel Platform"
date: 2026-10-11 09:00:00 -0400
categories: [Rover, ESP32, Projects]
tags: [embedded, esp32, esp-idf, freertos, robotics, motors]
---

Rover is a tiny remote controlled car that I built from a metal frame, an ESP32 DevKit V1, and a TT Motor / mecanum wheel drivetrain. It was a fun project that allowed me to get more familiar with ESP32 and FreeRTOS, resulting in a base platform that can be used to test new controls and sensors.

![Rover, a four-wheel omni-drive embedded project](/assets/img/rover/rover_landscape.jpg){:style="width:80%; display:block; margin-left:auto; margin-right:auto"}
*Rover is a small platform for experimenting with embedded controls and hardware.*

Rover uses Espressif's ESP-IDF rather than Arduino's higher-level APIs. This was an opportunity for me to gain familiarity with FreeRTOS (I've used Zephyr and was curious how they stack up). I also wanted Rover to be as portable as possible so I had a platform for playing with new microcontrollers in the future. Arduino's ESP32 framework also uses FreeRTOS underneath; the difference is the level of abstraction. The goal was not simply to make a car move, but to use low-level embedded software tools and build a platform for further exploration.

The source is available in the [Rover repository](https://github.com/JFolkens/esp32_peripheral_debug/tree/rover_v1).
There are in-depth posts on
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-modular-embedded-design-the-web-peripheral-interface.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-local-gtest-build.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-binary-assets-on-esp32.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>

## Hardware

Rover uses two L298N dual-motor control boards to control four wheels independently (a necessity for omni-wheels). Each motor has separate Forward, Reverse, and Speed (PWM) control signals. The ESP32 DevKit V1 also has an onboard blue LED.

![Rover ESP32 DevKit V1 pinout and L298N motor connections](/assets/img/diagrams/rover_pinout.png){:style="width:80%; display:block; margin-left:auto; margin-right:auto"}
*Rover GPIO pinout.*

Rover is powered from a battery pack mounted on the bottom of the chassis. This keeps the pack low and centered, while still being easy to remove. The frame is a basic aluminum template off [Amazon](https://www.amazon.com/DWWTKL-Omnidirectional-Educational-Raspberry-Unassembled/dp/B0CBRBCFMZ) which can be drilled through with essentially any drill bit.

<div style="display: grid; grid-template-columns: 30% 30%; gap: 10%; justify-content: center; padding-bottom: 20px;">
  <figure style="margin: 0; text-align: center;">
    <img src="/assets/img/rover/rover_bottom.jpg" width="100%" alt="Rover upside down showing battery pack" />
    <em>Rover upside down</em>
  </figure>
  <figure style="margin: 0; text-align: center;">
    <img src="/assets/img/rover/rover_bottom_battery_cover_off.jpg" width="100%" alt="Rover upside down with battery cover removed">
    <em>Without battery cover</em>
  </figure>
</div>

The battery supplies power to the motors via a strip of perfboard that I call my high-voltage rail (because I like to feel fancy). It also provides input to DC buck converter that provides a smooth 5V for the ESP32 board.

ESP32 DevKit V1 has ports to accept 5V or 3.3V power. The MP1584 Buck Converter is tuned via a small screw on the top; I set it to 5V and connected it to the ESP32 perf board via JST connector.

<div style="display: grid; grid-template-columns: 40% 40%; gap: 10%; justify-content: center; padding-bottom: 20px;">
  <figure style="margin: 0; text-align: center;">
    <img src="/assets/img/rover/high_voltage_rail.jpg" width="100%" alt="High-voltage rail powers motors and contains a DC step-down chip" />
    <em>High-voltage rail takes battery power and distributes to motor controllers and DC down converter.</em>
  </figure>
  <figure style="margin: 0; text-align: center;">
    <img src="/assets/img/rover/esp32_power_cable.jpg" width="100%" alt="Power cable from high-voltage rail to JST on ESP32 perfboard">
    <em>Power comes onto ESP32 board via JST connector to VIN/GND.</em>
  </figure>
</div>

Notice how the micro-usb for the ESP32 is hanging off the edge of the frame. This makes it easy to connect and upload code.

Here is a full wiring diagram:

![Rover battery, power rails, motor drivers, and 5 V buck converter](/assets/img/diagrams/rover_power.png)
*Power distribution for the motor drivers and ESP32 DevKit V1.*





<!-- TODO: Clean up this section, this is a rough draft that AI wrote from my prompt -->
<!-- 


## Software and control path

The control path runs from the browser to the motors:

```text
Browser controls
    -> HTTP request over the local Wi-Fi network
    -> ESP-IDF HTTP server and DeviceWebApp
    -> web peripheral wrapper
    -> DrivetrainOmni
    -> motor interfaces and ESP32 GPIO/PWM implementation
    -> L298N boards
    -> motors
```

`app_main()` creates the ESP32 GPIO and motor objects, combines the four motors
in `DrivetrainOmni`, and registers the web-facing peripherals with
`DeviceWebApp`. The web application and hardware interfaces keep page and
request handling separate from the low-level ESP32 implementation. That
separation also lets selected logic be built and tested on a desktop with
mocked hardware interfaces.

The drivetrain accepts x translation, y translation, and clockwise rotation
commands. It mixes those components into one command for each wheel, then
scales the outputs if needed to keep them within the supported range. The
commands are normalized requests, not measurements of wheel speed. Rover does
not currently have encoders, so it cannot report actual wheel speed or
position.

The control page is available at `http://192.168.1.10` on my local network. The
server uses plain HTTP and has no application-level authentication, so this is
a development interface for a trusted local network, not a hardened remote
control system.

## What I have tested

I brought the hardware up incrementally: first GPIO and the onboard LED, then
PWM, then a single motor, and finally the complete drivetrain. This let me
check smaller pieces before trying to debug the four-wheel mechanism as a
whole.

With the web controls, I have confirmed movement in both x directions, both y
directions, and both rotation directions. Here is a short video of Rover in
motion:

<video width="100%" controls muted playsinline>
  <source src="/assets/img/rover/rover_video_cropped.mp4" type="video/mp4">
  Your browser does not support embedded video.
</video>

The repository also has host-side GoogleTest coverage for selected logic.
Those tests help check software behavior without the physical rover, but they
do not validate the wiring, motor drivers, or how the chassis moves. The
[unit-testing post]({% post_url 2026-09-25-local-gtest-build %}) describes the
scope and setup in more detail.

## Limitations and next steps

There are two important limitations in the current control system. First, the
motor commands are open loop because there are no wheel encoders. Second, the
motors do not begin moving until the command is around 70% in my current setup.
The mapping from command to PWM is linear, so low-speed control is poor. That
threshold is an observation from this build, not a calibrated value for all
conditions; motor variation, load, battery voltage, and driver losses may all
affect it.

There is also no communication-loss stop. If Wi-Fi drops, the motors keep
running at their last commanded state. Adding and testing a command timeout
and stop behavior is a priority before extending the remote-control options.

After I finish setting up the project website, the next stage is to investigate
driving Rover with a PS4 controller over Bluetooth. Other ideas include tuning
the motor response, adding wheel encoders, and designing a custom PCB to
replace some of the perfboard wiring.

The PCB idea includes simplifying motor direction signals. Today each of the
four motors uses Forward, Reverse, and PWM signals, for 12 ESP32 outputs. One
direction output per motor plus inverter/gating logic to generate the driver
inputs, four PWM outputs, and one shared brake signal would use 9 outputs, a
net reduction of 3. The shared brake would affect all four motors, and its
logic would need to keep the L298N enable and direction inputs in the intended
states. This is only a design idea; I have not built or tested that circuit.

Longer term, I would like the project structure to make it easier to adapt the
software to a smaller, lower-cost microcontroller. The current hardware
implementations are specific to the ESP32, so that would still require a new
target implementation and testing on the new hardware. The interface
boundaries are a starting point for that work, not a claim that Rover is
already portable.

## Build information

The firmware targets the ESP32 DevKit V1 and was built with ESP-IDF v6.0.2.
Install the matching ESP-IDF tools, open the `web_server` project, select the
`esp32` target, and use the ESP-IDF build/flash workflow described in the
[project README](https://github.com/JFolkens/esp32_peripheral_debug). The
firmware connects to a Wi-Fi access point in station mode; configure the SSID
and provide the local password override described in `web_server/main/web_server.cpp`.
Keep real credentials out of source control.

For more detail, continue with the [Web-Peripheral Interface]({% post_url 2026-09-25-modular-embedded-design-the-web-peripheral-interface %}),
[Unit Testing an Embedded Platform]({% post_url 2026-09-25-local-gtest-build %}),
and [Binary Assets on ESP32]({% post_url 2026-09-25-binary-assets-on-esp32 %}) posts. -->
