---
title: "흐름 기반 프로그래밍의 에러 처리와 응답 반환 구현"
date: 2024-03-01 12:00:00 +0900
categories: "low-code"
tags: ["low-code", "golang", "flow-based-programming", "ddd"]
---

[uniflow](https://github.com/siyul-park/uniflow)는 [흐름 기반 프로그래밍 (FBP, Flow-Based Programming)](https://en.m.wikipedia.org/wiki/Flow-based_programming)을 기반으로 하는 로우 코드 엔진입니다.

이 프로젝트의 목적은 하나의 서비스를 구성하기 위해 여러 노드로 구성된 네트워크를 정의하고, 각 노드들끼리 포트를 통해 서로 연결하며, 정의된 연결을 통해 데이터 패킷을 교환하는 것입니다.

패킷이 전달되기 위해서는 명시적으로 포트들이 연결되어 있어야 합니다. 그러나 에러 처리와 같이 정상 흐름과 분리되는 특수한 경우에는 연결 구성이 더 복잡해집니다. 에러 처리를 위해 정상 처리를 위한 연결과는 반대 방향으로 노드들을 연결해야 하며, 이로 인해 연결에 상호 참조가 발생하여 명세를 수정하기 어려워지는 문제가 발생합니다.

```yaml
- kind: http
  name: server
  address: :8000
  links:
    out:
      - name: route
        port: in

- kind: route
  name: route
  routes:
    - method: GET
      path: /ping
      port: out[0]
  links:
    out[0]:
      - name: pong
        port: in
    error:
      - name: server
        port: in

- kind: snippet
  name: pong
  lang: text
  code: pong
  links:
    out:
      - name: server
        port: in  
```

의존 관계를 줄이고 명세를 간결하게 관리하기 위해 발생한 에러는 처리 흐름을 거슬러 올라가며 전파되어야 했으며, 또한 패킷의 최종 처리 결과도 마찬가지로 흐름을 거슬러 올라가며 전달되어야 했습니다.

이에 따라 `error` 포트에 다른 포트가 연결되지 않은 경우에는 에러를 요청자에게 전파하고, 결과가 반환되는 지점을 나타내는 `io` 포트를 도입하게 되었습니다. 이를 통해 처리 흐름을 간결하게 만들 수 있었습니다.

```yaml
- kind: http
  name: server
  address: :8000
  links:
    out:
      - name: route
        port: in

- kind: route
  name: route
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
  lang: text
  code: pong
```

## 그래프 구조 스택을 활용한 패킷 추적

이 기능을 구현하기 위해서는 패킷의 움직임을 추적하고 각 패킷이 어떤 포트들을 통과했는지를 관리해야 했습니다. 노드가 여러 개의 다른 노드와 연결되어 있고, 노드가 패킷을 비동기적으로 처리할 수 있기 때문에 패킷을 추적하기 위해 일반적인 스택이 아닌 [그래프 구조 스택 (Graph-Structured Stack)](https://en.m.wikipedia.org/wiki/Graph-structured_stack)의 변형을 사용했습니다.

이 변형된 스택은 패킷의 움직임을 추적하는 그래프 구조와 각 패킷이 어떤 노드와 어떤 포트를 통해 이동했는지를 개별적으로 추적하는 스택 구조를 결합하고 있습니다.

**그래프:**
```
[1] -> [2] -> [3]
   +-> [4] -> [5]
```

**스택:**
```
[1] -> (A, B)
[2] -> (C, D)
[3] -> (E, F)
[4] -> (G, H)
[5] -> (I, J)
```

패킷 [5]를 반환하기 위해, 노드는 먼저 스택에서 (J)를 확인하여 자신의 출력 포트인지를 판단합니다. 그런 다음 (J)를 제거하고, 스택에서 다음 요소인 (I)가 자신의 입력 포트인지를 확인하고 스택에서 제거합니다. 그 후에 패킷은 (I)에 연결된 다른 포트들에게 전파됩니다. 전파된 포트들 중에 스택의 다음 요소인 (H)와 같은 포트가 있다면, 해당 요소도 제거되고 입력 포트를 확인한 후 패킷을 다시 전파합니다. 이 과정을 거치면 그래프와 스택은 변경됩니다.

**변경된 그래프:**
```
[1] -> [2] -> [3]
   +-> [4] -> [5]
```

**변경된 스택:**
```
[2] -> (C, D)
[3] -> (E, F)
```

패킷 [3]을 반환할 경우, [1] 패킷을 생성한 노드는 이미 응답을 받아 스택에서 값이 제거되었기 때문에, 패킷이 반환될 때 [2] 패킷을 생성한 노드까지 패킷이 반환되어 스택이 비게 됩니다.

## 초기 설계의 한계

동일한 패킷이 여러 노드에서 동시에 처리될 수 있다고 생각했지만, 패킷을 공유하면서 스택도 함께 공유되어, 스택에 예상치 못한 순서로 여러 노드에 대한 포트 정보가 기록되었습니다. 이로 인해 반환될 포트를 식별하기 어려워졌습니다. 이 문제를 해결하기 위해 패킷을 여러 자식으로 분화시켜 서로 다른 노드로 전달해야 했습니다.

또한 초기 설계에서는 입/출력 포트가 명확히 구분되지 않았고, 포트는 단순히 양방향으로 패킷을 전달받을 수 있는 통로 역할을 수행했습니다. 패킷의 처리와 결과 반환은 포트의 역할이 아닌 노드의 역할이었습니다. 따라서 패킷과 포트의 계보를 스택과 그래프를 활용하여 노드에서 직접 처리해야 했습니다.

```go
// In 포트에서 Out 포트로 순방향 전파
outStream := n.outPort.Open(proc)

for {
	inPck, ok := <-inStream.Receive()
	if !ok {
		return
	}

	// 패킷 처리...

	proc.Graph().Add(inPck.ID(), outPck.ID())
	if outStream.Links() > 0 {
		proc.Stack().Push(outPck.ID(), inStream.ID())
		outStream.Send(outPck)
	} else {
		proc.Stack().Clear(inPck.ID())
	}
}
```

연산이 복잡해짐에 따라 문제가 발생했습니다. 패킷의 전송과 응답에 대한 개념이 명확하지 않아 노드 내에서 혼란이 생기고, 패킷 처리 과정을 설명하기 위해 많은 보일러플레이트가 필요했습니다. 이로 인해 구현에 개념적 중복이 발생했습니다. 그리고 연산의 과정만이 언어에 나타나고 명확한 용어가 없어서 커뮤니케이션 비용이 증가했습니다.

과정을 더 쉽고 명확하게 표현하고 구현을 이해하기 쉽게 하기 위해 새로운 심층적인 모델을 탐구해야 했습니다. 점차적으로 전송과 응답이라는 개념이 언어상에서 명확히 표현되기 시작했습니다.

## 더 깊은 모델을 향해서

첫 번째로 알아차린 점은 포트가 방향성을 갖는다는 것입니다. 즉, 출력 포트는 입력 포트와 연결되며, 패킷은 출력 포트를 통해 입력 포트로 전달됩니다. 이를 순방향 전달이라고 정의했습니다. 반면에 역방향 전달은 처리된 응답 패킷이 입력 포트에서 출력 포트로 반환됩니다. 이러한 구조로 인해 포트를 명확히 입력과 출력으로 구분하기로 결정했습니다.

```go
// Node represents an operational unit that processes packets.
type Node interface {
	In(name string) *port.InPort
	Out(name string) *port.OutPort
	Close() error
}
```

또한, 그래프 구조 스택에서 그래프의 각 노드마다 대부분 하나의 스택 요소를 가지고 있다는 것을 발견했습니다. 패킷이 안전하게 처리되고 추적되기 위해 여러 노드가 동시에 공유해서는 안되었고 대부분의 경우 하나의 포트만이 스택에 저장되었습니다.

따라서 별도로 처리되던 그래프 구조와 스택 구조를 통합하여, 패킷의 계보만 스택에 저장하고, 포트가 자신이 응답받아야 하는 패킷들을 직접 저장하고 관리하게 수정되었습니다.

**그래프 구조 스택의 데이터:**
```
[1] -> [2] -> [3]
   +-> [4] -> [5]
```

**그래프 구조 스택의 헤드:**
```
[1] -> [1]
[2] -> [2]
[3] -> [3]
[4] -> [4]
[5] -> [5]
```

**포트가 보낸 패킷:**
```
(A) -> [1]
(B) -> [2]
(C) -> [3]
(D) -> [4]
(E) -> [5]
```

패킷 [5]를 반환하기 위해, 포트 (E)는 자신이 보낸 패킷들과 응답 패킷인 [5] 패킷이 그래프에서 연결되어 있는지 확인합니다. 보낸 패킷 중에서 가장 작은 거리를 가지는 패킷을 선택하는데 이 경우에는 응답 패킷과 동일한 [5] 패킷이 선택됩니다. 그런 다음 그래프 구조 스택에서 [5] 패킷을 제거하고 응답 패킷의 헤더를 이동하여 자신이 보낸 패킷의 위치 전까지 스택 요소를 제거합니다.

**그래프 구조 스택의 데이터:**
```
[1] -> [2] -> [3]
   +-> [4]
```

**그래프 구조 스택의 헤드:**
```
[1] -> [1]
[2] -> [2]
[3] -> [3]
[4] -> [4]
[5] -> [4]
```

**포트가 보낸 패킷:**
```
(A) -> [1]
(B) -> [2]
(C) -> [3]
(D) -> [4]
(E) -> []
```

이 과정을 (D), (A) 포트에서 반복한다면 그래프 구조 스택과 포트에 저장된 데이터는 아래와 같이 변화합니다.

**그래프 구조 스택의 데이터:**
```
[2] -> [3]
```

**그래프 구조 스택의 헤드:**
```
[2] -> [2]
[3] -> [3]
```

**포트가 보낸 패킷:**
```
(A) -> []
(B) -> [2]
(C) -> [3]
(D) -> []
(E) -> []
```

이렇게 변경된 구현은 명확하게 패킷의 작성과 읽기, 그리고 응답이란 과정을 표현하게 되었고, 기존에 노드들에 흩어져 있던 지식은 응집되었습니다.

```go
// In 포트에서 Out 포트로 순방향 전파
inReader := n.inPort.Open(proc)
outWriter := n.outPort.Open(proc)

for {
	inPck, ok := <-inReader.Read()
	if !ok {
		return
	}

	// 패킷 처리...

	proc.Stack().Add(inPck, outPck)
	outWriter.Write(outPck)
}
```

```go
// Out 포트에서 In 포트로 역방향 전파
inReader := n.inPort.Open(proc)
outWriter := n.outPort.Open(proc)

for {
	backPck, ok := <-outWriter.Receive()
	if !ok {
		return
	}

	inReader.Receive(backPck)
}
```

[AB(Apache HTTP server benchmarking tool)](https://httpd.apache.org/docs/)사용하여 두 버전의 성능을 비교했을 때 거의 동일한 성능이 유지되었습니다.

```shell
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /ping
Document Length:        4 bytes

Concurrency Level:      256
Time taken for tests:   0.366 seconds
Complete requests:      1024
Failed requests:        0
Total transferred:      134144 bytes
HTML transferred:       4096 bytes
Requests per second:    2796.68 [#/sec] (mean)
Time per request:       91.537 [ms] (mean)
Time per request:       0.358 [ms] (mean, across all concurrent requests)
Transfer rate:          357.78 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   3.8      0      13
Processing:     1   84  40.5     92     159
Waiting:        1   84  40.5     92     158
Total:          1   86  41.9     92     170

Percentage of the requests served within a certain time (ms)
  50%     92
  66%    106
  75%    112
  80%    116
  90%    140
  95%    158
  98%    161
  99%    164
 100%    170 (longest request)
```

## 그 이후에

FBP에서는 여러 머신으로 이루어진 네트워크와 유사하게 독립적인 노드 간 비동기적 상호작용을 통해 연산을 처리합니다. 이러한 구조에서도 에러 전파와 응답 반환과 같은 기능이 필요합니다. 하지만 패킷 전달이 그래프처럼 분화되며 이루어지므로 이를 위해 스택 프레임과 같은 방식 대신 복잡한 구조를 가지는 그래프 구조 스택을 사용하여 전체 패킷을 추적해야 했습니다.

실제 네트워크와 유사하게 패킷에 출발지 주소와 목적지 주소를 저장하고 라우팅 게이트웨이를 통해 패킷을 전달하는 방법도 존재했지만, 이미 언어 상에 존재하는 노드, 포트, 그리고 패킷 외에 추가적인 모델을 도입하기에는 지식이 충분히 성숙해지지 않았습니다. 또한 연결을 통해 패킷이 전달된다는 개념을 직관적으로 표현하고 요청 패킷의 생산자 측에서 손쉽게 응답 패킷을 처리하기 쉽게 기존의 도메인 모델에 연산을 통합하게 되었습니다.

데이터 연산을 위한 흐름 외에 노드 간의 정보를 교환하는 방식이 더욱 필요해지면 여러 계층으로 분할하고 ARP, RARP와 같은 여러 프로토콜을 지원하도록 변경이 이루어질 수 있을 것으로 보입니다.
