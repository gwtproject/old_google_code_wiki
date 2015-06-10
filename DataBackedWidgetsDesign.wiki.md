**NOTE:** This is a work in progress and by no means finalized. Please leave comments if you see something that you love or hate.



# Introduction

Data backed widgets will replace the FastTree and PagingScrollTable widgets in incubator. They will incorporate models and views that efficiently display large amounts of data to users.


## Bike Shed

We've created a new subdirectory under trunk/ where you can view the data backed widget code as we work on it. This directory and the code it contains is intended to be highly unstable, so please do not rely on it in your applications.

Bike Shed includes a sample application (a stock trading game) that we are using to test the design.  You can include the project in Eclipse.

http://code.google.com/p/google-web-toolkit/source/browse/#svn/trunk/bikeshed



---

# Cells

[Cells](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/cells/client/Cell.java) are the basic blocks used to render the View and interpret events.  Cells are typed based on the type of data that the cell represents, and they must implement a `render` method that renders the typed value as an HTML string.  In addition, cells can override `onBrowserEvent` to act as a flyweight that handles events that are fired on elements that were rendered by the cell.


## Mutators

Cells can be associated with a [Mutator](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/cells/client/Mutator.java) that allows the cell to modify an Object in response to an event fired on a rendered Cell.

For example, in a table that represents object R (the row value), there may be multiple cells, each with a different type that represents a field in R.  The first column could use a [CheckboxCell](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/cells/client/CheckboxCell.java) to render check boxes based on some boolean inside of R.  When the user clicks on one of the check boxes, CheckboxCell can inform the application controller that a certain field in R has been set to true or false.



---

# Tables and Lists

## ListModel

[ListModel](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/list/shared/ListModel.java) will supply data to tables and lists.  It has one method to add ListHandlers, which receive event when data becomes available or is updated.

### AbstractListModel

[AbstractListModel](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/list/shared/AbstractListModel.java) provides a default implementation of ListModel with convenience methods for pushing data to the Views.  In particular, `updateDataSize` and `updateViewData` allow you to forward new data to the Views using data such as an RPC result.

### ListListModel

[ListListModel](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/list/shared/ListListModel.java) is a concrete ListModel backed by a List.  Any changes to the internal list, which can be accessed via `getList()` will be reflected in the model and forwarded to the views.

We use a DeferredCommand to ensure that the Views aren't spammed if you are performing multiple List operations in a single event loop.

### AsyncListModel

[AsyncListModel](http://code.google.com/p/google-web-toolkit/source/browse/trunk/bikeshed/src/com/google/gwt/list/shared/AsyncListModel.java) is a concrete class that links to an asynchronous DataSource.  It allows you to connect the model to an application level controller that uses RPC requests to fetch data.


## ListHandler

A ListHandler is created by the ListView (the widget) that needs to be updated as data becomes available or changes.  onDataChanged() is called any time new data is available.  We are considering adding a "Hint" (such as insert, remove, modify) so the view can optimize its rendering path.  For example, a table may choose to just insert a DOM row instead of re-rendering all data on an insert hint.

The other method, onSizeChanged(), lets the view know that the total number of items has changed.  Views that care about this, such as paging tables that want to display the total number of pages, can update themselves.  However, it is not required that the total number of items be known.

```
/**
 * Handler interface for {@link ListEvent} events.
 * 
 * @param <T> the value about to be changed
 */
public interface ListHandler<T> extends EventHandler {

  /**
   * Called when {@link ListEvent} is fired.
   * 
   * @param event the {@link ListEvent} that was fired
   */
  void onDataChanged(ListEvent<T> event);

  /**
   * Called when a {@link SizeChangeEvent} is fired.
   * 
   * @param event the {@link SizeChangeEvent} that was fired
   */
  void onSizeChanged(SizeChangeEvent event);
}
```


## ListRegistration

When a ListHandler is added to the ListModel, the ListModel returns a ListRegistration.  The ListRegistration is used by the view to set the range of interest, as in the items in the list that the view cares about.  It is the responsibility of the ListModel to provide the data as it becomes available.

```
/**
 * Registration returned from a call to ListModel#addListHandler.
 */
public interface ListRegistration extends HandlerRegistration {

  /**
   * Set the range of data that is interesting to the {@link ListHandler}.
   * 
   * @param start the start index
   * @param length the length
   */
  void setRangeOfInterest(int start, int length);
}
```



---

# Trees

We'll be working on this next...



---

# TreeTables

We'll tackle this beast later...