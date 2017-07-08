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

basically, **We are using I/O multiplexing here.**  What is I/O multiplexing? Before we get into that, there are some concepts need to be known.

## 3. Concepts

### 3.1 user space vs kernal space

![alt text](userspace-kernelspace.png)

Kernel modules run in kernel space and applications run in user space, as illustrated in Figure 2. Both kernel space and user space have their own unique memory address spaces that do not overlap. 

### 3.2 process switch

![alt text](5_state_process_model.png)

Multiple process take turns runing on the CPU. Thus, CPU might suspend some process and resume some other suspended process to continue, which is called `process switch`. A lot of things happen when switching:

* Event occurrence
* Save processor state (including program counter and other register) in **PCB** (process control block)
* Update PCB with new state and any accounting information
* Move the PCB to appropriate queue-ready, blocked
*  Select another process for execution
*  Update PCB of the process selected (new state and any accounting information)
*  update memory management data structures
*  Restore context of the selected process


### 3.3 blocking a process

When a process is waiting for some event(s) to happen, it will block itself to not using the cpu resource.

### 3.4 file descriptor

In Unix and related computer operating systems, a file descriptor is an abstract indicator (handle) used to access a file or other input/output resource, such as a pipe or network socket. 

Basically,
* file descriptor is a non-negtive integer
* the file descriptor is returned **by kernal**, **to process** when:
		1. opens an existing file,
		2. creates a new file or
		3. creates a new socket.

```python
import sys
import os


print( sys.stdin.fileno() )  # 0
print( sys.stdout.fileno() ) # 1
print( sys.stderr.fileno() ) # 2


os.write(0, b'hello Xiaowei\n')  # will print : hello Xiaowei


################################################################################
#   socket fileno
################################################################################

import socket

sock1 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
print( sock1.fileno(), sock2.fileno() )  #  3,  4
```

### 3.5 cache I/O  vs  direct I/O

**Cache I/O** : os will cache the data of I/O in kernal memory (page caches) and then copy the data to the memory location of an application. This is the very often I/O type.  

One drawback if this one is the data copying between kernal and application.

**direct I/O**: 


## 4. Linux I/O  (5 models)
* blocking I/O
* non-blocking I/O
* multiplexing I/O
* signal-driven I/O
* asynchronous I/O

### 4.1 blocking I/O

![alt text](blocking_io.png)

Blocking means the process will not use any CPU resource while waiting, which means process will do nothing.

By default, all sockets are blocking.  It's simple but usually not efficient. For example, if there are 10k connections, one connection trying to get data, others need to be waiting and not been processed at all.

### 4.2 non-blocking I/O

![alt text](non_blocking_io.png)

In this type of I/O, when a process's request can not be fullfilled (wait for the data), os will not put the process to sleep; instead it returns an error (**EAGAIN** or **EWOULDBLOCK**); at the mean time, process will keep on calling `recvfrom` until a `OK` returned.

**polling**: When an application sits in a loop calling recvfrom on a nonblocking descriptor like this, it is called polling.


Comparing to blocking I/O:

**Pros**: while waiting for the data, process is not blocked, so process can do other stuff.

**Cons**: about the timing of `datagram ready`, it could happend at any time between two `redvfrom` calls, so it may low the throughput the system can handle. Also, polling can be quite CPU consuming, because of the repeatedly checking. 

It looks like:

```cpp
while (true) {
	non-blocking read fd1;
    non-blocking read fd2;
    other ops;
    ...
}
```


### 4.3 multiplexing I/O

Let's see a scenario: 

a web server handling multiple connections, let say 10k (**but still single process and sinlge thread**). If we use `non-blocking I/O` model, then all those 10k connections will repeatedly checking with kernal to see if data is ready, and it will be quite cpu consuming. Basically, in multiplexing model, let say there are multiple connections, which means multiple FDs (file descriptors), and this sinlge process will be blocked while let kernal polling all sockets. Once one of the socket got the data, it then inform the process to know that one of the data is ready and process continues to run. 

![alt text](multiplexing_io.png)

I/O multiplexing is to monitor multiple FDs in a single process. Comparing to `non-blocking I/O`, it let a `third party` to do the polling instead of the process itself, and once any of the I/O FDs (for example a socket) is ready (readable), it will wake the process to handle.

It looks like this:

```cpp
while (true) {
	selected_result = select(fds);
    for (auto fd: selected_result) deal with fd;
}
```

There are 3 system calls to implement multiplexing:

**select()**

First, we need to know that for every FD, there is a queue (`struct wait_queue_head_t`) to store all processes which monitoring this FD. There is a callback for every FD, and what this callback do is to add new monitor to the `wait_queue_head_t`.

What `Select()` does is to call every callback for every FD to be monitored. So when one of the FD is ready, this FD will wake up all processes in the `wait_queue_head_t`.

Process does not know who wakes it up, and process itself will check wich FDs are ready.

**poll()**

`poll()` is similar to `select()`. To use select, there is a `__FD_SETSIZE` to define the maximum number of FDs. `poll()` removes this limits.

**epoll()**

Problems with `select() and poll()`

* every `select()` call, will copy FDs from user space to kernal space
* every `select()`, the kernal must check all the FDs to see if it's ready
* after calling `select()`, the program must inspect every element of the returned data structure to see which FDs are ready.

`epoll()` 



