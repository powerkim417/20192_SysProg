# Lecture 9. Memory Management

## Linux Memory Management

- "Demand Paged Virtual Memory" 모델
  - 물리적 페이지 매핑/할당/관리
  - 2차 메모리 관리: swapping
- 아키텍쳐-독립 모델
  - 다른 아키텍쳐의 서로 다른 메모리 매핑을 지원하는 인터페이스
  - 이 경우 아키텍쳐 매핑이 필요
    - 메모리 모델은 물리 메모리로 매핑되어야 함
    - 아키텍쳐 따라 다름(arch/xxx/mm)

### Virtual Address Space

- 선형 주소공간은 <u>유저 주소공간</u>과 <u>커널 주소공간</u>으로 나뉨
  - 유저 주소공간
    - 0x00000000 ~ PAGE_OFFSET-1(일반적으로 0xC0000000-1, 3GB)
    - 유저모드, 커널모드 프로세스 모두에게 사용됨
  - 커널 주소공간
    - PAGE_OFFSET ~ 0xFFFFFFFF(4GB)
    - 커널모드 프로세스에게만 사용됨
    - 커널 주소공간은 모든 프로세스에게 같은 매핑(공유)
      - 이렇게 커널 매핑을 공유하므로 프로세스를 생성해서 새로운 페이지 디렉토리가 생성될 때 커널의 PGD부분만 복사하면 됨

![image-20191217110452610](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217110452610.png)

- 주소공간을 유저-커널로 나누는 이유
  - 4GB를 통째로 사용하면 모드 스위칭 때 비용 많이 발생 + TLB flush 발생
  - 커널모드로 전환되면 커널은 안정적인 환경에 포함되어야 함(즉, 주소공간 분리되어야 함)
  - 물리 페이지와 커널의 시작은 직접 매핑되어있으므로 테이블 조회 안해도 됨
- split ratio
  - 3:1이 가장 보편적이지만 유일한 선택은 아님
  - 커널에서 돌아가는 코드가 메인이거나, 커널에 많은 메모리가 필요한 시스템의 경우2:2로 나누기도 함
    - Ex) 네트워크 라우터
    - 이점: 많은 물리 메모리가 커널 주소공간에 직접 매핑됨

### Physical Memory Layout for 'Kernel Core'

- 커널은 "고정"되어 있음(swap 불가, 영구 매핑)
  - 커널의 코드와 자료구조는 보호되고 있는 페이지 프레임들에 저장되어 있음
    - **이러한 프레임들은 동적으로 할당되거나 스왑될 수 없다**
    - 리눅스 커널은 RAM의 물리주소 0x00100000(2번째 MB)에서부터 설치됨
      - 그럼 1번째 MB는? BIOS에서 사용함
      - 연속적인 페이지 프레임을 위해 리눅스는 1번째 MB를 넘기고 실행함
- 커널 고정 이후 남는 물리 메모리는 Kernel memory allocator(버디 시스템)이 필요한 경우 할당해준다.

![image-20191217112347639](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217112347639.png)

### Physical Memory Mapping in Linux/x86

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217113628675.png" alt="image-20191217113628675" style="zoom: 80%;" />

- 물리 메모리 매핑
  - 물리 메모리는 커널 주소공간에 직접 매핑되어 있어 최대 1GB(PAGE_OFFSET이 3GB일 때)
  - 그러므로 직접 매핑되는 물리 메모리는 1GB보다 작다(896MB)
  - Implications
    - 커널 공간에서의 메모리 할당은 물리 RAM에서 진행됨
    - 물리 주소 = 가상 주소 - PAGE_OFFSET
- lowmem
  - 커널공간에 존재하는 논리주소를 위한 메모리
  - 커널주소공간에 직접 매핑할 수 있는 물리적 프레임
- highmem(896MB 이후 공간)
  - 커널공간에 존재하지 않는 논리주소를 위한 메모리
  - 커널주소공간에 직접 매핑할 수 없는 물리적 프레임

![image-20191217113930800](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217113930800.png)

- 물리 메모리 -> 커널 공간으로 직접 매핑하는 이유

  - 물리페이지 할당자의 사용자가 직접 페이지에 접근하게 하기 위해
  - DMA 전환에 적합: 커널 고정 + 물리적 연속성
  - 커널 공간에 확장 페이지 허용 -> 넓은 TLB 범위와 높은 성능을 위해

- 최대 lowmem이 1GB에 미치지 못하는 이유: 약 128MB의 공간을 다른 목적으로 보존해야 함

  ![image-20191217131322257](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217131322257.png)

  - vmalloc 공간: 연속 커널 가상주소를 비연속적인 물리 주소로 할당
    - ![image-20191217131344505](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217131344505.png)
    - 커널은 불연속 물리주소를 최대한 피하려 하나, 시스템이 오래 작동하면 물리메모리가 필요할 때 연속적인 공간이 없을 수도 있음(파편화)
  - high memory mapping(영구 매핑): highmem 페이지와 커널을 연결하는데 사용
    - 커널 주소공간으로 영구적으로 매핑이 불가..그래서 커널 주소공간에 동적으로 매핑/언매핑.
      - 현재 프로세스를 block할 수 있으므로 인터럽트 컨텍스트에선 사용 X
  - fixmap(임시 매핑): 물리적 주소공간에서 고정되었지만 자유롭게 선택할 수 있는 페이지와 관련된 가상주소공간 항목

### Constructing Page Tables

프로세스 당 페이지 테이블 생성

![image-20191217134123524](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217134123524.png)

#### Constructing Kernel Page Tables

- 커널 페이지 테이블

  - 마스터 커널 PGD 기반
  - 시스템 초기화 후, 테이블은 프로세스나 커널스레드에 의해 직접적으로 사용되지 않음
  - 마스터 커널 PGD의 높은 엔트리(769~1024, 3~4GB)는 시스템의 일반 프로세스의 해당 PGD 엔트리의 참조 모델임

- 커널 페이지 테이블 초기화의 2단계

  - 커널 이미지가 메모리에 로드되었을 때(CPU는 아직 real 모드)
  - 1. 커널은 RAM에 커널을 설치할 수 있는 크기인 8MB의 제한된 주소공간을 생성하여 핵심 자구조를 초기화
    2. 모든 RAM에 대한 page table을 세팅

- 동적 메모리를 위해 커널 페이지 테이블 할당

  - 모든 물리 페이지 프레임을 PAGE_OFFSET 위의 커널 주소 공간에 직접 매핑

- paging_init(): 마스터 커널 PGD를 (재)초기화

  - pagetable_init() 호출하여 테이블 엔트리 세팅
  - swapper_pg_dir의 물리 주소를 cr3 레지스터에 저장
  - flush_tlb_all() 호출하여 TLB 엔트리 무효화

- 3가지 경우(중 2가지만 알아보자)

  1. RAM 크기가 896MB보다 작음
  2. RAM 크기가 896~4096 사이
     - 128MB 부분까지 넘침

  ![image-20191217141629941](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217141629941.png)

#### Constructing User Page Tables

- 각 유저 프로세스 페이지 테이블
  - PGD의 첫 항목은 3GB 아래의 선형 주소에 매핑
  - PGD의 마지막 256개 항목은 마스터 커널 PGD에 대응하는 항목이랑 동일해야 함
  - 페이지 테이블은 커널 데이터 세그먼트에 저장됨(동적 할당)

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217142848839.png" alt="image-20191217142848839" style="zoom:80%;" />

## Physical (Dynamic) Memory Management

- Physical RAM
  - 커널에 의해 보호되는 영역: 커널 코드와 정적 커널 자료구조 저장
  - 동적인 영역(위 영역을 제외한 남는 부분): 프로세스와 커널에 의해 사용됨

![image-20191217143035963](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217143035963.png)

### Physical Memory Structure

Node, Zone, Page Frame으로 이루어짐

![image-20191125133214013](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191125133214013.png)

#### Page Frame

- 물리 메모리는 페이지 프레임들로 나뉨(4KB)
- 페이지 프레임을 위한 자료구조: struct page
  - 커널은 각 물리 페이지 프레임의 상태/정보를 트래킹해야 함. 가상 페이지 아님!!
    - 페이지 프레임이 빈 상태인지 계속 알아야 하므로
  - 물리 메모리 자체를 표현하기 위함. 그 내용물은 아님

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217145804806.png" alt="image-20191217145804806" style="zoom:80%;" />

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217145810388.png" alt="image-20191217145810388" style="zoom:80%;" />

페이지 프레임의 상태를 표현하는 플래그

- 페이지 디스크립터(struct page)는 mem_map 배열에 저장됨
  - 리눅스의 전역변수 mem_map은 한 엔트리가 한 물리페이지를 포함하여 각 페이지프레임의 현 상태를 트래킹한다.
  - 존 디스크립터의 zone_mem_map 필드는 존의 첫 페이지 디스크립터의 주소로 초기화 됨

<img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217150738729.png" alt="image-20191217150738729" style="zoom: 80%;" />

- 커널 가상주소 변환

  - ZONE_DMA, ZONE_NORMAL의 메모리는 직접매핑, 모든 페이지 프레임은 mem_map에 표현되어 있음
  - 커널 가상주소를 물리주소, 즉 struct page로 변환
    - 물리주소를 mem_map의 인덱스로 사용

  ![image-20191217151058324](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217151058324.png)

#### Node

- 리눅스에서 NUMA(non uniform memory access) 모델 지원

  - 특정 CPU에서 접근 시간이 다른 메모리 장소들이 존재
    - 각 노드의 메모리 접근 차이를 감안해서 동작하도록 운영
    - 확장성 증가, 지연 작음
  - 머신의 메모리는 노드들로 나뉨
  - UMA: 각 노드는 같은 접근 시간을 가짐
  - 노드에 대한 자료구조: struct pglist_data

  <img src="C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217152424849.png" alt="image-20191217152424849" style="zoom:80%;" />

#### Zone

- 노드의 페이지 프레임들은 서로 다른 존들로 그룹됨
- 각 존은 다른 종류의 메모리 사용이 일어남
- 존에 대한 자료구조: struct zone
- 여러 존: ZONE_DMA, ZONE_DMA32, ZONE_NORMAL, ZONE_HIGHMEM
- 존으로 나누는 이유
  - 물리 메모리에 접근하는 하드웨어상의 제한
    - ISA 카드는 24비트 어드레싱을 지원하는데, 그러면 DMA 접근은 최대 16MB(2^24)까지만 가능
  - 물리 메모리가 너무 커서 커널의 가상 주소공간을 넘기도 하므로
- ZONE_DMA(0~16M)
  - Direct Memory Access
  - 기존 DMA 장치의 물리적 메모리 범위가 낮음
  - DMA 가능 페이지: **DMA 처리하는 페이지 프레임 포함**
  - DMA는 하드웨어(ZONE_DMA)에서 접근할 수 있고, 물리적으로 연속적인 메모리 공간이 필요
    - DMA 하드웨어 드라이버는 dma_alloc_coherent()를 사용하여 DMA 가능 공간을 할당
    - DMA가 PCI 버스를 통한 경우 pci_alloc_consistent()를 사용
    - USB일 때는 usb_buffer_alloc()

- ZONE_NORMAL(16M~896M)
  - **가상 주소공간의 상위 영역에 직접 매핑**
  - 일반적으로 어드레싱 가능한 페이지
- ZONE_HIGHMEM(896M<)
  - 동적으로 매핑된 페이지
    - high memory 포함(커널의 주소공간에 영구적으로 매핑되지 않는 페이지 프레임)
    - 896MB의 물리적 주소 위의 메모리는 커널이 접근할 때마다 임시적으로 커널 가상 메모리에 매핑됨
    - Highmem 처리
      - highmem이 할당되면 이는 직접적으로 주소지정 불가
      - 하려면 kmap(), kunmap()을 사용ㄹ

### Process Address Mapping: 요약

![image-20191217194120585](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217194120585.png)

## Kernel Memory Allocation(KMA) in Linux

- Contiguous Page Frame Allocator
  - 연속적인 빈 프레임(물리 메모리) 할당
  - 메커니즘: zone allocator + buddy system
  - 인터페이스: alloc_pages(), free_pages()
- Noncontiguous Page Frame Allocator
  - 임의의 비연속적인 프레임을 할당하고 선형 커널주소로 매핑
  - 메커니즘: alloc_page()를 반복하여 커널 페이지테이블 업데이트
  - 인터페이스: vmalloc(), vfree()
- Memory object allocator
  - 자주 요청/해제된 커널 자료구조를 할당
  - 메커니즘: 버디시스템 가장 위에 있는 slab allocator
  - 인터페이스: kmalloc(), kfree(), kmem_cache_alloc(), kmem_cache_free()

### Contiguous Page Frame Allocator

- 연속적인 페이지 프레임 할당
  - 물리적으로, 그리고 가상적으로도 연속적인 메모리를 반환
- 물리적으로 연속인 메모리를 잡는 것의 이점
  - 많은 하드웨어 장비가 가상주소를 지정할 수 없음(DMA는 페이징 순회를 무시). 즉 메모리 블록을 접근하게 해야 하므로 블록이 물리적으로 연속적인 덩어리가 되어야 함
  - 물리적으로 연속적인 블록 메모리는 하나의 큰 페이지 매핑을 사용할 수 있음. 이렇게 하면 하나의 TLB 엔트리만 필요하므로 TLB 오버헤드를 줄일 수 있음
- 단점
  - **물리적으로 연속적인 메모리를 찾는 것은 어려움, 특히 크게 할당해야 할때..**
  - 가상주소만 연속적인 것이 더 찾기 쉬우니 물리적으로 연속적인 메모리가 딱히 필요하지 않으면 vmalloc() 사용할것

![image-20191217202905011](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217202905011.png)

- Zone allocator: 커널 페이지 프레임 allocator의 앞부분
  - 메모리 요청에 적합한 메모리 존을 할당
  - 고려 사항
    - 보호해야 할 프레임의 풀을 보호
    - 페이지 프레임 요구 알고리즘을 시행
    - ZONE_DMA 최대한 보존
- Buddy System: 존 내에서 페이지 프레임 처리
- 캐시: 할당되기 전 페이지 프레임 포함
  - 성능을 위해 하나의 요청만 처리

#### Buddy System

- 리눅스 커널에서의 기본 메모리 할당자
  - 페이지를 정밀하게 할당
  - 외부 단편화 감소
  - 페이지 프레임을 최대한 연속적으로 할당
  - 2^n개의 페이지 프레임으로 할당
  - 각 존에서 실행됨
- ![image-20191217203533298](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217203533298.png)
- block 할당 과정
  - 원하는 크기의 블럭을 찾으면 즉시 할당
  - 원하는 크기보다 큰 블럭을 사용해야 하면 큰 블럭을 2개의 작은 블럭으로 쪼갠다. 이중 upper half를 free list에 넣고, lower half에서 메모리 할당을 함. 재귀!
- block 해제 과정
  - 블럭의 buddy가 free면 2개를 붙여서 큰 블럭 하나로 만듦. 재귀!
- 모든 free page는 각각 1,2,4,...,1024개의 연속 페이지 프레임을 의미하는 11개 리스트의 블럭들로 그룹됨.

![image-20191217204016615](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217204016615.png)

### Noncontiguous Page Frame Allocator

- 불연속 페이지 프레임 할당
  - 임의의 프레임 할당 후 커널 선형 주소로 매핑
  - 장점: 외부 단편화 X
  - 단점:  커널 페이지 테이블을 계속 봐야 하므로 오버헤드
  - **큰 영역의 메모리가 필요할 때와 같이 꼭 필요한 상황에서만 사용!**
- 리눅스는 이 방법 사용
  - 모듈을 위한 공간 할당
  - IO장치를 위한 버퍼 할당
  - 활성화 스왑 공간용 자료구조 할당
  - high memory 페이지 프레임 사용할 때
- vmalloc() 함수
  - 물리적으로는 연속 아니지만 가상주소는 항상 선형.
  - 페이지 레벨에서 새 공간의 초기 선형 주소 반환
- vfree() 함수
  - 인자로 받은 주소에 대한 영역을 해제

### Memory Object Allocator

커널 메모리 객체 관리

반복적/자주 할당 및 해제되는 경우 사용

작은 메모리공간의 여러번 요청에 효율적

#### Slab Allocator

##### 구조

- 객체를 cache 단위로 그룹화함
  - 각 타입마다 이전에 할당/해제된 객체를 위한 메모리 캐시 관리
- 캐시는 slab으로 나뉨
  - 각 slab은 할당되거나 해제된 객체를 모두 포함. 1개 이상의 연속 페이지 프레임으로 이루어짐
  - 각 slab은 3가지 상태중 하나. full, partial, empty
    - full은 빈 object가 없고, empty는 할당된 object가 없음

![image-20191217210444059](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217210444059.png)

##### Specific Caches

- mm_struct와 같은 커널 자료구조용으로만 사용가능한 캐시
- 1개의 캐시는 1개의 객체 타입

##### General Caches

- 2^n개의 범용 캐시
- kmalloc()으로 할당하고 kfree()로 해제

##### /proc/slabinfo

##### Allocating and Releasing

- Cache에 Slab 할당
  - 새로 생성된 캐시는 slab이 없음 -> 즉 빈 객체가 없음
  - 두 조건을 만족하면 새로운 slab을 할당한다.
    - 새로운 객체 요청 발생
    - 캐시 안에 빈 객체가 없음
  - cache_grow() 호출해서 새로운 slab 할당
- Cache에서 Slab 해제 
  - slab 할당자는 빈 slab의 페이지 프레임을 해제하지 않음
  - 두 조건을 만족하면 slab을 해제
    - 버디시스템이 새로운 페이지 프레임 요청을 처리할 수 없을 때
      - 버디시스템이 요청을 처리하려고 slab의 페이지 프레임 회수
    - slab이 비어있을 때(slab 안의 객체를 사용하고 있지 않을 때)
  - slab_destroy() 호출해서 slab 해제

##### Summary

- <u>페이지 레벨</u> 할당
  - alloc_page(): 존 형태의 페이지 프레임 할당자 기본 형태
  - vmalloc(): 임의의(불연속적) 물리적 페이지 할당, 가상 주소는 연속적.
- <u>메모리 객체 레벨</u> 할당
  - kmalloc(): 버디시스템을 통해 물리적으로 연결된 가상 페이지 덩어리들을 찾고, 더 작은 단위로 나눔 
  - kmem_cache_alloc(): 커널 데이터 객체의 특정 타입 할당

![image-20191217211555414](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217211555414.png)

### Which Method?

- **연속적인 물리 페이지가 필요할 때**
  - **kmalloc()**
  - **low-level 페이지 할당자 alloc_pages()**
- **high memory로부터 할당**
  - **alloc_pages()**
    - struct page에 대한 포인터 반환(페이지 매핑 안될 수도 있음)
  - **kmap()**을 사용하여 페이지 매핑
- 가상 주소만 연속이어도 되는 경우
  - **vmalloc()** 사용(kmalloc()보다는 조금 느림)

