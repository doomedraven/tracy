ZipJail
=======

ZipJail is a usermode sandbox for unpacking archives using the ``unzip``,
``tar``, ``rar``, ``7z``, and ``unace`` utilities. Through the use of the
``tracy`` library it limits the attack surfaces to an absolute minimum in case
a malicious archive tries to exploit known or unknown vulnerabilities in said
archive tools.

**Motivation behind this small wrapper utility** may be found in the
`Security`_ chapter, below in this document.


Usage
=====

The ``zipjail`` command itself requires two parameters followed by the command
that should be executed and jailed (i.e., sandboxed). The two parameters
belonging to ``zipjail`` define the filepath to the archive and the output
directory to which file writes should be restricted.

.. code-block:: bash

    $ zipjail
    zipjail 0.5.5 - safe unpacking of potentially unsafe archives.
    Copyright (C) 2016-2018, Jurriaan Bremer <jbr@hatching.io>.
    Copyright (C) 2018-2021, Hatching B.V.
    Based on Tracy by Merlijn Wajer and Bas Weelinck.
        (https://github.com/MerlijnWajer/tracy)

    Usage: zipjail <input> <output> [options...] -- <command...>
      input:   input archive file
      output:  directory to extract files to
      verbose: some verbosity

    Options:
      -v           more verbosity
      -a=X         terminates process after X seconds
      --alarm=X    same as -a=X
      -c=N         more clones (default: 0)
      --clone=N    same as -c=N
      -w=X         maximum total file size (default: 1GB)
      --write=X    same as -w=X

    Please refer to the README for the exact usage.

Following we will demonstrate ``zipjail``'s usage based on an input file
called ``archive.zip`` and the output directory ``/tmp/unpacked/``.

Unzip
^^^^^

In order to run ``zipjail`` with ``unzip`` the command-line should be
constructed as follows.

.. code-block:: bash

    $ zipjail file.zip /tmp/unpacked -- unzip -o -d /tmp/unpacked file.zip

Tar
^^^

In order to run ``zipjail`` with ``tar`` the command-line should be
constructed as follows.

.. code-block:: bash

    $ zipjail file.tar     /tmp/unpacked -- tar xfv  file.tar     -C /tmp/unpacked
    $ zipjail file.tar.gz  /tmp/unpacked -- tar xfvz file.tar.gz  -C /tmp/unpacked
    $ zipjail file.tar.bz2 /tmp/unpacked -- tar xfvj file.tar.bz2 -C /tmp/unpacked

Rar
^^^

Just like for the ``7z`` command we require setting the multithreaded count
for the ``rar`` command. It should be noted that ``unrar`` version
``5.00 beta 8`` does not support the multithreaded option and thus ``zipjail``
is not capable of running with that version. So far we have only tested that
``zipjail`` works with ``rar`` version ``4.20``. Its usage is as follows.

.. code-block:: bash

    $ zipjail file.rar /tmp/unpacked -- rar x -mt1 file.rar /tmp/unpacked

7z
^^

Running ``zipjail`` with ``7z`` may be done as follows. Note that we pass
along the ``-mmt=off`` option which disables multithreaded decompression for
``bzip2`` targets. By keeping ``zipjail``'s sandboxing single-threaded we keep
its logic easy and secure (using multithreading race conditions would be
fairly trivial). In fact, as per our unittests, trying to instantiate
multithreading (e.g., through ``pthread``, which internally invokes the
``clone(2)`` system call) is blocked completely. (Also note that the directory
provided to ``7z``'s ``-o`` parameter should be added right away without
additional whitespaces).

.. code-block:: bash

    $ zipjail file.7z /tmp/unpacked -- 7z x -mmt=off -o/tmp/unpacked file.7z

In some cases, however, such as when decrypting a password-protected ``.7z``
file, it appears that up to two additional threads are used, so in order to
do so one will have to bump the allowed ``clone(2)`` calls to two.
Note: in terms of security this option weakens our protection a bit due as it
allows multiple threads that may be used for race conditions.

.. code-block:: bash

    $ zipjail pw.7z /tmp/unpacked --clone=1 -- \
        7z x -mmt=off -Pinfected -o/tmp/unpacked pw.7z

unace
^^^^^

Another utility, another command-line. This time, for ``unace``, which handles
``.ace`` files, the command-line is fairly straightforward except for the
input file path and the directory path that are passed along. The file path
must be an absolute path and the directory path needs to be slash-terminated,
i.e., the path should finish off with a forward slash.

.. code-block:: bash

    $ zipjail /tmp/file.ace /tmp/unpacked -- \
        unace x /tmp/file.ace /tmp/unpacked/

It should be noted that only ``unace`` version ``2.5`` is supported as the
older versions don't support either the command-line arguments or the ``.ace``
samples that are actually being used in-the-wild. Installing this particular
version may be done through ``sudo apt install unace-nonfree``.

PowerISO
^^^^^^^^

In order to run ``zipjail`` with ``poweriso`` the command-line should be
constructed as follows.

.. code-block:: bash

    $ zipjail file.zip /tmp/unpacked -- \
        poweriso extract file.daa -od /tmp/unpacked

Security
========

Given its security implications (and use in, e.g., `Cuckoo Sandbox`_) it is of
utmost importance that ``zipjail`` is completely secure. Therefore, may you
locate a potential security issue, please reach out to us at
``jbr@hatching.io``.

There has been some public research into vulnerabilities and exploits aiming
at archive implementations in particular. Following is a non-complete list of
such papers (feel free to reach out to add your research):

* `PlayingWithFire by Felix Wilhelm, directory traversal through symlinks bug
  leading to RCE in FireEye MPS appliance
  <https://www.ernw.de/download/ERNW_44CON_PlayingWithFire_signed.pdf>`_.
* `Various 7-Zip vulnerabilities, by Cisco Talos
  <http://blog.talosintel.com/2016/05/multiple-7-zip-vulnerabilities.html>`_.
* `Various libarchive vulnerabilities, by Cisco Talos
  <http://blog.talosintel.com/2016/06/the-poisoned-archives.html>`_.

.. _`Cuckoo Sandbox`: https://github.com/cuckoosandbox/cuckoo

Compilation
===========
To compile zipjail:
* Compile parent project with ``cd .. && make``
* Compile zipjail with ``cd zipjail && make``


How to investigate Killing child X
==================================

Run zipjail as standalone executable:

```bash
zipjail <target_file> <extraction_foldr> -c=30 -v -- <path_to_extractor> <extractor args>

Example:
    zipjail sample.zip /tmp -c30 -v -- /usr/bin/7z x -o/tmp -infected
```
