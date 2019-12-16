# Lecture 8. Memory Addressing

## Memory

### Memory Address(x86)

- 주소의 종류
  - 논리적 주소
    - 기계어 명령어에 사용되며, <u>명령어 또는 데이터의 주소를 참조</u>하는 데 사용되는 주소
    - segment 기반 주소 + 오프셋 으로 구성됨
  - 선형 주소(가상 주소)
    - IA32에서 단일 32비트 unsigned int
      - 4GB까지 주소를 지정하기 위해(0x00000000 ~ 0xFFFFFFFF)
  - 물리적 주소
    - 메모리 칩 안의 <u>메모리 셀에 접근</u>할 떄 사용되는 주소
- 논리적 주소 변환
  - 논리적 주소 → [**Segmentation** Unit] → 선형 주소 → [**Paging** Unit] → 물리적 주소

#### Execution Environments(x86)

![image-20191120151801012](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191120151801012.png)

#### Memory Management Units(x86)

![image-20191120154015592](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191120154015592.png)

- 메모리 관리는 **Segmentation**과 **Paging**으로 나뉨

## Segmentation

> ### *Logical address → Linear(Virtual) Address*

### Segmentation in Hardware(x86)

- Segmentation: 메모리를 논리적으로 분할 한 것(주소)
  - Logical address(48비트) := [segment selector(16비트), offset(32비트)]
    - Segment selector: 16비트 세그먼트 식별자
      - Segmentation register는 segment selector를 갖고 있음
    - Offset: 32비트, 세그먼트 내 상대 주소
- IA32는 주소를 빨리 받아오기 위해 segment register를 제공
  - cs
    - Code segment 레지스터
    - code segment를 가리킴
  - ds
    - Data segment 레지스터
    - static, global data를 가리킴
  - ss
    - Stack segment 레지스터
    - 현재의 Program stack을 가리킴
  - es, fs, gs
    - 일반적인 용도로 사용되며 임의의 세그먼트를 참조할 수 있음
  - 13비트의 인덱스 필드(최대 2^13 = 8K개의 세그먼트 인덱싱 가능)
    - Segment selector가 16비트일 때 15-3(13비트)가 index가 되고, 2는 TI(Table Indicator)이고, 1-0은 RPL(Requestor Privilege Level)이 된다.
- Segment Descriptor
  - 각 세그먼트는 그 세그먼트의 특성을 나타내는 8바이트의 segment descriptor로 표현된다.
- GDT / LDT
  - Segment descriptor는 Global Descriptor Table(GDT)와 Local Descriptor Table(LDT)에 저장됨
  - 일반적으로 GDT는 1개만 정의되며, 각 프로세스마다 GDT에 저장된 세그먼트 외에 추가적인 세그먼트를 생성해야 하는 경우 자체 LDT를 가질 수 있도록 허용됨
  - 단일프로세서 시스템에는 GDT가 오직 1개만 존재하며, 멀티프로세서 시스템에서는 각 CPU마다 1개의 GDT가 존재한다.
  - gdtr 레지스터(48비트)는 32비트(47-16)의 base address와 16비트(15-0)의 (GDT) table limit를 저장
- Logical address가 linear address로 바뀌는 전 과정

![image-20191120161446152](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191120161446152.png)

1. TI 필드의 값을 확인하여 gdtr를 참조할 지 ldtr를 참조할 지 결정한다.
2. selector에서 index부분(13)비트를 가져오고, left shift 3회 해서 16비트로 맞춘 값을 1에서 결정한 g/ldtr 레지스터의 값과 합한 값을 segment descriptor로 사용한다.(16비트 + 48비트)???
3. 논리 주소의 offset(32비트)과 segment descriptor의 base 필드를 더하여 최종적으로 linear address를 얻는다.

### Segmentation in Linux

- Linux는 segmentation을 매우 제한된 방법으로만 사용
  - 최소한의 접근
  - segmentation이 보편적이지 않고 복잡하기 때문
  
- Linux는 segmentation 대신 paging을 사용!
  - 간편한 메모리 관리
    - 모든 프로세서가 같은 세그먼트 레지스터 값을 가질 경우(모든 프로세스가 같은 선형 주소를 공유하는 경우) 메모리 관리가 더 쉬워진다!
  - 이식성
    - Linux는 다른 보편적 프로세서 구조에서도 돌아갈 수 있어야 한다.
    - 몇몇 RISC 프로세서는 segmentation을 제한적으로 지원함
  
- Linux에서 사용하는 segment(필요할 때만 사용)

  - 유저 모드(또는 커널 모드)에서 실행되는 모든 Linux 프로세스는 두 모드에 대해 명령어와 데이터에 주소 지정을 할 때 같은 pair의 세그먼트를 사용한다. <u>(모든 프로세스는 같은 논리 주소를 사용)</u>

    - Ex) \_\_KERNEL\_CS, \_\_KERNEL\_DS, \_\_USER\_CS, \_\_USER\_DS

  - 이러한 설계 때문에 세그먼트의 총 갯수는 제한되며, 모든 segment descriptor를 GDT에 저장하는 것이 가능하게 된다!

    - LDT는 커널에서 사용하지 않음

    ![image-20191121143337293](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191121143337293.png)

## Paging

> ### *Linear(Virtual) address → Physical Address*

### Paging in Hardware(x86)

- Paging unit은 선형 주소를 물리적 주소로 변환한다.
  - 접근 권한 확인(이 페이지에 접근할 권한이 있는가?)
  - 접근에 실패할 경우 Page fault exception 발생
- **Page**
  - 선형 주소들이 고정 길이의 간격으로 그룹화된 형태
  - 페이지 내의 연속적인 선형 주소는 연속적인 물리 주소로 매핑된다.
  - 페이지 프레임(물리 메모리)는 1페이지 크기
- <u>2-level paging</u> 지원(Page directory, Page table)
  - CR0 레지스터 내의 PG 비트를 set해서 활성화
  - **Page Table**: 물리 주소에 선형으로 매핑시켜주는 자료구조
    - <u>Page directory</u>: 페이지 테이블의 물리 주소(CR3 레지스터 내에 저장)
    - <u>Page table</u>: 페이지의 물리 주소
  - Page Size
    - Regular paging: 4KB
    - Extended paging: 4MB
    - 확장 페ㅇ

#### Regular Paging(x86)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128194830604.png" alt="image-20191128194830604" style="zoom: 67%;" />

- 페이지 크기: 4KB
- 선형 주소: 32비트
  - <u>디렉토리 10비트(디렉토리 내에서의 위치)</u>
  - <u>테이블 10비트(테이블 내에서의 위치)</u>
  - <u>오프셋 12비트: 4KB 페이지 내에서 위치</u>
  - 1024 \* 1024 \* 4096 = 총 4GB의 페이지 존재
- 최대 페이지 테이블: 1개의 페이지 디렉토리와 최대 1024개 테이블을 합해서 1025개가 된다.
- 최대 페이지 갯수: 2^10 * 2^10 = 2^20개

#### Page Directory and Page Table

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128202230636.png" alt="image-20191128202230636" style="zoom:67%;" />

- 테이블 구조
  - 1024개의 엔트리(각 엔트리는 4KB의 크기)
  - 하나의 페이지 프레임 안에 로드됨
- Page Directory Entry / Page Table Entry 구조
  - Present(P): 현재 메모리 안에 있는지 없는지
  - 페이지 프레임 물리 주소(Page Base Address): 엔트리에서 20 MSB 부분
    - 물리 주소의 12 LSB는 0이다???
  - Accessed(A): Paging unit이 페이지 프레임에 접근할 때마다 set됨
    - swap될 페이지를 선택하기 위해
  - Dirty(D): 페이지 프레임에서 write 작업이 실시되었을 때 set됨
    - Page Table Entry 에서만 사용됨
  - Read/Write(W): 페이지 권한(read/write 또는 read)
  - User/Supervisor(U): 접근에 필요한 권한 단계
  - PCD, PWT: 하드웨어 캐시를 위해 사용됨
  - Page size(PS): set될 경우 4MB 크기의 확장 페이지 프레임을 참조한다는 뜻
    - Page Directory Entry 에서만 사용됨
  - Global(G): TLB가 flush되는 것을 방지
    - Page Table Entry 에서만 사용됨

#### Extended Paging

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128222017483.png" alt="image-20191128222017483" style="zoom: 50%;" />

- 페이지 프레임의 크기를 4MB으로 사용(펜티엄 모델부터 가능)
  - 페이지 테이블에 대한 오버헤드 없이 크고 연속적인 물리 공간을 사용할 수 있다.
- 32비트 선형 주소
  - <u>디렉토리(10비트)</u>
  - <u>오프셋(22비트): 2^22 = 4MB</u>
  - PDE에서 Page size 플래그가 set되어 있어야 하고, CR4 프로세서 레지스터에서 PSE 플래그가 set되어 있어야 한다.

#### Paging Hardware(x86) 요약

![image-20191128222130193](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128222130193.png)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128222654708.png" alt="image-20191128222654708" style="zoom:80%;" />

- 4KB paging에는 Page Directory와 Page Table이 모두 사용되고, 4MB paging에는 Page Directory만 존재

### Paging in Linux

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128223000436.png" alt="image-20191128223000436" style="zoom: 67%;" />

- 4-level paging(2.6.11~)

  - 각 프로세스는 자신의 Page Global Directory와 그에 대한 페이지 테이블들을 가짐

  - Page Global Directory, Page Upper Directory, Page Middle Directory, Page Table

  - **2-level paging 하드웨어에서 4-level paging을 사용하는 경우**

    <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128225538291.png" alt="image-20191128225538291" style="zoom: 50%;" />

    1. Linear address에서 Page Upper Directory와 Page Middle Directory에 비트를 할당하지 않음으로써 제거
       - 포인터에서 두 디렉토리의 위치는 변하지 않기 때문에 32,64비트 상관없이 같은 코드 사용 가능
    2. 커널에서는 두 디렉토리의 엔트리 갯수를 1개로 설정한다. 이 2개의 엔트리는 적절한 Page Global Directory의 엔트리로 매핑된다.

#### Page Table Handling in Linux

- 페이지 플래그 read 함수

  ![image-20191128231552348](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231552348.png)

- 페이지 플래그 set 함수

  ![image-20191128231607942](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231607942.png)

- 페이지 테이블 엔트리에 작용하는 매크로

  ![image-20191128231637596](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231637596.png)

  ![image-20191128231739330](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231739330.png)

  - 5개의 type 변환 매크로

    ![image-20191128231748493](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231748493.png)

- 페이지 할당 함수

  ![image-20191128231800202](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231800202.png)

- 페이지 테이블 관련 자료구조: pgd_t, pud_t, pmd_t, pte_t
  - 페이지 테이블 pte_t
    - include/ams-i386/pgtable.h
    - 각각의 항목은 32비트 정수
    - 필드: 20비트 주소 + 플래그
  - 페이지 테이블 함수
    - 페이지 테이블 할당/해제
    - pte_*() 매크로: include/asm-i386/pgtable.h
  - 페이지 디렉토리 함수
    - pdg\_\*(), pmd\_\*() 매크로: include/asm-i386/pgtable.h

#### Kernel meets Hardware

![image-20191128231847906](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191128231847906.png)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191129174112412.png" alt="image-20191129174112412" style="zoom:67%;" />

#### Paging in x32_64(x64) and Linux

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191129205841219.png" alt="image-20191129205841219" style="zoom: 80%;" />![image-20191129205859922](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191129205859922.png)

![image-20191129205859922](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191129205859922.png)

- 4-level paging 구조
  - PML4 (Page Map Level 4)
  - PDPT (Page Directory Pointer Table)
  - PD (Page DIrectory)
  - PT (Page Table)
- 2개의 페이지 크기 존재
  - Regular paging: 4KB
  - Huge page: 2MB