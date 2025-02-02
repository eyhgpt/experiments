## Introduction

This repository showcases my explorations and findings on various topics, including Android app
development using AI, featuring experiments, techniques, and insights that I find useful and
intriguing.

---

## Google's `Now in Android` Demo App Project

The `Now in Android` project is an open-source Android application designed to demonstrate best
practices in modern Android development. I have been following this demo project closely and have
made some small contributions to its development.

![screenshots](https://github.com/android/nowinandroid/blob/main/docs/images/screenshots.png)
*Image © Google*

[My Notes on Now in Android](docs/NowInAndroidApp.md)

---

## DJI's `Mobile-SDK-Android-V5` Demo App Project

DJI (the drone company)'s `Mobile-SDK-Android-V5` project serves as an excellent example of
utilizing the DJI Mobile SDK for advanced scientific and industrial applications, as well as for
building sophisticated user interfaces for Android apps.

![dji_fly_planning](/diagrams/dji_fly_video_recording.png)
*Image © DJI*

[My Notes on Mobile-SDK-Android-V5](docs/DjiDronesAndroidSdk.md)

### Questionable Use of ViewModel for Global Purposes

[This article](docs/DjiSdkViewModelForGlobalPurposes.md) examines the misuse of `ViewModel` in the
DJI Android SDK sample project, focusing on its improper scoping to the `Application` class and
direct `Context` dependency. It highlights architectural concerns and provides refactoring
strategies to align with Android’s lifecycle-aware design principles.

### Design Analysis: How `WidgetModels` Update and Consume `UXKeys`

```mermaid
classDiagram
    class UXKeys
    class CameraKeys
    class GlobalPreferenceKeys
    class MessagingKeys
    class UXKey
    class WidgetModel
    class TakeOffWidgetModel
    class UnitModeListItemWidgetModel
    class GlobalPreferencesInterface {
        <<interface>>
    }
    class DefaultGlobalPreferences
    class DataProcessor
    class ObservableKeyedStore {
        <<interface>>
    }
    class ObservableInMemoryKeyedStore
    class FlatStore

    ObservableKeyedStore <|-- ObservableInMemoryKeyedStore
    FlatStore <-- ObservableInMemoryKeyedStore: reads and writes
    UXKeys <|-- CameraKeys
    UXKeys <|-- GlobalPreferenceKeys
    UXKeys <|-- MessagingKeys
    UXKeys --> UXKey: creates
    UXKey <-- WidgetModel: reads
    DataProcessor <-- WidgetModel: reads and writes
    WidgetModel <|-- TakeOffWidgetModel
    WidgetModel <|-- UnitModeListItemWidgetModel
    ObservableInMemoryKeyedStore <-- TakeOffWidgetModel: reads
    ObservableInMemoryKeyedStore <-- UnitModeListItemWidgetModel: reads and writes
    TakeOffWidgetModel --> GlobalPreferencesInterface: reads
    UnitModeListItemWidgetModel --> GlobalPreferencesInterface: reads and writes
    ObservableInMemoryKeyedStore --> UXKeys: initializes
    GlobalPreferencesInterface <|-- DefaultGlobalPreferences
```

[This article](docs/DjiUxSdkWidgetModelAndUXKey.md) explores how dual data sources (
`SharedPreferences` for persistence and `ObservableInMemoryKeyedStore` for real-time
synchronization) work together, and proposes a `UXDataSource` class to centralize data-source
handling and reduce boilerplate.

---

## From Architectural Dependency Diagram to Code

![flowchart_dependencies](diagrams/flowchart_dependencies.png)

[This article](docs/ArchitecturalDependencyDiagramToCode.md) explores the use of `Mermaid 
flowcharts` and `LLM` to design and implement Android app architectures. The
`Architectural Dependency Diagrams` visually represent the dependencies between an Android app's
architectural classes, serving as a foundation for generating Kotlin source code with `LLMs` while
promoting modularity, separation of concerns, and scalability.

---

## From State Diagram to Code

![state_diagram](diagrams/state_diagram.png)

[This article](docs/StateDiagramToCode.md) demonstrates how to model complex logic with Mermaid
state diagrams and use LLMs to generate Kotlin code and unit tests based on the diagrams,
incorporating best practices such as coroutines, dependency injection, and error handling in Android
development.

---

## From Packet Diagram to Code

![packet_diagram](diagrams/packet_diagram.png)

[This article](docs/PacketDiagramToCode.md) demonstrates how to model network communication with
Mermaid packet diagrams and use LLMs to generate Kotlin code and JUnit tests for UDP packet
structures, including checksum calculation.

---

## From Code to Entity Relationship Diagram, From Entity Relationship Diagram to Code

![er_diagram_nia](diagrams/er_diagram_nia.png)

[This article ](docs/EntityRelationshipDiagramToCode.md) demonstrates how to model complex data
relationships by converting Kotlin code into Mermaid ER diagrams and generating Kotlin code from
Mermaid ER diagrams using LLMs, showcasing a bidirectional workflow between code and visual
documentation.

---

## From Code to Mindmap

![Generated Mindmap](diagrams/mindmap.png)

[This article](docs/CodeToMindmap.md) explores how to use LLMs to generate Mermaid mindmaps from
source code, such as the complex `SyncWorker` in the Now in Android app, to visualize relationships,
dependencies, and features for improved understanding and documentation.

---

## Use Unit Tests to Consolidate Knowledge

[This article](docs/UseUnitTestsToConsolidateKnowledge.md) explores how writing unit tests can
deepen understanding of Kotlin features like null safety and experimental contracts, providing
practical examples, insights into edge cases, and safeguards for evolving language behaviors.

---

## Generate Python Code and Video Using ChatGPT

[This article](docs/GeneratePythonCodeAndProduceVideo.md) shares my experience using ChatGPT to
generate a 120fps video for testing an Android device's playback performance.