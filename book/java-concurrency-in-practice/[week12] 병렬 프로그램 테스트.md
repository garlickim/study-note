# 병렬 프로그램 테스트
- 병렬 프로그램을 테스트한 결과는 안전성과 활동성의 문제로 귀결됨
- 활동성은 진행중인 상태와 진행이 멈춘 상태를 테스트 하는 경우가 많고, 이때 더 이상 실행되지 않는 것처럼 보이는 경우가 생기면 실행 속도가 느린건지 실행도중 멈추는 오류가 발생한건지 확인해야 함
- 활동성 문제를 테스트하는 것은 성능 문제를 테스트 하는 것과 밀접한 관련이 있음
- 성능은 아래와 같이 수치화해 측정할 수 있음
	 - 처리량(throughput) : 병렬로 실행되는 여러 개의 작업이 각자가 할일을 끝내는 속도
	 - 응답성(responsiveness) : 요청이 들어온 이후 작업을 마치고 결과를 줄 때까지의 시간. 지연시간(latency) 이라고도 함
	 - 확장성(scalability) : 자원을 더 많이 확보할 때마다 그에 따라 처리할 수 있는 작업량이 늘어나는 정도

 </br>

## 정확성 테스트
~~~java
@ThreadSafe
public class BoundedBuffer<E> {
    private final Semaphore availableItems, availableSpaces;
 
    @GuardedBy("this") private final E[] items; 
    @GuardedBy("this") private int putPosition = 0, takePosition = 0;
    
    public BoundedBuffer(int capacity) {
        availableItems = new Semaphore(0);
        availableSpaces = new Semaphore(capacity); 
        items = (E[]) new Object[capacity];
    }
    
    public boolean isEmpty() {
        return availableItems.availablePermits() == 0;
    }
    
    public boolean isFull() {
        return availableSpaces.availablePermits() == 0;
    }
    
    public void put(E x) throws InterruptedException {  
        availableSpaces.acquire();  
        doInsert(x);
        availableItems.release(); 
    }
    
    public E take() throws InterruptedException { 
        availableItems.acquire(); 
        E item = doExtract();
        availableSpaces.release(); 
        return item;
    } 
    
    private synchronized void doInsert(E x) {
        int i = putPosition;
        items[i] = x;
        putPosition = (++i == items.length) ? 0 : i;
    }
    
    private synchronized E doExtract() {
        int i = takePosition;
        E x = items[i];
        items[i] = null;
        takePosition = (++i == items.length) ? 0 : i;
        return x;
    }
}
~~~
- Semaphore를 사용해 크기를 제한하고 제한된 크기를 초과한 경우에 대기상태에 들어가도록 함
- put(), take()는 개수가 지정된 세마포어를 사용해 동기화를 맞추고 있음
- avaliableItems 세마포어는 현재 버퍼 내부에서 뽑아 낼 수 있는 항목의 개수를 담고 있음
- take()는 avaliableItems 세마포어에서 가져갈 항목이 있는지에 대한 확인을 받아야 함
	- 버퍼에 항목이 하나 이상 들어 있었다면 즉시 확인에 성공하고, 버퍼가 비어있다면 대기 상태에 들어감


### 12.1.1 가장 기본적인 단위 테스트
- 가장 기본적인 단위 테스트는 BoundedBuffer 인스턴스를 하나 생성하고, 다양한 메소드를 호출한 뒤 최종 상태와 변수 값 등을 확인해보는 방법
~~~java
public class BoundedBufferTest extends TestCase {
   void testIsEmptyWhenConstructed() {
       BoundedBuffer<Integer> bb = new BoundedBuffer<Integer>(10);
       assertTrue(bb.isEmpty());
       assertFalse(bb.isFull());
   }
   
   void testIsFullAfterPuts() throws InterruptedException {
       BoundedBuffer<Integer> bb = new BoundedBuffer<Integer>(10);
       for(int i = 0; i < 10; i ++) {
           bb.put(i);
       }
       assertTrue(bb.isFull());
       assertFalse(bb.isEmpty());
   }
}
~~~
- 용량이 N인 BoundedBuffer 클래스에 N개의 항목을 추가하고, 버퍼 클래스 스스로가 용량이 가득찼다고 표현하는 코드를 테스트하는 케이스


### 12.1.2 블로킹 메소드 테스트
- 따로 실행되고 있는 스레드에서 성공과 실패 여부를 파악하는 경우에, 파악된 성공 또는 실패 여부를 다시 원래 테스트 케이스의 메소드에 알려줄 수 있는 방법이 마련돼 있어야 테스트 결과를 단위 테스트 프레임웍에서 제대로 리포팅할 수 있음
- 대기 상태에 들어가는 메소드를 테스트할 때는 복잡한 상황이 존재함
	- 테스트 시, 어떤 방법으로건 대기 상태를 풀어서 대기 상태에 들어갔었음을 확인해야 함
	- 가장 확실한 방법은 인터럽트를 거는 방법
- 인터럽트를 활용해 테스트하려면 대기 상태에 들어갈 대상 메소드가 인터럽트에 적절하게 대응하도록(인터럽트 걸리는 즉시 리턴되거나 InterruptedException을 던지는) 만들어져 있어야함
~~~java
void testTakeBlocksWhenEmpty() {
    final BoundedBuffer<Integer> bb = new BoundedBuffer<Integer>(10);
    Thread taker = new Thread() {
        @Override
        public void run() {
            try {
                int unused = bb.take();
                fail();  
            } catch (InterruptedException success) { }
        }
    };
    
    try {
        taker.start();
        Thread.sleep(LOCKUP_DETECT_TIMEOUT);
        taker.isInterrupted();
        taker.join(LOCKUP_DETECT_TIMEOUT);
        assertFalse(taker.isAlive());
    } catch (Exception unexcepted) {
        fail();
    }
}
~~~
- taker 스레드를 실행하고 적당량 이상 대기 후, taker에 인터럽트를 걺
- 만약 taker 스레드가 정사엊ㄱ으로 대기 상태에 들어가 있었다면 인터럽트가 걸렸을 때 InterruptedException을 띄울 것이고, catch 구문에서는 예외가 발생한 상황이 정상이라고 판단 후 스레드를 종료시킴
- taker 스레드를 실행시켰던 테스트 프로그램은 taker 스레드가 종료될 때까지 join 메소드로 기다리고, Thread.isAlive 메소드를 사용해 join 메소가 정상적으로 종료됐는지를 확인
	- taker 스레드가 정상적으로 인터럽트에 응답했다면 join 메소드가 즉시 종료됨
- join 메소드를 사용해 정상적으로 종료되는지를 확인하는 작업은 Thread 클래스를 직접 상속받아 사용하는 편이 나은 몇 안되는 방법 가운데 하나임
- Thread.getState 메소드를 사용하여 대기상태에 들어갔는지 확인하는 방법은 믿을만 하지 못함
	- JVM 구현 방법에 따라 스핀 대기(spin waiting) 기법을 활용할 수도 있으므로, 특정 스레드가 대기 상태라 해도 WAITING 또는 TIMED_WAITING 상태에 놓여있다고 볼 수 없기 때문
- 병렬성을 제어하는 용도로 Thread,getState 메소드를 사용하지 말아야 함


### 12.1.3 안전성 테스트
- 병렬 실행 환경에서는 올바른 테스트 프로그램을 작성하는 일이 테스트 대상 클래스 자체를 구현하는 일보다 훨씬 어려운 경우도 존재함
- 안전성을 테스트하는 프로그램을 효과적으로 작성하려면 뭔가 문제가 발생했을 때 잘못 사용되는 속성을 '높은 확률로' 찾아내는 작업을 해야 함고 동시에 오류를 확인하는 코드가 테스트 대상의 병렬성을 인위적으로 제한해서는 안된다는 점을 고려해야 함
- 테스트 하는 대상 속성의 값을 확인할 때 추가적인 동기화 작업을 하지 않아도 된다면 가장 좋은 상태라 볼 수 있음
- 프로듀서-컨슈머 디자인 패턴을 사용해 동작하는 클래스에 가장 적합한 방법은
	- 큐&버퍼에 추가된 항목을 모두 그대로 뽑아 낼 수 있는지를 확인하고, 그외에는 아무일도 하지 않는지를 확인
- 작성한 테스트 프로그램이 실제로 원하는 내용을 테스트하는지 확인하려면 사용하고 있는 체크섬 연산을 컴파일러가 예측할 수 없는 연산인지도 확인해야 함
- 똑똑한 컴파일러 때문에 발생하는 문제를 해결하기 위해선 테스트 데이터를 일련번호 대신 임의의 숫자를 생성해 사용
	- 단, 허술한 난수 발생기(random number generator)를 사용하면 테스트 결과가 잘못 나올 수 있음
- 허술한 난수 발생기(RNG)는 현재 시간과 클래스 간에 종속성이 있는 난수를 생성하는 경우가 있는데, 대부분의 난수 발생기가 스레드 안전성을 확보한 상태이며 추가적인 동기화 작업이 필요하기 때문
- 범용 난수 발생기를 사용하는 대신 간단한 난수 발생기를 사용하는 것도 좋은 방법
~~~java
static int xorShift(int y) {
    y ^= (y << 6);
    y ^= (y >>> 21);
    y ^= (y << 7);
    return y;
}
~~~
- 중간 품질의 난수 발생기
- 클래스 인스턴스의 hashCode값과 nanoTime을 사용해 xorShift 메소드를 사용하면 거의 예측이 불가능하며 실행할 때마다 새로운 난수를 생성함
~~~java
public class PutTakeTest {
    private static final ExecutorService pool = Executors.newCachedThreadPool();
    private final AtomicInteger putSum = new AtomicInteger(0);
    private final AtomicInteger takeSum = new AtomicInteger(0);
    private final CyclicBarrier barrier;
    private final BoundedBuffer<Integer> bb;
    private final int nTrials, nPairs;

    public static void main(String[] args) {
        new PutTakeTest(10, 10, 10000).test(); // 예제 인자 값
        pool.shutdown();
    }

    PutTakeTest(int capacity, int nPairs, int nTrials) {
        this.bb = new BoundedBuffer<Integer>(capacity);
        this.nPairs = nPairs;
        this.nTrials = nTrials;
        this.barrier = new CyclicBarrier(nPairs * 2 + 1);
    }

    void test() {
        try {
            IntStream.range(0, nPairs)
                    .forEach(i -> {
                        pool.execute(new Producer());
                        pool.execute(new Consumer());
                    });
            barrier.await(); // 모든 스레드가 준비될 때까지 대기
            barrier.await(); // 모든 스레드의 작업이 끝날 때까지 대기
            assertEquals(putSum.get(), takeSum.get());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    class Producer implements Runnable {
        @Override
        public void run() {
            try {
                int seed = (this.hashCode() ^ (int) System.nanoTime());
                int sum = 0;
                barrier.await();
                for(int i = nTrials; i > 0; --i) {
                    bb.put(seed);
                    sum += seed;
                    seed = xorShift(seed);
                }
                putSum.getAndAdd(sum);
                barrier.await();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }

    class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                barrier.await();
                int sum = 0;
                for(int i = nTrials; i > 0; --i) {
                    sum += bb.take();
                }
                takeSum.getAndAdd(sum);
                barrier.await();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
}
~~~
- 각 스레드마다 체크섬을 따로 운영하면 따로 동기화할 필요가 없으며, 경쟁이 발생하지 않아 테스트에만 집중 가능
- 스레드가 처리할 작업이 굉장히 짧은 시간이면 충분한 작업일 때, 이와 같은 스레드를 반복문 내부에서 차례로 생성해 실행시킨다면 최악의 경우에는 각 스레드가 순차적으로 실행될 가능성이 존재함
- CountDownLatch를 사용하여 모든 스레드가 준비될 때까지 대기하거나 모든 스레드가 완료될 때까지 대기하는 방법이 존재함
- CyclicBarrier를 사용해도 비슷한 효과를 낼 수 있음
	- 전체 작업 스레드의 개수에 1을 더한 크기로 초기화해두고, 작업 스레드와 테스트 프로그램이 시작하는 시점에 동시에 시작할 수 있도록 대기하고, 끝나는 시점도 한번에 끝내도록 대기하는 방법 
- 테스트 프로그램은 스레드가 교차 실행되는 경우의 수를 최대한 많이 확보할 수 있도록 CPU가 여러개 장착된 시스템에서 돌려보는 것이 좋음
- 절묘한 타이밍에 공유된 데이터를 사용하다 나타나는 오류를 찾으려면 CPU가 많이 있는 것보다 스레드를 더 많이 돌리는 것이 나음
- 스레드가 많아지면 실행 중인 스레드와 대기 상태에 들어간 스레드가 서로 교차하면서 스레드 간 상호작용이 발생하는 경우의 수가 많아지기 때문
- 미리 지정된 개수만큼 연산을 실행하고 테스트를 마치는 프로그램은 테스트가 종료되지 않고 계속 실행될 가능성이 존재하기 때문에 시간 제한을 두고 테스트를 중단하도록 해야함


### 12.1.4 자원 관리 테스트
- 메모리를 원하지 않음에도 불구하고 계속해서 잡고 있는 경우가 있는지 확인하려면 힙 조사(heap inspection) 도구를 사용해볼 만 함
~~~java
class Big {
        double[] data = new double[10000];
}

void teskLeadk() throws InterruptedException {
    BoundedBuffer<Big> bb = new BoundedBuffer<Big>(CAPACITY);
    int heapSize1 = /* 힙 스냅샷 */ ;
    for(int i = 0; i < CAPACITY; i++) {
        bb.put(new Big());
    }
    
    for(int i = 0; i < CAPACITY; i++) {
        bb.take();
    }
    
    int heapSize2 = /* 힙 스냅샷 */ ;
    
    assertTrue(Math.abs(heapSize1 - heapSize2) < THRESHOLD);
}
~~~
- 힙 조사 도구가 추가한 코드는 GC를 강제로 실행하고 힙 사용량과 기타 메모리 사용 현황을 불러오는 기능을 담당함 


### 12.1.5 콜백 사용
- 콜백 함수는 객체를 사용하는 동안 중요한 시점마다 그 내부의 값을 확인시켜주는 좋은 기회로 사용할 수 있음
- 스레드 풀이 제대로 동작하는지 테스트하려면 실행 정책에 맞게 여러 측면에서 절적한 수치를 뽑아낼 수 있는지를 테스트하면 됨
~~~java
class TestingThreadFactory implements ThreadFactory {
    public final AtomicInteger numCreated = new AtomicInteger(); 
    private final ThreadFactory factory = Executors.defaultThreadFactory();
    
    @Override
    public Thread newThread(Runnable r) {
        numCreated.incrementAndGet();
        return factory.newThread(r);
    }
}
~~~
- TestingThreadFactory 클래스는 생성된 스레드의 개수를 세는 기능을 갖고 있음
- 실제 테스트 케이스에서는 TestingThreadFactory가 알고 있는 스레드의 개수가 올바른지 확인해 볼 수 있음
- 기능이 추가된 Thread 객체를 생성하도록 코드를 변경하면 생성된 스레드가 언제 종료되는지를 추적할 수 있음
	- 테스트 케이스는 없어져야 할 스레드가 적절한 시점에 올바르게 사라지는지도 확인 가능
- 코어 풀 크기가 최대 풀 크기보다 작게 설정돼 있다면 실행할 대상이 늘어날 때마다 스레드의 개수가 함께 늘어나야 함
- 스레드 풀에 오래 실행될 작업을 많이 추가해두면, 스레드 개수가 올바르게 늘어나는지 등의 수치를 확인할 수 있음
~~~java
public void testPoolExpansion() throws InterruptedException  {
    int MAX_SIZE = 10;
    Example12_8.TestingThreadFactory threadFactory = new Example12_8.TestingThreadFactory();
    ExecutorService exec = Executors.newFixedThreadPool(MAX_SIZE, threadFactory);
    
    for(int i = 0; i < 10 * MAX_SIZE; i++) {
        exec.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(Long.MAX_VALUE);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
    }
    
    for(int i = 0; i < 20 && threadFactory.numCreated.get() < MAX_SIZE; i++) {
        Thread.sleep(100);
    }
    
    assertEquals(threadFactory.numCreated.get(), MAX_SIZE);
    exec.shutdownNow();
}
~~~


### 12.1.6 스레드 교차 실행량 확대
- CPU 프로세서의 개수, 운영체제, 프로세서 아키텍처 등을 다양하게 변경하면서 테스트해보면 특정 시스템에서만 발생하는 오류를 찾아낼 수 있음
- 스레드 교차 실행 정도를 크게 높이고, 테스트할 대상 공간을 크게 확대 시킬 수 있는 트릭은 공유된 자원을 사용하는 부분에서 Thread.yield 메소드를 호출해 컨텍스트 스위치가 많이 발생하도록 유도할 수 있음
~~~java
public synchronized void transferCredis(Account from, Account to, int amount) {
    from.setBalance(from.getBalance - amount);
    if(random.nextInt(1000) > THREADHOLD) {
        Thread.yield();
    }
    to.setBalance(to.getBalance + amount);
}
~~~
- 작업 도중 Thread.yield 메소드를 호출해주면 공유된 데이터를 사용할 때 적절한 동기화 방법을 사용하지 않은 경우 특정한 타이밍에 발생할 수 있는 버그가 노출되는 가능성을 높일 수 있음

</br>

## 12.2 성능 테스트
- 성능 테스트는 특정한 사용 환경 시나리오를 정해두고, 해당 시나리오를 통과하는데 얼마만큼의 시간이 걸리는지를 측정하고자 하는데 목적이 있음
- 성능 테스트의 두번째 목적은 성능과 관련된 스레드의 개수, 버퍼의 크기 등과 같은 각종 수치를 뽑아내고자 함

</br>

### 12.2.1 PutTakeTest에 시간 측정 부분 추가
- 단일 연산을 굉장히 많이 실행하여 전체 실행 시간을 구한 다음 실행했던 연산의 개수로 나누고, 단일 연산을 실행하는 데 걸린 평균 시간을 찾는 방법이 정확함
~~~java
public class BarrierTimer implements Runnable{
    private boolean started;
    private long startTime, endTime;
    
    @Override
    public synchronized void run() {
        long t = System.nanoTime();
        if(!started) {
            started = true;
            startTime = t;
        } else 
            endTime = t;
    }
    
    public synchronized void clear() {
        started = false;
    }
    
    public synchronized long getTime() {
        return endTime - startTime;
    }
}
~~~
- 배리어(barrier)가 적용되는 부분에서 시작 시간과 종료 시간을 측정할 수 있도록 기존 클래스를 확장할 수 있음
- 시간 측정 기능을 갖고 있는 배리어 액션을 사용하도록 하려면 CyclicBarrier를 초기화 하는 부분에 원하는 배리어 액션을 지정함
~~~java
this.timer = new BarrierTimer();
this.barrier = new CyclicBarrier(npairs * 2 + 1, timer);
~~~

</br>

~~~java
public void test() {
    try {
        timer.clear();
        for(int i = 0; i < nPairs; i++) {
            pool.execute(new Producer());
            pool.execute(new Consumer());
        }
        barrier.await();
        barrier.await();
        long nsPerItem = timer.getTime() / (nPairs * (long) nTrials);
        System.out.println("Throughput : " + nsPerItem + " ns/item");
        assertEquals(putSum.get(), takeSum.get());
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
~~~
- 베리어 기반의 타이머를 사용하도록 변경한 메소드

</br>

~~~java
public static void main(String[] args) throws Exception {
    int tpt = 100000; // 스레드별 실행 횟수
    for(int cap = 1; cap <= 1000; cap *= 10) {
        System.out.println("Capacity : " + cap);
        for(int pairs = 1;  pairs <= 128; pairs *= 2) {
            TimedPutTakeTest t = new TimedPutTakeTest(cap, pairs, tpt);
            System.out.print("Pairs: "+ pairs + "\t");
            t.test();
            System.out.print("\t");
            Thread.sleep(1000);
            t.test();
            System.out.println();
            Thread.sleep(1000);
        }
    }
    pool.shutdown();
}  
~~~
- 버퍼의 크기를 얼마로 제한해야 최고 성능을 알아내는지 알아보기 위해, 여러가지 인자에 다양한 값을 설정하면서 테스트 프로그램을 실행해봐야 함
- 위와 같은 테스트 실행 프로그램을 사용하면 편리함

![timedPutTakeTest01](/img/timedPutTakeTest01.png)     
  
- CPU가 4개 장착된 하드웨어서 버퍼의 크기를 1, 10, 100, 1000으로 변경하면서 실행한 결과
- 버퍼 크기가 1인 경우, 각 스레드가 대기 상태에 들어가고 나오면서 아주 적은 양의 작업밖에 할 수 없기 때문에 성능이 떨어지는 것은 당연
- 10을 넘는 크기를 지정하면 버퍼의 크기에 비해 성능이 향상되는 정도가 떨어지는 것을 볼 수 있음
- 스레드가 많이 실행되고 있더라도, 실제 작업을 하는 양은 많지 않고 스레드가 대기 상대에 들어갔다 나왔다하는 동기화를 맞추느라 CPU 용량의 대부분을 사용하기 때문
- 프로듀서-컨슈머 패턴으로 움직이는 애플리케이션을 생각한다면 항목을 생성하고 사용하는 과정에서 무시할 수 없을 만큼 상당한 양의 작업이 이뤄질 것
	- 스레드를 너무 많이 추가했다가는 그 여파를 눈으로 확인할 수 있게 됨..


### 12.2.2 다양한 알고리즘 비교
- BoundedBuffer 클래스의 속도가 떨어지는 가장 큰 이유는 put,take 연산 양쪽에서 모두 스레드 경쟁을 유발할 수 있는 연산을 사용하기 때문
	- 세마포어를 확보하거나 락을 확보하고 세마포어를 다시 해제하는 등의 연산
![queue-perform](/img/queue-perform.png)    
  
- 듀얼 하이퍼스레드 CPU가 장착된 하드웨어에서 버퍼 크기가 256인 클래스 3개를 비교 실행한 결과
- 연결 큐는 새로운 항목을 추가할 때마다 버퍼 항목을 메모리에 새로 할당 받아야하기 때문에 배열 기간의 큐보다 더 많은 일을 해야함
- 연결 리스트 기반의 큐는 put,take 연산에 대해 배열 기반의 큐보다 병렬 처리 환경에서 훨씬 안정적으로 동작함
	- 큐의 처음과 끝 부분에 서로 다른 스레드가 동시에 접근해 사용할 수 있기 때문
- 메모리 할당 작업은 일반적으로 스레드 내부에 한정돼 있기 떄문에 메모리를 할당한다 해도 스레드 간의 경쟁을 줄일 수 있는 알고리즘의 확장성이 더 높을 수 밖에 없음


### 12.2.3 응답성 측정
- 단일 작업 처리 시간을 측정할 때는 보통 측정 값의 분산(variance)을 중요한 수치로 생각함
	- 간혹 평균 처리 시간은 길지만 처리 시간의 분산이 작은 값을 유지하는 일이 더 중요할 수 있기 때문
- '예측성' 역시 중요한 성능 지표 가운데 하나임을 알아야 함
- 처리 시간에 대한 분산을 구해보면 "100밀리초 안에 작업을 끝내는 비율이 몇 % 정도인가?"와 같은 서비스 품질에 대한 수치를 결과로 제시할 수 있음
- 서비스 시간에 대한 분산을 시각적으로 표현할 수 있는 가장 효과적인 방법은 작업을 처리하는데 걸린 시간을 히스토그램으로 그려보는 방법
- 작업 처리 시간을 모두 더할 뿐 아니라 각 처리 시간을 목록으로 관리하고 있어야 함

![timedPutTakeTest02](/img/timedPutTakeTest02.png)    
  
- 버퍼의 크기를 1000으로 지정하고 256개의 병렬 작업이 각각 1000개의 항목을 버퍼에 넣음
	- 한쪽은 공정한 세마포어를 사용(회색)
	- 다른쪽은 불공정 세마포어를 사용(흰색)
- 불공정 세마포어를 사용한 결과는 최소 시간과 최대 시간의 차이가 80배가 넘음
- 최소 시간과 최대 시간의 차이를 줄이기 위해 동기화 코드에 공정성을 높이면 됨
- BoundedBuffer의 경우 세마포어를 생성할 때 공정한 모드로 초기화시켜 공정성을 높일 수 있음
- 동기화 부분에 공정성을 높이면 처리시간의 분산 값을 줄여주는 효과가 있지만, 처리 속도가 크게 떨어지는 역효과가 나타남

![timedPutTakeTest03](/img/timedPutTakeTest03.png)    
  
- 공정함의 문제가 평균 실행 시간을 크게 늦추거나 실행시간의 분산을 훨씬 낮게 바꿔주지 못한다는 사실이 나타남
- 스레드가 아주 빡빡한 동기화 요구사항 때문에 계속해서 대기 상태에 들어가는 상황이 아니라면 불공정한 세마포어를 사용해 처리 속도를 크게 높일 수 있고, 반대로 공정한 세마포어를 사용해 처리 시간의 분산을 낮출 수 있음
- 세마포어를 사용할 때에는 항상 어느 방법을 사용할 것인지 결정해야만 함

</br>

## 12.3 성능 측정의 함정 피하기
- 성능을 올바르게 나타내지 못하고 잘못된 수치를 뽑아내는 잘못된 코딩 방법으로 프로그램을 작성하지 않도록 주의해야 함

</br>

### 12.3.1 가비지 컬렉션



