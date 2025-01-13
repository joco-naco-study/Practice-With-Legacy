# 기존의 에러 처리 방식

```java
@RestControllerAdvice  
public class GlobalExceptionHandler {  

    @ExceptionHandler(BaseException.class)  
    public ResponseEntity<ErrorResponse> handleBaseException(BaseException e) {  
        return ResponseEntity  
                .status(e.getErrorCode().getStatus())  
                .body(ErrorResponse.of(e.getErrorCode()));  
    }  

    @ExceptionHandler(RuntimeException.class)  
    public ResponseEntity<ErrorResponse> handleRuntimeException(RuntimeException e) {  
        return ResponseEntity  
                .status(GlobalErrorCode.INTERNAL_SERVER_ERROR.getStatus())  
                .body(ErrorResponse.of(GlobalErrorCode.INTERNAL_SERVER_ERROR));  
    }  

    @ExceptionHandler(Exception.class)  
    public ResponseEntity<ErrorResponse> handleException(Exception e) {  
        return ResponseEntity  
                .status(GlobalErrorCode.INTERNAL_SERVER_ERROR.getStatus())  
                .body(ErrorResponse.of(GlobalErrorCode.INTERNAL_SERVER_ERROR));  
    }  
}
```

`GlobalExceptionHandler` 클래스는 Spring Boot의 전역 예외 처리를 담당하는 클래스입니다. `@RestControllerAdvice`와 `@ExceptionHandler`를 활용하여 애플리케이션 전반에서 발생하는 예외를 한 곳에서 처리하도록 구성되어 있습니다.


# `@RestControllerAdvice`

- `@RestControllerAdvice`는 Spring MVC에서 전역 예외 처리를 위한 어노테이션입니다.
- 특정 컨트롤러에 국한되지 않고, 애플리케이션 전체에서 발생하는 예외를 잡아 처리합니다.

# **예외 처리 메서드**

##  `handleBaseException(BaseException e)`

- **동작**:
    - `BaseException` 타입의 예외가 발생했을 때 호출됩니다.
    - 발생한 예외 객체 `e`에서 `ErrorCode`를 가져와 HTTP 상태 코드와 에러 응답 객체를 생성합니다.
- **구현 의도**:
    - `BaseException`은 커스텀 예외 클래스로, 각 도메인에서 정의한 특정 에러 코드와 메시지를 담고 있습니다.
    - 이를 통해 에러 응답을 클라이언트에 전달합니다.
- **ResponseEntity 반환**:
    - `status(e.getErrorCode().getStatus())`: 예외의 상태 코드를 HTTP 응답 상태로 설정합니다.
    - `body(ErrorResponse.of(e.getErrorCode()))`: 예외 코드와 메시지를 포함한 `ErrorResponse` 객체를 응답 본문에 담습니다.

##  `handleRuntimeException(RuntimeException e)`

- **동작**:
    - `RuntimeException` 타입의 예외가 발생했을 때 호출됩니다.
    - `GlobalErrorCode.INTERNAL_SERVER_ERROR`에 정의된 상태 코드와 에러 응답 객체를 반환합니다.
- **구현 의도**:
    - 예상치 못한 런타임 예외를 처리하며, 클라이언트에게 500 상태 코드와 일반적인 에러 메시지를 제공합니다.
- **ResponseEntity 반환**:
    - `status(GlobalErrorCode.INTERNAL_SERVER_ERROR.getStatus())`: HTTP 상태 코드를 500으로 설정합니다.
    - `body(ErrorResponse.of(GlobalErrorCode.INTERNAL_SERVER_ERROR))`: 500 에러에 대한 기본 응답 객체를 담습니다.

## `handleException(Exception e)`

- **동작**:
    - 모든 `Exception` 타입의 예외를 처리합니다.
    - 위의 두 메서드에 해당되지 않는 예외가 발생했을 때 실행됩니다.
- **구현 의도**:
    - 런타임 이외의 예외, 예상치 못한 예외를 처리하며 500 상태 코드와 기본적인 에러 메시지를 제공합니다.
- **ResponseEntity 반환**:
    - `status(GlobalErrorCode.INTERNAL_SERVER_ERROR.getStatus())`: HTTP 상태 코드를 500으로 설정합니다.
    - `body(ErrorResponse.of(GlobalErrorCode.INTERNAL_SERVER_ERROR))`: 기본 에러 메시지를 포함한 응답 객체를 반환합니다.

```java
@Getter  
@AllArgsConstructor  
public enum GlobalErrorCode implements ErrorCode {  
    INVALID_INPUT_VALUE(HttpStatus.BAD_REQUEST, "잘못된 입력 값입니다."),  
    RESOURCE_NOT_FOUND(HttpStatus.NOT_FOUND, "리소스를 찾을 수 없습니다."),  
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "서버 내부 오류가 발생했습니다."),  
    UNAUTHORIZED(HttpStatus.UNAUTHORIZED, "권한이 없습니다."),  
    FORBIDDEN(HttpStatus.FORBIDDEN, "접근이 금지되었습니다.");  

    public static final String PREFIX = "[GLOBAL ERROR] ";  

    private final HttpStatus status;  
    private final String rawMessage;  

    @Override  
    public String getMessage() {  
        return PREFIX + rawMessage;  
    }  

    @Override  
    public int getStatusValue() {  
        return status.value();  
    }  
}
```

`GlobalErrorCode`는 애플리케이션에서 전역적으로 사용될 에러 코드와 관련된 정보를 관리하기 위해 작성된 **열거형(enum)** 클래스입니다.  전역 에러 코드를 정의하고, 각 에러에 관련된 정보를 제공하는 클래스입니다.
HTTP 상태 코드와 에러 메시지를 포함하며, 이를 활용해 예외 처리 응답을 생성합니다.

#  `implements ErrorCode`

- `ErrorCode`라는 인터페이스를 구현합니다.
- 이 인터페이스는 에러 코드와 관련된 동작(`getMessage()`, `getStatusValue()`)을 정의하고 강제합니다.
- 덕분에 일관성 있는 에러 코드 관리를 지원합니다.
# ENUM

각 상수는 특정 에러에 대응하며, 생성자에 의해 다음 두 가지 정보를 갖습니다:

1. **`HttpStatus status`**: HTTP 상태 코드 (예: `400 BAD_REQUEST`, `404 NOT_FOUND` 등).
2. **`String rawMessage`**: 사용자에게 표시할 에러 메시지

```java
@Getter  
public abstract class BaseException extends RuntimeException {  

    private final ErrorCode errorCode;  

    public BaseException(ErrorCode errorCode) {  
        super(errorCode.getMessage());  
        this.errorCode = errorCode;  
    }  
}
```

`BaseException` 클래스는 애플리케이션에서 공통으로 사용할 **커스텀 예외의 기반 클래스**입니다. 이를 통해 에러 코드를 포함하는 일관된 예외 처리 체계를 구현할 수 있습니다. **기반 예외 클래스**로, 애플리케이션에서 발생하는 커스텀 예외의 상위 클래스 역할을 합니다. 모든 커스텀 예외 클래스는 이 클래스를 상속받아 구현됩니다.`ErrorCode` 인터페이스를 활용하여 예외와 관련된 정보를 제공합니다.

# `extends RuntimeException`

 `RuntimeException`을 상속하여 **언체크 예외**로 동작합니다. 언체크 예외는 `try-catch`로 강제 처리하지 않아도 됩니다.

# 생성자

1. **`super(errorCode.getMessage())`**
    - 부모 클래스(`RuntimeException`)의 생성자를 호출합니다.
    - `RuntimeException`의 `message` 필드에 `ErrorCode`의 메시지를 전달하여 예외 메시지를 설정합니다.
    - 결과적으로, `BaseException`은 예외 메시지를 포함한 일반적인 예외 객체로 동작할 수 있습니다.
2. **`this.errorCode = errorCode;`**
    - 전달받은 `ErrorCode` 객체를 `errorCode` 필드에 저장합니다.
    - 이를 통해 예외 객체가 발생한 에러의 상태 코드와 메시지를 포함할 수 있습니다.



```java
public interface ErrorCode {  
    HttpStatus getStatus();  
    String getMessage();  
    int getStatusValue();  
}
```

`ErrorCode`는 애플리케이션에서 예외와 관련된 정보를 제공하기 위한 **인터페이스**입니다. 이 인터페이스는 상태 코드와 메시지에 대한 **구조**를 정의하여, 모든 구현체가 일관된 방식으로 에러 코드를 관리할 수 있도록 돕습니다.

```java
@Getter  
@AllArgsConstructor  
public class ErrorResponse {  
    private final int status;  
    private final String message;  

    public static ErrorResponse of(ErrorCode errorCode) {  
        return new ErrorResponse(errorCode.getStatusValue(), errorCode.getMessage());  
    }  
}
```

`ErrorResponse` 클래스는 클라이언트에 반환될 에러 응답 데이터를 캡슐화한 **DTO** 클래스입니다. 예외가 발생했을 때, 서버는 이 클래스를 사용해 HTTP 응답 본문(body)에 에러 상태 코드와 메시지를 포함한 구조화된 데이터를 전달합니다. 클라이언트가 이해할 수 있는 형태로 **에러 상태 코드와 메시지**를 포함하는 응답 객체를 생성합니다. 예외 처리 과정에서 사용되며, `ErrorCode` 객체를 기반으로 응답을 생성합니다.

## 사용 예시

### 도메인에서 CustomExcetpion 생성
```java
public class GroupException extends BaseException {  
    public GroupException(GroupErrorCode errorCode) {  
        super(errorCode);  
    }  
}
```

### 도메인에서 CustomErrorCode 작성
```java
public enum GroupErrorCode implements ErrorCode {  
    // 그룹 조회  
    GROUP_NOT_FOUND(HttpStatus.NOT_FOUND, "그룹을 찾을 수 없습니다.");

    public static final String PREFIX = "[GROUP ERROR] ";  

    private final HttpStatus status;  
    private final String rawMessage;  

    @Override  
    public String getMessage() {  
        return PREFIX + rawMessage;  
    }  

    @Override  
    public int getStatusValue() {  
        return status.value();  
    }  
}
```

### 도메인 메서드에서 예외 처리
```java
private void validateLength(String code) {  
    if (code.length() != CODE_LENGTH) {  
        throw new GroupException(GroupErrorCode.INVALID_INVITE_CODE_LENGTH);  
    }  
}
