---
title: "Expandable modularization with Tuist Part 3 - Understanding Clean Architecture"
author: ethan
date: 2024-05-13 17:40:00 +0900
translation_key: clean-architecture
lang: en
categories: [Architecture]
tags: [tutorial, swift, tuist, iOS]

pin: false
image:
  path: /assets/img/tuist_03/cover.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Understanding modularization
---

## Introduction


As iOS development has matured, teams have moved beyond presentation-layer patterns such as MVC, MVVM, and MVP. There is now more interest in multi-layered architectures like Clean Architecture to improve separation of concerns.

Our team used to work in a single project, but now we run a multi-project setup and create sample apps when needed.

In practice, this means the app is no longer split into only UI and logic. We separate it into UI, UI logic, domain logic, data logic, and remote/local layers.

In this post, we'll review Clean Architecture, its pros and cons, and how to apply it in iOS development.

## Why design architecture up front?


Architecture defines how a system is structured, how components interact, and how responsibilities are separated.

In practical product development, a useful architecture gives your team:

- easier maintenance,
- safer scaling,
- better testability,
- and higher reusability.


In other words, architecture describes how a piece of software is structured and how it operates.

Usually, one team owns one service. In that situation, what goals should the team pursue?

1. It must be easy to maintain.

2. Software should scale safely.

3. It must be testable and stable.

4. It must be easy to reuse (to improve delivery speed).

Designing architecture means agreeing on shared principles for:

- how to split code,
- how to organize dependencies,
- and how to place responsibilities across layers.

So how do we design for those goals?

1. Testability


⇒ Each module should stay independent and be easy to replace with mocks in tests.

2. Reusability


⇒ Make code depend on abstractions (interfaces), not concrete implementations.

3. Safe scalability


⇒ Organize dependencies so high-volatility areas depend on low-volatility areas, minimizing change impact.

4. Maintainability


⇒ If the first three are done well, maintenance becomes naturally easier.

With that context, let's look at Clean Architecture and how the InnoSquad team adapted it.

<br>

## So what is [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)?


Clean Architecture is a software architecture methodology proposed by Robert C. Martin (Uncle Bob).

The goal is to make software systems flexible, testable, and maintainable.

![clean architecture](/assets/img/tuist_03/clean-architecture.png)

### Dependency Rule - Top priority rule


- Nothing in the inner circle knows anything about the outer circle.

  - That is, the names of items declared in the outer Circle must not be mentioned in the code of the inner Circle.

  - That is, data types used in the outer Circle should not be used in the inner Circle.

- Nothing in the outer Circle affects the inner Circle.

- Dependencies between layers should go from the outside in (abstraction).


⇒ Increases testability by making the external (UI or DB) depend on the internal business logic (which does not change frequently). (The internal logic becomes independent and can be stably replaced with mock data.)

### Entities


- The business object of the application.

- It may contain a method and may be a data structure.


### UseCase


- Application-specific business rules.

- It is responsible for coordinating the flow of data between entities.


### Interface Adapters


- If it is a GUI, the MVC architecture is included in this part.

- If it is a database gateway, the connection to the database must end at this layer.


### Frameworks & Drivers


- It consists of tools such as Database and Web Framework.

- Typically, this layer is not written except code that communicates with the interface adapter.


### Additional Notes


- There is no rule that the model must contain exactly four circles.

- However, the Dependency Rule must always be applied.

- The dependency direction of code always points internally.

- The further you go inward, the more abstract and encapsulated the software becomes.


### Advantages


- The further inward you go, the less dependent the code is on frameworks and external systems. (Higher testability)

- Changes to specific features have less impact on the overall system. (Better maintainability)

- As a result, teams can adapt requirements more flexibly. (More reliable scalability)


### Drawbacks


- Changes in inner policies may still require updates in outer layers.

- Also, the initial setup and wiring between layers adds upfront complexity.


<br>

## So how should we design the architecture and modularize it in iOS?


From the process above, we concluded that the goal is to build code that is maintainable, reusable, and testable.

The first design decision is layering.<br>
Clear layer boundaries separate concerns and responsibilities, which improves both maintenance and testing.

So how should we split the layers?<br>
Below is the structure our InnoSquad team adopted.

An app renders UI from data, receives user input, and produces new data or navigation results.<br>
From that flow, we derived the layer structure below.

![module architecture](/assets/img/tuist_03/module-architecture.png)

### App


- Entry point that launches the app.

- Contains initial app setup, including root screen and window configuration.

- Includes `@main` and `Bundle.main`.


### Features


A place that composes multiple feature modules.<br>
You can think of it as the place where each feature is registered for DI.<br>
This layer can depend on UI frameworks such as SwiftUI or UIKit.

#### UI


- Renders data on the screen.

- User interactions start here.

- In iOS terms, this includes `View` and `ViewController`.

- The UI layer handles lightweight UI logic (for example: routing on button tap, showing toasts on selection).


#### Presentation


- Contains `ViewModel`, `UIState`, and presentation models used by the screen.

- Communicates with the Domain Layer.

- The ViewModel exposes observable UI state and receives UI events through methods.

  - When an event arrives, the ViewModel handles it and updates `UIState` (events are not applied directly in the UI).

  - The ViewModel maps outputs from lower layers into UI-friendly state.

- Use ViewModel only on screen-level Views.

- Models should keep only the values needed by the screen, not every value from the Domain Layer.

- UI State must be immutable.

  - This guarantees a stable snapshot of app state at each point in time.

  - That lets the UI focus on rendering state.

  - Therefore, you should not modify UI state directly from the UI (unless the UI itself is the only source of data).


#### Notes


- The UI's role should be solely to use and display UI state. (UDF: Unidirectional Data Flow)

  - ViewModel holds and exposes state to be used in the UI.

  - The UI notifies the ViewModel of user events.

  - ViewModel handles tasks and updates state.

  - The updated state is fed back to the UI to be rendered.

  - The above is repeated for every event that causes a state change.

- ViewModel should not know about types such as `navigationController` or `UIApplication`.

  - These things belong to UI logic.


### Domain Layer


This layer encapsulates the application's core business logic and rules.<br>
It models the problem domain the system solves and can be considered the "brain" of the app.

- Define business rules: implement core processes and policy logic.

  - Typically you use a class called UseCase.

  - Each UseCase is responsible for one business logic.

    - Additionally, it cannot contain mutable data. Changeable data must be processed in the UI or Data Layer.

    - Complex calculations are performed in the Data Layer for reuse or caching.

    - ex> The “order processing” use case may involve the process of a user ordering a product and completing payment.

- Create domain models: objects that represent business entities and relationships (for example, users and orders).

- Validation: Validate data based on domain rules.

- Maintain independence: The domain layer must remain independent and not dependent on other layers (e.g. data access layer, presentation layer). This minimizes the impact of modifications to business logic on other layers.


### Data Layer


This layer primarily manages interactions with data stores.<br>
It is responsible for reading and writing data from sources such as databases, file systems, or external APIs.

- Data access abstraction: Abstracts database queries, schema management, data transaction processing, and more, freeing the business logic layer from the complexities of data storage.

- CRUD operations: Perform basic data manipulation operations such as Create, Read, Update, and Delete data.

- Data integration: combines data from multiple sources into a consistent model.

- Data caching: Cache frequently used data to improve performance.


The data exposed by this layer must be immutable.<br>
This eliminates the risk of creating inconsistent values. (eliminates the possibility of manipulation from elsewhere)

### Remote / Local Layer


- Implements the data sources used by the Data Layer.

- Remote defines and executes server communication contracts (API definitions).

- Local handles persistence with FileManager, UserDefaults, and Keychain.


### Core, Util, ThirdParty


These are cross-cutting modules used throughout the service.

#### Core


- Includes foundational modules such as Network and DesignSystem.

- Contains low-churn modules shared by many parts of the system.

- In our setup, these modules can be consumed directly without separate interface modules.


#### Util


- Contains utility functions used across UI/Foundation code.

- Utilities are often exposed through extensions; if behavior has a clear domain boundary, prefer a dedicated type.

- These modules are designed for broad reuse.


#### ThirdParty


- Contains adapters and interfaces for third-party SDKs.

- Third-party APIs change over time, and calling them directly everywhere hurts maintainability.

- Protocol-based interfaces keep the SDK boundary separate from implementation details.

- Code should use this module's interface instead of calling third-party SDKs directly.


---

## Conclusion


In this post, we looked at one practical way to modularize iOS applications.<br>
We revisited why architecture decisions matter and how Clean Architecture helps define layer boundaries and dependency flow.<br>
Using those principles, we outlined how to split layers and modules in a production iOS codebase.

There is no single "correct" structure for every team or product.<br>
But if responsibilities are clearly separated by layer, and dependencies consistently point inward, you are moving in the right direction.

In the next post, we'll move from principles to implementation and build this structure with Tuist.

---

## References


- [The Clean Code Blog](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

- [Android App Architecture Guide](https://developer.android.com/topic/architecture?hl=ko)

- [Core Ideas of Clean Architecture (by Gomtwigim)](https://iamchiwon.github.io/2020/08/27/main-thought-of-clean-architecture/#google_vignette)
