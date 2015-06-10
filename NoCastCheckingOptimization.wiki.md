# Introduction

Typecasts in Java are required to throw a ClassCastException when the cast is incorrect. In GWT compiled code, this results in expressions like `dynamicCast(..., 42)`.

Practically speaking, however, very few applications are written such that an invalid cast is a real possibility. And even when they are, even fewer applications actually **catch** this exception. Allowing cast-checking to be turned off in production code could result in a significant performance improvement.

# Caveats

This is technically incorrect behavior. If the developer is unaware of this optimization, it could turn into a real minefield. It thus **must** be an optional optimization.

# Performance Impact

This can have a significant positive impact on performance, as many checks can be eliminated.