
## MVCC란?

MVCC(Multi-Version Concurrency Control)은 데이터베이스에서 동시성 제어를 위해 사용하는 기법 중 하나다.  
쉽게 말하면 여러 트랜잭션이 동시에 데이터에 접근할 때, 서로 방해받지 않고 일관된 데이터를 읽을수 있도록 도와주는 메커니즘이다.

### 왜 MVCC가 필요할까?

전통적인 Lock 기반 시스템에서는 

- 트랜잭션 A가 데이터를 수정하는 동안 트랜잭션 B는 기다려야 함
- 읽기 작업도 쓰기 작업을 기다려야 하는 경우가 있음
- 성능저하와 데드락 위험성 존재

### MVCC의 핵심 아이디어

MVCC는 `데이터의 여러 버전을 유지`한다는 개념

- 각 트랜잭션은 자신만의 데이터 스냅샷을 본다
- 읽기 작업은 쓰기 작업을 기다리지 않는다
- 쓰기 작업은 읽기 작업을 기다리지 않는다

### 동작원리

```
시간 순서: T1 → T2 → T3

원본 데이터: { id: 1, name: "김철수", age: 25 }

T1: UPDATE age = 26 (버전 1 생성)
T2: SELECT * (여전히 age = 25를 봄 - 자신의 스냅샷)
T3: SELECT * (age = 26을 봄 - 최신 커밋된 버전)
```

### 주요 구성요소

트랜잭션 ID
- 각 트랜잭션마다 고유한 ID 부여
- 시간순서를 알려주는 타임스탬프 역할  

스냅샷 격리
- 각 트랜잭션은 시작 시점의 데이터를 기억함

MySQL의 경우  
- Undo Log를 통해 이전 버전 데이터를 관리하고
- Read View를 통해 트랜잭션별 가시성을 결정한다

### MVCC의 장단점

* 장점
  + 읽기 성능 향상(Lock 대기 없음)
  + 높은 동시성 지원
  + 일관된 읽기 보장
 
* 단점
  + 추가 저장공간 필요(여러 버전 저장)
  + 가비지 컬렉션 오버헤드
 
### 동시성 제어의 전체 분류

```
동시성 제어 기법
├── MVCC (기본 메커니즘)
├── 비관적 락 (Pessimistic Lock)
└── 낙관적 락 (Optimistic Lock)
```

* 비관적 락(Pessimistic Lock)
    - 충돌이 일어날거라는 가정 하에 `미리 락을 걸어두는` 방식

    ```java
    // JPA에서 비관적 락
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Product findByIdWithPessimisticLock(@Param("id") Long id);
    
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        // 1. 미리 락을 걸고 조회
        Product product = productRepository.findByIdWithPessimisticLock(productId);
        
        // 2. 다른 트랜잭션은 이 시점에서 대기
        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        
        // 3. 안전하게 재고 감소
        product.decreaseStock(quantity);
        
        // 4. 커밋 시 락 해제
    }
    ```

* 낙관적 락
    - `충돌이 드물거라는 가정` 하에 버전을 체크해서 충돌을 감지하는 방식
    
    ```java
    @Entity
    public class Product {
        @Id
        private Long id;
        
        private String name;
        private Integer stock;
        
        @Version  // 낙관적 락을 위한 버전 필드
        private Long version;
    }
    
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        // 1. 락 없이 조회 (빠름!)
        Product product = productRepository.findById(productId);
        
        // 2. 비즈니스 로직 수행
        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        
        product.decreaseStock(quantity);
        
        // 3. 저장 시 버전 체크
        productRepository.save(product);
        // UPDATE products SET stock = ?, version = version + 1 
        // WHERE id = ? AND version = ?
        
        // 만약 다른 트랜잭션이 먼저 수정했다면 OptimisticLockException 발생!
    }
    ```

### 비관적 락과 낙관적 락의 실제 동작 비교

동일한 시나리오 : 재고 10개의 상품을 두명이 동시에 5개씩 주문

1. MVCC만 사용(기본)
```java
@Transactional
public void orderWithMVCC(Long productId, int quantity) {
    Product product = productRepository.findById(productId);  // MVCC 읽기
    
    if (product.getStock() >= quantity) {
        product.decreaseStock(quantity);  // 둘 다 재고 10으로 보고 차감
        productRepository.save(product);
    }
    // 결과: 재고가 -5가 될 수 있음! (Lost Update)
}
```

2. 비관적 락
```java
@Transactional  
public void orderWithPessimistic(Long productId, int quantity) {
    Product product = productRepository.findByIdWithPessimisticLock(productId);
    // 두 번째 트랜잭션은 여기서 대기...
    
    if (product.getStock() >= quantity) {
        product.decreaseStock(quantity);
        productRepository.save(product);
    }
    // 결과: 첫 번째는 성공(재고 5), 두 번째도 성공(재고 0)
}
```

3. 낙관적 락
```java
@Transactional
public void orderWithOptimistic(Long productId, int quantity) {
    Product product = productRepository.findById(productId);  // version=1
    
    if (product.getStock() >= quantity) {
        product.decreaseStock(quantity);
        productRepository.save(product);  // version을 2로 업데이트 시도
    }
    // 결과: 첫 번째는 성공, 두 번째는 OptimisticLockException 발생
}
```

### 성능과 사용사례 비교

처리량:     낙관적 락 > MVCC > 비관적 락
안정성:     비관적 락 > 낙관적 락 > MVCC  
대기시간:   MVCC ≈ 낙관적 락 < 비관적 락
복잡성:     MVCC < 낙관적 락 < 비관적 락

- 비관적 락을 사용하는 경우
```java
// 1. 충돌 가능성이 높은 경우
@Transactional
public void processBankTransfer(Long accountId, BigDecimal amount) {
    // 계좌 이체는 충돌이 치명적이므로 비관적 락
    Account account = accountRepository.findByIdWithLock(accountId);
}

// 2. 재시도가 비용이 큰 경우  
@Transactional
public void processPayment(String orderId) {
    // 결제 처리는 재시도 비용이 크므로 비관적 락
    Order order = orderRepository.findByIdWithLock(orderId);
}
```

- 낙관적 락을 사용하는 경우
```java
// 1. 충돌 가능성이 낮은 경우
@Transactional
public void updateUserProfile(Long userId, UserProfileDto dto) {
    // 프로필 수정은 충돌이 드물므로 낙관적 락
    User user = userRepository.findById(userId);
    user.updateProfile(dto);
}

// 2. 높은 처리량이 필요한 경우
@Transactional  
public void incrementViewCount(Long articleId) {
    // 조회수 증가는 속도가 중요하므로 낙관적 락 + 재시도
    Article article = articleRepository.findById(articleId);
    article.incrementViewCount();
}
```

- MVCC만으로 충분한 경우
```java
@Transactional(readOnly = true)
public List<Product> getProductList() {
    // 단순 조회는 MVCC만으로 충분
    return productRepository.findAll();
}
```
