---
title: "InnoDI Best Practice: Swift Macro DI를 앱 구조로 고정하는 법"
date: 2026-02-23 00:00:00 +0900
translation_key: innodi-swift-macro-di
lang: ko-KR
description: "InnoDI를 왜 써야 하는지, Swift macro 기반 DI를 composition root와 feature boundary에 어떻게 배치해야 하는지, InnoSample의 실제 구조로 설명합니다."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, dependency-injection, macro, di, clean-architecture, swiftui, tuist]
author: ethan
toc: true
comments: true
---

## 왜 DI가 실무에서 어려운가

iOS 앱에서 의존성 주입은 처음에는 단순합니다. `APIClient`, `Repository`, `UseCase`를 생성자에 넣으면 됩니다. 문제는 앱이 커졌을 때 시작됩니다.

- 화면마다 임시 factory가 생깁니다.
- 전역 singleton이 늘어납니다.
- preview/test override가 실제 앱 graph와 달라집니다.
- feature가 다른 feature의 구현을 직접 알기 시작합니다.
- "이 객체는 누가 만들고 얼마나 오래 살아야 하는가"가 코드 리뷰에서 보이지 않습니다.

[`InnoDI`](https://github.com/InnoSquadCorp/InnoDI)는 이 문제를 런타임 container로 숨기지 않습니다. 대신 Swift macro로 DI graph를 선언하게 만들고, compile-time/build-time validation으로 구조 drift를 더 빨리 발견하게 합니다.

핵심 매력은 명확합니다. **의존성 wiring이 코드로 보이고, graph가 검증 가능하며, 생성 책임이 앱 구조 안에 남습니다.**

## InnoDI가 소유하는 경계

InnoDI가 잘하는 일은 객체를 "어디서든 꺼내 쓰게" 만드는 것이 아닙니다. 좋은 사용 방식은 반대입니다. InnoDI는 생성 경계에서만 강하게 쓰고, feature 내부 로직에는 최대한 드러내지 않는 편이 좋습니다.

InnoDI가 소유해야 하는 것:

- 앱의 composition root
- layer container와 feature container wiring
- shared/input scope
- parent-child container ownership
- graph validation과 DAG inspection
- SwiftUI root boundary 연결

InnoDI가 소유하지 말아야 하는 것:

- 화면 상태 전이
- navigation stack
- network retry/session lifecycle
- 비즈니스 규칙
- per-action runtime dependency override

즉 InnoDI는 앱의 "생성 시점 구조"를 다룹니다. 런타임 상태는 [`InnoFlow`]({% post_url 2026-02-23-innoflow-unidirectional-architecture %}), 화면 이동은 [`InnoRouter`]({% post_url 2026-02-23-innorouter-swiftui-navigation %}), 네트워크 실행 정책은 [`InnoNetwork`]({% post_url 2026-02-23-innonetwork-type-safe-networking %}) 같은 더 맞는 경계에 맡기는 편이 좋습니다.

## 설치와 기본 사용

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", from: "4.3.0")
]
```

SwiftUI root helper가 필요하면 `InnoDISwiftUI`도 함께 추가합니다.

```swift
.target(
    name: "YourApp",
    dependencies: [
        "InnoDI",
        "InnoDISwiftUI"
    ],
    plugins: [
        .plugin(name: "InnoDIDAGValidationPlugin", package: "InnoDI")
    ]
)
```

가장 작은 형태는 아래처럼 읽으면 됩니다.

```swift
import Foundation
import InnoDI

struct APIClient {
    let baseURL: URL
}

@DIContainer
struct AppContainer {
    @Provide(.input)
    var baseURL: URL

    @Provide(.shared, factory: { (baseURL: URL) in
        APIClient(baseURL: baseURL)
    }, concrete: true)
    var apiClient: APIClient
}
```

여기서 중요한 것은 `@Provide`가 "등록"이 아니라 "선언"이라는 점입니다. 컨테이너가 어떤 입력을 받고 어떤 shared dependency를 제공하는지 타입으로 드러납니다.

## Best practice 1. Composition root에서 시작합니다

앱의 최상위 컨테이너는 인프라와 큰 composition boundary만 연결해야 합니다. [`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)의 `AppContainer`는 좋은 예입니다.

```swift
@MainActor
@DIContainer(root: true, mainActor: true)
struct AppContainer {
    @Provide(.input)
    var baseURL: URL

    @Provide(.shared, factory: { (baseURL: URL) in
        LayerContainer(baseURL: baseURL)
    }, concrete: true)
    var layerContainer: LayerContainer

    @Provide(.shared, factory: { (layerContainer: LayerContainer) in
        layerContainer.featureUseCases
    })
    var featureUseCases: any FeatureUseCaseContaining

    @SubContainer(
        scope: .shared,
        bindings: [(child: \FeatureContainer.useCases, parent: \AppContainer.featureUseCases)],
        featureRoot: FeatureRootScene.self
    )
    var featureContainer: FeatureContainer
}
```

이 구조에서 `AppContainer`는 `RemoteContainer`, `DataContainer`, `DomainContainer`의 세부 구현을 직접 알지 않습니다. `LayerContainer`와 `FeatureContainer`라는 큰 경계만 연결합니다.

이것이 InnoDI의 첫 번째 best practice입니다. **상위 컨테이너는 모든 것을 만들지 말고, 다음 composition boundary만 알게 하십시오.**

## Best practice 2. 큰 graph 하나보다 작은 container 여러 개가 낫습니다

DI container 하나에 앱 전체 객체를 몰아넣으면 결국 이름만 다른 service locator가 됩니다. InnoDI는 계층마다 작은 container를 두었을 때 더 강합니다.

InnoSample의 `LayerContainer`는 `Remote -> Data -> Domain`을 연결하고, 밖으로는 feature가 필요한 use case surface만 제공합니다.

```swift
@DIContainer
public struct LayerContainer {
    @Provide(.input)
    public var baseURL: URL

    @Provide(.shared, factory: { (baseURL: URL) in
        RemoteContainer(baseURL: baseURL)
    }, concrete: true)
    var remoteContainer: RemoteContainer

    @SubContainer(
        scope: .shared,
        bindings: [(child: \DomainContainer.dataContainer, parent: \LayerContainer.repositories)]
    )
    var domainContainer: DomainContainer

    public var featureUseCases: any FeatureUseCaseContaining {
        domainContainer
    }
}
```

이 방식의 장점은 분명합니다.

- `App`은 remote/data/domain 내부 조립을 모릅니다.
- `Features`는 repository 구현을 모릅니다.
- `Domain`은 remote model과 network client를 모릅니다.
- graph는 계층별로 작게 검증됩니다.

DI는 객체를 많이 만드는 도구가 아니라 **의존 방향을 보존하는 도구**가 됩니다.

## Best practice 3. `@SubContainer`로 feature root를 연결합니다

InnoDI 4.x에서 특히 매력적인 부분은 parent-child container 관계를 코드로 선언할 수 있다는 점입니다.

```swift
@SubContainer(
    scope: .shared,
    bindings: [
        (child: \EntireTabContainer.peopleCoordinator, parent: \FeatureContainer.peopleCoordinator),
        (child: \EntireTabContainer.postsCoordinator, parent: \FeatureContainer.postsCoordinator),
        (child: \EntireTabContainer.settingsCoordinator, parent: \FeatureContainer.settingsCoordinator),
    ]
)
var entireTabContainer: EntireTabContainer
```

feature root를 이렇게 연결하면 상위는 child가 필요한 입력만 넘기고, child container는 자기 소유의 coordinator나 view root를 조립합니다.

SwiftUI 앱에서는 `featureRoot:`를 통해 root scene까지 연결할 수 있습니다. 그러면 앱 시작점에서 "container를 만들고 root view에 넘기는 반복 코드"가 줄어듭니다.

## Best practice 4. `.input`과 `.shared`를 아키텍처 언어로 씁니다

DI scope는 단순 성능 옵션이 아닙니다. scope는 객체의 수명과 ownership을 표현합니다.

- `.input`: 외부에서 주입되는 값입니다. `baseURL`, environment, feature input처럼 container가 직접 만들면 안 되는 값에 씁니다.
- `.shared`: container boundary 안에서 공유되는 인프라입니다. network client, repository, coordinator처럼 identity나 수명이 중요한 객체에 씁니다.
- computed property: stateless use case처럼 매번 만들어도 의미가 같은 값에 씁니다.

InnoSample은 repository는 shared로 두고, use case는 `DomainContainer`에서 concrete computed value로 제공합니다. 이 선택은 추상화를 줄이면서도 feature가 repository를 직접 알지 않게 합니다.

## 도입했을 때 얻는 장점

InnoDI를 제대로 쓰면 다음 이점이 생깁니다.

- 생성 책임이 앱 구조 안에 남습니다.
- feature가 다른 feature의 구현을 직접 만들지 않습니다.
- DI graph가 코드 리뷰에서 보입니다.
- macro/build validation으로 wiring mistake를 빨리 잡습니다.
- SwiftUI root와 test/previews의 entry point가 정리됩니다.
- 새 feature를 추가할 때 "어느 container에 붙여야 하는가"가 명확해집니다.

특히 팀 단위 개발에서는 마지막 장점이 큽니다. DI가 암묵적이면 새 feature가 가장 가까운 파일에서 아무 객체나 만들기 쉽습니다. InnoDI는 그 유혹을 줄이고, 앱의 생성 경계를 계속 같은 모양으로 유지하게 만듭니다.

## 언제 쓰지 않는 편이 나은가

모든 앱에 macro DI가 필요한 것은 아닙니다.

- endpoint 두세 개짜리 작은 prototype
- 런타임 plugin registration이 핵심인 앱
- global singleton을 의도적으로 유지하는 legacy 앱
- DI graph 검증보다 빠른 실험이 더 중요한 단계

이런 경우에는 가벼운 factory나 runtime dependency tool이 더 나을 수 있습니다.

반대로 앱이 여러 feature, layer, platform target으로 나뉘기 시작했다면 InnoDI는 좋은 선택입니다. 객체 생성 자체보다 **생성 경계를 유지하는 비용**이 커지는 시점부터 가치가 커집니다.

## InnoSample에서의 결론

[`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice %})은 InnoDI를 앱 전역에 흩뿌리지 않습니다.

- `AppContainer`는 앱 root composition만 소유합니다.
- `LayerContainer`는 `Remote/Data/Domain` 조립만 소유합니다.
- `FeatureContainer`는 leaf feature coordinator 조립만 소유합니다.
- feature logic은 생성된 dependency를 사용할 뿐, DI framework를 직접 알 필요가 없습니다.

이것이 InnoDI를 가장 설득력 있게 쓰는 방식입니다. **DI를 편의 기능이 아니라 아키텍처 경계의 표기법으로 쓰는 것.** 그때 InnoDI는 단순한 객체 생성 도구가 아니라, 앱 구조를 오래 유지하는 장치가 됩니다.
