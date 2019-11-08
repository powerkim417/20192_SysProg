# Lecture 3. Interrupts and Exceptions

## Interrupt

특정 action을 break하는 행위

- 예상치 못한 드문 event를 효율적으로 처리할 수 있는 방법 제공
  - Polling(충돌 회피/동기화 등의 목적으로 프로그램의 상태를 주기적으로 검사) 보다 나음!
- Ex) Processor interrupt의 경우
  - 프로세스에 의해 실행된 명령어의 순서를 변경함
  - CPU를 일반적 제어흐름 외의 코드로 전환시키는 방법을 제공함
  - 즉, 프로세서 인터럽트는 하드웨어(NIC, 프린터, 디스크, 마우스, 키보드, ...)와 CPU 사이의 통신 수단!

### Taxonomy(분류)

#### Exception

= <u>Synchronous</u> interrupt

= <u>Software</u> interrupt

- 명령어를 실행하는 도중에 **CPU**에 의해 발생한다.
  - Programming error (0으로 나누기, segmentation fault, ...)
  - Anomalous conditions (페이지 폴트, 부동 소수점 오류, ...)
  - System call (kernel service를 위한 프로세스의 요청)
  - 디버거

#### Interrupt

= <u>Asynchronous</u> interrupt

= <u>Hardware</u> interrupt

- CPU 외부의 **하드웨어 장비**에 의해 발생한다.
  - Timer interrupt, I/O devices (keystroke, ...), ...
  - CPU의 interrupt pin을 설정하여 표시

### Linux Kernel에서의 인터럽트 제어

인터럽트 신호 발생 시

- CPU는 현재 제어 흐름을 멈추고 새로운 activity로 jump한다(Interrupt handler).
- Kernel stack에 현재의 hardware context를 저장하고, 인터럽트와 관련된 주소를 Program counter에 배치한다.

Interrupt handler는 프로세스가 아니고 인터럽트가 발생하였을 때 실행중이었던 동일한 프로세스를 대신 실행시키는 커널 제어 경로(KCP) 이다! (즉, Interrupt handler는 interrupt context에서 실행됨)

### Process Context VS Interrupt Context

#### <span style='color:blue'>Process context</span>

- <span style='color:blue'>**정의: "커널이 프로세스를 대신하여 실행하고 있는 동안의 작동 모드"**</span>
  - Ex) System call 실행, 커널 스레드 실행
- 프로세스가 Process context에서 .커널과 결합되기 때문에, process context는 스케줄러를 sleep 상태로 돌리거나 호출할 수 있다.
- 스케줄링이 가능한 모든 것들은 process context로 간주됨

##### Kernel thread

- 커널 스레드도 process context에서 실행된다!
- 커널 스레드는 user space의 반대부분을 소유하지 않는 실행의 context으로 간주된다.
- 커널 스레드는 명확한 전환 가능한 자기자신과 연관된 context이므로 스케줄링이 가능하다!???

#### <span style='color:red'>Interrupt context</span>

- <span style='color:red'>**정의: "커널이 하드웨어 인터럽트에 의해 발동된 인터럽트 핸들러를 실행하고 있는 동안의 작동 모드"**</span>
- 인터럽트 핸들러는 어느 특정 프로세스의 context로 실행되는 것이 아님(현재 프로세스와 연관 X)
  - <u>즉, 인터럽트 핸들러는 스케줄링이 불가능하고, 사용에 제한이 있다!</u>
- Interrupt handler의 한계
  - Sleep / block 불가능: schedule() 호출이 불가능
  - Blocking lock 없음???: semaphore가 없기 때문에 spinlock을 사용(spin_lock_irqsave())
  - blocking KMA 없음???: kmalloc(size, GFP_KERNEL)이 없기 때문에 non-blocking KMA인 kmalloc(size, GFP_ATOMIC)을 사용
  - User space에 접근할 수 없음

#### 예시

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570519610996.png" alt="1570519610996" style="zoom:67%;" />

1. 프로세스가 exception(system call, page faule, program error)을 발동시켜 kernel space에 들어간다. 이 때 커널 제어 경로가 현재 프로세스를 대신 실행한다.
2. 인터럽트 핸들러는 프로세스를 대신하여 실행하지 않음
   - 인터럽트 핸들러는 현재 프로세스와 연관되어 있지 않음!

## Interrupt Handling

#### Requirements

- 커널 효율성
  - 인터럽트는 어느 때나 발생할 수 있다. (커널이 자신이 하려던 다른 일을 끝내고 싶을 때)
  - <u>그러므로 커널의 목표는 가능한 빨리 인터럽트를 끝내고(Top-half), 최대한 많은 processing을 지연시키는 것이다(Bottom-half).???</u>
- 다중 입출력에 대한 효율적인 핸들링
  - 커널은 기존의 인터럽트 신호를 처리하는 동안 새 인터럽트 신호를 받을 수 있다(I/O device를 busy하게 유지하기 위해)
  - <u>그러므로 커널 제어 경로는 중첩된 방식으로 실행되어야 한다.</u>

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570522670558.png" alt="1570522670558" style="zoom:67%;" />

IRQ: 인터럽트 요청 / iret: 인터럽트로부터의 복귀

- 커널 동기화
  - <u>임계 구역은 인터럽트가 비활성화되는 커널의 내부에 위치한다.</u>
    - 이러한 구역은 시스템의 응답성을 유지하기 위해 최대한 작게 유지되어야 한다.

#### 인터럽트 핸들링 하드웨어/소프트웨어 구조

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\1570523253254.png" alt="1570523253254" style="zoom:67%;" />

##### 하드웨어 부분

하드웨어 → Interrupt controller → Processor

프로세서가 kernel에 interrupt를 준다.

##### 소프트웨어(Kernel) 부분

do_IRQ() // 인터럽트 요청
↓
인터럽트 핸들러가 현재 라인???에 존재한다면 handle_IRQ_event() 를 호출하고 현재 라인???의 모든 인터럽트 핸들러를 실행시킨다.
↓
ret_from_intr() // 인터럽트로부터의 복귀
↓
interrupt된 kernel code로 복귀한다.

### Interrupt Handling: Hardware Part

- Ex) Intel 32 bit Architecture - IA32 Registers

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027144739430.png" alt="image-20191027144739430" style="zoom:67%;" />

#### IRQs and Interrupt Controller

##### Hardware Device Controller

- Interrupt request를 실행할 수 있는 디바이스 컨트롤러에 output line이 있음(Ex. IRQ)
  - 하드웨어 인터럽트의 종류: Timer, I/O, IPI(Inter-processor)
- IRQ line들은 PIC(Programmable Interrupt Controller)라는 하드웨어 회로의 input pin에 연결되어 있다.

##### Interrupt Controller(PIC)

1. IRQ 라인을 모니터링하면서 raised signal을 확인한다
2. Raised signal이 발생하면
   1. Raised signal을 이에 해당하는 벡터로 변환한다.
   2. 변환한 벡터를 PIC의 I/O 포트에 저장한다.(CPU가 data bus를 통해 이를 읽게 하기 위해)
   3. Raised signal을 프로세서의 INTR pin에 전달한다.(Ex. issues an interrupt)
   4. 한 PIC의 I/O 포트에 입력이 발생한 것을 통해 CPU가 interrupt signal을 인식할 때까지 기다린다. 그 후, INTR 라인을 clear한다.

- Legacy Interrupt Controller (8259A PIC)
  - Intel의 8259A PIC
    - 프로그래밍이 가능하며, 일반적으로 단일프로세서에 사용됨
    - CPU의 INTA pin 또는 I/O 포트를 통해 ACK signal을 수신
    - 전통적 PIC는 2개의 8259A 칩을 연쇄적으로 연결하여 구현함
      - 최대 가능 IRQ 라인은 15개(16개 아님???)
  - 한정된 수의 IRQ 라인 처리
    - IRQ sharing: 하나의 IRQ 라인에 여러 ISR
    - IRQ dynamic allocation: IRQ 라인을 임시적으로 사용

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027155706734.png" alt="image-20191027155706734"/>

- Advanced(Modern) Interrupt Controller (APIC)
  - I/O APIC
    - external interrupt를 하나 이상의 local APIC에 보내줌
    - 칩셋에 존재(PC architecture의 'south bridge' 부분)
  - Local APIC
    - I/O APIC 에서 interrupt를 수신하고 CPU로 보내줌
    - IPI(Inter processor Interrupts)를 송수신함
    - local interrupt 또한 수신함(thermal sensor, internal timer, ...)

![image-20191027160500761](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027160500761.png)

#### IA32: Interface to Hardware Interrupts

- Maskable interrupts (CPU의 INTR 핀)
  - I/O 디바이스에 의해 발생된 모든 IRQ는 maskable interrupt를 발생시킨다.
  - EFLAGS CPU 상태레지스터의 IF 플래그를 set/reset
  - sti(set interrupt) / cli(clear interrupt)
- Non-maskable interrupts (CPU의 NMI 핀)
  - 무시되지 않음!
  - 소수의 중요한 이벤트(hardware failure, memory error, ...)만이 non-maskable interrupt를 발생시킨다.
- 인터럽트 인식
  - PC: 8259A I/O 포트에 byte를 보냄으로써 IRQ를 인식한다.
    - Microcontroller: INTA 핀들 사이에 직접???
  - EOI(End Of Interrupt): PIC가 현재 인터럽트를 clear하고 다른 인터럽트를 issue할 수 있도록 PIC에 주는 명령어

#### IA32: Architectural Support for Interrupts Handling

- Interrupt Descriptor Table(IDT)
  - 8바이트의 interrupt descriptor로 표현된 interrupt handler
  - 가능한 256개의 인터럽트 각각에 대하여 1개의 엔트리를 포함하고 있다.
  - IDT는 interrupt handler entry point(진입점)에 interrupt vector를 매핑
  - interrupt vector는 IDT에서 index로 사용됨
  - idtr 레지스터: IDT가 메모리의 어디든 위치할 수 있도록 허용함(어디든 가리킬 수 있게?)
- x86은 3종류의 interrupt descriptor(=gate descriptor)를 제공해줌
  - Interrupt gate: Interrupt handler 에서의 segment selector + offset
    - EFLAGS의 IF 플래그를 clear하여 이후의 maskable interrupt를 비활성화함
  - Trap gate: Exception handler 에서의 segment selector + offset
  - Task gate: 프로세스의 TSS(Task state segment) selector. 리눅스에선 사용하지 않음

![image-20191027164417274](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027164417274.png)

#### IA32: "Hardware Handlings" of Interrupts

- (A)PIC의 interrupt vector number n을 결정
- idtr 레지스터에서 가리키는 IDT의 n번째 엔트리를 읽어 interrupt handler entry point의 주소를 결정
  - n번째 entry는 gate descriptor(=segment selector + offset)를 포함
  - gdtr 에서 GDT의 base address를 얻고, GDT에서 IDT의 segment selector에 의해 식별되는 segment descriptor를 읽는다.
- (User mode인 경우) kernel mode로 전환한다!
  - TSS로부터 새로운 privilege level의 ss(stack segment)와 esp(extended stack pointer)를 로드함
    - TSS는 x86 전용 세그먼트로, 커널 스택 등의 주소를 저장함 
  - 커널 스택에서의 이전 ss, esp 값들을 저장
  - 커널 스택의 eflags, cs, eip 값을 저장
  - cs+eip를 IDT의 n번째 엔트리에 저장된 gate descriptor의 'segment selector + offset'과 함께 로드
    - interrupt handler로 jump!
- iret을 통해 interrupt handler로부터 복귀한다.
  - 이전에 저장된 stack의 cs, eip, eflags를 복구
  - handler의 CPL을 확인
  - 스택으로부터 ss와 esp를 복구
  - privilege level을 처리하기 위해 ds, es, gs를 검사

**프로세스의 전환은 kernel mode 작업이 끝날 때 일어나며, switch_tomacro는 kernel mode에서 실행되는 프로세스의 hardware context(esp, eip, ...)를 전환한다!**

**하드웨어에서의 인터럽트 처리는 user mode와 kernel handler 사이에서 hardware context(esp, eip, ...)를 전환하는 것인데, 이러한 전환이 nested kernel control path에서 발생하지 않았을 경우 이를 "mode switching" 이라고 부른다!**

### Interrupt Handling: Software(Kernel) Part

#### Interrupt Vectors in Linux

- 각 interrupt(와 exception)은 0~255 사이의 수로 식별된다.
  - linux는 이 중 128을 system call로 사용

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027204756112.png" alt="image-20191027204756112" style="zoom:80%;" />

#### Data Structures for Interrupt Handlings

- 각 IRQ를 처리하기 위해 다양한 정보를 저장
  - 주요 자료구조: irq_desc_t(IRQ descriptor)
  - 왜 IDT, IRQ descriptor를 사용하는가?
    - IRQ descriptor가 리눅스에서 사용하는 정보를 더 많이 담음!(Ex. IRQ list)
  - IRQ action: 각 IRQ handler 당 1개 있음(struct irqaction)

```c
typedef struct irq_desc {
    hw_irq_controller *handler; // PIC hardware를 가리킴
    void *handler_data; // PIC 메서드에 의해 사용되는 데이터
    struct irqaction *action; // IRQ action list
    unsigned int status; // IRQ 상태 플래그
    unsigned int depth; // nested interrupt state
    unsigned int irq_count; // 인터럽트 발생 카운터
    unsigned int irqs_unhandled; // 처리하지 않은 인터럽트 카운터
    spinlock_t lock;
} ____cacheline_aligned irq_desc_t;
```

![image-20191027211133381](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027211133381.png)

```c
struct irqaction {
	irqreturn_t (*handler)(int, void *, struct pt_regs *); // 등록된 interrupt handler function
    ...
}
```

#### Interrupt Handler 등록

- request_irq() 호출을 통해 interrupt line을 할당한다.

  - 주로 디바이스 드라이버에 의해 사용됨: $ grep -r request_irq() 를 통해 확인 가능

  - Interrupt handler를 등록하고, interrupt line을 활성화

    ```c
    int request_irq(unsigned int irq,
                   irqreturn_t (*handler)(int, void *, struct pt_regs *),
                   unsigned long irqflags, const char * devname, void *dev_id)
    {
        ...
    }
    ```

    - irq: 할당할 interrupt의 번호
    - handler: interrupt handler를 가리키는 function pointer
    - irqflags
      - SA_SHIRQ: interrupt가 공유됨
      - SA_INTERRUPT: 실행하는 동안 local interrupt를 비활성화 시킴

##### /proc/interrupts

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027220121661.png" alt="image-20191027220121661" style="zoom:80%;" />

각각 interrupt line 번호, 받은 interrupt의 수???, interrupt controller 이름, interrupt handler의 디바이스명(request_irq()의 devname 파라미터. 2개 디바이스에 의해 공유되는 경우 이름이 2개 찍힌다)

#### Handling Interrupts (Interrupt Handler 실행)



![image-20191027203121004](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027203121004.png)

##### Step 1. Saving the IRQ number and Registers

프로세스가 IDT를 읽어 interrupt handler entry point를 결정한다. IRQ 번호와 register 정보는 kernel stack에 저장되어 있다. (arch/i386/kernel/entry.S)

##### Step 2. Executing Interrupt Service Routine

PIC에 ACK를 보내어 더 많은 인터럽트가 발생하는 것을 허용하고, IRQ를 공유하는 모든 디바이스와 연결된 ISR(Interrupt Service Routine)을 실행한다.

- arch/i386/kernel/irq.c::do_IRQ(struct pt_regs regs)

  - Interrupt와 연관된 모든 ISR을 실행하도록 호출
    - eax 레지스터는 SAVE_ALL에 의해 push된 마지막 레지스터 값을 포함하는 스택 위치를 가리킨다.
  - PIC에 있는 interrupt를 인지하고, IRQ line을 비활성화
    - mask_and_ack_8249A()
  - handle_IRQ_event()를 호출함으로써 ISR을 실행한다
    - 여기서 호출된 handler들은 request_irq()에 의해 등록된 interrupt handler들이다.
  - IRQ line을 다시 활성화한다(end_8259A_IRQ())
  - 최종적으로, nested interrupt context에 속해있지 않다면 do_softirq()를 실행한다.

- handle_IRQ_event()

  - 한 종류의 디바이스 전용 ISR?을 실행시킨다
  - IRQ가 여러 디바이스에 의해 공유된다면 각 ISR이 모두 호출되어야 함

  ```c
  do {action->handler(irq, action->dev_id, regs);
  	action = action->next;
  } while (action);
  ```

  ![image-20191027231016587](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191027231016587.png)

- Nested Executions

  - 커널 제어 경로는 중첩(nested)될 수 있음!
    - interrupt handler가 다른 interrupt handler에 의해 인터럽트 될 수 있기 때문에 커널 제어 경로 또한 중첩될 수 있다.(3단 이상도 가능)
    - Exception도 interrupt에 의해 인터럽트 될 수 있다.

  ![image-20191028015906607](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028015906607.png)

  - Nested Interrupt가 필요한 이유
    - PIC와 디바이스 컨트롤러의 처리량을 향상시키기 위해
    - priority level이 없는 인터럽트 모델을 구현하기 위해
      - 하드웨어 디바이스 간의 사전 우선순위를 정의할 필요가 없음
  - Nested interrupt가 일어나기 위한 조건
    - **현재 프로세스는 인터럽트와 연관된 커널 제어 경로가 실행되는 동안 전환되지 않는다!**
    - interrupt handler의 커널 제어 경로는 block되지 않아야 한다???(Ex. 디스크 I/O 작업 시작 등). 즉, **interrupt handler가 실행될 때까지는 어떠한 프로세스 전환도 발생할 수 없다.**
    - 중첩된 커널 제어 경로를 복구하기 위한 데이터는 현재 프로세스와 묶여 있는 커널 모드 stack에 저장되어 있다.

##### Step 3. Returning from Interrupts

ret_from_intr 주소로 jump하여 interrupt를 종료한다.

- Entry points(진입점) (arch/i386/kernel/entry.S)
  - ret_from_intr, ret_from_exception
- 특정 프로그램의 실행을 재개하기 위한 고려 사항
  - 중첩된 커널 제어 경로의 수 확인
    - 만약 중첩이 아닌 경우(1개인 경우) user mode로 돌아가야 함
  - 프로세스 전환 요청 대기 중
    - 프로세스 전환 요청이 있는 경우(current의 TIF_NEED_RESCHED가 1인 경우) 프로세스 스케줄링을 수행해야 함
    - 그 외의 경우 현재 프로세스로 돌아간다
  - 신호 대기 중
    - 신호를 현재 프로세스에 전달해야 한다면, 반드시 처리해야 함

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028034718185.png" alt="image-20191028034718185" style="zoom: 50%;" />

## Exceptions

(IA32)

- CPU-detected exceptions
  - <u>CPU가 비정상적인 상태를 감지할 때 생성됨</u>
  - Faults
    - 의도적으로 생성되는건 아니지만 복구 가능
    - 오류가 난 현재 명령어를 재실행 또는 중단시킴
    - Ex) Page fault(복구 가능), Protection fault(복구 불가)
  - Traps
    - 의도적으로 생성
    - 다음 명령어에 제어권을 넘김
    - Ex) Breakpoint traps(디버깅 목적)
  - Aborts
    - 의도적으로 생성되는 것도 아니며, 복구도 불가
    - 현재 프로그램을 중단시킴
    - Ex) 심각한 hardware failure(parity error, ...)
- Programmed exceptions (의도적)
  - <u>프로그래머의 요청에 의해 발생</u>
    - int, int3, into, bound 등의 명령어에 의해 발동
  - system call(int 0x80 또는 sysenter)을 구현할 때나, 디버거에 특정 이벤트를 알릴 때 사용

### Exception Handling

- 대부분의 exception은 이가 발생한 프로세스에게 UNIX 신호를 보내는 것으로 처리할 수 있다.

- 따라서 이 이후의 행동은 프로세스가 그 신호를 받기 전까지 전부 지연될 것이므로

  - 즉, 커널은 exception을 빠르게 처리할 수 있다. (모든 action이 다 멈추니까 프로세스도 전환되지 않고 신호를 받게 되어서 빠르게 처리할 수 있다는 뜻인듯)

- 한편, interrupt handling은

  - 일반적으로 관련된 프로세스가 중단되고 전혀 다른 프로세스가 실행중일 때 도착하게 되므로 UNIX 신호가 현재 프로세스에 전달되기 힘듦

- Linux에서의 서로 다른 20가지의 exceptions

  - 커널은 각 exception type에 맞는 전용 exception handler를 제공해야 한다.
  - exception handler는 exception을 발생시킨 프로세스에 해당 UNIX signal을 보낸다.

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028042236788.png" alt="image-20191028042236788" style="zoom:80%;" />

- 리눅스에서의 exception 사용

  - 프로그래밍 에러 또는 비정상 하드웨어 상태 처리
    - Ex) 프로그램이 '0으로 나누기'를 수행할 때
      1. CPU가 "divide error" exception을 띄운다
      2. Exception handler가 SIGFPE 신호를 현재 프로세스에 보낸다
      3. 복구하거나 중단
  - 하드웨어 자원을 효율적으로 관리(Ex. Page fault 처리 등)

- 일반적인 Exception handling 순서

  (Interrupt handling 순서와 유사)

  1. 대부분의 레지스터의 내용을 kernel mode stack에 저장
  2. exception을 처리(C 함수에 의해)
  3. Handler에서 빠져나옴(ret_from_exception())

### Interrupt Handler

#### Characteristics

- Interrupt handler는 interrupt context에서 실행된다!
  - Interrupt handler를 대신하는 프로세스는 항상 TASK_RUNNING 상태에 있어야 한다.
  - 결과: Interrupt handler가 I/O 디스크 작업과 같은 blocking procedure를 수행할 수 없다.
- 인터럽트 루틴 처리에 있어서 긴급성이 서로 다름
  - 인터럽트가 발생했을 때 수행해야 할 모든 작업이 같은 긴급성을 가지고 있는 것은 아님!
    - 일부 중요한 작업은 최대한 빨리 수행되어야 하고,
    - interrupt handler가 실행되는 동안 해당 IRQ line의 신호는 일시적으로 무시되므로 길고 중요하지 않은 작업은 지연되어야 한다.
  - 다른 클래스의 interrupt들

#### Interrupt Classes

- Critical(중요)
  - maskable interrupt의 비활성화와 함께 즉각적인 작업이 요구됨
    - Ex) PIC에 대한 interrupt를 인식, PIC/디바이스 컨트롤러를 재프로그래밍, 디바이스와 CPU 모두에서 접근하는 자료구조가 업데이트되는 경우
- Noncritical(중요 X)
  - 다른 interrupt가 허용되지만 즉각적인 작업이 요구됨
    - Ex) CPU에 의해서만 접근되어 업데이트되는 자료구조(키보드 키가 눌렸을 때 스캔 코드를 읽는 것)
- Noncritical deferrable(중요 X, 지연 가능)
  - 커널 작업의 영향을 미치지 않는 선에서 장시간 지연 가능
    - 특정 프로세스의 주소에 버퍼 내용을 복사하는 작업
      - 관심있는 프로세스는 데이터를 계속 기다린다!
  - 리눅스의 softirq, tasklets, work queues와 같은 분리된 함수들에 의해 수행됨

#### Conflicting Goals of Interrupt Handler

- Interrupt 처리는 빨라야 한다!
  - Interrupt handler는 현재 실행중인 코드를 비동기적으로 실행하며(커널 코드, 다른 인터럽트 핸들러, 유저 프로세스 등) interrupt 한다. 이렇게 interrupt된 코드가 정지되는 것을 막기 위해 인터럽트 핸들러는 빠르게 작업을 수행해야 한다.
  - Interrupt handler는 현재 프로세서의 인터럽트가 비활성화된 상태로 실행된다. 인터럽트를 최대한 빨리 완료하고 다시 활성화해야 한다.
- Interrupt handler는 가끔 상당한 양의 작업을 수행해야 한다!
  - 시간이 많이 걸림(NIC에서 네트워크 패킷을 처리하는 것 등)
  - Interrupt handler는 block할 수 없음! 이는 Interrupt handler에서 할 수 있는 것을 제한한다.???
- 해결책: Interrupt handler를 Top-half, Bottom-half로 나눈다

#### Top-half / Bottom-half

리눅스에 국한되지 않은 용어

##### Top-half(=hard IRQ handler)

- hardware interrupt가 수신된 즉시 실행
- 기본적인 하드웨어 관련 및 시간과 관련된 작업 수행
- 일부 또는 모든 interrupt가 비활성화되었을 때 실행
- 나중에 실행될 나머지 task들(bottom-half)을 스케줄링
- 즉시 종료
- 이 경우, kernel response time을 계속 작게 유지해야 한다!
- Hard IRQ Handler
  - Hardware interrupt context에서 실행
    - sleep/block 불가
    - Hardware interrupt 비활성화(활성화 여부 조정 가능)
    - 시간에 민감한, 하드웨어 제어에 관련된, 다른 interrupt에 의해 인터럽트되지 않는 작업을 수행

##### Bottom-half(=soft IRQ handler)

- 모든 interrupt가 활성화된 상태에서 나중에 실행되는 Interrupt handler의 subtask(덜 중요하고 시간이 오래 걸리는 작업)
- Interrupt에 대한 작업을 실제로 하는 역할
- softirq, tasklet
  - Software interrupt context에서 실행
    - Hardware interrupt 활성화
    - sleep/block 불가능
    - 다른 softirq, tasklet에 의해 선점되지 않음(hard IRQ handler에 의해서는 선점 가능)
- workqueue, kernel thread
  - Process context에서 실행
    - Hardware interrupt 활성화
    - sleep/block 가능

![image-20191028203659730](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191028203659730.png)

Interrupt handler는 **Hard IRQ handler(top-half)**와 **softirq, tasklet, workqueue, kernel thread(bottom-half)**로 이루어져 있음!

### softirq

정적으로 정의된 bottom-half로, 어떤 프로세스에서도 동시에 실행될 수 있다

- 특성
  - Interrupt context에서 돌아감(sleep/block 불가, blocking lock X, blocking KMA X, 유저공간 접근 불가)
  - 시간 및 성능이 중요한 처리에서 사용됨(Ex. 네트워크, 빠른 입출력)
  - Hardware interrupt처럼 raised와 checked가 가능
  - 같은 softirq는 여러 CPU에서 동시에 돌아갈 수 있다
    - 주의: 공유 데이터 접근에 lock을 걸어야 함

#### Initializing softirq

- softirq 핸들러는 **open_softirq()**를 통해 compile-time에 정적으로 할당된다.
- Linux 2.6.11에서는 6개의 사전정의된 softirq handler(action)을 사용함
  - General-purpose kernel softirq: tasklet_hi_action(), tasklet_action()
  - Device-specific softirq: run_timer_softirq(), net_tx_action(),  net_rx_action(), scsi_softirq()
- 현재 Linus 5.2.13(최신)에서는 10종류의 softirq 사용
  - HI_SOFTIRQ=0, TIMER_SOFTIRQ, NET_TX_SOFTIRQ, NET_RX_SOFTIRQ, BLOCK_SOFTIRQ, IRQ_POLL_SOFTIRQ, TASKLET_SOFTIRQ, SCHED_SOFTIRQ, HRTIMER_SOFTIRQ, RCU_SOFTIRQ

#### Raising softirq(실행)

- kernel/softirq.c:**raise_softirq_irqoff(nr)**
  - \_\_raise\_softirq\_irqoff(nr)을 호출하여 local_softirq_pending(cpu)의 적절한 비트를 set하게 함
  - nr 자리에는 위의 softirq 종류가 들어간다!

#### Checking softirq

- **local_softirq_pending(cpu)**: 대기중인 softirq를 확인함
  - 값이 0이 아닐 경우 do_softirq() 호출
- softirq 대기 여부 checkpoint
  - do_IRQ() 함수가 인터럽트 핸들링을 마쳤을 때(irq_exit())
  - ksoftirqd_CPUn 커널 스레드가 깨어났을 때
  - networking subsystem과 같이 대기중인 softirq를 명시적으로 확인하고 실행하는 코드

### tasklet

유연하고 동적으로 생성된 bottom-half(Ex. 인자가 존재하는 커널 함수)로, softirq의 위에 존재(HI_SOFTIRQ, TASKLET_SOFTIRQ)

- 특성
  - Interrupt context에서 돌아감(sleep/block 불가, blocking lock X, blocking KMA X, 유저공간 접근 불가)
  - 한번 스케줄되면 한번은 실행되도록 보장됨
  - 엄격하게 직렬화 되어있음!(nested execution X)
  - 같은 tasklet은 여러 프로세서에서 동시에 돌릴 수 없음(softirq보다 엄격한 동시성)
    - 약한 lock 요구사항을 가졌기 때문에 사용이 편함
  - sleep이 필요없는 디바이스 드라이버의 지연 가능 함수를 구현할 때 사용???
  - Tasklet은 사실상 동시에 못 돌리는 softirq라고 보면 됨

#### tasklet Data Structure

include/linux/interrupt.h

```c
struct tasklet_struct
{
    struct tasklet_struct *next; // 리스트에 있는 다음 tasklet
    unsigned long state; // tasklet의 상태
    atomic_t count; // 참조 카운터
    void (*func)(unsigned long); // tasklet handler function
    unsigned long data; // tasklet 함수의 인자
}
```

- CPU당 2개의 tasklet 리스트가 존재(우선순위 다름)

```c
/* /usr/src/linux/kernel/softirq.c */
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec) = {NULL};
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec) = {NULL};
```

![image-20191029004756143](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029004756143.png)

#### Declaring tasklet

- 1개의 인자를 가진 함수를 정의
  - **void function(unsigned long data)**
- **DECLARE_TASKLET(name, function, data)**
  - struct tasklet_struct 타입의 tasklet을 정의
- **DECLARE_TASKLET_DISABLED(name, function, data)**
  - tasklet을 초기 상태 "disabled"로 정의
    - 스케줄될 수는 있지만, 활성화되기 전까지는 실행되지 않음

#### Scheduling tasklet

- 후에 실행되도록 tasklet을 스케줄링
  - **void tasklet_schedule(struct tasklet_struct *t)**
  - **void tasklet_hi_schedule(struct tasklet_struct *t)**
  - 해당 tasklet 리스트의 맨 처음에 tasklet을 추가하고, softirq를 raise한다.
- tasklet 비활성화/활성화
  - **void tasklet_disable(struct tasklet_struct *t);**
  - **void tasklet_enable(struct tasklet_struct *t);**
  - 비활성화된 tasklet은 스케줄될 수 있지만, 실행될 수 없음
    - 다시 활성화될 때까지 tasklet 리스트에 남아있게 됨
- 예시

```c
static void kbd_bh(unsigned long dummy){
    ...
}

DECLARE_TASKLET_DISABLED(keyboard_tasklet, kbd_bh, 0);

int __init kbd_init(void){
    ...
    tasklet_enable(&keyboard_tasklet);
    tasklet_schedule(&keyboard_tasklet);
    return 0;
}
```

#### Running tasklet

![image-20191029011021471](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029011021471.png)

???

tasklet_struct *t가 있을 때 t->func를 실행시키는 방식(인자로는 t->data 사용)

### ksoftirqd Kernel Threads

(bottom-half 중 하나가 아님. softirq 보조)

- 연속적인 큰 부피의 softirq 흐름이 문제를 발생시킴
  - 네트워크 softirq와 tasklet softirq는 자기 자신을 재발동시킴
  - 네트워크 혼잡이 발생하면 softirq를 매우 자주 raise하게 되고, user mode 프로세스에게 엄청난 지연을 초래할 수 있음
- softirq에서의 중요한 균형 문제
  - softirq 반응성 vs user mode task 지연
- 이에 따른 대안 마련
  - Solution 1. do_softirq()가 실행중인 동안 발생하는 softirq를 무시!(재발동되는 softirq는 처리하지 않음)
    - 단점: softirq를 starve시킴
      - softirq 지연 시간이 발생하며, idle system의 장점을 살릴 수 없음
  - Solution 2. 대기중인 softirq를 연속적으로 다시 확인하며, 반환하기 전에 처리한다
    - 단점: 로드가 높은 환경에서는 do_softirq()가 반환되지 않으며 user mode 프로그램이 사실상 멈추게 된다.
  - **ksoftirqd** Solution
    - 재발동 softirq의 경우 즉시 처리하지 않는다!
    - 대신 softirq의 수가 넘치게 되면 커널이 ksoftirq라는 커널 스레드(CPU당)를 깨워 이를 처리하게 만든다.
      - 커널 스레드는 Process context에서 가장 낮은 priority(nice=+19)로 실행된다. 

```c
static int ksoftirqd(void * __bind_cpu)
{
	...
	for (;;){
		schedule(); //
		...
		while (local_softirq_pending()) {
			...
			do_softirq(); //
			...
		}
		...
	}
}
```

### Work queue

- 컨셉
  - 작업이 활성화되어 특별한 커널 스레드(worker_thread)에 의해 실행되는 것을 허용; 즉 커널 스레드에서 빌드됨
    - work queue의 요소는 수행해야 할 작업을 구현한 handler function이다.
  - work queue는 시스템의 CPU당 하나의 worker_thread를 가지고 있다.
- 특성
  - Process context에서 실행됨
    - 스케줄 가능, sleep 가능. (softirq, tasklet은 interrupt context에서 실행되므로 sleep 불가)
  - device driver writer의 bottom half가 sleep 되어야 할 때의 유일한 옵션
    - 많은 메모리 할당, 세마포어 획득, block I/O 실행, ...
  - 쉽게 사용할 수 있음(bottom-half 메커니즘 중 가장 단순)
  - context switch로 인해 overhead 발생

![image-20191029030346194](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029030346194.png)

![image-20191029030432563](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029030432563.png)

- worker_thread는 커널 스레드!

  - 7번째 줄에서 깨어났을 때, queued function을 실행(12번째 줄)
  - 더이상 할 작업이 남아있지 않으면 worker_thread는 sleep로 돌아감(5번째 줄)

- 'events' 라는 사전정의된 work queue

  - 일반적인 목적의 work queue는 커널 개발자에 의해 자유롭게 사용됨

    - struct workqueue_struct *keventd_wq; // 사전정의된 wq의 이름

    ![image-20191029030843286](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029030843286.png)

  - 대부분의 디바이스 드라이버들은 지연된 작업을 처리하기 위해 keventd_wq를 사용한다!

    - 이와 같은 특수 목적의 work queue는 비동기 I/O, blocked I/O(kblockd workqueue), XFS 파일시스템 등을 처리함

### 전체 흐름

![image-20191029023035428](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029023035428.png)

???

### 어느 Bottom-half를 사용할까?

| Bottom half |  Context  |     내부 직렬화     |
| :---------: | :-------: | :-----------------: |
|   Softirq   | Interrupt |        none         |
|   Tasklet   | Interrupt | 같은 tasklet에 대해 |
| Work queue  |  Process  |        none         |

- 여기 모두 일반적으로 인터럽트가 활성화되었을 때만 실행된다.
  - 만약 공유 데이터와 인터럽트 핸들러(top-half)가 있는 경우 interrupt를 비활성화 하거나 lock을 사용해야 함
- 가이드라인
  - softirq는 성능이 중요한 경우에 사용!
    - 자주 실행될 때, 스레드가 많을 때, time-critical할 때
  - tasklet은 동적으로 생성되고, locking requirement가 약하기 때문에 softirq보다 사용하기 더 단순함
    - 일반적인 하드웨어 디바이스의 bottom-half 처리에 있어 충분함!

### Interrupt Control

- 커널이 함수에게 제공하는 기능
  - 현재 프로세서의 interrupt를 비활성화
  - 기계 전체(모든 CPU)의 interrupt line을 mask out함???
- <u>동기화를 위해 interrupt 비활성화가 필요하다!</u>
  - **interrupt를 비활성화하면 현재 코드를 선점할 interrupt handler가 없다는 것을 보장받을 수 있음**
  - interrupt 비활성화는 커널의 선점 또한 막을 수 있다. (스케줄러의 time interrupt도 비활성화됨)
  - 하지만 다른 CPU로부터 동시 접근하는 것은 가능!
    - 이러한 다른 CPU로부터의 동시 접근은 lock으로 막을 수 있음

#### Local Interrupt Control

- **local_irq_disable() / local_irq_enable()**

  - 현재 프로세서에 local interrupt가 전달되는 것을 비활성화/활성화

- 문제

  - local_irq_disable() 이후 무조건 인터럽트를 활성화시키는 것은 위험하다!

    - 비활성화된 다른 인터럽트들이 활성화되어버린다...

  - 이 문제를 대체하기 위해 save, restore 함수를 사용!

    - local_irq_save(): 현재 전달되고 있는 local interrupt의 상태를 저장하고 disable함
    - local_irq_restore(): 전달되는 local interrupt의 상태를 복구(enable)

  - 예시) 만약 foo() 함수가 interrupt 비활성화 상태로 자신의 코드 일부를 실행하려 할 때

    ![image-20191029035151026](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029035151026.png)

    - disable - disable - enable - enable 순서로 실행되는데, foo1() 안의 local_irq_enable이 코드 전체의 interrupt를 활성화시켜버리므로 문제!

    ![image-20191029035446236](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191029035446236.png)

    - foo1() 에서 disable / enable 함수 대신 save / restore 함수를 사용하면 문제를 해결할 수 있음!

#### Global Interrupt Control

- 모든 CPU에 대해 특정 interrupt line을 비활성화/활성화 하는 경우
  - void disable_irq(unsigned int irq)
    - interrupt 번호 "irq"인 인터럽트를 비활성화
    - 동시 실행중인 interrupt handler가 완료될 때까지 return되지 않음
  - void disable_irq_nosync(insigned int irq)
    - interrupt 번호 "irq"인 인터럽트를 비활성화
    - 위 함수와 다르게 interrupt handler의 완료를 기다리지 않음
  - void enable_irq(unsigned int irq)
    - interrupt 번호 "irq"인 인터럽트를 활성화
  - void synchronize_irq(insigned int irq)
    - "irq"의 interrupt handler가 완료될 때까지 return되지 않음
    - sync 맞추기용으로 쓸듯?
- 위에서 문제가 되었던 nested 활성화/비활성화가 의도대로 작동하는듯?

