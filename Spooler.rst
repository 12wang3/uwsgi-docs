The uWSGI Spooler
=================

Updated to uWSGI 2.0.1

Supported on: Perl, Python, Ruby

The Spooler is a queue manager built into uWSGI that works like a printing/mail system. 

You can enqueue massive sending of emails, image processing, video encoding, etc. and let the spooler do the hard work in background while your users get their requests served by normal workers.

A spooler works by defining a directory in which "spool files" will be written, every time the spooler find a file in its directory it will parse it and will run a specific function.

You can have multiple spoolers mapped to different directories and even multiple spoolers mapped to the same one.

The ``--spooler <directory>`` option allows you to generate a spooler process, while the ``--spooler-processes <n>`` allows you to set how many processes to spawn for every spooler.

The spooler is able to manage uWSGI signals too, so you can use it as a target for your handlers.

This configuration will generate a spooler for your instance (myspool directory must exists)

.. code-block:: ini

   [uwsgi]
   spooler = myspool
   ...
   
while this one will create two spoolers:

.. code-block:: ini

   [uwsgi]
   spooler = myspool
   spooler = myspool2
   ...

having multiple spoolers allows you to prioritize tasks (and eventually parallelize them)

Spool files
-----------

Spool files are serialized hash/dictionary of strings. The spooler will parse them and pass the resulting hash/dictionary to the spooler function (see below).

The serialization format is the same used for the 'uwsgi' protocol, so you are limited to 64k (even if there is a trick for passing bigger values, see the 'body' magic key below). The modifier1
for spooler packets is the 17, so a {'hello' => 'world'} hash will be encoded as:

========= ============== ==============
header    key1           value1
========= ============== ==============
17|14|0|0 |5|0|h|e|l|l|o |5|0|w|o|r|l|d
========= ============== ==============

A locking system allows you to safely manually remove spool files if something goes wrong, or to move them between spoolers directory.

Spool dirs over NFS are allowed, but if you do not have proper NFS locking in place, avoid mapping the same spooler NFS directory to spooler on different machines.

Setting the spooler function/callable
-------------------------------------

To have a fully operation spooler you need to define a "spooler function/callable".

Independently by the the number of configured spoolers, the same function will be executed. It is up to the developer
to instruct it to recognize tasks.

This function must returns an integer value:

-2 (SPOOL_OK) the task has been completed, the spool file will be removed

-1 (SPOOL_RETRY) something is temporarely wrong, the task will be retried at the next spooler iteration

0 (SPOOL_IGNORE) ignore this task, if multiple languages are loaded in the instance all of the will fight for magaing the task. This return values allows you to skip a task in specific languages.

any other value will be mapped as -1 (retry)


Each language plugin has its way to define the spooler function:

Perl:

.. code-block:: pl

   uwsgi::spooler(
       sub {
           my ($env) = @_;
           print $env->{foobar};
           return uwsgi::SPOOL_OK;
       }
   );
   
Python:

.. code-block:: py

   import uwsgi
   
   def my_spooler(env):
       print env['foobar']
       return uwsgi.SPOOL_OK
       
    uwsgi.spooler = my_spooler
    
Ruby:

.. code-block:: rb

   module UWSGI
        module_function
        def spooler(env)
                puts env.inspect
                return UWSGI::SPOOL_OK
        end
    end


Spooler function must be defined in the master process, so if you are in lazy-apps mode, be sure to place it in a file that is parsed
early in the server setup. (in python you can use --shared-import, in ruby --shared-require, in perl --perl-exec).

Some language plugin could have support for importing code directly in the spooler. Currently only python supports it with the ``--spooler-import`` option.


Enqueing requests to a spooler
------------------------------

The 'spool' api function allows you to enqueue a hash/dictionary into the spooler:

.. code-block:: py

   # python
   import uwsgi
   uwsgi.spool({'foo': 'bar', 'name': 'Kratos', 'surname': 'the same of Zeus'})
   # or
   uwsgi.spool(foo='bar', name='Kratos', surname='the same of Zeus')
   # for python3 use bytes instead of strings !!!


.. code-block:: pl

   # perl
   uwsgi::spool({foo => 'bar', name => 'Kratos', surname => 'the same of Zeus'})
   
.. code-block:: rb

   # ruby
   UWSGI.spool(foo => 'bar', name => 'Kratos', surname => 'the same of Zeus')
   
Some keys have special meaning:

'spooler' => specify the ABSOLUTE path of the spooler that has to manage this task

'at' => unix time at which the task must be executed (read: the task will not be run until the 'at' time is passed)

'priority' => this will be the subdirectory in the spooler directory in which the task will be placed, you can use that trick to give a good-enough prioritization to tasks (for better approach use multiple spoolers)

'body' => use this key for objects bigger than 64k, the blob will be appended to the serialzed uwsgi packet and passed back to the spooler function as the 'body' argument


IMPORTANT:

spool arguments must be strings (or bytes for python3), the api function will try to cast non-string values to strings/bytes, but do not rely on it !!!

External spoolers
-----------------

You could want to implement a centralized spooler for your server.

A single instance will manage all of the tasks enqueued by multiple uWSGI servers.

To accomplish this setup each uWSGI instance has to know which spooler directories are valid (consider it a form of security)

To add an external spooler directory use the ``--spooler-external <directory>`` option.

The spooler locking subsystem will avoid mess

Networked spoolers
------------------

You can even enqueue tasks over the network (be sure the 'spooler' plugin is loaded in your instance, but generally it is builtin by default)

As we have already seen, spooler packets have modifier1 17, you can directly send those packets to a uwsgi socket of an instance with a spooler enabled.

We will use the perl Net::uwsgi module (exposing a handy uwsgi_spool function) in this example (feel free to use whatever you want):

.. code-block:: perl

   use Net::uwsgi;
   uwsgi_spool('localhost:3031', {'test'=>'test001','argh'=>'boh','foo'=>'bar'});
   uwsgi_spool('/path/to/socket', {'test'=>'test001','argh'=>'boh','foo'=>'bar'});
   
.. code-block:: ini

   [uwsgi]
   socket = /var/run/uwsgi-spooler.sock
   socket = localhost:3031
   spooler = /path/for/files
   spooler-processes=1
   ...
   
(thanks brianhorakh for the example)

Post-poning tasks
-----------------

The 'body' magic key
--------------------

Priorities
----------

Options
-------
spooler=directory 
run a spooler on the specified directory

spooler-external=directory
map spoolers requests to a spooler directory managed by an external instance

spooler-ordered
try to order the execution of spooler tasks (uses scandir instead of readdir)

spooler-chdir=directory
call chdir() to specified directory before each spooler task

spooler-processes=##
set the number of processes for spoolers

spooler-quiet
do not be verbose with spooler tasks

spooler-max-tasks=##
set the maximum number of tasks to run before recycling a spooler (to help alleviate memory leaks)

spooler-harakiri=##
set harakiri timeout for spooler tasks, see [harakiri] for more information.

Tips and tricks
---------------

You can re-enqueue a spooler request by returning ``uwsgi.SPOOL_RETRY`` in your callable:

.. code-block:: py

    def call_me_again_and_again(env):
        return uwsgi.SPOOL_RETRY
    
You can set the spooler poll frequency using the ``--spooler-frequency <secs>`` option (default 30 seconds).

You could use the :doc:`Caching <caching framework>` or :doc:`SharedArea` to exchange memory structures between spoolers and workers.

Python (uwsgidecorators.py) and Ruby (uwsgidsl.rb) exposes higher-level facilities to manage the spooler, try to use them instead of the low-level approach described here.
