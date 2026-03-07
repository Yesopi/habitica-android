# Libraries & Dependencies Analysis — Habitica Android

> **Project:** Habitica Android  
> **Modules:** `Habitica` (main app), `common` (shared Android library), `shared` (Kotlin Multiplatform), `wearos` (Wear OS app), `build-logic` (custom Gradle plugins)  
> **Dependency management:** Gradle Version Catalog (`gradle/libs.versions.toml`)

---

## Table of Contents

1. [Networking](#1-networking)
2. [REST API & Serialization](#2-rest-api--serialization)
3. [Image Loading](#3-image-loading)
4. [Dependency Injection](#4-dependency-injection)
5. [Database](#5-database)
6. [Firebase & Google Services](#6-firebase--google-services)
7. [AndroidX Core & UI](#7-androidx-core--ui)
8. [Jetpack Compose](#8-jetpack-compose)
9. [Navigation](#9-navigation)
10. [Lifecycle & Architecture](#10-lifecycle--architecture)
11. [Kotlin & Coroutines](#11-kotlin--coroutines)
12. [Markdown Rendering](#12-markdown-rendering)
13. [In-App Purchases & Billing](#13-in-app-purchases--billing)
14. [Authentication & Credentials](#14-authentication--credentials)
15. [Analytics](#15-analytics)
16. [Wear OS](#16-wear-os)
17. [UI Utilities](#17-ui-utilities)
18. [Debugging & Quality](#18-debugging--quality)
19. [Build Tooling & Plugins](#19-build-tooling--plugins)
20. [Testing](#20-testing)
21. [Summary Table](#21-summary-table)

---

## 1. Networking

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **OkHttp** (`com.squareup.okhttp3:okhttp`) | 5.1.0 | Habitica, wearos | High-performance HTTP client used for all network communication with the Habitica API. |
| **OkHttp Logging Interceptor** (`com.squareup.okhttp3:logging-interceptor`) | 5.1.0 | Habitica, wearos | Logs HTTP request/response details for debugging network calls during development. |

These two libraries are bundled together as `bundles.okhttp` in the version catalog.

---

## 2. REST API & Serialization

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Retrofit** (`com.squareup.retrofit2:retrofit`) | 3.0.0 | Habitica, wearos | Type-safe HTTP client that converts REST API endpoints into Kotlin interfaces, simplifying API consumption. |
| **Retrofit Converter – Gson** (`converter-gson`) | 3.0.0 | Habitica | Converts JSON responses into Kotlin/Java objects using Gson in the main app module. |
| **Retrofit Converter – Moshi** (`converter-moshi`) | 3.0.0 | wearos | Converts JSON responses using Moshi in the Wear OS module for more efficient serialization. |
| **Moshi Kotlin** (`com.squareup.moshi:moshi-kotlin`) | 1.15.2 | wearos | Kotlin-friendly JSON serialization library used in the Wear OS module. |
| **Moshi Codegen** (`moshi-kotlin-codegen`) | 1.15.2 | wearos | Generates Moshi JSON adapters at compile time via KSP, eliminating reflection overhead. |
| **Gson** (`com.google.code.gson:gson`) | 2.13.1 | build-logic | JSON parsing used in custom Gradle build-logic plugins for configuration handling. |

---

## 3. Image Loading

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Coil** (`io.coil-kt.coil3:coil`) | 3.3.0 | common | Core image loading library — lightweight, Kotlin-first, coroutine-based. |
| **Coil Compose** (`coil-compose`) | 3.3.0 | Habitica | Jetpack Compose integration for loading images directly in composable functions. |
| **Coil OkHttp** (`coil-network-okhttp`) | 3.3.0 | common | Uses OkHttp as the network backend for image downloads (shared HTTP configuration). |
| **Coil Cache Control** (`coil-network-cache-control`) | 3.3.0 | common | Adds HTTP cache-control header support for intelligent image caching. |
| **Coil GIF** (`coil-gif`) | 3.3.0 | common | Adds GIF and animated image decoding support (used for animated avatars/pets). |

---

## 4. Dependency Injection

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Hilt Android** (`com.google.dagger:hilt-android`) | 2.57.1 | Habitica, wearos | Compile-time dependency injection framework built on Dagger — provides scoped injection for Activities, Fragments, ViewModels, and Services. |
| **Hilt Compiler** (`hilt-compiler`) | 2.57.1 | Habitica, wearos | Annotation processor (via KSP) that generates the Hilt DI component code at compile time. |
| **javax.annotation-api** | 1.3.2 | Habitica | Provides `@Generated` and other javax annotations required by Hilt-generated code. |

---

## 5. Database

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Realm** (`io.realm:realm-gradle-plugin`) | 10.19.0 | Habitica (plugin) | Mobile-first object database for offline-first data storage. Stores user data, tasks, habits, and other game entities locally with automatic synchronization support. |

Realm is applied as a Gradle plugin (`realm-android`) in the main app module.

---

## 6. Firebase & Google Services

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Firebase BOM** | 34.1.0 | Habitica, wearos | Bill of Materials that manages consistent versions across all Firebase libraries. |
| **Firebase Crashlytics** | *(managed by BOM)* | Habitica, wearos | Automatic crash reporting — collects, organizes, and prioritizes crash reports in production. |
| **Firebase Cloud Messaging** | *(managed by BOM)* | Habitica, wearos | Push notification delivery for reminders, party invitations, and other real-time notifications. |
| **Firebase Remote Config** | *(managed by BOM)* | Habitica, wearos | Server-side feature flags and configuration — allows changing app behavior without an app update. |
| **Firebase Performance** | *(managed by BOM)* | Habitica, wearos | Monitors app performance metrics (startup time, network latency, rendering). |
| **Google Play Services – Auth** | 21.4.0 | Habitica, wearos | Google Sign-In integration for user authentication. |
| **Google Play Services – Wearable** | 19.0.0 | Habitica, wearos | Data synchronization layer between the phone app and Wear OS companion app. |
| **Google Services Plugin** | 4.4.3 | *(plugin)* | Processes `google-services.json` to configure Firebase at build time. |

---

## 7. AndroidX Core & UI

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Core KTX** (`androidx.core:core-ktx`) | 1.17.0 | Habitica, common, wearos | Kotlin extensions for Android framework APIs (intents, bundles, views, etc.). |
| **AppCompat** (`androidx.appcompat:appcompat`) | 1.7.1 | Habitica, common, wearos | Backward-compatible Material Design support for older Android versions. |
| **Material Components** (`com.google.android.material:material`) | 1.12.0 | Habitica, common | Material Design UI components (buttons, cards, dialogs, bottom sheets, etc.). |
| **RecyclerView** (`androidx.recyclerview`) | 1.4.0 | Habitica, common, wearos | Efficient scrollable list/grid rendering for tasks, items, and chat messages. |
| **SwipeRefreshLayout** | 1.1.0 | Habitica | Pull-to-refresh gesture support on list screens. |
| **Preference KTX** (`androidx.preference`) | 1.2.1 | Habitica, wearos | Settings/preferences screens with built-in Material Design UI. |
| **ConstraintLayout** | 2.2.1 | wearos | Flexible layout system for complex screen compositions on Wear OS. |
| **CoordinatorLayout** | 1.3.0 | wearos | Coordinates scroll-based animations and collapsing toolbars. |
| **FlexboxLayout** (`com.google.android.flexbox`) | 3.0.0 | Habitica | CSS-style flexbox layout for responsive UI arrangements (tag displays, reward grids). |
| **Core Splash Screen** | 1.2.0-rc01 | wearos | Android 12+ splash screen API for a consistent app launch experience. |

The `bundles.design` group packages AppCompat, Material, RecyclerView, Preference, and SwipeRefreshLayout together.

---

## 8. Jetpack Compose

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Compose Animation** (`androidx.compose.animation`) | 1.9.0 | Habitica, common | Animation APIs for Compose UI transitions and effects. |
| **Material 3** (`androidx.compose.material3`) | 1.3.2 | Habitica, common | Material Design 3 components for Compose (the modern design system). |
| **Activity Compose** (`activity-compose`) | 1.10.1 | Habitica, common | Integration bridge between `ComponentActivity` and Compose. |
| **Runtime LiveData** (`compose.runtime:runtime-livedata`) | 1.9.0 | Habitica, common | Observes `LiveData` as Compose `State` to bridge existing ViewModel patterns into Compose. |
| **UI Tooling** (`compose.ui:ui-tooling`) | 1.9.0 | Habitica, common | Preview support and layout inspector for Compose UIs during development. |
| **UI Text Google Fonts** | 1.9.0 | Habitica, common | Downloadable Google Fonts integration for Compose text styling. |
| **ViewModel Compose** (`lifecycle-viewmodel-compose`) | 2.9.2 | Habitica | Provides `viewModel()` composable function to access ViewModels from Compose. |
| **Paging Compose** (`paging-compose`) | 3.3.6 | Habitica | Compose integration for the Paging library — lazy loading of large data sets in composable lists. |
| **Kotlin Compose Plugin** | 2.2.10 | *(plugin)* | Kotlin compiler plugin enabling Compose compiler transformations. |

---

## 9. Navigation

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Navigation Fragment KTX** | 2.9.3 | Habitica, wearos | Handles fragment-based navigation with a navigation graph, back stack management, and deep links. |
| **Navigation Common KTX** | 2.9.3 | common | Shared navigation types and utilities used by the common library module. |
| **Navigation Runtime KTX** | 2.9.3 | common | Runtime navigation controller for programmatic navigation. |
| **Navigation Safe Args Plugin** | 2.9.3 | *(plugin)* | Generates type-safe argument classes for passing data between navigation destinations. |

---

## 10. Lifecycle & Architecture

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Lifecycle ViewModel KTX** | 2.9.2 | Habitica, wearos | MVVM ViewModel component — survives configuration changes (rotation, etc.). |
| **Lifecycle LiveData KTX** | 2.9.2 | Habitica, wearos | Observable data holder for UI-layer reactive data streams. |
| **Lifecycle Runtime KTX** | 2.9.2 | wearos | Lifecycle-aware coroutine scopes (`lifecycleScope`, `repeatOnLifecycle`). |
| **Lifecycle Common Java8** | 2.9.2 | Habitica, wearos | Default interface method support for lifecycle observers. |
| **Lifecycle Process** | 2.9.2 | Habitica | Provides `ProcessLifecycleOwner` for detecting app foreground/background transitions. |
| **Fragment KTX** | 1.8.9 | Habitica | Kotlin extensions for Fragment (e.g., `viewModels()` delegate, `by lazy` fragment arguments). |
| **Paging Runtime KTX** | 3.3.6 | Habitica | Loads large data sets incrementally (paginated task lists, chat messages). |

---

## 11. Kotlin & Coroutines

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Kotlin Stdlib** | 2.2.10 | Habitica, wearos | Kotlin standard library with core language utilities. |
| **Kotlin Reflect** | 2.2.10 | wearos | Runtime reflection support required by some serialization and DI frameworks. |
| **Kotlinx Coroutines Core** | 1.10.2 | Habitica, shared, wearos | Structured concurrency for asynchronous programming — used across all modules. |
| **Kotlinx Coroutines Android** | 1.10.2 | Habitica, wearos | Android `Main` dispatcher for running coroutines on the UI thread. |
| **Desugar JDK Libs** | 2.1.5 | Habitica | Backports Java 8+ APIs (java.time, streams, etc.) to older Android versions (minSdk 26). |

---

## 12. Markdown Rendering

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Markwon Core** | 4.6.2 | common | Renders Markdown text as Android `Spannable` content in TextViews. Used for in-app descriptions, chat, and quest text. |
| **Markwon Strikethrough** (`ext-strikethrough`) | 4.6.2 | common | Adds `~~strikethrough~~` syntax support. |
| **Markwon Image** | 4.6.2 | common | Inline image rendering within Markdown content. |
| **Markwon Linkify** | 4.6.2 | common | Automatic URL, email, and phone number detection and linking. |
| **Markwon Recycler** | 4.6.2 | common | RecyclerView adapter for rendering long Markdown documents with recycled views. |

These five libraries are bundled as `bundles.markwon`.

---

## 13. In-App Purchases & Billing

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Google Play Billing KTX** (`billing-ktx`) | 8.0.0 | Habitica | Handles in-app purchases and subscriptions (gems, subscription plans) through Google Play. Provides Kotlin coroutine wrappers for the billing flow. |

---

## 14. Authentication & Credentials

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Credentials** (`androidx.credentials`) | 1.5.0 | Habitica | Android Credential Manager API for modern sign-in flows (passkeys, passwords). |
| **Credentials Play Services Auth** | 1.5.0 | Habitica | Play Services backend for the Credential Manager, enabling Google Sign-In support. |
| **Google Identity – GoogleID** | 1.1.1 | Habitica | Google ID Token credential support for one-tap sign-in and sign-up. |
| **Google Play Services Auth** | 21.4.0 | Habitica, wearos | Legacy Google Sign-In SDK used alongside the newer Credentials API. |

---

## 15. Analytics

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Amplitude Analytics Android** | 1.22.2 | Habitica | User behavior analytics — tracks events, user engagement, and feature usage to support product decisions. |

---

## 16. Wear OS

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Wear** (`androidx.wear:wear`) | 1.3.0 | wearos | Core Wear OS UI components (curved layouts, round screen support). |
| **Wear Input** (`androidx.wear:wear-input`) | 1.1.0 | wearos | Rotary input (crown) and button handling for Wear OS devices. |
| **Play Services Wearable** | 19.0.0 | Habitica, wearos | Data layer for syncing tasks, stats, and notifications between the phone and watch. |

---

## 17. UI Utilities

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **Shimmer** (`com.facebook.shimmer`) | 0.5.0 | Habitica | Animated shimmer/loading placeholder effect shown while content is loading. |
| **Google Play In-App Review** (`review` + `review-ktx`) | 2.0.2 | Habitica | Prompts users to rate the app within the app (without leaving to the Play Store). |

---

## 18. Debugging & Quality

| Library | Version | Module(s) | Purpose |
|---------|---------|-----------|---------|
| **LeakCanary** (`com.squareup.leakcanary`) | 2.14 | Habitica (debug) | Automatically detects memory leaks in debug builds and provides heap dump analysis. Only included in debug builds to avoid production overhead. |

---

## 19. Build Tooling & Plugins

| Plugin / Tool | Version | Purpose |
|---------------|---------|---------|
| **Android Gradle Plugin (AGP)** | 8.12.1 | Core Android build system — compiles, packages, signs APK/AAB files. |
| **Kotlin Android Plugin** | 2.2.10 | Kotlin language support and compilation for Android targets. |
| **Kotlin Multiplatform** | 2.2.10 | Enables code sharing across Android, iOS, and JS in the `shared` module. |
| **KSP (Kotlin Symbol Processing)** | 2.2.10-2.0.2 | Faster annotation processing replacement for kapt — used by Hilt, Moshi, and other code generators. |
| **Detekt** | 1.23.8 | Static code analysis tool for Kotlin — enforces code style and detects code smells. |
| **ktlint** (via Gradle plugin) | 13.1.0 | Kotlin code formatter and linter — ensures consistent code style across the project. |
| **Navigation Safe Args** | 2.9.3 | Generates type-safe classes for navigation arguments. |
| **Realm Plugin** | 10.19.0 | Gradle plugin that configures the Realm database integration. |
| **Firebase Crashlytics Plugin** | 3.0.6 | Uploads mapping files and configures crash reporting at build time. |
| **Firebase Performance Plugin** | 2.0.1 | Instruments the app to collect performance metrics automatically. |
| **Google Services Plugin** | 4.4.3 | Processes `google-services.json` for Firebase configuration. |
| **JaCoCo** (report aggregation) | *(built-in)* | Code coverage report aggregation across modules. |
| **Foojay Toolchain Resolver** | 1.0.0 | Automatically resolves and downloads JDK toolchains for builds. |
| **HabitRPG Convention Plugin** | 0.1.0 | Custom build-logic plugin applying shared Detekt, ktlint, and project conventions. |
| **HabitRPG Application Plugin** | 0.1.0 | Custom build-logic plugin applying signing config, versioning, and flavor configuration. |

---

## 20. Testing

| Library | Version | Scope | Purpose |
|---------|---------|-------|---------|
| **Kotest Runner JUnit5** | 6.0.1 | `testImplementation` | Kotlin-native test framework runner — BDD-style test specs with JUnit 5 platform integration. |
| **Kotest Assertions** | 6.0.1 | `testImplementation` / `androidTestImplementation` | Expressive assertion library (e.g., `shouldBe`, `shouldContain`, `shouldThrow`). |
| **MockK** | 1.14.5 | `testImplementation` | Kotlin-first mocking library — creates mocks, stubs, and verifies interactions with idiomatic Kotlin syntax. |
| **MockK Android** | 1.14.5 | `testImplementation` / `androidTestImplementation` | Android-specific MockK variant supporting Android framework class mocking. |
| **MockK Agent** | 1.14.5 | `androidTestImplementation` | JVM instrumentation agent for MockK enabling inline mocking in instrumented tests. |
| **Kaspresso** | 1.6.0 | `androidTestImplementation` | Readable and stable Android UI test framework built on Espresso — reduces flakiness with automatic retries and screenshots. |
| **Kaspresso Compose** | 1.6.0 | `androidTestImplementation` | Kaspresso support for testing Jetpack Compose UIs. |
| **Turbine** | 1.2.1 | `testImplementation` (wearos) | Flow testing library — easily tests Kotlin `Flow` emissions with structured timeout-based assertions. |
| **AndroidX Test Core** | 1.7.0 | `testImplementation` | Android testing utilities (Robolectric compatibility, `ApplicationProvider`). |
| **AndroidX Test Core KTX** | 1.7.0 | `androidTestImplementation` | Kotlin extensions for Android test core APIs. |
| **AndroidX Test Runner** | 1.7.0 | `androidTestImplementation` | Instrumented test runner (`AndroidJUnitRunner`). |
| **AndroidX Test Rules** | 1.7.0 | `androidTestImplementation` | Test rules for activity scenarios, service rules, etc. |
| **AndroidX Test JUnit KTX** | 1.3.0 | `androidTestImplementation` | JUnit integration for AndroidX testing. |
| **AndroidX Test Monitor** | 1.8.0 | `debugImplementation` | Test instrumentation monitor for debugging test failures. |
| **AndroidX Test Orchestrator** | 1.6.1 | `androidTestUtil` | Runs each instrumented test in an isolated process to prevent shared state issues. |
| **Fragment Testing** | 1.8.9 | `debugImplementation` | Launches fragments in isolation for testing. |
| **Kotlin Test** | 2.2.10 | `commonTest` (shared) | Multiplatform test assertions used in the shared KMP module. |

---

## 21. Summary Table

| Category | Library Count | Key Libraries |
|----------|:------------:|---------------|
| Networking | 2 | OkHttp, Logging Interceptor |
| REST API & Serialization | 6 | Retrofit, Gson, Moshi |
| Image Loading | 5 | Coil (core, Compose, OkHttp, cache, GIF) |
| Dependency Injection | 3 | Hilt, Hilt Compiler, javax.annotation |
| Database | 1 | Realm |
| Firebase & Google Services | 8 | Crashlytics, Messaging, Config, Performance, Auth, Wearable |
| AndroidX Core & UI | 10 | Core KTX, AppCompat, Material, RecyclerView, FlexboxLayout |
| Jetpack Compose | 9 | Material 3, Activity Compose, Animation, Paging Compose |
| Navigation | 4 | Navigation Fragment, Common, Runtime, Safe Args |
| Lifecycle & Architecture | 7 | ViewModel, LiveData, Process, Fragment, Paging |
| Kotlin & Coroutines | 5 | Stdlib, Reflect, Coroutines Core, Coroutines Android, Desugar |
| Markdown | 5 | Markwon (core, strikethrough, image, linkify, recycler) |
| Billing | 1 | Google Play Billing KTX |
| Authentication | 4 | Credentials, GoogleID, Play Auth |
| Analytics | 1 | Amplitude |
| Wear OS | 3 | Wear, Wear Input, Play Services Wearable |
| UI Utilities | 2 | Shimmer, In-App Review |
| Debugging | 1 | LeakCanary |
| Build Plugins | 16 | AGP, Kotlin, KSP, Detekt, ktlint, Firebase, Realm, custom |
| Testing | 16 | Kotest, MockK, Kaspresso, Turbine, AndroidX Test |
| **Total** | **~109** | |

---

> **Note:** Version numbers are sourced from `gradle/libs.versions.toml`. Firebase sub-library versions are managed by the Firebase BOM (`34.1.0`) and are not individually specified.
