# check-it!

[![Build Status](https://travis-ci.org/DalekBaldwin/check-it.svg?branch=master)](https://travis-ci.org/DalekBaldwin/check-it)

This is a randomized property-based testing library for Common Lisp. Rather than being a full-fledged general test framework in its own right, it's designed to embed randomized tests in whatever framework you like.

The basic idea behind QuickCheck-style tools is simple, but they become complicated as different features interact, so some behaviors and API details may not be stable. I recommend that you configure your projects' test systems to optionally exclude check-it tests (e.g. by keeping them all in a separate package from your other tests) so that a breaking change won't impede your development workflow too much.

## Table of Contents
  * [Generating](#generating)
    * [Number Generators](#number-generators)
    * [Character Generators](#character-generators)
    * [List and Tuple Generators](#list-and-tuple-generators)
    * [String Generator](#string-generator)
    * [Or Generator](#or-generator)
    * [Guard Generator](#guard-generator)
    * [Struct Generator](#struct-generator)
    * [Mapped Generators](#mapped-generators)
    * [Chained Generators](#chained-generators)
    * [User-Defined Generators](#user-defined-generators)
    * [Generator Macros](#generator-macros)
  * [Checking](#checking)
    * [Regression Cases](#regression-cases)
      * [Note on Mapped Generators](#note-on-mapped-generators)
  * [Shrinking](#shrinking)
    * [Note on Destructive Operations](#note-on-destructive-operations)

## Generating

The heart of check-it is the generator DSL. Generator objects are constructed using the macro `generator`, which takes a single argument: a form denoting a type spec. Generator type specs are not exactly the same as Common Lisp type specs, but they're designed to look syntactically similar.

Values are generated by calling the function `generate` on generator objects. This function is normally called behind the scenes during a test run, but it can also be used à la carte to generate interesting data for other purposes.

The different varieties of generators are:

### Number Generators

The `integer` generator accepts the same syntax as the standard compound type specifier syntax for integers:

```lisp
;; generates any integer
(integer)

;; also generates any integer
(integer * *)

;; generates integers greater than or equal to 5
(integer 5)

;; generates integers less than or equal to 10
(integer * 10)

;; generates integers between -3 and 4
(integer -3 4)
```

The `real` generator works similarly.

A number generator's bounds can also be expressions: `(real 0 (* 2 pi))`

You could even do things like this if you wanted:

```lisp
(let ((x (something)))
  (generator (real 0 x)))
```

But you should probably define a custom generator type or use a combinator like `map` or `chain` instead.

In addition to the constraints you choose in the type specifier, the absolute values of generated numbers are also bounded by the parameter `*size*`.

### Character Generators

The `character` generator accepts a syntax much like the number generators. It accepts either characters or character codes as bounding values:

```lisp
;; any character
(character)

;; any character, again
(character * *)

(character #\a #\f)

;; same as previous
(character 97 102)
```

Two additional character generators are predefined for common cases: `alpha` for `#\a`-`#\z` and `#\A`-`#\Z`, and `alphanumeric` for `#\a`-`#\z`, `#\A`-`#\Z`, and `#\0`-`#\9`.

### List and Tuple Generators

The `list` generator generates a list of values each generated by its subgenerator. The generated list has a random length bounded by the parameter `*list-size*`.

```lisp
(let ((a-generator
       (generator (list (integer)))))
  (generate a-generator))
;; sample result: (9 -3 3 -2 3 10 6 -8 9 10)
```

You can also specify length bounds with `:min-length` and `:max-length`:

```lisp
(generator (list (integer) :min-length 5 :max-length 10))
```

`:max-length` provides a permanent bound, but `*list-size*` can change in certain situations described further down in this document. For convenience, you can specify lists of fixed length with a single keyword argument:

```lisp
(generator (list (integer) :length 5))
```

The `tuple` generator generates a list containing one result from each of its subgenerators in order, so unlike the `list` generator, its length is fixed:

```lisp
(let ((another-generator
       (generator (tuple (integer) (real)))))
  (generate another-generator))
;; sample result: (-6 -4.168296)
```

### String Generator

A special form of the `list` generator (equally length bounded by `*list-size*`) is the `string` generator to generate strings consisting of randomly chosen alphanumeric characters.

```lisp
(generate (generator (string :min-length 10 :max-length 20)))
;; sample output: "2k464h72CZ4TE1KQ"
```

### Or Generator

The `or` generator randomly chooses one of its subgenerators and returns its result. It biases its choice in certain situations, to be described later.

```lisp
(generate (generator (or (integer) (real))))
;; sample result: -1.0087109
```

### Guard Generator

The `guard` generator generates a result from its subgenerator and checks whether the result passes the guard predicate. If not, it keeps generating new results until it finds a value that does.

```lisp
;; same range of values as (integer 5)
(guard (lambda (x) (>= x 5)) (integer))
```

### Struct Generator

The `struct` generator generates a struct of a given struct type with slots generated by the corresponding slot generators. So if you have a struct definition that looks like:

```lisp
(defstruct a-struct
  a-slot
  another-slot)
```

You can create a generator type spec that looks like:

```lisp
(struct a-struct :a-slot (integer) :another-slot (real))
```

In order to use this kind of generator, the struct type must have a default constructor function. Of course, you can always use simple generators to specify all the atomic data elements you need and manually assemble them into more complex data structures at the start of your test body. However, for complex, nested generator forms, it may be difficult or impossible to unambiguously specify where to find the output of some deep sub-sub-generator that you wish to transform. For those situations, you need...

### Mapped Generators

A mapped generator applies a transformation function to the outputs of its subgenerators. This is a mapping in the mathematical sense, not the Lispy one; it does not map the function repeatedly to each element of a list. The transformation function should not mutate its arguments.

```lisp
(let ((*size* 100)
      (g (generator (tuple (map (lambda (x y) (list x y))
                                (integer 1 10)
                                (integer 20 30))
                           (map (lambda (x y) (list x (+ x y)))
                                (integer 1 9)
                                10)))))
  (generate g))
;;; sample result: ((10 28) (7 17))
```

### Chained Generators

You can create generators whose behavior is parameterized by other generators with `chain`. Every time a value is created from a chained generator,

1. Each parameterizing generator generates a new value,
2. Those values are bound to the associated variable names in the body of the `chain` form,
3. The body is evaluated to construct a new parameterized generator, and
4. The parameterized generator generates the end value.

For example, you can generate random NxN matrices like this:

```lisp
(let ((g (generator
          (chain ((n (integer 1 5)))
            (generator (list
                        (list (integer) :length n)
                        :length n))))))
  (generate g))
;; sample result: ((-2 0 -9 -6) (6 7 -3 -7) (9 -10 10 6) (-6 -10 -10 8))
```

The `generator` macro is implicitly wrapped around each parameterizer form for brevity. However, the body is regular Lisp code, so you must use `generator` to define your parameterized generator there.

If you provide only a name for a parameterizer instead of a list containing a name and a generator form, the name is assumed to be a variable already bound to a generator in the surrounding Lisp code:

```lisp
(let ((x (generator (integer 1 3)))
      (y (generator (integer 8 10))))
  (generator (chain (x y)
               (list (integer) :min-length x :max-length y))))
```

### User-Defined Generators

You can define your own generator types with `def-generator`.

User-defined generators can take a set of arguments specified by an ordinary lambda list, evaluated at the time of a generator object's construction. They can also be recursive (in which case recursive definition is evaluated anew to construct a generator, allowing child recursions be parameterized by parent recursions). Here's a useless example:

```lisp
;; same range of values as (integer x)
(def-generator recursive (x) (generator (or (integer x) (recursive x))))
```

You can also create higher-order recursive generators. However, each subgenerator needs to be a unique object for check-it to work correctly, so you need to explicitly create them like this:

```lisp
(def-generator higher-order (generator-maker)
  (generator (or (funcall generator-maker)
                 (higher-order generator-maker))))

(generate (generator (higher-order (lambda () (generator (integer))))))
```

With naive generation strategies, recursive generators can easily generate values of unbounded size. There are currently two ways to dampen the feedback explosion of recursively-generated data structures.

When a user-defined generator appears as an alternative in an `or` generator, its relative probability of being chosen decreases with each recursive descent.

```lisp
;; normally there would be an 87.5% chance of recursing on each generation
;; essentially guaranteeing unbounded growth
(def-generator tuple-explode (x)
  (generator (tuple (or x (tuple-explode (+ 1 x)))
                    (or x (tuple-explode (+ 1 x)))
                    (or x (tuple-explode (+ 1 x))))))

(let ((*recursive-bias-decay* 1.2)
      (*bias-sensitivity* 1.5))
  (generate (generator (tuple-explode 0))))
;; sample result: (0 0 (1 1 ((3 (4 4 (5 5 5)) (4 4 (5 5 5))) (3 3 3) 2)))
```

The change in bias at each recursive step is controlled by the parameter `*recursive-bias-decay*`, and the way biases of different alternatives interact to produce the actual relative probabilities is controlled by `*bias-sensitivity*`. The whole apparatus is set up in such a way that these parameters can be tuned without causing one alternative's probability to sharply shoot off toward zero or one, so you can play around with them and discover values that produce a reasonable distribution for your needs.

Additionally, the maximum possible list length is reduced with every `list` generation that occurs within the dynamic scope of another `list` generation.

```lisp
(def-generator list-explode ()
  (generator (or (integer) (list (list-explode)))))

(let ((*list-size* 10))
  (generate (generator (list-explode))))
;; sample result:
;; ((((-5 -9 (-7 -4)) NIL NIL (6) NIL) (-7 (3 (-4 (NIL)))) (5 -3) 3
;;   ((1 (4) 5) NIL ((NIL) -10)) -5 9)
;;  ((((-3) -6) NIL -4 (-5) -4)) ((-9) 0 1 0 3) -3 NIL)
```

But it's your responsibility not to write type specs that can't possibly generate anything other than unbounded values.

```lisp
(def-generator inherently-unbounded ()
  (generator (tuple (integer) (inherently-unbounded))))
```

### Generator Macros

You can define transformations using destructuring lambda lists that work just like macros within the generator DSL. These will be expanded in any place where a generator expression would be expected.

```lisp
(def-genex-macro maybe (form)
  `(or nil ,form))

(generate (generator (list (maybe (integer)))))
;; sample result: (NIL NIL NIL NIL 7 -4 2 NIL NIL 3 7 -8 NIL 7)
```

## Checking

Use the `check-it` macro to perform a test run. Here's another useless example:

```lisp
(let ((*num-trials* 100))
  (check-it (generator (integer))
            (lambda (x) (integerp x))))
```

This will generate `*num-trials*` random values and test them against the test predicate. If a random value fails, check-it will search for values of smaller complexity until it finds the least complex value it can that fails the test while respecting the generator's composite type spec, and print a failure description to `*check-it-output*`.

If the test predicate signals an uncaught error, `check-it` will catch it and treat it as a test failure, both for the initial test and the search for smaller failing values. If you want to test for an expected error, you should explicitly handle it within the test predicate and return a non-`nil` value.

`check-it` itself returns `t` if every trial passed and `nil` if one failed, so you can embed `check-it` forms within whatever test framework you're already using. Here's what a check-it test might look like using Stefil:

```lisp
(deftest useless-test ()
  (let ((*num-trials* 100))
    (is (check-it (generator (integer))
                  (lambda (x) (integerp x))))))
```

### Regression Cases

You can configure the `check-it` macro to automatically add new deterministic regression tests to your project when a randomized test fails. Here's the worst example yet:

```lisp
(deftest some-test-with-regression-cases ()
  (is
   (check-it (generator (struct a-struct
                                :a-slot (integer)
                                :another-slot (integer)))
             (lambda (x) (= (slot-value x 'a-slot) 0))
             :regression-id some-test-with-regression-cases
             :regression-file my-regression-test-file)))
```

This will (most likely) discover a failing case, shrink it, and append the following code to `my-regression-test-file`:

```lisp
(REGRESSION-CASE SOME-TEST-WITH-REGRESSION-CASES "#S(A-STRUCT :A-SLOT 1 :ANOTHER-SLOT 0)")
```

Regression cases must be given a `regression-id` to know which `check-it` form they apply to. It's probably simplest just to tag them with the same symbol you used to name the test. (This makes your test code a little more redundant, but it keeps check-it simple and framework-agnostic.) Although generators are meant to generate data with readable print representations, some objects (like structs) cannot be dumped into FASL files, so regression cases are encoded as strings which are read after the file is loaded in order to delay their construction.

It is recommended that you output your regression cases to a special file that initially contains only the line `(in-package :same-package-as-test-package)` and is loaded as part of the same system as your tests. Then the next time you run a check-it test, all the regression cases will be checked before any random cases are generated, which should hopefully light a fire under your ass to fix bugs you've already discovered before you hunt for new ones.

You can also provide a list of explicit inline examples that will also be checked before any random cases are generated using the `:examples` keyword argument to `check-it`.

Eventually, check-it will be integrated more elegantly with individual test frameworks by extending their APIs. For now, you can reduce code repetition by declaring a file to output regression cases for all `regression-id`s in a given package with `(register-package-regression-file :package file)`, so you don't have to mention the file in each test.

#### Note on Mapped Generators

All data types constructed with check-it's essential generators can be successfully converted to readable print representations, but it's possible that something you create with a mapped generator will produce a data type that can't. If you want to enable regression cases for tests involving unexternalizable data types, try to find a way to construct those data types within the test body if possible. Eventually, check-it may use a more robust solution for externalization, but this simple method can get you surprisingly far.

## Shrinking

When a generated value fails a test, check-it searches for a simpler value as follows:

An `integer` generator shrinks toward zero, or if zero is not included in its range, toward the bound in its type spec that has the smaller absolute value.

A `real` generator doesn't shrink, because the shrinkage procedure only really makes sense over discrete search spaces.

A `list` generator shrinks by repeatedly removing elements from the list and/or shrinking individual elements while respecting the type spec of the subgenerator.

An `or` generator shrinks by shrinking the one generator among its alternatives that was involved in generating the failed value. However, if this generator's value cannot be shrunk further, and other alternatives available to the `or` generator specify constant values, then one of those constant values may be substituted instead. So if the generator `(or (integer 5) :a-keyword)` failed after generating 5 from the `integer` subgenerator, check-it will attempt to use `:a-keyword`. This opens up more possible shrinkage paths in the overall space defined by the `or` generator's parent generators.

Other generators shrink by recursively shrinking their subgenerators while still respecting the overall type spec.

For mapped generators, the subgenerators shrink according to their own internal degrees of freedom, but the mapping is applied to the candidate replacement values from the subgenerators to determine whether resulting composite data structure still fails the test. There's a bit of a thorny issue here in that check-it doesn't know for sure whether a simpler value in a subgenerator will really correspond to a simpler value after the mapping is applied. For this reason, good mapping functions should typically specify one-to-one mappings that simply assemble new data structures, and more complicated logic should go in the test body.

For chained generators, the parameterizers are not involved in the shrinking process; the parameterized generator is shrunk according to the parameters it had when it failed the test.

### Note on Destructive Operations

Currently, generators cache their generated values and often make use of them during the shrinking process. If you perform destructive operations on the input to a test function, shrinking may not work correctly. This will be fixed eventually. In the meantime, you can turn off shrinking by giving `check-it` the keyword argument `:shrink-failures nil`.
