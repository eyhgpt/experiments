> [!NOTE]
> This document addresses questions related to Dynamic Feature Modules in Android app development.

### Table of Contents

- [Navigating from Base App to Dynamic Feature Module](#navigating-from-base-app-to-dynamic-feature-module)
- [FeatureComponentHolder for Dynamic Feature Modules' Dependency Injection](#featurecomponentholder-for-dynamic-feature-modules-dependency-injection)

# Navigating from Base App to Dynamic Feature Module

```
Question: In my Android app, there are two modules:
 - the base module `app`
 - a dynamic feature module `df1`, which depends on the base module `app`

Module `app` has the MainActivity, which is launched when the app launches. On it, there's an
`Open DF1` button. On clicking this button, the Df1Activity defined in the `df1` module
should launch.

Given `df1` depends on `app`, how could `app` launch `df1`?
```

To launch an activity from your dynamic feature module (`df1`) from the base module (`app`), you
need to work with the inversed dependency relationship. Since the base module cannot have a
compile-time dependency on the dynamic feature module, there are several effective approaches to
handle this navigation:

## 1. Using Explicit Intents with Class Names

The most straightforward approach is using explicit intents with the fully qualified class name:

```kotlin
// In MainActivity (app module)
binding.openDf1Button.setOnClickListener {
 val intent = Intent()
 intent.setClassName(packageName, "com.example.df1.Df1Activity")
 startActivity(intent)
}
```

This method works without compile-time dependencies but lacks type safety.

## 2. Using Intent Actions and Filters

Define a custom intent action in the dynamic feature module's manifest:

```xml
<!-- In df1 module's AndroidManifest.xml -->
<activity android:name=".Df1Activity">
    <intent-filter>
        <action android:name="com.example.app.OPEN_Df1" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

Then launch it from the base module:

```kotlin
// In MainActivity
binding.openDf1Button.setOnClickListener {
 val intent = Intent("com.example.app.OPEN_Df1")
 startActivity(intent)
}
```

## 3. Using SplitInstallManager for On-Demand Features

If your `df1` module is configured for on-demand delivery (not install-time), you'll need to ensure
it's downloaded first:

```kotlin
// In MainActivity
private val splitInstallManager = SplitInstallManagerFactory.create(this)

private fun openDf1Bot() {
 // Check if module is already installed
 if (splitInstallManager.installedModules.contains("df1")) {
  launchDf1Activity()
 } else {
  // Request to install the module
  val request = SplitInstallRequest.newBuilder()
   .addModule("df1")
   .build()

  splitInstallManager.startInstall(request)
   .addOnSuccessListener {
    launchDf1Activity()
   }
   .addOnFailureListener { exception ->
    // Handle installation failure
   }
 }
}

private fun launchDf1Activity() {
 val intent = Intent()
 intent.setClassName(packageName, "com.example.df1.Df1Activity")
 startActivity(intent)
}
```

## 4. Using Interface-Based Communication

For a more robust, type-safe approach:

### 1. Define an interface in the base module:

```kotlin
// In base module
interface Df1FeatureApi {
 fun launchDf1Activity(context: Context)
}
```

### 2. Create a service locator in the base module:

```kotlin
// In base module
object FeatureRegistry {
 private val features = mutableMapOf<Class<*>, Any>()

 fun <T> register(clazz: Class<T>, implementation: T) {
  features[clazz] = implementation
 }

 @Suppress("UNCHECKED_CAST")
 fun <T> get(clazz: Class<T>): T? = features[clazz] as? T
}
```

### 3. Implement the interface in the dynamic feature module:

```kotlin
// In df1 module
class Df1FeatureApiImpl : Df1FeatureApi {
 override fun launchDf1Activity(context: Context) {
  val intent = Intent(context, Df1Activity::class.java)
  context.startActivity(intent)
 }
}
```

### 4. Register the implementation using a ContentProvider in the dynamic feature:

```kotlin
// In df1 module
class Df1FeatureInitializer : ContentProvider() {
 override fun onCreate(): Boolean {
  FeatureRegistry.register(Df1FeatureApi::class.java, Df1FeatureApiImpl())
  return true
 }
 // Implement other required methods
}
```

### 5. Access from the base module:

```kotlin
binding.openDf1Button.setOnClickListener {
 FeatureRegistry.get(Df1FeatureApi::class.java)?.launchDf1Activity(this)
}
```

## 5. Using Navigation Component with Deep Links

If using Jetpack Navigation:

### 1. Define navigation graph with deep links in the dynamic feature:

```xml
<!-- In df1/res/navigation/Df1_navigation.xml -->
<navigation>
    <activity android:id="@+id/Df1Activity" android:name=".Df1Activity">
        <deepLink app:uri="app://example.com/Df1" />
    </activity>
</navigation>
```

### 2. Use the deep link from the base module:

```kotlin
binding.openDf1Button.setOnClickListener {
 val uri = Uri.parse("app://example.com/Df1")
 startActivity(Intent(Intent.ACTION_VIEW, uri))
}
```

## Recommendation

For an install-time dynamic feature, the intent-based approaches (options 1 or 2) are simplest. If
you need more type safety or maintainability, the interface-based approach (option 4) is worth the
additional setup. If your feature is on-demand, you must use SplitInstallManager (option 3) to
ensure the module is downloaded before attempting to launch it.

For all approaches, remember to configure SplitCompat in your base module's Application class:

```kotlin
override fun attachBaseContext(base: Context) {
 super.attachBaseContext(base)
 SplitCompat.install(this)
}
```

# FeatureComponentHolder for Dynamic Feature Modules' Dependency Injection

```
/**
 * Base interface that all feature component holders must implement.
 * This interface enables the app module to manage feature module components
 * without creating direct dependencies on feature module classes.
 *
 * The abstract interface pattern is used to decouple the app module from
 * specific implementations in feature modules, preventing circular dependencies.
 */
interface FeatureComponentHolder {
 /**
  * Called when resources need to be released.
  * Feature modules should implement this to clean up any resources
  * when the component is being disposed or the feature is no longer needed.
  */
 fun releaseResources()
}

 /**
  * Registry of active feature components.
  * This map stores references to all active feature components by feature ID,
  * allowing for efficient component reuse and lifecycle management.
  *
  * Using a map with string keys decouples the app module from specific feature module types,
  * avoiding circular dependencies between modules.
  */
 private val featureComponents = mutableMapOf<String, FeatureComponentHolder>()

 /**
  * Gets an existing feature component or creates a new one if it doesn't exist.
  * This method implements the Service Locator pattern for feature modules.
  *
  * @param featureId A unique identifier for the feature (typically the feature module name)
  * @param factory A lambda that creates a new component instance if needed
  * @return The existing or newly created component instance
  *
  * The method uses type parameters to ensure type safety while still allowing
  * generic handling of different component types.
  */
 fun <T : FeatureComponentHolder> getOrCreateFeatureComponent(
  featureId: String,
  factory: () -> T
 ): T {
  @Suppress("UNCHECKED_CAST")
  return featureComponents.getOrPut(featureId) { factory() } as T
 }
```

The approach demonstrated in the code is a well-known and recommended pattern in Android
development for managing dynamic feature modules. This pattern combines several established software
design principles:

### 1. Service Locator Pattern

- The `featureComponents` map acts as a service registry
- Components are accessed via unique keys (`featureId`) rather than concrete types
- Decouples service consumers from service implementations
- Particularly useful for dynamic features that may not always be installed

### 2. Dependency Inversion Principle

- `FeatureComponentHolder` interface serves as an abstraction layer
- App module depends on abstractions (interface) rather than concrete implementations
- Feature modules provide concrete implementations of the interface

### 3. Component Holder Pattern

- Generic type parameter `<T : FeatureComponentHolder>` ensures type safety
- `getOrCreateFeatureComponent` manages component lifecycle
- Centralized component management in the base `app` module

### 4. Modularization Best Practices

- Strict visibility control between modules
- Clear component ownership boundaries
- Support for dynamic delivery (Play Feature Delivery)
- Build-time optimization through proper scoping

### Key Benefits:

1. Circular Dependency Prevention
    - Base app module contains only interfaces
    - Feature modules implement these interfaces
    - No compile-time dependencies between app ↔ feature modules

2. Lifecycle Management
    - `releaseResources()` enables clean resource disposal
    - Prevents memory leaks when features are uninstalled

3. Testability
    - Easy mocking through interface implementations
    - Independent testing of feature modules

4. Dynamic Feature Support
    - Components are loaded only when needed
    - Compatible with on-demand module installation

This pattern is officially recommended by Android documentation and aligns with Google's
modularization guidance for large apps.

- https://developer.android.com/topic/modularization/patterns
- https://developer.android.com/guide/navigation/integrations/feature-modules

Major apps like Google Play Store and YouTube use
[similar approaches](https://www.infoq.com/news/2022/11/android-modularization-guide/)
to manage their dynamic features.