# Navigating from Base App to Dynamic Feature Module

> [!NOTE]
> In my Android app, there are two modules:
>  - the base module `app`
>  - a dynamic feature module `df1`, which depends on the base module `app`
>
> Module `app` has the MainActivity, which is launched when the app launches. On it, there's an
> `Open DF1` button. On clicking this button, the Df1Activity defined in the `df1` module
> should launch.
>
> Given `df1` depends on `app`, how could `app` launch `df1`?

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