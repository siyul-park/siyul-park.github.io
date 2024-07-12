---
title: "쿠버네티스는 리소스의 변경을 어떻게 감지할까?"
date: 2024-07-12 12:00:00 +0900
categories: "infrastructure"
tags: ["infrastructure", "kubernetes"]
---

쿠버네티스에서 컨트롤러는 지속적으로 리소스를 추적하며, 리소스의 변경을 감지하고 현재 상태를 정의된 원하는 상태에 조정합니다. 이 과정에서 컨트롤러는 쿠버네티스 API 서버를 통해 관련된 하위 리소스를 변경하거나 외부 자원을 제어합니다.

리소스가 수정되면, 이 변경 사항은 쿠버네티스 API 서버를 거쳐 `etcd`에 저장됩니다. `etcd`는 쿠버네티스 클러스터 전체의 상태를 저장하는 분산 키-값 저장소입니다. 변경된 상태는 각 노드에 전달되고, `kubelet`은 이 정보를 기반으로 컨테이너를 적절히 관리하고 실행합니다.

컨트롤러는 쿠버네티스 API 서버와 연결을 유지하면서 리소스의 변경을 감지하게 됩니다.

컨트롤러는 특정 리소스에 대해 API 서버에 `Watch` 요청을 보내어 해당 리소스의 변경 사항을 실시간으로 감시합니다. API 서버는 해당 리소스에 대한 변경 이벤트를 발생할 때마다 이를 컨트롤러에게 스트리밍으로 전달합니다.

```plaintext
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
{
  "type": "MODIFIED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
}
```

```sh
kubectl get pods --watch
```

API 서버는 일정 시간 동안의 변경 사항만을 저장하기 때문에, 해당 리소스 버전이 더 이상 유효하지 않으면 `410 Gone` 응답을 반환합니다. 이 경우 클라이언트는 로컬 캐시를 지우고 새로운 리소스 버전을 얻기 위해 `get` 또는 `list` 작업을 수행합니다. 그 후에는 새로운 리소스 버전에서 변경 사항을 감시하게 됩니다.

컨트롤러와 API 서버는 HTTP를 통해 연결되며, 실시간으로 변경 사항을 컨트롤러에게 전달하기 위해 연결을 끊지 않고 `Transfer-Encoding: chunked` 전송 방식을 사용하여 데이터는 일련의 청크로 전송됩니다. 이 경우 헤더를 전송하는 시점에서 바디의 길이를 결정할 수 없어 `Content-Length` 헤더는 생략되고, 각 청크의 시작 부분에 현재 청크의 길이가 16진수 형식으로 추가됩니다.

HTTP/2에서는 자체적인 효율적인 데이터 스트리밍 메커니즘을 지원하기 때문에 `chunked` 전송은 HTTP/1.1에서만 사용됩니다. HTTP/2를 사용하는 경우, API 서버는 바디를 데이터 프레임으로 전송합니다. 또한, [RFC7230](https://www.rfc-editor.org/rfc/rfc7230#section-4.1.1)에서는 수신자가 인식할 수 없는 청크 확장을 무시해야 한다고 정의하므로, `Transfer-Encoding` 헤더 역시 무시됩니다.

이러한 REST 기반의 이벤트 알림은 단일 생산자가 발행한 이벤트를 여러 소비자가 사용할 때 적합합니다. 그러나 여러 생산자가 여러 소비자에게 이벤트를 전달하려는 경우, 이벤트의 순서를 보장하기 어려워 이러한 방식은 적절하지 않습니다.

쿠버네티스에서는 여러 컨트롤러가 API 서버로 요청을 보내어 모든 리소스 변경을 반영합니다. 따라서 API 서버는 이벤트를 관리하고 발행하는 단일 주체로서 리소스 변경의 순서를 제어할 수 있으며, 이러한 방식으로 여러 컨트롤러에게 적절한 이벤트를 전달할 수 있습니다.

### 참고 문서

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Transfer-Encoding - HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding)
- [도메인 주도 설계 구현, 반 버논](https://product.kyobobook.co.kr/detail/S000000935852)
