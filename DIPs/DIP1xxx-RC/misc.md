# Additions
This part of the DIP adds miscellaneous things related to signatures. It includes no language syntax changes, but does introduce semantic changes.

## Semantic changes

1. ``.sizeof`` property on signatures, total size of signature instance, works identically to a ``struct``'s ``.sizeof`` property.
2. ``.offsetof`` works on each member field and method (with help of ``__traits(getOverloads)``) of a signature.

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review
