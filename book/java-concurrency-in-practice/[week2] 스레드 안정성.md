# 스레드 안전성
- 스레드에 안전한 코드를 작성하는 것은 **공유되고 변경할 수 있는 상태에 대한 접근을 관리**하는 것
- 객체의 상태는 인스턴스나 static 변수와 같은 상태 변수에 저장된 객체 데이터를 지칭
- **객체에 여러 스레드가 접근할지의 여부**에 따라 객체가 스레드에 안전해야 하느냐를 결정
- synchronized, volatile, 명시적 락, atomic variable를 사용하는 경우 "동기화"라는 용어를 사용
  
- 적절한 동기화없이 여러 스레드가 변경할 수 있는 상태 변수가 있을 때, 프로그램을 고치는 3가지 방법이 존재
	- 해당 상태 변수를 스레드 간 공유하지 않도록 함
	- 해당 상태 변수를 변경할 수 없도록 만듦
	- 해당 상태 변수에 접근할 땐 언제나 동기화를 사용하도록 함

- 캡슐화, 데이터 은닉과 같은 기법이 스레드에 안전한 클래스를 작성하는데 도움이 될 수 있음
	- 스레드에 안전한 클래스를 설계할 땐, 바람직한 객체 지향 기법이 왕도

</br>

## 2.1 스레드 안정성이란?
- 여러 스레드가 클래스에 접근할 때 계속 정확하게 동작하면 해당 클래스는 스레드 안전하다고 정의
  - 실행 환경이 스레드들의 실행 스케쥴을 변경하더라도, 호출하는 쪽에서 추가적인 동기화나 조율없이 정확하게 동작하면 해당 클래스는 Thread Safe 하다고 정의
- 스레드에 안전한 클래스는 클라이언트 쪽에서 별도로 동기화할 필요가 없도록 동기화 기능도 캡슐화 함

### 2.1.1 상태없는 서블릿
~~~java
@ThreadSafe
public class StatelessFactorizer implements Servlet {
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        encodeIntoResponse(resp, factors);
        
    }
}
~~~
- 선언한 변수가 없고, 다른 클래스의 변수를 참조하지 않음
- 특정 계산을 위한 일시적인 상태는 스택에 저장되는 지역 변수(i, factors)에만 저장
- 위의 클래스에 접근하는 특정 스레드는 동일한 클래스에 접근하는 다른 스레드의 결과에 영향을 줄 수 없음
- **상태 없는 객체는 항상 스레드에 안전**

</br>

## 2.2 단일 연산
~~~java
@NotThreadSafe
public class StatelessFactorizer implements Servlet {
    private long count = 0;
    
    public long getCount() { return count; }
    
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        ++count;
        encodeIntoResponse(resp, factors);

    }
}
~~~
- 처리한 요청 수를 기록하는 "접속 카운터(count)"를 추가한 예제
- 단일 스레드에서 위의 예제는 문제 없이 동작, 그러나 멀티 스레드에서는 변경한 값을 잃어버리는 경우가 생길 수 있음
- ++count 연산은 get -> plus operation -> save 3가지 연산을 합쳐놓은 것  
![UnTreadSafe-ex01](/img/UnTreadSafe-ex01.png)   
- 위와 같이 타이밍이 안 좋을 때 결과가 잘못될 가능성이 존재
	- 이런 상황을 "경쟁 조건(race condition)이라는 용어로 정의"

### 2.2.1 경쟁 조건
- 경쟁 조건은 상대적인 시점이나 또는 JVM이 여러 스레드를 교차해서 실행하는 **상황에 따라 계산의 정확성이 달라질 때** 나타남
- 가장 일반적인 경쟁 조건 형태는 잠재적으로 유효하지 않은 값을 참조하여 다음에 뭘 할지를 결정하는 점검 후 행동(check-then-act) 형태의 구문
- 대부분의 경쟁 조건은 관찰 결과의 무효화로 특징 지어짐
- 잠재적으로 유효하지 않은 관찰 결과로 결정을 내리거나 계산하는 형태의 경쟁 조건을 점검 후 행동이라 함
  
### 2.2.2 늦은 초기화 시 경쟁 조건
- 늦은 초기화(lazy initialization)는 특정 객체가 필요한 시점까지 초기화를 미루고, 동시에 단 한번만 초기화 되도록 하기 위한 것
~~~java
@NotThreadSafe
public class LazyInitRace {
    private ExpensiveObject instance = null;
    
    public ExpensiveObject getInstance() {
        if(instance == null)
            instance = new ExpensiveObject();
        return instance;
    }
}
~~~
- LazyInitRace는 경쟁 조건 때문에 제대로 동작하지 않을 가능성이 존재
- Thread-A / Thread-B 가 존재하는 경우, 타이밍 문제에 의해 두 스레드가 서로 다른 인스턴스를 가져갈 수 있음


### 2.2.3 복합 동작
- 경쟁 조건을 피하려면 변수가 수정되는 동안 다른 스레드가 해당 변수를 사용하지 못하도록 막는 방법이 필요함
- 단일 연산 작업은 자신을 포함해 같은 상태를 다루는 모든 작업이 단일 연산인 작업을 지칭
- 스레드에 안전하기 위해서는 전체가 단일 연산으로 실행되야 하는 일련의 동작으로 이루어져야 함
- 연산의 단일성을 보장하기 위해 자바에서는 락(lock)을 기본적으로 제공
</br>

~~~java
@ThreadSafe
public class CountingFactorizer implements Servlet {
    private final AtomicLong count = new AtomicLong(0);
    
    public long getCount() { return count.get(); }
    
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        count.incrementAndGet();
        encodeIntoResponse(resp, factors);

    }
}
~~~  
- java.util.concurrent.atomic 패키지에는 숫자나 객체 참조 값에 대해 **상태를 단일 연산으로 변경할 수 있도록 atomic variable 클래스**가 존재
- 서블릿의 상태 == 카운터의 상태 , 카운터가 스레드에 안전하기 때문에 서블릿은 스레드에 안전함
- 클래스 상태를 관리하기 위해 AtomicLong과 같은 스레드에 안전하게 이미 만들어진 객체를 사용하는 편이 좋음

</br>

## 2.3 락
~~~java
@NotTh readSafe
public class UnsafeCachingFactorizer implements Servlet {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<>();
    private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<>();

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);

        if (i.equals(lastNumber.get())) {
            encodeIntoResponse(resp, lastFactors.get());
        } else {
            BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
            encodeIntoResponse(resp, factors)
        }
    }
}
~~~
- 단일 연산 참조 변수 각각(lastNumber, lastFactors)은 스레드에 안전하지만, UnsafeCachingFactorizer 자체는 틀린 결과를 낼 수 있는 경쟁 조건을 가지고 있음
- 인수분해 결과를 곱한 값이 lastNumber에 캐시된 값과 같아야한다는 불변조건이 항상 성립해야만 서블릿이 정상적으로 동작함
- 상태를 일관성 있게 유지하려면 관련 있는 변수들을 하나의 단일 연산으로 갱신해야 함

### 2.3.1 암묵적인 락
- 자바는 단일 연산 특성을 보장하기 위해 synchronized 라는 구문으로 사용할 수 있는 락을 제공
- synchronized 구문은 락으로 사용될 객체의 참조 값과 해당 락으로 보호하려는 코드 블록으로 구성
~~~java
synchronized (lock) {
    // lock으로 보호된 공유 상태에 접근하거나 해당 상태를 수정
}
~~~
- 자바에 내장된 락을 암묵적인 락(intrinsic lock) 또는 모니터 락(monitor lock)이라 함
- 락은 스레드가 synchronized 블록에 들어가기 전에 자동으로 확보되며, 해당 블록을 벗어날 때 자동으로 해제됨
- 암묵적인 락은 뮤텍스(mutexes), 또는 mutual exclusion lock(상호 배제 락)으로 동작
    - **한 번에 한 스레드만 특정 락을 소유할 수 있음**
- 일련의 문장이 하나의 나눌 수 없는 단위로 실행되는 것처럼 보인다는 의미
- 한 스레드가 synchronized 블록을 실행 중이라면 같은 락으로 보호되는 synchronized 블록에 다른 스레드가 들어올 수 없음
~~~java
@ThreadSafe
public class SynchronizedFactorizer implements Servlet {
    @GuardedBy("this")
    private BigInteger lastNumber;
    @GuardedBy("this")
    private BigInteger[] lastFactors;

    public synchronized void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);

        if (i.equals(lastNumber)) {
            encodeIntoResponse(resp, lastFactors);
        } else {
            BigInteger[] factors = factor(i);
            lastNumber = i;
            lastFactors = factors;
            encodeIntoResponse(resp, factors);
        }
    }
}
~~~
- SynchronizedFactorizer는 스레드에 안전하지만, 여러 클라이언트가 동시에 접근할 수 없기 때문에 응답 속도가 느리고 성능이 떨어짐

### 2.3.2 재진입성
- 암묵적인 락은 재진입 가능(reentrant)하기 때문에 특정 스레드가 자기가 이미 획득한 락을 다시 확보할 수 있음
- 재진입성은 확보 요청 단위가 아닌 스레드 단위로 락을 얻는다는 것을 의미
- 재진입성을 구현하려면 각 락마다 확보 횟수와 확보한 스레드를 연결시켜 둠
    - 확보 횟수가 0이면 락은 해제된 상태
- 재진입성으로 인해 락의 동작을 쉽게 캡슐화할 수 있음  

~~~java
public class Widget {
    public synchronized void doSomething() {
        ...
    }
}

public class LoggingWidget extends Widget {
    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
~~~ 
- 암묵적인 락이 재진입 가능하지 않는다면, 데드락에 빠질 가능성이 존재함
- Widget과 LoggingWidget의 doSomething()은 각각 진행 전에 Widget에 대한 락을 얻으려 시도함. 
    --> 암묵적인 락이 재진입 가능하지 않다면, 이미 락을 누군가 확보했기 때문에 super.doSomething() 호출에서 락을 얻을 수 없게되어 데드락에 빠지게 됨  

</br>

## 2.4 락으로 상태 보호하기
- 특정 변수에 대한 접근을 조율하기 위해 동기화할 때는 해당 변수에 접근하는 모든 부분을 동기화해야 함
- 변수에 대한 접근을 조율하기 위해 락을 사용할 땐 해당 변수에 접근하는 모든 곳에서 반드시 같은 락을 사용해야 함
- 여러 스레드에서 접근할 수 있고 변경 가능한 모든 변수를 대상으로 해당 변수에 접근할 때는 항상 동일한 락을 먼저 확보한 상태여야 함
    - 해당 변수는 확보된 락에 의해 보호됨
- 모든 변경할 수 있는 공유 변수는 정확하게 단 하나의 락으로 보호해야 함
    - 유지 보수하는 사람이 알 수 있게 어느 락으로 보호하고 있는지를 명확하게 표시할 필요가 있음
- 무차별적인 synchronized의 사용은 동기화가 너무 과도하거나 부족할 수 있음
- Vector 처럼 모든 메소드가 단순히 동기화돼 있긴 하지만 Vector를 사용하는 복합 동작까지 단일 연산으로 만들지는 못함
~~~java
if(!vector.contains(element))
    vector.add(element);
~~~
- contains, add가 단일 연산이라 해도 위의 코드는 put-if-absent의 동작으로 경쟁 조건을 가지고 있음
- 여러 메소드가 하나의 복합 동작으로 묶일 땐 락을 사용해 추가로 동기화가 필요함

</br>

## 2.5 활동성과 성능
- SynchronizedFactorizer처럼 메소드 전체를 동기화하면 안전성을 확보할 순 있지만 성능이 떨어짐
    - 병렬 프로그래밍임에도 불구하고 병렬 처리 능력이 떨어지게 됨
- synchronized 블록의 범위를 줄이면 스레드 안정성을 유지하면서 동시성을 향상시킬 수 있음
    - 단, 블록의 범위를 너무 작게 줄이지 않도록 해야함
    - 오래 걸리는 작업을 synchronized 블록에서 뽑아낼 필요는 있음
~~~java
@ThreadSafe
public class CacheFactorizer implemnts Servlet {
    @GuardedBy("this") private BigInteger lastNamber;
    @GuardedBy("this") private BigInteger[] lastFactors;
    @GuardedBy("this") private long hits;
    @GuardedBy("this") private long cacheHits;

    public synchronized long getHits() {
        return hit;
    }

    public synchronized double getCacheHitRatio() {
        return (double) cacheHits / (double) hits;
    }

    public void service(ServletRequest req, ServletResponse reps) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = null;
        synchronized (this) {
            ++hits;
            if (i.equals(lastNumber)) {
                ++cacheHits;
                factors = lastFactors.clone();
            }
        }
        if (factors == null) {
            factors = factor(i);
            synchronized (this) {
                lastNumber = i;
                lastFactors = factors.clone();
            }
        }
        encodeIntoResponse(reps, factors);
    }
}
~~~
- CacheFactorizer는 두 개의 짧은 코드 블록을 synchronized 키워드로 보호함
- synchronized 블록 밖에 있는 코드는 다른 스레드와 공유하지 않는 지역(=stack) 변수만 사용하기 때문에 동기화가 필요없음
- 단순성과 병렬 처리 능력 사이에 균형을 맞춘 코드
- synchronized 블록의 크기를 적정하게 유지하려면 안정성, 단순성, 성능 등의 서로 상충하는 설계 원칙 사이에 적절한 타협이 필요할 수 있음
    - 성능을 위해 단순성을 희생하고픈 유혹은 버려야 함
- 락을 사용할 땐 블록 안의 코드 행위, 수행 시간을 파악해야 함
- 복잡하고 오래 걸리는 계산 작업, 네트웍 작업, 사용자 입출력 작업과 같이 빨리 끝나지 않을 수 있는 작업에서는 가능한 락을 잡지 말 것
