# Additions
This part of the DIP adds the signature type. It provides the basic type, but does not describe order of construction and the behavior associated with it.

## Semantic changes

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

# BNF changes

```BNF
Keyword:
    ...
    signature
    
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
```
