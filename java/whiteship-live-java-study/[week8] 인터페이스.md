## 인터페이스 정의하는 방법
- 인터페이스란?
  - 클래스를 작성할 때, 기본이 되는 틀이 제공하고 규약을 정의하여 제공하는 것
  - 자바는 클래스의 다중 상속을 지원하지 않지만, 인터페이스의 다중 상속은 지원
- 클래스를 정의하는 것과 비슷하며 class 대신 interface 키워드를 사용
~~~java
public interface Car{
  int distance = 0;
  void changeColor(String color);
  
  default void printNickName(String name){
      System.out.println("nickname is "+name);
  }
}
~~~

</br>

## 인터페이스 구현하는 방법
- 구체화된 클래스명을 가진 클래스에 implements 키워드를 사용하여 구현
- implements 키워드에 의해 인터페이스를 상속받으면 컴파일러에 의해 인터페이스를 구현하라고 에러 발생
  ![interface-compile-error](/img/interface-compile-error.png)  
- 인터페이스에 선언된 메소드를 모두 오버라이딩
  ![interface-sample](/img/interface-sample.png)  
- default 메소드는 오버라이딩 하지 않아도 됨
- 필수 오버라이딩이 아니기 때문에, 공통적인 내용을 담고 있는 메소드로 많이 사용되며 필요한 경우에만 오버라이딩하여 사용할 수 있음

</br>

## 인터페이스 레퍼런스를 통해 구현체를 사용하는 방법
- 인터페이스 자체로는 객체를 생성할 수 없음 (new 키워드 사용 불가)
- 인터페이스를 구체화한 클래스를 통해 객체를 생성할 수 있음
- 상속과 마찬가지로 인터페이스 또한 다형성이 적용되어 인터페이스를 타입으로 지정 가능
  ~~~java
  interface Car {
      int move(int distance);
  }


  class Seltos implements Car {
      @Override
      public int move(int distance) {
          return ++distance;
      }
  }

  class Example {
      public static void main(String[] args) {
          Car car = new Seltos(); // Car 라는 인터페이스 타입으로 지정 가능
          car.move(1);
      }
  }
  ~~~

</br>

## 인터페이스 상속
- extends 키워드를 사용하여 인터페이스끼리 상속이 가능함
- 인터페이스는 다중 상속을 지원
  ~~~java
  interface Color {
      String getBackgroundColor(String color);
  }

  interface Profile {
      String getNickname(String name);
  }


  interface Car extends Profile, Color {
      int move(int distance);
  }
  ~~~
- 상속받은 인터페이스를 구현하는 구현체는 모든 상위 인터페이스의 메소드를 구현해야 함
  ~~~java
  interface Color {
      String getBackgroundColor(String color);
  }

  interface Profile {
      String getNickname(String name);
  }


  interface Car extends Profile, Color {
      int move(int distance);
  }

  class Seltos implements Car {
      @Override
      public String getBackgroundColor(String color) {
          return "RED";
      }

      @Override
      public String getNickname(String name) {
          return "sally";
      }

      @Override
      public int move(int distance) {
          return ++distance;
      }
  }
  ~~~

</br>

## 인터페이스의 기본 메소드 (Default Method), 자바 8
- 인터페이스에서 default 키워드를 사용하여 메소드를 선언하면 메소드 내부를 구현할 수 있음
- defatul 메소드의 경우, 구현하는 클래스에서 메소드 오버라이딩이 필수가 아님
- 인터페이스에 메소드를 추가하면 상속받고 있는 모든 클래스에서 해당 메소드를 구현해야하는 이슈를 default 메소드를 통해 해결 가능
- 그렇다면 기본 메소드는 무조건 좋을까?
 - 동일한 메소드를 가지고 있는 인터페이스과 클래스가 있을 때, 이를 상속받고 있는 클래스가 있다고 하자
 - 과연 아래 결과는 답이 무엇일까?
 ~~~java
 interface Sum {
      default int calculate(int num1, int num2) {
          return num1 + num2;
     }
 }

 class Sub {
     public int calculate(int num1, int num2) {
          return num1 - num2;
      }
  }



 class Sample extends Sub implements Sum {
     public static void main(String[] args) {
          Sample sample = new Sample();
          System.out.println(sample.calculate(3, 2)); // interface ? class ? 누구의 메소드가 실행될 것 인가?
      }
  }
 ~~~
  - 정답은 class의 메소드가 실행 됨
  - 위와 같이 구현체에서는 어떠한 부모의 메소드가 실행되는지 알 수 없는 경우가 있을 수 있기에, 가급적 default 메소드는 지양하는게 좋을거라 생각함

</br>

## 인터페이스의 static 메소드, 자바 8
- 인터페이스에서 static 메소드를 정의 가능
- 인터페이스의 static 메소드는 오버라이딩이 불가능
  - 오버라이딩시 컴파일 에러 발생  
    ![interface-static-method-example](/img/interface-static-method.png)  
- 인터페이스명으로 직접 호출하여 사용해야 함 (구현체 없이 사용 가능)
  ~~~java
  interface Car {
      static int move(int distance){
          return ++distance;
      }
  }

  class Example {
      public static void main(String[] args) {
          System.out.println(Car.move(1));
      }
  }
  ~~~

</br>

## 인터페이스의 private 메소드, 자바 9
- 인터페이스 내에서 private 키워드를 사용하여 private method, private static method 생성이 가능해짐
- 인터페이스는 기본 public 접근제어자를 가지고 있지만, private으로 선언된 메소드는 상속이 되지 않음
- private 메소드는 구현부가 존재
  - 구현부가 없는 private 메소드는 컴파일 에러 발생
  ~~~java
  interface Car {
      private void name(String name){
          System.out.println(name);
      }

      private static int move(int distance){
          return ++distance;
      }
  }
  ~~~
- private method는 인터페이스 내의 다른 메소드를 호출 가능함 
 - private static method는 static method만 호출 가능
  ~~~java
  interface Car {
      private void name(String name){
          System.out.println("name is "+ name);
          System.out.println("color is "+ getColor("RED"));
      }

      private static int move(int distance){
          return ++distance;
      }

      String getColor(String color);
  }
  ~~~




