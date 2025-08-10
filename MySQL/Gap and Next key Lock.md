## Gap Lock과 Next key Lock

결론부터 말하자면 데이터베이스에서 `Phantom Read` 문제를 해결하기 위한 락

### Phantom Read란?

한 트랜잭션에서 같은 쿼리를 두번 실행했는데, 두번째 실행에서 첫번째에서 보지못한 새로운 행이 나타나는 현상

```sql
-- 트랜잭션 A
SELECT * FROM users WHERE age > 25; -- 결과: 3개 행

-- 이 시점에 트랜잭션 B가 age=30인 새 사용자 추가
SELECT * FROM users WHERE age > 25; -- 결과: 4개 행 (Phantom!)
```

### Gap Lock
Gap Lock은 존재하지 않는 구간을 잠그는 락  

예를 들어, 인덱스에 [10, 15, 20] 등의 값이 존재하면

- 10 이전 구간
- 10과 15 사이의 구간
- 15와 20 사이의 구간
- 20 이후의 구간

```sql
-- age 컬럼에 인덱스가 있고, 현재 값이 [20, 30, 40]이라고 가정
SELECT * FROM users WHERE age = 25 FOR UPDATE;

-- age=25는 존재하지 않지만, (20, 30) 구간에 Gap Lock 설정
-- 다른 트랜잭션이 age=21~29 값을 INSERT하는 것을 막음
```

### Record Lock
기존 레코드의 `동시 수정/삭제`를 막는 락

```sql
-- users 테이블: id(PK), name, balance
-- 현재 데이터: id=5, name='김철수', balance=10000

-- 트랜잭션 A: 계좌 잔고 확인 후 차감 예정
BEGIN;
SELECT balance FROM users WHERE id = 5 FOR UPDATE; 
-- Record Lock 설정 → id=5 레코드 잠김
-- balance = 10000 확인

-- 비즈니스 로직 처리 중... (시간이 좀 걸림)

-- 트랜잭션 B: 동시에 같은 계좌에서 출금 시도
UPDATE users SET balance = balance - 3000 WHERE id = 5;
-- 🚫 블록됨! Record Lock 때문에 대기

-- 트랜잭션 A: 출금 처리
UPDATE users SET balance = balance - 5000 WHERE id = 5;
-- balance = 5000으로 업데이트
COMMIT;

-- 이제 트랜잭션 B 실행 가능
-- balance = 2000으로 업데이트 (5000 - 3000)
```

만약 Record Lock이 없다면?

```sql
-- Record Lock이 없는 상황을 가정

-- 트랜잭션 A
SELECT balance FROM users WHERE id = 5; -- balance = 10000
-- 비즈니스 로직 처리...

-- 트랜잭션 B (동시 실행)
SELECT balance FROM users WHERE id = 5; -- balance = 10000
UPDATE users SET balance = balance - 3000 WHERE id = 5; -- balance = 7000

-- 트랜잭션 A
UPDATE users SET balance = balance - 5000 WHERE id = 5; 
-- 💥 문제! 10000 - 5000 = 5000으로 업데이트
-- 트랜잭션 B의 -3000이 무시됨 (Lost Update!)
```

유니크 제약조건은 중복 insert만 막고 Record Lock은 기존 레코드의 동시 수정/삭제를 방지한다.  
즉, Record Lock은 동시성 제어를 위한 락이고 유니크 제약조건은 데이터 무결성을 위한 것

### Next Key Lock
실제 존재하는 레코드와 그 이전 구간을 함께 잠그는 락  
REPEATABLE READ 격리 수준에서 자동으로 사용된다.

Next key Lock = Record Lock + Gap Lock

```sql
-- 현재 age 값이 [20, 30, 40]인 상황에서
SELECT * FROM users WHERE age >= 30 FOR UPDATE;

-- age=30 레코드에 Record Lock
-- (20, 30] 구간에 Gap Lock  
-- age=40 레코드에 Record Lock
-- (30, 40] 구간에 Gap Lock
-- (40, +∞) 구간에 Gap Lock
```

```sql
-- 테이블 상태: users 테이블, age 컬럼에 인덱스
-- 현재 데이터: age = [10, 20, 30]

-- 트랜잭션 A
BEGIN;
SELECT * FROM users WHERE age > 15 FOR UPDATE;
```

1. Record Lock 효과로 기존 레코드의 수정/삭제가 차단됨
```sql
-- 트랜잭션 B에서 이런 작업들이 블록됨
UPDATE users SET name = '김철수' WHERE age = 20; -- 🚫 블록!
UPDATE users SET name = '이영희' WHERE age = 30; -- 🚫 블록!
DELETE FROM users WHERE age = 20;               -- 🚫 블록!
```

2. Gap Lock 효과로 새로운 레코드의 삽입이 차단됨
```sql
-- 트랜잭션 B에서 이런 작업들이 블록됨
INSERT INTO users VALUES ('김철수', 25);  -- 🚫 블록! (20,30) 구간
INSERT INTO users VALUES ('박민수', 35);  -- 🚫 블록! (30,+∞) 구간
INSERT INTO users VALUES ('정수현', 16);  -- 🚫 블록! (10,20] 구간
```

실제로 적용된다면 아래와 같은 상황이 확인

```sql
-- 현재 데이터
SELECT * FROM users;
-- id=1, name='user1', age=10
-- id=2, name='user2', age=20  
-- id=3, name='user3', age=30

-- 트랜잭션 A
BEGIN;
SELECT * FROM users WHERE age > 15 FOR UPDATE;

-- 트랜잭션 B (모두 블록됨)
UPDATE users SET name = 'updated' WHERE id = 2;  -- age=20 레코드 수정 시도 🚫
DELETE FROM users WHERE age = 30;                -- age=30 레코드 삭제 시도 🚫
INSERT INTO users VALUES (4, 'new', 25);         -- 새 레코드 삽입 시도 🚫

-- 트랜잭션 B (성공)
UPDATE users SET name = 'success' WHERE age = 5; -- age=5는 영향받지 않음 ✅
INSERT INTO users VALUES (5, 'young', 8);        -- age=8은 (∞,10) 구간이라 성공 ✅
```

* 정리  
Next Key Lock이 설정되면
    
+ 조건에 맞는 기존 레코드들은 수정/삭제 불가 (Record Lock)
+ 조건에 맞는 새로운 레코드들은 삽입 불가 (Gap Lock)
> `기존 데이터 보호` + `새로운 데이터 삽입 방지`를 동시에 진행하며 Phantom Read를 완벽하게 차단한다

### Range Lock
특정 범위(구간)을 잠그는 락, 구현방식은 DBMS마다 다르다

+ Range Lock과 Next Key Lock의 비교
    - 공통점
        - 둘 다 범위를 잠그는 개념
        - Phantom Read 방지목적
        - 인덱스 기반으로 동장

    - 차이점

    |구분|Range Lock|Next Key Lock|
    |---|---|---|
    |범위 설정|논리적 범위|인덱스 키 간격 기준|
    |세밀함|정확한 범위만 잠금|인덱스 구조에 의존|
    |유연성|높음|상대적으로 낮음|

### 표준 SQL과 MySQL의 차이점
트랜잭션 격리수준에 대해 공부할 때, REPEATABLE READ는 Phantom Read가 발생할 수 있다고 배웠다.  
하지만 바로 위에서 Next Key Lock은 Phantom Read를 완벽하게 차단한다고 배웠다😟
서로 말이 다르다!

이유는 MySQL과 다른 표준 SQL의 REPEATABLE READ는 살짝 다르다.
+ 표준 SQL REPEATABLE READ
    - Phantom Read 발생 가능
    - Next Key Lock 사용하지 않음
    - Record Lock만 사용

+ MySQL(InnoDB) REPEATABLE READ
    - Phantom Read 발생하지 않음
    - Next Key Lock 자동 사용
    - 결론적으로 SERIALIZABLE 수준의 격리 제공