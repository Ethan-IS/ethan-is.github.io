---
title: "InnoDI: Swift Macro 기반 타입 안전 의존성 주입 라이브러리"
date: 2026-02-23 00:00:00 +0900
lang: ko-KR
translation_key: innodi-swift-macro-di
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, dependency-injection, macro, di, clean-architecture]
author: ethan
toc: true
comments: true
---

## 들어가며

`InnoDI`를 처음 소개했을 때는 "Swift Macro로 DI 보일러플레이트를 줄여주는 라이브러리"라는 설명이 중심이었습니다. 그 설명도 여전히 틀리지는 않지만, `3.0.1` 기준의 실제 공개 surface를 보면 지금의 핵심은 **정적 dependency graph와 scope validation**입니다.

즉, `InnoDI`는 런타임 service locator가 아니라 **DI wiring을 명시적으로 선언하고, 잘못된 그래프를 컴파일/빌드 단계에서 최대한 빨리 막는 프레임워크**에 가깝습니다.

이 글은 그 기준으로 `InnoDI`를 다시 정리합니다.

## 왜 다시 봐야 하나

`3.0.x`에서 `InnoDI`의 중심은 새 매크로 문법이 아닙니다. 오히려 아래 같은 검증 경계가 더 또렷해졌습니다.

- strict name-based resolution
- declaration-order enforcement
- `concrete: true` opt-in
- `asyncFactory`의 scope 제약
- `@DIContainer`가 선언된 타입의 custom `init` 금지
- build-stage DAG validation과 plugin 기반 검증

그래서 지금 `InnoDI`를 설명할 때는 "매크로가 init을 생성해 준다"에서 멈추면 부족합니다. **어떤 wiring이 허용되고, 어떤 wiring이 거부되는지**까지 함께 설명해야 실제 코드베이스와 맞습니다.

## InnoDI가 소유하는 것과 소유하지 않는 것

README의 표현을 그대로 빌리면, `InnoDI`는 **static dependency graph and scope validation** 프레임워크입니다.

`InnoDI`가 소유하는 것:

- `@DIContainer`와 `@Provide`로 선언한 의존성 wiring
- `DIScope` 기반 수명주기 표현
- compile-time / build-time validation
- DAG 시각화와 validation artifact

`InnoDI`가 소유하지 않는 것:

- 런타임 상태 전이
- 화면 이동이나 navigation policy
- 네트워크 세션/transport lifecycle
- 컨테이너 해석을 이용한 runtime state machine

InnoSquad 스택 기준으로 보면 상태 전이는 `InnoFlow`, navigation은 `InnoRouter`, transport/session lifecycle은 `InnoNetwork` 쪽 책임입니다. `InnoDI`는 이 레이어들을 연결하는 **정적 wiring**에 집중합니다.

## 핵심 API

## 설치

`InnoDI 3.0.1` 기준 설치는 아래처럼 시작하면 됩니다.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoDI.git", from: "3.0.1")
]
```

타겟에는 보통 이렇게 연결합니다.

```swift
.target(
    name: "YourApp",
    dependencies: ["InnoDI"]
)
```

### `@DIContainer`

컨테이너 struct를 선언합니다.

```swift
@DIContainer(validate: true, root: false, validateDAG: true, mainActor: false)
struct AppContainer {}
```

핵심 포인트는 두 가지입니다.

- 매크로가 컨테이너 initializer와 accessor를 생성합니다.
- 동시에 잘못된 wiring을 컴파일 단계에서 거부합니다.

또 하나 중요한 제약이 있습니다. `@DIContainer`가 붙은 타입에는 user-defined `init`을 둘 수 없습니다. 타입 본문은 물론, same-file extension과 cross-file extension까지 build validation이 확인합니다.

### `@Provide`

의존성을 선언합니다.

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

선언 방식은 크게 세 가지입니다.

- `.input`으로 외부 입력을 받기
- `factory` / `asyncFactory`로 생성 규칙을 명시하기
- `Type.self` + `with:` 조합으로 AutoWiring 쓰기

### `DIScope`

`InnoDI`의 scope는 단순한 convenience가 아니라, validation 규칙과 직접 연결됩니다.

| Scope | 의미 | 비고 |
|------|------|------|
| `.input` | 컨테이너 생성 시 외부에서 반드시 주입 | factory 불가 |
| `.shared` | 컨테이너 수명 동안 1회 생성 후 재사용 | sync/async declaration order 규칙 적용 |
| `.transient` | 접근할 때마다 새 인스턴스 생성 | 캐시되지 않음 |

## 시작 예제

가장 기본적인 모습은 아래와 같습니다.

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

여기서 중요한 건 "자동 생성"보다 **엄격한 matching rule**입니다. `with:`에 넘긴 key path 이름은 concrete type의 init parameter 이름과 정확히 맞아야 합니다.

## AutoWiring과 strict matching

`InnoDI 3.x`의 핵심 변화 중 하나는 **name-based resolution을 더 엄격하게 다루는 것**입니다.

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

위 선언은 개념적으로 `APIClient(config:logger:)` 같은 init을 기대합니다. 이름이 다르면 자동으로 맞춰주지 않습니다. 이 제약은 번거롭지만, 선언을 읽는 사람과 매크로가 **같은 wiring 규칙**을 보게 만든다는 장점이 있습니다.

AutoWiring이 맞지 않는 경우는 factory closure를 쓰는 편이 낫습니다.

```swift
@Provide(.shared, factory: { (config: AppConfig) in
    APIClient(configuration: config, timeout: 30)
})
var apiClient: any APIClientProtocol
```

## Declaration Order와 scope 규칙

`PolicyBoundaries.md` 기준으로 `InnoDI`는 선언 순서를 validation contract에 포함합니다.

- `.input`은 항상 참조 가능합니다.
- sync `.shared`는 input과 **앞서 선언된** sync shared만 참조할 수 있습니다.
- async `.shared`는 input, 모든 sync shared, 그리고 **앞서 선언된** async shared를 참조할 수 있습니다.
- `.transient`는 어떤 멤버든 참조할 수 있지만 이름 matching은 여전히 엄격합니다.

즉, 컨테이너 멤버 순서가 단순 스타일이 아니라 **의존성 가용성 규칙**이 됩니다.

## `concrete: true`와 protocol-first 설계

`InnoDI`는 protocol-first 설계를 기본값으로 둡니다. 그래서 concrete shared/transient dependency를 직접 노출할 때는 opt-in이 필요합니다.

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

이 규칙 덕분에 concrete 타입 노출이 "실수로" 늘어나는 것을 막고, 의존성 역전 원칙을 의식적으로 지키게 됩니다.

## 비동기 팩토리와 제약

비동기 초기화가 필요하면 `asyncFactory`를 사용할 수 있습니다.

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

여기에도 명확한 제약이 있습니다.

- `factory`와 `asyncFactory`는 동시에 사용할 수 없습니다.
- `.input`에는 `asyncFactory`를 둘 수 없습니다.
- `asyncFactory`는 실제 `async` closure여야 합니다.

결국 `InnoDI`는 "지원한다"보다 "어디까지 허용한다"를 명확히 문서화한 프레임워크에 가깝습니다.

## Validation Layers

`InnoDI`의 진짜 가치가 드러나는 부분은 validation입니다.

### 1. Local Container Validation

매크로 단계에서 아래를 검사합니다.

- unknown scope
- missing factory
- `.input`의 잘못된 factory 구성
- strict name-based resolution
- declaration-order 위반
- `concrete: true` 누락
- local cycle / unknown dependency
- async factory validity
- same-file `init` 충돌

### 2. Build-Stage Validation

빌드 단계에서는 same-package 소스를 더 넓게 스캔해서 **cross-file extension `init` 충돌**까지 확인합니다.

즉, 매크로만 통과했다고 끝이 아니라 빌드 레벨에서 한 번 더 보강합니다.

### 3. Global DAG Validation

그래프 전체 차원의 순환과 ambiguity는 CLI나 build plugin으로 검증합니다.

```bash
swift run InnoDI-DependencyGraph --root . --validate-dag
```

필요하면 특정 컨테이너는 DAG 검증에서 제외할 수 있습니다.

```swift
@DIContainer(validateDAG: false)
struct PreviewContainer {
    @Provide(.input)
    var mockAPIClient: any APIClientProtocol
}
```

이 옵션은 "validation을 끄는 편의 기능"이라기보다, preview/test 전용 컨테이너처럼 **그래프 전역 검증에 참여시키고 싶지 않은 타입**을 분리하는 데 더 적합합니다.

## `PolicyBoundaries`와 custom `init` restriction

`3.0.x`를 이해하려면 README만으로는 부족하고, `PolicyBoundaries.md`를 같이 봐야 합니다. 핵심은 다음입니다.

- cross-file extension target은 semantic resolution을 먼저 시도합니다.
- ambiguous / unsupported case는 추측하지 않고 conservative fallback 또는 exclusion으로 처리합니다.
- generic argument extension, constrained `where` extension은 build-stage custom `init` 규칙에서 제외됩니다.
- nested path(`Outer.Container`) matching을 지원합니다.

이 선택은 "최대한 많이 맞추겠다"보다 **deterministic validation**을 우선하는 방향입니다.

## 테스트와 override 전략

`InnoDI`의 테스트 경험은 별도 override container를 만드는 방식보다, **생성된 initializer override parameter**를 활용하는 쪽이 중심입니다.

```swift
let container = AppContainer(
    baseURL: "https://test.example.com",
    apiClient: MockAPIClient()
)
```

이 방식의 장점은 두 가지입니다.

- 프로덕션 wiring과 테스트 wiring의 차이가 init parameter에서 바로 드러납니다.
- 별도의 runtime registry 없이 mock 교체가 가능합니다.

## CLI, Plugin, Artifact

실무에서 `InnoDI`를 팀 단위로 쓰게 되면 매크로 진단보다 **artifact와 plugin**이 중요해집니다.

`InnoDIDAGValidationPlugin`을 target에 붙이면 DAG 문제를 빌드 실패로 올릴 수 있습니다. 또한 validation 과정에서 아래 artifact가 생성됩니다.

- `result.json`
- `validation-metrics.json`
- `validation-summary.md`
- `dag-validation-stamp.txt`
- `dag-validation-metrics.json`
- `dag-validation-summary.md`

CI에서는 raw stderr만 보는 대신 Markdown summary와 metrics artifact를 우선 읽는 편이 훨씬 낫습니다.

## 언제 어떤 옵션을 써야 하나

| 상황 | 권장 선택 |
|------|-----------|
| 앱 시작 시 외부 값 주입 | `.input` |
| 컨테이너 단위로 재사용할 서비스 | `.shared` |
| 접근할 때마다 새 인스턴스가 필요한 ViewModel/Adapter | `.transient` + 필요 시 `concrete: true` |
| init parameter 이름이 프로퍼티 이름과 정확히 맞음 | `Type.self` + `with:` |
| 이름이 다르거나 변환 로직이 필요함 | `factory` |
| 비동기 생성 필요 | `asyncFactory` |
| preview/test 전용 컨테이너를 전역 DAG에서 분리 | `validateDAG: false` |

## 마무리

`InnoDI 3.0.1`을 한 문장으로 요약하면, "Swift Macro 기반 DI"보다 **정적 그래프 규칙이 명확한 DI wiring framework**가 더 정확한 설명입니다.

보일러플레이트를 줄여주는 건 여전히 장점입니다. 하지만 실제로 팀에서 가치를 만드는 지점은, wiring이 커질수록 **문제가 나중이 아니라 빌드 시점에 드러난다**는 데 있습니다.

문서를 더 보고 싶다면 아래 순서가 가장 좋습니다.

1. [README](https://github.com/InnoSquadCorp/InnoDI)
2. [`Validation.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/Validation.md)
3. [`PolicyBoundaries.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/PolicyBoundaries.md)
4. [`ModuleWideInitDetection.md`](https://github.com/InnoSquadCorp/InnoDI/blob/main/Sources/InnoDI/InnoDI.docc/ModuleWideInitDetection.md)
