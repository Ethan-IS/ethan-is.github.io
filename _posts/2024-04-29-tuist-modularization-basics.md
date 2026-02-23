---
title: Tuist로 확장 가능한 모듈화 하기 Part 2 - 모듈화 기본
author: ethan
date: 2024-04-29 19:00:00 +0900
categories: [Architecture]
tags: [tutorial, swift, tuist, iOS]

pin: false
image:
  path: /assets/img/tuist_02/cover.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: 모듈화 기본
---

## 왜 Tuist를 써야할까? (다른 대안은 없어?)

### SPM 으로 모듈화

직접 각 모듈을 Swift Package로 만들어 모듈화를 진행하는 방식입니다.

장점)

- 애플에서 공식적으로 지원하는 의존성 관리 도구입니다.
- 패키지 단위로 모듈이 되어, 독립적으로 빌드 / 테스트가 가능합니다.

단점)

- 파일 추가 삭제 시 새로 Resolve를 진행해야 합니다.
  - 모듈이 많아지게 되면, 실제 프로젝트를 빌드하는 경우 인덱싱에 상당한 시간이 소요됩니다.
- 모듈을 만들고 Link 하는 과정이 Manual 하게 진행됩니다.

### [XcodeGen](https://github.com/yonaskolb/XcodeGen) 으로 모듈화

`YAML` 파일을 사용하여 프로젝트 구성을 정의하는 방식입니다.

이 파일을 통해 타겟, 소스파일, 리소스, 종속성 등을 분석하여 .xcodeproj 를 생성합니다.

장점)

- Swift로 개발되어있습니다.
- 협업에 있어 프로젝트 파일 (.xcodeproj)의 충돌을 방지해줍니다.
- 타갯 분리가 쉽습니다.
- 종속성 관리를 위한 외부 Tool의 사용이 용이합니다.

단점)

- `.yml` 파일에 익숙한 iOS개발자가 많지 않습니다.
  - 이에 따라 러닝커브가 높은 수준으로 존재하며, 디버깅 등에 많은 시간이 소요됩니다.
- 프로젝트 구성이 `.yml` 파일로 관리 되기 때문에 각 빌드 구성이 String 으로 관리되는 단점이 있습니다.

### [Tuist](https://github.com/tuist/tuist) 로 모듈화

Swift DSL을 사용하여 프로젝트 구성을 정의하는 방식입니다.

장점)

- .swift 파일을 통해 manifest를 구성합니다. → iOS 개발자가 유지보수하기 용이합니다.
- Xcode 를 통해 manifest 구성이 가능합니다. → IDE의 도움을 받을 수 있습니다.
- 협업에 있어 프로젝트 파일 (.xcodeproj)의 충돌을 방지해줍니다.
- 캐싱, 병렬 빌드, Cloud 등 성능 최적화 및 CI 빌드를 위한 도구들이 있습니다.

단점)

- CocoaPod과 Carthage를 잘 지원하지 않습니다. (사용이 불가능하지는 않습니다)
- 버전이 안정적이지 않습니다.
  - 4 버전 업데이트 이후 잦은 수정과 링킹 오류들이 발생하고 있습니다.

---

## Tuist 로 결정한 이유

InnoSquad iOS 팀에서는 소거법으로 프로젝트 모듈화 방식을 결정하였습니다.

첫번째, XcodeGen의 경우 빠르게 나가야 하는 배포 특성상 .yml 을 익히면서 유지보수 하기에는 어렵다는 결론을 내렸습니다.

두번째, 그러면 어떤것이 좋을까 라고 생각했을 때 FirstParty 인 SPM 을 통해서 프로젝트 모듈화를 진행하였습니다. 그 과정에서 발생한 문제는 아래와 같습니다.

1. 위에서 말씀드린것과 같이, 파일 추가 제거 후 새로이 Resolve를 해주어야 하는 문제가 지속적으로 발생하였습니다.
2. 파일 생성 시 Xcode에서 지원하는 템플릿 (상단 Author 정보 등) 이 자동으로 추가되지 않았습니다.
3. 모듈화를 진행하면서 상당한 수의 모듈이 생성되었는데, 수동으로 모듈을 생성하면서 피로도가 상당히 증가하였습니다.

이후 Tuist 를 학습 한 뒤 적용을 진행하였고, 완료된 상태입니다.

위에서 말씀 드린 문제는 모두 해결되었습니다.

파일 추가 혹은 삭제 시 바로 반영되었으며, 모듈 생성은 stencil 을 통해 자동 생성을 지원하게 되었습니다.

그리고 한 앱에서 별도의 타갯으로 빌드가 가능하게 되면서 개발 생산성이 증가하였습니다.

---

## Tuist 로 모듈화 하기 (기초)

그럼 이제 가볍게 Tuist 로 어떻게 모듈화를 하는지 설명하겠습니다.

모듈의 구조는 기본적으로 [지난 글](https://ethan-is.github.io/posts/understanding-modularity) 에서 정리하였던 것 과 같은 구조를 따르고 있습니다.

전체 샘플 코드는 [링크](https://github.com/Ethan-IS/iOS-Modularization-Sample/tree/basic)에 있으니 참고 부탁드립니다.

![iOS 모듈화](/assets/img/tuist_02/module_basic.png)

조금 더 단순화 한 모듈 구조입니다.

기본적으로 앱이 있고, 그 아래에 Feature가 두개, 공통으로 참조하고 있는 Data, 맨 아래에는 Core가 있는 구조입니다.

---

그럼 아예 처음 빈 프로젝트를 시작한다고 생각하고 진행하겠습니다.

빈폴더에서 `tuist init` 명령을 시작합니다.

그러면 아래와 같은 폴더 구조가 생성되게 됩니다. (폴더명을 따라 최초 프로젝트가 생성됩니다)

![폴더구조](/assets/img/tuist_02/tuist_tree.png)

프로젝트가 생성 되었으면, `tuist edit` 명령어를 통해 manifest 구성을 위한 xcodeproj를 오픈합니다.

![tuist edit](/assets/img/tuist_02/tuist_edit.png)

최초로 생성한 모습은 위와 같습니다.

이제 우리는 여기에 다양한 모듈을 생성해 주어야 합니다.

최초에는 수동으로 생성 후 다음 블로그를 통해 stencil을 사용하는 방법에 대해 알아보겠습니다.

그럼 우리가 생각했던 구조대로 폴더 구조를 생성합니다.

![폴더구조](/assets/img/tuist_02/tuist_tree_02.png)

> Workspace 가 생성된 이유
>
> 여러 프로젝트를 하나로 구성하여 실행하기 위한 구성입니다.
>
> - workspace는 **프로젝트 및 기타 문서를 그룹화하여 함께 작업**할 수 있는 Xcode document입니다
> - workspace 에 **원하는 수의 Xcode 프로젝트 및 다른 파일들을 포함**할 수 있습니다
> - 각 Xcode 프로젝트의 파일을 구성하는 것 외에도 workspace는 포함된 프로젝트와 target 간의 **암시적(implicit) 및 명시적(explicit) 관계를 제공**합니다.

- Workspace.swift

![workspace_code](/assets/img/tuist_02/workspace_code.png)

- Workspace의 이름을 지정하고, Project.swift 파일들이 들어있는 폴더들의 경로를 지정해줍니다.

- App Folder

  - App의 Project.swift 가 존재하는 곳입니다.
  - 메인앱 관련 세팅을 여기서 하신다고 보시면 됩니다.

  ![app](/assets/img/tuist_02/app.png)

- Featrures 폴더
  - Feature를 하나로 등록하기 위한 Layer 폴더 아래에 개별 피쳐들이 존재하는 구조입니다.
  - Features의 Project.swift는 각 피쳐를 dependency로 갖고 있습니다.
  - 각 Featrue의 Project.swift는 피쳐에서 원하는 dependency 그리고 project 설정 구조를 갖습니다.

![features](/assets/img/tuist_02/features.png)

![feature_home](/assets/img/tuist_02/feature_home.png)

- Data 폴더
  - Features와 동일한 구조를 갖고 있습니다.
- Core
  - Features와 동일한 구조를 갖고 있습니다.

이제 기본적인 프로젝트 구조를 만드는 작업은 완성이 되었습니다.

실제로 이 프로젝트가 잘 생성이 되고 빌드가 되는지 확인해보아야 하는데요.

이부분은 `tuist edit` 을 통해 열린 xcodeproj 를 닫은 후, `tuist generate` 명령어를 통해 확인할 수 있습니다.

(`tuist install` 은 외부 dependency를 가져올때 사용됩니다)

![workspace](/assets/img/tuist_02/workspace.png)

그럼 이렇게 위와 같은 프로젝트 구조를 갖게 되는 워크스페이스가 생성됩니다.

![tuist edit](/assets/img/tuist_02/tuist_link.png)

오른쪽 패널을 확인해보면 이처럼 각 모듈들에서 만들어진 결과물이 본체인 App 과 잘 링킹된 것이 확인되는데요.

자세한 사항은 다음 게시물에서 같이 알아보도록 하겠습니다.

---

## 결론

생각했던 것 보다 적용하기 어렵지는 않았죠?

Tuist는 가볍게 시작하기 좋은 모듈화 툴이라고 생각합니다.

(Dynamic framework, static library 등 알아야 최적화가 되는 부분들이 많기는 하지만)

다음에는 모듈화를 제대로 진행하기 위해서 Clean Architecture에 대해 심층적으로 이해해보는 시간을 갖도록 하겠습니다.

---

## 참고링크

- [Workspace란 무엇인가](https://sujinnaljin.medium.com/xcode-workspace%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80-2140fda6ccd5)
- [Tuist 사용 중인 기업 블로그들](https://www.codenary.co.kr/techblog/list?tag=tuist)
