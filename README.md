# Proposed New Approach for Pattern Matching in ECMAScript
Stage 0. Championed by Daniel Ethridge ([@wlib](https://git.io/de)).
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

## Proposal
So our solution to pattern matching should incorporate both destructuring and
`instanceof` in order to be maximally useful. It can be understood as just a
newer `switch` statement that allows matching if the value...

* ...is strictly equal (`===`) to a case, like a normal `switch` would.
* ...can be destructured by a case
* ...is an `instanceof` a case

Finally, our solution must abandon the current `switch` syntax in order to be
more consistent with the syntax of all of the other statements.

Here is roughly what its syntax looks like:

```
switch* (<expression>) {
  case (<expression>) <block>
  ...
  case (<destructuring>) <block>
  ...
  instanceof (<constructor>) <block>
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
```

But of course - the most common mistake is to forget to write all of the `break`'s.

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

Now, the clauses within `switch*` can do the same, making them much more
consistent with intuition for how pattern matching *should* work:

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
const bTreeSize = tree => {
  switch* (tree) {
    instanceof (Leaf)
      return 1

    instanceof (Node)
      return bTreeSize(Node.left) + bTreeSize(Node.right)
  }
}

const apply = (f, maybeSomething) => {
  switch* (maybeSomething) {
    instanceof (Just)
      return f(maybeSomething.value)

    instanceof (Nothing)
      return maybeSomething

    default
      throw new TypeError(maybeSomething, "needs to be a Maybe type")
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
clause matches the expression `[1]`.

Essentially, just remember that `case` clauses take identifiers if you need to
match an object or iterable.
