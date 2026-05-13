---
title: "InnoFlow Best Practices: Managing SwiftUI Feature Logic with One-Way State"
date: 2026-02-23 00:00:00 +0900
translation_key: innoflow-unidirectional-architecture
lang: en
description: "Why InnoFlow is useful, how to keep SwiftUI feature logic behind reducer/state/action/effect boundaries, and how InnoSample applies that pattern."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, state-management, unidirectional, reducer, swiftui, architecture, testing]
author: ethan
toc: true
comments: true
---

## Why feature state gets hard in real SwiftUI apps

SwiftUI is excellent for building small screens quickly. As an app grows, feature logic often leaks into views.

- Each `onAppear` gets its own loading rules.
- Error, retry, refresh, and empty states scatter across the view tree.
- Async cancellation policy varies by screen.
- Navigation intent mixes with business state.
- Tests become view-heavy while logic coverage stays thin.

[`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow) addresses this by organizing feature logic around reducers, state, actions, and effects. It is not trying to replace SwiftUI. It is trying to keep business and domain transitions out of the view layer.

## The boundary InnoFlow should own

InnoFlow should own business and domain state transitions inside a feature.

It should own:

- feature state
- user and system actions
- async effects
- loading/error/loaded phases
- feature-local one-shot intent before navigation consumes it
- reducer composition

It should not own:

- concrete navigation stacks
- URLSession or transport retry
- dependency graph construction
- SwiftUI layout
- app lifecycle, scenes, or windows

The boundary matters. If InnoFlow becomes a giant application framework that also owns networking and navigation, its value gets blurry. Keep it small: feature `Logic` targets own state transitions, and everything else stays outside.

## Installation and official authoring surface

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "4.0.0")
]
```

Use the products by role.

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

InnoFlow 4.x's recommended authoring surface is `@InnoFlow` plus `var body: some Reducer<State, Action>`.

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

## Best practice 1. Keep it inside `Logic` targets

InnoSample splits leaf features into `Interface / Logic / UI / Router / Testing / Tests`. InnoFlow belongs in `Logic`.

That placement gives you clean boundaries:

- reducers do not know SwiftUI layout
- reducers do not know InnoRouter route stacks
- reducers do not construct network clients
- UI renders state and sends actions
- Router consumes navigation intent

InnoFlow owns how a feature thinks. SwiftUI owns how that state is displayed.

## Best practice 2. Pass dependencies into reducers

InnoSample's `PeopleFeatureReducer` receives use cases through a small dependency bundle.

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

The reducer does not know repositories, network clients, or DI containers. It only knows a feature-level operation: `loadPeople`.

That choice pays off in tests. Reducer tests can swap the closure for success, failure, and delayed paths without building remote infrastructure.

## Best practice 3. Model phases explicitly

As screens grow, combinations of `isLoading`, `errorMessage`, and `hasLoaded` become hard to reason about. InnoFlow 4.x lets you express intended state transitions with `phaseManaged` and `PhaseMap`.

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

Phase is not a visual detail. It is the feature's progress model. Spinners, retry buttons, and empty states can then derive from that model.

## Best practice 4. Keep async effects and cancellation together

Async loading is not just creating a `Task`. Refresh and repeated appearance events need a policy.

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

The effect and cancellation policy live together. That is easier to review and test than scattered `Task` blocks in views.

## Best practice 5. Create navigation intent, not navigation stacks

If a reducer directly mutates route stacks, feature logic becomes tied to the navigation framework. InnoSample instead leaves one-shot intent in state.

```swift
case .openSettings(let request):
    state.pendingSettingsRequest = PeopleSettingsRequest(request: request)
    return .none
```

Then the feature coordinator consumes that intent and the root `EntireTabCoordinator` mediates the actual sibling-feature movement.

This keeps responsibilities separate:

- reducer says "I want to open settings"
- coordinator turns that intent into a route command
- sibling imports remain at the parent boundary

That is where InnoFlow and [`InnoRouter`]({% post_url 2026-02-23-innorouter-swiftui-navigation-en %}) complement each other.

## What you gain by adopting it

Used well, InnoFlow gives feature logic these properties:

- state transitions are readable in one place
- async loading and error paths are explicit actions
- reducers can be tested without UI
- navigation intent can stay separate from route execution
- features do not know network, DI, or router implementations
- larger features can be decomposed with `Reduce`, `Scope`, `IfLet`, and `ForEachReducer`

SwiftUI handles rendering well, but it does not automatically define ownership for business state. InnoFlow fills that missing boundary.

## When it is a good fit

InnoFlow is a good fit for:

- apps with repeated loading/error/retry flows
- teams that want logic tests without booting views
- SwiftUI features that are accumulating business logic
- teams that like reducer-based design but do not want a broad application framework
- apps that keep navigation and networking in separate boundaries

It may be too much for a very small screen or a one-off form. If the state transition fits in a few lines, `@State` and `Observable` may be enough.

## How InnoSample uses it

In [`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice-en %}), InnoFlow owns feature logic.

- `PeopleFeatureLogic` manages loading people and emitting settings intent.
- `PostsFeatureLogic` manages loading posts and highlights modal intent.
- `SettingsFeatureLogic` manages loading todos and emitting people intent.
- Routers consume intent and perform navigation.
- `App` and `Layers` do not know reducer internals.

That is the best way to use InnoFlow: **state transitions in reducers, navigation in routers, dependency construction in DI containers.** The result is SwiftUI feature code that is easier to test, easier to reason about, and easier to keep stable over time.
