# Proposed New Approach for Pattern Matching in ECMAScript
Author: Daniel Ethridge ([@wlib](https://git.io/de)).
## Introduction
The entire purpose of pattern matching is to branch logic. We currently use the
`if` statement and the `switch` statement to do this. The issue is that neither
of these constructs are ergonomic. `if` statements are usually ad hoc and using
`switch` is usually avoided because of its verbosity and unintuitive semantics.
We can solve these problems by reapproaching this issue from better principles.

First, understand that the main purpose of pattern matching is to take a value,
and to do something based on what that value is. In all cases, this can be just
seen as [elimination of sum types](https://en.wikipedia.org/wiki/Tagged_union).

This, in other words, means that a pattern matching solution needs to know about types.

### Types?
For better or worse, ECMAScript is dynamically and weakly typed (this can be
better described as [unityped](http://lists.seas.upenn.edu/pipermail/types-list/2014/001733.html)).
A result of this is that pattern matching is an inherently foreign concept within the language.

Fortunately, there is a solution to this. Here's a hint - destructuring:

```javascript
const someObject =
  { foo: 1
  , bar: [ "thank", "you", "tc39", "very", "cool" ]
  , bruh: true
  }

const { bruh, bar: [ , , ...message] } = someObject

console.log(bruh)    // true
console.log(message) // ["tc39", "very", "cool"]
```

[Destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)
in ECMAScript is structural. If the left hand side of the assignment
(the `{ bruh, bar: [ , , ...message] }` part) is a substructure of the right hand side,
the sides match and the destructuring assignment is successful.

This basically means that ECMAScript already has extensive support for
[duck typing](https://en.wikipedia.org/wiki/Duck_typing).

But on top of that, the language also somewhat supports
[nominal typing](https://en.wikipedia.org/wiki/Nominal_type_system) using the
[`instanceof` operator](https://javascript.info/instanceof):

```javascript
class Moment extends Date {}
class Bruh extends Moment {}

const bruh = new Bruh()

console.log(bruh instanceof Bruh)   // true
console.log(bruh instanceof Moment) // true
console.log(bruh instanceof Date)   // true
console.log(bruh instanceof Object) // true
```

...and on top of that the language has all kinds of arbitrary "types" that
need to be checked. For example, the only way to tell if a number is natural
is with a function:

```javascript
const isNaturalNumber = n =>
  Number.isInteger(n) && n >= 0
```

## Proposal
So our solution for pattern matching should incorporate both destructuring and
arbitrary matching in order to be maximally useful. It can be understood as
just a newer `switch` statement that allows matching if the value...

* ...is strictly equal (`===`) to a case, like a normal `switch` would.
* ...can be destructured by a case.
* ...fits some arbitrarily chosen criteria.

Finally, our solution must abandon the current `switch` syntax in order to be
more consistent with the syntax of all of the other statements.

Here is roughly what its syntax looks like:

```
switch* (<expression> [; <function>]) {
  case ([<expression> | <destructuring>]...) <block>
  ...
  default <block>
}
```

## Rationale

### Switch Asterisk
Using a `switch` followed by an asterisk is exactly the same approach that was taken when
[generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
where introduced in order to maintain backwards compatibility without taking a
new keyword. This decision could arbitrarily be changed to introduce and use a
keyword like `match`, but `switch *` will work without any unnecessary hassle.

### No Fall-Through
Unlike the classic `switch` statement, there is no fallthrough. This is
because fall-through is an unnecessary source of confusion.
Here's an example of counterintuitive fall-through behavior:

```javascript
switch (1) {
  case 0: console.log(0)
  default: console.log("default")
  case 2: console.log(2)
}
// default
// 2

switch (1) {
  case 0: console.log(0)
  case 1: console.log(1)
  default: console.log("default")
  case 2: console.log(2)
}
// 1
// default
// 2
```

But of course - the most common mistake is to forget to write all of the `break`'s.
Now, rather than fall-through, you can match one of multiple patterns like so:

```javascript
const userInput = "hello"

switch* (userInput) {
  case ("hi", "hello", "hey")
    console.log(`Oh, ${userInput}!`)
  
  case ("bye", "goodbye", "see ya")
    console.log("Have a good day!")

  default
    console.log("Sorry, I don't understand that much...")
}
```

### Optional Custom Match Function
Normally, a `case` clause only matches using strict equality or a destructuring
assignment. When matching another expression (when not destructuring), the
matching function is equivalent to `(a, b) => a === b`. This can be too rigid
for many cases in pattern matching, so an optional custom match function is
supported. This is especially useful for ranges or custom logic:

```javascript
// Contrived example that tests is two numbers are coprime
const gcd = (a, b) =>
  b == 0 ? Math.abs(a)
         : gcd(b, a % b)

const isCoPrimeTo = (a, b) =>
  gcd(a, b) === 1

switch* (14; isCoPrimeTo) {
  case (2, 4, 6, 8)
    console.log("This shouldn't happen")

  case (15)
    console.log("This should happen")
}

// Arbitrary pattern matching
const isLessThan = (a, b) => a < b

switch* (someNumber; isLessThan) {
  case (0)
    console.log("Negative")

  case (10)
    console.log("Single Digits")
  
  case (50)
    console.log("Under 50")

  default
    console.log("50 or more?")
}
```

### Blocks
A common pattern in ECMAScript is to use blocks:

```javascript
// Blocks can use { } to hold multiple statements
if (false) { 
  block;
  fullOf;
  statements;
}

while (false) {
  doNothingIGuess()
}

for (let i = 0; i < 3; i++) {
  doSomethingTo(i)
}

lambda = argument => {
  return result(argument)
}

// Or ignore the { } for just one statement
if (false)
  singleStatement;
else if (false)
  singleStatement
else
  singleStatement

while (false)
  doNothingIGuess()

for (let i = 0; i < 3; i++)
  doSomethingTo(i)

lambda = argument =>
  result(argument)
```

Now, the clauses within `switch*` behave the same way, making them much
more consistent with intuition for how pattern matching *should* work:

```javascript
const map = (f, iterable) => {
  switch* (iterable) {
    case ([])
      return []

    case ([x, ...xs])
      return [f(x), ...map(f, xs)]

    default {
      console.error("You cannot pass", iterable, "into the map function")
      throw new TypeError("Not an Iterable")
    }
  }
}
```

## More Examples

Matching sum types (though undeclared and not compile-time checked):

```javascript
const isInstanceOf = (a, b) => a instanceof b

const bTreeSize = tree => {
  switch* (tree; isInstanceOf) {
    case (Leaf)
      return 1

    case (Node)
      return bTreeSize(Node.left) + bTreeSize(Node.right)
  }
}

const apply = (f, maybeSomething) => {
  switch* (maybeSomething; isInstanceOf) {
    case (Just)
      return f(maybeSomething.value)

    case (Nothing)
      return maybeSomething

    default
      throw new TypeError(maybeSomething, "needs to be a Maybe type")
  }
}
```

This enables really creative pattern matching "types/sort of typeclasses":

```javascript
class Functor {
  static [Symbol.hasInstance](possibleFunctor) {
    return typeof possibleFunctor.map === "function"
  }
}

const myArray = [1, 2, 3, 4, 5]
const isInstanceOf = (a, b) => a instanceof b

switch* (myArray; isInstanceOf) {
  case (Functor) {
    console.log("This will match, because arrays are functors")
    console.log( myArray.map(n => n * 2) )
  }
}
```

## Caveats
There is an inherent ambiguity whether or not a case clause contains an
expression or a destructuring assignment. For example, what should this be?

```javascript
const stuff = [1]

switch* (someIterable) {
  case ([]) {
    // Is this an expression (match empty array) or destructuring?
  }
  case ([...stuff]) {
    // Does this match [1] or destructure?
  }
  case ([stuff]) {
    // Does this destructure or match [[1]]?
  }
  case (stuff) {
    // Does this destructure assign without actually destructuring?
    // Or does it evaluate as an expression to match [1]?
  }
}
```

This can be resolved by treating anything that can be parsed as a
destructuring assignment as a destructuring assignment. This includes
all `{}` and `[]`, but excludes bare identifiers and any other expressions.
So every clause except the last one is a destructuring match, while the last
clause matches the variable `stuff` itself.

Essentially, just remember that `case` clauses take identifiers if you need to
match an object or iterable.
