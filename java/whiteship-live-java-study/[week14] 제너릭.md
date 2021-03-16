# 학습내용
- 제너릭이란
- 제너릭을 사용해야하는 이유
- 제네릭 사용법
- 제네릭 주요 개념 (바운디드 타입, 와일드 카드)
- PECS
- 제네릭 메소드 만들기
- Erasure

## 제네릭이란?
- 말그대로, Generic(일반적인)
- 일반적인 코드를 작성하여 코드의 재생산성을 높이고 type에 관하여 런타임 시점에 발생할 수 있는 오류를
컴파일 타임에 오류를 걸려내기 위함

</br>

## 제너릭을 사용해야하는 이유?
- 컴파일 시점에 강한 type check가 가능
- type cast 비용이 사라짐
  ~~~java
  // non-generic
  List list = new ArrayList();
  list.add("kelly");
  String name = (String) list.get(0);
  
  // generic
  List<String> list = new ArrayList<>();
  list.add("kelly");
  String name = list.get(0);
  ~~~
- 읽기 쉽고, type safe하며 다른 type을 가지도록 generic 알고리즘을 사용하여 커스터마이징 할 수 있음

</br>

## 제너릭 사용법
- 사용할 컬렉션 뒤에 diamond 연산자를 사용하여 타입을 작성
- 선언된 타입만 컬렉션에 담을 수 있음
~~~java
List<String> list = new ArrayList<>();
list.add("kelly");
list.add("sally");
list.add(1234); // compile error !!
~~~

</br>

## 제네릭 주요 개념 (바운디드 타입, 와일드 카드)
- Type Parameter Naming Convention
  - 하나의 대문자로 선언
  - E : Element (자바 컬렉션 프레임워크에서 주로 사용)
  - K : Key
  - N : Number
  - T : Type
  - V : Value
  - S,U,V etc. : 2nd, 3rd, 4th types
- Bounded Type
  - 특정 Type의 하위 타입으로만 Type을 제한함
  - <T extends Rectangle> 와 같은 형식으로 사용
    - Rectangle를 포함한 하위 타입으로 제한
    - 하위 타입이 아닌 경우, compile error 발생
  ~~~java
  public class MainApplication {
      public static void main(String[] args) {
          Shape<Rectangle> rectangle = new Shape<>();
          Shape<Square> square = new Shape<>();
      }
  }

  class Shape<T extends Rectangle> {
      void set(T value) {}
  }

  class Rectangle{}
  class Square extends Rectangle{}
  ~~~
  - Multiple Bounded Type
    - Class의 하위 타입 뿐만 아니라 interface의 하위 타입도 선언 가능
    ~~~java
    class Shape<T extends Rectangle & A & B> {
        void set(T value) {}
    }

    class Rectangle implements A, B{}
    class Square extends Rectangle{}

    interface A{}
    interface B{}
    ~~~
- Bounded wildcard type
  - ?를 사용하여 <? extend T> 와 같은 형식으로 사용
  - Bounded Type과 같이 하위 타입으로 타입을 제한
  - **여기서 의문점!!! 그렇다면 Bounded Type과의 차이점은 도대체 뭐지??**
    - 아래 코드로 보자
      - Bounded Type과 Bounded wildcard type은 아래 예제와 같이 동일하게 사용 가능  
      ![bounded-sample](/img/bounded-sample.png)
      - 차이점은!! Bounded Type의 경우 interface를 함께 상속 받을 수 있음. 반면 Bounded wildcard type의 경우는 하나의 타입만 상속 가능  
      ![bounded-sample2](/img/bounded-sample2.png)
- Unbounded wildcard type 
  - List<?> 와 같은 형식으로 사용
  - Type의 관계없이 어떠한 것도 올 수 있음
  - Object 클래스에서 제공되는 기능을 사용하여 구현하는 메소드를 작성하는 경우에 사용
  - 코드가 type 파라미터에 의존하지 않는 Generic 클래스의 메소드를 사용하는 경우에 사용
    - 예를 들어 List.size 또는 List.clear와 같은 메소드는 T에 의존하지 않음   
    ![unbounded-wildcard-type](/img/unbounded-wildcard-type.png)

## PECS : Producer(생산자)-extends, Consumer(소비자)-super
- Produce를 하면 상속을 받고, Comsume을 하면 상속을 한다는 법칙?
- parameter type이 생산자 역할을 하면 <? extends T> 와 같이 사용
- parameter type이 소비자 역할을 하면 <? super T> 와 같이 사용
- Producer-extends는 READ만 가능
- Consumer-super는 WRITE만 가능  
  ![pecs-sample](/img/pecs-sample.png)

</br>

## 제네릭 메소드 만들기
- 메소드 return type 앞에 diamond 연산자를 사용하여 type parameter 목록을 선언
  - static 메소드의 경우 static과 return type 앞에 선언
~~~java
class Shape {
    <T> void setShape(T t) {
    }
}

class Name {
    static <T> void setName(T t) {
        
    }
}
~~~
~~~java
class Pair<K, V> {
  private K key;
  private V value;

  public Pair(K key, V value){
      this.key = key;
      this.value = value;
  }

  public K getKey() {
      return this.key;
  }
  public V getValue() {
      return this.value;
  }
}
~~~

</br>

## Erasure
- 컴파일러에 의해 모든 타입 파라미터가 지워지고, 제너릭 타입에 대한 정보가 없는 것
- 제너릭이 도입되기 전의 소스코드와의 호환성을 유지하기 위해 Erasure가 존재
- \<T extends Car\> 의 제너릭 타입은 컴파일시 Car로 치환 됨
- \<T\> 의 제너릭 타입은 컴파일시 Object로 치환 됨  
![erasure-ex01](/img/erasure-ex01.png)  
![erasure-ex02](/img/erasure-ex02.png)  


