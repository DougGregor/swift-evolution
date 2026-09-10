# Ownership for Subscript Parameters

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [swiftlang/swift#91992](https://github.com/swiftlang/swift/pull/91992)
* Experimental Feature Flag: `SubscriptParametersWithOwnership`
* Review: ([pitch](https://forums.swift.org/...))

## Summary of changes

This proposal introduces support for `inout`, `borrowing`, and `consuming` on parameters of subscripts. It brings the capabilities of subscript parameters in line with functions and initializers.

## Motivation

Subscript parameters have always had a limitation that function and initializer parameters didn't, namely that `inout` parameters were not possible:

```swift
struct X {
  subscript(oldValue: inout Int) -> Int { // error: 'inout' may only be used on function or initializer parameters}
    get {
      // ...
    }
    
    set {
      oldValue = newValue
      // set newValue
    }
  }
}
```

With the introduction of non-copyable types, the limitation grew larger: it's not possible to have a subscript parameter of non-copyable type, because neither `borrowing` nor `consuming` parameters are supported for subscripts.

The primary motivation for this proposal is consistency. The need for it also came up in the context of extending keypaths to support noncopyable root types, but that's an implementation detail rather than a motivation for the language itself.

## Proposed solution

This proposal lifts this restriction on subscript parameters, bringing them into line with functions and initializers. The code in the "Motivation" section will be accepted. Additionally, one can also use a non-copyable type as a parameter, either borrowing it:

```swift
struct Y {
  subscript(resourceHandle: borrowing Resource) -> Data {
    get { ... }
    set { ... }
  }
}
```

Or consuming it:

```swift
struct Z {
  subscript(resourceHandle: consuming Resource) -> Span<UInt>{
    borrow { ... }
    mutate { ... }
  }
}
```

## Detailed design

The most interesting aspect of the design for this feature center on the access to the arguments themseves. For example, when passing an argument with some kind of ownership specifier (`inout`, `borrowing`, `consuming`), the argument must remain fixed while the function call happens. For example, when calling a function and passing a local variable to an `inout`, nothing else can access that local variable for the duration of the call. Doing so will produce an "exclusivity violation", like this:

```swift
func doSomething<T>(on value: inout T, body: () -> Void) { }
var x = 17
doSomething(on: &x) { _ in print(x) } // error: exclusivity violation
```

Subscripts make this more interesting, because using a subscript might involve more than one call. For example:

```swift
struct MyMapping {
  subscript(i: Int) -> String { 
    get { ... }
    set { ... }
  }
}

var mapping = MyMapping(...)
doSomething(on: &mapping[17]) { ... }
```

To pass `mapping[17]` as `inout` , Swift will first call the subscript's `get` operation and store the result in a temporary. The address of that temporary is then passed down to `doSomething(on:)`. Once that function returns, Swift calls the subscript's `set` operation, providing it with the temporary value.

For an integer argument, the same value `i` is passed both times. However, when the subscript argument is `borrowing` or `inout`, the argument must remain fixed for the duration of the call to `doSomething(on:)`, so that the same index value is provided to both the `get` and the `set`. Consider another case with a borrowing subscript argument:

```swift
struct NC: ~Copyable { }

struct A {
  subscript(nc: borrowing NC) -> Int {
    get { ... }
    set { ... }
  }
}

func f<T, U>(_: inout T, _: inout U) { }

var a = A()
var nc = NC()
f(&a[nc], &nc) // error: exclusivity conflict because "nc" is borrowed by the subscript, mutated by the inout
```

When the subscript argument is `consuming`, there is no sensible way to use the `get`/`set` pair in this manner: the argument would be consumed by the call to `get`, leaving no argument when calling the `set` at the end. Therefore, a `consuming` subscript parameter requires that one either have a `get`-only property or use coroutine accessors (`borrow` or `mutate`, whether `yielding` or not).

## Source compatibility

This is a pure language extension with no effect on source compatibility.

## ABI compatibility

This feature's ABI is derived from that of functions with ownership specifiers.
