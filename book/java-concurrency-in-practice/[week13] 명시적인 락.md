# 명시적인 락
- ReentranLock은 암묵적인 락으로 할 수 없는 일도 처리할 수 있도록 고급 기능을 갖고 있음

## 13.1 Lock과 ReetrantLock
~~~java
public interface Lock {
	void lock();
	void lockInterruptibly() throws InterruptedException;
	boolean tryLock();
	boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
	void unlock();
	Condition newCondition();
}
~~~
- Lock 인터페이스는 여러가지 락 관련 기능에 대한 추상 메소드를 정의하고 있음
	- 조건 없는 락, 폴링 락, 타임아웃이 있는 락, 락 확보 대기 상태에 인터럽트를 걸 수 있는 방법이 포함되어 있음
	- 락을 확보하고 해제하는 모든 작업이 명시적임
- ReetrantLock 클래스 역시 Lock 인터페이스를 구현함
	- synchronized 구문과 동일한 메모리 가시성과 상호 배제 기능을 제공
- ReetrantLock는 synchronized 키워드와 동일하게 재진입이 가능하도록 허용함
- ReetrantLock는 Lock에 정의돼 있는 락 확보 방법을 모두 지원함
	- 락을 제대로 확보하기 어려운 시점에 synchronized 블록을 사용할 때보다 훨씬 능동적으로 대처할 수 있음
- 암묵적인 락만 사용해도 대부분의 경우에 별 문제 없이 사용할 수 있지만 기능적으로 제한되는 경우가 발생함
	- 락을 확보하고자 대기하고 있는 상태의 스레드에는 인터럽트를 거는 일이 불가능하고, 대기 상태에 들어가지 않으면서 락을 확보하는 방법 등이 필요한 상황이 있기 때문
- 암묵적인 락은 synchronized 블록이 끝나는 시점에 해제되어 이런 구조는 코딩하기에 간편하고 예외 처리 루틴과 잘 맞아 떨어지는 구조이긴 하지만 블록의 구조를 갖추지 않은 상황에서 락을 걸어야 하는 경우 적용이 불가능함
~~~java
Lock lock = new ReentrantLock();
// ...
lock.lock();
try {
    //객체 내부 값을 사용
    // 예외가 발생한 경우, 적절하게 내부값을 복원해야 할 수도 있음
} finally {
    lock.unlock();
}
~~~
- finally 블록에서 반드시 락을 해제해야 함
	- 예외 때문에 해당 객체가 불안정한 상태가 될 수 있다면 try-catch 구문이나 try-finally 구문을 추가로 지정해 안정적인 상태를 유지하도록 해야함
- 락을 해제하는 기능을 finally 구문에 넣어두지 않은 코드는 언제 터질지 모르는 시한폭탄과 같음
- ReetrantLock을 사용하면 해당하는 블록의 실행이 끝나고 통제권이 해당 블록을 떠나는 순간 락을 자동으로 해제하지 않기 때문에 굉장히 위험한 코드가 될 가능성이 높음

</br>

### 13.1.1 폴링과 시간 제한이 있는 락 확보 방법
- tryLock 메소드가 지원하는 폴링 락 확보 방법이나 시간 제한이 있는 락 확보 방법은 오류가 발생했을 때 무조건적으로 락을 확보하는 방법보다 오류를 잡아내기에 훨씬 깔끔한 방법
- 락을 확보할 때 시간 제한을 두거나 폴링하도록 하면 다른 방법, 즉 확률적으로 데드락을 회피할 수 있는 방법을 사용할 수 있음
- 락을 확보할 때 시간 제한을 두거나 폴링 방법(tryLock)을 사용하면 락을 확보하지 못하는 상황에도 통제권을 다시 얻을 수 있으며, 그러면 미리 확보하고 있던 락을 해제하는 등의 작업을 처리한 이후 다시 락을 재시도 할 수 있음
~~~java
public boolean transferMoney(Account fromAcct, Account toAcct, DollarAmount amount, long timeout, TimeUnit unit) throws InsufficientFundsException, InterruptedException {
    long fixedDelay = getFixedDelayComponentNanos(timeout, unit);
    long randMod = getRandomDelayModulusNanos(timeout, unit);
    long stopTime = System.nanoTime() + unit.toNanos(timeout);
    
    while (true) {
        if(fromAcct.lock.tryLock()) {
            try {
                if(toAcct.lock.tryLock()) {
                    try {
                        if(fromAcct.getBalance().compareTo(amount) < 0) {
                            throw new InsufficientFundsException();
                        } else {
                            fromAcct.debit(amount);
                            toAcct.credit(amount);
                            return true;
                        }
                    } finally {
                        toAcct.lock.unlock();
                    }
                }
            } finally {
                fromAcct.lock.unlock();
            }
        }
        
        if(System.nanoTime() >= stopTime) {
            return false;
        }
        NANOSECONDS.sleep(fixedDelay + rnd.nextLock() % randMod);
    }
}
~~~
- tryLock 메소드로 양쪽 락을 모두 확보하도록 돼 있지만 만약 양쪽 모두 확보할 수 없다면 잠시 대기했다가 재시도하도록 되어있음
- 대기하는 시간 간격은 라이브락이 발생할 확률을 최대한 줄일수 있도록 고정된 시간 또는 임의의 시간만큼 대기
- 만약 지정된 시간 이내에 락을 확보하지 못했다면 transferMoney 메소드는 오류가 발생했다는 정보를 리턴해주고 적절한 통제하에서 오류를 처리할 수 있음
~~~java
public boolean trySendOnSharedLine(String message, 
								   long timeout, 
								   TimeUnit unit) throws InterruptedException {
    long nanosToLock = unit.toNanos(timeout) - estimatedNanosToSend(message);
    if(!lock.tryLock(nanosToLock, TimeUnit.NANOSECONDS))
        return false;
    try {
        return sendOnSharedLine(message);
    } finally {
        lock.unlock();
    }
}
~~~
- 일정 시간 내에 작업을 처리하지 못하면 무리없이 적절한 방법으로 오류를 처리
	- tryLock 메소드에 타임아웃을 지정해 사용하면 시간이 제한된 작업 구조에 락을 함께 적용해 활용하기 좋음


### 13.1.2 인터럽트 걸 수 있는 락 확보 방법
- 일정 시간 안에 처리해야 하는 작업을 실행하고 있을 때 타임아웃을 걸 수 있는 락 확보 방법을 유용하게 사용할 수 있는 것처럼, 작업 도중 취소시킬 수 있어야 하는 작업인 경우에는 인터럽트를 걸 수 있는 락 확보 방법을 유용하게 사용할 수 있음
- lockInterruptibly 메소드를 사용하면 인터럽트는 그대로 처리할 수 있는 상태에서 락을 확보함
	- Lock 인터페이스에 lockInterruptibly 메소드를 포함하고 있기 때문에 인터럽트에 반응하지 않는 다른 종류의 블로킹 구조를 만들어야 할 필요가 없음
- 인터럽트에 대응할 수 있는 방법으로 락을 확보하는 코드의 구조는 일반적으로 락을 확보하는 모습보다 약간 복잡하지만 두 개의 try 구문을 사용해야 함
~~~java
public boolean sendOnSharedLine(String message) throws InterruptedException {
    lock.lockInterruptibly();
    try {
        return cancellableSendOnSharedLine(message);
    } finally {
        lock.unlock();
    }
}

private boolean cancellableSendOnSharedLine(String message) throws InterruptedException { ... }
~~~
- lockInterruptibly를 사용해 13.4에서 구현했던 sendOnSharedLine 메소드를 구현했으며 취소 가능한 작업으로 실행됨
- 타임아웃을 지정하는 tryLock 메소드 역시 인터럽트를 걸면 반응하도록 돼 잇으며, 인터럽트를 걸어 취소시킬수도 있어야 하면서 동시에 타임아웃을 지정할 수 있어야 한다면 tryLock을 사용하는 것만으로 충분함


### 13.1.3 블록을 벗어나는 구조의 락
- 암묵적인 락을 사용하는 경우에는 락을 확보하고 해제하는 부분이 완벽하게 블록의 구조에 맞춰져 있으며, 블록이 어떤 상태로 떠나는지에 관계없이 락은 항상 자신을 확보했던 블록이 끝나는 시점에 자동으로 해제됨
- 락을 적용하는 코드를 세분화할수록 애플리케이션의 확장성이 높아짐 (11장)
	- 락 스트라이핑 방법을 적용하면 해시기반의 컬렉션 클래스에서 여러 개의 해시 블록을 구성해 블록별로 다른 락을 사용함
	- 연결 리스트 역시 해시 컬렉션과 마찬가지로 락을 세분화 할 수 있음
		- 각각의 개별 노드마다 서로 다른 락을 적용

</br>

## 13.2 성능에 대한 고려 사항
- 여러가지 동기화 기법에 있어서 경쟁 성능은 확장성을 높이는데 가장 중요한 요소
- 락과 그에 관련된 스케쥴링을 관리하느라 컴퓨터의 자원을 많이 소모하면 할수록 실제 애플리케이션이 사용할 수 있는 자원은 줄어들 수 밖에 없음
잘 만들어진 동기화 기법일수록 시스템 호출을 더 적에 사용하고 컨텍스트 스위치 횟수를 줄이고 공유된 메모리 버스에 메모리 동기화 트래픽을 덜 사용하도록 하고, 시간을 많이 소모하는 작업을 줄여주며 연산 자원을 프로그램에서 우회시킬수 있음 

![ReentrantLock01](/img/ReentrantLock01.png)     
  
- 솔라리스 운영체제가 설치된 옵테론 프로세서 4개가 장착된 하드웨어에서 자바 5.0과 자바 6에서 각각 암묵적인 락과 ReentrantLock의 성능을 비교한 결과
- 특정 자바 버전에서 암묵적인 락에 비해 ReentrantLock의 성능이 얼마나 좋아졌는지 보여줌
- 자바 5.0에서는 암묵적인 락을 사용할 때 스레드 수가 1일때 보다 스레드 개수가 늘어나면 성능이 크게 떨어짐
	- ReentrantLock을 사용하면 성능이 떨어지는 정도가 훨씬 덜하며, 따라서 확장성이 더 낫다고 볼 수 있음
- 자바 6에서는 암묵적인 락을 사용했다 해도 스레드 간의 경쟁이 있는 상황에서 성능이 그다지 떨어지지 않고 ReentrantLock을 사용할 때와 별반 차이가 없음
- 성능과 확장성은 모두 CPU의 종류, 개수, 캐시의 크기, JVM의 여러가지 특성 등에 따라 굉장히 민감하게 바뀌기 때문에 성능과 확장성에 영향을 주는 여러가지 요인은 시간이 지나면서 계속해서 바뀌기 마련
- 성능 측정 결과는 움직이는 대상
	- 어제 X가 Y보다 빠르다는 결과를 산출했던 성능 테스트를 오늘 테스트해보면 다른 결과를 얻을 수 있음

</br>

## 13.3 공정성
- ReentrantLock 클래스는 두 종류의 공정성 설정을 지원
	- 불공정 락 방법 (default)
	- 공정한 방법
- 공정한 방법을 사용할 때는 요청한 순서를 지켜가면서 락을 확보
- 불공정한 방법을 사용하는 경우에는 순서 뛰어넘기가 일어나기도 하는데, 락을 확보하려고 대기하는 큐에 대기중인 스레드가 있다 하더라도 해제된 락이 있으면 대기자 목록을 뛰어 넘어 락을 확보할 수 있음
- 불공정한 ReentrantLock이 일부러 순서를 뛰어넘도록 하지는 않으며, 딱 맞는 타이밍에 락이 해제된다 해도 큐의 뒤쪽에 있어야 할 스레드가 순서를 뛰어넘지 못하게 제한하지 않을 뿐임
- 공정한 방법을 사용하면 확보하려는 락을 다른 스레드가 사용하고 있거나 동일한 락을 확보하려는 스레드가 큐에 대기하고 있으면 항상 큐의 맨 뒤에 쌓임 
- 불공정한 방법이라면 락이 당장 사용 중인 경우에만 큐의 대기자 목록에 들어감
- 락을 관리하는 입장에서 봤을때 공정하게만 처리하다 보면 스레드를 멈추고 다시 실행시키는 동안 성능에 큰 지장을 줄 수 있음
	- 실제로 통계적인 공정함 정도만으로도 충분히 괜찮은 결과를 얻을 수 있고, 그와 더불어 성능에도 훨씬 악영향이 적음
- 대부분의 경우 공정하게 순서를 관리해서 얻는 장점보다 불공정하게 처리해서 얻는 성능상의 이점이 큼
  
![ReentrantLock02](/img/ReentrantLock02.png)    
  
- HashMap을 놓고 공정한 락과 불공정한 락을 사용한 결과를 측정함
- 스레드 간의 경쟁이 심하게 나타나는 상황에서 락을 공정하게 관리하는 것보다 불공정하게 관리하는 방법의 성능이 훨씬 빠름
	- 대기 상태에 있던 스레드가 다시 실행 상태로 돌아가고 또한 실제로 실행되기까지는 상당한 시간이 걸리기 때문
- 예를 들어 스레드 A가 락을 확보하고 있는 상태에서 스레드 B가 락을 확보하고자 하는 경우
	- 락은 현재 누군가가 사용하고 있기 때문에 스레드 B는 일단 대기 상태에 들어감
	- 스레드 A가 락을 해제하면 스레드 B가 대기 상태가 풀리면서 다시 락을 확보하고자 요청함
	- 동시에 스레드 C가 끼어들면서 동일한 락을 확보하고자 한다면 스레드 B대신 스레드 C가 락을 미리 확보해버릴 확률이 꽤 높고, 스레드 B가 본격적으로 실행되기 전에 스레드 C는 이미 실행을 마치고 락을 다시 해제시키는 경우도 가능함
	- 이런 경우라면 모든 스레드가 원하는 성능을 충분히 발휘하면서 실행된 셈
	- 스레드 B는 사용할 수 있는 시점에 락을 확보할 수 있고, 스레드 C는 이보다 먼저 락을 사용할 수 있으니 처리량이 크게 늘어남
- 공정한 방법으로 락을 관리할 때는 락을 확보하고 사용하는 시간이 상대적으로 길거나 락 요청이 발생하는 시간 간격이 긴 경우에 유리
- 락 사용시간이 길거나 요청 간의 시간 간격이 길면 순서 뛰어넘기 방법으로 성능상의 이득을 얻을 수 있는 상태, 즉 락이 해제돼 있는 상태에서 다른 스레드가 해당 락을 확보하고자 대기 상태에서 깨어나고 있는 상태가 상대적으로 훨씬 덜 발생하기 때문
- 기본 ReentrantLock과 같이 암묵적인 락 역시 공정성에 대해 아무런 보장을 하지 않음
	- 통계적으로 공정하다는 사실을 놓고 보면 대부분의 락 구현 방법을 거의 모든 상황에 무리 없이 적용할 수 있음

</br>

## 13.4 synchronized 또는 ReetrantLock 선택
- ReetrantLock은 락을 확보할 때 타임아웃을 지정하거나 대기 상태에서 인터럽트에 잘 반응하고 공정성 여부를 지정할 수 있음
	- 블록의 구조를 갖추고 있지 않은 경우에도 락을 적용할 수 있는 유연함을 갖고 있음
- 암묵적인 락은 명시적인 락에 비해 상당한 장점이 있음
	- 코드에 나타나는 표현 방법도 훨씬 익숙하고 간결
	- 암묵적인 락과 명시적인 락을 섞어 쓴다고 하면 코드를 읽을 때 혼동될 뿐만 아니라 오류가 발생할 가능성도 높음
- ReetrantLock은 암묵적인 락에 비해 더 위험할 수도 있음
	- finally 블록에 unlock 메소드를 넣지 않는다면.. 시한폭탄을 심어두는 셈
- ReetrantLock은 synchronized 블록에서 제공하지 않는 특별한 기능이 꼭 필요할 때만 사용하는게 안전함
- ReetrantLock은 암묵적인 락만으로는 해결할 수 없는 복잡한 상황에서 사용하기 위한 고급 동기화 기능
- 고급 동기화 기법을 사용해야 하는 경우에만 ReetrantLock을 사용하도록 함
	- 락을 확보할 때 타임아웃을 지정해야 하는 경우
	- 폴링의 형태로 락을 확보하고자 하는 경우
	- 락을 확보하느라 대기 상태에 들어가 있을 때 인터럽트를 걸 수 있어야 하는 경우
	- 대기 상태 큐 처리 방법을 공정하게 해야 하는 경우
	- 코드가 단일 블록의 형태를 넘어서는 경우
- 암묵적인 락을 사용할 때는 항상 특정 스택 프레임에 락이 연관돼 있었지만, ReetrantLock은 블록을 벗어나는 범위에도 사용할 수 있기 때문에 특정 스택 프레임에 연결되지 않음
- 단순히 성능이 나아지기를 기대하면서 synchronized 대신 ReetrantLock을 사용하는 일은 그다지 좋은 선택이 아님

</br>

## 13.5 읽기-쓰기 락
- ReentrantLock은 표준적인 상호 배제(mutual exclusion) 락을 구현하고 있음
	- 한 시점에 단 하나의 스레드만이 락을 확보할 수 있음
- 상호 배제 규칙은 일반적으로 데이터의 완전성을 보장하는데 굉장히 엄격한 특징을 갖고 있음
	- 필요 이상으로 병렬프로그램의 장점을 제한함
- 상호 배제 규칙은 쓰기 연산과 쓰기 연산이 동시에 일어나거나 쓰기와 읽기 연산이 동시에 일어나는 경우를 제한할 뿐만 아니라 읽기와 읽기 연산이 동시에 일어나는 경우도 제한함
- 대부분의 경우 사용하는 데이터 구조는 읽기 작업이 많이 일어남
- 읽기 작업은 여러 개를 한꺼번에 처리할 수 있지만, 쓰기 작업은 혼자만 동작할 수 있는 구조의 동기화를 처리해주는 락이 바로 읽기-쓰기 락(read-write lock)
~~~java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
~~~
- ReadWriteLock으로 동기화된 데이터를 읽어가려면 읽기 락을 확보해야 하고, 해당 데이터를 변경하고자 한다면 쓰기 락을 확보해야 함
- ReadWriteLock에서 구현하고 있는 동기화 정책은 여러 개의 읽기 작업은 동시에 처리할 수 있지만, 쓰기 작업은 한 번에 단 하나만 동작 할 수 있음
- 성능, 스케줄링 특성, 락 확보 방법의 특성, 공정성 문제, 기타 다른 락 관련 의미가 서로 다르게 반영되도록 새로운 클래스를 구현할 수 있게 되어 있음
- 특정 상황에서 병렬 프로그램의 성능을 크게 높일 수 있도록 최적화된 형태로 설계된 락
- 구현상의 복잡도가 약간 높기 때문에 최적화된 상황이 아닌 곳에 적용하면 상호 배제시키는 일반적인 락에 비해 성능이 약간 떨어짐
- 읽기 락과 쓰기 락 간의 상호작용을 잘 활용하면 여러가지 특성을 갖는 다양한 ReadWriteLock을 구현할 수 있음
- ReadWriteLock을 구현할 때 적용할 수 있는 특성에는 아래와 같은 것이 있음
	- **락 해제 방법**
		- 쓰기 작업에서 락을 해제했을 때 대기 큐에 읽기 작업뿐만 아니라 쓰기 작업도 대기중이었다고 하면 누구에게 락을 먼저 넘겨줄것인가의 문제, 읽기 작업을 먼저할 것인지, 쓰기 작업을 먼저할 것인지, 아니면 큐에 먼저 대기하고 있던 작업을 먼저 처리하도록 해도 좋음
	- **읽기 순서 뛰어넘기**
		- 읽기 작업에서 락을 사용하고 있고 대기 큐에 쓰기 작업이 대기하고 있다면 읽기 작업이 추가로 실행됐을 때 읽기 작업을 그냥 실행할 것인지? 아니면 대기 큐의 쓰기 작업 뒤에서 대기할지 정할 수 있음
	- **재진입 특성**
		- 읽기 쓰기 작업 모두 재진입이 가능한지?
	- **다운그레이드**
		- 특정 스레드에서 쓰기 락을 확보하고 있을 때, 쓰기 락을 해제하지 않고도 읽기 락을 확보할 수 있는지? 
		가능하다면 쓰기 작업을 하던 스레드가 읽기 락을 직접 확보하고 읽기 작업을 할 수 있음.
		즉 읽기 락을 확보하려는 사이에 다른 쓰기 작업이 실행되지 못하게 할 수 있음
	- **업그레이드**
		- 읽기 락을 확보하고 있는 상태에서 쓰기 락을 확보하고자 할 때 대기 큐에 들어 있는 다른 스레드보다 먼저 쓰기 락을 확보하게 할 것인지? 직접적인 업그레이드 연산을 제공하지 않는 한 자동으로 업그레이드가 일어나면 데드락의 위험이 높기 때문에 ReadWriteLock 을 구현하는 대부분의 경우에 업그레이드를 지원하지 않음
- ReentrantReadWriteLock 클래스를 사용하면 읽기 락과 쓰기 락 모두에게 재진입 락 기능을 제공함
- 공정하게 락을 설정하는 경우에는 대기 큐에서 대기한 시간이 가장 긴 스레드에게 우선권이 돌아감
	- 즉 읽기 락을 확보하고 있는 상태에서 다른 스레드가 쓰기 락을 요청하는 경우, 쓰기 락을 요청한 스레드가 쓰기 락을 확보하고 해제하기 전에는 다른 스레드에서 읽기 락을 가져가지 못함
- 쓰기 락을 확보한 상태에서 읽기 락을 사용하는 다운그레이드는 허용되며, 읽기 락을 확보한 상태에서 쓰기 락을 사용하는 업그레이드는 제한됨
- Reentrantlock과 동일하게 ReentrantReadWriteLock 역시 쓰기 락을 확보한 스레드가 명확하게 존재하며, 쓰기 락을 확보한 스레드만이 쓰기 락을 해제 할 수 있음
- 읽기-쓰기 락은 락을 확보하는 시간이 약간은 길면서 쓰기 락을 요청하는 경우가 적을 때에 병렬성을 크게 높여줌
~~~java
public class ReadWriteMap<K, V> {
    private final Map<K,V> map;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock r = lock.readLock();
    private final Lock w = lock.writeLock();
    
    public ReadWriteMap(Map<K, V> map) {
        this.map = map;
    }
    
    public V put(K key, V value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    // remove(), putAll(), clear() 메소드도 put()과 동일하게 구현
    
    public V get(Object key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    // 다른 읽기 메소드도 get()과 동일하게 구현
}
~~~
- ReadWriteMap 클래스는 ReentrantReadWriteLock을 사용해 Map에 대한 접근을 제한함
- ReadWriteLock을 사용하기 때문에 Map의 값을 읽어가는 작업은 얼마든지 한꺼번에 실행될 수 있지만 실제로 병렬로 사용할 수 있는 해시 기반의 Map클래스가 필요했던 것이라면 ReadWriteMap을 사용하는 것보다 ConcurrentHashMap만 사용해도 충분히 괜찮은 성능을 낼 수 있음
- 여러 스레드에서 동시에 사용해야 하지만 ConcurrentHashMap과 약간 다른 LinkedHashMap같은 기능이 필요하다면 ReadWriteLock을 사용해 필요한 만큼의 성능을 뽑아낼 수 있음

![read-write-lock](/img/read-write-lock.png)     
   
- ArrayList를 ReentrantLock으로 동기화시킨 클래스와 ReadWriteLock으로 동기화시킨 클래스의 실행 속도를 비교하고 있음

</br>

## 요약
- 명시적으로 Lock 클래스를 사용해 스레드를 동기화하면 암묵적인 락보다 더 많은 기능을 활용할 수 있음
	- 예를 들어 락을 확보할 수 없는 상황에 유연하게 대처하는 방법이나 대기 큐에서 기다리는 방법과 규칙도 원하는대로 정할 수 있음
- ReentrantLock에서만 제공되고 synchronized 구문은 제공하지 않는 동기화 관련 기능이 꼭 필요한 경우에만 ReentrantLock을 사용하도록 권장
- 읽기-쓰기 락을 사용하면 읽기 작업만 처리하는 다수의 스레드는 동기화된 값을 얼마든지 동시에 읽어갈 수 있음
- 읽기 작업이 대부분인 데이터 구조에 읽기-쓰기 락을 사용하면 확장성을 높여주는 훌륭한 도구가 됨