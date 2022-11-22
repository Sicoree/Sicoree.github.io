---
title: Exception Handling 
author: Sicoree EE
date: 2022-11-15 15:21:00 +0900
categories: [Spring Boot]
tags: [Spring Boot, Refactoring]

---

# Exception Handling

Spring에서 제공하는 `@RestControllerAdvice`, `@ExceptionHandler` 를 활용하여 API 예외처리를 하였다.

그 후 다음과 같은 일관된 ErrorResponse 반환을 목표 하였다.

```json
{
  "status": 400,
  "code": "U001",
  "message": "존재하지 않는 유저입니다."
}
```

```json
{
  "status": 400,
  "code": "W001",
  "message": "이미 신고한 유저입니다."
}
```

도메인들을 공통적으로 처리해야 하기 때문에 global 패키지 구조로 작성하였다.

![](C:\Users\multicampus\AppData\Roaming\marktext\images\2022-11-15-15-36-59-image.png)

# ErrorResponse

`ErrorResponse`는 다양한 예외 처리에 대해 대응하기 위하여, 기본적인 생성자 대신 입력 매개변수에 따라 유연하게 `ErrorResponse` 객체를 반환할 수 있는 정적 팩터리 메서드(Static Factory Method)를 활용 하였다. 

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ErrorResponse {
    private int status;
    private String code;
    private String message;

    private ErrorResponse(final ErrorCode code) {
        this.message = code.getMessage();
        this.status = code.getStatus();
        this.code = code.getCode();
    }

    public static ErrorResponse of(final ErrorCode code) {
        return new ErrorResponse(code);
    }
}
```

`of`라는 정적 팩터리 매서드들은 여러가지 상황에 대응할 수 있다. 대표적으로는

+ javax.vaidation에서 제공하는 `@Valid`나 Spring Boot에서 제공하는 `@Validated`를 통해 검증 처리 시 binding error가 발생하는 경우

+ 일반적인 예외 핸들링

+ 데이터 유효성 검사 실패시 발생하는 예외에서 실패 정보를 담고 있는 `ConstraintViolation`를 가져오는 경우

# ErrorCode 정의

```java
/**
 * ErrorCode Convention
 * - 도메인 별로 나누어 관리
 * - [주체_이유] 형태로 생성
 * - 코드는 도메인명 앞에서부터 1~2글자로 사용
 * - 메시지는 "~~다."로 마무리
 */

@Getter
@AllArgsConstructor
public enum ErrorCode {
    // Global
    INPUT_VALUE_INVALID(400, "G001", "유효하지 않은 입력입니다."),
    INPUT_TYPE_INVALID(400, "G002", "입력 타입이 유효하지 않습니다."),
    TIME_FORMAT_INVALID(400, "G003", "날짜, 시간 타입 형식이 유효하지 않습니다."),
    JOB_NOT_FOUND(400,"G004", "존재하지 않는 직업입니다."),

    // User
    USER_NOT_FOUND(400, "U001", "존재하지 않는 유저입니다."),
    USERNAME_ALREADY_EXIST(400, "U002", "이미 존재하는 사용자 이름입니다."),
    AUTHENTICATION_FAIL(401, "U003", "로그인이 필요한 화면입니다."),
    AUTHORITY_INVALID(403, "U004", "권한이 없습니다."),
    ACCOUNT_MISMATCH(401, "U005", "계정 정보가 일치하지 않습니다."),

    // 중략 ...

    ;

    private final int status; // <- final 안달아 줘도 되는지 체크 필요
    private final String code;
    private final String message;
}
```

Enum 타입으로 위와 같이 한 곳에서 에러 코드를 관리하였다. 이는 에러 코드가 도메인 전체적으로 흩엊있을 경우, 코드 및 메세지의 중복이 발생하기 때문에 이를 해결 하기위한 가장 효율적인 방법이다.

# GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler
    protected ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        final ErrorCode errorCode = e.getErrorCode();
        final ErrorResponse response = ErrorResponse.of(errorCode);
        return new ResponseEntity<>(response, HttpStatus.valueOf(errorCode.getStatus()));
    }

    // @Valid, @Validated 에서 binding error 발생 시 (@RequestBody)
    @ExceptionHandler
    protected ResponseEntity<ErrorResponse> handleBindException(BindException e) {
        final ErrorResponse response = ErrorResponse.of(INVALID_INPUT_VALUE, e.getBindingResult());
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }
}
```

`@RestControllerAdvice`, `@ExceptionHandler`을 활용하여 모든 예외를 한 곳에서 처리할 수 있다.

`@ControllerAdvice`는 `@ExceptionHandler`, `@ModelAttribute`, `@InitBinder`가 적용된 메소드들을 AOP를 적용해 컨트롤러 단에 적용하기 위해 고안된 애너테이션이다.

# Business Exception

`Runtimeexception`을 상속받아 최상위에 `BusinessException`을 정의해두면 아래와 같이 구체적인 Exeption에 대해 예외 처리를 통일감 있게 할 수 있다.

```java
@Getter
public class BusinessException extends RuntimeException {
    private ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

이 `BusinessException`을 상속받은 세부적인 예외들은 도메인 내에서 예외가 발생할만한 부분마다 디테일하게 처리할 수 있다는 장점이 있다. 다만, 해당 예외 케이스들을 직접 한땀한땀 정의해야 하기 때문에 클래스가 많아질 수 있다는 점이 단점이라 할 수 있다.

다음은 이미 존재하는 entity에 대해 발생하는 예외를 처리하기 위한 클래스를 정의한 코드이다. 다른 디테일한 예외 클래스들도 이와 같은 형식으로 정의하면 된다.

```java
public class EntityAlreadyExistException extends BusinessException {

    public EntityAlreadyExistException(ErrorCode errorCode){
        super(errorCode);
    }
}
```

결론적으론 서버 내부 코드에서 예외가 발생할 여지가 있다면 Exception을 발생시키고 위처럼 미리 정의한 예외 핸들링을 탈 수 있도록 설계하는 방식이다.

# API Response

응답결과 또한 일관된 형식으로 통일 하고자 한다.

```json
{
  "status": 200,
  "code": "U004",
  "message": "회원 프로필을 수정하였습니다.",
  "data": {
    "userSeq": 38,
    "email": "tester@test.com",
    "nickname": "아르마딜로",
    "job": "공무원",
    "imgUrl": "z/data/user/0/com.d205.sdutyplus/cache/temp_file_20221111_042718.jpg",
    "fcmToken": null
  }
}
```

```json
{
  "status": 200,
  "code": "W001",
  "message": "신고가 완료되었습니다.",
  "data": true
}
```

위의 Error Code, Error Response를 정의한것과 비슷하게 Result Code, Result Response를 정의하면 된다.

# Result Response

```java
@ApiModel(description = "결과 응답 데이터")
@Getter
@Data
public class ResultResponse {

    @ApiModelProperty(value = "Http 상태 코드")
    private int status;
    @ApiModelProperty(value = "Business 상태 코드")
    private String code;
    @ApiModelProperty(value = "응답 메세지")
    private String message;
    @ApiModelProperty(value = "응답 데이터")
    private Object data;

    public ResultResponse (ResultCode resultCode, Object data) {
        this.status = resultCode.getStatus();
        this.code = resultCode.getCode();
        this.message = resultCode.getMessage();
        this.data = data;
    }

    // 전송할 데이터가 있는 경우
    public static ResultResponse of(ResultCode resultCode, Object data) {
        return new ResultResponse (resultCode, data);
    }

    // 전송할 데이터가 없는 경우
    public static ResultResponse of(ResultCode resultCode) {
        return new ResultResponse (resultCode, "");
    }
}
```

# Result Code

```java
/**
 * ResultCode Convention
 * - 도메인 별로 나누어 관리
 * - [동사_목적어_SUCCESS] 형태로 생성
 * - 코드는 도메인명 앞에서부터 1~2글자로 사용
 * - 메시지는 "~~다."로 마무리
 */

@Getter
@AllArgsConstructor
public enum ResultCode {
    // User
    LOGIN_SUCCESS(200, "U001", "로그인에 성공하였습니다."),
    GET_USERPROFILE_SUCCESS(200, "U002", "회원 프로필을 조회하였습니다."),
    UPLOAD_USER_IMAGE_SUCCESS(200, "U003", "회원 이미지를 등록하였습니다."),
    EDIT_PROFILE_SUCCESS(200, "U004", "회원 프로필을 수정하였습니다."),
    CHECK_NICKNAME_GOOD(200, "U005", "사용가능한 nickname 입니다."),
    CHECK_NICKNAME_BAD(200, "U006", "사용불가능한 nickname 입니다."),
    LOGIN_FAIL(200, "U007", "로그인에 실패하였습니다."),
    SAVE_PROFILE_SUCCESS(200, "U008", "회원 프로필을 저장하였습니다."),
    DELETE_SUCCESS(200, "U009", "회원 탈퇴에 성공하였습니다."),
    DELETE_FAIL(200, "U010", "회원 탈퇴에 실패하였습니다."),

    // Task
    CREATE_TASK_SUCCESS(200, "T001", "테스크가 생성되었습니다."),
    UPDATE_TASK_SUCCESS(200, "T002", "테스크가 수정되었습니다."),
    DELETE_TASK_SUCCESS(200, "T003", "테스크가 삭제되었습니다."),
    CREATE_SUBTASK_SUCCESS(200, "T004", "서브테스크가 생성되었습니다."),
    UPDATE_SUBTASK_SUCCESS(200, "T005", "서브테스크가 수정되었습니다."),
    DELETE_SUBTASK_SUCCESS(200, "T006", "서브테스크가 삭제되었습니다."),
    GET_TASK_DETAIL_SUCCESS(200, "T007", "테스크 상세 조회에 성공하였습니다."),
    GET_REPORT_SUCCESS(200, "T008", "리포트 조회에 성공하였습니다."),
    GET_REPORT_TOTALTIME_SUCCESS(200, "T009", "리포트 총 시간 조회에 성공하였습니다."),

    // 중

    ;

    private final int status;
    private final String code;
    private final String message;
}
```

도메인 내에 있는 API마다 응답 결과를 위와 같이 세부적으로 정의할 수 있다.  
Controller 계층에선 다음과 같이 앞서 정의한 `ResultResponse`를 `ResponseEntity`에 담아 반환하면 된다.

```java
@ApiOperation(value = "회원 프로필 조회")
@ApiResponses({
        @ApiResponse(code = 200, message = "U002 - 회원 프로필을 조회하였습니다."),
        @ApiResponse(code = 401, message = "U003 - 로그인이 필요한 화면입니다.")
})
@GetMapping
public ResponseEntity<ResultResponse> getUserProfile(@ApiIgnore Authentication auth){
    Long userSeq = (Long)auth.getPrincipal();
    final UserProfileDto userProfileDto = userService.getUserProfile(userSeq);

    return ResponseEntity.ok(ResultResponse.of(GET_USERPROFILE_SUCCESS, userProfileDto));
}
```

---

참고 : [https://velog.io/@songs4805/Exception-Handling%EA%B3%BC-Response-%EC%BD%94%EB%93%9C-%EA%B0%9C%EC%84%A0](https://velog.io/@songs4805/Exception-Handling%EA%B3%BC-Response-%EC%BD%94%EB%93%9C-%EA%B0%9C%EC%84%A0)
