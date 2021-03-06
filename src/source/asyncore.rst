asyncore
~~~~~~~~

Python's asyncore provides an event loop that can handle
transactions from multiple non-blocking sockets.

.. note::

    asyncore is not **threadsafe**.  The use of the map
    paramter in *asyncore.dispatcher.__init__()* and in
    *asyncore.loop()* is required for asyncore to work
    in multiple threads.

**Import Syntax**

.. code-block:: python

    import asyncore

**__init__ parameters**

.. code-block:: python

    asyncore.dispatcher.__init__(self, sock, map)

*sock*: is the socket to be passed in if it exists
already.

*map*: is a dictionary for use when different asyncore
loops are running on different threads.  The same map is
passed into *asyncore.loop()*

**asyncore.dispatcher.** Subclass asyncore.dispatcher:

.. code-block:: python

    import asyncore
    import socket

    class MyDispatcher(asyncore.dispatcher):

        def __init__(self, map=None):
            self.asyncore.dispatcher.__init__(self, map=map)
            self.create_socket(socket.AF_INET, socket.SOCK_STREAM)
            self.connect(("example.com",2222))
            self.out_buffer = ''
            self.in_buffer = ''


**Override methods.**  The methods that
#need to be overridden are listed below.

.. code-block:: python

        def handle_read(self):
            # Do something with data
            data = self.recv(4096)
            self.in_buffer += data

        def handle_write(self):
            #Data must be placed in a buffer somewhere.
            #(In this case out_buffer)
            sent = self.send(self.out_buffer)
            self.out_buffer = self.out_buffer[sent:]

        def readable(self):
            #Test for select() and friends
            return True

        #There is no 'e' in 'writeable' here.
        def writable(self):
            #Test for select(). Must have data to write
            #otherwise select() will trigger
            if self.connected and len(self.out_buffer) > 0: 
                return True
            return False

        def handle_close(self):
            #Flush the buffer
            while self.writable():
                self.handle_write()
            self.close()

**Provide a public api for writing():**  While the dispatcher has a *send()*
function, but that is for the use of the *handle_write()* function.  Create
a public api *write_data()* function which will add data to the buffer.  

The problem with using the *dispatcher.send()* function directly is that the sockets
are set in non-blocking mode.  *send()* might return a value that indicates 
that only part of the buffer was written.  It is better to let the 
*handle_write()* write the data above, as the *asyncore.loop()* will call 
*handle_write()* until all the data gets written.

.. code-block:: python

        @api
        def write_data(self, data):
            '''
            Public facing interface method.  This is the function
            external code will use to send data to this dispatcher.
            '''
            self.out_buffer += data



For dispatchers binding to a socket, 
*handle_accept()* must be provided as well
as the other handler functions necessary.

.. code-block:: python

    def __init__(self, bind_ip, bind_port):
        asyncore.dispatcher.__init__(self)
        self.bind((bind_ip, bind_port))
        self.listen(5)

    [...]

    def handle_accept(self):
        #Do something with the new socket
        port, dest = self.accept()


**asyncore.dispatcher_with_send**
This is just like the normal dispatcher, except that the writable() and
*handle_write()* methods have been already provided.  **Note:** to prevent
data loss upon the close provide a *handle_close()* similar to the one
listed above.

**asyncore.file_dispatcher**
Instead of a socket, a file descriptor is passed in.  Asyncore will wrap the
fd to be able to be called with the recv() and send() parameters.  This is
useful for devices which cannot be described from the standpoint as a socket (e.g. /dev/net/tun)

**Call asyncore.loop().**

.. code-block:: python

  asyncore.loop(timeout=30.0, use_poll=False, map=None, count=None)

*timeout:* Timeout in seconds.

*use_poll:* Use *poll()* instead of *select()*.

*map:* This is the same dictionary used with the optional map 
argument for the asyncore.dispatcher initializer.

*count:* The number of times to run through the loop.  This could make 
loop wait as long as count * timeout.

**Using asyncore with threads.** Asyncore keeps a global map 
keeping track of dispatchers to sockets.  With threading, this
map can be changed on the fly while the event loop is running.

To use with threads, a dictionary used exclusively by the thread is
passed with the map paramter.  Using the MyDispatcher example above:

.. code-block:: python

    #Create a dictionary
    map = {} 

    #MyDispatcher will pass this to
    #asyncore.dispatcher
    d = MyDispatcher(map=map)

    #Pass the map into the loop
    asyncore.loop(timeout=0.1, map=map)

If a dispatcher creates another dispatcher class which will need to be run 
in the same thread, the *map* must be the same map 
that was passed to the *__init__()* function of the parent class.  It is
likely this will be the *_map* attribute of the parent dispatcher. 

Also, *close()* will remove the dispatcher from the map it was assigned to 
automatically.  This could be due to a *socket.error* or a close event on
the other side.
