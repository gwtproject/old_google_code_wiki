_This article is a bit old (but still correct). I've added a more recent writeup at UnderstandingMemoryLeaks, with a bit more advice on dealing with these issues in practice._

You may ask yourself, "Why do I have to use bitfields to sink DOM events?", and you may ask yourself, "Why can I not add event listeners directly to elements?". If you find yourself asking these questions, it's probably time to dig into the murky depths of DOM events and memory leaks.

If you're creating a Widget from scratch (using DOM elements directly, as opposed to simply creating a Composite), the setup for event handling is a bit odd. Generally, it looks something like this:

```
class MyWidget extends Widget {
  public MyWidget() {
    setElement(DOM.createDiv());
    sinkEvents(Event.ONCLICK);
  }

  public void onBrowserEvent(Event evt) {
    switch (DOM.eventGetType(evt)) {
      case Event.ONCLICK:
        // Do something insightful.
        break;
    }
  }
}
```

This may seem a bit obtuse, but there's a good reason for it. To understand this, you may first need to brush up on browser memory leaks. There are some good resources on the web:
  * http://www.quirksmode.org/blog/archives/2005/02/javascript_memo.html
  * http://www-128.ibm.com/developerworks/web/library/wa-memleak/

The upshot of all this is that in some browsers, any reference cycle that involves a Javascript object and a DOM element (or other native object) has a nasty tendency to never get garbage-collected. The reason this is so insidious is that this is an extremely common pattern to create in Javascript UI libraries. Imagine the following (raw Javascript) example:

```
function makeWidget() {
  var widget = {};
  widget.someVariable = "foo";
  widget.elem = document.createElement ('div');
  widget.elem.onclick = function() {
    alert(widget.someVariable);
  };
}
```

Now, I'm not suggesting that you'd really build a Javascript library quite this way, but it serves to illustrate the point. The reference cycle created here is:

```
widget -> elem(native) -> closure -> widget
```

There are many different ways to run into the same problem, but they all tend to form a cycle that looks something like this. This cycle will never get broken unless you do it manually (often by clearing the `onclick` handler).

There are a number of ways developers try to deal with this issue. One of the more common I've seen is to walk the DOM when `window.onunload` is fired, clearing out all event listeners. This is problematic for two reasons:

  * It doesn't clear events on elements that are no longer in the DOM.
  * It doesn't deal with long-running applications, which are becoming more and more common.

## GWT's Solution

When designing GWT, we decided that leaks were simply unacceptable. You wouldn't tolerate egregious memory leaks in a desktop application, and a browser app should be no different. This raises some interesting problems, though. In order to avoid ever creating leaks, any Widget that **might** need to be get garbage collected must not be involved in a reference cycle with a native element. There's no way to find out "when a widget would have been collected had it not been involved in a reference cycle". So in GWT terms, a Widget must not be involved in a cycle when it is detached from the DOM.

How do we enforce this? Each Widget has a single "root" element. Whenever the Widget becomes attached, we create exactly one "back reference" from the element to the Widget ( i.e. `elem.__listener = widget`, performed in `DOM.setEventListener()`). This is set whenever the Widget is attached, and cleared whenever it is detached.

Which brings is back to that odd bitfield used in `sinkEvents()`. If you look at the implementation of `DOM.sinkEvents()`, you'll see that it does something like this:

```
  elem.onclick = (bits & 0x00001) ? $wnd.__dispatchEvent : null;
```

Each element's events point back to a central dispatch function, which looks for the target element's `__listener` expando, in order to call `onBrowserEvent()`. The beauty of this is that it allows us to set and clear a single expando to clean up any potential event leaks.

What this means in practice is that, as long as you don't set up any reference cycles on your own using JSNI, you can't write an app in GWT that will leak. We test carefully with every release to make sure we haven't done anything in the low-level code to introduce new leaks as well.

The downside, of course, is that you can't hook event listeners directly to elements that are children of a Widget's element. Rather, you have to receive the event on the Widget itself, and figure out which child element it came from.

But that's better than leaking gobs of memory on your users' machines, right?