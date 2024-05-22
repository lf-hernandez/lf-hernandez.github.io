---
title: "Making sense of Node.js internals: a first pass"
date: 2024-05-25T23:08:57-04:00
tags: ["nodejs"]
categories: ["nodejs"]
---
In this post, I attempt to peel back one of the layers that make up Node.js. I plan to continue my deep dive in future posts, so this is not meant to be exhaustive, but I hope to lay out some of the underlying principles that contribute to Node's non-blocking nature.

Which leads into my first question: what exactly does non-blocking mean? To examine, I'll first visit some basic operating systems theory concepts.

## Input/Output Operations

Computers primarily carry out two functions: I/O and processing. It's reasonable to view I/O as the primary function that drives our use of computers. Whether it's reading files, sending email, streaming music, responding to mouse and keyboard events, or displaying the very graphical user interface we see on our monitors, computers are constantly managing I/O. As programmers, the operating system allows us to determine how we wish to access a given I/O resource via two mechanisms: Blocking and Non-blocking I/O.

### Blocking I/O

This mode of operation can be illustrated by the following example: Suppose there's an application that needs to read from a file and process its contents among other tasks. The application issues a request to the operating system via a system call to retrieve the file from disk, causing the flow of execution to halt while it cedes control to the operating system to retrieve the data. The application moves from a running state to a waiting state, meaning it cannot process further requests until the system call completes. Once it does, control returns to the application, and it resumes execution. This mode of operations is easier to reason about from a programming perspective, as it is predictable but comes at a performance cost, namely, idle CPU time while the I/O request is completed.

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#ifndef BUF_SIZE
#define BUF_SIZE 256
#endif

void readF(const char *file) {
    FILE *f = fopen(file, "r");
    if (f == NULL) {
        perror("Error opening file");
        exit(EXIT_FAILURE);
    }

    char buffer[BUF_SIZE];
    ssize_t bytesRead;
    clock_t start, end;
    double idleTime = 0.0;

    while (1) {
        start = clock();
        bytesRead = fread(buffer, 1, BUF_SIZE, f);
        end = clock();
        idleTime += ((double) (end - start)) / CLOCKS_PER_SEC;

        if (bytesRead > 0) {
            fwrite(buffer, 1, bytesRead, stdout);
        } else {
            break;
        }
    }

    if (ferror(f)) {
        printF("Error reading file");
    }

    fclose(f);
    printf("\nTotal idle time: %f seconds\n", idleTime);
}

int main(int argc, char *argv[]) {
    readF(argv[1]);
    printf("\nThis prints after the file is read");

    return 0;
}
```

Output:

```bash
$ ./blockig_io.o big.txt
The Adventures of Sherlock Holmes
by Sir Arthur Conan Doyle
...
Total idle time: 0.020595 seconds
This prints after the file is read
```

Functions like `fopen`, `fread`, and `flcose` block the flow of execution until the I/O operation is completed.
I'm leveraging the `time.h` header to help simulate the amount of time the CPU is idle waiting for the I/O operation to finish. The calculation is done by simply diving the delta of end/start times by a predefined constant that represents the number of clock cycles per second. My specific system is set to `1000000` (1 million) ticks per second.

### Non-blocking I/O

Non-blocking I/O on the other hand allows for more efficient use of resources at the cost of increased programming complexity. With non-blocking operations, the application can issue the I/O operation and continue processing without ceding control of its execution state. This is because the operating system call returns immediately regardless of whether the resource is ready or not. Once the operation completes, the application is notified and can handle the result.

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <sys/select.h>
#include <time.h>

#ifndef BUF_SIZE
#define BUF_SIZE 256
#endif

void readF(const char *file)
{
    int fd = open(file, O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        perror("Error opening file");
        exit(EXIT_FAILURE);
    }

    char buffer[BUF_SIZE];
    ssize_t bytesRead;
    clock_t start, end;
    double idleTime = 0.0;

    fd_set readfds;
    struct timeval timeout;
    int retval;

    while (1) {
        FD_ZERO(&readfds);
        FD_SET(fd, &readfds);

        timeout.tv_sec = 1;

        start = clock();
        retval = select(fd + 1, &readfds, NULL, NULL, &timeout);
        end = clock();
        idleTime += ((double)(end - start)) / CLOCKS_PER_SEC;

        if (retval == -1) {
            perror("select() error");
            break;
        } else if (retval) {
            bytesRead = read(fd, buffer, BUF_SIZE);
            if (bytesRead == -1) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    continue;
                } else {
                    perror("Error reading file");
                    break;
                }
            } else if (bytesRead == 0) {
                break;
            } else {
                fwrite(buffer, 1, bytesRead, stdout);
            }
        } else {
            printf("No data within 0.5 seconds.\n");
        }
    }

    close(fd);
    printf("\nTotal idle time: %f seconds\n", idleTime);
}

int main(int argc, char *argv[])
{
    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (pid == 0) {
        readF(argv[1]);
    } else {
        printf("his prints after the file is read\n");
    }

    return 0;
}
```

Output:

```bash
$ ./non_blocking_io.out big.txt 
This prints before the file is read
The Adventures of Sherlock Holmes
by Sir Arthur Conan Doyle
...
Total idle time: 0.035097 seconds
```

The gist of this approach is that we change how the file is opened and read by using non-blocking I/O, specifically, using lower-level system calls `open`, `read`, `close` and `select` as well as making use of threads (fs is one of the few* resources that Node.js uses a thread pool for).

In Unix (and Linux), the OS provides lower-level (think kernel space) functionality to be accessed in user space via system calls.  Since we're dealing with a lower-level interface, you'll notice the lack of a file pointer. This is because in Unix everything is a file, and the OS utilizes a homogeneous interface to deal with files, namely, file descriptors. You may already familiar with 3 standard ones often used in the shell, 0 (standard input), 1 (standard output), 2 (standadrd error). e.g. `$ runSomeCmdWithError 1> output.txt 2>&1`. In this case, the file descriptor is the value that will be associated with a given file, such that any reference to it in other system calls can be made via this integer.

What is a file descriptor?
> All system calls for performing I/O refer to open files using a file descriptor, a (usually
small) nonnegative integer.
> File descriptors are used to refer to all types of open files, including pipes, FIFOs, sockets, terminals, devices, and regular files.
>
> The Linux Programming Interface

How does open() function?
> fd = open(pathname, flags, mode) opens the file identified by pathname, returning
a file descriptor used to refer to the open file in subsequent calls.
>
> The Linux Programming Interface

O_NONBLOCK flag?
> O_NONBLOCK
    When possible, the file is opened in nonblocking mode.
    Neither the open() nor any subsequent I/O operations on
    the file descriptor which is returned will cause the
    calling process to wait.
>
> open() man page

read()?
> numread = read(fd, buffer, count) reads at most count bytes from the open file
referred to by fd and stores them in buffer. The read() call returns the number of
bytes actually read. If no further bytes could be read (i.e., end-of-file was
encountered), read() returns 0.
>
> The Linux Programming Interface

close()?
> The close() system call closes an open file descriptor, freeing it for subsequent reuse
by the process. When a process terminates, all of its open file descriptors are automatically
closed.
>
> The Linux Programming Interface

And select()?
> select() allows a program to monitor multiple file descriptors,
    waiting until one or more of the file descriptors become "ready"
    for some class of I/O operation (e.g., input possible).  A file
    descriptor is considered ready if it is possible to perform a
    corresponding I/O operation (e.g., read(2), or a sufficiently
    small write(2)) without blocking.
>
> WARNING: select() can monitor only file descriptors numbers that
       are less than FD_SETSIZE (1024)—an unreasonably low limit for
       many modern applications—and this limitation will not change.
       All modern applications should instead use poll(2) or epoll(7),
       which do not suffer this limitation.
>
> select() man page

The program uses `fork` to create a child process that runs `readF` concurrently. Although, `fork` is not necessary to demonstrate non-blocking I/O. The key part of the approach is `select`, which allows the program to read from the file only when it is ready, avoiding blocking on I/O operations.

## How does Node.js approach non-blocking I/O?

### Event Demultiplexing

When handling non-blocking I/O operations on a single thread, the aim is to switch between tasks efficiently, reducing idle time. In Event Demultiplexing, resources are placed into a data structure and watched with their associated operations. When an operation completes, the watch creates an event, allowing the program to handle it, only blocking until new events are ready to be processed. This approach prevents CPU cycle wastage. This is preferred over other approaches like busy-waiting whereby the resource is continuously polled until it's ready. This technique is part of the basis that libuv and by extension Node.js builds upon to handle non-blocking I/O.

### Unicorn Velociraptor and the reactor pattern

Libuv ia a core component of Node.js. It implements the reactor pattern which provides an interface to create and manage event loops. It abstracts away the low-level system calls (similar to what was covered in the previous section with `select`) that are responsible for non-blocking I/O across varying platforms.

> The reactor software design pattern is an event handling strategy that can respond to many potential service requests concurrently. The pattern's key component is an event loop, running in a single thread or process, which demultiplexes incoming requests and dispatches them to the correct request handler.
>
> [Wikipedia](<https://en.wikipedia.org/wiki/Reactor_pattern>)

 When an I/O operation request is issued in Node, an associated handler (via callbacks) is also passed. The event demultiplexer then stores a structure with the resource, operation, and handler. Libuv manages the operations using system calls like `epoll` (Linux), `kqueue` (BSD/macOS), or `IOCP` (Windows). Once a worker completes its task, an event is generated and passed along with the registered callback to the event queue. The event loop processes them, passing control to the handler on the main thread.

### To be continued

I have lot more to learn and cover but these initial concepts will prove to be a useful base to understand the complexities the make up libuv and the event loop in a future post.
