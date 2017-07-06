## start with an example

```python
import socketserver


RESPONSE = """\
HTTP/1.1 200k OK

Hello from Xiaowei.
""".encode(encoding='utf-8')

SERVER_ADDRESS = (HOST, PORT) = 'localhost', 2017


class MyHandler(socketserver.BaseRequestHandler):

	def handle(self):
		self.request.recv(1024)
		self.request.send(RESPONSE)


server = socketserver.TCPServer(SERVER_ADDRESS, MyHandler)
server.serve_forever()
```

Now if we go to `http://localhost:2017/`, we will get response `Hello from Xiaowei.`. So now it is already a working tcp server and all we need is only one module  : **socketserver**

Now let's follow the actions step by step to see what excactly happend.

## step 1: `server = socketserver.TCPServer(SERVER_ADDRESS, MyHandler)`

Here we create a server which is an instance of class **TCPServer**, which inherits **BaseServer**.

`__init__()` in **BaseServer**
```python
class BaseServer:
	def __init__(self, server_address, RequestHandlerClass):
    	"""Constructor.  May be extended, do not override."""
    	self.server_address = server_address
    	self.RequestHandlerClass = RequestHandlerClass
    	self.__is_shut_down = threading.Event()
    	self.__shutdown_request = False
```

`__init__()` in **TCPServer**
```python
class TCPServer(BaseServer):

	     address_family = socket.AF_INET
            socket_type = socket.SOCK_STREAM
     request_queue_size = 5
    allow_reuse_address = False
    
	def __init__(self, server_address, RequestHandlerClass, bind_and_activate=True):
    	"""Constructor.  May be extended, do not override."""
    	BaseServer.__init__(self, server_address, RequestHandlerClass)
    	self.socket = socket.socket(self.address_family, self.socket_type)
    	if bind_and_activate:
        	try:
            	self.server_bind()
            	self.server_activate()
        	except:
            	self.server_close()
            	raise
```

We passed the `host` (localhost in out case) and `port` (2017) and custom handler `MyHandler` to the `TCPServer` `__init__` function, and it basically did 3 things:
* pass `host`, `port` and `MyHandler` to the instance server.
* do the binding : `self.server_bind()`
* and let socket listen to the right host and port: `self.server_activate()`

Here is the detail:

in **TCPServer**
```python
class TCPServer(BaseServer):

	...
    
	def server_bind(self):
        if self.allow_reuse_address:
            self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.socket.bind(self.server_address)     # binding the socket to the host and port
        self.server_address = self.socket.getsockname()
     
    def server_activate(self):
        self.socket.listen(self.request_queue_size)  # tell os that the socket is a listening socket 
```

## step 2: `server.serve_forever()`

Now we into `server.serve_forever()`, which eventually goes into a while loop, which constantly checking if there is a request come in.

```python
class BaseServer:

	...
   
	def serve_forever(self, poll_interval=0.5):
        self.__is_shut_down.clear()
        try:
            with _ServerSelector() as selector:
                selector.register(self, selectors.EVENT_READ)

                while not self.__shutdown_request:
                    ready = selector.select(poll_interval)
                    if ready:
                        self._handle_request_noblock()

                    self.service_actions()
        finally:
            self.__shutdown_request = False
            self.__is_shut_down.set()
            
     def service_actions(self):
        """Called by the serve_forever() loop.
        May be overridden by a subclass / Mixin to implement any code that
        needs to be run during the loop.
        """
        pass
```

till now, if no one try to reach `localhost:2017`, the `ready` in the code above will be always `False` and the server will keep on running `self.service_actions()` but not firing `self._handle_request_noblock()`. 

By reading the comment of `service_actions` method, we can see that we can controll the behavior of server when no request comes in.

## step3:  assuming there is a request come in

Then it goes into `self._handle_request_noblock()`, then invoke `self.process_request(request, client_address)` which invokes `self.finish_request(request, client_address)` which eventually calls ` self.RequestHandlerClass(request, client_address, self)`

```python
class BaseServer:
	
    ...
   
	def _handle_request_noblock(self):
        try:
            request, client_address = self.get_request()
        except OSError:
            return
        if self.verify_request(request, client_address):
            try:
                self.process_request(request, client_address)
            except Exception:
                self.handle_error(request, client_address)
                self.shutdown_request(request)
            except:
                self.shutdown_request(request)
                raise
        else:
            self.shutdown_request(request)


    def process_request(self, request, client_address):
        self.finish_request(request, client_address)
        self.shutdown_request(request)

    def finish_request(self, request, client_address):
        self.RequestHandlerClass(request, client_address, self)


```

And we know that ` self.RequestHandlerClass` is a class we passed, which is `MyHandler` in this case.

```python
class BaseRequestHandler:

    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()

    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass
```
We can see that ` self.RequestHandlerClass(request, client_address, self)` basically goes into `self.setup()` and `self.handle()`, and we overwrite the `self.handle()` in `MyHandler`, and our custom handler will be called and return the data we want to return.


Then, in `process_request`, we call `self.shutdown_request(request)` after the server handled the quest ( `self.finish_request(request, client_address)` ), and it eventually goes into `close_request` method in **TCPServer** to close the client connection.


Above 3 steps are shown in the pic below: (some methods are simplified or neglected)
* red    =>   step 1
* blue   =>   step 2
* pink and green   =>   step 3


![alt text](structure_of_socketserver.png)



















