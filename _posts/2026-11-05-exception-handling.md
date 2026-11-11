---
title: "[백엔드 여정] 05. 예외 처리: @RestControllerAdvice로 에러 응답 한 곳에서 관리하기"
date: 2026-11-05 09:00:00 +0900
categories: [Backend, Spring]
tags: [exception, exceptionhandler, restcontrolleradvice, validation, errorcode, responseentity]
---

## 👋 롤백은 됐는데, 클라이언트는 뭘 받을까?
지난 글에서 **Service에서 예외가 터지면 트랜잭션이 롤백된다**는 걸 정리했습니다. DB는 안전하게 지켜졌어요.

그런데 그 예외는 어디로 갈까요? Service 밖으로 던져진 예외는 Controller를 지나 **클라이언트에게 응답**으로 돌아갑니다. 이 응답을 어떻게 만드느냐가 이번 글의 주제입니다.

---

## 😵 1. 아무 처리도 안 하면 생기는 일
회원 조회 API에서 없는 회원을 찾는 상황을 볼게요.

```java
public MemberResponse findMember(Long id) {
    Member member = memberRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("회원이 없습니다."));
    return MemberResponse.from(member);
}
```

`GET /members/999`를 호출하면 Spring Boot는 기본으로 이런 응답을 보냅니다.

```json
{
  "timestamp": "2026-11-05T09:00:00.000+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/members/999"
}
```

### 🚫 무엇이 문제일까?
1. **상태 코드가 틀렸다:** 회원이 없는 건 클라이언트가 잘못된 ID를 보낸 것이니 **404 Not Found**여야 하는데, 서버가 고장 난 것처럼 **500**이 나갑니다.
2. **이유를 알 수 없다:** 우리가 적은 `"회원이 없습니다."` 메시지는 기본 설정에서 응답에 포함되지 않습니다. 프론트엔드는 "왜 실패했는지"를 사용자에게 보여줄 수 없어요.
3. **형식이 제각각:** API마다 에러 처리를 따로 하면 어떤 건 문자열, 어떤 건 JSON으로 응답 모양이 달라집니다.

> Spring Boot는 보안을 위해 기본적으로 **예외 메시지와 스택 트레이스를 응답에서 숨깁니다.** (`server.error.include-message`, `include-stacktrace` 기본값 `never`) 이걸 켜서 해결하려 하면 안 됩니다. 내부 클래스명, SQL 같은 정보가 외부로 새어 나갈 수 있어요. 우리가 원하는 건 **의도한 정보만 담은 응답**입니다.
{: .prompt-warning }

1편에서 **"상태 코드는 대화의 기본"** 이라고 했죠. 예외 처리는 결국 **실패했을 때도 매너 있게 대화하는 법**입니다.

---

## 🤔 2. Controller에서 `try-catch`로 처리하면?
가장 먼저 떠오르는 방법입니다.

```java
@GetMapping("/members/{id}")
public ResponseEntity<?> getMember(@PathVariable Long id) {
    try {
        return ResponseEntity.ok(memberService.findMember(id));
    } catch (IllegalArgumentException e) {
        return ResponseEntity.status(404).body(e.getMessage());
    }
}
```

동작은 하지만, 4편에서 봤던 JDBC 트랜잭션 코드와 **똑같은 문제**가 생깁니다.
* **모든 Controller 메서드마다** `try-catch`를 복사·붙여넣기 해야 합니다.
* 비즈니스 로직(조회)보다 **에러 처리 코드가 더 길어집니다.**
* 누군가 깜빡하고 안 넣으면 그 API만 500이 나갑니다.

4편에서 트랜잭션 코드를 프록시가 대신 둘러줬듯이, **예외 처리도 한 곳에 모아서 처리하고 싶다**는 생각이 듭니다.

---

## 🎯 3. `@ExceptionHandler`: 예외를 낚아채는 메서드
`@ExceptionHandler`를 붙인 메서드는 **지정한 예외가 Controller 밖으로 던져졌을 때 대신 응답을 만들어 줍니다.**

```java
@RestController
public class MemberController {

    @GetMapping("/members/{id}")
    public MemberResponse getMember(@PathVariable Long id) {
        return memberService.findMember(id);   // try-catch 없음!
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgument(IllegalArgumentException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(e.getMessage());
    }
}
```

Controller 메서드가 깔끔해졌습니다. 다만 이 `@ExceptionHandler`는 **이 Controller 안에서 난 예외만** 처리합니다. Controller가 10개면 10번 써야 하죠.

---

## 🌐 4. `@RestControllerAdvice`: 전역 예외 처리반 ⭐⭐가장 중요
`@ExceptionHandler` 메서드들을 **별도 클래스로 빼고 `@RestControllerAdvice`를 붙이면**, 모든 Controller에서 발생한 예외를 **한 곳에서** 처리할 수 있습니다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgument(IllegalArgumentException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(e.getMessage());
    }
}
```

* **`@ControllerAdvice`**: 여러 Controller에 공통 기능(예외 처리 등)을 적용하는 클래스
* **`@RestControllerAdvice`**: `@ControllerAdvice` + `@ResponseBody`. 반환값을 **JSON으로 응답 본문에** 담아 줍니다. REST API라면 이걸 씁니다.

식당으로 치면 **고객 응대 전담 창구**입니다. 주방에서 무슨 문제가 생기든 홀 직원이 손님에게 각자 둘러대는 게 아니라, 전담 창구가 **정해진 안내 문구와 절차**로 일관되게 응대하는 거예요.

### 예외가 흘러가는 길
1편의 DispatcherServlet, 4편의 트랜잭션 프록시와 연결해 보면 전체 흐름이 이렇게 됩니다.

```text
Service에서 예외 발생
  → 트랜잭션 프록시: 롤백 후 예외를 그대로 다시 던짐 (4편)
  → Controller 밖으로 전달
  → DispatcherServlet이 예외를 받음 (1편)
  → HandlerExceptionResolver가 처리할 @ExceptionHandler를 찾음
  → GlobalExceptionHandler가 ResponseEntity로 응답 생성
  → 클라이언트는 깔끔한 404 응답을 받음
```

> 롤백은 **트랜잭션 프록시**가, 응답 변환은 **`@RestControllerAdvice`** 가 맡습니다. 역할이 깔끔하게 나뉘어 있어서 Service는 **예외를 던지기만** 하면 됩니다. 4편의 "예외를 잡아서 삼키지 말자"와 같은 맥락이에요.
{: .prompt-tip }

---

## 🏗️ 5. 실전 구조: ErrorCode + 커스텀 예외 + ErrorResponse
`IllegalArgumentException`은 너무 범용적이라서 "회원 없음"인지 "잔액 부족"인지 구분이 안 됩니다. 실무에서는 보통 **세 가지를 직접 만들어서** 조합합니다.

### ① ErrorCode: 에러 목록을 enum으로 한눈에
```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    // Common
    INVALID_INPUT(HttpStatus.BAD_REQUEST, "C001", "입력값이 올바르지 않습니다."),
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "C999", "서버 오류가 발생했습니다."),

    // Member
    MEMBER_NOT_FOUND(HttpStatus.NOT_FOUND, "M001", "회원을 찾을 수 없습니다."),
    DUPLICATE_EMAIL(HttpStatus.CONFLICT, "M002", "이미 사용 중인 이메일입니다."),

    // Account
    INSUFFICIENT_BALANCE(HttpStatus.BAD_REQUEST, "A001", "잔액이 부족합니다.");

    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

* **`status`**: 어떤 HTTP 상태 코드로 응답할지
* **`code`**: 프론트엔드가 에러 종류를 구분하는 **고유 코드** (메시지 문구는 바뀔 수 있지만 코드는 안 바뀜)
* **`message`**: 사용자에게 보여줄 안내 문구

에러 종류가 **한 파일에 모여 있으니** 어떤 에러가 있는지 파악하기 쉽고, 프론트엔드와 문서로 공유하기도 좋습니다.

### ② 커스텀 예외: `RuntimeException`을 상속
```java
@Getter
public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

```java
public class MemberNotFoundException extends BusinessException {
    public MemberNotFoundException() {
        super(ErrorCode.MEMBER_NOT_FOUND);
    }
}
```

> **왜 `RuntimeException`일까?** 4편에서 봤듯이 **체크 예외는 기본적으로 롤백되지 않습니다.** 비즈니스 예외를 언체크 예외로 만들면 별도 설정 없이 롤백되고, 메서드마다 `throws`를 붙일 필요도 없어요.
{: .prompt-info }

이제 Service 코드가 이렇게 바뀝니다.

```java
public MemberResponse findMember(Long id) {
    Member member = memberRepository.findById(id)
            .orElseThrow(MemberNotFoundException::new);
    return MemberResponse.from(member);
}
```

```java
// Account 엔티티 안
public void withdraw(long amount) {
    if (this.balance < amount) {
        throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE);
    }
    this.balance -= amount;
}
```

### ③ ErrorResponse: 모든 에러 응답의 공통 모양
```java
public record ErrorResponse(
        String code,
        String message,
        List<FieldErrorDetail> errors
) {
    public record FieldErrorDetail(String field, String reason) {}

    public static ErrorResponse of(ErrorCode errorCode) {
        return new ErrorResponse(errorCode.getCode(), errorCode.getMessage(), List.of());
    }

    public static ErrorResponse of(ErrorCode errorCode, BindingResult bindingResult) {
        List<FieldErrorDetail> errors = bindingResult.getFieldErrors().stream()
                .map(e -> new FieldErrorDetail(e.getField(), e.getDefaultMessage()))
                .toList();
        return new ErrorResponse(errorCode.getCode(), errorCode.getMessage(), errors);
    }
}
```

`record`는 Java 16부터 생긴 문법으로, **필드·생성자·getter를 자동으로 만들어 주는 불변 데이터 클래스**입니다. DTO에 딱 맞아요. 어떤 에러든 응답이 **항상 같은 모양**이니까 프론트엔드는 `code`와 `message`만 보면 됩니다.

### 전부 조립한 GlobalExceptionHandler
```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 1. 우리가 직접 던진 비즈니스 예외
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        ErrorCode errorCode = e.getErrorCode();
        log.warn("[BusinessException] code={}, message={}", errorCode.getCode(), e.getMessage());
        return ResponseEntity
                .status(errorCode.getStatus())
                .body(ErrorResponse.of(errorCode));
    }

    // 2. @Valid 검증 실패 (6장)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        return ResponseEntity
                .badRequest()
                .body(ErrorResponse.of(ErrorCode.INVALID_INPUT, e.getBindingResult()));
    }

    // 3. 예상하지 못한 모든 예외 (최후의 방어선)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        log.error("[UnhandledException]", e);
        return ResponseEntity
                .internalServerError()
                .body(ErrorResponse.of(ErrorCode.INTERNAL_ERROR));
    }
}
```

`GET /members/999`의 응답이 이제 이렇게 바뀝니다. (상태 코드: **404 Not Found**)

```json
{
  "code": "M001",
  "message": "회원을 찾을 수 없습니다.",
  "errors": []
}
```

### 💡 알아두면 좋은 포인트
* **가장 구체적인 핸들러가 이깁니다.** `MemberNotFoundException`이 터지면 `Exception` 핸들러가 아니라 부모 타입 중 가장 가까운 **`BusinessException` 핸들러**가 처리합니다.
* **`Exception` 핸들러는 꼭 두세요.** 미처 예상하지 못한 예외도 공통 형식으로 응답하면서, 내부 정보는 숨길 수 있습니다.
* **로그 레벨을 나누세요.** 클라이언트 실수(4xx)는 `warn`, 서버 문제(5xx)는 스택 트레이스와 함께 `error`로 남깁니다. 응답에서는 숨기되, **로그에는 원인을 남겨야** 나중에 디버깅할 수 있어요.

---

## ✅ 6. 입력값 검증: `@Valid`로 Controller 입구에서 막기
"이름이 비어 있다", "이메일 형식이 아니다" 같은 잘못된 요청은 **Service까지 갈 필요도 없이** 입구에서 막는 게 좋습니다. 이때 쓰는 것이 **Bean Validation**입니다.

```gradle
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

### DTO에 검증 규칙 붙이기
```java
@Getter
@NoArgsConstructor
public class MemberCreateRequest {

    @NotBlank(message = "이름은 필수입니다.")
    @Size(max = 20, message = "이름은 20자 이하여야 합니다.")
    private String name;

    @NotBlank(message = "이메일은 필수입니다.")
    @Email(message = "이메일 형식이 올바르지 않습니다.")
    private String email;
}
```

| 어노테이션 | 검사 내용 |
| --- | --- |
| `@NotNull` | `null`이 아니어야 함 |
| `@NotEmpty` | `null`이 아니고 길이가 0보다 커야 함 |
| `@NotBlank` | `null`이 아니고 **공백이 아닌 문자**가 하나 이상 있어야 함 (문자열용) |
| `@Size(min, max)` | 문자열·컬렉션 길이 범위 |
| `@Email` | 이메일 형식 |
| `@Min`, `@Max`, `@Positive` | 숫자 범위 |

### Controller에서 `@Valid`로 검증 실행
```java
@PostMapping("/members")
public ResponseEntity<Long> join(@Valid @RequestBody MemberCreateRequest request) {
    Long memberId = memberService.join(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(memberId);
}
```

`@Valid`가 붙은 `@RequestBody`가 규칙을 어기면 Controller 메서드는 **실행조차 되지 않고** `MethodArgumentNotValidException`이 발생합니다. 그리고 우리가 만든 핸들러가 이걸 받아서 **어떤 필드가 왜 틀렸는지**까지 응답해 줍니다. (상태 코드: **400 Bad Request**)

```json
{
  "code": "C001",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    { "field": "name", "reason": "이름은 필수입니다." },
    { "field": "email", "reason": "이메일 형식이 올바르지 않습니다." }
  ]
}
```

> **검증 역할 나누기:** "형식이 맞는가"(빈 값, 길이, 이메일 형식)는 **DTO + `@Valid`** 가, "비즈니스 규칙에 맞는가"(이메일 중복, 잔액 부족)는 DB 조회가 필요하니 **Service/Entity** 가 `BusinessException`으로 처리합니다.
{: .prompt-tip }

---

## 📄 7. 참고: 표준 에러 형식 `ProblemDetail`
에러 응답 형식을 우리가 직접 정했지만, 사실 HTTP API 에러 응답에는 **RFC 9457(구 RFC 7807)** 이라는 표준 형식도 있습니다. Spring 6 / Spring Boot 3부터는 이 표준을 위한 **`ProblemDetail`** 클래스를 제공해요.

```java
@ExceptionHandler(BusinessException.class)
public ProblemDetail handleBusiness(BusinessException e) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            e.getErrorCode().getStatus(), e.getMessage());
    problem.setProperty("code", e.getErrorCode().getCode());
    return problem;
}
```

```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "회원을 찾을 수 없습니다.",
  "instance": "/members/999",
  "code": "M001"
}
```

직접 만든 `ErrorResponse`와 표준 `ProblemDetail` 중 무엇을 쓰든, 중요한 건 **팀 안에서 하나로 통일하는 것**입니다. 이런 표준이 있다는 것만 알아두면 충분해요.

---

## 🧭 8. 한 장으로 정리
| 상황 | 이렇게 하자 |
| --- | --- |
| 예외 처리 위치 | Controller `try-catch` ❌ → **`@RestControllerAdvice`** 한 곳 |
| 에러 종류 관리 | **`ErrorCode` enum** (상태 코드 + 고유 코드 + 메시지) |
| 비즈니스 예외 | **`RuntimeException`** 을 상속한 커스텀 예외 → 자동 롤백 |
| 응답 형식 | **공통 `ErrorResponse`** (또는 `ProblemDetail`)로 통일 |
| 입력 형식 검증 | DTO에 `@NotBlank` 등 + Controller에 **`@Valid`** |
| 예상 못 한 예외 | `Exception` 핸들러로 500 + **로그는 `error`로 상세히** |
| 내부 정보 | 스택 트레이스·SQL은 **응답에 절대 노출하지 않기** |

---

## ✍️ 9. 스스로 점검해 보기
1. 예외를 처리하지 않으면 Spring Boot는 어떤 응답을 보내고, 무엇이 문제인가?
2. `@ExceptionHandler`와 `@RestControllerAdvice`의 차이는?
3. Service에서 예외가 발생했을 때 롤백과 에러 응답 생성은 각각 누가 담당하는가?
4. 커스텀 비즈니스 예외를 `RuntimeException`으로 만드는 이유는?
5. `ErrorCode`에 HTTP 상태 코드와 별개로 고유 `code`를 두는 이유는?
6. 여러 `@ExceptionHandler`가 있을 때 어떤 핸들러가 선택되는가?
7. `@NotNull`, `@NotEmpty`, `@NotBlank`의 차이는?
8. 입력값 검증 중 DTO에서 할 것과 Service에서 할 것은 어떻게 나누는가?

---

## 🗓️ 다음 글 예고: 스프링 IoC와 DI
지금까지 `@Service`, `@RequiredArgsConstructor`, `private final MemberRepository`를 당연하게 써 왔습니다. 그런데 **`new`를 한 번도 안 했는데 객체는 누가 만들어서 넣어준 걸까요?** 다음 글에서는 스프링의 뿌리인 **IoC(제어의 역전)와 DI(의존성 주입)**, 그리고 **스프링 빈과 컨테이너**를 정리해 보겠습니다. 4편의 프록시가 어떻게 진짜 객체 대신 주입될 수 있었는지도 여기서 연결됩니다!
