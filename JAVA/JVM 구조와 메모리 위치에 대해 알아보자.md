## JVM 구조와 메모리 위치
### JVM의 구조
JVM은 크게 3가지 영역으로 나뉜다
+ Class Loader
    - 컴파일된 .class 파일을 읽어서 메모리에 올리는 역할
    - 로딩, 링킹, 초기화 과정을 거친다
+ Runtime Data Area
    - 메모리 영역
    - 프로그램 실행 중 데이터가 저장되는 공간
+ Execution Engine
    - 바이트코드를 실행하는 엔진
    - 인터프리터와 JIT 컴파일러가 위치한다
여기서 우리가 주목해야 할 영역은 Runtime Data Area
### Runtime Data Area의 세부 구조
JVM 메모리는 `스레드 공유 영역`과 `스레드 전용 영역`으로 나뉜다
+ 스레드 공유 영역
    - 모든 스레드가 공유
    - 메서드 영역 (Method Area)
        + 클래스 메타데이터, static 변수, 상수 등이 저장됨
        + JVM 시작시 생성되고 종료하면 사라짐
        + Java8부터는 메타스페이스라는 이름으로 Native Memory에 위치
    - 힙 영역 (Heap Area)
        + 객체 인스턴스와 배열이 저장되는 공간
        + 가비지 컬렉터의 주요 대상
        + Young Generation과 Old Generation으로 나뉜다
+ 스레드 전용 영역
    - 각 스레드마다 독립적인 영역
    - PC Register
        + 현재 실행 중인 JVM 명령어 주소를 저장
    - JVM Stack
        + 메서드를 호출할 때마다 스택 프레임이 생성
        + 로컬 변수, 파라미터, 리턴 값, 연산 중간 결과 등이 저장됨
    - Native Method Stack
        + 네이티브 메서드 실행을 위한 스택
스레드 공유 영역은 동기화가 필요하고 스레드 독립 영역은 동기화가 필요없다
### 각 변수별 저장 위치
+ static 변수 (클래스 변수)
    ```
    public class Example {
        private static int count = 0;  // Method Area에 저장
    }
    ```
    - 저장 위치 : 메서드 영역 (메타스페이스)
    - 클래스가 로딩될 때, 한번만 생성되고 모든 인스턴스가 공유
    - 프로그램 종료까지 메모리에 남아있는다
+ Instance 변수 (인스턴스 변수)
    ```
    public class Person {
        private String name;  // Heap에 저장
        private int age;      // Heap에 저장
    }
    ```
    - 저장 위치 : Heap
    - 객체가 생성될 때, 함께 생성
    - 해당 객체가 GC에 의해 제거되면 같이 제거
+ Local 변수 (지역 변수)
    ```
    public void calculate() {
        int result = 10;  // JVM Stack에 저장 (primitive)
        String text = "hello";  // 참조는 Stack, 실제 객체는 Heap
    }
    ```
    - 기본형 (primitive)
        + 저장 위치 : JVM Stack
        + 메서드가 호출될 때, 스택 프레임에 생성되고 메서드가 종료되면 즉시 사라짐
    - 참조형
        + 참조 위치 : JVM Stack, 실제 객체 : Heap
        + 변수는 스택에 실제 데이터는 힙에 존재
        + 참조가 Stack에서 사라지면 Heap의 객체는 GC의 대상
        ```
        public void process(int value, Person person) {
            // value: Stack에 저장 (primitive)
            // person: 참조는 Stack, 객체는 Heap
        }
        ```
+ 실제 메모리가 배치되는 예시
    ```
    public class MemoryExample {
        private static int staticVar = 100;      // Method Area
        private int instanceVar;                  // Heap
        public void method(int param) {           // param: Stack
            int localVar = 50;                    // Stack
            String str = new String("test");      // 참조: Stack, 객체: Heap
            Person person = new Person();         // 참조: Stack, 객체: Heap
        }
    }
    ```
### GC의 세대별 구조
#### 전통적인 Generation GC 구조 
왜 Young과 Old로 세대를 구분했을까?
IBM에서 연구한 결과에 따르면 대부분의 객체는 생성 후 금방 죽는다고 한다
이 특성을 사용하여 자주 GC를 진행하는 구역과 가끔 GC를 진행하는 구역을 분리한 것
Heap의 세대별 구조
```
Heap Memory
├── Young Generation (전체 힙의 1/3 정도)
│   ├── Eden Space (80%)
│   ├── Survivor 0 (10%)
│   └── Survivor 1 (10%)
└── Old Generation (전체 힙의 2/3 정도)
```
+ Young Generation
    - 빠르게 태어나고 빠르게 죽는 곳
    - Eden Space
        ```
        public void processRequest() {
            String temp = "임시 데이터";  // Eden에 생성
            StringBuilder sb = new StringBuilder();  // Eden에 생성
            // 메서드 끝나면 참조 사라짐 → GC 대상
        }
        ```
        + 객체가 처음 생성되는 곳
        + 여기서 대부분의 객체가 죽는다
        + GC 주기 : 매우 자주 발생 (Minor GC)
    - Minor GC 과정
        + Eden이 가득 참
        + 살아있는 객체만 Survivor 0으로 이동
        + Eden 비우기
        + 다음 GC때 Survivor 0 -> Survivor 1로 이동
        + 계속 반복하며 age 증가
        + -XX:MaxTenuringThreshold=15 : Old로 승격되는 age 값 (기본값 : 15)
+ Old Generation
    - Survivor에서 15번 살아남으면 승격
    - Eden에 들어가기 너무 크면 Old로 이동
    - Survivor에 공간이 부족하면 바로 승격
#### G1 GC의 등장
G1 GC는 전통적 GC와 다른 구조를 가진다
힙을 동일한 크기의 리전으로 분할
```
[E][E][S][E]
[O][O][H][E]
[O][E][S][O]
[O][E][S][O]
E = Eden
S = Survivor
O = Old
H = Humongous (큰 객체용, Region 크기의 50% 이상)
```
+ Young GC
    - Eden 리전 전체를 수집
    - 살아남는 객체를 survivor/old로 이동
    - STW가 발생하지만 짧다
+ Concurrent Marking
    ```
    // 애플리케이션 실행 중에 동시에 진행
    while (application.isRunning()) {
        g1gc.markLiveObjects();  // 어떤 객체가 살아있는지 표시
        g1gc.calculateGarbageRatio();  // Region별 쓰레기 비율 계산
    }
    ```
    - 백그라운드에서 애플리케이션과 동시에 실행
    - 각 리전의 쓰레기 비율 파악
+ Mixed GC
    - Old Generation을 한번에 청소하지 않고 쓰레기가 많은 리전부터 차례로 청소
    - -XX:MaxGCPauseMillis=200 : 정해진 퍼즈타임 안에서만 동작