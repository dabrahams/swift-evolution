# Pitch: Generalizing Override Control for Swift 6

**Authors:** Dave Abrahams (@dabrahams) and Doug Gregor (@dgregor)

## Abstract

We propose: 

- a way to prevent the unintentional creation of (notional) overload sets that
  include functions inherited from protocols and base classes.
- a way to diagnose unintentional failure to implement a protocol requirement.
- a way to disambiguate identically-named functions.

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
    [cited](https://forums.swift.org/t/se-0231-optional-iteration-wrapup/16981/7?u=dabrahams)
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

* We extend member qualification syntax to

* We introduce two new annotations in the family with `override`: `new` and
  `new(overload)` to indicate a new, distinct method that either hides or
  overloads another with the same base name, respectively.

* A diagnostic will be emitted when an un-annotated method's base name (not
  including argument labels) matches that of a method defined in a less-specific
  or remote context.  This includes `subscript` but excludes `init` and
  `deinit`:
  
    **Module A**
    ```swift
    protocol P {
        func requirement()
        func defaultedRequirement()
        subscript(_: Int) -> Int
    }
    
    extension P {
        func defaultedRequirement() {} // OK, same specificity as protocol
        func extension() {}
        subscript(_: String) -> String // OK, same specificity as protocol
    }
    
    struct X : P {
        func requirement() {}                        // Diagnostic: annotate 
        func defaultedRequirement(label: Int = 0) {} // Diagnostic: annotate 
    }
    ```
    
    **Module B**
    ```swift
    import A
    extension X {
        func requirement() -> Int? { return nil }   // Diagnostic: annotate
    }
    ```
    
* Contexts are ordered as follows:

    - A subclass is more specific than its superclass
    - A concrete type is more specific than the protocols it conforms to
    - A protocol is more specific than its inherited protocols
    - The current module is more specific than imported modules when type
      context is equivalent

## Open Questions

* Can we ever make this diagnostic an error?
* Can we order extensions based on specificity of constraints?

