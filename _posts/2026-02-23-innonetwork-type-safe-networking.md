---
title: "InnoNetwork: Swift Concurrency를 위한 타입 안전 네트워킹 프레임워크"
date: 2026-02-23 00:00:00 +0900
translation_key: innonetwork-type-safe-networking
lang: ko-KR
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, networking, async-await, swift-concurrency, clean-architecture, type-safe]
author: ethan
toc: true
comments: true
---

## 들어가며

초기 버전의 `InnoNetwork`를 설명할 때는 보통 "Swift Concurrency 기반 타입 안전 HTTP 클라이언트"라는 표현을 썼습니다. 지금도 틀린 말은 아니지만, `3.0.1` 기준으로는 그 설명만으로 부족합니다.

현재 공개 패키지는 아래 세 제품으로 나뉩니다.

- `InnoNetwork`
- `InnoNetworkDownload`
- `InnoNetworkWebSocket`

그리고 실제 public contract는 단순 HTTP wrapper가 아니라, **명시적인 configuration entry point, transport policy, event delivery, durability, reconnect model**까지 포함합니다.

이 글은 그 기준으로 `InnoNetwork`를 다시 정리합니다.

## 왜 다시 봐야 하나

`3.0.x`에서 가장 중요한 변화는 "더 많은 endpoint 예제"가 아닙니다. 대신 아래 관점이 또렷해졌습니다.

- `safeDefaults`가 권장 진입점이라는 점
- `advanced` builder로 운영 정책을 국소적으로 조정한다는 점
- 다운로드와 웹소켓이 별도 product로 분리되어 있다는 점
- protobuf가 별도 패키지 `InnoNetworkProtobuf`로 이동했다는 점
- bounded buffering, append-log durability, reconnect taxonomy 같은 운영 모델이 문서화되었다는 점

즉, 지금 `InnoNetwork`는 "HTTP 요청 보내는 법"보다 **어떤 네트워크 lifecycle을 어떤 surface로 다룰 것인지**가 더 중요한 프레임워크입니다.

## 제품군 구조와 ownership boundary

먼저 제품 경계를 명확히 보는 게 좋습니다.

### `InnoNetwork`

기본 request/response API를 담당합니다.

- `APIDefinition`
- `MultipartAPIDefinition`
- `DefaultNetworkClient`
- `NetworkConfiguration`
- `TrustPolicy`
- `RetryPolicy`

### `InnoNetworkDownload`

다운로드 lifecycle을 담당합니다.

- `DownloadConfiguration`
- `DownloadManager`
- progress / completion / failure event stream
- foreground / background orchestration

### `InnoNetworkWebSocket`

연결 지향 realtime flow를 담당합니다.

- `WebSocketConfiguration`
- `WebSocketManager`
- reconnect policy
- heartbeat / pong timeout
- event stream

이 구조 덕분에 `InnoNetwork`는 "모든 네트워크 문제를 하나의 클라이언트에 우겨 넣는 패키지"가 아니라, transport 성격에 따라 surface를 분리한 프레임워크 제품군이 됩니다.

## 설치

기본 설치는 `3.0.1` 기준으로 아래처럼 시작합니다.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", from: "3.0.1")
]
```

## Core Request Model

기본 요청 모델은 여전히 `APIDefinition`입니다.

```swift
import Foundation
import InnoNetwork

struct User: Decodable, Sendable {
    let id: Int
    let name: String
}

struct GetUser: APIDefinition {
    typealias Parameter = EmptyParameter
    typealias APIResponse = User

    var method: HTTPMethod { .get }
    var path: String { "/users/1" }
}
```

실행은 `DefaultNetworkClient`가 맡습니다.

```swift
let client = DefaultNetworkClient(
    configuration: .safeDefaults(
        baseURL: URL(string: "https://api.example.com/v1")!
    )
)

let user = try await client.request(GetUser())
print(user.name)
```

여기서 중요한 건 예전처럼 별도 `APIConfigure` 타입을 기본 설정 entry point로 두지 않는다는 점입니다. `3.0.x`에서는 **configuration object를 명시적으로 생성하는 방식**이 문서와 예제의 중심입니다.

## `safeDefaults`와 `advanced`

지금 문서에서 가장 강조하는 메시지는 간단합니다.

> 특별한 운영 요구가 없다면 `safeDefaults`에 머문다.

### 권장 시작점

```swift
let client = DefaultNetworkClient(
    configuration: NetworkConfiguration.safeDefaults(
        baseURL: URL(string: "https://api.example.com")!
    )
)
```

### 운영 정책이 필요할 때만 `advanced`

```swift
let configuration = NetworkConfiguration.advanced(
    baseURL: URL(string: "https://api.example.com")!
) { builder in
    builder.timeout = 30
    builder.retryPolicy = ExponentialBackoffRetryPolicy()
    builder.trustPolicy = .systemDefault
}

let client = DefaultNetworkClient(configuration: configuration)
```

이 구분이 중요한 이유는, `advanced`는 public API이긴 하지만 숫자나 세부 튜닝 값 자체를 contract로 고정하는 surface는 아니기 때문입니다. 프레임워크가 권장하는 경로는 여전히 `safeDefaults`입니다.

## Transport / Operational Surface

`InnoNetwork`를 `3.x` 기준으로 설명할 때는 request/response shape만이 아니라 운영 surface를 같이 봐야 합니다.

### `RetryPolicy`

재시도는 business code에 흩어지는 로직이 아니라 policy로 분리됩니다.

- `RetryPolicy`
- `ExponentialBackoffRetryPolicy`

### `TrustPolicy`

신뢰 평가도 명시적인 surface로 노출됩니다.

- `.systemDefault`
- public key pinning

### `EventDeliveryPolicy`

이 부분이 특히 `3.x`다운 변화입니다. 이벤트 전달은 무한 버퍼에 기대지 않고, **bounded buffering과 overflow policy**를 가진 별도 정책으로 다뤄집니다.

즉, 운영 중에 event stream이 과도하게 밀릴 때 어떤 식으로 처리할지를 public surface에서 조절할 수 있습니다.

### `URLQueryEncoder`와 `AnyResponseDecoder`

문서의 톤을 보면 query/form encoding과 decoding도 점점 더 명시적으로 다뤄집니다.

- `URLQueryEncoder`는 deterministic ordering을 가진 query / form 인코딩 경로를 담당합니다.
- `AnyResponseDecoder`는 explicit decoding strategy를 구성하는 public surface 중 하나입니다.

둘 다 "일상적으로 직접 만지는 타입"은 아닐 수 있지만, `3.x` public contract를 설명할 때는 존재를 짚고 넘어가는 편이 맞습니다.

## Interceptor와 request/response policy

기존 글에서는 `RequestInterceptor`, `ResponseInterceptor`, `RetryPolicy`를 프레임워크의 핵심 전부처럼 설명했습니다. `3.x`에서는 톤을 조금 바꾸는 편이 맞습니다.

이 타입들은 여전히 중요합니다.

- `RequestInterceptor`
- `ResponseInterceptor`
- `RetryPolicy`

하지만 프레임워크의 진짜 중심은 "인터셉터를 몇 개 붙일 수 있다"가 아니라, **request execution이 명시적인 정책 경계 안에서 실행된다**는 데 있습니다.

README와 changelog를 보면 request/response execution은 internal transport policy와 explicit decoding strategy 쪽으로 재정리되어 있습니다. 이 레이어는 구현의 핵심이지만, 블로그에서는 **public contract가 아닌 운영 모델 설명** 정도로만 다루는 편이 정확합니다.

## Download Lifecycle

다운로드는 `InnoNetworkDownload` product가 담당합니다.

```swift
import Foundation
import InnoNetworkDownload

let manager = DownloadManager.shared
let task = await manager.download(
    url: URL(string: "https://example.com/file.zip")!,
    toDirectory: FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
)

for await event in await manager.events(for: task) {
    print(event)
}
```

이 모듈의 핵심은 단순 파일 다운로드가 아닙니다.

- foreground / background orchestration
- pause / resume / retry
- listener retention
- append-log persistence 기반 durability

즉, 다운로드를 "URLSession downloadTask 래퍼"가 아니라 **지속성과 이벤트 전달이 있는 lifecycle manager**로 보는 편이 맞습니다.

필요하면 configuration도 명시적으로 생성할 수 있습니다.

```swift
let configuration = DownloadConfiguration.advanced { builder in
    builder.maxConnectionsPerHost = 3
    builder.maxRetryCount = 2
}
```

새 코드에서는 `safeDefaults()`가 기본, `advanced`가 조정 경로라는 패턴이 여기서도 반복됩니다.

## WebSocket Lifecycle

웹소켓은 `InnoNetworkWebSocket`이 맡습니다.

```swift
import Foundation
import InnoNetworkWebSocket

let task = await WebSocketManager.shared.connect(
    url: URL(string: "wss://echo.example.com/socket")!
)

for await event in await WebSocketManager.shared.events(for: task) {
    print(event)
}
```

이 product의 핵심은 단순 connect/send/receive API보다 아래 운영 모델입니다.

- reconnect policy
- handshake-aware close taxonomy
- reconnect suppression rules
- heartbeat / pong timeout
- event delivery policy

그래서 `WebSocketManager`를 설명할 때는 "실시간 통신을 지원한다"보다 **연결 실패와 재연결을 어떤 규칙으로 관리하는지**가 더 중요합니다.

## Error Handling

`InnoNetwork`는 opaque error보다 명시적인 transport error를 선호합니다.

```swift
do {
    let user = try await client.request(GetUser())
    print(user)
} catch let error as NetworkError {
    switch error {
    case .invalidBaseURL(let url):
        print("Invalid base URL: \(url)")
    case .invalidRequestConfiguration(let message):
        print("Invalid request configuration: \(message)")
    case .statusCode(let response):
        print("Unexpected status code: \(response.statusCode)")
    case .objectMapping(let underlying, _):
        print("Decoding failed: \(underlying)")
    case .trustEvaluationFailed(let reason):
        print("Trust evaluation failed: \(reason)")
    case .cancelled:
        print("Request cancelled")
    default:
        print(error)
    }
}
```

특히 `invalidRequestConfiguration`은 단순 "뭔가 실패했다"가 아니라, request shape와 policy가 맞지 않는다는 신호에 가깝습니다. 예를 들어 query 인코딩 규칙이나 multipart payload shape가 잘못됐을 때 이런 에러가 나올 수 있습니다.

## Protocol Buffers는 이제 별도 패키지

이전 글에서는 protobuf 지원을 본 패키지의 확장 기능처럼 다뤘지만, `3.0.1` 기준으로는 **`InnoNetworkProtobuf`가 별도 패키지**입니다.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", from: "3.0.1"),
    .package(url: "https://github.com/InnoSquadCorp/InnoNetworkProtobuf.git", branch: "main")
]
```

즉, protobuf가 필요한 앱은 `InnoNetwork`만 추가하면 끝이 아닙니다. 이 분리는 "기능이 사라졌다"기보다, core transport 패키지와 protobuf adapter layer를 더 명확히 분리한 것으로 보는 편이 맞습니다.

## 언제 무엇을 써야 하나

| 상황 | 권장 선택 |
|------|-----------|
| 일반적인 REST/HTTP API 요청 | `InnoNetwork` + `NetworkConfiguration.safeDefaults(baseURL:)` |
| retry / trust / metrics 같은 운영 정책 조정 필요 | `NetworkConfiguration.advanced(baseURL:_:)` |
| 파일 다운로드, 앱 생명주기와 연결된 progress 추적 | `InnoNetworkDownload` + `DownloadManager` |
| reconnect와 heartbeat가 중요한 실시간 연결 | `InnoNetworkWebSocket` + `WebSocketManager` |
| protobuf request/response 모델 필요 | `InnoNetwork` + `InnoNetworkProtobuf` |

## 마무리

`InnoNetwork 3.0.1`을 한 문장으로 정리하면, "타입 안전 HTTP 클라이언트"보다는 **운영 정책이 명시된 네트워크 프레임워크 제품군**이 더 정확한 설명입니다.

`APIDefinition`과 `DefaultNetworkClient`는 여전히 출발점입니다. 하지만 실제로 `3.x`를 잘 쓰려면 다음 네 가지를 같이 이해해야 합니다.

- `safeDefaults`가 기본 경로라는 점
- `advanced`는 운영 정책이 필요할 때만 쓴다는 점
- download / websocket lifecycle이 별도 product라는 점
- protobuf가 별도 패키지로 분리되었다는 점

더 자세히 보려면 아래 문서부터 읽는 걸 추천합니다.

1. [README](https://github.com/InnoSquadCorp/InnoNetwork)
2. [`API_STABILITY.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/API_STABILITY.md)
3. [`Examples/README.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/Examples/README.md)
4. [`docs/releases/3.0.1.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/docs/releases/3.0.1.md)
