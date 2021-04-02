# 객체 공유
- 여러 개의 스레드에서 특정 객체를 동시에 사용하려 할 때, 섞이지 않고 안전하게 동작하도록 객체를 공유하고 공개하는 방법을 소개
- 항상 특정 객체를 명시적으로 동기화시키거나, 객체 내부에 적절한 동기화 기능을 내장시켜야함

</br>

## 3.1 가시성
- 메모리 가시성이란, cpu의 캐시 메모리와 메인 메모리의 값이 다른 것
	- 즉, 한 스레드에서 변경한 값이 다른 스레드에서 접근했을 때 다른 값을 보게되는 현상
~~~java
public class NoVisibility {
    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
                Thread.yield();
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 43;
        ready = true;
    }
}
~~~
- 일반적으로, ReaderThread가 43 값을 출력할 것이라 생각하지만 **0을 출력할 수도, 값을 영원히 출력하지 못할 수도 있음**
- 여러 스레드에서 동일한 변수를 공유하여 사용할 때 동기화를 하지 않으면 위와 같은 문제가 발생할 수 있음
- ReaderThread 스레드가 메인 스레드에서 number 변수에 지정한 값보다 ready 변수 값을 먼저 읽어갈 수도 있음
	- 재배치(reordering)에 의해
	- 재배치 현상은 특정 메소드의 코드가 100% 코딩된 순서로 동작한다는 점을 보장할 수 없다는 점에 기인하는 문제
- 동기화 기능을 지정하지 않으면 컴파일러/프로세서/JVM 등에 의해 코드 실행 순서가 임의로 변경되는 경우가 발생하기도 함
	- **동기화되지 않은 상황에서 메모리상의 변수를 대상으로 작성해둔 코드가 '반드시 이런 순서로 동작할 것이다'라고 단정지을 수 없음**

### 3.1.1 스테일 데이터
- 스테일(stale) 데이터란 한 프로세서가 피연산자의 값을 변경하고, 그리고 이어서 그 피연산자를 불러왔을 때 피연산자의 새로운 값이 아닌 변경되기 이전의 값을 가지고 왔다면 이때 불러온 값을 일컫음
- 스테일 현상이 발생하면 예기지 못한 예외 상황이 발생하기도 하고, 데이터를 관리하는 자료 구조가 망가질 수도 있고, 계산된 결과 값이 올바르지 않을 수도 있으며 무한 반복에 빠져들 수도 있음
~~~java
@NotThreadSafe
public class MutableInteger {
    private int value;

    public int get() { return value; }
    public void set(int value) { this.value = value; }
}
~~~
- value 변수 값을 get/set 메소드에서 동시 사용함에도 불구하고 멀티 스레드 환경에서는 문제가 발생할 가능성이 높음
	- set 메소드에서 지정한 값을 get 메소드에서 제대로 읽어가지 못할 수 있음
  
~~~java
@ThreadSafe
public class SynchronizedInteger {
    @GuardedBy("this") private int value;

    public synchronized int get() { return value; }
    public synchronized void set(int value) { this.value = value; }
}
~~~
- synchronized 키워드를 사용하여 동기화시킴
- set 메소드만 동기화 처리한다면? get 메소드는 스테일 상황을 초래할 수 있음
	- 앞장에서 언급했던 것처럼 공유 변수를 사용하는 모든 곳을 동기화 해야한다는 것을 잊지말 것

### 3.1.2 단일하지 않은 64비트 연산
- volatile로 지정되지 않은 long이나 double 형의 64비트 숫자형을 사용하는 경우, 완전 난데없는 값을 읽어갈 가능성이 존재
- 예전 하드웨어의 경우, 32비트 CPU 임에 따라 64비트 long/double에 대해서는 32비트 연산을 2번 진행함
	- 이에 따라 2번의 연산 중 각각 다른 스레드에 의해 변수 읽기/쓰기가 일어난다면 이상한 데이터를 읽게될 가능성이 존재

### 3.1.3 락과 가시성
- 여러 스레드에서 사용하는 변수를 적당한 락으로 막아주지 않는다면, 스테일 상태에 쉽게 빠질 수 있음
- 락은 상호 배제(mutual exclusion)뿐만 아니라 정상적인 메모리 가시성을 확보하기 위해서도 사용함
- 변경 가능하면서 여러 스레드가 공유해 사용하는 변수를 각 스레드에서 최신의 정상값으로 활용하려면 **동일한 락을 사용해 모두 동기화**시켜야 함

### 3.1.4 volatile 변수
- volatile로 선언된 변수의 값을 바꿨을 때 다른 스레드에서 항상 최신 값을 읽어갈 수 있도록 해줌
	- 즉, 컴파일러와 런타임 모두 "이 변수는 공유 변수이며, 실행 순서를 재배치해서는 안 된다"라고 이해함
- CPU 캐시를 사용하지 않음
- volatile 변수를 사용할 때에는 아무런 락이나 동기화 기능이 동작하지 않기 때문에 synchronized를 사용한 동기화보다는 강도가 약함
- volatile 변수를 사용한 것과 synchronized 처리를 한 것의 메모리 가시성은 비슷한 효과를 지님
- volatile 변수만을 사용해 메모리 가시성을 확보한 코드는 synchronized로 직접 동기화한 코드보다 훨씬 읽기가 어렵고, 오류 발생 가능성이 높음
~~~java
volaile boolean asleep;
...
    while(!asleep)
        countSomeSheep();
~~~
- 양의 마리 수를 세는 기능이 제대로 동작하려면 asleep 변수가 반드시 volatile로 선언되어 있어야 함
- 단 하나의 스레드에서만 사용한다는 보장이 없는 상태라면 volatile 연산자의 기본적인 능력으로는 증가 연산자(count++)를 사요한 부분까지 동기화를 맞춰주지는 못함
- 락을 사용하면 가시성과 연상의 단일성을 모두 보장받음. 그러나 volatile 변수는 연산의 단일성은 보장하지 못하고 가시성만 보장함
- 정리하면 **volatile 변수는 다음 상황에서만 사용하는 것이 좋음**
	- 변수에 값을 저장하는 작업이 해당 변수의 현재 값과 관련이 없거나 해당 변수의 값을 변경하는 스레드가 하나만 존재
	- 해당 변수가 객체의 불변조건을 이루는 다른 변수와 달리 불변조건에 관련되어 있지 않음
	- 해당 변수를 사용하는 동안에는 어떤 경우라도 락을 걸어 둘 필요가 없는 경우

</br>
+) 추가    

~~~java
public class MyClass {
    private int years;
    private int months
    private volatile int days;


    public void update(int years, int months, int days){
        this.years  = years;
        this.months = months;
        this.days   = days;
    }
}
~~~
- volatile 변수가 쓰여질 때(days에 값이 set 될 때), 나머지 변수 years/days도 메인 메모리에 값이 쓰여짐  
- [volatile 관련하여 읽어보면 좋을 article](http://tutorials.jenkov.com/java-concurrency/volatile.html)  

</br>

## 3.2 공개와 유출
- 특정 객체를 현재 코드의 스코프 범위 밖에서 사용할 수 있도록 만들면 해당 객체는 공개되었다고 함
- 클래스 내부의 상태 변수를 외부에 공개하는 건 객체 캡슐화 작업이 물거품이 되거나 내부 데이터의 안정성에 문제가 생길 수 있음
- 외부에서 사용할 수 있게 공개된 경우를 유출 상태(escaped)라고 함

</br>

~~~java
public static Set<Secret> knownSecrets;

public void initialize(){
    knownSecrets = new HashSet<Secret>();
}
~~~
- public static 변수에 객체를 설정하면 가장 직접적인 방법으로 해당 객체를 외부에 공개
- knownSecrets 변수에 저장된 HashSet 객체는 스코프에 관계없이 완전히 공개됨

</br>

~~~java
class UnsafeStates {
    private String[] states = new String[] {
            "AK", "AL", ...
    };
    
    public String[] getStates() { return states; }
}
~~~
- private으로 선언된 배열 변수가 getStates() 메소드를 통해 공개됨
    - getStates()를 호출하는 쪽에서 states 변수의 값을 직접 변경할 수 있음
- 객체를 공개하면 private을 제외한 모든 변수 속성에 연결되어 있는 모든 객체가 공개됨
    - 즉, 객체를 공개하면 그 객체 내부의 private이 아닌 변수나 메소드를 통해 불러올 수 있는 모든 객체는 함께 공개됨
</br>

~~~java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener() (
                new EventListener() {
                    public void onEvent(Event e) {
                        doSomething(e);
                    }
                });
    }
}
~~~
- 내부 클래스의 인스턴스를 외부에 공개하는 경우의 예제
- 내부 클래스는 항상 부모 클래스에 대한 참조를 갖고 있음
    - ThisEscape 클래스가 EventListener 객체를 외부에 공개하면 ThisEscape 클래스도 함께 외부에 공개됨

### 3.2.1 생성 메소드 안전성
- 생성 메소드 실행 도중에 this 변수가 외부에 공개된다면, 이론적으로 해당 객체는 정상적으로 생성되지 않았다고 말할 수 있음
- 생성 메소드를 실행하는 도중에는 this 변수가 외부에 유출되지 않게 해야 함
- this 변수를 유출시키는 가장 흔한 오류는 생성 메소드에서 스레드를 새로 만들어 시작시키는 일
    - 스레드를 생성하면서 바로 시작시키기보다는 스레드를 시작시키는 기능을 start/initialize 등의 메소드로 만들어 사용하는 편이 좋음
- 생성 메소드에서 오버라이드 가능한 다른 메소드를 호출하는 경우가 있다면 this 참조가 외부에 유출될 가능성이 존재
~~~java
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }
    
    public static SafeListener new Instance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
~~~
- 생성 메소드에서 this 변수가 외부로 유출되지 않도록 팩토리 메소드를 사용하는 예제
- 생성 메소드를 private으로 지정하고 public으로 지정된 팩토리 메소드를 만들어 사용하는 방법이 좋음

</br>

## 3.3 스레드 한정
- 객체를 사용하는 스레드를 한정(confine)하는 방법으로 스레드 안전성을 확보할 수 있음
- 객체 인스턴스를 특정 스레드에 한정시켜두면, 해당하는 객체가 [CPJ2.3.2]가 아니라 해도 자동으로 스레드 안전성을 확보하게 됨
- JDBC의 Connection 객체를 풀링해 사용하는 경우, 스레드 한정 기법을 사용함
    - 서블릿 또는 EJB 호출 등의 요청은 단일 스레드에서 동기적으로 처리하며, DB 풀은 한쪽에서 DB 연결을 사용하는 동안에는 해당 연결을 다른 스레드가 사용하지 못하게 막기 때문에, 공유하는 Connection 객체를 풀로 관리하면 특정 Connection을 한 번에 하나 이상의 스레드가 사용하지 못하도록 한정할 수 있음
- 임의의 객체를 특정 스레드에 한정시키는 기능을 제공하지 않음
    - 설계하는 과정부터 프로그램 구현 내내 계속해서 스레드 한정 기법을 적용 해야함

### 3.3.1 스레드 한정 - 주먹구구식
- 스레드 한정 기법을 사용할 것인지를 결정하는 일은 GUI 모듈과 같은 특정 시스템을 단일 스레드로 동작하도록 만들 것이냐에 달려있음
- 특정 모듈의 기능을 단일 스레드로 동작하도록 구현한다면, 언어적인 지원없이 직접 구현한 스레드 한정 기법에서 나타날 수 있는 오류의 가능성을 최소화할 수 있음 ( 단일 스레드로 모듈을 구현하는 부분은 9장에서 다룸 )
- volatile이 붙은 객체는 단일 스레드에서만 쓰기 작업을 하도록 구현해야 안전함
- 임시 방편적인 스레드 한정 기법은 안전성을 완벽하게 보장할 수 있는 방법이 아니기 때문에 꼭 필요한 부분만 제한적으로 사용하도록 함

### 3.3.2 스택 한정
- 특정 객체를 로컬 변수를 통해서만 사용할 수 있도록 하는 특별한 경우의 스레드 한정 기법
- 로컬 변수는 모두 암묵적으로 현재 실행 중인 스레드에 한정되어 있다고 불 수 있음
    - 로컬 변수는 현재 실행중인 스레드 내부의 스택에만 존재하기 때문
    - 스레드 내부의 스택은 외부 스레드에서 볼 수 없음
- 스택 한정 기법은 스레드 내부/스레드 로컬 기법이라고도 불림 (ThreadLocal 클래스와는 다름)
~~~java
public int loadTheArk(Collection<Animal> candidates) {
    SortedSet<Animal> animals;
    int numPairs = 0;
    Animal candidate = null;
    
    // animals 변수는 메소드에 한정되어 있으며, 유출돼서는 안 된다.
    animals = new TreeSet<Animal>(new SpeciesGenderComparator());
    animals.addAll(candidates);
    for(Animal a : animals) {
        if(candidate == null || !candidate.isPotentialMate(a))
            candidate = a;
        } else {
            ark.load(new AnimalPair(candidate, a));
            ++numPairs;
            candidate = null;
        }
    return numPairs;
}
~~~
- 기본 변수형(numPairs)은 객체와 같이 참조되는 값이 아니기 때문에, 기본 변수형을 사용하는 로컬 변수는 언어적으로 스택 한정 상태가 보장됨
- 객체형 변수가 스택 한정 상태를 유지할 수 있게 하려면 해당 객체에 대한 참조가 외부로 유출되지 않도록 개발자가 직접 주의를 기울여야 함
- TreeSet 인스턴스에 대한 참조를 로컬 변수 animals에만 보관하고 있음
    - 만약 TreeSet 인스턴스에 대한 참조를 외부에 공개하면, 스택 한정 상태가 깨짐

### 3.3.3 ThreadLocal
- ThreadLocal 클래스에는 get/set 메소드가 존재하며, 호출하는 스레드마다 다른 값을 사용할 수 있도록 관리해줌
    - 즉, ThreadLocal get 메소드를 호출하면 현재 실행 중인 스레드에서 최근에 set 메소드를 호출해 저장했던 값을 가져올 수 있음
- ThreadLocal 변수는 변경 가능한 싱글턴이나 전역 변수 등을 기반으로 설계되어 있는 구조에서 변수가 임의로 공유되는 상황을 막기 위해 사용하는 경우가 많음
~~~java
    private static ThreadLocal<Connection> connectionHolder
                = new ThreadLocal<Connection>() {
                    public Connection initialValue() {
                        return DriverManager.getConnection(DB_URL);
                    }
    };
    
    public static Connection getConnection() {
        return connectionHolder.get();
    }
~~~
- JDBC 연결을 보관할 때 ThreadLocal을 사용하면 스레드는 저마다 각자의 연결 객체를 갖게됨
</br>
- ThreadLocal은 Thread 객체 자체에 값을 저장하는 방식이며, 스레드가 종료되면 할당되어있던 부분은 GC에 의해 처리됨
- 스레드 단위로 트랜잭션 컨텍스트를 관리하고자 할 때는 static으로 선언된 ThreadLocal 변수에 트랜잭션 컨텍스트를 넣어두면 편리함
- 전역 변수가 아님에도 전역 변수처럼 동작하기 때문에 프로그램 구조상 전역 변수를 남발하는 결과를 가져올 수 있음
- 재사용성(reusability)이 떨어질 수 있음

</br>

## 3.4 불변성
- 불변 객체는 맨 처음 생성되는 시점을 제외하고는 그 값이 전혀 바뀌지 않는 객체를 말함
- 불변 객체는 언제라도 스레드에 안전함
- 객체의 상태가 변경되는 경우에 따로 대비할 필요없이 공유하여 사용할 수 있음
- 불변 객체를 구성할 때 내부의 모든 변수를 final로 설정할 필요 없음
- 객체 내부의 모든 변수를 final로 설정해도 해당 객체가 불변이지 않음
    - 변수에 참조로 연결되어 있는 객체가 불변 객체가 아니라면 내용이 바뀔 수 있기 때문
- 다음 조건을 만족하면 해당 객체는 불변 객체
    - 생성되고 난 이후에는 객체의 상태를 변경할 수 없다.
    - 내부의 모든 변수는 final로 선언되어야 한다.
    - 적절한 방법으로 생성돼야 한다(예를 들어 this 변수에 대한 참조가 외부로 유출되지 않아야 한다).
~~~java
@Immutable
public final class ThreeStooges {
    private final Set<String> stooges = new HashSet<String>();

    public ThreeStooges() {
        stooges.add("Moe");
        stooges.add("Larry");
        stooges.add("Curly");
    }

    public boolean isStooge(String name) {
        return stooges.contains(name);
    }
}
~~~
- 생성 메소드를 실행한 이후에는 set 변수의 값을 변경할 수 없도록 되어있음
- 호출한 클래스나 생성 메소드 이외의 부분에서 참조를 가져갈 수 있는 일을 전혀 하고 있지 않기 때문에 ThreeStooges는 불변 객체
- 불변 객체를 사용하면 락이나 방어적인 복사본을 만들어 관리해야 할 필요가 없기 때문에 오히려 성능에 도움을 줌

### 3.4.1 final 변수
- final로 설정한 변수의 값은 변경할 수 없음
    - 단, 변수가 가리키는 객체가 분변 객체가 아니라면 해당 객체 들어 있는 값은 변경 할 수 있음
- final 키워드를 적절하게 사용하면 초기화 안전성(initialization safety)을 보장하기 때문에 별다른 동기화 작업 없이도 불변 객체를 자유롭게 사용하고 공유할 수 있음
- 외부에서 반드시 사용할 일이 없는 변수는 private으로, 나중에 변결할 일이 없다고 판단되는 변수는 final로 선언하는 것이 좋음

### 3.4.2 예제: 불편 객체를 공개할 때 volatile 키워드 사용
~~~java
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;

    public OneValueCache(BigInteger i, BigInteger[] factors) {
        lastNumber = i;
        lastFactors = Arrays.copyOf(factors, factors.length);
    }

    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
            return null;
        else
            return Arrays.copyOf(lastFactors, lastFactors.length);
    }
}
~~~
- 여러 개의 값이 단일하게 한꺼번에 행동해야 한다면 OneValueCache 클래스와 같이 여러 개의 값을 한데 묶는 불변 클래스를 만들어 사용하는 것이 좋음
- 여러 개의 변수를 묶어 사용하고자 할 때, 불변 객체가 아닌 일반 객체를 만들어 사용하면 락을 사용해야 연산의 단일성을 보장할 수 있음
- 불변 객체 내부에 들어 있는 변수 값으 ㄹ변경하면 새로운 불변 객체가 만들어지기 때문에, 기존에 변수 값이 변경되기 전의 불편 객체를 사용하는 다른 스레드는 아무런 이상없이 계속 동작함

~~~java
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    private volatile OneValueCache cache = new OneValueCache(null, null);

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        
        if(factors == null) {
            factors = factors(i);
            cache = new OneValueCache(i, factors);
        }
        
        encodeIntoResponse(resp, factors);
    }
}
~~~
- 스레드 하나가 volatile로 선언된 cache 변수에 새로 생성한 OneValueCache 인스턴스를 설정하면, 다른 스레드에서도 cache 변수에 담긴 새로운 값을 즉시 사용할 수 있음
- VolatileCachedFactorizer 클래스는 변경할 수 없는 상태 값을 여러 개 갖고 있는 불변 객체이며, volatile 키워드를 적용해 가시성을 확보하지 않았음에도 스레드에 안전함

</br>

## 3.5 안전 공개
~~~java
// 안전하지 않은 객체 공개
public Holder Holder;

public void initialize() {
    Holder = new Holder(42);
}
~~~
- 객체에 대한 참조를 public 변수에 넣어 공개하는 것은 객체를 공개하는 안전한 방법이 아님
- 위와 같은 방법으로 객체를 외부에 공개하면 생성 메소드가 채 끝나기도 전에 공개된 객체를 다른 스레드가 사용할 수 있음

### 3.5.1 절절하지 않은 공개 방법: 정상적인 객체도 문제를 일으킨다
~~~java
public class Holder {
    private int n;

    public Holder(int n) { this.n = n; }

    public void assertSanity() {
        if(n != n)
            throw new AssertionError("This statement is false");
    }
}
~~~
- 객체를 공개하는 스레드가 아닌 다른 스레드에서 assertSanity 메소드를 호출하면 AssertionError가 발생할 수 있음
    - n 변수를 final로 설정하면 Holder 객체가 변경 불가능한 상태로 지정되기 때문에 안전하지 않은 방법으로 공개하더라도 문제가 생기지 않도록 막을 수 있음
- 객체를 올바르지 않게 공개하면 2가지 문제가 발생할 수 있음
    - holder 변수에 스테일 상태가 발생할 수 있음
        - holder 변수에 값을 지정한 이후에도 null 값이 지정되어 있거나 예전에 사용하던 참조가 들어가 있을 수 있음
    - 다른 스레드는 모두 holder 변수에서 정상적인 참조 값을 가져갈 수 있지만 Holder 클래스 입장에서는 스테일 상태에 빠질 수 있음
        - 생성자에서 클래스의 변수에 값을 설정하도록 되어있을 때, 상속받은 클래스의 생성자가 호출되기 전 Object 클래스의 생성자가 실행되어 각 변수의 기본값을 설정하면 Object 클래스가 기본적으로 설정한 값을 클래스는 스테일 값으로 가지고 있을 수 있음
- 불변 객체를 공개하는 부분에 동기화 처리를 하지 않았다 해도 아무런 문제가 없음
- final로 선언된 변수에서 변경 가능한 객체가 지정되어 있다면 해당 변수에 들어 있는 객체의 값을 사용하려고 하는 부분을 모두 동기화시켜야 함

### 3.5.3 안전한 공개 방법의 특성
- 객체를 안전하게 공개하려면 해당 객체에 대한 참조와 객체 내부의 상태를 외부의 스레드에게 동시에 볼 수 있어야 함...?
- 올바르게 생성 메소드가 실행되고 난 객체는 다음과 같은 방법으로 안전하게 공개 할 수 있음
    - 객체에 대한 참조를 static 메소드에서 초기화시킨다
    - 객체에 대한 참조를 volatile 변수 또는 AtomicReference 클래스에 보관한다
    - 객체에 대한 참조를 올바르게 생성된 클래스 내부릐 final 변수에 보관한다
    - 락을 사용해 올바르게 막혀 있는 변수에 객체에 대한 참조를 보관한다
- 자바에서 제공하는 스레드 안전한 컬렉션은 스레드 동기화 기능을 가지고 있음
    - Hashtable, ConcurrentMap, synchronizedMap을 사용해 만든 Map 객체를 사용하면 그 안에 보관하고 있는 키와 값 모두를 어느 스레드에서라도 항상 안전하게 사용할 수 있다.
    - 객체를 Vetor, CopyOnWriteArrayList, CopyOnWriteArraySet이나 synchronizedList 또는 synchronizedSet 메소드로 만든 컬렉션은 그 안에 보관하고 있는 객체를 어느 스레드에서라도 항상 안전하게 사용할 수 있다
    - BlockingQueue나 ConcurrentLinkedQueue 컬렉션에 들어있는 객체는 어느 스레드라도 항상 안전하게 사용할 수 있다
- static 변수를 선언할 때 직접 new 연산자로 생성 메소드를 실행해 객체를 생성할 수 있다면 가장 쉬우면서도 안전한 객체 공개 방법임
    ~~~java
    public static Holder holder = new Holder(42);
    ~~~
    - static 초기화 방법은 JVM에서 클래스를 초기화하는 시점에 작업이 모두 진행됨

### 3.5.4 결과적으로 불변인 객체
- 안전하게 공개한 결과적인 불변 객체는 별다른 동기화 작업 없이도 여러 스레드에서 안전하게 호출해 사용할 수 있음
~~~java
public Map<String, Date> lastLogin = 
    Collections.synchronizedMap(new HashMap<String, Date>());
~~~
- Map에 한 번 들어간 Date 인스턴스 값이 더 이상 바뀌지 않는다면 synchronizedMap 메소드를 사용하는 것만으로 동기화 작업이 충분

### 3.5.5 가변 객체
- 가변 객체(mutable object)를 사용할 떄에는 공개하는 부분과 가변 객체를 사용하는 모든 부분에서 동기화 코드를 작성해야만 함
    - 객체 내용이 바뀌는 상황을 정확하게 인식하고 사용할 수 있기 때문에
- 가변 객체를 안전하게 사용하려면 안전하게 공개해야만 하고, 또한 동기화와 락을 사용해 스레드 안정성을 확보해야 함
- 가변성에 따라 객체를 공개할 때 필요한 점은
    - 불변 객체는 어떤 방법으로 공개해도 아무 문제가 없다.
    - 결과적으로 불변인 객체는 안전하게 공개해야 한다.
    - 가변 객체는 안전하게 공개해야 하고, 스레드에 안전하게 만들거나 락을 동기화시켜야 한다.


### 3.5.6 객체를 안전하게 공유하기
- 여러 스레드를 동시에 사용하는 병렬 프로그램에서 객체를 공유해 사용하고자 할 때 가장 많이 사용되는 몇 가지 원칙
    - **스레드 한정** : 스레드에 한정된 객체는 완전하게 해당 스레드 내부에 존재하면서 그 스레드에서만 호출해 사용할 수 있다.
    - **읽기 전용 객체를 공유** : 읽기 전용 객체를 공유해 사용한다면 동기화 작업을 하지 않더라도 여러 스레드에서 언제든지 마음껏 값을 읽어 사용할 수 있다. 물론 읽기 전용이기 때문에 값이 변경될 수 없다. 불변 객체와 결과적으로 불변인 객체가 읽기 전용 객체에 해당
    - **스레드에 안전한 객체를 공유** : 스레드에 안전한 객체는 객체 내부적으로 필수적인 동기화 기능이 만들어져 있기 때문에 외부에서 동기화를 신경 쓸 필요가 없고, 여러 스레드에서 마음껏 호출해 사용할 수 있음
    - **동기화 방법 적용** : 특정 객체에 동기화 방법을 적용해두면 지정한 락을 획득하기 전에는 해당 객체를 사용할 수 없다. 스레드에 안전한 내부 객체에서 사용하는 객체나 공개된 객체 가운데 특정 락을 확보해야 사용할 수 있도록 막혀 있는 객체 등에 동기화 방법이 적용되 있다고 불 수 있다.
