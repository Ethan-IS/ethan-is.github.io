---
title: "InnoFlow: SwiftUI를 위한 단방향 아키텍처 프레임워크"
date: 2026-02-23 00:00:00 +0900
translation_key: innoflow-unidirectional-architecture
lang: ko-KR
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, state-management, unidirectional, elm-architecture, tca, swiftui]
author: ethan
toc: true
comments: true
---

## 왜 다시 봐야 할까요?

기존 `InnoFlow` 소개 글은 분명 맞는 설명이었습니다. 다만 중심이 `v2` 시절 문법에 머물러 있었습니다.

- reducer를 `reduce(into:action:)`로 직접 작성하는 방식
- `Store.binding(\.field, ...)` 중심의 예제
- `2.0.0` 패키지 버전
- phase-driven modeling을 부가 기능처럼 다루는 설명

실제 `InnoFlow` 코드베이스는 `3.0.0`에서 방향이 더 명확해졌습니다.

> InnoFlow는 generic app state machine이 아니라, 비즈니스와 도메인 상태 전환을 위한 reducer-first framework입니다.

공식 authoring surface는 이제 `var body: some Reducer<State, Action>`입니다. composition은 `Reduce`, `CombineReducers`, `Scope`, `IfLet`, `IfCaseLet`, `ForEachReducer` 중심으로 정리됐고, phase-heavy feature는 `PhaseMap`이 runtime phase ownership을 맡습니다.

즉, 이번 업데이트의 핵심은 "단방향 상태관리 프레임워크"라는 추상적인 소개를 넘어서, 현재 InnoFlow가 실제로 무엇을 소유하고 어떤 방식으로 쓰이도록 설계되었는지 다시 설명하는 데 있습니다.

---

## InnoFlow가 소유하는 것과 소유하지 않는 것

현재 `ARCHITECTURE_CONTRACT.md` 기준으로 InnoFlow는 아래 영역을 소유합니다.

| 영역 | 핵심 타입 | 설명 |
|---|---|---|
| Reducer authoring | `@InnoFlow`, `Reducer`, `Reduce`, `CombineReducers` | 상태 전환 로직을 reducer body로 정의 |
| Child composition | `Scope`, `IfLet`, `IfCaseLet`, `ForEachReducer` | 자식 feature를 명시적으로 합성 |
| SwiftUI runtime | `Store`, `ScopedStore`, `SelectedStore`, `@BindableField` | SwiftUI에서 상태를 관찰하고 파생 projection을 다루는 계층 |
| Effect runtime | `EffectTask`, `EffectContext`, FIFO queue | 비동기 작업, 취소, 시간 제어 |
| Phase-driven modeling | `PhaseMap`, `PhaseTransitionGraph`, `ActionMatcher` | 의미 있는 phase를 가진 feature의 전환 계약 |
| Testing / observability | `TestStore`, `InnoFlowTesting`, `StoreInstrumentation` | 결정론적 테스트와 runtime 계측 |

반대로 아래는 InnoFlow가 의도적으로 소유하지 않습니다.

- concrete navigation stack
- transport, reconnect, session lifecycle
- construction-time dependency graph
- window, scene, immersive space 같은 앱 런타임 경계

이 경계가 중요한 이유는 단순합니다. InnoFlow가 모든 앱 상태를 가져가면 오히려 reducer가 거대한 orchestration 허브가 됩니다. `3.0`의 문서들은 이걸 분명히 피합니다.

---

## 공식 authoring surface

`3.0`에서 가장 먼저 바뀐 감각은 여기입니다. 이제 feature를 설명할 때 `reduce(into:action:)`를 먼저 보여주기보다 `body` 기반 reducer composition을 보여주는 것이 맞습니다.

### `@InnoFlow` + `var body: some Reducer`

```swift
import InnoFlow

@InnoFlow
struct CounterFeature {
  struct State: Equatable, Sendable, DefaultInitializable {
    var count = 0
    @BindableField var step = 1
  }

  enum Action: Equatable, Sendable {
    case increment
    case decrement
    case setStep(Int)
  }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .increment:
        state.count += state.step
        return .none

      case .decrement:
        state.count -= state.step
        return .none

      case .setStep(let step):
        state.step = max(1, step)
        return .none
      }
    }
  }
}
```

이게 `3.0`에서 말하는 공식 feature authoring입니다.

- feature는 `State`, `Action`, `body`를 가진다
- `@InnoFlow`는 이 구조를 기반으로 필요한 reducer entry point를 생성한다
- 사람이 직접 써야 하는 부분은 composition과 domain logic이지, boilerplate가 아니다

### `Reduce`는 primitive reducer다

`Reduce`는 closure 기반 primitive reducer입니다. InnoFlow의 다른 composition surface도 결국 여기에 얹힙니다.

```swift
Reduce<State, Action> { state, action in
  // mutate state
  // return EffectTask<Action>
}
```

핵심은 여전히 같습니다. 상태는 reducer 안에서만 바뀌고, effect는 `EffectTask<Action>`로 돌아옵니다. 다만 `3.0`은 그 진입점을 `body` composition으로 통일했습니다.

---

## Composition surface

InnoFlow 3.0은 여러 authoring 스타일을 늘리는 대신, 작은 composition surface를 공식 경로로 고정했습니다.

### `CombineReducers`

부모 logic과 보조 reducer를 선언 순서대로 결합합니다.

```swift
var body: some Reducer<State, Action> {
  CombineReducers {
    Reduce { state, action in
      switch action {
      case .load:
        state.isLoading = true
        return .send(.child(.start))
      case .child(.finished):
        state.isLoading = false
        return .none
      default:
        return .none
      }
    }

    AnalyticsReducer()
  }
}
```

### `Scope`

항상 존재하는 child state를 부모 action 공간으로 끌어올릴 때 씁니다.

```swift
@InnoFlow
struct ParentFeature {
  struct State: Equatable, Sendable, DefaultInitializable {
    var child = ChildFeature.State()
    var isLoading = false
  }

  enum Action: Equatable, Sendable {
    case load
    case child(ChildFeature.Action)
  }

  var body: some Reducer<State, Action> {
    CombineReducers {
      Reduce { state, action in
        switch action {
        case .load:
          state.isLoading = true
          return .send(.child(.start))
        case .child(.finished):
          state.isLoading = false
          return .none
        default:
          return .none
        }
      }

      Scope(
        state: \.child,
        action: .childCasePath,
        reducer: ChildFeature()
      )
    }
  }
}
```

`@InnoFlow`가 붙어 있으면 `case child(ChildFeature.Action)` 같은 케이스에서 `childCasePath`를 자동 합성해 줍니다.

### `IfLet`, `IfCaseLet`, `ForEachReducer`

이 세 가지는 child composition의 조건부/컬렉션 버전입니다.

- `IfLet`: optional child state
- `IfCaseLet`: enum case로 열리는 child state
- `ForEachReducer`: collection row reducer

이 surface가 중요한 이유는 "child reducer를 어떻게 올릴지"가 더 이상 팀마다 다르게 흩어지지 않기 때문입니다.

---

## SwiftUI runtime

### `Store`는 SwiftUI-first runtime이다

기본 사용법은 여전히 단순합니다.

```swift
import InnoFlow
import SwiftUI

struct CounterView: View {
  @State private var store: Store<CounterFeature>

  init(store: Store<CounterFeature> = Store(reducer: CounterFeature())) {
    _store = State(initialValue: store)
  }

  var body: some View {
    VStack(spacing: 20) {
      Text("Count: \(store.count)")
        .font(.largeTitle)

      HStack(spacing: 24) {
        Button("−") { store.send(.decrement) }
        Button("+") { store.send(.increment) }
      }

      Stepper(
        "Step: \(store.step)",
        value: store.binding(\.$step, send: CounterFeature.Action.setStep)
      )
    }
  }
}
```

여기서 바뀐 포인트는 binding 문법입니다. `3.0` 기준 문서는 projected key path를 사용합니다.

- 예전 설명: `store.binding(\.step, ...)`
- 현재 surface: `store.binding(\.$step, ...)`

binding도 의도적으로 explicit opt-in입니다. `@BindableField`가 붙은 프로퍼티만 SwiftUI에서 바인딩할 수 있습니다.

### `ScopedStore`

mutable child flow는 `ScopedStore`로 가져갑니다.

```swift
let child = store.scope(state: \.child, action: .childCasePath)
```

이건 "부모 상태 일부를 자식 뷰에 그냥 넘긴다"는 감각보다, 자식 feature에 맞는 mutable scope를 별도로 만든다는 개념에 가깝습니다.

### `SelectedStore`

반대로 read-only derived value는 `SelectedStore`가 공식 경로입니다.

```swift
let summary = store.select(dependingOn: (\.profile, \.permissions)) { profile, permissions in
  DashboardBadge(
    title: profile.name,
    isReady: profile.isReady && permissions.isReady
  )
}
```

`SelectedStore`가 중요한 이유는 단순 memoization이 아니라, 큰 SwiftUI 뷰에서 비싼 `Equatable` projection만 선택적으로 refresh하게 만들기 때문입니다.

선택 기준은 이렇습니다.

- mutable child flow면 `ScopedStore`
- read-only projection이면 `SelectedStore`
- 의존 slice를 1~3개로 명시할 수 있으면 `select(dependingOn:)`
- 그게 안 되면 `select { ... }`를 always-refresh fallback으로 사용

### Preview

현재 canonical preview entry point는 `Store.preview(...)`입니다.

```swift
#Preview("Counter") {
  CounterView(
    store: .preview(
      reducer: CounterFeature(),
      initialState: .init(count: 3, step: 2)
    )
  )
}
```

preview 전용 초기화를 production wiring에 섞지 않게 하려는 의도가 여기에도 드러납니다.

---

## Effect runtime

`EffectTask<Action>`는 여전히 effect DSL의 중심입니다.

- `.none`
- `.send(action)`
- `.run { send, context in ... }`
- `.merge(...)`
- `.concatenate(...)`
- `.cancel(id)`
- `.cancellable(id:cancelInFlight:)`
- `.debounce(id:for:)`
- `.throttle(id:for:leading:trailing:)`
- `.animation(_:)`

하지만 `3.0`에서 중요한 변화는 DSL 목록보다 runtime contract입니다.

### `EffectContext`

새 코드에서는 `.run { send, context in ... }` 스타일을 우선 쓰는 것이 맞습니다.

```swift
return .run { send, context in
  do {
    try await context.sleep(for: .milliseconds(300))
    try await context.checkCancellation()
    await send(.finished)
  } catch is CancellationError {
    return
  }
}
```

이유는 명확합니다.

- `StoreClock`가 debounce/throttle과 effect delay를 함께 제어
- 테스트에서 wall clock 의존을 줄임
- cancellation check를 ad-hoc하게 흩뿌리지 않아도 됨

### FIFO queue semantics

`Store`는 reducer input과 effect follow-up action을 단일 FIFO queue로 처리합니다.

| 동작 | 의미 |
|---|---|
| `.send` | 즉시 follow-up action처럼 보이지만 reducer-reentrant가 아니라 queue에 들어감 |
| `.run` | suspension boundary 이후 같은 queue로 다시 진입 |
| `.concatenate` | 선언 순서 유지 |
| `.merge` | 자식 completion 순서 기준 |

즉, InnoFlow는 effect ordering을 암묵적 재진입에 맡기지 않고 문서화된 contract로 고정합니다.

### `StoreInstrumentation`

운영 환경 observability도 공식 surface가 생겼습니다.

```swift
let instrumentation: StoreInstrumentation<Feature.Action> = .combined(
  .osLog(logger: logger),
  .sink { event in
    switch event {
    case .runStarted:
      metrics.increment("feature.effect.run_started")
    case .runFinished:
      metrics.increment("feature.effect.run_finished")
    case .actionEmitted:
      metrics.increment("feature.effect.emitted")
    case .actionDropped:
      metrics.increment("feature.effect.dropped")
    case .effectsCancelled:
      metrics.increment("feature.effect.cancelled")
    }
  }
)
```

핵심은 vendor SDK를 core reducer semantics에 섞지 않고 `sink`, `osLog`, `combined` 같은 확장 지점으로 분리했다는 점입니다.

---

## Phase-driven modeling

`3.0`에서 가장 크게 체감되는 새 장은 여기입니다. 다만 오해하면 안 됩니다.

> `PhaseMap`은 runtime phase ownership layer이고, `PhaseTransitionGraph`는 topology validation 도구입니다.

즉, InnoFlow가 generic FSM runtime으로 변한 것은 아닙니다.

### `PhaseMap`

```swift
@InnoFlow
struct ItemsFeature {
  struct State: Equatable, Sendable, DefaultInitializable {
    enum Phase: Hashable, Sendable {
      case idle
      case loading
      case loaded
      case failed
    }

    var phase: Phase = .idle
    var items: [Item] = []
    var errorMessage: String?
  }

  enum Action: Equatable, Sendable {
    case load
    case _loaded([Item])
    case _failed(String)
  }

  static var phaseMap: PhaseMap<State, Action, State.Phase> {
    PhaseMap(\.phase) {
      From(.idle) {
        On(.load, to: .loading)
      }
      From(.loading) {
        On(Action.loadedCasePath, to: .loaded)
        On(Action.failedCasePath, to: .failed)
      }
      From(.loaded) {
        On(.load, to: .loading)
      }
      From(.failed) {
        On(.load, to: .loading)
      }
    }
  }

  static var phaseGraph: PhaseTransitionGraph<State.Phase> {
    phaseMap.derivedGraph
  }

  var body: some Reducer<State, Action> {
    let phaseMap: PhaseMap<State, Action, State.Phase> = Self.phaseMap

    return Reduce { state, action in
      switch action {
      case .load:
        return .none
      case ._loaded(let items):
        state.items = items
        return .none
      case ._failed(let message):
        state.errorMessage = message
        return .none
      }
    }
    .phaseMap(phaseMap)
  }
}
```

이 모델에서 중요한 규칙은 다음입니다.

- `PhaseMap`은 base reducer 뒤에서 실행된다
- `PhaseMap`이 활성화되면 base reducer가 owned phase를 직접 바꾸지 않는 것이 원칙이다
- unmatched phase/action pair는 기본적으로 legal no-op이다
- stricter coverage는 runtime이 아니라 test에서 opt-in한다

### 언제 써야 하나

- `idle -> loading -> loaded`처럼 phase enum이 이미 있을 때
- legal transition이 business contract의 일부일 때
- `state.phase = ...`가 reducer 여러 branch에 퍼지고 있을 때

### 언제 쓰지 말아야 하나

- route stack 대체
- session lifecycle
- reconnect/transport retry
- dependency/container wiring
- 단순 effect bookkeeping

### `PhaseTransitionGraph`

`PhaseTransitionGraph`는 graph topology validation에 집중합니다.

```swift
let report = ItemsFeature.phaseGraph.validationReport(
  allPhases: [.idle, .loading, .loaded, .failed],
  root: .idle,
  terminalPhases: [.loaded]
)

precondition(report.issues.isEmpty)
```

payload가 있는 action 기준으로 phase를 나눠야 할 때는 `ActionMatcher`와 case path 기반 `On(...)` 규칙을 우선 쓰는 것이 현재 문서가 권장하는 방향입니다.

여기서 graph는 "정적 계약을 검증"하는 역할입니다. guard-bearing transition metadata까지 이 graph에 욱여넣지 않는 것도 `3.0` 문서가 강조하는 설계 의도입니다.

`validatePhaseTransitions(...)`는 남아 있지만, 새 글에서는 backward compatibility로만 언급하고 canonical path로 추천하지 않습니다.

---

## Testing

`InnoFlowTesting`의 `TestStore`는 여전히 핵심이지만, `3.0`에서는 phase-aware testing이 더 선명해졌습니다.

```swift
import InnoFlowTesting

@Test
@MainActor
func loadFlow() async {
  let store = TestStore(reducer: ItemsFeature())

  let phaseMap: PhaseMap<ItemsFeature.State, ItemsFeature.Action, ItemsFeature.State.Phase> =
    ItemsFeature.phaseMap

  await store.send(.load, through: phaseMap) {
    $0.phase = .loading
  }

  await store.receive(._loaded(items), through: phaseMap) {
    $0.phase = .loaded
    $0.items = items
  }
}
```

여기서 포인트는 `through: phaseMap`입니다. phase-heavy feature는 reducer 테스트와 phase ownership 검증을 같은 surface에서 다룹니다.

추가로 기억할 만한 점은 이렇습니다.

- parent `TestStore`를 `scope(state:action:)`해서 child 테스트를 이어갈 수 있다
- `StoreClock.manual(...)`로 debounce/throttle과 effect delay를 제어할 수 있다
- state mismatch diagnostics가 diff 중심으로 정리되어 있다

즉, 테스트도 "effect가 어떻게 돌았는가"보다 "문서화된 transition contract를 지키는가"에 더 가까워졌습니다.

---

## 언제 무엇을 쓰면 될까요?

| 상황 | 권장 surface |
|---|---|
| 단순 feature logic | `@InnoFlow` + `Reduce` |
| 부모/자식 조합 | `CombineReducers` + `Scope` |
| optional child | `IfLet` |
| enum-backed child | `IfCaseLet` |
| row collection | `ForEachReducer` |
| mutable child flow | `ScopedStore` |
| read-only expensive projection | `SelectedStore` |
| phase-heavy feature | `PhaseMap` + `phaseMap.derivedGraph` |
| runtime observability | `StoreInstrumentation` |
| preview/review 환경 | `Store.preview(...)` |

이 표만 봐도 `3.0`의 의도가 드러납니다. InnoFlow는 one big store pattern을 강요하는 프레임워크가 아니라, reducer composition과 state ownership을 작게 나눈 뒤 SwiftUI runtime이 그것을 안정적으로 실행하도록 돕는 프레임워크입니다.

---

## 설치와 요구사항

```swift
dependencies: [
  .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "3.0.0")
]
```

현재 패키지 요구사항은 다음과 같습니다.

- iOS 18+
- macOS 15+
- tvOS 18+
- watchOS 11+
- visionOS 2+
- Swift tools 6.2

테스트 타깃에서 `TestStore`, `ManualTestClock` 등을 쓰려면 `InnoFlowTesting`도 함께 추가하면 됩니다.

---

## 마무리

지금의 InnoFlow를 한 문장으로 요약하면 이렇습니다.

> InnoFlow는 SwiftUI 상태관리 라이브러리라기보다, business/domain transition을 reducer composition과 explicit runtime contract로 정리하는 framework입니다.

그래서 `3.0`에서 특히 좋아진 점은 다음과 같습니다.

1. feature authoring이 `body` 기반 reducer composition으로 명확해졌습니다.
2. child composition surface가 `Scope` 계열로 정리됐습니다.
3. `SelectedStore`, `Store.preview`, `StoreInstrumentation`처럼 SwiftUI 운영 surface가 선명해졌습니다.
4. `PhaseMap`과 `PhaseTransitionGraph`가 phase-heavy feature의 계약을 더 잘 드러냅니다.
5. navigation, session, dependency graph ownership을 프레임워크 바깥에 남겨 경계가 더 깨끗해졌습니다.

상태가 복잡하다고 해서 모든 앱 흐름을 하나의 giant state machine으로 밀어넣고 싶지는 않을 때, InnoFlow 3.0은 꽤 좋은 균형점을 제공합니다.

## 참고 자료

- [InnoFlow GitHub 저장소](https://github.com/InnoSquadCorp/InnoFlow)
- [Architecture Contract](https://github.com/InnoSquadCorp/InnoFlow/blob/main/ARCHITECTURE_CONTRACT.md)
- [Phase-Driven Modeling](https://github.com/InnoSquadCorp/InnoFlow/blob/main/PHASE_DRIVEN_MODELING.md)
- [Canonical Sample App](https://github.com/InnoSquadCorp/InnoFlow/tree/main/Examples/InnoFlowSampleApp)
