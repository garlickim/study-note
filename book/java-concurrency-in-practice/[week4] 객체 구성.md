# 객체 구성
- 스레드 안전성을 확보한 개별 컴포넌트를 가져다가 안전한 방법을 동원해 서로 연결해 사용한다면 규모 있는 컴포넌트나 프로그램을 좀 더 쉽게 작성할 수 있음

</br>

## 4.1 스레드 안전한 클래스 설계 
- 객체가 갖고 있는 여러가지 정보를 해당 객체 내부에 숨겨두면 전체 프로그램을 다 뒤져볼 필요없이 객체 단위로 스레드 안전성이 확보되어있는지 확인할 수 있음
- 스레드 안전성을 확보하도록 클래스를 설계하기 위해 고려하는 방법
	- 객체의 상태를 보관하는 변수가 어떤 것인가?
	- 객체의 상태를 보관하는 변수가 가질 수 있는 값이 어떤 종류, 어떤 범위에 해당하는가?
	- 객체 내부의 값을 동시에 사용하고자 할 때, 그 과정을 관리할 수 있는 정책
- 객체 내부의 변수가 모두 기본 변수형으로 만들어져 있다면 해당 변수만으로 객체의 상태를 완전하게 표현할 수 있음
~~~java
@ThreadSafe
public final class Counter {
	@GuardedBy("this") private long value = 0;

	public synchronized long getValue() {
		return value;
	}

	public synchronized long increment() {
		if(value == Long.MAX_VALUE) 
			throw new IllegalStateException("count overflow");
		return ++value;
	}
}
~~~
- 객체의 상태는 항상 객체 내부의 변수를 기반으로 함
- Counter 클래스는 value 변수만 보면 완벽하게 알 수 있음
- 단, 클래스 내부에서 A 객체 내부에 B 클래스를 참조하고 있다면 B 객체 내부에 들어있는 변수의 조합까지 A 객체가 가질 수 있는 전체 상태 범위에 포함시켜야 함
- 동기화 정책에는 객체의 불변성, 스레드 한정, 락 등을 어떻게 적절하게 활용해 스레드 안전성을 확보할 수 있으며 어떤 변수를 어떤 락으로 막아야 하는지 등의 내용을 명시해야 함

### 4.1.1 동기화 요구사항 정리
- 여러 스레드가 동시에 클래스를 사용하려 하는 상황에서 클래스 내부의 값을 안정적인 상태로 유지할 수 있다면, 스레드 안정성을 확보했다고 할 수 있음
- 항상 객체와 변수가 가질 수 있는 가능한 값의 범위를 상태 범위(state space)라고 함
- 상태 범위가 좁으면 좁을수록 객체의 논리적인 상태를 파악하기 쉬움
	- 사용할 수 있는 부분마다 final을 지정해두면 상태 범위를 크게 줄여주기 때문에 논리 범위를 줄여줌
- 클래스가 특정 상태를 가질 수 없도록 구현해야 한다면, 해당 변수는 클래스 내부에 숨겨둬야만 함
	- 변수를 숨겨두지 않으면 외부에서 클래스가 "올바르지 않다"고 정의하는 값을 지정할 수 있음
- 특정한 연산을 실행했을 때, 올바르지 않은 상태 값을 가질 가능성이 있다면 해당 연산은 단일 연산으로 구현해야 함
- 여러 개의 변수를 통해 클래스의 상태가 올바른지 아닌지를 정의한다면 단일 연산으로 구현해야 함
	- 서로 연관된 값은 단일 연산으로 한번에 읽거나 변경해야 한다는 의미
- 상태 범위에 두 개 이상의변수가 연결되어 동시에 관여하고 있다면 이런 변수를 사용하는 모든 부분에서 락을 사용해 동기화를 맞춰야 함
- 객체가 가질 수 있는 값의 범위와 변동 폭을 정확하게 인식하지 못한다면, 스레드 안전성을 완벽하게 확보할 수 없음

### 4.1.2 상태 의존 연산
- 클래스가 가질 수 있는 값의 범위와 값이 변화하는 여러 가지 조건을 살펴보면 어떤 상황이어야 클래스가 정상적인지 정의 할 수 있음
- 현재 조건에 따라 동작 여부가 결정되는 연산을 상태 의존(state-dependent) 연산이라고 함
- 병렬 프로그램을 작성할 때는 상태가 올바르게 바뀔 경우를 대비하고 기다리다가 실제 연산을 수행하는 방법도 생각할 수 있음
- 자바에 wait와 notify 명령은 본질적으로 락을 사용하는 것과 굉장히 밀접한 관련이 있고, wait와 notify를 사용하면 특정 상태가 원하는 조건에 다다를 때까지 효율적으로 기다릴 수 있음
	- 올바르게 사용하기가 쉽지 않으니 주의가 필요
- 어떤 동작을 실행하기 전에 특정한 조건을 만족할 때까지 기다리도록 프로그래밍하고자 한다면, 세마포어나 블록킹 큐와 같은 여러가지 라이브러리를 사용하는 편이 간단하고 안전함

### 4.1.3 상태 소유권
- 변수를 통해 객체의 상태를 정의하고자 할 때에는 해당 객체가 실제로 '소유하는' 데이터만을 기준으로 삼아야 함
- 자바 언어의 경우, 객체를 공유하는 데 있어 오류가 발생하기 쉬운 부분을 GC가 대부분 알아서 조절해주기 떄문에 소유권 개념이 C++에 비해 불명한 경우가 많음
- 대부분 소유권과 캡슐화 정책은 함께 고려하는 경우가 많음
	- 캡슐화는 클래스 내부에 객체와 함께 상태 정보를 숨기기 때문에 객체 상태에 대한 소유권이 있음
	- 특정 변수에 대한 소유권을 갖고 있기 때문에 특정 변수의 상태가 올바르게 유지되도록 조절하는 락 구조가 어떻게 움직이지에 대해서도 소유권을 가짐
- 특정 변수를 객체 외부에 공개하면 해당 변수에 대한 통제권을 어느정도 잃음
	- 즉, 객체를 공개하면 '공동 소유권'을 가지게 됨
- 클래스는 인자로 넘겨받은 객체 대한 소유권을 갖지 않는다는게 일반적인 원칙이지만, 넘겨받은 객체의 소유권을 확보하도록 메소드를 작성하면 소유권을 확보할 수도 있음
	- 동기화된 컬렉션 라이브러리에서 사용하는 팩토리 메소드가 전형적인 예
- 컬렉션 클래스에서는 '소유권 분리'의 형태를 사용하는 경우도 많음
- 소유권 분리는 컬렉션 클래스를 놓고 볼 때, 컬렉션 내부 구조에 대한 소유권은 컬렉션 클래스가 갖고, 컬렉션에 추가된 객체에 대한 소유권은 컬렉션을 호출해 사용하는 클라이언트 프로그램이 갖는 구조
- ServletContext에 넣어둔 객체를 사용할 때에는 반드시 스레드 안전성을 확보하거나, 불변 객체의 형태를 갖거나 지정된 락을 사용해 동시 사용을 막는 등의 동기화 작업을 거쳐야 함

</br>

## 4.2 인스턴스 한정
- 객체를 적정하게 캡슐화하는 것으로도 스레드 안전성을 확보할 수 있는데, 이런 경우 흔히 '한정'이라고 부르기도 하는 '인스턴스 한정' 기법을 활용하는 셈
- 특정 객체가 다른 객체 내부에 완벽하게 숨겨져 있다면 해당 객체를 활용하는 모든 방법을 한눈에 확실하게 파악할 수 있고, 객체 외부에서도 사용할 수 있는 상황보다 훨씬 간편하게 스레드 안전성을 분석해 볼 수 있음
- 데이터를 객체 내부에 캡슐화해 숨겨두면 숨겨진 내용은 해당 객체의 메소드에서만 사용할 수 있기 때문에 숨겨진 데이터를 사용하고자 할 때에는 항상 지정된 형태의 락이 적용되는지 쉽고 정확하게 파악할 수 있음
- 객체를 특정 클래스 인스턴스에 한정(클래스 내부에 private으로 선언된 변수)
- 문법적으로 블록 내부에 한정(블록 내부의 로컬 변수)
- 특정 스레드에 한정(특정 스레드 내부에서는 여러 메소드들을 넘어다닐 순 있지만, 다른 스레드로는 넘겨주지 않는 객체)
~~~java
@ThreadSafe
public class PersonSet {
	@GuardedBy("this")
	private final Set<Person> mySet = new HashSet<Person>();

	public synchronized void addPerson(Person p) {
		mySet.add(p);
	}

	public synchronized boolean containsPerson(Person p) {
		return mySet.contains(p);
	}
}
~~~
- PersonSet 클래스는 내부에 정의된 변수가 threadsafe 하지 않지만, 클래스 전체로 보면 스레드에 안전하게 구현되어 있음
	- HashSet 자체는 스레드에 안전한 객체가 아니기 때문
- HashSet으로 만든 mySet은 private으로 선언되어 외부에 직접적으로 노출되지 않음
- addPerson(), containsPerson() 은 synchronized 키워드를 통해 PersionSet 객체에 락을 걸음
- PersonSet 객체는 스레드 안정성을 확보했다고 볼 수 있음
- Person 객체에 대한 스레드 안정성은 확보되지 않은 상태로 볼 수 있으며, 만약 Person 객체가 갖고 있는 데이터가 변경될 수 있다면 PersonSet에서 Person 객체를 사용하고자 할 때 동기화 기법을 적용해야 함
  
~~~java
@ThreadSafe
public class ServerStatus {
	@GuardedBy("this") public final Set<String> users;
	@GuardedBy("this") public final Set<String> queries;

	public synchronized void addUser(String u) { users.add(u); }
	public synchronized void addQuery(String q) { queries.add(q); }
	public synchronized void removeUser(String u) { users.remove(u); }
	public synchronized void removeQuery(String q) { queries.remove(q); }

}
~~~
- 인스턴스 한정 기법을 사용하면 클래스 내부의 여러가지 데이터를 여러 개의 락을 사용해 따로 동기화 시킬 수 있음

- 자바 라이브러리에는 스레드에 안전하지 않은 ArrayList, HashMap과 같은 클래스를 멀티스레드 환경에서 안전하게 사용할 수 있도록 Collections.synchronizedList와 같은 팩토리 메소드가 존재
- 이런 팩토리 메소드는 컬렉션의 기본 클래스에 스레드 안전성을 확보하는 방법으로 대부분 데코레이터 패턴을 활용함
- 팩토리 메소드의 결과로 만들어진 래퍼(wrapper) 클래스는 기본 클래스의 메소드를 호출하는 연동 역할만하며 그와 동시에 모든 메소드가 동기화 되어 있음
	- 래퍼 클래스를 거쳐야만 원래 컬렉션 클래스의 내용을 사용
	- 원래 걸렉션 객체가 새로운 래퍼 객체 내부에 제한된 상태임
- 인스턴스 한정 기법을 사용하면 전체 프로그램을 다 뒤져보지 않고도 스레드 안전성을 확보하고 있는지 쉽게 분석해 볼 수 있기 때문에 스레드에 안전한 객체를 좀 더 쉽게 구현할 수 있음

### 4.2.1 자바 모니터 패턴
- 자바 모니터 패턴을 따르는 객체는 변경가능한 데이터를 모두 객체 내부에 숨긴 다음 객체의 암묵적인 락으로 데이터에 대한 동시 접근을 막음
- [Counter 객체](https://github.com/garlickim/study-note/blob/main/book/java-concurrency-in-practice/%5Bweek4%5D%20%EA%B0%9D%EC%B2%B4%20%EA%B5%AC%EC%84%B1.md#41-%EC%8A%A4%EB%A0%88%EB%93%9C-%EC%95%88%EC%A0%84%ED%95%9C-%ED%81%B4%EB%9E%98%EC%8A%A4-%EC%84%A4%EA%B3%84)는 자바 모니터 패턴의 전형적인 예
	- value 변수를 클래스 내부에 숨기고, 해당 변수를 사용하는 모든 메소드는 동기화 되어 있음
- 자바 모니터 패턴은 자바에 들어있는 Vector, Hashtable 등의 여러가지 라이브러리 클래스에서도 사용하고 있음
- 자바 모니터 패턴의 가장 큰 장점은 간결함
- 일정한 형태로 스레드 안전성을 확보할 수 있다면 어떤 형태의 락을 사용해도 무방
~~~java
public class PrivateLock {
	private final Object myLock = new Object();
	@GuardedBy("myLock") Widget widget;

	void someMethod() {
		synchronized(myLock) {
			// widget 변수의 값을 읽거나 변경
		}
	}
}
~~~
- private과 final로 선언된 객체를 락으로 사용해 자바 모니터 패턴을 활용하는 예제
- 락으로 활용하기 위한 private 객체를 준비해두면 여러가지 장점을 얻을 수 있음
	- 이런 락은 private으로 선언되어 있기 때문에 외부에서는 락을 건드릴수 없는데, 만약 락이 객체 외부에 공개되어 있다면 다른 객체도 해당하는 락을 활용해 동기화 작업에 함께 참여할 수 있음
		- 잘못된 방법으로 참여한다면 큰일 발생
- 락을 객체 외부로 공개했다면 공개된 락을 사용하는 코드가 올바르게 의도한 대로 동작하는지 프로그램 전체를 모두 뒤져봐야 함

### 4.2.2 예제: 차량 위치 추척
- 택시, 경찰차, 택배 트럭과 같은 차량의 위치를 추적하는 프로그램
- 자바 모니터 패턴에 맞춰 프로그램 작성 후, 스레드 안전성을 보장하는 한도 내에서 객체 캡슐화 정도를 조금씩 낮추는 방법을 찾아볼 예정
~~~java
Map<String, Point> locations = vehicles.getLocations();
for(String key : locations.keySet())
	renderVehicle(key, locations.get(key));
~~~
~~~java
void vehicleMoved(VehicleMovedEvent evt) {
	Point loc = evt.getNewLocation();
	vehicles.setLocation(evt.getVehicleId(), loc.x, loc.y);
}
~~~
- 위와 같은 구조에서는 뷰 스레드와 업데이터 스레드가 동시 다발적으로 데이터 모델을 사용하기 때문에 데이터 모델에 해당하는 클래스는 반드시 스레드 안전성을 확보해야 함

</br>

~~~java
@ThreadSafe
 public class MonitorVehicleTracker {
    @GuardedBy("this") private final Map<String, MutablePoint> locations;

    public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
        this.locations = deepCopy(locations);
    }

    public synchronized Map<String, MutablePoint> getLocations() {
        return deepCopy(locations);
    }

    public synchronized MutablePoint getLocation(String id) {
        MutablePoint loc = locations.get(id);
        return loc == null ? null : new MutablePoint(loc);
    }

    public synchronized void setLocation(String id, int x, int y) {
        MutablePoint loc = locations.get(id);
        if (loc == null)
            throw new IllegalArgumentException("No such ID: " + id);
        loc.x = x;
        loc.y = y;
    }

    private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> m) {
        Map<String, MutablePoint> result = new HashMap<String, MutablePoint>();

        for (String id : m.keySet())
            result.put(id, new MutablePoint(m.get(id)));

        return Collections.unmodifiableMap(result);
    }
}

@NotThreadSafe
public class MutablePoint {
    public int x, y;

    public MutablePoint() {
        x = 0;
        y = 0;
    }

    public MutablePoint(MutablePoint p) {
        this.x = p.x;
        this.y = p.y;
    }
}
~~~
- MutablePoint 클래스가 스레드 세이프하지 않지만, 차량 추적 클래스는 스레드 안전성을 확보하고 있음
- locations 변수와 Point 인스턴스 모두 외부에 공개되지 않음
- 클라이언트에게 위치를 넘겨줄 땐 MutablePoint 클래스의 생성자를 통해 복사본을 만들거나 deepCopy 메소드를 통해 새로운 location 인스턴스 복사본을 제공
- 외부에서 변경 가능한 데이터를 요청할 경우 복사본을 넘겨주는 방법을 사용하면 스레드 안전성을 부분적으로나마 확보할 수 있음
- getLocation()을 호출할 때마다 데이터를 복사하도록 구현한다면 시간이 지나서 차량의 위치가 바뀐다 해도 외부에서 가져간 위치 정보는 바뀌지 않음
	- synchronized 키워드가 지정된 메소드에서 deepCopy 메소드를 호출하기 때문에 복사 작업이 진행되는 동안 추적 프로그램에 대한 락이 걸림
	- 복사 시간이 길어지면 응답 속도가 떨어질 수 있음

</br>

## 4.3 스레드 안전성 위임
- 대부분의 객체는 둘 이상의 객체를 조합해 사용하는 합성(composite) 객체
~~~java
@ThreadSafe
public class CountingFactorizer implements Servlet {
	private final AtomicLong count = new AtomicLong(0);

	public long getCount() { return count.get(); }
	public void service(ServletRequest req, ServletResponse resp) { count.incrementAndGet(); }
}
~~~
- CountingFactorizer 클래스의 상태는 스레드에 안전한 AtomicLong 클래스의 상태와 같음
- CountingFactorizer는 스레드 안전성 문제를 AtomicLong 클래스에게 '위임(delegate)'한다고 표현함
- AtomicLong 클래스가 스레드에 안전하기 때문에 AtomicLong에게 스레드 안전성을 위임한 CountingFactorizer 역시 스레드 세이프함

### 4.3.1 예제: 위임 기법을 활용한 차량 추적
~~~java
@Immutable
public class Point {
	public final int x, y;

	public Point(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
~~~
- Point 클래스는 불변이기 때문에 스레드 세이프함

~~~java
@ThreadSafe
public class DelegatingVehicleTracker {
	private final ConcurrentMap<String, Point> locations;
	private final Map<String, Point> unmodifiableMap;

	public DelegatingVehicleTracker(Map<String, Point> points){
		locations = new ConcurrentHashMap<String, Point>(points);
		unmodifiableMap = new Collections.unmodifiableMap(locations);
	}

	public Map<String, Point> getLocations() {
		return unmodifiableMap;
	}

	public Point getLocation(String id) {
		return locations.get(id);
	}

	public void setLocation(String id, int x, int y) {
		if(location.replace(id, new Point(x,y)) == null) 
			throw new IllegalArgumentException("invalid vehicle name: " + id);
	}
}
~~~
- 스레드 안전성을 ConcurrentHashMap 클래스에 위임
- 모든 동기화 작업은 ConcurrentHashMap에서 담당하고, Map에 들어있는 모든 값은 불변 상태임
- 만약 Point 대신 MutablePoint를 사용했다면, getLocation()을 호출한 클라이언트쪽에 객체 상태를 공개하기 때문에 스레드 안전성이 깨질 수 있음
- 스레드 A가 getLocation()을 호출해 값을 가져간 다음, 스레드 B가 특정 차량의 위치를 변경하면 이전에 스레드 A가 받아갔던 Map에서도 스레드 B가 새로 변경한 값을 동일하게 사용할 수 있음

~~~java
public Map<String, Point> getLocations() {
	return Collections.unmodifiableMap(
			new HashMap<String, Point>(locations)
		);
}
~~~
- Map 내부에 들어있는 내용이 모두 불변이기 때문에 Map 내부의 데이터가 아닌 구조만 복사되어야 함

### 4.3.2 독립 상태 변수
- 위임하고자 하는 내부 변수가 두 개 이상이라 해도 두 개 이상의 변수가 서로 '독립적'이라면 클래스의 스레드 안전성을 위임 할 수 있음
- 독립적이라는 의미는 변수가 서로 상태 값에 대한 연관성이 없음을 일컫음
~~~java
public class VisualComponent{
	private final List<KeyListener> keyListeners = new CopryOnWriteArrayList<KeyListener>();
	private final List<MouseListener> mouseListeners = new CopryOnWriteArrayList<MouseListener>();

	public void addKeyListener(KeyListener listener){
		keyListeners.add(listener);
	}

	public void addMouseListener(MouseListener listener){
		mouseListeners.add(listener);
	}

	public void removeKeyListener(KeyListener listener){
		keyListeners.romove(listener);
	}

	public void removeMouseListener(MouseListener listener){
		mouseListeners.remove(listener);
	}
}
~~~
- 내부적으로 보면 keyListeners 와 mouseListeners 변수는 독립적임
- VisualComponent 클래스는 스레드 안전한 두 개의 이벤트 리스너 목록에게 클래스의 스레드 안전성을 위임할 수 있음
- 스레드에 안전한 List 클래스(CopryOnWriteArrayList)를 사용하고 있으며, 두 변수의 연관성이 없기 때문에 스레드 세이프 함

### 4.3.3 위임할 때의 문제점
~~~java
public class NumberRange {
	// 의존성 조건 : lower <= upper
	private final AtomicInteger lower = new AtomicInteger(0);
	private final AtomicInteger upper = new AtomicInteger(0);

	public void setLower(int i) {
		// 주의 - 안전하지 않은 비교문
		if( i > upper.get() )
			throw new IllegalArgumentException("can't set lower to " + i + " > upper");
		lower.set(i);
	}

	public void setUpper(int i) {
		// 주의 - 안전하지 않은 비교문
		if( i < lower.get() )
			throw new IllegalArgumentException("can't set upper to " + i + " < lower");
		upper.set(i);
	}

	public boolean isInRange(int i) {
		return (i >= lower.get() && i <= upper.get());
	}
}
~~~
- NumberRange 클래스는 스레드 안전성을 확보하지 못했으며, lower와 upper 변수의 의존성 조건을 100% 만족시키지 못함
- setLower(), setUpper() 메소드 둘다 비교문을 사용하지만, 단일 연산으로 처리하도록 동기화를 적용하기 않았기 때문
- lower와 upper 변수에 락을 사용하는 등의 방법을 통해 동기화하면 쉽게 의존성 조건을 충족시킬 수 있음
- lower와 upper가 외부로 공개되어 값의 변경이 일어나지 않도록 적절한 캡슐화도 필요
- 두 개 이상의 변수를 사용하는 복합 연산 메소드를 갖고 있다면 위임 기법으로는 스레드 안전성을 확보할 수 없음
- 내부적으로 락을 활용하여 단일 연산으로 처리될 수 있도록 동기화가 필요함
- 클래스가 서로 의존성 없이 독립적이고 스레드 안전한 두 개 이상의 클래스를 조합해 만들어져 있고 두 개 이상의 클래스를 한번에 처리하는 복합 연산 메소드가 없는 상태라면, 스레드 안전성을 내부 변수에게 모두 위임 할 수 있음
- 위의 예제에서 AtomicInteger를 사용했음에도 불구하고 스레드 세이프하지 않은 이유는 특정 변수가 다른 상태 변수와 아무런 의존성 없는 상황이라면 해당 변수를 volaile로 선언해도 스레드 안전성에는 지장이 없다는 규칙과 비슷함

### 4.3.4 내부 상태 변수를 외부에 공개
- 특정 변수가 현재 상태와 관련없는 현재 기온이나 가장 마지막으로 로그인했던 사용자의 ID 등의 값을 갖는다면 외부 프로그램이 해당하는 값을 바꾼다해도 클래스 내부의 상태 조건을 그다지 망가뜨리지 않을 가능성이 높아 필요하다면 외부에 변수를 공개하고 나쁘지는 않음
	- 변수를 외부에 공개하면 추후 하위 클래스 작성과 같은 부분에서 곤란한 경우가 발생할 수 있으므로 좋은 방법은 아님
- 상태 변수가 스레드 안전하고, 클래스 내부에서 상태 변수의 값에 대한 의존성을 갖고 있지 않으며 상태 변수에 대한 어떤 연산을 수행하더라도 잘못된 상태에 이를 가능성이 없다면 해당 변수는 외부에 공개해도 됨

### 4.3.5 예제: 차량 추적 프로그램의 상태를 외부에 공개
~~~java
@ThreadSafe
public class SafePoint {
    @GuardBy("this") private int x, y;

    private SafePoint(int[] a) { this(a[0], a[1]); }

    public SafePoint(int x, int y) {  set(x, y); }

    public SafePoint(SafePoint p) {
        this(p.get());
    }

    public synchronized int[] get() {
        return new int[]{x, y};
    }

    public synchronized void set(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
~~~
- private constructor capture 구문의 예
- getX(), getY()와 같은 메소드가 존재한다면 x 값을 가져오고 y 값을 가져오기 전에 차량의 위치가 바뀌는 상황이 발생하여 스테일 상황이 발생 가능 함
~~~java
@ThreadSafe
public class PublishingVehicleTracker {
    private final Map<String, SafePoint> locations;
    private final Map<String, SafePoint> unmodifiableMap;

    public PublishingVehicleTracker(Map<String, SafePoint> locations) {
        this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
        this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
    }

    public Map<String, SafePoint> getLocations() {
        return unmodifiableMap;
    }

    public SafePoint getLocation(String id) {
        return locations.get(id);
    }

    public void setLocation(String id, int x, int y) {
        if (!locations.containsKey(id))
            throw new IllegalArgumentException("invalid vehicle name: " + id);
        locations.get(id).set(x, y);
    }
}
~~~
- PublishingVehicleTracker 클래스는 ConcurrentHashMap 클래스에게 스레드 안전성을 위임하여 전체적으로 스레드 안전성을 확보함
- 스레드에 안전하고 변경가능한 SafePoint 클래스를 사용함
- 만약 차량의 위치에 대한 제약 사항을 추가해야 한다면 스레드 안전성을 해칠 수 있음
- 외부 프로그램이 차량의 위치를 변경하고자 할 때, 변경 값을 반영하지 못하도록 거부하거나 변경 사항을 반영하도록 선택할 수 있어야 한다면 해당 구현 방법으로는 충분치 않음

</br>

## 4.4 스레드 안전하게 구현된 클래스에 기능 추가
- 단일 연산 하나를 기존 클래스에 추가하고자 한다면 해당하는 단일 연산 메소드를 기조 ㄴ클래스에 직접 추가하는 방법이 가장 안전함
- 기능을 추가하는 또 다른 방법은 기존 클래스를 상속받는 방법으로, 기존 클래스를 외부에서 상속받아 사용할 수 있도록 설계한 경우만 사용 가능함
~~~java
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent)
            add(x);
        return absent;
    }
}
~~~
- 기존 클래스를 상속받아 기능을 추가하는 방법은 기존 클래스에 직접 기능을 추가하는 방법보다 문제가 생길 위험이 훨씬 많음
	- 동기화를 맞춰야 할 대상이 두 개 이상의 클래스에 걸쳐 분산되기 때문
	- 상위 클래스가 내부적으로 상태 변수의 스레드 안전성을 보장하는 동기화 기법을 수정한다면, 하위 클래스는 의도치 않게 적절한 락을 필요한 부분에 적용하지 못할 가능성이 높음


### 4.4.1 호출하는 측의 동기화
- Collections.synchronizedList 메소드를 사용해 동기화시킨 ArrayList에는 기존 클래스에 메소드를 추가하거나 하위 클래스에서 추가 기능 구현하는 방법을 적용할 수 없음
	- 동기화된 ArrayList를 받아간 외부 프로그램은 받아간 List 객체가 synchronizedList 메소드로 동기화되었는지 알 수 없기 때문
- 도우미 클래스를 따로 구현하여 추가 기능을 구현하는 방법으로 클래스에 원하는 기능을 추가할 수 있음
~~~java
@NotThreadSafe
public class ListHelper<E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());

    ...

    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !list.contains(x);
        if (absent)
            list.add(x);
        return absent;
    }
}
~~~
- 아무런 의미가 없는 락을 대상으로 동기화가 맞춰져 있음
- List가 자체적으로 동기화를 맞추기 위해 어떤 락을 사용했건 간에 ListHelper 클래스와 관련된 락이 아닌 것은 분명함
- putIfAbsent() 는 List 클래스의 다른 메소드와 다른 차원에서 동기화가 되고 있기 때문에 List 입장에서 보면 단일 연산이라고 볼 수 없음
- **결과적으로 putIfAbsent()가 실행되는 도중, 원래 List의 다른 메소드를 얼마든지 호출해서 내용을 변경할 수 있다는 의미**
- 도우미 클래스를 만들어 올바르게 구현하려면 클라이언트 측 락(client-side lock)이나 외부 락(external lock)을 사용해 List가 사용하는 것과 동일한 락을 사용해야 함
- Vector 클래스와 Collections.synchronizedList 메소드에 대한 문서를 읽어보면 Vector 클래스 자체나 synchronized의 결과 List를 통해 클라이언트 측 락을 지원
~~~java
@ThreadSafe
public class ListHelper<E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());

    ...

    public boolean putIfAbsent(E x) {
    	synchronized (list) {
	        boolean absent = !list.contains(x);
	        if (absent)
	            list.add(x);
	        return absent;
	    }
    }
}
~~~
- 제3의 클래스를 만들어 클라이언트 측 락 방법으로 단일 연산을 구현하는 방법은 특정 클래스 내부에서 사용하는 락을 전혀 관계없는 제3의 클래스에서 갖다 쓰기 때문에 훨씬 위험해보이는 방법임
- 클라이언트 측 락은 클래스 상속과 함께 봤을 때 여러가지 공통점, 예를 들어 클라이언트나 하위 클래스에서 새로 구현한 내용과 원래 클래스에 구현되어 있던 내용이 밀접하게 연관되어 있다는 등의 공통점이 존재
- 클라이언트 측 락을 구현할 때도 캡슐화되어 있는 동기화 정책을 무너뜨릴 가능성이 존재함

### 4.4.2 클래스 재구성
- 기존 클래스에 새로운 단일 연산을 추가하고자 할 때, 좀더 안전하게 사용할 수 있는 방법은 '재구성(composition)'
~~~java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
    private final List<T> list;

    public ImprovedList(List<T> list) { this.list = list; }

    public synchronized boolean putIfAbsent(T x) {
        boolean contains = list.contains(x);
        if (!contains)
            list.add(x);
        return !contains;
    }

    public synchronized void clear() { list.clear(); }
    // ... List 클래스의 다른 메소드도 clear와 비슷하게 구현
}
~~~
- ImprovedList 클래스는 그 자체를 락으로 사용해 그 안에 포함되어 있는 List와는 다른 수준에서 락을 활용하고 있음
- ImprovedList 클래스를 락으로 사용해 동기화하기 때문에 내부 List 클래스가 스레드 안전한지 신경쓰지 않음
- 이와 같은 동기화 기법을 한 단계 더 사용한다면 전체적인 성능 측면에서는 약간 부정적인 영향이 존재 할 수 있지만, 이전 사용한 클라이언트 측 락 등의 방법보다 훨씬 안전함
- List 클래스가 외부로 공개되지 않는 한 스레드 안전성을 확보할 수 있음

</br>

## 4.5 동기화 정책 문서화하기
- 클래스의 동기화 정책에 대한 내용을 문서로 남기는 일은 스레드 안전성을 관리하는데 있어 가장 강력한 방법 가운데 하나
- 구현한 클래스가 어느 수준까지 스레드 안전성을 보장하는지에 대해 충분히 문서를 작성해둬야 함
- 동기화 기법이나 정책을 잘 정리해두면 유지보수 팀이 원활하게 관리할 수 있음
- 동기화 정책을 구성하고 결정하고자 할 때 여러가지 사항을 고려해야 함
	- 어떤 변수를 volatile로 선언할지
	- 어떤 변수를 사용할 때 락으로 막아야 하는지
	- 어떤 변수는 불변 클래스로 만들고, 어떤 변수를 스레드에 한정시켜야 하는지
	- 어떤 연산을 단일 연산으로 구성해야하는지
- 자바5 부터 사용할 수 있는 @GuardBy 등의 어노테이션만 활용해도 훌륭한 편

### 4.5.1 애매한 문서 읽어내기
- 자바로 만들어진 여러가지 기술을 정의하는 스펙을 보면 스레드 안전성에 대한 요구사항이나 보장 범위에 대한 언급이 별로 없음
- 스펙에 명확하게 정의되어 있지 않은 부분을 좀 더 근접하게 추측하려면 스펙을 작성하는 사람의 입장에서 생각해야 함