---
title: Tuist로 확장 가능한 모듈화 하기 Part 3 - Clean Architecture 이해하기
author: ethan
date: 2024-05-13 17:40:00 +0900
categories: [Architecture]
tags: [tutorial, swift, tuist, iOS]

pin: false
image:
  path: /assets/img/tuist_03/cover.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: 모듈화 이해하기
---

## 서론

iOS 개발문화가 성숙하기 시작하면서 단순히 MVC, MVVM, MVP 와 같은 프리젠테이션 Layer의 아키텍쳐 패턴 뿐만 아니라. Clean Architecture와 같은 multi layer를 통해 관심사를 더 다양하게 분리할 수 있는 환경에 대한 관심도가 높아지고 있습니다.

이전까지는 하나의 프로젝트에서 폴더링을 하면서 개발을 해왔다면, 현재는 다양한 프로젝트를 만들고 중간중간 필요한 만큼 샘플앱을 만들면서 개발을 해오고 있는데요.

이렇게 멀티 프로젝트의 구조를 가져가게 되면서 하나의 UI와 로직의 두개의 Layer가 있는 것이 아니라, UI / UI 로직 / Domain 로직 / Data 로직 / Remote(Local) 로직 등으로 더 세분화 되고 있는 현실입니다.

그 중에서 가장 대중적으로 사용되고 있는 Layer 분리 방법 중에 하나인 Clean Architecture에 대해 알아보고
장점과 단점은 무엇이고 이것을 iOS 개발에서는 어떻게 활용하면 좋을지 알아보고자 합니다.

## 왜 아키텍쳐 설계를 하면서 개발을 해야할까?

아키텍쳐의 정의는 아래와 같습니다.

> [시스템 아키텍처(system Architecture)](https://ko.wikipedia.org/wiki/%EC%8B%9C%EC%8A%A4%ED%85%9C_%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98)는 [시스템](https://ko.wikipedia.org/wiki/%EC%8B%9C%EC%8A%A4%ED%85%9C)의 [구조](https://ko.wikipedia.org/wiki/%EA%B5%AC%EC%A1%B0), [행위](https://ko.wikipedia.org/wiki/%ED%96%89%EB%8F%99), 더 많은 [뷰](https://ko.wikipedia.org/w/index.php?title=%EB%B7%B0_%EB%AA%A8%EB%8D%B8&action=edit&redlink=1)를 정의하는 [개념적 모형](https://ko.wikipedia.org/wiki/%EA%B0%9C%EB%85%90%EC%A0%81_%EB%AA%A8%ED%98%95)이다.
> 시스템 목적을 달성하기 위해 시스템의 각 [컴포넌트](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8)가 무엇이며 어떻게 [상호작용](https://ko.wikipedia.org/wiki/%EC%83%81%ED%98%B8%EC%9E%91%EC%9A%A9)하는지, [정보](https://ko.wikipedia.org/wiki/%EC%A0%95%EB%B3%B4)가 어떻게 교환되는지를 설명한다.

즉, 아키텍쳐란 하나의 소프트웨어가 어떻게 구성되며 어떻게 동작하는지 설명하는 것 입니다.

보통 한개의 팀에서 한가지 서비스를 개발하기 마련인데요. 그러한 상황에서 팀이 추구해야 하는 목표는 어떤게 있을까요?

1. 유지보수하기 좋고 편해야 한다.
2. 소프트웨어가 안정적으로 확장될 수 있어야 한다.
3. 테스트 하기 좋아야 한다 (안정성)
4. 재사용하기 쉽고 편해야한다 (빠른 개발속도)

이렇게 정리해볼 수 있을 것 같습니다.

그럼 이 목표를 달성하면서, 아키텍쳐를 설계한다 라는건 어떤것을 의미하는 것일까요?

[코드를 어떤 기준으로 나누고 어떻게 구성할지 어떻게 배치할지에 대해서 공통의 원칙을 만들어 두는 것 이라고 생각합니다.](https://iamchiwon.github.io/2020/08/27/main-thought-of-clean-architecture/#google_vignette)

그러면 어떻게 구성하고 어떻게 배치하면 우리의 목표를 달성할 수 있을까요?

1. 테스트 하기 좋아야 한다

   ⇒ 각 모듈이 서로 독립적이고, Mock 데이터 등으로 변경하기 수월해야 한다.

2. 재사용하기 쉽고 편해야한다

   ⇒ interfacing 등을 통해서 구현에 의존하는 것이 아니라 인터페이싱에 의존하도록 코드를 설계한다.

3. 안정적으로 확장될 수 있어야 한다

   ⇒ 수정이 큰쪽이 수정이 적은쪽으로 의존하게 설계하여 변경하더라도 영향이 적게 모듈 구조를 조직한다.

4. 유지보수하기 좋고 편해야 한다.

   ⇒ 위의 3가지가 안정적으로 운영된다면 유지보수하기 좋고 편한 코드가 된다.

이렇게 볼 수 있을 것 같은데요. 구체적으로 어떤식으로 구조화를 하고 의존은 어떻게 가져가고 모듈이 독립적이기 위한 방법은 어떤것이 있는지. Clean Architecture와 저희 InnoSquad 팀에서 고민한 설계 방식을 함께 이야기해 보겠습니다.

<br>

## 그럼 [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)는 뭐야?

Robert C. Martin (aka. 엉클밥) 이 제안한 소프트웨어 아키텍쳐 방법론 입니다.

목표는 소프트웨어 시스템을 유연하고 테스트 가능하며 유지보수가 용이하도록 만드는 것입니다.

![clean architecture](/assets/img/tuist_03/clean-architecture.png)

### Dependency Rule - 최우선 규칙

- 내부 Circle의 어느것도 외부 Circle 에 대해 알 수 없습니다.
  - 즉, 외부 Circle에 선언된 항목의 이름은 내부 Circle의 코드에서 언급되어서는 안됩니다.
  - 즉, 외부 Circle에서 사용되는 데이터 형식은 내부 Circle에서 사용되어서는 안됩니다.
- 외부 Circle의 어떤 것도 내부 Circle에 영향을 미치지 않습니다.
- 계층 사이의 의존성은 안쪽에서 바깥쪽으로만 향합니다.

⇒ 외부 (UI 혹은 DB)가 내부 비즈니스 로직 (자주 바뀌지 않음)을 의존하게 하면서 테스트 용이성을 높이게 됩니다. (내부 로직은 독립적이게 되고, 안정적으로 Mock 데이터로 치환 가능해집니다.)

### Entities

- 어플리케이션의 비즈니스 개체입니다.
- method가 포함될 수 있으며, 데이터 구조일수 있습니다.

### UseCase

- 어플리케이션별 비즈니스 규칙입니다.
- Enitity 사이의 데이터 흐름을 조정하는 역할을 합니다.

### Interface Adapters

- 만약, GUI라면 MVC 아키텍쳐가 이부분에 포함됩니다.
- 만약, Database Gateway라면 이 계층에서 데이터베이스와 연결이 끝나야 합니다.

### Frameworks & Drivers

- Database, Web Framework 와 같은 도구로 구성됩니다.
- 대게 이 레이어는 Interface adapter와 통신하는 코드 이외에는 작성되지 않습니다.

### ETC

- 원은 반드시 4가지 이어야 한다는 규칙은 없습니다.
- 다만 Dependency Rule은 항상 적용되어야 합니다.
- 코드의 Dependency 방향은 항상 내부를 가리킵니다.
- 내부로 이동할수록 소프트웨어가 더 추상화 되고 캡슐화 됩니다.

### 장점

- 외부에서 내부로 들어갈 수록 프레임워크나 시스템에 독립적이게 됩니다. (테스트 용이성 증가)
- 특정 기능의 수정 및 추가가 전체에 영향을 주지 않게 됩니다. (유지보수 하기 좋고 편함)
- 따라서 기획자의 요구사항에 따라 유연하게 개발할 수 있다는 장점이 있습니다. (안정적으로 확장)

### 단점

- 내부를 변경하게 되면 필연적으로 외부의 코드가 변경되야 합니다.
- 그리고 내부를 처음 구성하게 되면 외부까지 연결하기 위한 다양한 작업들이 수반됩니다. (개발 피로도를 증가시키게 됩니다.)

<br>

## 그러면 iOS에서는 어떻게 아키텍쳐 설계를 하고 모듈화를 하는게 좋을까?

우리는 위의 과정을 통해서 어플리케이션의 아키텍쳐를 설계함에 있어서<br>
유지보수가 편하고, 재사용성이 높으며, 테스트 하기 좋은 코드를 만드는 것이 목표라는 것을 알게 되었습니다.

그리고 그 과정에서 가장 먼저 생각해야할 것은 바로 계층화 입니다. Layer를 나눈다 라고 볼 수 있을텐데요.<br>
이렇게 Layer를 나누게 되면 각각 관심사와 역할이 분리가 되면서 유지보수가 편하고 테스트하기 좋은 코드를 생산할 수 있게 됩니다.

그럼 이제 어떻게 Layer를 나누는게 좋을까요?<br>
저희 InnoSquad 팀에서 결정한 방식을 공유 드리겠습니다.

앱은 데이터를 통해 UI를 그리고, UI를 통해 사용자의 입력을 받아서 새로운 데이터 또는 화면 전환을 하는 것 이라고 볼 수 있습니다.<br>
그럼 여기서 우리가 나눌 수 있는 Layer는 어떤게 있을까요?<br>
여러 자료들을 찾아보면서 만든 최종적인 구조는 다음과 같습니다.

![module architecture](/assets/img/tuist_03/module-architecture.png)

### App

- 앱을 실행하는 본체입니다.
- 필요한 설정들을 작업하고, 화면의 시작점 그리고 Window 같은 설정들을 최초로 하게 되는 곳입니다.
- @main 이 위치하고 Bundle.main 이 위치한 곳입니다.

### Features

여러개의 피쳐 모듈이 모여 있는 곳입니다.<br>
DI를 위해 각 피쳐들을 등록하는 곳이라고 이해해주시면 좋을 것 같습니다.<br>
import SwiftUI, 또는 import UIKit 이 가능한 곳이기도 합니다.

#### UI

- 화면에 데이터를 표시하는 곳입니다.
- 사용자의 상호작용이 시작하는 곳입니다.
- iOS를 기준으로 한다면 View, ViewController 들이 존재하는 곳입니다.
- UI 로직 (UI 상태를 단순히 표시하는 방법)은 UI 레이어에서 처리됩니다. (버튼 클릭 시 routing, 특정 카테고리 선택시 토스트 노출 과 같은 행동)

#### Presentation

- ViewModel 과 화면에 표시하기 위한 UIState, Presentation Model이 존재하는 곳입니다.
- Domain Layer와 상호작용 하는 곳입니다.
- ViewModel 의 경우, 관찰자 패턴을 사용하여 UIState를 노출하고 method를 통해 UI 이벤트를 수신합니다.
  - 이때 뷰모델은 이벤트를 받은 즉시 처리하고 이벤트 처리 결과로 UIState를 업데이트 합니다 (직접 UI로 이벤트를 전달하지 않습니다)
  - ViewModel은 비즈니스 로직을 포함하며 계층 구조의 하위 레이어에서 얻은 결과를 UI 상태로 변환합니다.
- 화면 수준의 View에서만 ViewModel를 사용합니다.
- Model의 경우, Domain Layer에서 받는 모든 값이 아니라 화면에서 필요한 만큼만 정제하여 사용합니다.
- UI State는 변경할 수 없어야 한다.
  - 그래야 (변경 불가능 해야) 순간의 Application 상태를 보장할 수 있음.
  - 그래야 UI는 상태를 읽고 이에 따라 UI를 업데이트 한다는 역할에 집중 할 수 있음.
  - 그러므로 UI에서 UI 상태를 직접 수정해서는 안된다 (UI 자체가 데이터의 유일한 소스인 경우를 제외하고)

#### 주의 사항

- UI의 역할은 오직 UI 상태를 사용 및 표시하는 것이어야 합니다. (UDF: Unidirectial Data Flow)
  - ViewModel은 UI에 사용될 상태를 보유하고 노출합니다.
  - UI는 ViewModel에 사용자 이벤트를 알립니다.
  - ViewModel이 작업을 처리하고 상태를 업데이트 합니다.
  - 업데이트된 상태가 렌더링할 UI에 다시 제공됩니다.
  - 상태 변경을 야기하는 모든 이벤트에 위의 작업이 반복됩니다.
- ViewModel이 navigationController 또는 UIApplication 과 같은 정보를 갖고 있지 않습니다.
  - 이러한 것들은 UI 로직에 속한 것입니다.

### Domain Layer

애플리케이션의 핵심 비즈니스 로직과 규칙을 캡슐화하는 역할을 합니다.<br>
이 계층은 시스템이 해결하려는 문제 영역의 모델을 표현하며, 애플리케이션의 "뇌"라고 볼 수 있습니다

- 비즈니스 규칙 정의: 비즈니스 프로세스 및 규칙을 구현하여 애플리케이션의 비즈니스 로직을 관리합니다.
  - 일반적으로 UseCase 라는 클래스를 사용합니다.
  - 각 UseCase는 하나의 비즈니스 로직을 담당합니다.
    - 또한 변경 가능한 데이터를 포함할 수 없습니다. 변경가능한 데이터는 UI 또는 Data Layer에서 처리해야합니다.
    - 복잡한 계산은 재사용이나 캐싱을 위해 Data Layer에서 이루어집니다.
    - ex> "주문 처리" 유즈케이스는 사용자가 상품을 주문하고 결제를 완료하는 과정을 포함할 수 있습니다.
- 도메인 모델 생성: 실제 비즈니스 엔티티와 그 관계를 반영하는 객체(예: 사용자, 주문 등)를 포함합니다.
- 유효성 검사: 도메인 규칙을 기반으로 데이터 유효성을 검증합니다.
- 독립성 유지: 도메인 계층은 다른 계층(예: 데이터 액세스 계층, 프레젠테이션 계층)에 의존하지 않고 독립적으로 유지되어야 합니다. 이를 통해 비즈니스 로직의 수정이 다른 계층에 미치는 영향을 최소화합니다.

### Data Layer

주로 데이터 저장소와의 상호작용을 관리합니다.<br>
이 계층은 데이터베이스, 파일 시스템 또는 외부 API와 같은 데이터 소스로부터 데이터를 검색하거나 저장하는 기능을 담당합니다.

- 데이터 접근 추상화: 데이터베이스 쿼리, 스키마 관리, 데이터 트랜잭션 처리 등을 추상화하여 비즈니스 로직 계층이 데이터 저장소의 복잡성을 신경 쓰지 않도록 합니다.
- CRUD 작업: 데이터 생성(Create), 읽기(Read), 업데이트(Update), 삭제(Delete)와 같은 기본적인 데이터 조작 작업을 수행합니다.
- 데이터 통합: 여러 데이터 소스로부터 데이터를 통합하고 조화롭게 사용할 수 있게 돕습니다.
- 데이터 캐싱: 성능 향상을 위해 자주 사용되는 데이터를 캐싱합니다.

이 레이어에서 노출하는 데이터는 변경할 수 없어야 합니다.<br>
그래야 값을 일관되지 않은 상태로 만들 위험이 없어집니다. (다른 곳에서 조작할 가능성을 없앱니다)

### Remote / Local Layer

- Data Layer의 DataSource를 구현하는 곳입니다.
- Remote의 경우는 서버 통신을 위한 기본적인 구조 (API Definition)을 정의하고 실행합니다.
- Local의 경우, FileManager 또는 UserDefaults, KeyChain 과 관련된 정보를 정의하고 저장 및 fetch를 실행합니다.

### Core, Util, ThirdParty

서비스 전역에서 사용가능할 수 있는 모듈입니다.

#### Core

- Network, DesignSystem 과 같은 서비스 개발의 핵심 파트가 들어갑니다.
- 한번 구현된 이후, 변경될 가능성이 매우 적고 다양한 부분의 모듈에서 사용할 경우 이곳에 포함됩니다.
- 인터페이싱 하지 않고 바로 직접 사용하는 구조로 개발합니다.

#### Util

- UI 또는 Foundation 과 같은 곳에서 유틸성으로 사용하는 기능들을 만드는 곳입니다.
- 주로 extension 을 통해서 사용이 확장되고, 명확한 사용처가 있는 경우 별도의 클래스를 활용하여 개발하도록 합니다.
- 마찬가지로 다양한 곳에서 사용할 수 있는 것을 가정하고 모듈들을 생성합니다.

#### ThirdParty

- 서드파티의 인터페이싱을 만드는 곳입니다.
- 서드파티가 업데이트 됨에 따라 API 가 변경될 가능성이 있습니다. 이러한 부분을 직접 호출하게 되면, 유지보수에 어려움이 생깁니다.
- 이런 부분들을 protocol 로 만들어주어서 구현체와 분리되도록 합니다.
- 서드파티를 사용하는 곳에서는 서드파티를 직접 호출하는 것이 아니라 모듈에서 생성된 Interface를 사용하도록 강제합니다.

---

## 결론

iOS에서 어떻게 모듈화를 하면 좋을지 알아보는 시간이었습니다.<br>
아키텍쳐 왜 고민해야 하는지 생각해 보았고<br>
그에 따라 클린 아키텍쳐에 대해 한번 고민해보는 시간을 가져볼 수 있었습니다.<br>
그리고 다양한 아티클들과 함께 iOS에서 어떻게 Layer를 나누고 모듈을 가져가면 좋을지 생각해보았습니다.

지금 말씀드린 부분이 어느 상황에서나 맞는 정답이라고 할 수는 없습니다.<br>
다만 모듈 및 Layer마다 관심사 및 역할을 분리하여 개발하고, 내부는 외부를 모르게 개발한다면 어떻게 layer를 나누고 어떻게 모듈을 나누던지 옳은 방향으로 가고 있다고 생각할 수 있을 것 같습니다.

다음에는 구체적으로 그래서 어떻게 Tuist를 통해 우리가 결정한 모듈들을 생성할 수 있을지에 대해 알아보고<br>
그래서 [우리의 목적](#왜-아키텍쳐-설계를-하면서-개발을-해야할까) 은 어떻게 달성되는지를 함께 알아보도록 하겠습니다.

---

## 참고링크

- [The Clean Code Blog](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Android 앱 아키텍쳐 가이드](https://developer.android.com/topic/architecture?hl=ko)
- [클린 아키텍쳐의 핵심 아이디어 by 곰튀김](https://iamchiwon.github.io/2020/08/27/main-thought-of-clean-architecture/#google_vignette)
