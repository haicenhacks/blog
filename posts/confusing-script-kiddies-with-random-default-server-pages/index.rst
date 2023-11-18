.. title: Confusing script kiddies with random default server pages
.. slug: confusing-script-kiddies-with-random-default-server-pages
.. date: 2022-01-08 17:58:02 UTC-05:00
.. tags:
.. category: fun, php
.. link:
.. description:
.. type: text

Anyone who runs a server connected to the internet can tell you that they get hammered all day every day by bots.
Sometimes, a human is poking at things looking for vulnerable applications.
Quite a while ago, I created a simple php script to randomly pretend to be one of the 3 web servers (Apache, Nginx, IIS).

Due to copyright, I am not sharing the source code from the html page for IIS.
They are pretty easy to find without downloading/installing the package.
The NGINX source is available on `Github <https://github.com/nginx/nginx/blob/master/docs/html/index.html>`_
I tried to find the one served by Apache2, but could not find the page I wanted.
Fortunately, it happened to be available on one of my other servers so I just copied it over from there.

The php script is included below:

.. gist:: 49e8cca8f0a81c68c8f62e33a8d29d33
