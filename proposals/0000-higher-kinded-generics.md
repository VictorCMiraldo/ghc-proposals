---
author: Your name
date-accepted: ""
proposal-number: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/0>).
**After creating the pull request, edit this file again, update the number in
the link, and delete this bold sentence.**

# Make `GHC.Generics` accept GADTs and Higher Kinded Types.

  The [`kind-generics`](https://hackage.haskell.org/package/kind-generics) library
enables the programmer to access generic representations of higher kinded
types and GADTs in a uniform fashion. This lifts the restriction to
types of kind `*` and `* -> *` imposed by `GHC.Generics`. The `kind-generics` library
does that by extendind the `GHC.Generics` language with a `Field`, `(:=>:)` and `Exists`,
together with a new typeclass `GenericK`.

  This documents proposes to bring some of the functionality of `kind-generics`
into `GHC.Generics`, thus enabling a programmer to access a bigger universe of
generics by default.
  
  
  (builds [Haskell2018](https://victorcmiraldo.github.io/data/hask2018_draft.pdf),

Here you should write a short abstract motivating and briefly summarizing the
proposed change.


## Motivation

  ** @ryanglscott, At ICFP you mentioned you had found a usecase from within GHC;
  could you send me some more information? **

  The usage of generic programming within the Haskell community is ubiquitous.
We see a number of libraries relying on `GHC.Generics` to provide functionality
generically. The choice of relying on `GHC.Generics` is natural: it comes with
`base`, and hence requires no extra package. Moreover, there is compiler support
for deriving `Generic`. Unfortunately, however, this seamless interaction breaks
down when the programmer is using more involved datatypes.

  By extending the language of `GHC.Generics` with the combinators seen in
`kind-generics`, we would be able to provide this seamless interaction of libraries
and user-defined datatypes for virtually all possible datatypes. The library
author can easily forbid certain datatypes with an instance
such as:

```haskell
instance TypeError "Existentials are not allowed" => MyLibClass (Exists k f) x where
  ...
```

  With the help of `kind-generics` we can write generic representations for
datatypes such as well typed expressions, for example:

```haskell
data WTExp :: * -> * where
  WTLift :: a -> WTExp a
  WTAdd  :: (Num a) => WTExp a  -> WTExp a -> WTExp a
  WTCmp  :: (Ord a) => Ordering -> WTExp a -> WTExp a -> WTExp Bool
  WFIf   :: WTExp Bool -> WTExp a -> WTExp a -> WTExp a
```

  The generic representation will look something like:

```haskell
type RepK WTExp 
  =   Field Var0
  :+: ((Kon Num :@: Var0) :=>: Field (Kon WTExp :@: Var0)
                           :*: Field (Kon WTExp :@: Var0))
  :+: Exists (*) ((Kon Ord :@: Var0) :=>: 
                  (Kon (~) :@: Var1 :@: Bool) :=>: Field (Kon Ordering)
                                               :*: Field (Kon WTExp :@: Var0)
                                               :*: Field (Kon WTExp :@: Var0))
  :+: ( Field (Kon WTExp :@: Bool) 
    :*: Field (Kon WTExp :@: Var0)
    :*: Field (Kon WTExp :@: Var0))
```

  ** TODO: The example above might be too complex... **
                                             
## Proposed Change Specification

We propose to bring in the [`GenericK`](https://hackage.haskell.org/package/kind-generics-0.4.0.0/docs/src/Generics.Kind.html#GenericK) typeclass and its direct dependencies
into `GHC.Generics`. These dependencies are:

* [`Data.PolyKinded.LoT`](https://hackage.haskell.org/package/kind-apply-0.3.2.0/docs/Data-PolyKinded.html#t:LoT) 
* [`Data.PolyKinded.(:@@:)`](https://hackage.haskell.org/package/kind-apply-0.3.2.0/docs/Data-PolyKinded.html#t::-64--64-:)
* [`Data.PolyKinded.Atom.Atom`](https://hackage.haskell.org/package/kind-apply-0.3.2.0/docs/Data-PolyKinded-Atom.html#t:Atom)
* [`Data.PolyKinded.Atom.Interpret`](https://hackage.haskell.org/package/kind-apply-0.3.2.0/docs/Data-PolyKinded-Atom.html#t:Interpret)
* [`Generics.Kind.Field`](https://hackage.haskell.org/package/kind-generics-0.4.0.0/docs/Generics-Kind.html#t:Field)
* [`Generics.Kind.Exists`](https://hackage.haskell.org/package/kind-generics-0.4.0.0/docs/Generics-Kind.html#t:Exists)
* [`Generics.Kind.(:=>:)`](https://hackage.haskell.org/package/kind-generics-0.4.0.0/docs/Generics-Kind.html#t::-61--62-:)

  Next we provide a short summary of those constructions. For more detailed information
we refer the reader to the [Haskell 2018](https://victorcmiraldo.github.io/data/hask2018_draft.pdf) or the [PADL 2019](https://victorcmiraldo.github.io/data/padl2019.pdf) paper.

  **TODO: Explain the monsters here!**

## Examples

This section illustrates the specification through the use of examples of the
language change proposed. It is best to exemplify each point made in the
specification, though perhaps one example can cover several points. Contrived
examples are OK here. If the Motivation section describes something that is
hard to do without this proposal, this is a good place to show how easy that
thing is to do with the proposal.

## Effect and Interactions

  This proposal is backwards compatible with `GHC.Generics` and would not
break code that already relies on it.

## Costs and Drawbacks

  **TODO**
Give an estimate on development and maintenance costs. List how this effects
learnability of the language for novice users. Define and list any remaining
drawbacks that cannot be resolved.


## Alternatives

  The only known alternative to our proposal is for library authors would be
to depend on the `kind-generics` library and ask the users to use the provided
templated haskell machinery. Having builtin support from the compiler is much
more accesible.

  ** TODO: What else? **

## Unresolved Questions

  ** TODO: Ask Alejandro; I can't think of any **

## Implementation Plan

  ** @ryanglscott Ryan, I'd love to help implementing if necessary!
  I would need some guidance and supervision from you though. **

