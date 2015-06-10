This document is currently under construction.

# Browser-specific css

```
.foo {
  background: green;
}

@if user.agent ie6 {
  /* Rendering fix */
  .foo {
    position: relative;
  }
} @elif user.agent safari {
  .foo {
    \-webkit-border-radius: 4px;
  }
} @else {
  .foo {
    font-size: x-large;
  }
}
```

# Obfuscated CSS class names

CssResource will use method names as CSS class names to obfuscate at runtime.

```
interface MyCss extends CssResource {
  String className();
}

interface MyResources extends ClientBundle {
  @Source("my.css")
  MyCss css();
}
```

All instances of a selector with `.className` will be replaced with an obfuscated symbol when the CSS is compiled.  To use the obfuscated name:

```
MyResources resources = GWT.create(MyResources.class);
Label l = new Label("Some text");
l.addStyleName(resources.css().className());
```

If you have class names in your css file that are not legal Java identifiers, you can use the `@ClassName` annotation on the accessor method:
```
interface MyCss extends CssResource {
  @ClassName("some-other-name")
  String someOtherName();
}
```

# Background images / Sprites

CssResource reuses the ImageResource bundling techniques and applies them to CSS background images.  This is generally known as "spriting" and a special `@sprite` rule is used in CssResource.

```
interface MyResources extends ClientBundle {
  @Source("image.png")
  ImageResource image();

  @Source("my.css");
  CssResource css();
}
```

In `my.css`, sprites are defined using the `@sprite` keyword, followed by an arbitrary CSS selector, and the rule block must include a `gwt-image` property.  The `gwt-image` property should name the ImageResource accessor function.
```
@sprite .myImage {
  gwt-image: 'image';
}
```

The elements that match the given selection will display the named image and have their heights and widths automatically set to that of the image.

## Tiled images

If the ImageResource is decorated with an `@ImageOptions` annotation, the source image can be tiled along the X- or Y-axis.  This allows you to use 1-pixel wide (or tall) images to define borders, while still taking advantage of the image bundling optimizations afforded by ImageResource.

```
interface MyResources extends ClientBundle {
  @ImageOptions(repeatStyle = RepeatStyle.Horizontal)
  @Source("image.png")
  ImageResource image();
}
```

The elements that match the `@sprite`'s selector will only have their height or width set, based on the direction in which the image is to be repeated.

## 9-boxes

In order to make the content area of a 9-box have the correct size, the height and widths of the border images must be taken into account.  Instead of hard-coding the image widths into your CSS file, you can use the `value()` CSS function to insert the height or width from the associated ImageResource.

TODO: Move this setup into DecoratorPanel or other Widget that just accepts the following Resources interface.

```
  public interface Resources extends ClientBundle {
    Resources INSTANCE = GWT.create(Resources.class);

    @Source("bt.png")
    @ImageOptions(repeatStyle = RepeatStyle.Horizontal)
    ImageResource bottomBorder();

    @Source("btl.png")
    ImageResource bottomLeftBorder();

    @Source("btr.png")
    ImageResource bottomRightBorder();

    @Source("StyleInjectorDemo.css")
    CssResource css();

    @Source("lr.png")
    @ImageOptions(repeatStyle = RepeatStyle.Vertical)
    ImageResource leftBorder();

    @Source("rl.png")
    @ImageOptions(repeatStyle = RepeatStyle.Vertical)
    ImageResource rightBorder();

    @Source("tb.png")
    @ImageOptions(repeatStyle = RepeatStyle.Horizontal)
    ImageResource topBorder();

    @Source("tbl.png")
    ImageResource topLeftBorder();

    @Source("tbr.png")
    ImageResource topRightBorder();
  }
```

```
.contentArea {
  padding: value('topBorder.getHeight', 'px') value('rightBorder.getWidth', 'px')
      value('bottomBorder.getHeight', 'px') value('leftBorder.getWidth', 'px');
}

@sprite .contentAreaTopLeftBorder {
  gwt-image: 'topLeftBorder';
  position: absolute;
  top:0;
  left: 0;
}

@sprite .contentAreaTopBorder {
  gwt-image: 'topBorder';
  position: absolute;
  top: 0;
  left: value('topLeftBorder.getWidth', 'px');
  right: value('topRightBorder.getWidth', 'px');
}

@sprite .contentAreaTopRightBorder {
  gwt-image: 'topRightBorder';
  position: absolute;
  top:0;
  right: 0;
}

@sprite .contentAreaBottomLeftBorder {
  gwt-image: 'bottomLeftBorder';
  position: absolute;
  bottom: 0;
  left: 0;
}

@sprite .contentAreaBottomBorder {
  gwt-image: 'bottomBorder';
  position: absolute;
  bottom: 0;
  left: value('bottomLeftBorder.getWidth', 'px');
  right: value('bottomRightBorder.getWidth', 'px');
}

@sprite .contentAreaBottomRightBorder {
  gwt-image: 'bottomRightBorder';
  position: absolute;
  bottom: 0;
  right: 0;
}

@sprite .contentAreaLeftBorder {
  gwt-image: 'leftBorder';
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
}

@sprite .contentAreaRightBorder {
  gwt-image: 'rightBorder';
  position: absolute;
  top: 0;
  right: 0;
  height: 100%;
}
```
```
<div class="contentArea">
<div class="contentAreaTopLeftBorder"></div>
<div class="contentAreaTopBorder"></div>
<div class="contentAreaTopRightBorder"></div>
<div class="contentAreaBottomLeftBorder"></div>
<div class="contentAreaBottomBorder"></div>
<div class="contentAreaBottomRightBorder"></div>
<div class="contentAreaLeftBorder"></div>
<div class="contentAreaRightBorder"></div>
</div>
```