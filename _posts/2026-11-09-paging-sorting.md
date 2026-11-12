---
title: "[백엔드 여정] 09. 페이징과 정렬: 10만 건을 20건씩 나눠 보여주기"
date: 2026-11-09 09:00:00 +0900
categories: [Backend, JPA]
tags: [jpa, spring-data-jpa, paging, pageable, slice, sort, no-offset]
---

## 👋 `findAll()`은 언젠가 터진다
지금까지 회원 목록은 `memberRepository.findAll()`로 가져왔습니다. 회원이 10명일 땐 아무 문제 없죠. 그런데 서비스가 잘 돼서 **회원이 10만 명**이 되면?

* DB는 10만 행을 읽고
* 서버는 10만 개의 엔티티를 메모리(영속성 컨텍스트)에 올리고
* 클라이언트는 몇 MB짜리 JSON을 받아서 화면에 20개만 보여줍니다.

지난 글에서 **컬렉션 페치 조인에 페이징을 걸면 메모리 페이징이 일어나 위험하다**고 했는데, 정작 페이징 자체는 제대로 다룬 적이 없었습니다. 이번 글에서는 **필요한 만큼만 잘라서 가져오는 법**, 페이징과 정렬을 정리합니다.

---

## 📐 1. SQL로 보는 페이징
페이징의 정체는 SQL의 **`LIMIT`(몇 개)** 와 **`OFFSET`(몇 개 건너뛰고)** 입니다.

```sql
-- 3페이지(한 페이지 20개) = 40개 건너뛰고 20개
SELECT * FROM member
ORDER BY id DESC
LIMIT 20 OFFSET 40;

-- 전체 페이지 수를 알려면 전체 개수도 필요
SELECT COUNT(*) FROM member;
```

직접 짜려면 `OFFSET` 계산, `COUNT` 쿼리, "다음 페이지가 있나?" 판단까지 매번 해야 합니다. 그리고 DB마다 문법도 조금씩 달라요(MySQL `LIMIT`, Oracle `ROWNUM`/`FETCH FIRST`...). **Spring Data JPA는 이걸 전부 대신 해 줍니다.**

---

## 📄 2. `Pageable`: 페이지 요청서
Repository 메서드에 **`Pageable`** 파라미터만 추가하면 페이징이 적용됩니다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    Page<Member> findByName(String name, Pageable pageable);
}
```

`findAll(Pageable)`은 `JpaRepository`에 이미 들어 있어서 따로 선언할 필요도 없어요.

```java
PageRequest pageRequest = PageRequest.of(2, 20, Sort.by(Sort.Direction.DESC, "id"));
Page<Member> page = memberRepository.findAll(pageRequest);
```

* **`PageRequest.of(page, size, sort)`**: `Pageable`의 구현체. "몇 번째 페이지를, 몇 개씩, 어떤 순서로"
* **page는 0부터 시작합니다.** `PageRequest.of(2, 20)`은 **3번째 페이지**예요. 1페이지를 생각하고 `1`을 넣으면 두 번째 페이지가 나오니 주의!

식당으로 치면 **코스 요리 서빙**입니다. 20인분을 한꺼번에 식탁에 쏟아붓는 게 아니라 "3번째 접시를, 20개씩, 큰 것부터" 내오라는 **주문서**가 `Pageable`이에요.

---

## ⚖️ 3. `Page` vs `Slice` vs `List` ⭐⭐가장 중요
반환 타입에 따라 **나가는 쿼리와 얻는 정보가 달라집니다.**

### `Page`: 전체 개수까지 아는 페이지
```java
Page<Member> page = memberRepository.findAll(PageRequest.of(0, 20));

page.getContent();        // 이번 페이지의 데이터 20개
page.getTotalElements();  // 전체 데이터 수 (예: 100,000)
page.getTotalPages();     // 전체 페이지 수 (예: 5,000)
page.getNumber();         // 현재 페이지 번호 (0)
page.hasNext();           // 다음 페이지 있나?
```

```sql
select m.* from member m limit 20 offset 0;   -- 데이터 조회
select count(m.id) from member m;             -- 전체 개수 조회 (추가 쿼리!)
```

**`1 2 3 4 5 ... 5000` 같은 페이지 번호 버튼**이 필요한 게시판형 화면에 씁니다.

### `Slice`: 다음이 있는지만 아는 페이지
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    Slice<Member> findSliceBy(Pageable pageable);
}
```

```sql
select m.* from member m limit 21 offset 0;   -- 20개 요청인데 21개 조회!
-- count 쿼리 없음
```

`Slice`는 **요청한 개수보다 1개 더(`size + 1`)** 가져와서, 21번째가 있으면 `hasNext() == true`로 판단합니다. 전체 개수는 모르지만 **count 쿼리가 없어서 가볍죠.** **"더보기" 버튼, 무한 스크롤** 화면에 딱 맞습니다.

### `List`: 그냥 잘라서 데이터만
```java
List<Member> findTop20ByOrderByIdDesc();
List<Member> findByName(String name, Pageable pageable);
```
페이지 정보 없이 **데이터만** 필요할 때 씁니다. "최근 가입 회원 5명" 같은 위젯이 대표적이에요.

| 반환 타입 | count 쿼리 | 아는 것 | 어울리는 화면 |
| --- | --- | --- | --- |
| `Page` | ✅ 나감 | 전체 개수, 전체 페이지 수 | 페이지 번호 게시판 |
| `Slice` | ❌ 없음 | 다음 페이지 유무 | 더보기, 무한 스크롤 |
| `List` | ❌ 없음 | 데이터만 | 상위 N개 위젯 |

> 데이터가 많아질수록 **`COUNT(*)`는 생각보다 비싼 쿼리**입니다. 전체 페이지 수가 꼭 필요한 화면이 아니라면 `Slice`부터 고려하세요.
{: .prompt-tip }

---

## 🔃 4. 정렬: `Sort`
정렬은 `Sort`로 지정합니다. 여러 기준을 이어 붙일 수도 있어요.

```java
Sort sort = Sort.by(Sort.Direction.DESC, "id");

Sort multi = Sort.by("name").ascending()
        .and(Sort.by("id").descending());   // 이름 오름차순, 같으면 id 내림차순

PageRequest pageRequest = PageRequest.of(0, 20, multi);
```

`"name"`, `"id"`는 **DB 컬럼명이 아니라 엔티티의 필드명**입니다(2편). JPQL처럼 객체 기준으로 생각하면 돼요.

> **정렬 기준은 유일하게 만드세요.** `name`으로만 정렬하면 이름이 같은 회원들의 순서는 DB 마음대로라서, 페이지를 넘길 때 **같은 회원이 두 번 나오거나 누락**될 수 있습니다. 마지막에 `id` 같은 유일한 값을 덧붙이면 순서가 항상 고정돼요.
{: .prompt-warning }

---

## 🌐 5. Controller에서 `Pageable` 받기
Spring Data는 **쿼리 파라미터를 `Pageable`로 자동 변환**해 줍니다.

```java
@GetMapping("/members")
public Page<MemberResponse> getMembers(
        @PageableDefault(size = 20, sort = "id", direction = Sort.Direction.DESC)
        Pageable pageable) {
    return memberService.findMembers(pageable);
}
```

```text
GET /members?page=2&size=20&sort=name,asc&sort=id,desc
```

* **`page`, `size`, `sort`** 파라미터를 읽어서 `Pageable`을 만들어 줍니다.
* 파라미터가 없으면 **`@PageableDefault`** 의 값이 쓰입니다. (없으면 기본 `size=20`)

### Service: 엔티티 말고 DTO로 바꿔서
```java
@Transactional(readOnly = true)
public Page<MemberResponse> findMembers(Pageable pageable) {
    return memberRepository.findAll(pageable)
            .map(MemberResponse::from);   // Page 정보는 그대로, 내용만 DTO로
}
```

`Page.map()`을 쓰면 전체 개수·페이지 번호 같은 정보는 유지하고 **내용물만 DTO로** 바꿀 수 있습니다. 1편부터 강조한 **"엔티티를 API로 직접 내보내지 않기"** 를 페이징에서도 지키는 방법이에요.

### ⚠️ 꼭 막아 둘 것
```yaml
spring:
  data:
    web:
      pageable:
        max-page-size: 100          # size=100000 요청 방지 (기본값 2000)
        one-indexed-parameters: true # 클라이언트는 page=1부터 보내도 되게
```

* **`max-page-size`**: 누군가 `?size=1000000`을 보내면 페이징하는 의미가 없어집니다. 상한을 꼭 낮춰 두세요.
* **`one-indexed-parameters`**: 사람에게 익숙한 **1페이지부터** 받고 싶을 때. 단, 응답의 `number`는 여전히 0부터라 헷갈릴 수 있으니 팀에서 기준을 정하세요.
* **`sort` 파라미터**: 클라이언트가 아무 필드나 정렬 기준으로 보낼 수 있습니다. 인덱스 없는 컬럼으로 정렬하면 느려지니, 운영에서는 **허용할 정렬 기준을 정해 두는 것**이 안전해요.

> `Page` 객체(`PageImpl`)를 그대로 JSON으로 내보내면 응답 구조가 Spring 내부 구현에 묶여서, Spring Data 3.3부터는 **경고 로그**가 찍힙니다. `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)`를 설정하거나, **직접 만든 페이지 응답 DTO**(`content`, `page`, `size`, `totalElements`...)로 감싸서 내보내는 것을 권장합니다.
{: .prompt-info }

---

## 🔢 6. count 쿼리 최적화
`Page`는 데이터 조회와 별개로 **count 쿼리를 자동 생성**합니다. 그런데 데이터 조회 쿼리에 조인이 있으면 count에도 그대로 조인이 붙어요.

```java
@Query("select m from Member m left join m.team t")
Page<Member> findAllWithTeam(Pageable pageable);
```

```sql
select count(m.id) from member m left join team t on ...;   -- 조인이 필요 없는데도!
```

회원 수를 세는 데 팀은 필요 없죠. 이럴 땐 **`countQuery`를 따로 지정**합니다.

```java
@Query(value = "select m from Member m join fetch m.team",
       countQuery = "select count(m) from Member m")
Page<Member> findAllWithTeam(Pageable pageable);
```

* 8편의 **ToOne 페치 조인 + 페이징 조합**이 바로 이 모양입니다. 행이 늘어나지 않으니 `LIMIT`이 정상적으로 걸리고, count는 가볍게 따로 셉니다.
* 페치 조인 쿼리는 count로 자동 변환할 때 문제가 생길 수 있어서 **`countQuery`를 명시하는 게 안전**해요.

> Spring Data는 **마지막 페이지이거나 첫 페이지인데 데이터가 `size`보다 적으면 count 쿼리를 생략**하는 최적화도 알아서 해 줍니다. "1페이지에 3개뿐이면 전체도 3개"니까요.
{: .prompt-info }

---

## 🚀 7. OFFSET 페이징의 한계와 커서 페이징
`OFFSET` 방식에는 근본적인 약점이 있습니다.

```sql
SELECT * FROM member ORDER BY id DESC LIMIT 20 OFFSET 100000;
```

DB는 **100,020개를 읽은 뒤 앞의 100,000개를 버립니다.** 뒤 페이지로 갈수록 점점 느려져요. 게다가 사용자가 1페이지를 보는 사이 새 회원이 가입하면, 2페이지에서 **1페이지 마지막 회원이 또 나오는** 중복도 생깁니다.

### 커서(No-Offset) 페이징
"몇 개 건너뛰기" 대신 **"마지막으로 본 것 다음부터"** 를 조건으로 거는 방식입니다.

```sql
SELECT * FROM member
WHERE id < 98765          -- 직전 페이지의 마지막 id
ORDER BY id DESC
LIMIT 20;
```

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    Slice<Member> findByIdLessThan(Long lastId, Pageable pageable);
}

// 사용: 정렬은 Pageable로, page는 항상 0
memberRepository.findByIdLessThan(lastId, PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "id")));
```

```text
GET /members?lastId=98765&size=20
```

* `id`에는 **인덱스(PK)** 가 있으니 몇 페이지 뒤든 **항상 빠릅니다.**
* 중간에 데이터가 추가돼도 **중복·누락이 없어요.**
* 대신 **"37페이지로 바로 이동"은 불가능**합니다. 그래서 **무한 스크롤**에 주로 쓰여요.

| | OFFSET 페이징 | 커서 페이징 |
| --- | --- | --- |
| 방식 | `LIMIT n OFFSET m` | `WHERE id < 마지막id LIMIT n` |
| 뒤 페이지 성능 | 점점 느려짐 | 항상 일정 |
| 특정 페이지 이동 | ✅ | ❌ |
| 데이터 추가 시 | 중복/누락 가능 | 안정적 |
| 어울리는 화면 | 페이지 번호 게시판 | 무한 스크롤, 피드 |

> Spring Data 3.1부터는 커서 방식 페이징을 위한 **Scroll API**(`Window`, `ScrollPosition.keyset()`)도 제공합니다. 원리는 위와 같으니, 직접 구현해 보고 나서 살펴보면 이해가 쉬워요.
{: .prompt-tip }

---

## 🧭 8. 한 장으로 정리
| 상황 | 이렇게 하자 |
| --- | --- |
| 목록 조회 | `findAll()` ❌ → **`Pageable`** 로 필요한 만큼만 |
| 페이지 번호 | **0부터 시작** (`one-indexed-parameters`로 변경 가능) |
| 페이지 번호 게시판 | **`Page`** (count 쿼리 O) |
| 더보기 / 무한 스크롤 | **`Slice`** (size+1 조회, count 쿼리 X) |
| 정렬 | **`Sort`**, 마지막에 **유일한 값(id)** 덧붙이기 |
| API 응답 | `page.map(MemberResponse::from)`으로 **DTO 변환** |
| 안전장치 | **`max-page-size`** 상한, 정렬 기준 제한 |
| 조인 있는 페이징 | **`countQuery` 분리**, ToMany는 배치 사이즈(8편) |
| 대용량 / 무한 스크롤 | **커서(No-Offset) 페이징** |

---

## ✍️ 9. 스스로 점검해 보기
1. 전체 목록을 `findAll()`로 가져오면 어떤 문제가 생기는가?
2. `PageRequest.of(2, 20)`은 몇 번째 데이터부터 몇 개를 가져오는가?
3. `Page`와 `Slice`는 실행되는 쿼리와 제공하는 정보가 어떻게 다른가?
4. `Slice`는 count 쿼리 없이 다음 페이지 유무를 어떻게 알아내는가?
5. 정렬 기준에 유일한 값을 덧붙여야 하는 이유는?
6. Controller에서 `Pageable`을 받을 때 막아 두어야 할 것은?
7. `countQuery`를 따로 지정하면 좋은 경우는?
8. OFFSET 페이징의 한계와 커서 페이징의 장단점은?

---

## 🗓️ 다음 글 예고: JPA Auditing
이번 글에서 정렬 기준으로 계속 `id`만 썼습니다. 실제 서비스라면 **"최신 가입순"**, **"최근 수정순"** 처럼 **생성일·수정일**로 정렬하는 경우가 훨씬 많죠. 그런데 그 `createdAt`, `updatedAt`은 누가 채워야 할까요? 엔티티를 저장할 때마다 `LocalDateTime.now()`를 직접 넣어야 할까요? 다음 글에서는 **JPA Auditing**과 **`@MappedSuperclass`로 공통 필드를 한 곳에서 관리하는 법**을 정리해 보겠습니다!
