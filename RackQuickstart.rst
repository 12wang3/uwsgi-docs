Quickstart for ruby/Rack applications
=====================================

The following instructions will guide you through installing and running a ruby-based uWSGI distribution, aimed at running Rack apps.

Installing uWSGI with Ruby support
**********************************

To build uWSGI you need a c compiler (gcc and clang are supported) and the python binary (it will only run the uwsgiconfig.py script that will execute the various compilation steps). As we are building a uWSGI binary with ruby support we need ruby development headers too (uby-dev package on debian-based distros)

You can build uWSGI manually:

.. code-block:: sh

   make rack
   
that is the same as

.. code-block:: sh

   UWSGI_PROFILE=rack make
   
that is the same of

.. code-block:: sh

   make PROFILE=rack
  
and

.. code-block:: sh

   python uwsgiconfig.py --build rack
   
If you are lazy, you can download, build and install a uWSGI+ruby binary in a single shot:

.. code-block:: sh

   curl http://uwsgi.it/install | bash -s rack /tmp/uwsgi
   
Or, more ruby-friendly:

.. code-block:: sh

   gem install uwsgi
   
All of this methods build a "monolithic" uWSGI binary. The uWSGI project is composed by dozens of plugins, you can choose to build the server core and having a plugin for every feature (that you will load when needed), or you can
build a single binary with the features you need. This kind of builds are called 'monolithic'.

This quickstart assumes a monolithic binary (so you do not need to load plugins). If you prefer to use your package distributions (instead of building uWSGI from official sources), see below

Note for distro packages
************************

You distribution very probably contains a uWSGI package set. Those uWSGI packages tend to be highly modulars, so in addition to the core you need to install the required plugins. Plugins must be loaded in your configs. In the learning phase we strongly suggest to not use distribution packages to easily follow documentation and tutorials.

Once you feel confortable with the “uWSGI way” you can choose the best approach for your deployments.

As an example, the tutorial makes use of the 'http' and 'rack' plugins. If you are using a modular build be sure to load the with the ``--plugins http,rack`` option

Your first Rack app
*******************

Rack is the standard way for writing ruby web apps.

This is a standard Rack Hello world script (call it app.ru):

.. code-block:: rb

   class App

     def call(environ)
       [200, {'Content-Type' => 'text/html'}, ['Hello']]
     end
     
   end
   
   run App.new
   
The .ru extension stands for "rackup" that is the deployment tool included in the rack distribution. Rackup uses a little DSL, so to use it into uWSGI you need to install the rack gem:

.. code-block:: sh

   gem install rack
   
Now we are ready to deploy with uWSGI:

.. code-block:: sh

   uwsgi --http :8080 --http-modifier1 7 --rack app.ru

(remember to replace ‘uwsgi’ if it is not in your current $PATH)

or if you are using a modular build (like the one of your distro)

.. code-block:: sh

   uwsgi --plugins http,rack --http :8080 --http-modifier1 7 --rack app.ru

What is that '--http-modifier1 5' thing ???
*******************************************

uWSGI supports various languages and platform. When the server receives a request it has to know where to 'route' it.

Each uWSGI plugin has an assigned number (the modifier), the perl/psgi one has the 5. So --http-modifier1 5 means "route to the psgi plugin"

Albeit uWSGI has a more "human-friendly" :doc:`internal routing system <InternalRouting>` using modifiers is the fastest way, so, if possible always use them


Using a full webserver: nginx
*****************************

The supplied http router, is (yes, incredible) only a router. You can use it as a load balancer or a proxy, but if you need a full webserver (for efficiently serving static files or all of those task a webserver is good at), you can get rid of the uwsgi http router (remember to change --plugins http,psgi to --plugins psgi if you are using a modular build) and put your app behind nginx.

To communicate with nginx, uWSGI can use various protocol: http, uwsgi, fastcgi, scgi...

The most efficient one is the uwsgi one. Nginx includes uwsgi protocol support out of the box.

Run your psgi application on a uwsgi socket:

.. code-block:: sh

   uwsgi --socket 127.0.0.1:3031 --psgi myapp.pl

then add a location stanza in your nginx config


.. code-block:: c

   location / {
       include uwsgi_params;
       uwsgi_pass 127.0.0.1:3031;
       uwsgi_modifier1 5;
   }

Reload your nginx server, and it should start proxying requests to your uWSGI instance

Note that you do not need to configure uWSGI to set a specific modifier, nginx will do it using the ``uwsgi_modifier1 5;`` directive

Adding concurrency
******************

Adding robustness: the Master process
*************************************

It is highly recommended to have the master process always running on productions apps.

It will constantly monitor your processes/threads and will add funny features like the :doc:`StatsServer`

To enable the master simply add --master

.. code-block:: sh

   uwsgi --socket 127.0.0.1:3031 --psgi myapp.pl --processes 4 --master
   
Using config files
******************

uWSGI has literally hundreds of options. Dealing with them via command line is basically silly, so try to always use config files.
uWSGI supports various standards (xml, .ini, json, yaml...). Moving from one to another is pretty simple. The same options you can use via command line can be used
on config files simply removing the ``--`` prefix:

.. code-block:: ini

   [uwsgi]
   socket = 127.0.0.1:3031
   psgi = myapp.pl
   processes = 4
   master = true
   
or xml:

.. code-block:: xml

   <uwsgi>
     <socket>127.0.0.1:3031</socket>
     <psgi>myapp.pl</psgi>
     <processes>4</processes>
     <master/>
   </uwsgi>
   
To run uWSGI using a config file, just specify it as argument:

.. code-block:: sh

   uwsgi yourconfig.ini
   
if for some reason your config cannot end with the expected extension (.ini, .xml, .yml, .js) you can force the binary to
use a specific parser in this way:

.. code-block:: sh

   uwsgi --ini yourconfig.foo
   
.. code-block:: sh

   uwsgi --xml yourconfig.foo

.. code-block:: sh

   uwsgi --yaml yourconfig.foo

and so on

You can even pipe configs (using the dash to force reading from stdin):

.. code-block:: sh

   perl myjsonconfig_generator.pl | uwsgi --json -


Automatically starting uWSGI on boot
************************************

If you are thinking about writing some init.d script for spawning uWSGI, just sit (and calm) down and check if your system does not offer you a better (more modern) approach.

Each distribution has choosen its startup system (:doc:`Upstart<Upstart>`, :doc:`SystemD`...) and there are tons of process managers available (supervisord, god...).

uWSGI will integrate very well with all of them (we hope), but if you plan to deploy a big number of apps check the uWSGI :doc:`Emperor<Emperor>`
it is the dream of every devops.

Security and availability
*************************

ALWAYS avoid running your uWSGI instances as root. You can drop privileges using the uid and gid options

.. code-block:: ini

   [uwsgi]
   socket = 127.0.0.1:3031
   uid = foo
   gid = bar
   chdir = path_toyour_app
   psgi = myapp.pl
   master = true
   processes = 8


A common problem with webapp deployment is "stuck requests". All of your threads/workers are stuck blocked on a request and your app cannot accept more of them.

To avoid that problem you can set an ``harakiri`` timer. It is a monitor (managed by the master process) that will destroy processes stuck for more than the specified number of seconds

.. code-block:: ini

   [uwsgi]
   socket = 127.0.0.1:3031
   uid = foo
   gid = bar
   chdir = path_toyour_app
   psgi = myapp.pl
   master = true
   processes = 8
   harakiri = 30

will destroy workers blocked for more than 30 seconds. Choose carefully the harakiri value !!!

In addition to this, since uWSGI 1.9, the stats server exports the whole set of request variables, so you can see (in realtime) what your instance is doing (for each worker, thread or async core)

Enabling the stats server is easy:

.. code-block:: ini

   [uwsgi]
   socket = 127.0.0.1:3031
   uid = foo
   gid = bar
   chdir = path_toyour_app
   psgi = myapp.pl
   master = true
   processes = 8
   harakiri = 30
   stats = 127.0.0.1:5000
   
just bind it to an address (UNIX or TCP) and just connect (you can use telnet too) to it to receive a JSON representation of your instance.

The ``uwsgitop`` application (you can find it in the official github repository) is an example of using the stats server to have a top-like realtime monitoring tool (with colors !!!)


Offloading
**********

:doc:`OffloadSubsystem` allows you to free your workers as soon as possible when some specific pattern matches and can be delegated
to a pure-c thread. Examples are sending static file from the filesystem, transferring data from the network to the client and so on.

Offloading is very complex, but its use is transparent to the end user. If you want to try just add --offload-threads <n> where <n> is the number of threads to spawn (one for cpu is a good value).

When offload threads are enabled, all of the parts that can be optimized will be automatically detected


And now
*******

You should already be able to go in production with such few concepts, but uWSGI is an enormous project with hundreds of features
and configurations. If you want to be a better sysadmin, continue reading the full docs.
