.. title: Getting started with the proxmark3 easy clone
.. slug: getting-started-with-the-proxmark3-easy-clone
.. date: 2021-03-27 09:08:06 UTC-04:00
.. tags: rfid, hacking, security, physical security
.. category:
.. link:
.. description:
.. type: text

Introduction
============

I've been interested in RFID hacking for a really long time.
The "gold standard" has been the proxmark series of tools.
Unfortunately, these are quite expensive, especially for a hobbyist.
However, thanks to the internet and the usual sources, there are pre-assembled versions available for under $100.


In this post, I'll share a quick tutorial on how to clone an access control card to a rewriteable card.

.. TEASER_END

The Hardware
============

The hardware I'm using was purchased from Amazon for $67.
I don't know what firmware was loaded on it, because it didn't seem to work initially.
The first thing I did was clone the `RFID Research Group <https://github.com/RfidResearchGroup/proxmark3>`_ repository (sometimes known as the iceman repository).
This contains both the proxmark software and the firmware for the device.
The firmware is loaded by running ``./pm3-flash-all`` in the repository after building.
One issue is that the USB port on the long side of the device does not seem to work.
I didn't take any notes nor do I remember the exact process I followed, but I think it was pretty straight forward once it was actually speaking with my computer.
I'm assuming that is for powering the device with something that can deliver more current than a standard USB port.

.. image:: /images/rfid_getting_started/device_edited.jpeg

----

I know there are supposed to be different modes to select, such as replaying cards that can be done without being connected to a computer, but I have not figured any of this out yet.
I also have no idea what the LED's mean.
If you know or have details on how to configure those features, please contact me via twitter `@haicenhacks <https://twitter.com/haicenhacks>`_.

The Software
============

The proxmark software can be run by simply launching from the terminal.

``proxmark3 /dev/ttyACM0``

Note that you may need to assign the correct permissions to your user account to enable access to tty devices.
For arch linux, this is accomplished by ``sudo usermod -aG uucp <username>``. `Arch Wiki <https://wiki.archlinux.org/title/Users_and_groups>`_.
For other operating systems, I believe this is accomplished by ``sudo usermod -aG dialout <username>``.
You will need to fully log out or restart for those changes to take effect.

Once everything is open, the screen should look like this:

.. image:: /images/rfid_getting_started/pm3_first_run.png

----

Great, so now that everything is configured, it is time to start investigating card types.

Identifying the card type
=========================

Before a card can be cloned, it must first be determined if it is even possible to clone.
Some card types, such as the MiFare DESfire EV1 have strong cryptographic protection.
Attacks on such cards are way out of my league.
I have only experimented with HID and EM410x cards.

Step one is to figure out if the card is 125 KHz (lf) or 13.56 MHz (hf).

``hf search`` will search for hf cards

If ``hf search`` is run when the card is actually an lf card, it will display ``[!] ⚠️  No known/supported 13.56 MHz tags found``

.. image:: /images/rfid_getting_started/hf_lf_search1.png

----

However, if we run ``lf search`` on the same card, it will return the following result:

.. image:: /images/rfid_getting_started/lf_search1.png

----

This shows a HID Prox card.

Running the command again with a different card type shows an EM410x card

.. image:: /images/rfid_getting_started/lf_search2.png

----

Because basic HID and EM410 cards do not employ any cryptographic protection, all that is needed to clone them is a single scan.
Most techniques for obtaing the card to scan are done using social engineering or other means.
This was shown on Mr. Robot Season 1 Episode 5.
On some HID cards, the card number is printed on the back of the card.
However, the facility code is not, and both pieces are required to clone a working card.
In "dumb" card reader systems, the facility code may be brute forceable, since it is only 3 digits.
More sophisiticated systems may lock out the reader if too many attempts are made.
There are other techniques, such as the BLE key  `Github <https://github.com/linklayer/blekey>`_, and `Blackhat <https://www.youtube.com/watch?v=3QK3LoovWxo>`_, which requires physical access to the card reader.


All the information needed to clone the card is printed by the command that was just run, but a physical card is needed to clone it to.
The T5577 card is a rewriteable card that can emulate HID, EM410x, Indala and a few others.
These can be purchased multiple places, including `Red Team Tools <https://redteamtools.com/electronic-attacks/access-control-RFID/T5577-rewritable-RFID-card>`_, Amazon, etc.
This type of card may also be referred to as a "magic" card for the hf versions (MiFare 1k)

Recall that the card we are cloning has the following data
::

  [+] [H10301] - HID H10301 26-bit;  FC: 0  CN: 5381    parity: valid
  [=] raw: 000000000000002006002a0b

The command to clone the card is ``lf hid clone -w H10301 --fc 0 --cn 5381``

The following sequence is all done with the same card.

Read what is currently on the T5577 card

::

  [usb] pm3 --> lf hid reader
  [+] [H10301] - HID H10301 26-bit;  FC: 118  CN: 6969    parity: valid
  [=] raw: 000000000000002004ec3672


Now write the data of the card to be cloned
::

  [usb] pm3 --> lf hid clone -w H10301 --fc 0 --cn 5381
  [=] Preparing to clone HID tag
  [+] [H10301] - HID H10301 26-bit;  FC: 0  CN: 5381    parity: valid
  [=] Done
  [?] Hint: try `lf hid reader` to verify

Note that ``lf hid clone -w H10301 --fc 0 --cn 5381`` is the same as ``lf hid clone -r 2006002a0b``.
If you had access to a "guest" badge, you could determine the facility code and then brute force or guess valid card numbers.




That's all there is to it. The T5577 card has been rewritten and appears to the reader as if it is identical as the previous card.

::

  [usb] pm3 --> lf hid reader
  [+] [H10301] - HID H10301 26-bit;  FC: 0  CN: 5381    parity: valid
  [=] raw: 000000000000002006002a0b


The process is similar for EM410x cards.
I don't have any other cards to experiment with at the moment.

Hopefully this is helpful to someone else.
