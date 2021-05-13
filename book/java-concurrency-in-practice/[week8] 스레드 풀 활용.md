# 스레드 풀 활용

</br>

## 8.1 작업과 실행 정책 간의 보이지 않는 연결 관계
- Executor 프레임웍이 나름대로 실행정책을 정하거나 변경하는데 있어 유연성을 가지고 있지만, 특정 형태의 실행정책에서는 실행할 수 없는 작업이 존재함
- 일정한 조건을 갖춘 실행 정책이 필요한 작업에는 아래와 같은 것들이 있음
	- **의존성이 있는 작업**
		- 다른 작업에 의존성을 갖는 작업을 스레드 풀에 올려 실행하려는 경우에는 실행 정책에 보이지 않는 조건을 거는 셈
		- 스레드 풀이 동작하는 동안 활동성 문제(liveness problem)가 발생하지 않도록 하려면 실행 정책에 대한 이와 같은 보이지 않는 조건을 면밀하게 조사하고 관리해야 함
	- **스레드 한정 기법을 사용하는 작업**
		- 단일 스레드로 동작하는 스레드 풀은 여러 스레드가 동작하는 경우보다 병렬 프로그램 입장에서 안전하게 동작함
		- 작업에서 사용하는 객체를 스레드 수준에 맞춰 한정할 수 있으므로, 같은 스레드에 한정되어 있는 객체라면 해당 객체가 스레드 안전성을 갖추고 있지 않다해도 얼마든지 마음대로 사용할 수 있음
		- 해당 작업을 실행하려면 Executor 프레임웍이 단일 스레드로 동작해야 한다는 조건이 생기기 때문에 작업과 실행정책 간의 보이지 않는 연결고리가 걸려있는 상황
		- 단일 스레드를 사용하는 풀 대신 여러 개의 스레드를 사용하는 풀로 변경하면, 스레드 안전성을 쉽게 잃을 수 있음
	- **응답 시간이 민감한 작업**
		- 단일 스레드로 동작하는 Executor에 오랫동안 실행될 작업을 등록하거나, 서너개의 스레드로 동작하는 풀에 실행 시간이 긴 작업을 몇개만 등록하더라도 해당 Executor를 중심으로 움직이는 화면 관련 부분은 응답 성능이 크게 떨어짐
	- **ThreadLocal을 사용하는 작업**
		- ThreadLocal을 사용하면 각 스레드에서 같은 이름의 값을 각자의 버전으로 유지할 수 있음
		- Executor는 상황이 되는대로 기존 스레드를 최대한 재사용함
		- 기본적으로 포함된 Executor는 처리해야할 작업의 수가 적을 땐 쉬는 스레드를 제거하기도 하고, 작업량이 많을 땐 새로운 스레드를 생성하여 사용하기도 함
		- 스레드 풀에 속한 스레드에서 ThreadLocal을 사용할 떄에는 현재 실행 중인 작업이 끝나면 더 이상 사용하지 않을 값만 보관해야함
		- ThreadLocal을 편법으로 활용해 작업 간에 값을 전달하는 용도로 사용해서는 안됨
- 스레드 풀은 동일하고 서로 독립적인 다수의 작업을 실행할 떄 가장 효과적임
- 실행시간이 오래 걸리는 작업과 금방 끝나는 작업을 섞여서 실행하도록 하면 풀의 크기가 굉장히 크지 않는 한 작업 실행을 방해하는 것과 비슷한 상황이 발생함
- 크기가 제한되어 있는 스레드 풀에 다른 작업에 의존성을 갖고 있는 작업을 등록하면 데드락이 발생할 가능성이 높음
- 다른 작업에 의존성이 있는 작업을 실행해야할 때는 스레드 풀의 크기를 충분히 크게 잡아서 작업이 큐에서 대기하거나 등록되지 못하는 상황이 없도록 해야함
- 스레드 한정 기법을 사용하는 작업은 반드시 순차적으로 실행돼야 함

</br>

### 8.1.1 스레드 부족 데드락
- 스레드 풀에서 다른 작업에 의존성을 갖고 있는 작업을 실행시킨다면 데드락에 걸릴 가능성이 높음
- 단일 스레드로 동작하는 Executor에서 다른 작업을 큐에 등록하고 해당 작업이 실행된 결과를 가져다 사용하는 작업을 실행하면, 데드락이 제대로 걸림
- 스레드 풀의 크기가 크더라도 실행되는 모든 스레드가 큐에 쌓여 아직 실행되지 않은 작업의 결과를 바등려고 대기 중이라면 이와 동일한 상황이 발생할 수 있음
	- 스레드 부족 데드락(thread starvation deadlock)이라 함
- 스레드 부족 데드락은 특정 자원을 확보하고자 계속해서 대기하거나 풀 내부의 다른 작업이 실행돼야 알 수 있는 조건이 만족하기를 기다리는 것처럼 끝없이 계속 대기할 가능성이 있는 기능을 사용하는 작업이 풀에 등록된 경우에는 언제든지 발생할 수 있음
~~~java
public class ThreadDeadlock {
    ExecutorService exec = Executors.newSingleThreadExecutor();

    public class RenderPageTask implements Callable<String> {
        public String call() throws Exception {
            Future<String> header, footer;
            header = exec.submit(new LoadFileTask("header.html"));
            footer = exec.submit(new LoadFileTask("footer.html"));
            String page = renderBody();
            // 데드락 발생
            return header.get() + page + footer.get();
        }
    }
}
~~~
- Executor에서 스레드를 하나만 쓰도록 구현하다면 ThreadDeadlock 클래스는 항상 데드락에 걸림
- 배리어(barrier)를 사용해 서로의 동작을 조율하는 작업 역시 풀의 크리가 충분히 크지 않다면 스레드 부족 데드락이 발생할 수 있음
- 완전히 독립적이지 않은 작업을 Executor에 등록할 때는 항상 스레드 부족 데드락이 발생할 수 있다느 사실을 염두에 둬야 하며, 작업을 구현한 코드나 Executor를 설정하는 설정 파일 등에 항상 스레드 풀의 크기나 설정에 대한 내용을 설명해야 함
- 스레드 풀의 크기는 직접적으로 지정하는 것 이외에도 스레드 풀에서 필요로 하는 자원이 제한되어 원하는 크기보다 작은 수준에서 동작하는 경우도 있음

### 8.1.2 오래 실행되는 작업
- 오래 실행되는 작업이 있다면 스레드 풀은 전체적인 작업 실행 과정에 어려움을 겪게 되며 금방 끝나는 작업이 실행되는 속도에도 영향을 미침
- 제한 없이 대기하느 ㄴ기능 대신 일저 ㅇ시간 동안만 대기하는 메소드를 사용할 수 있다면, 오래 실행되는 작업이 주는 악영향을 줄일 수 있는 하나의 방법으로 볼 수 있음
- 자바 라이브러리에서 제공하는 대부분의 블로킹 메소드는 시간이 제한되지 않는 것과 시간이 제한된 것이 함께 만들어져 있음
	- Thread.join(), BlockingQueue.put(), CountDownLatch.await(), Selector.select(), etc..
- 스레드 풀을 사용하는 도중에 모든 스레드에서 실행 중인 작업이 대기 상태에 빠지는 경우가 자주 발생한다면, 스레드 풀의 크기가 작다는 것으로 이해할 수도 있음

</br>

## 8.2 스레드 풀 크기 조절
- 스레드 풀의 가장 이상적인 크기는 스레드 풀에서 실행할 작업의 종류와 스레드 풀을 활용할 애플리케이션의 특성에 따라 결정됨
- 스레드 풀의 크기는 설정 파일이나 Runtime.availableProcess 등의 메소드 결과 값에 따라 동적으로 지정되어야 함
- 스레드 풀의 크기가 너무 크게 설정되어 있다면, 스레드는 CPU나 메모리 등의 자원을 더 확보하기 위해 경쟁하게 됨 --> 자원 부족으로 이어짐
- 스레드 풀의 크기가 너무 작다면, 작업량은 계속해서 쌓이는데 CPU나 메모리는 남아돌며 작업 처리속도가 떨어질 수 있음
- 스레드 풀의 크기를 적절하게 산정하려면 현재 컴퓨터 환경이 어느정도인지 확인해야 하고, 확보하고 있는 자원의 양도 알아야 하며, 해야할 작업이 어떻게 동작하는지도 정확히 알아야 함
- CPU를 많이 사용하는 작업의 경우 N개의 CPU를 탑재하고 있는 하드웨어에서 스레드 풀을 사용할 때는 스레드의 개수를 N+1개로 맞추면 최적의 성능을 발휘한다고 알려져 있음
- I/O 작업이 많거나 다른 브로킹 작업을 해야하는 경우라면 어느 순간에는 모든 스레드가 대기 상태에 들어가 전체 진행이 멈출수 있기 때문에 스레드 풀의 크기를 훨씬 크게 잡아야할 필요가 있음
- 스레드 풀의 크기를 정하려면 처리해야 할 작업이 시작해서 끝날 때까지 실제 작업하는 시간 대비 시간의 비율을 구해봐야함

~~~
Ncpu = CPU 개수
Ucpu = 목표로 하는 CPU 활용도. 0보다 크거나 같고 1보다 작거나 같음
W/C = 작업 시간 대비 대기 시간의 비율

CPU가 원하는 활용도를 유지할 수 있는 스레드 풀의 크기는  
Nthreads = Ncpu * Ucput * ( 1 + W/C )
~~~

- CPU의 개수는 Runtime 클래스의 availableProcessors 메소드로 아래와 같이 확인 가능    
`int N_CPU = Runtime.getRuntime().availableProcessors();`  

- CPU가 아닌 다른 자원을 대상으로 하는 스레드 풀의 크기를 정하는 일은 CPU 보다 쉬움
	- 메모리, 파일 핸들, 소켓 핸들, 데이터베이스 연결, etc...
	- 각 작업에서 실제로 필요한 자원의 양을 모두 더한 값을 자원의 전체 개수로 나워주면 됨 --> 스레드 풀의 최대 크기에 해당됨
- 각 작업 하나가 데이터베이스 연결 하나를 사용한다고 가정하면 스레드 풀의 실제 크기는 데이터베이스 연결 풀의 크기로 제한되는 셈
- 데이터베이스 연결 풀을 특정 스레드 풀에서만 사용한다고 하면, 데이터베이스 연결 풀에 확보된 연결 가운데 실제로 스레드 풀의 크기에 해당되는 연결만 사용될 것 

</br>

## 8.3 ThreadPoolExecutor 설정
- ThreadPoolExecutor는 Executors 클래스에 들어있는 newCachedThreadPool, newFixedThreadPool, newScheduledThreadPool과 같은 팩토리 메소드에서 생성해주는 Executor에 대한 기본적인 내용이 구현되어 있는 클래스
- ThreadPoolExecutor 클래스는 유연하면서도 안정적이고 여러가지 설정을 통해 입맛에 맞게 바꿔 사용할 수 있도록 되어있음
- 팩토리 메소드를 사용해 만들어진 스레드 풀의 기본 실행 정책이 요구사항에 잘 맞지 않는다면 ThreadPoolExecutor의 생성자를 호출해 스레드 풀을 생성할 수 있음
	- 생성자의 파라미터 값을 통해 스레드 풀 설정을 조절할 수 있음

</br>

### 8.3.1 스레드 생성과 제거
- 풀의 코어(core) 크기나 최대(maximum) 크기, 스레드 유지(keep-alive) 시간 등의 값을 통해 스레드가 생성되고 제거되는 과정을 조절할 수 있음
- 코어 크기는 스레드 풀을 사용할 때 원하는 스레드 개수라고 볼 수 있음
- 스레드 풀 클래스는 실행할 작업이 없다 하더라도 스레드의 개수를 최대한 코어 크기에 맞추도록 되어있음
	- 최초 ThreadPoolExecutor를 생성한 이후에도 prestartAllCoreThreads 메소드를 호출하지 않는 한 코어 크기만큼의 스레드가 미리 만들어지지는 않음
	- 작업이 실행되면서 코어 크기까지의 스레드가 차례로 생성됨
- 큐에 작업이 가득 차지 앟는 이상 스레드의 수가 코어 크기를 넘지 않음
- 풀의 최대 크기는 동시에 얼마나 많은 개수의 스레드가 동작할 수 있는지를 제한하는 최대 값임
- 코어 크기와 스레드 유지 시간을 적절하게 조절하면 작업없이 쉬고 있는 스레드가 차지하고 있는 자원을 프로그램의 다른 부분에서 활용하게 반납하도록 할 수 있음
- newFixedThreadPool
	- 메소드 결과로 생성할 스레드 풀의 코어 크기와 최대 크기를 메소드에 지정한 값으로 동일하게 지정됨
	- 시간 제한은 무제한으로 설종되는 것과 같음
- newCachedThreadPool
	- 스레드 풀의 최대 크기를 Inter.MAX_VALUE 값으로 지정하고 코어 크기를 0으로, 스레드 유지 시간을 1분으로 지정함
	- 스레드 풀은 끝없이 크기가 늘어날 수 있으며, 사용량이 줄어들면 스레드 개수가 적당히 줄어드는 효과가 있음

### 8.3.2 큐에 쌓인 작업 관리
- 크기가 제한된 스레드 풀에서는 동시에 실행될 수 있는 스레드의 개수가 제한되어 있음
- 작업을 처리할 수 있느 ㄴ능력보다 많은 양의 요청이 들어오면 처리하지 못한 요청이 큐에 계속 쌓임
- 스레드 풀을 사용하는 경우에는 Executor 클래스에서 관리하는 큐에 Runnable로 정의된 작업이 계속해서 쌓을 뿐, 스레드 풀 없이 스레드가 계속해서 생성됐을 때 각 스레드가 CPU를 확보하기 위해 대기하는 것과 다를바 없는 상황이 발생함
- ThreadPoolExecutor를 생성할 때 작업을 쌓아둘 큐로 BlockingQueue를 지정할 수 있음
- 스레드 풀에서 작업을 쌓아둘 큐에 적용할 수 있는 전략에는 세가지가 존재
	- 큐에 크기 제한을 두지 않는 방법
	- 큐의 크기를 제한하는 방법
	- 작업을 스레드에게 직접 넘겨주는 방법
- newFixedThreadPool 메소드와 newSingleThreadExecutor 메소드에서 생성하는 풀은 기본 설정으로 크기가 제한되지 않은 LinkedBlokingQueue를 사용함
	- 모든 스레드가 busy한 상태이고, 계속 작업이 추가되면 큐에 작업이 쌓임
- 자원 관리 측면에서 ArrayBlockingQueue 또는 크기가 제한된 LinkedBlockingQueue 나 PriorityBlockingQueue와 같이 큐의 크기를 제한시켜 사용하는 방법이 안정적임
	- 자원 사용량을 한정시킬 수 있지만, 큐가 가득 찼을 때 새로운 작업을 등록하려는 상황을 어떻게 처리해야 하는지 문제가 발생함
- 스레드의 개수는 줄이고 큐의 크기를 늘리면 메모리와 CPU 사용량은 줄이면서 컨텍스트 스위칭 횟수를 줄일 수 있지만, 전체적인 성능에는 제한이 생길 수 있음

- 스레드의 개수가 많거나 제한이 거의 없는 상태인 경우에는 작업을 큐에 쌓는 절차를 생략할 수 있을텐데, 이럴때는 SynchronousQueue를 사용해 프로듀서에서 생성한 작업을 컨슈머 스레드에게 직접 전달할 수 있음
- SynchronousQueue에 작업을 추가하려면 컨슈머 스레드가 대기하고 있어야 함
- 대기 중인 스레드가 없는 상태에서 스레드의 개수가 최대 크기보다 작다면 ThreadPoolExecutor는 새로운 스레드를 생성해 동작시킴
- 쉬고 있는 스레드에게 처리할 작업을 직접 넘겨주는 방법이 있다면 훨씬 효율적일 수 있음

</br>

- 크기가 고정된 풀보다는 newCachedThreadPool 팩토리 메소드가 생성해주는 Executor가 나은 선택일 수 있음
- 크기가 고정된 풀은 자원 관리 측면에서 동시에 실행되는 스레드 수를 제한해야하는 경우에 현명한 선택임
	- 네트웍으로 클라이언트 요청을 받아 처리하는 애플리케이션과 같은 경우, 크기가 고정되어 있지 않다면 요청이 많아져 부하가 걸릴 떄 문제가 커짐
- 스레드 풀에서 실행할 작업이 서로 독립적인 경우에만 스레드의 개수나 작업 큐의 크기를 제한할 수 있음
- 다른 작업에 의존성을 갖는 작업을 실행해야할 때 스레드나 큐의 크기가 제한되어 있다면 스레드 부족 데드락에 걸릴 가능성이 높음
	- 해당 경우는 newCachedThreadPool 메소드에서 생성하는 것과 같이 크기가 제한되지 않은 풀을 사용해야 함


### 8.3.3 집중 대응 정책
- 크기가 제한된 큐에 작업이 가득차면 집중 대응 정책(saturation policy)이 동작함
- ThreadPoolExecutor의 집중 대응 정책은 setRejectedExecutionHandler 메소드를 사용해 원하는 정책으로 변경할 수 있음
- RejectedExecutionHandler에는 AbortPolicy, CallerRunsPolicy, DiscardPolicy, DiscardOldestPolicy 등이 존재
- 중단(abort) 정책
	- 기본적으로 사용하는 정책, execute 메소드에서 RejectedExecutionException을 던짐
	- execute 메소드를 호출하는 스레드는 RejectedExecutionException을 잡아서 작업을 더이상 추가할 수 없는 상황에 직접 대응해야 함
- 제거(discard) 정책
	- 큐에 작업을 더이상 쌓을 수 없다면 추가하려한 정책을 아무 반응없이 제거함
- 오래된 항목 제거(discard oldest) 정책
	- 큐에 쌓은 항목 중 가장 오래되어 다음번에 실행될 예정이던 작업을 제거하고, 추가하고자 한 작업을 큐에 다시 추가함
- 호출자 실행(caller runs) 정책
	- 작업을 제거해 버리거나 예외를 던지지 않으면서 큐의 크기를 초과하는 작업을 프로듀서에게 거꾸로 넘겨 작업 추가 속도를 늦출 수 있도록 속도 조절 방법으로 사용됨
	- 즉, 새로 등록하려한 작업을 스레드 풀의 작업 스레드로 실행하지 않고, execute 메소드를 호출해 작업을 등록하려한 스레드에서 실행시킴
- 스레드 풀에 적용할 집중 대응 정책을 선택하거나 실행 정책의 다른 여러가지 설정을 변경하는 일은 모두 Executor를 생성할 때 지정할 수 있음
~~~java
ThreadPoolExecutor executor 
		= new ThreadPoolExecutor(N_THREADS, N_THREADS, 
								 0L, TimeUnit.MILLISECONDS,
								 new LinkedBlockingQueue<Runnable>(CAPACITY));
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
~~~
- 스레드 개수와 작업 큐의 크기가 제한된 스레드 풀을 만들면서 호출자 실행 정책을 지정하는 예
- 작업 큐가 가득 찼을 때 execute 메소드가 그저 대기하도록 하는 집중 대응 정책은 따로 만들어진 것이 없음
~~~java
@ThreadSafe
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
    }

    public void submitTask(final Runnable command)
            throws InterruptedException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
        }
    }
}
~~~
- semaphore를 사용하면 작업 추가 속도를 적절한 범위 내에서 제한할 수 있음
	- 큐의 크기에 제한을 두지 않아야하고, 스레드 풀의 스래드 개수와 큐에서 대기하도록 허용하고자 하는 최대 작업 개수를 더한 값을 세마포어의 크기로 지정해야 함


### 8.3.4 스레드 팩토리
- 스레드 풀에서 새로운 스레드를 생성해야 할 시점이 되면, 새로운 스레드는 항상 스레드 팩토리를 통해 생성함
~~~java
public interface ThreadFactory {
	Thread newThread(Runnable r);
}
~~~
- 기본값으로 설정된 스레드 팩토리에서는 데몬이 아니면서 아무런 설정도 변경하지 않은 새로운 스레드를 생성하도록 되어 있음
- ThreadFactory 클래스에는 newThread라는 메소드 하나만 정의되어 있음
	- 스레드 풀에서 새로운 스레드를 생성할 때에는 해당 메소드를 호출함
- 스레드 팩토리를 직접 작성해 사용해야 하는 경우
	- 스레드 풀에서 사용하는 스레드에 UncaughtExceptionHandler를 직접 지정하고자 할 경우
	- Thread 클래스를 상속받은 또 다른 스레드를 생성해 사용하고자 하는 경우
	- 새로 생성한 스레드의 실행 우선 순위를 조절하고자 하는 경우
	- 데몬 상태를 직접 지정하는 경우
	- 스레드마다 이름을 지정해 오류가 발생했을 때 덤프 파일이나 로그 파일에서 스레드 이름이 표시되도록 하는 경우

</br>

~~~java
public class MyThreadFactory implements ThreadFactory {
    private final String poolName;

    public MyThreadFactory(String poolName) {
        this.poolName = poolName;
    }

    public Thread newThread(Runnable runnable) {
        return new MyAppThread(runnable, poolName);
    }
}
~~~
- MyAppThread를 생성할 때 생성자에 스레드 풀 이름을 넘겨 덤프 파일이나 로그 파일에서 특정 스레드가 어떤 스레드 풀에 속해 동작하는지 확인할 수 있도록 함
- 디버깅할 때 요긴함

</br>

~~~java
public class MyAppThread extends Thread {
    public static final String DEFAULT_NAME = "MyAppThread";
    private static volatile boolean debugLifecycle = false;
    private static final AtomicInteger created = new AtomicInteger();
    private static final AtomicInteger alive = new AtomicInteger();
    private static final Logger log = Logger.getAnonymousLogger();

    public MyAppThread(Runnable r) {
        this(r, DEFAULT_NAME);
    }

    public MyAppThread(Runnable runnable, String name) {
        super(runnable, name + "-" + created.incrementAndGet());
        setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            public void uncaughtException(Thread t,
                                          Throwable e) {
                log.log(Level.SEVERE,
                        "UNCAUGHT in thread " + t.getName(), e);
            }
        });
    }

    public void run() {
        // debug 플래그를 복사해 계속해서 동일한 값을 갖도록 한다
        boolean debug = debugLifecycle;
        if (debug) log.log(Level.FINE, "Created " + getName());
        try {
            alive.incrementAndGet();
            super.run();
        } finally {
            alive.decrementAndGet();
            if (debug) log.log(Level.FINE, "Exiting " + getName());
        }
    }

    public static int getThreadsCreated() {
        return created.get();
    }

    public static int getThreadsAlive() {
        return alive.get();
    }

    public static boolean getDebug() {
        return debugLifecycle;
    }

    public static void setDebug(boolean b) {
        debugLifecycle = b;
    }
}
~~~
- 스레드 이름을 지정하는 것을 포함해, 작업 실행중 오류 발생시 Logger 클래스를 통해 로그로 남겨주는 UncaughtExceptionHandler를 직접 지정함
- 스레드가 몇개나 생성되고 제거됐는지에 대한 간단한 통계값도 보관함
- 스레드가 생성되고 종료될 때 디버깅용 메시지도 출력하도록 구현

</br>

- 애플리케이션에서 보안 정책(security policy)를 사용해 각 부분마다 권한을 따로 지정하고 있다면, Executors에 포함되어 있는 privilegedThreadFactory 팩토리 메소드를 사용해 스레드 팩토리를 만들어 사용하 ㄹ수 있음
- privilegedThreadFactory에서 만들어 낸 스레드 팩토리는 privilegedThreadFactory 메소드를 호출한 스레드와 동일한 권한, 동일한 AccessControlContext, 동일한 contextClassLoader 결과를 갖는 스레드를 생성함


### 8.3.5 ThreadPoolExecutor 생성 이후 설정 변경
- ThreadPoolExecutor를 생성할 때 생정자에 넘겨줬던 설정값 대부분은 set 메소드르 사용해 생성 후 변경이 가능함
~~~java
ExecutorService exec = Executors.newCachedThreadPool();
if(exec instanceof ThreadPoolExecutor)
	( (ThreadPoolExecutor) exec ).setCorePoolSize(10);
else
	throw new AssertionError("Oops, bad assumption");
~~~
- 스레드 풀을 ThreadPoolExecutor로 형변환해 여러가지 set 메소드를 사용 가능
- Executors에는 unconfigurableExecutorService 메소드가 있는데, 현재 만들어져 있는 ExecutorService를 넘겨 받은 다음 ExecutorService의 메소드만을 외부에 노출하고 나머지는 가리도록 래핑하여 설정을 변경하지 못하도록 할 수 있음
- 스레드 풀을 사용하지 않는 newSingleThreadExecutor 메소드는 ThreadPoolExecutor 인스턴스를 만들어주는 대신 단일 스레드라는 기능에 맞춰 래핑한 ExecutorService를 생성함
- 설정을 변경할 가능성이 있는 외부 코드에서도 직접 구현한 Executor 클래스의 설정을 변경하지 못하도록 하려면 앞서 소개했던 unconfigurableExecutorService 메소드를 사용하는 것이 나음

</br>

## 8.4 ThreadPoolExecutor 상속

