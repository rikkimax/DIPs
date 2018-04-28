# Additions
This part of the DIP adds ``signature`` member function bodies. Method bodies are provided as a fallback for if the implementation is not provided. The default attribute is utilized to change behavior for specific implementations that match other unresolved or resolved signature types.

## Semantic changes

1. Method function bodies may be provided. This includes operator overloads and destructor/postblit. As a fallback if the implementation does not contain it, instead of erroring. The body is initialized at the implementation and not the signature side. 
2. A Method body may be provided, in addition or alternatively you may provide function bodies by using the default attribute to associate a filter to pick which body to use. If multiple match the first to compile will be used, else it will be an error in the implementation.
    Given a chosen method body:
    - If the given method is on the source type, then the body is ignored as normal.
    - The body is initialized in the child type, not in the parent.
      The below code will result in the output of ``Child``'s mangled name, not ``Parent``'s.

         ```D
         signature Parent {
            void func() { __traits(parent, func).mangleof }
         }
         signature Child : Parent {}
         ```

3. The default keyword is made into an attribute of the form ``default(Filters...) { ... }`` where Filters is a comma seperated list of signatures (unresolved) and may have a ``!`` before it to negate it. I.e. ``default(Bird, !Lizard) { return 2; }``. The filters when performing validation ignore other ``default`` attributes i.e. to ignore for all Bird's, in a Lizard use ``default(!Bird)``.
4. ``this`` is no longer an error and will always refer to the implementation and not the resolved signature instance itself. It is only valid in method bodies, variable values, ``alias T`` and ``enum T`` default values.

# BNF changes

```BNF
Attribute:
    ...
    default ( SignatureDefaultArgs|opt ) FunctionBody
    
SignatureDefaultArgs:
    SignatureDefaultArg
    SignatureDefaultArgs , SignatureDefaultArg
    
SignatureDefaultArg:
    !|opt Type
```
## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
