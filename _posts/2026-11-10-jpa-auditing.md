---
title: "[백엔드 여정] 10. JPA Auditing: 생성일·수정일은 엔티티가 알아서"
date: 2026-11-10 09:00:00 +0900
categories: [Backend, JPA]
tags: [jpa, auditing, mappedsuperclass, createddate, lastmodifieddate, auditoraware, base-entity]
---

## 👋 `createdAt`은 누가 채워야 할까?
지난 글에서 정렬 기준으로 계속 `id`만 썼습니다. 하지만 실제 화면은 **"최신 가입순"**, **"최근 수정순"** 이 훨씬 많죠. 그러려면 모든 테이블에 이런 컬럼이 필요합니다.

* `created_at`: 언제 만들어졌나
* `updated_at`: 언제 마지막으로 바뀌었나
* (가끔) `created_by`, `updated_by`: **누가** 만들고 바꿨나

이 값들은 비즈니스 로직과는 상관없지만, 장애가 났을 때 **"이 데이터 언제부터 이랬어?"** 를 추적하는 유일한 단서가 됩니다. 이번 글에서는 이 공통 필드를 **한 번만 정의하고, 자동으로 채우는 법**을 정리합니다.

---

## 😵 1. 직접 넣으면 생기는 일
가장 단순한 방법은 Service에서 직접 넣는 겁니다.

```java
@Transactional
public Long join(MemberCreateRequest request) {
    Member member = new Member(request.getName());
    member.setCreatedAt(LocalDateTime.now());
    member.setUpdatedAt(LocalDateTime.now());
    return memberRepository.save(member).getId();
}

@Transactional
public void changeName(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name);
    member.setUpdatedAt(LocalDateTime.now());   // 이거 까먹으면?
}
```

* **엔티티가 20개면** 똑같은 필드 선언과 `now()` 호출이 20벌씩 생깁니다.
* 수정 로직이 늘어날수록 **`setUpdatedAt()` 한 줄을 빠뜨릴** 확률도 같이 늘어나요. 빠뜨려도 에러가 안 나니 아무도 모릅니다.
* 2편에서 "엔티티에 무분별한 setter를 열지 말자"고 했는데, 시간 때문에 setter를 열어야 하죠.

식당으로 치면 **주문서마다 직원이 손으로 시각을 적는** 상황입니다. 바쁘면 까먹고, 사람마다 적는 형식도 달라요. 해결책은 **주문서가 들어가는 순간 자동으로 찍히는 타임스탬프 기계**입니다.

---

## 🪝 2. JPA 기본 기능: 엔티티 생명주기 콜백
JPA는 엔티티가 저장·수정되기 **직전에 호출되는 콜백**을 제공합니다.

```java
@Entity
public class Member {

    @Id @GeneratedValue
    private Long id;
    private String name;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist   // persist(INSERT) 직전
    void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate    // 변경 감지로 UPDATE가 나가기 직전
    void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

| 콜백 | 호출 시점 |
| --- | --- |
| `@PrePersist` / `@PostPersist` | `persist()` 직전 / 직후 |
| `@PreUpdate` / `@PostUpdate` | flush 때 UPDATE 직전 / 직후 |
| `@PreRemove` / `@PostRemove` | 삭제 직전 / 직후 |
| `@PostLoad` | DB에서 조회해 엔티티를 만든 직후 |

이제 Service에서 `now()`를 부를 필요가 없습니다. 그런데 **이 코드를 엔티티 20개에 복사**해야 한다는 문제는 그대로예요. 공통 부분을 한 곳으로 빼 봅시다.

---

## 🧱 3. `@MappedSuperclass`: 필드만 물려주는 부모 ⭐
공통 필드를 부모 클래스로 올리고 **`@MappedSuperclass`** 를 붙이면, 자식 엔티티가 그 필드를 **자기 컬럼처럼** 가져갑니다.

```java
@Getter
@MappedSuperclass
public abstract class BaseTimeEntity {

    @Column(updatable = false)
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist
    void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

```java
@Entity
public class Member extends BaseTimeEntity { ... }

@Entity
public class Team extends BaseTimeEntity { ... }
```

```sql
create table member (
    id bigint not null,
    name varchar(255),
    created_at timestamp(6),   -- 부모의 필드가
    updated_at timestamp(6),   -- 자식 테이블의 컬럼으로!
    primary key (id)
);
```

* **`BaseTimeEntity` 테이블은 생기지 않습니다.** 매핑 정보(컬럼)만 물려줄 뿐이에요.
* 그래서 엔티티가 아니고, `em.find(BaseTimeEntity.class, ...)`나 `select b from BaseTimeEntity b` 같은 **조회도 불가능**합니다.
* 직접 생성할 일이 없으니 **`abstract`** 로 만드는 게 의도를 잘 드러내요.
* **`@Column(updatable = false)`**: 생성일은 한 번 정해지면 바뀌면 안 되니, UPDATE 문에서 아예 빠지도록 막아 둡니다.

> **`@MappedSuperclass` vs `@Entity` 상속**: 부모에 `@Entity`를 붙이고 `@Inheritance`를 쓰면 **부모 테이블까지 생기는 "진짜 상속 매핑"** 이 됩니다(`Item` → `Book`, `Album` 같은 경우). 생성일·수정일처럼 **"관계는 없고 필드만 공유"** 하고 싶을 땐 `@MappedSuperclass`가 맞아요.
{: .prompt-info }

---

## 🌱 4. Spring Data JPA Auditing
콜백을 직접 짜도 되지만, Spring Data JPA는 이걸 **어노테이션 몇 개**로 끝내 주는 **Auditing** 기능을 제공합니다.

### ① 기능 켜기: `@EnableJpaAuditing`
```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
}
```

> 메인 클래스(`@SpringBootApplication`)에 바로 붙이는 예제가 많은데, **별도 설정 클래스로 빼는 걸 추천**합니다. 이유는 7번 "테스트에서 터지는 문제"에서 볼게요.
{: .prompt-tip }

### ② 부모 클래스에 리스너와 어노테이션
```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

* **`@EntityListeners(AuditingEntityListener.class)`**: 2번에서 직접 짠 `@PrePersist`/`@PreUpdate` 콜백을 **Spring이 만들어 둔 리스너**가 대신 해 줍니다.
* **`@CreatedDate`**: 처음 저장될 때 시각을 채움
* **`@LastModifiedDate`**: 저장·수정될 때마다 시각을 갱신

이제 엔티티는 `extends BaseTimeEntity` 한 줄이면 끝입니다. Service 코드에서는 시간 관련 코드가 **완전히 사라져요.**

```java
@Transactional
public void changeName(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName(name);   // updatedAt은 커밋 시점에 자동 갱신
}
```

4편에서 본 흐름 그대로입니다. 트랜잭션 커밋 → flush → **변경 감지(2편)** 가 UPDATE를 만들기 직전에 리스너가 `updatedAt`을 채워 넣어요.

> 처음 저장할 때 `updatedAt`도 `createdAt`과 **같은 값으로** 채워집니다(`@EnableJpaAuditing(modifyOnCreate = true)`가 기본값). "아직 수정된 적 없음"을 `null`로 표현하고 싶다면 `false`로 바꾸면 되지만, `updatedAt` 기준 정렬에서 `null`이 섞이면 귀찮아져서 보통 기본값을 그대로 씁니다.
{: .prompt-info }

---

## 🙋 5. "누가" 바꿨는지도: `@CreatedBy`, `@LastModifiedBy`
시간뿐 아니라 **작성자·수정자**도 같은 방식으로 채울 수 있습니다. 다만 "현재 사용자가 누구인지"는 Spring이 모르니 **알려 주는 빈**이 필요해요.

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity extends BaseTimeEntity {

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @LastModifiedBy
    private Long updatedBy;
}
```

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<Long> auditorProvider() {
        // 지금은 임시 값. 실제로는 로그인한 사용자의 id를 꺼내 옵니다.
        return () -> Optional.of(1L);
    }
}
```

* **`AuditorAware<T>`**: `getCurrentAuditor()`가 돌려준 값으로 `@CreatedBy`/`@LastModifiedBy`를 채웁니다. 6편에서 본 것처럼 **빈으로 등록만 하면** Spring이 찾아서 씁니다.
* 로그인하지 않은 요청(회원가입, 배치 작업 등)에서는 `Optional.empty()`를 돌려주면 `null`로 저장됩니다.
* 실무에서는 보통 **Spring Security의 `SecurityContextHolder`** 에서 로그인 사용자를 꺼내 옵니다. 이 부분은 인증을 다룰 때 다시 연결할게요.

### 시간만 필요한 엔티티 vs 작성자도 필요한 엔티티
```text
BaseTimeEntity          ← createdAt, updatedAt
   └─ BaseEntity        ← + createdBy, updatedBy
```

* 로그 테이블이나 **로그인 없이 생기는 데이터**는 `BaseTimeEntity`만
* 게시글, 주문처럼 **"누가"가 중요한 데이터**는 `BaseEntity`

이렇게 두 단계로 나눠 두면 필요한 만큼만 골라서 상속할 수 있어 편합니다.

---

## 🛠️ 6. Hibernate 방식도 있다
Spring Data 없이 Hibernate만으로도 비슷하게 할 수 있습니다.

```java
@CreationTimestamp
private LocalDateTime createdAt;

@UpdateTimestamp
private LocalDateTime updatedAt;
```

| | Spring Data JPA Auditing | Hibernate |
| --- | --- | --- |
| 어노테이션 | `@CreatedDate`, `@LastModifiedDate` | `@CreationTimestamp`, `@UpdateTimestamp` |
| 설정 | `@EnableJpaAuditing` + `@EntityListeners` 필요 | 없음 |
| 작성자 기록 | ✅ `@CreatedBy` + `AuditorAware` | ❌ 직접 구현 |
| 의존 | Spring Data | Hibernate 구현체 |

시간만 필요하면 둘 다 괜찮습니다. **작성자까지 함께 관리**하려면 Spring Data 방식으로 통일하는 게 깔끔해요.

---

## ⚠️ 7. 자주 만나는 함정

### ① `@WebMvcTest`가 갑자기 터진다
`@EnableJpaAuditing`을 메인 클래스에 붙이면, 7편에서 만든 **Controller 슬라이스 테스트**가 이런 에러를 냅니다.

```text
java.lang.IllegalArgumentException: JPA metamodel must not be empty!
```

`@WebMvcTest`도 메인 클래스의 설정을 읽는데, 웹 계층 빈만 올리는 테스트라 **JPA 관련 빈이 하나도 없어요.** 그런데 메인 클래스의 `@EnableJpaAuditing`은 JPA를 요구하니 충돌하는 겁니다. **설정 클래스를 따로 빼 두면** `@WebMvcTest`는 일반 `@Configuration`을 스캔하지 않아서 이 문제가 사라집니다.

### ② 반대로 `@DataJpaTest`에선 값이 `null`
설정 클래스를 분리하면, 이번엔 `@DataJpaTest`가 그 설정을 **올리지 않아서** `createdAt`이 `null`로 나옵니다. 필요한 테스트에만 직접 가져오면 돼요.

```java
@DataJpaTest
@Import(JpaAuditingConfig.class)
class MemberRepositoryTest {

    @Autowired MemberRepository memberRepository;
    @Autowired EntityManager em;

    @Test
    void 저장하면_생성일과_수정일이_자동으로_채워진다() {
        Member saved = memberRepository.save(new Member("지원"));

        assertThat(saved.getCreatedAt()).isNotNull();
        assertThat(saved.getUpdatedAt()).isEqualTo(saved.getCreatedAt());
    }

    @Test
    void 수정하면_수정일만_바뀐다() {
        Member member = memberRepository.save(new Member("지원"));
        em.flush();
        LocalDateTime createdAt = member.getCreatedAt();

        member.changeName("지원2");
        em.flush();   // 변경 감지 → UPDATE → @LastModifiedDate 갱신

        assertThat(member.getCreatedAt()).isEqualTo(createdAt);
        assertThat(member.getUpdatedAt()).isAfter(createdAt);
    }
}
```

7편에서 말한 것처럼 테스트는 끝나면 롤백되니, **`em.flush()`로 UPDATE를 강제로 내보내야** 리스너가 동작하는 걸 볼 수 있습니다.

> 두 번째 테스트는 저장과 수정이 거의 동시에 일어나서, 환경에 따라 시각이 같게 찍혀 **가끔 실패할 수 있습니다.** 정확히 검증하려면 Auditing에 시간을 공급하는 `DateTimeProvider`를 빈으로 등록하고(`@EnableJpaAuditing(dateTimeProviderRef = "...")`) 테스트에서 고정된 시각을 넣어 주면 됩니다.
{: .prompt-warning }

### ③ 벌크 수정은 `updatedAt`을 안 바꾼다
```java
@Modifying
@Query("update Member m set m.status = 'DORMANT' where m.lastLoginAt < :date")
int bulkDormant(@Param("date") LocalDateTime date);
```

Auditing은 **엔티티 리스너**라서 **영속성 컨텍스트를 거치는 저장·변경 감지**에서만 동작합니다. JPQL 벌크 연산은 영속성 컨텍스트를 건너뛰고 DB에 바로 SQL을 쏘기 때문에 **`updatedAt`이 그대로**예요. 필요하면 쿼리에 직접 넣어 줍니다.

```java
@Query("update Member m set m.status = 'DORMANT', m.updatedAt = :now where m.lastLoginAt < :date")
```

### ④ 바뀐 게 없으면 UPDATE도 없다
같은 이름으로 `changeName("지원")`을 호출하면 스냅샷과 값이 같아서 **변경 감지가 UPDATE를 만들지 않고**, 당연히 `updatedAt`도 그대로입니다. "저장 버튼을 누른 시각"이 아니라 **"데이터가 실제로 바뀐 시각"** 이라는 점을 기억해 두세요.

---

## 🔗 8. 9편과 연결: 생성일로 정렬하기
이제 "최신 가입순"을 만들 수 있습니다.

```java
@GetMapping("/members")
public Page<MemberResponse> getMembers(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable) {
    return memberService.findMembers(pageable);
}
```

* 필드명은 **부모 클래스에 있어도** 자식 엔티티의 필드처럼 그대로 쓸 수 있어요.
* 9편의 주의점 그대로, `createdAt`은 **같은 시각에 여러 건이 생길 수 있어 유일하지 않습니다.** 페이지 넘김 때 중복·누락을 막으려면 `id`를 덧붙이세요.

```java
Sort sort = Sort.by(Sort.Direction.DESC, "createdAt")
        .and(Sort.by(Sort.Direction.DESC, "id"));
```

* 데이터가 많다면 정렬 컬럼에 **인덱스**도 필요합니다. `@Table(indexes = @Index(columnList = "created_at"))`로 선언하거나 DDL로 직접 만들어 두세요.

> `LocalDateTime`에는 **시간대 정보가 없습니다.** 서버가 UTC로 돌고 있으면 한국 시간보다 9시간 이른 값이 저장돼요. 서버·DB·JVM의 타임존(`TZ`, `spring.jpa.properties.hibernate.jdbc.time_zone`)을 맞추거나, 여러 시간대를 다뤄야 한다면 `Instant`/`OffsetDateTime`을 고려하세요. 배포할 때 한 번은 꼭 확인해야 하는 부분입니다.
{: .prompt-warning }

---

## 🧭 9. 한 장으로 정리
| 상황 | 이렇게 하자 |
| --- | --- |
| 생성일·수정일 | Service에서 `now()` ❌ → **Auditing으로 자동** |
| 공통 필드 공유 | **`@MappedSuperclass`** 추상 클래스 (테이블 X, 조회 X) |
| 기능 켜기 | **`@EnableJpaAuditing`** 은 **별도 `@Configuration`** 에 |
| 리스너 등록 | **`@EntityListeners(AuditingEntityListener.class)`** |
| 생성일 | **`@CreatedDate`** + `@Column(updatable = false)` |
| 수정일 | **`@LastModifiedDate`** (처음엔 생성일과 같은 값) |
| 작성자·수정자 | **`@CreatedBy`/`@LastModifiedBy`** + `AuditorAware` 빈 |
| 계층 분리 | `BaseTimeEntity` ← `BaseEntity` |
| `@DataJpaTest` | **`@Import(JpaAuditingConfig.class)`** |
| 벌크 수정 | Auditing 동작 X → **쿼리에 직접 `updatedAt`** |
| 생성일 정렬 | **`createdAt` + `id`**, 인덱스, 타임존 확인 |

---

## ✍️ 10. 스스로 점검해 보기
1. 생성일·수정일을 Service에서 직접 넣으면 어떤 문제가 생기는가?
2. `@PrePersist`와 `@PreUpdate`는 각각 언제 호출되는가?
3. `@MappedSuperclass`와 `@Entity` 상속 매핑은 무엇이 다른가?
4. Spring Data JPA Auditing을 쓰려면 어떤 설정 세 가지가 필요한가?
5. 생성일 필드에 `@Column(updatable = false)`를 붙이는 이유는?
6. `@CreatedBy`는 "현재 사용자"를 어떻게 알아내는가?
7. `@EnableJpaAuditing`을 메인 클래스에 두면 어떤 테스트에서 무슨 문제가 생기는가?
8. JPQL 벌크 수정에서 `updatedAt`이 갱신되지 않는 이유는?

---

## 🗓️ 다음 글 예고: Spring Security와 인증
이번 글에서 `AuditorAware`에 **`Optional.of(1L)`** 이라는 임시 값을 넣었습니다. 진짜로는 **"지금 요청을 보낸 사람이 누구인지"** 를 알아야 하죠. 그런데 HTTP는 요청마다 남남인데, 서버는 어떻게 같은 사람인 걸 알아볼까요? 다음 글에서는 **세션과 토큰 인증의 차이**, 그리고 **Spring Security의 필터 체인**이 요청을 어떻게 검사하는지 정리해 보겠습니다!
