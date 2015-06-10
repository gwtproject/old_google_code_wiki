# I18N Annotations

John Tamplin

# Introduction
This documents an extension of GWT's i18n capabilities, replacing comment-based metadata with annotations, adding the ability to specify new metadata (such as meanings and parameter examples) with annotations, support for specifying key generation and output file generation, and the ability to embed the default text in the source file.

# Details

## Annotations

These annotations are divided into per-interface, per-method, and per-argument annotations.

### Package

All these annotations are inner classes on `Messages` (those appropriate only for `Messages`), `Constants` (those only for `Constants`/`ConstantsWithLookup`), and `LocalizableResource` (those common to both).  This helps to prevent using the wrong annotation with the wrong type of interface, as well as reducing clutter.

### Interface

  * `@DefaultLocale(String localeName)`

> Specifies that text in this file is of the specified locale.  If not specified, the default is en\_US for compatibility with existing behavior.
  * `@GeneratedFrom(String filename)`

> Indicates that this file was generated from the supplied file.  Note that it is not required that this file name be resolvable at compile time, as this file may have been generated on a different machine etc -- if the generator does check the source file, such as for staleness, it must not give any warning if the file is not present or if the name is not resolvable.
  * `@GenerateKeys(String generatorFQCN)`

> Requests that the keys for each method be generated with the specified generator (see below).  If this annotation is not supplied, keys will be the name of the method, and if specified without a parameter it will default to the MD5 implementation.  The specified generator class must implement the `KeyGenerator` interface.  By specifying a class literal, this will be extensible to other formats not in the GWT namespace -- the user just has to make sure the specified class is on the class path at compilation time.  This allows integration with non-standard or internal tools that may use its own hash function to coallesce duplicate translation strings between multiple applications or otherwise needed for compatibility with external tools.

> A string containing the fully-qualified class name is used instead of a class literal because the key generation algorithm is likely to pull in code that is not translatable, so cannot be seen directly in client code.
  * `@Generate(String[] formatFQCN, String filename, String[] locales)`

> Requests that a message catalog file is generated during the compilation process.  If the filename is not supplied, a default name based on the interface name is used.  The output file is created under the `-out` directory.  The format names are the fully-qualified class names which implement the `MessageCatalogFormat` interface. For example, this could generate an XLIFF or properties file based on the information contained in this file.  Specific `MessageCatalogFormat` implementations may define additional annotations for additional parameters needed for that `MessageCatalogFormat`.

> If any locales are specified, only the listed locales are generated.  If exactly one locale is listed, the filename supplied (or generated) will be used exactly; otherwise `_`_locale_ will be added before the file extension.

> A string containing the fully-qualified class name is used instead of a class literal because the message catalog implementation is likely to pull in code that is not translatable, so cannot be seen directly in client code.

### Method

  * `@DefaultMessage(String)`

> The default text to use if no better locale match is found.  The default text is expected to be in the default locale (see `@DefaultLocale`).  The format of the text is identical to the current properties file format for the Messages interface (specifically regarding `MessageFormat` quoting), except that Java quoting is used instead of property file quoting (ie, \" inside strings vs \-escaping #, !, trailing spaces, etc).  Also, the encoding matches that of the source file, which will typically be UTF8.

> This should be used only in Messages subinterfaces.
  * `@DefaultStringValue(String)` / `@DefaultIntValue(int)` / etc

> The default value to use if no better locale match is found.  The values should be appropriate for the default locale (see `@DefaultLocale`).  The format of text is identical to the current properties file format for `Constants` or `ConstantsWithLookup`, except as above.

> These are separate from the `@DefaultMessage` annotation to make it clear that quoting is handled differently and to have IDEs provide better autocompletion.  An alternative would be to have a single `@DefaultValue` with many optional parameters, such as `@DefaultValue(intValue = 5)` or `@DefaultValue(message="{0} widgets")` etc.  The downside to that approach is coming up with defaults which can be differentiated from actual values supplied to the annotation.  For example, what do you set as the default value of a shortValue.

> These should only be used in `Constants`/`ConstantsWithLookup` subinterfaces.
  * `@PluralText({String pluralForm, String pluralText, String pluralForm, String Pluraltext,...})`

> Specifies the default text for the plural forms of the default locale, quoted as in `@DefaultText`.  The array of string literals must have an even number of values and consists of pairs of plural forms and the text to use for that plural form.  The names of the plural forms is defined by the plural rule being used (see `@PluralCount` below).

> This should be used only in Messages subinterfaces.
  * `@Description(String)`

> A description of the text.  Note that this is not included in a hash of the text and depending on the file format may not be included in a way visible to a translator.
  * `@Key(String)`

> Specifies the key to use in the external format for this particular method.  If not supplied, it will be generated based on the @GenerateKeys annotation above.
  * `@Meaning(String)`

> Supplies a meaning associated with this text.  This information is provided to the translator to distinguish between different possible translations -- for example, orange might have meaning supplied as "the fruit" or "the color".  Note that two messages with identical text but different meanings should have different keys, as they may be translated differently.

### Parameter

As these annotate a parameter to a message method, they are only used with Messages subinterfaces.

  * `@Example(String)`

> An example for this variable.  Many translation tools will show this to the translator in place of the placeholder -- ie, `Hello {0}` with `@Example("John")` will show as Hello John with "John" highlighted to indicate it should not be translated.
  * `@Optional`

> Indicates that this parameter need not be present in all translations.  If this annotation is not supplied, it is a compile-time error if the translated string being compiled does not include the parameter.
  * `@PluralCount(PluralRule rule)`

> Inidicates that this parameter is used to select which form of text to use (ie, 1 widget vs. 2 widgets).  If the rule is not specified, a default plural rule is supplied which should be sufficient for most uses -- you can supply your own PluralRule instance, but this is typically only needed if you are using non-standard locale names.  The argument annotated must be int or short (long is currently not supported completely in GWT), and only one argument may be annotated with `@PluralCount`.

## Related Interfaces

### `KeyGenerator`
```
interface KeyGenerator {
  String generateKey(String className, String methodName, String text, String meaning);
}
```

Implementations are supplied for `MethodNameKeyGenerator` (which just returns methodName), `FullyQualifiedMethodNameKeyGenerator` (which just returns className + "." + methodName), and `MD5KeyGenerator` (which returns the MD5 hash of text and meaning as a hex string).

### `MessageCatalogFormat`
```
interface MessageCatalogFormat {
  void write(AbstractResource resource, PrintWriter out, JClassType messageInterface);
  String getExtension();
}
```

write generates the file (already opened with a `PrintWriter`) in the specified format given the resource and interface.  getExtension returns the extension to use for a file in this format.

Initially, only an implementation for writing Java property files will ship with 1.5.  Future work will add XLIFF and perhaps GNU gettext if there is interest. This API will be extended for reading after 1.5, and the existing properties AbstractResource implementation changed to use this class

**`***`** Note that this API will likely change as we implement more formats.  We need to include a way to generate a properties file for translation, but more work needs to be done to abstract other formats.  I propose that we document in this class that it is likely to change in future versions and that it should only be used with an existing implementation in an `@Generate` annotation.

### `PluralRule`
```
interface PluralRule {
  class PluralForm {
    public String getDescription();
    public String getName();
    public boolean getWarnIfMissing();
  }
  int select(int n);
  PluralForm[] pluralForms();
}
```

pluralForms() returns the list of possible plural forms, and select returns an index into that array for a given count.  For a given plural form, getDescription() returns a human-readable string describing the plural form, name returns a short name (such as "one" or "singular") which will be used in translated files to identify the plural form being translated.  getWarnIfMissing indicates that a compile-time warning should be issued if this plural form is not included for a translation of a plural message (in some languages, certain plural forms are so rarely used it is more useful to allow them to be left out of the translated file than to require them being copied if they are usually the same).

# Examples
```
package foo;

import com.google.gwt.i18n.LocalizedResource.DefaultLocale;
import com.google.gwt.i18n.LocalizedResource.Generate;
import com.google.gwt.i18n.LocalizedResource.GenerateKeys;

@DefaultLocale("en_US")
// Line below defaults to MyMessages_default.xlf, MyMessages_de.xlf, etc
@Generate(format = "com.google.gwt.i18n.rebind.format.Xliff")
@GenerateKeys // defaults to MD5 hash of text and meaning
public interface MyMessages extends com.google.gwt.i18.client.Messages {
  @DefaultMessage("red")
  String red();

  @DefaultMessage("orange")
  @Meaning("the color")
  String orangeColor();

  @DefaultMessage("orange")
  @Meaning("the fruit")
  String orangeFruit();

  @DefaultMessage("Access denied: {0} does not have access to {1}")
  @Description("The specified user does not have access to the specified file")
  String accessDenied(@Example("john.doe") String user, @Example("/etc/passwd") String file);

  @DefaultMessage("The amount due is {0,number,currency}.")
  String amountDue(@Example("$5.00") double amount);

  @DefaultMessage("There are {0} widgets.")
  @PluralText({"one", "There is one widget."})
  String widgetCount(@PluralCount int count); // note @Optional isn't needed as one plural form uses the parameter

  @DefaultMessage("Don''t use '{' and '}' unless quoted") // note the MessageFormat-style quoting
  String quoteWarning(@Optional String extraWarning); // optional since it doesn't appear in this translation but might in others
}
```

...

```
package foo;

import com.google.gwt.i18n.LocalizedResource.DefaultLocale;
import com.google.gwt.i18n.LocalizedResource.Generate;
import com.google.gwt.i18n.LocalizedResource.GenerateKeys;

@DefaultLocale("en_US")
// generated file will be MyConstants_base.properties, since exactly one locale specified
@Generate({format = "com.google.gwt.i18n.rebind.format.Properties"}, fileName = "MyConstants_base", locales = "default")
@GenerateKeys("com.example.MyKeyFormat")
public interface MyConstants extends com.google.gwt.i18.client.Constants {
  @DefaultIntValue(4)
  int decimalDigits();

  @DefaultStringValue("XXX/XXX-XXXX")
  @Meaning("format string for phone numbers")
  String phoneNumberFormat();
}
```

## Properties Files

Property files continue to work as before.  If a key is specified (using `@Key` or the deprecated `@gwt.key` comment) or generated (using `@GenerateKeys`), then the key in the properties file must match.  Plural forms are represented in the properties file with the plural form name in square brackets, such as `widgets[one]=A widget.`  A plural form is not inherited from another locale, since the plural rule may be different (some discussion has taken place suggesting that the plural form is inherited from a parent if the default form is also inherited; however this still runs a risk in a locale such as `pt_BR` which has a different plural rule than `pt` and might still be wrong in such a case -- the current approach is safest but requires more effort on the developer).

If default values are not supplied in the interface source file itself, a default properties (or other format when those are added) file must be included.