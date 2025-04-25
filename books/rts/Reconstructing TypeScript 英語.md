[[0-rt-intro]]
[[Reconstructing TypeScript 補足]]
# Reconstructing TypeScript, part 0: intro and background

### *2021-09-07*

I've been building a "document development environment" called [Programmable Matter](https://github.com/jaked/programmable-matter) that supports live code embedded in documents, with a simple TypeScript-like programming language. It's been fun figuring out how to implement it—the type system in [TypeScript](https://www.typescriptlang.org/) is unusual and very cool!

I want to dig into what's cool and unusual about TypeScript by presenting a type checker for a fragment of this language (written in actual TypeScript). I'll start with a tiny fragment and build it up over several posts. In this first post I'll give some background to help make sense of the code.

## What's a type checker?

I'm going to assume that you've used a type checker, and have an idea what "type" and "type checking" mean. But I want to unpack these concepts a little:

In JavaScript, a variable can hold values of different types. Suppose we don't know what type of value `x` holds—maybe we know it's a `string` or a `boolean` but not which one. If we test the type

```js
if (typeof x === 'string') {
  ...
} else {
  ...
}
```

then we can reason (at development time) that if the test succeeds (at run time) the value must have type `string`; so we can safely treat `x` as a `string` in the true branch (and a `boolean` in the false branch).

As programmers, reasoning about the behavior of programs is our main job! We mostly do it in our heads, but it's really useful to automate it—to catch mistakes, and to support interactive development features like code completion. This is where a type checker comes in: it's a way to reason automatically at development time about the behavior of programs at run time.

Our human reasoning can be arbitrarily creative and complex, but a type checker is just a program, so its "reasoning" is limited. A type checker can't reason that a program correctly sorts a list, for example; but it can check that it doesn't attempt any unsupported operations on values (such as accessing a property that doesn't exist).

Since a type checker runs at development time, it can't know the actual values flowing through a program at run time. Instead, for each expression in the program, it lumps together the values that might be computed by the expression, and gives the expression a *type* that describes attributes shared by all the values: what operations are supported; and, for some operations, what result they return.

For example, in

```
const vec = { x: x, y: y };
```

if `x` and `y` have type `number`, then `vec` has type

```
{ x: number, y: number }
```

As the program runs, `x` and `y` may take on many different values, so `vec` may also take on many values. But all values of `vec` support accessing the `x` and `y` properties; and for all such values the result of calling `typeof` on them is `"object"`.

Now, if the program contains an expression `vec.z`, the type checker sees from the type of `vec` that accessing the `z` property isn't a supported operation, so it flags an error. A type checker ensures that a program doesn't attempt any unsupported operations on *concrete values at run time*, by checking that it doesn't attempt any unsupported operations on *expression types at development time*.

## What's cool about TypeScript?

In most type systems, primitive types can't be mixed, so a variable can't hold either `string` or `boolean`. But there's usually a way to mix certain compound types and test the type of a value. In a language with classes and objects, a variable of class `Shape` can hold objects of subclasses `Circle` or `Square`, and we can test the class of the object with `instanceof` (or some equivalent). Or in a language with variants (aka sums or tagged unions), a variable of type `tree` can hold values of variant arms `Leaf` or `Node`, and we can learn what arm a value is by pattern-matching.

In TypeScript, any types can be mixed. If we know that `x` holds a `string` or a `boolean` we can give it a *union* type, `string | boolean`. When we test the type in the example above, the type checker *narrows* the type of `x` in the branches of the `if` / `else` according to the result of the test: in the true branch, `x` has type `string`; in the false branch, `boolean`.

This idea goes pretty far—for example, we can define a variant-like tree type as a union of leaf and node object types:

```
type tree = { value: number } | { left: tree, right: tree }
```

with values like:

```
const tree = {
  left: {
    left: { value: 7 },
    right: { value: 9 }
  },
  right: { value: 11 }
}
```

If we check the presence of the `value` property, TypeScript narrows the type in each branch, and reasons that it's safe to use the `left` and `right` properties in the false branch:

```
function height(t: tree): number {
  if ('value' in t) return 1;
  else return 1 + Math.max(height(t.left), height(t.right));
}
```

I like programming with union types and narrowing a lot. They make it possible to get useful checking of typical JavaScript idioms that depend on run-time type tests, so it's straightforward to translate most JavaScript code to TypeScript. And unions are an appealing alternative to variant types or class hierarchies, because they're simpler and more flexible. (I'll give some examples along the way to justify this claim.)

## Aside: unions vs. variants

If you're familiar with variants, as found in Haskell or OCaml, you might wonder how union types are different. When we define a variant (in OCaml):

```
type tree =
  Leaf of { value: int } |
  Node of { left: tree; right: tree }
```

we get several things:

- a type `tree`—a value of this type is a `Leaf` or `Node` but we don't know which one;

- a constructor to make values of each arm—the constructors wrap underlying values with a tag, so the implementation can tell which arm it is; and

- support for pattern-matching on values of the type—the implementation uses the tag on a value to tell which arm it is.

We write `tree` values with explicit constructors:

```
let tree = Node {
  left = Node {
    left = Leaf { value = 7 };
    right = Leaf { value = 9 }
  };
  right = Leaf { value = 11 }
}
```

and discriminate the arms with pattern-matching:

```
let rec height (tree: tree) =
  match tree with
    | Leaf { value } -> 1
    | Node { left; right } -> 1 + max (height left) (height right)
```

In TypeScript, when we define a union type we get just the type. As in the example in the previous section, to construct values of the union we write down ordinary values that satisfy the type of one of the arms, rather than using type-specific constructors. To discriminate the arms we use ordinary test operators and narrowing, rather than type-specific pattern-matching.

Because union types require no new expression syntax, they're flexible and light-weight: for example, if we want a function argument that's either `boolean` or `string`, we can write `boolean | string` directly in the argument type, without declaring a type outside the function or requiring callers to wrap the argument in a constructor.

## Reconstructing TypeScript

I have to confess—I don't know how actual TypeScript works! There isn't an [up-to-date specification](https://github.com/Microsoft/TypeScript/issues/15711) of how it's supposed to work, and I haven't tried to read the implementation. Also, the language in Programmable Matter fills a pretty different niche from actual TypeScript, so I've chosen to diverge from it in several ways.

What I'll present here is a reconstruction of TypeScript, based on TypeScript's informal documentation, experimenting with the actual TypeScript implementation, research papers about related systems, background knowledge about implementing type checkers, and my own opinions on how it should work. (I'll point out interesting differences between this reconstruction and actual TypeScript along the way.)

In the rest of the post I want to explain at a high level how the type checker works. Next time I'll dig into the actual code.

## Aside: Hindley-Milner type inference

You might have heard of *Hindley-Milner type inference*, an approach to type checking that's used in Haskell, OCaml, and others. What I'll cover here is **not** Hindley-Milner, but a different approach called *bidirectional type checking*, which is used in TypeScript, Scala, and others.

Why not Hindley-Milner? Actual TypeScript implements bidirectional type checking; bidirectional type checking is simpler to implement; and Hindley-Milner doesn't combine easily with subtyping and union types, so it isn't a good fit for TypeScript.

## Synthesizing a type from an expression

A type checker needs to know the type of every expression in the program, so it can ensure that a program doesn't attempt any unsupported operations on expression types. How does it compute these types?

We can view a program as a tree (called an *abstract syntax tree*, or *AST*), where each expression is a node in the tree, with its subexpressions as children; the leaves are atomic expressions like literal values.

With this view in mind, we can compute the type of each expression bottom up, from the leaves of the tree to the root:

- for an atomic expression, return the corresponding type: `"foo"` has type `string`, `true` has type `boolean`, and so on; and

- for a compound expression, find the types of its subexpressions, then combine them according to the top-level operation of the expression.

For example, to compute the type of an expression

```
{ x: 7, y: 9 }
```

we see that the subexpressions `7` and `9` both have type `number`; so the overall object expression has type

```
{ x: number, y: number }
```

This process is called *synthesizing* a type *from* an expression.

## Subtyping

For some kinds of expression, we need to check that the types of the subexpressions agree in some sense. For example, in a function call we need to check that the type of the argument passed to the function is compatible with the argument type expected by the function. What does "compatible" mean?

The type of a function argument amounts to a claim (verified by the type checker) that the function will attempt only the operations described by the type on its argument. So the type of an expression passed to a function is compatible with the argument type when the passed type supports *at least* the operations described by the function argument type—it can also support other operations. For example, given a function type

```
(vec: { x: number, y: number }) => number
```

the argument type requires that values passed to it support accessing the `x` or `y` properties (and further that the properties are `number`s). So these types are compatible since they support those at least those operations:

```
{ x: number, y: number }
{ x: number, y: number, z: number }
{ x: number, y: number, foo: string }
```

but these types are not compatible:

```
{ x: number, y: string }
{ x: number }
boolean
```

This idea of compatibility is called *subtyping*: a type `A` is a *subtype* of a type `B` when all the operations supported on `B` are also supported on `A`. Given this understanding in terms of supported operations, it makes sense that subtyping is *reflexive* (`A` is a subtype of `A`, for any type `A`) and *transitive* (if `A` is a subtype of `B` and `B` is a subtype of `C` then `A` is a subtype of `C`, for any types `A`, `B`, and `C`).

So to synthesize the type of a function call, we synthesize the type of the function; synthesize the type of the passed argument; check that the the passed type is a subtype of the function argument type; and finally return the result type of the function.

## Checking an expression against a type

A type checker that uses only synthesis and subtyping does the job: it ensures that a program doesn't attempt any unsupported operations. But there is an alternative that makes the type checker more usable: when we know the type we expect an expression to have (say, when it appears as the argument to a function and we know the function argument type), then we can *check* the expression *against* the type.

The idea is that when an expression and type have the same structure, we can break them both down and check each expression part against the corresponding type part. When we can't break them down any further, we fall back to synthesis and subtyping.

For example, to check an object expression

```
{ x: 7, y: "nine" }
```

against an object type

```
{ x: number, y: number }
```

we break down the expression and type into properties, then check each property value expression against the corresponding property type: for `x` we check `7` against `number`; and for `y` we check `"nine"` against `number`. We can't break these down further, so we fall back to synthesis and subtyping: for `x` we synthesize `number` from `7` then check that `number` is a subtype of `number` (it is); and for `y` we synthesize `string` from `"nine"` then check that `string` is a subtype of `number` (it is not, so we detect a type error).

One way this makes the type checker more usable is by localizing errors. In the example above, if we synthesize a type for the whole expression then check subtyping we get an error message like

```
{ x: 7, y: "nine" }
^^^^^^^^^^^^^^^^^^^

{ x: number, y: string } is not a subtype of { x: number, y: number }
```

But if we check the expression against the expected type, we get a message like

```
{ x: 7, y: "nine" }
           ^^^^^^
string is not a subtype of number
```

because the type checker breaks down the expression and type as far as it can, and reports the error at the precise subexpression. When expressions and types are large, this makes errors much easier to read and understand.

Another way this makes the type checker more usable is by reducing necessary type annotations, but I'm going to hold off explaining that until we get into the code.

## Checking + synthesis = bidirectional type checking

This approach—check an expression against a type when we know what type to expect, synthesize a type from an expression when we don't—is called *bidirectional type checking*, so named because type information flows in two directions in the abstract syntax tree: from leaves to root when synthesizing, from root to leaves when checking.

