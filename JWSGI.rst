The JWSGI interface
===================

.. note:: JWSGI is not a standard. Yet. If you like JWSGI, why not send an RFC to the uWSGI mailing list. We have no interest in a standard, but who knows...

JWSGI is a port of the WSGI/PSGI/Rack way of thinking for Java.

If, for some obscure reason, you'd feel like developing apps with JVM languages and you don't feel like deploying a huge servlet stack, JWSGI should be up your alley.

It is a very simple protocol where you call a public method that takes a ``HashMap`` as its sole argument.
This HashMap contains CGI style variables and ``jwsgi.input`` containing a Java InputStream object.

The function has to returns an array of 3 Objects:

status (java/lang/Integer) Example '200'

headers (HashMap) Example {"Content-type": "text/html", "Server": "uWSGI", "Foo": ["one","two"]}

body (can be a String, an array of String, a File or an InputStream object)

Example
-------

A simple JWSGI app looks like this:

.. code-block:: java

	public class MyApp {

	    public static Object[] application(HashMap env) {

	        int status = 200;

	        HashMap headers = new HashMap<Object,Object>();
                headers.put("Content-type", "text/html");
	        headers.put("Server", "uWSGI");

	        String body = "<h1>Hello World</h1>" + env.get("REQUEST_URI");

	        Object[] response = {status, headers, body};

	        return response;
	    }
	}



How to use it?
--------------


1. Compile your class with ``javac``.

   .. code-block:: sh

      javac utest.java

4. Run uWSGI and load the utest class.

   .. code-block:: sh

      ./uwsgi -s :3031 --plugins jvm,jwsgi --jvm-class utest -M -p 4 -m

