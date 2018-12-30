Locking
=====

1
-------
xv6는 멀티 프로세서에서 다수의 CPU들은 독립적으로 실행된다.

이러한 다수의 CPU들은 물리적인 렘을 공유하고 xv6는 데이터 스트럭처를 공유(read, write)할 수 있도록 유지 시켜준다. 
- 이는 한 cpu에서 데이터를 업데이트하는 동안 다른 cpu가 읽는 작업을 하는 현상을 야기할 수 있다. 
- 그러므로 병렬 접근에 대한 설계를 잘못하면 데이터 구조가 손상 될 수 있다. 

이러한 일은 단일 프로세서에서도 인터럽트 루틴과 동일한 데이터를 사용하는 인터럽트 루틴이 동시에 발생하면 일어 날 수 있다. 

2
----
동시에 공유 데이터에 접근 하는 모든 코드는 동시 사용(컨커런시)에도 불구하고 올바름(correctness)을 만족할 수 있는 전략이 필요하다. 

이는 다중 코어 혹은 다중 스레드 혹은 인터럽트 코드로 인해 발생한다. xv6는 소수의 간단한 전략을 사용한다. 
이것이 lock이다. 

3
---
lock은 상호 제외 기능을 제공한다. lock을 공유 데이터 항목과 연결하고 코드를 항상 보유하고 있다. 지정된 항목을 사용할 때 관련 lock이 있으면 항목이 사용중인지 알 수 있다. 이로서 데이터는 lock에 의해 보호 받는다. 

4
----
이 장에서는 
1. xv6에서 lock이 필요한 이유
2. xv6에서 lock을 구현한 방법
3. 사용하는 방법

에 대하여 살펴볼 것이다. 
xv6에서 몇가지 코드를 살펴보면, 다른 프로세서(또는 인터럽트)가 종속된 데이터(또는 하드웨어 리소스)를 수정하여 코드의 의도 된 동작을 변경할 수 있는지 생각해봐야 한다.

C언어는 여러개의 기계어가 될 수 있으므로 atomic하게 동작하지 않을 수 있다. 

동시성은 정확성을 추록하기 어렵게 한다. 


Race conditions
===============
잠금의 예시)
- 단일 디스크를 공유하는 여러 프로세서 
    - 디스크 드라이버는 미해결 디스크 요청의 링크 된 목록을 유지 관리(4226)
        - idequeue가 실행 대기열 인듯
    - 프로세스는 목록에 동시에 새 요청을 추가(4354)
        - iderw함수로 (142~146까지는 버퍼에 대하여 검사를 수행 b)
        - 락
        - request가 끝날 때까지 sleep
        


동시 요청이 없으면 다음 과 같이 링크된 목록을 구현할 수 있다. 

{{impage}}

isolation되게 실행할 경우 문제가 없지만 그렇지 않은 경우 2개 이상은 cpu에서 동시의 insert가 일어날 경우 데이터 손실이 발생할 수 있다. (그림과 같이 수행이 일어날 경우 데이터 손실, 먼저 할당된 노드가 손실) -lost update-
- shared data: list


race condition의 조건은 동시에 접근되고 적어도 하나는 write이다. 

race의 실행 결과는 CPU의 실행 time과 메모리 operations에 따라 달라진다. 따라서 이를 디버깅하기가 어렵다. 
ex) print문만 넣어도 실행 시간이 변경되어 race가 사라질 수 있음

race를 피하는 일반적인 방법은 lock을 사용하는 것이다. 이는 상호 배제를 보장하기 때문에 한번에 하나의 CPU만 수항하는 것을 보장해준다. 

단지 코드 몇개를 추가하면 lock이 보장된 버전이 된다. 

lock과 release로 보장된 인스트럭션 시퀀스들을 critical section이라고 한다. 

invariant
---
- lock의 목적은 데이터를 보호함에 있어서 불변성(invariant)을 지키는 것이다.
- 불변성은 데이터의 구조의 속성이 유지되는 것을 의미한다. 
- 작업은 일시적으로 불변성을 위반할 수 있지만 완료하기 전에 다시 설정해야 한다. 
- eg)15행에서 위반되고 16행에서 다시 만족함
- race가 발생한 이유도 불변성에 의존하는 코드를 실행했기 때문
- lock을 사용할때 이러한 critical 영역에서 하나의 CPU만 동작할 수 있으므로 불변 조건이 유지 되지 않을 때 하나의 CPU만 동작한다. 


lock은 크리티컬 섹션을 critical section을 순차적으로 수행하므로서 불변성을 보존한다. (고립된 상태에서 맞다고 가정)

critical section이 atomic하다고 생각할 수 있고 전체적인 변경 사항을 보고 부분적인 업데이트를 볼 순 없다. 

lock이 걸리는 부분을 최소화하기 위해서 라인 11, 12, 13과 같이 local에서 작업이 일어나는 순간은 lock으로 막아줄 필요가 없다. 


Code: Locks
====
xv6에는 spin lock과 sleep lock이 있다. 

spin lock
----
- 1501
- 구조적으로 중요한 필드가 락으로 잠겨있다. 
- 0이면 락이 가능하고 1이면 불가능하단 뜻이다. 

21 void

22 acquire(struct spinlock *lk)

23 {

24 for(;;) {

25 if(!lk->locked) {

26 lk->locked = 1;

27 break;

28 }

29 }

30 }

이 코드에서 25와 26 사이에 atomicity가 보장되지 않으므로 이 코드는 멀티 프로세서에서 서로의 차단를 보장하지 않는다. 

이를 보장하기 위해 xv6는 x86 명령어인 xchg(0569)를 사용한다.
- 0569코드 이해 다시


To execute those two lines atomically, xv6 relies on a special x86 instruction, xchg (0569). 

In one atomic operation, xchg swaps a word in memory with the contents of a
register. 

The function acquire (1574) repeats this xchg instruction in a loop; each iteration atomically reads lk->locked and sets it to 1 (1581). 

If the lock is already held, lk->locked will already be 1, so the xchg returns 1 and the loop continues. 

If the xchg returns 0, however, acquire has successfully acquired the lock—locked was 0 and is now 1—so the loop can stop. 

Once the lock is acquired, acquire records, for debugging, the CPU and stack trace that acquired the lock. 

If a process forgets to release a lock, this information can help to identify the culprit. 

These debugging fields are protected by the lock and must only be edited while holding the lock.