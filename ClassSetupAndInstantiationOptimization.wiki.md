# Introduction

Currently, the compiler generates a separate Javascript constructor function of the form `function ctor() {}`, followed by the Java constructors. This has the effect of often adding one extra empty function per class type, and complicating the instantiation of these types (because a Java `new` statement must often be translated to `javactor(new jsctor())`.

This is somewhat complicated by the fact that Java allows multiple overloaded constructors per class, whereas Javascript can only have one. But because the compiled output doesn't depend upon the Javascript constructor for class identity (which is normally conferred by the single Javascript constructor), there's nothing stopping the compiler from simply emitting multiple Javascript constructors for a given class and assigning them the same Java type id. This would remove many extra functions and simplify object instantiation code throughout.


# Performance Impact

We need to measure this, but expect the effect to be positive.