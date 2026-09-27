---
icon: fas fa-car
order: 3
---

_The modular embedded development platform_

![Rover](/assets/img/rover/rover_landscape.jpg){:width="50%": style="float: right; margin-left: 15px;"}

Rover is an ESP-32 powered car controlled through a hosted web server.


## Posts

- Overview *(coming soon)*
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-modular-embedded-design-the-web-peripheral-interface.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-local-gtest-build.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- Evaluating Binary Size and Updating the Partition Table *(coming soon)*
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-binary-assets-on-esp32.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>

<video width="50%" src="/assets/img/rover/rover_video_cropped.mp4" controls autoplay muted playsinline></video>
