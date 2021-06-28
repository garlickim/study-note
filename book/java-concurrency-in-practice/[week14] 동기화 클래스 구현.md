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
