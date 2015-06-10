One question that often comes up about the GWT Widget class is why it is an abstract base class as opposed to an interface. After all, one should always prefer an interface to a class wherever possible, right? And indeed, there are many reasons to use interfaces, most notably that they're easier to test and mock, and that you can create new extensions of them more easily, because you can mix them into existing classes.

So why is it that almost all client-side UI libraries, including GWT, provide an abstract base class rather than an interface for widgets/components? For example:
  * GWT: `Widget`
  * Swing: `JComponent`
  * SWT: `Widget`
  * Android: `View`

One problem is that widgets tend to be "different", in that they have to enforce a lot of constraints required by their underlying implementations. SWT, for example, has to deal with the fact that win32 (and probably other systems) requires that a parent window be known when creating a child window. This is expressed in SWT through the constructor `Widget(Widget parent, int style)`. Such a constraint cannot be expressed using an interface.

GWT's Widget class doesn't have to enforce that particular constraint, but there are others. One example is the logic found in `setParent()`, `onAttach()` and `onDetach()`, which contain very important, but subtle, logic -- failure to implement this properly will result in extremely difficult-to-debug memory leaks.

## Workaround: Testing

Admittedly, this introduces some difficulties. Let's say you're writing unit tests for a custom widget, and you want to avoid using the Widget base class, so that you can easily run tests without requiring a display. One approach to this would be to create a View interface that is implemented by both the concrete widget implementation and a mock implementation. But how can you do this if you need it to be a Widget? An approach that has worked well for a number of teams looks like this:

```
  interface View<T> {
    Widget asWidget();
    void setData(T data);
  }

  class MyView<T> extends Label implements View<T> {
    public void setData(T data) {
      setText(data.toString());
    }

    public Widget asWidget() {
      return this;
    }
  }

  class MockView<T> implements View<T> {
    public Widget asWidget() {
      return null;
    }

    public void setData(T data) {
      // Whatever.
    }
  }
```

The `asWidget()` method allows you to avoid having `View<T>` extend `Widget` at all -- only users of the view who need to actually add it to a panel will even call `asWidget()`. And the actual call will be inlined away by the compiler, which is nice.

## Workaround: Union Types

It is sometimes convenient to be able to write methods that require a parameter that is both a widget, and also requires some interface to be implemented by it. For example, say I want to write a method that simply requires a `Widget` that implements `HasText`. There is no `WidgetThatHasText` interface (one can imagine the interface explosion that would stem from going down that path), and there couldn't be one because `Widget` is a class. One solution to this problem is to use something like the `View<T>` approach above. But you could also use Java's union types, like so:

```
  setWidgetThatHasText(new Label());
  setWidgetThatHasText(new HTML());

  <W extends Widget & HasText> void setWidgetThatHasText(W foo) {
    RootPanel.get().add(foo);
  }
```

It's a bit verbose, but perfectly functional, and the user of the method is not exposed to its verbosity.