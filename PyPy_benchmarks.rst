Benchmarks for the PyPy plugin
==============================

This is mainly targeted at PyPy developers to spot slow paths or to fix corner-case bug.

uWSGI stresses lot of areas of PyPy (most of them rarely abused in pure-python apps), so making benchmarks is good both for uWSGI and PyPy

Results are rounded for easy of read, each test is executed 10 times on a Intel i7-3615QM CPU @ 2.30GHz.

CPython version is 2.7.5, pypy is latest tip at 23 of May 2013

Tests are run with logging disabled

Generally the command lines are:

.. code-block:: sh

   uwsgi --http-socket :9090 --wsgi hello --disable-logging
   
and

.. code-block:: sh

   uwsgi --http-socket :9090 --pypy-home /opt/pypy --pypy-wsgi hello --disable-logging

Simple Hello World
^^^^^^^^^^^^^^^^^^

The most useless of the tests (as it shows only how uWSGI performs instead of the python engine).

.. code-block:: py

   def application(e, sr):
       sr('200 Ok', [('Content-Type', 'text/html')])
       return "ciao"

CPython: 6500 requests per-second, memory used 7MB (no leak detected)

syscalls:

.. code-block:: sh

   0.000403 gettimeofday({1369293059, 218207}, NULL) = 0
   0.000405 read(5, "GET / HTTP/1.1\r\nUser-Agent: curl/7.30.0\r\nHost: ubuntu64.local:9090\r\nAccept: */*\r\n\r\n", 4096) = 83
   0.000638 write(5, "HTTP/1.1 200 Ok\r\nContent-Type: text/html\r\n\r\n", 44) = 44
   0.000678 write(5, "ciao", 4)       = 4
   0.000528 gettimeofday({1369293059, 220477}, NULL) = 0
   0.000394 close(5)

PyPy: 6560 requests per-second, memory used 71MB (no leak detected)

syscalls:

no differences with CPython

Considerations:

there is only slightly (read: irrelevant) better performance in PyPy, but memory usage is 10x higher with PyPy. It needs to be investigated.


CPU-Bound test (fibonacci)
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: py

   def fib(n):
      if n == 0:
          return 0
      if n == 1:
          return 1
      return fib(n-1) + fib(n-2)

   def application(e, sr):
      sr('200 Ok', [('Content-Type', 'text/html')])
      fib(36)
      return "ciao"


This is where PyPy shines.

CPython: time-to-complete 6400 milliseconds, memory used 65 MB

PyPy: time-to-complete 900 milliseconds, memory used 71 MB

Response time is astonishing, there is no debate about how PyPy is better for CPU intensive tasks (and with high grade of recursion)
, but more interesting is how the memory usage of PyPy remains the same of the simple hello world, while CPython increased 10x

Syscall usage is again the same

Werkzeug testapp
^^^^^^^^^^^^^^^^

You may think this is not very different from the hello world, but this specific application call lot of python functions
and inspect the whole WSGI environ dictionary. This is very near to a standard application without I/O

CPython: 600 requests per seconds, memory usage 13MB

syscalls

.. code-block:: sh

   0.000363 gettimeofday({1369294531, 360307}, NULL) = 0
   0.000421 read(5, "GET / HTTP/1.1\r\nUser-Agent: curl/7.30.0\r\nHost: ubuntu64.local:9090\r\nAccept: */*\r\n\r\n", 4096) = 83
   0.002046 getcwd("/root/uwsgi", 1024) = 12
   0.000483 stat("/root/uwsgi/.", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.000602 stat("/usr/local/lib/python2.7/dist-packages/greenlet-0.4.0-py2.7-linux-x86_64.egg", {st_mode=S_IFDIR|S_ISGID|0755, st_size=4096, ...}) = 0
   0.000530 stat("/usr/local/lib/python2.7/dist-packages/gevent-1.0dev-py2.7-linux-x86_64.egg", {st_mode=S_IFDIR|S_ISGID|0755, st_size=4096, ...}) = 0
   0.000506 stat("/usr/lib/python2.7", {st_mode=S_IFDIR|0755, st_size=28672, ...}) = 0
   0.000440 stat("/usr/lib/python2.7/plat-x86_64-linux-gnu", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000463 stat("/usr/lib/python2.7/lib-tk", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000624 stat("/usr/lib/python2.7/lib-old", 0x7fffb70da6a0) = -1 ENOENT (No such file or directory)
   0.000434 stat("/usr/lib/python2.7/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.000515 stat("/usr/local/lib/python2.7/dist-packages", {st_mode=S_IFDIR|S_ISGID|0775, st_size=4096, ...}) = 0
   0.000569 stat("/usr/lib/python2.7/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.000387 stat("/usr/lib/python2.7/dist-packages/gtk-2.0", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000347 stat("/usr/lib/pymodules/python2.7", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000675 write(5, "HTTP/1.1 200 OK\r\nContent-Type: text/html; charset=utf-8\r\nContent-Length: 7554\r\n\r\n", 81) = 81
   0.000575 write(5, "<!DOCTYPE HTML PUBLIC \"-//W3C//DTD HTML 4.01 Transitional//EN\"\n  \"http://www.w3.org/TR/html4/loose.dtd\">\n<title>WSGI Information</title>\n<style type=\"text/css\">\n  @import url(http://fonts.googleapis.com/css?family=Ubuntu);\n\n  body       { font-family: 'Lucida Grande', 'Lucida Sans Unicode', 'Geneva',\n               'Verdana', sans-serif; background-color: white; color: #000;\n               font-size: 15px; text-align: center; }\n  #logo      { float: right; padding: 0 0 10px 10px; }\n  div.box    { text-align: left; width: 45em; margin: auto; padding: 50px 0;\n               background-color: white; }\n  h1, h2     { font-family: 'Ubuntu', 'Lucida Grande', 'Lucida Sans Unicode',\n               'Geneva', 'Verdana', sans-serif; font-weight: normal; }\n  h1         { margin: 0 0 30px 0; }\n  h2         { font-size: 1.4em; margin: 1em 0 0.5em 0; }\n  table      { width: 100%; border-collapse: collapse; border: 1px solid #AFC5C9 }\n  table th   { background-color: #AFC1C4; color: white; font-size: "..., 7554) = 7554
   0.000469 gettimeofday({1369294531, 370471}, NULL) = 0
   0.000391 close(5)                  = 0

PyPy: 1500 requests per seconds, memory usage 74MB

syscalls

.. code-block:: sh

   0.000397 gettimeofday({1369294713, 743204}, NULL) = 0
   0.000431 read(5, "GET / HTTP/1.1\r\nUser-Agent: curl/7.30.0\r\nHost: ubuntu64.local:9090\r\nAccept: */*\r\n\r\n", 4096) = 83
   0.003217 gettimeofday({1369294713, 746909}, NULL) = 0
   0.000660 gettimeofday({1369294713, 747509}, NULL) = 0
   0.000958 gettimeofday({1369294713, 748463}, NULL) = 0
   0.000359 gettimeofday({1369294713, 748832}, NULL) = 0
   0.000586 gettimeofday({1369294713, 749427}, NULL) = 0
   0.000660 gettimeofday({1369294713, 750077}, NULL) = 0
   0.000626 gettimeofday({1369294713, 750695}, NULL) = 0
   0.000318 gettimeofday({1369294713, 751010}, NULL) = 0
   0.000598 gettimeofday({1369294713, 751586}, NULL) = 0
   0.000782 gettimeofday({1369294713, 752391}, NULL) = 0
   0.000738 gettimeofday({1369294713, 753129}, NULL) = 0
   0.000355 gettimeofday({1369294713, 753483}, NULL) = 0
   0.000617 gettimeofday({1369294713, 754156}, NULL) = 0
   0.000502 gettimeofday({1369294713, 754649}, NULL) = 0
   0.000484 gettimeofday({1369294713, 755139}, NULL) = 0
   0.000513 gettimeofday({1369294713, 755674}, NULL) = 0
   0.001537 getcwd("/opt/uwsgi", 256) = 12
   0.000641 stat("/opt/uwsgi/.", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.000668 stat("/opt/pypy/site-packages/setuptools-0.6c11-py2.7.egg", {st_mode=S_IFREG|0644, st_size=332005, ...}) = 0
   0.000766 stat("/opt/pypy/site-packages/pip-1.3.1-py2.7.egg", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000652 stat("/opt/pypy/lib_pypy/__extensions__", 0x7ff66a446030) = -1 ENOENT (No such file or directory)
   0.000570 stat("/opt/pypy/lib_pypy", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000688 stat("/opt/pypy/lib-python/2.7", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.000592 stat("/opt/pypy/lib-python/2.7/lib-tk", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000534 stat("/opt/pypy/lib-python/2.7/plat-linux2", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000840 stat("/opt/pypy/site-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
   0.000592 stat("/opt/uwsgi/.", {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
   0.001014 write(5, "HTTP/1.1 200 OK\r\nContent-Type: text/html; charset=utf-8\r\nContent-Length: 4915\r\n\r\n", 81) = 81
   0.000510 write(5, "<!DOCTYPE HTML PUBLIC \"-//W3C//DTD HTML 4.01 Transitional//EN\"\n  \"http://www.w3.org/TR/html4/loose.dtd\">\n<title>WSGI Information</title>\n<style type=\"text/css\">\n  @import url(http://fonts.googleapis.com/css?family=Ubuntu);\n\n  body       { font-family: 'Lucida Grande', 'Lucida Sans Unicode', 'Geneva',\n               'Verdana', sans-serif; background-color: white; color: #000;\n               font-size: 15px; text-align: center; }\n  #logo      { float: right; padding: 0 0 10px 10px; }\n  div.box    { text-align: left; width: 45em; margin: auto; padding: 50px 0;\n               background-color: white; }\n  h1, h2     { font-family: 'Ubuntu', 'Lucida Grande', 'Lucida Sans Unicode',\n               'Geneva', 'Verdana', sans-serif; font-weight: normal; }\n  h1         { margin: 0 0 30px 0; }\n  h2         { font-size: 1.4em; margin: 1em 0 0.5em 0; }\n  table      { width: 100%; border-collapse: collapse; border: 1px solid #AFC5C9 }\n  table th   { background-color: #AFC1C4; color: white; font-size: "..., 4915) = 4915
   0.000729 gettimeofday({1369294713, 766079}, NULL) = 0
   0.000616 close(5)                  = 0

Considerations:

this tests stresses standard function calls, we have about 2.5x improvement with PyPy, while memory usage is pretty similar (considering the 62MB base difference)

There is a syscall "problem" with PyPy, as soon before starting the path checks, it calls a blast of gettimeofday() syscalls.

Without them, request per-seconds could increase a bit
