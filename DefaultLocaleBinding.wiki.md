# Introduction

Today, we will always generate a locale named "default", which is used if your user agent asks for an unsupported locale (that is, any locale at all if your app isn't internationalized, or if you internationalize for French Canadian and German but your user asks for Japanese or Italian).  We also have a few generators that want to run in any single locale, which is generally done by running only in default.

So, in that default locale, what should be the currency symbol?  There is no good answer to that or other, similar questions.

What we think we want is a way to specify that a given actual locale---say, "fr\_CA"---should be _used as_ the default locale if no better answer can be negotiated.  We might not even generate a "default" at all.

Doing so, however, turns out to be more interesting than it may first appear.


# Usage Specification

I propose adding a new XML element, `<set-property-fallback name="_propname_" value="_fallbackvalue_"/>`.  I18N.gwt.xml would set the fallback for `locale` to "default" by I18N.gwt.xml, but should be reset by user modules to whatever locale they want to have as their default.

So, I18N.gwt.xml would contain:
```
  <!-- Browser-sensitive code should use the 'locale' client property. -->
  <!-- 'default' is always defined, but might not be set if            -->
  <!-- default.locale is used to specify an actual locale as default.  -->
  <define-property name="locale" values="default" />
  <set-property-fallback name="locale" value="default"/>
```

And my hypothetical app's .gwt.xml would contain:
```
  <extend-property name="locale" values="fr_CA,de" />
  <set-property-fallback name="locale" value="fr_CA" />
```

# Implementation Requirements

The fallback value would be stored in a new filed of `BindingProperty`, with a public getter, default setter, and default value of "".  The interesting changes happen in the two `SelectionProperty` classes, particularly `com.google.gwt.core.ext.linker.SelectionProperty`, which would use a template syntax in the property provider JavaScript and `getPropertyProvider()` would do template substitution, so that a property provider of

```
      while (!__gwt_isKnownPropertyValue("locale",  locale)) {
        var lastIndex = locale.lastIndexOf("_");
        if (lastIndex == -1) {
              locale = "/*-FALLBACK-*/";
          break;
        } else {
          locale = locale.substring(0,lastIndex);
        }
      }
```

Would become, for my hypothetical app,
```
      while (!__gwt_isKnownPropertyValue("locale",  locale)) {
        var lastIndex = locale.lastIndexOf("_");
        if (lastIndex == -1) {
              locale = "fa_CA";
          break;
        } else {
          locale = locale.substring(0,lastIndex);
        }
      }
```

Note that this is a static substitution, at compile time.  The substitution is used in generated the selection script, so a request for locale "jp" will fail to match known property values and be assigned "fr\_CA" instead, as the requested fallback.

Also note that the substitution token is, if somehow seen by naive JavaScript, merely a comment (much as JSNI is for Java code).  In this particular example, moreover, the token is inside a string literal, so a naive interpreter would merely see a very odd-looking (and in fact invalid) locale string.

Other uses might include setting a "default" logging configuration, which user modules could modify.  Not allowed by this scheme would be having a fallback that depended on any other input.  The fallback value _could_ be arbitrary JavaScript, if the property provider were written to execute the extracted value (whether as a string or as non-quoted code).  I see few uses for that, but perhaps the "default" logging might depend on the client's subnet, or on a combination of query inputs for different logging areas.

I also propose to make the fallback value visible to generators, via `com.google.gwt.core.SelectionProperty.getFallback()`.  This is so that the `AbstractResourceImplCreator` can recognize which actual permutation is the "default" one, as some generators might run only in the default locale but users might sensibly use `<set-property/>` to eliminate the literal value "default".

(_Note_: why do we have two such similar classes?  It seems one is for access by linkers, the other by generators, to respective subsets of `BindingProperty` data; is there enough distinction, and should we at least have more differentiated names?)