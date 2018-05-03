# Shared 2.0

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>                                       |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Adds shared as an attribute.

### Reference

See Q&A session with Walter and Andrei at DConf 2018.

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

Shared as a concept currently is incomplete and ill designed. This DIP adds a replacement design in its place which is both simpler and more coherant.

## Description

Shared is added as an attribute.

As a type qualifier it has no effect unless applied upon a pointer.

Shared may not be applied to functions or methods.
To variable declarations it will put restrictions unto accessing and add automatic atomic operations during access and modification if qualified.

As an attribute applied to variable declarations it acts as the type qualifier like ``const`` would but also includes the behavior of ``__gshared`` in putting it as a global.

When the attribute is applied to a struct or class/interface, it marks it (and for a class/interface its children too) as readable for fields and callable for methods.
If you have a pointer/reference to a class or struct not marked as shared that has type qualified to be ``shared``, it will be inaccessable and no methods including postblit or destructor can be called. Even if postblit is enabled in a struct, but it is not marked as shared when qualified as such, you may not copy it out of the pointer.

Once the attribute has been applied to a class/interface hierachy, all children have it applied automatically. There is no way to get rid of it. In terms of the vtable, the ``shared`` methods acts as if it isn't ``shared`` when overriding the parent. However all methods not overriden or ``abstract`` will be reinitialized (using the parents body and scope) in the child (with shared turned on) to ensure ``shared`` safety still exists in the form of atomic operations and type verification.

### Grammar changes

```diff
StorageClass:
	...
    shared

Attribute
	...
    shared

TypeSpecialization:
	...
    shared

BasicType2X:
	...
	delegate Parameters shared

TypeModifiers:
	...
    Shared
    Shared Const
    Shared Wild
    Shared Wild Const

Shared:
	O
```

### What is now valid:

Atomic loading of a global variable:

```D
shared shared(int) x;

int func() {
	return x+2;
}
```

Classes:
```D
import std.stdio;

interface Root {
	void func();
}

class A : Root {
	void func() {
		writeln("Hi!");
	}
}

shared class B : A {
	void func() {
		super.func();
		writeln("Bye!");
	}
}

void something() {
	shared B b = new shared B;
	b.func(); // ok

	shared A a1 = cast(shared(A))b; // ok, reference is shared but not the implementation
	a1.func(); // error, A isn't marked as shared
	A a2 = cast(A)b; // ok
	a2.func(); // ok matches non-shared reference + non-shared interface

	shared Root r1 = cast(shared(Root))b; // ok, reference is shared but not the implementation
	r1.func(); // error, Root isn't marked as shared
	Root r2 = cast(Root)b; // ok
	r2.func(); // ok matches non-shared reference + non-shared interface
}
```

Structs:

```D
import std.stdio;

struct Foo {
	this(int x) {
		this.x = x;
		writeln("Foo hiiiissss!");
	}

	~this() {
		writeln("Foo byez");
	}

	int x;

	int func(int y) {
		return x + y;
	}
}

shared struct Bar {
	this(int x) {
		this.x = x;
		writeln("Foo hiiiissss!");
	}

	~this() {
		writeln("Foo byez");
	}

	int x;

	int func(int y) {
		return x + y;
	}
}

void meh() {
	Foo* f1 = new Foo(7); // ok
	assert(f1.func(3) == 10); // ok non-shared struct, non-shared reference

	shared Foo* f2 = new Foo(7); // ok because of initializer and using attribute not type qualifier
	assert(f2.func(3) == 10); // error, reference is shared but not the implementation

	shared Bar* b1 = new Bar(7); // ok because of initializer and using attribute not type qualifier
	assert(b1.func(3) == 10); // ok shared struct, shared reference

	Bar* b2 = new Bar(7); // error, type is shared but the reference is not
}
```

Attribute applied and how it affects function pointers:

```D
struct A {
	void func() {}
	static void func2() {}
}

shared struct B {
	void func() {}
	static void func2() {}
}

void bar() {}
shared void bar2() {}

static assert(is(typeof(&(new A).func) == void delegate()));
static assert(is(typeof(&(new B).func) == void delegate() shared));
static assert(is(typeof(&bar) == void function()));
static assert(is(typeof(&bar2) == void function()));
static assert(is(typeof(&A.func2) == void function()));
static assert(is(typeof(&B.func2) == void function()));
```

## Breaking Changes and Deprecations

In the prior DIP ${...} shared was removed as a whole. Given this, no code breakage will occur by this DIP as that has already been done.

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
