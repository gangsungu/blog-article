## MDC에 대해 알아보자

**MDC(Mapped Diagnostic Context)**는 로깅 프레임워크에서 제공하는 기능으로 스레드별로 컨텍스트 정보를 저장하고 관리할 수 있게 도외주는 도구

### MDC의 동작원리
MDC는 내부적으로 ThreadLocal을 사용한다  
각 스레드마다 독립적인 Map을 가지고 있어 같은 스레드 내부에서는 언제나 저장된 값을 꺼내 사용할 수 있다

```java
// 요청 시작 시 컨텍스트 정보 저장
MDC.put("userId", "user123");
MDC.put("requestId", "req-456");

// 이후 어디서든 로그에 자동으로 포함됨
logger.info("주문 처리 시작"); 
// 출력: [userId:user123] [requestId:req-456] 주문 처리 시작

// 요청 종료 시 정리
MDC.clear();
```

MDC의 사용이 끝나면 clear()로 사용한 정보를 정리하는 것이 중요하다  
스레드 풀로 반납되는 과정에서 MDC를 정리하지 않기 때문에 다음 요청을 처리할 때, MDC가 남아있을 수 있다

### MDC의 내부구조

```
Thread-1 객체
├─ threadLocals (ThreadLocalMap)
│   ├─ MDC의 ThreadLocal → Map{userId: "A", orderId: "123"}
│   ├─ 다른 ThreadLocal → 다른 값
│   └─ ...

Thread-2 객체  
├─ threadLocals (ThreadLocalMap)
│   ├─ MDC의 ThreadLocal → Map{userId: "B", orderId: "456"}
│   └─ ...
```

MDC는 이중 맵 구조로 이루어져 있다  
각 스레드 객체가 자기만의 ThreadLocalMap을 가지고  
ThreadLocal 객체는 Map의 키로 사용된다  

```
ThreadLocal<Map<String, String>>
     ↓
Thread.threadLocals: ThreadLocalMap
     ↓ key: MDC의 ThreadLocal 인스턴스
     ↓ value: Map<String, String>
     ↓
```

### 비동기 처리에서 MDC 전달하기
같은 스레드에서 동작하는 동기 처리는 MDC가 정상 동작한다

```java
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        MDC.put("orderId", id);
        
        logger.info("주문 조회 시작");  // ✅ [orderId:123] 주문 조회 시작
        
        Order order = orderService.findById(id);
        
        logger.info("주문 조회 완료");  // ✅ [orderId:123] 주문 조회 완료
        
        return order;
    }
}
```

하지만 비동기 처리에서는 문제가 발생한다  
MDC는 내부적으로 ThreadLocal을 사용하는데, ThreadLocal은 스레드 간에 공유되지 않는다!

그렇다면 어떻게 MDC를 전달할 수 있을까?

1. 현재 스레드의 MDC를 캡처

```java
// Main Thread (요청 처리 스레드)
MDC.put("userId", "user-123");
MDC.put("requestId", "req-456");

// 이 시점의 MDC를 복사
Map<String, String> mdcContext = MDC.getCopyOfContextMap();
// mdcContext = {userId: "user-123", requestId: "req-456"}
```

2. 작업을 래핑

```java
public class MDCTaskDecorator implements TaskDecorator {
    
    @Override
    public Runnable decorate(Runnable originalTask) {
        // 1. 현재 스레드(Main)의 MDC 캡처
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();
        
        // 2. 원본 작업을 새로운 Runnable로 감싸기
        return () -> {
            try {
                // 3. 작업 실행 전: 워커 스레드에 MDC 설정
                if (mdcContext != null) {
                    MDC.setContextMap(mdcContext);
                }
                
                // 4. 실제 작업 실행
                originalTask.run();
                
            } finally {
                // 5. 작업 완료 후: MDC 정리
                MDC.clear();
            }
        };
    }
}
```

3. Executor 설정

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        
        // 여기가 핵심! Decorator 등록
        executor.setTaskDecorator(new MDCTaskDecorator());
        
        executor.initialize();
        return executor;
    }
}
```

상세한 흐름은 아래와 같다

```
[Main Thread - 요청 처리]
  1. MDC에 userId, requestId 설정
  2. @Async 메서드 호출
     ↓
[ThreadPoolTaskExecutor]
  3. TaskDecorator.decorate() 호출
     - 현재 스레드(Main)의 MDC 복사
     - 원본 작업(processOrder)을 래핑한 새 Runnable 생성
     ↓
  4. 래핑된 작업을 큐에 넣음
     ↓
[Worker Thread - 스레드 풀의 작업 스레드]
  5. 큐에서 작업 꺼내서 실행
     ↓
  6. 래핑된 Runnable 실행 시작
     - MDC.setContextMap(복사된 MDC) ← 여기서 MDC 복원!
     ↓
  7. 실제 processOrder() 실행
     - logger.info() ← MDC 정보 있음!
     ↓
  8. finally 블록 실행
     - MDC.clear() ← 정리
```

정리하면  
+ ThreadPoolTaskExecutor : 우체부
    - 작업을 전달하고 실행
+ TaskDecorator : 포장지
    - 작업 실행 전후로 할 일을 정의
+ MDC
    - 메모 내용(전달할 컨텍스트 정보)