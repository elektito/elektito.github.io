---
layout: page
title: DPDK, tamed!
---

[DPDK][1] is a fantastic piece of software.  I've used it both for
work and in my hobby projects (yes, I crunch packets as a hobby, say
what you will about me!) and it works great. My only grievance with it
has always been the complicated build system it forces on you. It's
ugly and inflexible and it sometimes drives you crazy.

Yesterday, I saw that DPDK is in Ubuntu's repositories. It took me
some time to realize wha that means. I was looking for the value of
the RTE_SDK variable when it clicked in my head; I could just use the
damn thing like any normal library; with `-l`, `-L`, `-I` and all
that. I just needed to add a `-msse4.2` flag for the compilation to
work properly (I've been adding that flag to the `TOOLCHAIN_CFLAGS`
make variable on some of my machines to make DPDK compile anyway).

Does it have any drawbacks? I have no idea, as I never managed to
penetrate the many layers of DPDK make files to see what they really
do. It's quite possible that this is not as efficient as compiling
DPDK yourself so it can detect and use the full capabilities of your
machine, but at the very least I learned a thing or two about building
DPDK by looking at the the [Git repository][1] of the Ubuntu
maintainers. Just do a `git clone
https://git.launchpad.net/~ubuntu-server/dpdk`. Switch to the
`ubuntu-xenial` branch to see the interesting bits.

This seems to be a Canonical endeavor, so thank you kind folks in
Canonical!

[1]: http://dpdk.org