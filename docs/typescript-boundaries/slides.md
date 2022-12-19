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

Domain: Selling Aggricultural Machines!

1. User creates some machines in portal.
2. Portal POSTs to backend.
3. Backend persists in database.

----

**Question**
What's wrong with this? :thinking:

```ts
type Machine = {
    machineType: string;
    id: string;
    model: string;
    brand: string;
    enginePower?: number;
    productionYear?: number;
    isValidated: boolean;
    metadata: any;
}
```

----

```ts
type Machine = {
    // Should be generic and/or enum
    machineType: string;
    // `typeof id === typeof model`
    id: string;
    model: string;
    // Should be enum.
    brand: string;
    // `typeof enginePower === typeof productionYear`
    enginePower?: number;
    productionYear?: number;
    // should be invariant
    isValidated: boolean;
    // should be unknown or unknown record
    metadata: any;
}
```

----

- what you need to know
    - basic TypeScript syntax and types
    - unions (`|`) and intersections (`&`)

- we'll talk about
    - unknown vs any
    - widening / narrowing
        - type assertions
    - Record vs mapped object type
    - enums vs faux-nums
    - generics
    - branded types
    - conditional types
    - exercises

---

- part #2
    - variance
    - boundaries
    - parsing

----

## `unknown` vs `any`

----

## Don't use `any`

It disables type checking.

```ts
type Machine = {metadata: any;}

const machine: Machine = {metadata: {foo: 42}};

machine.metadata.map(a => a + 2)
machine.metadata.foo.toUpperCase()
machine.metadata.bar + 4;
machine.metadata.foo + 4;
```

----

## Prefer `unknown`

```ts
type Machine = {metadata: unknown};

const machine: Machine = {metadata: {foo: 42}};

// TypeError: Object is of type 'unknown'.
machine.metadata.map(a => a + 2);
machine.metadata.foo.toUpperCase();
machine.metadata.bar + 4;
machine.metadata.foo + 4;
```

----

## If you want to **use** it, **parse** it!

```ts
import {z} from 'zod';

const machineSchema = z.object({metadata: z.object({foo: z.number()})});

const machine = machineSchema.parse({metadata: {foo: 42}});
machine.metadata.foo + 4;
```

----

## **Widening** and **Narrowing**

----

## `Record` vs **mapped object type**

----

Static set of constants.

### **Enums** vs "**Faux-nums**"

----

## Native `enum`

```ts
enum NativeEnum {
    A = 'a',
    B = 'b',
}

// usage
type Foo = {prop: NativeEnum};
const foo: Foo = {prop: NativeEnum.B};
```

----

## "Faux-num"

```ts
const FauxNum = {
    A: 'a',
    B: 'b',
} as const; // <- important, otherwise ts widens 
            //    the type to string

// type FauxNum = "a" | "b"
type FauxNum = typeof FauxNum[keyof typeof FauxNum];

// usage
type Bar = {prop: FauxNum};
const bar: Bar = {prop: FauxNum.B};
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
    // Type '"A"' is not assignable to type 'NativeEnum'.
    const foo: NativeEnum = FauxNum.A;
    const bar: NativeEnum = 'A'
    ```

----

## Benefits of using **enum**

- efficient encoding of bitmasks
- easy to define
- maybe we get better support in the future

----

## Prefer **faux-num** over **enum**

- a bit fiddly, but ...
- faux-nums can do everything that `enum` can
- you already know how to work with objects

----

## Variance via **generics**

Dynamic inputs / output types.

----

## Generic Types

```ts
//           [---------------------------]      definition
//             [------------]                   constraint on T
//                            [---------]       default type
type Machine<T extends string = 'tractor'> = {
    machineType: T;
};
```

----

## Utility for **faux-num**

```ts
type ValuesOf<T extends Record<string, unknown>> = T[keyof T];

const FauxNum = {
    A: 'a',
    B: 'b',
} as const;

type FauxNum = ValuesOf<typeof FauxNum>;
```

----
## MachineType

```ts
const MachineType = {
    TRACTOR: 'tractor',
    COMBINE_HARVESTER: 'combine-harvester',
} as const;

// MachineType: 'tractor' | 'combine-harvester'
type MachineType = ValuesOf<typeof MachineType>;
```

----

## Union

```ts
type BaseMachine<T extends MachineType> = {machineType: T};

type CombineHarvester = BaseMachine<typeof MachineType.COMBINE_HARVESTER>;
type Tractor = BaseMachine<typeof MachineType.TRACTOR>;

type Machine = CombineHarvester | Tractor;

// usage
const combine: CombineHarvester = {machineType: MachineType.COMBINE_HARVESTER>};
const tractor: Machine = {machineType: MachineType.TRACTOR>};
```

----

Two `string`s can mean very different things.

## Differentiating Primitives With **Branded Types**

----

## **Ephemeral** branded `string`

```ts
type Tagged<T, K> = T & {_tag: K};

// UUID: string & {_tag: 'UUID'}
type UUID = Tagged<string, 'UUID'>;
```

----

```ts
import {v4 as _uuid} from 'uuid';

type Foo = {id: UUID};

const uuid = (): UUID => _uuid() as UUID;
//                                     ↑
// We never actually set `_tag`, it's a lie!

const works: Foo = {id: uuid()};
// TypeError: Type 'string' is not assignable to type 'UUID'.
const broken: Foo = {id: _uuid()};
```

----

## **Ephemeral** branded `number`

```ts
type HorsePower = Tagged<number, 'UNIT:HORSE_POWER'>;

const horsePower = (n: number): HorsePower => {
    if (Number.isInteger(n) && Number.isFinite(n) && n > 0) {
        return n as HorsePower;
    }

    throw new TypeError('number is invalid HorsePower');
};
```
----

```ts
type Machine = {
    id: UUID;
    enginePower: HorsePower;
};

const works: Machine = {
    id: uuid(),
    enginePower: horsePower(40),
};

const broken: Machine = {
    id: uuid(),
    // Type 'number' is not assignable to type 'HorsePower'.
    enginePower: 40,
};
```
----

### **Serializable** distributed failures

```ts
type Failure = MachineOutOfStock | CustomerInvalidRegion;

type MachineOutOfStock = Tagged<
    {machineId: MachineId},
    typeof tag.MACHINE_OUT_OF_STOCK
>;

type CustomerInvalidRegion = Tagged<
    {region: Region; customerId: CustomerId},
    typeof tag.CUSTOMER_INVALID_REGION
>;

const tag = {
    MACHINE_OUT_OF_STOCK: 'FAILURE:MACHINE_OUT_OF_STOCK',
    CUSTOMER_INVALID_REGION: 'FAILURE:CUSTOMER_INVALID_REGION',
} as const;
```

----

### **Serializable** distributed failures

```ts
const MachineOutOfStock = {
    create: (payload: Omit<MachineOutOfStock, '_tag'>): MachineOutOfStock => ({
        _tag: tag.MACHINE_OUT_OF_STOCK,
        ...payload,
    }),
    is: (failure: Failure): failure is MachineOutOfStock =>
        failure._tag === tag.MACHINE_OUT_OF_STOCK,
};
```

----

### Elegant error handling

```ts
async function fireTheMissles() {
    await pressTheButton().catch((e: Failure) => {
        switch (e._tag) {
            case tag.MACHINE_OUT_OF_STOCK: {
                // do stuff
            }
            case tag.CUSTOMER_INVALID_REGION: {
                // do other stuff
            }
        }
    });
}
```

----

## When To Use

|                  | use when you need   |
|-----------------:|:--------------------|
|       primitives | whatever            |
|    **Ephemeral** | constrained type    |
| **Serializable** | ^ and cheap parsing |

----

The integrity of `unknown`.
## **Parse**, don't *validate*.
_Alexis King_

----

**Question**
What is wrong with this? :thinking:

```ts
const head = <T>(xs: T[]): T => xs[0];
```

----

**Oops**

```ts
const head = <T>(xs: T[]): T => xs[0];

const xs: string[] = [];
// string
const s = head(xs);

// v---- runtime exception
// TypeError: Cannot read properties of undefined (reading 'toUpperCase')
s.toUpperCase();
```

----

**Fixed**
How do I get rid of the `undefined`, though?

```ts
const head = <T>(xs: T[]): T | undefined => xs[0];

const xs: string[] = [];
// string | undefined
const s = head(xs);

// Object is possibly 'undefined'.
s.toUpperCase();
```

----

## Validation sucks.

```ts
type Machine = {id: string};

// Let's assume this throws when no machines are found.
declare function getMachines(): Promise<Machine[]>;

async function main() {
    const machines = await getMachines();

    // Machine | undefined
    const first = head(machines);
    // ugh
    if (!first) {
        // do stuff
    }
}
```
----

## Parse once, done.

```ts
type NonEmptyArray<T> = [T, ...T[]];

const fromArray = <T>(xs: T[]): NonEmptyArray<T> => {
    if (xs.length > 0) return xs as NonEmptyArray<T>;
    throw new TypeError('Array is empty.');
};

const head = <T>(xs: NonEmptyArray<T>): T => xs[0];

declare function getMachines(): Promise<NonEmptyArray<Machine>>;

async function main() {
    const machines = await getMachines();

    // Machine
    const first = head(machines);
}
```

----

## Making **illegal** state **unrepresentable**.
_Yaron Minsky_

----

Silly: We know only `combine-harvester` can have `harvesterThingies`, and we know then it's not empty.

```ts
type Machine = {
    machineType: 'tractor' | 'combine-harvester';
    harvesterThingies?: string[];
};

declare const machine: Machine;
if (machine.harvesterThingies !== undefined) {
    const harvesterThingy = head(machine.harvesterThingies);
    if (harvesterThingy !== undefined) {
        // ... do stuff
    }
}
```

----

### Fixing it

```ts
type Machine = Tractor | CombineHarvester;

type Tractor = {
    machineType: 'tractor';
};

type CombineHarvester = {
    machineType: 'combine-harvester';
    harvesterThingies: NonEmptyArray<string>;
};

declare const machine: CombineHarvester;

// string;
const first = head(machine.harvesterThingies);
```

----

- Define **precise types**.
- Write parsers / smart-contructors.
- **Parse once** at the _boundary_ of your app, instead of validating all over the place.

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
https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/
Alexis King: Parse, don't validate
    https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
```

----
