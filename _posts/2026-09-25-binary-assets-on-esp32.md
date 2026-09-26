---
title: Binary Assets on ESP32
date: 2026-09-25 10:00:00 -0500
categories: [Rover, Web, ESP32]
tags: [esp32, cmake, assets, javascript, pwm, web]
---

Rover is a tiny omni-wheel platform controlled via a small Http web server. The first iteration published a webserver whose content was created directly via C++ with `std::stringstream`. It was... something.

![PwmWeb with stringstream embedded javascript](/assets/img/rover/pwm_web_with_initial_javascript.png)

*The original PwmWeb class with in-line HTML/javascript code.*

See ![TODO: Link to other post](`Web Peripheral Interface`) for the architecture surrounding the `PwmWeb` class.

It worked, but it was hard to write, hard to read, and hard to keep clean as the project grew. `Stringstream` is no way to write javascript. Tracking the escape characters alone was frustrating and cumbersome.

Now `PwmWeb` uses an embedded asset file, and the javascript control script can be edited in a properly-highlighted `pwm_control.js` file:

![PwmWeb with C++ text embeddings](/assets/img/rover/pwm_web_with_embedded_javascript.png)

It still isn't perfect (see [TODO: Link to section below](`Next Steps`)), but it is a significantly nicer way to develop new embedded server assets.

## How it works

The basic idea is simple:

- write the browser-side logic as separate asset files
- embed those assets into the ESP32 build
- link the embedded data into the C++ program
- keep the peripheral classes focused on behavior instead of page construction

That sounds straightforward, but the implementation can get interesting.

The ESP32 build system has its own rules for how embedded assets are handled, and I had to work through the details carefully. I also wanted the build to stay organized as new files were added, without forcing the high-level web code to know too much about the low-level embedding process.

## Adding assets to the ESP32 build

The first step was getting the asset files into the build system.

ESP-IDF has a parameter in `idf_component_register` for embedded textfiles (`EMBED_TXTFILES`). In `main/CMakeLists.txt`:

```cmake
# Gather all Javascript (*.js) and CSS (*.css) assets
file(GLOB_RECURSE
    ASSET_FILES
    CMAKE_CONFIGURE_DEPENDS     # Catch new files without a full clean
    "${CMAKE_CURRENT_LIST_DIR}/*.js"
    "${CMAKE_CURRENT_LIST_DIR}/*.css"
)

# Add to ESP-IDF component build
idf_component_register(
    SRCS ${SRC_FILES}
    INCLUDE_DIRS "." ${GENERATED_ASSETS_DIR}
    EMBED_TXTFILES ${ASSET_FILES}
)
```

There is also `EMBED_FILES` for binary assets.

Once the assets are embedded to the build, they can be linked using symbols made available through the ESP-IDF compilation process. The [Espressif docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html) show an example of adding the binary data to source files via an `extern const uint8_t` declaration.

The docs also say to use the "full name of the file, as given in `EMBED_FILES`". This felt a little magick-y to me, but I hooked it all up. Linkage failed due to to the symbol not being found.

I tried a few more magic symbol variations ("maybe if I leave of the _js extension?") before realizing that I'd have better luck cracking a password I set ten years ago.

## Finding the real linked symbols

First, I used CMake commands to print out the exact value of my `${ASSET_FILES}`. This told me that CMakeLists was finding my files, and the paths looked as I expected.

* Stub: Picture of printing ${ASSET_FILES} in CMakeLists and seeing outputs *

The next question is how those assets appear as symbols in the linking stage. This was done by inspecting the linked binary to see what symbols were really present.

The linked binary lives in the build/ folder next to main/. It can be inspected using

`xtensa-esp32-elf-nm -C .\build\esp-idf\main\libmain.a`

If you're like me and you use VS Code to handle ESP-IDF via the extension, you may not remember where your ESP32 tooling was installed. Type `Ctrl+Shift+P` and search for the option "ESP-IDF: Open ESP-IDF Terminal". The terminal will have `xtensa-esp32-elf-nm` in its path.

To search the linked file for the asset symbols, run:

`xtensa-esp32-elf-nm -C .\build\esp-idf\main\libmain.a | Select-String "binary"`


![Embedded Link Names](/assets/img/rover/discovering_symbol_names_with_nm.png)

(This is a Windows PowerShell command; use `grep` on Linux)

Sure enough, I was using the wrong symbols in my C++ source code. Specifically, the path in the CMakeLists.txt was not being used; only the file name. Instead of `_binary_web_assets_led_control_js_start`, I needed `_binary_led_control_js_start`.

## Wrapping the binary assets
Linked symbols worked once the right `extern` variables were declared. The last inconvenience was the coupling between the linked symbols and the higher level application code. Because the linked symbol names are specific to ESP32, they belong in the `main/esp32` folder (see project architecture in [`Embedded Unit Tests`](TODO: Cross reference posting)). But I don't want to update the code in `main/esp32` every time the web application adds an asset file.

My solution was to use CMake magic. Since my `CMakeLists.txt` file already had a list of embedded asset files, I added a [helper script](TODO: Git repo link to main/cmake/GenerateAssetHeader.cmake) for embedding the symbols.

![Auto-generated Symbol Declarations](/assets/img/rover/asset_auto_generated_header.png)

I turned clang formatting off for that file because my pre-commit hooks where adding newlines, which then got removed on every build. The old "who will win in a fight between my linter and my build system".

Now a nice structure so I can retrieve asset locations by file name:

![Auto-generated Assets List](/assets/img/rover/asset_auto_generated_structs.png)

Notice that the map uses file name only. Because the symbols are throwing away the path, file names must be unique. This is not an implementation detail; trying to "fix" this in the C++ code will result in incorrect symbol linkage.

I prefer a very minimal code comment philosophy. My all-time favorite [best practices for writing code comments](https://stackoverflow.blog/2021/12/23/best-practices-for-writing-code-comments/) article starts with "comments should not duplicate code" and "good comments do not excuse unclear code". This "do-not-change-because-linkage-will-fail" is an excellent example of an absolutely necessary comment. Why? Because two years from now, I will go into this function, say "I want the key to depend on file paths, not just names", and I will break everything in hard-to-debug ways. If I was working with a team of developers? No chance.

![Warning my future self against re-introducing bugs](/assets/img/rover/asset_code_warning.png)

Another big place for comments is in the [GenerateAssetHeader.cmake](TODO: Link to github) helper script. If Espressif every brings their symbol names in line with the documentation, or if I ever fat-finger a file name, I need that `xtensa-esp32-elf-nm` command to sort things out. The best place to write it down? In the file I will go looking for when things break.

![Putting useful commands in the source where I will see them](/assets/img/rover/asset_cmake_helper_comments.png)

And, in case it was not obvious, this blog post itself is my ultimate note-to-self on how all of this works. Writing reinforces learning and preserves knowledge. Don't lie to yourself about what you will remember two years from now - just write it down.

### Summary

Embedded asset linkage is one of those areas where the build system, linker, and target toolchain intersect. They either all work together, or they all fail together. Find ways to isolate and test each step is crucial. And if you can find an example online, use it. If you can't, then type one up for the next poor sap.

## Next Steps

The `PwmWeb` example above shows the Javascript being referenced from an asset file. I decided not to take the time to embed the HTML, in the interest of diminishing returns. The HTML elements are named after C++ variables. The complicated Javascript sections (complicated for me, a humble embedded developer) are bracketed by HTML and would need start and end blocks. Being able to edit the Javascript in a syntax-highlighted `.js` file was the knee in the curve.

There is one improvement that still itches at me. A sharp reader will notice is that `pwm_control.js` defines the same anonymous function for each PWM. A cleaner long-term approach would be:

- each `PeripheralInterface` class exposes optional static asset properties like `style_css` or `control_js`
- `DeviceWebApp` collects unique asset blocks for each peripheral class it owns
- the page includes those assets once, instead of repeating them per instance

This was not worth the effort to date, but would be in a larger project. The anonymous functions do not scale well, but given the number of GPIOs actually on an ESP32 device, the duplication is not significant.

## Takeaway

Moving browser-side code into embedded asset files made Rover’s web layer much cleaner.

It reduced the amount of awkward C++ string construction, made the JavaScript easier to work on directly, and gave me a more maintainable path for the ESP32 build.

The next post will cover how embedded assets can be integrated with the unit test structure, which brings in a separate set of linking and build concerns.
