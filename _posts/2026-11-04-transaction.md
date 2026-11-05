---
title: "[백엔드 여정] 04. 트랜잭션과 @Transactional: 전부 성공하거나, 전부 없던 일로"
date: 2026-11-04 15:00:00 +0900
categories: [Backend, Spring]
tags: [transaction, transactional, acid, isolation, propagation, aop, proxy]
---

## 👋 "트랜잭션 안에서만"의 정체
2편과 3편을 쓰면서 **"트랜잭션 안에서만 동작한다"** 는 말을 계속 했습니다.
* 변경 감지(dirty checking)는 트랜잭션이 커밋될 때 UPDATE를 만들고,
* 지연 로딩 프록시는 트랜잭션이 끝나면 초기화할 수 없었죠.

그런데 정작 **트랜잭션이 뭔지**, `@Transactional` 한 줄이 **어떻게** 그걸 가능하게 하는지는 넘어갔습니다. 이번 글에서 정리해 보겠습니다.

---

## 💸 1. 트랜잭션이란?
**트랜잭션(Transaction)** 은 **더 이상 쪼갤 수 없는 작업의 묶음**입니다. 묶음 안의 작업은 **전부 성공하거나(Commit), 전부 취소되어야(Rollback)** 합니다.

가장 유명한 예시는 계좌 이체입니다. 지원이 민수에게 1만 원을 보낸다고 해 볼게요.

```sql
UPDATE account SET balance = balance - 10000 WHERE owner = '지원';  -- ① 출금
UPDATE account SET balance = balance + 10000 WHERE owner = '민수';  -- ② 입금
```

①은 성공했는데 ②를 실행하기 직전에 서버가 죽으면? **지원의 돈은 빠졌는데 민수는 못 받은** 상태가 됩니다. 1만 원이 허공으로 사라진 거죠.

트랜잭션으로 ①과 ②를 묶으면, 중간에 실패했을 때 ①까지 **없던 일로 되돌립니다(Rollback).** 둘 다 성공했을 때만 **확정합니다(Commit).**

식당으로 치면 **코스 요리 주문**입니다. 메인 요리 재료가 떨어졌다면 애피타이저만 내보내는 게 아니라, 주문 전체를 취소하고 처음 상태로 되돌리는 거예요.

---

## 🧱 2. ACID: 트랜잭션이 지켜야 할 4가지 약속 ⭐
| 성질 | 뜻 | 계좌 이체로 보면 |
| --- | --- | --- |
| **A**tomicity (원자성) | 전부 성공 or 전부 실패 | 출금만 되고 입금이 안 되는 일은 없다 |
| **C**onsistency (일관성) | 실행 전후로 DB 규칙(제약 조건)이 지켜진다 | 잔액이 음수가 되면 안 된다는 규칙이 깨지지 않는다 |
| **I**solation (격리성) | 동시에 실행되는 트랜잭션끼리 간섭하지 않는다 | 이체 도중의 '중간 상태'를 다른 사람이 보지 못한다 |
| **D**urability (지속성) | 커밋된 결과는 영구히 남는다 | 이체 완료 후 서버가 꺼져도 결과는 유지된다 |

이 중에서 개발자가 가장 많이 신경 써야 하는 건 **격리성(Isolation)** 입니다. 완벽하게 격리하면 안전하지만 트랜잭션이 한 줄로 서서 기다려야 해서 **느려지기** 때문에, DB는 격리 수준을 단계별로 고를 수 있게 해 줍니다.

---

## 🚧 3. 격리 수준(Isolation Level)
격리가 약하면 생기는 문제부터 보겠습니다.

* **Dirty Read:** 다른 트랜잭션이 **아직 커밋하지 않은** 데이터를 읽어 버림. 그 트랜잭션이 롤백하면 존재한 적 없는 데이터를 본 셈.
* **Non-Repeatable Read:** 같은 행을 두 번 읽었는데, 그 사이 다른 트랜잭션이 **수정·커밋**해서 값이 달라짐.
* **Phantom Read:** 같은 조건으로 두 번 조회했는데, 그 사이 다른 트랜잭션이 **행을 추가·삭제**해서 결과 행 개수가 달라짐.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | :---: | :---: | :---: |
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | ✅ 방지 | 발생 | 발생 |
| REPEATABLE READ | ✅ 방지 | ✅ 방지 | 발생 (표준 기준) |
| SERIALIZABLE | ✅ 방지 | ✅ 방지 | ✅ 방지 |

아래로 갈수록 안전하지만 동시 처리 성능은 떨어집니다.

> 기본값은 DB마다 다릅니다. **MySQL(InnoDB)은 `REPEATABLE READ`**, **PostgreSQL과 Oracle은 `READ COMMITTED`** 가 기본이에요. 그리고 MySQL InnoDB는 MVCC와 넥스트 키 락 덕분에 `REPEATABLE READ`에서도 대부분의 Phantom Read를 막아줍니다. 스프링에서 따로 지정하지 않으면 DB 기본값을 따릅니다.
{: .prompt-info }

---

## 🌱 4. 스프링 없이 트랜잭션 다루기: 반복과의 싸움
순수 JDBC로 트랜잭션을 걸면 이런 모양이 됩니다.

```java
Connection conn = dataSource.getConnection();
try {
    conn.setAutoCommit(false);          // 트랜잭션 시작

    accountDao.withdraw(conn, "지원", 10000);
    accountDao.deposit(conn, "민수", 10000);

    conn.commit();                      // 성공 → 커밋
} catch (Exception e) {
    conn.rollback();                    // 실패 → 롤백
    throw e;
} finally {
    conn.close();
}
```

문제가 보이시나요?
1. **비즈니스 로직은 두 줄인데 나머지가 전부 트랜잭션 코드**입니다.
2. 같은 커넥션을 써야 같은 트랜잭션이라서, **`conn`을 메서드마다 파라미터로 넘겨야** 합니다.
3. 이 `try-catch-finally`가 서비스 메서드마다 **복사·붙여넣기** 됩니다.

---

## ✨ 5. `@Transactional`: 한 줄로 끝
스프링에서는 위 코드가 이렇게 바뀝니다.

```java
@Service
@RequiredArgsConstructor
public class AccountService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromId, Long toId, long amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.withdraw(amount);   // 잔액 부족이면 예외 발생 → 전체 롤백
        to.deposit(amount);
        // save() 없음! 커밋 시점에 변경 감지로 UPDATE 2개 실행 (2편 내용)
    }
}
```

**비즈니스 로직만 남았습니다.** 커넥션을 얻고, 시작하고, 커밋하거나 롤백하는 일은 스프링이 대신 해줍니다.

### 어떻게 가능할까? 프록시(Proxy) ⭐⭐가장 중요
3편에서 지연 로딩의 '가짜 객체'로 **프록시**가 나왔는데, 여기서 또 등장합니다.

`@Transactional`이 붙은 클래스를 스프링 빈으로 등록할 때, 스프링은 **진짜 `AccountService`를 감싼 프록시 객체**를 만들어서 대신 등록합니다. 그래서 Controller가 주입받는 건 진짜가 아니라 **프록시**예요.

```text
Controller
  └─ transfer() 호출
      ▼
   [AccountService 프록시]  ← 스프링이 만든 가짜
      1. 트랜잭션 시작
      2. 진짜 AccountService.transfer() 호출  → 비즈니스 로직 실행
      3. 정상 종료면 커밋 / 예외 발생이면 롤백
```

프록시가 하는 일은 결국 4장의 `try-catch-finally` 코드와 같습니다. 그걸 **우리 대신 앞뒤로 둘러주는 것**뿐이에요. 이렇게 **공통 관심사(트랜잭션, 로깅 등)를 핵심 로직과 분리**하는 기법을 **AOP(관점 지향 프로그래밍)** 라고 합니다.

식당으로 치면 **매니저**입니다. 홀 직원(Controller)은 주방(Service)에 직접 주문하는 줄 알지만, 사실 중간에 매니저(프록시)가 주문서를 받아 "이 주문 시작합니다"를 기록하고, 요리가 망하면 주문 전체를 취소 처리해 줍니다.

또, 트랜잭션이 시작될 때 스프링은 **커넥션을 현재 스레드에 보관**해 둡니다. 그래서 Repository들이 커넥션을 파라미터로 받지 않아도 **같은 커넥션(=같은 트랜잭션)** 을 쓸 수 있어요. 4장의 2번 문제도 해결된 거죠.

---

## ⚠️ 6. `@Transactional`이 안 먹히는 대표적인 실수
프록시 구조를 이해하면 왜 안 되는지도 바로 보입니다.

### 실수 ① 같은 클래스 안에서 호출 (Self-Invocation)
```java
@Service
public class OrderService {

    public void order() {
        // ...
        this.saveOrder();   // ❌ 트랜잭션 적용 안 됨!
    }

    @Transactional
    public void saveOrder() { ... }
}
```

외부에서 `order()`를 호출하면 프록시를 거치지만, `order()`는 `@Transactional`이 없으니 프록시는 트랜잭션 없이 진짜 객체로 넘깁니다. 그리고 진짜 객체 안에서 `this.saveOrder()`를 부르면 **프록시를 거치지 않고 자기 자신을 직접 호출**합니다. 매니저를 거치지 않고 주방 안에서 직접 일을 처리한 셈이라 트랜잭션이 걸리지 않아요.

**해결:** `saveOrder()`를 **별도의 클래스(빈)로 분리**해서 주입받아 호출하는 것이 가장 깔끔합니다.

### 실수 ② `private` 메서드에 붙이기
프록시는 진짜 클래스를 **상속**해서 메서드를 오버라이드하는 방식으로 만들어지기 때문에, 오버라이드할 수 없는 `private` 메서드는 가로챌 수 없습니다. **`@Transactional`은 `public` 메서드에 붙이는 것**을 기본으로 생각하세요.

### 실수 ③ Checked Exception은 롤백되지 않는다 ⭐
스프링의 기본 롤백 규칙은 이렇습니다.

| 예외 종류 | 예시 | 기본 동작 |
| --- | --- | --- |
| **Unchecked** (`RuntimeException`, `Error`) | `IllegalArgumentException`, `NullPointerException` | **롤백** |
| **Checked** (`Exception`을 상속, `RuntimeException` 제외) | `IOException`, 직접 만든 `extends Exception` | **커밋** 😱 |

체크 예외는 "비즈니스적으로 예상 가능한 상황"이라고 보고 커밋해 버립니다. 체크 예외에도 롤백하고 싶다면 직접 지정해야 해요.

```java
@Transactional(rollbackFor = Exception.class)
public void doSomething() throws Exception { ... }
```

### 실수 ④ 예외를 잡아서 삼켜버리기
```java
@Transactional
public void transfer(...) {
    try {
        from.withdraw(amount);
        to.deposit(amount);
    } catch (Exception e) {
        log.error("이체 실패", e);   // 예외를 먹어버림
    }
}
```

예외가 메서드 밖으로 나가지 않으니 프록시는 **"정상 종료됐네?"** 하고 **커밋**합니다. 출금만 반영되는 최악의 상황이 다시 생길 수 있어요. 잡았다면 **다시 던지거나**, 정말 처리할 거라면 롤백 여부를 신중히 판단해야 합니다.

---

## 🎛️ 7. 알아두면 좋은 옵션
### `readOnly = true`
```java
@Service
@Transactional(readOnly = true)   // 클래스 전체 기본값: 읽기 전용
public class MemberService {

    public MemberResponse findMember(Long id) { ... }   // 읽기 전용 적용

    @Transactional                                       // 쓰기 메서드만 덮어쓰기
    public Long join(MemberCreateRequest request) { ... }
}
```

2편에서 썼던 패턴입니다. 읽기 전용이면 JPA(Hibernate)가 **변경 감지용 스냅샷을 만들지 않고, flush도 하지 않아서** 조회 성능이 좋아집니다. DB나 설정에 따라서는 읽기 전용 복제 DB로 요청을 보내는 데 활용하기도 해요.

### 전파(Propagation): 트랜잭션 안에서 또 트랜잭션을 만나면?
`@Transactional` 메서드가 다른 `@Transactional` 메서드를 호출하면 어떻게 될까요? 이걸 정하는 게 **전파 옵션**입니다.

| 옵션 | 동작 |
| --- | --- |
| **`REQUIRED`** (기본값) | 이미 트랜잭션이 있으면 **참여**, 없으면 새로 시작 |
| **`REQUIRES_NEW`** | 기존 트랜잭션은 잠시 멈추고 **항상 새 트랜잭션** 시작 |

기본값 `REQUIRED`에서는 바깥과 안쪽이 **하나의 트랜잭션**입니다. 그래서 안쪽에서 런타임 예외가 터지면 트랜잭션 전체가 **"롤백 전용"으로 표시**되고, 바깥에서 그 예외를 `catch`로 잡아도 결국 커밋 시점에 `UnexpectedRollbackException`이 발생하며 전부 롤백됩니다.

> **"주문은 실패해도 실패 로그는 꼭 남겨야 한다"** 처럼 바깥과 운명을 분리하고 싶을 때 로그 저장 쪽에 `REQUIRES_NEW`를 씁니다. 다만 커넥션을 하나 더 쓰기 때문에 남용하면 커넥션 풀이 고갈될 수 있어요.
{: .prompt-warning }

---

## 🏛️ 8. 그래서 `@Transactional`은 어디에?
1편의 계층 구조를 다시 떠올려 보면 답이 나옵니다.

* **Controller:** 요청/응답만 담당 → ❌
* **Service:** 비즈니스 로직 = **하나의 작업 단위** → ✅ **여기에!**
* **Repository:** Spring Data JPA가 이미 메서드 단위로 트랜잭션을 걸어 줌 → 따로 안 붙여도 됨

계좌 이체처럼 "출금 + 입금"이 **하나의 의미 있는 작업**이 되는 곳이 Service 계층이기 때문에, 트랜잭션 경계도 Service에 두는 것이 정석입니다.

---

## ✍️ 9. 스스로 점검해 보기
1. 트랜잭션이 필요한 이유를 계좌 이체 예시로 설명할 수 있는가?
2. ACID 각각의 의미는?
3. Dirty Read, Non-Repeatable Read, Phantom Read의 차이는? MySQL의 기본 격리 수준은?
4. `@Transactional`은 내부적으로 어떻게 동작하는가? (프록시, AOP)
5. 같은 클래스 안의 메서드를 호출하면 `@Transactional`이 적용되지 않는 이유는?
6. 체크 예외가 발생하면 기본적으로 롤백되는가?
7. `readOnly = true`를 쓰면 어떤 이점이 있는가?
8. `REQUIRED`와 `REQUIRES_NEW`의 차이는?

---

## 🗓️ 다음 글 예고: 예외 처리
이번 글에서 **예외가 나면 롤백된다**는 걸 배웠습니다. 그런데 롤백된 뒤에 클라이언트는 어떤 응답을 받을까요? 그대로 두면 지저분한 `500 Internal Server Error`와 스택 트레이스가 날아갑니다. 다음 글에서는 **`@RestControllerAdvice`와 `@ExceptionHandler`로 예외를 한 곳에서 처리**하고, 1편의 `ResponseEntity`로 **깔끔한 에러 응답**을 만드는 방법을 정리하겠습니다!
