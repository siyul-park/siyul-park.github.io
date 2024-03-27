---
title: "컴파일을 통한 인코딩 최적화"
date: 2024-04-05 12:00:00 +0900
categories: "system"
tags: ["system", "golang"]
---

런타임 시점에 객체를 유연하게 처리하기 위해 리플렉션은 인코딩 라이브러리에서 자주 사용됩니다. 더 빠른 성능을 얻기 위해서는 자동 코드 생성이나 전용 인터페이스를 활용하는 것이 좋지만, 추가적인 코드 관리를 필요로 하기 때문에 간편한 리플렉션을 선호하는 경우도 많습니다.

[uniflow](https://github.com/siyul-park/uniflow)에서는 여러 연산 단위인 노드 간에 데이터를 자유롭게 전달 하기 위해 JSON과 유사한 타입을 새롭게 정의하여 활용합니다. 이 타입은 여러 노드에서 동시에 사용해도 문제가 없도록 불변하게 설계되었습니다.

```go
type Value interface {
	Kind() Kind
	Compare(v Value) int
	Interface() any
}
```

이 타입은 종종 외부 라이브러리와 통신을 위해 다른 구조체나 표준 객체로 변환되어야 합니다. 이러한 변환 과정에서 유연하게 모든 타입에 대응하기 위해 리플렉션을 활용하여 타입에 대한 정보를 동적으로 처리합니다.

```go
var httpPayload *HTTPPayload
err := primitive.Unmarshal(rawPayload, &httpPayload)
```

그러나 객체를 변환하는 과정에서 활용되는 리플렉션은 많은 자원을 소모합니다. `GET /ping` 요청에 대한 응답으로 `pong`을 반환하는 서버에서는 객체 변환 작업이 전체 CPU 자원의 27%를 소모했습니다.

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
  code: pong
```

```shell
  flat  flat%   sum%        cum   cum%
 0.03s  0.22% 87.61%      3.76s 27.73%  github.com/siyul-park/uniflow/pkg/encoding.(*EncoderGroup[go.shape.interface {},go.shape.interface { Compare(github.com/siyul-park/uniflow/pkg/primitive.Value) int; Interface() interface {}; Kind() github.com/siyul-park/uniflow/pkg/primitive.Kind }]).Encode
```

성능 문제를 해결하기 위해 리플렉션을 대체하는 효율적인 방법을 찾는 중에, [go-json](https://github.com/goccy/go-json)이라는 `encoding/json`과 호환되면서도 높은 성능을 보이는 라이브러리를 발견했습니다. 이 라이브러리는 버퍼 재사용, 컴파일 과정을 통한 리플렉션 최적화, 커스텀 리플렉션 라이브러리를 활용한 이스케이프 회피 등 다양한 방법을 사용하여 성능을 향상시킵니다.

특히, 컴파일 과정을 통한 리플렉션 최적화는 적용하기 쉽고 Go의 내부 런타임에 과도하게 접근하지 않아 안정성이 있는 것으로 보였습니다. 따라서 컴파일 과정을 통한 최적화를 우선 적용해보기로 결정했습니다.

이 방법은 변환 프로세스를 컴파일하는 과정을 추가합니다. 이를 통해 한 번만 이루어져야 하는 높은 비용의 연산들을 분리하고 런타임에서 사용하는 리플렉션을 최소화하여 각 필드의 주소에 직접 접근하거나 복사하는 방식으로 변환 프로세스를 구현합니다. 이러한 과정은 각 타입별로 단 한 번만 일어나며, 생성된 변환 프로세스는 리플렉션 사용을 최소화하여 빠른 속도를 보장합니다.

```go
if typ.Elem().Kind() == reflect.Struct {
    var decoders []encoding.Encoder[*Map, unsafe.Pointer]
    for i := 0; i < typ.Elem().NumField(); i++ {
        field := typ.Elem().Field(i)
        tag := getMapTag(field)

        if !field.IsExported() || tag.ignore {
            continue
        }

        child, err := decoder.Compile(field.Type)
        if err != nil {
            return nil, err
        }

        offset := field.Offset
        alias := NewString(tag.alias)

        var dec encoding.Encoder[*Map, unsafe.Pointer]
        if tag.inline {
            dec = encoding.EncodeFunc[*Map, unsafe.Pointer](func(source *Map, target unsafe.Pointer) error {
                return child.Encode(source, unsafe.Pointer(uintptr(target)+offset))
            })
        } else {
            dec = encoding.EncodeFunc[*Map, unsafe.Pointer](func(source *Map, target unsafe.Pointer) error {
                value, ok := source.Get(alias)
                if !ok {
                    if !tag.omitempty {
                        return errors.WithMessage(encoding.ErrInvalidValue, fmt.Sprintf("key(%v) is zero value", field.Name))
                    }
                    return nil
                }
                return child.Encode(value, unsafe.Pointer(uintptr(target)+offset))
            })
        }

        decoders = append(decoders, dec)
    }

    return encoding.EncodeFunc[Value, unsafe.Pointer](func(source Value, target unsafe.Pointer) error {
        if s, ok := source.(*Map); ok {
            for _, dec := range decoders {
                if err := dec.Encode(s, target); err != nil {
                    return err
                }
            }
            return nil
        }
        return errors.WithStack(encoding.ErrUnsupportedValue)
    }), nil
}
```

성능 최적화 이후에는 최대 3.25배까지 성능이 향상되는 결과를 얻었습니다. 그러나 타입을 미리 추론하기 어려운 `map`과 같은 경우에는 컴파일 단계에서 약간의 오버헤드가 발생하여 성능이 소폭 감소했습니다.

```shell
--Prev--
BenchmarkMap_Encode/map-16         	 1892743	       629.1 ns/op
BenchmarkMap_Encode/struct-16      	 1004965	      1176 ns/op
BenchmarkMap_Decode/map-16         	 1824111	       665.8 ns/op
BenchmarkMap_Decode/struct-16      	  990968	      1044 ns/op

--Curr--
BenchmarkMap_Encode/map-16         	 1585542	       748.2 ns/op
BenchmarkMap_Encode/struct-16      	 1810263	       656.2 ns/op
BenchmarkMap_Decode/map-16         	 2069302	       553.2 ns/op
BenchmarkMap_Decode/struct-16      	 3717460	       321.8 ns/op
```

[AB(Apache HTTP server benchmarking tool)](https://httpd.apache.org/docs/)를 사용하여 ping 예제를 실험했을 때, 0.128ms가 걸렸던 요청이 0.090ms로 1.42배 빨라졌습니다.

```shell
--Prev--
This is ApacheBench, Version 2.3 <$Revision: 1903618 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 204 requests
Completed 408 requests
Completed 612 requests
Completed 816 requests
Completed 1020 requests
Completed 1224 requests
Completed 1428 requests
Completed 1632 requests
Completed 1836 requests
Completed 2040 requests
Finished 2048 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /ping
Document Length:        4 bytes

Concurrency Level:      64
Time taken for tests:   0.262 seconds
Complete requests:      2048
Failed requests:        0
Total transferred:      268288 bytes
HTML transferred:       8192 bytes
Requests per second:    7817.00 [#/sec] (mean)
Time per request:       8.187 [ms] (mean)
Time per request:       0.128 [ms] (mean, across all concurrent requests)
Transfer rate:          1000.03 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.4      1       2
Processing:     1    7   2.4      8      14
Waiting:        1    7   2.3      8      14
Total:          2    8   2.4      8      15

Percentage of the requests served within a certain time (ms)
  50%      8
  66%      9
  75%     10
  80%     10
  90%     11
  95%     12
  98%     13
  99%     14
 100%     15 (longest request)

--Curr--
This is ApacheBench, Version 2.3 <$Revision: 1903618 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 204 requests
Completed 408 requests
Completed 612 requests
Completed 816 requests
Completed 1020 requests
Completed 1224 requests
Completed 1428 requests
Completed 1632 requests
Completed 1836 requests
Completed 2040 requests
Finished 2048 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /ping
Document Length:        4 bytes

Concurrency Level:      64
Time taken for tests:   0.185 seconds
Complete requests:      2048
Failed requests:        0
Total transferred:      268288 bytes
HTML transferred:       8192 bytes
Requests per second:    11095.52 [#/sec] (mean)
Time per request:       5.768 [ms] (mean)
Time per request:       0.090 [ms] (mean, across all concurrent requests)
Transfer rate:          1419.45 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   0.8      2       6
Processing:     0    3   1.2      3       8
Waiting:        0    3   1.1      3       8
Total:          1    6   1.4      5      11

Percentage of the requests served within a certain time (ms)
  50%      5
  66%      6
  75%      6
  80%      7
  90%      8
  95%      8
  98%      9
  99%      9
 100%     11 (longest request)
```

cpu profile 상에서도 인코딩이 사용하는 자원이 5.08%로 감소한 것을 확인할 수 있었습니다.

```shell
  flat  flat%   sum%        cum   cum%
     0     0%   100%      0.06s  5.08%  github.com/siyul-park/uniflow/pkg/encoding.(*Assembler[go.shape.*uint8,go.shape.interface {}]).Encode
```

이 최적화 과정을 통해 만족할 만한 성능 향상을 이끌어 냈습니다. 더욱 성능을 높이기 위해 바이트 코드를 활용하여 익명 함수의 오버헤드를 줄이고 캐시 히트를 높이는 방식을 적용할 수 있지만 이러한 과정은 코드를 좀 더 복잡하게 만들 수 있습니다.

컴파일 과정을 추가하여 최적화를 적용하니, 인코딩 과정이 병목 현상을 유발하지 않게 되었습니다. 코드의 단순성과 가독성을 우선시하여 더 이상의 최적화를 진행하지 않고 추후 성능 개선이 요구되면 다른 방법들도 적용해 볼 것입니다.
