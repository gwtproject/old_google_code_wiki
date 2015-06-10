# RequestFactory changes in GWT 2.4

[Issue tracker search](http://code.google.com/p/google-web-toolkit/issues/list?can=1&q=requestfactory%20Milestone=2_4&sort=id&colspec=ID%20Type%20Status%20Owner%20Milestone%20Summary%20Stars%20Releasenote)



## Overview
  * (Issue 5367 (on Google Code)) RequestFactory now supports polymorphic return values.
  * (Issue 5394 (on Google Code)) Make type and operation tokens more compact
    * Method overloads are now supported in RequestContexts.
  * (Issue 5901 (on Google Code)) RequestFactory now uses the `javax.validation.ConstraintViolation` interface
  * (Issue 6035 (on Google Code)) Proxy interfaces can be composed
  * (Issue 6139 (on Google Code)) Memory leak in JRE-clean implementation fixed
  * (Issue 6234 (on Google Code)) RequestContext interfaces can be composed
  * (Issue 6253 (on Google Code)) All RequestFactory and AutoBean interfaces have moved to the `com.google.web.bindery' namespace
  * (Issue 6393 (on Google Code)) `RequestFactoryServlet.getThreadLocalServletContext()` provides access to the current HTTP request's `ServletContext`
  * (Issue 6469 (on Google Code)) Provide getters for `ServerFailure -> Request -> RequestContext -> RequestFactory -> RequestTransport`
  * (Issue 6705 (on Google Code)) RequestFactory validation is now compile-time instead of runtime. See RequestFactoryInterfaceValidation for more details
  * `RequestContext.append()` and `RequestBatcher` added

## Polymorphism support

A proxy type will be available on the client if it is:
  * Referenced from a `RequestContext` as a `Request` parameter or return type.
  * Referenced from a referenced proxy.
  * A supertype of a referenced proxy that is also a proxy (i.e. assignable to `EntityProxy` or `ValueProxy` and has an `@ProxyFor(Name)` annotation).
  * Referenced via an `@ExtraTypes` annotation placed on the RequestFactory, `RequestContext`, or a referenced proxy.
    * Adding an `@ExtraTypes` annotation on the RequestFactory or `RequestContext` allows you to add subtypes to "some else's" proxy types.

Type-mapping rules:
  * All properties defined in a proxy type or inherited from super-interfaces must be available on the domain type.
    * This allows a proxy interface to extend a "mix-in" interface.
  * All proxies must map to a single domain type via a `@ProxyFor(Name)` annotation.
  * The `@ProxyFor` of the proxy instance is used to determine which concrete type on the server to instantiate.
  * Any supertypes of a proxy interface that are assignable to `EntityProxy` or `ValueProxy` and have an `@ProxyFor(Name)` annotation must be valid proxies.
    * Given `BProxy extends AProxy`: if only `BProxy` is referenced (e.g. via `@ExtraTypes`), it is still permissible to create an `AProxy`.
  * Type relationships between proxy interfaces do not require any particular type relationship between the mapped domain types.
    * Given `BProxy extends AProxy`: it is allowable for `BEntity` not to be a subclass of `AEntity`.
    * This allows for duck-type-mapping of domain objects to proxy interfaces.
  * To return a domain object via a proxy interface, the declared proxy return type must map to a domain type assignable to the returned domain object.
  * The specific returned proxy type will be the most-derived type assignable to the declared proxy type that also maps to the returned domain type or one of its supertypes.

## RequestFactorySource and Annotation Processing

Users who depend on RequestFactorySource must now compile their proxy interfaces with the RequestFactory annotation processor.  This tool is bundled in the `requestfactory-client.jar` or available separately in `requestfactory-apt.jar`.  For Java 6 users, `javac` will automatically detect the annotation processor.  Eclipse users will need to enable annotation processing via `<Project properties> --> Java Compiler --> Annotation Processing` and add `requestfactory-apt.jar` to the list of jars in `Java Compiler --> Annotation Processing --> Factory Path`.

Additionally, the `-Averbose=true` flag can be passed to `javac` (or specified in the `Annotation Processing` configuration UI) to enable diagnostic output for the annotation processor.

More information can be found on the RequestFactoryInterfaceValidation wiki page.

## ServiceLayer changes

Several of the `ServiceLayer.resolveX()` method signatures have changed in this release.  These changes were made in order to allow the use of obfuscated type and operation tokens to reduce payload and generated JS size and to allow the use of overloaded method names in `RequestContext` subtypes.  Users who have written their own `ServiceLayerDecorator` subclasses that override any of the `resolveX()` methods will need to modify their code to conform to the new API.  In general, the expected behavior of the `resolveX()` methods is unchanged. Users who need to retain compatibility with 2.3 and 2.4 RequestFactory server code can leave methods with the old signatures in place, removing any `@Override` annotations.

## Improved request batching

Prior to GWT 2.4, it was only possible to chain `Request` instances that originated within a single `RequestContext`.  GWT 2.4 adds a `RequestContext.append()` method, which allows cross-context chaining.
```
interface ContextA extends RequestContext {
  Request<Boolean> requestA();
}
interface ContextB extends RequestContext {
  Request<Integer> requestB();
}
interface MyRequestFactory extends RequestFactory {
  ContextA ctxA();
  ContextB ctxB();
}
class Caller {
  MyRequestFactory factory;
  void doSomething() {
    ContextA ctxA = factory.ctxA();
    ctxA.requestA().to(new Receiver<Boolean>(){});
    ContextB ctxB = ctxA.append(factory.ctxB());
    // ctxA is still completely usable after calling append()
    ctxB.requestB().to(new Receiver<Integer>(){});
    ctxA.fire(); // ctxB.fire() would be equivalent
  }
}
```

Any `RequestContext` may have another newly-created `RequestContext` appended to it.  Once two `RequestContext` instances are joined, they may be used independently of one another until `fire()` is called on any of the `RequestContext` instances.

Building on top of the `append()` facility is the utility class `RequestBatcher` which makes it easy to combine all `Request`s made during one tick of the event loop into a single HTTP request.  A `RequestBatcher` vends an instance of a `RequestContext` and automatically calls `fire()` on it before the JavaScript execution returns to the browser (via `Scheduler.scheduleFinally()`).  The public `fire()` methods on the returned `RequestContext` and its `Request` objects will not trigger an HTTP request, although any `Receiver` provided will still be enqueued, allowing the `RequestContext` vended by a `RequestBatcher` to be used with existing code.

```
public class MyRequestBatcher extends RequestBatcher<MyRequestFactory, MyRequestContext> {
  // Could be provided to consumers via DI instead
  public static final MyRequestBatcher INSTANCE = new MyRequestBatcher();

  public MyRequestBatcher() {
    // MyRequestFactory could also be injected
    super(GWT.create(MyRequestFactory.class));
  }

  @Override
  protected MyRequestContext createContext(MyRequestFactory factory) {
    return factory.myRequestContext();
  }

  // Provide batched getter for an additional RequestContext type
  public OtherRequestContext otherRequestContext() {
    return get().append(getRequestFactory().otherRequestContext());
  }
}
```
```
public void respondToUserAction() {
  MyRequestContext ctx = MyRequestBatcher.INSTANCE.get();
  ctx.doUsualThings();
  // Calling fire() would be a no-op, although fire(Receiver<Void>) will enqueue the callback
}
```