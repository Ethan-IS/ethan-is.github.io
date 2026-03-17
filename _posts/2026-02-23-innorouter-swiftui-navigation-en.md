---
title: "InnoRouter: A type-safe navigation framework for SwiftUI"
date: 2026-02-23 00:00:00 +0900
translation_key: innorouter-swiftui-navigation
lang: en
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, navigation, swiftui, coordinator, clean-architecture, unidirectional]
author: ethan
toc: true
comments: true
---

## Why revisit it now?

`NavigationStack` made SwiftUI navigation much cleaner, but real apps still run into the same problems.

- path state gets scattered across screens and features
- authentication, deep links, logging, and policy checks leak into view code
- "the user moved from A to B" is hard to model as testable data

`InnoRouter` 3.0 approaches this as a typed navigation runtime, not as a set of push/pop helpers. The important part is the boundary: `RouteStack` for state, `NavigationCommand` for execution, and SwiftUI-facing hosts and stores that keep mutation authority in one place.

This post reframes InnoRouter around the runtime surface that exists in the codebase today.

---

## What InnoRouter actually owns

According to the current README, InnoRouter owns five areas.

| Area | Core types | Responsibility |
|---|---|---|
| Stack state | `RouteStack`, `RouteStackValidator` | current navigation snapshot and validation rules |
| Command execution | `NavigationCommand`, `NavigationEngine`, `NavigationResult` | typed push/pop/replace execution |
| SwiftUI authority | `NavigationStore`, `NavigationHost`, `@EnvironmentNavigationIntent` | stack routing authority for views |
| Modal authority | `ModalStore`, `ModalHost`, `ModalIntent` | `sheet` and `fullScreenCover` routing |
| Deep-link planning | `DeepLinkMatcher`, `DeepLinkPipeline`, `PendingDeepLink`, `NavigationPlan` | URL matching and explicit execution planning |

Just as important: InnoRouter does not try to become your whole application state machine.

Keep these concerns outside it:

- authentication and session lifecycle
- business workflow state
- networking retry or transport state
- feature-local presentation like `alert` and `confirmationDialog`

---

## The runtime model

At the center of InnoRouter are `RouteStack` and `NavigationCommand`.

### 1. `RouteStack`

`RouteStack` is the value-type snapshot of your current stack navigation state.

```swift
import InnoRouter

let stack = try RouteStack<HomeRoute>(
    validating: [.list, .detail(id: "123")],
    using: .nonEmpty.combined(with: .rooted(at: .list))
)
```

`RouteStackValidator` lets you pull app invariants closer to the state model.

- should empty stacks be allowed?
- must the stack start from a specific root?
- should duplicate routes be rejected?

### 2. `NavigationCommand`

Navigation changes are expressed as commands.

- `.push(route)`
- `.pop`
- `.popCount(n)`
- `.popTo(route)`
- `.popToRoot`
- `.replace([route])`
- `.sequence([...])`

That changes the discussion from "which screen did we end up on?" to "which command did we execute?" which is much easier to test and log.

### 3. `NavigationEngine` and result types

Execution returns `NavigationResult`, so failures and cancellations are part of the public surface instead of hidden side effects.

InnoRouter distinguishes three execution semantics.

| Mode | Best use case |
|---|---|
| Single command | one normal push/pop |
| Batch | sequential commands with aggregated observation |
| Transaction | all-or-nothing commits with rollback semantics |

`.sequence` is not a transaction. It runs left-to-right, and earlier successful steps remain applied even if a later step fails.

---

## The SwiftUI entry point

The most important SwiftUI type is `NavigationStore`.

### `NavigationStore` is shared authority

`NavigationStore` is not just a convenience wrapper. It is the stack-routing authority.

- it owns the current `RouteStack`
- it applies middleware
- it reconciles `NavigationStack(path:)`
- it converts `NavigationIntent` into semantic command execution

The baseline example should now match `Examples/StandaloneExample.swift`.

```swift
import SwiftUI
import InnoRouter
import InnoRouterMacros

@Routable
enum HomeRoute {
    case list
    case detail(id: String)
    case settings
}

struct AppRoot: View {
    @State private var store = try! NavigationStore<HomeRoute>(
        initialPath: [.list],
        configuration: NavigationStoreConfiguration(
            routeStackValidator: .nonEmpty.combined(with: .rooted(at: .list))
        )
    )

    var body: some View {
        NavigationHost(store: store) { route in
            switch route {
            case .list:
                HomeListView()
            case .detail(let id):
                DetailView(id: id)
            case .settings:
                SettingsView()
            }
        } root: {
            HomeListView()
        }
    }
}
```

### Views emit intent instead of mutating state

Views should send intent through `@EnvironmentNavigationIntent`.

```swift
struct HomeListView: View {
    @EnvironmentNavigationIntent(HomeRoute.self) private var navigationIntent

    var body: some View {
        List {
            Button("Detail") {
                navigationIntent.send(.go(.detail(id: "123")))
            }

            Button("Settings") {
                navigationIntent.send(.go(.settings))
            }

            Button("Back") {
                navigationIntent.send(.back)
            }
        }
    }
}
```

The current stack intent surface is:

```swift
public enum NavigationIntent<R: Route>: Sendable, Equatable {
    case go(R)
    case goMany([R])
    case back
    case backBy(Int)
    case backTo(R)
    case backToRoot
    case resetTo([R])
}
```

Older descriptions sometimes showed `deepLink(URL)` as part of `NavigationIntent`. That is no longer the model. Deep links live in a separate planning layer.

---

## Coordinators are policy objects

`Coordinator` is the layer between SwiftUI intent and command execution when routing policy needs to intervene.

```swift
@Observable
@MainActor
final class AppCoordinator: Coordinator {
    typealias RouteType = AppRoute
    typealias Destination = AppDestinationView

    let store = NavigationStore<AppRoute>()
    var isAuthenticated = false

    func handle(_ intent: NavigationIntent<AppRoute>) {
        switch intent {
        case .go(.settings) where !isAuthenticated:
            _ = store.execute(.replace([.login]))
        default:
            store.send(intent)
        }
    }

    @ViewBuilder
    func destination(for route: AppRoute) -> AppDestinationView {
        AppDestinationView(route: route)
    }
}
```

Use this layer when:

- routing needs authentication or app policy checks first
- an app shell composes multiple navigation authorities
- view code should not know about routing policy

You connect it through `CoordinatorHost` or `CoordinatorSplitHost`.

```swift
struct RootView: View {
    @State private var coordinator = AppCoordinator()

    var body: some View {
        CoordinatorHost(coordinator: coordinator) {
            ContentView()
        }
    }
}
```

`FlowCoordinator` and `TabCoordinator` are also part of the public surface, but they complement `NavigationStore`; they do not replace it.

---

## Modal routing is a separate surface

One of the biggest runtime clarifications in InnoRouter 3.0 is that modal routing is not mixed into stack routing.

`sheet` and `fullScreenCover` live behind `ModalStore` and `ModalHost`.

```swift
@Routable
enum AppModalRoute {
    case profile
    case onboarding
}

struct ShellView: View {
    @State private var modalStore = ModalStore<AppModalRoute>()

    var body: some View {
        ModalHost(store: modalStore) { route in
            switch route {
            case .profile:
                ProfileView()
            case .onboarding:
                OnboardingView()
            }
        } content: {
            HomeView()
        }
    }
}
```

Views send presentation intent through `@EnvironmentModalIntent`.

```swift
struct HomeView: View {
    @EnvironmentModalIntent(AppModalRoute.self) private var modalIntent

    var body: some View {
        VStack {
            Button("Profile") {
                modalIntent.send(.present(.profile, style: .sheet))
            }

            Button("Dismiss") {
                modalIntent.send(.dismiss)
            }
        }
    }
}
```

This separation matters because:

- stack path and modal queue stay independent
- modal lifecycle can be observed explicitly
- modal routing intentionally does not expose middleware the way stack routing does

---

## Deep links are plans, not hidden side effects

Deep-link handling is one of the clearest examples of the runtime shift in 3.0.

1. `DeepLinkMatcher` maps a URL to a route
2. `DeepLinkPipeline` applies scheme, host, and authentication policy
3. the result becomes `.plan`, `.pending`, `.rejected`, or `.unhandled`
4. the app explicitly decides when to execute the resulting plan

### Matcher

```swift
let matcher = DeepLinkMatcher<ProductRoute> {
    DeepLinkMapping("/products") { _ in .list }
    DeepLinkMapping("/products/:id") { params in
        params.firstValue(forName: "id").map { .detail(id: $0) }
    }
}
```

`DeepLinkMatcher` can also surface structural diagnostics:

- duplicate patterns
- wildcard shadowing
- parameter shadowing

That helps catch authoring mistakes without changing declaration-order precedence.

### Pipeline

```swift
let pipeline = DeepLinkPipeline<ProductRoute>(
    allowedSchemes: ["myapp", "https"],
    allowedHosts: ["myapp.com"],
    resolve: { matcher.match($0) },
    authenticationPolicy: .required(
        shouldRequireAuthentication: { route in
            if case .detail = route { return true }
            return false
        },
        isAuthenticated: { authManager.isLoggedIn }
    ),
    plan: { route in NavigationPlan(commands: [.push(route)]) }
)
```

The important change is this: URLs do not immediately mutate navigation state.

- `.plan` means "safe to execute now"
- `.pending` means "hold until auth succeeds"
- `.rejected` means scheme/host policy blocked it
- `.unhandled` means no mapping resolved the URL

A typical SwiftUI integration looks like this:

```swift
.onOpenURL { url in
    switch pipeline.decide(for: url) {
    case .plan(let plan):
        _ = store.executeBatch(plan.commands)

    case .pending(let pending):
        self.pendingDeepLink = pending

    case .rejected(let reason):
        logger.error("Rejected deep link: \(String(describing: reason))")

    case .unhandled(let url):
        logger.error("Unhandled deep link: \(url.absoluteString)")
    }
}
```

That makes "resume after login" a first-class model through `PendingDeepLink`, not an ad hoc branch scattered across the app.

---

## Middleware is the cross-cutting policy layer

InnoRouter middleware is more than a callback hook. It sits on the command boundary.

### What it can do

- rewrite commands before execution
- block execution with typed cancellation
- observe and fold results after execution

### What it cannot do

- mutate store state directly

For example, this blocks duplicate pushes.

```swift
store.addMiddleware(
    AnyNavigationMiddleware(
        willExecute: { command, state in
            if case .push(let next) = command, state.path.last == next {
                return .cancel(.conditionFailed)
            }
            return .proceed(command)
        }
    ),
    debugName: "PreventDuplicatePush"
)
```

That is the right place for auth checks, analytics, and preconditions. Views remain focused on intent, not policy.

---

## Use effect modules at the app boundary

If app-shell or coordinator code wants an explicit execution facade instead of talking to a store directly, InnoRouter ships effect modules for that boundary.

### `InnoRouterNavigationEffects`

This module wraps navigation execution in a small synchronous `@MainActor` API.

- `execute(_:)`
- `execute(_ commands:)`
- `executeTransaction(_:)`
- `executeGuarded(_:, prepare:)`

That keeps app-boundary code focused on execution decisions instead of store internals.

### `InnoRouterDeepLinkEffects`

This module combines deep-link planning with execution helpers.

- execute a deep-link plan
- receive typed effect outcomes
- resume pending deep links after authentication

It is the boundary where deep-link planning turns into app policy and actual execution.

---

## Macros are now the preferred public style

Current public examples use `@Routable` as the default style instead of hand-written `Route` conformance.

```swift
import InnoRouter
import InnoRouterMacros

@Routable
enum HomeRoute {
    case list
    case detail(id: String)
    case settings
}
```

`@Routable` removes the manual `Route` conformance declaration for route enums.

`@CasePathable` is the companion macro when you want lightweight case-path extraction and composition for enum hierarchies. It is not required in every example, but it belongs in the current public API story.

---

## Which surface should you use?

| Situation | Recommended surface |
|---|---|
| single stack-based app | `NavigationStore` + `NavigationHost` |
| policy must run before navigation | `CoordinatorHost` + `Coordinator.handle(_:)` |
| iPad/macOS detail stack | `NavigationSplitHost` or `CoordinatorSplitHost` |
| `sheet` / `fullScreenCover` routing | `ModalStore` + `ModalHost` |
| URL entry and auth resumption | `DeepLinkMatcher` + `DeepLinkPipeline` |
| explicit boundary execution facade | `InnoRouterNavigationEffects`, `InnoRouterDeepLinkEffects` |

That table captures the design intent well: InnoRouter is not about pushing everything into one giant coordinator. It is about separating navigation state, policy, modal routing, and deep-link execution into typed boundaries.

---

## Installation and requirements

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", from: "3.0.0")
]
```

Current requirements:

- iOS 18+
- macOS 15+
- tvOS 18+
- watchOS 11+
- Swift 6.2+

The runtime surface assumes Swift 6.2 and strict concurrency, which is reflected in `Sendable`, `@MainActor`, and typed result modeling across the package.

---

## Closing thoughts

The shortest accurate description of InnoRouter today is this:

> InnoRouter is not just a SwiftUI navigation abstraction. It is a typed navigation runtime that organizes app boundaries around route state and command execution.

That pays off in a few concrete ways.

1. navigation state becomes explicit through `RouteStack`
2. command, batch, and transaction semantics stay visible and testable
3. stack, modal, and deep-link authority remain separate
4. policy moves out of views and into coordinators, middleware, and effect boundaries

If navigation keeps spreading through your SwiftUI codebase as view-local behavior, InnoRouter gives you a stronger model: treat it as state, commands, and explicit execution boundaries.

## References

- [InnoRouter GitHub Repository](https://github.com/InnoSquadCorp/InnoRouter)
- [Latest InnoRouter DocC](https://innosquadcorp.github.io/InnoRouter/latest/)
- [SwiftUI NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)
