# Unicode table availability in Embedded Swift

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Doug Gregor](https://github.com/DougGregor)
* Review Manager: TBD
* Status: **Awaiting review**
* Vision: [Embedded Swift](https://github.com/swiftlang/swift-evolution/blob/main/visions/embedded-swift.md)
* Implementation: [swiftlang/swift#85324](https://github.com/swiftlang/swift/pull/85324) 
* Review: ([pitch](https://forums.swift.org/...))

## Summary of Change

The data tables required to support Unicode operations including normalization and scalar properties are prohibitively large for some Embedded Swift clients. This proposal introduces a new availability domain for Embedded Swift, `@available(Unicode)`, that makes it easier to write Swift code that avoids making use of those tables, leading to smaller binaries.

## Introduction

[Embedded Swift](https://github.com/swiftlang/swift-evolution/blob/main/visions/embedded-swift.md) is a subset of the Swift language and standard library that aims for small code size. It leaves out some language features, such as dynamically casting to an `any` type and uses of unspecialized generics, that require a large runtime or a lot of metadata. Similarly, it leaves out some standard library features (such as `Codable`) that depend on those language features.

Unicode support is an important aspect of the Swift standard library that has a significant code size impact due to a number of large static tables that are required for normalization, grapheme breaking, and querying the properties of scalars. For some applications, Unicode support is required and the code size impact (from dozens to hundreds of kilobytes depending on API usage) is acceptable. Other applications might want to avoid using APIs that depend on the Unicode tables to keep code size smaller.

## Motivation

At present, non-Embedded Swift always includes full Unicode support. Embedded Swift separates the Unicode tables into a separate static library (`swiftUnicodeDataTables`) that can be linked in explicitly to enable Unicode support. However, the failure mode for an Embedded Swift application that doesn't link `swiftUnicodeDataTables` but somehow makes use of the tables (say, by calling `uppercased()` on a ` String`) is a link error referencing one or more `_swift_stdlib` symbols. From these link failures, it can be very hard to determine exactly what Swift code is depending on the Unicode tables, making it hard to develop in (or port code to) Embedded Swift without the Unicode tables.

## Proposed solution

This proposal introduces a custom availability domain named `Unicode`. Code that does not enable the `Unicode` availability domain will not need to link the Unicode data tables. Within the standard library, every API that depends on the larger Unicode tables will be annotated with `@available(Unicode)`. For example:

```swift
extension StringProtocol {
  @available(Unicode)
  func uppercased() -> String { /* depends on Unicode tables */ }
}
```

User code can do one of three things:

* Assume that `Unicode` is always available, which is the default for non-Embedded Swift. It is also applicable to Embedded Swift where the program is willing to link in the Unicode tables.
* Assume that `Unicode` is never available. This is applicable for Embedded Swift where one wants to avoid the code size cost of the Unicode tables. APIs marked with `@available(Unicode)` are unavailable.
* Annotate anything that uses a Unicode table with `@available(Unicode)`, transitively. This applies to the standard library and any other library that has some functionality that depends on the Unicode tables and other functionality that does not. The library can be compiled with or without the Unicode tables.

## Detailed design

The `Unicode` availability domain is used to annotate a number of APIs in the standard library that depend on the Unicode tables. The detailed design covers the specific APIs that will be annotated, as well as how Swift code can opt in to using (or not using) the Unicode tables.

### Unicode table categories 

The Unicode tables are divided into several different categories, each of which enables a set of standard library APIs. The categories and very rough code-size contributions are as follows:

* Grapheme breaking (~5kb): breaking a sequence of Unicode code points into grapheme clusters, needed to provide `String`'s conformance to `Collection` and express the notion that a `Character` is a single grapheme cluster.
* Normalization (>50kb): normalized comparisons of `String`, `Substring`, and `Character`, including their `Equatable`, `Comparable`, and `Hashable` conformances. This also includes the `Character` operations `isUppercase`, `isLowercase`, `isCased`; `StringProtocol`'s `hasPrefix` and `hasSuffix`, `lowercased()` and `uppercased`(), and `StringProtocol`'s inheritance from `Equatable` and `Hashable`.
* [Scalar properties](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0211-unicode-scalar-properties.md) (> 500kb): classifying scalars into, e.g., digits, operators, identifier characters, emoji, and so on.

In this proposal we propose a single availability domain, `Unicode`, that covers the second two categories (normalization and scalar properties) that contribute most to code size and are easily separable from the core of the library. See Alternatives Considered for different approaches we could have taken.

#### Grapheme breaking

Swift's `Character` type describes a single Unicode grapheme cluster, which can consist of several Unicode scalars. Both `String` and `Substring`'s `Collection` conformances use `Character` as the `Element` type, and core collection operations such as `index(after:)` need to consult the tables to find the appropriate places to break up the sequence of Unicode scalars into grapheme clusters.

In practice, it is hard to use `String` without these operations: one cannot validate the correctness of the Unicode scalars in a string without the grapheme breaking logic, nor correctly handle insertions or removal of Unicode scalars from the string. Therefore, the tables that support grapheme breaking (~5kb) are assumed to always be available. Note that, if the tables end up not being used (for example, because an Embedded Swift program doesn't use `String` in any nontrivial way), they will won't end up in the final binary. However, this does mean that there is no way to easily avoid using one of the String APIs that depends on this table.

#### Normalization

String comparisons in Swift depend on [Unicode normalization](https://en.wikipedia.org/wiki/Unicode_equivalence), so that different representations of the same character compare equivalently. For example, the strings `"\u{212B}"` and `"\u{00C5}"` are represented with different Unicode scalars, but compare equal because both are `"Å"`.

The normalization tables add more than 50kb to the code size. The corresponding APIs, listed below, are marked with `@available(Unicode)`:

```swift
@available(Unicode)
extension Character: Equatable, Comparable, Hashable {
  static func ==(lhs: Character, rhs: Character) -> Bool
  static func <(lhs: Character, rhs: Character) -> Bool
  func hash(into hasher: inout Hasher)
 
  var isUppercase: Bool
  var isLowercase: Bool
  var isCased: Bool
}

@available(Unicode)
extension String: Equatable, Comparable, Hashable {
  static func ==(lhs: String, rhs: String) -> Bool
  static func <(lhs: String, rhs: String) -> Bool
  func hash(into hasher: inout Hasher)  
}

@available(Unicode)
extension Substring: Equatable, Comparable, Hashable {
  static func ==(lhs: Substring, rhs: Substring) -> Bool
  static func <(lhs: Substring, rhs: Substring) -> Bool
  func hash(into hasher: inout Hasher) 
}
```

The definition of `StringProtocol` causes some problems. Specifically, `StringProtocol` refines both `Hashable` and `Comparable`. However, the types that conform to `StringProtocol`, `String` and `Substring`, only provide `Hashable` and `Comparable` conformances when `Unicode` is available. There is currently no way to model that `StringProtocol`'s refinement of `Hashable` is `@available(Unicode)`.

We propose that `StringProtocol` not refine `Hashable`, `Equatable`, or `Comparable` in Embedded Swift. This is accomplished by introducing two new type aliases that different in Embedded vs. non-Embedded Swift:

```swift
#if hasFeature(Embedded)
public typealias UnicodeStringProtocol = StringProtocol & Hashable & Comparable
public typealias StringProtocolMixin = Any
#else
public typealias UnicodeStringProtocol = StringProtocol
public typealias StringProtocolMixin = Hashable & Comparable
#endif

```

`StringProtocol` is then defined as follows:

```swift
public protocol StringProtocol
  : BidirectionalCollection,
  TextOutputStream, TextOutputStreamable,
  LosslessStringConvertible, ExpressibleByStringInterpolation,
  StringProtocolMixin
  where Iterator.Element == Character,
        Index == String.Index,
        SubSequence: StringProtocol,
        StringInterpolation == DefaultStringInterpolation
{
  // ...
  
  @available(Unicode)
  func hasPrefix(_ prefix: String) -> Bool

  @available(Unicode)
  func hasSuffix(_ suffix: String) -> Bool

  @available(Unicode)
  func lowercased() -> String

  @available(Unicode)
  func uppercased() -> String
}  
```

Several operations within `StringProtocol` depend on the Unicode tables, so they are annotated as `@available(Unicode)`.

Note that a generic API that uses `StringProtocol` and depends on a conformance to `Hashable`, `Equatable`, or `Comparable` will compile correctly in non-Embedded Swift but fail to compile with Embedded Swift. For example:

```swift
extension StringProtocol {
  func compareStrings(_ lhs: Self, _ rhs: Self) -> Bool {
    lhs == rhs // currently well-formed, will break in Embedded Swift with this proposal
  }
}
```

The fix for such code is to extend `UnicodeStringProtocol` instead (or use `UnicodeStringProtocol` as the generic requirement), which mixes in the additional requirements. The portable implementation is, therefore:

```swift
extension UnicodeStringProtocol {
  @available(Unicode)
  func compareStrings(_ lhs: Self, _ rhs: Self) -> Bool {
    lhs == rhs // well-formed in both Embedded and non-Embedded
  }
}
```

Fortunately, `StringProtocol` is mostly only used within the standard library, and it is documented that [there should not be any conformances declared outside the standard library](https://developer.apple.com/documentation/swift/stringprotocol):

> Do not declare new conformances to `StringProtocol`. Only the `String`and `Substring` types in the standard library are valid conforming types.

#### Scalar properties

The scalar properties introduced in [SE-0211](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0211-unicode-scalar-properties.md) all have `Unicode` availability:

```swift
extension Unicode.Scalar {

  @available(Unicode)
  public struct Properties {
    // ...
  }

  @available(Unicode)
  public var properties: Properties { get }
}
```

The Unicode tables supporting these scalar properties contribute more than 500kb to the resulting binary if these APIs are used.

### Command-line interface

As noted previously, there are effectively three "modes" for the `Unicode` availability domain: always enabled, always disabled, and conditionally-enabled. The command-line interface to the Swift compiler offers options for all three:

* `-define-always-enabled-availability-domain Unicode`: The `Unicode` availability domain is always enabled, and code can make use of `@available(Unicode)`-annotated APIs freely. This is similar to a platform-specific availability annotation (such as `@available(macOS 12)` where the deployment target has been set to be set to at least that platform (e.g., macOS >= 12 deployment). This is the default behavior for non-Embedded Swift (where the Unicode tables are always present) and the current default behavior for Embedded Swift.
* `-define-disabled-availability-domain Unicode`: The `Unicode` availability domain is never available. Any code referencing an API marked `@available(Unicode)` needs to itself be marked `@available(Unicode)`, and all code annotated with `@available(Unicode)` will be deleted by the compiler. This matches with the desired behavior for Embedded Swift when not linking the Unicode tables.
* `-define-enabled-availability-domain Unicode`: The `Unicode` availability domain is conditionally available. Any code referencing an API marked `@available(Unicode)` must itself be marked as `@available(Unicode)`, but code that depends on `@available(Unicode)` is not deleted.  Swift code using this option could be safely built with `Unicode` either always-enabled or always-disabled. This mode is applicable to both Embedded and non-Embedded Swift.

When compiling for Embedded Swift, the Swift compiler should link the Unicode table support library unless `-define-disabled-availability-domain Unicode` was passed on the command line. Note that an Embedded Swift program using the default `alwaysEnabled` that doesn't actually touch the Unicode tables will likely not end up including the tables in the final binary, because the linker will strip them out as dead code, so implicitly linking the Unicode tables won't cause a regression in code size.

### Swift package manager

The Swift package manager needs to determine which of the options to pass down to a given Swift target. A Swift target should be able to opt in to supporting the `Unicode` availability domain. We can accomplish this by extending `SwiftSettings` with support for availability domains:

```swift
extension SwiftSetting {
  enum AvailabilityMode {
    case alwaysEnabled
    case disabled
    case conditional
  }

  static func availabilityDomain(
    _ name: String, 
    _ mode: SwiftSetting.AvailabilityMode = .conditional, 
    _ condition:  BuildSettingCondition?
  ) -> SwiftSetting
}
```

A SwiftPM target can define its relationship to the `Unicode` availability domain via settings. For example, a target can disable Unicode support using the setting:
```swift
swiftSettings: [
  .availabilityDomain("Unicode", .disabled)
]
```

Similarly, any target can enable diagnostics that ensure that `Unicode` API usage is always covered by `@available(Unicode)` with, e.g.,

```swift
swiftSettings: [
  .availabilityDomain("Unicode", .conditional)
]
```

If not provided, the default is assumed to be `alwaysEnabled`, which provides source compatibility. 

## Source compatibility

This proposal has no effect on programs compiled with non-Embedded Swift, because the `Unicode` availability domain is always-enabled by default, so users are not required to annotate anything with `@available(Unicode)`.

This proposal can break some code using Embedded Swift if it depends on `StringProtocol`'s refinement of `Equatable`, `Comparable`, or `Hashable`. Such code would need to adopt `UnicodeStringProtocol` in places where `StringProtocol` is used. This does introduce a new difference between the Embedded and non-Embedded Swift standard library.

Disabling the `Unicode` availability domain or making it conditionally available affects source compatibility, because the code will need add to `@available(Unicode)` annotations. However, this is an opt-in step, so it does not have any source-compatibility effect until the user requests it.

## ABI compatibility

The introduction of `Unicode` availability does not affect ABI. The change to `StringProtocol` does have an effect on ABI for Embedded Swift, but this is acceptable because Embedded Swift does not guarantee a stable ABI.

## Implications on adoption

This feature can be adopted on a per-module basis, adding `@available(Unicode)` annotations as needed. There are no deployment constraints on this availability, as it has no impact on code generation except that `Unicode`-available definitions will be removed when the `Unicode` availability domain is disabled.

Adding `@available(Unicode)` annotations to a module can cause modules that import it to fail to compile if those modules have also opted in to `Unicode` (whether via `disabled` or `conditional`).

## Future directions

There are other aspects of the Swift standard library that could potentially be modeled via availability domains. For example, various APIs require the system to provide a random number generator. Some platforms don't have a suitable random number generator, and it would be reasonable to have an availability domain covering the APIs that need random number generation so it's easier to avoid them where needed.

## Alternatives considered

### Introduce several availability domains

This proposal provides a single availability domain, `Unicode`, covering the Unicode normalization and scalar properties but not grapheme breaking. We could instead provide several availability domains:

* `UnicodeNormalization`: for Unicode normalization, e.g., `String` conformances to `Equatable`, `Comparable`, and `Hashable`.
* `UnicodeScalarProperties`: for Unicode scalar properties, as accessed via `Unicode.Scalar.properties`.
* (Optionally) `UnicodeGraphemeBreaking`: for grapheme breaking, e.g., `String` conformance to `Collection`, which is currently not included in the `Unicode` availability domain.

The advantage of introducing several availability domains like this is that one can better control code size at a finer granularity: if the 50kb from the normalization tables is acceptable but the 500kb from scalar properties is not, one can opt in to `UnicodeNormalization` but not `UnicodeScalarProperties`. 

The disadvantage is complexity: rather than developers choosing "Unicode or not", they'll need to reason about specifically which aspects of Unicode they want to adopt. This may be acceptable complexity in the embedded space, but it does complicate the standard library for all users to have so many such annotations.

### Custom availability domains

The `Unicode` availability domain could be modeled with a language feature for [custom availability domains](https://forums.swift.org/t/pitch-extensible-availability-checking/79308). For example, the Swift standard library itself could declare the `Unicode` availability domain using the syntax pitched in the aforementioned thread, e.g.,

```swift
@availabilityDomain(Unicode)
public let _unicodeDomain: Bool = true
```

To account for the three configurations we would need different implementations in the standard library:

* Always-enabled means the domain is always enabled and it is unchecked, written as:
  ```swift
  @availabilityDomain(Unicode)
  @unchecked
  @const public let _unicodeDomain: Bool = true
  ```

* Always-disabled means the domain is always disabled, written as:
  ```swift
  @availabilityDomain(Unicode)
  @const public let _unicodeDomain: Bool = false
  ```

* Conditionally-enabled leaves checking enabled but assumes the Unicode tables are already there:
  ```swift
  @availabilityDomain(Unicode)
  @const public let _unicodeDomain: Bool = true
  ```

This is somewhat less flexible than the current formulation because the decision about the `Unicode` domain is baked into the standard library itself. Other modules won't be able to "opt in" to conditionally enabling the domain without the standard library having done so.

### Provide similar functionality without using Unicode

When disabling Unicode support, some commonly-used functionality (like comparison of strings) is unavailable in Swift. This may require nontrivial changes to the code, for example to compare the underlying UTF-8 representation (without canonicalizing).

We could take a different approach in the Standard Library where disabling Unicode support means that we revert to non-Unicode-correct versions of the same constructs: `String` would retain its `Equatable` conformance, but equality would be defined as UTF-8 equality without normalization.

This approach would make more code compile as Embedded Swift with Unicode disabled, but that code would *behave differently* from non-Embedded Swift and Embedded Swift with Unicode enabled. This is a poor tradeoff: it's better to have restrictions that you can decide how to live with than to silently have the same code behave differently in different environments.

## Acknowledgments

Allan Shortlidge pitched and implemented custom availability traits in the compiler, which is the basis for this proposal.
