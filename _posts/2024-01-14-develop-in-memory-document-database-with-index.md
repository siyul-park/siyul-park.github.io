---
layout: single
title:  "인덱스를 지원하는 인메모리 도큐먼트 데이터베이스 개발기"
date:   2024-01-14 12:00:00 +0900
categories: develop
---

[uniflow](https://github.com/siyul-park/uniflow)는 Stand-Alone을 지원하여 사용 편의성을 높이고, 빠르고 독립적인 테스트를 위해 인메모리 도큐먼트 데이터베이스인 `memdb`를 내장하고 있습니다.

하지만 외부 설계가 변화하는 동안 memdb는 테스트를 통과할 정도만 바뀌었고, 이로 인해 코드에 오류를 숨기려는 부분이 섞여 가독성이 떨어졌습니다. 또한 읽기 성능이 `mongodb`보다 떨어지며 인덱스를 사용할 경우 성능이 인덱스를 사용하지 않았을 때보다 낮은 더 이상 사용하기 어려운 정도로 노후화가 진행되었습니다.

```go
// collection.findMany
	fullScan := true
	scan := treemap.NewWith(comparator)
	if examples, ok := FilterToExample(filter); ok {
		if ids, err := coll.indexView.findMany(ctx, examples); err == nil {
			for _, id := range ids {
				if scanSize == scan.Size() {
					break
				} else if doc, ok := coll.data.Get(id); ok && match(doc.(*primitive.Map)) {
					scan.Put(id, doc)
				}
			}
			fullScan = false
		}
	}
	if fullScan {
		for _, key := range coll.data.Keys() {
			value, _ := coll.data.Get(key)
			if scanSize == scan.Size() {
				continue
			}
			if match(value.(*primitive.Map)) {
				scan.Put(key, value)
			}
		}
	}

	if skip >= scan.Size() {
		return nil, nil
	}

	var docs []*primitive.Map
	for _, doc := range scan.Values() {
		docs = append(docs, doc.(*primitive.Map))
	}
```

```shell
BenchmarkCollection_FindOne/with_index-4         202    6776052 ns/op   211865 B/op    6044 allocs/op
BenchmarkCollection_FindOne/without_index-4      849    1325947 ns/op    27821 B/op     839 allocs/op
```

`memdb`의 이해하기 어려운 코드를 개선하고, 전반적인 성능을 올리기 위해 재구현을 진행하게 되었습니다.

