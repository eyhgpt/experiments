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
Kotlin Flow for reactive database queries. When a DAO (Data Access Object) returns a
`Flow<List<T>>`, Room automatically emits updated results whenever the underlying data changes,
eliminating manual refresh logic.

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
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")
val THEME_KEY = booleanPreferencesKey("dark_mode")

val themeFlow: Flow<Boolean> = dataStore.data
 .map { preferences -> preferences[THEME_KEY] ?: false }
```

DataStore emits the latest preference values on collection and caches them to minimize disk I/O.

##### Proto DataStore:

Proto DataStore uses Protocol Buffers to serialize custom objects, providing type safety:

```kotlin  
data class UserPreferences(val lastLogin: Long)

val userPreferencesFlow: Flow<UserPreferences> = dataStore.data
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
val workInfoFlow: Flow<WorkInfo> =
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
 private val _userState = MutableStateFlow<UserState>(UserState.Loading)
 val userState: StateFlow<UserState> = _userState.asStateFlow()

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

The Paging Library delivers paginated data as `Flow<PagingData>`, enabling efficient loading of
large
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
class FlowUT {
 /*
 * https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html
 */
 private fun fibonacci(): Flow<BigInteger> = flow {
  // this: FlowCollector<BigInteger>

  println("3: Flow builder code started executing")
  var x = BigInteger.ZERO
  var y = BigInteger.ONE

  println("4: Variables initialized, starting loop")
  var count = 0

  while (true) {
   println("5: About to emit value #${++count}: $x")

   // FlowCollector<T>'s public suspend fun emit(value: T)
   emit(x)

   println("7: After emission of $x")

   x = y.also { y += x }
  }
 }

 @Test
 fun flow_01_flow_builder_to_emit_fibonacci() = runBlocking {
  println("1: Test started")

  // fun <T> Flow<T>.take(count: Int): Flow<T>(source)
  // Returns a flow that contains first count elements. When count elements are consumed, the
  // original flow is cancelled. Throws IllegalArgumentException if count is not positive.
  val fibonacciFlow: Flow<BigInteger> = fibonacci().take(10)

  println("2.1: Flow defined but not yet collected")

  // First collection
  val firstCollectionValues = mutableListOf<BigInteger>()

  fibonacciFlow.collectIndexed { index, number ->
   println("6.1: First collection value: $number; index: $index")

   firstCollectionValues.add(number)
  }

  println("8.1: First collection finished")

  // Second collection to demonstrate cold flow behavior
  println("2.2: Starting second collection")

  val secondCollectionValues = mutableListOf<BigInteger>()

  fibonacciFlow.collectIndexed { index, number ->
   println("6.2: Second collection value: $number; index: $index")

   secondCollectionValues.add(number)
  }

  println("8.2: Second collection finished")

  // Verify both collections produce the same sequence (cold flow behavior)
  assertEquals(firstCollectionValues, secondCollectionValues)

  // Verify the actual Fibonacci sequence values
  val expectedSequence = listOf(
   BigInteger.ZERO,
   BigInteger.ONE,
   BigInteger.ONE,
   BigInteger.valueOf(2),
   BigInteger.valueOf(3),
   BigInteger.valueOf(5),
   BigInteger.valueOf(8),
   BigInteger.valueOf(13),
   BigInteger.valueOf(21),
   BigInteger.valueOf(34)
  )

  assertEquals(expectedSequence, firstCollectionValues)
 }
}
```

This flow generates an infinite sequence of Fibonacci numbers, but we limit collection to the first
10 using the `take` operator.

This example creates a cold flow using the flow builder function. A cold flow doesn't start emitting
values until it's collected, and the code inside the flow builder executes fresh each time it's
collected.

##### The log sequence should look like this:

```
1: Test started
2.1: Flow defined but not yet collected
3: Flow builder code started executing
4: Variables initialized, starting loop
5: About to emit value #1: 0
6.1: First collection value: 0; index: 0
7: After emission of 0
5: About to emit value #2: 1
6.1: First collection value: 1; index: 1
7: After emission of 1
5: About to emit value #3: 1
6.1: First collection value: 1; index: 2
7: After emission of 1
5: About to emit value #4: 2
6.1: First collection value: 2; index: 3
7: After emission of 2
5: About to emit value #5: 3
6.1: First collection value: 3; index: 4
7: After emission of 3
5: About to emit value #6: 5
6.1: First collection value: 5; index: 5
7: After emission of 5
5: About to emit value #7: 8
6.1: First collection value: 8; index: 6
7: After emission of 8
5: About to emit value #8: 13
6.1: First collection value: 13; index: 7
7: After emission of 13
5: About to emit value #9: 21
6.1: First collection value: 21; index: 8
7: After emission of 21
5: About to emit value #10: 34
6.1: First collection value: 34; index: 9
8.1: First collection finished
2.2: Starting second collection
3: Flow builder code started executing
4: Variables initialized, starting loop
5: About to emit value #1: 0
6.2: Second collection value: 0; index: 0
7: After emission of 0
5: About to emit value #2: 1
6.2: Second collection value: 1; index: 1
7: After emission of 1
5: About to emit value #3: 1
6.2: Second collection value: 1; index: 2
7: After emission of 1
5: About to emit value #4: 2
6.2: Second collection value: 2; index: 3
7: After emission of 2
5: About to emit value #5: 3
6.2: Second collection value: 3; index: 4
7: After emission of 3
5: About to emit value #6: 5
6.2: Second collection value: 5; index: 5
7: After emission of 5
5: About to emit value #7: 8
6.2: Second collection value: 8; index: 6
7: After emission of 8
5: About to emit value #8: 13
6.2: Second collection value: 13; index: 7
7: After emission of 13
5: About to emit value #9: 21
6.2: Second collection value: 21; index: 8
7: After emission of 21
5: About to emit value #10: 34
6.2: Second collection value: 34; index: 9
8.2: Second collection finished
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

##### Cold Flow Characteristics

The fact that `fibonacciFlow` can be collected twice and produces the same values each time is a
clear indication that it is a "cold" flow in Kotlin coroutines.

In the code example, we can observe several key characteristics of cold flows:

1. Execution per collector: Notice how the console output shows the messages like "Flow builder code
   started executing" and "Variables initialized" appearing twice - once for each collection.
   This happens because the flow block is executed independently for each collector.

2. Same sequence from the beginning: Both collections receive the exact same Fibonacci sequence
   starting from zero. Each collector triggers a fresh execution of the flow producer code.

3. On-demand execution: The flow doesn't start producing values until `collect` is called. This
   is evident from the output showing that nothing happens between flow definition and collection.

4. Independent streams: Each collector gets its own completely independent stream of values. The
   first collection completes fully before the second one begins, with each getting the complete
   sequence.

##### Contrast with Hot Flows

If `fibonacciFlow` were a hot flow (like `SharedFlow` or `StateFlow`), its behavior would be
significantly different:

1. Single execution: A hot flow would execute its producer code only once, regardless of how many
   collectors are attached.

2. Shared emissions: Multiple collectors would share the same stream of values rather than each
   getting their own stream.

3. Missed values: If a collector starts late, it would miss values that were emitted before it
   started collecting (unless configured with replay). In your example, the second collector
   would likely miss the first few Fibonacci numbers.

4. Active without collectors: A hot flow could be actively emitting values even if there are no
   collectors present.

To make your Fibonacci example behave like a hot flow, you would need to change the implementation
to use something like `MutableSharedFlow` or `MutableStateFlow`, and you'd observe that the second
collection would not restart the sequence from zero but would continue from wherever the flow
currently was.

In summary, the ability to collect the same sequence multiple times confirms that `fibonacciFlow` is
indeed a cold flow, which is the default type created by the `flow {}` builder.

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

---

### Identifying Cold vs Hot Flows in Kotlin

When you encounter a function returning a `Flow<Something>` in Kotlin, distinguishing between cold and hot
flows is crucial for understanding its behavior and avoiding unexpected issues. This detailed guide
will help you identify flow types and understand their implications.

#### Checking the Return Type

The simplest way to determine if a flow is cold or hot is by checking its actual concrete return
type:

- Cold Flows: Functions returning a regular `Flow<T>` interface type (without specifying
  implementation) typically return cold flows.
- Hot Flows: Functions explicitly returning `StateFlow<T>`, `SharedFlow<T>`, 
  `MutableStateFlow<T>`, or `MutableSharedFlow<T>` are returning hot flows.

For example:

```kotlin
// Likely returns a cold flow
fun getData(): Flow<Data>

// Definitely returns a hot flow
fun getUpdates(): StateFlow<Updates>
```

#### Inspecting Implementation Details

If you have access to the function implementation, look for:

- Cold Flow Signs: Uses the `flow { ... }` builder, `flowOf()`, or `asFlow()` methods.
- Hot Flow Signs: Creates `MutableStateFlow`, `MutableSharedFlow`, or converts cold flows using
  `.stateIn()` or `.shareIn()`.

#### Function Naming Conventions

Function names often provide clues:

- Functions with names like `observeX`, `watchX`, or `getXUpdates` often return hot flows designed
  for ongoing observation.
- Functions with names like `getX`, `fetchX`, or `loadX` typically return cold flows that execute
  when collected.

#### Behavioral Characteristics to Check

If you're still unsure, you can verify through testing:

#### Multiple Collection Test

```kotlin
val myFlow = someFunction() // The Flow<T> you want to test

// Collect twice and observe behavior
myFlow.collect { /* first collector */ }
myFlow.collect { /* second collector */ }
```

- If the producer code runs twice (emitting the same sequence for each collector), it's a cold
  flow.
- If the second collector receives only new emissions (missing previous values), it's likely a hot
  flow.

#### Time Gap Test

```kotlin
val myFlow = someFunction()
delay(2000) // Wait before collecting
myFlow.collect { /* collector */ }
```

- If you receive all values regardless of the delay, it's likely a cold flow.
- If you miss values emitted during the delay, it's a hot flow.

#### Special Cases and Transformations

#### Library-Specific Flows

- Room Database: Although Room's Flow queries appear cold (they start when collected), they
  behave somewhat like hot flows once subscribed because they update with DB changes.
- Android ViewModels: Often expose hot flows (StateFlow) for UI state and events.

#### Cold-to-Hot Conversions

Be aware that cold flows can be converted to hot flows:

```kotlin
val coldFlow = flow { /* emissions */ }
val hotFlow = coldFlow.shareIn(scope, SharingStarted.Eagerly)
```

If a function uses such transformations internally but still returns a `Flow<T>` interface type, it
might actually be returning a hot flow despite the generic return type.

#### Practical Examples

#### Cold Flow Example

```kotlin
fun getNumbers(): Flow<Int> = flow {
 for (i in 1..5) {
  delay(1000)
  emit(i)
 }
}
```

This will only execute when collected, and each collector gets its own sequence starting from 1.

#### Hot Flow Example

```kotlin
private val _updates = MutableStateFlow<Data>(initialData)
fun getUpdates(): StateFlow<Data> = _updates.asStateFlow()
```

This is already active and emitting values regardless of collection, and all collectors share the
same emission stream.

#### Conclusion

When faced with a `Flow<Something>` return type, default to assuming it's cold unless proven 
otherwise by checking concrete types, naming patterns, or observable behavior. Understanding 
flow types is essential for proper flow management in Kotlin coroutines, helping you avoid 
missed emissions and unnecessary computation.     

---

