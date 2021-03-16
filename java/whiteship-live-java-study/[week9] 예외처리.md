# 학습내용
- 자바에서 예외 처리 방법 (try, catch, throw, throws, finally)
- 자바가 제공하는 예외 계층 구조
- Exception과 Error의 차이는?
- RuntimeException과 RE가 아닌 것의 차이는?
- 커스텀한 예외 만드는 방법

<br>

## 자바에서 예외 처리 방법 (try, catch, throw, throws, finally)
- 자바에서 예외를 처리하는 방법은 여러가지가 있음
- try-catch-finally 구문
  - 예외 발생 가능성이 있는 실행 코드를 try 블록 안에 작성
  - 발생 가능한 예외를 catch의 괄호에 작성
    - 여러개의 catch 블록을 사용할 수 있음. 다만! exception의 범위가 작은->큰 순서로 작성되어야 함
  - 예외 발생시 행동할 코드를 catch 블록안에 작성
  - try-catch 블록과 관계없이 항상 실행되는 코드는 finally 블록에 작성
  ~~~java
  try {
      int number = Integer.parseInt("a");
  } catch (NumberFormatException e) {
      System.out.println("NumberFormatException 발생");
  } catch (Exception e) {
      System.out.println("Exception 발생");
  } finally {
      System.out.println("finally block 안");
  }
  ~~~
- try-with-resource 구문
  - jdk7에서 등장
  - try()에 AutoCloseable의 구현체를 전달하면, try 블록이 끝난 후 자동으로 자원을 close 처리함
    - 기존에 catch 또는 finally에서 처리했던 방식을 벗어남
    - catch에서 다시 exception이 발생하는 경우, 자원은 닫히지 않는 상태가 발생할 수 있어 try-with-resource는 이와 같은 문제를 해결
  ~~~java
  try (BufferedReader bufferedReader = new BufferedReader(new FileReader("/tmp"))) {
      bufferedReader.readLine();
  }
  ~~~
- throw
  - 개발자가 특정 상황에 예외를 발생시키기 위하여 throw 키워드를 사용
  - throw 뒤에 new 키워드를 사용하여 예외를 생성하고 발생시킴
~~~java
throw new IllegalArgumentException():
~~~
- throws
  - 예외 발생시, try-catch 구문으로 예외를 핸들링할 수 있지만 throws 키워드를 사용하여 자신을 호출한 메소드로 예외를 넘길 수 있음
  - 여러개의 예외를 ,(콤마)를 사용하여 명시할 수 있음
  - throws를 사용하여 계속해서 예외를 넘기는 경우, 어느곳에서도 처리하지 않으면 JVM에서 프로그램을 종료하는 현상 발생
~~~java
class Example {
    public static void main(String[] args) throws FileNotFoundException {
        FileReader fileReader = new FileReader("/tmp");
    }
}
~~~

</br>  

## 자바가 제공하는 예외 계층 구조
![throwable-structure](/img/throwable-structure.png)  
- Throwable 클래스는 Object 클래스로부터 시작
- Throwable 클래스는 Error와 Exception으로 나뉨
- Exception 클래스는 RuntimeException을 상속하는지에 따라 CheckedException과 UncheckedExcpetion으로 나뉨

</br>

## Exception과 Error의 차이는?
- Error
  - 프로그램/시스템 오류로 발생
  - 가장 대표적인 error는 OutOfMemoryError, IOError
    - OutOfMemoryError: 프로그램의 memory 리소스 사용량이 부족하여 발생하는 error
  - 발생시 복구 불가
- Exception
  - 개발자가 작성한 코드에 의해 주로 발생
  - 컴파일/런타임시 발생
  - 개발자에 의해 핸들링이 가능함
  - 발생시 복구 가능

</br>

## RuntimeException과 RE가 아닌 것의 차이는?
- RuntimeException을 상속하는지에 따라 CheckedException과 UncheckedExcpetion으로 나뉨
- CheckedException은 개발자에 의해 핸들링되지 않으면 컴파일 오류가 남
- RuntimeException은 말그대로 런타임 시점에 발생하는 예외로 컴파일 타임에 알 수 없음
  - RuntimeException을 사용하는 경우, 컴파일 타임에 에러를 알 수 없으므로 핸들링을 잘해야 함
  - 가장 대표적인 RuntimeException은 NPE(NullPointException)

</br>

## 커스텀한 예외 만드는 방법
- 비즈니스, 업무 단위에 따라 예외를 만들어 사용하면 정확한 예외 파악을 하기 쉬움
- 예외별 커스텀 처리 및 집합 처리를 하기에 유연함
- 커스텀 예외를 만드는 방법은 Exception 클래스를 상속하면 끝
- 실제 업무에서 커스텀 예외를 만들 경우, RuntimeException을 상속하여 사용
- 필요하다면 상위 클래스(RuntimeException, Excpetion, Throwable, Object)의 메소드를 오버라이딩 할 수 있음
~~~java
class Example {
    public static void main(String[] args) {
        throw new CustomException("custom excpetion!!");
    }
}


class CustomException extends RuntimeException {
    public CustomException() {
        super();
    }

    public CustomException(String message) {
        super(message);
    }

    public CustomException(String message, Throwable cause) {
        super(message, cause);
    }

    public CustomException(Throwable cause) {
        super(cause);
    }

    protected CustomException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}

~~~
