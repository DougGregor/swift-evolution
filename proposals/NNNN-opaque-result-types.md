# Opaque Result Types for Public Interfaces

* Proposal: [SE-NNNN](NNNN-hidden-result-types.md)
* Author: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting review**
* Pull Request: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)

## Introduction

This proposal introduces the ability to "hide" the result types of specific functions from the caller in public interfaces. Instead of providing a specific concrete type, such functions will return an unknown-but-unique type described only by its capabilities, e.g., a `Collection` with a specific `Element` type. 
For example, [SE-0222](https://github.com/apple/swift-evolution/blob/master/proposals/0222-lazy-compactmap-sequence.md) introduces a new standard library type `LazyCompactMapCollection` for the sole purpose of describing the result type of `compactMap(_:)`:

```swift
extension LazyMapCollection {
  public func compactMap<U>(_ transform: @escaping (Element) -> U?)
      -> LazyCompactMapCollection<Base, U> {
    return LazyCompactMapCollection<Base, U>(/*...*/)
  }
}
```

With this proposal, the standard library could hide the result type, exposing it as "an opaque `Collection` whose `Element` type is `U`:

```swift
private struct LazyCompactMapCollection<Base: Collection, Element> { ... }

extension LazyMapCollection {
  public func compactMap<U>(_ transform: @escaping (Element) -> U?)
      -> opaque Collection where _.Element == U {
    return LazyCompactMapCollection<Base, U>(/*...*/)
  }
}
```

The implementation of `compactMap(_:)` can still return an instance of `LazyCompactMapCollection`, but now `LazyCompactMapCollection` can be private: its identity is hidden from clients, and could change from one version of the library to the next without breaking those clients, because the actual type identity was never exposed. This allows us to provide potentially-more-efficient implementations without expanding the surface area of the library.

Swift-evolution thread: [Opaque result types](https://forums.swift.org/t/opaque-result-types/15645)

## Motivation

Libraries in Swift often end up having to expose a number of generic types in their public interface to describe the result of generic operations. `LazyCompactMapCollection `, noted in the introduction, would be one such type in the standard library, along with existing types like `EnumeratedSequence`, `FlattenSequence`, `JoinedSequence`, and many others (including several `Lazy` variants). These generic types, which are needed to express the results of generic algorithms (`enumerated`, `flatten`, `joined`), are generally not of interest to users, but can significantly increase the surface area of the library.

## Proposed solution

This proposal introduces an `opaque` type that can be used to describe an opaque result type for a function. `opaque` types can only be used in the result type of a function, the type of a property, or the element type of a subscript declaration. An `opaque` type is backed by a specific concrete type, but that type is only known to the implementation of that function/property/subscript. Everywhere else, the type is opaque, and is described by its characteristics and originating function/property/subscript. For example:

```swift
func makeMeACollection<T>(_: T.Type)
     -> opaque MutableCollection & RangeReplaceableCollection where _.Element == T {
   return [T]()   // okay: an array of T satisfies all of the requirements
}
```

The syntax of the "opaque" type is derived from the discussions of [generalized existentials](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md). Essentially, following the `opaque` keyword is a class, protocol, `AnyObject`, or composition thereof (joined with `&`), optionally followed by a `where` clause indicating other requirements. Within that `where` clause, one can use `_` to refer to the opaque result type itself, so `_.Element` would refer to the `Element` associated type of the `opaque` type (which otherwise cannot be named).

A caller to `makeMeACollection(_:)` can rely on the result type having all of the requirements listed. For example:

```swift
var c = makeMeACollection(Int.self)
c.append(17)         // okay: it's a RangeReplaceableCollection with Element == Int
c[c.startIndex] = 42 // okay: it's a MutableCollection with Element == Int
print(c.reversed())  // okay: all Collection/Sequence operations are available

func foo<C: Collection>(_ : C) { }
foo(c)               // okay: unlike existentials, opaque types work with generics
```

Moreover, these types can be used freely with other generics, e.g., forming a collection of the results:

```swift
var cc = makeMeACollection(type(of: c))
cc.append(c)         // okay: Element == the result type of makeMeACollection
var c2 = makeMeACollection(Int.self)
cc.append(c2)        // okay: Element == the result type of makeMeACollection
```

An opaque result type is not considered equivalent to its underlying type by the static type system:

```swift
var intArray = [Int]()
cc.append(intArray)         // error: [Int] is not known to equal the result type of makeMeACollection
```

However, as with generic type parameters, one can inspect an opaque type's underlying type at runtime. For example, a conditional cast could determine whether the result of `makeMeACollection` is of a particular type:

```swift
if let arr = makeMeACollection(Int.self) as? [Int] {
  print("It's an [Int], \(arr)\n")
} else {
  print("Guessed wrong")
}
```

In other words, opaque result types are only opaque to the static type system: they don't exist at runtime.

### Implementing a function returning an opaque type 
The implementation of a function returning an opaque type must return a value of the same concrete type `T` from each `return` statement, and `T` must meet all of the constraints stated on the opaque type. For example:

```swift
protocol P { }
extension Int : P { }
extension String : P { }

func f1() -> opaque P {
  return "opaque"
}

func f2(i: Int) -> opaque P {   // okay: both returns produce Int
  if i > 10 { return i }
  return 0;
}

func f2(flip: Bool) -> opaque P {
  if flip { return 17 }
  return "a string"       // error: different return types Int and String
}

func f3() -> opaque P {
  return 3.1419           // error: Double does not conform to P
}

func f4() -> opaque P {
  let p: P = "hello"
  return p                // error: protocol type P does not conform to P
}

func f5() -> opaque P {
  return f1()             // okay: f1() returns an opaque type that conforms to P
}

protocol Initializable { init() }

func f6<T: P & Initializable>(_: T.Type) -> opaque P {
  return T()              // okay: T will always be a concrete type conforming to P
}
```

These rules guarantee that there is a single concrete type produced by any call to the function. The concrete type can depend on the generic type arguments (as in the `f6()` example), but must be consistent across all `return` statements.

Note that recursive calls *are* allowed, and are known to produce a value of the same concrete type, but that the concrete type itself is not known:

```swift
func f7(_ i: Int) -> opaque P {
  if i == 0 {
    return f7(1)                 // okay: returning the opaque result type of f(), similar to f5()
  else if i < 0 {
    let result: Int = f7(-i)     // error: opaque result type of f() is not convertible to Int
    return result
  } else {
    return 0
  }
}
```

Of course, there must be at least one `return` statement that provides a concrete type!

Opaque result types can be used with properties. For example:

```swift
extension Collection {
  var lazy: opaque Collection where _.Element == Element { ... }
}
```

For computed properties, the concrete type is determined by the `return` statements in the getter. Opaque result types can also be used in stored properties that have an initializer, in which case the concrete type is the type of the initializer:

```swift
var strings: opaque Collection where _.Element == String = ["hello", "world"]
```

### "Naming" the opaque result type
While one can use type inference to declare variables of the opaque result type of a function, there is no direct way to name the opaque result type:

```swift
func f1() -> opaque P { /* ... */ }

let vf1 = f1()    // type of vf1 is the opaque result type of f1()
```

However, the type inference used to satisfy associated type requirements can be used to give a name to the opaque result type. For example:

```swift
protocol P { }

protocol Q {
  associatedtype SomeType: P
  
  func someValue() -> SomeType
}

struct S: Q {
  func someValue() -> opaque P {
    /* ... */
  }
  
  /* infers typealias SomeType = opaque result type of S.someValue() */
}

let sv: S.SomeType     // okay: names the opaque result type of S.someValue()
sv = S().someValue()   // okay: returns the same opaque result type
```

### Opaque result types vs. existentials
On the surface, opaque types are quite similar to existential types: in each case, the specific concrete type is unknown to the static type system, and can be manipulated only through the stated capabilities (e.g., protocol and superclass constraints). For example:

```swift
protocol P
  func foo()
}

func f() -> P { /* ... */ }
func g() -> opaque P { /* ... */ }

f().foo()   // okay
g().foo()   // okay

let pf: P = f()   // okay
let pg: P = g()   // okay
```

The primary difference is that the concrete type behind an opaque type is constant at run-time, while an existential's type can change. For example, assuming both `Int` and `String` conform to `P`, `f()` could return a value of a different concrete type:

```swift
func f() -> P {
  if Bool.random() {
    return 17
  } else {
    return "hello, existential"
  }
}
```

This means that, for example, an array populated by calls to `f()` could be heterogeneous:

```swift
let fArray = [f(), f(), f()]  // contains a mix of String and Int at run-time
```

With an opaque type, there is a single concrete type:

```swift
let gArray = [g(), g(), g()]  // homogeneous array of g()'s opaque result type
```

The guarantee of a single concrete type allows opaque result types to compose much better with generics. For example, an opaque result type can make use of protocols with `Self` requirements and use those values with generic operations like `Collection.sort()`:

```swift
func h() -> opaque Comparable { return /* ... */ }

var hArray = [h(), h(), h()]
hArray.sort()   // okay! the Element type is Comparable, and all types are the same
```

Existentials do not allow such an operation, even with [generalized existentials](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md), because two values of the same existential type may have different types at runtime.

### Interaction with conditional conformances

When a generic function returns an adapter type, it's not uncommon for the adapter to use [conditional conformances](https://github.com/apple/swift-evolution/blob/master/proposals/0143-conditional-conformances.md) to reflect the capabilities of its underlying type parameters. For example, consider the `reversed()` operation:

```swift
extension BidirectionalCollection {
  public func reversed() -> ReversedCollection<Self> {
    return ReversedCollection<Self>(...)
  }
}
```

`ReversedCollection` is always a `BidirectionalCollection`, with a conditional conformance to `RandomAccessCollection`:

```swift
public struct ReversedCollection<C: BidirectionalCollection>: BidirectionalCollection {
  /* ... */
}

extension ReversedCollection: RandomAccessCollection
    where C: RandomAccessCollection {
  /* ... */
}
```

What happens if we hid the `ReversedCollection` adapter type behind an opaque result type?

```swift
extension BidirectionalCollection {
  public func reversed() -> opaque BidirectionalCollection
      where _.Element == Element {
    return ReversedCollection<Self>(...)
  }
}
```

Now, clients that call `reversed()` on a `RandomAccessCollection` (like an array), would get back something that is only known to be a `BidirectionalCollection`: there would be no way to treat it as a `RandomAccessCollection`. The library could provide another overload of `reversed()`:

```swift
extension RandomAccessCollection {
  public func reversed() -> opaque RandomAccessCollection
      where _.Element == Element {
    return ReversedCollection<Self>(...)
  }
}
```

However, doing so is messy, and the client would have no way to know that the type returned by the two `reversed()` functions are, in fact, the same.

Alternatively, we can introduce syntax to describe the conditional conformances of an opaque result type. For example, we could state that the result of `reversed()` is *also* a `RandomAccessCollection` when `Self` is a `RandomAccessCollection`. One possible syntax:

```swift
extension BidirectionalCollection {
  public func reversed() -> opaque BidirectionalCollection
      where _.Element == Element
      where Self: RandomAccessCollection -> _: RandomAccessCollection {      
    return ReversedCollection<Self>(...)
  }
}
```

Here, we add a second where claus that states the conditional requirements (`Self: RandomAccessCollection`) and the consequence of that conditional requirement (`_`, the opaque result type, conforms to `RandomAccessCollection`). One could have multiple conditional clauses, e.g.,

```swift
extension BidirectionalCollection {
  public func reversed() -> opaque BidirectionalCollection
      where _.Element == Element
      where Self: RandomAccessCollection -> _: RandomAccessCollection
      where Self: MutableCollection -> _: MutableCollection {
    return ReversedCollection<Self>(...)
  }
}
```

Here, the opaque result type conforms to `MutableCollection` when the `Self` type conforms to `MutableCollection`. This conditional result is independent of whether the opaque result type conforms to `RandomAccessCollection`.

## Detailed design

### Grammar of opaque result types

The grammatical productions for opaque result types aren't straightforward:

```
type ::= 'opaque' type where-clause[opt] opaque-conditional-requirement*
     |   '_'
     
opaque-conditional-requirement ::= where-clause '->' requirement-list
```

The first `type` production introduces the `opaque` type with its optional `where` clause and conditional requirements; the second `type` production introduces the contextual type `_` to describe the opaque result type.

The conditional requirements is a set of `where` clauses, each followed by a `requirement-list`.

### Restrictions on opaque result types

Opaque result types can only be used within the result type of a function, the type of a variable, or the element type of a subscript. However, they can occur within some part of the structure of that type. For example, one can return an optional opaque result type:

```swift
func f(flip: Bool) -> (opaque P)? {
  if flip {
    return 1       // concrete type for the opaque result type is Int
  }
  
  return nil
}
```

Opaque result types cannot be used in the requirements of a protocol:

```swift
protocol Q {
  func f() -> opaque P        // error: cannot use opaque result type within a protocol
}
```

Associated types provide a better way to model the same problem, and the requirements can then be satisfied by a function that produces an opaque result type.

Contextual named types (e.g., `.Element`) can only be used within the `where` clause of an opaque result type. [Generalized existentials](https://github.com/austinzheng/swift-evolution/blob/az-existentials/proposals/XXXX-enhanced-existentials.md) are likely to expand the usefulness of this syntax.

### Single opaque result type per entity
As a (possibly temporary) restriction, a particular function/variable/subscript can only contain a single opaque result type, so the following is ill-formed:

```swift
func g(flip: Bool) -> (opaque P, opaque P) {  // error: two opaque result types
  return (1, "hello")
}
```

While it is technically possible to support multiple opaque result types in a given function/variable/subscript, the contextual named type syntax starts to break down. For example, say we want to return two opaque types that conform to `Collection` but whose element types are the same: 

```swift
func g() -> (opaque Collection where _.Element: Equatable, opaque Collection) 
    where /* Element types of both opaque types are equivalent? */ {
  // ...
}
```

The `_.Element` syntax that works nicely for saying that the `Element` type of the first `Collection` is `Equatable` doesn't allow us to relate that `Element` type to the second `Collection` element type. We would need to invent more syntax, e.g., `where _.0.Element == _.1.Element`.

### Uniqueness of opaque result types

Opaque result types are uniqued based on the function/property/subscript and any generic type arguments. For example:

```swift
func makeOpaque<T>(_: T.Type) -> opaque Any { /* ... */ }
var x = makeOpaque(Int.self)
x = makeOpaque(Double.self)  // error: "opaque" type from makeOpaque<Double> is distinct from makeOpaque<Int>
```

This includes any generic type arguments from outer contexts, e.g.,

```swift
extension Array where Element: Comparable {
  func opaqueSorted() -> opaque Sequence where _.Element == Element { /* ... * }
}

var x = [1, 2, 3]. opaqueSorted()
x = ["a", "b", "c"].opaqueSorted()  // error: opaque result types for [Int].opaqueSorted() and [String].opaqueSorted() differ
```

### Ambiguity with `where` clauses

Opaque result types contain optional trailing `where` clauses, as do function and subscript declarations, leading to a parsing ambiguity. For example:

```swift
func sorted<C: RandomAccessCollection>(_: C) -> opaque RandomAccessCollection
    where C.Element: Comparable, _.Element == C.Element {
  /* ... */
}
```

Is the `where` clause part of the opaque result type or part of the function itself? The [maximal munch](https://en.wikipedia.org/wiki/Maximal_munch) principle implies that it should be part of the opaque result type, e.g., that this declaration is equivalent to:

```swift
func sorted<C: RandomAccessCollection>(_: C)
     -> (opaque RandomAccessCollection where C.Element: Comparable, _.Element == C.Element) {
  /* ... */
}
```

rather than being part of the declaration itself:

```swift
func sorted<C: RandomAccessCollection>(_: C) -> (opaque RandomAccessCollection)
    where C.Element: Comparable, _.Element == C.Element {
  /* ... */
}
```

Note that the last of these won't compile, because `_.Element` does not make sense outside of the opaque result type.

We propose to follow the maximal munch principle here, associating the `where` clause with the opaque result type, because it is strictly better than the alternative:

* All constraints of the result type are implied constraints of the function itself, so stating the `C.Element: Comparable` constraint as part of opaque result type also makes that a constraint on the generic function as a whole.
* The generic type parameters are already part of the uniqueness criteria for opaque result types, so there is little downside to exposing all of the known capabilities of the generic type parameters in the opaque result type, because the client already has to satisfy those constraints.

To the second point, consider:

```swift
func weirdSorted<C: RandomAccessCollection>(_: C)
  -> (opaque RandomAccessCollection where _.Element: Equatable,
                                        _.Element == C.Element)
  where C.Element: Comparable {
/* ... */
}
```

This could be interpreted to mean that the resulting opaque result type only guarantees that its element type is `Equatable`, even though the `weirdSorted` function only works with collections whose element types are `Comparable`, and the argument and result element types are equivalent. For simplicity, we state that there is no difference between the opaque result types of `weirdSorted()` and any of the former `sorted()` examples, i.e., the entire generic signature of the function is used to describe the capabilities of the opaque result type.

### Implementation strategy

From an implementation standpoint, a client of a function with an opaque result type needs to treat values of that result type like any other resilient value type: its size, alignment, layout, and operations are unknown.

However, when the body of the function is known to the client (e.g., due to inlining or because the client is in the same compilation unit as the function), the compiler's optimizer will have access to the specific concrete type, eliminating the indirection cost of the opaque result type.

## Source compatibility

Opaque result types are purely additive. If they were to be adopted in the standard library, that change could affect source compatibility.

## Effect on ABI stability

Opaque result types are an ABI-additive feature. If they were to be adopted in the standard library, it would affect the standard library ABI. This feature can be useful for ABI stability, because it would allow the standard library to hide a large number of detail types (`EnumeratedSequence`, `FlattenSequence`, `JoinedSequence`, and so on) from both source code and binaries, allowing more customization in the future.

## Effect on API resilience

Opaque result types are part of the result type of a function/type of a variable/element type of a subscript, which cannot be changed without affecting API resilience.

We could allow an API originally specified using an opaque result type to later evolve to specify the specific result type. The result type itself would have to become visible to clients, and this might affect source compatibility, but (mangled name aside) such a change would be resilient.

## Rust's `impl Trait`

The proposed Swift feature is largely based on Rust's `impl Trait` language feature, described by [Rust RFC 1522](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md) and extended by [Rust RFC 1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md). There are only a small number of differences between this feature as expressed in Swift vs. Rust's `impl Trait` as described in RFC 1522:

* Swift's need for a stable ABI necessitates translation of opaque result types as resilient types, which is unnecessary in Rust's model, where the concrete type can always be used for code generation.
* Swift's opaque result types are fully opaque, because Swift doesn't have pass-through protocols like Rust's `Send` trait, which simplifies the type checking problem slightly.
* Due to associated type inference, Swift already has a way to "name" an opaque result type in some cases.
* We're not even going to mention the use of `opaque` in argument position, because it's a distraction for the purposes of this proposal; see [Rust RFC 1951](https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md).
* Rust [didn't tackle the issue of conditional constraints](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md#compatibility-with-conditional-trait-bounds).


## Alternatives considered

Let implementation details continue to leak out of generic libraries.

