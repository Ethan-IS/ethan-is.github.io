---
title: "InnoSample 아키텍처로 보는 InnoDI, InnoFlow, InnoRouter, InnoNetwork 실전 사용법"
date: 2026-04-03 00:00:00 +0900
translation_key: innosample-inno-libraries-in-practice
lang: ko-KR
description: "InnoSample 코드 기준으로 InnoDI, InnoFlow, InnoRouter, InnoNetwork를 실제 iOS 앱 아키텍처에 어떻게 배치하고 연결해야 하는지 설명합니다."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, architecture, modularization, dependency-injection, navigation, networking, tuist, clean-architecture, swiftui]
author: ethan
toc: true
comments: true
---

## 들어가며

[`InnoDI`](https://github.com/InnoSquadCorp/InnoDI), [`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow), [`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter), [`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork)를 각각 따로 설명하는 글은 이미 많습니다. 그런데 실제 iOS 앱 아키텍처에서는 이 네 개를 "각자 잘 쓰는 것"보다 **어디까지를 누가 소유하는지**를 먼저 맞추는 일이 더 중요합니다.

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)은 그 경계를 보여주기 위한 샘플입니다. 이 저장소의 목적은 기능을 많이 넣는 것이 아니라, Inno 계열 앱에서 반복해서 쓰게 되는 **Swift 모듈 아키텍처, DI wiring, navigation ownership, network boundary**를 먼저 고정하는 데 있습니다.

이 글은 `InnoSample` 코드 기준으로 아래 질문에 답하는 방식으로 정리합니다.

- `InnoDI`는 어디서 써야 하나
- `InnoFlow`는 어떤 레이어에 둬야 하나
- `InnoRouter`는 feature 간 이동을 어떻게 맡아야 하나
- `InnoNetwork`는 앱 코드에 어디까지 드러나야 하나

개별 라이브러리의 public surface 자체는 이미 별도 글로 정리해 두었습니다. 이 글은 그 글들을 대체하는 글이 아니라, **실제 앱 구조 안에서 네 라이브러리를 함께 배치하는 방법**에 집중합니다.

- `InnoDI` 자체 설명: [InnoDI: Swift Macro 기반 타입 안전 의존성 주입 라이브러리]({% post_url 2026-02-23-innodi-swift-macro-di %})
- `InnoFlow` 자체 설명: [InnoFlow: SwiftUI를 위한 단방향 아키텍처 프레임워크]({% post_url 2026-02-23-innoflow-unidirectional-architecture %})
- `InnoRouter` 자체 설명: [InnoRouter: SwiftUI를 위한 타입 안전 내비게이션 프레임워크]({% post_url 2026-02-23-innorouter-swiftui-navigation %})
- `InnoNetwork` 자체 설명: [InnoNetwork: Swift Concurrency를 위한 타입 안전 네트워킹 프레임워크]({% post_url 2026-02-23-innonetwork-type-safe-networking %})
- 모듈 분리 배경: [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity %})
- 계층 분리 배경: [Clean Architecture]({% post_url 2024-05-13-clean-architecture %})

## 먼저 구조부터 봐야 합니다

`InnoSample`의 핵심 의존 방향은 아래처럼 잡혀 있습니다.

- `Feature -> Domain`
- `Data -> Domain`
- `Remote -> Data + CoreNetwork`
- `Layers -> Domain + Data + Remote`
- `Features -> Domain + Feature`
- `App -> CoreNetwork + Layers + Features + ThirdParty`

이 구조에서 중요한 점은 두 가지입니다.

첫째, `CoreNetwork`와 `Layers`, `Features`가 전부 "구현 레이어"가 아니라 **조립과 경계 정리를 위한 레이어**라는 점입니다.

둘째, `PeopleFeature -> SettingsFeature` 같은 런타임 이동은 허용하지만, `PeopleFeature`가 `SettingsFeature`를 직접 import 하지는 않는다는 점입니다. 즉 **런타임 흐름과 컴파일 의존을 분리**합니다.

## 샘플이 실제로 쓰는 패키지 버전

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)의 `Tuist/Package.swift` 기준으로 샘플은 아래 버전을 고정해서 사용합니다.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", exact: "3.0.1"),
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow", exact: "3.0.2"),
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", exact: "3.1.0"),
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", exact: "3.0.0"),
]
```

여기서 중요한 건 단순히 "패키지를 추가했다"가 아닙니다. `InnoSample`은 Inno 라이브러리를 각 feature가 제각각 참조하게 두지 않고, **App -> Layers -> Features** 구조 안에서 역할별로 분배해서 사용합니다.

즉 검색 키워드로 표현하면, 이 글은 "`InnoDI 사용법`", "`InnoFlow 사용법`", "`InnoRouter 사용법`", "`InnoNetwork 사용법`"을 각각 설명하는 글이라기보다, **"InnoSample 기반 Swift iOS 모듈 아키텍처에서 네 라이브러리를 어떻게 함께 쓰는가"**에 답하는 글입니다.

## 1. InnoDI는 composition root와 container 경계에 둡니다

가장 먼저 봐야 할 코드는 `AppContainer`입니다. [`InnoDI`](https://github.com/InnoSquadCorp/InnoDI)가 실제 앱에서 어디에 위치해야 하는지가 가장 잘 드러나는 지점입니다.

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

이 코드를 보면 `InnoDI`는 서비스 로케이터처럼 앱 전체에서 아무 데서나 꺼내 쓰는 도구가 아닙니다. 대신 **조립을 선언하는 위치**에만 등장합니다.

이 샘플에서 `InnoDI` 사용 원칙은 아래처럼 읽으면 됩니다.

- `AppContainer`는 앱의 최상위 composition root다.
- `baseURL` 같은 외부 입력은 `.input`으로 넣는다.
- 오래 살아야 하는 인프라는 `.shared`로 둔다.
- 상위는 `NetworkTransport -> LayerContainer -> FeatureContainer` 순서로만 조립한다.
- 상위 컨테이너가 `RemoteContainer`, `DataContainer`, `DomainContainer` 내부 구조를 직접 알지 않게 만든다.

즉 `InnoDI`의 역할은 "객체 생성 편의"가 아니라 **의존 그래프를 코드로 고정하는 것**입니다.

`InnoDI`의 scope 규칙, declaration order, `concrete: true` 같은 세부 contract는 위에 링크한 [InnoDI 글]({% post_url 2026-02-23-innodi-swift-macro-di %})에서 더 자세히 볼 수 있습니다. 여기서는 그 기능을 어떤 아키텍처 경계에서 써야 하는지에 집중하면 충분합니다.

## 2. Layers는 InnoDI로 조립하지만 외부에는 얇은 surface만 노출합니다

`LayerContainer`는 `Remote -> Data -> Domain`을 연결하는 composition-only 모듈입니다.

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

여기서 좋은 점은 외부가 `DomainContainer` 구체 타입을 몰라도 된다는 것입니다. `App`과 `Features`는 `featureUseCases`만 받습니다.

세부 조립은 각 레이어 컨테이너로 더 내려갑니다.

### RemoteContainer

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

### DataContainer

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

### DomainContainer

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

이 패턴이 주는 이점은 분명합니다.

- repository는 진짜 레이어 경계 계약이므로 protocol로 유지합니다.
- use case는 stateless wrapper이므로 computed concrete value로 가볍게 조합합니다.
- feature는 repository를 모르고 use case만 받습니다.
- app root는 remote/data/domain 내부 wiring을 몰라도 됩니다.

즉 `InnoDI`는 전체 그래프를 크게 하나로 만드는 게 아니라, **경계마다 작은 container를 두고 surface를 줄이는 방식**으로 쓰는 편이 낫습니다.

이 지점은 모듈 경계를 먼저 세우고 wiring을 뒤에 얹는 방식이라, 기존에 정리한 [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity %})와도 결이 같습니다.

## 3. InnoFlow는 Logic 타깃 안에서만 비즈니스 상태를 바꾸게 둡니다

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)에서 [`InnoFlow`](https://github.com/InnoSquadCorp/InnoFlow)는 feature의 `Logic` 타깃에만 들어갑니다. 예를 들어 `PeopleFeatureReducer`는 이렇게 생겼습니다.

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

이 구조에서 눈여겨볼 점은 `State` 안에 단순 화면 상태만 있는 것이 아니라, **한 번 소비하고 버릴 intent**도 같이 들어 있다는 점입니다.

- `selectedUser`
- `pendingOverviewRequest`
- `pendingSettingsRequest`

이 값들은 `InnoRouter` 명령 자체가 아닙니다. 대신 reducer가 "이런 이동이 필요하다"는 **도메인 친화적인 요청**만 기록합니다.

실제 비동기 side effect도 reducer 내부에서 의존성을 통해 실행합니다.

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

이 방식의 장점은 아래와 같습니다.

- reducer는 `SwiftUI`를 모릅니다.
- reducer는 `InnoRouter`를 모릅니다.
- reducer는 use case 실행 결과를 action으로 다시 받습니다.
- 테스트는 navigation side effect가 아니라 state/action 변화를 기준으로 작성할 수 있습니다.

즉 `InnoFlow`는 "화면 제어 프레임워크"가 아니라, **feature 로직을 상태 전이와 effect로 고정하는 도구**로 쓰는 편이 맞습니다.

`Reducer`, `EffectTask`, `Store`, `PhaseMap` 같은 runtime surface 자체는 [InnoFlow 글]({% post_url 2026-02-23-innoflow-unidirectional-architecture %})에서 더 자세히 다뤘고, 여기서는 그 reducer를 왜 `Logic` 타깃 안에 가둬야 하는지가 더 중요합니다.

## 4. Model은 InnoFlow store를 감싸고, Router는 화면 전환만 맡습니다

`PeopleFeatureModel`은 reducer store를 감싸는 얇은 모델입니다.

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

여기서 model은 reducer의 상태를 SwiftUI 친화적인 public surface로 투영하고, one-shot 요청을 `consume...()` 형태로 꺼내 줍니다.

그 다음 `PeopleFeatureCoordinator`가 `InnoRouter` store를 소유합니다.

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

여기서 중요한 포인트는 reducer가 `navigationStore.send(...)`를 직접 호출하지 않는다는 것입니다.  
로직은 로직대로 끝내고, coordinator가 그 결과를 읽어서 route command로 번역합니다.

이 분리가 되면 아래 경계가 깨지지 않습니다.

- `Logic`은 상태와 effect를 소유
- `Model`은 SwiftUI-friendly projection을 소유
- `Router/Coordinator`는 navigation state를 소유

## 5. InnoRouter는 feature 내부보다 상위 중재에 더 중요합니다

개별 feature route host는 [`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter)의 `NavigationHost`, `ModalHost`로 화면을 붙입니다.

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

하지만 이 샘플에서 더 중요한 코드는 `EntireTabCoordinator`입니다.

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

이 구조가 좋은 이유는 `PeopleFeature`가 `SettingsFeature`를 직접 import 하지 않아도 되기 때문입니다.

즉 흐름은 이렇습니다.

1. `PeopleFeatureReducer`가 `pendingSettingsRequest`를 만든다.
2. `PeopleFeatureCoordinator`가 그 요청을 꺼낸다.
3. `EntireTabCoordinator`가 탭 전환과 sibling feature 진입을 중재한다.
4. `SettingsFeatureCoordinator`가 자신의 route stack을 갱신한다.

이 패턴을 쓰면 얻는 것이 많습니다.

- sibling feature 간 컴파일 의존 순환을 피할 수 있습니다.
- cross-feature 이동 정책을 상위 shell에 모을 수 있습니다.
- 딥링크, 인증, 탭 전환 같은 전역 navigation 판단을 한 곳에서 제어할 수 있습니다.

즉 `InnoRouter`는 "각 화면에서 push 하는 법"보다 **상위 coordinator가 어떤 이동 권한을 가지는지**를 정리할 때 더 빛납니다.

`NavigationStore`, `NavigationHost`, `ModalHost`, `NavigationCommand` 자체 설명은 [InnoRouter 글]({% post_url 2026-02-23-innorouter-swiftui-navigation %})을 보면 됩니다. `InnoSample`이 주는 실전 포인트는 그 surface를 leaf feature보다 **상위 중재 계층**에서 더 전략적으로 쓴다는 점입니다.

## 6. InnoNetwork는 앱 전역에서 직접 쓰지 말고 CoreNetwork 뒤로 숨깁니다

[`InnoSample`](https://github.com/InnoSquadCorp/InnoSample)은 [`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork)를 바로 `Feature`나 `Remote`에 퍼뜨리지 않습니다. 대신 `CoreNetwork`라는 boundary를 하나 둡니다.

`NetworkFactory`는 앱 공통 transport 정책을 모읍니다.

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

그리고 바깥으로는 `NetworkTransport`만 노출합니다.

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

이 패턴의 핵심은 `InnoNetwork`의 low-level surface를 앱 전체에 공개하지 않는 것입니다.

- retry policy
- interceptor
- logger
- environment
- request/response adaptation

이런 transport 정책은 `CoreNetwork`가 소유하고, `Remote`는 그냥 요청 정의와 실행만 맡습니다.

예를 들어 사용자 조회는 아래처럼 흘러갑니다.

### RequestDefinition

```swift
struct FetchUsersRequest: RequestDefinition {
    typealias ResponseBody = [UserRemoteModel]

    let featureName = "People"
    var path: String { "/users" }
    var headerPolicy: HeaderPolicy { .external }
}
```

### RemoteDataSource

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

### UseCase

```swift
public struct FetchPeopleUseCase: Sendable {
    public func callAsFunction() async throws -> [UserSummary] {
        try await repository.fetchUsers()
    }
}
```

이 흐름을 보면 `Feature`는 `InnoNetwork`를 모르고, `Domain`도 모릅니다.  
오직 `CoreNetwork`와 `Remote`만 transport 세부사항을 알고 있습니다.

이게 실제 서비스에서 중요합니다. 네트워크 라이브러리를 바꾸거나 interceptor 정책을 바꾸더라도, 수정 범위를 `CoreNetwork` 쪽으로 최대한 몰 수 있기 때문입니다.

`NetworkConfiguration`, `RetryPolicy`, `InnoNetworkDownload`, `InnoNetworkWebSocket` 같은 제품군 설명은 [InnoNetwork 글]({% post_url 2026-02-23-innonetwork-type-safe-networking %})을 참고하면 됩니다. 이 글에서는 그중 어떤 surface를 앱 아키텍처 어디에 두는지가 핵심입니다.

## 7. 결국 네 라이브러리는 이렇게 나눠 쓰는 편이 좋습니다

`InnoSample` 기준으로 각 라이브러리의 ownership을 한 줄씩 정리하면 이렇습니다.

| 라이브러리 | 소유해야 하는 것 | 소유하면 안 되는 것 |
| --- | --- | --- |
| `InnoDI` | container 선언, composition root, layer wiring | 화면 로직, 비즈니스 상태, 전역 service locator 사용 |
| `InnoFlow` | feature 상태, action, effect, one-shot intent | route stack 직접 제어, SwiftUI view 조립 |
| `InnoRouter` | stack/modal state, coordinator, cross-feature mediation | 비즈니스 상태 머신, repository 호출 |
| `InnoNetwork` | request execution, retry, interceptor, logger, transport policy | domain 모델, feature 로직, app-shell wiring |

이 표가 중요한 이유는 "라이브러리를 많이 쓴다"가 좋은 구조를 의미하지 않기 때문입니다.  
오히려 각 라이브러리를 **자기 경계 바깥으로 넘기지 않는 것**이 더 중요합니다.

## 8. 새 앱에서 그대로 가져가면 좋은 규칙

`InnoSample`을 실제 앱의 출발점으로 쓴다면, 저는 아래 규칙부터 고정하는 편을 권합니다.

1. `AppContainer` 하나만 최상위 composition root로 둡니다.
2. `CoreNetwork`를 따로 두고 `InnoNetwork` 세부 surface를 감춥니다.
3. `Layers`는 composition-only 모듈로 유지합니다.
4. feature는 `Interface / Logic / UI / Router / Testing / Tests`처럼 물리적으로 나눕니다.
5. `Logic` 타깃에서는 `SwiftUI`, `InnoRouter` import를 금지합니다.
6. feature 간 직접 이동은 금지하고, 상위 coordinator가 중재하게 합니다.
7. repository는 protocol, use case는 stateless concrete value를 기본값으로 둡니다.

이 정도만 지켜도, Inno 계열 라이브러리를 "도입한 것"이 아니라 **아키텍처 경계 안에 제대로 배치한 것**에 가까워집니다.

## 마무리

`InnoSample`이 보여주는 가장 중요한 메시지는 이것입니다.

> Inno 라이브러리는 각각 잘 쓰는 것보다, 서로의 책임이 섞이지 않게 쓰는 것이 더 중요합니다.

정리하면:

- `InnoDI`는 조립을 선언합니다.
- `InnoFlow`는 상태 전이를 관리합니다.
- `InnoRouter`는 화면 흐름과 feature 간 이동을 중재합니다.
- `InnoNetwork`는 transport policy를 캡슐화합니다.

이 네 개가 한 파일 안에서 다 보이면 구조가 잘못됐을 가능성이 높습니다.  
반대로 `InnoSample`처럼 `App -> Layers -> Features` 경계 안에서 각각 등장 위치가 분리되어 있으면, 그때부터는 새 기능이 들어와도 구조가 쉽게 무너지지 않습니다.

관련 글도 같이 보면 흐름이 더 자연스럽습니다.

- 아키텍처 배경: [Understanding Modularity]({% post_url 2024-04-27-understanding-modularity %}), [Clean Architecture]({% post_url 2024-05-13-clean-architecture %})
- 라이브러리 상세: [InnoDI]({% post_url 2026-02-23-innodi-swift-macro-di %}), [InnoFlow]({% post_url 2026-02-23-innoflow-unidirectional-architecture %}), [InnoRouter]({% post_url 2026-02-23-innorouter-swiftui-navigation %}), [InnoNetwork]({% post_url 2026-02-23-innonetwork-type-safe-networking %})

## 레포 링크

- [InnoSample GitHub 저장소](https://github.com/InnoSquadCorp/InnoSample)
- [InnoDI GitHub 저장소](https://github.com/InnoSquadCorp/InnoDI)
- [InnoFlow GitHub 저장소](https://github.com/InnoSquadCorp/InnoFlow)
- [InnoRouter GitHub 저장소](https://github.com/InnoSquadCorp/InnoRouter)
- [InnoNetwork GitHub 저장소](https://github.com/InnoSquadCorp/InnoNetwork)
