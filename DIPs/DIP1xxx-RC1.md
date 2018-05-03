# Remove shared

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Richard Andrew Cattermole <firstname@lastname.co.nz>                                       |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Removes shared attribute/qualifier from the language.

### Reference

See Q&A session with Walter and Andrei at DConf 2018.

## Contents
* [Rationale](#rationale)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Acknowledgements](#acknowledgements)
* [Reviews](#reviews)

## Rationale

Shared as a concept currently is incomplete and ill designed. This DIP removes it so that it can be redesigned properly without the worries of baggage.

## Description

No new syntax is provided, this DIP removes existing syntax and behavior.

Shared static constructors and destructors are not affected by this DIP.

The purpose of this DIP is to remove shared as an attribute, and type qualifier. This includes other syntax which is directly tied to it like the is expression.


### Grammar changes

```diff
StorageClass:
	...
-   shared

Attribute
	...
-   shared

TypeSpecialization:
	...
-   shared

MemberFunctionAttribute:
	...
-   shared

TypeModifiers:
	...
-   Shared
-   Shared Const
-   Shared Wild
-   Shared Wild Const

- Shared:
-   O

TypeCtor:
	...
-   shared

```

## Breaking Changes and Deprecations

This DIP will break a large amount of code.
Breakage is expected and highly desirable as part of redesigning of shared.
There is no resonable way to prevent this, including deprecation cycle or opt-in. All options will have problems. Deprecate, break, replace should be the policy for fixing shared.

This DIP will not be applied without a replacement design ready to be in place.

## Copyright & License

Copyright (c) 2018 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
