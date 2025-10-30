# IOC와 DI의 차이점과 스프링에서 구현하는 방법

## 전통적인 방식의 문제점

```java
public class UserService {
    private UserRepository repository = new UserRepository();
    
    public void registerUser(User user) {
        repository.save(user);
    }
}
```

위 방식의 문제점은 UserService가 UserRepository의 생성과 생명주기를 직접 제어한다는 것  
이렇게 되면 3가지 문제점이 발생한다  
1. 테스트가 어렵고
2. 코드 변경이 어렵고
3. 결합도가 높아진다

1번과 2번은 나중에 설명하고 우선 3번에 대해 설명해보자!  
코드를 보면 UserRepository는 UserService 안에서 생성된다.  
따라서 UserService의 실행이 끝나면 UserRepository도 같이 생명주기가 끝나게 된다.
그래서 코드의 결합도가 높아진다고 이야기한다.

## IoC(Inversion of Control)
IoC는 설계원칙이자 개념이다. `프로그램의 제어 흐름을 개발자가 아닌 프레임워크가 담당`하자는 의미로
- 기존 : 내가 필요한 객체를 내가 직접 생성하고 관리
    + 상단 코드의 UserRepository처럼
- IoC : 누군가(컨테이너)가 객체를 생성하고 관리해주고 나는 그것을 필요할때 받아서 사용  

IoC는 DI뿐만 아니라 여러 방식으로 구현될 수 있는데,
- Dependency Injection(의존성 주입)
- Service Locator Pattern
- Factory Pattern
- Template Method Pattern 
등이 있다.

## DI(Dependency Injection)
의존성 주입은 말 그대로 의존성(다른 객체)를 외부에서 주입받는 방식이다.

```java
public class UserService {
    private final UserRepository repository;
    
    // 생성자를 통해 의존성을 '주입' 받음
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
    
    public void registerUser(User user) {
        repository.save(user);
    }
}
```

|구분|IoC|DI|
|---|---|---|
|개념 수준|설계 원칙/패러다임|구현 기법/패턴|
|범위|넓음 (전체 제어 흐름)|좁음 (객체 간 의존성)|
|관계|상위 개념|IoC를 실현하는 방법 중 하나|

## 스프링에서의 구현
스프링의 핵심은 IoC Container고 이 컨테이너가 객체(빈)의 생성, 생명주기 관리, 의존성 연결을 담당한다  
DI는 세가지 방법이 있는데, 

- 생성자 주입(권장)
```java
@Service
public class UserService {
    private final UserRepository repository;
    
    @Autowired // 생성자가 하나면 생략 가능
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

- 세터 주입
```java
@Service
public class UserService {
    private UserRepository repository;
    
    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}
```

- 필드 주입(권장하지 않음)
```java
@Service
public class UserService {
    @Autowired
    private UserRepository repository;
}
```

### 생성자 주입을 권장하는 이유
생성자를 사용하여 주입하는 것을 권장하고 필드 주입을 권장하지 않는 이유는 무엇일까?  
필드 주입은 @Autowired 어노테이션을 사용하면 간편하게 의존성을 주입할 수 있는데, 왜 권장하지 않는 방법인걸까?

- final 키워드로 인한 불변성 보장
    ```java
    private final UserRepository repository;  // 한번 주입되면 변경 불가!
    ```
    + 멀티스레드 환경에서 안전하다
    + 의도치 않은 변경을 방지한다

- 의존성을 명확하게 확인이 가능하다
    ```java
    // 이 클래스는 3개의 의존성이 필요하구나!
    public UserService(UserRepository repo, 
                    EmailService email, 
                    LogService log) {
        // ...
    }
    ```
    + Lombok의 @RequiredArgsConstructor를 사용하면...

- 순환 참조 방지
    ```java
    @Service
    class A {
        public A(B b) { }  // A가 B를 의존
    }

    @Service
    class B {
        public B(A a) { }  // B가 A를 의존
    }
    ```
    + 생성자 주입이면 `애플리케이션 시작 시점`에서 순환 참조 에러가 발생
    + 필드 주입이면 실행 중 문제가 생길 가능성이 높다

### 순환 참조 방지가 왜 중요한데?
순환 참조란 A가 B를 필요로 하고 B가 A를 필요로 하는 상태를 말한다  

```java
@Service
public class AService {
    private final BService bService;
    
    public AService(BService bService) {
        this.bService = bService;
    }
}

@Service
public class BService {
    private final AService aService;
    
    public BService(AService aService) {
        this.aService = aService;
    }
}
```

그래서 왜 문제라는 걸까?
바로 해당 코드는 실행 자체가 불가능하다  

```bash
Spring Container : AService를 만들자!
Spring Container : AService 생성자에 BService가 필요하다!
Spring Container : 그럼 BService를 먼저 만들어야지

Spring Container : BService를 만들려고 보니..
Spring Container : BService 생성자에 AService가 필요하네..

무한반복..
```

그래서 생성자 주입으로 의존성을 주입하면 애플리케이션 시작이 실패하여 바로 문제를 수정할 수 있지만  
필드 주입은 실행 자체는 가능하다

```java
@Service
public class AService {
    @Autowired
    private BService bService;  // 시작은 됨
    
    public void doSomething() {
        bService.work();  // 💥 실행 중에 터짐!
    }
}
```

그리고 필드 주입은 4가지 문제점을 위반한다.  
1. 서로가 서로를 의존하므로 결합도가 높다
2. 책임이 명확하게 분리되지 않았다
3. 한 클래스를 수정하면 다른 클래스도 영향을 받는다
4. 가장 중요한 `단일 책임 원칙` 위반

### 스프링에서 구현하는 방법
필드 주입은 new 키워드를 사용하여 의존성을 주입한다  
그런데 생성자 주입은 new 키워드가 없는데 의존성이 주입된다고 한다  

결론부터 말하자면 컨테이너가 내부적으로 new 키워드를 사용하여 객체를 생성한다

```java
@Service
public class UserService {
    private final UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}

// new 키워드가 없다, 이것을 스프링 컨테이너가 아래와 같은 방법으로 사용한다

Class<?> clazz = Class.forName("com.example.UserRepository");
Constructor<?> constructor = clazz.getConstructor();
Object instance = constructor.newInstance();  // 리플렉션으로 생성!
```

그렇다면 왜 리플렉션 API를 사용하는걸까?  
스프링 컨테이너는 어떤 클래스를 빈으로 만들어야 하는지 모른다!

```java
// 스프링 입장: 이 중에 뭘 만들어야 하지?
@Service
public class UserService { }

@Service  
public class OrderService { }

@Service
public class PaymentService { }

// 사용자가 @Service를 추가하면 그것도 만들어야 함!
```

이것을 리플렉션 API를 사용하면 아래처럼 작업할 수 있다

```java
// ❌ 이렇게 할 수 없음 (컴파일 타임에 모르니까)
UserService service1 = new UserService();
OrderService service2 = new OrderService();
PaymentService service3 = new PaymentService();
// 새로운 @Service가 추가되면? 코드 수정 불가능!

// ✅ 리플렉션 사용
List<String> serviceClasses = scanForServices();  // @Service 찾기

for (String className : serviceClasses) {
    Class<?> clazz = Class.forName(className);  // 런타임에 로딩
    Object bean = clazz.getDeclaredConstructor().newInstance();  // 생성!
    beans.put(className, bean);
}
```

비유로 예를 들면  
new 키워드는 직접 주문  

```bash
나: "김치찌개 1개 주세요" (코드에 명시)
주방: "네, 김치찌개 만들겠습니다"
```

리플렉션은 메뉴 번호로 주문  

```bash
나: "7번 메뉴 주세요" (문자열로 지정)
주방: (메뉴판 확인) "7번이 김치찌개네요? 만들겠습니다"
```