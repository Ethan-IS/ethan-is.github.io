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

## Why was it made?


With the release of SwiftUI's `NavigationStack`, the navigation code has become much cleaner. But when it comes to actual projects, there are still some headaches.

**The state is all over the place.** Some screens have the path as `@State`, some have it as `@Binding`, and some have it taken out from EnvironmentObject. When you look back a month later, it becomes difficult to know where and what you manage.

**Deep link processing becomes complicated.** If you take a single URL and try to process it, if-else branches will quickly add up. As concerns such as how to handle screens that require authentication and what to do when a deep link comes in while not logged in are repeated, temporary expedient codes accumulate.

**Difficult to test.** We often end up wondering how to verify the flow of “the user moved from screen A to screen B,” and end up checking by opening the actual screen.

**InnoRouter** was created to solve this problem. The core idea is simple.

> Let’s express navigation as data and make it flow in one direction.


---

## Core Concepts


### Three Principles


1. **State-based**: Screen transitions are expressed as a state called `RouteStack`. SwiftUI subscribes to this state and draws the screen.

2. **One-way flow**: The screen only sends the intent “I want to move.” The Store is responsible for actual state changes.

3. **Dependency Inversion**: The screen or coordinator only depends on the `Navigator` protocol. It is not coupled to a concrete implementation.


### Main Components


| component | role |
|---------|------|
| `Route` | An enum that identifies the screen. Receives parameters as associated values |
| `RouteStack` | Navigation route status |
| `NavigationCommand` | Commands like push and pop |
| `NavigationIntent` | Movement intent sent from screen |
| `NavigationStore` | Holds status and executes commands |
| `NavigationMiddleware` | Intervention before and after command execution (authentication guard, logging, etc.) |

---

## Basic usage


### Route definition


```swift
enum HomeRoute: Route {
    case list
    case detail(id: String)
    case settings
}
```

Route follows `Hashable` and `Sendable`. It serves as a screen identifier, so it's best to keep it as simple as possible.

### Store and Host Settings


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

`NavigationHost` wraps `NavigationStack` and shows the appropriate screen according to the route.

### Move around the screen


```swift
struct HomeView: View {
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

The screen does not mutate navigation state directly. It only sends intents like "go to detail."

---

## NavigationIntent type


```swift
public enum NavigationIntent<R: Route>: Sendable, Equatable {
    case go(R)              // navigate to one screen
    case goMany([R])        // push multiple screens at once
    case back               // go back one level
    case backBy(Int)        // go back N levels
    case backTo(R)          // pop back to a specific route
    case backToRoot         // pop to root
    case resetTo([R])       // replace stack
    case deepLink(URL)      // handle deep link
}
```

When an intent is sent, it is internally converted to the appropriate `NavigationCommand` and executed.

---

## Coordinator pattern


If the screen switching logic becomes complicated, separate it into a coordinator.

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
            // Handle routes that require authentication
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

The coordinator manages navigation policies in one place. Rules such as “Settings screen requires login” can be separated from the view.

### Use coordinator


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

## middleware


You can intervene before or after command execution. Things like authentication guard, logging, analysis, and duplication prevention are handled here.

### Prevent duplicate pushes


```swift
store.addMiddleware(
    AnyNavigationMiddleware(
        willExecute: { command, state in
            if case .push(let next) = command, state.path.last == next {
                return nil  // cancel duplicate consecutive pushes
            }
            return command
        }
    )
)
```

### Logging/Analysis


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

Thanks to middleware, the screen code becomes cleaner. Authentication checks, logging, etc. are not mixed into the view.

---

## Deep link processing


InnoRouter handles deep links structurally.

### pattern matching


```swift
let matcher = DeepLinkMatcher<ProductRoute> {
    DeepLinkMapping("/products") { _ in .list }
    DeepLinkMapping("/products/:id") { params in
        params.firstValue(forName: "id").map { .detail(id: $0) }
    }
}
```

It supports parameter extraction such as `:id`, and wildcards (`*`) can also be used.

### pipeline


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

What a pipeline does:

1. **Scheme/Host Verification** - Only allowed passes

2. **Apply authentication policy** - If the screen requires login, it is in pending status.

3. **Create execution plan** - decide which commands to execute


### Used in SwiftUI


```swift
.onOpenURL { url in
    switch pipeline.decide(for: url) {
    case .plan(let plan):
        for command in plan.commands {
            _ = store.execute(command)
        }
    case .pending(let pending):
        // store to run after login
        self.pendingDeepLink = pending
    case .rejected(let reason):
        print("Rejected: \(reason)")
    case .unhandled(let url):
        print("Unhandled URL: \(url)")
    }
}
```

After logging in, just run the saved `pendingDeepLink`.

---

## @Routable macro


This macro reduces boilerplate.

```swift
@Routable
enum HomeRoute {
    case list
    case detail(id: String)
    case settings(section: SettingsSection)
}
```

You can automatically do this by writing:

- `Route` protocol compliance

- Check case with `is(_:)` method

- Extracting related values ​​with subscript


---

## Problems InnoRouter Solve


| problem | How to solve |
|------|----------|
| Status scattered all over the place | `NavigationStore` is the single source of truth |
| Imperative navigation invocation | Intent-based declarative approach |
| Deep link if-else hell | Structured as `DeepLinkPipeline` |
| Authentication guards mixed into views | Separated into middleware |
| test difficulty | Commands are data, so unit testing is possible |
| The hassle of implementing the coordinator pattern | Provides `Coordinator` protocol and `CoordinatorHost` |

---

## Influenced by other frameworks


| framework | borrowed |
|-----------|----------|
| SwiftNavigation | Type-safe route/state modeling |
| TCACoordinators | Deterministic instruction execution, testing strategy |
| FlowStacks | Deep link playback model |
| Stinsen | Host-based coordinator boundary |

While collecting good ideas, we designed it to operate natively in SwiftUI.

---

## Real architecture example


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

## Requirements


- iOS 18+ / macOS 15+ / tvOS 18+ / watchOS 11+

- Swift 6.2+


Supports strict concurrency in Swift 6. All types are `Sendable`, and the necessary ones are isolated as `@MainActor`.

---

## Conclusion


InnoRouter is a framework that handles navigation as data.

### When recommended


- Apps with complex navigation flows

- If you want to handle deep links structurally

- When you want to use the coordinator pattern but the implementation is cumbersome

- When you want to test navigation logic


### Summary of Benefits


1. **Type safety** - route verification at compile time

2. **One-way flow** - Easy to track state

3. **Middleware** - Cleanly separates authentication, logging, and analysis

4. **Deep Link Pipeline** - Processes authentication policy and pending status

5. **Swift 6 ready** - Full support for Sendable, @MainActor


---

## Reference Materials


- [InnoRouter GitHub Repository](https://github.com/InnoSquadCorp/InnoRouter)

- [SwiftUI NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)

- [Coordinator Pattern](https://www.hackingwithswift.com/articles/175/advanced-coordinator-pattern-tutorial)
