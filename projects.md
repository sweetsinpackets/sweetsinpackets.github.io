---
layout: page
permalink: /projects/index.html
title: Projects
---

## Reachability Types

Reachability types track resources and their aliases, offering similar static guarantees like Rust but in a higher-order system with shared mutable states.

```scala
  val r = new Ref(42)       // RHS : Ref[Int]^⋄, r : Ref[Int]^r
  val s = r                 // s : Ref[Int]^s <: Ref[Int]^r
  def f1() = !r             // : (() => Int)^r , captures r
  val f2 = () => new Ref(7) // : (() => Ref[Int]^f2)^f2 , avoidance 
```

Reachability tracking allows reasoning about controlled sharing and separation.
```scala
  val u = new Ref(0); val v = new Ref(9); val w = new Ref(3);
  def f(x : Ref[Int]^{⋄u}) = !u + !v      // : (Ref[Int]^⋄u => Int)^{u,v}
  f(u)  // OK, u in argument qualifier, shared
  f(v)  // rejected!
  f(w)  // OK. w is contextually fresh to f
```

Now, reachability types can have the advantage from region-based systems with no cost: (1) collective resource reasoning (bulk reasoning), (2) lexical or non-lexical lifetime, (3) timely guaranteed deallocation reasoning.

### First-class and Shadow Regions

Our design choice is to let reachability and resources define their regions implicitly:

- Every fresh allocation defines creates a new region
- Coallocation automatically identifies the target region

Thus, we don't need any type distinction, and reachability alone suffices to identify regions.

```scala
  val a = new Ref(7)          // defines its own region
  val a1 = new Ref(6) at a    // coallocates in the same region as a
  val a2 = new Ref(5) at a1   // any element identifies the entire region 
```

This is more convenient than the explicit region handles, especially when the lifetime becomes non-lexical:

```scala
  def c1() = { val a = new Ref(()); new Ref(7) at a } 
  c1()    // : Ref[Int]^⋄ , a fresh region

  val c2 = { val b = new Ref(()); () => new Ref(8) at b }
  c2()    // : Ref[Int]^c2 , each call in a same region
```

The resources themselves can identify implicit regions, and a dummy allocation can always be used if explicit region handler is welcomed.

### Selective Scoped Lifetime

We also have selective scoped lifetime allowing RAII style and automatic reclaiming.

```scala
  val sc = {
    val a = new Ref(()) scoped      // lexical
    val u = new Ref(42) at a        // coallocation
    val v = new Ref(!u)             // ok, no restriction
    new Ref(v)                      // OK to return, separate from a
  }   // a and u are reclaimed
  !sc2                              // OK
```

We don't have type distinction for lexical resources, so still first-class and non-distinguishable from non-lexical resources in their scope.
The only requirement is the scope return is separate from the scoped resource, and any other reasoning is hidden from user, which is rather clean.




## Boundless Quantification: Lightweight Polymorphism

Let us start from a higher-order function programming language with higher-rank polymorphism, which is formally studied in System F-sub (and F-omega-sub with type operators).

Consider a minimal polymorphic higher-order function, an `apply` wrapper:
```scala
  def apply[A, B](f : A => B, x : A) : B = f(x)
```
There exists an issue that the type of `f : A => B` is explicitly destructed with two parameters. But why?

- We need to build the relation between return and `x : A` with the function `f`'s type
- The standard T-App requires a function type to apply

It is not quite convenient as we need to remember that the pair `[A, B]` actually describes one function. 
Meanwhile, it reveals a more important observation: at the definition site, the application `f(x)` has not yet happen, but in order to reason about it, we need to provide the evidences (arrow shape of its type) earlier than necessary.


### Delay Reasoning via Projection Types

In Typescript, utility types `ReturnType<T>` and `Parameters<T>` provide a pattern:
```scala
  def apply[F <: any => any](f : F, x : Parameters(F)) : ReturnType(F)
```
where the `any` is unsafe top and bypass all the static type checking. Why we need that?

- `f : ?` needs an arrow type to apply as a function, so we upcast to the bound
- all bounds are less precise than the expected `Parameters(F) => ReturnType(F)`

So, **early exposure costs precision**. Our work proposes a solution to delay the application reasoning from the definition site to the call site. We have:

```scala
  def apply[F](f : F, x : Dom(F)) : Range(F)
```
as well as the proposed new application rule:
```
  f : F    x : Dom(F)
  -------------------
  f(x) : Range(F)
```

No informative bound of `F` is needed. The fact that `Dom(F)` is inhabited itself serves as the evidence where `F` is an arrow type.

Informally, we propose a new perspective about application (and other things):

- In standard system, declaring a function `f : A => B` says: for any future application on argument `x : A`, it results in a value of type `B`
- In our system, we treat all term as potential function, i.e. any value `: F` can apply to its domain `x : Dom(F)` and gets its range `f(x) : Range(F)`. So, all the reasoning is delayed until actual call, where we have precise values in hand.

It gives a strict extension on standard F-sub, enabling polymorphic eta-expansion to arbitrary terms, but still proven sound statically.

### Lightweight Effect Polymorphism

A natural question is that: besides the type, what else can we benefit from the delay? A good answer is effect polymorphism.

In effect type systems, a function carries *latent effect* and needs an extra parameter (omitting other effects):
```scala
  def apply[A, B, e](f : A =e=> B, x : A) : ...
```
In order to make the effect polymorphic, we have to destruct the type of `f` for the sole purpose.
But we can also delay the effect as well, to say something like *effect of f* like a projection.
```scala
  def apply[F](f : F, x : A) : ... @ Eff(F)
```
It can achieve the relative effects (`@pure(f)`) from Lucas Rytz, with more potential expressiveness for arbitrary types.

The idea is general: instead of reasoning about what has not yet happened at the definition site, we delay the reasoning to the call site. 
More patterns may benefit from the delay: callbacks, ownership transfer (closure capture), etc...


### Lightweight Call-Dependent Types

The delay can also extend in another axis, theoretical foundation, by asking: can we introduce lightweight dependent types by application?
```scala
  def apply[F](f : F, x : T) : F[T]   where T <: Dom(F)
```
Here, we have an image type `F[T]`, or a type level application, that depends on the actual argument type of `T`.
This forms a lightweight form of dependent type, while the dependency is encoded in the function abstraction and application.

<!-- ### Lightweight Polymorphism: Resource

Resources are not necessarily checked in fine grained, or per reference. We introduce arenas and scopes for guaranteed deallocation.
```scala
  def worker(r : Ref[T]^⋄) : Ref[T]^⋄ = {
    val d = !r + 1
    val r2 = new Ref(d) at r  // coallocation, region is shadow
    new Ref(d)                // fresh allocation
  }
  worker(new Ref(..) scoped)  // scoped allocation
  // argument and internal r2 is freed
```
 -->

  <br>

## Selective Other Projects

**Research Purposes**

- Proving Semantic Type Soundness of System F-omega-sub in Logical Relations.
- [Taxiway Finding Algorithm Verification](https://github.com/sweetsinpackets/pathway_finding_verification). Artifact for DASC 2020 paper.
- [Extraction from Hazel AST to OCaml](https://github.com/sweetsinpackets/hazel_extraction_history). Legacy project to extract a legacy version of [Hazel](https://hazel.org/) into OCaml. 

**Course Projects**

- [A Visualization of CDMA](https://github.com/sweetsinpackets/CDMA-visualize). SI 630, Umich.
- [A MIPS CPU implementation using Verilog](https://github.com/sweetsinpackets/MIPS-CPU). VE 370, SJTU.
- [A Video Game Search Engine](https://github.com/sweetsinpackets/search-engine-project). SI 650, Umich.

  <br>

<br>
