# What's new in trunk

  * RequestFactory is now completely usable from non-client code by using `RequestFactoryMagic.create()` to instantiate the RequestFactory type.
    * The `InProcessRequestTransport` can be used for synchronous, end-to-end tests in non-GWT test code.
  * **API break** The `JsonRequestProcessor` type has been replaced with the `SimpleRequestProcessor` type.
    * **New API** `ServiceLayer` isolates the request processing from the type system and reflection operations. This API is very low-level and the default `ReflectiveServiceLayer` will eventually accept a helper object to inject high-level behaviors.
  * **API break** The AutoBean framework has been promoted to a top-level package and is usable in both client and server code.
  * A service method declared in a `RequestContext` can take parameters of `EntityProxyId` type to avoid the need to `find()` an object just to use it as a method parameter.
    * TODO: Allow return types of `EntityProxyId`.
  * Multiple methods may be invoked in a single `RequestContext` before `fire()` is called.
  * (Issue 5549 (on Google Code)) Support boolean is/hasFoo() properties
  * (Issue 5522 (on Google Code), issue 5357 (on Google Code)) Value types / Embedded objects
    * **New API** A user-defined value object must extend the empty ValueProxy interface and declare a @ProxyFor annotation.
    * ValueProxy instances are never sparse and will implement equals() and hashCode() based on the values in the proxy.
      * The VP will include the ids for any referenced EntityProxy fields, but will not force data for the referenced EP's to be returned unless there's a with() clause is used.
    * A ValueProxy returned from an immutable EntityProxy is immutable.  The EntityProxy must be placed into an editable mode via the usual RequestContext.edit() before you can make a call like entityProxy.getValue().setFoo("bar").
    * A VP returned from a service method invocation is immutable.
    * **New API** A new "BaseProxy" interface will be added as a superclass of ValueProxy and EntityProxy to allow `RequestFactory.create` to operate on both value and entity types.
      * **API Break** RequestContext.edit() now specifies a `BaseProxy` as the lower bound.  If it were possible to place a `ValueProxy` into an editable mode without a reference to a `RequestContext`, it would be impossible for the mutable `ValueProxy` to guarantee that its `EntityProxy` getters could return mutable objects.
    * Because ValueProxy doesn't have a stableId() method, there's no way to use a VP with a call to find() or with any other service method that has an EntityProxyId argument.
      * If you think about a Date object, there's no real meaning in giving a Date an id.
    * If an EntityProxy has a value property, all of the properties of the VP are checked for mutations and sent to the server.
    * The value in the domain object will be replaced by a new instance if anything in the VP's state has changed.
      * This makes client updates applied to a ValueProxy potentially destructive to server-side state if the ValueProxy represents only a subset of the data in the value domain object.
      * If you don't want the destructive operation, don't use value objects, or use the to-be-written service helper / Locator to cook up your own id scheme.
    * Since we currently support Date (and it is mutable) the RF client-side code will use a subclass of Date that can be frozen to ensure that the owning EntityProxy must be edited.
  * (Issue 5368 (on Google Code)) Remove integer version constraint
    * **New API** Any simple value type, `ValueProxy`, or `EntityProxy` may be used as the version or id property for an `EntityProxy`.
    * This should make composite keys easier to work with.
  * (Issue 5111 (on Google Code)) Improve integration with arbitrary backends
    * **New API** The `ServiceLayer` API used by the request processor can be arbitrarily decorated via a `ServiceLayerDecorator`.
    * **New API** A `Locator` can be specified via the `@ProxyFor` or `@ProxyForName` annotations to remove the need for `findFoo()`, `getId()`, and `getVersion()` methods on the domain type.
  * (Issue 5111 (on Google Code)) Bulk operations in ServiceLayer API
    * This should allow for better datastore query planning.
  * (Issue 5512 (on Google Code)) Fix for covariant return types in the domain API
  * (Issue 5134 (on Google Code)) Ensure that RequestFactory interfaces can be extended
  * (Issue 5523 (on Google Code)) Client-side EntityProxy persistence
    * **New API** A `ProxySerializer` can be retrieved from the `RequestFactory` interface to allow `EntityProxy` and `ValueProxy` objects to be serialized into a `ProxyStore`.
      * The `ProxyStore` API is essentially a string:string map and it would be trivial to write a implementation based on an HTML `Storage` object.
    * **New API** The `DefaultProxyStore` is a simple in-memory store that can encode its contents into a JSON object literal.
      * It would be possible to pre-seed a host page using this store.
  * (Issue 5680 (on Google Code)) **New API** `ServiceLocator` API to allow `Request` methods to be invoked on non-static methods.
    * The `@Service` and `@ServiceName` annotations now have an optional `locator` value that specifies a `ServiceLocator` which will provide a service object instance.
  * (Issue 5675 (on Google Code)) Subtypes of `java.util.Date` cause exceptions
  * (Issue 5564 (on Google Code)) RF has bad handling of SC\_UNAUTHORIZED
    * **API Break** The `RequestEvent`; `UserInformation` and related proxies, requests and services; `AuthenticationFailureHandler` and `LoginWidget` were hacky work arounds for the lack of value objects and a service layer, and have been deleted. For the same reason, `DefaultRequestTransport` no longer takes an `EventBus` constructor argument.
    * **API Break** `TransportReceiver#onTransportFailure(ServerFailure)` replaces `TransportReceiver#onTransportFailure(String)`, to give custom transports useful error reporting.
    * **API Break** `ServerFailure` now has a boolean `isFatal()` method (usually true), which `Request#onFailure` checks before throwing a RuntimeException
    * The Expenses sample has been extended to show how to handle user authentication in the wake of these changes. It's easier now, and more flexible.
  * (Issue 5674 (on Google Code)) The server code will explicitly disallow persisted entities with null versions.
  * (Issue 5357 (on Google Code)) Proxy types may use primitive types for proxy property accessor methods.
    * **API Break** The RequestFactory interface validator has been tightened to require that the a proxy property type matches the domain type exactly. If the domain type defines an `int getFoo()` property accessor, the matching property accessor in the proxy type must also be an `int` instead of an `Integer`.

# What's in review
# What's coming
  * That's it!

# Add to 2.1.1 smoke testing
  * The server components of RequestFactory have been completely rewritten.
    * Since most of the interesting code is now shared between the client and the server, there should be fewer cases of divergent expectations.