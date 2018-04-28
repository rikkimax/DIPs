# Additions
This part of the DIP adds ``signature`` resolving. Resolving is the process of determining the hidden arguments and producing a resolved ``signature`` type. A resolved signature type is one which has determined the hidden arguments, an unresolved signature type is one which is provided by the source code without the hidden arguments resolved.

## Semantic changes

1. Extension to ``typeof(Signature, Identifier=Value, ...)``. Where Value can be a type or expression. Evaluates out the hidden arguments to a signature.
2. Is expression is extended to support checking if a given type is a signature.
    - If a type is a signature of any type but is unresolved will match ``is(T==signature)``, for resolved signatures they will match ``is(T:signature)``.
    - To verify a resolved signature type against an unresolved signature type use ``is(T:Signature)``, against a resolved signature use ``is(T==typeof(Signature, ...)``.
    - If ``is(T:Signature)`` is evaluated at compile time, it will always return false if the initialization of the given signature is occuring from within another ``is(T:Signature)``. This prevents a diamond dependency problem.

                 implementation
                /              \
            signature ----- signature
                |               |
            Signature       Signature

3. Template Argument is extended to support checking if is signature e.g. ``void foo(IImage:Image)(IImage theImage) {``.
  This will automatically evaluate the argument passed as ``theImage`` into a unresolved ``Image`` who is resolved as an ``IImage``. See ``is(T:Signature)`` for more information.
4. A signature may be used as the return type without resolving the hidden arguments. However it will act as auto with a requirement of it matching ``is(T:Signature)``.
5. Without any hidden arguments supplied, an unresolved signature type must still be resolved, just without any arguments provided to ``typeof``.

# BNF changes

```BNF
Typeof:
    ...
    typeof ( Identifier )
    typeof ( Identifier , Typeof_Signature_Args )

Typeof_Signature_Args:
    Typeof_Signature_Arg
    Typeof_Signature_Arg , Typeof_Signature_Args

Typeof_Signature_Arg:
    Identifier = Expression

EnumDeclaration:
    ...
    enum Identifier ;

AliasDeclarationY:
    ...
    Identifier
```
## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
