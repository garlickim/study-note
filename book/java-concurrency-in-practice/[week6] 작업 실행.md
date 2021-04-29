# 작업 실행
- 작업(task)란 추상적이면서 명확하게 구분된 업무 단위를 말함
- 애플리케이션이 해야할 일을 작업 단위로 분할하면 프로그램 구조를 간결하게 하고, 트랜잭션 범위를 지정하여 오류에 효과적으로 대응이 가능함

</br>

## 6.1 스레드에서 작업 실행
- 다른 작업의 상태, 결과, 부수 효과에 영향을 받지 않는 독립적인 작업은 적절한 자원이 확보된 상태에서 병렬로 실행될 수 있음
- 스케쥴링, 부하 분산(load balancing)을 할 때 유연성을 갖기 위해 작업은 작은 부분으로 구성되어 있어야 함
- 애플리케이션이 부하에따라 점진적으로 떨어지도록 설계되도록, 작업의 단위를 적절하게 설정하고 작업 실행 정책(task execution policy)을 구성해야 함
- 클라이언트의 개별 요청 단위를 작업의 범위로 지정한다면 대략 독립성을 보장받으며 작업의 크기를 적절하게 설정하는 것이라 판단 가능함

</br>

### 6.1.1 작업을 순차적으로 실행
- 작업을 실행하는 가장 간단한 방법은 단일 스레드에서 작업 목록을 순차적으로 실행하는 방법
~~~java
class SingleThreadWebServer {
	public static void main(String[] args) throws IOException {
		ServerSocket socket = new ServerSocket(80);
		while (true) {
			Socket conn = socket.accept();
			handleRequest(conn);
		}
	}
}
~~~
- 80 포트에 접속하는 클라이언트 요청을 순차적으로 처리
- 한번에 하나의 요청만을 처리할 수 있기 때문에 성능이 떨어짐
- 기본 스레드는 네트웍 소켓 연결을 기다리고 있다가 클라이언트가보내온 요청을 처리하는 과정을 반복함
	- 웹서버가 아직 요청을 처리하는 중이라면, 클라이언트는 이전 작업이 끝나기를 기다려야 함
- 단일 스레드로 처리하는 도중에는 어떤 작업에건 대기 상태에 들어간다는 의미가 처리 시간이 길어짐 뿐만 아니라 다른 요청을 전혀 처리하지 못한다는 문제임
- 단일 스레드에서 I/O 작업을 하는 동안 CPU가 대기하고 있어야 하는 등 서버 하드웨어 자원을 제대로 활용하지 못한다는 문제도 있음


### 6.1.2 작업마다 스레드를 직접 생성
- 반응 속도를 높이기 위해 요청이 들어올 때마다 새로운 스레드를 만들어 실행시키는 방법이 존재
~~~java
class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket conn = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(conn);
                }
            };
            new Thread(task).start();
        }
    }
}
~~~
- 작업을 처리하는 기능이 메인 스레드에서 분리됨
	- 메인 반복문은 다음 요청을 받을 수 있도록 빨리 넘어감
	- 서버의 응답 속도를 높여줌
- 동시에 여러 작업을 병렬로 처리할 수 있어 2개 이상의 요청을 받아 처리가 가능함
	- 여러 개의 CPU가 장착되어 있다면 처리 속도를 향상시킬 수 있고, I/O, 락 확보를 위한 대기, 특정 자원 확보를 위한 대기 등에서 서버의 처리 속도를 높임
- 작업 처리 스레드는 동시에 동작할 가능성이 높아 스레드 안정성을 확보해야 함
- 클라이언트 요청 전송 속도 < 요청 처리 응답 속도(더 빨라야 함) 여야 한다는 제약 사항이 지켜져야 괜찮은 응답 속도와 성능을 보여줌

### 6.1.3 스레드를 많이 생성할 때의 문제점
- 특정 상황에서 대량의 스레드가 생성되는 경우, 아래와 같은 문제가 발생함
	- **스레드 라이프 사이클 문제**
		- 스레드를 생성하고 제거하는 작업에도 자원이 소모됨
		- 스레드 생성에 시간이 요소되어, 요청 처리시 기본적인 딜레이가 발생함
		- 클라이언트 요청이 간단하며, 빈도가 잦다면 요청 처리 시간 보다 스레드 생성 시간이 더 소요될 수 있음 
	- **자원 낭비**
		- 실행중인 스레드는 시스템의 자원(특히 메모리)을 많이 소모함
		- 프로세서의 수보다 생성된 스레드의 수가 많으면, 실제로는 대부분의 스레드가 대기(idle) 상태
		- 대기 상태의 스레드가 많아질수록 많은 메모리를 필요로 하며, GC 부하가 늘어남
		- CPU를 사용하기 위한 스레드 경쟁이 일어남
		- CPU 개수 만큼의 스레드가 동작 중이라면, 스레드를 더 생성해도 성능은 나아지지 않음
	- **안정성 문제**
		- 모든 시스템에는 생성할 수 있는 스레드의 개수가 제한되어 있음
		- 제한된 양을 모두 사용하면 OutOfMemoryError가 발생할 수 있음
- 일정 수준까지는 스레드를 추가로 만들어 사용해 성능상의 이점을 얻을 수 있으나, 특정 수준을 넘어가면 성능이 떨어짐
- 애플리케이션이 만들어 낼 수 있는 스레드의 수에 제한을 두는 것이 현명한 방법
- 애플리케이션이 제한된 수의 스레드만으로 동작할 때 다량의 요청이 들어오는 상황에서도 자원이 고갈되어 멈추는 경우가 없는지 세심한 테스트가 필요함

</br>

## 6.2 Executor 프레임웍
- 작업(task)은 논리적인 업무 단위이며, 스레드는 특정 작업을 비동적으로 동작시킬 수 있는 방법을 제공
- 스레드 풀(thread pool)은 스레드를 관리하는 측면에서 통제력을 갖출수 있도록 해줌
	- java.util.concurrent 패키지에 보면 Executor 프레임웍의 일부분으로 유연하게 사용할 수 있는 스레드 풀이 만들어져 있음
~~~java
public interface Executor {
	void execute(Runnable command);
}
~~~
- 자바 클래스 라이브러리에서 작업을 실행하고자 할 때는 Thread 보다 Executor가 훨씬 추상화가 잘되어 있어 사용하기 좋음
- Executor는 여러가지 종류의 작업 실행 정책을 지원하는 유연하면서도 비동기적 작업 실행 프레임웍의 근간을 이루는 인터페이스
- Executor는 작업 등록(task submission)과 작업 실행(task execution)을 분리하는 표준적인 방법이며, 각 작업은 Runnable 의 형태로 정의함
- Executor를 구현한 클래스는 작업의 라이프 사이클을 관리하는 기능을 갖고 있으며, 통계값 추출/실행과정 모니터링을 위한 기능도 지님
- 프로듀서-컨슈머 패턴을 애플리케이션에 적용해 구현할 수 있는 가장 쉬운 방법이 Executor 프레임웍을 사용하는 방법

</br>

### 6.2.1 예제: Executor를 사용한 웹서버
~~~java
class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket conn = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(conn);
                }
            };
            exec.execute(task);
        }
    }
}
~~~
- 요청 처리 작업을 등록하는 부분과 실제로 처리 기능을 실행하는 부분이 Executor를 사이에 두고 분리되어 있음
- Executor를 사용하면 Executor의 설정을 변경하는 것만으로 서버의 동작 특성을 쉽게 변경할 수 있음
- Executor에 필요한 설정은 처음 실행하는 시점에 설정 값을 지정하는 편이 좋음
- 다만, Executor를 사용해 작업을 등록하는 코드는 프로그램 전체에 퍼져있는 경우가 많아 한눈에 보기 어려움

</br>

~~~java
public class ThreadPerTaskExecutor implements Executor {
	public void execute(Runnable r) {
		new Thread(r).start();
	};
}
~~~
- TaskExecutionWebServer의 구조를 유지하며, 요청마다 새로운 스레드를 생성해 실행하도록 함

</br>

~~~java
public class WithinThreadExecutor implements Executor {
	public void execute(Runnable r) {
		r.run();
	};
}
~~~
- TaskExecutionWebServer가 작업을 순차적으로 처리하도록 만들 수 있음
	- execute() 안에서 요청에 대한 처리 작업을 모두 실행하고, 처리가 끝나면 executor 에서 리컨되도록 구현하면 됨
- 작업을 등록한 스레드에서 직접 동작시킴

### 6.2.2 실행 정책
- 작업을 등록하는 부분과 실행하는 부분을 분리하면, 실행 정책(execution policy)을 언제든지 쉽게 변경할 수 있다는 장점이 존재
- 실행 정책은 '무엇을, 어디에서, 언제, 어떻게' 실행하는지를 지정할 수 있음
	- 작업을 어느 스레드에서 실행할 것인가?
	- 작업을 어떤 순서로 실행할 것인가? (FIFO, LIFO, 우선순위 정책)
	- 동시에 몇 개의 작업을 병렬로 실행할 것인가?
	- 최대 몇 개까지의 작업이 큐에서 실행을 대기할 수 있게 할 것인가?
	- 시스템에 부하가 많이 걸려서 작업을 거절해야 하는 경우, 어떤 작업을 희생양으로 삼아야 할 것이며, 작업을 요청한 프로그램에 어떻게 알려야 할 것인가?
	- 작업을 실행하기 직전이나 실행한 직후에 어떤 동작이 있어야 하는가?
- 가장 최적화된 실행 정책을 찾기위해 하드웨어/소프트웨어 자원을 얼마나 확보할 수 있는지 확인해야 하고 , 성능과 반응 속도의 요구사항을 파악해야 함
- 실행 정책과 작업 등록 부분을 명확하게 분리시켜두면, 설치할 하드웨어와 기타 자원의 양에 따라 적절한 실행 정책을 임의로 지정할 수 있음
- 프로그램에서 `new Thread(runnable).start()` 와 같은 코드가 존재한다면, 유연한 실행 정책을 적용할 준비를 해야함
	- Executor를 사용해 구현하는 방안도 고려해야 함

### 6.2.3 스레드 풀
- 스레드 풀(thread pool)은 작업을 처리할 수 있는 동일한 형태의 스레드를 풀의 형태로 관리함
- 작업 스레드의 동작 주기 (반복)
	- 작업 큐에서 실행할 작업 꺼내오기 -> 작업 실행 -> 다음 작업이 나타날 때까지 대기 
- 스레드 풀을 사용하면 이전에 사용한 스레드를 재사용하기 때문에 스레드를 계속해서 생성할 필요가 없음
	- 여러 개의 요청을 처리하는 데 필요한 시스템 자원이 줄어드는 효과가 있음
	- 클라이언트의 요청시, 요청을 처리할 스레드가 이미 만들어진 상태로 대기중이기 때문에 딜레이가 발생하지 않아 전체적인 반응 속도가 향상됨
- 적절한 크기의 스레드 풀은 프로세서가 쉬지 않고 동작하도록 할 수 있으며, 메모리를 전부 소모하거나 한정된 자원을 두고 경쟁하는 현상도 사라짐
- 미리 정의되어 있는 스레드 풀을 사용하려면 Executors 클래스에 만들어져 있는 메소드를 호출
	- **newFixedThreadPool**
		- 처리할 작업이 등록되면 그에 따라 실제 작업할 스레드를 하나씩 생성
		- 생성할 수 있는 스레드 최대 개수는 제한되어 있으며, 제한된 개수 이후는 더이상 생성하지 않고 스레드 수를 유지함
		- 만약 스레드가 작업 도중, 예상치 못한 예외가 발생하여 스레드가 종료되면 하나씩 더 생성하기도 함
	- **newCachedThreadPool**
		- 현재 풀에 갖고 있는 스레드의 수가 처리할 작업의 수보다 많아서 쉬는 스레드가 많이 발생할 때, 쉬는 스레드는 종료시켜 유연하게 대응 가능함
		- 처리할 작업의 수가 많아지면 필요한 만큼 스레드를 새로 생성
		- 스레드 수에는 제한을 두지 않음
	- **newSingleThreadExecutor**
		- 단일 스레드로 동작하는 Executor로서 작업을 처리하는 스레드가 1개뿐
		- 작업중 Exception이 발생하여 비정상적으로 스레드가 종료되면, 새로운 스레드를 생성하여 나머지 작업을 실행함
		- 등록된 작업은 설정된 큐에서 지정하는 순서(FIFO, LIFO, 우선순위)에 따라 순차적으로 처리됨
	- **newScheduledThreadPool**
		- 일정 시간 이후에 실행하거나 주기적으로 작업을 실행할 수 있음
		- 스레드의 수가 고정되어 있는 형태의 Executor.Timer 클래스의 기능과 유사함
- 작업별로 스레드를 생성하는 전략(thread-per-task)에서 풀을 기반으로 하는 전략(pool-based)으로 변경하면 안정성 측면에서 장점을 얻을 수 있음
	- 웹서버에 부하가 걸리더라도 메모리가 부족해 죽는일(스레드를 생성하느라 메모리가 모자라 죽는일)이 발생하지 않는다는 점
	- 웹서버에 부하가 걸릴 때, 성능이 점진적으로 서서히 떨어지는 특징을 갖음
- Executor를 사용하면 사용하지 않을 때보다 애플리케이션의 성능 튜닝, 모니터링, 로깅, 오류 처리 등의 방법으로 효과적으로 처리하기 좋음

### 6.2.4 Executor 동작 주기
- Executor를 구현하는 클래스는 대부분 작업을 처리하기 위한 스레드를 생성하도록 되어있음
- Executor를 제대로 종료하지 않으면 JVM 자체가 종료되지 않고 대기하기도 함
- Executor는 작업을 비동적으로 실행하기 때문에 앞서 실행시켰던 작업의 상태를 특정 시점에 정확하게 파악하기 어려움
- 애플리케이션을 종료하는 방법은 안전한 종료 방법과 강제적인 종료 방법이 존재
- 서비스를 실행하는 동작 주기와 관련해 Executor를 상속받는 ExecutorService 인터페이스에는 동작 주기를 관리할 수 있는 메소드가 추가되어 있음
~~~java
public interface ExecutorService extends Executor {
	void shutdown();
	List<Runnable> shutdownNow();
	boolean isShutdown();
	boolean isTerminated();
	boolean awaitTermination(long timeout, TimeUnit unit)
		throws InterruptedException;
	// ... 작업을 등록할 수 있는 몇 가지 추가 메소드
}
~~~
- ExecutorService가 갖고 있는 동작 주기에는 실행중(running), 종료중(shutting down), 종료(terminated) 세가지 상태가 있음
- ExecutorService를 처음 생성하면 실행중 상태
- shutdown 메소드를 실행하면 안전한 종료 절차를 진행하며 종료중 상태로 진입
	- 종료중 상태에서는 새로운 작업을 등록받지 않으며, 이전 작업을 모두 끝마침
- shutdownNow 메소드를 실행하면 강제 종료 절차를 진행
	- 현재 진행중인 작업도 가능한 한 취소시키고, 실행되지 않고 대기 중이던 작업은 더이상 실행시키지 않음
- ExecutorService의 하위 클래스인 ThreadPoolExecutor는 종료 절차가 진행중일 때, 새로운 작업을 등록하려하면 실행 거절 핸들러(rejected execution handler)를 통해 오류로 처리함
	- 실행 거절 핸들러에 따라 다르지만, 등록하려는 작업을 무시할 수도 있으며 RejectedExecutionException을 발생시킬수도 있음
- ExecutorService가 종료 상태로 들어갈때까지 기다리고자 한다면 awaitTermination 메소드로 대기할 수 있음
- isTerminated 메소드를 주기적으로 호출해 종료 상태로 들어갔는지 확인 가능
~~~java
public class LifecycleWebServer {
    private final ExecutorService exec = ...;

    public void start() throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (!exec.isShutdown()) {
            try {
                final Socket conn = socket.accept();
                exec.execute(new Runnable() {
                    public void run() {
                        handleRequest(conn);
                    }
                });
            } catch (RejectedExecutionException e) {
                if (!exec.isShutdown())
                    log("task submission rejected", e);
            }
        }
    }

    public void stop() {
        exec.shutdown();
    }

    void handleRequest(Socket connection) {
        Request req = readRequest(connection);
        if (isShutdownRequest(req))
            stop();
        else
            dispatchRequest(req);
    }
}
~~~
- LifecycleWebServer는 두가지 방법으로 종료시킬 수 있음
	- stop 메소드를 호출하는 방법
	- 클라이언트 축에서 특정한 형태의 HTTP 요청을 전송하는 방법

### 6.2.5 지연 작업, 주기적 작업
- 자바 라이브러리에 포함된 Timer 클래스를 사용하면 특정 시간 이후에 원하는 작업을 실행하는 지연 작업이나 주기적인 작업을 실행할 수 있음
- Timer는 그 자체로 약간의 단점이 있기 때문에 ScheduledThreadPoolExecutor를 사용하는 방법이 나음
- ScheduledThreadPoolExecutor를 생성하는 방법
	- ScheduledThreadPoolExecutor 클래스의 생성자를 호출해 생성
	- newScheduledThreadPool 팩토리 메소드를 사용해 생성
- Timer 단점
	- Timer 클래스는 등록된 작업을 실행시키는 스레드를 하나만 생성해 사용함
	- Timer에 등록된 작업이 오래 걸리면, 등록된 다른 Task 작업이 정해진 시간에 실행되지 않을 가능성이 높음
	- TimerTask가 동작하던 도중에 예상치 못한 Exception을 던지는 경우, 예측하지 못한 상태로 넘어갈 수 있음
		- Timer 스레드는 예외를 전혀 처리하지 않기 때문에 Timer 스레드 자체가 멈춰버릴 가능성이 존재
	- 오류가 발생해 스레드가 종료된 상황에서 자동으로 새로운 스레드를 만들어 주지 않음
		- 이런 상황에 다다르면 Timer에 등록된 모든 작업이 취소되었다고 판단하고, 등록됐던 TimerTask는 실행되지 않으며 새로운 작업을 등록할 수 없음 
- ScheduledThreadPoolExecutor를 사용하면 지연 작업과 주기적 작업마다 여러 개의 스레드를 할당해 작업을 실행함
	- 각자 실행 예정 시각을 벗어나는 일이 없도록 조절해줌
~~~java
public class OutOfTime {
    public static void main(String[] args) throws Exception {
        Timer timer = new Timer();
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(1);
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(5);
    }

    static class ThrowTask extends TimerTask {
        public void run() {
            throw new RuntimeException();
        }
    }
}
~~~
- Timer 클래스가 내부적으로 어떻게 꼬일 수 있는지 보여주고, 문제 발생시 작업을 등록하려는 애플리케이션에서 어떤 문제가 발생하는지 보여주는 예
- 6초 동안 실행되다 종료될 것이라 예상되지만, 1초 후 "Timer already cancelled" 메시지를 뿌리고 IllegalStateException 발생 후 종료됨
	- ScheduledThreadPoolExecutor에서는 이와 같이 오류가 발생하는 경우를 안정적으로 처리해줌
- 특별한 스케쥴 방법을 지원하는 서비스를 구현해야 한다면 DelayQueue 사용을 권장
- DelayQueue는 큐 내부에 여러 개의 Delayed 객체로 작업을 관리함
	- 각각의 Delayed 객체는 저마다의 시각을 갖고 있음
	- DelayQueue에서 뽑아내는 객체는 객체마다 지정되어 있던 시각 순서로 정렬되어 뽑아짐

</br>

## 6.3  병렬로 처리할 만한 작업
- 서버 애플리케이션에서 클라이언트의 요청 한 건을 처리하는 과정에서도 병렬화해 처리하는 모습을 볼 수 있음
	- 데이터베이스 서버 같은 경우에 이런 기법을 많이 사용

</br>

### 6.3.1 예제: 순차적 페이지 렌더링
~~~java
public class SingleThreadRenderer {
    void renderPage(CharSequence source) {
        renderText(source);
        List<ImageData> imageData = new ArrayList<ImageData>();
        for (ImageInfo imageInfo : scanForImageInfo(source))
            imageData.add(imageInfo.downloadImage());
        for (ImageData data : imageData)
            renderImage(data);
    }
}
~~~
- 이미지를 다운로드 받는 작업은 I/O 작업이며, I/O 완료시까지 대기 시간이 상당히 길지만 실제로 CPU가 하는 일은 거의 없음
- 순차적인 방법은 CPU의 능력을 제대로 활용하지 못하는 경우가 많으며, 사용자는 똑같은 내용을 오랫동안 봐야하는 문제가 있음
- 처리해야 할 큰 작업(HTML 페이지 렌더링)은 작은 단위의 작업으로 쪼개 동시에 실행 할 수 있도록 하여 CPU를 더 활용하고, 처리 속도 및 응답 속도를 개선하는 것이 좋음


### 6.3.2 결과가 나올 때까지 대기 : Callable과 Future
- Runnable은 run 메소드 실행이 끝난 다음 결과 값을 리턴해 줄 수 없으며, 예외 발생 가능성을 throws 구문으로 표현할 수 없음
	- 결과 값을 만들어 공유된 장소에 저장해야 하고, 로그 파일에 오류 내용을 기록하는 정도가 일반적인 처리 방법
- 결과를 얻어오는데 시간이 걸리는 기능은 Runnable 대신 Callable을 사용하는 게 모양새가 좋음
- Callable 인터페이스에서는 call 메소드를 실행하고 나면 결과 값을 돌려받을 수 있으며, Exception도 발생시킬 수 있음
	- 결과값을 리턴하지 않는 작업을 Callable로 지정하려면 Callable<Void>로 표현 가능
- Runnable과 Callable은 어떤 작업을 추상화하기 위한 도구
- Executor에서 실행한 작업은 생성(created), 등록(submitted), 실행(started), 종료(completed)와 같은 네 가지의 상태를 통과함
- 작업은 상당 시간동안 실행될 수 있어 작업을 중간에 취소할 수 있는 기능이 있어야 함
- Executor는 등록됐지만 시작되지 않은 작업은 언제든지 실행하지 않도록 취소 시킬 수 있음
- Future는 특정 작업이 정상적으로 완료됐는지, 아니면 취소됐는지 등에 대한 정보를 확인할 수 있도록 만들어진 클래스
~~~java
public interface Callable<V> {
	V call() throws Exception;
}

public interface  Future<V> {
	boolean cancel(boolean mayInterruptIfRunning);
	boolean isCancelled;
	boolean isDone;
	V get() throws InterruptedException, ExecutionException, CancellationException
	V get(long timeout, TimeUnit unit)
		throws InterruptedException, ExecutionException, CancellationException, TimeoutException;
}
~~~
- Future는 한번 지나간 상태는 되돌릴 수 없다는 점이 특정
	- 사이클을 되돌릴 수 없다는 것은 ExecutorService와 동일
- get 메소드는 작업이 진행되는 상태에 따라 다른 유형으로 동작함
	- 작업이 진행되는 상태 : 시작되지 않은 상태, 시작한 상태, 완료된 상태 등
	- 작업이 완료 상태에 들어가 있다면 get 메소드를 호출했을 때 즉시 결과 값을 리턴하거나 Exception을 발생시킴
	- 작업을 시작하지 않았거나 실행중이라면 작업이 완료될 때까지 대기
	- 작업이 끝난 상태에서 Exception이 발생했다면 get 메소드는 원래 발생했던 Exception을 ExecutionException에 담아 던짐
	- 작업이 중간에 취소됐다면 get 메소드에서 CancellationException이 발생
	- get 메소드에서 ExecutionException이 발생한 경우, 원래 발생한 오류는 ExecutionException의 getCause()로 확인 가능

</br>

- 실행하고자 하는 작업을 나타내는 Future 클래스는 여러가지 방법으로 만들어 낼 수 있음
- ExecutorServer 클래스의 submit 메소드는 Future 인스턴스를 리턴
- Executor에 Runnable이나 Callable을 등록하면 Future 인스턴스를 받을 수 있음
	- 받은 인스턴스를 사용해 작업 결과 확인 및 실행 도중 작업을 취소할 수 있음
- Runnable 이나 Callable을 사용해 직접 FutureTask 인스턴스를 생성하는 방법도 존재
~~~java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> task) {
	return new FutureTask<T>(task);
}
~~~
- ExecutorService를 구현하는 클래스에서 AbstractExecutorService에 정의된 newTaskFor 메소드를 오버라이드 할 수 있도록 되어있음
- newTaskFor를 오버라이드해 등록된 Runnable이나 Callable에 따라 Future를 생성하는 기능에 직접 관여 할 수 있음
- 기본 설정으로는 새로운 FutureTask 인스턴스를 생성함

</br>

- Executor에 Runnable이나 Callable을 넘겨 등록하는 것은 실제 작업을 실행할 스레드로 안전하게 공개하는 과정을 거치도록 되어있음
- Future에 결과값을 설정하는 부분도 get 메소드로 결과 값을 가져가려는 스레드로 결과 객체를 안전하게 공개하도록 되어있음

### 6.3.3 예제: Future를 사용해 페이지 렌더링
- CPU를 많이 사용하는 텍스트를 그려넣는 작업과 I/O를 많이 사용하는 이미지 다운로드 기능을 분리
- Callable과 Future 인터페이스를 사용하면 여러 스레드가 서로 상대방을 살펴가며 동작하는 논리구조를 쉽게 설계 할 수 있음
~~~java
public class FutureRenderer {
    private final ExecutorService executor = ...;

    void renderPage(CharSequence source) {
        final List<ImageInfo> imageInfos = scanForImageInfo(source);
        Callable<List<ImageData>> task =
                new Callable<List<ImageData>>() {
                    public List<ImageData> call() {
                        List<ImageData> result = new ArrayList<ImageData>();
                        for (ImageInfo imageInfo : imageInfos)
                            result.add(imageInfo.downloadImage());
                        return result;
                    }
                };

        Future<List<ImageData>> future = executor.submit(task);
        renderText(source);

        try {
            List<ImageData> imageData = future.get();
            for (ImageData data : imageData)
                renderImage(data);
        } catch (InterruptedException e) {
            // 스레드의 인터럽스 상태를 재설정
            Thread.currentThread().interrupt();
            // 결과는 더 이상 필요없으니 해당 작업도 취소한다
            future.cancel(true);
        } catch (ExecutionException e) {
            throw launderThrowable(e.getCause());
        }
    }
}
~~~
- 이미지를 다운로드 받는 기능의 Callable을 만들어 ExecutorService에 등록
- Callable이 등록되는 즉시 해당하는 작업에 대한 Future 인스턴스를 받을 수 있음
- Future.get()을 통해 이미지 파일 확보
	- 낙관적인 관점에서 get 메소드 호출하기 전 이미지를 모두 다운로드 했을 것이며, 메인 스레드는 필요한 이미지를 바로 사용할 수 있음
	- 보통의 관점에서도 이미지 다운로드 기능이 시작된 상태이기 때문에 순차적인 방법보다는 효율적임
- Future를 활용하는 작업이 스레드 안전성을 확보했다고 할 수 있음
	- 작업 등록/결과 조회시 안전한 공개 방법을 사용하기 때문
- Future.get() 을 감싸고 있는 오류 처리 구문에서는 발생할 수 있는 두가지 가능성을 대응해야 함
	- Exception이 발생하는 경우
	- 결과 값을 얻기 전에 get 메소드를 호출해 대기하던 메인 스레드가 인터럽트되는 경우


### 6.3.4 다양한 형태의 작업을 병렬로 처리하는 경우의 단점
- 다양한 종류의 작업을 여러 작업 스레드에서 나워 처리하도록 할 때에는 나눠진 작업이 일정한 크기를 유지하지 못할 수 있다는 단점이 존재
	- A,B 작업 스레드에서 A를 실행하는데 B 보다 10대의 시간이 걸린다고 하면, 전체적인 실행 시간 측면에서 9% 이득만 존재함
- 여러 개의 작업 스레드가 하나의 작업을 나눠 실행시킬 때는 항상 작업 스레드 간에 필요한 내용을 조율하는데 일부 자원을 소모하게 됨
- 작업을 잘게 쪼개는 의미를 찾으려면 병렬로 처리해서 얻을 수 있는 성능상의 이득이 이와 같은 부하를 훨씬 넘어야 함
- 여러 종류의 작업을 병렬로 처리해 병렬성을 높이고자 노력하는 것은 상당한 양의 업무 부하가 될 수 있지만, 결과로 얻을 수 있는 이득에는 한계가 있음을 알아야 함
- 실제적인 이점을 얻기 위해서는 프로그램이 하는 일을 대량의 동일한 작업으로 재정의해 병렬로 처리할 수 있어야 함


### 6.3.5 CompletionService: Executor와 BlockingQueue의 연합
- 처리해야 할 작업을 갖고 있고, 작업 결과가 나오는 즉시 값을 가져다 사용하고자 할 때 Completion service가 존재
- CompletionService는 Executor의 기능과 BlockingQueue의 기능을 하나로 모은 인터페이스
- 필요한 Callable 작업을 등록해 실행시킬 수 있으며, take/poll과 같은 큐 메소드를 사용해 작업이 완료되는 순간 Future 인스턴스를 받아올 수 있음
- CompletionService를 구현한 클래스로는 ExecutorCompletionService가 있으며, 등록된 작업은 Executor를 통해 실행됨
- ExecutorCompletionService
	- 생성자에서 완료된 결과값을 쌓아 둘 BlockingQueue 생성
	- FutureTask의 작업이 모두 완료되면 done 메소드가 한 번씩 호출됨
~~~java
private class QueueingFuture<V> extends FutureTask<V> {
	QueueingFuture(Callable<V> c) { super(c); }
	QueueingFuture(Runnable t, V r) { super(t, r); }

	protected void done() {
		completionQueue.add(this);
	}
}
~~~
- 작업을 처음 등록하면 FutureTask을 상속받은 QueueingFuture 클래스로 변환
- QueueingFuture의 done 메소드에서는 결과를 BlockingQueue에 추가하도록 되어 있음
- take/poll 메소드를 호출하면 그대로 BlockingQueue의 해당 메소드로 넘겨 처리함


### 6.3.6 예제: CompletionService를 활용한 페이지 렌더링
- CompletionService를 활용하면 HTML 렌더링 프로그램의 성능을 두 가지 측면에서 개선할 수 있음
	- 전체 실행시간을 줄임
	- 응답 속도를 높임
~~~java
public class Renderer {
    private final ExecutorService executor;

    Renderer(ExecutorService executor) {
        this.executor = executor;
    }

    void renderPage(CharSequence source) {
        final List<ImageInfo> info = scanForImageInfo(source);
        CompletionService<ImageData> completionService =
                new ExecutorCompletionService<ImageData>(executor);
        for (final ImageInfo imageInfo : info)
            completionService.submit(new Callable<ImageData>() {
                public ImageData call() {
                    return imageInfo.downloadImage();
                }
            });

        renderText(source);

        try {
            for (int t = 0, n = info.size(); t < n; t++) {
                Future<ImageData> f = completionService.take();
                ImageData imageData = f.get();
                renderImage(imageData);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            throw launderThrowable(e.getCause());
        }
    }
}
~~~
- 이미지를 다운로드받는 작업을 생성하여 Executor를 활용해 다운로드 작업을 실행
	- 순서대로 다운로드하던 부분을 병렬화 함 --> 다운로드 시간을 줄임
- 다운로드 받은 이미지는 CompletionService를 통해 찾아가도록하여, 다운로드 즉시 해당 위치에 그림을 그림
- 사용자 입장에서는 동적으로 빠르게 업데이트되는 페이지를 볼 수 있음
- 실행을 맡은 Executor는 하나만 두고 동일한 작업을 처리하는 여러개의 ExecutorCompetionService를 생성해 사용가능 함
- Future가 단일 작업 하나에 대한 진행 과정을 관리한다고 할 때, CompletionService는 특정한 배치 작업을 관리하는 모습을 띤다고 볼 수 있음


### 6.3.7 작업 실행 시간 제한
- 타임아웃을 지정할 수 있는 Future.get 메소드를 사용하면 시간 제한 요구사항을 만족 할 수 있음
- 결과가 나오는 즉시 리턴되는 것은 타임아웃을 지정하지 않는 경우와 같지만, 지정한 시간이 지나도 결과를 만들어 내지 못하면 TimeoutException을 던지면서 실행이 멈추게 되어있음
- 제한된 시간을 넘었을 때 해당 작업을 실제로 멈추도록하여 시스템 자원을 소모하지 않도록 해야함
	- 제한된 시간이 되면 스스로 작동을 멈추게 하거나 강제로 취소 시키는 방법이 존재
- Future를 사용하여, 시간 제한을 걸어둔 get 메소드에서 TimeoutException이 발생하면 해당 Future 작업을 직접 취소 시킬 수 있음
~~~java
    Page renderPageWithAd() throws InterruptedException {
        long endNanos = System.nanoTime() + TIME_BUDGET;
        Future<Ad> f = exec.submit(new FetchAdTask());

        // 광고 가져오는 작업을 등록했으니, 원래 페이지를 작업한다
        Page page = renderPageBody();
        Ad ad;

        try {
            // 남은 시간 만큼만 대기한다
            long timeLeft = endNanos - System.nanoTime();
            ad = f.get(timeLeft, NANOSECONDS);
        } catch (ExecutionException e) {
            ad = DEFAULT_AD;
        } catch (TimeoutException e) {
            ad = DEFAULT_AD;
            f.cancel(true);
        }
        
        page.setAd(ad);
        return page;
    }
~~~
- 프로그램은 요청한 실제 내용과 독립 서버에서 내용을 받아오는 광고를 묶은 웹 페이지를 하나 생성하는 예제
- 제한 시간이 지나버리면 광고를 가져오는 작업을 취소시키고, 가져오려 했던 광고 대신 기본 광고를 사용함


### 6.3.8 예제: 여행 예약 포털
- 업체별로 입찰 정보를 가져오는 작업은 업체를 단위로 완전히 독립적인 작업
- 입찰 정보를 가져오는 작업을 병렬로 처리할 수 있음
- 입찰 정보를 가져오는 작업 N개를 생성해 스레드 풀에 등록하고, 등록한 작업마다 Future 객체를 확보하여 타임 아웃을 지정한 get 메소드로 각 정도를 가져오도록 함
	- invokeAll 메소드를 통해 쉽게 작업 가능
~~~java
private class QuoteTask implements Callable<TravelQuote> {
    private final TravelCompany company;
    private final TravelInfo travelInfo;
...

    public TravelQuote call() throws Exception {
        return company.solicitQuote(travelInfo);
    }
}

public List<TravelQuote> getRankedTravelQuotes(
        TravelInfo travelInfo, Set<TravelCompany> companies,
        Comparator<TravelQuote> ranking, long time, TimeUnit unit)
        throws InterruptedException {
    List<QuoteTask> tasks = new ArrayList<QuoteTask>();
    for (TravelCompany company : companies)
        tasks.add(new QuoteTask(company, travelInfo));
    List<Future<TravelQuote>> futures =
            exec.invokeAll(tasks, time, unit);
    List<TravelQuote> quotes =
            new ArrayList<TravelQuote>(tasks.size());
    Iterator<QuoteTask> taskIter = tasks.iterator();
    for (Future<TravelQuote> f : futures) {
        QuoteTask task = taskIter.next();
        try {
            quotes.add(f.get());
        } catch (ExecutionException e) {
            quotes.add(task.getFailureQuote(e.getCause()));
        } catch (CancellationException e) {
            quotes.add(task.getTimeoutQuote(e));
        }
    }
    Collections.sort(quotes, ranking);
    return quotes;
}
~~~
- 결과를 받아오는 부분에 타임아웃을 지정한 invokeAll 메소드를 활용
- invokeAll 메소드는 작업 객체가 담긴 컬렉션 객체를 넘겨받고, Future 객체가 담긴 컬렉션 객체를 리턴함
- invokeAll 메소드는 넘겨받은 작업 컬렉션의 iterator가 뽑아주는 순서에 따라 결과 컬렉션에 Future 객체를 쌓음
- 시간 제한이 있는 invokeAll 메소드는 등록된 모든 작업이 완료됐거나, 작업을 등록한 스레드에 인터럽트가 걸리거나, 제한 시간이 지날때까지 대기하다가 리턴됨
- invokeAll 메소드가 리턴되면 등록된 모든 작업은 완료되어 결과값을 가지고 있거나 취소되거나 2가지 상태만 가짐
- 작업을 등록했던 스레드는 모든 작업을 대상으로 get() 또는 isCancelled()를 통해 작업 상태를 확인할 수 있음

</br>

## 요약
- 애플리케이션을 작업이라는 단위로 구분해 실행 할 수 있도록 구조를 잡으면 개발 과정을 간소화하고 병렬성을 확보할 수 있음
- Executor 프레임웍을 사용하면 작업을 생성하는 부분과 작업을 실행하는 부분을 분리하여 정책을 수립할 수 있고, 원하는 형태의 정책을 만들어 사용할 수 있음
- 작업을 처리하는 부분에서 스레드를 생성하도록 되어있다면, 스레드를 직접 사용하는 것 대신 Exector를 사용하도록 권장
- 애플리케이션에서는 일반적인 작업 범위가 잘 적용되지만, 일부 애플리케이션에서는 스레드를 사용해 병렬로 처리시킨 이득을 보려면 약간의 분석을 통해 병렬로 처리할 작업을 찾아낼 필요가 있음