---
title: "한계를 넘어, 더 큰 메모리 공간으로"
date: 2024-02-11 12:00:00 +0900
categories: "system"
tags: ["system", "memory", "java", "javascript"]
---

64비트 프로세스로의 전환은 속도, 안정성, 그리고 보안 측면에서 상당한 이점을 제공합니다. 64비트 프로세스는 더 많은 레지스터를 활용하여 데이터를 효율적으로 처리할 수 있으며, 메모리 공간도 4GB에서 최대 18EB로 확장되어 더 많은 데이터를 처리할 수 있습니다.

그러나 64비트 시스템에서는 포인터의 크기가 4바이트에서 8바이트로 두 배가 되어 포인터 자체가 차지하는 메모리와 캐시 공간이 많이 늘어났습니다. 가상 머신에서 Heap에 많은 객체들이 생성되어 포인터의 크기는 메모리 사용에 상당한 영향을 미칩니다. 많은 가상 머신에서는 이러한 문제를 해결하기 위해 포인터 압축을 사용하고 있습니다.

### HotSpot JVM에서의 포인터 압축

HotSpot JVM에서는 객체의 헤더 정보를 OOP(Ordinary Object Pointers)로 나타냅니다. 이 OOP는 Biased Locking, Identity Hash Code, GC 등에 사용되는 `Mark Word`와 클래스 메타데이터에 대한 포인터를 나타내는 `Klass`를 가지고 있습니다. `Klass`는 Java 7 이전에는 Permanent Generation을 가리켰지만, Java 8부터는 Metaspace를 가리킵니다.

```c++
class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  volatile markWord _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;

  // ...
};
```

`Mark Word`는 `uintptr_t`를 가지고 있고 32비트 아키텍처에서는 4바이트, 64비트 아키텍처에서는 8바이트 크기로 나타납니다. 데이터 구조는 객체의 상태에 따라 달라집니다. 

```c++
class markWord {
 private:
  uintptr_t _value;

  // ...
};
```

**32비트 (Normal)**
```
+----------------------+-------+---------------+--------+
| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2 |
+----------------------+-------+---------------+--------+
```

**64비트 (Normal)**
```
+-----------+----------------------+----------+-------+---------------+--------+
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |
+-----------+----------------------+----------+-------+---------------+--------+
```

- identity_hashcode: 객체의 해시값으로, 객체가 생성될 때 System.identityHashCode(obj)가 호출되어 이 값을 설정합니다.
- age: GC에서 객체가 얼마나 오래 살아있는지를 나타냅니다.
- biased_lock: 동일한 스레드에서만 자원을 경쟁할 때 OS lock보다 가볍게 활용 가능한 biased locking이 사용 가능한지를 정의합니다. JDK 15에서는 [Deprecated](https://openjdk.org/jeps/374) 되었습니다.
- lock: 잠금 정보를 저장합니다.

Instance OOP는 객체 인스턴스를 나타내는 OOP의 한 종류입니다. Instance OOP는 `Mark Word`와 `Klass`를 포함하고 있는 헤더, 객체 정렬을 하기 위한 32비트 패딩, 그리고 인스턴스 필드에 대한 참조로 구성됩니다.

```c++
class instanceOopDesc : public oopDesc {
 public:
  // aligned header size.
  static int header_size() { return sizeof(instanceOopDesc)/HeapWordSize; }

  // If compressed, the offset of the fields of the instance may not be aligned.
  static int base_offset_in_bytes() {
    return (UseCompressedClassPointers) ?
            klass_gap_offset_in_bytes() :
            sizeof(instanceOopDesc);

  }
};
```

64비트 아키텍처는 객체 참조로 32비트 아키텍처보다 두 배의 공간을 사용하므로 일반적으로 더 많은 메모리를 소비하고 GC(Garbage Collection)가 더 자주 발생합니다. JVM은 객체 포인터를 압축하여 메모리를 절약하고 32비트 참조를 사용하여 4GB 이상의 Heap 공간을 활용할 수 있도록 지원합니다.

객체의 참조는 32비트 패딩 이후에 위치합니다. Heap의 시작 지점이 32비트로 정렬되어 있으면 객체를 참조하는 포인터의 마지막 3비트가 항상 0이므로 Heap에 저장할 필요가 없습니다. 포인터가 주소를 참조하는 대신 오프셋을 나타내도록 바꾸어 64비트 참조를 사용하지 않고도 최대 32GB(2^32+3 = 2^35 = 32GB)의 공간을 활용할 수 있습니다. 이러한 최적화 과정은 Zero-Based Compressed OOPs로 알려져 있으며, JVM은 객체를 찾을 때 포인터를 3비트 왼쪽으로 시프트시켜 최적화를 수행합니다. 반면, Heap에서 포인터를 로드할 때는 포인터를 3비트 오른쪽으로 시프트하여 이전에 추가된 0을 삭제합니다. 이렇게 조정된 객체의 포인터는 Heap의 시작점으로부터 32비트로 나누어진 오프셋를 표현합니다.

그러나 메모리 단편화와 같은 문제로 Heap의 시작점이 32비트로 정렬되지 않으면 쉬프트 연산만으로 원래의 주소를 찾을 수 없습니다. 이에 따라 Heap의 시작 지점의 마지막 3비트를 더하는 과정이 필요해지며 성능이 감소합니다.

객체의 참조를 압축하는 것과 유사하게 클래스에 대한 메타데이터를 나타내는 `Klass`도 압축할 수 있습니다. Compressed Class Pointer로 불리는 이 과정을 통해 64비트 아키텍처에서 객체의 해더를 64비트인 `Mark Word`와 32비트인 `Klass`로 구성하여 96비트로 압축할 수 있습니다.

ZGC 이전에 나온 GC에서는 Compressed OOPs를 지원했지만, ZGC에서는 64-bit colored pointers를 사용하기 때문에 Compressed OOPs를 사용할 수 없습니다. 또한, Compressed Class Pointer는 Compressed OOPs와 관련이 있어서 이전에는 지원되지 않았지만, Java 15부터는 의존성이 분리되어 이를 지원하기 시작했습니다.

### V8 JavaScript 엔진에서의 포인터 압축

V8 JavaScript 엔진도 포인터 압축을 지원하지만 HotSpot JVM과는 다른 방식을 사용합니다.

V8에서는 JavaScript 값이 객체, 배열, 숫자 또는 문자열인지 관계없이 객체로 취급되며 Heap에 할당됩니다.

JavaScript는 인덱스를 증가시키는 반복문과 같이 많은 정수 연산을 수행합니다. 정수가 증가할 때마다 새로운 숫자 객체를 할당하지 않기 위해 포인터 태깅 기술을 사용하여 Heap 포인터에 추가 데이터를 저장합니다. 이 포인터 태깅은 HotSpot JVM에서 사용되는 객체 헤더와 유사합니다. 태그는 Heap에 있는 객체에 대한 강한/약한 참조를 나타내거나 작은 정수를 표현할 수 있습니다. 정수 값을 직접 태그에 저장하여 추가 메모리를 사용하지 않고도 적재할 수 있습니다.

객체는 `WORD`에 맞게 정렬된 주소에 할당되어 2~3개의 최하위 비트를 태깅에 사용할 수 있습니다. 32비트 아키텍처에서는 최하위 비트를 사용하여 작은 정수인 Smi(Small integers)를 Heap 객체 포인터와 구별합니다. Heap 포인터의 경우 두 번째 최하위 비트를 사용하여 강한 참조와 약한 참조를 구별합니다.

```
                        |----- 32 bits -----|
Pointer:                |_____address_____w1|
Smi:                    |___int31_value____0|
```

Smi 값은 부호 비트를 포함하여 31비트 페이로드로 전달할 수 있습니다. 포인터의 경우 Heap 객체 주소로 사용할 수 있는 30비트가 있습니다. `WORD` 정렬로 인해 할당 단위는 4바이트이므로 주소 지정 가능한 공간은 4GB(2^30+2)입니다.

64비트 아키텍처에서는 Smi 값에 32비트를 사용할 수 있습니다.

```
            |----- 32 bits -----|----- 32 bits -----|
Pointer:    |________________address______________w1|
Smi:        |____int32_value____|00...............00|
```

포인터 압축을 위해 이 두 종류의 태그 값을 64비트 아키텍처의 32비트에 맞추어야 합니다.

첫 번째 방법은 Heap을 4GB 영역에 정렬하여 할당하면 베이스의 부호 없는 32비트 오프셋으로 포인터를 고유하게 식별할 수 있습니다.

```
+------------------+-----------------+-------+-----------------+---------------+
| V8 Instance Data | Allocated Chunk |       | Allocated Chunk |               |
+------------------+-----------------+-------+-----------------+---------------+
+------------------------------------ 4GB -------------------------------------+
Base (4GB 정렬됨)
```

Heap은 4GB 영역에 정렬되어 있기 때문에 상위 32비트는 모든 포인터에 대해 동일합니다.

```
            |----- 32 bits -----|----- 32 bits -----|
Pointer:    |________base_______|______offset_____w1|
```

또한, Smi를 31비트로 제한하고 하위 32비트에 배치하여 Smi를 압축 가능하게 만들 수 있습니다. 기본적으로 32비트 아키텍처의 Smi와 유사하게 만듭니다.

```
         |----- 32 bits -----|----- 32 bits -----|
Smi:     |sssssssssssssssssss|____int31_value___0|
```

여기서 s는 Smi 페이로드의 부호 값입니다. 부호 확장이 있는 경우 1비트 산술 시프트만으로 Smi를 압축 및 해제할 수 있습니다. 이제 하위 비트들만 메모리에 저장하여 태그 값을 저장하는 데 필요한 메모리를 절반으로 줄일 수 있습니다.

압축은 값을 절반으로 자르면 됩니다.

```c++
uint64_t uncompressed_tagged;
uint32_t compressed_tagged = uint32_t(uncompressed_tagged);
```

그러나 압축 해제 코드는 조금 더 복잡합니다. Smi의 부호 확장과 포인터의 확장을 구별해야 하며, Base를 추가할지 여부도 구분해야 합니다.

```c++
uint32_t compressed_tagged;

uint64_t uncompressed_tagged;
if (compressed_tagged & 1) {
  // pointer case
  uncompressed_tagged = base + uint64_t(compressed_tagged);
} else {
  // Smi case
  uncompressed_tagged = int64_t(compressed_tagged);
}
```

4GB의 시작 부분에 Base를 두는 대신 중간에 배치하여 압축 된 값을 Base의 부호 있는 32비트 오프셋으로 처리할 수도 있습니다. 전체 Heap은 더 이상 4GB로 정렬되지 않지만 Base는 정렬됩니다.

```
+-----------------+--------------------+------------------+-----------------+--+
| Allocated Chunk |                    | V8 Instance Data | Allocated Chunk |  |
+-----------------+--------------------+------------------+-----------------+--+
+----------------- 2GB ----------------+----------------- 2GB -----------------+
                               Base (4GB 정렬됨)
```

이 새로운 레이아웃에서는 압축 코드가 동일하게 유지되지만 압축 해제 코드는 개선되었습니다. 부호 확장은 이제 Smi 및 포인터 모두에 공통적이며 유일한 분기는 포인터에 Base를 추가할지 여부입니다.

```c++
int32_t compressed_tagged;

// Common code for both pointer and Smi cases
int64_t uncompressed_tagged = int64_t(compressed_tagged);
if (uncompressed_tagged & 1) {
  // pointer case
  uncompressed_tagged += base;
}
```

Smi의 상위 32비트를 무시하고 사용하지 않도록 정의하면 분기를 완전히 제거할 수 있습니다. 이러한 접근 방식을 "Smi-corrupting"라고 부릅니다. 이 방식을 통해 첫 번째 Heap 레이아웃으로 돌아갈 수 있습니다.

```
         |----- 32 bits -----|----- 32 bits -----|
Pointer: |________base_______|______offset_____w1|
Smi:     |......garbage......|____int31_value___0|
```

```c++
// New decompression implementation
int64_t uncompressed_tagged = base + int64_t(compressed_tagged);
```

V8은 포인터 압축 방식을 활용하여 Heap의 크기를 최대 43%까지 줄이고 더 적은 CPU 자원을 사용하며 GC 시간을 단축시켰습니다. 

그러나 HotSpot JVM과 달리 Heap의 크기가 여전히 4GB로 제한되지만, 해더에 더 적은 메모리를 사용합니다. Chrome의 V8은 이미 64비트 아키텍처에서 Heap 크기에 대해 최대 4GB 제한이 있기 때문에 이러한 방식을 사용했습니다. Node.js와 같이 V8을 사용하는 다른 런타임에서는 4GB를 넘어가면 포인터 압축을 사용할 수 없습니다.

### 32비트의 크기와 64비트 성능, x32 ABI

리눅스 커널에서는 이러한 32비트 포인터를 직접 지원합니다. x32 ABI는 Linux 커널에서 사용되는 ABI(Application Binary Interface) 중 하나로, Intel 및 AMD 64비트 하드웨어에서 32비트 정수, long, 그리고 포인터를 제공합니다. 이를 ILP32(Integers, Longs, and Pointers are 32 bits)라고 합니다.

x86-64 아키텍처의 많은 CPU 레지스터, 우수한 부동 소수점 성능, 빠른 위치 독립적 코드, 공유 라이브러리 사용, 레지스터를 통한 함수 매개변수 전달, 그리고 빠른 syscall 명령어를 여전히 활용하면서 64비트 포인터의 오버헤드를 줄일 수 있습니다. 메모리 공간은 4GB의 가상 주소 공간으로 제한되지만, 포인터를 작게 만들어 캐시에 더 많은 코드와 데이터를 추가하여 실행 속도를 높일 수 있습니다. 

테스트 결과에 따르면, x32 ABI는 x86-64에 비해 181.mcf SPEC CPU 2000 벤치마크에서 40% 빠르며, SPEC CPU 정수 벤치마크에서도 5~8% 더 빠릅니다. 그러나 부동 소수점 벤치마크에서는 x86-64에 비해 속도 이점이 없는 경우도 있습니다.

[HotSpot](https://bugs.openjdk.org/browse/JDK-8046561)과 [V8](https://groups.google.com/g/v8-users/c/c-_URSZqTq8?pli=1)에서는 x32 지원이 제안되었지만, 너무 많은 변경이 요구되어 현재까지는 반영되지 않았습니다.

## 참고 자료

- [Java 객체는 어떻게 이루어져 있을까](https://velog.io/@dev_dong07/Java-객체는-어떻게-이루어져-있을까)
- [Compressed OOPs in the JVM](https://www.baeldung.com/jvm-compressed-oops)
- [Java Objects Inside Out](https://shipilev.net/jvm/objects-inside-out)
- [ObjectHeader32.txt](https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c)
- [OpenJDK](https://github.com/openjdk/jdk)
- [Java Biased Lock](https://taes-k.github.io/2021/08/29/java-biased-lock/)
- [Pointer Compression](https://v8.dev/blog/pointer-compression)
- [x32 ABI](https://en.wikipedia.org/wiki/x32_ABI)
