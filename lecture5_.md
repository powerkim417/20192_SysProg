# Lecture 5. System Calls

### Principles

- 애플리케이션과 하드웨어 사이에 별도의 layer를 끼워넣음
- 이점
  - 프로그래밍하기 쉬움
    - 유저가 하드웨어 장치의 low-level한 특징을 몰라도 됨 
  - 시스템 보안 향상
    - 커널은 인터페이스 레벨에서의 요청이 올바른지 확인할 수 있음
  - 프로그램의 이식성 향상
- System calls
  - UNIX 시스템은 커널에 발생한 <u>system call</u>을 통해 유저 모드 프로세스와 하드웨어 장치 사이의 대부분의 인터페이스를 구현
    - 유저모드 프로세스와 하드웨어 장치 간의 인터페이스
    - 커널 서비스를 요청

### POSIX APIs and System Calls

- API (Application Programming Interface)
  - 주어진 서비스를 얻는 방법을 지정하는 함수 정의
    - Ex) POSIX API인 malloc(), calloc(), free()는 libc 안에서 brk() system call로 구현되어 있다.
    - "POSIX-호환": 시스템이 애플리케이션에 적절한 API 집합을 제공하는 경우(기능이 어떻게 구현되었는지는 상관 없음)
  - 프로그래머의 관점: '유저 모드 라이브러리'
- System call
  - 소프트웨어 인터럽트를 통해 커널에 전달되는 명시적인 request
    - 커널 설계자의 관점: '커널 안에 속한다'
  - 몇몇 system call은 1개 이상의 인자를 필요로 한다.
  - 정상적으로 종료되었을 경우 0 또는 양수를 반환
    - 실패할 경우 -1을 반환하고, 실패 오류와 관련된 오류 코드를 errno 변수에 저장
      - -1를 반환하지 않았을 경우(정상적 종료) errno 값은 의미가 없음!
  - 유저 공간의 libc 안에 wrapper 함수로써 구현되어 있음
    - Ex) libc 안의 _syscall3()

### System Call Handling

#### System Call Handling via "int 0x80"

### Wrapper Routines

### Parameter Passing

### Parameter Checking

### Accessing User-Space Memory Address

### Adding a New System Call