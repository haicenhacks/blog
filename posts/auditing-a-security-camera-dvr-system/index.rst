.. title: Auditing a $50 Security Camera DVR System
.. slug: auditing-a-security-camera-dvr-system
.. date: 2021-03-19 11:21:26 UTC-04:00
.. tags: hacking, IoT
.. category:
.. link:
.. description:
.. type: text

FYI, this is a rewrite of some work I did in 2016 and was previously hosted on my old blogspot account. In doing research for the rewrite, I found that several people had done largely similar work, but I have identified some new information, particularly the ability to **format the disk drive by sending a post request**.

Introduction
============

Before the Internet of Things (IoT) took over, homes and businesses were watched by closed circuit cameras, and some did have LAN and WAN remote viewing. The WAN connection was not through a cloud system, but rather a direct connection to the IP address of the device. Unfortunately, given the price point of these devices, a lot of security corners were cut.

On a whim, I ordered one of these devices (it was on sale at Newegg for $50). The specific device was from `Newegg <https://www.newegg.com/shieldeye-rscm-0704b042-4-channel-kit-solution/p/N82E16881147043?Item=N82E16881147043>`_, (`Archive.org backup <https://web.archive.org/web/20210319155009/https://www.newegg.com/shieldeye-rscm-0704b042-4-channel-kit-solution/p/N82E16881147043?Item=N82E16881147043>`_). It had a lot of neat features (night vision, motion detection based recording, 720p resolution), but I mostly bought it because I wanted to see just how bad the security was.

Project Scope
=============

The goal is to emulate an attacker could disable or delete recordings with no knowledge of the system (i.e. no physical access to the device or model numbers), with the assumption that the attacker is already on the same network as the device. If that isn't successful, the scope will be revised to assume attacker knows the model number of the device.

.. TEASER_END


Previous work
=============

I am not the first person to look at these devices, however I haven't seen any work on this specific model. While doing research for the rewrite, I found several pages and blogs about this topic (including a metasploit module). What first got me interested in this topic was a presentation at Blackhat [1]_.


Recon
=====

The first step is an nmap scan of the network to locate the DVR.

After locating the IP address, a further nmap scan of the host reveals the following ports are open and listening

``nmap -sS 192.168.1.10``

.. image:: /images/dvr_audit/nmap_scan.png

----

- Port 23 is a telnet interface. This is probably useful, but at this stage I don't know the password.
- Port 80 is http, so I opened the page. It was a simple login form with no details about what sort of device (model number, manufacturer, or other identifying information).
- Port 554 is for RTSP video. This is an unauthenticated live view of the cameras. The URL that makes this possible is rtsp://192.168.1.10:554/h264?ch=1&subtype=1, where ch=1 selects which channel. I'm not sure what the subtype parameter might do, but it only seems to work if it is set to 1.
- Port 8000 doesn't seem to actually do anything. Opening it in a web browser and netcat, both yield nothing. There is no response from the device. I believe this is the port that is used by the mobile app (that I was too sketched out to install)
- Port 49152 has some UPnP information, nothing relevant as far as I can tell.



The Web App
------------

The web interface is pretty simple. The page itself is a java server page. The login box is processed by some javascript and posted to a page called "pro_login.do". Attempting to brute force passwords through either the main login page or by posting data directly to "pro_login.do" will cause the username to be locked out for some period of time. The login page will also alert if the username does not exist (which would allow some enumeration of the device users). Further investigation of the javascript indicates that the login information is stored in a cookie along with a session id. This would be exploitable if I can sniff credentials from an open wifi network by using wireshark and a wifi adapter that supports monitor mode.

.. image:: /images/dvr_audit/web_login.png
  :align: center

----

The login page is actually http://192.168.1.10/new/index.jsp, which is an interesting URL. There is indeed a /old/index.htm file, but it tries to load some ActiveX control. The underlying logic points to the same "pro-login.do" endpoint, so I didn't spend any further time on this for now.

There are a few other directories that are accessible, most notably /admin, but every link on the page returns a 404.


Unfortunately, without any more information, there isn't much that can be done. I could try default credentials on the web app, or try brute forcing the password on the telnet login (but this could be quite time consuming with no guarantee of success in a reasonable time).

At this point, I decided to modify the scope slightly to be able to proceed further. On the bottom of the cameras themselves is a sticker with a model number. The model number is RSCM-0700B011 and was enough to identify the retailer's page for the DVR system. That can lead to more specific information about the device.



Further Recon
==============

Now that the DVR system model is known, the next step is to see if the manufacturer has any firmware available to download. This was the case in 2016 when I was doing the initial research and I was still able to find a version of the firmware still available in 2021 [5]_.

The firmware was downloaded and then extracted using binwalk. It was a standard cramfs file system, with multiple cramfs images inside. These images are actually uboot images, so they can be mounted by first stripping 64 bytes using ``dd`` [3]_.

  ``dd bs=1 skip=64 if=romfs-x.cramfs.img of=rom2.img``

Unpacking firmware is a common technique for embedded devices [6]_, [7]_, [8]_.

Looking at ``/etc/passwd`` shows only one hash, ``root:$1$$64lU4r1qa6icjzK/sBmQo.:0:0::/root:/bin/sh``

I tried with the standard password lists (rockyou.txt, etc), and nothing worked, so my assumption is that it was chosen at random. I ran John for a few hours with no success. I did a web search for that string and it was found on some Russian password cracking forum (no longer accessible, but this is how I did it originally). Since it was included in the firmware, this is likely the password on every device. Given the performance power of modern GPU's, this is a trivial hash to crack (md5crypt)

The password for the hash is "zlxx." - only 5 characters, all lowercase, one special character. That was indeed the correct password because I am greeted by a root shell.

.. image:: /images/dvr_audit/telnet_login.png
  :align: center

----

.. image:: /images/dvr_audit/hackerman.gif
  :align: center


The telnet interface
====================

The telnet interface is not mentioned in any of the product literature or manuals, nor is there any reference to this being enabled in the local interface. This is problematic because the only layer of protection would be the router's firewall.

The DVR system is a bit of a labyrinth of files. Below is a list of interesting file locations:

- ``/etc/passwd`` - the telnet password previously obtained
- ``/etc/passwd-`` - no idea what this is for. The hash contents is ``root:ab8nBoH3mb8.g:0:0::/root:/bin/sh``, but that is easily cracked to "helpme". This does not work on the telnet prompt or the web app, so I'm not sure why it's function is.
- ``/opt/sav/Config/passwd`` - this is the file that gets modified when users are added, more on this below.
- ``/opt/sav/encrypt_info`` The contents of this file is pretty bare, and makes me wonder if the recordings are encrypted in some way. contents shown below

  - ::

        ethaddr e0:61:b2:14:93:fd
        product 4b27040000000000
        oem 000015


- ``/sbin/poweroff`` - this may be useful for disrupting the recording process


Now that I have a root shell, I can poke around and try to accomplish the stated goal of disrupting video recordings. I started by seeing if I could access the hard drive directly. I did not have any success. The disk is /dev/sda, but it is not mounted. Most of the directories are read only. However, in the few directories that are writeable, I cannot mount the drive.


.. image:: /images/dvr_audit/fdisk.png
  :align: center

----

.. image:: /images/dvr_audit/mount.png
  :align: center

----

Since deleting the recordings directly isn't possible, the next step was to look at the ``/opt/sav/Config/passwd`` file. This file holds the passwords for accessing the web interface and the local interface. These are hashed, but curiously there is one more user present in the file than shows up in the admin interface. Because the web interface requires ActiveX to run, and I can't get InternetExplorer to run the ActiveX controls under wine, some of the remaining screenshots have to be taken with a phone pointed at a screen. For comparison, the local interface shows 4 users:

- admin
- user
- default
- test

The first three users are enabled by default. Comparing that to the file that is accessible via telnet, there is an extra user called `genius` with a password hash of ``OzYqRThN``. The passwords for the web interface are hashed using the "dahua" algorithm. Dahua is/was a manufacturer/brand name of similar devices. This can be cracked using john, and the result is ``Q5M2Zk`` (note that this user has full admin access).

.. image:: /images/dvr_audit/user_list.jpg
  :align: center

----

In order to crack these hashes using john, the hash format needs to be altered slightly. The original file looks like this:

::

  1:admin:nTBCS19C:1:<permissions>:123456:1
  2:user:d2gNs1nj:2:<permissions>:1234:0
  3:default:1cDuLJ7c:2:<permissions>:default account:0
  4:genius:OzYqRThN:1:<permissions>:Q5M2Zk:1
  8:test:Qw23AbMB:1:<permissions>:0

The format is ``id, username, password hash, group, <permissions>, memo, <unkownn>``. I'm not sure what the last field indicates.

John requires the hash file to look like this:

::

  $dahua$nTBCS19C
  $dahua$d2gNs1nj
  $dahua$1cDuLJ7c
  $dahua$OzYqRThN
  $dahua$Qw23AbMB


The default password for the admin user is ``123456``. These can be found in the user manual. Admin level privileges is needed to make any changes. Other users can view recordings.

It is possible to modify this file. I previously wrote a hash generator so it is easy to create your own password, change the admin password to something else, delete users, change permissions, and otherwise disrupt the end user of the device (https://github.com/haicen/DahuaHashCreator). Through the local interface, the user can only change the password for themselves. The admin user cannot change the password of other users, nor can

The ``/sbin/poweroff`` is a symbolic link to busybox. Executing this command puts the device into a zombie-like state. The local interface will continue to show the live feed, but the menu is inaccessible, and *all recording stops*. The only way to get things running again is to power cycle the device.

Without doing some reverse engineering of the underlying binaries, I didn't find anything else.

The web app
===========

Using either the ``admin`` or ``genius`` passwords, I can now log in to the web interface. Unfortunately, neither the live view, nor any of the buttons work, because this seems to rely on components that aren't available or no longer work (ActiveX). The login information is indeed stored as a cookie and is posted as form data.

.. image:: /images/dvr_audit/http_post_data.png

Looking at the URL, there is a parameter called ``sid`` with a numeric value. I assume this to mean session ID.

A further review of the page source, particularly the ``commands.js`` file, there are interesting function names.

- ``pro_login`` - this sends a post request to `pro_login.do` with the username and password (in clear text) and returns the session id.
- ``pro_getEmailData`` - the system also has the capability to send email alerts on motion detection, so if that is configured, this should provide those credentials, but instead it causes the system to crash.
- ``pro_getFTPDataById`` - similar idea as getEmailData, but again causes a crash
- ``pro_addUser`` - looks like it should add a user, but also causes a crash
- ``pro_getCloudToken`` - again, sounds interesting but causes a crash
- ``pro_diskFormat`` - this causes it to crash, but when it reboots, it does actually format the disk (maybe).

The pro_login endpoint
----------------------

Code from ``command.js`` is shown below
::

    this.pro_login = function (__name, __password) {
        http.setAsync(false);
        return http.PostData("/pro_login.do", "{\"username\":\"" + __name + "\",\"password\":\"" + __password + "\"}", null);
    };

From that, I thought I could do ``curl http://192.168.1.10/pro_login.do -d '{"username": "admin", "password": "123456"}'``. It did work, and I got the following response: ``{"message":"Login Is Successful","sid":1}``

Just to recap, the username and password here could be found by logging into the device with telnet, or by sniffing open wifi networks because the device does not support https. If I happened to be on an open guest network with someone viewing or managing the device, I could grab the cookie that contains the username and password.

The pro_diskFormat endpoint
---------------------------

The code for this endpoint is provided below. Of note, it only needs a valid session ID. It isn't really practical to try to brute force the sid, because each time the url is hit, it causes it to crash. For some reason, it does actually cause the disk to be formatted upon reboot. I'm not sure if it actually formats the disk, or if it just marks everything for deletion. I'm not really a reverse engineer, so I haven't been able to see what is actually happening in the binary.

::

    this.pro_diskFormat = function(_id){
        http.setAsync(false);
        return http.PostData("/pro_diskFormat.do", "{\"sid\":"+this.getSessionId() +",\"id\":"+_id+"}", null);
    };



My test methodology was:

- Wave to the camera to trigger the motion recording
- Verified the recording was made and viewable
- Used the pro_login endpoint with curl to get a valid session id
- Used the pro_diskFormat endpoint with curl and the valid session id
- Logged in through telnet to reboot the device (easier than unplugging it)
- Logged in to the local interface and found no recordings exist

::

  > curl http://192.168.1.10/pro_login.do -d '{"username": "admin", "password": "123456"}'
  {"message":"Login Is Successful","sid":1}

  > curl http://192.168.1.10/pro_diskFormat.do -d '{"sid": "1"}'
  curl: (52) Empty reply from server




The metasploit module
---------------------

While not specific to this device, there is a metasploit module developed by Depth Security [2]_, and there is a CVE 2013-6117 [4]_  issued for this particular exploit. I have confirmed that some functions do work, while others do not. The two that have the highest impact are the EMAIL and NAS actions. This is shown below, where the smtp username and password are visible.

.. image:: /images/dvr_audit/metasploit.png
  :align: center

----

The ones that I tested that worked are CHANNEL, EMAIL, NAS, GROUP, and USER. The email action does return the  This must use the underlying system, because the ``genius`` user is not shown. Using the metasploit module is much less work than the other methods. There are plenty of these devices connected to the internet, and searchable on shodan https://www.shodan.io/search?query=%22IVSWeb%22, all of which are potentially have the same flaw.

I may try to tweak the existing metasploit module and see if I can get the reset function to work.

The Local Interface
===================

There isn't much to report on the local interface. The only significant finding is that the passwords for users are limited to 6 characters. I believe this may be an artifact of the underlying hash algorithm.   It is a rather clunky system where the only way to input text is though an on screen keyboard, so passwors are likely to be lowercase if not just numeric.

Summary of vulnerabilities
==========================

There are multiple, low skill vulnerabilities in the device.

- Insecure authentication

  - Live stream available via RTSP requires no authentication
  - device only supports http
  - credentials stored as a cookie
  - relies on an integer as a session id (always starts at 1).

    - can erase all footage from the device by making a POST request by simply guessing the session id

  - passwords have max length of 6

- No authentication
  - metasploit module can retrieve password hashes and other configuration details (smtp password, ftp password)

- Hard coded credentials

  - telnet password is easily cracked, and is the same on all devices
  - ``genius`` web user that is not listed in the local user admin interface, but has full system access

- Users likely unaware telnet is enabled



Mitigations
===========

- Don't connect it to the internet
  - If it must be connected to the internet, put it behind a VPN.
  - Segregate it from the guest network

- Change the telnet password
- manually edit ``/opt/sav/Config/passwd`` to remove the ``genius`` user

Citations
=========

.. [1] Craig Heffner via HackersOnBoard, “Black Hat 2013 - Exploiting Network Surveillance Cameras Like a Hollywood Hacker,” Nov. 19, 2013. https://www.youtube.com/watch?v=B8DjTcANBx0
.. [2] Reynolds, Jake, “Dahua DVR Authentication Bypass - CVE-2013-6117.” https://depthsecurity.com/blog/dahua-dvr-authentication-bypass-cve-2013-6117
.. [3] “Hacking RAM disks,” Boundary Devices, Sep. 07, 2012. https://boundarydevices.com/hacking-ram-disks/
.. [4] “NVD - CVE-2013-6117.” https://nvd.nist.gov/vuln/detail/CVE-2013-6117
.. [5] “ReaderDump RTS5301 VDD V18 11003 Zip,” Rosewill. https://www.rosewill.com/download/readerdump-rts5301-vdd-v18-11003-zip/
.. [6] B. Tamasi, “Reverse Engineering DVR firmware,” Medium, Apr. 23, 2020. https://medium.com/@halftome/reverse-engineering-dvr-firmware-e7fec42f2a88
.. [7] “Reverse Engineering Firmware Primer,” Security Weekly Wiki, Dec. 13, 2012. https://wiki.securityweekly.com/Reverse_Engineering_Firmware_Primer
.. [8] K. Makan, “[Reverse Engineering Primer] Unpacking cramfs firmware file systems.” http://blog.k3170makan.com/2018/06/reverse-engineering-primer-unpacking.html
