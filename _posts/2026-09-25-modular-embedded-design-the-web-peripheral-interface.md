---
title: "Modular Embedded Design: The Web-Peripheral Interface"
date: 2026-09-25 09:00:00 -0500
categories: [Rover, Web, Architecture]
tags: [esp32, embedded, web, peripherals, architecture]
---


## Overview
Rover was designed to be tested incrementally; it is meant as a platform for adding and testing new embedded sensors/controls (and - let's be honest - it is a really cool toy).

![Rover - an embedded platform with omni wheels](/assets/img/rover/rover_dark.jpg){:style="width:50%; display: block; margin: 0 auto;"}

Rover's development interface is a locally hosted web server. The debug web server was created to be modular. Each peripheral has a section on the website that can be added or removed with a few lines of code.

![The ESP32 Debug Web Server](/assets/img/web_server/webpage_motor_pwm_led.png){:style="width:50%; display:block; margin-left:auto; margin-right:auto"}

To generated the web page above:

![Source code for ESP32 Debug Web Server](/assets/img/web_server/webpage_motor_pwm_led_code.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}

To add a peripheral to the webpage,

- Instantiate the hardware object
- Pass hardware to a "Web" wrapper class
- Call `app->add_peripheral`

And that's it! Instead of re-architecting the web page, hardware changes can be made in a few lines of code. Bringing up new peripherals is done in isolation, without remembering how the web page design actually works.

## The Power of a Modular Interface

Rover's drivetrain is a composite peripheral:

![Drivetrain composite class architecture](/assets/img/web_server/drivetrain_composite.png){:style="width:60%; display:block; margin-left:auto; margin-right:auto"}

Waiting to test the hardware until the Drivetrain was complete would be a complete nightmare.

Instead, I started by testing GPIOs (via LED). Then a single PWM. Then I sat on my couch and confirmed that a PWM and two GPIOs could move a single wheel.

![Rover with wheel up in testing configuration](/assets/img/rover/early_debugging.jpg){:style="width:50%; display:block; margin-left:auto; margin-right:auto"}

By the time I wrote the Motor controller class, I was highly confident in the underlying hardware.

Because Rover uses omni wheels, going "forward" is done by setting [half of the wheels in reverse.](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/drivetrain/drivetrain_omni.cpp#L15) If that was the first time I tested the motors - oh boy.

And, when I have any issues (say a motor wire snaps), I can quickly revert to a web page with lower-level controls. Instead of guessing why my tiny car is staggering, I can control the wheels individually and quickly isolate the issue.

## Web Peripheral Interface: Adding Peripherals

Each peripheral appears on the webpage in the order that they were added to [`DeviceWebApp`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/web/device_web_app.h).

To be a peripheral, you must extend the [`PeripheralInterface`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/web/peripherals/peripheral_interface.h) class. This means defining:
- The visual HTML representation (state and controls)
- Hooks for updating hardware (controls)
- A hardware update function

![LedWeb Interface](/assets/img/web_server/led_web_interface.png){:style="width:80%; display:block; margin-left:auto; margin-right:auto"}
*LedWeb implements the PeripheralInterface*

`LedWeb` is an example of a class that extens `PeripheralInterface`.






For example, [`LedWeb`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/web/peripherals/led_web.h) has a button that toggles between **Turn ON** and **Turn OFF**.

The parent constructor `PeripheralInterface` takes in the peripheral name (`green_led`, `front_left_wheel`, etc), and typically an ownership reference to the underlying hardware module (`PwmInterface`, `GpioInterface`, etc).

## Integrating Peripherals

[`DeviceWebApp`](https://github.com/JFolkens/esp32_peripheral_debug/blob/main/main/web/device_web_app.h) is a wrapper around a list of `PeripheralInterface` objects and an `HttpServer` instance. It renders the overall HTML page, provides the header, and calls the appropriate function for each registered peripheral. It also owns the `HttpServer` and registers the URI endpoints needed for the callbacks. This keeps endpoint and server logic away from the peripheral development process.

Once a class implements `PeripheralInterface`, adding additional instances is as straightforward as calling `device_web_app->add_peripheral(...)`.

`HttpServer` includes code for parsing URI endpoing key-value pairs. Each Peripheral has a `name`. Update endpoints are of the format `/name/update?key=value&key2=value2`. When `HttpServer` receives a request, it parses the path and key-value parameters and then hands control off to `DeviceWebApp`. Then `DeviceWebApp` does a lookup of peripherals by their name, and calls `peripheral->handle_update()` with a map of parameters.


## Next Steps

The `PeripheralInterface` architecture is a start. The routing through `HttpServer` and `DeviceWebApp` means that URI calls to devices are parsed uniformly. However, the HTML/Javascript code generating those URIs lives directly in each `PeripheralInterface` implementation. That means that unfortunately new Peripherals do need to have some knowledge of the action / update protocol. This could be extracted down into template / helper classes or up into `DeviceWebApp` or `PeripheralInterface` utility functions.

A big next step is adding the first sensor-based Peripheral type. The Rover currently uses motors without encoders. Sensors need to be refreshed dynamically without user interaction with the web page. There are a few ways this could be handled:
- Have each PeripheralInterface embed a callback in the HTML/Javascript for refreshing itself. The benefit is that each sensor has ultimate control over its refresh rate. However, it also means more coupled architecture code inside of the Peripheral classes.
- Implement `SensorInterface` as an extension of `PeripheralInterface` for polling.
- Have `DeviceWebApp` aware of sensor-vs-control peripherals, and add an optional argument to `add_peripheral()` for a refresh rate.

Another decision is whether the polling happens on the server - in Javascript - or using FreeRTOS for timing (of course the server is running on FreeRTOS, but there are layers of explicitness). The decision of where to put polling logic and what language to write it in must be made together.
