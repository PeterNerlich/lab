.. highlight:: console

.. author:: Peter Nerlich <peter.nerlich+dev@googlemail.com>

.. tag:: lang-python
.. tag:: web
.. tag:: mail

########################
Mail Password Reset Tool
########################

.. tag_list::

A little python application that allows resetting the password for a mailbox without having access to the uberspace account (or nagging an administrator about it).

----

License
=======

* `GNU LGPLv3 https://choosealicense.com/licenses/lgpl-3.0`_



Working Principle
=================

This is intended to be used in situations where you, the uberspace account holder, provide mailboxes to one or more clients and want them to change their password on their own, without access to the shell.
I tried to make it as secure as I could imagine:

1. Users go to the website and enter their mailbox name.
2. If that mailbox exists and is not blacklisted, the system deposits a new mail in the mailbox with a token link (there is no mail actually being sent, there is just a file created in the maildir). That token is saved to a database, including for which mailbox it got requested.
3. When the link is used and the token is not older than 1 hour (by default), it is marked as used and a hidden, temporary password is added to the token entry in the database.
4. If the user actually clicks on "Generate new password" on the site they opened with the link in the email and the token is not older than 1 hour (by default) + 10 seconds, and the token has been used only within the last 5 minutes (by default), a random six-word password is generated (based on `xkcdpass https://github.com/redacted/XKCD-password-generator`_) and set as the new password for the originally selected mailbox.
5. The password is shown exactly once to the user, along with the request to keep it safe and to contact an administrator if anything happens.

There is also a page on how to access the mailbox via both webmail.uberspace.de and a local mail client, much similar to https://manual.uberspace.de/mail-access/, but with the host put in (since other users wouldn't necessarily know what the uberspace host is you put your stuff on).

There are English and German translations. Currently, there is no way to switch between them except through autodetection based on browser settings. `Contributions like other translations are always welcome! https://github.com/PeterNerlich/uberspace_mail_pw_reset`_


Installation
============

Clone the repository
--------------------

::

  [isabell@stardust ~]$ git clone https://github.com/PeterNerlich/uberspace_mail_pw_reset.git mail_pw_reset

This will download the repository to a directory named ``mail_pw_reset``. Adjust as you see fit, but this will be used throughout this guide.

Create virtual environment and install dependencies
---------------------------------------------------

::

  [isabell@stardust ~]$ cd mail_pw_reset
  [isabell@stardust mail_pw_reset]$ python3.9 -m venv .
  [isabell@stardust mail_pw_reset]$ source bin/activate
  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ CPATH=/usr/include/python3.9 pip install -r requirements.txt
  Collecting Flask
    Using cached Flask-2.0.1-py3-none-any.whl (94 kB)
  Collecting flask_sqlalchemy
  
  ...
  
  Successfully installed flask-babel-2.0.0 flask-sqlalchemy-2.5.1 python-dotenv-0.18.0 uWSGI-2.0.19.1
  WARNING: You are using pip version 21.1.1; however, version 21.1.3 is available.
  You should consider upgrading via the '/home/masonbee/mail_pw_reset/test/bin/python3.9 -m pip install --upgrade pip' command.
  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ 

You can ignore that last warning about ``pip`` not being up to date.

Environment variables
---------------------

Create the ``.env`` file from the ``env.example``: ``cp env.example .env`` and adjust the values to your needs. For example:

::

  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ cat .env
  cat env.example 
  DATABASE_URI=sqlite:///uberspace_mail_pw_reset.sqlite3
  
  SECRET_KEY=[INSERT_SECRET_KEY_HERE]
  
  BLACKLISTED_MAILBOXES=info,post
  
  PREFERRED_URL_SCHEME=https
  SERVER_NAME=isabell.example
  APP_ROOT=/mail_reset
  MAIL_SENDER=noreply@isabell.example
  MAIL_RECEIVER_DOMAIN=isabell.example
  
  TOKEN_SECONDS_VALID=3600  # 1 hour
  INITIAL_TOKEN_SECONDS_VALID=172800  # 48 hours
  TOKEN_MAX_DELAY_TO_RESET=300  # 5 min
  
  LANGUAGES=en,de
  DEFAULT_LOCALE=en
  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ 

Obviously, replace ``[INSERT_SECRET_KEY_HERE]`` with a good string. You don't have to remember it, so you can be lazy and, for example, `generate some UUID https://duckduckgo.com/?q=uuid`_ to use for that.

We use a SQLite database for storing tokens here. That is not a real database, just a file pretending to be one, but for this purpose it should be fine: We don't expect to receive a lot of traffic to the application and our clients will probably not want to reset their mailbox password every second...
When someone requests a reset link for a mailbox name in ``BLACKLISTED_MAILBOXES``, the system will ignore that request, even though the mailbox exists.
``APP_ROOT`` is the prefix the application will be served under. This has to be the same as later defined in ``uberspace web backend`` later, as the token link the app writes into the mail needs to point back to the application correctly.
``MAIL_SENDER`` is the address that should be in the ``FROM:`` field and ``MAIL_RECEIVER_DOMAIN`` is what should be appended with an ``@`` to the mailbox name when writing the ``TO:`` field of the mail.

UWSGI Configuration
-------------------

UWSGI ist the server we'll use to run the python/flask application. For that, we need to create a configuration file like this:

::

  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ cat uwsgi.ini 
  [uwsgi]
  mount = /mail_reset=main:app
  manage-script-name = true
  pidfile = uberspace_mail_pw_reset.pid
  master = true
  processes = 1
  http-socket = :1024
  chmod-socket = 660
  vacuum = true
  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ 

In the value for ``mount``, use the same prefix as for ``APP_ROOT`` in the ``.env`` file. If you don't want to have any prefix, use the line ``module = main:app`` instead of the lines starting with ``mount`` and ``manage-script-name``.

Create the service
------------------

Lastly, let's define it as a service so it can be automatically managed by :manual:`supervisord <daemons-supervisord>`. It should look like this:

::

  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ cat ~/etc/services.d/uberspace_mail_pw_reset_flask.ini
  [program:uberspace_mail_pw_reset_flask]
  directory=/home/isabell/mail_pw_reset
  command=/home/isabell/mail_pw_reset/bin/uwsgi /home/isabell/mail_pw_reset/uwsgi.ini
  (mail_pw_reset) [isabell@stardust mail_pw_reset]$ 


Finally, it's time to turn on the lights:

.. include:: includes/supervisord.rst
