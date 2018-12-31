---
layout: page
title: "LRFS Part 4: Early Userspace: initrd and initramfs"
---

Although [init][1] is considered the beginning of the Linux userspace,
this is not technically the case. There are other facilities that are
part of the userspace and run even before init. Using these is not
mandatory, and as it happens we are not going to use them in our
distro (not now, at least), but I thought it might be educational to
examine them here.

## Why?

So why do we need something to run before init? Here are a few cases
that this might be necessary:

 - Mounting the root file system might need access to kernel modules
   that are not built into the kernel, but are instead built as kernel
   modules and reside on the very file system we are going to
   mount. It is, of course, possible to built these into the kernel,
   but we might want to keep the kernel from becoming too large. This
   was especially the case before, when RAM used to be more limited
   than it is now.

 - The root file system might be encrypted and it might need being
   decrypted before being mounted.

 - The system might be in hibernation and need special treatment
   before waking up.

The kernel provides two facilities for running a small userspace
before the actual init: these are called `initrd` and `initramfs`, the
latter being a more recent addition to the kernel, although both have
been there for quite some time now.

The names `initrd` and `initramfs`, although referring to distinct
facilities, are frequently used interchangeably.

 - `initrd` is a file system image that is mounted as root. It usually
   contains an executable named `/linuxrc` that is run after the image
   is mounted. After performing any necessary preparations and
   mounting the real root file system in a temporary location, this
   program then uses the `pivot_root` system call to switch to the new
   root and then unmounts initrd.

 - `initramfs` is a file archive that becomes the root file system. An
   executable called `/init` on this archive is then run by the
   kernel, effectively becoming init (that is, PID 1). This can
   continue as an init, or later mount a new root and `exec` the real
   init.

In the next two sections, we will examine each of these carefully and
see some examples.

## initrd

We will be using the following C program as the "early init":

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int
main(int argc, char *argv[])
{
  printf("Early Init\n");
  printf("----------\n");
  printf("PID=%d UID=%d\n", getpid(), getuid());
  printf("----------\n");
  for(;;);

  return 0;
}
```

Save this as `earlyinit.c` and compile it statically:

    gcc earlyinit.c -static -Wl,-s -o linuxrc

Now create a file system image and copy linuxrc to it:

    dd if=/dev/zero of=image.img bs=20M count=1
    mke2fs image.img
    sudo mount image.img /mnt
    sudo cp linuxrc /mnt
    sudo umount /mnt

You can also compress this file:

    gzip -9 image.img

You probably won't be able to use your distribution's kernel for this
as support for initrd is probably not built into the kernel. Build a
new kernel from source and make sure the `CONFIG_BLK_DEV_INITRD` and
`CONFIG_BLK_DEV_RAM` options are set to `y` in the `.config` file.

Like in previous sections, we will be using `qemu` to run the kernel
and initrd. `qemu` has a `-initrd` option that we can use:

    qemu-system-x86_64 -kernel /path/to/bzImage \
                       -initrd image.img.gz \
                       -enable-kvm \
                       -append "console=ttyS0" \
                       -nographic

Take a look at the output. Notice the PID reported in the output. It
is not 1. An initrd is not an init.

A real `initrd` would mount the root file system as part of its
work. When initrd returns, the kernel assumes that root is already
mounted and so proceeds to running `init` (from `/sbin/init`, for
example).

Let's try this. Update the C program above like this:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <string.h>

int
main(int argc, char *argv[])
{
  printf("Init\n");
  printf("----------\n");
  printf("PID=%d UID=%d argv[0]=%s\n", getpid(), getuid(), argv[0]);
  printf("----------\n");
  if (strcmp(argv[0], "linuxrc") == 0) {
    /* running as initrd */
    return 0;
  } else {
    for(;;);
  }
}
```

We are going to use this as both `linuxrc` and `init`. Recompile, like
before and update the image like this:

    gunzip image.img.gz
    sudo mount image.img /mnt
    sudo cp linuxrc /mnt
    sudo mkdir /mnt/sbin
    sudo cp linuxrc /mnt/sbin/init
    sudo umount /mnt

We won't be compressing the image this time, since `qemu` does not
accept a compressed image as an argument to `-hda`. Run `qemu` like
this:

    qemu-system-x86_64 -kernel /path/to/bzImage \
                       -initrd image.img \
                       -enable-kvm \
                       -hda image.img \
                       -append "console=ttyS0 root=/dev/sda" \
                       -nographic

Again we are not actually mounting the real root file system here. So
in this case, when initrd returns, the kernel runs `/sbin/init` from
the already mounted RAM disk (i.e. initrd iteself). In the output, you
will see two invocations of our program, one as `linuxrc`, the other
as `/sbin/init`.

## initramfs

Originally, initramfs is supposed to be a an archive embedded into the
Linux kernel itself. This archive is mounted as root and a `/init`
file inside it is executed as init (i.e. with PID 1).

What this incarnation of early init does is slightly different from
initrd. First of all, initramfs cannot be unmounted. It usually
deletes all of its contents in the end, however, chroots into the real
root file system and invokes init using one of the `exec` system
calls. `pivot_root` cannot be used in initramfs. `klibc` and `busybox`
each have a utility (called `run-init` and `switch-root` respectively)
that helps initramfs writers with the usual tasks (deleting files,
chroot and exec, among other things).

In order to try this, we'll revert to the original version of our
simple early init:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int
main(int argc, char *argv[])
{
  printf("Early Init\n");
  printf("----------\n");
  printf("PID=%d UID=%d\n", getpid(), getuid());
  printf("----------\n");
  for(;;);

  return 0;
}
```

Compile this as init:

    gcc earlyinit.c -o init -static -Wl,-s

Now we have to create an archive, instead of a file system image. This
is a cpio archive. The concept is very similar to the more widely used
tar archive. Let's create an archive with init as its only content:

    echo init | cpio -o -H newc | gzip -9 >initramfs

The `-H` flag specifies the variant of cpio that the kernel uses. Now,
as we said before, initramfs is an archive that is embedded inside the
kernel and indeed when building a kernel you can configure an
initramfs to be built inside the kernel. However, there is a simpler
way of doing this as when the kernel receives a cpio archive instead
of a file system image for its initrd parameter, it uses the archive
as if it is a built-in initramfs archive.

So in effect, you can pass the initramfs archive like an initrd to
qemu. Let's try it:

    qemu-system-x86_64 -kernel /path/to/bzImage \
                       -initrd initramfs \
                       -enable-kvm \
                       -append "console=ttyS0" \
                       -nographic

You will see that this time our program is run with PID 1. It can
simply do its work and exec the real init in the end.

## Tools for initramfs writers

If writing an initramfs in C, in many cases, alternative C standard
libraries are used in place of glibc which is feature-rich and very
large. `musl` is one popular implementation of the C standard library
that is used when size is important. `klibc` is another which, although
not implementing the full extent of the standard library, has been
specifically written for writing an early init. Both provide wrapper
scripts for building against them. For musl, you can use the
`musl-gcc` script:

    musl-gcc earlyinit.c -static -Wl,-s -o linuxrc

while for `klibc` you can use `klcc`:

    klcc earlyinit.c -static -Wl,-s -o linuxrc

Both provide very smaller executables than when linking against glibc.

In many cases, the early init program is in fact written in shell
script, so a shell and a number of utilities are included. You can use
bash and GNU coreutils for this, but again these are quite large and
all of their features is probably not necessary for the small
initramfs script.

`busybox` is one alternative, which includes a shell, and a large
number of utilities, including the previously-mentioned `switch-root`.

`klibc` also comes with a number of utilities, which are more limited,
and smaller, than the ones with busybox. It also includes the
`run-init` utility which helps with wrapping up the work in initramfs.

## A note on Ubuntu's initramfs

If you try taking a peek at the initramfs on an Ubuntu system with
cpio, a few files are extracted and then you'll receive an error
message. At least, that's how it is on Ubuntu 16.04 and 18.04 where I
tried this. This is because the Ubuntu initramfs is actually two cpio
archives put one after the other in a single file.

Ubuntu 18.04 comes with an `unmkinitramfs` utility (installed with the
initramfs-tools-core package) capable of extracting the contents of
this initramfs. It's a shell script so you can take a look at it and
see how it actually works.

## Wrapping up

[supermin][2] contains a small initramfs program that can be very
educational to look at. Just get the source code and open the
`init/init.c` file.

I also found the following links very informative:

 - <https://www.kernel.org/doc/Documentation/early-userspace/README>
 - <https://www.linux.com/learn/kernel-newbie-corner-initrd-and-initramfs-whats>
 - <https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt>

As I said in the beginning, we are not going to use either initrd or
initramfs in our distro-to-be. So we'll just carry on.

[1]: /2018/12/21/lrfs-3-init/
[2]: https://github.com/libguestfs/supermin
