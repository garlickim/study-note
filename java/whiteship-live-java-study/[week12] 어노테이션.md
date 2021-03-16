# 학습내용
- 애노테이션 정의하는 방법
- @retention
- @target
- @documented
- 애노테이션 프로세서

</br>

## 애노테이션 정의하는 방법
- @ 문자 뒤에 오는것을 컴파일러는 어노테이션으로 인식함
- 어노테이션 만드는 방법
  ~~~java
  @interface 어노테이션명 {
    // 코드
  }
  ~~~
- 반복되는 정보를 제공하는 주석이 있을 경우, 어노테이션을 이용하여 깔끔하게 처리가 가능함
  - @Documented에 의해 javadoc에 @ClassPreamble으로 채운 내용이 모두 작성됨
  이전 코드
  ~~~java
  public class Generation3List {
     // Author: John Doe
     // Date: 3/17/2002
     // Current revision: 6
     // Last modified: 4/12/2004
     // By: Jane Doe
     // Reviewers: Alice, Bill, Cindy

     // class code goes here
  }
  ~~~
  이후 코드
  ~~~java
  @interface ClassPreamble {
     String author();
     String date();
     int currentRevision() default 1;
     String lastModified() default "N/A";
     String lastModifiedBy() default "N/A";
     // Note use of array
     String[] reviewers();
  }
  
  @ClassPreamble (
     author = "John Doe",
     date = "3/17/2002",
     currentRevision = 6,
     lastModified = "4/12/2004",
     lastModifiedBy = "Jane Doe",
     // Note array notation
     reviewers = {"Alice", "Bob", "Cindy"}
  )
  public class Generation3List {
    // class code goes here
  }
  ~~~
  
  </br>
  
## @retention
- 어노테이션의 life time을 나타냄
- 어노테이션 선언시 @retention이 없다면 default는 RetentionPolicy.CLASS
- RetentionPolicy.CLASS
  - 컴파일 시점까지는 어노테이션 정보가 유지되지만, 런타임시에는 유지되지 않음
- RetentionPolicy.RUNTIME
  - 런타임 시점까지 어노테이션 정보가 유지되어, JVM에 의한 참조가 가능함
- RetentionPolicy.SOURCE
  - 컴파일러에 의해 삭제됨 --> 클래스 파일에 존재하지 않음
  
</br>

## @target
- 어노테이션을 적용할 수 있는 context를 지정하는 것
- 어노테이션 선언에서만 사용 가능함
- ElementType.TYPE
  - 클래스, 인터페이스, 어노테이션, enum에 적용 가능
- ElementType.ANNOTATION_TYPE
  - 어노테이션에 적용 가능
- ElementType.CONSTRUCTOR
  - 생성자에 적용 가능
- ElementType.FIELD
  - 필드, enum 상수에 적용 가능
- ElementType.LOCAL_VARIABLE
  - 지역 변수에 적용 가능
- ElementType.METHOD
  - 메소드에 적용 가능
- ElementType.PACKAGE
  - 패키지에 적용 가능
- ElementType.PARAMETER
  - 파라미터에 적용 가능
- ElementType.TYPE_PARAMETER
  - 타입 파라미터(T,R,E,..)에 적용 가능
- ElementType.TYPE_USE
  - 타입이 사용되는 모든곳에 적용 가능

</br>

## @documented
- 어노테이션에 대한 정보가 컴파일러에 의해 javadoc에 작성됨
- 자바 표준 어노테이션에는 @Documented 어노테이션이 붙어있음
  - @Override, @SuppressWarnings 제외
  ~~~java
  @Documented
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.ANNOTATION_TYPE)
  public @interface Inherited {
  }
  ~~~

</br>

## 애노테이션 프로세서
- 컴파일 타임에 어노테이션을 프로세싱하여 코드를 변경 또는 조작하는 프로세서
- 롬복이 대표적인 예로, @Getter/@Setter 어노테이션을 통해 컴파일시 getter/setter 메소드를 생성함
- AbstractProcessor를 상속받아 구현체를 만들 수 있음
- 읽어볼만한 아티클 추가
  - https://www.baeldung.com/java-annotation-processing-builder
