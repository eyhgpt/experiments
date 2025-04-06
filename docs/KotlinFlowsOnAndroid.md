https://developer.android.com/kotlin/flow

A flow is a type that can emit multiple values sequentially, as opposed to suspend functions
that return only a single value.

![Suspend Function vs Flow](../diagrams/kotlin/flow/suspend_fun_vs_reactive.png)

![Data Flowing in Multiple Directions](../diagrams/kotlin/flow/data_flowing_in_multiple_directions.png)

![Data Flowing in One Directions](../diagrams/kotlin/flow/data_flowing_in_one_direction.png)

![Flow Producer ](../diagrams/kotlin/flow/flow_producer_emits_consumer_collects.png)

### Popular Android Libraries Providing Data in Kotlin Flow

Kotlin Flow has emerged as a cornerstone of modern Android development, enabling reactive
programming through asynchronous data streams. Its integration with coroutines and seamless
interoperability with Jetpack libraries has made it a preferred choice for handling real-time
data updates, UI events, and network operations. Below is an exhaustive analysis of prominent
Android libraries that leverage Flow to deliver data efficiently, ensuring type safety,
backpressure handling, and lifecycle awareness.

#### 1. Room Database

Room, part of Android Jetpack, provides an abstraction layer over SQLite and natively supports
Kotlin Flow for reactive database queries. When a DAO (Data Access Object) returns a `Flow>`,
Room automatically emits updated results whenever the underlying data changes, eliminating
manual refresh logic.

```kotlin  
@Dao
interface UserDao {
 @Query("SELECT * FROM users")
 fun getUsers(): Flow<List<User>>
}
```

This emits a new list of `User` objects whenever the `users` table is modified, enabling
real-time UI updates in ViewModels. Room’s integration with Flow ensures transactional
consistency and efficient resource management, as it leverages coroutine scopes tied to
lifecycle-aware components.

#### 2. Jetpack DataStore

DataStore replaces SharedPreferences with a modern, Flow-based API for key-value pairs
(Preferences DataStore) or typed objects (Proto DataStore). It guarantees asynchronous,
transactional operations and automatic error handling.

##### Preferences DataStore:

```kotlin  
val Context.dataStore: DataStore by preferencesDataStore(name = "settings")
val theme_key = booleanPreferencesKey("dark_mode")

val themeFlow: Flow = dataStore.data
 .map { preferences -> preferences[theme_key] ?: false }  
```

DataStore emits the latest preference values on collection and caches them to minimize disk I/O.

##### Proto DataStore:

Proto DataStore uses Protocol Buffers to serialize custom objects, providing type safety:

```kotlin  
data class UserPreferences(val lastLogin: Long)

val userPreferencesFlow: Flow = dataStore.data
 .catch { exception ->
  if (exception is IOException) emit(UserPreferences(0))
  else throw exception
 }  
```

This is ideal for complex configurations requiring structured data.

#### 3. Retrofit with Coroutines

Retrofit, a type-safe HTTP client, integrates with coroutines to return `Flow` from suspend
functions. This enables reactive network request handling and seamless combination with database
operations.

```kotlin  
interface ApiService {
 @GET("users")
 suspend fun getUsers(): List<User>
}

fun fetchUsers(): Flow<List<User>> = flow {
 val users = apiService.getUsers()
 emit(users)
}.flowOn(Dispatchers.IO)
```

For pagination, Flow can be combined with the Paging Library to stream data incrementally.

#### 4. FlowBinding

FlowBinding converts Android UI events into Flow streams, replacing traditional listeners. It
supports widgets like `View`, `RecyclerView`, and `ViewPager2`, providing backpressure-aware
event streams.

```kotlin  
button.clicks()
 .onEach { /* Handle click */ }
 .launchIn(lifecycleScope)

viewPager2.pageSelections()
 .onEach { position -> updateTab(position) }
 .launchIn(viewLifecycleOwner.lifecycleScope)  
```

This library simplifies event-driven logic while ensuring automatic lifecycle management.

#### 5. WorkManager

WorkManager schedules background tasks and exposes work status via `Flow`. The `WorkInfo` stream
includes states like `ENQUEUED`, `RUNNING`, and `SUCCESS`, enabling reactive UI updates.

```kotlin  
val workInfoFlow: Flow =
 WorkManager.getInstance(context).getWorkInfoByIdFlow(workRequest.id)

workInfoFlow
 .filter { it.state == WorkInfo.State.SUCCEEDED }
 .onEach { /* Update UI */ }
 .launchIn(viewModelScope)  
```

#### 6. Jetpack ViewModel and StateFlow

ViewModels expose UI state using `StateFlow`, a state-holder that emits the latest value to
collectors. It ensures consistent state across configuration changes and integrates with Jetpack
Compose.

```kotlin  
class UserViewModel : ViewModel() {
 private val _userState = MutableStateFlow(UserState.Loading)
 val userState: StateFlow = _userState.asStateFlow()

 fun loadUser() {
  viewModelScope.launch {
   _userState.value = UserState.Success(repository.fetchUser())
  }
 }
}  
```

#### 7. Firebase Realtime Database and Firestore

Firebase’s Android SDKs offer Flow extensions for real-time data synchronization. The
`firebase-firestore-ktx` library provides `snapshotFlow()` to listen for document changes:

```kotlin  
firestore.collection("users")
 .document("123")
 .snapshotFlow()
 .map { it.toObject(User::class.java) }
 .onEach { user -> /* Update UI */ }
 .launchIn(lifecycleScope)  
```

#### 8. Paging Library 3

The Paging Library delivers paginated data as `Flow`, enabling efficient loading of large
datasets from network or database sources.

```kotlin  
@Dao
interface UserDao {
 @Query("SELECT * FROM users")
 fun pagingSource(): PagingSource<Int, User>
}

val users: Flow<PagingData<User>> = Pager(config) {
 userDao.pagingSource()
}.flow
```

#### Conclusion

Kotlin Flow has become integral to Android’s ecosystem, with libraries like Room, DataStore, and
Retrofit adopting it for real-time data handling. FlowBinding and StateFlow further bridge UI
events and state management, while WorkManager and Paging Library extend its utility to
background tasks and large datasets. By leveraging these libraries, developers can build
responsive, maintainable applications that adhere to modern architectural patterns like MVVM and
MVI. Future advancements may see deeper integration with Jetpack Compose and
multi-platform projects, solidifying Flow’s role in Android development.

---

### Creating a flow

The `flow` builder function in Kotlin is a fundamental way to create a cold flow, which means the
flow's code block executes each time a terminal operator is applied to it. Here are examples
demonstrating how to use this builder:

#### Basic Flow Creation

A simple flow that emits three sequential numbers:

```kotlin
//import kotlinx.coroutines.flow.Flow
//import kotlinx.coroutines.flow.flow

fun simpleCountFlow(): Flow = flow {
 for (i in 1..3) {
  emit(i)
 }
}

// Usage
suspend fun main() {
 simpleCountFlow().collect { value ->
  println(value) // Prints: 1, 2, 3
 }
}
```

In this example, the flow emits three integers sequentially. The `emit` function is used inside
the flow builder to send values downstream.

#### Practical Example: Fibonacci Sequence

Here's how to create a flow that generates Fibonacci numbers:

```kotlin
//import java.math.BigInteger
//import kotlinx.coroutines.flow.take

fun fibonacci(): Flow<BigInteger> = flow {
 println("LOG: Flow builder code started executing")
 var x = BigInteger.ZERO
 var y = BigInteger.ONE

 println("LOG: Variables initialized, starting loop")
 var count = 0
 while (true) {
  println("LOG: About to emit value #${++count}: $x")
  emit(x)
  println("LOG: After emission of $x")
  x = y.also {
   y += x
  }
 }
}

// Usage
suspend fun main() {
 println("LOG: Program started")
 val fibonacciFlow = fibonacci().take(10)
 println("LOG: Flow defined but not yet collected")

 fibonacciFlow.collect { number ->
  println("LOG: Collected value: $number")
 }

 println("LOG: First collection finished")

 // Collecting again to demonstrate cold flow behavior
 println("LOG: Starting second collection")
 fibonacciFlow.collect { number ->
  println("LOG: Second collection value: $number")
 }

 println("LOG: Program finished")
}
```

This flow generates an infinite sequence of Fibonacci numbers, but we limit collection to the first
10 using the `take` operator.

This example creates a cold flow using the flow builder function. A cold flow doesn't start emitting
values until it's collected, and the code inside the flow builder executes fresh each time it's
collected.

##### The log sequence should look like this:

```
LOG: Program started
LOG: Flow defined but not yet collected
LOG: Flow builder code started executing
LOG: Variables initialized, starting loop
LOG: About to emit value #1: 0
LOG: Collected value: 0
LOG: After emission of 0
LOG: About to emit value #2: 1
LOG: Collected value: 1
LOG: After emission of 1
LOG: About to emit value #3: 1
LOG: Collected value: 1
LOG: After emission of 1
LOG: About to emit value #4: 2
LOG: Collected value: 2
LOG: After emission of 2
LOG: About to emit value #5: 3
LOG: Collected value: 3
LOG: After emission of 3
LOG: About to emit value #6: 5
LOG: Collected value: 5
LOG: After emission of 5
LOG: About to emit value #7: 8
LOG: Collected value: 8
LOG: After emission of 8
LOG: About to emit value #8: 13
LOG: Collected value: 13
LOG: After emission of 13
LOG: About to emit value #9: 21
LOG: Collected value: 21
LOG: After emission of 21
LOG: About to emit value #10: 34
LOG: Collected value: 34
LOG: After emission of 34
LOG: First collection finished
LOG: Starting second collection
LOG: Flow builder code started executing
LOG: Variables initialized, starting loop
LOG: About to emit value #1: 0
LOG: Second collection value: 0
LOG: After emission of 0
...
```

##### Key Observations About Cold Flow Execution

- The code inside the flow builder doesn't execute until collect() is called.

- The flow building code executes sequentially, initialized variables first.

- Each emission is processed one at a time: First, the code reaches the emit() function. Then the
  collector receives and processes the value. Only after the collector finishes processing does
  execution continue to the next line after emit(). This back-and-forth pattern continues for
  each value.

- When a second collection happens, the flow builder code starts from scratch as if it were the
  first time, demonstrating the "cold" nature of the flow.

- The take(10) operator limits collection to the first 10 values, even though the flow is
  designed to produce an infinite sequence.

This interleaved execution between producer and consumer is a key characteristic of flows in Kotlin
coroutines, allowing for efficient processing of asynchronous data streams with backpressure
management built in.

#### Real-time Data Simulation Example

Creating a flow that simulates real-time sensor data:

```kotlin
//import kotlinx.coroutines.delay
//import kotlin.random.Random

data class SensorData(val timestamp: Long, val value: Double)

fun sensorDataFlow(intervalMillis: Long): Flow = flow {
 while (true) {
  val sensorData = SensorData(
   timestamp = System.currentTimeMillis(),
   value = Random.nextDouble(0.0, 100.0)
  )

  emit(sensorData)
  delay(intervalMillis)
 }
}

// Usage
suspend fun main() {
 val sensorFlow = sensorDataFlow(1000L) // Emit data every second
 sensorFlow.collect { data ->
  println("Received sensor data: $data")
 }
}
```

This example creates a flow that emits random sensor readings at regular intervals, simulating
real-time data collection.

#### File Operation Example

Using a flow to handle a background file operation:

```kotlin
//import kotlinx.coroutines.Dispatchers
//import kotlinx.coroutines.flow.flowOn

fun moveFile(source: String, destination: String): Flow = flow {
 // FileUtils is a hypothetical utility class
 // This operation runs in the context specified by flowOn
 FileUtils.move(source, destination)
 emit("Done")
}.flowOn(Dispatchers.IO)

// Usage
suspend fun main() {
 moveFile("path/to/source.txt", "path/to/destination.txt")
  .collect { status ->
   println("File move status: $status")
  }
}
```

This example demonstrates using the flow builder with `flowOn` to execute a file operation on a
background thread while emitting the result to the collector.

## Important Notes

1. Emissions from the flow builder are cancellable by default—each call to `emit` also calls
   `ensureActive`.

2. The `emit` function should happen strictly in the dispatcher of the flow block to preserve
   context. Using `emit` in a different context (like inside a `withContext` block) will cause an
   `IllegalStateException`.

3. If you need to switch the execution context of a flow, use the `flowOn` operator rather than
   changing contexts within the flow builder.

4. The flow builder creates a "cold" flow, meaning the code inside the builder executes only when
   the flow is collected.

These examples demonstrate the versatility of the flow builder for creating reactive data streams in
Kotlin coroutines.

