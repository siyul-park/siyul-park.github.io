---
title: "HTTP Simple Request 탐험하기"
date: 2024-04-19 12:00:00 +0900
categories: "network"
tags: ["network", "http", "oauth2"]
---

웹은 사용자의 의도와 무관하게 공격자가 특정 웹사이트에 의도하지 않은 행위를 요청하여 정보를 탈취하는 사이트 간 요청 위조(Cross-site request forgery, CSRF, XSRF)를 막기 위해 기본적으로 같은 출처에서만 리소스를 공유할 수 있는 동일 출처 정책(SOP, Same-Origin Policy)를 사용합니다.

다른 출처에 있는 리소스가 필요한 경우, 추가 HTTP 헤더를 사용하여 한 출처에서 실행 중인 웹 애플리케이션이 다른 출처의 선택한 자원에 접근할 수 있는 권한을 부여하여 교차 출처 리소스 공유(CORS, Cross-Origin Resource Sharing)를 허용합니다.

웹 브라우저는 다른 출처에 있는 리소스에 접근할 때 `OPTIONS` 메소드를 통해 Preflight 요청을 보내어 실제 요청이 전송하기에 안전한지 확인합니다. 이때 `Origin` 헤더를 사용하여 원래 출처를 전달하고, `Access-Control-Request-Method`와 `Access-Control-Request-Headers`에 허용되어야 할 메소드와 헤더를 넣습니다.

```http
OPTIONS https://siyul-park.github.io

Origin: http://localhost:3000
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Content-type
```

서버는 이에 대한 응답으로 허용하는 출처, 메소드, 헤더를 반환합니다.

```http
OPTIONS https://siyul-park.github.io/ 200 OK

Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET
Access-Control-Allow-Headers: Content-type
```

일부 요청은 Preflight를 전송하지 않고도 다른 출처에 있는 리소스에 접근할 수 있습니다. 이런 요청을 단순 요청(Simple Request)라고 합니다. 단순 요청은 `GET`, `HEAD`, `POST` 메소드를 사용하고 유저 에이전트가 자동으로 설정한 헤더 외에 `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` 헤더만 사용할 수 있습니다. 또한 `Content-Type`은 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` 만 허용합니다.

다른 출저로 리소스 공유가 필요하지만 요청의 출처를 명확히 알 수 없는 경우, 단순 요청을 사용하여 API를 설계할 수 있습니다.

예를 들어, 3자 인증을 위한 표준 프로토콜인 [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)은 다양한 출처에서의 요청을 처리할 수 있어야 하여 단순 요청을 적극적으로 활용합니다. 토큰을 얻는 과정에서 `application/json`을 `Content-Type`로 사용하는 응답의 본문과는 달리 요청의 본문에 `application/x-www-form-urlencoded`를 사용하여 단순 요청의 조건을 충족시킵니다.

```http
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA&client_id=s6BhdRkqt3&client_secret=7Fjfp0ZBr1KtDRbnfVdmIw
```

표준을 살펴보고 그 의도를 이해함으로써 수많은 전문가들이 직면한 고민과 결론을 통해 멋진 인사이트를 얻을 수 있습니다. 새로운 명세를 설계할 때 기존에 존재하던 관련된 표준을 참고하면 문제에 대한 해결책과 이미 검증된 방법을 얻을 수 있죠. 거인의 어깨에 올라서서 더 넓은 세상을 바라보세요. 바퀴를 재발명할 필요는 없습니다.

## 참고 문서
- [교차 출처 리소스 공유 (CORS)](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS)
- [CORS는 왜 이렇게 우리를 힘들게 하는걸까?](https://evan-moon.github.io/2020/05/21/about-cors/#sopsame-origin-policy)
- [사이트 간 요청 위조](https://ko.m.wikipedia.org/wiki/%25EC%2582%25AC%25EC%259D%25B4%25ED%258A%25B8_%25EA%25B0%2584_%25EC%259A%2594%25EC%25B2%25AD_%25EC%259C%2584%25EC%25A1%25B0)