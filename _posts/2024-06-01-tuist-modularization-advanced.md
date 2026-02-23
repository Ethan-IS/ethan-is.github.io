---
title: Tuist로 확장 가능한 모듈화 하기 Part 4 - 모듈화 실전
author: ethan
date: 2024-06-01 17:40:00 +0900
categories: [Architecture]
tags: [tutorial, swift, tuist, iOS]

pin: false
image:
  path: /assets/img/tuist_04/cover.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: 모듈화 이해하기
---

## 서론

이제 마지막으로 실제 현업에서 사용할 수 있을 정도의 프로젝트를 만들어보려고 합니다.

여러명이 동시에 개발한다고 가정하고, 최소 2개 이상의 피쳐가 존재한다고 생각하고 진행할 예정입니다.

[지난 블로그](https://ethan-is.github.io/posts/clean-architecture/) 에서 볼 수 있듯 아래와 같은 프로젝트 구조를 기반으로 Tuist Project를 생성하려고 합니다.

![module-architecture](/assets/img/tuist_04/module-architecture.png)

프로젝트 생성전에 먼저

모듈 추가 원칙과 Feature 모듈의 디테일한 부분 그리고 모듈을 추가할 때 주의해야할 점을 알아보겠습니다.

## 설계 원칙 (혹은 모듈 추가 원칙)

- 역할에 맞는 계층을 선택합니다. (Feature, Core, Layer 등)
- 인터페이스와 구현체를 분리할지 판단합니다.
- 사용처 및 사용 방식에 따른 framework / library 타입을 결정합니다. (dynamic, static)

### 주의사항

- MVVM, MVP 와 같은 디자인 패턴은 Feature와 관련이 있는 부분입니다.
- 모듈화는 그것보다 더 큰 범위에서 일어난 다는 점을 기억해주세요.

## Feature 모듈

Feature 모듈의 경우, UI를 그리는 부분과 데이터를 가져와서 가공하는 부분 두가지로 나뉘게 되는데요.<br>
모듈간 순환참조를 피하고, 샘플앱을 만들고, 테스트를 진행하는 부분이 다양하게 역여 있어서 조금 복잡한 구조를 띄게 됩니다.

![feature-module](/assets/img/tuist_04/feature-module.png)

**Interface**

- Feature 모듈에서 제공하는 기능 (UI 진입점) 에 대한 외부에서 접근 가능한 인터페이스와 모델을 제공합니다.

**UI**

- Feature의 UI 부분이 위치한 모듈입니다. (가능한 멍청한 UI를 만드는 것이 목표입니다)

**Presentation**

- UI의 동작에 대한 응답과 외부 Domain 과 연결된 지점입니다. (MVVM 구조에서 Model과 ViewModel이 위치한 곳입니다)

**Testing**

- Mock 데이터를 제공합니다.
- Example 앱에서 사용하기 위한 코드를 제공합니다.

**Tests**

- Unit Test와 UI Test가 위치한 모듈입니다.

**Example**

- Feature의 기능을 간단히 체험해 볼 수 있는 작은 앱입니다.

## 주의사항

### 모듈간 순환참조

예를 들어 Setting 화면에서 Profile 화면으로 이동하는 기능이 있고, Profile 화면에서 Setting 화면으로 이동하는 기능이 있다고 가정해볼까요?<br>
이렇게 된다면 Setting 모듈이 Profile 모듈을 들고있게 되고, Profile 모듈도 Setting 모듈을 들고 있게 되겠죠?<br>
이런 문제를 모듈간 순환참조 문제라고 합니다.

![feature-module-2](/assets/img/tuist_04/feature-module-2.png)

순환참조가 발생되면 컴포넌트 간의 명확한 경계가 사라지고 연쇄적으로 변경에 의한 영향이 발생할 수 있습니다.

엉클밥은 컴포넌트 간 순환참조가 있으면 안된다고 말하며 ADP (Acyclic Dependencies Principle) 라는 설계 원칙까지 주창합니다. 의존성 구조에 순환이 발생하는지 항상 관찰하며 순환이 발생한다면 어떤 식으로든 끊어야 한다고 강조합니다.

### 모듈간 순환참조 해결책

Case 1) Route 모듈을 만들어서 해결

- 문제점 → 모든 기능 모듈들이 라우팅 모듈에 의존해야하기 때문에 라우팅 모듈에 대한 의존성이 너무 커진다.

Case 2) 인터페이싱을 통한 의존성 우회 (DIP: Dependency Inversion Principle)

- 모듈을 인터페이스 모듈과 구현 모듈로 분리한다.
- 각 구현모듈은 인터페이스 모듈을 참조하여 순환참조를 해결한다.

![feature-module-3](/assets/img/tuist_04/feature-module-3.png)

## 실제 Tuist Project 생성

이제 주의사항까지 확인했으니 실제로 어떻게 Tuist 프로젝트를 만들면 좋을지 이야기 해보도록 하겠습니다.

지난번에 만들었던 [Basic 구조](https://github.com/Ethan-IS/iOS-Modularization-Sample/tree/basic)에서 시작하여 [Advanced 구조](https://github.com/Ethan-IS/iOS-Modularization-Sample/tree/advanced)로 만드는 것을 목표로 합니다.

### Workspace.swift

![workspace_swift](/assets/img/tuist_04/workspace_swift.png)

- Workspace를 만듭니다.
  - 프로젝트는 App, Features, Layers, Cores, ThirdParties, Utils로 구분하여 등록합니다. (위의 이미지를 프로젝트로 만들었다고 생각해주시면 됩니다)

### App

![app_swift](/assets/img/tuist_04/app_swift.png)

- 앱은 지난번과 달라진것이 없습니다.
  - 디팬던시로 각 레이어에 속하는 가장 큰 프로젝트를 소유하게 됩니다.

### Features

![features_swift_01](/assets/img/tuist_04/features_swift_01.png)

#### Features 모듈

각 피쳐들의 Interface 모듈과 UI모듈을 갖고 있습니다.<br>
이곳에서 Dependency Injection 관련 작업을 진행합니다.

![features_swift_02](/assets/img/tuist_04/features_swift_02.png)
![features_swift_03](/assets/img/tuist_04/features_swift_03.png)

#### 개별 Feature

- 미리 정의해둔 Project extension을 통해 프로젝트를 생성합니다.
- Interface 모듈, UI 모듈, Presentation 모듈, Testing, Tests 모듈을 생성합니다. (Example은 필요에 따라 생성합니다)

- Interface
  - 별도의 dependency 없이 dynamic framework 의 형태를 갖습니다.
- UI
  - 필요한 dependency를 주입받고, DI에 필요한 NeedleFoundation을 갖고, interface와 presentation을 갖고 있습니다.
- Presentation
  - 필요한 dependency를 주입받고, 데이터를 가져오는 작업 등을 위한 Domain 모듈을 갖게 됩니다.

### Layers

![layers_swift_01](/assets/img/tuist_04/layers_swift_01.png)
![layers_swift_02](/assets/img/tuist_04/layers_swift_02.png)
![layers_swift_03](/assets/img/tuist_04/layers_swift_03.png)

- 현재 Layer의 구조에는 Domain, Data, Remote 의 구조를 갖고 있습니다.
  - DI는 전체 App에서 진행합니다.
- 소유 관계는 Domain ← Data ← Remote 로 되어있으며
  - Domain은 dynamic framework
  - Data, Remote 는 static library로 되어있습니다.

### Cores

![cores_swift_01](/assets/img/tuist_04/cores_swift_01.png)
![cores_swift_02](/assets/img/tuist_04/cores_swift_02.png)
![cores_swift_03](/assets/img/tuist_04/cores_swift_03.png)

- Core의 경우에는 각 모듈이 서로 의존관계를 형성하지 않고
  - 각 모듈이 필요한 곳에서 직접 사용하는 형태를 띄게 됩니다.
  - 현재 구조상으로는 모두 dynamic framework를 갖고 있으나 필요에 따라 static library로 구현해도 무방합니다.

### ThirdParties

![third_parties_swift_01](/assets/img/tuist_04/third_parties_swift_01.png)
![third_parties_swift_02](/assets/img/tuist_04/third_parties_swift_02.png)
![third_parties_swift_03](/assets/img/tuist_04/third_parties_swift_03.png)

- ThirdParty의 경우, interfacing 이 필요한 경우 만들어서 사용하게 됩니다.
  - 이렇게 만들어진 서드파티 라이브러리의 경우 다른곳에서 직접 사용하지는 않고 내부 구현 모듈에서만 참조하게 됩니다.
  - 우리가 생성한 모듈들에서는 인터페이스를 통해 접근하게 됩니다.

### Utils

![utils_swift_01](/assets/img/tuist_04/utils_swift_01.png)
![utils_swift_02](/assets/img/tuist_04/utils_swift_02.png)
![utils_swift_03](/assets/img/tuist_04/utils_swift_03.png)

- Util의 경우에는 Core 와 마찬가지로 각 모듈이 서로 의존관계를 형성하지 않고
  - 각 모듈이 필요한 곳에서 직접 사용하는 형태를 띄게 됩니다.

### Helper extensions

![helper_extensions](/assets/img/tuist_04/helper_extensions.png)

- 각 모듈들이 많아지게 되면 모듈을 직접 사용하는 곳에서 직접 String으로 접근해야 하는 경우가 많아지는데요.
- 일반적인 코드를 짤때도 마찬가지지만 휴먼에러를 방지하기 위한 작업입니다.

## 앱 내부 DI 및 기본 코드 작성

`tuist edit` 을 통해서 프로젝트를 생성하는 방법을 앞에서 확인해보았는데요.

이제 어떻게 DI를 하고, DIP를 실제로 사용할 수 있는지 확인해보고자 합니다.

### NeedleFoundation을 통한 DI 작성

- 기본적으로 DI 작성은 [Needle](https://github.com/uber/needle) 이라는 라이브러리를 사용하고 있습니다.

![app](/assets/img/tuist_04/app.png)

- Feature의 DI는 Features에서 진행합니다.
- Layer의 DI는 전체 앱에서 진행합니다.
- ThirdParty의 경우 필요에 따라 App이나 ThirdParties에서 진행합니다.

### Feature 모듈 자세히 둘러보기

![feature_detail_01](/assets/img/tuist_04/feature_detail_01.png)
![feature_detail_02](/assets/img/tuist_04/feature_detail_02.png)

Interface에서 홈화면을 만들어주는 protocol을 생성합니다.<br>
Interface의 구현은 UI 모듈에서 진행하게 됩니다.

![feature_detail_03](/assets/img/tuist_04/feature_detail_03.png)

각 Screen 에서는 다음화면으로 넘어가기 위한 Builder와 ViewModel을 주입받게 됩니다.

![feature_detail_04](/assets/img/tuist_04/feature_detail_04.png)

ViewModel은 필요한 UseCase를 dependency로 선언하여 갖고 오게 됩니다.

---

## 결론

[`Part 1. 모듈화 이해하기`](https://ethan-is.github.io/posts/understanding-modularity/)부터 `Part 4. 모듈화 실전`까지 전체적으로 iOS에서 모듈화를 하는 방식에 대해서 한번 깊게 고민해볼 수 있는 시간이었습니다.<br>
왜 모듈화를 해야하고, 어떻게 모듈화를 하는게 좋을지 고민하면서<br>
DIP, DI, Clean Architecture와 같은 다양한 개념들을 마주할 수 있던 좋은 기회였습니다.

이후 iOS 팀에서는 Swift의 동시성에 대해 한번 깊게 알아볼 예정입니다.<br>
Actor, Sendable 등 다양한 개념에 대해 제대로 이해하고 쓸 수 있는 개발자가 되기 위해 모두 함께 노력할 수 있는 팀이 되면 좋겠습니다.

모듈화 아티클은 여기서 마치고 다음 시리즈로 돌아오겠습니다.

---

## 참고링크

- [당근 테크 블로그](https://medium.com/daangn/tuist-%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%B4-%EB%AA%A8%EB%93%88-%EA%B5%AC%EC%A1%B0-%EC%9E%90%EB%8F%99%ED%99%94%ED%95%98%EA%B8%B0-f200992d4bf2)
- [토스 기술 블로그](https://toss.tech/article/slash23-iOS)
