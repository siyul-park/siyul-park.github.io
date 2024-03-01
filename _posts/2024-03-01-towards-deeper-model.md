---
title: "더 깊은 심층 모델을 향해: uniflow 리팩토링 일지"
date: 2024-03-01 12:00:00 +0900
categories: "low-code"
tags: ["low-code", "golang", "ddd"]
---

[uniflow](https://github.com/siyul-park/uniflow)는 연산의 단위인 노드들을 포트를 통해 서로 연결하여 패킷을 전달하여 백엔드 워크플로우를 만드는 로우 코드 엔진입니다. 보통은 명시적으로 노드 간 연결을 정의하여 패킷의 처리 결과를 전달하지만, 사용자 편의성과 에러의 전파를 위해 암시적으로 패킷을 생산한 노드들에게 반환할 수도 있습니다.

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

요청이 발생하면 `server`에서 `route`로 패킷이 전달되고, 이후에 `route`에서 `pong`으로 패킷이 전달됩니다. `pong`에서 생성된 데이터는 명시적으로 `server`와 연결되어 있지 않더라도 호출 스택을 거슬러 올라가 `server`까지 전달됩니다.

명시적으로 선언한다면 다음과 같이 작성할 수 있습니다:

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

- kind: snippet
  name: pong
  lang: text
  code: pong
  links:
    out:
      - name: server
        port: in  
```

## 그래프 구조 스택을 활용한 패킷 추적

한 노드가 여러 개의 노드와 연결되어 있고, 노드가 패킷을 비동기적으로 처리할 수 있어 패킷을 추적하기 위해 일반적인 스택 대신 [그래프 구조 스택](https://en.m.wikipedia.org/wiki/Graph-structured_stack)의 변형을 사용합니다. 이 변형된 스택은 패킷의 계보를 추적하는 그래프 구조와 각 패킷이 어떤 노드와 어떤 포트를 통해 이동했는지를 개별적으로 추적하는 스택 구조를 가지고 있습니다.

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

초기 설계에서는 입/출력 포트가 명확하게 구분되지 않았으며, 포트는 단순히 양방향으로 패킷을 전달받을 수 있는 통로 역할을 수행했습니다. 패킷을 처리하고 결과를 전송하고 반환받는 과정은 포트의 역할이 아닌 노드의 역할이었습니다. 이로 인해 스택을 활용하여 패킷과 포트의 계보를 노드에서 직접 처리해야 했습니다.

```go
// Node represents an operational unit that processes packets.
type Node interface {
	Port(name string) *port.Port 
	Close() error                
}
```

```go
errStream := n.errPort.Open(proc)

if inStream != outStream {
    outStream.AddSendHook(port.SendHookFunc(func(pck *packet.Packet) {
        proc.Stack().Push(pck.ID(), outStream.ID())
    }))
}
errStream.AddSendHook(port.SendHookFunc(func(pck *packet.Packet) {
    proc.Stack().Push(pck.ID(), errStream.ID())
}))

for {
    inPck, ok := <-inStream.Receive()
    if !ok {
        return
    }

    forward := func(outStream *port.Stream, outPck *packet.Packet, backward bool) {
        proc.Graph().Add(inPck.ID(), outPck.ID())
        if outStream.Links() > 0 {
            if outStream != inStream {
                proc.Stack().Push(outPck.ID(), inStream.ID())
            }
            outStream.Send(outPck)
        } else if backward {
            inStream.Send(outPck)
        } else {
            proc.Stack().Clear(outPck.ID())
        }
    }

    if outPck, errPck := n.action(proc, inPck); errPck != nil {
        forward(errStream, errPck, true)
    } else if outPck != nil {
        forward(outStream, outPck, false)
    } else {
        proc.Stack().Clear(inPck.ID())
    }
}
```

암시적 응답이나 에러 처리와 같은 부가적인 도메인 개념이 명확해짐에 따라 문제가 발생했습니다. 패킷의 전송과 응답에 대한 개념이 정제되지 못하고 노드 내에서 떠돌게 되었고 노드 내에서 패킷 처리 과정을 설명하기 위해서는 많은 상세 내용을 설명해야 했습니다. 점차적으로 구현이 복잡해지면서 커뮤니케이션 비용이 증가하고, 언어의 한계가 더욱 뚜렷해졌습니다.

과정을 더 쉽고 명확하게 설명해야 했고, 언어를 보다 명확하게 표현하고 구현을 이해하기 쉽게 만들기 위해 새로운 심층적인 모델을 탐구했습니다. 점차적으로 전송과 응답이라는 개념이 언어상에서 명확히 표현되기 시작했습니다.

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
func TestStack_Add(t *testing.T) {
	s := newStack()
	defer s.Close()

	pck1 := packet.New(nil)
	pck2 := packet.New(nil)

	s.Add(nil, pck1)
	assert.True(t, s.Has(nil, pck1))

	s.Add(nil, pck2)
	assert.True(t, s.Has(nil, pck2))

	s.Add(pck1, pck2)
	assert.True(t, s.Has(pck1, pck2))
}

func TestStack_Unwind(t *testing.T) {
	s := newStack()
	defer s.Close()

	pck1 := packet.New(nil)
	pck2 := packet.New(nil)
	pck3 := packet.New(nil)

	s.Add(pck1, pck2)
	s.Add(pck2, pck3)

	s.Unwind(pck3, pck2)
	assert.False(t, s.Has(pck2, pck3))
	assert.False(t, s.Has(nil, pck2))
	assert.False(t, s.Has(nil, pck3))

	s.Unwind(pck3, pck1)
	assert.False(t, s.Has(nil, pck1))
}

func TestIO_WriteAndRead(t *testing.T) {
	proc := process.New()
	defer proc.Close()

	w := newWriter(proc.Stack(), 0)
	defer w.Close()

	r := newReader(proc.Stack(), 0)
	defer r.Close()

	w.link(r)

	pck1 := packet.New(nil)
	pck2 := packet.New(nil)

	w.Write(pck1)
	w.Write(pck2)

	ctx, cancel := context.WithTimeout(context.TODO(), time.Second)
	defer cancel()

	select {
	case pck := <-r.Read():
		r.Receive(pck)
	case <-ctx.Done():
		assert.NoError(t, ctx.Err())
	}

	select {
	case pck := <-r.Read():
		proc.Stack().Clear(pck)
	case <-ctx.Done():
		assert.NoError(t, ctx.Err())
	}

	select {
	case <-w.Receive():
	case <-ctx.Done():
		assert.NoError(t, ctx.Err())
	}
}
```

변경된 포트는 명확하게 패킷의 작성과 읽기, 그리고 응답이란 과정을 표현하게 되었고, 기존에 노드들에 흩어져 있던 지식은 응집되었습니다.

```go
inReader := n.inPort.Open(proc)
outWriter := n.outPort.Open(proc)

for {
    inPck, ok := <-inReader.Read()
    if !ok {
        return
    }

    if outPck, errPck := n.action(proc, inPck); errPck != nil {
        proc.Stack().Add(inPck, errPck)
        n.throw(proc, errPck)
    } else if outPck != nil {
        proc.Stack().Add(inPck, outPck)
        outWriter.Write(outPck)
    } else {
        proc.Stack().Clear(inPck)
    }
}
```

```go
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
