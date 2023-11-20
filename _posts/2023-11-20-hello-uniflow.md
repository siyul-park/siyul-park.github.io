---
layout: single
title:  "Hello, uniflow"
date:   2023-11-20 18:44:43 +0900
categories: uniflow low-code faas
---

Low-Code의 향연이 펼쳐진 순간, [카카오 클라우드 AI 서비스](https://kakaocloud.com)의 미들웨어 개발은 기존의 지혜를 활용해 새로운 AI 서비스를 세상에 선보이기 위한 첫 발걸음이었습니다. [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)를 활용하여 백엔드의 AI 모델 팀으로부터 도착한 API를 기반으로, 미들웨어에 추가 요구사항을 열정적으로 구현하였습니다. 각 AI 서비스는 설정 파일에 담긴 프로퍼티를 통해 어떤 미들웨어를 거쳐야 하는지 결정하며, 동적으로 설정을 가져오는 [Spring Cloud Config Server](https://spring.io/projects/spring-cloud-config)가 이를 가능케 했습니다.

초기에는 이미 갖춰진 미들웨어의 훌륭한 기능과 성능에 만족했지만, AI 서비스의 다양화로 각각의 특화된 추가 처리가 필요해지자, 수정이 번거로운 미들웨어 대신 신속하게 대응할 수 있는 새로운 서비스를 개발하기로 결정했습니다. 각 미들웨어 모듈을 동적으로 연결하고, 스크립트 작성이 가능한 Low-Code 서비스를 개발하여 복잡한 데이터 처리에 효과적으로 대응했습니다. 이 모든 도전과 실험은 PoC 단계부터 네이티브로 작성된 코드와 거의 차이 없이 탁월한 성과를 보여주었습니다.

이 멋진 경험은 Low-Code의 세계에서의 첫 발걸음이었습니다.

# Hello, uniflow
[uniflow](https://github.com/siyul-park/uniflow)는 강력한 Low-Code FaaS 서비스로, 적은 양의 코드로 다양한 데이터 소스를 통합할 수 있어 자체 인프라를 신경쓰지 않아도 됩니다. 높은 확장성과 성능을 제공하여 다양한 추상화 관점을 통해 시각화를 가장 효과적으로 제공합니다.

이 독특한 서비스의 핵심은 연산 유닛인 노드입니다. 각각의 노드들은 서로 포트를 통해 연결되어 있으며, 포트 간 패킷을 주고받아 상호 작용합니다. 각 노드는 독립적인 루프를 가지고 있어 패킷을 효과적으로 처리합니다. 또한, 각 연산 흐름은 프로세스로 추상화되며 각 프로세스는 독립적으로 처리됩니다. 이 구조는 uniflow가 데이터 처리와 흐름을 효율적으로 관리하며, 뛰어난 유연성을 제공하는 핵심 메커니즘 중 하나입니다.

## 설치와 사용
먼저, uniflow를 사용하기 위해 Go를 최소 1.21 버전 이상으로 다운로드하고 설치합니다. 그 후, GitHub에서 프로젝트를 클론하고 초기화합니다.

```bash
git clone https://github.com/siyul-park/uniflow
cd uniflow
make init
```

프로젝트를 빌드하려면 다음 명령어를 사용합니다.

```bash
make build
```

빌드 결과물은 /dist 디렉토리에 생성됩니다.

```bash
ls /dist
uniflow
```

ping 예제를 살펴봅시다!

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
http 노드는 http server를 :8000에서 실행하며, 패킷을 router 노드로 전달합니다. router 노드는 패킷이 등록된 경로와 동일한지 확인한 후 pong 노드로 패킷을 전달 합니다. pong 노드는 최종 결과인 "pong"을 생산하여 응답합니다.

이제 uniflow를 시작해봅시다. 

```bash
./dist/uniflow start --boot example/ping.yaml
```

시작한 uniflow가 정상적으로 HTTP 엔드포인트를 제공하는지 확인합시다.

```bash
curl localhost:8000/ping
pong#
```

uniflow에는 흥미로운 예제들이 많이 포함되어 있습니다. 자세한 내용은 [예제](https://github.com/siyul-park/uniflow/examples)를 참조하세요.

첫걸음이었던 uniflow의 세계, 함께 더 멋진 여정을 떠나보세요! 🚀🌟
