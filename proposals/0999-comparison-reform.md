# Comparison Reform

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Robert Widmann](https://github.com/codafi), [Jaden Geller](https://github.com/jadengeller), [Harlan Haskins](https://github.com/harlanhaskins), [Alexis Beingessner](https://github.com/Gankro)
* Status: **Awaiting review**
* Review manager: TBD



## Introduction

This proposal fixes several shortcomings with the current Comparable system by introducing a new ternary-valued `compared(to: Self) -> SortOrder` method. It also makes FloatingPoint comparison context sensitive, so that its Comparable conformance provides a proper total ordering. These are technically two distinct proposals that can be done independently, but are included together for historical reasons.




## Motivation

The standard comparison operators have an intuitive meaning to programmers. Swift encourages encoding that in an implementation of `Comparable` that respects the rules of a [total order](https://en.wikipedia.org/wiki/Total_order). The standard library takes advantage of these rules to provide consistent implementations for sorting and searching generic collections of `Comparable` types.  

Not all types behave so well in this framework, unfortunately. There are cases where the semantics of a total order cannot apply and still maintain the traditional definition of “comparison” over these types.  Take, for example, sorting an array of `Float` s.  Today, `Float`'s instance of `Comparable` follows IEEE-754 and returns `false` for all comparisons of `NaN`. In order to sort this array, `NaN` s are considered outside the domain of `<`, and the order of a sorted array containing them is unspecified. Similarly, a Dictionary keyed off floats can leak entries and memory.

In addition, generic algorithms in the Swift Standard Library that make use of the current `Comparable` protocol may have to make twice as many comparisons to request the ordering of values with respect to each other than they should. Having a central operation to return information about the ordering of values once should provide a speedup for these operations.

Comparable is also over-engineered in the customization points it provides: to our knowledge, there's no good reason to ever override `>=`, `>`, or `<=`. Each customization point bloats vtables and mandates additional dynamic dispatch.

Finally, comparison operators don't "generalize" well. There's no clean way to add a third or fourth argument to `<` to ask for non-default semantics. An example where this would be desirable would be specifying the locale or case-sensitivity when comparing Strings.

In the interest of cleaning up the semantics of `Comparable` types of all shapes and sizes and their uses in the Swift Standard Library, this proposal rearranges the requirements of the `Comparable` and `Equatable` protocols in a backwards-compatible way.






## Proposed solution



### Comparable

A new `enum SortOrder { case before, same, after }` will be introduced.

Comparable will be changed to have a new ternary comparison method: `compared(to: Self) -> SortOrder`. `x.compared(to: y)` specifies where to place x relative to y. So if it yields `.before`, then x comes before y. This will be considered the new "main" dispatch point of Comparable that implementors should provide.

Most code will continue to use `<` or `==`, as it will be optimal for their purposes. However code that needs to make a three-way branch on comparison can use the potentially more efficient `compared`. Note that `compared` is only expected to be more efficient in this specific case. If a two-way branch is all that's being done, `<` will be more efficient in many cases (if only because it's easier for the optimizer).

For backwards compatibility reasons, `compared` will have a default implementation defined in terms of `<`, but to enable only using `compared`, `<` and `==` will also have default implementations in terms of `compared`.

The compiler will verify that either `compared` or `<` and `==` are provided by every type that claims to conform to Comparable. This will be done in some unspecified way unavailable outside the standard library (it can be made available to in the future, but that's an unnecessary distraction for this proposal).

Types that wish to provide comparison "variants" can do so naturally by adding `compared` methods with additional arguments. e.g. `String.compared(to: Self, in: Locale) -> SortOrder`. These have no language-level connection to Comparable, but are still syntactically connected, implying the same total order semantics. This makes them easier to discover, learn, and migrate to.

To reduce bloat, the operators `<=`, `>=`, and `>` will be removed from the set of requirements that the Comparable protocol declares. These operators will however continue to exist with the current default implementations.




### Floating Point

No changes will be made to the FloatingPoint protocol itself. Instead, new extensions will be added to it to change the behaviour of comparison.

The new behaviour centers around the fact that `compared(to: Self) -> SortOrder` will provide a total ordering that's consistent with Level 2 in the IEEE 754 (2008) spec. This is mostly the same as the standard (Level 1) IEEE ordering, except:

  * `-0 < +0`
  * `NaN == NaN`
  * `NaN > +Inf` (an arbitrary choice, NaN can be placed anywhere in the number line)

Level 2's distinguishing of `-0` and `+0` is a bit strange, but is consistent with Equatable's Substitutability requirement. `-0` and `+0` have different behaviours: `1/-0 = -Inf` while `1/+0 = +Inf`. The main problem this can lead to is that a keyed collection may have two "0" entries. In practice this probably won't be a problem because it's fairly difficult for the same algorithm to produce both `-0` and `+0`. Any algorithm that does is also probably concerned with the fact that `1.0E-128` and `2.0E-128` are considered distinct values.

Note: IEEE specifies several other potential total orderings: level 3, level 4, and the totalOrder predicate. For our purposes, these orderings are too aggressive in distinguishing values that are semantically equivalent in Swift. For most cases, the relevant issue is that they distinguish different encodings of NaN. For more exotic encodings that don't guarantee normalization, these predicates also consider `10.0e0 < 1.0e1` to be true. An example where this can occur is *IEEE-754 decimal coded floating point*, which FloatingPoint is intended to support.

We will then make the comparison operators (`<`, `<=`, `==`, `!=`, `>=`, `>`) dispatch to one of `compared(to:)` or FloatingPoint's IEEE comparison methods (`isLess`, `isEqual`, `isLessThanOrEqualTo`) based on the context.

* If the context knows the type is FloatingPoint, then level 1 ordering will be used.
* If the context only knows the type is Comparable or Equatable, then level 2 ordering will be used.

This results in code that is explicitly designed to work with FloatingPoint types getting the expected IEEE behaviour, while code that is only designed to work with Comparable types (e.g. sort and Dictionary) gets more reasonable total ordering behaviour.

To clarify: Dictionary and sort won't somehow detect that they're being used with FloatingPoint types and use level 1 comparisons. Instead they will unconditional use level 2 behaviour. For example:

```
let nan = 0.0/0.0

func printEqual<T: Equatable>(_ x: T, _ y: T) {
  print(x == y)
}

func printEqualFloats<T: FloatingPoint>(_ x: T, _ y: T) {
  print(x == y)
}

print(nan == nan)          // false, (concrete)
printEqual(nan, nan)       // true,  (generic Equatable but not FloatingPoint)
printEqualFloats(nan, nan) // false, (generic FloatingPoint)
```

If one wishes to have a method that works with all Equatable/Comparable types, but uses level 1 semantics for FloatingPoint types, then they can simply provide two identical implementations that differ only in the bounds:

```
let nan = 0.0/0.0

func printEqual<T: Equatable>(_ x: T, _ y: T) {
  print(x == y)
}

func printEqual<T: FloatingPoint>(_ x: T, _ y: T) {
  print(x == y)
}

printEqual(0, 0)           // true (integers use `<T: Equatable>` overload)
printEqual(nan, nan)       // false (floats use `<T: FloatingPoint>` overload)
```





### Standard Library

Functions that take an ordering predicate `(Self, Self) -> Bool` will now also have an overload for `(Self, Self) -> Ordering`. 

Types that conform to Comparable should be audited for places where implementing or using Comparable would be a win. This update can be done incrementally, as the only potential impact should be performance. As an example, a default implementation of `compared(to:)` for Array will likely be suboptimal, performing two linear scans to determine the result in the worst-case. (See the default implementation provided in the detailed design.)

Some free functions will have `<T: FloatingPoint>` overloads to better align with IEEE-754 semantics. This will be addressed in a follow-up proposal. (example: `min` and `max`)

SortOrder should be bridged to Foundation's ComparisonResult type.



## Detailed Design

The protocols will be changed as follows:

```
public enum SortOrder: Equatable {
    case before, same, after
}

public protocol Comparable: Equatable {
  func compared(to other: Self) -> SortOrder

  static func < (lhs: Self, rhs: Self) -> Bool

  static func == (lhs: Self, rhs: Self) -> Bool
}

extension Comparable {
  func compared(to other: Self) -> SortOrder {
    if self == other {
      return .same
    } else if self < other {
      return .before
    } else {
      return .after
    }
  }

  static func < (lhs: Self, rhs: Self) -> Bool {
    return lhs.compared(to: rhs) == .before
  }

  static func == (lhs: Self, rhs: Self) -> Bool {
    return self.compared(to: rhs) == .same
  }

  static func <= (lhs: Self, rhs: Self) -> Bool {
    return rhs < lhs
  }
  
  static func >= (lhs: Self, rhs: Self) -> Bool {
    return !(rhs < lhs)
  }

  static func > (lhs: Self, rhs: Self) -> Bool {
    return !(lhs < rhs)
  }
}

// IEEE comparison operators (these implementations already exist in std)
extension FloatingPoint {
  public static func == (lhs: T, rhs: T) -> Bool {
    return lhs.isEqual(to: rhs)
  }

  public static func < (lhs: T, rhs: T) -> Bool {
    return lhs.isLess(than: rhs)
  }

  public static func <= (lhs: T, rhs: T) -> Bool {
    return lhs.isLessThanOrEqualTo(rhs)
  }

  public static func > (lhs: T, rhs: T) -> Bool {
    return rhs.isLess(than: lhs)
  }

  public static func >= (lhs: T, rhs: T) -> Bool {
    return rhs.isLessThanOrEqualTo(lhs)
  }
}


// Comparable comparison operators (provides a total ordering)
extension FloatingPoint {
  @_inline
  public func compared(to other: Self) -> SortOrder {
    // Can potentially be implemented more efficiently -- this is just the clearest version
    if self.isLess(than: other) {
      return .before
    } else if other.isLess(than: self) {
      return .after
    } else {
      // Special cases

      // -0 < +0
      if self.isZero && other.isZero {
        // .plus == 0 and .minus == 1, so flip ordering to get - < +
        return (other.sign as Int).compared(to: self.sign as Int)
      }

      // NaN == NaN, NaN > +Inf
      if self.isNaN {
        if other.isNaN {
          return .same
        } else {
          return .after
        }
      } else if other.isNaN {
        return .before
      } 

      // Otherwise equality agrees with normal IEEE
      return .same
    }
  }

  @_implements(Equatable.==)
  public static func _comparableEqual(lhs: Self, rhs: Self) -> Bool {
    lhs.compared(to: rhs) == .same
  }

  @_implements(Comparable.<)
  public static func _comparableLessThan(lhs: Self, rhs: Self) -> Bool {
    lhs.compared(to: rhs) == .after
  }
}
```


Note that this design mandates several changes to the compiler:

* `@_implements` (or an equivalent mechanism) must be implemented to get the context-sensitive FloatingPoint behaviour. [There is a draft of how it should work here](https://gist.github.com/Gankro/f766cfacc98fb6a2cc4e2dc61f3742c6).

* Overload resolution needs to be fixed to resolve several ambiguities that the new default implementations of `<` and `==` for Comparable introduce:
  * Some of these are bugs in the algorithm, like `<T, U>(T, U)` being ordered the same as `<T>(T, T)`.
  * Some of these are "fundamental", and require a special annotation to resolve if we wish to have maximal source-stability. For instance, a blanket `== <T: SomeProtocolUnrelatedToComparable>(T, T)` implementation, which previously would have worked, is fundamentally ambiguous with the `== <T: Comparable>(T, T)` default. [There is a draft of how this could be resolved here](https://gist.github.com/Gankro/956f03240546a810e326e41c223b056f).

* The compiler must verify that either `==` and `<`, or `compared(to:)` is overridden by every type that conforms to Comparable.




## Source compatibility

Existing implementors of Comparable will be unaffected, though they should consider implementing the new `compared` method as the default implementation may be suboptimal.

Consumers of Comparable will be unaffected, though they should consider calling the `compared` method if it offers a performance advantage. 

APIs which provide a `(Self, Self) -> Bool` overload for comparison should consider adding a `(Self, Self) -> SortOrder` overload.

Existing implementors of FloatingPoint should be unaffected -- they will automatically get the new behaviour as long as they aren't manually implementing the requirements of Equatable/Comparable.

Existing code that works with floats may break if it's relying on some code bounded on `<T: Equatable/Comparable>`providing IEEE semantics. For most algorithms, NaNs would essentially lead to unspecified behaviour, so the primary concern is whether -0.0 == +0.0 matters.




## ABI stability 

N/A (this must be implemented before ABI stability is declared)



## Effect on API resilience

N/A




## Alternatives Considered



### Spaceship

Early versions of this proposal aimed to instead provide a `<=>` operator in place of `compared`. The only reason we moved away from this was that it didn't solve the problem that comparison didn't generalize.

Spaceship as an operator has a two concrete benefits over `compared` today:

* It can be passed as a higher-order function
* Tuples can implement it

In our opinion, these aren't serious problems, especially in the long term. 

Passing `<=>` as a higher order function basically allows types that aren't Comparable, but do provide `<=>`, to be very ergonomically handled by algorithms which take an optional ordering function. Types which provide the comparable operators but don't conform to Comparable are only pervasive due to the absence of conditional conformance. We shouldn't be designing our APIs around the assumption that conditional conformance doesn't exist. 

When conditional conformance is implemented, the only should-be-comparable-but-aren't types that remain are tuples, which we should potentially have the compiler synthesize conformances for.

Similarly, it should be one day possibly to extend tuples, although this is a more "far future" matter. Until then, the `(T, T) -> Bool` predicate will always also be available, and `<` can be used there with the only downside being a potential performance hit.



### Just Leave Floats Alone

The fact that sorting floats leads to a mess, and storing floats can lead to memory leaks and data loss isn't acceptable.




### Just Make Floats Only Have A Total Order

This was deemed too surprising for anyone familiar with floats from any other language. It would also probably break a lot more code than this change will.




### Just Make Floats Not Comparable

Although floats are more subtle than integers, having places where integers work but floats don't is a poor state of affairs. One should be able to sort an array of floats and use floats as keys in data structures, even if the latter is difficult to do correctly.



### PartialComparable

PartialComparable would essentially just be Comparable without any stated ordering requirements, that Comparable extends to provide ordering requirements. This would be a protocol that standard IEEE comparison could satisfy, but in the absence of total ordering requirements, PartialComparable is effectively useless. Either everyone would consume PartialComparable (to accept floats) or Comparable (to have reasonable behaviour).

The Rust community adopted this strategy to little benefit. The Rust libs team has frequently considered removing the distinction, but hasn't because doing it backwards compatibly would be complicated. Also because merging the two would just lead to the problems Swift has today.




### Different Names For SortOrder

A few different names for SortOrder and its variants were passed around:

* `enum Ordering { case less, equal, greater }` ([as used by Rust](https://doc.rust-lang.org/std/cmp/enum.Ordering.html))
* `enum ComparisonResult { case ascending, same, descending }` ([as used by Foundation](https://developer.apple.com/reference/foundation/comparisonresult))
* `enum SortOrder { case inOrder, same, outOfOrder }`

The choice of case names is non-trivial because the enum shows up in different contexts where different names makes more sense. Effectively, one needs to keep in mind that the "default" sort order is ascending to map between the concept of "before" and "less". 

We found the before/after naming to provide the most intuitive model for custom sorts -- referring to `ascending` or `less` is confusing when trying to implement a descending ordering. Similarly the inOrder/outOfOrder naming was too indirect -- it's more natural to just say where to put the element.

Once we had decided that the enum should focus on the sorting case, it made sense to call it SortOrder to emphasize this fact.




### Comparison Mode Decorators

In response to the desire to generalize comparison, it has been suggested that types should instead provide decorators that provide different comparison modes. For instance:

```
string1.withLocale(locale) < string2.withLocale(locale)
```

The benefits are as follows:

* Updating most code, which only uses `<` or `==`, to use a non-default comparison will be more clean -- just tack on `withLocale` instead of rewriting it to use `compared`.

* This can potentially be used to pass down the new ordering behaviour to container types -- `sortedTree[string.withLocale(locale)] = y`



The concerns for this are as follows:

* If one is specifically asking the question "how do I compare in a locale-sensitive way", then generalizations are potentially easier to discover than decorators. Especially if the decorator has a name that the user didn't guess.

* Having to provide a locale to both sides of the comparison is a bit tedious; but allowing asymmetric comparison between a String and DecoratedString is potentially a bloat issue.

* Should these decorators be type-level, value-level, or a flag? That is, are they `StringWithLocale<Locale>`, `StringWithLocale(locale)`, or `String(with: locale)`? Hoisting this to the type level can have some safety and clarity gains over the others by eliminating the question of "what happens if the locales don't match" (it's a compilation error). But this may also make the API unwieldy. Making the decorator a flag that String itself stores would lead to bloat in a core type.

* How should multiple arguments compose? Specifically in the context of API evolution. Say we want to add a case insensitive mode -- with `compared` this is easy: `compared(to: Self, with: Locale = defaultLocale, caseSensitive: Bool = true)`. Type level decorators are a bit messy here `StringWithLocale<Locale>` wants to become `StringWithLocaleAndCaseSensitivity<Locale, Sensitivity>. A solution that works with ABI stability requires the default generics proposal to be accepted -- `StringWithComparisonMode<Locale=DefaultLocale, CaseSensitivity=Sensitive>`. Without this such an upgrade can't be done in an ABI-stable way.


Ultimately we decided against this approach because it seemed more complex to get right. It's also an approach that isn't *prevented* by this strategy. If we later decide it's a good solution, we can add these decorators later backwards-compatibly. Our users can also add them with extensions to their own code, as usual.




## Future Work

More things can be done here, but those should hopefully be backwards compatible to introduce. Two paths that are worth exploring:



### Ergonomic Generalized Comparison for Keyed Containers 

Can we make it ergonomic to use an (arbitrary) alternative comparison strategy for a Dictionary or a BinaryTree? Should they be type-level Comparators, or should those types always store a `(Key, Key) -> SortOrder` closure?

We can avoid answering this question because Dictionary is expected to keep a relatively opaque (resilient) ABI for the foreseeable future, as many interesting optimizations will change its internal layout. Although if the answer is type-level, then Default Generic Parameters must be accepted to proceed down this path.



### SortOrder Conveniences

There are a few conveniences we could consider providing to make SortOrder more ergonomic to manipulate. Such as:

```
// A way to combine orderings
func SortOrder.breakingTiesWith(_ order: () -> SortOrder) -> SortOrder

array.sort {
  $0.x.compared(to: $0.y)
  .breakingTiesWith { $0.y.compared(to: $1.y) }
}
```

and

```
var inverted: SortOrder

// A perhaps more "clear" way to express reversing order than `y.compared(to: x)`
x.compared(to: y).inverted
```

But these can all be added later once everyone has had a chance to use them.
