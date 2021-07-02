# 동기화 클래스 구현
- FutureTask, Semaphore, BlockingQueue 등은 상태 기반 선행 조건을 가진 클래스
- 상태 의존적인 클래스를 새로 구현하는 가장 간단한 방법은 이미 만들어져 있는 상태 의존적인 클래스를 활용하여 필요한 기능을 구현하는 것

## 14.1 상태 종속성 관리
- 순차적으로 실행되는 프로그램은 원하는 상태를 만족시키지 못하는 부분이 있다면 반드시 오류가 발생하게 됨
- 병렬 프로그래밍에서는 선행조건이 만족하지 않았을 때 오류가 발생하는 문제에서 비켜날 수도 있지만, 비켜가는 일보다는 선행 조건을 만족할 때까지 대기하는 경우가 많아짐
- 상태 종속적인 기능을 구현할 때 선행 조건이 만족할 때까지 작업을 멈추고 대기하도록 하면 조건이 맞지 않았을 때 프로그램이 멈춰버리는 방법보다 훨씬 간편하고 오류도 적게 발생함
- 원하는 상태에 다다를 때까지 폴링하고 잠깐 기다리고 다시 폴링하고 다시 잠깐 기다리는 것보다 조건 큐를 사용하면 많은 이득을 볼 수 있음
~~~java
void blockingAction() throws InterruptedException {
	상태 변수에 대한 락 확보
	while( 선행 조건이 만족하지 않음 ){
		확보했던 락을 풀어줌
		선행 조건이 만족할만한 시간만큼 대기
		인터럽트에 걸리거나 타임아웃이 걸리면 멈춤
		락을 다시 확보
	}
	작업 실행
	락 해제
}
~~~
- 상태 종속적인 블로킹 작업의 모양
- 선행조건에 해당하는 클래스 내부 상태 변수는 값을 확인하는 동안에도 적절한 락으로 반드시 동기화해야 올바른 값을 확인할 수 있음
- 선행조건이 만족하지 않았을 때 락을 풀어주지 않는다면, 다른 스레드에서 상태 변수 값을 변경할 수 없기 때문에 선행 조건을 영원히 만족시키지 못함
- 프로듀서-컨슈머 패턴으로 구현된 애플리케이션에서는 ArrayBlockingQueue와 같이 크기가 제한된 큐를 많이 사용함
	- 크기가 제한된 큐는 put(),take()를 제공하며, 각각 선행조건을 가지고 있음(버퍼가 비어있으면 take 불가, 버퍼가 가득차있다면 put 불가)
- 상태 종속적인 메소드에서 선행 조건과 관련된 오류가 발생하면 예외를 발생시키거나 오류 값을 리턴하기도 하고, 선행조건이 원하는 상태에 도달할 때까지 대기하기도 함
~~~java
@ThreadSafe
public abstract class BaseBoundedBuffer<V> {
    @GuardedBy("this") private final V[] buf;
    @GuardedBy("this") private int tail;
    @GuardedBy("this") private int head;
    @GuardedBy("this") private int count;

    protected BaseBoundedBuffer(int capacity) {
        this.buf = (V[]) new Object[capacity];
    }

    protected synchronized final void doPut(V v) {
        buf[tail] = v;
        if (++tail == buf.length)
            tail = 0;
        ++count;
    }

    protected synchronized final V doTake() {
        V v = buf[head];
        buf[head] = null;
        if(++head == buf.length)
            head = 0;
        --count;
        return v;
    }
    
    public synchronized final boolean isFull() {
        return count == buf.length;
    }
    
    public synchronized final boolean isEmpty() {
        return count == 0;
    }
}
~~~
- 크기가 제한된 버퍼 기반의 클래스

</br>

### 14.1.1 예제: 선행 조건 오류를 호출자에게 그대로 전달
~~~java
public class GrumpyBoundedBuffer<V> extends BaseBoundedBuffer {
    public GrumpyBoundedBuffer(int size) {
        super(size);
    }
    
    public synchronized void put(V v) throws BufferFullException {
        if(isFull()) {
            throw new BufferFullException();
        }
        doPut(v);
    }
    
    public synchronized V take() throws BufferEmptyException {
        if(isEmpty())
            throw new BufferEmptyException();
        return doTake();
    }
}
~~~
- put(), take()는 check-then-act 구조로 구현했기 때문에 synchronized 키워드를 적용해 버퍼 내부의 상태 변수에 동기화된 상태로 접근하게 되어있음
- 선행 조건이 맞지 않으면 그냥 멈춰버리는 클래스의 예
- GrumpyBoundedBuffer을 호출하는 쪽은 put(), take() 사용시 예회 상황을 매번 처리해줘야 함
~~~java
while (true) {
    try {
        V item = buffer.take();
        // 값을 사용한다
        break;
    } catch (BufferEmptyException e) {
        Thread.sleep(SLEEP_GRANUALRITY);
    }
}
~~~
- GrumpyBoundedBuffer 클래스의 take()를 호출하는 일반적인 구조의 예
- 원하는 상태가 아닐 때 오류 값을 리턴하는 방법도 존재함
	- 예외를 리턴하는 것보다 약간 나은 방법일 수는 있으나, 호출자가 오류를 맡아서 처리해야하는 원론적인 문제를 해결하지는 못함
- 호출자가 대기 시간 없이 take()를 즉시 다시 호출하는 방법으로 재시도를 구현하는 방법이 존재함 (스핀 대기 방법이라 함)
	- 버퍼의 상태가 원하는 값으로 얼른 돌아오지 않는다면 상당량의 CPU를 소모함
	- CPU를 덜 사용하도록 일정시간 동안 대기하게 한다면 버퍼의 상태가 원하는 값으로 돌아왔음에도 불구하고 계속해서 대기 상태에 빠져있는 '과다 대기' 문제가 발생하기도 함

### 14.1.2 예제: 폴링과 대기를 반복하는 세련되지 못한 대기 상태
~~~java
@ThreadSafe
public class SleepyBoundedBuffer<V> extends BaseBoundedBuffer {
    public SleepyBoundedBuffer(int size) { super(size); }
    
    public void put(V v) throws InterruptedException {
        while (true) {
            synchronized (this) {
                if(!isFull()) {
                    doPut(v);
                    return;
                }
            }
            Thread.sleep(SLEEP_GRANUALITY);
        }
    }
    
    public V take() throws InterruptedException {
        while (true) {
            synchronized (this) {
                if(!isEmpty())
                    return doTake();
            }
            Thread.sleep(SLEEP_GRANUALITY);
        }
    }
}
~~~
- '폴링하고 대기하는' 재시도 반복문을 put(), take() 내부에 내장시켜 외부의 호출 클래스가 매번 직접 재시도 반복문을 사용하지 않도록 불편함을 줄여주고자 함
- 잠시 대기하고 상태 조건을 확인하는 반복문을 계속해서 실행하다 조건이 적절해지면 반복문을 빠져나와 작업을 처리
- 잠자기 대기 상태에 들어가는 시간을 길게 잡거나 짧게 잡으면 응답 속도와 CPU 사용량 간의 트레이드 오프가 발생함
	- 대기 시간을 짧게 잡으면 응답성은 좋아지지만 CPU 사용량은 올라감

![many-waiting](/img/many-waiting.png)   
    
- 대기 시간에 따라 응답속도가 어떻게 변하는지를 보여줌
- 버퍼에 공간이 생긴 이후 스레드가 대기 상태에서 빠져나와 상태 조건을 확인하기까지 약간의 시간 차이가 발생하기도 한다는 점을 주의
- 메소드 내부에서 원하는 조건을 만족할 때까지 대기해야 한다면 작업을 취소할 수 있는 기능을 제공하는 편이 좋음

</br>

### 14.1.3 조건 큐 - 문제 해결사
- 조건 큐는 여러 스레드를 한 덩어리(대기 집합 wait set이라고 부름)로 묶어 특정 조건이 만족할 때까지 한꺼번에 대기할 수 있는 방법을 제공하기 때문에 '조건 큐'라는 이름으로 불림
- 데이터 값으로 일반적인 객체를 담아두는 보통의 큐와는 달리 조건 큐에는 특정 조건이 만족할 때까지 대기해야 하는 스레드가 값으로 들어감
- 모든 객체는 스스로를 조건 큐로 사용할 수 있으며, 모든 객체가 갖고 있는 wait, notify, notifyAll 메소드는 조건 큐의 암묵적인 API 라고 봐도 좋음
- 자바 객체의 암묵적인 락과 암묵적인 조건 큐는 서로 관련된 있는 부분이 존재함
	- 객체 내부 상태를 확인하기 전에는 조건이 만족할 때까지 대기할 수가 없고, 객체 내부 상태를 변경하지 못하는 한 조건 큐에서 대기하는 객체를 풀어줄 수 없기 때문에
- Object.wait 메소드는 현재 확보하고 있는 락을 자동으로 해제하면서 운영체제에게 현재 스레드를 멈춰달라고 요청하고, 따라서 다른 스레드가 락을 확보해 객체 내부의 상태를 변경할 수 있도록 해줌
	- 대기 상태에서 깨어나는 순간에는 해제했던 락을 다시 확보함
~~~java
@ThreadSafe
public class BoundedBuffer<V> extends BaseBoundedBuffer<V> {
    //조건 서술어 : not-full (!isFull())
    //조건 서술어 : not-empty(!isEmpty())
    
    public BoundedBuffer2(int size) {
        super(size);
    }
    
    //만족할때까지 대기 : not-full
    public synchronized void put(V v) throws InterruptedException {
        while (isFull()) {
            wait();
        }
        doPut(v);
        notifyAll();
    }
    
    //만족할대까지 대기 : not-empty
    public synchronized V take() throws InterruptedException {
        while (isEmpty()) {
            wait();
        }
        
        V v = doTake();
        notifyAll();
        return v;
    }
}
~~~
- wait(), notifyAll()을 사용해 크기가 제한된 버퍼를 구현함
- sleep()으로 대기 상태에 들어가던 메소드보다 구현하기 간편하고, 훨씬 효율적이며 응답성도 높음
- 조건 큐를 사용한다고 해서 폴링과 대기 상태를 반복하던 버전에서 할 수 없던 일을 할 수 있게 되는 경우는 없음
- 조건 큐를 사용하면 상태 종속성을 관리하거나 표현하는 데 있어서 훨씬 효율적이면서 간편한 방법이긴 함

</br>

## 14.2 조건 큐 활용
- 조건 큐를 사용하면 효율적이면서 응답 속도도 빠른 상태 종속적인 클래스를 구현할 수 있지만, 올바르지 않은 방법으로 사용할 가능성도 높음

### 14.2.1 조건 서술어
- 조건 큐를 올바르게 사용하기 위한 가장 핵심적인 요소는 해당 객체가 대기하게 될 조건 서술어를 명확하게 구분해내는 일
- 조건 서술어는 애초에 특정 기능이 상태 종속적이 되도록 만드는 선행 조건을 의미함
- 조건 서술어는 클래스 내부의 상태 변수에서 유추할 수 있는 표현식
- 조건 큐와 연결된 조건 서술어를 항상 문서로 남겨야 하며, 그 조건 서술어에 영향을 받는 메소드가 어느 것인지도 명시해야 함
- 조건부 대기와 관련된 락과 wait() 와 조건 서술어는 중요한 삼각 관계를 유지하고 있음
	- 조건 서술어는 상태 변수를 기반으로 하고 있으며, 상태 변수는 락으로 동기화되어 있어 조건 서술어를 만족하는지 확인하려면 반드시 락을 확보해야만 함
	- 락 객체와 조건 큐 객체(wait와 notify 메소드를 호출하는 대상 객체)는 반드시 동일한 객체여야 함
- wait 메소드를 호출하는 모든 경우에는 항상 조건 서술어가 연결돼 있음
- 특정 조건 서술어를 놓고 wait 메소드를 호출할 때, 호출자는 항상 해당하는 조건 큐에 대한 락을 이미 확보한 상태여야 함
- 확보한 락은 조건 서술어를 확인하는데 필요한 모든 상태 변수를 동기화하고 있어야 함

### 14.2.2 너무 일찍 깨어나기
- wait()를 호출하고 리턴됐다고 해서 반드시 해당 스레드가 대기하고 있던 조건 서술어를 만족한다는 것은 아님
	- 하나의 암묵적인 조건 큐를 두 개 이상의 조건 서술어를 대상으로 사용할 수도 있음
	- wait()은 누군가가 notify 해주지 않아도 리턴되는 경우도 존재
- 하나의 조건 큐에 여러 개의 조건 서술어를 연결해 사용하는 일은 굉장히 흔한 방법
	- BoundedBuffer 역시 하나의 조건 큐를 놓고 "버퍼에 값이 있어야 한다"와 "버퍼에 빈공간이 있다"는 두 개의 조건 서술어를 한번에 연결해 사용하고 있음
- wait()가 깨어나 리턴되고 나면 조건 서술어를 한번 더 확인해야하고, 조건 서술어를 만족하지 않으면 다시 wait()을 호출해 대기 상태에 들어가야 함
- 조건 서술어를 만족하지 않은 상태에서 wait()가 여러차례 리턴될 가능성도 있기 때문에 wait()를 반복문 안에 넣어 사용해야 함
	- 매번 반복할 때마다 조건 서술어를 확인해야 함
~~~java
void stateDependentMethod() throws InterruptedException {
    //조건 서술어는 반드시 락으로 동기화된 이후에 확인해야 한다.
    synchronized(lock) {
        while(!conditionPredicate())
            lock.wait();
        // 객체가 원하는 상태에 맞춰졌다.
    }
}
~~~
- 상태 종속적인 메소드의 표준적인 형태
- 조건부 wait 메소드(Object.wait 또는 Condition.wait)를 사용할 때에는 
	- 항상 조건 서술어를 명시해야 함
	- wait()를 호출하기 전에 조건 서술어를 확인하고, wait에서 리턴된 이후에도 조건 서술어를 확인해야 함
	- wait()는 항상 반복문 내부에서 호출해야 함
	- 조건 서술어를 확인하는데 관련된 모든 상태 변수는 해당 조건 큐의 락에 의해 동기화 돼 있어야 함
	- wait, notify, notifyAll 메소드를 호출할 때는 조건 큐에 해당하는 락을 확보하고 있어야 함
	- 조건 서술어를 확인한 이후 실제로 작업을 실행해 작업이 끝날 때까지는 락을 해제해서는 안됨

### 14.2.3 놓친 신호
- 특정 스레드가 이미 참(true)을 만족하는 조건을 놓고 조건 서술어를 제대로 확인하지 못해 대기 상태에 들어가는 상황을 놓친 신호라 함
- 놓친 신호 문제가 발생한 스레드는 이미 지나간 일에 대한 알림을 받으려 대기하게 됨
- 놓친 신호 현상이 발생하는 원인은 스레드에 대한 알림이 일시적이라는데 있음
- 놓친 신호는 프로그램을 작성할 때 앞에서 소개한 여러가지 주의 사항을 지키지 않아서 발생함
	- wait()를 호출하기 전에 조건 서술어를 확인하지 못하는 경우가 생기는 경우

### 14.2.4 알림
- 특정 조건을 놓고 wait()를 호출해 대기 상태에 들어간다면, 해당 조건을 만족하게 된 이후에 반드시 알림 메소드를 사용해 대기 상태에서 빠져나오도록 해야함
- 조건 큐 API에서 알림 기능을 제공하는 메소드에는 두가지가 있음
	- notify, notifyAll
- notify, notifyAll 어느 메소드를 호출하더라도 해당하는 조건 큐 객체에 대한 락을 확보한 상태에서만 호출할 수 있음	
- nofify()를 호출하면 JVM은 해당하는 조건 큐에서 대기 상태에 들어가 있는 다른 스레드 하나를 골라 대기 상태를 풀어줌
- notifyAll()을 호출하면 해당하는 조건 큐에서 대기 상태에 들어가 있는 모든 스레드를 풀어짐
- 단 한번만 알림 메시지를 전달하게 되면 앞서 '놓친 신호'와 유사한 문제가 생길 가능성이 높음
- notifyAll 대신 notify 메소드를 사용하려면 다음과 같은 조건에 해당하는 경우에만 사용하는 것이 좋음
	- **단일 조건에 따른 대기 상태에서 깨우는 경우**
		- 해당하는 조건 큐에 단 하나의 조건만 사용하고 있는 경우이고, 따라서 각 스레드는 wait 메소드에서 리턴될 때 동일한 방법으로 실행됨
	- **한 번에 하나씩 처리하는 경우**
		- 조건 변수에 대한 알림 메소드를 호출하면 하나의 스레드만 실행시킬 수 있는 경우
- 일반적인 경우에는 notifyAll을 사용하는 편이 나음
	- notify보다 덜 효율적일 수는 있으나 클래스를 제대로 동작시키려면 notifyAll을 사용하는 쪽이 더 쉬움
- 버퍼가 비어 있다가 값이 들어오거나 가득찬 상태에서 값을 뽑아내는 경우에만 대기 상태에서 빠져나올 수 있다는 점을 활용해 take, put 메소드가 대기 상태에서 빠져나올 수 있는 상태를 만들어주는 경우에만 알림메소드를 호출하도록 하면 보수적인 측면을 최적화 할 수 있음
	- 최적화 방법을 조건부 알림이라고 부름
~~~java
public synchronized void put(V v) throws InterruptedException {
    while(isFull()) 
        wait;
    boolean wasEmpty = isEmpty();
    doPut(v);
    if(wasEmpty)
        notifyAll();
}
~~~
- BoundedBuffer 클래스의 put 메소드에 조건부 알림 방법을 적용한 모습


### 14.2.5 예제: 게이트 클래스
~~~java
@ThreadSafe
public class ThreadGate {
    //조건 서술어 : opened-since(n) (isOpen || generation > n)
    @GuardedBy("this") private boolean isOpen;
    @GuardedBy("this") private int generation;
    
    public synchronized void close() {
        isOpen = false;
    }
    
    public synchronized void open() {
        ++generation;
        isOpen = true;
        notifyAll();
    }
    
    //만족할때까지 대기 : opened-since(generation on entry)
    public synchronized void await() throws InterruptedException {
        int arrivalGeneration = generation;
        while (!isOpen && arrivalGeneration == generation)
            wait();
    }
}
~~~
- 조건부 대기 기능을 활용하면 여러 번 닫을 수 있는 ThreadGate와 같은 클래스를 어렵지 않게 구현 가능
- 문이 열리는 시점에 N개의 스레드가 문이 열리기를 기다리고 있었다면 대기중이던 스레드 N개가 모두 대기 상태에서 빠져나오도록 해야함
- 문이 열린 이후 짧은 시간 후에 다시 닫히는 상황이 발생하면 await 메소드에서 단순하게 isOpen 메소드만을 기준으로 대기 상태에서 깨어나도록 하는 방법이 충분하지 않을 수 있음
- 대기 중이던 스레드가 알림을 받고는 대기 상태에서 깨어나 락을 확보하고 wait 메소드에서 리턴하고 나니 이미 isOpen 메소드의 값이 다시 '닫힘'으로 바뀔 수도 있음


### 14.2.6 하위 클래스 안전성 문제
- 조건부 알림 기능이나 단일 알림 기능을 사용하고 나면 해당 클래스의 하위 클래스를 구현할 때 상당히 문제가 복잡해지는 문제가 생길 수 있음
- 최소한 상태 기반으로 동작하면서 하위 클래스가 상속받을 가능성이 높은 클래스를 구현하려면 조건 큐와 락 객체 등을 하위 클래스에게 노출시켜 사용할 수 있도록 해야 하고, 조건과 동기화 정책 등을 문서로 남겨야 함
- 클래스를 상속받는 과정에서 발생할 수 있는 오류를 막을 수 있는 간단한 방법 가운데 하나는 클래스를 final로 선언해 상속 자체를 금지하거나 조건 큐, 락, 상태 변수 등을 하위 클래스에서 접근할 수 없도록 막아두는 방법이 존재함


### 14.2.7 조건 큐 캡슐화
- 일반적으로 조건 큐를 클래스 내부에 캡슐화해서 클래스 상속 구조의 외부에서는 해당 조건 큐를 사용할 수 없도록 막는게 좋음
	- 그렇지 않으면 클래스를 사용하는 외부 프로그램에서 조건 큐에 대한 대기와 알림 규칙을 추측한 상태에서 클래스를 처음 구현할 때 설계했던 방법과 다른 방법으로 호출할 가능성이 있음
- 조건 큐로 사용하는 객체를 클래스 내부에 캡슐화하라는 방법은 스레드 안전한 클래스를 구현할 때 적용되는 일반적인 디자인 패턴과 비교해 볼 때 일관적이지 않은 부분이 존재함
	- 객체 내부에서 객체 자체를 기준으로 한 암묵적인 락을 사용하는 경우


### 14.2.8 진입 규칙과 완료 규칙
- 상태를 기반으로 하는 모든 연산과 상태에 의존성을 갖고 있는 또 다른 상태를 변경하는 연산을 수행하는 경우에는 항상 진입 규칙과 완료 규칙을 정의하고 문서화해야 함
- 진입 규칙은 해당 연산의 조건을 뜻함
- 완료 규칙은 해당 연산으로 인해 변경됐을 모든 상태 값이 변경되는 시점에 다른 연산의 조건도 함께 변경됐을 가능성이 있으므로, 다른 연산의 조건도 함께 변경됐다면 해당 조건 큐에 알림 메시지를 보내야 한다는 규칙

</br>


## 14.3 명시적인 조건 객체
- 암묵적인 락을 일반화한 형태가 Lock클래스인 것처럼 암묵적인 조건 큐를 일반화한 형태는 바로 Condition 클래스
- 암묵적인 조건 큐에는 여러 가지 단점이 존재
	- 모든 암묵적인 락 하나는 조건 큐를 단 하나만 가질 수 있음
		- BoundedBuffer와 같은 클래스에서 여러 개의 스레드가 하나의 조건 큐를 놓고 여러 가지 조건을 기준으로 삼아 대기 상태에 들어갈 수 있음
	- 락과 관련해 가장 많이 사용되는 패턴은 조건 큐 객체를 스레드에게 노출시키도록 되어 있음
- 암묵적인 락이나 조건 큐 대신 Lock 클래스와 Condition 클래스를 활용하면 여러가지 종류의 조건을 사용하는 병렬 처리 객체를 구현하거나 조건 큐를 노출시키는 것에 대한 공부를 할 때 훨씬 유연하게 대처할 수 있음
- Condition 클래스 역시 내부적으로 하나의 Lock 클래스를 사용해 동기화를 맞춤
	- 인스턴스를 생성하려면 Lock.newCondtion 메소드를 호출
~~~java
public interface Condition {
    void await() throws InterruptedException;
	boolean await(long time, TimeUnit unit) throws InterruptedException;
	long awaitNanos(long nanosTimeout) throws InterruptedException;
    void awaitUninterruptibly();    
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
~~~
- Condtion 객체는 암묵적인 조건 큐와 달리 Lock 하나를 대상으로 필요한 만큼 몇 개 라도 만들 수 있음
- 공정한 Lock에서 생성된 Condition 객체는 Condition.await메소드에서 리턴될 때 정확하게 FIFO 순서를 따름
~~~java
@ThreadSafe
public class ConditionBoundedBuffer<T> {
    protected final Lock lock = new ReentrantLock();
    //조건 서술어 : notFull (count < items.length)
    private final Condition notFull = lock.newCondition();
    //조건 서술어 : notEmpty (count > 0)
    private final Condition notEmpty = lock.newCondition();
    
    @GuardedBy("lock")
    private final T[] items = (T[]) new Object[BUFFER_SIZE];
    @GuardedBy("lock")
    private int tail, head, count;
    
    //만족할떄까지 대기 : notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[tail] = x;
            if(++tail == items.length)
                tail = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    
    //만족할때까지 대기 : notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            T x = items[head];
            items[head] = null;
            if(++head == items.length)
                head = 0;
            -- count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
~~~
- 두 개의 Condition 객체를 사용해 notEmpty 조건을 처리
- 기존의 BoundedBuffer 클래스와 동작하는 모습은 동일하지만, 내부적으로 조건 큐를 사용하는 모습은 훨씬 읽기 좋게 작성되어 있음
- 하나의 암묵적인 조건 큐를 사용해 여러 개의 조건을 처리하느라 복잡해지는 것보다 조건별로 각각의 Condition 객체를 생성해 사용하면 클래스 구조를 분석하기도 쉬움
- Condition 객체를 활용하면 대기 조건들을 각각의 조건 큐로 나눠 대기하도록 할 수 있기 때문에 단일 알림 조건을 간단하게 만족시킬 수 있음
- signalAll 대신 그보다 더 효율적인 signal 메소드를 사용해 동일한 기능을 처리할 수 있으므로, 컨텍스트 스위치 횟수도 줄일 수 있고 버퍼의 기능이 동작하는 동안 각 스레드가 락을 확보하는 횟수 역시 줄일 수 있음
- 공정한 큐 관리 방법이나 하나의 락에서 여러 개의 조건 큐를 사용할 필요가 있으면 Condition 객체를 사용하고, 그럴 필요가 없다면 암묵적인 조건 큐를 사용하는 편이 더 나음

</br>

## 14.4 동기화 클래스의 내부 구조
- ReentratnLock과 Semaphore 모두 다른 여러 동기화 클래스와 같이 AbstractQueuedSynchronizer(AQS)를 상속받아 구현됨
	- AQS는 락이나 기타 동기화 클래스를 만들 수 있는 프레임워크 역할을 하며 AQS를 기반으로 하면 엄청나게 다양한 종류의 동기화 클래스를 간단하면서 효율적으로 구현할 수 있음
- ReentrantLock이나 Semaphore 클래스 뿐만 아니라 CountDownLatch, ReentrantReadWriteLock, SynchronousQueue, FutureTask 클래스도 AQS 기반으로 만들어져 있음
~~~java
// java.util.concurrent.Semaphore 클래스가 실제로 이렇게 구현되어 있지는 않다.
@ThreadSafe
public class SemaphoreOnLock {
    private final Lock lock = new ReentrantLock();
    // 조건 서술어 : permitsAvailable (permits > 0)
    private final Condition permitsAvailable = lock.newCondition();
    @GuardedBy("lock") private int permits;
    
    SemaphoreOnLock(int initialPermits) {
        lock.lock();
        try {
            permits = initialPermits;
        } finally {
            lock.unlock();
        }
    }
    
    // 만족할떄까지 대기 : permitsAvailable
    public void acquire() throws InterruptedException {
        lock.lock();
        try {
            while (permits <= 0) 
                permitsAvailable.await();
            --permits;
        } finally {
            lock.unlock();
        }
    }
    
    public void release() {
        lock.lock();
        try {
            ++permits;
            permitsAvailable.signal();
        } finally {
            lock.unlock();
        }
    }
}
~~~
- Lock을 사용해 구현한 카운팅 세마포어
- 대기 중인 스레드를 FIFO 큐에서 관리하는 기능 등을 AQS에서 처리해줌
- AQS를 기반으로 만들어진 개별 동기화 클래스는 스레드가 대기 상태에 들어가야 하는지 아니면 그냥 통과해야 하는지의 조건을 유연하게 정의할 수 있음
- 동기화 클래스를 AQS 기반으로 작성하면 여러가지 장점이 존재함
	- 동기화 클래스 하나를 기반으로 다른 동기화 클래스를 구현할 떄 여러 면에서 신경 써야하는 부분이 줄어듦
	- 대기 상태에 들어갈 수 있는 지점이 단한군데이기 때문에 컨텍스트 스위칭 부하를 줄일 수 있으며 전체적인 성능을 높일 수 있음

</br>

## 14.5 AbstractQueuedSynchronizer
- 개발자가 AQS를 직접 사용할 일은 거의 없을 것
- AQS 기반의 동기화 클래스가 담당하는 작업 가운데 가장 기본이 되는 연산은 바로 확보(acquire)와 해제(release)
- 확보 연산은 상태 기반으로 동작하며 항상 대기 상태에 들어갈 가능성이 있음
	- 호출자는 항상 원하는 상태에 다다를 때까지 대기할 수 있다는 가능성을 염두해야함
- 해제 연산은 대기 상태에 들어가지 않으며 대신 확보 연산에서 대기중인 스레드를 풀어주는 역할
- AQS는 동기화 클래스의 상태 변수를 관리하는 작업도 어느 정도 담당하는데 getState, setState, compareAndSetState 등의 메소드를 통해 단일 int 변수 기반의 상태 정보를 관리해줌
~~~java
boolean acquire() throws InterruptedException {
    while (확보 연산을 처리할 수 없는 상태이다) {
        if(확보 연산을 처리할 때까지 대기하길 원한다) {
            현재 스레드가 큐에 들어 있지 않다면 스레드를 큐에 넣는다
            대기 상태에 들어간다
        } else 
            return 실패;
    }
    
    상황에 따라 동기화 상태 업데이트
    스레드가 큐에 들어 있었다면 큐에서 제거한다
    return 성공;
}

void release() {
    동기화 상태 업데이트
    if(업데이트된 상태에서 대기 중인 스레드를 풀어줄 수 있다)
        큐에 쌓여 있는 하나 이상의 스레드를 풀어준다
}
~~~
- AQS 내부의 확보와 해제 연산의 형태
- AQS를 구현한 동기화 클래스에 따라 다르지만 확보 연산은 ReentrantLock에서와 같이 배타적으로 동작할 수도 있고 Semaphore나 CoundDownLatch 클래스에서와 같이 배타적이지 않을 수도 있음
- 확보 연산은 두가지 부분으로 나눠 볼 수 있음
	- 동기화 클래스에서 확보 연산을 허용할 수 있는 상태인지 확인하는 부분
		- 허용할 수 있는 상태면 해당 스레드는 작업을 계속 진행
		- 허용할 수 없는 상태면 확보 연산에서 대기 상태에 들어가거나 실패
	- 동기화 클래스 내부의 상태를 업데이트 하는 부분
		- 특정 스레드 하나가 동기화 클래스의 확보 연산을 호출하면 다른 스레드가 해당 동기화 클래스의 확보 연산을 호출했을 때 성공할지의 여부가 달라질 수 있음
 		- 스레드 하나가 래치의 확보 연산을 호출했다는 것으로는 다른 스레드가 해당 래치의 확보 연산을 호출하는 결과에 영향을 주지 못하므로, 래치에 대한 확보 연산은 그 내부의 상태를 변경하지 않음
- 배타적인 확보 기능을 제공하는 동기화 클래스는 tryAcquire, tryRelease, isHeldExclusivley 등의 메소드를 지원해야 함
- 배타적이지 않은 확보 기능을 지원하는 클래스는 tryAcquireShared, tryReleaseShared 메소드를 제공해야 함
- AQS에 들어있는 acquire, acquireShared, release, releaseShared 메소드는 해당 연산을 실행할 수 있는지를 확인할 때 상속받은 클래스에 들어있는 메소드 가운데 이름 앞에 try가 붙은 메소드를 호출함
- AQS 에서는 조건 큐 기능을 지원하는 락을 간단하게 구현할 수 있도록 동기화 클래스와 연동된 조건 변수를 생성하는 방법을 제공함


### 14.5.1 간단한 래치
~~~java
@ThreadSafe
public class OneShotLatch {
	private final Sync sync = new Sync();

	public void signal() { sync.releaseShared(0); }

	public void await() throws InterruptedException {
		sync.acuireSharedInterruptibly(0);
	}

	private class Sync extends AbstractQueuedSynchronizer {
		protected int tryAcquireShared(int ignored) {
			// 래치가 열려있는 상태(state==1)라면 성공, 아니면 실패
			return (getState() == 1) ? 1 : -1;
		}

		protected boolean tryReleaseShared(int ignored) {
			setState(1); // 래치가 열렸다
			return true; // 다른 스레드에서 확보 연산에 성공할 가능성이 있다
		}
	}
}
~~~
- AQS를 기반으로 구현한 바이너리 래치
- await 메소드를 호출하는 모든 스레드는 래치가 열린 상태로 넘어가기 전까지 모두 대기 상태에 들어감
- 누군가가 signal을 호출해 해제 연산을 실행하면 그동안 await에서 대기하던 스레드가 모두 해제되고 signal 호출 이후에 await를 호출하는 스레드는 대기 상태에 들어가지 않고 바로 실행됨
- OneShotLatch 클래스에서는 래치의 상태를 AQS 상태 변수로 표현하는데, 0이면 닫힌 상태이고 1이면 열린 상태
- OneShotLatch 는 AQS의 핵심 기능을 위임하는 형식으로 구현했는데, 대신 AQS를 직접 상속받는 방법으로 구현하는것도 가능함
- AQS를 직접 상속받아 구현했다면 위와 같은 예제의 단순함(메소드가 단지 2개라는 것)을 잃을 수밖에 없으며, AQS에 정의되어 있지만 사용하지 않는 메소드가 public으로 노출돼 있기 때문에 여러가지 비슷한 메소드에 혼동돼 잘못 사용할 위험이 높음
- java.util.concurrent 패키지에 들어있는 동기화 클래스는 AQS를 private 내부 클래스로 선언해 위임 기법을 사용하고 있음

</br>

## 14.6 java.util.concurrent 패키지의 동기화 클래스에서 AQS 활용 모습

### 14.6.1 ReentrantLock
- ReentrantLock은 배타적인 확보 연산만 제공하기 때문에 tryAcquire, tryRelease, isHeldExclusively와 같은 메소드만 구현하고 있음
~~~java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, 1)) {
        	owner = current;
            return true;
        }
    }
    else if (current == owner) {
        setState(c+1);
        return true;
    }
    return false;
}
~~~
- 공정하지 않은 형태로 동작하는 tryAcquire 메소드의 코드
- ReentrantLock에서는 동기화 상태 값을 확보한 락의 개수를 확인하는데 사용하고, owner 라는 변수를 통해 락을 가져간 스레드가 어느 스레드인지도 관리함
- tryAcquire 메소드에서는 락을 확보하려는 시도가 재진입 시도인지 아니면 최초로 락을 확보하려는 것인지 구분하기 위한 용도로 owner 변수의 내용을 사용함
- 스레드에서 락을 확보하려고 하면 tryAcquire 메소드는 먼저 락의 상태를 확인함
	- 락이 풀려있는 상태라면 락을 확보했다는 사실을 알릴 수 있도록 상태 값을 업데이트해봄
- 락의 상태를 확인햇는데 이미 확보된 상태라고 판단되면, 락을 확보하고 있는 스레드가 현재 스레드인지를 확인하고 만약 그렇다면 락 확보 개수를 증가시킴
- 락을 확보하고 있는 스레드가 현재 스레드가 아니라면 확보 시도가 실패한 것으로 처리함


### 14.6.2 Semaphore와 CountDownLatch
- Semaphore는 AQS의 동기화 상태를 사용해 현재 남아 있는 퍼밋의 개수를 관리함
~~~java
protected int tryAcquireShared(int acquires) {
    while(true) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 || compareAndSetState(available, remaining))
            return remaining;
    }
}

protected final boolean tryReleaseShared(int releases) {
    while(true) {
        int p = getState();
        if (compareAndSetState(p, p + releases))
            return true;
    }
}
~~~
- tryAcquireShared 메소드는 현재 남아 있는 퍼밋의 개수를 알아내고, 남아 있는 퍼밋의 개수가 모자란다면 확보에 실패했다는 결과를 리턴함
	- 반대로 충분한 개수의 퍼밋이 남아 있었다면 compareAndSetState 메소드를 사용해 단일 연산으로 퍼밋의 개수를 필요한 만큼 줄임
	- 리턴되는 결과 값에는 성공여부와 함께 다른 스레드에서 실행하던 확보 연산을 처리할 수 있을지의 여부도 포함돼 있는데, 그렇다면 다른 스레드 역시 대기 상태에서 풀려날 수 있음
- tryReleaseShared 메소드는 퍼밋의 개수를 증가시키며, 따라서 현재 대기 상태에 들어가 있는 스레드를 풀어줄 가능성도 있고, 성공할 때까지 상태 값 변경 연산을 재시도함
	- 리턴 결과를 보면 해제 연산에 따라 다른 스레드가 대기 상태에서 풀려났을 가능성 여부를 알 수 있음
- CountDownLatch 클래스도 동기화 상태 값을 현재 개수로 사용하는 세마포어와 비슷한 형태로 AQS를 활용함


### 14.6.3 FutureTask
- FutureTask.get 메소드는 래치 클래스와 굉장히 비슷한 기능을 갖고 있음
	- 특정 이벤트가 발생하면 해당 스레드가 계속 진행할 수 있고, 아니면 원하는 이벤트가 발생할 때까지 스레드가 대기 상태에 들어감
- FutureTask는 작업의 실행 상황, 즉 실행 중이거나 완료됐거나 취소되는 등의 상황을 관리하는데 AQS 내부의 동기화 상태를 활용함
- 작업을 끝나면서 만들어낸 결과 값이나 작업에서 오류가 발생했을 때 해당하는 예외 객체를 담아둘 수 있는 추가적인 상태 변수도 갖고 있음
- 실제 작업을 처리하고 있는 스레드에 대한 참조도 갖고 있으며, 그래야만 인터럽트 요청이 들어 왔을 때 해당 스레드에 인터럽트를 걸 수 있음


### 14.6.4 ReentrantReadWriteLock
- AQS 기반으로 구현된 ReentrantReadWriteLock은 AQS 하위 클래스 하나로 읽기 작업과 쓰기 작업을 모두 담당함
- ReentrantReadWriteLock은 상태 변수의 32개 비트 가운데 16비트는 쓰기 락에 대한 개수를 관리하고 나머지는 읽기 락의 개수를 관리함
- 읽기 락에 대한 기능은 독점적이지 않은 확보와 해제 연산으로 구현돼 있고, 쓰기 락에 대한 기능은 독점적인 확보와 해제 연산을 사용함
- 내부적으로 AQS를 상속받은 클래스는 대기 중인 스레드의 큐를 관리하고, 스레드가 독점적인 연산을 요청했는지 아니면 독점적이지 않은 연산을 요청했는지도 관리함
- ReentrantReadWriteLock은 락에 여유가 생겼을 때 
	- 대기 큐의 맨 앞에 들어있는 스레드가 쓰기 락을 요청한 상태였다면 해당 스레드가 락을 독점적으로 가져감
	- 맨 앞에 있는 스레드가 읽기 락을 요청한 상태였다면 쓰기 락을 요청한 다음 스레드가 나타나기 전까지 읽기 락을 요청하는 모든 스레드가 독점적이지 않은 락을 가져감

</br>

## 요약
- 메소드 가운데 하나라도 상태 값에 따라 대기 상태에 들어갈 가능성이 있는 클래스를 작성해야 할 때 가장 좋은 방법은 Semaphore, BlockingQueue, CountDownLatch 등을 활용해 구현하는 방법
- 제공되는 동기화에도 적절한 기능이 없다면, 암묵적인 조건 큐나 명시적인 Condition클래스 또는 AQS 클래스등을 활용해 직접 원하는 기능의 동기화 클래스를 작성할 수 있음
- 상태 의존성을 관리하는 작업은 상태의 일관성을 유지하는 방법과 맞물려있기 때문에 암묵적인 조건 큐 역시 암묵적인 락과 굉장히 밀접하게 관련돼 있음
- 명시적인 조건 큐인 Condition 클래스도 명시적인 Lock 클래스와 밀접하게 관련돼 있으며, 암묵적인 버전의 조건 큐나 락보다 훨씬 다양한 기능을 제공함
	- 락 하나에서 다수의 대기 큐를 활용하거나 대기 상태에서 인터럽트에 어떻게 반응하는지를 지정하는 기능
	- 스레드 대기 큐의 관리 방법에 대한 공정성 여부를 지정하는 기능
	- 대기 상태에서 머무르는 시간을 제한할 수 있는 기능