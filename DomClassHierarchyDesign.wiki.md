# Explicit DOM Classes in GWT

Joel Webber, Bruce Johnson

## Background

[Recent enhancements](JavaScriptObjectRedesign.md) to GWT's treatment of the JavaScriptObject (JSO) class in GWT 1.5 have
made it possible to extend JSO with user-defined subclasses having arbitrary methods.

The GWT 1.5 UI library uses this newfound capability to more accurately model the W3C Node and HTML Element type
hierarchy. In addition to aesthetic improvements in GWT-based DOM coding, the new model provides a more consistent and
reliable cross-browser DOM API than JavaScript/DHTML itself because it builds on GWT's deferred binding approach to
browser quirks specialization.

## Goals
  * Provide explicit classes for important DOM types so as to...
    * add clarity to Widget and ad hoc DHTML coding
    * provide a clear mechanism for documenting DOM-level APIs
    * facilitate type checking within Java IDEs and at compile time
    * facilitate code completion in Java IDEs to make DOM-level coding more productive and less prone to errors
  * Suffer exactly zero additional size/speed overhead due to the new DOM types
  * Where possible, expose an already-vetted pattern for exposing HTML in terms of Java types (i.e. the W3C DOM interfaces)

## Non-Goals
  * Improve the manner in which HTML concepts are mapped onto Java types by attempting to add more type specificity than the inconsistency of the underlying DHTML models actually allow for (in other words, we aren't trying to invent something new)
  * Prevent runtime errors statically via the type system when DOM classes are misused; see JavaScriptObjectRedesign for details on how casts among JSO subclasses are treated; there are limits for how well this could work even in theory

## Details
This enhancement involves the introduction of a new class hierarchy that extends the JavaScriptObject class. A sketch
of the hierarchy is listed below, although individual methods are too numerous to list. (See the related patch for full
details.). These class names mirror the W3C HTML specification (including their inconsistencies, such as `LIElement` vs.
`OListElement`).

```
JavaScriptObject
  Node
    Text
    Document
    Element
      AnchorElement
      BodyElement
      ButtonElement
      DivElement
      FieldSetElement
      FormElement
      IFrameElement
      ImageElement
      InputElement
      LabelElement
      LegendElement
      LIElement
      OListElement
      OptionElement
      SelectElement
      SpanElement
      TableCaptionElement
      TableCellElement
      TableColElement
      TableElement
      TableRowElement
      TableSectionElement
      TextAreaElement
      UListElement
  Event
  NodeList
  Style
```

These classes take advantage of GWT's new JavaScriptObject method support to directly implement DOM methods such as
`appendChild()`, `setAttribute()`, and so forth.

## Benefits

### Prettier Code, Part 1: Bypassing GWT's (admittedly yucky) DOM class
This new, more explicit model has many benefits. Most obviously, the new classes clean up the aesthetics of DOM coding
in Widget implementations and other straight-to-the-DOM programming tasks.

Consider this example of a previous implementation from GWT 1.4:
```
Element getCellElement(int row, int col) {
  return DOM.getChild(DOM.getChild(tbody, row), col);
}
```

This looks much nicer when rewritten in GWT 1.5 like so:
```
Element getCellElement(int row, int col) {
  return tbody.getChildElement(row).getChildElement(col);
}
```

### Prettier Code, Part 2: Less need for JSNI
Prior to GWT 1.5 JSO improvements, GWT developers often needed to use JSNI to accomplish low-level operations in
JavaScript. For example,
```
private native Element getCellElement(Element table, int row, int col) /*-{
  var out = table.rows[row].cells[col];
  return (out == null ? null : out);
}-*/;
```

This has a few disadvantages:
  * JSNI methods don't provide code completion for the JavaScript code
  * A Java debugger can't be used to step through each line of the JSNI method body
  * JSNI methods can't be auto-formatted by IDE code formatters
  * Refactoring isn't generally supported (e.g. renaming parameters, introducing locals)

The new JSO support and Element subclasses allow the same code to be written fully in Java code:
```
private Element getCellElement(TableElement table, int row, int col) {
  return table.getRows().getItem(row).getCells().getItem(col);
}
```

### Improved correctness and documentation
  * Less JSNI means more opportunities for code completion, which results in fewer typo-induced bugs
  * Methods on the new DOM classes afford an opportunity to add sanity-checking assertions; in hosted mode, these assertions help quickly identify bugs, whereas in web mode they are automatically and completely removed


## Implementation

### Package Structure and Backwards Compatibility
All of the above objects, with the exception of JavaScriptObject and Event, will be created in the
com.google.gwt.dom.client package. Part of the DOMImpl abstraction (and its related browser-specific implementations)
will be moved into this package as well.

For backwards compatibility, all existing interfaces (i.e. DOM, Element, and Event) will remain in the
com.google.gwt.user.client package, with the following changes:
  * user.client.Element will extend dom.client.Element
  * The DOM static methods will have their argument types loosened to accept either Element class
  * The DOM static methods will delegate to the new Element method implementations (where possible)
  * The user.client.impl.DOMImpl abstractions will contain only those abstractions not moved to dom.client.DOMImpl

The only required change to the user.client.ui package will be to loosen the type of the argument to
`setElement()` (it will accept dom.client.Element), add an overload of setElement() that takes
user.client.Element (so that existing classes that override it will not break), and add a
`getRootElement()` that will return dom.client.Element.

These changes are 100% backwards compatible with existing code. One slightly unfortunate side-effect
of this is that `UIObject.getStyleElement()` still returns `user.client.Element`. It is tempting to
replace its return type with dom.client.Element (existing overrides would remain correct through
covariance), but this would run the risk of breaking callers (see the next section for an example of
how you can deal with this in practice).

### dom.client.Element and user.client.Element
If there are now two Element classes, which one should I use? For new code, you can always use
dom.client.Element. The old Element class has no new methods on it, so you lose nothing by doing
this, and gain direct convertibility (though a simple typecast) to its subtypes (e.g.
ButtonElement).

Converting existing code to use Element methods (instead of calls to static DOM methods) is very
straightforward. Almost all static DOM methods have obvious equivalents on dom.client.Element. The
only exceptions are event handling methods (see below) and the methods that allow you to enumerate
child Elements by index. These latter methods are not directly exposed in the Element API because
they were not a great idea in the first place -- they look like they should run in constant time but
are in fact linear time on all other browsers than IE.

If you do need to work with both Element types in the same code (which should occur very rarely),
keep in mind that they're freely convertible because of the way JavaScriptObject works (see
JavaScriptObjectRedesign for details).

### Event
The Event object has also been extended to provide methods to directly access its properties and
functions. It also contains static helper methods for sinking events on Elements (specifically,
`sinkEvents()` and `getEventsSunk()`). This object was not moved into the dom package because it is
not clear that this is the best, or only, way to deal with Javascript event handling.

### Changes to Existing Widgets
The above changes will not affect existing widgets (the only change to user.client.ui being the aforementioned tweaks to
UIObject). Once these changes are committed, we can update the existing widgets to use the new interfaces. This will be
effectively a no-op, but lead to much easier to read widget code.

### No overhead
All of these changes would be worthless if they imposed any performance or size overhead on the JavaScript produced by the GWT compiler.
Fortunately, the compiler has gotten very good at throwing away extra levels of abstraction. The following examples of
compiled output confirm this in practice.

The first example uses the existing DOM methods to manipulate a `<button>` element:
```
  com.google.gwt.user.client.Element elem = DOM.createButton();
  DOM.setStyleAttribute(elem, "color", "green");
  DOM.setInnerText(elem, "foo!");
  DOM.appendChild(RootPanel.getBodyElement(), elem);
```

The second uses the new Element interfaces to do the same thing:
```
  com.google.gwt.dom.client.ButtonElement elem = Document.get().createButtonElement();
  elem.getStyle().setAttribute("color", "green");
  elem.setInnerText("foo!");
  Document.get().getBody().appendChild(elem);
```

The following is the compiled output on Internet Explorer. Note that it's identical in both cases.
```
  var elem = $doc.createElement('button');
  elem.style['color'] = 'green';
  elem.innerText = 'foo!';
  $doc.body.appendChild(elem);
```