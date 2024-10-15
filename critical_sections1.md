# 공유 자원(Shared resources)과 임계구역(Critical sections)

## 공유 자원의 접근

- 공유 자원은 여러 프로세스가 공동으로 이용하는 변수, 메모리, 파일등을 말한다. 공유 자원은 공동으로 이용되기 때문에 누가 언제 어떻게 데이터를 읽거나 쓰거나에 따라 그 결과가 달라진다. 그래서 원치 않은 문제가 발생하기도 한다.

### 경쟁 조건(Race condition)

- 경쟁 조건은 2개 이상의 프로세스가 공유 자원을 병렬적으로 읽거나 쓰는 상황을 말하며, 공유 자원 접근 순서에 따라 실행 결과가 달라지는 상황을 말한다. (a.k.a 동시성 문제)

### 임계구역(Critical section)

- 공유 자원 접근 순서에 따라 실행 결과가 달라지는 프로그램의 코드 영역을 임계구역이라고 한다. 즉, 임계구역 안에서 race condition이 발생하는 것이다.

### 데이터 레이스(Data race)

- 여러 스레드가 동시에 동일한 메모리 위치에 접근하여 값을 변경할 때 발생하는 문제를 Data race라고 한다.

### 생산자-소비자 문제

- Data race의 전형적인 예시로 생산자-소비자 문제(producer-consumer problem)가 있다.
- 생산자-소비자 문제에서 생산자 프로세스와 소비자 프로세스는 서로 독립적으로 작업을 한다. 가운데에는 buf라는 큐 자료구조가 존재하며 sum은 buf 안의 원소 수를 나타낸다. producer는 buf에 원소를 맨 뒤부터 차근차근 넣고 consumer는 맨 앞에서부터 원소를 빼낸다.

  ```c
      producer() {
          input(buf);
          sum += 1;
      }

      consumer() {
          output(buf);
          sum -= 1;
      }
  ```

- 이 producer와 consumer가 동시에 실행되면 문제가 발생한다.
- 예시
  - 현재 sum이 3인 buf가 있다.
  - producer가 buf에 물건을 하나 추가한다. 그러나 sum += 1 코드가 실행되기 전이라 현재 sum은 3이다.
  - consumber가 buf에서 물건을 하나 소비한다. 그러나 sum -= 1 코드가 실행되기 전이라 현재 sum은 3이다.
  - 이 상태에서 sum += 1과 sum -= 1이 동시에 실행되는 문제가 생긴다.
  - 만약 sum += 1이 먼저 실행되고, sum -= 1이 그뒤에 실행되면 sum은 2가 된다. 왜냐하면 코드가 실행될 때의 sum은 4가 아니라 3이기 때문이다.
  - 만약 sum -= 1이 먼저 실행되고, sum += 1이 그뒤에 실행되면 sum은 4가 된다. 왜냐하면 코드가 실행될 때의 sum은 3이기 때문이다.
  - 실행 순서에 따라 값이 변한다.

### 임계구역 해결 조건

임계구역 문제를 해결할 수 있는 방법은 다음 3가지 조건을 만족해야 한다.

- 상호 배제(mutual exclusion): 한 프로세스가 임계구역에 들어가면 다른 프로세스는 임계구역에 들어갈 수 없다.
- 한정 대기(bounded waiting): 상호 배제 때문에 기다리게 되는 프로세스가 무한 대기하지 않아야 한다. 즉, 특정 프로세스가 임계구역에 진입하지 못하면 안 된다.
- 진행의 융통성(progress flexibility): 임계구역에 프로세스가 없다면 어떠한 프로세스라도 들어가서 자원을 활용할 수 있다. 즉, 두 푸로세스가 자원을 번갈아 쓴다고 가정할 때, 한 쪽에서 자원을 사용하는다면 자신의 차례와 상관없이 자원을 사용하는게 효율적이라는 것이다.

## 임계구역 해결방법 (해결 조건을 충족하지 못한 방법)

Data race 문제를 해결하는 방법은 Lock을 사용하는 것이다. 즉, 한 프로세스가 임계구역에 들어간다면 lock을 걸어 다른 프로세스가 들어오지 못하게 하는 것이다. 그리고 프로세스가 임계구역에서 빠져나오게 되면 lock을 해제하고 동시에 동기화 신호를 보내 다음 프로세스에게 임계구역을 사용해도 좋다는 신호를 준다. 이 때, 상호 배제, 한정 대기, 진행의 융통성을 모두 만족해야 한다. 이번 섹션에서 살펴볼 해결방법은 임계구역 해결 조건을 완벽하게 충족하지 못한 방법이다.

### 예제 코드

```c
#include <stdbool.h>
extern bool lock = false;
extern int balance;

int main() {
    while (lock == true);
    lock = true;
    balance += 10; // critical sections
    lock = false;
}
```

임계구역에 들어갈 변수는 balance이고, 잠금은 lock으로 걸어놓는다. false이면 프로세스가 들어가 balance를 쓸 수 잇고, true면 기다리도록 한다.

### 상호 배제 위반 문제 (mutual exclusion)

위 예제는 IPC 방법으로 shared memory를 사용하는 방식이다. 공유 변수 lock은 shared memory에 있는 변수이다. 프로세스 p1과 p2는 임계구역에 진입하기 전에 코드를 통해 임계구역에 잠금이 걸려 있는지 확인한다. (lock == true)

만약 lock이 true라면 다른 프로세스가 임계구역에서 작업을 하고 있다는 뜻이므로 무한 루프를 통해 기다리게 된다.

임계구역에 있는 프로세스가 작업을 마치고 lock를 false로 만들면 기다리고 있던 프로세스는 무한 루프에 빠져나와 작업을 한다.

즉, lock = false는 임계구역을 사용해도 좋다고 다른 프로세스에 보내는 동기화 신호다. 그런데 이 방식에는 일부 문제가 존재한다.

- 예시
  - p1은 while(lock == true);문을 실행한다. lock은 false이므로 p1은 무한 루프를 빠져나와 임계구역에 진입할 준비를 한다.
  - 이때 p1이 할당된 CPU 시간을 다 사용하여 타임아웃(timeout) 상태가 된다. 이로 인해 컨텍스트 스위칭(context switching)이 발생하고, p1은 레디 상태(ready state)로 전환된다.
    - 컨텍스트 스위칭이란, 한 프로세스가 실행 중 다른 프로세스가 실행을 위해 CPU를 할당받는 과정이다. 이 과정에서 p1의 상태는 저장되며, p2가 실행된다.
  - p2가 실행 상태가 되면서 while(lock == true);문을 실행한다. 이때 lock이 여전히 false이기 때문에 (p1이 아직 lock = true;를 실행하지 못했기 때문), p2는 무한 루프를 빠져나와 임계구역에 진입할 준비를 한다.
  - p1이 레디 상태에서 실행 상태로 돌아오면, lock = true;문을 실행하며 임계구역에 진입한다.
  - 문제는 이 시점에서 p2도 이미 임계구역에 진입할 수 있는 상태라는 것이다.
  - p2도 lock = true;를 설정하여 lock을 활성화시키고 임계구역에 진입한다.
  - 결과적으로 p1과 p2는 모두 임계구역에 진입하게 되어, 상호 배제(mutual exclusion)를 위배한다.
  - 또 다른 문제는 잠금이 해제되기 전까지 busy waiting이 발생한다는 것이다. while(lock == true);문에서 작업을 할 필요가 없음에도 무한 루프를 돌면서 시스템 자원을 낭비한다.

### 한정 대기 위반 문제 (bounded waiting)

이전에는 lock을 두 프로세스가 공유하였기 때문에 문제였다. 그러면 각 프로세스가 자신들만의 lock을 가지고 있도록 lock을 두 개를 두자.

```c
#include <stdbool.h>
extern bool lock1 = false;
extern bool lock2 = false;
extern int balance;

int p1() {
    lock1 = true;
    while (lock2 == true);
    balance += 10;
    lock1 = false;
}

int p2() {
    lock2 = true;
    while (lock1 == true);
    balance += 10;
    lock2 = false;
}
```

위 코드는 담금을 하고 다른 프로세스가 잠겼는지 확인하므로 두 프로세스의 상호 배제가 보장된다. 그런데 두 프로세스가 모두 임계구역에 진입하지 못하는 무한 대기 현상이 일어난다.

- 예시
  - 프로세스 P1은 lock1=true; 문을 실행한 후 주어진 CPU 시간을 다 써버려 ready 상태로 돌아간다. context switching이 발생하고 프로세스 P2가 running 상태로 바뀐다.
  - 프로세스 P2도 lock2=true;문을 실행한 후 자신의 CPU 시간을 다 써버려 ready 상태로 돌아간다. context switching이 발생하고 프로세스 P1이 running 상태로 바뀐다.
  - 프로세스 P2가 lock2=true;문을 실행했기 때문에 프로세스 P1은 while(lock2 == true); 문에서 무한 루프에 빠진다.
  - 프로세스 P1이 lock1=true;문을 실행했기 때문에 프로세스 P2도 whilc(lock1 == true); 문에서 무한 루프에 빠진다.

결국 p1, p2 둘다 while문을 빠져나오지 못하고 무한 루프에 빠져 임계구역에 진입하지 못한다. 이는 bounded waiting을 보장하지 못하고 deadlockdp 빠진 것이다.
또한, 위의 코드는 확장성의 문제가 있다. 만약 p3가 추가된다면 또 lock3 주어야 한다.

### 진행의 융통성 문제

```c
int lock = 1;
extern int balance;

int p1() {
    while (lock == 2);
    balance += 10;
    lock = 2;
}

int p2() {
    while (lock == 1);
    balance += 10;
    lock = 1;
}
```

잠금을 확인하는 문장은 하나이므로 mutual exclusion과 bounded waiting을 보장한다. 그러나 서로 번갈아가면서 실행된다는 것이 문제이다. 한 프로세스가 두 번 연달아 임계구역에 진입하고 싶어도 그럴 수 없다.

p1는 p2가 critical section에 진입하고 나서야 critical section에 진입할 수 있다. 이렇게 프로세스의 진행이 다른 프로세스로 인해 방해받는 현상을 경직된 동기화(lockstep synchronization)라고 한다. 따라서 위 예시는 progress를 보장하지 못한다.
