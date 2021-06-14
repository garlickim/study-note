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
- 