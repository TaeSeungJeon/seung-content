---
title: "백오피스 구현 정리! 권한·역할·레벨링·JPA·React 이렇게 만들었다"
date: "2026-06-27"
description: "직접 구현한 백오피스의 핵심 기술 회고. 메뉴 RBAC 권한 설계, 역할 관리, 권한 레벨링, JPA 소프트삭제·복합키, JWT 인가와 React 권한 게이팅까지."
category: "Dev"
---

# 들어가며

백오피스를 하나 구현했다. 회원·권한·게시판·감사 로그·공통코드까지 다루는 관리자 시스템이다.

이번 글은 그 안에서 직접 구현했던 핵심 기술들을 정리하는 회고다.

특히 제일 공들였던 **권한, 역할 관리, 레벨링, JPA 연동, React 인가** 다섯 가지를 코드와 함께 풀어본다.

스택은 이렇다.

- **백엔드** : Spring Boot 3.4.4 + Java 21 + JPA(Hibernate) + PostgreSQL 16
- **프론트** : React 19 + Vite + TypeScript + TanStack React Query + React Router 7
- **인증/인가** : JWT(Stateless) + BCrypt + 메뉴 RBAC

---

# 1. 권한 설계 — `PERM:모듈:액션`

가장 먼저 잡은 건 권한 모델이다. **3계층 RBAC** 로 구현했다.

```
사용자(User) ── N:M ──> 역할(Role) ── N:M ──> 권한(Permission)
```

사용자는 역할을 갖고, 역할은 권한을 묶고, 실제 인가 판정은 가장 작은 단위인 **권한 코드**로 한다.

권한 코드는 `PERM:모듈:액션` 형태로 통일했다.

```
PERM:MEMBER:READ      회원 조회
PERM:MEMBER:UPDATE    회원 수정
PERM:NOTICE:CREATE    공지 등록
PERM:ROLE:ASSIGN      역할 부여
```

액션은 6종으로 고정했다.

```java
public enum ActionCode { READ, CREATE, UPDATE, DELETE, MANAGE, ASSIGN }
```

이 권한 코드 하나가 백엔드 인가부터 프론트 버튼 노출까지 전 구간을 관통하는 단일 키가 된다.

## N:M 매핑은 복합키로

사용자-역할, 역할-권한 사이 매핑 테이블은 복합키를 `@EmbeddedId` + `@MapsId` 로 구현했다.

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

`@MapsId` 를 쓰면 복합키 필드와 연관관계 FK가 같은 컬럼을 공유한다. 키 중복 없이 깔끔하게 떨어진다.

## 권한은 세 갈래로 합산된다

구현하면서 제일 신경 쓴 부분. 권한이 **역할로만 오는 게 아니다.**

역할 / 사용자 / 조직 세 경로를 모두 합산하도록 `UserAuthorityResolver` 를 만들었다.

```
1) 역할 기반   : USER_ROLE_MAP → ROLE_AUTHRT_MAP → 권한코드
2) 사용자 직접 : 특정 사용자에게 메뉴 접근권 직접 부여
3) 부서 부여   : 부서 단위로 메뉴 접근권 부여 (스코프 적용)
```

특히 부서 부여는 **조직 트리를 타고 올라가며** 판정하는데, 스코프를 세 가지로 구현했다.

```java
boolean match = switch (md.getScope()) {
    case SELF     -> grantedDeptId.equals(selfDeptId);       // 그 부서 본인만
    case SUBTREE  -> true;                                   // 본인 + 모든 하위
    case SUB_ONLY -> !grantedDeptId.equals(selfDeptId);      // 하위만 (본인 제외)
};
```

"마케팅팀 + 하위 조직 전부에게 부여" 같은 권한이 한 줄로 표현된다.

그리고 N+1을 조심했다. 역할마다 권한을 하나씩 조회하면 쿼리가 폭발하니까, **역할 ID를 모아서 단일 `IN` 쿼리**로 한 번에 가져온다.

```java
List<Long> activeRoleIds = userRoles.stream()
        .filter(ur -> ur.getRole().getStatus() == ACTIVE)
        .map(ur -> ur.getRole().getId())
        .toList();

rolePermissionRepository.findByRoleIdIn(activeRoleIds);   // N+1 방지
```

---

# 2. 역할 관리 — 시스템 역할은 못 건드리게

역할(Role)은 CRUD가 되지만, 아무 역할이나 막 지우게 두면 안 된다.

`master_admin`, `admin`, `manager`, `user` 같은 **핵심 역할 코드**는 백엔드/프론트 곳곳에서 하드코딩으로 참조되기 때문에, 코드 자체를 단일 상수로 묶고 불변으로 보호했다.

```java
public final class RoleCode {
    public static final String MASTERADMIN = "ROLE_MASTERADMIN";
    public static final String ADMIN       = "ROLE_ADMIN";
    public static final String MANAGER     = "ROLE_MANAGER";
    public static final String USER         = "ROLE_USER";
}
```

역할 코드를 바꾸거나 시스템 역할을 삭제하는 동작은 최상위 권한자만 가능하게 가드를 걸었다.

```java
/** 시스템 역할 생성·수정·삭제 등 최상위 전용 동작 */
public void requireMasterAdmin() {
    if (!isMasterAdmin(SecurityUtils.currentUserSn())) {
        throw new AccessDeniedException("이 작업은 master_admin만 가능합니다.");
    }
}
```

또 역할 엔티티엔 `listed` 플래그를 둬서, '권한 관리에서 등재한 역할만 역할 관리 목록에 노출'되게 했다. 목록에서 빼도 역할이 실제로 삭제되는 건 아니다. 운영 중 실수로 역할이 날아가는 걸 막는 안전장치다.

---

# 3. 레벨링 — 역할에 위계를 준다

권한 코드가 '할 수 있는 행동'을 정의한다면, **레벨**은 '얼마나 높은 사람인가'를 정의한다.

역할을 4단계 등급으로 줄세웠다.

```
user(0) < manager(1) < admin(2) < master_admin(3)
```

핵심은 **한 사용자가 역할을 여러 개 가질 때, 가장 높은 등급을 대표값으로** 쓴다는 것.

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

데이터가 `ROLE_ADMIN` 으로 오든, 영문 `admin` 이든, 한글 `관리자` 든 전부 한 레벨로 정규화했다. 표기가 사방에서 제각각으로 들어와도 한 군데서 통일되니까 관리가 편하다.

이 레벨링은 **메뉴/라우트 가드의 see-all 바이패스**에 쓰인다. 메뉴 관리에서 동적 생성된 권한은 관리자 역할에 자동 매핑이 안 될 수 있어서, 사이드바·라우트 가드는 순수 권한 비교 대신 `admin` 이상이면 전부 보여주도록 레벨을 본다.

```tsx
const isSuper = ['admin', 'master_admin'].includes(highestPermLevel(user.roles));
```

권한(액션)과 레벨(위계)을 분리하니까 "권한은 많지만 등급은 낮은 운영자" 같은 케이스도 자연스럽게 표현됐다.

---

# 4. JPA 연동 — 공통 베이스로 깔아둔 패턴

JPA로 도메인을 짜면서 실무에 꼭 필요한 패턴들을 공통으로 깔았다.

## 소프트 삭제 (Soft Delete)

백오피스는 데이터를 진짜 `DELETE` 하는 일이 거의 없다. 감사 추적 때문에 '지웠다는 사실'까지 남겨야 한다.

모든 엔티티에 `DEL_AT` 플래그를 두고, Hibernate `@SQLRestriction` 으로 조회 시 자동 필터링했다.

```java
@Entity
@Table(name = "ROLE_INFO")
@SQLRestriction("DEL_AT = 'N'")   // 모든 조회에 자동으로 붙는다
public class Role extends BaseEntity { ... }
```

이제 어떤 쿼리를 날려도 삭제된 행은 알아서 빠진다. 매번 `where del_at = 'N'` 을 손으로 안 붙여도 된다.

## Y/N ↔ boolean 자동 변환

공공/레거시 표준 DB는 boolean을 `'Y'`/`'N'` 문자로 저장한다. `AttributeConverter` 로 자바 `boolean` 과 자동 매핑했다.

```java
@Convert(converter = YnConverter.class)
@Column(name = "USE_AT", length = 1)
private boolean visible = true;     // DB엔 'Y'/'N', 코드에선 true/false
```

## BaseEntity + Auditing

생성자/수정자/생성·수정일시는 `BaseEntity` 에 모아두고 JPA Auditing(`AuditorAware`)으로 자동 주입했다. 누가 언제 만들고 고쳤는지가 공짜로 따라온다.

DB 컬럼/테이블 명명은 **행정안전부 공통표준용어**를 따랐다 (`ROLE_SN`, `AUTHRT_CD`, `STTUS_CD` ...). 공공/엔터프라이즈 환경에 그대로 꽂을 수 있게 한 선택이다.

---

# 5. JWT 인가 + React 권한 게이팅

인증은 세션 없는 **Stateless JWT** 방식이다.

## 토큰에 권한을 굽는다

로그인에 성공하면 그 사용자의 권한 코드 목록을 통째로 토큰 `authorities` 클레임에 **구워서(bake)** 넣는다. 매 요청마다 DB에서 권한을 다시 조회할 필요가 없다.

```java
// 로그인 성공 시
List<String> authorities = authorityResolver.resolve(user.getId());
String token = jwtTokenProvider.createToken(
        user.getId(), user.getEmployeeId(), sessionId, authorities);
```

## 인가는 어노테이션 한 줄로

권한 체크는 커스텀 어노테이션으로 추상화했다. 안에 `@PreAuthorize` 가 들어있다.

```java
@Target({METHOD, TYPE})
@PreAuthorize("hasAuthority('PERM:MEMBER:READ')")
public @interface RequireMemberRead {}
```

컨트롤러엔 의미만 남는다.

```java
@RequireMemberRead
@GetMapping("/api/members")
public ApiResponse<...> list() { ... }
```

권한 코드를 문자열로 일일이 쓰는 대신 **타입 안전한 어노테이션**으로 박으니 오타로 권한이 새는 일이 없다.

## 토큰만 믿지는 않는다

Stateless JWT의 함정 하나. 한 번 발급되면 만료 전까진 유효하다. 로그아웃해도 토큰은 살아있다.

그래서 필터에서 **세션 DB를 한 번 더 검증**해서, 로그아웃·강제 종료된 세션의 토큰 재사용을 막았다.

```java
// 로그아웃/만료된 세션의 JWT 재사용 차단
if (!userSessionRepository.existsBySessionIdAndStatus(sessionId, ACTIVE)) {
    audit.authz(userSn, ..., AuthResult.EXPIRED);   // 감사 로그까지
    chain.doFilter(request, response);
    return;
}
```

로그인 보안 기본기도 챙겼다.

- **계정 잠금** : 비밀번호 5회 틀리면 30분 잠금
- **사용자 열거 방지** : 아이디가 틀리든 비번이 틀리든 동일 메시지
- **BCrypt** 해시

## React — 백엔드와 같은 기준으로 화면을 그린다

프론트의 핵심은 **"백엔드가 구워준 권한을 그대로 디코드해서 화면을 그린다"** 는 것.

```ts
/** JWT authorities 클레임 디코드. 토큰 없거나 위조면 빈 배열(fail-closed) */
export function getAuthorities(): string[] {
  const token = getToken();
  if (!token) return [];
  const payload = JSON.parse(atob(token.split('.')[1]));
  return Array.isArray(payload.authorities) ? payload.authorities : [];
}

export function hasPerm(perm?: string): boolean {
  if (!perm) return true;
  return getAuthorities().includes(perm);
}
```

포인트는 **fail-closed**. 토큰이 없거나 깨졌으면 무조건 빈 권한으로 본다. 애매하면 막는다.

덕분에 컴포넌트는 깔끔해진다.

```tsx
{hasPerm('PERM:MEMBER:CREATE') && <button>회원 추가</button>}
```

물론 이건 **UX용**이다. 버튼을 숨겼다고 보안이 되는 게 아니니까, 진짜 차단은 백엔드 `@PreAuthorize` 가 한다. **프론트는 "보여줄지 말지", 백엔드는 "허용할지 말지".** 역할이 명확히 갈린다.

나머지는 요즘 React 표준대로다.

- **TanStack React Query** : 서버 상태 캐싱·동기화
- **React Router 7** : 라우트 가드에서 레벨/권한 체크
- **공유 HTTP 클라이언트** : Bearer 토큰 자동 첨부 + 401이면 로그인으로 리다이렉트

---

# 마치며

이번 구현에서 제일 크게 배운 건 **"권한(액션)과 레벨(위계)은 다른 축"** 이라는 거였다.

처음엔 admin이면 다 되고 user면 조회만, 이렇게 뭉뚱그렸는데 실제 백오피스엔
"권한은 많은데 등급은 낮은 운영자", "등급은 높은데 특정 메뉴만 보는 사람" 같은 케이스가 수두룩했다.

그래서 `PERM:모듈:액션` 으로 행동을 쪼개고, 레벨로 위계를 따로 세우고,
토큰 하나에 다 구워서 백엔드와 프론트가 같은 기준으로 판정하게 만들었다.

다음 글에선 감사 로그(Audit)를 AOP로 자동 수집한 얘기를 풀어볼 예정이다.
