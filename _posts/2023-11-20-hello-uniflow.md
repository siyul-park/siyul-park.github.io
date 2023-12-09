---
layout: single
title:  "안녕, uniflow"
date:   2023-11-20 18:44:43 +0900
categories: low-code
---

[카카오 클라우드 AI 서비스](https://kakaocloud.com)의 미들웨어 개발은 현존하는 지식을 적극 활용하여 새로운 AI 서비스를 선보이기 위한 첫 발걸음이었습니다. 백엔드 AI 모델 팀으로부터 도착한 API를 기반으로 [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)를 활용하여 열정적으로 미들웨어에 추가 요구사항을 구현했습니다. 각 AI 서비스는 설정 파일의 프로퍼티를 통해 미들웨어의 동적인 선택을 가능케 하며, [Spring Cloud Config Server](https://spring.io/projects/spring-cloud-config)가 동적 설정을 가져오는 역할을 했습니다.

처음에는 이미 갖춰진 미들웨어의 훌륭한 기능과 성능에 만족했지만, AI 서비스의 다양화로 각각의 특화된 추가 처리가 필요해지자, 수정이 번거로운 미들웨어보다는 신속한 대응이 가능한 새로운 서비스를 개발하기로 결정했습니다. 각 미들웨어 모듈을 동적으로 연결하고, 스크립트 작성이 가능한 Low-Code 서비스를 개발하여 복잡한 데이터 처리에 효과적으로 대응했습니다. 이 모든 도전과 실험은 PoC 단계부터 네이티브로 작성된 코드와 거의 차이 없이 탁월한 성과를 보여주었습니다.

이 멋진 경험은 Low-Code의 세계에서의 첫 발걸음이었습니다.

# 안녕, uniflow
[uniflow](https://github.com/siyul-park/uniflow)는 강력한 Low-Code FaaS 서비스로, 자체 인프라를 신경쓰지 않고 적은 양의 코드로 다양한 데이터 소스를 통합할 수 있습니다. 높은 확장성과 성능을 제공하며 다양한 추상화 관점을 통해 시각화를 가장 효과적으로 제공합니다.

이 독특한 서비스의 핵심은 연산 유닛인 노드입니다. 각각의 노드들은 서로 포트를 통해 연결되어 있으며, 포트 간 패킷을 주고받아 상호 작용합니다. 각 노드는 독립적인 루프를 가지고 있어 패킷을 효과적으로 처리합니다. 또한, 각 연산 흐름은 프로세스로 추상화되며 각 프로세스는 독립적으로 처리됩니다. 이 구조는 uniflow가 데이터 처리와 흐름을 효율적으로 관리하며, 뛰어난 유연성을 제공하는 핵심 메커니즘 중 하나입니다.

## 설치와 사용
먼저, uniflow를 사용하기 위해 Go를 1.21 버전 이상으로 다운로드하고 설치합니다. 그 후, GitHub에서 프로젝트를 클론하고 초기화합니다.

```bash
git clone https://github.com/siyul-park/uniflow
cd uniflow
make init
```

명령어를 실행하여, 프로젝트를 빌드합니다.

```bash
make build
```

빌드 결과물은 /dist 디렉토리에 생성됩니다.

```bash
ls /dist
uniflow
```

실행할 [ping 예제](https://github.com/siyul-park/uniflow/blob/main/examples/ping.yaml)를 확인해봅시다.

```yaml
- kind: http
  name: http
  address: :8000
  links:
    io:
      - name: router
        port: in

- kind: router
  name: router
  routes:
    - method: GET
      path: /ping
      port: out[0]
  links:
    out[0]:
      - name: pong
        port: io

- kind: snippet
  name: pong
  lang: json
  code: |
    "pong"
```

`http` 노드는 http server를 :8000에서 실행하며, 패킷을 `router` 노드로 전달합니다. `router` 노드는 패킷이 등록된 경로와 동일한지 확인한 후 `pong` 노드로 패킷을 전달 합니다. 그 후 `pong` 노드는 "pong"을 응답합니다.

이제 uniflow를 시작해봅시다. 

```bash
./dist/uniflow start --boot example/ping.yaml
```

시작한 uniflow가 정상적으로 HTTP 엔드포인트를 제공하는지 확인합시다.

```bash
curl localhost:8000/ping
pong#
```

## 조금 더, 안으로

좀 더 흥미로운 [boot 예제](https://github.com/siyul-park/uniflow/blob/main/examples/boot.yaml) 을 살펴 봅시다.

```yaml
...

- kind: router
  name: router
  namespace: system
  routes:
    ...
    - method: GET
      path: /v1/nodes
      port: out[1]
   ...
  links:
    ...
    out[1]:
      - name: get_nodes
        port: in
    ...

...

- kind: snippet
  name: get_nodes
  namespace: system
  lang: json
  code: |
    null
  links:
    out:
      - name: select
        port: io

...

- kind: reflect
  name: select
  namespace: system
  op: select
  links:
    error:
      - name: not_found
        port: io

...

- kind: snippet
  name: not_found
  namespace: system
  lang: json
  code: |
    { 
      "body": "Not Found",
      "status": 404
    }
```

`boot`은 초기 설치를 권장하는 API 서버와 다양한 노드 명세를 포함하고 있습니다.

새로운 명세 중에서 특히 강조되는 부분은 네임스페이스입니다. 네임스페이스는 리소스를 그룹화하여 격리하는 역할을 하며, `system` 네임스페이스는 자체 운영을 위해 예약되어 있습니다. uniflow를 실행할 때 명시한 네임스페이스에 속한 리소스만을 실행할 수 있어 더욱 효율적인 운영을 가능케 합니다.

`reflect` 노드는 uniflow 런타임에 설치된 노드를 동적으로 제어합니다. 이 기능은 시스템의 상태나 동작을 런타임 중에 신속하게 조정할 수 있도록 도와줍니다.

에러 처리를 위한 `error` 포트는 노드에서 에러가 발생했을 때 중요한 역할을 수행합니다. 포트가 연결되어 있으면 에러가 해당 포트로 전달되어 처리되며, 연결이 되어 있지 않으면 에러는 처리 흐름을 거슬러 올라가 전파됩니다. 프로세스는 이와 함께 모든 처리 흐름이 완료되면 종료됩니다. 이렇게 설계된 에러 처리 메커니즘은 uniflow의 안정성과 신뢰성을 강화하는 핵심적인 부분입니다.

```yaml
- kind: snippet
  name: patch_node
  namespace: system
  lang: jsonata
  code: |
    $merge([{ "id": $.params.node_id }, $.body])
  links:
    out:
      - name: update
        port: io
```

위 코드는 `snippet` 노드의 설정을 담고 있습니다. 여기서는 경량 언어인 [jsonata](https://jsonata.org)를 활용하여 JSON 데이터를 변환하고 질의하는 작업을 처리합니다.

[jsonata](https://jsonata.org)는 uniflow에서 제공하는 가장 중요한 언어입니다. JSON 데이터를 변환하고 질의 하기 위해서 `switch` 노드 등에서도 기본 언어로 사용이 됩니다.

```yaml
- kind: switch
  match:
    - when: '$.permission = "admin"'
      port: out[0]
  links:
    out[0]:
      - name: admin
        port: in
```

`snippet` 노드는 다양한 언어를 지원하며, 현재는 javascript와 typescript도 사용 가능합니다. 이 노드는 내장된 인터프리터를 활용하여 언어를 실행하며, 이를 통해 시스템의 효율성을 높이고 Inter-Process Communication (IPC)에 소요되는 시간을 최소화합니다.

내장된 인터프리터를 사용함으로써 다른 프로세스를 경유하지 않고 데이터 처리를 수행할 수 있어 성능적 이점을 얻을 수 있습니다. 그러나 연산이 많이 필요한 경우 외부 런타임을 선택할 수도 있습니다. 이는 선택적으로 적용 가능한 유연성을 제공하며, 런타임 성능에 따라 선택을 조절할 수 있습니다.

`boot`를 실행하고 설치된 노드를 조회해보겠습니다.

```bash
./dist/uniflow start --boot example/boot.yaml --namespace system
```

```bash
curl localhost:8000/v1/nodes                                                      
[{"id":"01HFVFR8AQQW80FW9C40V4NWDJ","kind":"reflect","links":{"error":[{"name":"not_found","port":"io"}]},"name":"insert","namespace":"system","op":"insert"},{"id":"01HFVFR8B05TJCKGDEP1J8Q5EC","kind":"reflect","links":{"error":[{"name":"not_found","port":"io"}]},"name":"update","namespace":"system","op":"update"},...]#
```

uniflow에는 다양하고 흥미로운 [예제](https://github.com/siyul-park/uniflow/tree/main/examples)들이 포함되어 있습니다. 이 예제들을 통해 uniflow의 세계를 탐험하며 놀라운 여정을 시작해보세요! 🚀🌟