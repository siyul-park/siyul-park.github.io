---
title: "RESTful API 설계 가이드"
date: 2024-09-08 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "network"]
---

[gRPC](https://grpc.io/)와 같은 원격 프로시저 호출이나 [GraphQL](https://graphql.org/)과 같은 쿼리 언어가 인기를 끌고 있지만, REST(Representational State Transfer)는 여전히 그 단순함과 직관성 덕분에 가장 널리 사용되는 방법입니다.

REST는 2000년에 로이 필딩(Roy Fielding)의 박사 논문에서 처음 소개된 소프트웨어 아키텍처 스타일입니다. 필딩은 웹과 같은 분산 하이퍼미디어 시스템을 분석하고 이를 바탕으로 REST 아키텍처를 설계했습니다.

초기 웹에서는 서버가 제공하는 모든 데이터를 리소스(resource)로 보고 URI를 리소스의 주소로 사용했습니다. 하지만 이 방식은 리소스가 변하지 않는 것처럼 보이게 하고, 리소스가 변경될 때마다 새로운 URI를 생성해야 한다고 오해할 수 있었습니다. 또한, 리소스를 문서 형식으로 표현하기 어려운 경우도 있었고, URI 생성에 대한 명확한 규칙도 부족했습니다.

REST는 리소스를 실제 데이터나 문서가 아니라 **데이터에 대한 식별 정보**로 정의합니다. 웹이 하이퍼미디어 네트워크인 만큼, 텍스트 문서뿐만 아니라 HTML, JSON, 이미지, 동영상 등 다양한 형식의 데이터도 리소스에 포함됩니다. 필딩은 **리소스**와 그 **표현 방식**을 분리했습니다. 즉, 동일한 리소스도 XML, HTML, JSON 등 다양한 형식으로 표현할 수 있으며, 리소스 자체와 표현 방식은 독립적으로 다뤄집니다. 웹에서 컴포넌트들은 리소스 자체를 전송하는 것이 아니라 **리소스의 표현**을 주고받으며 소통합니다.

REST는 웹 페이지들로 이루어진 네트워크를 하나의 **상태 머신**처럼 봅니다. 사용자가 링크를 클릭하면 새로운 상태로 전환되며, 이를 통해 애플리케이션이 진행됩니다. 즉, 웹은 여러 상태를 가질 수 있으며, 사용자가 링크를 클릭할 때마다 상태에 해당하는 URI로 이동하고, 상태 정보는 적절한 표현 방식으로 변환되어 전송됩니다.

REST 아키텍처는 웹의 동작 방식을 체계적으로 설명하며, 웹 프로토콜 표준을 위한 가이드라인을 제시합니다. REST는 다양한 네트워크 기반 구조를 통합하여 분산 하이퍼미디어 시스템을 다루며, 공통된 인터페이스를 제공하기 위해 여러 제약을 단계적으로 추가합니다. 필딩은 그의 논문에서 제약이 없는 상태에서 시작해 점진적으로 제약을 더해 나가며 REST를 도출했습니다.

첫 번째 제약은 **클라이언트-서버 아키텍처**입니다. 이는 사용자 인터페이스와 데이터 저장소를 분리하여 클라이언트의 이식성과 서버의 확장성을 높입니다. 이를 통해 클라이언트와 서버는 각각 독립적으로 발전할 수 있습니다.

다음은 **상태 비저장(Stateless)**입니다. 각 요청은 필요한 모든 정보를 포함해야 하며, 서버는 클라이언트의 세션 상태를 유지하지 않습니다. 이를 통해 서버의 가시성이 높아지고, 장애 발생 시 복구가 용이하며, 리소스 관리도 더 효율적으로 이뤄집니다.

상태 비저장 제약으로 인해 동일한 데이터를 반복해서 전송해야 하는 비효율성이 발생할 수 있습니다. 이를 해결하기 위해 **캐시**가 추가되었습니다. 서버 응답에 캐시 가능 여부를 명시함으로써 성능과 효율성을 개선할 수 있습니다.

REST 아키텍처의 핵심은 **균일한 인터페이스**입니다. 이 인터페이스는 시스템을 단순화하고, 각 구성 요소가 독립적으로 발전할 수 있도록 기반을 제공합니다. 다만, 특정 요구 사항에 맞춘 효율성은 다소 떨어질 수 있습니다.

REST는 네 가지 주요 제약을 통해 균일한 인터페이스를 구현합니다. 첫째, **리소스 식별**은 고유한 URI를 통해 이루어집니다. 둘째, **표현을 통한 리소스 조작**은 리소스 자체가 아니라 그 표현을 통해 상호작용합니다. 셋째, **자체 설명 메시지**는 메시지가 외부 정보 없이도 완결성을 가져야 합니다. 넷째, **애플리케이션 상태 엔진으로서의 하이퍼미디어**는 클라이언트가 하이퍼링크를 통해 리소스에 접근하고 상태를 전환할 수 있도록 합니다.

마지막으로, **계층 시스템** 제약은 시스템을 여러 계층으로 나누어 각 구성 요소가 자신이 상호작용하는 계층 너머의 세부 사항을 알 수 없게 함으로써 의존성을 줄이고 복잡성을 낮춥니다.

계층 시스템과 균일한 인터페이스는 **파이프-필터 스타일** 구조를 형성합니다. 각 메시지가 자체 설명적이기 때문에 중개자는 이를 변환하거나 처리할 수 있는 유연성을 갖습니다.

REST는 여러 제약을 통해 완성되지만, 이 제약들은 장점과 단점이 공존합니다. 애플리케이션에 따라 일부 제약은 장점보다 단점이 더 두드러질 수 있으며, 그런 경우 해당 제약을 굳이 따르지 않는 것이 더 나을 수 있습니다. REST의 모든 제약을 엄격히 지키지 않는 아키텍처는 **RESTful API**보다는 **HTTP API**라고 부르는 것이 더 적절할 수 있습니다. 그러나 이러한 아키텍처가 REST 스타일에 영향을 받는 경우가 많아, 종종 **RESTful**이라고 표현되기도 합니다.

## RESTful API 설계 가이드

### 리소스

리소스는 RESTful API에서 가장 중요한 개념입니다. 리소스를 식별하고 접근하기 위해 URI(Uniform Resource Identifier)를 사용합니다. URI는 애플리케이션 전체에서 일관되게 사용되어야 하며, 직관적으로 이해할 수 있어야 합니다. HTTP 메서드는 리소스에 대해 수행할 작업을 정의하므로, URI에 동사를 포함하는 것은 피하는 것이 좋습니다.

**잘못된 예시:**

```
/create-users
```

**올바른 예시:**

```
/users
```

URI 끝에 붙는 트레일링 슬래시(`/`)는 디렉토리를 나타냅니다. 트레일링 슬래시를 통해 서버 측에 리소스의 형식을 전달하여 성능을 개선할 수는 있지만, API에서는 일관성을 위해 이를 사용하지 않는 것이 좋습니다.

**잘못된 예시:**

```
/users/
```

**올바른 예시:**

```
/users
```

리소스 이름은 일관되게 유지하는 것이 중요합니다. 여러 하위 리소스를 가진 컬렉션 리소스는 단수형보다는 복수형으로 작성하는 것이 좋습니다.

**잘못된 예시:**

```
/user
/user/:id
```

**올바른 예시:**

```
/users
/users/:id
```

리소스 이름은 간결하고 이해하기 쉽게 작성해야 합니다. 단일 단어가 가장 좋지만, 여러 단어가 필요할 경우 하이픈(-)을 사용하여 구분하는 것이 좋습니다. URI의 호스트에서는 밑줄(_)이나 대소문자를 구분하지 않기 때문에, 경로에서도 하이픈을 사용하는 것이 통일성을 유지하는 데 도움이 됩니다.

**잘못된 예시:**

```
/orderHistories
```

**올바른 예시:**

```
/order-histories
```

리소스의 형식과 식별을 분리하여 유연한 표현을 지원하기 위해, 형식을 URI에 포함하기보다는 서버와 클라이언트 간에 HTTP `Accept` 헤더를 통해 형식을 협상하는 것이 바람직합니다.

**잘못된 예시:**

```
/users.json
/users.xml
```

**올바른 예시:**

```
/users
```

리소스가 다른 리소스의 하위 리소스인 경우, 슬래시(`/`)를 사용하여 계층 구조를 만들어 리소스 간의 관계를 명확히 표현합니다.

**잘못된 예시:**

```
/orders?user_id=:user_id
```

**올바른 예시:**

```
/users/:user_id/orders
```

일관성과 간결함을 유지하기 위해, URI의 깊이는 가능한 한 낮추고 동일한 리소스에 대해 단 하나의 URI를 유지하는 것이 좋습니다.

**잘못된 예시:**

```
/users/:user_id/orders
/users/:user_id/orders/:order_id
/products/:product_id/orders
/products/:product_id/orders/:order_id
```

**올바른 예시:**

```
/users/:user_id/orders
/products/:product_id/orders
/orders/:order_id
```

컬렉션 리소스의 일부를 필터링하려면 쿼리 매개변수를 사용하여 다양한 검색 조건을 지원하고 복잡한 조회를 할 수 있습니다.

**예시:**

```
/users?status=active
/products?price_min=50&price_max=200
/orders?created_after=2024-01-01
```

정규화된 방식을 사용할 수도 있습니다.

**예시:**

```
/users?eq(status)=active
/products?min(price)=50&max(price)=200
/orders?gt(created_at)=2024-01-01
```

컬렉션 리소스를 정렬하려면 `sort` 쿼리 매개변수를 사용하여 정렬 기준을 지정할 수 있습니다.

**예시:**

```
/orders?sort=created_at&sort=id
/orders?sort=-created_at&sort=id
```

불필요한 데이터 전송을 줄이기 위해, 필요한 리소스의 일부 정보만 요청할 때는 `fields` 쿼리 매개변수를 사용하여 필요한 필드를 지정할 수 있습니다.

**예시:**

```
/orders?fields=id&fields=user_id&fields=product_id
```

컬렉션 리소스의 특정 위치에 있는 하위 리소스만 요청하려면 `limit`과 `offset` 쿼리 매개변수를 사용할 수 있습니다.

**예시:**

```
/orders?limit=64&offset=64
```

### 행위

행위는 HTTP 메서드를 통해 표현됩니다. 사용자 정의 메서드를 추가할 수 있지만, 다양한 클라이언트와의 호환성을 고려하여 표준화된 메서드만 사용하는 것이 좋습니다.

**리소스를 읽을 때**는 **GET** 메서드를 사용합니다. **GET** 요청이 성공하면 200(OK) 응답 코드와 함께 요청한 리소스의 데이터가 반환됩니다. 리소스가 존재하지 않으면 404(Not Found) 응답 코드가, 요청이 잘못되었거나 파라미터가 부족하면 400(Bad Request) 응답 코드가 반환됩니다. **GET** 메서드는 데이터를 조회하는 용도로만 사용되며, 서버의 데이터나 상태를 변경하지 않습니다. 이 메서드는 멱등성을 보장하므로, 같은 요청을 여러 번 보내도 서버의 상태나 응답이 변하지 않으며, 응답을 캐시할 수 있습니다.

**리소스를 생성할 때**는 **POST** 메서드를 사용합니다. 요청이 성공적으로 처리되면 201(Created) 응답 코드와 함께 생성된 리소스에 대한 정보가 반환됩니다. 요청이 잘못되었거나 실패하면 400(Bad Request) 또는 422(Unprocessable Entity) 응답 코드가 사용될 수 있습니다. **POST** 메서드는 주로 서버에 데이터를 추가하거나 리소스의 상태를 변경하는 데 사용됩니다. 이 메서드는 멱등성이 보장되지 않으며, 같은 요청을 여러 번 보내면 여러 개의 리소스가 생성될 수 있습니다.

**리소스를 대체할 때**는 **PUT** 메서드를 사용합니다. 이 요청이 성공하면 기존 리소스가 새로 지정된 리소스로 대체되거나, 리소스가 존재하지 않는 경우 새로운 리소스가 생성됩니다. 요청이 성공적으로 처리되면 200(OK) 또는 201(Created) 응답 코드와 함께 업데이트된 리소스가 반환됩니다. 만약 요청이 성공적으로 처리되었지만 응답 본문이 필요 없을 경우, 204(No Content) 응답 코드가 사용될 수 있습니다. 요청이 잘못되었거나 실패하면 400(Bad Request) 또는 404(Not Found) 응답 코드가 반환될 수 있습니다. **PUT** 메서드는 멱등성을 보장합니다. 즉, 같은 요청을 여러 번 보내더라도 리소스의 상태는 동일하게 유지되며, 이를 통해 리소스를 완전히 대체하거나 새로 생성할 수 있습니다.

**리소스의 부분적인 수정을 수행할 때**는 **PATCH** 메서드를 사용합니다. **PATCH** 요청이 성공하면 200(OK) 응답 코드와 함께 업데이트된 리소스가 반환됩니다. 만약 요청이 잘못되었거나 실패하면 400(Bad Request) 또는 404(Not Found) 응답 코드가 반환될 수 있습니다. **PATCH** 메서드는 멱등성을 보장하지 않으며, 같은 요청을 여러 번 수행하면 리소스의 상태가 달라질 수 있습니다.

**리소스를 삭제할 때**는 **DELETE** 메서드를 사용합니다. **DELETE** 요청이 성공하면 204(No Content) 응답 코드가 반환되며, 요청한 리소스가 서버에서 삭제됩니다. 리소스가 존재하지 않으면 404(Not Found) 응답 코드가 반환될 수 있습니다. 요청이 잘못되었거나 실패하면 400(Bad Request) 응답 코드가 사용될 수 있습니다. **DELETE** 메서드는 멱등성을 보장합니다. 즉, 같은 요청을 여러 번 보내더라도 결과는 동일하게 유지되며, 리소스는 삭제된 상태로 남게 됩니다.

행위는 표준 HTTP 메서드를 통해 리소스를 제어하는 방식으로 표현하는 것이 권장됩니다. 예를 들어, 사용자가 비밀번호를 분실하고 비밀번호를 초기화하려는 경우, **PATCH** 메서드를 사용하여 사용자 리소스에서 비밀번호 정보를 제거하는 요청을 보낼 수 있습니다. 서버는 비밀번호가 제거된 것을 감지한 후, 등록된 연락처로 새로운 비밀번호를 발급할 수 있는 절차를 안내하는 메시지를 보낼 수 있습니다.

일부 행위는 HTTP 메서드로 직접 표현하기 어려울 수 있습니다. 이럴 때는 행위를 선언적으로 표현할 수 있는 리소스를 정의하고, 이를 통해 HTTP 메서드를 사용해 제어할 수 있습니다. 

**예시:**

```
POST /search
```

### 표현

클라이언트는 서버와 협상하여 동일한 리소스를 다양한 형식으로 요청할 수 있습니다. 클라이언트는 `Accept-Language` 헤더를 통해 원하는 언어를 서버에 전달하고, 서버는 `Content-Language` 헤더를 사용해 응답의 언어를 명시합니다. 이렇게 하면 클라이언트는 데이터를 올바르게 이해할 수 있습니다. 

또 다른 예로, `Accept-Encoding` 헤더를 통해 클라이언트가 지원하는 압축 방식을 서버에 알려주면, 서버는 이 정보를 바탕으로 데이터를 압축하여 전송합니다. 응답 본문이 어떤 방식으로 압축되었는지는 `Content-Encoding` 헤더로 알려주어 클라이언트가 이를 해제할 수 있게 합니다. 

클라이언트는 `Accept` 헤더를 사용해 원하는 데이터 형식을 요청하고, 서버는 `Content-Type` 헤더로 응답 본문의 데이터 형식을 알려줍니다. 이러한 과정은 클라이언트와 서버 간의 데이터 전송을 유연하고 효율적으로 조정해 줍니다.

API는 시간이 지남에 따라 변경될 수 있으며, 변경이 있더라도 최대한 호환성을 유지하는 것이 중요합니다. 서버는 데이터를 유연하게 수용하고, 응답 데이터는 일관되게 유지해야 합니다. 예를 들어, 리소스에 새로운 필드가 추가될 경우, 기존 클라이언트가 문제없이 처리할 수 있도록 해당 필드에 기본 값을 설정할 수 있습니다. 또한, 응답 데이터의 표현 방식은 변경되지 않도록 유지하여 클라이언트가 변경된 응답을 잘 처리할 수 있도록 해야 합니다.

만약 `limit`과 `offset` 쿼리 매개변수를 지원하지 않던 리소스가 페이지네이션을 지원하게 되더라도, 기존 방식과 호환되도록 응답을 유지해야 합니다. 페이지네이션 관련 추가 정보는 응답 본문에 포함하지 않고 응답 헤더에 추가하여 응답 본문의 구조를 그대로 유지하는 것이 좋습니다.

**예시:**

```
GET /orders?limit=50&offset=100 HTTP/1.1
Host: api.example.com
Accept: application/json
```

**응답 본문:**

```json
[
    {
        "id": 12345,
        "user_id": 67890,
        "product_id": 11223,
        "created_at": 1726194623
    },
    {
        "id": 12346,
        "user_id": 67891,
        "product_id": 11224,
        "created_at": 1726194623
    }
]
```

**응답 헤더:**

```
Link: <https://api.example.com/orders?limit=50&offset=150>; rel="next"
X-Total-Count: 1000
```

API 호환성을 유지하기 어려울 때는 여러 버전으로 API를 관리해야 합니다. 버전 관리는 URI에 버전 정보를 포함시키거나, 헤더에 담거나, 쿼리 매개변수로 전달할 수 있습니다.

URI에 버전 정보를 포함하면 새로운 버전을 쉽게 제공할 수 있지만, 이 경우 캐시된 데이터가 무효화될 수 있고, URI가 많아져 관리가 어려울 수 있습니다.

**예시:**

```
GET /v1/users
```

헤더로 API 버전 정보를 전달하면 담으면 구조를 일관되게 유지할 수 있습니다. 하지만 이 방법은 대규모 변경에 대응하기 어려울 수 있으며, 다양한 버전의 캐시가 혼동을 줄 수 있습니다.

**예시:**

```
GET /users HTTP/1.1
Host: api.example.com
Accept-Version: v1
```

쿼리 매개변수로 API 버전 정보를 전달하면 URI 구조를 일관되게 유지할 수 있습니다. 클라이언트가 쉽게 버전을 조정할 수 있는 장점이 있지만, 마찬가지로 다양한 버전의 캐시가 혼동을 줄 수 있습니다.

**예시:**

```
GET /users?version=v1 HTTP/1.1
Host: api.example.com
```

API의 메이저 버전은 URI로 관리하고, 마이너 버전은 헤더나 쿼리 매개변수로 제어하는 방법도 있습니다. 이 방법은 URI의 수를 최소화하면서 복잡한 버전 관리 전략을 적용할 수 있는 장점이 있습니다.

**예시:**

```
GET /v1/users HTTP/1.1
Host: api.example.com
Accept-Version: v1.1.6
```

## 참고 문서

- [Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)
- [논문을 통한 REST에 대한 고찰](https://shoark7.github.io/programming/knowledge/what-is-rest)
- [실용적인 RESTful API 설계에 대해](https://velog.io/@wo_ogie/restful-api-design)