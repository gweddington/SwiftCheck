[![Build Status](https://travis-ci.org/typelift/SwiftCheck.svg?branch=master)](https://travis-ci.org/typelift/SwiftCheck)

SwiftCheck
==========

QuickCheck for Swift.

For those already familiar with the Haskell library, check out the source.  For
everybody else, see the [Tutorial Playground](Tutorial.playground) for a
beginner-level introduction to the major concepts and use-cases of this library.
 
Introduction
============

SwiftCheck is a testing library that automatically generates random data for 
testing of program properties.  A property is a particular facet of an algorithm
or data structure that must be invariant under a given set of input data,
basically an `XCTAssert` on steroids.  Where before all we could do was define
methods prefixed by `test` and assert, SwiftCheck allows program properties and tests to be treated like *data*.

To define a program property the `forAll` quantifier is used with a type
signature like `(A, B, C, ... Z) -> Testable where A : Arbitrary, B : Arbitrary ...
Z : Arbitrary`.  SwiftCheck implements the `Arbitrary` protocol for most STL types
and implements the `Testable` protocol for `Bool` and several other related
types.  For example, if we wanted to test the property that every Integer is
equal to itself, we would express it as such:

```swift
func testAll() {
    // 'property' notation allows us to name our tests.  This becomes important
    // when they fail and SwiftCheck reports it in the console.
    property("Integer Equality is Reflexive") <- forAll { (i : Int) in
        return i == i
    }
}
```

For a less contrived example, here is a program property that tests whether
Array identity holds under double reversal:

```swift
// Because Swift doesn't allow us to implement `Arbitrary` for certain types,
// SwiftCheck instead implements 'modifier' types that wrap them.  Here,
// `ArrayOf<T : Arbitrary>` generates random arrays of values of type `T`.
property("The reverse of the reverse of an array is that array") <- forAll { (xs : ArrayOf<Int>) in
    // This property is using a number of SwiftCheck's more interesting 
    // features.  `^&&^` is the conjunction operator for properties that turns
    // both properties into a larger property that only holds when both sub-properties
    // hold.  `<?>` is the labelling operator allowing us to name each sub-part
	// in output generated by SwiftCheck.  For example, this property reports:
    //
    // *** Passed 100 tests
    // (100% , Right identity, Left identity)
    return
        (xs.getArray.reverse().reverse() == xs.getArray) <?> "Left identity"
        ^&&^
        (xs.getArray == xs.getArray.reverse().reverse()) <?> "Right identity"
}
```

Because SwiftCheck doesn't require tests to return `Bool`, just `Testable`, we
can produce tests for complex properties with ease:

```swift
property("Shrunken lists of integers always contain [] or [0]") <- forAll { (l : ArrayOf<Int>) in
    // Here we use the Implication Operator `==>` to define a precondition for
    // this test.  If the precondition fails the test is discarded.  If it holds
    // the test proceeds.
    return (!l.getArray.isEmpty && l.getArray != [0]) ==> {
        let ls = self.shrinkArbitrary(l).map { $0.getArray }
        return (ls.filter({ $0 == [] || $0 == [0] }).count >= 1)
    }()
}
```

Properties can even depend on other properties:

```swift
property("Gen.oneOf multiple generators picks only given generators") <- forAll { (n1 : Int, n2 : Int) in
    let g1 = Gen.pure(n1)
    let g2 = Gen.pure(n2)
    // Here we give `forAll` an explicit generator.  Before SwiftCheck was using
    // the types of variables involved in the property to create an implicit
    // Generator behind the scenes.
    return forAll(Gen.oneOf([g1, g2])) { $0 == n1 || $0 == n2 }
}
```

All you have to figure out is what to test.  SwiftCheck will handle the rest.  

Shrinking
=========
 
What makes QuickCheck unique is the notion of *shrinking* test cases.  When fuzz
testing with arbitrary data, rather than simply halt on a failing test, SwiftCheck
will begin whittling the data that causes the test to fail down to a minimal
counterexample.

For example, the following function uses the Sieve of Eratosthenes to generate
a list of primes less than some n:

```swift
/// The Sieve of Eratosthenes:
///
/// To find all the prime numbers less than or equal to a given integer n:
///    - let l = [2...n]
///    - let p = 2
///    - for i in [(2 * p) through n by p] {
///          mark l[i]
///      }
///    - Remaining indices of unmarked numbers are primes
func sieve(n : Int) -> [Int] {
    if n <= 1 {
        return [Int]()
    }
    
    var marked : [Bool] = (0...n).map(const(false))
    marked[0] = true
    marked[1] = true
    
    for p in 2..<n {
        for i in stride(from: 2 * p, to: n, by: p) {
            marked[i] = true
        }
    }
    
    var primes : [Int] = []
    for (t, i) in Zip2(marked, 0...n) {
        if !t {
            primes.append(i)
        }
    }
    return primes
}

/// Short and sweet check if a number is prime by enumerating from 2...⌈√(x)⌉ and checking 
/// for a nonzero modulus.
func isPrime(n : Int) -> Bool {
    if n == 0 || n == 1 {
        return false
    } else if n == 2 {
        return true
    }
    
    let max = Int(ceil(sqrt(Double(n))))
    for i in 2...max {
        if n % i == 0 {
            return false
        }
    }
    return true
}

```

We would like to test whether our sieve works properly, so we run it through SwiftCheck
with the following property:

```swift
import SwiftCheck

property("All Prime") <- forAll { (n : Int) in
    return sieve(n).filter(isPrime) == sieve(n)
}
```

Which produces the following in our testing log:

```
Test Case '-[SwiftCheckTests.PrimeSpec testAll]' started.
*** Failed! Falsifiable (after 10 tests):
4
```

Indicating that our sieve has failed on the input number 4.  A quick look back
at the comments describing the sieve reveals the mistake immediately:

```diff
- for i in stride(from: 2 * p, to: n, by: p) {
+ for i in stride(from: 2 * p, through: n, by: p) {
```

Running SwiftCheck again reports a successful sieve of all 100 random cases:

```
*** Passed 100 tests
```

Custom Types
============

SwiftCheck implements random generation for most of the types in the Swift STL.
Any custom types that wish to take part in testing must conform to the included
`Arbitrary` protocol.  For the majority of types, this means providing a custom
means of generating random data and shrinking down to an empty array. 

For example:

```swift
import SwiftCheck
 
public struct ArbitraryFoo {
    let x : Int
    let y : Int

    public static func create(x : Int) -> Int -> ArbitraryFoo {
        return { y in ArbitraryFoo(x: x, y: y) }
    }

    public var description : String {
        return "Arbitrary Foo!"
    }
}

extension ArbitraryFoo : Arbitrary {
    public static var arbitrary : Gen<ArbitraryFoo> {
        return ArbitraryFoo.create <^> Int.arbitrary <*> Int.arbitrary
    }
}

class SimpleSpec : XCTestCase {
    func testAll() {
        property("ArbitraryFoo Properties are Reflexive") <- forAll { (i : ArbitraryFoo) in
            return i.x == i.x && i.y == i.y
        }
    }
}
```

For everything else, SwiftCheck defines a number of combinators to make working
with custom generators as simple as possible:

```swift
let onlyEven = Int.arbitrary.suchThat { $0 % 2 == 0 }

let vowels = Gen.fromElementsOf(["A", "E", "I", "O", "U" ])

let randomHexValue = Gen<UInt>.choose((0, 15))

let uppers : Gen<Character>= Gen<Character>.fromElementsIn("A"..."Z")
let lowers : Gen<Character> = Gen<Character>.fromElementsIn("a"..."z")
let numbers : Gen<Character> = Gen<Character>.fromElementsIn("0"..."9")
 
/// This generator will generate `.None` 1/4 of the time and an arbitrary
/// `.Some` 3/4 of the time
let weightedOptionals = Gen<Int?>.frequency([
	(1, Gen<Int?>.pure(nil)),
	(3, Optional.Some <^> Int.arbitrary)
])
```
 
For instances of many complex or "real world" generators, see 
[`ComplexSpec.swift`](SwiftCheckTests/ComplexSpec.swift).

System Requirements
===================

SwiftCheck supports OS X 10.9+ and iOS 7.0+.

Setup
=====

SwiftCheck can be included one of two ways:

**Using Carthage**

- Add SwiftCheck to your Cartfile
- Run `carthage update`
- Drag the relevant copy of SwiftCheck into your project.
- Expand the Link Binary With Libraries phase
- Click the + and add SwiftCheck
- Click the + at the top left corner to add a Copy Files build phase
- Set the directory to `Frameworks`
- Click the + and add SwiftCheck

**Framework**

- Drag SwiftCheck.xcodeproj into your project tree
  as a subproject
- Under your project's Build Phases, expand Target Dependencies
- Click the + and add SwiftCheck
- Expand the Link Binary With Libraries phase
- Click the + and add SwiftCheck
- Click the + at the top left corner to add a Copy Files build phase
- Set the directory to Frameworks
- Click the + and add SwiftCheck

License
=======

SwiftCheck is released under the MIT license.

