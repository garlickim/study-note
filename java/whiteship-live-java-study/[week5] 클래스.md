### 클래스 정의하는 방법
- 최소한의 클래스 선언 방법
  ~~~java
    class MyClass {
        // field, constructor, and 
        // method declarations
    }
  ~~~
  - 클래스 선언에는 public, private 등과 같은 접근 제어자를 붙일 수 있음
  - 클래스의 첫 글자는 대문자로 시작해야 함
  - 선언하는 클래스의 부모 클래스가 있는 경우 extends 키워드를 사용함
    - 클래스는 하나의 클래스만 확장 가능함
  - 하나의 클래스는 여러개의 인터페이스를 implement 키워드를 사용하여 확장 가능함
  - 클래스 바디는 {} 로 감싸져 있음


</br>

### 객체 만드는 방법(new 키워드 이해하기)
- new operator를 사용하여 객체를 생성함
- new operator 다음에는 생성자 호출이 이어지며, 새 객체를 초기화 함
  ~~~java
    Point point = new Point(10, 20);
  ~~~
- 아래 예와 같이 단순히 참조 변수를 선언하는 것은 객체를 생성하지 않음
  ~~~java
    Point point;
  ~~~
  
- new operator는 새로운 object를 위한 메모리를 할당하고, 그 메모리에 대한 참조를 반환하여 클래스를 인스턴스화 함 (클래스의 인스턴스화 == 객체 생성)

</br>

### 메소드 정의하는 방법
- 일반적인 메소드 선언은 아래와 같음
  ~~~java
  returnType methodName() {
    // method body
  }
  ~~~
- 메소드 선언에는 6개의 구성요소가 있음
  1. modifiers : 접근제어자, public/private/protected/default
  2. return type : 메소드가 리턴할 값의 type, return 값이 없는 경우 void
  3. method name : 메소드 이름
  4. parameter list in parenthesis : 매개 변수 목록, 괄호()로 묶임, 매개 변수가 없는 경우는 빈괄호 사용
  5. exception list : 메소드에서 발생할 수 있는 예외 목록
  6. method body : braces로 둘러쌓인 메소드 body, 지역 변수 및 코드가 포함 됨
- 메소드 이름 규칙
  - 소문자로된 동사 또는 여러 단어로 이루어짐
  - 여러 단어로 이루어진 경우, 다음 단어의 글자는 대문자로 작성
  - 메소도는 클래스 내에서 고유함 (method overload)
    ~~~java
    public class DataArtist { 
      ... 
      public void draw (String s) { 
          ... 
      } 
      public void draw (int i) { 
          ... 
      } 
    }
    ~~~
  
</br>

### 생성자 정의하는 방법
- 생성자(constructor) : 객체를 초기화하는데 사용됨
- 객체 속성에 대한 초기값을 설정하는데 사용 가능
- 생성자 이름은 클래스 이름과 일치해야하며, return type을 가질 수 없음
- 모든 클래스에서는 기본 생성자가 존재
- 클래스 생성자를 직접 생성하지 않으면 Compiler에 의해 기본 생성자가 생성 됨
~~~java
public class Car {
  public Car() { // 기본 생성자
  }
  
  public Car(String name){  // 파라미터가 있는 생성자
  }
}
~~~

</br>

### this 키워드 이해하기
- 메소드 또는 생성자의 현재 Object를 참조함
- 다음 용도로 사용 가능
  - 현재 클래스의 생성자 호출
  - 현재 클래스의 메소드 호출
  - 현재 클래스의 객체를 반환
  - 메소드 호출에서 parameter 전달
  - 생성자 호출에서 parameter 전달
~~~java
public class Point {
    public int x = 0;
    public int y = 0;
        
    //constructor
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

public class Rectangle {
    private int x, y;
    private int width, height;
        
    public Rectangle() {
        this(0, 0, 1, 1);
    }
}
~~~

</br>



## Optional
- int 값을 가지고 있는 이진 트리를 나타내는 Node 라는 클래스를 정의하세요.
- int value, Node left, right를 가지고 있어야 합니다.
- BinrayTree라는 클래스를 정의하고 주어진 노드를 기준으로 출력하는 bfs(Node node)와 dfs(Node node) 메소드를 구현하세요.
- DFS는 왼쪽, 루트, 오른쪽 순으로 순회하세요.

Node.java
~~~java
public class Node {
    Node left;

    Node right;

    int value;

    public Node() {
    }

    public Node(int value, Node left, Node right) {
        this.value = value;
        this.left = left;
        this.right = right;
    }
}
~~~

BinaryTree.java
~~~java
public class BinaryTree {
    Queue<Node> queue = new LinkedList<>();
    Queue<Integer> result = new LinkedList<>();

    // bfs : 너비 우선
    void bfs(Node root) {
        queue.add(root);
        while (!queue.isEmpty()) {
            Node tmpNode = queue.poll();
            result.add(tmpNode.value);

            if (tmpNode.left != null) {
                queue.add(tmpNode.left);
            }

            if (tmpNode.right != null) {
                queue.add(tmpNode.right);
            }
        }

    }

    Stack<Integer> stack = new Stack<>();
    Integer rootNumber = null;

    // dfs : 깊이 우선
    void dfs(Node root) {

        Boolean isRoot = false;

        if(rootNumber == null) {
            isRoot = true;
            rootNumber=root.value;
        } else {
            stack.add(root.value);
        }

        if (root.left != null) {
            dfs(root.left);
        }

        if(isRoot){
            stack.add(rootNumber);
        }

        if (root.right != null) {
            dfs(root.right);
        }
    }
}
~~~

BinaryTreeTest.java (test code)
~~~java
class BinaryTreeTest {

    @Test
    @DisplayName("bfs 탐색을 하면 1,2,3,4가 나와야한다")
    void bfs() {
        // given
        Node subNode_02 = new Node(3, null, null);
        Node subNode_03 = new Node(4, null, null);
        Node subNode_01 = new Node(2, subNode_03, null);
        Node node = new Node(1, subNode_01, subNode_02);

        // when
        BinaryTree binaryTree = new BinaryTree();
        binaryTree.bfs(node);


        // then
        assertEquals(List.of(1, 2, 3, 4), binaryTree.result);
    }


    @Test
    @DisplayName("bfs 탐색을 왼쪽 -> 루트 -> 오른쪽으로 하면 2,4,1,3이 나와야한다")
    void dfs() {
        // given
        Node subNode_02 = new Node(3, null, null);
        Node subNode_03 = new Node(4, null, null);
        Node subNode_01 = new Node(2, subNode_03, null);
        Node node = new Node(1, subNode_01, subNode_02);

        // when
        BinaryTree binaryTree = new BinaryTree();
        binaryTree.dfs(node);


        Stack<Integer> expected = new Stack<>();
        expected.add(2);
        expected.add(4);
        expected.add(1);
        expected.add(3);

        // then
        assertEquals(expected, binaryTree.stack);
    }
}
~~~
