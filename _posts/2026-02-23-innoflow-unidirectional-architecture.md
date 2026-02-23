---
title: "InnoFlow: SwiftUI를 위한 단방향 아키텍처 프레임워크"
date: 2026-02-23 00:00:00 +0900
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, state-management, unidirectional, elm-architecture, tca, swiftui]
author: ethan
toc: true
comments: true
---

## 들어가며

SwiftUI가 도입된 이후, 상태 관리에 대한 고민은 깊어졌습니다. `@State`, `@StateObject`, `@EnvironmentObject` 등 다양한 프로퍼티 래퍼가 제공되지만, 프로젝트가 커질수록 상태의 흐름을 추적하기 어려워집니다. "이 상태가 어디서 변경되었지?"라는 질문에 답하기 위해 여러 파일을 오가는 경험, 한 번쯤 있으실 겁니다.

**InnoFlow**는 이 문제를 해결하기 위해 만들어진 단방향 아키텍처 프레임워크입니다. Elm Architecture를 기반으로 하면서도, Swift 6의 `@Observable` 매크로를 적극 활용해 SwiftUI와 자연스럽게 통합됩니다.

## 왜 InnoFlow를 만들었나요?

### 기존 상태 관리의 한계

SwiftUI의 기본 상태 관리 도구들은 작은 규모에서는 훌륭합니다. 하지만 다음과 같은 상황에서는 한계를 맞이합니다:

```swift
// 뷰 곳곳에 흩어진 상태 변경 로직
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

이런 방식은 다음과 같은 문제를 야기합니다:

1. **상태 변경 로직이 분산됨** - 여러 메서드에서 상태를 직접 수정
2. **비동기 처리 복잡도** - 에러 처리, 로딩 상태 관리가 반복됨
3. **테스트 어려움** - 상태 변경을 격리해서 테스트하기 까다로움
4. **디버깅 난이도** - 상태가 언제, 왜 변경되었는지 추적하기 어려움

### 단방향 아키텍처의 답

InnoFlow는 **단방향 데이터 흐름(Unidirectional Data Flow)**을 통해 이 문제를 해결합니다:

```
Action → Reduce → State Mutation → Effect → Action → ...
```

모든 상태 변경은 **Action**을 통해 시작되고, **Reducer**에서 한 곳에 모여 처리됩니다. 이렇게 하면:

- 상태 변경 로직이 **한 곳에 집중**됨
- 상태 변경의 **원인(Action)과 결과(State)**가 명확함
- **테스트 가능한** 순수 함수로 로직을 분리 가능

---

## 핵심 개념

### 1. `@InnoFlow` 매크로

`@InnoFlow` 매크로를 struct에 적용하면, Reducer 프로토콜 준수 코드가 자동으로 생성됩니다.

```swift
@InnoFlow
struct CounterFeature {
    // 상태: 반드시 Sendable이어야 함
    struct State: Equatable, Sendable, DefaultInitializable {
        var count = 0
        @BindableField var step = 1  // SwiftUI 바인딩 가능
    }

    // 액션: 상태 변경을 유발하는 이벤트
    enum Action: Equatable, Sendable {
        case increment
        case decrement
        case reset
        case setStep(Int)
    }

    // Reducer: 상태 변경 로직의 유일한 장소
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

### 2. `Store` - 관찰 가능한 상태 컨테이너

Store는 Reducer를 감싸고 SwiftUI에서 관찰 가능한 상태 컨테이너입니다.

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

`@Observable` 기반이므로, `store.count`, `store.step` 같은 프로퍼티 접근만으로도 자동으로 뷰가 갱신됩니다.

### 3. `EffectTask` - 비동기 작업의 통합 모델

EffectTask는 비동기 작업을 선언적으로 표현하는 DSL입니다.

```swift
// 작업 없음
return .none

// 즉시 액션 전송
return .send(.loadingCompleted)

// 비동기 작업 실행
return .run { send in
    let data = try await networkService.fetch()
    await send(.dataLoaded(data))
}

// 병렬 실행
return .merge(
    .run { await send(.loadUser()) },
    .run { await send(.loadSettings()) }
)

// 순차 실행
return .concatenate(
    .send(.startLoading),
    .run { /* 비동기 작업 */ }
)

// 취소
return .cancel("network-request")
```

---

## 실전 예제: Todo 앱

### Feature 정의

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

        // 내부 액션 (Effect 결과)
        case _todosLoaded([Todo])
        case _loadFailed(String)
    }

    // 의존성 주입
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

### SwiftUI 뷰

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

## EffectTask의 고급 기능

### Debounce와 Throttle

사용자 입력을 처리할 때 유용합니다.

```swift
case .searchTextChanged(let text):
    state.searchText = text

    let service = self.searchService
    return .run { send in
        let results = try await service.search(text)
        await send(.searchResultsLoaded(results))
    }
    .debounce("search", for: .milliseconds(300))
    // 사용자가 입력을 멈춘 후 300ms 뒤에 실행
```

```swift
case .refreshPulled:
    return .run { /* 새로고침 로직 */ }
    .throttle("refresh", for: .seconds(1), leading: false, trailing: true)
    // 1초 내 추가 요청 무시
```

### 애니메이션

상태 변경 시 애니메이션을 적용합니다.

```swift
case .addItem:
    state.items.append(newItem)
    return .none
    .animation(.spring())
```

### 취소 가능한 작업

장기 실행 작업을 관리합니다.

```swift
case .startLongTask:
    return .run { send in
        // 긴 작업
    }
    .cancellable("long-task", cancelInFlight: true)

case .cancelTask:
    return .cancel("long-task")
```

---

## 테스트하기

InnoFlow는 `TestStore`를 통해 결정론적 테스트를 지원합니다.

```swift
import Testing
import InnoFlowTesting

@Suite("TodoFeature Tests")
@MainActor
struct TodoFeatureTests {

    @Test("할 일 추가")
    func testAddTodo() async {
        let store = TestStore(
            reducer: TodoFeature(todoService: MockTodoService())
        )

        await store.send(.addTodo("새 할 일")) {
            $0.todos.count = 1
            $0.todos.first?.title = "새 할 일"
        }

        await store.assertNoMoreActions()
    }

    @Test("할 일 로드 플로우")
    func testLoadTodosFlow() async {
        let mockService = MockTodoService()
        mockService.mockTodos = [
            Todo(id: UUID(), title: "테스트 할 일")
        ]

        let store = TestStore(
            reducer: TodoFeature(todoService: mockService)
        )

        // 액션 전송 및 상태 변화 검증
        await store.send(.loadTodos) {
            $0.isLoading = true
            $0.errorMessage = nil
        }

        // Effect에서 발생한 액션 검증
        await store.receive(._todosLoaded(mockService.mockTodos)) {
            $0.todos = mockService.mockTodos
            $0.isLoading = false
        }

        await store.assertNoMoreActions()
    }
}
```

### 테스트의 장점

1. **순수 함수 테스트** - Reducer는 사이드 이펙트 없는 순수 함수
2. **시간 독립적** - 비동기 작업도 결정론적으로 테스트 가능
3. **상태 스냅샷** - 각 액션 후의 상태를 명확히 검증

---

## Store Scope - 부모-자식 상태 격리

대규모 앱에서는 기능 단위로 Store를 분리하는 것이 좋습니다. InnoFlow는 `scope`를 통해 부모 Store에서 자식 Store를 생성할 수 있습니다.

```swift
// 부모 Feature
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
            // 자식 Feature에 위임
            return TodoFeature().reduce(into: &state.todo, action: todoAction)
                .map { Action.todo($0) }

        case .profile(let profileAction):
            return ProfileFeature().reduce(into: &state.profile, action: profileAction)
                .map { Action.profile($0) }
        }
    }
}

// 자식 뷰에서 스코프 사용
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

## 다른 프레임워크와 비교

| 기능 | InnoFlow | TCA | ReactorKit | MVVM |
|------|----------|-----|------------|------|
| 학습 곡선 | 낮음 | 높음 | 중간 | 낮음 |
| SwiftUI 통합 | 네이티브 | 네이티브 | Rx 필요 | 수동 |
| 보일러플레이트 | 적음 | 많음 | 중간 | 적음 |
| 테스트 용이성 | 높음 | 높음 | 높음 | 중간 |
| 상태 추적성 | 높음 | 높음 | 높음 | 낮음 |
| Effect 관리 | DSL | 복잡 | Rx 기반 | 수동 |

### InnoFlow가 추구하는 것

- **실용성**: TCA의 강력함을 유지하되, 학습 곡선을 낮춤
- **SwiftUI 퍼스트**: `@Observable`을 적극 활용해 자연스러운 통합
- **명확한 런타임 모델**: Effect 생명주기와 취소가 명확함

---

## 개발 철학

### v2 설계 원칙

InnoFlow v2는 다음 원칙을 기반으로 재설계되었습니다:

1. **단일 Reducer 계약** - `reduce(into:action:) -> EffectTask<Action>` 하나로 통합
2. **명시적 비동기 모델** - EffectTask DSL로 run/merge/concatenate/cancel 통합
3. **취소 완료 계약** - Store 취소 API가 async로, 결정론적 정리 보장
4. **SwiftUI 퍼스트 런타임** - `@Observable` Store + `@MainActor` 어댑터
5. **엄격한 바인딩 의도** - `@BindableField` 프로퍼티만 SwiftUI에서 바인딩 가능
6. **결정론적 테스트** - 타임아웃/취소 지향 TestStore

### 왜 이런 선택을 했나요?

기존 아키텍처에서 겪은 문제들을 해결하기 위해서입니다:

- **암시적 상태 변경**을 방지하고자 모든 변경이 Action을 거치도록 강제
- **추적 불가능한 비동기**를 피하고자 Effect를 일급 객체로 다룸
- **테스트 불가능한 로직**을 없애고자 Reducer를 순수 함수로 유지
- **스파게티 바인딩**을 막고자 `@BindableField`로 명시적 opt-in

---

## 설치

### Swift Package Manager

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoFlow.git", from: "2.0.0")
]
```

### 요구사항

- iOS 18.0+ / macOS 15.0+ / tvOS 18.0+ / watchOS 11.0+
- Swift 6.2+

---

## 결론

InnoFlow는 **SwiftUI 시대에 맞는 실용적인 단방향 아키텍처**를 지향합니다.

### 추천 대상

- **상태 관리 복잡도**에 고민인 SwiftUI 개발자
- **테스트 가능한 아키텍처**를 찾는 팀
- **TCA는 너무 복잡**하다고 느끼는 분
- **Elm Architecture**에 관심 있는 Swift 개발자

### InnoFlow의 장점 요약

1. **@InnoFlow 매크로** - 보일러플레이트 없이 Reducer 정의
2. **@Observable Store** - SwiftUI와 자연스러운 통합
3. **EffectTask DSL** - 선언적 비동기 작업 관리
4. **TestStore** - 결정론적 테스트 지원
5. **@BindableField** - 명시적이고 안전한 바인딩
6. **낮은 학습 곡선** - 기존 SwiftUI 지식을 그대로 활용

상태 관리가 복잡해지는 순간, InnoFlow를 고려해 보세요. 단방향 데이터 흐름이 주는 예측 가능성과 테스트 용이성을 경험하실 수 있습니다.

---

## 참고 자료

- [InnoFlow GitHub 저장소](https://github.com/InnoSquadCorp/InnoFlow)
- [InnoFlow API 문서 (DocC)](https://innosquad-mdd.github.io/InnoFlow/documentation/innoflow/)
- [Elm Architecture](https://guide.elm-lang.org/architecture/)
- [TCA (The Composable Architecture)](https://github.com/pointfreeco/swift-composable-architecture)
