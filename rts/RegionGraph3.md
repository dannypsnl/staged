
# Lightweight region memory management in a two-stage language

A big part of my recent work is about PL theory and design, trying to get good
combinations of predictable high performance and high-level abstractions.
Staged compilation is an important part of it:

- [Staged Compilation With Two-Level Type Theory](https://andraskovacs.github.io/pdfs/2ltt.pdf)
- [Closure-Free Functional Programming in a Two-Level Type Theory](https://dl.acm.org/doi/10.1145/3674648)

I've been doing some prototyping and design exploration. There's no usable
standalone implementation but I do want to get to that point eventually.

*Garbage collection* is present in all of my designs, simply because the
starting point is functional programming with ADTs and closures, and the most
convenient setup for this is to have GC.

If we aim for the best performance, GC can be crucial for some workloads, but
it's more common that we explicitly *don't* want to use a GC. Most
latency-critical programs are written in C, C++ or Rust, without GC.

Naturally, I'm interested in improving memory management performance, but there
are some trade-offs and tensions.

First, *type safety* is non-negotiable for me, both at runtime and compile
time. By the latter I mean that well-typed code-generating programs must yield
well-typed code. So every design choice has to accommodate type safety.

Second, two-level type theory (2LTT) is extremely expressive and convenient for
two-stage compilation. But: systems that are memory-safe and GC-free tend to be
[*sub-structural*](https://en.wikipedia.org/wiki/Substructural_type_system). This
means that there are restrictions on the usage of variables. Sub-structural
object languages haven't been researched for 2LTT-s, and I don't think that they
work very well there.

The nicest thing about 2LTT is that we never have to care about *scoping* for
the object language. At compile time, we can freely combine object expressions
without regard for their free variables, and we only need to care about typing.

If the object language is sub-structural, this doesn't work anymore. If a linear
variable is consumed in some subterm, it can't be used in another subterm. So if
we're writing metaprograms, aiming to generate substructural programs, we have
to keep track of free variable occurrences in expressions. This is anything but
lightweight if we want to maintain full type safety.

**Summary of my proposal**:

- We can use *regions* to reduce GC workload, by skipping copying and scanning
  of structures that are stored in regions. The more we know about the
  allocation patterns in our program, the more GC work we can remove.
- Regions are quite liberal: they can be stored in existentials and closures,
  they may contain pointers to the GC-d heap and other regions, and there's no
  sub-structural typing.
- Objects stored in a region are alive as long as the region itself is alive.
- We use *tag-free garbage collection*, aggressive unboxing and bit-stealing to
  improve locality and to shrink runtime objects.
- We use per-datatype GC routines that exploit static information about regions.

I expand on the design in the following.

## Basics

We start from the setup in [Closure-Free Functional
Programming](https://dl.acm.org/doi/10.1145/3674648), but I shall reiterate it
here as well.

- There's a compile-time (meta) language, which is dependently typed.
- There's an object language, which is *polarized* and simply typed.

The metalanguage is highly expressive and has fancy types, and we can write
metaprograms in it that generate object programs. The object language is easy to
compile and optimize, but it's often tedious to directly program in.

Concretely, the system looks like a dependently typed language with some
universes and "staging" operations.

- `MetaTy : MetaTy` is the universe of compile-time types (or: metatypes). It
  has itself as type; this is a logically inconsistent feature that you may know
  as "type-in-type" from other languages. We have it as a simplification and
  convenience feature.
- `Ty : MetaTy` is the universe of object types.
- `ValTy : MetaTy` and `CompTy : MetaTy` are both cumulative sub-universes of `Ty`,
  e.g. if we have `A : ValTy` then we also have `A : Ty`.
- `ValTy` contains *value types*; these are primitive types, ADTs and
  closures. At runtime, a value lives in dynamic memory (stack or heap).
- `CompTy` contains *computation types*; these are functions and finite products
  of computations. At runtime, a computation is represented by an address
  pointing to a chunk of machine code in the executable.

The polarization to `ValTy` and `CompTy` is used to control *closures*. There is
a separate type former for closures, and we *only* get runtime closures if we
use it.

- Function domain types must be value types.
- ADT constructor fields must be value types.
- For `A : CompTy`, we have `Close A : ValTy`
- For `t : A` we have `close t : Close A`.
- For `t : Close A`, we have `open t : A`.

An example:
```
    data List (A : ValTy) := Nil | Cons A (List A)

    map : Close (Int → Int) → List Int → List Int
    map f xs := case xs of
      Nil       → Nil
      Cons x xs → Cons (open f x) (mapPlus f xs)
```
The function argument to `map` has to be wrapped in a closure;
it's not possible to have `(Int → Int) → List Int → List Int`
because function domains must be value types.

It's not possible to have a polymorphic object-level `map`, simply because
there's no polymorphism in the object language. Fortunately, we can abstract
over anything in the metalanguage. For that, we need *staging operations*:

- For `A : Ty`, we have `↑A : MetaTy`, pronounced "lift `A`", as the type of
  metaprograms which produce `A`-typed programs.
- For `t : A : Ty`, we have `<t> : ↑A`, pronounced "quote `t`", as the metaprogram
  which immediately returns `t`.
- For `t : ↑A`, we have `~A`, pronounced "splice `t`", which runs the metaprogram
  and inserts its output in an object program.
- `<~t>` is definitionally equal to `t`.
- `~<t>` is definitionally equal to `t`.
- Splicing binds stronger that function application, e.g. `f ~x` is parsed
  as `f (~x)`.
- Lifting, quoting and splicing is the only way to mix programs at different
  stages.

A polymorphic `map` is now written as
```
    map : {A B : ValTy} → (↑A → ↑B) → ↑(List A) → ↑(List B)
    map f as = <letrec go as := case as of Nil       → Nil
                                           Cons a as → Cons ~(f <a>) (go as);
                go ~as>
```

- The braces for `{A B : ValTy}` mark implicit arguments in the style of Agda.
  In GHC's more verbose style it would be like `forall (a :: ValTy) (b :: ValTy)`.
- The semicolon at the end of `go`'s definition is used to delimit the defined
  thing in a `letrec`. We have non-recursive `let` too, as `let x : A := t; u`.
  Non-recursive `let` can shadow previously defined names.
- A meta-level definition uses `=` while an object-level one uses `:=`.

The previous monomorphic `map` can be reproduced:

```
    monoMap : Close (Int → Int) → List Int → List Int
    monoMap f xs := ~(map (λ x. <open f ~x>) <xs>)
```
At compile time, the splices are replaced with generated code, and we get the same
code that we defined before.

However, it's not the best idea to have a `map` which takes a closure as
argument. We might as well inline the function arguments to skip the runtime
closure:

```
    monoMapPlus : List Int → List Int
    monoMapPlus xs = ~(map (λ x. <~x + 10>) <xs>)
```

### Stage inference

The staging operations in the examples so far caused some noise. It's possible
to have *stage inference* as an elaboration feature, and we can skip pretty much
all quotes and splices, and also most lifts. The following works:

```
    map : {A B : ValTy} → (A → B) → List A → List B
    map f as = letrec go as := case as of Nil       → Nil
                                          Cons a as → Cons (f a) (go as);
               go as
```
Note that we still have to write `=` and `:=` for definitions at the different
stages. I have found that as soon as we pin down the stages of let-definitions,
almost everything else becomes unambiguously inferable.

## Regions

Let's add regions to the mix.

- `Loc : MetaTy` is the type of memory locations.
- `Region : MetaTy` is the type of region identifiers. It's a subtype of `Loc`,
  so that we can implicitly coerce from `Region` to `Loc`.
- `Hp : Loc` represents the general GC-d heap.
- In the object language, polymorphic functions over `Region` are computation
  types.  For example `(r : Region) → List r Int → List r Int` is in `CompTy`
  (we'll see lists in regions shortly). We also have `{r : Region} → ...`.
- ADT constructors may store existential regions.
- `let r : Region; t` creates a new region and binds it to `r` in `t`.

We also need a way to put things in regions. We extend ADT declarations with
location specification. Example:

```
    data List (l : Loc)(A : ValTy) :=
        Nil
      | Cons@l A (List l A)
```

`Cons@l` specifies that a cons cell is a pointer to a pair of `A` and `List l A`,
allocated in `l`.

`Nil` has no location specification, which means that it's *unboxed*, i.e. we
don't need any indirection to store an empty list; at runtime it could be just a
null integer.

For any type, all values of the type must have the same "flat" size in memory.
In the case of lists, a `Nil` would be a null value and a `Cons` would be a
pointer to a pair, so that checks out.

We can also define Rust-style unboxed sums:

```
    data Foo := Foo1 Int | Foo2 Int Int
```

Since neither `Foo1` nor `Foo2` specify a location, they are both unboxed.  At
runtime, we need a tag bit to distinguish the two constructors, and we also need
to pad out `Foo1` to have the same size as `Foo2`. So, `Foo` requires 1+64+64
bits of storage. Depending on the contexts in which we use `Foo`, this data can be
stored differently; see "bit-stealing" a bit later on. We assume word
granularity for runtime objects, so a `Foo` value takes up three words at most.

Recursive fields can only be placed under a pointer. The following is rejected
by the compiler:

```
    data List A := Nil | Cons A (List A)
```

Unlike in Rust, we can mix unboxed and boxed constructors in a declaration (as
we saw for lists already). Take lambda terms, using De Bruijn indices:

```
    data Tm = Var Int32 | Lam@Hp Tm | App@Hp Tm Tm
```

This representation is far more efficient than what we get in Haskell
and OCaml.

- `Var Int32` is a single word, containing two bits for the constructor tag and
  32 bits for the integer.
- `Lam@Hp Tm` is a pointer tagged with two bits, pointing to a `Tm` on the heap.
- `App@Hp Tm Tm` is a pointer tagged with two bits, pointing to two `Tm`-s on the heap.

There's also one bit reserved for GC in every pointer; I'll write more about
GC implementation later.

Consider `App (Var 0) (Var 1)`. In Haskell and OCaml, `App` has three words, one
header and two pointer for the fields, and `Var 0` and `Var 1` each contain two
words, one header and one unboxed field. That's a total of **seven** words on
the heap.

In 2LTT, it's **two** words on the heap instead! The `App` pointer itself has
the constructor tag, it points to a tag-free pair of two words, and the two
words are unboxed `Var`-s.

### Bit-stealing

It would be already quite efficient to *uniformly* represent ADT-s, but we can
do more layout compression. Let's take `List (Maybe Int)`:

```
    data List A := Nil | Cons@Hp A (List A)

    data Maybe A := Nothing | Just A
```
`List` is on the heap while `Maybe` is an unboxed sum. Using uniform representation,
we would have that `Maybe Int` takes two words (tag + payload), so a `Cons` cell
would be three words on the heap.

Using bit-stealing, we can move immutable data from constructors into free space
in pointers. In this case, `Cons` uses 0 bits for tagging (since `Nil` can be
represeted as a null value) and reserves 1 bit for GC.

The free tag space varies depending on the architecture and the exact tagging
scheme. At least, we have 2 lower bits available because of pointer alignment
(recall that reserve 1 bit for GC). At best, we have additional 16 high bits
available.

If plenty high bits are available, we might choose to only exploit high bits and
leave low bits alone; this can make tag operations use fewer machine
instructions.

In any case, we can certainly use bit-stealing for `List (Maybe Int)`. Moving
the `Maybe` tag into `Cons`, we get **two words** on the heap for a `Cons`.

### Using regions

The most important property about regions is the following:

**Objects stored in a region are alive as long as the region itself is alive.**

First I give some code examples, then explain how the region property is used to optimize GC.

Location-polymorphic mapping looks like this, fully explicitly:

```
    data List (l : Loc) (A : ValTy) := Nil | Cons@l A (List l A)

    map : {A B : ValTy}{l l' : Loc} → (↑A → ↑B) → ↑(List l A) → ↑(List l' B)
    map {A}{B}{l}{l'} f as =
       <letrec go as := case as of
                 Nil       → Nil
                 Cons a as → Cons@l' ~(f <a>) (go as);
        go ~as>
```

We can rely on a lot of inference though. We've seen stage inference before, and we can also
make location annotations on constructors implicit when they are clear from the
expected type of an expression.

```
    data List l A := Nil | Cons@l A (List l A)

    map : {A B : ValTy}{l l'} → (A → B) → List l A → List l' B
    map f as =
       letrec go as := case as of
                 Nil       → Nil
                 Cons a as → Cons (f a) (go as);
       go as
```

Concrete instantiations of `map` are object functions where the function
argument is inlined, working on lists stored at given locations.

We can recover a region-polymorphic object function as follows:

```
    myMap : {r r' : Region} → List r Int → List r' Int
    myMap {r}{r'} xs := ~(map {Int}{Int}{r}{r'} (λ x. <~x + 10>) <xs>)
```
With inference:

```
    myMap : {r r'} → List r Int → List r' Int
    myMap xs := map (λ x. x + 10) xs
```
We get an inner recursive `go` function which refers to the `r` and `r'`
parameters of the outer function. Hence, when we're in the middle of mapping,
`r` and `r'` are both reachable on the stack (or in registers). Hence, no objects
stored in `r` and `r'` can be freed by GC.

This enables a remarkable amount of GC optimization. Object types contain
information about locations, and for each type we can implement a GC strategy
which takes locations into account.

Consider `List r Int` for `r : Region`. Whenever a value of this type is
reachable, the region `r` must be reachable as well (it must be in scope, or
otherwise the type is not even well-formed). Therefore, *doing nothing* is a
valid GC strategy for this type! When `r` becomes dead, the whole region gets
freed, and with it all the `List r Int` values are freed too. When GC
processes a `Region`, it does not look at its contents, it simply marks the
region itself as alive.

Consider `List r (List Hp Int)`. When a value of this type is reachable, we know
that `r` must be also reachable, but we don't know which heap-based inner lists are
stored. Hence, GC scans the outer list, in order to reach and relocate the inner lists. But GC does not relocate the region-based cons
cells. `Hp` may use copying GC, but regions are not copied and region pointers
are stable.

Lastly, consider `List Hp (List r Int)`. GC traverses and relocates the outer
cons cells, but it doesn't look into the inner lists.

### Strict regions

It's not too rare in practice (at least in the compilers and elaborators that
I've implemented) that we have some long-lived tree structure which may contain
references to the general heap. Toy example:

```
    data HpVal := HpVal@Hp Int
    data Tree (r : Region) := Leaf@r HpVal | Node@r Tree Tree
```

In elaborators, I often embed the *lazy values* of top-level definitions into
core terms, right next to top-level variable nodes, so that I can eliminate the
cost of top-level lookups during evaluation, and also avoid passing around a
top-level store. Core terms are persistent but lazy values are very much
dynamic; before we finish elaboration, we don't know which top-level thunks need
to be forced, and forcing may generate arbitrary amount of garbage.

It's awkward if GC has to deeply traverse trees because they contain heap
references. A common pattern is to replace heap pointers with integer indices
pointing to an *array* of heap values.

```
    Index : ValTy
	Index = Int

    data Tree (r : Region) := Leaf@r Index | Node@r Tree Tree
```

This style is actually more common in PL implementations than my own style of
embedding values in terms. With this, GC doesn't scan trees anymore, but the
programmer has the extra job of correctly managing the arrays and the indices.

We extend the system with **strict regions**:

- There's `StrictRegion : ValTy → MetaTy`. A value of `StrictRegion A` can be
  implicitly cast to `Loc`.
- We can only allocate values of type `A` in an `r : StrictRegion A`.
- When GC touches a strict region, *it scans all of its contents*.
- `let r : StrictRegion A; t` creates a new strict region.

We need to track the types of values in strict regions, because tag-free GC
needs the type info (or more precisely, memory layout) to do the eager scanning.

Adding objects to strict regions has a bit of a spin: assuming `r : StrictRegion
A` and `a : A`, we have `addToStrictRegion r a : Ptr r A`, where `Ptr` is
defined as:

```
    data Ptr (l : Loc) A := Box@l A
```
Why can we only get pointers to objects in a strict region? If we try to use
the same allocation API as with vanilla regions, we run into a problematic recursion
in typing. What if I want to have lists in a strict region:

```
    data List (r : StrictRegion (List ?)) := ...
```
Lists depend on the region where they're stored, but the region is indexed with the type of
stored values... There's a good chance that this recursion could be supported with some additional
magic, but for now I'd like to stick to simpler type-theoretic features.

In our current API, we add lists into a strict region like this:

```
    data List l A := Nil | Cons@l A (List l A)

    main : ()
	main :=
	  let r : StrictRegion (List Hp Int);
	  let x := Cons 10 (Cons 20 Nil)
	  let x' := addToStrictRegion r x;
	  case x' of
	    Box x → ...
```

For lists, the "flat" size of a value is a single word (a pointer or `Nil`), and
we copy that single word when we add a list to a strict region, and we get a pointer
to the new copy. The tree example now:

```
    data HpVal := HpVal@Hp Int
    data Tree (r : Region)(r' : StrictRegion HpVal) :=
	  Leaf@r (Ptr r' HpVal) | Node@r Tree Tree
```

Now, GC doesn't scan values of `Tree r r'`, because the contents of `r'` are
known to be scanned elsewhere.

### Closures

Let's look at the interplay of closures and regions. First, the location of a
closure can be specified:

```
    Close : Loc → CompTy → ValTy
```

Although we can store closures in regions, they always have to be scanned by GC,
because closures can capture any data, including heap pointers and regions
themselves.

Example for a closure capturing a region:

```
    index : {r : Region} → List r Int → Int → Int
	index xs x := ...

    indexClosure : {r : Region} → List r Int → Close (Int → Int)
	indexClosure xs := close (λ x. index xs x)
```

The closure captures `xs`, but since the type of `xs` depends on `r`, it
captures `r` too. The closure can be then stored in any ADT constructor as a
value. So we have to be mindful that closures incur the same GC costs as heap
values.

As I mentioned before, free variables are implicit in vanilla 2LTTs, so we can't
control what gets captured in a closures. A popular solution is to add a
*modality*. In our case, we could add a modality for closed object-level
terms. This has been used several times in 2LTTs, and its implementation is
[fairly well
understood](https://users-cs.au.dk/birke/papers/implementing-modal-dependent-type-theory-conf.pdf).
It definitely bumps up the complexity of the system, and I don't plan to
implement it any soon. I write a bit more about it in [Appendix: closed
modality](TODO).

Alternatively, we can often
[*defunctionalize*](https://en.wikipedia.org/wiki/Defunctionalization) closures
to ADT-s, in which cases we can specify locations of captured data.

Defunctionalization isn't always good for performance though.[Threaded
interpreters](https://en.wikipedia.org/wiki/Threaded_code) and
[closure-generating
interpreters](https://ndmitchell.com/downloads/slides-cheaply_writing_a_fast_interpreter-23_feb_2021.pdf)
are all about replacing case switching with jumps and calls to dynamic
addresses. We probably need the closed modality for GC elision here.

### Existential regions

ADT constructors may have fields of type `Region` and `StrictRegion A`, and
types of fields to the right can depend on them. The following is a generic
existential wrapper type.

```
    data Exists l (F : Region → ValTy) := SomeRegion @l (r : Region) (F r)
```

The eliminator:
```
	match : {l : Loc}{F : Region → ValTy}{B : Ty} → ↑(Exists F) → ((r : Region) → ↑(F r) → ↑B) → ↑B
	match e f = <case ~e of SomeRegion r fr → ~(f r <fr>)>
```

It's not possible to project out only the region, because `Region : MetaTy` and
the result of matching must be in `Ty`.

"Pinned" arrays could be defined in terms of existential regions. Let's have
`Array : Loc → ValTy → ValTy` as a primitive type. An `Array l A` is represented
as an `l`-pointer to a length-prefixed array. A pinned array is an array packed
together with its region:

```
    data PinnedArray A := MkPinnedArray (r : Region) (Array r A)
```

Pinned arrays can be used in pretty much the same way as heap arrays, the only
difference is in memory costs. Pinned arrays are effective when we have
relatively few relatively large arrays at runtime.


## Non-escaping regions

Our region setup permits an optimization that should be easy to implement
and often applicable.

If the compiler can discern that a region never escapes the scope of its
creation, it can insert code to eagerly free the region, without waiting for GC
to run. For example:

```
    f : Int → Int
	f x :=
	  let r : Region;
	  let xs := [x, x+1, x+2] in r;
	  sum xs;
```

`r` does not escape, so the compiler can generate:

```
    f : Int → Int
	f x :=
	  let r : Region;
	  let xs := [x, x+1, x+2] in r;
	  let res := sum xs;
	  free r;
	  res;
```

It's important that `free r` by itself does not keep `r` alive. It can happen
that `r` becomes dead and is collected well before `free r` is executed, in
which case `free r` does nothing. So we can only make freeing strictly more
timely.

How often do we have non-escaping regions? Very often, I conjecture. The only
way for regions to escape are:

- Returning a closure or an existential.
- Throwing an exception containing a closure or an existential.
- Making a mutable write that contains a closure or an existential.

My experience so far with the polarized 2LTT is that *closures of any kind* are
already rarely used in practice. And closure-free programs should be especially
easy to escape-analyze because they only contain statically known function calls.

## Appendix: closed modality

Let's have `■ A : MetaType` for `A : MetaType`. Adding the `■` ensures that only
closed object-level terms can be produced. For example, `■ (↑A)` is the type of
metaprograms which generate closed `A`-typed object terms. If we have `■ A` for
a purely meta-level type `A`, like meta-level natural numbers, the modality
doesn't have any effect.

For the details of `■`, you can look at [this
paper](https://users-cs.au.dk/birke/papers/implementing-modal-dependent-type-theory-conf.pdf).

The simplest use case is to support raw *machine code addresses* as values.
This is the special case of closures where there is no captured environment.

```
    CompPtr : CompTy → ValTy
	close   : {A : CompTy} → ■ (↑A) → ↑(CompPtr A)
	open    : {A : CompTy} → ↑(CompPtr A) → ■ (↑A)
```

`■ (↑A)` ensures that no object-level free variables appear in the generated
code.

We can use this to implement transparent closures, where the capture is visible
in the type:

```
    TransparentClosure : ValTy → CompTy → ValTy
	TransparentClosure A B = (A, CompPtr (A → B))
```

It takes more work to implement closures where we only specify the storage
locations of the capture, and the capture is otherwise abstract.

The current setup doesn't let us talk about value types with specified storage
locations. We'd need something like indexing `ValTy` with a set of
locations. This would add a lot of noise, and in fact I find it to be a great
convenience feature that we *don't* have to thread locations through
metaprograms all the time.

Perhaps we could add typeclass-like constraints for location tracking.

- We have `Locations : MetaTy` as a primitive type, whose values are *sets* of
  locations. We need at least the empty set and the union operation. It's also
  good to have some built-in definitional equality magic for location sets.
- We have `HasStorage A ss : Constraint`, where `A : ValTy` and `ls : Locations`
  and `Constraint : MetaTy` is similar to Haskell's constraint kind.

Location-restricted closures could have this API:

```
    Closure : Locations → ValTy → ValTy
	close   : HasStorage A ls => ↑A → ■ (↑(A → B)) → ↑(Closure ls B)
	open    : ↑(Closure ls B) → ↑B
```

Of course, `HasStorage` needs to have built-in behavior for pretty much every
object type. For example, `HasStorage A ls` should imply `HasStorage (List l A)
(l ∪ ls)`.

This is definitely extra bureaucracy, but note that a similar location tracking
already needs to happen in the compilation pipeline, just to generate code and
GC routines.