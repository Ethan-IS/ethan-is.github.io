---
title: "InnoDI: A Type-Safe Dependency Injection Library Built with Swift Macros"
date: 2026-02-23 00:00:00 +0900
lang: en
translation_key: innodi-swift-macro-di
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, dependency-injection, macro, di, clean-architecture]
author: ethan
toc: true
comments: true
---

## Introduction

When I first introduced `InnoDI`, the framing was mostly "a Swift Macro library that removes DI boilerplate." That description is still directionally true, but it no longer captures the real public surface of `3.0.1`.

Today, the core story is **static dependency graph and scope validation**.

In other words, `InnoDI` is not a runtime service locator. It is a framework for declaring DI wiring explicitly and rejecting invalid graphs as early as possible at compile time and build time.

This post revisits `InnoDI` from that perspective.

## Why revisit it now

The `3.0.x` line is not mainly about new macro syntax. The bigger shift is that the framework makes its validation boundaries much more explicit:

- strict name-based resolution
- declaration-order enforcement
- `concrete: true` opt-in
- `asyncFactory` scope restrictions
- custom `init` rejection for `@DIContainer` types
- build-stage DAG validation and plugin-based enforcement

So if we describe `InnoDI` only as "a macro that generates init code," we miss the part that matters most in a larger codebase: **which wiring patterns are allowed, and which are rejected**.

## What InnoDI owns and what it does not

The README describes `InnoDI` as a **static dependency graph and scope validation** framework.

What `InnoDI` owns:

- dependency wiring declared with `@DIContainer` and `@Provide`
- lifecycle modeling through `DIScope`
- compile-time and build-time validation
- DAG rendering and validation artifacts

What `InnoDI` does not own:

- runtime state transitions
- navigation policy
- network session or transport lifecycle
- container resolution as a runtime state machine

Within the InnoSquad stack, state transitions belong in `InnoFlow`, navigation belongs in `InnoRouter`, and transport/session lifecycle belongs in `InnoNetwork`. `InnoDI` stays focused on **static wiring** across those layers.

## Core API

## Install

For `InnoDI 3.0.1`, the package setup starts here.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", from: "3.0.1")
]
```

Then add the product to your target.

```swift
.target(
    name: "YourApp",
    dependencies: ["InnoDI"]
)
```

### `@DIContainer`

Declare a struct as a DI container.

```swift
@DIContainer(validate: true, root: false, validateDAG: true, mainActor: false)
struct AppContainer {}
```

Two things matter here:

- the macro generates container initializers and accessors
- the same declaration is also validation input

There is an important constraint as well: a type annotated with `@DIContainer` cannot define its own custom `init`. That rule is enforced in the type body, same-file extensions, and cross-file extensions during build validation.

### `@Provide`

Declare a dependency.

```swift
@Provide(
    _ scope: DIScope = .shared,
    _ type: Type.self? = nil,
    with: [KeyPath] = [],
    factory: Any? = nil,
    asyncFactory: Any? = nil,
    concrete: Bool = false
)
```

There are three main styles:

- receive external values with `.input`
- describe construction with `factory` or `asyncFactory`
- use `Type.self` + `with:` for AutoWiring

### `DIScope`

In `InnoDI`, scope is not just a convenience feature. It is directly tied to validation rules.

| Scope | Meaning | Notes |
|------|------|------|
| `.input` | must be injected when the container is created | no factory allowed |
| `.shared` | created once and cached for the container lifetime | order rules apply |
| `.transient` | creates a fresh instance on each access | not cached |

## A minimal example

This is the shape most users start with.

```swift
import InnoDI

protocol APIClientProtocol {
    func fetch() async throws -> Data
}

struct APIClient: APIClientProtocol {
    let baseURL: String

    func fetch() async throws -> Data {
        Data()
    }
}

@DIContainer
struct AppContainer {
    @Provide(.input)
    var baseURL: String

    @Provide(.shared, APIClient.self, with: [\.baseURL])
    var apiClient: any APIClientProtocol
}

let container = AppContainer(baseURL: "https://api.example.com")
let client = container.apiClient
```

The most important part is not the generated code. It is the **strict matching rule**: the key paths listed in `with:` must match the initializer parameter names of the concrete type exactly.

## AutoWiring and strict matching

One of the defining characteristics of `InnoDI 3.x` is that **name-based resolution is intentionally strict**.

```swift
@DIContainer
struct AppContainer {
    @Provide(.input)
    var config: AppConfig

    @Provide(.input)
    var logger: Logger

    @Provide(.shared, APIClient.self, with: [\.config, \.logger])
    var apiClient: any APIClientProtocol
}
```

Conceptually, this expects something like `APIClient(config:logger:)`. If the names do not match, the macro does not try to guess.

That can feel rigid, but it also means the declaration and the generated wiring follow the exact same rule. Humans and tools read the graph the same way.

When names do not line up, a factory closure is the better choice.

```swift
@Provide(.shared, factory: { (config: AppConfig) in
    APIClient(configuration: config, timeout: 30)
})
var apiClient: any APIClientProtocol
```

## Declaration order and scope rules

According to `PolicyBoundaries.md`, declaration order is part of the validation contract.

- `.input` members are always available
- sync `.shared` members can reference inputs and **earlier** sync shared members
- async `.shared` members can reference inputs, all sync shared members, and **earlier** async shared members
- `.transient` members may reference any container member, but matching is still strict

That means member order is not just style. It defines **dependency availability**.

## `concrete: true` and protocol-first design

`InnoDI` is protocol-first by default. Exposing a concrete shared or transient dependency requires explicit opt-in.

```swift
@DIContainer
struct AppContainer {
    @Provide(.input)
    var apiClient: any APIClientProtocol

    @Provide(.transient, factory: { (apiClient: any APIClientProtocol) in
        HomeViewModel(apiClient: apiClient)
    }, concrete: true)
    var homeViewModel: HomeViewModel
}
```

This is a small but useful rule. It makes concrete exposure deliberate instead of accidental and nudges the codebase toward dependency inversion.

## Async factories and their limits

Use `asyncFactory` when construction is asynchronous.

```swift
@DIContainer
struct AppContainer {
    @Provide(.input)
    var config: AppConfig

    @Provide(.shared, asyncFactory: { (config: AppConfig) async throws in
        try await DatabaseClient.connect(config: config)
    })
    var databaseClient: any DatabaseClientProtocol
}
```

Again, the framework is explicit about what it allows:

- `factory` and `asyncFactory` cannot be used together
- `.input` cannot use `asyncFactory`
- `asyncFactory` must actually be an `async` closure

This is a good example of the broader `3.x` direction: not just "we support it," but "we define the boundary precisely."

## Validation layers

This is where `InnoDI` becomes much more than a macro convenience layer.

### 1. Local container validation

At macro expansion time, `InnoDI` validates:

- unknown scopes
- missing factories
- invalid `.input` factory configuration
- strict name-based resolution
- declaration-order violations
- missing `concrete: true`
- local cycles and unknown dependencies
- async factory validity
- same-file `init` conflicts

### 2. Build-stage validation

Build validation scans package sources more broadly and extends the rule set to **cross-file extension `init` conflicts**.

So macro success is not the whole story. The build phase tightens the contract further.

### 3. Global DAG validation

Graph-wide cycles and ambiguity are validated through the CLI or the build plugin.

```bash
swift run InnoDI-DependencyGraph --root . --validate-dag
```

You can also opt a container out of DAG validation:

```swift
@DIContainer(validateDAG: false)
struct PreviewContainer {
    @Provide(.input)
    var mockAPIClient: any APIClientProtocol
}
```

That is best used for preview or test containers that you do not want included in the global graph contract.

## `PolicyBoundaries` and custom `init` restrictions

To understand `3.0.x`, the README is not enough. `PolicyBoundaries.md` matters because it explains the framework's determinism choices.

Key points:

- cross-file extension targets try semantic resolution first
- ambiguous or unsupported cases are not guessed
- generic argument extensions and constrained `where` extensions are excluded from the build-stage custom `init` rule
- nested paths such as `Outer.Container` are supported

This is a deliberate tradeoff. `InnoDI` prefers **deterministic validation** over speculative matching.

## Testing and override strategy

The main testing story is not a separate runtime override registry. It is the generated initializer with override parameters.

```swift
let container = AppContainer(
    baseURL: "https://test.example.com",
    apiClient: MockAPIClient()
)
```

That has two practical benefits:

- the difference between production and test wiring is visible directly in the initializer
- mock replacement works without a runtime registry or container mutation step

## CLI, plugin, and artifacts

In a team setting, the most important part is often not the macro diagnostic itself but the **plugin and artifacts**.

Attach `InnoDIDAGValidationPlugin` to a target if you want DAG problems to fail the build. During validation, the framework produces artifacts such as:

- `result.json`
- `validation-metrics.json`
- `validation-summary.md`
- `dag-validation-stamp.txt`
- `dag-validation-metrics.json`
- `dag-validation-summary.md`

In CI, those summaries and metrics are usually more useful than raw stderr alone.

## When to use which option

| Situation | Recommended choice |
|------|------|
| external values at app startup | `.input` |
| services reused for one container lifetime | `.shared` |
| ViewModels or adapters that need fresh instances | `.transient` with `concrete: true` when needed |
| initializer parameter names match member names | `Type.self` + `with:` |
| names differ or custom mapping is needed | `factory` |
| async construction is required | `asyncFactory` |
| preview or test-only containers should stay out of global DAG checks | `validateDAG: false` |

## Closing thoughts

If I had to summarize `InnoDI 3.0.1` in one sentence, I would no longer call it just "macro-based DI." A better description is **a DI wiring framework with explicit static graph rules**.

Reducing boilerplate is still useful. But in a larger codebase, the real value is that wiring failures show up during the build instead of surfacing later as runtime surprises.

If you want to go deeper, this reading order works best:

1. [README](https://github.com/InnoSquadCorp/InnoDI)
2. [`Validation.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/Validation.md)
3. [`PolicyBoundaries.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/PolicyBoundaries.md)
4. [`ModuleWideInitDetection.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/ModuleWideInitDetection.md)
