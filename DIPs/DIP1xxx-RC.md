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

Signatures are a very uncommon language feature originating in the ML family, examples of this are [Ocaml](https://caml.inria.fr/pub/docs/manual-ocaml/moduleexamples.html) and [SML](http://www.smlnj.org/doc/Conversion/modules.html).

A more recent example of signatures is [Rust's](https://doc.rust-lang.org/1.8.0/book/traits.html) traits or [Swift's](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267) Protocols which both focus upon implementing a specification, not a specification matching an existing representation.

Alternative designs to D's structs is C#, which can [inherit interfaces](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/structs) but not other structs or classes.

Supplementary code is provided by the author in the [repository](https://github.com/rikkimax/stdc-signatures).

## Rationale

In the D community we utilize idioms and design patterns which emphasize template usage. Template usage enables high code reuse but it also causes code bloat with quite herrdenous error messages. These supplementary outcomes are not positive for both new and existing users in understanding why a template does not succeed.

At the core of the problem, we aim to pass around types based upon a statically known interface without knowledge of what the implementation is during compilation. It is quite common for when template conditions failure at a function level, to not know if it is because of a function specific specialization failure or a more general interface one.

The goal of signatures in this DIP is to present a language construct that is familiar to the (S)ML developers but also ties into how it would be used by the D community. So in our case, signatures are stack heavy constructs that can be made from either structs or classes. They have a statically created virtual table that may be assigned functions at runtime with patching to make an implementation match the interface. This allows a runtime decision of the implementation but a static compile-time design for what it must describe.

There exist existing methods to do similar things like Design by Introspection. They do not become obsolete instead, they become stronger more useful in customizing behavior using more concrete types. Allowing for cleaner interfaces and most importantly, a focus on concepts not code "versions" of them.

## Description

This DIP is limited to Classes and structs.
A signature may have its instance stored on the stack, heap (via malloc for returning with scope) or heap via already constructed memory.

Signatures by their very nature are dynamic. In this DIP they are always templated and any usage of them are inherently templated.

### Additions

1. The signature type.<br/>
    1. Is implicitly constructed given an appropriate class or struct instance.<br/>
        1. Implicit construction of a signature may occur during assignment or passing as an argument to a function call.<br/>
        2. Implicit construction works in two forms. First as direct assignment and patching for class instances and pointers to structs. Second is a memory allocation and move of the struct instance, should it be copyable. Where it will be assigned and then patched for the given vtable.<br/>
        3. If the implicit construction is going into a scope'd variable, no allocation needs to take place unless it is being returned from a function. If it is being returned from a function it will be malloc'd and then free'd when it goes out of scope.<br/>
        4. Should a struct instance not be going into a scope'd variable (or being returned without scope) it will be allocated into GC owned memory where it will be moved into, should it be copyable. If it cannot be copied, it is an error.<br/>

    2. There may be fields, methods and static functions. Operator overloads are also valid. A signature copies what structs supports here. Attributes that provide protection and storage class allow more restrictive ones into less inside the signature. This allows you to use ``@safe`` and ``@nogc`` as if it wasn't tagged as such. However you may not tag a signature method/field as ``@safe`` and then assign it a ``@system`` method/field.
    3. There are hidden arguments to a signature, these are aliases and enums. Provided by the syntax ``alias Type;`` and ``enum Value;`` or ``enum Type Value;``. Where it is inferred during implicit construction to come from the source type, in the form of ``Source.Type`` or ``Source.Value``. Anonymous enum members, may also be used as an alternative to ``enum Value;`` syntax. If ``alias Type=...;``, ``enum Name=...;`` or ``enum Type Name=...;`` (including anonymous enum form) refers to ``this`` in anyway, then it will come under the hidden argument resolution rules as above.
    4. ``this`` will always refer to the implementation and never the resolved signature instance itself.
    5. A signature may inherit from others, similar in syntax to interfaces in this manner. However the diamond problem is not valid with signatures. If a field/method/enum/alias is duplicated and is similar, it can be ignored. If it is different (e.g. different types or different attributes) then it is an error at the child signature.
    6. A signature may be null. Because of this ``v is null`` works also for signatures instances.
    7. Signatures may be cast down for their inheritence. This can be computed statically and does not require any runtime knowledge. However they may not be cast up again. Rules regarding const, immutabe, shared ext. still apply like any other type.
    8. Method bodies if provided give a "default" implementation should it not be provided by the child. Removes conditionally defining them.
       - If the given method is on the source type, then the body is ignored as normal.
       - If documented with a body this should be treated as documentation for what it should do.
       - The body is initialized in the child type, not in the parent.
         The below code will result in the output of ``Child``'s mangled name, not ``Parent``'s.

         ```D
         signature Parent {
            void func() { __traits(parent, func).mangleof }
         }
         signature Child : Parent {}
         ```

    9. Signatures may not have constructors. But they all have destructors + postblit and will automatically forward to the implementation if it exists. Otherwise they have empty function bodies.
    10. Inside of a signature multiple ``alias this`` acts as additional inheritance to the ones defined at the signature name level. This adds no additional logic associated with ``alias this`` and calls into the existing inheritance support.
    11. Signatures must be specialized when used inside of a signature (e.g. method return/method arg/field). Reference to ones self does not require specialization unless wanting to initiate another hidden parameters. For example the signature ``signature Foo { alias T; Foo func(); }`` is equivalent to ``signature Foo { alias T; typeof(Foo, T=T) func(); }``.
    12. Usage of signatures as return types and arguments obey the same rules associated with interfaces and classes inside said interfaces/classes hierachies which is covariance and invariance. Patch functions will be required to implement this however and have each version available for casting to parent signatures.
   
2. Extension to ``typeof(Signature, Identifier=Value, ...)``. Where Value can be a type or expression. Evaluates out the hidden arguments to a signature.
3. Is expression is extended to support checking if a given type is a signature. Unresolved (the hidden arguments) will match ``is(T==signature)``. Resolved signatures will match ``is(T:signature)`` e.g. ``is(T:typeof(Signature, Type=MyType))``. It is unexpected that a library/framework can do much with the prior. For checking if a type is a specific unresolved signature ``is(T:Signature)`` is to be used e.g. ``is(T:Image)``.

    - Evaluated signatures via ``typeof(Signature, Identifier=Value, ...)`` must be compared using ``is(T==U)``, where U is the evaluated type.
    - The form ``is(Signature2:Signature1)`` will evaluate if Signature2 inherits from Signature1.
    - If ``is(T:Signature)`` is used inside a static if, it will always return false if the initialization of the given signature is occuring from within another ``is(T:Signature)``. This prevents a diamond dependency problem.
    
                 implementation
                /              \
            signature ----- signature
                |               |
            Signature       Signature

4. Template Argument is extended to support checking if is signature e.g. ``void foo(IImage:Image)(IImage theImage) {``.
  This will automatically evaluate the argument passed as ``theImage`` into a unresolved ``Image`` who is resolved as an ``IImage``. See ``is(T:Signature)`` for more information.
5. A signature may be used as the return type without resolving the hidden arguments. However it will act like auto does, only with a requirement of it matching ``is(T:Signature)``.

TODO: describe its vtable nature.

### Breaking changes / deprecation process

The primary breaking change is the token ``signature`` is becoming a keyword.
Existing code that uses it would need to rename existing symbols and variables but nothing requiring major changes.

### BNF changes

```BNF
Keyword:
    ...
    signature

Typeof:
    ...
    typeof ( Identifier )
    typeof ( Identifier , Typeof_Signature_Args )

Typeof_Signature_Args:
    Typeof_Signature_Arg
    Typeof_Signature_Arg , Typeof_Signature_Args

Typeof_Signature_Arg:
    Identifier = Expression
    
TypeSpecialization:
    ...
    signature
    
AggregateDeclaration:
    ...
    SignatureDeclaration
    
SignatureDeclaration:
    signature Identifier AggregateBody
    signature Identifier : SignatureInheritance AggregateBody
    SignatureTemplateDeclaration
    
SignatureInheritance:
    Identifier
    Identifier , SignatureInheritance

SignatureTemplateDeclaration:
    signature Identifier TemplateParameters Constraint|opt : IdentifierList AggregateBody
    signature Identifier TemplateParameters : IdentifierList Constraint|opt AggregateBody

EnumDeclaration:
    ...
    enum Identifier ;
    
AliasDeclarationY:
    ...
    Identifier
```

Where:
- identifier in ``typeof`` refers to an unresolved signature.
- Identifier in ``Typeof_Signature_Arg`` refers to either an ``alias`` or ``enum`` inside the signature.

### Examples

#### Factory method pattern without classes

The [factory method pattern](https://en.wikipedia.org/wiki/Factory_method_pattern) is a very popular OOP design pattern.
But why does it have to class-only? Here is a small example of how it can be done with signatures instead.

```D
/**
 * The base definition of what a factory has.
 *
 * It is unknown what Type you are creating,
 *  so it uses inference via a hidden argument
 *  called Type which is an alias to a type/alias
 *  in the implementation.
 * 
 * It has a method called create, as described by the pattern
 *  and uses the Type specified earlier as the return type.
 * Allowing the implementation define what it is creating and returning.
 * For example it could return a pointer to a struct instead of a class instance.
 */
signature Factory {
    alias Type;
    Type create();
}

/**
 * Creates a new type of factory, which constructs some sort of "room".
 * A room requires a size to be created, but we don't know much more than that.
 */
signature RoomFactory : Factory {
    void setSize(vec2 size);
}

/**
 * A very simple test struct.
 *
 * It contains a size (width and height) and not much else.
 */
struct Room {
    vec2 size;
}

/**
 * The RoomFactory implementation.
 *
 * We don't set that we are a RoomFactory here though.
 * After all, we may not even know that we are a Factory!
 */
struct MyRoomFactory {
    alias Type=Room*;

    vec2 size_;
    void setSize(vec2 size) { size_ = size; }
    Room* create() { return new Room(size_); }
}

/**
 * A sample main function.
 * 
 * Constructs a MyRoomFactory, sets the size of a room and then constructs a room.
 * Not terribly useful by itself.
 *
 * It does however show how to resolve the hidden arguments to a signature.
 */
void main() {
    MyRoomFactory factory = ...;
    setSize(factory);
    myFunc!(typeof(Factory, Type=Room*))(factory);
}

/**
 * Sets the size of a room in a RoomFactory.
 * 
 * Uses the template argument implicit construction syntax.
 */
void setSize(IFactory : RoomFactory)(scope ref IFactory factory) {
    factory.setSize(...);
}

/**
 * Does something with a room that it creates.
 *
 * Doesn't know the implementation at all.
 */
void myFunc(IFactory : Factory)(scope ref IFactory factory) {
    IFactory.Type myRoom = factory.create();
    ...
}
```

#### Image library
The challenge for performance in image libraries, is two-fold. First you either use classes and impose strong costs associated with lookups or use a templated approach which loses the ability to move any implementation around.

An image library using signatures allows to custom many different attributes. In the following example, we assume the color and index type can be changed by the implementation. This allows you to support both canvas-style API in which negative values are valid and other color types.

__Definitions of what an image is:__

There are four signatures defined in the following code. The first is ``ImageBase`` which defines the color, index type, width and height that an image implementation must posses. The next two signatures ``UniformImage`` and ``IndexedImage`` provide variants of what constitutes an image, specifically about how it is indexed. The last signature ``Image`` has no additional members, but defines an image to include both ``UniformImage`` and ``IndexedImage`` indexing. Since both inherit from ``ImageBase`` so does ``Image``.

```D
bool isColor(T) { ... }

signature ImageBase() {
    alias Color;
    alias IndexType;
    
    static assert(isColor!Color, "An image Color must match isColor");
    static assert(isIntegral!IndexType, "An image IndexType must be an integer");

    @property {
        IndexType width();
        IndexType height();
    }
}

signature UniformImage : ImageBase {
    static if (is(typeof(this):IndexedImage)) {
        Color opIndex(IndexType i) {
            // !__traits(compiles, {typeof(this) t; Color c = t.opIndex(IndexType.init);})
            return IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)];
        }
        
        void opIndexAssign(Color v, IndexType i) {
            // !__traits(compiles, {typeof(this) t; t.opIndexAssign(Color.init, IndexType.init);})
            IndexedImage(this)[cast(IndexType)(i % width), cast(IndexType)floor(i / width)] = v;
        }
    } else {
        Color opIndex(IndexType i);
        void opIndexAssign(Color v, IndexType i);
    }
}

signature IndexedImage : ImageBase {
    static if (is(typeof(this):UniformImage)) {
        Color opIndex(IndexType x, IndexType y) {
            // !__traits(compiles, {typeof(this) t; Color c = t.opIndex(IndexType.init, IndexType.init);})
            return UniformImage(this)[y*width+x];
        }

        void opIndexAssign(Color v, IndexType x, IndexType y) {
            // !__traits(compiles, {typeof(this) t; t.opIndexAssign(Color.init, IndexType.init, IndexType.init);})
            UniformImage(this)[y*width+x] = v;
        }
    } else {
        Color opIndex(IndexType x, IndexType y);
        void opIndexAssign(Color v, IndexType x, IndexType y);
    }
}

signature Image : UniformImage, IndexedImage {}
```

Resolution for an ``Image`` takes two phases. First is given type even an ``Image``, and then resolving ``Image`` out.
To determine if a type is an ``Image``, you need to determine three things.

1. Is the type a ``ImageBase``?<br/>
   This should be rather simple and quick to do.
2. Is the type a ``UniformImage``?<br/>
   Is the type a ``IndexedImage``? Ignoring the first static if branch.
    - If yes, is a ``UniformImage``.
    - If no, try the method prototypes, do they match?

3. Is the type a ``IndexedImage``?<br/>
   Is the type a ``UniformImage``? Ignoring the first static if branch.
    - If yes, is a ``IndexedImage``.
    - If no, try the method prototypes, do they match?
4. If it is not a ``UniformImage``<br/>
    If ``IndexedImage`` has been resolved (from first static if branch), it is a ``UniformImage``. Otherwise it isn't.
5. If it is not a ``IndexedImage``<br/>
    If ``UniformImage`` has been resolved (from first static if branch), it is a ``IndexedImage``. Otherwise it isn't.

If the type has been discovered to be``ImageBase``, ``UniformImage`` and ``IndexedImage``, then the type is a ``Image``.
Steps 4 and 5 exist because of the static if branching. In essence while signatures are being resolved and not complete, keep attempting to resolving (without is returning false).

Next take all known hidden arguments and construct a ``typeof`` as ``typeof(Image, Color=Impl.Color, IndexType=Impl.IndexType)``.
This becomes your resolved instance of ``Image``. Where ``Impl`` is your implementation type.

__Example usage:__

In the example usage code, two functions are defined (without the bodies since that is a complex operation). These functions are used to manipulate an image in some form. Of note is that the signature used as an argument is specified as part of the template argument and not implicit. But the creation as it is passed is.

```D
void rotate(IImage:Image)(scope IImage image, float rotation) if (!is(IImage.IndexType == size_t)) { /* ... */ }
void rotate(IImage:Image)(scope IImage image, float rotation) if (is(IImage.IndexType == size_t)) { /* ... */ }

void main() {
    HorizontalStorage!RGBA source = HorizontalStorage(/*width*/100, /*height*/100);
    // ... fill?
    source.rotate(80f);
}
```

Alternatively to implementing rotate using template argument, if you only need one type of the ``Image`` signature you could expand it via ``typeof``. Via:

```D
void rotate(scope typeof(Image, Color=RGBA, IndexType=size_t) image, float rotation) { /* ... */ }

void main() {
    HorizontalStorage!RGBA source = HorizontalStorage(/*width*/100, /*height*/100);
    // ... fill?
    source.rotate(80f);

}
```

In the example above with HorizontalStorage with ``RGBA`` as the color and as defined by it using the ``IndexType`` as size_t.

### Ranges

Ranges are a key idiom in the D community. They are composed of two different kinds, input and output based. These are currently described by using interfaces and templates. This unfortunately has the side effect of severe bloat in the number of template initaitions that occur to accommodate voldermort types and the symbols themselves.

The complixity of ranges as signatures comes from the need to modify inherited methods return type and arguments. This is supported already in class/interface hierachies. The general concept comes from [covariance and invariance](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) and can be used inside of signatures as well.

The below code is a direct translation from the input range hierachy as provided in [``std.range.interface``](https://dlang.org/phobos/std_range_interfaces.html). A package of signature based representations would look vastly different with the goal of providing optional parts with fallbacks instead of being a translation.

```D
signature InputRange {
    alias Type;
    
    Type moveFront();
    
    @property {
        Type front();
        bool empty();
    }
    
    void popFront();
    
    int opApply(scope int delegate(Type));
    int opApply(scope int delegate(size_t, Type));

}

signature InputAssignable : InputRange {
    @property {
        void front(Type newVal);
    }
}

signature ForwardRange : InputRange {
    @property {
        ForwardRange save();
    }
}

signature ForwardAssignable : InputAssignable, ForwardRange {
    @property {
        ForwardAssignable save();
    }
}

signature BidirectionalRange : ForwardRange {
    @property {
        BidirectionalRange save();
        Type back();
    }
    
    Type moveBack();
    void popBack();
}

signature BidirectionalAssignable : ForwardAssignable, BidirectionalRange {
    @property {
        BidirectionalAssignable save();
        void back(Type newVal);
    }
}

siganture RandomAccessFinite : BidirectionalRange {
    @property {
        RandomAccessFinite save();
        size_t length();
    }
    
    alias opDollar = length;
    
    Type opIndex(size_t);
    Type moveAt(size_t);
    RandomAccessFinite opSlice(size_t, size_t);
}

signature RandomAccessAssignable : ForwardAssignable, BidirectionalRange {
    @property {
        RandomAccessAssignable save();
        void back(Type newVal);
    }
}

signature RandomAccessInfinite : ForwardRange {
    @property {    
        RandomAccessInfinite save();
    }
    
    Type moveAt(size_t);
    Type opIndex(size_t);
}
```

Finally output ranges are special compared to input ranges. They don't really have any form of inheritance.
But they do tend to have one special method, the ability to add multiple values ``~=``.
The problem with operator overloads like ``opOpAssign`` is the operation itself is a template argument.
Because of this they cannot be done automatically like other methods. So we patch it manually and provide a fallback into using put.

```D
signature OutputRange {
    alias Type;
    
    void opOpAssign(string op)(Type[] values...) if (op == "~=") {
        static if (__traits(compiles, { this ~= values; })) {
            this ~= values;
        } else {
            foreach(v; values) {
                put(v);
            }
        }
    }
    
    void put(Type);
}
```

An optimization that can be implemented is for any voldermort type going into a signature is to not generate ``TypeInfo``, if it has not been referenced. This potentially will cut down the number of template initaitions down the road in other symbols and prevent the need (potentially) for vtables for classes. An example of this is as follows:

```D
scope InputRange multiplier(IInputRange:InputRange)(scope IInputRange input) {
    struct Voldermort {
        alias Type = IInputRange.Type;
        IInputRange input;
    
        Type moveFront() { assert(0); }

        @property {
            Type front() { return input.front * 2; }
            bool empty() { return input.empty; }
        }

        void popFront() { input.popFront; }

        int opApply(scope int delegate(Type)) { assert(0); }
        int opApply(scope int delegate(size_t, Type)) { assert(0); }
    }
    
    return Voldermort(input);
}
```

In the above example no ``TypeInfo`` will be generated. If ``scope`` is not provided, the above code will still work however because input ranges tend to be on the stack, it will be unsafe and the input could be deallocated at any time causing a segfault.

## FAQ

### Why not extend interfaces?
There are three main differences between an interface and a signature, making for a rather non-direct comparison. First and foremost, interfaces are designed around the fact that they get inherited by other interfaces and classes. This produces a very rigid hierachy with minimal ability to change parts without breaking quite a bit of code. Second, interfaces are fully unaware of their implementations and who inherits from them. This makes them very undesirable if you want an interface to modify how an implementation does something. Lastly, interfaces are very strict in how they work with regards to memory, specifically heap allocated and only via a class.

A signature on the other hand desires to be more of a temporary description of another type instance. It does not describe how it must be in a perfact way, only what it must be able to conform to. While signatures do use inheritance, this is only to build up a description of what an implementation instance must have. It is not building up to something that must be inherited at the implementation side. Fundamentally it allows you to adapt an implementation type to an abstraction of unknown origin.

### Shouldn't this work for arrays and unions?
Sure they could. Unions by themselves are mostly useless and unsafe. Arrays can't have methods. They can be emulated via free-functions with UFCS. But over all it just complicates things for this DIP. It can be added later on should it be desired. In the mean time, structs make excellant wrappers.

### Library solution or language?
Library solutions cannot change the language semantics and provide implicit construction against implementations.
The ability to "hide" that a signature is always templated, is a killer feature i.e.

```D
auto myFunction(IR:InputRange)(IR from) if (is(IR.Type == int)) {
    ...
}
```

Instead of:

```D
auto myFunction(IR)(IR from) if (is(IR.Type == int) && isInputRange!IR) {
    ...
}
```

Less code, clearer purpose and less template initializations over all.

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

Where a method has a type of function, not a delegate. You will need ``extern(C)`` them in the signature to make it work, if it is comming from C.

A word of caution for you is that unless a struct instance is already allocated on the heap and it isn't going into a scope variable then it will error out. Since no GC is available to handle allocation + cleanup.

### But what about...?
There are definitely other ways that this could have been done. The problem is, it needs to be first class citizen along side classes, structs and unions. And not elongated to library like vector types are.

It has to work well with other language features. If you can do it better, show us!

## Appendix

### scope [ref] return idea
This is an idea of how to support the below code, but may not be required if an alternative is available:

```D
scope Signature foo(scope Signature input) {
    return Wrapper(input);
}
```

It can be rewritten as:

```D
void foo(scope Signature input, scope ref Wrapper output) {
    output = Wrapper(input);
}
```

This forces the return value to be pushed into the callers scope on to the stack. Allowing it to be chained.
Because a Signature as a return type acts as ``auto`` with extra validation, you cannot return multiple different types as the return value. So only one hidden rewritten argument is required.

In full it would end in having code similar to:

```D
void caller(int[] input) {
    Foo_Wrapper __fooWrapper;
    foo(input, __fooWrapper);
    Bar_Wrapper __barWrapper;
    bar(__fooWrapper, __barWrapper);
    
    foreach(v; __barWrapper) {
        ...
    }
}
```

Instead of:

```D
void caller(int[] input) {
    foreach(v; foo(input).bar) {
        ///
    }
}
```

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
