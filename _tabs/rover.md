---
icon: fas fa-car
order: 3
---

![Rover](/assets/img/rover/rover_landscape.jpg){:style="width:50%; display:block; margin-left:auto; margin-right:auto"}
*Rover: A modular embedded development platform*

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
