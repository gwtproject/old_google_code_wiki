

Object derived from `java.lang.Throwable` are implemented in the same way as any other type derived from `java.lang.Object`.  The normal Java try/catch semantics are implemented, including type-based dispatch in catch blocks.  Furthermore, the runtime libraries will attempt to provide some degree of stack trace data via `Throwable.getStackTrace()`.

# Control flow

The Java statement
```
try {
  throw new FooException();
} catch (BarException t) {
  // Do something
}
```

is implemented in the JavaScript as a typical try/catch block, except that a sequence of type checks are made in the catch block to simulate the normal JVM behavior.  If the exception would be uncaught in the Java code, the exception object is simply re-thrown.

Exceptions thrown by the JS runtime or by external libraries are modeled with a specific `com.google.gwt.core.client.JavaScriptException` type, which serves as a type-wrapper around the original thrown value (which may be of any JavaScript type).

An example of the compiled code is below:

```
try {
  // Object instantiation and JS throw statement
  throw FooException$(new FooException());
}
 catch ($e0) {
  // If $e0 is not a Java-derived object, wrap it in a a JavaScriptException
  $e0 = caught_0($e0);

  // Type-check the wrapped object to simulate JVM dispatch
  if (instanceOf($e0, 5)) {
    // Do something
  }
   else // Rethrow uncaught exceptions (e.g. RuntimeException)
    throw $e0;
}
```

# Stack traces

The instance initializer of `java.lang.Throwable` calls `fillInStackTrace()` which in turn invokes `com.google.gwt.core.client.impl.StackTraceCreator`.  The `StackTraceCreator` type has an inner class `Collector` that is sensitive to deferred-binding decisions.  The Collector type implements two main functions: `collect()` which creates a stack trace based on the current state of execution and `infer(JavaScriptObject e)` which attempts to infer a stack trace from a previously-thrown object and is used by `JavaScriptException`.

Because the call to `fillinStackTrace()` will be inlined into each `Throwable` constructor function, the type of exception can always be inferred from the stack trace.  This is especially useful when `Class` metadata has been removed from the compilation.

As presently implemented, only `StackTraceElement.getMethodName()` returns useful data.  The other methods in `StackTraceElement` return placeholder values.

## Strategies

`StrackTraceCreator.Collector` is sensitive to deferred-biding and various implementations provide the following strategies for gathering the list of methods.

### Safari, IE

  * `collect()` is implemented by crawling `arguments.callee.caller`.  Because the `caller` property returns function objects and not activation records, data returned by this implementation is necessarily incomplete in the case of reentrant functions.
  * `inferFrom()` is not implemented and returns a zero-length array.

### Mozilla, Opera

  * `collect()` raises an exception and passes it to `infer()`.
  * `infer()` is implemented and uses the browser's `e.stack` or `e.message` property to read activation records to produce a reasonably complete stack trace.

# Emulated Stack Data

The GWT compiler can optionally emit JavaScript code to maintain a meta-stack that emulates standard JS dispatch semantics.  This meta-stack can be used to provide stack trace data on browsers that do not provide (complete) stack data for all exception types.  This is implemented as an additional compiler pass that is conditionalized on the value of the `compiler.emulatedStack` deferred-binding property being set to `true`.  Full documentation on the transforms performed is located in the javadoc for [JsStackEmulator](http://code.google.com/p/google-web-toolkit/source/browse/trunk/dev/core/src/com/google/gwt/dev/js/JsStackEmulator.java), but the basic transformations are as follows:
  * The execution stack is modeled as an array that contains references to the JavaScript functions being executed.  There is a global variable that maintains the current stack depth and an optional co-array that provides current file and line number data.
  * A stack frame is pushed on every function entrance.  This involves incrementing the stack depth counter and assigning the current function into the meta-stack array.  The stack depth for the function's invocation is saved in a local variable.
  * Before every last statement in normal flow control, the stack is popped simply by decrementing the stack depth variable.  The last statement is any return statement not in a try/finally block, conditionally the last statement in a finally block whose associated try block contains a return statement, and the last statement in the function.
    * A return statement inside of a try block with an associated finally block simply sets a function-local variable indicating that the stack should be popped at the end of the finally block.
  * When any exception is thrown (Java-derived or a native browser exception), the stack depth is reset after the call to `Exceptions.caught` to the function's local stack index variable.
    * In order to ensure that a native exception's stack data is recorded properly and any subsequent finally block is executed with the correct global stack depth, any try/finally statement without a catch block has a trivial catch block added to ensure the currently-propagating exception has been correctly wrapped, to reset the global stack depth to the local stack index, and then to re-throw the exception.

## Controls

Control properties are defined in [EmulateJsStack.gwt.xml](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/core/EmulateJsStack.gwt.xml) which is inherited via `Core.gwt.xml`.

  * `compiler.stackMode` is a binding property with values:
    * `native`, the default value, which provides no emulated stack trace support. The amount of stack information available is browser and call stack dependent.
    * `strip`, allows for more compact JavaScript code, at the cost of loss and/or corruption of stack trace information, again browser and call stack dependent.
    * `emulated`, provides full stack trace emulation, at the cost of an increase in the compiled JavaScript.
  * `compiler.emulatedStack` is a legacy binding property with values `true` or `false`. Setting the value to `true` causes `compiler.stackMode` to be set to `emulated`.
  * `compiler.emulatedStack.recordLineNumbers` is a boolean configuration property that causes line-number data to be encoded in the emitted JS.
  * `compiler.emulatedStack.recordFileNames` is a boolean configuration property that also includes source filename information in the emitted JS.
    * `recordFileNames` implies `recordLineNumbers`.
    * If you are using a stack trace resymbolization technique (see below), then the use of `recordFileNames` is not required since the symbol maps already contain the file names.

# Resymbolization / Deobfuscation

Method names in your deployed JavaScript code will have been obfuscated and will not be particularly useful as encoded in the module permutations.  To that end, a means for resymbolizing a stack trace has been implemented.

  * Symbol maps can be generated at compile time using the `-extra` GWT compiler argument, e.g. `-extra war/WEB-INF/classes/`
  * [StackTraceDeobfuscator](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/logging/server/StackTraceDeobfuscator.java) provides server side stack trace deobfuscation and is integrated into [GWT logging](http://code.google.com/webtoolkit/doc/latest/DevGuideLogging.html).
  * In order to obtain useful and complete stack traces across all browser permutations, it's sufficent to set `compiler.stackMode` to `emulated` and `compiler.emulatedStack.recordLineNumbers` to `true`, since both the original method name and source filename are available in the symbol maps. It is not necessary to set `compiler.emulatedStack.recordFileNames` to `true`.

Implementation details:

  * The Linker artifact `CompilationResult` provides a method `getSymbolMap()` which returns a string mapping of JSNI identifiers for types, methods, and fields to their obfuscated identifiers.
  * A `symbolMaps` linker is added in `Core.gwt.xml` that emits the symbol map data as a series of files `<permutationStrongName>_symbolMap.properties`.
    * The data emitted by this linker is very raw; one method may exist in several methods due to compiler optimizations.  Consider `ArrayList.add(Object)`:
```
java.util.ArrayList = ArrayList
java.util.ArrayList::$add(Ljava/util/ArrayList;Ljava/lang/Object;) = $add
java.util.ArrayList::add(Ljava/lang/Object;) = add_3
```
    * The first entry `$add` is a static-dispatch variant used when the compiler is able to determine that a variable must be an `ArrayList`.  The second variant `add_3` is used for polymorphic dispatch when the variable would have been accessed through the `List` interface.
  * The method `GWT.getPermutationStrongName()` and the associated `$strongName` variable have been added to allow permutations to identify themselves at runtime.
  * `RemoteServiceServlet` also provides a `getPermutationStrongName()` method, which provides the requesting client's permutation strong name for the current RPC.
  * A utility class `HttpThrowableReporter` has been provided to send JSON-formatted representations via HTTP requests.


# Example Production Mode stack traces without emulation
The amount of built-in support for native stack traces differs greatly by browser. To demonstrate this, set an `UncaughtExceptionHandler` to examine the exception thrown in Production Mode (compiled with `-style OBFUSCATED`) by the following snippet:
```
  String s = null;
  s.length();
```

## Sample results: Firefox 3.6.13
| **Property** | **Value** |
|:-------------|:----------|
| Exception Class | `com.google.gwt.core.client.JavaScriptException` |
| Message      | `(TypeError): null has no properties` |
| `lineNumber` | `657`     |
| `stack`      | `aM()@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:657`<br><code>ll([object Array],[object Array])@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:600</code><br><code>bl([object Object])@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:475</code><br><code>ql()@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:654</code><br><code>hl([object Object])@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:228</code><br><code>Rk(hl,[object XPCCrossOriginWrapper],[object Object])@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:612</code><br><code>([object Object])@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:456</code><br><code>(3)@http://127.0.0.1:8888/stackdump/71307A90B78B5C9E2B2ADEFF5EE31AF0.cache.html:517</code> </tbody></table>

<h2>Sample results: Safari 5.0.3</h2>
<table><thead><th> <b>Property</b> </th><th> <b>Value</b> </th></thead><tbody>
<tr><td> Exception Class </td><td> <code>com.google.gwt.core.client.JavaScriptException</code> </td></tr>
<tr><td> Message         </td><td> <code>(TypeError): Result of expression 'null' [null] is not an object.</code> </td></tr>
<tr><td> <code>line</code> </td><td> <code>669</code> </td></tr>
<tr><td> <code>sourceId</code> </td><td> <code>4665371256</code> </td></tr>
<tr><td> <code>sourceURL</code> </td><td> <code>http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html</code> </td></tr>
<tr><td> <code>expressionBeginOffset</code> </td><td> <code>9137</code> </td></tr>
<tr><td> <code>expressionCaretOffset</code> </td><td> <code>9141</code> </td></tr>
<tr><td> <code>expressionEndOffset</code> </td><td> <code>9144</code> </td></tr>
<tr><td> Stack Trace     </td><td> <no native stack trace information> </td></tr></tbody></table>

<h2>Sample results: Chrome 9</h2>
<table><thead><th> <b>Property</b> </th><th> <b>Value</b> </th></thead><tbody>
<tr><td> Exception Class </td><td> <code>com.google.gwt.core.client.JavaScriptException</code> </td></tr>
<tr><td> Message         </td><td> <code>(TypeError): Cannot call method 'tb' of null</code> </td></tr>
<tr><td> <code>type</code> </td><td> <code>non_object_property_call</code> </td></tr>
<tr><td> <code>arguments</code> </td><td> <code>tb,</code> </td></tr>
<tr><td> <code>stack</code> </td><td> <code>TypeError: Cannot call method 'tb' of null</code><br><code>    at Object.YM [as A] (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:669:9138)</code><br><code>    at rl (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:614:103)</code><br><code>    at hl (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:487:60)</code><br><code>    at Object.wl [as R] (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:666:19024)</code><br><code>    at nl (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:236:25)</code><br><code>    at Xk (http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:623:61)</code><br><code>    at http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:472:45</code><br><code>    at http://127.0.0.1:8888/stackdump/EB4A0B3EC0DB33BFD6A62912052DC4D6.cache.html:528:65</code> </td></tr></tbody></table>

<h2>Sample results: Internet Explorer 6/7/8</h2>
<table><thead><th> <b>Property</b> </th><th> <b>Value</b> </th></thead><tbody>
<tr><td> Exception Class </td><td> <code>com.google.gwt.core.client.JavaScriptException</code> </td></tr>
<tr><td> Message         </td><td> <code>(TypeError): 'null' is null or not an object</code> </td></tr>
<tr><td> <code>number</code> </td><td> <code>-2146823281</code> </td></tr>
<tr><td> <code>description</code> </td><td> <code>'null' is null or not an object</code> </td></tr>
<tr><td> Stack Trace     </td><td> <no native stack trace information> </td></tr></tbody></table>

<h2>Sample results: Opera 11.0</h2>
<table><thead><th> <b>Property</b> </th><th> <b>Value</b> </th></thead><tbody>
<tr><td> Exception Class </td><td> <code>com.google.gwt.core.client.JavaScriptException</code> </td></tr>
<tr><td> Message         </td><td> <code>(TypeError): Cannot convert 'null' to object</code> </td></tr>
<tr><td> Stack Trace     </td><td> <no native stack trace information> </td></tr>