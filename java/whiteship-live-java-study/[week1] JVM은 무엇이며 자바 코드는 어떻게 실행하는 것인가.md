### JVM이란 무엇인가
- Java Virtual Mechine(JVM) : 자바 가상 머신  
- .class 파일의 바이트코드를 읽어와 해석하고 실행될 OS에 명령어를 전달  
- java 뿐만 아니라 .class로 컴파일되는 언어는 JVM 위에서 실행 가능  

</br>
  
### 컴파일 하는 방법
- IDE(intelliJ, Eclipse) : 코드 작성 -> Run
- javac 명령어 사용 :   
~~~bash
javac HelloWorld.java  
~~~

</br>
  
### 실행하는 방법
- IDE(intelliJ, Eclipse) : Run
- java 명령어 사용 : java HelloWorld

</br>  
  
### 바이트코드란 무엇인가
- JVM이 이해할 수 있는 코드
- JVM이 바이트코드를 해석하여 OS로 명령어 전달
- javac 명령어를 통해 코드가 컴파일되고 .class 파일은 바이트코드로 이루어짐

</br>  
  
### JIT 컴파일러란 무엇이며 어떻게 동작하는지
- JVM은 컴파일된 바이트코드를 한줄한줄 읽어 해석하여 명령어를 실행함 (인터프리터)
- 자주 사용되는 코드의 경우, 해당 코드를 캐싱하여 코드가 실행되는 시점에 JIT 컴파일러에 의해 캐싱된 코드를 사용  

</br>  

### JVM 구성 요소
![jvm structure](/img/jvm%20structure.jpeg)  
출처 [https://medium.com/webeveloper/jvm-java-virtual-machine-architecture-94b914e93d86](https://medium.com/webeveloper/jvm-java-virtual-machine-architecture-94b914e93d86)

- JVM은 크게 클래스로더, 데이터영역, 실행엔진, 네이티브영역으로 나뉘어짐
- 클래스로더 : 런타임 시점에 바이트코드를 읽어 메모리에 로드  
- 데이터영역 : 클래스 파일이 적재되어 있는 공간
  - 메소드 영역 : static 변수, 데이터/리턴 타입, 클래스 멤버 변수가 저장됨
  - 힙 영역 : new 키워드에 의해 생성된 객체 및 배열이 저장됨
  - 스택 영역 : 지역 변수 변수가 저장됨
  - PC Register : 현재 스레드의 실행 정보(주소값, 명령)를 저장함
  - 네이티브 메소드 스택 : 자바 외의 언어로 작성된 코드가 저장됨  
- 실행엔진 : 바이트코드를 읽어 컴퓨터가 이해할 수 있는 언어(기계어)로 변경하여 명령어를 실행하는 역할을 담당
  - GC : 힙 영역에서 더 이상 참조되지 않는 객체들을 메모리에서 제거하는 역할
- 네이티브영역 :
  - 네이티브 메소드 인터페이스 : 다른 언어로 작성된 코드를 자바에서 실행하고 싶을 때, 해당 인터페이스를 통해 작업을 수행

</br>  

### JDK와 JRE의 차이
- JDK : java development kit  
  - java 어플리케이션을 개발하는데 필요한 compiler, debugger, documentation, JRE를 포함한 것

- JRE : java runtime environment
  - libraries, JVM, plugin을 포함한 것
