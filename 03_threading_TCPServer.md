## Threading version

```python
import socketserver
import time


RESPONSE = """\
HTTP/1.1 200k OK

Hello from Xiaowei.
""".encode(encoding='utf-8')


class MyHandler(socketserver.BaseRequestHandler):

	def handle(self):
		time.sleep(10)
		self.request.recv(1024)
		self.request.send(RESPONSE)


server = socketserver.ThreadingTCPServer(('localhost', 2017), MyHandler)
server.serve_forever()
```

The only differece is now we are using **`ThreadingTCPServer`** instead of **`TCPServer`**.

## Definition of ThreadingTCPServer

```python
class ThreadingMixIn:
    """Mix-in class to handle each request in a new thread."""

    # Decides how threads will act upon termination of the
    # main process
    daemon_threads = False

    def process_request_thread(self, request, client_address):
        """Same as in BaseServer but as a thread.
        In addition, exception handling is done here.
        """
        try:
            self.finish_request(request, client_address)
        except Exception:
            self.handle_error(request, client_address)
        finally:
            self.shutdown_request(request)

    def process_request(self, request, client_address):
        """Start a new thread to process the request."""
        t = threading.Thread(target = self.process_request_thread, args = (request, client_address))
        t.daemon = self.daemon_threads
        t.start()
        
        


class ThreadingTCPServer(ThreadingMixIn, TCPServer): pass
```

We can see that **`ThreadingTCPServer`** is basically just a **`TCPServer`** with a **`ThreadingMixIn`**.

So basically, all **Steps** metioned in the non-threading version applies here except that:

When try to call **`process_request()`** in **`BaseServer`** class, now we call above version in **`ThreadingMixIn`**, it does:

* start a new thread
* this new threading will run **`process_request_thread`** in **`ThreadingMixIn`**, which will eventually calls **`self.finish_request(request, client_address)`** in **`BaseServer`** and it continues the rest the same as non-threaidng version.

So basically, all it does is: instead of calling **`self.finish_request(request, client_address)`** directly in non-threading version, it starts a new thread to process the request.