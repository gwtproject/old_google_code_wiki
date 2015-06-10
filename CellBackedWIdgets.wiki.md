# Table of Contents


# Introduction

Existing GWT Widgets construct a specific DOM structure to represent the widget, then apply specific style names to specific elements within the structure.  This works fine for users who are satisfied with the default look and feel of the Widget, but it frustrates users who want to reskin a widget in more than a trivial way.  The assumptions that current widgets make regarding their structure prevent us from modernizing existing widgets without breaking applications that rely on the existing DOM structures.

Due to this constraint, the GWT widget library is becoming increasingly out of date, so the time is ripe for us to revamp the Widget library. This document outlines a pattern for a new Cell-Backed Widget library that enables the following common use cases:
  * Replace the styles of a widget instance
  * Replace the DOM structure of a widget instance
  * Reskin an entire GWT app (DOM and/or styles)
    * Allow third parties to provide skins
  * Isolate CSS code for each widget
    * Dead strip CSS code that is not used within the app
  * Separate presenter logic from DOM view
  * Offer an identical Cell equivalent of every (most) new widgets
    * Shared code between Cell and Widget

New Widgets in GWT will be backed by a Cell, so users can call upon common view components for CellTable and the rest of their application. GWT already provides the {{{CellWidget}} wrapper class that can convert any Cell into a Widget. More details on this appear toward the end of the document, but understand that when we refer to Cells throughout this document, all Cells will have an equivalent Widget.



# Appearance Pattern

Each Cell will expose an abstract Appearance class that plays two important roles:
  1. Renders the view as a SafeHtml string.
  1. Updates the view in response to state changes of the widget, such as by applying styles to elements.

In this way, the Cell becomes the presenter for the Appearance (the view).  Cells will be written such that they do not rely on a specific DOM structure.  For example, in `ButtonCell` below, the Appearance could be rendered as a `button` element or a series of nested divs, but the Cell would still respond to click events in the same way.

```
public class ButtonCell extends AbstractCell<String> {

  // The appearance of this cell.
  public abstract static class Appearance {

    // Called when the user hovers the button with the mouse.
    public void onHover(Context context, Element parent, String value) {}

    // Called when the user pushes the button down.
    public void onPush(Context context, Element parent, String value) {}

    // Called when the user unhovers the button with the mouse.
    public void onUnhover(Context context, Element parent, String value) {}

    // Called when the user releases the button from being pushed.
    public void onUnpush(Context context, Element parent, String value) {}

    // Render the DOM structure of the cell.
    public abstract void render(Context context, SafeHtml data, SafeHtmlBuilder sb);
  }

  // The default implementation of the appearance.
  public static class DefaultAppearance extends Appearance {

    public static interface Resources extends ClientBundle {
      @Source(Style.DEFAULT_CSS)
      Style buttonCellStyle();
    }

    @ImportedWithPrefix("gwt-ButtonCell")
    public interface Style extends CssResource {
      String DEFAULT_CSS = "com/google/gwt/cell/client/ButtonCell.css";

      String hover();

      String push();
    }

    private final Resources resources;

    public DefaultAppearance() {
      this(GWT.create(Resources.class));
    }

    public DefaultAppearance(Resources resources) {
      this.resources = resources;
      resources.buttonCellStyle().ensureInjected();
    }

    @Override
    public void onHover(Context context, Element parent, String value) {
      addClassName(parent, resources.buttonCellStyle().hover());
    }

    @Override
    public void onPush(Context context, Element parent, String value) {
      addClassName(parent, resources.buttonCellStyle().push());
    }

    @Override
    public void onUnhover(Context context, Element parent, String value) {
      removeClassName(parent, resources.buttonCellStyle().hover());
    }

    @Override
    public void onUnpush(Context context, Element parent, String value) {
      removeClassName(parent, resources.buttonCellStyle().push());
    }

    @Override
    public void render(Context context, SafeHtml data, SafeHtmlBuilder sb) {
      sb.appendHtmlConstant("<button type=\"button\" tabindex=\"-1\">");
      if (data != null) {
        sb.append(data);
      }
      sb.appendHtmlConstant("</button>");
    }

    protected void addClassName(Element parent, String styleName) {
      parent.getFirstChildElement().addClassName(styleName);
    }

    protected void removeClassName(Element parent, String styleName) {
      parent.getFirstChildElement().removeClassName(styleName);
    }
  }

  private final Appearance appearance;

  // Use the default look and feel of the Cell.
  public ButtonCell() {
    this(GWT.create(Appearance.class));
  }

  // Replace the styles used by this cell instance.
  public ButtonCell(DefaultAppearance.Resources resources) {
    this(new DefaultAppearance(resources));
  }

  // Replace the appearance used by this cell instance.
  public ButtonCell(Appearance appearance) {
    this.appearance = appearance;
  }

  ...
}
```

First, notice that the Appearance class does not provide any view implementations, and thus the Cell does not make any assumptions about the structure of the Appearance.  Even the Resources and Styles live in the default implementation of the appearance, instead of in the Cell itself.  In the DefaultAppearence, we update state by simply applying a class name to the outer `button` element.  A more complex Appearance may apply class names not just to the outer element, but to inner elements as well.

Now take a look at the constructors for ButtonCell.  The default constructor calls `GWT.create(Appearance.class)`, which is bound to the DefaultAppearance.  However, users can use the third constructor to provide a specific appearance for this instance of the Cell. Alternatively, the second constructor (which is just there for convenience) takes a `DefaultAppearance.Resources`, which allows users to replace just the styles of the DefaultAppearance.

As mentioned, we call `GWT.create(Appearance.class)` by default.  The DefaultAppearance is bound in a gwt.xml file:
```
  <!-- Bind the default ButtonCell.Appearance. -->
  <replace-with class="com.google.gwt.cell.client.ButtonCell.DefaultAppearance">
    <when-type-is class="com.google.gwt.cell.client.ButtonCell.Appearance"/>
  </replace-with>
```

By overriding the deferred binding rule, a user can reskin a Cell for an entire application.  Alternatively, third party libraries can provide skins for multiple widgets, which would be applied by inheriting a single Module containing a bunch of deferred binding overrides. This also allows the GWT team to replace Appearances with more modern ones from time to time without breaking applications.

Finally, note that Appearance is an abstract class.  This allows us to add more state logic in the future without breaking existing Appearances.  For example, we could add new methods "setRightFlush()" and "setLeftFlush()" which would make the right and left edges of the button flat, such that they could be lined up next to each other. Existing appearances may not support the new feature, but they would still work.


## Okay, _Some_ Assumptions

In some cases, the implementation of an Appearance will have a significant effect on the Cell logic.  For example, `SelectionCell.Appearance` could use the native `select` element to render a list, or the Appearance could render a custom list using divs.  In the first case, we can rely on a native change event, but in the latter case, we would need custom logic to handle selection and keyboard movement.  Pushing this logic down to the Appearance would defeat the purpose of the pattern and wouldn't make the Cell very reusable.

In these cases, we would provide two Cells, such as SelectionCell and ListBoxCell.  SelectionCell would require that the Appearance contain a native select element and would respond to native change events.  ListBoxCell would respond provide logic for keyboard and mouse events, which can be mapped into the Appearance (glossy I know.  More details when we actually implement it).

This edge case appears to be limited to elements that can change state natively (things that fire onchange events), such as `select` elements and checkboxes.  For checkboxes, we would want to support using images instead of a native input element.


# Cell Backed Widgets

Every Cell will have an equivalent Widget, which is backed by the Cell and shares all of the logic.  We will use the existing `CellWidget` class to wrap the widget:
```
public class ButtonWidget extends CellWidget<String> {

  public ButtonWidget() {
    this(new ButtonCell());
  }

  public ButtonWidget(ButtonCell.Appearance appearance) {
    this(new ButtonCell(appearance));
  }

  public ButtonWidget(ButtonCell.DefaultAppearance.Resources resources) {
    this(new ButtonCell(resources));
  }

  public ButtonWidget(ButtonCell cell) {
    super(cell);
  }

  public void setTabIndex(int tabIndex) {
    getCell().setTabIndex(tabIndex);
  }

  protected ButtonCell getButtonCell() {
    return (ButtonCell) getCell();
  }
}
```

ButtonWidget provides convenience constructors to specify the `ButtonCell.Appearance` or `ButtonCell.DefaultAppearance.Resources`.  ButtonWidget inherits `addValueChangeHandler()` from CellWidget, but users can add additional event handlers just as they would with any other widget.

Some Cells support methods to change how the Cell is rendered.  For example, ButtonCell could provide a setTabIndex() method to set the tab index of the element.  ButtonWidget would expose the same methods and forward through to the Cell.



## Packages and Naming Conventions

Cells will continue to go in the `com.google.gwt.cell` package. In fact, existing Cells do not expose any ClientBundles or styles, so we can retrofit them using the Appearance pattern.

The new widget library will be a fairly significant departure from the old library, and so it probably deserves a new package to separate it.  I suggest `com.google.gwt.user.widget.client`.

Since we now provide both Cell and Widget versions, I suggest that new widgets use the postfix "Widget", such as "ButtonWidget".  That compliments "ButtonCell" nicely.  It also avoids naming conflicts with existing widgets, such as "Button".



# Conclusion

The new pattern described above will allow us to update widgets by providing new Appearances to replace old Appearances.  We will start with the basic form widgets (buttons, text boxes, etc.) and move on to more complex widgets over time.