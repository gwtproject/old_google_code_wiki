**Work in progress**

For documentation on using RequestFactory, see the [RequestFactory developer's guide](http://code.google.com/webtoolkit/doc/latest/DevGuideRequestFactory.html).



# Introduction

The RequestFactory system is composed of a number of discrete component pieces. This document will describe the rough functionality of the RequestFactory components as an aid to developers who wish to work on RequestFactory or adapt RequestFactory to non-standard deployment environments.  This document assumes a working knowledge of how to use RequestFactory.

<img src='http://i.imgur.com/WNqrg.jpg' alt='Schematic drawing of the major components of RequestFactory' title="If it doesn't fit on a sticky note, it's too complicated." />

# Flow

  * Instantiate an instance of a `RequestFactory` via `GWT.create()` or [RequestFactorySource](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/vm/RequestFactorySource.java)
  * Obtain an instance of `RequestContext` by calling an accessor method defined within the `RequestFactory`
    * Use `RequestContext.create()` and `RequestContext.edit()` to accumulate zero or more **operations** to be applied to domain objects by the server.
    * Obtain one or more `Request` objects to accumulate **invocations** on server code.
      * The `Request.to()` method provides a `Receiver` that will be notified of the outcome of the invocation associated with the `Request`.
    * Call `Request.fire()` or `RequestContext.fire()` to send the accumulated data to the server.
  * The accumulated state is transmitted to the server, where the following operations take place:
    * All domain objects referred to by the payload will be loaded.
    * Proxy objects are created for each referenced domain object.  These proxies are used later in the request processing to minimuze the amount of data returned to the client.
    * All accumulated operations will be applied to the domain objects by traversing properties of the proxies.
    * All method invocations in the payload are executed.
    * The versions of entities referred to in the original payload are compared to the up-to-date versions of the entities after all work has been performed.
      * Entities in the return payload graph (i.e. those reachable from the return values of an invocation) are returned to the client with a subset of properties controlled by the original `Request.with()` specification.
      * Entities referred to solely in the invocation arguments graph are checked to see if their version properties have changed.  If so, an `UPDATE` message is sent to the client for each changed entity, but no property data is sent.  This allows ephemeral ids to updated in a chained-persist use pattern.
      * Entities that can be no longer retrieved after all work has been performed are reported as having been deleted.
  * The return payload is assembled and sent to the client.
    * The various `Receivers` attached to `Requests` or the `RequestContext` are invoked to report success, failure, or validation violations.

# Parts Glossary

This section contains a brief discussion of each of the major parts of RequestFactory, in no particular order.

## Code

Depending on the terminology used, this may be called "business logic", an "RPC service layer", or "domain behavior."  Regardless of the phraseology used, this code exists as either static or instance methods that are mapped onto a `RequestContext` interface.

RequestFactory assumes that server code will execute in a synchronous, blocking fashion.

## POJO

RequestFactory minimizes the number of assumptions that it makes about server-side ("domain") objects.  Most operations are performed assuming that domain objects are simply "plain old Java objects" that are default-instantiable (i.e. a public no-arg constructor) and have pairs of getters and setters.  Setters are not required for immutable properties.  The requirement for default-instantiability can be removed via the use of a `Locator`.

### Entity

An "Entity" is any object with a well-defined identity and version.  Entities are mapped on the client as `EntityProxy` subtypes, while all other POJOS can be mapped as `ValueProxy` subtypes.  No assumption is made about the semantic meaning of the version property (no requirement to be `Comparable` or otherwise ordered), just that it is non-`equal()` to "old" version values.

While entity domain objects need not extend any particular base type or interface, they must implement the following informal protocol:
  * The domain type is default-instantiable if the client is allowed to use `RequestContext.create()` to arbitrarily create new instances.
  * `I getId()` must return a stable value.
  * `V getVersion()` must change every time the meaningful state of the domain object changes.
  * `static Entity findEntity(I id)` is used to retrieve a previously-referenced entity, using a value returned by `getId()`.  The `findEntity()` method may return `null` to indicate that the entity has been deleted or is otherwise unavailable. A specific naming scheme is used for this method: `find + Entity.class.getSimpleName()`.

Domain types that cannot satisfy some or all of the requirements of the entity protocol may participate as entities by using a `Locator`.

The id and version properties may return any type that can be transported by `RequestFactory`.  Objects with composite ids or versions may return a domain object, provided that there is an associated proxy for the domain type.  For example, if users wish to treat embedded objects as entities, the enclosing domain object can be used as the id.

Entities can be transferred in a sparse manner, as any undefined properties can be later retrieved since the entity has a persistent id.  By default, reference properties are not transferred unless explicitly specified by a `Request.with()`predicate.

### Values

A "Value" is any other POJO type that does not have a well-defined identity or version concept.  Value types must be default-instantiable, but do not need to implement any other methods.  Value types that are not default-instantiable may still be used by providing a `Locator` to vend instances.

All properties of a value type are transferred every time the value is referenced in a payload, since the value lacks an identity concept.

## PojoProxy

A proxy is a lightweight interface that exposes a subset of the property accessors of domain types to the RequestFactory client.  All developer-defined proxies must extend either 'EntityProxy' or 'ValueProxy'.  While the two proxy types extend a common `BaseProxy' type, this type exists solely to reduce API method count and is not directly usable by developers.

Every proxy must be annotated with a `@ProxyFor` or `@ProxyForName` annotation to declare which domain type the proxy provides access to.  It is legal for multiple proxy types to be mapped to the same domain type.  The effects of `@ProxyFor` and `@ProxyForName` are equivalent, although the latter uses string values instead of class literals to support the compilation of clients without access to server classes.

Proxies that map non-default-instantiable domain types can define a `Locator` in their `@ProxyFor` annotation.

It is important to note that the proxy types are required by the server code.  The RequestFactory server code uses the properties defined on the proxy objects to determine which properties may be accessed by clients and the `@ProxyFor` annotation to determine the domain type and optional `Locator`.  This level of indirection provides some security against malicious payloads attempting to manipulate arbitrary domain types.

## Locator

A `Locator` can be used to allow types that do not conform to the informal entity protocol to be used as entities with RequestFactory.  Locators are assumed to be default-instantiable, however this assumption can be modified by providing an alternate implementation of `ServiceLayer.createLocator()`.

## Persistence layer

RequestFactory has no particular assumptions about the persistence system used by the developer (or even if one is used at all).  The actual details of how any given pojo is persisted is left to the developer and treated as an implementation detail by RequestFactory.

Many samples include a `persist()` method as one of the first examples of server code, however this is merely a convention.  RequestFactory does not treat this as a special method.

## Request

<img src='http://i.imgur.com/yaYFp.jpg' alt='Schematic diagram of AbstractRequestContext and related types' />

A `Request` and its related type `InstanceRequest` represent a method invocation to be performed on the server.  The [AbstractRequest](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/AbstractRequest.java) base type is extended by the generator for each `RequestContext` method declaration.  The `Request` objects associate a `RequestData` bag of metadata and parameter values with the user-provided `Receiver` that was registered with `Request.to()` or implicitly by `Request.fire(Receiver)`.

The `InstanceRequest` interface allows instance methods on domain pojo types to be called.  The `InstanceRequest.using()` method sets the implicit 0-th argument of the invocation to the domain object represented by the proxy.

Instance methods on service objects can be called by providing a `ServiceLocator` type which will provide the instance of an object on which to invoke the desired business logic method.  This allows dependency-injection frameworks to be used to configure the service object.

## RequestContext

A `RequestContext` maps domain code into the client's view.  Each `RequestContext` must be annotated with a `@Service` or `@ServiceName` annotation.  For a domain method with the following signature
```
ReturnType methodName(SomeParam p0, int p1, String p2);
```
the mapped `RequestContext` must have a similar method declaration that uses the client types instead of the domain types:
```
Request<ReturnTypeProxy> methodName(SomeParamProxy p0, int p1, String p2);
```

The bulk of the `RequestContext` implementation can be found in [AbstractRequestContext](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/AbstractRequestContext.java).  Implementations of user-defined methods are provided by a generator or reflection-based proxy.  The generated subtypes of `AbstractRequestProxy` provide their own `AutoBeanFactory` interfaces which define only those proxy types that are reachable from the methods defined within the `RequestContext` interface in order to improve code pruning.

The `ProxySerializer` provided by the `RequestFactory` is essentially a loopback version of `AbstractRequestContext` implemented in [ProxySerializerImpl](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/ProxySerializerImpl.java).

## RequestFactory

The `RequestFactory` interface defines the top-level API used by developers.  Through the `RequestFactory` interface, the developer can obtain instances of `RequestContext` objects, serialize and restore id and proxy objects, and use the `find()` method to retrieve previously-accessed entities.

Most of the `RequestFactory` API is implemented by [AbstractRequestFactory](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/AbstractRequestFactory.java), with the user-defined methods added via code-generation or a reflection-based proxy.  The `AbstractRequestFactory` type send events to a user-provided `EventBus` and delegates payload handling to a optional `RequestTransport` type.  The factory also maintains a map of encoded id and version pairs to suppress redundant `UPDATE` messages.

## RequestFactoryinterfaceValidator

The [RequestFactoryInterfaceValidator](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/RequestFactoryInterfaceValidator.java) class compares the structures of the client-side interfaces with the method signatures of the domain objects.  Starting from a RequestFactory interface, the validator examines each `RequestContext` and the various proxy types reachable from the contexts.  The validator records the mappings between the client interfaces and the domain types and their optional locators as expressed in the `@Proxy` and `@Service` annotations.  The `ResolverServiceLayer` uses this accumulated data to find the proxy `Class` and `Method` objects that a payload refers to.

Individual methods in a `RequestContext` or proxy type can be exempted from validation by applying the `@SkipInterfaceValidation` annotation.  This annotation is only necessary when customizations to the `ServiceLayer` are being used to offer alternate method dispatch semantics, to provide synthetic properties, or otherwise manipulate `SimpleRequestProcessor`'s view of the domain.

## RequestState

A [RequestState](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/RequestState.java) is used by `SimpleRequestProcessor` to encapsulate all temporary data necessary to process a single request.  In general, a `RequestState` maps domain objects to id objects and the id objects to proxy instances.  The proxy instances have ephemeral tag values that associate them with their domain objects.

## ServiceLayer

The [ServiceLayer](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/ServiceLayer.java) mediates all interactions between the `SimpleRequestProcessor` and the domain.  RequestFactory's default behaviors may be overridden by providing one or more `ServiceLayerDecorator` instance to `ServiceLayer.create()`.  For instance, the `ServiceLayer.setProperty()` method can be used to provide access control for specific users.

Because the API expressed by `ServiceLayer` is intimately tied to the services required by the `SimpleRequestProcessor`, the API is subject to change over time.  Efforts will be made to keep `ServiceLayerDecorator` source-compatible, however this API should be treated as only semi-public.  Developers who are advanced enough to take advantage of decorating the `ServiceLayer` will be able to adapt to changes as they come up.

The default implementation of `ServiceLayer` is comprised of several `ServiceLayerDecorator` types that group implementation implementation details by overall function.

Methods in the `ServiceLayer` API have a mostly-uniform naming scheme:
  * `createFoo()` controls the instantiation of domain objects, `Locator`, `ServiceLocator`, and service objects.  Users of the dependency-injection pattern will typically override the create methods to use their framework of choice.
  * `resolveFoo()` controls the mappings between client types and domain types.
  * `getFoo()` retrieves various declared and synthetic properties and methods of domain objects.

The `die()` and `report()` methods in `ServiceLayerDecorator` are both used to report fatal errors and abort request processing.  The difference between the two is that `report()` will send diagnostic information to the client whereas `die()` fails in a client-opaque way.  Typically, `report()` is used to send exceptions thrown by user-provided service methods, while `die()` is used for all other failures.

The default stack:
  * [ServiceLayerCache](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/ServiceLayerCache.java) is used to reduce the number of type-introspection calls.
  * User-provided `ServiceLayerDecorators` are inserted here.
  * [LocatorServiceLayer](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/LocatorServiceLayer.java) implements support for `Locator` and `ServiceLocator` types.
  * [ReflectiveServiceLayer](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/ReflectiveServiceLayer.java) implements the majority of the domain-object manipulation code.
  * [ResolverServiceLayer](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/ResolverServiceLayer.java) provides the client to domain type and method mappings.

Implementors of `ServiceLayerDecorator` types should always use `getTop().serviceMethod()` when calling methods defined in the `ServiceLayer` API in order to allow those methods to be further overridden.  Implementations of `ServiceLayerDecorator` should be stateless, as the default `RequestFactoryServlet` does not create a new `ServiceLayer` for each request.  If a stateful `ServiceLayerDecorator` is written, the developer is responsible for ensuring the appropriate lifecycle is used by the caller to `SimpleRequestProcessor`.

## ServiceLocator

A `ServiceLocator` is used to provide instance objects for non-static domain methods that are not defined on a domain object (where an `InstanceRequest` could be used instead).  Implementations of `ServiceLocators` are assumed to be default-instantiable, however this may be changed by providing alternate implementation of `ServiceLayer.createServiceLocator()`.

## SimpleRequestProcessor

<img src='http://i.imgur.com/i7eN0.jpg' alt='Schematic drawing of SimpleRequestProcessor' title='Boxes within boxes, wheels within wheels.' />

The [SimpleRequestProcessor](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/server/SimpleRequestProcessor.java) is relatively stateless.  All transient state necessary for processing an individual request is contained in a `RequestState` object.

## StableId

Every `EntityProxy` has a stable id object that can be accessed via `EntityProxy.stableId()` and is implemented via [SimpleProxyId](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/SimpleProxyId.java).  Most operations involving the lifecycle or encoding of id objects are contained in [IdFactory](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/IdFactory.java), which is a base type of `AbstractRequestFactory`, and [IdUtil](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/impl/IdUtil.java).

The `SimpleProxyId` may use one or more of the following fields to track identity information: `encodedAddress`, `clientId`, and `syntheticId`.  The `encodedAddress` is an opaque string that is never directly interpreted by the client and contains a serialized representation of the domain entity's id property.  The `clientId` and `syntheticId` fields are used for objects that do not have a persistent identity on the server.

At runtime, id objects have the following flavors:
  * Persistent, which represent entities that have been persisted by the server.  These id objects only define a meaningful `encodedAddress`.  Persistent ids and their serialized forms are valid indefinitely.
  * Ephemeral, which have a `null` `encodedAddress`.  Ephemeral ids are used to track client-created proxies before being persisted on the server.  In the general use pattern, ephemeral ids are upgraded to persistent ids upon receipt of a `PERSIST` message from the server that provides an `encodedAddress` for a given `clientId`.  Ephemeral ids and their serialized forms are valid for the lifespan of a `RequestFactory` instance.
  * Synthetic, which are valid for the lifetime of a single `RequestContext` and are used to encode object references to `ValueProxies` in serialized payloads.

An instance of the `SimpleProxyId` type provides stable `hashCode()` and `equals()` behavior across its entire lifetime, allowing it to be used as a `Map` key.  This guarantee applies across an upgrade from ephemeral to persistent status.  The implementation of `IdFactory.getId()` will reuse the same instance of an ephemeral or formerly-ephemeral object when upgraded to persistent status, however developers should not generally depend the runtime object identity of id objects.

## Wire format

RequestFactory supports two wire formats, a custom JSON-based protocol for communicating with `SimpleRequestProcessor` and a JSON-RPC format for communicating with other web services.  These JSON messages are generated from AutoBean types defined in [MessageFactory](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/shared/messages/MessageFactory.java).

# JRE compatible implementation

RequestFactory provides a JRE-compatible implementation that can be used for fast integration tests, server performance monitoring, or ad-hoc utility clients.  The implementation is accessed via the [RequestFactorySource](http://code.google.com/p/google-web-toolkit/source/browse/trunk/user/src/com/google/web/bindery/requestfactory/vm/RequestFactorySource.java) type.

# JSON-RPC support

RequestFactory includes experimental support for interacting with JSON-RPC services.  This mode is activated by placing a `@JsonRpcService` annotation on a `RequestContext`.  Each `Request` method within a `RequestContext` must be annotated with a `@JsonRpcWireName` annotation to specify the RPC method name.  Each parameter of the `Request` method declaration must be annotated with a `@PropertyName` annotation.  Exactly one parameter may instead by annotated with the `@JsonRpcContent` annotation to use the object as the request body for the RPC request.  When in JSON-RPC mode, it is legal to define additional setters within a `Request` subtype that can be used to set optional properties.  The additional setters must be annotated with a `@PropertyName` annotation.

# Random notes without a home

  * The `gwt.rpc.dumpPayload` environment variable can be set to `true` when attempting to debug RequestFactory behavior.