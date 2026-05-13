---
title: "InnoNetwork Best Practice: Swift Concurrency 네트워크 경계를 설계하는 법"
date: 2026-02-23 00:00:00 +0900
translation_key: innonetwork-type-safe-networking
lang: ko-KR
description: "InnoNetwork를 왜 써야 하는지, Swift Concurrency 기반 네트워크 정책을 Remote 레이어 안에 어떻게 격리하는 것이 좋은지 설명합니다."
categories: [iOS, Swift, Networking]
tags: [swift, iOS, networking, async-await, swift-concurrency, clean-architecture, type-safe, api]
author: ethan
toc: true
comments: true
---

## 왜 네트워크 코드가 실무에서 어려운가

네트워크 코드는 처음에는 `URLSession.data(for:)`로 충분해 보입니다. 하지만 운영 앱에서는 곧 다른 문제가 생깁니다.

- endpoint마다 header와 timeout 정책이 달라집니다.
- retry가 HTTP method의 안전성을 무시하고 붙습니다.
- auth refresh가 중복 실행됩니다.
- error classification이 feature마다 달라집니다.
- logging과 tracing이 뒤늦게 들어오며 호출부가 지저분해집니다.
- DTO, domain model, repository 경계가 섞입니다.

[`InnoNetwork`](https://github.com/InnoSquadCorp/InnoNetwork)는 단순 URLSession wrapper가 아니라, Swift Concurrency 기반의 typed request pipeline입니다. request definition, client configuration, retry, interceptor, logger, cache, trust, test support를 명확한 제품군으로 나눕니다.

핵심 매력은 **네트워크 실행 정책을 feature 밖으로 꺼내고, 타입이 있는 endpoint 계약으로 고정할 수 있다는 점**입니다.

## InnoNetwork가 소유하는 경계

InnoNetwork가 소유하기 좋은 것:

- typed endpoint definition
- request/response decoding
- retry, timeout, transport policy
- request/response interceptor
- auth refresh/coalescing
- logging, tracing, event observation
- download, websocket, persistent cache 같은 transport-adjacent feature
- consumer test support

InnoNetwork가 소유하지 말아야 하는 것:

- domain entity 설계
- repository business policy
- feature loading state
- 화면 이동
- DI graph construction

즉 InnoNetwork는 "외부 API를 어떻게 호출할 것인가"를 소유합니다. "그 결과를 앱 도메인에서 어떻게 해석할 것인가"는 `Data`와 `Domain`의 일입니다.

## 제품군 선택

InnoNetwork 4.x는 public product가 역할별로 나뉩니다.

- `InnoNetwork`: core typed request pipeline
- `InnoNetworkAuthAWS`: AWS SigV4 reference signer
- `InnoNetworkDownload`: foreground/background download lifecycle
- `InnoNetworkWebSocket`: realtime connection lifecycle
- `InnoNetworkPersistentCache`: on-disk response cache
- `InnoNetworkTrust`: public-key pinning evaluation
- `InnoNetworkOpenAPI`: generated client transport support
- `InnoNetworkTestSupport`: consumer test helpers

일반 앱의 첫 진입점은 보통 `InnoNetwork`입니다.

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoNetwork.git", .upToNextMinor(from: "4.0.0"))
]
```

앱이 stable ledger만 사용한다면 `.upToNextMajor(from: "4.0.0")`도 선택할 수 있지만, minor 단위로 provisionally stable surface가 발전할 수 있으므로 `CHANGELOG.md`를 같이 확인하는 편이 좋습니다.

## Best practice 1. 앱 전체가 아니라 `Remote` 레이어에 가둡니다

InnoNetwork를 feature마다 직접 import하면 네트워크 정책이 앱 전체로 퍼집니다. InnoSample은 이 선택을 피합니다.

현재 구조는 다음과 같습니다.

- `Remote`: InnoNetwork client, request definition, interceptor, remote failure mapping
- `Data`: remote data source contract와 repository 구현
- `Domain`: repository protocol, entity, use case
- `Feature`: use case만 호출

이 구조에서 `PeopleFeature`는 `NetworkClient`를 모릅니다. feature는 `FetchPeopleUseCase`만 알고, 실제 HTTP 호출은 `Remote` 안에서 끝납니다.

## Best practice 2. client configuration은 한곳에서 조립합니다

운영 네트워크 정책은 endpoint마다 흩어지면 안 됩니다. InnoSample은 `RemoteClientFactory`에서 기본 client를 만듭니다.

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

여기서 retry, metadata, status handling, timeout이 한곳에 모입니다. 이 배치는 feature code를 깨끗하게 만들 뿐 아니라 운영 정책 리뷰를 쉽게 합니다.

## Best practice 3. endpoint는 의미 있는 타입으로 둡니다

반복적인 단순 API는 `EndpointBuilder`가 좋은 시작점입니다. 하지만 sample이나 production app에서 endpoint의 의미가 중요해지면 `APIDefinition` 타입으로 빼는 편이 좋습니다.

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

이렇게 하면 공통 header와 logger 정책을 remote request 계층에 모을 수 있습니다. endpoint 호출부는 문자열 path 조합보다 훨씬 명확해집니다.

## Best practice 4. error를 remote failure로 한 번 변환합니다

`NetworkError`를 feature까지 그대로 올리면 UI가 transport detail을 알게 됩니다. 반대로 모든 error를 `Error`로 뭉개면 원인을 잃습니다.

좋은 방식은 `Remote`에서 네트워크 error를 `RemoteFailure`로 변환하고, `Data/Domain`으로 올라가며 더 앱다운 error로 바꾸는 것입니다.

이렇게 나누면 각 레이어의 책임이 선명해집니다.

- InnoNetwork: HTTP/transport/decoding error classification
- Remote: 외부 API 호출 실패 의미로 변환
- Data: repository policy 적용
- Domain: feature가 이해할 수 있는 domain error 제공

feature는 "timeout인지, 500인지, decoding 실패인지"를 필요할 때만 domain language로 받습니다.

## Best practice 5. 테스트는 URLSession 경계에서 끊습니다

InnoNetwork는 `URLSessionProtocol`과 test support를 통해 네트워크 경계를 테스트하기 좋게 만듭니다. InnoSample의 remote test는 stub session으로 실제 request header와 response mapping을 확인합니다.

검증해야 할 것은 "진짜 인터넷이 되는가"가 아닙니다.

- path가 맞는가
- header가 붙는가
- status error가 변환되는가
- decoding failure가 올바르게 분류되는가
- retry/interceptor ordering이 의도와 맞는가

이런 테스트는 feature UI test보다 빠르고 안정적입니다.

## 도입했을 때 얻는 장점

InnoNetwork를 잘 쓰면 다음 장점이 생깁니다.

- endpoint 계약이 타입으로 남습니다.
- retry/auth/interceptor 정책이 호출부 밖으로 빠집니다.
- 네트워크 error가 더 일관되게 분류됩니다.
- feature가 transport 구현을 모릅니다.
- remote layer test가 쉬워집니다.
- download/websocket/cache/trust 같은 운영 기능을 같은 철학으로 확장할 수 있습니다.

단순히 "HTTP 호출을 쉽게 한다"가 핵심이 아닙니다. **네트워크가 앱 아키텍처를 침범하지 못하게 막는 것**이 더 큰 가치입니다.

## 언제 쓰면 좋은가

InnoNetwork는 이런 상황에서 매력이 큽니다.

- endpoint가 많고 공통 정책이 필요한 앱
- auth refresh, retry, timeout, logging이 중요한 앱
- feature별 네트워크 코드 중복이 늘어난 앱
- Swift Concurrency와 strict error classification을 선호하는 팀
- download, websocket, persistent cache까지 같은 package family로 다루고 싶은 팀

반대로 일회성 script나 아주 작은 prototype에서는 URLSession만으로 충분할 수 있습니다.

## InnoSample에서의 결론

[`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice %})에서 InnoNetwork는 `Remote` 안에만 강하게 존재합니다.

- `RemoteContainer`가 `NetworkClient`와 remote data source를 조립합니다.
- `RemoteClientFactory`가 retry/interceptor/timeout/logger 정책을 만듭니다.
- `RemoteRequest`가 공통 request surface를 제공합니다.
- `Data`는 remote data source protocol만 압니다.
- `Domain`과 `Feature`는 InnoNetwork를 모릅니다.

이것이 InnoNetwork의 좋은 사용법입니다. **네트워크 실행 정책은 강하게 타입화하되, 앱 바깥 경계에 가둡니다.** 그러면 feature는 더 단순해지고, 네트워크 정책은 더 운영 가능한 형태로 남습니다.
