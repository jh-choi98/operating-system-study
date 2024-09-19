# Threads

## Overview

- Thread: a basic unit of CPU utilization, consisting of a program counter, a stack, and a set of registers, and a thread ID
- Traditional processes have a single thread of control. There is one program counter, and one sequence of instructions
- Multi-threaded applications have multiple threads within a single process, each having their own program counter, stack and set of registers, but sharing common code, data, and certain structures such as open files

### Motivation

- Threads are very useful whenever a process has multiple tasks to perform independently of the others
- For example in a web server: multiple threads allow for multiple requests to be satisfied simultaneously, without having to service requests sequentially or to fork off separate processes for every incoming request

### Multithreading Models

- There are 2 types of threads to be managed: User threads and Kernel threads
- User threads are supported above the kernel, without kernel support
- Kernel threads are supported within the kernel of the OS itself
- The user threads must be mapped to kernel threads, using one of the following strategies:

  - Many-To-One model

    - many user-level threads are all mapped onto a single kernel thread
    - Thread management is handled by the thread library in user space, which is very efficient
    - However, if a blocking system call is made, then the entire process blocks, even if the other user threads would be able to continue
    - Because a single kernel thread can operate only on a single CPU, the many-to-one model does not allow individual processes to be split across multiple CPUs.
    - Few systems continue to do so today

  - One-To-One model

    - creates a separate kernel thread to handle each user thread
    - Overcomes blocking system calls and enables the splitting of processes across multiple CPUs
    - However, causes more overhead and slows down the system

  - Many-To-Many Model
    - multiplexes any number of user threads onto an equal or smaller number of kernel threads, combining the best features of the one-to-one and many-to-one models
    - Users have no restrictions on the number of threads created
    - Blocking kernel system calls do not block the entire process
    - Processes can be split across multiple processors
    - Individual processes may be allocated variable numbers of kernel threads

### Thread Libraries

- provide with an API for creating and managing threads
- may be implemented either in user space or in kernel space. The former involves API functions implemented solely within user space, with no kernel support. The latter involves system calls, and requires a kernel with thread library support
- There are 3 amin thread libraries in use today:

  1. POSIX Pthreads - may be provided as either a user or kernel library, as an extension to the POSIX standard
  2. Win32 threads - provided as a kernel-level library on Windows systems
  3. Java thread

- Example: Multithreaded C program using the Pthreads API

```c
#include <pthread.h>
#include <stdio.h>

int sum; // this data is shared by the thread(s)
         // stores the result from threads
void *runner(void *param); // the thread

int main(int argc, char *argv[]) {
    pthread_t tid; // the thread identifier
    pthread_attr_t attr; // set of thread attributes

    if (argc != 2) {
        fprintf(stderr, "usage: a.out <integer value>\n");
        return -1;
    }
    if (atoi(argv[1]) < 0) {
        fprintf(stderr, "%d must be >= 0 \n", atoi(argv[1]));
        return -1;
    }

    // get the default attributes
    pthread_attr_init(&attr);

    // create the thread
    // 이때, argv[1]로 전달된 인자가 runner 함수의 매개변수로 전달
    pthread_create(&tid, &attr, runner, argv[1]);

    // wait for the thread to exit
    pthread_join(tid, NULL);

    printf("sum = %d\n", sum);
}

// The thread will begin control in this function
void *runner(void *param) {
    int i, upper = atoi(param);
    sum = 0;

    for (i = 1; i <= upper; i++) {
        sum += i;
    }

    pthread_exit(0);
}

// 이 프로그램은 스레드를 사용하여 계산을 비동기적으로 처리한 후, 메인 프로그램에서
// 스레드가 완료될 때까지 기다렸다가 결과를 출력하는 방식을 작동
```

```c
int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: a.out <integer value>\n");
        return -1;
    }
    if (atoi(argv[1]) < 0) {
        fprintf(stderr, "%d must be >= 0 \n", atoi(argv[1]));
        return -1;
    }
    int sum = 0;
    int upper = atoi(argv[1]);
    for (int i = 0; i <= upper; i++) {
        sum += i;
    }
    printf("sum = %d\n", sum);
}

// 스레드를 사용하지 않고 메인 함수에서 직접 계산을 수행
```

- 두 함수 차이점
  - Multi-threads vs Single-thread
    - 첫 번째: POSIX thread를 사용하여 별도의 스레드에서 계산 작업을 처리. 프로그램이 동시에 여러 작업을 처리할 수 있는 구조로 확장 가능
    - 두 번째: 단일 스레드로 작동. 작업이 순차적으로 처리됨.
  - 비동기적 작업 vs 동기적 작업
    - 첫 번째: 비동기적으로 작동. 즉, 메인 프로그램이 스레드를 생성한 후, 그 스레드에서 계산이 완료될 때까지 기다리는 구조. 이를 통해 동시에 여러 작업을 실행할 수 있다.
    - 두 번째: 동기적으로 작동. 프로그램이 한 적업을 완료하기 전까지 다른 작업을 시작하지 않는다. 즉, 계산이 끝날 때까지 다른 작업을 할 수 없다.
  - 스레드 관리
    - 첫 번째: 스레드 관리를 위해 pthread_create와 pthread_join을 사용하여, 스레드를 생성하고 종료를 기다리는 메커니즘을 포함.
    - 두 번째: 스레드 사용x
