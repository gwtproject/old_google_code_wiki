# Introduction

Virtual methods are added to a class' prototype using a structure like this:

```
function $method0() { ... }
function $method1() { ... }

_ = MyClass.prototype = new SuperClass();
_.method0 = $method0;
_.method1 = $method1;
// etc.
```

There are some cases where a method's implementation is only used by a single class. In these cases, it's not necessary to declare it as a separate top-level function. In this example, if `$method1` is only used by `MyClass`, it could simply be initialized as follows:

```
function $method0() { ... }

_ = MyClass.prototype = new SuperClass();
_.method0 = $method0;
_.method1 = function() { ... }
// etc.
```

# Performance Impact

Likely none.

# Correctness impact

Removal of names from functions would negatively impact stack traces as described in WebModeExceptions.