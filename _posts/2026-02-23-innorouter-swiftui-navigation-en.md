---
title: "InnoRouter Best Practices: Managing SwiftUI Navigation with Routes and Coordinators"
date: 2026-02-23 00:00:00 +0900
translation_key: innorouter-swiftui-navigation
lang: en
description: "Why InnoRouter is useful, how to replace view-local path mutation with route, store, and coordinator boundaries, and how InnoSample applies that pattern."
categories: [iOS, Swift, Architecture]
tags: [swift, iOS, navigation, swiftui, coordinator, deep-link, architecture, route]
author: ethan
toc: true
comments: true
---

## Why SwiftUI navigation gets hard in real apps

SwiftUI's navigation APIs are enough for small flows. In larger apps, navigation becomes an architecture problem.

- Route path state scatters across views.
- Deep links mix with screen code.
- Modal, push, and tab movement are handled differently.
- Cross-feature navigation creates direct imports.
- Navigation tests require booting UI.

[`InnoRouter`](https://github.com/InnoSquadCorp/InnoRouter) treats navigation as typed state and explicit command execution, not view-local side effects. Its strongest appeal is that **screen movement becomes data, and coordinators execute that data.**

## The boundary InnoRouter should own

InnoRouter should own:

- route stack state
- navigation command execution
- modal presentation authority
- tab coordinator state
- deep-link matching and planning
- navigation effect adapters
- host-less navigation tests

It should not own:

- business workflow state
- network retry or session lifecycle
- dependency graph construction
- feature-local alerts and confirmation dialogs
- authentication domain policy itself

InnoRouter answers "which screen should this feature show, and how?" The reason to move comes from feature logic; the UI renders the route.

## Installation and imports

```swift
dependencies: [
    .package(url: "https://github.com/InnoSquadCorp/InnoRouter.git", from: "4.2.1")
]
```

Most app code imports the umbrella product. Files using route macros import `InnoRouterMacros` explicitly.

```swift
import InnoRouter
import InnoRouterMacros

@Routable
enum PeopleRoute {
    case detail(UserSummary)
}
```

Keeping macro imports local means non-macro files do not pay unnecessary macro-plugin resolution cost.

## Best practice 1. Do not let views own path mutation

When SwiftUI views directly own `NavigationPath`, push and pop logic spreads through the view tree.

InnoRouter pushes the ownership outward:

- routes are declared as enums
- stores own route stacks
- hosts connect those stores to SwiftUI
- coordinators translate user intent into navigation commands

```swift
final class PeopleFeatureCoordinator {
    let navigationStore = NavigationStore<PeopleRoute>()
    let modalStore = ModalStore<PeopleModalRoute>()

    func showDetail(user: UserSummary) {
        navigationStore.send(.go(.detail(user)))
    }
}
```

Views render routes through `NavigationHost` or `ModalHost`. Push and pop become feature-boundary state, not local view state.

## Best practice 2. Leaf features should know only their own navigation

If one feature imports another feature's router directly, dependency cycles appear quickly. InnoSample solves this by mediating at the parent coordinator.

`PeopleFeature` does not push settings directly. Its reducer emits `pendingSettingsRequest`; `PeopleFeatureCoordinator` exposes a way to consume that request. The root `EntireTabCoordinator` performs the tab switch and asks `SettingsFeatureCoordinator` to show the detail.

```swift
@MainActor
@Observable
public final class EntireTabCoordinator: TabCoordinator {
    let peopleCoordinator: PeopleFeatureCoordinator
    let postsCoordinator: PostsFeatureCoordinator
    let settingsCoordinator: SettingsFeatureCoordinator

    func syncCrossFeatureNavigationFromPeople() {
        guard let request = peopleCoordinator.consumeSettingsRequest() else { return }
        selectedTab = .settings
        settingsCoordinator.showDetail(assigneeID: request.assigneeID)
    }

    func syncCrossFeatureNavigationFromSettings() {
        guard let request = settingsCoordinator.consumePeopleRequest() else { return }
        selectedTab = .people
        peopleCoordinator.showDetail(userID: request.userID)
    }
}
```

The key is separating runtime movement from compile-time dependency.

- Runtime: `People -> Settings -> People` movement is allowed.
- Compile time: `People -> EntireTab <- Settings` stays intact.

That is one of the strongest reasons to use InnoRouter in a multi-feature SwiftUI app.

## Best practice 3. Separate navigation intent from business state

Navigation can be the result of business logic, but the route stack itself is not business state.

A good flow is:

1. An InnoFlow reducer emits intent.
2. A feature coordinator consumes the intent.
3. The coordinator sends a route command to an InnoRouter store.
4. A SwiftUI host renders the route.

That split lets reducer tests run without a navigation framework and router tests run without domain use cases.

## Best practice 4. Use one language for modal, tab, and stack navigation

SwiftUI apps often scatter push, modal, and tab handling across `NavigationStack`, `.sheet`, and `TabView`. InnoRouter lets you describe them with the same route/coordinator mindset.

- `NavigationStore`: push stacks
- `ModalStore`: sheet and full-screen cover authority
- `TabCoordinator`: selected tab, content, and badge state
- `DeepLinkPipeline`: app-boundary URL planning

The products are split, but the idea is the same: navigation is data, and hosts execute it.

## Best practice 5. Plan deep links at the app boundary

Parsing deep links inside feature views scatters app-level route policy. InnoRouter's deep-link surface moves matching and planning to the app boundary.

In production, a good default is:

- parse URLs and plan routes at the root coordinator or app boundary
- pass only feature-level route intent into leaf features
- keep auth/session gating in app policy
- store and resume pending deep links explicitly

This becomes more valuable as the number of deep links grows.

## What you gain by adopting it

Used well, InnoRouter gives you:

- route definitions as typed data
- views that do not directly mutate path state
- host-less navigation tests
- fewer compile-time cycles between sibling features
- one mental model for modal, tab, push, and deep link flows
- a coordinator boundary for navigation side effects

In apps with many features, tabs, modals, and deep links, navigation becomes an explicit model instead of a side effect of view implementation.

## When it is a good fit

InnoRouter is a good fit for:

- SwiftUI apps with navigation spread across multiple features
- apps that combine tabs, modals, pushes, and deep links
- teams trying to reduce direct feature-to-feature imports
- teams that want navigation tests
- teams that prefer strict concurrency and typed route models

For a tiny app with only a few screens, SwiftUI's native navigation APIs may be enough.

## How InnoSample uses it

In [`InnoSample`]({% post_url 2026-04-03-innosample-inno-libraries-in-practice-en %}), InnoRouter lives in `Router` targets.

- `PeopleFeatureRouter` knows only People navigation.
- `PostsFeatureRouter` knows only Posts navigation.
- `SettingsFeatureRouter` knows only Settings navigation.
- `EntireTabCoordinator` mediates sibling movement.
- `FeatureContainer` wires leaf coordinators together.

That is the best way to use InnoRouter: **leaf features own their own routes, and cross-feature navigation belongs to a parent coordinator.** The app can move flexibly at runtime while compile-time dependencies stay clean.
