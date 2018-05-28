# Signature type

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

A signature is a static vtable representation of complex objects such as structs and classes.
They primarily represent heap allocated objects but can take advantage of the stack for scope.

They are very good at presenting behavior and description of objects in a way familiar to D programmers while limiting _template if constraints_ that is hard to read and understand. The addition could lead to cleaner more interchangable code, as well as seeing a higher uptake in -betterC codebases.

### Reference

Signatures are a very uncommon language feature originating in the ML family, examples of this are [Ocaml](https://caml.inria.fr/pub/docs/manual-ocaml/moduleexamples.html) and [SML](http://www.smlnj.org/doc/Conversion/modules.html).

A more recent example of signatures is [Rust's](https://doc.rust-lang.org/1.8.0/book/traits.html) traits or [Swift's](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267) Protocols which both focus upon implementing a specification, not a specification matching an existing representation.

Alternative designs to D's structs is C#, which can [inherit interfaces](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/structs) but not other structs or classes.

Preliminary designs in the C++ community include a type verification mechanism in the form of [concepts](https://accu.org/index.php/journals/2198). Discussion in the D [NewsGroup](https://forum.dlang.org/post/oju39p$1il3$1@digitalmars.com) along with a WIP [DIP](https://wiki.dlang.org/User:9rnsr/DIP:_Template_Parameter_Constraint) for it.

Supplementary code is provided by the author in the [repository](https://github.com/rikkimax/stdc-signatures).

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

In the D community we utilize idioms and design patterns which emphasize template usage. Template usage enables high code reuse but it also causes code bloat with quite horrendous error messages. These supplementary outcomes are not positive for both new and existing users in understanding why a template does not succeed.

At the core of the problem, we aim to pass around types based upon a statically known interface without knowledge of what the implementation is during compilation. It is quite common when template conditions fail at a function level, to not know if it is because of a function specific specialization failure or a more general interface one of the arguments.

The goal of signatures in this DIP is to present a language construct that is familiar to the (S)ML developers but also ties into how it would be used by the D community. So in our case, signatures are stack heavy constructs that can be made from either structs or classes. They have a statically created virtual table that may be assigned functions at runtime with patching to make an implementation match the interface. This allows a runtime decision of the implementation but a static compile-time design for what it must describe.

Methods exist that do similar things like _Design by Introspection_. They do not become obsolete instead, they become stronger more useful in customizing behavior using more concrete types. Allowing for cleaner interfaces and most importantly, a focus on concepts not code "versions" of them.

## Description

This DIP has been split into multiple seperate documents. The size of signatures exceeds the average size of a DIP in terms of additions and so, multiple parts of it will require seperate reviews. This documents purpose is is to give an overview of the concept and how it could benefit the D community.

### Additions

At the center of this DIP is the type [``signature``](type.md). It introduces a type similar to a struct with each member field and method being a pointer/delegate instead of the value itself transparently. Additional changes are provided by [misc](misc.md) which gives convenient behavioral support to match ``struct``'s and classes.

In [resolving](resolving.md) introduces the concept of resolving a ``signature`` type, into an resolved type and what is required to do this with the hidden arguments.

Later in [link](bodies.md) function bodies are defined as supported along with support for a new attribute ``default(...)`` to add impressive new behavior for adaption to other signatures types for the implementation without the use of ``static if``.

### Semantic changes

1. Construction: Is implicitly constructed given an appropriate class or struct instance.
    1. Implicit construction of a signature may occur during assignment or passing as an argument to a function call.
    2. Implicit construction works in two forms. First as direct assignment and patching for class instances and pointers to structs. Second is a memory allocation and move of the struct instance, should it be copyable. Where it will be assigned and then patched for the given vtable.
    3. If the implicit construction is going into a scope'd variable, no allocation needs to take place unless it is being returned from a function. If it is being returned from a function it will be malloc'd and then free'd when it goes out of scope.
    4. Should a struct instance not be going into a scope'd variable (or being returned without scope) it will be allocated into GC owned memory where it will be moved into, should it be copyable. If it cannot be copied, it is an error.
2. Members are virtual:
    1. There may be fields and methods. These are pointers/delegates.
    2. Operator overloads may be used.
    3. Must not depend upon template parameters which are not supplied (this will be supported in another part of the DIP).
    4. Types, alias and enum work as is with existing structs and classes.
    5. ``@safe`` field/method may be assigned to a method without any attribution. Same with ``@trusted``, ``nothrow``, ``@nogc``, ``const`` and ``shared``. Different linkage/calling conventions are supported by the compiler creating a small patch function.
3. ``this`` will not refer to anything and will be an error (will be changed in a seperate part of the DIP).
4. A signature may inherit from others, similar in syntax to interfaces in this manner. However the diamond problem is not valid with signatures. If a field/method is duplicated and is similar, it can be ignored. If it is different (e.g. different types or different attributes) then it is an error at the child signature.
5. The first pointer a signature instance stores is to the data it represents. This can be accessed by ".ptr". If this pointer is null, so is the signature instance. To check for this use ``v is null``.
6. Signatures may be cast up for their inheritence. This can be computed statically and does not require any runtime knowledge. However they may not be cast down again. Rules regarding const, immutabe, shared ext. still apply like any other type.
7. Method bodies may not be provided except for static methods (who cannot be assigned by the implementation).
8. Usage of signatures as return types and arguments obey the same rules associated with interfaces and classes inside said interfaces/classes hierachies which is covariance and invariance. Patch functions will be required to implement this however and have each version available for casting to parent signatures.
9. ``.sizeof`` property on signatures, total size of signature instance, works identically to a ``struct``'s ``.sizeof`` property.
10. ``.offsetof`` works on each member field and method (with help of ``__traits(getOverloads)``) of a signature.

### Grammar changes

```diff
Keyword:
+    signature

TypeSpecialization:
+    signature

AggregateDeclaration:
+    SignatureDeclaration

+ SignatureDeclaration:
+    signature Identifier AggregateBody
+    signature Identifier : SignatureInheritance AggregateBody
+    SignatureTemplateDeclaration

+ SignatureInheritance:
+    Identifier
+    Identifier , SignatureInheritance

+ SignatureTemplateDeclaration:
+    signature Identifier TemplateParameters Constraint|opt : IdentifierList AggregateBody
+    signature Identifier TemplateParameters : IdentifierList Constraint|opt AggregateBody

Attribute:
+    default ( SignatureDefaultArgs|opt ) FunctionBody

+ SignatureDefaultArgs:
+    SignatureDefaultArg
+    SignatureDefaultArgs , SignatureDefaultArg

+ SignatureDefaultArg:
+     !|opt Type
```

## Breaking Changes and Deprecations

The primary breaking change is the token ``signature`` is becoming a keyword.
Existing code that uses it would need to rename existing symbols and variables but nothing requiring major changes.


## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
