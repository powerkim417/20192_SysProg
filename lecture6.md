# Lecture 6. Signals

- **Signal**: 이벤트를 알리기 위해 프로세스에 전해지는 것
  - 프로세스 하나 또는 여러 프로세스들에게 전해지는 짧은 메시지를 이용한 프로세스간 통신(Inter Process Communication, IPC)
    - Signal을 식별할 수 있는 숫자 1개로 이루어져 있으며, 다른 인자는 없다.
    - 단순하고 효율적이라서 널리 사용된다
      - 최초 UNIX에서 처음 등장하고, 40년동안 작은 개선만 있었음
  - 일반적으로, signal에 대한 응답으로 프로세스는 user-space function(signal-handler)을 호출
- 목적
  - 프로세스에게 특정 이벤트가 발생했음을 알리기 위해
  - 프로세스가 자신의 코드에 포함된 signal handler function을 실행하도록 하기 위해
  - "프로세스에서 signal은 커널에서 interrupt와 같다"

- Signal의 종류
  - Signal number: SIGxxx 형태의 매크로
    - Ex) SIGCHLD(값: 17): 자식 프로세스가 멈추거나 종료될 때 부모 프로세스에게 보내지는 signal
    - Ex) SIGINT(값: 2): 'Ctrl+C'를 입력할 때 발생하는 signal
      - 이를 이용해 쉘의 foreground process를 kill 함
  - POSIX 표준에 의해 정의된 "real-time" signal
- 프로세스는 특정 signal을 처리하는 방법을 변경할 수 있음
  - (기본) 커널 default 사용
  - signal을 무시
  - signal을 block(나중에 unblock될 때까지 대기)
  - 사용자가 만든(user-provided) 다른 signal-handler 함수 사용

### (Remind) Exceptions in Linux

- 20개의 서로 다른 exception에 대해 커널은 각 exception type에 해당하는 전용 exception handler를 제공해야 하며, exception handler는 exception이 발생한 프로세스에게 해당 UNIX signal을 보낸다.
- 즉, exception마다 해당하는 signal이 있음!

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191111141057251.png" alt="image-20191111141057251" style="zoom: 80%;" />

### Signals in Linux/x86

Table 11-1. Linux/i386의 첫 31개 signal

|  #   |   Signal Name    | Default Action |              Comment               | POSIX |
| :--: | :--------------: | :------------: | :--------------------------------: | :---: |
|  1   |      SIGHUP      |   Terminate    |   터미널 또는 프로세스 제어 문제   |   O   |
|  2   |      SIGINT      |   Terminate    |     키보드 인터럽트(Ctrl + C)      |   O   |
|  3   |     SIGQUIT      |      Dump      |          키보드에서 나옴           |   O   |
|  4   |      SIGILL      |      Dump      |           잘못된 명령어            |   O   |
|  5   |     SIGTRAP      |      Dump      |         디버깅 breakpoint          |   X   |
|  6   | SIGABRT / SIGIOT |      Dump      |            비정상 종료             | O / X |
|  7   |      SIGBUS      |      Dump      |             버스 오류              |   X   |
|  8   |      SIGFPE      |      Dump      |       부동 소수점 exception        |   O   |
|  9   |     SIGKILL      |   Terminate    |         프로세스 강제 종료         |   O   |
|  10  |     SIGUSR1      |   Terminate    |        프로세스에 사용 가능        |   O   |
|  11  |     SIGSEGV      |      Dump      |         잘못된 메모리 참조         |   O   |
|  12  |     SIGUSR2      |   Terminate    |        프로세스에 사용 가능        |   O   |
|  13  |     SIGPIPE      |   Terminate    |   reader가 없는 pipe에 write???    |   O   |
|  14  |     SIGALRM      |   Terminate    |          Real time clock           |   O   |
|  15  |     SIGTERM      |   Terminate    |           프로세스 종료            |   O   |
|  16  |     SIGTKFLT     |   Terminate    |        코프로세서 스택 오류        |   X   |
|  17  |     SIGCHLD      |     Ignore     |     자식 프로세스 stop or 종료     |   O   |
|  18  |     SIGCONT      |    Continue    | 만약 실행 stop 상태인 경우 resume  |   O   |
|  19  |     SIGSTOP      |      Stop      |         프로세스 실행 stop         |   O   |
|  20  |     SIGTSTP      |      Stop      |   tty에 의한 프로세스 실행 stop    |   O   |
|  21  |     SIGTTIN      |      Stop      | input이 필요한 background process  |   O   |
|  22  |     SIGTTOU      |      Stop      | output이 필요한 background process |   O   |
|  23  |      SIGURG      |     Ignore     |     소켓에서의 긴급한 조건???      |   X   |
|  24  |     SIGXCPU      |      Dump      |         CPU time 제한 초과         |   X   |
|  25  |     SIGXFSZ      |      Dump      |        파일 크기 제한 초과         |   X   |
|  26  |    SIGVTALRM     |   Terminate    |          가상 timer clock          |   X   |
|  27  |     SIGPROF      |   Terminate    |         프로필 timer clock         |   X   |
|  28  |     SIGWINCH     |     Ignore     |          윈도우 리사이징           |   X   |
|  29  | SIGIO / SIGPOLL  |   Terminate    |      I/O가 이제 가능함을 알림      |   X   |
|  30  |      SIGPWR      |   Terminate    |          Powersupply 실패          |   X   |
|  31  |    SIGUNUSED     |      Dump      |          (사용하지 않음)           |   X   |

32~63번은 real-time signal

### Characteristics of Signals

- Signal은 언제든 프로세스에 보내질 수 있다.
- Signal이 실행되고 있지 않은 프로세스에 보내질 경우
  - "Signal이 전송되었다는 것"을 해당 프로세스가 다시 실행될 때까지 커널에서 저장하고 있어야 한다!
- Signal은 일반적으로 현재 실행중인 프로세스에 의해서만 처리된다.(current 프로세스)
- 전송된 각 Signal은 2번 이상 수신할 수 없다.
  - Signal을 프로세스에 저장하기 위해 bitmap 사용
- Signal은 프로세스에 의해 선택적으로 block될 수 있다.
  - 프로세스가 block을 해제하기 전까지는 signal을 수신하지 않음
- 프로세스가 signal-handler function을 실행하면, 해당 handler가 종료될 때까지 signal을 block한다.
  - signal handler는 또 다른 handled signal???이 발생해도 인터럽트될 수 없다. 

### Signal Handling

- 각 task에는 다음과 같은 자료구조들이 유지되고 있음

  - Signal handler의 배열(각 인덱스 당 하나의 signal type)
  - Pending signal의 리스트
  - Blocked signal의 mask(SIGHUP, SIGINT 같은거?)

- Pending signal

  - Pending: Signal이 task에 보내졌지만 아직 처리되지 않은 경우
  - Signal은 system call을 포함하는 interrupt 또는 exception으로부터 돌아왔을 때 처리된다.
  - Pending list에는 type 당 1개의 signal만 가능.
  - <u>Signal이 얼마나 오래 pending 상태로 남아있을 지 예측할 수 없음!</u>

- Kernel에서의 Signal 구현

  1. 각 프로세스에서 어떤 signal이 block되었는지 기억해야 함

  2. Kernel → User mode로 전환할 때, 어떤 프로세스에 대한 signal이 도착했는지 확인(pending signal)

     - 거의 모든 timer interrupt.. 즉 대략 1ms마다 발생

     - ```assembly
       # /arch/i365/kernel/entry.S
       
       ...
       work_notifysig:
       	...
       	call do_notify_resume
       	...
       ...
       ```

     - ```c
       // /arch/i386/kernel/signal.c
       
       void do_notify_resume(struct pt_regs *regs, sigset_t *oldset, __u32 thread_info_flags)
       {
       	...
       	if (thread_info_flags & _TIF_SIGPENDING)
       		do_signal(regs, oldset);
       	...
       }
       ```

  3. 이 signal을 무시해도 되는지 결정

  4. Signal 처리

     - 프로세스의 실행 기간동안 특정 지점에서 프로세스를 handler function으로 전환
     - handler function이 끝나면 원래 실행중이던 context 복구

#### Data structures for Signal Handling

```c
// /include/linux/sched.h

struct task_struct {
    ...
    struct sigpending pending; // private pending signal의 리스트
	struct signal_struct *signal; // 프로세스의 shared signal descriptor의 포인터
	struct sighand_struct *sighand; // 프로세스의 signal handler descriptor의 포인터
    sigset_t blocked; // block된 signal의 비트마스크
    ...
};

struct sighand_struct {
    ...
    struct k_sigaction action[64];
    /*
    task->sighand->action[x].sa_handler 를 통해 signal handler 호출
    */
    ...
};

// /include/linux/signal.h

struct sigpending { // 위의 task_struct에 존재
    ...
    struct list_head list; // 리스트의 첫번째 원소
    sigset_t signal; // 리스트의 모든 signal의 비트마스크
    /*
    64비트 마스크(1~31은 Table 11-1에서, 32~63는 real-time signal)
    */
    ...
};

struct sigqueue {
    ...
    struct sigqueue *next; // 리스트에서 다음 element
    siginfo_t info; // 이 pending signal에 대한 정보
    ...
};

// /include/asm-i386/siginfo.h

struct siginfo {
    ...
    int si_signo; // signal 번호
    int si_code; // user, kernel, timer 중 누가 signal을 raise했는지
    // 이 외에 각 signal type마다 필요한 정보들...
    ...
} siginfo_t;


```

- **struct task_struct** (/include/linux/sched.h)
  - **struct sigpending** <u>pending</u>: private pending signal의 리스트
    - struct list_head list: 리스트의 첫번째 원소
    - sigset_t signal: 리스트의 모든 signal의 비트마스크
    - 64비트 마스크(각 비트의 자리가 해당 signal을 의미)
      - 1~31은 Table 11-1의 signal, 32~63은 real-time signal
  - struct signal_struct *<u>signal</u>: 프로세스의 shared signal descriptor의 포인터
  - **struct sighand_struct** *<u>sighand</u>: 프로세스의 signal handler descriptor의 포인터
    - struct k_sigaction action[64]
    - **task->sighand->action[x].sa_handler** 를 통해 signal handler 호출
  - sigset_t <u>blocked</u>: block된 signal의 비트마스크
- **struct sigqueue** (/include/linux/signal.h)
  - struct sigqueue *next: 리스트에서 다음 element
  - **struct siginfo_t** info: 이 pending signal에 대한 정보
    - /include/asm-i386/siginfo.h
    - int si_signo: signal 번호
    - int si_code: user, kernel, timer 중 누가 signal을 raise했는지
    - 이 외에 각 signal type마다 필요한 정보들...

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191114144700032.png" alt="image-20191114144700032" style="zoom: 67%;" />

#### Operations on Signal Data Structures

커널에서 signal을 처리하기 위해 사용되는 함수와 매크로들

- sigemptyset(set), sigfillset(set)
- sigaddset(set, nsig), sigdelset(set, nsig)
- sigaddsetmask(set, mask), sigdelsetmask(set, mask)
- sigismember(set, nsig)
- sigmask(nsig), sigtestsetmask(set, mask)
- sigandsets(d, s1, s2), sigorsets(d, s1, s2), signandsets(d, s1, s2)
- siginitset(set, mask), siginitsetinv(set, mask)
- signal_pending(p)
- recalc_sigpending_tsk(t), recalc_sigpending()
- rm_from_queue(mask, q)
- flush_sigqueue(q), flush_signals(t)

### Signal Transmission

1. Signal 생성(Signal Generation)
   - 커널은 새로운 signal이 전송되었음을 표현하기 위해 대상 프로세스의 descriptor를 업데이트한다. 
2. Signal 전달(Signal Action)
   - 커널은 대상 프로세스가 자신의 실행 상태를 변경하거나 특정 signal handler를 실행하거나 둘 다 하도록 강제함으로써 signal에 반응하도록 한다.

1. 

#### Signal Generation

- Signal handling의 첫번째 단계

  - 필요에 따라 process descriptor를 업데이트한다.
    - 커널 또는 다른 프로세스에서 signal을 프로세스로 보낼 때, 커널은 관련 기능을 통해 해당 signal을 생성한다.
  - signal의 종류와 프로세스의 상태에 따라 해당 함수들은 프로세스를 깨우고 강제로 signal을 받게 할 수도 있음

- Signal generation

  - 커널 또는 다른 프로세스에서 signal을 **특정 프로세스**에 보낼 때

    - 아래 테이블의 커널 함수들에 의해 **"specific_send_sig_info()"**가 호출됨

      ![image-20191115030106441](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191115030106441.png)

  - 커널 또는 다른 프로세스에서 signal을 **전체 스레드 그룹**에 보낼 때

    - 아래 테이블의 커널 함수들에 의해 **"group_send_sig_info()"**가 호출됨

      ![image-20191115030221688](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191115030221688.png)

- 코드 분석

  - **specific_send_sig_info(int <u>sig</u>, struct siginfo *<u>info</u>, struct task_struct *<u>t</u>)**

    - 매개변수
      - sig: signal 번호
      - info: 'real-time signal과 연관된 siginfo_t의 주소' 또는 '0'(유저 모드 프로세스에서 signal을 보낸 경우) 또는 '1'(커널에서 signal을 보낸 경우)
      - t: 대상 프로세스의 descriptor를 가리키는 포인터
    - 주요 기능
      - signal을 무시해도 되는지 체크
      - signal을 task에 전달: send_signal(sig, info, t, &t->pending) 호출
      - signal이 block되지 않았다면 signal_wake_up(t)를 호출하여 프로세스에게 새로 보류중인 signal이 있는지 알려준다.
      - 특별히 따로 처리하는 signal: SIGKILL, SIGCONT, SIGSTOP, SIGSTP, SIGTTIN, SIGTTOU

  - 함수 **send_signal(sig, info, t, &t->pending)**

    - "Signal을 task t의 pending list에 추가!"
    - Slab cache에 새 리스트 요소를 위한 메모리 할당
    - signal 정보를 채워서 리스트에 추가
    - pending signal bitmask(pending->signal)를 업데이트

  - 함수 **signal_wake_up(struct task_struct *t, int resume)**

    - 프로세스에게 새로운 active signal이 있다는 것을 알림

    ```c
    void signal_wake_up(struct task_struct *t, int resume){
    	unsigned int mask;
    	
    	set_tsk_thread_flag(t, TIF_SIGPENDING);
    	mask = TASK_INTERRUPTIBLE; // 0x0001
    	if (resume) mask |= TASK_STOPPED | TASK_TRACED;
    	if (!wake_up_state(t, mask)) kick_process(t);
    }
    ```

#### Signal Action

- 프로세스가 interrupt/exception으로부터 돌아올 때

  - ret_from_intr()/syscallexit 은 user mode로 전환하기 전에 block되지 않은 pending signal이 있는지 확인(TIF_SIGPENDING 플래그)한다.

  - 만약 TIF_SIGPENDING 플래그가 set 되어있다면 **do_signal()** 호출

    - arch/i386/kernel/signal.c::do_signal(struct pt_regs *regs, sigset_t *oldset)

    - 각 signal을 pending list로부터 deque한다. (block된 signal 제외)

    - signal을 전달하는 동안 수행할 수 있는 <u>3가지</u> 작업

      - Signal을 명시적으로 <u>무시</u>

      - Signal과 관련된 <u>default action 실행</u>

        - Terminate: 프로세스 종료(kill)
        - Dump: 프로세스가 종료되고, 이 프로세스에 대한 execution context를 담고 있는 core file 생성
        - Ignore: Signal 무시
        - Stop: 프로세스 중지(TASK_STOPPED 상태)
        - Continue: 만약 프로세스가 중지된 상태라면 이 프로세스를 TASK_RUNNING 상태로 만든다.

      - Signal에 해당하는 <u>signal-handler function을 호출</u>하여 signal을 catch: **handle_signal()**

        - user mode와 kernel mode 간 전환 시 스택의 전환이 매우 신중히 진행되어야 하므로 signal handler를 실행하는 것은 복잡한 작업이다!

          <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191116043419994.png" alt="image-20191116043419994" style="zoom: 33%;" />

        - Linux 해결책

          - <u>kernel mode stack에 저장된 hardware context를 현재 프로세스의 user mode stack에 복사한다.</u>
          - 반대로, user mode stack은 signal handler가 종료될 때 sigreturn() system call이 자동으로 호출되어 hardware context를 다시 kernel mode stack로 복사하고, user mode stack의 원래 내용을 복구하는 식으로 수정된다.(위와 반대 작업)

        - handle_signal()의 세부 기능

          - 프레임 세팅

          - signal flag 판별

          - signal handler 시작

          - signal handler 종료

            <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191116044226904.png" alt="image-20191116044226904"  />

    - SIGKILL과 SIGSTOP signal은 명시적으로 무시되거나 block되거나 catch될 수 없고, 그들의 default action은 반드시 실행되어야 한다.

    - do_signal() 함수는 block되지 않은 pending signal이 다 없어질 때까지 반복 실행된다.

### Real-time Signals in Linux

- POSIX 표준에 의해 정의됨
  - Signal number: 32 ~ 63
  - 같은 종류의 real-time signal이 여러 개가 queue될 수 있음
  - Linux kernel은 real-time signal을 사용하지 않지만, 몇가지 특정 system call을 통해 POSIX 표준을 완전히 지원함???
- Linux가 지원하는 것은 진정한 의미의 real-time signal이 아니다!
  - Linux에서의 "real-time signal"은 그저 <u>같은 종류의 signal을 pending signal list에 여러 개 허용하는 것</u> 뿐이다.

### System Calls Related to Signal Handling

- User mode에서 실행되는 프로그램
  - 다양한 종류의 system call에 의해 signal을 주고 받도록 허용됨
  - 예시
    - kill(pid, sig) → sys_kill() → kill_something_info() → kill_proc_info() 또는 kill_pg_info()
    - sigaction(sig, act, oact)
    - sigpending()
    - sigprocmask() → sys_sigprocmask(oset, set, how)
    - sigsuspend()
- Real-time signal을 위한 system call
  - rt_sigaction(), rt_sigpending(), rt_sigprocmask(), rt_sigsuspend()
  - rt_sigqueueinfo(), rt_sigtimedwait()

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191115040614678.png" alt="image-20191115040614678" style="zoom: 50%;" />