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

There have been many conversations on D's NewsGroup attempting to suggest named arguments. For example [1](https://forum.dlang.org/post/khcalesvxwdaqnzaqotb@forum.dlang.org) and [2](https://forum.dlang.org/post/n8024o$dlj$1@digitalmars.com).

Multiple library solutions have been attempted [1](https://forum.dlang.org/post/awjuoemsnmxbfgzhgkgx@forum.dlang.org) and [2](https://github.com/CyberShadow/ae/blob/master/utils/meta/args.d). Both of which do work.

A [DIP](https://wiki.dlang.org/DIP88) (88) was has been drafted, but never PR'd.

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

Named arguments are a fairly popular language feature from dynamic languages which has been very highly requested on D's NewsGroup. It is available in Objective-C so it is also compatibility issue not just an enhancement for D.

## Description

Named arguments are not affected by the passing of unnamed arguments in terms of order. Start, middle or end; it does not matter where they go. So ``func(1, 2, o:=true)`` is the same as ``func(1, o:=true, 2)`` or ``func(o:=true, 1, 2)``.

At the template side, if the template arguments does not have any non-named arguments you may omit the curved brackets. So ``struct Foo(<T>) {}`` is equivalent to ``struct Foo<T> {}``.

If a named argument does not have a default value, it must be assigned or it is an error.
Named arguments can be in any order, they cannot have values depending on each other, default or otherwise.

Named arguments may be specified on structs, classes and unions. As well as functions and methods. When used with structs, classes or unions named arguments may be accessed by their identifier. E.g.

```D
struct MyWorld<string Name> {
}

static assert(MyWorld!(Name:="Earth").Name) == "Earth");

void hello(<string message="world!">) {
	import std.stdio;
	writeln("hello ", message);
}

void main() {
	// Will print "Hello Walter"
	hello(message := "Walter");
}
```

This feature is quite convenient because it means that variadic template arguments and variadic function arguments can come before named arguments. So:

```D
void func(T..., <alias FreeFunc>)(T t, <bool shouldFree=false>) {
}

void abc() {
	func!(FreeFunc := &someRandomFunc)(1, 2, 3, shouldFree := true);
}
```

Is in fact valid.

For convenience at the definition side, you may combine named arguments together or keep them seperate, it is up to you.

```D
struct TheWorld<string Name, Type> {
}

alias YourWorld = TheWorld!(Name:="Here", Type:=size_t);

void goodies(T, <T t>, <U:class=Object>)(string text) {
}

alias myGoodies = goodies!(int, t:=8);
alias myOtherGoodies = goodies!(U:=Exception, string, t := "hi!");
```

Any symbol that has named arguments, the named arguments are not considered for overload resolution. This puts a requirement on the unnamed arguments being unique and easily resolved.

### Grammar changes

```diff
TemplateParameters:
+    < NamedTemplateParameterList|opt >

TemplateParameter:
+    < NamedTemplateParameterList|opt >

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
+     Identifier := TemplateArgument

Parameter:
+   < NamedParameterList|opt >

+ NamedParameterList:
+    Parameter
+    Parameter ,
+    Parameter , NamedParameterList

+ NamedArgument:
+    Identifier := ConditionalExpression
```

## Breaking Changes and Deprecations

No breaking changes are expected.
Angle brackets are not a valid start or end of template parameter or function argument.


## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
