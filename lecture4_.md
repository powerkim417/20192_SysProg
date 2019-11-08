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
  - Timer interrupt frequenc를 증가시키는 것은 timer interrupt를 더 자주 발생시킨다는 것을 의미
  
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



### 2. Kernel Data Structures and Functions to measure time

#### Linux Timekeeping Architecture

#### Timer Interrupt Handling

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
  - **settimeofday()**
  - **adjtimex()**
- 유저 모드에 '인터벌 타이머' 제공
  - 프로세스에 <u>주기적으로</u> UNIX 신호 전송(periodic) 또는 특정 딜레이 이후 <u>하나의</u> 신호 전송(single-shot)
  - POSIX settime(), alarm() system call을 통해 활성화됨
  - **setitimer()**
    - Interval timer의 값을 설정
      - Interval timer의 종류: ITIMER_REAL ,ITIMER_VIRTUAL, ITIMER_PROF
  - **alarm()**
    - 특정 시간 간격이 경과했을 때 호출 프로세스에 SIGALRM 신호를 보냄