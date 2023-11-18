.. title: Configuring custom headers for OWASP ZAP
.. slug: configuring-custom-headers-for-owasp-zap
.. date: 2022-06-18 08:47:43 UTC-04:00
.. tags: hacking, tutorial
.. category: hacking
.. link:
.. description:
.. type: text

I've been trying some bugbounty programs recently.
I often alternate between using BurpSuite and ZAP.
Many programs want you to add a custom header to your requests so the traffic can be identified, and in some cases, bypass some roadblocks.
In this post, I'll show how to configure ZAP to add the custom header.

.. image:: /images/zap-header/teaser.png


.. TEASER_END


At first, I was pretty confused about how to do this.
Through some googling and some github issue searching, I found the answer.

Step 1: Open the scripts pane. It may not be visible, so click the + icon next to sites.

.. image:: /images/zap-header/scripts_pane_hidden.png

Step 2: Right click on "HTTP Sender" and create a new script. Choose the `AddZapHeader.js` as the template, check the enable box and add a description (optional)

.. image:: /images/zap-header/scripts_new.png

Step 3: Edit the script to add the header(s) desired.

.. image:: /images/zap-header/scripts_modify.png

Step 4: In the scripts pane, right click the script you created and click "save"

Step 5: Test it

.. image:: /images/zap-header/scripts_test.png
