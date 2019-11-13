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
  - Blocked signal의 mask

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

#### Operations on Signal Data Structures

### Signal Transmission

#### Signal Generation

#### Signal Action

### Real-time Signals in Linux

### System Calls Related to Signal Handling