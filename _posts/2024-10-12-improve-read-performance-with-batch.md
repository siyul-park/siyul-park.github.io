---
title: "배치 처리를 통해 읽기 성능 높이기"
date: 2024-10-12 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "ddd", "system"]
---

캐시를 활용하면 리소스를 사용자에게 가까운 위치에 저장해 더 빠르게 제공할 수 있습니다. 하지만 서로 다른 리소스를 요청할 경우, 여전히 반복적인 조회 요청이 발생하게 됩니다. 인덱싱된 리소스는 빠르게 찾을 수 있지만, I/O 요청으로 인한 오버헤드는 여전히 존재하며, 인덱싱되지 않은 리소스의 경우 여러번의 풀 스캔이 필요해 더 큰 오버헤드가 발생할 수 있습니다.

이러한 오버헤드를 줄이기 위해 요청을 병합하여 배치 처리할 수 있습니다. 요청을 영속성 저장소로 바로 보내지 않고 일정 시간 동안 큐에서 대기시킨 후, 모인 요청들을 한 번에 처리하는 방식입니다. 이 방법은 I/O 횟수를 줄이고 오버헤드를 최소화할 수 있으며, 도메인 계층을 수정하지 않고 영속성 계층에서 요청 병합을 처리할 수 있어 계층 간 정보 누수를 방지할 수 있습니다.

그러나 이 방식에서는 큐에서 대기하는 동안 쓰레드가 차단될 수 있습니다. 이를 해결하기 위해 I/O 전용 쓰레드를 동적으로 할당하거나 경량 쓰레드를 사용할 수 있지만, 일정 시간 동안 대기하는 문제를 완전히 해결하지는 못합니다. 마이크로 배치 처리는 효율성을 높일 수 있으나, 사용자 응답 시간이 길어질 수 있고, 동시 요청이 적을 경우 병합되는 요청 수가 제한적일 수 있습니다. 또한, 트랜잭션 격리 수준에 따라 잘못된 리소스가 반환될 위험도 있습니다.

또 다른 방법으로는 읽기 요청을 미리 수집하는 방식이 있습니다. 이 방식은 여러 요청을 하나의 컨텍스트로 묶어 동시에 처리할 수 있는 읽기 요청들을 사전에 모읍니다. 이렇게 수집된 요청들은 하나의 통합된 읽기 요청으로 병합되어 처리되고, 그 결과는 컨텍스트에 저장되어 각 요청에 맞는 데이터를 반환합니다. 이를 통해 중복된 요청을 줄이고 성능을 최적화할 수 있으며, 동일한 컨텍스트 내에서 발생하는 다른 요청에도 즉시 결과를 제공할 수 있습니다.

이 방식은 즉시 로딩과 지연 로딩의 중간 정도로, 읽기 요청을 미리 준비하되 실행 시점은 지연시키는 특징이 있습니다. 또한, 모든 리소스를 로딩하지 않고 동일한 컨텍스트 내의 리소스만을 로딩하기 때문에 즉시 로딩에 비해 오버패칭이 적습니다.

레포지토리에서 여러 어그리게잇을 동시에 로드할 때도 이 방식을 사용할 수 있습니다. 동일한 연산을 수행할 가능성이 높은 어그리게잇을 하나의 컨텍스트로 묶어 처리하고, 필요한 조회 요청을 미리 등록해둔 후 실제 조회 시 여러 어그리게잇의 데이터를 한꺼번에 로딩하여 각 어그리게잇에 적절히 분배할 수 있습니다.

이 방식은 큐에서 대기하는 마이크로 배치 처리와 달리 순차적인 대기 없이 여러 요청을 병합할 수 있어 효율적입니다. 하지만 예측이 틀릴 경우 불필요한 읽기 요청이 발생할 수 있고, 미리 등록된 요청을 기반으로 하기 때문에 동적 요청 처리에 유연성이 떨어질 수 있습니다. 또한, 도메인 계층에 영속성 저장소 개념이 유입되어 도메인 계층의 복잡성을 높일 위험도 있습니다. 이러한 문제는 구현 방식에 따라 어느 정도 완화될 수 있지만, 직관성은 저하될 수 있습니다.

그럼에도 불구하고, 이러한 배치 처리 방식은 데이터 중심 설계를 유지하지 않으면서도 성능상의 이점을 누릴 수 있는 효과적인 방법입니다. 여러 요청을 통합하여 처리함으로써 I/O 오버헤드를 줄이고 데이터 조회 효율성을 극대화할 수 있습니다.
