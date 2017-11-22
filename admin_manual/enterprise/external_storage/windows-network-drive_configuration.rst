===========================================================================
Installing and Configuring the External Storage: Windows Network Drives App
===========================================================================

The "External Storage: Windows Network Drives" app creates a control panel in your Admin page for seamlessly integrating Windows and Samba/CIFS shared network drives as external storages.

Any Windows file share and Samba servers on Linux and other Unix-type operating systems use the SMB/CIFS file-sharing protocol. 
The files and directories on the SMB/CIFS server will be visible on your Files page just like your other ownCloud files and folders. 

They are labeled with a little four-pane Windows-style icon, and the left pane of your Files page includes a Windows Network Drive filter. 
Figure 1 shows a new Windows Network Drive share marked with red warnings. 

These indicate that ownCloud cannot connect to the share because it requires the user to login, it is not available, or there is an error in the configuration. 

.. figure:: images/wnd-1.png
   :alt: Windows Network Drive share on your Files page.
   
   *Figure 1: Windows Network Drive share on your Files page.*

Files are synchronized bi-directionally, and you can create, upload, and delete files and folders. 
ownCloud server admins can create Windows Network Drive mounts and optionally allow users to set up their own personal Windows Network Drive mounts. 

Depending on the authentication method, passwords for each mount are encrypted and stored in the ownCloud database, using a long random secret key stored in ``config.php``, which allows ownCloud to access the shares when the users who own the mounts are not logged in. 
Or, passwords are not stored and available only for the current session, which adds security.

Installation
------------

Install `the External Storage: Windows Network Drives app`_ from the ownCloud Market App or ownCloud Marketplace. 
For it to work, there are a few dependencies to install.

- A Samba client. This is included in all Linux distributions. On Debian, Ubuntu, and other Debian derivatives it is called ``smbclient``. On SUSE, Red Hat, CentOS, and other Red Hat derivatives it is ``samba-client``. 
- ``php-smbclient`` (version 0.8.0+). It should be included in most Linux distributions. You can use `eduardok/libsmbclient-php`_, if your distribution does not provide it.
- ``which`` and ``stdbuf``. These should be included in most Linux distributions.

Example
~~~~~~~

Assuming that your ownCloud installation is on Ubuntu, then the following commands will install the required dependencies:

.. code-block:: console
   
  # Install core packages
  sudo apt-get update -y
  sudo apt-get install -y smbclient coreutils
  
  # Install php-smbclient using PECL
  pecl install smbclient
  
  # Install it from source
  git clone git://github.com/eduardok/libsmbclient-php.git
  cd libsmbclient-php ; phpize
  ./configure
  make
  sudo make install
  
  # Enable the extension in your PHP installation
  extension="smbclient.so"

Creating a New Share
--------------------

When you create a new WND share you need: the login credentials for the share, the server address, the share name, and the folder you want to connect to. 

1. Enter the ownCloud mount point for your new WND share. This must not be an existing folder.
2. Then select your authentication method; See  :doc:`enterprise_only_auth` for complete information on the five available authentication methods.
   
.. figure:: images/wnd-2.png
   :alt: WND mountpoint and auth.
   
   *Figure 2: WND mountpoint and authorization credentials.*
   
3. Enter the address of the server that contains the WND share.
4. The Windows share name.
5. The root folder of the share. This is the folder name, or the 
   ``$user`` variable for user's home directories. Note that the LDAP 
   ``Internal Username Attribute`` must be set to the ``samaccountname`` for either the share or the root to work, and the user's home directory needs to match the ``samaccountname``. (See 
   :doc:`../../configuration/user/user_auth_ldap`.)
6. Login credentials.
7. Select users or groups with access to the share. The default is all users.
8. Click the gear icon for additional mount options. Note that previews are enabled by default, while sharing is not (see figure 2). Sharing is not available for all authorization methods; see :doc:`enterprise_only_auth`. For large storages with many files, you may want to disable previews, because this can significantly increase performance.

.. figure:: images/wnd-3.png
   :alt: WND server and credentials.

   *Figure 3: WND server, credentials, and additional mount options.*  

Your changes are saved automatically.

.. note:: When you create a new mountpoint using Login credentials, you must log out of ownCloud and then log back in so you can access the share. You only have to do this the first time.

Personal WND Mounts
-------------------

Users create their own WND mounts on their Personal pages. 
These are created the same way as Admin-created shares. 
Users have four options for login credentials: 

* Username and password
* Log-in credentials, save in session
* Log-in credentials, save in database
* Global credentials

libsmclient Issues
------------------

If your Linux distribution ships with ``libsmbclient 3.x``, which is included in the Samba client, you may need to set up the HOME variable in Apache to prevent a segmentation fault. 
If you have ``libsmbclient 4.1.6`` and higher it doesn't seem to be an issue, so you won't have to change your HOME variable.
To set up the HOME variable on Ubuntu, modify the ``/etc/apache2/envvars`` file::

  unset HOME
  export HOME=/var/www

In Red Hat/CentOS, modify the ``/etc/sysconfig/httpd`` file and add the following line to set the HOME variable in Apache::

  export HOME=/usr/share/httpd
 
By default, CentOS has activated SELinux, and the ``httpd`` process can not make outgoing network connections. 
This will cause problems with the ``curl``, ``ldap`` and ``samba`` libraries. 
You'll need to get around this to make this work. First, check the status::

  getsebool -a | grep httpd
  httpd_can_network_connect --> off

Then enable support for network connections::

  setsebool -P httpd_can_network_connect 1

In openSUSE, modify the ``/usr/sbin/start_apache2`` file::
 
  export HOME=/var/lib/apache2

Restart Apache, open your ownCloud Admin page and start creating SMB/CIFS mounts.

==============================
Windows Network Drive Listener
==============================

The SMB protocol supports registering for notifications of file changes on remote Windows SMB storage servers. 
Notifications are more efficient than polling for changes, as polling requires scanning the whole SMB storage.
ownCloud supports SMB notifications with an ``occ`` command, ``occ wnd:listen``.

.. Note:: The notifier only works with remote storage on Windows servers. It
   does not work reliably with Linux servers due to technical limitations.

Your ``smbclient`` version needs to be 4.x, as older versions do not support notifications.
The ownCloud server needs to know about changes to files on integrated storage so that the changed files will be synced to the ownCloud server, and to desktop sync clients.

Files changed through the ownCloud Web Interface, or sync clients are automatically updated in the ownCloud file cache, but this is not possible when files are changed directly on remote SMB storage mounts.

To create a new SMB notification, start a listener on your ownCloud server with ``occ wnd:listen``.
The listener marks changed files, and a background job updates the file metadata.

Windows network drive connections and setup of ``occ wnd:listen`` often does not always work the first time.
If you encounter issues using it, then try the following troubleshooting steps:

1. Check the connection with smbclient_ on the command line of the ownCloud server
2. If you are connecting to `Distributed File Shares`_ (DFS), be aware that the
   shares are case-sensitive

Take the example of attempting to connect to the share named `MyData` using ``occ wnd:listen``.
Running the following command would work::

   su www-data -s /bin/bash -c 'php /var/www/owncloud/occ ``wnd:listen`` dfsdata MyData svc_owncloud password'

However, running this command would not::

   su www-data -s /bin/bash -c 'php /var/www/owncloud/occ ``wnd:listen`` dfsdata mydata svc_owncloud password'

Setting Up the WND Listener
---------------------------

The WND listener for ownCloud 10 includes two different commands that need to be executed:

- `wnd:listen`_
- `wnd:process-queue`_

wnd:listen
----------

This listens and stores notifications in the database, coming from one specific host and share and is intended to be run as a service.
The command requires the host and share which the listener will listen to and the Windows/Samba account that will listen.
The command does not produce any output by default, unless errors happen.

You can increase the verbosity (``wnd:listen -vvv``) to know what the listener is doing.
Doing so will display a timestamp and the notifications received.
Although unknown, the exact permissions required for the Windows/samba account, read-only should be enough.

wnd:process-queue
-----------------

Processes the stored notifications for a given host and share.
This process is intended to be run periodically as a Cron job, or via a similar mechanism.
The command will process the notifications stored by the ``wnd:listen`` process, showing only errors by default.
If you need more information, increase the verbosity by calling ``wnd:process-queue -vvv``.

Basic Setup for One ownCloud Server
-----------------------------------

First, go to the admin settings and set up the required WND mounts.
Be aware though, that there are some limitations.
These are:

- We need access to the Windows account password for the mounts to update the file cache properly. This means that "*login credentials, saved in session*" won't work with the listener. "*login credentials, saved in DB*" should work and could be the best replacement.
- The ``$user`` placeholder in the share, such as ``//host/$user/path/to/root``, for providing a share which is accessible per/user won't work with the listener. This is because the listener won't scale, as you'll need to setup one listener per/share. As a result, you'll end up with too many listeners. An alternative is to provide a common share for the users and use the ``$user`` placeholder in the root, such as ``//host/share/$user/folder``.

Second, start the ``wnd:listen`` process if it's not already started, ideally running it as a service.
If it isn't running, no notification are stored.
The listener stores the notifications.
Any change in the mount point configuration, such as adding or removing new mounts, and logins by new users, won't affect the behavior, so there is no need to restart the listener in those cases.

In case you have several mount point configurations, note that each listener attaches to one host and share.
If there are several mount configurations targeting different shares, you'll need to spawn one listener for each.
For example, if you have one configuration with ``10.0.0.2/share1`` and another with ``10.0.0.2/share2``, you'll need to spawn 2 listeners, one for the first configuration and another for the second.

Third, run the ``wnd:process-queue`` periodically, usually via :ref:`a Cron job <cron_job_label>`.
The command processes all the stored notifications for a specific host and share.
If you have several, you could set up several Cron jobs, one for each host and share with different intervals, depending on the load or update urgency.
As a simple example, you could run the command every 2 minutes for one server and every 5 minutes for another.

As said, the command processes all the stored notifications, squeeze them and scan the resulting folders.
The process might crash if there are too many notifications, or if it has too many storages to update.
The ``--chunk-size`` option will help by making the command process all the notifications in buckets of that size.

On the one hand the memory usage is reduced, on the other hand there is more network activity.
We recommend using the option with a value high enough to process a large number of notifications, but not so large to crash the process.
Between 200 and 500 should be fine, and we'll likely process all the notifications in one go.

Optimizing wnd:process-queue
----------------------------

``wnd:process-queue`` creates all the storages that need to be updated from scratch.
To do so, we need to fetch all the users from all the backends (currently only the ones that have logged in at least once because the others won't have the storages that we'll need updates)

To optimize this, ``wnd:process-queue`` make use of two switches: "--serializer-type" and "--serializer-params".
These serialize storages for later use, so that future executions don't need to fetch the users, saving precious time — especially for large organizations.

======================= =======================================================================
Switch                  Allowed Values 
======================= =======================================================================
``--serializer-type``   ``file``.  Other valid values may be added in the future, as more
                        implementations are requested.
``--serializer-params`` Depends on ``--serializer-type``, because those will be the parameters
                        that the chosen serializer will use. For the ``file`` serializer, you 
                        need to provide a file location in the host FS where the storages will 
                        be serialized. You can use ``--serializer-params file=/tmp/file`` as an 
                        example.
======================= =======================================================================

While the specific behavior will depend on the serializer implementation, the overall behavior can be simplified as follows: 

If the serializer’s data source (such as *a file*, *a database table*, or some *Redis keys*) has storage data, it uses that data to create the storages; otherwise, it creates the storages from scratch. 

After the storages are created, notifications are processed for the storages. 
If the storages have been created from scratch, those storages are written in the data source so that they can be read on the next run.

.. note::
   It's imperative to periodically clean up the data source to fetch fresh data, such as for new storages and updated passwords. There isn't a generic command to do this from ownCloud, because it depends on the specific serializer type. Though this option could be provided at some point if requested.

The File Serializer
-------------------

The file serializer is a serializer implementation that can be used with the ``wnd:process-queue`` command.
It requires an additional parameter where you can specify the location of the file containing the serialized storages. 

There are several things you should know about this serializer:

- The generated file contains the encrypted passwords for accessing the backend. This is necessary in order to avoid re-fetching the user information, when next accessing the storages.
- The generated file is intended to be readable and writeable **only** for the web server user. Other users shouldn't have access to this file. Do not manually edit the file. You can remove the file if it contains obsolete information.

Usage Recommendations
~~~~~~~~~~~~~~~~~~~~~

Number of Serializers
^^^^^^^^^^^^^^^^^^^^^

Only one file serializer should be used per server + share, as the serialized file has to be per server + share.
Consider the following usage scenarios:

- You have several Windows servers. Then you'll need to serialize to different files for each of server.
- You have three shares: ``10.0.2.2/share1``, ``10.0.2.2/share2``, and ``10.0.10.20/share2``. Then you should use three different calls to ``wnd:process-queue``, changing the target file for the serializer for each one.

Since the serialized file has to be per server + share, the serialized file has some checks to prevent misuse. 
Specifically, if we detect you're trying to read the storages for another server + share from the file, the contents of the file won't be read and will fallback to creating the storage from scratch. 
We'll then update the contents of that file with the new storage.

.. "unneeded competition" needs clarification or rewording
Note that this will create an unneeded competition, so again, use a different file per server and share.

File Clean Up
^^^^^^^^^^^^^

The file will need to cleaned up from time to time. 
The easiest way to do this is to remove the file when it is no longer needed.
The file will be regenerated with fresh data the next execution if the serializer option is set.

Interaction Between Listener, Serializer, and Windows Password Lockout
----------------------------------------------------------------------

Windows supports `password lockout policies`_.
If one is enabled on the server where an ownCloud share is located, and a user fails to enter their password correctly several times, they may be locked out and unable to access the share.

This is `a known issue`_ that prevents these two inter-operating correctly.
Currently, the only viable solution is to ignore that feature and use the ``wnd:listen`` and ``wnd:process-queue``, without the serializer options.

There is also an additional issue to take into account though, which is that parallel runs of ``wnd:process-queue`` might lead to a user lockout.
The reason for this is that several ``wnd:process-queue`` might use the same wrong password because it hasn't been updated by the time they fetch it.

As a result, it's recommended to force the execution serialization of that command to prevent this issue.
You might want to use `Anacron`_, which seems to have an option for this scenario, or wrap the command with `flock`_.

Multiple Server Setup
---------------------

Setups with several servers might have some difficulties in some scenarios:

- The ``wnd:listen`` component *might* be duplicated among several servers. This shouldn't cause a problem, depending on the limitations of the underlying database engine. The supported database engines should be able to handle concurrent access and de-duplication.
- The ``wnd:process-queue`` *should* also be able to run from any server, however limitations for concurrent executions still apply. As a result, you might need to serialized command execution of the ``wnd:process-queue`` among the servers (to avoid for the password lockout), which might not be possible or difficult to achieve. You might want to execute the command from just one specific server in this case.
- ``wnd:process-queue`` + serializer. First, check the above section to know the interactions with the password lockout. Right now, the only option you have to set it up is to store the target file in a common location for all the server. We might need to provide a specific serializer for this scenario (based on Redis or DB)

Basic Command Execution Examples
--------------------------------

::

  sudo -u www-data ./occ ``wnd:listen`` host share username password
  
  sudo -u www-data ./occ ``wnd:process-queue`` host share
  
  sudo -u www-data ./occ ``wnd:process-queue`` host share -c 500
  
  sudo -u www-data ./occ ``wnd:process-queue`` host share -c 500 \
      --serializer-type file \
      --serializer-params file=/opt/oc/store
  
  sudo -u www-data ./occ ``wnd:process-queue`` host2 share2 -c 500 \
      --serializer-type File \
      --serializer-params file=/opt/oc/store2

To set it up, make sure the listener is running as a system service: 

::

  sudo -u www-data ./occ ``wnd:listen`` host share username password

Setup a Cron job or similar with something like the following two commands:

::

  sudo -u www-data ./occ wnd:process-queue host share -c 500 \
      --serializer-type file \
      --serializer-params file=/opt/oc/store1

  rm -f /opt/oc/store1 # With a different schedule

The first run will create the ``/opt/oc/store1`` with the serialized storages, the rest of the executions will use that file.
The second Cron job, the one removing the file, will force the ``wnd:process-queue`` to refresh the data.

It's intended to be run in a different schedule, so there are several executions of the ``wnd:process-queue`` fetching the data from the file.
Note that the file can be removed manually at any time if it's needed (for example, the admin has reset some passwords, or has been notified about password changing).

.. Links

.. _systemd: https://en.wikipedia.org/wiki/Systemd
.. _smbclient: https://www.samba.org/samba/docs/man/manpages-3/smbclient.1.html
.. _Distributed File Shares: https://en.wikipedia.org/wiki/Distributed_File_System_(Microsoft)
.. _the External Storage\: Windows Network Drives app: https://marketplace.owncloud.com/apps/Windows_network_drive
.. _eduardok/libsmbclient-php: https://github.com/eduardok/libsmbclient-php
.. _Anacron: http://www.thegeekstuff.com/2011/05/anacron-examples
.. _flock: http://linuxaria.com/howto/linux-shell-introduction-to-flock
.. _a known issue: https://github.com/owncloud/Windows_network_drive/issues/94
.. _password lockout policy: https://technet.microsoft.com/en-us/library/dd277400.aspx
