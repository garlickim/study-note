# 구성 단위
- 병렬 프로그래밍 과정에서 유용하게 사용할 수 있는 자바에 포함된 클래스와 디자인 패턴에 대해서 다룰 예정

</br>

## 5.1 동기화된 컬렉션 클래스
- 동기화된 컬렉션 클래스의 대표는 Vector와 Hashtable
- JDK 1.2 이후로는 Collections.synchronizedXXX 메소드를 사용해 동기화된 클래스를 만들어 사용할 수 있음
- 위의 클래스들은 public 으로 선언된 모든 메소드를 클래스 내부에 캡슐화해 내부 값을 한번에 한 스레드만 사용할 수 있도록 제어 & 스레드 안전성을 확보함
~~~java
// Vector의 trimToSize()
public synchronized void trimToSize() {
    modCount++;
    int oldCapacity = elementData.length;
    if (elementCount < oldCapacity) {
        elementData = Arrays.copyOf(elementData, elementCount);
    }
}

// Collections.synchronizedMap의 values()
public Collection<V> values() {
    synchronized (mutex) {
        if (values==null)
            values = new SynchronizedCollection<>(m.values(), mutex);
        return values;
    }
}
~~~

### 5.1.1 동기화된 컬렉션 클래스의 문제점
- 여러 개의 연산을 묶어 하나의 단일 연산처럼 활용해야 할 필요성이 발생할 때, 동기화된 컬렉션을 사용하면 따로 락이나 동기화 기법을 사용하지 않는다 해도 대부분의 기능이 모두 스레드 안전함
- **그러나**, 여러 스레드가 해당 컬렉션 하나를 놓고 동시에 그 내용을 변경하려 한다면 컬렉션 클래스가 상식적이고 올바른 방법으로 동작하지 않을 수 있음
~~~java
public static Object getLast(Vector list) {
	int lastIndex = list.size() - 1;
	return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
	int lastIndex = list.size() - 1;
	list.remove(lastIndex);
}
~~~
- 위의 두 가지 메소드를 호출해 사용하는 외부 프로그램의 입장에서 보면 A스레드와 B스레드가 동시에 getLast(), deleteLast()를 각각 호출할 때 ArrayIndexOutOfBoundsException이 발생할 수 있음
// 그림삽입
- 이런 문제가 발생하지만, Vector에 입장에서는 스레드 안전성에 문제가 없는 상태
	- 단순히 없는 항목을 요청했기에 예외가 발생하는 것

</br>

~~~java
public static Object getLast(Vector list) {
	synchronized (list) {
		int lastIndex = list.size() - 1;
		return list.get(lastIndex);
	}
}

public static void deleteLast(Vector list) {
	synchronized (list) {
		int lastIndex = list.size() - 1;
		list.remove(lastIndex);
	}
}
~~~
- 동기화된 컬렉션 클래스는 대부분 클라이언트 측 락을 사용할 수 있도록 만들어져 있기 때문에 컬렉션 클래스가 사용하는 락을 함께 사용한다면 새로 추가하는 기능을 컬렉션 클래스에 들어있는 다른 메소드와 같은 수준으로 동기화 시킬 수 있음

</br>

~~~java
for(int i = 0; i < vector.size(); i++) {
	doSomething(vector.get(i));
}
~~~
- 위의 예제는 size(), get()을 호출하는 사이에 vector의 값 변경 기능을 호출하지 않을 것이라는 어설픈 가정하게 만들어졌기 때문에 오류가 발생 가능
- 멀티스레드 환경에서 vector 내부의 값을 변경하는 상황에서 위와 같은 반복 기능을 사용한다면, ArrayIndexOutOfBoundsException이 발생 가능

</br>

~~~java
synchronized (vector) {
	for(int i = 0; i < vector.size(); i++) {
		doSomething(vector.get(i));
	}
}
~~~
- 클라이언트 측 락을 사용하면 예외 상황이 발생하지 않도록 정확하게 동기화시킬 수 있음
- 성능의 측면에서는 약간의 손해가 생길 가능성이 존재
	- 반복문을 실행하는 동안 락이 걸려있기 때문에, vector 클래스 내부 값을 변경하려는 모든 스레드는 대기 상태로 들어가기 때문
	- 즉, 반복문이 실행되는 동안 동시 작업을 막아버리기 때문에 병렬 프로그래밍의 장점을 잃어버림

### 5.1.2 Iterator와 ConcurrentModificationException
- 새로 추가된 컬렉션 클래스 역시 다중 연산을 사용할 때에 발생하는 문제점을 해결하지는 못함
- 동기화된 컬렉션 클래스에서 만들어낸 Iterator를 사용한다 해도 다른 스레드가 같은 시점에 컬렉션 클래스 내부의 값을 변경하는 작업을 처리하지는 못하게 만들어져 있고, 즉지 멈춤(file-fast)의 형태로 반응하도록 되어있음
	- 즉시 멈춤이란 반복문을 실행하는 도중 컬렉션 내부의 값을 변경하는 행위가 일어나면 ConcurrentModificationException 예외를 발생시키고 멈추는 처리 방법
- 컬렉션 클래스는 내부에 값 변경 횟수를 카운트하는 변수를 가지고 있음
	- 반복문이 실행되는 동안 변경 횟수 값이 바뀌면 hasNext(), next()에서 ConcurrentModificationException를 발생시킴
	- 변경 횟수를 세는 확인하는 부분이 적절하게 동기화가 되어있지 않아 스테일 값을 사용할 수도 있으며, 변경 작업 여부를 모를 수도 있음
	- 변경 작업이 있었다는 상황을 확인하는 기능에 정확한 동기화 기법을 적용하지 않았다고 볼 수 있음
~~~java
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
 ...
// ConcurrentModificationException이 발생할 수 있음
for(Widget w : widgetList) {
	doSomething(w);
}
~~~

~~~java
List<String> stringList = Collections.synchronizedList(new ArrayList());
Iterator var2 = stringList.iterator();

while(var2.hasNext()) {
    String w = (String)var2.next();
    doSomething(w);
}
~~~
- foreach를 사용한 코드이지만, 컴파일시 Iterator의 hasNext() 또는 next()를 호출하면서 진행되는 코드로 변경됨
- 반복문을 실행할 때 ConcurrentModificationException이 발생할 수 있음
- 반복문 전체를 적절한 락으로 동기화를 시키는 방법으로 ConcurrentModificationException를 방지해야 함
- 반복문을 실행하는 코드 전체를 동기화 시키는 방법이 훌륭한 방법은 아님
	- 컬렉션에 많은 값이 들어 있거나 작업 시간이 오래걸리는 작업이라면, 클래스 내부 값을 사용하고자 하는 스레드가 오랜 시간 대기 상태로 유지됨
	- 반복문에서 락을 잡고 있는 상황이고, 반복문 안에 특정 메소드가 또 다른 락을 확보해야하는 상황이라면 데드락(deadlock)이 발생할 가능성이 높음
- 소모상태(starvation)나 데드락의 위험이 있는 상태에서 컬렉션 클래스를 오랜 시간동안 락을 잡는다면, 어플리케이션 전체의 확장성을 해침
- 반복문에서 락을 오래 잡고 있을수록, 락을 확보하고자 하는 스레드가 대기 상태에 많이 쌓일 수 있고, 대기 상태에 스레드가 많아질수록 cpu 사용률이 급격하게 증가할 수 있음
- 반복문에서 락을 잡는 것과 비슷하게 clone을 사용하여 복사본을 사용하는 방법도 있음
	- 복사본을 사용하는 방식 또한 컬렉션의 데이터 갯수, 작업 시간, 컬렉션의 기능에 비해 반복 기능을 얼마나 빈번하게 사용하는지, 응답성, 실행 속도 등의 여러가지 요구 사항을 고려하여 적절하게 사용해야 함

### 5.1.3 숨겨진 Iterator
- 락을 걸어 동기화시키면 Iterator를 사용할 때 ConcurrentModificationException이 발생하지 않도록 제어할 수 있음
	- 컬렉션을 공유해 사용하는 모든 부분에서 동기화를 맞춰야 함
~~~java
public class HiddenIterator {
    @GuardedBy("this") 
    private final Set<Integer> set = new HashSet<Integer>();

    public synchronized void add(Integer i) {
        set.add(i);
    }

    public synchronized void remove(Integer i) {
        set.remove(i);
    }

    public void addTenThings() {
        Random r = new Random();
        for (int i = 0; i < 10; i++)
            add(r.nextInt());
        System.out.println("DEBUG: added ten elements to " + set);
    }
}
~~~
- Iterator 가 숨겨진 케이스
- sysout 구문에서 Iterator를 사용
	- 두 개의 문장을 + 연산자로 연결하는 것은 컴파일러에 의해 StringBuilder.append(Object) 메소드를 사용하는 코드로 변환됨
	- 코드 변환중 toString 메소드를 호출하게 되어 있고, 컬렉션 클래스의 toString()은 iterator를 사용하여 출력 문자열을 만들어냄
- addTenThings() 실행 도중, 메시지 출력을 위해 set 변수의 Iterator를 사용하기 때문에 ConcurrentModificationException이 발생할 가능성이 존재
- 스레드 안전성을 확보하지 못한 클래스
- sysout 구문에서 set 변수를 사용하기 전에 락을 확보해 동기화 시켜야 스레드 안전성을 확보할 수 있음
- 개발자는 상태 변수와 상태 변수의 동기화를 맞춰주는 락이 멀리 떨어져 있을수록 동기화를 맞춰야 한다는 필요성을 잊기 쉬움
- HashSet이 아닌 synchronizedSet 메소드로 동기화된 컬렉션을 사용하면 동기화가 이루어져 있기 때문에 위와 같은 문제는 발생하지 않음
- 클래스 내부에서 필요한 변수를 모두 캡슐화하면 그 상태를 보존하기가 훨씬 편리한 것처럼 동기화 기법을 클래스 내부에 캡슐화하면 동기화 정책을 적용하기 쉬움
- 컬렉션 클래스의 내부적으로 iterator를 사용하는 모든 메소드에서 ConcurrentModificationException이 발생할 가능성이 존재
	- hashCode, equals, removeAll, retainAll, etc..

</br>

## 5.2 병렬 컬렉션
- 동기화된 컬렉션 클래스는 컬렉션 내부 변수 접근하는 통로를 일련화하여 스레드 안전성을 확보함
	- 여러 스레드가 한번에 동기화된 컬렉션을 사용하려하면 동시 사용성은 상당 부분 손해를 봄
- 병렬 컬렉션은 여러 스레드에서 동시에 사용할 수 있도록 설계되어 있음
- HashMap의 병렬성을 확보한 ConcurrentHashMap이 존재
- ArrayList와 동일하지만, 컨텐츠 값을 전달할 때 복사본을 전달하는 CopyOnWriteArrayList가 존재
	- 추가되어 있는 개체 목록을 반복시키며 열람하는 연산의 성능을 최우선으로 구현
- putIfAbsent, replace, confitional remove 등을 정의하는 ConcurrentMap이 존재
- 기존에 사용하던 동기화 컬렉션 클래스를 병렬 컬렉션으로 교체하는 것만으로도 별다른 위험 요소 없이 전체적인 성능을 상당히 끌어 올릴 수 있음
- Queue, BlockingQueue
	- ConcurrentLinkedQueue : FIFO 형태의 queue
	- PriorityQueue : 우선순위에 따라 큐에 쌓여있는 항목이 추출되는 순서가 변경됨
	- BlockingQueue	: 큐에 항목을 추가하거나 뽑아낼 때 상황에 따라 대기할 수 있도록 구현됨
		- 큐가 비어있다면 큐에서 값을 뽑아내는 연산은 큐에 새로운 항목이 추가될 때까지 대기함
		- 큐가 가득 차있다면 큐에 빈자리가 생길 때까지 대기함
		- 프로듀서-컨슈머 패턴을 구현할 때 굉장히 편리하게 사용할 수 있음
- ConcurrentSkipListMap : SortedMap의 병렬성을 높인 형태
- ConcurrentSkipListSet : SortedSet의 병렬성을 높인 형태
- SortedMap : TreeMap을 synchronizedMap으로 처리해 동기화 시킨 컬렉션과 같음
- SortedSet : TreeSet을 synchronizedMap으로 처리해 동기화 시킨 컬렉션과 같음

</br>

### 5.2.1 ConcurrentHashMap
- ConcurrentHashMap은 HashMap과 같이 해시를 기반으로 하는 Map
- 내부적으로 이전에 사용하던 것과 전혀 다른 동기화 기법을 채택해 병렬성과 확장성이 훨씬 나아짐
	- 이전에는 하나의 락을 사용했기 때문에 특정 시점에 하나의 스레드만 해당 컬렉션을 사용할 수 있었음
- 락 스트라이핑(lock striping)을 사용하여 세밀한 동기화 방법을 사용하여 여러 스레드에서 공유하는 상태에 훨씬 잘 대응함
- 값을 읽어가는 연산은 많은 스레드에서도 동시에 처리할 수 있고, 읽기/쓰기 연산도 동시 처리가 가능함
	- 쓰기 연산은 제한된 갯수만큼 동시에 수행할 수 있음
