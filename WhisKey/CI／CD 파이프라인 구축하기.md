## CI/CD 파이프라인 구축하기

### CI/CD의 기초개념
CI (Continuous Integration, 지속적 통합)은 개발자들이 코드를 자주 메인 브랜치에 통합하는 개발 방식  
CD (Continuous Deployment, 지속적 배포)는 테스트를 통과한 코드가 자동으로 프로덕션까지 배포되는 방식  
사람의 개입이 없는 완전 자동화를 추구  

왜 CI/CD가 나왔을까?  

**전통적인 방식의 문제점:**
- 배포 주기가 길어서 피드백이 늦음
- 수동 배포 과정에서 실수 발생
- "내 컴퓨터에서는 되는데..." 같은 환경 차이 문제

**CI/CD의 장점:**
- 버그를 빨리 발견하고 수정
- 배포 위험 감소 (작은 단위로 자주 배포)
- 반복 작업 자동화로 생산성 향상
- 일관된 배포 프로세스

```
feature 브랜치에서 작업
    ↓
커밋 & Push
    ↓
CI가 feature 브랜치에서도 자동 빌드/테스트 실행 ← 여기가 핵심!
    ↓
PR(Pull Request) 생성
    ↓
코드 리뷰
    ↓
승인 후 main 브랜치에 merge
    ↓
main에서 다시 CI 실행
```

### CI/CD 파이프라인 전체 로드맵

+ Phase 1 (기본 CI 구축)
    - GitHub Actions로 빌드/테스트 자동화
    - 테스트 환경 격리 (MySQL, Redis 등)

+ Phase 2 (컨테이너화)
    - Docker 이미지 빌드 자동화
    - 이미지 레지스트리 푸시 (Docker Hub나 ECR)
+ Phase 3 (CD 구축)
    - AWS 배포 자동화
    - 무중단 배포 전략

여기서 이미지 레지스트리란?  
Docker 이미지를 저장하는 저장소다  
GitHub가 코드 저장소라면 이미지 레지스트리는 Docker 이미지 저장소라고 볼 수 있다  
대표적인 서비스로는 Docker Hub, AWS ECR, GitHub Container Registry 등이 있다

+ 실제 워크플로우
    - 빌드 단계 (CI)
        ```
        코드 푸시 → GitHub Actions 실행
        → Docker 이미지 빌드
        → 이미지를 레지스트리에 푸시
        ```
    - 배포 단계 (CD)
        ```
        AWS EC2 접속
        → 레지스트리에서 이미지 Pull
        → 컨테이너 실행
        ```

그렇다면 이미지 레지스트리가 왜 필요한가?  

+ 이미지 레지스트리없이 빌드
    - CI에서 빌드한 이미지가 GitHub Actions 내에 존재
    - 배포 서버에서 접근 불가
    - 매번 배포 서버에 직접 빌드해야 함
+ 이미지 레지스트리에 빌드하면
    - CI에서 빌드한 이미지를 어디서든 가져올 수 있음
    - 같은 이미지로 개발/스테이징/운영에 배포가능
    - 이미지 버전 관리

### docker-compose.yml 분리
우선 로컬 개발용과 CI 테스트용 docker-compose.yml을 분리하자  
분리하는 이유는 docker-compose.test.yml에서는 볼륨 설정을 사용하지 않는다  

왜 볼륨 설정을 하지않는데?  

+ 테스트 격리 원칙 위반
    - 이전 테스트 실행 결과가 남아있을 수 있다 (A 위스키를 품절처리했는데, 다음 CI 테스트때 A 위스키가 품절)
    - 어? 어제는 테스트가 성공했는데, 오늘은 실패하네?
+ CI 환경에서 문제
    - GitHub Actions는 항상 깨끗한 환경에서 시작해야 함
    - 로컬에서는 정상적으로 실행되는데, CI에서는 실패하는 상황이 발생할 수 있다
> 결국 테스트의 신뢰성 하락  

```java
@SpringBootTest
@ActiveProfiles("test")
class WhiskeyServiceTest {
    
    @Autowired
    WhiskeyRepository whiskeyRepository;
    
    @BeforeEach
    void setUp() {
        // 각 테스트 전에 필요한 데이터 직접 생성
        Whiskey whiskey = Whiskey.builder()
            .name("맥캘란 12년")
            .price(100000)
            .build();
        whiskeyRepository.save(whiskey);
    }
    
    @Test
    void 위스키_조회_테스트() {
        // 테스트 실행
    }
    
    @AfterEach
    void tearDown() {
        // 테스트 후 데이터 정리
        whiskeyRepository.deleteAll();
    }
}
```
이렇게 도커 볼륨에 의존하지 않고 테스트 코드에 직접 데이터를 생성하면 다음과 같은 장점이 생긴다

+ 테스트가 독립적 (다른 테스트에 영향을 받지않음)
+ 코드만 봐도 테스트 상황을 이해 가능 (아, 위스키를 추가하는 테스트구나!)
+ CI 환경에서도 동일하게 동작

SQL 스크립트를 작성하여 테스트용 더미 데이터를 추가해도 괜찮지만, 개인적으로 코드가 한눈에 들어오지 않아 좋아하지 않는 방법  

하나 주의할 사항이 있는데, docker-compose.test.yml에 설정한 MySQL과 Redis 포트에는 포트매핑을 잡아줘야 한다
```
ports:
  - "127.0.0.1:3307:3306"
```

+ 개발용 MySQL 컨테이너: 3306 포트 사용 중
+ 테스트용 MySQL 컨테이너: 3306 포트 사용하려고 함
> 포트 충돌! 한 포트에 두 개를 못 띄움!


<!-- CI를 도입하는 이유는 코드를 자주 브랜치에 푸시하여 기존 코드와 충돌이 발생하는지 빠르게 확인하고  
충돌이 없으면 자동으로 빌드와 테스트를 진행하여 코드의 품질을 유지하는 방법이다  

그런데 우리가 로컬에서 개발할 때는 MySQL과 Redis는 Docker로 띄우고 애플리케이션은 IntelliJ에서 Run Application으로 실행한다!  

Docker에 애플리케이션 빌드도 같이 추가해서 실행해도 되지만 너무 오랜 시간이 걸린다.. -->

+ 로컬 개발
    - Docker : MySQL, Redis
    - 애플리케이션 IntelliJ Run Application
+ CI 테스트
    - Docker : MySQL, Redis
    - 애플리케이션 : Gradle로 빌드/테스트

### 테스트 코드 실행하기
docker-compose.yml 설정이 끝났으면 이제 테스트 코드를 실행해볼 차례다  
아래 커맨드를 입력해보자  

```
./gradlew clean test
```