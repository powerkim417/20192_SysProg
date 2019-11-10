# Lecture 4. Timing Measurements

## Computer Time

- 많은 컴퓨터화된 작업들은 시간을 중심으로 돌아간다
  - OS에 시간에 대한 개념이 있어야 함!
- Time-driven activity의 예시
  - 주기적으로 소프트웨어 업데이트를 확인하는 작업
  - 일정 시간 유저가 활동하지 않을 경우 컴퓨터 스크린을 끄는 작업
  - 일정 주기마다 사용자의 비밀번호를 변경하라고 권고하는 작업
  - 프로세스의 time quantum을 트래킹하고 프로세스 스케줄링을 하는 등의 작업
  - time-out 처리(네트워크 연결, 하드웨어 디바이스, ...)
- Linux Kernel에서의 주요 시간 관련 서비스
  - 시스템 업타임 유지
  - 바탕화면 시간 유지
  - 특정 시각에 트리거를 수행하는 메커니즘 제공
    - 타이머는 알람시계와 같이 커널이나 유저 프로그램에 특정 시간 간격이 지났음을 알린다!

![image-20191028131319659](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028131319659.png)

### Kernel(OS) 에서의 Time 개념

- System uptime: 시스템 시작 이후 경과한 시간
  - 컴퓨터에서 시간의 흐름은 연속적이지 않고 discrete하다..
  
  - **Computer time t_c** 는 discrete timer에 의해 업데이트 됨
  
  - 연속적인 시간을 전부 담지 못하고 아주 작은 단위에 의해서라도 쪼개져서 표현된다는 뜻인듯
  
    ![image-20191030173839198](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191030173839198.png)
  
- 하드웨어는 system timer(=Kernel timer = PIT)를 제공한다

  - **Tick rate**라는 사전에 프로그래밍된 주파수만큼 지나고 꺼진다.(goes off)
  - 타이머가 꺼지면 system timer는 인터럽트를 issue한다.
  - Linux kernel은 timer interrupt를 처리하는데, 이 때 "t_c++"가 발생!

- 위의 두 interrupt 간의 간격을 **tick**이라고 부름

  - tick = (1 / tick_rate) (s)
  - 시간의 경과는 두 uptime(t_c)의 차이엥서 온다.

### Timer Interrupt Frequency

Timer interrupt frequency = tick rate

- 주파수에 따른 장단점
  - 타이머의 주파수가 더 높을 수록 시간이 더 세분화(finer granularity)되지만, 그에 따른 오버헤드가 증가하게 된다.

  - 반대로, 타이머의 주파수가 낮을 수록 오버헤드는 줄어들지만 시간을 세분화해서 표현하기 힘들어진다.

    ![image-20191031165820783](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191031165820783.png)

    - 그림에 표시된 간격동안 일어난 모든 일은 커널에서 '같은' 시간에 발생한 것으로 간주!

- Tick rate는 HZ 매크로에 의해 정의된다.

  - i386에서는 #define HZ 1000 으로 정의됨
  - Timer interrupt frequency를 증가시키는 것은 timer interrupt를 더 자주 발생시킨다는 것을 의미
  
- Timer interrupt frequency(=tick rate)를 증가시키는 경우

  - Timed event에 대한 resolution(분해능)이 높아짐
    - Ex) Kernel 2.6에서 tick rate가 100에서 1000으로 증가함에 따라 시간을 10ms가 아닌 1ms 단위로 측정할 수 있게 됨
  - Timed event에 대한 정확도가 증가
    - Ex) 위의 예시에서 특정 작업을 10ms로 세분화하는게 아닌 1ms로 세분화하여 실행할 수 있게 됨
  - 단점: timer interrupt가 더 자주 발생하므로 **timer interrupt handler에 더 많은 시간이 소모**되고, 그로 인해 **유의미한 연산을 할 수 있는 시간이 줄어든다**.

- **Jiffies**: system boot 이후 timer interrupt(=tick)의 횟수

  - 부팅이 일어나는 동안 jiffies는 0이다.
  - Jiffies는 각 timer interrupt 동안에 증가한다.
  - Jiffies는 32비트 변수!
    - 1000Hz의 timer interrupt frequency일 때, jiffie 변수는 50일이 지났을 때 overflow가 발생한다.
  - Overflow를 방지하기 위한 64비트 확장으로는 jiffies64 를 사용하며, 기존의 32비트 값은 0~31에 들어간다.
    - get_jiffies64() 함수는 xtime_lock를 사용하여 전체 64비트 값을 읽어온다
      - timer interrupt와 두 읽기 작업(jiffies_64 변수의 상위/하위 워드를 읽는 작업) 간의 race condition(경합 조건)을 방지하기 위해 32비트 아키텍쳐에서는 lock이 필요!

### Timing Measurements

- (Remind) 리눅스 커널이 할 수 있어야 하는 것들(what linux kernel should perform)
  - 현재 시간과 날짜를 계속 유지(Wall clock time)
  - 현재 프로세스의 실행 시간 결정
  - 리소스 통계 업데이트
  - 커널 또는 유저 프로그램에 일정 시간간격이 경과했음을 알려주기 위한 타이머 유지

#### 구성 요소

- <u>Hardware Clock Devices</u>(**하드웨어** 시계 장치)
  - Real Time Clock: RTC
  - CPU Cycle Counter: TSC(Time Stamp Counter) Register
  - Kernel Clock: Programmable Interval Timer(PIT)
- <u>Kernel Data Structures and Functions to measure time</u>(시간 측정 **커널 자료구조/함수**)
- <u>Timing-related System Calls</u>(시간 관련 **system call**)

### 1. Hardware Clock Devices

#### Real Time Clock(RTC)

- PC가 꺼져 있어도 계속 시간을 트래킹한다(tick이 계속됨)
  - 마더보드의 배터리에 의해 전원이 유지되기 때문!
  - Ex) Motolora MC146818P
- IRQ8에서는 2Hz~8192Hz로 주기적으로 인터럽트가 발생하거나, RTC가 특정 값에 도달하면 인터럽트가 발생하도록 프로그래밍됨(알람시계처럼!)
- <u>Linux는 부팅하는 동안에만 RTC를 사용하여 시간과 날짜를 받아온다.</u>
  - 부팅하는 동안 커널은 RTC를 읽어와 wall clock time을 초기화됨
  - Wall clock time은 xtime 변수에 의해 유지됨
  - /dev/rtc 디바이스 파일에 의해 RTC를 프로그래밍함
  - /sbin/clock 프로그램에 의해 clock을 세팅함

#### Time Stamp Counter(TSC Register)

- 매우 정밀한 시간 측정에 사용됨
  - 시간 세분도(Granularity): CPU 사이클(Ex. 1GHz)
- Ex) Intel x86: 64-bit TSC Register
  - 하드웨어에 의해 업데이트됨(각 clock 신호마다 증가함)
  - 모든 Intel CPU는 외부 오실레이터로부터 clock 신호를 받아오는 CLK 입력핀을 가지고 있음
  - rdtsc 어셈블리어 명령어를 통해 읽어옴
    - 400MHz(<u>4\*10^8</u>Hz)이면 TSC는 2.5ns(2.5\*10^(-9) = 1/<u>4*10^8</u>)마다 증가한다.
- <u>Linux는 PIT의 시간 측정보다 훨씬 더 정확한 시간 측정을 얻기 위해 TSC의 장점을 활용!</u>

#### Programmable Interval Timer(PIT)

- 커널이 시간을 유지할 수 있게 함
- <u>PIT는 timer interrupt라는 특수한 인터럽트를 발생시킴</u>
  - <u>timer interrupt는 커널에게 한번 더 time interval이 경과했음을 알림</u>
- Ex) legac Intel 8254 CMOS 칩
- Linux는 PIT가 1000Hz 주파수로(1ms마다) IRQ0에 timer interrupt를 발생시키도록 프로그래밍한다.
  - 커널 버전 2.6 이전에는 100Hz로 timer interrupt를 발생시켰음
  - Linuz HZ 매크로: 초당 timer interrupt 발생 횟수 = Timer interrupt frequency
    - IBM PC에서 HZ=1000

### 2. Kernel Data Structures and Functions to measure time

#### Linux Timekeeping Architecture

- 시스템 부팅 중
  - 커널은 wall clock time을 초기화하기 위해 RTC를 읽어온다.
  - wall clock time은 xtime 변수에 의해 유지됨
- PIT에 의해 Kernel timer interrupt가 발동됨
  - Interrupt handler + bottom half (softirq)
- 커널이 Timer interrupt handler에서 하는 일들
  - jiffies_64 변수 값을 1 증가시킴
  - xtime 변수에서의 wall clock time을 갱신
  - 리소스 사용 통계 갱신
    - 시스템 시간 또는 user mode 시간의 마지막 tick을 현재 프로세스에 book-keep???
  - 만료된 dynamic timer를 실행하기 위해 softirq를 raise
  - scheduler_tick() 실행
    - 현재 프로세스의 time slice를 감소시키고, 필요한 경우 TIF_NEED_RESCHED 플래그를 set

#### Timer Interrupt Handling(Uniprocessor)

- arch/i386/kernel/time.c:**time_init()**
- **timer_interrupt()**
- kernel/timer.c:**do_timer()**
- kernel/timer.c:**update_process_times()**

#### Updating the Time and Date

#### Updating System Statistics

#### Usage Statistics via Interval Counting

#### Supporting Software Timers

#### Implementing Dynamic Timers

#### Delaying Execution

### 3. Timer-Related System Calls

몇몇 system call은 유저 모드 프로세스가 시간 및 날짜를 읽고 수정하고, 타이머를 생성할 수 있도록 허용한다.

- 시간 및 날짜 읽기 및 수정
  - **gettimeofday()**
    - timeval 이라는 자료구조를 반환
      - 1970.1.1 부터 현재까지 지난 초의 값과 마지막 초로부터 지난 microsecond의 값
        - Ex) 12345.6789초가 지났으면 12345와 678900을 반환하는듯?
  - **settimeofday()**
    - 현재 날짜와 시간을 수정
    - root 권한만 가능!!!!
  - **adjtimex()**
    - xtime을 사용하여 시계를 조율(keep the clock tuned)
    - NTP(네트워크 시간 프로토콜)와 같은 시간 동기화 프로토콜을 실행하도록 구성됨
- 유저 모드에 '인터벌 타이머' 제공
  - 프로세스에 <u>주기적으로</u> UNIX 신호 전송(periodic) 또는 특정 딜레이 이후 <u>하나의</u> 신호 전송(single-shot)
  - POSIX settime(), alarm() system call을 통해 활성화됨
  - **setitimer()**
    - Interval timer의 값을 설정
      - Interval timer의 종류(3종류): ITIMER_REAL ,ITIMER_VIRTUAL, ITIMER_PROF
    - ITIMER_REAL은 dynamic timer를 사용하여 구현한다.
      - 어느 타이머가 끝나면 SIGALRM 신호를 프로세스로 전달하고, 타이머는 잠재적으로 재시작됨
  - **alarm()**
    - 특정 시간 간격이 경과했을 때(특정 타이머에서 지정한 시간이 다 끝났을 때) 호출할 프로세스에 SIGALRM 신호를 보냄