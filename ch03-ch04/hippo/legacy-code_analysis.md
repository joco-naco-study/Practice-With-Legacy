## 프로젝트에 사용한 예외 처리 방식

```java
@Getter
public class FluffyException extends RuntimeException {

    private final ErrorCode errorCode;

    public FluffyException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(FluffyException.class)
    protected ResponseEntity<ErrorResponse> handle(FluffyException e) {
        log.error("FluffyException = {}", e.getMessage());
        return createErrorResponseEntity(e.getErrorCode());
    }

    @ExceptionHandler(GlobalException.class)
    protected ResponseEntity<ErrorResponse> handle(GlobalException e) {
        log.error("GlobalException = {}", e.getMessage());
        return createErrorResponseEntity(e.getErrorCode());
    }

    @ExceptionHandler(AuthException.class)
    public ResponseEntity<ErrorResponse> handleBadCredentialsException(AuthException e) {
        log.error("AuthException = {}", e.getMessage());
        return createErrorResponseEntity(e.getErrorCode());
    }

    private ResponseEntity<ErrorResponse> createErrorResponseEntity(ErrorCode code) {
        return ResponseEntity.status(code.getHttpStatus()).body(ErrorResponse.of(code));
    }
}
```

```java
public interface ErrorCode {

    HttpStatus getHttpStatus();

    String getCode();

    String getMessage();
}
```

```java
@Getter
public class ErrorResponse {

    private final String code;
    private final String message;

    private ErrorResponse(ErrorCode code) {
        this.code = code.getCode();
        this.message = code.getMessage();
    }

    public static ErrorResponse of(ErrorCode code) {
        return new ErrorResponse(code);
    }
}
```

```java
@Getter
@AllArgsConstructor
public enum GlobalErrorCode implements ErrorCode {

    USER_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-001", "유저를 찾을 수 없습니다."),
    POST_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-002", "게시글을 찾을 수 없습니다."),
    COMMENT_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-003", "댓글을 찾을 수 없습니다."),
    REPLY_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-004", "답글을 찾을 수 없습니다."),
    TAG_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-006", "태그를 찾을 수 없습니다."),
    PERMISSION_NOT_EDIT(HttpStatus.FORBIDDEN, "GE-007", "편집 권한이 없습니다."),
    PERMISSION_NOT_ACCESS(HttpStatus.FORBIDDEN, "GE-008", "접근 권한이 없습니다."),
    SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "GE-009", "서버 에러"),
    BAD_REQUEST(HttpStatus.BAD_REQUEST, "GE-010", "잘못된 요청"),
    ROLE_NAME_NOT_FOUND(HttpStatus.NOT_FOUND, "GE-011", "유저 권한 명을 찾을 수 없습니다"),
    FAILED_TO_SEND_EMAIL(HttpStatus.BAD_REQUEST, "GE-012", "인증 메일 전송 실패");

    private final HttpStatus httpStatus;
    private final String code;
    private final String message;
}
```

---

## 그렇다면 오류를 명시적으로 들어냈는가?


명시적 방식을 사용했다고 생각되는 코드 => Optional을 이용해서 널 값이 가능한 반환 유형 이용

```java
    // 비밀번호 변경
    @Transactional
    public void changePassword(PasswordChangeRequest passwordChangeRequest) {
        String email = passwordChangeRequest.getEmail();
        String password = passwordChangeRequest.getPassword();

        User findUser = userRepository.findByEmail(email).orElseThrow(UserNotFound::new);

        findUser.setPassword(passwordEncoder.encode(password));

        userRepository.save(findUser);
    }
```

암시적 방식이라고 생각되는 코드 => 비검사 예외를 이용..

```java
    @Transactional
    public CheckDuplicationResponse checkExistsEmail(
        String email) {
        boolean isExists = userRepository.findByEmail(email).isPresent();
        if (isExists) {
            throw new AlreadyExistsEmail();
        }
        return CheckDuplicationResponse.builder()
                .isExists(false)
                .build();
    }
```

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // 사용자 이름 중복 여부
    boolean existsByNickname(String nickname);

    boolean existsUserByEmail(String email);

    Optional<User> findByEmail(String email);

    Optional<User> findByNickname(String nickname);

    List<User> findByNicknameContaining(String keyword);

    Page<User> findAll(Pageable pageable);

    @Query(
        value = "SELECT new com.fluffytime.domain.admin.dto.DailyUserCount(u.registrationAt, COUNT(u)) " +
            "FROM User u " +
            "WHERE u.registrationAt BETWEEN :startDate AND :endDate " +
            "GROUP BY u.registrationAt " +
            "ORDER BY u.registrationAt ASC"
    )
    List<DailyCount> findUserCountByRegistrationAtBetween(
        LocalDateTime startDate,
        LocalDateTime endDate
    );
}
```

---
## 헷갈렸던 점

책에서는 명시적 방식으로 검사 예외를 추천했다.

그런데 @RestControllerAdvice를 이용한 전역적 예외 처리 방식이 명시적인가에 대한 의문이 들었다!

그래서 GPT 한테 물어보았다!


## 명시적 방식을 사용하라!!


### GPT 검색

`@RestControllerAdvice`는 **명시적인 오류 처리**의 관점에서 봤을 때, **부분적으로 명시적**인 접근 방식이라고 할 수 있습니다. 

하지만 "완전히 명시적"이라고 보기는 어렵습니다. 이를 이해하려면 몇 가지 측면에서 살펴볼 필요가 있습니다.

---

### 1. **RestControllerAdvice의 동작 방식**
`@RestControllerAdvice`는 Spring 애플리케이션에서 **전역적으로 예외를 처리**하기 위한 기능입니다.  
다음과 같은 특징이 있습니다:
- 특정 예외에 대해 `@ExceptionHandler`를 통해 처리 방법을 정의.
- 예외가 발생했을 때 이를 캡처하고, 클라이언트에 적절한 HTTP 응답을 반환.
- 호출자가 예외를 직접 다루지 않아도, 전역적인 예외 처리기를 통해 간접적으로 처리.

#### 예제:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<String> handleCustomException(CustomException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body("Error: " + ex.getMessage());
    }
}
```

이 방식은 호출자에게 예외를 명시적으로 전달하기보다는, **발생한 예외를 숨기고 적절한 응답을 반환**하는 방식입니다.

---

### 2. **명시적인 접근과 비교**
`@RestControllerAdvice`는 다음과 같은 방식으로 명시적 접근과 차이가 있습니다:

#### 명시적 방식의 핵심
- 호출자가 **오류 발생 가능성**을 인지하고, 이를 처리하도록 강제.
- 검사 예외(`throws`)나 **결과 객체 반환**을 통해 오류를 전달.

#### RestControllerAdvice의 접근
- 호출자(클라이언트)가 오류 발생 가능성을 직접적으로 알지 못함.
- 예외는 **컨트롤러 내부에서만 처리되고**, 호출자는 HTTP 상태 코드와 응답 메시지로 오류를 간접적으로 인지.

---

### 3. **RestControllerAdvice의 명시성 평가**
#### 장점:
1. **전역적이고 일관된 예외 처리**:
    - 다양한 예외를 통합적으로 처리할 수 있어 코드 중복 감소.
    - HTTP 응답으로 오류 상태를 클라이언트에 명확히 전달.

2. **응답에 초점**:
    - 예외 처리 로직이 클라이언트와 상호작용(REST API 설계)에 맞춰 설계.
    - 클라이언트는 HTTP 상태 코드를 통해 오류를 간접적으로 이해.

#### 단점:
1. **호출자 입장에서의 불명확성**:
    - 호출자는 해당 메서드에서 어떤 예외가 발생할지 명시적으로 알 수 없음.
    - 메서드의 문서화 없이도 예외 처리 로직이 숨겨질 수 있음.

2. **복구 가능성을 다루기 어려움**:
    - 호출자가 오류를 복구하고 싶어도, `@RestControllerAdvice`는 주로 HTTP 응답 수준에서 처리하므로 복구 논리를 직접 포함하지 않음.

---

### 4. **결론: RestControllerAdvice는 명시적인가?**
- **부분적으로 명시적**:
    - 예외를 특정 HTTP 상태 코드와 메시지로 매핑하는 점에서는 클라이언트에게 명확한 응답을 제공합니다.
- **완전히 명시적이지 않음**:
    - 호출자가 메서드 호출 단계에서 오류 발생 가능성과 이를 처리하는 방식을 명시적으로 알 수는 없습니다.

---

### 5. **언제 RestControllerAdvice를 사용할까?**
`@RestControllerAdvice`는 다음과 같은 상황에서 유용합니다:
- 클라이언트와 상호작용하는 REST API 설계에서, **HTTP 상태 코드와 메시지로 예외를 표준화**하려는 경우.
- 호출자(클라이언트)에게 예외를 명시적으로 노출할 필요가 없고, **일관된 응답 구조**만 제공하면 되는 경우.
- 복구 가능한 오류보다, 단순히 "예외 발생 시 적절한 응답을 반환"하려는 경우.

---

### 추가 제안
만약 "완전히 명시적인 방식"을 원한다면, 아래와 같은 방식을 고려해볼 수 있습니다:
1. **결과 객체 반환**:
    - 예외를 HTTP 응답 수준에서 숨기지 않고, 호출자가 처리할 수 있도록 설계.
2. **검사 예외를 활용**:
    - 클라이언트 코드가 예외 처리에 대해 명확하게 준비하도록 강제.

RestControllerAdvice는 이런 방식을 보완하며, 특히 **HTTP 레이어에서의 오류 처리**에 중점을 둔 도구라고 보면 됩니다.