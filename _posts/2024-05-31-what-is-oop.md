---
title: "그래서 객체지향이란 무엇인가요?"
date: 2024-05-31 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "oop"]
---

1967년 5월, 크리스틴 니갈(Kristen Nygaard)과 올 요한 다히(Ole-Johan Dahl)는 오슬로에서 열린 IFIP 시뮬레이션 언어 실무 회의에서 클래스와 서브클래스 선언에 관한 논문을 발표했습니다. 이 논문은 객체지향 기능이 추가된 Simula 67의 첫 번째 공식 정의가 되었습니다.

1970년대에는 미국 제록스(Xerox)사의 팔로 알토 연구센터(PARC)에서 앨런 케이(Alan Kay)와 그의 팀이 순수 객체지향 언어인 Smalltalk를 개발했습니다. Smalltalk는 동적 타이핑과 인터프리터 방식을 특징으로 하며, 언어 수준에서의 객체지향 지원과 편리한 그래픽 개발 환경으로 주목받았습니다. 다양한 버전을 거치며 발전한 Smalltalk는 Simula 67의 아이디어에 영향을 받았지만, 클래스가 동적으로 생성되고 수정될 수 있게 확장되었습니다.

1979년에 개발된 Lisp는 다중 상속과 믹스인을 도입했습니다. 1980년대 중반에는 브래드 콕스(Brad Cox)가 Smalltalk에 영향을 받아 Objective-C를 개발했고, 1985년 비야네 스트로스트룹(Bjarne Stroustrup)은 C 언어를 바탕으로 객체지향 기능을 추가하여 C++를 만들었습니다.

결국, 1990년대 초반과 중반에 객체지향 프로그래밍은 지배적인 패러다임으로 발전했습니다.

## 그래서 객체지향이란 무엇인가요?

함수형 프로그래밍을 비롯한 여러 패러다임이 탄생했지만, 객체지향 프로그래밍은 여전히 가장 보편적인 패러다임으로 자리 잡고 있습니다. 그러나 이 패러다임은 명확하게 정리된 경우가 드물며, 주로 경험을 바탕으로 직관적으로 접근되거나 SOLID와 같은 파생된 원칙과 패턴들로 규정됩니다.

초창기의 프로그램은 매우 특화된 한 가지 작업만 처리했습니다. 하지만 컴퓨팅 비용이 점점 저렴해지면서 컴퓨터는 더욱 다양한 역할을 수행하게 되었고, 이에 따라 프로그램도 점차 복잡해졌습니다. 너무나 복잡해진 프로그램은 인간의 인지 능력에 부하를 주었고, 이는 수많은 오류와 생산성 저하로 이어졌습니다. 이러한 문제를 해결하기 위해 다익스트라(Dijkstra)는 그의 논문 "[GOTO 문의 해로움](https://web.archive.org/web/20070703050443/http://www.acm.org/classics/oct95/)"에서 구조적 프로그래밍에 대한 논의를 시작했습니다.

구조적 프로그래밍은 프로그램을 간단하고 계층적인 제어 구조로 분리했습니다. 이를 위해 `GOTO` 문을 제거하여 코드의 흐름과 실행 흐름을 동일하게 만들었고, 프로그램을 작은 단위의 구조로 분할할 수 있게 되었습니다. 입증 가능한 순차, 선택, 반복의 조합으로 시작된 프로시저는 서로 합쳐져 큰 기능을 가진 프로그램을 구성하게 되었습니다. 이러한 방식은 복잡한 프로시저를 만들 때 한 단계 낮은 하위 프로시저의 기능에만 집중할 수 있게 하여, 개발자가 한 번에 인지해야 하는 개념을 줄여주었습니다.

전통적인 구조적 프로그래밍이 추상화를 제공했지만, 하위 프로시저와 연관된 모든 기능은 상위 프로시저가 제어해야 했습니다. 더 추상적인 상위 프로시저는 그보다 작고 구체적인 프로시저의 실제적 구현에 의존하고 있었으며, 완전한 분리가 이루어지지 않았습니다. 하위 프로시저가 가지는 너무나 구체적인 제어 플래그들은 추상화를 넘어서 상위 프로시저에까지 흘러오게 되었고, 이는 프로시저를 재사용하기 어렵고 복잡하게 만들었습니다.

그래서 객체지향 프로그래밍은 프로그램을 상태와 연산을 가지는 객체들로 분리했습니다. 객체의 상태에 따라 연산을 동적으로 제어될 수 있게 되면서, 호출자는 실제 구현을 알지 않아도 객체의 상태에 따라 연산을 간접적으로 호출할 수 있게 되었습니다. 이로 인해 의존성이 역전되어, 실제 구현에 의존하지 않고 고수준의 인터페이스에 의존하게 되어 각 모듈이 더 독립적이고 확장하기 쉽게 만들 수 있었습니다.

이러한 다형성을 통해 세부 구현은 객체 내부로 캡슐화되어 개념의 추상화가 이루어졌습니다. 객체들은 서로 동일한 수준에서 협력할 수 있게 되었고, 더 구체적인 연산에 의존하지 않게 되어 프로그램의 복잡도를 유지하며 기능을 확장할 수 있게 되었습니다.

## 객체 그리고 추상화

절차지향적 프로그래밍에서는 프로그램의 기본 단위인 프로시저가 단순히 특정 절차를 수행하는 역할만 할 수 있었습니다. 그러나 객체지향 프로그래밍에서는 프로그램이 상태와 행동을 가지는 여러 객체들로 구성됩니다. 이로 인해 객체들이 다양한 역할을 맡을 수 있게 되었고, 현실 세계의 개념이 자연스럽게 프로그램으로 전이되었습니다.

객체가 나타내는 개념이 실제로 존재하더라도, 실제 객체와 프로그램에서 표현되는 객체는 큰 차이가 있습니다. 예를 들어, 프로그램에서 "사용자" 객체는 실제 사람을 나타내지만, 그 사람의 모든 특징을 포함하지는 않습니다. 대신 이름, 이메일, 권한 등 특정 속성만을 포함합니다. 마찬가지로, "은행 계좌" 객체는 실제 계좌가 아니며, 잔액과 거래 내역 등 일부 속성만을 표현합니다.

프로그램에서는 현실 세계의 개념 일부를 차용해 단순화하고, 도메인 영역에 맞추어 변형합니다. 프로그램에서 나타나는 객체는 현실 세계 개념의 은유에 불과하지만, 이미 알고 있는 개념을 단순하게 추상화하여 직관적으로 이해할 수 있게 도와줍니다.

## 여러 모습의 객체지향

이렇게 다양한 객체들이 프로그램 속에 살아가게 되면서, 객체들을 분류하고 일반화하여 복잡도를 낮추려는 시도들이 일어나게 되었습니다.

영어권 사고방식의 근본이 되었던 플라톤(Πλάτων)은 모든 사물의 원인이자 본질인 이데아가 존재한다고 생각했습니다. 이데아는 우리가 사는 현상 세계 밖에 위치하며 현상 세계의 물체는 이데아의 한 상에 해당합니다. 이러한 이데아는 현상 세계의 사물들이 가지는 보편성을 기반으로 인지할 수 있습니다. 플라톤의 제자 아리스토텔레스(Ἀριστοτέληò)는 이런 이데아를 식별하기 위해 사물의 속성을 분석하여 공통적인 본질을 찾고 같은 속성을 공유하는 사물은 같은 범주로 분류하였습니다.

이러한 방식이 객체지향 프로그래밍에 녹아들어, 공통의 속성과 동작들로 객체들을 일반화하여 본질적인 클래스로 정의하게 되었습니다. 이러한 클래스는 인스턴스화를 통해 실행 환경에서 보편적으로 접근 가능한 객체가 됩니다. 클래스들 사이에서도 비슷한 분류가 일어납니다. 더 추상적인 상위 클래스는 이상적인 본질을 나타내며, 하위 클래스는 이 본질을 확장하고 구체화합니다. 하위 클래스는 상위 클래스를 상속하여 공통적인 특성을 물려받으면서도 고유한 특성과 행동을 추가하여 구체적이고 독립적인 존재로 나타납니다.

한편, 공유 속성의 관점에서 분류하기 어려운 개념이 존재한다는 주장도 있습니다. 루트비히 요제프 요한 비트겐슈타인(Ludwig Josef Johann Wittgenstein)은 대상을 완전히 규정하는 언어는 존재하지 않으며, 표현은 삶의 흐름 속에서만 의미를 가진다는 용도의미론을 제시했습니다. 용도의미론에서 의미는 그 대상이 상황과 맥락 속에서 어떻게 사용되는지에 따라 정의되므로, 진정한 본래의 의미는 존재하지 않는다고 했습니다.

그리고 비트겐슈타인은 대상을 속성이 아닌 유사성을 통해 분류해야 한다고 주장했습니다. 가족들이 완전히 같은 속성을 가지지는 않지만 서로 유사한 특징들을 가지고 있듯이, 대상을 다른 대상과 유사성을 기반으로 구분할 수 있다고 했습니다. 본질적이고 명확한 범주 없이 직관적으로 파악할 만한 유사성을 기반으로 대상의 관계를 나타내는 것은 전통적으로 범주로 묶어두던 울타리를 해체합니다.

비트겐슈타인의 이론들은 1970년대에 엘리노어 로쉬(Eleanor Rosch)에 의해 프로토타입 이론으로 정리되었습니다. 이 이론에서는 분류할 범주의 각 구성원이 다른 구성원들과 공유하는 공통 속성의 개수를 확보하며, 가장 점수가 높은 전형적인 객체를 도출합니다. 가장 전형적인 원형을 확장해 나가며 객체들은 한계 없이 유기적으로 퍼져나갈 수 있습니다.

이러한 프로토타입 이론에 영향을 받아 프로토타입 기반 객체지향 언어들이 탄생하게 되었습니다. 프로토타입 기반 객체지향 언어는 클래스 기반 객체지향 언어와는 달리 객체를 클래스로 추상화 하지는 않습니다. 메소드와 변수는 객체에 직접 추가되고 기존의 객체를 복사하여 새로운 객체를 생성합니다. 그렇게 만들어진 새로운 객체는 원본이 된 객체에 기존 연산을 위임하게 됩니다. 결과적으로 객체는 해당 객체가 속한 문맥에 따라 평가됩니다.

상속을 지원하는 전통적인 클래스와 프로토타입을 전부 지원하지 않는 언어도 등장하고 있습니다. Go나 Rust는 이 두 가지 개념을 지원하지 않지만, 인터페이스를 통한 다형성과 객체 합성을 통한 코드 재사용을 제공합니다. 하위 클래스가 부모의 모든 특성을 공유하길 원하지 않을 수 있지만, 상속을 사용하면 필요한 것보다 많은 기능이 공유될 수 있습니다. 대신 이들 언어는 상속 대신 객체 합성을 통해 코드 공유를 제공합니다. 이러한 언어는 종종 객체지향 언어가 아니라고도 말해지지만, 상태와 연산을 객체로 추상화하고 객체들의 협력으로 전체 프로그램을 구성할 수 있기 때문에 객체지향을 지원하는 언어라고 말해도 큰 무리는 아닙니다.

## 참고 문서

- [자바스크립트는 왜 프로토타입을 선택했을까](https://medium.com/%2540limsungmook/%25EC%259E%2590%25EB%25B0%2594%25EC%258A%25A4%25ED%2581%25AC%25EB%25A6%25BD%25ED%258A%25B8%25EB%258A%2594-%25EC%2599%259C-%25ED%2594%2584%25EB%25A1%259C%25ED%2586%25A0%25ED%2583%2580%25EC%259E%2585%25EC%259D%2584-%25EC%2584%25A0%25ED%2583%259D%25ED%2596%2588%25EC%259D%2584%25EA%25B9%258C-997f985adb42)
- [객체지향 언어의 특성 - The Rust Programming Language](https://rinthel.github.io/rust-lang-book-ko/ch17-01-what-is-oo.html)
- [객체지향의 사실과 오해](https://product.kyobobook.co.kr/detail/S000001628109)
- [Object-oriented programming - Wikipedia](https://en.m.wikipedia.org/wiki/Object-oriented_programming)