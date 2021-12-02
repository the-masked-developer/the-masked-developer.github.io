---

title      : RESTful API design.
date       : 2021-12-02 23:11:45 +0900
tags       : 
categories : 
toc        : true

---

## RESTful API design's Goals.

1. Platform indenpendency.
2. Self-evolution.

> 굳이 HTTP 일 필요는 없다. design 의 문제.

## Design Principles.

1. Resource 를 중심으로 설계된다. 
   * Resource 는 고유의 identifier 를 가진다.
2. Representation 을 가진다. 보통 JSON.
3. 각 Request 는 Stateless, Atomic, Indenpendent 하다. 
   * Server 의 scale out 을 가능케 한다.
   * Indenpendent 하기 때문에 요청의 순서에 신경쓰지 않아도 된다.

## Maturity of RESTful HTTP API.

1. POST only.
2. Resource + URI 분리
3. Resource + HTTP methods.
4. HATEOAS (+hyperlinks)

> 보통, 대부분의 RESTful HTTP API 는 2 ~ 3 그 어딘가에 있다.
## Design guide.

1. No verbs. (Use methods for it!)
2. There's no need to match with database schema. (just abstract it. hide implementation.)
3. resource 의 collection 들은 다른 URI 를 가진다. 
   * ex) `/orders`
4. Resource 의 relationship 은 hierachy 하게 표현한다. 
   * ex) `/customers/{customer_id}/orders/{order_id}`...
   * 관계가 너무 많으면 관리가 힘들다. 그러므로 여러번 요청을 날린다던지 너무 계층이 많아지지 않도록 조심한다.
5. chatty 한 API design 보다는, 하나의 큰 load 를 가진 API 가 낫다. (denormalized API is better) 
   * 지나친 Overfetching 은 조심해야 한다.
6. Abstract database schema. do not expose it.
7. Resource 와 관련없는 서버의 function, procedure 를 실행해야 할 경우도 있다. 적당히 사용한다.

### HTTP methods.

1. `GET`: Resource's representation. 
   * `200`
   * `404`
2. `POST`: Resource's creation or call server's function. 
   * `201`: creation success.
   * `200`: no creation.
   * `400`: bad representation.
   * 생성시 `Location` Header 에 resource 의 ID 를 담아서 리턴한다.
3. `PUT`: Create or update resource (w/ complete representation) 
   * `201`, `200`, `400`
   * `409`(Conflict): Valid 한 요청이지만, Server 의 state 로 인해 update 를 실패했다.
   * 생성의 경우 Client 가 적절하게 좋은 ID 를 생성할 수 있어야 한다.
4. `PATCH`: Update resource partially 
   * `200`, `400`, `409` 
   * Representation fields are optional.
   * `null` means delete the field's value.
5. `DELETE`: Delete resource. 
   * `204`(No Content): success

### Filter and Pagination

TBW. https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design#filter-and-paginate-data


### Versioning

1. No versioning. acceptable for internal APIs.
2. Header versioning. 
   * ex) `App-Version: ...`
3. Query string versioning. 
   * ex) `/...?version=1`
4. URI versiong. 
   * ex) `/v1/...`.
5. media type versioning.


## Open API

TBW. https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design#open-api-initiative

## References

* [Microsoft RESTful API best practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)

