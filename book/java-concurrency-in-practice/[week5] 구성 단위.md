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
![ArrayIndexOutOfBoundsException](/img/ArrayIndexOutOfBoundsException.png)    

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
- 스레드가 블록되면 동작이 멈춰진 다음 블록된 상태(BLOCKED, WAITING, TIMED_WAITING) 가운데 하나를 갖게 됨
- 블로킹 연산은 멈춘 상태에서 특정한 신호를 받아야 계속해서 실행할 수 있는 연산을 말함
	- 외부 신호가 확인되면 스레드의 상태가 다시 RUNNABLE 상태로 넘어가고 다시 시스템 스케줄러를 통해 CPU를 사용할 수 있게 됨
- BlockingQueue 인터페이스의 put(), take()는 Thread.sleep()과 같이 InterruptedException을 발생시킬 수 있음
	- 특정 메소드가 InterruptedException을 발생시킬 수 있다는 것은 해당 메소드가 블로킹 메소드라는 의미 
	- 메소드에 인터럽트가 걸리면 해당 메소드는 대시중인 상태에서 풀려나고자 노력함
- 인터럽트는 스레드가 서로 협력해서 실행하기 위한 방법
- 일반적으로 인터럽트는 특정 작업을 중간에 멈추게 하려는 경우에 사용함
- 인터럽트를 원활하게 처리하도록 만들어진 메소드는 실행 시간이 너무 길어질 때 일정 시간이 지난 이후 실행을 중단할 수 있도록 구성하기 좋음
- 사용하는 메소드가 블로킹 메소드라면 InterruptedException이 발생했을 때 대처할 수 있는 방법을 마련해둬야 함
	- InterruptedException 전달 
		- 받아낸 InterruptedException을 그대로 호출한 메소드에 넘겨버리는 방법
		- 인터럽트에 대한 처리가 복잡하거나 귀찮을 때 쉽게 책임을 떠넘길 수 있음
	- 인터럽트를 무시하고 복구
		- 특정 상황에서는 InterruptedException을 throw할 수 없는 경우가 존재 (Runnable을 구현한 경우)
		- InterruptedException을 catch한 다음, 현재 스레드의 interrupt 메소드를 호출해 살위 호출 메소드가 인터럽트 상황이 발생했음을 알 수 있도록 함
		~~~java
		public class TaskRunnable implements Runnable {
		  BlockingQueue<Task> queue;
		  ...
		  public void run() {
		    try {
		      processTask(queue.take());
		    }catch (InterruptedException e){
		      //인터럽트가 발생한 사실을 저장한다.
		      Thread.currentThread().interrupt();
		    }
		  }
		}
		~~~
- InterruptedException을 catch 후 아무런 대응도 하지 않는 행동은 하지 말아야 함
	- Thread 클래스를 직접 상속하는 경우는 제외한 경우는 제외
		- 인터럽트에 필요한 대응 조치를 취했다고 간주하기 때문


</br>

## 5.5 동기화 클래스
- 객체를 담을 수 있는 컬렉션 클래스이며, 프로듀서와 컨슈머 사이에서 take(), put() 등의 블로킹 메소드를 사용하여 작업 흐름을 조절 할 수 있음
- 상태 정보를 사용하여 스레드 간의 작업 흐름을 조절할 수 있는 모든 클래스를 동기화 클래스(synchronizer)라고 함
- 동기화 클래스의 예 : 세마포어(semaphore), 배리어(barrier), 래치(latch)
- 동기화 클래스에 접근하려는 스레드가 어느 경우에 통과하고 대기하도록 멈추게 해야하는지 결정하는 상태 정보를 갖고 있으며, 그 상태를 변경할 수 있는 메소드를 제공하고, 동기화 클래스가 특정 상태에 진입할 때까지 효과적으로 대기할 수 있는 메소드도 제공함

### 5.5.1 래치
- 스스로가 터미널(terminal) 상태에 이를 때까지의 스레드가 동작하는 과정을 늦출 수 있도록 해주는 동기화 클래스
- 래치가 터미널 상태에 이르기 전에는 관문이 닫혀 있다고 볼 수 있으며, 어떤 스레드도 통과할 수 없음
- 래치가 터미널 상태에 다다르면 관문이 열리고 모든 스레드가 통과함
- 터미널 상태에 다다르면 다시 이전으로 되돌릴 수 없으며, 한번 열린 관문은 계속해서 열린 상태로 유지됨
- 특정한 단일 동작이 완료되기 이전에는 어떤 기능도 동작하지 않도록 막아내야 하는 경우에 적절함
- CountDownLatch는 하나 또는 둘 이상의 스레드가 여러 개의 이벤트가 일어날 때까지 대기할 수 있도록 되어있음
~~~java
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task)
            throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException ignored) {
                    }
                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
}
~~~
- 하나는 시작하는 관문, 하나는 종료하는 관문
- 작업 스레드가 시작되면 가장 먼저 하는일은 시작하는 관문이 열리기를 기다리는 일
	- 특정 이벤트가 발생한 이후에 각 작업 스레드가 동작하도록 제어할 수 있음
- 작업 스레드가 작업을 마치고 가장 마지막에 하는 일은 종료하는 관문의 카운트를 감소 시키는 일
- 모든 작업 스레드가 끝나는 시점이 오면, 메인 스레드는 모든 작업 스레드가 작업을 마쳤다는 것을 알 수 있으며, 작업에 걸린 시간을 쉽게 확인할 수 있음
- n개의 스레드가 동시에 동작할 때 전체 작업 시간이 얼마나 걸리는지 확인하고 싶었기 때문에 스레드 생성과 동시에 작업을 실행하지 않음

### 5.5.2 FutureTask
- FutureTask는 Future를 구현한 인터페이스이며, Future는 결과를 알려주는 연산 작업을 나타냄
- FutureTask가 나타내는 연산 작업은 Callable 인터페이스를 구현하도록 되어 있음
	- 시작 전 대기, 시작됨, 종료됨 3가지 상태를 가짐
- FutureTask의 get()은 작업이 종료됐다면 결과를 즉시 알려줌
	- 종료 상태에 이르지 못했다면 작업이 종료 상태에 이를 때까지 대기하고, 종료된 이후 연산 결과나 예외를 알려줌
- FutureTask는 실제로 연산을 실행했던 스레드에서 만들어 낸 결과 객체를 실행시킨 스레드에게 넘겨줌
	- 결과 객체는 안전한 공개 방법을 통해 넘겨줌
- Executor 프레임웍에서 비동기적인 작업을 실행하고자할 때 사용하며, 시간이 많이 필요한 작업이 있을 때 실제 결과가 필요한 시점 이전에 미리 작업을 실행시켜두는 용도로 사용함
~~~java
public class Preloader {
    private final FutureTask<ProductInfo> future =
            new FutureTask<ProductInfo>(new Callable<ProductInfo>() {
                public ProductInfo call() throws DataLoadException {
                    return loadProductInfo();
                }
            });
    private final Thread thread = new Thread(future);

    public void start() {
        thread.start();
    }

    public ProductInfo get()
            throws DataLoadException, InterruptedException {
        try {
            return future.get();
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof DataLoadException)
                throw (DataLoadException) cause;
            else
                throw launderThrowable(cause);
        }
    }
}
~~~
- Preloader 클래스는 FutureTask를 사용하여 결과 값이 필요한 시점 이전에 시간이 많이 걸리는 작업을 미리 실행시켜둠
- Preloader를 호출한 프로그램에서 실제 제품 정보가 필요한 시점이 되면 Preloader.get 메소드를 호출하면 되고, get 호출시 제품 정보를 모두 가져온 상태라면 즉시 ProductInfo를 제공하고, 데이터를 가져오는 중이라면 작업을 완료할 때까지 대기 후 결과를 제공함
- Calllable 내부에서 예외 발생시 Future.get 에서 ExecutionException으로 한 번 감싼 다음 다시 throw함
- Future.get 에서 ExecutionException 발생시, 원인은 3가지중 하나
	- Callable이 던지는 예외, RuntimeException, Error
	- 원래는 3가지 경우를 분리해 처리해야 하지만, 편의상 아래 예제와 같이 처리
	~~~java
	/**
    * 변수 t의 내용이 Error라면 그대로 throw한다. 
    * 변수 t의 내용이 RuntimeException이라면 그대로 리턴한다.
    * 다른 모든 경우에는 IllegalStateException을 throw한다.
    */
    public static RuntimeException launderThrowable(Throwable t) {
        if (t instanceof RuntimeException)
            return (RuntimeException) t;
        else if (t instanceof Error)
            return (Error) t;
        else 
            throw new IllegalStateException("RuntimeException이 아님", t);
    }
	~~~

### 5.5.3 세마포어
- 카운팅 세마포어(counting semaphore)는 특정 자원이나 특정 연산을 동시에 사용하거나 호출할 수 있는 스레드의 수를 제한하고자 할 때 사용
- 자원 풀(pool)이나 컬렉션의 크기에 제한을 두고자 할 때 유용함
- 가상의 퍼밋(permit)을 만들어 내부 상태를 관리하며, 세마포어를 생성할 때 생성자에 최초로 생성할 퍼밋 수를 넘김
- 외부 스레드는 퍼밋을 요청해 확보하거나, 이전에 확보한 퍼밋을 반납할 수 있음
- 현재 사용할 수 있는 퍼밋이 없는 경우, acquire 메소드는 남는 퍼밋이 생시거나, 인터럽트가 걸리거나, 타임아웃이 걸리기 전까지 대기함
- 자원 풀을 만들 때, 모든 자원을 빌려주고 남아 있는 자원이 없을 때 요청이 들어오는 경우 단순하게 오류를 발생시키고 끝나버리는 정도의 풀은 쉽게 구현이 가능함 (그러나 일반적인 자원 풀을 구현하기 위해 카운팅 세마포어를 사용할 수 있음)
	- 카운팅 세마포어를 이용하여 최초 퍼밋 개수로 원하는 풀의 개수를 지정
	- 풀에서 자원을 할당받아 가려고 할 때에는 먼저 acquire를 호출해 퍼밋을 확보하도록 하고, 다 사용한 자원을 반납하고 난 다음에는 항상 release를 호출해 퍼밋도 반납하도록 함
	- 풀에 자원이 남아있지 않은 경우에 acquire 메소드가 대기 상태에 들어가기 때문에 객체가 반납될 때까지 자연스럽게 대기하게 됨
~~~java
public class BoundedHashSet <T> {
    private final Set<T> set;
    private final Semaphore sem;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        sem.acquire();
        
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded)
                sem.release();
        }
    }

    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
~~~
- 세마포어를 사용하면 어떤 클래스라도 크기가 제한된 컬렉션 클래스로 활용할 수 있음
- 해당하는 컬렉션 클래스가 가질 수 있는 최대 크기의 해당하는 숫자로 초기화
- 크기와 관련된 내용은 모두 BoundedHashSet에서 세마포어를 사용해 관리함

### 5.5.4 배리어
- 배리어(barrier)는 특정 이벤트가 발생할 때까지 여러 개의 스레드를 대기 상태로 잡아둘 수 있다는 측면에서 래치와 비슷함
- 래치와의 차이점은 모든 스레드가 배리어 위치에 동시에 이르러야 관문이 열리고 계속해서 실행 할 수 있다는 점
- 배치는 "이벤트"를 기다리기 위한 동기화 클래스이고, 배리어는 "다른 스레드"를 기다리기 위한 동기화 클래스
- "모두들 오후 6시 출판사 앞에서 만나자. 약속 장소에 도착하면 모두 도착할 때까지 대기하고, 모두 도착하면 모두가 모인 후 어디로 이동할지 생각해보자"
- CyclicBarrier 클래스를 사용하면 여러 스레드가 특정한 배리어 포인트에서 반복적으로 서로 만나는 기능을 모델링 할 수 있고, 커다란 문제 하나를 여러개의 작은 문제로 분리하여 반복하는 병렬 처리 알고리즘을 구현하고자 할 때 적용하기 좋음
- 스레드 각자가 배리어 포인트에 다다르면 await 메소드를 호출하며, await 메소드는 모든 스레드가 배리어 포인트에 도달할 때까지 대기함
- await를 호출하고 시간이 너무 오래 지나 타임아웃이 발생하거나 await 메소드에서 대기하던 스레드에 인터럽트가 걸리면 배리어는 깨짐
	- BrokenBarrierException 발생
- 배리어가 성공적으로 동과하면 await 메소드는 각 스레드별로 배리어 포인트에 도착한 순서를 알려줌 
- 배리어 작업은 Runnable 인터페이스를 구현한 클래스이며, 배리어가 성공적으로 통과된 이후 대기하던 스레드를 놓아주기 직전에 실행됨
- 배리어는 대부분 실제 작업은 모두 여러 스레드에서 병렬로 처리하고, 다음 단계로 넘어가기전 이번 단계에서 계산해야할 내용을 모두 취합하는 등의 작업이 많이 일어나는 시뮬레이션 알고리즘에서 유용하게 사용할 수 있음
~~~java
public class CellularAutomata {
    private final Board mainBoard;
    private final CyclicBarrier barrier;
    private final Worker[] workers;

    public CellularAutomata(Board board) {
        this.mainBoard = board;
        int count = Runtime.getRuntime().availableProcessors();
        this.barrier = new CyclicBarrier(count,
                new Runnable() {
                    public void run() {
                        mainBoard.commitNewValues();
                    }});
        this.workers = new Worker[count];
        for (int i = 0; i < count; i++)
            workers[i] = new Worker(mainBoard.getSubBoard(count, i));
    }

    private class Worker implements Runnable {
        private final Board board;

        public Worker(Board board) { this.board = board; }
        public void run() {
            while (!board.hasConverged()) {
                for (int x = 0; x < board.getMaxX(); x++)
                    for (int y = 0; y < board.getMaxY(); y++)
                        board.setNewValue(x, y, computeValue(x, y));
                try {
                    barrier.await();
                } catch (InterruptedException ex) {
                    return;
                } catch (BrokenBarrierException ex) {
                    return;
                }
            }
        }
    }

    public void start() {
        for (int i = 0; i < workers.length; i++)
            new Thread(workers[i]).start();
        mainBoard.waitForConvergence();
    }
}
~~~
- 시뮬레이션 과정을 병렬화할 때, 일반적으로는 항목별로 연산할 내용을 스레드 단위로 모두 분리시키는 일은 그다지 효율적이지 않음
	- 셀의 개수가 많은 경우가 대부분이므로 스레드 역시 굉장히 많이 만들어질 수 있기 때문
	- 많은 스레드를 관리하는건 전체적인 속도가 크게 떨어질 수 있음

</br>

- Exchanger 클래스는 두 개의 스레드가 연결되는 배리어이며, 배리어 포인트에 도달하면 양쪽의 스레드가 서로 갖고 있던 값을 교환함
- Exchanger는 양쪽 스레드가 서로 대칭되는 작업을 수행할 때 유용함
	- 한쪽 스레드는 데이터 버퍼에 값을 채워 넣고, 다른 스레드는 버퍼에 있는 값을 빼내어 사용하는 일은 하는 경우에 두 개의 스레드를 Exchanger로 묶고 배리어 포인트에 도달할 때마다 데이터 버퍼를 교환함

</br>

## 5.6 효율적이고 확장성 있는 결과 캐시 구현
- 캐시를 대출 만들면 단일 스레드로 처리할 때 성능이 높아질 수는 있겠지만, 나중에 성능의 병목 현상을 확장성의 병목으로 바꾸는 결과를 초래할 수 있음
~~~java
interface Computable <A, V> {
    V compute(A arg) throws InterruptedException;
}

class ExpensiveFunction implements Computable<String, BigInteger> {
    public BigInteger compute(String arg) {
        // 잠시 생각 좀 하고...
        return new BigInteger(arg);
    }
}

public class Memoizer1 <A, V> implements Computable<A, V> {
    @GuardedBy("this") 
    private final Map<A, V> cache = new HashMap<A, V>();
    private final Computable<A, V> c;

    public Memoizer1(Computable<A, V> c) {
        this.c = c;
    }

    public synchronized V compute(A arg) throws InterruptedException {
        V result = cache.get(arg);
        if (result == null) {
            result = c.compute(arg);
            cache.put(arg, result);
        }
        return result;
    }
}
~~~
- HashMap과 동기화 기능을 사용해 구현한 캐시
- Computable를 구현한 ExpensiveFunction 클래스는 결과를 뽑아 내는데 상당한 시간이 소요됨
- 이전 결과를 기억하는 캐시(메모이제이션_memoization) 기능을 추가한 Memoizer1
- HashMap은 스레드 안전하지 않아 compute 메소드에 synchronized 처리를 함
	- 스레드 안전성은 확보하였으나 확장성 측면에서 문제가 생김
	- 특정 시점에 여러 스레드 가운데 하나만 compute 메소드를 실행 할 수 있기 때문.   
	![memoizer1](/img/memoizer1.png)  

</br>  

~~~java
public class Memoizer2 <A, V> implements Computable<A, V> {
    private final Map<A, V> cache = new ConcurrentHashMap<A, V>();
    private final Computable<A, V> c;

    public Memoizer2(Computable<A, V> c) {
        this.c = c;
    }

    public V compute(A arg) throws InterruptedException {
        V result = cache.get(arg);
        if (result == null) {
            result = c.compute(arg);
            cache.put(arg, result);
        }
        return result;
    }
}
~~~
- ConcurrentHashMap은 스레드 안전성을 확보하고 있기 때문에 별다른 동기화 방법을 사용하지 않아도 됨
- Memoizer1 에서 발생한 성능상 문제는 사라졌음
- 2개 이상의 스레드가 동시에 같은 값을 넘기면서 compute 메소드를 호출해 같은 결과를 받아갈 가능성이 있기 때문에 캐시 기능으로 부족한 면이 존재
- 메모이제이션 측면에서는 효율성이 약간 떨어지는 것 뿐
	- 캐시는 같은 값으로 같은 결과를 연산하는 일을 두번 이상 실행하지 않겠다는 것이기 때문
- 캐시할 객체를 한번만 생성해야하는 객체의 캐시의 경우에는 똑같은 결과를 2개 이상 만들어내는 문제점이 안전성 문제로 이어질 수 있음. 
![memoizer2](/img/memoizer2.png)   
- 특정 스레드가 compute 메소드에서 연산을 시작했을 때, 다른 스레드는 현재 어떤 연산이 이뤄지고 있는지 알 수 없음
	- 따라서 그림과 같이 동일한 연산을 시작 할 수 있음

</br>  

~~~java
public class Memoizer3 <A, V> implements Computable<A, V> {
    private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
    private final Computable<A, V> c;

    public Memoizer3(Computable<A, V> c) {
        this.c = c;
    }

    public V compute(final A arg) throws InterruptedException {
        Future<V> f = cache.get(arg);
        if (f == null) {
            Callable<V> eval = new Callable<V>() {
                public V call() throws InterruptedException {
                    return c.compute(arg);
                }
            };
            FutureTask<V> ft = new FutureTask<V>(eval);
            f = ft;
            cache.put(arg, ft);
            ft.run(); // c.compute는 이 안에서 호출
        }
        try {
            return f.get();
        } catch (ExecutionException e) {
            throw LaunderThrowable.launderThrowable(e.getCause());
        }
    }
}
~~~

- 결과를 저장하는 Map을 ConcurrentHashMap<A, Future<V>> 으로 정의
- 원하는 값에 대한 연산 작업이 시작됐는지를 확인함
- 시작된 작업이 없다면 FutureTask를 하나 만들어 Map에 등록하고, 연산 작업을 시작
- 시작된 작업이 있다면 현재 실행중인 연산 작업이 긑나고 결과가 나올 때까지 대기
- 캐시 측면에서는 FutureTask를 사용하여 거의 완벽하게 구현함
- 동시 사용성도 갖고 있으며, 결과를 이미 알고 있다면 중복 계산 과정이 없이 결과를 즉시 가져갈 수 있음
	- 특정 스레드가 연산 작업을 진행중이라면 뒤어어 오는 스레드는 진행 중인 연산 작업의 결과를 기다림
- 여전히 여러 스레드가 같은 값에 대한 연산을 시작 할 수 있지만, memoizer2에 비하여 현저한 수준임.   
![memoizer3](/img/memoizer3.png)    
- Memoizer3 허점은 단일 연산이 아닌 복합 연산을 사용하기 때문, 락을 사용한 단일 연산으로 구성할 수가 없음

</br>

~~~java
public class Memoizer <A, V> implements Computable<A, V> {
    private final ConcurrentMap<A, Future<V>> cache
            = new ConcurrentHashMap<A, Future<V>>();
    private final Computable<A, V> c;

    public Memoizer(Computable<A, V> c) {
        this.c = c;
    }

    public V compute(final A arg) throws InterruptedException {
        while (true) {
            Future<V> f = cache.get(arg);
            if (f == null) {
                Callable<V> eval = new Callable<V>() {
                    public V call() throws InterruptedException {
                        return c.compute(arg);
                    }
                };
                FutureTask<V> ft = new FutureTask<V>(eval);
                f = cache.putIfAbsent(arg, ft);
                if (f == null) {
                    f = ft;
                    ft.run();
                }
            }
            try {
                return f.get();
            } catch (CancellationException e) {
                cache.remove(arg, f);
            } catch (ExecutionException e) {
                throw LaunderThrowable.launderThrowable(e.getCause());
            }
        }
    }
}
~~~
- ConcurrentMap의 putIfAbsent 메소드르 ㄹ통해 단일 연산으로 결과를 저장
- Future 객체를 캐시하는 방법은 캐시 공해를 유발할 수 있음
	- 특정 시점에 시도했던 연산이 취소되거나 오류가 발생하면 Future 객체 역시 취소되거나 오류가 발생했던 상황을 알려줄 것
- Memoizer 클래스는 연산이 취소된 경우엔 캐시에서 해당하는 Future 객체를 제거함
- 캐시 만료 기능은 FutureTask 클래스를 상속받아 만료된 결과인지 여부를 알 수 있는 새로운 클래스를 만들고, 결과 캐시를 주기적으로 조회하여 제거하는 기능을 구현하는 방식으로 해결할 수 있음

</br>

~~~java
@ThreadSafe
public class Factorizer implements Servlet {
    private final Computable<BigInteger, BigInteger[]> c =
            new Computable<BigInteger, BigInteger[]>() {
                public BigInteger[] compute(BigInteger arg) {
                    return factor(arg);
                }
            };
    private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<BigInteger, BigInteger[]>(c);

    public void service(ServletRequest req, ServletResponse resp) {
        try {
            BigInteger i = extractFromRequest(req);
            encodeIntoResponse(resp, cache.compute(i));
        } catch (InterruptedException e) {
            encodeError(resp, "factorization interrupted");
        }
    }
}
~~~
- Memoizer를 사용하여 이전에 계산했던 값을 효율적이면서 확장성 있게 관리함