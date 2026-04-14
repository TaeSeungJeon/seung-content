---
title: "드디어 배포다! Oracle Cloud + GitHub Pages 배포 삽질 기록 🚀"
date: "2026-04-13"
description: "인스턴스 자리 뚫기부터 HTTPS 적용까지, 배포 과정과 트러블슈팅 기록"
category: "Dev"
---

# 배포를 시작하며

코드는 다 짰고, 로컬에서도 잘 돌아간다. 마지막 배포만 남았었다.

배포가 이렇게 험난할 줄은 진짜 몰랐다... Oracle Cloud 인스턴스 자리 뚫기부터 HTTPS 적용까지, 

삽질의 연속이었지만 결국 해냈고 그 과정을 기록해둔다. 

---

# 배포 구조

React 프론트엔드 : GitHub Pages 
Spring Boot 백엔드 : Oracle Cloud VM.Standard.A1.Flex (ARM, 1OCPU / 4GB) 
도메인 / SSL : DuckDNS + Let's Encrypt (Certbot) 
리버스 프록시 : Nginx 

---

# 배포 순서 요약

1. Oracle Cloud 인스턴스 생성 (Python 자동화 스크립트)
2. SSH 접속 → 패키지 업데이트 → JDK 21 설치
3. 로컬에서 JAR 빌드 → SCP로 서버 전송
4. `/etc/environment` 에 환경변수 등록
5. systemd 서비스 등록 → 자동 재시작 설정
6. ufw + OCI Security List 포트 오픈 (22, 80, 443, 8080)
7. Nginx 설치 → DuckDNS 무료 도메인 발급 → Certbot SSL 인증서 발급
8. Nginx 리버스 프록시 설정 (80 → 443 리다이렉트 + Spring Boot 프록시)
9. 프론트 `.env.production` 수정 → 재배포

---

# 트러블슈팅 기록

## 1. 인스턴스 자리가 없다...

Oracle Cloud Always Free `VM.Standard.A1.Flex` 는 전 세계 사람들이 몰려서 `Out of host capacity` 에러가 계속 난다. 

직접 콘솔에서 생성 버튼을 눌러봤자 의미없어서 Python 스크립트로 자동 재시도를 돌렸다.

며칠동안 소득이 없어서 검색을 통해 새로운 정보를 얻었는데.. 이걸 지금 하다니....

인스턴스 자리뚫기는 실패하고, 카드 등록 절차 이후 자리를 확보할 수 있었다... 야호..🎉

---

## 3. Mixed Content 에러

프론트는 GitHub Pages라 `https` 인데 백엔드 API가 `http` 라 브라우저가 차단했다.

```
Access to XMLHttpRequest at 'http://134.185.119.179:8080/api/posts'
from origin 'https://taeseungjeon.github.io' has been blocked
```

해결책은 백엔드에 HTTPS 적용하는 방법으로 DuckDNS 무료 도메인 + Certbot으로 해결했다.

---

## 4. CORS 에러 — 대소문자 불일치

```
No 'Access-Control-Allow-Origin' header is present
```

`application.yml` 에 `https://TaeSeungJeon.github.io` (대문자 T) 로 등록했는데, 

실제 요청 Origin은 소문자였다... 

참 사소한 트러블들이 건건히 괴롭히더라... 

그래도 가벼운 트러블이었으니 가벼운 수정으로 해결 !

```yaml
# application.yml
cors:
  allowed-origins: http://localhost:5173,https://TaeSeungJeon.github.io,https://taeseungjeon.github.io
```

---

## 5. Nginx 설정 충돌

Certbot이 SSL을 `default` 설정 파일에 적용하면서 직접 만든 `seung-backend` 설정과 충돌났다. 

`default` 설정을 비활성화하고 `seung-backend` 설정에 HTTPS를 직접 포함시켜서 해결했다.

```nginx
# /etc/nginx/sites-available/seung-backend
server {
    listen 80;
    server_name seungblog.duckdns.org;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name seungblog.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/seungblog.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/seungblog.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 6. Vite public 폴더 이미지 경로

`src="/public/gitIcon.png"` 로 평소처럼 작성했더니 배포 환경에서 404가 떴다. 

Vite에서 `public` 폴더 이미지는 `/public/` 없이 파일명만 써야 `base` 경로가 정상 적용되기 때문에 

아래와 같이 코드를 수정하는 과정이 있었다.

```tsx
src="/public/gitIcon.png"

->

const GIT_ICON = 'gitIcon.png';
src={GIT_ICON}
```

---

## 7. GitHub OAuth 콜백 URL 404

GitHub Pages + HashRouter 조합에서 OAuth 콜백 URL을 `/callback` 으로 설정했더니 GitHub Pages가 404로 처리했다. 

HashRouter는 `#` 이후 경로를 서버로 전송하지 않기 때문에, 콜백 URL도 `#/callback` 형태로 맞춰줘야 한다.

```
# GitHub OAuth App Authorization callback URL
https://taeseungjeon.github.io/SeungBlog/#/callback
```

---

# 마치며

배포가 이렇게 복잡한 과정인 줄은 몰랐다. 

코드를 짜는 것과 실제로 서비스로 띄우는 건 완전히 다른 영역이라는 걸 몸으로 느꼈다. 

포트 하나 열기 위해 방화벽을 두 군데서 설정해야 하고, HTTPS 하나를 위해 도메인과 인증서와 Nginx를 다 엮어야 했다.

브라우저에서 내 블로그가 실제 데이터를 불러오는 걸 처음 봤을 때의 희열은....엄지 다섯번 척!

GitHub API를 사용하면서 흥미로웠던 점이 기억에 남는다. 

백엔드에서 GitHub Contents API를 호출해서 마크다운 파일을 가져오고, 

그걸 프론트가 받아서 렌더링하는 흐름이 재미있었고 , 실제로 동작하는 걸 보니까 

REST API 소통하는 방식이 체감이 됐어서 좋은 기억이었다. 

DB 없이 GitHub 레포가 CMS 역할하는 설계에서 실제로 데이터가 오가는 걸 보면서

처음 기획하고 채택했을때 생각했던 것 보다 더 흥미롭게 다가왔다.

CORS도 흥미로운 부분이 있었는데, 로컬에서는 멀쩡하던 게 배포 환경에서 갑자기 막히는 경험을 하고, 

프론트와 백엔드 도메인이 다를 때 브라우저가 왜 요청을 차단하는지, 

allowed-origins 설정이 왜 필요한지를 직접 부딪히게 됐다. 

심지어 대소문자 하나 차이로도 막힌다는 걸 직접 겪어보니 CORS는 이론으로만 배울 수 없다는 생각이 들었다.

많은 부분에 있어서 복잡했고 아직은 많이 어려웠지만 다방면으로 재밌는 경험을 안겨준 프로젝트였다!

이상 긴 글 읽어주신 여러분께 감사를.... 여러분도 읽어주시느라 고생 많으셨습니다 ㅎㅎㅎㅎ
