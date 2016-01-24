---
layout: page
title: Python/asyncio based BitTorrent tracker
---

I just uploaded a Python/asyncio-based UDP BitTorrent tracker to my
github account. It's called `pybtracker` and you can find it
[here][1].

You can install pybtracker using pip by running `pip3 install
pybtracker`. You'll need Python 3.5. After installing pybtracker you
can simply run it like `pybtracker -b 127.0.0.1:8000 -O`. An
interactive client is also included which can be used by running
`pybtracker-client udp://127.0.0.1:8000` (update server address as you
wish).

 [1]: http://github.com/elektito/pybtracker
