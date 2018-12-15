---
layout: page
title: "Linux Really from Scratch: Part 1"
---

[Linux from Scratch][1] has been one of the projects that I've always
been interested in...and never gotten around to actually going through
with it! I think I downloaded the book a few years ago and actually
read a couple of chapters, but never went past that.

So for one reason or another, during the last few days I've been
looking at how Linux is actually booted and how the user space is
launched, and I thought maybe I can do it all by myself. Without even
going through the LFS book.

So in this, hopefully, series of articles, I'm going to document the
process of building a very basic Linux system, all the pieces built
from scratch. This will be an iterative process in which we do things
step-by-step, in each step adding a bit more complexity.

This is what I'm trying to achieve with this series:

 - We'll take a look at how Linux actually boots.

 - We'll see the building blocks of the user space.

 - We'll try to see what it is the distros do for us. Hopefully we're
   going to get a lot more respect for the folks who do all that hard
   work for us!

 - I'll try to keep everything deterministic and repeatable.

 - My main focus will be creating an image that is run in a VM and
   accessed with SSH. So no graphics, at least, not for some time. SSH
   will also take a while to arrive, but I'll try to get there as soon
   as possible, since I really hate working in a console without a
   proper terminal.

## Where to begin?

We will start with two pieces: the Linux kernel and a tiny init
program. This will be our init:

```c
#include <stdio.h>

int
main(int argc, char *argv[])
{
    printf("Hello, World!\n");
    printf("This is your friendly init system.\n");
    printf("Just hanging here...\n");
    for (;;);

    return 0;
}
```

This will simply print out a message and then loop indefinitely, since
an init is not supposed to ever exit. We will need to compile this
statically. We don't have glibc or other dynamic libraries right
now. Compile the program by running:

    gcc hello.c -static -o hello
    strip hello

The resulting executable, `hello`, will be our primitive init.

Then we need to build the kernel. Get the source code for the stable
branch of the kernel:

    git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git

The latest stable version is v4.19.8 right now. Checkout that version:

    git checkout v4.19.8

I decided to simply use the config for current Ubuntu 18.04 kernel as
the basis. (Thinking about repeatability? You're right. More on that
later.)

Configure and build the kernel:

    cd linux
    cp /boot/config-$(uname -r) ./.config
    make oldconfig
    make

## Running what we have

We're going to use qemu to run what we have. But first, let's create
the root image.

Create an image file and mount it:

    qemu-img create image.img 2G
    sudo mount image.img /mnt

Now copy the `hello` executable to the mount directory and rename it
to `init`. Then unmount.

    cp hello /mnt/init
    sudo umount /mnt

All right. We're all set. Launch it all by running:

    qemu-system-x86_64 -kernel /path/to/bzImage \
                       -append "root=/dev/sda init=/init console=ttyS0" \
                       -hda /path/to/image.img \
                       -enable-kvm \
                       -nographic \
                       -serial mon:stdio

You need to fix the path to the kernel and the disk image. The kernel
should be in the `arch/x86_64/boot` directory after the build is
complete.

If everything is okay, you'll see the boot messages and at the end
you'll get the "Hello, World!" message from our "init system."

What's in this command?

 - `-kernel /path/to/bzImage`: specifies the kernel image to
   use. Using this option, we won't have to create a bootable disk, we
   just directly provide the kernel to boot.

 - `-append "root=/dev/sda init=/init console=ttyS0"`: adds a few
   options to the kernel command-line. `root` is the root file-system
   that is mounted on `/`, `init` is the path to the init program to
   use, and `console` specifies the output console device (needed in
   combination with the `-serial` option).

 - `-enable-kvm`: use KVM for virtualization.

 - `-nographic`: do not show the SDL window that is used as VM display
   by default.

 - `-serial mon:stdio`: redirect the serial port to stdio. This, in
   combination with the `console` parameter passed to the kernel,
   causes kernel output to be displayed on the current terminal.

## It's just the beginning

I had actually prepared a lot more material, especially about
repeatability and build automation, but since the article was getting
too long, I'll leave those for another article. Stay tuned.

[1]: http://www.linuxfromscratch.org/
[2]: https://github.com/elektito/electus.git
