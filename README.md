# Bind-`this` operator for JavaScript
ECMAScript Stage-0 Proposal. J. S. Choi, 2021.

* **[Formal specification][]**
* Babel plugin: Not yet

[formal specification]: http://jschoi.org/21/es-bind-operator/

## Description
(A [formal specification][] is available.)

**Method binding** `->` is a **left-associative infix operator**.
Its right-hand side is an **identifier** (like `f`)
or a parenthesized **expression** (like `(hof())`),
either of which must evaluate to a **function**.
Its left-hand side is some expression that evaluates to an **object**.
The `->` operator **binds** its left-hand side
to its right-hand side’s `this` value,
creating a **bound function** in the same manner
as [`Function.prototype.bind`][bind].

For example, `arr->fn` would be roughly
equivalent to `fn.bind(arr)`,
except that its behavior does **not change**
if code elsewhere **reassigns** the **global method** `Function.prototype.bind`.

Likewise, `obj->(createMethod())` would be roughly
equivalent to `createMethod().bind(obj)`.

If the operator’s right-hand side does not evaluate to a function during runtime,
then the program throws a `TypeError`.

The bound functions created by the bind-`this` operator
are **indistinguishable** from the bound functions
that are already created by [`Function.prototype.bind`][bind].
Both are **exotic objects** that do not have a `prototype` property,
and which may be called like any typical function.

From this definition, `o->f(...args)`
is **indistinguishable** from `Function.prototype.call(f, o, ...args)`,
except that its behavior does **not change**
if code elsewhere **reassigns** the global method `Function.prototype.call`.

The `this`-bind operator has equal **[precedence][]** with
**member expressions**, call expressions, `new` expressions with arguments,
and optional chains.
Like those operators, the `this`-bind operator also may be short-circuited
by optional chains in its left-hand side.

[precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

| Left-hand side                     | Example      | Grouping
| ---------------------------------- | ------------ | --------------
| Member expressions                 |`a.b->fn.c`   |`((a.b)->fn).c`
| Call expressions                   |`a()->fn()`   |`((a())->fn)()`
| Optional chains                    |`a?.b->fn`    |`(a?.b)->fn`
|`new` expressions with arguments    |`new C(a)->fn`|`(new C(a))->fn`
|`new` expressions without arguments*|`new a->fn`   |`new (a->fn)`

\* Like `.` and `?.`, the `this`-bind operator also have tighter precedence
than `new` expressions without arguments.
Of course, `new a->fn` is not a very useful expression,
just like how `new (fn.bind(a))` is not a very useful expression.

Similarly to the `.` and `?.` operators,
the `->` operator may be **padded by whitespace**.\
For example, `a -> m`\
is equivalent to `a->fn`,\
and `a -> (createFn())`\
is equivalent to `a->(createFn())`.

There are **no other special rules**.

## Why a bind-`this` operator
[`Function.prototype.bind`][call] and [`Function.prototype.call`][bind]
are very common in **object-oriented JavaScript** code.
They are useful methods that allows us to apply functions to any object,
binding their first arguments to the `this` bindings within those functions,
no matter the current object environment.
`bind` and `call` allow us to **extend** an **object** with a function
as if that function were **its own method**.
They serve as an important link between
the **object-oriented** and **functional** styles in JavaScript.

[bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
[call]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call

Why then would we need an operator that does the same thing?
Because `bind` and `call` are vulnerable to **global mutation**.

For example, when we run our code in an untrusted environment,
an adversary may mutate global prototype objects
such as `Array.prototype`,
reassigning or deleting their methods.

```js
// The adversary’s code.
delete Array.prototype.slice;

// Our own trusted code, running later.
// Due to the adversary, this unexpectedly throws an error.
[0, 1, 2].slice(1, 2);
```

In order to harden JavaScript applications against this attack,
we can extract critical global prototype methods into variables
before any untrusted code may run.
We would then use our critical methods with their `call` methods.

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;

// Our own trusted code, running later.
// In spite of the adversary, this no longer throws an error.
slice.call([0, 1, 2], 1, 2);
```

But this approach is still vulnerable to mutation of `Function.prototype`:

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;
delete Function.prototype.call;

// Our own trusted code, running later.
// Due to the adversary, this throws an error again.
slice.call([0, 1, 2], 1, 2);
```

There is currently no way to harden code against mutation of `Function.prototype`.
A new operator, however, would not be vulnerable to mutation:

```js
// Our own trusted code, running before any adversary.
const { slice } = Array.prototype;

// The adversary’s code.
delete Array.prototype.slice;
delete Function.prototype.call;

// Our own trusted code, running later.
// In spite of the adversary, this no longer throws an error.
// It also is considerably more readable.
[0, 1, 2]->slice(1, 2);
```

As a bonus, the syntax is also considerably more readable
than code that uses `bind` or `call`.
A bound function could be called inline
as if it were **actually a method** in the object
whose **property key** is the **function itself**:

```js
function extensionMethod () {
  return this;
}

obj.actualMethod();
obj->extensionMethod();
// Compare with extensionMethod.call(obj).
```

## Real-world examples
Only minor formatting changes have been made to the status-quo examples.

### Node.js
Node.js’s runtime depends on many built-in JavaScript global intrinsic objects
that are vulnerable to mutation or prototype pollution by third-party libraries.
When initializing a JavaScript runtime, Node.js therefore caches
wrapped versions of every global intrinsic object (and its methods)
in a [large `primordials` object][primordials.js].

Many of the global intrinsic methods inside of the `primordials` object
rely on the `this` binding.
`primordials` therefore contains numerous entries that look like this:
```js
ArrayPrototypeConcat: uncurryThis(Array.prototype.concat),
ArrayPrototypeCopyWithin: uncurryThis(Array.prototype.copyWithin),
ArrayPrototypeFill: uncurryThis(Array.prototype.fill),
ArrayPrototypeFind: uncurryThis(Array.prototype.find),
ArrayPrototypeFindIndex: uncurryThis(Array.prototype.findIndex),
ArrayPrototypeLastIndexOf: uncurryThis(Array.prototype.lastIndexOf),
ArrayPrototypePop: uncurryThis(Array.prototype.pop),
ArrayPrototypePush: uncurryThis(Array.prototype.push),
ArrayPrototypePushApply: UncurryThisStaticApply<typeof Array.prototype.push>
ArrayPrototypeReverse: uncurryThis(Array.prototype.reverse),
```
…and so on, where `uncurryThis` is `Function.bind.bind(Function.call)`
(also called [“call-binding”][call-bind]).

[call-bind]: https://npmjs.com/call-bind

In other words, Node.js must **wrap** every `this`-sensitive global intrinsic method
in a `this`-uncurried **wrapper function**,
whose first argument is the method’s `this` value,
using the `uncurryThis` helper function.

The result is that code that uses these global intrinsic methods,
like this code adapted from [node/lib/internal/v8_prof_processor.js][]:
```js
  // specifier is a string.
  const file = specifier.slice(2, -4);
```
…
```js
  if (process.platform === 'darwin') {
    tickArguments.push('--mac');
  } else if (process.platform === 'win32') {
    tickArguments.push('--windows');
  }
  tickArguments.push(...process.argv.slice(1));
```
…must instead look like this:
```js
// Note: This module assumes that it runs before any third-party code.
const {
  ArrayPrototypePush,
  ArrayPrototypeSlice,
  StringPrototypeSlice,
} = primordials;
```
…
```js
  const file = StringPrototypeSlice(specifier, 2, -4);
```
…
```js
  if (process.platform === 'darwin') {
    ArrayPrototypePush(tickArguments, '--mac');
  } else if (process.platform === 'win32') {
    ArrayPrototypePush(tickArguments, '--windows');
  }
  ArrayPrototypePush(tickArguments,
                     ...ArrayPrototypeSlice(process.argv, 1));
```

This code is now protected against prototype pollution by accident and by adversaries
(e.g., `delete Array.prototype.push`).
However, this protection comes at two costs:

1. These [uncurried wrapper functions sometimes dramatically reduce performance][#38248].
   This would not be a problem if Node.js could cache
   and use the intrinsic methods directly.
   But the only current way to use intrinsic methods
   would be with `Function.prototype.call`, which is also vulnerable to mutation.

2. The Node.js community has had [much concern about barriers to contribution][#30697]
   by ordinary JavaScript developers, due to the unidiomatic code encouraged by these
   uncurried wrapper functions.

Both of these problems are much improved by the bind-`this` operator.
Instead of wrapping every global method with `uncurryThis`,
Node.js could cached and used **directly**
without worrying about `Function.prototype.call` mutation:

```js
// Note: This module assumes that it runs before any third-party code.
const $push = Array.prototype.push;
const $arraySlice = Array.prototype.slice;
const $stringSlice = String.prototype.slice;
```
…
```js
  const file = specifier->stringSlice(2, -4);
```
…
```js
  if (process.platform === 'darwin') {
    tickArguments->push('--mac');
  } else if (process.platform === 'win32') {
    tickArguments->push('--windows');
  }
  tickArguments->push(...process.argv->slice(1));
```

Performance has improved, and readability has improved.
There are no more uncurried wrapper functions;
instead, the code uses the intrinsic methods in a notation
similar to normal method calling with `.`.

[node/lib/internal/v8_prof_processor.js]: https://github.com/nodejs/node/blob/e46c680bf2b211bbd52cf959ca17ee98c7f657f5/lib/internal/v8_prof_processor.js
[#38248]: https://github.com/nodejs/node/pull/38248
[#30697]: https://github.com/nodejs/node/issues/30697

## Non-goals
A goal of this proposal is **simplicity**.
Therefore, this proposal purposefully
does *not* address the following use cases.

**Tacit method extraction** with another operator
(like `arr&.slice` for `arr.slice.bind(arr.slice)` hypothetically)
would be **nice to have**,
but method extraction is **already possible** with this proposal.\
`const slice = arr->(arr.slice); slice(1, 3);`\
is not much wordier than\
`const slice = arr&.slice; slice(1, 3);`

**Extracting property accessors** (i.e., getters and setters)
is not a goal of this proposal.
Get/set accessors are **not like** methods. Methods are **values**.
Accessors themselves are **not values**;
they are functions that activate when getting or setting properties.
Getters/setters have to be extracted using `Object.getOwnPropertyDescriptor`;
they are not handled in a special way.
This verbosity may be considered to be desirable [syntactic salt][]:
it makes the developer’s intention (to extract getters/setters – and not methods)
more explicit.

```js
const { get: $getSize } =
  Object.getOwnPropertyDescriptor(
    Set.prototype, 'size');

// The adversary’s code.
delete Set; delete Function;

// Our own trusted code, running later.
new Set([0, 1, 2])->$getSize();
```

**Function/expression application**,
in which **deeply nested** function calls and other expressions
are untangled into **linear pipelines**,
is important but not addressed by this proposal.
Instead, it is addressed by the **pipe operator**,
with which this proposal’s syntax **works well**.\
For example, we could untangle `h(await g(o->f(0, v)), 1)`\
into `v |> o->f(0, %) |> await g(%) |> h(%, 1)`.

[syntactic salt]: https://en.wikipedia.org/wiki/Syntactic_sugar#Syntactic_salt
[primordials.js]: https://github.com/nodejs/node/blob/master/lib/internal/per_context/primordials.js
