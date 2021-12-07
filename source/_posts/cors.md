---
title: CORS 이해하기
date: 2021-03-09 22:22:22
index_img:
category: web
tags:
  - web
math: true
mermaid: true
sticky: 100
author: seonjl
---

### CORS 란

교차 출처 리소스 공유 ( Cross-Origin Resource Sharing) 을 의미한다.

여기서 출처 ( Origin ) 이란

`https://api.server.com/blah/blah`  와 같은 URL 에서

프로토콜 + 호스트 + 포트 를 전부 합친 서버의 위치, 리소스의 출처를 의미한다

포트는 기본 프로토콜 포트번호가 정해져 있기 때문에 생략할 수 있으며, 만약 명시적으로 포트가 포함되어 있다면 포트번호까지 모두 일치해야 한다.

### SOP ( Same-Origin Policy )

다른 출처로의 리소스 요청을 제한하는 보안 정책

다른 출처의 리소스를 마음대로 가져와서 사용할 수 있는 웹 환경은 매우 보안에 취약하다.그래서, 이러한 SOP 정책을 도입하여 안전한 환경을 구축 할 수 있지만, 다른 출처에 리소스를 가져와서 사용하는 일은 분명히 필요한 일이기에, 예외적으로 이를 허용해 줄 필요가 있다.

그 중 하나가 바로 CORS 정책을 지킨 리소스 요청이다.

이러한 보안 정책은 브라우저에 내장되어 있기 때문에 브라우저를 통하지 않고 서버 간 통신을 할 떄는 적용되지 않는다.

### CORS 의 기본적인 동작 방식

클라이언트는 HTTP 프로토콜을 사용하여 요청을 보내게 되고,
이때 브라우저는 요청 헤더에 `Origin` 필드에 출처를 함께 담아 보낸다.

이 요청을 받은 서버는 응답 헤더의 `Access-Control-Allow-Origin` 이라는 키에 허용된 출처 값을 내려주고, 응답을 받은 브라우저는 자신이 보낸 요청의 Origin 과 서버가 보내준 응답의 `Access-Control-Allow-Origin` 을 비교해 본 후 유효성을 결정한다.

## 세가지 시나리오

### Preflight Request

클라이언트가 요청을 보내기 전 브라우저가 보내게 되는 예비 요청이다

OPTIONS 메소드를 사용하여 서버가 어떤 것들을 허용하고, 어떤 것들을 금지하고 있는지에 대한 정보를 담아오게 된다. 브라우저는 이 요청을 검사한 후 본 요청의 송신 여부를 결정하게 된다.

### Simple Request

브라우저가 예비요청을 보내지 않고 바로 본 요청을 보내는 경우도 있다.

아래에 특정한 조건을 만족하는 경우에 보내지게 된다.

1. 요청의 메소드는 `GET`, `HEAD`, `POST` 중 하나여야 한다.
2. `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`, `DPR`, `Downlink`, `Save-Data`, `Viewport-Width`, `Width`를 제외한 헤더를 사용하면 안된다.
3. 만약 `Content-Type`를 사용하는 경우에는 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`만 허용된다.

### Credentialed Request

클라이언트에 요청에 자격증명모드가 `include` 일경우 발생한다. 

이러한 경우 브라우저에 보안 정책이 한가지 더 추가 되게 된다.

브라우저는 쿠키정보나 인증과 관련된 헤더를 함부로 요청에 담지 않는다. 이러한 인증정보를 요청에 담을 수 있게 해주는 옵션이 바로 `Request.credentails` 옵션이다.

1. `same-origin (default)`  :  같은 출처 간 요청에만 인증 정보를 담을 수 있다.
2. `include`  :  모든 요청에 인증 정보를 담을 수 있다.
3. `omit`  :  모든 요청에 인증 정보를 담지 않는다

만약 다른 출처로 인증관련 헤더를 포함하여 요청을 보내고 싶다면
credentials 의 include 모드를 사용하여 요청을 작성하여 보내야 한다. 

이때 발생할 수 있는 두가지 정책 위반이 있다.

{% note info %}
🚨  Access to fetch at ’https://api.server.com’ from origin ’http://localhost:8000’ has been blocked by CORS policy: The value of the ‘Access-Control-Allow-Origin’ header in the response must not be the wildcard ’*’ when the request’s credentials mode is ‘include’.
{% endnote %}


{% note info %}
🚨  Access to XMLhttpRequest at ’https://api.server.com’ from origin ’http://localhost:8000’ has been blocked by CORS policy: Response to preflight request doesn't pass access control check: The value of the ‘Access-Control-Allow-Credentials’ header in the response is ' ' which must be 'true' when the request's credentials mode is 'include'. The credentials mode of requests initiated by the  XMLhttpRequest is controlled by the withCredentials attribute.
{% endnote %}


{% note warning %}
1. \* 를 `Access-Control-Allow-Origin` 헤더에 사용하면 안된다.
2. 응답 헤더에 반드시 `Access-Control-Allow-Credentials: true` 가 존재해야한다.
{% endnote %}


## 해결방법

### Access-Control-Allow-Origin 세팅하기

nginx 설정에서 헤더값을 세팅해주거나

application 에서 세팅해주거나 하면 된다.

\* 로 전체를 허용해 주기 보다는 특정한 출처를 명시해주는 편이 좋다.

### Webpack Dev Server 로 리버스 프록싱하기

devServer 에 proxy 옵션을 사용해 특정 주소로 프록싱을 해주면

브라우저는 내가 세팅한 특정주소를 Origin 으로 알고 요청을 보내기 때문에 

CORS 정책을 지킨 것 처럼 브라우저를 속이고 통신할 수 있다.

### Hosts 파일 이용하기

hosts 파일 위치 : `C:\Windows\System32\drivers\etc`

```bash
127.0.0.1:8080		api.front.com
```

호스트파일을 위와 같이 변경하여 마치 같은 Origin 인 것처럼 속일 수 도 있다.

### 참고자료
---
[CORS는 왜 이렇게 우리를 힘들게 하는걸까?](https://evan-moon.github.io/2020/05/21/about-cors/)
