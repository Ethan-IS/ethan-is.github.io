---
title: "InnoNetwork Best Practices: Designing Swift Concurrency Networking Boundaries"
date: 2026-02-23 00:00:00 +0900
translation_key: innonetwork-type-safe-networking
lang: en
description: "Why InnoNetwork is useful, how to isolate Swift Concurrency networking policy inside a Remote layer, and how InnoSample applies that boundary."
categories: [iOS, Swift, Networking]
tags: [swift, iOS, networking, async-await, swift-concurrency, clean-architecture, type-safe, api]
author: ethan
toc: true
comments: true
---

## Why networking gets hard in production apps

Network code looks simple when it starts with `URLSession.data(for:)`. Production apps quickly need more.

- Headers and timeout policies vary by endpoint.
- Retry is added without respecting HTTP method safety.
- Auth refresh runs multiple times under concurrent failures.
- Error classification differs by feature.
- Logging and tracing are bolted onto call sites.
- DTOs, domain models, and repositories blur together.

[`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork) is not just a URLSession wrapper. It is a Swift Concurrency request pipeline with typed endpoint definitions, client configuration, retry, interceptors, logging, cache, trust, and test support.

Its strongest appeal is that **network execution policy can stay outside features while endpoint contracts remain typed.**

## The boundary InnoNetwork should own

InnoNetwork should own:

- typed endpoint definitions
- request and response decoding
- retry, timeout, and transport policy
- request and response interceptors
- auth refresh and coalescing
- logging, tracing, and event observation
- download, websocket, persistent cache, and trust surfaces
- consumer test support

It should not own:

- domain entity design
- repository business policy
- feature loading state
- navigation
- dependency graph construction

InnoNetwork answers "how does this app call external APIs?" How the result becomes app meaning belongs to `Data` and `Domain`.

## Product selection

InnoNetwork 4.x exposes products by role.

- `InnoNetwork`: core typed request pipeline
- `InnoNetworkAuthAWS`: AWS SigV4 reference signer
- `InnoNetworkDownload`: foreground/background download lifecycle
- `InnoNetworkWebSocket`: realtime connection lifecycle
- `InnoNetworkPersistentCache`: on-disk response cache
- `InnoNetworkTrust`: public-key pinning evaluation
- `InnoNetworkOpenAPI`: generated-client transport support
- `InnoNetworkTestSupport`: consumer test helpers

Most apps start with `InnoNetwork`.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", .upToNextMinor(from: "4.0.0"))
]
```

If you only use the stable API ledger, `.upToNextMajor(from: "4.0.0")` may be appropriate. If you use provisionally stable surfaces, review `CHANGELOG.md` before taking minor upgrades.

## Best practice 1. Keep it inside `Remote`

When features import the network client directly, transport policy leaks throughout the app. InnoSample avoids that.

The current layering is:

- `Remote`: InnoNetwork client, request definitions, interceptors, remote failure mapping
- `Data`: remote data source contracts and repository implementations
- `Domain`: repository protocols, entities, and use cases
- `Feature`: use case calls only

`PeopleFeature` does not know `NetworkClient`. It calls `FetchPeopleUseCase`; the HTTP call finishes inside `Remote`.

## Best practice 2. Build client policy in one place

Production network policy should not be scattered endpoint by endpoint. InnoSample builds the client in `RemoteClientFactory`.

```swift
enum RemoteClientFactory {
    static func makeClient(
        baseURL: URL,
        session: URLSessionProtocol = URLSession.shared
    ) -> any NetworkClient {
        let configuration = NetworkConfiguration.advanced(
            baseURL: baseURL,
            resilience: ResiliencePack(
                retry: ExponentialBackoffRetryPolicy(
                    maxRetries: 2,
                    maxTotalRetries: 2,
                    retryDelay: 0.4
                )
            ),
            auth: AuthPack(
                additionalSigners: [
                    RemoteMetadataInterceptor(environment: .init(baseURL: baseURL)),
                ],
                additionalResponseInterceptors: [
                    RemoteStatusInterceptor(),
                ]
            ),
            transport: TransportPack(
                timeout: 20.0,
                cachePolicy: .reloadIgnoringLocalCacheData
            )
        )
        return DefaultNetworkClient(configuration: configuration, session: session)
    }
}
```

Retry, metadata, status handling, and timeout policy are reviewed in one place. Feature call sites stay clean.

## Best practice 3. Give endpoints meaningful types

For simple onboarding, `EndpointBuilder` is a good first path. Once endpoint meaning matters, dedicated `APIDefinition` types become easier to maintain.

```swift
import InnoNetwork

protocol RemoteRequest: APIDefinition {
    var featureName: String { get }
}

extension RemoteRequest {
    var method: HTTPMethod { .get }

    var headers: HTTPHeaders {
        var headers = HTTPHeaders.default
        headers.update(name: "X-Sample-Feature", value: featureName)
        return headers
    }

    var logger: NetworkLogger {
        RemoteRequestLogger()
    }
}
```

This keeps common headers and logging policy in the remote request layer instead of repeating string-based setup at every call site.

## Best practice 4. Convert errors once at the remote boundary

Passing `NetworkError` all the way to features exposes transport detail. Erasing everything to `Error` loses useful classification.

A better shape is to convert network errors to `RemoteFailure` in `Remote`, then translate again as needed in `Data` or `Domain`.

- InnoNetwork: HTTP, transport, and decoding classification
- Remote: external API failure meaning
- Data: repository policy
- Domain: feature-facing domain error

Features receive app language, not raw transport language, unless they explicitly need it.

## Best practice 5. Test at the URLSession boundary

InnoNetwork's `URLSessionProtocol` and test support make the remote boundary testable. InnoSample's remote tests use stub sessions to verify requests and response mapping.

The important questions are not "does the real internet work?"

- Is the path correct?
- Are headers applied?
- Is status failure converted correctly?
- Is decoding failure classified correctly?
- Do retry and interceptor ordering match the policy?

Those tests are faster and more stable than UI tests.

## What you gain by adopting it

Used well, InnoNetwork gives you:

- typed endpoint contracts
- retry/auth/interceptor policy outside feature code
- more consistent network error classification
- features that do not know transport implementation
- testable remote boundaries
- a consistent family for download, websocket, cache, and trust features

The core value is not "HTTP made easy." The core value is **preventing networking from leaking across your app architecture.**

## When it is a good fit

InnoNetwork is attractive when:

- an app has many endpoints and shared policies
- auth refresh, retry, timeout, and logging matter
- feature-level network duplication is growing
- the team prefers Swift Concurrency and strict error classification
- download, websocket, persistent cache, or trust features should follow the same package family

For one-off scripts or very small prototypes, plain URLSession may be enough.

## How InnoSample uses it

In [`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice-en %}), InnoNetwork is strongly contained inside `Remote`.

- `RemoteContainer` builds `NetworkClient` and remote data sources.
- `RemoteClientFactory` owns retry, interceptor, timeout, and logger policy.
- `RemoteRequest` provides the common request surface.
- `Data` sees only remote data source protocols.
- `Domain` and `Feature` do not know InnoNetwork.

That is the best way to use InnoNetwork: **type the networking policy strongly, then keep it at the app boundary.** Features stay simpler, and network policy remains operationally reviewable.
