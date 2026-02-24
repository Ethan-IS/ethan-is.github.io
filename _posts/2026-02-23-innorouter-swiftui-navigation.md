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

## 왜 만들었을까요?

SwiftUI의 `NavigationStack`이 나오면서 내비게이션 코드가 많이 깔끔해졌습니다. 하지만 실제 프로젝트에 들어가면 여전히 골치 아픈 문제가 있습니다.

**상태가 여기저기 흩어져 있습니다.** 어떤 화면은 `@State`로 경로를 들고 있고, 어떤 화면은 `@Binding`으로 넘겨받고, 또 어떤 화면은 EnvironmentObject에서 꺼내 씁니다. 한 달 뒤에 돌아보면 어디서 무엇을 관리하는지 파악하기가 어려워집니다.

**딥링크 처리가 복잡해집니다.** URL 하나를 받아 처리하려고 하면 if-else 분기가 빠르게 늘어납니다. 인증이 필요한 화면은 어떻게 처리할지, 로그인되지 않은 상태에서 딥링크가 들어오면 어떻게 할지 같은 고민이 반복되면서 임시 방편 코드가 쌓입니다.

**테스트하기 어렵습니다.** "사용자가 A 화면에서 B 화면으로 이동했다"는 흐름을 어떻게 검증할지 고민하게 되고, 결국 실제 화면을 띄워 확인하게 되는 경우가 많습니다.

**InnoRouter**는 이런 문제를 해결하기 위해 만들었습니다. 핵심 아이디어는 단순합니다.

> 내비게이션을 데이터로 표현하고, 단방향으로 흐르게 만들어 보겠습니다.

---

## 핵심 개념

### 세 가지 원칙

1. **상태 기반**: 화면 전환을 `RouteStack`이라는 상태로 표현합니다. SwiftUI가 이 상태를 구독해서 화면을 그립니다.
2. **단방향 흐름**: 화면은 "이동하고 싶다"는 의도(Intent)만 보냅니다. 실제 상태 변경은 Store가 담당합니다.
3. **의존성 역전**: 화면이나 코디네이터는 `Navigator` 프로토콜에만 의존합니다. 구체적인 구현과 결합하지 않습니다.

### 주요 컴포넌트

| 컴포넌트 | 역할 |
|---------|------|
| `Route` | 화면을 식별하는 enum. 연관값으로 파라미터를 받는다 |
| `RouteStack` | 내비게이션 경로 상태 |
| `NavigationCommand` | push, pop 같은 명령 |
| `NavigationIntent` | 화면에서 보내는 이동 의도 |
| `NavigationStore` | 상태를 들고 있고 명령을 실행한다 |
| `NavigationMiddleware` | 명령 실행 전후로 개입 (인증 가드, 로깅 등) |

---

## 기본 사용법

### Route 정의

```swift
enum HomeRoute: Route {
    case list
    case detail(id: String)
    case settings
}
```

Route는 `Hashable`과 `Sendable`을 따르면 됩니다. 화면 식별자 역할을 하니 최대한 단순하게 두는 것이 좋습니다.

### Store와 Host 설정

```swift
struct AppRoot: View {
    @State private var store = NavigationStore<HomeRoute>()

    var body: some View {
        NavigationHost(store: store) { route in
            switch route {
            case .list:
                HomeView()
            case .detail(let id):
                DetailView(id: id)
            case .settings:
                SettingsView()
            }
        } root: {
            HomeView()
        }
    }
}
```

`NavigationHost`가 `NavigationStack`을 감싸고, route에 따라 적절한 화면을 보여준다.

### 화면에서 이동하기

```swift
struct HomeView: View {
    @EnvironmentNavigationIntent(HomeRoute.self) private var navigationIntent

    var body: some View {
        List {
            Button("상세 화면") {
                navigationIntent.send(.go(.detail(id: "123")))
            }
            Button("설정") {
                navigationIntent.send(.go(.settings))
            }
            Button("뒤로") {
                navigationIntent.send(.back)
            }
        }
    }
}
```

화면은 직접 상태를 건드리지 않는다. 그냥 "상세 화면 가고 싶어"라고 Intent만 보낸다.

---

## NavigationIntent 종류

```swift
public enum NavigationIntent<R: Route>: Sendable, Equatable {
    case go(R)              // 단일 화면 이동
    case goMany([R])        // 여러 화면 한 번에 push
    case back               // 한 칸 뒤로
    case backBy(Int)        // N칸 뒤로
    case backTo(R)          // 특정 화면까지 pop
    case backToRoot         // 루트까지
    case resetTo([R])       // 스택 교체
    case deepLink(URL)      // 딥링크 처리
}
```

Intent를 보내면 내부적으로 적절한 `NavigationCommand`로 변환되어 실행된다.

---

## 코디네이터 패턴

화면 전환 로직이 복잡해지면 코디네이터로 분리한다.

```swift
@MainActor
@Observable
final class AppCoordinator: Coordinator {
    typealias RouteType = AppRoute

    let store = NavigationStore<AppRoute>()
    var isAuthenticated = false

    func handle(_ intent: NavigationIntent<AppRoute>) {
        switch intent {
        case .go(let route):
            // 인증이 필요한 화면 처리
            if route == .settings, !isAuthenticated {
                store.send(.go(.login))
                return
            }
            store.send(intent)
        default:
            store.send(intent)
        }
    }

    @ViewBuilder
    func destination(for route: AppRoute) -> some View {
        switch route {
        case .home: HomeView()
        case .settings: SettingsView()
        case .login: LoginView()
        }
    }
}
```

코디네이터는 내비게이션 정책을 한 곳에서 관리한다. "설정 화면은 로그인 필요" 같은 규칙을 뷰에서 분리할 수 있다.

### 코디네이터 사용

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

---

## 미들웨어

명령 실행 전후에 개입할 수 있다. 인증 가드, 로깅, 분석, 중복 방지 같은 걸 여기서 처리한다.

### 중복 push 방지

```swift
store.addMiddleware(
    AnyNavigationMiddleware(
        willExecute: { command, state in
            if case .push(let next) = command, state.path.last == next {
                return nil  // 같은 화면 연속 push 취소
            }
            return command
        }
    )
)
```

### 로깅/분석

```swift
store.addMiddleware(
    AnyNavigationMiddleware(
        didExecute: { command, result, state in
            analytics.track("navigation", [
                "command": String(describing: command),
                "result": String(describing: result)
            ])
        }
    )
)
```

미들웨어 덕분에 화면 코드가 깔끔해진다. 인증 체크, 로깅 같은 게 뷰에 섞이지 않는다.

---

## 딥링크 처리

InnoRouter는 딥링크를 구조적으로 처리한다.

### 패턴 매칭

```swift
let matcher = DeepLinkMatcher<ProductRoute> {
    DeepLinkMapping("/products") { _ in .list }
    DeepLinkMapping("/products/:id") { params in
        params.firstValue(forName: "id").map { .detail(id: $0) }
    }
}
```

`:id` 같은 파라미터 추출을 지원하고, 와일드카드(`*`)도 쓸 수 있다.

### 파이프라인

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

파이프라인이 하는 일:

1. **스킴/호스트 검증** - 허용된 것만 통과
2. **인증 정책 적용** - 로그인 필요한 화면이면 보류(pending) 상태로
3. **실행 계획 생성** - 어떤 명령들을 실행할지 결정

### SwiftUI에서 사용

```swift
.onOpenURL { url in
    switch pipeline.decide(for: url) {
    case .plan(let plan):
        for command in plan.commands {
            _ = store.execute(command)
        }
    case .pending(let pending):
        // 로그인 후 실행하기 위해 저장
        self.pendingDeepLink = pending
    case .rejected(let reason):
        print("거부됨: \(reason)")
    case .unhandled(let url):
        print("처리할 수 없음: \(url)")
    }
}
```

로그인 후에는 저장해둔 `pendingDeepLink`를 실행하면 된다.

---

## @Routable 매크로

보일러플레이트를 줄이는 매크로다.

```swift
@Routable
enum HomeRoute {
    case list
    case detail(id: String)
    case settings(section: SettingsSection)
}
```

이렇게 쓰면 자동으로:

- `Route` 프로토콜 준수
- `is(_:)` 메서드로 케이스 체크
- 서브스크립트로 연관값 추출

---

## InnoRouter가 해결하는 문제들

| 문제 | 해결 방법 |
|------|----------|
| 상태가 여기저기 흩어짐 | `NavigationStore`가 단일 소스 오브 트루스 |
| 명령형 내비게이션 호출 | Intent 기반 선언형 접근 |
| 딥링크 if-else 지옥 | `DeepLinkPipeline`으로 구조화 |
| 인증 가드가 뷰에 섞임 | 미들웨어로 분리 |
| 테스트 어려움 | 명령이 데이터라 유닛 테스트 가능 |
| 코디네이터 패턴 구현 번거로움 | `Coordinator` 프로토콜과 `CoordinatorHost` 제공 |

---

## 다른 프레임워크에서 영향 받은 것

| 프레임워크 | 차용한 것 |
|-----------|----------|
| SwiftNavigation | 타입 안전 route/state 모델링 |
| TCACoordinators | 결정적 명령 실행, 테스트 전략 |
| FlowStacks | 딥링크 재생 모델 |
| Stinsen | Host 기반 코디네이터 경계 |

좋은 아이디어들을 모으면서도, SwiftUI 네이티브하게 동작하도록 설계했습니다.

---

## 실제 아키텍처 예시

```swift
// Domain
protocol ProductRepository {
    func fetchProduct(id: String) async throws -> Product
}

// Presentation
@MainActor
@Observable
class ProductDetailViewModel {
    let productID: String
    let repository: any ProductRepository
    var product: Product?

    init(productID: String, repository: any ProductRepository) {
        self.productID = productID
        self.repository = repository
    }

    func load() async {
        product = try? await repository.fetchProduct(id: productID)
    }
}

// Route
enum ProductRoute: Route {
    case list
    case detail(id: String)
}

// Coordinator
@MainActor
@Observable
final class ProductCoordinator: Coordinator {
    typealias RouteType = ProductRoute

    let store = NavigationStore<ProductRoute>()
    let repository: any ProductRepository

    init(repository: any ProductRepository) {
        self.repository = repository
    }

    func handle(_ intent: NavigationIntent<ProductRoute>) {
        store.send(intent)
    }

    @ViewBuilder
    func destination(for route: ProductRoute) -> some View {
        switch route {
        case .list:
            ProductListView()
        case .detail(let id):
            ProductDetailView(viewModel: ProductDetailViewModel(
                productID: id,
                repository: repository
            ))
        }
    }
}

// App
@main
struct ProductApp: App {
    @State private var coordinator: ProductCoordinator

    init() {
        let repository = ProductRepositoryImpl()
        _coordinator = State(initialValue: ProductCoordinator(repository: repository))
    }

    var body: some Scene {
        WindowGroup {
            CoordinatorHost(coordinator: coordinator) {
                ProductListView()
            }
        }
    }
}
```

---

## 요구사항

- iOS 18+ / macOS 15+ / tvOS 18+ / watchOS 11+
- Swift 6.2+

Swift 6의 strict concurrency를 지원합니다. 모든 타입이 `Sendable`이고, 필요한 곳은 `@MainActor`로 격리되어 있습니다.

---

## 결론

InnoRouter는 **내비게이션을 데이터로 다루는 프레임워크**입니다.

### 추천하는 경우

- 복잡한 내비게이션 흐름이 있는 앱
- 딥링크를 구조적으로 처리하고 싶은 경우
- 코디네이터 패턴을 쓰고 싶은데 구현이 귀찮을 때
- 내비게이션 로직을 테스트하고 싶을 때

### 장점 요약

1. **타입 안전** - 컴파일 타임에 route 검증
2. **단방향 흐름** - 상태 추적이 쉽습니다
3. **미들웨어** - 인증, 로깅, 분석을 깔끔하게 분리
4. **딥링크 파이프라인** - 인증 정책, pending 상태까지 처리
5. **Swift 6 준비됨** - Sendable, @MainActor 완벽 지원

---

## 참고 자료

- [InnoRouter GitHub 저장소](https://github.com/InnoSquadCorp/InnoRouter)
- [SwiftUI NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)
- [코디네이터 패턴](https://www.hackingwithswift.com/articles/175/advanced-coordinator-pattern-tutorial)
