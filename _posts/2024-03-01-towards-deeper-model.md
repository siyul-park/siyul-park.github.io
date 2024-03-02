---
title: "더 깊은 심층 모델을 향해: uniflow 리팩토링 일지"
date: 2024-03-01 12:00:00 +0900
categories: "low-code"
tags: ["low-code", "golang", "ddd"]
---

[uniflow](https://github.com/siyul-park/uniflow)는 로우 코드 엔진으로 [흐름 기반 프로그래밍 (FBP, Flow-Based Programming)](https://en.m.wikipedia.org/wiki/Flow-based_programming)을 기반으로 합니다. 서비스는 여러 노드로 이루어진 네트워크로 정의되며, 각 노드는 포트를 통해 연결됩니다. 이러한 연결은 미리 정의되어 있으며, 데이터 패킷은 이 연결을 통해 교환됩니다.

패킷이 전달되기 위해서는 명시적으로 포트들이 연결되어야 합니다. 하지만 에러 처리와 같이 정상 흐름과 분리되는 흐름이 있을 때 연결은 더욱 복잡해집니다. 에러 처리를 위해 정상 처리를 위한 연결과 반대 방향으로 노드들을 연결해야 했고, 이로 인해 연결에 상호 참조가 발생하여 명세를 수정하기 어려워졌습니다.

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

의존 관계를 줄여 명세를 간결하게 관리하기 위해 패킷이 암시적으로 전달되어야 했습니다. 발생한 에러는 처리 흐름을 거슬러 올라가며 전파되어야 했고, 패킷의 최종 처리 결과도 마찬가지로 흐름을 거슬러 올라가며 전달되어야 했습니다. 

그래서 `error` 포트에 다른 포트가 연결되지 않은 경우, 에러를 요청자에게 전파하고, 결과가 반환되는 지점을 나타내는 `io` 포트를 도입했습니다.

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

이 기능을 구현하기 위해서는 패킷의 계보를 추적하고 패킷이 어떤 포트들을 통과했는지를 관리해야 했습니다. 한 노드가 여러 개의 노드와 연결되어 있고, 노드가 패킷을 비동기적으로 처리할 수 있어서 패킷을 추적하기 위해 일반적인 스택 대신 [그래프 구조 스택](https://en.m.wikipedia.org/wiki/Graph-structured_stack)의 변형을 사용했습니다. 

이 변형된 스택은 패킷의 계보를 추적하는 그래프 구조와 각 패킷이 어떤 노드와 어떤 포트를 통해 이동했는지를 개별적으로 추적하는 스택 구조를 가지고 있습니다.

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

패킷 [5]를 반환하기 위해, 노드는 스택에서 먼저 (J)를 확인하여 자신의 출력 포트인지를 판단합니다. 그 후에 (J)를 제거하고, 스택에서 다음 요소인 (I)가 자신의 입력 포트인지를 확인하고 스택에서 제거합니다. 패킷은 (I)에 연결된 다른 포트들에게 전파됩니다. 전파된 포트들 중에 스택의 다음 요소인 (H)와 같은 포트가 있다면, 해당 요소도 제거되고 입력 포트를 확인한 후 패킷을 다시 전파합니다. 이 과정을 거치면 그래프와 스택은 변경됩니다.

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

패킷 [3]을 반환할 경우에는 [1] 패킷을 생성한 노드는 이미 응답을 받아 스택에서 값이 제거되었으므로, [2] 패킷을 생성한 노드까지 패킷이 반환되어 스택이 비게 됩니다.

## 초기 설계의 한계

동일한 패킷이 여러 노드에서 동시에 처리될 수 있다고 생각했지만 패킷을 공유하면서 스택도 함께 공유되어, 스택에 예상치 못한 순서로 여러 노드에 대한 포트 정보가 기록되었습니다. 이로 인해 반환될 포트를 식별하기 어려워졌습니다. 이 문제를 해결하기 위해 패킷이 여러 자식으로 분화되어 서로 다른 노드로 전달되어야 했습니다.

또한 초기 설계에서는 입/출력 포트가 명확히 구분되지 않았고, 포트는 단순히 양방향으로 패킷을 전달받을 수 있는 통로 역할을 수행했습니다. 패킷의 처리와 결과 반환은 포트의 역할이 아닌 노드의 역할로 설정되어 역할이었습니다. 패킷과 포트의 계보를 스택과 그래프를 활용하여 노드에서 직접 처리해야 했습니다.

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

연산이 복잡해짐에 따라 문제가 발생했습니다. 패킷의 전송과 응답에 대한 개념이 명확하지 않아 노드 내에서 혼란이 생기고, 패킷 처리 과정을 설명하기 위해 보일러플레이트가 많이 필요했습니다. 이로 인해 개념적 중복이 발생했습니다. 또한, 연산의 과정만이 언어에 나타나고 명확히 정제된 언어가 없어서 커뮤니케이션 비용이 증가했습니다.

과정을 보다 쉽고 명확하게 언어애 표현하고 구현을 이해하기 쉽게 만들기 위해 새로운 심층적인 모델을 탐구해야 했습니다. 점차적으로 전송과 응답이라는 개념이 언어상에서 명확히 표현되기 시작했습니다.

## 더 깊은 모델을 향해서

첫 번째로 알아차린 점은 포트가 방향성을 갖는다는 것입니다. 즉, 출력 포트는 입력 포트와 연결되며, 패킷은 출력 포트를 통해 입력 포트로 전달됩니다. 이를 순방향 전달이라고 정의했습니다. 역방향 전달은 이와 반대로 처리한 응답 패킷을 입력 포트에서 출력 포트로 반환합니다. 이로 인해 포트를 입력과 출력으로 명확히 구분하기로 결정했습니다.

```go
// Node represents an operational unit that processes packets.
type Node interface {
	In(name string) *port.InPort
	Out(name string) *port.OutPort
	Close() error
}
```

또한, 그래프 구조 스택에서 그래프의 각 노드마다 대부분 하나의 스택 요소를 가지고 있다는 것을 발견했습니다. 그래프의 각 노드인 패킷은 안전한 처리와 추적을 위해 여러 노드가 같이 공유해서는 안되었고, 대부분 하나의 포트만 스택 요소로 저장하고 있었습니다. 

별도로 처리되던 그래프 구조와 스택 구조를 통합하여 패킷의 계보만 스택에 저장하고, 포트가 자신이 응답받아야 하는 패킷들을 직접 저장하고 관리하게 수정되었습니다.

```go
func (w *Writer) Write(pck *packet.Packet) {
	w.mu.Lock()
	defer w.mu.Unlock()

	if w.pipe.Links() == 0 {
		w.stack.Clear(pck)
		return
	}

	var stem *packet.Packet
	if w.stack.Has(nil, pck) {
		stem = pck
		pck = packet.New(stem.Payload())
	}

	w.written = append(w.written, pck)
	w.stack.Add(stem, pck)
	w.pipe.Write(pck)
}

func (w *Writer) pop(pck *packet.Packet) bool {
	w.mu.Lock()
	defer w.mu.Unlock()

	for len(w.written) > 0 && !w.stack.Has(nil, w.written[0]) {
		w.written = w.written[1:]
	}

	for i := 0; i < len(w.written); i++ {
		if w.stack.Cost(w.written[i], pck) == 0 {
			w.stack.Unwind(pck, w.written[i])
			w.written = append(w.written[:i], w.written[i+1:]...)
			return true
		}
	}
	return false
}
```

변경된 포트는 명확하게 패킷의 작성과 읽기, 그리고 응답이란 과정을 표현하게 되었고, 기존에 노드들에 흩어져 있던 지식은 응집되었습니다.

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

이러한 수정 작업은 2,576 줄의 추가와 3,070 줄의 삭제를 수반했지만, 구현이 언어를 더 잘 표현하게 되었고, 변경에 더 신속하게 대응할 수 있게 되었습니다. [수정 내역](https://github.com/siyul-park/uniflow/pull/107)을 통해 이러한 변화를 확인할 수 있습니다.
