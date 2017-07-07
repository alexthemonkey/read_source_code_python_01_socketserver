## 1. Start from a simple server

```python
import socket


HOST, PORT = '', 4000
MAX_QUEUED_CONNECTIONS = 1
RESPONSE = b"""\
HTTP/1.1 200k OK

Hello from Xiaowei.
"""

listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
listen_socket.bind((HOST, PORT))
listen_socket.listen(MAX_QUEUED_CONNECTIONS)

print('Serving HTTP on port : {}'.format(PORT))

while True:
	print("Before...")
	client_connection, client_address = listen_socket.accept()
	print("After...")
	request = client_connection.recv(1024)
	print("Request: {}".format(request))

	client_connection.sendall(RESPONSE)
	client_connection.close()
```

if we run this server, then we will see the console prints `Before...` and stops there. That is because the `.accept()` method actually waits for the internet request, so it waits there. Once there is a request come in, we can see that it will continue and print `After...` and so on.


## 2. Something strange in `serve_forever()`

In the code of `serve_forvever` in **BaseServer** class, we do something like this:

```python
import selectors

if hasattr(selectors, 'PollSelector'):
    _ServerSelector = selectors.PollSelector
else:
    _ServerSelector = selectors.SelectSelector
    
...

with _ServerSelector() as selector:

    selector.register(self, selectors.EVENT_READ)
   
    while not self.__shutdown_request:
        ready = selector.select(poll_interval)       
        if ready:
            self._handle_request_noblock()   # eventually calls socket.accept() method

        self.service_actions()
```

what does that `selector` do here?

basically, **We are using I/O multiplexing here.**


## 3. What is I/O multiplexing?





