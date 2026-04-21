---
layout: page
permalink: /projects/index.html
title: Projects
---

## Research

With standard System F-sub, we can achieve:
```scala
  def map[A,B](f : A => B, x : A) : B = f(x)
```
With prior works on reachability types (Wei et al. 2024), we can track resources and aliases, and reason about their separation and sharing.
```scala
  def map[A^q,B](f : (y : A^⋄) => B^{y} , x : A^q ) : B^{x} = f(x)
```
The question is, can we do better?

### Lightweight Polymorphism: Types

We don't need parameters `A` and `B` to encode such an application. With our projection types, we can type polymorphism without type parameters.
```scala
  def map[F](f : F , x : Dom(F)) : Range(F) = f(x)
```
Similar to the utility types `ReturnType` and `Parameters` in TypeScript, but it's formalized and statically checked.

### Lightweight Polymorphism: Resource

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
