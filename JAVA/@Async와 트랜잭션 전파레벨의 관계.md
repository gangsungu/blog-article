## @Async와 트랜잭션 전파레벨

아래와 같은 코드가 있다  
@Async와 트랜잭션 전파레벨을 하나씩 풀어보자

```java
@GetMapping("/whiskey/{id}")
@ActivityLog(type = ActivityType.VIEW, target = TargetType.WHISKEY)
@Operation(summary = "위스키 조회", description = "위스키 ID로 위스키 정보를 조회합니다.")
public ApiResponse<WhiskeyResponse> get(@Parameter(description = "위스키 ID") @PathVariable("id") @TargetId Long id) {
    WhiskeyInfo whiskeyInfo = whiskeyService.findById(id);
    WhiskeyResponse response = WhiskeyResponse.from(whiskeyInfo);
    return ApiResponse.success("위스키를 조회하였습니다.", response);
}

@Around("@annotation(activityLog)")
public Object activityLogPointcut(ProceedingJoinPoint joinPoint, ActivityLog activityLog) throws Throwable {
    Object result = joinPoint.proceed();

    try {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Object[] args = joinPoint.getArgs();

        Long targetId = getTargetId(signature, args);
        Long memberId = getMemberId();

        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        ActivityLogDto logDto = ActivityLogDto.of(
            memberId,
            activityLog.type(),
            activityLog.target(),
            targetId,
            request.getRemoteAddr()
        );

        activityLogService.saveAsync(logDto);
        log.info("활동 로그 처리 완료 - 타입: {}, 대상: {}, ID: {}", activityLog.type(), activityLog.target(), targetId);
    }
    catch (Exception e) {
        log.error("활동 로그 AOP 처리 실패", e);
    }

    return result;
}

@Async("logExecutor")
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAsync(ActivityLogDto activityLogDto) {
    try {
        log.info("활동 로그 저장 시작");

        ActivityLog logInfo = ActivityLog.builder()
            .memberId(activityLogDto.memberId())
            .activityType(activityLogDto.activityType())
            .targetType(activityLogDto.targetType())
            .targetId(activityLogDto.targetId())
            .ipAddress(activityLogDto.ipAddress())
            .build();

        activityLogRepository.save(logInfo);
    }
    catch (Exception e) {
        log.error("활동 로그 저장 실패", e);
    }
}
```

### 동작원리
@Async를 사용하면 어노테이션이 붙어있는 메서드가 별도의 스레드에서 비동기로 실행된다  

```java
@Service
public class MyService {
    @Async
    public void asyncMethod() {
        // 이 코드는 새로운 스레드에서 실행됨
    }
}
```

트랜잭션의 전파수준은 `트랜잭션이 이미 진행 중일때, 새로운 트랜잭션의 경계를 만나면 해야할 동작을 정의`한다

+ REQUIRED(기본값) : 진행 중인 트랜잭션이 있으면 참여하고 없으면 새로 생성
+ REQUIRED_NEW : 항상 새로운 트랜잭션을 만들고 기존 트랜잭션이 있으면 잠시 중단