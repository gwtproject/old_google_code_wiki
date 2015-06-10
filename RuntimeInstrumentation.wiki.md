**Experimental, In Progress**

# Introduction

Some applications need to perform code coverage with respect to code splitting, in order to get a precise idea of which methods are run, in which order, at which times.

A new 'compiler.instrument' deferred binding property is added which allows the compiler to modify each function to log its entry and exit. The precise entry and exit logic that is run will be customizable.

# Details

The following changes are being made to the compiler
  * New deferred binding property compiler.instrument = none, profile, custom
  * New helper class Instrumentation (immortal type) which profiles utility functions for instrumenting functions and profiling.
  * Modified GenerateJavaScriptAST to instrument all Java methods and static methods with the instrumentation during translation.

# How functions are instrumented

Typically, a prototype function in GWT is in the following form:

```
_.methodName = function methodName(args) { ... }
```

After instrumentation, the function will now look like:

```
_.methodName = $profile(function methodName(args) { ... }, "Java Method Signature Name");
```

Likewise, static methods are instrumented as follows:

```
function $staticMethod(args) {
  return $profile(function() {
    // original body
  }, "Java Method Signature Name").apply(this, arguments);
}
```

# Profile Implementation
The instrumentation function looks like this:

```
function $instrument(toInstrument, entry, exit) {
  return function() {
    entry();
    var $tmp = toInstrument.apply(this, arguments);
    exit();
    return $tmp;
  }
}
```

A profile function is then defined which invokes an entry and exit function with the method name that is running,

```
function $profile(toProfile, name) {
  return $instrument(toProfile, function() {
     $wnd.__gwtStatsEvent({ entry data...});
  },
  function() {
     $wnd.__gwtStatsEvent({ exit data ...});
  });
}
```

Thus, the builtin profile functions log function entry and exit using the GWT LightweightMetricsDesign mechanism. The format of the JSON data logs is as follows:

```
{
        moduleName: $moduleName,
        sessionId: $sessionId,
        subSystem: 'profile',
        evtGroup: 'entry',
        millis: Date.now(),
        type: 'method',
        method: methodName
}
```

# Custom Instrumentation
By setting compiler.instrument = custom, and defining configuration properties compiler.instrument.function.entry and compiler.instrument.function.exit, you can provide Javascript expressions to be used in place of the builtin entry/exit functions used for profiling.