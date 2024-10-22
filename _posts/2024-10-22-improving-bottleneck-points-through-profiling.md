---
title: "프로파일링을 통한 병목 지점 개선"
date: 2024-10-22 12:00:00 +0900
categories: "system"
tags: ["system", "profile", "golang"]
---

[Uniflow](https://github.com/siyul-park/uniflow)는 다양한 작업을 효율적으로 관리할 수 있는 범용 워크플로우 엔진입니다. API 서버, 데이터 스트리밍, 일괄 작업, 챗봇 등 여러 유스 케이스를 고려하여 설계되었으며, 최상의 성능을 목표로 하기보다는 다양한 상황에서 높은 성능을 제공하는 데 중점을 두고 있습니다.

워크플로우는 여러 노드가 결합되어 연산을 처리하는 구조로 구성됩니다. 워크플로우가 실행되면 각 노드는 프로세스를 통해 격리된 흐름에서 작업을 수행하며, 노드 간에는 패킷을 주고받으며 작업이 진행됩니다.

API 서버와 같은 특정 상황에서는 요청에 대한 응답이 필수적이며, 이에 따라 요청 패킷에 대한 응답 패킷이 반드시 전달되어야 합니다. 여러 포트를 활용하여 명시적으로 응답 흐름을 정의할 수 있지만, 이 과정에서 노드 간의 상호 참조가 발생하여 워크플로우 명세가 복잡해질 수 있습니다. 이러한 복잡성을 줄이기 위해 암시적 응답 패킷 방식을 채택하였습니다.

패킷을 처리하는 가장 직관적인 방법인 동기적인 함수 호출은 단일 데이터 처리에는 적합하지만, 여러 데이터를 동시에 처리하거나 공유 리소스를 사용할 경우 잠금으로 인해 성능이 저하될 수 있습니다. 각 노드는 프로세스별 데이터 처리 루프를 통해 패킷을 순차적으로 처리하고 결과를 반환합니다. 이러한 방식은 잠금을 최소화하고 파이프라이닝을 가능하게 하여 성능을 향상시킵니다.

또한, 불필요한 오버헤드를 줄이기 위해 컴파일 단계와 런타임 단계를 분리했습니다. 즉, 한 번만 처리하면 되는 연산은 컴파일 단계에서 처리하고, 런타임에서는 필수적인 연산만 수행하도록 설계되었습니다.

높은 성능을 달성하기 위해 지속적으로 벤치마크를 실행하고 변화를 추적하고 있었습니다. 테스트 대상 워크플로우로는 GET /ping 요청에 대한 응답으로 pong을 반환하는 간단한 테스트를 사용하고 있으며, 아파치 벤치마크를 통해 성능을 측정하고 있습니다.

```yaml
- kind: listener
  name: listener
  protocol: http
  port: 8000
  ports:
    out:
      - name: router
        port: in

- kind: router
  name: router
  routes:
    - method: GET
      path: /ping
      port: out[0]
  ports:
    out[0]:
      - name: pong
        port: in

- kind: snippet
  name: pong
  language: text
  code: pong
```

그러나 최근 리팩토링과 오류 수정 후 벤치마크를 실행한 결과, 응답 시간이 0.1ms에서 0.6ms로 약 6배 증가한 것을 확인했습니다. 이 차이가 발생한 원인을 파악하기 위해 추가적인 검토가 필요했습니다.

```sh
ab -n 102400 -c 1024 http://127.0.0.1:8000/ping
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Server Software:        
Server Hostname:        127.0.0.1
Server Port:            8000

Document Path:          /ping
Document Length:        4 bytes

Concurrency Level:      1024
Time taken for tests:   62.555 seconds
Complete requests:      102400
Failed requests:        0
Total transferred:      12288000 bytes
HTML transferred:       409600 bytes
Requests per second:    1636.95 [#/sec] (mean)
Time per request:       625.554 [ms] (mean)
Time per request:       0.611 [ms] (mean, across all concurrent requests)
Transfer rate:          191.83 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   3.8      0      47
Processing:    10  616 300.7    568    1872
Waiting:        1  616 300.6    568    1872
Total:         10  617 300.2    570    1872

Percentage of the requests served within a certain time (ms)
  50%    570
  66%    708
  75%    796
  80%    861
  90%   1040
  95%   1195
  98%   1354
  99%   1423
 100%   1872 (longest request)
```

먼저 CPU 프로파일을 확인하여 어떤 과정이 가장 많은 시간을 소요하는지 분석하였습니다.

```sh
./dist/uniflow start --from-specs examples/ping.yaml --cpuprofile=cpu.prof
```

의심되는 부분은 `(*OutPort).Open`이며 여러 프로세스에서 공유해야 하는 연산을 선형적으로 처리하는 과정입니다. 이 연산은 프로세스가 포트에 통신하기 위해 리더와 라이터를 여는 과정으로, 각 노드가 소유하는 포트는 여러 프로세스에서 공유됩니다. 안전하게 리더와 라이터를 열기 위해 잠금을 활용하여 선형적으로 처리되고 있었습니다.

```sh
go tool pprof cpu.prof 
File: uniflow
Build ID: a9511493c9faf9722f919ee7d4d7fee72bee09fd
Type: cpu
Time: Oct 22, 2024 at 3:54am (EDT)
Duration: 168.58s, Total samples = 577.86s (342.77%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top 100
Showing nodes accounting for 475.62s, 82.31% of 577.86s total
Dropped 966 nodes (cum <= 2.89s)
Showing top 100 nodes out of 168
      flat  flat%   sum%        cum   cum%
    24.39s  4.22%  4.22%     24.39s  4.22%  runtime.(*gList).pop (inline)
    19.04s  3.29%  7.52%     44.25s  7.66%  runtime.scanobject
    16.44s  2.84% 10.36%     17.89s  3.10%  runtime.findObject
    16.27s  2.82% 13.18%     16.28s  2.82%  runtime.(*gQueue).pop (inline)
    15.74s  2.72% 15.90%     30.07s  5.20%  runtime.selectgo
    14.55s  2.52% 18.42%    154.85s 26.80%  runtime.mallocgc
    14.36s  2.49% 20.90%     14.36s  2.49%  runtime.memclrNoHeapPointers
    12.50s  2.16% 23.07%     25.36s  4.39%  runtime.lock2
    12.29s  2.13% 25.19%     12.29s  2.13%  runtime.nanotime (inline)
    12.22s  2.11% 27.31%     12.49s  2.16%  runtime.gopark
    11.95s  2.07% 29.38%     12.02s  2.08%  runtime.gostartcall (inline)
    11.78s  2.04% 31.41%     12.10s  2.09%  runtime.(*waitq).dequeue (inline)
    11.01s  1.91% 33.32%     21.94s  3.80%  runtime.(*unwinder).resolveInternal
     9.81s  1.70% 35.02%        75s 12.98%  github.com/siyul-park/uniflow/pkg/packet.NewWriter.func1
     9.24s  1.60% 36.62%      9.24s  1.60%  runtime.procyield
     8.83s  1.53% 38.14%     57.97s 10.03%  runtime.schedule
     8.71s  1.51% 39.65%     18.80s  3.25%  runtime.pcvalue
     8.66s  1.50% 41.15%     11.63s  2.01%  runtime.acquireSudog
     8.37s  1.45% 42.60%     41.84s  7.24%  runtime.gfget
     8.30s  1.44% 44.03%     23.93s  4.14%  runtime.wakep
     8.26s  1.43% 45.46%     42.29s  7.32%  runtime.closechan
     7.51s  1.30% 46.76%      7.53s  1.30%  sync/atomic.(*Int32).Add (inline)
     7.10s  1.23% 47.99%     14.57s  2.52%  runtime.(*mspan).writeHeapBitsSmall
     6.79s  1.18% 49.17%     15.08s  2.61%  runtime.execute
     6.69s  1.16% 50.33%     12.90s  2.23%  runtime.casgstatus
     6.58s  1.14% 51.46%      7.67s  1.33%  runtime.step
     6.54s  1.13% 52.60%      6.54s  1.13%  runtime.futex
     6.06s  1.05% 53.64%     11.88s  2.06%  runtime.unlock2
     5.69s  0.98% 54.63%      5.69s  0.98%  internal/runtime/syscall.Syscall6
     5.62s  0.97% 55.60%      5.62s  0.97%  runtime.readgstatus (inline)
     5.58s  0.97% 56.57%      5.58s  0.97%  runtime.heapBitsSlice (inline)
     5.22s   0.9% 57.47%     22.85s  3.95%  runtime.chanrecv
     5.18s   0.9% 58.37%      5.84s  1.01%  runtime.stackpoolalloc
     5.08s  0.88% 59.25%      5.08s  0.88%  runtime.nextFreeFast (inline)
     4.93s  0.85% 60.10%      4.98s  0.86%  runtime.(*gcBits).bitp (inline)
     4.87s  0.84% 60.94%      4.89s  0.85%  runtime.(*mspan).heapBitsSmallForAddr
     4.65s   0.8% 61.75%     21.47s  3.72%  sync.(*Mutex).lockSlow
     4.56s  0.79% 62.54%      4.62s   0.8%  runtime.(*semaRoot).dequeue
     4.40s  0.76% 63.30%     12.36s  2.14%  runtime.isSystemGoroutine
     4.31s  0.75% 64.04%      5.07s  0.88%  runtime.findfunc
     4.26s  0.74% 64.78%     17.59s  3.04%  runtime.ready
     3.97s  0.69% 65.47%      4.88s  0.84%  runtime.mapaccess2_fast64
     3.72s  0.64% 66.11%      3.99s  0.69%  runtime.(*gcControllerState).addScannableStack (inline)
     3.67s  0.64% 66.75%     18.97s  3.28%  runtime.semrelease1
     3.62s  0.63% 67.37%     70.13s 12.14%  runtime.deductAssistCredit
     3.38s  0.58% 67.96%      6.86s  1.19%  runtime.runqput
     3.34s  0.58% 68.54%      3.34s  0.58%  runtime.(*gcWork).putFast (inline)
     3.30s  0.57% 69.11%    314.25s 54.38%  github.com/siyul-park/uniflow/pkg/port.(*OutPort).Open
     3.20s  0.55% 69.66%      5.44s  0.94%  runtime.findnull
     3.19s  0.55% 70.21%     13.16s  2.28%  runtime.(*stkframe).getStackMap
     3.15s  0.55% 70.76%     58.45s 10.11%  runtime.makechan
     3.12s  0.54% 71.30%      3.12s  0.54%  internal/runtime/atomic.(*Int32).CompareAndSwap (inline)
     3.05s  0.53% 71.83%     24.98s  4.32%  sync.(*RWMutex).Lock
     3.04s  0.53% 72.35%      8.23s  1.42%  runtime.sellock
     2.49s  0.43% 72.78%     29.58s  5.12%  github.com/siyul-park/uniflow/pkg/packet.(*Writer).Close
     2.47s  0.43% 73.21%      3.09s  0.53%  runtime.mapdelete_fast64
     2.39s  0.41% 73.62%      2.96s  0.51%  runtime.mapassign_fast64ptr
     2.33s   0.4% 74.03%     93.43s 16.17%  runtime.newobject
     2.23s  0.39% 74.41%     64.71s 11.20%  runtime.gcDrainN
     2.22s  0.38% 74.80%      9.72s  1.68%  runtime.scanblock
     2.19s  0.38% 75.18%     24.17s  4.18%  runtime.sweepone
     2.12s  0.37% 75.54%    219.94s 38.06%  runtime.systemstack
     2.05s  0.35% 75.90%     71.47s 12.37%  runtime.newproc1
     1.79s  0.31% 76.21%     34.06s  5.89%  runtime.findRunnable
     1.77s  0.31% 76.51%     13.88s  2.40%  runtime.semacquire1
     1.72s   0.3% 76.81%      7.30s  1.26%  runtime.(*mspan).heapBits
     1.67s  0.29% 77.10%    350.07s 60.58%  github.com/siyul-park/uniflow/pkg/port.(*listener).Accept
     1.60s  0.28% 77.38%     79.39s 13.74%  runtime.mcall
     1.50s  0.26% 77.64%     48.64s  8.42%  runtime.markroot
     1.47s  0.25% 77.89%         4s  0.69%  runtime.stackfree
     1.42s  0.25% 78.14%     15.48s  2.68%  runtime.gdestroy
     1.40s  0.24% 78.38%      3.01s  0.52%  runtime.markrootFreeGStacks
     1.36s  0.24% 78.61%      3.43s  0.59%  runtime.(*spanSet).push
     1.33s  0.23% 78.84%      3.19s  0.55%  runtime.releaseSudog
     1.28s  0.22% 79.07%     53.56s  9.27%  github.com/siyul-park/uniflow/pkg/port.(*OutPort).Open.func1
     1.24s  0.21% 79.28%     20.90s  3.62%  sync.(*Mutex).Unlock (inline)
     1.08s  0.19% 79.47%     23.39s  4.05%  runtime.(*sweepLocked).sweep
     1.04s  0.18% 79.65%    126.02s 21.81%  github.com/siyul-park/uniflow/pkg/packet.NewWriter
     1.03s  0.18% 79.83%     92.25s 15.96%  runtime.newproc
     0.97s  0.17% 79.99%     64.91s 11.23%  github.com/siyul-park/uniflow/pkg/port.Listeners.Accept
     0.93s  0.16% 80.15%      6.82s  1.18%  gcWriteBarrier
     0.93s  0.16% 80.32%     57.69s  9.98%  github.com/siyul-park/uniflow/pkg/process.(*Process).AddExitHook
     0.86s  0.15% 80.46%     90.78s 15.71%  runtime.newproc.func1
     0.84s  0.15% 80.61%      9.12s  1.58%  runtime.adjustframe
     0.83s  0.14% 80.75%      5.72s  0.99%  runtime.gfput
     0.80s  0.14% 80.89%      8.97s  1.55%  runtime.(*unwinder).initAt
     0.76s  0.13% 81.02%     15.33s  2.65%  runtime.heapSetType
     0.69s  0.12% 81.14%     17.61s  3.05%  runtime.(*unwinder).next
     0.62s  0.11% 81.25%     54.72s  9.47%  github.com/siyul-park/uniflow/pkg/process.(*exitHook).Exit
     0.62s  0.11% 81.36%      7.96s  1.38%  runtime.(*timers).check
     0.62s  0.11% 81.46%      9.93s  1.72%  runtime.greyobject
     0.57s 0.099% 81.56%      7.73s  1.34%  runtime.stackalloc
     0.56s 0.097% 81.66%     15.31s  2.65%  runtime.scanframeworker
     0.56s 0.097% 81.76%     19.66s  3.40%  sync.(*Mutex).unlockSlow
     0.55s 0.095% 81.85%      4.29s  0.74%  github.com/siyul-park/uniflow/pkg/process.(*Process).Status
     0.55s 0.095% 81.95%     12.57s  2.18%  runtime.gostartcallfn
     0.53s 0.092% 82.04%     35.97s  6.22%  runtime.(*mcache).refill
     0.52s  0.09% 82.13%     32.17s  5.57%  runtime.gcDrain
     0.52s  0.09% 82.22%      6.36s  1.10%  runtime.wbBufFlush1
     0.51s 0.088% 82.31%     20.15s  3.49%  runtime.chanrecv2
```

워크플로우 처리 흐름은 프로세스를 통해 추상화되어 병렬적으로 진행되지만, 선형 연산이 증가하면 병렬화로 인한 성능 향상이 크게 제한될 수 있습니다. 이는 암달의 법칙에 해당하며, 프로그램의 속도를 높이는 것은 병렬화가 불가능한 부분의 비율에 따라 영향을 받습니다. 예를 들어, 프로그램의 90%가 병렬화 가능하다면 이론적으로 최대 10배의 속도 향상이 가능하지만, 그 이상은 어렵습니다. 따라서 성능을 개선하려면 선형 연산에 소요되는 시간을 최대한 줄이는 것이 중요합니다.

기존 코드를 분석하여 비선형적으로 변경할 수 있는 부분을 우선 식별하였습니다.

```go
func (p *OutPort) Open(proc *process.Process) *packet.Writer {
	p.mu.Lock()

	writer, ok := p.writers[proc]
	if ok {
		p.mu.Unlock()
		return writer
	}

	writer = packet.NewWriter()
	p.writers[proc] = writer

	for _, in := range p.ins {
		reader := in.Open(proc)
		writer.Link(reader)
	}

	openHooks := p.openHooks
	listeners := p.listeners

	p.mu.Unlock()

	proc.AddExitHook(process.ExitFunc(func(_ error) {
		p.mu.Lock()
		defer p.mu.Unlock()

		delete(p.writers, proc)
		writer.Close()
	}))

	openHooks.Open(proc)
	listeners.Accept(proc)

	return writer
}
```

`OutPort`에 연결된 `InPort`에서 `reader`를 열어 `writer`에 연결하는 과정은 선형적으로 진행될 필요가 없었습니다. 따라서 이 과정을 잠금 범위 외부로 이동하였습니다.

```go
func (p *OutPort) Open(proc *process.Process) *packet.Writer {
	p.mu.Lock()

	writer, ok := p.writers[proc]
	if ok {
		p.mu.Unlock()
		return writer
	}

	writer = packet.NewWriter()
	p.writers[proc] = writer

	ins := p.ins
	openHooks := p.openHooks
	listeners := p.listeners

	p.mu.Unlock()

	openHooks.Open(proc)
	listeners.Accept(proc)

	proc.AddExitHook(process.ExitFunc(func(_ error) {
		p.mu.Lock()
		defer p.mu.Unlock()

    delete(p.writers, proc)
		writer.Close()
	}))

	for _, in := range ins {
		reader := in.Open(proc)
		writer.Link(reader)
	}

	return writer
}
```

그러나 이 과정은 프로파일링 결과에서 나타났듯이 시간이 오래 걸리는 연산은 아니었으며, 큰 성능 향상으로 이어지지 않았습니다. 프로파일링에서 `(*Writer).Close` 연산이 많은 시간을 소비하는 것으로 확인되었고, 이 연산이 수행될 때 `OutPort`가 잠기면서 처리 과정이 선형적으로 진행되었습니다. 따라서 해당 연산을 잠금 범위 외부로 이동하였습니다.

```sh
2.49s  0.43% 72.78%     29.58s  5.12%  github.com/siyul-park/uniflow/pkg/packet.(*Writer).Close
```

```go
func (p *OutPort) Open(proc *process.Process) *packet.Writer {
	p.mu.Lock()

	writer, ok := p.writers[proc]
	if ok {
		p.mu.Unlock()
		return writer
	}

	writer = packet.NewWriter()
	p.writers[proc] = writer

	ins := p.ins
	openHooks := p.openHooks
	listeners := p.listeners

	p.mu.Unlock()

	openHooks.Open(proc)
	listeners.Accept(proc)

	proc.AddExitHook(process.ExitFunc(func(_ error) {
		p.mu.Lock()
		delete(p.writers, proc)
		p.mu.Unlock()

		writer.Close()
	}))

	for _, in := range ins {
		reader := in.Open(proc)
		writer.Link(reader)
	}

	return writer
}
```

이러한 변경 후 벤치마크를 실행하여 전체적인 성능을 측정한 결과, 응답 시간이 약 0.1ms로 정상화된 것을 확인할 수 있었습니다.

```sh
ab -n 102400 -c 1024 http://127.0.0.1:8000/ping
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Server Software:        
Server Hostname:        127.0.0.1
Server Port:            8000

Document Path:          /ping
Document Length:        4 bytes

Concurrency Level:      1024
Time taken for tests:   12.932 seconds
Complete requests:      102400
Failed requests:        0
Total transferred:      12288000 bytes
HTML transferred:       409600 bytes
Requests per second:    7918.38 [#/sec] (mean)
Time per request:       129.319 [ms] (mean)
Time per request:       0.126 [ms] (mean, across all concurrent requests)
Transfer rate:          927.93 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   4.5      0      40
Processing:     0  126  60.7    120     535
Waiting:        0  125  60.6    119     534
Total:          0  129  60.7    122     536

Percentage of the requests served within a certain time (ms)
  50%    122
  66%    144
  75%    159
  80%    169
  90%    202
  95%    239
  98%    283
  99%    324
 100%    536 (longest request)
```

다행히 원인을 파악하고 수정을 진행했지만, 문제가 있었던 코드의 변경에 즉각적으로 반응하지 못했습니다. CI에 벤치마크 테스트 과정이 통합되어 있었으나, 수치 변경에 대한 알림이 없어 성능 변화를 감지하지 못한 것으로 보입니다.

앞으로는 성능에 큰 영향을 미치는 특정 메서드들에 대한 벤치마크를 강화하고, 비정상적인 성능 변화가 발생할 경우 CI를 통해 경고를 받아 즉각적으로 대응할 수 있도록 절차를 개선할 계획입니다.
