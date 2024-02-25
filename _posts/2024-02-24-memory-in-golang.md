---
title: "함께 보는 Go 메모리"
date: 2024-02-24 12:00:00 +0900
categories: "system"
tags: ["system", "memory", "golang"]
---

Go는 정적 타입 컴파일 언어의 효율성을 유지하면서도 동적 언어처럼 사용하기 쉽도록 설계되었습니다.

빠른 컴파일 속도와 함께 덕 타이핑(Duck Typing)과 런타임 리플렉션(Reflection)을 지원하여 동적 언어를 사용하는 것과 유사한 경험을 제공합니다. 그리고 적은 수의 예약어를 사용하고 간결하고 단순한 언어를 지향하여 코드를 이해하고 관리하는데 들어가는 비용을 줄입니다. 더불어 매우 작은 크기의 스택(2KB)과 프로그램 카운터(PC) 및 스택 포인터(SP)만을 차지하는 경량 스레드인 고루틴(Goroutine)과 고루틴 간 쉬운 데이터 전달을 위한 채널(Channel)을 통해 높은 수준의 동시성(Concurrency) 모델을 제공합니다.

수천개의 고루틴이 동시에 실행되더라도 안전하고 효율적으로 메모리가 관리되어야 했습니다. 이를 위해 높은 비용이 드는 원자성 연산이 필요한 레퍼런스 카운팅(Reference Counting) 대신 CMS(Concurrent Mark Sweep) 방식의 GC(Garbage Collection)를 사용합니다. 또한 Go는 가상 머신에서 돌아가는 언어가 아니므로 런타임에 관련된 모든 기능이 실행 파일에 내장되어 GC를 포함한 런타임의 기능은 오버헤드를 줄이기 위해 최대한 간결하게 작성되어야 했습니다.

## 메모리 구조

초창기 GC는 매우 끔찍한 성능을 보여주었습니다. 전체 CPU 자원의 25%를 항상 사용하며 STW가 빈번하게 발생하여 런타임이 정지되었습니다. Read Barrier Free Concurrent Coping GC로 개선이 시도되었지만 C로 짜여진 런타임을 Go로 재작성하고 컴파일러의 성능을 개선시키는 것이 우선되어, 병렬적으로 GC가 실행이 가능하도록 변경하는데 그쳤습니다.

이후에는 압축 대신 TCMalloc을 기반으로 한 자체 메모리 관리가 도입되었습니다.

### mspan

이 방식은 메모리를 67가지 다른 크기의 페이지 블록들로 분리합니다. 그런 다음, 동일한 크기의 페이지 블록들을 묶어 페이지의 시작 주소, 크기 및 포함된 페이지 수, sweep 상태를 가지는 이중 연결 리스트인 `mspan`으로 구성합니다. 동일한 페이지 크기에 대해 포인터를 가지는 객체들을 저장하는 `scan`과 포인터가 없어 객체의 의존성을 더 탐색하지 않아도 되는 `nonscan` 두 종류의 `mspan`이 존재합니다.

```go
type mspan struct {
	// ...
	
	next *mspan 
	prev *mspan 
	
	startAddr uintptr
	npages    uintptr

	allocBits  *gcBits
	gcmarkBits *gcBits
	pinnerBits *gcBits

	sweepgen  uint32
	spanclass spanClass     
	
	// ...
}
```

객체의 메타 정보를 저장하기 위해 헤더를 사용하는 대신, `mspan` 내의 비트를 활용합니다. 여기서 `allocBits`는 할당 여부를, `gcmarkBits`는 GC 과정에서 사용되는 정보를 나타내며, `pinnerBits`는 객체의 고정 여부를 나타냅니다. GC의 정리 과정에서 이전의 `gcmarkBits`는 초기화되며, `allocBits`는 `gcmarkBits`로 설정됩니다.

### mcache

그리고 각각의 고루틴에서 잠금 없이 빠르게 메모리를 할당하기 위해 로컬 스레드 캐쉬인 `mcache`가 존재합니다. 

```go
type mcache struct {
	// ...

	tiny       uintptr
	alloc      [numSpanClasses]*mspan 
	stackcache [_NumStackOrders]stackfreelist
	
	// ...
}
```

`mcache`는 크기와 수명이 정해진 객체들을 저장하는 `stackcache`와 16B 보다 작은 객체를 저장하기 위한 `tiny` 그리고 32KB보다 작은 값을 저장하기 위한 `alloc`을 가지고 있습니다. 사용가능한 `mspan`이 없으면 `mcentral`에 요청하여 새로운 `mspan`을 할당받습니다.

### mcentral

동일한 페이지 크기를 가지는 모든 `mspan`들은 `mcentral`로 그룹화 됩니다. `mcentral`은 비어 있는 페이지가 존재하여 할당이 가능한 `partial`과 더 이상 할당이 불가능한 `full`을 가지고 있습니다.

```go
type mcentral struct {
	// ...

	spanclass spanClass
	partial  [2]spanSet
	full     [2]spanSet
	
	// ...
}
```

`partial`와 `full`은 정리되지 않은 `mspan`들과 정리된 `mspan`들로 이뤄져 있습니다. 이 두 그룹은 주기적으로 역할을 교환합니다. 메모리가 아직 사용 중이면 정리되지 않은 그룹에서 메모리를 가져와서 정리된 그룹에 추가합니다. 메모리 할당은 정리된 그룹에서 이루어집니다.

### mheap

모든 종류의 `mcentral`이 모여 힙을 형성합니다. 각 `spanClass`에 대해 별도의 `mcentral`이 있어서 다른 `spanClass`를 가지는 할당 요청은 잠금 없이 동시에 실행될 수 있습니다. 

```go
type mheap struct {
	//...

	sweepgen  uint32 
	allspans  []*mspan 
	allArenas []arenaIdx
	central   [numSpanClasses]struct {
		mcentral mcentral
		pad      [(cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize) % cpu.CacheLinePadSize]byte
	}
	
	// ...
}
```

할당 가능한 메모리 공간이 고갈되면 `mheap`은 운영체제로부터 큰 크기의 메모리 페이지인 `arena`를 할당받아 `mspan`을 생성하고 `mcentral`을 확장합니다.

## 메모리 할당

Go는 객체를 크기별로 나누어 다른 정책으로 메모리를 할당합니다.

### Tiny

작은 문자열과 같이 16바이트보다 작은 객체들은 "Tiny allocator"를 통해 여러 요청을 단일 16바이트 메모리 블록으로 묶어서 `mcache`에 할당됩니다. 이러한 객체들은 블록을 해제하기 용이하고 낭비를 줄이기 위해 포인터를 가지고 있지 않아야 합니다. `mcache`는 사용 가능한 `mspan`이 없을 경우 `mheap`에 있는 `mcentral`에서 새로운 `mspan`을 할당 받습니다. 마찬가지로, `mcentral`도 사용 가능한 `mspan`이 없으면 OS로부터 새로운 `arena`를 할당받아 `mspan`을 구성하고 반환합니다.

### Small

16바이트보다 크고 32킬로바이트보다 작은 일반적인 객체는 Tiny와 유사하게 `mcache`에 할당됩니다. 하지만 이러한 객체들은 하나의 단일 메모리 블록으로 묶는 과정이 없으며, 객체들은 포인터를 가지고 있어 GC 과정에서 재귀적으로 스캔 될 수 있습니다. 요청된 객체의 크기와 최대한 가까운 `mspan` 종류를 찾고 `mcache`에 할당됩니다. `mcache`는 사용 가능한 `mspan`이 없을 경우 `mheap`에 있는 `mcentral`에서 새로운 `mspan`을 할당 받습니다. 마찬가지로, `mcentral`도 사용 가능한 `mspan`이 없으면 OS로부터 새로운 `arena`를 할당받아 `mspan`을 구성하고 반환합니다.

### Large

32킬로바이트보다 큰 객체들은 여러 고루틴에서 공유될 가능성이 높아 객체를 할당하고 접근하기 위한 오버헤드를 줄이기 위해 `mcache`에 할당되지 않고 `mheap`에 직접 할당됩니다. 요청된 객체의 크기와 최대한 가까운 `mspan` 종류를 찾고 `mcentral`에서 새로운 `mspan`을 할당 받습니다. 사용 가능한 `mspan`이 없으면 OS로부터 새로운 `arena`를 할당받아 `mspan`을 구성하고 반환합니다.

## Garbage Collector (GC)

GC는 쓰기 장벽을 사용하며 여러 스레드가 병렬로 마킹하고 정리합니다. JVM에서 제공하는 CMS GC와는 다르게 세대별 GC나 압축을 지원하지 않습니다.

컴파일 단계에서 수명이 정해진 객체를 최대한 스택에 할당하고 완료되거나 휴면 상태의 고루틴과 관련된 자원은 더 빨리 해제될 수 있다는 가정을 가지는 Request Oriented Collector (ROC)를 사용하여 전역 동기화를 최대한 줄이고 쉽게 확장되게 만들어졌고, TCMalloc에서 영감을 받은 고루틴 별 캐시와 크기별 할당을 사용하여 단편화를 줄였습니다.

초기 구현에서는 쓰기 장벽을 항상 사용해야 했기 때문에 비복사 세대별 GC의 구현이 시도되었습니다. 하지만 개선된 쓰기 장벽의 속도가 충분히 빠르지 않았고, 똑똑한 컴파일러 덕분에 많은 수명이 짧은 객체들이 스택에 할당되어 다른 런타임 언어들의 세대별 GC보다 효과가 적었습니다. 그 결과, 세대별 GC를 지원하지 않는 상태에서 쓰기 장벽을 최소화하는 방향으로 나아가게 되었습니다.

### GC 준비

GC가 실행될 수 있는 안전 지점에 도달하기 위해 모든 고루틴을 정지합니다. 그 후에는 마킹된 상태이지만 아직 정리되지 않은 `mspan`을 정리합니다.

### 객체 마킹

`gcphase`를 `_GCoff`에서 `_GCmark`로 변경하여 마킹 단계를 준비합니다. 이 과정에서는 모든 고루틴의 쓰기 장벽을 활성화하고 루트 마킹 작업을 준비합니다. 쓰기 장벽을 활성화하는 동안 Stop-the-World (STW)가 발생합니다. 그 후, 병렬로 고루틴의 스택을 스캔하여 관련된 객체를 회색으로 표시합니다. 객체가 스캔되는 동안 해당 고루틴은 일시 중지됩니다. 모든 고루틴의 스캔이 완료되면 찾은 회색 객체가 아직 살아 있는지 확인하고, 살아 있다면 검은색으로 표시합니다. 회색 객체를 스캔하는 동안에도 STW가 발생합니다.

그 후, 다시 STW를 발생시키고 `gcphase`를 `_GCmarktermination`로 설정하여 마킹 단계에서 사용한 워커들을 정리하고 Pacer가 다음 GC 주기를 계산합니다. Pacer는 다음 Heap의 크기가 살아 남은 Heap 크기의 2배에 도달하면 GC를 실행합니다. 이 비율은 `GOGC`를 사용하여 조정할 수 있습니다.

### 객체 정리

정리를 준비하기 위해 `gcphase`를 `_GCoff`로 설정하고 쓰기 장벽을 비활성화합니다. 이후에는 STW를 해제하고 검은색이 아닌 흰색 객체를 정리합니다. 이 과정은 고루틴을 중단시키지 않고 `mspan`을 병렬로 정리합니다.

## 참고 자료
- [Go](https://github.com/golang/go)
- [Getting to Go: The Journey of Go's Garbage Collector](https://go.dev/blog/ismmkeynote)
- [A Guide to the Go Garbage Collector](https://tip.golang.org/doc/gc-guide)
- [A visual guide to Go Memory Allocator from scratch (Golang)](https://medium.com/%2540ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed)
- [Request Oriented Collector (ROC) Algorithm](http://golang.org/s/gctoc)
- [Go/Golang Memory Management](https://syntaxsugar.tistory.com/entry/Golang-Memory-Management)
