## Background
In GWT 1.6, a new module XML element was introduced called `<set-configuration-property>`. Its purpose is to provide to generators and linkers configuration parameters that cannot be appropriately expressed via deferred binding properties,
either because they would never drive multiple permutations or because they have values that aren't enumerable. For example, if a generator needs to be passed a filename at compile-time, deferred binding properties make no sense, but a configuration property is perfect.

However, in a recent design discussion for RPC blacklist support, we found `<set-configuration-property>` a bit deficient. The subsequent sections explain the current behavior, spotlighting the problematic aspects.

### Shared namespace
At present, the namespaces of deferred binding properties and configuration properties are the same. This was intentional, and the original motivations were good:

  * module writers couldn't accidentally create a configuration property and deferred binding property with the same name, causing subsequent ambiguity and
  * it might be useful to transparently change a property from a configuration property into a deferred binding property.

Based on the reasoning above, `PropertyOracle#getPropertyValue` is specified to return either a deferred binding property or a configuration property. Generators can simply query for a property value without knowing whether they are receiving a configuration property or a deferred binding property as an answer.

In retrospect, unifying the namespace creates more problems than it solves.
Although deferred binding properties have a strict identifier-like token syntax, configuration properties do not,
so the idea of swapping a property from one type to the other could break accidental assumptions about the format of a result value of a `getPropertyValue` call in generators.
Additionally, `PropertyOracle#getProperyValueSet` has no relevant meaning for configuration properties,
creating the hard-to-understand behavior that a given property name can succeed in a call to `getPropertyValue`
even though `getPropertyValueSet` with the same property name will throw an exception (that is, in the case where the name refers to a configuration property).

### Single-valued versus multi-valued configuration properties
The original design of configuration properties was supposed to be very simple, similar to Java system properties.
But what happens when a multi-valued configuration property is needed?
In the case of RPC blacklists, we needed to allow multiple modules to contribute to an accumulated value -- the list of types that should not participate in RPC.

There are two main ways to approach this. The most obvious is something like `<append-configuration-property>` which simply appends to the string value, perhaps adding a delimiter.
This has the virtue of being simple and not requiring APIs changes.
However, it pushes extra complications onto generator writers.
Questions such as
  * Should a delimiter be added automatically?
    * If so, which one will make everyone happy (if such a thing exists)?
    * If not, what happens if the string was previously empty, thus creating a bogus initial delimiter?
  * How do we differentiate the concept of 'no value' from the concept of 'has a value that is the empty string'?
    * Can you append a 'no value'?
  * How would we recommend people parse this string within generators?
So, it gets tricky using strings fro multi-valued configuration properties.

## Proposal
To address the above weakness, approximately the following changes are proposed.

1) Deprecate `PropertyOracle#getPropertyValue` and `#getPropertyValueSet`, leaving their behavior unchanged

2) Add `com.google.gwt.core.ext.generator.SelectionProperty`
```
package com.google.gwt.core.ext.generator;
public interface SelectionProperty {
  // The name of the property.
  String getName();

  // The value for the permutation currently being considered.
  String getCurrentValue();

  // In unspecified but consistent order.
  SortedSet<String> getPossibleValues(); 
}
```

3) Add `com.google.gwt.core.ext.generator.ConfigurationProperty`
```
package com.google.gwt.core.ext.generator;
public interface ConfigurationProperty {
  // The name of the property.
  String getName(); 

  // Values are guaranteed to be listed in lexical order (i.e. the last <extends> appears at the end).
  // If the value has never been set or extended, returns the empty list.
  List<String> getValues();
}
```

4) Add methods to `PropertyOracle`
```
public com.google.gwt.core.ext.generator.SelectionProperty getSelectionProperty(String name);
```
and
```
public com.google.gwt.core.ext.generator.ConfigurationProperty getConfigurationProperty(String name);
```

5) Add new module XML element `<define-configuration-property name="the-name" is-multi-valued="true|false"/>`
  * `the-name` uses the same syntax as deferred binding property names
  * `is-multi-valued` can be `true` or `false`; if `false` then `<extend-configuration-property>` will fail during module XML parse
  * For backward compatibility, in 2.0, accept a set without a `define`, but warn about it

6) Add `<extend-configuration-property name="the-name" value="any-string-value"/>`

7) Remove `<append-configuration-property>` which landed in trunk as part of runtime locales

8) Keep `<set-configuration-property name="the-name" value="any-string-value"/>`
  * It replaces all the values with the single specified value

9) Add new module XML element `<clear-configuration-property name="the-name"/>`
  * It puts the config property in exactly the same state as after it is defined but no value set
  * `ConfigurationProperty#getValues` will return an empty list as a result of clearing a config property

10) Fix runtime locales generator to use new config properties behavior