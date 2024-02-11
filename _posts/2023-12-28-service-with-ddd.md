---
title: "DDD와 함께하는 서비스 가이드"
date: 2023-12-27 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "ddd", "spring", "java"]
---

Spring Boot를 사용하면서 `Controller`, `Service`, `Repository`, `Entity`가 각 도메인마다 하나씩 존재하는 정형화된 패턴을 쉽게 볼 수 있습니다. 하지만 도메인의 복잡도가 올라가다 보면 그 도메인이 가진 모든 연산을 수행하는 거대한 `Service`와 그 모든 연산에 필요한 데이터들의 집합이 된 `Entity`가 되는 경우가 많습니다.

```kotlin
@Controller
class AccountController {
    // ...
}

@Service
class AccountService {
    // 매우 방대한 크기
}

@Repository
class AccountRepository {
    // ...
}

@Entity
class Account {
    // 속성만 가짐
}
```

이러한 접근은 초기에는 편리하지만, 도메인이 복잡해지면서 클래스의 크기가 늘어나고 복잡도가 증가합니다. 도메인 로직을 단순히 나열하기만 하고 그 본질을 파고들지 못하는 문제가 발생합니다.

그렇다면 어떻게 서비스를 구성해야 할까요? 이에 대한 힌트는 `@Service` 어노테이션의 정의에서 찾을 수 있습니다.

> 이 어노테이션은 도메인 주도 설계(DDD)에서 정의한 "서비스"를 나타내며, 이는 모델 내에서 독립적으로 존재하면서도 상태를 캡슐화하지 않은 인터페이스로써의 연산을 의미합니다.  
> 뿐만 아니라, 이 어노테이션은 종종 "비즈니스 서비스 퍼사드" 또는 유사한 역할을 하는 클래스일 수 있다는 표시로 사용됩니다.  
> 이 어노테이션은 일종의 표준 스테레오타입이지만, 각 팀은 그 의미를 자체적으로 조정하고 필요에 맞게 활용할 수 있습니다.  
> [Service.java](https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/stereotype/Service.java)


즉 이 정의에 따르면 서비스는 도메인 주도 설계의 개념에 기반한 `Service` 와 비즈니스 서비스 파사드의 역할을 할 수 있습니다.

### 도메인 주도 설계와 서비스

도메인 연산 중 `Entity`나 `Value Object`에서 찾을 수 없는 연산이 있습니다. 이러한 연산은 개발자와 도메인 전문가를 비롯한 모든 팀 구성원이 함께 사용하는 통합 언어인 유비쿼터스 언어에서 주로 활동이나 행위로 표현됩니다. 이를 `Entity`나 `Value Object`에 맡기면 모델의 정의가 왜곡되거나 무의미한 객체가 추가될 수 있습니다.

`Service`는 `Entity`나 `Value Object`가 명사로 이름을 부여하는 것과 달리 동사로 나타나는 활동으로 이름을 지을 수 있습니다. 이러한 `Service`는 도메인 모델의 일부로 추상적이고 명확한 정의를 갖고 있으며, 규정된 책임을 부여받습니다.

잘 구성된 `Service`의 연산은 `Entity`나 `Value Object`의 일부로써 구성되지 않는 독립적인 도메인 개념과 관련이 있습니다. 그리고 인터페이스는 도메인 모델의 외적 측면에서 정의되어야 하고 상태를 가지지 않아야 합니다.

그러나 `Entity`나 `Value Object`내의 연산으로 충분히 표현이 될 수 있지만, 연산을 다듬는 것을 빨리 포기하여 절차적 프로그래밍에 빠지는 경우가 많습니다. 특히 동일한 객체에 많은 행위가 정의되면 높은 결합도와 복잡성을 가져올 수 있습니다.

유비쿼터스 언어에 `Entity`나 `Value Object`로 표현되지 않는 명확한 행위가 있는 경우에만 해당 연산을 처리하는 역할을 갖는 `Service`를 구성해야 합니다. 이 행위자는 단일한 책임을 가지며, 책임을 자연스럽고 명확하게 표현할 수 있는 이름을 가져야 합니다. `Service`나 `Manager`와 같은 일반적인 이름을 피하고 유비쿼터스 언어에 맞는 이름을 선택하는 것이 좋습니다.

이러한 행위는 여러 도메인 모델 간의 상호 작용에서 자주 발견되며 대표적으로 `헥사고날 아키텍처`의 `Use Case`가 있습니다. 상호 작용이 여러 서브 도메인이나 바운디드 컨텍스트에 걸쳐 있다면 응용 계층에서 나타날 수 있으며, 동일한 서브 도메인이나 바운디드 컨텍스트에 속한다면 도메인 계층에 나타날 수 있습니다. 응용 계층에서 표현될 경우 `Controller`에서 행위가 표현될 수도 있지만, `Controller`도 과도한 책임을 가지지 않도록 주의해야 합니다.

하나의 도메인 모델만이 연산의 정의에 나타나는 경우 해당 도메인 모델의 연산으로 추가하는 것이 적절합니다. 분명하게 행위가 유비쿼터스 언어상에서 도메인 모델과 독립적으로 표현되지 않는 경우 우선은 해당 객체에 연산을 추가하고, 유비쿼터스 언어와 구현의 어색함을 고려하여 나중에 언어를 수정하고 `Service`로 추출할 수 있습니다.

```kotlin
@Service
class Authenticator(private val accountRepository: AccountRepository) {
    fun authenticate(username: String, password: String): Account {
        val account = accountRepository.findOneOrFail(where(Account::username).`is`(username))
        if (!account.isPasswordCorrect(password)) {
            throw AuthenticationException(username)
        }
        return account
    }
}

@Entity
class Account(/* ... */) {
    fun isPasswordCorrect(password: String): Boolean {
        // ... 비밀번호 일치 여부 확인 로직
    }

    // ...
}
```

### 비즈니스 서비스 파사드

비즈니스 서비스 파사드는 애플리케이션의 클라이언트와 내부 시스템 간의 인터페이스 역할을 하는 서비스를 나타냅니다. 이는 각 서비스 호출을 캡슐화하고 클라이언트가 내부 시스템의 복잡성을 알 필요 없이 단일한 퍼사드를 통해 서비스에 접근할 수 있게 합니다. 

파사드는 시스템 간 경계에서 사용되며, 다른 시스템의 복잡한 인터페이스를 개발하고 있는 하위 시스템에 친숙한 형태의 대안 인터페이스로 바꾸어 주는 역할을 합니다. 이런 다른 시스템은 제어가 어려운 외부 시스템이거나, 레거시 시스템일 가능성이 높습니다. 종종 비즈니스 서비스 파사드는 인프라스트럭처 계층에서 발견됩니다. 

```kotlin
@Service
class Authenticator(
    private val identityCenter: IdentityCenter,
    private val accountRepository: AccountRepository
) {
    fun authenticate(username: String, password: String): Account {
        val token = identityCenter.getToken(username, password)
        val externalAccount = identityCenter.getAccount(token)

        return accountRepository.findOneOrFail(where(Account::externalId).`is`(iamAccount.id))
    }
}
```

Spring Boot에서는 `Controller`, `Service`, `Repository`, `Entity`가 각 도메인에 맞게 하나씩 존재하는 정형화된 패턴을 손쉽게 작성할 수 있습니다. 그러나 도메인의 복잡성이 증가하면서 거대한 `Service`와 `Entity`가 생성되는 문제가 발생할 수 있습니다.

이러한 고민을 해결하기 위해 `Service`는 도메인 주도 설계의 원칙을 따르며, 유비쿼터스 언어를 기반으로 도메인의 핵심 행위를 명확하게 표현해야 합니다. 더불어, `비즈니스 서비스 파사드`를 활용하여 애플리케이션 복잡성을 감추고 클라이언트에게 편리한 인터페이스를 제공하여 시스템 간 상호 작용을 향상시킵니다.

이렇게 코드를 구성함으로써 개발자는 코드의 가독성과 유지보수성을 향상시키면서 동시에 도메인 전문가와의 협업을 원활하게 유지할 수 있습니다.

## 참고 자료

- [도메인 주도 설계, 에릭 에반스](https://product.kyobobook.co.kr/detail/S000001514402)

## 도움

- [ewlkkf](https://github.com/ewlkkf)
- [Seungwoo-Yu](https://github.com/Seungwoo-Yu)
- [5d-jh](https://github.com/5d-jh)
