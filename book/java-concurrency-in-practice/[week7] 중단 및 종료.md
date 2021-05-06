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
