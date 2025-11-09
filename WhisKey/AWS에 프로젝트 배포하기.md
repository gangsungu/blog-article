## AWS에 프로젝트 배포하기

### 1단계 : Dockerfile + GitHub Action 빌드 자동화
Dockerfile이 뭐야?  

+ Docker 이미지
    - 애플리케이션 + 실행 환경(애플리케이션을 실행하는데 필요한 모든 파일)이 패키징된 파일
    - 이것을 실행하면 컨테이너가 됨
+ Dockerfile
    - Docker 이미지를 어떻게 만들지 적어둔 레시피

비유하자면  
Dockerfile = 요리 레시피
Docker Image = 완성된 요리
Docker Container = 완성된 요리를 먹기  

Dockerfile을 사용하지 않고 프로젝트를 AWS에 배포하면  
1. EC2에 Java를 설치
2. jar 파일 업로드
3. java -jar app.jar 실행
4. 스프링부트가 아닌 옛날 스프링인 경우, Tomcat도 설치  

하지만 Dockerfile을 사용하면  
1. Dockerfile을 작성
2. Docker 이미지 생성
3. docker run으로 애플리케이션 실행  

자, 이제 Dockerfile을 만들어보자  
프로젝트 루트에서 빌드 결과물이 어디에 위치하는지 확인해야 한다  
멀티 모듈의 경우, 메인이 되는 모듈에서 확인하자  

```
./gradlew clean build
```

커맨드를 입력했을 때, jar 파일이 생성되는 위치가 메인 모듈이다  
Whiskey 프로젝트를 예로 들면, 메인 모듈인 module-api에 jar가 위치한다  

```
module-api/build/libs를 보면

module-api-0.0.1-SNAPSHOT.jar
# 실행가능한 jar (executable jar)
# 의존성이 다 포함된 fat jar

module-api-0.0.1-SNAPSHOT-plain.jar
# 순수 클래스 파일만 첨부
# 의존성 없음
# 라이브러리로 사용할 때, 필요한 jar
# 실행 X (의존성이 없으니까!)
```