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

## 왜 만들었을까요?

iOS에서 네트워킹 코드를 작성할 때마다 비슷한 고민이 반복됩니다.

**URLSession 코드가 장황합니다.** 요청 하나를 보내려면 URL을 만들고, URLRequest를 설정하고, dataTask를 만들고, response를 체크하고, JSON을 디코딩해야 합니다. 이 과정에서 같은 보일러플레이트를 반복해서 작성하게 됩니다.

**에러 처리가 일관되지 않습니다.** 404인지, 서버 에러인지, 디코딩 실패인지 구분하기 어렵고, 모든 것을 `catch { print(error) }`로 처리하는 코드가 쉽게 누적됩니다.

**타입 안전성이 부족합니다.** 요청 파라미터와 응답 타입이 명확하지 않아서 `Any`를 남발하거나 강제 캐스팅을 사용하는 코드가 늘어납니다.

**재시도 로직이 복잡합니다.** 네트워크가 끊겼을 때 재시도는 어떻게 할지, 지수 백오프는 어떻게 적용할지 같은 로직이 비즈니스 코드에 쉽게 섞입니다.

**InnoNetwork**는 이런 문제를 해결하기 위해 만들었습니다. 핵심 아이디어는 단순합니다.

> API를 타입으로 정의하고, Swift Concurrency로 깔끔하게 처리해 보겠습니다.

---

## 핵심 개념

### 세 가지 원칙

1. **타입 안전**: 모든 API 요청과 응답을 제네릭으로 강타입 처리합니다. 컴파일 타임에 타입 오류를 잡을 수 있습니다.
2. **프로토콜 기반**: `APIDefinition` 프로토콜로 API를 정의합니다. 기본 구현을 제공하면서도 커스터마이징이 자유롭습니다.
3. **Swift Concurrency**: async/await, actor, Sendable을 지원합니다. 콜백 지옥에서 벗어날 수 있습니다.

### 주요 컴포넌트

| 컴포넌트 | 역할 |
|---------|------|
| `APIDefinition` | API 요청을 정의하는 프로토콜 |
| `MultipartAPIDefinition` | 파일 업로드용 프로토콜 |
| `DefaultNetworkClient` | 네트워크 요청을 실행하는 actor |
| `NetworkConfiguration` | 타임아웃, 캐시 정책, 재시도 정책 등 설정 |
| `RequestInterceptor` | 요청 전 처리 (인증 토큰 추가 등) |
| `ResponseInterceptor` | 응답 후 처리 |
| `RetryPolicy` | 재시도 로직 정의 |

---

## 기본 사용법

### API 설정

먼저 API의 기본 설정을 정의한다.

```swift
import InnoNetwork

struct MyAPI: APIConfigure {
    var host: String { "https://api.example.com" }
    var basePath: String { "v1" }
}
```

`APIConfigure` 프로토콜을 따르면 된다. `baseURL`은 자동으로 계산된다.

### API 정의

각 API 엔드포인트를 타입으로 정의한다.

```swift
struct GetUsers: APIDefinition {
    typealias Parameter = EmptyParameter  // 요청 파라미터 없음
    typealias APIResponse = [User]       // 응답 타입

    var method: HTTPMethod { .get }
    var path: String { "/users" }
}

struct CreateUser: APIDefinition {
    struct UserParameter: Encodable, Sendable {
        let name: String
        let email: String
    }

    typealias Parameter = UserParameter
    typealias APIResponse = User

    var parameters: UserParameter?
    var method: HTTPMethod { .post }
    var path: String { "/users" }

    init(name: String, email: String) {
        self.parameters = UserParameter(name: name, email: email)
    }
}
```

`APIDefinition` 프로토콜만 따르면 된다. 나머지는 기본 구현이 제공된다.

### 요청 실행

```swift
let client = try DefaultNetworkClient(configuration: MyAPI())

// GET 요청
let users = try await client.request(GetUsers())

// POST 요청
let newUser = try await client.request(CreateUser(name: "홍길동", email: "hong@example.com"))
```

단 세 줄이다. URLSession을 직접 쓰면 최소 20줄은 될 코드다.

---

## HTTP 메서드

모든 표준 HTTP 메서드를 지원한다.

```swift
struct GetPost: APIDefinition {
    typealias Parameter = EmptyParameter
    typealias APIResponse = Post

    let postId: Int
    var method: HTTPMethod { .get }
    var path: String { "/posts/\(postId)" }
}

struct UpdatePost: APIDefinition {
    struct PostParameter: Encodable, Sendable {
        let title: String
        let body: String
    }

    typealias Parameter = PostParameter
    typealias APIResponse = Post

    var parameters: PostParameter?
    var method: HTTPMethod { .put }
    var path: String { "/posts/1" }
}

struct PatchPost: APIDefinition {
    struct PatchParameter: Encodable, Sendable {
        let title: String?
    }

    typealias Parameter = PatchParameter
    typealias APIResponse = Post

    var parameters: PatchParameter?
    var method: HTTPMethod { .patch }
    var path: String { "/posts/1" }
}

struct DeletePost: APIDefinition {
    typealias Parameter = EmptyParameter
    typealias APIResponse = EmptyResponse  // 응답 본문 없음

    let postId: Int
    var method: HTTPMethod { .delete }
    var path: String { "/posts/\(postId)" }
}
```

---

## 커스텀 헤더

인증, 컨텐츠 타입 등 헤더 커스터마이징을 쉽게 적용할 수 있습니다.

```swift
struct GetPrivateData: APIDefinition {
    typealias Parameter = EmptyParameter
    typealias APIResponse = PrivateData

    var method: HTTPMethod { .get }
    var path: String { "/private" }

    var headers: HTTPHeaders {
        var headers = HTTPHeaders.default
        headers.add(.authorization(bearerToken: "my-jwt-token"))
        headers.add(.accept("application/json"))
        headers.add(name: "X-API-Version", value: "2")
        return headers
    }
}
```

### 기본 헤더

```swift
HTTPHeaders.default  // Content-Type, Accept 등 기본 헤더
```

### 사용 가능한 헤더 타입

```swift
.authorization(username: "user", password: "pass")  // Basic Auth
.authorization(bearerToken: "token")                // Bearer Token
.contentType("application/json")
.accept("application/json")
.acceptLanguage("ko-KR")
.userAgent("MyApp/1.0")
HTTPHeader(name: "X-Custom-Header", value: "value")
```

---

## 컨텐츠 타입

JSON 외에도 다양한 컨텐츠 타입을 지원한다.

### JSON (기본)

```swift
struct CreatePost: APIDefinition {
    var contentType: ContentType { .json }  // 기본값
    // ...
}
```

### Form URL-Encoded

```swift
struct LoginRequest: APIDefinition {
    struct LoginParameter: Encodable, Sendable {
        let email: String
        let password: String
    }

    typealias Parameter = LoginParameter
    typealias APIResponse = AuthResponse

    var parameters: LoginParameter?
    var method: HTTPMethod { .post }
    var path: String { "/login" }
    var contentType: ContentType { .formUrlEncoded }  // x-www-form-urlencoded

    init(email: String, password: String) {
        self.parameters = LoginParameter(email: email, password: password)
    }
}
```

---

## 파일 업로드 (Multipart)

파일 업로드는 `MultipartAPIDefinition`을 사용합니다.

```swift
struct UploadImage: MultipartAPIDefinition {
    typealias APIResponse = UploadResponse

    let imageData: Data
    let title: String

    var multipartFormData: MultipartFormData {
        var formData = MultipartFormData()
        formData.append(title, name: "title")
        formData.append(
            imageData,
            name: "file",
            fileName: "image.jpg",
            mimeType: "image/jpeg"
        )
        return formData
    }

    var method: HTTPMethod { .post }
    var path: String { "/upload" }
}

// 사용
let response = try await client.upload(UploadImage(imageData: data, title: "프로필 사진"))
```

`upload` 메서드를 사용하면 됩니다.

---

## 에러 처리

`NetworkError` 열거형으로 모든 네트워크 에러를 체계적으로 처리한다.

```swift
do {
    let user = try await client.request(GetUser(id: 1))
    print("사용자: \(user.name)")
} catch let error as NetworkError {
    switch error {
    case .statusCode(let response):
        // HTTP 상태 코드 에러 (404, 500 등)
        print("HTTP 에러: \(response.statusCode)")
        if response.statusCode == 401 {
            // 인증 만료 처리
        }
    case .objectMapping(let decodingError, let response):
        // JSON 디코딩 실패
        print("디코딩 실패: \(decodingError.message)")
    case .invalidBaseURL(let url):
        print("잘못된 URL: \(url)")
    case .underlying(let underlyingError, _):
        // 네트워크 연결 실패 등
        print("기타 에러: \(underlyingError.message)")
    case .cancelled:
        // 요청 취소
        print("요청이 취소됨")
    default:
        print("알 수 없는 에러: \(error)")
    }
}
```

### NetworkError 종류

| 케이스 | 의미 |
|-------|------|
| `invalidBaseURL` | 잘못된 베이스 URL |
| `invalidRequestConfiguration` | 잘못된 요청 설정 |
| `statusCode` | HTTP 상태 코드 에러 (200-299 외) |
| `objectMapping` | JSON 디코딩 실패 |
| `jsonMapping` | JSON 파싱 실패 |
| `underlying` | 기타 에러 |
| `trustEvaluationFailed` | SSL 인증서 검증 실패 |
| `cancelled` | 요청 취소 |

---

## 재시도 정책

네트워크 실패 시 자동 재시도를 지원한다.

```swift
let retryPolicy = ExponentialBackoffRetryPolicy(
    maxRetries: 3,              // 최대 재시도 횟수
    retryDelay: 1.0,            // 기본 대기 시간 (초)
    maxDelay: 30.0,             // 최대 대기 시간
    jitterRatio: 0.2,           // 지터 (±20%)
    waitsForNetworkChanges: true, // 네트워크 변경 대기
    networkChangeTimeout: 10.0   // 네트워크 변경 대기 최대 시간
)

let config = NetworkConfiguration(
    baseURL: URL(string: "https://api.example.com/v1")!,
    retryPolicy: retryPolicy
)

let client = try DefaultNetworkClient(
    configuration: MyAPI(),
    networkConfiguration: config
)
```

### 지수 백오프

`ExponentialBackoffRetryPolicy`는 지수 백오프를 자동으로 적용한다.

- 1차 재시도: ~1초 대기
- 2차 재시도: ~2초 대기
- 3차 재시도: ~4초 대기

지터(jitter)를 추가해서 서버 부하를 분산시킨다.

### 재시도 조건

기본적으로 다음 경우에 재시도한다:

- 408 Request Timeout
- 429 Too Many Requests
- 5xx 서버 에러
- 네트워크 연결 실패

---

## 인터셉터

요청/응답을 가로채서 처리할 수 있다.

### RequestInterceptor

요청 보내기 전에 헤더를 추가하거나 수정한다.

```swift
struct AuthInterceptor: RequestInterceptor {
    let tokenProvider: () -> String?

    func adapt(_ urlRequest: URLRequest) async throws -> URLRequest {
        var request = urlRequest
        if let token = tokenProvider() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        return request
    }
}

struct RefreshTokenAPI: APIDefinition {
    // ...
    var requestInterceptors: [RequestInterceptor] {
        [AuthInterceptor(tokenProvider: { TokenStorage.shared.accessToken })]
    }
}
```

### ResponseInterceptor

응답 받은 후 데이터를 가공한다.

```swift
struct ErrorResponseInterceptor: ResponseInterceptor {
    func adapt(_ urlResponse: Response, request: URLRequest) async throws -> Response {
        // 에러 응답에서 메시지 추출 등
        if urlResponse.statusCode >= 400 {
            // 에러 로깅
        }
        return urlResponse
    }
}
```

---

## 네트워크 모니터링

네트워크 상태 변화를 감지할 수 있다.

```swift
let monitor = NetworkMonitor.shared

// 현재 상태 확인
let snapshot = await monitor.currentSnapshot()
print("연결됨: \(snapshot.status == .satisfied)")
print("인터페이스: \(snapshot.interfaceTypes)")

// 네트워크 변경 대기
let newSnapshot = await monitor.waitForChange(from: snapshot, timeout: 30.0)
```

### NetworkSnapshot

```swift
struct NetworkSnapshot: Sendable {
    let status: NetworkReachabilityStatus      // .satisfied, .unsatisfied, .requiresConnection
    let interfaceTypes: Set<NetworkInterfaceType>  // .wifi, .cellular, .wiredEthernet 등
}
```

---

## 이벤트 옵저버

네트워크 이벤트를 관찰할 수 있다.

```swift
struct LoggingObserver: NetworkEventObserving {
    func handle(_ event: NetworkEvent) {
        switch event {
        case .requestStart(let requestID, let method, let url, let retryIndex):
            print("[\(requestID)] \(method) \(url)")
        case .responseReceived(let requestID, let statusCode, let byteCount):
            print("[\(requestID)] 응답: \(statusCode), \(byteCount) bytes")
        case .requestFinished(let requestID, let statusCode, let byteCount):
            print("[\(requestID)] 완료: \(statusCode)")
        case .requestFailed(let requestID, let errorCode, let message):
            print("[\(requestID)] 실패: \(message)")
        case .retryScheduled(let requestID, let retryIndex, let delay, let reason):
            print("[\(requestID)] \(retryIndex + 1)차 재시도 예정 (\(delay)초 후)")
        default:
            break
        }
    }
}

let config = NetworkConfiguration(
    baseURL: URL(string: "https://api.example.com")!,
    eventObservers: [LoggingObserver()]
)
```

### NetworkEvent 종류

| 이벤트 | 시점 |
|-------|------|
| `requestStart` | 요청 시작 |
| `requestAdapted` | 인터셉터 적용 후 |
| `responseReceived` | 응답 수신 |
| `requestFinished` | 요청 완료 |
| `requestFailed` | 요청 실패 |
| `retryScheduled` | 재시도 예약 |

---

## Protobuf 지원

Protocol Buffers도 지원한다.

```swift
import SwiftProtobuf

struct GetProtobufData: ProtobufAPIDefinition {
    typealias Parameter = MyRequestProto  // Protobuf 메시지
    typealias APIResponse = MyResponseProto

    var parameters: MyRequestProto?
    var method: HTTPMethod { .post }
    var path: String { "/data" }
}

let response = try await client.protobufRequest(GetProtobufData(...))
```

---

## 실제 아키텍처 예시

### Repository 레이어와 함께

```swift
// Domain
struct User: Decodable, Sendable {
    let id: Int
    let name: String
    let email: String
}

protocol UserRepository {
    func fetchUsers() async throws -> [User]
    func createUser(name: String, email: String) async throws -> User
}

// Data
final class UserRepositoryImpl: UserRepository {
    private let client: DefaultNetworkClient

    init(client: DefaultNetworkClient) {
        self.client = client
    }

    func fetchUsers() async throws -> [User] {
        try await client.request(GetUsers())
    }

    func createUser(name: String, email: String) async throws -> User {
        try await client.request(CreateUser(name: name, email: email))
    }
}

// App
@main
struct MyApp: App {
    let client: DefaultNetworkClient
    let userRepository: UserRepository

    init() {
        do {
            self.client = try DefaultNetworkClient(configuration: MyAPI())
        } catch {
            fatalError("Failed to create DefaultNetworkClient: \(error)")
        }
        self.userRepository = UserRepositoryImpl(client: client)
    }

    var body: some Scene {
        WindowGroup {
            UserListView(repository: userRepository)
        }
    }
}
```

API 정의와 비즈니스 로직이 깔끔하게 분리된다.

---

## 다운로드 (InnoNetworkDownload)

대용량 파일 다운로드는 별도 모듈을 사용한다.

```swift
import InnoNetworkDownload

let manager = DownloadManager.shared

// 다운로드 시작
let task = await manager.download(
    url: URL(string: "https://example.com/large-file.zip")!,
    toDirectory: documentsDirectory
)

// 진행률 모니터링 (AsyncSequence)
for await event in manager.events(for: task) {
    switch event {
    case .progress(let progress):
        print("진행률: \(progress.percentCompleted)%")
        print("받은 바이트: \(progress.bytesReceived)")
    case .completed(let url):
        print("완료: \(url)")
    case .failed(let error):
        print("실패: \(error)")
    case .paused:
        print("일시정지")
    case .resumed:
        print("재개")
    }
}

// 일시정지/재개
await manager.pause(task)
await manager.resume(task)
```

백그라운드 다운로드와 재개 가능한 다운로드를 지원한다.

---

## InnoNetwork가 해결하는 문제들

| 문제 | 해결 방법 |
|------|----------|
| URLSession 장황한 코드 | `APIDefinition` 프로토콜로 간결화 |
| 타입 안전성 부족 | 제네릭으로 요청/응답 강타입 |
| 에러 처리 불명확 | `NetworkError` 열거형으로 체계화 |
| 재시도 로직 복잡 | `RetryPolicy` 프로토콜로 추상화 |
| 콜백 지옥 | async/await 기반 |
| 인터셉터 구현 번거로움 | `RequestInterceptor`, `ResponseInterceptor` 프로토콜 |
| 네트워크 상태 추적 어려움 | `NetworkMonitor`, 이벤트 옵저버 |

---

## 요구사항

- iOS 26+ / macOS 14+ / tvOS 26+ / watchOS 26+ / visionOS 26+
- Swift 6.2+

Swift 6의 strict concurrency를 지원합니다. `DefaultNetworkClient`는 `actor`로 구현되어 있고, 주요 타입이 `Sendable`을 준수합니다.

---

## 결론

InnoNetwork는 **타입 안전하고 Swift Concurrency 네이티브한 네트워킹 프레임워크**입니다.

### 추천하는 경우

- API 요청 코드를 깔끔하게 정리하고 싶을 때
- 타입 안전성을 보장받고 싶을 때
- 재시도, 인터셉터 등을 구조적으로 처리하고 싶을 때
- Swift 6 strict concurrency 환경에서 작업할 때

### 장점 요약

1. **타입 안전** - 제네릭으로 요청/응답 강타입
2. **간결한 API** - 프로토콜 기반 정의, 기본 구현 제공
3. **Swift Concurrency** - async/await, actor 완벽 지원
4. **재시도 정책** - 지수 백오프, 네트워크 변경 감지
5. **인터셉터** - 요청/응답 가로채기
6. **이벤트 관찰** - 로깅, 분석, 모니터링
7. **Swift 6 준비됨** - Sendable, actor 완벽 지원

---

## 참고 자료

- [InnoNetwork GitHub 저장소](https://github.com/InnoSquadCorp/InnoNetwork)
- [Swift Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
- [URLSession](https://developer.apple.com/documentation/foundation/urlsession)
