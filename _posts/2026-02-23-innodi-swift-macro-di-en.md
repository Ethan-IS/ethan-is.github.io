---
title: "InnoDI Best Practices: Using Swift Macro DI to Preserve App Structure"
date: 2026-02-23 00:00:00 +0900
translation_key: innodi-swift-macro-di
lang: en
description: "Why InnoDI is useful, how to place Swift macro-based dependency injection at composition roots and feature boundaries, and how InnoSample applies those best practices."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, dependency-injection, macro, di, clean-architecture, swiftui, tuist]
author: ethan
toc: true
comments: true
---

## Why dependency injection gets hard in real apps

Dependency injection starts simple in an iOS app. You pass an `APIClient`, a `Repository`, or a `UseCase` through an initializer. The trouble starts when the app grows.

- Temporary factories appear around screens.
- Global singletons become harder to remove.
- Preview and test wiring drift away from the real app graph.
- Features start constructing implementation details from other features.
- Code review no longer shows who owns each dependency or how long it lives.

[`InnoDI`](https://github.com/InnoSquadCorp/InnoDI) does not hide this problem behind a runtime container. It asks you to declare the dependency graph with Swift macros, then catches structural drift through compile-time and build-time validation.

The appeal is straightforward: **wiring remains visible, graph ownership is reviewable, and failures move earlier than runtime.**

## The boundary InnoDI should own

InnoDI is not most valuable as a global place to pull dependencies from. It is most valuable at construction boundaries.

InnoDI should own:

- app composition roots
- layer and feature container wiring
- shared/input scopes
- parent-child container ownership
- graph validation and DAG inspection
- SwiftUI root-boundary wiring

InnoDI should not own:

- screen state transitions
- navigation stacks
- network retry or session lifecycle
- business rules
- per-action runtime overrides

In other words, InnoDI owns construction-time structure. Runtime state belongs in [`InnoFlow`]({% post_url 2026-02-23-innoflow-unidirectional-architecture-en %}), navigation belongs in [`InnoRouter`]({% post_url 2026-02-23-innorouter-swiftui-navigation-en %}), and network execution policy belongs in [`InnoNetwork`]({% post_url 2026-02-23-innonetwork-type-safe-networking-en %}).

## Installation and the smallest useful shape

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", from: "4.3.0")
]
```

Add `InnoDISwiftUI` when a target needs the SwiftUI root helpers.

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

A minimal container looks like this:

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

The important point is that `@Provide` is a declaration, not a dynamic registration. The container surface says what enters the graph and what the graph provides.

## Best practice 1. Start at the composition root

The top-level app container should connect infrastructure and major composition boundaries, not every detail in the app. [`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)'s `AppContainer` is the clearest example.

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

`AppContainer` does not need to know the internals of `RemoteContainer`, `DataContainer`, `DomainContainer`, or each leaf feature. It connects the next composition boundary and stops there.

That is the first rule: **a parent container should not build the whole app; it should connect the next owned boundary.**

## Best practice 2. Prefer small boundary containers over one giant graph

One huge app-wide DI graph often becomes a service locator with a nicer name. InnoDI works better when each architectural boundary gets a small container.

InnoSample's `LayerContainer` connects `Remote -> Data -> Domain` and exposes only the use case surface needed by features.

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

The benefits are practical:

- `App` does not know remote/data/domain wiring details.
- `Features` do not know repository implementations.
- `Domain` does not know remote models or network clients.
- Each graph stays small enough to review.

DI becomes a way to preserve dependency direction, not just a way to construct objects.

## Best practice 3. Use `@SubContainer` for feature ownership

InnoDI 4.x is especially useful when parent-child container ownership should be explicit.

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

The parent passes only the inputs a child needs. The child container owns its coordinator and view-root composition.

For SwiftUI apps, `featureRoot:` can connect the generated root scene as well. This removes repetitive startup wiring while keeping ownership visible.

## Best practice 4. Treat scopes as architecture language

DI scope is not just a performance setting. It says something about ownership and lifetime.

- `.input`: values supplied from outside the container, such as `baseURL`, environment, or feature input.
- `.shared`: infrastructure or identity-bearing objects owned by the boundary, such as network clients, repositories, or coordinators.
- computed properties: lightweight stateless values, such as use cases that can be rebuilt from shared repositories.

InnoSample keeps repositories shared and exposes concrete stateless use cases from `DomainContainer`. That removes unnecessary abstraction while keeping features away from repository implementations.

## What you gain by adopting it

Used this way, InnoDI gives you:

- visible construction ownership
- fewer feature-to-feature implementation leaks
- reviewable dependency graphs
- macro and build validation for wiring mistakes
- clearer SwiftUI root and preview/test entry points
- a consistent place to attach new features

The team benefit is large. When wiring is implicit, new features tend to construct whatever is closest. InnoDI makes the intended boundary harder to ignore.

## When not to use it

Macro DI is not necessary for every app.

- Small prototypes with only a few endpoints
- Apps where runtime plugin registration is the main requirement
- Legacy apps that intentionally keep global singletons
- Experiments where speed matters more than graph validation

In those cases, simple factories or runtime dependency tools may be a better fit.

InnoDI becomes attractive when the app has multiple features, layers, and platform targets. The more expensive it is to preserve construction boundaries, the more valuable InnoDI becomes.

## How InnoSample uses it

[`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice-en %}) does not scatter InnoDI throughout the app.

- `AppContainer` owns app-root composition.
- `LayerContainer` owns `Remote/Data/Domain` composition.
- `FeatureContainer` owns leaf-feature coordinator composition.
- Feature logic receives dependencies and does not need to know the DI framework.

That is the strongest way to use InnoDI: **treat DI as architecture notation, not convenience access.** Then InnoDI becomes a tool for keeping app structure intact over time.
