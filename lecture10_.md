# Lecture 10. Process Address Space

### Linear Address Space: Overall View

![image-20191217212007709](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191217212007709.png)

선형 주소 공간에서의 각 역할

### Kernel Memory Allocation

- 관련 커널 함수
  - alloc_pages(), \_\_get\_free\_pages(): 버디시스템으로부터 연속적인 페이지 프레임 획득
  - vmalloc(): 페이지 레벨에서의 불연속적 메모리 공간 획득
  - kmem_cache_alloc(), kmalloc(): 바이트 레벨에서의 메모리 공간 획득
- 커널 메모리 할당(KMA)의 특성
  - 커널은 OS에서 가장 높은 우선순위의 구성요소이므로 이 작업은 지연되지 않음
  - 커널은 자신을 신뢰함(에러 X, 프로그래밍 에러에 대한 보호 필요 X)

### User Memory Allocation

- 유저모드 프로세스에 메모리 할당할 때
  - 프로세스는 동적 메모리를 요청하는데, 이는 우선순위가 낮음(malloc)
    - 일반적으로, 커널은 유저모드 프로세스에 페이지 프레임을 할당하는걸 지연함
  - 유저 프로그램은 믿을 수 없으므로 이에서 발생하는 주소 고나련 오류에 대해 대비할 수 있어야 함
- 유저 프로세스 메모리 할당의 특성
  - 유저모드 프로세스가 동적 메모리를 요청하면, 추가적인 페이지 프레임을 얻지 않음
  - 대신, 새로운 범위의 선형 주소(memory region)를 할당함. 이 메모리는 주소공간의 일부가 됨

### User Address Space(Abstract)

- 유저모드 프로세스 주소공간
  - 프로세스가 유저모드에서 사용가능한 선형 주소 모두 포함
  - 커널은 선형 주소의 간격을 수정함으로써 동적으로 유저 주소공간을 수정할 수 있음
- Memory Region
  - 선형 주소에서 시작점, 길이, 접근권한 등으로 규정된 선형 주소공간에서의 일부 구간
  - 효율성 문제로 인해 시작점과 길이는 4KB의 배수(페이지 프레임에 맞게)

#### When Process Gets New Memory Regions?

1. 새로 프로세스가 생성될 때: fork()
2. 다른 프로그램을 로드할 때: exec() 종류
3. 파일에서 메모리 매핑을 수행할 때: mmap()
   - 이걸 수행함으로써 프로세스 주소공간을 늘림
4. 프로세스의 유저모드 스택에 데이터 추가
5. 다른 협업 프로세스와 데이터 공유할 때
6. 프로세스의 힙을 늘릴 때

#### Related System Calls

![image-20191218133110142](C:\Users\KJH\AppData\Roaming\Typora\typora-user-images\image-20191218133110142.png)

#### Memory Descriptor(mm_struct)

task_struct -> mm_struct (메모리 디스크립터)

- 유저 주소공간과 관련된 정보

#### Memory Region

mm_struct -> vm_area_struct

- 연속적인 페이지 번호를 가지는 페이지들의 set을 포함
- 선형 주소 간격을 의미
  - vm_start~vm_end
  - vm_mm: 프로세스의 디스크립터를 가리킴
  - vm_next_share: 같이 physical memory를 공유할 다음 mmap

#### Adding and Removing Linear Address Interval

#### Handling Memory Regions

- Memory region의 수가 엄청 많을 때 리눅스는 RBtree를 씀
  - 수가 적을 때는 비효율적
- 하나를 찾을 때는 RBtree
- 순회할 때는 list

#### 할당 및 해제

(high-level)

- do_mmap()
  - 새 주소간격 생성, 초기화 - 주소공간 확장
- do_munmap()
  - 주소간격 해제 - 주소공간 축소

(low-level)

- find_vma(): 해당 주소에서 가장 가까운 영역 찾아줌
- find_vma_intersection(): 해당 주소 포함하는 영역 찾아줌
- get_unmapped_area(): 빈 주소간격 찾아줌
- insert_vm_struct(): 메모리 디스크립터 리스트에 영역 삽입

### User Address Space

#### Creating User Address Space

copy_mm()

- 새 주소공간 생성(새 프로세스의 페이지 테이블과 메모리 디스크립터 세팅)
- 경량 프로세스: CLONE_VM 플래그 set하고 clone()
  - 플래그가 set이면 스레드, not set이면 프로세스 fork
    - set이면 주소공간 공유, 아니면 새 주소공간 할당 후 부모프로세스의 M.R과 페이지테이블 복사

#### Deleting User Address Space

exit_mm()

### Managing the Heap

유저의 동적 메모리 요청을 채워주는 곳

start_brk ~ brk

- malloc, 

### Page Fault Exception Handling

주소 전환과정에서 발생, do_page_fault() 통해 처리

- do_page_fault()
  - 현재 프로세스의 M.R에서 페이지 폴트를 발생시킨 주소 비교
    - 물리 메모리에서 페이지 프레임 찾아서
    - 미싱 페이지를 로드하고
    - 페이지 테이블 업데이트

#### Linux Page Fault Handling

##### Overall Scheme

##### do_page_fault()

- good_area: 프로세스 주소공간에 주소가 속하는 경우
  - 메모리 리전이 쓰기 가능한
- bad_area: 프로세스 주소공간에 주소가 속하지 않는 경우

#### Allocate a New Page Frame

- handle_pte_fault()
  - 접근 페이지가 없으면
    - 새 페이지 할당 후 초기화: Demand paging
  - 접근 페이지가 있지만 readonly일 때
    - 기존의 페이지 프레임을 할당하고 내용은 앞의 내용을 복붙: Copy On Write

##### Demand Paging



##### Copy On Write