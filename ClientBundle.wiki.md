**Documentation for the [GWT 2.0 release of ClientBundle](http://code.google.com/webtoolkit/doc/latest/DevGuideClientBundle.html) can be found on the GWT Developer's Guide website.**



# Introduction

> The resources in a deployed GWT application can be roughly categorized into resources to never cache `.nocache.js`, to cache forever `.cache.html`, and everything else `myapp.css`.  The ClientBundle interface moves entries in the everything-else category into the cache-forever category.

# Goals

  * No more uncertainty if your application is getting the right contents for program resources.
  * Decrease non-determinism caused by intermediate proxy servers.
  * Enable more aggressive caching headers for program resources.
  * Eliminate mismatches between physical filenames and constants in Java code by performing consistency checks during the compile.
  * Use 'data:' URLs, JSON bundles, or other means of embedding resources in compiled JS when browser- and size-appropriate to decrease the number of round-trips entirely.
  * Provide an extensible design for adding new resource types.
  * Ensure there is no penalty for having multiple ClientBundle resource functions refer to the same content.

# Non-Goals

  * To provide a file-system abstraction

# Examples

To use the ClientBundle, add an inherits tag to your `gwt.xml` file:
```
<inherits name="com.google.gwt.resources.Resources" />
```

Interfaces:
```
public interface MyResources extends ClientBundle {
  public static final MyResources INSTANCE =  GWT.create(MyResources.class);

  @Source("my.css")
  public CssResource css();

  @Source("config.xml")
  public TextResource initialConfiguration();

  @Source("manual.pdf")
  public DataResource ownersManual();
}
```

You can then say:
```
  Window.alert(MyResources.INSTANCE.css().getText());
  new Frame(MyResources.INSTANCE.ownersManual().getURL());
```

# I18N

ClientBundle is compatible with GWT's I18N module.

Suppose you defined a resource:
```
@Source("default.txt")
public TextResource defaultText();
```

For each possible value of the `locale` deferred-binding property, the ClientBundle generator will look for variations of the specified filename in a manner similar to that of Java's `ResourceBundle`.

Suppose the `locale` were set to `fr_FR`.  The generator would look for files in the following order:
  1. default\_fr\_FR.txt
  1. default\_fr.txt
  1. default.txt

This will work equally well with all resource types, which can allow you to provide localized versions of other resources, say `ownersManual_en.pdf` versus `ownersManual_fr.pdf`.

# Pluggable Resource Generation

Each subtype of `ResourcePrototype` must define a `@ResourceGenerator` annotation whose value is a concrete Java class that extends `ResourceGenerator`.  The instance of the `ResourceGenerator` is responsible for accumulation (or bundling) of incoming resource data as well as a small degree of code generation to assemble the concrete implementation of the ClientBundle class.  Implementors of `ResourceGenerator` subclasses can expect that only one `ResourceGenerator` will be created for a given type of resource within an ClientBundle interface.

The methods on a `ResourceGenerator` are called in the following order
  1. `init` to provide the `ResourceGenerator` with a `ResourceContext`
  1. `prepare` is called for each `JMethod` the `ResourceGenerator` is expected to handle
  1. `createFields` allows the `ResourceGenerator` to add code at the class level
  1. `createAssignment` is called for each `JMethod`.  The generated code should be suitable for use as the right-hand side of an assignment expression.
  1. `finish` is called after all assignments should have been written.

`ResourceGenerators` are expected to make use of the `ResourceGeneratorUtil` class.

# Potential pitfalls
  * Changing the content of the resources will change the filenames (or data: encoding), thus forcing a recompile of the GWT application. To avoid this, the inlining and renaming features can be globally toggled off in your gwt.xml file during the development phase.
  * Inlining files into the compiled JS may not make sense if those files are not always accessed by the program, thus inlining should be configurable on a per-resource or per-ClientBundle basis.

# Levers and knobs

  * `ClientBundle.enableInlining` is a deferred-binding property that can be used to disable the use of `data:` URLs in browsers that would otherwise support inlining resource data into the compiled JS.
  * `ClientBundle.enableRenaming` is a configuration property that will disable the use of strongly-named cache files.

# Resource types

  * CssResource
  * DataResource
  * ExternalTextResource
  * GwtCreateResource
  * ImageResource
  * TextResource

# Migrating from ImmutableResourceBundle

## Plan

  * Expected that incubator IRB system will initially co-exist with ClientBundle in trunk while integration issues are worked out.
  * Will then change incubator code to add deprecations.
  * No plans to remove types from incubator for _long period of time_.
  * No additional features or bug-fixes will be added to incubator APIs.

## Changes
  * Package changed to `com.google.gwt.resources`
  * Inherit `com.google.gwt.resources.Resources`
  * Renamed `ImmutableResourceBundle` to `ClientBundle`
  * Renamed `@Resource` annotation to `@Source`
  * Map-style methods moved to `ClientBundleWithLookup`
  * Switched most gwt.xml properties to be configuration properties instead of deferred-binding properties
  * Some `gwt.xml` properties have changed names:
    * `ResourceBundle.*` to `ClientBundle.*`
    * `CssResource.enableMerge` to `CssResource.mergeEnabled`
    * `CssResource.forceStrict` to `CssResource.strictAccessors`
    * `CssResource.globalPrefix` to `CssResource.obfuscationPrefix`
    * `CssResource.style` has changed to a configuration property; switch from `<set-property>` to `<set-configuration-property>` tags to configure this.
  * StyleInjector moved to `com.google.gwt.dom.client` package