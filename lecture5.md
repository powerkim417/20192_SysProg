# Lecture 5. System Calls

### Principles

- 애플리케이션과 하드웨어 사이에 <u>별도의 layer</u>를 끼워넣음
- 이점
  - 프로그래밍하기 쉬움
    - 유저가 하드웨어 장치의 low-level한 특징을 몰라도 됨 
  - 시스템 보안 향상
    - 커널은 인터페이스 레벨에서의 요청이 올바른지 확인할 수 있음
  - 프로그램의 이식성 향상
- System calls
  - UNIX 시스템은 커널에 발생한 <u>system call</u>을 통해 유저 모드 프로세스와 하드웨어 장치 사이의 대부분의 인터페이스를 구현
    - <u>유저모드 프로세스와 하드웨어 장치 간의 인터페이스</u>
    - 커널 서비스를 요청

### POSIX APIs and System Calls

- API (Application Programming Interface)
  - 주어진 서비스를 얻는 방법을 지정하는 함수 정의
  
  - 응용프로그램이 사용하는 프로그래밍 인터페이스 제공 (대표적으로 POSIX)
    
    > (응용프로그램) printf() 호출 → (C라이브러리) printf() → write() → (시스템 콜) write()
    
    - Ex) POSIX API인 malloc(), calloc(), free()는 libc 안에서 brk() system call로 구현되어 있다.
    - "POSIX-호환": 시스템이 애플리케이션에 적절한 API 집합을 제공하는 경우(기능이 어떻게 구현되었는지는 상관 없음)
    
  - 프로그래머의 관점: '유저 모드 라이브러리'(시스템콜 중요 X, API만 중요)
- System call
  - 소프트웨어 인터럽트를 통해 커널에 전달되는 명시적인 request
    - 커널 설계자의 관점: '커널 안에 속한다' (시스템콜만 중요)
  - 몇몇 system call은 1개 이상의 인자를 필요로 한다.
  - 정상적으로 종료되었을 경우 0 또는 양수를 반환
    - 실패할 경우 -1을 반환하고, 실패 오류와 관련된 오류 코드를 errno 변수에 저장
      - -1를 반환하지 않았을 경우(정상적 종료) errno 값은 의미가 없음!
  - 유저 공간의 libc 안에 wrapper 함수로써 구현되어 있음
    - Ex) libc 안의 _syscall3()

### System Call Handling

#### 시스템 콜 핸들링 개요

1. 커널 모드 스택에 있는 대부분의 레지스터 내용 저장
2. 시스템 콜 서비스 루틴을 실행시켜 시스템 콜을 처리한다
3. 핸들러로부터 나온다. 레지스터 정보를 복구하고 유저 모드로 돌아옴

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191215221322731.png" alt="image-20191215221322731" style="zoom: 80%;" />

#### 시스템 콜 호출/종료를 하는 2가지 방법

- "int 0x80", "iret"
  - 리눅스 커널 예전 버전 기준으로, <u>유저모드 → 커널모드만 가능</u>
- "sysenter", "sysexit"
  - 인텔 펜티엄II 마이크로프로세서에서 최초 소개, 리눅스 2.6 버전 이상에서 지원
- 커널은 이 2가지 라이브러리를 모두 지원할 수 있어야 하고, sysenter 기반 표준 라이브러리는 int 0x80만을 지원하는 커널에서도 사용할 수 있어야 한다. 즉, <u>커널과 표준 라이브러리는 최신/예전 프로세서 모두에서 작동할 수 있어야 한다!</u>

#### Wrapper Routines

- 매크로를 사용하여 시스템 콜을 정의하는 이유
  - **라이브러리 함수를 사용할 수 없는 커널 스레드도 시스템 콜을 호출할 수 있으므로!**
  - 해당 wrapper routine의 정의를 단순하게 하기 위해서!
  - 리눅스는 \_syscall0~\_syscall6의 7개 매크로를 정의(/include/asm-i386/unistd.h)
    - 0~6은 매크로화할 <u>시스템 콜에 필요한 파라미터의 수</u>(시스템 콜 번호 제외)를 의미
    - _syscall**n**() <u>매크로는 2+2\***n**개의 파라미터</u>가 필요
    - 6개 초과의 파라미터를 필요로 하거나, 표준 자료형이 아닌 반환값을 가지는 시스템 콜은 정의할 수 없음
    - ![image-20191215223003280](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191215223003280.png)
- 예시: write()

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191215233134021.png" alt="image-20191215233134021" style="zoom:80%;" />

#### 1. System Call Handling via <u>"int 0x80"</u>

- 유저모드 프로세스가 시스템 콜을 호출하면...

  - CPU는 커널 모드로 전환하고 커널 함수를 실행한다.
  - 리눅스에서는 어셈블리 명령어인 "int &0x80"을 통해서 시스템 콜을 호출해야 한다.
    - int &0x80은 벡터값 128로, 예외로 인식하도록 프로그래밍되어있음
  - 시스템 콜 번호
    - 각 시스템 콜을 식별하기 위해 사용하는 번호로, <u>eax</u> 레지스터에 저장

- 시스템 콜 핸들러에서의 작업

  1. ENTRY(system_call): 커널 모드 스택의 대부분 레지스터의 내용을 저장
     - SAVE_ALL 매크로
  2. 시스템 콜 서비스 루틴이라는 해당 C 함수를 호출(dispatch table 참조)하여 시스템 콜을 처리
     - sys_SystemCallName()
  3. syscall_exit 프로시저를 통해 핸들러 종료
     - RESTORE_ALL 매크로를 통해 레지스터 복구

  ![image-20191216132315295](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216132315295.png)

- 시스템 콜 초기화

  - trap_init() 함수는 커널 초기화시 호출되어 벡터 128에 해당하는 IDT 엔트리를 set
    - set_system_gate(0x80, &system_call);

- arch/i386/kernel/entry.S:ENTRY(system_call)

  - 모드 전환 이후의 커널 엔트리 포인트
  - 시스템 콜 번호(eax)와 스택에 있는 exception handler에서 사용하는 CPU 레지스터들을 저장
    - eflags, cs, eip, ss, esp와 같이 하드웨어 제어유닛에서 자동으로 저장하는 레지스터 제외
  - ebp 레지스터에 현재 프로세스의 thread_info structure의 주소 저장
  - 시스템 콜 번호 유효성 검사
    - 만약 유효하다면 eax에 저장된 그 시스템 콜 번호에 해당하는 서비스 루틴 호출

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216134445437.png" alt="image-20191216134445437" style="zoom:80%;" />

##### System Call Dispatch Table

- 역할: 시스템 콜의 서비스 루틴 주소를 저장
  - sys_call_table 배열의 NR_syscalls 엔트리에 저장
  - 약 280개의 시스템 콜이 구현되어있음

##### 시스템 콜 서비스 루틴의 예시

```c
asmlinkage long sys_getpid(void)
{
	return current->tgid;
}

asmlinkage long sys_pause(void)
{
	current->state = TASK_INTERRUPTIBLE;
	schedule();
	return -ERESTARTNOHAND;
}
```

![image-20191216142037634](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216142037634.png)

1. 유저 프로세스에서 라이브러리 호출
2. 라이브러리에서 0x80 실행, IDT 엔트리에서 0x80 접근
3. IDT 엔트리에서 ENTRY(system_call) 호출
4. ENTRY에서 eax를 확인하고 syscall dispatch table의 해당 시스템콜 엔트리를 찾아서
5. 실행

#### 2. System Call Handling via "sysenter"

- 빠른 시스템 콜이 특징
  - int 명령어는 일관성 및 보안 검사가 필요하므로 본질적으로 느림
  - sysenter는 유저모드에서 커널모드로 전환되는데 훨씬 빠름!
  - x86 마이크로프로세서의 일부 모델에서만 가능
  - libc 라이브러리의 wrapper 함수는 CPU와 리눅스 커널이 이를 지원할 때만 "sysenter "명령어를 사용할 수 있다.
- 시스템 콜로 들어갈 때
  - \_\_kernel_vsyscall()을 통해 레지스터 저장
  - sysenter_entry()는 system_call()과 같은 역할을 한다.
- 시스템 콜로부터 나올 때
  - "sysexit "명령어를 사용하는데, 이 또한 커널 모드에서 유저 모드로 빠르게 전환된다.

#### Parameter Passing

시스템 콜 핸들링 시 넘겨야 할 정보: 시스템 콜 번호(eax), 파라미터(<u>CPU 레지스터를 통해 넘기고</u> 커널 스택에 복사)

- 왜 레지스터를 통해 파라미터를 넘기는가?
  - 유저 스택에서 커널 스택으로 직접 파라미터를 복사하는 것은 복잡!
  - 시스템 콜 핸들러의 구조를 다른 exception 핸들러와 유사하게 만듦
- IA32에서의 parameter passing
  - 각 파라미터는 워드 크기(32비트)
  - 시스템 콜 번호: eax
  - 파라미터 수는 레지스터에 저장되어 넘어가므로 레지스터 수(ebx, ecx, edx, esi, edi, ebp)에 의해 제한되어 최대 6개만을 넘길 수 있다.
- 복잡한 데이터(32비트보다 큰, 또는 파라미터가 6개가 넘는)를 넘기는 방법
  - POSIX 표준에 의해, 큰 파라미터들은 그 주소를 통해서 넘긴다.
    - 6개 이상의 파라미터를 담고 있는 프로세스 주소공간을 가리키는 <u>하나의 레지스터</u>로 넘김!
  - 커널은 프로세스의 유저모드 메모리 공간으로부터/에게 데이터를 복사한다.
- Parameter checking: 시스템 콜 파라미터를 검증
  - 모든 시스템 콜 파라미터는 커널이 유저의 request를 처리하기 전에 확인해야 한다.
  - 시스템 콜과 특정 파라미터에 따라 다름
  - 일반적인 검증 방식: Address validity
    - 파라미터가 주소를 나타내는 경우: 커널은 프로세스 주소 공간 안을 확인
    - 아래의 두가지 방법으로 검증

##### Parameter Checking

- 방법 1: 선형 주소가 <u>프로세스 주소 공간에 속하는지</u> 확인
  - 맞다면 이를 포함하는 메모리 영역에 적절한 접근 권한이 있는 것!
  - 시간이 많이 소요됨
- 방법 2: 선형 주소가 <u>PAGE_OFFSET보다 낮은 위치에 있는지</u> 확인
  - 효율적이지만 꼼꼼한 방식은 아님(필요조건.. 올바른 주소는 PAGE_OFFSET 보다 아래에 있으나, PAGE_OFFSET 아래의 모든 주소가 다 올바른 주소는 아님)
  - 페이징 유닛이 선형 주소를 물리적 주소로 바꿀 때까지 실제 확인을 미룸(dynamic address checking)

### Accessing User-Space Memory Address

- 요구사항

  - 시스템 콜 서비스 루틴은 종종 프로세스의 주소공간에서 데이터를 읽어오거나 써야 할 때가 있음
  - 이를 위해 유저공간 메모리 접근 함수(User-space memory access function)을 사용!

- 유저공간 메모리 접근 함수

  - 어셈블리 코드로 매크로가 정의되어 있음(/include/asm-i386/uaccess.h)

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216201258781.png" alt="image-20191216201258781" style="zoom: 67%;" />

  - 예시) stime()을 위한 시스템 콜 서비스 루틴에서 get_user() 사용

    - int stime(time_t *t), 시스템 시각을 t에서 가리키는 값으로 설정(메모리 주소가 파라미터)

    ![image-20191216202115081](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191216202115081.png)

### Adding a New System Call

1. 유저공간 코드에 wrapper function을 추가
   - include <linux/unistd.h>
   - _syscall*n*() 사용!
2. "\_\_NR\_**newsyscall**"에 대한 매크로 정의를 포함
   - /include/asm-i386/unistd.h 에 새로운 시스템콜 번호를 정의
   - "sys_ni_syscall" 엔트리 중 하나를 사용할 수 있음
3. 커널 소스 트리에 해당하는 서비스 루틴을 작성
   - asmlinkage int sys_**newsyscall()**
4. 시스템콜 dispatch 테이블에 엔트리 추가
   - /arch/i386/kernel/entry.S::entry(sys_call_table)에 .long sys_**newsyscall** 추가