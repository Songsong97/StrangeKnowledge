# Operating System
Some notes for operating systems and cpp programming in Unix.

## Table of contents
1. [Memory Leak](#Chapter1)
    1. [Still reachable memory](#Chapter1.1)
    2. [Possibly lost memory](#Chapter1.2)
2. [Inter-process communication](#Chapter2)
    1. [Communicate between processes using pipe](#Chapter2.1)
    2. [Communicate between threads using pipe](#Chapter2.2)
    3. [Socket and SIGPIPE](#Chapter2.3)
3. [Some commonly used commands in Linux](#Chapter3)
    1. [g++/gdb](#Chapter3.1)
    2. [valgrind](#Chapter3.2)

<a name="Chapter1"></a>
## Memory Leak
This chapter contains some common causes for memory in unix programming and may give you some hints on troubleshooting.

<a name="Chapter1.1"></a>
### Still reachable memory
This kind of memory leak is not causing fatal problems, since the process will exit explicitly and the memory allocated will be reclaimed by the operating system. But it is good practice to free the memory and graphics people should be fastidious!

If you run the code below with ```valgrind ./test``` it will tell you there is still a reachable memory block. The reason is that when we call exit() the call stack will not jump out of helper and the memory will not be freed since delete is unreachable.

```cpp
// test.cpp
#include <iostream>
#include <sys/types.h> // for pid_t
#include <unistd.h> // for fork()

using namespace std;

void helper() {
    pid_t pid;
    if ((pid = fork()) == 0) {
        exit(0);
    }
    else {
        cout << "child id = " << pid << endl;
    }
}

int main() {
    int *p = new int[3];
    helper();
    delete [] p;
}
```

A simple workaround for this issue if you do want to shut the process in helper, is to pass the pointer to the helper and free the memory before we exit.

```cpp
// ...
void helper(int *p) {
    pid_t pid;
    if ((pid = fork()) == 0) {
        delete [] p;
        exit(0);
    }
    else {
        cout << "child id = " << pid << endl;
    }
}

int main() {
    int *p = new int[3];
    helper(p);
    delete [] p;
}
```

<a name="Chapter1.2"></a>
### Possibly lost memory
As noted in the Valgrind manual page, it will detect pointers that could point into the middle of an allocated block and in most case, this should be interpreted as memory leak. It is not so intuitive at the first glance what makes this happen, but here I will just show a typical situation with the use of threads.

```cpp
#include <iostream>
#include <pthread.h>

void *threadWorker(void *arg) {
    int comm_fd = *(int*)arg;
    
    // Communicate with client
}

int main() {

    // listen_fd is created here
    
    while (true) {
        struct sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        int comm_fd = accept(listen_fd, (struct sockaddr*)&clientaddr, &clientaddrlen);
        pthread_t tid;
        
        // Create a new thread to handle this connection
        pthread_create(&tid, NULL, threadWorker, &comm_fd);
    }
}
```

This is a basic architecture for a thread-based server. When a new connection is accepted, the main thread will spawn a new thread to handle this connection. Suppose the administrater may press Ctrl+C to end the server. In that case, a SIGINT will be sent to the process, which will terminate the whole process. If that happens, the program will exit without reclaiming resources allocated to the child threads, and that will cause the possibly lost memory allocated in pthread_create().

One way to solve this issue is to use ```signal(SIGINT, sigintHandler)``` to register the process to use our own handler for SIGINT. The main challenge here is that when a SIGINT is received, it is not certain which thread will actually run the signal handler. But we can use pthread_self() to identify the current thread ID, then we can send user defined signal to other threads and that signal, e.g. SIGUSR1, can be handled by another handler. The handler for SIGUSR1 will simply call pthread_exit(0), while the handler for SIGINT will wait for all other threads to exit and free any resources until it shuts the entire program. Also note that the default behavior of a threading receiving SIGUSR1 is to exit its execution.

```cpp
#include <iostream>
#include <pthread.h>

void sigusr1Handler(int arg) {
    pthread_exit(0);
}

void sigintHandler(int arg) {

    // Close any unclosed sockets
    
    // threadIDs are collected
    
    int selfTID = pthread_self();
    for (pthread_t tid : threadIDs) {
    	if (selfTID != tid) {
            pthread_kill(tid, SIGUSR1);
            void *status;
            pthread_join(tid, &status);
    	}
    }
    
    // Free any other resources
    
    exit(0);
}

void *threadWorker(void *arg) {
    int comm_fd = *(int*)arg;
    
    // Communicate with client
}

int main() {
    signal(SIGINT, sigintHandler);
    signal(SIGUSR1, sigusr1Handler);

    // listen_fd is created here
    
    while (true) {
        struct sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        int comm_fd = accept(listen_fd, (struct sockaddr*)&clientaddr, &clientaddrlen);
        pthread_t tid;
        
        // Create a new thread to handle this connection
        pthread_create(&tid, NULL, threadWorker, &comm_fd);
    }
}
```

<a name="Chapter2"></a>
## Inter-process communication
Some mechanism of inter-process communication will be briefly introduced here.
1. signal(in the previous chapter)
2. pipe
3. socket

<a name="Chapter2.1"></a>
### Communicate between processes using pipe

In unix like systems, a pipe or pipeline is a mechanism for inter-process communication using FIFO message passing.

```cpp
#include <unistd.h>
#include <sys/types.h>
#include <iostream>

int main() {
    int myPipe[2];
    pipe(myPipe); // Create a new pipe
    
    pid_t pid;    
    if ((pid = fork()) == -1) {
        std::cerr << "fork error" << std::endl;
    }
    else if (pid == 0) {
        close(myPipe[0]); // The read end is not used by child
        FILE* writeData = fdopen(myPipe[1], "w");
        for (int i = 0; i < 10; i++) {
            fprintf(writeData, "%d", i);
        }
        fclose(writeData);
    }
    else {
        close(myPipe[1]); // The write end is not used by parent
        FILE* readData = fdopen(myPipe[0], "r");
        char buffer[20];
        while (fgets(buffer, 20, readData) != nullptr) {
            std::cout << atoi(buffer) << std::endl;
        }
        fclose(readData);
    }
    return 0;
}
```

The end of the pipe is actually regarded as file decriptor in Unix. That's also why we can use fdopen() to create a FILE pointer. When the process exits, all file descriptors will be closed by the operating system.

<a name="Chapter2.2"></a>
### Communicate between threads using pipe
It is similar to use pipe for communication between threads, but it is tricky because all threads in a single process share the same address space. We should be cautious of this, especially when passing a pointer argument.

```cpp
#include <iostream>
#include <pthread.h>

void *threadFunc(void *arg) {
    int id = *(int*)arg;
    std::cout << id << std::endl;
    pthread_exit(NULL);
}

int main() {
    pthread_t threads[10];
    for (int i = 0; i < 10; i++) {
        pthread_create(&threads[i], NULL, threadFunc, &i);
    }
    
    void *status;
    for (int i = 0; i < 10; i++) {
        pthread_join(threads[i], &status);
    }
}
```

Each thread has its unique stack pointer, but all threads here share the same address space. So, pointers to the variable i will have the same address value. Since threads run concurrently, the value of i could change before it is read to id of each thread.

A solution for this is to store the value passed to each thread in different places.

```cpp
// ...
int main() {
    pthread_t threads[10];
    int arguments[10];
    for (int i = 0; i < 10; i++) {
        arguments[i] = i;
        pthread_create(&threads[i], NULL, threadFunc, &arguments[i]);
    }
    
    void *status;
    for (int i = 0; i < 10; i++) {
        pthread_join(threads[i], &status);
    }
}
```

<a name="Chapter2.3"></a>
### Socket and SIGPIPE
Socket is another way for inter-process communication and can be used for internet communication with TCP or UDP. Similar to a pipe, it is using file descriptor to identify the target of communication. But unlike a pipe, the communication of socket is two-way, which means you can not only read but also write to another process.

When a process attempts to write to a socket that is no longer open for reading, it will receive a SIGPIPE signal, meaning "broken pipe". This broken status can also be detected by the write() function with a negative return value. Since the default action of receiving SIGPIPE is to terminate the process, it is convenient for some applications to just ignore this signal using ```signal(SIGPIPE, SIG_IGN);``` and check the return value of write() function.

<a name="Chapter3"></a>
## Some commonly used commands in Linux
This chapter contains some commands very useful for c++ programming in Linux.

<a name="Chapter3.1"></a>
### g++/gdb
Suppose we have a file named `main.cpp`, we would like to compile it and run in gdb.

`g++ main.cpp -g -o main` will compile the file using g++. `-g` option will allow debug information to be generated for gdb and `-o` option will allow code optimization. It generates an executable in the working directory. Then we can use `./main` to run the program in the console.

`gdb main` will start debugging using gdb. We can then use `r` to run the program and we should also pass arguments here. For example, `r -v` will start the program with an argument "-v".

We can use `b 19` to set a breakpoint at line 19. gdb will create a unique ID for this breakpoint so we enable it and disable it using command `en 1` and `dis 1`. A complete version such as `enable 1` is also acceptable. `where`, `list`, `n(next)`, `step` are also helpful. `up` and `down` can be used to enter or exit a call stack.

Suppose we have a variable named "theta", then we can use `p theta` to print its value. We can also use `p *f` if f is a pointer and we want to know the value it is pointing to.

<a name="Chapter3.2"></a>
### valgrind
As mentioned in previous chapters, valgrind is very useful for debugging memory management errors. We can use `valgrind ./main` to run the program under valgrind. Another helpful tool is memory sanitizer, which also gives us helpful information about memory usage.

