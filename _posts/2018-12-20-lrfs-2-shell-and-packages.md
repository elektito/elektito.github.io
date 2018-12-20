---
layout: page
title: "LRFS Part 2: Adding a shell and packages"
---

In this [first part][1] of this series we built a kernel and ran it
with a very minimal (and useless) init program. That's not very
useful. Let's add a shell.

But before that, let's backtrack a bit. Remember we briefly talked
about repeatability in the last post. Let's get back to that and later
see what it has to do with us wanting to add a shell to our distro.

## Repeatability

So what's repeatability and why is it important? Repeatability is the
quality of a quality of a process to assures us we arrive at the same
results every time we follow it, whether we do it now or ten years
later.

How are we supposed to do that? By documenting a record of every
little thing that has a meaningful impact on our results. This
includes configuration, build options, patches, environment, etc.

I am going to put everything in a git repository. Every piece (which
we are going to call a package) will be in its own sub-directory. In
that sub-directory, we'll have a file named `pkg.json` which describes
the package, and a `Makefile` that contains build and installation
instructions. The `pkg.json` file will look something like this:

    {
        "version": "1.0.0",
        "source": {
            "type": "local"
        }
    }

This tells the build script what the current package version number
and how to obtain the source code (local in this case, since the code
is included right there in the package directory).

The `Makefile` should contain at least the following targets: `all`
which will be used for building the package, and `install` which will
install the package files to a path determined in the `INSTDIR`
environment variable.

But this is not all that is needed for repeatable builds. Another
factor that might affect the build in longer periods of time, is the
tools in use. It might so happen, for example, that a warning added in
a new version of gcc breaks a build in which all warnings are
considered errors.

However, it sounds a bit impractical to me, to have tools like
compilers and linkers as part of the package. A better approach, could
be to use the same version of tools for all the packages at every
point in time. In order to do so, we'll simply describe the build
environment for all packages in the root of the `pkgs`
directory. We'll use a `build.json` file like this:

    {
        "env": {
            "name": "ubuntu",
            "version": "18.04"
        }
    }

Given that there are generally no breaking changes in development
tools in a single version of Ubuntu, this should work for now.

The directory tree looks like this at the moment:

    /
    /pkgs/
    /pkgs/build.json
    /pkgs/kernel/
    /pkgs/kernel/pkg.json
    /pkgs/kernel/Makefile
    /pkgs/kernel/config
    /pkgs/hello/
    /pkgs/hello/pkg.json
    /pkgs/hello/Makefile
    /pkgs/hello/hello.c
    /tools/
    /tools/build
    /tools/build-rootfs
    /tools/run
    /README.md

As you can see, we have two packages right now, `kernel` and `hello`,
both of which are in the `pkgs` directory. We'll also have a top-level
`tools` directory, which contains a script for building the packages,
a script for building a root file system from a list of packages, and
a script to run everything in qemu.

## The build script

The build script (aptly named `build`), located in the `/tools`
directory, builds one or more packages, according to the instructions
in the `pkg.json` file and the `Makefile`. Apart from "local" source
code, it also supports downloading a source tarball or obtaining it
from a git repository.

The build script also supports applying one or more patches to the
code before building it. For each package, a `.tar.xz` file is created
which contains all the files needed for installation.

The script needs a number of tools to run:

 - `jq`: A versatile, command-line JSON parser.
 - `awk`: For text processing.
 - `lsb_release`: For getting information about the build environment.
 - `fakeroot`: Runs another program in an environment with fake root
   privilege for file manipulation. This is needed because sometimes
   install scripts need to do change a file's group, something that
   only root can do, but we do not want our build system to run as
   root, hence we use fakeroot for creating the package. Extracting
   the packages, however, will obviously need root permissions.

In order to build the `hello` package, for example, go the `tools`
directory and run `./build hello`. When the script is done running,
you should have a `hello-1.0.0.tar.xz` package in your current
directory.

## And a shell

Here we are at last. We are going to build `bash` as our shell. This
is the `pkg.json` file to use:

```json
{
    "version": "4.4.23",
    "source": {
        "type": "dl",
        "location": "https://ftp.gnu.org/gnu/bash/bash-4.4.tar.gz",
        "inner_dir": "bash-4.4"
    },
    "patches": {
        "options": "-p0",
        "apply_dir": ".",
        "files": [
            "bash44-001",
            "bash44-002",
            "bash44-003",
            "bash44-004",
            "bash44-005",
            "bash44-006",
            "bash44-007",
            "bash44-008",
            "bash44-009",
            "bash44-010",
            "bash44-011",
            "bash44-012",
            "bash44-013",
            "bash44-014",
            "bash44-015",
            "bash44-016",
            "bash44-017",
            "bash44-018",
            "bash44-019",
            "bash44-020",
            "bash44-021",
            "bash44-022",
            "bash44-023"
        ]
    }
}
```

This is a bit more complicated than the one we saw before. Let's see
what it does. First, we are saying this is the `4.4.23` release of
bash, which happens to be the latest release at the moment. You can
find all bash releases [here][2].

After that, we determine how the source code is to be obtained. The
value `dl` for `type` means that a tarball is going to be
downloaded. The `location` field determines the download address and
the `inner_dir` field tells the build system where in the tarball the
source code resides.

We then have a list of patches, twenty-three of them, that need to be
applied to the `4.4` release, so that we arrive at `4.4.23` (all of
these are downloaded from the bash [release page][2] mentioned
before).

Then there's the Makefile:

```make
all:
    cd ../_src_ && ./configure --prefix=/usr --exec-prefix= --enable-static-link && $(MAKE)

install:
    $(MAKE) -C ../_src_ install DESTDIR=${INSTDIR}

.PHONY: all install
```

<sub>(Yes, my code coloring tool messes up the colors for this `make`
snippet. Ugly, but can't do nuffing about that right now!)</sub>

As you can see the `all` target configures and builds the code, while
the `install` target actually installs the files. There are a few
points to explain:

 - For building every package, a temporary directory created and the
   package directory is copied here and renamed to `pkg`. The source
   code, if obtained from an external source, is fetched and put in an
   `_src_` directory inside the temporary directory.

 - We configure `bash` so that it is linked statically. We still don't
   have `glibc` or any of the other shared libraries needed.

 - We use the `$(MAKE)` special variable instead of running `make`
   directly. This way, some of the properties of the parent `make` are
   communicated to the sub-makes (like the `-j` argument, so that the
   right number of parallel jobs are used).

## The source

Everything I've talked about in this article can be found in this
[repository][3] on Github. Why electus as the repo name? Well, I can't
do anything without a name, and I felt like calling this "electus" at
the moment!

[1]: /2018/12/15/linux-really-from-scratch
[2]: https://ftp.gnu.org/gnu/bash/
[3]: https://github.com/elektito/electus
