# Introduction

The GWT compiler currently mimics the standard Java behavior when calling class initializers. This leads to a number of expressions like this:

```
($clinit_8() , new ClickEvent())
```

However, if we loosened this contract to allow static initializers to be called aggressively on startup, most such expressions could be eliminated. Only those clinit functions involved in circular relationships (i.e., those where two clinit functions call each other) would need to be deferred. We could also inline aggressive clinits into a single initializer, eliminating them as separate top-level functions.

# Caveats

Because this technically violates the Java spec, it would probably be appropriate to make this an optional optimization.

# Performance Impact

This optimization can only improve performance. One caveat to consider is that, because the work performed in clinits is normally amortized over the life of the application, it could slow down startup time for some apps.