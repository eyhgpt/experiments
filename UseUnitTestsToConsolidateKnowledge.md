# Introduction

In my experience, one of the best ways to deepen my understanding of a programming language is by
writing unit tests to explore its features and behaviors. For example, while learning Kotlin, I
wrote a series of unit tests to investigate its null safety features, including scenarios like
explicit null pointer exceptions, the not-null assertion operator, initialization quirks, and Java
interoperability.

By doing this, I not only clarified how Kotlin handles various null-related situations but also
created a personal reference that I can revisit whenever I’m uncertain. This approach has been
incredibly useful when I’ve faced confusing situations, as I can rely on these tests to confirm how
the language behaves in edge cases.

Additionally, I’ve found that as the language evolves, these tests act as a safeguard—if something
breaks, it’s a clear indicator that Kotlin’s behavior has changed. This hands-on practice has been
instrumental in solidifying my understanding and keeping my knowledge current.

# Unit Tests on Kotlin's Null Safety Related Features and Behaviors

```kotlin
import org.junit.jupiter.api.Assertions.assertDoesNotThrow
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.fail
import org.junit.jupiter.api.Test

// Docs: https://kotlinlang.org/docs/null-safety.html

class NullSafetyUT {

 /**
  * Demonstrates how to explicitly throw a NullPointerException in Kotlin.
  * This test verifies that the thrown exception is of type NullPointerException
  * and contains the expected error message.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * An explicit call to throw NullPointerException().
  * (other causes omitted)
  * '''
  */
 @Test
 fun null_01_explicit_throw_null_pointer_exception() {
  assertThrows(NullPointerException::class.java) {
   throw NullPointerException("Explicitly thrown NullPointerException")
  }
 }

 /**
  * Demonstrates how the not-null assertion operator (!!) causes a NullPointerException
  * when used on a null value. The !! operator converts any nullable value to a
  * non-null type, throwing NPE if the value is actually null.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * Usage of the not-null assertion operator !!
  * (other causes omitted)
  * '''
  */
 @Test
 fun null_02_not_null_assertion_operator() {
  assertThrows(NullPointerException::class.java) {
   val nullableString: String? = null
   @Suppress("KotlinConstantConditions")
   nullableString!!.length
  }
 }

 /**
  * Demonstrates how a NullPointerException occurs due to data inconsistency during initialization
  * when accessing an uninitialized property through a "leaking this" reference in the constructor.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * Data inconsistency during initialization, such as when:
  *  An uninitialized this available in a constructor is used somewhere else (a "leaking this").
  * (other causes omitted)
  * '''
  */
 @Test
 fun null_03_leaking_this_in_constructor() {
  assertThrows(NullPointerException::class.java) {
   // Triggers the initialization sequence:
   LeakingThis()
  }
 }

 class LeakingThis {
  // 1. Property is declared but not yet initialized
  val myProperty: String

  // 2. During initialization block execution:
  init {
   // 3. Creates Helper instance and passes 'this' (current LeakingThis instance)
   //    before myProperty is initialized - this is the "leak"
   Helper(this).doSomething()

   // 5. This line never executes because NPE is thrown before reaching here
   myProperty = "initialized"
  }
 }

 class Helper(private val instance: LeakingThis) {
  fun doSomething() {
   // 4. Attempts to access myProperty, but it's still uninitialized
   //    Since myProperty is a non-null String type but hasn't been assigned a value yet,
   //    accessing it throws NullPointerException
   instance.myProperty.length
  }
 }

 /**
  * Demonstrates a case where NullPointerException occurs in Kotlin during initialization
  * when a superclass constructor calls an open member whose implementation in the derived class
  * uses an uninitialized state. This is one of the few legitimate cases where NPE can occur
  * in Kotlin's null-safe type system.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * Data inconsistency during initialization, such as when:
  *  A superclass constructor calling an open member whose implementation in the derived class uses
  *  an uninitialized state.
  * (other causes omitted)
  * '''
  */
 @Test
 fun null_04_initialization_npe_with_open_member() {
  assertThrows(NullPointerException::class.java) {
   // Triggers the initialization sequence:
   // 1. Create Child instance, which first calls Parent constructor
   Child()
  }
 }

 open class Parent {
  init {
   // 2. Parent's init block runs, calling the overridden printMessage()
   printMessage()

   // Note:  IDE shows warning "Calling non-final function printMessage in constructor"
   // This is because calling non-final methods in constructor is dangerous as they can be
   // overridden by subclasses and accessed uninitialized state, exactly what happens in this case
  }

  open fun printMessage() {}  // 6. This implementation is not used since Child overrides it
 }

 class Child : Parent() {
  // 4. At this point message is NOT initialized yet because:
  // - Parent constructor must complete first (which includes init block)
  // - Child's properties are initialized only after Parent constructor completes
  private val message: String = "Hello"

  // 3. Child's printMessage is called from Parent's init, but message isn't initialized yet
  override fun printMessage(): Unit = println(message.length)
 }

 /**
  * This test demonstrates how Kotlin's null safety can be bypassed when interoperating with Java,
  * specifically when accessing members of a null platform type reference. Platform types come from
  * Java code where nullability is not explicitly declared, leading to potential NPEs at runtime.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * Java interoperation:
  *   Attempts to access a member of a null reference of a platform type.
  * (other causes omitted)
  * '''
  *
  * As quoted from https://kotlinlang.org/docs/java-interop.html#null-safety-and-platform-types
  * '''
  * Any reference in Java may be null, which makes Kotlin's requirements of strict null-safety
  * impractical for objects coming from Java. Types of Java declarations are treated in Kotlin in a
  * specific manner and called platform types. Null-checks are relaxed for such types, so that
  * safety guarantees for them are the same as in Java.
  * '''
  */
 @Test
 fun null_05_platform_type_java_interop() {
  assertThrows(NullPointerException::class.java) {
   // Platform types come from Java code where nullability is unknown to Kotlin Since Java's type
   // system doesn't encode null-safety information, the return type becomes a platform type
   // (String!) in Kotlin By explicitly declaring javaString as String (non-nullable), we're forcing
   // an unsafe cast from platform type to non-null.
   //
   // This will throw NPE because:
   // 1. getNullString() returns null from Java
   // 2. Forcing this null into a non-null String type causes NPE
   @Suppress("UNUSED_VARIABLE")
   val javaString: String = JavaDemoClass.getNullString()
  }

  // This doesn't throw NPE because:
  // 1. We explicitly declare the variable as nullable (String?)
  // 2. Kotlin's smart cast handles the platform type safely
  // 3. The safe call operator (?.) is used to safely access length
  // 4. When the value is null, safe call returns null instead of throwing NPE
  assertDoesNotThrow {
   val javaStringNullable: String? = JavaDemoClass.getNullString()
   assertNull(javaStringNullable)
  }
 }

 /**
  * Demonstrates how Java-Kotlin interop can cause NPE with generic collections
  * when Java code adds null to a Kotlin MutableList<String> that expects non-null elements
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * The only possible causes of an NPE in Kotlin are:
  * Data inconsistency during initialization, such as when:
  *  Nullability issues with generic types. For example, a piece of Java code adding null into a Kotlin MutableList<String>, which would require MutableList<String?> to handle it properly.
  * (other causes omitted)
  * '''
  */
 @Test
 fun null_06_safety_generic_java_interop_npe() {

  // MutableList<String> in Kotlin is compiled to java.util.List<String> in Java bytecode.
  // ArrayList() implements MutableList, making it compatible with Java's List interface.
  // This allows seamless interoperability between Kotlin and Java collections.
  val list1: MutableList<String> = ArrayList()

  // Invoke Java code to add a null to the list
  JavaDemoClass.addNullTo(list1)

  assertThrows(NullPointerException::class.java) {
   // This will throw NPE when accessing null element because Kotlin's type system assumes
   // all elements in MutableList<String> are non-null. However, Java code bypassed this
   // assumption by adding a null element, leading to an unexpected null value at runtime.
   list1[0].length
  }

  val list2 = mutableListOf<String>()

  // This compiles because casting to java.util.List<String> removes Kotlin's null safety checks
  // at compile time. The cast tells the compiler to treat the list as a Java List, which allows
  // null elements by default.
  @Suppress("UNCHECKED_CAST", "PLATFORM_CLASS_MAPPED_TO_KOTLIN")
  (list2 as java.util.List<String>).add(null)

  assertThrows(NullPointerException::class.java) {
   // This will throw NPE when accessing null element because Kotlin's type system assumes
   // all elements in MutableList<String> are non-null. However, the explicit cast to Java's List
   // type circumvented Kotlin's type system checks, allowing a null to be inserted directly into
   // the underlying collection, leading to an unexpected null value at runtime.
   list2[0].length
  }
 }


 /**
  * This test demonstrates Kotlin's null safety by showcasing how accessing an uninitialized
  * `lateinit` variable throws an `UninitializedPropertyAccessException`.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html
  * '''
  * Besides NPE, another exception related to null safety is UninitializedPropertyAccessException.
  * Kotlin throws this exception when you try to access a property that has not been initialized,
  * ensuring that non-nullable properties are not used until they are ready. This typically happens
  * with lateinit properties.
  * '''
  */
 @Test
 fun null_07_uninitialized_property_access() {
  assertThrows(UninitializedPropertyAccessException::class.java) {
   val example = ExampleClass()
   // Attempting to access the uninitialized property will throw an
   // UninitializedPropertyAccessException
   @Suppress("UNUSED_VARIABLE")
   val length = example.uninitializedProperty.length
  }
 }

 /**
  * This class serves as an example to demonstrate the behavior of `lateinit` variables.
  */
 class ExampleClass {
  // The `lateinit` keyword allows the property to be initialized later. However, accessing it
  // before initialization will result in an `UninitializedPropertyAccessException`. To initialize
  // it properly, you must assign a value to this property before accessing it in any way.
  lateinit var uninitializedProperty: String
 }

 /**
  * This test function demonstrates the use of the safe call operator on the left side of an assignment in Kotlin.
  * It showcases that the assignment does not throw an exception even if one of the properties in the chain is null.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html#safe-call-operator
  * '''
  * You can also place a safe call on the left side of an assignment:
  *    person?.department?.head = managersPool.getManager()
  * In the example above, if one of the receivers in the safe call chain is null, the assignment is
  * skipped, and the expression on the right is not evaluated at all. For example, if either person
  * or person.department is null, the function is not called.
  * '''
  */
 @Test
 fun null_08_safe_call_assignment() {
  val person1: Employee? = null
  val person2: Employee? = Employee("John Doe", null)
  val person3: Employee? = Employee("Jane Doe", Department(null))

  // These assertions check the initial state of the objects:
  // - `person1` is null, so everything related to it should be null.
  // - `person2` exists but has no department, hence `department` is null.
  // - `person3` has a department, but the department's `head` is null.
  assertNull(person1)
  assertNotNull(person2)
  assertNull(person2?.department)
  assertNotNull(person3)
  assertNotNull(person3?.department)
  assertNull(person3?.department?.head)

  assertDoesNotThrow {
   // Test case where both person and department are null
   // This assignment is skipped since `person1` is null, no exception is thrown
   @Suppress("KotlinConstantConditions")
   person1?.department?.head = "Manager 1"

   // Test case where person is not null, but department is null
   // This assignment is also skipped because `person2?.department` is null, no exception is thrown
   person2?.department?.head = "Manager 1"

   // Test case where both person and department are not null
   // Here, the assignment is executed because all properties exist
   person3?.department?.head = "Manager 1"
  }

  // These assertions verify the state after the safe call assignments:
  // - `person1` remains null
  // - `person2` still has no department, so `department` remains null
  // - `person3`'s department now has a head, set to "Manager 1"
  assertNull(person1)
  assertNotNull(person2)
  assertNull(person2?.department)
  assertNotNull(person3)
  assertNotNull(person3?.department)
  assertEquals(person3?.department?.head, "Manager 1")
 }

 class Employee(val name: String, var department: Department?)
 class Department(var head: String?)

 /**
  * This test function demonstrates the usage of Elvis operator (?:) with throw and return
  * expressions in Kotlin. It shows that since throw and return are expressions, they can be used
  * on the right side of the Elvis operator to handle null cases by either throwing an exception or
  * returning from the function.
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html#elvis-operator
  * '''
  * Since throw and return are expressions in Kotlin, you can also use them on the right-hand side
  * of the Elvis operator.
  * '''
  */
 @Test
 fun null_09_elvis_operator_with_throw_and_return() {
  val num: Int? = null

  assertThrows(IllegalArgumentException::class.java) {
   @Suppress("KotlinConstantConditions")
   num?.let { println("Correct: $it") } ?: throw IllegalArgumentException("Wrong number: $num")
  }

  @Suppress("KotlinConstantConditions")
  num?.let { println("Correct: $it") } ?: return

  // If the previous `?: return` executes, this assertion will not run, and the unit test will pass.
  // Otherwise, the unit test will fail.
  fail<Nothing>("This line should not be reached")
 }

 /**
  * This test demonstrates how Kotlin handles toString() calls on nullable receivers in two ways:
  * 1. Direct toString() call which safely returns "null" string for null objects
  * 2. Safe-call operator (?.) with toString() which returns null for null objects
  *
  * As quoted from https://kotlinlang.org/docs/null-safety.html#nullable-receiver
  * '''
  * You can use extension functions with a nullable receiver type, allowing these functions to be
  * called on variables that might be null.
  *
  * For example, the .toString() extension function can be called on a nullable receiver. When
  * invoked on a null value, it safely returns the string "null" without throwing an exception.
  *
  * If you expect the .toString() function to return a nullable string (either a string
  * representation or null), use the safe-call operator ?.. The ?. operator calls .toString() only
  * if the object is not null, otherwise it returns null.
  * '''
  */
 @Test
 fun null_10_safe_toString_behavior() {
  // Initialize test objects: one null Person and one non-null Person
  val person1: Person? = null
  val person2: Person? = Person("Peter")

  // Test toString() behavior with null object 'person1':

  // A call to the Any?.toString() extension function safely returns the "null" string for null
  // objects
  assertEquals("null", person1.toString())

  // The safe-call operator (?.) returns null (not the "null" string) as the Any::toString()
  // member function is never invoked on a null reference
  @Suppress("KotlinConstantConditions")
  assertNull(person1?.toString())

  // Note: 'person1.toString()' and 'person1?.toString()' invoke different toString()
  // implementations. 'person1.toString()' calls the extension function defined as Any?.toString(),
  // while 'person1?.toString()' attempts to call the member function defined in Any (overridden by
  // data class Person). When 'person1' is null, the extension function Any?.toString() is
  // invoked and returns "null". Conversely, with 'person1?.toString()', the member
  // function Any::toString() is never called when 'person1' is null due to the safe-call operator

  // Test toString() behavior with non-null object:
  assertEquals("Person(name=Peter)", person2.toString())
  assertEquals("Person(name=Peter)", person2?.toString())
 }

 data class Person(val name: String)

}
```

# Unit Tests on Kotlin's Experimental Contracts

> [!NOTE]
> This feature is `Experimental`, but it has been available since 2018 and has been used in Google's
> example Android app, `Now in Android`.

```kotlin
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import kotlin.contracts.ExperimentalContracts
import kotlin.contracts.*

// https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/contracts/ContractBuilder.kt
// https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/samples/test/samples/contracts/contracts.kt
// https://www.nutrient.io/blog/kotlin-contracts/

@OptIn(ExperimentalContracts::class)
class ContractsUT {

 /**
  * The contract says: if this function returns normally, then the condition
  * (maybeString != null) must be true.
  */
 fun isNotNull(condition: Boolean) {
  contract {
   returns() implies (condition)
  }

  if (!condition) throw IllegalArgumentException()
 }

 @Test
 fun contracts_01_condition_check_with_contract() {
  var maybeString: String? = "Hello"

  fun doTheTest() {
   isNotNull(maybeString != null)

// Without the contract above, 'maybeString' wouldn't be smart-cast to non-null:
   val length = maybeString.length
   assertEquals(5, length)
  }

  doTheTest()
 }

 fun performTask(block: () -> Unit) {
  contract {
   callsInPlace(block, InvocationKind.EXACTLY_ONCE)

  }

  block()

// `block` can not be called multiple times with the `InvocationKind.EXACTLY_ONC` contract.
// Otherwise, it would show error:
// [WRONG_INVOCATION_KIND] Wrong invocation kind 'EXACTLY_ONCE' for 'block: () -> Unit'
// specified, the actual invocation kind is 'AT_LEAST_ONCE'.

// `block` can not be not called with the `InvocationKind.EXACTLY_ONC` contract.
// Otherwise, it would show error:
// [WRONG_INVOCATION_KIND] Wrong invocation kind 'EXACTLY_ONCE' for 'block: () -> Unit'
// specified, the actual invocation kind is 'AT_MOST_ONCE'.
 }

 @Test
 fun contracts_06_compiles_only_with_call_in_place_exactly_once() {
  val intValue: Int

  performTask {
// Does not compile without the contract of `InvocationKind.EXACTLY_ONCE`, or
// `InvocationKind.AT_MOST_ONCE`
// Error would be:
// [CAPTURED_VAL_INITIALIZATION] Captured values cannot be initialized because of possible reassignments
   intValue = 1
  }

// Does not compile with the contract of `InvocationKind.AT_MOST_ONCE`. Error would be:
// [UNINITIALIZED_VARIABLE] Variable 'intValue' must be initialized.
  assertEquals(1, intValue)
 }
}
```