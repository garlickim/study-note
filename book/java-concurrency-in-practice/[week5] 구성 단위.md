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











