# Protocol-oriented integers

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-updated-integers.md)
* Author(s): [Max Moiseev](https://github.com/moiseev)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This proposal:

* Cleans up Swift's integer protocols, making them useful for
  generic programming
* Makes bit-shifting more general and usable
* Reduces the proliferation of arithmetic operator overloads
* Eliminates the hard-coded and artificial “maximum-width integer
  type” `IntMax`.
* Enables mixed-type comparisons and shifting

The problem of generalized mixed-type arithmetic—a much more
complicated topic—should be addressed in the future but is an explicit
*non-goal* of this proposal.

## Motivation

Although generalized algorithms and data types that can work for any
integer are common, Swift's integer protocols were never properly
designed for that purpose, and have
[proven frustrating](http://blog.krzyzanowskim.com/2015/03/01/swift_madness_of_generic_integer/)
to those who have tried.

The `Integer` and `IntegerArithmetic` protocols are not of much use,
as they define all binary operations on pairs of arguments with the
same concrete type, thus making generic programming impossible at the
same slowing down the type checker, due to a large number of
overloads.

Another annoying problem, that gets mentioned often, is inability to use
integers of different types in comparison and bit-shift operations. For example,
the following snippet won't compile:

```Swift
let x: Int8 = 42
let y = 1
x << y    // error: binary operator '<<' cannot be applied to operands of type 'Int8' and 'Int'
x > y     // error: binary operator '>' cannot be applied to operands of type 'Int8' and 'Int'
```

Moreover, current design predates many of the improvements that came in Swift 2,
and haven't been revised since then.

Here is the basic layout of integer protocols as of Swift 2:

```
         +---------------------+   +---------------------+
         |  IntegerArithmetic  |   |  BitwiseOperations  |
         +------------------+--+   ++--------------------+
                            ^       ^
                            |       |
+---------------------+   +-+-------+--+   +----------------+
|  RandomAccessIndex  |   |  _Integer  |   |  SignedNumber  |
+----------------+----+   +---+--------+   +------+---------+
                 ^            ^    ^---------+    ^
                 |            |              |    |
                 |        +---+-------+  +---+----+---------+
                 +--------+  Integer  |  |  _SignedInteger  |
                          ++----------+  +-----+------------+
                           ^          ^        |
                           |          |        |
         +-----------------+-+       ++--------+-------+
         |  UnsignedInteger  |       |  SignedInteger  |
         +-------------------+       +-----------------+
```

## Proposed solution

We propose a new protocol hierarchy that does not have above mentioned problems
and allows simpler extension.

~~~~
                +--------------+  +----------+
        +------>+  Arithmetic  |  |Comparable|
        |       +-------------++  +---+------+
        |                     ^       ^
+-------+------------+        |       |
|  SignedArithmetic  |      +-+-------+-+
+------+-------------+      |  Integer  |
       ^                    +----+------+
       |         +-----------^   ^     ^-----------+
       |         |               |                 |
+------+---------++    +---------+-----------+    ++------------------+
|  SignedInteger  |    |  FixedWidthInteger  |    |  UnsignedInteger  |
+---------------+-+    +-+-----------------+-+    ++------------------+
                ^        ^                 ^       ^
                |        |                 |       |
                |        |                 |       |
               ++--------+-+             +-+-------+-+
               |Int family |             |UInt family|
               +-----------+             +-----------+
~~~~

## Detailed design

Prototype implementation can be found at
https://github.com/apple/swift/blob/master/test/Prototypes/Integers.swift.gyb


### Proposed protocols

#### `Arithmetic` protocol

Declares methods backing binary arithmetic operators, such as  `+`, `-` and
`*`; and their mutating counterparts. These methods operate on arguments of the
same type.

Both mutating and non-mutating operations are declared in the protocol, but
only the mutating ones are required. Should conforming type omit non-mutating
implementations, they will be provided by a protocol extension.  Implementation
in that case will copy `self`, perform a mutating operation on it and return
the resulting value.

```Swift
public protocol Arithmetic : Equatable, IntegerLiteralConvertible {
  init()

  @warn_unused_result
  func adding( rhs: Self) -> Self
  @warn_unused_result
  func subtracting( rhs: Self) -> Self
  @warn_unused_result
  func multiplied(by rhs: Self) -> Self
  @warn_unused_result
  func divided(by rhs: Self) -> Self


  mutating func add( rhs: Self)
  mutating func subtract( rhs: Self)
  mutating func multiply(by rhs: Self)
  mutating func divide(by rhs: Self)
}
```

#### `SignedArithmetic` protocol

Will only be conformed to by signed numbers, otherwise it would be posible to
negate an unsigned value.
 
The only method of this protocol has the default implementation in an
extension, that uses a parameterless initializer and subtraction.

```Swift
public protocol SignedArithmetic : Arithmetic {
  func negate() -> Self
}
```

#### `Integer` protocol

This is a base protocol for all the integer types that are available in the
standard library, and besides should be suffecient to implement arbitrary
precision integer types.

`isEqual(to:)` and `isLess(than:)` methods are the ones responsible for
`Equatable` and `Comparable` protocol conformances. In a way similar to how
arithmetic operations are dispatched in `Arithmetic`, `==` and `<` operators
for homogeneous comparisons have default implementations that call
`isEqual(to:)` and `isLess(than:)` respectively.

This protocol adds 3 new initializers to the parameterless one, inherited
from `Arithmetic`. These initializers allow to construct values of type from
instances of any other type, conforming to `Integer`, using different
strategies:
  - Perform checks whether the value is representable in `Self`
  - Try to represent the value without bounds checking
  - Substitute values beyond `Self` bounds with maximum/minimum value of
    `Self` respectively

A type that can hold absolute values of all the possible values of `Self`.
Concrete types do not have to provide a typealias for it as it can be inferred
from an `absoluteValue` property. This property (and type) can be useful in
operations that are simpler to implement in terms of (potentially larger)
unsigned values, for example, printing a value of an integer (it's just adding a
'-' character in front of an absolute value).

Please note, that `absoluteValue` property should not in general be used as a
substitute for an `abs` free function, that returns a value of the same type.

```Swift
public protocol Integer:
  Comparable, Arithmetic,
  IntegerLiteralConvertible, CustomStringConvertible {

  associatedtype AbsoluteValue : Integer // this is not an actual code

  static var isSigned: Bool { get }

  var absoluteValue: AbsoluteValue { get }

  @warn_unused_result
  func isEqual(to rhs: Self) -> Bool
  @warn_unused_result
  func isLess(than rhs: Self) -> Bool

  /// Creates an instance of `Self` from the value of any other `Integer`,
  /// trapping if value of `source` cannot be represented by `Self`.
  init<T : Integer>(_ source: T)

  /// Creates in instance of `Self` from the value of any other `Integer`,
  /// using bits from `source` up to the available width of `Self`, if `T` is
  /// wider than `Self`, and filling extra bits on the left with sign bit
  /// value otherwise.
  init<T : Integer>(extendingOrTruncating source: T)

  /// Creates in instance of `Self` from the value of any other `Integer`,
  /// resulting in `Self.min` (or `Self.max`) if the value cannot be represented
  /// by `Self`, and exact value of `source` otherwise.
  init<T : Integer>(clamping source: T)

  /// Return n-th word (counting from the right) in the underlying
  /// representation of `self`.
  @warn_unused_result
  func nthWord(n: Int) -> UInt

  /// A number of bits in current representation of `self`
  /// Will be constant for fixed-width integer types.
  var bitWidth : Int { get }

  /// If `self` is negative, returns the index of the least significant bit of
  /// our representation such that all more-significant bits are 1.
  /// Has the value -1 if `self` is 0.
  /// Has the value equal to `bitWidth - 1` for fixed width integers.
  var signBitIndex: Int { get }

  /// In order for floating point numers to conform to `Arithmetic`,
  /// `remainder` (and `%`) are defined here.
  @warn_unused_result
  func remainder(dividingBy rhs: Self) -> Self
  mutating func formRemainder(dividingBy rhs: Self)

  /// An extension point to provide an efficient implementation of `divRem`
  /// operation, producing a pair of quotient and remainder of division of
  /// `self` by `rhs`.
  /// Default implementation simply invokes `divided(by:)` and
  /// `remainder(dividingBy:)`, which in case of built-in types will be fused
  /// into a single instruction by the compiler.
  func quotientAndRemainder(dividingBy rhs: Self) -> (Self, Self)
}
```

#### `FixedWidthInteger` protocol

A protocol for all the fixed width integer types. Main addition to the
`Integer` protocol is binary bitwise operations and bit shifts.
 
`WithOverflow` family of methods is used in default implementations of
mutating arithmetic methods (from `Arithmetic` protocol), provided by a protocol
extension. Having these methods allows to provide both safe (trapping) and
unsafe implementation of arithmetic operations without duplicating code.
 
Bitwise binary and shift operators are implemented the same way as arithmetic
operations: free function dispatches a call to a corresponding protocol method.
 
`doubleWidthMultiply` method is a necessary building block to implement
support for integer types of a greater width and as a consequence, arbitrary
precision integers.

```Swift
public protocol FixedWidthInteger : Integer {
  static var bitWidth : Int { get }

  static var max: Self { get }
  static var min: Self { get }

  @warn_unused_result
  func addingWithOverflow(
     rhs: Self
  ) -> (partialValue: Self, overflow: ArithmeticOverflow)

  @warn_unused_result
  func subtractingWithOverflow(
     rhs: Self
  ) -> (partialValue: Self, overflow: ArithmeticOverflow)

  @warn_unused_result
  func multipliedWithOverflow(
    by rhs: Self
  ) -> (partialValue: Self, overflow: ArithmeticOverflow)

  @warn_unused_result
  func dividedWithOverflow(
    by rhs: Self
  ) -> (partialValue: Self, overflow: ArithmeticOverflow)

  @warn_unused_result
  func remainderWithOverflow(
    dividingBy rhs: Self
  ) -> (partialValue: Self, overflow: ArithmeticOverflow)

  @warn_unused_result
  func and(rhs: Self) -> Self
  @warn_unused_result
  func or(rhs: Self) -> Self
  @warn_unused_result
  func xor(rhs: Self) -> Self
  @warn_unused_result
  func maskingShiftRight(rhs: Self) -> Self
  @warn_unused_result
  func maskingShiftLeft(rhs: Self) -> Self

  @warn_unused_result
  func doubleWidthMultiply(other: Self) -> (high: Self, low: AbsoluteValue)

  init(_truncatingBits bits: UInt)
}
```

#### Auxilliary protocols

```Swift
public protocol UnsignedInteger : Integer {
  associatedtype AbsoluteValue : Integer
}
public protocol SignedInteger : Integer, SignedArithmetic {
  associatedtype AbsoluteValue : Integer
}
```


### Operators

#### Arithmetic

```Swift
public func + <T: Arithmetic>(lhs: T, rhs: T) -> T
public func += <T: Arithmetic>(lhs: inout T, rhs: T)
public func - <T: Arithmetic>(lhs: T, rhs: T) -> T
public func -= <T: Arithmetic>(lhs: inout T, rhs: T)
public func * <T: Arithmetic>(lhs: T, rhs: T) -> T
public func *= <T: Arithmetic>(lhs: inout T, rhs: T)
public func / <T: Arithmetic>(lhs: T, rhs: T) -> T
public func /= <T: Arithmetic>(lhs: inout T, rhs: T)
public func % <T: Arithmetic>(lhs: T, rhs: T) -> T
public func %= <T: Arithmetic>(lhs: inout T, rhs: T)
```

##### Implementation example

_Only homogeneous arithmetic operations are supported._

Operator `+` is defined as a free function on arguments of the same type,
conforming to `Arithmetic` protocol, and is implemented in terms of copying the
left operand and calling `add` on it, returning the result.  `FixedWidthInteger`
provides a default implementation for `add`, that delegates work to
`addingWithOverflow`, which is implemented efficiently by every concrete type
using intrinsics.


#### Masking arithmetics

```Swift
public func &* <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &- <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &+ <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
```

##### Implementation

These operators call `WithOverflow` family of methods from `FixedWidthInteger`
and simply return the `partialValue` part, ignoring the possible overflow.


#### Homogeneous comparison

```Swift
public func == <T : Integer>(lhs:T, rhs: T) -> Bool
public func != <T : Integer>(lhs:T, rhs: T) -> Bool
public func < <T : Integer>(lhs: T, rhs: T) -> Bool
public func > <T : Integer>(lhs: T, rhs: T) -> Bool
public func >= <T : Integer>(lhs: T, rhs: T) -> Bool
public func <= <T : Integer>(lhs: T, rhs: T) -> Bool
```

Implementation is similar to the homogeneous arithmetic operators above.


#### Heterogeneous comparison

```Swift
public func == <T : Integer, U : Integer>(lhs:T, rhs: U) -> Bool
public func != <T : Integer, U : Integer>(lhs:T, rhs: U) -> Bool
public func < <T : Integer, U : Integer>(lhs: T, rhs: U) -> Bool
public func > <T : Integer, U : Integer>(lhs: T, rhs: U) -> Bool
public func >= <T : Integer, U : Integer>(lhs: T, rhs: U) -> Bool
  public func <= <T : Integer, U : Integer>(lhs: T, rhs: U) -> Bool
```

##### Implementation example

The overloaded version of `==` operator accepts two arguments of different
types both conforming to `Integer` protocol. One of the arguments then
gets transformed into the value of the type of another using an
`extendingOrTruncating` initializer, introduced by `Integer`, and a
homogeneous version of `==` is called, which, as in the example above,
delegates all the work to `isEqual(to:)` method, implemented by every concrete
type using runtime intrinsics.


#### Shifts

```Swift
public func << <T: FixedWidthInteger, U: Integer>(lhs: T, rhs: U) -> T
public func << <T: FixedWidthInteger>(lhs: T, rhs: Word) -> T
public func <<= <T: FixedWidthInteger, U: Integer>(lhs: inout T, rhs: U)
public func <<= <T: FixedWidthInteger>(lhs: inout T, rhs: T)

public func >> <T: FixedWidthInteger, U: Integer>(lhs: T, rhs: U) -> T
public func >> <T: FixedWidthInteger>(lhs: T, rhs: Word) -> T
public func >>= <T: FixedWidthInteger>(lhs: inout T, rhs: T)
public func >>= <T: FixedWidthInteger, U: Integer>(lhs: inout T, rhs: U)

public func &<< <T: FixedWidthInteger, U: Integer>(lhs: T, rhs: U) -> T
public func &<< <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &<<= <T: FixedWidthInteger, U: Integer>(lhs: inout T, rhs: U)
public func &<<= <T: FixedWidthInteger>(lhs: inout T, rhs: T)

public func &>> <T: FixedWidthInteger, U: Integer>(lhs: T, rhs: U) -> T
public func &>> <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &>>= <T: FixedWidthInteger, U: Integer>(lhs: inout T, rhs: U)
public func &>>= <T: FixedWidthInteger>(lhs: inout T, rhs: T)
```

##### Implementation example (mixed-type left shift)

Situation here is almost similar to the one with heterogeneous comparison.  The
only difference is that it is hard to define what a left shift would mean to an
infinitely large integer, therefore we only allow shifts, where left operand
conforms to `FixedWidthInteger`. Right operand, however, can be an arbitrary
`Integer`. Aside from that, implementation is quite similar to that of binary
arithmetic operations: generic function delegates task to a non-generic one,
implemented efficiently on a concrete type.


#### Bitwise operations

```Swift
public func | <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func |= <T: FixedWidthInteger>(lhs: inout T, rhs: T)
public func & <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func &= <T: FixedWidthInteger>(lhs: inout T, rhs: T)
public func ^ <T: FixedWidthInteger>(lhs: T, rhs: T) -> T
public func ^= <T: FixedWidthInteger>(lhs: inout T, rhs: T)
```


## Impact on existing code

Existing code that does not implement its own integer types (or rely on
existing protocol hierarchy in any other way) should not be affected. It will
be slightly wordier due to all the type conversions that are no longer required,
but will continue to work. Migration is possible but not strictly necessary.


## Non-goals

This proposal:

- *DOES NOT* solve the integer promotion problem, which would allow mixed-type
  arithmetic. However, we believe that it is an important step in the right
  direction.

- *DOES NOT* propose an introduction of a `BigInt` type, but, we believe, allows it
  to be implemented in the future.
