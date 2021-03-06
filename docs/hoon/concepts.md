---
navhome: /docs/
sort: 14
next: true
title: Concepts
---

# Concepts and terminology

Usually, a new language is an improvement on one you already
know.  If it isn't, it probably at least shares concepts with a
large family of relatives.  These concepts come with words, like
_variable_ or _function_, that map cleanly onto its semantics.

In Hoon we throw away many of these words and invent new
ones, which mean almost the same things.  But why?

## Motivation

Hoon has no close relatives.  The normal jargon of programming
maps poorly onto its semantics. Some of the common concepts used 
only informally in Hoon include "function," "object," "event," 
"variable," "label," "closure", "environment," "scope," and probably 
a few more.  We do use these words informally, but we're always 
making an analogy whose precision depends on context.

Hoon has concepts *like* all these abstractions, but they remain
*false cognates*.  The closer an inexact cognate, the more
unsettling it feels when the abstractions don't match.  Learning
a few new words is a small price for avoiding this pain point.

## Concepts

A few major Hoon concepts: `noun` (data), `nock` (interpreter),
[`hoon`](#-hoon) (AST node), [`mint`](#-mint) (compiler), [`type`](#-type) (type), 
`gate` (function), `mold` (constructor), `core` (object), `mark`
(protocol).

### <a name="-noun">`noun`</a> (data value)

A value in Hoon is called a `noun`.  A noun is either a
unsigned integer of any size (`atom`), or an ordered pair of any
two nouns (`cell`).  Nouns are like Lisp S-expressions, but
simpler (Lisp atoms effectively have dynamic type bits).

The simplest noun syntax uses brackets, like `[a b]` for the cell
of `a` and `b`.  Brackets nest right; `[a b c d]` is `[a [b [c
d]]]`.

A noun list, by convention, goes from left to right and is 
null-terminated. That said, Hoon uses tuples (improper nouns) more 
freely than Lisp; only genuinely iterative structures are 
null-terminated lists.

When an atom is used as a string (a `cord`), the bytes are always UTF-8 
in LSB first order.  When you see `'foo'` and `%foo`, this refers to the 
integer `0x6f.6f66`, aka `7.303.014`. (Hoon syntax uses `.` to break 
integer atoms in the German style, because `.`'s are URL-safe and `,`'s
aren't.)

One common operation on nouns is `slot`, a tree addressing
scheme which maps an atom to a subtree.  The whole noun is slot
`1`; the left child of `n` is `2n`, the right child `2n+1`.

Nothing in Nock, Hoon, or Urbit can create cycles in a noun or
detect pointer equivalence.  Nouns are generally implemented
with reference counting, and have a lazy short Merkle hash to
help with equality testing and associative containers.

### <a name="-nock">`nock`</a> (interpreter)

Hoon compiles itself to the [Nock](../../nock) VM.  Nock is a
Turing-complete, non-lambda combinator function.  The function
takes a cell `{subject formula}`, and produces a noun `product`.

To program in Hoon, this is all you need to know about Nock.  But
it's still fun and useful to [learn more](../../nock).

## Compilation concepts

### <a name="-hoon">`hoon`</a> (AST node)

A [`hoon`](../reference) is the result of parsing a Hoon source 
expression into an AST node. These AST nodes are nouns, like all 
other Hoon data. Because every Hoon program is, in its entirety, a 
single expression of Hoon, the result of parsing the whole thing 
into an AST is a single `hoon`.

A hoon is a tagged union of the form `[%tag data]`, where the tag 
is a constant such as `%brts` (from the source `|=`, i.e. "bartis"), 
and is matched up with the appropriate type of data (often more 
`hoon`s, from source subexpressions). For example, the expression 
`:-(2 17)`, once parsed into an AST, becomes the following:

```
[%clhp p=[%sand p=%ud q=2] q=[%sand p=%ud q=17]]
```

The `%clhp` is produced from the rune `:-` (i.e. "colhep"). The 2 and 
17 have each been parsed as `%sand`-tagged hoons, which represent 
atoms (in this case each with an aura of `%ud`, i.e. unsigned 
decimal).

To parse Hoon source into a hoon AST, use `ream` on a `cord` 
containing Hoon source. Try the following in `dojo`:

```
(ream ':+(12 7 %a)')
```

The result should be:

```
[%clls p=[%sand p=%ud q=12] q=[%sand p=%ud q=7] r=[%rock p=%tas q=97]]
```

### <a name="-mint">`mint`</a> (compiler)

[`mint`](#-mint) is the Hoon compiler.  It maps a cell `[type hoon]`
to a cell `[type nock]`, where a [`type`](#-type) is type 
information, a [`hoon`](#-hoon) is a parsed expression (AST), and a 
`nock` is a Nock formula. `mint` takes as input the subject type 
and a parsed source expression; it produces a product type and an 
executable formula.

Calculating the output type from the input `hoon` and subject type 
is called "type inference". If you've used another typed functional 
language, like Haskell, the Hoon compiler's type inference does the 
same job but with less intelligence.

Haskell infers backward and forward; Hoon only infers forward.
Hoon can't figure out the type of a noun from how you use it,
only from how you make it.  Hoon can infer tail recursion, but
not head recursion.

Low-powered type inference means you need more type annotations,
which makes your program more readable anyway.  Also, the dumber
the compiler, the easier it is for a dumb human to understand
what the compiler is thinking.

The `mint` gate is inside the core `++  ut`, so call it using the 
following syntax:

```
(~(<gate> <core> <arg 1>) <arg 2> <arg 3>)
```

The `<gate>` is mint, the `<core>` is `ut`, `<arg 1>` is the subject 
type, and `<arg 3>` is the parsed Hoon AST (i.e., a `hoon` from the 
last section). What's `<arg 2>`? It's another piece of type 
information. As part of the type-checking process, mint checks that 
the output type "nests" within the input type, i.e. that the output 
type is a subset of the input type. If not, mint halts with a 
`nest-fail` crash. 

Let's compile an example `hoon` from the last subsection into Nock. 
Try the following in Dojo:

(~(mint ut %noun) %noun [%clhp p=[%sand p=%ud q=2] q=[%sand p=%ud q=17]])

### <a name="-type">`type`</a> (type, as range)

A `type` defines a set (finite or infinite) of nouns and ascribes 
some semantics to it.  There is no direct syntax for defining 
types; they are always defined by inference (i.e., by
[`mint`](#-mint)), usually using a constructor (`mold`).

All types are assembled out of base types defined in `++type`. 
(Look up `++  type` in hoon.hoon for examples.) When the compiler 
does type-inference on a program, it assembles complex types out 
of the simpler built-in types.

### <a name="-gate">`gate`</a> (function)

A `gate` is a Hoon function (lambda or closure).  Other
constructs in Urbit are functions in the mathematical sense -- for
instance, a Nock formula is also a function.  But a gate is not a
formula.  The gate's argument is its *sample*; the result is its
*product*.

### <a name="-core">`core`</a> (object)

A `core` has no exact equivalent in conventional languages, but
the closest equivalent is an object.  An object has methods; a
core has functionally computed attributes (*arms*).  An arm that
produces a gate is the Hoon equivalent of a conventional method;
think of it as a computed attribute whose value is a lambda.

A gate is a special case of a core: a core with one arm, whose
name is the empty string.  The shape of a core is `[battery
payload]`, where *battery* is a tree of Nock formulas

### <a name="-mold">`mold`</a> (constructor)

A `mold` is a constructor function (`gate`).  Its sample is any
noun; its product is a structured noun.  A mold is idempotent;
`(mold (mold x))` always equals `(mold x)`.

Because there's no such thing as a [`type`](#-type) declaration, 
we use molds to define useful types by example; we also use them 
to validate untrusted network data.

### <a name="-face">`face`</a> (named variable)

In a conventional language, we have a scope, environment or
symbol table.  Declaring a variable, like e.g. `int foo = 3`, 
adds the name `foo` to the table with type `int`.

The Hoon analogue is `=/(foo atom 3 ...)`, where `foo` is a `face` 
and `atom` is the type. But Hoon has a homoiconic heap; there is no 
inscrutable scope or environment. There is just the subject, which 
is one noun.  To "declare a variable" is to make a cell 
`[variable old-subject]`, and use that as the subject of the next 
expression.

The label `foo` isn't a key in a symbol table; it's stored as a kind 
of metadata, in the type ([`type`](#-type)) of the new value.  It's not 
a variable named `foo`, whose value is of type `atom`; it's a subtree 
of type `foo:atom`.

### <a name="-limb">`limb`</a> (attribute or variable reference)

A [`limb`](../limb/limb/), like `foo`, is Hoon's equivalent of a variable
reference.  A limb is a [`hoon`](#-hoon); given a subject [`type`](#-type),
[`mint`](#-mint) resolves it to a Nock formula and a product [`type`](#-type).

To resolve limb `foo`, [`mint`](#-mint) searches the subject depth-first,
head-first for either a `face` named `foo`, or a `core` with the
arm `foo`.  If it finds a face, the product is a *leg*, or
subtree of the subject.  If it finds a core, it computes the arm
formula with that core as the subject. A limb can also be a slot 
(direct tree address), like `+15`.

The hoon compiler always resolves a limb to a tree axis lookup, 
regardless of the kind of limb.

`boo:[foo=12 baz=23 boo=34]` evaluates to `34`. 
`+6:[foo=12 baz=23 boo=34]` evaluates to `baz=23`.

### <a name="-wing">`wing`</a> ([`limb`](#-limb) path)

A [wing](../limb/wing/) is a list of limbs.  Like attribute 
references in a conventional language, it uses the syntax `a.b.c`, 
but inside out: Hoon `a.b.c` means "a within b within c", the equivalent 
of `c.b.a` in any other language.  All limbs but the last, here `a`, 
must be legs.

### <a name="-mark">`mark`</a> (type, as protocol name)

A `mark` (really an Arvo concept, not a Hoon concept) is Urbit's
version of a MIME type, if a MIME type was an executable
specification.  The mark is just a label that's used as a path to
a local source file in the Arvo filesystem.  This source file
defines a core that can mold untrusted data, diff and patch,
convert to other marks, etc.

If this sounds like magic, it isn't quite magic.  There's no way
for different urbits to make sure they mean the same thing by the
same mark.  However, when incompatibility happens, marks ensure
that we at least handle the situation in a predictable way.
