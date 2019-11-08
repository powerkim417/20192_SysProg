# Lecture 2. Linux Process Scheduling

## Process Scheduling

### Process Scheduling의 원리

- 한 프로세스에서 다른 프로세스로 전환하면서 여러 프로세스를 동시에 실행
- 결정해야 할 것
  - 언제 전환할지(the decision mode)
  - 어느 프로세스를 선택할 지(the selection function)

### Scheduling의 목적

- 빠른 프로세스 응답 시간
- 백그라운드 작업에 대한 처리량 최적화
- 프로세스의 starvation 방지
- 우선순위가 높은 프로세스와 낮은 프로세스간의 요구사항 조정

## Process Classification

프로세스 내의 작업 중 가장 많은 시간이 소요되는 작업이 그 프로세스를 분류하는데 사용됨.

- 전통적인 분류
  - 입출력 프로세스: I/O 장치 사용량이 많음
    - Ex) DB 어플리케이션, ...
  - CPU 프로세스: 많은 CPU time이 필요
    - Ex) 수치 처리(number crunching) 어플리케이션, ...
- 기타 분류
  - 상호작용 프로세스
    - 키 입력과 마우스 조작을 기다리는데 많은 시간이 소요됨
    - 따라서, 입력을 받으면 process를 빨리 깨워야 한다.
      - 일반적으로 50~150ms의 delay
      - Ex) 커맨드 쉘, 텍스트 편집기, 그래픽 어플리케이션
  - 배치 프로세스
    - 일반적으로 백그라운드에서 실행되며, 스케줄러에 의해 불이익을 받음
      - Ex) 컴파일러, DB 검색엔진, 과학적 계산
  - 실시간 프로세스
    - Deadline을 지켜야 함
    - 우선순위가 낮은 프로세스에게 막혀서는 안되며, 응답성이 우수해야 함
      - Ex) 비디오 및 음성 어플리케이션, 로봇 컨트롤러

## Linux Scheduling Principles

### Scheduling policy

"언제" 및 "어떻게" 새 프로세스를 결정할지 결정하는 규칙의 집합

- Time-sharing
  - CPU-time을 quantum이라는 조각 단위로 분할하는데, 프로세스들은 이 time slice들을 부여받음
  - 이러한 time slice(quantum)이 모두 만료되면 scheduling이 발생
- Priority-based
  - 우선 순위에 따른 프로세스 처리 순서 지정
    - 각 프로세스는 CPU에 할당되는 것이 얼마나 적절한지 나타내는 값(priority)을 가지게 됨
  - 정적 프로세스와 동적 프로세스 모두 지원
    - 정적 우선 순위: 고정 우선 순위
    - 동적 우선 순위: 스케줄러는 프로세스의 진행 상태를 추적하고, 이를 통해 프로세스들의 우선순위를 주기적으로 조정한다.
- Preemptive scheduling
  - 리눅스 프로세스는 선점적이다.
  - RUNNING → READY???
    - 프로세스는 퀀텀이 만료됐을 때 선점될 수 있다.???
  - BLOCKED → READY???
    - 프로세스가 TASK_RUNNING 상태가 되면 커널은 이 프로세스의 동적 우선순위가 현재 실행중인 프로세스의 우선순위보다 높은지 확인한다.
    - 만약 전자의 프로세스의 우선순위가 더 높다면, 현재 실행중인 프로세스는 인터럽트되고, 스케줄러는 더 높은 순위의 프로세스를 선택하여 실행한다.

### Linux Scheduler의 역사

- Linux 2.4 이전
  - 우선순위 기반 스케줄링 / Round-robin 스케줄링
  - Multi-level feedback queue
  - Linux 2.2에서 SMP(Symmetric multiprocessing) 지원 시작
- Linux 2.4.0 ~ Linux 2.4.31
  - O(n) scheduler
- Linux 2.6.0 ~ Linux 2.6.22
  - O(1) scheduler
- Linux 2.6.23 ~ Linux 5.x
  - CFS(Completely Fair Scheduler)
    - O(log n)

## O(1) Scheduler

### 특징

- Linux 2.6.0 ~ Linux 2.6.22 에서 사용
- Ingo Molnar(Redhat)에 의해 소개됨
- **기존의 O(n) scheduler의 성능 문제를 해결!**
- Priority-based & time-sharing & preemptive
- Task의 갯수와 무관하게 O(1)의 시간복잡도로 스케줄링 수행
- 높은 우선순위를 가진 프로세스에게 더 긴 time quantum을 부여(800ms ~ 5ms)
- 두개의 우선순위 배열(active, expired)을 이용해 우선순위를 재계산
- CPU 별로 run queue 부여
- Interactivity metrics 포함(I/O-bound, processor-bound)???
- **Fairness에 대한 보장이 없음**
- 복잡하고, 경험적이며, 오류가 발생하기 쉬운 로직으로 상호작용성을 향상시킴???

### POSIX & Linux Scheduling Policy

#### POSIX.4(=POSIX1003.1b)

3가지의 스케줄링 정책을 정의한다.

- SCHED_FIFO
  - 선입선출 실시간 프로세스
  - 시간 제한 없이 고정된 우선 순위
  - "실시간"은 best-effort??? process에 의해 block되지 않는 다는 뜻일 뿐, 엄격하게 실시간 그 자체를 의미하지는 않음
  - 프로세스가 I/O에 의해 blocked 되거나 더 높은 우선 순위의 프로세스가 실행할 수 있는 상황일 때만 선점이 발생한다!
- SCHED_RR
  - 고정 우선순위 실시간 클래스 + time quantum (Round robin)
- SCHED_OTHER
  - Best-effort class??? (보편적인 프로세스)

#### Linux

<u>POSIX.4를 따라서 설계했으므로 위의 3가지 스케줄링 정책이 모두 제공됨!</u>

- 실시간 서비스(SCHED_FIFO, SCHED_RR)
  - Supervisor mode에서만 사용 가능
- Best-effort??? 서비스(SCHED_OFFER)
  - Linux는 암묵적으로 **I/O-bound 프로세스**를 CPU-bound 프로세스보다 더 선호함
    - 상호작용 어플리케이션에 대해 **빠른 응답 시간**을 제공하기 위해

### Task Priority

140개의 priority level(더 작은 값이 더 높은 우선순위)

- \[0,99\]: 실시간(real-time) task에 대한 우선순위
  - SCHED_FIFO, SCHED_RR
- \[100,139\]: 일반적인 시분할(time-sharing, 한 컴퓨터를 여러명의 사용자가 동시에 사용하는 경우 사용자들이 CPU의 시간 자원을 나눠 쓰는 것) task에 대한 우선순위
  - SCHED_NORMAL (=SCHED_OTHER)
  - nice value: (-20)~(+19)의 값으로 나타낸 경우, default=0.(여기에 120을 더한 값이 위에서의 priority level)
    - 모든 UNIX 시스템에서 표준으로 사용되는 우선순위 범위

### Runqueue

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1569677837508.png" alt="1569677837508" style="zoom:67%;" />



- 각 CPU는 140개의 priority list로 이루어진 runqueue를 가지고 있다.
- 커널은 각각의 우선순위에 대해 실행할 준비가 된 프로세스의 목록을 유지관리한다(TASK_RUNNING 상태).
- Task들은 각각 해당하는 priority list에 들어간다.
- **스케줄러는 다음 task를 스케줄링하기 위해 가장 높은(숫자가 작은) priority list만 보면 된다!**
  - 따라서, 가장 높은 우선순위의 task를 고르기까지 프로세스의 수에 대하여 O(1)의 시간이 소요된다. (140개의 상수 우선순위인 경우) → 프로세스의 수와 무관
- 각 task는 실행 허용 시간을 결정하는 time slice가 존재

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1569685351269.png" alt="1569685351269" style="zoom:67%;" />



### 스케줄링 정책

SCHED_XXX class에 있는 프로세스들은 XXX의 규칙을 따르며 스케줄링되는 방식???

#### SCHED_FIFO Scheduling

- 가장 높은 우선순위를 가진 프로세스가 CPU를 점유함
  - 만약 해당 queue에 여러 프로세스가 있다면, 가장 오래 기다린 프로세스가 CPU를 점유(선입선출, FIFO)
- 현재 실행중인 FIFO 프로세스는 선점을 뺏기지 않음, 그러나 예외 존재
  - 더 높은 우선순위의 FIFO 프로세스가 ready 상태가 될 때
  - 실행중인 FIFO 프로세스가 block 상태가 될 때(I/O 완료 대기 등)
  - 실행중인 FIFO 프로세스가 자발적으로 CPU를 포기할 때(SCHED_YIELD)
- 선점
  - 실행중인 FIFO 프로세스가 인터럽트 된다면, 프로세스는 각각의 runqueue의 현재 위치에 유지됨???
  - FIFO 프로세스가 ready 상태가 되고, 이 프로세스가 현재 실행중인 프로세스보다 더 높은 우선순위를 가진다면 현재 실행중인 프로세스는 더 높은 순위의 프로세스에게 선점당함

#### SCHED_RR Scheduling

- 원리
  - SCHED_RR은 SCHED_FIFO와 같은 방식으로 작동하나, 각 프로세스에 time slice가 적용된다는 차이점을 가짐
    - 만약 SCHED_RR 프로세스가 자신의 time slice만큼의 실행을 마치면 이 프로세스는 CPU에서 제거되며 자신의 run-queue의 맨 뒤에 들어간다.
    - FIFO에서 D-B-C-A 순서로 수행이 되었다면 RR에서는 D-B-C-B-C-A 순서처럼 수행
      - **단, B-C-B-C와 같은 time sharing은 같은 우선순위를 가진 프로세스(여기서는 B, C) 사이에서만 일어남!!**

#### SCHED_NORMAL Scheduling

- 사용되는 경우
  - 만약 ready 상태의 real-time process가 없을 경우, SCHED_NORMAL class에 있는 프로세스들이 스케줄링된다. 
- "Active" & "Expired" Runqueue
  - 만약 프로세스가 자신의 timeslice를 다 사용하지 않은 상태일 때는 Active queue에 들어간다.
  - <u>모든 스케줄링은 Active queue에서 수행된다.</u>
  - 자신의 time slice를 모두 사용한 프로세스는 Active queue에서 Expired queue로 이동된다.
  - 최종적으로 모든 SCHED_NORMAL process들이 모두 만료된(expired queue로 이동된) 경우, 각 프로세스에 새로운 time slice가 할당되며, 모든 프로세스는 다시 active queue로 이동한다.
- 구현
  - "active"와 "expired" 포인터의 교환을 통해!
  - 각 프로세스의 우선순위와 time slice를 한번에 재연산하는 대신, O(1) 스케줄러는 간단히 2단계에 걸친 배열 swap을 이용해 해결한다.???
- 실시간 / 비실시간 프로세스 간의 관계
  - 실시간 프로세스의 경우 정적 우선순위만 존재!
  - SCHED_FIFO 프로세스는 time slice를 할당받지 않는다. 이러한 프로세스들은 FIFO 규율을 따른다.
  - SCHED_FIFO 프로세스가 blocked 상태가 되면, blocked 상태가 해제될 때 까지 active queue list에 있는 동일한 priority queue로 돌아가 있는다.???
  - SCHED_RR 프로세스는 time slice를 할당받지만, expired queue로 이동하지는 않는다.???
  - **"active"와 "expired" queue list 간의 전환은 TASK_RUNNING 상태에 실시간 task가 없을 때만 발생할 수 있다.???**
- Fairness 제공
  - SCHED_NORMAL 프로세스는 Fairness를 제공받는다.(얘네 끼리만!)
  - 자신의 time slice를 모두 사용한 프로세스는 expired 상태가 된다.(more or less???)
  - 결론적으로 모든 active 프로세스가 CPU를 점유하게 된다.
  - SCHED_NORMAL class의 프로세스의 경우 Linux 스케줄러에서 이 프로세스가 I/O-bound 인지 CPU-bound 인지 확인한다.
    - 커널은 프로세스가 얼마나 sleep 하는지와 얼마나 실행되는지의 시간을 계속 트래킹한다.
- 좋은 상호작용 응답 시간을 유지하기 위해 → Linux 커널은 I/O-bound 프로세스를 선호! (동적 우선 순위)

### Static & Dynamic Priority

#### Static Priority

- 모든 SCHED_NORMAL 프로세스는 정적 우선순위를 가지고 있음
  - 100(최고, nice: -20) ~ 139(최저, nice: +19)
  - 기본: 120(nice: 0)
  - 정적 우선순위에 따라 time slice의 길이가 정해짐(=base time quantum)
    - 이 길이는 사용자가 nice() 또는 setpriority() 시스템 콜을 사용하여 변경 가능

##### Time slice

- Task가 다른 task에 의해 선점되기 전까지 얼마나 오래 실행될 수 있을지를 나타내는 값
- Issue
  - 너무 짧은 경우: Task switch가 빈번하게 발생해 시스템 오버헤드가 지나치게 높아짐
    - quantum 10ms에 task switch 10ms인 경우 50%의 활용률...
  - 너무 긴 경우: 프로세스가 더이상 동시에 실행되는 것 같지 않게 됨
    - 시스템의 응답성이 저하됨
  - Linux에서 채택한 경험 법칙
    - 좋은 시스템 응답 시간을 유지하는 동안은 지속시간을 최대한 길게 잡는다.???
- Time slice length 결정
  - time slice를 (재)할당하는 경우 Linux 스케줄러는 프로세스의 정적 우선순위를 기반으로 프로세스의 time slice(=base time quantum)를 결정한다.
    - <u>(정적 우선순위 < 120) Base time quantum = (140 - static priority) * 20</u>
    - <u>(정적 우선순위 >= 120) Base time quantum = (140 - static priority) * 5</u>
    - static priority가 높은(값이 낮은) 수일 수록 더 많은 시간을 할당받음
    - static priority가 높은(값이 낮은) 수일 수록 더 적은 sleep time threshold를 가진다.???
  - Time slice의 범위: 5ms(s.p = 139) ~ 800ms(s.p = 100)
- task_struct 에서 int static_prio로 선언

#### Dynamic Priority

- 좋은 응답 시간을 유지하기 위해 사용
- CPU-bound 프로세스보다 I/O-bound 프로세스를 더 선호
  - 동적 우선순위는 정적 우선순위와 해당 task의 상호작용성을 바탕으로 계산된다.
  - 프로세스의 Average sleep time 또한 task의 상호작용성과 관련이 있어 동적 우선순위에 영향을 미친다.
    - **Sleep을 많이 하는 프로세스가 더 높은 동적 우선순위를 가진다.**
    - 이러한 프로세스들은 I/O와 같은 이벤트로 인한 대기가 자주 발생함
  - SCHED_NORMAL 스케줄링은 동적 우선순위에 의해 결정된다
    - bonus: task의 상호작용성 또는 평균 sleep 시간에 의해 결정
    - Dynamic priority = max( 100 , min( (static_priority-bonus+5) , 139 ) )
      - 기본적으로 static_priority - bonus + 5의 값이되, 100~139 범위를 넘어가면 그 경계값을 취한다.
- task_struct 에서 int prio로 선언

### Scheduling for fork()

새 프로세스가 생성되면, do_fork()은 current(부모 프로세스)와 p(자식 프로세스)의 time_slice 값을 설정한다.

- p->time_slice = (cur->time_slice + 1) >> 1;
- current->time_slice >>= 1;
- 부모 프로세스의 남은 time slice를 **반씩 나눠 가지는 방식**! 남은 time slice가 짝수면 반씩 가지고, 홀수면 자식이 k+1, 부모가 k를 가짐
  - (부모 프로세스에서 나눠 가지는 방식이므로) 사용자는 쉘에서 많은 백그라운드 실행을 시키거나 GUI desktop에서 많은 윈도우를 띄우는 것과 같은 방법으로 프로세서를 독차지할 수 없다.
  - 즉, 프로세스는 여러 자손 프로세스를 fork하는 것으로 리소스를 독차지하지 못한다.(커널이 fork에 대해 보상해주는 것이 없음)

### 주요 Scheduler 함수

/usr/src/linux/kernel/sched.c 에 위치

- scheduler_tick()
  - time_slice 카운터를 최신으로 유지
- try_to_wake_up()
  - sleep 중인 프로세스를 깨움
  - recalc_task_prio() 함수를 호출하고 dynamic priority에 의해 index된 active list에 process descriptor를 삽입한다. (enqueue_task())
- recalc_task_prio()
  - 평균 sleep 시간과 프로세스의 동적 우선순위를 갱신한다.
- schedule()
  - 새 프로세스를 runqueue list에서 선택하고 CPU에 할당한다.
- load_balance()
  - 멀티프로세서 시스템에서 run-queue들의 균형을 유지

### schedule() 을 이용한 Scheduling

#### Outline

kernel/sched.c 에 위치

```c++
void schedule(void){
	// lots of checking...
    
    // 활성 프로세스를 run-queue로부터 선택
    
    switch_mm();
    // 프로세스의 가상 주소공간을 switch함(prev → next)
    // (prev의 가상 주소공간을 가리키고 있던 포인터를 next의 가상 주소공간으로 옮김)
    // (새 주소공간을 마련하기 위해 Page Global Directory를 전환하는 것)
    
    switch_to(prev, next, prev);
    // 프로세스가 prev에서 next로 전환됨
}
```

위의 두 함수(switch_mm, switch_to)는 schedule() 내부에서 호출된 context_switch()에서 호출한다.

#### schedule() 함수 호출

- runqueue list에서 프로세스를 찾은 뒤 CPU에 할당해준다.
  - 여러 커널 루틴에 의해 직접(directly) 또는 지연된(deferred) 방식으로 호출된다. 

 <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570020103066.png" alt="1570020103066" style="zoom:67%;" />???

- Direct invocation of schedule(): <span style='color:blue'>Case (1), (4)</span> 

- Deferred invocation of schedule(): <span style='color:red'>Case (2), (3)</span> (Ready로 가는 경우)
  - 지연 호출의 경우 선점적인 CPU 스케줄링을 구현하기 위해 필요하다!

##### Direct invocation

- <u>현재 프로세스가</u> 필요로 하는 자원이 현재 사용할 수 없는 상태라서 <u>바로 block되어야 할 때</u> 스케줄러가 직접 호출!
  - sleep 상태로 갈 때(sleep(), blocking in-kernel API, lock in-kernel API)???
- $ grep -r "schedule()" /usr/src/linux 명령어 실행시???
- 만약 현재 프로세스 current가 block되어야 하는 경우, 커널은 다음의 순서를 따른다.
  1. current를 올바른 wait queue에 삽입한다.
  2. current의 상태를 TASK_INTERRUPTIBLE 또는 TASK_UNINTERRUPTIBLE로 바꾼다.
  3. **schedule() 함수를 호출한다!**
  4. resource가 사용 가능한지 확인한다. 만약 사용 불가능할 경우 step 2로 간다.
  5. resource를 사용 가능할 경우, current를 wait queue에서 제거한다.
     - Resource의 사용 가능여부를 지속적으로 확인한다. 만약 사용 불가능한 경우, schedule() 함수를 호출해 CPU를 다른 프로세스에게 양보한다. 마침내 resource가 사용 가능하게 될 경우, step 5가 실행된다.
- Note
  - 긴 반복작업을 실행하는 다양한 장치 드라이버에 의해 직접 호출됨
  - <u>프로세스가 다음에 실행되기로 예정되어 있을 때 Step 4, 5가 실행된다!</u> 

- 예시
  - USB 장비를 통한 유저 공간 상호작용(drivers/usb/media/dabusb.c)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570038737354.png" alt="1570038737354" style="zoom:67%;" />???

##### Deferred (or, lazy) invocation

- <u>커널 모드 작업에서</u> 스케줄링 요청을 확인
- 필요한 경우, 스케줄러는 TIF_NEED_RESCHED flag의 값을 현재 값에서 1로 바꾸어 설정하는 방식으로 지연된 호출을 한다.
- 유저 모드 프로세스의 실행을 재개하기 전에 확인
  - Ex) System call 또는 인터럽트 핸들러로부터 반환되는 경우
- Deferred invocation이 발생할 때의 모드 전환

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570020833405.png" alt="1570020833405" style="zoom:67%;" />???

1. intr 0x80 실행(system call을 시작하기 위한 exception)
2. sys_read() 커널 제어 경로는 디스크 읽기 작업을 시작하고, 데이터가 도착할 때까지 block 상태가 된다.
3. 데이터가 준비된 디스크 컨트롤러에서 인터럽트 발생. 커널은 데이터를 가져와 P1의 유저 공간에 있는 "buf" 버퍼에 복사한다.
4. P1의 우선 순위가 P2보다 높다고 가정할 때, P1이 다시 실행되도록 스케줄링 될것이다.
   - schedule()의 지연된 호출을 통한 프로세스 전환 (P2 → P1)

- 원리
  - 커널 모드 작업에서 스케줄링 요청을 확인
  - TIF_NEED_RESCHED 필드의 값을 current에서 1로 설정함으로써 스케줄러가 지연된 방식으로 호출된다.
  - set_tsk_need_resched() 또는 resched_task() 로 설정된다. (#ifdef CONFIG_SMP)
  - TIF_NEED_RESCHED 값의 확인이 항상 유저모드 프로세스로의 복귀 전에 이루어지므로, schedule() 함수는 반드시 가까운 미래에 호출될 것이다.???
- 커널 모드에서의 scheduling 기회
  - current 프로세스가 자신의 CPU time의 quantum을 모두 사용한 경우(timer interrupt)
    - 이 경우 scheduler_tick() 함수에 의해 수행된다.
  - 어떤 프로세스가 깨어났을 때 이의 우선순위가 현재 프로세스의 우선순위보다 높은 경우
    - 이 경우 try_to_wake_up() 함수에 의해 수행된다.
  - System call 중 sched_setscheduler() 또는 sched_yield() 이 실행된 경우
- 예시???

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570039960330.png" alt="1570039960330" style="zoom: 67%;" />

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570040005029.png" alt="1570040005029" style="zoom: 67%;" />

#### System calls related to Scheduling

##### Normal user

- 사용자는 항상 자신의 프로세스의 우선순위를 낮추는 것이 허용된다.
- nice()
  - 프로세스가 자신의 우선순위를 낮춤
  - 프로세스의 우선순위를 높이는 것은 root만이 가능하다.
- getpriority(), setpriority()
  - 2.6 버전 이후의 커널에서 nice()가 setpriority()로 대체되지만, 역방향 호환성을 위해 존재한다.

##### Supervisor mode

- 주로 실시간 프로세스와 관련있음
- sched_getscheduler(), sched_setscheduler() // scheduler get/set
- sched_getparam(), sched_setparam() // parameter get/set
- sched_yield() // yield
- sched_get_priority_min(), sched_get_priority_max() // get priority min/max
- sched_rr_get_interval() // round robin get interval

### O(1) Scheduler의 문제점

- Time-slice 기반으로 인한 문제(공정성 결여)
  - 우선순위마다 switching rate가 다름
    - 우선순위가 높은 경우 time-slice가 크므로 switching이 드물게 발생해 응답 시간이 느려짐
    - 우선순위가 낮은 경우 time-slice가 작으므로 switching이 빈번히 발생해 throughput이 저하됨
  - 불공평한 CPU 할당
    - 우선순위가 높은 경우 Pri=120일 때와 Pri=121일 때의 CPU time 차이는 100ms와 95ms로 5%밖에 차이가 나지 않는다.
    - 그러나 우선순위가 낮은 경우 Pri=138일 때와 Pri=139일 때의 CPU time 차이는 10ms와 5ms로 50%나 차이가 나게 된다.

- "expired" queue의 task에 대해 starvation 발생
  - "active" queue에 있는 일부 task들로 인해...
- 동적 우선순위를 조정하기 위해 Task를 특성에 따라 분류해야 하는데, 이를 위해 어마어마한 양의 경험적인 코드가 필요 

## CFS(Completely Fair Scheduler)

### 특징

- Linux 2.6.23부터 현재까지 사용되고 있음
- Con Kolvivas의 작업에 영감을 받아 Ingo Molnar가 처음으로 소개
- 기존의 UNIX 프로세스 스케줄러에 비해 크게 달라짐
- **목표: 모든 프로세스가 공평하게 CPU를 공유할 수 있도록 보장!**
- O(1) 스케줄러에서의 상호작용 성능을 향상시킴
- "실행이 가장 필요한 Task"를 우선 실행
- 공평성을 위해 Priority 대신 Weight 사용
- 2개의 priority array(active, expired) 대신 Red-black tree 기반의 runqueue를 사용한다. → O(log N) 
- Virtual runtime(vruntime)이 스케줄링의 기준이 된다.

### Scheduling Class

- 2.6.23버전 커널 이후로, 리눅스 스케줄러는 모듈화된 형태로 제공된다.
  - 이 모듈을 <u>scheduling class</u>라고 부른다.
  - Scheduling class는 모듈화 방식을 통해 여러 다른 스케줄링 정책을 구현할 수 있다.
  - 각 스케줄링 정책은 base scheduler code를 통해 독립적으로 구현할 수 있다.
    - Ex) RT, Deadline, CFS, Idle-task, ..., Custom 등등의 스케줄러
  - 각 scheduling class는 우선 순위가 존재
    - base scheduler coder가 우선순위를 기반으로 각 scheduler class를 반복 순회함.???

- 각 Task는 하나의 특정한 scheduling class에 속한다.

  - 각 scheduling class는 singly linked list를 통해 다른 scheduling class와 연결되어 있으며, 이로 인해 class 간의 iterated가 가능하다.???

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570206862612.png" alt="1570206862612" style="zoom:67%;" />	

  - struct sched_class 라는 특수한 자료구조를 통해 표현

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570206949951.png" alt="1570206949951" style="zoom:67%;" />

  - 코드 예시

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570207264085.png" alt="1570207264085" style="zoom:67%;" />

  rt_sched_class의 next가 fair_sched_class이고, 이의 next가 idle_sched_class인걸 보아 class 단위로 iterate 되는듯..

  |      Function      |                          Operations                          |
  | :----------------: | :----------------------------------------------------------: |
  |    enqueue_task    |      Task를 run queue에 삽입, nr_running 변수 값 1 증가      |
  |    dequeue_task    |     Task를 run queue에서 제거, nr_running 변수 값 1 감소     |
  |     yield_task     |   task를 run queue에서 제거하고 다시 삽입(대기번호 리셋0)    |
  | check_preempt_curr | 현재 실행중인 task가 새로운 task에 의해 선점될 수 있는지 확인 |
  |   pick_next_task   |       다음에 실행시킬 수 있는 가장 적합한 task를 선택        |
  |   set_curr_task    |          task의 scheduling class 또는 group을 변경           |
  |    load_balance    |     load balancing code를 발동시킴(run-queue 균형 유지)      |

### Concept

- 이상적인 멀티태스킹 시스템을 달성하기 위한 목적!

- 각 Task의 CPU Time을 그의 weight에 비례해 제공한다.
  $$
  C_{\tau_i}(t_1, t_2) = \dfrac{W({\tau_i})}{S_\phi}\times(t_2-t_1)
  $$

  - 각각 "해당 Task의 CPU time", "해당 Task의 weight 비율", "전체 CPU가 실행되는 시간(t1과 t2는 CPU가 시작/종료되는 시각)"을 의미

### Task Weight

- CFS에서는 task의 우선순위를 표현하기 위해 task의 weight를 사용한다.
- 1 nice level이 떨어질 때(우선 순위가 높아질 때)  = weight가 1.25배 증가
  - CPU 효율 10% 증가!
  - nice level = -20일 때 weight = 88761
  - nice level = 0일 때 weight = 1024 (기준)
  - nice level = 19일 때 weight = 15

### Virtual Runtime

- CFS가 모델링하는 이상적 멀티태스킹을 근사화함

- "프로세스가 지금까지 얼마나 실행되었는지"와 "프로세스가 얼마나 더 실행되어야 하는지"를 설명
  
- 정확히 무슨 수치를 의미하는 값일까???
  $$
  VirtualRuntime(\tau, t) = \dfrac{Weight_0}{Weight_\tau}\times ExecutedRuntime(\tau, t)
  $$

  - $Weight_0$: nice value 0의 weight(=1024), $Weight_\tau$: task $\tau$의 weight
  - ExecutedRuntime: 프로세스의 누적 실행시간
  - virtual runtime이 작을 때: task가 자신에게 필요한 CPU time보다 적게 할당받음
  - virtual runtime이 클 때: task가 자신에게 필요한 CPU time보다 많이 할당받음
  
- <u>스케줄링 시 가장 virtual runtime이 작은 task를 고른다!</u>

- Task의 누적 실행 런타임은 **weight에 반비례**

  - 우선순위는 값마다 다르게 소멸되는 것을 초래한다. (역효과) ??? 
  - 더 높은 우선순위의 task는 더 많은 CPU time을 필요로 하므로 vruntime을 더 적게 축적한다.
  - 더 낮은 우선순위의 task는 vruntime이 더 빨리 증가하여 더 일찍 선점될 것이다. 

### Time Slice

- Task가 선점당하지 않고 실행을 할 수 있는 시간

- Task의 time slice 길이는 **task의 weight에 비례**한다. (runqueue의 모든 프로세스 중 해당 task의 weight의 비율을 곱함)
  $$
  TimeSlice_{\tau_i} = \dfrac{Weight_{\tau_i}}{\sum_{j\inφ}Weight_{\tau_j}}\times P
  $$

  - φ: 실행 가능한 task의 집합, $\sum_{j\inφ}Weight_{\tau_j}$: runqueue의 총 weight
  - $P$: 주어진 작업량에 따른 상수
    - n < nr_latency일 때, P = sched_latency
    - n >= nr_latency일 때, P = min_granularity * n
    - sched_latency: CPU-bound task에 대한 예상 round-robin time(선점 시간)???
    - nr_latency: minimum preemption granularity for CPU-bound tasks???
    - n: task의 갯수
    - time slice가 가질 수 있는 최소값: min_granularity * n???
    - 현재 구현에서는 sched_latency: 6ms, nr_latency: 8, min_granularity: 0.75ms???

#### Example

P가 10으로 주어졌을 때

- Time slice
  - priority가 x, x일 경우: time slice의 비율은 1:1이 되어 각각 5ms, 5ms를 가져간다.
  - priority가 x, x+1일 경우: time slice의 비율은 1.25:1이 되어 각각 5.6s, 4.4ms를 가져간다.
  - priority가 x, x+5일 경우: time slice의 비율은 $1.25^5$:1이 되어 각각 7.5ms, 2.5ms를 가져간다.
- Virtual time
  - priority가 120, 120일 경우 time slice는 5ms, 5ms
    - 만약 ExecutedRuntime이 각각 40ms, 30ms라면
    - VirtualRuntime은 1024/1024 * 40ms = <u>40ms</u>, 1024/1024 * 30ms = <u>30ms</u>
  - priority가 120, 125일 경우 time slice는 7.5ms, 2.5ms
    - 만약 ExecutedRuntime이 각각 40ms, 30ms라면
    - VirtualRuntime은 1024/1024 * 40ms = <u>40ms</u>, 1024/335 * 30ms = <u>90ms</u>

### Red-Black(RB) Tree

RB Tree를 통한 Task 선택

- CFS는 모든 실행 가능한 task들이 vruntime에 의해 정렬된 시간 순서의 red-black tree를 관리한다.
- 어떤 경로도 다른 경로보다 2배 이상 길지 않다(RB Tree의 특징: Self-balancing)
- 트리에 대한 연산은 O(log N)
- 노드는 sched_entity를 표현하고, Task의 vruntime 값이 해당 노드의 key가 된다.
- 다음에 실행할 task를 고르려면 가장 왼쪽의 노드를 택하면 된다.
  - 가장 왼쪽의 노드를 실행하기 위해 tree에서 제거하면, self-balancing 특성에 따라 트리의 구조가 바뀌면서 새로운 가장 왼쪽 노드가 생긴다.

#### Scheduler Entity Structure

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570216203530.png" alt="1570216203530" style="zoom:67%;" />

```c++
struct rb_node run_node; // RB 트리의 노드
unsigned int on_rq; // runqueue 관련 상태 플래그
u64 exec_start; // 실행 시작 시각???
u64 sum_exec_runtime; // 총 실행 시간???
u64 vruntime; // virtual runtime
u64 prev_sum_exec_runtime; // 이전까지의 총 실행 시간???
```

- 프로세스 관리
  - CFS는 각 프로세스가 실행되는 시간을 고려해야 한다.
  - Scheduler Entity Structure(struct sched_entity 로 구현)를 사용하여 프로세스 관리를 트래킹한다.
  - struct task_struct에서 se 라는 멤버변수로 포함되어 있음

### Runqueue 구조

Linux 2.6.23 이후

- 각 CPU는 하나의 run queue를 가지고 있다.
  - run queue는 struct rq 자료구조를 통해 구현
  - CFS RB-tree로 이루어진 일반(시분할) task와 배열로 이루어진 실시간(real-time) task에 대한 sub run-queue로 구성되어 있다. (struct rt_rq, struct cfs_rq)
- cfs_rq 구조의 주요 필드 변수들

```c++
struct load_weight load; // CFS run queue 로드???
unsigned long nr_running; // run queue에 있는 task의 갯수
...
struct rb_root tasks_timeline; // RB-tree의 root node를 가리키는 포인터
struct rb_node *rb_leftmost; // RB-tree의 가장 왼쪽 node를 가리키는 포인터
...
u64 load_avg, load_period, load_stamp, load_last, load_unacc_exec_time;
// CPU group scheduling을 위해 CPU당 로드하는 정보
```

### Big picture

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570210115192.png" alt="1570210115192"  />

실시간 task는 doubly linked list로 저장되어있고

일반 시분할 task는 RB-tree로 저장됨

### Overall Scheduling flow(전체적인 흐름)

CFS에서 각 scheduling tick마다 실행되는 것들: checking preemptability, invoking schedule()

- 선점 여부 확인

  - 현재 실행중인 task의 time slice를 tick 간격으로 뺀다.
    - time slice가 0이 되면 TIF_NEED_RESCHED 플래그가 1로 설정된다.
  - 현재 실행중인 task의 virtual runtime을 갱신한다.
    - 만약 현재 task의 갱신된 virtual runtime이 다음 task(RB tree의 가장 왼쪽에 있는 task)의 virtual runtime보다 커지면 TIF_NEED_RESCHED 플래그가 1로 설정된다.

  TIF_NEED_RESCHED 플래그가 1이 되면 다른 프로세스에게 CPU를 내주는듯.

- schedule() 함수 호출

  - TIF_NEED_RESCHED 플래그를 확인한다.
    - 만약 true일 경우, run queue에서 가장 작은 virtual runtime을 가진 task를 스케줄링한다.
    - 현재 실행중이던 task를 RB tree에 삽입한다.
    - 가장 왼쪽에 있는 노드에 해당하는 task(위에서 스케줄링하고자 하는 task)를 RB tree에서 제거한다.

#### TIF_NEED_RESCHED 플래그 설정

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570275199559.png" alt="1570275199559" style="zoom:67%;" />

**scheduler_tick()**
**→ task_tick_fair()**
**→ update_curr()**
**→ (실행 가능한 task가 2개 이상인 경우) check_preempt_tick()**
**→ (선점되어야 하는 경우) resched_task()**
 **→ TIF_NEED_RESCHED = 1**

- 현재 실행 가능한 task가 1개 이하인 경우는 선점 따질 필요 없음
- 현재 task가 자신의 time slice를 모두 소모하였거나, 현재 task의 vruntime이 다음 task(RB-tree 가장 왼쪽 노드)의 vruntime보다 커지는 경우 resched_task() 호출 후 TIF_NEED_RESCHED = 1이 된다.

#### schedule() 함수 호출

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570275543551.png" alt="1570275543551" style="zoom:67%;" />

**(TIF_NEED_RESCHED == 1인 경우)**
**→ schedule()**
**→ preempt_disable(): 프로세스 전환이 일어나는 동안은 다른 preempt가 일어나면 안된다!**
**→ put_prev_task() (→ \_\_enqueue_entity()): 현재 task를 run queue에 다시 넣음**
**→ pick_next_task() (→\_\_dequeue_entity()): 다음 task를 run queue에서 제거하여 현재 task로 설정**
**→(rescheduling이 더 필요한 경우 preempt_disable()로 돌아간다.)**

### Linux에서의 멀티프로세서 스케줄링

- Load balancing: CPU간의 workload를 균등하게 분산
  - 최적의 자원 활용도, 처리량 극대화, 응답 시간 최소화를 위해!
  - Load balancing overhead
    - Information collection: task migration overhead보다는 마이너한 overhead
    - Task migration
      - Scheduling: 커널 자료구조 갱신, context switch
      - Cold cache effect: migration 시 register state, TLB state와 같은 transferring program state가 필요.???
- Linux CFS 스케줄러는 여러 CPU(runqueue)에 load를 균등하게 분산하려고 함
  - 2.6.7 버전부터 Linux는 CPU의 topology에 기반한 runqueue balancing을 지원함
  - CPU(run queue)의 load = task load(weight * utilization)의 합
- 원리
  - 다른 CPU들이 실행 대기중인 task들이 있는동안 **CPU가 유휴 상태가 되지 않도록 방지**
  - 모든 CPU에서의 **ready task의 수 차이를 가능한 한 최소화**
  - <u>Scheduling domain</u>의 불균형이 심한지 확인
    - 프로세스를 가장 바쁜 group에서 local CPU의 runqueue로 옮긴다면 불균형이 줄어들 수 있는지 확인

#### Scheduling Domains & Scheduling Groups

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570278074290.png" alt="1570278074290" style="zoom:67%;" />

##### Scheduling Domain

- 각 Scheduling domain은 여러 CPU에 걸쳐 구성된다.
- **Load balancing은 scheduling domain 내에서 이루어진다.**
- 각 Scheduling domain은 1개 이상의 scheduling group을 가져야 한다.

##### Scheduling Group

- 각 Scheduling group은 1개 이상의 CPU를 포함하고 있다.
- **Load balancing은 scheduling group 간에 이루어진다.**
- Scheduling group 내의 CPU끼리는 overhead가 적게 발생하고, Scheduling group 간에는 overhead가 높게 발생한다.

#### Linux에서의 Load Balancing 발동

##### Time-driven load balancing

- Timer interrupt에 의해 발동
- load balancing code인 trigger_load_balance()가 scheduler_tick()과 함께 주기적으로 실행됨
- 필요한 경우 run_rebalance_domains() 함수를 통해 load balancing을 수행한다.

##### Event-driven load balancing

- Scheduling event에 의해 발동
- Task가 새로 생성되거나 fork(), exec(), wakeup() 등을 통해 깨어난 경우
  - 현재 domain에서 가장 load가 적은 group을 선택
  - 가장 load가 적은 CPU로 task를 이동시킨다.
  - 즉, 가장 load가 적은 group에서 가장 load가 적은 CPU 선택
- CPU가 새로이 유휴상태가 되는 경우
  - 현재 domain에서 가장 load가 많은 group을 선택
  - 가장 load가 많은 CPU에서 이 CPU로 task들을 이동시킨다.
  - 즉, 가장 load가 많은 group에서 가장 load가 많은 CPU 선택
  - Ex) CPU1이 새로이 유휴상태가 되는 경우

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570280337352.png" alt="1570280337352" style="zoom:67%;" />

### Kernel-Mode Preemption

- 참고) User-Mode Preemption
  - 커널이 인터럽트 또는 system call 이후 user-space로 복귀하려 할 때
  - 만약 TIF_NEED_RESCHED == 1인 경우, 스케줄러는 실행할 새로운 프로세스를 선택하도록 호출된다.

#### Kernel-Mode Preemption

- 2.6 버전 이후의 Linux kernel은 fully preemptive하게 되었다.
- 커널 코드는 완료될 때까지 실행될 필요가 없음
- 커널 모드에서도 프로세스의 실행이 선점될 수 있음!
- 이점: **현재 프로세스가 user mode로 돌아가기 전까지 선점을 지연시킬 필요가 없음**

#### 작동 원리

- <u>프로세스에 lock이 걸려있지 않는 한</u> 커널 모드에서도 선점될 수 있다.
  - Lock은 비선점성에 대한 표시로 사용됨
- 모든 프로세스는 struct thread_info에 preempt_count 라는 선점 카운터가 있다.
  - 0에서 시작하고, lock을 얻을 때마다 1씩 증가한다.
  - lock이 해제될 때마다 1씩 감소한다.
  - 값이 0일 때 프로세스는 선점될 수 있다.
- 인터럽트에서 kernel space(user space 아님)로 돌아올 때
  - 커널은 TIF_NEED_RESCHED 플래그와 preempt_count의 값을 확인한다.
  - TIF_NEED_RESCHED==1일 때 reschedule이 일어나야 하지만, preempt_count의 값에 따라 바로 reschedule 할지에 대한 여부가 달라진다.
    - (TIF_NEED_RESCHED==1) && (preempt_count==0) 일 때: Reschedule
    - (TIF_NEED_RESCHED==1) && (preempt_count>0) 일 때: 모든 lock이 풀릴 때까지 reschedule이 지연된다.