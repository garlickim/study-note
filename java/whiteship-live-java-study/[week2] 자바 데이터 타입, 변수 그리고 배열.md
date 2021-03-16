### 프리미티브 타입 종류와 값의 범위 그리고 기본 값
- primitive type 이란?  
  - jvm 스택 영역에 저장되는 타입
  - 실제 값을 저장
  - null 불가
  - generic type에서 사용 불가
  
|       | type   |      size      |  default | range |
|--------|---------|---------------|-------|  -------|
| 논리형   | boolean |  1 byte | false | true, false |
| 정수형   | byte | 1 byte | 0 | -128 ~ 127 |
| 		 | short | 2 byte | 0 | -2<sup>16</sup> ~ 2<sup>16</sup>-1 |
| 		 | int | 4 byte | 0 | -2<sup>32</sup> ~ 2<sup>32</sup>-1 |
| 		 | long | 8 byte | 0L | -2<sup>64</sup> ~ 2<sup>64</sup>-1 |
| 실수형   | float | 4 byte | 0.0F | (3.4 * 10<sup>-38</sup>) ~ (3.4 * 10<sup>38</sup>) |
| 	     | double | 8 byte | 0.0 | (1.7 X 10<sup>-308</sup>) ~ (1.7 X 10<sup>308</sup>) |
| 문자형   | char | 2 byte | '\u0000' | 0 ~ 2<sup>16</sup>-1 |

  </br>

### 프리미티브 타입과 레퍼런스 타입
- reference type 이란?
  - jvm 힙 영역에 저장되는 타입
  - 실제 값이 아닌 주소값을 저장
  - null 가능
  - gernic type에서 사용 가능
  - Integer, Long, Boolean, Double, Float, String, Array, List, Class 등..  

  </br>


### 리터럴
- 변수에 할당하는 변하지 않는 데이터를 의미  
~~~java
int size = 1;
double temperature = 30.5d
String name = "garlic"
~~~
위의 예제에서 1, 30.5d, "garlic"이 리터럴에 해당  

  </br>


### 변수 선언 및 초기화하는 방법
~~~java
class Car{
  String name; // 선언과 동시에 null로 초기화
  
  public  void move() {
        int length;
        System.out.println(length); // compile error
        
        int length2 = 3;  // 변수 생성 및 초기화
        System.out.println(length2);
    }
}
~~~

  </br>
  
  
### 변수의 스코프와 라이프타임
- 변수가 선언된 위치가 해당 변수의 스코프와 라이프 타임을 결정  

| 변수   		|  	설명	 |   스코프      |  라이프타임 |
|-----------|----------|------------|-------|
| 인스턴스 변수  |  클래스 내부에 선언된 변수 | 클래스 전체 <br> (static 메소드 제외) |	new 라는 keyword로 객체가 생성된 이후부터 더이상의 객체 참조가 일어나지 않아 메모리상에 삭제될 때 까지 |
| 클래스 변수  |  클래스 내부에 선언된 변수(static 변수 포함) | 클래스 전체 | 클래스 로딩 시점부터 프로그램 종료시까지 |
| 지역 변수  |  메소드 내부에 선언된 변수 | 해당 메소드 내 |  해당 메소드를 벗어나면 소멸	|
~~~java
class Apple {
    static int count;   // 클래스 변수
    
    int price = 1000;   // 인스턴스 변수

    public void eat() {
        Timestamp timestamp;   // 지역 변수
    }
}
~~~

</br>


### 타입 변환, 캐스팅 그리고 타입 프로모션
- primitive 타입과 reference 타입 사이의 타입 변환에는 2가지 방식이 존재
- 캐스팅(Casting) : 강제 형변환, 자동 형 변환의 규칙에는 맞지 않지만 변환이 필요한 경우에 사용
  - 큰 타입에서 작은 타입으로의 변환
  ~~~java
    long longSize = 3L;
    int intSize = (int) longSize;
  ~~~
- 프로모션(Promotion) : 자동 형변환
  - 작은 타입에서 큰 타입으로의 변환
  ~~~java
    int intSize = 3;
    long longSize = intSize;
  ~~~

+) Auto Boxing & Auto UnBoxing
- primitive type 과 warrapper class(primitive type 변수를 객체로 한번 감싼 것) 사이의 형변환
- Auto Boxing
  ~~~java
  int intSize = 3;
  Integer size = intSize; // auto boxing : Integer size = new Integer(3);
  ~~~
- Auto UnBoxing
  ~~~java
  Integer size = 3;
  int intSize = size; // auto unboxing : int intSize = size.intValue();
  ~~~

</br>


### 1차 및 2차 배열 선언하기
- 배열?
  동일한 타입의 데이터를 연속된 공간에 저장하는 자료 구조
  배열의 index는 0부터 시작
  고정된 길이로 생성됨
- 1차 배열  
  | index[0] = 0 |  	index[1] = 0 |   index[2] = 4000 |  index[3] = 0 |
  |-----------|----------|------------|-------|  
  ~~~java
  // 생성 방법1 : 배열 선언 후 값 할당
  int[] intArray = new int[4]; // 사이즈가 4인 int형 array 선언
  intArray[2] = 4000; // index 2번에 4000 이라는 값 할당
  
  // 생성 방법2 : 배열 선언과 동시에 값 할당
  int[] intArray = {0,0,4000,0};
  ~~~  
  
- 2차 배열  
  - 행과 열로 나타낼 수 있는 배열  
  
   | index[0][0] = 0 |  	index[0][1] = 4000 |   index[0][2] = 0 |   
   |-----------|----------|------------|  
   | **index[1][0] = 0** |  	**index[1][1] = 0** |   **index[1][2] = 0**  |   
  ~~~java
    // 생성 방법1 : 배열 선언 후 값 할당
    int[][] intArray = new int[2][3];
    intArray[0][1] = 4000;
    
    // 생성 방법2 : 배열 선언과 동시에 값 할당
    int[][] intArray = {{0,4000,0}, {0,0,0}};
  ~~~


</br>



### 타입 추론, var
- 타입추론이란?  
  코드 작성 당시 정해지지 않은 코드의 타입을 Compiler가 유추하는 것
- var
  - java 10부터 등장한 타입 추론을 지원하는 키워드
  - 지역 변수로 선언되어야 하며, 선언과 동시에 초기화가 필요 (메소드 외부에서 선언 불가)
  ~~~java
  class Test {
    var name = "garlic"; // compile error
    public void test() {
        var name = "garlic";
    }
  ~~~
  - var 변수와 다이아몬드 연산자<>를 함께 사용하면...?
    굉장히 위험해보인다! 
  ~~~java 
  ArrayList<String> stringList = new ArrayList<>(); 
  // 위와 같이 사용할 땐 해당 list의 데이터 타입이 보장되는데 
  ~~~

  ~~~java
  // var와 함께 사용하는 경우 데이터 타입이 보장 받지 못해 런타임시 에러가 발생할 가능성이 커보임
  var list = new ArrayList<>();
  list.add("aaa");
  list.add(2);

  System.out.println(list.get(0)); // aaa
  System.out.println(list.get(1)); // 2
  ~~~
