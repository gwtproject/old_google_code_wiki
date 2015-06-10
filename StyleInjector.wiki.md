# Goal

  * Provide a cross-browser abstraction for injecting additional CSS at runtime.

# Details

StyleInjector is part of the `com.google.gwt.dom.user` package.

Define your ClientBundle and CssResource:
```
public interface Resources extends ClientBundle {
  public static final Resources INSTANCE =
      GWT.create(Resources.class);

  @ImageOptions(repeatStyle = RepeatStyle.Horizontal)
  @Source("myBackground.png")
  public ImageResource backgroundFunction();

  @Source("myCss.css")
  public CssResource css();
}
```

The CSS contents (see CssResource for more information):
```
@sprite .some .selector {
  gwt-image: 'backgroundFunction';
}
```

In your `onModuleLoad`:
```
// Preferred
Resources.INSTANCE.css().ensureInjected();

// Older style
StyleInjector.injectStylesheet(Resources.INSTANCE.css().getText());
```


You now have your stylesheet applied to the document while taking advantage of strongly-named or inlined resource URLs.  Because the standard I18N-style of resource naming is applied to the `ImageResource` instance, it is possible to provide localized CSS background images without the need for multiple, per-locale stylesheets.  It is now possible to have `myBackground_fr.png` and `myBackground_en.png` substituted based on the `locale` deferred binding property.

# Optimization

DOM manipulation, especially manipulating `style` elements, has a measurable cost.  In the interest of improving runtime performance, StyleInjector does not manipulate the DOM immediately.  Each call to `CssResource.ensureInjected()` or `StyleInjector.inject()` will append the contents to be injected into a buffer and use `Scheduler.scheduleFinally()` to perform the DOM manipulation immediately before control returns to the browser's event loop.  This optimization makes the inject-in-static-initializer pattern operate without excessive runtime cost.

If it is necessary to have StyleInjector mutate the DOM immediately, there are overloads of the `inject()` method which accept a boolean parameter indicating that the DOM should be updated immediately.

# Caveats

Certain browsers have an upper bound on the total number of stylesheets that can be injected.  In this case, StyleInjector will append to previously-created stylesheets when `injectStylesheet()` is called.  If the developer wishes to maintain relative ordering of CSS content, the `injectAtEnd()` and `injectAtStart()` methods should be used instead of `inject()`.