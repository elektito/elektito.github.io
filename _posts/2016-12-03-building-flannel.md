---
layout: page
title: Building a static flanneld binary on Ubuntu
---

I just spent some time trying to build `flannel` and since there were
some nuanced, I decided to list the instructions here.

1. Install build dependencies:

    {% highlight shell %}
    sudo apt-get install linux-libc-dev golang gcc
    {% endhighlight %}

2. Make Go directories:

    {% highlight shell %}
    mkdir -p ~/go/src
    cd ~/go/src
    export GOPATH=~/go
    {% endhighlight %}

3. Clone flannel:

    {% highlight shell %}
    git clone https://github.com/coreos/flannel.git
    {% endhighlight %}

4. Install Go dependencies:

    {% highlight shell %}
    cd flannel
    go install
    {% endhighlight %}

5. Since I wanted a statically linked binary, I edited the Makefile
   and updated the build instruction like this:

    {% highlight make %}
    dist/flanneld: $(shell find . -type f  -name '*.go')
        go build -o dist/flanneld \
          -ldflags '-extldflags "-static" -X github.com/coreos/flannel/version.Version=$(TAG)'
    {% endhighlight %}

6. Now build the binary:

    {% highlight shell %}
    make dist/flanneld
    {% endhighlight %}

7. `flanneld` binary should now be created in the dist directory. You
   can strip it to make it smaller:

    {% highlight shell %}
    strip dist/flanneld
    {% endhighlight %}

One other problem I encountered was that you need at least 2GB of RAM
for this. I was trying this in a VM with 1GB and I ran out of memory.
