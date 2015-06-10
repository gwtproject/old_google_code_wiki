# This document is out-of-date and must be updated with current implementation details

# GWT Linking Subsystem

## The status quo
As of GWT 1.4, invoking the compile normally like this

> `/home/bruce/gwt$ GWTCompiler -out www com.google.gwt.sample.hello.Hello`

creates the following output:

  * Legacy resources
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/gwt.js`
  * Always-there resources
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/clear.cache.gif`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/history.html`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/hosted.html`
  * Standard bootstrap + scripts
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/com.google.gwt.sample.hello.Hello.nocache.js`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/6B30A99615F91CED74DAB7A87B6F70A1.cache.html`
    * _...more MD5.cache.html files..._
  * Cross-site bootstrap + scripts
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/com.google.gwt.sample.hello.Hello-xs.nocache.js`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/6B30A99615F91CED74DAB7A87B6F70A1.cache.js`
    * _...more MD5.cache.js files..._
  * Compiler intermediate files that shouldn't be deployed
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/6B30A99615F91CED74DAB7A87B6F70A1.cache.xml`
    * _...more MD5.cache.xml files..._

## Problems with the status quo
  1. Having both the standard and cross-site output in every build doubles the number of files that users have to fiddle with, even though they are inevitably using only one or the other. As deferred binding becomes more popular, this doubling will only increase the risk that people will be turned off by the sheer number of files -- regardless of how good the reason is.
  1. Users mistakenly deploy files that aren't really meant for deployment, such as `MD5.cache.xml` files.
  1. Upcoming file types (e.g. compiler map files) will further confuse which files should be deployed vs. not.
  1. The current scheme is not extensible to other forms of packaging and distribution by an external mechanism.

## Use cases we definitely want to enable
  1. ~~Create a legacy module (i.e. gwt.js based) deployment from your module.~~
  1. Create a standard (i.e. IFRAME-based) deployment from your module.
  1. Create a module that can be included cross-site.
  1. Create a module that can be deployed as a Gadget.
  1. Create a Gears-compatible version of your module.
  1. Create a module that appears to be a "regular old" JavaScript library (i.e. no deferred binding, no two-step startup).

## Potential, though unplanned, use cases
  1. Create a .war file from your module (tricky b/c we don't know how to package bytecode depependencies).
  1. Create a .swf (that is, ActionScript + resources) from your module.
  1. Create an Adobe AIR package from your module.
  1. Create Java/bytecode for the server-side RPC stub as part of any of the above use cases.

## Proposed plan
We should
  * divide the build process into two distinct steps: (1) compilation and (2) linking
  * make the linking step pluggable so that it is possible to incorporate externally-provided linkers
  * have a smooth way to integrate external linkers into the familiar compilation process
  * create three built-in linkers: (1) normal (2) cross-site and (3) single-script
  * relocate compiler output such that compiler intermediate files will be less prominent and will no longer be confused with deployable files; in other words, the most prominent results of the build process will be the linker output, not the compiler's working files

## How it should look when you use it
When building with GWT 1.5, the default command line will look exactly the same as before

> `/home/bruce/gwt$ GWTCompiler -out www com.google.gwt.sample.hello.Hello`

And it works essentially the same as it did in GWT 1.4, except only the default "standard deployment" linker is used, and results go in subdirectory `std/`

  * Always-there resources
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/clear.cache.gif`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/history.html`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/hosted.html`

  * Standard bootstrap + scripts
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/com.google.gwt.sample.hello.Hello.nocache.js`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/6B30A99615F91CED74DAB7A87B6F70A1.cache.html`
    * `_...more MD5.cache.html files..._`

The difference is that the compiler's intermediate files go in the special (less prominent) subdirectory `.gwt-build/`

  * Compiler intermediate files that shouldn't be deployed
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/.gwt-build/6B30A99615F91CED74DAB7A87B6F70A1.cache.xml`
    * _...more MD5.cache.xml files..._

It is possible to invoke any number of linkers in a single compile. For example, suppose you want the "standard" and "cross-site" style of output. Your module XML might look like this:
```
  ...earlier stuff in the module...

  <set-linker name="std,xs"/>

  ...later stuff in the module...
```

Running the compiler against the module above would create two sets of output:

  * Standard bootstrap + scripts
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/com.google.gwt.sample.hello.Hello.nocache.js`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/std/6B30A99615F91CED74DAB7A87B6F70A1.cache.js`
    * _...more MD5.cache.js files and other boilerplate..._

  * Cross-site bootstrap + scripts
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/xs/com.google.gwt.sample.hello.Hello-xs.nocache.js`
    * `/home/bruce/www/com.google.gwt.sample.hello.Hello/xs/6B30A99615F91CED74DAB7A87B6F70A1.cache.js`
    * _...more MD5.cache.js files and other boilerplate..._

## Configuring linkers
A linker is a class that
  * implements the `Linker` interface,
  * is specified in module XML using the `<define-linker name="_abbrev_" class="_linker_">` construct, and
  * is found on the classpath when the compiler runs.

The most common way to invoke a linker is ~~via the compiler using the `-linker <abbrev>` switch~~ by using the `<set-linker name="_abbrev_">` construct in module XML. The output of a linker is always placed in a subdirectory that matches the configured linker abbreviation.

Linker abbreviations are mapped to Java classes via GWT modules. As with most module settings, settings prioritize more "derived" modules, so that, for example, settings in leaf modules have the highest priority.

Each linker entry follows the simple pattern in the following example:

> `<define-linker name="std" class="com.google.gwt.dev.StandardLinker"/>`
> `<define-linker name="xs" class="com.google.gwt.dev.CrossSiteLinker"/>`

The natural cascading behavior of modules allows existing linker abbreviations to be overridden (which will hopefully be rare in practice) but more importantly, makes it possible for libraries to contribute linkers. The "name" attribute for linkers must follow the same naming rules as deferred binding client properties (basically, they have to look like valid keys in Java properties files).

A module can specify a default linker that will be used if no target is specified on the command line. For example, you might specify "gadget" as the default linker in the Gadget.gwt.xml module.

> `<set-linker name="gadget"/>`

~~If a linker is explicitly specified on the command line, then the default linker setting is ignored.~~ It is possible to specify more than one linker name in a `<set-linker>` element by separating multiple names with commas, meaning that each linker should be run when compiling. A subsequent `<set-linker>` element in a derived module's XML replaces the previous linker selection.

## Linkers are single-purpose
To keep things as simple as possible, linkers should not require arguments, and no mechanism will be included to allow arguments in the first version. In most cases where variability of behavior is required, hopefully it will make sense to use a different-but-equally-simple no-arg linker rather than creating a complex linker with lots of switches. Very specific, single-purpose linkers are preferred because:
  * they can be written more simply,
  * there is less to document,
  * there is less for users to learn in order to "try out" a new linker type, and perhaps most importantly,
  * error messages can always be written very specifically and usefully, since the user's purpose is well-known.

## Running library-contributed linkers
By design, there is no distinction between "built-in" linkers and library-contributed linkers. Consider the use case of GALGWT contributing a "gadget" linker. Within the well-known module spec `Gadget.gwt.xml`, we'd see

```
<define-linker name="gadget" class="com.google.gwt.gadget.dev.GadgetLinker"/>
<add-linker name="gadget" />
```

and in the module spec `WeatherGadget.gwt.xml`, we'd see

```
<inherits name="com.google.gwt.gadget.Gadget"/>
```

Consequently, the gadget output is created when the module is compiled:

> `java -cp gwt-dev.jar:gwt-user.jar:gwt-gadgets.jar GWTCompiler com.google.gwt.sample.weather.WeatherGadget`

The linker writes the deployable files in a `gadget/` subdirectory (note that there is no selection script):

  * The Gadget manifest for an "html" Gadget, with the selection script inlined into the XML
    * `/home/bruce/www/com.google.gwt.sample.weather.WeatherGadget/gadget/WeatherGadget.xml`
  * Deferred binding permutations
    * `/home/bruce/www/com.google.gwt.sample.weather.WeatherGadget/gadget/6B30A99615F91CED74DAB7A87B6F70A1.cache.js`
    * _...more MD5.cache.js files..._

Invoking linkers independently from the compiler is not a goal for the initial version, although we should strive to design so as not to prevent it in the future.

## Altering the observed behavior of LinkerContext

In some deployment cases, the desired compiler output can be achieved simply by modifying the behavior of the LinkerContext that is presented to the Linkers.  To support mix-in behavior, the compiler provides LinkerContextShim, which is a base class that implementors can use to inject behaviors into the linking process.  A shim is registered in the `gwt.xml` file with the `<extend-linker-context>` tag.
```
<extend-linker-context class="com.google.gwt.gears.offline.linker.GearsManifestLinkerShim" />
```

Shims will be chained in the order in which the first instance of the class appears when evaluating a module's `gwt.xml` file.  Shims should have behaviors that are orthogonal to the implementation details of Linkers.