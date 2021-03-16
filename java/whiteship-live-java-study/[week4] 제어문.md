# Mandatory
### 선택문 (jdk8 기준으로 작성함)
- switch statement
- 여러개의 if문으로 구성할 수 있는 구문을 swich-case 문으로 표현
- switch 의 매개변수에 따라 블록이 구성됨
- switch는 기본 primitive type(char, int, short, char)에서 동작하며, primitive를 wrapping한 일부 클래스(Character, Byte, Short, Integer)와 몇가지 클래스(Enum, String)에서도 동작
- 기본 구문
~~~java
switch (조건){
    case 값1:
        코드;
        break;
    case 값2:
        코드;
        break;
    ...
    default:
        코드;
        break;
}
~~~

- if 문으로 표현된 구문을 switch 문으로 표현하면 ..
~~~java
String name = "garlic";

// if statement
if(name.equals("kelly"))
    System.out.println("she is kelly");
else if(name.equals("garlic"))
    System.out.println("she is garlic");
else if(name.equals("sally"))
    System.out.println("she is sally");
else
    System.out.println("none");

// switch statement
switch (name) {
    case "kelly":
        System.out.println("she is kelly");
        break;
    case "garlic":
        System.out.println("she is garlic");
        break;
    case "sally":
        System.out.println("she is sally");
        break;
    default:
        System.out.println("none");
        break;
}
~~~

- break 문
    - switch 구문 사용시 break 사용은 일치하는 case 문을 만나면 코드를 실행하고 switch 문을 종료
    - 일치하는 case문 실행 후 break를 사용하지 않으면, 이후의 case는 값이 일치하지 않아도 실행함!!! (주의해서 사용)
    ~~~java
    List<String> list = new ArrayList<>();
    String name = "garlic";
    
    switch (name) {
        case "kelly":
            list.add("kelly");
        case "garlic":
            list.add("garlic");
        case "sally":
            list.add("sally");
        default:
            break;
    }
    for (String s : list) {
        System.out.println(s); // 결과는 ?? garlic sally
    }
    ~~~

</br>

### 반복문
- for statement
    - 값의 범위를 반복할 때 사용
    - 값이 만족할 때까지 loop을 반복하므로 for loop으로 많이 불림
    - 기본 구문
    ~~~java
    for (initialization(초기화); termination(조건식); increment(증감)) {
        statement(s) // 식
    }
    ~~~
    - initialization(초기화) 는 구문 시작 시, 최초 값을 초기화하는데 1번 실행됨
    - termination(조건식) 은 false가 될 때까지, loop을 실행
    - increment(증가) 은 loop 실행시 매번 실행 됨. 값을 증가/감소 시키는데 사용함
    - for loop 밖에서 변수가 사용되지 않는다면, 초기화 구문에서 변수를 선언하고 초기화하는 것을 권장
        - 이유는?? 변수의 수명이 제한되고 오류가 줄어들기 때문!
    - 무한 loop를 만들고 싶은 경우는 for( ; ; )로 표현
    
- foreach statement
    - collection 또는 배열의 반복을 위해 확장 설계된 for statement
    - 기본 for loop 보다 간결하게 표현되어, 가독성이 높음
    ~~~java
    List<String> list = List.of("A", "B", "C");

    // 기본 for loop
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }

    // foreach 구문 (코드가 훨씬 간결함!)
    for (String s : list) {
        System.out.println(s);
    }
    ~~~

- while statement
    - while문 내의 조건이 true 일 동안, 코드 블록을 실행함
    - 기본 구문
    ~~~java
    while(조건식){
        // 코드작성
    }
    
    // example
    int num = 5;
    while (num > 0) {
        System.out.println(num); // 결과 : 5 4 3 2 1
        num--;
    }
    ~~~

- do-while statement
    - do 블록을 포함한 while 문
    - do 블록은 while 문과 관계없이 무조건 1번은 실행됨
    - while 문과의 차이는 판단식이 하단에 존재
    - 기본 구문
    ~~~java
    int num = 5;
    do {
        System.out.println(num); // 결과는 5 4 3 2 1
        num--;
    } while (num > 0);
    
    // 조건식을 만족하지 않게 바꾸면??
    int num = 5;
    do {
        System.out.println(num); // 결과는 5 (do 문은 while 문과 관계없이 최초 1번은 꼭 실행 됨)
        num--;
    } while (num > 5);
    ~~~

</br>

# Option
### 과제 1. live-study 대시 보드를 만드는 코드를 작성하세요.
~~~java
public class Application {

    private final static String TOKEN = "";

    public static void main(String[] args) {
        Application application = new Application();
        application.run();
    }

    private void run() {
        try {
            GitHub github = new GitHubBuilder().withOAuthToken(TOKEN).build();

            // Get Repository
            GHRepository repository = github.getRepository("garlickim/whiteship-live-java-study");

            // Get Issues
            List<GHIssue> issues = repository.getIssues(GHIssueState.ALL);

            // Count of attendance by person
            Map<String, Integer> countOfAttendance = calculateCountOfAttendance(issues);

            // Rate of attendance by person
            Map<String, String> rateOfAttendance = calculateRateOfAttendance(countOfAttendance);

            // print
            System.out.println("이름 | 참여율");
            for (String name : rateOfAttendance.keySet()) {
                System.out.println(name + " : " + rateOfAttendance.get(name));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private Map<String, Integer> calculateCountOfAttendance(List<GHIssue> issues) {
        Map<String, Integer> countOfAttendance = new HashMap<>();

        try {
            for (GHIssue issue : issues) {
                for (GHIssueComment comment : issue.getComments()) {
                    String name = comment.getUser().getName();

                    Integer count = countOfAttendance.containsKey(name) ? countOfAttendance.get(name) : 0;
                    countOfAttendance.put(name, ++count);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return countOfAttendance;
    }

    private Map<String, String> calculateRateOfAttendance(Map<String, Integer> countOfAttendance) {
        Map<String, String> rateOfAttendance = new HashMap<>();

        for (String name : countOfAttendance.keySet()) {
            Double avg = (countOfAttendance.get(name) / 18.0) * 100;
            rateOfAttendance.put(name, String.format("%.2f", avg));
        }
        return rateOfAttendance;
    }
}
~~~

</br>


### 과제 3. Stack을 구현하세요.
~~~java
public class Stack {
    private int size = 10;
    private int[] nums;
    private int top;

    public Stack() {
        nums = new int[size];
        top = -1;
    }

    public void push(int data) {
        top++;

        if (nums.length == top) {
            int[] tmpNums = new int[size * 2];
            for (int i = 0; i < nums.length; i++) {
                tmpNums[i] = nums[i];
            }
            tmpNums[top] = data;
            nums = tmpNums;
        } else {
            nums[top] = data;
        }
    }

    public int pop() {
        if (top == -1) {
            throw new EmptyStackException();
        } else {
            int result = nums[top];
            top--;
            return result;
        }
    }
}
~~~

테스트 코드
~~~java
class StackTest {
    Stack stack = new Stack();

    @BeforeEach
    void initialize() {
        stack.push(3);
        stack.push(4);
        stack.push(5);
    }


    @Test
    @DisplayName("3,4,5를 push 하고 pop한 결과는 5이다")
    void push() {
        assertEquals(5, stack.pop());
    }

    @Test
    @DisplayName("3,4,5를 push 하고 두번 pop 한 이후 8을 푸시&팝 한 결과는 8이다")
    void pop() {
        stack.pop();
        stack.pop();
        stack.push(8);

        assertEquals(8, stack.pop());
    }
}
~~~

</br>

### 과제 5. Queue를 구현하세요.
- 배열을 사용한 경우
    ~~~java
    public class Queue {
        private int size = 10;
        private int[] nums;
        private int last;

        public Queue() {
            nums = new int[size];
            last = -1;
        }

        public void enQueue(int data) {
            last++;

            if (nums.length == last) {
                int[] tmpNums = new int[size * 2];
                for (int i = 0; i < nums.length; i++) {
                    tmpNums[i] = nums[i];
                }
                tmpNums[last] = data;
                nums = tmpNums;
            } else {
                nums[last] = data;
            }
        }

        public int deQueue() throws Exception {
            if (last == -1) {
                throw new Exception("Queue is empty");
            } else {
                int result = nums[0];

                for (int i = 1; i < last; i++) {
                    nums[i - 1] = nums[i];
                }
                last--;
                return result;
            }
        }

        public int peek() throws Exception {
            if (last == -1) {
                throw new Exception("Queue is empty");
            } else {
                return nums[0];
            }
        }
    }
    ~~~
    테스트 코드
    ~~~java
    class QueueTest {
        Queue queue = new Queue();

        @BeforeEach
        void initialize() {
            queue.enQueue(1);
            queue.enQueue(2);
            queue.enQueue(3);
        }

        @Test
        @DisplayName("Queue에 1, 2, 3을 삽입 후 peek 하면 결과는 1이다")
        void enQueue() throws Exception {
            assertEquals(1, queue.peek());
        }

        @Test
        @DisplayName("Queue에 1, 2, 3을 삽입 후 3번의 deQueue 후 4를 삽입 & deQueue하면 결과는 41이다")
        void deQueue() throws Exception {
            queue.deQueue();
            queue.deQueue();
            queue.deQueue();
            queue.enQueue(4);

            assertEquals(4, queue.deQueue());
        }

        @Test
        @DisplayName("Queue에 1, 2, 3을 삽입 후 여러번 peek을 해도 결과는 1이다")
        void peek() throws Exception {
            queue.peek();
            queue.peek();

            assertEquals(1, queue.peek());
        }
    }
    ~~~
