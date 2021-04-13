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

