---
title: "InnoFlow: A one-way architecture framework for SwiftUI"
date: 2026-02-23 00:00:00 +0900
translation_key: innoflow-unidirectional-architecture
lang: en
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, state-management, unidirectional, elm-architecture, tca, swiftui]
author: ethan
toc: true
comments: true
---

## Why revisit it now?

The original InnoFlow post was directionally right, but it was still centered on the `v2` mental model.

- reducers were shown as handwritten `reduce(into:action:)`
- examples used `Store.binding(\.field, ...)`
- installation still pointed to `2.0.0`
- phase-driven modeling looked like an extra feature instead of a core `v3` story

The actual codebase has moved. In `3.0.0`, the framework is much more explicit about what it is:

> InnoFlow is not a generic app state machine. It is a reducer-first framework for business and domain state transitions.

The official authoring surface is now `var body: some Reducer<State, Action>`. Composition is centered on `Reduce`, `CombineReducers`, `Scope`, `IfLet`, `IfCaseLet`, and `ForEachReducer`. Phase-heavy features use `PhaseMap` as the runtime phase ownership layer.

So this update is not about polishing wording. It is about describing the framework that exists today.

---

## What InnoFlow owns, and what it does not

According to the current `ARCHITECTURE_CONTRACT.md`, InnoFlow owns these areas.

| Area | Core types | Responsibility |
|---|---|---|
| Reducer authoring | `@InnoFlow`, `Reducer`, `Reduce`, `CombineReducers` | define state transitions through reducer composition |
| Child composition | `Scope`, `IfLet`, `IfCaseLet`, `ForEachReducer` | compose child features explicitly |
| SwiftUI runtime | `Store`, `ScopedStore`, `SelectedStore`, `@BindableField` | observe state and derive projections in SwiftUI |
| Effect runtime | `EffectTask`, `EffectContext`, FIFO queue | async work, timing, cancellation |
| Phase-driven modeling | `PhaseMap`, `PhaseTransitionGraph`, `ActionMatcher` | phase contracts for phase-heavy features |
| Testing / observability | `TestStore`, `InnoFlowTesting`, `StoreInstrumentation` | deterministic testing and runtime instrumentation |

And it intentionally does not own:

- concrete navigation stacks
- transport, reconnect, and session lifecycle
- construction-time dependency graphs
- window, scene, and immersive-space runtime concerns

That boundary matters. If a framework owns all of those concerns at once, reducers turn into giant orchestration hubs. InnoFlow 3.0 explicitly avoids that.

---

## The official authoring surface

The biggest practical shift in `3.0` is this: the framework no longer presents handwritten `reduce(into:action:)` as the primary way to author features.

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

This is the official `3.0` feature shape.

- a feature defines `State`, `Action`, and `body`
- `@InnoFlow` generates the reducer entry point from that composition
- the handwritten part is now domain logic and composition, not boilerplate

### `Reduce` is the primitive

`Reduce` is the closure-backed primitive reducer that the rest of the surface builds on.

```swift
Reduce<State, Action> { state, action in
  // mutate state
  // return EffectTask<Action>
}
```

The unidirectional model itself has not changed. State still mutates in the reducer, and effects still come back as `EffectTask<Action>`. What changed is the official authoring entry point.

---

## Composition surface

InnoFlow 3.0 intentionally keeps the composition surface small instead of growing multiple authoring styles.

### `CombineReducers`

Use `CombineReducers` to run parent logic and helper reducers in declaration order.

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

Use `Scope` when child state is always present and should be lifted into the parent action space.

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

With `@InnoFlow`, cases like `case child(ChildFeature.Action)` synthesize `childCasePath` automatically.

### `IfLet`, `IfCaseLet`, `ForEachReducer`

These complete the official child-composition set:

- `IfLet` for optional child state
- `IfCaseLet` for enum-backed child state
- `ForEachReducer` for collection-backed child state

That is a meaningful part of the `3.0` story. Child reducer composition is no longer a pattern each team has to reinvent.

---

## SwiftUI runtime

### `Store` is a SwiftUI-first runtime

The basic usage is still simple.

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

The binding syntax is one of the concrete `v3` updates:

- older style: `store.binding(\.step, ...)`
- current style: `store.binding(\.$step, ...)`

Bindings are intentionally explicit opt-in. Only `@BindableField` properties participate in SwiftUI binding.

### `ScopedStore`

Mutable child flows use `ScopedStore`.

```swift
let child = store.scope(state: \.child, action: .childCasePath)
```

That is different from a read-only projection. It creates a mutable child boundary that participates in the store runtime.

### `SelectedStore`

For expensive read-only derived values, `SelectedStore` is now the official path.

```swift
let summary = store.select(dependingOn: (\.profile, \.permissions)) { profile, permissions in
  DashboardBadge(
    title: profile.name,
    isReady: profile.isReady && permissions.isReady
  )
}
```

This is not just a convenience memoization trick. It is the documented way to let large SwiftUI views observe only an `Equatable` projection.

The rule of thumb is:

- mutable child flow -> `ScopedStore`
- read-only derived value -> `SelectedStore`
- one to three explicit dependency slices -> `select(dependingOn:)`
- otherwise -> `select { ... }` as the always-refresh fallback

### Preview

The canonical preview entry point is `Store.preview(...)`.

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

That keeps preview-only setup explicit without changing production store wiring.

---

## Effect runtime

`EffectTask<Action>` remains the only effect DSL:

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

But the important `3.0` story is the runtime contract around it.

### `EffectContext`

New code should prefer `.run { send, context in ... }`.

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

That matters because:

- `StoreClock` controls both scheduling operators and explicit delays
- tests stay deterministic
- cancellation checks stop spreading as ad hoc code

### FIFO queue semantics

`Store` dispatches reducer input and effect follow-up actions through a single FIFO queue.

| Behavior | Meaning |
|---|---|
| `.send` | immediate follow-up, but queued rather than reducer-reentrant |
| `.run` | re-enters the same queue after its suspension boundary |
| `.concatenate` | preserves declaration order |
| `.merge` | observes child completion order |

In other words, effect ordering is documented runtime behavior, not incidental implementation detail.

### `StoreInstrumentation`

`3.0` also gives observability a clearer public surface.

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

The design intent is clear: backend-specific metrics or traces should plug into `sink`, `osLog`, or `combined`, not alter reducer semantics.

---

## Phase-driven modeling

This is the most visible new chapter in InnoFlow 3.0, but it is easy to misunderstand.

> `PhaseMap` is the runtime phase ownership layer. `PhaseTransitionGraph` is the topology validation tool.

That does not turn InnoFlow into a generic FSM runtime.

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

The important rules are:

- `PhaseMap` runs after the base reducer
- once active, `PhaseMap` owns the declared phase key path
- unmatched phase/action pairs remain legal no-ops by default
- stronger coverage is opt-in in tests, not a forced runtime rule

### When to use it

- when a feature already has a meaningful phase enum
- when legal transitions are part of the business contract
- when imperative `state.phase = ...` writes are spreading across reducer branches

### When not to use it

- route stacks
- session lifecycle
- reconnect or transport retries
- dependency graph wiring
- generic effect bookkeeping

### `PhaseTransitionGraph`

`PhaseTransitionGraph` is for static topology validation.

```swift
let report = ItemsFeature.phaseGraph.validationReport(
  allPhases: [.idle, .loading, .loaded, .failed],
  root: .idle,
  terminalPhases: [.loaded]
)

precondition(report.issues.isEmpty)
```

When phase movement depends on payload-bearing actions, the current docs prefer case-path based `On(...)` rules and `ActionMatcher`-style matching over ad hoc graph metadata.

The graph is deliberately kept focused on validation. Guard-bearing transition metadata and runtime conditional resolution stay out of it on purpose.

`validatePhaseTransitions(...)` still exists, but it is now a backward-compatibility surface, not the recommended one for new examples.

---

## Testing

`TestStore` remains central, but `3.0` makes phase-aware testing a much more explicit part of the story.

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

The important part is `through: phaseMap`. A phase-heavy feature can now test reducer behavior and documented phase ownership through the same public surface.

A few other testing points are worth calling out:

- project child tests from a parent `TestStore` with `scope(state:action:)`
- inject `StoreClock.manual(...)` for debounce/throttle and delay control
- diff-oriented mismatch diagnostics make reducer failures easier to read

Testing moved closer to "verify the documented contract" and further away from "poke the runtime and hope it behaves."

---

## Which surface should you use?

| Situation | Recommended surface |
|---|---|
| simple feature logic | `@InnoFlow` + `Reduce` |
| parent/child composition | `CombineReducers` + `Scope` |
| optional child | `IfLet` |
| enum-backed child | `IfCaseLet` |
| collection-backed child | `ForEachReducer` |
| mutable child flow | `ScopedStore` |
| read-only expensive projection | `SelectedStore` |
| phase-heavy feature | `PhaseMap` + `phaseMap.derivedGraph` |
| runtime observability | `StoreInstrumentation` |
| previews and review passes | `Store.preview(...)` |

That table captures the design intent well. InnoFlow is not a "one giant store" framework. It is a framework that keeps reducer composition, state ownership, and runtime behavior explicit.

---

## Installation and requirements

```swift
dependencies: [
  .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "3.0.0")
]
```

Current package requirements:

- iOS 18+
- macOS 15+
- tvOS 18+
- watchOS 11+
- visionOS 2+
- Swift tools 6.2

If tests need `TestStore`, `ManualTestClock`, and phase-aware helpers, add `InnoFlowTesting` to the test target as well.

---

## Closing thoughts

The shortest accurate description of InnoFlow today is this:

> InnoFlow is less a "SwiftUI state management library" and more a reducer-first framework that organizes business and domain transitions around explicit runtime contracts.

That is why `3.0` feels different.

1. feature authoring is now clearly centered on reducer body composition
2. child composition is standardized around the `Scope` family
3. SwiftUI runtime surfaces like `SelectedStore`, `Store.preview`, and `StoreInstrumentation` are much sharper
4. `PhaseMap` and `PhaseTransitionGraph` make phase-heavy features easier to document and validate
5. navigation, session lifecycle, and dependency graph ownership remain outside the framework boundary

If your state logic is getting more complex but you do not want every app concern shoved into one giant state machine, InnoFlow 3.0 offers a more disciplined balance.

## References

- [InnoFlow GitHub Repository](https://github.com/InnoSquadCorp/InnoFlow)
- [Architecture Contract](https://github.com/InnoSquadCorp/InnoFlow/blob/main/ARCHITECTURE_CONTRACT.md)
- [Phase-Driven Modeling](https://github.com/InnoSquadCorp/InnoFlow/blob/main/PHASE_DRIVEN_MODELING.md)
- [Canonical Sample App](https://github.com/InnoSquadCorp/InnoFlow/tree/main/Examples/InnoFlowSampleApp)
