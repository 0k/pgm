Postgres manager
================

``pgm`` is a simple tool to manage postgres databases: copy, delete,
listing, saving local and distant database in the most simple way
through the command line.

It leverage ``ssh`` and ``psql`` to chain commands that you would
probably use yourself to achieve the similar task. Cutting open
connection, using quick copy when available, making backups and/or
deleting target database to allow easy overwriting, changing ownership
or verifying that your postgres version are compatibles... ``pgm``
tries to do in 1 simple command line what you would in several.

This is code is alpha software and contains known bugs. Use at your
own risk.


Installation
============


Getting the binary
------------------

``pgm`` requires very few dependencies. It uses ``pv`` and ``buffer``.

You can install it and his dependency by using::

     sudo apt-get install -y buffer pv &&
     sudo wget https://github.com/0k/pgm/raw/master/bin/pgm -O /usr/local/bin/pgm &&
     sudo chmod +x /usr/local/bin/pgm


Setup hosts
-----------


Password-less access
''''''''''''''''''''

You must have ``postgres`` and ``ssh`` server installed and running of
course on each host you need to join.

To avoid typing passwords, you should use and setup each of your
intended hosts with password less authentification (Key pair
authentification for instance).

And at least, you should make sure your system user on the destination
host can ``sudo -u postgres`` without password. You could add this
line to ``/etc/sudoers`` for instance::

    bob    ALL=(postgres) NOPASSWD: /usr/bin/psql, /usr/bin/pg_dump, /usr/bin/createdb

A good test would then to try (directly on the target host)::

    sudo -u postgres psql -l

Then try to issue the same command from the controling host::

    ssh myuser@target_host "sudo -u postgres psql -l"


Database Backup folder permissions
''''''''''''''''''''''''''''''''''

The ``saving`` mecanism of ``pgm`` (for example ``pgm save`` or the automatic
backup of ``-b`` option of ``pgm copy`` or ``pgm rm``) requires you to have
created a ``/var/backups/pg`` directory writeable by your user. To set it
up your could::

    mkdir /var/backups/pg -p &&
    chown "$UID" /var/backups/pg &&
    chmod 700 /var/backups/pg


Security concerns
'''''''''''''''''

You should understand all the implication of these before doing
it. You are giving this user and all the hosts that can connect to
this host a password less (but protected with key pairs) access. This
means that if the host that is able to join the others is compromised,
you'll offer the keys to all your networked postgres instances.

USE AT YOUR OWN RISK.


Limitation
----------

In the current state, ``pgm`` permission features (``pgm chown`` or
``pgm cp srcdb foo@db``) are limited to full base ownership by one
member. So if you have any more complex scenario, please contribute.


Quick tour
==========


Getting help
------------

``pgm`` supports it's own integrated help::

    $ pgm --help
    usage:
      pgm chown [-v] USER DBNAME
      pgm cp [-q] [-f | -b] SRC_DBNAME DST_DBNAME
      pgm kill [-v] DBNAME
      pgm ls [-s] [-c] [DBPATTERN]
      pgm rm [-f] [-b] DBNAME
      pgm save [-v] DBNAME [LABEL]


Each command can be queried deeper::

    $ pgm cp --help
    Copy database SRC to DST database

    usage:
      pgm cp [-q] [-f | -b] SRC_DBNAME DST_DBNAME

    options:
      -q       Use quicker templating system to make the copy, note that
               all active session will be dropped.
      -f       Force copy even if DST_DBNAME already exists (it'll overwrite it).
      -b       As 'force' but will backup the database before overwriting it.
    $

The dbpattern
-------------

Most of the command will use a database pattern syntax to target a
database. This is the general syntax:

    [[USER@]HOST:][PGUSER@][DBPATTERN][:PGPORT]

There are some examples::

    mydb            ## a local database named ``mydb``
    host1:mydb      ## a database ``mydb`` on the host ``host1``
    host2:bob@mydb  ## target a database name/user, usefull for specifying destination for ``cp`` or filtering ``ls``

``USER`` and ``HOST`` will be used directly by ssh. You can use anything that
ssh will understand (IP, resolvable domain name, ssh aliases...).

``PGPORT`` is still not implemented. If you are under debian, you can
probably use ``pg_local_opts`` environment variable to set ``--cluster
9.1/main`` option to be added to each commands, but this remains to be
clarified, both in implementation and documentation.


Listing databases
-----------------

By default, the ``ls`` command will list local database, along with
their owner and their sizes::

   $ pgm ls
   postgres                 postgres           6540 kB
   dbA                      bob                  32 MB

You can list distant databases::

   $ pgm ls host1:
   dev61                    openerp              37 MB
   dummy                    openerp              32 MB
   postgres                 postgres           5320 kB
   test                     bob                 203 MB

And even filter by user::

   $ pgm ls host1:bob@
   test                     bob                 203 MB


Or use wildcards in the database names to target a subselection, the
``-c`` optional argument will ask for the number of open connection
currently on the database::

   $ pgm ls host1:d* -c
   dev61                    openerp              37 MB    8
   dummy                    openerp              32 MB    0

You can also have a very short output (practical for scripting for
instance), which list only the matching database names::

   $ pgm ls host1:d* -s
   dev61
   dummy


Copying databases
-----------------

Copy ``dbA`` on to ``dbB``... will use templating automatically to go
quicker, if both database are on the same server, and if there's no
open connection to ``dbA``::

    $ pgm cp dbA dbB
    Quick copy of dbA to dbB.

or, if ``dbA`` has open connections::

    $ pgm cp dbA dbB
    Copy of dbA to dbB.
     received: 4.43MB 0:00:03 [ 1.4MB/s] [      <=>             ]
     unpacked: 4.43MB 0:00:03 [1.44MB/s] [      <=>             ]
         fill: 4.26MB 0:01:03 [9.88kB/s] [               <=>    ]
    Finished copy successfully.
    $

Either source or destination or both support distant databases, so you can
use ``pgm cp`` for exporting purpose::

    $ pgm cp dbA host:dbA

or importing purpose, notice the ``-f`` to force overwriting destination::

    $ pgm cp -f host2:dbA dbA

You can also force ownership of destination database by using
``<owner>@<dbname>`` syntax::

    $ pgm cp dbA host1:alice@dbC
    ...
    Chowning dbC on host1 to user alice.

Note that in case of different version of postgres, a warning will be
issued. For instance::

    Warning: Postgres version mismatch between hosts (src: 8.4.22, dst: 9.4rc1). This might generate errors !


Other commands
--------------

Documentation is still to be done for those::

      pgm chown [-v] USER DBNAME
      pgm kill [-v] DBNAME
      pgm rm [-f] [-b] DBNAME
      pgm save [-v] DBNAME [LABEL]
