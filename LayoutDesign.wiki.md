## Introduction and Motivation

GWT's current mechanisms for handling application-level layout have a number of
significant problems:

  * They're unpredictable.
  * They often require extra code to fix up their deficiencies:
    * For example, causing an application to fill the browser's client area with interior scrolling is nearly impossible without extra code.
  * They don't work very well in standards mode.

Their underlying motivation was sound -- the intention was to let the browser's
native layout engine do almost all of the work. But the above deficiencies are
crippling.

## Goals
  * Perfectly predictable layout behavior. It should be possible, nay encouraged, to write pixel-precise layout tests.
    * It should also work in the presence of CSS decorations (border, margin, and padding) with any units.
  * Work in standards-mode (this should eliminate any and all reasons to run in quirks mode).
  * Get the browser to do almost all of the work in its layout engine.
    * Manual adjustments should occur only when necessary and never create feedback loops with the layout engine.
      * Work with arbitrary units, and handle user-driven font-size changes.
  * Smooth, automatic animation.

## Non-Goals
  * Work in quirks-mode.
  * Swing-style layout based upon "preferred size". This is effectively intractable in the browser.
  * Take over all layout. This is intended to handle coarse-grained "desktop-like" layout. The individual bits and pieces, such as form elements, buttons, and text should still be laid out naturally.

## Details
Given that the most important goals are predictability and performance, any
reasonable solution must leverage the browser's built-in layout engine as much
as possible. This leaves few options, and the one I've settled on is to use
absolute positioning, but to take advantage of the very simple constraint
system implied by the existence of the CSS properties 'left', 'top', 'width',
'height', 'right', and 'bottom'. Most everyone's familiar with the first four,
but when you add 'right' and 'bottom', things get interesting. Take the
following CSS example:

```
.parent {
  position: relative; /* to establish positioning context */
}

.child {
  position: absolute; left:1em; top:1em; right:1em; bottom:1em;
}
```

In this example, the child will automatically consume the parent's entire
space, minus 1em of space around the edge. Any two of these properties (on each
axis) forms a valid constraint pair (three would be degenerate), producing lots
of interesting possibilities. This is especially true when you consider various
mixtures of relative units, such as "em" and "%".

The following are some examples of layouts and the sets of constraints that
produce them:

_Fixed size, pinned to corners, sized in ems:_
```
+--------------------------------------+
|                                      |
|  +--------+                          |
|  |l/t:1em |                          |
|  |w/h:10em|                          |
|  |        |                          |
|  +--------+                          |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                         +--------+   |
|                         |r/b:1em |   |
|                         |w/h:10em|   |
|                         |        |   |
|                         +--------+   |
|                                      |
+--------------------------------------+
```

_Dock layout, mixing % and em units:_
```
+--------------------------------------+
| l/r:0                                |
| t:0                                  |
| h:20%                                |
+-------------------------------+------+
| l:0                           |r:0   |
| t:20%                         |w:10em|
| r:10em                        |t:20% |
| b:0                           |b:0   |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
|                               |      |
+-------------------------------+------+
```

_Full width popup, 25% height:_
```
+--------------------------------------+
|                                      |
|    +----------------------------+    |
|    |l/r/t:1em                   |    |
|    |h:25%                       |    |
|    |                            |    |
|    |                            |    |
|    +----------------------------+    |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
+--------------------------------------+
```

_Stack (all l/r:0):_
```
+--------------------------------------+
|t:0em                                 |
|h:2em                                 |
+--------------------------------------+
|t:2em                                 |
|h:2em                                 |
+--------------------------------------+
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
|                                      |
+--------------------------------------+
|b:0em                                 |
|h:2em                                 |
+--------------------------------------+
```

## API Design
There is a single "Layout" class, which is not tied to any particular widget
(or for that matter, to the idea of widgets at all). It allows its users to
designate an element as a layout "parent", and manages its children. A "Layer"
is associated with each child, which is the interface to specify its
constraints. This structure is designed to make it easy for higher-level
widgets to work with.

```
public class Layout {
  // Callback used with animate() to perform custom animations.
  public interface AnimationCallback {
    void onAnimationComplete();
    void onLayout(Layer layer, double progress);
  }

  // The interface used to manipulate child elements.
  public class Layer {
    public void setLeftWidth(
      double left, Unit leftUnit, double right, Unit rightUnit);
    // setLeftRight()
    // setRightWidth()
    // setTopHeight()
    // setTopBottom()
    // setBottomHeight()
  }

  // Create a new Layout on the given parent element.
  public Layout(Element parent);

  // Attach and remove child elements.
  // A Layer is associated with each child.
  public Layer attachChild(Element child);
  public void removeChild(Layer layer);

  // Layout immediately.
  public void layout();

  // Animate from the present state to the next.
  public void animate(int duration);
  public void animate(int duration, final AnimationCallback callback);

  // Cause the parent element to fill *its* parent. This is most useful for
  // top-level elements (e.g., <body>).
  public void fillParent();
}
```

_Animation_

The Layout class is also responsible for animating from one set of constraints
to the next. This is necessary and cannot be layered on top of the layout
system because changes in constraints (e.g., from {left, width} to {left,
right}) requires significant calculation in order to animate smoothly. One
consequence of this is that you must set each Layer's constraints, then call a
method on the Layout class in order to either jump to the new set of
constraints or animate to them.

_Programmatic layout_

While most of the layout work should, and will, be performed by the browser's
layout engine, there are sometimes cases where a widget absolutely must perform
some fixup work when its size changes. Unfortunately, no browser apart from IE
implements the "onresize" event, which would make this simple to implement
efficiently. The only way around this is to walk the hierarchy whenever a
top-level widget's size changes, giving each widget in the hierarchy a chance
to perform necessary fixup. However, doing this for every widget could become
prohibitively expensive. I propose that we deal with this by adding a
`RequiresLayout` interface, and only walking the parts of the hierarchy that form
an unbroken chain of `RequiresLayout` widgets.

## Benefits

_Very fast layout_

'nuff said.

_Completely predictable layout_

Also 'nuff said.

_Better window resize handling on Firefox_

Gecko-based browsers such as Firefox typically don't send window resize events
frequently, often waiting several seconds to fire an event after the window is
resized. Layout systems that perform all of their work in Javascript, usually
in response to the window resize event, can spend a significant amount of time
in an ugly state on Firefox. Because this system uses the browser's native
layout behavior, it is not as subject to this.

_Faster reflow_

Reflow can become very expensive for complex DOM structures. However, the
browser's layout engine can stop reflowing at absolutely-positioned elements.
Because this system uses absolute positioning, it tends to break the document
into smaller sub-trees that can be reflowed separately.

## Drawbacks

_All sizes must be explicit_

Using this layout mechanism requires that all children be explicitly sized, or
that their sizes be otherwise implied by their constraints. Practically
speaking, this means that certain layout widgets will require sizes as part of
their APIs, where they were previously implicit. For example, the new version
of DockPanel will likely have an add method that looks like this:

```
  boolean add(Widget child, Direction direction, double size, Unit unit);
```

This is mitigated somewhat by the fact that you can use any unit you like,
including font- and container-relative units such as "em" and "%". This also
means that each parent element managed by a Layout will have to be explicitly
sized, because its absolutely-positioned children will not influence its size.

_Centering can still be difficult_

Oddly, it is impossible to express the idea that a child element must be
centered, but with a fixed width or height, using these constraints. You can
get close to this by specifying that a child element be spaced an equal
distance from both edges (e.g., {left:25%; right:25%;} will give you an element
that is centered and takes 50% of its parent's width).

## Widgets

Because of the need for explicit sizes, it will not be possible to preserve
that APIs of all existing widgets (c.f. the `DockPanel.add()` example above).
Several widgets are likely to be either modified or rewritten as a result (note
that we definitely **won't** break existing widgets, though some will be
replaced).

Those that will need to be modified or extended:
  * RootPanel
  * PopupPanel
  * ScrollPanel
  * GlassPanel

Those that will need to be replaced:
  * AbsolutePanel
  * DockPanel
  * StackPanel
  * SplitPanel
  * Deck/TabPanel

## IE6?

Those familiar with the vagaries of CSS may have noted by now that these styles
don't behave as expected on IE6 (though they do on IE7). To accomodate IE6, we
can use the "onresize" event, combined with a fair amount of javascript, to
emulate their behavior. This results in about 3k of extra code for the IE6
permutation, which is unnecessarily downloaded for IE7 (since the determination
of whether to use it must be made dynamically), but this seems like a
reasonable tradeoff--especially since IE8 is eating away at IE7's installed
base.

## Alternatives

_CSS Layout_

There are many, many articles on the web describing various tricks to create
layouts using CSS. See the CSS Zen Garden for plenty of examples. Most of these
are geared towards laying out static HTML, though some articles propose that
they be used for complex Ajax applications as well. There are far too many
approaches to document here, but one approach that is frequently used, and is
similar to many other patterns, is as follows:

  * `.leftPanel { float: left; width: 128px; }`
  * `.content { margin-left: 128px; }`

There are several problems with this approach that make it untenable for our purposes:
  * It's very difficult to generalize and compartmentalize layout:
    * Different components need to know about one-another, and there's little consistency to the various tricks.
    * Even worse, there are a lot of different hacks one needs to know about to make it work across browsers.
  * It usually still requires script tricks to make a layout fill vertical space, and to get interior scrolling.
  * It can be somewhat slow, because almost any change to the DOM can cause the entire page to be re-laid-out.

_Absolute Javascript Layout_

A number of existing Javascript and GWT libraries take matters entirely into
their own hands by absolutely-positioning all (or at least most) containers in
pixel units and writing code to perform all layout manually. This has a lot of
appeal, because you can simply build your layout algorithms as you see fit, so
it's completely predictable and quite flexible. It does have one very
significant problem, though: It can be quite slow.