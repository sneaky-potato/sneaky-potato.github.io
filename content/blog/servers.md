---
title: "Sockets, servers and scalability"
date: 2025-10-25
description: "linux, networks, sockets"
tags: ["tech"]
math: true
---

# Scaling a server to handle many requests

> Everyone can talk, listening is where the actual difficulty lies.

Caution: If the word _**socket**_ does not ring any bells in your mind, this article will be difficult for you to follow.

## Bare bones server

At the time of writing this article, I am employed as a fullstack software engineer and a lot of my work intersects with writing backend microservices.
Microservices that scale for traffic. I primarily use [Go](https://go.dev/) to write these backend APIs.
Often times, I think about how backend servers are implemented internally and I decided to research a bit on the topic and that is the discussion of this very article.
So if you are someone who has written any form of backend APIs, web servers or network programming, this article is going to be perfect for you.

Let us start our discussion using first princple thinking, which I like to apply to any new problem which is thrown at me.
So, what are web servers actually? Having done my networks course in C, I would have answered this question with one word: ***socket***.

### Socket

Sockets are abstractions of connections. For simplicity you can think of a socket as a tuple of size 5 containing the following information:

`(src IP, src port, dst IP, dst port, protocol)`

where `src IP` = source IP address, `src port` = source port, `dst IP` = destination IP address, `dst port` = destination port and `protocol` is either TCP or UDP.
So a socket in essence is a connection using port and protocol. This leads us to an important conclusion.

Therefore, you could open two sockets, a TCP and a UDP on the same port, StackOverflow answer [here](https://stackoverflow.com/questions/6437383/can-tcp-and-udp-sockets-use-the-same-port)

### How to open a socket

Let's look at all the steps involved while opening a socket for serving an incoming request.
```c
int server_fd;
struct sockaddr_in addr;

server_fd = socket(AF_INET, SOCK_STREAM, 0); // SOCK_STREAM = TCP
if (server_fd < 0) { perror("socket"); exit(1); }

addr.sin_family = AF_INET;
addr.sin_addr.s_addr = INADDR_ANY;
addr.sin_port = htons(9000);

if (bind(server_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
    perror("bind"); exit(1);
}

// 5 connection requests will be queued before further requests are refused.
if (listen(server_fd, 5) < 0) { perror("listen"); exit(1); }
printf("Listening on port 9000...\n");
```

This will open a listening TCP socket on port 9000. But we cannot connect to it yet, for that we need to add the logic when a client connects to this socket.

```c
char buf[1024];

while (1) {
    // accept a connection
    int client_fd = accept(server_fd, NULL, NULL);
    if (client_fd < 0) { perror("accept"); continue; }

    printf("Client connected.\n");

    while (1) {
        // read whatever client writes to the socket
        // this is a blocking call
        ssize_t n = read(client_fd, buf, sizeof(buf));

        if (n <= 0) break; // client closed or error
        // echo back whatever client writes
        write(client_fd, buf, n);
    }

    printf("Client disconnected.\n");
    close(client_fd);
}

close(server_fd);
```

I want the readers to look into the `read()` system call, it is a blocking call. Which means the program will not proceed ahead until it has read
whatever content client wants to send. And this is only natural: **You have to wait and read what the client wants to say**.

This leads us to an important conclusion: this code can only handle **one client request** at a time, which can be verified using netcat.
If a request arrives when this server is serving an already existing connection, it has to wait.

This is not very good, let us improve this design.

## Forked server

To enable the process to serve more than one client, we could replicate the process altogether using `fork()` system call, right?
**Yes** we can do that, let us try this approach. Update the main loop as follows:

```c
char buf[1024];

while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
    if (client_fd < 0) { perror("accept"); continue; }

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        close(client_fd);
        continue;
    }

    if (pid == 0) {
        // Child process
        close(server_fd); // child doesn't need the listening socket
        printf("[Child %d] Client connected.\n", getpid());

        while (1) {
            ssize_t n = read(client_fd, buf, sizeof(buf));
            if (n <= 0) break; // client closed or error
            write(client_fd, buf, n); // echo back
        }

        printf("[Child %d] Client disconnected.\n", getpid());
        close(client_fd);
        exit(0);
    } else {
        // Parent process
        close(client_fd); // parent doesn’t handle this connection
    }
}
```

Now the process makes a copy of itself to serve an incoming request, this lets the server cater to many concurrent requests. But how many?

### Performance

Compile the forked server and run it
```bash
gcc -o forked_server forked_server.c
./forked_server
```

Inside another terminal connect to it using netcat (make 3 different connections)
```bash
nc localhost 9000
```
Do this 2 more times in different terminals.

Measure the memory usage using ps
```bash
ps -o pid,rss,cmd -C forked_server
```

This will give output somthing similar to this
```
  PID   RSS CMD
49006  1568 ./forked_server
49244  1276 ./forked_server
49513  1276 ./forked_server
```

So this means, the 3 connections have peak memory usage as 1568KB (1.5MB), 1276KB (1.2MB) and 1276KB (1.2MB) respectively.

This approach works fine for small loads, but hits limits fast when scaling up. Replicating a process comes with memory overhead, also the OS has limited process ids.

| Limit | Explanation | Symptom |
| ----- | ----------- | ------- |
| **Memory per process** | Each process has its own address space (~1–2 MB minimum) | Memory spikes with 1000+ clients |
| **Context switch** | Kernel must constantly switch between thousands of processes | High CPU, slow response |
| **PID** | Linux has finite number of process IDs | fork: Resource temporarily unavailable |
| **File descriptor** | Each process inherits its own FD table | "Too many open files" errors |

The conclusion that we can draw from here is: **this approach scales linearly until OS overhead takes over, then collapses**.

If processes are expensive, what about threads? Aren't those light weight processes?

## Threads

Surely, we can make the program better by multithreaded each request?
Update the main loop like so.

```c
while (1) {
    int *client_fd = malloc(sizeof(int));
    if (!client_fd) continue;

    *client_fd = accept(server_fd, NULL, NULL);
    if (*client_fd < 0) { perror("accept"); free(client_fd); continue; }

    pthread_t tid;
    if (pthread_create(&tid, NULL, handle_client, client_fd) != 0) {
        perror("pthread_create");
        close(*client_fd);
        free(client_fd);
        continue;
    }

    pthread_detach(tid); // don't need to join later
}

close(server_fd);
```

And define the handler function for the thread like so:
```c
void *handle_client(void *arg) {
    int client_fd = *(int*)arg;
    free(arg);

    char buf[1024];
    printf("[Thread %ld] Client connected.\n", pthread_self());

    while (1) {
        ssize_t n = read(client_fd, buf, sizeof(buf));
        if (n <= 0) break;
        write(client_fd, buf, n);
    }

    printf("[Thread %ld] Client disconnected.\n", pthread_self());
    close(client_fd);
    return NULL;
}
```

### Performance

Threads are better than the previous approach, but they also fail to serve beyond 1k concurrent requests.
Each thread consumes 1MB stack + scheduler state.

| Limitation                        | Explanation                                          | Symptom                                       |
| --------------------------------- | ---------------------------------------------------- | --------------------------------------------- |
| **Memory per thread**             | 10k threads × 1MB ≈ 10GB just for stacks             | Out of memory                                 |
| **Context switch**                | Thousands of runnable threads cause kernel thrashing | CPU usage spikes to 100% even when idle       |
| **File descriptor**               | Each socket = 1 FD; OS default often 1024            | `accept: Too many open files`                 |
| **Synchronization**               | If you share data between threads                    | Lock contention, latency                      |

What is the next step, how can we make this any better, how can we scale this simple echo server to 10k requests?

## Epoll

The problem is `read()` system call, which is blocking in nature. We have to wait for the client to write to the file descriptor and then only we can move ahead.

So the problem boils down to the synchronouse nature of listening: you wait and listen what the talking party has to say. But what about when the talking party
is not saying anything at the moment? Surely you can utilize that time to listen to other talking party who want to say something right?

This mean, there is a need of event driven architecture. Something that bring asynchonity to the model.
How about an event queue, if some client writes to a socket, we push an event and tell our program to read. This way we will only read when the client writes.

This can be done by the `epoll()` system call.

Imagine a queue of I/O readiness events. You tell the kernel: **Please watch these sockets and wake me when they have something to do.**

Then the kernel keeps track of all those sockets for you.

There are three main system calls:
| Call              | Purpose                                            | Analogy                                        |
| ----------------- | -------------------------------------------------- | ---------------------------------------------- |
| `epoll_create1()` | Create a new epoll instance (kernel event manager) | Create an empty watch list                     |
| `epoll_ctl()`     | Add / remove / modify which FDs you want to watch  | Add items to your interest list                |
| `epoll_wait()`    | Block until one or more watched FDs become ready   | Wait for events to appear in the ready queue   |

1. Create an epoll instance
```c
int epfd = epoll_create1(0);
```
This gives you a handle to an internal epoll object in the kernel.

2. Register sockets to watch
```c
struct epoll_event ev;
ev.events = EPOLLIN;    // interested in read
ev.data.fd = sock;
epoll_ctl(epfd, EPOLL_CTL_ADD, sock, &ev);
```

This tranlates to a request to kernel
> When data arrives on socket sock, add an event for it to my internal ready list

3. Wait for events
```c
struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
```
This call sleeps until any socket you registered becomes ready.
When something happens, `epoll_wait()` returns with up to `MAX_EVENTS` ready sockets.

You then loop through:
```c
for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;
    // read or write as needed
}
```

epoll is the better alternative to `poll` and `select` system calls.
However epoll’s complexity is roughly O(1) per ready FD, while select()/poll() are O(N) per call.

# Why It Scales?

- One kernel data structure holds all your FDs.
- Wake-ups happen only when necessary.
- No linear scanning through all sockets like in `select()`or `poll()`.

That’s why epoll is the engine behind:

- nginx
- Redis
- Node.js
- Go runtime’s netpoller

Following table summarises the whole discussion

| Model     | Memory per conn | Kernel objects | Max connections | Typical use                                             |
| --------- | --------------- | -------------- | --------------- | ------------------------------------------------------- |
| `fork()`  | ~1–2 MB         | process + fd   | 500–1k          | legacy UNIX daemons                                     |
| `pthread` | ~1 MB           | thread + fd    | 1k–5k           | simple chat/web servers                                 |
| `epoll()` | ~few KB         | fd only        | 100k+           | modern high-performance servers (nginx, Go, Rust, etc.) |
