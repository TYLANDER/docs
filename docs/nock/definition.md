---
navhome: '/docs/'
next: True
sort: 1
title: Definition
---

# Definition

The Nock VM is a Turing-complete function that maps a cell `{subject formula}`
to a noun `product`, where `subject` is the data, `formula` is the code, and
`product` is the result.

To program in Hoon, this is all you need to know about Nock. It's probably at
least worth skimming the definition, though.

(Also, the Hoon programmer should know that (a) tail call elimination in Nock is
straightforward; and (b) tail calls in Hoon always become tail calls in Nock.)

## Rules

Nock is a combinator interpreter defined by the spec below.

This pseudocode (any system of axioms is properly defined in pseudocode) is a
system of problem reduction rules. Variables match any noun; the first rule from
the top matches.

A problem that reduces to itself is an infinite loop, hence a crash ("bottom" in
formal logic). Obviously a real interpreter can detect this crash and produce an
out-of-band value instead.

    ?[a b]           0
    ?a               1
    +[a b]           +[a b]
    +a               1 + a
    =[a a]           0
    =[a b]           1
    =a               =a

    /[1 a]           a
    /[2 a b]         a
    /[3 a b]         b
    /[(a + a) b]     /[2 /[a b]]
    /[(a + a + 1) b] /[3 /[a b]]
    /a               /a

    *[a [b c] d]     [*[a b c] *[a d]]

    *[a 0 b]         /[b a]
    *[a 1 b]         b
    *[a 2 b c]       *[*[a b] *[a c]]
    *[a 3 b]         ?*[a b]
    *[a 4 b]         +*[a b]
    *[a 5 b]         =*[a b]

    *[a 6 b c d]     *[a 2 [0 1] 2 [1 c d] [1 0]
                       2 [1 2 3] [1 0] 4 4 b]
    *[a 7 b c]       *[a 2 b 1 c]
    *[a 8 b c]       *[a 7 [[7 [0 1] b] 0 1] c]
    *[a 9 b c]       *[a 7 c 2 [0 1] 0 b]
    *[a 10 [b c] d]  *[a 8 c 7 [0 3] d]
    *[a 10 b c]      *[a c]

    *a               *a

## Operators

The pseudocode notation defines five prefix operators: `?`, `+`, `=`, `/`, and
`*`.

`?[x]` produces `0` if `x` is a cell, `1` if an atom.

`+[x]` produces the atom `x` plus `1`.

`=[x y]` produces `0` if `x` and `y` are the same noun, `1` otherwise.

`/` (`slot`) is a tree addressing operator. The root of the tree is `1`; the
left child of any node `n` is `2n`; the right child is `2n+1`. `/[x y]` is the
subtree of `y` at address `x`.

(For instance, `/[1 [531 25 99]]` is `[531 25 99]`; `/[2 [531 25 99]]` is `531`;
`/[3 [531 25 99]]` is `[25 99]`; `/[6 [531 25 99]]` is `25`; `/[12 [531 25 99]]`
crashes.)

`*[x y]` is the Nock interpreter function. `x` is the subject, `y` is the
formula.

An invalid use of any operator (eg, incrementing a cell) crashes. This is
specified as an infinite loop, but it's not actually an infinite loop.

## Instructions

A valid formula is always a cell. If the head of the formula is a cell, Nock
treats both head and tail as formulas, resolves each against the subject, and
produces the cell of their products. In other words, the Lisp program
`(cons x y)` becomes the Nock formula `[x y]`.

If the head of the formula is an atom, it's an instruction from `0` to `10`.

A formula `[0 b]` produces the noun at tree address `b` in the subject.

A formula `[1 b]` produces the constant noun `b`.

A formula `[2 b c]` treats `b` and `c` as formulas, resolves each against the
subject, then computes Nock again with the product of `b` as the subject, `c` as
the formula.

In formulas `[3 b]`, `[4 b]`, and `[5 b]`, `b` is another formula, whose product
against the subject becomes the input to an axiomatic operator. `3` is `?`, `4`
is `+`, `5` is `=`.

Instructions `6` through `10` are macros; deleting them from Nock would decrease
compactness, but not expressiveness.

`[6 b c d]` is *if* `b`, *then* `c`, *else* `d`. Each of `b`, `c`, `d` is a
formula against the subject. Remember that `0` is true and `1` is false.

`[7 b c]` composes the formulas `b` and `c`.

`[8 b c]` produces the product of formula `c`, against a subject whose head is
the product of formula `b` with the original subject, and whose tail is the
original subject. (Think of `8` as a "variable declaration" or "stack push.")

`[9 b c]` computes the product of formula `c` with the current subject; from
that product `d` it extracts a formula `e` at tree address `b`, then computes
`*[d e]`. (`9` is designed to fit Hoon; `d` is a `core` (object), `e` points to
an arm (method).)

`[10 b c]` is a *hint* semantically equivalent to the formula `c`. If `b` is an
atom, it's a *static hint*, which is just discarded. If `b` is a cell, it's a
*dynamic* hint; the head of `b` is discarded, and the tail of `b` is executed as
a formula against the current subject; the product of this is discarded.

A practical interpreter can do anything with discarded data, so long as the
result it computes complies with the semantics of Nock. For example, the
simplest use of hints is the ordinary "debug printf." It is erroneous for an
interpreter to skip computing dynamic hints; they might crash.

## Implementation

The reader might wonder how an interpreter whose only arithmetic operation is
increment can ever be practical.

The short answer is that a Nock interpreter doesn't have to use the algorithm
above. It just has to get the *same result* as the algorithm above.

The algorithm for decrementing an atom is to count up to it, an O(n) operation.
But if the interpreter knows it's running a decrement formula, it can use the
CPU to decrement directly.

Detecting any Nock formula whose semantics match some function, like decrement,
is a hard problem. But in practice there is only one decrement we run -- the
decrement in the kernel library. Declaring this formula (with a hint,
instruction `10` above) and recognizing it is not hard.
