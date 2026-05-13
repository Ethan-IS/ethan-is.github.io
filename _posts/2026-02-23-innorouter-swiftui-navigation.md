---
title: "InnoRouter Best Practice: SwiftUI Navigation을 Route와 Coordinator로 다루기"
date: 2026-02-23 00:00:00 +0900
translation_key: innorouter-swiftui-navigation
lang: ko-KR
description: "InnoRouter를 왜 써야 하는지, SwiftUI navigation을 view-local path 조작 대신 route, store, coordinator 경계로 관리하는 best practice를 설명합니다."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, navigation, swiftui, coordinator, deep-link, architecture, route]
author: ethan
toc: true
comments: true
---

## 왜 SwiftUI navigation이 실무에서 어려운가

SwiftUI의 navigation API는 작은 화면 흐름에서는 충분합니다. 하지만 앱이 커지면 navigation은 단순한 화면 전환이 아니라 app architecture 문제가 됩니다.

- route path가 여러 view에 흩어집니다.
- deep link가 화면 코드와 섞입니다.
- modal, push, tab 전환이 서로 다른 방식으로 관리됩니다.
- feature 간 이동이 직접 import로 연결됩니다.
- navigation을 테스트하려면 실제 UI를 띄워야 합니다.

[`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter)는 navigation을 view-local side effect가 아니라 typed state와 command로 다루게 합니다. 핵심 매력은 **화면 이동을 데이터화하고, coordinator가 그 데이터를 실행하게 만드는 것**입니다.

## InnoRouter가 소유하는 경계

InnoRouter가 소유하기 좋은 것:

- route stack state
- navigation command execution
- modal presentation authority
- tab coordinator state
- deep-link matching and planning
- navigation effect adapter
- host-less navigation test

소유하지 말아야 하는 것:

- business workflow state
- network retry/session lifecycle
- DI graph construction
- feature-local alert/confirmation dialog
- authentication domain policy 자체

즉 InnoRouter는 "어떤 화면으로 어떻게 이동할 것인가"를 맡습니다. "왜 이동해야 하는가"는 feature logic이 intent로 표현하고, "무엇을 보여줄 것인가"는 UI가 맡는 편이 좋습니다.

## 설치와 import

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", from: "4.2.1")
]
```

일반 앱 코드는 umbrella product인 `InnoRouter`를 import합니다. route macro가 필요한 파일에서만 `InnoRouterMacros`를 추가합니다.

```swift
import InnoRouter
import InnoRouterMacros

@Routable
enum PeopleRoute {
    case detail(UserSummary)
}
```

macro import를 필요한 파일로 제한하면 비-macro 파일이 macro plugin resolution 비용을 함께 지지 않아도 됩니다.

## Best practice 1. View가 path를 직접 소유하지 않게 합니다

SwiftUI view가 직접 `NavigationPath`를 들고 push/pop을 처리하면 처음에는 간단하지만, feature가 커질수록 path mutation이 분산됩니다.

InnoRouter의 기본 방향은 다릅니다.

- route는 enum으로 선언합니다.
- store가 route stack을 소유합니다.
- host가 SwiftUI navigation API와 연결합니다.
- coordinator가 user intent를 navigation command로 바꿉니다.

```swift
final class PeopleFeatureCoordinator {
    let navigationStore = NavigationStore<PeopleRoute>()
    let modalStore = ModalStore<PeopleModalRoute>()

    func showDetail(user: UserSummary) {
        navigationStore.send(.go(.detail(user)))
    }
}
```

view는 `NavigationHost`나 `ModalHost` 안에서 route를 화면으로 해석합니다. 이렇게 하면 push/pop이 view local state가 아니라 feature boundary의 state가 됩니다.

## Best practice 2. Leaf feature는 자기 navigation만 압니다

feature가 다른 feature router를 직접 import하면 dependency cycle이 빠르게 생깁니다. InnoSample은 이 문제를 상위 coordinator 중재로 해결합니다.

`PeopleFeature`는 settings 화면을 직접 push하지 않습니다. 대신 reducer가 `pendingSettingsRequest`를 만들고, `PeopleFeatureCoordinator`가 이 request를 소비할 수 있게 합니다. 실제 탭 전환과 `SettingsFeature` detail push는 상위 `EntireTabCoordinator`가 처리합니다.

```swift
@MainActor
@Observable
public final class EntireTabCoordinator: TabCoordinator {
    let peopleCoordinator: PeopleFeatureCoordinator
    let postsCoordinator: PostsFeatureCoordinator
    let settingsCoordinator: SettingsFeatureCoordinator

    func syncCrossFeatureNavigationFromPeople() {
        guard let request = peopleCoordinator.consumeSettingsRequest() else { return }
        selectedTab = .settings
        settingsCoordinator.showDetail(assigneeID: request.assigneeID)
    }

    func syncCrossFeatureNavigationFromSettings() {
        guard let request = settingsCoordinator.consumePeopleRequest() else { return }
        selectedTab = .people
        peopleCoordinator.showDetail(userID: request.userID)
    }
}
```

이 구조의 핵심은 런타임 이동과 컴파일 의존을 분리하는 것입니다.

- 런타임: `People -> Settings -> People` 이동 가능
- 컴파일: `People -> EntireTab <- Settings` 구조 유지

이것이 multi-feature SwiftUI 앱에서 InnoRouter를 쓰는 가장 큰 이유 중 하나입니다.

## Best practice 3. Navigation intent와 business state를 분리합니다

navigation은 business logic의 결과일 수 있지만, navigation stack 자체가 business state는 아닙니다.

좋은 흐름은 다음과 같습니다.

1. InnoFlow reducer가 "이동하고 싶다"는 intent를 만든다.
2. feature coordinator가 intent를 소비한다.
3. InnoRouter store에 route command를 보낸다.
4. SwiftUI host가 route를 화면으로 렌더링한다.

이렇게 나누면 reducer test는 navigation framework 없이도 가능하고, router test는 business use case 없이도 가능합니다.

## Best practice 4. Modal, tab, stack을 같은 언어로 둡니다

SwiftUI 앱에서 push는 `NavigationStack`, modal은 `.sheet`, tab은 `TabView`로 흩어지기 쉽습니다. InnoRouter는 이들을 route/coordinator 중심으로 묶습니다.

- `NavigationStore`: push stack
- `ModalStore`: sheet/fullScreenCover authority
- `TabCoordinator`: selected tab, tab content, badge state
- `DeepLinkPipeline`: app boundary의 URL planning

제품이 분리되어 있지만 사고방식은 같습니다. "화면 이동은 데이터이고, host가 실행한다"는 원칙입니다.

## Best practice 5. Deep link는 app boundary에서 계획합니다

deep link를 feature view 안에서 파싱하면 앱 전체 route policy가 흩어집니다. InnoRouter의 deep-link surface는 matching과 planning을 app boundary로 끌어올립니다.

실무에서는 보통 다음 방식이 좋습니다.

- URL parsing과 route planning은 app/root coordinator에서 처리합니다.
- feature는 자신이 처리 가능한 route intent만 받습니다.
- auth/session gating은 InnoRouter가 아니라 app policy에서 결정합니다.
- pending deep link는 명시적으로 저장하고 재개합니다.

이 구조는 deep link가 늘어날수록 더 가치가 커집니다.

## 도입했을 때 얻는 장점

InnoRouter를 잘 쓰면 다음 장점이 생깁니다.

- route가 enum/type으로 남습니다.
- view가 path mutation을 직접 들고 있지 않습니다.
- navigation을 host 없이 테스트할 수 있습니다.
- sibling feature 이동에서 compile dependency cycle을 줄입니다.
- modal, tab, push, deep link를 같은 구조로 설명할 수 있습니다.
- SwiftUI navigation의 side effect를 coordinator 경계에 모을 수 있습니다.

특히 feature가 많고 탭/딥링크/모달이 섞이는 앱에서는 이 장점이 큽니다. navigation이 화면 구현의 부산물이 아니라, 앱 흐름의 명시적인 모델이 됩니다.

## 언제 쓰면 좋은가

InnoRouter는 이런 앱에 잘 맞습니다.

- SwiftUI navigation이 여러 feature에 걸쳐 있는 앱
- tab, modal, push, deep link가 함께 필요한 앱
- feature 간 직접 import를 줄이고 싶은 앱
- navigation 흐름을 테스트하고 싶은 팀
- strict concurrency와 typed route model을 선호하는 팀

반대로 화면 두세 개짜리 단순 앱에서는 SwiftUI navigation API만으로 충분할 수 있습니다.

## InnoSample에서의 결론

[`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice %})에서 InnoRouter는 `Router` 타깃에만 들어갑니다.

- `PeopleFeatureRouter`는 People navigation만 압니다.
- `PostsFeatureRouter`는 Posts navigation만 압니다.
- `SettingsFeatureRouter`는 Settings navigation만 압니다.
- `EntireTabCoordinator`가 sibling 이동을 중재합니다.
- `FeatureContainer`가 leaf coordinator들을 조립합니다.

이것이 InnoRouter의 좋은 사용법입니다. **leaf feature는 자기 route만 소유하고, cross-feature navigation은 상위 coordinator가 맡습니다.** 그러면 앱은 유연하게 이동하면서도 컴파일 의존은 단단하게 유지됩니다.
