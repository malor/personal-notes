The story of one subtle optimization
####################################

:date: 2013-02-24
:author: Roman Podolyaka
:tags: python, optimization
:category: programming
:slug: story-of-optimization
:lang: en


On my previous job I worked on an RPC-like Web-service that was passing JSON-blobs between
workers via the `Gearman <http://gearman.org/>`_ task queue. One day I was hacking on our source
code trying to find out why it took such a long time (a few seconds) to transfer a 3-megabyte
JSON-blob. I was sure that it was a problem in our code.  To my surprise the profiler showed that
the bottleneck was in this function of `python-gearman <https://github.com/Yelp/python-gearman/>`_
client library:

.. code-block:: python

   def read_data_from_socket(self, bytes_to_read=4096):
       """Reads data from socket --> buffer"""

       if not self.connected:
           self.throw_exception(message='disconnected')

       recv_buffer = ''
       try:
           recv_buffer = self.gearman_socket.recv(bytes_to_read)
       except socket.error, socket_exception:
           self.throw_exception(exception=socket_exception)

       if len(recv_buffer) == 0:
           self.throw_exception(message='remote disconnected')

       self._incoming_buffer += recv_buffer
       return len(self._incoming_buffer)


The problem is in the line: ``self._incoming_buffer += recv_buffer`` where the ``_incoming_buffer``
instance variable has the type ``str``. I was always taught that it is a bad idea to do a massive
string concatenation in a programming language where strings are considered to be immutable. So I
wrote a small snippet to find out how slow it really was:

.. code-block:: python

    def f():
        spam = ''
        eggs = 'a' * 4096

        for i in xrange(1000):
            spam += eggs


The ``timeit.timeit()`` function showed that it took less than **1 ms** to call the function ``f()``
that did 1000 string concatenation operations though. That was really amazing. I started to think
that I was missing something. So I tried to run the following snippet:

.. code-block:: python

    def g():
        class Foo:
            def __init__(self):
                self.spam = ''

        eggs = 'a' * 4096

        f = Foo()
        for i in xrange(1000):
            f.spam += eggs


It took about 450 ms - that's much "better" (or probably closer to my expectations)!
So what the heck? These two functions look very similar except that the latter uses an instance
attribute instead of a simple variable.

The bytecode comparison of these two function didn't show anything interesting: the for-loop
is completely the same except that the function ``g()`` uses ``LOAD_ATTR/STORE_ATTR`` bytecodes
instead of ``LOAD_FAST/STORE_FAST`` ones. But how can attribute access be so slow? It surely
must not be a bottleneck while concatenating strings in a loop.

Another interesting thing was that it took the same amount of time (about 1.6 s) to execute both
``f()`` and ``g()`` functions using the `PyPy <http://pypy.org/>`_ interpreter.

Now I was almost sure that `CPython <http://python.org/>`_ had some kind of optimization that
allowed it to execute the ``f()`` function so fast. But the second question, why it didn't work
for the ``g()`` function, remained.  Running ``python`` with ``strace`` showed that execution the
``g()`` function did almost 100 times more memory allocations than execution of the ``f()``
function. The time came to debug the interpreter using ``gdb``. The debugging allowed to find
the ``CPython`` function that did string concatenation.

The source code of ``string_concatenate()`` function had a few interesting lines:

.. code-block:: c

     if (v->ob_refcnt == 1 && !PyString_CHECK_INTERNED(v)) {
        /* Now we own the last reference to 'v', so we can resize it
         * in-place.
         */
        if (_PyString_Resize(&v, new_len) != 0) {
            /* XXX if _PyString_Resize() fails, 'v' has been
             * deallocated so it cannot be put back into
             * 'variable'.  The MemoryError is raised when there
             * is no value in 'variable', which might (very
             * remotely) be a cause of incompatibilities.
             */
            return NULL;
        }
        /* copy 'w' into the newly allocated area of 'v' */
        memcpy(PyString_AS_STRING(v) + v_len,
               PyString_AS_STRING(w), w_len);
        return v;
    }
    else {
        /* When in-place resizing is not an option. */
        PyString_Concat(&v, w);
        return v;
    }


So-so, strings actually **can** be mutable! This allows to resize ones in-place that makes strings
concatenation very fast. But there is a strict constraint - there must be **at most** 1 reference
to the string to be resized (otherwise the language contract that strings are immutable would be
broken).

It's important to understand that this is only a ``CPython``-specific optimization, and you'd better
not take benefit of it in your code! Everyone knows that string concatenation in ``Python`` is
a bad idea, so why to change our mind? If you really need to construct a string this way you should
read the docs for `str.join() <http://docs.python.org/2/library/stdtypes.html#str.join>`_
method, `StringIO <http://docs.python.org/2/library/stringio.html>`_ and
`array <http://docs.python.org/2/library/array.html>`_ modules first - this is the way to write
code that works efficiently in all ``Python`` interpreters.

**P. S.** I wrote a small patch for `python-gearman`_
that uses an ``array`` type for handling of incoming data that greatly improves the performance.
The patch was accepted in the **master** branch. Everyone who is interested in using of ``Gearman``
task queue in ``Python`` programs might want to check it out.
