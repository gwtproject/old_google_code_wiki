# Design: Overlay Types
Bruce Johnson, Scott Blum, Lex Spoon, Bob Vawter

## Background
The `JavaScriptObject` class has been an extremely useful concept because it provides zero-overhead interoperation with external (typically, non-GWT) JavaScript while adding additional value to external JavaScript objects by representing them as actual Java types that are amenable to refactoring, code completion, and Javadoc-style documentation. However, subclassing `JavaScriptObject` was not generally supported before GWT 1.5 because we weren't sure if it was the right solution, or if we might need to evolve it in breaking ways.  This document describes the "old model" prior to GWT 1.5, and the new model, overlay types, which shipped in GWT 1.5 as well as extensions to overlay types that will first ship in GWT 2.0.

The old model used two very different approaches for hosted mode and web mode, but in both cases the point of interest was the boundary point between Java and JavaScript code, when a JavaScript object passes into "the Java world".  This boundary point was most often the return value of a JSNI function, but could also occur when a JSNI function accessed Java code through a field assignment, or by passing parameters to a Java function called from JSNI.

In hosted mode, we created an instance of `JavaScriptObject` (or a `JavaScriptObject` subclass) as we marshalled the value from JavaScript to Java.  This instance served as a Java wrapper for the underlying JavaScript object.  It had strong type identity, and handled `instanceof`, casts, and polymorphic calls through the normal Java mechanisms.

In web mode, we did not wrap the underlying JavaScript object in the traditional sense with a peer object, because that would have impeded performance.  Instead, we decorated the underlying JavaScript object with sufficient type information to handle runtime type checks and polymorphic dispatch as needed.

## Problems with old approach
The biggest problem was the differences in behavior between hosted and web mode.  In hosted mode, one particular JavaScript object could be wrapped by different `JavaScriptObject` wrapper instances (this happened when the same JavaScript object crossed the boundary at two different times).  This broke object identity because two different wrapper objects wrapping the same underlying JavaScript object should have been `==` to each other, but weren't.  In addition, the two different wrappers on the same object could be different different `JavaScriptObject` subtypes, which was even more confusing.

In web mode, a single `JavaScriptObject` retained its identity throughout, but other problems arose.  When a JavaScript object was decorated, it was given a permanent runtime type.  That type could later be changed to a more specific type, but it could not be made looser and could not reliably be "cross cast" to a sibling type in the hierarchy.  Even more confusing, the type of such an object could appear in certain cases to change "on the fly" in unexpected ways.  Example:

```
JavaScriptObject jso = getJsoFromNativeCode();
alert(jso instanceof Element); // false
Element element = getThatSameJsoAsElement();
alert(jso instanceof Element); // unexpectedly true
Event event = getThatSameJsoAsEvent();
alert(jso instanceof Event);   // nondeterministic answer depending on compiler output order!!
```

One other problem with the old approach was that the **declared** type at the boundary point was absolutely critical.  This raised problems when Java 1.5 generics come into the mix, because of type erasure.  The declared generic type ended up having no impact on type at the boundary point, with disastrous results.  Example:

```
class JsArray<T> extends JavaScriptObject {
  native T get(index i) /*-{ ... }-*/;
}

JsArray<Element> myArray = getJsElementArray();
Element element = myArray.get(0);  // Class cast exception!
```

The problem here was that the `get()` method is erased to `Object`, which defeated the model of giving an object the correct type at the boundary point.

## Goals of the new model, Overlay Types
  1. Pin down a specification so that developers can safely subclass `JavaScriptObject` in a forwards-compatible way.
  1. Overlay types behave consistently between hosted mode and web mode.
  1. Zero overhead to use overlay types in production.
  1. Provide a shorthand syntax for JavaScript interop to reduce boilerplate code.
  1. Do not modify the underlying JavaScript object.
  1. The design makes it possible to run in a mode where assumptions are checked.

## Non-goals
  1. It is not a goal to honor every Java language semantic.  We are willing to restrict some Java language semantics for overlay types.

## Use cases
  1. A developer should be able to easily create arbitrary overlay types for zero-overhead (in web mode) integration with JavaScript APIs.
  1. Allow casting between any overlay types via assertion; this is a more realistic model of the JavaScript type system and allows the flexibility we want.
  1. Support the use of generics in overlay types (see the `JsArray` example above).
  1. Easy zero-overhead interop with JSON and the XML DOM (which does not support expandos).

## Solution
We solved this with the concept of "Overlay Types".  An Overlay Type is defined as a subclass of `JavaScriptObject`, but in some cases the term will include `JavaScriptObject` itself.  The key idea is to prevent the need use any kind of virtual dispatch when invoking methods on an overlay type.  Barring virtual dispatch leads to good implementations in both web mode and hosted mode, and it seems necessary anyway due to the constraint of not modifying the underlying object.  The removal of virtual dispatch is enforced by several restrictions on overlay types described below.

An additional note is that any overlay type can be cast to any other overlay type.  The cast will always succeed, and the Java `instanceof` construct will always evaluate to "true".  This is where the term "overlay type" stems from; they Java declared type is "overlaid" on top of a JavaScript object.  When programmers use an overlay type, they assert to GWT that they know what the underlying `JavaScriptObject` is and how it will behave.  If they want extra checking, they should write some kind of wrapper objects around overlay types.


## Restrictions on Overlay Types
The restrictions are as follows.  An Overlay Type is defined as a subclass of `JavaScriptObject`, but in some cases the term will include `JavaScriptObject` itself.


  1. All instance methods on overlay types must be one of: explicitly final, a member of a final class, or private.  _Methods of overlay types cannot be overridden, because calls to such methods could require dynamic dispatch._
  1. Overlay types cannot implement interfaces that define methods. _This prevents virtual calls that would arise by upcasting to the interface and then calling through the interface.  The programmer should instead use a wrapper, for example using `Comparator` instead of implementing `Comparable`._
  1. No instance methods on overlay types may override another method. _This catches accidents where `JavaScriptObject` itself did not finalize some method from its superclass._
  1. Overlay types cannot have instance fields.  _The fields would have no place to live in web mode.  Programmers should instead make an explicit wrapper class and put the fields there._
  1. Nested overlay types must be static.  _The implicit `this` fields of a non-static inner class has the same problems as an explicit field._
  1. "new" operations cannot be used with overlay types.  _This avoids ever being able to try to instantiate overlay types using the new keyword. New overlay type instances can only come from JSNI, as in previous versions of GWT._
  1. Every overlay type must have precisely one constructor, and it must be protected, empty, and no-argument.



## Implementing in web mode
The trickiest part of the web mode implementation resolves around these four facts:
  * every reference type extends `Object`, including `JavaScriptObject`
  * every reference type can be explicitly upcast to `Object`
  * typical use of generics relies on erasure to `Object`
  * baseline polymorphic behavior on `Object` focuses on `toString()`, `equals()`, `hashCode()`, and `getClass()`

`JavaScriptObject` must define `toString()`, `equals()`, `hashCode()`, making them final as specified in Rule #1. This prevents anyone from attempting to override them expecting polymorphic behavior.  `getClass()` will be handled internally by the compiler.  But to maintain no polymorphism with overlay types, we must conclude that in web mode all calls to `toString()`, `equals()`, `hashCode()`, and `getClass()` must not rely on virtual dispatch where the instance might be an overlay type.

Given a type T, generating code for calls to `toString()`, `equals()`, `hashCode()`, or `getClass()` always falls into one of three cases.

  1. T is known to be any overlay type. In this case, we call the final version of `toString()`, `equals()`, `hashCode()`, or`getClass()` on `JavaScriptObject` itself.  This implies that `getClass()` will return `JavaScriptObject.class` for **all** `JavaScriptObject` instances.  It will never return the class object of a subclass of `JavaScriptObject`.
  1. T is known to be a subclass of `Object` that is definitely not an overlay type. In this case, we can use normal polymorphic dispatch since there is no risk the instance is actually an overlay type.
  1. T is known only to be an `Object`, and it isn't known whether or not it is an overlay type or not. We can test for a special method available only on non-overlay types to make the determination as to whether to go to Case 1 or Case 2.
```
    String s = CompilerMagic.isJavaObject(o) ? o.toString() : ((JavaScriptObject)o).toString();
```

> ### Consequences
  * We can freely allow casts wherever they make sense, including being able to upcast overlay types to `Object` and vice-versa. Only in cases whether type tightening is helpless do we pay the additional cost of Case 3 above.
  * We never need to wrap overlay types. This simplifies the compiler, reduces the chance for bugs, and eliminates a potential source of runtime memory costs.

## Implementing in hosted mode
In hosted mode, there is a single concrete instantiable type that represents `JavaScriptObject` and all of its subclasses.  This type is named `JavaScriptObject$`.  The real overlay type hierarchy is transformed into a hierarchy of totally empty interfaces.  `JavaScriptObject$` then implements every overlay interface type; this allows things like type casting and overload resolution to work properly in the JVM without additional magic.

The second problem that must be solved is how to actually implement overlay methods.  To solve this, we transform every overlay type into an "implementation type", which is named such that `Element` becomes `Element$`.  The overlay implementation types contain all of the static methods from the original class, as well as any instance methods which are rewritten as static methods.

The last step is to update all call sites and static field references to all overlay types.  Method calls to the `Object` methods (e.g. `toString()`, `equals()`, etc) are dispatched via real polymorphism to the final implementation of those methods in `JavaScriptObject$`.  All other instance method calls are designedly non-virtual, and therefore transformed into the exact static implementation in the corresponsding overlay implementation class.  Static field accesses and method calls are simply retargetted to the corresponding overlay implementation class.

> ### Creating the overlay interface type
    1. Every overlay type is transformed from a class to an empty interface.

> ### Creating the overly implementation type
    1. Every overlay type is copied into an implementation class.
    1. The new type has the same name as the old type, plus the `$` character appended.
    1. The new type's superclass is the implementation class of its original superclass (except for `JavaScriptObject$` itself whose superclass remains `Object`).
    1. All instance methods are made static and given a synthetic `this` parameter of the interface type.
      * All `this` references are changed to access the synthetic parameter.
    1. All static methods and fields are copied unchanged.

> ### Rewrites on all classes
    1. References methods and fields in overlay types are rewritten to target that overlay implementation type.
    1. Instance methods calls are also rewritten as static calls; the instance qualifier becomes argument 0.

> ### Marshalling from JavaScript to Java
    1. Marshalling of reference-type JsValues into Java used to be based only on the declared type of the value.  Now it is based only on the runtime type of the value.
    1. A `ClassCastException` is thrown if the marshalled value does not conform to the declared type of the value.
    1. JavaScript strings (both primitive and wrapper) **always** marshal as Java `String`.  It no longer legal to pass a string if an overlay type is declared.
    1. JavaScript non-string objects **always** marshal as `JavaScriptObject$`.
    1. Wrapped Java objects are **always** unwrapped.  It is no longer legal to treat a wrapped Java `Object` as an overlay type.

> ### Marshalling from Java to JavaScript
    1. Previously, declaring an overlay type argument used to be the trigger for unwrapping an overlay type into its native value.  Declaring an argument of type `Object` and passing in an overlay type instance would cause the instance to be rewrapped.  Now, any overlay type object passing into JavaScript is always unwrapped back to the underlying JavaScript object.
    1. Java `String` is always passed as a JavaScript string value, even if the declared type is `Object`.

## Examples of how the pieces fit together in hosted mode
> ### Original overlay types
```
public class Customer extends JavaScriptObject {
  public final native String getFirstName() /*-{ return this.first_name; }-*/;

  public final native String getLastName() /*-{ return this.last_name; }-*/;

  public final native int computeAge() /*-{ return this.getComputedAge(); }-*/;

  final native String getArea();
}

public class Shape extends JavaScriptObject {
  public final native double getArea() /*-{ return this.getArea(); }-*/;
}

public class Rectangle extends Shape {
  public final native int getWidth();
  public final native int getHeight();
}
```
> ### Original call site
```
public void foo(Customer c) {
  System.out.println(c.getFirstName());
  System.out.println(c.getArea());        // prints a string
  Shape s = (Shape) (JavaScriptObject) c; // succeeds always
  System.out.println(s.getArea());        // prints a double
}
```
> ### Overlay types rewritten with static methods
```
interface Customer extends JavaScriptObject { }
class Customer$ {
  public static String getFirstName(Customer this) { ... }
  // etc
}

interface Shape extends JavaScriptObject {
class Shape$ {
  public static double getArea(Shape this) { ... }
}

interface Rectangle extends Shape {
class Rectangle$ {
  public static int getWidth(Rectangle this) { ... }
  public static int getHeight(Rectangle this) { ... }
}
```
> ### Call site after rewrites
```
public void foo(Customer c) {
  System.out.println(Customer$.getFirstName(c));
  System.out.println(Customer$.getArea(c)); // prints a string
  Shape s = (Shape) (JavaScriptObject) c;   // succeeds always
  System.out.println(Shape$.getArea(s));    // prints a double
}
```

## Updates for GWT 2.0 (In-progress DRAFT)

Overlay types have demonstrated their utility for producing type-safe JavaScript (e.g. the `dom.client` package).  It is difficult to isolate the JSO implementation from consuming code without the use of an intermediate type due to the restriction on declaring JSO types that implement interfaces.  A new feature in GWT 2.0 will introduce the single-JSO-implementation, `@SingleJsoImpl`, annotation applied to an interface type that will allow exactly one JSO subtype to implement that interface.

### Example Use
```
@SingleJsoImpl
interface Person {
  String getName();
  String getAddress();
}

/** The only JSO-based implementation of Person */
public final class JsoPerson extends JavaScriptObject implements Person {
  protected JsoPerson() {}
  public native String getName() /*-{return this.name;}-*/;
  public native String getAddress() /*-{return this.address;}-*/;
}

/** Legal to have any number of regular Java types implement the interface. */
public class JavaPerson implements Person{ ... }
```

### Goals
  * Allow increased decoupling of overlay types and consuming code by allowing an interface to be extracted from the overlay types. _This should improve the testability and reusability of consuming code, especially in the common use case of JSO-cum-DTO._

### Restrictions
The restrictions on overlay types given above still apply, with the following modifications:
  * An overlay type may implement any number of non-trivial interfaces that are annotated with `@SingleJsoImpl`. _This makes it obvious to the system which interfaces should have these alternate rules applied._
  * Any given `@SingleJsoImpl` interface may be implemented by exactly one overlay type, although that overlay type may be further extended. Any number of non-overlay types may implement the interface. _This allows any method defined in the interface to be statically-dispatched in the fallback case._
  * If a `@SingleJsoImpl` interfaces extends a non-trivial interface, that super-interface must also be be annotated with `@SingleJsoImpl`.
  * The `@SingleJsoImpl` annotation may only be applied to interfaces.
  * Native JSNI methods may not refer to instance methods within `@SingleJsoImpl` types, just as they may not refer to instance methods on overlay types.

### Hosted Mode Implementation

The hosted-mode implementation relies on `JavaScriptObject$` implementing all `@SingleJsoImpl` interfaces and supporting polymorphic dispatch.  The following bytecode transformations are applied: (note that code samples are only intended to be representative of transformations applied)

  * `JavaScriptObject$` implements all `@SingleJsoImpl` interfaces via the disemboweled overlay types.
  * All methods in a `@SingleJsoImpl` interface are renamed to include the type name of the declaring interface type.
    * Implemented in `RewriteSingleJsoImplDispatches.java`.
    * `Person.getName()` would be transformed into `Person.com_google_Person_getName()`
    * All call sites with a `@SingleJsoImpl` type as the owner are rewritten with a mangled name.  Call sites to more specific types are left unchanged.
    * _This allows `JavaScriptObject$` to unambiguously dispatch methods with identical descriptors from unrelated interfaces_.
  * `JavaScriptObject$` implements all of the mangled methods as trampoline functions to the single JSO-derived implementation.
    * Implemented in `WriteJsoImpl.java`.
```
public String com_google_Person_someMethod(int a, int b) {
  JsoPerson$.someMethod$(this, a, b);
}
```
  * Non-overlay types that implement a `@SingleJsoImpl` interface have trampoline methods added to call the un-mangled methods.
    * Implemented in `RewriteSingleJsoImplDispatches.java`.
```
public class JavaPerson implements Person {
  public String com_google_Person_someMethod(int a, int b) {
    return this.someMethod(a, b);
}
```

> This implementation is relatively simple because it does not require significant transformation of call-sites, and continues to rely on regular `INVOKEINTERFACE` JVM behaviors.  The net effect is that the `JavaScriptObject$` type is assignable to all `@SingleJsoImpl` types and contains many trivial trampoline methods.

### Web Mode Implementation

The changes to web mode are similar to those in hosted mode, where the dispatch, casts, and instanceof checks are the primary changes made.

'JavaScriptObject' implements all interfaces that are known to have both overlay and non-overlay implementations.

Casts and instanceof checks are accomplished by adding a flag to `JCastOperation` and `JInstanceOf` to allow a non-null, non-Java-derived (i.e. `o.typeMarker$ != nullMethod`) object to pass the type checks.  Additional methods are added to the `Cast` utility class and `CastNormalizer` updated. _This means that `x instanceof Person` or `(Person)x` will succeed for **any** `JavaScriptObject`, regardless of whether or not the underlying object is in fact compatible with the overlay type._
  * TODO: Consider whether or not a magic `boolean allowCast(JavaScriptObject o)` should be supported in each overlay type definition.

Any untightened, polymorphic method dispatch sites are rewritten to use the single implementation of a `@SingleJsoImpl` method as a fallback in the case that the target object is a non-Java-derived object.  The use of `typeMarker$` covers objects from external GWT modules.
```
  Person p;
  p.doSomething();
```
becomes
```
  p.typeMarker$ == nullMethod ? p.doSomething() : JsoPerson_doSomething$(p);
```


### Other Updates

  * `JsoRestrictionsChecker` has been updated with the new rules.  One design change is that the checker must accumulate state before a final check pass in order to completely validate the type hierarchy of overlay types.  The order in which the JDT visitors will visit types is undefined, so it may be the case that an overlay type is visited before its `@SingleJsoImpl` interface.
  * `TypeOracle` provides a getter to return all types annotated with `@SingleJsoImpl`.
  * `JTypeOracle` gains similar functionality and some extra utility methods.

### Notes

  * The implementation of overlay types in GWT 1.5 does not support Generators defining new JSO subtypes (due to the need to redefine or otherwise extend `JavaScriptObject$` during subsequent compilation).  This restriction is still in place.