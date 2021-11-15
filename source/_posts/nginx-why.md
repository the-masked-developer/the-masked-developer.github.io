---
title      : "Nginx 간단 개념 (why 중심)"
toc: true
date       : 2021-11-15 22:31:04 +0900
tags:
categories:
  - 프로그래밍
---

## Nginx 보다 먼저 만들어진 Apache server

1995년에 만들어진 Apache 는 많은 클라이언트의 요청을 처리할 수 있게 해주어 많이 사용되었으며, **확장성** 과 **안정성** 이 매우 뛰어남. client 의 요청이 들어올 경우, 미리 fork 해 둔 Process Pool 에서 해당 커넥션을 처리할 Process 를 할당하고 그 프로세스가 요청을 처리하게 됨. (초과 될 경우 fork) 이런 특징은 한 서버가 여러 요청을 처리할 수 있게 만들었으며 개발자들이 다양한 모듈을 작성할 수 있게 했다고 함. 

## C10k 시대와 Apache

**C10k** (10,000 Connection problem): 스마트폰의 등장과 인터넷의 활성화로 점점 client 가 많아지면서, 다음과 같은 문제점들에 직면하게 됨.

1. TCP Connection 생성의 비용은 비싸다. 그러므로 KeepAlive 로 client 들은 Connection 을 끊지 않기 시작했다.
2. Process 생성에는 메모리가 필요하다. Connection 의 수 = 메모리 필요량이 비례하면서 부족지기 시작했다.
3. Request 를 처리하기 위해 Process 를 switching 하는 context switching 비용도 비싸다.

그래서 위 문제를 해결하고자 Apache 진영도 많은 고민을 하고 개선해나가고 있으며, 새로운 구조로 만들어진 게 Nginx 이다.

## Nginx 의 특장점

**Event Driven** 구조이다. Nginx 에서 Event 란, Connection 의 생성, 제거, 처리 등을 말한다. Nginx 에는 Master Process 와 그것이 생성하는 Worker Process 가 있는데, Worker 는 Event 를 기다렸다가 처리한다. 처리는 Worker Process 의 Thread Pool 의 Thread 에게 맡기므로, 혹시 Event 늖 Blocking 이므로 처리중 I/O 등으로 오래걸리는 Event 가 있을 경우도 동시에 처리가 가능하다.

개략적인 구조의 변경은 위와 같고, 정리해서/ 추가적으로 특장점은 다음과 같다.

* 하나의 Worker Process 가 많은 Connection Event 을 처리할 수 있게 되었다.
* 적은 Worker Process 의 수를 가지기 때문에, (보통 코어개수) 메모리 사용량이 적다. (요청이 늘어도 Thread stack 크기 정도의 메모리 추가사용)
* I/O 가 많이 필요한 Blocking 작업들 또한 효율적으로 처리하게 되었다.
* Worker Process 를 새로 띄우고 Event 를 맡기면 되므로, 동적으로 설정변경이 가능하게 되었다. (최 앞단에서 로드밸런싱등을 수행할 때 Single Point 잉므로 이 기능은 매우 핵심)

## Nginx 의 기능

https://nginx.org/en/

* 정적파일 서빙, 및 오토인덱싱, open fd 캐싱? 등
* 리버스 프록시서버로써 동작 및 캐싱.
* 로드밸런싱, fault tolerance
* filter 로써, 데이터 압축, 변환, ...
* SSL Termination - SSL 처리 부담을 로직서버에 안가게끔 앞단에서 처리
* CORS / HTTP/2 등

## 참고자료

* [Nginx official docs](https://nginx.org/en/)
* [우아한 형제들 10분 테코톡](https://www.youtube.com/watch?v=6FAwAXXj5N0)


