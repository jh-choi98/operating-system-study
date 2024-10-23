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

### 주요 함수

- int sem_init(sem_t \*s, int pshared, unsigned int value)
  - 세마포어를 초기화하는 함수. 세마포어 객체 s를 초기화하며, 세마포어의 초기 값을 value로 설정한다.
  - s: 초기화할 세마포어 객체의 포인터
  - pshared: 프로세스 간 공유 여부를 결정합니다. 0으로 설정하면, 세마포어는 스레드 간에만 공유되고, 0이 아니면 프로세스 간에도 공유할 수 있다.
  - value: 세마포어의 초기 값
  - 성공 시 0을 반환하고, 실패 시 -1을 반환
- sem_wait(sem_t \*s)
  - 세마포어를 잠그는 함수. 세마포어가 0보다 크면 값을 1 감소시키고 자원을 사용한다. 세마포어가 0이면 자원이 모두 사용 중인 상태이므로 대기 상태로 전환
  - s: 잠글 세마포어의 포인터
  - 이 함수는 자원이 사용 가능할 때까지 스레드를 대기 상태로 만들며, 자원이 해제되면 스레드를 깨워 실행을 계속할 수 있게 한다.
- sem_post(sem_t \*s)
  - 세마포어를 해제하는 함수. 자원이 사용 중인 상태를 끝내고, 세마포어의 값을 1 증가시켜 자원을 사용할 수 있음을 나타낸다. 대기 중인 스레드가 있으면, 이를 깨워 자원에 접근하게 한다.

## Mutex

A mutex is a synchronization mechanism used to prevent multiple processes or threads from accessing a shared resource (like memory, files, or variables) at the same time. By allowing only one process or thread to use the resource at a given moment, it prevents race conditions and ensures the consistency of the resource.

There is an ambiguity btw semaphore and mutex. We might have come across that a mutex is a binary semaphore. But it is not.

A mutex is a locking mechanism used to synchronize access to a resource. Only one task can acquire the mutex. It means there is ownership associated with a mutex, and only the owner can release the lock.

뮤텍스는 Mutual Exclusion의 줄임말로, 여러 스레드나 프로세스가 공유 자원에 동시에 접근하는 것을 막기 위해 사용되는 동기화 메커니즘이다. 상호 배제를 보장함으로써 임계구역 문제를 해결하는 도구이다.

### 특징

- 상호 배제 보장: 한 번에 하나의 스레드만 임계구역에 들어갈 수 있도록 보장
- 잠금과 해제: 뮤텍스는 두 가지 주요 연산을 사용하여 작동
  - Lock: 스레드가 공유 자원에 접근하기 전에 lock을 걸어 다른 스레드가 그 자원에 접근하지 못하게 한다.
  - Unlock: 스레드가 임계구역에서의 작업을 마치면 unlock을 호출하여 다른 대기중인 스레드가 해당 자원에 접근할 수 있도록 한다.
- 소유권: 뮤텍스는 소유권의 개념을 가지고 있다. 즉, 특정 스레드가 뮤텍스를 잠그면 그 스레드만이 잠금을 해제할 수 있다. 이는 세마포어와의 주요 차이점 중 하나. 세마포어는 소유권 개념에 없어, 잠금과 해제가 다른 스레드에 의해 실행될 수 있다.

Mutex의 동작 방식

- Lock을 걸 때: 만약 다른 스레드가 이미 임계구역에서 작업 중이라면, 해당 스레드는 잠금을 걸 수 없습니다. 대신, 잠금이 해제될 때까지 대기 상태에 들어갑니다.

- Unlock을 할 때: 임계구역에서 작업을 마친 스레드가 unlock을 호출하면, 대기 중인 다른 스레드가 임계구역에 진입할 수 있게 됩니다.

### PThread Mutex API

- int pthread_mutex_init(pthread_mutex_t *m, pthread_mutexattr_t *attr)

  - Initialize a mutex
  - m: 뮤텍스 객체를 가리키는 포인터
  - attr: 뮤텍스 속성 (NULL이면 기본 속성으로 초기화)
  - 성공시 0을 반환하며, 실패 시 오류 코드를 반환

- int pthread_mutex_destroy(pthread_mutex_t \*m)

  - Destroy a mutex
  - 성공시 0을 반환하며, 실패 시 오류 코드를 반환

- int pthread_mutex_lock(pthread_mutex_t \*m)

  - 지정된 뮤텍스를 잠그는 함수. 다른 스레드가 이미 뮤텍스를 잠근 상태라면, 해당 스레드는 뮤텍스가 잠금 해제될 때까지 대기하게 된다. 이 함수는 뮤텍스가 잠금 상태일 때 다른 스레드가 동시에 접근하지 못하도록 한다.
  - 성공시 0을 반환하며, 실패 시 오류 코드를 반환

- int pthread_mutex_unlock(pthread_mutex_t \*m)

  - 이전에 잠금된 뮤텍스를 해제하는 함수. 이 함수가 호출되면 해당 뮤텍스를 잠글 수 있는 다른 스레드가 접근할 수 있게 된다.
  - 뮤텍스를 잠근 스레드만 호출할 수 있다. 뮤텍스를 잠그지 않은 (소유권이 없는) 스레드가 호출하면 정의되지 않은 동작이 발생할 수 있다.
  - 성공시 0을 반환하며, 실패 시 오류 코드를 반환

- int pthread_mutex_trylock(pthread_mutex_t \*m)
  - 뮤텍스를 잠그려고 시도하지만, 이미 다른 스레드가 뮤텍스를 잠그고 있을 경우 즉시 반환하는 함수. 다른 스레드가 뮤테스를 잠그지 않았다면 성공적을 잠그고 0을 반환하며, 뮤텍스가 이미 잠겨 있으면 EBUSY를 반환.
  - 성공 시 0을 반환하며, 실패 시 EBUSY(뮤텍스가 이미 잠긴 상태)를 반환합니다.

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

## Condition Variables

조건 변수는 스레드들이 공유 자원에 접근할 때 발생하는 동시성 문제를 해결하기 위해 사용된다. 여러 스레드가 하나의 자원을 공유할 때, 스레드가 특정 조건을 충족할 때까지 대기하게 하고, 다른 스레드가 그 조건을 충족시키면 대기 중인 스레드를 깨워서 실행하도록 만든다.

### 뮤텍스와 조건 변수의 차이

- 뮤텍스: 상호 배제를 보장하며, 공유 자원에 접근하는 동안 다른 스레드가 그 자원에 접근하지 못하게 하는 잠금 메커니즘이다.
- 조건 변수: 특정 조건이 만족될 때까지 스레드를 대기 상태로 만들고, 조건이 만족되면 그 스레드를 깨우는 방식으로 동기화를 관리한다. 뮤텍스와 함께 사용되며, 상태의 변화를 기다리는데 중점을 둔다.

### 동작 과정

1. 뮤텍스 잠금: 스레드가 임계 구역(critical section)에 진입하기 전에, 뮤텍스를 사용해 자원을 잠근다.
2. 조건 확인: 스레드는 조건 변수를 사용해 특정 조건을 확인한다. 예를 들어, 소비자가 생산자가 데이터를 준비할 때까지 기다리는 상황에서, 소비자는 데이터가 준비되었는지 확인한다. 이때, 데이터가 준비되지 않았다면 조건이 충족되지 않으므로 소비자는 대기해야 한다.
3. 조건 대기: 스레드가 조건이 충족되지 않았을 때, pthread_cond_wait() 함수를 호출하여 조건이 충족될 때까지 대기 상태로 들어간다. 이때 다른 스레드들이 자원을 사용할 수 있도록 뮤텍스는 자동으로 해제된다.
4. 조건 만족 신호: 다른 스레드(예: 생산자)가 조건을 충족시킬 때, pthread_cond_signal() 또는 pthread_cond_broadcast()를 호출하여 대기 중인 스레드에게 조건이 충족되었음을 알리는 신호를 보낸다.
5. 뮤텍스 재잠금 및 작업 재개: 신호를 받은 스레드는 조건이 충족되었으므로 pthread_cond_wait()이 끝나고, 뮤텍스를 다시 잠금으로써 자원에 접근하여 작업을 재개한다. 작업이 완료되면 뮤텍스를 해제하여 다른 스레드들이 자원에 접근할 수 있도록 한다.

### 주요 함수

- int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr)

  - 조건 변수를 초기화하는 함수.
  - cond: 조건 변수의 객체 포인터
  - attr: 조건 변수 속성 (기본 속성은 NULL)
  - 성공 시 0을 반환하고, 실패 시 오류 코드를 반환

- int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex): 스레드가 조건을 기다릴 때 사용

  - 조건 변수가 신호를 받을 때까지 스레드를 대기 상태로 만든다. 뮤텍스를 자동으로 잠금 해제하면서 스레드를 대기 상태로 만들고, 조건 변수가 신호를 받으면 뮤텍스를 다시 잠그고 실행을 계속한다.
  - 뮤텍스 잠금 해제: 대기하는 동안 다른 스레드들이 임계 구역에 진입하여 자원을 사용할 수 있도록 뮤텍스가 자동으로 해제된다.
  - 대기 상태로 전환: 스레드는 조건이 충족될 때까지 대기 상태로 들어가며, CPU 자원을 소모하지 않고 효율적으로 대기한다.
  - 조건 충족 후 뮤텍스 재잠금: 다른 스레드가 조건을 충족시켜 pthread_cond_signal() 또는 pthread_cond_broadcast()가 호출되면, 스레드는 다시 뮤텍스를 잠금 상태로 변경한 후 작업을 재개한다.

- int pthread_cond_signal(pthread_cond_t \*cond)

  - 조건이 충족되었을 때, 대기 중인 스레드 중 하나를 깨우기는 함수
  - 이 함수는 하나의 대기 중인 스레드만 깨운다.
  - 스레드가 깨어나면, 다시 뮤텍스를 잠그고 조건을 확인한 후 임계 구역에서 작업을 수행한다.
  - 조건 변수는 뮤텍스와 함께 사용되어 뮤텍스에 접근하려고 대기 중인 스레드들 중 하나가 신호를 받기 때문에, 특정 스레드를 인자로 줄 필요가 없다.
  - 즉, 어떤 스레드가 신호를 받는지는 미리 지정되지 않지만, pthread_cond_wait()을 호출하여 조건 변수에서 대기하고 있는 스레드 중 하나가 신호를 받는 것이다.
  - cond: 신호를 보낼 조건 변수의 포인터
  - 성공 시 0을 반환하고, 실패 시 오류 코드를 반환

- int pthread_cond_broadcast(pthread_cond_t \*cond)
  - 조건이 충족되었을 때, 대기 중인 모든 스레드를 깨우는 함수
  - 반환값: 성공 시 0을 반환하고, 실패 시 오류 코드를 반환합니다.

### 예시

```c
#include <pthread.h>
#include <stdio.h>

// PTHREAD_MUTEX_INITIALIZER, PTHREAD_COND_INITIALIZER
// 정적 초기화를 위해 제공되는 매크로. 기본값을 지정.
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cont_t cond = PTHREAD_COND_INITIALIZER;
int buff = 0; // 공유 자원: 버퍼
int produced = 0; // 생성된 항목의 개수

void *producer(void *arg) {
    for (int i = 0; i < 5; i++) {
        pthread_mutex_lock(&mutex); // 뮤텍스 잠금
        buff += 1; // 공유 자원에 접근
        printf("Produced: %d\n", buff);
        produced = 1; // 생산 완료 상태 설정
        pthread_cond_signal(&cond); // 소비자에게 신호 보내기
        pthread_mutex_unlock(&mutex); // 뮤텍스 해제
    }
    return NULL;
}

void *consumer(void *arg) {
    for (int i = 0; i < 5; i++) {
        pthread_mutex_lock(&mutex); // 뮤텍스 잠금
        while (produced == 0) { // 생산이 완료되지 않았으면 대기
            pthread_cond_wait(&cond, &mutex); // 조건이 만족될 때까지 대기
        }
        buff -= 1; // 공유 자원에 접근
        printf("Consumed: %d\n", buff);
        produced = 0; // 소비 완료 상태 설정
        pthread_mutex_unlock(&mutex); // 뮤텍스 해제
    }
    return NULL;
}

int main() {
    // 스레드 ID를 저장할 변수를 선언
    pthread_t producer_thread, consumer_thread;

    // 새로운 스레드들 생성
    pthread_create(&producer_thread, NULL, producer, NULL);
    pthread_create(&consumer_thread, NULL, consumer, NULL);

    // 특정 스레드가 종료될 때까지 현재 스레드가 대기하게 만드는 함수
    // 스레드가 완전히 종료되기 전까지 리소스 해제를 방지
    // NULL: 반환 값을 저장하기 않겠다.
    pthread_join(producer_thread, NULL);
    pthread_join(consumer_thread, NULL);

    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
}

/*
1. producer 스레드는 5번 반복하며 공유 자원(buffer)에 값을 추가하고, produced 플래그를 1로 설정하여 생산이 완료되었음을 알립니다. 이후 pthread_cond_signal()을 호출하여 소비자 스레드에게 신호를 보냅니다.

2. consumer 스레드는 produced == 0일 경우 pthread_cond_wait()을 호출하여 생산자가 생산을 완료할 때까지 대기합니다. 신호를 받으면 buffer 값을 감소시키고 자원을 소비한 후 produced 플래그를 0으로 설정합니다.

3. 두 스레드는 뮤텍스를 사용하여 공유 자원을 보호하며, 조건 변수를 통해 상태 변화를 동기화합니다.
*/
```

### Re-check conditions

- 조건 재확인이 필요한 이유
  - 조건 변수를 기다리던 스레드가 깨워져도, 그 순간 조건이 만족되지 않았을 가능성이 존재. 이런 경우에 조건을 다시 확인하지 않으면, 잘못된 상태에서 동작하게 될 수 있다. 따라서 조건을 만족해 신호를 받고 깨어나더라도, 다시 조건을 확인해야 한다.
  - Spurious Wakeups
    - 어떤 스레드가 조건 변수를 신호하지 않았음에도 스레드가 깨어나는 경우
  - Multiple Threads waiting
    - 만약 두개 이상의 스레드가 동시에 조건 변수를 기다리고 있다면, 조건 변수가 신호되었을 때 두 스레드가 깨어날 수 있다.
    - 하나의 스레드만 실제로 사용할 수 있는 자원이 있을 수 있으므로, 다른 스레드는 다시 조건을 확인하고 대기해야 한다.
