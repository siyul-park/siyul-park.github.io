---
layout: single
title:  "인덱스를 지원하는 인메모리 도큐먼트 데이터베이스 개발기"
date:   2024-01-14 12:00:00 +0900
categories: develop
---

[uniflow](https://github.com/siyul-park/uniflow)는 Stand-Alone을 지원하여 사용 편의성을 높이고, 빠르고 독립적인 테스트를 위해 인메모리 도큐먼트 데이터베이스인 `memdb`를 내장하고 있습니다.

하지만 외부 설계가 변화하는 동안 `memdb`는 테스트를 통과할 정도만 바뀌었고, 이로 인해 코드에 오류를 숨기려는 부분이 섞여 가독성이 떨어졌습니다. 

또한 읽기 성능이 `mongodb`보다 떨어지며 인덱스를 사용할 경우가 인덱스를 사용하지 않았을 때보다 성능이 낮아 더 이상 사용하기 어려운 정도로 노후화가 진행되었습니다.

```shell
BenchmarkCollection_FindOne/with_index-4         202    6776052 ns/op   211865 B/op    6044 allocs/op
BenchmarkCollection_FindOne/without_index-4      849    1325947 ns/op    27821 B/op     839 allocs/op
```

`memdb`의 이해하기 어려운 코드를 개선하고, 전반적인 성능을 올리기 위해 리팩토링을 진행하게 되었습니다.

### 최대한 단순하게

인덱스 사용 시 성능이 저하되는 주된 이유는 인덱스 스캔 이후에도 전체 도큐먼트를 순회하는 풀 스캔 과정이었습니다. 이 문제를 해결하기 위해 정상적으로 동작하지 않는 인덱스 부분을 제거하여 코드를 보다 이해하기 쉽고 수정하기 용이하게 다듬었습니다.

또한, `collection`과 `indexView`에 분산된 데이터 관리 연산을 `section`이라 명명된 스토리지 계층으로 분리하여 응집성을 높이고, `collection`과 `indexView`의 인터페이스와 독립적으로 구성되도록 추출합니다.

```go
type Section struct {
	data maps.Map
	mu   *sync.RWMutex
}

// ...

func (s *Section) Set(doc *primitive.Map) (primitive.Value, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	id, ok := doc.Get(keyID)
	if !ok {
		return nil, errors.WithStack(ErrPKNotFound)
	}

	if _, ok := s.data.Get(id); ok {
		return nil, errors.WithStack(ErrPKDuplicated)
	}

	s.data.Put(id, doc)

	return id, nil
}

func (s *Section) Delete(doc *primitive.Map) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	id, ok := doc.Get(keyID)
	if !ok {
		return false
	}

	if _, ok := s.data.Get(id); !ok {
		return false
	}

	s.data.Remove(doc.GetOr(keyID, nil))

	return true
}

func (s *Section) Range(f func(doc *primitive.Map) bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	for _, doc := range s.data.Values() {
		if !f(doc.(*primitive.Map)) {
			break
		}
	}
}

// ...

```

```go
// collection.FindMany

c.section.Range(func(doc *primitive.Map) bool {
	if match == nil || match(doc) {
		docs = append(docs, doc)
	}
	return len(sorts) > 0 || limit < 0 || len(docs) < limit+skip
})
```

이렇게 단순화된 데이터베이스 모델은 `map`에 데이터를 저장하며, 모든 데이터를 순회하여 제공된 필터와 일치하는 도큐먼트를 찾습니다.

### 인덱싱과 역 인덱싱

인덱스를 구현하는 방법으로는 각 브랜치 노드에는 인덱스의 키 값을 저장하고, 리프 노드에는 기본 키를 저장했습니다. 각 노드는 `red-black tree`로 구현된 `map`을 사용했습니다.

또한, 인덱스를 명세하는 `constraint`를 추가하였습니다. 이로써 복합 인덱스(Compound Index), 유니크 인덱스(Unique Index), 그리고 부분 인덱스(Partial Index)를 지원할 것입니다.

```go
type Section struct {
	data        maps.Map
	indexes     []maps.Map // []map[index key]...[]primary key
	constraints []Constraint
	mu          sync.RWMutex
}
```

```go
type Constraint struct {
	Name    string
	Keys    []string
	Unique  bool
	Partial func(*primitive.Map) bool
}
```

인덱싱은 인덱스를 타고 내려가며 인덱스 키에 맞는 노드를 생성합니다. 그리고 리프 노드에 도달하면 해당 리프 노드에 기본 키를 삽입합니다. 특히, 유니크 인덱스인 경우 리프 노드에 두 개 이상의 기본 키가 들어 있다면 충돌이 발생하게 됩니다.

```go
func (s *Section) index(doc *primitive.Map) error {
	id, ok := doc.Get(keyID)
	if !ok {
		return errors.WithStack(ErrPKNotFound)
	}

	for i, constraint := range s.constraints {
		partial := constraint.Partial
		if partial != nil && !partial(doc) {
			continue
		}

		cur := s.indexes[i]

		for i, k := range constraint.Keys {
			value, _ := primitive.Pick[primitive.Value](doc, k)

			c, _ := cur.Get(value)
			child, ok := c.(maps.Map)
			if !ok {
				child = treemap.NewWith(comparator)
				cur.Put(value, child)
			}

			if i < len(constraint.Keys)-1 {
				cur = child
			} else {
				child.Put(id, nil)

				if constraint.Unique && child.Size() > 1 {
					s.unindex(doc)
					return errors.WithStack(ErrIndexConflict)
				}
			}
		}
	}

	return nil
}
```

`section`에 도큐먼트를 추가하면 인덱싱이 수행됩니다.

```go
func (s *Section) Set(doc *primitive.Map) (primitive.Value, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	id, ok := doc.Get(keyID)
	if !ok {
		return nil, errors.WithStack(ErrPKNotFound)
	}

	if _, ok := s.data.Get(id); ok {
		return nil, errors.WithStack(ErrPKDuplicated)
	}

	if err := s.index(doc); err != nil {
		return nil, err
	}
	s.data.Put(id, doc)

	return id, nil
}
```

역 인덱싱은 인덱싱과는 반대로, 도큐먼트가 생성한 키들을 인덱스에서 제거하는 작업입니다. 인덱싱과 유사하게 인덱스를 타고 리프 노드까지 탐색하며 해당 리프 노드에 있는 기본 키를 제거합니다. 그리고 경로를 거슬러 올라가면서 비어 있는 노드를 삭제합니다.

```go
func (s *Section) unindex(doc *primitive.Map) {
	id, ok := doc.Get(keyID)
	if !ok {
		return
	}

	for i, constraint := range s.constraints {
		cur := s.indexes[i]
		
		paths := []maps.Map{cur}
		keys := []primitive.Value{nil}

		for i, k := range constraint.Keys {
			value, _ := primitive.Pick[primitive.Value](doc, k)

			c, _ := cur.Get(value)
			child, ok := c.(maps.Map)
			if !ok {
				paths = nil
				keys = nil

				break
			}
			
			paths = append(paths, child)
			keys = append(keys, value)

			if i < len(constraint.Keys)-1 {
				cur = child
			} else {
				child.Remove(id)
			}
		}

		for i := len(paths) - 1; i >= 0; i-- {
			child := paths[i]

			if child.Empty() && i > 0 {
				parent := paths[i-1]
				key := keys[i]

				parent.Remove(key)
			}
		}
	}
}
```
