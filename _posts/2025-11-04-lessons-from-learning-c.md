## Lessons from Learning C

Starting out on Code Warriors, I needed a more interesting project to sink my teeth into and decided to build an electricity monitor. This involved writing a server in C to handle communications with a device I built that could measure power, current, voltage and frequency draw from any 240V appliance. It also served web requests from a website I build to control the monitoring device.

This meant it needed to handle both websocket and HTTP protocols and simultaneous connections. I learned about how computers handle multiple processes, multi-threading, WS protocol and various C paradigms. Now, a couple of months on, I've decided to write a post about the things I learned in the hope that it helps the lessons stick.

Knowing that I'd never touch this program again, towards the end I had no incentive to write really high quality code and everything started deteriorating - I put in the minimum amount of effort required to get it working. So this will be about the more general concepts and principles I learned, rather than anything specific about my raggedy project.

**They generally get more technical and more interesting down the list.**

### Building a Router

There's not much to say about this because it's fairly simple, but a rewarding exercise to do and realise just how simple they are.

### Pass-by-Reference VS Pass-by-Value

All functions in C are pass-by-value. When passing by value, changes to the value do not persist beyond the scope of the function that operates on the value, because the function creates a copy of the variable which is cleaned up when the function returns. You can get around this by passing a pointer, which is still pass-by-value, but you're passing the value of the address where some data is stored, and which persists beyond the scope of the function.

This is relevant when constructing sockets. You pass a pointer like `struct sockaddr *` to a function which modifies the values referenced by the pointer. This updated struct then persists beyond the scope of the function that modifies them.

You can pass a struct by value into a function but this has performance costs so pass-by-reference is often preferred.

```C
struct some_struct_t some_struct, another_struct;
set_struct_props_with_pointer(&some_struct);
set_struct_props_with_value(another_struct);

printf("the value at some_struct.a_value is: %s\n", some_struct.a_value); // value is accessible here
printf("the value at another_struct.a_value is: %s\n", another_struct.a_value); // another_struct.a_value is NULL, which is undefined behaviour
```

### Error Code Pattern

C cannot return multiple values so the common way to indicate the success or failure of a function is to return an error code, or 0 for success. You _could_ return a struct which contains a `success` boolean but this would pollute the data model so this is not commonly done. Also, returning a struct incurs a performance overhead, while returning an int is a register operation, which is much faster.

The example below shows how this pattern is used.

```C
if (bind(sock_fd, server_info->ai_addr, server_info->ai_addrlen) < 0) {
    // `bind()` returns 0 if successful. If the return value is less than 0, this represents
    // an error which can be handled.
    perror("bind");
    exit(1);
}
```

### File Descriptors

File descriptors (fd) are used throughout Unix-like systems because of the philosophy that "everything is a file"—sockets, pipes, regular files, and devices are all accessed through the same interface. A file descriptor is a positive integer that serves as an abstract handle to an open I/O resource.

File descriptors are unique to each process. Every process automatically starts with three standard file descriptors:

- **0 (STDIN_FILENO)**: Standard Input, typically the keyboard
- **1 (STDOUT_FILENO)**: Standard Output, typically the console/terminal
- **2 (STDERR_FILENO)**: Standard Error, also typically the console/terminal, used for error messages

When you open a file, create a socket, or establish a pipe, the system assigns the next available file descriptor number (3, 4, 5, etc.) in sequential order. This abstraction is why functions like `read()`, `write()`, and `close()` work the same way whether you're dealing with a file, socket, or pipe.

### Websockets (NOT Sockets)

Mozilla's resource on the matter is excellent and contains almost everything you need to know about the protocol: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers.

#### Establishing a Connection: Handshake

A websocket is a connection protocol that allows two devices to continuously send and receive data without requiring a request/response pattern after the initial connection is established. Both can send a message at any time.

A connection is established using normal HTTP requests via a "websocket handshake". The handshake negotiates the details of the connection:

1. **Client sends a connect request** containing a randomly generated `Sec-WebSocket-Key` header:
```HTTP
GET /chat HTTP/1.1
Host: example.com:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

2. **Server responds** with an HTTP response containing a `Sec-WebSocket-Accept` header:
```HTTP
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

3. **Connection established** - both sides can now send and receive websocket payloads.

The `Sec-WebSocket-Accept` header is calculated by concatenating the magic string `"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"` to the client's `Sec-WebSocket-Key`, taking the SHA-1 hash of that, and returning the base64 encoding of the hash. 

The `Sec-WebSocket-Accept` calculation allows the client to ensure that the server actually supports the websocket protocol. It doesn't provide any security, since the magic string is universally known.

#### Websocket Frame Structure

Binary data is sent. Messages from client to server are always masked with XOR using a 32-bit masking key. Messages from server to client do not have to be masked. The below diagram shows the meaning of each bit.

```
Data frame from the client to server (masked) for a message of length <= 125 bytes

 0                   1                   2                   3                   4                   5                  "n"   
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6  ...  0 1  
+-+-+-+-+-------+-+-------------+---------------------------------------------------------------+-----------------  ...  ----|
|F|R|R|R| opcode|M| Payload len |                         Masking key                           |    Payload Data            |
|I|S|S|S|  (4)  |A|     (7)     |                             (32)                              |                            |
|N|V|V|V|       |S|             |                                                               |                            |
| |1|2|3|       |K|             |                                                               |                            |
+-----------------------------------------------------------------------------------------------------------------  ...  ----|
```

- FIN: 1 indicates that this is the end of a multi-frame payload, 0 indicates there is more to come
- Bits 1 - 3 are reserved for future expansion.
- opcode allows you to specify the data type in the payload
- Mask: this bit indicates whether the payload is masked.
- Payload len specifies the length of the payload in bytes.
    If 0 - 125 (inclusive), this is the length of the payload
    If 126, the next 16 bits should be read to get the payload length
    If 127, the next 64 bits should be read to get the payload length.
    In this way, larger payloads can be sent without requiring smaller payloads to waste 64 bits on specifying the payload length.

#### Decoding the Payload

Step through the bytes of the encoded payload one at a time and XOR each byte with the (i modulo 4)th octet of MASK. So, looping through the 4 byte mask.

And since DATA ^ MASK ^ MASK = DATA, you can use the same function to mask and unmask a payload. Here's how I wrote it:

```C
int mask_unmask_payload(ws_frame_t *frame) {
    // frame->payload already stores the masked payload
  if (!frame->payload || !frame->maybe_mask) {
    return -1;
  }

  size_t actual_payload_len = frame->extended_payload_len
                                  ? frame->extended_payload_len
                                  : frame->payload_len;

  for (size_t i = 0; i < actual_payload_len; i++) {
    // bit-shifting to get the (i modulo 4)th to the least significant byte position then
    // masking it off with 0xFF (equals 11111111) to get ONLY that byte from the mask
    unsigned char mask_byte = (frame->maybe_mask >> (8 * (3 - i % 4))) & 0xFF;
    // XOR the ith byte of the payload with the relevant byte from the mask
    frame->payload[i] ^= mask_byte;
  }
  return 0;
}
```

### Sockets (NOT Websockets)

Beej's guide is the only reason I succeeded in this project. All thanks goes to whoever put that incredible resource together: https://beej.us/guide/bgnet/html/index-wide.html

A TCP socket, identified by an fd, is a bidirectional communication endpoint that establishes a network connection.

#### Creating a Socket (server)

The below is a significantly simplified explanation...

First, set up a `struct addrinfo *server_info` using `getaddrinfo()`.

Then call `socket()`, passing in parameters extracted from the `addrinfo` struct:
- **ai_family:** specifies either IPv4 or IPv6
- **ai_socktype:** specifies either `SOCK_STREAM` or `SOCK_DGRAM`. `SOCK_STREAM` creates a TCP socket, a protocol which guarantees delivery order and facilitates 2-way communication. It is used for HTTP, HTTPS, WebSocket and SSH. `SOCK_DGRAM` creates a UDP socket, which does not guarantee delivery or delivery order. It is used where speed matters, eg: for video streaming and gaming, among other things.
- **ai_protocol:** usually set to 0, which sets the default protocol based on the ai_socktype selected, but can be explicitly set.

Calling `bind()` associates the fd of the socket created with `socket()`, to the network address/port from the `addrinfo` struct

Calling `listen()` marks the socket as listening and sets the backlog queue size. Internally, the kernel handles incoming connections.

Now we have a socket that can receive connections and we start an infinite loop that will handle incoming connections:

```C
while (1) {
    if ((new_fd = accept(sock_fd, (struct sockaddr *)&client_addr, &addr_size)) <
        0)
    {
      perror("accept");
      exit(1);
    }
    
    // handle the connection
}
```

`accept()` is a blocking call, so nothing will happen until a connection is made. When a connection arrives, it returns a new file descriptor (`new_fd`) representing the client connection, which is different from the listening socket (`sock_fd`).

### Processes

A single program can have one or more processes, each consisting of one or more threads.

A process represents some code being run, performing some action. Each process has its own memory space (its own stack, heap, data segment etc), as opposed to threads, which share memory. The OS assigns each process a unique PID (Process ID).

Processes cannot access each other's memory, however this can be achieved through various inter-process communication (IPC) techniques.

#### `fork()`

Any process can create a child process by calling `fork()`. This creates an almost exact duplicate of the parent, with the same code and the same open fds, which are duplicated, not shared. `fork()` returns 0 to the child process, and a positive value (the PID) to the parent, so that both processes know which process they are.

The parent can call `wait()` or `waitpid()` to wait for the child process to finish executing (by calling `exit()`) before proceeding. If the child process exits but the parent didn't call `wait()` to read its exit status, the child becomes a zombie process, which means that while it doesn't consume memory or compute, it still occupies a row in the process table maintained by the kernel.

Zombie processes can also be cleaned up asynchronously with a SIGCHLD handler.

If zombie processes accumulate, this can be a problem because it clutters the process table.

### Inter-process Communication (IPC)

As a bit of background, each time a connection was made, a new `fork()`ed process would be created to serve the connection. A websocket connection resulted in a long-lived process, and an HTTP connection was ephemeral. My monitoring device would connect first, establish a websocket connection in its forked process, then an HTTP request (served on a different process) would need to be able to tell it to start recording. Inter-process communication was therefore required to trigger a websocket request from the server to the device.

#### Pipes

A pipe allows IPC by establishing a one-way communication channel, two pipes must be created to facilitate two-way communication. A pipe is created with a system call, which returns an array of two fds. It is managed by the OS as a FIFO queue. The default size of a pipe buffer is 64kB but the size can be specified up to the system's maximum.

Note: the "|" (pipe) operator in bash uses the C `pipe()` system call under the hood. for `command1 | command2` it would go as follows:
- Create a pipe
- Call `fork()` twice to create two child processes
- In process 1, close the read end of the pipe and redirect stdout to the write end of the pipe using `dup2()`
- In process 2, close the write end of the pipe and redirect stdin to the read end of the pipe using `dup2()`
- Execute both commands. `command2` will block until `command1` writes to the pipe.

Creating a pipe:
```C
int fd[2];
if (pipe(fd) != 0) {
    perror("pipe");
}
// fd[0] is the read end of the pipe
// fd[1] is the write end of the pipe

if (!fork()) {
    // this is the child process. We want to close the read end of the pipe
    close(fd[0]);
} else {
    // this is the parent process. We want to close the write end of the pipe
    close(fd[1]);
}
```

The fds can now be used in the same way as for normal files. Reading and writing are blocking operations. If a process tries to read from an empty pipe, it will block until data is written to it, and if a process tries to write to a full pipe, it will block until it can write.

If the write end is closed, the next `read()` from the read end will receive an EOF (0), indicating to that process that it can close its fd.

#### FIFO Queues (Named Pipes)

Unlike normal pipes, which exist only as long as the process that created them is running, named pipes persist beyond the process that created them. They exist as files, allowing multiple processes to read and write to them.

A named pipe is created with `mkfifo()`:

```C
int mkfifo(const char *pathname, mode_t mode);
```

`mode` sets the permissions as with a normal file.

Unlike an anonymous pipe, read/write fds are not created upon creation of the pipe, so any process that wants to use the pipe must call `open()`.

#### Shared memory

Shared memory is a simple way to implement two-way IPC. Though it would not have scaled, it was the easiest approach for my project since I could create a basic struct that both processes could read and modify.

Calling `mmap()` creates a memory mapping in the process's address space and returns a pointer to that memory. With `MAP_ANONYMOUS`, it creates anonymous memory (not backed by a file) that can be shared between processes. When used with a file descriptor, it maps the file into memory, allowing direct access to file content as if it were in memory, avoiding the need for read/write system calls.

When `mmap()` is called with `MAP_SHARED`, memory is accessible by different processes such that a modification to the variable is immediately visible to all processes sharing the mapping. When called with `MAP_PRIVATE`, the memory is copied when a new process writes to it such that changes are not visible to other processes.

With `MAP_SHARED`, in order to avoid issues around concurrently reading/writing the same memory, access must be synchronised, for example, with a mutex or semaphore, which are discussed in the next section.

**Example implementation**

Declare the struct variable in ipc.h as `extern` so that it can be accessed in multiple files.
```C
extern struct ws_ipc_t *shared_mem;
```

Initialise it somewhere and create the mapping.
```C
struct ws_ipc_t *shared_mem = NULL;

shared_mem = mmap(NULL, sizeof(struct ws_ipc_t), PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
if (shared_mem == MAP_FAILED) {
    perror("mmap failed");
    exit(1);
}
```

`shared_mem` can now be accessed in any child process forked from a process in which the `shared_mem` map exists. When using `MAP_SHARED`, changes made by any process are visible to all processes sharing the memory. The memory is automatically cleaned up when all processes using it exit.

### Synchronisation Between Shared Resources

Mutexes and semaphores are examples of constructs that prevent race conditions wherein multiple threads/processes might simultaneously access a resource.

#### Mutexes 

A mutex, short for mutual exclusion, locks a resource so that only one thread/process can read/write it at a time. There are a few ways to implement this, but conceptually, when a resource is accessed, its mutex is locked and when the process is finished, the mutex is unlocked, signalling to other processes that try to access the resource that it is available.

### Having to Program Defensively

I don't enjoy the verbosity of having to check for NULL pointers all over the place and calling so many functions inside an `if` statement to check its output. This is necessary in C because unlike other languages, it does not have a concept of an exception. Not doing so risks segfaults, NULL references and undefined behaviour. Common examples include:

NULL pointer checks before dereferencing

```C
char *buffer = malloc(1024);
if (buffer == NULL) {
    perror("malloc");
    exit(1);
}
// Now safe to use buffer
```
Bounds checking for arrays/buffers

```C
if (index >= 0 && index < buffer_size) {
    buffer[index] = 'a';
} else {
    return -1;
}
```
Return value checking for system calls

```C
if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
    perror("socket");
    exit(1);
}
```

### Comparison of C and JS

#### Similarities

Objects in JS are passed by "value of reference", which is similar to passing a pointer in C. This means if a function modifies an object's properties, the modifications are visible outside the function scope.

#### Differences

**Function declarations:**
- **C**: Functions must be declared or defined before use.
- **JavaScript**: Function declarations are "hoisted" so they can be called before they appear in the code. Function expressions (like `const f = function() {}`) are not hoisted.

**Memory management:**
- **C**: Manual memory management with `malloc()`/`free()`. You must explicitly allocate and deallocate memory.
- **JavaScript**: Automatic garbage collection.

**Error handling:**
- **C**: Functions return error codes (typically 0 for success, negative for errors). You must explicitly check return values.
- **JavaScript**: Uses exceptions with `try/catch` blocks. Errors can be thrown and caught at any level.

**Array bounds:**
- **C**: Accessing out-of-bounds array indices causes undefined behavior or segmentation faults.
- **JavaScript**: Accessing out-of-bounds array indices returns `undefined` instead of crashing.

**Variable initialization:**
- **C**: Uninitialized variables contain garbage values. You must explicitly initialize them.
- **JavaScript**: Variables are automatically initialized to `undefined`.
