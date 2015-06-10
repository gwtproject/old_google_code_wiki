# Introduction

As of GWT 1.5, it became possible to treat null and undefined as equivalent values (c.f. NullIsUndefined). Because of this, it is no longer necessary to initialize reference types to null, nor is it necessary to explicitly compare reference types against null. This comes up in several different contexts:

`var a = null;` => `var a;`
`_.a = null;` => [nothing](nothing.md) (this comes up frequently in class initialization)
`(a == null)` => `(a)`
`(a != null)` => `(!a)`
`return null` => `return` (or nothing at all, if it's at the end of a function)

# Caveats

This won't work with String types, because of Javascript's tendency to implicitly convert things like `""` and `"0"` to `false` in a boolean context.

# Performance Impact

We've not measured this yet, but it seems very likely to only benefit performance, as there are fewer tokens, and some statements can be eliminated entirely.

# Correctness

The expression `null.nullMethod()` can be observed in compiled code when the compiler has proven that a variable, field, or expression is provably null and a method is invoked on it.  In these cases, the compiler could warn the developer that there is a guaranteed NullPointerException.