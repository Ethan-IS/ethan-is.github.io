---
title: "InnoNetwork: A type-safe networking framework for Swift Concurrency"
date: 2026-02-23 00:00:00 +0900
translation_key: innonetwork-type-safe-networking
lang: en
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, networking, async-await, swift-concurrency, clean-architecture, type-safe]
author: ethan
toc: true
comments: true
---

## Why was it made?


Similar concerns arise over and over again every time I write networking code on iOS.

**URLSession code is verbose.** To send a request, you need to create a URL, set up a URLRequest, create a dataTask, check the response, and decode the JSON. In this process, you'll be writing the same boilerplate over and over again.

**Error handling is inconsistent.** It's hard to tell if it's a 404, a server error, or a decoding failure, and code that treats everything as `catch { print(error) }` easily accumulates.

**Lack of type safety.** Request parameters and response types are unclear, resulting in more code that overuses `Any` or uses forced casting.

**Retry logic is complex.** Logic such as how to retry when the network is disconnected and how to apply exponential backoff can be easily mixed into business code.

**InnoNetwork** was created to solve this problem. The core idea is simple.

> Let’s define the API as a type and handle it neatly with Swift Concurrency.


---

## Core Concepts


### Three Principles


1. **Type Safety**: All API requests and responses are generic and strongly typed. You can catch type errors at compile time.

2. **Protocol-based**: Defines the API with the `APIDefinition` protocol. While providing a basic implementation, customization is free.

3. **Swift Concurrency**: Supports async/await, actor, and Sendable. You can escape callback hell.


### Main Components


| component | role |
|---------|------|
| `APIDefinition` | Protocol that defines API requests |
| `MultipartAPIDefinition` | Protocol for file upload |
| `DefaultNetworkClient` | actor executing network requests |
| `NetworkConfiguration` | Setting timeout, cache policy, retry policy, etc. |
| `RequestInterceptor` | Pre-processing of requests (adding authentication tokens, etc.) |
| `ResponseInterceptor` | Post-response processing |
| `RetryPolicy` | Define retry logic |

---

## Basic usage


### API settings


First, define the basic settings of the API.

```swift
import InnoNetwork

struct MyAPI: APIConfigure {
    var host: String { "https://api.example.com" }
    var basePath: String { "v1" }
}
```

Just follow the `APIConfigure` protocol. `baseURL` is calculated automatically.

### API Definition


Each API endpoint is defined as a type.

```swift
struct GetUsers: APIDefinition {
    typealias Parameter = EmptyParameter  // no request parameters
    typealias APIResponse = [User]       // response type

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

Just follow the `APIDefinition` protocol. For the rest, default implementations are provided.

### execute request


```swift
let client = try DefaultNetworkClient(configuration: MyAPI())

// GET request
let users = try await client.request(GetUsers())

// POST request
let newUser = try await client.request(CreateUser(name: "John Doe", email: "john@example.com"))
```

It's just three lines. Using `URLSession` directly usually takes 20+ lines.

---

## HTTP method


All standard HTTP methods are supported.

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
    typealias APIResponse = EmptyResponse  // no response body

    let postId: Int
    var method: HTTPMethod { .delete }
    var path: String { "/posts/\(postId)" }
}
```

---

## custom header


Header customization such as authentication and content type can be easily applied.

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

### default header


```swift
HTTPHeaders.default  // default headers (Content-Type, Accept, etc.)
```

### Available header types


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

## Content type


In addition to JSON, it supports various content types.

### JSON (default)


```swift
struct CreatePost: APIDefinition {
    var contentType: ContentType { .json }  // default
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

## File Upload (Multipart)


File upload uses `MultipartAPIDefinition`.

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

// usage
let response = try await client.upload(UploadImage(imageData: data, title: "Profile Photo"))
```

You can do this by using the `upload` method.

---

## Error handling


All network errors are systematically handled with the `NetworkError` enumeration.

```swift
do {
    let user = try await client.request(GetUser(id: 1))
    print("User: \(user.name)")
} catch let error as NetworkError {
    switch error {
    case .statusCode(let response):
        // HTTP status errors (404, 500, etc.)
        print("HTTP error: \(response.statusCode)")
        if response.statusCode == 401 {
            // handle expired authentication
        }
    case .objectMapping(let decodingError, let response):
        // JSON decoding failure
        print("Decoding failed: \(decodingError.message)")
    case .invalidBaseURL(let url):
        print("Invalid URL: \(url)")
    case .underlying(let underlyingError, _):
        // network connectivity failure, etc.
        print("Other error: \(underlyingError.message)")
    case .cancelled:
        // request cancelled
        print("Request cancelled")
    default:
        print("Unknown error: \(error)")
    }
}
```

### NetworkError type


| case | meaning |
|-------|------|
| `invalidBaseURL` | Invalid base URL |
| `invalidRequestConfiguration` | Invalid request settings |
| `statusCode` | HTTP status code errors (other than 200-299) |
| `objectMapping` | JSON decoding failed |
| `jsonMapping` | JSON parsing failed |
| `underlying` | Other errors |
| `trustEvaluationFailed` | SSL certificate verification failed |
| `cancelled` | Cancel request |

---

## Retry Policy


Supports automatic retry in case of network failure.

```swift
let retryPolicy = ExponentialBackoffRetryPolicy(
    maxRetries: 3,              // maximum retry count
    retryDelay: 1.0,            // base delay (seconds)
    maxDelay: 30.0,             // maximum delay
    jitterRatio: 0.2,           // jitter (±20%)
    waitsForNetworkChanges: true, // wait for network changes
    networkChangeTimeout: 10.0   // max wait for network change
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

### Exponential backoff


`ExponentialBackoffRetryPolicy` automatically applies exponential backoff.

- 1st retry: wait ~1 second

- 2nd retry: wait ~2 seconds

- 3rd retry: wait ~4 seconds


Add jitter to distribute server load.

### Retry conditions


By default, it retries in the following cases:

- 408 Request Timeout

- 429 Too Many Requests

- 5xx server error

- Network connection failure


---

## interceptor


Requests/responses can be intercepted and processed.

### RequestInterceptor


Add or modify headers before sending the request.

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


After receiving the response, the data is processed.

```swift
struct ErrorResponseInterceptor: ResponseInterceptor {
    func adapt(_ urlResponse: Response, request: URLRequest) async throws -> Response {
        // parse message from error responses
        if urlResponse.statusCode >= 400 {
            // error logging
        }
        return urlResponse
    }
}
```

---

## network monitoring


Changes in network status can be detected.

```swift
let monitor = NetworkMonitor.shared

// inspect current status
let snapshot = await monitor.currentSnapshot()
print("Connected: \(snapshot.status == .satisfied)")
print("Interfaces: \(snapshot.interfaceTypes)")

// wait for network changes
let newSnapshot = await monitor.waitForChange(from: snapshot, timeout: 30.0)
```

### NetworkSnapshot


```swift
struct NetworkSnapshot: Sendable {
    let status: NetworkReachabilityStatus      // .satisfied, .unsatisfied, .requiresConnection
    let interfaceTypes: Set<NetworkInterfaceType>  // .wifi, .cellular, .wiredEthernet, etc.
}
```

---

## event observer


You can observe network events.

```swift
struct LoggingObserver: NetworkEventObserving {
    func handle(_ event: NetworkEvent) {
        switch event {
        case .requestStart(let requestID, let method, let url, let retryIndex):
            print("[\(requestID)] \(method) \(url)")
        case .responseReceived(let requestID, let statusCode, let byteCount):
            print("[\(requestID)] Response: \(statusCode), \(byteCount) bytes")
        case .requestFinished(let requestID, let statusCode, let byteCount):
            print("[\(requestID)] Completed: \(statusCode)")
        case .requestFailed(let requestID, let errorCode, let message):
            print("[\(requestID)] Failed: \(message)")
        case .retryScheduled(let requestID, let retryIndex, let delay, let reason):
            print("[\(requestID)] Retry \(retryIndex + 1) scheduled in \(delay)s")
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

### NetworkEvent Type


| event | point of view |
|-------|------|
| `requestStart` | Start request |
| `requestAdapted` | After applying the interceptor |
| `responseReceived` | Receive response |
| `requestFinished` | Request completed |
| `requestFailed` | request failed |
| `retryScheduled` | Schedule a retry |

---

## Protobuf support


Protocol Buffers are also supported.

```swift
import SwiftProtobuf

struct GetProtobufData: ProtobufAPIDefinition {
    typealias Parameter = MyRequestProto  // Protobuf message
    typealias APIResponse = MyResponseProto

    var parameters: MyRequestProto?
    var method: HTTPMethod { .post }
    var path: String { "/data" }
}

let response = try await client.protobufRequest(GetProtobufData(...))
```

---

## Real architecture example


### With the Repository layer


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

API definition and business logic are neatly separated.

---

## Download (InnoNetworkDownload)


Downloading large files uses a separate module.

```swift
import InnoNetworkDownload

let manager = DownloadManager.shared

// start download
let task = await manager.download(
    url: URL(string: "https://example.com/large-file.zip")!,
    toDirectory: documentsDirectory
)

// monitor progress (AsyncSequence)
for await event in manager.events(for: task) {
    switch event {
    case .progress(let progress):
        print("Progress: \(progress.percentCompleted)%")
        print("Bytes received: \(progress.bytesReceived)")
    case .completed(let url):
        print("Completed: \(url)")
    case .failed(let error):
        print("Failed: \(error)")
    case .paused:
        print("Paused")
    case .resumed:
        print("Resumed")
    }
}

// pause/resume
await manager.pause(task)
await manager.resume(task)
```

Supports background downloads and resumable downloads.

---

## Problems InnoNetwork Solve


| problem | How to solve |
|------|----------|
| URLSession Long-winded code | Simplified with `APIDefinition` protocol |
| Lack of type safety | Request/response strongly typed with generics |
| Error handling unclear | Systematized as `NetworkError` enumeration |
| Complex retry logic | Abstracted with `RetryPolicy` protocol |
| callback hell | async/await based |
| Interceptor implementation hassle | `RequestInterceptor`, `ResponseInterceptor` protocol |
| Difficulty tracking network status | `NetworkMonitor`, event observer |

---

## Requirements


- iOS 26+ / macOS 14+ / tvOS 26+ / watchOS 26+ / visionOS 26+

- Swift 6.2+


Supports strict concurrency in Swift 6. `DefaultNetworkClient` is implemented as `actor`, and the main type conforms to `Sendable`.

---

## Conclusion


InnoNetwork is a type-safe, native Swift Concurrency networking framework.

### When recommended


- When you want to neatly organize your API request code

- When you want to ensure type safety

- When you want to structurally handle retries, interceptors, etc.

- When working in a Swift 6 strict concurrency environment


### Summary of Benefits


1. **Type safety** - request/response strongly typed with generics

2. **Concise API** - Protocol-based definition, default implementation provided

3. **Swift Concurrency** - fully supports async/await, actor

4. **Retry Policy** - exponential backoff, network change detection

5. **Interceptor** - Intercept requests/responses

6. **Event Observation** - Logging, Analysis, Monitoring

7. **Swift 6 ready** - Full support for Sendable and actor


---

## Reference Materials


- [InnoNetwork GitHub Repository](https://github.com/InnoSquadCorp/InnoNetwork)

- [Swift Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)

- [URLSession](https://developer.apple.com/documentation/foundation/urlsession)
