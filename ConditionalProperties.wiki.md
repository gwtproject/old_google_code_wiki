# Introduction

Historically, one of the limiting factors in using GWT's deferred binding system is the very real possibility of "permutation explosion."  The compiler will compute the cross-product of every possible combination of binding property values, even in those cases where certain combinations are not useful.

To allow the developer to specify a subset of the total cross product, the `<set-property>` module directive has been extended to allow it to accept `<when-property-is>` conditions and the standard set of combinators `<any>`, `<all>`, and `<none>`.

## Example 1: Macro-like properties

Imagine that you have an application targeted at five browsers and localized to forty languages and would like to enable different features for a certain subset of your users.

This could be written as:
```
<replace-with class="com.example.Feature1Alternate">
  <when-type-is class="com.example.Feature1" />
  <when-property-is name="user.agent" value="safari" />
  <any>
    <when-property-is name="locale" value="en" />
    <when-property-is name="locale" value="fr" />
  </any>
</replace-with>
```

This isn't so bad, unless you want to add additional deferred-bindings based on the same or similar critera, because you would need to duplicate the conditions across every `<replace-with>` or `<generate-with>` rule.

Instead, the complex condition can be expressed as another deferred-binding property:

```
<define-property name="alternateFeatures" values="true,false" />
<!-- Provide a default -->
<set-property name="alternateFeatures" value="false" />

<set-property name="alternateFeatures" value="true">
  <when-property-is name="user.agent" value="safari" />
  <any>
    <when-property-is name="locale" value="en" />
    <when-property-is name="locale" value="fr" />
  </any>
</set-property>

<replace-with class="com.example.Feature1Alternate">
  <when-type-is class="com.example.Feature1" />
  <when-property-is name="alternateFeatures" value="true" />
</replace-with>

<replace-with class="com.example.Feature2Alternate">
  <when-type-is class="com.example.Feature2" />
  <when-property-is name="alternateFeatures" value="true" />
</replace-with>
```

This `alternateFeatures` property is called **derived** because its value can be determined solely from other deferred-binding properties.  There is never a need to execute a property provider for a derived property.  Moreover, derived properties do not expand the permutation matrix and have no deployment cost.

## Example 2: Avoiding permutation explosion

Suppose that you are targeting conventional desktop and WebKit-based mobile devices and need some tweaks here and there to account for device-specific functions.

```
<define-property name="mobile.user.agent" values="android, iphone, not_mobile" />
<property-provider name="mobile.user.agent"><![CDATA[
  {
    var ua = window.navigator.userAgent.toLowerCase();
    if (ua.indexOf('android') != -1) { return 'android'; }
    if (ua.indexOf('iphone') != -1) { return 'iphone'; }
    return 'not_mobile';
  }
]]></property-provider>

<!-- Constrain the value for non-webkit browsers -->
<set-property name="mobile.user.agent" value="not_mobile" >
  <none> <!-- Actually means NOR, in this case "not safari" -->
    <when-property-is name="user.agent" value="safari" />
  </none>
</set-property>
```

Instead of having `[user.agent] * [mobile.user.agent] = 6 * 3 = 18` permutations, this will produce `[user.agent] + [mobile.user.agent] = 9` permutations.  Add forty `locale` values, and the compare `40 * 6 * 3 = 720` to `40 * (6 + 3) = 360` permutations.


## Example 3: Selective feature addition
```
<define-property name="someFeature" values="off, limited, full" />
<property-provider name="someFeature">...</property-provider>

<!-- Default to off -->
<set-property name="someFeature" value="off" />

<!-- set-property can accept multiple values -->
<set-property name="someFeature" value="off, limited, full" >
  <when-property-is name="user.agent" value="safari" />
</set-property>
```