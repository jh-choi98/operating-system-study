```c
// Simple Thread Creation Code
#include <stdio.h>
#include <assert.h>
#include <pthread.h>
#include "common.h"
#include "common_threads.h"

void  *mythread(void *arg) {
    printf("%s\n", (char *) arg);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p1, p2;
    int rc;
    printf("main: begin\n");
    pthread_create(&p1, NULL, mythread, "A");
    pthread_create(&p2, NULL, mythread, "B");
    // join waits for the threads to finish
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    printf("main: end\n");
    return 0;
}
```

1. int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *( *fn)(void *), void *arg);
   - creates a new thread identified by 'thread' with optional attributes, run fn with arg
   - attr: usually NULL
   - fn: function pointer
   - arg: argument to be passed to fn

```c
// Creating a thread
#include <stdio.h>
#include <pthread.h>

typedef struct {
    int a;
    int b;
} myarg_t;

void *mythread(void *arg) {
    myarg_t *args = (myarg_t *) arg;
    printf("%d %d\n", args->a, args->b);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    myarg_t args = {10, 20};

    int rc = pthread_create(&p, NULL, mythread, &args);
    ...
}
```

2. int pthread_join(pthread_t thread, void \*\*value_ptr);
   - waits for 'thread' to exit and receive the return value
   - pthread_t thread: used to specify which thread to wait for
   - void \*\*value_ptr: a pointer to the return value you expect to get back (스레드가 종료될 때 반환 값을 받아오는 곳)

```c
// Waiting for thread completion
typedef struct {int a; int b;} myarg_t;
typedef struct {int x; int y;} myret_t;

void *mythread(void *arg) {
    myret_t *rvals = Malloc(sizeof(myret_t)); // heap에 동적으로 할당 -> 스레드가 종료된 후에도 메인 스레드에서 해당 메모리에 접근할 수 있다.
    rvals->x = 1;
    rvals->y = 2;
    return (void *) rvals;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    myret_t *ravls;
    myarg_t args = {10, 20};
    pthread_create(&p, NULL, mythread, &args);
    pthread_join(p, (void **) &rvals);
    free(rvals);
    return 0;
}
```

```c
void *mythread(void *arg) {
    long long int value = (long long int) arg;
    return (void *) (value + 1);
}

int main(int argc, char *argv[]) {
    pthread_t p;
    long long int rvalue;
    pthread_create(&p, NULL, mythread, (void *) 100);
    pthread_join(p, (void **) &rvalue);
    return 0;
}
```

3. Locks

   - int pthread_mutex_lock(pthread_mutex_t \*mutex);
   - int pthread_mutex_unlock(pthread_mutex_t \*mutex);

   ```c
   pthread_mutex_t lock;
   pthread_mutex_lock(&lock);
   x += 1; // critical section
   pthread_mutex_unlock(&lock);
   /*
   If no other thread holds the lock when pthread_mutex_lock() is called, the thread will acquire the lock and enter the critical section.
   If another thread holds the lock, the thread trying to grab the lock will not return from the call until it has acquired the lock.
   ONLY the thread with the lock acquired should call unlock
   */
   ```

   However, the code example above is broken in two ways:

   1. Lack of proper initialization
      => Ways to initialize locks

   - PTHREAD_MUTEX_INITIALIZER
     pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
   - pthread_mutex_init(): dynamic way
     int rc = pthread_mutex_init(&lock, NULL);
     assert(rc == 0); // always check success

   2. Fails to check error codes when calling lock and unlock

4. Conditional Variables
   한 스레드가 특정 조건이 만족될 때까지 기다리도록 하고, 다른 스레드가 그 조건을 만족시키면 신호를 보내서 기다리던 스레드를 깨운다.

- int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
  - 조건이 충족될 때까지 스레드를 기다리게 한다.
  - 스레드를 일시정지 시키고, 전달된 mutex 잠금을 해제하여 다른 스레드가 mutex에 접근할 수 있게 한다.
  - 조건이 충족되어 cond 조건 변수에 신호가 오면 다시 mutex를 획득한 후 스레드가 깨어나 실행을 재개한다.
- int pthread_cond_signal(pthread_cond_t \*cond);
  - cond 조건 변수를 기다리고 있는 스레드 중 하나에게 신호를 보내어 깨어나도록 한다.
  - 조건을 만족시키는 이벤트가 발생할 때 호출되어 조건 변수 cond를 기다리고 있는 스레드를 깨운다.

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

void *thread1(void *arg) {
    pthread_mutex_lock(&lock);
    while (ready == 0) { // 조건이 충족되지 않으면 대기
        pthread_cond_wait(&cond, &lock); // 효울적으로 대기
    }
    // 조건이 충족된 후 실행할 코드
    pthread_mutex_unlock(&lock);
    return NULL;
}

void *thread2(void *arg) {
    pthread_mutex_lock(&lock);
    ready = 1; // 조건을 충족시키는 코드
    pthread_cond_signal(&cond); // 기다리고 있는 스레드에 신호를 보냄
    pthread_mutex_unlock(&lock);
    return NULL;
}
```
