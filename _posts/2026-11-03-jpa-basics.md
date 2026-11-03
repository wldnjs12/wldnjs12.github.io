---
title: "[백엔드 여정] 02. JPA 기초: 객체와 DB 사이의 통역사"
date: 2026-11-03 09:00:00 +0900
categories: [Backend, JPA]
tags: [jpa, orm, hibernate, spring-data-jpa, entity, persistence-context]
---

## 👋 객체와 테이블, 왜 이렇게 안 맞을까?
지난 글에서 Spring MVC의 흐름을 정리하면서 **Repository(냉장고/재료 창고)** 가 DB에 접근한다고 했는데, 정작 "어떻게" 접근하는지는 다루지 않았습니다. 이번 글에서는 그 연결 고리인 **JPA**를 정리해 보려고 합니다.

자바는 **객체**로 생각하고, 데이터베이스는 **테이블(행과 열)** 로 생각합니다. 이 둘은 사고방식 자체가 달라서, 그 차이를 메우는 코드를 개발자가 직접 짜야 했습니다.

---

## 😵 1. JPA 이전의 세상: SQL 중심 개발
JPA가 없던 시절(JDBC, MyBatis)에는 이런 코드를 반복해서 작성해야 했습니다.

```java
String sql = "SELECT id, name, email FROM member WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setLong(1, memberId);
ResultSet rs = pstmt.executeQuery();

if (rs.next()) {
    Member member = new Member();
    member.setId(rs.getLong("id"));
    member.setName(rs.getString("name"));
    member.setEmail(rs.getString("email"));
}
```

### 🚫 무엇이 문제였을까?
1. **반복 작업:** 테이블마다 CRUD SQL과 객체 매핑 코드를 계속 써야 합니다.
2. **필드 하나 추가의 공포:** `Member`에 `phone` 필드를 추가하면 관련된 SQL을 전부 찾아서 고쳐야 합니다.
3. **패러다임 불일치:** 객체는 `member.getTeam()`처럼 **참조**로 관계를 따라가지만, 테이블은 `team_id` 같은 **외래 키(FK)와 JOIN**으로 관계를 맺습니다. 상속도 테이블에는 없는 개념이죠.

결국 "객체지향적으로 설계해도 SQL에 끌려다니는" 상황이 되어 버립니다.

---

## 🌉 2. ORM과 JPA: 통역사의 등장
**ORM(Object-Relational Mapping)** 은 객체와 테이블을 자동으로 매핑해 주는 기술입니다. 개발자는 객체를 다루듯이 코드를 짜고, SQL은 ORM이 대신 만들어 줍니다.

지난 글의 **Http Message Converter**가 자바 객체 ↔ JSON 통역사였다면, **JPA는 자바 객체 ↔ DB 테이블 통역사**라고 생각하면 이해가 쉽습니다.

### JPA, Hibernate, Spring Data JPA 구분하기 ⭐중요
처음에 가장 헷갈렸던 부분입니다.

| 이름 | 정체 | 비유 |
| --- | --- | --- |
| **JPA** | 자바 ORM **표준 명세(인터페이스)** | 요리 레시피 표준 |
| **Hibernate** | JPA를 실제로 **구현한 구현체** | 레시피대로 요리하는 셰프 |
| **Spring Data JPA** | JPA를 더 편하게 쓰도록 **스프링이 감싼 모듈** | 셰프를 부리는 주문 키오스크 |

즉, 우리가 Spring Boot에서 `JpaRepository`를 쓰면 **Spring Data JPA → JPA(표준) → Hibernate(구현체) → JDBC → DB** 순서로 내려가며 동작합니다.

---

## 🏷️ 3. Entity 매핑: 객체에 이름표 붙이기
JPA가 관리할 클래스에는 `@Entity`를 붙여 "이건 테이블과 연결된 객체야"라고 알려줍니다.

```java
@Entity
@Table(name = "member")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 20)
    private String name;

    @Column(unique = true)
    private String email;

    public Member(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public void changeName(String name) {
        this.name = name;
    }
}
```

* **`@Entity`**: JPA가 관리하는 객체임을 표시합니다.
* **`@Table`**: 매핑할 테이블 이름을 지정합니다. (생략하면 클래스 이름을 사용)
* **`@Id`**: 기본 키(PK)를 지정합니다.
* **`@GeneratedValue(strategy = IDENTITY)`**: PK 생성을 DB에 맡깁니다. (MySQL의 `AUTO_INCREMENT`)
* **`@Column`**: 컬럼 이름, `null` 허용 여부, 길이 등 세부 설정을 합니다.

### 💡 기본 생성자가 꼭 필요한 이유
JPA는 DB에서 데이터를 꺼낼 때 **리플렉션으로 기본 생성자를 호출해 객체를 만든 뒤** 값을 채웁니다. 그래서 기본 생성자가 반드시 있어야 하고, 아무나 쓰지 못하게 `protected`로 막아두는 것이 일반적입니다.

### 💡 Setter 대신 의미 있는 메서드
`setName()`을 열어두면 어디서든 값을 바꿀 수 있어서 추적이 어렵습니다. 대신 `changeName()`처럼 **의도가 드러나는 메서드**로 변경을 제한하는 것이 좋습니다. (지난 글에서 Entity를 직접 반환하지 말자고 한 것과 같은 맥락으로, Entity는 소중하게 다뤄야 합니다!)

---

## 🧠 4. 영속성 컨텍스트: JPA의 핵심 ⭐⭐가장 중요
JPA를 이해하는 데 가장 중요한 개념입니다. **영속성 컨텍스트(Persistence Context)** 는 "엔티티를 영구 저장하는 환경"으로, 애플리케이션과 DB 사이에 있는 **중간 저장소(작업 공간)** 라고 생각하면 됩니다.

식당 비유를 이어가 보면,
* **DB (냉장고):** 재료를 오래 보관하는 곳
* **영속성 컨텍스트 (조리대):** 지금 요리에 쓰는 재료를 꺼내 올려두는 곳

매번 냉장고 문을 열지 않고, 한 번 꺼낸 재료는 조리대에서 바로 집어 쓰는 거죠.

### 엔티티의 생명주기 (4가지 상태)
| 상태 | 설명 |
| --- | --- |
| **비영속 (new)** | `new Member()`로 막 만든 상태. JPA와 아무 관계 없음 |
| **영속 (managed)** | `em.persist()` 또는 조회로 영속성 컨텍스트에 **관리되는** 상태 |
| **준영속 (detached)** | 관리되다가 분리된 상태 (`em.detach()`, 트랜잭션 종료 등) |
| **삭제 (removed)** | `em.remove()`로 삭제 예약된 상태 |

### 영속성 컨텍스트가 주는 3가지 이점

**① 1차 캐시**
같은 트랜잭션 안에서 같은 엔티티를 두 번 조회하면, 두 번째는 DB에 가지 않고 1차 캐시에서 꺼냅니다.

```java
Member a = em.find(Member.class, 1L); // SELECT 쿼리 실행
Member b = em.find(Member.class, 1L); // 쿼리 없음! 1차 캐시에서 반환
System.out.println(a == b);           // true (동일성 보장)
```

**② 쓰기 지연 (Transactional Write-Behind)**
`persist()`를 호출할 때마다 바로 INSERT를 날리지 않고, **쓰기 지연 SQL 저장소**에 모아뒀다가 트랜잭션 커밋 시점에 한꺼번에 DB로 보냅니다. (flush)

> 단, `IDENTITY` 전략은 DB에 INSERT를 해야 PK를 알 수 있기 때문에 `persist()` 시점에 바로 INSERT가 나갑니다.
{: .prompt-info }

**③ 변경 감지 (Dirty Checking)** ⭐
제가 가장 신기했던 부분입니다. **`update()` 메서드가 따로 없습니다.**

```java
@Transactional
public void updateName(Long id, String newName) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(newName);
    // save() 호출 안 해도 됨!
}
```

영속성 컨텍스트는 엔티티를 처음 가져온 순간의 **스냅샷**을 저장해 둡니다. 트랜잭션이 끝날 때 현재 엔티티와 스냅샷을 비교해서, 달라진 게 있으면 **UPDATE 쿼리를 자동으로 생성**합니다. 자바 컬렉션에서 객체 값을 바꾸고 다시 넣지 않는 것처럼요.

> 변경 감지는 **영속 상태 + 트랜잭션 안**에서만 동작합니다. `@Transactional`이 빠지면 값이 바뀌어도 DB에 반영되지 않아요!
{: .prompt-warning }

---

## 🚀 5. Spring Data JPA: 인터페이스만 만들면 끝
순수 JPA는 `EntityManager`를 직접 다뤄야 하지만, Spring Data JPA를 쓰면 **인터페이스 선언만으로** 기본 CRUD가 완성됩니다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    // 메서드 이름만으로 쿼리 자동 생성 (쿼리 메서드)
    Optional<Member> findByEmail(String email);

    List<Member> findByNameContaining(String keyword);
}
```

* `JpaRepository<엔티티 타입, PK 타입>`을 상속하면 `save()`, `findById()`, `findAll()`, `delete()` 등이 자동으로 제공됩니다.
* `findByEmail`처럼 **정해진 규칙으로 메서드 이름을 지으면** `SELECT ... WHERE email = ?` 쿼리를 만들어 줍니다.
* 구현 클래스는 스프링이 실행 시점에 **프록시 객체**로 대신 만들어 줍니다.

### 지난 글의 계층 구조와 연결해 보면
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public Long join(MemberCreateRequest request) {
        Member member = new Member(request.getName(), request.getEmail());
        return memberRepository.save(member).getId();
    }

    public MemberResponse findMember(Long id) {
        Member member = memberRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("회원이 없습니다."));
        return MemberResponse.from(member); // Entity → DTO 변환 후 반환
    }
}
```

**Controller(DTO 받음) → Service(트랜잭션 + 비즈니스 로직) → Repository(JPA) → DB** 의 흐름이 이제 한 줄로 이어집니다. `@Transactional(readOnly = true)`를 클래스에 걸어두면 조회 시 스냅샷 비교를 생략하는 등 성능상 이점이 있고, 변경이 필요한 메서드에만 `@Transactional`을 따로 붙입니다.

---

## ✍️ 6. 스스로 점검해 보기
공부한 내용을 확인하려고 질문을 만들어 봤습니다. 답을 바로 말할 수 있는지 체크해 보세요.

1. JPA, Hibernate, Spring Data JPA의 차이는 무엇인가?
2. 엔티티에 기본 생성자가 필요한 이유는?
3. 영속성 컨텍스트의 1차 캐시는 언제 동작하고, 어떤 이점이 있는가?
4. `save()`를 호출하지 않았는데 UPDATE 쿼리가 나가는 이유는?
5. `@Transactional`이 없으면 변경 감지가 동작하지 않는 이유는?
6. `IDENTITY` 전략에서 쓰기 지연이 동작하지 않는 이유는?

---

## 🗓️ 다음 글 예고: 연관관계 매핑
이번 글에서는 테이블 하나와 엔티티 하나를 연결했습니다. 하지만 실제 서비스에서는 **회원과 팀, 주문과 상품**처럼 엔티티끼리 관계를 맺습니다. 다음 글에서는 `@ManyToOne`, `@OneToMany`, 연관관계의 주인, 그리고 악명 높은 **N+1 문제**의 입구까지 정리해 보겠습니다!
