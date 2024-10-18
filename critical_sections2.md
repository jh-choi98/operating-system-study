# Synchronization

critical sections의 동기화를 위해선 3가지 조건을 만족해야 한다.

- Mutual exclusion
- Bounded waiting
- Progress

## Semaphore

세마포어는 여러 threads에 공유되는 음이 아닌 정수이다. 세마포어는 critical section에 진입하기 전에 스위치를 사용중으로 놓고 들어간다. 이후에 도착하는 프로세스는 앞의 프로세스가 작업을 마칠 때까지 기다린다. 프로세스가 작업을 마치면 세마포어는 다음 프로세스에 critical section을 사용하라는 동기화 신호를 보낸다. 세마포어는 다른 알고리즘과 달리 Synchronization이 잠겼는지 직접 점검하거나 busy waiting을 하거나, 다른 프로세스에 동기화 메세지를 보낼 필요가 없다.

세마포어는 3가지 방식으로 이루어져 있다.

- Semaphore(n)로 초기 설정을 한다. 이 때 n은 공유 가능한 자원의 수를 나타낸다.
- Wait(P): P()으로 critical section에 들어가기 전에 사용중이라는 표시를 한다.
- Signal(V): V()으로 critical section에 나올 때 비었다고 표시를 한다.

```c
Semaphore(n);
P();
// critical section
V();
```

예시

```c
int RS = n; // Semaphore(n);

// P()
if (RS > 0) {
    RS -= 1;
} else {
    block();
}

// Critical sections

// V()
RS += 1;
wake_up();
```

1. Semaphore(n): 전역 변수 RS를 n으로 초기화한다. RS에는 현재 사용 가능한 자원의 수가 저장된다.
2. P(): 잠금을 수행하는 코드로 RS가 0보다 크면 (사용 가능한 자원이 있다면) 1만큼 감소시키고 임계구역에 진입한다. 만약 RS가 0보다 작으면 (사용할 수 있는 자원이 없으면) 0보다 커질 때까지 기다린다.
3. V(): 잠금 해제와 동기화를 같이 수행하는 코드로, RS 값을 1 증가시키고 세마포어에서 기다리는 프로세스에 임계구역에 진입해도 좋다는 wake_up 신호를 보낸다.

세마포어에서 잠금이 해제되기를 기다리는 프로세스는 세마포어 큐에 저장되어 있다가 wake_up 신호를 받으면 큐에서 나와 임계구역에 진입한다. 따라서 busy waiting하는 프로세스가 없게 된다.

그러나 세마포어의 P(), V() 내부 코드가 실행되는 도중에 context switching이 발생하면 mutial exclusion과 bounded waiting 조건을 보장하지 못한다. 그 이유는, 프로세스가 세마포어의 상태를 검사하고 변경하는 도중에 다른 프로세스가 끼어들면, 세마포어의 상태가 일관되지 않을 수 있기 때문이다.

그래서 세마포어의 P()와 V() 함수 내부 코드는 원자적(atomic)으로 실행되어야 한다. Atomicity는 코드가 중간에 방해받지 않고 완전히 실행되는 것을 의미한다. 이를 통해 세마포어의 상태를 안전하게 변경하고, 상호 배제와 유한 대기 조건을 만족시킬 수 있다.

예를 들어, 프로세스가 세마포어를 통해 임계 구역에 진입하려고 할 때, 세마포어의 상태를 검사한 후(잠금이 풀렸는지 확인) 진입 상태로 변경하는 과정이 단일 작업으로 이루어져야 한다. 만약 이 과정이 분리되어 각각 따로 실행된다면, context switching이 발생할 경우 문제가 생길 수 있다. 따라서 검사와 지정이 원자적으로 동시에 실행되어야 한다.

즉, 세마포어의 P(), V() 함수는 중간에 방해받지 않고 원자적으로 실행되어야만 상호 배제와 유한 대기 조건을 만족시킬 수 있습니다.

예시

```
공유자원: 예금
p1: 예금 += 10만원
p2: 예금 -= 5만원
```

위 예시에서 공유 자원은 예금이 하나이므로 RS의 초기값은 1이다.

1. 먼저 도착한 프로세스 P1이 임계구역에 진입한다. 현재 RS는 1이므로 이 값을 1 감소시키고 임계구역에 진입한다.
2. 나중에 도착한 프로세스 P2는 현재 RS값이 0이므로 프로세스 P1이 임계구역에 빠져나올 때까지 세마포어 큐에서 기다린다.
3. 프로세스 P1은 현재 예금이 10만원 인 것을 확인하고 10만원을 더해 20만원으로 바꾼 다음 작업을 마친다.
4. 프로세스 P1은 V()를 실행하여 RS 값을 1 증가시키고 wake_up 신호를 프로세스 P2에 보낸다.
5. block 상태에 있던 프로세스 P2는 코드를 실행시킨다.

세마포어는 공유 자원이 여러 개일 때도 사용할 수 있다. 위 그림은 세마포어를 사용하여 2개의 공유 자원을 가지고 3개의 프로세스가 작업하는 예를 나타낸 것이다.

1. 프로세스 P1은 RS 값을 1 감소시키고 임계구역에 진입한다.
2. 프로세스 P2도 RS 값을 1 감소시키고 임계구역에 진입한다.
3. 프로세스 P3은 RS 값이 0이므로 다른 프로세스가 임계구역을 빠져나올 때까지(RS가 0보다 커질 때까지) 기다린다.
4. 프로세스 P1이 작업을 마치고 V()를 실행하면 RS 값은 1이 되고, wake_up 신호가 프로세스 P3에 전달된다.
5. 프로세스 P3가 임계구역에 진입한다.

## Mutex

A mutex is a synchronization mechanism used to prevent multiple processes or threads from accessing a shared resource (like memory, files, or variables) at the same time. By allowing only one process or thread to use the resource at a given moment, it prevents race conditions and ensures the consistency of the resource.

There is an ambiguity btw semaphore and mutex. We might have come across that a mutex is a binary semaphore. But it is not.

A mutex is a locking mechanism used to synchronize access to a resource. Only one task can acquire the mutex. It means there is ownership associated with a mutex, and only the owner can release the lock.

뮤텍스는 Mutual Exclusion의 줄임말로, 여러 스레드나 프로세스가 공유 자원에 동시에 접근하는 것을 막기 위해 사용되는 동기화 메커니즘이다. 상호 배제를 보장함으로써 임계구역 문제를 해결하는 도구이다.

특징

- 상호 배제 보장: 한 번에 하나의 스레드만 임계구역에 들어갈 수 있도록 보장
- 잠금과 해제: 뮤텍스는 두 가지 주요 연산을 사용하여 작동
  - Lock: 스레드가 공유 자원에 접근하기 전에 lock을 걸어 다른 스레드가 그 자원에 접근하지 못하게 한다.
  - Unlock: 스레드가 임계구역에서의 작업을 마치면 unlock을 호출하여 다른 대기중인 스레드가 해당 자원에 접근할 수 있도록 한다.
- 소유권: 뮤텍스는 소유권의 개념을 가지고 있다. 즉, 특정 스레드가 뮤텍스를 잠그면 그 스레드만이 잠금을 해제할 수 있다. 이는 세마포어와의 주요 차이점 중 하나. 세마포어는 소유권 개념에 없어, 잠금과 해제가 다른 스레드에 의해 실행될 수 있다.
- Busy Waiting 방지: 뮤텍스를 사용할 때, busy waiting(자원을 계속 점검하며 대기하는 상태)를 방지할 수 있다. 스레드가 lock을 걸지 못하면 대기 상태로 전환되고, 자원이 해제될 때까지 CPU 자원을 낭비하지 않고 대기한다.

Mutex의 동작 방식

- Lock을 걸 때: 만약 다른 스레드가 이미 임계구역에서 작업 중이라면, 해당 스레드는 잠금을 걸 수 없습니다. 대신, 잠금이 해제될 때까지 대기 상태에 들어갑니다.

- Unlock을 할 때: 임계구역에서 작업을 마친 스레드가 unlock을 호출하면, 대기 중인 다른 스레드가 임계구역에 진입할 수 있게 됩니다.

```c
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t mutex; // 뮤텍스 선언
int shared_resource = 0; // 공유 자원

void* thread_function(void* arg) {
    pthread_mutex_lock(&mutex); // 임계구역 진입 전에 잠금
    shared_resource++; // 임계구역: 공유 자원 수정
    printf("Shared Resource: %d\n", shared_resource);
    pthread_mutex_unlock(&mutex); // 임계구역 나갈 때 잠금 해제
    return NULL;
}

int main() {
    pthread_t thread1, thread2;

    // 뮤텍스 초기화
    pthread_mutex_init(&mutex, NULL);

    // 두 개의 스레드 생성
    pthread_create(&thread1, NULL, thread_function, NULL);
    pthread_create(&thread2, NULL, thread_function, NULL);

    // 스레드 종료 대기
    // pthread_join: 특정 스레드가 종료될 때까지 대기.
    //               스레드가 종료되면 그 결과를 가져오는 방식
    // 즉, 여기서는 thread1과 thread2가 완전히 종료될 때까지 메인 스레드가
    // 기다리게 하는 역할을 한다.
    // pthread_join()을 호출하지 않으면, 스레드가 종료되더라도 일부 리소스가
    // 해제되지 않고 남아있을 수 있다. 따라서 리소스 누수를 방지하기 위해 스레드가
    // 끝난 후 반드시 호출해야 한다.
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);

    // 뮤텍스 소멸
    pthread_mutex_destroy(&mutex);

    return 0;
}
```
