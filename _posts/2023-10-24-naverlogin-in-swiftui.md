---
title: SwiftUI로 네이버 로그인 (네아로) 제대로 구현하기 (복붙아님)
author: ethan
date: 2023-10-24 18:55:00 +0900
categories: [SwiftUI]
tags: [tutorial, swiftui]
pin: false
image:
  path: /assets/img/naverlogin/naverlogin.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: SwiftUI로 네이버 로그인 제대로 구현하기
---

## 사전지식

[Pod](https://cocoapods.org/), [Carthage](https://github.com/Carthage/Carthage), [SPM](https://www.swift.org/package-manager/) 등에 대한 기본적인 이해가 있다고 가정합니다.

---

## 서론
네이버 로그인을 붙이려고 여러 방면으로 검색을 해본 결과

이상한 복붙 가이드만 잔뜩 있는 상황이었습니다. 추가로 그런 블로그의 가이드대로 하면 100% 협업시 문제가 생길 수 밖에 없는 구조로 안내하고 있어 정리하여 포스팅을 올리게 되었습니다.

(덕분에 github.io 시작...)

(+ 네이버는 업데이트할 생각이 없는게 분명하다. 왜 사용자 정보는 또 API를 이용하라는거냐..)

---

## 본론

### 프로젝트 설정 (Pod 기준으로 설명합니다.)

#### 1. Pod install

프로젝트 폴더 생성한 후 `pod init` 명령을 실행하면 `Podfile`이 생성됩니다.

이 `Podfile`에 아래 내용을 추가한 후 `pod install` 명령을 실행합니다.

```
target '<ProjectName>' do
  use_frameworks!

  
  pod 'naveridlogin-sdk-ios'
end
```

이후 `.xcworkspace` 파일이 생성되는데, 이 파일을 사용하여 작업을 계속 진행합니다.


#### 2. 네이버 Developers 에서 애플리케이션 등록

네이버 로그인을 등록할 때 네이버에서는 `Download URL 과 URL Scheme`을 요구합니다.

![네이버 개발자 센터](/assets/img/naverlogin/naver_developers.png)


- Download URL의 경우, https://apps.apple.com/app/id{앱스토어커넥트에서 제공되는 번호}
- URL Scheme의 경우, 적당한 string (소문자로 된) 을 넣으면 된다.


#### 3. 프로젝트 설정에 정보 추가

네이버 Developers에서 애플리케이션을 등록 한 뒤 다시 프로젝트로 돌아와 프로젝트 설정을 수정합니다.

- info.plist 설정

![Info plist](/assets/img/naverlogin/xcode_plist.png)

프로젝트 설정 - [Info] - [Custom iOS Target Properties]에 LSApplicationQueriesSchemes를 추가 (Array 타입) 합니다.

item 0: `naversearchapp`
item 1: `naversearchthirdlogin`


- URL Scheme 설정

![URL Types](/assets/img/naverlogin/xcode_scheme.png)

프로젝트 설정 - [Info] - [URL Types]에 새로운 타입을 추가하고,

URL Schemes 영역에 이전에 작성한 `URL Scheme` 을 추가합니다. (여기서는 naverloginsample을 사용)


#### 주의사항
다른 가이드들에서는 Pods 폴더 내부에 있는 #define에 값을 할당해 주어야 한다고 설명합니다.

가이드를 적용하게 될 경우, 올바르게 동작하나 `pod update`를 실행하게 되면 이전에 수정했던 값들이 초기화됩니다.

이렇게 사용하게 되면 협업 혹은 CD(배포 자동화) 등에서 문제가 발생할 가능성이 아주 높습니다. (100%..?)


### 코드 작성

(네이버 로그인의 경우, 로그인 버튼이 독립된 컴포넌트이자 화면이라고 생각하여 작업을 진행하였습니다.)

#### 1. `App.swift` 작성

```swift
@main
struct NaverLoginSampleApp: App {
    
    init() {
        NaverLogin.configure() // 1. 네이버 로그인을 하기 위한 기본 설정 적용
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

#### 2. `NaverLogin.swift` 작성

```swift
import Foundation

import NaverThirdPartyLogin


final public class NaverLogin: NSObject {
    public static let shared = NaverLogin()
    
    static func configure() {
        NaverThirdPartyLoginConnection.getSharedInstance().isInAppOauthEnable = true
        NaverThirdPartyLoginConnection.getSharedInstance().isNaverAppOauthEnable = true
        
        NaverThirdPartyLoginConnection.getSharedInstance().serviceUrlScheme = "naverloginsample"
        NaverThirdPartyLoginConnection.getSharedInstance().consumerKey = "[Developers에 있는 값]"
        NaverThirdPartyLoginConnection.getSharedInstance().consumerSecret = "[Developers에 있는 값]"
        NaverThirdPartyLoginConnection.getSharedInstance().appName = "내 앱의 이름"
        NaverThirdPartyLoginConnection.getSharedInstance().delegate = NaverLogin.shared
    }
    
    private override init() { }
}

// MARK: - Public method
public extension NaverLogin {
    func login() {
        guard !isValidAccessTokenExpireTimeNow else {
            retreiveInfo()
            return
        }
        
        if isInstalledNaver {
            NaverThirdPartyLoginConnection.getSharedInstance().requestThirdPartyLogin()
        } else {
            NaverThirdPartyLoginConnection.getSharedInstance().openAppStoreForNaverApp()
        }
    }
    
    func logout() {
        NaverThirdPartyLoginConnection.getSharedInstance().resetToken()
    }
    
    func unlink() {
        NaverThirdPartyLoginConnection.getSharedInstance().requestDeleteToken()
    }
    
    func receiveAccessToken(_ url: URL) {
        guard url.absoluteString.contains("[내가 Developers에 등록한 URL Scheme]://") else { return }
        NaverThirdPartyLoginConnection.getSharedInstance().receiveAccessToken(url)
    }
    
}

// MARK: - Private variable
private extension NaverLogin {
    var isInstalledNaver: Bool {
        NaverThirdPartyLoginConnection.getSharedInstance().isPossibleToOpenNaverApp()
    }
    
    var isValidAccessTokenExpireTimeNow: Bool {
        NaverThirdPartyLoginConnection.getSharedInstance().isValidAccessTokenExpireTimeNow()
    }
}
 
// MARK: - Private method
private extension NaverLogin {
    func retreiveInfo() {
        guard isValidAccessTokenExpireTimeNow,
            let tokenType = NaverThirdPartyLoginConnection.getSharedInstance().tokenType,
            let accessToken = NaverThirdPartyLoginConnection.getSharedInstance().accessToken else {
            NaverThirdPartyLoginConnection.getSharedInstance().requestAccessTokenWithRefreshToken()
            return
        }
        
        Task {
            do {
                var urlRequest = URLRequest(url: URL(string: "https://openapi.naver.com/v1/nid/me")!)
                urlRequest.httpMethod = "GET"
                urlRequest.allHTTPHeaderFields = ["Authorization": "\(tokenType) \(accessToken)"]
                let (data, _) = try await URLSession.shared.data(for: urlRequest)
                let response = try JSONDecoder().decode(NaverLoginResponse.self, from: data)

            } catch {
                await NaverThirdPartyLoginConnection.getSharedInstance().requestAccessTokenWithRefreshToken()
            }
        }
    }
}

// MARK: - Delegate
extension NaverLogin: NaverThirdPartyLoginConnectionDelegate {
    // Required
    public func oauth20ConnectionDidFinishRequestACTokenWithAuthCode() {
        // 토큰 발급 성공시
        retreiveInfo()
    }
    
    public func oauth20ConnectionDidFinishRequestACTokenWithRefreshToken() {
        // 토큰 갱신시
        retreiveInfo()
    }
    
    public func oauth20ConnectionDidFinishDeleteToken() {
        // Logout
    }
    
    public func oauth20Connection(_ oauthConnection: NaverThirdPartyLoginConnection!, didFailWithError error: Error!) {
        // Error 발생
    }
    
    
    // Optional
    public func oauth20Connection(_ oauthConnection: NaverThirdPartyLoginConnection!, didFinishAuthorizationWithResult recieveType: THIRDPARTYLOGIN_RECEIVE_TYPE) {
        
    }
    
    public func oauth20Connection(_ oauthConnection: NaverThirdPartyLoginConnection!, didFailAuthorizationWithRecieveType recieveType: THIRDPARTYLOGIN_RECEIVE_TYPE) {
        
    }
}
```

- Public method를 통해 로그인, 로그아웃, unlink, url scheme 컨트롤 등을 진행합니다.
- Private method를 통해 로그인 후 openapi를 통해 계정 정보를 획득합니다.
- url scheme 컨트롤(receiveAccessToken 함수) 시에는 내가 Developers에 등록한 URL Scheme 값을 확인하여 네이버에서 로그인 요청한 URL 인지 확인합니다.

#### 3. `NaverLoginResponse.swift` 작성

```swift
import Foundation

struct NaverLoginResponse: Decodable {
    var response: NaverResponse
    
    struct NaverResponse: Decodable {
        let id: String // 동일인 식별 정보, 네이버 아이디마다 고유하게 발급되는 유니크한 일련번호 값
        let nickname: String
        let name: String
        let email: String
        let gender: String // - F: 여성 - M: 남성 - U: 확인불가
        let age: String // 사용자 연령대
        let birthday: String // 사용자 생일(MM-DD 형식)
        let birthyear: Int
        let mobile: String
        let profileImage: URL?
        
        enum CodingKeys: String, CodingKey {
            case id
            case nickname
            case name
            case email
            case gender
            case age
            case birthday
            case birthyear
            case mobile
            case profileImage = "profile_image"
        }
    }
}
```

- 네이버에서 제공되는 정보들을 가져올 수 있는 데이터. 
- 모두 필수 값으로 null은 제공되지 않는 것으로 문서상 확인하였습니다.

4. `Content.swift` 작성

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        VStack {
            Button(action: { NaverLogin.shared.login() }) {
                Image(systemName: "globe")
                    .imageScale(.large)
                    .foregroundStyle(.tint)
                Text("네이버 로그인")
            }
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

- 로그인 시에는 `NaverLogin.shared.login()` 과 같은 형태로 사용합니다.


---

## 결론

최신 컨텐츠가 없던것에 비하면 로그인 통합 하는 과정이 어렵지는 않았습니다.

(아무래도 이미 하고 있는 사람들은 컨텐츠가 필요없고, 모르는 사람들만 작성하기 때문이 아닌가 싶다.)

로그인을 진행하게 되면 아래와 같은 구조로 앱 전환이 진행되며 로그인을 하게 됩니다.

![sample.GIF](/assets/img/naverlogin/sample.gif){: width="200" }


샘플 코드는 [여기](https://github.com/Ethan-IS/NaverLoginSample-SwiftUI)에서 확인할 수 있습니다.


틀린 정보나 잘못된 부분이 있다면 댓글 부탁드립니다.

---

### 참고링크

- [네이버로그인 Github](https://github.com/naver/naveridlogin-sdk-ios)
- [네아로 iOS 가이드](https://developers.naver.com/docs/login/ios/ios.md)