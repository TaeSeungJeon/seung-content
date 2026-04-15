---
title: "Spring Boot + React + GitHub API 이 조합 맛있잖아!! Seung Blog 소개! "
date: "2026-04-12"
description: "GitHub API를 활용한 서버리스 아키텍처 기반의 풀스택 개인 포트폴리오 플레이그라운드 블로그 등장. 두둥탁!"
category: "Dev"
---

# 프로젝트를 시작한 이유

백엔드 개발자를 목표로 공부하면서 단순히 강의를 듣고 예제를 따라 치는 것만으로는 한 없이 부족하다고 느꼈다. 

실제로 기획부터 설계, 구현, 배포까지 전체 사이클을 경험해봐야 진짜 실력이 쌓인다고 생각했고, 

그 결과물을 포트폴리오로도 활용할 수 있고 가지고 놀 수 있는 재밌는 프로젝트가 만들고 싶었다. 

그러던 중 한 프로젝트를 우연히 접하게되었는데, 개츠비라는 프레임워크를 구경을 하다가 나도 

이런 프로젝트 하나 만들고 싶다는 생각이 들었다. 하지만 개츠비 프로젝트는 나의 개발 방향과는 조금 거리가 있었다. 

React 정적 사이트를 다룰 수 있는 점에 끌렸지만 너무 front쪽 으로만 쏠려있는 프로젝트라고 생각했기 때문인데 

그래서 그 프로젝트를 참고해서 내 입맛에 맞게 나에게 필요한 스킬업 프로젝트로 기술 전환, 기획을 하고싶었다. 

초보 개발자인 내 관점에선 굳이? 오합지졸? 일지 모르겠지만 '이거 재밌겠는데?'라는 생각이 들었고, 

나에게 필요한 학습내용이라고 생각했기 때문에 곧 바로 프로젝트에 착수하기로 결정했다.

기존의 블로그 서비스(티스토리, 벨로그 등)를 그냥 쓰는 것이 아닌 내가 직접 만들어서 

Spring Boot, REST API 설계, OAuth 인증, JWT, CORS, 예외처리 등 백엔드 핵심 가지고 놀면서 

개념들을 실무 흐름에 맞게 적용해보는 것, 만들어진 프로젝트에 내 블로그를 작성, 보관하는것을 목표로 트리거를 당겼다!!

# 🔫 렛스고!!! 

---

# 프로젝트 목표

1. Spring Boot 기반 REST API 직접 설계 및 구현
2. GitHub OAuth 2.0 + JWT 인증 흐름 이해 및 적용
3. DB 없이 GitHub API를 활용한 서버리스 아키텍처 설계
4. React + TypeScript 프론트엔드 직접 구현하기
5. Oracle Cloud 인프라 직접 세팅 및 배포 경험 쌓기
6. 실제 서비스로 운영 가능한 수준의 완성도 끌어 올리기

---

# 기술 스택 및 채택

## Backend
### Spring Boot + Java

백엔드 개발자를 목표로 하고 있기 때문에 Java와 Spring Boot를 선택했다. 

이전에 MVC패턴을 활용한 Servlet을 학습한 경험이 있어서 Spring의 동작 원리를 

이해한 상태에서 Spring Boot를 적용할 수 있었다. 

Spring Boot는 설정을 최소화하면서도 실무에서 가장 많이 사용되는 

프레임워크이기 때문에 포트폴리오로서의 가치도 높다고 판단했다.

### Spring Security + JWT

로그인 기능 구현 시 세션 방식 대신 JWT를 선택했다. 

프론트(GitHub Pages)와 백엔드(Oracle Cloud)가 완전히 분리된 구조를 채택했고, 

세션은 서버에 상태를 저장해야 하므로 분리된 환경에서 관리가 복잡해질 수 있으므로, 

서버가 상태를 저장하지 않는 Stateless 방식인 JWT가 이 아키텍처에 더 적합하다고 생각했다. 

### GitHub API (DB 대신 API를 채택했다!)

블로그 글 저장소로 DB 대신 GitHub API를 채택한 데는 네 가지 정도의 이유를 말할 수 있겠다. 

첫째, 블로그 글은 버전 관리가 중요하고, 

둘째, 마크다운 형식으로 작성되기 때문에 GitHub 레포지토리와 궁합이 좋다고 생각했다. 

셋째, 자주 변경되지 않는 데이터이기 때문에 굳이 DB를 운영할 필요가 없었습니다. 

GitHub Contents API로 글을 읽어오고, GitHub Issues API로 방명록을 관리하는 구조로 설계했다. 

마지막 네번째, 오래 사용할 목적인 만큼 개발자라면 대부분 사용하는 GitHub를 통해서 

불필요한 개인정보 노출을 막고 내 블로그가 누구나 친근하게 접근 할 수 있는 공간이었으면 좋겠다고 생각했다.

---
## Frontend

### React + TypeScript + Vite

백엔드 개발자도 자신이 만든 API가 실제로 동작하는 화면을 직접 구현할 수 있어야 로직을 눈으로 확인할 수 있고, 

프론트엔드 팀과 원활하게 협업할 수 있다고 생각한다. 

React는 현재 프론트엔드 생태계에서 가장 많이 사용되는 라이브러리이고, 

TypeScript를 함께 사용하면 타입 안정성을 보장할 수 있어서 선택하게 됐다. 

Vite는 빠른 개발 환경을 제공하고 GitHub Pages 배포에도 적합하다고 생각했다.

### Tailwind CSS

별도의 CSS 파일을 관리하지 않고 클래스명만으로 스타일링이 가능하고, 

다크모드 지원이 내장되어 있어서 채택하게됐는데 이게 물건이었다. 

현재 실무에서도 많이 사용되는 방식이라하여 학습 가치도 높다고 판단했고, 

그 찍먹은 상당히 좋은 경험이었다.

---

# Infrastructure

### Oracle Cloud Free Tier

다른 무료 플랜들은 Sleep이 걸려있어서 Sleep 없는 영구 무료 서버가 필요했다. 

(덕분에 서버 자리 뚫는데 귀중한 시간을 헌납했지만...) 

AWS나 GCP의 무료 플랜은 기간 제한이 있거나 일정 시간 요청이 없으면 Sleep 상태가 되는 문제가 싫었다. 

반면에 Oracle Cloud Always Free는 VM 인스턴스 2개를 영구적으로 무료로 제공하기 때문에 과감히 선택! 

또한 직접 서버를 세팅하고 배포하는 과정을 (통해 약간의 땀을...) 경험함으로서

인프라 관련 역량도 키울 수 있는 필요하고, 중요한 경험을 했다.

### GitHub Pages

React 프론트엔드 배포는 GitHub Pages를 선택! 

별도의 서버 없이 GitHub 레포지토리에서 바로 정적 파일을 서빙할 수 있고, 

git push 후 배포 스크립트 한 줄로 자동 배포되는 편리함 적용.

---

# 레포지토리 구조 및 분리 이유

총 3개의 레포지토리로 분리했다. 관심사의 분리(Separation of Concerns) 원칙을 적용

seungBolg : React 코드 -> 배포 : GitHub Pages 
seungBlog-backend : Spring Boot 코드 -> 배포 : Oracle Cloud
seung-content : .md 파일 작성 

이렇게 분리하면 글을 추가할 때는 seung-content 레포만, 

UI를 바꿀 때는 seungBolg 레포만, 

API 로직을 수정할 때는 seung-backend 레포만 건드리면 된다.

---

# 백엔드 레이어 설계


Controller : HTTP 요청/응답 처리만 담당 
Service : 비즈니스 로직, GitHub API 호출 
Security : JWT 검증 필터, GitHub OAuth 처리 
Config : CORS, Security, Bean 설정 
DTO : 요청/응답 데이터 형식 정의 
Exception : 전역 예외처리 (@RestControllerAdvice) 

Controller와 Service를 분리한 이유는 단일 책임 원칙(SRP)을 지키기 위함이다. 

Controller는 요청/응답만 담당하고, 비즈니스 로직은 Service에만 작성하였다. 

이렇게 하면 로직이 변경되어도 Controller는 수정하지 않아도 되고, 

레이어별로 독립적인 테스트 작성도 가능하다.

---

# GitHub OAuth + JWT 인증 흐름

1. 사용자가 로그인 버튼 클릭
2. GitHub 로그인 페이지로 리다이렉트
3. 사용자 GitHub 로그인 승인
4. GitHub이 임시 code를 Spring Boot 서버로 전달
5. Spring Boot가 code로 GitHub에 access_token 교환
6. access_token으로 GitHub 사용자 정보 조회
7. 사용자 정보를 담은 JWT 발급
8. JWT를 프론트로 전달 후 localStorage에 저장
9. 이후 API 요청마다 Authorization 헤더에 JWT 포함
10. Spring Security JwtAuthFilter가 JWT 검증

JWT를 선택한 이유는 프론트와 백엔드가 완전히 분리된 구조이기 때문에 

세션 방식은 서버에 상태를 저장해야 하지만, JWT는 서버가 아무것도 저장하지 않는 

Stateless 방식이라 이 구조에 더 적합하다고 생각했다.

---

# CORS 설정 이유

프론트(GitHub Pages)와 백엔드(Oracle Cloud)의 도메인이 다르기 때문에 CORS 설정이 필수였다. 

브라우저는 보안상 다른 도메인 간의 요청을 기본으로 차단. 고로 Spring Boot에서 허용할 도메인을 

명시적으로 설정해서 이 문제를 해결. 개발 환경(localhost:5173)과 배포 환경(github.io)을 

모두 허용하도록 yml 파일에서 환경별로 관리했다.

---

# 전역 예외처리 설계

@RestControllerAdvice를 사용한 전역 예외처리를 구현했다.

- **CustomException**: 직접 정의한 비즈니스 예외
- **HttpClientErrorException**: GitHub API 호출 실패
- **Exception**: 예상치 못한 모든 예외

이렇게 하면 Controller에 예외처리 코드가 없어도 되고, 모든 에러 응답 형식이 통일되며, 

클라이언트는 항상 같은 형태의 에러 JSON을 받기 때문에 프론트엔드에서 에러 처리가 일관성 있게 가능하다.

---

# GitHubApiClient 분리 이유

처음에는 PostService와 GuestbookService 각각에 GitHub API 호출 코드와 헤더 생성 코드가 중복됐었는데, 

DRY 원칙(Don't Repeat Yourself)을 적용해서 GitHubApiClient로 분리. 

덕분에 GitHub API 호출 방식이 바뀔 때 한 곳만 수정하면 되는 구조 형성.

---

DB 없이 GitHub 레포가 CMS(콘텐츠 관리 시스템) 역할.

---

# 구현 기능 목록

- ✅ 블로그 글 목록/상세 조회 (GitHub Contents API)
- ✅ 마크다운 렌더링 (react-markdown)
- ✅ 방명록 목록 조회/작성/삭제 (GitHub Issues API)
- ✅ GitHub OAuth 2.0 로그인
- ✅ JWT 발급 및 검증
- ✅ 다크/라이트 모드 토글
- ✅ 스크롤 애니메이션 (IntersectionObserver)
- ✅ 타이핑 애니메이션
- ✅ 스켈레톤 로딩 UI
- ✅ 전역 예외처리
- ✅ CORS 설정
- ✅ 프론트 GitHub Pages 배포
- ✅ 백엔드 Oracle Cloud 배포

---

# 아쉬운 점 및 개선 방향

**1. 캐싱 미적용**

현재 매 요청마다 GitHub API를 호출하는 구조로 글 목록처럼 자주 바뀌지 않는 데이터는 

Spring Cache를 적용하면 성능을 개선, 

GitHub API는 시간당 5000회 호출 제한이 있기 때문에 트래픽이 많아질 경우 문제가 될 수 있다.

**2. CI/CD 파이프라인 부재**

현재는 수동으로 배포하는 구조이다. GitHub Actions를 활용한 자동 배포 파이프라인을 

구축하면 코드 변경 시 자동으로 빌드 및 배포가 이루어지도록 개선할 수 있다.

---

# 프로젝트를 통해 배운 것

1. Spring Boot 기반 REST API 전체 설계 및 구현 경험
2. OAuth 2.0 인증 흐름 직접 구현 및 이해
3. JWT Stateless 인증 방식의 장단점 이해
4. 외부 API(GitHub)를 활용한 서버리스 아키텍처 설계
5. CORS, Spring Security 등 실무 필수 개념 적용
6. React + TypeScript 프론트엔드 직접 구현
7. 클라우드 인프라(Oracle Cloud) 직접 세팅 경험
8. 레이어 분리, DRY 원칙 등 설계 원칙 실무 적용
---

### 마치며..
가벼운 프로젝트를 혼자서 진행했지만 기대 이상으로 재밌었다. 

GitHub API를 실무에서 사용할 일이 있는지는 잘 모르겠지만 가벼운 프로젝트를 

개인적으로 다루거나 가지고 놀기에 정말 매력있는 API인 것 같다고 생각했다. 

물론 동시에 아쉬운 점은 분명히 있었다. 어찌보면 첫 포트폴리오 작업물 중 하나일 

이 프로젝트가 DB를 품고있지 않아서 역량을 담아내지 못한 아쉬움이 남는다. 

하지만 후회는 없다. 

앞으로 다른 프로젝트에서 충분히 DB를 선보일 수 있고, 사용할 경험도 많고, 

최종 프로젝트와 다른 개인 프로젝트로 DB를 다룰 예정이기 때문이다. 

그리고 이 프로젝트에서는 한번씩 방문할 사람들의 발자취를 간직하고 싶었고, 

개인정보에 예민하지 않은 서비스를 제공하고 싶었다. 고작 한 사람만을 위한 서비스에 

DB로 무게감을 주고싶지 않았고, 누구나 편하게 들렀다가 갈 수 있는 곳이었으면 좋겠다는 생각이 강했다. 

비록 짬뽕같은 느낌이 있는 프로젝트 처럼 보일지라도 GitHub 관련 API를 다루어본 덕분에 

REST API 설계, JWT 토큰 방식, 서버리스 아키텍처 설계, 외부 API 의존성 관리 등 재밌는 경험을 했고, 

내가 원하는 학습의 목표로 방향을 맞추는 방법을 구상하여 경험했음이 만족스러운 프로젝트였다.

### 끝까지 읽어주신 방문자님께 행운을  🍀🍀
---

![Image](https://github.com/user-attachments/assets/7eb5e70e-9b6c-4a1a-8113-dabfe11cc9ae)
