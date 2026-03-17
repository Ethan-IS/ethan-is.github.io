---
title: "InnoRouter: SwiftUI를 위한 타입 안전 내비게이션 프레임워크"
date: 2026-02-23 00:00:00 +0900
translation_key: innorouter-swiftui-navigation
lang: ko-KR
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, navigation, swiftui, coordinator, clean-architecture, unidirectional]
author: ethan
toc: true
comments: true
---

## 왜 다시 봐야 할까요?

처음 `NavigationStack`를 도입했을 때 SwiftUI 내비게이션은 분명 좋아졌습니다. 하지만 앱이 커지면 문제는 다시 생깁니다.

- path 상태가 여러 화면과 feature에 흩어집니다.
- 인증, 딥링크, 로깅, 정책 판단이 화면 코드에 섞입니다.
- "A에서 B로 간다"는 흐름을 테스트 가능한 데이터로 다루기 어렵습니다.

`InnoRouter` 3.0은 이 문제를 "내비게이션을 위한 typed runtime"으로 풀어냅니다. 핵심은 단순히 push/pop helper를 제공하는 것이 아니라, `RouteStack` 상태와 `NavigationCommand` 실행을 중심으로 SwiftUI 경계를 명확하게 나누는 데 있습니다.

이 글은 예전 "Intent 기반 내비게이션 프레임워크" 설명을 넘어, 현재 코드베이스가 실제로 제공하는 런타임 surface를 기준으로 InnoRouter를 다시 정리합니다.

---

## InnoRouter가 실제로 소유하는 것

README 기준으로 InnoRouter는 아래 다섯 축을 소유합니다.

| 축 | 핵심 타입 | 설명 |
|---|---|---|
| Stack state | `RouteStack`, `RouteStackValidator` | 현재 내비게이션 스냅샷과 검증 규칙 |
| Command execution | `NavigationCommand`, `NavigationEngine`, `NavigationResult` | push/pop/replace를 typed command로 실행 |
| SwiftUI authority | `NavigationStore`, `NavigationHost`, `@EnvironmentNavigationIntent` | 뷰가 상태를 직접 바꾸지 않고 intent를 보내도록 만드는 계층 |
| Modal authority | `ModalStore`, `ModalHost`, `ModalIntent` | `sheet`, `fullScreenCover`를 별도 surface로 분리 |
| Deep-link planning | `DeepLinkMatcher`, `DeepLinkPipeline`, `PendingDeepLink`, `NavigationPlan` | URL을 즉시 실행하지 않고 계획으로 다루는 모델 |

중요한 점은 InnoRouter가 앱의 모든 상태를 가져가지 않는다는 것입니다. 아래는 여전히 앱이 직접 소유해야 합니다.

- 인증/세션 라이프사이클
- 비즈니스 워크플로 상태
- 네트워크 재시도나 transport 상태
- `alert`, `confirmationDialog` 같은 feature-local presentation 상태

---

## Runtime 모델

InnoRouter의 중심에는 `RouteStack`과 `NavigationCommand`가 있습니다.

### 1. `RouteStack`

`RouteStack`은 현재 stack navigation 상태를 표현하는 값 타입입니다. 중요한 점은 "지금 화면이 어디인가"를 view-local side effect가 아니라 명시적인 스냅샷으로 다룬다는 것입니다.

```swift
import InnoRouter

let stack = try RouteStack<HomeRoute>(
    validating: [.list, .detail(id: "123")],
    using: .nonEmpty.combined(with: .rooted(at: .list))
)
```

`RouteStackValidator`를 붙이면 앱 규칙도 타입 근처로 끌어올릴 수 있습니다.

- 빈 스택을 금지할지
- 특정 루트에서만 시작하게 할지
- 중복 route를 허용할지

### 2. `NavigationCommand`

내비게이션 변경은 전부 command로 표현됩니다.

- `.push(route)`
- `.pop`
- `.popCount(n)`
- `.popTo(route)`
- `.popToRoot`
- `.replace([route])`
- `.sequence([...])`

이렇게 하면 "어디로 갔는가"보다 "어떤 명령을 실행했는가"를 테스트하고 로깅할 수 있습니다.

### 3. `NavigationEngine`과 결과 타입

실행 결과는 `NavigationResult`로 돌아옵니다. 그래서 명령 실패나 취소도 암묵적으로 삼키지 않습니다.

InnoRouter는 세 가지 실행 semantics를 구분합니다.

| 방식 | 언제 쓰면 좋은가 |
|---|---|
| Single command | 일반적인 push/pop 한 번 |
| Batch | 여러 command를 순차 실행하되 관찰은 한 번으로 묶고 싶을 때 |
| Transaction | 하나라도 실패하면 전체 commit을 취소해야 할 때 |

여기서 `.sequence`는 transaction이 아닙니다. 왼쪽부터 실행되고, 앞선 성공은 뒤 명령이 실패해도 유지됩니다.

---

## SwiftUI 진입점

SwiftUI에서 가장 중요한 타입은 `NavigationStore`입니다.

### `NavigationStore`는 shared authority다

`NavigationStore`는 단순한 convenience wrapper가 아니라 stack routing authority입니다.

- 현재 `RouteStack` 보유
- middleware 적용
- `NavigationStack(path:)`와의 path reconciliation 처리
- `NavigationIntent`를 실제 command 실행으로 변환

기본 예제는 현재 `Examples/StandaloneExample.swift`와 거의 같습니다.

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

### 뷰는 store를 직접 mutate하지 않는다

뷰는 `@EnvironmentNavigationIntent`를 통해 intent만 보냅니다.

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

현재 stack intent surface는 아래 일곱 가지입니다.

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

예전 설명에서 보이던 `deepLink(URL)` 같은 케이스는 현재 `NavigationIntent` surface에 없습니다. 딥링크는 stack intent가 아니라 별도의 planning 모델에서 다루는 것이 핵심 변화입니다.

---

## Coordinator는 정책 계층입니다

`Coordinator`는 또 다른 store가 아니라, SwiftUI intent와 실행 사이에 정책을 끼워 넣는 경계입니다.

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

이 레이어가 필요한 경우는 아래와 같습니다.

- 인증 정책이 화면 이동 전에 개입해야 할 때
- 앱 셸에서 여러 navigation authority를 조합해야 할 때
- view 코드가 알 필요 없는 라우팅 규칙을 한 곳에 모으고 싶을 때

실제 사용은 `CoordinatorHost` 또는 `CoordinatorSplitHost`로 연결합니다.

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

`FlowCoordinator`와 `TabCoordinator`도 있습니다. 다만 이것들은 `NavigationStore`를 대체하는 것이 아니라, step progression이나 탭 선택처럼 다른 종류의 UI 흐름을 보완하는 helper에 가깝습니다.

---

## Modal surface는 별도입니다

InnoRouter 3.0에서 중요하게 봐야 할 부분 중 하나는 modal routing을 stack routing과 분리했다는 점입니다.

`sheet`와 `fullScreenCover`는 `ModalStore`와 `ModalHost`로 다룹니다.

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

뷰에서는 `@EnvironmentModalIntent`로 presentation intent를 보냅니다.

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

여기서 얻는 장점은 분명합니다.

- stack path와 modal queue를 섞지 않습니다.
- `sheet`와 `fullScreenCover` 라이프사이클을 별도 관찰 지점으로 다룹니다.
- modal은 stack middleware와 다른 성격이므로 의도적으로 middleware를 노출하지 않습니다.

---

## Deep link는 실행이 아니라 계획입니다

예전 InnoRouter 설명에서 딥링크는 "URL이 들어오면 화면 이동" 정도로 보이기 쉽습니다. 현재 모델은 더 명확합니다.

1. `DeepLinkMatcher`가 URL을 route로 해석합니다.
2. `DeepLinkPipeline`이 scheme/host/auth policy를 적용합니다.
3. 결과는 `.plan`, `.pending`, `.rejected`, `.unhandled` 중 하나로 나옵니다.
4. 앱은 그 계획을 언제 실행할지 직접 결정합니다.

### Matcher

```swift
let matcher = DeepLinkMatcher<ProductRoute> {
    DeepLinkMapping("/products") { _ in .list }
    DeepLinkMapping("/products/:id") { params in
        params.firstValue(forName: "id").map { .detail(id: $0) }
    }
}
```

`DeepLinkMatcher`는 단순 매칭기 이상입니다. 현재 구현은 아래 authoring 문제를 진단할 수 있습니다.

- duplicate pattern
- wildcard shadowing
- parameter shadowing

즉, "동작은 되는데 나중에 이상한 URL이 앞 규칙에 먹히는" 문제를 문서화된 surface로 다룹니다.

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

`DeepLinkPipeline`의 핵심은 URL을 바로 실행하지 않는다는 점입니다.

- `.plan`: 지금 실행 가능한 계획
- `.pending`: 로그인 후 재개해야 하는 계획
- `.rejected`: 허용되지 않는 scheme/host
- `.unhandled`: 매칭 실패

SwiftUI에서 보통 이렇게 씁니다.

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

이 구조 덕분에 "로그인 후 다시 실행"도 임시 if-else가 아니라 `PendingDeepLink` 재개 시나리오로 모델링할 수 있습니다.

---

## Middleware는 cross-cutting policy layer입니다

현재 InnoRouter의 middleware는 단순 훅이 아니라 command boundary입니다.

### 할 수 있는 것

- command를 실행 전에 재작성
- 조건이 맞지 않으면 typed cancellation으로 중단
- 실행 후 결과를 가공하거나 로깅

### 할 수 없는 것

- store 상태를 직접 mutate

예를 들어 중복 push를 막을 수 있습니다.

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

핵심은 middleware가 "뷰 바깥 정책 계층"이라는 점입니다. 인증 가드, 분석 이벤트, 사전 조건 검사는 여기서 처리하고, 화면은 intent만 보냅니다.

---

## App boundary에서는 effect 모듈을 씁니다

앱 셸이나 coordinator boundary에서 store에 직접 접근하고 싶지 않다면 effect 모듈이 있습니다.

### `InnoRouterNavigationEffects`

이 모듈은 navigation command 실행을 감싸는 작은 facade입니다.

- `execute(_:)`
- `execute(_ commands:)`
- `executeTransaction(_:)`
- `executeGuarded(_:, prepare:)`

즉, 앱 경계 코드가 `NavigationStore` 내부 세부사항보다 "어떤 명령을 어떤 조건에서 실행할지"에 집중하게 도와줍니다.

### `InnoRouterDeepLinkEffects`

이 모듈은 deep-link planning과 boundary execution을 연결합니다.

- deep-link plan 실행
- typed outcome 수신
- 인증 후 pending deep link 재개

딥링크가 단순 URL parsing이 아니라 앱 경계 정책이라는 점을 코드 구조로 분리한 셈입니다.

---

## 매크로는 public-facing 기본 스타일입니다

현재 public examples는 hand-written `Route` 준수보다 `@Routable`를 기본 스타일로 사용합니다.

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

`@Routable`는 route enum의 수동 `Route` 선언을 제거합니다. README와 `Examples/`도 이 스타일을 기준으로 작성됩니다.

`@CasePathable`은 enum extraction과 조합이 필요할 때 쓰는 보조 매크로입니다. 모든 예제에 꼭 필요하진 않지만, route hierarchy를 더 다루기 쉽게 만드는 선택지로 보면 됩니다.

---

## 언제 무엇을 쓰면 될까요?

| 상황 | 권장 surface |
|---|---|
| 단일 stack 기반 앱 | `NavigationStore` + `NavigationHost` |
| 이동 전에 정책 판단 필요 | `CoordinatorHost` + `Coordinator.handle(_:)` |
| iPad/macOS detail stack | `NavigationSplitHost` 또는 `CoordinatorSplitHost` |
| `sheet`/`fullScreenCover` 분리 | `ModalStore` + `ModalHost` |
| URL 진입과 인증 재개 | `DeepLinkMatcher` + `DeepLinkPipeline` |
| 앱 셸 경계에서 실행 facade 필요 | `InnoRouterNavigationEffects`, `InnoRouterDeepLinkEffects` |

이 표만 기억해도 설계가 훨씬 단순해집니다. InnoRouter는 모든 흐름을 하나의 giant coordinator로 몰아넣는 프레임워크가 아니라, 상태와 권한 경계를 typed surface로 나누는 프레임워크입니다.

---

## 설치와 요구사항

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", from: "3.0.0")
]
```

현재 요구사항은 다음과 같습니다.

- iOS 18+
- macOS 15+
- tvOS 18+
- watchOS 11+
- Swift 6.2+

Swift 6.2와 strict concurrency 전제를 두고 설계되어 있고, `Sendable`, `@MainActor`, typed result surface가 런타임 전반에 반영되어 있습니다.

---

## 마무리

InnoRouter를 한 문장으로 요약하면 이렇습니다.

> InnoRouter는 SwiftUI 내비게이션 helper가 아니라, route state와 command execution을 중심으로 앱 경계를 정리하는 typed navigation runtime입니다.

그래서 좋은 점도 분명합니다.

1. 내비게이션 상태를 `RouteStack`으로 명시적으로 다룰 수 있습니다.
2. command, batch, transaction semantics를 구분해 테스트와 운영이 쉬워집니다.
3. stack, modal, deep link를 서로 다른 권한 경계로 분리할 수 있습니다.
4. coordinator와 effect boundary에서 앱 정책을 화면 바깥으로 밀어낼 수 있습니다.

복잡한 앱에서 내비게이션이 자꾸 "UI 코드의 일부"처럼 흩어진다면, InnoRouter는 그 문제를 상태와 실행 모델의 문제로 다시 정의하게 도와줍니다.

## 참고 자료

- [InnoRouter GitHub 저장소](https://github.com/InnoSquadCorp/InnoRouter)
- [InnoRouter 최신 DocC](https://innosquadcorp.github.io/InnoRouter/latest/)
- [SwiftUI NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)
