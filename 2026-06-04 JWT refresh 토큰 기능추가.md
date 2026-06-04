# [TIL] Spring Boot & React에서 JWT + Refresh Token 구현 트러블 슈팅 및 연동 정리
- **작성일**: 2026-06-04
- **주제**: JWT Access Token 만료에 따른 Refresh Token 기반 자동 재발급(Silent Reissue) 및 동시성 제어 해결
오늘 프로젝트에 리프레시 토큰(Refresh Token) 기능을 추가 및 연동하는 과정에서 발생한 **백엔드 필터 차단 문제**와 **프론트엔드 비동기 동시 요청 문제**를 트러블 슈팅하며 배운 내용을 정리한다.
---
## 1. 오늘 마주한 문제 상황 (Trouble)
### ① [Backend] `/api/user/reissue` (재발급 API) 호출 시 401 Unauthorized 에러 발생
* **현상**: Access Token이 만료된 상태에서 프론트엔드가 재발급을 요청했으나, 컨트롤러로 요청이 전달되기도 전에 `401 Unauthorized` 에러로 차단되었다.
* **원인**: `JwtAuthenticationFilter`에서 헤더의 토큰이 만료되었을 때 즉시 응답코드 401을 설정하고 `return` 처리하여 필터 체인을 중단시켰기 때문이다. 프론트엔드의 Axios 인터셉터가 만료된 Access Token을 자동으로 헤더에 담아서 reissue API를 보냈기 때문에 필터 단계에서 무조건 차단되는 모순이 생겼다.
### ② [Frontend] Access Token 만료 시 즉시 로그아웃 처리되는 현상
* **현상**: 기존에는 API 호출 중 401 에러를 받으면 리프레시 토큰을 이용한 재발급을 시도하지 않고 즉시 로컬 스토리지를 비운 후 로그인 페이지로 튕겼다.
* **원인**: 프론트엔드 Axios 인터셉터에 재발급 로직이 연동되지 않아 백엔드의 리프레시 토큰 기능이 활용되지 못하고 있었다.
### ③ [Frontend] 동시에 여러 API를 호출할 때 리프레시 토큰 중복 갱신 충돌
* **현상**: 만료 시점에 여러 개의 API가 동시에 비동기로 요청될 경우, 각 요청이 개별적으로 `/reissue`를 요청하게 된다. 이때 **Refresh Token Rotation (RTR)** 정책 때문에 첫 요청만 성공하고 나머지 요청들은 이미 사용되어 만료된 리프레시 토큰을 전송한 꼴이 되어 강제 로그아웃되는 버그가 발생할 수 있었다.
---
## 2. 해결 방법 및 배운 점 (Shooting & Learnt)
### ① 필터는 인증 정보만 세팅하고, 차단 책임은 Spring Security에 위임하기
* **배운 점**: 필터 단에서 특정 API(예: 로그인, 재발급 등)를 우회하기 위해 요청 경로를 하드코딩해서 예외처리하는 것은 확장성이 떨어진다. 
* **해결**: `JwtAuthenticationFilter`에서는 토큰이 유효할 때만 `SecurityContext`에 정보를 채우고, 만료/유효하지 않더라도 `filterChain.doFilter(...)`를 호출하여 무조건 필터 체인을 통과시킨다.
* **결과**: 인증 주머니가 비어 있는 요청은 `SecurityConfig`에 설정된 `AuthenticationEntryPoint`에 의해 시큐리티 단에서 일괄적으로 401 차단 처리가 되므로, 허용된 경로(permitAll)인 `/api/user/reissue`는 정상적으로 컨트롤러까지 도달할 수 있게 되었다.
### ② Axios 응답 인터셉터에서 대기 큐(Request Queue) 구현하기
* **배운 점**: 비동기 통신 환경에서 토큰 만료는 동시다발적으로 발생할 수 있다. 한 번에 여러 개의 API가 401 에러를 만났을 때 중복으로 토큰 갱신을 요청하지 않도록 동시성을 제어해야 한다.
* **해결**: 
  1. `isRefreshing` 플래그를 두어 첫 번째 401 에러가 재발급 프로세스를 시작하도록 하고 플래그를 `true`로 바꾼다.
  2. 재발급이 진행되는 동안 실패하는 다른 API 요청들은 `failedQueue` 배열에 `Promise` 객체로 보류시킨다.
  3. 토큰 재발급 성공 시 로컬 스토리지를 갱신하고, `failedQueue`에 담긴 대기 중인 요청들에 새 Access Token을 입혀서 순차적으로 다시 실행(`resolve`)한다.
  4. 실패 시에만 최종 로그아웃 처리한다.
---
## 3. 핵심 수정 코드 비교 정리
### Backend: Filter & Security Config
#### JwtAuthenticationFilter (즉시 차단 제거)
```java
// [수정 전]
if (token != null) {
    if (jwtTokenProvider.validateToken(token)) {
        // ...
    } else {
        response.setStatus(401); // 🚨 직접 차단 후 체인 끊어버림
        return;
    }
}
// [수정 후]
// 💡 검증이 실패해도 401로 직접 막지 않고, Context만 비워둔 채 다음 필터로 진행시킨다.
if (token != null && jwtTokenProvider.validateToken(token)) {
    Authentication auth = jwtTokenProvider.getAuthentication(token);
    SecurityContextHolder.getContext().setAuthentication(auth);
}
filterChain.doFilter(request, response);
```
#### SecurityConfig (예외 처리 등록)
```java
// 💡 시큐리티 설정에 exceptionHandling과 EntryPoint를 추가해 보호된 리소스의 차단 처리를 도맡아 하도록 구성
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/user/**","/uploads/**").permitAll()
        .anyRequest().authenticated()
)
.exceptionHandling(exception -> exception
        .authenticationEntryPoint((request, response, authException) -> {
            response.setStatus(401);
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write("{\"error\": \"Unauthorized\", \"message\": \"토큰이 만료되었습니다.\"}");
        })
)
```
---
### Frontend: Axios Interceptor (Silent Reissue 구현)
```javascript
// src/utils/api.js
import axios from "axios";
const API = axios.create({
  baseURL: "http://localhost:8080", 
  timeout: 5000,
});
let isRefreshing = false;
let failedQueue = [];
// 대기 중인 요청을 다시 수행하기 위한 헬퍼 함수
const processQueue = (error, token = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });
  failedQueue = [];
};
// ... 요청 인터셉터는 기존과 동일하게 Bearer 토큰 탑재 ...
// 응답 인터셉터 수정
API.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    // 401 Unauthorized 에러 감지 시
    if (error.response && error.response.status === 401 && !originalRequest._retry) {
      
      // 재발급 요청 자체가 401 에러를 만났다면 만료된 리프레시 토큰이므로 로그아웃
      if (originalRequest.url === "/api/user/reissue") {
        clearAuthDataAndRedirect();
        return Promise.reject(error);
      }
      originalRequest._retry = true;
      // 이미 토큰 재발급이 진행 중이라면 큐에 담아두고 대기
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            originalRequest.headers.Authorization = `Bearer ${token}`;
            return API(originalRequest);
          })
          .catch((err) => Promise.reject(err));
      }
      isRefreshing = true;
      try {
        const rawRefreshToken = localStorage.getItem("refreshToken");
        if (!rawRefreshToken) throw new Error("No refresh token");
        const refreshToken = rawRefreshToken.replace(/^"|"$/g, '');
        // 💡 중요: 헤더 중첩 및 무한 루프를 피하기 위해 순수 axios로 재발급 요청 전송
        const res = await axios.post("http://localhost:8080/api/user/reissue", {
          refreshToken: refreshToken,
        });
        if (res.status === 200) {
          const { accessToken: newAccessToken, refreshToken: newRefreshToken } = res.data;
          localStorage.setItem("accessToken", JSON.stringify(newAccessToken));
          localStorage.setItem("refreshToken", JSON.stringify(newRefreshToken));
          API.defaults.headers.common["Authorization"] = `Bearer ${newAccessToken}`;
          originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
          // 대기 큐의 요청들에게 새 토큰 제공 후 일괄 실행
          processQueue(null, newAccessToken);
          isRefreshing = false;
          return API(originalRequest); // 본래 실패했던 요청 다시 실행
        }
      } catch (refreshError) {
        processQueue(refreshError, null);
        isRefreshing = false;
        clearAuthDataAndRedirect();
        return Promise.reject(refreshError);
      }
    }
    return Promise.reject(error);
  }
);
```
---
## 4. 요약 및 주의할 점 (Summary)
1. **DB 스키마 검증**: `ddl-auto=validate` 환경에서는 `refresh_tokens` 테이블이 DB에 직접 생성되어 있어야 하므로 아래 테이블 생성 쿼리를 실행해 두어야 한다.
   ```sql
   CREATE TABLE public.refresh_tokens (
       refresh_token_id BIGSERIAL PRIMARY KEY,
       token VARCHAR(500) NOT NULL UNIQUE,
       user_id BIGINT NOT NULL,
       expiry_date TIMESTAMP NOT NULL
   );
   ```
2. **Refresh Token Rotation (RTR)**: 재발급 시 리프레시 토큰까지 함께 갱신해 주어 보안성을 향상한다.
3. **Queueing 기법**: 동시에 다발적으로 날아가는 통신 환경에서 재발급 요청의 중복 실행과 레이스 컨디션 문제를 해결하는 핵심 패턴임을 배웠다.
