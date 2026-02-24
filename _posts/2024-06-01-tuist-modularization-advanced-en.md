---
title: "Expandable modularization with Tuist Part 4 - Modularization in practice"
author: ethan
date: 2024-06-01 17:40:00 +0900
translation_key: tuist-modularization-advanced
lang: en
categories: [Architecture]
tags: [tutorial, swift, tuist, iOS]

pin: false
image:
  path: /assets/img/tuist_04/cover.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Understanding modularization
---

## Introduction


In this final part, we'll build a project structure that is practical for real production work.

We'll assume multiple developers are working in parallel and that the app has at least two feature modules.

As covered in [the previous post](https://ethan-is.github.io/posts/clean-architecture/), this guide builds a Tuist project using the structure below.

![module-architecture](/assets/img/tuist_04/module-architecture.png)

Before creating the project, let's review principles for adding modules, the internal structure of feature modules, and common pitfalls.

## Design Principles for Adding Modules


- Choose the appropriate tier for each module (Feature, Core, Layer, and so on).

- Decide whether to split interface and implementation.

- Choose framework/library type based on usage (`dynamic` vs `static`).


### Notes


- Design patterns such as MVVM and MVP are related to Features.

- Remember that modularization operates at a larger scale than presentation-layer patterns.


## Feature module


Feature modules are typically split into a UI-rendering area and a data-handling area.<br>
To prevent circular dependencies and support sample apps and tests, the resulting structure is slightly more complex.

![feature-module](/assets/img/tuist_04/feature-module.png)

**Interface**

- Provides externally accessible interfaces and models for the feature's UI entry points.


**UI**

- Holds the UI portion of the feature. (Goal: keep UI as dumb as possible.)


**Presentation**

- Handles UI actions and connects to the domain layer. (In MVVM, this is where `Model` and `ViewModel` live.)


**Testing**

- Provides mock data.

- Provides code for use in the Example app.


**Tests**

- Contains unit tests and UI tests.


**Example**

- A lightweight app used to quickly validate feature behavior.


## Cautions


### Circular references between modules


For example, assume there is navigation from Settings to Profile, and also from Profile back to Settings.<br>
If implemented directly, the Settings module depends on Profile while Profile also depends on Settings.<br>
This is an inter-module circular dependency.

![feature-module-2](/assets/img/tuist_04/feature-module-2.png)

When circular references occur, clear boundaries between components disappear and chain changes may occur.

Uncle Bob emphasizes that dependencies between components should be acyclic (ADP: Acyclic Dependencies Principle). We continuously check our dependency graph for loops and break them when they appear.

### Solution for circular references between modules


Case 1) Resolve by creating a route module

- Problem → Every feature module must depend on the route module, which can make that module too central and heavy.


Case 2) Break dependencies through interfaces (DIP: Dependency Inversion Principle)

- Separate modules into interface modules and implementation modules.

- Each implementation module resolves circular references by referring to the interface module.


![feature-module-3](/assets/img/tuist_04/feature-module-3.png)

## Create a Real Tuist Project


Now that we've covered the caveats, let's move on to building an actual Tuist project.

Starting from [the basic structure](https://github.com/Ethan-IS/iOS-Modularization-Sample/tree/basic), which was created last time, we aim to create [the advanced structure](https://github.com/Ethan-IS/iOS-Modularization-Sample/tree/advanced).

### Workspace.swift


![workspace_swift](/assets/img/tuist_04/workspace_swift.png)

- Create a workspace.

  - Register projects grouped as App, Features, Layers, Cores, ThirdParties, and Utils. (You can think of the image above as one project map.)


### App


![app_swift](/assets/img/tuist_04/app_swift.png)

- The app setup is mostly the same as in the previous post.

  - It depends on the top-level project of each layer.


### Features


![features_swift_01](/assets/img/tuist_04/features_swift_01.png)

#### Features module


Each feature has an Interface module and UI module.<br>
This is where we configure dependency injection.

![features_swift_02](/assets/img/tuist_04/features_swift_02.png)
![features_swift_03](/assets/img/tuist_04/features_swift_03.png)

#### Individual Feature


- Create a project using a predefined Project extension.

- Creates `Interface`, `UI`, `Presentation`, `Testing`, and `Tests` modules. (`Example` is added when needed.)


- Interface

  - Uses a dynamic framework form without extra dependencies.

- UI

  - Receives required dependencies and includes `NeedleFoundation`, interfaces, and presentation dependencies.

- Presentation

  - Receives required dependencies, including domain modules for data retrieval.


### Layers


![layers_swift_01](/assets/img/tuist_04/layers_swift_01.png)
![layers_swift_02](/assets/img/tuist_04/layers_swift_02.png)
![layers_swift_03](/assets/img/tuist_04/layers_swift_03.png)

- The current layer structure has Domain, Data, and Remote structures.

  - Layer-level DI is configured across the entire app.

- The ownership relationship is Domain ← Data ← Remote.

  - Domain is a dynamic framework

  - Data and Remote are static libraries.


### Cores


![cores_swift_01](/assets/img/tuist_04/cores_swift_01.png)
![cores_swift_02](/assets/img/tuist_04/cores_swift_02.png)
![cores_swift_03](/assets/img/tuist_04/cores_swift_03.png)

- In Core, modules do not depend on each other.

  - Each module is consumed directly where needed.

  - In this structure they are dynamic frameworks, but static libraries are also valid when needed.


### ThirdParties


![third_parties_swift_01](/assets/img/tuist_04/third_parties_swift_01.png)
![third_parties_swift_02](/assets/img/tuist_04/third_parties_swift_02.png)
![third_parties_swift_03](/assets/img/tuist_04/third_parties_swift_03.png)

- ThirdParty modules are created when interface-based wrapping is needed.

  - Wrapped third-party libraries are not called directly elsewhere; only internal implementation modules reference them.

  - Access these modules through interfaces.


### Utils


![utils_swift_01](/assets/img/tuist_04/utils_swift_01.png)
![utils_swift_02](/assets/img/tuist_04/utils_swift_02.png)
![utils_swift_03](/assets/img/tuist_04/utils_swift_03.png)

- Like Core, Util modules do not depend on each other.

  - Each module is consumed directly where needed.


### Helper extensions


![helper_extensions](/assets/img/tuist_04/helper_extensions.png)

- As the number of modules grows, hard-coded module names/strings become more error-prone.

- Helper extensions reduce those manual errors.


## Write DI and basic code inside the app


We previously checked how to create a project through `tuist edit`.

Now let's implement DI and verify that DIP works in practice.

### DI Authoring with NeedleFoundation


- We use [Needle](https://github.com/uber/needle) as the DI framework.


![app](/assets/img/tuist_04/app.png)

- Feature-level DI is configured in `Features`.

- Layer-level DI is configured across the app.

- Third-party DI is configured through `App` or `ThirdParties`, depending on context.


### A closer look at the Feature module


![feature_detail_01](/assets/img/tuist_04/feature_detail_01.png)
![feature_detail_02](/assets/img/tuist_04/feature_detail_02.png)

Define the protocol for creating the home screen in `Interface`.<br>
Implement that interface in the `UI` module.

![feature_detail_03](/assets/img/tuist_04/feature_detail_03.png)

In each screen, `Builder` and `ViewModel` are injected to handle navigation.

![feature_detail_04](/assets/img/tuist_04/feature_detail_04.png)

The ViewModel declares required `UseCase` dependencies and receives them via injection.

---

## Conclusion


From [Part 1](https://ethan-is.github.io/posts/understanding-modularity/) to Part 4, we covered a practical end-to-end approach to iOS modularization.<br>
We walked through why modularization matters, how to structure modules, and how concepts like DIP, DI, and Clean Architecture fit together.

Next, our iOS team will dig deeper into Swift Concurrency.<br>
We'll continue with topics such as `Actor` and `Sendable`, focusing on practical usage in production code.

This concludes the modularization series, and we'll return with the next topic soon.

---

## References


- [Karrot Engineering Blog](https://medium.com/daangn/tuist-%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%B4-%EB%AA%A8%EB%93%88-%EA%B5%AC%EC%A1%B0-%EC%9E%90%EB%8F%99%ED%99%94%ED%95%98%EA%B8%B0-f200992d4bf2)

- [Toss Engineering Blog](https://toss.tech/article/slash23-iOS)
