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
TODO: Rust traits (significantly more limited and basically are interfaces).

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

    1. Is implicitly constructed given an appropriate class or struct instance.<br/>
        1. Implicit construction of a signature may occur during assignment or passing as an argument to a function call.<br/>
        2. Implicit construction works in two forms. First as direct assignment and patching for class instances and pointers to structs. Second is a memory allocation and move of the struct instance, should it be copyable. Where it will be assigned and then patched for the given vtable.<br/>
        3. If the implicit construction is going into a scope'd variable, no allocation needs to take place unless it is being returned from a function. If it is being returned from a function it will be malloc'd and then free'd when it goes out of scope.<br/>
        4. Should a struct instance not be going into a scope'd variable (or being returned without scope) it will be allocated into GC owned memory where it will be moved into, should it be copyable. If it cannot be copied, it is an error.<br/>

    2. There may be fields, methods and static functions. Operator overloads are also valid. A signature copies what structs supports here.
    3. There are hidden arguments to a signature, these are aliases and enums. Provided by the syntax ``alias Type;`` and ``enum Value;`` or ``enum Type Value;``. Where it is inferred during implicit construction to come from the source type, in the form of ``Source.Type`` or ``Source.Value``.
    4. Use ``this T`` template argument to provide the implementation instance. Must be last. Can be used after ``T...`` argument.
        Can only be assigned by the compiler. Will be ignored by template initialization syntax. Most signatures would not have any template arguments other than ``this T``.
    5. A signature may inherit from others, similar in syntax to interfaces in this manner. However the diamond problem is not valid with signatures. If a field/method/enum/alias is duplicated and is similar, it can be ignored. If it is different (e.g. different types or different attributes) then it is an error at the child signature.
    6. A signature may be null. Because of this ``v is null`` works also for signatures instances.
    7. Signatures may be cast down for their inheritence. This can be computed statically and does not require any runtime knowledge. However they may not be cast up again. Rules regarding const, immutabe, shared ext. still apply like any other type.
   
2. Extension to ``typeof(Signature, Identifier=Value, ...)``. Where Value can be a type or expression. Evaluates out the hidden arguments to a signature.
3. Is expression is extended to support checking for if it matches a specific signature e.g. ``is(HorizontalImage!RGBA : Image)``.
  See std.traits : TemplateOf for the equivalent template check.
4. Is expression is extended to support checking if a given type is a signature. Unresolved (the hidden arguments) will match ``is(T==signature)``. Resolved signatures will match ``is(T:signature)``. It is unexpected that a library/framework can do much with the prior.
5. Template Argument is extended to support checking if is signature e.g. ``void foo(IImage:Image)(IImage theImage) {``.
  See ``is(T:Signature)`` for more information.
6. A signature may be used as the return type without resolving the hidden arguments. However it will act like auto does, only with a requirement of it matching ``is(T:Signature)``.
7. Scope attribute on function arguments and return type may not be inferred for a signature, if it is the return type or an argument.

### Breaking changes / deprecation process

The primary breaking change is the token ``signature`` is becoming a keyword.
Existing code that uses it would need to rename existing symbols and variables but nothing requiring major changes.

### Examples

#### Image library

__Definitions of what an image is:__

```D
signature ImageBase() {
    alias Color;
    alias IndexType;

    @property {
        IndexType width();
        IndexType height();
    }
}

signature UniformImage(this T) : ImageBase {
    static if (is(T:IndexedImage)) {
        static if (__traits(compiles, {T t; Color c = t.opIndex(IndexType.init);})) {
            Color opIndex(IndexType i);
        } else {
            Color opIndex(IndexType i) {
                return IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)];
            }
        }
        
        static if (__traits(compiles, {T t; t.opIndexAssign(Color.init, IndexType.init);})) {
           void opIndexAssign(Color v, IndexType i);
        } else {
            void opIndexAssign(Color v, IndexType i) {
                IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)] = v;
            }
        }
    } else {
        Color opIndex(IndexType i);
        void opIndexAssign(Color v, IndexType i);
    }
}

signature IndexedImage(this T) : ImageBase {
    static if (is(T:UniformImage)) {
        static if (__traits(compiles, {T t; Color c = t.opIndex(IndexType.init, IndexType.init);})) {
            Color opIndex(IndexType x, IndexType y);
        } else {
            Color opIndex(IndexType x, IndexType y) {
                return UniformImage(this)[y*width+x];
            }
        }
        
        static if (__traits(compiles, {T t; t.opIndexAssign(Color.init, IndexType.init, IndexType.init);})) {
           void opIndexAssign(Color v, IndexType i);
        } else {
            void opIndexAssign(Color v, IndexType x, IndexType y) {
                UniformImage(this)[y*width+x] = v;
            }
        }
    } else {
        Color opIndex(IndexType x, IndexType y);
        void opIndexAssign(Color v, IndexType x, IndexType y);
    }
}

signature Image(this T) : UniformImage, IndexedImage {}
```

__Example usage:__

```D
void rotate(IImage:Image)(scope IImage image, float rotation) if (!is(IImage.IndexType == size_t)) { /* ... */ }
void rotate(IImage:Image)(scope IImage image, float rotation) if (is(IImage.IndexType == size_t)) { /* ... */ }

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

## FAQ

### Why not extend interfaces?
Interfaces are designed to list methods that must be implemented. In some languages they may even list fields. However they are extended and manipulated. Signatures are on the other hand, extend and manipulate a given type into a new form. They are not only fully aware of the implementation, but instead are free to manipulate it into a very nice and usable form by the user.

### Shouldn't this work for arrays and unions?
Sure they could. Unions by themselves are mostly useless and unsafe. Arrays can't have methods. They can be emulated via free-functions with UFCS. But over all it just complicates things for this DIP. It can be added later on should it be desired. In the mean time, structs make excellant wrappers.

### Isn't this costly?
No! If the implementation type is a struct and passed to a scope variable, it is nearly free. The most expensive thing is copying some pointers over into a region of the stack.

For everything else, it gets more expensive what with a memory allocation either malloc'd or GC'd. Of course you could do all that and only pay the price of the vtable creation.

At the current time it is expected that storage of signature instances will not occur on the heap. However it is supported like all of D's types.

### Does this work with -betterC?
Yes! It will also work from C!

The ABI is as follows:

```D
struct Signature {
    void* __context;
    Fields... fields;
    Methods... methods;
}
```

Where a method has a type of function, not a delegate. You will need ``extern(C)`` them in the signature to make it work however if it is comming from C.

A word of caution for you is that unless a struct instance is already allocated on the heap and it isn't going into a scope variable then it will error out. Since no GC is available to handle allocation + cleanup.

### But what about...?
There are definately other ways that this could have been done. The problem is, it needs to be first class citizen along side classes, structs and unions. And not elongated to library like vector types are.

It has to work well with other language features. If you can do it better, show us!

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
