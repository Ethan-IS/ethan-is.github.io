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

## Introduction

In the earlier versions of this post, I described `InnoNetwork` mostly as "a type-safe HTTP client built on Swift Concurrency." That is still true, but it no longer describes the full public surface of `3.0.1`.

The package now exposes three distinct products:

- `InnoNetwork`
- `InnoNetworkDownload`
- `InnoNetworkWebSocket`

And the public contract is no longer just about sending HTTP requests. It also includes **explicit configuration entry points, transport policy, event delivery, durability, and reconnect behavior**.

This post revisits `InnoNetwork` from that perspective.

## Why revisit it now

The important shift in `3.0.x` is not just "more examples." It is that the framework now has clearer operational boundaries:

- `safeDefaults` is the recommended entry point
- `advanced` is where operational tuning belongs
- download and websocket lifecycle are separated into dedicated products
- Protocol Buffers support moved into a separate `InnoNetworkProtobuf` package
- bounded buffering, append-log durability, and reconnect taxonomy are now part of the documented runtime model

So the right way to explain `InnoNetwork` today is not only "how to send a request," but **which network lifecycle belongs to which surface**.

## Product structure and ownership boundary

It helps to start with the product split.

### `InnoNetwork`

This is the core request/response module.

- `APIDefinition`
- `MultipartAPIDefinition`
- `DefaultNetworkClient`
- `NetworkConfiguration`
- `TrustPolicy`
- `RetryPolicy`

### `InnoNetworkDownload`

This product owns download lifecycle.

- `DownloadConfiguration`
- `DownloadManager`
- progress, completion, and failure streams
- foreground and background orchestration

### `InnoNetworkWebSocket`

This product owns connection-oriented realtime flows.

- `WebSocketConfiguration`
- `WebSocketManager`
- reconnect behavior
- heartbeat and pong timeout
- event streams

That separation is important. `InnoNetwork` is not one giant "network utility" module. It is a framework family that splits surfaces by transport lifecycle.

## Install

The base package installation for `3.0.1` starts here.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", from: "3.0.1")
]
```

## Core request model

The core modeling primitive is still `APIDefinition`.

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

Execution goes through `DefaultNetworkClient`.

```swift
let client = DefaultNetworkClient(
    configuration: .safeDefaults(
        baseURL: URL(string: "https://api.example.com/v1")!
    )
)

let user = try await client.request(GetUser())
print(user.name)
```

One meaningful change from older examples is that `APIConfigure` is no longer the main entry point for configuration in the docs. In `3.0.x`, the dominant style is **explicit configuration objects**.

## `safeDefaults` and `advanced`

The strongest message in the current documentation is simple:

> stay on `safeDefaults` unless you have a real operational reason to tune behavior

### Recommended starting point

```swift
let client = DefaultNetworkClient(
    configuration: NetworkConfiguration.safeDefaults(
        baseURL: URL(string: "https://api.example.com")!
    )
)
```

### Move to `advanced` only when needed

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

That distinction matters because `advanced` is public and supported, but its tuning values are not the main compatibility story. The recommended public path is still `safeDefaults`.

## Transport and operational surface

To understand `InnoNetwork 3.x`, request and response modeling is only one part of the story.

### `RetryPolicy`

Retry behavior is isolated into policy rather than leaking into business logic.

- `RetryPolicy`
- `ExponentialBackoffRetryPolicy`

### `TrustPolicy`

Trust evaluation is also an explicit public surface.

- `.systemDefault`
- public key pinning

### `EventDeliveryPolicy`

This is one of the clearest signs of the `3.x` direction. Event delivery is not treated as an unbounded implementation detail. It is modeled as a bounded buffering policy with overflow behavior that can be tuned explicitly.

That means the framework gives you a public way to talk about what should happen when event streams start backing up under load.

### `URLQueryEncoder` and `AnyResponseDecoder`

The documentation also makes encoding and decoding behavior more explicit than before.

- `URLQueryEncoder` is the public surface behind deterministic query and form encoding.
- `AnyResponseDecoder` is one of the public pieces used for explicit decoding strategies.

You may not touch these types every day, but they are still part of the public `3.x` contract and worth naming explicitly.

## Interceptors and request/response policy

Earlier versions of this post made `RequestInterceptor`, `ResponseInterceptor`, and `RetryPolicy` sound like the whole framework. In `3.x`, the tone should be slightly different.

These types still matter:

- `RequestInterceptor`
- `ResponseInterceptor`
- `RetryPolicy`

But the more important story is that **request execution happens inside explicit policy boundaries**.

The README and changelog make it clear that request/response execution was reorganized around internal transport policies and explicit decoding strategies. That is essential to the implementation, but in a blog post it is better framed as runtime architecture rather than a stable public contract.

## Download lifecycle

Download behavior belongs to `InnoNetworkDownload`.

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

The important part is not just "it downloads files." The product is designed around:

- foreground and background orchestration
- pause, resume, and retry
- listener retention
- append-log persistence for durability

So `DownloadManager` is better understood as a **download lifecycle manager** than as a convenience wrapper around `URLSessionDownloadTask`.

You can also move to explicit configuration when needed.

```swift
let configuration = DownloadConfiguration.advanced { builder in
    builder.maxConnectionsPerHost = 3
    builder.maxRetryCount = 2
}
```

Again, the pattern is the same: `safeDefaults` first, `advanced` only for concrete tuning needs.

## WebSocket lifecycle

Realtime connection behavior belongs to `InnoNetworkWebSocket`.

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

The main value here is not just connect/send/receive. It is the operational model around:

- reconnect policy
- handshake-aware close taxonomy
- reconnect suppression rules
- heartbeat and pong timeout
- event delivery policy

So when explaining `WebSocketManager`, it is more accurate to focus on **how it manages failure and reconnect behavior** than on the bare existence of a websocket client.

## Error handling

`InnoNetwork` favors explicit transport errors over opaque failure buckets.

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

`invalidRequestConfiguration` is especially useful because it usually means the request shape and policy do not agree. That is much more actionable than an opaque generic failure.

## Protocol Buffers are now a separate package

Older versions of this post treated Protocol Buffers support as if it were still part of the main package surface. That is no longer accurate in `3.0.1`.

If you need protobuf request/response support, you now add **`InnoNetworkProtobuf` as a separate package**.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", from: "3.0.1"),
    .package(url: "https://github.com/InnoSquadCorp/InnoNetworkProtobuf.git", branch: "main")
]
```

That is best read as a clearer architectural split between the core transport package and the protobuf adapter layer, not as a missing feature.

## When to use which surface

| Situation | Recommended choice |
|------|------|
| standard REST or HTTP APIs | `InnoNetwork` + `NetworkConfiguration.safeDefaults(baseURL:)` |
| retry, trust, metrics, or event delivery tuning | `NetworkConfiguration.advanced(baseURL:_:)` |
| file download with lifecycle tracking | `InnoNetworkDownload` + `DownloadManager` |
| realtime connection with reconnect and heartbeat requirements | `InnoNetworkWebSocket` + `WebSocketManager` |
| protobuf request or response modeling | `InnoNetwork` + `InnoNetworkProtobuf` |

## Closing thoughts

If I had to summarize `InnoNetwork 3.0.1` in one sentence, I would no longer call it only a type-safe HTTP client. A better description is **a networking framework family with explicit operational policy**.

`APIDefinition` and `DefaultNetworkClient` are still the starting point. But to use `3.x` well, it helps to keep four things in view:

- `safeDefaults` is the default path
- `advanced` is for local operational tuning
- download and websocket lifecycle belong to separate products
- protobuf support now lives in a separate package

For a deeper look, I recommend reading these in order:

1. [README](https://github.com/InnoSquadCorp/InnoNetwork)
2. [`API_STABILITY.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/API_STABILITY.md)
3. [`Examples/README.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/Examples/README.md)
4. [`docs/releases/3.0.1.md`](https://github.com/InnoSquadCorp/InnoNetwork/blob/main/docs/releases/3.0.1.md)
