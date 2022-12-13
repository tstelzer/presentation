---
class: invert
size: 16:9
---

<h2 style="text-shadow: 2px 2px 0 black, -1px -1px 0 black;">TypeScript Domain Modeling</h2>
<p style="font-weight: 500; text-shadow: 2px 2px 0 black, -1px -1px 0 black; font-weight: 500;">Unlocking the type system for your own sanity.</p>

![bg cover](./cover.png)

----

*Why Should I Care?*

**You aren't using TypeScript effectively!**

In this workshop you'll learn how encoding more business logic in the type system, you achieve (*some measure of*) **type-safety** and **reduce anxiety** from dealing with ambivalent state.

----

## Example

Domain: Tractors!

1. User creates some machines in portal.
2. Portal POSTs to backend.
3. Backend persists tractor in database.

----

## What's wrong with this? :thinking:

```ts
type Machine = {
    machineType: string;
    id: string;
    model: string;
    brand: string;
    enginePower?: number;
    productionYear?: number;
    isValidated: boolean;
}
```

----

```ts
type Machine = {
    // Should be generic or invariant
    machineType: string;
    // `typeof id === typeof model`
    id: string;
    model: string;
    // Should be enum.
    brand: string;
    // `typeof enginePower === typeof productionYear`
    enginePower?: number;
    productionYear?: number;
    // Found another invariant!
    isValidated: boolean;
}
```

- what about **boundary types** (db, dto, etc)?
----

## Talk

- The **Basics**. (enumerations, branded types)
- Types at the **boundaries**. (dto, database, i/o, etc)
- Making illegal states **unrepresentable**. (invariants)
- **Parse**, don't *validate*!

## Workshop

- Refactoring a dummy app

----

Using

### **Enums**

to get a static set of constants.

----

## Native `enum`

```ts
// real enum
enum Real {
    A = 'A',
    B = 'B',
}

// usage
type Foo = {prop: Real};
const foo: Foo = {prop: Real.B};
```

----

## "Faux-num"

```ts
// faux-num
const Faux = {
    A: 'A',
    B: 'B',
} as const; // <- important, otherwise ts widens 
            //    the type to string

// type Faux = "A" | "B"
type Faux = typeof Faux[keyof typeof Faux];

// usage
type Bar = {prop: Faux};
const bar: Bar = {prop: Faux.B};
```

----

## Problems with using **enum**

- different syntax (learning curve, etc)
- not JS, compile to complex stuff (IIFE)
- can't use them as class properties
- number enums are not type-safe
    ```ts
    enum Silly { 'A', 'B' }
    Silly[0]   // 'A'
    Silly[999] // undefined
    ```
- **enums are nominal**
    ```ts
    // Type '"A"' is not assignable to type 'Real'.
    const foo: Real = Faux.A;
    const bar: Real = 'A'
    ```

----

## Benefits of using **enum**

- efficient encoding of bitmasks
- easy to define
- ... that's it

----

## Prefer **faux-num** over **enum**

- faux-nums can do everything that `enum` can
- you already know how to work with objects
- runtime construct === type representation

----

## Variance via **generics**
enable dynamic inputs / output types.

----

## Generic Types

```ts
//           [---------------------------]      definition
//             [------------]                   constraint on T
//                            [---------]       default type
type Machine<T extends string = 'tractor'> = {
    machineType: T;
};

// Apply to get concrete types.
type CombineHarvester = Machine<'combine-harvester'>;

// Using the default here.
type Tractor = Machine;
```

----

## Utility for **faux-num**

```ts
type ValuesOf<T extends Record<string, unknown>> = T[keyof T];

const Faux = {
    A: 'A',
    B: 'B',
} as const;

type Faux = ValuesOf<typeof Faux>;
```

----

Encoding your state in the type system with
## **Invariance** and **unions**

----

Two `string`s can mean very different things.

## Differentiating Primitives With **Branded Types**

----

## Ephemeral branded `string`

```ts
import {v4 as _uuid} from 'uuid';

type Tagged<T, K> = T & {readonly _tag: K};

type UUID = Tagged<string, 'UUID'>;

// Note: We never actually set `_tag`, it's a lie!
const uuid = (): UUID => _uuid() as UUID;
```

----

## Ephemeral branded `number`

```ts
type Tagged<T, K> = T & {readonly _tag: K};

type HorsePower = Tagged<number, 'UNIT:HORSE_POWER'>;
const horsePower = (n: number): HorsePower => {
    if (Number.isInteger(n) && Number.isFinite(n) && n > 0 && n < 20_000) {
        return n as HorsePower;
    }

    throw new TypeError('number is invalid HorsePower');
};
```

----

## Reified branded `object`

```ts
type Machine = Tagged<{id: UUID; enginePower: HorsePower}, 'MACHINE'>;

const works: Machine = {
    // this time, we're actually setting the `_tag`
    _tag: 'MACHINE',
    id: uuid(),
    enginePower: horsePower(40),
};

// Property '_tag' is missing in type '...' but required in type '...'.
const broken: Machine = {
    // Type 'string' is not assignable to type 'UUID'.
    id: 'foo',
    // Type 'number' is not assignable to type 'HorsePower'.
    enginePower: 40,
};
```

----

|               | use when you need         |
|--------------:|:--------------------------|
|    primitives | whatever                  |
| **Ephemeral** | constrained type alias    |
|   **Reified** | ^ and cheap serialization |
|       `class` | state + behaviour         |

----

Stop using `any`.
## The power of `unknown`

----

effective domain modeling

- don't stop at the domain, model api/db/dto types too
- instead of optionality, make use of unions for state
    - Raw
    - Domain
    - DB
    - DTO
- Errors
- make it runtime parseable, if necessary (e.g. tag)
- branded / Opaque
    - more specific primitives (strings, dates, ints)
    - e.g. units
- use
    - different logic depending on type
    - passes a boundary

----

- what everyone has: "types" folder with arbitrary types
    - two questions
        - _when_ are those types?
        - do they encode all invariants?
        - are field types fine-grained enough?

----

## Further Information

```text
Scott Wlaschin: Domain Modeling Made Functional
    https://www.youtube.com/watch?v=2JB1_e5wZmU
    https://pragprog.com/titles/swdddf/domain-modeling-made-functional/

Alexis King: Parse, don't validate
    https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
```
