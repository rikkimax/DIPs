# Named arguments

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>            |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Adds named arguments to templates and functions.
Named arguments are seperated from unnamed, to allow differentiation and easier access by the user.

### Reference

DIP88

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

There have been many conversations on D's NewsGroup attempting to suggest named arguments. For example [1](https://forum.dlang.org/post/khcalesvxwdaqnzaqotb@forum.dlang.org) [2](https://forum.dlang.org/post/n8024o$dlj$1@digitalmars.com)

## Description

Named arguments do not affect the passing of unnamed arguments in terms of order. Start, middle or end; it does not matter where they go. So ``func(1, 2, o=true)`` is the same as ``func(1, o=true, 2)`` or ``func(o=true, 1, 2)``.

At the template side, if the template arguments does not have any non-named arguments you may omit the curved brackets. So ``struct Foo(<T>) {}`` is equivalent to ``struct Foo<T> {}``.

If a named argument does not have a default value, it must be assigned or it is an error.

Named arguments may be specified on structs, classes and unions. As well as functions and methods. When used with structs, classes and unions named arguments may be accessed by their identifier. E.g.

Named arguments can be in any order, they cannot have values depending on each other, default or otherwise.

```D
struct MyWorld<string Name> {
}

static assert(MyWorld!(Name="Earth").Name) == "Earth");
```

This feature is quite convenient because it means that variadic template arguments and variadic function arguments can come before named arguments. So:

```D
void func(T..., <alias FreeFunc>)(T, bool shouldFree=false) {
}
```

Is in fact valid.

For convenience at the definition side, you may combine named arguments together or keep them seperate, it is up to you.

```D
struct TheWorld<string Name, Type> {
}

alias YourWorld = TheWorld!(Name="Here", Type=size_t);

void goodies(T, <T t>, <U:class=Object>)(string text) {
}

alias myGoodies = goodies!(int, t=8);
alias myOtherGoodies = goodies!(U=Exception, string, T="hi!");
```

When you have a named argument and a variable, the named argument will be preferred. If it does not match, it is an error.

```D
void func(int a, int b = 6) {}

void main() {
	int z;
	func(1, b=2); // named argument
	func(z=3); // assignment
	int b;
	func(2, b=4); // named argument
}

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
+    alias Identifier

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

Using assignment inside a function call, assigns the value to the variable. This could possibly cause problems, but because named arguments are new with preference to it, this should not be an issue. Template arguments assignment isn't valid. 

Angle brackets are not a valid start or end of template parameter or function argument.


## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
