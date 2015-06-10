**Documentation for the [GWT 2.0 release of CssResource](http://code.google.com/webtoolkit/doc/latest/DevGuideClientBundle.html) can be found on the GWT Developer's Guide website.**



See also the CssResourceCookbook and StyleInjector.

# Goals

  * Primary
    * Compatibility with non-GWT-aware CSS parsers (i.e. any extensions should be valid CSS syntax)
      * This does not imply that the stylesheet would necessarily make sense if you just displayed it in a browser
    * Syntax validation
    * Minification
    * Leverage GWT compiler
      * Different CSS for different browsers, automatically
      * Static evaluation of content
  * Secondary
    * Basic CSS Modularization
      * Via dependency-injection API style
      * Widgets can inject their own CSS only when it's needed
    * BiDi (Janus-style?)
    * CSS image strips
    * "Improve CSS"
      * Constants
      * Simple expressions
  * Tertiary
    * Runtime manipulation (StyleElement.setEnabled() handles many cases)
    * Compile-time class-name checking (Java/CSS)
    * Obfuscation


# Non-Goals
  * Server-side manipulation
    * All features in CssResource must be implemented with compile-time and runtime code only.  No features may depend on runtime support from server-side code.

# Overview

  1. Write a CSS file, with or without GWT-specific extensions
  1. If GWT-specific extensions are used, define a custom subtype of CssResource
  1. Declare a method that returns CssResource or a subtype in an ClientBundle
  1. When the bundle type is generated with `GWT.create()` a Java expression that evaluates to the contents of the stylesheets will be created
    * Except in the simplest case where the Java expression is a string literal, it is generally not the case that a CSS file could be generated into the module output
  1. At runtime, call `CssResource.ensureInjected()` to inject the contents of the stylesheet
    * This method is safe to call multiple times, as subsequent invocations will be a no-op
    * The recommended pattern is to call `ensureInjected()` in the static initializer of your various widget types

# Features

## Constants
(**Working**)
```
@def small 1px;
@def black #000;
border: small solid black;
```
  * The parse rules make it difficult to use delimiting tokens for substitutions
  * Redefining built-in sizes allows users to write plain CSS to draft a style and then tweak it.
  * Suggest that users use upper-case names, similar to static final members.
  * Any legal property value or expression may be used with `@def`
  * `@def` rules that define a single numeric value may be accessed in a manner similar to obfuscated class names by defining an accessor method on the CssResource type that returns a primitive numeric value.
```
interface MyResources extends CssResource {
  int small();
}
```
    * Calling `small()` would return the value `1`.
  * `@def` rules can be accessed as a String as well.  You can retrieve the two definitions above with:
```
interface MyResources extends CssResource {
  String small(); 
  String black();
}
```
    * Calling `small()` returns "1px"
    * Calling `black()` returns "#000"
  * The Generator will not allow you to declare an `@def` rule with the same name as a class, unless you annotate method to retrieve the class with the `@ClassName` annotation.
```
@def myIdent 10px;
.myIdent {
  ...
}

interface MyResources extends CssResource {
  String myIdent();

  @ClassName("myIdent")
  String myIdentClass(); 
}
```
    * Calling `myIdent()` returns @def value "10px"
    * Calling `myIdentClass()` returns the obfuscated class name for .myIdent

## Runtime substitution
(**Working**)
```
@eval userBackground com.module.UserPreferences.getUserBackground();
div {
  background: userBackground;
}
```
  * Provides runtime support for evaluating static methods when the stylesheet is injected.  Triggered / dynamic updates could be added in the future if we allow programmatic manipulation of the style elements.
    * If the user-defined function can be statically evaluated by the compiler, then the implementation of the specific CssResource should collapse to just a string literal.
  * This allows easy support for non-structural skinning changes.

## Value function
```
.myDiv {
  offset-left: value('imageResource.getWidth', 'px');
}
```

  * The `value()` function takes a sequence of dot-separated identifiers and an optional suffix.  The identifiers are interpreted as zero-arg method invocations, using the interface passed to GWT.create() as the root namespace.  By only allowing zero-arg methods, there's no need to attempt to perform type checking in the Generator.  The only validation necessary is to ensure that the sequence of methods exists.  There may be arbitrarily many identifiers in the chain.
  * The `value()` function may be combined with `@def`
```
@def SPRITE_WIDTH value('imageResource.getWidth', 'px')

.selector {
  width: SPRITE_WIDTH;
}
```

## Literal function
(**Working**)

Some user agents make use of property values that do not conform to the CSS grammar.  The `literal()` function exists to allow these non-standard property values to be used.

```
div-with-literal {
  top: literal("expression(document.compatMode==\"CSS1Compat\" ? documentElement.scrollTop : document.body.scrollTop \\ 2)");
}
```

Note that it is necessary to escape the backslash (`\`) and double-quote (`"`) characters.

## Conditional CSS
(**Working**)
```
/* Runtime evaluation in a static context */
@if (com.module.Foo.staticBooleanFunction()) {
  ... css rules ...
}

/* Compile-time evaluation */
@if <deferred-binding-property> <space-separated list of values> {
  ... css rules ...
}
@if user.agent safari gecko1_8 { ... }
@if locale en { ... }

/* Negation is supported */
@if !user.agent ie6 opera {
  ...
}

/* Chaining is also supported */
@if (true) {
} @elif (false) {
} @else {
}
```
  * This allows for more advanced skinning / theming / browser quirk handling by allowing for structural changes in the CSS.
  * The contents of an @if block can be anything that would be a top-level rule in a CSS stylesheet.
  * @if blocks can be arbitrarily nested.
  * What does it mean to have an @def or @eval in an @if block?  Easy to make this work for property-based @if statements; would have to generate pretty gnarly runtime code to handle the expression-based @if statement.  Could have block-level scoping; but this seems like a dubious use-case.
  * If the function in the first form can be statically evaluated by the compiler in a permutation, there is no runtime cost.  The second form will never have a runtime cost because it is evaluated during compilation.

## Image Sprites
(**Working**):
```
@sprite .mySpriteClass {gwt-image: "imageAccessor"; other: property;} => generates =>
  .mySpriteClass {
    background-image: url(gen.png);
    background-position: ...;
    width: 27px;
    height: 42px;
    other: property;
  }
```
```
interface MyCssResource extends CssResource {
  String mySpriteClass();
}

interface MyResources extends ClientBundle {
  @Source("my.css")
  MyCssResource css();

  @Source("some.png")
  ImageResource imageAccessor();

  @Source("some.png")
  @ImageOptions(repeatStyle=RepeatStyle.Horizontal)
  ImageResource repeatingImage();
}
```
  * @sprite is sensitive to the FooBundle in which the CssResource is declared; a sibling ImageResource method named in the @sprite declaration will be used to compose the background sprite.
  * @sprite entries will be expanded to static CSS rules, possibly with data: urls.
  * The expansion is sensitive to any RepeatStyle value defined on the ImageResource accessor function.  The appropriate `repeat-x` or `repeat-y` properties will be added to the @sprite selector.
  * Any CSS selector can be specified for @sprite.
  * Full support for transparency in IE6 isn't feasible in this format, because structural changes to the DOM are necessary to implement a "windowing" effect.  Once it's possible to distinguish ie6 and ie7 in user.agent, we could revisit support for ie6.  In the current implementation, the ie6 code won't render correctly, although is a purely cosmetic issue.
  * Combining `@sprite` with other properties that affect the size of the selected element's CSS [padding box](http://www.w3.org/TR/CSS21/box.html#box-dimensions) will likely cause rendering issues if the ImageResource is a composite image, because the [background position](http://www.w3.org/TR/CSS21/colors.html#propdef-background-position) is relative to the padding box.  Specifically, avoid the `width`, `height`, and `padding` properties.  If the ImageResource has been given a `repeatStyle` in its `@ImageOptions` annotation, it is safe to adjust the size of the padding box along the `repeatStyle` axis.

## References to Data Resources
(**Working**)
```
/* @url <constant name> <DataResource method name> */
@url myCursorUrl fancyCursorResource;

.myClass {
  cursor: myCursorUrl, pointer;
}
```
```
interface MyResources extends ClientBundle {
  @Source("myCursor.cur")
  DataResource fancyCursorResource();

  @Source("my.css")
  CssResource css();
}
```
  * The identifier will be expanded to `url('some_url')` based on the return value of `DataResource.getUrl()`.

## RTL support
(**Working**)
  * CssResource supports automatic transformations of CSS code into a right-to-left variant at compile time.
  * The use of the RTL variant is keyed by `com.google.gwt.i18n.client.LocaleInfo.getCurrentLocale().isRTL()`
  * Transformations applied:
    * The `left` and `right` properties are flipped.
    * Any properties that have values `left` or `right` are flipped: `clear float text-align page-break-before page-break-after`
    * The `background`/`background-position` property is flipped. Attachments expressed in percentage points are mirrored: 40% becomes 60%
    * `margin padding border-color border-style` and `border-width` four-valued properties are flipped: `1px 2px 3px 4px` becomes `1px 4px 3px 2px`
    * Any `xyz-right` or `xzy-right-abc` property is flipped to `xzy-left` or `xzy-left-abc`
    * The `direction` property on a `body` selector will be flipped from `ltr` to `rtl`; on any other selectors, the `direction` property is unchanged
    * When the cursor property has an `resize` value, it will be flipped: `ne-resize` becomes `nw-resize`
  * Sections of CSS can be exempted from automatic flipping by enclosing it in a `@noflip` block:
```
@noflip {
  .selector {
    left: 10;
  }
}
```
  * A `background` property value that uses pixel-based offsets, such as `background-position: 4px 10px;` will not be transformed automatically.
    * The four-valued CSS3 `background-position` property will be automatically flipped by the RTL support
```
background-position: left 4px top 10px;
```
    * For CSS2 browsers, it will be necessary to use an `@sprite` rule:
```
@sprite .bgImage {
  gwt-image: 'background-image';
  position: absolute;
  left: 4px;
  top: 10px;
}
```
  * `ImageResources` can be automatically flipped in RTL contexts via the use of the `@ImageOptions` annotation:
```
@Source("icon128.png")
@ImageOptions(flipRtl = true)
ImageResource logo();
```
  * [Current auto-RTL test cases](http://google-web-toolkit.googlecode.com/svn/trunk/user/test/com/google/gwt/resources/rg)

## Selector obfuscation
(**Working**):
```
java:
    class Resources {
      MyCssResource myCssResource();
    }
    interface MyCssResource extends CssResource {
      Sprite mySpriteClass();
      String someOtherClass();
      String hookClass();
    }
    myWidget.addStyleName(resource.mySpriteClass());

css:
    @sprite mySpriteClass mySpriteImage;
    .someOtherClass {
      /* ... */
    }
    .hookClass{} /* Empty and stripped, but left for future expansion */
```
  * The function just returns the CSS class name, but verifies that the CSS class exists in the stylesheet.
  * Accessing class names through the interface ensures that there can be no typos in code that consumes the CssResource.
  * For obfuscation, we'll use a Adler32 checksum of the source css file expressed in base36 as a prefix (7 chars). The developer can override this with the `CssResource.obfuscationPrefix` deferred-binding property.
    * `<set-configuration-property name="CssResource.obfuscationPrefix" value="empty" />` can be used for minimal-length selector names, but this is only recommended when the GWT module has total control over the page.
  * The `@external` at-rule can be used to selectively disable obfuscation for named selectors; see [additional detail](#External_and_legacy_scopes.md).

# Unplanned use cases

These use cases are not planned, but remain here for the sake of completeness.

## Compile-time import
```
@import cssResourceFunctionName;
```
  * What does this mean?
    * Is this a preprocessor-like #include?  If so, it is problematic to have an n:1 relationship?
    * Is this like a Java static import where we just inherit metadata like @def or @url?
  * To what extent is this actually necessary?  The point of the @import statement is to allow for source and deployment modularization.  Source modularization can be achieved by using multiple CssResource interfaces and we're only concerned with static resources in the Bundle.
  * The @import statement will only work for other CssResources, not for URLs at runtime, since the `.gwt.xml` or StyleInjector can be used in those cases.

**Conclusion:** too confusing.  The modularity afforded by CSS's `@import` statement is inherent in the CssResource design.

## Implementation fix-ups
(~~Not planned~~):
```
    div:hover { }
       Could use top-level event handler to fake on IE
    div:focus { }
       Problematic -- focus/blur don't bubble
```

**Conclusion:** technically possible, but likely unaffordable at runtime.  The rendering engines that don't provide the wanted features in the first place likely don't provide the means to implement those missing features efficiently.

# Optimizations

## Basic minification

Basic minification of the CSS input results in the minimum number of bytes required to retain the original structure of the input.  In general, this means that comments, unnecessary whitespace, and empty rules are removed.

```
.div {
  /* This is the default background color */
  background: blue;
}
.empty {}
```

would be transformed into

```
.div{background:blue;}
```

## Selector merging

Rules with identical selectors can be merged together.

```
.div {prop: value;}
.div {foo: bar;}
```

becomes

```
.div {prop:value;foo:bar;} 
```

However, it is necessary that the original semantic ordering of the properties within the CSS is preserved.  To ensure that all selector merges are correct, we impose the restriction that **no rule can be promoted over another if the two rules define a common property**.  We consider `border` and `border-top` to be equivalent properties, however `padding-left` and `padding-right` are not equivalent.

Thus

```
.a {background: green;}
.b {border: thin solid blue;}
.a {border-top: thin solid red;}
```

cannot be merged because an element whose CSS class matches both `.a` and `.b` would be rendered differently based on the exactly order of the CSS rules.

When working with `@if` statements, it is preferable to work with the form that operates on deferred-binding properties because the CSS compiler can evaluate these rules statically, before the merge optimizations.  Consider the following:

```
.a {
  background: red;
}

@if user.agent safari {
  .a {
    \-webkit-border-radius: 5px;
  }
} @else {
  .a {
    background: url('picture_of_border.png');
  }
}
```

In the safari permutation, the rule becomes `.a{background:red;\-webkit-border-radius:5px;}` while in other permutations, the `background` property is merged.

## Property merging

Rules with identical properties can be merged together.

```
.a {background: blue;}
.b {background: blue;}
```

can be transformed into

```
.a,.b{background:blue;}
```

Promotion of rules follows the previously-established rule of not promoting a rule over other rules with common properties.

# Levers and Knobs

  * The configuration property `CssResource.style` may be set to `pretty` which will disable class-name obfuscation as well as pretty-print the CSS content.  Combine this with a `ClientBundle.enableInlining` value of `false` to produce a CSS expression which is amenable to client-side editing.
  * The configuration property `CssResoure.mergeEnabled` can be set to `false` to disable modifications that re-order rules.  This should be considered a temporary measure until the merge logic has been fully vetted.
  * The configuration property `CssResource.reservedClassPrefixes` can be extended to prevent the CssResource obfuscator from allocating class names that conflict with external CSS libraries.
```
<!-- This is included in Resources.gwt.xml -->
<extend-configuration-property name="CssResource.reservedClassPrefixes" value="gwt-" />
```
  * To allow for client-side tweaking of the effective (i.e. permutation-specific) style rules, you can store the value of CssResource.getText() into a TextArea.  Wire some UI action to pass the contents of the TextArea into `StyleInjector.setContents()` to overwrite the original, injected stylesheet.

# Selector obfuscation details

## Strict scoping

In the normal case, any class selectors that do not match String accessor functions is an error.  This behavior can be disabled by adding a `@NotStrict` annotation to the CSS accessor method.  Enabling `@NotStrict` behavior is only recommended for applications that are transitioning from external CSS files to CssResource.

```
interface MyCssResource extends CssResource {
  String foo();
}

interface Resources {
  @Source("my.css")
  @CssResource.NotStrict
  MyCssResource css();
}
```
```
/* This is ok */
.foo {}

/* This would normally generate a compile error in strict mode */
.other {}
```

## Scope
Scoping of obfuscated class names is defined by the return type of the CssResource accessor method in the resource bundle.  Each distinct return type will return a wholly separate collection of values for String accessor methods.

```
interface A extends CssResource {
  String foo();
}

interface B extends A {
  String foo();
}

interface C extends A {
  String foo();
}

interface D extends C {
  // Intentionally not defining foo()
}

interface Resources {
  A a();
  A a2();
  B b();
  C c();
  D d();
  D d2();
```

It will be true that a().foo() != b().foo() != c().foo() != d().foo().  However, a().foo() == a2().foo() and d().foo() == d2().foo().

## Shared scopes

In the case of "stateful" CSS classes like `focused` or `enabled`, it is convenient to allow for certain String accessor functions to return the same value, regardless of the CssResource type returned from the accessor method.

```

@Shared
interface FocusCss extends CssResource {
  String focused();
  String unfocused();
}

interface A extends FocusCss {
  String widget();
}

interface B extends FocusCss {
  String widget();
}

interface C extends B {
  // Intentionally empty
}

interface Resources {
  A a();
  B b();
  C c();
  FocusCss f();
}
```

In this example, a().focused() == b().focused() == c().focused == f().focused().  However, a().widget() != b().widget != c.widget(), as in the previous example.

The short version is that if distinct CSS types need to share obfuscated class names, the CssResource subtypes to which they are attached must share a common supertype that defines accessors for those names and has the `@Shared` annotation.

## Imported scopes

The Java type system can be somewhat ambiguous when it comes to multiple inheritance of interfaces that define methods with identical signatures, although there exist a number of cases where it is necessary to refer to multiple, unrelated CssResource types.  Consider the case of a Tree that contains Checkboxes.

```
@ImportedWithPrefix("tree")
interface TreeCss extends CssResource {
  String widget();
}

@ImportedWithPrefix("checkbox")
interface CbCss extends CssResource {
  String widget();
}

interface MyCss extends CssResource {
  String other();
}

interface Resources extends ClientBundle {
  @Import({TreeCss.class, CbCss.class})
  MyCss css();
}
```
```
/* Now we can write a descendant selector using the prefixes defined on the CssResource types */
.tree-widget .checkbox-widget {
  color: red;
}

.other {
  something: else;
}
```

Composing a "TreeCbCss" interface would be insufficient because consumers of the TreeCss interface and CbCss interface would receive the same value from the widget method.  Moreover, the use of just `.widget` in the associated CSS file would also be insufficient without the use of some kind of class selector prefix.  The prefix is defined on the CssResource type (instead of on the CssResource accessor method) In the interest of uniformity across all CSS files that import a given scope.  It is a compile-time error to import multiple classes that have the same prefix or simple name.

The case of shared scopes could be handled solely with importing scopes, however this form is somewhat more verbose and relationships between unrelated scopes is less common than the use of stateful selectors.

### Example: StackPanel inside a StackPanel

This is a use-case that is currently impossible to style correctly in GWT.

```
// Assume this interface is provided by the UI library
interface StackPanelCss extends CssResource {
  String widget();
  // and many more class names
}

// App code defines the following interfaces:

@ImportedWithPrefix("inner")
interface StackPanelInner extends StackPanelCss {
  // Empty interface
}

interface StackPanelOuter extends StackPanelCss {
  // Empty interface
}

interface Resources {
  @Source("stackPanel.css")
  StackPanelInner inner();

  @Import(StackPanelInner.class)
  @Source("stackPanel.css", "outer.css")
  StackPanelOuter outer();
}
```

The file `stackPanel.css` defines the basic structure of any given stackPanel:
```
.widget .title {}
.widget .content {}
/* Other stuff to make a StackPanel work */
```

The `outer()` method can continue to use the base `stackPanel.css` file, because the accessor methods defined in `StackPanelCss` are mapped into the default (no-prefix) namespace.  The inner StackPanel's style members are also available, but in the `inner` prefix.  Here's what `outer.css` might contain:
```
.widget {color: red;}

.inner-widget {
  color: blue;
  font-size: smaller;
}
```

## External and legacy scopes

In many cases, newly-developed CSS will need to be combined with external or legacy CSS. The `@external` at-rule can be used to suppress selector obfuscation while still allowing programmatic access to the selector name.

```
interface MyCssResource extends CssResource {
  String obfuscated();
  String legacySelectorA();
}

interface Resource extends ClientBundle {
  @Source("my.css")
  MyCssResource css();
}
```
```
@external .legacySelectorA, .legacySelectorB;
.obfuscated .legacySelectorA { .... }
.obfuscated .legacySelectorB { .... }

/* @external also accepts tail-globs */
@external .myProject-*;
.myProject-text {}
.myProject-foo {}
```

In the above example, the `.obfuscated` class selector will be obfuscated, and the `obfuscated()` method will return the replaced name. Neither of the legacy selectors will be obfuscated and the `legacySelectorA()` method will return the unobfuscated value.  Furthermore, because the `legacySelectorB` is explicitly defined in the `@external` declaration, the inaccessible class name will not trigger an error.

When it is necessary to prevent CssResource from allocating obfuscated class identifiers that use a particular prefix, the `CssResource.reservedClassPrefixes` module configuration variable can be extended.  Suppose that a module intends to integrate with a hypothetical "FooJs" library and that this library utilizes CSS class selectors that start with "fj".
```
<extend-configuration-property name="CssResource.reservedClassPrefixes" value="fj" />
```

Note that this configuration property and `@external` serve orthogonal purposes.  The `reservedClassPrefixes` variable means "do not export these names" while `@external` means "do not obfuscate these names".

## Automatically generating CssResource interfaces

A utility is included in the GWT distribution which will analyze a CssResource-compatible CSS file and create a corresponding Java interface for accessing the classnames used in the file.

```
java -cp gwt-dev.jar:gwt-user.jar com.google.gwt.resources.css.InterfaceGenerator \
  -standalone -typeName some.package.MyCssResource -css input.css
```

The generated interface will be emitted to `System.out`. The `-standalone` option will add the necessary `package` and `import` statements to the output so that it can be used as part of a build process.

# Debugging and testing

Runtime debugging information can be turned on for testing and QA builds by adding the following to your module definition:
```
<inherits name="com.google.gwt.resources.EnableCssResourceDebugging" />
```

This will enable the `CssResource.getDebugInfo()` method.  This returns a `DebugInfo` object which provides access to various metadata about the CssResource, including a deobfuscation map.  Additionally, this module will install a [CssResourceObserver](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/resources/client/impl/CssResourceObserver.java) that will manage a global CssResource obfuscation map to enable integration testing to use CssResource class names.

Given the following CssResource declaration:
```
package com.example;
interface MyCssResource extends CssResource {
  String someClass();
}
interface MyClientBundle extends ClientBundle {
  MyCssResource css();
}
```
a test of the `com.example.MyModule` GWT module can look up the value that would be returned by `someClass()` using the following snippet of JavaScript:
```
var obf = window.gwtCssResource['com.example.MyModule']['com.example.MyClientBundle.css.someClass'];
var elements = document.findElementsByClassName(obf);
// test elements
```

# Important open questions
  * Can we manipulate style rules across browsers?
    * Maybe make MyCssResource.mySpriteClass() return Rule object
    * The @eval rule implies that stylesheets must be removable or can be updated in-place.
    * Does doing so update actual rendering on all browsers?
  * Other meta-classes
    * @Opacity
    * @RoundedCorners
      * Can we generalize the meta-classes into an extensible Java provider API?