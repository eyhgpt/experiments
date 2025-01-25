# Introduction

`DJI` (the drone company)'s `Mobile-SDK-Android-V5` project serves as an excellent example of
utilizing the DJI Mobile SDK to control DJI drones for complex scientific and industrial tasks, as
well as for developing sophisticated user interfaces for Android apps.

![dji_fly_planning](/diagrams/dji_fly_video_recording.png)
![dji_fly_planning](/diagrams/dji_fly_route_planning.png)
*Sourced from
[support.dji.com](https://support.dji.com/help/content?customId=en-us03400007343&lang=en&re=AU&spaceId=34)
© DJI*

> [!NOTE]
> The screenshots above are from the DJI Fly app, which is also built on top of the same Mobile-SDK.

The project demonstrates sophisticated UI components tailored for aviation control, mapping,
navigation, video recording, and error messaging. In terms of Android architecture and library
choices, the project utilizes `ViewModel`, `rxJava`, `LiveData`, and `Views`, while notably not
incorporating dependency injection (DI), coroutines, or `Jetpack Compose`.

![dji_msdk_v5_architecture](/diagrams/dji_msdk_v5_architecture.png)
*Sourced from https://developer.dji.com/mobile-sdk/ © DJI*

This article aims to highlight the strengths of this sample application and suggest areas for
improvement, based on my personal perspective. However, I acknowledge that the rapid evolution of
Android app architecture can make it challenging to refactor existing codebases to align with the
latest best practices.

# `android-sdk-v5-sample` Project

## Questionable Use of ViewModel for Global Purposes

[This article](DjiSdkViewModelForGlobalPurposes.md) examines the misuse of `ViewModel` in the
DJI Android SDK sample project, focusing on its improper scoping to the `Application` class and
direct `Context` dependency. It highlights architectural concerns and provides refactoring
strategies to align with Android’s lifecycle-aware design principles.

# `android-sdk-v5-uxsdk` Project

## Design Analysis: How `WidgetModels` Update and Consume `UXKeys`

[This article](DjiUxSdkWidgetModelAndUXKey.md) explores how dual data sources (
`SharedPreferences` for persistence and `ObservableInMemoryKeyedStore` for real-time
synchronization) work together, and proposes a `UXDataSource` class to centralize data-source
handling and reduce boilerplate.