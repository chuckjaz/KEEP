# Compound Extensions

* Type: Design proposal
* Author: Chuck Jazdzewski, Alexander Nozik
* Status: New
* Prototype: none


## Summary

The proposal describes a language syntax to declare methods and functional types with multiple receiver types and resolution rules to bind those functions and methods to specific call place.

## Introduction

### Description

Kotlin currently supports two types of extensions:
* top level extensions, that could have several arguments, but only one receiver type (and is called on one receiver object),
* member extensions that exist inside the class (and could be overridden) and so in fact has two receiver objects - the member of the class which they are called in and actual receiver object.

The dispatch in the first case is done statically, using only compile-time information on the receiver type. In second case, dispatch is mixed since actual method is dispatched dynamically based on the actual type of calling object.

There are two major uses for receivers: simplify the inner logics of the method and to define the lexical scope (context) for some function application. It is currently obvious that in some cases, it is really useful to have intersection of scope behavior and/or have one type to gain some additional behavior when called in scope of some object of another type. It could be done via member extensions, but the problem is that member extensions could not be declared on existing class and could not have full static dispatch.

We propose to both unify and extend the way extensions work by adding possibility to declare not one, but several receiver on a method or functional type. There are several proposals on grammar:

```kotlin
operator fun [Body, String].unaryPlus(s: String) = escape(s)
```

```kotlin
operator fun (Body, String).unaryPlus(s: String) = escape(s)
```

```kotlin
operator fun Body.String.unaryPlus(s: String) = escape(s)
```

The proposal with the round brackets is a neutral one, so we will use it in the proposal samples.

is interpreted as extending `Body` with the extension `String.unaryPlus()` which further
extends `String` with a unary `+` operator. In other words, in the scope of a `Body`
receiver, expressions of type `String` have a unary `+` operator.

### Motivation / use cases

* Context-oriented programming as discussed [here](https://proandroiddev.com/an-introduction-context-oriented-programming-in-kotlin-2e79d316b0a2).

* Contextual extensions ([from](https://youtrack.jetbrains.com/issue/KT-10468))

  In Android it is common to refer to density independent pixels. In Kotlin this can be
  improved by the use of an extension `val` but the calculation requires the current
  `Context`.

  Given this proposal, `View` can be extended with a  `Float` and `Int` extensions to simplify
  usage in a `View`.

  ```kotlin
  val (View, Float).dp get() = context.displayMetrics.density * this
  val (View, Int).dp get() = this.toFloat().dp
  ```

  which would then be used like,

  ```kotlin
  class SomeView : View {
    val someDimension = 4.dp
  }
  ```

* DSL by extension ([from](https://discuss.kotlinlang.org/t/add-infix-fun-to-a-class-without-extend-the-class/10772))

  Consider the Android type `JSONObject`, the following extensions can be written:

  ```kotlin
  inline operator fun (JSONObject, String).invoke(build: JSONObject.() -> Unit) = put(this, JSONObject().build())
  inline infix operator fun (JSONObject, String).to(value: Any?) = put(this, value)
  fun json(build: JSONObject.() -> Unit) = JSONObject().build()
  ```

  which could be used like,

  ```kotlin
  val obj = json {
      "key" {
          "key1" to "value"
          "key2" to 3
      }
  }

* Allows a fluent programming style in a receiver scope

  Consider wishing to use `produce` from `CoroutineScope` to implement a filter
  of a `ReceiverChannel`, the developer is faced with three choices,

  1. Create a `filter` that takes `CoroutineScope` and the `ReceiverChannel` as
       such as,

       ```kotlin
       fun <T> filter(
           scope: CoroutineScope,
           channel: ReceiveChannel<T>,
           predicate: (value: T) -> Boolean) = scope.produce<T> {
         for (value in channel) if (predicate) send(value)
       }
       ```

       ```kotlin
       fun main() = runBlocking {
         filter(this, numbersFrom(2)) { it % 2 }.forEach { println(it) }
       }
       ```

  2. Create a `filter` that extends `CoroutineScope` and takes `ReceiverChannel` as
       a parameter such as,

       ```kotlin
       fun <T> CoroutineScope.filter(
          channel: ReceiveChannel<T>,
          predicate: (value: T) -> Boolean) = produce<T> {
          for (value in channel) if (predicate) send(value)
       }
       ```

       ```kotlin
       fun main() = runBlocking {
         filter(numbersFrom(2)) { it % 2 }.forEach { println(it) }
       }
       ```

  3. Create a `filter` that `ReceiverChannel` and takes `CoroutineScope` as a
       parameter such as,

       ```kotlin
       fun <T> RecieverChannel<T>.filter(
           scope: CoroutineScope,
           predicate: (value: T) -> Boolean) = scope.produce<T> {
         for (value in this) if (predicate) send(value)
       }
       ```

       ```kotlin
       fun main() = runBlocking {
         numbersFrom(2).filter(this) { it % 2 }.forEach { println(it) }
       }
       ```

    With this proposal the developer could introduce use a more fluent `filter`,

    ```kotlin
    fun <T> (CoroutineScope, ReceiverChannel<T>).filter(
        predicate: (value: T) -> Boolean) = produce<T> {
      for (value in this) if (predicate) send(value)
    }
    ```

    ```kotlin
    fun main() = runBlocking {
        numbersFrom(2).filter { it % 2 }.forEach { println(it) }
    }
    ```

* Generalizing local extensions

  Extensions in local scope are difficult to extract. Consider a simplified HTML DSL
  such as:

    ```kotlin
    open class HtmlDsl() { fun write(s: String) {  } fun escape(s: String) { } }
    class Html(): HtmlDsl() {}
    class Body(): HtmlDsl() { }

    fun html(block: Html.() -> Unit) {
        with (Html()) {
            write("<html>")
            block()
            write("</html>")
        }
    }

    fun Html.body(block: Body.() -> Unit) {
        write("<body>")
        Body().block()
        write("<body>")
    }

    fun Body.div(block: Body.() -> Unit) {
        write("<div>")
        block()
        write("</div>")
    }

    fun Body.span(block: Body.() -> Unit) {
        write("<span>")
        block()
        write("</span>")
    }
    ```

   To emit string content it would be convenient to overload the unary `+` operator to
   mean emit text with html escapes. The developer cannot use extension methods for this
   to be generally available and must add the operator to the correct classes directly
   such as,

   ```kotlin
   class Body(): HtmlDsl() {
       operator fun String.unaryPlus(s: String) = escape(s)
   }
   ```

   In a local scope the developer can declare extension operators for special purposes
   such as introducing a unary `-` operator to underline text,

   ```kotlin
   html {
        body {
            operator fun String.unaryMinus() {
                underline {
                    +this@unaryMinus
                }
            }
            div {
                -"Customer"
                customer(customer)

                -"Orders"
                orders(orders)
            }
        }
    }
    ```

    However, the developer is required to modify `Body` class to allow the unary `-` to be
    used outside its current local scope. This is problematic if the HTML DSL is a third-party
    library.

    Compound extensions allows the original unary `+` operator to be introduced as an extension
    function such as,

    ```kotlin
    operator fun (Body, String).unaryPlus() = escape(this)
    ```

    therefore allowing unary `-` to be provided generally by extension of `Body`,

    ```kotlin
    operator fun (Body, String).unaryMinus() = underline { +this@unaryMinus }
    ```

* Better comperhansion of libraries like Jetpack Compose. 

    Righ now, Jetpack Compose usese @Composable annotation for a method and then calls methods starting with capital letters:

    ```kotlin
    @Composable
    fun NewsStory() {
        Column {
            Text("A day in Shark Fin Cove")
            Text("Davenport, California")
            Text("December 2018")
        }
    }
    ```

    Here `Column` is in fact not a constructor, but a function that changes some kind of implicit state of GUI builder. Sadly, it looks like a constructor with a side-effect and does not look good. It would make sense to make a signature of this function like `ComposeScope.column(block: ColumnScope.() -> Unit)`, and change the whole composing function to `ComposeScope.newStory(){}`. In this case the compiler could infere composable functions not by annotations, but by receiver type and it would work without breakin language ideology and tooling. Sadly, it would mean, that other receivers would not be available to the functions inside the builder since there could be only one. For example it is not possible to add `CoroutineScope` receiver that is needed for coroutines.

    Multiple receiver could help solve this problem by introducing composable objects with `CoroutineScope` like this: `(ComposeScope, CoroutineScope).doSomething()`.

* Type intersections.

    Currently Kotlin suports type intersections with `where` keyword. But application of such intersections is limited and the syntax is cumbersome. Multiple receivers could solve this problem as well and add some functionality to it. By declaring a function that requires reciever of type `A` and reveiver with type `B`, we can write  `(A, B).doSomething()` and either provide both receivers as `with(a,b){doSomething()}` or provide single object of type that inherits both `A` and `B`.


## Proposal

### Syntax

No new syntax is required. Extension declarations that are currently invalid are givena valid interpretation. There are several posibilities with their own pros and cons:

#### Using dot

```kotlin
operator fun Body.String.unaryPlus() = escape(this)
```

is syntactically valid but reports that `String` is not valid as there is no nested
`String` class of `Body`.

The type expression syntax is similarly interpreted. For example the type of,

```kotlin
fun A.B.C.someMethod(v: Int): Int = ...
```

would be written, `A.B.C.(Int) -> Int`.

Obviously this notation could be confused with nested classes. Also it implyies ordered resolution strategy (see Matching rules).


#### Using parenthesis
```kotlin
fun (A, B, C).someMethod(v: Int): Int = ...

val a: (A,B,C).(v: Int)->Int = ...
```

The types are unambiguous such that `fun (A.B, C).someMethod(v: Int)`, the `A.B` is
unambiguously a reference to the nested type `B` in `A`.

The type expression syntax would be similarly extended to allow `(A.B, C).() -> Unit`.

However, the type expression syntax is ambiguous at the `(` and it is not until the `.` is seen after the initial closing `)`.

#### Using brackets
```kotlin
fun [A, B, C].someMethod(v: Int): Int = ...

val a: [A, B, C].(Int)->Int = ...
```

The types are similarly unambiguous without introducing the same syntactic ambiguity. However,
this require introducing a use for `[` in a type expression that might be better reserved by
a feature more widely leveraged such as tuples.

#### Using a pseudo-keyword

```kotlin
extension fun (A, B, C).someMethod(v: Int): Int = ...
val a: extension (A, B, C).(Int)->Int = ...
```

Using a pseudo keyword avoids the ambiguity of the `(` in the type expression. A pseudo-
keyword could also be used with the `[`...`]` syntax to leave a unadorned `[`...`]` to
mean a tuple in the future.

### Matching rules

Matching rules is the most complicated point of the proposal. In order to give more freedom to discussion we will provide two slightly different descriptions:

#### First variant: ordered resolution

1. **Create an ordered list of implicit receivers** in scope with types
<code>I<sup>1</sup></code>...<code>I<sup>m</sup></code> where <code>I<sup>1</sup></code>
is the outer most implicit receiver and the <code>I<sup>m</sup></code> is the inner most.

2. **Checking** given an extension function with receivers
<code>T<sup>1</sup></code>...<code>T<sup>n</sup></code> and a receiver scope type of `R`
and implicit receiver scopes in the scope tower of
<code>I<sup>1</sup></code>...<code>I<sup>m</sup></code> an extension is valid if `R` :>
<code>T<sup>n</sup></code> and there exists some permutation of (<code>T<sup>i</sup></code>,
<code>I<sup>j</sup></code>) in `i` `1`...`n-1` and `j` in `1`..`m` where
<code>I<sup>j</sup></code> :> <code>T<sup>i</sup></code> and for each element in the
permutation all `i` and `j` values are in increasing order. If multiple valid permutations are
possible then the last permutation is selected when the permutations are ordered by the values
of `i` and `j`.

3. **Report ambiguous calls** If multiple valid candidates are possible the call is ambiguous
and an error is reported.

##### Implementation note

Finding a last valid permutation can be done efficiently if the permutations are produced by
pairing  <code>T<sup>n-1</sup></code> to the last <code>I<sup>j</sup></code> where
<code>T<sup>n-1</sup></code> :> <code>I<sup>j</sup></code> and then pairing
<code>T<sup>n-2</sup></code> to some <code>I<sup>k</sup></code> where `k` < `j` until a valid
permutation is found. The first valid pairing found will be the last permutation given the
ordering described above. This can be accomplished in worst case *O(nm)* steps.

#### Second variant: Unordered resolution

The resolution could be done in following steps:
1. **Create a list of actual contexts** `G, A, B, C, ...` where `G` is a global context (global context usually means file, but also could be used to describe top level script implicit receiver). Top level non-extension functions have only `G` as context. Class members have that class as context. Extension function have their receivers added to context where they are defined. Running a function with receiver adds that receiver to the list of contexts where this function is defined.Types in the list could be duplicating. We will call those actual types `Cy` where `y` is the index
2. **Checking** the definition of the function. Function receiver list is written in form of `[R1, R2, R3]`. Duplicate types or ambiguities throw compile error. Types could have parameters defined outside type list like `fun <T> [T, Operation<T>].doSomething()`. We will call receiver types `Rx` where `x` is the index.
3. **Binding** of extension function. Each of types `R` in receiver set is matched against the elements of context list from right to left, binding it to first match and thus creating a map `Rx to Cy`. If there is a bound pair for each of `Rx` then function is considered resolved and bound to context. Multiple `R` could be bound to the same context `C`, so it is possible to have just one context for multiple receiver types if it matches them both.

*This* resolution is done via declared receiver types `Rx` which are then substituted by actual runtime objects representing `Cy`. *Since the binding is done to the closest receiver matching the type, *this* will also represent the closest receiver of matching type*.

##### Compatibility check

* `[A].f === A.f`.Extension function with single argument should work exactly like existing extension function. It seems like it does. It is resolved and bound to the closest context matching its type.

* [Match current member receivers strategy](https://discuss.kotlinlang.org/t/compound-extension/10722/17?u=darksnake). Seems to be working the same way. It resolves to the closest context even if this context implements both receivers.

### Disambiguation of `this`

The `this` keyword refers to the most nested receiver. Reference to an outer receiver can be disambiguated by using the simple type name (without type parameters). In cases where that is not possible (such as a function type expression) a type alias with a simple name would need to be used in the declaration.

For example,

```kotlin
fun (A, B, C).doSomething() = this@A.amember(this@B.bmember(this.cmember())
```

### Semantics

A synthetic receiver parameter is introduced for every receiver type in a compound receiver.

For example,

```kotlin
fun (A, B, C).doSomething() = this@A.amember(this@B.bmember(this.cmember())
```

would roughly translate into the Java function

```Java
  void doSomething(A receiver2, B receiver1, C receiver0) {
      receiver2.amember(receiver1.bmember(receiver0.cmember()))
  )
```

A call to a compound receiver collects ambient receivers into the synthetic parameter,

```kotlin
var a: A
var b: B
var c: C

with (a) {
    with (b) {
        with (c) {
            doSomething()
        }
    }
}
```

would roughly translate into,

```Java
doSomething(a, b, c)
```

### `With` construct and `run` simplification

There are two standard ways to add a receiver scope: `with` and `run` functions. If the proposal is accepted,  there should be several new functions in `with` famiily. For example:

```kotlin
inline fun <A,B> with(block: (A,B).() -> Unit)
```

Currently `run` function is seldom needed to properly call member extension when the member scope and extension scope are switched. If unordered resolution is approved, then this problem is automatically solved since the order of actual receivers does not matter.

## Future development and relation to other features

### KEEP-87
This proposal obviously covers a lot of use-cases defined in [KEEP-87](https://github.com/Kotlin/KEEP/pull/87) formerly known as typ-classes. The key difference is that KEEP-87 uses implicit type substitution for additional receivers, while here we propose to use explicit object as a context. On the one hand, this proposal requires additional lines of code to explicitely define the context, on the other hand explicit is better than implicit and in this proposals it is possible to used stateful objects as contexts. If not for implicit receiver resolution, both proposals are quite similar.

### Reflections

Obviously, the full implementation of proposal requires change in how reflections and metadata works. But it is possible to allow only double receivers at first (it covers most of current use-cases). Double receivers are not quite hard to do since they already are present both in metadata and reflections as member extensions.

### Evolution: file level context
`G` in unordered resolution scheme is basically resolved to file which does not have any type of its own. But if there was a way to bind a context or a set of contexts to the file itself (not proposing any syntax solution, but it should be quite easy). Then everywhere in my schemes above we will just replace `G` with `G, F1, F2`. Meaning that all classes and functions in this file will have additional implicit contexts, just like type-classes. Of-course, it means that `F`s could only be singletons in this case.

*This mechanism is in fact currently used in KEEP-75 for script implicit receivers. But of course, for code, it should be explicit and only singleton objects will do.*

### Evolution: extension classes and interfaces
We consider a class or interface to be top-level non-empty context. It seems to be not so hard to add a set of external contexts to the class context. It could look like `class [T1,T2].MyClass{}`. In this case the instance of this class could be created only in context bound to both `T1` and `T2` and all members of this class will have additional implicit contexts, meaning member of `MyClass` will be able to call members of `T1` without additional declarations. From the implementation point of view, the instances of `T1` and `T2` could be class constructor implicit parameters (or a single map parameter, which is probably better), it should not violate anything, even Java compatibility (you will have to pass instances of contexts to create class instance). The interfaces could work the same way, just provide synthetic (invisible) property which holds the `Map<KClass, Any>` of actual receiver.

**Note:** There will be some problem with class type parameter syntax here since unlike functions, type parameters in classes are declared after class name, but probably it could be solved.

This second proposal is in fact much more flexible than first one since we can use non-singleton contexts and define them explicitly on class creation. Also, both ideas probably could cover most of type-classes use-cases.