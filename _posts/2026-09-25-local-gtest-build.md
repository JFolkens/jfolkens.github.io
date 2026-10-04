---
title: Unit Testing an Embedded Platform
date: 2026-09-25 10:00:00 -0500
categories: [Rover, Testing]
tags: [gtest, cmake, embedded, esp32, pre-commit]
---

Unit testing is important and is particularly difficult on embedded platforms. There are two flavors of embedded tests:

| | On-target | Off-target |
| --- | --- | --- |
| **Execution Environment** | Microcontroller | Computer |
| **Toolchain** | Target-specific | Native (GCC, Clang, MSVC) |
| **Hardware Coupling** | High | Low |
| **Portability** | Low | High |
| **Automation** | Low | High |

Off-target tests can be developed before hardware is in hand, and are useful for testing complex pieces of non-embedded logic. On-target tests require a second binary to be flashed to your device; they are less convenient and so tend to be run less often. But on-target tests catch actual hardware and low-level library bugs. For an in-depth discussion of embedded testing, see James W. Grenning's book on [Test-Driven Development for Embedded C](https://www.amazon.com/dp/193435662X?lv=shuf&channelId=500&plpRedirect=mhFallback).

Rover uses the {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-modular-embedded-design-the-web-peripheral-interface.md" | first %}
<a href="{{ my_post.url | relative_url }}">Web-Peripheral Interface</a> for on-target testing and debugging of new hardware. This post covers off-target unit testing via the GoogleTest framework.

## Writing Embedded Code That Can Be Tested

The key to off-target tests is to isolate embedded code from higher-level logic. This requires constant vigilance; hardware modules want to be written with their algorithmic buddies, and they must be pulled apart at every turn.

For compilation reasons, Rover splits this into two folders:

- [`main/hal`](https://github.com/JFolkens/esp32_peripheral_debug/tree/main/main/hal) - the Hardware Abstraction Layer which defines hardware behavior that is consistent across target platforms
- [`main/esp32`](https://github.com/JFolkens/esp32_peripheral_debug/tree/main/main/esp32) - any actual ESP32 library calls

For example, each PWM should be able to `set_speed`, `get_speed`, and `turn_off`; this definition belongs in the HAL. PWM controls do not have sensors, so `get_speed` is based on the last call to `set_speed`. This logic can be tested. What you can not do is control an ESP32 PWM port without calling the ESP32 libraries - that code belongs in `main/esp32` and can not be tested off-target. The section below on `HttpServer` demonstrates a less-trivial separation of concerns.

Finally, there is the `tests` folder. Note that while ESP32 libraries can not be compiled locally, the test code likewise can not be compiled on-target (and absolutely does not belong on an embedded device, where flash size is constrained).

The next section shows the magic of compiling the right code into the right places at the right time.

## Adding Unit Tests to an ESP32 project

The test entry point is [`tests/run_unit_tests.bat`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/tests/run_unit_tests.bat), which drives the local test workflow. It only works on Windows because I have not had a reason to support Linux yet, but I suspect ChatGPT can write the equivalent `.sh` script. The key is that `run_unit_tests.bat` automates the download and installation of the GTest library to the `tests/.deps` folder. I like this even better than `README` instructions which could be out of date or hard to follow. Before sharing the project, I can remove the `tests/.deps` folder, re-run the tests, and have some hope that another user will be successful.

A default ESP32 project looks for source code in the `main` folder. I use VS Code with the ESP-IDF extension and did not have issues with ESP32 and the root-level `CMakeLists.txt` trying to compile my `tests` folder (I did have issues trying to rename `main` to `src`, but decided it wasn't worth fighting ESP32 on that one).

```text
rover/
├── main/
|   ├── esp32/
|   ├── web_server.cpp
|   ├── CMakeLists.txt
├── tests/
|   ├── .deps/
|   ├── run_unit_tests.bat
|   ├── CMakeLists.txt
├── CMakeLists.txt
```

What [`tests/CMakeLists.txt`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/tests/CMakeLists.txt) does is:

- Confirm GTest is available
- Exclude `main/esp32` which depends on ESP32 target libraries
- Exclude `main/web_server.cpp` (the main entrypoint) which must be aware that the target is an ESP32
- Build `*.cpp` test files in `tests/` against `*.cpp` files in `main`, producing an executeable `tests/.build/Debug/unit_tests.exe`

The script `run_unit_tests.bat` runs the resulting test executeable.

![Local test runner - first build installs dependencies](/assets/img/build_logs/run_unit_tests_first_build.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}
*First invocation of test script is slow because it installs GoogleTest dependencies.*

![Local test runner - second build uses cached dependencies](/assets/img/build_logs/run_unit_tests_second_build.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}
*Second invocation uses cached dependencies.*

![Local test runner - unit tests pass](/assets/img/build_logs/unit_tests_pass.png){:style="width:80%; display:block; margin-left:auto; margin-right:auto"}
*Unit tests showing all green.*


## Example Unit Test: `HttpServerInterface`

`HttpServer` is a hardware device with non-trivial logic and testing.

```text
rover/
├── main/
|   ├── esp32/
|   |   ├── http_server_esp32.h
|   |   ├── http_server_esp32.cpp
|   ├── hal/
|   |   ├── http_server/
|   |   |   ├── http_server_interface.h
|   |   |   ├── http_server_interface.cpp
├── tests/
|   ├── hal/
|   |   ├── http_server/
|   |   |   ├── http_server_interface_tests.cpp
```

[`HttpServerInterface`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/hal/http_server/http_server_interface.h) covers the routing logic and the helper functions for parsing key-value pairs from URI requests. String parsing and callback logic should be unit tested because it is bug-prone and does not require ESP32 hardware to validate.

[`HttpServerESP32`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/main/esp32/http_server_esp32.h) contains the code needed to implement `HttpServerInterface` on an actual ESP32 device. These function calls can not be tested locally (atleast, not easily - high-end chips tend to ship with simulators, but the $20 ESP32 has no lead time and why simulate when you can just buy one).

## Pre-commit Integration
The final piece is integrating unit tests with pre-commit hooks. Unit tests are only useful if they are run regularly; otherwise they aren't checking anything. Some projects integrate CI/CD into git or Jenkins pipelines. For small projects, I find it easiest to simply prevent myself from commiting code that fails.

Both a clang formatter and the unit test executeable are wired into [`.pre-commit-config.yaml`](https://github.com/JFolkens/esp32_peripheral_debug/blob/rover_v1/.pre-commit-config.yaml). I am a lazy person; if it is hard to push code that is unformatted and untested, I tend to do more formatting and testing.

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: check-added-large-files
      - id: trailing-whitespace

  - repo: https://github.com/pre-commit/mirrors-clang-format
    rev: v18.1.8
    hooks:
      - id: clang-format
        args: ["-style=file:.clang-format"]
        types_or: [c++, c]

  - repo: local
    hooks:
      - id: run-unit-tests
        name: Build and Run C++ Unit Tests
        entry: 'cmd.exe /c .\\tests\\run_unit_tests.bat'
        language: system
        pass_filenames: false
```

Python projects can use `pip install pre-commit` to access the handy CLI `pre-commit` command. I don't feel like putting `pre-commit` in my base environment or creating a Rover python venv, so when I want to run pre-commit checks without commiting I use `.git/hooks/pre-commit`. Or, the `run_unit_tests.bat` script.

![Pre-commit hook stub](/assets/img/build_logs/git_pre-commit_failed.png){:style="width:90%; display:block; margin-left:auto; margin-right:auto"}
*Using git pre-commit to avoid commiting un-formatted code.*

## Takeaway

The main benefit of this setup is that Rover’s unit tests stay practical while the project stays embedded-friendly.

The host-side test build lets me exercise the testable logic locally, while the HAL/ESP32 split keeps the target-specific code where it belongs. That gives me a cleaner development loop now, and it should also make the codebase easier to adapt later.

## Next steps

The Web Peripheral Interface is a form of on-target debugging. Typically I will start by developing the interface/tests, then the ESP32 integration, and finally adding the new peripheral to the debug web page.

A future post will cover mocking some trickier embedded functionality.

Up Next: {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-binary-assets-on-esp32.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
