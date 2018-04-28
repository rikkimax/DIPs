# Concept: Signatures type

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id)                                                     |
| Review Count:   | 0                                                               |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>            |
| Implementation: |                                                                 |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

A signature is a static vtable representation of complex objects such as structs and classes.
They primarily represent heap allocated objects but can take advantage of the stack for scope.

They are very good at presenting behavior and description of objects in a way familiar to D programmers while limiting ugly template if constraints that is hard to read and understand. The addition could lead to cleaner more interchangable code, as well as seeing a higher uptake in -betterC codebases.

### Links

Signatures are a very uncommon language feature originating in the ML family, examples of this are [Ocaml](https://caml.inria.fr/pub/docs/manual-ocaml/moduleexamples.html) and [SML](http://www.smlnj.org/doc/Conversion/modules.html).

A more recent example of signatures is [Rust's](https://doc.rust-lang.org/1.8.0/book/traits.html) traits or [Swift's](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267) Protocols which both focus upon implementing a specification, not a specification matching an existing representation.

Alternative designs to D's structs is C#, which can [inherit interfaces](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/structs) but not other structs or classes.

Preliminary designs in the C++ community include a type verification mechanism in the form of [concepts](https://accu.org/index.php/journals/2198). Discussion in the D [NewsGroup](https://forum.dlang.org/post/oju39p$1il3$1@digitalmars.com) along with a WIP [DIP](https://wiki.dlang.org/User:9rnsr/DIP:_Template_Parameter_Constraint) for it.

Supplementary code is provided by the author in the [repository](https://github.com/rikkimax/stdc-signatures).

## Rationale

In the D community we utilize idioms and design patterns which emphasize template usage. Template usage enables high code reuse but it also causes code bloat with quite horrendous error messages. These supplementary outcomes are not positive for both new and existing users in understanding why a template does not succeed.

At the core of the problem, we aim to pass around types based upon a statically known interface without knowledge of what the implementation is during compilation. It is quite common for when template conditions failure at a function level, to not know if it is because of a function specific specialization failure or a more general interface one.

The goal of signatures in this DIP is to present a language construct that is familiar to the (S)ML developers but also ties into how it would be used by the D community. So in our case, signatures are stack heavy constructs that can be made from either structs or classes. They have a statically created virtual table that may be assigned functions at runtime with patching to make an implementation match the interface. This allows a runtime decision of the implementation but a static compile-time design for what it must describe.

There exist existing methods to do similar things like Design by Introspection. They do not become obsolete instead, they become stronger more useful in customizing behavior using more concrete types. Allowing for cleaner interfaces and most importantly, a focus on concepts not code "versions" of them.

## Description

This DIP has been split into multiple seperate documents. The size of signatures exceeds the average size of a DIP in terms of additions and so, multiple parts of it will require seperate reviews. This documents purpose is is to give an overview of the concept and how it could benefit the D community.

### Additions

At the center of this DIP is the type [``signature``](type.md). It introduces a type similar to a struct with each member field and method being a pointer (delegate) instead of the value itself transparently.

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
