Running uWSGI on Dreamhost shared hosting
*****************************************

Note: the following tutorial gives suggestions on how to name files with the objective of hosting multiple applications
on your account. You are obviously free to change naming schemes.

The tutorial assumes a shared hostign account, but it works even on the VPS offer (even if on such a system you have lot more freedom and you could use
better techniques to accomplish the result)


Preparing the environment
*************************

Log in via ssh to your account and move to the home (well, you should be already there after login).

Download a uWSGI tarball (anything >= 1.4 is good, but for maximum performance use >= 1.9), explode it and build it
normally (run make).

At the end of the procedure copy the resulting ``uwsgi`` binary to your home (just to avoid writing longer paths later).

Now move to the document root of your domain (it should be named like the domain) and put a file named ``uwsgi.fcgi`` in it with that content:

.. code-block:: sh

   #!/bin/sh
   /home/XXX/uwsgi /home/XXX/YYY.ini

change XXX with your account name and YYY with your domain name (it is only a convention, if you know what you are doing feel free to change it)

Give the file 'execute' permission

.. code-block:: sh

   chmod +x uwsgi.fcgi

Now in your home create a YYY.ini (remember to change YYY with your domain name) with that content

.. code-block:: ini

   [uwsgi]
   flock = /home/XXX/YYY.ini
   account = XXX
   domain = YYY

   protocol = fastcgi
   master = true
   processes = 3
   logto = /home/%(account)/%(domain).uwsgi.log
   virtualenv = /home/%(account)/venv
   module = werkzeug.testapp:test_app
   touch-reload = %p
   auto-procname = true
   procname-prefix-spaced = [%(domain)]

change the first three lines accordingly.

Preparing the python virtualenv
*******************************

As we want to run the werkzeug test app, we need to install its package in a virtualenv.

Move to the home:

.. code-block:: sh

   virtualenv venv
   venv/bin/easy_install werkzeug

The .htaccess
*************

Move again to the document root to create the .htaccess file that will instruct Apache to forward request to uWSGI

.. code-block:: sh

   RewriteEngine On
   RewriteBase /
   RewriteRule ^uwsgi.fcgi/ - [L]
   RewriteRule ^(.*)$ uwsgi.fcgi/$1 [L]

Ready
*****

Go to your domain and you should see the Werkzeug test page. If it does not show you can check uWSGI logs in the file you specified with the
logto option.

The flock trick
***************

As the apache mod_fcgi/mod_fastcgi/mod_fcgid implemenetations are very flaky on process management, you can easily end with lot of copies
of the same process running. The flock trick avoid that. Just remember that the flock option is very special as you cannot use
placeholder or other advanced techniques with it. You can only specify the absolute path of the file to lock.

Statistics
**********

As always remember to use uWSGI internal stats system

