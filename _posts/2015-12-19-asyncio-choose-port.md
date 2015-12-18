---
layout: page
title: Letting AsyncIO choose a port for you
---

Say you are writing a test case for an AsyncIO-based network
function. You want to write a test server and have the code being
tested connect to it. You can choose a port number and hope it's not
taken when the test is run, or you can have a free port chosen for you
each time the test is run. Simply pass `0` as the local port number:

    {% highlight python %}
    transport, protocol = await loop.create_datagram_endpoint(
        ServerProto, local_addr=('127.0.0.1', 0))
    {% endhighlight %}

Afterwards, you can query the transport object for the port number
chosen for you:

    {% highlight python %}
    server_addr, server_port = transport._sock.getsockname()
    {% endhighlight %}

And that's it; now you know the port number. I'm not sure if using a
member variable with an underscore at the start of its name is the
best or the official way to do this but it's the best method I've
found so far.
