# Value type exceptions

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard (Rikki) Andrew Cattermole <firstname@lastname.co.nz>    |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Value type exceptions is a new mechanism for handling alternative pathways in a code base that do not require heap memory allocations or runtime machinery to perform.
It is designed around a new attribute which describes what exceptions (both class and struct based) that get thrown from within a function.
Infering of this attribute is paramount to prevent excessive writing and complaints by the compiler.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Exceptional pathways in D currently require either explicit types with very little in the way of language assistence or requires runtime information and an unwinding library to assist it.

Both solutions may not be desirable. Instead this DIP aims to fulfill:

- No (heap) memory allocations.
- Uses the standard throw and try catch statements.
- No runtime libraries or information that is not directly offered to the user, this allows it to be appropriete for betterC code and resource constrained systems.
- Does not introduce new syntax that is invasive in all code bases.

Ideally it will allow the removal of ``throw``, ``nothrow`` and default of any child of ``Throwable``.

These restrictions allow for exceptions to work on GPU's in compute situations where exception handling and function pointers do not work.

## Prior Work

A key feature of the ``@throws`` attribute is that it will be typically inferred. This is contrary to languages like Java with its checked exceptions which rely heavily on IDE support to automatically set what goes into the list.

Alternative to exception syntax and mechanisms is that of erroneous returns or optional data. Erroneous returns can take the form of an error code.

The mechanism is implemented as a tagged union. This is the most direct comparison to comparable error methods in terms of performance that should be expected.

This design has some roots in Herb Sutter's Zero-overhead deterministic exceptions paper and further attributes can be seen by Emil Dotchevski modification of the mechanism.

Using the table that Herb Sutter came up with in his paper:

A. “Error” flow is distinct from “success”
    Yes for both raising and handling via throw and catch.
    
B. Error propagation and handling
    Yes for ignored only be explicitly writing a catch, unhandled error propagation is automated, and writing error-preserving error-neurtral function is simple.
    No for unhandled error propagation is visible unless the caller explicitly writes a throws attribute that is non-empty.
    
C. Zero-overhead and determinism
    Yes for no heap (all stack allocated), statically typed no runtime information required apart from the tagged union id, space/time cost equal to return (tagged union, matching other values in original table) and it is fully deterministic (as per other items in the original table).

Some of the criticism of this style of exception handling by Bjarne Stroustrup can be resolved by the infering of the attribute, no one needs to be aware that ``@throws`` attribute exists except when dealing with function pointers or if you do something wrong.

Another designer of exceptional behavior mechanisms is Joe Duffy, who works on Midori, in their [article](https://joeduffyblog.com/2016/02/07/the-error-model/) on error models they covered many different options including aspects of this article. Of particular note is the concept of ``try`` ``else`` expression. Which is used to expressly annotate that an expression throws, and should it throw decide what to do (including assert out). This has benefits over a ``try`` ``catch``, since you will not be able to do other things inside the try statement block.

TODO: all the above really does not read well.

TODO: [Zig](https://ziglang.org/documentation/0.9.1/#Errors)

## Description

This DIP adds a new attribute, trait and a mechanism for throwing and catching exceptions.

The attribute is represented by a set internally to the compiler storing types. Valid types are structs, classes or interfaces that inherit from Throwable.

All functions will infer the attribute automatically, and will error if it does not match the attribute if provided with a set.

```d
int add(int x, int y) {
    return x + y;
}
```

The above example when it does not have the throws attribute added, implies that it needs to be inferred. In the above case it does not throw therefore will have an empty set. Which can be written in code as ``@throws()``.

```d
struct MyException {
    int someData;
    string lastThrow, originalThrow;
}

int add(int x, int y) {
    if (x == y)
        throw MyException(x + y);
}

int add2(int x, int y) {
    try {
        return add(x, y);
    } catch(MyException e) {
        stderr.writeln("Oh noes... something happened: ", e.originalThrow);
        return e.someData;
    }
}
```

In the above example ``add`` will have the attribute set as ``@throws(MyException)``, and ``add2`` will have ``@throws()``. This is because ``add2`` catches the exception in all code paths it can be removed from the set.

While classes are supported by the throws attribute and validation does occur, it is very permissive and should not affect existing code.

Struct based exceptions must be copied not moved, this is in reference to copy constructors and destructors required for member data, memory integrity for use in reference counting. It does not prevent optimizations where the calls are unnecessary for memory consistency.

### Grammar

```diff
AtAtribute:
...
    + @ throws
    + @ throws ( ThrowsArgumentList|opt )
    
+ ThrowsArgumentList:
    + Type
    + Type ,
    + Type , ThrowsArgumentList
    
TraitsExpression:
    + getThrowSet
```

### Attribute

A new attribute is added, ``@throws`` with an optional arguments list. The argument list values, must evaluate to either a class which inherits from ``Throwable`` or a struct.

All functions without the ``@throws`` attribute with an argument list will default to infering the attribute values. The ``nothrow`` attribute is equivalent to ``@throws()`` and is counted as if the user had written the new attribute instead.

Infering will occur as part of semantic analysis regardless of what attributes exist. If the attribute has been set and is not equivalent the compiler will error. This has occured due to programmer error.

Equivalence checks type qualifiers for implicit conversion and in the class based mechanism is both parent and child aware. This means that ``Throwable`` if set in an attribute list will match an ``Exception`` should it be thrown.

If a function body, class or interface virtual method has not been provided it will infer ``@throws(Exception)``, this matches the current behavior and is slated for removal as incorrect behavior.

To get the throw set the trait ``__traits(getThrowSet, symbol)`` can be used, which can be used to generate try catch statements to remove types from the throw set.

It is not an error to provide exception types in the ``@throws`` attribute without a corresponding throw.

### Try Catch

The ``@throws`` attribute is a set. The way to add or remove values is by throwing and catching exceptions.

If you attempt to catch an exception that is not represented in the throws set, a warning should be given. This is a potentially unnecessary path way or worse an undocumented one. If you are using conditional compilation (CTFE generated code, or versions) and you receive this warning, you should explicitly set the throws set with all possible exceptions instead of relying on inferation.

For classes if an exception is represented in the throw set but it is not a parent of or equal to any of the entries, no entries can be removed. Due to potential of unknown children still being able to be thrown.

Example which uses infering and adding plus removing an exception from the set.

```d
int toCall() /* @throws(MyException) */ {
    throw new MyException;
}

int caller() /* @throws() */ {
    int result;

    try {
        result = toCall();
    } catch(MyException) {
        result = 0xDEADBEEF;
    }
    
    return result;
}
```

Catching all exceptions is not a solved problem, but may be solved in the future with pattern matching.

### Mechanism

A new exception mechanism for throwing and catching is introduced in this DIP that leverages the new ``@throws`` attribute.

The return type of all D functions that have a throws set that is non-empty with struct members has been modified to become equivalent to a tagged union. However unlike ordinary tagged unions, the tag value is not based upon an offset into the list of possible choices. Instead it should be based upon an integral value that the compiler can generate based upon the symbol that is independent of where it is being used such as a hash of a fully qualified name.

If a non-``extern(D)`` function has its members in its throw set which was set via inferring and to have struct members in the throws set it is an error. This is due to modifications in the function signature that cannot be represented by other languages.

No syntax changes are required specific to this new mechanism, class based exceptions may have its implementation rewritten to take advantage of this instead of requiring a runtime library to handle throwing and catching.

For structs this meachnism is aware of three members. Of these members, two serve the same purpose, for identifing throwing locations. The third functions as the tag value of the tagged union and accounts for the non-zero size of structs and allows getting the tag used to identifier the type.

The throwing locations members are ``lastThrow`` and ``originalThrow``. If ``originalThrow`` member has been set to non-null and it is being thrown, the ``lastThrow`` member will be set instead. This means that a stack trace if generated will only contain two function calls at most. It cannot be used to create a full stack trace between the original throw and where it was last caught. These members will be set to a string stored in ROM, which should contain the fully qualified module name and line number. An example might be ``"my.mod.ule:102"``.

The tag value uses the member name of ``tag``. It must be either a ``uint`` or a ``ulong`` and it has be the first field. The purpose of this field is to allow structs to be effectively zero sized while copying up the call stack. This may initially be perceived to be useless, but it does have optimization potential. For uncaught value type exceptions, you can use a single else block to rethrow any that do not require copy constructors, destructors or postblits to be called without inspection of the data being rethrown.

If a value type exception is not caught in a function, it will be automatically rethrown by the compiler in the caller function, however it won't change ``lastThrow`` member if it exists.

This mechanism should be kept as cheap as possible, it is meant for catching near to where an exception is being thrown. This is typically used by library authors and may not be appropriete for application authors except in resource constrained systems. It should not require any heap allocations and must guarantee cleanup of any variables within a function scope as it goes up the call stack.

## Breaking Changes and Deprecations

The ``nothrow`` keyword is recommended for deprecation in favor of the ``@throws()`` syntax as it is equivalent.

The ``throw`` attribute is recommended for deprecation in favor of the ``@throws(Exception)`` syntax as it is equivalent.

After two years, the default infering of ``@throws(Exception)`` on all class, interface virtual methods and function pointers is to be deprecated for removal. This incorrect behavior exists to prevent initial code breakage, and allows for it to be defered until code bases have been updated.

Potentially a breaking change for any user of a UDA which is called throws as this should become disallowed.

## Reference

Zero-overhead deterministic exceptions: Throwing values by Herb Sutter: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0709r4.pdf

Zero-Overhead Deterministic Exceptions: Catching Values by Emil Dotchevski: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2232r0.html

C++ exceptions and alternatives by Bjarne Strousrup: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1947r0.pdf

## Copyright & License
Copyright (c) 2022 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
