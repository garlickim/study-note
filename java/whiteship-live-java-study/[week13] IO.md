# 학습내용
- 스트림 (Stream) / 버퍼 (Buffer) / 채널 (Channel) 기반의 I/O
- InputStream과 OutputStream
- Byte와 Character 스트림
- 표준 스트림 (System.in, System.out, System.err)
- 파일 읽고 쓰기

</br>

## 스트림 (Stream) / 버퍼 (Buffer) / 채널 (Channel) 기반의 I/O
- I/O Streams  
  ![stream-io-read](/img/stream-io-read.png)
  ![stream-io-write](/img/stream-io-write.png)
  - 입력과 출력의 데이터 전송이 단방향으로 이루어짐
  - FIFO 구조를 지님

- Buffered Streams
  - 메모리 Buffer 영역을 사용하여 데이터를 읽고 씀
  - 필요한 시점에 Buffer에 쌓인 데이터를 한번에 처리

- 채널 (Channel) 기반의 I/O
  - Non Blocking I/O
  - 데이터가 지나가는 통로..?
  - 양방향의 입출력이 가능
  - buffer를 통해 읽거나 씀

</br>

## InputStream과 OutputStream
- byte stream 기반의 클래스들의 부모 클래스로, 추상 클래스임
- InputStream
  - AudioInputStream, ByteArrayInputStream, FileInputStream, FilterInputStream, ObjectInputStream, PipedInputStream, SequenceInputStream, StringBufferInputStream 의 서브 클래스가 존재
  ~~~java
  String filePath = "./study.txt";
  try(InputStream inputStream = new FileInputStream(filePath)){
      int i;
      while((i = inputStream.read()) != -1){
          System.out.write(i);
          // hello
          // kelly
      }
  }
  ~~~
  study.txt 
  ~~~txt
  hello  
  kelly  
  ~~~
- OutputStream
  - ByteArrayOutputStream, FileOutputStream, FilterOutputStream, ObjectOutputStream, PipedOutputStream의 서브 클래스가 존재
  ~~~java
  String filePath = "./study.txt";
  try(OutputStream outputStream = new FileOutputStream(filePath)){
      outputStream.write("HELLO".getBytes());
      outputStream.flush();
  }
  ~~~
  study.txt 
  ~~~txt
  HELLO  
  ~~~

</br>

## Byte와 Character 스트림
- Byte Stream
  - byte stream을 사용하여 8bit bytes의 입출력을 수행
  - 한번에 한바이트씩 처리
  - 더 이상 사용하지 않을 때, stream을 꼭 닫는 처리를 하여 리소스 누출을 방지
- Character Stream
  - 자바는 기본적으로 유니 코드 규칙을 사용하여 Character 값을 저장
  - byte stream의 속도가 더 빠르지만, 2byte의 유니코드를 다루기엔 byte stream보다는 charactor stream이 적절
  - Character Stream 클래스는 Reader 및 Writer 클래스를 상속받음
    - 대표적으로 FileReader, FileWriter가 있음
  ~~~java
  // FileReader
  String filePath = "./study.txt";
  try(FileReader reader = new FileReader(filePath)){
      int i;
      while((i = reader.read()) != -1){
          System.out.write(i);
          // hello
          // kelly
      }
  }
  
  // FileWriter
  String filePath = "./study.txt";
  try(FileWriter writer = new FileWriter(filePath)){
      writer.write("HELLO");
      writer.flush();
  }
  ~~~

</br>

## 표준 스트림 (System.in, System.out, System.err)
- 표준 스트림이란, 모니터/키보드등을 통하여 입출력하는 스트림
- java.lang.System 를 통해 표준 스트림을 제공하며, System.in, System.out, System.err가 있음
- System.in
  - 표준 입력 스트림
  - 스트림은 열려있는 상태로 입력을 받으면 됨
  - InputStream Type 으로 System.in의 객체를 할당  
  ![system.in](/img/system.in.png)
- System.out
  - 표준 출력 스트림
  - 스트림은 열러있는 상태로 데이터를 출력하면 됨
  - 호스트 환경이나 사용자가 지정한 출력 대상에 스트림이 출력 됨
  - PrintStream Type
  - 일반적으로 자바에서 콘솔에 데이터를 찍을 때 흔히 사용하던 System.out.println() 이 대표적인 예  
  ![system.out](/img/system.out.png)
- System.err
  - 표준 오류 출력 스트림
  - 스트림은 열려있는 상태로 출력 데이터를 받을 준비가 되어 있음
  - 사용자가 주목해야하는 오류 메시지 및 기타 정보를 출력하는데 사용
  - System.out과 마찬가지로 PrintStream Type  
  ![system.err](/img/system.err.png)

</br>

## 파일 읽고 쓰기
- File 클래스
  - 파일 크기, 속성, 이름 등의 정보를 얻고, 파일 생성 및 삭제 기능을 제공
  - | return type | method | description |   
    |---------|---------------------|--------------|  
    | boolean | canExecute() | 실행가능한 파일인지 확인 |  
    | boolean | canRead() | 읽을 수 있는 파일인지 확인 |  
    | boolean | canWrite() | 쓸 수 있는 파일인지 확인 |  
    | int | compareTo(File pathname) | 경로 이름 비교 |  
    | boolean | createNewFile() | 경로에 파일이 존재하지 않는 경우, 빈 파일 생성 |  
    | static File | createTempFile(String prefix, String suffix) | prefix와 suffix를 사용하여 파일 이름 생성 및 임시 파일 생성 | 
    | static File | createTempFile(String prefix, String suffix, File directory) | prefix와 suffix를 사용하여 파일 이름 생성 및 지정된 경로에 임시 파일 생성 |   
    | boolean | delete() | 파일 또는 디렉토리 삭제 | 
    | void | deleteOnExit() | VM이 종료될 때 해당 결로의 파일 또는 디렉토리를 삭제하도록 요청 |  
    | boolean | equals(Object obj) | object 비교 |  
    | boolean | exists() | 해당 경로에 파일 또는 디렉토리가 존재하는지 확인 |
    | File | getAbsoluteFile() | pathname의 절대 경로 리턴 |
    | String | getAbsoluteFile() | pathname의 절대 경로 리턴 |
    | File | getCanonicalFile() | pathname의 정식 경로 리턴 |
    | String | getCanonicalFile() | pathname의 정식 경로 리턴 |
    | long | getFreeSpace() | 해당 경로의 빈 공간 크기 리턴 |
    | String | getName() | 파일 또는 디렉토리의 이름 리턴 | 
    | String | getParent() | 추상 경로의 부모 경로명 리턴, 추상 경로에 부모 디렉토리를 지정하지 않으면 null 리턴 | 
    | File | getParentFile() | 추상 경로의 부모 경로명 리턴, 추상 경로에 부모 디렉토리를 지정하지 않으면 null 리턴 |
    | String | getPath() | 절대 경로 리턴 |
    | long | getTotalSpace() | 해당 경로의 파티션 크기 리턴 | 
    | long | getUsableSpace() | 해당 경로의 사용할 수 있는 공간의 바이트를 리턴 |
    | int | hashCode() | hachCode 리턴 | 
    | boolean | isAbsolute() | 절대경로인지 여부 리턴 |
    | boolean | isDirectory() | 디렉토리 여부 리턴 |
    | boolean | isFile() | file 여부 리턴 |
    | boolean | isHidden() | 숨김 파일인지 여부 리턴 |
    | long | lastModified() | 최근 수정된 시간 시턴 |
    | long | length() | 파일의 길이 리턴 |
    | String[] | list() | 파일 및 디렉토리를 나타내는 문자열 배열 리턴 | 
    | String[] | list(FilenameFilter filter) | filter를 만족하는 파일 및 디렉토리를 나타내는 문자열 배열 리턴 |
    | File[] | listFiles() | 지정된 경로 아래의 파일 및 디렉토리 리스트를 리턴 | 
    | File[] | listFiles(FileFilter filter) | filter를 만족하는 지정된 경로 아래의 파일 및 디렉토리 리스트를 리턴 | 
    | File[] | listFiles(FilenameFilter filter) | filter를 만족하는 지정된 경로 아래의 파일 및 디렉토리 리스트를 리턴 |
    | static File[]	| listRoots() | 사용 가능한 파일 시스템 루트를 리턴 |
    | boolean	| mkdir() | 해당 경로에 디렉토리 생성 |
    | boolean	| mkdirs() | 해당 경로에 디렉토리 생성. 부모 디렉토리가 없는 경우 생성 |
    | boolean	| renameTo(File dest) | 파일 이름 변경 |
    | boolean	| setExecutable(boolean executable) | executable에 따라 실행 가능한/불가능한 파일로 셋팅 |
    | boolean	| setExecutable(boolean executable, boolean ownerOnly) | executable에 따라 실행 가능한/불가능한 파일로 셋팅. ownerOnly에 따라 실행권한 설정 |
    | boolean	| setLastModified(long time) | 주어진 time으로 최근 수정시간을 변경 |
    | boolean	| setReadable(boolean readable) | readable에 따라 읽기가 가능한/불가능한 파일 또는 디렉토리로 셋팅 |
    | boolean	| setReadable(boolean readable, boolean ownerOnly) | executable에 따라 읽기가 가능한/불가능한 파일로 셋팅. ownerOnly에 따라 읽기 권한 설정 |
    | boolean	| setReadOnly() | 읽기만 가능하도록 파일 또는 디렉토리 권한 변경 |
    | boolean	| setWritable(boolean writable) | writable에 따라 읽기가 가능/불가능 하도록 설정 |
    | boolean	| setWritable(boolean writable, boolean ownerOnly) | writable에 따라 읽기가 가능/불가능 하도록 설정. ownerOnly에 따라 쓰기 권한 설정 |
    | Path |	toPath() | 해당 경로의 java.nio.file.Path 객체 리턴 |
    | String | toString() | 해당 경로의 string 값 리턴 |
    | URI | toURI() | 해당 경로를 나타내는 URI 타입의 값을 리턴 |
- FileReader/FileWriter 샘플은 위의 예시 코드 참고
