# Deferred Bindings for JRE Emulation Classes

## Introduction

Deferred binding is mostly used in GWT to handle browser quirks and to handle internationalization.  Sometimes it is also useful for JRE classes, however.  This page outlines what you need to do to add a browser-specific deferred binding for a user agent.  `StringBuffer`'s deferred binding is used as a running example.


## The Bare Minimum

You should make a class for each implementation strategy you will use.  These classes are called "implementation" classes.  You then access the strategy by making virtual method calls to an implementation class.  If everything is set up right, deferred binding will ensure that the responding method is an appropriate one.  For example, `StringBuffer` starts with a line like this:
```
  private final StringBufferImpl impl = GWT.create(StringBufferImpl.class);
```

In this case, `StringBufferImpl` is the abstract class for string buffer implementations, and there are several subclasses that implement it.  Any part of `StringBuffer` that is browser-specific gets moved to the implementation class and replaced by a call to a method on this `impl` field.  For example, here is the `append` method:
```
  public StringBuffer append(String x) {
    impl.append(data, x);
    return this;
  }
```

The `data` variable in this example might be surprising.  It is described below in Performance Notes.

Because the browser is not always known, you must specify a default deferred binding that is suitable on all browsers.  Add that deferred binding to `Emulation.gwt.xml`.  Here is what it looks like for `StringBuffer`:
```
  <replace-with class="com.google.gwt.core.client.impl.StringBufferImplArray">
    <when-type-is class="com.google.gwt.core.client.impl.StringBufferImpl"/>
  </replace-with>
```

For cases where the user agent is in fact known, add bindings to `EmulationWithUserAgent.gwt.xml`.  Here are the bindings for `StringBuffer`:
```
  <!-- If the user agent is known, use Append as the default StringBuffer implementation. -->
  <replace-with class="com.google.gwt.core.client.impl.StringBufferImplAppend">
    <when-type-is class="com.google.gwt.core.client.impl.StringBufferImpl"/>
  </replace-with>

  <!--  Append is awful on IE.  Use Array instead. -->
  <replace-with class="com.google.gwt.core.client.impl.StringBufferImplArray">
    <when-type-is class="com.google.gwt.core.client.impl.StringBufferImpl"/>
    <any>
      <when-property-is name="user.agent" value="ie6"/>
    </any>
  </replace-with>
```

## Testing

Any class that has a deferred binding like this needs to have its tests run twice: once with the default binding, and once with the browser-specific binding.  The two versions use entirely different code, so they each need to be tested.

You can usually subclass the original class's test case and use a variant on the original test suite's module.  Instead of using the original module, use a new one that inherits from the old one but undoes any deferred bindings that might have taken effect.  For the case of `StringBuffer`, the tests are in `StringBufferTest`, which uses the module `EmulSuite`.  To test as if the browser were not known, there is an additional test class that looks like this:
```
public class StringBufferDefaultImplTest extends StringBufferTest {
  @Override
  public String getModuleName() {
    return "com.google.gwt.emultest.EmulSuiteUnknownAgent";
  }
}
```
The module `EmulSuiteUnknownAgent` inherits from `EmulSuite` but undoes all JRE deferred bindings.  The effect is to run all of the tests of the superclass (`StringBufferTest`), but using the default bindings instead of the browser-specific ones.  Here's what `EmulSuiteUnknownAgent` looks like at the time of writing.
```
<module>
  <inherits name='com.google.gwt.emultest.EmulSuite'/>

  <!--  Remove JRE deferred bindings, so that the default implementations can be tested -->
  <replace-with class="com.google.gwt.core.client.impl.StringBufferImplArray">
    <when-type-is class="com.google.gwt.core.client.impl.StringBufferImpl"/>
  </replace-with>
</module>
```
Assuming the JRE class you are manipulating is tested by `EmulSuite`, then you can  follow this same pattern.  Subclass the relevant test suite, have it use `EmulSuiteUnknownAgent`, and update `EmulSuiteUnknownAgent` to reinstate the default deferred binding for your class.


## Performance Note

If you can avoid having any fields in your implementation class, the compiler can optimize away the instantiation of the object and turn all of its methods into static methods.  That's why the `StringBuffer` example code passes around a data object.  By passing the data object to all methods of the implementation object, the implementation object itself does not need to be instantiated.