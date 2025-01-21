# Introduction

DJI (the drone company)'s `Mobile-SDK-Android-V5` project serves as an excellent example of
utilizing the DJI Mobile SDK for advanced scientific and industrial applications, as well as for
building sophisticated user interfaces for Android apps. It demonstrates sophisticated UI components
tailored for aviation control, mapping, navigation, and error messaging. In terms of Android
architectural and library choices, the project utilizes `ViewModel`, `rxJava`, `LiveData`, and
`Views`, while not incorporating dependency injection (DI), coroutines, or `Jetpack Compose`.

This article aims to highlight the strengths of this sample application and suggest areas for
improvement, based on my personal perspective. However, I acknowledge that the rapid evolution of
Android app architecture can make it challenging to refactor existing codebases to align with the
latest best practices.

## Questionable Use of ViewModel for Global Purposes

> [!NOTE]
> I didn't intend to begin this notes with a critique. This project demonstrates impressively
> sophisticated UI architecture, which I plan to discuss in detail later.

In this project, the use of `ViewModel` raises some questions regarding its adherence to Android
architectural best practices. Specifically, `ViewModel` appears to be used for global purposes
rather than being scoped to `Activity` or `Fragment` lifecycles, which is its intended design. For
example, `MSDKManagerVM` is injected into `DJIApplication`, a design choice that seems to misalign
with the principle that `ViewModels` should be tied to UI components like `Activity` or `Fragment`.
This approach deviates from the recommended Android architecture, which aims to ensure clear
separation of concerns and proper lifecycle management. Below, I discuss how this implementation
could be refined for better alignment with these principles.

### 1) ViewModel Held by Application

One of the notable design decisions is the injection of the `MSDKManagerVM` into `DJIApplication`:

```kotlin
open class DJIApplication : Application() {

 private val msdkManagerVM: MSDKManagerVM by globalViewModels()

 override fun onCreate() {
  super.onCreate()

  // Ensure initialization is called first
  msdkManagerVM.initMobileSDK(this)
 }
}
```

Code copied from
[DJIApplication.kt](https://github.com/dji-sdk/Mobile-SDK-Android-V5/blob/dev-sdk-main/SampleCode-V5/android-sdk-v5-sample/src/main/java/dji/sampleV5/aircraft/DJIApplication.kt)

This is surprising because:

1. Typical Role of a ViewModel  
   In Android, a `ViewModel` is conceptually tied to an `Activity` or `Fragment` lifecycle. The
   primary goal is to manage and retain UI-related data in a lifecycle-conscious way. By design,
   `ViewModel` does not outlive the lifecycle of its owner (the UI controller). Injecting a
   `ViewModel` into an `Application` is discouraged because the `Application` object has a
   different (much longer) lifecycle than an `Activity`/`Fragment`.

2. Purpose of Holding the ViewModel  
   The `DJIApplication` seems to hold `msdkManagerVM` but isn’t really using it. Instead,
   `MSDKManagerVM` is used in `DJIMainActivity`. Hence, it would be
   more intuitive to instantiate `MSDKManagerVM` directly where it is actually used—namely in
   `DJIMainActivity`—instead of in `DJIApplication`.

3. Application Lifecycle  
   The `Application` instance is created before any `Activity` and remains alive as long as the
   process is alive. A `ViewModel` injected in `Application` can easily cause resource leaks or hold
   onto references longer than necessary if not carefully managed.

If the goal is to keep track of registration or SDK connection state at a global level (beyond a
single Activity), a more conventional approach would be to use other patterns such as using a
singleton manager within `Application` that handles the SDK lifecycle (not in a `ViewModel`).

### 2) Application Context injected into ViewModel

The second design observation is that `MSDKManagerVM` receives a `Context` (in the form of
`DJIApplication`) directly in its `initMobileSDK` function:

```kotlin
class MSDKManagerVM : ViewModel() {
 // The data is held in livedata mode, but you can also save the results of the sdk callbacks any way you like.
 val lvRegisterState = MutableLiveData<Pair<Boolean, IDJIError?>>()
 // ...

 fun initMobileSDK(appContext: Context) {
  // Initialize and set the sdk callback, which is held internally by the sdk until destroy() is called
  SDKManager.getInstance().init(appContext, object : SDKManagerCallback {
   override fun onRegisterSuccess() {
    lvRegisterState.postValue(Pair(true, null))
   }

   override fun onRegisterFailure(error: IDJIError) {
    lvRegisterState.postValue(Pair(false, error))
   }

   // ...
  })

  // ...
 }
}
```

Code copied from
[DJIApplication.kt](https://github.com/dji-sdk/Mobile-SDK-Android-V5/blob/dev-sdk-main/SampleCode-V5/android-sdk-v5-sample/src/main/java/dji/sampleV5/aircraft/models/MSDKManagerVM.kt)

This is strongly discouraged by the Android architecture guidelines because:

- ViewModel Independence  
  [The official architecture recommendations](https://developer.android.com/topic/architecture/recommendations)
  emphasize that a `ViewModel` should be lifecycle-agnostic and should not hold references to
  `Context` or other lifecycle-bound objects (`Activity`, `Fragment`, etc.). The reason is to avoid
  memory leaks and to allow the `ViewModel` to survive configuration changes or other lifecycle
  events in a safe manner.

- Direct Context Usage  
  If a `ViewModel` truly needs a `Context` to perform certain operations (e.g., reading from shared
  preferences or accessing Android system services), we should evaluate whether such operations
  belong in that layer. Often, these operations can be moved to a separate abstraction ( e.g., a
  repository, a data source class, or some other manager) that can be injected into the `ViewModel`
  without passing a `Context`.

- SDK Initialization  
  In this case, `SDKManager.getInstance().init(appContext, ...)` sets up the DJI SDK. Such an
  initialization is typically an application-level concern, because it must be done before many
  other components can use the SDK. We can separate the actual SDK initialization—which requires
  a `Context`—from the `ViewModel` logic that handles the callback events. A possible approach:

    - Perform the SDK initialization (which needs `Application` context) in `DJIApplication`.

    - Handle the SDK’s callback events in `ViewModels`, which are later injected into Activities
      or Fragments.

By splitting up these responsibilities:

- You avoid the `ViewModel` having to deal with a `Context`.
- You keep a clear separation of concerns: the application-level initialization is done in the
  `Application` class or a dedicated manager, and the UI or data-holding logic remains in the
  `ViewModel`.

### Potential Refactoring

A structure more aligned with Android’s guidelines could look like this:

1. `DJIApplication`
    - Responsible only for initializing the `SDKManager` by calling
      `SDKManager.getInstance().init(applicationContext)`.

2. `MSDKManagerVM`
    - Wires up callbacks to LiveData for UI updates by invoking
      `SDKManager.getInstance().onCallbacks(object : SDKManagerCallback { ... })`.

3. `DJIMainActivity`
    - Injects `MSDKManagerVM` into `DJIMainActivity` (using Hilt, optionally).
    - Observes LiveData from `MSDKManagerVM` for UI updates related to registration, connectivity,
      and other SDK states.

This approach separates the concerns of ensuring the SDK is initialized (an application-level
responsibility) from managing UI or user-facing states around the SDK (a ViewModel
responsibility).

