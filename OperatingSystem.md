# Operating System
Random Notes

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

## Using pipe to communicate
In unix like systems, a pipe or pipeline is a mechanism for inter-process communication using FIFO message passing.

### Communicate between processes
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

### Communicate between threads
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

Each thread has its unique stack pointer, but all threads here share the same address space. So, A pointer to the variable i will have the same address value. Since threads run concurrently, the value of i could change before it is read to id of each thread.

A solution for this is to store the value passed to each thread in different places.

```cpp
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
