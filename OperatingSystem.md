# Operating System
Random Notes

### still reachable memory
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
