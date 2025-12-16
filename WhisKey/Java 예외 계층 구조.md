## Java의 예외 계층 구조를 알아보자

```
Throwable
├── Error (시스템 레벨 에러 - 개발자가 처리 불가)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...
│
└── Exception (개발자가 처리 가능한 예외)
    ├── Checked Exception (컴파일러가 강제)
    │   ├── IOException
    │   ├── SQLException
    │   ├── ClassNotFoundException
    │   └── ...
    │
    └── RuntimeException (Unchecked Exception)
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── IllegalStateException
        └── 우리의 BusinessException ✅
```

### Checked vs Unchecked Exception
+ Checked Exception (Exception 직접 상속)
    - RuntimeException을 상속하지 않는 클래스
    - 컴파일러가 `try-catch` 또는 `throws` 강제 (컴파일 시점에 컴파일러가 확인한다)
    - 반드시 에러 처리를 해야하는 특징(try/catch or throw)

    ```java
    try {
        File file = new File("example.txt");
        Scanner scanner = new Scanner(file);
        while (scanner.hasNextLine()) {
            String line = scanner.nextLine();
            System.out.println("line = " + line);
        }
        scanner.close();
    } 
    catch (FileNotFoundException e) {
        System.out.println("An error occurred while reading the file: " + e.getMessage());
    }

    // example.txt를 읽어야하는데 example.txt라는 파일이 없으면 FileNotFoundException을 던진다
    // An error occurred while reading the file: example.txt (No such file or directory)
    ```
+ Unchecked Exception (Runtime Exception)
    - RuntimeException을 상속하는 클래스
    - 에러 처리를 강제하지 않는다
    - 프로그래밍 오류나 비즈니스 규칙 위반에 사용 (런타임 단계에서 확인 가능)

    ```java
    authService.login(email, password);  // ✅ try-catch 선택사항
    // GlobalExceptionHandler가 알아서 잡아줌
    ```

### 왜 RuntimeException을 선택했는가

+ Spring은 기본적으로 RuntimeException만 자동으로 롤백
    ```java
    @Transactional
    public void createOrder() {
        // ...
        throw new BusinessException(...);  // RuntimeException → 자동 롤백 ✅
    }

    @Transactional
    public void createOrder() throws CheckedException {
        // ...
        throw new CheckedException(...);  // Checked Exception → 롤백 안 됨 ❌
    }
    ```

+ 코드 가독성
    - 비즈니스 로직에 `throws`가 붙으면 이쁘지 않다
    ```java
    // RuntimeException (현재)
    public JwtResponse login(String email, String password) {
        Member member = authenticateMember(email, password);
        // ...
    }

    // Checked Exception이면?
    public JwtResponse login(String email, String password) 
        throws BusinessException {  // 모든 메서드에 throws 붙여야 함
        Member member = authenticateMember(email, password);
        // ...
    }
    ```

+ GlobalExceptionHandler의 활용
    - RuntimeException은 try-catch를 명시적으로 선언하지 않아도 되므로 GlobalExceptionHandler에서 통합 처리하기 편하다