# 중단 및 종료
- 자바에는 스레드가 작업을 실행하고 있을 때 강제로 멈추도록 하는 방법이 없음
- 인터럽트 방법을 사용하여 특정 스레드에게 작업을 멈춰 달라고 요청하는 형태로 사용 가능
- 작업이나 서비스를 실행하는 부분의 코드를 작성할 때, 멈춰달라는 요청을 받으면 진행 중이던 작업을 모두 정리한 다음 종료하도록 해야함
- 작업을 진행하던 스레드가 직접 마무리하는 것이 가장 적절한 방법
- 작업이나 스레드가 스스로 작업을 멈출 수 있도록 구성해두면 시스템의 유연성이 크게 늘어날 것임

</br>

## 7.1 작업 중단
- 실행 중인 작업을 취소하고자 하는 요구사항은 여러가지 경우에 나타남
	- 사용자가 취소하기를 요청한 경우
	- 시간이 제한된 작업
	- 애플리케이션 이벤트
	- 오류
	- 종료
- 특정 작업을 임의로 종료시킬 수 있는 방법이 없음
	- 작업을 실행하는 스레드와 작업을 취소했으면 한다고 요청하는 스레드가 함께 작업을 멈추는 협력적인 방법을 사용해야 함
- 협력적인 방법 가운데 가장 기본적인 형태는 '취소 요청이 들어왔다'는 플래그를 설정하고, 실행 중인 작업은 해당 플래그를 주기적으로 확인하는 방법
~~~java
@ThreadSafe
public class PrimeGenerator implements Runnable {
    @GuardedBy("this")
    private final List<BigInteger> primes = new ArrayList<BigInteger>();
    private volatile boolean cancelled;

    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }

    public void cancel() {
        cancelled = true;
    }

    public synchronized List<BigInteger> get() {
        return new ArrayList<BigInteger>(primes);
    }
}
~~~

</br>

~~~java
List<BigInteger> aSecondOfPrime() throws InterruptedException {
    PrimeGenerator generator = new PrimeGenerator();
    new Thread(generator).start();
    try {
        SECONDS.sleep(1);
    } finally {
        generator.cancel();
    }
    return generator.get();
}
~~~
- 취소를 요청하는 시점 이후에 run 메소드 내부의 반복문이 cancelled 플래그를 확인할 때까지 최소한의 시간이 흘러야하기 때문에 소수를 계산하는 작업이 정확하게 1초 후에는 멈춰있으리라는 보장은 없음
- sleep 메소드를 호출해 대기하던 도중 인터럽트가 걸려도 finally에서 cancel을 호출하기 때문에 계산작업은 반드시 멈출 수 있음
- 작업을 쉰게 취소시킬 수 있도록 만들려면 작업을 취소하려 할 때 '어떻게', '언제', '어떤 일'을 해야 하는지, 취소 정책(cancellation policy)을 명확히 정의해야 함
	- 외부 프로그램에서 작업을 취소하려 할 때 어떤 방법으로 취소 요청을 보낼 수 있는지, 작업 내부에서 취소 요청이 들어왔는지를 언제 확인하는지, 취소 요청이 들어오면 실행 중이던 작업이 어떤 형태로 동작하는지 등에 대한 정보를 제공해야 안전하게 사용할 수 있음

</br>

### 7.1.1 인터럽트
- 기본적인 취소 정책(취소 플래그로 판단)을 사용하는 작업 내부에서 BlockingQueue.put 과 같은 블로킹 메소드를 호출하는 부분이 있다면 큰 문제가 발생할 수 있음
- 작업 내부에서 취소 요청이 들어 왔는지를 확인하지 못하는 경우도 생길 수 있으며, 작업이 영원히 멈추지 않을 수도 있음
~~~java
class BrokenPrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;
    private volatile boolean cancelled = false;

    BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!cancelled)
                queue.put(p = p.nextProbablePrime());
        } catch (InterruptedException consumed) {
        }
    }

    public void cancel() {
        cancelled = true;
    }
}

void consumePrimes() throws InterruptedException {
    BlockingQueue<BigInteger> primes = ...;
    BrokenPrimeProducer producer = new BrokenPrimeProducer(primes);
    producer.start();
    try {
        while (needMorePrimes())
            consume(primes.take());
    } finally {
        producer.cancel();
    }
}
~~~
- 컨슈머가 가져가는 것보다 프로듀서가 소수를 찾아내는 속도가 더 빠르다면 큐는 곧 가득 찰 것이며 큐의 put 메소드는 블록될 것
- 부하가 걸린 컨슈머가 큐에 put 하려고 대기 중인 프로듀서의 작업을 취소시키려 하면?
 	- 프로듀서는 put 메소드에서 멈춰있고, put 메소드에서 멈춘 작업을 풀어줘야 할 컨슈머가 더이상 작업을 처리하지 못하기 때문에 cancelled 변수를 확인할 수 없음
- API나 언어 명세 어디를 보더라도 인터럽트가 작업을 취소하는 과정에 어떤 역할을 하는지에 대해 명시되어 있는 부분은 없음
- 실제 상황에서는 작업을 중단하고자 하는 부분이 아닌 다른 부분에 인터럽트를 사용한다면 오류가 발생하기 쉬울 수 밖에 없으며, 애플리케이션 규모가 커질수록 관리하기도 어려워짐

</br>

- 모든 스레드는 boolean 값으로 인터럽트 상태를 가지고 있음
	- 스레드에 인터럽트를 걸면 인터럽트 상태 변수 값이 true로 설정됨
~~~java
public class Thread {
	public void interrupt() { ... }
	public boolean isInterrupted() { ... }
	public static boolean interrupted() { ... }
	...
}
~~~
- Thread 클래스에는 해당 스레드에 인터럽트를 거는 메소드과, 인터럽트 걸린 상태를 확인하는 메소드가 존재
	- interrupt() : 해당하는 스레드에 인터럽트를 거는 역할
	- isInterrupted() : 해당 스레드에 인터럽트가 걸려있는지를 알려줌
	- interrupted() : 현재 스레드의 인터럽트 상태를 해제하고, 해제하기 이전의 값이 무엇이었는지를 알려줌 (인터럽트 상태를 해제할 수 있는 유일한 방법)
- Thread.sleep 이나 Object.wait 메소드와 같은 블로킹 메소드는 인터럽트 상태를 확인하고 있다가 인터럽트가 걸리면 즉시 리턴됨
	- 대기하던 중, 인터럽트가 걸리면 인터럽트 상태를 해제하면서 InterruptedException을 던짐
- 스레드가 블록되어 있지 않은 실행 상태에서 인터럽트가 걸린다면, 먼저 인터럽트 상태 변수가 설정되긴 하지만 인터럽트가 걸렸는지 확인하고, 인터럽트가 걸렸을 경우 그에 대응하는 일은 해당 스레드에서 알아서 해야함
- 특정 스레드의 interrupt 메소드를 호출한다 해도 해당 스레드가 처리하던 작업을 멈추지는 않음
	- 단지 해당 스레드에게 인터럽트 요청이 있었다는 메세지를 전달할 뿐임
- 인터럽트를 이해하고자 할 때 중요한 사항은 실행 중인 스레드에 실제적인 제한을 가해 멈추도록 하지 않는다는 것
- static interrupted 메소드는 현재 스레드의 인터럽트 상태를 초기화하기 때문에 사용할 때 상당한 주의를 기울여야 함
	- interrupted 호출 후 결과값으로 true가 리턴되면 인터럽트에 대응하는 어떤 작업을 진행해야 함
- 직접 작성한 스레드가 인터럽트 요청에 빠르게 반응하도록 하려면, 인터럽트를 사용해 작업을 취소 할 수 있도록 준비해야하고, 다양한 자바 라이브러리 클래스에서 제공하는 인터럽트 관련 기능을 충분히 지원해야 함
- 작업 취소 기능을 구현하고자 할 때에는 인터럽트가 가장 적절한 방법이라고 할 수 있음

</br>

~~~java
public class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;

    PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted())
                queue.put(p = p.nextProbablePrime());
        } catch (InterruptedException consumed) {
            /* 스레드를 종료한다 */
        }
    }

    public void cancel() {
        interrupt();
    }
}
~~~
- 인터럽트를 사용하면 작업 취소 기능을 훨씬 쉽고 간결하게 구현할 수 있음
- 인터럽트에 반응하는 블로킹 메소드를 상대적으로 적게 사용하고 있다면, 반복문의 조건 확인 부분에서 인터럽트 여부를 확인하는 방법으로 응답 속도를 개선할 수 있음

### 7.1.2 인터럽트 정책
- 인터럽트 처리 정책은 인터럽트 요청이 들어왔을 때, 해당 스레드가 인터럽트를 어떻게 처리해야 하는지에 대한 지짐
- 일반적으로 가장 범용적인 인터럽트 정책은 스레드 수준이나 서비스 수준에서 작업 중단 기능을 제공하는 것
- 작업(task)과 스레드(thread)가 인터럽트 상황에서 서로 어떻게 동작해야 하는지를 명확히 구분할 필요가 있음
- 작업은 그 작업을 소유하는 스레드에서 실행되지 않고, 스레드 풀과 같이 실행만 전담하는 스레드를 빌려 사용하게 됨
	- 실제로 작업을 실행하는 스레드를 갖고 있지 않은 프로그램은 작업을 실행하는 스레드의 인터럽트 상태를 그대로 유지해 스레드를 소유하는 프로그램이 인터럽트 상태에 직접 대응할 수 있도록 해야함
	- 대부분의 블로킹 메소드에서 인터럽트가 걸렸을 때 InterruptedException을 던지도록 되어 있는 이유가 바로 이것 때문
- 블로킹 메소드를 스스로의 스레드에서 실행하는 일은 전혀 없기 때문에 외부 작업이나 자바 내부의 라이브러리 메소드에서 동시에 적용할 수 있는 가장 적절한 인터럽트 정책, 즉 실행중에 최대한 빠릴 작업을 중단하고 자신을 호출한 스레드에서 전달받은 인터럽트 요청을 넘겨 인터럽트에 대응해 추가적인 작업을 할 수 있도록 배려하는 정책을 구현하고 있음
- 작업 실행 도중에 인터럽트가 걸렸다면 인터럽트 상황을 작업 중단이라는 의미로 해석할 수도 있고 아니면 인터럽트에 대응해 뭔가 작업을 처리할 수도 있는데, 작업을 실행중인 스레드의 인터럽트 상태는 그대로 유지시켜야 함
	- 일반적인 방법은 InterruptedException을 던지는 것인데, 그렇게 하지 못한다해도 `Thread.currentThread().interrupt();` 과 같은 코드를 실행해 스레드의 인터럽트 상태를 유지해야 함
- 스레드에는 해당 스레드를 소유하는 클래스에만 인터럽트를 걸어야함
	- 스레드를 소유하는 클래스는 shutdown과 같은 메소드에서 적절한 작업 중단 방법과 함께 스레드 인터럽트 정책을 확립해 내부적으로 적용하고 있기 때문
- 각 스레드는 각자의 인터럽트 정책을 갖고 있음
	- 해당 스레드에서 인터럽트 요청을 받았을 때 어떻게 동작할지를 정확하게 알고 있지 않는 경우에는 함부로 인터럽트를 걸어선 안됨

### 7.1.3 인터럽트에 대한 대응
- 인터럽트를 걸 수 있는 블로킹 메소드를 호출하는 경우에 InterruptedException이 발생했을 때 처리할 수 있는 실질적인 방법에는 대략 2가지가 존재
	- 발생한 예외를 호출 스택의 상위 메소드로 전달함
		- 이 방법을 사용하는 메소드 역시 인터럽트를 걸 수 있는 블로킹 메소드가 됨
	- 호출 스택의 상단에 위치한 메소드가 직접 처리할 수 있도록 인터럽트 상태를 유지함
~~~java
BlockingQueue<Task> queue;
...
public Task getNextTask() throws InterruptedException {
	return queue.take();
}
~~~
- InterruptedException을 상위 메소드로 전달할 수 없거나 전달하지 않고자 하는 상황이라면 인터럽트 요청이 들어왔다는 것을 유지할 수 있는 다른 방법을 찾아야 함
- 인터럽트 상태를 유지할 수 있는 가장 일반적인 방법은 interrupt 메소드를 다시 한번 호출하는 것
- catch 블록에서 InterruptedException을 잡내고 예외를 먹어버리는 일은 없어야 함
- 대부분의 프로그램 코드는 자신이 어느 스레드에서 동작할지 모르기 때문에 인터럽트 상태를 최대한 그대로 유지해야 함
- 스레드의 인터럽트 처리 정책을 정확하게 구현하는 작업만이 인터럽트요청을 삼겨버릴 수 있음
- 일반적인 용도로 작성된 작업이나 라이브러리 메소드는 인터럽트 요청을 그냥 삼켜버려서는 안됨
- 작업 중단 기능을 지원하지 않으면서 인터럽트를 걸 수 있는 블로킹 메소드를 호출하는 작업은 인터럽트가 걸렸을 때, 블로킹 메소드의 기능을 자동으로 재시도하도록 반복문 내부에서 블로킹 메소드를 호출하도록 구성하는 것이 좋음
	- InterruptedException이 발생하면 인터럽트 상태를 내부적으로 보관하고 있다가 메소드가 리턴되기 직전에 인터럽트 상태를 원래대로 복구하고 리턴되도록 해야함
~~~java
public Task getNextTask(BlockingQueue<Task> queue) {
	boolean interrupted = false;
	try {
		while(true) {
			try {
				return queue.take();
			} catch (InterruptedException e) {
				interrupted = true;
				// 그냥 넘어가고 재시도
			}
		}
	} finally {
		if(interrupted)
			Thread.currentThread().interrupt();
	}
}
~~~
- 인터럽트를 걸 수 있는 블로킹 메소드는 대부분 실행되자마자 가장 먼저 인터럽트 상태를 확인하며, 인터럽트가 걸린 상태라면 즉시 InterruptedException을 던지는 경우가 많기 때문에, 인터럽트 상태를 너무 일찍 지정하면 반복문이 무한반복에 빠질 수 있음
- 인터럽트가 걸릴 수 있는 블로킹 메소드를 전혀 사용하지 않는다해도 작업이 진행되는 과정 곳곳에서 현재 스레드의 인터럽트 상태를 확인해준다면 인터럽트에 대한 응답 속도를 높일 수 있음
- 작업 중단 기능은 인터럽트 상태뿐만 아니라 여러 가지 다른 상태와 관련이 있을 수 있음
- 인터럽트는 해당 스레드의 주의를 끄는 정도로만 사용하고, 인터럽트를 요청하는 스레드가 갖고 있는 다른 상태 값을 사용해 인터럽트 걸린 스레드가 어떻게 동작해야 하는지를 지정하는 경우도 있음

### 7.1.4 예제: 시간 지정 실행
- 작업을 실행할 때 실행 도중에 예외를 띄우는 경우가 있는지 확인하고자 하는 작업을 실행시키는 경우가 존재함
~~~java
private static final ScheduledExecutorService cancelExec = ...;

public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
	final Thread taskThread = Thread.currentThread();
	cancelExec.schedule(new Runnable() {
		public void run() { taskThread.interrupt(); }
	}, timeout, unit);
	r.run();
}
~~~
- 실제 작업을 호출하는 스레드 내부에서 실행시키고, 일정 시간이 지난 이후에 인터럽트를 걸도록 되어있는 작업 중단용 스레드를 따로 실행시킴
- 작업을 실행하는 도중에 확인되지 않은 예외가 발생하는 상황에 대응할 수 있음
- 스레드에 인터럽트를 걸 때, 대상 스레드의 인터럽트 정책을 알고 있어야 한다는 규칙을 어기고 있음
	- 호출하는 스레드의 인터럽트 정책은 알 수 없기 때문
- 작업 내부가 인터럽트에 제대로 반응하지 않도록 만들어져 있다면 timedRun 메소드는 작업이 끝날 때까지 리턴되지 않고 계속 실행될 것

</br>

~~~java
public static void timedRun(final Runnable r, 
							long timeout, TimeUnit unit) 
							throws InterruptedException {
	class RethrowableTask implements Runnable {
		private volatile Throwable t;
		public void run() {
			try { r.run(); }
			catch(Throwable t) { this.t = t; }
		}

		void rethrow() {
			if (t != null)
				throw launderThrowable(t);
		}
	}

	RethrowableTask task = new RethrowableTask();
	final Thread taskThread = new Thread(task);
	taskThread.start();
	cancelExec.schedule(new Runnable() {
		public void run() { taskThread.interrupted(); }
	}, timeout, unit);
	taskThread.join(unit.toMillis(timeout));
	task.rethrow();
}
~~~
- 작업을 실행하도록 생성한 스레드에는 적절한 실행 정책을 따로 정의할 수도 있고, 작업이 인터럽트에 응답하지 않는다 해도 시간이 제한된 메소드 자체는 호출한 메소드에게 리턴됨
- 작업 실행 과정에서 예외가 발생했다면 해당 예외를 상위 메소드에게 다시 던짐
- Throwable 클래스는 일단 저장해두고 호출 스레드와 작업 스레드가 서로 공유하는데, 예외를 작업 스레드에서 호출 스레드로 안전하게 공개할 수 있도록 volatile로 선언하고 있음
- timeRun 메소드가 리턴됐을 때 정상적으로 스레드가 종료된 것인지 join 메소드에서 타임아웃이 걸린 것인지 알 수 없다는 단점을 지님

### 7.1.5 Future를 사용해 작업 중단
- ExecutorService.submit 메소드를 실행하면 등록한 작업을 나타내는 Future 인스턴스를 리턴받음
- Future에는 cancel 메소드가 있고, mayInterruptIfRunning이라는 불린 값을 하나 넘겨 받으며, 취소 요청에 따른 작업 중단 시도가 성공인지를 알려주는 결과 값을 리턴받을 수 있음
- mayInterruptIfRunning 값으로 false를 넘겨주면 "아직 실행하지 않았다면 실행시키지 말아라" 라는 의미로 해석되며, 인터럽트에 대응하도록 만들어지지 않은 작업에서는 항상 false를 넘겨줘야함
- Executor에서 기본적으로 작업을 실행하기 위해 생성하는 스레드는 인터럽트가 걸렸을 때 작업을 중단할 수 있도록 하는 인터럽트 정책을 사용함
	- 기본 Executor에 작업을 등록하고 넘겨받은 Future에서는 cancel 메소드에 mayInterruptIfRunning 값으로 true를 넘겨 호출해도 문제 없음
- 작업을 중단하려 할 때에는 항상 스레드에 직접 인터럽트를 거는 대신 Future의 cancel 메소드를 사용해야 함
~~~java
public static void timedRun(Runnable r,
                            long timeout, TimeUnit unit) 
							throws InterruptedException {
    Future<?> task = taskExec.submit(r);
    try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        // finally 블록에서 작업이 중단될 것이다.
    } catch (ExecutionException e) {
        // 작업 내부에서 예외 상황 발생. 예외를 다시 던진다.
        throw launderThrowable(e.getCause());
    } finally {
        // 이미 종료됐다 하더라도 별다른 악영향은 없다.
        task.cancel(true); // 실행중이라면 인터럽트를 건다.
    }
}
~~~
- Future.get 메소드에서 InterruptedException이 발생하거나 TimeoutException이 발생했을 때, 만약 예외 상황이 발생한 작업의 결과는 필요가 없다고 한다면 해당 작업에 대해 Future.cancel 메소드를 호출해 작업을 중단하는 것이 좋음

### 7.1.6 인터럽트에 응답하지 않는 블로킹 작업 다루기 
- 여러 블로킹 메소드는 대부분 인터럽트가 발생하는 즉시 멈추면서 InterruptedException을 띄우도록 되어 있으며, 작업 중단 요청에 적절하게 대응하는 작업을 쉽게 구현할 수 있음
- 일부 상황에서는 인터럽트와 유사한 기법을 활용해 블로킹 메소드에서 대기 중인 스레드가 작업을 멈추도록 할 수 있긴 하지만, 해당 스레드가 대기 상태에 멈춰 있는 이유가 무엇인지 정확하게 이해해야 함
	- **java.io 패키지의 동기적 소켓 I/O**
		- 가장 대표적인 블로킹 I/O의 예는 소켓에서 데이터를 읽어오거나 데이터를 쓰는 부분
		- InputStream의 read(), OutputStream write() 인터럽트에 반응하지 않도록 되어있음
		- 해당 스트림이 연결된 소켓을 직접 닫으면 대기중이던 read(),write()가 중단되며 SocketException이 발생함
	- **java.nio 패키지의 동기적 I/O**
		- InterruptibleChannel에서 대기하고 있는 스레드에 인터럽트를 걸면 ClosedByInterruptException이 발생하면서 해당 채널이 닫힘
		- InterruptibleChannel을 닫으면 해당 채널로 작업을 실행하던 스레드에서 AsynchronousCloseException이 발생함
	- **Selector를 사용한 비동기적 I/O**
		- 스레드가 Selector 클래스의 select 메소드에서 대기 중인 경우, close 메소드를 호출하면 closeSelectorException을 발생시키면서 즉시 리턴함
	- **락 확보**
		- Lock 인터페이스를 구현한 락 클래스의 lockInterruptibly 메소드를 사용하면 락을 확보할 때까지 대기하면서 인터럽트에도 응답하도록 구현할 수 있음

~~~java
public class ReaderThread extends Thread {
    private final Socket socket;
    private final InputStream in;

    public ReaderThread(Socket socket) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }

    public void interrupt() {
        try {
            socket.close();
        } catch (IOException ignored) {
        } finally {
            super.interrupt();
        }
    }

    public void run() {
        try {
            byte[] buf = new byte[BUFSZ];
            while (true) {
                int count = in.read(buf);
                if (count < 0)
                    break;
                else if (count > 0)
                    processBuffer(buf, count);
            }
        } catch (IOException e) { /* 스레드를 종료한다 */
        }
    }
}
~~~
- 표준적이지 않은 방법으로 작업을 중단하는 기능을 속으로 감춰버리는 방법
- ReaderThread 클래스에 인터럽트를 걸었을 때 read 메소드에서 대기 중인 상태이거나 기타 인터럽트에 응답할 수 있는 블로킹 메소드에 멈춰 있을 때에도 작업을 중단시킬 수 있음

### 7.1.7 newTaskFor 메소드로 비표준적인 중단 방법 처리
- newTaskFor 메소드도 ExecutorService의 submit 메소드와 마찬가지로 드록된 작업을 나타내는 Future 객체를 리턴해주는데, 이전과는 다른 RunnableFuture 객체를 리턴함
- RunnableFuture는 Runnable과 Future 인터페이스를 모두 상속받음
	- java 6이후로는 FutureTask에서 RunnableFuture를 구현함
- Future.cancel 메소드를 오버라이드하면 작업 중단 과정을 원하는 대로 변경할 수 있음
	- 중단과정의 로깅, 통계 값 보관, etc..
~~~java
public interface CancellableTask <T> extends Callable<T> {
    void cancel();

    RunnableFuture<T> newTask();
}


@ThreadSafe
public class CancellingExecutor extends ThreadPoolExecutor {
    ...
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        if (callable instanceof CancellableTask)
            return ((CancellableTask<T>) callable).newTask();
        else
            return super.newTaskFor(callable);
    }
}

public abstract class SocketUsingTask <T> implements CancellableTask<T> {
    @GuardedBy("this") private Socket socket;

    protected synchronized void setSocket(Socket s) {
        socket = s;
    }

    public synchronized void cancel() {
        try {
            if (socket != null)
                socket.close();
        } catch (IOException ignored) {
        }
    }

    public RunnableFuture<T> newTask() {
        return new FutureTask<T>(this) {
            public boolean cancel(boolean mayInterruptIfRunning) {
                try {
                    SocketUsingTask.this.cancel();
                } finally {
                    return super.cancel(mayInterruptIfRunning);
                }
            }
        };
    }
}
~~~
- CancellingExecutor는 ThreadPoolExecutor 클래스를 상속받고, newTaskFor 메소드르 오버라이드해 CancellableTask에 추가된 기능을 활용할 수 있도록 함
- SocketUsingTask 클래스는 CancellableTask를 상속받으면서 Future.cancel 메소드에서 super.cancel 메소드를 호출하고 소켓도 닫도록 구현함
	- 실행 중인 작업에 중단 요청이 있을 때, 대응하는 속도를 크게 개선할 수 있음
- 작업을 중단하는 과정에서도 응답속도를 떨어뜨리지 않으면서 인터럽트에 대응하는 블로킹 메소드를 안전하게 호출할 수 있으며, 대기 상태에 들어갈 수있는 소켓 I/O 메소드와 같은 기능도 호출 할 수 있음

</br>

## 7.2 스레드 기반 서비스 중단
- 스레드를 직접 소유하고 있는 않는 한 해당 스레드에 인터럽트를 걸거나 우선 순위를 조정하는 등의 작업을 해서는 안됨
- 스레드를 소유하는 객체는 대부분 해당 스레드를 생성한 객체라고 볼 수 있음
- 스레드 풀에 들어 있는 모든 작업 스레드는 해당 하는 스레드 풀이 소유한다고 볼 수 있고, 개별 스레드에 인터럽트를 걸어야 하는 상황이 된다면 그 작업은 스레드를 소유한 스레드 풀에서 책임을 져야 함
- 애플리케이션은 개별 스레드를 직접 소유하고 있지 않기 떄문에 개별 스레드를 직접 조작하는 일이 없어야 함
	- 애플리케이션이 개별 스레드에 직접 액세스하는 대신 스레드 기반 서비스가 스레드의 시작부터 종료까지 모든 기능에 해당하는 메소드를 직접 제공해야 함
- 스레드 기반 서비스를 생성한 메소드보다 생성된 스레드 기반 서비스가 오래 실행될 수 있는 상황이라면, 스레드 기반 서비스에서는 항상 종료시키는 방법을 제공해야 함

</br>

### 7.2.1 예제: 로그 서비스
- 대부분의 서버 애플리케이션은 저마다 적절한 로그 서비스를 갖고 있음
- PrintWriter와 같은 스트림 기반 클래스는 스레드에 안전하기 때문에 println으로 필요한 내용을 출력하는 기능을 사용할 때 별다른 동기화 기법이 필요하지는 않음
~~~java
public class LogWriter {
    private final BlockingQueue<String> queue;
    private final LoggerThread logger;

    public LogWriter(Writer writer) {
        this.queue = new LinkedBlockingQueue<String>(CAPACITY);
        this.logger = new LoggerThread(writer);
    }

    public void start() {
        logger.start();
    }

    public void log(String msg) throws InterruptedException {
        queue.put(msg);
    }

    private class LoggerThread extends Thread {
        private final PrintWriter writer;
        ...
        public void run() {
            try {
                while (true)
                    writer.println(queue.take());
            } catch (InterruptedException ignored) {
            } finally {
                writer.close();
            }
        }
    }
}
~~~
- LogWriter 클래스에서는 로그 출력 기능을 독립적인 스레드로 구현하는 모습을 보여주고 있음
- 로그 메시지를 실제로 생성하는 스레드가 BlockingQueue를 사용해 메시지를 출력 전담 스레드에게 넘겨주며, 출력 전담 스레드는 큐에 쌓인 메시지를 가져다 화면에 출력함
	- 전형적인 다수의 프로듀서와 단일 컨슈머가 동작하는 패턴
- 로그 출력 전담 스레드에 문제가 생기면, 출력 스레드가 올바르게 동작하기 전까지 BlockingQueue가 막혀버리는 경우가 발생할 수 있음
- 애플리케이션을 종료하려 할 때 로그 출력 전담 스레드가 계속 실행되느라 JVM이 정상적으로 멈추지 않는 경우가 발생할 수도 있음
- 로그 출력 스레드에서 작업 도중 인터럽트가 걸리면, InterruptedException을 잡아 그냥 리턴되도록 구현하면 출력 스레드는 쉽게 종료시킬 수 있음
- 단순히 멈춰버리기만 하면 그동안 출력시키려고 큐에 쌓여있던 로그를 잃어버리게 되며, 로그 출력을 위해 log 메소드 호출시 큐가 가득 차서 메시지를 큐에 넣을 때까지 대시 상태에 들어가 있던 스레드는 영원히 대기 상태에 머물게 됨

</br>

~~~java
public void log(String msg) throws InterruptedException {
	if(!shutdownRequested)
		queue.put(msg);
	else
		throw new IllegalStateException("logger is shutdown");
}
~~~
- 클래스를 종료시키고자 할 때 LogWriter 내부에 "종료 요청이 들어왔다"는 플래그를 마련해두고, 플래스가 설정된 경우에는 더이상 로그 메시지를 큐에 넣을 수 없도록 하는 것
- 컨슈머 부분에서는 종료 요청이 들어왔을 때 큐에 있는 메시지를 모두 가져가 쌓여 있던 메시지를 모두 출력할 기회를 얻음
- 실행 도중 경쟁 조건에 들어 갈 수 있기 때문에 완전히 신뢰할 만한 방법은 아님
- 프로듀서는 로그 출력 서비스가 아직 종료되지 않았다고 판단하고 실제로 종료된 이후에도 로그 메시지를 큐에 쌓으려고 대기 상태에 들어갈 가능성이 존재
- LogWriter 클래스에 안정적인 종료 방법을 추가하려면 경쟁 조건에 들어가지 않는 방법을 찾아야 함
	- 로그 메시지를 추가하는 부분을 단일 연산으로 구현해야 한다는 의미
	- 단일 연산으로 종료됐는지를 확인하며 로그 메시지를 추가할 수 있는 권한이라고 볼 수 있는 카운터를 하나 증가시키는 방법을 사용

	~~~java
	public class LogService {
	    private final BlockingQueue<String> queue;
	    private final LoggerThread loggerThread;
	    private final PrintWriter writer;
	    @GuardedBy("this") private boolean isShutdown;
	    @GuardedBy("this") private int reservations;

	    public LogService(Writer writer) {
	        this.queue = new LinkedBlockingQueue<String>();
	        this.loggerThread = new LoggerThread();
	        this.writer = new PrintWriter(writer);
	    }

	    public void start() {
	        loggerThread.start();
	    }

	    public void stop() {
	        synchronized (this) {
	            isShutdown = true;
	        }
	        loggerThread.interrupt();
	    }

	    public void log(String msg) throws InterruptedException {
	        synchronized (this) {
	            if (isShutdown)
	                throw new IllegalStateException(...);
	            ++reservations;
	        }
	        queue.put(msg);
	    }

	    private class LoggerThread extends Thread {
	        public void run() {
	            try {
	                while (true) {
	                    try {
	                        synchronized (LogService.this) {
	                            if (isShutdown && reservations == 0)
	                                break;
	                        }
	                        String msg = queue.take();
	                        synchronized (LogService.this) {
	                            --reservations;
	                        }
	                        writer.println(msg);
	                    } catch (InterruptedException e) { /* 재시도 */
	                    }
	                }
	            } finally {
	                writer.close();
	            }
	        }
	    }
	}
	~~~


### 7.2.2 ExecutorService 종료
- ExecutorService를 종료하는 두가지 방법
	- shutdown 메소드를 사용해 안전하게 종료하는 방법
	- shutdownNow 메소드를 사용해 강제로 종료하는 방법
- shutdownNow를 사용해 강제로 종료시키고 나면 먼저 실행중인 모든 작업을 중단하도록 한 다음 아직 시작하지 않은 작업의 목록을 결과로 리턴해줌
- 강제로 종료하는 방법은 응답이 빠르지만 실행 도중 스레드에 인터럽트를 걸어야하기 때문에 중단 과정에서 여러가지 문제 발생 가능성이 존재
- 안전하게 종료하는 방법은 종료 속도가 느리지만 큐에 등록된 모든 작업을 처리할 때까지 스레드를 종료시키지 않고 놔두기 때문에 작업을 잃을 가능성이 없어 안전

~~~java
public class LogService {
	private final ExecutorService exec = newSingleThreadExecutor();
	...
	public void start() { }

	public void stop() throws InterruptedException {
		try {
			exec.shutdown();
			exec.awaitTermination(TIMEOUT, UNIT);
		} finally {
			writer.close();
		}
	}
	public void log(String msg) {
		try {
			exec.execute(new WriteTask(msg));
		} catch (RejectedExecutionException ignored) { }
	}
}
~~~
- ExecutorService를 직접 활용하는 대신 다른 클래스의 내부에 캡슐화해서 시작과 종료 등의 기능을 연결해 호출할 수 있음
- ExecutorService를 특정 클래스 내부에 캡슐화하면 애플리케이션에서 서비스와 스레드로 이어지는 소유 관계에 한단계를 더 추가하는 셈이고, 각 단계에 해당하는 클래스는 모두 자신이 소유한 서비스나 스레드의 시작과 종료에 관련된 기능을 관리함

### 7.2.3 독약
- 프로듀서-컨슈머 패턴으로 구성된 서비스를 종료시키도록 종용하는 또 다른 방법으로 독약(poison pill)이라고 불리는 방법이 있음
	- 특정 객체를 큐에 쌓도록 되어 있으며, 객체는 "이 객체를 받았다면, 종료해야 한다"는 의미를 갖고 있음
- FIFO 유형의 큐를 사용하는 경우에는 독약 객체를 사용했을 때 컨슈머가 쌓여 있던 모든 작업을 종료하고 독약 객체를 만나 종료되도록 할 수 있음
- FIFO 큐에서는 객체의 순서가 유지되기 떄문에 독약 객체보다 먼저 큐에 쌓인 객체는 항상 독약 객체보다 먼저 처리됨
- 프로듀서 측에서는 독약 객체를 한 번 큐에 넣고 나면 더 이상 다른 작업을 추가해서는 안됨
~~~java
public class IndexingService {
    private static final File POISON = new File("");
    private final IndexerThread consumer = new IndexerThread();
    private final CrawlerThread producer = new CrawlerThread();
    private final BlockingQueue<File> queue;
    private final FileFilter fileFilter;
    private final File root;

    class CrawlerThread extends Thread {
        public void run() {
            try {
                crawl(root);
            } catch (InterruptedException e) { /* 통과 */
            } finally {
                while (true) {
                    try {
                        queue.put(POISON);
                        break;
                    } catch (InterruptedException e1) { /* 재시도 */
                    }
                }
            }
        }

        private void crawl(File root) throws InterruptedException {
            ...
        }
    }

    class IndexerThread extends Thread {
        public void run() {
            try {
                while (true) {
                    File file = queue.take();
                    if (file == POISON)
                        break;
                    else
                        indexFile(file);
                }
            } catch (InterruptedException consumed) { }
        }
    }

    public void start() {
        producer.start();
        consumer.start();
    }

    public void stop() {
        producer.interrupt();
    }

    public void awaitTermination() throws InterruptedException {
        consumer.join();
    }
}
~~~
- 단일 프로듀서, 단일 컨슈머를 사용하면서 독약 객체를 사용해 작업을 종료하도록 만듦
- 독약 객체는 프로듀서의 개수와 컨슈머의 개수를 정확히 알고 있을 때에만 사용할 수 있음
- 독약 객체 방법은 크게 제한이 없는 큐를 사용할 때 효과적으로 동작함

### 7.2.4 예제: 단번에 실행하는 서비스
- 일련의 작업을 순서대로 처리해야 하며, 작업이 모두 끝나기 전에는 리턴되지 않는 메소드는 내부에서만 사용할 Executor 인스턴스를 하나 확보할 수 있다면 서비스의 시작과 종료를 쉽게 관리할 수 있음
	- invokeAll, invokeAny 메소드가 유용
~~~java
public boolean checkMail(Set<String> hosts, long timeout, TimeUnit unit)
        throws InterruptedException {
    ExecutorService exec = Executors.newCachedThreadPool();
    final AtomicBoolean hasNewMail = new AtomicBoolean(false);
    try {
        for (final String host : hosts)
            exec.execute(new Runnable() {
                public void run() {
                    if (checkMail(host))
                        hasNewMail.set(true);
                }
            });
    } finally {
        exec.shutdown();
        exec.awaitTermination(timeout, unit);
    }
    return hasNewMail.get();
}
~~~
- checkMail 메소드는 먼저 메소드 내부에 Executor 인스턴스를 하나 생성하고, 각 서버별로 구별된 작업을 실행시킴
- Executor 서비스를 종료시킨 다음, 각 작업이 모두 끝나면 Executor가 종료될 때까지 대기함

### 7.2.5 shutdownNow 메소드의 약점
- shutdownNow 메소드를 사용해 ExecutorService 를 강제로 종료시키는 경우에는 현재 실행 중인 모든 스레드의 작업을 중단시키도록 시도하고, 동록됐지만 실행은 되지 않았던 모든 작업의 목록을 리턴해줌
- 실행 시작은 했지만 아직 완료되지 않은 작업이 어떤 것인지를 알아볼 수 있는 방법은 없음
- 개별 작업 스스로가 작업 진행 정도 등의 정보를 외부에 알려주기 전에는 서비스를 종료하라고 했을 때 실행 중이던 작업의 상태를 알아볼 수 없음
- 종료 요청을 받았지만 아직 종료되지 않은 작업이 어떤 작업인지 확인하려면 실행이 시작되지 않은 작업도 알아야 할 뿐더러 Executor 가 종료될 때 실행 중이던 작업이 어떤 것인지도 알아야 함
~~~java
public class TrackingExecutor extends AbstractExecutorService {
    private final ExecutorService exec;
    private final Set<Runnable> tasksCancelledAtShutdown =
            Collections.synchronizedSet(new HashSet<Runnable>());

    ...

    public List<Runnable> getCancelledTasks() {
        if (!exec.isTerminated())
            throw new IllegalStateException(...);
        return new ArrayList<Runnable>(tasksCancelledAtShutdown);
    }

    public void execute(final Runnable runnable) {
        exec.execute(new Runnable() {
            public void run() {
                try {
                    runnable.run();
                } finally {
                    if (isShutdown()
                            && Thread.currentThread().isInterrupted())
                        tasksCancelledAtShutdown.add(runnable);
                }
            }
        });
    }
    // ExecutorService의 다른 메소드는 모두 exec에게 위임
}
~~~
- ExecutorService를 내부에 캡슐화해 숨기고, execute 메소드를 정교하게 호출하면서 종료 요청이 발생한 이후에 중단된 작업을 기억해둠
- TrackingExecutor는 시작은 됐지만 정상적으로 종료되지 않은 작업이 어떤 것인지를 정확하게 알 수 있음
- 이런 기법이 제대로 동작하도록 하려면 개별 작업이 리턴될 때, 자신을 실행했던 스레드의 인터럽트 상태를 유지시켜야 함

</br>

~~~java
public abstract class WebCrawler {
    private volatile TrackingExecutor exec;
    @GuardedBy("this") private final Set<URL> urlsToCrawl = new HashSet<URL>();

    ...

    public synchronized void start() {
        exec = new TrackingExecutor(Executors.newCachedThreadPool());
        for (URL url : urlsToCrawl) submitCrawlTask(url);
        urlsToCrawl.clear();
    }

    public synchronized void stop() throws InterruptedException {
        try {
            saveUncrawled(exec.shutdownNow());
            if (exec.awaitTermination(TIMEOUT, UNIT))
                saveUncrawled(exec.getCancelledTasks());
        } finally {
            exec = null;
        }
    }

    protected abstract List<URL> processPage(URL url);

    private void saveUncrawled(List<Runnable> uncrawled) {
        for (Runnable task : uncrawled)
            urlsToCrawl.add(((CrawlTask) task).getPage());
    }

    private void submitCrawlTask(URL u) {
        exec.execute(new CrawlTask(u));
    }

    private class CrawlTask implements Runnable {
        private final URL url;
        ...
        public void run() {
            for (URL link : processPage(url)) {
                if (Thread.currentThread().isInterrupted())
                    return;
                submitCrawlTask(link);
            }
        }

        public URL getPage() {
            return url;
        }
    }
}
~~~
- 작업 도중에 문서 수집기를 종료시키면 아직 시작하지 않은 작업과 실행 도중에 중단된 작업이 어떤 것인지를 찾아내며, 찾아낸 작업이 처리하던 URL을 기록해둠
- 기록해 둔 URL을 통해 문서 수집기를 다시 시작했을 때 처리해야 할 페이지 목록으로 쉽게 등록할 수 있음
- 멱등이 아닌 경우 안전성에 문제가 될 수 있으니 주의해야 함

</br>

## 7.3 비정상적인 스레드 종료 상황 처리
- 스레드를 예상치 못하게 종료시키는 가장 큰 원인은 바로 RuntimeException
- RuntimeException 은 대부분 프로그램이 잘못 짜여져서 발생하거나 기타 회복 불가능의 문제점을 나타내는 경우가 많기 때문에 try/catch 구문으로 잡지 못하는 경우가 많음
- RuntimeException 은 호출 스택을 따라 상위로 전달되기보다는 현재 실행되는 시점에서 콘솔에 스택 호출 추적 내용을 출력하고 해당 스레드를 종료시키도록 되어 있음
- 스레드 풀에서 사용하는 작업용 스레드나 스윙의 이벤트 처리 스레드와 같은 작업 처리용 스레드는 항상 Runnable 등의 인터페이스를 통해 남이 정의하고, 그래서 그 내용을 알 수 없는 작업을 실행하느라 온 시간을 보냄
- 작업 처리 스레드는 자신이 실행하는 남의 작업이 제대로 동작하지 않을 수 있다고 가정하고 조심스럽게 실행해야 함
- 작업 처리 스레드는 실행할 작업을 try-catch 구문 내부에서 실행해 예상치 못한 예외 상황에 대응할 수 있도록 준비하거나, try-finally 구문을 사용해 스레드가 피치 못할 사정으로 종료되는 경우에도 외부에 종료된다는 사실을 알려 프로그램의 다른 부분에서라도 대응할 수 있도록 해야 함
- RuntimeException 을 catch 구문에서 잡아 처리해야 할 상황은 그다지 많지 않은데, 몇 안 되는 상황 가운데 하나가 바로 남이 Runnable 등으로 정의해 둔 작업을 실행하는 프로그램을 작성하는 경우
~~~java
public void run() {
	Throwable thrown = null;
	try {
		while(!isInterrupted()) 
			runTask(getTaskFromWorkQueue());
	} catch (Throwable e) {
		thrown = e;
	} finally {
		threadExited(this, thrown);
	}
}
~~~
- 실행 중이던 작업에서정의도지 않은 예외 상황이 발생한다면, 결국 해당 스레드가 종료되지는 하지만 종료되기 직전에 스레드 풀에게 스스로가 종료된다는 사실을 알려주고 멈춤
	- 스레드 풀은 종료된 스레드를 삭제하고 새로운 스레드를 생성해 작업을 계속할 수 있음

</br>

### 7.3.1 정의되지 않은 예외 처리
- 스레드 API를 보면 UncaughtExceptionHandler라는 기능을 제공
- UncaughtExceptionHandler 기능을 사용하면 처리하지 못한 예외 상황으로 인해 특정 스레드가 종료되는 시점을 정확히 알 수 있음
- 처리하지 못한 예외 상황 때문에 스레드가 종료되는 경우, JVM이 애플리케이션에서 정의한 UncaughtExceptionHandler를 호출하도록 할 수 있음
~~~java
public interface UncaughtExceptionHandler {
	void uncaughtException(Thread t, Throwable e);
}
~~~

~~~java
public class UEHLogger implements Thread.UncaughtExceptionHandler {
    public void uncaughtException(Thread t, Throwable e) {
        Logger logger = Logger.getAnonymousLogger();
        logger.log(Level.SEVERE, "Thread terminated with exception: " + t.getName(), e);
    }
}
~~~
- 오류 메시지를 출력하고, 애플리케이션에서 작성하는 로그 파일에 스택 트레이스를 출력하는 등의 작업이 일반적
- 잠깐 실행하고 마는 애플리케이션이 아닌 이상, 예외가 발생했을 때 로그 파일에 오류를 출력하는 간단한 기능만이라도 확보할 수 있도록 모든 스레드를 대상으로 UncaughtExceptionHandler를 활용해야 함
- 작업을 실행하는 도중에 예외가 발생해 작업이 중단되는 경우가 생길 때 오류가 발생했다는 사실을 즉시 알고자 한다면, 
	- Runnable이나 Callable 인터페이스를 구현하면서 run 메소드에서 try/catch 구문으로 오류를 처리하도록 되어 있는 클래스르 ㄹ거쳐 실제 작업을 실행하도록 하거나
	- ThreadPoolExecutor 클래스에 마련되어 있는 afterExecute 메소드를 오버라이드하는 방법으로 오류 상황을 알리도록 함
- 예외 상황이 발생했을 때 UncaughtExceptionHandler가 호출되도록 하려면 **반드시** execute를 통해서 작업을 실행해야 함
- submit 메소드로 작업을 등록했다면, 그 작업에서 발생하는 모든 예외 상황은 모두 해당 작업의 리턴 상태로 처리해야 함
	- Future.get 메소드에서 해당 예외가 ExecutionException에 감싸진 상태로 넘어옴

</br>

## 7.4 JVM 종료
- JVM 이 종료되는 두 가지 경우
	- 예정된 절차대로 종료되는 경우
	- 예기치 못하게 임의로 종료되는 경우

</br>

### 7.4.1 종료 훅
- 예정된 절차대로 종료되는 경우에 JVM 은 가장 먼저 등록되어 있는 모든 종료 훅(shutdown hook)을 실행시킴
- 종료 훅은 Runtime.addShutdownHook 메소드를 사용해 등록된 아직 시작되지 않은 스레드를 의미
- 하나의 JVM 에 여러개의 종료 훅을 등록할 수도 있으며, 두 개 이상의 종료 훅이 등록되어 있는 경우에 어떤 순서로 훅을 실행하는지에 대해서는 아무런 규칙이 없음
- 종료 훅이 모두 작업을 마치고 나면 JVM 은 runFinalizersOnExit 값을 확인해 true 라고 설정되어 있으면 클래스의 finalize 메소드를 모두 호출하고 종료함
- JVM 은 종료 과정에서 계속해서 실행되고 있는 앱 내부의 스레드에 대해 중단 절차를 진행하거나 인터럽트를 걸지 않음
- 만약 종료 훅이나 finalize 메소드가 작업을 마치지 못하고 계속해서 실행된다면 종료 절차가 멈추는 셈이며, JVM 은 계속해서 대기 상태로 머무르기 때문에 결국 JVM 을 강제로 종료하는 수 밖에 없음
- JVM 을 강제로 종료시킬 때는 JVM 이 스스로 종료되는 것 이외에 종료 훅을 실행하는 등의 어떤 작업도 하지 않음
- 종료 훅은 스레드 안전하게 만들어야 함
- 공유된 자료를 사용해야 하는 경우에는 반드시 적절한 동기화 기법을 적용해야 함
- 앱의 상태에 대해 어떤 가정도 해서는 안 되며, JVM 이 종료되는 원인에 대해서도 생각해서는 안 되는 등 어떤 상황에서도 아무런 가정 없이 올바르게 동작할 수 있도록 굉장히 방어적인 형태로 만들어야 함
- JVM 이 종료될 때 종료 훅의 작업이 끝나기를 기다리기 때문에 마무리 작업을 최대한 빨리 끝내고 바로 종료해야 함
- 종료 훅은 어떤 서비스나 앱 자체의 여러 부분을 정리하는 목적으로 사용하기 좋음
	- 임시로 만들어 사용했던 파일을 삭제하거나, 운영체제에서 알아서 정리해주지 않는 모든 자원을 종료 훅에서 정리해야 함
~~~java
public void start() {
	Runtime.getRuntime().addShutdownHook(new Thread() {
		public void run() {
			try { LogService.this.stop(); }
			catch(InterruptedException ignored) { }
		}
	});
}
~~~
- 종료 훅이 여러개 등록되어 있는 경우에는 여러 개의 종료 훅이 서로 동시에 실행되지 때문에 다른 종료 훅에서 해당 LogService를 사용하고 있었다면 로그를 남기고자 할 때 이미 LogService가 종료되어 문제가 발생할 수 있음
- 종료 훅에서는 애플리케이션이 종료되거나 다른 종료 훅이 종료시킬 수 있는 서비스는 사용하지 말아야 함
- 모든 서비스를 정리할 수 있는 하나의 종료 훅을 사용해 각 서비스를 의존성에 맞춰 순서대로 정리하는 것도 방법
	- 각 서비스를 차례대로 정리하면 경쟁조건이나 데드락 등의 상황을 미연에 방지 할 수 있음

### 7.4.2 데몬 스레드
- 스레드는 두 가지 종류로 볼 수 있음
	- 일반 스레드, 데몬 스레드
- JVM 이 처음 시작할 때 main 스레드를 제외하고 JVM 내부적으로 사용하기 위해 실행하는 스레드(가비지 컬렉터 스레드나 기타 여러 부수적인 스레드)는 모두 데몬 스레드임
- 새로운 스레드가 생성되면 자신을 생성해 준 부모 스레드의 데몬 설정 상태를 확인해 그 값을 그대로 사용하며, 따라서 main 스레드에서 생성한 모든 스레드는 기본적으로 데몬 스레드가 아닌 일반 스레드임
- 일반 스레드와 데몬 스레드는 종료될 때 처리 방법이 약간 다를 뿐 그 외에는 모든 것이 완전히 동일함
- 스레드 하나가 종료되면 JVM 은 남아있는 모든 스레드 가운데 일반 스레드가 있는지를 확인하고, 일반 스레드는 모두 종료되고 남아있는 스레드가 모두 데몬 스레드라면 즉시 JVM 종료 절차를 진행함
- JVM 이 중단(halt)될 때는 모든 데몬 스레드가 버려지는 셈
- finally 블록의 코드도 실행되지 않으며, 호출 스택도 원상 복구되지 않음
- 데몬 스레드는 보통 부수적인 용도로 사용하는 경우가 많음
- 데몬 스레드에 사용했던 자원을 꼭 정리해야 하는 일을 시킨다면, JVM이 종료될 때 자원을 정리하지 못할 수 있기 때문에 적절하지 않음
- 데몬 스레드는 예고 없이 종료될 수 있기 때문에 앱 내부에서 시작시키고 종료하기에는 그다지 좋은 방법이 아님


### 7.4.3 finalize 메소드
- finalize 메소드는 과연 실행이 될 것인지 그리고 언제 실행될지에 대해서 아무런 보장이 없고, finalize 메소드를 정의한 클래스를 처리하는 데 상당한 성능상의 문제점이 생길 수 있음
- finalize 메소드를 올바른 방법으로 구현하기도 쉬운 일이 아님
- 대부분의 경우에는 finalize 메소드를 사용하는 대신 try-finally 구문에서 각종 close 메소드를 적절하게 호출하는 것만으로도 finalize 메소드에서 해야 할 일을 훨씬 잘 처리할 수 있음
- finalize 메소드가 더 나을 수 있는 유일한 예는 바로 네이티브 메소드에서 확보했던 자원을 사용하는 객체 정도밖에 없음
- finalize 메소드는 사용하지 마라

</br>

## 요약
- 작업, 스레드, 서비스, 앱 등이 할 일을 모두 마치고 종료되는 시점을 적절하게 관리하려면 프로그램이 훨씬 복잡해질 수 있음
- 자바에서는 선점적으로 작업을 중단하거나 스레드를 종료시킬 수 있는 방법을 제공하지 않음
- 인터럽트라는 방법을 사용해 스레드 간의 협력 과정을 거쳐 작업 중단 기능을 구현하도록 하고 있으며, 작업 중단 기능을 구현하고 전체 프로그램에 일괄적으로 적용하는 일은 모두 개발자의 몫
- FutureTask 나 Executor 등의 프레임웍을 사용하면 작업이나 서비스를 실행 도중에 중단할 수 있는 기능을 쉽게 구현할 수 있다는 점을 알아두어야 함