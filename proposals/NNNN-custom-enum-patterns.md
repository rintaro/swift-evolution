# Enumeration Case Patterns for Non-Enum Types

* Proposal: [SE-NNNN](NNNN-filename.md)  
* Authors: [Rintaro Ishizaki](https://github.com/rintaro)  
* Review Manager: TBD  
* Status: **Awaiting implementation**  
* Implementation: TBD  
* Feature Flags: TBD  
* Review: ([Pitch Discussion](https://forums.swift.org/...))

---

## Introduction

Enumeration case patterns allow matching specific cases of an enumeration type and matching and extracting associated values. However, their functionality is currently restricted to enum types. This proposal extends enumeration case pattern matching to non-enum types, enabling developers to define custom patterns for structs and classes, improving both flexibility and performance.

## Motivation

Pattern matching in Swift is a powerful construct for exhaustive handling of cases, but its current restriction to enum types limits its utility. This limitation leads to inefficiencies and syntactic inconsistencies when working with non-enum types that conceptually resemble enumerations. Developers working with struct-based models often need to create artificial enum wrappers for pattern matching, introducing unnecessary computation and boilerplate.

Consider this struct representing tagged text segments:

```swift
struct TaggedStringSegment {
    enum Kind {
        case number
        case link
        case quote
    }
    
    var kind: Kind
    var text: Substring
}
```

Pattern matching currently requires introducing an extra enum wrapper with associated values, just to gain access to enum-style pattern matching:

```swift
enum TaggedStringSegmentEnum {
    case number(Int)
    case link(URL)
    case quote(String)

    init(_ value: TaggedStringSegment) {
        switch value.kind {
        case .number:
            self = .number(Int(value.text)!)
        case .link:
            self = .link(URL(string: String(text)))
        case .quote:
            self = .quote(String(text))
        }
    }
}

func describe(segment: TaggedStringSegment) {
    switch TaggedStringSegmentEnum(segment) {
    case .number(0..<10): print("Small number")
    case .quote(let str): print("Quoted: '\(str)'")
    default: print("Other")  // Unused values like URL are wasted here.
    }
}
```

Alternatively, developers can expose computed properties and let the users manually choose which one to evaluate:

```swift
extension TaggedStringSegment {
    var asNumber: Int { Int(text)! }
    var asLink: URL { URL(string: String(text))! }
    var asQuote: String { String(text) }
}

func describe(segment: TaggedStringSegment) {
    switch segment.kind {
    case .number where 0..<10 ~= segment.asNumber:
        print("Small number")
    case .quote:
        print("Quoted: '\(segment.asQuote)'")
    default:
        print("Other")
    }
}
```

However, this approach makes it the user's responsibility to call the correct accessor for each case, which can easily lead to mistakes or runtime crashes.

This proposal addresses these issues by allowing `case` declarations to be defined directly on the type, preserving compile-time exhaustiveness checking, enabling safe and lazy evaluation of associated values, and providing a more consistent and expressive pattern-matching syntax.

## Proposed Solution

We introduce a `MatchableWithEnumCasePattern` protocol that allows structs and classes to define case pattern matching behavior directly.

Example:

```swift
struct TaggedStringSegment {
    enum Kind {
        case number
        case link
        case quote
    }
    
    var kind: Kind
    var text: Substring
}

extension TaggedStringSegment: MatchableWithEnumCasePattern {
    var enumCasePatternTag: Kind { self.kind } // Determines the case for matching.
    
    case number(Int) {
        get { Int(text)! }
    }

    case link(URL) {
        get { URL(string: String(text))! }
    }

    case quote(String) {
        get { String(text) }
    }
}
```

Now clients can match the `TaggedStringSegment` value directly with enumeration case patterns:

```swift
switch segment {
case .number(1..<100):
    print("Small number")
case .number(_)
    print("Big number")
case .link(let url):
    print("URL is \(url)")
case .quote(let text):
    print("Quoted: \(text)")
}
```

This syntax improves readability, and maintainability. Also, since the associated values are lazily materialized, performance would be improved if the case is not handled in `switch` or the matched pattern doesn't handle the associated values.

## Detailed Design

### Protocol Definition

`MatchableWithEnumCasePattern` is a compiler-recognized protocol in the standard library that nominal types can conform to declare the type is compatible with case pattern matching. To minimize the scope of this proposal, only `struct`, `final class` and `final actor` can conform to this protocol.

```swift
/// Conforming this enables enum case pattern matching on this type.
public protocol MatchableWithEnumCasePattern {
    /// An associated tag type used for pattern matching.
    /// This must be trivial enum type _without_ any associated values.
    associatedtype EnumCasePatternTag
       
    /// A property representing the tag used to determine the matched case.
    var enumCasePatternTag: EnumCasePatternTag { get }
}
```

### Case declarations

Each case declaration has the name and optionally declares the associated values just like `case` in `enum` types. But unlike `enum`, `case` with associated values is followed by an accessor clause. Like computed properties, this supports any getter strategies, including implicit getter, explicit `get`, `read`, `_read`, and `unsafeAddress`, providing flexibility for various scenarios. However, effect specifiers (`throws` and `async`) are _not_ supported on associated value accessors. 

Example of a case declaration in a `TaggedStringSegment` struct:

```swift
case number(Int) {
    get { Int(text)! }
}
```

Conceptually, the compiler synthesizes a corresponding computed property:

```swift
var $case_associatedValue$number: Int {
    get { Int(text)! }
}
```

This accessor provides the logic for extracting the associated value when the `number` case is matched, and is lazily evaluated only when the tag matches. 

`case` declaration without associated values is allowed:

```swift
case other
```

Such cases match simply based on the `EnumCasePatternTag` enumeration and do not require an associated value accessor.

`case` declaration can have mutiple associated values, just like `enum` cases. In that case, the associated value accessor returns a tuple containing all of the values. For example:

```swift
case circle(center: Point, diameter: Double) { ... }
```

the associated value accessor is conceptually equivalent to:
```swift
var $case_associatedValue$circle: (center: Point, diameter: Double) { ... }
```

This allows destructuring and sub-pattern matching to work in the same way as they do for `enum` cases.

### Case declaration limitations

Protocols can inherit `MatchableWithEnumCasePattern`, but in this proposal, `case` declarations in `extension` of the protocol are not allowed. Also, `case` is not allowed in the `protocol` itself either. That means `case` declaration cannot be a protocol requirement. (Note: This restriction could be lifted in the future - see [Case as protocol requirement](#case-as-protocol-requirement))

All `case` declarations must be written in the same scope where `MatchableWithEnumCasePattern` conformance is declared. In other words, cases cannot be introduced before declaring conformance, nor added retroactively from another extension or modules. (Note: This restriction could be lifted in the future - see [Support "extensible" case names](#support-extensible-case-names))

### `enumCasePatternTag` property

Types conforming to `MatchableWithEnumCasePattern` must implement `enumCasePatternTag` property of the associated type `EnumCasePatternTag` as it is a protocol requirement. It returns the "tag" of the current value. It can be a stored property or a computed property.

### `EnumCasePatternTag` associated type

`EnumCasePatternTag` must be a simple `enum` type without any associated values. The cases in the enum must match exactly with the `case` _names_ in the type. (Note: This restriction could be lifted in the future - see [Support "extensible case names"](#support-extensible-case-names))

```swift
struct Thing: MatchableWithEnumCasePattern {
    enum Kind { // error: missing required enum case 'unknown' as a case name of `Thing`.
                // note: add missing case? (with fix-it adding 'case unknown')
        case foo
        case bar
        case baz // error: extraneous case 'baz' in 'Thing.EnumCasePatternTag' (a.k.a 'Thing.Kind')
    }
    var enumCasePatternTag: Kind
    
    case foo(Int) { ... }
    case bar
    case unknown(arg: String) { ... }
}
```

### Automatic synthesis of `EnumCasePatternTag` enumeration

If a type does not declare `EnumCasePatternTag` associated type, the compiler synthesizes it from all the `case` declarations in the type. For example, for:

```swift
enum ColorElementKind {
case red, green, blue
case cyan, magenta, yellow, key
case hue, saturation, brightness
}

struct RBGColorElement: MatchableWithEnumCasePattern {
    var kind: ColorElementKind
    var value: Double

    case red(Double) { value }
    case green(Double) { value }
    case blue(Double) { value }
}
```

This `enum` is synthesized:

```swift
/* synthesized */
extension RBGColorElement {
    enum EnumCasePatternTag {
        case red, green, blue
    }
}
```

The developer needs to implement `enumCasePatternTag` property.

```swift
extension RBGColorElement {
    var enumCasePatternTag: EnumCasePatternTag {
        return switch self.kind {
        case .red: .red
        case .green: .green
        case .blue: .blue
        default: fatalError("invalid element kind for RBGColorElement")
        }
    }
}
```

### Cases with shared base name

Although [SE-0155 Normalize Enum Case Representation](0155-normalize-enum-case-representation.md) allows multiple `case` declarations shares the same base name, this proposal would not allow them as the match is based on the "tag" enum name.

### Case and other declarations with the same name

This proposal does not allow constructing instances from `case` declarations. As a result, [SE-0280: Enum Cases as Protocol Witnesses](0280-enum-cases-as-protocol-witnesses.md) does not apply here, since `case` declarations do not introduce any callable APIs. (Note: This restriction could be lifted in the future - see [Instance Creation with Case Declarations](#instance-creation-with-case-declarations)).

Because `case` does not conflict with value constructors, users are free to declare other members (functions, or static properties) with the same base name. For example:

```swift
extension TaggedStringSegment {
    static func number(_ value: Int) -> Self {
        return .init(kind: .number, text: "\(value)"[...])
    }
}

if segment == .number(12) {
    // handle
}
```

When resolving patterns, if the subject value is of a `MatchableWithEnumCasePattern` type, the `case` declaration takes precedence over other members with the same name. In other words:

```swift
if case .number(12) = segment {
    // handle
}
```

is _not_ treated as an expression pattern invoking `number(_:)`, even if such a static method exists. Instead, it is resolved as a case-pattern match using the `case number(Int)` declaration.

### Exhaustiveness

A `switch` statement over a type conforming to `MatchableWithEnumCasePattern` can be considered exhaustive if all `case`declarations defined on that type are covered. Exhaustiveness checking for associated values follows the same rules as for `enum` types: the compiler verifies that all possible associated-value patterns are handled.

### Future `case` additions

In library-evolution-enabled modules, `case` declarations are considered *frozen* if the `EnumCasePatternTag` type is marked `@frozen`. In this situation, the compiler can rely on the set of cases being closed and can enforce exhaustiveness at compile time.

If `EnumCasePatternTag` is not marked `@frozen`, `switch` statements over the type must include an `@unknown default:` branch (just like with non-frozen enums) to remain forward-compatible with potential future case additions.

Additionally, following [SE-0487: Nonexhaustive enums](0487-extensible-enums.md), an `@nonexhaustive` could be explicitly added to the `EnumCasePatternTag` to mark it as non-exhaustive.

### Case declaration visibility

The visibility of case declarations are resolved in the same way as enum cases. Just like in `enum`, case declarations don't accept accessibility modifiers such as `private`, but they have the same visibility as the type by default. Cases can still be mark as `@_spi`, again just like enum cases.

### Misc

* This proposal won't synthesize implementation of `CaseIterable` (i.e. `allCases`) even if all the `case` declarations are without associated values.
* Case declarations in non-enum types won't affect the equalablitiy of the type in any way. Automatic `Equatable` implementation synthesis continues to be based on the stored properties only.
* Actors conforming to `MatchableWithEnumCasePattern` can only use enum-case-patterns when both the user-site and and all the associated value accessors are nonisolated.
* If a `switch` has multiple patterns with the same base name, e.g. `switch value { case .foo(0..<10), .foo(100...): ...`, the associated value accessor may or may not be evaluated multiple times.
* Conditionally conforming `MatchableWithEnumCasePattern` is possible. (TBD)

### Grammar/Syntax

`case` declarations for non-enum types basically resembles `case` declaration in enum types, but with an associated value accessor block. Similar to computed properties, `case` with associated value accessor block can only declare single `enum-case-name`. `case` for non-enum types cannot have raw value assignment.

```
non-enum-case-clause → non-enum-case-clause-head enum-case-name tuple-type associated-value-getter-block
non-enum-case-clause → non-enum-case-clause-head name-only-style-enum-case-list
non-enum-case-clause-head -> attributes? 'case' 
associated-value-getter-block → code-block
associated-value-getter-block → '{' getter-clause '}'
name-only-style-enum-case-list → enum-case-name | enum-case-name ',' name-only-style-enum-case-list

struct-member → non-enum-case-clause
class-member → non-enum-case-clause
actor-member → non-enum-case-clause
extension-member → non-enum-case-clause // only when the extending type is struct, class, or actor.
```

`SwiftSynatx`-wise, `EnumCaseDeclSyntax` will have optional `.accessorBlock: AccessorBlockSyntax` between `.rawValue` and `.trailingComma`

```swift
      Child(
        name: "accessorBlock",
        kind: .node(kind: .accessorBlock),
        documentation: """
          Associated value accesor for non-enum `case` declaration with associated values.
          """,
        isOptional: true
      ),
```

## Source Compatibility

This change is additive and does not impact existing code.

## ABI Stability

This proposal adds new functionality without altering existing ABI.

* `MatchableWithEnumCasePattern` protocol introduces additional ABI in the standard library. (Can we make it `@_marker` protocol?)
* `var enumCasePatternTag: Self.EnumCasePatternTag` accessor will be a ABI.
* Associated value accessors will be a ABI. Mangled by the same rule as property accessors but with a certain naming rule.
* Associated value accessor inliable-ness can be controlled by attributes (e.g. `@inlinable`) just like regular variable accessors.
* Matching with `enumCasePatternTag` should be performed with existing enum case pattern matching mechanism.

## Future Directions

### Instance creation with case declarations

Support for initializing types with case-like syntax:

```swift
extension TaggedStringSegment {
    case link(URL) {
        get { URL(string: self.text)! }
        init(initialValue) { self.text = initialValue.description }
    }
}

let segment: TaggedStringSegment = .link(URL(string: "https://swift.org")!)
```

### Expand C/C++ interoperation

Even without ClangImporter support, developers can make imported C/C++ types conform to `MatchableWithEnumCasePattern` protocol by retroactively declaring an extension in Swift.  But by expanding C/C++ interoperation, it'd be possible to annoate C++ code to conform the protocol without writing Swift code. 

```cpp
SWIFT_MATCHABLE_WITH_ENUM_CASE_PATTERN
class JSONValue {
public:
  enum class Kind {
    number, string, boolean, dictionary, array
  };

private:
  union Value {
    std::string stringValue;
    bool booleanValue;
    double numberValue;
    std::vector<JSONValue> arrayValue;
    std::map<std::string, JSONValue> dictionaryValue;
  };
  Kind kind;
  Value value;

public:
  SWIFT_ENUM_CASE_PATTERN_TAG
  Kind getKind() const {
    return kind;
  }
  
  SWIFT_ENUM_CASE_ASSOCIATED_VALUE("string")
  std::string getString() const {
    assert(kind == Kind::string);
    return value.stringValue;
  }

  // ...
};
```

By annotating C++ declarations, C++ interpolation mechanism can synthesize the protocol conformance and the 'case' declarations.

### Support "extensible" case names

Allow clients to add new cases outside the original module. This would require sacrificing compile-time exhaustiveness checking but would remove several current restrictions:

* `EnumCasePatternTag` would no longer be required to be a simple `enum` without associated values.

* The set of cases would no longer need to exactly match the `EnumCasePatternTag` values.

* `case` declarations could be introduced in other modules, not just in the scope where `MatchableWithEnumCasePattern` conformance is declared.

This would still give clients the full benefit of enum-case-patterns - including lazy associated value evaluation, sub-pattern matching, value binding, and destructuring of associated values - even for cases declared outside the original module.

For example:

```swift
public struct Value: MatchableWithEnumCasePattern {
    public struct Kind: Equatable {
        public var name: String
        public init(name: String) { self.name = name }
    }

    public let kind: Kind
    public let data: String

    public init(kind: Kind, data: String) {
      self.kind = kind
      self.data = data
    }
  
    var enumCasePatternTag: Kind { self.kind }
}
```
And a client module could extend it with new cases:
```swift
extension Value.Kind {
    static var url: Self = .init(name: "url")
}

extension Value {
    init(url: URL) {
        self.init(kind: .url, data: url.absoluteString)
    }

    case url(URL) {
        URL(string: data)!
    }
}
```

Which allows ergonomic pattern matching:

```swift
switch value {
case .url(let url): // full sub-pattern matching and extraction still work
    handle(url)
default:
    handleOther(value)
}
```

Tag matching would be performed using the `static var` name and the regular `~=` operator. Conceptually, the example above could be translated as:

```swift
var $tag = value.enumCasePatternTag
if Value.EnumCasePatternTag.url ~= $tag,
   case (let url) = value.$case_associatedValue$url {
    handle(url)
} else {
    handleOther(value)
}
```

Because the set of cases is open-ended, the compiler cannot guarantee exhaustiveness, so all `switch` statements on such types must include a `default:` case.

### Case as protocol requirement

Allow `case` declarations in `protocol`. Types conforming such protocol must have the `case` implementation.

### Inherit cases in base types

Allow `case` declarations in protocol extensions and non-final class/actor.

## Alternatives Considered

### Associated type accessor as the matching function

Another design considered was to use the associated value accessors as the matching functions without needing the _tag_ enum type. In such design, the accessor's result type would be the `Optioanl` of the accociated type. For example:

```swift
struct Thing: MatchableWithCaseEnumPattern {
    var text: String

    case httpURL(URL) { /* -> Optional<URL> */
        return if text.hasPrefix("http://"), let result = URL(string: text) {
            result
        } else {
            nil
        }
    }
  
    case quotedString(String) { /* -> Optional<String> */
        return if text.hasPrefix("'"), text.hasSuffix("'") {
             String(text.dropFirst().dropLast())
        } else {
             nil
        }
    }
}

switch thing {
case .httpURL(let url):
    print("URL: \(url)")
case .quotedString(let str):
    print("value: \(str)")
}
```

In this design, switch above behaves like

```swift
if case (let url)? = thing.$case_associatedValue$httpURL {
    print("URL: \(url)")
} else if case (let str)? = thing.$case_associatedValue$quotedString {
    print("value: \(str)")
}
```

Although this model might be simpler, it can't be performant compared to the proposed _tag_ enum design because the matchings are ultimately a liner search.

> Or not? `switch` over these cases may be optimized to a look table 
> 
> ```swift
> case red(Double) { self.kind == .red ? self.value : nil }
> case green(Double) { self.kind == .green ? self.value : nil }
> case blue(Double) { self.kind == .blue ? self.value : nil }
> ```

But considering other things like exhaustive checks, or lazy associated value computations _only for the pattern with the associated value clause_, I don't think this is a good idea.

### Do nothing, just use expression patterns for matching

By declaring `~=` functions, we can customize the matching behavior, but this approach lacks the ability to use value-binding-pattern.

### Introducing another pattern to support associated value binding

There was a related effort in pre-Swift 1.0 era (`NominalTypePattern` a.k.a. type destructing pattern), but it was abandoned and removed in https://github.com/swiftlang/swift/commit/62e4811dacf4fcd1082c5e58a75b933adf6153f0 

## Acknowledgments

Special thanks to the Swift community for discussions.
