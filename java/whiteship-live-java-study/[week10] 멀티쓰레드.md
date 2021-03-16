# 학습내용
- Thread 클래스와 Runnable 인터페이스
- 쓰레드의 상태
- 쓰레드의 우선순위
- Main 쓰레드
- 동기화
- 데드락

</br>

## Thread 클래스와 Runnable 인터페이스
- Thread란?
  - 프로세스가 할당받은 자원을 이용하는 실행 단위
  - 즉, 프로그램이 실행된 상태가 프로세스이며, 프로세스의 실행 단위가 쓰레드
  - 하나의 프로세스는 하나 이상의 쓰레드를 가짐
- 자바에서 쓰레드를 생성하는 방법은 Thread 클래스를 사용, Runnable 인터페이스를 구현하는 방법이 존재
- Thread 클래스
  - Runnable 인터페이스를 상속받아 구현되어 있음
  - Thread 클래스를 상속받아 run 메소드를 오버라이딩하여 사용
  ~~~java
  class Example {
      public static void main(String[] args) throws InterruptedException {
          Thread aThread = new AThread();
          aThread.start();

          Thread bThread = new BThread();
          bThread.start();
      }
  }


  class AThread extends Thread {
      @Override
      public void run() {
          for (int i = 0; i < 5; i++) {
              System.out.println("running AThread : "+i);
          }
      }
  }

  class BThread extends Thread {
      @Override
      public void run() {
          for (int i = 0; i < 5; i++) {
              System.out.println("running BThread : "+i);
          }
      }
  }
  ~~~
  [결과]
  ~~~java
  running AThread : 0
  running BThread : 0
  running AThread : 1
  running BThread : 1
  running AThread : 2
  running BThread : 2
  running AThread : 3
  running BThread : 3
  running AThread : 4
  running BThread : 4
  ~~~
- Runnable Interface
  - 인터페이스이기 때문에, 다중 상속이 가능하며 구현클래스를 통해 구체화 됨
  - Runnable 인터페이스를 상속받아 구현한 객체를 호출하여 쓰레드를 생성
  ~~~java
  class Example {
      public static void main(String[] args) throws InterruptedException {
          Runnable aRunnable = new ARunnable();
          Thread aThread = new Thread(aRunnable);
          aThread.start();

          Runnable bRunnable = new BRunnable();
          Thread bThread = new Thread(bRunnable);
          bThread.start();
      }
  }


  class ARunnable implements Runnable {
      @Override
      public void run() {
          for (int i = 0; i < 5; i++) {
              System.out.println("running AThread : "+i);
          }
      }
  }

  class BRunnable implements Runnable {
      @Override
      public void run() {
          for (int i = 0; i < 5; i++) {
              System.out.println("running BThread : "+i);
          }
      }
  }
  ~~~

</br>

## 쓰레드의 상태
- 사용자 입장에서의 쓰레드 시작은 start() 메소드를 사용
- 실제 쓰레드의 순서는 아래와 같음
  ![java-thread-life-cycle](/img/java-thread-life-cycle.png)
  출처 : http://www.btechsmartclass.com/java/java_images/java-thread-life-cycle.png
- start()
  - 쓰레드 실행 (쓰레드 실행대기 상태에서 OS 스케쥴러에 의해 차례가 돌아오면 실행됨)
- start()와 run() 차이
  - run()을 호출하는 것은 생성된 쓰레드를 실행하는 것이 아닌 오버라이딩 된 run() 메소드를 실행하는 것
  - start()를 호출해야 쓰레드가 실행됨
- 실행이 종료된 쓰레드는 재실행할 수 없음
  - 재실행 시, IllegalThreadStateException 발생
  
</br>

## 쓰레드의 우선순위
- 2개 이상의 쓰레드가 실행중인 경우, 쓰레드에 우선순위를 부여하여 높은 우선순위의 쓰레드가 먼저 실행하도록 우선권을 부여할 수 있음
- 우선순위 지정을 위한 필드/메소드
  ~~~java
  public final static int MIN_PRIORITY = 1;  //  우선순위 최소값

  public final static int NORM_PRIORITY = 5;  // 기본 우선순위값

  public final static int MAX_PRIORITY = 10;  // 우선순위 최대값

  setPriority(int newPriority); // 쓰레드 우선순위를 지정한 값으로 변경
  getPriority();  // 쓰레드 우선순위 반환
  ~~~
- 우선순위 지정을 위한 값의 범위는 1~10 사이
- main 메소드를 실행하는 쓰레드의 우선순위는 5 (NORM_PRIORITY)

- Thread가 관리되는 JVM 스케쥴링 규칙
  - 라운드 로빈
  - 철저한 우선순위 기반
  - 가장 높은 우선순위의 스레드가 우선적으로 스케쥴링
  - 동일한 우선순위의 스레드는 돌아가면서 스케쥴링
</br>

## Main 쓰레드
- java 어플리케이션에서 main 쓰레드는 main() 메소드를 통해서 실행
- main() 메소드의 return 또는 메소드의 끝이 오면 main 쓰레드는 종료됨
- main 쓰레드가 종료되더라도, 실행중인 쓰레드가 있다면 모든 쓰레드가 종료되기 전까지 어플리케이션은 종료되지 않음

- Daemon 쓰레드
  - 사용자 쓰레드의 작업을 돕는 보조 역할의 쓰레드
  - 보조 역할의 쓰레드이므로, 사용자 쓰레드가 종료되면 daemon 쓰레드도 종료됨
  - setDaemon(), isDaemon() 메소드를 통해 daemon 쓰레드를 지정하고 확인할 수 있음
  - 대표적인 예로 GC가 있음

</br>

## 동기화
- 싱글 쓰레드의 경우, 하나의 쓰레드로만 작업하기 때문에 프로세스 자원의 문제가 없음
- 멀티 쓰레드의 경우, 프로세스 자원을 공유하기 때문에 서로 작업에 문제가 발생 할 수 있음
  - 예) a 쓰레드에서 count=1로 변경, 동시에 b 쓰레드에서 count=2로 변경 ==> count 값을 보장할 수 없다!
- 여러 쓰레드가 하나의 자원을 사용하려할 때, 하나의 쓰레드만 허용하고 나머지 쓰레드는 해당 자원에 접근하지 못하도록 하는 것을 동기화라고 함
- Thread Safe라 부름
- 동기화하는 방법
  - Synchronized 키워드
  - Atomic 클래스
  - Volatile 키워드
- Synchronized 키워드
  - 자바 예약어 중 하나
  - 메소드에 synchronized 키워드를 붙이는 방법과 특정 문장만 synchronized로 감싸는 방법이 존재
  - 메소드에 synchronized 키워드를 붙이는 방법
  ~~~java
  public synchronized void calculator() {
      ....
  }
  ~~~
  - 특정 문장만 synchronized로 감싸는 방법
  ~~~java
  synchronized (객체의 참조변수) {
      ....
  }
  ~~~
- Atomic 클래스
  - atomic : 쪼갤 수 없는 가장 작은 단위
  - Wrapping 클래스의 일종으로, 개발자의 별다른 처리없이 동기화를 할 수 있음
  - AtomicInteger, AtomicBoolean, AtomicLong 등이 존재
- Volatile 키워드
  - java 변수를 main 메모리에 저장하겠다는 의미
  - 변수값을 읽고 쓸때마다 캐시가 아닌 main 메모리에 읽고 쓰는 형태

</br>

## 데드락
- 데드락, 즉 교착상태
- 둘 이상의 쓰레드가 lock을 획득하기 위해 대기하는데, 이 lock을 잡고 있는 쓰레드들도 똑같이 다른 lock을 기다리면서 서로 block 상태에 놓이는 것
- 데드락에 빠진 쓰레드는 서로가 데드락에 빠진줄 모르기 때문에 영원히 이상태를 유지하게 됨
