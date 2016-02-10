---
layout: page
title: Controlling rtorrent with XML-RPC
---

I'm a huge fan of [rtorrent][1]. It's simple, lean and elegant. I have
so far only used it as an interactive BitTorrent client inside
screen/tmux or as a batch downloader with a watch directory.

There is more to rtorrent however. It supports XML-RPC which means you
can control it programmatically. As I was in dire need of a client I
can manipulate from a script, I spent some time today to setup
rtorrent correctly and call it remotely.

First we're going to need something like this line in our
`.rtorrent.rc` file.

    scgi_port = localhost:5005

You can also make rtorrent listen to a UNIX socket:

    scgi_local = /tmp/rtorrent-listening-here.sock


After that, we need a web server to forward our requests to rtorrent,
and responses back to us. The web server will be communicating with
rtorrent through [SCGI][2]. I use [nginx][3] for this purpose. Here's
the relevant nginx config for this:

    server {
        listen 8000;
        server_name localhost;
        root /rtorrent/;
        location ^~ /RPC2 {
            include scgi_params;
    #        scgi_pass  unix:/tmp/rtorrent-listening-here.sock;
            scgi_pass   127.0.0.1:5005;
        }
    }

After this, reload nginx config, make sure rtorrent is running, and
you're ready to go.

In order to test your setup, or contact rtorrent from a shell script,
you can use the `xmlrpc` utility accompanied by the `libxmlrpc`
library. You can get this in Ubuntu by running `sudo apt-get install
libxmlrpc-core-c3-dev`. After that, run this:

    xmlrpc localhost:8000 system.listMethods

If everything is done correctly, you will get a list of RPC methods
supported by rtorrent.

Some useful commands:

 - Add a torrent file and start downloading it:

        xmlrpc localhost:8000 load_start ~/foo.torrent

 - See list of current downloads:

        xmlrpc localhost:8000 download_list

 - Start a torrent:

        xmlrpc localhost:8000 d.start 07BF391725A0891F3A1E84784B078D5EF337F82F

 - Stop a torrent:

        xmlrpc localhost:8000 d.stop 07BF391725A0891F3A1E84784B078D5EF337F82F

 - Get the status of a torrent:

        xmlrpc localhost:8000 d.state 07BF391725A0891F3A1E84784B078D5EF337F82F

There are a lot more commands. Try some of them for yourself. I admit
that it's difficult to find out how each command works this way, but
in the absence of a comprehensive documentation, it seems to be the
only way. There are a few examples [here][4].

## Complete Vagrant Setup

I have put the complete setup for use with vagrant in this
[github repository][5]. Simply clone the repo, and run `vagrant up` in
its directory. After the VM is up, you can access rtorrent on the
forwarded port `8080`:

    xmlrpc localhost:8080 load_start /vagrant/foo.torrent

[1]: https://rakshasa.github.io/rtorrent/
[2]: https://en.wikipedia.org/wiki/Simple_Common_Gateway_Interface
[3]: http://nginx.org/
[4]: https://github.com/rakshasa/rtorrent-doc/blob/master/RPC-Setup-XMLRPC.md
[5]: https://github.com/elektito/rtorrent-xmlrpc-sample
