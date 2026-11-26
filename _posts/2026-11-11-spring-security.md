---
title: "[백엔드 여정] 11. Spring Security 기초: 요청을 보낸 사람이 누구인지 알아내기"
date: 2026-11-11 09:00:00 +0900
categories: [Backend, Spring]
tags: [spring-security, authentication, authorization, filter-chain, session, jwt, securitycontext]
---

## 👋 `Optional.of(1L)`의 정체를 밝힐 시간
지난 글에서 `AuditorAware`에 이런 코드를 넣었습니다.

```java
@Bean
public AuditorAware<Long> auditorProvider() {
    return () -> Optional.of(1L);   // 모든 데이터를 1번 회원이 만든 걸로...
}
```

누가 글을 쓰든 작성자가 1번 회원으로 찍히는 엉터리 코드죠. 진짜로 하려면 **"지금 이 요청을 보낸 사람이 누구인가"** 를 알아야 합니다. 이번 글에서는 그 질문에 답해 주는 **Spring Security**의 기본 구조를 정리합니다.

---

## 🔑 1. 인증과 인가
보안 이야기를 시작하면 두 단어가 계속 나옵니다. 헷갈리기 쉬우니 먼저 짚고 갈게요.

| | 인증 (Authentication) | 인가 (Authorization) |
| --- | --- | --- |
| 질문 | **너 누구야?** | **너 이거 해도 돼?** |
| 예시 | 아이디·비밀번호 로그인 | 관리자만 회원 삭제 가능 |
| 실패하면 | **401 Unauthorized** | **403 Forbidden** |
| 순서 | 먼저 | 인증 다음 |

식당으로 치면 **인증은 입구에서 예약자 이름 확인**, **인가는 "VIP룸은 VIP 손님만"** 입니다. 누군지 모르면 VIP인지도 판단할 수 없으니 순서는 항상 **인증 → 인가**예요.

> 401의 이름은 Unauthorized지만 실제 의미는 **"인증 안 됨"** 입니다. "로그인은 했는데 권한이 없음"은 **403**이에요. 5편에서 `ErrorCode`를 만들 때 이 둘을 구분해 두면 프론트엔드가 "로그인 페이지로 보낼지, 권한 없음 안내를 띄울지" 바로 판단할 수 있습니다.
{: .prompt-tip }

---

## 🧠 2. HTTP는 기억력이 없다
문제는 HTTP가 **무상태(stateless)** 라는 점입니다. 요청 하나하나가 남남이라, 방금 로그인한 사람이 다음 요청을 보내도 서버는 **처음 보는 사람**으로 대해요.

그래서 로그인에 성공하면 서버는 **"나 아까 로그인한 사람이야"를 증명할 무언가**를 클라이언트에게 쥐여 주고, 클라이언트는 매 요청마다 그걸 보여 줍니다. 그 "무언가"를 어떻게 만드느냐에 따라 두 방식으로 나뉩니다.

### 세션 방식: 서버가 기억하고, 번호표만 준다
```text
1. POST /login (id, pw)
2. 서버: 세션 저장소에 { abc123: 회원 1번 } 저장
3. 응답: Set-Cookie: JSESSIONID=abc123
4. 이후 요청: Cookie: JSESSIONID=abc123  → 서버가 저장소에서 찾아봄
```

**외투 보관소의 번호표**와 같습니다. 번호표엔 아무 정보가 없고, 진짜 정보(외투)는 서버가 들고 있어요.

### 토큰 방식(JWT): 정보를 담은 출입증을 준다
```text
1. POST /login (id, pw)
2. 서버: { sub: 1, role: USER, exp: ... } + 서명 → 토큰 발급
3. 응답: { "accessToken": "eyJhbGciOi..." }
4. 이후 요청: Authorization: Bearer eyJhbGciOi...  → 서버가 서명만 검증
```

**위조 방지 홀로그램이 붙은 출입증**입니다. 서버는 아무것도 기억하지 않고, 출입증의 **서명이 진짜인지만** 확인해요.

| | 세션 | 토큰(JWT) |
| --- | --- | --- |
| 상태 저장 | 서버 (세션 저장소) | 클라이언트 (토큰 자체) |
| 서버 여러 대 | 세션 공유 필요 (Redis 등) | 어느 서버든 검증 가능 |
| 강제 로그아웃 | 세션 삭제하면 끝 ✅ | 만료 전까지 유효 ❌ (별도 장치 필요) |
| 주 사용처 | 서버 렌더링 웹 | REST API, 모바일 앱 |
| 주의할 공격 | CSRF | 토큰 탈취 (XSS 등) |

> JWT의 내용(payload)은 **암호화가 아니라 Base64 인코딩**일 뿐입니다. 누구나 디코딩해서 읽을 수 있어요. 서명은 "위조되지 않았다"만 보장하니, **비밀번호 같은 민감 정보는 절대 넣지 마세요.**
{: .prompt-warning }

어느 방식이든 공통점은, **매 요청마다 "이 사람 누구지?"를 확인하는 단계가 Controller보다 앞에 있어야 한다**는 겁니다. 그 자리를 맡는 게 Spring Security예요.

---

## 🚪 3. 필터 체인: Controller 앞의 검문소 ⭐⭐가장 중요
1편에서 모든 요청은 **DispatcherServlet**을 거쳐 Controller로 간다고 했습니다. Spring Security는 그보다 **더 앞, 서블릿 필터** 단계에 자리를 잡습니다.

```text
클라이언트
   │
   ▼
[ 서블릿 필터들 ]
   └─ DelegatingFilterProxy
        └─ FilterChainProxy
             └─ SecurityFilterChain
                  ├─ SecurityContextHolderFilter   (이전 인증 정보 불러오기)
                  ├─ CsrfFilter                    (CSRF 토큰 검사)
                  ├─ UsernamePasswordAuthenticationFilter (폼 로그인 처리)
                  ├─ ... (BasicAuth, 익명 사용자 등)
                  ├─ ExceptionTranslationFilter    (인증/인가 예외 → 401/403)
                  └─ AuthorizationFilter           (이 URL 들어가도 돼?)
   │
   ▼
DispatcherServlet → Controller → Service → Repository
```

* **`DelegatingFilterProxy`**: 서블릿 컨테이너(톰캣)의 필터와 스프링 빈 세계를 잇는 다리. 필터는 톰캣이 관리하지만, 실제 일은 **스프링 빈(6편)** 에게 넘깁니다.
* **`FilterChainProxy`**: 요청 URL에 맞는 `SecurityFilterChain`을 골라 실행
* **`SecurityFilterChain`**: 실제 검문 필터들의 묶음. **우리가 설정하는 대상이 이것**입니다.

건물로 치면 Controller는 사무실이고, 필터 체인은 **1층 로비의 보안 게이트**입니다. 출입증 확인 → 짐 검사 → 층별 출입 권한 확인을 차례로 통과해야 엘리베이터(DispatcherServlet)를 탈 수 있어요. 게이트에서 막히면 **사무실까지 가지도 못합니다.**

> 필터는 **DispatcherServlet보다 앞**에 있기 때문에, 필터에서 터진 예외는 5편의 **`@RestControllerAdvice`가 잡지 못합니다.** 인증 실패 응답은 Security 쪽에서 따로 만들어 줘야 해요(6번에서 다룹니다).
{: .prompt-warning }

---

## 🗂️ 4. 인증 정보는 어디에 저장될까?
필터를 통과하며 인증에 성공하면, 그 결과는 **`SecurityContextHolder`** 에 담깁니다.

```text
SecurityContextHolder
   └─ SecurityContext
        └─ Authentication
             ├─ principal     누구인가 (보통 UserDetails)
             ├─ credentials   비밀번호 (인증 후엔 지워짐)
             └─ authorities   권한 목록 (ROLE_USER, ROLE_ADMIN ...)
```

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> roles = auth.getAuthorities();
```

* `SecurityContextHolder`는 기본적으로 **`ThreadLocal`** 에 저장합니다. 요청 하나는 스레드 하나가 처리하니, **같은 요청 안에서는 Controller든 Service든 어디서나** 꺼낼 수 있어요.
* 요청이 끝나면 비워지고, 다음 요청 때 세션(또는 토큰)에서 **다시 채워집니다.**

4편의 트랜잭션도 `ThreadLocal`로 커넥션을 공유했던 것 기억나시나요? 스프링은 **"요청 하나 동안만 유지되는 정보"** 를 이렇게 스레드에 묶어 두는 방식을 자주 씁니다.

---

## 🧑‍💻 5. 회원 정보로 로그인하기
starter를 추가하기만 해도 Security는 동작합니다.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
testImplementation 'org.springframework.security:spring-security-test'
```

서버를 띄워 보면 **모든 API가 막히고**, 로그에 임시 비밀번호가 찍힙니다.

```text
Using generated security password: 8e557245-73e2-4286-969a-ff57fe326336
```

아이디 `user`와 이 비밀번호로만 들어갈 수 있는 상태예요. 이제 이걸 **우리 DB의 회원**으로 바꿔 봅시다.

### ① 비밀번호는 반드시 해시로
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

```java
@Transactional
public Long join(MemberCreateRequest request) {
    String encoded = passwordEncoder.encode(request.getPassword());
    Member member = new Member(request.getEmail(), encoded, Role.USER);
    return memberRepository.save(member).getId();
}
```

```text
1234  →  $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
```

* **BCrypt**는 단방향 해시라 **원래 비밀번호로 되돌릴 수 없습니다.** DB가 털려도 비밀번호는 지켜져요.
* 같은 `1234`라도 매번 **다른 결과**가 나옵니다(salt). 그래서 비교는 `equals()`가 아니라 **`passwordEncoder.matches(raw, encoded)`** 로 해야 해요.

### ② `UserDetailsService`: Security에게 회원 찾는 법 알려 주기
Security는 우리 `Member` 엔티티를 모릅니다. **"아이디로 회원을 찾아서 Security가 아는 모양(`UserDetails`)으로 바꿔 주는"** 빈을 등록해야 해요.

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(String email) {
        Member member = memberRepository.findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException("회원 없음: " + email));
        return new CustomUserDetails(member);
    }
}
```

```java
@Getter
public class CustomUserDetails implements UserDetails {

    private final Long id;
    private final String email;
    private final String password;
    private final Collection<? extends GrantedAuthority> authorities;

    public CustomUserDetails(Member member) {
        this.id = member.getId();
        this.email = member.getEmail();
        this.password = member.getPassword();
        this.authorities = List.of(new SimpleGrantedAuthority("ROLE_" + member.getRole().name()));
    }

    @Override public String getUsername() { return email; }
    @Override public String getPassword() { return password; }
    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
}
```

* 엔티티를 그대로 들고 있지 않고 **필요한 값만 복사**했습니다. 인증 정보는 세션에 저장되기도 하는데, 엔티티를 넣으면 **영속성 컨텍스트 밖에서 지연 로딩(3편)** 을 건드리는 사고가 나기 쉬워요.
* 권한 이름 앞의 **`ROLE_`** 접두사는 Security의 약속입니다. `hasRole("ADMIN")`은 내부적으로 `ROLE_ADMIN`을 찾아요.

로그인 요청이 들어오면 Security는 이 흐름으로 인증합니다.

```text
UsernamePasswordAuthenticationFilter
   → AuthenticationManager
      → DaoAuthenticationProvider
         → UserDetailsService.loadUserByUsername(email)   ← 우리가 만든 것
         → PasswordEncoder.matches(입력 pw, DB pw)          ← 우리가 등록한 것
   → 성공: SecurityContextHolder에 Authentication 저장 (+ 세션에 보관)
```

우리는 **"회원 찾는 법"과 "비밀번호 비교하는 법"** 만 빈으로 알려 주고, 나머지 흐름은 Security가 다 해 줍니다. 6편의 IoC가 여기서도 그대로 보이죠.

---

## ⚙️ 6. `SecurityFilterChain` 설정하기
Spring Security 6(Spring Boot 3)부터는 **`SecurityFilterChain`을 빈으로 등록**하는 방식으로 설정합니다. (예전 글에서 보이는 `WebSecurityConfigurerAdapter`는 **삭제**됐어요.)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/members/join", "/login").permitAll()
                .requestMatchers(HttpMethod.GET, "/posts/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new CustomAuthenticationEntryPoint())  // 401
                .accessDeniedHandler(new CustomAccessDeniedHandler())            // 403
            );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### URL 권한 규칙
* **`permitAll()`**: 누구나 (회원가입, 로그인, 공개 조회)
* **`authenticated()`**: 로그인한 사람만
* **`hasRole("ADMIN")`**: 해당 권한이 있는 사람만
* 규칙은 **위에서부터 순서대로** 매칭됩니다. `anyRequest()`를 맨 위에 두면 아래 규칙이 전부 무시되니, **구체적인 것 → 넓은 것** 순서로 쓰세요.

### 401/403 응답을 우리 형식으로
3번에서 말했듯 필터의 예외는 `@RestControllerAdvice`까지 오지 않습니다. 대신 **`ExceptionTranslationFilter`** 가 예외를 잡아서 두 핸들러에게 넘겨요.

```java
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException e) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("""
                {"code":"AUTH_001","message":"로그인이 필요합니다."}
                """);
    }
}
```

`AccessDeniedHandler`도 같은 모양으로 403을 내려 주면, 5편에서 정한 **에러 응답 형식을 인증 실패에서도** 지킬 수 있습니다. (실무에서는 `ObjectMapper`로 5편의 `ErrorResponse`를 직렬화해서 쓰면 돼요.)

### 메서드 단위 권한: `@PreAuthorize`
URL 규칙만으로 부족할 땐 **메서드에 직접** 권한을 걸 수 있습니다. `@EnableMethodSecurity`가 켜 줘요.

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/members/{id}")
public void delete(@PathVariable Long id) { ... }
```

4편의 `@Transactional`처럼 **프록시(AOP)** 로 동작합니다. 그래서 같은 클래스 안에서 호출하면 적용되지 않는 함정도 똑같이 있어요.

> `@PreAuthorize`에서 터진 `AccessDeniedException`은 필터가 아니라 **Controller 단계**에서 발생합니다. 그래서 `@RestControllerAdvice`에 `@ExceptionHandler(Exception.class)` 같은 넓은 핸들러가 있으면 **그쪽이 먼저 잡아서 500으로** 바꿔 버릴 수 있어요. `AccessDeniedException`을 따로 처리하거나 다시 던지도록 확인하세요.
{: .prompt-warning }

---

## 🙋 7. Controller와 Auditing에서 "나" 꺼내기

### Controller: `@AuthenticationPrincipal`
```java
@GetMapping("/members/me")
public MemberResponse me(@AuthenticationPrincipal CustomUserDetails user) {
    return memberService.findById(user.getId());
}
```

`SecurityContextHolder`를 직접 뒤질 필요 없이, **로그인한 사용자를 파라미터로 바로** 받을 수 있습니다. 클라이언트가 보낸 `memberId`를 믿는 대신 **인증 정보에서 꺼낸 id**를 쓰니, 남의 정보를 조회하는 요청도 막을 수 있어요.

### 10편의 숙제 해결: 진짜 `AuditorAware`
```java
@Component
public class SecurityAuditorAware implements AuditorAware<Long> {

    @Override
    public Optional<Long> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(Authentication::isAuthenticated)
                .filter(auth -> !(auth instanceof AnonymousAuthenticationToken))
                .map(Authentication::getPrincipal)
                .filter(CustomUserDetails.class::isInstance)
                .map(principal -> ((CustomUserDetails) principal).getId());
    }
}
```

* 로그인하지 않은 요청은 Security가 **익명 사용자(`AnonymousAuthenticationToken`)** 를 넣어 두기 때문에, `isAuthenticated()`만으로는 걸러지지 않습니다. 익명 사용자도 따로 걸러 줘야 해요.
* 회원가입처럼 로그인 전에 생기는 데이터는 `Optional.empty()` → `createdBy`가 `null`로 저장됩니다.

이제 `Optional.of(1L)` 대신, **글을 쓴 사람의 id가 진짜로 `createdBy`에 찍힙니다.** 🎉

---

## 🧪 8. 테스트에서 Security 다루기
Security를 추가하는 순간, 7편에서 만든 `@WebMvcTest`가 **401이나 403으로 우르르 실패**합니다. 테스트도 이제 "누가 요청하는지"를 알려 줘야 해요.

```java
@WebMvcTest(MemberController.class)
@Import(SecurityConfig.class)
class MemberControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean MemberService memberService;

    @Test
    void 로그인하지_않으면_401() throws Exception {
        mockMvc.perform(get("/members/me"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void 일반_회원은_관리자_API에_403() throws Exception {
        mockMvc.perform(delete("/admin/members/1").with(csrf()))
                .andExpect(status().isForbidden());
    }
}
```

* **`@WithMockUser`**: 가짜 로그인 사용자를 `SecurityContext`에 넣어 줍니다. `@AuthenticationPrincipal CustomUserDetails`처럼 **직접 만든 타입**이 필요하면 `@WithUserDetails`나 커스텀 어노테이션(`@WithSecurityContext`)을 써요.
* **`with(csrf())`**: 기본 설정에서는 POST·PUT·DELETE 요청에 **CSRF 토큰**이 없으면 403이 납니다. 테스트에서 토큰을 넣어 주는 도구예요.
* 10편처럼 `@WebMvcTest`는 직접 만든 `@Configuration`을 스캔하지 않으니, **내 Security 설정으로 테스트하려면 `@Import`** 가 필요합니다.

> **CSRF**는 "사용자가 로그인된 브라우저로 다른 사이트가 몰래 요청을 보내는" 공격입니다. **쿠키가 자동으로 실려 가는 세션 방식**에서 위험하기 때문에 기본으로 켜져 있어요. 다음 글의 JWT처럼 토큰을 `Authorization` 헤더에 직접 실어 보내는 방식은 브라우저가 자동으로 붙여 주지 않으니, 보통 CSRF 보호를 끕니다.
{: .prompt-info }

---

## 🧭 9. 한 장으로 정리
| 개념 | 핵심 |
| --- | --- |
| 인증 / 인가 | **누구냐(401)** / **해도 되냐(403)** — 인증이 먼저 |
| 세션 | 서버가 기억, 클라이언트는 `JSESSIONID` 번호표 |
| JWT | 서명된 토큰을 클라이언트가 보관, 서버는 검증만 (payload는 공개!) |
| 필터 체인 | **DispatcherServlet 앞**에서 검사 → `@RestControllerAdvice`가 못 잡음 |
| `SecurityContextHolder` | 인증 결과를 **`ThreadLocal`** 에 보관 |
| 비밀번호 | **`BCryptPasswordEncoder`**, 비교는 `matches()` |
| 회원 연결 | **`UserDetailsService`** 가 엔티티 → `UserDetails` 변환 |
| 설정 | **`SecurityFilterChain` 빈** (Adapter 상속 방식은 삭제됨) |
| 401 / 403 응답 | **`AuthenticationEntryPoint`** / **`AccessDeniedHandler`** |
| 현재 사용자 | **`@AuthenticationPrincipal`**, Auditing은 `AuditorAware`에서 꺼내기 |
| 테스트 | **`@WithMockUser`**, **`with(csrf())`**, `@Import(SecurityConfig.class)` |

---

## ✍️ 10. 스스로 점검해 보기
1. 인증과 인가의 차이는? 각각 실패하면 어떤 상태 코드를 내려야 하는가?
2. HTTP가 무상태라는 건 무슨 뜻이고, 로그인 상태는 어떻게 유지하는가?
3. 세션 방식과 JWT 방식의 장단점은?
4. Spring Security 필터에서 발생한 예외를 `@RestControllerAdvice`가 잡지 못하는 이유는?
5. `SecurityContextHolder`는 인증 정보를 어디에, 얼마 동안 보관하는가?
6. 비밀번호를 `equals()`가 아니라 `matches()`로 비교해야 하는 이유는?
7. `UserDetailsService`는 인증 흐름에서 어떤 역할을 하는가?
8. `AuditorAware`에서 익명 사용자를 따로 걸러야 하는 이유는?

---

## 🗓️ 다음 글 예고: JWT 로그인 직접 구현하기
이번 글에서는 Security의 **뼈대**를 세우고, 로그인은 기본 폼 로그인에 맡겼습니다. 하지만 우리가 만드는 건 화면 없는 **REST API**고, 모바일 앱이나 프론트엔드가 따로 붙는 구조라 **JWT**가 더 잘 어울리죠. 다음 글에서는 **JWT를 발급하고, 매 요청마다 검증하는 필터를 직접 만들어** 필터 체인에 끼워 넣어 보겠습니다. Access Token이 탈취되면 어떡하지? 라는 질문에 답하는 **Refresh Token**까지 정리할게요!
