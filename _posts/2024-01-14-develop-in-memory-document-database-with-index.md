---
layout: single
title:  "인덱스를 지원하는 인메모리 도큐먼트 DB 만들기"
date:   2024-01-14 12:00:00 +0900
categories: database
---

[uniflow](https://github.com/siyul-park/uniflow)은 Stand-Alone 지원과 효과적인 테스트를 위해 인메모리 도큐먼트 데이터베이스인 `memdb`를 제공합니다. 이는 개발 및 테스트 환경에서 `mongodb`를 대체하여 빠르고 간편한 환경 구성을 위해 설계되었습니다.

`mongodb`를 테스트 환경에서 사용할 때는 각 테스트 간 독립성을 유지하기 위해 개별적으로 `mongodb` 인스턴스를 활용했습니다. 테스트 속도를 향상시키기 위해 인스턴스 풀과 `memory` 모드를 활용했지만, 새로운 `mongodb` 인스턴스를 생성하는 데 걸리는 시간이 길어 원하는 성능을 얻지는 못했습니다.

```shell
BenchmarkCollection_InsertOne-4    	       1	1412007646 ns/op	  896992 B/op	    4648 allocs/op
BenchmarkCollection_InsertMany-4   	       7	 195963402 ns/op	 6438740 B/op	  119611 allocs/op
BenchmarkCollection_UpdateOne-4    	     328	   3257259 ns/op	   13387 B/op	     205 allocs/op
BenchmarkCollection_UpdateMany-4   	      31	  39023772 ns/op	   17394 B/op	     249 allocs/op
BenchmarkCollection_DeleteOne-4    	     561	   3238568 ns/op	    7004 B/op	      83 allocs/op
BenchmarkCollection_DeleteMany-4   	      38	  37795620 ns/op	   10713 B/op	     129 allocs/op
BenchmarkCollection_FindOne/With_Index-4         	    2521	    426688 ns/op	   10949 B/op	     136 allocs/op
BenchmarkCollection_FindOne/Without_Index-4      	     884	   1300319 ns/op	   10998 B/op	     139 allocs/op
BenchmarkCollection_FindMany/With_Index-4        	    2980	    561747 ns/op	   10629 B/op	     131 allocs/op
BenchmarkCollection_FindMany/Without_Index-4     	     516	   2980139 ns/op	   10657 B/op	     134 allocs/op
BenchmarkServerAndRelease-4                      	       1	1439755650 ns/op	  532400 B/op	    3858 allocs/op
```

그리고 개발 환경에서 `mongodb` 인스턴스를 추가로 실행하는 과정이 번거로웠습니다. 개발 서버의 성능이 제한적이어서 `mongodb`를 실행하는 것이 부담스러웠으며, 탐색적 테스트와 데모를 위해 추가한 일회성 데이터가 다음 실행 때도 그대로 유지되는 것이 불편했습니다.

임베디드 인메모리 데이터베이스가 필요하다는 생각이 들었고, 이에 `sqlite`를 비롯한 여러 구현체를 살펴봤습니다. 몇몇 주목할 만한 프로젝트가 있었지만, 통합하기에 번거로우면서도 `mongodb`에서 이용하는 모든 기능을 지원하지 않았습니다.

그래서 인메모리 도큐먼트 데이터베이스를 직접 만들어보고 싶다는 욕심이 생겨, 개발 및 테스트 환경에서 빠르고 간편하게 사용할 수 있는 인메모리 도큐먼트 데이터베이스인 `memdb`를 개발하게 되었습니다.

`memdb`는 `mongodb`의 클라이언트를 참고하여 만들어진 독립적인 데이터베이스 인터페이스를 구현합니다. 이는 데이터베이스 벤더와 독립적으로 정의되어 있어서 비지니스 코드가 특정 데이터베이스 벤더에 의존하지 않게 도와줍니다. 이 인터페이스를 `mongodb`나 추후에 지원할 다른 데이터베이스도 동일하게 구현하게 됩니다. 

```go
type Collection interface {
	Name() string

	Indexes() IndexView

	Watch(ctx context.Context, filter *Filter) (Stream, error)

	InsertOne(ctx context.Context, doc *primitive.Map) (primitive.Value, error)
	InsertMany(ctx context.Context, docs []*primitive.Map) ([]primitive.Value, error)

	UpdateOne(ctx context.Context, filter *Filter, patch *primitive.Map, options ...*UpdateOptions) (bool, error)
	UpdateMany(ctx context.Context, filter *Filter, patch *primitive.Map, options ...*UpdateOptions) (int, error)

	DeleteOne(ctx context.Context, filter *Filter) (bool, error)
	DeleteMany(ctx context.Context, filter *Filter) (int, error)

	FindOne(ctx context.Context, filter *Filter, options ...*FindOptions) (*primitive.Map, error)
	FindMany(ctx context.Context, filter *Filter, options ...*FindOptions) ([]*primitive.Map, error)

	Drop(ctx context.Context) error
}

type IndexView interface {
	List(ctx context.Context) ([]IndexModel, error)
	Create(ctx context.Context, index IndexModel) error
	Drop(ctx context.Context, name string) error
}

type IndexModel struct {
	Name    string
	Keys    []string
	Unique  bool
	Partial *Filter
}
```

초기에는 인터페이스를 설계할 때 어떤 타입이든 받을 수 있도록 `interface{}`를 사용했지만, 이로 인해 너무 많은 타입에 대응하기 어려웠고, `reflection` 사용이 빈번하여 성능이 감소했습니다. 이에 따라 명확한 타입 정의가 필요하다고 판단하여 `BSON`과 유사한 `Boolean`, `Integer`, `Float`, `String`, `Map`, `Slice`, `Binary`를 지원하기로 결정했습니다. 이 값들은 공통 인터페이스인 `primitive.Value`를 구현합니다.

```go
type Value interface {
	Kind() Kind    
	Compare(v Value) int 
	Interface() any
}
```

문서를 저장하고 인덱싱하기 위해 처음에 고려한 방법은 해쉬 맵을 사용하는 것이었습니다. 해쉬 맵은 인덱스 유니크 스캔 이외의 다른 스캔 방식을 지원하기 어렵지만, 인덱스 키가 동일한 값을 찾는 경우가 가장 많아 다른 스캔 방식을 지원할 필요성이 크지 않았습니다. 개발 및 테스트 데이터가 적어 해시 충돌도 적을 것으로 예상되었습니다. 또한, `go`에서 네이티브로 지원하는 `map`을 그대로 활용할 수 있어 개발이 용이했습니다.

문서를 탐색하기 위해 `SQL`의 `WHERE` 절을 `AST`로 변환한 것과 유사한 `Filter`를 `Example`로 만들었습니다. `Example`는 인덱스 키와 그에 해당하는 값을 가지고 있는 `map`이며, 탐색을 원하는 문서의 부분집합이었습니다. 인덱스 유니크 스캔만을 고려했기 때문에 이런 단순한 구조로 인덱스를 탐색할 수 있을 것으로 가정했습니다. `Filter`가 `Example`로 변환된 후에는 해당 `Example`를 기반으로 어떻게 인덱스를 탐색할지 결정하게 됩니다.

```go
type Filter struct {
	OP       OP
	Key      string
	Value    primitive.Value
	Children []*Filter
}

type Example *primitive.Map

func FilterToExample(filter *Filter) Example {
	// ...
}
```

이런 설계는 아쉽게도 매우 복잡한 구조를 만들었습니다. 

`Filter`를 `Example`로 변환하고, 이를 기반으로 인덱스를 탐색하는 과정은 자연스럽게 분리되지 않았습니다. 인덱스 탐색 로직이 두 단계로 분산되어 코드가 복잡하게 되고 많은 오류가 발생했습니다. 또한, 동등성 비교만 하는 `Example`의 구조는 다른 인덱스 스캔을 지원하는데 어려움을 만들었습니다.

해쉬 맵을 사용하는 결정은 초기에는 괜찮았지만, 모든 타입을 허용하는 대신 `primitive.Value`를 사용하면서 네이티브 해쉬 맵 사용이 어려워졌습니다. `primitive.Value`에 해쉬 함수를 추가하고 하위 타입들에도 해쉬 함수를 구현해야 했습니다.

그리고 인터페이스를 그대로 구현하려는 급한 마음에 데이터 처리 로직이 `Collection`과 `IndexView`에 분산되어, 서로 큰 연관관계를 형성하면서 데이터 처리 과정이 더욱 복잡해졌습니다.

이러한 이유로 가독성이 심각하게 떨어지고, 인덱스를 사용하는 경우 비정상적으로 성능이 저하되었습니다. 읽기 성능도 `mongodb`보다 떨어지는 결과를 보여주었습니다.

```shell
BenchmarkCollection_FindOne/with_index-4         202    6776052 ns/op   211865 B/op    6044 allocs/op
BenchmarkCollection_FindOne/without_index-4      849    1325947 ns/op    27821 B/op     839 allocs/op
```

코드를 이해하기 쉽고 간결하게 다듬어 가독성을 향상시키고, 성능을 개선하기 위해 리팩토링과 설계 변경이 절실했습니다.

### 다시 처음으로

원점으로 돌려 가장 간결하고 단순한 구현체부터 시작해봅시다. 

정상적으로 동작하지 않는 인덱스 부분을 제거하고 데이터 관리 연산을 스토리지 계층으로 분리하여 응집성을 높이고, 표현 계층과 분리되어 구성되도록 추출했습니다.

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

이 간소화된 데이터베이스 모델은 기본 키와 문서를 저장하며, 모든 문서를 탐색하여 조건에 맞는 문서를 찾습니다. 심지어 기본 키로 검색하는 경우에도 동일합니다.

### 인덱싱과 역 인덱싱

이 간단하지만 다소 기본적인 데이터베이스를 개선해봅시다. 

모든 문서를 순회하는 대신에 인덱스를 활용하여 효율적으로 문서를 찾을 수 있습니다.

인덱스를 트리로 구현했는데, 각 브랜치 노드에는 인덱스 키 값을, 리프 노드에는 기본 키를 저장했습니다. 이전에 사용한 해쉬 맵과는 달리 각 노드는 `red-black tree`를 활용했습니다. `b+ tree`가 더 적합한 경우도 있었지만, 적절한 라이브러리를 찾지 못해 `red-black tree`를 선택하게 되었습니다. 트리를 사용함으로써 해쉬 함수를 제거하고 `Filter`와 일치 여부를 확인하기 위해 사용되는 비교 연산으로 대체할 수 있었습니다.

```go
type Section struct {
	data        *treemap.Map
	indexes     []*treemap.Map // []map[index key]...[]primary key
	constraints []Constraint
	mu          sync.RWMutex
}
```

그리고 인덱스의 제약 조건인 `Constraint`을 정의합니다. 이 `Constraint`는 `IndexModel`과 유사하지만 스토리지 계층을 위한 세부적인 정의를 제공합니다. 복합 인덱스(Composite Index), 유니크 인덱스(Unique Index), 부분 인덱스(Partial Index)를 지원할 것입니다.

```go
type Constraint struct {
	Name    string
	Keys    []string
	Unique  bool
	Partial func(*primitive.Map) bool
}
```

인덱싱은 트리 구조를 따라 내려가며 인덱스 키에 맞는 노드를 생성하고, 리프 노드에 도달하면 해당 리프 노드에 기본 키를 삽입합니다. 유니크 인덱스인 경우 리프 노드에 두 개 이상의 기본 키가 들어 있으면 충돌이 발생합니다.

이 과정의 시간 복잡도는 O(N * D * log(K))입니다. 여기서 N은 `Constraint`의 수이고, D은 각 `Constraint`에서의 키의 개수인 인덱스의 깊이 입니다. 그리고 K은 이 인덱스 노드에 저장되어 있는 유일한 키의 개수인 카디널리티입니다.

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

역 인덱싱은 문서가 생성한 키들을 인덱스에서 제거하는 과정으로, 인덱싱과 비슷하게 인덱스를 타고 리프 노드까지 탐색하며 해당 리프 노드에 있는 기본 키를 제거합니다. 그리고 경로를 거슬러 올라가면서 비어 있는 노드를 삭제합니다.

역 인덱싱은 인덱싱과 동일한 시간 복잡도인 O(N * D * log(K))를 가집니다.

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

구성된 인덱스를 탐색하여 문서를 찾아보겠습니다. 인덱스 키가 최소값과 최대값 범위 내에 위치하는 하위 인덱스 노드들을 병합하여 새로운 인덱스를 생성합니다. 이를 통해 인덱스의 깊이를 줄여가며 후보군을 좁혀나갈 수 있습니다.

범위 내의 키를 찾기 위해 인덱스 전체를 탐색했습니다. 이웃 노드 간에 연결된 `b+ tree`와 같은 자료구조를 활용하면 인덱스 범위 스캔(Index Range Scan)을 효율적으로 수행할 수 있습니다.

이 인덱스 풀 스캔(Index Full Scan)은 O(K)의 시간 복잡도를 가집니다. 

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

최소값과 최대값이 동일한 경우, 인덱스 풀 스캔 대신 인덱스 유니크 스캔(Index Unique Scan)을 사용하여 더 효율적으로 결과를 찾을 수 있도록 개선할 수 있습니다. 

인덱스 유니크 스캔은 O(log(K))의 시간 복잡도를 가져 인덱스 풀 스캔보다 더 빠르게 결과를 찾을 수 있습니다.

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

제한된 인덱스 범위 내의 모든 문서를 순회하기 위해 리프 노드까지 이동하여 모든 노드들을 병합한 후, 해당 리프 노드에 저장된 기본 키를 활용하여 문서를 검색합니다.

인덱스 범위에 있는 모든 문서를 순회하는 것은 O(D * log(K))의 시간 복잡도를 가집니다.

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

`Filter`을 바탕으로 인덱스를 어떻게 탐색할지를 결정해야 합니다.

`Example`과는 달리, `executionPlan`은 인덱스 탐색 방법을 명확히 규정합니다. 이를 통해 여러 곳에 퍼진 암묵적인 개념을 명확히 표현하고, 관련 로직을 응집시켜 복잡성을 줄일 수 있었습니다. 그리고 훨씬 유연한 구조를 가질수 있게 되었죠.

`executionPlan` 특정한 인덱스 키의 범위를 가지고 있고 다음에 실행할 `executionPlan`을 갖습니다.

```go
type executionPlan struct {
	key  string
	min  primitive.Value
	max  primitive.Value
	next *executionPlan
}
```

인덱스 키와 `Filter`을 기반으로 적절한 `executionPlan`을 만들 수 있습니다.

`AND` 연산자은 모든 자식을 탐색 범위의 겹치는 부분, 즉 교집합을 나타내도록 병합합니다. 그리고 `EQ`, `GT`, `GTE`, `LT`, `LTE` 연산자는 인덱스 키와 `Filter`의 키가 일치하면 해당 키를 이용하여 인덱스를 탐색할 수 있습니다. 일치하지 않는 다른 키들은 모든 영역을 순회합니다.

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

작은 범위로 합치는 대신 더 큰 범위로 병합할 때도 있습니다. 연산자가 `IN`이나 `OR`이면 모든 자식의 탐색 범위를 포함하게 합집합으로 병합합니다. 인덱스 루스 스캔(Index Loose Scan)를 지원하면 더 효율적이지만 현재는 고려하지 않겠습니다.

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

### 문서 검색

이제 `Filter`가 주어지면 인덱스를 활용할 수 있습니다. 각 `Constraint` 을 순회하여 `executionPlan`을 만들고, 그에 따라 인덱스를 스캔하여 조건에 맞는 문서를 수집합니다. 

대부분의 경우에는 인덱스 스캔을 사용하면 테이블 풀 스캔이 필요하지 않지만, 부분 인덱스를 활용하는 경우 모든 문서가 인덱싱되어 있지 않을 수 있어 추가적으로 테이블 풀 스캔을 수행해야 합니다.

`Filter`을 더 지능적으로 활용하여 부분 인덱스를 사용했을 때에도 테이블 풀 스캔을 피하도록 할 수 있지만, 현재는 최대한 간단한 구현을 유지하고 있고 반드시 필요한 기능이 아니라 이를 생략했습니다.

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

이렇게 만들어진 인덱스를 활용하면 인덱스를 사용하지 않은 것보다 200배 더 빠른 성능을 보여줍니다. 벤치마크에 사용되는 `Collection`에는 1000개의 문서가 저장되어 있습니다.

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

또한, 최적의 `executionPlan`을 도출하기 위해 첫 번째로 생성된 `executionPlan`이 아닌 가장 적은 비용을 가진 `executionPlan`를 선택할 수 있도록 개선할 수 있습니다.

## 도움

- [ewlkkf](https://github.com/ewlkkf)
