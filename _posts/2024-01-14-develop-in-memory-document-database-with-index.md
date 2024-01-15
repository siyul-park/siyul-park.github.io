---
layout: single
title:  "인덱스를 지원하는 인메모리 도큐먼트 DB 만들기"
date:   2024-01-14 12:00:00 +0900
categories: database
---

[uniflow](https://github.com/siyul-park/uniflow)은 Stand-Alone 지원과 효과적인 테스트를 위해 인메모리 도큐먼트 데이터베이스인 `memdb`를 제공합니다. 이는 개발 및 테스트 환경에서 `mongodb`를 대체하여 빠르고 간편한 환경 구성을 지원합니다.

그러나 최근 외부 설계 변경으로 가독성이 심각하게 저해되었으며, 인덱스 사용 시 성능 문제가 발생하고 읽기 성능이 `mongodb`보다 떨어진다는 문제가 있습니다.

```shell
BenchmarkCollection_FindOne/with_index-4         202    6776052 ns/op   211865 B/op    6044 allocs/op
BenchmarkCollection_FindOne/without_index-4      849    1325947 ns/op    27821 B/op     839 allocs/op
```

이해하기 쉽고 간결하게 코드를 다듬어 가독성을 향상시키고, 성능을 개선하기 위해 리팩토링을 진행하였습니다.

### 가장 간결하게

성능 저하의 주 원인은 인덱스 스캔 후에 테이블 풀 스캔이 발생하여 전체 도큐먼트를 순회하는 과정이었습니다. 이를 해결하기 위해 정상적으로 동작하지 않는 인덱스 부분을 제거하여 코드를 더 이해하기 쉽게 수정했습니다.

그리고 데이터 관리 연산을 스토리지 계층으로 분리하여 응집성을 높이고, 외부 인터페이스와 독립적으로 구성되도록 추출했습니다.

```go
type Section struct {
	data *treemap.Map
	mu   sync.RWMutex
}

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
	// ...
}

func (s *Section) Range(f func(doc *primitive.Map) bool) {
	// ...
}
```

```go
func (c *Collection) FindMany(_ context.Context, filter *database.Filter, opts ...*database.FindOptions) ([]*primitive.Map, error) {
	// ...

	match := parseFilter(filter)

	var docs []*primitive.Map
	c.section.Range(func(doc *primitive.Map) bool {
		if match == nil || match(doc) {
			docs = append(docs, doc)
		}
		return len(sorts) > 0 || limit < 0 || len(docs) < limit+skip
	})

	if skip >= len(docs) {
		return nil, nil
	}
	if len(sorts) > 0 {
		compare := parseSorts(sorts)
		sort.Slice(docs, func(i, j int) bool {
			return compare(docs[i], docs[j])
		})
	}
	docs = docs[skip:]
	if limit >= 0 && len(docs) > limit {
		docs = docs[:limit]
	}

	return docs, nil
}
```

이런 간소화된 데이터베이스 모델은 `map`에 데이터를 저장하고, 조건에 맞는 도큐먼트를 찾기 위해 모든 데이터를 순회합니다.

### 인덱싱과 역 인덱싱

모든 도큐먼트를 순회하는 것이 아닌 인덱스를 활용하여 효율적으로 도큐먼트를 찾을 수 있습니다.

인덱스를 트리로 구현했는데, 각 브랜치 노드에는 인덱스의 키 값을, 리프 노드에는 기본 키를 저장했습니다. 각 노드는 `red-black tree`를 사용했습니다. 인덱스 범위 스캔을 지원하려면 `b+ tree`가 더 적합하지만, 적절한 라이브러리를 찾지 못해 부득이하게 `red-black tree`를 사용하게 되었습니다.

```go
type Section struct {
	data        *treemap.Map
	indexes     []*treemap.Map // []map[index key]...[]primary key
	constraints []Constraint
	mu          sync.RWMutex
}
```

그리고 인덱스의 제약 조건을 정의하여 복합 인덱스(Composite Index), 유니크 인덱스(Unique Index), 부분 인덱스(Partial Index)를 지원할 것입니다.

```go
type Constraint struct {
	Name    string
	Keys    []string
	Unique  bool
	Partial func(*primitive.Map) bool
}
```

인덱싱은 인덱스를 따라 내려가며 인덱스 키에 맞는 노드를 생성하고, 리프 노드에 도달하면 기본 키를 삽입합니다. 유니크 인덱스인 경우 리프 노드에 두 개 이상의 기본 키가 들어 있으면 충돌이 발생합니다.

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
			child, ok := c.(*treemap.Map)
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

역 인덱싱은 도큐먼트가 생성한 키들을 인덱스에서 제거하는 과정으로, 인덱싱과 비슷하게 인덱스를 타고 리프 노드까지 탐색하며 해당 리프 노드에 있는 기본 키를 제거합니다. 그리고 경로를 거슬러 올라가면서 비어 있는 노드를 삭제합니다.

```go
func (s *Section) unindex(doc *primitive.Map) {
	id, ok := doc.Get(keyID)
	if !ok {
		return
	}

	for i, constraint := range s.constraints {
		cur := s.indexes[i]

		paths := []*treemap.Map{cur}
		keys := []primitive.Value{nil}

		for i, k := range constraint.Keys {
			value, _ := primitive.Pick[primitive.Value](doc, k)

			c, _ := cur.Get(value)
			child, ok := c.(*treemap.Map)
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

### 인덱스 스캔

인덱스를 탐색하여 도큐먼트를 찾아보겠습니다. 인덱스 키가 최소값과 최대값 범위 내에 위치하는 하위 인덱스 노드들을 병합하여 새로운 인덱스를 생성합니다. 이를 통해 인덱스의 깊이를 줄여가며 후보군을 좁혀 나갈 것입니다.

범위 내의 키를 찾기 위해 인덱스 전체를 탐색했습니다. 이웃 노드 간에 연결된 `b+ tree`와 같은 자료구조를 활용하면 범위 스캔을 더 효율적으로 수행할 수 있습니다.

```go
func (s *Section) Scan(name string, min, max primitive.Value) (*Sector, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	for i, constraint := range s.constraints {
		if constraint.Name != name {
			continue
		}

		sector := &Sector{
			keys:  constraint.Keys,
			data:  s.data,
			index: s.indexes[i],
			mu:    &s.mu,
		}
		return sector.Scan(constraint.Keys[0], min, max)
	}

	return nil, false
}
```

```go
func (s *Sector) Scan(key string, min, max primitive.Value) (*Sector, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	if len(s.keys) == 0 || s.keys[0] != key {
		return nil, false
	}

	index := treemap.NewWith(comparator)

	s.index.Each(func(key, value any) {
		k := key.(primitive.Value)
		v := value.(*treemap.Map)

		if (min != nil && primitive.Compare(k, min) < 0) || (max != nil && primitive.Compare(k, max) > 0) {
			return
		}

		v.Each(func(key, value any) {
			if v, ok := value.(*treemap.Map); ok {
				if old, ok := index.Get(key); ok {
					value = deepMerge(old.(*treemap.Map), v)
				}
			}
			index.Put(key, value)
		})
	})

	return &Sector{
		data:  s.data,
		keys:  s.keys[1:],
		index: index,
		mu:    s.mu,
	}, true
}
```

최소값과 최대값이 동일한 경우, 인덱스 풀 스캔 대신 인덱스 유니크 스캔을 사용하여 더 효율적으로 결과를 찾을 수 있도록 개선할 수 있습니다. 

```go
func (s *Sector) Scan(key string, min, max primitive.Value) (*Sector, bool) {
	// ...

	if min != nil && max != nil && primitive.Compare(min, max) == 0 {
		value, ok := s.index.Get(min)
		if !ok {
			value = treemap.NewWith(comparator)
		}

		return &Sector{
			data:  s.data,
			keys:  s.keys[1:],
			index: value.(*treemap.Map),
			mu:    s.mu,
		}, true
	}

	// ...
}
```

제한된 인덱스 범위 내의 모든 도큐먼트를 순회하기 위해 리프 노드까지 이동하여 모든 노드들을 병합한 후, 해당 리프 노드에 저장된 기본 키를 활용하여 도큐먼트를 검색합니다.

```go
func (s *Sector) Range(f func(doc *primitive.Map) bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	sector := s
	for len(sector.keys) > 0 {
		sector, _ = sector.Scan(sector.keys[0], nil, nil)
	}

	for iterator := sector.index.Iterator(); iterator.Next(); {
		key := iterator.Key()
		doc, ok := s.data.Get(key)
		if ok {
			if !f(doc.(*primitive.Map)) {
				break
			}
		}
	}
}
```

### 실행 계획

검색 조건을 바탕으로 인덱스를 어떻게 탐색할지를 결정해야 합니다. 실행 계획은 특정한 인덱스 키의 범위를 가지고 있고 다음에 실행할 계획을 갖습니다.

```go
type executionPlan struct {
	key  string
	min  primitive.Value
	max  primitive.Value
	next *executionPlan
}
```

인덱스 키와 검색 조건을 기반으로 적절한 실행 계획이 만들어지게 됩니다.

`AND` 연산은 모든 자식을 탐색 범위의 겹치는 부분, 즉 교집합을 나타내도록 병합합니다. 그리고 `EQ`, `GT`, `GTE`, `LT`, `LTE` 연산자는 인덱스 키와 필터의 키가 일치하면 해당 키를 이용하여 인덱스를 탐색할 수 있습니다. 일치하지 않는 다른 키들은 모든 영역을 순회합니다.

```go
func newExecutionPlan(keys []string, filter *database.Filter) *executionPlan {
	if filter == nil {
		return nil
	}

	var plan *executionPlan

	switch filter.OP {
	case database.AND:
		for _, child := range filter.Children {
			plan = plan.intersect(newExecutionPlan(keys, child))
		}
	case database.EQ, database.GT, database.GTE, database.LT, database.LTE:
		var pre *executionPlan
		for _, key := range keys {
			var cur *executionPlan
			if key != filter.Key {
				cur = &executionPlan{
					key: key,
				}
			} else {
				value := filter.Value

				var min primitive.Value
				var max primitive.Value
				if filter.OP == database.EQ {
					min = value
					max = value
				} else if filter.OP == database.GT || filter.OP == database.GTE {
					min = value
				} else if filter.OP == database.LT || filter.OP == database.LTE {
					max = value
				}

				cur = &executionPlan{
					key: key,
					min: min,
					max: max,
				}
			}

			if pre == nil {
				plan = cur
			} else {
				pre.next = cur
			}
			pre = cur

			if cur.min != nil || cur.max != nil {
				break
			}
		}
		if pre != nil && pre.min == nil && pre.max == nil {
			plan = nil
		}
	}

	return plan
}
```

작은 범위로 합치는 대신 더 큰 범위로 병합할 때도 있습니다. 연산자가 `IN`이나 `OR`이면 모든 자식의 탐색 범위를 포함하게 합집합으로 병합합니다.

```go
func newExecutionPlan(keys []string, filter *database.Filter) *executionPlan {
	// ...

	switch filter.OP {
	case database.AND:
		// ...
	case database.OR:
		for _, child := range filter.Children {
			plan = plan.union(newExecutionPlan(keys, child))
		}
	case database.IN:
		value := filter.Value.(*primitive.Slice)
		for _, v := range value.Values() {
			plan = plan.union(newExecutionPlan(keys, database.Where(filter.Key).EQ(v)))
		}
	case database.EQ, database.GT, database.GTE, database.LT, database.LTE:
		// ...
	}

	return plan
}
```

### 도큐먼트 검색

이제 검색 조건이 주어지면 인덱스를 활용할 수 있습니다. 각 제약조건을 순회하여 실행 계획을 만들고, 그에 따라 인덱스를 스캔하여 조건에 맞는 도큐먼트를 수집합니다. 

대부분의 경우에는 인덱스 스캔을 사용하면 테이블 풀 스캔이 필요하지 않지만, 부분 인덱스를 활용하는 경우 모든 도큐먼트가 인덱싱되어 있지 않을 수 있어 추가적으로 테이블 풀 스캔을 수행해야 합니다.

```go
func (c *Collection) FindMany(_ context.Context, filter *database.Filter, opts ...*database.FindOptions) ([]*primitive.Map, error) {
	// ...

	fullScan := true

	var plan *executionPlan
	for _, constraint := range c.section.Constraints() {
		if plan = newExecutionPlan(constraint.Keys, filter); plan != nil {
			plan.key = constraint.Name
			fullScan = constraint.Partial != nil
			break
		}
	}

	scan := treemap.NewWith(comparator)
	appends := func(doc *primitive.Map) bool {
		if match == nil || match(doc) {
			scan.Put(doc.GetOr(keyID, nil), doc)
		}
		return len(sorts) > 0 || limit < 0 || scan.Size() < limit+skip
	}

	if plan != nil {
		sector, ok := c.section.Scan(plan.key, plan.min, plan.max)
		plan = plan.next

		for ok && plan != nil {
			sector, ok = sector.Scan(plan.key, plan.min, plan.max)
			plan = plan.next
		}

		if ok {
			sector.Range(appends)
		} else {
			fullScan = true
		}
	}

	if fullScan {
		c.section.Range(appends)
	}

	// ...
}
```

이렇게 만들어진 인덱스를 활용하면 인덱스를 사용하지 않은 것보다 200배 더 빠른 성능을 보여줍니다.

```go
BenchmarkCollection_FindOne/With_Index-4                  426932              3373 ns/op             416 B/op         15 allocs/op
BenchmarkCollection_FindOne/Without_Index-4                 1986            665737 ns/op           65168 B/op       5014 allocs/op
BenchmarkCollection_FindMany/With_Index-4                 377970              3690 ns/op             408 B/op         14 allocs/op
BenchmarkCollection_FindMany/Without_Index-4                2036            685697 ns/op           65068 B/op       5013 allocs/op
```

지금까지 작성된 코드의 최신 버전은 [memdb](https://github.com/siyul-park/uniflow/tree/main/pkg/database/memdb)에서 확인하실 수 있습니다.

### 그 이후에

개발된 데이터베이스는 테스트와 개발 환경에서 사용하기에 충분히 우수하지만, 몇 가지 개선할 수 있는 점이 있습니다.

현재 구현된 인덱스는 `red-black tree`를 사용하여 인덱스 풀 스캔과 인덱스 유니크 스캔만을 지원하고 있어 제약이 있습니다. 이를 극복하기 위해 `b+ tree`를 도입하여 다양한 스캔 방법을 지원하면 성능을 향상시킬 수 있습니다.

또한, 최적의 실행 계획을 도출하기 위해 첫 번째로 생성된 실행 계획이 아닌 비용을 기반으로 한 실행 계획 비교를 통해 최적의 경로를 선택할 수 있도록 개선할 수 있습니다.