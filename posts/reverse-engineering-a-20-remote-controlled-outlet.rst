.. title: Reverse Engineering a $20 remote controlled outlet
.. slug: reverse-engineering-a-20-remote-controlled-outlet
.. date: 2021-08-14 19:47:14 UTC-04:00
.. tags: hacking, radio, SDR
.. category:
.. link:
.. description:
.. type: text


Introduction
============

A few years ago, I bought a remote controlled outlet from Walmart.
I don't recall the exact price, but it was less than $20.
It was pretty much purchased for the explicit reason of trying to understand how they worked.
The end goal, is that I wanted to know if

1) I could decode/intercept the signals
2) It was vulnerable to replay attacks

Tools Used
----------

* RTL SDR
* Yardstick One
* Inspectrum
* GQRX
* rfcat

This is going to be quite a long post, so I suggest getting a cup of coffee before continuing.

.. TEASER_END

Background/Motivation
=====================

Every day, we are bombarded with wireless signals, and I like to know what they are.
My passion for radios started very early on.
When I was about 5 years old, I had a set of walkie talkies that had a "base station" with a large antenna.
I believe operated on the 27 MHz band.
My grandmother's cordless phone was also on the 27 MHz band, or was close enough.
The first time I understood the power of radio was when I managed to listen to conversations.
I couldn't quite make out all the words, but there was enough for me to figure out that I was listening to a phone call.
In retrospect, I think the receive end of my walkie talkie was wide enough to get some of the signal from the phone.
I managed to find a picture on the internet of what I had.

.. image:: /images/rev_eng_outlet/walkie_talkie.jpg


The wireless outlet
===================

In order to understand how this system works, Let's look at the internals of the outlet and the remote.

The outlet end is secured with 4 screws.
The circuit board is pretty boring; a relay rated for 10A/240VAC, a DIP IC with "AUT980202-B", and a separate board soldered on at a right angle.
The smaller board has the antenna connected to it (the blue coiled wire)
I was unable to locate a datasheet for these components, but they are likely analog rf chips.
The only interesting thing is that the remote and the outlet both have the same chip.


Outlet and remote

.. image:: /images/rev_eng_outlet/remote_and_outlet.jpeg

----

The outlet board

.. image:: /images/rev_eng_outlet/outlet_board_front.jpeg

----

The remote board

.. image:: /images/rev_eng_outlet/remote_internal.jpeg

----

The datasheet wasn't really needed.
The FCC ID printed on the remote is enough to find the testing documents.

.. image:: /images/rev_eng_outlet/remote_fccid.jpeg

----

`fccid.io <https://fccid.io/PAGTR-009-1B>`_ (not affiliated with the FCC, just a more user friendly version)

The FCC report reveals the device operates on 315 MHz.

With that information, it is time to move on to actually looking at the signals.

Capturing the waveform
======================

The first step is determining the modulation and data stream looks like.
For that, I used my RTLSDR and GQRX to visualize the signal.

.. image:: /images/rev_eng_outlet/remote_fccid.jpeg

----

In that waterfall display, I clicked the on and off buttons several times.
The continous vertical lines in the waterfall can be ignored.
Those are harmonics of something else.

Now the modulation must be determined

Brief primer on modulation types
================================

Amplitude Shift Keying
----------------------

Amplitude shift keying (ASK) is the simplest modulation.
In this mode, the amplitude of the radio signal is modulated between 2 distinct amplitudes representing 0 and 1.
Sometimes this alternates between 1 and 0.5, or directly between 0 and 1.
The latter is referred to as On-Off Keying (OOK)
This presents a major problem for low power radios: the further away from the transmitter, the signal decreases in amplitude.
Far away, the 50% bit may be lower than the baseline RF noise.
This is less of a concern for OOK.
Personally, I have never encountered any ASK signals that are not OOK, but that doesn't mean they don't exist.

More detailed information can be found on `Wikipedia <https://en.wikipedia.org/wiki/Amplitude-shift_keying>`_

.. image:: /images/rev_eng_outlet/OOK-modulation.jpg

----

Frequency Shift Keying
----------------------

Frequency Shift Keying (FSK) is a bit different. ASK is similar to AM radio, FSK must be transmitted using FM.
FSK is a bit like an additional layer on top of FM.
FSK is identified by hearing 2^n distinct frequencies.
The most common variations are 2FSK and 4FSK.
As the name suggests, FSK shifts the frequency of the carrier wave between two distinct frequencies.
FSK is often visually identifiable in a waterfall display because the frequency shift will be apparent.

More detailed information can be found on `Wikipedia <https://en.wikipedia.org/wiki/Frequency-shift_keying>`_

.. image:: /images/rev_eng_outlet/FSK-modulation.jpg

----

Based on the waterfall from earlier, the remote is using amplitude shift keying.

Decoding the signal
===================

There are multiple ways of decoding the signal.
All require capturing the signal, either as raw I/Q samples, or as an audio file.
For ASK, GQRX can be put in AM recieve mode and the audio stream can be saved as a .wav file.
The .wav file can then be manually analyzed using Audacity.
This is quite time consuming, but may be worthwhile as a learning exercise.
The challenge is recovering the symbol rate of the data stream.


.. image:: /images/rev_eng_outlet/remote_fccid.jpeg

----

This image shows the .wav file captured.
There are two distinct amplitudes, low and high.
The "on" time can be measured for each position, and then translated to 1's and 0's.
This "on" time is used to calculate the symbol rate or baudrate.

:math:`symbol rate = \frac{1}{on time (seconds)}`

aa
