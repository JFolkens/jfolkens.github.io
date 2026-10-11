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

![Rover battery, power rails, motor drivers, and 5 V buck converter](/assets/img/diagrams/rover_power.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}
*Power distribution for the motor drivers and ESP32 DevKit V1.*


## Software and control

The first iteration of Rover is controlled via Web server interface:

![Web server interface](/assets/img/web_server/webpage.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}
*Rover webpage.*

The webpage is designed to be [modular]({% post_url 2026-09-25-modular-embedded-design-the-web-peripheral-interface %}) and easily re-configurable. My next iteration of Rover will be controlled via Bluetooth with an actual PS4 controller.

The firmware targets the ESP32 DevKit V1 and was built with ESP-IDF v6.0.2. Setup using the VS Code ESP-IDF extension is described on the
[project README](https://github.com/JFolkens/esp32_peripheral_debug). Rover connects to your local Wi-fi and instructions for providing Wi-fi name and password (not included) are described in [source code comments](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/web_server.cpp#L34).

The [Rover codebase](https://github.com/JFolkens/esp32_peripheral_debug/tree/rover_v1) is publicly available on Github, and the rest of the posts on this webpage cover the software design and architecture for the platform.

## Rover in action

<div style="display: grid; grid-template-columns: 80%; justify-content: center; padding-bottom: 20px;">
  <video width="60%" controls muted playsinline>
    <source src="/assets/img/rover/rover_video_cropped.mp4" type="video/mp4">
    Your browser does not support embedded video.
  </video>
</div>

## Future design spirals

Rover is current v1: a basic platform with known limitations. Future design spirals will address the following:

- Rover's drivetrain is currently open loop because I have not yet purchased and installed wheel encoders. This gives the platform a bit of a stagger since the motors and wheels are operating at slightly different strengths.

- The motor control logic is basic - "speed" is raw PWM duty cycle. With the 8V batteries, the PWM has to hit 70% before the wheels move. Once encoders are installed, I can implement an actual PID control algorithm.

- Rover is not Wi-fi drop safe. That is, if my Wi-fi goes down, it will continue on its current path until I can catch it. Adding and testing a connection-loss behavior would be more "safe" (for my shins).

- Rover v2 will be controlled with a PS4 controller over Bluetooth.

- Custom PCB with simplified / fewer motor control pins. Currently there are `4 x 3 = 12` GPIO pins used for motor control (not including encoders). This can be reduced to `(4 x 2 + 1) = 9` pins by using an inverter to set `Forward = !Reverse`. Then each motor is controlled by `{Direction | Speed}`. The additional pin is a braking pin; setting `Forward = Reverse` is a hard brake for the L298N controllers. A single `Stop` GPIO can be distributed to all four wheels (no wheelies, but very efficient).

## Rover posts

See additional topics on Rover design, testing, and implementation:

- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-modular-embedded-design-the-web-peripheral-interface.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-local-gtest-build.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-binary-assets-on-esp32.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>

