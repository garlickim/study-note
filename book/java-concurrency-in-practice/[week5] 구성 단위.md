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


### 5.2.1 ConcurrentHashMap
- ConcurrentHashMap은 HashMap과 같이 해시를 기반으로 하는 Map
- 내부적으로 이전에 사용하던 것과 전혀 다른 동기화 기법을 채택해 병렬성과 확장성이 훨씬 나아짐
	- 이전에는 하나의 락을 사용했기 때문에 특정 시점에 하나의 스레드만 해당 컬렉션을 사용할 수 있었음
- 락 스트라이핑(lock striping)을 사용하여 세밀한 동기화 방법을 사용하여 여러 스레드에서 공유하는 상태에 훨씬 잘 대응함
- 값을 읽어가는 연산은 많은 스레드에서도 동시에 처리할 수 있고, 읽기/쓰기 연산도 동시 처리가 가능함
	- 쓰기 연산은 제한된 갯수만큼 동시에 수행할 수 있음
- ConcurrentHashMap이 만들어낸 Iterator는 ConcurrentModificationException이 발생하지 않음
	- ConcurrentHashMap의 항목을 대상으로 반복문을 실행하는 경우는 따로 락을 걸어 동기화 시킬 필요가 없음
	- Iterator는 미약한 일관성 전략을 취함
		- 미약한 일관성 : 반복문과 동시에 컬렉션의 내용을 변경한다 해도 Iterator를 만들었던 시점의 상황대로 반복을 계속할 수 있음
		- Iterator를 만든 시점 이후에 변경된 내용을 반영해 동작할 수도 있음 (반드시 보장되지는 않음)
- size()는 결과를 리턴하는 시점에 객체의 수가 바뀌었을 수 있기 때문에 결과가 정확한 값이 아닌 추정 값임 (isEmpty()도 비슷)
- get, put, containsKey, remove 등의 핵심 연산의 병렬성과 성능을 위해서라면 size나 isEmpty의 의미가 약간 변할 수 밖에 없음
- 동기화된 Map에서는 지원하지만 ConcurrentHashMap에서는 맵을 독점적으로 사용할 수 있도록 막아버리는 기능을 제공하지 않음
- Hashtable 이나 synchronizedMap 메소드를 사용하는 것보다 훨씬 많은 장점이 있음
- 작업 중인 애플리케이션에서 특정 Map을 완전히 독점하여 사용해야 한다면, 그부분에서 ConcurrentHashMap을 적용하는 건 충분히 신경을 써야함

### 5.2.2 Map 기반의 또 다른 단일 연산
- 이미 구현되어 있지 않은 기능을 사용해야 한다면, ConcurrentHashMap 보다는 ConcurrentMap을 사용하는 편이 나음
	- ConcurrentHashMap은 독점적으로 사용할 수 있는 락이 없기 때문
~~~java
public interface ConcurrentMap<K,V> extends Map<K,V> {
	// key라는 키가 없는 경우에만 value 추가
	V putIfAbsent(K key, V value);

	// key라는 키가 value 값을 갖고 있는 경우 제거
	boolean remove(K key, V value);

	// key라는 키가 oldValue 값을 가지고 있는 경우 newValue로 치환
	boolean replace(K key, V oldValue, V newValue);

	// key라는 키가 들어있는 경우에만 newValue로 치환
	V replace(K key, V newValue);
}
~~~

### 5.2.3 CopyOnWriteArrayList
- 동기화된 List 클래스보다 병렬성을 훨씬 높이고자 만들어짐
- list에 들어있는 값을 Iterator로 불러다 사용하려 할 때, list 전체에 락을 걸거나 list를 복제할 필요가 없음
- 컬렉션의 내용이 변경될 때마다 복사본을 새로 만들어내는 전략을 취함
- Iterator로 값을 뽑아내 사용한다면 Iterator를 뽑아내는 시점의 컬렉션 데이터를 기준으로 동작함
- Iterator 사용중 컬렉션의 값이 변경된다면 별도의 컬렉션이므로 동시 사용성에 문제가 없음
- 반복문에서 반복할 대상 전체를 한꺼번에 락을 거는 대신 개별 항목마다 가시성 확보 목적으로 잠깐의 락을 거는 정도로 충분함
- 컬렉션의 데이터가 변경될 때마다 복사본을 만들어내기 때문에 성능 측면에서는 손해를 볼 수 있음
	- 많은 양의 데이터가 들어있다면 손실이 클 수 있음
- 변경할 때마다 복사하는 컬렉션은 변경 작업보다 반복문으로 읽어내는 일이 훨씬 빈번한 경우에 효과적임

</br>

## 5.3 블로킹 큐와 프로듀서-컨슈머 패턴
- 블로킹 큐는 put과 take라는 핵심 메소드를 갖고 있음
	- 큐가 가득차 있다면 put 메소드는 값을 추가할 공간이 생길 때까지 대기
	- 큐가 비어 있다면 take 메소드는 큐에 값이 들어올 때까지 대기
- 블로킹 큐는 프로듀서-컨슈머 패턴을 구현할 때 사용하기 좋음
- 프로듀서-컨슈머 패턴은 작업을 만들어내는 주체와 처리하는 주체를 분리시키기 위한 설계 방법
- 작업 생성 주체와 처리 주체가 분리되기 때문에 개발 과정을 좀 더 명확하게 단순화 시킬 수 있고, 프로듀서와 컨슈머 각각 감당할 수 있는 부하를 조절할 수 있음
- 프로듀서는 작업을 새로 만들어 큐에 쌓아두고, 컨슈머는 큐에 쌓여있는 작업을 가져다 처리하는 구조
- 프로듀서는 어떤 컨슈머가 몇개나 동작하고 있는지 신경 쓸 필요가 없음
- 블록킹 큐를 사용한 여러개의 프로듀서, 여러개의 컨슈머가 작동하는 패턴을 쉽게 구현할 수 있음
	- 큐와 함께 스레드 풀을 사용하는 경우
	- Executor 프레임웍 
- 접시를 닦는 2명의 모습을 비유하여 설명 (접시를 닦는 사람, 접시를 건조시키는 사람)
- '프로듀서'나 '컨슈머'는 어디까지나 상대적인 개념
- 블로킹 큐를 사용하면 값이 들어올 때까지 take 메소드가 알아서 멈추고 대기하기 때문에 컨슈머 코드를 작성하기가 편리함
- 서버 애플리케이션을 놓고 보면 클라이언트의 수가 적거나 요청량이 많지 않아 컨슈머가 '놀고 있는' 상황이 정상적일 수 있음
- 반면, 프로듀서와 컨슈머의 비율이 적절하지 않다고 판단할 수 있음
	- 하드웨어 자원을 효율적으로 사용하지 못하는 것으로 판단

</br>

- 큐에 크기 제한을 두면 
	- 큐에 빈공간이 생길 때까지 put 메소드는 대기하기 때문에 프로듀서 코드를 작성하기가 편해짐
	- 컨슈머가 작업을 처리하는 속도에 프로듀서가 맞춰야 하며, 컨슈머가 처리하는 양보다 많은 작업을 만들어낼 수 없음
- offer 메소드는 큐에 값을 넣을 수 없을 때 대기하지 않고, 공간이 모자라 추가할 수 없다는 오류를 알려줌
	- 프로듀서가 작업을 많이 만들어 과부하에 이르는 상태를 효과적으로 처리할 수 있음
		- 부하를 분배하거나, 작업할 내용을 직렬화하여 디스크에 임시 저장하거나, 프로듀서 스레드 수를 동적으로 줄이거나, etc..
- 블로킹 큐는 애플리케이션이 안정적으로 동작하도록 만들고자 할 때 요긴하게 사용할 수 있는 도구
- 블로킹 큐를 사용하면 처리할 수 있는 양보다 훨씬 많은 작업이 생겨 부하가 걸리는 사황에서 작업량을 조절해 안정적인 서비스를 하도록 유도함

</br>

- 프로듀서-컨슈머 패턴을 사용하면 각각의 프로그램 코드는 서로를 연결하는 큐를 기준으로 분리되지만, 동작 자체를 큐를 사이에 두고 서로 간접적으로 연결되어 있음
- 블로킹 큐를 사용해 설계 과정에서부터 프로그램에 자원 관리 기능을 추가하는 것을 고려해야함
- 프로그램이 블로킹 큐를 쉽게 적용할 수 없는 모양새를 가졌다면 세마포어(Semaphore)를 사용해 사용하기 적합한 데이터 구조를 만들어야 함
- BlockingQueue 인터페이스를 구현한 클래스
	- FIFO 형태의 LinkedBlockingQueue, ArrayBlockingQueue
		- 병렬 프로그램 환경에서는 LinkedList나 ArrayList에서 동기화된 list 인스턴스를 뽑아 사용하는 것보다 성능이 좋음
	- 우선 순위를 기준으로 동작하는 PriorityBlockingQueue 
		- Comparator 인터페이스를 사용해 원하는 대로 정렬 가능
	- SynchronousQueue
		- 큐에 항목이 쌓이지 않으며, 큐 내부에 값을 저장할 수 있도록 공간을 할당하지도 않음
		- 큐에 값을 추가하려는 스레드나 값을 읽어가려는 스레드의 큐를 관리함
		- 프로듀서와 컨슈머 사이에 접시를 쌓아 둘 공간이 전혀 없고, 프로듀서는 컨슈머의 손에 접시를 직접 넘겨주는 구조
		- 프로듀서에서 컨슈머로 데이터가 넘어가는 순간은 굉장히 짧아진다는 특성을 가짐
		- 데이터를 직접 넘겨주기 때문에 넘겨준 데이터와 관련된 컨슈머의 정보를 알 수 있음
		- 큐에 추가된 데이터를 보관할 공간이 없기 때문에 put(), take() 메소드를 호출하면 메소드의 상대편 측에 해당하는 메소드를 다른 스레드가 호출할 때까지 대기함
		- 데이터를 넘겨 받을 수 있는 충분한 개수의 컨슈머가 대기하고 있는 경우에 사용하기 좋음


### 5.3.1 예제: 데스크탑 검색
~~~java
public class FileCrawler implements Runnable {
    private final BlockingQueue<File> fileQueue;
    private final FileFilter fileFilter;
    private final File root;
    ...

    public void run() {
        try {
            crawl(root);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void crawl(File root) throws InterruptedException {
        File[] entries = root.listFiles(fileFilter);
        if (entries != null) {
            for (File entry : entries)
                if (entry.isDirectory())
                    crawl(entry);
                else if (!alreadyIndexed(entry))
                    fileQueue.put(entry);
        }
    }
}

public class Indexer implements Runnable {
    private final BlockingQueue<File> queue;

    public Indexer(BlockingQueue<File> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            while (true)
                indexFile(queue.take());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
~~~
- FileCrawler 프로그램은 디스크에 들어 있는 디렉토리 계층 구조를 따라가면서 검색 대상 파일이라고 판단되는 파일의 이름을 작업 큐에 넣는 프로듀서 역할을 담당
- Indexer 프로그램은 작업 규에 있는 파일 이름을 뽑아내 해당 파일의 내용을 색인하는 컨슈머의 역할을 담당
- 멀티스레드를 사용하는 경우, 프로그램의 세부 기능을 쉽게 컴포넌트화 할 수 있음
- 코드 가독성을 높이고, 재사용성을 높임
	- 분리된 각 클래스는 자신이 담당하는 한가지 일만 처리하고, 두 클래스 사이의 작업 흐름은 블로킹 큐가 조절하기 때문에 코드가 간결하고 가독성이 높음
- 프로듀서의 작업은 디스크나 네트웍 I/O에 시간을 많이 소모하고, 컨슈머는 CPU를 많이 사용하는 특성이 있다면 단일 스레드의 순차적 실행보다 성능이 높아질 수 있음
- 프로듀서와 컨슈머가 멀티스레드로 동작하는 수준에서 차이가 있다면, 프로듀서와 컨슈머가 긴밀하게 연결되어 있을수록 병렬 처리 성능이 떨어짐
~~~java
    public static void startIndexing(File[] roots) {
        BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);

        FileFilter filter = new FileFilter() {
            public boolean accept(File file) {
                return true;
            }
        };


        for (File root : roots)
            new Thread(new FileCrawler(queue, filter, root)).start();


        for (int i = 0; i < N_CONSUMERS; i++)
            new Thread(new Indexer(queue)).start();

    }
~~~
- 파일을 찾아내는 기능과 내용 색인하는 모듈 여러개를 각각의 스레드를 통해 동작시킴
- 색인을 담당하는 컨슈머 스레드는 계속해서 작업을 기다리느라 종료되지 않기 때문에, 파일을 모두 찾아내 처리했음에도 프로그램이 종료되지 않는 상황이 발생함
- 프로듀서-컨슈머 패턴을 사용하는 대부분의 경우는 Executor 작업 실행 프레임웍을 사용해 표현할 수 있음
	- Executor 내부에서 프로듀서-컨슈머 패턴을 사용하기 있기 때문

### 5.3.2 직렬 스레드 한정
- java.util.concurrent 패키지에 들어 있는 블로킹 큐 관련 클래스는 모두 프로듀서 스레드에서 객체를 가져와 컨슈머 스레드에 넘겨주는 과정이 세심하게 동기화되어 있음
- 프로듀서-컨슈머 패턴과 블로킹 큐는 mutable object(가변객체)를 사용할 때 객체의 소유권을 프로듀서에서 컨슈머로 넘기는 과정에서 직렬 스레드 한정(serial thread confinement) 기법을 사용함
	- 스레드에 한정된 객체는 특정 스레드 하나만이 소유권을 가짐
	- 객체를 안전한 방법으로 공개하면 객체에 대한 소유권 이전(transfer)할 수 있음
	- 이전 받은 컨슈머 스레드가 객체에 대한 소유권을 완전히 갖게됨
	- 원래 소유권을 갖고 있던 스레드는 소유권을 완전히 잃어, 새로운 스레드 내부에 객체가 완전히 한정됨
- 객체 풀(object pool)은 직렬 스레드 한정 기법을 잘 활용한 예
- 항상 소유권을 이전받는 스레드는 단 하나여야 한다는 점을 주의해야함
	- 블로킹 큐를 사용하면 이런점을 정확하게 지킬 수 있음
	- ConcurrentMap의 remove()를 사용하거나 AtomicReference의 compareAndSet() 을 사용하는 경우에도 약간의 추가 작업만 해준다면 원할하게 처리할 수 있음

### 5.3.3 덱, 작업 가로채기
- Deque과 BlockingDeque은 각각 Queue와 BlockingQueue를 상속받은 인터페이스
- Deque은 앞/뒤 어느 쪽에도 객체를 쉽게 삽입하거나 제거할 수 있도록 준비된 큐
- ArrayDeque과 LinkedBlockingDeque 클래스가 존재
- 작업 가로채기(work stealing)라는 패턴을 적용할 때에는 덱을 그대로 가져다 사용할 수 있음
- 작업 가로채기 패턴은 모든 컨슈머가 각자의 덱을 갖음
- 특정 컨슈머가 자신의 덱에 있던 모든 작업을 처리하고 나면, 다른 컨슈머 덱에 쌓여있는 맨 뒤에 추가된 작업을 가로채 가져올 수 있음
	- 맨 앞의 작업을 가져가려는 원래 작업 소유자와 경쟁이 일어나지 않음
- 모든 컨슈머가 하나의 큐를 바라보지 않기 때문에, 규모가 큰 시스템을 구현하기에 적당함
- 가로채기 패턴은 컨슈머가 프로듀서의 역할도 갖고 있는 경우에 적용하기 좋음 (하나의 작업을 처리한 후 더 많은 작업이 생길 수 있는 상황)
	- GC 도중 힙을 마킹하는 작업과 같이 대부분의 그래프 탐색 알고리즘을 구현할 때, 작업 가로채기 패턴을 적용하면 멀티스레드를 사용해 병렬화할 수 있음
- 다른 작업 스레드의 덱을 보고 밀린 작업이 있다면 가져다 처리해 자신의 덱이 비었다고 쉬는 스레드가 없도록 관리함

</br>

## 5.4 블로킹 메소드, 인터럽터블 메소드
- 

