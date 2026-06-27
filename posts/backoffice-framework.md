---
title: "백오피스, 매번 다시 만들지 말자 — 권한·역할·레벨링 다 박은 재사용 프레임워크 만들기"
date: "2026-06-27"
description: "Spring Boot 3 + React 19로 만든 범용 백오피스 프레임워크. 메뉴 RBAC, 권한 레벨링, JPA 소프트삭제·복합키, JWT 인가까지 핵심 기술을 정리한다."
category: "Dev"
---

# 또 백오피스를 만든다고?

회사 프로젝트든 사이드 프로젝트든, 새 서비스를 시작하면 항상 똑같은 걸 또 만든다.

회원 관리, 권한 관리, 게시판, 감사 로그, 공통 코드, 시스템 설정...

서비스마다 도메인은 다른데 이 **백오피스 70%는 매번 복붙**이다.

그래서 마음먹고 만들기 시작한 게 `code.solution.backoffice.v1` 이다.

> 한 번 잘 만들고, 모든 프로젝트가 그냥 import 한다.

목표는 단순하다. 신규 프로젝트 백오피스 셋업을 **2~3주 → 30분**으로 줄이는 것.

이번 글에서는 그중에서도 제일 공들인 **권한/역할 관리, 레벨링, JPA 연동, React 인가**를 정리해본다.

---

# 기술 스택부터

| 영역 | 기술 |
|---|---|
| Backend | Spring Boot 3.4.4 + Java 21 |
| Frontend | React 19 + Vite 8 + TypeScript |
| DB | PostgreSQL 16 + JPA(Hibernate) |
| 상태관리 | TanStack React Query + React Router 7 |
| 인증 | JWT(Stateless) + BCrypt |
| 인가 | 메뉴 RBAC (R/C/U/D/X) |

프론트는 React Query로 서버 상태를, Tailwind v4로 스타일을 잡았다.

백엔드는 전통 MVC + 동기 JPA. WebFlux 같은 거 안 쓴다. 단순하고 검증된 길로 간다.

---

# 1. 권한/역할 관리 — RBAC를 제대로

권한 모델은 교과서적인 **3계층 RBAC** 로 잡았다.

```
사용자(User) ── N:M ──> 역할(Role) ── N:M ──> 권한(Permission)
```

- **사용자**는 여러 **역할**을 가질 수 있고
- **역할**은 여러 **권한**을 묶고 있고
- 실제 인가는 가장 작은 단위인 **권한 코드**로 판정한다

엔티티로 풀면 이렇게 된다. N:M 사이엔 매핑 테이블이 들어가고, 복합키는 `@EmbeddedId` + `@MapsId` 로 처리했다.

```java
@Entity
@Table(name = "USER_ROLE_MAP")
public class UserRole extends BaseEntity {

    @EmbeddedId
    private UserRoleId id;            // (userSn, roleSn) 복합키

    @MapsId("userSn")
    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "USER_SN")
    private User user;

    @MapsId("roleSn")
    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "ROLE_SN")
    private Role role;

    @Column(name = "GRANTER_ID")
    private String grantedBy;         // 누가 부여했는지 추적
}
```

권한 코드는 `PERM:모듈:액션` 형태로 통일했다.

```
PERM:MEMBER:READ      회원 조회
PERM:MEMBER:UPDATE    회원 수정
PERM:NOTICE:CREATE    공지 등록
PERM:ROLE:ASSIGN      역할 부여
```

액션은 6종으로 고정 — `READ / CREATE / UPDATE / DELETE / MANAGE / ASSIGN`.

이 권한 코드 하나가 백엔드 인가, 프론트 버튼 노출까지 **전 구간을 관통하는 단일 키**가 된다.

## 권한은 세 갈래로 합산된다

재밌는 건 권한이 **역할로만 오는 게 아니라는** 점이다.

전자정부 표준 모델처럼 '권한 주체 = 역할 / 사용자 / 조직' 세 경로를 모두 합산한다.

```
1) 역할 기반   : USER_ROLE_MAP → ROLE_AUTHRT_MAP → 권한코드
2) 사용자 직접 : 특정 사용자에게 메뉴 접근권 직접 부여
3) 부서 부여   : 부서 단위로 메뉴 접근권 부여 (스코프 적용)
```

이걸 `UserAuthorityResolver` 가 한 방에 모아서 정렬된 중복없는 집합으로 돌려준다.

특히 부서 부여는 **조직 트리**를 타고 올라가는데, 스코프가 세 가지다.

```java
boolean match = switch (md.getScope()) {
    case SELF     -> grantedDeptId.equals(selfDeptId);       // 그 부서 본인만
    case SUBTREE  -> true;                                   // 본인 + 모든 하위
    case SUB_ONLY -> !grantedDeptId.equals(selfDeptId);      // 하위만 (본인 제외)
};
```

"마케팅팀 + 하위 조직 전부" 같은 부여가 한 줄로 표현된다.

그리고 여기서 N+1을 조심했다. 역할 → 권한 조회를 루프 돌면서 하나씩 긁으면 쿼리가 폭발하니까, **역할 ID 목록을 모아 단일 `IN` 쿼리**로 한 번에 가져온다.

```java
List<Long> activeRoleIds = userRoles.stream()
        .filter(ur -> ur.getRole().getStatus() == ACTIVE)
        .map(ur -> ur.getRole().getId())
        .toList();

// N+1 방지 — 한 방에
rolePermissionRepository.findByRoleIdIn(activeRoleIds);
```

---

# 2. 레벨링 — 역할에 위계를 준다

권한 코드가 '할 수 있는 행동'을 정의한다면, **레벨**은 '얼마나 높은 사람인가'를 정의한다.

역할을 4단계 등급으로 줄세웠다.

```
user(0) < manager(1) < admin(2) < master_admin(3)
```

핵심은 **한 사용자가 여러 역할을 가질 때, 가장 높은 등급을 대표값으로** 쓴다는 것.

```ts
const RANK = { user: 0, manager: 1, admin: 2, master_admin: 3 };

/** 역할 코드 목록 → 가장 높은 권한 레벨 */
export function highestPermLevel(codes?: string[]): PermLevel {
  if (!codes?.length) return 'user';
  return codes.reduce((acc, c) => {
    const lv = toPermLevel(c);
    return RANK[lv] > RANK[acc] ? lv : acc;
  }, 'user');
}
```

데이터가 `ROLE_ADMIN` 으로 오든, 영문 `admin` 으로 오든, 한글 `관리자` 로 오든 전부 한 레벨로 정규화한다. 표기가 사방에서 제각각으로 들어와도 한 군데서 통일되니까 마음이 편하다.

이 레벨링이 두 군데서 일한다.

**1) 화면 게이팅의 see-all 바이패스.** 메뉴 관리에서 동적 생성된 메뉴 권한은 관리자 역할에 자동 매핑이 안 될 수 있다. 그래서 사이드바·라우트 가드는 순수 권한 비교 대신, `admin` 이상이면 다 보여주는 식으로 레벨을 본다.

**2) 시스템 역할 보호.** `master_admin`, `admin` 같은 핵심 역할은 함부로 못 건드리게 막아야 한다. 그래서 백엔드에 `AdminGuard` 라는 가드를 따로 뒀다.

```java
/** 시스템 역할 생성·수정·삭제 등 최상위 전용 동작 */
public void requireMasterAdmin() {
    if (!isMasterAdmin(SecurityUtils.currentUserSn())) {
        throw new AccessDeniedException("이 작업은 master_admin만 가능합니다.");
    }
}
```

권한(액션) 따로, 레벨(위계) 따로. 이 둘을 분리하니까 "권한은 많지만 등급은 낮은 운영자" 같은 케이스도 자연스럽게 표현됐다.

---

# 3. JPA 연동 — 엔터프라이즈 패턴 몇 가지

JPA를 쓰면서 실무에서 꼭 필요한 패턴 몇 개를 공통 베이스로 깔았다.

## 소프트 삭제 (Soft Delete)

백오피스에서 데이터를 진짜 `DELETE` 하는 일은 거의 없다. 감사 추적 때문에 지웠다는 사실까지 남겨야 한다.

그래서 모든 엔티티에 `DEL_AT` 플래그를 두고, Hibernate `@SQLRestriction` 으로 조회 시 자동 필터링했다.

```java
@Entity
@Table(name = "ROLE_INFO")
@SQLRestriction("DEL_AT = 'N'")   // 모든 조회에 자동으로 붙는다
public class Role extends BaseEntity { ... }
```

이제 어떤 쿼리를 날려도 삭제된 행은 알아서 빠진다. 개발자가 매번 `where del_at = 'N'` 안 붙여도 된다.

## Y/N ↔ boolean 변환

레거시/공공 표준 DB는 boolean을 `'Y'`/`'N'` 문자로 저장하는 경우가 많다. `AttributeConverter` 로 자바 `boolean` 과 자동 매핑했다.

```java
@Convert(converter = YnConverter.class)
@Column(name = "USE_AT", length = 1)
private boolean visible = true;     // DB엔 'Y'/'N', 코드에선 true/false
```

## BaseEntity + Auditing

생성자/수정자/생성일시는 매번 손으로 넣을 게 아니다. `BaseEntity` 에 모아두고 JPA Auditing(`AuditorAware`)으로 자동 주입했다. 누가 언제 만들고 고쳤는지가 공짜로 따라온다.

여기에 DB 컬럼/테이블 명명은 **행정안전부 공통표준용어**를 따랐다 (`ROLE_SN`, `AUTHRT_CD`, `STTUS_CD` ...). 공공/엔터프라이즈 환경에 그대로 꽂을 수 있게.

---

# 4. JWT 인가 — 토큰에 권한을 굽는다

인증은 세션 없는 **Stateless JWT** 방식.

로그인에 성공하면, 그 사용자의 권한 코드 목록을 통째로 토큰 `authorities` 클레임에 **구워서(bake)** 넣는다.

```java
// 로그인 성공 시
List<String> authorities = authorityResolver.resolve(user.getId());
String token = jwtTokenProvider.createToken(
        user.getId(), user.getEmployeeId(), sessionId, authorities);
```

이러면 매 요청마다 DB에서 권한을 다시 조회할 필요가 없다. 토큰만 까보면 된다.

## 인가는 어노테이션 한 줄로

권한 체크는 커스텀 어노테이션으로 추상화했다. 컨트롤러에 의미가 바로 읽힌다.

```java
@RequireMemberRead          // 안에 @PreAuthorize("hasAuthority('PERM:MEMBER:READ')")
@GetMapping("/api/members")
public ApiResponse<...> list() { ... }
```

`@RequireMemberRead`, `@RequireRoleAssign` ... 이런 식으로 권한 코드를 일일이 문자열로 쓰는 대신 **타입 안전한 어노테이션**으로 박았다. 오타로 권한이 새는 일이 없다.

## 토큰만 믿지는 않는다

여기서 한 가지 함정. Stateless JWT는 한 번 발급되면 만료 전까지 유효하다. 로그아웃해도 토큰 자체는 살아있다.

그래서 필터에서 **세션 DB를 한 번 더 검증**한다. 로그아웃·강제 종료된 세션의 토큰 재사용을 막는 장치다.

```java
// 로그아웃/만료된 세션의 JWT 재사용 차단
if (!userSessionRepository.existsBySessionIdAndStatus(sessionId, ACTIVE)) {
    audit.authz(userSn, ..., AuthResult.EXPIRED);   // 감사 로그까지
    chain.doFilter(request, response);
    return;
}
```

이 외에도 로그인 보안은 기본기를 챙겼다.

- **계정 잠금** : 비밀번호 5회 틀리면 30분 잠금
- **사용자 열거 방지** : 아이디가 틀리든 비번이 틀리든 동일 메시지
- **BCrypt** 해시 + 비밀번호 정책

---

# 5. React — 권한이 화면을 그린다

프론트의 핵심 아이디어는 **"백엔드와 똑같은 기준으로 화면을 그린다"** 는 것.

백엔드가 토큰에 구워준 `authorities` 를 프론트에서 디코드해서, 메뉴·라우트·버튼 노출을 결정한다.

```ts
/** JWT authorities 클레임 디코드. 토큰 없거나 위조면 빈 배열(fail-closed) */
export function getAuthorities(): string[] {
  const token = getToken();
  if (!token) return [];
  const payload = JSON.parse(atob(token.split('.')[1]));
  return Array.isArray(payload.authorities) ? payload.authorities : [];
}

/** 이 권한 있어? */
export function hasPerm(perm?: string): boolean {
  if (!perm) return true;
  return getAuthorities().includes(perm);
}
```

포인트는 **fail-closed**. 토큰이 없거나 깨졌으면 무조건 빈 권한으로 본다. 애매하면 막는다.

그래서 컴포넌트는 이렇게 깔끔해진다.

```tsx
{hasPerm('PERM:MEMBER:CREATE') && <button>회원 추가</button>}
```

물론 이건 어디까지나 **UX용**이다. 화면에서 버튼을 숨겼다고 보안이 되는 게 아니니까, 진짜 차단은 위에서 본 백엔드 `@PreAuthorize` 가 한다. 프론트는 "보여줄지 말지", 백엔드는 "허용할지 말지". 역할이 명확하게 갈린다.

나머지는 요즘 React 표준대로다.

- **TanStack React Query** : 서버 상태 캐싱·동기화
- **React Router 7** : 라우트 가드 (`PrivateRoute` 에서 레벨/권한 체크)
- **공유 HTTP 클라이언트** : Bearer 토큰 자동 첨부 + 401이면 로그인으로 리다이렉트
- **AuthContext** : 로그인 사용자/역할 전역 관리

---

# 마치며

이번에 권한 시스템을 처음부터 다시 설계하면서 제일 크게 배운 건,

**"권한(액션)과 레벨(위계)은 다른 축"** 이라는 거였다.

처음엔 그냥 admin이면 다 되고 user면 조회만 되고... 이렇게 뭉뚱그렸는데,

실제 백오피스는 "권한은 많은데 등급은 낮은 운영자", "등급은 높은데 특정 메뉴만 보는 임원" 같은 케이스가 수두룩했다.

그래서 `PERM:모듈:액션` 으로 행동을 쪼개고, 레벨로 위계를 따로 세우고,

토큰 하나에 다 구워서 백/프론트가 같은 기준으로 판정하게 만들었다.

아직 갈 길이 멀다. SSO, MFA, 감사 로그 retention, CLI(`npx create-codesolution-app`) 까지가 로드맵에 있다.

목표는 변함없다 — **한 번 잘 만들고, 모든 프로젝트가 그냥 가져다 쓴다.** 🔫

다음 글에선 감사 로그(Audit)를 AOP로 자동 수집한 얘기를 풀어볼 예정이다.
