# MoonBit QuickCheck

## Introduction
The idea of QuickCheck was originally introduced in John's paper [*QuickCheck: a lightweight tool for random testing of Haskell programs*](https://doi.org/10.1145/351240.351266) and its Haskell derivation QuickCheck, which aimed at simplifying the writing of tests by generating them. The core is to automatically generate tests based on a the signature of the **specification / theorem** (i.e. properties which functions should satisfy) and QuickCheck then tests that the properties hold. The idea spread to other languages and is now implemented in **MoonBit**. Because of the differences in type systems between Haskell and MoonBit, the original idea morphed into something different. There are several differences to this approach:

- We have discarded many ideas in Haskell QuickCheck that have been proven inappropriate by time.
- Some features requires a very fancy type system were removed (Or not considered currently). For instance, the generation of functions requires GADT but MoonBit does not.
- MoonBit QuickCheck brings many modern academic ideas into industrial practice (listed [below](#references)).

## Beginner's Example

### First Look

Let's start with something very simple. Suppose that we just wrote a function `reverse` which takes an array as argument Failure Diagnosisand returns its reverse. And we want to test its functionality by writing unit tests:
```moonbit
test "reverse" {
  inspect!(reverse(([] : Array[Int])), content="[]")
  inspect!(reverse([1, 2, 3]), content="[3, 2, 1]")
}
```

Is this enough to prove that this function is correct? No, the bugs may lies in the remaining untested code. But if we want more rigorous verification, we may need to write more test cases. For the complicated program, there would never be time to. How can we reduce the cost of testing? The key to doing so must be to generalize the test functions, so that each function covers not only one test case, but many. This is where QuickCheck comes in. We can write a property that `reverse` should hold for any array, and QuickCheck will generate a large number of random array and check that the property holds for all of them.

But what property should we write? We may notice that the reverse of the reverse of an array is the original array. So we can write a property that `reverse` should hold for any array:
```moonbit
fn prop_reverse_identity(arr : Array[Int]) {
  reverse(reverse(arr)) == arr
}
```

We may consider the code logically as:

$$
\forall x: \text{Array[T]}.\space \text{reverse}(\text{reverse}(x)) = x
$$

Looks good, now we can use QuickCheck to test this property (Note that we should wrap the function in `Arrow` to make it a `Testable` though implicit conversion not work now):
```moonbit
test {
  quick_check!(Arrow(prop_reverse_identity))
}
```

The result (The three number in the brackets are the number of passed, discarded and total tests):
```
+++ [100/0/100] Ok, passed!
```

### Failure Diagnosis

Now let's see another example: Suppose we wrote a function `remove(arr : Array[Int], x : Int) -> Array[Int]` that takes an array and an element and returns a new array with **all** occurrences of `x` removed. The intuitive implementation is to search for `x` in the array and remove it if found:

```moonbit
fn remove(arr : Array[Int], x : Int) -> Array[Int] {
  match arr.search(x) {
    Some(i) => arr.remove(i) |> ignore
    None => ()
  }
  arr
}
```

We may consider the following properties:
- If an element was removed from the array, the length of the array should be less than or equal to the original array

$$
\forall x:\text{T}, a:\text{Array[T]} . \space
\text{length}(\text{remove}(a,x)) \leq \text{length}(a)
$$

- If an element was removed from the array, the element should not exist in the array

$$
\forall x:\text{T}, a:\text{Array[T]} . \space
x\not\in\text{remove}(a,x)
$$

Now we translate the first property into MoonBit code (Note that the property function should have **only one** argument, but we can use tuple to pass multiple arguments):
```moonbit
fn prop_length_is_not_greater(iarr : (Int, Array[Int])) -> Bool {
  let (x, arr) = iarr
  let len = arr.length()
  remove(arr, x).length() <= len
}
```

Run QuickCheck, all tests passed. However, this property is not considered a good property because it is can be fulfilled easily and the test is not very meaningful. Most Bugs may still exist in the function.
```
test {
  quick_check!(Arrow(prop_length_is_not_greater))
}

// +++ [100/0/100] Ok, passed!
```

The later property is considered better (The `with_max_success` function sets the number of tests that should be performed to `1000`):
```moonbit
fn prop_remove_not_presence(iarr : (Int, Array[Int])) -> Bool {
  let (x, arr) = iarr
  remove(arr, x).contains(x).not()
}

test {
  quick_check!(Arrow(prop_remove_not_presence) |> with_max_success(1000))
}
```

The QuickCheck reports a failure with a counterexample in the second line after almost 100 tests:
```
*** [107/0/1000] Failed! Falsified.
(0, [0, 0])
```

When `x = 0` and `arr = [0, 0]` the property does not hold. The `remove` function should remove all occurrences of `x` in the array, but it only removes the first one. In this example, as in many cases, the randomly generated test case contains junk values that have nothing to do with the test failure itself. When QuickCheck finds a failing test case, it tries to simplify it by applying some shrinking strategies. In fact the failing case is much complex but the shrinking algorithm has simplified it to the minimal case and presented it to us.

### Traits

The `quickcheck` function has the following signature:

```moonbit
fn quick_check[P : Testable](P) -> Unit!Failure
```

It takes a value of type that implemented `Testable` as argument and runs the test. 

```moonbit
type Arrow[A, P] (A) -> P
impl[P : Testable, A : Arbitrary + Shrink + Show] Testable for Arrow[A, P]
```

The `Arrow` type (wrapped function) does implement the `Testable` trait, so we can pass the property function directly to `quick_check`. Note that the argument type has implemented the `Arbitrary` , `Shrink` and `Show` traits. The `Show` is necessary for the `quick_check` function to print the **counterexample** when a test fails. Now we give some brief introduction to other two traits.

`Arbitrary` is a trait that generates random values of a type.  It has a method `arbitrary` that takes a size and a random number generator and  then returns a random value of the type.
```moonbit
pub trait Arbitrary {
  arbitrary(Int, RandomState) -> Self
}
```

`Shrink` is a trait that shrinks a value to a simpler one. It has a method `shrink` that takes a value and returns a iter of simpler values (Lazy).
```moonbit
pub trait Shrink {
  shrink(Self) -> Iter[Self]
}
```

### Generators

Test data is produced by test data generators. QuickCheck defines default generators for some often used types, but you can use your own, and will need to define your own generators for any new types you introduce (Or for simple types we can use the trivial definition by `derive(Arbitrary)`).

Generators have types of the form `Gen[T`, which is a generator for values of type `T`. It was defined as a struct, it contains single field named `gen`, with type `(Int, RandomState) -> T` (The first parameter is the size of the generated value, and the second is the random number generator):
```moonbit
struct Gen[T] {
  gen : (Int, RandomState) -> T
}
```

QuickCheck defines a series of useful methods for `Gen[T]`, for the most basic ones, we can use `Gen::new(f: (Int, RandomState) -> T)` to create a generator from a function and run it by invoking the `run` method:
```moonbit
let g : Gen[Int] = Gen::new(..) // Suppose we have a generator for Int
let x : Int = g.run(100, RandomState::new()) // Generate a random Int at size 100
```

`Gen[T]` was implemented as functor, applicative and monad which means you can compose it in many ways. 
```moonbit
fn pure[T](val : T) -> Gen[T]
fn fmap[T, U](self : Gen[T], f : (T) -> U) -> Gen[U]
fn ap[T, U](self : Gen[(T) -> U], v : Gen[T]) -> Gen[U]
fn bind[T, U](self : Gen[T], f : (T) -> Gen[U]) -> Gen[U]
```

For instance, you can use the `fmap` method to transform the generated value:
```moonbit
let g : Gen[Int] = Gen::new(..)
let g2 : Gen[Int] = g.fmap(fn(x) { x + 1 })
let g3 : Gen[String] = g.fmap(fn(x) { x.to_string() })
```

Or create a dependent generator:
```moonbit
let g : Gen[Int] = Gen::new(..)
let g2 : Gen[Int] = g.bind(fn(x : Int) { 
  if x == 0 {
    Gen::pure(100)
  } else {
    Gen::pure(200)
  }
})
```

The following documents explains some useful combinators for `Gen[T]`.

#### Choosing between alternatives

A generator may take the form `one_of` which chooses among the generators in the array with equal probability. For example, this generates a random boolean which is true with probability one half:

```moonbit
let gen_bool : Gen[Bool] = one_of([pure(true), pure(false)])
```

If you want to control the distribution of results using frequency instead. We have `frequency` which chooses a generator from the array randomly, but weighs the probability of choosing each alternative by the factor given. For example, this generates true in the probability of $4/5$.

```moonbit
let gen_freq : Gen[Bool] = frequency([(4, pure(true)), (1, pure(false))])
```

#### Size parameter

We have pointed that test data generators have an size parameter. QuickCheck begins by generating small test cases, and gradually increases the size as testing progresses. Different test data generators interpret the size parameter in different ways: some _ignore_ it, some interprets it as an upper bound on the size of containers. Most generators defined in `Arbitrary` trait depends on the size parameter but most separated generator combinators does not. 

You can obtain the value of the size parameter using `sized` combinator.
```moonbit
pub fn sized[T](f : (Int) -> Gen[T]) -> Gen[T]
```

For example, we can make a trivial generator for a list of integers with a given length:
```moonbit
let gen : Gen[Int] = sized(fn { size => pure(size) })
let arr = Array::makei(20, fn { i => gen.sample(size=i) })
inspect!(arr, content="[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]")
```

The purpose of size control is to ensure that test cases are large enough to reveal errors, while remaining small enough to test fast. Sometimes the default size control does not achieve this. So we have provided some tool functions like `resize` and `scale` (check the document for details) for this purpose.

#### Default Generators

QuickCheck defines default test data generators and shrinkers for some often used types (By trait `Arbitrary` and `Shrink`). You do not need to define or apply these explicitly for every property because QuickCheck can provide a property with appropriate generators and shrinkers for the property's arguments. But if you are required to do so, you can use the `forall` for explicitly universal quantification. 

```moonbit
quick_check!(forall(spawn(), fn(x : List[Int]) { x.rev().rev() == x }))
```

Note that the `spawn` function is useful for creating `Gen[T]` from its arbitrary instance, in this example the type checker infers the type of the first argument in `forall` to be `Gen[List[T]]` from the type of the property function.
```moonbit
fn Gen::spawn[T : Arbitrary]() -> Gen[T]
```

### Shrinkers

Shrinkers have types of the form `(T) -> Iter[T]`. when given a value, a shrinker produces a sequence of values that are (in some way) simpler than the given value. If QuickCheck finds a set of values that fails a given property, it will try to make that value simpler than the original value by getting the shrinks for the value and trying each one in turn to check that the property is still false. If it is, the smaller value becomes the new counterexample and the shrinking process continues with that value.

## Advanced Topics

### Custom Generator

```moonbit
test {
  quick_check!(
    forall(
      spawn(),
      fn(a : Array[Int]) {
        forall(one_of_array(a), 
          fn(y : Int) { remove(a, y).contains(y).not() })
        |> filter(a.length() != 0)
      },
    ),
  )
}
```

### Conditional Properties

Before we step further to verify out hypothesis, we present another manner to write tests:

Now let's further verify our hypothesis: The function `remove` only removes the first one but not all. So if we instead formulate a conditional property which restricts the input array contains no duplicated elements, then our tests should pass. Use the function `filter` can filter out unwanted test cases:

```moonbit
test {
  fn no_duplicate(x : Array[Int]) -> Bool {
    @sorted_set.from_iter(x.iter()).size() == x.length().to_int64()
  }

  quick_check!(
    forall(
      spawn(),
      fn(iarr : (Int, Array[Int])) {
        let (x, arr) = iarr
        filter(remove(arr, x).contains(x).not(), no_duplicate(arr))
      },
    ),
  )
}
```

- The `no_duplicate` function checks whether the input array contains no duplicated elements.
- If a type `T` is implemented with `Arbitrary` class, we can use `spawn()` to get the generator `Gen[T]`.
- `forall` is an explicit universal quantification, which uses an explicitly given test case generator.
- The `filter` combinator filters out the test cases that do not satisfy the condition.


## Application
We have applied MoonBit QuickCheck to test the correctness of the MoonBit core. For now we have found several bugs in the core, including:

- [Escaped characters in JSON](https://github.com/moonbitlang/core/pull/812)
- [Invalid JSON in inspect function](https://github.com/moonbitlang/core/issues/819)
- [BigInt.op_sub edge cases](https://github.com/moonbitlang/core/issues/860)
- [Unterminated moon test](https://github.com/moonbitlang/core/issues/875)
- [Incorrect BigInt.op_div due to the ill normalization step](https://github.com/moonbitlang/core/issues/942)

## Roadmap and Future Work

### Testing Strategy

- [x] Randomized property test: Run the test for `N` random elements drawn from the argument type.
  - Similar to Haskell QuickCheck
- [ ] Exhaustive specialized property test: Run the test for each element of a subset of the argument type.
  - Similar to SmallCheck and Feat

### Data Generation

- [x] Functional Enumeration 
- [ ] Falsify (with free shrinkers)
- [x] Random Generation
  - Currently use the SplitMix algorithms

### Property Verification

- [x] Boolean Property
- [x] Test Invariants
- [x] Operation Invariance
- [x] `Testable` Trait
- [ ] Existential Quantification

### Shrinking

- [x] Linear Shrinking
- [ ] Integrated Shrinking 
- [ ] Internal Shrinking

## References

- [OCaml QCheck](https://github.com/c-cube/qcheck)
- [Haskell QuickCheck](https://hackage.haskell.org/package/QuickCheck)
- [Feat: functional enumeration of algebraic types](https://doi.org/10.1145/2430532.2364515)
- [QuickCheck: a lightweight tool for random testing of Haskell programs](https://doi.org/10.1145/351240.351266)
- [SmallCheck and Lazy SmallCheck automatic exhaustive testing for small values](https://doi.org/10.1145/1411286.1411292)
- [Automatic Testing of Operation Invariance](https://ceur-ws.org/Vol-1335/wflp2014_paper9.pdf)
- [Fast Splittable Pseudorandom Number Generators](https://doi.org/10.1145/2660193.2660195)
- [falsify: Internal Shrinking Reimagined for Haskell](https://doi.org/10.1145/3609026.3609733)
- [Selective applicative functors](https://doi.org/10.1145/3342521)
