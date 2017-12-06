---
layout: page
title: Interop guide
site_nav_category: interop
is_site_nav_category: true
site_nav_category_order: 300
---

A set of rules for authoring public APIs in Java and Kotlin with the intent that the code will feel idiomatic when consumed from the other language.

_<a href="changelog.html">Last update: {{ site.changes.last.date | date: "%Y-%m-%d" }}</a>_


# Java (for Kotlin consumption)

## No hard keywords

Do not use Kotlin's [hard keywords](https://kotlinlang.org/docs/reference/keyword-reference.html#hard-keywords) as the name of methods or fields. These require the use of backticks to escape when calling from Kotlin. [Soft keywords](https://kotlinlang.org/docs/reference/keyword-reference.html#soft-keywords), [modifier keywords](https://kotlinlang.org/docs/reference/keyword-reference.html#modifier-keywords), and [special identifiers](https://kotlinlang.org/docs/reference/keyword-reference.html#special-identifiers) are allowed.

For example, Mockito's `when` function requires backticks when used from Kotlin:

```kotlin
val callable = Mockito.mock(Callable::class.java)
Mockito.`when`(callable.call()).thenReturn(/* … */)
```


## Lambda parameters last

Parameter types eligible for [SAM conversion](https://kotlinlang.org/docs/reference/java-interop.html#sam-conversions) should be last.

For example, RxJava 2's `Flowable.create()` method signature is defined as:

```java
public static <T> Flowable<T> create(
    FlowableOnSubscribe<T> source,
    BackpressureStrategy mode) { /* … */ }
```

Because `FlowableOnSubscribe` is eligible for SAM conversion, function calls of this method from Kotlin look like this:

```kotlin
Flowable.create({ /* … */ }, BackpressureStrategy.LATEST)
```

If the parameters were reversed in the method signature, though, function calls could use the trailing-lambda syntax:

```kotlin
Flowable.create(BackpressureStrategy.LATEST) { /* … */ }
```


## Property prefixes

For a method to be represented as a property in Kotlin, strict "bean"-style prefixing must be used.

Accessor methods require a 'get' prefix or for `boolean`-returning methods an 'is' prefix can be used.

```java
public final class User {
  public String getName() { /* … */ }
  public boolean isActive() { /* … */ }
}
```
```kotlin
val name = user.name // Invokes user.getName()
val active = user.active // Invokes user.isActive()
```

Associated mutator methods require a 'set' prefix.

```java
public final class User {
  public String getName() { /* … */ }
  public void setName(String name) { /* … */ }
}
```
```kotlin
user.name = "Bob" // Invokes user.setName(String)
```

If you want methods exposed as properties, do not use non-standard prefixes like 'has'/'set' or non-'get'-prefixed accessors. Methods with non-standard prefixes are still callable as functions which may be acceptable depending on the behavior of the method.


## Operator overloading

Be mindful of method names which allow special call-site syntax (i.e., [operator overloading](https://kotlinlang.org/docs/reference/operator-overloading.html)) in Kotlin. Ensure that methods names as such make sense to use with the shortened syntax.

```java
public final class IntBox {
  private final int value;
  public IntBox(int value) {
    this.value = value;
  }
  public IntBox plus(IntBox other) {
    return new IntBox(value + other.value);
  }
}
```
```kotlin
val one = IntBox(1)
val two = IntBox(2)
val three = one + two // Invokes one.plus(two)
```


## Nullability annotations

Every non-primitive parameter, return, and field type in a public API should have a nullability annotation. Non-annotated types are interpreted as ["platform" types](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types) which have ambiguous nullability.

JSR 305 package annotations could be used to set up a reasonable default but are currently discouraged. They require an opt-in flag to be honored by the compiler and conflict with Java 9's module system.


# Kotlin (for Java consumption)

## File name

When a file contains top-level functions or properties, *always* annotate it with `@file:JvmName("Foo")` to provide a nice name.

By default, top-level members in a file `Foo.kt` will end up in a class called `FooKt` which is unappealing and leaks the language as an implementation detail.

Consider adding `@file:JvmMultifileClass` to combine the top-level members from multiple files into a single class.


## Lambda arguments

[Function types](https://kotlinlang.org/docs/reference/lambdas.html#function-types) which are meant to be used from Java should avoid the return type `Unit`. Doing so requires specifying an explicit `return Unit.INSTANCE;` statement which is unidiomatic.

```kotlin
fun sayHi(callback: (String) -> Unit) = /* … */
```
```kotlin
// Kotlin caller:
greeter.sayHi { Log.d("Greeting", "Hello, $it!") }
```
```java
// Java caller:
greeter.sayHi(name -> {
    Log.d("Greeting", "Hello, " + name + "!");
    return Unit.INSTANCE;
});
```

This syntax also does not allow providing a semantically named type such that it can be implemented on other types.

Defining a named, single-abstract method (SAM) interface in Kotlin for the lambda type corrects the problem for Java, but prevents lambda syntax from being used in Kotlin.

```kotlin
interface GreeterCallback {
    fun greetName(name: String): Unit
}

fun sayHi(callback: GreeterCallback) = /* … */
```
```kotlin
// Kotlin caller:
greeter.sayHi(object : GreeterCallback {
    override fun greetName(name: String) {
        Log.d("Greeting", "Hello, $name!")
    }
})
```
```java
// Java caller:
greeter.sayHi(name -> Log.d("Greeting", "Hello, " + name + "!"))
```

Defining a named, SAM interface in Java allows the use of a slightly inferior version of the Kotlin lambda syntax where the interface type must be explicitly specified.

```java
// Defined in Java:
interface GreeterCallback {
    void greetName(String name);
}
```
```kotlin
fun sayHi(greeter: GreeterCallback) = /* … */
```
```kotlin
// Kotlin caller:
greeter.sayHi(GreeterCallback { Log.d("Greeting", "Hello, $it!") })
```
```java
// Java caller:
greeter.sayHi(name -> Log.d("Greeter", "Hello, " + name + "!"));
```

At present there is no way to define a parameter type for use as a lambda from both Java and Kotlin such that it feels idiomatic from both languages. The current recommendation is to prefer the function type despite the degraded experience from Java when the return type is `Unit`.

_Note: This recommendation might change in the future. See [KT-7770](https://youtrack.jetbrains.com/issue/KT-7770) and [KT-21018](https://youtrack.jetbrains.com/issue/KT-21018)._


## Avoid `Nothing` generics

A type whose generic parameter is `Nothing` is exposed as raw types to Java. Raw types are rarely used in Java and should be avoided.


## Document exceptions

Functions which can throw checked exceptions should document them with `@Throws`. Runtime exceptions should be documented in KDoc.

Be mindful of the APIs a function delegates to as they may throw checked exceptions which Kotlin otherwise silently allows to propagate.


## Defensive copies

When returning shared or unowned read-only collections from public APIs, wrap them in an unmodifiable container or perform a defensive copy. Despite Kotlin enforcing their read-only property, there is no such enforcement on the Java side. Without the wrapper or defensive copy, invariants can be violated by returning a long-lived collection reference.


## Companion functions

Public functions in a `companion object` must be annotated with `@JvmStatic` to be exposed as a static method.

Without the annotation, these functions are only available as instance methods on a static `Companion` field.

_Incorrect: no annotation_

```kotlin
class KotlinClass {
    companion object {
        fun doWork() {
            /* … */
        }
    }
}
```
```java
public final class JavaClass {
    public static void main(String... args) {
        KotlinClass.Companion.doWork();
    }
}
```

_Correct: `@JvmStatic` annotation_


```kotlin
class KotlinClass {
    companion object {
        @JvmStatic fun doWork() {
            /* … */
        }
    }
}
```
```java
public final class JavaClass {
    public static void main(String... args) {
        KotlinClass.doWork();
    }
}
```


## Companion constants

Public, non-`const` properties which are [_effective constants_](TODO) in a `companion object` must be annotated with `@JvmField` to be exposed as a static field.

Without the annotation, these properties are only available as oddly-named instance 'getters' on the static `Companion` field. Using `@JvmStatic` instead of `@JvmField` moves the oddly-named 'getters' to static methods on the class which is still incorrect.

_Incorrect: no annotation_

```kotlin
class KotlinClass {
    companion object {
        const val INTEGER_ONE = 1
        val BIG_INTEGER_ONE = BigInteger.ONE
    }
}
```
```java
public final class JavaClass {
    public static void main(String... args) {
        System.out.println(KotlinClass.INTEGER_ONE);
        System.out.println(KotlinClass.Companion.getBIG_INTEGER_ONE());
    }
}
```

_Incorrect: `@JvmStatic` annotation_

```kotlin
class KotlinClass {
    companion object {
        const val INTEGER_ONE = 1
        @JvmStatic val BIG_INTEGER_ONE = BigInteger.ONE
    }
}
```
```java
public final class JavaClass {
    public static void main(String... args) {
        System.out.println(KotlinClass.INTEGER_ONE);
        System.out.println(KotlinClass.getBIG_INTEGER_ONE());
    }
}
```

_Correct: `@JvmField` annotation_

```kotlin
class KotlinClass {
    companion object {
        const val INTEGER_ONE = 1
        @JvmField val BIG_INTEGER_ONE = BigInteger.ONE
    }
}
```
```java
public final class JavaClass {
    public static void main(String... args) {
        System.out.println(KotlinClass.INTEGER_ONE);
        System.out.println(KotlinClass.BIG_INTEGER_ONE);
    }
}
```

## Idiomatic naming

Kotlin has different calling conventions than Java which can change the way you name functions. Use `@JvmName` to design names such that they'll feel idiomatic for both language's conventions or to match their respective standard library naming.

This most frequently occurs for extension functions and extension properties because the location of the receiver type is different.

```kotlin
sealed class Optional<T : Any>
data class Some<T : Any>(val value: T): Optional<T>()
object None : Optional<Nothing>()

@JvmName("ofNullable")
fun <T> T?.asOptional() = if (this == null) None else Some(this)
```
```kotlin
// FROM KOTLIN:
fun main(vararg args: String) {
    val nullableString: String? = "foo"
    val optionalString = nullableString.asOptional()
}
```
```java
// FROM JAVA:
public static void main(String... args) {
    String nullableString = "Foo";
    Optional<String> optionalString =
          Optionals.ofNullable(nullableString);
}
```

## Function overloads for defaults

Functions with parameters having a default value must use `@JvmOverloads`. Without this annotation it is impossible to invoke the function using any default values.

When using `@JvmOverloads`, inspect the generated methods to ensure they each make sense. If they do not, perform one or both of the following refactorings until satisfied:

 1. Change the parameter order to prefer those with defaults being towards the end.
 2. Move the defaults into manual function overloads.

_Incorrect: No `@JvmOverloads`_

```kotlin
class Greeting {
    fun sayHello(prefix: String = "Mr.", name: String) {
        println("Hello, $prefix $name")
    }
}
```
```java
public class JavaClass {
    public static void main(String... args) {
        Greeting greeting = new Greeting();
        greeting.sayHello("Mr.", "Bob");
    }
}
```

_Correct: `@JvmOverloads` annotation._

```kotlin
class Greeting {
    @JvmOverloads
    fun sayHello(prefix: String = "Mr.", name: String) {
        println("Hello, $prefix $name")
    }
}
```
```java
public class JavaClass {
    public static void main(String... args) {
        Greeting greeting = new Greeting();
        greeting.sayHello("Bob");
    }
}
```
