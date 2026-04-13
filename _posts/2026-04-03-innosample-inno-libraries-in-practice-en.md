---
title: "How InnoSample uses InnoDI, InnoFlow, InnoRouter, and InnoNetwork in a real app architecture"
date: 2026-04-03 00:00:00 +0900
translation_key: innosample-inno-libraries-in-practice
lang: en
description: "A practical guide to how InnoSample wires InnoDI, InnoFlow, InnoRouter, and InnoNetwork together inside a modular Swift iOS app architecture."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, architecture, modularization, dependency-injection, navigation, networking, tuist, clean-architecture, swiftui]
author: ethan
toc: true
comments: true
---

## Why this post exists

There are already separate posts for [`InnoDI`](https://github.com/InnoSquadCorp/InnoDI), [`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow), [`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter), and [`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork). But in a real Swift iOS app architecture, the harder problem is not learning each library in isolation. The harder problem is deciding **which boundary each library should own**.

That is what [`InnoSample`](https://github.com/InnoSquadCorp/InnoSample) is good at showing.

`InnoSample` is not mainly a feature showcase. It is a baseline scaffold that fixes the parts that tend to drift early in a project:

- module boundaries
- DI wiring
- navigation ownership
- network boundaries

This post uses the actual `InnoSample` codebase to answer four practical questions.

- Where should `InnoDI` live?
- Which layer should own `InnoFlow` reducers?
- How should `InnoRouter` handle cross-feature navigation?
- How far should `InnoNetwork` be exposed in app code?

The detailed public surface of each library is already covered in separate posts. This article is the integration guide: how the four libraries work **together** inside one modular app.

- `InnoDI`: [InnoDI: A Type-Safe Dependency Injection Library Built with Swift Macros]({% post_url 2026-02-23-innodi-swift-macro-di-en %})
- `InnoFlow`: [InnoFlow: A unidirectional architecture framework for SwiftUI]({% post_url 2026-02-23-innoflow-unidirectional-architecture-en %})
- `InnoRouter`: [InnoRouter: A type-safe navigation framework for SwiftUI]({% post_url 2026-02-23-innorouter-swiftui-navigation-en %})
- `InnoNetwork`: [InnoNetwork: A type-safe networking framework for Swift Concurrency]({% post_url 2026-02-23-innonetwork-type-safe-networking-en %})
- Modularity background: [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity-en %})
- Layering background: [Clean Architecture]({% post_url 2024-05-13-clean-architecture-en %})

## Start with the dependency direction

`InnoSample` is organized around this dependency flow.

- `Feature -> Domain`
- `Data -> Domain`
- `Remote -> Data + CoreNetwork`
- `Layers -> Domain + Data + Remote`
- `Features -> Domain + Feature`
- `App -> CoreNetwork + Layers + Features + ThirdParty`

Two things matter here.

First, `CoreNetwork`, `Layers`, and `Features` are not just implementation modules. They are mostly **composition and boundary modules**.

Second, runtime movement between features is allowed, but compile-time dependency cycles are not. A flow like `PeopleFeature -> SettingsFeature` may happen at runtime, but `PeopleFeature` does not directly import `SettingsFeature`.

That separation between runtime flow and compile-time dependency is one of the most useful things the sample teaches.

## The package versions used by the sample

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)'s `Tuist/Package.swift` pins these versions:

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", exact: "3.0.1"),
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow", exact: "3.0.2"),
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", exact: "3.1.0"),
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", exact: "3.0.0"),
]
```

The important point is not only that the packages are added. The important point is that `InnoSample` does not let every feature consume these libraries however it wants. It distributes them through the `App -> Layers -> Features` structure on purpose.

So if someone lands on this page searching for phrases like "InnoDI tutorial", "InnoFlow architecture", "InnoRouter SwiftUI example", or "InnoNetwork usage", the right framing is this:

> This is not a library-by-library tutorial. It is a practical guide to how those libraries should be placed inside a modular Swift app architecture.

## 1. Use InnoDI at composition roots and container boundaries

The first file to look at is `AppContainer`. It is the clearest example of where [`InnoDI`](https://github.com/InnoSquadCorp/InnoDI) should live in a real app.

```swift
@MainActor
@DIContainer(root: true)
struct AppContainer {
    @Provide(.input)
    var baseURL: URL

    @Provide(.shared, factory: {
        AnalyticsClient(apiKey: "innosample-demo-key")
    }, concrete: true)
    var analyticsClient: AnalyticsClient

    @Provide(.shared, factory: { (baseURL: URL) in
        NetworkFactory.makeTransport(baseURL: baseURL)
    }, concrete: true)
    var networkTransport: NetworkTransport

    @Provide(.shared, factory: { (networkTransport: NetworkTransport) in
        LayerContainer.make(networkTransport: networkTransport)
    }, concrete: true)
    var layerContainer: LayerContainer

    @Provide(.shared, factory: { (layerContainer: LayerContainer) in
        FeatureContainer.make(useCases: layerContainer.featureUseCases)
    }, concrete: true)
    var featureContainer: FeatureContainer
}
```

This is a good example of what `InnoDI` should be doing in a real codebase.

It is not being used as a runtime service locator. It is being used where wiring should be declared explicitly.

The pattern is simple:

- external values like `baseURL` enter as `.input`
- long-lived infrastructure becomes `.shared`
- the app composes in one direction only: `NetworkTransport -> LayerContainer -> FeatureContainer`
- the root container does not need to know the internal steps of `RemoteContainer`, `DataContainer`, and `DomainContainer`

That is the right mental model for `InnoDI`: not "object creation convenience," but **explicit dependency graph declaration**.

If you want the detailed rules around scopes, declaration order, `concrete: true`, and validation, the dedicated [InnoDI post]({% post_url 2026-02-23-innodi-swift-macro-di-en %}) goes deeper. This article stays focused on where that wiring belongs architecturally.

## 2. Layers use InnoDI internally, but expose only a narrow surface

`LayerContainer` connects `Remote -> Data -> Domain`.

```swift
public struct LayerContainer {
    private let remoteContainer: RemoteContainer
    private let dataContainer: DataContainer
    private let domainContainer: DomainContainer

    public init(networkTransport: NetworkTransport) {
        self.remoteContainer = RemoteContainer(networkTransport: networkTransport)
        self.dataContainer = DataContainer(remoteContainer: remoteContainer)
        self.domainContainer = DomainContainer(dataContainer: dataContainer)
    }

    public var featureUseCases: any FeatureUseCaseContaining {
        domainContainer
    }
}
```

This matters because `App` and `Features` do not need to know the concrete `DomainContainer` type. They only receive `featureUseCases`.

The underlying composition is pushed down one level at a time.

### `RemoteContainer`

```swift
@DIContainer
public struct RemoteContainer {
    @Provide(.input)
    public var networkTransport: NetworkTransport

    @Provide(.shared, factory: { (networkTransport: NetworkTransport) in
        UserRemoteFactory.make(networkTransport: networkTransport)
    })
    public var userRemoteDataSource: any UserRemoteDataSourceProtocol
}
```

### `DataContainer`

```swift
@DIContainer
public struct DataContainer {
    @Provide(.input)
    public var remoteContainer: any RemoteDataSourceContaining

    @Provide(.shared, factory: { (remoteContainer: any RemoteDataSourceContaining) in
        UserRepositoryFactory.make(remoteContainer: remoteContainer)
    })
    public var userRepository: any UserRepositoryProtocol
}
```

### `DomainContainer`

```swift
@DIContainer
public struct DomainContainer: FeatureUseCaseContaining {
    @Provide(.input)
    public var dataContainer: any RepositoryContaining
}

extension DomainContainer {
    public var fetchPeopleUseCase: FetchPeopleUseCase {
        FetchPeopleUseCase(repository: dataContainer.userRepository)
    }
}
```

This gives the sample a few good properties.

- repositories remain protocol-based because they are real cross-layer contracts
- use cases stay lightweight concrete values because they are stateless wrappers
- features depend on use cases, not repositories
- the app root does not need to know remote/data/domain internals

That is a good default: not one giant graph, but **small containers at each boundary with a narrow outward surface**.

This is also consistent with the modularity-first approach discussed in [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity-en %}).

## 3. Keep InnoFlow inside Logic targets

In [`InnoSample`](https://github.com/InnoSquadCorp/InnoSample), [`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow) belongs inside feature `Logic` targets. `PeopleFeatureReducer` is a representative example.

```swift
@InnoFlow
struct PeopleFeatureReducer {
    struct Dependencies: Sendable {
        let loadPeople: @Sendable () async throws -> [UserSummary]
    }

    struct State: Equatable, Sendable, DefaultInitializable {
        var isLoading = false
        var hasLoaded = false
        var people: [UserSummary] = []
        var errorMessage: String?
        var selectedUser: UserSummary?
        var pendingOverviewRequest: PeopleOverviewRequest?
        var pendingSettingsRequest: PeopleSettingsRequest?
    }

    enum Action: Equatable, Sendable {
        case onAppear
        case refresh
        case peopleLoaded([UserSummary])
        case peopleFailed(String)
        case select(UserSummary)
        case showOverview
        case openSettings(OpenSettingsRequest)
    }
}
```

The important part is not just that the feature has `State` and `Action`. The more important part is that state also carries **one-shot intents** such as:

- `selectedUser`
- `pendingOverviewRequest`
- `pendingSettingsRequest`

Those are not `InnoRouter` commands. They are domain-friendly requests emitted by the reducer.

The async work also stays inside reducer-driven effects.

```swift
private func loadPeople() -> EffectTask<Action> {
    let loadPeople = dependencies.loadPeople

    return .run { send, _ in
        do {
            let people = try await loadPeople()
            await send(.peopleLoaded(people))
        } catch {
            let message = (error as? LocalizedError)?.errorDescription ?? error.localizedDescription
            await send(.peopleFailed(message))
        }
    }
    .cancellable("people-feature-load", cancelInFlight: true)
}
```

That gives the sample a clean split.

- reducers do not know `SwiftUI`
- reducers do not know `InnoRouter`
- reducers receive use case outputs back as actions
- tests can target state transitions instead of UI side effects

So the right way to think about `InnoFlow` here is not "screen control framework." It is a **feature logic boundary built around state transitions and effects**.

If you want the deeper runtime model, including `Reducer`, `Store`, `EffectTask`, and `PhaseMap`, read the dedicated [InnoFlow post]({% post_url 2026-02-23-innoflow-unidirectional-architecture-en %}). The architectural lesson in `InnoSample` is that reducer ownership stays inside `Logic`.

## 4. The model wraps the InnoFlow store, and the router owns screen movement

`PeopleFeatureModel` wraps the reducer store into a SwiftUI-friendly interface.

```swift
@MainActor
@Observable
public final class PeopleFeatureModel {
    private let store: Store<PeopleFeatureReducer>

    public init(loadPeople: @escaping @Sendable () async throws -> [UserSummary]) {
        self.store = Store(
            reducer: PeopleFeatureReducer(
                dependencies: .init(loadPeople: loadPeople)
            )
        )
    }

    public func loadIfNeeded() { store.send(.onAppear) }
    public func select(_ user: UserSummary) { store.send(.select(user)) }

    public func consumeSettingsRequest() -> OpenSettingsRequest? {
        let request = store.pendingSettingsRequest?.request
        guard let request else { return nil }
        store.send(.clearSettingsRequest)
        return request
    }
}
```

Then `PeopleFeatureCoordinator` owns the `InnoRouter` stores.

```swift
@MainActor
@Observable
public final class PeopleFeatureCoordinator {
    let navigationStore = NavigationStore<PeopleRoute>()
    let modalStore = ModalStore<PeopleModalRoute>()
    let model: PeopleFeatureModel

    func syncNavigationFromSelection() {
        guard let selectedUser = model.consumeSelectedUser() else { return }
        navigationStore.send(.resetTo([.detail(selectedUser)]))
    }

    func syncModalPresentation() {
        guard let users = model.consumeOverviewUsers() else { return }
        modalStore.send(.present(.overview(users), style: .sheet))
    }
}
```

This separation is important.

The reducer never calls `navigationStore.send(...)` directly. It finishes at business intent. The coordinator reads that intent and translates it into route commands.

That keeps the ownership clean:

- `Logic` owns state and effects
- `Model` owns SwiftUI-friendly projection
- `Router/Coordinator` owns navigation state

## 5. InnoRouter becomes even more valuable at the shell level

At the leaf level, route hosts use [`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter)'s `NavigationHost` and `ModalHost` in the expected way.

```swift
public struct PeopleFeatureRouteHost: View {
    let coordinator: PeopleFeatureCoordinator

    public var body: some View {
        ModalHost(store: coordinator.modalStore) { route in
            switch route {
            case .overview(let users):
                PeopleOverviewSheet(users: users) {
                    coordinator.modalStore.send(.dismiss)
                }
            }
        } content: {
            NavigationHost(store: coordinator.navigationStore) { route in
                switch route {
                case .detail(let user):
                    PeopleDetailScreen(user: user, onOpenSettings: coordinator.openSettings)
                }
            } root: {
                PeopleScreen(
                    model: coordinator.model,
                    onSelect: coordinator.select,
                    onShowOverview: coordinator.showOverview
                )
            }
        }
    }
}
```

But the more interesting file is `EntireTabCoordinator`.

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

This is the real architectural win.

`PeopleFeature` does not directly import `SettingsFeature`, and `SettingsFeature` does not directly own `PeopleFeature` navigation. Instead:

1. the reducer emits a one-shot cross-feature request
2. the feature coordinator consumes that request
3. the shell coordinator mediates the tab switch and sibling entry
4. the destination coordinator updates its own route state

That prevents compile-time dependency cycles while still allowing bidirectional runtime movement.

The dedicated [InnoRouter post]({% post_url 2026-02-23-innorouter-swiftui-navigation-en %}) explains the runtime surface itself in detail. What `InnoSample` adds is a strong example of **shell-level mediation** instead of direct sibling wiring.

## 6. Keep InnoNetwork behind CoreNetwork

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample) does not let [`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork) leak everywhere. It introduces a `CoreNetwork` boundary first.

`NetworkFactory` centralizes transport policy.

```swift
public enum NetworkFactory {
    public static func makeTransport(
        environment: NetworkEnvironment,
        session: URLSessionProtocol = URLSession.shared
    ) -> NetworkTransport {
        let defaults = makeDefaults(environment: environment)
        return NetworkTransport(
            client: makeClient(environment: environment, session: session),
            defaults: defaults
        )
    }

    static func makeDefaults(environment: NetworkEnvironment) -> APIDefaults {
        APIDefaults(
            environment: environment,
            logger: RequestLogger(),
            requestInterceptors: [
                NetworkMetadataInterceptor(environment: environment),
            ],
            responseInterceptors: [
                NetworkStatusInterceptor(),
            ]
        )
    }
}
```

And it exposes only `NetworkTransport` upward.

```swift
public actor NetworkTransport {
    public func send<Request: RequestDefinition>(_ request: Request) async throws -> Request.ResponseBody {
        do {
            return try await client.perform(executable:
                RequestAdapter(request: request, apiDefaults: defaults)
            )
        } catch let error as NetworkError {
            throw NetworkFailure(networkError: error)
        } catch {
            throw NetworkFailure.transport(SendableUnderlyingError(error), request: nil)
        }
    }
}
```

That means transport-specific concerns stay inside `CoreNetwork`:

- retry policy
- interceptors
- logging
- environment defaults
- request/response adaptation

`Remote` only defines requests and executes them.

Here is the full flow for loading people.

### Request definition

```swift
struct FetchUsersRequest: RequestDefinition {
    typealias ResponseBody = [UserRemoteModel]

    let featureName = "People"
    var path: String { "/users" }
    var headerPolicy: HeaderPolicy { .external }
}
```

### Remote data source

```swift
public actor JSONPlaceholderUserRemoteDataSource: UserRemoteDataSourceProtocol {
    private let transport: NetworkTransport

    public func fetchUsers() async throws -> [UserRemoteModel] {
        try await transport.send(FetchUsersRequest())
    }
}
```

### Repository

```swift
public struct DefaultUserRepository: UserRepositoryProtocol, Sendable {
    public func fetchUsers() async throws -> [UserSummary] {
        let users = try await remoteDataSource.fetchUsers()
        guard !users.isEmpty else {
            throw DomainError.emptyResponse("사용자")
        }
        return users.map(\.domainModel)
    }
}
```

### Use case

```swift
public struct FetchPeopleUseCase: Sendable {
    public func callAsFunction() async throws -> [UserSummary] {
        try await repository.fetchUsers()
    }
}
```

By the time the request reaches `Feature`, the app no longer knows anything about `InnoNetwork`.

That is exactly what you want in a real app. If transport policy changes later, most of the blast radius stays inside `CoreNetwork`.

The product-family view of `InnoNetwork`, including `NetworkConfiguration`, retry policy, downloads, and websockets, is covered in the dedicated [InnoNetwork post]({% post_url 2026-02-23-innonetwork-type-safe-networking-en %}). Here the important lesson is boundary placement.

## 7. The practical ownership model

`InnoSample` suggests a simple ownership table.

| Library | What it should own | What it should not own |
|---|---|---|
| `InnoDI` | container declarations, composition roots, layer wiring | screen logic, business state, service-locator-style runtime access |
| `InnoFlow` | feature state, actions, effects, one-shot intents | route stacks, SwiftUI view composition |
| `InnoRouter` | stack and modal state, coordinators, cross-feature mediation | business state machines, repository calls |
| `InnoNetwork` | request execution, retry, interceptors, logging, transport policy | domain modeling, feature logic, app-shell composition |

This matters because a codebase is not healthier just because it "uses all four libraries." It is healthier when each library stays inside its intended boundary.

## 8. Good defaults to copy into a new app

If you use `InnoSample` as the starting point for a real app, these are strong defaults:

1. Keep a single `AppContainer` as the top-level composition root.
2. Put `InnoNetwork` behind a dedicated `CoreNetwork` module.
3. Keep `Layers` as a composition-only boundary.
4. Physically split each feature into `Interface / Logic / UI / Router / Testing / Tests`.
5. Do not import `SwiftUI` or `InnoRouter` inside feature `Logic`.
6. Do not let sibling features navigate directly to each other.
7. Keep repositories protocol-based, and keep stateless use cases as concrete values by default.

Those defaults make it much easier to scale the architecture without mixing ownership boundaries too early.

## Closing

The most useful lesson in `InnoSample` is this:

> The Inno libraries work best when they are not mixed together in the same place.

In practice:

- `InnoDI` declares wiring
- `InnoFlow` drives state transitions
- `InnoRouter` mediates screen flow
- `InnoNetwork` encapsulates transport policy

If all four are visible inside one file, that is often a warning sign. If they appear in clearly separated places across `App -> Layers -> Features`, the architecture tends to stay understandable as the app grows.

For adjacent reading:

- Architecture background: [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity-en %}), [Clean Architecture]({% post_url 2024-05-13-clean-architecture-en %})
- Library-specific posts: [InnoDI]({% post_url 2026-02-23-innodi-swift-macro-di-en %}), [InnoFlow]({% post_url 2026-02-23-innoflow-unidirectional-architecture-en %}), [InnoRouter]({% post_url 2026-02-23-innorouter-swiftui-navigation-en %}), [InnoNetwork]({% post_url 2026-02-23-innonetwork-type-safe-networking-en %})

## Repository links

- [InnoSample GitHub repository](https://github.com/InnoSquadCorp/InnoSample)
- [InnoDI GitHub repository](https://github.com/InnoSquadCorp/InnoDI)
- [InnoFlow GitHub repository](https://github.com/InnoSquadCorp/InnoFlow)
- [InnoRouter GitHub repository](https://github.com/InnoSquadCorp/InnoRouter)
- [InnoNetwork GitHub repository](https://github.com/InnoSquadCorp/InnoNetwork)
