---
title: "다단계 캐시를 통해 읽기 성능 높이기"
date: 2024-10-01 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "ddd", "system"]
---

**도메인 주도 설계(Domain-Driven Design, DDD)**는 도메인 개념을 명확히 표현하고 트랜잭션 일관성을 유지할 경계를 식별하여 **애그리게잇(Aggregate)**을 정의합니다. 애그리게잇은 루트 엔티티를 중심으로 긴밀하게 결합된 객체들의 집합으로, 내부 엔티티와 값 객체는 트랜잭션 일관성 하에 관리됩니다.

수정 연산의 경우 대부분 단일 리소스만 수정하는 경향이 있어 애그리게잇 방식이 매우 효과적입니다. 하지만 수정이 없는 조회 연산에서는 여러 관련 리소스를 한 번에 읽어야 하는 경우가 많습니다. 애그리게잇은 도메인 로직을 충실히 반영하지만, 영속성 저장소에서 데이터를 효율적으로 가져오는 데 어려움이 따를 수 있습니다. 특히 여러 트랜잭션 경계를 넘나드는 상황에서는 여러 애그리게잇이 동일한 리소스를 지연 로딩할 때 1+N 문제가 발생할 수 있습니다.

이 문제를 해결하는 한 가지 방법은 하위 리소스를 지연 로딩하지 않고 애그리게잇을 로드할 때 하위 리소스도 함께 로드하는 것입니다. 이는 영속성 로직이 애그리게잇에 스며들지 않도록 하여 설계를 깔끔하게 유지할 수 있지만, 불필요한 리소스까지 함께 로딩하는 **과도한 패칭(Over-fetching)**이 발생할 수 있으므로 애그리게잇을 작게 설계하는 것이 중요합니다.

또 다른 방법은 조회 연산에 최적화된 별도의 읽기 모델을 사용하는 것입니다. 이를 통해 조회 시마다 필요한 데이터를 즉시 로드하는 대신, 미리 계산된 읽기 모델을 사용할 수 있습니다. 변경이 발생할 때마다 이벤트를 구독해 읽기 모델을 동기화하거나, 조회 시 한 번 계산하여 캐시하는 방법도 가능합니다.

그러나 트랜잭션 내에서 여러 도메인 서비스가 동일한 애그리게잇을 로드하는 경우, 이 방식은 그다지 효과적이지 않을 수 있습니다. 이미 로드된 애그리게잇을 여러 도메인 서비스 간에 전달하기 어려운 상황이 발생할 수 있으며, 특히 애그리게잇이 다른 애그리게잇을 참조하고 있을 경우, 도메인 서비스가 이를 로드하는 과정에서 동일한 애그리게잇을 여러 번 로드하는 상황이 불가피해집니다.

트랜잭션 내에서 이미 로딩된 리소스를 캐시하면 동일한 리소스를 한 번만 로딩할 수 있습니다. 이를 통해 도메인 계층은 리소스 로딩에 대한 부담을 덜고, 도메인 로직에 더욱 집중할 수 있습니다. 이 **다단계 캐시**는 영속성 저장소의 최신 상태를 반영하는 루트 캐시와 트랜잭션 중 발생한 변경 사항을 저장하는 자식 캐시로 구성됩니다.

루트 캐시는 로컬 인메모리 방식으로 구현될 수 있으며, `Redis`나 `Memcached`와 같은 원격 인메모리 저장소를 사용할 수도 있습니다. 경우에 따라 영속성 저장소 자체를 그대로 사용해 캐시를 생략하기도 합니다. 반면, 자식 캐시는 분산 트랜잭션을 사용하지 않는 경우 각 머신의 로컬 인메모리로 구현하는 것이 적합합니다.

다단계 캐싱 방식은 단일 계층 캐시보다 자주 접근되는 리소스를 사용자에게 더 가까운 위치에 배치하여 성능을 향상시키며, 각 트랜잭션이 캐시에서 독립적으로 처리될 수 있도록 격리성을 보장합니다.

리소스를 조회할 때는 먼저 트랜잭션에 할당된 자식 캐시를 확인하고, 자식 캐시에 데이터가 없으면 루트 캐시를 조회합니다. 루트 캐시에도 리소스가 없을 경우, 영속성 저장소에서 리소스를 로드한 후 자식 캐시에 저장합니다. 트랜잭션이 커밋되면 자식 캐시의 내용이 루트 캐시에 반영됩니다.

캐시 조회는 도메인 계층에서 명시적으로 처리할 필요가 없습니다. 영속성 계층에서 조회 요청을 분석하여 해당 리소스를 캐시에서 자동으로 조회합니다. 리소스가 캐시에서 발견되면 원격 저장소에서 추가 검색을 하지 않아도 되므로, 조회 요청에서 불필요한 조건을 제거해 비효율적인 검색을 방지할 수 있습니다.

캐시는 기본키 기반의 키-값 저장소와 유사한 구조일 수 있지만, 추가적인 인덱스를 활용해 더 다양한 조회 요청을 처리할 수 있습니다. 또한, 수명이 짧고 크기가 작은 자식 캐시는 풀스캔을 통해 더 많은 요청을 효율적으로 처리할 수 있습니다.

다단계 캐시는 중복된 요청을 줄여 읽기 성능을 향상시키지만, 캐시와 영속성 저장소 간의 데이터 불일치를 충분히 고려해야 합니다. 트랜잭션이 커밋된 후 서비스가 다운되어 루트 캐시가 갱신되지 않거나, 영속성 저장소의 트랜잭션 격리 수준이 로컬 캐시의 격리 수준과 달라 서로 다른 격리 수준이 공존하는 상황이 발생할 수 있습니다.

문제를 최소화하려면 적절한 캐시 수명을 설정하고 필요에 따라 루트 캐시를 생략하여 동기화 빈도를 높일 수 있지만, 복잡성을 완전히 해소하기는 어렵습니다.

위험을 충분히 이해하고 감수할 수 있다면, 다단계 캐시를 통해 읽기 성능을 향상시킬 수 있으며, 도메인 계층이 도메인 로직에 더욱 집중할 수 있도록 도움을 줄 수 있습니다.