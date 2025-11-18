## IO 병목, 어떻게 해결하지

### 블로킹과 논블로킹
+ 블로킹 IO
    ```java
    // 데이터를 읽는 동안 여기서 멈춰있음
    int data = inputStream.read();  // ← 이 줄에서 데이터가 올 때까지 대기
    // 데이터가 도착해야 다음 줄 실행
    System.out.println(data);
    ```

    - 작업이 완료될 때까지 스레드가 대기
    - 간단하지만 기다리는 동안 CPU가 낭비
    - 서버에서 동시에 100명이 접속하면? 100개의 스레드가 필요(메모리 부담 up)

+ 컨텍스트 스위칭
    - CPU가 한 작업에서 다른 작업으로 전환하는 것
        + CPU는 한번에 한가지 작업만 진행하지만 스레드를 스위칭하여 여러가지 작업을 진행하므로 동시에 실행되는 것처럼 보임
    - 컨텍스트 스위칭 과정
        + 현재 스레드 상태 저장
            - PC(Program Counter) : 현재 실행 위치
            - 레지스터 : CPU 레지스터 값
            - 스택 포인터
            - 기타 정보
            - OS의 PCB(Process Control Block)에 저장
        + 다음 스레드 선택
            - 실행 가능한 스레드 목록 확인
            - 우선순위, 대기 시간 등 고려
        + 새 스레드 상태 복원
            - PC 설정
            - 레지스터 복원
            - 스택 포인터 복원
            - 이전 멈춘 위치부터 계속 실행
    - 언제 컨텍스트 스위칭이 발생할까?
        + 타임 슬라이스 만료(OS가 각 스레드에 할당)
        + IO 블로킹
        + sleep() 호출
        + lock 대기

+ 논블로킹 IO
    ```java
    // 논블로킹 모드 설정
    channel.configureBlocking(false);

    // 데이터 읽기 시도
    int bytesRead = channel.read(buffer);

    if (bytesRead > 0) {
        // 데이터가 있으면 처리
        processData(buffer);
    } else {
        // 데이터 없으면 다른 일 하러 감
        doOtherWork();
    }
    ```

    - 데이터가 준비되지 않았으면 `즉시 리턴`
    - 스레드가 블록되지 않고 다른 일을 계속 할 수 있음
    - 하지만 데이터가 준비되었나 계속해서 확인해야함(polling)

+ 차이점 정리

        |구분|블로킹 IO|논블로킹 IO|
        |---|---|---|
        |대기 방식|작업 완료까지 멈춤|즉시 리턴, 계속 진행|
        |스레드 상태|대기|실행|
        |리소스 효율|요청 당 스레드 방식|스레드 적게 필요|
        |구현 난이도|쉬움|어려움(상태 관리가 추가되므로)|
        |적합한 경우|단순한 요청-응답|대량 연결 처리|

왜 중요할까?  
블로킹 방식은 요청 당 스레드로 처리되므로 사용자 1명당 스레드 1개가 필요하다  
사용자가 계속해서 늘어나면 그만큼 필요한 스레드 수도 늘어나는 상황

논블로킹은 스레드 몇개로 수천명을 처리할 수 있다  
하지만 코드가 복잡해짐  
그래서 `가상 스레드`나 `리액터 패턴`과 같은 방법이 등장했다  

### 가상 스레드란?
Java 21부터 도입된 **경량 스레드**  
기존 스레드(플랫폼 스레드)의 한계때문에 탄생했다  

```java
// 스레드 1만 개 만들기
for (int i = 0; i < 10000; i++) {
    new Thread(() -> {
        // 작업
    }).start();
}

// 결과: OutOfMemoryError! 💥
// 왜? 스레드 하나당 1MB 메모리 필요
// 10,000개 * 1MB = 10GB!
```

+ 플랫폼 스레드의 문제
    - 무겁다 (1개당 1MB)
    - 비싸다 (생성/제거 비용이 큼)
    - 제한적 (메모리 제한으로 많이 만들기 어려움)

```
[플랫폼 스레드 1개]
  |
  |─ [가상 A] 실행
  |    ↓ IO 대기 (unmount)
  |─ [가상 B] 실행  ← 즉시 다른 가상 스레드!
  |    ↓ IO 대기 (unmount)
  |─ [가상 C] 실행
  |    ↓ IO 대기 (unmount)
  |─ [가상 D] 실행
  |    ↓ IO 완료 (A가 준비됨!)
  |─ [가상 A] 재개  ← 다시 실행!
  
플랫폼 스레드 1개로 수천 개 가상 스레드 처리!
```

+ 가상스레드 사용의 유의사항
    - Pinning 문제
        
    ```java
    // ❌ 나쁜 예 - 가상 스레드가 "고정"됨
    Thread.startVirtualThread(() -> {
        synchronized (lock) {
            String data = httpClient.get(url);  // IO 대기
            // 문제: unmount 못함!
            // 플랫폼 스레드를 계속 점유
        }
    });

    // synchronized는 JVM의 모니터 락 사용
    // → OS 레벨 구현
    // → 가상 스레드를 unmount하면 락 정보 유실
    // → 따라서 unmount 못하고 고정(pin)됨
    // → 플랫폼 스레드 계속 점유
    // → 가상 스레드의 이점 사라짐!
    ```

### 셀렉터 패턴이란?  
Selector는 `여러개의 채널을 하나의 채널이 감시하는 패턴`이다

+ 동작원리
1. 준비(등록)

```java
// Selector 생성
Selector selector = Selector.open();

// 서버 소켓 채널 생성 및 논블로킹 설정
ServerSocketChannel serverSocket = ServerSocketChannel.open();
serverSocket.configureBlocking(false);  // 중요!
serverSocket.bind(new InetSocketAddress(8080));

// Selector에 등록 - "새 연결 올 때 알려줘!"
serverSocket.register(selector, SelectionKey.OP_ACCEPT);
```

2. 감시(대기)

```java
while (true) {
    // 준비된 이벤트가 있을 때까지 대기 (여기서만 블로킹!)
    selector.select();  
    
    // 준비된 이벤트들 가져오기
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iterator = selectedKeys.iterator();
    
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        iterator.remove();  // 처리한 이벤트는 제거
        
        // 이벤트 타입별로 처리
        if (key.isAcceptable()) {
            handleAccept(key);      // 새 연결
        } else if (key.isReadable()) {
            handleRead(key);        // 데이터 읽기
        } else if (key.isWritable()) {
            handleWrite(key);       // 데이터 쓰기
        }
    }
}
```

3. 처리

```java
// 새 연결 처리
private void handleAccept(SelectionKey key) {
    ServerSocketChannel server = (ServerSocketChannel) key.channel();
    SocketChannel client = server.accept();  // 논블로킹이라 즉시 리턴
    
    client.configureBlocking(false);
    // 이 클라이언트도 Selector에 등록 - "데이터 오면 알려줘!"
    client.register(selector, SelectionKey.OP_READ);
}

// 데이터 읽기 처리
private void handleRead(SelectionKey key) {
    SocketChannel client = (SocketChannel) key.channel();
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    
    int bytesRead = client.read(buffer);  // 논블로킹 읽기
    
    if (bytesRead > 0) {
        buffer.flip();
        // 데이터 처리
        processData(buffer);
        
        // 이제 쓰기 준비됨을 알려달라고 등록
        key.interestOps(SelectionKey.OP_WRITE);
    } else if (bytesRead == -1) {
        // 연결 종료
        client.close();
    }
}
```

```
스레드 1개 → Selector가 감시 → 메모리 1MB만 소비
일 있을 때만 깨어남 ⚡

[Selector] - 감시자
    |
    |-- [ServerSocketChannel] (연결 대기중)
    |
    |-- [SocketChannel 1] (읽기 준비됨!) ✅
    |-- [SocketChannel 2] (대기중...)
    |-- [SocketChannel 3] (쓰기 준비됨!) ✅
    |-- [SocketChannel 4] (대기중...)
    |-- ...
    |-- [SocketChannel 1000] (대기중...)
```

+ 셀렉터 패턴이 감시할 수 있는 이벤트
    - SelectionKey.OP_ACCEPT   // 새 연결 수락 가능
        + 서버 전용 이벤트
        + 클라이언트가 connect()를 호출했을 때
        + 3-way handshake 완료 후, 서버 측에서 요청을 수락 가능할 때
    - SelectionKey.OP_CONNECT  // 연결 완료
        + 클라이언트 전용
        + 논블로킹 connect() 호출 후
        + 서버와의 연결이 실제로 완료됐을 때
    - SelectionKey.OP_READ     // 읽기 가능
        + 상대방이 데이터를 보냈을 때
        + 읽을 데이터가 버퍼에 도착했을 때
        + 연결이 종료되었을 때(read()는 -1 반환)
    - SelectionKey.OP_WRITE    // 쓰기 가능
        + 소켓 버퍼에 쓸 공간이 있을 때
        + 대부분 true를 리턴(버퍼는 대부분 여유공간이 있으므로)
    - 왜 4개가 다인가?
        + OS 레벨 IO 멀티플렉싱의 한계(OS가 제공하는 기본 이벤트의 한계)
        + Netty는 4가지 기본 이벤트를 기반으로 좀 더 확장된 기능의 이벤트를 제공

+ 장점
    - 리소스 효율적 : 스레드를 적게 사용하니까
    - 확장성 좋음 : 채널만 추가하면 된다
    - 메모리 절약 : 컨텍스트 스위칭 비용 감소

+ 단점
    - 코드 복잡 : 상태 관리가 어려움
    - 디버깅 어려움 : 비동기라 흐름 파익이 어려움
    - CPU 작업 부적합 : IO 대기가 많을 때 유리

### IO 멀티플렉싱이란?
네트워크 프로그래밍에서 `여러개의 IO 채널을 하나의 스레드가 감시`하는 방식

+ 전통적 방식(동기)
    ```java
    // 클라이언트 1명당 스레드 1개
    while (true) {
        Socket client = serverSocket.accept();  // 블로킹
        
        new Thread(() -> {
            InputStream in = client.getInputStream();
            int data = in.read();  // 블로킹
            process(data);
        }).start();
    }
    ```

    **문제점:**
    ```
    [클라이언트 1] ─── [스레드 1] (대기 중... 😴)
    [클라이언트 2] ─── [스레드 2] (대기 중... 😴)
    [클라이언트 3] ─── [스레드 3] (대기 중... 😴)
    ...
    [클라이언트 1000] ─── [스레드 1000] (대기 중... 😴)

    메모리: 1000개 * 1MB = 1GB!
    대부분 시간을 놀면서 보냄
    ```

+ 멀티플렉싱
    ```java
    Selector selector = Selector.open();

    // 여러 채널을 하나의 Selector에 등록
    channel1.register(selector, SelectionKey.OP_READ);
    channel2.register(selector, SelectionKey.OP_READ);
    channel3.register(selector, SelectionKey.OP_READ);
    // ... 1000개 등록

    while (true) {
        // 모든 채널을 한 번에 감시
        selector.select();  // "준비된 채널 있으면 알려줘!"
        
        // 준비된 채널만 처리
        for (SelectionKey key : selector.selectedKeys()) {
            if (key.isReadable()) {
                handleRead(key);
            }
        }
    }
    ```

    **장점:**
    ```
    [클라이언트 1]  ╲
    [클라이언트 2]   ├─── [Selector] ─── [스레드 1개]
    [클라이언트 3]   │
    ...              │
    [클라이언트 1000]╱

    메모리: 거의 그대로 (스레드 1개!)
    준비된 채널만 처리하니까 효율적!
    ```

자바의 멀티플렉싱은 내부적으로 OS의 멀티플렉싱을 사용  
JVM이 알아서 선택하므로 개발자는 신경쓸 필요가 없다

+ 멀티스레딩 vs 멀티플렉싱
    - 멀티스레딩
        + 컨텍스트 스위칭 많이 발생
        + 컨텍스트 스위칭은 멀티스레딩의 핵심 비용
    - 멀티플렉싱
        + 컨텍스트 스위칭이 거의 없음

### 리액터 패턴이란?
리액터 패턴은 논블로킹 IO를 이용해서 구현하는 패턴  
리액터 패턴은 크게 리액터와 핸들러 두 요소로 구성된다  
먼저 리액터는 이벤트가 발생할 때까지 대기하다가 이벤트가 발생하면 알맞은 핸들러에 이벤트를 전달한다  
이벤트를 전달받은 핸들러는 필요한 로직을 수행한다

```java
// 1. 초기화
Selector selector = Selector.open();
ServerSocketChannel serverSocket = ServerSocketChannel.open();
serverSocket.register(selector, SelectionKey.OP_ACCEPT);

// 2. 이벤트 루프 (Reactor의 핵심!)
while (true) {
    // 3. 이벤트 대기 (Demultiplexer)
    selector.select();  // "이벤트 발생할 때까지 기다려!"
    
    // 4. 발생한 이벤트들 가져오기
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    
    // 5. 각 이벤트 처리 (Dispatch)
    for (SelectionKey key : selectedKeys) {
        if (key.isAcceptable()) {
            handleAccept(key);      // Handler 호출
        } else if (key.isReadable()) {
            handleRead(key);        // Handler 호출
        } else if (key.isWritable()) {
            handleWrite(key);       // Handler 호출
        }
    }
    
    selectedKeys.clear();
}
```

+ 리액터 패턴의 장점
    - 단일 스레드로 많은 연결 처리
        ```java
        // 전통적 방식
        for (int i = 0; i < 10000; i++) {
            new Thread(() -> {
                handleClient();  // 스레드 10,000개 필요!
            }).start();
        }

        // 리액터 패턴
        Selector selector = Selector.open();
        // 1만 개 클라이언트를 스레드 1개로 처리!
        while (true) {
            selector.select();
            // 준비된 것만 처리
        }
        ```

    - 리소스의 효율적인 사용
        + 스레드 수 적음
        + 메모리 사용량 적음
        + 컨텍스트 스위칭 없음
        ```
        메모리 사용:
        Thread-per-Connection: 10,000 * 1MB = 10GB
        Reactor: ~10MB
        
        컨텍스트 스위칭:
        Thread-per-Connection: 엄청 많음
        Reactor: 거의 없음
        ```

    - 확장성
        ```
        // 싱글 리액터 (간단한 경우)
        Reactor reactor = new Reactor(port);

        // 멀티 리액터 (고성능)
        Reactor mainReactor = new Reactor(port);      // Accept만 담당
        Reactor[] subReactors = new Reactor[4];       // Read/Write 담당
        ```

+ 리액터 패턴의 단점
    - 복잡한 구현 난이도
        + 상태 관리 어려움
        + 콜백 지옥
    - 디버깅 어려움
        + 비동기 흐름 추적 어려움
        + 스택트레이스 불명확