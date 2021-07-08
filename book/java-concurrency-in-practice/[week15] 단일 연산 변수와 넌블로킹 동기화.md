# 단일 연산 변수와 넌블로킹 동기화
- Semaphore, ConcurrentLinkedQueue와 같이 java.util.concurrent 패키지에 들어 있는 다수의 클래스는 단순하게 synchronized 구문으로 동기화를 맞춰 사용하는 것에 비교하면 속도와 확장성이 좋음
- 넌블로킹 알고리즘은 운영체제나 JVM에서 프로세스나 스레드를 스케줄링 하거나 가비지 컬렉션 작업, 그리고 락이나 기타 병렬 자료 구조를 구현하는 부분에서 굉장히 많이 사용하고 있음
- 넌블로킹 알고리즘은 훨씬 세밀한 수준에서 동작하며, 여러 스레드가 동일한 자료를 놓고 경쟁하는 과정에서 대기 상태에 들어가는 일이 없기 떄문에 스케줄링 부하를 대폭 줄여줌
	- 데드락이나 기타 활동성 문제가 발생할 위험도 없음
- AtomicInteger나 AtomicReference 등의 단일 연산 변수를 사용해 넌블로킹 알고리즘을 효율적으로 구현할 수 있음
	- 단일 연산 변수는 volatile 변수와 동일한 메모리 유형을 갖고 있으며 이에 덧붙여 단일 연산으로 값을 변경할 수 있는 기능을 갖고 있음

## 15.1 락의 단점
- 락을 기반으로 세밀한 작업을 주로 하도록 구현돼 있는 클래스는 락에 대한 경쟁이 심해질수록 실제로 필요한 작업을 처리하는 시간 대비 동기화 작업에 필요한 시간의 비율이 상당한 수치로 높아질 가능성이 있음
- volatile 변수는 락과 비교해 봤을 때 컨텍스트 스위칭이나 스레드 스케쥴링과 아무런 연관이 없기 때문에 락보다 훨씬 가벼운 동기화 방법임
	- 그러나 volatile 변수는 락과 비교할 때 가시성 측면에서는 비슷한 수준을 보장하긴 하지만, 복합 연산을 하나의 단일 연산으로 처리할 수 있게 해주는 기능은 전혀 갖고 있지 않음
	- 하나의 변수가 다른 변수와 관련된 상태로 사용해야 하거나, 하나의 변수라도 해당 변수의 새로운 값이 이전 값과 연관이 있다면 volatile 변수를 사용할 수가 없음
	- volatile 변수로는 카운터나 뮤텍스를 구현할 수 없으며, 전체적으로 volatile 변수를 사용할 수 있는 부분이 상당히 제한됨
- Count 클래스는 스레드간 경쟁이 심한 상황에서 컨텍스트 스위칭 부하와 스케쥴링 관련 지연 현상이 발생하며 성능이 떨어짐
- 스레드가 락을 확보하기 위해 대기하고 있는 상태에서 대기중인 스레드는 다른 작업을 전혀 못함
	- 락을 확보하고 있는 스레드의 작업이 지연되면 해당 락을 확보하기 위해 대기하고 있는 모든 스레드의 작업이 전부 지연됨
	- 락을 확보하고 지연되는 스레드의 우선 순위가 떨어지고 대기 상태에 있는 스레드의 우선 순위가 높다면 프로그램 성능에 심각한 영향을 미치는 우선 순위 역선 현상이 발생함
- 카운터 값을 증가시키는 등의 세밀한 작업은 연산을 동기화하기에는 락은 무거운 방법임

</br>

## 15.2 병렬 연산을 위한 하드웨어적인 지원
- 배타적인 락 방법은 보수적인 동기화 기법
	- 락을 확보하고 나면 다른 스레드가 절대 간섭하지 못하는 구조
- 세밀하고 단순한 작업을 처리하는 경우에는 일반적으로 훨씬 효율적으로 동작할 수 있는 낙관적인 방법이 존재
	- 일단 값을 변경하고 다른 스레드의 간섭 없이 값이 제대로 변경되는 방법
	- 충돌 검출 방법을 사용해 값을 변경하는 동안 다른 스레드에서 간섭이 있었는지를 확인할 수 있으며, 간섭이 있었다면 해당 연산이 실패하게 되고 이후에 재시도하거나 아예 재시도조차 하지 않기도 함
- 최근에는 거의 모든 프로세서에서 읽고-변경하고-쓰는 단일 연산을 하드웨어적으로 제공함


### 15.2.1 비교 후 치환
- IA32나 Sparc와 같은 프로세서에서 채택하고 있는 방법은 비교 후 치환(CAS, compare and swap) 명령을 제공하는 방법
- CAS 연산에는 3개의 인자를 넘겨줌
	- 작업할 대상 메모리의 위치인 V, 예상하는 기존 값인 A, 새로 설정할 값인 B
- CAS 연산은 V 위치에 있는 값이 A와 같은 경우에 B로 변경하는 단일 연산
- 이전 값이 A와 달랐다면 아무런 동작도 하지 않고, 값을 B로 변경했던 못했던 간에 V를 리턴함
- CAS 연산은 성공적으로 치환할 수 있을것이라고 희망하는 상태에서 연산을 실행해보고, 값을 마지막으로 확인한 이후에 다른 스레드가 해당하는 값을 변경했다면 그런 사실이 있는지를 확인이나 하자는 의미
~~~java
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;
    
    public synchronized int get() { return  value; }
    
    public synchronized int compareAndSwap(int expectedValue, int newValue) {
        int oldValue = value;
        if(oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }
    
    public synchronized boolean compareAndSet(int expectedValue, int newValue) {
        return (expectedValue == compareAndSwap(expectedValue, newValue));
    }
}
~~~
- 여러 스레드가 동시에 CAS 연산을 사용해 한 변수의 값을 변경하려고 한다면, 
	- 스레드 가운데 하나만 성공적으로 값을 변경할 것이고, 다른 나머지 스레드는 모두 실패
- 값을 변경하지 못했다고 해서 락을 확보하는 것처럼 대기 상태에 들어가는 대신, 값을 변경하지 못했지만 다시 시도할 수 있다고 통보를 받는 셈
- CAS 연산에 실패한 스레드도 대기 상태에 들어가지 않기 때문에 스레드마다 CAS 연산을 다시 시도할 것인지, 아니면 다른 방법을 취할 것인지, 아니면 아무 조치도 취하지 않을 것인지 결정할 수 있음
- CAS 연산을 사용하면 다른 스레드와 간섭이 발생했는지를 확인할 수 있기 때문에 락을 사용하지 않으면서도 읽고-변경하고-쓰는 연산을 단일 연산으로 구현해야 한다는 문제를 간단하게 해결해줌


### 15.2.2 넌블로킹 카운터
~~~java
@ThreadSafe
public class CasCounter {
    private SimulatedCAS value;
    
    public int getValue() {
        return value.get();
    }
    
    public int increment() {
        int v;
        do {
            v = value.get();
        }
        while (v != value.compareAndSwap(v, v + 1));
        return v + 1;
    }
}
~~~
- CAS 연산을 사용해 대기 상태로 들어가지 않으면서도 스레드 경쟁에 안전한 카운터 클래스
- 이전 값을 가져오고, 1을 더해 새로운 값으로 변경하고, CAS 연산으로 새 값을 설정함
- CAS 연산이 실패하면 그 즉시 전체 작업을 재시도
- CAS 기반의 클래스가 락 기반의 클래스보다 성능이 훨씬 좋고, 경쟁이 없는 경우에도 나은 경우가 존재함
	- 락 기반으로 구현한 카운터 클래스에서 가장 최적의 조건으로 실행되는 경우에도 CAS 기반의 카운터 클래스에서 일반적인 경우에 해당하는 경우보다 더 많은 작업을 하는 셈
- CAS 연산은 대부분 성공하는 경우가 많기 때문에 하드웨어에서 분기 지점과 흐름을 예측해 코드에 들어 있는 while 반복문을 포함한 복잡한 논리적인 작업 흐름 구조에서 발생할 수 있는 부하를 최소화할 수 있음
- 애플리케이션 수준에서는 코드가 더 복잡해보이지만 JVM이나 운영체제의 입장에서는 훨씬 적은 양의 프로그램만 실행하는 셈
- CAS의 단점은 호출하는 프로그램에서 직접 스레드 경쟁 조건에 대한 처리를 해야 한다는 점이 있는데, 반면 락을 사용하면 락을 사용할 수 있을 때까지 대기 상태에 들어가도록 하면서 스레드 경쟁 문제를 알아서 처리해준다는 차이점이 있음


### 15.2.3 JVM에서의 CAS 연산 지원
- 자바 5.0부터는 int, long 그리고 모든 객체의 참조를 대상으로 CAS 연산이 가능하도록 기능이 추가됐고, JVM은 CAS 연산을 호출받았을 때 해당하는 하드웨어에 적당한 가장 효과적인 방법으로 처리하도록 되어 있음
- 저수준의 CAS 연산은 단일 연산 변수 클래스, 즉 AtomincInteger와 같이 java.uti.concurrent.atomic 패키지의 AtomicXxx 클래스를 통해 제공함

</br>

## 15.3 단일 연산 변수 클래스
- 단일 연산 변수는 락보다 훨씬 가벼우면서 세밀한 구조를 갖고 있으며, 멀티 프로레서 시스템에서 고성능의 병렬 프로그램을 작성하고자 할 때 핵심적인 역할을 함
- 단일 연산 변수를 사용하면 스레드가 경쟁하는 범위를 하나의 변수로 좁혀주는 효과, 이정도의 범위는 프로그램에서할 수 있는 가장 세밀한 범위임
- 락 대신 단일 연산 변수를 기반의 알고리즘으로 구현된 프로그램은 내부의 스레드가 지연되는 현상이 거의 없으며, 스레드 간의 경쟁이 발생한다 하더라도 훨씬 쉽게 경쟁 상황을 헤쳐나갈 수 있음
- 단일연산변수 클래스는 volatile 변수에서 읽고-변경하고-쓰는 것과 같은 조건부 단일 연산을 지원하도록 일반화한 구조임
	- AtomicInteger 클래스는 int 값을 나타내며, get/set 메소드, compareAndSet 메소드도 제공하며, 그외의 편의 사항인 단일 연산으로 값을 더하거나 증가, 감소시키니는 메소드도 제공함 
- AtomicIntger는 Counter 클래스 보다 훨씬 높은 확장성을 제공함
	- 동기화를 위한 하드웨어의 기능을 직접적으로 활용할 수 있기 때문
- 많이 사용되는 클래스는 AtomicInteger, AtomicLong, AtomicBoolean, AtomicReference 클래스
	- 모두 CAS연산 제공
	- AtomicInteger와 AtomicLong은 간단 산술 기능도 제공
- 단일 연산 배열 변수 클래스는 배열의 각 항목을 단일 연산으로 업데이트 할 수 있도록 구성돼 있는 배열 클래스
	- 각 항목에 대해 volatile 변수와 같은 구조의 접근 방법을 제공하며, 일반적인 배열에서는 제공하지 않는 기능임
- 해시 값을 기반으로 하는 클래스에 키 값으로 사용하기에는 적절하지 않음


### 15.3.1 '더 나은 volatile' 변수로의 단일 연산 클래스
- 범위라는 조건은 항상 두 변수의 값을 동시에 사용해야 하며 필요한 조건을 만족하면서 그와 동시에 양쪽 범위 값을 동시에 업데이트할 수 없기 떄문에 volatile 참조를 사용하거나 AtomicInteger를 사용한다 해도 확인하고 동작하는 연산을 안전하게 수행할 수 없음
~~~java
public class CasNumberRange {
    @Immutable
    private static class IntPair {
        final int lower; //불변조건 : lower <= upper;
        final int upper;

        public IntPair(int lower, int upper) {
            this.lower = lower;
            this.upper = upper;
        }
    }
    private final AtomicReference<IntPair> values = new AtomicReference<IntPair>(new IntPair(0,0));
    
    public int getLower() {return values.get().lower; }
    public int getUpper() {return values.get().upper; }
    
    public void setLower(int i) {
        while (true) {
            IntPair oldv = values.get();
            if( i > oldv.upper) {
                throw new IllegalArgumentException("Can't set lower to " + i + " > upper");
            }
            IntPair newv = new IntPair(i, oldv.upper);
            if(values.compareAndSet(oldv, newv)) {
                return;
            }
        }
    }
    //setUppser도 비슷
}
~~~
- CasNumberRange 클래스는 범위 양쪽에 해당하는 숫자 두 개를 갖고 있는 IntPair 클래스에 AtomicReference 클래스를 적용함
- compareAndSet 메소드를 사용해 NumberRange와 같은 경쟁 조건이 발생하지 않게 하면서 범위를 표현하는 값 두개를 한꺼번에 변경할 수 있음

### 15.3.2 성능 비교: 락과 단일 연산 변수
- 락과 단일 연산 변수 간의 확장성의 차이점을 확인할 수 있도록 여러 가지 방법으로 구현된 난수 발생기의 처리 속도를 비교하는 벤치 마크 테스트를 준비함
- 난수 발생기는 항상 이전 결과 값을 내부 상태로 보존하고 있어야 함
~~~java
@ThreadSafe
public class ReentrantLockPseudoRandom  extends PseudoRandom {
    private final Lock lock = new ReentrantLock(false);
    public int seed;
    
    ReentrantLockPseudoRandom(int seed) {
        this.seed = seed;
    }
    
    public int nextInt(int n) {
        lock.lock();
        try {
            int s = seed;
            seed = calculateNext(s);
            int remainder = s % n;
            return remainder > 0 ? remainder : remainder + n;
        } finally {
            lock.unlock();
        }
    }
}

@ThreadSafe
public class AtomicPseudoRandom extends PseudoRandom {
    private AtomicInteger seed;
    
    AtomicPseudoRandom(int seed) {
        this.seed = new AtomicInteger(seed);
    }
    
    public int nextInt(int n) {
        while (true) {
            int s = seed.get();
            int nextSeed = calculateNext(s);
            if(seed.compareAndSet(s, nextSeed)) {
                int remainder = s % n;
                return remainder > 0 ? remainder : remainder + n;
            }
        }
    }
}
~~~
- 스레드 안전한 난수 발생 함수의 코드
	- 하나는 ReentrantLock을 사용해 구현, 하나는 AtomicInteger 사용해 구현
- 스레드 내부의 값만을 사용해 '복잡한 작업'에 해당하는 반복 작업을 수행함

![LockAndActomic](/img/LockAndActomic.png)  

- 매번 반복될 때마다 각각 적은 양과 많은 양에 해당하는 작업을 처리하는 경우의 성능을 볼 수 있음
- 스레드 내부의 데이터만으로 처리하는 작업량이 적은 경우에는 락이나 단일 연산 변수 쪽에서 상당한 경쟁 상황을 겪음
- 스레드 내부 작업의 양이 많아지면 상대적으로 락이나 단일 연산 변수에서 경쟁 상황이 덜 벌어짐
- 경쟁이 많은 상황에서는 단일 연산 변수보다 락이 더 빠르게 처리되고, 실제적인 경쟁 상황에서는 단일 연산 변수가 락보다 성능이 더 좋음
- 단일 연산 변수를 사용하면 경쟁 조건에 대한 처리 작업의 책임이 경쟁하는 스레드에게 넘어감
	- CAS 연산 기반의 알고리즘이 대부분 그렇지만 경쟁이 발생하면 그 즉시 재시도하는것으로 대응하며, 경쟁이 심할 경우 경쟁을 계속 심하게 만드는 요인이 되기도 함
- 단일 연산 변수는 일반적인 경쟁 수준에서 경쟁 상황을 더 효율적으로 처리하기 떄문에 단일 연산 변수가 락에 비해서 확장성이 좋음
- 경쟁이 적거나 보통 수준의 경쟁 수준에서는 단일 연산 변수를 사용해야 확장성을 높일 수 있고 경쟁 수준이 아주 높은 경우에는 락을 사용해야함

</br>

## 15.4 넌블로킹 알고리즘
- 락 기반으로 동작하는 알고리즘은 항상 다양한 종류의 가용성 문제에 직면할 위험이 존재함
- 락을 현재 확보하고 있는 스레드가 I/O 작업 때문에 대기중이라거나, 메모리 페이징 때문에 대기 중이라거나, 기타 어떤 원인 때문에라도 대기 상태에 들어간다면 다른 모든 스레드가 전부 대기 상태에 들어갈 가능성이 있음
- 특정 스레드에서 작업이 실패하거나 또는 대기 상태에 들어가는 경우에 다른 어떤 스레드라도 그로 인해 실패 하거나 대기 상태에 들어가지 않는 알고리즘을 넌블로킹 알고리즘이라 함
- 각 작업 단계마다 일부 스레드는 항상 작업을 진행할 수 있는 경우 락 프리 알고리즘이라고 함
	- CAS를 독점적으로 사용할 경우 락 프리 특성과 대기상태에 빠지지 않는 특성 가지게 됨
- 넌블로킹 알고리즘은 데드락이나 우선 순위 역전 등의 문제점이 발생하지 않음
- 스택, 큐, 우선순위 큐, 해시 테이블 등과 같은 일반적인 데이터 구조를 구현할 때 대기 상태에 들어가지 않는 좋은 알고리즘이 많이 공개되어 있음


### 15.4.1 넌블로킹 스택
- 대기 상태에 들어가지 않도록 구현한 알고리즘은 락 기반으로 구현한 알고리즘에 비해 상당히 복잡한 경우가 많음
- 넌블로킹 알고리즘을 구성할 때 가장 핵심이 되는 부분은 바로 데이터의 일관성을 유지하면서 단일 연산 변경 작업의 범위를 단 하나의 변수로 제한하는 부분임
- 큐와 같이 연결된 구조를 갖는 컬렉션 클래스에서는 상태 전환을 개별적인 링크에 대한 변경 작업이라 간주하고, ActomicReference로 각 연결 부분을 관리해서 단일 연산으로만 변경할 수 있도록하면 어느정도 구현이 가능함
~~~java
@ThreadSafe
public class ConcurrentStack <E>{
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();
    
    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }
    
    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        
        do {
            oldHead = top.get();
            if(oldHead == null)
                return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.item;
    }
    
    private static class Node<E> {
        public final E item;
        public Node<E> next;
        
        public Node(E item) {
            this.item = item;
        }
    }
}
~~~
- 단일 연산 참조를 사용해 스택을 어떻게 구현하는지 보여주는 좋은 예
- 스택 자체는 Node 클래스로 구성된 연결 리스트이며 최초 항목은 top 변수에 들어있고, 각 항목마다 자신의 값과 다음 항목에 대한 참조를 갖고 있음
- push 메소드에서는 새로운 노드 인스턴스를 생성하고, 새 노드의 next 연결값으로 현재의 top 항목을 설정한 다음, CAS 연산을 통해 새로운 노드 스택의 top을 설정함
- CAS 연산이 성공하거나 실패하는 어떤 경우라 해도 스택은 항상 안정적인 상태를 유지함
- CasCounter와 ConcurrentStack 클래스는 대기 상태에 들어가지 않는 알고리즘의 여러 가지 특성을 모두 보여주고 있음
	- 경쟁 상황이라면 재시도할 준비를 하고 있음
- ConcurrentStack에서와 같이 대기 상태에 들어가지 않는 알고리즘은 락과 같이 compareAndSet 연산을 통해 단일 연산 특성과 가시성을 보장하기 때문에 스레드 안전성을 보장함
- 특정 스레드에서 스택의 상태를 변경했다면 상태를 변경할 때 volatile 쓰기 특성이 있는 compareAndSet 연산을 사용해야만 함

### 15.4.2 넌블로킹 연결 리스트
- 넌블로킹 알고리즘을 작성할 때의 핵심은 바로 단일 연산의 범위를 단 하나의 변수로 제한하는 부분임
- 연결 큐는 리스트의 머리와 꼬리 부분에 직접적으로 접근할 수 있어야 하기 때문에 스택보다 훨씬 복잡한 구조를 갖고 있음
- 새로운 항목을 연결 큐에 추가하려면 마지막 항목을 가리키는 두 개의 참조가 동시에 단일 연산으로 변경되어야 함
- 두 개의 참조를 업데이트 할 때 두 번의 CAS 연산이 필요한 데, 만약 첫번째 CAS연산이 성공했지만 두번째 CAS 연산이 실패했다고 하면 연결 큐가 올바르지 않은 상태에 놓이게 됨
- 연결 큐를 대기 상태에 들어가지 않도록 구현할 수 있는 알고리즘은 두 가지 경우를 모두 처리할 수 있어야 함
~~~java
@ThreadSafe
public class LinkedQueue<E> {
    private static class Node<E> {
        final E item;
        final AtomicReference<Node<E>> next;

        public Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<Node<E>>(next);
        }
    }
    
    private final Node<E> dummy = new Node<E>(null, null);
    private final AtomicReference<Node<E>> head = new AtomicReference<Node<E>>(dummy);
    private final AtomicReference<Node<E>> tail = new AtomicReference<Node<E>>(dummy);
    
    public boolean put(E item) {
        Node<E> newNode = new Node<E>(item, null);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> tailNext = curTail.next.get();
            if(curTail == tail.get()) {
                if(tailNext != null) {  // A
                    // 큐는 중간 상태이고, 꼬리 이동
                    tail.compareAndSet(curTail, tailNext); // B
                } else {
                    // 평온한 상태에서 항목 추가 시도
                    if(curTail.next.compareAndSet(null, newNode)) { // C
                        // 추가 작업 성공, 꼬리 이동 시도
                        tail.compareAndSet(curTail, newNode); // D
                        return true;
                    }
                }
            }
        }
    }
}
~~~
- LinkedQueue 클래스를 보면 마이클 스콧의 넌블로킹 연결 큐 알고리즘 가운데 값을 추가하는 부분의 코드가 소개되어 있음
- 값이 없어 비어 있는 큐는 '표식' 또는 '의미 없는' 노드만 갖고 있고, 머리와 꼬리 변수는 이와 같은 표식 노드를 참조하고 있음
- 꼬리 변수는 항상 표식 노드, 큐의 마지막 항목, 또는 맨 뒤에서 두 번째 항목을 가리킴
- 새로운 항목을 추가하려면 두 개의 참조를 변경해야 함
- 현재 큐의 마지막 항목이 갖고 있는 next 참조 값을 변경해서 새로운 항목의 큐의 끝에 연결하는 작업
- 꼬리를 가리키는 변수가 새로 추가된 항목을 가리키도록 참조를 변경하는 작업
- 큐가 평온한 상태에 있을 때 tail 변수가 가리키는 항목의 next 값이 null을 유지하도록 하고, 반대로 중간 상태인 경우에는 tail이 가리키는 항목의 next 값이 null이 아닌 값을 갖도록 하는 전략은 위의 두가지 전략을 모두 성공적으로 구현하는 방법

### 15.4.3 단일 연산 필드 업데이터
~~~java
public class ConcurrentLinkedQueue<E> {
    private static class Node<E> {
        private final E item;
        private volatile Node<E> next;

        public Node(E item) {
            this.item = item;
        }
    }

    private static AtomicReferenceFieldUpdater<Node, Node> nextUpdater 
    	= AtomicReferenceFieldUpdater.newUpdater(Node.class, Node.class, "next");
}
~~~
- 각 Node 인스턴스를 단일 연산 참조 클래스로 연결하는 대신 일반적인 volatile 변수를 사용해 연결하고, 연결 구조를 변경할 때는 리플렉션 기반의 AtomicReferenceFeildUpdater 클래스를 사용해 변경
- 단일 연산 필드 업데이터 클래스는 현재 사용중인 volatile 변수에 대한 리플렉션 기반의 '뷰'를 나타내며 따라서 일반 volatile 변수에 대해 CAS 연산을 사용할 수 있도록 해줌
- 단일 연산 필드 업데이터 클래스는 생성 메소드가 없으며, 인스턴스를 생성하려면 생성 메소드를 호출하는 대신 newUpdater라는 팩토리 메소드에 해당하는 클래스와 필드, 즉 변수의 이름을 넘겨서 생성할 수 있음
- 업데이터 클래스에서 보장하는 단일성은 일반적인 단일 연산 변수보다 약함
- 업데이터 클래스에 지정한 클래스의 지정 변수가 업데이터 클래스를 통하지 않고 직접 변경하는 경우가 있다면 연산의 단일성을 보장할 수 없음
- 필드 업데이터 클래스로 지정한 변수에 대해 연산의 단일성을 보장하려면 모든 스레드에서 해당 변수의 값을 변경할 때 항상 compareAndSet 메소드나 기타 산술 연산 메소드를 사용해야만 함 


### 15.4.4 ABA 문제
- ABA 문제는 노드를 재사용하는 알고리즘에서 CAS연산을 고지식하게 사용하다 보면 발생할 수 있는 이상 현상을 말함
- CAS연산은 "V 변수의 값이 여전히 A인지?"를 확인하고 만약 그렇다면 값을 B로 변경하는 작업을 말함
- 간혹 "V 변수의 값이 내가 마지막으로 A값이라고 확인한 이후에 변경된 적이 있는지?" 라는 질문의 답을 알아야 할 경우도 생김
- 일부 알고리즘은 V변수의 값이 A -> B -> 다시 A로 변경된 경우 변경사항이 있었다는 것으로 인식하고 그에 해당하는 재시도 절차를 밟아야 할 필요가 있기도 함
	- ABA 문제는 연결 노드 객체에 대한 메모리 관리 부분을 직접 처리하는 알고리즘을 사용할 때 많이 발생함
	- 해결 방법 -> 참조값 하나만 변경하는 것이 아니라, 참조와 버전 번호의 두가지 값을 한꺼번에 변경하는 방법
- AtomicStampedReference 클래스는 
	- 두 개의 값에 대한 조건부 단일 연산 업데이트 기능을 제공함
	- 객체에 대한 참조와 숫자 값을 함께 변경하며, 버전 번호를 사용해 ABA 문제가 발생하지 않는 참조의 역할을 함
- AtomicMarkableReference 클래스 역시 이와 유사하게 객체에 대한 참조와 불린 값을 변경하며 일부 알고리즘 노드 객체를 그대로 놓아두지만 삭제된 상태임을 표시하는 기능으로 활용하기도 함

</br>


## 요약
- 대기 상태에 들어가지 않는 넌블로킹 알고리즘은 락 대신 비교 후 치환 같은 저수준의 명령을 활용해 스레드 안전성을 유지하는 알고리즘임
- 저수준의 기능은 특별하게 만들어진 단일 연산 클래스를 통해 사용할 수 있으며, 단일 연산 클래스는 '더 나은 volatile 변수'로써 정수형 변수나 객체에 대한 참조 등을 대상으로 단일 연산 기능을 제공하기도 함
- 넌블로킹 알고리즘은 설계하고 구현하기는 훨씬 어렵지만 특정 조건에서는 훨씬 나은 확장성을 제공하기도 하고, 가용성 문제를 발생시키지 않는다는 장점이 있음
- JVM이나 플랫폼 라이브러리 대기 상태에 들어가지 않는 알고리즘을 적절히 활용하는 범위가 넓어지면서 JVM의 버전이 올라갈 때마다 병렬 프로그램의 성능이 좋아지고 있음