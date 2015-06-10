

# Goal
  * Provide access to image data at runtime in the most efficient way possible
  * Do not bind the ImageResource API to a particular widget framework

# Non-goals
  * To provide a general-purpose image manipulation framework.

# Overview

  1. Define a ClientBundle with one or more ImageResource accessors:
```
interface Resources extends ClientBundle {
  @Source("logo.png")
  ImageResource logo();

  @Source("arrow.png")
  @ImageOptions(flipRtl = true)
  ImageResource pointer();
}
```
  1. Instantiate the ClientBundle via a call to `GWT.create()`.
  1. Instantiate one or more Image widget or use with the [CssResource @sprite](CssResource#Image_Sprites.md) directive.

## `ImageOptions`
The `@ImageOptions` annotation can be applied to the ImageResource accessor method in order to tune the compile-time processing of the image data
  * `flipRtl` is a boolean value that will cause the image to be mirrored about its Y-axis when `LocaleInfo.isRTL()` returns `true`.
  * `repeatStyle` is an enumerated value that is used in combination with the`@sprite` directive to indicate that the image is intended to be tiled.
  * One or both of the `height` and `width` properties can be set to have the source image scaled at compile-time.  If only one of the scaling properties are set, the image will be scaled so as to maintain the original aspect ratio.

## Supported formats
`ImageBundleBuilder` uses the Java [ImageIO](http://java.sun.com/javase/6/docs/api/javax/imageio/package-summary.html) framework to read image data. This includes support for all common web image formats.

Animated GIF files are supported by ImageResource.  While the image data will not be incorporated into an image strip, the resource is still served in a browser-optimal fashion by the larger ClientBundle framework.

# Converting from ImageBundle

Only minimal changes are required to convert existing code to use ImageResource.
  * `ImageBundle` becomes `ClientBundle`
  * `AbstractImageProtoype` becomes `ImageResource`
  * `AbstractImagePrototype.createImage()` becomes `new Image(imageResource)`
  * Other methods on `AbstractImagePrototype` can continue to be used by calling `AbstractImagePrototype.create(imageResource)` to provide an API wrapper.