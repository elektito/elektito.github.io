---
layout: page
title: Benchmarking Python XML Parsers
---

I've written a small benchmarking tool for some of the different XML
parsers available to Python programmers. It calculates each option's
throughput by sending a large amount of XML data to each parser. You
need to provide it with some XML input.

    $ ./pyxmlperftests.py 1.xml 2.xml 3.xml 4.xml

You can find the source code [here][1] on Github.

These are the results on my computer:

    Results:
       xml.dom.minidom: 7.49 MBps
       lxml.etree: 89.63 MBps
       xml.etree.ElementTree.iterparse: 31.77 MBps
       xml.etree.ElementTree: 58.43 MBps
       xml.sax: 25.68 MBps

As you can see, [lxml][2] rocks. Although, to be honest, I'm still
looking for something faster than that!

A word of warning. I don't claim this is in anyway a fair and
scientific benchmark. I just wanted to see how these relatively
compare and cooked this script to get me some numbers.

[1]: https://github.com/elektito/pyxmlperftest
[2]: http://lxml.de
