# [배포 학습 - Part 4] Nginx 리버스 프록시(Reverse Proxy) 및 CORS 극복

이 파트에서는 Nginx 웹서버를 프론트 브라우저 요청의 단일 관문(Reverse Proxy)으로 두고, SPA의 고유 이슈인 라우팅 에러 해결 및 CORS 방화벽을 우회하는 이론을 공부합니다.

---

## 1. nginx.conf 코드 분석
[nginx.conf](file:///c:/workspace/Insajang/insajang-frontend/nginx.conf)

```nginx
server {
    listen 80;
    server_name localhost;

    # 1. React 정적 파일 서빙
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html; # React Router(SPA) 지원용 리다이렉트
    }

    # 2. 백엔드 API 요청 프록시 우회 (CORS 이슈 원천 차단)
    location /api/ {
        proxy_pass http://insajang-backend:8080; # 도커 네트워크 상의 백엔드 컨테이너 포워딩
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 2. 핵심 이론 및 원리 학습

### ① `try_files $uri $uri/ /index.html;` (SPA 라우팅의 핵심)
*   **문제**: React는 HTML이 단 한 장 뿐인 SPA(Single Page Application)입니다. 화면 상의 메뉴를 클릭해 `/profile`로 이동하는 것은 실제 서버의 폴더 구조가 아닌 브라우저 상의 자바스크립트가 처리한 가상 주소입니다. 이 상태에서 새로고침을 하면 브라우저는 Nginx 서버에 "서버 내의 `/profile` 폴더나 파일 줘"라고 요청하게 되며, 당연히 해당 경로에 실제 파일이 없으므로 **404 Not Found** 에러가 떨어집니다.
*   **해결**: `try_files`는 사용자가 요청한 경로에 파일(`$uri`)이나 폴더(`$uri/`)가 진짜로 없으면 에러를 내지 말고 무조건 **`/index.html`**로 요청을 우회(Fallback)시키는 규칙입니다. 일단 `index.html`을 클라이언트에게 보낸 뒤, 그 뒤의 자바스크립트(React Router)가 주소창을 분석하여 알맞은 화면을 띄우게 되므로 새로고침 시 404 문제가 완벽하게 해결됩니다.

### ② `location /api/` 리버스 프록시와 CORS 회피 원리
*   **문제**: 브라우저 보안 정책상 프론트엔드 웹 서비스와 백엔드 API 서비스의 **출처(Origin, IP와 포트)**가 다르면 통신을 강제 거부하는 CORS 방화벽이 동작합니다.
*   **해결**: Nginx가 80포트 대문자리가 되어 모든 브라우저 요청을 홀로 받아줍니다. 
    - 브라우저가 화면을 보여달라고 요청하면 Nginx는 React의 HTML을 보냅니다.
    - 브라우저가 `/api/user/login`으로 데이터 요청을 보내면, Nginx는 이 요청을 가로채서 도커 내부망에 조용히 띄워져 있는 스프링부트 컨테이너 포트(`insajang-backend:8080`)로 우회(Proxy Pass)하여 보내줍니다.
*   **효과**: 브라우저가 볼 때는 모든 통신이 **포트 80**이라는 하나의 문으로만 일어나므로 포트 불일치가 발생하지 않으며, **CORS 방화벽 필터를 완전히 패스**하게 됩니다.
