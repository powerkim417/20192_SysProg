# Lecture 1. Linux Processes

## Dual Mode Operation

현대 CPU는 최소 2종류의 작업 모드(유저 모드, 커널 모드, ...)를 구별할 수 있는 하드웨어를 지원 제공

- Intel x86은 4종류의 실행 상태가 있지만, 모든 표준 UNIX 커널은 유저모드(level 3)와 커널모드(level 0)만 사용함.
- 일반적으로 모든 표준 UNIX 커널은 유저 모드와 커널 모드만을 사용함.

### User mode

- 유저에 의해 실행되고, 권한이 적음(less-privileged mode)
- 유저 프로그램은 일반적으로 이 모드에서 실행됨

### Kernel mode

- = Supervisor mode
- OS에 의해 실행되고, 권한이 많음(more-privileged mode)
- 운영체제의 커널은 프로세서와 프로세서의 모든 명령어, 레지스터, 메모리에 대해 완벽한 제어권을 가지고 있음

프로세스의 생성/종료는 Kernel mode에서 진행된다.

### Frequent mode changes

모드가 전환되는 3가지 요인

1. Hardware interrupt
2. Software interrupt (exception)
3. System call
   - 유저모드에서 system call이 발생한 경우 커널모드로 이동하고, 수행이 완료되면 다시 유저모드로 돌아옴

## Process

### Running OS (or kernel code)

- 커널 코드는 일반적인 컴퓨터 소프트웨어와 같은 방식으로 작동함 -> 프로세서에 의해 실행
- 커널 코드는 주로 제어권을 프로세서에게 넘기고 의존함으로서 OS에 대한 제어권을 복원한다.(커널 모드 코드)
- 커널 코드가 프로세스인가? 맞다면 어떻게 제어하는가?

### Process Virtual Address (Image)

하나의 프로세스가 생성되면 그 프로세스에 해당하는 가상 주소 공간이 할당됨

- 일반적인 Process Virtual Address의 모습

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1569613796794.png" alt="1569613796794" style="zoom: 67%;" />

- Linux Image(x86)의 모습

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1569613949525.png" alt="1569613949525" style="zoom:67%;" />

- 4GB의 크기, 이중 마지막 1GB가 Kernel Address Space로 할당됨.
  - 이 중 Kernel(code, data), Physical memory는 각 프로세스에 대해 같고
  - Process-specific data structures는 프로세스마다 각각 다르다.
    - Page tables / kernel stack / task / mm structs
- 유저 프로세스에서의 실행
  - 사실상 모든 OS 코드(커널 코드)는 <u>유저 프로세스의 context 내에서 실행됨</u>
    - 커널은 유저 프로세스의 일부처럼 보임. 즉, 커널 코드는 프로그램 이미지와 사실상 연결되어 있음
  - 인터럽트, system call, trap에서 CPU가 커널 모드로 전환되어 유저 프로세스의 context 내에서 OS 루틴을 실행한다.
    - **Full process switch가 일어나는 것이 아니고, 같은 프로세스에 대해서 mode switch만 일어남 (2회의 process switch에 대한 오버헤드 절약)???**
  - Full process switch는 필요할 때만 발생하도록 하는데, 이는 매우 큰 이점!
- Multiprocessing: 여러개의 Process image에서 Kernel Space 부분들이 하나의 큰 Kernel Segment를 통해 공유됨

![image-20191028040740070](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028040740070.png)

### Process란 무엇인가?

- 프로세스: 실행되는 프로그램의 인스턴스
- 프로세스 = image + process context
  - Image
    - Code: 기계어 명령어
    - Data: 변수
    - Stack: 함수 호출을 위한 상태 저장
    - Heap: 동적 메모리
  - Process Context
    - Program context
      - 데이터 레지스터, 프로그램 카운터(PC), 스택 포인터(SP)
    - Kernel context
      - pid, gid, sid, environment
      - VM 구조(page table)
      - 파일 오픈
      - 신호 핸들러, ...

Process = process context +  (code, data, stack)

#### Process Model의 한계

- 프로세스 협력
  - 많은 어플리케이션은 자신의 copy를 fork하여 여러 task를 동시에 수행한다
    - Ex) 웹 서버
  - 이 경우 주소공간과 자원은 공유될 수 있음
    - 즉, 모델은 비효율적이다???
- 멀티프로세싱
  - 전통적인 프로세스는 멀티프로세서 구조의 장점을 수용할 수 없다. 프로세스는 한번에 한 프로세서만 사용할 수 있기 때문
    - 자동 병렬화 불가
    - 명시적으로 프로세스를 생성하고, 적합한 프로세서에 연결???

#### Process with Multiple Threads

- 한 프로세스에 여러 개의 스레드가 있을 수 있음

  - 각 스레드는 자신 고유의 logical control flow가 있음(PC값의 sequence)
  - 각 스레드는 같은 코드, 데이터, kernel context를 공유한다
  - 각 스레드는 고유의 스레드 id가 있음(tid)

  ![image-20191029201707855](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029201707855.png)

### Process VS Thread

- 프로세스 주소 공간

  ![image-20191029201924309](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029201924309.png)

  - 스레드가 여러개가 되면?
    - stack이 여러개로 나누어져 스레드 각각의 SP가 따로 생김
    - 코드 내에서 각 스레드에 대한 PC가 생김

- 프로세스는 계층 구조인 반면, 스레드는 peer pool 구조로 이루어져 있음

### Pthreads

- 스레드 생성과 동기화를 위한 POSIX 표준 API
- API는 스레드 라이브러리의 동작을 지정하며, 구현은 라이브러리의 개발에 달려 있음
  - 대부분 유저 레벨 스레드; 일부는 커널 스레드의 도움을 받는다
  - pthread_create(), pthread_exit(), pthread_join(), ...
- UNIX 운영체제에서 자주 쓰임
  - 요즘 대부분의 멀티스레드 어플리케이션은 pthread 라이브러리를 사용하여 개발된다!

#### Pthreads API

- 스레드를 생성할 때: pthread_create()
- 스레드를 거둘 때: pthread_join()
- 스레드 id를 결정: pthread_self()
- 스레드 제거: pthread_cancel(), pthread_exit()
- 공유 변수에 접근 동기화시 사용: pthread_mutex_init(), pthread_mutex\_[un]lock(), pthread_cond_init(), pthread_cond_[timed]wait()

### Process State

#### 일반적인 모델

new, ready, running, blocked, exit

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029204616276.png" alt="image-20191029204616276" style="zoom:80%;" />

- new: 프로세스가 생성됨
- ready: 프로세스가 프로세서에게 할당되기를 기다림
- running: 명령어가 실행중인 상태
- blocked: 프로세스가 특정 이벤트(I/O 완료)의 발생을 기다림
- exit: 프로세스가 실행을 마침

#### Linux Process State

![image-20191029204716879](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029204716879.png)

- 이미 존재하는 task가 fork()를 호출하여 새 프로세스를 생성. fork된 프로세스는 'ready'로 간다.
- TASK_RUNNING: 실행중(running) 또는 실행을 기다리는 중(ready)
  - schedule()에 의해 context_switch()가 호출되면 스케줄러가 해당 task를 CPU에 dispatch하고, 'running'으로 간다
  - task가 상위 우선순위 task에 의해 선점되면 'ready'로 간다.
- TASK_UNINTERRUPTIBLE, TASK_INTERRUPTIBLE: 특정 이벤트 발생때까지 block되며, UNINTERRUPTIBLE의 경우 말그대로 interrupt되지 않는다. (하드웨어 조건만을 기다리며, signal은 받지 않음) 재개될 때는 'ready'로 간다.
- TASK_STOPPED: 디버깅할 때 멈추고, 재개될 때는 'ready'로 간다.
- EXIT_ZOMBIE: do_exit()을 통해 task 종료

### Linux Process Descriptor

- 프로세스 정보를 담는 자료구조(PCB, process control block)
  - 프로세스가 소유한 자원을 참조(linux에서는 task)
    - Task info, Task image, Program context
  - struct task_struct 타입으로 정의
  - struct thread_info ***thread_info**: task info
  - struct mm_struct ***mm**: task image
  - struct thread_struct **thread**: program context
- 프로세스 식별
  - 각 프로세스는 자신의 process descriptor를 가지고 있음
    - 독립적으로 스케줄될 수 있는 각 실행 컨텍스트는 모두 자신의 process descriptor를 가지고 있어야 함(프로세스, 스레드)
    - 경량 프로세스조차도 task_struct 구조를 가지고 있음
    - Process descriptor pointer
      - 커널이 프로세스를 식별하는데 유용
        - 커널이 만드는 프로세스에 대한 대부분의 참조는 이 포인터에 의해 이루어짐 
  - Process id(PID)
    - 16비트 하드웨어 플랫폼에서 0~32767
    - 커널이 32768번째 프로세스를 생성하려 하면 가장 낮은 unused PID부터 재활용한다

### User Stack and Kernel Stack

- dual mode operation
  - 2가지 모드에 대해 각각의 별도의 스택 존재
- Kernel mode stack로의 전환
  - 커널로 진입할 때 커널도 전환되어야 함
  - 프로그램의 안정성을 위해 필요: 유저 모드 스택이 꽉 찰 수 있으므로
  - 보안을 위해서도 필요

#### User Mode Stack

- 위치: user space의 가장 높은 곳에서부터 아래로 자람(0xBFFFFFFF)
- main()에 들어가는 동안 user stack에 저장되는 것
  - 프로그램의 종료 주소
  - command line argument
  - 환경변수
- main() 실행되는 동안 user stack에 저장되는 것
  - 함수 파라미터와 반환주소
  - 로컬변수 저장소의 위치

#### 현재 프로세스 식별

- thread_union은 thread_info와 stack을 묶고 있음
- 따라서 커널은 esp 레지스터 값에서 현재 실행중인 CPU의 스레드 정보 구조의 주소를 쉽게 얻을 수 있다.???

#### Kernel Mode Stack

- main()에 들어가는 동안 kernel stack에 저장되는 것
  - task의 레지스터
- main() 실행되는 동안 kernel stack에 저장되는 것
  - 함수 파라미터와 반환주소
  - 로컬변수 저장소의 위치
- Linux는 task의 thread_info 구조를 저장하기 위해 task의 kernel mode stack의 일부를 사용한다.

### Process List

- 요구사항
  - 커널은 주어진 타입의 프로세스를 통해 효율적으로 탐색할 수 있도록 몇 개의 프로세스 리스트를 갖고 있음
    - Ex) 모든 runnable state의 프로세스에 접근
  - 존재하는 모든 process descriptor는 doubly linked list로 연결되어 있다 -> process list
- init_task
  - 모든 프로세스의 조상, process 0 이라고 부름(swapper)
- 매크로
  - SET_LINKS, REMOVE_LINKS: 삽입/삭제
  - for_each_task 매크로: 모든 프로세스 리스트를 스캔
- TASK_RUNNING 프로세스 관리
  - 모든 runnable process를 찾기 위해 process list를 스캔하는 것은 비효율적!
  - 각 priority에 대해 runnable process에 대한 doubly linked list
    - struct list_head[140] queue: 모든 priority list의 head
  - process descriptor끼리는 task_struct의 run_list 필드를 통해 연결되어 있다
  - 연관 함수
    - enqueue_task(p, array): p.d를 run queue에 삽입, dequeue_task(p, array)

![image-20191029225653234](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029225653234.png)

#### pidhash Table

- 요구사항
  - 프로세스 리스트를 순차적으로 스캔하고 p.d의 PID 필드를 확인하는 것은 비효율적
  - 이를 효율화하기 위해 hash table을 사용!
- pidhash
  - PID를 사용하여 p.d를 탐색하는 hash table
  - PID 충돌은 chaining으로 처리

### Process Organization

- 프로세스가 어떻게 조직되는가?
  - runqueue 리스트들은 TASK_RUNNING 상태의 모든 프로세스를 모은다
  - TASK_STOPPED / TASK_ZOMBIE 상태 프로세스는 특정 리스트에 들어가지 않음
  - TASK_[UN]INTERRUPTIBLE은 여러 class로 나뉘며, 각각 특정 이벤트를 기다린다???
    - 프로세스를 쉽게 얻으려면 wait queue라는 process list 사용!

#### wait queue

- 원리
  - 프로세스들은 특정 이벤트의 발생을 기다림
  - 이렇게 block 상태인 프로세스는 자신을 해당 wait queue로 보내고 제어를 넘긴다
  - <u>Wait queue는 sleep중인 프로세스의 집합</u>이 되는 것!
    - 특정 조건이 만족되면 커널에 의해 깨워진다
      - 독점 프로세스는 선택적으로 커널에 의해 깨어난다
      - 비독점 프로세스는 해당 이벤트가 발생할 때 깨어난다
  - 커널은 wait queue의 프로세스들을 TASK_RUNNING 상태로 바꿈으로서 깨운다

### Process Switching

- 커널이 프로세스 전환을 허용하는 경우
  - 프로세스가 자신을 sleep 상태로 돌릴 때
  - 프로세스가 종료될 때
  - 프로세스가 system call으로부터 user mode로 돌아왔지만 실행하기 가장 적합한 프로세스가 아닐 때
  - 커널이 인터럽트를 처리하여 프로세스가 유저모드로 돌아왔지만 실행하기 가장 적합한 프로세스가 아닐 때
- 프로세스 전환 단계
  1. **언제** context switch를 할지 정하고, context switch가 허용되는지 확인
  2. **old process**의 context를 저장
  3. 프로세스 스케줄링 알고리즘을 통해 실행을 위해 스케줄되기 **가장 적합한 프로세스를 찾는다**
  4. 해당 프로세스의 context를 **복구**
- Linux에서는 process switching = task switching = context switching
  - 현재 실행중인 프로세스의 실행을 미루고, 다른 미뤄진 프로세스를 실행
- prev 에서 next로 전환
  1. 모든 prev의 user mode 레지스터값을 prev의 커널 스택에 저장
  2. prev의 **주소공간**을 next의 주소공간으로 전환
  3. <u>prev의 **커널 모드 스택**을 next의 커널 모드 스택으로 전환</u>
  4. <u>prev의 **하드웨어 컨텍스트**을 next의 하드웨어 컨텍스트로 전환</u>

#### Hardware context

- 필요성
  - 각 프로세스가 고유 주소공간이 있지만, CPU 레지스터는 공유해야 함
  - 커널은 이 레지스터가 '프로세스가 중단되었을 때 가졌던 값'으로 로드되는지를 확인해야 한다
- Hardware Context
  - 프로세스가 실행으로 돌아가기 전에 데이터가 CPU 레지스터에 로드되어야 함
  - 프로세스 실행에 필요한 모든 정보 포함
  - 프로세스의 하드웨어 컨텍스트의 일부는 p.d에 포함되어 있고(struct thread_struct thread), 나머지 부분은 커널 모드 스택에 저장되어 있다

#### thread_struct 구조

- 각 process descriptor에 struct thread_struct thread라는 필드가 있는데, 프로세스가 CPU로부터 제거되었을 때 커널이 이 곳에 하드웨어 컨텍스트를 저장한다.
- eip, esp 존재

#### Process Switch 구현

- 프로세스 전환이 되는 순간: schedule() 함수가 호출되었을 때!
- 프로세스 전환의 단계
  1. 새 주소공간을 설치하기 위해 Page Global Directory를 전환
  2. 커널 모드 스택 전환, 하드웨어 컨텍스트 전환(switch_to 매크로)

##### switch_to 매크로

- schedule() 함수에 의해 호출되어 프로세스 전환을 실행
- 커널에서 가장 하드웨어에 종속된 루틴

![image-20191030024912516](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030024912516.png)

- 빨간색 프로세스에서 파란색 프로세스로 전환되는 것
- ESP: 프로세스의 stack pointer를 가리킴
- EIP: 프로세스의 code를 가리킴

#### Process Switching 단계(switch_to())

![image-20191030030349625](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030030349625.png)

kernel code는 위와 같이 되어있을 때

1. esp는 prev의 스택, eip는 프로세스 전환 코드의 시작점
2. prev의 프로세서 플래그와 ebp 레지스터의 값을 prev의 커널모드 스택에 저장한다.
   - switch_to가 끝날 때까지 바뀌지 않게 하기 위해
3. prev의 thread 필드(program context)의 esp가 'esp가 가리키는 값''을 가리키도록 한다
4. esp가 next의 thread의 esp를 가리키도록 한다
   - 이제부터 커널은 next의 커널모드 스택에 대한 작업을 진행하게 됨
   - **이 부분이 실질적 프로세스 전환**
   - <u>esp 바뀜</u>
5. prev의 thread의 eip가 L1을 가리키게 한다
   - 실행으로 돌아왔을 때 L1부터 실행하게 됨(스택에서 다시 %ebp, flags 꺼내기)
6. next의 스택에 next의 thread의 eip를 저장(L1)
   - next도 예전에 전환되었을 때 thread의 eip가 L1을 가리키게 되었으므로
7. __switch_to 로 점프한다.
   - eax와 edx에서 prev, next 파라미터를 가져옴
8. prev의 thread의 나머지 hardware context 부분(그림에서 esp eip 제외)을 저장하고, next의 prev의 나머지 hardware context 부분을 불러온다(그림에서 가운데 부분)
   - **실질적 hardware context 전환**
   - <u>hardware context 나머지 부분 바뀜</u>
9. ret와 함께 종료. eip는 L1을 가리키게 됨
   - <u>eip 바뀜</u>
10. stack에서 %ebp와 flag를 꺼내어 적용
    - <u>ebp, flags 바뀜(모두 바뀜!)</u>

### Creating Processes and Threads

- 예전에는 실행과 자원소유가 혼합되었다? 프로세스는 자원을 혼자 소유하고 스레드를 하나씩만 씀
- 요즘은 실행과 자원이 분리됨!
- 여러 스레드가 한 프로세스에서 자원을 공유
- 애플리케이션 내에서 병렬화를 적용하는 것이 전통적 접근에서는 어려웠다..
  - 애플리케이션의 각 부분마다 별도의 프로세스가 생성되어야 했음
  - 별도의 프로세스는 별도의 주소공간을 가짐
    - 공유데이터 X, 프로세스간 공유정보 X -> 사용하기 어려웠음
  - 애플리케이션의 부분을 보호하기 위해 서로를 overkill함???
  - 자원의 중복 문제
- 해결책: 한 프로세스를 여러개의 스레드로 실행
  - 스레드는 주소공간, 파일 처리 등의 자원을 공유하고,
  - 스케줄링은 개별적으로 된다.

#### Thread Implementation

- User-level threads, kernel-level threads

  ![image-20191030034100071](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030034100071.png)

  - 프로세스는 항상 kernel space에 존재
  - Pure user-level: 유저 공간에서 멀티스레드 분기
  - Pure kernel-level:  커널 공간에서 멀티스레드 분기
  - Combined: 두 공간 모두에서 분기 가능

### Lightweight Processes

#### Linux에서의 kernel-level thread

- Linux Threads
  - Linus는 스레드로 리눅스 커널을 확장하는 것에 반대
  - 그 대신, 프로세스가 리소스를 공유할 수 있게 됨
  - 프로세스가 생성될 때, 부모 프로세스는 어느 리소스를 자식과 공유할 지 정할 수 있다.
    - 메모리 주소공간, 파일 오픈, 신호대기 및 신호핸들러 등
  - 리소스를 공유할 수 있는 프로세스는 가벼우므로 Lightweight Process!

#### Lightweight Linux Processes

- 스레드 그룹으로 조직화
- 그룹의 첫번째 프로세스가 스레드 그룹 리더가 됨
- 각 프로세스는 자신의 PID, process descriptor, user/kernel stack 등을 가지고 있다.
- Thread group ID(TGID): 스레드 그룹 리더의 pid

![image-20191030034641632](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030034641632.png)

- clone() 함수에 의해 생성됨
- clone() 함수의 메인 목적은 스레드 구현! 공유 메모리 속에서 동시에 실행되는 프로그램에서 여러개의 제어 스레드
- int clone(int (*fn), void *child_stack, int flags, void *arg);

  - **새로운 프로세스를 생성**하고, child_stack에서 가리키는 stack space를 사용하는 프로세스에서 **fn 함수를 arg 인자와 함께 실행**
  - 새 프로세스의 PID를 반환
  - fn
    - 새 프로세스에서 실행될 함수
    - 이 함수가 반환되면 child는 종료된다
  - arg
    - fn 함수에 들어갈 데이터에 대한 포인터
  - flags
    - 커널 컨텍스트 공유의 정도를 나타냄
    - CLONE_VM: 가상 주소공간 공유
    - CLONE_FS: 파일 시스템 정보 공유
    - CLONE_FILES: open file descriptor 공유
    - child가 종료될 때 parent에 보낼 signal number 포함
  - child_stack
  - child의 esp 레지스터에 할당될 user mode stack pointer를 지정
    - 만약 0인 경우 현재 parent의 stack pointer를 할당해줌. 그렇게 되면 parent와 child가 임시로 같은 유저모드 스택을 사용하게 되지만, 둘중 하나가 스택을 바꾸려고 하면 Copy-on-Write 메커니즘에 의해 둘이 서로 다른 복사본을 얻게 되어 겹칠 일은 없게 됨
  
- wrapper function clone() 작동 원리
  - System call: _syscall2(int, clone, int, flags, void *, child_stack);
  - sys_clone: clone()을 구현하기 위한 커널 함수
  - C 라이브러리의 **clone()** -> 시스템 콜 **_syscall2(clone)** -> 커널 함수 **sys_clone()** -> 커널 함수 **do_fork()**
    - 시스템 콜 까지는 user address space, 커널 함수부터는 kernel address space

#### Less Lightweight Processes

- 프로세스 생성 중 리소스 보존을 위한 2가지 전통적 방법
  - fork() 시스템 콜, Copy-On-Write 메커니즘
    - 부모와 자식이 같은 물리적 페이지를 읽을 수 있게 해줌!(쓰는건 안됨)
    - 둘중 하나가 물리적 페이지에 write를 하려 하면 커널이 해당 페이지를 복사해서 새로운 물리적 페이지를 만듦
  - vfork() 시스템콜은 자신의 부모의 memory address space를 공유하는 프로세스를 생성함
    - 부모의 실행은 자식이 종료되거나 새로운 프로그램을 실행하기 전까지 block됨
- clone()을 이용한 두 시스템 콜 구현
  - fork()
    - SIGCHLD 신호를 지정하고, 모든 플래그를 clear
    - clone(0, 0, SIGCHLD, 0);
  - vfork()
    - SIGCHLD 신호를 지정하고, CLONE_VM, CLONE_VFORK 플래그 발동
    - clone(0, 0, SIGCHLD|CLONE_VM|CLONE_VFORK, 0);
  - 리눅스에서는 스레드가 그저 같은 커널 컨텍스트를 공유하는 프로세스일 뿐이다
- 커널은 clone(), fork(), vfork()를 따라 do_fork()를 호출

#### do_fork()

1. Child를 위한 새 PID를 할당
   - pidmap_array 비트맵에서 정함
2. copy_process()를 호출해 process descriptor를 복사
   1. 새로운 thread_info 구조와 커널 모드 스택을 설정
   2. 유저가 허용된 프로세스 수를 초과했는지 체크
   3. file descriptor, page table 등을 복사
   4. child의 상태를 TASK_RUNNING으로 설정
   5. parenthood 관계 설정
   6. thread group 관계 설정
3. child를 runqueue중 하나에 넣음
4. CLONE_VFORK가 지정된 경우 부모를 wait queue에 넣음
5. child의 PID를 반환하고 종료한다

### Kernel Thread

Kernel-level thread와 다름!!!!!!!!!!!!!!!!!!

- 전통적 UNIX daemon(주기적 서비스 요청 처리)
  - 중요한 커널 task(디스크 캐시 flush, 미사용 페이지 프레임 제거)???
- 현대 UNIX
  - 이러한 기능들을 커널 스레드에 위임
- 커널 스레드는 lightweight process로 구현됨
  - Process descriptor가 있고, schedulable하다! (PID도 존재)
  - 커널모드에서만 실행됨(유저모드 주소공간이 없음)
  - 다른 커널 스레드들와 커널 주소공간을 공유
  - 다른 커널 자료구조들도 공유
    - 파일 시스템(홈 디렉토리, 현재 작업중 디렉토리), 오픈 파일 디스크립터 등
- **커널 스레드와 일반 프로세스의 차이**
  - **일반 프로세스는 커널 함수를 시스템 콜로밖에** 부르지 못하지만, **커널 스레드는 단순한 커널 c 함수로 실행** 가능
  - **커널 스레드는 커널 모드에서만** 실행되지만, **일반 프로세스는 커널 모드, 유저 모드 모두** 실행 가능
  - **커널 스레드**는 **PAGE_OFFSET(3GB) 이상의 선형 주소**만을 사용하지만, **일반 프로세스**는 어느 모드에서 실행되어도 **4GB의 선형 주소 모두 사용 가능**
- 전형적인 커널은 많은 커널 스레드를 가지고 있음
- 유저 프로세스와 커널 스레드 모두 스케줄 가능

![image-20191030045054885](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030045054885.png)

### Process 0 (swapper)

- 모든 프로세스의 조상 격, start_kernel() 함수에 의해 리눅스 커널이 초기화될 때 생성됨
- 커널에서 필요한 모든 자료구조를 초기화하고, interrupt를 활성화하고, 다른 커널 스레드를 생성(process 1: init process)
- init process를 생성한 뒤, process 0은 cpu_idle() 함수를 실행하여 반복적으로 hlt 명령어를 실행한다.
- Process 0은 TASK_RUNNING 상태의 프로세스가 하나도 없을 때 스케줄러에 의해 선택된다.
- 다중 CPU 시스템에서 각각의 CPU에 대해 process 0 존재

### Process 1 (init process)

- 커널의 초기화를 완료하는 init() 함수를 실행
- 메모리 캐시와 swapping 작업을 처리하는 다른 커널 스레드들을 시작시킴
  - bdflush: 메모리를 다시 부를 때 dirty buffer를 flush함
  - kapmd: Advanced Power Management(APM) 처리
  - kswapd: 메모리를 다시 부름
  - keventd, ksoftirqd, ...
- <u>init process는 모든 프로세스를 생성하고 행동을 관찰하기 때문에 종료되지 않는다.</u>

### Destroying Processes

- 프로세스를 종료하기 위해서는 일반적으로 exit()을 하거나 강제로 SIGKILL 신호를 보냄
- 프로세스 종료: do_exit()
- \_\_exit_mm(), \_\_exit_files(), \_\_exit_fs(), \_\_exit_sighand() 등을 호출하여 커널 데이터 구조에서 종료할 프로세스에 대한 대부분의 참조를 제거
  - parenthood relationship을 업데이트
  - parent가 종료되어 parent가 없게 된 child는 init process의 child가 된다(TASK_ZOMBIE 상태)
  - schedule() 함수를 호출하여 새로 실행할 프로세스를 선택
- 프로세스 제거
  - UNIX 커널은 프로세스가 종료되었을 때 바로 데이터를 삭제하지 않고, parent가 프로세스 종료를 의미하는 wait() 류의 system call을 issue한 이후에 삭제가 가능하도록 함(TASK_ZOMBIE 상태)
  - release_task() 호출: 좀비 프로세스의 process descriptor를 release함
  - thread_info와 커널 모드 스택을 담는데 사용한 8KB 메모리 공간을 해제함