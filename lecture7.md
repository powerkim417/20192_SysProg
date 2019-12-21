# Lecture 7. Kernel Synchronization

- 커널 프로그래밍: 본질적으로 "메모리 공유 모델"이 되도록!
  - Critical Region의 존재
    - 각 프로세스에서 lock을 걸고 공유 자원을 사용하는 코드부분
  - Race condition을 피해야 함

## Kernel Control Paths

"Kernel request를 처리하기 위해 커널 모드에서 실행되는 명령어의 순서"

- Kernel request: 시스템 콜, Exception, Interrupt 

![image-20191216203745562](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216203745562.png)

위의 공간들이 모두 KCP!

### Interleaving Kernel Control Paths(KCP의 상호배치)

- CPU는 KCP를 상호배치(interleave)할 수 있음

  - 컨텍스트는 schedule() 함수가 호출될 때 전환된다.
  - CPU가 KCP를 실행하는 동안 인터럽트가 발생했을 때
    - 이 경우 첫번째 KCP는 끝나지 않은 상태로 남고, 인터럽트를 처리하기 위해 다른 KCP가 시작됨

- KCP Interleaving이 필요한 이유(중요)

  - 멀티프로세싱을 구현하기 위해
  - PIC와 디바이스 컨트롤러의 처리량 향상

- 고려해야 할 사항

  - KCP 상호배치를 하는 동안 커널 자료구조에 접근하는 경우를 고려해야 함
  - 즉, race condition을 방지하기 위해 Kernel Synchronization이 필요!
    - Race condition: <u>일부 계산의 결과가</u> 두개 이상의 상호배치된 <u>KCP의 중첩 방식에 따라 달라질 때</u> 발생!
      - 예시: 인터럽트 핸들러와 시스템 콜로부터의 동기화되지 않은 접근

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216211225209.png" alt="image-20191216211225209" style="zoom: 80%;" />

  1. [시스템 콜] int 0x80 실행(시스템 콜 초기화를 위한 exception)
  2. KCP가 critical region에 들어간다.
  3. [인터럽트] 인터럽트 핸들러가 P1의 실행을 중단하고 critical region을 실행(race condition)
  4. 프로세스 P1이 critical region의 실행으로부터 복귀한다.

- Interleaving으로 인해 리눅스 커널은 race condition이 발생할 여지가 많다!

  - 커널은 인터럽트, SMP(대칭 멀티프로세싱), 선점으로부터 안전해야 함
  - **Interrupt-safe**: 인터럽트 핸들러의 동시성으로부터의 보호
    - 인터럽트는 거의 언제든지 비동기적으로(병렬) 발생하여 현재 실행중인 코드를 인터럽트할 수 있음
    - 커널은 거의 언제든지 softirq나 tasklet을  raise하여 현재 실행중인 코드를 인터럽트할 수 있음
  - **SMP-safe**: 멀티프로세서의 동시성으로부터의 보호
    - 커널 코드는 여러 CPU에서 동시에 실행될 수 있음
  - **Preemption-safe**: 커널 선점의 동시성으로부터의 보호
    - KCP를 실행하는 프로세스는 sleep하여 새로운 프로세스가 스케줄되게 할 수 있다.
      - 커널이 선점적이므로... 커널의 한 task가 다른 커널을 선점할 수 있음 

## Kernel Synchronization Primitives

- Kernel Synchronization
  - 안전하지 않은 동시성을 방지하여 race condition이 발생하지 않는 것을 보장하는 것
- Synchronization Primitive
  - 공유 데이터간의 race condition을 피하면서 KCP를 동시배치할 수 있는 커널 자료구조를 보호하는 메커니즘

---

### 1. Atomic Operations

- 칩 수준에서 "atomic"한 연산을 보장!

  - read-modify-write 과정을 1개의 명령어로 처리하며, 처리 중 인터럽트되지 않음

- 인텔 x86 명령어

  - 메모리 접근이 0회 또는 1회인 명령어가 "atomic"
  - read-modify-write 명령어(inc 또는 dec)
    - read와 write 사이에 다른 프로세스가 memory bus를 가져가지 않았을 때 "atomic"
  - opcode에서 lock byte prefix가 있는 명령어
    - 멀티프로세서 시스템에서 "atomic"
    - "lock" 명령어 prefix: 해당 명령어에 대해 memory bus를 잠금

- Atomic operation의 사용법

  - Critical section을 보호하기 위해 더 유연하고 강력한 동기화 메커니즘을 구현

- C에서의 Atomic operation

  - C 코드를 작성할 때, 컴파일러가 단일 atomic 명령어를 사용한다고 보장할 수 없음
    - a = a + 1 같은 경우 더하기할 때 synchronization이 보장되지 못함
  - 리눅스 커널이 제공하는 특별한 함수
    - 멀티프로세서 시스템에서, 각 명령어는 lock byte에 의해 prefix됨
    - 이 함수들을 사용하면 synchronization이 보장됨
  - 정수에서의 atomic operation

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216233036765.png" alt="image-20191216233036765" style="zoom:50%;" />

  - bit operation: 메모리 주소에서 비트를 변경하는 연산

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216233049766.png" alt="image-20191216233049766" style="zoom:50%;" />

### 2. Barriers

- 현대의 컴파일러와 프로세서는
  - 성능적 이유로 read/write 작업의 순서를 재구성한다
  - compile time 또는 execution time 동안..!
  - (그러므로 atomic operation을 쓸 수 없음)
- Compiler barrier( = Optimization barrier)
  - <u>컴파일러</u>가 배리어를 넘어서 store와 load의 순서를 바꾸는 것을 방지
  - barrier()
- Memory barrier
  - 특정 지점에서 <u>프로세서</u>가 명령어의 순서를 재구성하는 것을 방지
  - rmb()/wmb(): 각각 read와 write을 방지 
  - mb(): read와 write 둘다 방지

---

### 3. Locking

- Kernel locking(일반적 메커니즘)
  - (atomic operation은 길고 복잡한 critical region에서의 공유 데이터 보호에 적합하지 않음)
  - KCP가 공유 자료구조에 접근하거나, critical region에 들어가려 할 때, "lock"을 얻어야 하는 것!
- Linux에서는 2종류의 locking을 지원
  - Spin lock: busy-waiting(non-blocking) lock
    - busy-waiting: 프로세스가 계속 lock을 향해 접근.. lock이 풀리면 바로 점유
    - 멀티프로세서 시스템에서만 사용
  - Kernel semaphore: blocking lock
    - blocking lock: 프로세스가 lock을 향해 한번 접근하고, 잠겨있으면 block되고 끝.
    - 유니프로세서와 멀티프로세서 시스템 모두에서 널리 쓰임
- Lock은 알아서 구현해야 한다!
  - 프로그래머가 락 없이 공유 자료구조에 접근하는 것이 가능한데, 이렇게 접근하면 race condition이 발생하거나 공유 데이터가 깨질 수 있음..
- Lock을 구현할 때 체크해야 할 것(커널 코드 작성시)
  - 데이터가 전역변수인가? 현재 실행중이 아닌 스레드가 접근할 수 있는가?
  - 프로세스 컨텍스트와 인터럽트 컨텍스트 사이에서 공유하는 데이터인가?
  - 두 다른 인터럽트 핸들러 사이에서 공유하는 데이터인가?
  - 현재 프로세스가 block될 수 있는가? 그렇다면 공유 데이터는 어떤 상태로 남는가?
  - 프로세스가 공유데이터 접근 중 선점된다면 새로 스케줄된 프로세스가 같은 데이터에 접근할 수 있는가?

#### Lock Contention and Scalability

- Lock contention

  - 락이 현재 사용중일 때 다른 프로세스가 이를 얻으려고 하는 것
  - Lock은 작업을 serialize(직렬화)함!
  - Highly contended lock은 많은 프로세스가 동시에 락을 얻으려고 하는 락인데, 이 지점이 병목지점이 되어 시스템의 성능을 제한함

- 확장성

  - 시스템이 얼마나 확장될 수 있는지(프로세서 수, 프로세스 수, 메모리 크기, ...)
  - Highly contended lock은 확장성을 제한함
    - 모든 프로세스가 들어가서 작업을 처리하게 되므로 낮은 수준의 병렬작업이 진행됨
  - SMP 이후 많은 프로세서에서 리눅스의 확장성이 크게 향상됨

- Lock 정밀도

  - Lock이 보호하는 데이터의 크기를 표현

- Coarse lock

  - 서브시스템 전체와 같이 큰 양의 데이터를 보호
  - 이러한 lock은 highly contended될 가능성이 높음

- Fine-grained lock

  - 큰 구조에서의 하나의 요소와 같이 작은 크기의 데이터만 보호

- Lock 튜닝

  - lock은 coarse한 상태에서 시작되어 lock contention이 문제가 될 경우 fine-grained한 방식으로 수정된다.

    ![image-20191217002837642](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217002837642.png)

    - 시스템의 모든 runqueue를 하나의 lock으로 -----> 한 priority list에 한 lock

> Non-blocking lock

#### 3-1. Spin Locks

멀티프로세서 시스템에서 사용하는 locking 메커니즘

- 작동 메커니즘
  - 공유 변수를 set하여 lock을 얻음
  - 해당 변수가 unset될 때까지 busy-wait loop이 돌아감
- 아키텍처에 종속적이고, 어셈블리어로 구현됨
- Lock이 커널 선점을 막음
- 인터럽트 컨텍스트에서 사용할 수 있음
  - 커널은 인터럽트 컨텍스트에서 공유할 수 있는 자료구조를 위한 특별한 spinlock API를 제공한다.
  - 프로세스 컨텍스트에서, 선점이 비활성화 되어있으므로 스핀락을 갖고있는 동안은 sleep되지 않음
- **유니프로세서 시스템에서 스핀락을 사용하지 않는 이유**
  - **어차피 동시에 돌아가는 프로세스가 없고,**
  - **대기중인 프로세스가 계속 spin되고, lock에 있는 프로세스가 lock을 풀지 않으므로 쓸모 X**
- 매크로 및 사용 예시

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217011300663.png" alt="image-20191217011300663" style="zoom:50%;" />

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217011322359.png" alt="image-20191217011322359" style="zoom:67%;" />

#### 3-2. Read/Write Spin Locks

"여러 reader를 허용하지만 writer는 한명"

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217011637598.png" alt="image-20191217011637598" style="zoom: 67%;" />

- 동시에 여러 reader를 허용하므로 확장성 향상
- 데이터 타입: include/linux/rwlock_t
- RWLock에서의 작업
  - DEFINE_RWLOCK(my_lock)
  - void read_lock(rwlock_t *rw), void read_unlock(rwlock_t *rw)
  - void write_lock(rwlock_t *rw), void write_unlock(rwlock_t *rw)
  - 등등

##### Spin Locks in Interrupt Handlers

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217015222065.png" alt="image-20191217015222065" style="zoom: 67%;" />

- 스핀락은 인터럽트 핸들러에서 사용될 수 있음(SMP): 스핀락은 block하지 않으므로!!

- 인터럽트 핸들러가 lock을 사용하면,

  - **락을 얻기 전에 "로컬 CPU의 인터럽트 비활성화"를 해야 함!!**

    - **인터럽트가 발생하면 nested되어 lock이 꼬임**

  - 다른 CPU의 인터럽트는 비활성화할 필요 X

    - 스핀 락은 로컬 CPU에서만 돌아가므로!

  - Local interrupt를 끄지 않을 때 데드락이 발생하는 예제

    ![image-20191217015356872](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217015356872.png)

    1. 인터럽트 핸들러에서 lock 획득
    2. 인터럽트가 새로 들어와서 nested
    3. nested 인터럽트 핸들러에서 lock 획득하려고 함
    4. **그런데 lock은 이미 첫번째 인터럽트에서 획득되었고, 이는 nested가 끝나지 전까지 풀 수 가 없으므로**
    5. DEADLOCK

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217022447432.png" alt="image-20191217022447432" style="zoom: 50%;" />

> Blocking lock

#### 3-3. Kernel Semaphores

- "Sleeping locks"
  1. 프로세스가 이미 잡힌 세마포어를 얻으려 하는 경우, 세마포어는 프로세스를 wait queue에 넣는다(그리고 block한다)
  2. 다른 프로세스가 스케줄됨
  3. 세마포어가 풀리고, 세마포어의 wait queue에 있는 프로세스가 깨어남

  **짧게 묶이는 lock에는 부적합(오버헤드)**

- 잠재적으로 block함

  - 세마포어는 sleep이 허용된 함수를 통해서만 얻을 수 있음
  - 그러므로 **인터럽트 컨텍스트에서는 사용할 수 없음**

- 여러개를 홀드할 수 있음

##### Using Kernel Semaphores

![image-20191217020513922](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217020513922.png)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217020922171.png" alt="image-20191217020922171" style="zoom: 67%;" />

- 초기화 시: sema_init(struct semaphore *, int count)

  - count==1이면 MUTEX(1개 수용)
  - count>1이면 counting semaphore(여러개 수용)

- 리소스 사용: down(struct semaphore * sem)

  - sem->count를 자동으로 감소
    - -1보다도 더 아래로 떨어질 수 있음. 그만큼 대기자가 많다는것
  - 감소 후 count<0인 경우 current를 wait queue에 넣음

- 리소스 해제: up(struct semaphore * sem)

- sem->count를 자동으로 증가

- 증가 후 count<=0인 경우 wait queue의 task 하나를 깨운다.

- MUTEX(프로세스 1개만 가능)

  ![image-20191217021024107](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217021024107.png)

  - P1이 먼저 사용하고, P2는 사용하려고 했더니 없어서 대기. cnt는 -1
  - P1이 다 사용하면 cnt는 0, wait queue의 P2를 깨워서 사용시킴.

- Counting Semaphore(프로세스 2개 이상 가능)

  ![image-20191217021043100](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217021043100.png)

  - P1, P2가 사용하고, P3는 대기 cnt = -1
  - P2가 다 써서 cnt = 0, wait queue의 P3를 깨워서 사용시킴.

#### 3-4. Read/Write Semaphores

![image-20191217022407648](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217022407648.png)

![image-20191217022417135](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217022417135.png)



- RW 스핀락처럼 세마포어의 RW 버전
- reader는 아니고, **writer에 한해서만 상호 배제 강요!**
- down_read(), up_read(), down_write(), up_write()

#### 3-5. Mutexes

![image-20191217022357365](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217022357365.png)

- 세마포어보다 더 간단한 sleeping lock
  - 바이너리 세마포어와 비슷하지만, 더 간단한 인터페이스와 더 효율적인 성능, 그리고 추가적인 제약조건이 있다
  - (바이너리 세마포어보다) 엄격하고 제한적인 사용 조건
    - 한번에 하나의 스레드만 뮤텍스를 사용할 수 있음
    - 뮤텍스를 lock한 스레드만이 unlock할 수 있음
    - 재귀 lock X
    - 뮤텍스를 잡고 있는 스레드는 종료될 수 없음
    - 인터럽트 컨텍스트에서는 뮤텍스를 얻을 수 없음
    - 뮤텍스는 공식적인 API를 통해서만 관리 가능

#### 3-6. Completions

- 세마포어의 특수 케이스
  - 서로 다른 CPU에서 up()과 down()이 동시에 실행될 수 있는 경우 멀티프로세서 시스템에서 세마포어를 사용할 때 발생할 수 있는 미묘한 race condition 해결
  - **커널에서 두 task를 동기화시킬 수 있는 쉬운 방법**
- 작동
  1. 다른 프로세스들이 프로세스 P가 작업이 끝나기를 기다린다.
  2. P가 작업을 끝내면, P가 기다리는 프로세스들을 깨운다.
  3. Completion 변수가 정적으로 선언: DECLARE_COMPLETION(cmp);
  4. 이 변수 호출에 대기하고 싶은 프로세스(자원 할당을 하고싶어하는 프로세스)의 경우
     - wait_for_completion(&cmp); // down()과 비슷
  5. P가 complete(&cmp)를 호출해 completion을 알림 // up()과 비슷

#### 세마포어에서 데드락을 피하는 방법

- 데드락
  - 프로그램이 2개 이상의 세마포어를 쓸 때
  - 두 다른 경로가 서로의 세마포어의 해제를 기다릴 수 있음
  - 각 KCP가 한번에 1개의 세마포어만 획득하면 되므로 이런 문제가 발생???
- **데드락을 피하는 방법**
  - **Lock ordering: 2개 이상의 세마포어가 정해진 순서대로 요청을 받음**

#### 3-7. BKL

Global Kernel Lock

- "커널모드에서 한번에 한 CPU만 실행된다" 보장
- lock_kernel() / unlock_kernel(): BKL 획득 / 해제
- 일반 세마포어보다 조금 더 복잡한 구조.. 흥미로운 특징들이 있다
- 그러나 쓰는건 비추!!!(구식 컨셉)
  - 커널이 점점 더 복잡해지고 유연해지므로 하나의 스핀락에 의존할 수 없음
  - 그러므로 새 코드는 BKL을 쓰면 안됨

#### Spin Locks vs. Semaphores (Non-blocking lock vs. Blocking lock)

- 세마포어를 얻으려 할 때 프로세스는 sleep될 수 있음
  - 그러므로 인터럽트 컨텍스트(IH, softirq, tasklet)에서는 안됨
  - 세마포어는 프로세스 컨텍스트(시스템 콜, workqueue)에서만 사용 가능
- 프로세스는 사용 가능할 때까지 sleep하면서 세마포어를 기다림
  - 오래동안 잡아두는 락이 적절!!(스핀락은 CPU 사이클을 낭비)
- 짧게 잡아두는 경우 스핀락은 부적절
  - wait queue, 스케줄링, 깨우기에 대한 오버헤드가 많이 나옴
- 스핀락과 달리, 세마포어를 잡는 것은 커널 선점을 비활성화하지 않음
  - **세마포어를 잡고 있는 프로세스는 선점 가능**
  - 스케줄링 지연에 영향 X
- 카운팅 세마포어는 동시에 몇명의 락 홀더를 허용할지를 정할 수 있음
  - 스핀락은 최대 1개 프로세스
- **스핀락을 써야 하는 경우: 오버헤드가 적을 때, 홀드 시간이 짧을 때, 인터럽트에서 락을 걸어야 할 때**
- **뮤텍스를 써야 하는 경우: 홀드 시간이 길 때, 락을 잡는동안 sleep해야 할 때**

---

### 4. Read-Copy-Update(RCU)

"동시에 read 접근과 함께 write 업데이트"

- writer가 reader를 block하지 않음(lock-free)

- 거의 0에 가까운 오버헤드로 여러 reader를 허용

- reader 성능 최적화(RWLock보다)

- 핵심 아이디어

  - 공유 자료구조에 대한 포인터의 모든 사용자를 트래킹
  - 구조가 변경되는 경우 copy를 먼저 만들고 그 copy에 변경사항이 반영됨
  - 이전 reader들이 이전 copy에서 reading 작업을 끝내고 나면 포인터가 새로운 copy를 가리키는 포인터로 교체될 수 있음

- RCU 제약 조건

  - 커널은 RCU에 의해 보호받는 영역에서 sleep될 수 없음
  - 보호되는 자원은 포인터를 통해서만 접근 가능

- Read 작업이 주인 자료구조에서 주로 사용

  - 디렉토리 엔트리 캐시, 네트워크 레이어, 가상 파일시스템, ...

- RCU 작업

  - rcu_read_lock(), rcu_read_unlock(), rcu_deference(), rcu_assign_pointer(), synchronize_rcu(), call_rcu(), …

    ![image-20191217043926385](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217043926385.png)

---

### 5. Preemption Disabling(선점 비활성화)

- 스핀 락과 함께 동기화 진행하는 경우
  - 스핀 락이 걸려있는 경우, 커널은 선점 불가능
- 스핀 락 없이 단독으로 동기화 진행
  - 선점 비활성화가 필요한 경우가 있음
  - ![image-20191217042717482](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217042717482.png)
- 커널 선점 관련 함수

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217042756857.png" alt="image-20191217042756857" style="zoom:80%;" />

### 6. Local Interrupt Disabling

- 로컬 CPU의 모든 인터럽트를 비활성화

  - 인터럽트 핸들러가 현재 코드를 선점할 수 있으므로 일련의 커널 statement가 critical section으로 작동한다고 보장 
  - 인터럽트 비활성화는 곧 커널선점 비활성화도 된다
  - local_irq_disable(), local_irq_enable() 매크로 사용
    - 각각 eflags 제어레지스터의 IF 인터럽트 플래그를 clear/set
  - 시스템이 멈출 수 있기 때문에 **커널은 인터럽트가 비활성화되어있을 때 blocking 작업을 실행하면 안됨!**

- 멀티프로세서 시스템

  - 로컬 인터럽트 비활성화는 다른 CPU에 영향이 없으므로 다른 프로세스의 동시 접근으로부터 보호되지 않음
  - 락(스핀락)은 로컬 인터럽트 비활성화와 함께 사용됨

- 인터럽트는 중첩해서 실행될 수 있음

  - 커널은 인터럽트 비활성화 전에 eflags 레지스터의 정보를 저장하고, 재활성화 후 복구해야 함
  - local_irq_save(): eflags 레지스터의 정보를 로컬 변수로 복사하고 IF 플래그 clear
  - local_irq_restore(): eflags 레지스터의 원래 정보 복구

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217050746385.png" alt="image-20191217050746385" style="zoom:50%;" />

---

### 어떤 Synchronizaton Primitive를 사용해야 할까?

- 정수 하나를 포함하는 공유 자료구조
  - atomic_t 자료형으로 선언하고 **atomic operation** 사용
- KCP에 의해 접근하는 자료구조에 대한 보호가 필요할 때
  - **Exception: Semaphore(UP)**
    - 주로 시스템 콜 서비스 루틴을 통해 접근
    - 이처럼 exception에 의해서만 접근되는 자료구조는 1개 이상의 프로세스에 할당될 수 있는 자원을 상징
  - **Interrupt: Local Interrupt Disabling(UP) / Spin Lock(MP)**
    - 각 인터럽트 핸들러는 직렬화되어 있으므로 synchronization이 따로 필요 없음???
    - 여러 인터럽트 핸들러로부터의 접근
      - nested handler 또는 멀티프로세서에서 여러 핸들러가 동시에 실행
  - **Deferrable function: None or Spin Lock(MP)**
    - UP에서는 race condition 불가(지연 함수가 CPU에서 직렬화되어있으므로)
    - MP에선 race condition이 존재(몇몇 지연함수가 동시에 실행될 수 있으므로)
    - softirq: Spin lock(같은 softirq가 다른 CPU에서 동시에 실행될 수 있음)
    - 1개의 tasklet: None(같은 종류의 tasklet은 동시에 실행될 수 없음)
    - 여러개의 tasklet: Spin lock(다른 종류의 tasklet은 동시에 실행될 수 있음)
  - **E + I: Local Interrupt Disabling(UP) / Spin Lock(MP)**
  - **E + D: Local Softirq Disabling(UP) / Spin Lock(MP)**
  - **I + D: Local Interrupt Disabling(UP) / Spin Lock(MP)**
  - **E + I + D: Local Interrupt Disabling(UP) / Spin Lock(MP)**

### Summary

![image-20191217053123929](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217053123929.png)

- 일반적 사용
  - 세마포어는 인터럽트 컨텍스트에서 사용될 수 없음
  - UP: 인터럽트 비활성화
  - SMP: 스핀락 + 인터럽트 비활성화

