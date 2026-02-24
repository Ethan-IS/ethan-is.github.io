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

## Introduction


Since the introduction of SwiftUI, concerns about state management have deepened. Various property wrappers are provided, such as `@State`, `@StateObject`, and `@EnvironmentObject`, but as the project grows, it becomes difficult to track the flow of states. You've probably found yourself switching between files to answer the question, "Where did this state change?"

**InnoFlow** is a one-way architecture framework created to solve this problem. While based on Elm Architecture, it integrates naturally with SwiftUI by actively utilizing Swift 6's `@Observable` macro.

## Why did you create InnoFlow?


### Limitations of Existing State Management


SwiftUI's built-in state management tools are great on a small scale. However, there are limitations in the following situations:

```swift
// State mutation logic scattered across the view
struct ProfileView: View {
    @State private var user: User?
    @State private var isLoading = false
    @State private var errorMessage: String?

    var body: some View {
        // ...
    }

    func loadUser() async {
        isLoading = true
        errorMessage = nil
        do {
            user = try await userService.fetch()
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }
}
```

This approach causes the following problems:

1. **State change logic is distributed** - multiple methods modify state directly

2. **Asynchronous processing complexity** - Error handling and loading state management are repeated

3. **Difficult to test** - State changes are difficult to isolate and test

4. **Debugging Difficulty** - Difficult to track when and why state changed


### Why Unidirectional Architecture


InnoFlow solves this problem through **Unidirectional Data Flow**:

```
Action → Reduce → State Mutation → Effect → Action → ...
```

All state changes are initiated via **Action** and are centralized and processed in **Reducer**. If you do this:

- State change logic is **centralized**

- **Cause (Action) and result (State**) of state change are clear

- **Testable** Separate logic into pure functions


---

## Core Concepts


### 1. `@InnoFlow` macro


When you apply the `@InnoFlow` macro to a struct, Reducer protocol compliant code is automatically generated.

```swift
@InnoFlow
struct CounterFeature {
    // State: must conform to Sendable
    struct State: Equatable, Sendable, DefaultInitializable {
        var count = 0
        @BindableField var step = 1  // bindable from SwiftUI
    }

    // Actions: events that trigger state changes
    enum Action: Equatable, Sendable {
        case increment
        case decrement
        case reset
        case setStep(Int)
    }

    // Reducer: the single place where state changes happen
    func reduce(into state: inout State, action: Action) -> EffectTask<Action> {
        switch action {
        case .increment:
            state.count += state.step
            return .none
        case .decrement:
            state.count -= state.step
            return .none
        case .reset:
            state.count = 0
            return .none
        case .setStep(let newStep):
            state.step = max(1, newStep)
            return .none
        }
    }
}
```

### 2. `Store` - Observable state container


Store is a state container that wraps a Reducer and is observable in SwiftUI.

```swift
import SwiftUI
import InnoFlow

struct CounterView: View {
    @State private var store = Store(reducer: CounterFeature())

    var body: some View {
        VStack(spacing: 20) {
            Text("Count: \(store.count)")
                .font(.largeTitle)

            HStack(spacing: 32) {
                Button("-") { store.send(.decrement) }
                Button("Reset") { store.send(.reset) }
                Button("+") { store.send(.increment) }
            }

            Stepper(
                "Step: \(store.step)",
                value: store.binding(\.step, send: { .setStep($0) })
            )
        }
    }
}
```

Since it is based on `@Observable`, the view is automatically updated just by accessing properties such as `store.count` and `store.step`.

### 3. `EffectTask` - Unified model of asynchronous operations


EffectTask is a DSL that expresses asynchronous tasks declaratively.

```swift
// no work
return .none

// send an action immediately
return .send(.loadingCompleted)

// run asynchronous work
return .run { send in
    let data = try await networkService.fetch()
    await send(.dataLoaded(data))
}

// run in parallel
return .merge(
    .run { await send(.loadUser()) },
    .run { await send(.loadSettings()) }
)

// run sequentially
return .concatenate(
    .send(.startLoading),
    .run { /* async work */ }
)

// cancel
return .cancel("network-request")
```

---

## Practical example: Todo app


### Feature definition


```swift
@InnoFlow
struct TodoFeature {
    struct State: Equatable, Sendable, DefaultInitializable {
        var todos: [Todo] = []
        var isLoading = false
        var errorMessage: String?
        @BindableField var filter: Filter = .all
    }

    enum Action: Equatable, Sendable {
        case loadTodos
        case addTodo(String)
        case toggleTodo(UUID)
        case deleteTodo(UUID)
        case setFilter(Filter)

        // Internal actions (effect output)
        case _todosLoaded([Todo])
        case _loadFailed(String)
    }

    // dependency injection
    let todoService: TodoServiceProtocol

    init(todoService: TodoServiceProtocol = TodoService.shared) {
        self.todoService = todoService
    }

    func reduce(into state: inout State, action: Action) -> EffectTask<Action> {
        switch action {
        case .loadTodos:
            state.isLoading = true
            state.errorMessage = nil

            let service = self.todoService
            return .run { send in
                do {
                    let todos = try await service.loadTodos()
                    await send(._todosLoaded(todos))
                } catch {
                    await send(._loadFailed(error.localizedDescription))
                }
            }
            .cancellable("todo-load", cancelInFlight: true)

        case ._todosLoaded(let todos):
            state.todos = todos
            state.isLoading = false
            return .none

        case ._loadFailed(let message):
            state.errorMessage = message
            state.isLoading = false
            return .none

        case .addTodo(let title):
            var newTodo = Todo(title: title)
            newTodo.id = UUID()
            state.todos.append(newTodo)
            return .none

        case .toggleTodo(let id):
            if let index = state.todos.firstIndex(where: { $0.id == id }) {
                state.todos[index].isCompleted.toggle()
            }
            return .none

        case .deleteTodo(let id):
            state.todos.removeAll { $0.id == id }
            return .none

        case .setFilter(let filter):
            state.filter = filter
            return .none
        }
    }
}
```

### SwiftUI View


```swift
struct TodoView: View {
    @State private var store = Store(
        reducer: TodoFeature(todoService: TodoService.shared)
    )

    var filteredTodos: [Todo] {
        switch store.filter {
        case .all: return store.todos
        case .active: return store.todos.filter { !$0.isCompleted }
        case .completed: return store.todos.filter { $0.isCompleted }
        }
    }

    var body: some View {
        NavigationStack {
            Group {
                if store.isLoading {
                    ProgressView("Loading...")
                } else {
                    List {
                        ForEach(filteredTodos) { todo in
                            TodoRowView(todo: todo) {
                                store.send(.toggleTodo(todo.id))
                            }
                        }
                        .onDelete { indexSet in
                            for index in indexSet {
                                store.send(.deleteTodo(filteredTodos[index].id))
                            }
                        }
                    }
                }
            }
            .navigationTitle("Todos")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Picker("Filter", selection: store.binding(\.filter, send: { .setFilter($0) })) {
                        Text("All").tag(Filter.all)
                        Text("Active").tag(Filter.active)
                        Text("Completed").tag(Filter.completed)
                    }
                    .pickerStyle(.menu)
                }
            }
            .alert("Error", isPresented: .constant(store.errorMessage != nil)) {
                Button("OK") { store.send(.loadTodos) }
            } message: {
                if let error = store.errorMessage {
                    Text(error)
                }
            }
            .task {
                store.send(.loadTodos)
            }
        }
    }
}
```

---

## Advanced features of EffectTask


### Debounce and Throttle


This is useful for handling user input.

```swift
case .searchTextChanged(let text):
    state.searchText = text

    let service = self.searchService
    return .run { send in
        let results = try await service.search(text)
        await send(.searchResultsLoaded(results))
    }
    .debounce("search", for: .milliseconds(300))
    // run 300ms after the user stops typing
```

```swift
case .refreshPulled:
    return .run { /* refresh logic */ }
    .throttle("refresh", for: .seconds(1), leading: false, trailing: true)
    // ignore additional requests within 1 second
```

### animated movie


Apply animation when state changes.

```swift
case .addItem:
    state.items.append(newItem)
    return .none
    .animation(.spring())
```

### Cancellable Action


Manage long-running tasks.

```swift
case .startLongTask:
    return .run { send in
        // long-running task
    }
    .cancellable("long-task", cancelInFlight: true)

case .cancelTask:
    return .cancel("long-task")
```

---

## Testing


InnoFlow supports deterministic testing through `TestStore`.

```swift
import Testing
import InnoFlowTesting

@Suite("TodoFeature Tests")
@MainActor
struct TodoFeatureTests {

    @Test("Add todo")
    func testAddTodo() async {
        let store = TestStore(
            reducer: TodoFeature(todoService: MockTodoService())
        )

        await store.send(.addTodo("New todo")) {
            $0.todos.count = 1
            $0.todos.first?.title = "New todo"
        }

        await store.assertNoMoreActions()
    }

    @Test("Todo loading flow")
    func testLoadTodosFlow() async {
        let mockService = MockTodoService()
        mockService.mockTodos = [
            Todo(id: UUID(), title: "Test todo")
        ]

        let store = TestStore(
            reducer: TodoFeature(todoService: mockService)
        )

        // Send action and assert state changes
        await store.send(.loadTodos) {
            $0.isLoading = true
            $0.errorMessage = nil
        }

        // Verify action emitted from effect
        await store.receive(._todosLoaded(mockService.mockTodos)) {
            $0.todos = mockService.mockTodos
            $0.isLoading = false
        }

        await store.assertNoMoreActions()
    }
}
```

### Benefits of Testing


1. **Pure function test** - Reducer is a pure function without side effects

2. **Time independent** - even asynchronous operations can be tested deterministically

3. **State Snapshot** - Clearly verify the state after each action


---

## Store Scope - Parent-child state isolation


For large apps, it's a good idea to separate Stores into functional units. InnoFlow can create a child Store from a parent Store through `scope`.

```swift
// Parent feature
@InnoFlow
struct AppFeature {
    struct State: Equatable, Sendable {
        var todo: TodoFeature.State = .init()
        var profile: ProfileFeature.State = .init()
    }

    enum Action {
        case todo(TodoFeature.Action)
        case profile(ProfileFeature.Action)
    }

    func reduce(into state: inout State, action: Action) -> EffectTask<Action> {
        switch action {
        case .todo(let todoAction):
            // Delegate to child feature
            return TodoFeature().reduce(into: &state.todo, action: todoAction)
                .map { Action.todo($0) }

        case .profile(let profileAction):
            return ProfileFeature().reduce(into: &state.profile, action: profileAction)
                .map { Action.profile($0) }
        }
    }
}

// Use scope in child view
struct TodoSectionView: View {
    let store: Store<AppFeature>

    var body: some View {
        TodoFeatureView(
            store: store.scope(
                state: \.todo,
                action: AppFeature.Action.todo
            )
        )
    }
}
```

---

## Comparison with other frameworks


| function | InnoFlow | TCA | ReactorKit | MVVM |
|------|----------|-----|------------|------|
| learning curve | lowness | height | middle | lowness |
| SwiftUI integration | native | native | Need Rx | passivity |
| boiler plate | less | plenty | middle | less |
| Testability | height | height | height | middle |
| Status traceability | height | height | height | lowness |
| Effect Management | DSL | complication | Rx-based | passivity |

### What InnoFlow pursues


- **Practical**: Retains the power of TCA, but lowers the learning curve

- **SwiftUI First**: Natural integration by actively utilizing `@Observable`

- **Clear runtime model**: Effect lifecycle and cancellation are clear


---

## development philosophy


### v2 design principles


InnoFlow v2 has been redesigned based on the following principles:

1. **Single Reducer Contract** - `reduce(into:action:) -> EffectTask<Action>` combined into one

2. **Explicit asynchronous model** - run/merge/concatenate/cancel integration with EffectTask DSL

3. **Cancellation completion contract** - Store cancellation API is async, ensuring deterministic cleanup

4. **SwiftUI First Runtime** - `@Observable` Store + `@MainActor` Adapter

5. **Strict binding intent** - Only the `@BindableField` property can be bound in SwiftUI

6. **Deterministic Testing** - Timeout/Cancellation Oriented TestStore


### Why did you make this choice?


These choices were made to solve problems in the previous architecture:

- Force all changes to go through Action to prevent **implicit state changes**

- Treat Effect as a first-class object to avoid **untraceable asynchrony**

- Keep Reducer as a pure function to eliminate **untestable logic**

- Explicit opt-in with `@BindableField` to prevent **spaghetti binding**


---

## Installation


### Swift Package Manager


```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "2.0.0")
]
```

### Requirements


- iOS 18.0+ / macOS 15.0+ / tvOS 18.0+ / watchOS 11.0+

- Swift 6.2+


---

## Conclusion


InnoFlow aims for **a practical one-way architecture suitable for the SwiftUI era**.

### Recommended for


- SwiftUI developer concerned about **state management complexity**

- Team looking for **testable architecture**

- **Those who feel that TCA is too complicated**

- Swift developer interested in **Elm Architecture**


### Summary of InnoFlow Benefits


1. **@InnoFlow Macro** - Define Reducer without boilerplate

2. **@Observable Store** - Natural integration with SwiftUI

3. **EffectTask DSL** - Declarative asynchronous task management

4. **TestStore** - Supports deterministic testing

5. **@BindableField** - explicit and safe binding

6. **Low learning curve** - Leverage existing SwiftUI knowledge


As soon as state management becomes complex, consider InnoFlow. Experience the predictability and testability of one-way data flow.

---

## Reference Materials


- [InnoFlow GitHub Repository](https://github.com/InnoSquadCorp/InnoFlow)

- [InnoFlow API Documentation (DocC)](https://innosquad-mdd.github.io/InnoFlow/documentation/innoflow/)

- [Elm Architecture](https://guide.elm-lang.org/architecture/)

- [TCA (The Composable Architecture)](https://github.com/pointfreeco/swift-composable-architecture)
