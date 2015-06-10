# The Problem

Currently, all JSNI methods are required to guarantee that they never return the value undefined. This leads to the following sorts of coersion in native code:

```
native int getFoo() /*-{
  return this.foo || 0;
}-*/;

native Object getBar() /*-{
  return this.bar || null;
}-*/;

native String getBaz() /*-{
  var s = this.baz;
  return (s == null) ? null : s;
}-*/;
```

This is both suboptimal and difficult to get right in practice. Failure to do so (i.e. letting undefined escape into the Java type system) can lead to very difficult-to-debug runtime failures. This is compounded by the fact that different browsers return `null` and `undefined` at different times.

# The Solution
Let both `null` and `undefined` be treated equally in generated Javascript code. If they are treated equally, those writing JSNI methods will no longer be required to coerce all return values to `null` (this will still be necessary for non-String primitive types).

In development mode, this can be dealt with by simply coercing `undefined` return values to `null` in JsValueGlue. In production mode, the generated Javascript code will simply need to ensure that all generated constructs will work properly in the presence of either `null` or `undefined`.

# Details
The only Java expression affected by this change will be identity comparison for Object types (e.g. `Object a,b; if (a == b)`).

Consider the case where `a` is an empty string and `b` in an Integer with value 0.  GWT uses native JavaScript string and numeric types for the respective Java types.  A JavaScript equality comparison would compare `"" == 0`, which is **true** is JavaScript.  One might choose to emit the Java `==` operator as the JavaScript `===` operator, however this is incorrect when you consider the following scenario.

Assume the same Java comparison of `Object a == Object b`.  Suppose `a` had been initialized from a JSNI method with a value of `null` and `b` had been initialized with a value of `undefined`.  The JavaScript equality comparison `null == undefined` is true, while the identity comparison `null === undefined` is false.  This would produce an unexpected result since `undefined` and `null` should be equal as far as the Java semantics are concerned.

The net result of these cases is that the Java object identity operator cannot always be mapped into one JavaScript operator.

The matrix below describes the required comparison operators in each case. We use `==` when possible, because it's the shortest option.  However, `==` cannot be used to compare a string to an object other than a string, because JavaScript will convert the non-string to a string and compare those.  In such cases, `===` is used, and then it is necessary to canonicalize each side of the comparison to replace undefined by null.  That way, if one side is undefined and the other null, after canonicalization they will both be null and thus `===`.


|      |        |        | !null  |        |        | ?null |       |      |
|:-----|:-------|:-------|:-------|:-------|:-------|:------|:------|:-----|
|      |        |Object  | String | Widget | Object | String| Widget| null |
|      | Object | ===    |        |        |        |       |       |      |
|!null | String | ===    | ==     |        |        |       |       |      |
|      | Widget | ===    | false  | ==     |        |       |       |      |
|      | Object | ===    | ===    | ===    | (1)    |       |       |      |
|?null | String | ===    | ==     | false  | (1)    |   ==  |       |      |
|      | Widget | ===    | false  | ==     | (1)    |   (2) | ==    |      |
|      | null   | false  | false  | false  | ==     |   ==  | ==    | true |

```
  µ(O) := ((O === undefined) ? null : O)
  (1)  := µ(O1) === µ(O2)
  (2)  := (O1 == null) & (O2 == null)
```

  * Widget: Any non-String subclass of Object
  * !null: A value that is statically determined to be non-`null`
  * ?null: A value that may or may not be `null`

# Performance
We expect that most comparisons between Objects will be performed using the == operator [TODO: performance metrics on == vs. ===].

The **1 and**2 cases will only be required when the comparison being performed is of one of the following forms:
  * Object == Object
  * String == Object
  * Widget == Object
  * Widget == String

While these cases do occur in practice, they are not the norm -- three of them involve comparisons with untyped (and un-type-tightened) Objects, and the last is extremely odd [TODO: performance metrics on === vs. **1].**

The initial implementation will be performed without knowledge of the !null cases described above. Gathering this information in the compiler will allow many cases to be reverted to the === operator, and in some cases statically evaluated to false.

# Caveats
It will still be necessary for JSNI methods that return non-string primitive values to coerce their return values if there is any chance it may be `undefined`.