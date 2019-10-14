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
does that by extending the `GHC.Generics` language with a `Field`, `(:=>:)` and `Exists`,
together with a new typeclass `GenericK`.

  This documents proposes to bring some of the functionality of `kind-generics`
into `GHC.Generics`, thus enabling a programmer to access a bigger universe of
generics by default.


  (builds [Haskell2018](https://victorcmiraldo.github.io/data/hask2018_draft.pdf),

Here you should write a short abstract motivating and briefly summarizing the
proposed change.


## Motivation

  The usage of generic programming within the Haskell community is ubiquitous.
We see a number of libraries relying on `GHC.Generics` to provide functionality
generically. The choice of relying on `GHC.Generics` is natural: it comes with
`base`, and hence requires no extra package. Moreover, there is compiler support
for deriving `Generic`.

  Unfortunately, this all breaks down when a programmer needs to
use more involved datatypes. One example can be found in the source code of
GHC itself. The `GHCi.Message` module defines a GADT named
[`THMessage`](https://gitlab.haskell.org/ghc/ghc/blob/241921a0c238a047326b0c0f599f1c24222ff66c/libraries/ghci/GHCi/Message.hs#L236-268)
that has 23 constructors. When GHC needs to serialize a `THMessage` to bytes,
it does so with the
[`putTHMessage`](https://gitlab.haskell.org/ghc/ghc/blob/241921a0c238a047326b0c0f599f1c24222ff66c/libraries/ghci/GHCi/Message.hs#L303-327)
function:

```haskell
putTHMessage :: THMessage a -> Put
putTHMessage m = case m of
  NewName a                   -> putWord8 0  >> put a
  Report a b                  -> putWord8 1  >> put a >> put b
  ...
  ReifyType a                 -> putWord8 22 >> put a
```

  This is a horribly tedious function to write, and to make things worse, it
must be updated every time `THMessage` changes. (Which is often!) The
implementation of `putTHMessage` is so predictable that it is practically
begging to be written with a `GHC.Generics`-based solution instead. Alas, this
is not currently possible, as `GHC.Generics` does not support GADTs.

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

  In this section we describe the features that `kind-generics` can and cannot handle.

### Features that `kind-generics` can handle that `GHC.Generics` cannot.

* Classes that have kinds besides `Type -> Constraint` or `(k -> Type) -> Constraint`.
  Caveat: not all kinds can be represented. See the sections below for more.

* Constructors with existentially quantified type variables, as in
  `data Foo = forall a. MkFoo a` 
  (or, equivalently, `data Foo where MkFoo :: a -> Foo`).  
  Caveat: certain limitations are placed on what
  the kinds of existentially quantified type variables can be. See the
  sections below for more.
 
* Constructors with existential contexts, as in `data Foo a = Show a => MkFoo` 
  (or, equivalently, `data Foo a where MkFoo :: Show a => Foo`).  
  Caveat: quantified constraints cannot be represented. See
  the section below for more details.
   
* Constructors with rank-n types, as in `data Foo = MkFoo (forall a. a
  -> a)`.  Caveat: due to GHC's inability to bind type variables in
  lambda expressions, manipulating the representation types for rank-n
  type variables is somewhat awkward at present. Possible, but
  awkward. Implementing [this GHC
  proposal](https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0155-type-lambda.rst)
  would make things less awkward.

### Things that `kind-generics` cannot handle at all 

Attempting to derive GenericK for any data types with these properties will result in an error:

* Any data type with a kind that does not end with Type. One consequence is that 
  [unlifted newtypes](https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0098-unlifted-newtypes.rst) 
  are not representable.

* Any field types with a kind besides Type. This would rule out a data type like `data BoxedInt = BoxedInt Int#`.    

* Field types that feature a type family applied to an existentially quantified or 
  rank-n type variable, as in the following examples:

 ```haskell
 type family F a

 data T1 where
   MkT1 :: a -> F a -> T1

 data T2 = MkT2 (forall a. a -> F a)
 ```

  This is an artifact of the way kind-generics decomposes all
  applications of types in representation types. Because type
  families must be saturated with a certain number of arguments in
  order to be a well formed type, decomposing type family
  applications is generally not possible. Implementing [this GHC](https://github.com/ghc-proposals/ghc-proposals/pull/242)
  proposal may make it possible to lift this restriction.
    
* Constructors that contain quantified constraints, such as in data
 Foo where `MkFoo :: (forall a. C a) => Foo`.

 This ultimately comes down to technical limitations of GHC itself, as various 
 bugs (such as [this one](https://gitlab.haskell.org/ghc/ghc/issues/16365) 
 and [this one](https://gitlab.haskell.org/ghc/ghc/issues/17333)) make 
 it impractical to represent quantified constraints.
    
* Existentially quantified type variables whose kinds mention other 
  existentially quantified type variables, such as in 
  `data Foo where MkFoo :: forall k (a :: k). Proxy a -> Foo`.
    
* Existentially quantified type variables with higher-rank kinds, 
  such as in `data Foo where MkFoo :: forall (f :: forall k. k -> Type). k Int -> k Maybe -> Foo`.

### Things that kind-generics cannot handle, but can "fail gracefully"

  Here, "fail gracefully" means that GHC can derive a `GenericK`
instance for a data type applied to all of its arguments, but it will
refrain from deriving instances for partial applications of the data
type at a certain point):

* Any data type with a kind that uses visible dependent quantification, 
  such as in `data Foo :: forall k -> k -> Type`.

* Any data type with a kind that uses nested foralls (e.g., `data Foo :: forall a. a -> forall b. b -> Type`) or is otherwise 
  higher-rank (e.g., `data Foo :: (forall a. a -> a) -> Type`).
   
* Field types that feature a type family applied to a universally quantified type variable, such as in the following example:
 ```haskell
 type family F a
 
 data Foo a where
   MkFoo :: a -> F a -> Type
 ```

     
* Existentially quantified type variables whose kinds mention universally 
  quantified type variables, such as in 
  `data Foo a where MkFoo :: forall a (ex :: a). Proxy ex -> Foo`.

* Certain forms of data family instances cannot be represented. For example, 
  consider this data instance:
 ```haskell
 data family D1 a
 data instance D1 (Maybe a) = MkD1 a
 ```
 
 GHC would be able to derive a `GenericK (D1 (Maybe a))` instance, but not a `GenericK D1` instance. Similarly, in this data instance:

 ```haskell
 data family D2 a b
 data instance D2 a a = MkD2 a
 ```

   GHC would be able to derive a `GenericK (D2 a a)` instance, but not a `GenericK (D2 a)` or `GenericK D2` instance.

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

