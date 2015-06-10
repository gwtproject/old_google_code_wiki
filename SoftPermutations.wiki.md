# Introduction

Historically, every distinct set of deferred-binding type replacements required a separate permutation to be emitted by the compiler.  The indiscriminate use of type replacement would then contribute to permutation explosion.  One technique for [controlling permutation explosion](ControllingPermutationExplosion.md) is to allow certain rebind decisions to be evaluated at runtime, thereby collapsing two or more hard permutations together.

# Use case

A new GWT module directive will be introduced to indicate that certain values of a deferred-binding property should be considered to be equivalent for the purposes of creating module permutations.
```
<collapse-property name="property.name" values="two, or, more, values" />
```

The `values` attribute will optionally accept glob patterns such as:
```
<collapse-property name="locale" values="en_*" />
```
or more simply:
```
<collapse-property name="user.agent" values="*" />
```

It is possible to collapse all properties to decrease the amount of time spend compiling a module for testing purposes by adding the following:
```
<collapse-all-properties />
```

# Example

To address the common use-case of rendering-engine variants where using deferred-binding:
```
<!-- Define an adjunct property -->
<define-property name="webkitVariant" values="none, safari, chrome, android, iphone" />
<!-- Collapse all of its values -->
<collapse-property name="webkitVariant" values="*" />
<!-- Additional property provider to distinguish Chrome from Safari, etc... -->
<property-provider name="webkitVariant">...</property-provider>

<!-- Default value -->
<set-property name="webkitVariant" value="none">
  <none>
    <when-property-is name="user.agent" value="safari" />
  </none>
</set-property>

<!-- Regular rebind rules -->
<replace-with name="com.google.ServiceImplAndroid">
  <when-property-is name="webkitVariant" value="android" />
  <when-type-is name="com.google.Service" />
</replace-with>

<!-- Generators are fine as well -->
<generate-with name="com.google.ChromeServiceGenerator">
  <when-property-is name="webkitVariant" value="chrome" />
  <when-type-is name="com.google.Service" />
</generate-with>
```

Instead of adding an extra `5 * N` factor to the number of hard permutations where `GWT.create(Service.class)` would be replaced with a `new ServiceImpl()`, the `GWT.create()` call is replaced with a call to a factory method, whose implementation looks like:

```
public static Object com_google_Service() {
  switch (CollapsedPropertyHolder.permutationId$) {
    case 0:
      return new ServiceImplAndroid();
    case 1: 
      return new ServiceImplChrome();
  }
  return new ServiceImplSafari();
}
```

Even though there is a runtime aspect to the construction of a new `ServiceImpl` type, the virtual-permutation-specific method dispatch uses standard polymorphic dispatch to the implementation object and is subject to normal compiler optimizations.  This continues to recommend the pattern of assigning the implementation type to a static, final field, e.g. `private static final Service = GWT.create(Service.class)`.

# Virtual permutation id

The virtual permutation id is determined by the bootstrap linker by examining the values returned by the deferred-binding property providers.  The virtual permutation id is passed into the `gwtOnLoad` function.  The mapping of binding property values to the soft permutation id is found in the [SoftPermutation](http://code.google.com/p/google-web-toolkit/source/browse/trunk/dev/core/src/com/google/gwt/core/ext/linker/SoftPermutation.java) objects returned from [CompilationResult](http://code.google.com/p/google-web-toolkit/source/browse/trunk/dev/core/src/com/google/gwt/core/ext/linker/CompilationResult.java).