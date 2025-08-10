## 파티셔닝
파티셔닝은 하나의 큰 테이블을 여러 개의 작은 물리적 단위로 나누는 기술

### 왜 파티셔닝을 사용하는가
+ 쿼리 성능 향상(필요한 파티션만 조회)
+ 관리의 용이성(특정 파티션만 백업/삭제 가능)
+ 병렬 처리 가능

### Range Partitioning
연속된 값의 범위를 기준으로 파티션을 나누는 방식

```sql
CREATE TABLE sales (
    id INT,
    sale_date DATE,
    amount DECIMAL(10,2)
)
PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

+ 날짜나 숫자 범위로 데이터를 자주 조회할 때
+ 오래된 데이터를 주기적으로 삭제할 때

### List Partitioning
명시적인 값 목록을 기준으로 파티션을 나누는 방식

```sql
CREATE TABLE customers (
    id INT,
    name VARCHAR(100),
    region VARCHAR(20)
)
PARTITION BY LIST COLUMNS(region) (
    PARTITION p_north VALUES IN ('서울', '경기', '인천'),
    PARTITION p_south VALUES IN ('부산', '대구', '울산'),
    PARTITION p_etc VALUES IN ('제주', '기타')
);
```

+ 카테고리나 지역별로 데이터를 분리할 때
+ 특정 값 그룹별로 자주 조회할 때

### Hash Partitioning
해시 함수를 사용해 데이터를 균등하게 분산하는 방식

```sql
CREATE TABLE users (
    id INT,
    username VARCHAR(50),
    email VARCHAR(100)
)
PARTITION BY HASH(id)
PARTITIONS 4;  -- 4개의 파티션으로 분할
```

+ 데이터가 자동으로 균등하게 분배
+ 파티션 개수를 지정하면 됨
+ 데이터베이스에서 자동으로 분배하므로 특정 파티션을 선택할 수 없다

### Key Partitioning
MySQL 내부의 해시 함수를 사용하는 방식(MySQL이 Hash 함수를 직접 선택하는 것이 Hash Partitioning과 다름)

```sql
CREATE TABLE orders (
    id INT,
    customer_id INT,
    order_date DATE,
    PRIMARY KEY(id, customer_id)  -- 복합키
)
PARTITION BY KEY(customer_id)
PARTITIONS 3;
```

### 샤딩과의 차이점
샤딩(Sharding)은 실제로 테이블이 분리된다

```sql
-- 샤딩 방식 (수동 관리)
CREATE TABLE Member_2022 (...);
CREATE TABLE Member_2023 (...);

-- 애플리케이션에서 직접 테이블 선택
INSERT INTO Member_2022 VALUES (...);  -- 개발자가 판단
SELECT * FROM Member_2022 WHERE ...;   -- 개발자가 지정
```

파티셔닝은 MySQL이 자동으로 관리한다

```sql
-- 파티셔닝 방식 (자동 관리)
INSERT INTO Member VALUES (...);       -- MySQL이 자동 분배
SELECT * FROM Member WHERE ...;        -- MySQL이 자동 선택
```