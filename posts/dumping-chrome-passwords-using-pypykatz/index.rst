.. title: Dumping chrome passwords using pypykatz
.. slug: dumping-chrome-passwords-using-pypykatz
.. date: 2022-05-30 09:27:07 UTC-04:00
.. tags:
.. category: hacking, ctf writeup
.. link:
.. description:
.. type: text

I recently competed in the Cyber Apocalypse 2022 CTF on Hackthebox.
This was a really fun experience (and my first ever live CTF).
I only solved a few challenges (which I'll be writing in subsequent posts), but there was one vexating challenge which I wasn't able to complete.
I kinda had the right idea, but I was missing some key pieces of information.
The point of this post is to explain what I was missing and how it works (and to document how I would have done it).

This particular challenge was called "Siezed"

The challenge description read

  Miyuki is now after a newly formed ransomware division which works for Longhir. This division's goal is to target any critical infrastructure and cause financial losses to their opponents. They never restore the encrypted files, even if the victim pays the ransom. This case is the number one priority for the team at the moment. Miyuki has seized the hard-drive of one of the members and it is believed that inside of which there may be credentials for the Ransomware's Dashboard. Given the AppData folder, can you retrieve the wanted credentials?

A zip file was provided which contained the full AppData from a windows user.
Based on the description, I felt like all I needed to do was search for and extract the browser's saved passwords.

.. TEASER_END

Google Chrome stores saved passwords in 2 locations:

``AppData/Local/Google/Chrome/User Data/Default/Login Data` or `AppData/Local/Google/Chrome/User Data/<profile name>/Login Data``


I didn't see evidence of any other profiles, so I started looking in Default.
Open the 'Login Data' file with sqlite

::

  sqlite> .mode json
  sqlite> select * from logins;
  [{
    "origin_url": "https://windowsliveupdater.com/",
    "action_url": "https://windowsliveupdater.com/",
    "username_element": "mat-input-0",
    "username_value": "ransomoperator@draeglocker.com",
    "password_element": "mat-input-1",
    "password_value": "v10TK�Wzk\u0001]%\n+��䋼iܙdOm�JF�*|:�\f0\u0017\n/M�u0000R",
    "submit_element": "",
    "signon_realm": "https://windowsliveupdater.com/",
    "date_created": 13292435735835041,
    "blacklisted_by_user": 0,
    "scheme": 0,
    "password_type": 0,
    "times_used": 0,
    "form_data": "<snip>",
    "display_name": "",
    "icon_url": "",
    "federation_url": "",
    "skip_zero_click": 0,
    "generation_upload_status": 0,
    "possible_username_pairs": "\u0000\u0000\u0000\u0000",
    "id": 1,
    "date_last_used": 13292435729963090,
    "moving_blocked_for": "\u0000\u0000\u0000\u0000",
    "date_password_modified": 13292435735835381
  }]

So there's a password here, but it's encrypted.
Bummer.

This is where I gave up, because I didn't know how to break this encryption, and didn't even really know what I was looking at.
After looking at writeups from other users after the CTF was over, I learned that this is ecrypted using `DPAPI <https://support.microsoft.com/en-us/topic/bf374083-626f-3446-2a9d-3f6077723a60>`_.

john can be used to crack the password, it just needs to know the SID and the path to the master key certificate.
The SID can be found in ``AppData\Roaming\Microsoft\Protect\``, and it is ``S-1-5-21-3702016591-3723034727-1691771208-1002``.
The master key

Browsing the directory, there are a few files here, but I don't immediately know which one is the right one.

::

  AppData/Roaming/Microsoft/Protect/
  |-- CREDHIST
  `-- S-1-5-21-3702016591-3723034727-1691771208-1002
    |-- 865be7a6-863c-4d73-ac9f-233f8734089d (486 bytes)
    `-- Preferred (24 bytes)

Based on the file sizes, I guessed that the ``865be7a6-863c-4d73-ac9f-233f8734089d`` file would be the one.
Use DPAPImk2john to extract the master key certificate to a format that can be cracked *must use python 2 venv*.
``DPAPImk2john -S S-1-5-21-3702016591-3723034727-1691771208-1002 -mk AppData/Roaming/Microsoft/Protect/S-1-5-21-3702016591-3723034727-1691771208-1002/865be7a6-863c-4d73-ac9f-233f8734089d -c local > /hash.txt``

::

  $DPAPImk$2*1*S-1-5-21-3702016591-3723034727-1691771208-1002*aes256*sha512*8000*a17612a0ebdfc203316e0c18c04729f1*288*7dd629ab5efc8442596e5fbe5b9fc695bf8a51384dfacabd7a1a214245f894383540eb3e00c009bd76f836ae991cef540d74c0a6a31527b7e1df4b0d55a6760e41271f3dcaad163a6fb648f898281424728485335676c0374735cab055088e66bc55a72fc2087d64038d1d716f5efd4bdd4ce19971d082db004a36de70c351a2bd9b6ba9cf8f89a7481150b26f5808bc

Crack the file ``john --wordlist=/usr/share/wordlists/rockyou.txt ./hash.txt``.
The password is ``ransom``

Since this encryption relies on windows encryption, mimikatz would be the best tool, but I don't have access to a windows computer to be able to run it.
Instead, I'll use something called `pypykatz <https://github.com/skelsec/pypykatz>`_ which is a python implementation of mimikatz.
This allows offline attacks from my linux system.

The first step is to generate the prekey files using the SID and password.

``pypykatz dpapi prekey password 'S-1-5-21-3702016591-3723034727-1691771208-1002' 'ransom' -o key``

Next, the master key itself needs to be extracted from the certificate.

``pypykatz dpapi masterkey AppData/Roaming/Microsoft/Protect/S-1-5-21-3702016591-3723034727-1691771208-1002/865be7a6-863c-4d73-ac9f-233f8734089d key -o masterkey``

Finally, the chrome submodule can decrypt the credential files using

``pypykatz dpapi chrome ./masterkey AppData/Local/Google/Chrome/User\ Data/Local\ State --logindata AppData/Local/Google/Chrome/User\ Data/Default/Login\ Data``

This gives the flag for the challenge.

``file: AppData/Local/Google/Chrome/User Data/Default/Login Data user: <username> pass: b'HTB{fake_flag}' url: <redacted url>``

I admit, I still don't have a great understanding of how this works, but I'll explain what I do know.
There is a json key

::

  "os_crypt": {
      "encrypted_key": "RFBBUEkBAAAA0Iyd3wEV0RGMegDAT8KX6wEAAACm51uGPIZzTayfIz+HNAidAAAAAAIAAAAAABBmAAAAAQAAIAAAACc7RsTHfaauxrhBBjIqqmhrpu4YgBuonvNnS6mwHh46AAAAAA6AAAAAAgAAIAAAAL/cUy0IhgQQbDrc+KvOqsr+VCQsd9QUwZOC0v962Hf0MAAAAEHBCEaKa1Z9JzasA7wpTHI5PjeCJgrNbSTeklRxKbLst8qd8SnSo9hCOn5xwIOhwkAAAAA6QhhJeJDGW4UU26/TX3q4czhgLkuzjqXFgeH+CHdrTLjkK90vaEpXJerbw41eqFYSlsouQspBo/5R0HYeX295"
    }

in the local state.

That is a base64 encoded string.
The first 5 characters are DPAPI, and the rest are non-printable.
From reading the code in pypykatz, it seems that this key a blob encrypted with the DPAPI key.
