[//]: # (title: Interoperability with Swift/Objective-C)

This document covers some details of Kotlin/Native interoperability with
Swift/Objective-C.

## Usage

Kotlin/Native provides bidirectional interoperability with Objective-C.
Objective-C frameworks and libraries can be used in Kotlin code if
properly imported to the build (system frameworks are imported by default).
See [compilation configurations](multiplatform-configure-compilations.md#configure-interop-with-native-languages) for more details.
A Swift library can be used in Kotlin code if its API is exported to Objective-C
with `@objc`. Pure Swift modules are not yet supported.

Kotlin modules can be used in Swift/Objective-C code if compiled into a
framework ([see here for how to declare binaries](multiplatform-build-native-binaries.md#declare-binaries)).
See [calculator sample](https://github.com/JetBrains/kotlin/tree/master/kotlin-native/samples/calculator) for an example.

## Mappings

The table below shows how Kotlin concepts are mapped to Swift/Objective-C and vice versa.

"->" and "<-" indicate that mapping only goes one way.

| Kotlin                 | Swift                            | Objective-C                      | Notes                                                                              |
|------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------|
| `class`                | `class`                          | `@interface`                     | [note](#name-translation)                                                          |
| `interface`            | `protocol`                       | `@protocol`                      |                                                                                    |
| `constructor`/`create` | Initializer                      | Initializer                      | [note](#initializers)                                                              |
| Property               | Property                         | Property                         | [note 1](#top-level-functions-and-properties), [note 2](#setters)                  |
| Method                 | Method                           | Method                           | [note 1](#top-level-functions-and-properties), [note 2](#method-names-translation) |
| `suspend` ->           | `completionHandler:`/ `async`    | `completionHandler:`             | [note 1](#errors-and-exceptions), [note 2](#suspending-functions)                  |
| `@Throws fun`          | `throws`                         | `error:(NSError**)error`         | [note](#errors-and-exceptions)                                                     |
| Extension              | Extension                        | Category member                  | [note](#extensions-and-category-members)                                           |
| `companion` member <-  | Class method or property         | Class method or property         |                                                                                    |
| `null`                 | `nil`                            | `nil`                            |                                                                                    |
| `Singleton`            | `shared` or `companion` property | `shared` or `companion` property | [note](#kotlin-singletons)                                                         |
| Primitive type         | Primitive type / `NSNumber`      |                                  | [note](#nsnumber)                                                                  |
| `Unit` return type     | `Void`                           | `void`                           |                                                                                    |
| `String`               | `String`                         | `NSString`                       |                                                                                    |
| `String`               | `NSMutableString`                | `NSMutableString`                | [note](#nsmutablestring)                                                           |
| `List`                 | `Array`                          | `NSArray`                        |                                                                                    |
| `MutableList`          | `NSMutableArray`                 | `NSMutableArray`                 |                                                                                    |
| `Set`                  | `Set`                            | `NSSet`                          |                                                                                    |
| `MutableSet`           | `NSMutableSet`                   | `NSMutableSet`                   | [note](#collections)                                                               |
| `Map`                  | `Dictionary`                     | `NSDictionary`                   |                                                                                    |
| `MutableMap`           | `NSMutableDictionary`            | `NSMutableDictionary`            | [note](#collections)                                                               |
| Function type          | Function type                    | Block pointer type               | [note](#function-types)                                                            |
| Inline classes         | Unsupported                      | Unsupported                      | [note](#unsupported)                                                               |

### Name translation

Objective-C classes are imported into Kotlin with their original names.
Protocols are imported as interfaces with `Protocol` name suffix,
i.e. `@protocol Foo` -> `interface FooProtocol`.
These classes and interfaces are placed into a package [specified in build configuration](#usage)
(`platform.*` packages for preconfigured system frameworks).

The names of Kotlin classes and interfaces are prefixed when imported to Objective-C.
The prefix is derived from the framework name.

Objective-C does not support packages in a framework. Thus, the Kotlin compiler renames Kotlin classes which have the
same name but different package in the same framework. This algorithm is not stable yet and can change between Kotlin
releases.
As a workaround, you can rename the conflicting Kotlin classes in the framework.

### Initializers

Swift/Objective-C initializers are imported to Kotlin as constructors and factory methods
named `create`. The latter happens with initializers declared in the Objective-C category or
as a Swift extension, because Kotlin has no concept of extension constructors.

Kotlin constructors are imported as initializers to Swift/Objective-C. 

### Setters

Writeable Objective-C properties overriding read-only properties of the superclass are represented as `setFoo()` method for the property `foo`. Same goes for a protocol's read-only properties that are implemented as mutable.

### Top-level functions and properties

Top-level Kotlin functions and properties are accessible as members of special classes.
Each Kotlin file is translated into such a class. E.g.

```kotlin
// MyLibraryUtils.kt
package my.library

fun foo() {}
```

can be called from Swift like

```swift
MyLibraryUtilsKt.foo()
```

### Method names translation

Generally Swift argument labels and Objective-C selector pieces are mapped to Kotlin
parameter names. Anyway these two concepts have different semantics, so sometimes
Swift/Objective-C methods can be imported with a clashing Kotlin signature. In this case
the clashing methods can be called from Kotlin using named arguments, e.g.:

```swift
[player moveTo:LEFT byMeters:17]
[player moveTo:UP byInches:42]
```

in Kotlin it would be:

```kotlin
player.moveTo(LEFT, byMeters = 17)
player.moveTo(UP, byInches = 42)
```

### Errors and exceptions

Kotlin has no concept of checked exceptions, all Kotlin exceptions are unchecked.
Swift has only checked errors. So if Swift or Objective-C code calls a Kotlin method
which throws an exception to be handled, then the Kotlin method should be marked
with a `@Throws` annotation specifying a list of "expected" exception classes.

When compiling to Objective-C/Swift framework, non-`suspend` functions having or inheriting
`@Throws` annotation are represented as `NSError*`-producing methods in Objective-C
and as `throws` methods in Swift.  Representations for `suspend` functions always have
`NSError*`/`Error` parameter in completion handler.

When Kotlin function called from Swift/Objective-C code throws an exception
which is an instance of one of the `@Throws`-specified classes or their subclasses,
it is propagated as `NSError`. Other Kotlin exceptions reaching Swift/Objective-C
are considered unhandled and cause program termination.

`suspend` functions without `@Throws` propagate only
`CancellationException` as `NSError`. Non-`suspend` functions without `@Throws`
don't propagate Kotlin exceptions at all.

Note that the opposite reversed translation is not implemented yet:
Swift/Objective-C error-throwing methods aren't imported to Kotlin as
exception-throwing.

### Suspending functions

> Support for calling `suspend` functions from Swift code as `async` is [Experimental](components-stability.md).
> It may be dropped or changed at any time.
> Use it only for evaluation purposes. We would appreciate your feedback on it in [YouTrack](https://youtrack.jetbrains.com/issue/KT-47610).
>
{type="warning"}

Kotlin's [suspending functions](coroutines-basics.md) (`suspend`) are presented in the generated Objective-C headers as
functions with callbacks, or [completion handlers](https://developer.apple.com/documentation/swift/calling_objective-c_apis_asynchronously)
in Swift/Objective-C terminology.

Starting from Swift 5.5, Kotlin's `suspend` functions are also available for calling from Swift as
`async` functions without using the completion handlers. Currently, this functionality is highly experimental and has certain limitations. See [this YouTrack issue](https://youtrack.jetbrains.com/issue/KT-47610)
for details.

Learn more about the [`async`/`await` mechanism in Swift](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html).

### Extensions and category members

Members of Objective-C categories and Swift extensions are imported to Kotlin
as extensions. That's why these declarations can't be overridden in Kotlin.
And the extension initializers aren't available as Kotlin constructors.

Kotlin extensions to "regular" Kotlin classes are imported to Swift and Objective-C as extensions and category members respectively.
Kotlin extensions to other types are treated as [top-level declarations](#top-level-functions-and-properties)
with an additional receiver parameter. These types include:

* Kotlin `String` type
* Kotlin collection types and subtypes
* Kotlin `interface` types
* Kotlin primitive types
* Kotlin `inline` classes
* Kotlin `Any` type
* Kotlin function types and subtypes
* Objective-C classes and protocols

### Kotlin singletons

Kotlin singleton (made with an `object` declaration, including `companion object`)
is imported to Swift/Objective-C as a class with a single instance.

The instance is available through the `shared` and `companion` properties.

For the following Kotlin code:

```kotlin
object MyObject {
    val x = "Some value"
}

class MyClass {
    companion object {
        val x = "Some value"
    }
}
```

Access these objects as follows: 

```swift
MyObject.shared
MyObject.shared.x
MyClass.companion
MyClass.Companion.shared
```

> Access objects through `[MySingleton mySingleton]` in Objective-C and `MySingleton()` in Swift has been deprecated.
> 
{type="note"}

### NSNumber

Kotlin primitive type boxes are mapped to special Swift/Objective-C classes.
For example, `kotlin.Int` box is represented as `KotlinInt` class instance in Swift
(or `${prefix}Int` instance in Objective-C, where `prefix` is the framework names prefix).
These classes are derived from `NSNumber`, so the instances are proper `NSNumber`s
supporting all corresponding operations.

`NSNumber` type is not automatically translated to Kotlin primitive types
when used as a Swift/Objective-C parameter type or return value.
The reason is that `NSNumber` type doesn't provide enough information
about a wrapped primitive value type, i.e. `NSNumber` is statically not known
to be a e.g. `Byte`, `Boolean`, or `Double`. So Kotlin primitive values
should be cast to/from `NSNumber` manually (see [below](#casting-between-mapped-types)).

### NSMutableString

`NSMutableString` Objective-C class is not available from Kotlin.
All instances of `NSMutableString` are copied when passed to Kotlin.

### Collections

Kotlin collections are converted to Swift/Objective-C collections as described
in the table above. Swift/Objective-C collections are mapped to Kotlin in the same way,
except for `NSMutableSet` and `NSMutableDictionary`. `NSMutableSet` isn't converted to
a Kotlin `MutableSet`. To pass an object for Kotlin `MutableSet`,
you can create this kind of Kotlin collection explicitly by either creating it 
in Kotlin with e.g. `mutableSetOf()`, or using the `KotlinMutableSet` class in Swift 
(or `${prefix}MutableSet` in Objective-C, where `prefix` is the framework names prefix).
The same holds for `MutableMap`.

### Function types

Kotlin function-typed objects (e.g. lambdas) are converted to 
Swift functions / Objective-C blocks. However there is a difference in how
types of parameters and return values are mapped when translating a function
and a function type. In the latter case primitive types are mapped to their
boxed representation. Kotlin `Unit` return value is represented
as a corresponding `Unit` singleton in Swift/Objective-C. The value of this singleton
can be retrieved in the same way as it is for any other Kotlin `object`
(see singletons in the table above).
To sum the things up:

```kotlin
fun foo(block: (Int) -> Unit) { ... }
```

would be represented in Swift as

```swift
func foo(block: (KotlinInt) -> KotlinUnit)
```

and can be called like

```kotlin
foo {
    bar($0 as! Int32)
    return KotlinUnit()
}
```

### Generics

Objective-C supports "lightweight generics" defined on classes, with a relatively limited feature set. Swift can import 
generics defined on classes to help provide additional type information to the compiler.

Generic feature support for Objective-C and Swift differ from Kotlin, so the translation will inevitably lose some information,
but the features supported retain meaningful information.

#### Limitations

Objective-C generics do not support all features of either Kotlin or Swift, so there will be some information lost
in the translation.

Generics can only be defined on classes, not on interfaces (protocols in Objective-C and Swift) or functions.

#### Nullability

Kotlin and Swift both define nullability as part of the type specification, while Objective-C defines nullability on methods
and properties of a type. As such, the following:

```kotlin
class Sample<T>() {
  fun myVal(): T
}
```

will (logically) look like this:

```swift
class Sample<T>() {
  fun myVal(): T?
}
```

In order to support a potentially nullable type, the Objective-C header needs to define `myVal` with a nullable return value.

To mitigate this, when defining your generic classes, if the generic type should *never* be null, provide a non-null 
type constraint:

```kotlin
class Sample<T : Any>() {
  fun myVal(): T
}
```

That will force the Objective-C header to mark `myVal` as non-null.

#### Variance

Objective-C allows generics to be declared covariant or contravariant. Swift has no support for variance. Generic classes coming
from Objective-C can be force-cast as needed.

```kotlin
data class SomeData(val num: Int = 42) : BaseData()
class GenVarOut<out T : Any>(val arg: T)
```

```swift
let variOut = GenVarOut<SomeData>(arg: sd)
let variOutAny : GenVarOut<BaseData> = variOut as! GenVarOut<BaseData>
```

#### Constraints

In Kotlin you can provide upper bounds for a generic type. Objective-C also supports this, but that support is unavailable 
in more complex cases, and is currently not supported in the Kotlin - Objective-C interop. The exception here being a non-null
upper bound will make Objective-C methods/properties non-null.

#### To disable

To have the framework header written without generics, add the flag to the compiler config:

```kotlin
binaries.framework {
     freeCompilerArgs += "-Xno-objc-generics"
}
```

## Casting between mapped types

When writing Kotlin code, an object may need to be converted from a Kotlin type
to the equivalent Swift/Objective-C type (or vice versa). In this case a plain old
Kotlin cast can be used, e.g.

```kotlin
val nsArray = listOf(1, 2, 3) as NSArray
val string = nsString as String
val nsNumber = 42 as NSNumber
```

## Subclassing

### Subclassing Kotlin classes and interfaces from Swift/Objective-C

Kotlin classes and interfaces can be subclassed by Swift/Objective-C classes
and protocols.

### Subclassing Swift/Objective-C classes and protocols from Kotlin

Swift/Objective-C classes and protocols can be subclassed with a Kotlin `final` class.
Non-`final` Kotlin classes inheriting Swift/Objective-C types aren't supported yet, so it is
not possible to declare a complex class hierarchy inheriting Swift/Objective-C types.

Normal methods can be overridden using the `override` Kotlin keyword. In this case
the overriding method must have the same parameter names as the overridden one.

Sometimes it is required to override initializers, e.g. when subclassing `UIViewController`. 
Initializers imported as Kotlin constructors can be overridden by Kotlin constructors
marked with the `@OverrideInit` annotation:

```swift
class ViewController : UIViewController {
    @OverrideInit constructor(coder: NSCoder) : super(coder)

    ...
}
```

The overriding constructor must have the same parameter names and types as the overridden one.

To override different methods with clashing Kotlin signatures, you can add a
`@Suppress("CONFLICTING_OVERLOADS")` annotation to the class.

By default the Kotlin/Native compiler doesn't allow calling a non-designated
Objective-C initializer as a `super(...)` constructor. This behaviour can be
inconvenient if the designated initializers aren't marked properly in the Objective-C
library. Adding a `disableDesignatedInitializerChecks = true` to the `.def` file for
this library would disable these compiler checks.

## C features

See [Interoperability with C](native-c-interop.md) for an example case where the library uses some plain C features,
such as unsafe pointers, structs, and so on.

## Unsupported

Some features of Kotlin programming language are not yet mapped into respective features of Objective-C or Swift.
Currently, following features are not properly exposed in generated framework headers:
   * inline classes (arguments are mapped as either underlying primitive type or `id`)
   * custom classes implementing standard Kotlin collection interfaces (`List`, `Map`, `Set`) and other special classes
   * Kotlin subclasses of Objective-C classes
