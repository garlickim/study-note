# 활동성을 최대로 높이기
- 스레드 안전성을 확보하기 위해 락을 사용함
	- 락이 우연찮게 일정한 순서로 동작하다 보면 락 순서에 따라 데드락이 발생하기도 함
- 시스템 자원 사용량을 적절한 수준에서 제한하고자 할 때 스레드 풀이나 세마포어를 사용하기도 함
	- 스레드 풀이나 세마포어가 동작하는 구조를 정확하게 이해하지 못하면 더 이상 자원을 할당받지 못하는 다른 형태의 데드락이 발생할 수 있음

</br>

## 10.1 데드락
- 스레드 하나가 특정 락을 놓지 않고 계속해서 잡고 있으면 그 락을 확보해야 하는 다른 스레드는 락이 풀리기를 영원히 기다리는 수 밖에 없음
- 여러 개의 스레드가 사이클을 이루면서 상대방이 확보한 락을 얻으려 대기하는 상태의 데드락은 자주 발생함
- 데이터베이스 시스템은 데드락을 검출한 다음 데드락 상황에서 복구하는 기능을 갖추고 있음
	- 데이터베이스의 트랜잭션을 활용하다보면 여러 개의 락이 필요할 수 있으며, 락은 해당 트랜잭션이 커밋 될 때까지 풀리지 않음 
	- 흔하지는 않지만 두 개 이상의 트랜잭션이 데드락 상태에 빠지는 일이 충분히 가능함
	- 외부에서 조치가 없다면 해당 트랜잭션은 영원이 끝나지 않고 대기할 수 밖에 없음(서로 다른 트랜잭션에 필요한 락을 확보하고 풀어주지 않는 상태)
- 데이터베이스 서버는 데드락이 발생하면 데드락이 발생한 채로 시스템이 멈추도록 방치하지 않음
	- 트랜잭션 간에 데드락이 발생했다는 사실을 확인하고 나면, 데드락이 걸린 트랙잭션 가운데 희생양을 하나 선택해 해당 트랜잭션을 강제 종료시킴
	- 데드락 확인 작업에는 보통 대기 상태를 나타내는 그래프에서 사이클이 발생하는지를 확인하는 방법을 사용함
	- 트랜잭션이 강제 종료되면 남아 있는 다른 트랜잭션은 락을 확보하고 계속 진행할 수 있음
	- 애플리케이션에서 중단된 트랜잭션을 재시도하도록 구현했다면, 재시도한 트랜잭션은 문제없이 결과를 얻을 수 있음
- JVM은 데드락 상태를 추적하는 기능은 갖고 있지 않음 --> 데드락이 발생하면 게임 끝
- 데드락이 걸린 상태에서 애플리케이션을 정상적인 상태로 되돌릴 수 있는 방법은 애플리케이션을 종료하고 다시 실행하는 것 밖에 없음
- 데드락은 상용 서비스를 시작하고 나서 시스템에 부하가 걸리는 경우와 같이 항상 최악의 상황에서 드러나곤 함

</br>

### 10.1.1 락 순서에 의한 데드락
~~~java
// 데드락 위험!
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();
    
    public void leftRight() {
        synchronized (left) {
            synchronized (right) {
                doSomething();
            }
        }
    }
    
    public void rigthLeft() {
        synchronized (right) {
            synchronized (left) {
                doSomethingElse();
            }
        }
    }
}
~~~
- leftRight()와 rigthLeft()는 각각 left, right 락을 확보하게 되어 있음
- A 스레드에서 leftRight() 호출, B 스레드에서 rigthLeft() 호출하면 아래와 같이 서로 엮여 데드락이 발생하게 됨  
![deadlock-ex01](/img/deadlock-ex01.png)     

- 두 개의 스레드가 서로 다른 순서로 동일한 락을 확보하려 했기 때문에 데드락이 발생함
- 양쪽 스레드에서 같은 순서로 락을 확보하도록 되어 있었다면 종속성 그래프에서 사이클이 발생하기 않기 때문에 데드락이 생기지 않음
- 프로그램 내부의 모든 스레드에서 필요한 락을 모두 같은 순서로만 사용한다면, 락 순서에 의한 데드락은 발생하지 않음
- 락을 사용하는 순서가 일정한지 확인하려면 프로그램 내부에서 락을 사용하는 패턴과 방법을 전반적으로 검증해야 함
- 락을 공유하는 상황에서 데드락을 방지하려면 오른손이 하는 일을 왼손이 알고 있어야 함


### 10.1.2 동적인 락 순서에 의한 데드락
~~~java
public void transferMoney(Account fromAccount, 
						  Account toAccount, 
						  DollarAmount amount) throws InsufficientFundsException {
        synchronized (fromAccount) {
            synchronized (toAccount) {
                if(fromAccount.getBalance().compareTo(amount) < 0) {
                    throw new InsufficientFundsException();
                } else  {
                    fromAccount.debit(amount);
                    toAccount.credit(amount);
                }
            }
        }
    }
~~~
- 모든 스레드가 락을 동일한 순서로 확보하려 할 떄 데드락이 발생할 수 있음
- 락을 확보하는 순서는 전적으로 transferMoney 메소드를 호출할 때 넘겨주는 인자의 순서에 달림
- 2개의 스레드가 transferMoney()를 호출하되 **A 스레드는 X 계좌에서 Y 계좌로, B 스레드는 Y 계좌에서 X 계좌로** 이체하는 경우 데드락이 발생
- 중첩된 구조에서 락을 가져가려는 상황을 찾아내는 것으로 검출 가능
- 락을 확보하려는 순서를 내부적으로 제어할 수 없기 때문에 데드락을 방지하려면 락을 특정 순서에 맞춰 확보하도록 해야하고, 락 확보 순서를 프로그램 전반적으로 동일하게 적용해야 함
- 객체에 순서를 부여할 수 있는 방법 중에 하나는 `System.identityHashCode`를 사용하는 방법
	- 해당 객체의 `Object.hashCode`메소드를 호출했을 때의 값을 알려줌
~~~java
private static final Object tieLock = new Object();

public void transferMoney(Account fromAccount, 
				    	  Account toAccount, 
				    	  DollarAmount amount) throws InsufficientFundsException {
    class Helper {
        public void transfer() throws InsufficientFundsException {
            if(fromAccount.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            } else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
    
    int fromHash = System.identityHashCode(fromAccount);
    int toHash = System.identityHashCode(toAccount);
    
    if(fromHash < toHash) {
        synchronized (fromAccount) {
            synchronized (toAccount) {
                new Helper().transfer();
            }
        }
    } else if(fromHash > toHash) {
        synchronized (toAccount) {
            synchronized (fromAccount) {
                new Helper().transfer();
            }
        }
    } else  {
        synchronized (tieLock) {
            synchronized (fromAccount) {
                synchronized (toAccount) {
                    new Helper().transfer();
                }
            }
        }
    }
}
~~~
- identityHashCode()를 사용해 락 순서를 조절하도록 변경하여, 데드락의 위험을 없앤 코드
- 두 개의 객체가 동일한 hashCode 값을 갖고 있는 경우, 다른 방법을 사용해 락 확보 순서를 조절해야 함
- 락 순서가 일정하지 않을 수 있다는 문제점을 제거하려면 타이 브레이킹(tie-breaking) 락을 사용하는 방법이 존재
	- 계좌에 대한 락을 확보하기 전에 먼저 타이 브레이킹 락을 확보
	- 타이 브레이킹 락을 확보한다는 것은 두 개의 락을 임의의 순서로 확보하는 위험한 작업을 특정 순간에 하나의 스레드에서만 할 수 있도록 막는다는 의미
- hashCode가 동일한 값을 갖는 경우가 자주 발생한다면 타이 브레이킹 락을 확보하는 부분이 일종의 병목으로 작용할 가능성도 존재
- `System.identityHashCode` 값이 충돌하는 경우는 거의 없기 때문에 타이 브레이킹 방법을 쓰지 않더라도 최소한의 비용으로 최대의 결과를 얻을 수 있음
- Account 클래스 내부에 계좌 번호와 같이 유일하면서 불변이고 비교도 가능한 값을 키로 갖고 있다면, 쉬운 방법으로 락 순서를 지정할 수 있음
- Account 객체를 내부의 키를 기준으로 정렬한 다음 순서대로 락을 확보한다면 타이 브레이킹 방법을 사용하지 않고 계좌를 사용할 떄 락이 걸리는 순서를 일정하게 유지할 수 있음
~~~java
public class DemonstratedDeadlock {
    private static final int NUM_THREADS = 20;
    private static final int NUM_ACCOUNTS = 5;
    private static final int NUM_ITERATORS = 1000000;

    public static void main(String[] args) {
        final Random rnd = new Random();
        final Account[] accounts = new Account(NUM_ACCOUNTS);
        
        for(int i = 0; i < accounts.length; i++) {
            accounts[i] = new Account();
        }
        
        class TransferThread extends Thread {
            @Override
            public void run() {
                for(int i = 0; i < NUM_ITERATORS; i++) {
                    int fromAcct = rnd.nextInt(NUM_ACCOUNTS);
                    int toAcct = rnd.nextInt(NUM_ACCOUNTS);
                    DollarAmount amount = new DollarAmount(rnd.nextInt(1000));
                    transferMoney(accounts[fromAcct], accounts[toAcct], amount);
                }
            }
        }
        for(int i = 0; i < NUM_THREADS; i++) {
            new TransferThread().start();
        }
    }
}
~~~
- 일반적으로 데드락에 금방 빠지는 반복문의 예제

### 10.1.3 객체 간의 데드락
~~~java
// 데드락 위험!
class Taxi {
    @GuardedBy("this") private Point location, destination; 
    private final Dispatcher dispatcher; 
    
    public Taxi(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }
    
    public synchronized Point getLocation() {
        return location;
    }
    
    public synchronized void setLocation(Point location) {
        this.location = location;
        if(location.equals(destination))
            dispatcher.notifyAvailable(this);
    }
}

class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;
    
    public Dispatcher() {
        taxis = new HashSet<Taxi>();
        availableTaxis = new HashSet<Taxi>();
    }
    
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }
    
    public synchronized Image getImage() {
        Image image = new Image();
        for(Taxi t :taxis)
            image.drawMarker(t.location);
        return image;
    }
}
~~~
- Taxi 클래스 : 현재 위치와 이동 중인 목적지를 속성으로 갖는 개별 택시를 의미
- Dispatcher 클래스 : 한 무리의 택시를 의미
- 두 개의 락을 모두 사용해야하는 메소드는 하나도 없음에도 불구하고 setLocation 메소드와 getImage 메소드를 호출하는 클래스는 두 개의 락을 사용하는 셈이 됨
- setLocation() 과 notifyAvailable() 은 모두 synchronized 로 묶여 있기 때문에 setLocation()을 호출하는 클래스는 Taxi에 대한 락을 확보하는 셈이고, 다음으로 Dispatcher 락을 확보함
- getImage() 를 호출하는 스레드 역시 Dispatcher 락을 확보하고, 다음으로 Taxi 락을 화보함
- 두 개의 스레드에서 두개의 락을 서로 다른 순서로 가져가려는 상황, 즉 데드락이 발생하게 됨
- 락을 확보한 상태에서 에일리언 메소드를 호출하는지 확인하면 데드락이 발생하는 부분을 찾아내는데 도움이 됨
- 락을 확보한 상태에서 에일리언 메소드를 호출한다면 가용성에 문제가 생길 수 있음
- 에일리언 메소드 내부에서 다른 락을 확보하려고 하거나, 아니면 예상하지 못한 만큼 오랜 시간 동안 계속해서 실행된다면 호출하기 전에 확보했던 락이 필요한 다른 스레드가 계속해서 대기해야 하는 경우도 생길 수 있음

### 10.1.4 오픈 호출
- 데드락이 발생한 상황에서 자신 각자가 데드락의 원인이라는 사실을 알지 못하며, 알지 못해야만 함
	- 메소드 호출이라는 것이 그 너머에서 어떤 일이 일어나는지 모르게 막아주는 추상화 방법이기 때문
- 특정 락을 확보한 상태에서 에일리언 메소드를 호출한다는 건 파급 효과를 분석하기가 굉장히 어렵고, 따라서 위험도가 높은 일임
- 락을 전혀 확보하지 않은 상태에서 메소드를 호출하는 것을 오픈 호출(open call)이라고 함
- 메소드를 호출하는 부분이 모두 오픈 호출로만 이뤄진 클래스는 락을 확보한 채로 메소드를 호출하는 클래스보다 훨씬 안정적이며 다른 곳에서 불러다 쓰기도 좋음
- 데드락을 미연에 방지하고자 오픈 호출을 사용하는 것은 스레드 안전성을 확보하기 위해 캡슐화 기법(encapsulation)을 사용하는 것과 비슷하다고 볼 수 있음
- 활동성이 확실한지를 분석하는 경우에도 오픈 호출 기법을 적용한 프로그램이라면 그렇지 않은 프로그램보다 분석 작업이 훨씬 간편함
- Taxi 와 Dispatcher 클래스는 오픈 호출을 사용하도록 쉽게 리팩토링 할 수 있으며, 그 결과로 데드락을 막을 수 있음
~~~java
@ThreadSafe
class Taxi {
    @GuardedBy("this") private Point location, destination;
    private final Dispatcher dispatcher;

    ...

    public synchronized Point getLocation() {
        return location;
    }

    public void setLocation(Point location) {
        boolean reachedDestination;
        synchronized (this) {
            this.location = location;
            reachedDestination = location.equals(destination);
        }
        if(reachedDestination)
            dispatcher.notifyAvailable(this);
    }
}

@ThreadSafe
class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;

    ...

    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }

    public Image getImage() {
        Set<Taxi> copy;
        synchronized (this) {
            copy = new HashSet<Taxi>(taxis);
        }
        Image image = new Image();
        for(Taxi t :copy)
            image.drawMarker(t.location);
        return image;
    }
}
~~~
- synchronized 블록의 범위를 최대한 줄여 공유되 ㄴ상태 변수가 직접 관련된 부분에서만 락을 확보하도록 함
- 객체 간의 데드락 방지를 위해 오픈 호출을 사용
- Taxi 첫번째 예제는 습관적으로 메소드 전체에 synchronized 구문으로 동기화를 걸어주는 것이 원인일 수 있음
- 프로그램을 작성할 때 최대한 오픈 호출 방법을 사용하도록 함
- 내부의 모든 부분에서 오픈 호출을 사용하는 프로그램은 락을 확보한 상태로 메소드를 호출하곤 하는 프로그램보다 데드락 문제를 찾아내기 위한 분석 작업을 훨씬 간편하게 해줌
- synchronized 블록의 구조를 변경해 오픈 호출 방법을 사용하도록 하면 원래 단일 연산으로 실행되던 코드를 여러 개로 쪼개 실행하기 때문에 예상치 못한 상황에 빠지기도 함
- 연산의 단일성을 잃는다는 것이 일부 상황에서는 큰 문제가 되기도하며, 연산의 단일성을 확보하는 방법에는 오픈 호출된 이후에 실행될 코드가 한 번에 하나씩만 실행되도록 객체의 구조를 정의할 수 있음
	- 서비스를 종료하고자 할 때, 현재 실행작업이 끝나기를 대기 -> 완료 후 서비스에서 사용하던 자원 반납의 순서를 밟음
	- 서비스 상태를 "종료중" 이라고 설정할 동안만 락을 확보하고, 다른 스레드가 새로운 작업을 시작하거나 종료 절차를 시작하지 못하도록 미리 예방하는 방법이 존재
	- 코드 가운데 크리티컬 섹션(critical section)에 다른 스레드가 들어오지 못하도록 하기 위해 락을 사용하는 대신 이와같이 스레드 간의 약속을 정해 다른 스레드가 작업을 방해하지 않도록 하는 방법이 존재함


### 10.1.5 리소스 데드락
- 필요한 자원을 사용하기 위해 대기하는 과정에도 데드락이 발생할 수 있음
	- A 스레드는 데이터베이스 D1에 대한 연결을 확보한 상태에서 데이터베이스 D2에 대한 연결을 확보하고자 하고, B 스레드는 데이터베이스 D2에 대한 연결을 확보한 상태에서 데이터베이스 D1을 확보하고자 하는 상황
- 자원과 관련해 발생할 수 있는 또 다른 데드락 상황은 스레드 부족 데드락(thread starvation deadlock)
	- 단일 스레드로 동작하는 Executor에서 현재 실행 중인 작업이 또 다른 작업을 큐에 쌓고는 그 작업이 끝날 때까지 대기하는 데드락 상황
	- - 다른 작업의 실행 결과를 사용해야만 하는 작업이 있다면 스레드 소모성 데드락의 원인이 되기 쉬움
- 크기가 제한된 풀과 다른 작업과 연동돼 동작하는 작업은 잘못 사용하면 이와 같은 문제를 일으킬 수 있음

</br>

## 10.2 데드락 방지 및 원인 추적
- 한번에 하나 이상의 락을 사용하지 않는 프로그램은 락의 순서에 의한 데드닭이 발생하지 않음
- 여러 개의 락을 사용해야만 한다면 락을 사용하는 순서 역시 설계 단계부터 고려해야 함
	- 설계 과정에서 여러 개의 락이 서로 함께 동작하는 부분을 최대한 줄이고, 락의 순서를 지정하는 규칙을 정해 문서로 남겨 규칙을 정확하게 따라 프로그램을 작성해야 함
- 두 단계 전략으로 데드락 발생 가능성이 없는지를 확인해야 함
	- 첫번째 단계 : 여러 개의 락을 확보해야하는 부분이 어디인지를 찾아내는 단계
	- 두번째 단계 : 첫번째 단계에서 찾은 부분에 대한 전반적인 분석 작업을 진행해 프로그램 어디에서건 락을 지정된 순서에 맞춰 사용하도록 함

</br>

### 10.2.1 락의 시간 제한
- 데드락 상태를 검출하고 데드락에서 복구하는 또 다른 방법은 Lock 클래스의 메소드 가운데 시간을 제한할 수 있는 tryLock 메소드를 사용하는 방법
	- synchonized 암묵적인 락은 락을 확보할 때까지 영원히 기다림
	- Lock 클래스 등의 명시적인 락은 일정시간을 정해두고 락을 확보하지 못하면 tryLock 메소드가 오휴를 발생시키도록 할 수 있음
	- 락을 확보하는데 걸릴 것이라고 예상하는 시간보다 훨씬 큰 값을 타임아웃으로 지정하고 tryLock을 호출하면, 일반적이지 않은 상황 발생 시 제어권을 되돌려 받을 수 있음
- 지정한 시간이 되도록 락을 확보하지 못해도, 락을 왜 확보하지 못했는지는 몰라도 됨
- 명시적인 락을 사용하면 락을 화보하려고 했지만 실패했다는 사실을 기록해 둘 기회는 갖는 셈이고, 그동안 발생했던 내용을 로그로 남길 수 있음
	- 프로그램 전체 재시작 대신, 프로그램 내부에서 필요한 작업을 재시도하도록 할 수도 있음
- 여러개의 락을 확보할 때 이와 같이 타임아웃을 지정하는 방법을 적용하면, 프로그램 전체에서 모두 타임아웃을 사용하지 않는다 해도 데드락을 방지하는데 효과를 볼 수 있음


### 10.2.2 스레드 덤프를 활용한 데드락 분석
