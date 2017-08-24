# Type signatures

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id)                                                     |
| Review Count:   | 0                                                               |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>            |
| Implementation: |                                                                 |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

A combination of behavior and interface; enables passing of a set of behaviors and values from unknown sources using a encapsulation.
It provides dynamic templating of types while appearing as one to the end programmer with a strong focus on heap allocation of memory.

### Links

TODO: Ocaml/SML

## Rationale

Encapsulating behavior directly into an interface of a wrapped type provides benefit in documentation and ease of passing around of data.
In (S)ML based languages signatures provides additional behavior, verification of existing behavior and interfaces.
Commonly this is used to seperate an implementation from the usage, without using classes with inheritence.

## Description

This DIP is limited to Classes and structs.
A signature may have its instance stored on the stack, heap (via malloc for returning with scope) or heap via already constructed memory.

Signatures by their very nature are dynamic. In this DIP they are always templated and any usage of them are inherently templated.

### Additions

1. The signature type.
    1. Any symbol that contains or has a signature in arguments or as return type is automatically templated.
    2. Is implicitly constructured given an appropriete class or struct instance.
    
        1. When returning from a function, a struct (not a pointer to a struct) will be moved into into malloced memory only when the return type is marked with scope.
          When the caller scope ends it will automatically free said memory.
        2. If the caller is passing to a function via argument and it has not already been constructed it will be constructed.
          By value struct will using the stack as context memory block. Requiring no extra actions.
  
    3. There may be variables, methods and static functions. Operator overloads are also valid. A signature copies what structs supports here.
    4. May be templated. May include exactly one to be initialize value. Valid: ``signature``, ``signature(T)``, ``signature(T, U=Unqual!T)``; Invalid: ``signature(T, U)``.

2. Implementation of ``alias this``.
  This will not cause conflicts like in other situations. The implementation can be fairly dumb.
  A definition of one thing via ``alias T;`` can be duplicated, same situation with methods. But not static functions.
  If a variable is declared multiple times with different types, this is an error.
    
3. Dynamic alias mapping via ``alias T;``.
  Alias T set from the source type as ``Source.T``.
  Only valid within a signature.
  
4. Less important but existing nontheless template ``if (signature)``.
5. Trait evaluation to get/initialize a template instance of a function for a specific signature initialization.
  ``RT [function|delegate] (Args) pointer = __traits(getSignatureEvaluated, <function/method>, <description>);``
  It provides a function pointer or delegate to a function or method who uses signatures. The function/method passed must have template arguments already evaluated.<br/>
  Example function argument is ``foobar``, for method ``Foo.bar``. If it has template arguments, then they must be passed at this point e.g. ``foobar!int``.<br/>
  The description is in the form of ``Name=Type, ..., [return|argument name], ...`` e.g. ``__traits(getSignatureEvaluated, foo, Color=RGBA, IndexType=size_t, return)``.
  When you finish a description (except last one), the argument/function name must end in a semicolon.
  The evaluation process is a simple match, all descriptions must match and the descriptions must include every ``alias T;`` in every signature types.
6. Is statement is extended to support checking for if it matches a signature e.g. ``is(HorizontalImage!RGBA == Image)``.
  This is equivalent to ``__traits(compiles, Image(HorizontalImage!RGBA.init))``.

### Breaking changes / deprecation process

The primary breaking change is the token ``signature`` is becomming a keyword.
Existing code that uses it would need to rename existing symbols and variables but nothing requiring major changes.

### Grammar changes

Signatures are aggregates and require very few changes. The new construct is parsed from the keyword `signature`, just like classes are parsed from `class`.

```diff
AggregateDeclaration:
    ClassDeclaration
    InterfaceDeclaration
    StructDeclaration
    UnionDeclaration
+    SignatureDeclaration 

+ SignatureDeclaration:
+    signature Identifier AggregateBody
+    SignatureTemplateDeclaration

+ SignatureTemplateDeclaration
+    singature Identifier TemplateParameters Constraint AggregateBody
```

To this, a trait identifier is added: 

```diff
TraitsKeyword:
    ...
+    getSignatureEvaluated 
```

(TODO <identifier> part of the new traits argument)

### Examples

#### Image library

__Definitions of what an image is:__

```D
signature ImageBase(T) {
  alias Color;
  alias IndexType;

  @property {
    IndexType width();
    IndexType height();
  }
}

signature UniformImage(T) {
  alias ImageBase!T this;
  
  Color opIndex(IndexType i) if (signature);
  Color opIndex(IndexType i) if (is(T:IndexedImage)) {
    return IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)];
  }
  
  void opIndexAssign(Color v, IndexType i) if (signature);
  void opIndexAssign(Color v, IndexType i) if (is(T:IndexedImage)) {
    IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)] = v;
  }
}

signature IndexedImage(T) {
  alias ImageBase!T this;
  
  Color opIndex(IndexType x, IndexType y) if (signature);
  Color opIndex(IndexType x, IndexType y) if (is(T:UniformImage)) {
    return UniformImage(this)[y*width+x];
  }
  
  void opIndexAssign(Color v, IndexType x, IndexType y) if (signature);
  void opIndexAssign(Color v, IndexType x, IndexType y) if (is(T:UniformImage)) {
    UniformImage(this)[y*width+x] = v;
  }
}

signature Image(T) {
  alias UniformImage!T this;
  alias IndexedImage!T this;
}
```

__Example usage:__

```D
void rotate(scope Image image, float rotation) { /* ... */ }
void rotate(scope Image image, float rotation) if (is(Image.IndexType == size_t)) { /* ... */ }

void main() {
  HorizontalStorage!RGBA source = HorizontalStorage(/*width*/100, /*height*/100);
  // ... fill?
  source.rotate(80f);
}
```

__Another use case:__

HorizontalStorage has postblit disabled. Yet the image was copied and returned via the heap.
It was moved into a newly GC'd block of memory prior to returning.
If the return type had scope added, it would be malloc'ed and free'd immediately upon the caller went out of scope.

```D
Image myCreator() {
  HorizontalStorage!RGBA source = HorizontalStorage(/*width*/100, /*height*/100);
  // ... fills /some how/
  return source;
}

scope Image myCreator2() {
  HorizontalStorage!RGBA source = HorizontalStorage(/*width*/100, /*height*/100);
  // ... fills /some how/
  return source;
}

void main() {
  Image first = myCreator();
  scope Image second = myCreator2();
  
  /+
    scope(exit)
      free(second.__context);
  +/
}

```

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
