---
layout: single
author_profile: false
permalink: /
---

Welcome to the pas organizational site.

# What is pas?

pas is an audio server capable of sending multiple concurrent analog or digital stereo feeds to some (as yet undetermined maximum) number of outboard DACs. Each DAC drives a separate audio zone in a multi-zone or multi-room installation.

A key feature of pas is that it is quite light weight, capable of running multiple concurrent streams even from a $50 ARM-based dev-board. See below (and the [development environment](development-environment) page) for information about the microscopic computer pas being being developed on.

Some information about pas:

- pas is Linux based.
- pas is heavily multithreaded and likes multicore machines.
- pas is written to be headless. UI's are provided via ssh or other means such as a web server.
- audio is emitted using <a href="https://www.freedesktop.org/wiki/Software/PulseAudio/">pulseaudio</a> via USB ports.
- audio is decoded using <a href="https://ffmpeg.org/">ffmpeg</a> so pas supports those formats supported by ffmpeg.
- data is maintained using <a href="https://www.mysql.com/">MySQL</a>.
- pas *may* expose a MPD-compatible interface as the pas API is quite robust.
- pas is being developed on an <a href="http://www.hardkernel.com/main/products/prdt_info.php?g_code=G143452239825">odroid XU4</a>.
- pas is being developed using an audioengine D3 USB DAC, a DragonFly Black from AudioQuest and 3 Fiio DACs - see the hardware page for more information..