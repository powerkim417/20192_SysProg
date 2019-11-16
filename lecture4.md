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

- 초기화: arch/i386/kernel/time.c:**time_init()**
  - ```c
    static struct irqaction irq0 = {timer_interrput, SA_INTERRUPT, CPU_MASK_NONE, "timer", NULL, NULL};
    
    // 시스템 타이머를 위한 특정 초기화 수행
    void __init time_init_hook(void)
    {
        setup_irq(0, &irq0);
    }
    ```

  - ![image-20191116172404382](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191116172404382.png)

- Timer interrupt handler: **timer_interrupt()**

  - ```c
    /* /arch/i396/kernel/time.c */
    
    irqreturn_t timer_interrupt(int irq, void *dev_id, struct pt_regs *regs)
    {
        ...
        do_timer_interrupt(irq, NULL, regs); // PIT의 일반적 서비스 루틴 호출
        ...
    }
    
    static inline void do_timer_interrupt(int irq, void *dev_id, struct pt_regs *regs)
    {
        ...
        do_timer_interrupt_hook(regs);
        set_rtc_mmss(xtime.tv_sec); // 업데이트된 wall time을 RTC에 주기적으로 저장
        ...
    }
    
    /* /include/asm-i386/mach-default/do_timer.h */
    
    static inline void do_timer_interrupt_hook(struct pt_regs *regs)
    {
        do_timer(regs); // 시스템 시각을 업데이트하고, load 계산
        update_process_times(user_mode(regs)); // local CPU를 위한 time-related accounting 작업
        profile_tick(CPU_PROFILING, regs); // 커널 코드 프로파일링(성능 분석)
    }
    ```

  - timer_interrupt()는 real-time clock을 유지해야 하며, 이를 위해 매 clocktick마다 "do_timer()" 루틴을 호출해야 한다.

- arch/i386/kernel/timer.c:**do_timer()**

  - jiffies_64 변수를 1 증가시킨다.

  - 시스템 날짜와 시간을 업데이트하고 현재 시스템 로드를 계산한다.

  - ```c
    // Timer interrupt에 의해 호출됨.
    // xtime_lock이 timer IRQ에 의해 미리 설정된 상태여야 함
    
    static inline void update_times(void)
    {
        unsigned long ticks;
        
        ticks = jiffies - wall_jiffies;
        if (ticks){
            wall_jiffies += ticks;
            update_wall_time(ticks);
        }
        calc_load(ticks);
    }
    
    void do_timer(struct pt_regs *regs){
        jiffies_64++;
        update_time();
    }
    ```

- arch/i386/kernel/timer.c:**update_process_times()**

  - local CPU workload(accounting)에 대한 통계를 업데이트???

  - 만료된 dynamic timer를 실행한다.(TIMER_SOFTIRQ 활성화)

  - scheduler_tick()을 호출하여 현재 프로세스의 time slice counter를 감소시키고, 프로세스의 quantum이 다 소진되었는지 체크한다.

  - ```c
    // timer interrupt handler에서 호출되어 현재 프로세스에 하나의 tick을 부과
    // tick이 user time일 경우 user_tick=1, system time일 경우 user_tick=0
    
    void update_process_times(int user_tick)
    {
    	struct task_struct *p = current;
    	int cpu = smp_processor_id();
    	
    	if (user_tick)
    		account_user_time(p, jiffies_to_cputime(1));
        else
            account_system_time(p, HARDIRQ_OFFSET, jiffies_to_cputime(1));
        run_local_timers();
        if (rcu_pending(cput)) rcu_check_callbacks(cpu, user_tick);
        scheduler_tick();
    }
    ```

#### Updating the Time and Date

- Wall clock(현재 시간) 관리
  - 자료구조: **struct timespec xtime**
    - xtime.tv_sec: 1970년 1월 1일(UTC)로부터 지난 초의 수를 저장
    - xtime.tv_nsec: 마지막 초로부터 지난 nanosecond의 값을 저장
  - kernel은 update_times() 함수를 통해 xtime을 매 틱마다 업데이트한다.
- **gettimeofday()**
  - Wall clock time을 받아오기 위한 기본 user-space function
  - sys_gettimeofday system call에 의해 구현됨
  - Wall clock time은 주로 user-space 프로그램과 관련됨
  - 커널 자체에서 wall clock time이 필요한 경우는 파일 시스템 관련 작업을 할 때!
    - 파일 생성/접근/수정을 위해 inode???에 timestamp 저장

#### Updating System Statistics

커널은 주기적으로 여러 데이터들을 수집해야 한다.

- <u>커널 코드 프로파일링(성능 분석)</u>

  - Kernel의 hot spot(자주 실행되는 커널 코드 조각) 식별
  - do_timer_interrupt()에 의해 호출되는 <u>profile_tick()</u>에 의해 수집된다.
    - eip 레지스터 값을 체크

- <u>시스템 로드 트래킹</u>

  - 매 tick마다 update_times()에 의해 호출되는 <u>calc_load()</u>에 의해 시스템 로드가 수집된다.
  - TASK_RUNNING 또는 TASK_UNINTERRUPTABLE 상태의 프로세스의 수를 세고, 이를 평균 시스템 로드 업데이트에 반영한다.

- <u>CPU 사용량 통계 업데이트</u>

  - update_process_times()에 의해 호출되는 <u>account\_user\_time() 또는 account\_system\_time()</u>에 의해 수집된다.

  - **Interval Counting**

    - 커널은 프로세스가 유저 모드에서 실행된 tick의 수(task_struct.utime)와 커널 모드에서 실행된 tick의 수(task_struct.stime)를 통해 book-keeping??? 정보 유지

    - "Interval Counting"은 조잡한 방법!

      - 각 timer interrupt가 발생했을 때 실행중인 프로세스에게 tick을 부여하는 방식으로, 실제 실행 시각과는 조금 차이가 있다.

        ![image-20191117023007795](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191117023007795.png)

#### Supporting Software Timers

- Software Timer
  - 주어진 시간 간격이 경과한 후에(time-out) 함수를 호출할 수 있게 하는 소프트웨어 기능
- Software timer의 종류
  - Dynamic timer( = kernel timer = kernel event timer)
    - 커널에 의해 사용되며, 대부분 장치 드라이버에서 비정상 상태를 감지하기 위해 사용된다.
  - Interval timer
    - 프로그래머(유저 프로세스)에 의해 사용되며, system call을 통해 나중에 특정 함수가 실행되는 것을 강제하기 위해 사용된다.
- Linux software timer의 한계
  - **Timer function을 체크하는 작업이 bottom half에서 이뤄지기 때문에, 커널은 timer가 만료된 정확한 시점에 timer function이 호출된다는 보장을 하지 못한다.**
  - 따라서 **real-time 어플리케이션에는 부적합!**

##### Implementing Dynamic Timers

- Dynamic Timer의 특징

  - 특정/사전정의된 시각에 함수 호출이 스케줄됨
  - 동적으로 생성/삭제됨
  - 활성화된 dynamic timer의 갯수의 제한이 없음

- Dynamic timer를 저장하는 자료구조: struct timer_list

  - ```c
    // /include/linux/timer.h
    
    struct timer_list {
    	struct list_head entry; // 삽입할 timer vector list
    	unsigned long expires; // dynamic timer 만료 시각(HZ단위) 저장
    	spinlock_t lock;
    	unsigned long magic;
    	void (*function)(unsigned long); // 만료 시 실행되는 함수와 인자
    	unsigned long data;
    	struct tvec_t_base_s *base;
    };
    ```

- Dynamic timer 생성 및 활성화

  1. 새로운 struct timer_list 객체를 local 또는 global variable로 생성
  2. 객체를 init_timer()로 초기화
  3. 객체의 "function"과 "data" 필드를 로드하여 세팅
  4. add_timer()를 호출하여 객체를 kernel timer list에 삽입

- 만료되기 전에(timer list에 있는 동안) 할 수 있는 작업

  - 타이머 재스케줄: mod_timer()
  - 타이머 삭제: del_timer_sync(), del_timer()

- 사용 예시) chedule_timeout(): 다양한 장치 드라이버에 의해 사용됨

  - 현재 프로세스를 2초간 중단할 때: schedule_timeout(2*HZ);

    - HZ: tick과 tick 사이의 시간 단위

  - ```c
    static void process_timeout(unsigned long __data)
    {
    	wake_up_process((task_t *)__data);
    }
    
    // timeout이 될 때까지 현재 task를 sleep상태로 만듦
    // /kernel/timer.c
    fastcall signed long __sched schedule_timeout(signed long timeout)
    {
        struct timer_list timer;
        unsigned long expire;
        ...
        expire = timeout + jiffies; // jiffies: system boot 이후 timer interrupt(=tick)의 횟수
        
        init_timer(&timer);
        timer.expires = expire;
        timer.data = (unsigned long) current;
        timer.function = process_timeout;
        
        add_timer(&timer);
        schedule();
        del_singleshot_timer_sync(&timer); // dynamic timer의 큰 틀 중 만료되기 전 할 수 있는 작업 부분에 해당
        
        timeout = expire - jiffies; // 코드가 실행되는 동안 jiffies가 바뀌나???
        
        return (timeout<0) ? 0 : timeout;
    }
    ```

- Dynamic timer 체크 및 실행: TIMER_SOFTIRQ

  - 각 timer interrupt마다 softirq를 raise
    - update_process_times()::run_local_timers() 에서 raise_softirq(TIMER_SOFTIRQ) 실행
  - 실행
    - kernel/timer.c:run_timer_softirq()가 현재 프로세스의 모든 만료된 타이머를 실행한다.

- 구현 관련 문제

  - 매 tick마다 dynamic list 전체를 스캔하는 것은 매우 많은 비용(시간) 소모!

  - 해결책: **tvec_base_t** 자료구조(kernel/timer.c에 존재)

    - 이벤트를 만료 시각에 따라 512개의 리스트로 그룹화

      - 첫 256개 리스트: 다음 1, 2, ..., 256 tick 이후에 만료되는 이벤트
      - 다음 64개 리스트: 다음 1\*2^8, 2\*2^8, ..., 64\*2^8 tick 이후에 만료되는 이벤트
      - 다음 64개 리스트: 다음 1\*2^14, 2\*2^14, ..., 64\*2^14 tick 이후에 만료되는 이벤트
      - 다음 64개 리스트: 다음 1\*2^20, 2\*2^20, ..., 64\*2^20 tick 이후에 만료되는 이벤트
      - 마지막 64개 리스트: 다음 1\*2^26, 2\*2^26, ..., 64\*2^26(=2^32) tick 이후에 만료되는 이벤트

    - 5개의 그룹을 각각 tv1, tv2, tv3, tv4, tv5 라는 포인터로 식별

      <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191117042157489.png" alt="image-20191117042157489" style="zoom: 80%;" />

      - tv1은 index 필드와 vec 배열(timer_list 요소에 대한 256개의 포인터) 존재
      - (index+k)번째 리스트는 정확히 k회의 tick 이후에 만료되는 모든 dynamic timer를 담고 있는 리스트가 된다.
        - index: 매 tick마다 1씩 증가(mod 256)
        - 매 256 tick마다 tv1에 있는 모든 타이머가 스캔됨
      - index가 0을 반환하면, tv2.vec[tv2.index]를 가리키는 리스트는 tv1를 채우는 데 사용된다???

      ```c
      struct tvec_t_base s {
      	spinlock_t lock;
      	unsigned long timer_jiffies;
      	struct timer_list *running_timer;
      	tvec_root_t tv1;
      	tvec_t tv2;
      	tvec_t tv3;
      	tvec_t tv4;
      	tvec_t tv5;
      } ____cacheline_aligned_in_smp;
      typedef struct tvec_t_base_s tvec_base_t;
      ```

#### Delaying Execution

- 상황

  - 커널이 수 ms 보다도 작은 짧은 시간동안 기다려야 하는 경우 software timer가 쓸모 없어진다!
  - Dynamic timer는 상당한 설정 오버헤드와 다소 큰 최소 대기 시간(1ms)이 존재
  - 커널 코드(Ex. 드라이버)는 timer를 사용하지 않고 실행을 한동안 delay시킬 필요가 있다.
    - 주로 특정 task를 수행하기 위해 하드웨어에게 추가 시간을 주는 것과 같은 짧은 delay
      - Ex) 이더넷 카드의 속도를 설정한 후 계속 하기까지 2ms의 시간이 소요됨
  - 문제점: jiffie 기반 delay는 그 단위가 매우 커진다(ms)
    - 해결책: "Small delay loop"을 통해 커널 실행을 delay한다.

- Small delay loop

  - 커널은 ms delay와 ns delay에 대한 각각의 함수를 제공하며, 이 함수들은 특정 시간 간격동안 정확히 동작한다.

    - void udelay(unsigned long usecs), void ndelay(unsigned long nsecs)
    - Ex) udelay(150); // 150ms 동안 delay

  - 구현

    - 시스템 부팅 중 보정

      - calibrate_delay(): CPU가 실행할 수 있는 spinning loop 반복의 수(=BogoMIPS)를 결정한다(loops_per_jiffy 변수에 저장되어 있음).

        ![image-20191117043804830](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191117043804830.png)

    - delay function은 이 값을 사용하여 필요한 실제 delay 간격을 위한 iteration의 횟수를 계산한다.

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