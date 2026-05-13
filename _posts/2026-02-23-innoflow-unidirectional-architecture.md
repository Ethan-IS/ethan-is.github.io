---
title: "InnoFlow Best Practice: SwiftUI Feature Logic을 단방향으로 관리하기"
date: 2026-02-23 00:00:00 +0900
translation_key: innoflow-unidirectional-architecture
lang: ko-KR
description: "InnoFlow를 왜 써야 하는지, SwiftUI feature logic을 reducer/state/action/effect 경계로 어떻게 나누는 것이 좋은지 InnoSample 기준으로 설명합니다."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, state-management, unidirectional, reducer, swiftui, architecture, testing]
author: ethan
toc: true
comments: true
---

## 왜 feature state가 실무에서 어려운가

SwiftUI는 작은 화면을 빠르게 만들기 좋습니다. 하지만 앱이 커질수록 feature logic은 view 안으로 쉽게 새어 들어갑니다.

- `onAppear`마다 loading 조건이 달라집니다.
- error, retry, refresh, empty state가 view에 흩어집니다.
- async task 취소 정책이 화면마다 다릅니다.
- navigation intent와 business state가 섞입니다.
- 테스트는 view snapshot 중심으로만 남고 logic test가 약해집니다.

[`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow)는 이 문제를 reducer/state/action/effect 구조로 정리합니다. 핵심은 "SwiftUI를 대체하는 것"이 아닙니다. **SwiftUI view에서 business/domain state transition을 분리하는 것**입니다.

## InnoFlow가 소유하는 경계

InnoFlow는 feature의 비즈니스 상태 전이를 소유합니다.

소유해야 하는 것:

- feature state
- user/action event
- async effect
- loading/error/loaded phase
- one-shot intent를 만들기 전까지의 feature-local 상태
- reducer composition

소유하지 말아야 하는 것:

- concrete navigation stack
- URLSession이나 network retry
- DI graph construction
- SwiftUI layout
- app lifecycle, scene, window

이 경계가 중요합니다. InnoFlow를 navigation과 networking까지 다루는 거대한 app framework처럼 쓰면 장점이 흐려집니다. InnoFlow는 작게 둘수록 선명합니다. feature `Logic` 타깃 안에서 상태 전이를 책임지고, 나머지는 바깥 경계로 내보내는 것이 좋습니다.

## 설치와 기본 authoring surface

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "4.0.0")
]
```

제품은 역할별로 나누어 가져갑니다.

```swift
.target(
    name: "YourFeatureLogic",
    dependencies: ["InnoFlow"]
)

.target(
    name: "YourFeatureUI",
    dependencies: ["InnoFlowSwiftUI"]
)

.testTarget(
    name: "YourFeatureTests",
    dependencies: ["InnoFlowTesting"]
)
```

InnoFlow 4.x의 권장 authoring surface는 `@InnoFlow`와 `var body: some Reducer<State, Action>`입니다.

```swift
import InnoFlow

@InnoFlow
struct CounterFeature {
    struct State: Equatable, Sendable, DefaultInitializable {
        var count = 0
    }

    enum Action: Equatable, Sendable {
        case increment
        case decrement
    }

    var body: some Reducer<State, Action> {
        Reduce { state, action in
            switch action {
            case .increment:
                state.count += 1
                return .none
            case .decrement:
                state.count -= 1
                return .none
            }
        }
    }
}
```

## Best practice 1. `Logic` 타깃 안에만 둡니다

InnoSample은 leaf feature를 `Interface / Logic / UI / Router / Testing / Tests`로 나눕니다. 이때 InnoFlow는 `Logic` 타깃에만 들어갑니다.

이 배치가 주는 이점은 큽니다.

- reducer가 SwiftUI layout을 모릅니다.
- reducer가 InnoRouter route stack을 모릅니다.
- reducer가 network client를 직접 만들지 않습니다.
- UI는 state를 표시하고 action을 보낼 뿐입니다.
- Router는 reducer가 만든 intent를 소비할 뿐입니다.

즉 InnoFlow는 "feature의 생각"을 맡고, SwiftUI는 "그 생각을 보여주는 화면"을 맡습니다.

## Best practice 2. dependency는 reducer 입력으로 제한합니다

InnoSample의 `PeopleFeatureReducer`는 use case를 dependency bundle로 받습니다.

```swift
@InnoFlow(phaseManaged: true)
struct PeopleFeatureReducer {
    struct Dependencies: Sendable {
        let loadPeople: @Sendable () async throws -> [UserSummary]
    }

    struct State: Equatable, Sendable, DefaultInitializable {
        enum Phase: Hashable, Sendable {
            case idle
            case loading
            case loaded
            case failed
        }

        var phase: Phase = .idle
        var isLoading = false
        var people: [UserSummary] = []
        var errorMessage: String?
        var pendingSettingsRequest: PeopleSettingsRequest?
    }

    let dependencies: Dependencies
}
```

여기서 reducer는 repository도, network client도, DI container도 모릅니다. `loadPeople`이라는 feature가 이해할 수 있는 action만 압니다.

이것이 실무에서 중요한 이유는 테스트 때문입니다. reducer test는 network stub이 아니라 `loadPeople` closure를 바꿔서 성공/실패/지연 케이스를 검증할 수 있습니다.

## Best practice 3. phase를 명시적으로 모델링합니다

화면이 복잡해질수록 `isLoading`, `errorMessage`, `hasLoaded` 같은 Boolean 조합은 쉽게 꼬입니다. InnoFlow 4.x에서는 `phaseManaged`와 `PhaseMap`을 통해 상태 전이의 의도를 더 분명하게 둘 수 있습니다.

```swift
static var phaseMap: PhaseMap<State, Action, State.Phase> {
    PhaseMap(\State.phase) {
        From(.idle) {
            On(.onAppear, to: .loading)
            On(.refresh, to: .loading)
        }
        From(.loading) {
            On(Action.peopleLoadedCasePath, to: .loaded)
            On(Action.peopleFailedCasePath, to: .failed)
        }
        From(.loaded) {
            On(.refresh, to: .loading)
        }
    }
}
```

phase는 UI 상태 이름이 아니라 feature의 진행 상태입니다. 이 관점으로 보면 loading indicator, retry button, empty state는 phase에서 파생되는 표현이 됩니다.

## Best practice 4. async effect는 취소 정책까지 같이 둡니다

비동기 loading은 단순히 `Task {}`를 만드는 문제가 아닙니다. refresh가 연속으로 들어오면 이전 요청을 취소할지, 중복 요청을 허용할지 결정해야 합니다.

```swift
private func loadPeople() -> EffectTask<Action> {
    let loadPeople = dependencies.loadPeople

    return .run { send, _ in
        do {
            let people = try await loadPeople()
            await send(.peopleLoaded(people))
        } catch {
            await send(.peopleFailed(error.localizedDescription))
        }
    }
    .cancellable("people-feature-load", cancelInFlight: true)
}
```

이 코드는 loading effect와 cancel policy를 한곳에 둡니다. view의 `onAppear`나 button action에 흩어진 `Task`보다 리뷰하기 쉽고 테스트하기도 쉽습니다.

## Best practice 5. navigation은 intent까지만 만듭니다

InnoFlow reducer가 route stack을 직접 조작하면 feature logic이 navigation framework에 묶입니다. InnoSample은 이 대신 one-shot intent를 state에 남깁니다.

```swift
case .openSettings(let request):
    state.pendingSettingsRequest = PeopleSettingsRequest(request: request)
    return .none
```

그 다음 `PeopleFeatureCoordinator`가 이 intent를 소비하고, 상위 `EntireTabCoordinator`가 실제 sibling feature 이동을 중재합니다.

이 구조가 좋은 이유는 명확합니다.

- reducer는 "settings로 가고 싶다"는 intent만 표현합니다.
- coordinator는 그 intent를 navigation command로 바꿉니다.
- sibling feature import는 상위 coordinator에만 모입니다.

InnoFlow와 [`InnoRouter`]({% post_url 2026-02-23-innorouter-swiftui-navigation %})가 서로 침범하지 않는 지점입니다.

## 도입했을 때 얻는 장점

InnoFlow를 잘 쓰면 feature logic이 다음 성질을 갖습니다.

- 상태 전이가 한 파일에서 읽힙니다.
- async loading과 error path가 action으로 드러납니다.
- UI 없이 reducer만 테스트할 수 있습니다.
- navigation intent를 business state와 분리할 수 있습니다.
- feature가 network/DI/router 구현을 직접 알지 않습니다.
- `Reduce`, `Scope`, `IfLet`, `ForEachReducer`로 큰 feature를 작게 쪼갤 수 있습니다.

특히 SwiftUI 앱에서 이 장점은 큽니다. SwiftUI는 view 갱신을 잘하지만, feature의 business state ownership까지 자동으로 정리해 주지는 않습니다. InnoFlow는 그 빠진 경계를 채웁니다.

## 언제 쓰면 좋은가

InnoFlow는 이런 앱에 잘 맞습니다.

- 화면마다 loading/error/retry가 반복되는 앱
- feature test를 view 없이 작성하고 싶은 앱
- SwiftUI view에서 business logic을 덜어내고 싶은 앱
- TCA만큼 큰 app framework는 부담스럽지만 reducer 기반 구조는 원하는 팀
- navigation과 networking은 별도 경계로 분리하고 싶은 팀

반대로 아주 작은 화면이나 일회성 form에는 과할 수 있습니다. 상태 전이가 몇 줄이면 `@State`와 `Observable`만으로 충분합니다.

## InnoSample에서의 결론

[`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice %})에서 InnoFlow는 feature logic을 담당합니다.

- `PeopleFeatureLogic`은 사람 목록 loading과 settings 이동 intent를 관리합니다.
- `PostsFeatureLogic`은 posts loading과 highlights modal intent를 관리합니다.
- `SettingsFeatureLogic`은 todos loading과 people 이동 intent를 관리합니다.
- `Router`는 이 intent를 navigation으로 바꿉니다.
- `App`과 `Layers`는 reducer 내부를 모릅니다.

이 배치가 InnoFlow의 가장 좋은 사용법입니다. **상태 전이는 reducer에, 화면 이동은 router에, 의존성 생성은 DI container에 둡니다.** 그렇게 나누면 SwiftUI feature는 더 테스트 가능하고, 더 예측 가능하며, 더 오래 유지되는 구조가 됩니다.
