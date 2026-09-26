---
icon: fas fa-network-wired
order: 4
---

# Modular Embedded Design: The Web-Peripheral Interface

One common issue with embedded development is testing incremental stages without leaving novels of commented out code lying around. Rover is designed to be iterated on; it is fundamentally a platform for adding and testing new embedded sensors/controls (and - let's be honest - it is also a really cool toy).

The web-peripheral interface means that new Peripheral types can be created with one class. Instead of re-architecting the web page, peripherals can be registered on one line and can be removed just as easily. Bringing up a new peripheral can be done in isolation, without remembering how the HttpServer actually works.

This was especially useful for developing Rover's drivetrain as a composite peripheral - instead of testing the entire platform "all at once". The Drivetrain itself is composed of four Motor objects. Each Motor is actually a PWM with two direction GPIOs (gpio-forward and gpio-reverse). Rover began with a single PWM and two GPIOs. I sat on my couch and confirmed that I could move a single wheel forwards and backwards, at different speeds. Then I formed the Motor interface and confirmed that I could still move the wheel. Then I wrote Drivetrain as four wheels. I still have the PWM web interface as a class in my codebase - not an old branch on Github, but something I can quickly add back in.

## Peripheral Interface

The web UI is organized around **cards**. In practice, each peripheral gets its own section of the page, shown as a card in the HTML. These cards appear from top to bottom in the same order that peripherals are added to `DeviceWebApp`.

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

`DeviceWebApp` then handles the wrapper around a list of `PeripheralInterface` objects and an `HttpServer` instance. It renders the overall HTML page, provides the header, and calls the appropriate function for each registered peripheral. It also owns the `HttpServer` and registers the URI endpoints needed for the callbacks. This keeps endpoint and server logic away from the peripheral development process.

Once a class implements `PeripheralInterface`, adding additional instances is as straightforward as calling `device_web_app->add_peripheral(...)`.

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

## Debugging with smaller pieces

Because Rover is built from smaller peripherals, I can expose lower-level components when I need to debug them.

For example, instead of only testing the full drivetrain as one unit, I can temporarily work with the individual `PWM`, `MotorL298N`, or `DrivetrainOmni` layers. That makes it much easier to isolate bugs and compare behavior while the project is still evolving.

This is also why I included a separate HTML reference for the four-motor debugging view.

![Rover peripheral cards page stub](/assets/img/rover/web-peripheral-cards-stub.png)

*Stub: screenshot of the Rover webpage showing multiple peripheral cards rendered top to bottom, including one or more debug-oriented cards.*

![LED on/off stub](/assets/img/rover/led-toggle-stub.png)

*Stub: image showing an LED turning on and off through the web interface.*

## Takeaway

The big advantage of this design is that it keeps Rover flexible while I am still building it.

I can test new hardware quickly without going back to my old code, and I can add new peripherals to the webpage without touching the underlying server or needing to understand all of the URI endpoint details. That has made development faster, debugging easier, and experimentation much less painful.
