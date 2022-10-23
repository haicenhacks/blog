.. title: Fun with GUID's
.. slug: fun-with-guids
.. date: 2022-10-21 22:42:13 UTC-04:00
.. tags: node.js, guid, zap, web
.. category: hacking
.. link: 
.. description: 
.. type: text

Recently, Intigrity (https://twitter.com/intigriti) posted a challenge on Twitter.
I found this challenge to be pretty interesting, as I had not really heard of any issues regarding GUIDs (Global Unique IDentifier), sometimes also listed as UUID (Universally unique identifier)
These are all what I had previously assumed were essentially random and non-predictable.
Unfortunately, some versions of the UUID are not so random, at least for UUIDv1.

The rest of the post continues after the break.

.. TEASER_END

Background
==========

https://www.sohamkamani.com/uuid-versions-explained/ has some helpful illustrations that show how the UUID changes (or doesn't) across multiple generations.

The key to understanding which version of a UUID is by looking at the first digit of the 3rd group. See below

::

    59abef16-51ff-11ed-9dd6-b42e99f2932a
    xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx

In this case, M is 1, which corresponds to UUIDv1.
In contrast, a UUIDv4 is shown below:

::

    ca59a8a2-ff4c-4b2c-a99d-65e2f7e40641
    xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx

In this example, position M is 4, meaning this is a UUIDv4.

Fortunately, someone else has created a tool to inspect UUID's.

https://github.com/intruder-io/guidtool


Inspecting the two previous UUIDs:

::

    guidtool -i ca59a8a2-ff4c-4b2c-a99d-65e2f7e40641   
    UUID version: 4
    
    ---

    guidtool -i 59abef16-51ff-11ed-9dd6-b42e99f2932a
    UUID version: 1
    UUID time: 2022-10-22 11:47:48.130383
    UUID timestamp: 138857320681303830
    UUID node: 198112244306730
    UUID MAC address: b4:2e:99:f2:93:2a
    UUID clock sequence: 7638


What's interesting about UUIDv1 is that the MAC address is included in the UUID.
This is confirmed by looking at the MAC address of my ethernet adapter.

.. figure:: /images/fun-with-guids/mac_address.png

This could be useful in using flask debugging console.
A full write-up of exploiting the flask debugging console is a topic for a future discussion, but essentially there are only two pieces of information that are not guessable, and the MAC address is one of them.
By leaking the MAC address as part of the UUID, that leaves just the machine ID as unknown without local access to the system.

The Challenge
=============

Now, back to the challenge.
I modified the original script to make the viewing of transactions created by the non-admin user visible.
Follow the build instructions to make the app, then visit http://localhost:3000 to interact.

.. figure:: /images/fun-with-guids/homepage.png


The first thing to note is there are two account balances, one for my user, and one for admin.
To solve the lab, drain the whole balance from the admin.

I started by clicking the link to send the admin $1.
I can now list the transfers.

.. figure:: /images/fun-with-guids/list_transfers.png


The confirm link is a URL that points to ``http://localhost:3000/confirmTransfer/04a243c0-5214-11ed-bd4e-a9f1f88fd888``

Now I investigate the UUID using ``guidtool``

::

    guidtool -i 04a243c0-5214-11ed-bd4e-a9f1f88fd888
    UUID version: 1
    UUID time: 2022-10-22 14:15:44.892000
    UUID timestamp: 138857409448920000
    UUID node: 186856722389128
    UUID MAC address: a9:f1:f8:8f:d8:88
    UUID clock sequence: 15694


This gives all the information that is needed to forge the uuid when I send funds from the admin to myself.
To start, I'll initiate the transfer from admin to me.

.. figure:: /images/fun-with-guids/transfer_admin_haicen.png

Now I will generate the possible UUIDs for that transaction.

``guidtool  04a243c0-5214-11ed-bd4e-a9f1f88fd888 -t '2022-10-22 14:18:48' -p 10000 | xclip -sel clipboard``

Next, I fuzzed the ``confirmTransfer`` endpoint.

.. figure:: /images/fun-with-guids/fuzz_confirmation.png
    

This may take some time since there are 2000 possible UUIDs that could have been created.
The exact time the request was processed is unknown due to the precision difference between what is reported in the response and the true time.
In this case, it required over 1400 requests to find the right one.

.. figure:: /images/fun-with-guids/fuzz_success.png


Finally, looking at the main page:

.. figure:: /images/fun-with-guids/solved.png



Node.js quirk
=============

Previously, I stated that the last group of a UUIDv1 is the MAC address.
Node.js does not. Instead using a random 48-bit node ID (node in this context is unrelated to Node.js) as allowed in the official RFC [1]_.
Each time the Node.js server restarts, it selects a new random number.
However, the python uuid module does use the actual MAC address for one of the network interfaces.

Conclusion and Remediation
==========================

There are a few separate issues that lead to this type of exploit.
This lab is not a fully featured app, so authentication and authorization are not fully implemented.
In a real world application, there would need to be a series of checks performed.

1) Is the user authenticated?

2) Is the user authorized to make a transfer from the specified account?

3) Are sufficient funds available in the account?

4) Is the unique confirmation number cryptographicaly sound?

5) Is the user interacting with the confirmation authenticated?

6) Is the user interacting with the confirmation the same user for which the confirmation is assigned?

The central focus in this lab is exploiting condition 4.
Remediation is as simple as using UUIDv4 instead of UUIDv1.
UUIDv4 uses 122 bits of random numbers, with only 6 bits predetermined [5]_.
Shown below are 5 sequentially generated UUIDv4's.
The only constant digit is 4 in the 3rd group, which identifies it as a version 4.

::

    39fc832b-fc05-4e89-9847-8c6ba359e71d
    7a17bcdf-b2a3-4a53-ac34-2b0a3ad0621e
    8dbce5c6-2eb8-418b-8e2a-bca81f4f8e82
    0db75846-6a50-4099-a743-d1a080fec409
    2ad8400f-c8bb-413b-97ed-39c4457097fe

In contrast, UUIDv1 has 6 bits for version and variant information, 48 bits for the MAC address/node ID, 14 bits for the clock sequence (68 bits).
The timestamp is another 60 bits.
When the timestamp can be known or approximated to the nearest second, that leaves at most 6 digits of unknown data.
Below are 5 sequentially generated version 1 UUID's.

::

    0bef9616-52cb-11ed-9e93-b42e99f2932a
    0bef9904-52cb-11ed-9e93-b42e99f2932a
    0bef99f4-52cb-11ed-9e93-b42e99f2932a
    0bef9aa8-52cb-11ed-9e93-b42e99f2932a
    0bef9b52-52cb-11ed-9e93-b42e99f2932a

Only 3 hex digits are different in each one.

References
==========

.. [1] https://www.ietf.org/rfc/rfc4122.txt (Accessed Oct. 22, 2022).

.. [2] INTIGRITI [@intigriti], “Can you spot the vulnerability? https://t.co/7gJZJHkjYd,” Twitter, Oct. 20, 2022. https://twitter.com/intigriti/status/1583060520835293185 (accessed Oct. 22, 2022).

.. [3] “In GUID We Trust.” https://www.intruder.io/research/in-guid-we-trust (accessed Oct. 22, 2022).

.. [4] Installation. Intruder, 2022. Accessed: Oct. 22, 2022. [Online]. Available: https://github.com/intruder-io/guidtool

.. [5] “Universally unique identifier,” Wikipedia. Oct. 17, 2022. Accessed: Oct. 22, 2022. [Online]. Available: https://en.wikipedia.org/w/index.php?title=Universally_unique_identifier&oldid=1116582443#Version_1_(date-time_and_MAC_address)

.. [6] “What is a GUID.” http://guid.one/guid (accessed Oct. 22, 2022).

