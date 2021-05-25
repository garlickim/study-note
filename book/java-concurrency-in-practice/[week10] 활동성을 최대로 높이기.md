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
