# GWT Editor Framework

The GWT Editor framework allows data stored in an object graph to be mapped onto a graph of Editors.  The typical scenario is wiring objects returned from an RPC mechanism into a UI.



## Goals
  * Decrease the amount of glue code necessary to move data from an object graph into a UI and back.
  * Be compatible with any object that looks like a bean, regardless of its implementation mechanism (POJO, JSO, RPC, RequestFactory).
  * Support arbitrary composition of Editors.
  * For post-GWT 2.1 release, establish the following trajectories:
    * Create an API that can be used by UiBinder
    * Support client-side JSR 303 Validation when it's available

## Quickstart

Import the `com.google.gwt.editor.Editor` module in your `gwt.xml` file.

```
// Regular POJO, no special types needed
public class Person {
  Address getAddress();
  Person getManager();
  String getName();
  void setManager(Person manager);
  void setName(String name);
}

// Sub-editors are retrieved from package-protected fields, usually initialized with UiBinder.
// Many Editors have no interesting logic in them
public class PersonEditor extends Dialog implements Editor<Person> {
  // Many GWT Widgets are already compatible with the Editor framework
  Label nameEditor;
  // Building Editors is usually just composition work
  AddressEditor addressEditor;
  ManagerSelector managerEditor;

  public PersonEditor() {
    // Instantiate my widgets, usually through UiBinder
  }
}

// A simple demonstration of the overall wiring
public class EditPersonWorkflow{
  // Empty interface declaration, similar to UiBinder
  interface Driver extends SimpleBeanEditorDriver<Person, PersonEditor> {}

  // Create the Driver
  Driver driver = GWT.create(Driver.class);

  void edit(Person p) {
    // PersonEditor is a DialogBox that extends Editor<Person>
    PersonEditor editor = new PersonEditor();
    // Initialize the driver with the top-level editor
    driver.initialize(editor);
    // Copy the data in the object into the UI
    driver.edit(p);
     // Put the UI on the screen.
    editor.center();
  }

  // Called by some UI action
  void save() {
    Person edited = driver.flush();
    if (driver.hasErrors()) {
      // A sub-editor reported errors
    }
    doSomethingWithEditedPerson(edited);
  }
}
```

## Definitions

  * _Bean-like object_: (henceforth "bean") An object that supports retrieval of properties through strongly-typed `Foo getFoo()` methods with optional `void setFoo(Foo foo);` methods.
  * _Editor_: An object that supports editing zero or more properties of a bean.
    * An Editor may be composed of an arbitrary number of sub-Editors that edit the properties of a bean.
    * Most Editors are Widgets, but the framework does not require this.  It is possible to create "headless" Editors that perform solely programmatically-driven changes.
  * _Driver_: The "top-level" controller used to attach a bean to an Editor.  The driver is responsible for descending into the Editor hierarchy to propagate data.  Examples include the [SimpleBeanEditorDriver](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/editor/client/SimpleBeanEditorDriver.java) and the [RequestFactoryEditorDriver](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/requestfactory/client/RequestFactoryEditorDriver.java).
  * _Adapter_: One of a number of provided types that provide "canned" behaviors for the Editor framework.

## General workflow

  * Instantiate and initialize the Editors.
    * If the Editors are UI based, this is usually the time to call `UiBinder.createAndBindUi()`
  * Instantiate and initialize the driver.
    * Drivers are created through a call to `GWT.create()` and the specific details of the initialization are driver-dependent, although passing in the editor instance is common.
    * Because the driver is stateful, driver instances must be paired with editor hierarchy instances.
  * Start the editing process by passing the bean into the driver.
  * Allow the user to interact with the UI.
  * Call the `flush()` method on the driver to copy Editor state into the bean hierarchy.
  * Optionally check `hasErrors()` and `getErrors()` to determine if there are client-side input validation problems.

## Editor contract

The basic `Editor` type is simply a parameterized marker interface that indicates that a type conforms to the editor contract or informal protocol.  The only expected behavior of an `Editor` is that it will provide access to its sub-Editors via one or more of the following mechanisms:
  * An instance field with at least package visibility whose name exactly the property that will be edited or `propertyNameEditor`.  For example:
```
class MyEditor implements Editor<Foo> {
  // Edits the Foo.getBar() property
  BarEditor bar;
  // Edits the Foo.getBaz() property
  BazEditor bazEditor;
}
```
  * A no-arg method with at least package visibility whose name exactly is the property that will be edited or `propertyNameEditor`.  This allows the use of interfaces for defining the Editor hierarchy. For example:
```
interface FooEditor extends Editor<Foo> {
  // Edits the Foo.getBar() property
  BarEditor bar();
  // Edits the Foo.getBaz() property
  BazEditor bazEditor();
}
```
  * The `@Path` annotation may be used on the field or accessor method to specify a dotted property path or to bypass the implicit naming convention.  For example:
```
class PersonEditor implements Editor<Person> {
  // Corresponds to person.getManager().getName()
  @Path("manager.name");
  Label managerName;
}
```
    * The zero-length string (i.e. `@Path("")`) may be used to map the "current value" into a sub-editor.
  * The `@Ignored` annotation may be used on a field or accessor method to make the Editor framework ignore something that otherwise appears to be a sub-Editor.
  * Sub-Editors may be null. In this case, the Editor framework will ignore these sub-editors.

Where the type `Editor<T>` is used, the type `IsEditor<Editor<T>>` may be substituted.  The `IsEditor` interface allows composition of existing Editor behavior without the need to implement N-many delegate methods in the composed Editor type.  For example, most leaf GWT Widget types implement `IsEditor` and are immediately useful in an Editor-based UI.  By implementing `IsEditor`, the Widgets need only implement the single `asEditor()` method, which isolates the Widgets from any API changes that may occur in the component Editor logic.

## Editor delegates

Every `Editor` has a peer `EditorDelegate` that provides framework-related services to the Editor.
  * `getPath()` returns the current path of the Editor within an attached Editor hierarchy.
  * `recordError()` allows an Editor to report input validation errors to its parent Editors and eventually to the driver.  Arbitrary data can be attached to the generated `EditorError` by using the `userData` parameter.
  * `subscribe()` can be used to receive notifications of external updates to the object being edited.  Not all drivers may support subscription.  In this case, the call to `subscribe()` may return `null`.

## Editor subtypes

In addition to the `Editor` interface, the Editor framework looks for these specific interfaces to provide basic building blocks for more complicated Editor behaviors.  This section will document these interfaces and provide examples of how the Editor framework will interact with the API at runtime.  All of these core Editor sub-interface can be mixed at will.

### LeafValueEditor
`LeafValueEditor` is used for non-object, immutable, or any type that the Editor framework should not descend into.
  1. `setValue()` is called with the value that should be edited (e.g. `fooEditor.setValue(bean.getFoo());`).
  1. `getValue()` is called when the Driver is flushing the state of the Editors into the bean.  The value returned from this method will be assigned to the bean being edited (e.g. `bean.setFoo(fooEditor.getValue());`).

### HasEditorDelegate
`HasEditorDelegate` provides an Editor with its peer `EditorDelegate`.
  1. `setEditorDelegate()` is called before any value initialization takes place.

### ValueAwareEditor
`ValueAwareEditor` may be used if an Editor's behavior depends on the value that it is editing, or if the Editor requires explicit flush notification.
  1. `setEditorDelegate()` is called, per `HasEditorDelegate` super-interface.
  1. `setValue()` is called with the value that the Editor is responsible for editing.  If the value will affect with sub-editors are or are not provided to the framework, they should be initialized or nullified at this time.
  1. If `EditorDelegate.subscribe()` has been called, the Editor may receive subsequent calls to `onPropertyChange()` or `setValue()` at any point in time.
  1. `flush()` is called in a depth-first manner by the driver, so Editors generally do not flush their sub-Editors.  Editors that directly mutate their peer object should do so only when `flush()` is called in order to allow an edit workflow to be canceled.

### CompositeEditor
`CompositeEditor` allows an unknown number of homogenous sub-Editors to be added to the Editor hierarchy at runtime.  In addition to the behavior described for `ValueAwareEditor`, `CompositeEditor` has the following additional APIs:
  1. `createEditorForTraversal()` should return a canonical sub-editor instance that will be used by the driver for computing all edited paths.  If the composite editor is editing a Collection, this method solves the problem of having no sub-Editors available to examine for an empty Collection.
  1. `setEditorChain()` provides the `CompositeEditor` with access to the `EditorChain`, which allows the component sub-Editors to be attached and detached from the Editor hierarchy.
  1. `getPathElement()` is called by the Editor framework for each attached component sub-Editor in order to compute the return value for `EditorDelegate.getPath()`.  A `CompositeEditor` that is editing an indexable datastructure, such as a `List`, might return `[index]` for this method.

### HasEditorErrors
`HasEditorErrors` indicates that the Editor wishes to receive any unconsumed errors reported by sub-Editors through `EditorDelegate.recordError()`.  The Editor may mark an `EditorError` as consumed by calling `EditorError.setConsumed()`.

## Provided Adapters

The GWT distribution provides the following Editor adapter classes that provide reusable logic.  To reduce the amount of generics boilerplate, most types are equipped with a static `of()` method to

  * `HasDataEditor` adapts a `List<T>` to a `HasData<T>`.
  * `HasTextEditor` adapts the `HasText` interface to `LeafValueEditor<String>`.
    * New widgets should prefer `TakesValue<String>` over `HasText`.
  * `ListEditor` keeps a `List<T>` in sync with a list of sub-Editors.
    * The `ListEditor` is created with a user-provided `EditorSource` which vends sub-Editors (usually Widget subtypes).
    * Changes made to the structure of the `List` returned by `ListEditor.getList()` will be reflected in calls made to the `EditorSource`.
    * [Sample code](http://code.google.com/p/google-web-toolkit/source/browse/trunk/samples/dynatablerf/src/com/google/gwt/sample/dynatablerf/client/widgets/FavoritesWidget.java).
  * `OptionalFieldEditor` can be used with nullable or resettable bean properties.
  * `SimpleEditor` can be used as a headless property Editor
  * `TakesValueEditor` adapts a `TakesValue<T>` to a `LeafValueEditor<T>`
  * `ValueBoxEditor` adapts a `ValueBoxBase<T>` to a `LeafValueEditor<T>`.  If the `getValueOrThrow()` method throws a `ParseException`, the exception will reported via an `EditorError`.
  * `ValueBoxEditorDecorator` is a simple UI decorator that combines a `ValueBoxBase` with a `Label` to show any parse errors from the contained `ValueBoxBase`.

## Driver types

The GWT Editor framework provides the following top-level drivers:
  * `SimpleBeanEditorDriver` can be used with any bean-like object.
    * It does not provide support for update subscriptions.
  * `RequestFactoryEditorDriver` is designed to integrate with `RequestFactory` and edit `EntityProxy` subtypes.
    * This driver type requires a `RequestContext` in order to automatically call `RequestContext.edit()` on any `EntityProxy` instances that are encountered.
    * Subscriptions are supported by listening for `EntityProxyChange` events on the `RequestFactory`'s `EventBus`.

The top-level drivers extend a base interface, `EditorDriver`, which provides the following methods:
  * `accept(EditorVisitor)` allows traversal of the Editor hierarchy.
  * `getErrors()` returns all unconsumed errors generated during the last call to `flush()`.
  * `hasErrors()` returns a boolean value.
  * `isDirty()` returns `true` if any of the Editors in the hierarchy have been modified felative to the last value passed into `edit()`.
  * `setConstraintViolations()` maps `ConstraintViolation` objects into the editor hierarchy.

## EditorVisitor

The Editor framework provides for traversal of an Editor hierarchy via a visitor pattern.  Each Editor in the hierarchy will be visited with a paired `visit(EditorContext) / endVisit(EditorContext)` call in the `EditorVisitor` passed into `EditorDriver.accept()`.  The `EditorContext` object provides the following methods:
  * `asXYZEditor()` is a convenience casting method which will return `null` if the current Editor does not conform to the desired interface.
```
LeafValueEditor asLeaf = editorContext.asLeafValueEditor();
if (asLeaf != null) {
  doSomething(asLeaf.getValue());
}
```
  * `canSetInModel()` returns `true` if `setInModel(Object)` can be called.
  * `checkAssignment(Object)` avoids the need for an unsafe generic cast when using `setInModel`.
  * `getAbsolutePath()` returns the absolute path of the current Editor.
  * `getEditedtype()` returns the base type the Editor operates on.
  * `getEditor()` returns the current Editor
  * `getEditorDelegate()` will return the `EditorDelegate` associated with the current Editor, if it exists.  Editors that only implement `LeafValueEditor` do not have `EditorDelegate` instances.
  * `getFromModel()` queries the object graph being edited for the value the Editor should display.
  * `setInModel()` updates the object graph being edited with a new value.  This method should always be accompanied by a check of `canSetInModel()` since not all values may be mutable in the model graph.
  * `traverseSyntheticCompositeEditor()` allows an editor created by `CompositeEditor.createEditorForTraversal()` to be traversed in order to examine the canonical structure of the composite Editors.

As all of the built-in Editor framework behaviors are built in terms of `EditorVisitor`, it can be helpful to refer to [implementation classes](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/gwt/editor/client/impl/Flusher.java) for examples of how to implement additional behaviors.

## FAQ

### Editor vs. IsEditor

**Q:** Should my `Widget` implement an `Editor` interface or `IsEditor`?

**A:** If the `Widget` contains multiple sub-Editors is a simple, static hierarchy, use the `Editor` interface.

The `IsEditor` interface is intended to be used when a view type is reusing an Editor behavior provided by an external type.  For instance, a `LabelDecorator` type would implement `IsEditor` because it re-uses its Label's existing Editor behavior:
```
class LabelDecorator extends Composite implements IsEditor<LeafValueEditor<String>> {
  private final Label wrapped = new Label();

  public LabelDecorator() {
    // Construct a pretty UI around the wrapped label
    initWidget(prettyContents);
  }

  public LeafValueEditor<String> asEditor() {
    return wrapped.asEditor();
  }
}
```
Similarly a `WorkgroupMembershipEditor` might implement `IsEditor<ListEditor<Person, PersonNameLabel>>`.

### Read-only Editors
**Q:** Can I use Editors to view read-only data?

**A:** Yes, just don't call the `flush()` method on the driver type.   `RequestFactoryEditorDriver` has a convenience `display()` method as well.

### Very large objects
**Q:** How can I edit objects with a large number of properties?

**A:** An Editor doesn't have to edit all of the properties of its peer domain object. If you had a `BagOfState` type with many properties, it might make sense to write several Editor types that edit conceptually-related subsets of the properties:
```
class BagOfStateBiographicalEditor implements Editor<BagOfState> {
  AddressEditor address;
  Label name; 
}

class BagOfStateUserPreferencesEditor implements Editor<BagOfState> {
  CheckBox likesCats;
  CheckBox likesDogs;
}
```
Whether or not these editors are displayed all at the same time or sequentially is a user experience issue.  The Editor framework allows multiple Editors to edit the same object:
```
class BagOfStateEditor implements Editor<BagOfState> {
  // Use the zero-length string to map the peer object into the sub-editor
 @Editor.Path("")
 BagOfStateBiographicalEditor bio;

 @Editor.Path("")
 BagOfStateUserPreferencesEditor prefs;
}
```