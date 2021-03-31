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
