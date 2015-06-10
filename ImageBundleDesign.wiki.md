# Automatic Compile-Time Image Bundling
Bruce Johnson, Rajeev Dayal

# Introduction

Making AJAX apps very fast requires minimizing the number of HTTP round-trips. Images, particularly icon-style images, are a major cause of multiple small requests. What's worse, such image requests typically happen at startup, which creates a sluggish startup experience  for the end user, working against [our first design axiom](http://code.google.com/webtoolkit/makinggwtbetter.html#designaxioms). That's no good, but fortunately GWT has the machinery to create a highly-optimized solution to this recurring problem.

# Motivation
Why optimize treatment of images?

## Use Case: Toolbar
Toolbars containing image (or "icon") buttons are commonly needed, and they often have  these characteristics:
  * Icons are of the same image type (e.g. 'gif').
  * Icons are of similar or identical size.
  * The number of icons tends to grow as the application grows in complexity.
  * Icon content changes relatively rarely compared to other aspects of the application.
  * When icons do change, the associated application code usually changes as well.
  * Application programmers prefer not to have to hard-code icon dimensions into their application code because doing so would create a vital but implicit dependency between the source code and external image resources.

At startup, each toolbar icon typically causes an HTTP freshness check (i.e. "If-Modified-Since"), which has several disadvantages:
  * HTTP requests that result in a 304 ("Not Modified") are ultimately a waste of time and would ideally be avoided altogether.
  * The browser lays out the UI using a default image placeholder image, which typically isn't the same size as the real icon, making the UI repeatedly re-layout, which is slow and unattractive. This will be referred to as "bouncy UI startup".
  * HTTP 1.1 requires browser to limit outgoing HTTP requests to two per domain/port. Unnecessary freshness checks, in addition to slowing down the display of the icons themselves, actually block other HTTP that are truly necessary to do real work (e.g. RPCs).
  * While possible in theory to avoid round-trips by setting HTTP caching/expiration response headers for each icon, these response headers are often not used in practice because it is difficult to decide upon an appropriate expiration policy (e.g. "Is 2 hours long enough? 2 days? 2 weeks?").

## Goals
  * Allow developers to easily create bundles of static images that can be downloaded in a single HTTP request.
  * Solve the "bouncy UI startup" problem in an automated way that does not require developers to specify image sizes explicitly in code.
  * Do not require any external tools to be executed. This should not be a separate command-line tool that complicates the build process.
  * Avoid even simple HTTP freshness checks, leaving HTTP connections available for important work.
  * Never issue more than one request for the same image URL. This is of particular relevance when the image is not already cached on IE.
  * Ensure the new functionality is compatible with the existing Image class.

## Non-Goals
  * Find the optimal way to combine individual images into a single image in multiple dimensions. Simply lining up images left-to-right to create an image strip is sufficient.
  * Support use cases that need a dynamic set of images, such as those in which end user input controls the set of images. For example, this mechanism is not intended as a way to implement a photo gallery.

# Solution
With a few tweaks to the existing `Image` class, the addition of a deferred binding generator and a ImageBundle tag interface, we can achieve all of the goals above in a way that actually simplifies life for GWT developers.

## ImageBundleGenerator and ImageBundle
The two key types are `ImageBundleGenerator` and `ImageBundle`.

### Additional Documentation
The GWT 1.6 javadoc contains additional `ImageBundle` documentation:

http://google-web-toolkit.googlecode.com/svn/javadoc/1.6/com/google/gwt/user/client/ui/ImageBundle.html

### Defining an Image Bundle
This is a similar design to the [Constants](http://code.google.com/webtoolkit/documentation/com.google.gwt.i18n.client.Constants.html) interface. `ImageBundle` is an empty tag interface that can be extended to create custom image bundles.

Derived interfaces contain zero or more methods, each of which must have
  * A return type of `AbstractImagePrototype`
  * No parameters
  * An optional annotation that specifies `@Resource` that names an image file in the module's classpath <sup>(should this be the public path instead?)</sup>. If `@Resource` is not specified, then a file whose base filename matches the method name itself is sought (the file extension can be gif, png, or jpg, but if multiple such files are present, which file is chosen is unspecified in the current implementation).

For example, to create an image bundle for a word processor toolbar, you could define an image bundle as follows:
```
interface WordProcessorImageBundle extends ImageBundle {

  /**
   * Would match either file 'newFileIcon.gif' or 'newFileIcon.png' in the same package as this type. Note that other file extensions may also be recognized.
   */
  AbstractImagePrototype newFileIcon();

  /**
   * Would bundle the file 'open-file-icon.gif' residing in the same package as this type.
   */
  @Resource("open-file-icon.gif")
  AbstractImagePrototype openFileIcon();

  /**
   * Would bundle the file 'savefile.gif' residing in the package 'org.example.icons'.
   */
  @Resource("org/example/icons/savefile.gif")
  AbstractImagePrototype saveFileIcon();
}
```

### Using an Image Bundle
Create the image bundle object using `GWT.create()` as would for any other object subject to deferred binding. For example,
```
  WordProcessImageBundle wpib = 
    (WordProcessImageBundle)GWT.create(WordProcessImageBundle.class);
  Toolbar tb = new Toolbar();
  tb.addImageButton(wpib.newFileIcon());  
  tb.addImageButton(wpib.openFileIcon());  
  tb.addImageButton(wpib.saveFileIcon());  
```

### Configuring Your Module for Image Bundles
To use image bundling, your module needs to explicitly inherit `ImageBundle`.

### Behavior/Restrictions/Caveats
For the `ImageBundle`-compatible type `T` specified in `GWT.create(T.class)`,
the following must be true:
  * `T` must be an interface, not a class.
  * All of the methods on `T` must conform to the required signature (see above)
  * An optional `@Resource` annotation may be used to override the default image filename.

The generated image bundle output file will
  * have same type "png"; files referenced via `@Resource` will be converted
  * be named `md5.cache.ext`, where `md5` is a hash of the bytes in all the consistuent images and `ext` is the image file type
  * be written into the output directory for the module (that is, the same location into which compiled JS is written

## Clipping Constructor for Image
To make the new image bundling functionality work well with the existing `Image` class, a new "clipped" mode is introduced, allowing an image to refer to a sub-image within a larger image. There are a variety of formulations of this idea, but the one that should have the least impact is to
  * add a constructor overload that allows the (left, top, width, height) of the clipping region to be set once when the object is created
  * not have a method for changing the clipping region dynamically (the API wouldn't prevent this in the future, but we wouldn't add it for now)
  * use the CSS "background-image" and "background-position" attribute to effect a "viewport" within the larger image without requiring the introduction of additional DOM elements.

A clipped image might appear like this in the DOM:
```
  <img src="clear.cache.gif" style="background-image:md5.cache.ext; background-position:-Xpx -Ypx; width:Wpx; height:Hpx">
```

The other more obvious implementation technique requires you to encase the <img> in a <div>, then set "overflow:hidden", width and height on the <div>. However, this adds weight and makes the DOM structure for <code>Image</code> less predictable to client code. Some browsers (not naming any names, but...Internet Explorer) may require extra weirdness to support transparency.<br>
<br>
<h3>Caveats</h3>
<ul><li>This technique requires the introduction of a well-known 1x1 transparent image called <code>clear.cache.gif</code>. On the plus side, it would be permanently cacheable and could be useful in other circumstances.<br>
</li><li>Setting CSS "padding" on the image element would reveal clipped portions of the underlying image. This could be worked around by wrapping the image in another <div> (or the appropriate <code>Panel</code>) explicitly in user code. It seems like a rare enough use case that accepting this flaw is a worthwhile compromise.</li></ul>

<h2>Image Request Batching</h2>
TBD -- IE6 has a bug when the same image URL is requested multiple times from code that causes it to issue all the requests instead of realizing a single result image can be shared. We could solve this through code in Image or in the DOM.<br>
<br>
<h1>Implications</h1>

<h2>Design Goals Achieved</h2>
How does this approach address the goals?<br>
<ul><li>Easy? Yes. The only extra overhead is defining a new interface.<br>
</li><li>Solves "bouncy UI startup"? Yes. The generated subclass specifies sizes for all images explicitly in code, yet does not require the programmer to ever know/see/care.<br>
</li><li>No external tools? Yes. Uses standard deferred binding, which is integrated into GWT compilation.<br>
</li><li>Avoid even simple HTTP freshness checks? Yes. The name of the compound image file is based on the MD5 of its contents, making it safely infinitely cacheable. At worst, the compound image file name changes when the app itself is recompiled, so as long as you are running a cached compilation, then the requested compound image file wouldn't have changed (because its name is embedded in the compilation) and the infinitely-cacheable version will be used, which does not require an HTTP round trip.<br>
</li><li>No more than one request for the same image URL? We'd have to implement image request batching to solve this.<br>
</li><li>Compatible with existing Image? Almost totally. The only downside is CSS "padding", which is easily worked around and rare enough to be considered worth breaking perfect compatibility.</li></ul>

<h2>Interaction With Localization</h2>
Image bundles are (in the current implementation) orthogonal to localization and do not use localization concepts directly. It is possible to localize image bundles using a locale-specific factory.<br>
<br>
Suppose that we have the following <code>ImageBundle</code>:<br>
<pre><code>public interface MyImageBundle extends ImageBundle {<br>
   /**  <br>
    * The default icon if no locale-specific image is specified.<br>
    */<br>
   @Resource("help_icon.gif")<br>
   AbstractImagePrototype helpIcon();<br>
<br>
<br>
   /**  <br>
    * The default icon if no locale-specific image is specified.<br>
    */<br>
   @Resource("compose_new_message_icon.gif")<br>
   AbstractImagePrototype composeNewMessageIcon();<br>
}<br>
</code></pre>

We can define French and English variations of each <code>ImageBundle</code> image by extending <code>MyImageBundle</code> for each locale:<br>
<pre><code>public interface MyImageBundle_en extends MyImageBundle {<br>
   // Note that we are not re-declaring helpIcon(), so this bundle<br>
   // uses the inherited metadata.<br>
<br>
   /**<br>
    * The English version of this icon.<br>
    */<br>
   @Resource("compose_new_message_icon_en.gif")<br>
   AbstractImagePrototype composeNewMessageIcon();<br>
}<br>
<br>
public interface MyImageBundle_fr extends MyImageBundle {<br>
   /**<br>
    * The French version of this icon.<br>
    */<br>
   @Resource("help_icon_fr.gif")<br>
   AbstractImagePrototype helpIcon();<br>
<br>
   /**<br>
    * The French version of this icon.<br>
    */<br>
   @Resource("compose_new_message_icon_fr.gif")<br>
   AbstractImagePrototype composeNewMessageIcon();<br>
}<br>
</code></pre>

By extending <a href='http://code.google.com/webtoolkit/documentation/com.google.gwt.i18n.client.Localizable.html'>Localizable</a>, we can create a locale-sensitive factory to select the appropriate <code>MyImageBundle</code>-derived interface based on the user's locale.<br>
<br>
<pre><code>public interface MyImageBundleFactory extends Localizable {<br>
    MyImageBundle createImageBundle();<br>
}<br>
<br>
public class MyImageBundleFactory_en implements MyImageBundleFactory {<br>
    MyImageBundle createImageBundle() {<br>
        return (MyImageBundle) GWT.create(MyImageBundle_en.class);<br>
    }<br>
}<br>
<br>
public class MyImageBundleFactory_fr implements MyImageBundleFactory {<br>
    MyImageBundle createImageBundle() {<br>
        return (MyImageBundle) GWT.create(MyImageBundle_fr.class);<br>
    }<br>
}<br>
</code></pre>

The application code looks something like the following:<br>
<pre><code>// Create a locale-specific MyImageBundleFactory.<br>
MyImageBundleFactory myImageBundleFactory = (MyImageBundleFactory) GWT.create(MyImageBundleFactory.class);<br>
<br>
// This will return a locale-specific MyImageBundle, since we are using a locale-specific<br>
// factory to create it.<br>
MyImageBundle myImageBundle = myImageBundleFactory.createImageBundle();<br>
<br>
// Get the image prototype for the icon we are interested in.<br>
AbstractImagePrototype helpIconProto = myImageBundle.helpIcon();<br>
<br>
// Create an Image object from the prototype.<br>
somePanel.add(helpIcon.createImage());<br>
</code></pre>