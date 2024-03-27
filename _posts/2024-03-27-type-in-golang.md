---
title: "Go에서 타입 정보는 어디에 저장될까?"
date: 2024-03-27 00:00:00 +0900
categories: "system"
tags: ["system", "memory", "golang"]
---

리플렉션은 런타임 시점에 객체의 타입 정보에 접근하여 객체를 유연하게 처리할 수 있도록 해줍니다. 이를 위해서는 런타임에서 객체의 메타 정보에 접근할 수 있어야 합니다.

`reflect` 패키지의 `TypeOf` 함수는 객체의 참조를 `unsafe.Pointer`로 변환한 후, 객체의 헤더를 나타내는 `*emptyInterface`로 형변환합니다. 이 `emptyInterface`는 원시 타입을 나타내는 `abi.Type`와 객체의 시작 주소를 나타내는 `unsafe.Pointer`를 가지고 있습니다. 

`unsafe.Pointer`를 `*emptyInterface`로 변환할 때 두 타입이 동일한 메모리 레이아웃을 사용해야 하므로, `unsafe.Pointer`는 객체의 주소와 함께 타입의 주소도 가지고 있어야 합니다.

```go
// emptyInterface is the header for an interface{} value.
type emptyInterface struct {
	typ  *abi.Type
	word unsafe.Pointer
}

// TypeOf returns the reflection [Type] that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i any) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	// Noescape so this doesn't make i to escape. See the comment
	// at Value.typ for why this is safe.
	return toType((*abi.Type)(noescape(unsafe.Pointer(eface.typ))))
}
```

타입은 `abi.Type` 구조체를 통해 표현됩니다. 이 구조체에는 객체의 크기, 포인터를 포함한 바이트 수, 해시 값, 추가 타입 정보 플래그 등이 저장됩니다.

```go
// Type is the runtime representation of a Go type.
//
// Be careful about accessing this type at build time, as the version
// of this type in the compiler/linker may not have the same layout
// as the version in the target binary, due to pointer width
// differences and any experiments. Use cmd/compile/internal/rttype
// or the functions in compiletype.go to access this type instead.
// (TODO: this admonition applies to every type in this package.
// Put it in some shared location?)
type Type struct {
	Size_       uintptr
	PtrBytes    uintptr // number of (prefix) bytes in the type that can contain pointers
	Hash        uint32  // hash of type; avoids computation in hash tables
	TFlag       TFlag   // extra type information flags
	Align_      uint8   // alignment of variable with this type
	FieldAlign_ uint8   // alignment of struct field with this type
	Kind_       uint8   // enumeration for C
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	Equal func(unsafe.Pointer, unsafe.Pointer) bool
	// GCData stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, GCData is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	GCData    *byte
	Str       NameOff // string form
	PtrToThis TypeOff // type for pointer to this type, may be zero
}
```

이러한 메타 정보는 객체의 크기에 따라 다르게 저장됩니다. 작은 객체들은 `mheap`의 비트맵에, 중간 크기의 객체들은 객체의 값 앞에, 큰 객체들은 타입별로 독립적인 `mspan`을 사용하므로 각각의 `mspan`에 저장됩니다.

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// ...

	// Determine if it's a 'small' object that goes into a size-classed span.
	//
	// Note: This comparison looks a little strange, but it exists to smooth out
	// the crossover between the largest size class and large objects that have
	// their own spans. The small window of object sizes between maxSmallSize-mallocHeaderSize
	// and maxSmallSize will be considered large, even though they might fit in
	// a size class. In practice this is completely fine, since the largest small
	// size class has a single object in it already, precisely to make the transition
	// to large objects smooth.
	if size <= maxSmallSize-mallocHeaderSize {
		if noscan && size < maxTinySize {
			// ...

			// Allocate a new maxTinySize block.
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if !raceenabled && (size < c.tinyoffset || c.tiny == 0) {
				// Note: disabled when race detector is on, see comment near end of this function.
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			hasHeader := !noscan && !heapBitsInSpan(size)
			if goexperiment.AllocHeaders && hasHeader {
				size += mallocHeaderSize
			}
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(x, size)
			}
			if goexperiment.AllocHeaders && hasHeader {
				header = (**_type)(x)
				x = add(x, mallocHeaderSize)
				size -= mallocHeaderSize
			}
		}
	} else {
		shouldhelpgc = true
		// For large allocations, keep track of zeroed state so that
		// bulk zeroing can be happen later in a preemptible context.
		span = c.allocLarge(size, noscan)
		span.freeindex = 1
		span.allocCount = 1
		size = span.elemsize
		x = unsafe.Pointer(span.base())
		if needzero && span.needzero != 0 {
			if noscan {
				delayedZeroing = true
			} else {
				memclrNoHeapPointers(x, size)
			}
		}
		if goexperiment.AllocHeaders && !noscan {
			header = &span.largeType
		}
	}
	if !noscan {
		if goexperiment.AllocHeaders {
			c.scanAlloc += heapSetType(uintptr(x), dataSize, typ, header, span)
		} else {
			var scanSize uintptr
			heapBitsSetType(uintptr(x), size, dataSize, typ)
			if dataSize > typ.Size_ {
				// Array allocation. If there are any
				// pointers, GC has to scan to the last
				// element.
				if typ.Pointers() {
					scanSize = dataSize - typ.Size_ + typ.PtrBytes
				}
			} else {
				scanSize = typ.PtrBytes
			}
			c.scanAlloc += scanSize
		}
	}

	// ...

	return x
}
```

```go
// heapSetType records that the new allocation [x, x+size)
// holds in [x, x+dataSize) one or more values of type typ.
// (The number of values is given by dataSize / typ.Size.)
// If dataSize < size, the fragment [x+dataSize, x+size) is
// recorded as non-pointer data.
// It is known that the type has pointers somewhere;
// malloc does not call heapSetType when there are no pointers.
//
// There can be read-write races between heapSetType and things
// that read the heap metadata like scanobject. However, since
// heapSetType is only used for objects that have not yet been
// made reachable, readers will ignore bits being modified by this
// function. This does mean this function cannot transiently modify
// shared memory that belongs to neighboring objects. Also, on weakly-ordered
// machines, callers must execute a store/store (publication) barrier
// between calling this function and making the object reachable.
func heapSetType(x, dataSize uintptr, typ *_type, header **_type, span *mspan) (scanSize uintptr) {
	const doubleCheck = false

	gctyp := typ
	if header == nil {
		if doubleCheck && (!heapBitsInSpan(dataSize) || !heapBitsInSpan(span.elemsize)) {
			throw("tried to write heap bits, but no heap bits in span")
		}
		// Handle the case where we have no malloc header.
		scanSize = span.writeHeapBitsSmall(x, dataSize, typ)
	} else {
		// ...

		// Write out the header.
		*header = gctyp
		scanSize = span.elemsize
	}
    
    // ...
	return
}
```

객체의 참조를 `unsafe.Pointer`로 변환할 때 런타임은 객체의 값의 주소와 함께 객체의 타입 정보도 찾아 변환합니다. 이를 통해 런타임에서 객체의 타입 정보에 접근할 수 있게 됩니다.

```go
// Pointer represents a pointer to an arbitrary type. There are four special operations
// available for type Pointer that are not available for other types:
//   - A pointer value of any type can be converted to a Pointer.
//   - A Pointer can be converted to a pointer value of any type.
//   - A uintptr can be converted to a Pointer.
//   - A Pointer can be converted to a uintptr.
// ...
type Pointer *ArbitraryType
```

## 참고 자료
- [Go](https://github.com/golang/go)
