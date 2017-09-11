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
    
        1. Implicit construction of a signature may occur during assignment or passing as an argument to a function call.
        2. Implicit construction works in two forms. First as direct assignment and patching for class instances and pointers to structs. Second is a memory allocation and move of the struct instance, should it be copyable. Where it will be assigned and then patched for the given vtable.
        3. If the implicit construction is going into a scope'd variable, no allocation needs to take place unless it is being returned from a function. If it is being returned from a function it will be malloc'd and then free'd when it goes out of scope.
        4. Should a struct instance not be going into a scope'd variable (or being returned without scope) it will be allocated into GC owned memory where it will be moved into, should it be copyable.
  
    3. There may be fields, methods and static functions. Operator overloads are also valid. A signature copies what structs supports here.
    4. Dynamic alias mapping via ``alias Type;``.
  Alias T set from the source type as ``Source.Type``.
  Only valid within a signature.
    5. Dynamic enum mapping via ``enum Type Value;`` or ``enum Value;``.
      Enum T set from the source type as ``Source.Value``.
      Only valid within a signature.
    6. Use ``this T`` template argument to provide the implementation instance. Must be last. Can be used after ``T...`` argument.
       Can only be assigned by the compiler. Will be ignored by template initialization syntax. Most signatures would not have any template arguments other than ``this T``.
    7. A signature may inherit from others, similar in syntax to interfaces in this manner. However the diamond problem is not valid with signatures. If a field/method/enum/alias is duplicated and is similar, it can be ignored. If it is different (e.g. different types or different attributes) then it is an error at the child signature.
    
2. Extension to ``typeof(Signature, Identifier=Value, ...)``. Where Value can be a type or expression. Evaluates out the hidden arguments to a signature.
 
3. Trait evaluation to get/initialize a template instance of a function for a specific signature initialization.
  ``RT [function|delegate] (Args) pointer = __traits(getSignatureEvaluated, <function/method>, <description>);``
  It provides a function pointer or delegate to a function or method who uses signatures. The function/method passed must have template arguments already evaluated.<br/>
  Example function argument is ``foobar``, for method ``Foo.bar``. If it has template arguments, then they must be passed at this point e.g. ``foobar!int``.<br/>
  The description is in the form of ``Identifier=Value, ..., [return|argument name], ...``. Where Value can be either a type or an expression. e.g. ``__traits(getSignatureEvaluated, foo, Color=RGBA, IndexType=size_t, return)``.
  When you finish a description (except last one), the argument/function name must end in a semicolon.
  The evaluation process is a simple match, all descriptions must match and the descriptions must include every ``alias Type;`` and ``enum Type Value;`` in every signature types.
4. Is expression is extended to support checking for if it matches a signature e.g. ``is(HorizontalImage!RGBA : Image)``.
  See std.traits : TemplateOf for the equivalent template check.
5. Template Argument is extended to support checking if is signature e.g. ``void foo(IImage:Image)(IImage theImage) {``.
  See ``is(T:Signature)`` for more information.
6. Scope attribute on function arguments and return type may be inferred for a signature, if it is the return type or an argument.

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

signature IndexedImage(this T) : ImageBase!T {
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
