---
icon: fas fa-car
order: 3
permalink: /
---

![Rover](/assets/img/rover/rover_landscape.jpg){:style="width:50%; display:block; margin-left:auto; margin-right:auto"}
*Rover: A modular embedded development platform*

Rover is an ESP-32 powered car controlled through a hosted web server.


## Posts

- {% assign my_post = site.posts | where: "path", "_posts/2026-10-11-rover-overview.md" | first %}
<a href="{{ my_post.url | relative_url }}">Rover: Overview</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-modular-embedded-design-the-web-peripheral-interface.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-local-gtest-build.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>
- {% assign my_post = site.posts | where: "path", "_posts/2026-09-25-binary-assets-on-esp32.md" | first %}
<a href="{{ my_post.url | relative_url }}">{{ my_post.title }}</a>

<div style="display: grid; grid-template-columns: 80%; justify-content: center; padding-bottom: 20px;">
  <video width="90%" controls muted playsinline>
    <source src="/assets/video/rover_dance_video_cropped.mp4" type="video/mp4">
    Your browser does not support embedded video.
  </video>
</div>
