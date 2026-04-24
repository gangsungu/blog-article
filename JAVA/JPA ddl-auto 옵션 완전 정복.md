## JPA ddl-auto 옵션 완전 정복
DDL (Data Definition Language)는 데이터베이스의 구조를 정의하는 SQL 명령어  
spring.jpa.hibernate.ddl-auto 옵션은 애플리케이션이 실행될 때, Hibernate가 DB 스키마를 어떻게 처리할지 결정하는 설정

+ none
    - 아무것도 하지않음
        + Hibernate가 스키마를 작업하지 않음, DB는 준비되었다고 가정한다
    - 사용시점
        + 서비스 운영 환경
        + 스키마는 Flyway나 Liquibase와 같은 마이그레이션 툴로 관리
+ validate
    - 엔티티 <-> 테이블 구조 검증만 진행
        + 테이블을 변경하거나 생성하지 않고 JPA 엔티티와 실제 DB 테이블의 컬럼/타입이 일치하는지만 확인
        + 불일치시 애플리케이션 구동 실패
    - 사용시점
        + 운영 환경에서 스키마 정합성을 확인하고 싶을때
        + none보다 한단계 안전장치 역할
+ update
    - 변경된 부분만 update로 반영
        + 기존 테이블은 유지하면서 새로 추가된 컬럼이나 테이블만 DB에 반영
            - 컬럼 삭제는 반영 X
            - 엔티티에서 지워도 DB 컬럼은 그대로 남아있음 (단방향 수정)
        + 컬럼 이름 변경시 기존 컬럼은 두고 새 컬럼을 추가하므로 데이터 유실 위험 존재
    - 사용시점
        + 로컬 개발 환경
        + 데이터 유실 위험이 있으므로 운영 환경엔 절대 금지
+ create
    - 애플리케이션 시작할 때, 기존 테이블 DROP -> CREATE
        + 앱을 재시작할 때, 기존 테이블을 전부 삭제하고 새로 생성
        + 데이터 전부 유실
    - 사용시점
        + 초기 개발 단계에서 스키마가 자주 변경될 때
+ create-drop
    - 애플리케이션 시작할 때, CREATE 종료할 때 DROP
    - 사용시점
        + 테스트 환경 (테스트 후 깔끔히 정리되어야 할 때)

|옵션|시작 시|종료 시|데이터 유지|권장 환경|
|---|---|---|---|---|
|none|아무것도 안 함|-|유지|운영|
|validate|엔티티와 테이블 간 검증 진행|-|유지|운영|
|update|ALTER (추가만)|-|다른 컬럼은 유지<br/>ALTER 컬럼만 변경|로컬 개발 환경|
|create|DROP > CREATE|-|삭제|로컬/초기 개발|
|create-drop|DROP > CREATE|DROP|삭제|테스트|

### 확장-축소 (Expand-contract) 패턴
운영 서버는 보통 무중단 배포를 목표로 한다  
그런데 스키마를 변경했다면 여기서 문제가 발생한다  

```
컬럼 이름을 user_name에서 username으로 변경

배포 전 앱 : select user_name from users
배포 후 앱 : select username from users

DB 마이그레이션 직후, 앱 배포 전 순간에는 컬럼의 불일치로 인해 구버전 앱에서 장애가 발생!
```

확장-축소 패턴은 불일치 문제를 크게 3단계로 나누어서 해결한다

+ Expand
    - 기존 구조는 수정하지 않고 새 구조를 추가
        ```sql
        -- user_name은 그대로 두고, username 컬럼을 새로 추가
        -- 구버전 앱은 user_name을 사용하므로 장애가 발생하지 않는다
        ALTER TABLE users ADD COLUMN username VARCHAR(50);
        ```
+ Migration
    - 두 컬럼을 동시에 쓴다
        ```java
        // 배포된 앱이 두 컬럼 모두에 데이터를 씀
        user.setUserName(name);   // 기존 컬럼 (하위 호환)
        user.setUsername(name);   // 새 컬럼
        ```
        + 기존 데이터를 백필한다
            ```sql
            -- 기존 데이터를 새 컬럼으로 복사
            UPDATE users SET username = user_name WHERE username IS NULL;
            ```
+ Contrace
    - 구버전 제거
        ```java
        // user.setUserName() 제거, username만 사용
        user.setUsername(name);
        ```
        + 구 버전을 읽는 코드를 제거하고 이후 컬럼을 drop
            ```sql
            ALTER TABLE users DROP COLUMN user_name;
            ```
