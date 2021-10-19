# memoize-one

A memoization library that only caches the result of the most recent arguments.

> Also [async version](https://github.com/microlinkhq/async-memoize-one).

[![Build Status](https://travis-ci.org/alexreardon/memoize-one.svg?branch=master)](https://travis-ci.org/alexreardon/memoize-one)
[![npm](https://img.shields.io/npm/v/memoize-one.svg)](https://www.npmjs.com/package/memoize-one)
![types](https://img.shields.io/badge/types-typescript%20%7C%20flow-blueviolet)
[![dependencies](https://david-dm.org/alexreardon/memoize-one.svg)](https://david-dm.org/alexreardon/memoize-one)
[![minzip](https://img.shields.io/bundlephobia/minzip/memoize-one.svg)](https://www.npmjs.com/package/memoize-one)
[![Downloads per month](https://img.shields.io/npm/dm/memoize-one.svg)](https://www.npmjs.com/package/memoize-one)

## Rationale

Unlike other memoization libraries, `memoize-one` only remembers the latest arguments and result. No need to worry about cache busting mechanisms such as `maxAge`, `maxSize`, `exclusions` and so on, which can be prone to memory leaks. A function memoized with `memoize-one` simply remembers the last arguments, and if the memoized function is next called with the same arguments then it returns the previous result.

## Usage

```js
// memoize-one uses the default import
import memoizeOne from 'memoize-one';

function add(a, b) {
  return a + b;
}
const memoizedAdd = memoizeOne(add);

memoizedAdd(1, 2);
// add function: is called
// [new value returned: 3]

memoizedAdd(1, 2);
// add function: not called
// [cached result is returned: 3]

memoizedAdd(2, 3);
// add function: is called
// [new value returned: 5]

memoizedAdd(2, 3);
// add function: not called
// [cached result is returned: 5]

memoizedAdd(1, 2);
// add function: is called
// [new value returned: 3]
// 👇
// While the result of `add(1, 2)` was previously cached
// `(1, 2)` was not the *latest* arguments (the last call was `(2, 3)`)
// so the previous cached result of `(1, 3)` was lost
```

## Installation

```bash
# yarn
yarn add memoize-one

# npm
npm install memoize-one --save
```

## Function argument equality

By default, we apply our own _fast_ and _relatively naive_ equality function to determine whether the arguments provided to your function are equal. You can see the full code here: [are-inputs-equal.ts](https://github.com/alexreardon/memoize-one/blob/master/src/are-inputs-equal.ts).

(By default) function arguments are considered equal if:

1. there is same amount of arguments
2. each new argument has strict equality (`===`) with the previous argument
3. **[special case]** if two arguments are not `===` and they are both `NaN` then the two arguments are treated as equal

What this looks like in practice:

```js
import memoizeOne from 'memoize-one';

// add all numbers provided to the function
const add = (...args = []) =>
  args.reduce((current, value) => {
    return current + value;
  }, 0);
const memoizedAdd = memoizeOne(add);
```

> 1. there is same amount of arguments

```js
memoizedAdd(1, 2);
// the amount of arguments has changed, so underlying add function is called
memoizedAdd(1, 2, 3);
```

> 2. new arguments have strict equality (`===`) with the previous argument

```js
memoizedAdd(1, 2);
// each argument is `===` to the last argument, so cache is used
memoizedAdd(1, 2);
// second argument has changed, so add function is called again
memoizedAdd(1, 3);
// the first value is not `===` to the previous first value (1 !== 3)
// so add function is called again
memoizedAdd(3, 1);
```

> 3. **[special case]** if the arguments are not `===` and they are both `NaN` then the argument is treated as equal

```js
memoizedAdd(NaN);
// Even though NaN !== NaN these arguments are
// treated as equal as they are both `NaN`
memoizedAdd(NaN);
```

## Custom equality function

You can also pass in a custom function for checking the equality of two sets of arguments

```js
const memoized = memoizeOne(fn, isEqual);
```

An equality function should return `true` if the arguments are equal. If `true` is returned then the wrapped function will not be called.

**Tip**: A custom equality function needs to compare `Arrays`. The `newArgs` array will be a new reference every time so a simple `newArgs === lastArgs` will always return `false`.

Equality functions are not called if the `this` context of the function has changed (see below).

Here is an example that uses a [lodash.isEqual](https://lodash.com/docs/4.17.15#isEqual) deep equal equality check

> `lodash.isequal` correctly handles deep comparing two arrays

```js
import memoizeOne from 'memoize-one';
import isDeepEqual from 'lodash.isequal';

const identity = (x) => x;

const shallowMemoized = memoizeOne(identity);
const deepMemoized = memoizeOne(identity, isDeepEqual);

const result1 = shallowMemoized({ foo: 'bar' });
const result2 = shallowMemoized({ foo: 'bar' });

result1 === result2; // false - different object reference

const result3 = deepMemoized({ foo: 'bar' });
const result4 = deepMemoized({ foo: 'bar' });

result3 === result4; // true - arguments are deep equal
```

The equality function needs to conform to the `EqualityFn` `type`:

```ts
// TFunc is the function being memoized
type EqualityFn<TFunc extends (...args: any[]) => any> = (
  newArgs: Parameters<TFunc>,
  lastArgs: Parameters<TFunc>,
) => boolean;

// You can import this type
import type { EqualityFn } from 'memoize-one';
```

The `EqualityFn` type allows you to create equality functions that are extremely typesafe. You are welcome to provide your own less type safe equality functions.

Here are some examples of equality functions which are ordered by most type safe, to least type safe:

<details>
  <summary>Example equality function types</summary>
  <p>

```ts
// the function we are going to memoize
function add(first: number, second: number): number {
  return first + second;
}

// Some options for our equality function
// ↑ stronger types
// ↓ weaker types

// ✅ exact parameters of `add`
{
  const isEqual = function (first: Parameters<typeof add>, second: Parameters<typeof add>) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ tuple of the correct types
{
  const isEqual = function (first: [number, number], second: [number, number]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ❌ tuple of incorrect types
{
  const isEqual = function (first: [number, string], second: [number, number]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().not.toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ array of the correct types
{
  const isEqual = function (first: number[], second: number[]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ❌ array of incorrect types
{
  const isEqual = function (first: string[], second: number[]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().not.toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ tuple of 'unknown'
{
  const isEqual = function (first: [unknown, unknown], second: [unknown, unknown]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ❌ tuple of 'unknown' of incorrect length
{
  const isEqual = function (first: [unknown, unknown, unknown], second: [unknown, unknown]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().not.toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ array of 'unknown'
{
  const isEqual = function (first: unknown[], second: unknown[]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ spread of 'unknown'
{
  const isEqual = function (...first: unknown[]) {
    return !!first;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ tuple of 'any'
{
  const isEqual = function (first: [any, any], second: [any, any]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ❌ tuple of 'any' or incorrect size
{
  const isEqual = function (first: [any, any, any], second: [any, any]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().not.toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ array of 'any'
{
  const isEqual = function (first: any[], second: any[]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ two arguments of type any
{
  const isEqual = function (first: any, second: any) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ a single argument of type any
{
  const isEqual = function (first: any) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}

// ✅ spread of any type
{
  const isEqual = function (...first: any[]) {
    return true;
  };
  expectTypeOf<typeof isEqual>().toMatchTypeOf<EqualityFn<typeof add>>();
}
```

  </p>
</details>

## `this`

### `memoize-one` correctly respects `this` control

This library takes special care to maintain, and allow control over the the `this` context for **both** the original function being memoized as well as the returned memoized function. Both the original function and the memoized function's `this` context respect [all the `this` controlling techniques](https://github.com/getify/You-Dont-Know-JS/blob/master/this%20%26%20object%20prototypes/ch2.md):

- new bindings (`new`)
- explicit binding (`call`, `apply`, `bind`);
- implicit binding (call site: `obj.foo()`);
- default binding (`window` or `undefined` in `strict mode`);
- fat arrow binding (binding to lexical `this`)
- ignored this (pass `null` as `this` to explicit binding)

### Changes to `this` is considered an argument change

Changes to the running context (`this`) of a function can result in the function returning a different value even though its arguments have stayed the same:

```js
function getA() {
  return this.a;
}

const temp1 = {
  a: 20,
};
const temp2 = {
  a: 30,
};

getA.call(temp1); // 20
getA.call(temp2); // 30
```

Therefore, in order to prevent against unexpected results, `memoize-one` takes into account the current execution context (`this`) of the memoized function. If `this` is different to the previous invocation then it is considered a change in argument. [further discussion](https://github.com/alexreardon/memoize-one/issues/3).

Generally this will be of no impact if you are not explicity controlling the `this` context of functions you want to memoize with [explicit binding](https://github.com/getify/You-Dont-Know-JS/blob/master/this%20%26%20object%20prototypes/ch2.md#explicit-binding) or [implicit binding](https://github.com/getify/You-Dont-Know-JS/blob/master/this%20%26%20object%20prototypes/ch2.md#implicit-binding). `memoize-One` will detect when you are manipulating `this` and will then consider the `this` context as an argument. If `this` changes, it will re-execute the original function even if the arguments have not changed.

## Clearing the memoization cache

A `.clear()` property is added to memoized functions to allow you to clear it's memoization cache.

This is helpful if you want to:

- Release memory
- Allow the underlying function to be called again without having to change arguments

```ts
import memoizeOne from 'memoize-one';

function add(a: number, b: number): number {
  return a + b;
}

const memoizedAdd = memoizeOne(add);

// first call - not memoized
const first = memoizedAdd(1, 2);

// second call - cache hit (underlying function not called)
const second = memoizedAdd(1, 2);

// 👋 clearing memoization cache
memoizedAdd.clear();

// third call - not memoized (cache was cleared)
const third = memoizedAdd(1, 2);
```

## When your result function `throw`s

> There is no caching when your result function throws

If your result function `throw`s then the memoized function will also throw. The throw will not break the memoized functions existing argument cache. It means the memoized function will pretend like it was never called with arguments that made it `throw`.

```js
const canThrow = (name: string) => {
  console.log('called');
  if (name === 'throw') {
    throw new Error(name);
  }
  return { name };
};

const memoized = memoizeOne(canThrow);

const value1 = memoized('Alex');
// console.log => 'called'
const value2 = memoized('Alex');
// result function not called

console.log(value1 === value2);
// console.log => true

try {
  memoized('throw');
  // console.log => 'called'
} catch (e) {
  firstError = e;
}

try {
  memoized('throw');
  // console.log => 'called'
  // the result function was called again even though it was called twice
  // with the 'throw' string
} catch (e) {
  secondError = e;
}

console.log(firstError !== secondError);

const value3 = memoized('Alex');
// result function not called as the original memoization cache has not been busted
console.log(value1 === value3);
// console.log => true
```

## Function properties

Functions memoized with `memoize-one` do not preserve any properties on the function object.

> This behaviour correctly reflected in the TypeScript types

```ts
import memoizeOne from 'memoize-one';

function add(a, b) {
  return a + b;
}
add.hello = 'hi';

console.log(typeof add.hello); // string

const memoized = memoizeOne(add);

// hello property on the `add` was not preserved
console.log(typeof memoized.hello); // undefined
```

If you feel strongly that `memoize-one` _should_ preserve function properties, please raise an issue. This decision was made in order to keep `memoize-one` as light as possible.

For _now_, the `.length` property of a function is not preserved on the memoized function

```ts
import memoizeOne from 'memoize-one';

function add(a, b) {
  return a + b;
}

console.log(add.length); // 2

const memoized = memoizeOne(add);

console.log(memoized.length); // 0
```

There is no (great) way to correctly set the `.length` property of the memoized function while also supporting ie11. Once we [remove ie11 support](https://github.com/alexreardon/memoize-one/issues/125) then we will set the `.length` property of the memoized function to match the original function

[→ discussion](https://github.com/alexreardon/memoize-one/pull/124).

## Memoized function `type`

The resulting function you get back from `memoize-one` has *almost* the same `type` as the function that you are memoizing

```ts
declare type MemoizedFn<TFunc extends (this: any, ...args: any[]) => any> = {
  clear: () => void;
  (this: ThisParameterType<TFunc>, ...args: Parameters<TFunc>): ReturnType<TFunc>;
};
```

- the same call signature as the function being memoized
- a `.clear()` function property added
- other function object properties on `TFunc` as not carried over

You are welcome to use the `MemoizedFn` generic directly from `memoize-one` if you like:

```ts
import memoize, { MemoizedFn } from 'memoize-one';
import isDeepEqual from 'lodash.isequal';
import { expectTypeOf } from 'expect-type';

// Takes any function: TFunc, and returns a Memoized<TFunc>
function withDeepEqual<TFunc extends (...args: any[]) => any>(fn: TFunc): MemoizedFn<TFunc> {
  return memoize(fn, isDeepEqual);
}

function add(first: number, second: number): number {
  return first + second;
}

const memoized = withDeepEqual(add);

expectTypeOf<typeof memoized>().toEqualTypeOf<MemoizedFn<typeof add>>();
```

In this specific example, this type would have been correctly inferred too

```ts
import memoize, { MemoizedFn } from 'memoize-one';
import isDeepEqual from 'lodash.isequal';
import { expectTypeOf } from 'expect-type';

// return type of MemoizedFn<TFunc> is inferred
function withDeepEqual<TFunc extends (...args: any[]) => any>(fn: TFunc) {
  return memoize(fn, isDeepEqual);
}

function add(first: number, second: number): number {
  return first + second;
}

const memoized = withDeepEqual(add);

// type test still passes
expectTypeOf<typeof memoized>().toEqualTypeOf<MemoizedFn<typeof add>>();
```

## Performance 🚀

### Tiny

`memoize-one` is super lightweight at [![min](https://img.shields.io/bundlephobia/min/memoize-one.svg?label=)](https://www.npmjs.com/package/memoize-one) minified and [![minzip](https://img.shields.io/bundlephobia/minzip/memoize-one.svg?label=)](https://www.npmjs.com/package/memoize-one) gzipped. (`1KB` = `1,024 Bytes`)

### Extremely fast

`memoize-one` performs better or on par with than other popular memoization libraries for the purpose of remembering the latest invocation.

The comparisons are not exhaustive and are primarily to show that `memoize-one` accomplishes remembering the latest invocation really fast. There is variability between runs. The benchmarks do not take into account the differences in feature sets, library sizes, parse time, and so on.

<details>
  <summary>Expand for results</summary>
  <p>

node version `16.11.1`

You can run this test in the repo by:

1. Add `"type": "module"` to the `package.json` (why is things so hard)
2. Run `yarn perf:library-comparison`

**no arguments**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | memoize-one                   | 80,384,605            |
| 2        | moize                         | 72,852,842            |
| 3        | memoizee                      | 35,743,907            |
| 4        | lodash.memoize                | 31,473,156            |
| 5        | mem (JSON.stringify strategy) | 4,510,714             |
| 6        | no memoization                | 505                   |
| 7        | fast-memoize                  | 505                   |

**single primitive argument**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | fast-memoize                  | 45,888,796            |
| 2        | moize                         | 34,577,180            |
| 3        | lodash.memoize                | 30,123,904            |
| 4        | memoize-one                   | 28,516,529            |
| 5        | memoizee                      | 20,230,305            |
| 6        | mem (JSON.stringify strategy) | 3,874,762             |
| 7        | no memoization                | 504                   |

**single complex argument**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | lodash.memoize                | 28,862,391            |
| 2        | moize                         | 27,275,762            |
| 3        | memoize-one                   | 26,613,077            |
| 4        | memoizee                      | 17,302,985            |
| 5        | mem (JSON.stringify strategy) | 2,123,364             |
| 6        | fast-memoize                  | 1,576,180             |
| 7        | no memoization                | 504                   |

**multiple primitive arguments**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | lodash.memoize                | 25,169,091            |
| 2        | moize                         | 21,501,957            |
| 3        | memoize-one                   | 17,269,441            |
| 4        | memoizee                      | 9,728,831             |
| 5        | mem (JSON.stringify strategy) | 3,037,060             |
| 6        | fast-memoize                  | 1,209,636             |
| 7        | no memoization                | 505                   |

**multiple complex arguments**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | lodash.memoize                | 24,606,356            |
| 2        | moize                         | 21,348,419            |
| 3        | memoize-one                   | 17,633,659            |
| 4        | memoizee                      | 9,455,600             |
| 5        | mem (JSON.stringify strategy) | 797,349               |
| 6        | fast-memoize                  | 676,376               |
| 7        | no memoization                | 504                   |

**multiple complex arguments (spreading arguments)**

| Position | Library                       | Operations per second |
| -------- | ----------------------------- | --------------------- |
| 1        | lodash.memoize                | 22,626,554            |
| 2        | moize                         | 19,081,480            |
| 3        | memoizee                      | 17,748,567            |
| 4        | memoize-one                   | 16,108,391            |
| 5        | mem (JSON.stringify strategy) | 786,317               |
| 6        | fast-memoize                  | 557,983               |
| 7        | no memoization                | 504                   |

  </p>
</details>

## Code health 👍

- Tested with all built in [JavaScript types](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/types%20%26%20grammar/ch1.md)
- Written in `Typescript`
- Correct typing for `Typescript` and `flow` type systems
- No dependencies
