---
layout: page
title: Building a static flanneld binary on Ubuntu
---

I just spent some time trying to build `flannel` and since there were
some nuances, I decided to list the instructions here.

1. Install build dependencies:

        sudo apt-get install linux-libc-dev golang gcc

2. Make Go directories:

        mkdir -p ~/go/src
        cd ~/go/src
        export GOPATH=~/go

3. Clone flannel:

        git clone https://github.com/coreos/flannel.git

4. Install Go dependencies:

        cd flannel
        go install

5. Since I wanted a statically linked binary, I edited the Makefile
   and updated the build instruction like this:

        dist/flanneld: $(shell find . -type f  -name '*.go')
            go build -o dist/flanneld \
               -ldflags '-extldflags "-static" -X github.com/coreos/flannel/version.Version=$(TAG)'

6. Now build the binary:

        make dist/flanneld

7. `flanneld` binary should now be created in the dist directory. You
   can strip it to make it smaller:

        strip dist/flanneld

One other problem I encountered was that you need at least 2GB of RAM
for this. I was trying this in a VM with 1GB and I ran out of memory.
