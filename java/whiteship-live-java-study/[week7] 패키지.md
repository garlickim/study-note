## package 키워드
- class,interface 등의 액세스 보호 및 name space 관리를 제공하는 **type**들의 그룹핑
- 위에서 언급한 **type** 이란 class, interface, enum, annotation
- java 기본 클래스는 java.lang 패키지에 있으며, IO 관련 클래스는 java.io 패키지에 존재
- 아래와 같은 이유로 클래스와 인터페이스는 패키지로 묶어야 함
  - 다른 프로그래머가 관련 type을 쉽게 파악할 수 있음
  - 패키지마다 새로운 name space를 가지므로, 동일한 클래스명이 있어도 다른 패키지와 충돌을 일으키지 않음
  - 동일한 패키지 내 액세스를 허용하며, 외부 패키지에 대한 액세스는 제한할 수 있음
- package 이름
  - package 이름은 클래스,인터페이스 이름과 충돌하지 않도록 소문자로 작성
  - 보통 인터넷 도메인 이름을 역순으로 하여 사용 (ex: com.naver)
  - java 언어에 대한 패키지는 java. 또는 javax. 
  - package 이름에 java 예약어가 사용되거나 하는 경우가 있다면 _을 사용하거나 인터넷 도메인이 아닌 다른 규칙으로 사용 될 수도 있음
- package 생성
  - java 파일 맨 위에 선언
  - package 명렁어로 시작하며 그 뒤에 package 이름을 작성
  ~~~java
  package com.naver.dto
  
  public class Student{
  }
  ~~~

</br>

## import 키워드
- 외부에서 package 멤버를 사용하려면 몇가지 방식이 존재
  - full package 명으로 접근
    - 코드 가독성이 떨어짐
     ~~~java
     graphics.Rectangle myRect = new graphics.Rectangle();
     ~~~
  - package member import
    - package 명령문 다음에 import 키워드를 사용하여 특정 멤버를 가져옴
    ~~~java
    package com.naver.dto // 현재 패키지
    import graphics.Rectangle; 
    
    class Shape {
    }
    ~~~
  - 전체 package import
    - 특정 패키지에 포함된 모든 type을 가져오려면 import *(wild card)를 사용
    ~~~java
    import graphics.*; 
    ~~~
  
</br>

## 클래스패스
- 클래스를 찾기위한 경로
- JVM이 클래스 파일을 찾는 기준 결로
- :(콜론)을 사용하여 경로를 구분
- application class loader는 설정된 classpath를 기반으로 지정된 경로에 있는 클래스를 로딩
- 클래스패스는 package 계층 구조의 "최상위 경로"
~~~java
class A {
    void println(){
        System.out.println("inside A");
    }
}

class B {
    public static void main(String[] args) {
        A a = new A();
        a.println();
    }
}
~~~
![classpath-sample](/img/classpath-sample.png)  
java 명령에서 사용한 -cp
  - javac에 의해 컴파일되어 생성된 A.class와 B.class
  - 동일한 경로에서 java 명령어를 사용하여 실행하면 정상
  - A.class 파일을 하위 tmp 폴더로 이동
  - 아까와 동일한 경로에서 java 명령어를 사용하여 실행하면 java.lang.ClassNotFoundException: A 에러 발생
  - -cp 옵션을 사용하여 classpath 경로를 지정하여 실행하면 정상
    - -cp ".:tmp" 은 현재 디렉토리 기준으로 class 파일을 찾고 없으면 tmp 폴더에서 찾는다는 의미
    
![classpath-sample2](/img/classpath-sample2.png)  
javac 명령에서 사용한 -cp

</br>

## CLASSPATH 환경변수
- 환경변수란?
  - 프로세스가 컴퓨터에서 동작하는 방식에 영향을 미치는 동적인 값
  - OS상에서 동작하는 응용프로그램들이 참조하기위한 설정 기록
  - 환경변수는 2가지로 구분됨
    - 사용자 변수 : OS내의 사용자 별로 다르게 설정가능한 환경변수
    - 시스템 변수 : 시스템 전체에 모두 적용되는 환경변수
- 자바 실행시 매번 -cp 와 같은 클래스패스 옵션을 주는게 번거롭다면 CLASSPATH 환경변수 설정을 통해 지정 가능
- linux 의 jdk CLASSPATH 설정 예시
  ~~~bash
   vi /etc/profile 에 들어가서 맨밑에 추가

  export JAVA_HOME=/usr/local/src/jdk1.8.0_144
  export PATH=$JAVA_HOME/bin:$PATH
  export CLASSPATH=$CLASSPATH:$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar
  ~~~
- macos 의 jdk CLASSPATH 설정 예시
  ~~~bash
  vi ~/.bash_profile

  export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_192.jdk/Contents/Home
  export PATH=${PATH}:$JAVA_HOME/bin
  ~~~

</br>

## -classpath 옵션
-  -cp <디렉토리 및 zip/jar 파일의 클래스 검색 경로>
-  -classpath <디렉토리 및 zip/jar 파일의 클래스 검색 경로>
-  --class-path <디렉토리 및 zip/jar 파일의 클래스 검색 경로> 클래스 파일을 검색하기 위한 디렉토리, JAR 아카이브 및 ZIP 아카이브의 :(으)로 구분된 목록

</br>

## 접근지시자
- 클래스, 변수, 메소드, 인스턴스의 접근 허용 범위를 지정하는 지시자 (Access Modifier)
- public, private, default(package-private), protected가 있음

| 지시자 | 클래스 내부 | 동일 패키지 | 상속 클래스 | 이외 영역 |
| ----- | ---------|----------|----------|--------|
| private | O | X | X | X |
| default(package-private) | O | O | X | X |
| protected | O | O | O | X |
| public | O | O | O | O |

- 접근 범위는 public > protected > default > private 순으로 넓음
- 접근제어자를 아무것도 작성하지 않은 경우는 default, 즉 package-private으로 지정됨
  - 같은 패키지 내에서만 접근 가능함

