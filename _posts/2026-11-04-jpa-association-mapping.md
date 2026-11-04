---
title: "[백엔드 여정] 03. JPA 연관관계 매핑: 객체는 참조로, 테이블은 외래 키로"
date: 2026-11-04 09:00:00 +0900
categories: [Backend, JPA]
tags: [jpa, association, manytoone, onetomany, mappedby, lazy-loading, n+1]
---

## 👋 엔티티끼리 관계 맺기
지난 글에서는 `Member` 엔티티 하나를 `member` 테이블 하나에 연결했습니다. 하지만 실제 서비스에서는 **회원은 팀에 소속되고, 주문에는 상품이 담기는** 것처럼 데이터끼리 관계가 있습니다.

이번 글에서는 이 관계를 JPA로 표현하는 **연관관계 매핑**을 정리합니다. 2편에서 말했던 **패러다임 불일치**를 JPA가 실제로 어떻게 해결하는지 보는 글이기도 해요.

---

## 🔗 1. 테이블과 객체는 관계를 맺는 방식이 다르다
**회원(Member)과 팀(Team)** 을 예로 들어 보겠습니다. 한 팀에는 여러 회원이 있고, 한 회원은 한 팀에만 속합니다. (**회원 N : 팀 1**)

### 테이블: 외래 키(FK) 하나로 끝
```sql
CREATE TABLE team (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE member (
    id      BIGINT PRIMARY KEY,
    name    VARCHAR(255),
    team_id BIGINT,
    FOREIGN KEY (team_id) REFERENCES team(id)
);
```

테이블은 `member.team_id` 하나만 있으면 **양쪽 방향 모두** 조회할 수 있습니다.
* 회원의 팀 찾기: `member JOIN team ON member.team_id = team.id`
* 팀의 회원들 찾기: `team JOIN member ON team.id = member.team_id`

### 객체: 참조가 있는 쪽으로만 갈 수 있음
```java
class Member {
    Team team;              // 회원 → 팀 (O)
}

class Team {
    List<Member> members;   // 팀 → 회원은 이 필드가 따로 있어야 가능
}
```

객체는 **참조(필드)를 가진 쪽에서만** 상대에게 갈 수 있습니다. 양쪽으로 다니려면 **양쪽 모두에 필드**를 둬야 하죠.

> **핵심 차이:** 테이블의 FK는 방향이 없지만(양방향), 객체의 참조는 방향이 있다(단방향).
{: .prompt-tip }

---

## ➡️ 2. 단방향 연관관계: `@ManyToOne`
가장 기본이자 **가장 많이 쓰는** 매핑입니다. 회원이 팀을 참조하도록 만들어 볼게요.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    public Member(String name, Team team) {
        this.name = name;
        this.team = team;
    }
}
```

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    public Team(String name) {
        this.name = name;
    }
}
```

* **`@ManyToOne`**: "회원(Many)이 팀(One)을 참조한다"는 다대일 관계를 표시합니다.
* **`@JoinColumn(name = "team_id")`**: 이 관계를 테이블의 어떤 FK 컬럼과 연결할지 지정합니다.
* **`fetch = FetchType.LAZY`**: 지금은 "꼭 붙이는 옵션" 정도로만 기억하고, 4장에서 자세히 설명합니다.

이제 `member.getTeam().getName()`처럼 **객체 그래프를 탐색**하듯 코드를 쓸 수 있습니다. `team_id`를 꺼내서 팀을 다시 조회하는 코드를 직접 짤 필요가 없어진 거죠.

```java
Team team = teamRepository.save(new Team("백엔드팀"));
Member member = memberRepository.save(new Member("지원", team));

// JPA가 INSERT INTO member (name, team_id) VALUES ('지원', 1) 을 만들어 줌
```

---

## ↔️ 3. 양방향 연관관계와 '연관관계의 주인' ⭐⭐가장 중요
"팀에 속한 회원 목록"도 객체로 조회하고 싶다면 `Team`에도 필드를 추가합니다.

```java
@Entity
public class Team {
    // ...
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```

그런데 여기서 문제가 생깁니다. 이제 **FK(`team_id`)를 바꿀 수 있는 곳이 두 군데**가 됐어요.
* `member.team`을 바꾸면 `team_id`가 바뀌어야 할까?
* `team.members`에 회원을 추가하면 `team_id`가 바뀌어야 할까?

테이블에는 FK가 **하나**뿐인데 객체에는 참조가 **두 개**라서, JPA는 둘 중 하나만 믿기로 정합니다. 이 믿는 쪽을 **연관관계의 주인(Owner)** 이라고 합니다.

### 규칙은 딱 두 줄
1. **FK가 있는 쪽이 주인이다.** 회원-팀에서는 `member` 테이블에 `team_id`가 있으니 **`Member.team`이 주인**입니다. (보통 **N쪽**, 즉 `@ManyToOne` 쪽)
2. **주인이 아닌 쪽은 `mappedBy`로 "나는 저 필드에 의해 매핑됐어"라고 표시**하고, **읽기만** 가능합니다.

| | `Member.team` | `Team.members` |
| --- | --- | --- |
| 어노테이션 | `@ManyToOne` + `@JoinColumn` | `@OneToMany(mappedBy = "team")` |
| 역할 | **주인** (FK 등록·수정) | 읽기 전용 거울 |
| DB 반영 | ✅ | ❌ (값을 넣어도 무시됨) |

### 🚫 가장 흔한 실수
```java
Team team = new Team("백엔드팀");
em.persist(team);

Member member = new Member("지원", null);
team.getMembers().add(member);   // 주인이 아닌 쪽에만 값 설정
em.persist(member);
// 결과: member.team_id = NULL 😱
```

주인이 아닌 `team.members`에만 넣었기 때문에 JPA는 이 변경을 **무시**합니다. FK는 반드시 **주인 쪽(`member.team`)** 에 설정해야 해요.

### 💡 연관관계 편의 메서드
그렇다고 주인 쪽에만 값을 넣으면, 같은 트랜잭션 안에서 `team.getMembers()`를 조회했을 때 방금 추가한 회원이 안 보입니다. (아직 DB에서 다시 읽어온 게 아니라 1차 캐시의 객체를 보고 있으니까요.)

그래서 **양쪽을 한 번에 맞춰주는 메서드**를 만들어 두는 것이 좋습니다.

```java
// Member 클래스 안
public void changeTeam(Team team) {
    this.team = team;               // 주인 쪽: DB에 반영됨
    team.getMembers().add(this);    // 반대쪽: 객체 상태 일관성 유지
}
```

> **양방향 매핑은 꼭 필요할 때만** 추가하세요. 단방향(`@ManyToOne`)만으로도 연관관계 매핑은 완성됩니다. 반대 방향 조회가 필요해졌을 때 `mappedBy` 필드를 추가해도 **테이블은 전혀 바뀌지 않아요.**
{: .prompt-info }

### ⚠️ 양방향일 때 무한 루프 주의
`Member`는 `Team`을, `Team`은 다시 `List<Member>`를 참조합니다. 이 상태에서
* Lombok의 `@ToString`을 양쪽에 붙이거나
* **엔티티를 그대로 JSON으로 반환**하면

`member → team → members → member → team → ...` 식으로 끝없이 서로를 호출해서 `StackOverflowError`가 납니다. 1편에서 **"Entity를 직접 반환하지 말고 DTO를 쓰자"** 고 한 이유가 하나 더 생긴 셈이에요.

---

## 💤 4. 지연 로딩(LAZY) vs 즉시 로딩(EAGER)
회원을 조회할 때 **팀도 같이 가져와야 할까요?** 회원 이름만 필요한 화면이라면 팀 정보는 낭비입니다. JPA는 이 시점을 고를 수 있게 해 줍니다.

| | 즉시 로딩 (`EAGER`) | 지연 로딩 (`LAZY`) |
| --- | --- | --- |
| 동작 | 회원 조회 시 팀도 **바로** 조회 | 팀은 **실제로 쓸 때** 조회 |
| 팀 필드에 들어가는 것 | 진짜 `Team` 객체 | **프록시(가짜) 객체** |
| 기본값 | `@ManyToOne`, `@OneToOne` | `@OneToMany`, `@ManyToMany` |

### 프록시: 가짜 객체로 자리만 맡아두기
지연 로딩을 쓰면 `member.getTeam()`에는 진짜 `Team` 대신 **`Team`을 상속한 프록시 객체**가 들어갑니다. 식당으로 치면 **"재료 예약표"** 를 조리대에 올려둔 거예요. 실제로 재료가 필요해지면 그때 냉장고(DB)에서 꺼내옵니다.

```java
Member member = memberRepository.findById(1L).orElseThrow();
// SELECT * FROM member WHERE id = 1   (팀은 조회 안 함)

Team team = member.getTeam();   // 아직 쿼리 없음 (프록시)
team.getName();                 // 이 순간 SELECT * FROM team WHERE id = ? 실행
```

> 프록시는 **영속성 컨텍스트가 살아 있을 때(트랜잭션 안)** 만 초기화할 수 있습니다. 트랜잭션이 끝난 뒤에 처음 접근하면 `LazyInitializationException`이 발생할 수 있어요. 필요한 데이터는 Service 계층(트랜잭션 안)에서 DTO로 다 꺼내 담아 두는 습관을 들이면 됩니다.
{: .prompt-warning }

### 결론: 실무에서는 전부 LAZY
`EAGER`는 편해 보이지만 **예상하지 못한 쿼리**가 마구 나갈 수 있습니다. 그래서 **모든 연관관계를 `LAZY`로 설정**하고, 같이 조회해야 할 때만 다음 장의 **fetch join**을 쓰는 것이 정석입니다. 특히 `@ManyToOne`은 기본값이 `EAGER`라서 **직접 `LAZY`를 적어줘야** 합니다.

---

## 💣 5. N+1 문제 맛보기
JPA를 쓰다 보면 반드시 만나는 성능 문제입니다. 회원 목록과 각 회원의 팀 이름을 보여주는 기능을 만들어 볼게요.

```java
List<Member> members = memberRepository.findAll();   // ① 쿼리 1번

for (Member m : members) {
    System.out.println(m.getTeam().getName());      // ② 회원마다 팀 조회 쿼리
}
```

```sql
SELECT * FROM member;                  -- 1번
SELECT * FROM team WHERE id = 1;       -- 회원1의 팀
SELECT * FROM team WHERE id = 2;       -- 회원2의 팀
SELECT * FROM team WHERE id = 3;       -- 회원3의 팀
...                                    -- N번
```

**처음 1번의 쿼리로 N개의 결과를 가져왔는데, 그 N개 각각에 대해 추가 쿼리가 나가서 총 1 + N번**이 실행되는 것을 **N+1 문제**라고 합니다. (같은 팀은 1차 캐시 덕분에 다시 조회하지 않으므로, 정확히는 '서로 다른 팀 수'만큼 나갑니다.) 회원이 1,000명이면 쿼리도 최대 1,001번이에요.

> **"그럼 EAGER로 하면 되지 않나?"** 아닙니다. `findAll()`은 JPQL(`SELECT m FROM Member m`)로 실행되는데, JPQL은 일단 쓰인 그대로 회원만 조회한 뒤 EAGER 설정을 보고 팀을 **따로** 조회합니다. 결국 똑같이 N+1이 발생해요.
{: .prompt-danger }

### 해결 맛보기: fetch join
**"이번엔 팀까지 한 번에 가져와!"** 라고 직접 지정하는 방법입니다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    @Query("select m from Member m join fetch m.team")
    List<Member> findAllWithTeam();
}
```

```sql
SELECT m.*, t.*
FROM member m
INNER JOIN team t ON m.team_id = t.id;   -- 단 1번!
```

이 외에도 `@EntityGraph`, `default_batch_fetch_size` 설정 같은 방법이 있는데, 이건 따로 깊게 다뤄볼 만한 주제라서 나중에 N+1 심화 글로 정리하겠습니다.

---

## 🧭 6. 한 장으로 정리
| 상황 | 이렇게 하자 |
| --- | --- |
| 연관관계 기본 | `@ManyToOne` + `@JoinColumn` **단방향**부터 |
| 반대 방향 조회 필요 | `@OneToMany(mappedBy = "...")` 추가 (테이블 변화 없음) |
| 연관관계의 주인 | **FK가 있는 쪽 = N쪽** |
| 값 설정 | **주인 쪽에 반드시**, 편의 메서드로 양쪽 동시에 |
| 로딩 전략 | **전부 `LAZY`**, `@ManyToOne`은 직접 명시 |
| 같이 조회 필요 | **fetch join** |
| API 응답 | 엔티티 말고 **DTO** (무한 루프 + 지연 로딩 문제 회피) |

---

## ✍️ 7. 스스로 점검해 보기
1. 테이블의 외래 키와 객체의 참조는 '방향' 측면에서 어떻게 다른가?
2. 연관관계의 주인은 어떤 기준으로 정하고, 주인이 아닌 쪽에서 값을 바꾸면 어떻게 되는가?
3. `mappedBy`는 어느 쪽에 붙이며, 무엇을 의미하는가?
4. 연관관계 편의 메서드는 왜 필요한가?
5. 지연 로딩에서 `member.getTeam()`이 반환하는 것은 무엇이고, 실제 쿼리는 언제 나가는가?
6. N+1 문제란 무엇이고, `EAGER`로 바꿔도 해결되지 않는 이유는?
7. fetch join은 N+1 문제를 어떻게 해결하는가?

---

## 🗓️ 다음 글 예고: 트랜잭션과 `@Transactional`
2편과 3편 내내 **"트랜잭션 안에서만"** 이라는 말이 계속 나왔습니다. 변경 감지도, 지연 로딩도 결국 트랜잭션이 열려 있어야 동작하죠. 다음 글에서는 **트랜잭션의 ACID**, 스프링의 **`@Transactional`이 동작하는 원리(프록시)**, 그리고 자주 하는 실수까지 정리해 보겠습니다!
