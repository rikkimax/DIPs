# Named arguments

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Required.

Short and concise description of the idea in a few lines.

### Reference

DIP88

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

Required.

A short motivation about the importance and benefits of the proposed change.  An existing,
well-known issue or a use case for an existing projects can greatly increase the
chances of the DIP being understood and carefully evaluated.

## Description

Adds named arguments to templates and functions.

Named arguments do not affect the passing of unnamed arguments in terms of order. Start, middle or end and it does not matter where they go. So ``func(1, 2, o=true)`` is the same as ``func(1, o=true, 2)`` and ``func(o=true, 1, 2)``.

At the template side, if the template arguments does not have any non-named arguments you may omit the curved brackets. So ``struct Foo(<T>) {}`` becomes ``struct Foo<T> {}``.

If a named argument does not have a default value, it must be assigned or it is an error.

Named arguments may be specified on structs, classes and unions. As well as functions and methods. When used with structs, classes and unions named arguments may be accsed by their identifier. E.g.

Because named arguments can be in any order, they cannot have values depending on each other.

```D
struct MyWorld<string Name> {
}

static assert(MyWorld!(Name="Earth").Name) == "Earth");
```

For convenience at the definition side, you may combine named arguments together or keep them seperate, its up to you.

```D
struct TheWorld<string Name, Type> {
}

alias YourWorld = TheWorld!(Name="Here", Type=size_t);

void goodies(T, <T t>, <U:class=Object>)(string text) {
}

alias myGoodies = goodies!(int, t=8);
alias myOtherGoodies = goodies!(U=Exception, string, T="hi!");
```

### Grammar changes

```diff
TemplateParameters:
+    TemplateParameter

TemplateParameter:
+    < NamedTemplateParameterList >

+ NamedTemplateParameterList:
+    NamedTemplateParameter
+    NamedTemplateParameter ,
+    NamedTemplateParameter , NamedTemplateParameterList

+ NamedTemplateParameter:
+    Identifier = TemplateParameter

TemplateArgument:
+   NamedTemplateArgumentList

+ NamedTemplateArgumentList:
+     NamedTemplateArgument
+     NamedTemplateArgument ,
+     NamedTemplateArgument , NamedTemplateArgumentList

+ NamedTemplateArgument:
+     Identifier = TemplateArgument

Parameter:
+   Identifier = ConditionalExpression
```

## Breaking Changes and Deprecations

No breaking changes are expected.

Using equals in templates parameters or function call arguments should already be recognised as an error.

Angle brackets are not valid start or end of template parameter or function argument.


## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
