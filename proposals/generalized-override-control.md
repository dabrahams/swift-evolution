# Pitch: Generalizing Override Control

**Authors:** Dave Abrahams (@dabrahams) and Doug Gregor (@dgregor)

## Abstract

We propose: 

- a way to prevent the unintentional creation of (notional) overload sets that
  include functions inherited from protocols and base classes.
- a way to diagnose unintentional failure to implement a protocol requirement.
- a way to disambiguate uses of identically-named functions.

## Motivation

This proposal solves four problems:

1. It's very common to intentionally define a same-named method with a more
   specific signature than one inherited from a protocol:
   
    ```swift
    protocol P {}
    extension P {
        func f() -> Any { return 0 }
    }
    struct X : P {
        func f() -> Int { return 1 }
    }
    ```
    
    Most of this time, this overload works as expected, but there are occasional
    surprises:
    
    ```swift
    let a = X().f()
    print(a)              // 1
    print(X().f())        // 0 (Surprise!)
    ```
    
    In this case, the fact that `print` specifically takes arguments of type
    `Any` causes overload resolution to select the better matching signature
    defined in `P`.  It would have been better to hide the `Any`-returning
    overload that was inherited from `P`, but **today we have no way to do
    that.** 
    
    There are numerous examples of such overloads in the standard library,
    notably the `SubSequence`-returning methods on `Sequence` and `Collection`
    and the intentionally-overloaded algorithms defined on `LazyCollection`.
    The latest example of a problem caused by failure to control this kind of
    overload has been
    [cited](https://forums.swift.org/t/se-0231-optional-iteration/16737/298?u=dabrahams)
    as the source of performance and correctness traps.
    
2. It's very common to write a method with the intention of implementing a
   protocol requirement, but to fail to do so because the requirement has a
   default implementation and the intended implementation has a slightly-wrong
   signature.
   
    ```swift
    protocol P {
        func f() -> Int
        associatedtype A = String
        func g(_: A?) -> Int
    }

    extension P {
        func f() -> Int { return 0 }
        func g(_: A?) -> Int { return 2 }

        /// Call the f requirement implementation
        func f1() -> Int { return f() }

        /// Call the g requirement implementation
        func g1() -> Int { return g(nil) }
    }

    struct X : P {
        typealias A = Int

        // Intended implementations of f and g requirements
        func f(option: Bool = true) -> Int { return 1 }
        func g(_: String?) -> Int { return 3 }
    }

    print(X().f(), X().g(nil)) // 1 3
    print(X().f1(), X().g1())  // 0 2 (Surprise!)
    ```
    
    This long-standing problem has been the subject of
    [much](https://forums.swift.org/t/override-like-keyword-for-default-implementations)
    [discussion](https://forums.swift.org/t/mark-protocol-methods-with-their-protocol)
    in the swift community and the target of
    [several](https://forums.swift.org/t/pitch-requiring-proactive-overrides-for-default-protocol-implementations)
    [pitches](https://forums.swift.org/t/pitch-requiring-special-keyword-to-mark-protocol-implementation-methods)
    intended to address it.  Standard library developers have made this mistake
    and shipped the results.

3. There are many cases where the intended method is accessible but can't be
   called or referenced, either due to ambiguity or the existence of a
   more-specific overload.
   
    ```swift
    protocol P {
        func g() -> Int
    }

    protocol Q {}

    extension P {
        func f() -> Int { return 0 }
        func g() -> Int { return 1 }
    }

    extension Q {
        func f() -> Int { return 2 }
    }

    struct X : P, Q {
        func g() -> Int {
            // Can't forward to the default implementation here
            return 3
        }
    }

    let a = X()
    print(a.f()) // error: ambiguous use of 'f'
    ```

4. When two protocols, or a protocol and a class, define customization points
   with identical signatures, there's no way to specify which one is being
   implemented by a model or subclass.

## Solution

* We extend name syntax to allow the specification of a module in which to look
  for an extension.  Specifically, if module `A` declares the following extension:
  
  ```swift
  extension X.Y {
      public var f() -> Int { return 0 }
  }
  ```
  
  the method it declares can be referred to from any module as `A::X.Y.f`.
  
* We extend the `override` keyword:
    - it will be allowed on implementations of protocol requirements (probably
      except for associated types, at least initially).
    - it will accept an optional argument that names a class or protocol having
      the customization point being overridden.
  
    ```swift
    protocol P1 { func f() }
    protocol P2 { func f() }
    class Base { func f() }
    
    struct Model : P1, P2 {
        override(P1) func f() {} // implements P1's f()
        override(P2) func f() {} // implements P2's f()
    }
    
    class Derived : Base, P1 {
        override(Base) func f() {} // Implements Base's f()
        override(P1) func f() {}   // Implements P1's f()
    }
    ```
    
* We introduce two new annotations, in a syntactic family with `override`: 
    - `new` indicates a distinct method that *hides* methods, declared in
      less-specific contexts, that have the **same base name.**  Note: there is
      precedent in C# for using `new` this way.
    - `new(overload)` indicates a distinct method that *overloads* methods,
      declared in less-specific contexts, that have the **same base name.**

* We make it an error to declare an un-annotated method's base name matches that
  of a method defined in a less-specific or remote context.  This includes
  `subscript` but excludes `init` and `deinit`.  In the code that follows, all
  instances of `override` and `new` would be required.
  
    ```swift
    protocol P {
        func defaultedRequirement()
        subscript(_: Int) -> Int
    }
    
    // A protocol extension in the module where the protocol is declared
    // has same specificity as the protocol does.
    extension P {
        // This declaration implements (a.k.a. overrides) P's requirement.
        func defaultedRequirement() {}
        
        // No base name match; this declaration introduces a new function.
        func extension() {}
        
        // This declaration *overloads* the subscript declared in P.  There's no
        // way to hide a P requirement in an extension of P.
        subscript(_: String) -> String
    }
    
    struct X : P {
        // This definition implements P's requirement.
        override subscript(x: Int) -> Int { return x }
        
        // This declaration makes hides P's requirement of the same base name
        // from access on an instance of static type X.
        new func defaultedRequirement(label: Int = 0) {}
        
        // This declaration adds an overload to the set of subscripts visible
        // to code handling an instance of static type X.
        new(overload) subscript(x: Double) -> Double { return x }
    }
    ```
    
* Contexts are ordered as follows:

    - A subclass is more specific than its superclass
    - A concrete type is more specific than the protocols it conforms to
    - A protocol is more specific than its inherited protocols
    - The current module is more specific than imported modules when type
      context is equivalent

## Open Questions

* Can (and should) we order protocol and generic type extensions based on
  specificity of constraints?
* Shall we loosen the annotation requirements for requirements in conformance
  declarations?  For example:
  
    ```swift
    protocol P {
        func requirement()
    }
    
    extension X : P {
        func requirement() {}
    }
    ```

    It could reasonably be assumed that, if `P` has a `requirement` requirement,
    and the extension declares conformance to `P`, that the method above is
    intended to override it.  If we do assume that, what are the rules?  Suppose
    the extension declares conformance to two protocols?  Can we make the same
    assumption in the main declaration of `X`, outside an extension?
* What is the migration story for existing code?

## Possible Future directions

* Allow an `override` to add parameters, as long as they have defaulted argument
  values, have contravariant argument and/or covariant return types.
* Allow/require `override` annotations on associated types.
