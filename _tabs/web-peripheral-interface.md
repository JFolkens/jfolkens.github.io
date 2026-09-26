---
icon: fas fa-network-wired
order: 4
---

# Modular Embedded Design: The Web-Peripheral Interface

One common issue with embedded development is testing incremental stages without leaving novels of commented out code lying around. Rover is designed to be iterated on; it is fundamentally a platform for adding and testing new embedded sensors/controls (and - let's be honest - it is also a really cool toy).

The web-peripheral interface means that new Peripheral types can be created with one class. Instead of re-architecting the web page, peripherals can be registered on one line and can be removed just as easily. Bringing up a new peripheral can be done in isolation, without remembering how the HttpServer actually works.

This was especially useful for developing Rover's drivetrain as a composite peripheral - instead of testing the entire platform "all at once". The Drivetrain itself is composed of four Motor objects. Each Motor is actually a PWM with two direction GPIOs (gpio-forward and gpio-reverse). Rover began with a single PWM and two GPIOs. I sat on my couch and confirmed that I could move a single wheel forwards and backwards, at different speeds. Then I formed the Motor interface and confirmed that I could still move the wheel. Then I wrote Drivetrain as four wheels. I still have the PWM web interface as a class in my codebase - not an old branch on Github, but something I can quickly add back in.


![Rover on blocks stub](/assets/img/rover/rover-on-blocks.png)

*Stub: Picture of Rover on blocks so I can test the wheels without it running off*

## Peripheral Interface

The web UI is organized around **cards**. In practice, each peripheral gets its own section of the page, shown as a card in the HTML. These cards appear from top to bottom in the same order that peripherals are added to `DeviceWebApp`.

![Motor webpage stub](/assets/img/rover/motor-webpage-stub.png)

*Stub: image HTML page with multiple cards.*

That gives the page a simple, modular feel:

- each peripheral owns its own display and controls
- the page can show multiple peripherals together
- peripherals can be added or removed without redesigning the whole site
- the UI stays useful while the hardware is still in flux

The content of each card is defined by the PeripheralInterface class. The `LedWeb` class has a button that toggles between **Turn ON** and **Turn OFF**. `PwmWeb` is a slider from 0 to 100 percent duty cycle.

Each new `PeripheralInterface` class must define:
- The visual HTML representation (state and controls)
- Hooks for updating hardware (controls)
- A hardware update function

The parent constructor `PeripheralInterface` takes in the peripheral name (`green_led`, `front_left_wheel`, etc), and typically an ownership reference to the underlying hardware module (`PwmInterface`, `GpioInterface`, etc).

## Integrating Peripherals

`DeviceWebApp` is a wrapper around a list of `PeripheralInterface` objects and an `HttpServer` instance. It renders the overall HTML page, provides the header, and calls the appropriate function for each registered peripheral. It also owns the `HttpServer` and registers the URI endpoints needed for the callbacks. This keeps endpoint and server logic away from the peripheral development process.

Once a class implements `PeripheralInterface`, adding additional instances is as straightforward as calling `device_web_app->add_peripheral(...)`.

`HttpServer` includes code for parsing URI endpoing key-value pairs. Each Peripheral has a `name`. Update endpoints are of the format `/name/update?key=value&key2=value2`. When `HttpServer` receives a request, it parses the path and key-value parameters and then hands control off to `DeviceWebApp`. Then `DeviceWebApp` does a lookup of peripherals by their name, and calls `peripheral->handle_update()` with a map of parameters.

## Example: `LedWeb`

Each `LedWeb` gets its own card on the page. That card shows the LED’s current state and provides a simple control for turning it on or off.

```cpp
// Adding a new LED to the webpage
app->add_peripheral(std::make_shared<LedWeb>(...));
```

```cpp
// Example sketch of the interface responsibilities
class PeripheralInterface {
 public:
  virtual std::string html_state() = 0;
  virtual std::string html_control() = 0;
  virtual void handle_update(...) = 0;
};
```

![LED on/off stub](/assets/img/rover/led-toggle-stub.png)

*Stub: image showing an LED turning on and off through the web interface.*

![Rover peripheral cards page stub](/assets/img/rover/web-peripheral-cards-stub.png)


*Stub: screenshot of the Rover webpage showing multiple peripheral cards rendered top to bottom, including one or more debug-oriented cards.*

## Next Steps

The `PeripheralInterface` architecture is a start. The routing through `HttpServer` and `DeviceWebApp` means that URI calls to devices are parsed uniformly. However, the HTML/Javascript code generating those URIs lives directly in each `PeripheralInterface` implementation. That means that unfortunately new Peripherals do need to have some knowledge of the action / update protocol. This could be extracted down into template / helper classes or up into `DeviceWebApp` or `PeripheralInterface` utility functions.

A big next step is adding the first sensor-based Peripheral type. The Rover currently uses motors without encoders. Sensors need to be refreshed dynamically without user interaction with the web page. There are a few ways this could be handled:
- Have each PeripheralInterface embed a callback in the HTML/Javascript for refreshing itself. The benefit is that each sensor has ultimate control over its refresh rate. However, it also means more coupled architecture code inside of the Peripheral classes.
- Implement `SensorInterface` as an extension of `PeripheralInterface` for polling.
- Have `DeviceWebApp` aware of sensor-vs-control peripherals, and add an optional argument to `add_peripheral()` for a refresh rate.

Another decision is whether the polling happens on the server - in Javascript - or using FreeRTOS for timing (of course the server is running on FreeRTOS, but there are layers of explicitness). The decision of where to put polling logic and what language to write it in must be made together.
