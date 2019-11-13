# Lecture 7. Kernel Synchronization

## Kernel Control Paths

### Interleaving Kernel Control Paths

## Kernel Synchronization Primitives

### Atomic Operations

### Barriers

---

### Locking

#### Lock Contention and Scalability

> Non-blocking lock

#### Spin Locks

#### Read/Write Spin Locks

##### Spin Locks in Interrupt Handlers

> Blocking lock

#### Kernel Semaphores

##### Using Kernel Semaphores

##### MUTEX

##### Counting Semaphore

#### Read/Write Semaphores

#### Mutexes

#### Completions

#### BKL

#### Spin Locks vs. Semaphores (Non-blocking lock vs. Blocking lock)

---

### Read-Copy-Update(RCU)

---

### Preemption Disabling

### Local Interrupt Disabling

---

### 어떤 Synchronizaton Primitive를 사용해야 할까?